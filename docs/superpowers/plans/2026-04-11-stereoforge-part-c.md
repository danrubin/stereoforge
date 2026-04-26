# StereoForge — Implementation Plan (Part C: Features + Polish)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Complete the app by adding the font system (local fonts, Google Fonts, custom font injection), PNG export, project save/load, and mobile layout.

**Architecture:** All code is added to `index.html`, replacing the three remaining Part C placeholders: `// === PLACEHOLDER — fonts added in Part C ===`, `// === PLACEHOLDER — export added in Part C ===`, and `// === PLACEHOLDER — save/load added in Part C ===`. Mobile layout adds CSS to `<style>` and a `#toolbar-tab` element to the HTML. Note: keyboard shortcuts and output presets are already wired in Part B and Part A respectively.

**Tech Stack:** Vanilla JS, Local Font Access API (`queryLocalFonts`), Google Fonts CDN, Canvas 2D API, FileReader, Blob/URL APIs.

---

## File Structure

- Modify: `index.html` — five sequential edits (one per task)

---

### Task 1: Font System — Local Fonts (Tier 1)

Populate the `#prop-font-family` dropdown with system fonts via `queryLocalFonts()`. Fall back to a hardcoded list and show the inline note on unsupported browsers.

**Files:**
- Modify: `index.html` — replace `// === PLACEHOLDER — fonts added in Part C ===`

- [ ] **Step 1: Replace `// === PLACEHOLDER — fonts added in Part C ===` with the local font loader**

```js
// === FONT SYSTEM ===

const FONT_FALLBACK = [
  'Arial', 'Courier New', 'Georgia', 'Helvetica',
  'Impact', 'Times New Roman', 'Trebuchet MS', 'Verdana',
];

function populateFontSelect(families) {
  const select = document.getElementById('prop-font-family');
  select.innerHTML = '';
  for (const name of families) {
    const opt = document.createElement('option');
    opt.value = name;
    opt.textContent = name;
    select.appendChild(opt);
  }
}

async function initLocalFonts() {
  if (!window.queryLocalFonts) {
    populateFontSelect(FONT_FALLBACK);
    document.getElementById('font-api-note').classList.remove('hidden');
    return;
  }
  try {
    const fonts = await window.queryLocalFonts();
    const families = [...new Set(fonts.map(f => f.family))].sort();
    populateFontSelect(families.length > 0 ? families : FONT_FALLBACK);
  } catch (_err) {
    // Permission denied
    populateFontSelect(FONT_FALLBACK);
    document.getElementById('font-api-note').classList.remove('hidden');
  }
}

// === PLACEHOLDER — Google Fonts + custom font added in Task 2 ===
```

- [ ] **Step 2: Call `initLocalFonts()` from the existing `DOMContentLoaded` handler**

Inside `window.addEventListener('DOMContentLoaded', ...)` just before the closing `});`, add:

```js
  initLocalFonts();
```

- [ ] **Step 3: Verify local fonts in Chrome**

Open `index.html` in Chrome. Place a text element and select it — the Typography section should appear. The Font Family dropdown should show a browser permission prompt, then populate with your system fonts (typically 20–200+ entries). Reject the permission — the dropdown should fall back to the 8 hardcoded fonts and the note "Full system font access requires Chrome or Edge." should appear.

In Firefox or Safari, the note should appear immediately without any prompt.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: local font system via queryLocalFonts with fallback"
```

---

### Task 2: Font System — Google Fonts (Tier 2) + Custom Font (Tier 3)

Populate `#gfont-select` with the curated Google Fonts list and inject link tags on selection. Wire `#btn-apply-custom-font` to accept `<link>` tags, `@font-face` blocks, and bare URLs.

**Files:**
- Modify: `index.html` — replace `// === PLACEHOLDER — Google Fonts + custom font added in Task 2 ===`

- [ ] **Step 1: Replace the Google Fonts placeholder with Tier 2 + 3 font loading**

```js
// === GOOGLE FONTS + CUSTOM FONT ===

function initGoogleFontsSelect() {
  const select = document.getElementById('gfont-select');
  select.innerHTML = '<option value="">— Add a Google Font —</option>';
  for (const name of GOOGLE_FONTS) {
    const opt = document.createElement('option');
    opt.value = name;
    opt.textContent = name;
    select.appendChild(opt);
  }

  select.addEventListener('change', async () => {
    const name = select.value;
    if (!name) return;
    await loadGoogleFont(name);
    addFontToDropdown(name, 'Google');
    select.value = '';
    if (selectedIds.length > 0) {
      pushUndo();
      updateSelectedElements({ fontFamily: name });
      refreshPanel();
    }
  });
}

async function loadGoogleFont(name) {
  const id = `gfont-${name.replace(/\s+/g, '-').toLowerCase()}`;
  if (document.getElementById(id)) return; // already injected
  const url = `https://fonts.googleapis.com/css2?family=${encodeURIComponent(name)}:wght@400;700&display=swap`;
  const link = document.createElement('link');
  link.id = id;
  link.rel = 'stylesheet';
  link.href = url;
  document.head.appendChild(link);
  await document.fonts.ready;
}

// Add a font name to #prop-font-family if not already present, then select it.
function addFontToDropdown(name, tag) {
  const select = document.getElementById('prop-font-family');
  if (![...select.options].some(o => o.value === name)) {
    const opt = document.createElement('option');
    opt.value = name;
    opt.textContent = tag ? `${name} (${tag})` : name;
    select.insertBefore(opt, select.firstChild);
  }
  select.value = name;
}

// Tier 3 — Custom font
document.getElementById('btn-apply-custom-font').addEventListener('click', async () => {
  const raw = document.getElementById('custom-font-input').value.trim();
  const status = document.getElementById('custom-font-status');
  if (!raw) return;

  let node;
  let fontName = '';

  if (raw.startsWith('<link')) {
    // Extract href and inject a clean link tag
    const hrefMatch = raw.match(/href=['"]([^'"]+)['"]/i);
    if (!hrefMatch) { status.textContent = 'Could not find href in <link> tag.'; return; }
    node = document.createElement('link');
    node.rel = 'stylesheet';
    node.href = hrefMatch[1];
    // Try to extract font name from Google Fonts URL
    const familyMatch = hrefMatch[1].match(/family=([^&:+]+)/i);
    if (familyMatch) fontName = decodeURIComponent(familyMatch[1].replace(/\+/g, ' '));
  } else if (raw.includes('@font-face') || raw.includes('@import')) {
    node = document.createElement('style');
    node.textContent = raw;
    // Try to extract font-family name from @font-face block
    const faceMatch = raw.match(/font-family\s*:\s*['"]?([^'";\n]+)['"]?\s*;/i);
    if (faceMatch) fontName = faceMatch[1].trim().replace(/['"]/g, '');
  } else {
    // Treat as a bare URL — inject as a stylesheet link
    node = document.createElement('link');
    node.rel = 'stylesheet';
    node.href = raw;
  }

  try {
    document.head.appendChild(node);
    await document.fonts.ready;
    if (fontName) {
      addFontToDropdown(fontName, 'custom');
      status.textContent = `✓ "${fontName}" loaded.`;
      if (selectedIds.length > 0) {
        pushUndo();
        updateSelectedElements({ fontFamily: fontName });
        refreshPanel();
      }
    } else {
      status.textContent = '✓ Font injected — type the font-family name to use it.';
    }
  } catch (err) {
    document.head.removeChild(node);
    status.textContent = `Error: ${err.message}`;
  }
});
```

- [ ] **Step 2: Call `initGoogleFontsSelect()` from the `DOMContentLoaded` handler**

Inside `window.addEventListener('DOMContentLoaded', ...)`, directly after the `initLocalFonts()` call added in Task 1, add:

```js
  initGoogleFontsSelect();
```

- [ ] **Step 3: Verify Google Fonts in Chrome (requires internet connection)**

Open `index.html`, place a text element and select it. In the Typography section, the "Add a Google Font" dropdown should list all 32 fonts. Select "Pacifico" — a `<link>` should be injected into `<head>` and the text element should render in Pacifico font. Verify in DevTools → Elements → `<head>` that a `<link id="gfont-pacifico">` exists.

- [ ] **Step 4: Verify custom font input (Adobe Fonts or any CDN)**

Paste the following into the custom font textarea and click Apply:

```
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Lobster&display=swap">
```

Expected: status shows `✓ "Lobster" loaded.` and the font family dropdown now shows "Lobster (custom)" at the top.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: Google Fonts injection and custom font input"
```

---

### Task 3: PNG Export

Wire the Export PNG button to render the scene on a hidden full-resolution canvas and download it.

**Files:**
- Modify: `index.html` — replace `// === PLACEHOLDER — export added in Part C ===`

- [ ] **Step 1: Replace the export placeholder**

```js
// === PNG EXPORT ===

// Pre-warm the SVG image cache and wait for all images to finish loading.
// getSVGImage (Part A) is synchronous but image.onload fires async —
// this helper ensures they're fully decoded before drawScene runs.
function waitForSVGImages() {
  const svgEls = scene.elements.filter(el => el.type === 'svg' && el.svgSource);
  return Promise.all(svgEls.map(el => {
    const img = getSVGImage(el);
    if (!img) return Promise.resolve();
    if (img.complete && img.naturalWidth > 0) return Promise.resolve();
    return new Promise(resolve => {
      const prev = img.onload;
      img.onload = (...args) => { if (prev) prev(...args); resolve(); };
      img.onerror = resolve;
    });
  }));
}

document.getElementById('btn-export').addEventListener('click', async () => {
  const btn = document.getElementById('btn-export');
  btn.disabled = true;
  btn.textContent = 'Exporting…';

  try {
    await document.fonts.ready;
    await waitForSVGImages();

    const exportCanvas = document.createElement('canvas');
    exportCanvas.width  = scene.outputWidth;
    exportCanvas.height = scene.outputHeight;
    const ctx = exportCanvas.getContext('2d');
    drawScene(ctx, scene, { showDivider: false });

    const url = exportCanvas.toDataURL('image/png');
    const a = document.createElement('a');
    a.href = url;
    a.download = 'stereoforge-export.png';
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
  } finally {
    btn.disabled = false;
    btn.textContent = 'Export PNG';
  }
});
```

- [ ] **Step 2: Verify export in Chrome**

Open `index.html`, place a white rectangle and a text element with some text. Click Export PNG — a file `stereoforge-export.png` should download. Open it — it should show the stereo pair at full output resolution (default 1080×1350) with no divider line.

Check that the exported file dimensions match the preset: open DevTools console and run:

```js
scene.outputWidth   // e.g. 1080
scene.outputHeight  // e.g. 1350
```

Open the PNG in Finder → Get Info — width should be 1080px, height 1350px.

- [ ] **Step 3: Test export with an SVG element**

In the console, add a test SVG:

```js
scene.elements.push(createElement('svg', {
  x: scene.outputWidth / 4,
  y: scene.outputHeight / 2,
  width: 200, height: 200,
  svgSource: '<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 100 100"><circle cx="50" cy="50" r="45" fill="white"/></svg>',
}));
```

Click Export PNG — the white circle should appear in the exported image in both panels. If it's missing, it means the SVG image wasn't loaded before export, which would indicate a bug in `waitForSVGImages`.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: PNG export at full output resolution"
```

---

### Task 4: Project Save / Load

Wire Save (JSON download) and Load (file picker → parse → replace scene) to `#btn-save`, `#btn-load`, and `#load-file-input`.

**Files:**
- Modify: `index.html` — replace `// === PLACEHOLDER — save/load added in Part C ===`

- [ ] **Step 1: Replace the save/load placeholder**

```js
// === SAVE / LOAD ===

document.getElementById('btn-save').addEventListener('click', () => {
  const json = JSON.stringify(scene, null, 2);
  const blob = new Blob([json], { type: 'application/json' });
  const url  = URL.createObjectURL(blob);
  const a    = document.createElement('a');
  a.href = url;
  a.download = 'stereoforge-project.json';
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);
  URL.revokeObjectURL(url);
});

document.getElementById('btn-load').addEventListener('click', () => {
  document.getElementById('load-file-input').click();
});

document.getElementById('load-file-input').addEventListener('change', e => {
  const file = e.target.files[0];
  if (!file) return;
  const reader = new FileReader();
  reader.onload = ev => {
    try {
      const loaded = JSON.parse(ev.target.result);
      if (!loaded || typeof loaded !== 'object' || !Array.isArray(loaded.elements)) {
        alert('Invalid project file — expected a StereoForge JSON.');
        return;
      }
      pushUndo();
      scene.preset       = loaded.preset       || 'ig-portrait';
      scene.outputWidth  = loaded.outputWidth   || 1080;
      scene.outputHeight = loaded.outputHeight  || 1350;
      scene.bgColor      = loaded.bgColor       || '#000000';
      scene.elements     = loaded.elements;
      selectedIds = [];

      // Sync topbar UI to loaded state
      const presetSel = document.getElementById('preset-select');
      presetSel.value = scene.preset;
      if (scene.preset === 'custom') {
        document.getElementById('custom-w').style.display = 'inline-block';
        document.getElementById('custom-h').style.display = 'inline-block';
        document.getElementById('custom-w').value = scene.outputWidth / 2;
        document.getElementById('custom-h').value = scene.outputHeight;
      } else {
        document.getElementById('custom-w').style.display = 'none';
        document.getElementById('custom-h').style.display = 'none';
      }
      document.getElementById('bg-color-input').value = scene.bgColor;

      fitCanvas();
      refreshPanel();
    } catch (err) {
      alert('Could not load project: ' + err.message);
    }
  };
  reader.readAsText(file);
  e.target.value = ''; // allow re-loading the same file
});
```

- [ ] **Step 2: Verify save and load round-trip**

Open `index.html`, place two shapes with different depths and colors. Click Save — a `stereoforge-project.json` file should download. Inspect it in a text editor — it should be valid JSON with `preset`, `outputWidth`, `outputHeight`, `bgColor`, and `elements` keys.

Refresh the page (clears all state), then click Load and select the JSON file. The scene should restore with the same shapes, same colors, same depths.

- [ ] **Step 3: Verify undo after load**

After loading a project, press Cmd+Z — the scene should revert to the empty state before loading (because `pushUndo()` was called before applying the loaded state).

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: project save and load (JSON round-trip)"
```

---

### Task 5: Mobile Layout

Add responsive CSS for viewport < 768px. The left toolbar becomes a slide-up bottom drawer; the right panel becomes a bottom sheet that opens automatically when an element is selected.

**Files:**
- Modify: `index.html` — add HTML element, add CSS to `<style>`, add JS

- [ ] **Step 1: Add `#toolbar-tab` as the first child of `#toolbar`**

Find the opening `<div id="toolbar">` line in the HTML and add one line immediately after it:

```html
    <div id="toolbar">
      <div id="toolbar-tab">▲ Tools</div>
```

- [ ] **Step 2: Add mobile CSS at the end of the `<style>` block, just before `</style>`**

```css
    /* ── Mobile layout (< 768px) ── */
    #toolbar-tab { display: none; } /* hidden on desktop */

    @media (max-width: 767px) {
      #workspace { position: relative; }

      /* Canvas fills full width */
      #canvas-area { margin-bottom: 44px; }

      /* Toolbar: fixed bottom drawer, initially peeking with just the tab visible */
      #toolbar {
        position: fixed;
        bottom: 0; left: 0; right: 0;
        width: 100%;
        height: auto;
        max-height: 60vh;
        flex-direction: row;
        flex-wrap: wrap;
        justify-content: center;
        align-items: center;
        padding: 8px 6px 10px;
        border-right: none;
        border-top: 1px solid var(--border);
        z-index: 30;
        transform: translateY(calc(100% - 32px));
        transition: transform 0.22s ease;
        overflow-y: auto;
      }
      #toolbar.open { transform: translateY(0); }

      /* Tab visible above the collapsed toolbar */
      #toolbar-tab {
        display: flex;
        align-items: center;
        justify-content: center;
        position: absolute;
        top: -28px;
        left: 50%;
        transform: translateX(-50%);
        background: var(--panel);
        border: 1px solid var(--border);
        border-bottom: none;
        border-radius: 6px 6px 0 0;
        padding: 4px 16px;
        cursor: pointer;
        font-size: 9px;
        letter-spacing: 0.5px;
        color: var(--text-secondary);
        white-space: nowrap;
        z-index: 31;
      }

      /* Right panel: fixed bottom sheet */
      #right-panel {
        position: fixed;
        bottom: 32px; /* sit above toolbar tab */
        left: 0; right: 0;
        width: 100%;
        height: 55vh;
        max-height: 55vh;
        border-left: none;
        border-top: 1px solid var(--border);
        transform: translateY(100%);
        transition: transform 0.22s ease;
        z-index: 20;
        overflow-y: auto;
      }
      #right-panel.open { transform: translateY(0); }

      /* Close button for right panel on mobile */
      #panel-close-btn {
        display: flex;
        align-items: center;
        justify-content: center;
        position: sticky;
        top: 0;
        background: var(--panel);
        border-bottom: 1px solid var(--border);
        padding: 6px;
        cursor: pointer;
        font-size: 9px;
        color: var(--text-muted);
        z-index: 21;
      }

      .tool-btn { width: 36px; height: 36px; }
      .toolbar-sep { width: 1px; height: 24px; margin: 0 4px; }
    }
```

- [ ] **Step 3: Add `#panel-close-btn` as the first child of `#right-panel`**

Find `<div id="right-panel">` in the HTML and insert one line after it:

```html
    <div id="right-panel">
      <div id="panel-close-btn">▼ Close</div>
```

- [ ] **Step 4: Add mobile JS at the end of the `<script>` block, just before `</script>`**

```js
// === MOBILE LAYOUT ===

function isMobile() { return window.innerWidth < 767; }

function initMobile() {
  // Toolbar tab toggle
  document.getElementById('toolbar-tab').addEventListener('click', () => {
    const toolbar = document.getElementById('toolbar');
    const isOpen = toolbar.classList.toggle('open');
    document.getElementById('toolbar-tab').textContent = isOpen ? '▼ Tools' : '▲ Tools';
  });

  // Close button on right panel
  document.getElementById('panel-close-btn').addEventListener('click', () => {
    document.getElementById('right-panel').classList.remove('open');
  });

  // Auto-open right panel when element is selected (fires after all mouseup handlers)
  document.addEventListener('mouseup', () => {
    if (isMobile() && selectedIds.length > 0) {
      document.getElementById('right-panel').classList.add('open');
    }
  });

  // Close toolbar when a tool is selected
  document.querySelectorAll('.tool-btn[data-tool]').forEach(btn => {
    btn.addEventListener('click', () => {
      if (isMobile()) {
        document.getElementById('toolbar').classList.remove('open');
        document.getElementById('toolbar-tab').textContent = '▲ Tools';
      }
    });
  });
}

// Only run mobile init; desktop layout is handled by CSS
if (isMobile()) initMobile();

// Re-check on resize (handles orientation change)
window.addEventListener('resize', () => {
  if (!isMobile()) {
    document.getElementById('toolbar').classList.remove('open');
    document.getElementById('right-panel').classList.remove('open');
  }
});
```

- [ ] **Step 5: Verify mobile layout in Chrome DevTools**

Open Chrome DevTools → Toggle Device Toolbar (Cmd+Shift+M). Set viewport to 375×812 (iPhone). Verify:

1. The canvas fills the full width
2. A small tab "▲ Tools" is visible at the bottom of the screen
3. Clicking the tab reveals the full toolbar row
4. Clicking a shape tool collapses the toolbar and switches to crosshair cursor
5. Placing a shape causes the right panel sheet to slide up
6. Clicking "▼ Close" hides the right panel

Switch back to desktop viewport (> 768px) — the standard side-toolbar layout should be restored.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: mobile layout with bottom drawer toolbar and bottom sheet panel"
```

---

## Self-Review

### Spec coverage

| Spec requirement | Covered by |
|---|---|
| queryLocalFonts for system fonts | Task 1 |
| Fallback list if queryLocalFonts unsupported | Task 1 |
| Inline note "Full system font access requires Chrome or Edge" | Task 1 |
| Google Fonts curated list (~30) injected on demand | Task 2 |
| `document.fonts.ready` awaited before drawing | Task 2 (`loadGoogleFont`), Task 3 (`btn-export` handler) |
| Custom font via `<link>` / `@font-face` / kit URL | Task 2 |
| Custom font error message on failure | Task 2 |
| PNG export at `outputWidth × outputHeight` | Task 3 |
| Export uses hidden canvas, no divider in export | Task 3 (`showDivider: false`) |
| SVG elements drawn correctly in export | Task 3 (`waitForSVGImages`) |
| Project save → JSON blob download | Task 4 |
| Project load → file picker → parse → validate → replace scene | Task 4 |
| Load pushes to undo stack | Task 4 (`pushUndo()` before applying) |
| Mobile: toolbar → bottom drawer | Task 5 |
| Mobile: right panel → bottom sheet on selection | Task 5 |
| Mobile: canvas fills full width | Task 5 (CSS) |
| Output presets (dropdown wired) | Part A Task 2 ✓ |
| BG color picker wired | Part A Task 2 ✓ |
| Keyboard shortcuts | Part B Task 6 ✓ |

### Placeholder scan

No TBDs, TODOs, "fill in later", or "similar to Task N" patterns. Every step has exact code.

### Type consistency

- `populateFontSelect(families)` defined Task 1, called internally — ✓
- `addFontToDropdown(name, tag)` defined Task 2, called by Google Fonts handler and custom font handler — ✓
- `loadGoogleFont(name)` defined Task 2, called only within Task 2 scope — ✓
- `waitForSVGImages()` defined Task 3, calls `getSVGImage(el)` from Part A — ✓ (`getSVGImage` returns HTMLImageElement, not a Promise)
- `drawScene(ctx, scene, opts)` signature matches Part A definition — ✓
- `refreshPanel()` defined in Part B, called here in load handler and Google Fonts handler — ✓ (all Part C code runs after Part B in the single `<script>` block)
- `fitCanvas()` defined in Part A Task 3, called in load handler — ✓
- `selectedIds`, `pushUndo()`, `updateSelectedElements()` all from Part A/B — ✓
- `GOOGLE_FONTS` array defined in Part A Task 2 — ✓

---

**Plan complete and saved to `docs/superpowers/plans/2026-04-11-stereoforge-part-c.md`. Two execution options:**

**1. Subagent-Driven (recommended)** — Fresh subagent per task, review between tasks, fast iteration

**2. Inline Execution** — Execute tasks in this session using executing-plans, with checkpoints

**Which approach?**
