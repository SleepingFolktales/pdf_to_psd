# PDF → PSD Converter: Complete Concept Guide

> Written for someone who has never opened Photoshop or worked with PSD files.

---

## First: What Even Is a PSD File?

A **PSD** (Photoshop Document) is Adobe Photoshop's native file format. The key thing that makes PSD special compared to formats like PNG or JPEG is that it stores a file as a **stack of independent layers** — like a stack of transparent sheets, each containing one piece of content.

Imagine you have a poster with:
- A background color
- A photo
- Some text
- A decorative shape

In a flat image format (PNG/JPEG), all of that is permanently baked together into one flat grid of pixels. If you want to move the text, you can't — it's fused with everything else.

In a PSD, each of those things lives on its own layer. You can:
- Click on the text layer and edit the words
- Move the photo without touching anything else
- Change the background color by editing only the background layer
- Resize the decorative shape without redrawing anything

### The Three Types of Layers in PSD

Understanding these three layer types is essential to understanding what our converter is trying to build:

**1. Pixel Layers (Raster Layers)**
The simplest kind. Just a grid of colored pixels, like a mini PNG floating at a certain position. Fast and reliable, but if you zoom in enough, you'll see individual square pixels. Not editable in any meaningful way — you can paint on it, but you can't say "change this word."

**2. Type Layers (Text Layers)**
A special layer where Photoshop remembers the *actual text* — the string of characters, the font, the size, and the color. The text is stored as real characters, not pixels. You can double-click it in Photoshop and retype the words, change the font, etc. Photoshop renders the visual pixels from this metadata on the fly.

**3. Shape Layers (Vector Layers)**
A special layer that stores a shape as **mathematical curves** (Bezier paths), not pixels. No matter how much you zoom in, the edge stays perfectly sharp because Photoshop recalculates the curve at whatever zoom level you're at. You can change the fill color, adjust the curve handles, etc.

> **The goal of our converter is to produce as many Type and Shape layers as possible, and only fall back to pixel layers when there is no other choice.** A PSD full of pixel layers is just a fancy ZIP of images — not useful for editing. A PSD full of proper text and shape layers is genuinely editable in Photoshop.

---

## What Is a PDF, Really? (Under the Hood)

A PDF (Portable Document Format) is not an image — it is actually a **list of drawing commands**. It reads like a program:

```
Set fill color to red
Move to position (100, 200)
Draw a rectangle 300 wide, 50 tall
Fill it
Set fill color to black
Move to position (120, 215)
Set font to Arial 14pt
Show text "Hello World"
```

These commands are called the **operator list**. The PDF rendering engine (in our case, **PDF.js** — Mozilla's open-source PDF reader for browsers) can execute this list to paint the final visual onto a screen.

Our converter intercepts this list *before* the painting happens, and instead of just rendering it into a flat image, it decodes what each command means, groups related commands into logical objects, and then builds a matching PSD layer for each one.

---

## The Coordinate Problem

PDF and PSD use different coordinate systems, and translating between them is one of the biggest technical challenges in this converter.

**PDF coordinate space:**
- Origin (0, 0) is at the **bottom-left**
- Y increases **upward**
- Units are "points" (1 point = 1/72 inch)

**Canvas/PSD coordinate space:**
- Origin (0, 0) is at the **top-left**
- Y increases **downward**
- Units are **pixels** (dependent on DPI setting)

On top of this, PDF supports nested coordinate transforms — a shape can be inside a group that is scaled, rotated, and translated, which is itself inside another group. Each PDF drawing command only knows its own **local coordinates**, not where it ends up on the final page.

To solve this, the converter tracks a **Current Transformation Matrix (CTM)** — a 6-number mathematical structure that encodes translation, scale, and rotation. Every time the PDF enters or exits a transformation context, the CTM is updated. When a drawing command finally fires, the CTM at that exact moment is saved alongside it, so the converter knows precisely where that shape ends up in pixel coordinates.

The transformation chain looks like this:

```
Local Coordinates (what the PDF command says)
  ↓  × the saved CTM at draw time
Device Coordinates (PDF's internal space)
  ↓  × the viewport transform (PDF.js scale factor + top-left flip)
Canvas Pixel Coordinates (pixels on screen, top-left origin)
```

This chain must be applied correctly to every single point of every single path, text position, and image corner.

---

## The Four-Phase Pipeline

The converter processes each PDF page through four sequential phases. Think of it as an assembly line where raw PDF data enters one end and a structured PSD layer stack emerges from the other.

---

### Phase 1: Operator Group Segmentation

**What it does:** Reads the raw operator list (the list of drawing commands) from the PDF and groups related commands together into logical "drawing units" called **Operator Groups**.

**Why this is necessary:** The PDF operator list is a flat stream — thousands of commands one after another with no inherent grouping. It looks like:
```
save
  transform [1.2, 0, 0, 0.8, 150, 300]
  save
    setFillColor [0.8, 0.2, 0.2]
    moveTo(10, 10)
    lineTo(100, 10)
    lineTo(100, 50)
    fill
  restore
  save
    setFontSize(14)
    showText("JOIN NOW")
  restore
restore
```

The `save` / `restore` commands create nested scopes (like opening and closing parentheses in code). The converter tracks **depth** — each `save` increases depth, each `restore` decreases it.

- **Depth 2 save/restore** = a major group boundary (like a component on the page — a button, a sidebar section)
- **Depth 3 save/restore** = a sub-element boundary within that component (like the background fill of the button vs. the label text of the button)

This nesting is critical. If the converter didn't split at depth-3 boundaries, a button's background rectangle and its text label would be bundled into one group — and it would be impossible to create a proper separate text layer and a separate shape layer for them.

**What it tracks during parsing:**
- The **CTM** (transformation matrix) at every depth level, starting from the viewport transform (not identity — this was a key bug fix, because starting from identity missed the entire PDF-to-canvas scale)
- A **snapshot of the CTM at the moment each draw operation fires** — this `ctmAtDraw` is saved with every draw op and used in all later phases
- The **fill color, stroke color, and line width** in effect at each draw op
- Any **clip paths** encountered before the draw ops in the same group (clipping regions that mask what gets drawn)

**Output:** An array of `OperatorGroup` objects, each containing:
- The actual draw operations (fill a path, stroke a path, paint an image, show text)
- The CTM snapshot for each draw op
- Any associated clip paths
- A z-index (its paint order number — lower = painted first = visually behind)

---

### Phase 2: Group Classification

**What it does:** Looks at each Operator Group produced by Phase 1 and decides what kind of Photoshop layer it should become.

**Why this is necessary:** Not all groups are the same. A group that only contains image paint commands should become an image pixel layer. A group that only contains text commands should become a text layer. A group that contains a single solid-color filled rectangle should become a shape layer. The converter needs to make this decision before it can build the right kind of layer.

**The classification logic (in priority order):**

| Type | Condition | Becomes |
|------|-----------|---------|
| **IMAGE** | Only image paint commands | A pixel layer containing the image |
| **TEXT** | Only text show commands | A Type layer (editable text) |
| **BACKGROUND_FILL** | Single fill covering >90% of the page at z-index 0 | The base background layer |
| **BUTTON_BACKGROUND** | Thick stroke + the next group is text | Shape layer (styled box around a label) |
| **VECTOR_COMPOUND** | Multiple non-text draw ops | Shape layer or rasterized fallback |
| **VECTOR_FILL** | Single fill or fill+stroke | Shape layer |
| **VECTOR_STROKE** | Single stroke only | Shape layer or rasterized line |

After initial classification, two **post-processing merges** happen:

**Clip-Companion Merge:** Some PDF elements are drawn as two separate groups — a fill group and a stroke group with the same clip path (the clipping region they share). These are really one shape with a fill and an outline, so they get merged into one `VECTOR_COMPOUND` group. A spatial overlap check (>50% overlap) prevents unrelated groups from being accidentally merged.

**Spatial Overlap Merge:** Thin fill-only groups (height ≤ 60px) that overlap heavily (>70%) are merged. This handles progress bars and similar widgets that PDF encodes as a stack of small slices.

**Bounds computation:** For every group, the converter computes an accurate pixel bounding box by transforming all the path coordinate points through each draw op's `ctmAtDraw`. For stroked paths, it also expands the bounding box by half the stroke width (scaled by the CTM's scale factor), because a stroke paints on both sides of the path line.

**Output:** An array of `ClassifiedGroup` objects, each now knowing what kind of layer it wants to become, what pixel rectangle it occupies, and whether it can be a native vector/text layer or needs to be rasterized.

---

### Phase 3: Layer Creation

This is the most complex phase. It splits into two sub-paths based on what the classification decided.

---

#### Phase 3A: Shape Layer Creation (Vector Path)

**What it does:** For groups classified as vector types with a solid fill or stroke color, this phase builds a true **PSD Shape Layer** with mathematical Bezier curves — not pixels.

**Why this matters:** A shape layer in PSD is resolution-independent and fully editable. The user can open the PSD in Photoshop and change the fill color, adjust the curve handles, or resize the shape without any quality loss. A rasterized pixel fallback can't do any of that.

**The core conversion: `pdfPathToPsdVectorMask`**

This is where PDF Bezier paths become PSD "knots." Let's understand both formats.

A **PDF path** is a sequence of commands:
- `moveTo(x, y)` — lift the pen and place it at this point (starts a new sub-path)
- `lineTo(x, y)` — draw a straight line to this point
- `curveTo(cp1x, cp1y, cp2x, cp2y, ex, ey)` — draw a cubic Bezier curve: two control points guide the curve shape, endpoint is where it lands
- `rectangle(x, y, w, h)` — shorthand for a closed 4-point rectangular path
- `closePath()` — connect the last point back to the first point

A **PSD knot** (ag-psd's format) stores each anchor point as:
```
{
  linked: true,
  points: [cp1x, cp1y, anchorX, anchorY, cp2x, cp2y]
}
```
Where:
- `anchor` = the point itself (where the path actually passes through)
- `cp1` = the incoming control point handle (pulls the curve as it arrives at this anchor)
- `cp2` = the outgoing control point handle (pulls the curve as it leaves this anchor)
- For a straight-line corner: all three are equal (cp1 = anchor = cp2)
- `linked: true` means the two handles move symmetrically (smooth curve)

The conversion walks every PDF path command and builds this structure. The trickiest part is `curveTo` — in PDF, a cubic Bezier's control points are associated with the *segment* (the space between two anchors), but in PSD, the control points are associated with the *anchors* themselves. So when we encounter a `curveTo`, we need to go back and update the *previous* anchor's outgoing control point (`cp2`), then create the new anchor with its incoming control point (`cp1`).

All coordinate values are transformed from local PDF space to pixel coordinates using the `ctmAtDraw` matrix at the time of the draw operation.

> **Critical v4.5 fix:** The knot points must be in `[x, y]` order. An earlier version incorrectly pre-swapped them to `[y, x]` thinking it was matching PSD's internal binary format — but the `ag-psd` library handles that swap internally. Providing `[y, x]` caused a double-swap, resulting in all shapes appearing transposed (mirrored along the diagonal).

**Path source selection — which path to use as the shape?**

This is subtle. Some PDF elements use a rectangular fill path, but a *curved clip path* to make the shape appear rounded. Think of a progress bar: the fill command draws a plain rectangle, but the clip region is a pill-shaped rounded rectangle. The *visual* shape the user sees is the pill, not the rectangle.

The converter uses this logic:
- If the draw operation's path itself has curves → use the draw-op path (it is the actual shape)
- If the draw-op path has no curves but there's a curved clip path → use the clip path as the shape (the pill case)
- If the draw-op path is rectangle-only → always use the draw-op path, even if a clip exists (the clip is just constraining it, not defining it)

**The `vectorFill` and `vectorStroke` properties:**

Once the path is converted to knots, the converter also attaches:
- `vectorFill` — the solid fill color: `{ type: 'color', color: { r, g, b } }`
- `vectorStroke` — stroke properties including color, width (scaled by CTM), alignment, etc.

For **stroke-only shapes** (like outline boxes, section dividers), no `vectorFill` is added — because adding a white fill to a stroke-only outline shape would incorrectly cover everything behind it.

**Preview canvas:**

PSD shape layers also require a rasterized thumbnail (a pixel preview). The converter always generates this by replaying the draw ops onto a canvas — so even if the vector data were misinterpreted by Photoshop, the pixel preview would still look correct visually.

---

#### Phase 3B: Rasterization Fallback

**What it does:** For groups that don't qualify for a shape layer (mixed content, images, complex compound paths, or 2-point line segments), the converter falls back to creating a **pixel layer** by rendering the group to a canvas.

**Why some things can't be shape layers:** PSD shape layers support a single vector path with a single solid fill. If a group has an image, or has multiple overlapping fills of different colors, or is just a 2-point straight line (which would create a degenerate zero-area shape), it can't meaningfully be a vector shape layer.

**How it works:** The converter creates a canvas sized to the group's bounding box (plus 2px padding), then:
1. Applies any clip paths using their clip-specific CTM
2. For each draw op, sets the canvas transform to `ctmAtDraw` (offset by the bounding box origin) and replays the path fill/stroke operations

The result is an isolated, transparent pixel layer that can be composited at the correct position in the PSD.

---

### Phase 4: Text Assembly, Color Matching, and Z-Order

This phase handles everything that isn't pure vector shapes, and then assembles all layers into the final correctly-ordered PSD stack.

---

#### Text Extraction and Color Matching

**Text extraction** uses PDF.js's `page.getTextContent()` method, which returns each text "item" with its string content, position, font name, font size, and direction. However, it does *not* reliably return text colors — those are embedded in the operator list as fill color commands that fire just before `showText` commands.

**Color matching problem:** The converter needs to connect the right color to each text item. Simply matching by index doesn't work because the operator list and the text content list don't maintain a 1:1 correspondence — some text items might be split, reordered, or drawn in non-sequential order.

**v4.5 solution — position-correlated matching:** The converter also walks the operator list, tracking the **text matrix** (a position matrix updated by `setTextMatrix`, `moveText`, `setLeadingMoveText`, and `nextLine` commands). Each time a `showText` fires, the current text matrix gives the canvas position of that text, and the current fill color is stored as `{ x, y, r, g, b }`.

Then, each text item from `getTextContent()` is matched to the nearest `showText` color emission by spatial distance, with a threshold of `2 × fontSize²`. This handles minor positioning differences and ensures the right color lands on the right text item.

#### Text Merging

Raw PDF text content is often fragmented — a single word like "Hello" might come back as three separate items `"H"`, `"el"`, `"lo"` due to kerning adjustments or font encoding. Before building text layers, these fragments must be intelligently merged into logical blocks (words, then lines, then paragraphs).

This is done in two passes:

**Pass 1 — Same-line word coalescing:**
- Sort all items by Y position, then X
- Merge adjacent items that share the same line (Y difference < 2px), same font/size/color/rotation, and whose X gap is less than 3× the average character width
- Spaces are automatically inserted where there's a visible gap

**Pass 2 — Paragraph merge:**
This is the most complex part of the entire converter, because merging too aggressively creates incorrect blocks (a heading merged with body text below it), but merging too conservatively leaves fragmented text layers that don't re-flow properly.

Ten guards protect against incorrect merges:

| Guard | What it checks | Why it matters |
|-------|---------------|----------------|
| ① Same line | Y within 65% of font height, X adjacent | Catch closely-spaced words on the same baseline |
| ② Next line | X within 120% font height, Y gap ≤ 1.8× font height | Only merge visually adjacent lines |
| ③ Rotation | Must match | Keep rotated text separate from horizontal text |
| ④ Font name | Must match exactly | Never merge a heading font with body text font |
| ⑤ Font size | Ratio ≤ 1.10 | Prevent a 36px title from absorbing 16px body text |
| ⑥ Color | RGB must match exactly | Respect intentional color changes |
| ⑦ Block size | Width < 45% canvas, height < 30% canvas | Prevent cross-column merges in multi-column layouts |
| ⑧ Section boundary | No thin vector lines between the two items | Respect visual dividers between sections |
| ⑨ Baseline step | Gap between last merged baseline and new item ≤ 2× font size | Prevents merging items with large vertical jumps (e.g., five separate percentage values) |
| ⑩ Heading interleave | No different-font items in the Y gap | Prevents body text on both sides of a heading from merging *through* the heading |

#### Image Layers

Images are extracted during the operator list walk in Phase 1 (they were passed through as part of the `ctmAtDraw` tracking). Each `paintImageXObject` command places an image in the **unit square [0,1]×[0,1]** in the current CTM's local coordinate space. The converter transforms the four corners of this unit square through `CTM × viewportTransform` to get pixel bounding box coordinates.

Filters applied during extraction:
- Skip images smaller than 30px (icons, decorative tiles)
- Skip images covering more than 90% of the canvas (background fill handled separately)
- Skip near-duplicate images (bounds within 5px of an already-extracted image)

Some images have circular or curved clip masks applied in the PDF (e.g., a profile photo cropped to a circle). The converter detects matching curved clip paths for image groups and applies them as `vectorMask` on the image layer in the PSD.

#### Z-Order Assembly (The Final Sort)

This is conceptually simple but critically important. PDF renders things in **paint order** — later things paint on top of earlier things. PSD layers are ordered the opposite way — the **topmost layer in the layers panel is the front-most visual element**.

Every layer (text, vector shape, image) has been tagged with a `_zIndex` value that was assigned sequentially as its originating Operator Group was emitted in Phase 1.

The assembly process:
1. Collect all built layers from all three types
2. Sort by `_zIndex` ascending (paint order: first painted = smallest index = visually behind)
3. Reverse the array (so the last-painted, frontmost item becomes the first PSD layer)
4. Append the Background pixel layer (the full-page raster render) as the bottom layer
5. Call `writePsd()` with `{ noBackground: true }` to prevent ag-psd from adding its own background

The result is a layer stack where visual stacking order is correct — foreground elements appear above background elements in Photoshop's layers panel, exactly as they do visually in the PDF.

---

## End-to-End Example

To make this concrete, here is what happens when the converter processes a simple PDF with a red button labeled "JOIN NOW":

**In the PDF operator list, this looks approximately like:**
```
save                          ← depth 2: new group
  transform [1, 0, 0, 1, 150, 300]
  save                        ← depth 3: sub-element 1 (the button background)
    setFillColor [0.8, 0, 0]  ← red
    rectangle(0, 0, 200, 50)
    fill
  restore                     ← flush sub-group 1
  save                        ← depth 3: sub-element 2 (the label)
    setFillColor [1, 1, 1]    ← white text
    setFont(Arial, 18)
    moveTo(40, 15)
    showText("JOIN NOW")
  restore                     ← flush sub-group 2
restore                       ← depth 2: close group
```

**Phase 1** emits two separate Operator Groups:
- Group A (z=1): fill command, path = rectangle, fill color = red, ctmAtDraw includes the translation
- Group B (z=2): text command, text = "JOIN NOW", color = white, position from ctmAtDraw

**Phase 2** classifies:
- Group A → `VECTOR_FILL` (single fill, solid color)
- Group B → `TEXT`

**Phase 3A** builds a Shape Layer for Group A:
- `pdfPathToPsdVectorMask` converts the rectangle to 4 straight knots (corners)
- `vectorFill` = `{ r: 204, g: 0, b: 0 }`
- Bounds: left=150, top=300, width=200, height=50

**Phase 4** builds a Type Layer for Group B:
- Text = "JOIN NOW", font = Arial, size = 18, color = white (matched by canvas position)
- Canvas positioned at the correct pixel coordinate

**Z-order sort and assembly:**
- Group B (z=2) goes first in the PSD layers list (topmost = frontmost)
- Group A (z=1) goes second
- Background render goes last (bottom of stack)

**Result in Photoshop:** A red shape layer beneath an editable white "JOIN NOW" text layer. The user can double-click the text to retype it, or click the shape layer to repaint it blue.

---

## Known Limitations

**Photoshop "Update Text" prompt:** When you first open the generated PSD, Photoshop shows a prompt asking to update text layers. This is expected — Photoshop needs to re-render the text using fonts installed on *your* machine, since the converter doesn't embed font files.

**RGB only:** The converter always outputs RGB color mode. If the source PDF uses CMYK colors (common in print production), the converter warns you and converts to RGB. The visual appearance may shift slightly.

**Font availability:** Text layers specify font names extracted from the PDF. If that font isn't installed on your machine, Photoshop will substitute a different font.

**Complex layouts:** Multi-column layouts with complex text flow may not merge perfectly. The merge guards are tuned to be conservative, so you may get more separate text layers than expected — but they won't be incorrectly combined across columns.

**2-point line shapes:** Straight lines (a `moveTo` + `lineTo` with no curves) can't be PSD shape layers because they have zero fill area. They are rasterized into pixel layers instead.

---

## Dependency Summary

| Library | Version | Role |
|---------|---------|------|
| **PDF.js** (`pdfjs-dist`) | 3.11.174 | Parses PDF, renders pages, extracts text content and operator list |
| **ag-psd** | 30.1.0 | Writes PSD files in the browser, handles all layer types |
| **FileSaver.js** | 2.0.5 | Triggers the browser file download of the generated PSD |

All three load from CDN. No server needed, no installation — the entire conversion runs client-side in the browser.