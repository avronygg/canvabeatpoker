# BeatPoker Studio — Editor de plantillas (handoff)

> ⚠️ **Antes de portar nada, lee `../beatpoker-studio/README.md`.**
> El auto-ajuste que se describe abajo (§4) **nunca funcionó** en este `.dc.html`:
> `componentDidUpdate` lanzaba en cada edición y abortaba `fitMonto`/`fitPanel`; además
> `fitMonto` medía la caja en vez del número y `fitPanel` no podía ver un desborde de flex
> `flex-end`. Los cuatro bugs (esos tres más las fuentes remotas del export) están
> corregidos y verificados en `../beatpoker-studio/`, que es el build desplegado.
> Portar este archivo tal cual los reintroduce.

Editor visual para que un no-diseñador de **BeatPoker** genere historias/posts de cobertura
de poker (OSOP y otros torneos): elige una plantilla, edita textos, tamaños, sube foto y
logos, y exporta un **PNG en alta calidad** listo para Instagram (Historia 9:16 y Post 4:5).

---

## 1. Qué son estos archivos (LÉEME ANTES DE TOCAR)

- **`BeatPoker Studio.dc.html`** → es la app real (editor + plantillas). Está escrita como
  un **"Design Component" (DC)**: un formato propietario del entorno donde se creó
  (plantilla HTML con huecos `{{ ... }}` + una clase de lógica tipo React). **Depende de
  `support.js`** (runtime incluido en el proyecto). Fuera de ese runtime el `.dc.html` NO
  corre por sí solo.
- **`support.js`** → runtime del DC (NO editar, NO es de producción).
- **`BeatPoker Studio.html`** → versión **empaquetada autónoma** (funciona offline con doble
  clic; imágenes incrustadas). Es un build de salida, no la fuente.
- **`BeatPoker Studio Export.dc.html`** → copia usada solo para generar el empaquetado.
- **`assets/`** → imágenes (logos, foto y bandera por defecto).

> **Importante:** el objetivo del handoff es **re-implementar este diseño de forma
> profesional** (recomendado: **React + Vite + TypeScript**) tomando este DC como
> **referencia de diseño y de comportamiento**, NO copiar el `.dc.html` tal cual. Toda la
> lógica, medidas, tokens y reglas están documentadas abajo para reconstruirlo 1:1.

**Fidelidad:** alta (hi-fi). Colores, tipografías, medidas y layout son finales.

---

## 2. Cómo correr / ver lo actual

- Ver el diseño empaquetado: abrir **`BeatPoker Studio.html`** en el navegador (doble clic).
- La fuente `.dc.html` solo se renderiza dentro del entorno original (no es Vite/CRA).

---

## 3. Arquitectura (para reconstruir en React)

La app es de **una sola vista** con 3 zonas:

```
┌──────────┬───────────────────────────┬──────────────┐
│ Galería  │  Lienzo (artboard escalado)│  Panel Editar │
│ (izq)    │  + toolbar (formato/export)│  (der)        │
└──────────┴───────────────────────────┴──────────────┘
```

- **Galería (izq):** lista de plantillas. Al elegir una, se cargan sus *presets*.
- **Lienzo (centro):** el "artboard" a tamaño real (1080×1920 / 1080×1350) **escalado**
  con `transform: scale()` para caber en pantalla (se recalcula con `ResizeObserver`).
  Toolbar arriba: toggle **Historia 9:16 / Post 4:5**, zoom %, botón **Exportar PNG**.
- **Panel Editar (der):** controles que dependen de la plantilla activa (ver §6).

### Estado (state) — modelo de datos

```ts
type Template = 'ganador' | 'chipleader' | 'proximos' | 'live';
type Format   = 'story' | 'post';

state = {
  template: Template,
  format: Format,
  scale: number,            // escala visual del artboard (auto)
  flagCode: string,         // 'cl', 'ar', ... (código país flagcdn)
  photoUrl: string,         // dataURL o ruta
  flagUrl: string,          // https://flagcdn.com/w320/<code>.png
  eventLogoUrl: string,     // logo del evento (der.), editable
  photoPosY: number,        // 0–100  (encuadre vertical de la foto)
  photoZoom: number,        // 100–220 (zoom de la foto, %)
  montoFit: number,         // auto (0–1) para que el número no se corte
  panelFit: number,         // auto (0–1) para que el contenido no se desborde
  sizes: { titulo, monto, nombre, logoSize },  // tamaños base editables
  data: { ...campos de texto... }              // ver abajo
}
```

### `data` (superset de campos de texto, todos strings)

Comunes: `kicker`, `titulo` (`\n` = salto de línea), `web`.
Persona (Ganador/Chip Leader): `montoLabel`, `monto`, `moneda`, `rank`, `nombre`, `bajada`.
Próximos eventos: `fecha`, `montoLabel`, `monto`, `moneda`, `sub1val`, `sub1label`,
`sub2val`, `sub2label`, `nombre` (sede), `bajada` (CTA).
Live / Cobertura: `statFichas`, `statNivel`, `statJugadores`, `statBuyin`,
`statGarantizado`, `statFecha`.

> Existe un `allDefaults` con TODOS los campos. Al cambiar de plantilla se hace
> `data = { ...allDefaults, ...preset.data }` para que nunca queden campos indefinidos.
> Cada plantilla solo **renderiza** los campos que le corresponden.

### Presets por plantilla
Cada plantilla trae `data` (valores de ejemplo reales) + `sizes`. Ejemplos:
- **ganador**: título "Campeón / del Main Event", monto `$18.500.000` `CLP`, rank vacío.
- **chipleader**: título "Chip / Leader", `Stack de fichas 2.450.000 PTS`, rank `1°`.
- **proximos**: "OSOP / Elite", `$100.000.000` garantizados, 2 sub-montos, sede, CTA.
- **live**: "Así / va la Mesa", 6 stats (fichas, nivel mesa, nivel jugadores, buy-in,
  garantizado, fecha).

---

## 4. Formatos y auto-ajuste (CLAVE — no romper)

| Formato | Tamaño     | `photoH` | `panelTop` | `gap` |
|---------|------------|----------|------------|-------|
| story   | 1080×1920  | 1360     | 1120       | 34    |
| post    | 1080×1350  | 800      | 660        | 20    |

- **Escalado de texto por formato:** en post los textos se reducen con `fmtK = 0.82`
  (`sizeMostrado = round(sizeBase * fmtK)`). En historia `fmtK = 1`. Los sliders editan el
  **tamaño base**; ambos formatos escalan desde ahí.
- **`fitMonto` (auto):** mide el ancho real del número y aplica `scale` para que **nunca se
  corte** (máx ancho útil = 948px = 1080 − 2×66 de padding).
- **`fitPanel` (auto):** mide la altura del contenido vs. el espacio disponible y aplica
  `scale` (origen abajo) para que **nada se desborde ni se encime** en ningún formato.
  Requisito: cada bloque hijo del panel debe llevar `flex: none` (si no, flexbox los
  encoge y el auto-ajuste no detecta el desborde).

---

## 5. Reglas de marca (respetar en la reconstrucción)

- **Fondo:** negro cálido `#0d0b07` con glows radiales (dorado arriba, rojo-fieltro abajo),
  textura de puntos sutil, marco/hairlines dorados. Base uniforme para que la foto **se
  funda con un degradado hacia abajo, sin línea de corte**.
- **Foto (protagonista):** ocupa la zona superior; degradado que la disuelve hacia el fondo
  (mismo color final `#0d0b07` → sin costura). Editable + **encuadre** (subir/bajar + zoom).
- **Logo IZQUIERDA = BeatPoker (blanco), fijo, arriba-izquierda.** Nunca cambia.
- **Logo DERECHA = logo del evento/serie (OSOP), arriba-derecha, en un recuadro tipo
  "sello".** Es el **editable** (subir otro evento). Tamaño del recuadro ajustable con slider
  "Tamaño logos".
- **Footer:** logo BeatPoker (dorado) + **Beatpoker.cl** en una barra tipo buscador
  (editable). (Se quitó "Presentado por / sponsor".)
- **Cobertura en vivo:** badge "EN VIVO" (rojo, punto pulsante), mini-resumen de stats, y un
  **recuadro reservado "Espacio para el enlace"** donde en IG se pega el sticker de link.

---

## 6. Edición (controles del panel)

- **Imágenes y logos:** Cambiar foto (upload) · Subir/bajar + Zoom (encuadre) · Logo del
  evento (upload) · Tamaño logos (slider) · Bandera (select, solo Persona).
- **Textos:** campos según plantilla + sliders de tamaño (título, número, nombre).
- **Subida de imágenes:** `FileReader.readAsDataURL` → se guarda el dataURL en estado (se
  incrusta en el PNG exportado; funciona offline).
- **Bandera:** `https://flagcdn.com/w320/<code>.png` (requiere internet salvo la de Chile,
  que es local). Para offline total, incluir un set de banderas propio.

---

## 7. Exportar (alta calidad)

- Usa **html-to-image** (`toPng`) sobre el nodo del artboard con:
  `width/height` reales (1080×1920 o 1080×1350), `pixelRatio: 2` (→ 2160×3840 / 2160×2700),
  y `style: { transform: 'none' }` para exportar a tamaño real (ignora el escalado visual).
- En un rebuild React puedes seguir con `html-to-image`/`dom-to-image-more`, o renderizar a
  `<canvas>` para máxima fidelidad. Cargar las fuentes antes de exportar.

---

## 8. Design tokens

```
Colores
  bg base        #0d0b07     (fondo)
  panel UI       #17130c / #0f0c07
  dorado         #D4A017     (marca)
  dorado claro   #F5D879     (highlights/gradientes)
  dorado oscuro  #a97a12
  crema texto    #e8e0cf / #c9bfa6
  gris apagado   #8f866f / #7a7259
  rojo fieltro   #c0341f     (acento / EN VIVO)
  blanco         #ffffff

Tipografías (Google Fonts)
  Display / números:  Anton (400)
  Texto / UI:         Barlow Condensed (400–800)
  Wordmark logo:      (imagen PNG, no fuente)

Gradiente dorado (números/monto)
  linear-gradient(180deg,#FBEBA6,#F5D879 32%,#D4A017 62%,#a97a12)

Radios: 6–18px · Marco artboard: inset 0 0 0 2px rgba(212,160,23,.35)
```

---

## 9. Assets (en `assets/`)

Usados por la app:
- `beatpoker_white_logo.png` — BeatPoker blanco (header izq., FIJO).
- `beatpoker_logo.png` — BeatPoker dorado (sidebar + footer).
- `osop_original.png` — logo del evento por defecto (der., editable).
- `jc_photo.jpg` — foto por defecto (ejemplo).
- `flag_chile.png` — bandera por defecto.

Intermedios (se pueden borrar): `osop_final.png`, `osop_clean.png`, `osop_logo*.png`,
`player_demo.png`, `src_chip.png`, `src_cal.png`, `beatpoker_gold.png`.

> Los logos de BeatPoker y OSOP son marca registrada de sus dueños; usarlos solo para
> BeatPoker/OSOP.

---

## 10. Plantillas: estado

- ✅ **Ganador** (Historia + Post)
- ✅ **Chip Leader** (Historia + Post)
- ✅ **Próximos eventos** (Historia + Post)
- ✅ **Cobertura en vivo / Live blog** (Historia + Post) — mini-resumen + espacio de enlace
- ⏳ **Resultados / posiciones** — PENDIENTE (layout de lista/podio de top posiciones).

---

## 11. Recomendaciones para el rebuild profesional

1. **Stack:** React + Vite + TypeScript. Un componente `<Artboard>` por plantilla
   (`GanadorArtboard`, `ProximosArtboard`, `LiveArtboard`) + `<EditorPanel>` + `<Gallery>`.
2. **Estado:** `zustand` o `useReducer`. Mantener `template`, `format`, `data`, `sizes`,
   `photo/flag/eventLogo`, `photoPosY/Zoom`.
3. **Auto-ajuste:** portar `fitMonto` y `fitPanel` como hooks (`useFitToWidth`,
   `useFitToHeight`) con `ResizeObserver`.
4. **Export:** `html-to-image` (o canvas). `pixelRatio 2`, `transform:none`, fuentes listas.
5. **Persistencia:** guardar borradores en `localStorage`/backend; permitir varias banderas
   locales para export offline.
6. **No romper:** las reglas de logos (§5), el degradado sin costura, el clipping del monto
   (line-height/padding en texto con `background-clip:text`), y el escalado historia/post.

---

## 12. Archivos del diseño

- `BeatPoker Studio.dc.html` — app fuente (referencia principal).
- `support.js` — runtime del DC (referencia, no producción).
- `BeatPoker Studio.html` — build autónomo (para ver el resultado).
- `assets/` — imágenes.
