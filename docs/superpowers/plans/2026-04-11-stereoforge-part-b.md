# StereoForge — Implementation Plan (Part B: Interaction + UI)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire complete canvas interaction (tool selection, element placement, hit testing, drag, resize/rotate handles) and all right-panel controls (transform, depth, style, typography, layers, keyboard shortcuts) to produce a fully interactive editor.

**Architecture:** All interaction code is added to `index.html`, replacing `// === PLACEHOLDER — interaction added in Part B ===`. A `dragState` object drives the mouse interaction state machine. A central `refreshPanel()` function syncs all right-panel inputs from the current selection after every state change.

**Tech Stack:** Vanilla JS, Canvas 2D API, DOM events. No dependencies.

---

## File Structure

- Modify: `index.html` — replace `// === PLACEHOLDER — interaction added in Part B ===` across six sequential edits (one per task)

---

### Task 1: Tool Selection + Element Placement

Wire toolbar buttons to `activeTool`. Implement click-to-place for all shape/text tools. Open SVG modal for the SVG tool.

**Files:**
- Modify: `index.html` — replace `// === PLACEHOLDER — interaction added in Part B ===`

- [ ] **Step 1: Replace `// === PLACEHOLDER — interaction added in Part B ===` with the interaction section opening and tool wiring**

```js
// === INTERACTION ===

// --- Tool wiring ---
document.querySelectorAll('.tool-btn[data-tool]').forEach(btn => {
  btn.addEventListener('click', () => {
    const tool = btn.dataset.tool;
    if (tool === 'svg') { openSVGModal(); return; }
    activeTool = tool;
    document.querySelectorAll('.tool-btn').forEach(b => b.classList.remove('active'));
    btn.classList.add('active');
    updateCanvasCursor();
  });
});

function setActiveTool(tool) {
  activeTool = tool;
  document.querySelectorAll('.tool-btn').forEach(b => b.classList.remove('active'));
  const btn = document.querySelector(`.tool-btn[data-tool="${tool}"]`);
  if (btn) btn.classList.add('active');
  updateCanvasCursor();
}

function updateCanvasCursor() {
  const canvas = document.getElementById('preview-canvas');
  if (activeTool === 'select') {
    canvas.style.cursor = 'default';
  } else {
    canvas.style.cursor = 'crosshair';
  }
}

// --- Element placement on canvas click ---
function placeElement(type, cx, cy) {
  pushUndo();
  const half = Math.min(scene.outputWidth, scene.outputHeight) / 8;
  const defaults = { x: cx, y: cy, width: half, height: half };
  if (type === 'line' || type === 'arrow') {
    defaults.width = half * 2;
    defaults.height = 0;
  }
  if (type === 'text') {
    defaults.width = half * 2;
    defaults.height = half / 2;
  }
  if (type === 'polygon') {
    const sidesStr = window.prompt('Number of sides (3–12):', '5');
    const sides = parseInt(sidesStr);
    if (!sides || sides < 3 || sides > 12) return;
    defaults.sides = sides;
  }
  const el = createElement(type, defaults);
  addElement(el);
  selectedIds = [el.id];
  setActiveTool('select');
  refreshPanel();
}

// --- SVG modal ---
function openSVGModal() {
  document.getElementById('svg-modal').classList.remove('hidden');
}
function closeSVGModal() {
  document.getElementById('svg-modal').classList.add('hidden');
  document.getElementById('svg-paste-area').value = '';
}

document.getElementById('btn-svg-cancel').addEventListener('click', closeSVGModal);

document.getElementById('btn-svg-upload').addEventListener('click', () => {
  document.getElementById('svg-file-input').click();
});

document.getElementById('svg-file-input').addEventListener('change', e => {
  const file = e.target.files[0];
  if (!file) return;
  const reader = new FileReader();
  reader.onload = ev => {
    importSVGString(ev.target.result);
    closeSVGModal();
  };
  reader.readAsText(file);
  e.target.value = '';
});

document.getElementById('btn-svg-import').addEventListener('click', () => {
  const raw = document.getElementById('svg-paste-area').value.trim();
  if (!raw) return;
  importSVGString(raw);
  closeSVGModal();
});

function importSVGString(raw) {
  const clean = sanitizeSVG(raw);
  if (!clean) return;
  pushUndo();
  const el = createElement('svg', {
    x: scene.outputWidth / 2,
    y: scene.outputHeight / 2,
    width: Math.min(scene.outputWidth, scene.outputHeight) / 4,
    height: Math.min(scene.outputWidth, scene.outputHeight) / 4,
    svgSource: clean,
  });
  addElement(el);
  selectedIds = [el.id];
  setActiveTool('select');
  refreshPanel();
}

// === PLACEHOLDER — hit testing added in Task 2 ===
// === PLACEHOLDER — drag added in Task 3 ===
// === PLACEHOLDER — handles added in Task 4 ===
// === PLACEHOLDER — panel wiring added in Task 5 ===
// === PLACEHOLDER — layers + keyboard added in Task 6 ===
```

- [ ] **Step 2: Verify tool selection in browser**

Open `index.html` in Chrome. Click each toolbar button — it should highlight blue. Click Rectangle, then click on the canvas — a rectangle should appear and the Select tool should become active. Open DevTools console and run:

```js
scene.elements.length   // Expected: 1 after placing one shape
activeTool              // Expected: "select"
```

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: tool selection and element placement"
```

---

### Task 2: Hit Testing + Click Selection

Implement `hitTestElement()`, `hitTest()`, and canvas click-to-select with shift-click multi-select. Draw a blue bounding outline around selected elements.

**Files:**
- Modify: `index.html` — replace `// === PLACEHOLDER — hit testing added in Task 2 ===`

- [ ] **Step 1: Replace the hit-testing placeholder with hit testing + selection overlay**

```js
// === HIT TESTING ===

function localToWorld(el, lx, ly) {
  const cos = Math.cos(el.rotation * Math.PI / 180);
  const sin = Math.sin(el.rotation * Math.PI / 180);
  return {
    x: el.x + lx * cos - ly * sin,
    y: el.y + lx * sin + ly * cos,
  };
}

function worldToLocal(el, wx, wy) {
  const dx = wx - el.x;
  const dy = wy - el.y;
  const cos = Math.cos(el.rotation * Math.PI / 180);
  const sin = Math.sin(el.rotation * Math.PI / 180);
  return {
    x:  dx * cos + dy * sin,
    y: -dx * sin + dy * cos,
  };
}

function hitTestElement(el, px, py) {
  const local = worldToLocal(el, px, py);
  return Math.abs(local.x) <= el.width / 2 + 2 &&
         Math.abs(local.y) <= el.height / 2 + 2;
}

// Returns the topmost visible, unlocked element id at (px, py), or null.
function hitTest(px, py) {
  for (let i = scene.elements.length - 1; i >= 0; i--) {
    const el = scene.elements[i];
    if (!el.visible || el.locked) continue;
    if (hitTestElement(el, px, py)) return el.id;
  }
  return null;
}

// Draw blue outline + fill for selected elements (preview only).
function drawSelectionOutlines(ctx) {
  if (selectedIds.length === 0) return;
  for (const id of selectedIds) {
    const el = scene.elements.find(e => e.id === id);
    if (!el || !el.visible) continue;
    ctx.save();
    ctx.translate(el.x, el.y);
    ctx.rotate(el.rotation * Math.PI / 180);
    ctx.strokeStyle = 'rgba(74, 158, 255, 0.9)';
    ctx.lineWidth = 1.5 / previewScale;
    ctx.setLineDash([4 / previewScale, 4 / previewScale]);
    ctx.strokeRect(-el.width / 2, -el.height / 2, el.width, el.height);
    ctx.setLineDash([]);
    ctx.restore();
  }
}
```

- [ ] **Step 2: Call `drawSelectionOutlines` at the end of `drawScene`**

In `index.html`, find the end of the `drawScene` function (the line that draws the divider). Add one call after the divider drawing block:

```js
  // After the divider block, at the end of drawScene:
  if (opts.showDivider !== false) {
    drawSelectionOutlines(ctx);
    // handles drawn in Task 4
  }
```

- [ ] **Step 3: Wire canvas `mousedown` for click-select (no drag yet)**

At the end of `// === HIT TESTING ===`, add:

```js
document.getElementById('preview-canvas').addEventListener('mousedown', handleCanvasMouseDown);

function handleCanvasMouseDown(e) {
  const { x, y } = canvasPos(e);

  // Shape/text placement tool: place on click and return
  if (activeTool !== 'select') {
    placeElement(activeTool, x, y);
    return;
  }

  // Select tool: hit test
  const hitId = hitTest(x, y);
  if (hitId) {
    if (e.shiftKey) {
      if (selectedIds.includes(hitId)) {
        selectedIds = selectedIds.filter(id => id !== hitId);
      } else {
        selectedIds = [...selectedIds, hitId];
      }
    } else if (!selectedIds.includes(hitId)) {
      selectedIds = [hitId];
    }
  } else {
    if (!e.shiftKey) selectedIds = [];
  }
  refreshPanel();
}
```

- [ ] **Step 4: Verify selection in browser**

Open `index.html`, place two shapes, then click each one — a blue dashed outline should appear. Shift-click the second to multi-select both. Click empty canvas to deselect.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: hit testing and click-to-select"
```

---

### Task 3: Move Drag + Rubber-Band Selection

Add `dragState` and full `mousemove`/`mouseup` handlers. Dragging an element moves it; dragging on empty canvas draws a rubber-band rectangle and selects enclosed elements.

**Files:**
- Modify: `index.html` — replace `// === PLACEHOLDER — drag added in Task 3 ===`

- [ ] **Step 1: Replace the drag placeholder with `dragState` and move/rubber-band logic**

```js
// === DRAG ===

let dragState = null;
// Shapes:
// { type: 'move', startX, startY, snaps: [{id, x, y}] }
// { type: 'rubber-band', startX, startY, currentX, currentY }
// (resize and rotate shapes added in Task 4)

// Upgrade the mousedown handler to also start drag.
// Replace handleCanvasMouseDown with this full version:
document.getElementById('preview-canvas').removeEventListener('mousedown', handleCanvasMouseDown);

function handleCanvasMouseDown(e) {
  const { x, y } = canvasPos(e);

  if (activeTool !== 'select') {
    placeElement(activeTool, x, y);
    return;
  }

  // Check handles first (populated in Task 4; no-op here)
  const handle = typeof hitTestHandles === 'function' ? hitTestHandles(x, y) : null;
  if (handle) return; // Task 4 takes over

  const hitId = hitTest(x, y);
  if (hitId) {
    // Select
    if (e.shiftKey) {
      if (selectedIds.includes(hitId)) {
        selectedIds = selectedIds.filter(id => id !== hitId);
      } else {
        selectedIds = [...selectedIds, hitId];
      }
    } else if (!selectedIds.includes(hitId)) {
      selectedIds = [hitId];
    }
    refreshPanel();

    // Start move drag
    pushUndo();
    dragState = {
      type: 'move',
      startX: x,
      startY: y,
      snaps: selectedIds.map(id => {
        const el = scene.elements.find(e => e.id === id);
        return { id, x: el.x, y: el.y };
      }),
    };
  } else {
    if (!e.shiftKey) selectedIds = [];
    refreshPanel();
    // Start rubber-band
    dragState = { type: 'rubber-band', startX: x, startY: y, currentX: x, currentY: y };
  }
}

document.getElementById('preview-canvas').addEventListener('mousedown', handleCanvasMouseDown);

document.addEventListener('mousemove', e => {
  if (!dragState) return;
  const { x, y } = canvasPos(e);

  if (dragState.type === 'move') {
    const dx = x - dragState.startX;
    const dy = y - dragState.startY;
    for (const snap of dragState.snaps) {
      updateElement(snap.id, { x: snap.x + dx, y: snap.y + dy });
    }
    refreshPanel();
  }

  if (dragState.type === 'rubber-band') {
    dragState.currentX = x;
    dragState.currentY = y;
  }
});

document.addEventListener('mouseup', e => {
  if (!dragState) return;
  if (dragState.type === 'rubber-band') {
    const rx = Math.min(dragState.startX, dragState.currentX);
    const ry = Math.min(dragState.startY, dragState.currentY);
    const rw = Math.abs(dragState.currentX - dragState.startX);
    const rh = Math.abs(dragState.currentY - dragState.startY);
    if (rw > 4 || rh > 4) {
      selectedIds = scene.elements
        .filter(el => el.visible && !el.locked)
        .filter(el => {
          const hw = el.width / 2, hh = el.height / 2;
          return el.x - hw < rx + rw && el.x + hw > rx &&
                 el.y - hh < ry + rh && el.y + hh > ry;
        })
        .map(el => el.id);
      refreshPanel();
    }
  }
  dragState = null;
});

// Draw rubber-band rect in drawScene. Called from inside drawScene (preview only).
function drawRubberBand(ctx) {
  if (!dragState || dragState.type !== 'rubber-band') return;
  const rx = Math.min(dragState.startX, dragState.currentX);
  const ry = Math.min(dragState.startY, dragState.currentY);
  const rw = Math.abs(dragState.currentX - dragState.startX);
  const rh = Math.abs(dragState.currentY - dragState.startY);
  ctx.save();
  ctx.strokeStyle = 'rgba(74, 158, 255, 0.8)';
  ctx.fillStyle = 'rgba(74, 158, 255, 0.08)';
  ctx.lineWidth = 1 / previewScale;
  ctx.setLineDash([4 / previewScale, 3 / previewScale]);
  ctx.fillRect(rx, ry, rw, rh);
  ctx.strokeRect(rx, ry, rw, rh);
  ctx.setLineDash([]);
  ctx.restore();
}
```

- [ ] **Step 2: Call `drawRubberBand` inside `drawScene`**

Find the `drawSelectionOutlines(ctx)` call added in Task 2 and add `drawRubberBand` right after:

```js
  if (opts.showDivider !== false) {
    drawSelectionOutlines(ctx);
    drawRubberBand(ctx);
    // handles drawn in Task 4
  }
```

- [ ] **Step 3: Verify drag in browser**

Open `index.html`, place two shapes. Drag one — it should move. Drag on empty canvas — a blue rectangle should appear and on release the shapes it covers should be selected.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: element move drag and rubber-band selection"
```

---

### Task 4: Resize + Rotate Handles

Draw 8 resize handles and 1 rotation handle around the selected element. Implement handle hit testing and resize/rotate drag.

**Files:**
- Modify: `index.html` — replace `// === PLACEHOLDER — handles added in Task 4 ===`

- [ ] **Step 1: Replace the handles placeholder with handle drawing and hit testing**

```js
// === HANDLES ===

const HANDLE_CORNERS = [
  { lx: -1, ly: -1, corner: 'tl', cursor: 'nw-resize' },
  { lx:  0, ly: -1, corner: 'tm', cursor: 'n-resize'  },
  { lx:  1, ly: -1, corner: 'tr', cursor: 'ne-resize' },
  { lx:  1, ly:  0, corner: 'mr', cursor: 'e-resize'  },
  { lx:  1, ly:  1, corner: 'br', cursor: 'se-resize' },
  { lx:  0, ly:  1, corner: 'bm', cursor: 's-resize'  },
  { lx: -1, ly:  1, corner: 'bl', cursor: 'sw-resize' },
  { lx: -1, ly:  0, corner: 'ml', cursor: 'w-resize'  },
];
const ROT_HANDLE_OFFSET = 20; // px above top-center in element local space

function getHandleWorld(el, hDef) {
  const lx = hDef.lx * el.width / 2;
  const ly = hDef.ly * el.height / 2;
  return localToWorld(el, lx, ly);
}

function getRotHandleWorld(el) {
  return localToWorld(el, 0, -el.height / 2 - ROT_HANDLE_OFFSET);
}

function drawHandles(ctx, el) {
  const hs = 7 / previewScale; // handle size in output pixels
  ctx.save();

  // Draw line to rotation handle
  const rotPt = getRotHandleWorld(el);
  const topMid = getHandleWorld(el, HANDLE_CORNERS.find(h => h.corner === 'tm'));
  ctx.strokeStyle = 'rgba(74, 158, 255, 0.7)';
  ctx.lineWidth = 1 / previewScale;
  ctx.beginPath();
  ctx.moveTo(topMid.x, topMid.y);
  ctx.lineTo(rotPt.x, rotPt.y);
  ctx.stroke();

  // Rotation handle (circle)
  ctx.fillStyle = '#4a9eff';
  ctx.strokeStyle = '#fff';
  ctx.lineWidth = 1 / previewScale;
  ctx.beginPath();
  ctx.arc(rotPt.x, rotPt.y, hs / 2, 0, Math.PI * 2);
  ctx.fill();
  ctx.stroke();

  // Resize handles (squares)
  for (const h of HANDLE_CORNERS) {
    const pt = getHandleWorld(el, h);
    ctx.fillStyle = '#fff';
    ctx.strokeStyle = '#4a9eff';
    ctx.lineWidth = 1 / previewScale;
    ctx.fillRect(pt.x - hs / 2, pt.y - hs / 2, hs, hs);
    ctx.strokeRect(pt.x - hs / 2, pt.y - hs / 2, hs, hs);
  }

  ctx.restore();
}

// Returns { corner, type } or null. Only works for single-element selection.
function hitTestHandles(px, py) {
  if (selectedIds.length !== 1) return null;
  const el = scene.elements.find(e => e.id === selectedIds[0]);
  if (!el) return null;
  const hs = 10 / previewScale; // slightly larger hit zone

  // Rotation handle
  const rotPt = getRotHandleWorld(el);
  if (Math.abs(px - rotPt.x) <= hs && Math.abs(py - rotPt.y) <= hs) {
    return { corner: 'rot', type: 'rotate' };
  }

  // Resize handles
  for (const h of HANDLE_CORNERS) {
    const pt = getHandleWorld(el, h);
    if (Math.abs(px - pt.x) <= hs && Math.abs(py - pt.y) <= hs) {
      return { corner: h.corner, type: 'resize', cursor: h.cursor };
    }
  }
  return null;
}
```

- [ ] **Step 2: Call `drawHandles` inside `drawScene` for single selection**

Find the line `// handles drawn in Task 4` inside `drawScene` and replace it:

```js
  if (opts.showDivider !== false) {
    drawSelectionOutlines(ctx);
    drawRubberBand(ctx);
    if (selectedIds.length === 1) {
      const selEl = scene.elements.find(e => e.id === selectedIds[0]);
      if (selEl && selEl.visible) drawHandles(ctx, selEl);
    }
  }
```

- [ ] **Step 3: Add resize + rotate drag to the mousedown handler**

Upgrade `handleCanvasMouseDown` — replace the comment `// Check handles first (populated in Task 4; no-op here)` with live handle handling:

```js
  const handle = typeof hitTestHandles === 'function' ? hitTestHandles(x, y) : null;
  if (handle) {
    e.preventDefault();
    const el = scene.elements.find(e => e.id === selectedIds[0]);
    pushUndo();
    if (handle.type === 'rotate') {
      dragState = {
        type: 'rotate',
        startAngle: Math.atan2(y - el.y, x - el.x),
        startRot: el.rotation,
        snap: { id: el.id },
      };
    } else {
      // Find the anchor corner (opposite corner/edge in world space)
      const opp = {
        tl: 'br', tm: 'bm', tr: 'bl', mr: 'ml',
        br: 'tl', bm: 'tm', bl: 'tr', ml: 'mr',
      }[handle.corner];
      const oppDef = HANDLE_CORNERS.find(h => h.corner === opp);
      const anchorWorld = getHandleWorld(el, oppDef);
      dragState = {
        type: 'resize',
        corner: handle.corner,
        anchorWorld,
        snap: { id: el.id, rotation: el.rotation, width: el.width, height: el.height },
        shiftKey: false,
        origAspect: el.width / el.height,
      };
    }
    return;
  }
```

- [ ] **Step 4: Add resize + rotate handling inside the `mousemove` handler**

Inside the `document.addEventListener('mousemove', ...)` handler, after the rubber-band block, add:

```js
  if (dragState && dragState.type === 'resize') {
    const { corner, anchorWorld, snap } = dragState;
    const el = scene.elements.find(e => e.id === snap.id);
    const cos = Math.cos(snap.rotation * Math.PI / 180);
    const sin = Math.sin(snap.rotation * Math.PI / 180);
    // Vector from anchor to mouse in world space
    const dx = x - anchorWorld.x;
    const dy = y - anchorWorld.y;
    // Project onto element local axes
    const localX = dx * cos + dy * sin;
    const localY = -dx * sin + dy * cos;

    let newW = snap.width, newH = snap.height;
    if (['tl', 'tr', 'bl', 'br'].includes(corner)) {
      newW = Math.max(10, Math.abs(localX));
      newH = Math.max(10, Math.abs(localY));
      if (e.shiftKey) {
        const side = Math.max(newW, newH);
        newW = newH = side;
      }
    } else if (['tm', 'bm'].includes(corner)) {
      newH = Math.max(10, Math.abs(localY));
    } else {
      newW = Math.max(10, Math.abs(localX));
    }

    // New center = midpoint(anchor, dragged corner in world)
    const newCX = (anchorWorld.x + x) / 2;
    const newCY = (anchorWorld.y + y) / 2;
    updateElement(snap.id, { x: newCX, y: newCY, width: newW, height: newH });
    refreshPanel();
  }

  if (dragState && dragState.type === 'rotate') {
    const { snap } = dragState;
    const el = scene.elements.find(e => e.id === snap.id);
    const currentAngle = Math.atan2(y - el.y, x - el.x);
    const delta = (currentAngle - dragState.startAngle) * (180 / Math.PI);
    let newRot = (dragState.startRot + delta) % 360;
    if (newRot < 0) newRot += 360;
    if (e.shiftKey) newRot = Math.round(newRot / 15) * 15;
    updateElement(snap.id, { rotation: newRot });
    refreshPanel();
  }
```

- [ ] **Step 5: Update canvas cursor on hover over handles**

Add a `mousemove` listener on the canvas element for cursor updates:

```js
document.getElementById('preview-canvas').addEventListener('mousemove', e => {
  if (dragState) return;
  if (activeTool !== 'select') return;
  const { x, y } = canvasPos(e);
  const handle = hitTestHandles(x, y);
  const canvas = document.getElementById('preview-canvas');
  if (handle) {
    canvas.style.cursor = handle.type === 'rotate' ? 'grab' : handle.cursor;
  } else if (hitTest(x, y)) {
    canvas.style.cursor = 'move';
  } else {
    canvas.style.cursor = 'default';
  }
});
```

- [ ] **Step 6: Verify handles in browser**

Open `index.html`, place a rectangle and select it. Eight square handles and one circular rotation handle should appear. Drag a corner handle — the element should resize. Drag the rotation handle — it should rotate. Hold Shift during rotation — it snaps to 15° increments.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat: resize and rotate handles"
```

---

### Task 5: Right Panel — Transform, Depth, Style, Element Actions

Implement `refreshPanel()` to sync all inputs from the selection, and wire every input back to `updateSelectedElements`.

**Files:**
- Modify: `index.html` — replace `// === PLACEHOLDER — panel wiring added in Task 5 ===`

- [ ] **Step 1: Replace the panel placeholder with `refreshPanel` and all input wiring**

```js
// === PANEL WIRING ===

function refreshPanel() {
  const count = selectedIds.length;
  const sections = ['section-transform', 'section-depth', 'section-style', 'section-actions', 'section-typography'];
  sections.forEach(id => document.getElementById(id).classList.toggle('hidden', count === 0));
  document.getElementById('no-selection').style.display = count === 0 ? '' : 'none';

  if (count === 0) { renderLayersList(); return; }

  const el = count === 1 ? scene.elements.find(e => e.id === selectedIds[0]) : null;

  // Transform
  const allSameX = selectedIds.every(id => scene.elements.find(e => e.id === id).x === selectedIds.map(i => scene.elements.find(e => e.id === i).x)[0]);
  function valOrBlank(getter) {
    const vals = selectedIds.map(id => getter(scene.elements.find(e => e.id === id)));
    return vals.every(v => v === vals[0]) ? String(Math.round(vals[0])) : '';
  }
  document.getElementById('prop-x').value   = valOrBlank(e => e.x);
  document.getElementById('prop-y').value   = valOrBlank(e => e.y);
  document.getElementById('prop-w').value   = valOrBlank(e => e.width);
  document.getElementById('prop-h').value   = valOrBlank(e => e.height);
  document.getElementById('prop-rot').value = valOrBlank(e => e.rotation);

  // Depth
  const depthVal = valOrBlank(e => e.depth);
  document.getElementById('prop-depth').value     = depthVal;
  document.getElementById('prop-depth-num').value = depthVal;
  const depthNum = depthVal !== '' ? parseFloat(depthVal) : 0;
  const pct = ((depthNum + 100) / 200) * 100;
  document.getElementById('depth-track').style.left  = `${Math.min(pct, 50)}%`;
  document.getElementById('depth-track').style.width = `${Math.abs(pct - 50)}%`;
  document.getElementById('depth-thumb').style.left  = `${pct}%`;
  document.getElementById('depth-readout').textContent = depthVal !== '' ? `${depthVal} px` : '—';

  // Style
  if (el) {
    document.querySelectorAll('#fill-style-toggle .style-btn').forEach(b => {
      b.classList.toggle('active', b.dataset.style === el.fillStyle);
    });
    document.getElementById('prop-fill-color').value   = el.fillColor;
    document.getElementById('prop-stroke-color').value = el.strokeColor;
    document.getElementById('prop-stroke-w').value     = el.strokeWidth;
  }

  // Typography — show only for single text element
  const isText = el && el.type === 'text';
  document.getElementById('section-typography').classList.toggle('hidden', !isText);
  if (isText) {
    document.getElementById('prop-font-family').value    = el.fontFamily;
    document.getElementById('prop-font-size').value      = el.fontSize;
    document.getElementById('prop-font-weight').value    = el.fontWeight;
    document.getElementById('prop-italic').checked       = el.fontStyle === 'italic';
    document.getElementById('prop-letter-spacing').value = el.letterSpacing;
    document.getElementById('prop-line-height').value    = el.lineHeight;
    document.getElementById('prop-text-align').value     = el.textAlign;
  }

  // Visibility / lock buttons
  if (el) {
    document.getElementById('btn-vis').textContent  = el.visible ? '👁 Hide' : '👁 Show';
    document.getElementById('btn-lock').textContent = el.locked  ? '🔒 Unlock' : '🔓 Lock';
  }

  renderLayersList();
}

// ---- Transform inputs ----
function wireNumberInput(id, prop, parser = parseFloat) {
  document.getElementById(id).addEventListener('change', e => {
    const v = parser(e.target.value);
    if (isNaN(v)) return;
    pushUndo();
    updateSelectedElements({ [prop]: v });
    refreshPanel();
  });
}
wireNumberInput('prop-x', 'x');
wireNumberInput('prop-y', 'y');
wireNumberInput('prop-w', 'width');
wireNumberInput('prop-h', 'height');
wireNumberInput('prop-rot', 'rotation');

// ---- Depth ----
function applyDepth(val) {
  const v = parseFloat(val);
  if (isNaN(v)) return;
  pushUndo();
  updateSelectedElements({ depth: v });
  refreshPanel();
}
document.getElementById('prop-depth').addEventListener('input', e => applyDepth(e.target.value));
document.getElementById('prop-depth-num').addEventListener('change', e => applyDepth(e.target.value));

// ---- Style toggle ----
document.querySelectorAll('#fill-style-toggle .style-btn').forEach(btn => {
  btn.addEventListener('click', () => {
    pushUndo();
    updateSelectedElements({ fillStyle: btn.dataset.style });
    refreshPanel();
  });
});

// ---- Colors ----
document.getElementById('prop-fill-color').addEventListener('input', e => {
  updateSelectedElements({ fillColor: e.target.value });
});
document.getElementById('prop-stroke-color').addEventListener('input', e => {
  updateSelectedElements({ strokeColor: e.target.value });
});
document.getElementById('prop-stroke-w').addEventListener('change', e => {
  const v = parseFloat(e.target.value);
  if (!isNaN(v) && v >= 0) { pushUndo(); updateSelectedElements({ strokeWidth: v }); }
});

// ---- Element actions ----
document.getElementById('btn-fwd').addEventListener('click', () => {
  pushUndo();
  selectedIds.forEach(id => bringForward(id));
  refreshPanel();
});
document.getElementById('btn-back').addEventListener('click', () => {
  pushUndo();
  selectedIds.forEach(id => sendBack(id));
  refreshPanel();
});
document.getElementById('btn-dup').addEventListener('click', () => {
  pushUndo();
  const newIds = duplicateSelected();
  selectedIds = newIds;
  refreshPanel();
});
document.getElementById('btn-del').addEventListener('click', () => {
  pushUndo();
  removeElements(selectedIds);
  selectedIds = [];
  refreshPanel();
});
document.getElementById('btn-vis').addEventListener('click', () => {
  pushUndo();
  const el = scene.elements.find(e => e.id === selectedIds[0]);
  if (el) updateSelectedElements({ visible: !el.visible });
  refreshPanel();
});
document.getElementById('btn-lock').addEventListener('click', () => {
  pushUndo();
  const el = scene.elements.find(e => e.id === selectedIds[0]);
  if (el) updateSelectedElements({ locked: !el.locked });
  refreshPanel();
});
document.getElementById('btn-clear').addEventListener('click', () => {
  if (!confirm('Remove all elements?')) return;
  pushUndo();
  scene.elements = [];
  selectedIds = [];
  refreshPanel();
});

// ---- Undo / Redo buttons (wire keyboard too, see Task 6) ----
document.addEventListener('keydown', e => {
  if ((e.metaKey || e.ctrlKey) && e.key === 'z' && !e.shiftKey) {
    e.preventDefault(); applyUndo(); refreshPanel();
  }
  if ((e.metaKey || e.ctrlKey) && (e.key === 'y' || (e.key === 'z' && e.shiftKey))) {
    e.preventDefault(); applyRedo(); refreshPanel();
  }
});
```

- [ ] **Step 2: Verify panel wiring in browser**

Open `index.html`, place a rectangle and select it. The right panel should show Transform, Depth, Style sections. Change the X value in the panel — the shape should move. Drag the depth slider — the shape should shift left/right in the stereo preview.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: right panel transform, depth, style, and action wiring"
```

---

### Task 6: Typography, Text Editing, Layers List + Keyboard Shortcuts

Wire typography inputs, implement double-click inline text editing, build the layers list with visibility toggle and drag-to-reorder, and add all keyboard shortcuts.

**Files:**
- Modify: `index.html` — replace `// === PLACEHOLDER — layers + keyboard added in Task 6 ===`

- [ ] **Step 1: Replace the layers + keyboard placeholder**

```js
// === TYPOGRAPHY ===

function wireTypographyInput(id, prop, parser) {
  const el = document.getElementById(id);
  const evt = el.tagName === 'SELECT' || el.type === 'checkbox' ? 'change' : 'change';
  el.addEventListener(evt, e => {
    if (selectedIds.length !== 1) return;
    pushUndo();
    const v = parser ? parser(e.target.value) : (e.target.type === 'checkbox' ? (e.target.checked ? 'italic' : 'normal') : e.target.value);
    updateSelectedElements({ [prop]: v });
    refreshPanel();
  });
}

wireTypographyInput('prop-font-family', 'fontFamily', null);
wireTypographyInput('prop-font-size', 'fontSize', parseFloat);
wireTypographyInput('prop-font-weight', 'fontWeight', null);
wireTypographyInput('prop-italic', 'fontStyle', null);  // checkbox: handled specially above
wireTypographyInput('prop-letter-spacing', 'letterSpacing', parseFloat);
wireTypographyInput('prop-line-height', 'lineHeight', parseFloat);
wireTypographyInput('prop-text-align', 'textAlign', null);

// Fix italic: checkbox value is not the raw value
document.getElementById('prop-italic').addEventListener('change', e => {
  if (selectedIds.length !== 1) return;
  pushUndo();
  updateSelectedElements({ fontStyle: e.target.checked ? 'italic' : 'normal' });
  refreshPanel();
});

// === TEXT INLINE EDITOR ===

let editingTextId = null;

document.getElementById('preview-canvas').addEventListener('dblclick', e => {
  const { x, y } = canvasPos(e);
  const hitId = hitTest(x, y);
  if (!hitId) return;
  const el = scene.elements.find(el => el.id === hitId);
  if (!el || el.type !== 'text') return;
  openTextEditor(el);
});

function openTextEditor(el) {
  editingTextId = el.id;
  const overlay = document.getElementById('text-editor-overlay');
  const textarea = document.getElementById('text-editor');
  const canvas = document.getElementById('preview-canvas');
  const rect = canvas.getBoundingClientRect();

  // Position the overlay over the element
  const screenX = (el.x - el.width / 2) * previewScale + rect.left;
  const screenY = (el.y - el.height / 2) * previewScale + rect.top;
  const screenW = el.width * previewScale;
  const screenH = Math.max(el.height * previewScale, 40);

  overlay.style.left   = `${screenX}px`;
  overlay.style.top    = `${screenY}px`;
  overlay.style.width  = `${screenW}px`;
  overlay.style.height = `${screenH}px`;
  overlay.style.display = 'block';
  overlay.style.position = 'fixed';

  textarea.style.fontSize   = `${el.fontSize * previewScale}px`;
  textarea.style.fontFamily = el.fontFamily;
  textarea.style.fontWeight = el.fontWeight;
  textarea.style.fontStyle  = el.fontStyle;
  textarea.style.textAlign  = el.textAlign;
  textarea.style.color      = el.fillColor;
  textarea.value = el.text;
  textarea.focus();
  textarea.select();
}

function commitTextEdit() {
  if (!editingTextId) return;
  const textarea = document.getElementById('text-editor');
  pushUndo();
  updateElement(editingTextId, { text: textarea.value });
  editingTextId = null;
  document.getElementById('text-editor-overlay').style.display = 'none';
  refreshPanel();
}

document.getElementById('text-editor').addEventListener('keydown', e => {
  if (e.key === 'Escape') { e.preventDefault(); commitTextEdit(); }
});
document.getElementById('text-editor').addEventListener('blur', () => commitTextEdit());

// === LAYERS LIST ===

let layerDragId = null;

function renderLayersList() {
  const list = document.getElementById('layers-list');
  list.innerHTML = '';
  // Reverse: top of list = frontmost (highest index)
  const reversed = [...scene.elements].reverse();
  for (const el of reversed) {
    const item = document.createElement('div');
    item.className = 'layer-item' + (selectedIds.includes(el.id) ? ' selected' : '');
    item.dataset.id = el.id;
    item.draggable = true;

    const eye = document.createElement('span');
    eye.className = 'layer-eye';
    eye.textContent = el.visible ? '👁' : '○';
    eye.title = el.visible ? 'Hide' : 'Show';
    eye.addEventListener('click', e => {
      e.stopPropagation();
      pushUndo();
      updateElement(el.id, { visible: !el.visible });
      refreshPanel();
    });

    const label = document.createElement('span');
    label.className = 'layer-label';
    const typeLabel = el.type === 'text' ? `"${el.text.slice(0, 14)}"` : el.type;
    label.textContent = el.locked ? `🔒 ${typeLabel}` : typeLabel;
    label.style.opacity = el.visible ? '1' : '0.4';

    item.appendChild(eye);
    item.appendChild(label);

    item.addEventListener('click', () => {
      selectedIds = [el.id];
      refreshPanel();
    });

    // Drag-to-reorder
    item.addEventListener('dragstart', e => {
      layerDragId = el.id;
      e.dataTransfer.effectAllowed = 'move';
    });
    item.addEventListener('dragover', e => {
      e.preventDefault();
      e.dataTransfer.dropEffect = 'move';
      item.style.borderTop = '2px solid var(--accent)';
    });
    item.addEventListener('dragleave', () => { item.style.borderTop = ''; });
    item.addEventListener('drop', e => {
      e.preventDefault();
      item.style.borderTop = '';
      if (!layerDragId || layerDragId === el.id) return;
      pushUndo();
      const fromIdx = scene.elements.findIndex(e => e.id === layerDragId);
      const toIdx   = scene.elements.findIndex(e => e.id === el.id);
      const [moved] = scene.elements.splice(fromIdx, 1);
      scene.elements.splice(toIdx, 0, moved);
      refreshPanel();
    });

    list.appendChild(item);
  }
}

// === KEYBOARD SHORTCUTS ===

document.addEventListener('keydown', e => {
  // Don't fire shortcuts when typing in inputs
  const tag = document.activeElement.tagName;
  if (tag === 'INPUT' || tag === 'TEXTAREA' || tag === 'SELECT') return;

  if (e.key === 'Escape') {
    selectedIds = [];
    setActiveTool('select');
    refreshPanel();
  }
  if ((e.key === 'Delete' || e.key === 'Backspace') && selectedIds.length > 0) {
    e.preventDefault();
    pushUndo();
    removeElements(selectedIds);
    selectedIds = [];
    refreshPanel();
  }
  if ((e.metaKey || e.ctrlKey) && e.key === 'd') {
    e.preventDefault();
    pushUndo();
    const newIds = duplicateSelected();
    selectedIds = newIds;
    refreshPanel();
  }
  if (e.key === 'ArrowLeft' || e.key === 'ArrowRight' || e.key === 'ArrowUp' || e.key === 'ArrowDown') {
    e.preventDefault();
    const step = e.shiftKey ? 10 : 1;
    const dx = e.key === 'ArrowLeft' ? -step : e.key === 'ArrowRight' ? step : 0;
    const dy = e.key === 'ArrowUp'   ? -step : e.key === 'ArrowDown'  ? step : 0;
    pushUndo();
    selectedIds.forEach(id => {
      const el = scene.elements.find(e => e.id === id);
      if (el) updateElement(id, { x: el.x + dx, y: el.y + dy });
    });
    refreshPanel();
  }
});

// Call refreshPanel once on init to set initial state
refreshPanel();
```

- [ ] **Step 2: Add CSS for `.layer-item` and `.layer-eye` inside the `<style>` block**

Find the end of the CSS `<style>` block (just before `</style>`) and append:

```css
    /* Layers list */
    .layer-item {
      display: flex;
      align-items: center;
      gap: 6px;
      padding: 4px 6px;
      border-radius: 3px;
      cursor: pointer;
      border: 1px solid transparent;
      border-top: 2px solid transparent;
    }
    .layer-item:hover { background: var(--surface); }
    .layer-item.selected { background: rgba(58, 123, 213, 0.15); border-color: rgba(58, 123, 213, 0.3); }
    .layer-eye { font-size: 11px; flex-shrink: 0; cursor: pointer; }
    .layer-eye:hover { opacity: 0.7; }
    .layer-label { font-size: 10px; color: var(--text-secondary); overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }

    /* Text editor overlay */
    #text-editor-overlay { display: none; position: fixed; z-index: 100; }
    #text-editor {
      width: 100%; height: 100%;
      background: rgba(255,255,255,0.08);
      border: 1px solid var(--accent);
      outline: none;
      resize: none;
      padding: 2px 4px;
      font-family: inherit;
      color: #fff;
      caret-color: var(--accent);
    }
```

- [ ] **Step 3: Verify all features in browser**

Open `index.html` and test the following:
1. Place a text element, double-click it — an editable textarea should appear over it
2. Type text, press Escape — the new text should render in the canvas
3. The Layers list at the bottom of the right panel should show all elements
4. Click a layer item to select that element
5. Click the eye icon on a layer to hide/show it
6. Drag a layer item up/down to reorder
7. Select an element and press Delete — it removes
8. Press Cmd/Ctrl+Z — it undoes
9. Press Arrow keys — the selected element nudges 1px; Shift+Arrow nudges 10px
10. Press Escape — deselects

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: typography, inline text editing, layers list, keyboard shortcuts"
```

---

## Self-Review

### Spec coverage

| Spec requirement | Covered by |
|---|---|
| Select / deselect click | Task 2 |
| Shift-click multi-select | Task 2 |
| Rubber-band selection | Task 3 |
| Element move drag | Task 3 |
| 8 resize handles | Task 4 |
| Rotation handle | Task 4 |
| Shift-constrain resize aspect ratio | Task 4 (Step 4, `e.shiftKey`) |
| Shift-snap rotation to 15° | Task 4 (Step 4, rotate block) |
| Transform inputs (X/Y/W/H/Rot) | Task 5 |
| Depth slider + numeric input | Task 5 |
| Fill style toggle | Task 5 |
| Fill / stroke color pickers | Task 5 |
| Stroke width | Task 5 |
| Bring Fwd / Send Back | Task 5 |
| Duplicate | Task 5 |
| Delete button | Task 5 |
| Visibility + Lock toggles | Task 5 |
| Clear All | Task 5 |
| Undo / Redo (Cmd+Z / Cmd+Shift+Z) | Task 5 |
| Typography inputs (all) | Task 6 |
| Double-click inline text edit | Task 6 |
| Layers list + reorder | Task 6 |
| Layers visibility toggle | Task 6 |
| Delete/Backspace shortcut | Task 6 |
| Cmd+D duplicate shortcut | Task 6 |
| Arrow key nudge (1px + 10px) | Task 6 |
| Escape deselect / exit text edit | Task 6 |
| SVG upload + paste modal | Task 1 |
| Tool selection (all tools) | Task 1 |
| Multi-select depth delta | Spec says "adjusts all by same delta" — Task 5 `applyDepth` calls `updateSelectedElements` which applies uniformly ✓ |
| Multi-select blank inputs when values differ | Task 5 `valOrBlank` helper ✓ |

### Placeholder scan

No TBDs, TODOs, or "fill in later" phrases. Every step includes exact code.

### Type consistency

- `localToWorld` / `worldToLocal` defined in Task 2, used in Task 4 — ✓
- `HANDLE_CORNERS` defined in Task 4, used by `getHandleWorld`, `drawHandles`, `hitTestHandles` — ✓
- `dragState.type` values: `'move'`, `'rubber-band'`, `'resize'`, `'rotate'` — used consistently in Tasks 3/4 ✓
- `refreshPanel()` defined in Task 5, called from Tasks 1/2/3/4/5/6 — ✓ (Task 1–4 call it; must be defined before it's needed at runtime since JS is not hoisted for `let` functions — **fix**: Tasks 1–4 use `refreshPanel()` calls that happen at runtime in event handlers, after Task 5's code runs. Since all code is in a single `<script>` block executed top to bottom, and event handlers fire after the full script runs, the call order is safe ✓)
- `hitTestHandles` called in Task 3 via `typeof hitTestHandles === 'function'` guard, then defined in Task 4 — guard makes it safe before Task 4's code is inserted ✓
- `sanitizeSVG` called in Task 1, defined in Part A Task 6 — ✓
- `duplicateSelected` returns new IDs in Part A definition (returns `newIds`) — confirmed at Part A line `selectedIds = newIds` ✓

---

**Plan complete and saved to `docs/superpowers/plans/2026-04-11-stereoforge-part-b.md`.**

**Two execution options:**

**1. Subagent-Driven (recommended)** — Fresh subagent per task, review between tasks, fast iteration

**2. Inline Execution** — Execute tasks in this session using executing-plans, with checkpoints

**Which approach?**
