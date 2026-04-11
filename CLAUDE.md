# StereoForge

Single-file HTML stereoscopic illustration creator. No framework, no build step — the entire app lives in `index.html`.

## Current Status

**Design phase complete. Part A implementation plan written. `index.html` not yet created.**

### What exists
- `docs/superpowers/specs/2026-04-11-stereoforge-design.md` — full approved design spec
- `docs/superpowers/plans/2026-04-11-stereoforge-part-a.md` — Part A implementation plan (Tasks 1–6: HTML shell, state, canvas engine, shapes, text, SVG)

### What's next
1. Write Part B plan (Interaction + UI): tool selection, hit testing, drag, resize/rotate handles, right panel wiring, layers list
2. Write Part C plan (Features): font system, PNG export, save/load, keyboard shortcuts, presets, mobile layout
3. Execute the plans (execution approach TBD — subagent-driven or inline)

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
- **Part A** — Foundation (done)
- **Part B** — Interaction + UI (6 tasks): tool selection + element placement, hit testing + drag, resize/rotate handles, right panel transform/depth/style, right panel typography, layers list
- **Part C** — Features + Polish (6 tasks): font system, PNG export, save/load, keyboard shortcuts, output presets + BG picker, mobile layout
