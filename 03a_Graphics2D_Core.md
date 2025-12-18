# 3. Graphics 2D: Core Primitives & Drawing

This chapter documents the foundational layer of the **Retained Mode 2D Engine**. Unlike immediate-mode APIs (like HTML5 Canvas `ctx.lineTo`), this engine requires you to construct immutable resource objects (`Path`, `Paint`, `Style`) and submit them to a `Scene`.

The renderer is resolution-independent, using floating-point coordinates. It handles anti-aliasing, tessellation, and colour space conversion automatically.

## 3.1 Coordinate System & Math Helpers

The coordinate system origin `(0,0)` is at the **top-left** corner.
*   **+X** extends to the right.
*   **+Y** extends downwards.
*   Units are logical pixels. In `HIRES_4_3`, the bottom-right is `(800, 600)`.

### Basic Types

These types are value objects used throughout the API.

| Type | Constructor | Description |
| :--- | :--- | :--- |
| **Vec2** | `Vec2(x, y)` | A 2D vector. Supports operators `+`, `-`, `*`, `/`. |
| **Rect** | `Rect(x, y, w, h)` | An axis-aligned rectangle. |
| **Color** | `Color(r, g, b, a)` | Constructs a colour. Inputs are **sRGB** floats [0.0–1.0]. The runtime converts these to **Linear Premultiplied** values immediately. |

**Note on Colour:**
```c
// 50% Red.
// Input: sRGB (1.0, 0, 0) @ 0.5 Alpha
// Stored internally as Linear Premultiplied: (0.5, 0, 0, 0.5)
local red = Color(1.0, 0.0, 0.0, 0.5);
```

## 3.2 The Scene Object

The `Scene` is the container for all draw commands for a single frame. You obtain it at the start of drawing and submit it at the end.

```c
// In your draw() function:
local scene = gfx.beginFrame();

// ... add draw commands to 'scene' ...

gfx.drawScene(scene); // Dispatch to GPU
```

| Method | Description |
| :--- | :--- |
| `scene.clear(color)` | Clears the entire framebuffer with the specified `Color`. **MUST** be the first command if used. |
| `scene.drawPath(path, style)` | Adds a path rendering command to the display list. |
| `scene.drawText(text, x, y, style)` | Adds a text rendering command (see Chapter 4). |

## 3.3 Path (Geometry)

A `Path` describes a geometric shape. It is a sequence of contours (lines and curves).
Paths are **mutable during construction** but should be treated as **immutable resources** once passed to `drawPath`. Reusing a `Path` object is highly optimised (the GPU caches the tessellation).

To create a path: `local p = gfx.Path();`

### Construction Methods
All methods return `this` to allow chaining.

| Command | Parameters | Description |
| :--- | :--- | :--- |
| **moveTo** | `x, y` | Moves the "pen" to coordinates `(x, y)` without drawing. Starts a new sub-path. |
| **lineTo** | `x, y` | Adds a straight line from current point to `(x, y)`. |
| **quadTo** | `cx, cy, x, y` | Quadratic Bézier curve. Control point `(cx, cy)`, end point `(x, y)`. |
| **cubicTo** | `c1x, c1y, c2x, c2y, x, y` | Cubic Bézier curve. Control points 1 & 2, end point `(x, y)`. |
| **arcTo** | `rx, ry, rot, large, sweep, x, y` | Elliptical arc (SVG style). Radii `rx, ry`, rotation `rot` (degrees), flags `large` (0/1) and `sweep` (0/1). |
| **close** | *none* | Closes the current sub-path with a straight line to the start point. |

### Shape Helpers
These methods append common shapes to the path.

| Helper | Parameters | Description |
| :--- | :--- | :--- |
| **rect** | `x, y, w, h, [rx, ry]` | Rectangle at `x,y`. Optional `rx, ry` for rounded corners. |
| **circle** | `cx, cy, r` | Circle centered at `cx, cy` with radius `r`. |
| **ellipse** | `cx, cy, rx, ry` | Ellipse centered at `cx, cy` with radii `rx, ry`. |
| **addPath** | `otherPath, [transform]` | Appends all contours from `otherPath`, optionally transforming them. |

### Utilities
*   `path.bounds()` → Returns `Rect` (the exact bounding box of the geometry).
*   `path.clear()` → Resets the path to empty (allows recycling the object).

## 3.4 Paint (Fill & Stroke Server)

A `Paint` defines *how* pixels are coloured. It can be a solid colour, a gradient, or an image pattern. Paints are immutable factories.

### 1. Solid Colour
```c
local p = gfx.Paint.color(r, g, b, a); // Uses standard Color() logic
```

### 2. Linear Gradient
```c
// Gradient from (x1,y1) to (x2,y2)
local p = gfx.Paint.linearGradient(x1, y1, x2, y2, stops, [units], [spread]);
```
*   **stops**: Array of tables `{offset=float, color=Color}`. Offset 0.0 to 1.0.
*   **units**: `gfx.Units.USER_SPACE` (default) or `gfx.Units.OBJECT_BBOX` (relative to shape 0..1).
*   **spread**: `gfx.Spread.PAD` (default), `REFLECT`, or `REPEAT`.

### 3. Radial Gradient
```c
// Gradient defined by a start circle and an end circle
local p = gfx.Paint.radialGradient(cx, cy, r, fx, fy, fr, stops, ...);
```
*   `cx, cy, r`: The outer circle.
*   `fx, fy, fr`: The focal (inner) circle.

### 4. Pattern (Image)
```c
local p = gfx.Paint.pattern(image, width, height, [units], [spreadX], [spreadY]);
```
*   Tiles an image/surface across the shape.

## 3.5 StrokeStyle (Line Attributes)

Defines the geometric properties of a stroke. Created via builder: `gfx.StrokeStyle()`.

```c
local ss = gfx.StrokeStyle()
    .width(4.0)                 // Thickness in user units
    .cap(gfx.LineCap.ROUND)     // BUTT, ROUND, SQUARE
    .join(gfx.LineJoin.MITER)   // MITER, ROUND, BEVEL
    .miterLimit(10.0)           // Max ratio for miter joins
    .dash([10, 5], 0);          // Dash array [on, off...] and phase offset
```

## 3.6 Style (The Binder)

The `Style` object combines geometry rules, paints, and stroke settings into a single state object passed to the draw call.

```c
local style = gfx.Style()
    .fill(paintObj)             // The Paint to use for filling
    .stroke(paintObj)           // The Paint to use for stroking
    .strokeStyle(strokeObj)     // The geometric stroke attributes
    .fillRule(gfx.FillRule.NONZERO) // NONZERO or EVENODD
    .order(gfx.PaintOrder.FILL_THEN_STROKE); // Render order
```

**Paint Orders:**
*   `FILL`: Fill only (ignores stroke).
*   `STROKE`: Stroke only (ignores fill).
*   `FILL_THEN_STROKE`: Standard vector behavior.
*   `STROKE_THEN_FILL`: Useful for "inner" strokes or specific artistic effects.

## 3.7 Comprehensive Example

This example creates a custom shape with a gradient fill and a dashed outline.

```c
local myPath;
local myStyle;

function init() {
    // 1. Build Geometry (The "What")
    myPath = gfx.Path();
    myPath.moveTo(100, 100);
    myPath.lineTo(300, 100);
    myPath.cubicTo(350, 100, 400, 150, 400, 200);
    myPath.lineTo(400, 400);
    myPath.lineTo(100, 400);
    myPath.close();
    
    // Add a hole (demonstrating fill rules)
    myPath.circle(250, 250, 50);

    // 2. Build Paints (The "Color")
    // Linear gradient from top-left to bottom-right
    local grad = gfx.Paint.linearGradient(100, 100, 400, 400, [
        { offset=0.0, color=Color(1, 0.2, 0, 1) },   // Orange
        { offset=1.0, color=Color(0.2, 0, 0.8, 1) }  // Purple
    ]);
    
    local outline = gfx.Paint.color(1, 1, 1, 1);

    // 3. Build Stroke Attributes (The "Thickness")
    local stroke = gfx.StrokeStyle()
        .width(5)
        .join(gfx.LineJoin.ROUND)
        .dash([15, 10], 0);

    // 4. Build Style (The "How")
    myStyle = gfx.Style()
        .fill(grad)
        .stroke(outline)
        .strokeStyle(stroke)
        .fillRule(gfx.FillRule.EVENODD) // Handles the hole correctly
        .order(gfx.PaintOrder.FILL_THEN_STROKE);
}

function draw() {
    local scene = gfx.beginFrame();
    
    scene.clear(Color(0.05, 0.05, 0.05, 1));
    
    // Submit the immutable resources to the scene
    scene.drawPath(myPath, myStyle);
    
    gfx.drawScene(scene);
}
```
