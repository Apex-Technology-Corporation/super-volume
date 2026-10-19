# 🔊 Super Volume

Extensión para **Google Chrome** (Manifest V3) que amplifica el audio de la
pestaña activa hasta **600%** usando la **Web Audio API**, e incorpora un
ecualizador de 3 bandas con presets, refuerzo de voz y graves, y modos de
sonido virtual (2.0 / 2.1 / 5.1 / 7.1). Todo el procesamiento ocurre dentro
del navegador: no modifica el volumen del sistema operativo y no envía datos
a ningún servidor.

- **Versión:** 2.0.0 (`manifest.json`)
- **Autor:** [alvarosiles](https://github.com/alvarosiles)
- **Contacto:** alvarosiles.developer@gmail.com
- **Licencia:** MIT (ver [LICENSE](LICENSE))

---

## Índice

1. [Descripción general](#1-descripción-general)
2. [Objetivo y problema que resuelve](#2-objetivo-y-problema-que-resuelve)
3. [Características principales](#3-características-principales)
4. [Arquitectura](#4-arquitectura)
5. [Estructura del proyecto](#5-estructura-del-proyecto)
6. [Tecnologías y versiones](#6-tecnologías-y-versiones)
7. [Módulos y funcionalidades](#7-módulos-y-funcionalidades)
8. [Base de datos (almacenamiento)](#8-base-de-datos-almacenamiento)
9. [Backend y API (protocolo de mensajes)](#9-backend-y-api-protocolo-de-mensajes)
10. [Configuración e instalación](#10-configuración-e-instalación)
11. [Variables de entorno](#11-variables-de-entorno)
12. [Ejecución en desarrollo](#12-ejecución-en-desarrollo)
13. [Compilación y despliegue](#13-compilación-y-despliegue)
14. [Seguridad y consideraciones importantes](#14-seguridad-y-consideraciones-importantes)
15. [Limitaciones identificadas](#15-limitaciones-identificadas)
16. [Personalización](#16-personalización)

---

## 1. Descripción general

Super Volume es una extensión de navegador sin backend ni dependencias
externas, compuesta por tres piezas:

- un **popup** (`popup.html` / `popup.css` / `popup.js`) con la interfaz,
- un **content script** (`content.js`) que se inyecta en las páginas y
  construye el grafo de audio,
- un **service worker** (`background.js`) que mantiene el *badge* del icono.

La extensión detecta los elementos `<video>` y `<audio>` de la página
(incluidos los que se añaden dinámicamente en aplicaciones de una sola
página) y enruta su salida por una cadena de nodos de Web Audio:
filtros de ecualización → etapa de espacialización → ganancia.

## 2. Objetivo y problema que resuelve

| Problema | Cómo lo resuelve |
|---|---|
| Un video suena muy bajo aunque el volumen del sistema esté al máximo. | Un `GainNode` multiplica la señal de la pestaña de ×0 a ×6 (0–600%). |
| Subir el volumen del sistema afecta a todo lo que suena en el equipo. | El procesamiento vive en el `AudioContext` de la pestaña; el volumen del sistema no se toca. |
| Los diálogos se pierden entre música y efectos. | *Voice Boost* (+10 dB en un filtro *peaking* a 2,5 kHz) y preset *Cine*. |
| Hay que reajustar el volumen cada vez que se vuelve a un sitio. | Opción "Recordar volumen por sitio" que guarda el valor por dominio y lo reaplica al cargar la página. |

## 3. Características principales

Verificadas en el código fuente:

- Volumen de **0% a 600%** (slider con paso de 5%) con transiciones suaves
  (`setTargetAtTime`) para evitar *clicks*.
- Botones **Reset** (vuelve a 100%) y **Mute/Unmute** (conserva el volumen previo).
- **Ecualizador de 3 bandas** con presets *Ninguno*, *Cine*, *Música* y *Juegos*.
- **Voice Boost** y **Bass Boost** independientes, que se suman al preset activo.
- **Modos de sonido virtual**: Stereo 2.0, Virtual 2.1, Virtual 5.1 y
  Virtual 7.1 (este último con intensidad configurable 0–100%).
- **Recordar volumen por dominio** (desactivado por defecto).
- **Badge** en el icono de la extensión con el % activo.
- **Tema** automático (según el sistema), claro u oscuro.
- **Idioma** automático (según el navegador), español o inglés (`_locales/`).
- Información de la pestaña activa: favicon, título y dominio; aviso cuando
  la página no tiene audio o no admite content scripts.
- Sin librerías externas ni peticiones a servidores.

## 4. Arquitectura

```
┌──────────────┐  chrome.tabs.sendMessage (SV_*)  ┌────────────────────────┐
│   popup.js   │ ───────────────────────────────▶ │       content.js       │
│  (interfaz)  │ ◀─────────────────────────────── │ (en cada página/frame) │
└──────┬───────┘        { ok, state }             └───────────┬────────────┘
       │                                                      │
       │ chrome.storage.local                                 │ Web Audio API
       │ (volumen por dominio, preferencias)                  ▼
       ▼                                     <video>/<audio> → MediaElementSource
┌──────────────┐                              → lowshelf 150 Hz → peaking 2,5 kHz
│ background.js│ badge del icono +            → highshelf 8 kHz → etapa surround
│ (service     │ valores por defecto          → GainNode (0–6) → destination
│  worker)     │
└──────────────┘
```

**Grafo de audio por frame (`content.js`)**

```
fuente ─▶ bassFilter ─▶ midFilter ─▶ trebleFilter ─▶ surroundInput
                                                        ├─▶ dryGain ───────────────────────┐
                                                        ├─▶ splitter ─▶ (directo + cross-feed
                                                        │      invertido con delay Haas) ─▶ merger ─▶ wideGain ─┤
                                                        └─▶ reverbSend ─▶ convolver (impulso sintético) ───────┤
                                                                                                    surroundOut ─▶ gainNode ─▶ destination
```

Decisiones de diseño relevantes:

- **Creación diferida del `AudioContext`**: solo se crea cuando hace falta
  (volumen ≠ 100%, mute, EQ o modo de sonido activos) y además existe al
  menos un elemento multimedia. Si el usuario nunca toca los controles, la
  pestaña no ejecuta procesamiento de audio extra.
- **Grafo fijo**: la estructura de nodos se crea una sola vez; cambiar de
  preset o de modo solo modifica ganancias y tiempos con `setTargetAtTime`,
  sin reconectar nodos.
- **Un `GainNode` por frame**, compartido por todos los elementos multimedia.
- **`MutationObserver`** para conectar elementos añadidos dinámicamente; solo
  actúa si hay algún efecto activo.
- **Reverb sin archivos externos**: la respuesta al impulso se genera con
  ruido blanco con caída exponencial (`createSyntheticImpulse`).
- **Selección de frame**: el popup consulta primero el frame principal
  (`frameId: 0`) y, si no tiene medios, envía el mensaje a todos los frames.

## 5. Estructura del proyecto

```
super-volume/
├── manifest.json            # Manifest V3: permisos, content script, service worker, i18n
├── background.js            # Service worker: valores por defecto + badge del icono
├── content.js               # Web Audio API: detección de medios, EQ, surround, ganancia
├── popup.html               # Interfaz del popup
├── popup.css                # Estilos y variables de tema (claro/oscuro)
├── popup.js                 # Lógica del popup: UI, mensajería, storage, i18n, tema
├── _locales/
│   ├── es/messages.json     # Textos en español (idioma por defecto)
│   └── en/messages.json     # Textos en inglés
├── icons/                   # icon16/32/48/128.png
├── gen_icons.py             # Regenera los iconos (requiere Pillow)
├── scripts/
│   ├── 1-install.sh         # Abre chrome://extensions en el Chrome del usuario
│   ├── 2-test-extension.sh  # Abre Chrome con perfil aislado y la extensión cargada (CDP)
│   ├── _cdp_loader.js       # Helper de 2-test-extension.sh (Node)
│   ├── _cdp_loader.py       # Helper alternativo (Python, sin dependencias)
│   ├── 3-build.sh           # Valida y empaqueta en dist/*.zip (bash)
│   └── 3-build.ps1          # Equivalente para PowerShell
├── docs/
│   ├── index.html           # Página pública del proyecto
│   └── privacy.html         # Política de privacidad
├── store-assets/            # Imágenes para la ficha de la Chrome Web Store
├── presentacion/            # Capturas del popup real para el portafolio
├── PUBLISHING.md            # Guía de publicación en la Chrome Web Store
├── AUDITORIA.md             # Auditoría de permisos/políticas previa a publicar
└── LICENSE                  # MIT
```

Los archivos que forman el paquete final son únicamente los que lista el
script de build: `manifest.json`, los tres `.js`, `popup.html`, `popup.css`,
los cuatro iconos y los dos `messages.json`.

## 6. Tecnologías y versiones

| Categoría | Tecnología | Versión / detalle verificable |
|---|---|---|
| Plataforma | Chrome Extensions, Manifest V3 | `manifest_version: 3`, `minimum_chrome_version: "102"` |
| Lenguajes | JavaScript (ES2017+, `async/await`), HTML5, CSS3 | Sin transpilador ni bundler |
| Audio | Web Audio API | `AudioContext`, `MediaElementAudioSourceNode`, `BiquadFilterNode`, `GainNode`, `DelayNode`, `ChannelSplitterNode`, `ChannelMergerNode`, `ConvolverNode` |
| APIs de Chrome | `chrome.storage.local`, `chrome.tabs`, `chrome.runtime`, `chrome.action`, `chrome.i18n` | — |
| Internacionalización | `_locales` + `chrome.i18n` | 55 claves en `es` y `en` |
| Herramientas | Bash, PowerShell, Node.js (`node --check`), Python 3 + Pillow (`gen_icons.py`) | Versiones no fijadas en el proyecto |
| Dependencias de runtime | Ninguna | No hay `package.json` |

## 7. Módulos y funcionalidades

### `content.js` — motor de audio

- `ensureAudioGraph()` crea el `AudioContext` y el grafo completo una sola vez.
- `attachElement()` conecta cada `<video>`/`<audio>` con
  `createMediaElementSource`; si falla, marca el elemento con
  `data-sv-skip="1"` para no reintentar.
- `applyVolume()` / `setMuted()` controlan la ganancia (límite `MAX_VOLUME_PERCENT = 600`).
- `setEqPreset()` / `setEqBoost()` combinan preset + boosts + refuerzo del modo
  de sonido (`computeEqGains()`).

  | Preset | Graves (150 Hz) | Medios (2,5 kHz) | Agudos (8 kHz) |
  |---|---|---|---|
  | Ninguno | 0 dB | 0 dB | 0 dB |
  | Cine | +6 | +7 | −1 |
  | Música | +5 | +1 | +4 |
  | Juegos | +3 | +2 | +7 |
  | *Bass Boost* | +9 extra | — | — |
  | *Voice Boost* | — | +10 extra | — |

- `setSurroundMode()` / `setSurroundIntensity()` ajustan la etapa de
  espacialización:

  | Modo | Cross-feed | Delay | Reverb | Refuerzo EQ |
  |---|---|---|---|---|
  | Stereo 2.0 | — | — | — | — |
  | Virtual 2.1 | — | — | — | graves +4, medios +2 |
  | Virtual 5.1 | 0,35 | 8 ms | 0,12 | graves +2, medios +1 |
  | Virtual 7.1 | 0,5 × intensidad | 14 ms × intensidad | 0,2 × intensidad | graves +3, medios +1 |

- Al cargar, `init()` restaura el volumen recordado del dominio (si la opción
  está activa) y las preferencias globales de EQ y modo de sonido.

### `popup.js` — interfaz

- Lee la pestaña activa (`chrome.tabs.query({active, currentWindow})`), pide
  el estado real a `content.js` y lo refleja en la UI.
- Si no hay content script disponible (páginas `chrome://`, la Web Store,
  PDFs…), deshabilita los controles y muestra un aviso.
- Guarda el volumen por dominio con *debounce* de 300 ms; el valor 100% no se
  guarda (se elimina la entrada).
- Cambio de tema (auto → claro → oscuro) y de idioma (auto → EN → ES). Para
  forzar un idioma distinto al del navegador, carga el `messages.json`
  correspondiente desde el propio paquete.

### `background.js` — service worker

- `onInstalled`: inicializa `rememberEnabled: false` y `volumes: {}`.
- `tabs.onActivated` / `tabs.onUpdated`: actualiza el badge con el volumen
  guardado del dominio de la pestaña.

No existen roles, cuentas ni permisos de usuario: la extensión es de uso individual.

## 8. Base de datos (almacenamiento)

No hay base de datos ni servidor. La persistencia usa **`chrome.storage.local`**
(clave-valor, local al perfil del navegador):

| Clave | Tipo | Alcance | Descripción |
|---|---|---|---|
| `rememberEnabled` | boolean | global | Activa el volumen por sitio (por defecto `false`). |
| `volumes` | object `{ dominio: number }` | por dominio | Volumen 0–600 guardado; el dominio se normaliza sin `www.`. |
| `eqPreset` | `"none" \| "cine" \| "musica" \| "juegos"` | global | Preset del ecualizador. |
| `bassBoostEnabled` | boolean | global | Bass Boost. |
| `voiceBoostEnabled` | boolean | global | Voice Boost. |
| `surroundMode` | `"stereo" \| "virtual21" \| "virtual51" \| "virtual71"` | global | Modo de sonido. |
| `surroundIntensity` | number 0–100 | global | Intensidad de Virtual 7.1 (por defecto 60). |
| `theme` | `"auto" \| "light" \| "dark"` | global | Tema del popup. |
| `language` | `"auto" \| "en" \| "es"` | global | Idioma del popup. |

No hay migraciones ni esquemas versionados.

## 9. Backend y API (protocolo de mensajes)

No hay backend HTTP ni endpoints. La comunicación interna es por mensajes
`chrome.tabs.sendMessage` del popup al content script. Todas las respuestas
tienen la forma `{ ok: true, state }`.

| `type` | `value` | Efecto |
|---|---|---|
| `SV_GET_STATE` | — | Devuelve el estado actual. |
| `SV_SET_VOLUME` | number 0–600 | Aplica el volumen (con *clamp*). |
| `SV_SET_MUTED` | boolean | Mute/unmute conservando el volumen previo. |
| `SV_SET_EQ_PRESET` | `none\|cine\|musica\|juegos` | Cambia el preset. |
| `SV_SET_BASS_BOOST` | boolean | Activa/desactiva Bass Boost. |
| `SV_SET_VOICE_BOOST` | boolean | Activa/desactiva Voice Boost. |
| `SV_SET_SURROUND_MODE` | `stereo\|virtual21\|virtual51\|virtual71` | Cambia el modo de sonido. |
| `SV_SET_SURROUND_INTENSITY` | number 0–100 | Intensidad de Virtual 7.1. |

Objeto `state`:

```js
{ volume, muted, hasMedia, domain, eqPreset, bassBoost, voiceBoost, surroundMode, surroundIntensity }
```

Los valores de entrada se validan en `content.js`: los números se limitan a
su rango, y los presets o modos desconocidos vuelven a `none`/`stereo`.

## 10. Configuración e instalación

**Requisitos:** Google Chrome 102 o superior (u otro navegador Chromium con
soporte de Manifest V3).

Instalación manual en modo desarrollador:

1. Clona o descarga el repositorio.
2. Abre `chrome://extensions`.
3. Activa el **Modo de desarrollador**.
4. Pulsa **Cargar descomprimida** y selecciona la carpeta raíz del proyecto
   (la que contiene `manifest.json`).

## 11. Variables de entorno

La extensión **no utiliza variables de entorno** ni credenciales. Tampoco
requiere servicios externos para funcionar.

## 12. Ejecución en desarrollo

Scripts disponibles en `scripts/` (bash; en Windows, Git Bash):

```bash
./scripts/1-install.sh                  # Abre chrome://extensions en tu Chrome y muestra la ruta a cargar
./scripts/2-test-extension.sh [URL]     # Chrome con perfil aislado en /tmp, Developer mode activado
                                        # y la extensión cargada vía CDP (Extensions.loadUnpacked)
```

`2-test-extension.sh` necesita Node.js o Python 3 para ejecutar
`_cdp_loader.js` / `_cdp_loader.py`.

Tras editar el código, recarga la extensión desde `chrome://extensions` (↻).

Regenerar iconos (opcional, requiere Pillow):

```bash
python gen_icons.py
```

## 13. Compilación y despliegue

```bash
./scripts/3-build.sh                    # bash (requiere zip y Node.js)
powershell -File scripts\3-build.ps1    # Windows PowerShell
# → dist/super-volume-v<version>.zip
```

El script de build:

- valida que `manifest.json` y los `_locales/*/messages.json` sean JSON válido
  y que existan las claves `__MSG_*__` usadas en el manifest;
- comprueba que `description` no supere los 132 caracteres;
- bloquea el build si encuentra nombres de plataformas de terceros en el
  manifest, `README.md`, `docs/`, `store-assets/` o `_locales/`, porque la
  Chrome Web Store puede rechazarlo como *spam* de palabras clave;
- comprueba la sintaxis de los `.js` con `node --check`;
- empaqueta solo los archivos de runtime.

Publicación: sube el `.zip` al
[Chrome Web Store Developer Dashboard](https://chrome.google.com/webstore/devconsole).
El paso a paso (ficha, justificación de permisos, política de privacidad,
2FA) está en [PUBLISHING.md](PUBLISHING.md). La política de privacidad está en
`docs/privacy.html` y debe estar publicada en una URL pública antes de
enviar la extensión a revisión.

## 14. Seguridad y consideraciones importantes

| Permiso | Uso |
|---|---|
| `storage` | Guardar preferencias y volumen por dominio (`chrome.storage.local`). |
| `activeTab` | Identificar la pestaña activa desde el popup. |
| `host_permissions: <all_urls>` | Inyectar `content.js` en cualquier sitio y en todos sus frames (`all_frames`, `match_about_blank`). |

- No hay `eval`, `new Function`, `innerHTML` ni scripts remotos. El único
  `fetch` lee `_locales/<lang>/messages.json` del propio paquete
  (`chrome.runtime.getURL`).
- No hay telemetría ni llamadas de red. Los datos guardados (niveles de volumen
  por dominio y preferencias) no salen del dispositivo.
- El popup no carga fuentes ni recursos externos; los iconos son SVG en línea.
- `host_permissions: <all_urls>` es el permiso que más escrutinio recibe en la
  revisión de la Web Store; está justificado en `PUBLISHING.md` y `AUDITORIA.md`.
- La lista de dominios guardados en `volumes` revela qué sitios se visitaron
  con la opción activada. Se almacena solo de forma local.

## 15. Limitaciones identificadas

Detectadas en la revisión del código y al ejecutar la extensión:

- **Solo elementos `<video>`/`<audio>`**: los sitios que reproducen audio con
  su propio grafo de Web Audio, o cuyos elementos ya están conectados a otro
  `AudioContext`, no se amplifican (el elemento se marca con `svSkip`).
- **Medios de otro origen sin CORS (riesgo, no verificado en ejecución)**:
  según la especificación de Web Audio, un `MediaElementAudioSourceNode` sobre
  un recurso *cross-origin* sin CORS produce silencio en lugar de lanzar una
  excepción. En esos casos activar el boost podría silenciar el medio. El
  comentario de `attachElement()` supone que se lanza un error.
- **Una vez conectado, un elemento queda en el grafo** hasta que se recarga la
  página; volver a 100% no lo desconecta.
- **Ecualizador y modo de sonido son globales**; solo el volumen se recuerda por sitio.
- **El modo surround es virtual**: es una aproximación psicoacústica sobre
  salida estéreo (cross-feed, efecto Haas y reverb), no audio multicanal real.
- **Ganancias altas pueden saturar**: no hay compresor ni limitador antes de
  la salida.
- **Detalles visuales del popup** observados al ejecutarlo:
  - El relleno del slider de intensidad (Virtual 7.1) no se actualiza: hereda
    el degradado fijo de `.slider` y `popup.js` solo repinta el slider de volumen.
  - El favicon de respaldo se muestra en negro porque el color `%239aa0a6`
    queda doblemente codificado por `encodeURIComponent`.
  - Con Virtual 7.1 activo, el popup supera la altura disponible y necesita scroll.
- **El límite de 600% está definido en tres lugares** (ver sección 16).
- **Sin pruebas automatizadas** ni atajos de teclado (`commands` no está declarado).

## 16. Personalización

**Colores**: variables CSS al inicio de [`popup.css`](popup.css) (`--bg`,
`--card`, `--accent`, `--text-primary`…), con variantes para
`[data-theme='light']` y `prefers-color-scheme`.

**Límite de volumen**: hay que cambiarlo en tres lugares a la vez:

1. `content.js` → `MAX_VOLUME_PERCENT`
2. `popup.js` → `MAX_VOLUME`
3. `popup.html` → atributo `max="600"` de `#volumeSlider`

**Nuevo control**: añade el elemento en `popup.html`, su estilo en
`popup.css` y su listener en `popup.js`. Si debe afectar al audio, define un
nuevo `type` de mensaje y gestiónalo en el `switch` de
`chrome.runtime.onMessage` en `content.js`.

**Nuevos textos**: añade la clave en `_locales/es/messages.json` y
`_locales/en/messages.json`, y usa `data-i18n="clave"` en el HTML o `t('clave')`
en `popup.js`.

**Iconos**: reemplaza los PNG de `icons/` con los mismos nombres o regenera
con `gen_icons.py`.
