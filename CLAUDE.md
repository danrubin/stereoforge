# StereoForge

Single-file HTML stereoscopic illustration creator. No framework, no build step — the entire app lives in `index.html`.

## Current Status

**All three plans written. `index.html` not yet created. Ready to execute.**

### What exists
- `docs/superpowers/specs/2026-04-11-stereoforge-design.md` — full approved design spec
- `docs/superpowers/plans/2026-04-11-stereoforge-part-a.md` — Part A: Foundation (Tasks 1–6: HTML shell, state, canvas engine, shapes, text, SVG)
- `docs/superpowers/plans/2026-04-11-stereoforge-part-b.md` — Part B: Interaction + UI (Tasks 1–6: tool selection, hit testing, drag, resize/rotate handles, right panel, layers + keyboard)
- `docs/superpowers/plans/2026-04-11-stereoforge-part-c.md` — Part C: Features + Polish (Tasks 1–5: local fonts, Google Fonts + custom, PNG export, save/load, mobile layout)

### What's next
Execute the plans in order: **Part A → Part B → Part C**. All three plans build a single `index.html` by sequentially replacing `// === PLACEHOLDER — X ===` comment markers. The order is strict — Part B depends on the DOM and JS scaffold Part A creates; Part C depends on `refreshPanel()` and other functions from Part B.

**Recommended execution approach:** Use the `superpowers:subagent-driven-development` skill. Run one task at a time, review output, proceed.

**Testing requires Chrome** (for `queryLocalFonts`, and to verify canvas rendering with DevTools).

## Key Architectural Decisions

- **Rendering:** Canvas 2D API. One preview canvas (CSS-scaled to viewport) + one hidden export canvas (full output resolution). Both driven by the same `drawScene()` function.
- **Stereo depth formula:** Left panel draws at `xOffset = depth/2`, right panel at `xOffset = outputWidth/2 - depth/2`. Positive depth = element appears in front, for parallel/wall-eyed free viewing.
- **Coordinate system:** Element `x,y` is center position within a single panel (0..outputWidth/2). All stored at output resolution.
- **No OffscreenCanvas** — output sizes are small enough that plain canvas export is imperceptible.
- **Font system:** `queryLocalFonts()` only for system fonts (no canvas fingerprint fallback). If unsupported, show inline note: "Full system font access requires Chrome or Edge." Google Fonts (curated ~30) injected on demand. Custom fonts via paste-in `<link>` / `@font-face` / kit URL.
- **UI Layout:** Left toolbar (44px) + center canvas + right all-in-one panel (220px, scrollable). Top bar has preset, BG color, Export, Save, Load.
- **Mobile:** viewport < 768px → toolbar becomes bottom drawer, right panel becomes bottom sheet.

## Plan Split

This project is split into 3 sub-plans to stay within the ~1500-line plan budget:
- **Part A** — Foundation (6 tasks): HTML/CSS shell, scene state + undo/redo, canvas rendering engine, shape rendering, text rendering, SVG rendering. Also wires: output preset dropdown, BG color picker, window resize → fitCanvas.
- **Part B** — Interaction + UI (6 tasks): tool selection + element placement, hit testing + click selection, move drag + rubber-band, resize/rotate handles, right panel (transform/depth/style/actions + undo/redo keys), layers list + typography + all keyboard shortcuts.
- **Part C** — Features + Polish (5 tasks): local font system (queryLocalFonts), Google Fonts + custom font injection, PNG export, project save/load, mobile layout.

## Execution Notes

- Each task replaces a `// === PLACEHOLDER — X added in Part Y ===` comment in `index.html` (except Part A Task 1 which creates the file).
- **Part B Task 3** replaces the `handleCanvasMouseDown` function written in Task 2 — this is an upgrade, not a placeholder replacement. The plan explains this explicitly.
- **Part B Task 4** upgrades the `// Check handles first (populated in Task 4; no-op here)` comment inside `handleCanvasMouseDown` with live handle detection. This requires that Part B Task 3's version of the function is in place first.
- Do not skip tasks or reorder within a part — each task's code depends on the previous task's placeholder structure.
