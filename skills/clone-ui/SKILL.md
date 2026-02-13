---
name: clone-ui
description: Clone a live web page into Figma. Use when the user wants to replicate a web page design in Figma, or asks to "clone this page to Figma" or "recreate this UI in Figma".
argument-hint: [url]
allowed-tools: chrome-devtools_*, Read, Write, Bash
---

Clone the page at $ARGUMENTS into the active Figma file using the Figma Plugin API.

You have access to Chrome DevTools Protocol MCP tools (`chrome-devtools_*`).
Use `chrome-devtools_list_pages` to find tabs, `chrome-devtools_select_page` to switch between them, `chrome-devtools_evaluate_script` to run JS in either the live page or the Figma tab, and `chrome-devtools_take_screenshot` for captures.

Follow this exact workflow:

### PHASE 1: Measure Everything First

Before writing any Figma code, select the live page tab with `chrome-devtools_select_page` and extract every measurement using `chrome-devtools_evaluate_script` with `getBoundingClientRect()`:

1. Page dimensions: `window.innerWidth`, `window.innerHeight`, `devicePixelRatio`
2. Major layout regions (sidebar, header, content area): exact x, y, width, height
3. Background colors: `getComputedStyle(el).backgroundColor` for every region
4. Typography: font-family, font-weight, font-size, color for every text style
5. Table/grid structure: column widths, row heights, cell padding, header vs body
6. Every text element: exact position via `getBoundingClientRect()`, content via `textContent`
7. Border/divider colors, widths, radii
8. Icon positions and sizes (note which are SVGs — these will be approximated)

IMPORTANT: Use `el.getAttribute('class') || ''` instead of `el.className` —
SVG elements return SVGAnimatedString for className, which breaks `.substring()`.

Store all measurements as structured data before moving to Phase 2.

### PHASE 2: Build in Figma (Single Pass)

Switch to the Figma tab with `chrome-devtools_select_page`. Execute all Figma Plugin API code via `chrome-devtools_evaluate_script`, wrapped in `(async () => { ... })()` — top-level `await` is not supported in the Figma plugin eval context.

Build order:
1. Main frame: `figma.createFrame()`, set exact page dimensions, `layoutMode = 'NONE'`
2. Background fills for each region
3. Sidebar elements (logo, nav, avatar) — top to bottom
4. Content area elements (title, table/grid) — top to bottom
5. Table internals: header row → filter row → border → data rows

Key Figma API rules:
- ALWAYS `await figma.loadFontAsync({family, style})` before creating/modifying text
- Strokes do NOT accept an `a` (alpha) property — use `opacity` on the node instead
- `strokesTop` is not a valid property — use a 1px rectangle as a divider instead
- `layoutMode = 'NONE'` on parent frames for absolute positioning of children
- Set `clipsContent = true` on the table frame

### PHASE 3: Set Up DIFFERENCE Overlay for Comparison

1. Start a CORS HTTP server to serve screenshots to Figma:
   ```python
   # /tmp/serve_screenshot.py
   from http.server import HTTPServer, SimpleHTTPRequestHandler
   import os
   os.chdir('/tmp')
   class CORSHandler(SimpleHTTPRequestHandler):
       def end_headers(self):
           self.send_header('Access-Control-Allow-Origin', '*')
           self.send_header('Access-Control-Allow-Methods', 'GET')
           self.send_header('Access-Control-Allow-Headers', '*')
           super().end_headers()
       def do_OPTIONS(self):
           self.send_response(200)
           self.end_headers()
   HTTPServer(('', 9876), CORSHandler).serve_forever()
   ```

2. Switch to the live page tab and take a full-page screenshot via `chrome-devtools_take_screenshot`, save to /tmp/page-ref.png

3. Switch to the Figma tab and run `chrome-devtools_evaluate_script` to fetch the image and create an overlay rectangle:
   ```javascript
   (async () => {
     const resp = await fetch('http://localhost:9876/page-ref.png');
     const buf = await resp.arrayBuffer();
     const img = figma.createImage(new Uint8Array(buf));
     const overlay = figma.createRectangle();
     overlay.name = 'OVERLAY_REF';
     overlay.resize(frameWidth, frameHeight);
     overlay.fills = [{ type: 'IMAGE', imageHash: img.hash, scaleMode: 'FILL' }];
     overlay.blendMode = 'DIFFERENCE';
     mainFrame.appendChild(overlay);
   })()
   ```

4. Register window-level toggle utilities:
   ```javascript
   (async () => {
     window.toggleOverlay = () => { overlay.visible = !overlay.visible; };
     window.setOverlayOpacity = (v) => { overlay.opacity = v; };
     window.overlayNormal = () => { overlay.blendMode = 'NORMAL'; };
     window.overlayDiff = () => { overlay.blendMode = 'DIFFERENCE'; };
   })()
   ```

Reading the overlay:
- **Black pixels** = perfect match
- **Bright/white pixels** = difference (misalignment, wrong color, missing element)
- Use `overlayNormal()` at 50% opacity to see both layers simultaneously
- Use `overlayDiff()` at 100% to spot even 1px offsets

### PHASE 4: Iterate with DIFFERENCE Overlay

Use `chrome-devtools_take_screenshot` on the Figma tab to see the overlay, then zoom into bright areas. For each:
1. Identify the Figma node causing the mismatch
2. Switch to the live page tab, measure the correct value via `chrome-devtools_evaluate_script`
3. Switch back to Figma tab, fix the node via `chrome-devtools_evaluate_script`
4. Re-screenshot and re-check

Common fixes needed:
- Text y-positions off by 0.5px (use fractional values like 13.5)
- Background colors slightly wrong (always use getComputedStyle, not guessing)
- Borders/dividers using wrong color or missing entirely
- Corner radii wrong
- Element colors wrong (verify SVG fill attributes, not just CSS)

### PHASE 5: Componentize Repeated Elements

After pixel-perfect alignment, refactor repeated patterns into Figma components:

1. **Best candidates**: Data table rows (N identical layouts), nav items, list items
2. **Poor candidates**: Elements with varying widths/positions (headers, filters) — keep these as absolute-positioned nodes

Component creation pattern:
```javascript
(async () => {
  const comp = figma.createComponent();
  comp.name = 'TableDataRow';
  comp.resize(width, height);
  comp.layoutMode = 'NONE';
  comp.fills = [];
  // Add children with positions relative to component...
  // Then create instances:
  const inst = comp.createInstance();
  inst.name = 'Row_0';
  inst.x = 0; inst.y = startY;
  // Override text content:
  inst.findOne(n => n.name === 'Label').characters = 'actual text';
  parentFrame.appendChild(inst);
})()
```

CRITICAL component rules:
- You CANNOT override child positions (x, y) in component instances
- You CAN override: text content (.characters), fills, visibility, opacity
- Auto-layout components resize well BUT text positions shift unpredictably — only use auto-layout if children genuinely flow (e.g., icon + label + spacer)
- For varied-width elements (table columns), absolute positioning beats auto-layout
- Group component definitions into a "-- Components --" frame off to the side

### Known Limitations (Don't Waste Time On These)

- SVG logos/icons → use text approximation or gray placeholder rectangles
- Font icon libraries (e.g., AG Grid's agGridQuartz) → approximate with unicode (›, ⋮, ≡)
- Sub-pixel text rendering → ~1px ghosting in DIFFERENCE is inherent, not fixable
- Retina screenshots at 2x DPR → Figma handles this correctly via IMAGE fill scaleMode

---

## Quick Reference: Figma Plugin API Gotchas

| Mistake | Fix |
|---|---|
| `await` at top level | Wrap in `(async () => { ... })()` |
| `node.strokes = [{color: {r,g,b,a}}]` | Remove `a`, use `node.opacity` instead |
| `node.strokesTop = ...` | Not a real property — use a 1px rect divider |
| `el.className.substring(...)` in CDP | Use `el.getAttribute('class') \|\| ''` (SVG compat) |
| Auto-layout + `SPACE_BETWEEN` for headers | Use `layoutMode = 'NONE'` for varied-width cells |
| Setting `child.x` on instance children | Not allowed — bake positions into the component definition |
| Calling `figma.loadFontAsync` once | Must call before EVERY text creation/modification |
| `window.toggleOverlay()` stops working | Re-register `window.*` functions after page reload — node refs go stale |
