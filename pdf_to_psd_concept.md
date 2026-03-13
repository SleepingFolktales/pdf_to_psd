# PDF-to-PSD Conversion — Language-Agnostic Conceptual Guide

A conceptual description of how to convert a PDF document into a layered
Adobe Photoshop (PSD) file with **editable text layers**.  This document is
independent of any programming language or library — use it as a blueprint
for reimplementation in JavaScript, Rust, Go, or any other stack.

---

## 1. High-Level Goal

Given a PDF page, produce a PSD file where:

- **Background** — a single raster layer that faithfully reproduces all visual
  content (shapes, images, gradients, backgrounds).
- **Text** — one editable TypeLayer per text block, positioned exactly where the
  text appeared in the PDF.  Photoshop shows these with the "T" icon and they
  can be edited with the Type tool.

This two-layer architecture (background + text) eliminates overlap problems
that arise when trying to separately extract images, vector shapes, and text.

---

## 2. Pipeline Overview

```
PDF File
  │
  ▼
┌─────────────────────────┐
│  1. PDF Parser           │  Extract text blocks + render full page
│     • Text extraction    │
│     • Full-page raster   │
└──────────┬──────────────┘
           │
           ▼
┌─────────────────────────┐
│  2. Post-Processing      │  Merge text blocks, erase text from background
│     • Merge text blocks  │
│     • Erase text regions │
└──────────┬──────────────┘
           │
           ▼
┌─────────────────────────┐
│  3. PSD Builder          │  Assemble PSD with background + TypeLayers
│     • Background pixel   │
│     • TypeLayer inject   │
│     • Strip stale data   │
└──────────┬──────────────┘
           │
           ▼
       PSD File
```

---

## 3. Stage 1 — PDF Parsing

### 3.1 Text Extraction

Use a PDF library to extract text blocks with metadata:

- **Bounding box** (x, y, width, height) in PDF points
- **Text content** (the actual string)
- **Font name** (e.g., `ArialMT`, `ABCDEF+Montserrat-Bold`)
- **Font size** (points)
- **Color** (RGB)
- **Bold / italic** flags (often inferred from font name)

Each text block may contain multiple "spans" (font runs).  A span is a
contiguous sequence of characters sharing the same font, size, and color.

**Coordinate conversion**: PDF points → PSD pixels via `scale = dpi / 72`.
All coordinates must be converted: `px = pt × scale`.

### 3.2 Full-Page Background Render

Render the entire PDF page as a single RGBA raster image at the target DPI.
This captures everything visible on the page:

- Vector shapes and paths
- Embedded images (with clipping, masks, and transforms)
- Gradients and fills
- Text (which we'll erase in post-processing)

The result is one PIL/Canvas/Buffer image per page.

---

## 4. Stage 2 — Post-Processing

### 4.1 Merge Adjacent Text Blocks

Design tools (Canva, Figma, InDesign) often export each line of a text frame
as a separate PDF text block.  Merge them back into logical groups:

**Merge criteria:**
- Same font family (strip weight/style suffixes for comparison)
- Horizontal start aligned within ~20px
- Vertical gap ≤ 1.5× font height

After merging, re-compute the bounding box as the union of all merged blocks,
and concatenate the text content.

### 4.2 Erase Text Regions from Background

The background raster contains text (since it's a full-page render).  Since
text will be rendered by Photoshop's TypeLayer engine, we erase text regions
from the background to prevent double-rendering:

For each text block, paint its bounding box (with ~2px padding) transparent
in the background image.

---

## 5. Stage 3 — PSD Assembly

### 5.1 PSD Structure

```
PSD File
├── Canvas (max page width × max page height, RGBA)
└── Group: "Page 1"
    ├── PixelLayer: "Background (Page 1)"    ← full-page raster (text erased)
    ├── TypeLayer:  "Text: Hello World"       ← editable text
    ├── TypeLayer:  "Text: Lorem ipsum..."    ← editable text
    └── ...
```

### 5.2 Background Layer

Create a standard PixelLayer with the post-processed background image.
Position at (0, 0), full page dimensions.

### 5.3 TypeLayers — The Critical Part

Photoshop TypeLayers are NOT just pixel layers with text drawn on them.
They are PixelLayers with an additional **TypeToolObjectSetting** tagged block
(tag key `TySh`) that contains structured text data.

**A TypeLayer has three synchronized text storage locations:**

```
TypeLayer
├── [1] text_data → "Txt " key         ← Display string Photoshop reads first
├── [2] text_data → EngineData          ← Binary blob with structured text
│       └── EngineDict
│           ├── Editor → Text           ← Must match #1 exactly
│           ├── StyleRun
│           │   ├── RunArray            ← Style per run (font, size, color)
│           │   └── RunLengthArray      ← Chars each run covers
│           └── ParagraphRun
│               ├── RunArray            ← Paragraph style per run
│               └── RunLengthArray      ← Must sum to len(text)
└── [3] Document-level TEXT_ENGINE_DATA ← Global mirror (delete it)
```

**CRITICAL RULES:**

1. **`Txt` and `Editor > Text` must be identical strings.**
2. **`StyleRun.RunLengthArray` must sum to exactly `len(text)`.**
3. **`ParagraphRun.RunLengthArray` must sum to exactly `len(text)`.**
4. **Text must use `\r` for newlines** (not `\n`).
5. **Text must end with a trailing `\r`** (Photoshop convention).
6. **Delete the document-level `TEXT_ENGINE_DATA` block** so Photoshop
   rebuilds it from per-layer data on open.

A single-character mismatch in any RunLengthArray causes Photoshop to
**silently reject** the TypeLayer and fall back to showing pixel data.

### 5.4 Building a TypeLayer (From Scratch)

For each text block from the PDF:

1. **Create a transparent PixelLayer** at the text block's position and size.
   No text is drawn into the pixels — Photoshop renders text from EngineData.

2. **Build the TypeToolObjectSetting** structure:
   ```
   TypeToolObjectSetting:
     version: 1
     transform: (1.0, 0.0, 0.0, 1.0, left, top)  ← identity + translation
     text_version: 50
     text_data: DescriptorBlock("TxLr")
       "Txt ": String(normalized_text)
       "EngineData": RawData(engine_data_dict)
     warp_version: 1
     warp: DescriptorBlock("warp")  ← empty (no warp)
     left, top, right, bottom: bounding box
   ```

3. **Build the EngineData** dict-tree:
   ```
   EngineData:
     EngineDict:
       Editor:
         Text: "Hello World\r"              ← normalized text
       StyleRun:
         RunLengthArray: [12.0]             ← single float = len(text)
         RunArray: [{StyleSheet: {
           StyleSheetData: {
             Font: 0,                       ← index into FontSet
             FontSize: 24.0,
             AutoLeading: true,
             FillColor: {
               Type: 1,
               Values: [1.0, R, G, B]      ← alpha + RGB (0.0–1.0)
             }
           }
         }}]
       ParagraphRun:
         RunLengthArray: [12.0]             ← same as StyleRun
         RunArray: [{ParagraphSheet: {
           DefaultStyleSheet: 0,
           Properties: {}
         }}]
     ResourceDict:
       FontSet: [{
         Name: "Arial",
         Script: 0,
         FontType: 0,
         Synthetic: 0
       }]
       StyleSheetSet: [{Name: "Normal RGB"}]
       ParagraphSheetSet: [{
         Name: "Normal RGB",
         DefaultStyleSheet: 0,
         Properties: {}
       }]
   ```

4. **Inject the tagged block** into the PixelLayer's tagged blocks
   dictionary with key `TYPE_TOOL_OBJECT_SETTING`.

### 5.5 The Collapsed Single-Run Approach

Instead of creating one style run per text span (which requires exact
character counting including spaces between spans), use a **single collapsed
run** that covers the entire text:

```
RunLengthArray: [len(text)]    ← one element, total text length
RunArray: [first_style]        ← one style covering everything
```

This sacrifices per-span styling (mixed fonts/colors within a block) but
**guarantees** the RunLengthArray sum matches the text length.  It's the
safest approach when building TypeLayers from scratch.

### 5.6 Font Name Sanitization

PDF fonts often have subset prefixes: `ABCDEF+ArialMT`.  Strip the prefix
before the `+` sign.  Map common PostScript names to system font names:

| PDF Name            | System Name       |
|---------------------|-------------------|
| ArialMT             | Arial             |
| Arial-BoldMT        | Arial Bold        |
| Helvetica           | Helvetica         |
| TimesNewRomanPSMT   | Times New Roman   |

---

## 6. Template/Mutation Approach (Alternative)

If building TypeLayers from scratch doesn't produce reliable results, use
the **template approach**:

1. **Create a seed PSD** in Photoshop with one or more TypeLayers containing
   placeholder text.  Commit this file to your repository.

2. **At runtime**, open the seed PSD, mutate the TypeLayer text:
   - Update `Txt` descriptor key
   - Update `EngineData > Editor > Text`
   - Collapse `StyleRun` to single run, set `RunLengthArray = [len(new_text)]`
   - Collapse `ParagraphRun` similarly
   - Delete document-level `TEXT_ENGINE_DATA`

3. **Save** the modified PSD.

This is more reliable because the seed PSD has TypeLayers created by
Photoshop itself, with all internal structures valid.  You only mutate
the text content while preserving the structural scaffolding.

---

## 7. Coordinate Systems

| System       | Origin        | Y Direction | Unit    |
|--------------|---------------|-------------|---------|
| PDF          | Bottom-left*  | Up          | Points  |
| PyMuPDF      | Top-left      | Down        | Points  |
| PSD          | Top-left      | Down        | Pixels  |

*Note: PyMuPDF normalizes PDF coords to top-left origin, so no Y-flip needed.

**Conversion**: `pixel = point × (dpi / 72)`

---

## 8. Key Design Decisions

### Why background + text (not separate shapes/images)?

Extracting shapes and images separately from a PDF causes **overlap**:
clip-rendering a drawing captures everything in its bounding rect (including
text and other images).  This creates layers that overlap and cover each
other.  A single background + text approach avoids this entirely.

### Why transparent PixelLayer for text?

Photoshop renders TypeLayer text from EngineData, not from pixel data.
A transparent pixel layer ensures nothing covers the background content.
If we rasterized text into the pixel layer, it would be visible even if
the TypeLayer injection failed, causing confusing double-rendering.

### Why single collapsed StyleRun?

Per-span style runs require exact character counting including inter-span
spaces, newlines, and the trailing `\r`.  A single run covering the entire
text is mathematically guaranteed to match `len(text)`, eliminating the
most common cause of TypeLayer rejection by Photoshop.

### Why delete TEXT_ENGINE_DATA?

The document-level `TEXT_ENGINE_DATA` tagged block is a global mirror of
all text engine state.  If stale (not matching per-layer EngineData),
Photoshop may prioritize it and ignore layer-level changes.  Deleting it
forces Photoshop to rebuild from authoritative per-layer data.

---

## 9. Error Handling

- **Text extraction fails** → page becomes background-only (no TypeLayers)
- **TypeLayer injection fails** → layer stays as transparent PixelLayer
  (text won't be visible, but PSD structure is valid)
- **Font not available** → Photoshop prompts for font substitution on open
- **RunLengthArray mismatch** → Photoshop silently shows pixel data instead
  of editable text (the #1 bug to avoid)

---

## 10. JavaScript Implementation Notes

For a JS/Node.js reimplementation:

- **PDF parsing**: Use `pdf.js` (Mozilla) or `mupdf.js` (WebAssembly port)
- **Image manipulation**: Use `sharp` or `canvas` (node-canvas)
- **PSD writing**: Use `ag-psd` (supports reading/writing PSD with TypeLayers)
  or build binary PSD manually following the Adobe PSD spec
- **EngineData**: The EngineData blob is a custom binary format; `ag-psd`
  handles it natively.  If writing manually, study the psd-tools Python
  source for the exact binary encoding.

Key `ag-psd` TypeLayer fields:
```javascript
layer.text = {
  text: "Hello World\r",
  transform: [1, 0, 0, 1, left, top],
  engineData: { /* EngineDict + ResourceDict */ },
  // ag-psd has higher-level helpers for text layers
};
```

---

## 11. Summary of the Pipeline

```
Input:  PDF file (any source: Canva, Figma, InDesign, Word, etc.)
Output: PSD file with:
        - 1 background PixelLayer (full page render, text erased)
        - N TypeLayers (editable text, transparent pixel data)

Steps:
  1. Extract text blocks from PDF (position, content, font, color)
  2. Merge adjacent text blocks into logical groups
  3. Render full page as background raster image
  4. Erase text regions from background (prevent double-rendering)
  5. Create PSD with canvas size = page size
  6. Add background as PixelLayer at (0,0)
  7. For each text block:
     a. Create transparent PixelLayer at text position
     b. Build EngineData with single collapsed StyleRun
     c. Build TypeToolObjectSetting with matching Txt + EngineData
     d. Inject tagged block into PixelLayer
  8. Strip document-level TEXT_ENGINE_DATA
  9. Save PSD
```
