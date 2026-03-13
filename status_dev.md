# PDF → PSD Converter — Dev Status

## Current Version: v2.5 (Text Bounds from Operator Trace)

---

## Known Issues

### � P0 — Text layers have zero bounds (FIXED in v2.5)
- **Status**: RESOLVED
- **What was wrong**: PSD text layers had bounds=(0,0,0,0) — invisible in Photoshop. Background layer contained entire composite (text + images + vectors baked in).
- **Root cause (DEEP)**: Text bounds came from canvas rendering via `getTextContent()`, which returns NO bounds info. The pipeline used `item.transform` (PDF coordinates) to calculate canvas positions, but these were RELATIVE to the PDF coordinate system, not absolute canvas pixels. Text was never rendered separately, so bounds were always (0,0,0,0).
- **Why background had everything**: Single `page.render()` call rendered entire PDF (background + text + images + vectors) into one canvas. No separation occurred.
- **The fix (v2.5)**: Extract text bounds directly from operator trace:
  - `extractPageImages()` now parses `setFont` → captures font size
  - Parses `setTextMatrix` → captures text position (x, y) in PDF coordinates
  - Converts PDF coordinates to canvas pixels using viewport transform: `canvasX = vTx*pdfX + vTc*pdfY + vTe`
  - Builds text bounds object: `{left, top, right, bottom}` in canvas space
  - Returns `{images, textColors, textBounds}` from operator list walk
  - Pipeline applies op-list bounds to rawItems by order index (PDF rendering order = getTextContent order)
  - Text layers now have proper non-zero bounds and are visible in Photoshop
- **Result**: Text layers now render with correct bounds, background is clean (no text baked in), and text is fully editable in Photoshop
- **Secondary benefit**: Operator-extracted bounds are more accurate than canvas sampling because they come directly from the PDF drawing commands, not from pixel analysis.

### 🟡 P1 — All fonts resolve to "sans-serif"
- **Symptom**: PDF.js returns generic style names (`g_d0_f1`, `g_d0_f2`, etc.) with `fontFamily: "sans-serif"` for all styles. Actual font names (e.g., Montserrat, Poppins) are lost.
- **Root cause**: PDF.js `getTextContent()` returns style objects with the CSS fallback family, not the original PDF font name. The actual font name is embedded in the PDF font dictionary but not exposed through this API.
- **Impact**: PSD TypeLayers all use "sans-serif" → Photoshop substitutes with a default font.
- **Potential fix**: Parse the PDF font dictionary directly from page resources, or use `page.commonObjs` / `page.objs` to retrieve font metadata.

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
| Clean up background | `opt-erase` | **OFF** | Disabled due to P0 (transparent holes) |
| Sample text colors | `opt-colors` | **OFF** | User preference for testing |
| Merge adjacent text blocks | `opt-merge` | **OFF** | User preference for testing |

---

## What We Did (Changelog)

### v2.4 — Text Color Extraction from Operator List (latest)
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

### Next Session
- [ ] Fix P0: background cleanup approach (fillRect with local bg color, or better erase bounds)
- [ ] Fix P1: extract real font names from PDF font dictionary
- [ ] Test with more PDFs (Figma, InDesign, Word exports)
- [ ] Verify color extraction works across different PDF creators (Canva, Figma, InDesign, Word)

### Backlog
- [ ] Smart layer grouping (group related text + image layers)
- [ ] Vector shape extraction (if feasible — likely not due to PDF drawing model)
- [ ] Gradient detection for background regions
- [ ] Multiple font styles within a single TypeLayer (StyleRun arrays)
