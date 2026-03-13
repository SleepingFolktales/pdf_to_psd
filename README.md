# PDF → PSD Converter

A fully client-side, single-HTML-file converter that transforms PDFs into layered Adobe Photoshop files with **editable text layers**. No server, no installation, no build step required.

## Features

- **Editable Text Layers** — Each text block becomes a real Photoshop TypeLayer (marked with "T" icon), fully editable in Photoshop
- **Background Preservation** — Full-page raster background captures all visual content (shapes, images, gradients)
- **Smart Text Handling** — Intelligently merges adjacent text blocks and removes text from the background to prevent double-rendering
- **Color Detection** — Automatically samples text colors from the rendered PDF
- **Multi-Page Support** — Convert entire PDFs page-by-page with customizable page ranges
- **Adjustable Resolution** — Choose output DPI (72, 144, or 216) to balance file size and quality
- **Zero Privacy Concerns** — Everything runs in your browser; PDFs never leave your machine

## How It Works

### Pipeline Overview

```
PDF File
  ↓
1. PDF Parsing
   • Extract text blocks with position, font, size, color
   • Render full page as high-DPI raster image
  ↓
2. Post-Processing
   • Merge adjacent text blocks into logical groups
   • Paint over text regions with page background color
  ↓
3. PSD Assembly
   • Create background PixelLayer from raster
   • Create TypeLayer for each text block
   • Inject editable text data via ag-psd
  ↓
PSD File (ready to download)
```

### Three-Stage Conversion

#### Stage 1: PDF Parsing
- **Text Extraction** — Uses PDF.js to extract text items with bounding boxes, font names, sizes
- **Background Render** — Renders the entire page at your chosen DPI to a canvas
- **Coordinate Mapping** — Converts PDF points to PSD pixels using the scale factor (DPI / 72)

#### Stage 2: Post-Processing
- **Text Block Merging** — Groups nearby same-column text items into fewer TypeLayers (optional)
  - Criteria: same font family, horizontal alignment within ~20px, vertical gap ≤ 1.8× font height
  - Prevents excessive layer count while preserving layout
- **Background Cleanup** — Paints over each text region with the detected page background color
  - Keeps the background layer opaque (no transparent holes)
  - Prevents text from appearing twice (once in background, once in TypeLayer)

#### Stage 3: PSD Assembly
- **Background Layer** — Single PixelLayer with the post-processed raster
- **TypeLayers** — One per text block with:
  - Editable text content
  - Font name and size
  - Sampled text color
  - Exact position and dimensions
- **ag-psd Integration** — Handles all binary PSD structure internally; no manual EngineData construction

## Usage

### Quick Start

1. Open `index.html` in any modern web browser
2. Drop a PDF onto the drop zone (or click to browse)
3. Adjust options:
   - **Output Resolution** — 72 DPI (smaller files), 144 DPI (default, balanced), or 216 DPI (highest quality)
   - **Page Range** — Convert all pages or specify a range (e.g., `1-3` or `all`)
   - **Remove text from background layer** — Paint over text with page background color (recommended: ON)
   - **Sample text colors** — Detect text color from rendered canvas (recommended: ON)
   - **Merge adjacent text blocks** — Group nearby items into fewer layers (recommended: ON)
4. Click **Convert to PSD**
5. Download the generated PSD file(s)

### Opening in Photoshop

When you open the generated PSD in Photoshop, you may see this prompt:

> **"Some text layers need to be updated before they can be used for vector-based output. Do you want to update these layers now?"**

Click **Update**. This is expected behavior for externally-generated PSDs — Photoshop rebuilds the text layer bitmaps from the editable text data. After clicking Update, all text layers are fully editable.

## Technical Details

### Libraries Used

- **PDF.js** (Mozilla) — In-browser PDF parsing and rendering
  - Extracts text content with position and font metadata
  - Renders pages to canvas at any DPI
  - No server required; works entirely in the browser

- **ag-psd** (v30.1.0) — JavaScript PSD writer
  - Creates real TypeLayers with editable text
  - Handles EngineData binary blob construction automatically
  - Supports RGB 8-bit color mode (standard for design exports)
  - Actively maintained (637 GitHub stars, latest release March 2025)

- **FileSaver.js** — Browser download API
  - Enables direct file download without server

### Architecture

Single HTML file with:
- Inline CSS for styling
- Vanilla JavaScript (no framework dependencies)
- CDN-loaded libraries (no build step)
- Client-side processing (no data leaves your browser)

### Coordinate Systems

| System | Origin | Y Direction | Unit |
|--------|--------|-------------|------|
| PDF | Bottom-left | Up | Points |
| PDF.js | Top-left | Down | Points |
| PSD | Top-left | Down | Pixels |

Conversion: `pixel = point × (DPI / 72)`

### Text Layer Implementation

Each TypeLayer contains:
- **Txt** — Display string (e.g., "Hello World\r")
- **EngineData** — Binary blob with structured text metadata
  - Editor text (must match Txt exactly)
  - StyleRun with font, size, color per run
  - ParagraphRun for paragraph-level styling
- **Transform** — Translation matrix positioning the layer

ag-psd automatically:
- Constructs EngineData from the `layer.text` object
- Ensures RunLengthArray sums match text length (critical for Photoshop recognition)
- Syncs Txt and Editor.Text
- Deletes stale document-level TEXT_ENGINE_DATA

### Font Name Handling

PDF fonts often have subset prefixes (e.g., `ABCDEF+Montserrat-Bold`). The converter:
1. Strips the prefix before the `+` sign
2. Maps common PostScript names to system font names:
   - `ArialMT` → `Arial`
   - `Arial-BoldMT` → `Arial Bold`
   - `TimesNewRomanPSMT` → `Times New Roman`
3. Falls back to the stripped name if no mapping exists

If a font isn't installed on your machine, Photoshop will prompt for a substitution on open.

### Background Color Detection

The converter samples the page background by:
1. Reading 8×8 pixel patches from the four corners and edges
2. Averaging pixels with high luminance (luma > 180)
3. Falling back to white (255, 255, 255) if no light pixels found

This ensures text regions are painted with the actual page background color, maintaining visual continuity.

## Known Limitations

### Fully Supported
- ✓ RGB color mode (8-bit)
- ✓ Writing pixel/image layers
- ✓ Writing text (TypeLayers)
- ✓ Layer groups, order, and names
- ✓ Layer opacity and blending
- ✓ Horizontal text orientation

### Requires Action on Open in Photoshop
- ⚠ Text bitmap not pre-rendered — click **Update** on the prompt Photoshop shows
- ⚠ Composite image not generated — Photoshop rebuilds on first save
- ⚠ Predefined Paragraph/Character Styles not written — apply manually in Photoshop
- ⚠ Blending option changes need manual re-save in Photoshop to update bitmap data

### Not Supported by ag-psd
- ✗ CMYK, LAB, Indexed, Duotone, Multichannel color modes
- ✗ 16-bit or 32-bit channels
- ✗ Large Document Format (.psb)
- ✗ Color palettes
- ✗ Patterns & Pattern Overlay effects
- ✗ Frame/timeline animations
- ✗ 3D layer effects
- ✗ Vertical text orientation (causes corrupt PSD)
- ✗ Some smart object filter types

## Troubleshooting

### "Failed to load PDF.js" or other library errors
- Check your internet connection — libraries load from CDN
- Refresh the page and try again
- If using `file://` protocol, serve from `localhost` instead (some browsers block workers on local files)

### Text layers appear as pixel layers in Photoshop
- Click **Update** when Photoshop prompts on open
- If the prompt doesn't appear, the text data may be corrupted — try re-converting with different options

### Text colors are wrong
- Disable **Sample text colors** and manually set colors in Photoshop
- Or re-convert with **Sample text colors** enabled (may need to adjust threshold in code)

### Too many or too few text layers
- Adjust the **Merge adjacent text blocks** toggle
- ON: Groups nearby items (fewer layers, simpler structure)
- OFF: One layer per text item (more layers, exact layout)

### File size is too large
- Lower the **Output Resolution** from 216 DPI to 144 or 72 DPI
- Disable **Sample text colors** (slightly reduces file size)

## Development

### File Structure
- `index.html` — Main application (single file, ~700 lines)
- `pdf_to_psd_concept.md` — Detailed conceptual guide to the conversion pipeline
- `structure.md` — Feasibility report and technical architecture notes
- `README.md` — This file

### Extending the Converter

The code is organized into logical sections:
- **Library Loading** — CDN imports and initialization
- **State Management** — PDF document, options, results
- **Helpers** — DOM utilities, logging, progress tracking
- **Drag & Drop** — File input handling
- **PDF Parsing** — Text extraction and rendering
- **Post-Processing** — Text merging, background cleanup
- **PSD Assembly** — TypeLayer creation and download

To add features:
1. Add UI controls in the HTML
2. Add corresponding state variables and event handlers in `initApp()`
3. Integrate into the conversion pipeline (stages 1–3)
4. Test with sample PDFs from Canva, Figma, InDesign

### Testing

Recommended test sources:
- **Canva** — Simple layouts, standard fonts
- **Figma** — Complex layouts, multiple text styles
- **InDesign** — Professional documents, embedded fonts
- **Word/Google Docs** — Exported PDFs with mixed content

## License

This project is provided as-is. Feel free to use, modify, and distribute.

## Credits

Built with:
- [PDF.js](https://mozilla.github.io/pdf.js/) — Mozilla's PDF parser
- [ag-psd](https://github.com/Agamnentzar/ag-psd) — JavaScript PSD writer
- [FileSaver.js](https://github.com/eligrey/FileSaver.js/) — Browser download API

## Questions?

Refer to the detailed technical docs:
- `pdf_to_psd_concept.md` — Deep dive into the conversion algorithm
- `structure.md` — Feasibility analysis and library assessment
