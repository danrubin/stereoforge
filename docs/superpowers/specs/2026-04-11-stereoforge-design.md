# StereoForge — Design Spec
**Date:** 2026-04-11
**Status:** Approved

---

## Overview

StereoForge is a single-page HTML application (no framework, no build step, one self-contained `.html` file) for creating stereoscopic illustrations. The output is a side-by-side (SBS) stereo pair image suitable for free-viewing. Users place and style elements once; the tool renders each element into both the left and right panels simultaneously. Each element carries its own horizontal depth offset that controls perceived depth in the stereo image.

Default visual style: white elements on a black background.

---

## Architecture

### Single file
The entire application is one `index.html` with inline `<style>` and `<script>`. No external JS dependencies. Google Fonts and `queryLocalFonts()` are the only optional network/API touches.

### Two canvases
- **Preview canvas** — visible, CSS-scaled to fit the viewport, drawn every `requestAnimationFrame`.
- **Export canvas** — hidden, always at true output resolution. Drawn on demand when the user clicks Export PNG.

Both canvases are driven by the same `drawScene(canvas, scene, { showDivider })` function.

### Render loop
On each frame, the preview canvas:
1. Fills the background color.
2. Iterates all visible elements in z-order (array index 0 = back).
3. For each element, draws it twice: once at `centerX + depth/2` (left panel) and once at `centerX - depth/2` (right panel), both clipped to their respective panel halves. This produces the correct uncrossed (positive) disparity for parallel/wall-eyed free viewing — positive depth = object appears in front of the screen plane.
4. Draws the divider line last (preview only; omitted on export).

### Coordinate system
All element positions, sizes, and depth offsets are stored in output-resolution pixels. The preview canvas applies a uniform `scale` transform so all draw code thinks in output coordinates.

### Scene state
A single plain JS object is the source of truth:
```js
{
  preset: 'ig-portrait' | 'ig-square' | 'twitter' | 'facebook' | 'custom',
  outputWidth: number,
  outputHeight: number,
  bgColor: string,
  elements: Element[]   // ordered by z-index; index 0 = backmost
}
```

### Undo / Redo
An array of JSON-stringified scene snapshots. Every user action that mutates the scene pushes a snapshot before the mutation. Stack capped at 50 steps. Redo stack is cleared on any new mutation.

---

## Data Model

### Element
```js
{
  id: string,           // UUID
  type: 'rect' | 'ellipse' | 'triangle' | 'polygon' | 'line' | 'arrow' | 'text' | 'svg',

  // Transform (output-resolution pixels)
  x: number,            // center X
  y: number,            // center Y
  width: number,
  height: number,
  rotation: number,     // degrees

  // Stereo
  depth: number,        // horizontal shift in output pixels; positive = in front, negative = behind

  // Style
  fillStyle: 'solid' | 'outline' | 'both',
  fillColor: string,
  strokeColor: string,
  strokeWidth: number,

  // Polygon-specific
  sides: number,        // used when type === 'polygon'

  // Text-specific
  text: string,
  fontFamily: string,
  fontSize: number,
  fontWeight: string,   // '400' | '700' | etc.
  fontStyle: string,    // 'normal' | 'italic'
  letterSpacing: number,
  lineHeight: number,
  textAlign: string,    // 'left' | 'center' | 'right'

  // SVG-specific
  svgSource: string,    // raw sanitized SVG markup

  // Layer controls
  visible: boolean,
  locked: boolean
}
```

### Output presets
| Key | Width | Height |
|---|---|---|
| `ig-portrait` | 1080 | 1350 |
| `ig-square` | 1080 | 1080 |
| `twitter` | 1200 | 675 |
| `facebook` | 1200 | 630 |
| `custom` | user-defined | user-defined |

Each panel is `outputWidth / 2` wide. Elements are centered within a virtual full-width space; depth shifts their L and R instances horizontally.

---

## UI Layout

**Layout A — Left toolbar + Right all-in-one panel.**

### Top bar (full width, ~38px tall)
- App name / wordmark (left)
- Output size preset dropdown (center-left)
- Background color picker (center)
- Export PNG button (right, primary action)
- Save Project / Load Project buttons (right)

### Left toolbar (44px wide, full height)
Tools from top to bottom:
- Select (arrow cursor) — active tool highlighted
- Divider
- Rectangle, Ellipse, Triangle, Polygon, Line, Arrow
- Divider
- Text, SVG Import

### Center canvas
- Dark grey background (`#171717`)
- Preview canvas centered, scaled to fit available space
- Left panel and right panel visible side by side
- 1px vertical divider between panels (preview only)
- "L" / "R" labels at panel corners (preview only, faint)

### Right panel (220px wide, full height, scrollable)
Sections from top to bottom:

1. **Transform** — X, Y, W, H inputs (2×2 grid) + Rotation input
2. **Stereo Depth** — "Behind ←→ Front" labeled slider + numeric input showing value in output pixels. Range: -100 to +100, adjustable by typing outside the range.
3. **Style** — Solid / Outline / Both toggle; fill color swatch + label; stroke color swatch + label; stroke weight input
4. **Typography** *(text elements only)* — font family picker (see Font System), size, weight, italic toggle, letter spacing, line height, alignment
5. **Element actions** — Bring Forward, Send Back, Duplicate, Delete, Visibility toggle, Lock toggle
6. **Layers list** — all elements in z-order (top of list = front), selected element highlighted, drag-to-reorder, visibility eye icon per layer

When nothing is selected, sections 1–5 are hidden and only the Layers list is shown.

### Mobile layout (viewport < 768px)
- Left toolbar collapses into a bottom drawer (slide up to reveal tools)
- Right panel collapses into a bottom sheet that slides up when an element is selected
- Canvas fills the full width above the bottom chrome

---

## Font System

### Tier 1 — System fonts via Local Font Access API
Call `window.queryLocalFonts()` on load. Prompts user for permission once. Populate the font family dropdown with all returned fonts, sorted alphabetically.

If the browser does not support `queryLocalFonts()` (Firefox, Safari), show a small inline note in the Typography section:
> "Full system font access requires Chrome or Edge."
Fall back to a short hardcoded list (Arial, Georgia, Helvetica, Times New Roman, Courier New, Verdana, Trebuchet MS, Impact).

### Tier 2 — Google Fonts (curated list of ~30)
Roboto, Open Sans, Lato, Montserrat, Oswald, Playfair Display, Raleway, Merriweather, Nunito, Poppins, Source Sans Pro, Ubuntu, PT Sans, Noto Sans, Josefin Sans, Libre Baskerville, Abril Fatface, Pacifico, Lobster, Dancing Script, Bebas Neue, Cinzel, Permanent Marker, Righteous, Titan One, Archivo Black, Alfa Slab One, Fredoka One, Comfortaa, Exo 2.

When selected, inject a `<link>` tag into `<head>`. Await `document.fonts.ready` before drawing to canvas or exporting. Fails gracefully if offline — font falls back to system default.

### Tier 3 — Custom font input
A textarea in the Typography section. Accepts:
- A `<link>` tag (Adobe Fonts kit, Google Fonts, any CDN)
- A `@font-face` CSS block
- Any valid CSS font-family import string

The app strips to the injectable portion, injects it into `<head>`, awaits `document.fonts.ready`, and adds the font name to the font family field. On failure (network error, bad input), shows a brief error message and makes no change.

---

## Interaction Model

### Selection
- Click an element → select it; show 8 resize handles + 1 rotation handle
- Click empty canvas → deselect all
- Shift-click → add/remove from selection
- Rubber-band drag on empty canvas → select all elements whose bounding boxes intersect

### Handles
- **Corner handles** — drag to resize width and height; Shift constrains aspect ratio
- **Edge midpoint handles** — drag to resize one axis
- **Rotation handle** (above top-center) — drag to rotate around element center

### Multi-select behavior
- Depth slider adjusts all selected elements by the same delta
- Position/size inputs show the value only if all selected elements share it; otherwise blank
- Delete / Duplicate / layer-order actions apply to all selected elements

### Text editing
- Double-click a text element → enter inline edit mode (transparent overlay input positioned over element)
- Escape or click outside → commit edit

### SVG import
Two entry points via the toolbar SVG button:
1. **Upload** — file input accepting `.svg`; read via FileReader
2. **Paste** — modal textarea for raw SVG markup

Sanitization: strip all `<script>` tags and `on*` event attributes before storing. SVG is stored as raw string and rendered to canvas via `Image` + blob URL.

### Keyboard shortcuts
| Key | Action |
|---|---|
| `Delete` / `Backspace` | Delete selected elements |
| `Cmd/Ctrl+D` | Duplicate selected |
| `Cmd/Ctrl+Z` | Undo |
| `Cmd/Ctrl+Shift+Z` / `Cmd/Ctrl+Y` | Redo |
| `Escape` | Deselect / exit text edit |
| `Arrow keys` | Nudge 1px |
| `Shift+Arrow` | Nudge 10px |

---

## Export

### PNG export
1. Create hidden `<canvas>` at `outputWidth × outputHeight`
2. Run `drawScene(exportCanvas, scene, { showDivider: false })`
3. Await `document.fonts.ready` to ensure all loaded fonts are available to the canvas context
4. Call `exportCanvas.toDataURL('image/png')`
5. Create a temporary `<a download="stereoforge-export.png">`, trigger click, revoke

The export canvas is never shown in the DOM. Each export creates it fresh to avoid stale state.

### Project save / load
- **Save** — `JSON.stringify(scene)` → Blob → `<a download="stereoforge-project.json">` trigger
- **Load** — `<input type="file" accept=".json">` → FileReader → parse → validate shape → replace scene state → push to undo stack → re-render

---

## Visual Design

- Dark chrome: `#111` body, `#1c1c1c` panels, `#2a2a2a` secondary surfaces
- Borders: `#2a2a2a` (subtle) / `#333` (interactive)
- Accent: `#3a7bd5` (blue) for active tool, Export button, selected element highlight
- Depth slider fill: `#2a4a7f` track, `#4a9eff` handle
- Default canvas background: `#000000`
- Default element fill: `#ffffff`
- Default element stroke: `#ffffff`
- Typography in UI: system-ui / sans-serif, 9–11px for labels, 10–12px for values

---

## Constraints & Notes

- Single `.html` file — no external JS libraries, no build step
- Google Fonts and Local Font Access API are the only optional external dependencies; both fail gracefully
- Adobe Fonts support is via Tier 3 custom input — the user pastes their kit `<link>` tag; the app does not manage Adobe credentials
- The divider line between panels is visible in the preview canvas only; it does not appear in PNG exports
- Depth offset range is -100 to +100px at output resolution by default; users can type values outside this range in the numeric input
- Preview canvas scales to fit the viewport using CSS `transform: scale()` on the canvas element; all internal coordinates remain in output-resolution pixels
- Undo stack is capped at 50 steps
