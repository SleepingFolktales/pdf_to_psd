# PDF → PSD Converter — Dev Status

## Current Version: v3.0 (Three-Phase Pipeline — Operator Segmentation → Classification → Canvas Replay Rasterization)

---

## Known Issues

### � P0 — Text layers have zero bounds (RESOLVED in v2.6)
- **Status**: RESOLVED — v2.6 Direction A fix implemented. Text layers now have non-zero bounds and are visible in Photoshop.
- **Fix**: Each text block renders to an offscreen canvas, providing pixel data that ag-psd uses for bounds calculation while preserving TypeLayer metadata for editability.

---

## P0 Deep Analysis (v2.5 post-mortem — Mar 14 2026)

Test file: "Abstract Grand Opening Announcement Free Instagram Post.pdf" (Canva, 810×810pt, 1620×1620px @144DPI)

### Discrepancy 1: Our bounds are written but Photoshop reads zero

**What our debug log shows (non-zero bounds set on TypeLayer objects):**
```
[0] TypeLayer: "T1: opening"     bounds=(914,387)→(1482,591)  transform=[1,0,0,1,914,546]
[1] TypeLayer: "T2: GRAND"       bounds=(617,148)→(1491,421)  transform=[1,0,0,1,618,361]
[2] TypeLayer: "T3: GRAND"       bounds=(608,138)→(1482,411)  transform=[1,0,0,1,608,351]
[3] TypeLayer: "T4: Will Be…"    bounds=(901,703)→(1482,802)  transform=[1,0,0,1,901,780]
[4] TypeLayer: "T5: really…"     bounds=(978,1251)→(1482,1316) transform=[1,0,0,1,978,1302]
[5] TypeLayer: "T6: 10 July…"    bounds=(697,808)→(1482,975)  transform=[1,0,0,1,698,938]
[6] TypeLayer: "T7: JOIN NOW!"   bounds=(1154,1143)→(1439,1199) transform=[1,0,0,1,1154,1187]
```

**What PSD manifest returns (ALL zero for every text layer):**
```
T1: bounds={height:0, left:0, top:0, width:0}
T2: bounds={height:0, left:0, top:0, width:0}
... (all 7 text layers identical)
```

**Conclusion:** We set `top/left/bottom/right` on each TypeLayer JS object, but the PSD binary either:
- (a) ag-psd ignores `top/left/bottom/right` for layers with `text` property (TypeLayers) — it only uses these for pixel layers with `canvas` data, OR
- (b) `invalidateTextLayers: true` tells Photoshop to discard stored bounds and re-render the text — and since the font is missing, re-rendering produces empty bounds.
- **(Most likely: BOTH a and b are true.)**

### Discrepancy 2: Operator-extracted bounds are in WRONG coordinate space

**getTextContent positions (correct — verified visually):**
```
T1 "opening":        pos=(914, 546)    ← upper-right, matches visual
T2 "GRAND":          pos=(618, 361)    ← upper-center, matches visual
T7 "JOIN NOW!":      pos=(1154, 1187)  ← lower-right, matches visual
```

**Operator-extracted bounds (WRONG — completely different locations):**
```
T1 "opening":        bounds=(863, 1159)→(1450, 1394)  ← lower area?!
T2 "GRAND":          bounds=(367, 996)→(1155, 1312)   ← middle area?!
T7 "JOIN NOW!":      bounds=(38, 1523)→(200, 1588)    ← far bottom-left?!
```

**Root cause: Missing CTM (Current Transform Matrix) in coordinate conversion.**

The operator trace shows text is drawn inside nested `save/transform/restore` groups:
```
[108] SAVE (depth=3)
[109] TRANSFORM: [3.125, 0, 0, 3.125, 555.879, 760.136]   ← LOCAL CTM
...
[119] SET TEXT MATRIX: [1, 0, 0, -1, 431.531, 121.000]      ← LOCAL coords
```

The `setTextMatrix` args (431.531, 121.000) are in LOCAL space (relative to the CTM at [109]).
Our code applies ONLY the viewport transform: `canvas = viewport × localCoords` ❌
Correct calculation: `canvas = viewport × CTM × localCoords` ✓

**Proof for T1 "opening":**
- Base transform [1]: `[0.240, 0, 0, -0.240, 0, 817.920]`
- Local CTM [109]: `[3.125, 0, 0, 3.125, 555.879, 760.136]`
- Composed CTM = base × local = `[0.75, 0, 0, -0.75, 133.411, 635.487]`
- Text matrix e,f = (431.531, 121.000)
- Page coords = CTM × textPos = **(457.059, 544.737)** ← matches getTextContent transform[4,5] EXACTLY
- Canvas coords = viewport × pageCoords = **(914.1, 546.4)** ← matches our pos=(914,546) ✓
- Our WRONG calc: viewport × textPos directly = (863, 1394) ❌

**Conclusion:** The operator bounds feature extracts systematically wrong coordinates. `getTextContent` already provides correct page-space coordinates — the operator bounds extraction is redundant AND broken.

### Discrepancy 3: Font names make TypeLayers un-renderable

**All 7 text layers report `fontAvailable: false`** with `fontName: "sans-serif"`.

"sans-serif" is a CSS generic family name, NOT a Photoshop font. Photoshop cannot find it, cannot render the text, and therefore reports zero bounds — even if the PSD file contained correct bounds.

This is a **fundamental blocker** for TypeLayer rendering in Photoshop. The actual fonts used in this PDF are likely Montserrat or similar (Canva design), but PDF.js `getTextContent()` only returns the CSS fallback family.

### Discrepancy 4: Colors are actually CORRECT (no issue)

**Our code sets:** rgb(94,55,109) for T1
**PSD manifest shows:** rgb(red=12079, green=7067, blue=14006)
**Verification:** 12079/32768×255 = 94.0, 7067/32768×255 = 54.9, 14006/32768×255 = 108.9 ✓

Photoshop API uses 16-bit color space (0–32768). All colors are correct after conversion. The operator-list color extraction (v2.4) is working properly.

### Discrepancy 5: Background still contains text (by design, not a bug)

With `erase=false`, the background is the full `page.render()` composite. This is expected.
With `erase=true`, `clearRect` cuts transparent holes — but since text layers are invisible (zero bounds), the holes would show through as transparent gaps with nothing underneath.

**This creates a chicken-and-egg problem:** Erase only works if text layers render. Text layers only render if fonts are available. Fonts aren't available because we use CSS generic names.

### Photoshop Screenshot Analysis

Looking at the Photoshop screenshot with all layers visible:
- **Background** (pixel): 1620×1620, shows full design including text ← text is baked in here
- **IMG 1** (pixel): 878×991 at (0,629) — correct bounds ✓
- **IMG 2** (pixel): 734×856 at (67,705) — correct bounds ✓
- **T1–T7** (TypeLayer): All zero bounds — text is INVISIBLE, what user sees is from Background

When user hides Background layer, text disappears entirely because TypeLayers render nothing.

---

### P0 Root Cause Summary (3 independent problems)

| # | Problem | Severity | Can fix? |
|---|---------|----------|----------|
| A | **ag-psd ignores explicit bounds on TypeLayers** — only pixel layers (with `canvas`) get real bounds written to PSD binary | CRITICAL | Yes — provide canvas data alongside text property |
| B | **`invalidateTextLayers: true` + missing font = zero bounds** — Photoshop re-renders text, can't find font, gets nothing | CRITICAL | Yes — set to false AND provide rasterized preview |
| C | **Font names are CSS generics ("sans-serif")** — Photoshop can't render | HIGH | Partial — extract real names from PDF font dictionary |

### P0 Fix Directions (research — no code yet)

**Direction A: Rasterized text layers with TypeLayer metadata (RECOMMENDED)**
- For each text item, render the string to a small off-screen `<canvas>` using browser's Canvas 2D API
- Set this canvas as the layer's `canvas` property (pixel data for bounds + preview)
- ALSO set the `text` property (TypeLayer metadata for editability)
- ag-psd will write the pixel data for bounds AND the text descriptor for editing
- Photoshop opens with visible text (from pixels) and editable text (from TypeLayer)
- When user installs correct font + clicks "Update", Photoshop re-renders crisply
- **This is how professional PSD generators work.**

**Direction B: Pure pixel text layers (simpler, not editable)**
- Render text to canvas, use as pixel layers only (no `text` property)
- Text is visible with correct bounds but NOT editable in Photoshop
- Simplest implementation but loses key feature

**Direction C: Fix font names + hope user has fonts installed**
- Extract real font names from PDF font dictionary (`page.commonObjs`)
- If user has the font installed, TypeLayers render correctly
- If not, same zero-bounds problem
- Fragile — depends on user's font library

**Direction D: Combination (A + C)**
- Use Direction A for guaranteed visible text + Direction C for better font matching
- Best user experience: text always visible, editable with correct font if installed

### 🟡 P1 — Font names often resolve to "sans-serif" (PARTIALLY RESOLVED in v2.6)
- **Symptom**: PDF.js returns generic style names (`g_d0_f1`, `g_d0_f2`, etc.) with `fontFamily: "sans-serif"` for all styles. Actual font names (e.g., Montserrat, Poppins) are lost.
- **Root cause**: PDF.js `getTextContent()` returns style objects with the CSS fallback family, not the original PDF font name. The actual font name is embedded in the PDF font dictionary but not exposed through this API.
- **Impact**: PSD TypeLayers often use "sans-serif" → Photoshop substitutes with a default font.
- **v2.6 Fix**: Added `resolveFont()` helper that attempts to extract real PostScript font names from `page.commonObjs` after `page.render()`, falling back to `normFont()` for CSS generic names. This improves font matching for many PDFs but doesn't resolve all cases.

### � P2 — Text colors now extracted from operator list (FIXED)
- **Status**: RESOLVED in v2.4
- **What was wrong**: When "Sample text colors" is OFF, all text defaulted to `rgb(0,0,0)`. The correct colors were in the PDF operator list (e.g., plum rgb(94,55,109), coral rgb(255,115,130), white rgb(255,255,255)) but never extracted.
- **Root cause**: Color extraction only happened via canvas sampling when `doColors=true`. The operator list colors were logged but not applied to text items.
- **Fix**: `extractPageImages()` now does a parallel pass during operator loop: tracks `setFillRGBColor`/`setFillGray`/`setFillCMYKColor` state and records fill color for each `beginText…endText` block. Returns `{images, textColors}` instead of just images. Pipeline now applies op-list colors to rawItems by order index (PDF rendering order = getTextContent order). The "Sample text colors" toggle now acts as a canvas override on top of op-list colors, not the only source.
- **Result**: Text layers now have actual colors (plum, coral, white, etc.) instead of all-black, regardless of toggle state.

### 🟢 P3 — Debug logging comprehensive (FIXED)
- **Status**: RESOLVED in v2.3
- **What was added**: PDF document metadata (title, author, creator, dates, format version), page-level properties (dimensions in points/inches/mm, rotation, userUnit), full operator trace (every save/restore, transform, color change, font set, showText, path, image paint, gstate, marked content).

---

## Toggles Status (Default State)

| Toggle | ID | Default | Notes |
|--------|-----|---------|-------|
| Clean up background | `opt-erase` | **OFF** | Can now be enabled — P0 fixed in v2.6 |
| Sample text colors | `opt-colors` | **OFF** | User preference for testing |
| Merge adjacent text blocks | `opt-merge` | **OFF** | User preference for testing |

---

## What We Did (Changelog)

### v3.0 — Three-Phase Vector Layer Extraction Pipeline
- **Architecture**: New three-phase pipeline walks the PDF.js operator list, segments it into discrete visual groups by save/restore depth, classifies each group, and rasterizes vector groups to isolated transparent canvases
- **Phase 1 — Operator Group Segmentation** (`segmentOperatorList`): Single-pass scan of `fnArray`/`argsArray` tracking save/restore depth. Each depth-2 save/restore cycle = one visual element. Accumulates transforms (`mulMat`), colors, clips (with `clipTransform`), graphics state, and terminal draw ops (fill/stroke/image/text)
- **Phase 2 — Group Classification** (`classifyAllGroups`): Classifies each group as `IMAGE`, `TEXT`, `VECTOR_FILL`, `VECTOR_STROKE`, `VECTOR_COMPOUND`, `BACKGROUND_FILL`, or `BUTTON_BACKGROUND`. Computes bounding boxes via `computeGroupBounds` (applies full transform chain to path coordinates). Generates descriptive layer names with shape type, color hex, and dimensions
- **Phase 3 — Canvas Replay Rasterization** (`rasterizeVectorGroups`): For each vector group, creates a bounding-box-sized offscreen canvas. Applies clip regions with their own transforms, sets the drawing transform via `ctx.setTransform()`, then replays path operations (`replayPathOps`) mapping PDF operators → Canvas 2D API (moveTo, lineTo, bezierCurveTo, rect, fill, stroke). Produces isolated transparent pixel layers
- **PSD Assembly**: Unified layer stack ordered by z-index: text → vector → images → background. Vector layers are pixel layers with correct position/bounds. Text layers retain Direction A pattern (canvas + TypeLayer metadata). Download UI now shows vector layer counts
- **New helper functions**: `mulMat` (matrix multiply), `txPt` (transform point), `PATH_OPS` constants, `describeShape`, `generateVectorName`, `computeGroupBounds`
- **Background cleanup**: Now also clears vector regions when "Clean up background" is enabled
- **Debug logging**: Comprehensive per-phase debug output — segmentation groups, classification results, rasterization targets, and unified PSD assembly manifest
- **RESULT**: Vector shapes (colored rectangles, rotated polygons, bezier decorations, button backgrounds) now extracted as independent PSD layers instead of being baked into the monolithic background

### v2.6 — Direction A: Rasterized Canvas + TypeLayer Metadata (SUCCESS)
- **Core Fix**: Each text block renders to an offscreen canvas (`document.createElement('canvas')`) with `fillText()` providing pixel data that ag-psd uses to derive non-zero bounds (lines ~1319-1352)
- **TypeLayer Objects**: Now have both `canvas` AND `text` properties — Photoshop sees editable TypeLayer metadata while ag-psd gets the pixel data it needs for bounds calculation
- **Removed `invalidateTextLayers: true`** from `writePsd()` call — this flag was actively harmful, causing Photoshop to ignore our pixel data. Now uses just `{generateThumbnail:false, noBackground:true}`
- **Removed broken operator-extracted `textBounds`** — the `setTextMatrix`/`setFont` position extraction was in LOCAL coordinate space (missing CTM chain), making bounds systematically wrong. Only color extraction is retained from the operator list
- **Added `resolveFont()` helper** (line ~520) — attempts to extract real PostScript font names from `page.commonObjs` after `page.render()`, falling back to `normFont()` for CSS generic names
- **Background layer**: No longer sets explicit `bottom`/`right` — ag-psd derives these from the canvas dimensions
- **RESULT: SUCCESS** — Text layers now have non-zero bounds and are visible in Photoshop. Users can double-click to edit text content.

### v2.5 — Text Bounds from Operator Trace (FAILED)
- Added `textBounds` extraction from `setFont` (size) and `setTextMatrix` (position) in `extractPageImages()`
- Added `tWidth` passthrough to unmerged blocks
- Added explicit `top/left/bottom/right` to TypeLayer objects in PSD assembly
- Returns `{images, textColors, textBounds}` from operator list walk
- **RESULT: FAILED** — PSD manifest still shows zero bounds for all text layers
- **Root cause 1:** ag-psd ignores `top/left/bottom/right` on TypeLayers without canvas pixel data
- **Root cause 2:** `invalidateTextLayers: true` + missing font = Photoshop re-renders to nothing
- **Root cause 3:** Operator bounds extraction applies only viewport transform, missing CTM chain → wrong coordinates
- **Root cause 4:** `getTextContent` already provides correct page-space coordinates, making operator extraction redundant
- Debug log confirmed bounds were set on JS objects: e.g., T1 "opening" bounds=(914,387)→(1482,591)
- But PSD manifest confirmed Photoshop reads: bounds=(0,0,0,0) for ALL text layers

### v2.4 — Text Color Extraction from Operator List
- Modified `extractPageImages()` to track color state during operator loop: `setFillRGBColor`, `setFillGray`, `setFillCMYKColor`
- Records fill color for each `beginText…endText` block (text rendering order)
- Returns `{images, textColors}` instead of just images
- Pipeline applies op-list colors to rawItems by order index (PDF rendering order = getTextContent order)
- Handles both 0–255 integer and 0–1 float color value ranges gracefully
- "Sample text colors" toggle now acts as canvas override on top of op-list colors, not the only source
- Result: Text layers now have actual colors (plum, coral, white, etc.) regardless of toggle state
- Confirmed: vector shapes remain baked in (inherent PDF.js limitation — no way to isolate render)

### v2.3 — Debug Logging Enhancement
- Added RAW PDF.js DATA section: shows ALL text items (including empty), styles dictionary, full transform matrices
- Added RAW OPERATOR LIST section: image ops, vector ops, text ops, color ops, transform ops, top 10 most frequent
- Added PDF DOCUMENT METADATA: title, author, creator, dates, format version, linearization, form/XFA/collection/signature flags
- Added PAGE RAW PROPERTIES: mediaBox, dimensions in points/inches/mm, rotation, userUnit, scale, output canvas size
- Added FULL OPERATOR TRACE: every save/restore (with depth), transform matrices, color changes (RGB/gray/CMYK), font sets, showText with preview, text positioning, image operations with native dimensions, path construction, fill/stroke, graphics state, marked content

### v2.2 — clearRect Bounds Fix + Image Types + Debug Panel
- Fixed `clearRect` bounds: `eraseH = fontSize × 1.6`, `eraseY = baseline - fontSize × 1.25`, 4px padding
- Added `paintInlineImageXObject` and `paintImageXObjectRepeat` to image extraction
- Added debug log panel with copy-to-clipboard button
- Added per-step debug logging throughout pipeline

### v2.1 — Pristine-First Extraction Pipeline
- Restructured pipeline: render → extract pristine → clearRect → build
- Added image extraction via operator list walking + CTM tracking
- Added image deduplication (< 5px bounds difference)
- Added rotation detection via viewport × item.transform
- Added CMYK detection warning
- Added canvas size guard (15,000 px cap, 30,000 px PSD limit warning)

### v2.0 — Initial Restructure
- Moved from `fillRect` (opaque paint-over) to `clearRect` (transparent holes)
- Added color sampling from pristine canvas (before modification)
- Added text block merging with column guards

### v1.0 — Initial Implementation
- Basic PDF.js text extraction + full-page raster background
- Simple `fillRect` background cleanup with page background color
- ag-psd TypeLayer creation with font name, size, position
- Single HTML file, CDN-loaded libraries

---

## What We Need To Do

### Completed (v2.3 + v2.4)
- [x] Add PDF document metadata to debug log (title, author, creator, page count, mediaBox, rotation)
- [x] Add page-level properties (dimensions in points, userUnit, cropBox vs mediaBox)
- [x] Add per-operator trace: log color values for setFillRGBColor, transform matrices, image object details
- [x] Add image object introspection: native width/height via page.objs.get() in operator trace
- [x] Log every processing decision with reasoning
- [x] Full operator trace: every save/restore, transform, color change, font set, showText, path, image paint, gstate, marked content
- [x] Extract text colors from operator list (setFillRGBColor, setFillGray, setFillCMYKColor)
- [x] Apply op-list colors to text items by rendering order (PDF order = getTextContent order)
- [x] Make "Sample text colors" toggle a canvas override, not the only source

### Completed (v2.6)
- [x] **Fix P0 (Direction A):** Render each text item to an off-screen canvas, use as layer `canvas` + `text` property combo
  - Browser Canvas API renders text → gives ag-psd real pixel data → non-zero bounds guaranteed
  - TypeLayer metadata preserved for Photoshop editability
  - Erase toggle now works correctly (clearRect creates holes, pixel text layers fill them)
- [x] **Fix P1:** Extract real font names from PDF font dictionary via `page.commonObjs` — implemented `resolveFont()` helper
- [x] **Remove broken operator bounds:** Deleted the `textBounds` extraction from `extractPageImages()` — it was redundant and used wrong coordinate space
- [x] Remove `invalidateTextLayers: true` from `writePsd()` call — was actively harmful

### Completed (v3.0)
- [x] **Three-phase vector extraction pipeline**: Segment → Classify → Rasterize
- [x] **Phase 1**: Operator group segmentation by save/restore depth with full CTM tracking
- [x] **Phase 2**: Group classification (image/text/vector/background/button) with bounds computation
- [x] **Phase 3**: Canvas replay rasterization for vector groups (fill, stroke, bezier paths, clips)
- [x] **Unified PSD assembly**: Text + vector + image + background layers at correct z-order
- [x] **Vector shape extraction**: Filled rectangles, rotated polygons, bezier decorations, button backgrounds now separate layers

### Next Session (v3.1 plan)
- [ ] Test with Canva, Figma, InDesign, Word PDFs to verify vector extraction works across creators
- [ ] Tune depth-2 segmentation heuristic for non-Canva PDFs (may need adaptive depth detection)
- [ ] Improve `curveTo2`/`curveTo3` handling (currently approximated with quadratic/degenerate cubic)
- [ ] Consider enabling "Clean up background" toggle by default now that vector layers are extracted
- [ ] Improve font name resolution — enhance `resolveFont()` for more edge cases

### Backlog
- [ ] Smart layer grouping (group related text + image layers)
- [ ] Gradient detection for background regions
- [ ] Multiple font styles within a single TypeLayer (StyleRun arrays)
- [ ] Adaptive segmentation depth for non-Canva PDF generators
