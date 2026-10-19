# Super Volume

## Descripción para tarjeta

Extensión de Chrome (Manifest V3) que amplifica el audio de cualquier pestaña
hasta 600% con la Web Audio API, con ecualizador, refuerzo de voz y graves, y
sonido envolvente virtual. Funciona sin servidores ni dependencias.

**Etiquetas:** Chrome Extension · Manifest V3 · Web Audio API · JavaScript · HTML · CSS · i18n

---

## Descripción completa

Super Volume es una extensión para Google Chrome que procesa el audio de la
pestaña activa directamente en el navegador. Detecta los elementos `<video>`
y `<audio>` de la página, incluidos los que aparecen dinámicamente, y los
enruta por un grafo de Web Audio propio. El grafo aplica ecualización de 3
bandas, una etapa de espacialización estéreo y una ganancia final de 0% a 600%.

El usuario controla todo desde un popup con slider de volumen, botones de
Reset y Mute, presets de ecualizador (Cine, Música, Juegos), Voice Boost,
Bass Boost y cuatro modos de sonido (Stereo 2.0, Virtual 2.1, 5.1 y 7.1).
La extensión puede recordar el volumen elegido para cada sitio y reaplicarlo
automáticamente. Incluye tema claro/oscuro/automático e interfaz en español
e inglés.

## Problema que resuelve

Muchos videos y audios web tienen un volumen de origen bajo, y subir el
volumen del sistema afecta a todas las aplicaciones a la vez. Super Volume
amplifica solo la pestaña que lo necesita y permite ajustar el tono (diálogos
más claros, graves más presentes) sin instalar software de audio en el equipo.

## Objetivo del proyecto

Ofrecer un amplificador y ecualizador de audio por pestaña que sea ligero,
privado (sin red ni telemetría) y compatible con las políticas de la Chrome
Web Store para Manifest V3.

## Público objetivo

Usuarios de Chrome que consumen video o audio en el navegador (cursos,
conferencias, música, cine, juegos web) y necesitan más volumen o claridad
que la que ofrece el sitio o el sistema.

## Mi participación técnica

- Según el historial de git, todos los commits del repositorio están firmados
  por **alvarosiles / Alvaro Siles Estrada**, que figura como `author` en
  `manifest.json` y como titular en la licencia MIT.
- *Pendiente de confirmar:* el reparto concreto de tareas (diseño de la
  arquitectura de audio, UI, scripts de build y publicación) y el uso de
  herramientas de asistencia durante el desarrollo.

## Funcionalidades destacadas

- **Amplificación 0–600%** con transiciones suaves (`setTargetAtTime`) para
  evitar *clicks* al mover el slider.
- **Ecualizador de 3 bandas** (low-shelf 150 Hz, peaking 2,5 kHz,
  high-shelf 8 kHz) con presets Cine, Música y Juegos.
- **Voice Boost (+10 dB) y Bass Boost (+9 dB)** independientes, que se suman
  al preset activo.
- **Sonido envolvente virtual** 2.1 / 5.1 / 7.1 mediante cross-feed de
  polaridad invertida, efecto Haas y reverb con impulso sintético. Virtual 7.1
  tiene intensidad regulable.
- **Volumen recordado por dominio**, reaplicado al cargar la página sin abrir el popup.
- **Badge** en el icono con el porcentaje activo.
- **Tema** claro, oscuro o automático e **idioma** español, inglés o automático.
- **Detección de contexto**: avisa cuando la página no tiene audio o no admite
  extensiones (`chrome://`, Web Store, PDFs) y deshabilita los controles.

## Tecnologías utilizadas

| Categoría | Tecnologías |
|---|---|
| Plataforma | Chrome Extensions, Manifest V3 (Chrome 102+) |
| Lenguajes | JavaScript (vanilla), HTML5, CSS3 |
| Audio | Web Audio API: `AudioContext`, `BiquadFilterNode`, `GainNode`, `DelayNode`, `ConvolverNode`, `ChannelSplitter/MergerNode` |
| APIs del navegador | `chrome.storage`, `chrome.tabs`, `chrome.runtime`, `chrome.action`, `chrome.i18n`, `MutationObserver` |
| Internacionalización | `_locales` (es, en) + carga dinámica para forzar idioma |
| Tooling | Bash y PowerShell (build y validación), Node.js (`node --check`), Chrome DevTools Protocol (carga automatizada para pruebas), Python + Pillow (iconos) |
| Distribución | Chrome Web Store (paquete `.zip` generado por script) |

## Arquitectura y características técnicas

- **Tres componentes desacoplados**: el popup (UI), el content script (motor de
  audio por frame) y el service worker (badge y valores por defecto),
  comunicados por un protocolo de mensajes propio (`SV_GET_STATE`,
  `SV_SET_VOLUME`, `SV_SET_EQ_PRESET`, `SV_SET_SURROUND_MODE`…).
- **Recursos bajo demanda**: el `AudioContext` solo se crea cuando hay un
  efecto activo y existe un elemento multimedia. Si no se usa, la extensión
  no añade carga de audio a la pestaña.
- **Grafo de audio fijo**: los nodos se crean una vez y los cambios de preset o
  modo solo ajustan parámetros, sin reconectar. Así se evitan cortes y *pops*.
- **Soporte para SPA**: un `MutationObserver` conecta los reproductores que se
  montan después de la carga, y solo actúa si hay efectos activos.
- **Multi-frame**: el content script corre en todos los frames y el popup
  consulta primero el frame principal, con respaldo al resto. Así se resuelven
  las páginas que cargan iframes auxiliares sin medios.
- **Pipeline de build con validaciones**: JSON del manifest y de los locales,
  longitud de la descripción, sintaxis JS y detección de nombres de marcas de
  terceros que podrían provocar un rechazo en la Web Store.

## Integraciones

- **APIs nativas de Chrome** (storage, tabs, action, i18n). No hay
  integraciones con servicios externos ni APIs de terceros.
- **Chrome Web Store**: preparación del paquete, ficha y política de
  privacidad (`docs/privacy.html`).

## Seguridad y privacidad

- Sin llamadas de red, telemetría ni código remoto (`eval`, scripts externos).
- Los datos (volumen por dominio y preferencias) se guardan solo en
  `chrome.storage.local`.
- Permisos mínimos y justificados: `storage`, `activeTab` y
  `host_permissions: <all_urls>`, este último imprescindible para funcionar en
  cualquier sitio.
- Validación de todos los mensajes entrantes en el content script (tipos y rangos).

## Retos técnicos identificados

- Insertar procesamiento de audio en páginas ajenas sin romper su
  reproducción: los elementos no conectables se marcan y se omiten.
- Evitar consumo innecesario de CPU y advertencias de *autoplay*, creando el
  `AudioContext` de forma diferida.
- Simular sonido envolvente sobre una salida estéreo usando solo nodos nativos
  y un impulso de reverb sintético.
- Elegir el frame correcto en páginas con varios iframes.
- Forzar un idioma distinto al del navegador, algo que `chrome.i18n` no
  permite de forma nativa.

## Mejoras futuras recomendadas

- Añadir un limitador o compresor (`DynamicsCompressorNode`) antes de la salida
  para evitar saturación a ganancias altas.
- Detectar los medios *cross-origin* sin CORS para no silenciarlos al
  conectarlos al grafo.
- Corregir el relleno del slider de intensidad y el color del favicon de
  respaldo.
- Centralizar el límite de 600% en un único lugar.
- Atajos de teclado (`commands`) y ecualizador por sitio.
- Pruebas automatizadas del grafo de audio y del protocolo de mensajes.

## Estado del proyecto

- **Versión 2.0.0**, funcional: las capturas se tomaron con la extensión
  cargada en Chrome y aplicando efectos sobre audio real reproduciéndose.
- El último commit se titula *"Prepare v2.0.0 for Chrome Web Store
  submission"*. El repositorio no permite confirmar si ya está publicada en
  la Chrome Web Store.
- Rama de trabajo actual: `dev`. Según `PUBLISHING.md`, la política de
  privacidad (`docs/`) debe llegar a `main` para servirse desde una URL pública.
- No tiene pruebas automatizadas.

## Capturas

Capturas del popup real, tomadas con la extensión cargada en Chrome sobre una
página local que reproduce un audio de demostración.

1. **Vista principal**: popup con la información de la pestaña, el volumen al
   100%, el ecualizador y el modo de sonido en su estado inicial (tema oscuro).
   `presentacion/01-vista-principal.png`
2. **Funcionalidad principal**: amplificación al 350% con "Recordar volumen
   por sitio" activado.
   `presentacion/02-funcionalidad-principal.png`
3. **Funcionalidad complementaria**: preset Cine, Voice Boost y modo Virtual
   7.1 con intensidad al 80%.
   `presentacion/03-funcionalidad-complementaria.png`

## Enlaces

- Repositorio: https://github.com/alvarosiles/super-volume (`homepage_url` del manifest)
- Licencia: MIT
