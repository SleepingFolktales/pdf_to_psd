# PDF-to-PSD Converter — Feasibility Report
## ag-psd (JavaScript) · Single HTML File Approach

---

## Verdict: ✅ Feasible — with one known UX caveat

The plan is sound. A fully client-side, single-HTML-file PDF→PSD converter using **ag-psd** and **PDF.js** is achievable with no server, no install, and no build step. Both libraries ship browser bundles that load directly from CDN inside a `<script>` tag.

---

## Library Assessment

### ag-psd (v30.1.0)

The library is actively maintained (637 stars, latest release March 2025) and is the most capable open-source PSD writer in JavaScript.

**What it can do for this project:**
- Write PSD files entirely in the browser — no Node.js required
- Supports `layer.text` property, which creates real TypeLayers (editable "T" layers in Photoshop)
- Accepts `transform`, `style.font`, `style.fontSize`, `style.fillColor` per text layer
- Supports multi-span `styleRuns` for mixed colors/fonts within one block
- Outputs `ArrayBuffer` → trivially downloadable with `FileSaver.js`
- Has a pre-built bundle at `unpkg.com/ag-psd/dist/bundle.js` (loads as `window.agPsd`)

**Text layer API (the critical part):**
```js
{
  name: 'Text: Hello World',
  text: {
    text: 'Hello World\r',               // \r required, not \n
    transform: [1, 0, 0, 1, left, top],  // translate to position
    style: {
      font: { name: 'ArialMT' },
      fontSize: 24,
      fillColor: { r: 255, g: 0, b: 0 },
    },
  }
}
```
This is dramatically simpler than the raw EngineData/TypeToolObjectSetting approach you were fighting with in psd-tools. ag-psd handles all the binary blob construction internally.

**Known limitations relevant to this project:**
- Text layer implementation is described as "incomplete" in the docs — but the `text` property works reliably for the use case here (write text blocks from scratch)
- Writing vertical text orientation can crash Photoshop — not an issue for Canva exports
- Does NOT redraw the bitmap raster of text layers — this triggers a **Photoshop prompt** on open (see caveat below)
- No support for 16bpc or CMYK — Canva exports are always RGB 8bpc, so this is fine

**Why ag-psd will succeed where psd-tools failed:**

psd-tools requires you to manually construct EngineData binary blobs, keep `RunLengthArray` sums matching, manage `TEXT_ENGINE_DATA` deletion, and inject raw `TypeToolObjectSetting` tagged blocks. One character count mismatch silently drops the TypeLayer. ag-psd abstracts all of that into the `layer.text` object — it handles EngineData construction, `Txt`/`Editor.Text` sync, RunLengthArray, and the document-level cleanup automatically.

---

### PDF.js (pdfjs-dist)

Mozilla's PDF.js is the de facto standard for in-browser PDF rendering and text extraction. It's what Firefox uses internally.

**What it provides for this project:**

**Text extraction** — `page.getTextContent()` returns per-item objects with:
- `str` — the text string
- `transform` — `[scaleX, 0, 0, scaleY, x, y]` matrix (position + font size)
- `width`, `height` — bounding box dimensions
- `fontName` — e.g. `"g_d0_f2"` (internal name, requires font dictionary lookup)
- `dir` — text direction (ltr/rtl)

**Font metadata** — `page.getOperatorList()` combined with `PDFDocumentProxy.getFontData()` gives access to the font dictionary, including the "actual" font name (e.g. `ArialMT`, `ABCDEF+Montserrat-Bold`).

**Page rendering** — `page.render({ canvasContext, viewport })` renders the full page to an HTML Canvas at any DPI. This is the background layer.

**CDN availability** — `https://unpkg.com/pdfjs-dist/build/pdf.min.mjs` (ESM) or the legacy UMD build. The worker can point to `https://unpkg.com/pdfjs-dist/build/pdf.worker.min.mjs`.

**Coordinate system note:** PDF.js uses top-left origin (unlike native PDF which is bottom-left). Coordinates map directly to canvas pixels after applying `viewport.scale` — no Y-flip required, matching the PSD coordinate system perfectly.

---

## Architecture: Single HTML File

Both libraries are loadable from CDN with no bundler. The entire app runs in the browser.

```
┌─────────────────────────────────────────────────────┐
│  Single HTML File (no server, no install)           │
│                                                     │
│  CDN imports:                                       │
│  • pdfjs-dist/build/pdf.min.js  (PDF parsing)      │
│  • ag-psd/dist/bundle.js        (PSD writing)      │
│  • FileSaver.js                 (download)          │
│                                                     │
│  Flow:                                              │
│  [Drop PDF] → PDF.js extracts text + renders bg     │
│            → Merge adjacent text blocks             │
│            → Erase text from background canvas      │
│            → ag-psd builds PSD with TypeLayers      │
│            → FileSaver downloads .psd               │
└─────────────────────────────────────────────────────┘
```

Everything runs client-side. The PDF never leaves the user's browser. Share the HTML file via Slack, email, USB — it just works.

**One technical wrinkle with the PDF.js worker in a local file:**  
When opening the HTML from the filesystem (`file://`), some browsers block cross-origin workers. The fix is to use the "fake worker" mode:
```js
pdfjsLib.GlobalWorkerOptions.workerSrc =
  'https://unpkg.com/pdfjs-dist/build/pdf.worker.min.mjs';
```
Chrome accepts this. Firefox with strict local file security may not — in that case the HTML should be served from a simple local HTTP server (e.g. `python -m http.server 8080`), or you can inline the worker as a Blob URL. This can be documented in a README next to the file.

---

## The One UX Caveat: Photoshop's Update Prompt

When Photoshop opens a PSD with text layers created programmatically (by any library, not just ag-psd), it shows this dialog:

> **"Some text layers need to be updated before they can be used for vector-based output. Do you want to update these layers now?"**
> `[Update]` `[No]`

**You must click "Update."** This is a one-time action per file open. After clicking Update, the text layers are fully editable TypeLayers — exactly as if you had created them in Photoshop yourself. This is the same behavior documented in ag-psd's README and is the expected result from any external PSD writer.

This is NOT the same as psd-tools's failure mode (silently falling back to pixel data). ag-psd's layers ARE real TypeLayers — Photoshop just needs to rebuild the raster cache from the EngineData on first open.

---

## Implementation Plan

### Stage 1 — PDF Parsing

```js
const pdf = await pdfjsLib.getDocument(arrayBuffer).promise;
const page = await pdf.getPage(1);

// Background render
const viewport = page.getViewport({ scale: 2.0 }); // 144 DPI
const canvas = document.createElement('canvas');
canvas.width = viewport.width;
canvas.height = viewport.height;
await page.render({ canvasContext: canvas.getContext('2d'), viewport }).promise;

// Text extraction
const textContent = await page.getTextContent();
const items = textContent.items; // [{str, transform, width, height, fontName}]
```

Each item's position comes from `transform[4]` (x) and `transform[5]` (y). Font size is `Math.abs(transform[3])`. This maps directly to PSD pixel coordinates after applying the same scale.

### Stage 2 — Text Block Merging

Group `items` that are:
- Same font family (strip subset prefix before `+`)
- Horizontally aligned within ~20px
- Vertically adjacent (gap ≤ 1.5× font height)

Compute union bounding box, concatenate strings with `\r` as line separator.

### Stage 3 — Background Cleanup

For each merged text block, erase its bounding box from the background canvas:
```js
ctx.clearRect(x - 2, y - 2, width + 4, height + 4);
```

### Stage 4 — PSD Assembly with ag-psd

```js
const { writePsd } = agPsd;

const psd = {
  width: canvas.width,
  height: canvas.height,
  children: [
    // Background layer
    {
      name: 'Background',
      canvas: backgroundCanvas,
    },
    // One TypeLayer per text block
    ...textBlocks.map(block => ({
      name: `Text: ${block.text.slice(0, 30)}`,
      text: {
        text: block.text + '\r',
        transform: [1, 0, 0, 1, block.left, block.top],
        style: {
          font: { name: sanitizeFontName(block.fontName) },
          fontSize: block.fontSize,
          fillColor: block.color,
        },
      },
    })),
  ],
};

const buffer = writePsd(psd, { invalidateTextLayers: true });
saveAs(new Blob([buffer]), 'output.psd');
```

The `invalidateTextLayers: true` option is what triggers the Photoshop update prompt — it tells Photoshop to redraw the text rasters from the EngineData, which is exactly what we want.

---

## Risk Table

| Risk | Severity | Mitigation |
|------|----------|------------|
| Photoshop update prompt on open | Low | Expected behavior, one click, fully documented |
| Font not installed on user's machine | Medium | Photoshop asks for substitution — user picks a replacement |
| PDF.js worker blocked on `file://` | Low | Serve from `localhost` or inline worker as Blob URL |
| Font name extraction from PDF.js | Medium | `fontName` from `getTextContent` is internal; need to cross-reference `page.commonObjs` for actual font name |
| Complex multi-column layouts | Medium | Merging heuristic may group wrong blocks — tunable thresholds |
| Very large PDFs (10+ pages) | Low | Process page-by-page, show progress bar |
| Canva-specific font subsets (`ABCDEF+`) | Low | Strip prefix before `+`, map common PostScript names to system names |

---

## Font Name Resolution

PDF.js exposes internal font names from `getTextContent` (e.g. `g_d0_f2`). To get the real font name, resolve via:

```js
const ops = await page.getOperatorList();
// After render, fonts are loaded into commonObjs
page.commonObjs.get(fontName, (font) => {
  const realName = font.data?.fontFamily || font.name;
});
```

Alternatively, render first (which loads fonts), then query. For a safe fallback:
- Strip subset prefix: `'ABCDEF+Montserrat-Bold'.split('+').pop()` → `'Montserrat-Bold'`
- Map PostScript suffixes: `ArialMT` → `Arial`, `Arial-BoldMT` → `Arial Bold`

---

## Comparison: ag-psd vs psd-tools

| Dimension | psd-tools (Python) | ag-psd (JavaScript) |
|---|---|---|
| Text layer creation | Manual EngineData binary | `layer.text` object |
| RunLengthArray management | Manual, error-prone | Automatic |
| TEXT_ENGINE_DATA cleanup | Manual deletion required | `invalidateTextLayers` flag |
| Failure mode | Silent (shows pixel data) | Photoshop update prompt |
| Browser deployment | No (server required) | Yes (CDN bundle) |
| Single file app | No | Yes |

---

## What to Build Next

The recommended path:

1. **HTML prototype** — single file, CDN imports, basic pipeline (no merging, no font resolution). Get a PSD opening in Photoshop with TypeLayers.
2. **Add text block merging** — tune thresholds against real Canva exports.
3. **Add font resolution** — map PDF internal names to system font names.
4. **Polish** — drag-and-drop, progress bar, multi-page support, DPI selector.

The prototype can be done in one session. The polish is where the iteration happens.