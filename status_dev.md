# PDF → PSD Converter — Dev Status

## Current Version: v2 (Pristine-First Extraction Pipeline)

---

## Known Issues

### 🔴 P0 — Background cleanup creates transparent holes
- **Status**: Active / workaround = toggle OFF
- **Toggle**: "Clean up background (separate text & images)" (`opt-erase`)
- **Symptom**: When enabled, `clearRect` cuts transparent holes where text and images were. The PSD composite looks broken because the transparent regions are visible — TypeLayers and image layers are supposed to fill those gaps, but positioning/sizing mismatches leave visible holes.
- **Root cause**: The erase bounds (`fontSize × 1.6` height, `baseline - fontSize × 1.25` top) don't perfectly align with the rendered text glyphs. Also, `clearRect` creates full transparency rather than painting over with the local background color.
- **Workaround**: User has unchecked the toggle (default now OFF). Background stays intact with text baked in. Text layers sit on top as editable duplicates.
- **Potential fix**: Switch from `clearRect` (transparent holes) to `fillRect` with the sampled local background color, OR improve erase bounds to match actual glyph bounding boxes more precisely.

### 🟡 P1 — All fonts resolve to "sans-serif"
- **Symptom**: PDF.js returns generic style names (`g_d0_f1`, `g_d0_f2`, etc.) with `fontFamily: "sans-serif"` for all styles. Actual font names (e.g., Montserrat, Poppins) are lost.
- **Root cause**: PDF.js `getTextContent()` returns style objects with the CSS fallback family, not the original PDF font name. The actual font name is embedded in the PDF font dictionary but not exposed through this API.
- **Impact**: PSD TypeLayers all use "sans-serif" → Photoshop substitutes with a default font.
- **Potential fix**: Parse the PDF font dictionary directly from page resources, or use `page.commonObjs` / `page.objs` to retrieve font metadata.

### 🟡 P2 — Color sampling disabled = all text is black
- **Symptom**: When "Sample text colors" is OFF, all text defaults to `rgb(0,0,0)`.
- **Root cause**: Default color is hardcoded `{r:0,g:0,b:0}` and only overwritten when color sampling is enabled.
- **Impact**: Minor when sampling is ON. When OFF, all text appears black in PSD.

### 🟢 P3 — Debug logging needs more visibility
- **Symptom**: Logs show text items and operator counts but not the full raw PDF data (image objects, color state changes, transform values, PDF metadata, page properties).
- **Status**: Being addressed now — adding comprehensive raw data logging.

---

## Toggles Status (Default State)

| Toggle | ID | Default | Notes |
|--------|-----|---------|-------|
| Clean up background | `opt-erase` | **OFF** | Disabled due to P0 (transparent holes) |
| Sample text colors | `opt-colors` | **OFF** | User preference for testing |
| Merge adjacent text blocks | `opt-merge` | **OFF** | User preference for testing |

---

## What We Did (Changelog)

### v2.3 — Debug Logging Enhancement (current session)
- Added RAW PDF.js DATA section: shows ALL text items (including empty), styles dictionary, full transform matrices
- Added RAW OPERATOR LIST section: image ops, vector ops, text ops, color ops, transform ops, top 10 most frequent
- Next: Add PDF metadata, per-operator trace, image object introspection, effective DPI calculation

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

### Immediate (This Session)
- [x] Add PDF document metadata to debug log (title, author, creator, page count, mediaBox, rotation)
- [x] Add page-level properties (dimensions in points, userUnit, cropBox vs mediaBox)
- [x] Add per-operator trace: log color values for setFillRGBColor, transform matrices, image object details
- [x] Add image object introspection: native width/height via page.objs.get() in operator trace
- [x] Log every processing decision with reasoning
- [x] Full operator trace: every save/restore, transform, color change, font set, showText, path, image paint, gstate, marked content

### Next Session
- [ ] Fix P0: background cleanup approach (fillRect with local bg color, or better erase bounds)
- [ ] Fix P1: extract real font names from PDF font dictionary
- [ ] Investigate whether PDF.js exposes image native resolution
- [ ] Test with more PDFs (Figma, InDesign, Word exports)

### Backlog
- [ ] Smart layer grouping (group related text + image layers)
- [ ] Vector shape extraction (if feasible — likely not due to PDF drawing model)
- [ ] Gradient detection for background regions
- [ ] Multiple font styles within a single TypeLayer (StyleRun arrays)
