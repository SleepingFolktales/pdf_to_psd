# PDF-to-PSD Converter — Architecture & Structure
## ag-psd (JavaScript) · Single HTML File · v4 Pipeline

---

## Verdict: ✅ Feasible — production-grade with known UX caveat

A fully client-side, single-HTML-file PDF→PSD converter using **ag-psd** and **PDF.js**. No server, no install, no build step. Both libraries load from CDN.

---

## Library Assessment

### ag-psd (v30.1.0)

Actively maintained (637 stars, latest release March 2025). Most capable open-source PSD writer in JavaScript.

**Capabilities:**
- Write PSD files entirely in the browser
- Real TypeLayers via `layer.text` property (editable "T" layers in Photoshop)
- Shape Layers via `layer.vectorMask` + `layer.vectorFill` / `layer.vectorStroke`
- `transform`, `style.font`, `style.fontSize`, `style.fillColor` per text layer
- Rotated text via affine transform matrix `[cosA, sinA, -sinA, cosA, tx, ty]`
- Multi-span `styleRuns` for mixed colors/fonts within one block
- Outputs `ArrayBuffer` → downloadable with `FileSaver.js`
- CDN bundle: `unpkg.com/ag-psd/dist/bundle.js` (loads as `window.agPsd`)

**Text layer API:**
```js
{
  name: 'Text: Hello World',
  text: {
    text: 'Hello World\r',
    transform: [1, 0, 0, 1, left, top],
    style: {
      font: { name: 'ArialMT' },
      fontSize: 24,
      fillColor: { r: 255, g: 0, b: 0 },
    },
  },
  canvas: previewCanvas,  // pixel data for bounds (Direction A)
}
```

**Shape layer API (v4.0):**
```js
{
  name: 'Rect #ff0000 (200×100)',
  top, left, bottom, right,
  vectorMask: {
    paths: [{ open: false, knots: [
      { linked: true, points: [ny, nx, ny, nx, ny, nx] },  // [y,x] ordering
    ]}]
  },
  vectorFill: { type: 'color', color: { r: 255, g: 0, b: 0 } },
  vectorStroke: { strokeEnabled: true, lineWidth: { value: 3, units: 'Points' }, ... },
  canvas: previewCanvas,  // rasterized preview for thumbnail
}
```

**Known limits:**
- Text implementation "incomplete" in docs — but `text` property works reliably for writing from scratch
- Vertical text orientation: supported via rotation transform workaround (not native vertical mode)
- Does NOT pre-render text bitmap — triggers Photoshop "Update" prompt on open
- No 16bpc, no CMYK output — always RGB 8-bit; CMYK PDFs detected + user warned

### PDF.js (pdfjs-dist v3.11.174)

Mozilla's PDF.js — the standard for in-browser PDF rendering and text extraction.

**Text extraction** — `page.getTextContent()` returns items with:
- `str`, `transform` (position + font size), `width`, `height`, `fontName`, `dir`

**Image extraction** — `page.getOperatorList()` returns the full draw-call stream.
Walking this with CTM tracking reveals every embedded image's position and size.

**Vector extraction (v4.0)** — Same `getOperatorList()`, walked with full CTM accumulation
starting from the viewport transform. Save/restore depth tracking segments groups,
and each draw op (fill/stroke/image/text) snapshots `ctmAtDraw`.

**Page rendering** — `page.render()` renders full page to Canvas at any DPI.

**Coordinate system:** PDF.js uses top-left origin after viewport transform — maps directly to canvas/PSD pixel coordinates.

---

## v4 Architecture: Four-Phase Pipeline (updated v4.7)

The v4 pipeline fundamentally restructures PDF processing around the **operator list** (raw PDF drawing commands) rather than just text/image extraction. This enables:
- **Native PSD Shape Layers** with vectorMask (Bezier curves) instead of rasterized pixels
- **Per-draw-op CTM tracking** — each vector operation knows its exact transformation context
- **Clip path handling** — curved clips become vector masks; rectangular clips are content-area bounds
- **Unified z-order** — paint order from operator list determines layer stacking
- **Text color precision** — position-correlated matching from operator list instead of index-based

v4.5 reverts vectorMask knot ordering back to `[x,y]` (ag-psd API convention), replaces index-based text color mapping with position-correlated spatial matching, and hardens font name resolution.
v4.6 adds rect clip-fallback fix, line rasterization, circular image clip masks, merge gap/interleave guards, and stroke-only fill fixes.
v4.7 adds stroke lineWidth clamping (min 1.5px), multi-subpath pattern rasterization, transparent stroke outlines, progress bar rasterization, circular clip dot masks, center-alignment detection, and fakeBold fallback.

### Transform chain: `pixelCoord = viewportTransform × pageTransform × elementTransform × localCoord`

```
PDF File
  │
  ▼
┌──────────────────────────────────────────────────────┐
│  1. RENDER — Full-page raster at target DPI          │
│     page.render() → pristine bgCanvas                │
└──────────┬───────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────┐
│  2. EXTRACT TEXT — page.getTextContent()              │
│     Per-item: str, bx/by, fontName, fontSize, color  │
│     Rotation detection via viewport × item.transform  │
│     + Extract text colors from operator list          │
│     + Extract images via CTM-tracked operator walk    │
└──────────┬───────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────┐
│  PHASE 1: OPERATOR GROUP SEGMENTATION                │
│     segmentOperatorList(opList, viewportTransform)    │
│                                                      │
│  • CTM starts from viewportTransform (Fix 1)         │
│  • Accumulates transforms at ALL depths (0,1,2,3)    │
│  • Depth-2 save/restore = group envelope             │
│  • Depth-3 save/restore = sub-element boundary       │
│  • Each draw op snapshots ctmAtDraw (Fix 1)          │
│  • Sub-groups flushed at depth-3 restore (Fix 3)     │
│  • Each sub-group gets its own zIndex                │
│  → Emits: OperatorGroup[] with zIndex, drawOps,     │
│    clips, fillColor, strokeColor, graphicsState      │
└──────────┬───────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────┐
│  PHASE 2: GROUP CLASSIFICATION                       │
│     classifyAllGroups(groups, W, H)                  │
│                                                      │
│  • IMAGE — only image draw ops                       │
│  • TEXT — only text draw ops                         │
│  • BACKGROUND_FILL — >90% coverage, z=0, single fill│
│  • BUTTON_BACKGROUND — thick stroke + next is text   │
│  • VECTOR_COMPOUND — multiple non-text draw ops      │
│  • VECTOR_FILL — single fill/fillStroke              │
│  • VECTOR_STROKE — single stroke                     │
│  • computeGroupBounds uses per-draw-op ctmAtDraw     │
│  → Emits: ClassifiedGroup[] with layerType, bounds,  │
│    needsRasterization flag, zIndex                   │
└──────────┬───────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────┐
│  PHASE 3A: SHAPE LAYER CREATION (v4.4)               │
│     shouldUseShapeLayer() → createShapeLayer()       │
│                                                      │
│  Eligibility: any vector-only group with solid color │
│    — clips do not disqualify.                        │
│  • v4.4 Path source selection (INVERTED from v4.3):  │
│    PRIMARY: draw-op fill/stroke paths (actual shape) │
│    FALLBACK: curved clip paths ONLY when draw-ops    │
│      have no curves (progress bars = clip is pill)   │
│    NEVER: rectangular clips as vectorMask            │
│  • pdfPathToPsdVectorMask: PDF paths → ag-psd knots  │
│    - v4.5: [x,y] knot ordering (ag-psd API expects │
│      [x,y]; library handles PSD binary [y,x])       │
│    - Apply ctmAtDraw or clipTransform for abs pixels │
│  • vectorFill: { type:'color', color:{r,g,b} }      │
│  • vectorStroke: searches ALL draw ops, CTM-scaled   │
│  • Sidebar containers: stroke-only + clip → synth bg │
│  • Preview canvas via rasterizeForPreview()          │
│                                                      │
│  PHASE 3B: RASTERIZATION FALLBACK (remaining vectors)│
│     rasterizeForPreview() + rasterizeVectorGroup()   │
│                                                      │
│  • Per-draw-op CTM applied to each replay step       │
│  • Clips applied with their own clipTransform        │
│  • Produces isolated transparent pixel layer         │
└──────────┬───────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────┐
│  TEXT COLORS + IMAGE LAYERS + TEXT MERGE (v4.5)       │
│  • v4.5: Position-correlated color mapping:          │
│    - Track Tm/Td/TD/T* for showText canvas position  │
│    - Match each rawItem to nearest showTextColor     │
│      by spatial distance (threshold: 2× fontSize)   │
│  • Build image layers from extractedImgs + z-tag     │
│  • v4.4: Extract section boundaries from thin strokes│
│  • mergeBlocks always runs with guards:              │
│    - Pre-pass: same-line word coalescing             │
│    - Paragraph merge with v4.4 guards:               │
│      ④ fontName must match seed                      │
│      ⑤ fontSize ratio ≤ 1.10                        │
│      ⑥ color must match (RGB equality)              │
│      ⑧ no merge across section boundary lines       │
│    - Y-gap tightened: ≤ 1.8× fh (was 2.5×)         │
│  • v4.4: tWidth computed from item positions         │
│  • v4.4: canvas size = multi-line height, clamped    │
│  • Build TypeLayers with canvas + text metadata      │
│  • Tag all layers with _zIndex for sorting           │
└──────────┬───────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────┐
│  CLEAN BACKGROUND (if enabled)                       │
│  clearRect for text + image + vector regions         │
└──────────┬───────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────┐
│  UNIFIED Z-ORDER ASSEMBLY (Fix 2)                    │
│                                                      │
│  1. Collect all layers: text + vector + image        │
│  2. Sort by _zIndex ascending (paint order)          │
│  3. Reverse for PSD (first child = topmost)          │
│  4. Append Background as bottommost layer            │
│  5. writePsd({ noBackground: true })                 │
│                                                      │
│  → Layers interleaved by actual paint order, not     │
│    grouped by type. Matches visual stacking.         │
└──────────┬───────────────────────────────────────────┘
           │
           ▼
       PSD File
```

---

## Detailed Mathematical Formulas & Transformation Pipeline

### 1. Coordinate Transformation Chain

Every element in the PDF goes through a series of transformations to reach canvas pixel coordinates:

```
Local Coordinates (PDF path data)
  ↓
  × drawOpCTM (per-draw-op transformation matrix)
  ↓
Device Coordinates (PDF user space)
  ↓
  × viewportTransform (PDF.js viewport matrix)
  ↓
Canvas Pixel Coordinates (top-left origin, pixels)
```

**Affine Matrix Multiplication (2D homogeneous):**

For matrices M = [a, b, c, d, e, f] and N = [a', b', c', d', e', f']:

```
M × N = [
  a*a' + c*b',    b*a' + d*b',
  a*c' + c*d',    b*c' + d*d',
  a*e' + c*f' + e,  b*e' + d*f' + f
]
```

**Point Transformation:**

```
[x', y'] = M × [x, y]
         = [a*x + c*y + e, b*x + d*y + f]
```

### 2. CTM Tracking in segmentOperatorList()

The operator list parser maintains a **CTM stack** that mirrors PDF's graphics state save/restore:

```javascript
// Initial CTM = viewportTransform (Fix 1)
let currentCTM = viewportTransform.slice();
const ctmStack = [];

// On save (depth increases):
ctmStack.push(currentCTM.slice());

// On transform operator:
currentCTM = mul(currentCTM, transformArgs);

// On restore (depth decreases):
currentCTM = ctmStack.pop() || viewportTransform.slice();

// On each draw op:
drawOp.ctmAtDraw = currentCTM.slice();  // Snapshot for later use
```

**Critical Fix 1:** CTM is accumulated at **ALL depths** (0, 1, 2, 3), not just depth-2. This ensures nested transforms (e.g., a rotated shape inside a scaled group) are correctly applied.

### 3. Image Extraction via CTM

When the operator list encounters `paintImageXObject`, the image occupies the **unit square [0,1]×[0,1]** in the current CTM's coordinate space:

```
Image corners in local coords: [0,0], [1,0], [0,1], [1,1]

For each corner (lx, ly):
  devX = ctm[0]*lx + ctm[2]*ly + ctm[4]
  devY = ctm[1]*lx + ctm[3]*ly + ctm[5]
  
  canvasX = vpT[0]*devX + vpT[2]*devY + vpT[4]
  canvasY = vpT[1]*devX + vpT[3]*devY + vpT[5]

Bounds = [min(xs), min(ys), max(xs), max(ys)]
```

### 4. Text Color Tracking via Text Matrix

The operator list tracks text positioning through a **text matrix** (Tm) that changes with:
- `setTextMatrix(a,b,c,d,e,f)` — absolute position
- `moveText(tx, ty)` — relative offset
- `setLeadingMoveText(tx, ty)` — relative + set line leading
- `nextLine()` — move down by leading

For each `showText` call:

```
textMatrixPos = [Tm[4], Tm[5]]  // Translation component

devX = ctm[0]*Tm[4] + ctm[2]*Tm[5] + ctm[4]
devY = ctm[1]*Tm[4] + ctm[3]*Tm[5] + ctm[5]

canvasX = vpT[0]*devX + vpT[2]*devY + vpT[4]
canvasY = vpT[1]*devX + vpT[3]*devY + vpT[5]

showTextColors.push({x: canvasX, y: canvasY, r, g, b})
```

Later, each text item from `getTextContent()` is matched to the nearest `showTextColor` by spatial distance (threshold: `2 × fontSize²`).

### 5. Path-to-VectorMask Conversion (pdfPathToPsdVectorMask)

PDF paths are converted to PSD vectorMask knots using **Bezier curve representation**:

**ag-psd knot format:**
```
knot = {
  linked: true,
  points: [cp1x, cp1y, anchorx, anchory, cp2x, cp2y]
}
```
- `cp1` = incoming control point (before anchor)
- `anchor` = the point itself
- `cp2` = outgoing control point (after anchor)
- For straight lines: `cp1 = anchor = cp2`

**Path operation mapping:**

| PDF Op | Args | Conversion |
|--------|------|-----------|
| `moveTo` | (x, y) | Create new path, add straight knot at (x,y) |
| `lineTo` | (x, y) | Add straight knot at (x,y) |
| `curveTo` | (cp1x, cp1y, cp2x, cp2y, ex, ey) | Set prev knot's cp2, add new knot with cp1 |
| `curveTo2` | (cp2x, cp2y, ex, ey) | Quadratic: cp1 = current point, cp2 = arg, anchor = endpoint |
| `curveTo3` | (cp1x, cp1y, ex, ey) | Cubic: cp1 = arg, cp2 = endpoint, anchor = endpoint |
| `rectangle` | (x, y, w, h) | Create closed path with 4 straight knots (corners) |
| `closePath` | — | Mark path as closed (open=false) |

**CTM Application (v4.5 fix):**

```javascript
function toPx(lx, ly) {
  return {
    px: ctm[0]*lx + ctm[2]*ly + ctm[4],
    py: ctm[1]*lx + ctm[3]*ly + ctm[5]
  };
}

// For each knot point:
const {px, py} = toPx(localX, localY);
knot.points = [px, py, px, py, px, py];  // Straight knot
```

**v4.5 Critical Fix:** Use `[x, y]` order (ag-psd API expects this). The library internally converts to PSD binary `[y, x]` format. v4.4 incorrectly pre-swapped to `[y, x]`, causing double-swap → transposed shapes.

### 6. Group Bounds Calculation (computeGroupBounds)

For each draw op in a group, expand the bounding box by transforming all path points through `ctmAtDraw`:

```javascript
for (const drawOp of group.drawOps) {
  const ctm = drawOp.ctmAtDraw || [1,0,0,1,0,0];
  
  if (drawOp.type === 'image') {
    // Image occupies [0,1]×[0,1]
    expand(ctm, 0, 0);
    expand(ctm, 1, 0);
    expand(ctm, 0, 1);
    expand(ctm, 1, 1);
  }
  
  if (drawOp.pathData) {
    const {ops, coords} = drawOp.pathData;
    let ci = 0;
    for (const op of ops) {
      switch (op) {
        case PATH_OPS.rectangle:
          const [rx, ry, rw, rh] = [coords[ci++], coords[ci++], coords[ci++], coords[ci++]];
          expand(ctm, rx, ry);
          expand(ctm, rx+rw, ry);
          expand(ctm, rx, ry+rh);
          expand(ctm, rx+rw, ry+rh);
          break;
        case PATH_OPS.moveTo, PATH_OPS.lineTo:
          expand(ctm, coords[ci++], coords[ci++]);
          break;
        case PATH_OPS.curveTo:
          for (let j=0; j<3; j++) expand(ctm, coords[ci++], coords[ci++]);
          break;
        // ... etc
      }
    }
  }
  
  // For strokes, expand by half line width
  if (drawOp.type === 'stroke' || drawOp.type === 'fillStroke') {
    const lw = drawOp.lineWidth || 1;
    const scaleX = Math.sqrt(ctm[0]**2 + ctm[1]**2);
    const halfStroke = (lw * scaleX) / 2;
    minX -= halfStroke; minY -= halfStroke;
    maxX += halfStroke; maxY += halfStroke;
  }
}

return {
  left: Math.max(0, Math.floor(minX)),
  top: Math.max(0, Math.floor(minY)),
  width: Math.ceil(maxX) - left,
  height: Math.ceil(maxY) - top
};
```

### 7. Text Merging Guards (mergeBlocks)

**Pre-pass: Same-line word coalescing**

Sort items by Y, then X. Merge adjacent items on the same line if:
- Y difference < 2 px
- Same fontName, fontSize (±0.5), color, rotation
- X gap < 3× average character width

```javascript
const avgCharW = cur.width / (cur.str.length || 1);
const gap = nxt.bx - (cur.bx + cur.width);
if (gap < -avgCharW*0.5 || gap > avgCharW*3) break;  // Don't merge
```

**Paragraph merge with v4.6 guards:**

```javascript
// ① Same line: y-band within 65% font height, x adjacent
const sameLine = Math.abs(it.y - mnY) < fh*0.65 && 
                 it.x >= mnX - fh*0.5 && 
                 it.x <= mxX + fh*4.0;

// ② Next line: x-aligned within 120% fh, vertical gap ≤ 1.8×fh
const gapY = it.y - mxY;
const nextLine = Math.abs(it.x - mnX) < fh*1.2 && 
                 gapY >= -(fh*0.3) && 
                 gapY <= fh*1.8;

// ③ Rotation must match
if (!!it.isRotated !== !!seed.isRotated) continue;

// ④ Font name must match (v4.4 Fix 3)
if (it.fontName !== seed.fontName) continue;

// ⑤ Font size ratio ≤ 1.10 (same hierarchy level)
const sizeRatio = Math.max(it.fontSize, seed.fontSize) / 
                  Math.min(it.fontSize, seed.fontSize);
if (sizeRatio > 1.10) continue;

// ⑥ Color must match (v4.4 Fix 3)
if (it.color.r !== seed.color.r || 
    it.color.g !== seed.color.g || 
    it.color.b !== seed.color.b) continue;

// ⑦ Block size guards (prevent cross-column merges)
const nW = Math.max(mxX, it.x+it.width) - Math.min(mnX, it.x);
const nH = Math.max(mxY, it.y+it.height) - Math.min(mnY, it.y);
const maxW = canvasW * 0.45, maxH = canvasH * 0.30;
if (nW > maxW || nH > maxH) continue;

// ⑧ Section boundary check (v4.4 Fix 4)
if (crossesBoundary(newMnY, newMxY, newMnX, newMxX)) continue;

// ⑨ v4.6 P4: Consecutive baseline step ≤ 2× fontSize
if (nextLine) {
  const byStep = it.by - mxBy;
  if (byStep > seed.fontSize * 2.0) continue;
}

// ⑩ v4.6 P5: Heading-interleave guard
if (nextLine && it.by > mxBy) {
  let headingInGap = false;
  for (let hk=0; hk<sorted.length; hk++) {
    if (used[hk]) continue;
    const hd = sorted[hk];
    if (hd.fontName === seed.fontName) continue;
    if (hd.by <= mxBy || hd.by >= it.by) continue;
    if (hd.bx >= mxX || hd.bx+hd.width <= mnX) continue;
    headingInGap = true; break;
  }
  if (headingInGap) continue;
}
```

### 8. Shape Layer Path Source Selection (v4.4 Fix 1+C)

```javascript
let useCurvedClipFallback = false;

// Check if draw-op paths have curves
const drawOpHasCurves = group.drawOps.some(dop =>
  dop.pathData?.ops.some(op => 
    op === PATH_OPS.curveTo || 
    op === PATH_OPS.curveTo2 || 
    op === PATH_OPS.curveTo3
  )
);

// If no curves in draw-ops, check for curved clips
if (!drawOpHasCurves && group.clips.length > 0) {
  for (const clip of group.clips) {
    if (clip.ops?.some(op => 
        op === PATH_OPS.curveTo || 
        op === PATH_OPS.curveTo2 || 
        op === PATH_OPS.curveTo3)) {
      useCurvedClipFallback = true;
      break;
    }
  }
}

// v4.6 P1: Rect-only draw ops always use their own geometry
if (useCurvedClipFallback) {
  const drawOpsAreRectOnly = group.drawOps.every(dop =>
    dop.pathData?.ops.every(op => 
      op === PATH_OPS.rectangle || 
      op === PATH_OPS.closePath
    )
  );
  if (drawOpsAreRectOnly) useCurvedClipFallback = false;
}

// Use clip path or draw-op path accordingly
if (useCurvedClipFallback) {
  // Progress bar case: clip is rounded-rect, draw-op is plain rect
  for (const clip of group.clips) {
    const mask = pdfPathToPsdVectorMask(
      {ops: clip.ops, coords: clip.coords},
      clip.clipTransform || [1,0,0,1,0,0],
      docWidth, docHeight
    );
    allPaths.push(...mask.paths);
  }
} else {
  // Decorative shapes / simple rects: use draw-op paths
  for (const dop of group.drawOps) {
    if (dop.pathData) {
      const mask = pdfPathToPsdVectorMask(
        dop.pathData,
        dop.ctmAtDraw || ctm,
        docWidth, docHeight
      );
      allPaths.push(...mask.paths);
    }
  }
}
```

### 9. Stroke-Only Shape Handling (v4.6 P6)

```javascript
const isStrokeOnly = group.drawOps.every(d => d.type === 'stroke');

// Synthesize fill for sidebar containers (stroke-only + clip)
let syntheticFill = false;
if (!fillColor && bgColor) {
  if (isStrokeOnly && group.clips.length > 0) {
    fillColor = bgColor;
    syntheticFill = true;
  }
}

// Only add vectorFill if not stroke-only, or if synthesized
const useFill = !isStrokeOnly || syntheticFill;

if (useFill) {
  layer.vectorFill = {type: 'color', color: {r, g, b}};
}

// Set fillEnabled on vectorStroke
if (vectorStroke) {
  vectorStroke.fillEnabled = useFill;
}
```

### v4.6 vs v4.5 improvements

| Problem | v4.5 (broken) | v4.6 (fixed) |
|---------|---------------|--------------|
| Gold dot (19×19) expands to full canvas | Rect draw op inside curved parent clip → clip fallback chosen as vectorMask (1029×1433) | `createShapeLayer()`: rect-only draw ops force `useCurvedClipFallback=false`; use draw-op rect directly |
| Separator lines render as white fill | 2-point moveTo+lineTo groups → shape layer attempt → degenerate fill | `shouldUseShapeLayer()` returns `false` for 2-point stroke-only paths; routed to rasterization |
| Profile photo rectangular crop | Image layer built with no mask | `imgLayers` builder checks matching group clips; curved clip → `pdfPathToPsdVectorMask()` → `imgLayer.vectorMask` |
| Five percentage values merge into one block | `nextLine` gap threshold 1.8×fh too permissive (74px step with ~34px font passes) | `mergeBlocks()` tracks `mxBy`; `byStep = it.by - mxBy` must be ≤ `seed.fontSize * 2.0` |
| Experience body text merges across heading | Same-font body text on both sides of heading fools merge loop | `mergeBlocks()` scans for unmerged different-font items in the Y gap; rejects merge if heading found |
| "About Me" / "Experience" outlines white-filled | All shape layers unconditionally get `vectorFill`; stroke-only gets synthetic white fill | `isStrokeOnly` + `syntheticFill` flags: no `vectorFill` for standalone outlines; `fillEnabled:false` on vectorStroke |

### v4.5 vs v4.4 improvements

| Problem | v4.4 (broken) | v4.5 (fixed) |
|---------|---------------|--------------|
| VectorMask knot order | `[y,x]` — transposed in Photoshop | `[x,y]` — matches ag-psd API; library handles binary swap |
| Shape positions transposed | top↔left swapped in all shape layers | Correct positions matching Phase 3 log |
| Fill layer bounds explosion | Small rects expand to full-canvas bounds | Correct bounds from correct mask paths |
| Text color mapping | Index-based — breaks on non-linear paint order | Position-correlated spatial match via text matrix tracking |
| "Greta Mae" color | Black (rgb 0,0,0) | Gold (#a07f13) matched by canvas position |
| Font name fallback | Only checks `fontObj.name` | Also checks `fontObj.loadedName` for complete coverage |
| Decorative shapes use clip mask | Curved clip paths always preferred → full-canvas bounds | Draw-op paths preferred; curved clips only as fallback for progress bars |
| "Evans" merged with "Digital Marketing" | No font/size/color guards in paragraph merge | fontName match + fontSize ratio ≤ 1.10 + color equality required |
| "About Me" merged with paragraph text | No section boundary awareness | Thin vector strokes extracted as boundaries, merges blocked across them |
| Text canvas oversized for multi-line | Single-line height `fs*1.4` for all blocks | `fs * 1.4 * lineCount`, clamped to document dimensions |
| Y-gap too permissive | Next-line gap ≤ 2.5× font height | Tightened to ≤ 1.8× font height |
| tWidth not computed | Always fallback `fs * text.length * 0.6` | Measured from actual item positions in merged block |

### v4.3 vs v4.2 improvements

| Problem | v4.2 (broken) | v4.3 (fixed) |
|---------|---------------|---------------|
| Progress bars transparent | Groups with >1 clip forced to rasterization which failed | Clip paths used as vector masks in shape layers, bypassing rasterization |
| Unrelated groups merged | Clip-companion merge had no spatial overlap check | Added >50% overlap requirement |
| Text color wrong | Per-block tracking with consumed/fallback caused cascading errors | Per-showText emission — simple, direct fill color at each showText |
| Text merge optional | Toggle-gated, off by default | Always-on — word coalescing + paragraph grouping essential for output |
| Sidebar containers hollow | Stroke-only groups got black fill | Synthesize fill from page background color |
| Stroke descriptor missed | Only checked first draw op for strokes | Searches ALL draw ops in compound groups |

### v4 vs v3 improvements

| Problem | v3 (broken) | v4 (fixed) |
|---------|-------------|------------|
| Transform chain | CTM started from identity, missed viewport | CTM starts from viewportTransform, per-draw-op ctmAtDraw |
| Z-order | Layers grouped by type (text→vector→image) | Unified sort by paint order — interleaved correctly |
| Mixed-content groups | Button bg + label bundled into one group | Split at depth-3 boundaries into separate sub-groups |
| Vector layer type | Rasterized pixel layers only | Native PSD Shape Layers (vectorMask + vectorFill) with raster fallback |

---

## PSD Layer Structure (v4.0)

```
PSD File (width × height = canvas pixels at DPI)
│
│  ← Layers sorted by paint order (highest z-index first)
│
├── Rect #ff6b6b (200×50)          ← ShapeLayer (vectorMask + vectorFill + preview canvas)
├── T1: "JOIN NOW!"                ← TypeLayer (canvas + text metadata, editable)
├── Closed Path #4a2d5e (300×300)  ← ShapeLayer (bezier curves, solid fill)
├── T2: "GRAND"                    ← TypeLayer
├── T3: "opening"                  ← TypeLayer
├── IMG 1 (878×991)                ← Pixel layer (photo, cropped from pristine render)
├── Rect #5e3572 (1620×1620)       ← ShapeLayer or raster fallback (background fill)
│   ...
└── Background                     ← Pixel layer (full render with holes if erase enabled)
```

---

## Key Functions (v4.6)

### Phase 1: Operator List Segmentation

**`segmentOperatorList(opList, viewportTransform)`**

Parses the PDF operator list and emits operator groups with full CTM context.

- **Input**: `opList` (PDF.js operator list), `viewportTransform` (6-element affine matrix)
- **Output**: `OperatorGroup[]` with fields:
  - `opStart`, `opEnd` — operator indices
  - `zIndex` — paint order counter
  - `clips[]` — clip paths encountered before draw ops
  - `drawOps[]` — fill/stroke/image/text operations with `ctmAtDraw` snapshot
  - `fillColor`, `strokeColor`, `graphicsState` — current graphics state
  - `_currentSubDraw[]` — temporary buffer for depth-3 sub-elements

**Algorithm:**
1. Initialize `currentCTM = viewportTransform` (Fix 1)
2. Maintain `ctmStack` for save/restore pairs
3. Accumulate transforms at **all depths** (0, 1, 2, 3)
4. At depth-2 save: create new group envelope
5. At depth-3 save/restore boundaries: flush sub-group with new zIndex
6. For each draw op: snapshot `ctmAtDraw = currentCTM.slice()`
7. At depth-2 restore: finalize group

**Critical Fixes:**
- Fix 1: CTM starts from `viewportTransform`, not identity
- Fix 3: Sub-groups split at depth-3 boundaries (prevents button bg + label bundling)

---

**`flushSubGroup(group, groups, zCounter, ctm, fillColor, strokeColor, gState)`**

Emits a depth-3 sub-element as a separate OperatorGroup with its own zIndex.

- **Input**: Current group, accumulated draw ops, graphics state
- **Output**: Adds new OperatorGroup to `groups[]`, increments `zCounter`
- **Purpose**: Ensures nested elements (e.g., button background + text label) are classified separately

---

### Phase 2: Group Classification

**`classifyAllGroups(groups, canvasWidth, canvasHeight)`**

Analyzes each operator group and assigns a layer type.

- **Input**: `OperatorGroup[]`, canvas dimensions
- **Output**: `ClassifiedGroup[]` with fields:
  - `layerType` — IMAGE | TEXT | VECTOR_FILL | VECTOR_STROKE | VECTOR_COMPOUND | BUTTON_BACKGROUND | BACKGROUND_FILL
  - `bounds` — computed via `computeGroupBounds()`
  - `needsRasterization` — boolean flag
  - `group` — reference to original OperatorGroup
  - `zIndex` — preserved from operator group

**Classification Logic:**
```
if (only image ops) → IMAGE
else if (only text ops) → TEXT
else if (>90% coverage, z=0, single fill) → BACKGROUND_FILL
else if (thick stroke + next group is text) → BUTTON_BACKGROUND
else if (multiple non-text ops) → VECTOR_COMPOUND
else if (single fill/fillStroke) → VECTOR_FILL
else if (single stroke) → VECTOR_STROKE
else → VECTOR_FILL (default)
```

**Post-pass 1: Clip-Companion Merge**
- Find adjacent fill + stroke groups with matching clip paths
- Require >50% spatial overlap (v4.3 fix)
- Merge into single VECTOR_COMPOUND group

**Post-pass 2: Spatial Overlap Merge**
- Collect thin fill-only groups (height ≤ 60 px)
- Merge clusters with >70% overlap (progress bars, widgets)

---

**`computeGroupBounds(group, canvasWidth, canvasHeight)`**

Computes the bounding box of all draw ops using their per-op CTM.

- **Input**: OperatorGroup, canvas dimensions
- **Output**: `{left, top, width, height}` in canvas pixels
- **Algorithm**: For each draw op, transform all path points through `ctmAtDraw`, expand bounds
- **Stroke handling**: Expand by `(lineWidth × scaleX) / 2` where `scaleX = √(ctm[0]² + ctm[1]²)`

---

### Phase 3A: Shape Layer Creation

**`shouldUseShapeLayer(classifiedGroup)`**

Gates whether a group is eligible for native PSD shape layer.

- **Input**: ClassifiedGroup
- **Output**: Boolean
- **Criteria**:
  - Must have at least one fill or stroke with solid color
  - Must not contain text or image ops
  - v4.6 P2: Reject 2-point stroke-only paths (moveTo+lineTo only, no curves/rects)

**v4.6 P2 Logic:**
```javascript
const isLine = group.drawOps.every(dop => {
  const {ops} = dop.pathData;
  const hasCurveOrRect = ops.some(op => 
    op === PATH_OPS.curveTo || op === PATH_OPS.curveTo2 || 
    op === PATH_OPS.curveTo3 || op === PATH_OPS.rectangle
  );
  if (hasCurveOrRect) return false;
  const ptCount = ops.filter(op => 
    op === PATH_OPS.moveTo || op === PATH_OPS.lineTo
  ).length;
  return ptCount <= 2;
});
if (isLine) return false;  // Rasterize instead
```

---

**`pdfPathToPsdVectorMask(pathData, ctm, docWidth, docHeight)`**

Converts PDF path data to ag-psd vectorMask knots.

- **Input**: `{ops, coords}` (PDF path operations), `ctm` (transformation matrix), canvas dimensions
- **Output**: `{paths}` where each path is `{open, knots[]}`
- **Knot format**: `{linked: true, points: [cp1x, cp1y, anchorx, anchory, cp2x, cp2y]}`

**Path Operation Mapping:**

| Op | Local Coords | Conversion |
|----|--------------|-----------|
| `moveTo(x, y)` | (x, y) | New path, straight knot at (x, y) |
| `lineTo(x, y)` | (x, y) | Straight knot at (x, y) |
| `curveTo(cp1x, cp1y, cp2x, cp2y, ex, ey)` | 6 coords | Cubic Bezier: set prev cp2, add new knot with cp1 |
| `curveTo2(cp2x, cp2y, ex, ey)` | 4 coords | Quadratic: cp1 = current, cp2 = arg, anchor = endpoint |
| `curveTo3(cp1x, cp1y, ex, ey)` | 4 coords | Cubic: cp1 = arg, cp2 = endpoint, anchor = endpoint |
| `rectangle(x, y, w, h)` | 4 coords | Closed path with 4 corners |
| `closePath()` | — | Mark path as closed |

**CTM Application (v4.5 fix):**
```javascript
function toPx(lx, ly) {
  return {
    px: ctm[0]*lx + ctm[2]*ly + ctm[4],
    py: ctm[1]*lx + ctm[3]*ly + ctm[5]
  };
}
```

**v4.5 Critical Fix**: Use `[x, y]` knot order (ag-psd API). Library internally converts to PSD binary `[y, x]`. v4.4 pre-swapped → double-swap → transposed shapes.

---

**`createShapeLayer(classifiedGroup, docWidth, docHeight, bgColor)`**

Builds a PSD shape layer with vectorMask, vectorFill, and vectorStroke.

- **Input**: ClassifiedGroup, canvas dimensions, background color
- **Output**: Layer object with `vectorMask`, `vectorFill`, `vectorStroke`, preview canvas
- **Key decisions**:
  - Path source: draw-op paths (primary) vs curved clip fallback (v4.4 Fix 1+C)
  - v4.6 P1: Rect-only draw ops force use of draw-op geometry (not clip fallback)
  - v4.6 P6: Stroke-only shapes get `fillEnabled: false`, no `vectorFill`

**Path Source Selection (v4.4 Fix 1+C):**
```
if (draw-op has curves) → use draw-op paths
else if (draw-op has no curves AND curved clip exists) → use clip path (progress bar case)
else → use draw-op paths

v4.6 P1: if (draw-op is rect-only) → force use draw-op, ignore clip fallback
```

**Fill & Stroke:**
- `vectorFill`: `{type: 'color', color: {r, g, b}}` (omitted if stroke-only without synthetic fill)
- `vectorStroke`: Searches all draw ops, applies CTM-scaled line width, sets `fillEnabled` based on `useFill`

---

### Phase 3B: Rasterization Fallback

**`rasterizeForPreview(classifiedGroup, canvasWidth, canvasHeight)`**

Canvas-based replay of draw ops for preview/fallback rasterization.

- **Input**: ClassifiedGroup, canvas dimensions
- **Output**: Canvas element with isolated transparent layer
- **Algorithm**:
  1. Create canvas sized to group bounds + 2px padding
  2. Apply clip regions with their `clipTransform`
  3. For each draw op: set transform to `ctmAtDraw`, replay path ops, fill/stroke
  4. Return canvas

**Per-Draw-Op CTM Application:**
```javascript
ctx.setTransform(ctm[0], ctm[1], ctm[2], ctm[3], ctm[4]-offX, ctm[5]-offY);
```

---

**`rasterizeVectorGroup(classifiedGroup, canvasWidth, canvasHeight)`**

Wrapper that calls `rasterizeForPreview()` and returns positioned layer object.

---

### Phase 4: Text Assembly

**`extractPageImages(page, vp, canvasW, canvasH, pageNum)`**

Walks operator list to extract images and text colors via position-correlated tracking.

- **Input**: PDF page, viewport, canvas dimensions, page number
- **Output**: `{images, showTextColors}`

**Image Extraction:**
- For each `paintImageXObject`, `paintJpegXObject`, `paintInlineImageXObject`:
  - Image occupies unit square [0,1]×[0,1] in current CTM
  - Transform 4 corners through `viewport × CTM`
  - Bounds = min/max of transformed corners
  - Filter: skip if < 30px, > 90% canvas, or duplicate

**Text Color Tracking (v4.5):**
- Track text matrix (Tm) through `setTextMatrix`, `moveText`, `setLeadingMoveText`, `nextLine`
- For each `showText`/`showSpacedText`: compute canvas position via `viewport × CTM × Tm`
- Store `{x, y, r, g, b}` in `showTextColors[]`
- Later matched to text items by spatial proximity (threshold: `2 × fontSize²`)

---

**`mergeBlocks(items, canvasW, canvasH, sectionBoundaries)`**

Groups adjacent text items into logical blocks with comprehensive guards.

- **Input**: Raw text items, canvas dimensions, section boundaries (thin vector lines)
- **Output**: Merged text blocks with `{text, bx, by, tWidth, fontName, fontSize, color, isRotated, cosA, sinA}`

**Pre-pass: Same-line Word Coalescing**
- Sort by Y, then X
- Merge adjacent items if: Y diff < 2px, same font/size/color/rotation, X gap < 3× char width
- Inserts spaces when needed

**Paragraph Merge with v4.6 Guards:**

| Guard | Condition | Purpose |
|-------|-----------|---------|
| ① Same line | Y within 65% fh, X adjacent | Catch spaced words |
| ② Next line | X within 120% fh, Y gap ≤ 1.8×fh | Catch indented paragraphs |
| ③ Rotation | Must match | Keep vertical text separate |
| ④ Font name | Must match (v4.4) | Prevent cross-font merges |
| ⑤ Font size | Ratio ≤ 1.10 (v4.4) | Same hierarchy level |
| ⑥ Color | RGB equality (v4.4) | Prevent color breaks |
| ⑦ Block size | Width < 45% canvas, height < 30% canvas | Prevent cross-column merges |
| ⑧ Section boundary | No merge across thin vector lines (v4.4) | Respect dividers |
| ⑨ Baseline step | `byStep = it.by - mxBy ≤ 2× fontSize` (v4.6 P4) | Prevent wide-gap merges |
| ⑩ Heading interleave | No different-font items in Y gap (v4.6 P5) | Prevent merging across headings |

---

## Key Functions (v4.6)

| Function | Phase | Purpose |
|----------|-------|---------|
| `segmentOperatorList` | 1 | Walk operator list, emit sub-groups with ctmAtDraw per draw op |
| `flushSubGroup` | 1 | Emit depth-3 sub-element as separate OperatorGroup |
| `classifyAllGroups` | 2 | Classify groups, compute bounds, clip-companion merge, spatial overlap merge |
| `computeGroupBounds` | 2 | Bounding box from all draw ops using their ctmAtDraw |
| `shouldUseShapeLayer` | 3A | Gate: vector-only ops with solid colors, reject 2-point lines (v4.6 P2) |
| `pdfPathToPsdVectorMask` | 3A | PDF paths → ag-psd vectorMask knots ([x,y] order, v4.5 fix) |
| `createShapeLayer` | 3A | v4.6: draw-op paths primary, rect-only force (P1), stroke-only no fill (P6) |
| `rasterizeForPreview` | 3B | Canvas replay with per-draw-op CTM for preview/fallback |
| `rasterizeVectorGroup` | 3B | Wrapper returning positioned canvas for pixel layers |
| `extractPageImages` | — | Image extraction + per-showText color extraction (v4.5 position-correlated) |
| `mergeBlocks` | text | v4.6: word coalescing + paragraph merge with 10 guards (P4, P5 new) |

---

## Guards & Warnings

| Guard | Trigger | Action |
|-------|---------|--------|
| Canvas size cap | Any dimension > 15,000 px | Auto-scale down, log warning |
| PSD format limit | Any dimension > 30,000 px | Log error (Photoshop will reject) |
| CMYK detection | `/DeviceCMYK` in raw PDF bytes | Show amber notice — output is RGB |
| Image dedup | Near-identical bounds (< 5 px) | Skip duplicate extraction |
| Full-page image | Image covers > 90 % of canvas | Skip (it's a background fill) |
| Tiny image | Image < 30 px on either side | Skip (bullet, icon, pattern tile) |
| Shape layer fallback | Mixed content (text/image) or no solid color | Fall back to rasterized pixel layer |

---

## Risk Table

| Risk | Severity | Mitigation |
|------|----------|------------|
| Photoshop update prompt on open | Low | Expected behavior, one click, documented |
| Font not installed on user's machine | Medium | Photoshop asks for substitution |
| PDF.js worker blocked on `file://` | Low | Serve from `localhost` or inline Blob URL |
| Complex multi-column layouts | Medium | Merge guards prevent cross-column; tunable |
| Very large PDFs (10+ pages) | Low | Page-by-page processing, progress bar |
| Canva font subsets (`ABCDEF+`) | Low | Strip prefix, map PostScript names |
| ag-psd vectorMask compatibility | Medium | Preview canvas ensures visual correctness even if vector data misinterpreted |

---

## Font Name Resolution

PDF.js internal names → system names:
- Strip subset prefix: `'ABCDEF+Montserrat-Bold'.split('+').pop()`
- Map PostScript: `ArialMT` → `Arial`, `TimesNewRomanPSMT` → `Times New Roman`
- `resolveFont()` extracts real PostScript names from `page.commonObjs` after render

---

## CDN Dependencies

```
pdfjs-dist@3.11.174  — PDF parsing + rendering
ag-psd@30.1.0        — PSD file writing
FileSaver.js@2.0.5   — Browser file download
```

All loaded from jsdelivr with unpkg fallback. No build step required.
