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

## v4 Architecture: Four-Phase Pipeline (updated v4.4)

The v4 pipeline fixes three critical bugs from v3 and adds native PSD Shape Layer support.
v4.4 fixes vectorMask coordinate order, inverts path source selection (draw-op first, clip fallback), adds font-homogeneity/color/section-boundary guards to text merge, and clamps text canvas sizes.

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
│    - v4.4: [y,x] knot ordering (was [x,y] in v4.3) │
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
│  TEXT COLORS + IMAGE LAYERS + TEXT MERGE (v4.4)       │
│  • Per-showText color emission (not per-block)       │
│  • Apply op-list colors to text items by order       │
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

### v4.4 vs v4.3 improvements

| Problem | v4.3 (broken) | v4.4 (fixed) |
|---------|---------------|---------------|
| VectorMask coordinates swapped | Knot points written as `[x,y]` | Corrected to `[y,x]` per ag-psd convention |
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

## Key Functions (v4.4)

| Function | Phase | Purpose |
|----------|-------|---------|
| `segmentOperatorList` | 1 | Walk operator list, emit sub-groups with ctmAtDraw per draw op |
| `flushSubGroup` | 1 | Emit depth-3 sub-element as separate OperatorGroup |
| `classifyAllGroups` | 2 | Classify groups, compute bounds, clip-companion merge, spatial overlap merge |
| `computeGroupBounds` | 2 | Bounding box from all draw ops using their ctmAtDraw |
| `shouldUseShapeLayer` | 3A | Gate: vector-only ops with solid colors, clips allowed |
| `pdfPathToPsdVectorMask` | 3A | PDF paths → ag-psd vectorMask knots (v4.4: absolute pixels, [y,x] order) |
| `createShapeLayer` | 3A | v4.4: draw-op paths primary, curved clip fallback; vectorMask/Fill/Stroke + preview |
| `rasterizeForPreview` | 3B | Canvas replay with per-draw-op CTM for preview/fallback |
| `rasterizeVectorGroup` | 3B | Wrapper returning positioned canvas for pixel layers |
| `mergeBlocks` | text | v4.4: word coalescing + paragraph merge with font/size/color/boundary guards |
| `extractPageImages` | — | Image extraction + per-showText color extraction from operator list |

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
