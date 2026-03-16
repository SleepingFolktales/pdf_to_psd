# ag-psd TypeLayer Bounds: Deep Research & Forward Path

**Date:** March 14, 2026  
**Context:** PDF-to-PSD conversion tool using ag-psd. Text (TypeLayer) bounds are `{0,0,0,0}` in Photoshop after write. Three independent root causes identified in v2.5 post-mortem.

---

## 1. What ag-psd Actually Does With TypeLayer Bounds

### 1.1 The Core Contract (Confirmed From Source & Docs)

The ag-psd README_PSD.md is explicit on how layer bounds are derived when writing:

> "**bottom and right values can be omitted and will always be ignored when writing** and will instead be calculated from canvas (or imageData) width and height."

This is the critical sentence. `bottom` and `right` are **not read at all** — they are always recomputed from canvas dimensions. `top` and `left` are honored, but only when a canvas is present to give the layer meaning.

The implication: **if you set `top`, `left`, `bottom`, `right` on a TypeLayer but provide no `canvas`, the layer has no spatial extent in the binary file.** ag-psd will write `top` and `left` from your values, but `bottom` and `right` become `top + 0` and `left + 0` (canvas is 0×0 = nothing). Photoshop reads this as `{0,0,0,0}`.

This behavior is **by design, not a bug**. ag-psd defers all visual rendering to the host application (the developer). The library is a PSD binary serializer, not a renderer.

### 1.2 The Required Pattern for Working TypeLayers

ag-psd explicitly documents this requirement:

> "This library also does NOT generate new layer canvas based on layer settings, so if you're changing any layer properties that impact layer bitmap, **you also need to update layer.canvas or layer.imageData.** This includes: **text layer properties**, vector layer properties, smart object, etc."

And from README_PSD.md:

> "**Vector, text and smart object layers still have image data with pregenerated bitmap. You also need to provide that image data when writing PSD files.**"

The correct structure for a working TypeLayer is therefore:

```js
{
  name: 'T1: opening',
  top: 387,          // honored by ag-psd
  left: 914,         // honored by ag-psd
  // bottom/right are IGNORED — derived from canvas
  canvas: renderedCanvas,  // ← REQUIRED for non-zero bounds
  text: {
    text: 'opening',
    transform: [1, 0, 0, 1, 914, 546],
    style: { font: { name: 'MontserratRoman-Bold' }, fontSize: 30 },
  },
}
```

Without `canvas`, bounds are zero. **This is confirmed behavior, not a known bug.**

### 1.3 What `invalidateTextLayers: true` Actually Does

From ag-psd's TypeScript source (`psd.ts`):

```ts
/** Invalidates text layer data, forcing Photoshop to redraw them on load.
 *  Use this option if you're updating loaded text layer properties. */
invalidateTextLayers?: boolean;
```

This flag was added specifically for the workflow of **reading an existing PSD, modifying text, and writing it back.** When set, it tells Photoshop to discard the stored pixel bitmap and re-render the TypeLayer from its metadata descriptor on open.

The critical condition: **Photoshop re-renders using its own font engine.** If the font named in the TypeLayer descriptor (`font.name`) is not installed on the user's machine, Photoshop cannot re-render, and the bounds remain zero in the reported manifest.

**This means `invalidateTextLayers: true` is harmful for your use case.** You are generating a PSD from scratch with CSS generic font names (`sans-serif`). Setting this flag tells Photoshop to throw away any stored pixel data you provided and re-render — but it can't because it doesn't have the font. **Leave it `false` or omit it entirely.**

### 1.4 Issue History Confirms This Understanding

GitHub Issue #53 (ag-psd) shows a user who:
1. Modified `text.text` on a layer
2. Set `canvas = undefined`
3. Used `writePsd(psd, { invalidateTextLayers: true })`
4. Read the output back — saw the old text, not the new text

The conclusion from that issue: ag-psd does NOT redraw text. It cannot. You must provide the canvas yourself. The `invalidateTextLayers` flag only signals Photoshop to redraw on open inside Photoshop itself.

---

## 2. Confirming the Three Root Causes

### Root Cause A — Confirmed ✅: ag-psd ignores explicit bounds on TypeLayers

**Status: Fully confirmed by official documentation.**

ag-psd derives `bottom` and `right` exclusively from canvas dimensions. Setting `top/left/bottom/right` on a TypeLayer object without providing a canvas will always result in zero-dimension bounds in the PSD binary. This is not a bug — it is the library's documented design.

**What your code was doing:** Setting `top`, `left`, `bottom`, `right` on TypeLayer JS objects without providing `canvas`. ag-psd wrote `top` and `left` correctly but computed `bottom = top + 0` and `right = left + 0` (zero canvas). Photoshop reads that as `{height: 0, width: 0}`.

### Root Cause B — Confirmed ✅: `invalidateTextLayers: true` + missing font = zero bounds

**Status: Confirmed by library behavior + Photoshop font rendering logic.**

When `invalidateTextLayers: true` is used, Photoshop attempts to re-render the TypeLayer from its descriptor on file open. With `font.name = "sans-serif"` (a CSS generic, not a PostScript/TrueType font name), Photoshop cannot locate the font in its font cache and renders nothing. The reported bounds are then zero.

Two compounding effects:
1. The flag is designed for a different workflow (read-modify-write an existing PSD)
2. CSS generic family names are not valid Photoshop font identifiers

### Root Cause C — Confirmed ✅: Font names from `getTextContent()` are CSS fallbacks, not real font names

**Status: Confirmed by PDF.js documentation and open issues.**

`getTextContent()` returns `fontName` as a style code (`g_d0_f2`) that maps to a CSS style entry via `textContent.styles`. Those CSS styles use `fontFamily: "sans-serif"` or `fontFamily: "serif"` — generic fallbacks for browser rendering.

The real PostScript font name is accessible via:
```js
const fontFace = page.commonObjs.get(item.fontName);
// fontFace contains: name, type, binary font data
```

However, `commonObjs` access requires the font to be fully loaded in PDF.js's renderer, which only happens after `page.render()` completes. Additionally, the actual PostScript name embedded in the PDF may differ from any installed Photoshop font name — Canva PDFs often use embedded subset fonts with names like `ABCDEF+Montserrat-Bold`, which Photoshop won't find by that name.

---

## 3. Can ag-psd Actually Solve This?

### Short Answer: **Yes, but you must do the rendering work yourself.**

ag-psd can write valid, functional TypeLayers that Photoshop opens with correct bounds — but only if you supply the rasterized canvas alongside the text metadata. The library has never claimed to auto-render text; it explicitly disclaims this in its README.

### What ag-psd *Can* Do:

| Capability | Supported |
|---|---|
| Write TypeLayer metadata (text, style, transform) | ✅ |
| Write pixel data for a layer (canvas property) | ✅ |
| Derive bounds from canvas dimensions | ✅ |
| Serialize font name, size, color to PSD descriptor | ✅ |
| Write boxBounds for paragraph text boxes | ✅ |
| Auto-render text to canvas from text property | ❌ (explicitly not supported) |
| Map CSS font names to PostScript names | ❌ (not a PDF tool) |
| Compute CTM-aware text positions from PDF.js operators | ❌ (out of scope) |

### What ag-psd *Cannot* Do (and Never Claimed To):

- **Render text to pixels.** The library has no text rendering engine. There is no planned support for this per the maintainer's stated scope.
- **Resolve font mismatches.** It accepts whatever font name you give it. Resolving CSS generics to real fonts is your responsibility.
- **Read the PSD spec's font rendering rules.** It serializes what you give it.

The library is correct and functional for its stated purpose. The problem is that your pipeline was not supplying the canvas data that the library requires.

---

## 4. The Fix Architecture (Direction A: Rasterized Canvas + TypeLayer)

This is the industry-standard approach used by all professional PSD generators. The principle:
- Provide the **canvas** (pixel data) so Photoshop has something to display immediately
- Provide the **text** descriptor so Photoshop knows it's a TypeLayer (editable when font becomes available)
- Set `invalidateTextLayers: false` (or omit it) so Photoshop uses your pixel data instead of trying to re-render

### 4.1 Core Pipeline for Each Text Item

```js
async function buildTextLayer(item, viewport) {
  // 1. Correct coordinate transformation (CTM-aware)
  const canvasX = item.transform[4] * viewport.scale;
  const canvasY = (viewport.height - item.transform[5]) * viewport.scale;
  const fontSize = Math.abs(item.transform[3]) * viewport.scale;

  // 2. Render text to an offscreen canvas (rasterized preview)
  const padding = 4; // small buffer so descenders don't clip
  const textCanvas = document.createElement('canvas');
  const metrics = measureText(item.str, fontSize, item.fontFamily);
  textCanvas.width = Math.ceil(metrics.width) + padding * 2;
  textCanvas.height = Math.ceil(fontSize * 1.4) + padding * 2;

  const ctx = textCanvas.getContext('2d');
  ctx.clearRect(0, 0, textCanvas.width, textCanvas.height);
  ctx.font = `${fontSize}px ${item.fontFamily}`;
  ctx.fillStyle = item.color; // extracted color from PDF.js
  ctx.fillText(item.str, padding, fontSize + padding * 0.5);

  // 3. Build the ag-psd layer with BOTH canvas and text
  return {
    name: item.str.substring(0, 32),
    top: Math.round(canvasY - fontSize),  // top of glyph bounding box
    left: Math.round(canvasX),
    // bottom/right are derived from canvas — do NOT set them
    canvas: textCanvas,   // ← gives ag-psd real dimensions for bounds
    text: {
      text: item.str,
      transform: [1, 0, 0, 1, canvasX, canvasY],
      style: {
        font: { name: resolvePostScriptFontName(item.fontName) },
        fontSize: fontSize,
        fillColor: item.colorRGB,
      },
    },
  };
}
```

### 4.2 The `top` and `left` Relationship to Canvas

A subtle but important point: with ag-psd, `top` and `left` specify where the canvas's top-left corner is placed in the document. The canvas size determines the layer's visual extent. So:

- `top` = document Y coordinate of the topmost pixel of your rendered canvas
- `left` = document X coordinate of the leftmost pixel of your rendered canvas
- The canvas must be sized exactly to contain the rendered text

If your canvas is 200px wide × 40px tall, and `top=100, left=50`, the layer occupies `{top:100, left:50, bottom:140, right:250}` in the PSD. **This is how ag-psd will report it, and it's correct.**

### 4.3 Font Name Resolution Strategy

The font name problem has three layers:

**Layer 1 — Get real font data from PDF.js:**
```js
// After page.render() completes:
const fontFace = page.commonObjs.get(item.fontName);
// fontFace.name might be: "ABCDEF+Montserrat-Bold" (embedded subset)
// Strip the subset prefix:
const cleanName = fontFace.name.replace(/^[A-Z]{6}\+/, '');
// → "Montserrat-Bold"
```

**Layer 2 — Map to a Photoshop PostScript name:**
```js
// Common Canva font mappings:
const fontMap = {
  'Montserrat-Bold':    'Montserrat-Bold',
  'Montserrat-Regular': 'Montserrat-Regular',
  'Lato-Regular':       'Lato-Regular',
  // Photoshop uses PostScript names, not display names
};
const psName = fontMap[cleanName] ?? 'ArialMT'; // fallback
```

**Layer 3 — Use the Canvas 2D API with available fonts:**
For the rasterized canvas, use whatever font is available in the browser. The TypeLayer metadata carries the ideal PostScript name. The canvas gives immediate visibility. This separation of concerns is the key insight.

### 4.4 `invalidateTextLayers` Setting

```js
const buffer = writePsd(psd);
// NOT: writePsd(psd, { invalidateTextLayers: true })
```

With `invalidateTextLayers` omitted (defaults to `false`), Photoshop opens the file and uses your rasterized canvas for display. It shows the "Update Text Layers?" prompt because it detects that the pixel data doesn't match what it would render from the TypeLayer descriptor. The user clicks **Update** and Photoshop re-renders cleanly from the descriptor — but only if they have the font. If they don't, the pixel preview (your canvas) remains visible as a fallback.

---

## 5. The CTM Bug in Operator Extraction (Discrepancy 2)

This is a separate issue from the ag-psd bounds problem, but it's blocking correct `top/left` values for the layer.

### The Problem

Your operator-based extraction ignores the Current Transform Matrix (CTM) stack. Text is drawn inside `save/transform/restore` groups, and the `setTextMatrix` arguments are in local coordinate space relative to the enclosing CTM — not in page space.

### The Proof (From Your Post-Mortem)

For T1 "opening":
```
Base viewport transform:    [0.240, 0, 0, -0.240, 0, 817.920]
Local CTM from TRANSFORM:   [3.125, 0, 0, 3.125, 555.879, 760.136]
Composed CTM = base × local: [0.75, 0, 0, -0.75, 133.411, 635.487]
Text matrix e,f:             (431.531, 121.000)

Page coords = CTM × textPos = (457.059, 544.737)
Canvas coords = viewport × pageCoords = (914.1, 546.4) ← matches getTextContent ✓
Your code: viewport × textPos directly = (863, 1394) ← wrong ✗
```

### The Fix

Don't use the operator extractor for position. **`getTextContent()` already gives you correctly composed page-space coordinates.** The `transform[4]` and `transform[5]` fields in each text item are the final page coordinates, fully accounting for all nested CTM transforms. They're verified correct in your post-mortem.

Use `getTextContent()` for positions. Drop the operator-based extraction entirely for position data. Only use the operator list for things `getTextContent()` doesn't expose, such as color (which your v2.4 extraction handles correctly).

---

## 6. Background Layer & Erase Strategy (Discrepancy 5)

### The Chicken-and-Egg Problem

With `erase=true`, `clearRect()` cuts transparent holes in the background canvas where text was. But with TypeLayers having zero bounds, the holes show nothing underneath. With the fix in place (TypeLayers with real canvas), this resolves itself:

- The background layer has holes where text was erased
- The TypeLayer sits above the background at the correct position with the correct canvas
- The composite result looks correct

However, there's a subtlety: the erase bounds must match the TypeLayer position exactly. Since you're now using canvas-derived bounds (not explicit `bottom/right`), you need to compute the erase rect from the same canvas dimensions:

```js
// After building textCanvas:
const eraseRect = {
  x: layer.left,
  y: layer.top,
  width: textCanvas.width,
  height: textCanvas.height,
};
backgroundCtx.clearRect(eraseRect.x, eraseRect.y, eraseRect.width, eraseRect.height);
```

---

## 7. What Photopea Does Differently

Photopea's PDF import (described in their 2018 blog post) converts PDFs to layered PSDs with:
- Text turned into proper TypeLayers with editable font names
- Vector shapes converted to PSD shape layers
- Raster images as pixel layers

Photopea open-sourced part of its PDF tooling as **PDFI.js**. The key difference from your pipeline:

1. Photopea renders each page element using its own compositing engine, which derives correct bounds from the rendered output — not from metadata alone
2. It resolves font names from the PDF's embedded font dictionary to PostScript names, then falls back to system fonts
3. It provides rasterized pixel data for every TypeLayer at the actual rendered size

This is exactly Direction A. Photopea is not doing anything architecturally special — it's just executing the same pattern ag-psd expects: **render first, serialize second.**

---

## 8. Alternative Libraries & Their Trade-offs

If the ag-psd approach proves too complex to maintain, here are the realistic alternatives:

### 8.1 psd-tools (Python)

- Full PSD read/write library in Python
- TypeLayer support including `frompil()` for pixel layers
- Would require a Python microservice for your otherwise JS/browser-side pipeline
- Better for server-side processing; not suitable for pure browser use

### 8.2 Aspose.PSD (Java / .NET / Python)

- Commercial library with `AddTextLayer()` method that accepts position as a `Rectangle`
- Handles font resolution and renders text internally
- Expensive license; not open source
- `textBoundBox` property gives accurate bounds after text rendering

### 8.3 Server-Side Photoshop Scripting (Node.js + Photoshop)

- Use Photoshop's UXP API or CEP scripting via a local server
- Photoshop itself handles all TypeLayer rendering and font resolution
- Requires Photoshop to be installed on the server — not viable for a web service

### 8.4 Photopea API (Headless)

- Photopea exposes a JavaScript API for automating document operations
- Can open a PDF and export PSD with correct layers
- Requires network access to photopea.com; not self-hosted
- Best-effort PDF conversion; complex Canva PDFs may have edge cases

### 8.5 Stay With ag-psd + Direction A

**This is the recommended path.** ag-psd is well-maintained (638 stars, active issues), the TypeLayer + canvas pattern is documented and supported, and Direction A is exactly what the library was designed for. The fix is scoped and implementable without architectural changes.

---

## 9. Summary: What To Do

### Root Cause Resolution Map

| Root Cause | Fix |
|---|---|
| **A** — ag-psd ignores explicit bounds | Always provide `canvas` property on TypeLayers. Derive bounds from canvas size, not from explicit `bottom/right`. |
| **B** — `invalidateTextLayers` + missing font | Remove `invalidateTextLayers: true`. Let Photoshop use your rasterized canvas pixel data for display. |
| **C** — CSS font names unusable in Photoshop | Extract real PostScript names via `page.commonObjs.get(fontName).name` after `page.render()`. Strip subset prefixes. Fall back to `ArialMT`. |
| **CTM bug** — operator positions in wrong space | Stop using operator extraction for position. Use `getTextContent()` transform[4]/[5] which are already in page space. |
| **Background erase** — holes with nothing underneath | Will self-resolve once TypeLayers have real bounds. Compute erase rect from canvas dimensions, not from explicit bounds. |

### Implementation Priority

1. **Stop setting explicit `bottom`/`right` on TypeLayers.** These values are ignored.
2. **Render each text item to an offscreen canvas using Canvas 2D API.** This is the critical missing step.
3. **Attach that canvas to the layer alongside the `text` property.** Both must be present.
4. **Remove `invalidateTextLayers: true`.** It actively harms your use case.
5. **Fix CTM in position extraction.** Use `getTextContent()` transform[4]/[5] for positions.
6. **Attempt real font name extraction via `page.commonObjs`.** Even partial success improves editability for users who have the fonts.

---

## 10. References

- [ag-psd README (main)](https://github.com/Agamnentzar/ag-psd/blob/master/README.md) — Documents `invalidateTextLayers`, canvas requirement
- [ag-psd README_PSD.md](https://github.com/Agamnentzar/ag-psd/blob/master/README_PSD.md) — "bottom and right values...will always be ignored when writing"
- [ag-psd src/psd.ts](https://github.com/Agamnentzar/ag-psd/blob/master/src/psd.ts) — TypeScript type definitions for WriteOptions
- [ag-psd Issue #3](https://github.com/Agamnentzar/ag-psd/issues/3) — "Layer left, top, bottom, right cannot be set" — confirms design
- [ag-psd Issue #53](https://github.com/Agamnentzar/ag-psd/issues/53) — Canvas must be provided alongside text
- [PDF.js Issue #7914](https://github.com/mozilla/pdf.js/issues/7914) — `getTextContent().styles.fontName` vs real font dict
- [Medium: PDF.js Font Extraction](https://medium.otranslator.com/pdf-transaction-how-to-use-pdf-js-b669f092fc1d) — `page.commonObjs.get(item.fontName)` pattern
- [Photopea Blog: PDF import](https://blog.photopea.com/photopea-3-3-pdf-import-and-export.html) — How Photopea converts PDF to PSD with real TypeLayers
- [psd-gen (ag-psd based)](https://github.com/arition/psd-gen) — Example showing text layer requiring "Update" prompt on open