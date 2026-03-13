# PDF-to-PSD Converter — Architecture & Structure
## ag-psd (JavaScript) · Single HTML File · v2 Pipeline

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
    transform: [1, 0, 0, 1, left, top],   // identity + translation (horizontal)
    // OR:  [cosA, sinA, -sinA, cosA, left, top]  // rotation (vertical text)
    style: {
      font: { name: 'ArialMT' },
      fontSize: 24,
      fillColor: { r: 255, g: 0, b: 0 },
    },
  }
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

**Page rendering** — `page.render()` renders full page to Canvas at any DPI.

**Coordinate system:** PDF.js uses top-left origin after viewport transform — maps directly to canvas/PSD pixel coordinates.

---

## v2 Architecture: Pristine-First Extraction

The v1 pipeline rendered everything to one canvas, then tried to erase text with `fillRect`. This left colored rectangles, couldn't separate images, and baked vectors/shapes into the background.

**v2 inverts the order: extract from the pristine render first, then cut.**

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
└──────────┬───────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────┐
│  3. SAMPLE COLORS — from the pristine canvas         │
│     Per text item: most-common non-white pixel in    │
│     the item's bbox → accurate text fill color       │
└──────────┬───────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────┐
│  4. EXTRACT IMAGES — from the pristine canvas        │
│     Walk page.getOperatorList(), track CTM through   │
│     save/restore/transform.  On paintImageXObject:   │
│     map unit square through viewport×CTM → bbox.     │
│     Copy each bbox from the CLEAN canvas → layer.    │
│     Dedup near-identical bounds.  Skip < 30 px and   │
│     full-page fills (> 90 % area).                   │
└──────────┬───────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────┐
│  5. CLEAN BACKGROUND — clearRect (transparency)      │
│     For every text item:  clearRect its bbox         │
│     For every image layer: clearRect its bbox        │
│     Result: background has transparent holes where   │
│     text and images were — the layers above fill     │
│     those holes in the PSD composite.                │
└──────────┬───────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────┐
│  6. MERGE TEXT — group adjacent items into blocks     │
│     Same-line:  y-band < 65 % fh, x-gap ≤ 4× fh    │
│     Next-line:  x-align < 120 % fh, y-gap ≤ 2.5×fh │
│     Guards:     block width < 45 %, height < 30 %    │
│     Rotation:   never merge rotated + non-rotated    │
└──────────┬───────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────┐
│  7. ASSEMBLE PSD                                     │
│     Layer stack (top → bottom in Photoshop):         │
│       • TypeLayers (editable text, Photoshop renders)│
│       • Image pixel-layers (photos, logos, decor)    │
│       • Background pixel-layer (base design only)    │
│                                                      │
│     writePsd() with invalidateTextLayers: true       │
└──────────┬───────────────────────────────────────────┘
           │
           ▼
       PSD File
```

### Why v2 is better than v1

| Problem | v1 (old) | v2 (current) |
|---------|----------|--------------|
| Text on background | `fillRect` with sampled color → visible artifacts | `clearRect` → clean transparent holes |
| Image separation | Extracted AFTER canvas was modified | Extracted from PRISTINE render |
| Layer ordering | Images copied modified pixels | Images have original clean pixels |
| Vector shapes | Baked into flat background | Still baked (inherent PDF limit) but text+images cleanly separated |

### Why vector shapes remain in the background

PDF is a flat drawing model — shapes are just fill/stroke paint operations, not discrete objects. Extracting them as separate layers would require re-implementing a PDF renderer that categorizes every draw call. This is architecturally infeasible in a browser-only tool. The practical tradeoff: shapes stay in the background, but text and images (the parts users actually need to edit) are fully separated.

---

## PSD Layer Structure

```
PSD File (width × height = canvas pixels at DPI)
│
├── T1: "Greta Mae Evans"         ← TypeLayer (editable text)
├── T2: "Digital Marketing"       ← TypeLayer
├── T3: "About Me\nI have been…" ← TypeLayer (merged paragraph)
├── T4: "+123-456-7890"           ← TypeLayer (rotated transform)
│   ...
├── IMG 1 (310×310)               ← Pixel layer (photo)
├── IMG 2 (60×60)                 ← Pixel layer (ornament)
├── IMG 3 (120×40)                ← Pixel layer (logo)
│   ...
└── Background                    ← Pixel layer (shapes, borders, fills)
                                     Text + image holes = transparent
```

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
| Vector shapes not separated | Low | Inherent PDF limit — documented for users |

---

## Font Name Resolution

PDF.js internal names → system names:
- Strip subset prefix: `'ABCDEF+Montserrat-Bold'.split('+').pop()`
- Map PostScript: `ArialMT` → `Arial`, `TimesNewRomanPSMT` → `Times New Roman`

---

## CDN Dependencies

```
pdfjs-dist@3.11.174  — PDF parsing + rendering
ag-psd@30.1.0        — PSD file writing
FileSaver.js@2.0.5   — Browser file download
```

All loaded from jsdelivr with unpkg fallback. No build step required.
