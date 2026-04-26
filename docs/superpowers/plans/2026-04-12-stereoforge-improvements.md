# StereoForge Improvements Plan
**Date:** 2026-04-12
**File:** `index.html` (single-file app, no build step)

## Overview

Four targeted improvements to the existing StereoForge app. All changes go into `index.html`. Tasks are ordered so later tasks don't depend on earlier ones (except Task 1 which is foundational for selection).

---

## Task 1 — Depth-aware selection, bounding box, and handles

**Goal:** Fix selection, bounding box outlines, handles, and rubber-band so they all account for stereo depth offset. Also enable clicking elements on the right panel.

### Background

- Left panel draws element at `x = el.x + el.depth/2`
- Right panel draws element at `x = outputWidth/2 + (el.x - el.depth/2)`  
  (equivalently: `el.x` relative to right panel start = `el.x - el.depth/2`)
- `el.x` is panel-relative center (0..`outputWidth/2`)
- `worldToLocal` / `localToWorld` currently use `el.x` as the world origin — no depth offset
- `drawSelectionOutlines` translates to `(el.x, el.y)` — misaligned when depth ≠ 0
- `hitTest` only tests left-panel position — right-panel clicks miss

### Steps

**Step 1.1 — Update `localToWorld` and `worldToLocal` to accept `xOffset`**

Find:
```js
function localToWorld(el, lx, ly) {
  const cos = Math.cos(el.rotation), sin = Math.sin(el.rotation);
  return {
    x: el.x + cos * lx - sin * ly,
    y: el.y + sin * lx + cos * ly,
  };
}
function worldToLocal(el, wx, wy) {
  const dx = wx - el.x;
  const dy = wy - el.y;
  const cos = Math.cos(el.rotation), sin = Math.sin(el.rotation);
  return {
    x:  cos * dx + sin * dy,
    y: -sin * dx + cos * dy,
  };
}
```

Replace with:
```js
function localToWorld(el, lx, ly, xOffset) {
  if (xOffset === undefined) xOffset = el.depth / 2;
  const cos = Math.cos(el.rotation), sin = Math.sin(el.rotation);
  return {
    x: el.x + xOffset + cos * lx - sin * ly,
    y: el.y + sin * lx + cos * ly,
  };
}
function worldToLocal(el, wx, wy, xOffset) {
  if (xOffset === undefined) xOffset = el.depth / 2;
  const dx = wx - (el.x + xOffset);
  const dy = wy - el.y;
  const cos = Math.cos(el.rotation), sin = Math.sin(el.rotation);
  return {
    x:  cos * dx + sin * dy,
    y: -sin * dx + cos * dy,
  };
}
```

Note: `xOffset` defaults to `el.depth/2` (left panel) when not supplied, preserving all existing callers.

**Step 1.2 — Update `hitTestElement` and `hitTest` for both panels + depth**

Find:
```js
function hitTestElement(el, px, py) {
  const local = worldToLocal(el, px, py);
  return Math.abs(local.x) <= el.width/2 + 2 && Math.abs(local.y) <= el.height/2 + 2;
}
function hitTest(px, py) {
  for (let i = scene.elements.length-1; i >= 0; i--) {
    const el = scene.elements[i];
    if (!el.visible || el.locked) continue;
    if (hitTestElement(el, px, py)) return el.id;
  }
  return null;
}
```

Replace with:
```js
function hitTestElement(el, px, py, xOffset) {
  const local = worldToLocal(el, px, py, xOffset);
  return Math.abs(local.x) <= el.width/2 + 2 && Math.abs(local.y) <= el.height/2 + 2;
}
function hitTest(px, py) {
  const halfW = scene.outputWidth / 2;
  // Determine which panel was clicked and compute local xOffset for that panel
  const inRight = px >= halfW;
  for (let i = scene.elements.length-1; i >= 0; i--) {
    const el = scene.elements[i];
    if (!el.visible || el.locked) continue;
    if (inRight) {
      // Right panel: element draws at halfW + el.x - el.depth/2
      // so xOffset relative to el.x = halfW - el.depth/2
      if (hitTestElement(el, px - halfW, py, -el.depth / 2)) return el.id;
    } else {
      // Left panel: element draws at el.x + el.depth/2
      if (hitTestElement(el, px, py, el.depth / 2)) return el.id;
    }
  }
  return null;
}
```

Note: For the right panel click, we shift `px` by `-halfW` so coordinates are panel-relative, then use xOffset = `-el.depth/2` to match the right-panel draw position.

**Step 1.3 — Fix `hitTestHandles` to reject right-panel clicks and use depth**

Find the `hitTestHandles` function. It will look like:
```js
function hitTestHandles(px, py) {
```

Add a right-panel rejection guard at the top:
```js
function hitTestHandles(px, py) {
  if (px >= scene.outputWidth / 2) return null;
```

Then ensure any calls to `getHandleWorld` or `localToWorld` inside `hitTestHandles` use the left-panel xOffset (`el.depth / 2`). Since `localToWorld` now defaults to `el.depth/2`, existing calls inside `getHandleWorld` will be correct automatically.

**Step 1.4 — Fix `drawSelectionOutlines` to draw on both panels with depth**

Find `drawSelectionOutlines`. It currently translates to `(el.x, el.y)`. The function draws a dashed rectangle around selected elements.

Replace the single-panel drawing loop with a two-panel loop:

```js
function drawSelectionOutlines(ctx) {
  const halfW = scene.outputWidth / 2;
  scene.elements.forEach(el => {
    if (!el.selected) return;
    const panels = [
      el.depth / 2,              // left panel xOffset
      halfW - el.depth / 2,      // right panel xOffset (draw pos = el.x + xOffset, centered at halfW + el.x - el.depth/2)
    ];
    panels.forEach((xOffset, panelIdx) => {
      ctx.save();
      ctx.translate(el.x + xOffset, el.y);
      ctx.rotate(el.rotation);
      ctx.strokeStyle = '#2196F3';
      ctx.lineWidth = 1.5 / (previewCanvas.width / previewCanvas.clientWidth);
      ctx.setLineDash([4, 3]);
      ctx.strokeRect(-el.width/2, -el.height/2, el.width, el.height);
      ctx.setLineDash([]);
      ctx.restore();
    });
  });
}
```

Note: The line width scaling factor may already be in the original — keep whatever is there.

**Step 1.5 — Fix `drawHandles` to use depth offset**

Find `drawHandles`. It calls `getHandleWorld(el, h)` and `getRotHandleWorld(el)`. These call `localToWorld`. Since `localToWorld` now defaults to `el.depth/2`, handles will automatically draw at the correct left-panel position.

Verify there is no hardcoded `el.x` origin inside `getHandleWorld` or `getRotHandleWorld`. If they delegate to `localToWorld`, they're already fixed by Step 1.1. If they compute positions manually, update them to add `el.depth/2`.

**Step 1.6 — Fix rubber-band selection filter to account for depth**

Find the rubber-band selection filter. It looks like:
```js
.filter(el => {
  const hw = el.width/2, hh = el.height/2;
  return el.x - hw < rx + rw && el.x + hw > rx &&
         el.y - hh < ry + rh && el.y + hh > ry;
})
```

Replace with:
```js
.filter(el => {
  // Compare against element's left-panel visual position (el.x + el.depth/2)
  const cx = el.x + el.depth / 2;
  const hw = el.width / 2, hh = el.height / 2;
  return cx - hw < rx + rw && cx + hw > rx &&
         el.y - hh < ry + rh && el.y + hh > ry;
})
```

### Verification
- Start dev server, open in Chrome
- Place a shape on canvas, change its depth slider to a non-zero value (e.g. 80)
- Verify: selection bounding box aligns with the element's visual position in the left panel
- Click the element in the right panel — verify it becomes selected
- With depth=0 and depth≠0, verify rubber-band selection selects elements it visually covers
- Verify handles appear at correct corners and can be dragged

---

## Task 2 — Update output presets

**Goal:** Replace the four current presets (IG Portrait, IG Square, Twitter, Facebook) with five new presets.

### New presets
| Key | Label | Width | Height |
|-----|-------|-------|--------|
| `ig-4-5` | IG 4:5 | 1080 | 1350 |
| `square` | Square | 1080 | 1080 |
| `story` | Story / Reel / TikTok | 1080 | 1920 |
| `hd-landscape` | HD 16:9 | 1920 | 1080 |
| `4k-landscape` | 4K 16:9 | 3840 | 2160 |

### Steps

**Step 2.1 — Update the PRESETS object**

Find:
```js
const PRESETS = {
  'ig-portrait': { w: 1080, h: 1350 },
  'ig-square':   { w: 1080, h: 1080 },
  'twitter':     { w: 1200, h: 675  },
  'facebook':    { w: 1200, h: 630  },
};
```

Replace with:
```js
const PRESETS = {
  'ig-4-5':        { w: 1080, h: 1350 },
  'square':        { w: 1080, h: 1080 },
  'story':         { w: 1080, h: 1920 },
  'hd-landscape':  { w: 1920, h: 1080 },
  '4k-landscape':  { w: 3840, h: 2160 },
};
```

**Step 2.2 — Update the `<select>` HTML options**

Find the `<select id="preset-select">` element. It will contain four `<option>` elements with values `ig-portrait`, `ig-square`, `twitter`, `facebook`.

Replace those options with:
```html
<option value="ig-4-5">IG 4:5</option>
<option value="square">Square</option>
<option value="story">Story / Reel / TikTok</option>
<option value="hd-landscape">HD 16:9</option>
<option value="4k-landscape">4K 16:9</option>
```

The first option (`ig-4-5`) should be selected by default (keep it first in the list).

**Step 2.3 — Verify DOMContentLoaded preset initialization**

Find where the preset select's `change` handler fires on load (typically the DOMContentLoaded handler calls `presetSelect.dispatchEvent(new Event('change'))` or reads `presetSelect.value`). Confirm the default selected value matches the new first preset key (`ig-4-5`). If the value is hardcoded to an old key, update it.

### Verification
- Reload page, verify dropdown shows "IG 4:5" as default
- Switch to each preset and verify canvas resets to correct dimensions
- Verify no console errors about missing preset keys

---

## Task 3 — Add opacity control to Style section

**Goal:** Add a per-element opacity slider (0–100%) to the Style section of the right panel, stored as `el.opacity` (0.0–1.0).

### Steps

**Step 3.1 — Add `opacity` to `createElement`**

Find the `createElement` function. It returns an object with properties like `x`, `y`, `width`, `height`, `rotation`, `depth`, etc.

Add `opacity: 1` to the returned object alongside the other style properties (near `fill`, `stroke`, `strokeWeight`, etc.).

**Step 3.2 — Apply `el.opacity` in `drawElement`**

Find `drawElement(ctx, el, xOffset)`. At the very start of the function body (before any drawing), add:
```js
ctx.save();
ctx.globalAlpha = el.opacity ?? 1;
```

At the very end of the function body (after all drawing), add:
```js
ctx.restore();
```

Use `el.opacity ?? 1` for backward compatibility with saved projects that don't have the field.

**Step 3.3 — Add opacity row to `#section-style` HTML**

Find the `<div id="section-style">` in the HTML. It contains rows for fill, stroke color, and stroke weight.

Add an opacity row after the stroke weight row:
```html
<div class="prop-row">
  <label>Opacity</label>
  <input type="range" id="el-opacity" min="0" max="100" step="1" value="100">
  <span id="el-opacity-val">100%</span>
</div>
```

**Step 3.4 — Wire opacity in `refreshPanel`**

Find `refreshPanel`. It sets values for all the style inputs (fill color, stroke, strokeWeight, etc.).

Add alongside the other style property assignments:
```js
const opacityInput = document.getElementById('el-opacity');
const opacityVal   = document.getElementById('el-opacity-val');
if (opacityInput && el) {
  const pct = Math.round((el.opacity ?? 1) * 100);
  opacityInput.value = pct;
  opacityVal.textContent = pct + '%';
}
```

**Step 3.5 — Add `input` event handler for opacity slider**

Find where the other style input event handlers are wired (fill color change, stroke weight input, etc.).

Add:
```js
document.getElementById('el-opacity').addEventListener('input', e => {
  const el = getSelectedElement();
  if (!el) return;
  pushUndo();
  el.opacity = parseInt(e.target.value) / 100;
  document.getElementById('el-opacity-val').textContent = e.target.value + '%';
  drawScene();
});
```

### Verification
- Select an element, verify opacity slider appears in Style section showing 100%
- Drag slider to 50% — element on canvas should become semi-transparent
- Deselect and reselect — verify slider still shows 50%
- Save project, reload, load project — verify opacity is preserved
- Verify that existing saved projects without `opacity` field render at full opacity (no crash)

---

## Task 4 — Empty canvas feedback + export button disabled state

**Goal:** Show contextual messages in the right panel when canvas is empty vs. when nothing is selected. Disable the Export button when the canvas has no elements.

### Steps

**Step 4.1 — Update `#no-selection` HTML to support two states**

Find the `<div id="no-selection">` element. It currently contains one static string.

Replace with two child spans (or paragraphs), one for each state:
```html
<div id="no-selection">
  <p id="msg-empty-canvas">Add an item to the canvas to get started.</p>
  <p id="msg-no-selection" style="display:none">Select an element to edit its properties.</p>
</div>
```

**Step 4.2 — Update `refreshPanel` to show the correct message**

Find where `refreshPanel` shows/hides `#no-selection`. It currently does something like:
```js
noSelection.style.display = selEl ? 'none' : 'block';
```

Expand this to handle three states:
```js
const noSel = document.getElementById('no-selection');
const msgEmpty = document.getElementById('msg-empty-canvas');
const msgNone  = document.getElementById('msg-no-selection');
if (selEl) {
  noSel.style.display = 'none';
  // (sections are shown below)
} else {
  noSel.style.display = 'block';
  const isEmpty = scene.elements.length === 0;
  msgEmpty.style.display = isEmpty ? 'block' : 'none';
  msgNone.style.display  = isEmpty ? 'none'  : 'block';
}
```

**Step 4.3 — Disable/enable Export button in `refreshPanel`**

Find the export button reference in the JS (or query it). Add to `refreshPanel`:
```js
const btnExport = document.getElementById('btn-export');
if (btnExport) {
  btnExport.disabled = scene.elements.length === 0;
}
```

**Step 4.4 — Set initial export button state in DOMContentLoaded**

Find the `DOMContentLoaded` handler. Add:
```js
document.getElementById('btn-export').disabled = scene.elements.length === 0;
```

This handles the initial page load state (empty canvas → button disabled).

**Step 4.5 — Optional: add CSS for disabled export button**

Find the `#btn-export` CSS rule (or the button's style class). Ensure the disabled state has visible feedback. If a disabled style already exists (opacity or cursor change on `button:disabled`), this step is a no-op. Otherwise add:
```css
#btn-export:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}
```

### Verification
- Load page with empty canvas — Export button is disabled, right panel shows "Add an item to the canvas to get started."
- Add an element — Export button enables, right panel shows element properties
- Deselect element (click empty canvas) — right panel shows "Select an element to edit its properties."
- Delete last element — Export button disables again, right panel shows "Add an item to the canvas to get started."
- Export button click while disabled does nothing

---

## Execution order

Tasks are largely independent. Recommended order:
1. Task 2 (presets) — simplest, no risk
2. Task 4 (empty canvas + export disabled) — pure UI, no rendering impact
3. Task 3 (opacity) — adds new field + rendering path
4. Task 1 (depth-aware selection) — most complex, touches hit testing + rendering
