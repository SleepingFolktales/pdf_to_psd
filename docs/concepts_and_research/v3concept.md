# Vector Layer Extraction Architecture

## PDF Operator Segmentation → Isolated Rasterization → PSD Layer Assembly

**Problem Statement:** The current PDF→PSD pipeline only creates discrete layers for `paintImageXObject` (raster images) and text operations (`beginText`/`showText`/`endText`). All vector drawing operations — filled rectangles, clipped polygons, bezier paths, thick strokes acting as button backgrounds — are composited into a single monolithic "Background" pixel layer. This loses the layered editability that makes PSD files useful.

**Solution:** A three-phase extraction pipeline that walks the PDF.js operator list, segments it into discrete visual groups based on save/restore depth boundaries, classifies each group by its terminal drawing operation, and rasterizes vector groups to isolated transparent canvases for insertion as independent PSD layers at their correct z-order position.

---

## Phase 1: Operator Group Segmentation

### 1.1 Concept

A PDF page's content stream is a flat sequence of operators executed top-to-bottom in painter's order — later operations paint over earlier ones. PDF.js exposes this as two parallel arrays: `fnArray` (operator codes as integers) and `argsArray` (arguments for each operator). The z-order is implicit in the array index: operator `[0]` paints first (bottommost), operator `[409]` paints last (topmost).

The key structural signal is the **save/restore depth pattern**. PDF content streams use `q` (save) and `Q` (restore) to push and pop the graphics state — transform matrix, clip path, fill/stroke color, line width, blend mode. Canva (and most design tools) emit each visual element inside its own save/restore block: a `save` opens a fresh graphics state, the element's transform and clip are applied, the drawing operation fires, and a `restore` tears it all down.

This means that each **depth-2 or depth-3 save/restore cycle** in the trace corresponds to exactly one visual unit from the original design. Our job is to identify these cycles and package each one as a candidate layer.

The depth numbering works like this:

```
depth=0  (document root — the implicit outermost state)
  SAVE → depth=1  (page-level setup: global transform, page clip)
    SAVE → depth=2  (first visual element envelope)
      SAVE → depth=3  (element's local state: transform, color, clip)
        ... drawing ops ...
      RESTORE → depth=2
    RESTORE → depth=1
    SAVE → depth=2  (second visual element envelope)
      ...
```

In the Canva-generated PDF from the debug log, the repeating pattern is:

1. `SAVE` to depth 2 — opens the element's clip region (a rectangle bounding the element)
2. `CONSTRUCT PATH` + `CLIP` or `EOCLIP` — establishes the element's clip mask
3. `SAVE` to depth 3 — pushes a new state for the element's local transform and color
4. `TRANSFORM` — positions the element within the page
5. `setFillRGBColor` / `setStrokeRGBColor` — sets the element's colors
6. `SET GSTATE` — sets opacity, blend mode
7. Drawing operations — `FILL`, `STROKE`, `paintImageXObject`, or `beginText`...`endText`
8. `RESTORE` to depth 2
9. `RESTORE` to depth 1

Each depth-2-to-depth-1 cycle is one visual group.

### 1.2 Data Structures

Each segmented group needs to carry enough information for later classification and rasterization:

```js
/**
 * A single segmented visual group extracted from the operator list.
 * Represents one "visual unit" in the original design — a shape,
 * an image placement, or a text block.
 */
class OperatorGroup {
  constructor() {
    /** Inclusive start index in fnArray/argsArray */
    this.opStart = 0;
    /** Inclusive end index in fnArray/argsArray */
    this.opEnd = 0;
    /** Z-order position (0 = bottommost, ascending) */
    this.zIndex = 0;

    /**
     * Accumulated transform matrix at the point of the drawing operation.
     * This is the product of all TRANSFORM ops encountered from the
     * outermost save down to the drawing op.
     * Format: [a, b, c, d, e, f] matching the PDF/canvas convention:
     *   | a  c  e |
     *   | b  d  f |
     *   | 0  0  1 |
     */
    this.transform = [1, 0, 0, 1, 0, 0];

    /**
     * Clip regions accumulated for this group.
     * Each entry is { type: 'rect'|'path', coords: [...], rule: 'nonzero'|'evenodd' }
     * Multiple clips are intersected (nested clips).
     */
    this.clips = [];

    /** Fill color at drawing time: { r, g, b } in 0-255 range, or null */
    this.fillColor = null;

    /** Stroke color at drawing time: { r, g, b } in 0-255 range, or null */
    this.strokeColor = null;

    /** Graphics state properties: opacity (ca), blend mode (BM), line width (LW), etc. */
    this.graphicsState = {
      opacity: 1,
      blendMode: 'source-over',
      lineWidth: 1,
      lineCap: 0,
      lineJoin: 0,
      miterLimit: 10,
    };

    /**
     * The terminal drawing operations within this group.
     * Each entry describes one draw call:
     *   { type: 'fill'|'stroke'|'fillStroke'|'image'|'text',
     *     pathOps: [...],  // for vector: the constructPath data
     *     imageId: '...',  // for image: the XObject name
     *     textItems: [...] // for text: the showText sequences
     *   }
     */
    this.drawOps = [];

    /**
     * Computed bounding box in output pixel coordinates (post-viewport-transform).
     * { left, top, width, height }
     * Computed after segmentation by applying all transforms to the path/image bounds.
     */
    this.bounds = null;
  }
}
```

### 1.3 The Segmentation Walker

The walker is a single-pass scan of `fnArray` with a depth counter and a group accumulator. The critical insight is that we don't segment on every save/restore — we segment on the **outermost element-level** save/restore, which is depth 2 in Canva PDFs. Depth-3 saves are internal to the element and belong to the same group.

```js
const OPS = pdfjsLib.OPS; // PDF.js operator constants

/**
 * Segments a PDF.js operator list into discrete visual groups.
 *
 * @param {Object} opList - The result of page.getOperatorList()
 *   opList.fnArray:  Uint8Array of operator codes
 *   opList.argsArray: Array of argument arrays
 * @param {Array} viewportTransform - The viewport transform [a, b, c, d, e, f]
 *   from page.getViewport({ scale }).transform
 * @returns {OperatorGroup[]} - Ordered array of visual groups (z-order ascending)
 */
function segmentOperatorList(opList, viewportTransform) {
  const { fnArray, argsArray } = opList;
  const groups = [];

  let depth = 0;
  let currentGroup = null;
  let zCounter = 0;

  // Transform stack: each save pushes the current CTM, restore pops.
  // We maintain this to resolve the accumulated transform at each drawing op.
  const transformStack = [viewportTransform.slice()];
  let currentTransform = viewportTransform.slice();

  // Color state tracking
  let fillColor = null;
  let strokeColor = null;
  let graphicsState = {
    opacity: 1,
    blendMode: 'source-over',
    lineWidth: 1,
    lineCap: 0,
    lineJoin: 0,
    miterLimit: 10,
  };

  // Active clip regions for the current group
  let activeClips = [];

  // Pending path segments (constructPath ops accumulate until a fill/stroke fires)
  let pendingPath = null;

  for (let i = 0; i < fnArray.length; i++) {
    const op = fnArray[i];
    const args = argsArray[i];

    switch (op) {
      // ─── SAVE / RESTORE ───
      case OPS.save: {
        depth++;
        transformStack.push(currentTransform.slice());

        // When we enter depth 2 (the element envelope), start a new group
        if (depth === 2 && !currentGroup) {
          currentGroup = new OperatorGroup();
          currentGroup.opStart = i;
          currentGroup.zIndex = zCounter;
          currentGroup.clips = [];
        }
        break;
      }

      case OPS.restore: {
        // When we exit back to depth 1, finalize the current group
        if (depth === 2 && currentGroup) {
          currentGroup.opEnd = i;
          currentGroup.transform = currentTransform.slice();

          // Only keep groups that actually drew something
          if (currentGroup.drawOps.length > 0) {
            groups.push(currentGroup);
            zCounter++;
          }
          currentGroup = null;
          pendingPath = null;
        }

        depth--;
        currentTransform = transformStack.pop() || [1, 0, 0, 1, 0, 0];
        break;
      }

      // ─── TRANSFORM ───
      case OPS.transform: {
        // args = [a, b, c, d, e, f]
        // Multiply: currentTransform = currentTransform × args
        currentTransform = multiplyMatrices(currentTransform, args);

        // If we're inside a group, update its accumulated transform
        if (currentGroup) {
          currentGroup.transform = currentTransform.slice();
        }
        break;
      }

      // ─── COLOR STATE ───
      case OPS.setFillRGBColor: {
        // args = [r, g, b] in 0-65025 range (PDF.js uses 255*255 internally)
        fillColor = {
          r: Math.round((args[0] / 65025) * 255),
          g: Math.round((args[1] / 65025) * 255),
          b: Math.round((args[2] / 65025) * 255),
        };
        if (currentGroup) currentGroup.fillColor = { ...fillColor };
        break;
      }

      case OPS.setStrokeRGBColor: {
        strokeColor = {
          r: Math.round((args[0] / 65025) * 255),
          g: Math.round((args[1] / 65025) * 255),
          b: Math.round((args[2] / 65025) * 255),
        };
        if (currentGroup) currentGroup.strokeColor = { ...strokeColor };
        break;
      }

      // ─── GRAPHICS STATE ───
      case OPS.setGState: {
        // args[0] is an array of [key, value] pairs
        if (Array.isArray(args[0])) {
          for (const [key, value] of args[0]) {
            switch (key) {
              case 'ca':  graphicsState.opacity = value; break;
              case 'CA':  /* stroke opacity — track if needed */ break;
              case 'BM':  graphicsState.blendMode = value; break;
              case 'LW':  graphicsState.lineWidth = value; break;
              case 'LC':  graphicsState.lineCap = value; break;
              case 'LJ':  graphicsState.lineJoin = value; break;
              case 'ML':  graphicsState.miterLimit = value; break;
            }
          }
        }
        if (currentGroup) {
          currentGroup.graphicsState = { ...graphicsState };
        }
        break;
      }

      // ─── CLIPPING ───
      case OPS.clip:
      case OPS.eoClip: {
        // The clip applies to the most recently constructed path.
        // Store it so the rasterizer can apply it.
        if (pendingPath && currentGroup) {
          currentGroup.clips.push({
            type: pendingPath.type,
            coords: pendingPath.coords.slice(),
            rule: op === OPS.eoClip ? 'evenodd' : 'nonzero',
          });
        }
        pendingPath = null; // clip consumes the path
        break;
      }

      // ─── PATH CONSTRUCTION ───
      case OPS.constructPath: {
        // args[0] = array of path op codes (moveTo, lineTo, curveTo, rect, closePath)
        // args[1] = flat array of coordinates
        const pathOps = args[0];
        const coords = args[1];

        // Determine if this is a simple rectangle or a complex path
        const isRect = pathOps.length === 1 && pathOps[0] === OPS.rectangle;

        pendingPath = {
          type: isRect ? 'rect' : 'path',
          ops: pathOps.slice(),
          coords: coords.slice(),
        };
        break;
      }

      // ─── DRAWING OPERATIONS (terminals) ───
      case OPS.fill:
      case OPS.eoFill: {
        if (currentGroup && pendingPath) {
          currentGroup.drawOps.push({
            type: 'fill',
            pathData: { ...pendingPath },
            fillColor: fillColor ? { ...fillColor } : null,
            fillRule: op === OPS.eoFill ? 'evenodd' : 'nonzero',
          });
        }
        pendingPath = null;
        break;
      }

      case OPS.stroke: {
        if (currentGroup && pendingPath) {
          currentGroup.drawOps.push({
            type: 'stroke',
            pathData: { ...pendingPath },
            strokeColor: strokeColor ? { ...strokeColor } : null,
            lineWidth: graphicsState.lineWidth,
            lineCap: graphicsState.lineCap,
            lineJoin: graphicsState.lineJoin,
            miterLimit: graphicsState.miterLimit,
          });
        }
        pendingPath = null;
        break;
      }

      case OPS.fillStroke:
      case OPS.eoFillStroke: {
        if (currentGroup && pendingPath) {
          currentGroup.drawOps.push({
            type: 'fillStroke',
            pathData: { ...pendingPath },
            fillColor: fillColor ? { ...fillColor } : null,
            strokeColor: strokeColor ? { ...strokeColor } : null,
            fillRule: (op === OPS.eoFillStroke) ? 'evenodd' : 'nonzero',
            lineWidth: graphicsState.lineWidth,
          });
        }
        pendingPath = null;
        break;
      }

      case OPS.paintImageXObject: {
        if (currentGroup) {
          currentGroup.drawOps.push({
            type: 'image',
            imageId: args[0], // XObject name like "img_p0_2"
          });
        }
        break;
      }

      case OPS.beginText: {
        // Text groups are handled as a unit. We mark the start here
        // and collect showText ops until endText.
        if (currentGroup) {
          currentGroup._textAccumulator = {
            items: [],
            font: null,
            fontSize: 0,
            textMatrix: [1, 0, 0, 1, 0, 0],
          };
        }
        break;
      }

      case OPS.setFont: {
        if (currentGroup && currentGroup._textAccumulator) {
          currentGroup._textAccumulator.font = args[0];
          currentGroup._textAccumulator.fontSize = args[1];
        }
        break;
      }

      case OPS.setTextMatrix: {
        if (currentGroup && currentGroup._textAccumulator) {
          currentGroup._textAccumulator.textMatrix = args.slice();
        }
        break;
      }

      case OPS.showText: {
        if (currentGroup && currentGroup._textAccumulator) {
          currentGroup._textAccumulator.items.push({
            glyphs: args[0],
            font: currentGroup._textAccumulator.font,
            fontSize: currentGroup._textAccumulator.fontSize,
            textMatrix: currentGroup._textAccumulator.textMatrix.slice(),
          });
        }
        break;
      }

      case OPS.endText: {
        if (currentGroup && currentGroup._textAccumulator) {
          if (currentGroup._textAccumulator.items.length > 0) {
            currentGroup.drawOps.push({
              type: 'text',
              textItems: currentGroup._textAccumulator.items,
            });
          }
          currentGroup._textAccumulator = null;
        }
        break;
      }

      case OPS.moveText: {
        if (currentGroup && currentGroup._textAccumulator) {
          // dx, dy offset — adjust the text matrix
          const [dx, dy] = args;
          const tm = currentGroup._textAccumulator.textMatrix;
          // Td: translate the text matrix by (dx, dy)
          tm[4] += dx * tm[0] + dy * tm[2];
          tm[5] += dx * tm[1] + dy * tm[3];
        }
        break;
      }
    }
  }

  return groups;
}
```

### 1.4 Matrix Multiplication Helper

The transform accumulation requires multiplying 3x3 affine matrices in the PDF/canvas `[a, b, c, d, e, f]` format:

```js
/**
 * Multiplies two 2D affine transform matrices.
 * Input format: [a, b, c, d, e, f] representing:
 *   | a  c  e |
 *   | b  d  f |
 *   | 0  0  1 |
 *
 * @returns {number[]} The product matrix [a', b', c', d', e', f']
 */
function multiplyMatrices(m1, m2) {
  return [
    m1[0] * m2[0] + m1[2] * m2[1],       // a'
    m1[1] * m2[0] + m1[3] * m2[1],       // b'
    m1[0] * m2[2] + m1[2] * m2[3],       // c'
    m1[1] * m2[2] + m1[3] * m2[3],       // d'
    m1[0] * m2[4] + m1[2] * m2[5] + m1[4], // e'
    m1[1] * m2[4] + m1[3] * m2[5] + m1[5], // f'
  ];
}

/**
 * Applies a transform matrix to a point.
 * @returns {{ x: number, y: number }}
 */
function transformPoint(matrix, x, y) {
  return {
    x: matrix[0] * x + matrix[2] * y + matrix[4],
    y: matrix[1] * x + matrix[3] * y + matrix[5],
  };
}
```

### 1.5 Edge Cases and Robustness

**Non-Canva PDFs:** The depth-2 segmentation heuristic is tuned for Canva's output. Other tools (Illustrator, InDesign, Figma) may use different nesting depths. A more robust approach is to segment on **any save/restore block that contains a terminal drawing op** — regardless of depth. This means instead of checking `depth === 2`, you track whether a save/restore cycle at *any* depth contains at least one fill, stroke, or paintImage, and if so, emit it as a group.

**Shared state across groups:** Colors and line widths set in one group persist into the next (until overridden). The walker handles this by tracking state outside the group scope and copying it in when a group captures a drawing op.

**Nested clips (compound clipping):** Some elements use multiple clip operations before the draw — rectangle clip → rectangle clip → polygon clip (as seen in ops 33-43 of the trace). All clips within a group must be intersected when rasterizing. The group's `clips` array preserves them in order.

**Empty groups:** Some save/restore blocks only set up clipping or marked content without drawing anything. These produce groups with `drawOps.length === 0` and are discarded.

---

## Phase 2: Group Classification

### 2.1 Concept

Each segmented group now needs to be classified into one of three categories that determine how it's handled in the PSD assembly:

| Category | Terminal Op | Current Handling | Target Handling |
|----------|------------|------------------|-----------------|
| **Image** | `paintImageXObject` | Extracted as pixel layer | No change needed |
| **Text** | `beginText`...`endText` | Extracted as TypeLayer + canvas | No change needed |
| **Vector** | `fill`, `stroke`, `fillStroke` of a `constructPath` | Baked into background | **New: isolated rasterization → pixel layer** |

The classification is trivial — it's a direct lookup on the `type` field of each entry in `drawOps`. The more interesting work in Phase 2 is **sub-classification** of vector groups, which affects naming, grouping, and special handling.

### 2.2 The Classifier

```js
/**
 * Classification result for a single operator group.
 */
const LayerType = {
  IMAGE: 'image',
  TEXT: 'text',
  VECTOR_FILL: 'vector_fill',       // Filled shape (rectangle, polygon, etc.)
  VECTOR_STROKE: 'vector_stroke',    // Stroked path (lines, outlines)
  VECTOR_COMPOUND: 'vector_compound', // Multiple draws in one group (rare)
  BACKGROUND_FILL: 'background_fill', // Full-canvas fill → background layer
  BUTTON_BACKGROUND: 'button_bg',    // Thick stroke near text → button pill
};

/**
 * Classifies a segmented operator group and returns enriched metadata.
 *
 * @param {OperatorGroup} group - The segmented group from Phase 1
 * @param {number} canvasWidth - Output canvas width in pixels
 * @param {number} canvasHeight - Output canvas height in pixels
 * @param {OperatorGroup[]} allGroups - All groups (for adjacency checks)
 * @param {number} groupIndex - This group's index in allGroups
 * @returns {Object} Classification result with layer type and metadata
 */
function classifyGroup(group, canvasWidth, canvasHeight, allGroups, groupIndex) {
  const drawTypes = group.drawOps.map(d => d.type);
  const uniqueTypes = new Set(drawTypes);

  // ─── IMAGE ───
  if (uniqueTypes.has('image')) {
    const imageOp = group.drawOps.find(d => d.type === 'image');
    return {
      layerType: LayerType.IMAGE,
      name: imageOp.imageId,
      needsRasterization: false,
    };
  }

  // ─── TEXT ───
  if (uniqueTypes.has('text') && !uniqueTypes.has('fill') && !uniqueTypes.has('stroke')) {
    return {
      layerType: LayerType.TEXT,
      name: extractTextContent(group),
      needsRasterization: false,
    };
  }

  // ─── VECTOR: further sub-classify ───

  // Check 1: Is this a full-canvas background fill?
  const bounds = computeGroupBounds(group, canvasWidth, canvasHeight);
  const coverageRatio = (bounds.width * bounds.height) / (canvasWidth * canvasHeight);

  if (
    coverageRatio > 0.95 &&
    group.zIndex === 0 &&
    group.drawOps.length === 1 &&
    group.drawOps[0].type === 'fill'
  ) {
    return {
      layerType: LayerType.BACKGROUND_FILL,
      name: 'Background',
      color: group.fillColor,
      bounds,
      needsRasterization: false, // handled as PSD background
    };
  }

  // Check 2: Is this a button background?
  // Heuristic: a stroke with large line width (>20 in local coords)
  // immediately preceding a text group suggests a button/pill shape.
  const isThickStroke = group.drawOps.some(
    d => d.type === 'stroke' && d.lineWidth > 20
  );
  const nextGroup = allGroups[groupIndex + 1];
  const nextIsText = nextGroup && nextGroup.drawOps.some(d => d.type === 'text');

  if (isThickStroke && nextIsText) {
    return {
      layerType: LayerType.BUTTON_BACKGROUND,
      name: `Button BG: ${extractTextContent(nextGroup)}`,
      bounds,
      needsRasterization: true,
    };
  }

  // Check 3: Compound vector group (multiple draw ops)
  if (group.drawOps.length > 1) {
    return {
      layerType: LayerType.VECTOR_COMPOUND,
      name: generateVectorName(group, bounds),
      bounds,
      needsRasterization: true,
    };
  }

  // Check 4: Simple fill or stroke
  const drawOp = group.drawOps[0];
  if (drawOp.type === 'fill' || drawOp.type === 'fillStroke') {
    return {
      layerType: LayerType.VECTOR_FILL,
      name: generateVectorName(group, bounds),
      color: group.fillColor,
      bounds,
      needsRasterization: true,
    };
  }

  if (drawOp.type === 'stroke') {
    return {
      layerType: LayerType.VECTOR_STROKE,
      name: generateVectorName(group, bounds),
      color: group.strokeColor,
      bounds,
      needsRasterization: true,
    };
  }

  // Fallback
  return {
    layerType: LayerType.VECTOR_FILL,
    name: `Vector Z${group.zIndex}`,
    bounds,
    needsRasterization: true,
  };
}
```

### 2.3 Helper Functions for Classification

```js
/**
 * Extracts human-readable text content from a group's text draw ops.
 */
function extractTextContent(group) {
  const textOps = group.drawOps.filter(d => d.type === 'text');
  // In practice, connect this to your existing text extraction
  // that maps glyph codes → Unicode via the font's toUnicode map.
  // Simplified here — assumes textItems carry decoded strings.
  return textOps
    .flatMap(d => d.textItems)
    .map(item => item.decoded || '')
    .join(' ')
    .trim()
    .substring(0, 40);
}

/**
 * Generates a descriptive name for a vector layer based on its properties.
 */
function generateVectorName(group, bounds) {
  const color = group.fillColor || group.strokeColor;
  const colorHex = color
    ? `#${[color.r, color.g, color.b].map(c => c.toString(16).padStart(2, '0')).join('')}`
    : '';

  const shapeType = describeShape(group);
  const size = `${Math.round(bounds.width)}×${Math.round(bounds.height)}`;

  return `${shapeType} ${colorHex} (${size})`.trim();
}

/**
 * Describes the shape type based on path operations.
 */
function describeShape(group) {
  const firstDraw = group.drawOps[0];
  if (!firstDraw || !firstDraw.pathData) return 'Shape';

  const { ops } = firstDraw.pathData;
  if (!ops) return 'Shape';

  const hasRect = ops.includes(OPS.rectangle);
  const hasCurve = ops.includes(OPS.curveTo) || ops.includes(OPS.curveTo2) || ops.includes(OPS.curveTo3);
  const hasClose = ops.includes(OPS.closePath);

  // Check if transform includes rotation (non-zero b or c components)
  const [a, b, c, d] = group.transform;
  const isRotated = Math.abs(b) > 0.01 || Math.abs(c) > 0.01;

  if (hasRect && !hasCurve && !isRotated) return 'Rect';
  if (hasRect && !hasCurve && isRotated) return 'Rotated Rect';
  if (hasCurve && hasClose) return 'Closed Path';
  if (hasCurve) return 'Curve';
  if (hasClose) return 'Polygon';

  return 'Path';
}
```

### 2.4 Bounding Box Computation

Computing accurate bounds requires applying the group's accumulated transform to every coordinate in the path data:

```js
/**
 * Computes the bounding box of a group in output pixel coordinates.
 * Applies the group's full transform chain to all path coordinates.
 *
 * @returns {{ left: number, top: number, width: number, height: number }}
 */
function computeGroupBounds(group, canvasWidth, canvasHeight) {
  const transform = group.transform;
  let minX = Infinity, minY = Infinity;
  let maxX = -Infinity, maxY = -Infinity;

  function expandBounds(localX, localY) {
    const { x, y } = transformPoint(transform, localX, localY);
    if (x < minX) minX = x;
    if (x > maxX) maxX = x;
    if (y < minY) minY = y;
    if (y > maxY) maxY = y;
  }

  for (const drawOp of group.drawOps) {
    if (drawOp.type === 'image') {
      // Images are painted into a 1×1 unit square in local coords,
      // then scaled by the transform to their final size.
      expandBounds(0, 0);
      expandBounds(1, 0);
      expandBounds(0, 1);
      expandBounds(1, 1);
      continue;
    }

    if (!drawOp.pathData) continue;

    const { ops, coords } = drawOp.pathData;
    let coordIdx = 0;

    for (const pathOp of ops) {
      switch (pathOp) {
        case OPS.rectangle: {
          // coords: [x, y, width, height]
          const rx = coords[coordIdx++];
          const ry = coords[coordIdx++];
          const rw = coords[coordIdx++];
          const rh = coords[coordIdx++];
          expandBounds(rx, ry);
          expandBounds(rx + rw, ry);
          expandBounds(rx, ry + rh);
          expandBounds(rx + rw, ry + rh);
          break;
        }
        case OPS.moveTo:
        case OPS.lineTo: {
          expandBounds(coords[coordIdx++], coords[coordIdx++]);
          break;
        }
        case OPS.curveTo: {
          // 3 control points: cp1, cp2, endpoint
          // For tight bounds we'd need to solve the cubic,
          // but control-point bbox is a safe conservative estimate.
          for (let j = 0; j < 3; j++) {
            expandBounds(coords[coordIdx++], coords[coordIdx++]);
          }
          break;
        }
        case OPS.curveTo2:
        case OPS.curveTo3: {
          // 2 control points
          for (let j = 0; j < 2; j++) {
            expandBounds(coords[coordIdx++], coords[coordIdx++]);
          }
          break;
        }
        case OPS.closePath: {
          // No coordinates consumed
          break;
        }
      }
    }

    // For strokes, expand by half the line width (scaled)
    if (drawOp.type === 'stroke' || drawOp.type === 'fillStroke') {
      const lw = (drawOp.lineWidth || group.graphicsState.lineWidth) || 1;
      // Scale line width by the transform's scale factor
      const scaleX = Math.sqrt(transform[0] ** 2 + transform[1] ** 2);
      const halfStroke = (lw * scaleX) / 2;
      minX -= halfStroke;
      minY -= halfStroke;
      maxX += halfStroke;
      maxY += halfStroke;
    }
  }

  // Clamp to canvas
  minX = Math.max(0, Math.floor(minX));
  minY = Math.max(0, Math.floor(minY));
  maxX = Math.min(canvasWidth, Math.ceil(maxX));
  maxY = Math.min(canvasHeight, Math.ceil(maxY));

  return {
    left: minX,
    top: minY,
    width: maxX - minX,
    height: maxY - minY,
  };
}
```

### 2.5 Full Classification Pass

```js
/**
 * Classifies all segmented groups and produces a layer manifest.
 *
 * @param {OperatorGroup[]} groups - From Phase 1
 * @param {number} canvasWidth
 * @param {number} canvasHeight
 * @returns {Object[]} Classified groups with layer type, name, bounds, and rasterization flag
 */
function classifyAllGroups(groups, canvasWidth, canvasHeight) {
  return groups.map((group, index) => {
    const classification = classifyGroup(
      group, canvasWidth, canvasHeight, groups, index
    );
    return {
      ...classification,
      group,       // preserve the original group data for Phase 3
      zIndex: group.zIndex,
    };
  });
}
```

---

## Phase 3: Canvas Replay Rasterization

### 3.1 Concept

For every group classified as `needsRasterization: true`, we need to produce a transparent RGBA pixel buffer at the output DPI. The approach is **canvas replay**: create an off-screen `<canvas>` (or Node.js `Canvas` via `canvas`/`@napi-rs/canvas` package), apply the group's transform and clip state, then replay only that group's drawing operations against the canvas 2D API.

This is essentially doing what PDF.js's `CanvasGraphics` does internally, but in isolation — one canvas per vector group, rendering only the ops belonging to that group, on a transparent background.

The mapping from PDF operators to Canvas 2D API calls is direct:

| PDF Operator | Canvas 2D API |
|-------------|---------------|
| `constructPath` (moveTo) | `ctx.moveTo(x, y)` |
| `constructPath` (lineTo) | `ctx.lineTo(x, y)` |
| `constructPath` (curveTo) | `ctx.bezierCurveTo(cp1x, cp1y, cp2x, cp2y, x, y)` |
| `constructPath` (rectangle) | `ctx.rect(x, y, w, h)` |
| `constructPath` (closePath) | `ctx.closePath()` |
| `fill` | `ctx.fill()` |
| `eoFill` | `ctx.fill('evenodd')` |
| `stroke` | `ctx.stroke()` |
| `clip` | `ctx.clip()` |
| `eoClip` | `ctx.clip('evenodd')` |
| `setFillRGBColor` | `ctx.fillStyle = 'rgb(r,g,b)'` |
| `setStrokeRGBColor` | `ctx.strokeStyle = 'rgb(r,g,b)'` |
| `setGState` (ca) | `ctx.globalAlpha = value` |
| `setGState` (BM) | `ctx.globalCompositeOperation = value` |
| `setGState` (LW) | `ctx.lineWidth = value` |
| `save` | `ctx.save()` |
| `restore` | `ctx.restore()` |
| `transform` | `ctx.transform(a, b, c, d, e, f)` |

### 3.2 The Replay Engine

```js
const { createCanvas } = require('@napi-rs/canvas');
// Alternative: const { createCanvas } = require('canvas');

/**
 * PDF.js OPS constants for path operations (from pdf.js source).
 * These are the sub-opcodes used inside constructPath's first argument.
 */
const PATH_OPS = {
  moveTo: 13,     // OPS.moveTo
  lineTo: 14,     // OPS.lineTo
  curveTo: 15,    // OPS.curveTo (cubic bezier, 3 control points)
  curveTo2: 16,   // OPS.curveTo2 (2 control points variant)
  curveTo3: 17,   // OPS.curveTo3 (2 control points variant)
  closePath: 18,  // OPS.closePath
  rectangle: 19,  // OPS.rectangle
};

/**
 * Rasterizes a single vector group to a transparent RGBA buffer.
 *
 * The canvas is sized to the group's bounding box (not the full page),
 * with the origin translated so that the group's top-left corner maps
 * to canvas (0, 0). This produces a tight pixel buffer that can be
 * placed directly into the PSD at (bounds.left, bounds.top).
 *
 * @param {Object} classifiedGroup - From Phase 2 (contains .group and .bounds)
 * @param {number} canvasWidth - Full output canvas width (for reference)
 * @param {number} canvasHeight - Full output canvas height (for reference)
 * @returns {{ buffer: Buffer, left: number, top: number, width: number, height: number }}
 */
function rasterizeVectorGroup(classifiedGroup, canvasWidth, canvasHeight) {
  const { group, bounds } = classifiedGroup;

  // Add small padding to avoid edge clipping from anti-aliasing
  const pad = 2;
  const layerWidth = Math.max(1, bounds.width + pad * 2);
  const layerHeight = Math.max(1, bounds.height + pad * 2);
  const offsetX = bounds.left - pad;
  const offsetY = bounds.top - pad;

  const canvas = createCanvas(layerWidth, layerHeight);
  const ctx = canvas.getContext('2d');

  // Translate so that the group's world-space position maps correctly
  // onto this smaller canvas. The group's transform already includes
  // the viewport transform (which maps PDF coords → pixel coords),
  // so we only need to subtract our canvas offset.
  ctx.translate(-offsetX, -offsetY);

  // ─── Apply clip regions ───
  if (group.clips.length > 0) {
    for (const clip of group.clips) {
      ctx.save();
      ctx.beginPath();
      replayPathOnContext(ctx, clip, group.transform);
      ctx.clip(clip.rule);
    }
  }

  // ─── Apply the group's accumulated transform ───
  // The transform from Phase 1 is the full chain from viewport down to
  // the drawing operation's local space. We need to decompose this:
  //
  //   fullTransform = viewportTransform × pageTransform × ... × localTransform
  //
  // Since our canvas already has the viewport offset handled via translate(),
  // and the bounds were computed in viewport space, we apply the full
  // transform and let the path coordinates be in local space.
  const t = group.transform;
  ctx.setTransform(t[0], t[1], t[2], t[3], t[4] - offsetX, t[5] - offsetY);

  // ─── Set graphics state ───
  ctx.globalAlpha = group.graphicsState.opacity;
  ctx.globalCompositeOperation = group.graphicsState.blendMode;
  ctx.lineWidth = group.graphicsState.lineWidth;
  ctx.lineCap = ['butt', 'round', 'square'][group.graphicsState.lineCap] || 'butt';
  ctx.lineJoin = ['miter', 'round', 'bevel'][group.graphicsState.lineJoin] || 'miter';
  ctx.miterLimit = group.graphicsState.miterLimit;

  // ─── Replay each drawing operation ───
  for (const drawOp of group.drawOps) {
    switch (drawOp.type) {
      case 'fill':
      case 'fillStroke': {
        if (drawOp.fillColor) {
          const { r, g, b } = drawOp.fillColor;
          ctx.fillStyle = `rgb(${r},${g},${b})`;
        }
        if (drawOp.strokeColor) {
          const { r, g, b } = drawOp.strokeColor;
          ctx.strokeStyle = `rgb(${r},${g},${b})`;
        }
        if (drawOp.lineWidth) {
          ctx.lineWidth = drawOp.lineWidth;
        }

        ctx.beginPath();
        replayPathOps(ctx, drawOp.pathData);

        ctx.fill(drawOp.fillRule || 'nonzero');
        if (drawOp.type === 'fillStroke') {
          ctx.stroke();
        }
        break;
      }

      case 'stroke': {
        if (drawOp.strokeColor) {
          const { r, g, b } = drawOp.strokeColor;
          ctx.strokeStyle = `rgb(${r},${g},${b})`;
        }
        if (drawOp.lineWidth) ctx.lineWidth = drawOp.lineWidth;
        if (drawOp.lineCap !== undefined) {
          ctx.lineCap = ['butt', 'round', 'square'][drawOp.lineCap] || 'butt';
        }
        if (drawOp.lineJoin !== undefined) {
          ctx.lineJoin = ['miter', 'round', 'bevel'][drawOp.lineJoin] || 'miter';
        }

        ctx.beginPath();
        replayPathOps(ctx, drawOp.pathData);
        ctx.stroke();
        break;
      }
    }
  }

  // ─── Restore clip saves ───
  for (let i = 0; i < group.clips.length; i++) {
    ctx.restore();
  }

  // Extract RGBA pixel buffer
  const imageData = ctx.getImageData(0, 0, layerWidth, layerHeight);

  return {
    buffer: Buffer.from(imageData.data.buffer),
    left: offsetX,
    top: offsetY,
    width: layerWidth,
    height: layerHeight,
  };
}
```

### 3.3 Path Replay Helpers

```js
/**
 * Replays PDF.js path operations onto a Canvas 2D context.
 * Translates the constructPath sub-opcodes into ctx.moveTo/lineTo/etc.
 *
 * @param {CanvasRenderingContext2D} ctx
 * @param {Object} pathData - { ops: number[], coords: number[] }
 */
function replayPathOps(ctx, pathData) {
  const { ops, coords } = pathData;
  let ci = 0; // coordinate index

  for (const op of ops) {
    switch (op) {
      case PATH_OPS.moveTo: {
        ctx.moveTo(coords[ci++], coords[ci++]);
        break;
      }
      case PATH_OPS.lineTo: {
        ctx.lineTo(coords[ci++], coords[ci++]);
        break;
      }
      case PATH_OPS.curveTo: {
        // 6 coords: cp1x, cp1y, cp2x, cp2y, x, y
        ctx.bezierCurveTo(
          coords[ci++], coords[ci++],
          coords[ci++], coords[ci++],
          coords[ci++], coords[ci++]
        );
        break;
      }
      case PATH_OPS.curveTo2: {
        // 4 coords: cp2x, cp2y, x, y (cp1 = current point)
        // Canvas doesn't have a direct equivalent —
        // approximate by using current point as cp1
        const cp2x = coords[ci++], cp2y = coords[ci++];
        const x = coords[ci++], y = coords[ci++];
        // For accuracy, we'd need to track the current point.
        // Simplified: use quadraticCurveTo as approximation
        ctx.quadraticCurveTo(cp2x, cp2y, x, y);
        break;
      }
      case PATH_OPS.curveTo3: {
        // 4 coords: cp1x, cp1y, x, y (cp2 = endpoint)
        const cp1x = coords[ci++], cp1y = coords[ci++];
        const x = coords[ci++], y = coords[ci++];
        ctx.bezierCurveTo(cp1x, cp1y, x, y, x, y);
        break;
      }
      case PATH_OPS.rectangle: {
        ctx.rect(coords[ci++], coords[ci++], coords[ci++], coords[ci++]);
        break;
      }
      case PATH_OPS.closePath: {
        ctx.closePath();
        break;
      }
    }
  }
}

/**
 * Replays a clip path, handling the case where clip coordinates
 * may need transform adjustment.
 *
 * @param {CanvasRenderingContext2D} ctx
 * @param {Object} clipData - { type, coords, ops, rule }
 * @param {number[]} transform - The group's accumulated transform
 */
function replayPathOnContext(ctx, clipData, transform) {
  if (clipData.type === 'rect') {
    // Simple rectangle clip: coords = [x, y, w, h] in local space
    // The transform is already on the context, so use local coords
    const [x, y, w, h] = clipData.coords;
    ctx.rect(x, y, w, h);
  } else {
    // Complex path clip: replay the path ops
    replayPathOps(ctx, { ops: clipData.ops, coords: clipData.coords });
  }
}
```

### 3.4 Transform Handling: The Tricky Part

The most subtle aspect of canvas replay is getting the transform chain right. Here's the mental model:

```
PDF coordinate space (origin at bottom-left, y-up):
  ┌─────────────────────────┐
  │ (0, 810)         (810, 810) │  ← top of page in PDF coords
  │                             │
  │ (0, 0)           (810, 0)   │  ← bottom of page
  └─────────────────────────┘

Viewport transform flips Y and scales to DPI:
  [scale, 0, 0, -scale, 0, pageHeight * scale]
  For this PDF: [2.0, 0, 0, -2.0, 0, 1635.84]

Output pixel space (origin at top-left, y-down):
  ┌─────────────────────────┐
  │ (0, 0)         (1620, 0)    │  ← top of page in pixels
  │                             │
  │ (0, 1620)     (1620, 1620)  │  ← bottom
  └─────────────────────────┘
```

The Phase 1 walker accumulates transforms by multiplying them into `currentTransform`, starting from the viewport transform. By the time a drawing op fires, `group.transform` is the full chain from pixel space down to the element's local coordinate system.

For canvas replay, we apply this full transform to the canvas context via `ctx.setTransform()`. The path coordinates in `drawOp.pathData` are in the element's local space, so the transform maps them directly to pixel coordinates.

The offset subtraction (`t[4] - offsetX`, `t[5] - offsetY`) adjusts for the fact that our canvas is cropped to the bounding box rather than being the full page size.

**Critical caveat with clips:** Clip paths in the PDF are defined *before* the element's local transform is applied. They exist in a parent coordinate space. This means the clip replay needs to use the **clip's own transform context**, not the element's drawing transform. In practice, the clips are applied at depth 2 (before the depth-3 transform), so they need the transform that was active at depth 2.

To handle this correctly, the Phase 1 walker should also capture the transform that was active when each clip was defined:

```js
// In the walker, when processing clips:
case OPS.clip:
case OPS.eoClip: {
  if (pendingPath && currentGroup) {
    currentGroup.clips.push({
      type: pendingPath.type,
      ops: pendingPath.ops ? pendingPath.ops.slice() : undefined,
      coords: pendingPath.coords.slice(),
      rule: op === OPS.eoClip ? 'evenodd' : 'nonzero',
      // Capture the transform active at clip-definition time
      clipTransform: currentTransform.slice(),
    });
  }
  pendingPath = null;
  break;
}
```

Then in the replay:

```js
// Apply clips with their own transforms
for (const clip of group.clips) {
  ctx.save();
  // Set the clip's own transform (not the drawing transform)
  const ct = clip.clipTransform;
  ctx.setTransform(ct[0], ct[1], ct[2], ct[3], ct[4] - offsetX, ct[5] - offsetY);
  ctx.beginPath();
  replayPathOnContext(ctx, clip, clip.clipTransform);
  ctx.clip(clip.rule);
}

// Now set the drawing transform for the actual draw ops
const t = group.transform;
ctx.setTransform(t[0], t[1], t[2], t[3], t[4] - offsetX, t[5] - offsetY);
```

### 3.5 Handling Compound Vector Groups

Some groups contain multiple draw operations — like the three decorative × marks in ops 183-206, which are three separate bezier paths all in the same save/restore block with the same transform and color. These are handled naturally by the replay engine: each draw op in `group.drawOps` is replayed sequentially onto the same canvas, producing a single pixel layer that contains all three marks.

This is actually the *correct* behavior — grouping them into one layer matches Canva's intent (they're a single decorative element), and the user can still manipulate them as a unit in Photoshop.

### 3.6 Putting It Together: The Full Rasterization Pass

```js
/**
 * Rasterizes all vector groups that need isolated rendering.
 *
 * @param {Object[]} classifiedGroups - From Phase 2
 * @param {number} canvasWidth
 * @param {number} canvasHeight
 * @returns {Object[]} The same array, with rasterized groups now carrying
 *   a .pixelData property containing the RGBA buffer and placement info.
 */
function rasterizeVectorGroups(classifiedGroups, canvasWidth, canvasHeight) {
  return classifiedGroups.map(cg => {
    if (!cg.needsRasterization) return cg;

    const pixelData = rasterizeVectorGroup(cg, canvasWidth, canvasHeight);

    return {
      ...cg,
      pixelData,
    };
  });
}
```

---

## Integration: Assembling the PSD Layer Stack

### The Final Assembly

With all three phases complete, the PSD assembly receives a unified layer manifest where every visual element — whether it was originally a raster image, a text block, or a vector shape — is now a discrete layer with position, bounds, and (for vector groups) a pixel buffer.

```js
/**
 * Builds the PSD layer stack from classified and rasterized groups.
 * This replaces the current assembly logic that only handles
 * image + text layers.
 *
 * @param {Object[]} processedGroups - From Phase 3 (classified + rasterized)
 * @param {number} canvasWidth
 * @param {number} canvasHeight
 * @param {Buffer} backgroundPixels - Full-page raster for the background
 * @param {Object} imageCache - Map of imageId → pixel buffers (existing)
 * @param {Object[]} textLayers - Existing text layer data (existing)
 * @returns {Object} PSD document structure for ag-psd / writePsd
 */
function assemblePsdLayers(
  processedGroups, canvasWidth, canvasHeight,
  backgroundPixels, imageCache, textLayers
) {
  const layers = [];

  // Sort by z-index (ascending = bottom to top in paint order)
  // PSD layer order is top-to-bottom in the layers panel,
  // so we reverse for PSD serialization.
  const sorted = [...processedGroups].sort((a, b) => a.zIndex - b.zIndex);

  for (const pg of sorted) {
    switch (pg.layerType) {
      case LayerType.BACKGROUND_FILL: {
        // Handled separately as the PSD document background
        break;
      }

      case LayerType.IMAGE: {
        // Use existing image extraction logic
        const imgData = imageCache[pg.group.drawOps[0].imageId];
        if (imgData) {
          layers.push({
            name: `IMG: ${pg.name}`,
            left: pg.bounds.left,
            top: pg.bounds.top,
            canvas: imgData.canvas, // existing canvas element
            blendMode: 'normal',
            opacity: Math.round(pg.group.graphicsState.opacity * 255),
          });
        }
        break;
      }

      case LayerType.TEXT: {
        // Use existing text layer creation logic.
        // Match by z-index or position to the existing textLayers array.
        const matchingText = findMatchingTextLayer(textLayers, pg);
        if (matchingText) {
          layers.push(matchingText);
        }
        break;
      }

      case LayerType.VECTOR_FILL:
      case LayerType.VECTOR_STROKE:
      case LayerType.VECTOR_COMPOUND:
      case LayerType.BUTTON_BACKGROUND: {
        // ← This is the new path: vector groups become pixel layers
        if (pg.pixelData) {
          layers.push({
            name: pg.name,
            left: pg.pixelData.left,
            top: pg.pixelData.top,
            // Convert RGBA buffer to a canvas for ag-psd
            canvas: bufferToCanvas(
              pg.pixelData.buffer,
              pg.pixelData.width,
              pg.pixelData.height
            ),
            blendMode: pg.group.graphicsState.blendMode || 'normal',
            opacity: Math.round(pg.group.graphicsState.opacity * 255),
          });
        }
        break;
      }
    }
  }

  // Reverse for PSD (top of panel = first in array = topmost visually)
  layers.reverse();

  // Add background as the bottommost layer
  layers.push({
    name: 'Background',
    left: 0,
    top: 0,
    canvas: bufferToCanvas(backgroundPixels, canvasWidth, canvasHeight),
    blendMode: 'normal',
    opacity: 255,
  });

  return { width: canvasWidth, height: canvasHeight, children: layers };
}

/**
 * Converts an RGBA buffer to a canvas element (for ag-psd compatibility).
 */
function bufferToCanvas(rgbaBuffer, width, height) {
  const canvas = createCanvas(width, height);
  const ctx = canvas.getContext('2d');
  const imageData = ctx.createImageData(width, height);
  imageData.data.set(new Uint8Array(rgbaBuffer));
  ctx.putImageData(imageData, 0, 0);
  return canvas;
}
```

---

## Concrete Example: Tracing the Debug Log

To validate the architecture, here's how the three phases would process the specific operator trace from the "Grand Opening" PDF:

### Segmentation Output (Phase 1)

| Group | Op Range | Depth Cycle | Transform | Terminal Op |
|-------|----------|-------------|-----------|-------------|
| G0 | 8–17 | d2→d3→d2→d1 | scale 3.125 | `fill` (white rect) |
| G1 | 18–32 | d2→d3→d2→d1 | scale 3.125 | `fill` (purple rect) |
| G2 | 33–53 | d2→d3→d2→d1 | rotated [2.834, 1.316, ...] | `fill` (lilac polygon) |
| G3 | 54–74 | d2→d3→d2→d1 | rotated [3.026, 0.781, ...] | `fill` (white-ish polygon) |
| G4 | 75–89 | d2→d3→d2→d1 | scale [1828, 0, 0, -2128, ...] | `paintImageXObject` |
| G5 | 90–103 | d2→d3→d2→d1 | scale [1528, 0, 0, -1781, ...] | `paintImageXObject` |
| G6 | 104–182 | d2→d3→(mult)→d1 | multiple sub-groups | `text` (opening, GRAND ×2) |
| G7 | 183–206 | d2→d3→d2→d1 | scale 1.473 | `fill` (3× bezier paths) |
| G8 | 207–343 | d2→d3→(mult)→d1 | multiple sub-groups | `text` (Will Be Held On, etc.) |
| G9 | 344–354 | d3 (within G8's d2) | scale 3.125, offset | `stroke` (LW=58) |
| G10 | 356–385 | d3 (within G8's d2) | scale 3.125, offset | `text` (JOIN NOW!) |

### Classification Output (Phase 2)

| Group | LayerType | Name | Needs Raster? |
|-------|-----------|------|---------------|
| G0 | `BACKGROUND_FILL` | Background | No (PSD bg) |
| G1 | `VECTOR_FILL` | Rect #5E376D (1620×1620) | **Yes** |
| G2 | `VECTOR_FILL` | Rotated Rect #D4BBD5 (varies) | **Yes** |
| G3 | `VECTOR_FILL` | Rotated Rect #F8F8FF (varies) | **Yes** |
| G4 | `IMAGE` | img_p0_2 | No (existing) |
| G5 | `IMAGE` | img_p0_2 | No (existing) |
| G6 | `TEXT` | opening / GRAND | No (existing) |
| G7 | `VECTOR_COMPOUND` | Closed Path #5E376D (×3 marks) | **Yes** |
| G8 | `TEXT` | Will Be Held On / etc. | No (existing) |
| G9 | `BUTTON_BACKGROUND` | Button BG: JOIN NOW! | **Yes** |
| G10 | `TEXT` | JOIN NOW! | No (existing) |

### Rasterization Targets (Phase 3)

Only 5 groups need canvas replay: G1, G2, G3, G7, G9. Each produces an isolated transparent pixel buffer that becomes a new PSD layer.

### Resulting PSD Layer Stack (Top → Bottom)

```
T7: JOIN NOW!           ← text layer (existing, white on transparent)
Button BG: JOIN NOW!    ← NEW: rasterized thick stroke pill
T6: 10 July, 2023      ← text layer (existing)
T5: reallygreatsite.com ← text layer (existing)
T4: Will Be Held On     ← text layer (existing)
Decorative Marks (×3)   ← NEW: rasterized bezier compound
T1: opening             ← text layer (existing)
T2: GRAND (pink)        ← text layer (existing)
T3: GRAND (purple)      ← text layer (existing)
IMG 2 (734×856)         ← image layer (existing)
IMG 1 (878×991)         ← image layer (existing)
White Panel             ← NEW: rasterized rotated polygon
Lilac Diagonal          ← NEW: rasterized rotated polygon
Purple Panel            ← NEW: rasterized filled rect
Background              ← solid white (existing)
```

**Total: 15 layers** (up from 10), with full layer separation matching the original Canva design.

---

## Performance Considerations

**Canvas creation cost:** Each `createCanvas()` call allocates a pixel buffer. For a 1620×1620 canvas at 4 bytes/pixel, that's ~10MB. Most vector layers are much smaller (a button pill might be 300×80 = 96KB). The bounding-box-sized canvases keep memory reasonable.

**Number of rasterization passes:** One `createCanvas` + one draw pass per vector group. For this PDF, that's 5 passes. Even for complex Canva designs with 20-30 decorative elements, this is sub-second on modern hardware.

**Bezier precision:** The control-point bounding box method for curves slightly overestimates bounds (the true bounds of a cubic bezier may be tighter than its control polygon). This is acceptable — it means the pixel layer has a few extra transparent pixels around the edges, which doesn't affect visual fidelity.

**curveTo2/curveTo3 approximation:** The Canvas 2D API doesn't have a direct equivalent of PDF's `v` and `y` curve operators. The implementation above uses `quadraticCurveTo` and degenerate `bezierCurveTo` as approximations. For pixel-perfect fidelity, these should be converted to full cubic beziers by inferring the missing control point from the current point (which requires tracking `currentX`/`currentY` through the path replay).

---

## Testing Strategy

**Visual regression testing:** Render the full page to a reference canvas using PDF.js's standard `page.render()`. Then composite all extracted layers (background + vector pixel layers + image layers + text rasters) into a test canvas. Pixel-diff the two. Any significant difference indicates a segmentation or rasterization bug.

**Layer isolation test:** For each extracted vector layer, verify that its pixel buffer contains non-transparent pixels only within the expected region, and that compositing it at its stated (left, top) position onto the background produces the same visual result as the reference render in that region.

**Z-order verification:** Composite layers in z-order (bottom to top) and compare against the reference. Out-of-order compositing will show as incorrect overlaps — elements that should be behind appearing in front.