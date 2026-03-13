# PDF → PSD Converter

A fully client-side, single-HTML-file converter that transforms PDFs into layered Adobe Photoshop files with **editable text layers** and **separated image layers**. No server, no installation, no build step required.

## Features

- **Editable Text Layers** — Each text block becomes a real Photoshop TypeLayer ("T" icon), fully editable with the Type tool
- **Separated Image Layers** — Embedded photos, logos, and decorations are extracted as individual pixel layers
- **Clean Background** — Text and image regions are cut from the background, leaving only base design (shapes, borders, fills)
- **Rotated Text Support** — Vertical and angled text detected and rendered via affine rotation transforms
- **Smart Color Sampling** — Text fill colors sampled from the pristine render before any canvas modification
- **Text Block Merging** — Adjacent text items grouped into logical blocks (tunable, optional)
- **CMYK Detection** — Warns when a PDF uses CMYK colorspace (output is always RGB 8-bit)
- **Canvas Size Guard** — Auto-caps pages > 15,000 px to prevent browser crashes; warns on PSD 30k limit
- **Multi-Page Support** — Convert entire PDFs page-by-page with customizable page ranges
- **Adjustable Resolution** — 72, 144, or 216 DPI
- **Debug Logging** — Detailed pipeline trace with copy-to-clipboard for diagnostics
- **Zero Privacy Concerns** — Everything runs in your browser; PDFs never leave your machine

## How It Works

### v2 Pipeline: Pristine-First Extraction

The key insight: extract everything from the **unmodified** canvas first, then cut.

```
PDF File
  ↓
1. RENDER        — Full-page raster at target DPI (pristine canvas)
  ↓
2. EXTRACT TEXT  — page.getTextContent() → position, font, size per item
                   Rotation detection via viewport × item.transform
  ↓
3. SAMPLE COLORS — From the pristine canvas (before any modification)
                   Most-common non-white pixel in each text item's bbox
  ↓
4. EXTRACT IMAGES — Walk page.getOperatorList(), track CTM through
                    save/restore/transform. Copy each image bbox from
                    the CLEAN canvas into its own layer.
  ↓
5. CLEAN BACKGROUND — clearRect text + image regions → transparent holes
                      TypeLayers and image layers fill the gaps in the PSD
  ↓
6. MERGE TEXT    — Group adjacent items into fewer TypeLayers (optional)
  ↓
7. ASSEMBLE PSD  — Layer stack (top → bottom in Photoshop):
                     TypeLayers → Image layers → Background
  ↓
PSD File (ready to download)
```

### PSD Layer Structure

```
PSD File
├── T1: "Greta Mae Evans"         ← TypeLayer (editable text)
├── T2: "Digital Marketing"       ← TypeLayer
├── T3: "About Me\nI have been…" ← TypeLayer (merged paragraph)
├── T4: "+123-456-7890"           ← TypeLayer (rotated via transform)
│   ...
├── IMG 1 (310×310)               ← Pixel layer (photo)
├── IMG 2 (60×60)                 ← Pixel layer (ornament)
├── IMG 3 (120×40)                ← Pixel layer (logo)
│   ...
└── Background                    ← Pixel layer (shapes, borders, fills)
                                     Text + image regions = transparent
```

## Usage

### Quick Start

1. Open `index.html` in any modern browser (Chrome recommended)
2. Drop a PDF onto the drop zone (or click to browse)
3. Adjust options:
   - **Output Resolution** — 72 DPI (small), 144 DPI (default), 216 DPI (high quality)
   - **Page Range** — `all`, or specific pages like `1-3`
   - **Clean up background** — Cuts text/image regions out of background (recommended: ON)
   - **Sample text colors** — Detects fill color per text item (recommended: ON)
   - **Merge adjacent text blocks** — Groups nearby items into fewer layers (recommended: ON)
4. Click **Convert to PSD**
5. Download the generated PSD file(s)

### Opening in Photoshop

When you open the PSD, Photoshop shows:

> **"Some text layers need to be updated before they can be used for vector-based output. Do you want to update these layers now?"**

Click **Update**. This is expected — Photoshop rebuilds text rasters from the editable EngineData. After updating, all text layers are fully editable.

### Debug Logging

The converter includes a detailed debug log panel below the progress bar. It traces every step of the pipeline:
- PDF operator list statistics (image ops, vector ops, transforms)
- Per-text-item extraction details (position, font, erase bounds)
- Per-image detection (found/skipped with reason and bounds)
- Background cleanup (clearRect bounds for each region)
- Merged text blocks and final PSD layer stack

Click **Copy Debug Log** to copy the full trace to your clipboard for diagnostics.

## Technical Details

### Libraries

| Library | Version | Purpose |
|---------|---------|---------|
| [PDF.js](https://mozilla.github.io/pdf.js/) | 3.11.174 | PDF parsing, rendering, text/image extraction |
| [ag-psd](https://github.com/Agamnentzar/ag-psd) | 30.1.0 | PSD file writing with TypeLayers |
| [FileSaver.js](https://github.com/eligrey/FileSaver.js/) | 2.0.5 | Browser file download |

All loaded from CDN (jsdelivr + unpkg fallback). No build step.

### Architecture

Single HTML file (~1100 lines) with inline CSS + vanilla JavaScript. No frameworks, no bundler, no server. Share via Slack, email, USB — it just works.

### Image Extraction

The converter walks the PDF operator list (`page.getOperatorList()`) and tracks the Current Transformation Matrix (CTM) through `save`/`restore`/`transform` operations. When it encounters an image paint operation, it maps the unit square through `viewport × CTM` to get canvas-pixel bounds, then copies that rectangle from the pristine render.

**Supported image operations:**
- `paintImageXObject` — standard embedded images
- `paintJpegXObject` — JPEG-compressed images
- `paintInlineImageXObject` — inline images (common in Canva exports)
- `paintImageXObjectRepeat` — tiled/repeated images

**Filters:**
- Images < 30 px on either side → skipped (bullets, icons, pattern tiles)
- Images covering > 90% of canvas → skipped (background fills)
- Near-identical bounds (< 5 px difference) → deduplicated

### Text Extraction & Rotation

Text items come from `page.getTextContent()`. Each item's PDF text-direction vector is mapped through the viewport transform to detect canvas-space rotation:

- **Horizontal text** (angle < 15°): `transform = [1, 0, 0, 1, x, y]`
- **Rotated text** (angle ≥ 15°): `transform = [cosA, sinA, -sinA, cosA, x, y]`

Rotated and non-rotated items are never merged together.

### Erase Bounds

Text regions are cleared with generous bounds to cover ascenders and descenders:
- **Erase height** = `fontSize × 1.6` (full glyph bounding box)
- **Erase top** = `baseline - fontSize × 1.25`
- **Padding** = 4 px on each side

### Text Block Merging

Adjacent text items are grouped when:
- **Same line**: y-band within 65% font height, x-gap ≤ 4× font height
- **Next line**: x-aligned within 120% font height, y-gap ≤ 2.5× font height
- **Guards**: merged block width < 45% canvas, height < 30% canvas
- **Rotation**: rotated items never merge with non-rotated items

### Font Name Handling

PDF fonts often have subset prefixes (`ABCDEF+Montserrat-Bold`). The converter:
1. Strips the prefix before `+`
2. Maps common PostScript names: `ArialMT` → `Arial`, `TimesNewRomanPSMT` → `Times New Roman`
3. Falls back to the stripped name

If a font isn't installed, Photoshop prompts for substitution on open.

### Coordinate Systems

| System | Origin | Y Direction | Unit |
|--------|--------|-------------|------|
| PDF | Bottom-left | Up | Points |
| PDF.js | Top-left | Down | Points |
| PSD | Top-left | Down | Pixels |

Conversion: `pixel = point × (DPI / 72)`

## Guards & Warnings

| Guard | Trigger | Action |
|-------|---------|--------|
| Canvas size cap | Any dimension > 15,000 px | Auto-scale down, show warning |
| PSD format limit | Any dimension > 30,000 px | Log error (Photoshop will reject) |
| CMYK detection | `/DeviceCMYK` in raw PDF bytes | Show amber notice — output is RGB |
| Image dedup | Near-identical bounds (< 5 px) | Skip duplicate extraction |
| Full-page image | Image covers > 90% of canvas | Skip (background fill) |
| Tiny image | Image < 30 px on either side | Skip (bullet, icon, pattern tile) |

## Known Limitations

### Fully Supported
- ✓ RGB color mode (8-bit)
- ✓ Pixel/image layers
- ✓ Editable text (TypeLayers)
- ✓ Layer groups, order, and names
- ✓ Layer opacity and blending
- ✓ Horizontal text
- ✓ Rotated/vertical text (via rotation transform)
- ✓ Embedded images as separate layers

### Requires Action on Open
- ⚠ Click **Update** when Photoshop prompts (rebuilds text rasters)
- ⚠ Composite image not pre-generated (Photoshop rebuilds on first save)
- ⚠ Paragraph/Character Styles not written (apply manually)
- ⚠ CMYK PDFs detected and warned — output is always RGB 8-bit
- ⚠ Canvas > 15,000 px auto-capped (lower DPI to avoid)

### Not Supported (ag-psd limits)
- ✗ True CMYK/LAB output — always RGB 8-bit
- ✗ 16-bit or 32-bit channels
- ✗ Large Document Format (.psb)
- ✗ Color palettes / Indexed mode
- ✗ Patterns & Pattern Overlay effects
- ✗ Frame/timeline animations
- ✗ 3D layer effects
- ✗ Some smart object filter types

### Inherent PDF Limitations
- ✗ Vector shapes (borders, progress bars, decorative elements) remain baked into the background — PDF is a flat drawing model with no discrete shape objects

## Troubleshooting

### Library loading fails
Check your internet connection. Libraries load from CDN. If using `file://` protocol, serve from `localhost` instead (some browsers block cross-origin workers).

### Text still visible in background
The erase bounds may not cover unusual fonts. Check the debug log for erase coordinates vs text positions. The current bounds use `fontSize × 1.6` height — increase this in code if needed.

### Only 1 image found (expected more)
Many "images" in Canva/Figma PDFs are actually vector paths (fill/stroke operations), not embedded bitmaps. These show up as vector draw ops in the debug log. Vector shapes cannot be extracted as separate layers — they remain in the background.

### Text colors are wrong
Colors are sampled from the rendered canvas. If text overlaps complex backgrounds, the sampler may pick up the wrong color. Disable **Sample text colors** and set colors manually in Photoshop.

### Too many or too few text layers
Toggle **Merge adjacent text blocks**:
- **ON**: fewer layers, grouped paragraphs
- **OFF**: one layer per text item, exact layout

### File size too large
Lower DPI from 216 to 144 or 72.

## Development

### File Structure
- `index.html` — Complete application (~1100 lines)
- `pdf_to_psd_concept.md` — Conceptual guide to the conversion algorithm
- `structure.md` — Architecture documentation and v2 pipeline design
- `README.md` — This file

### Code Organization

Inside `initApp()`:
- **Debug logger** — `dbg()`, `dbgClear()`, `copyDebugLog()`
- **CMYK detection** — `detectCmyk(buf)`
- **Coordinate conversion** — `toCanvas(viewport, x, y)`
- **Color sampling** — `sampleColor()`, `samplePageBackground()`, `sampleLocalBg()`
- **Image extraction** — `extractPageImages(page, vp, w, h, pageNum)`
- **Text merging** — `mergeBlocks(items, canvasW, canvasH)`
- **Main pipeline** — `startConversion()` (7-step pristine-first flow)

### Testing

Recommended test sources:
- **Canva** — Simple layouts, standard fonts, inline images
- **Figma** — Complex layouts, multiple text styles
- **InDesign** — Professional documents, embedded fonts
- **Word/Google Docs** — Exported PDFs with mixed content

## License

This project is provided as-is. Feel free to use, modify, and distribute.

## Credits

Built with:
- [PDF.js](https://mozilla.github.io/pdf.js/) — Mozilla's PDF parser
- [ag-psd](https://github.com/Agamnentzar/ag-psd) — JavaScript PSD writer
- [FileSaver.js](https://github.com/eligrey/FileSaver.js/) — Browser file download

## Docs

- `pdf_to_psd_concept.md` — Deep dive into the conversion algorithm
- `structure.md` — Architecture, v2 pipeline design, and library assessment
