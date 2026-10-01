# BeatPoker Studio — build de prueba (estático)

Editor de plantillas de cobertura para BeatPoker. Sitio **estático**: sin build, sin
dependencias en tiempo de ejecución, sin red. Se sube tal cual.

En producción: <https://beatpoker-studio.vercel.app>

Esto es el **parche desplegable** del diseño original (`../Plantillas épicas BeatPoker/`),
no la reconstrucción profesional. Esa sigue pendiente — ver *Reconstrucción* abajo.

## Correr en local

```
npx http-server -p 4321 .
```

Abrir <http://localhost:4321>. No sirve abrir `index.html` con doble clic: las fuentes y
`localStorage` necesitan un origen `http://`.

## Desplegar

```
npx vercel --prod
```

## Qué hay dentro

| Ruta | Qué es |
|---|---|
| `index.html` | La app entera: markup + lógica (`class Component extends DCLogic`). |
| `support.js` | Runtime del Design Component. **No editar** — es generado. |
| `vendor/` | React 18.3.1 (UMD), ReactDOM, html-to-image 1.11.13. Antes venían de CDN. |
| `fonts/` | Anton + Barlow Condensed, subsets `latin` y `latin-ext`, autoalojadas. |
| `flags/` | Las 13 banderas del selector. Antes venían de `flagcdn.com`. |
| `assets/` | Logo BeatPoker (blanco y dorado), logo OSOP por defecto, foto de ejemplo. |

## Qué se arregló respecto del original

Cuatro bugs, tres de ellos invisibles hasta que exportabas.

**1. `componentDidUpdate` lanzaba en cada edición.** El runtime lo llama con un solo
argumento — `componentDidUpdate(prevProps)`, y son *props*, no *state* (ver
`support.js:914`). El código leía `prev.format` de un `undefined`, la excepción se la
tragaba el `try/catch` del runtime, y **todo lo que venía después dejaba de correr**: el
auto-ajuste solo se ejecutaba una vez, al montar. Los dos bugs siguientes estaban tapados
por este.

**2. `fitMonto` medía la caja, no el número.** El wrapper del monto es un flex item de una
columna con `align-items:stretch`, así que su `offsetWidth` vale siempre 948 (el ancho
útil) sin importar el contenido. El `<span>` con `white-space:nowrap` se desbordaba y lo
recortaba el `overflow:hidden` del artboard. Ahora se mide `scrollWidth`, que sí incluye
el desborde. Un premio largo ya no se corta.

**3. `fitPanel` no podía ver el desborde que existía para detectar.** El panel era un flex
`justify-content:flex-end`: cuando el contenido no cabe, se escapa por el borde *superior*,
y ese desbordamiento no lo cuenta `scrollHeight`. Además el límite real nunca fue el tope
del panel — el título se monta sobre el degradado de la foto **a propósito**. El límite es
el borde inferior del header de logos. Ahora el contenido va en `position:absolute`
anclado abajo (un flex item se aplasta o desborda, y en ambos casos deja de ser medible),
y se mide contra `safeTop = 46 + tamañoLogo + 24`.

**4. Las fuentes del PNG exportado dependían de la red.** Venían de Google Fonts;
`html-to-image` las descarga e incrusta al exportar. Si la red fallaba, el PNG salía con
la fuente del sistema — en silencio, sin error. Ahora van autoalojadas y el export espera
a `document.fonts.ready` antes de rasterizar.

> Al autoalojarlas hay una trampa: en el CSS de Google el comentario `/* latin */`
> **precede** a su bloque `@font-face`. Si lo buscas *dentro* del bloque te llevas el
> subset equivocado y te quedas sin `U+0000-00FF` — o sea sin A-Z ni tildes. Los archivos
> se llamarían bien y pesarían lo esperado, y aun así el navegador usaría fallback.

## Además

- **Autoguardado** en `localStorage` (clave `beatpoker-studio-v1`), con *debounce* de 500 ms.
  Si la foto en dataURL excede la cuota, guarda todo lo demás y descarta las imágenes.
  Botón **Restablecer** en la barra superior.
- **Plantilla Resultados**: un `<textarea>` con una línea por puesto (`Nombre | Premio`),
  máximo 10. El número de puesto se pone solo; oro / plata / bronce en los tres primeros.
- **Cobertura en vivo**: los nombres de los 6 recuadros del mini resumen son campos
  (`labelFichas`, `labelNivel`, …), no texto fijo. Se corrigieron dos que estaban mal:
  *Nivel de jugadores* → **Jugadores** (el valor es `42 de 180`, no un nivel) y
  *Nivel de la mesa* → **Nivel** (el valor ya dice `Nivel 18`). Vaciar el **valor** sigue
  ocultando el recuadro; el nombre no influye.
- **Banderas locales** + opción *Sin bandera*.
- **Export**: espera las fuentes, hace una pasada de calentamiento (la primera llamada a
  `toPng` incrusta imágenes y fuentes; la segunda es la que sale completa), deshabilita el
  botón mientras corre y nombra el archivo con el jugador.
- Se eliminó código muerto del diseño anterior con sponsor (`sponsorUrl`, `presentado`,
  `showName`, `hasSponsor`, `noSponsor`).

## Limitaciones conocidas

- **3 × 404 al cargar** (`{{ photoUrl }}`, `{{ eventLogoUrl }}`, `{{ flagUrl }}`). El
  navegador parsea el markup crudo de `<x-dc>` antes de que React lo reemplace, y pide esas
  URLs literales. Es cosmético: las imágenes están ocultas y la app no las usa. Desaparece
  con la reconstrucción.
- **La flecha `→`** cae a una fuente de fallback: ningún subset de Barlow Condensed cubre
  `U+2192`. Ya pasaba con Google Fonts; no es una regresión.
- **Ancho mínimo 1180 px.** Abajo de eso aparece scroll horizontal. Es una herramienta de
  escritorio.
- El PNG de historia pesa ~8 MB (2160×3840, `pixelRatio: 2`). Instagram lo recomprime.

## Reconstrucción (pendiente)

React + Vite + TypeScript, como pide el README del diseño original. Un `<Artboard>` por
plantilla, `<EditorPanel>`, `<Gallery>`; `fitMonto`/`fitPanel` como hooks con
`ResizeObserver`. Al portar, **conservar las cuatro correcciones de arriba** — sobre todo
la semántica de `safeTop`: el contenido debe poder montarse sobre la foto y encoger solo
cuando vaya a chocar con los logos.
