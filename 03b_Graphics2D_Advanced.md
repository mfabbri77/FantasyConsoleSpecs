# 3. Graphics 2D: Advanced Compositing & Effects

This chapter expands on the core drawing primitives by introducing the hierarchical state management system. In a retained mode engine, you manipulate the **Context State** to affect all subsequent draw commands. This includes coordinate transformations, clipping regions, layer compositing, and advanced post-processing filters.

## 3.8 State Management

The rendering context maintains a stack of states. Each state contains:
1.  The current **Transformation Matrix**.
2.  The current **Clipping Region**.
3.  The current **Mask**.

You **MUST** balance every `save()` with a `restore()`.

| Method | Description |
| :--- | :--- |
| `scene.save()` | Pushes a copy of the current state onto the stack. |
| `scene.restore()` | Pops the last state, reverting transforms and clips to their previous values. Throws an error if the stack is empty. |

## 3.9 Transformations

Coordinate transformations allow you to move, rotate, scale, and skew the drawing grid. The engine uses a 3×2 affine matrix structure `[a, b, c, d, e, f]` equivalent to the SVG matrix:

$$
\begin{bmatrix}
a & c & e \\
b & d & f \\
0 & 0 & 1
\end{bmatrix}
$$

### The Transform Object
The `gfx.Transform` object is immutable. You chain methods to build complex matrices.

| Factory Method | Description |
| :--- | :--- |
| `gfx.Transform.identity()` | Returns the identity matrix (no change). |
| `gfx.Transform.translate(dx, dy)` | Moves the origin by `(dx, dy)`. |
| `gfx.Transform.scale(sx, [sy])` | Scales by `sx` (and optional `sy`). |
| `gfx.Transform.rotate(angle)` | Rotates by `angle` degrees clockwise around the origin (0,0). |
| `gfx.Transform.skewX(angle)` | Skews along the X-axis by `angle` degrees. |
| `gfx.Transform.skewY(angle)` | Skews along the Y-axis by `angle` degrees. |

### Composition & Application
Matrices are multiplied **last-multiplied, first-applied**.
To apply a transform to the current scene state:

```c
// Create a transform: Rotate 45 degrees around a pivot point (100, 100)
local t = gfx.Transform.translate(100, 100)
    .multiply(gfx.Transform.rotate(45))
    .multiply(gfx.Transform.translate(-100, -100));

scene.save();
scene.transform(t); // Apply to current state
// ... draw commands here are rotated ...
scene.restore();
```

## 3.10 Clipping

Clipping constrains drawing to a specific region. The clipping region is defined by a `Path`.

*   **Intersection:** New clips intersect with the current clip. You can only shrink the visible area, never expand it, until you `restore()`.
*   **Anti-aliasing:** Clip boundaries are fully anti-aliased.

```c
scene.save();
scene.clipPath(myPath, gfx.FillRule.NONZERO);
// ... drawing outside 'myPath' is discarded ...
scene.restore();
```

## 3.11 Grouping & Blending

Groups allow you to render a sequence of commands into an offscreen buffer (layer) and then composite that result back into the scene. This is essential for applying opacity to a collection of overlapping shapes or applying filters.

| Method | Description |
| :--- | :--- |
| `scene.groupBegin(options)` | Pushes a new compositing layer. |
| `scene.groupEnd()` | Composites the layer onto the parent. |

### Group Options
The `options` table supports:
*   `opacity` (float): 0.0 to 1.0. Multiplies the alpha of the entire group.
*   `blendMode` (enum): How the group merges with the background.
    *   `gfx.BlendMode.SRC_OVER` (Default)
    *   `gfx.BlendMode.SRC_IN`, `SRC_OUT`, `SRC_ATOP`
    *   `gfx.BlendMode.DST_OVER`, `DST_IN`, `DST_OUT`, `DST_ATOP`
    *   `gfx.BlendMode.XOR`, `ADD` (Linear Dodge)
    *   `gfx.BlendMode.MULTIPLY`, `SCREEN`, `OVERLAY`, `DARKEN`, `LIGHTEN`
*   `isolate` (bool): If true, the group is an "isolated group" (SVG term). Blending within the group does not interact with the background until the group ends.
*   `filter` (Filter): An optional filter effect (see 3.13).

```c
// Draw a semi-transparent ghost character
scene.groupBegin({ opacity=0.5, blendMode=gfx.BlendMode.SCREEN });
    scene.drawPath(bodyPath, bodyStyle);
    scene.drawPath(eyesPath, eyesStyle);
scene.groupEnd();
```

## 3.12 Masking

Masking uses the pixel values of one drawing to control the visibility of another.

1.  `scene.maskBegin(type)`: Starts recording the mask.
2.  ... draw shapes that define the mask ...
3.  `scene.maskEnd()`: Stops recording. The mask is now active.
4.  ... draw content to be masked ...
5.  `scene.restore()`: (Required) Pops the mask state (masks are part of the state stack).

**Mask Types:**
*   `gfx.MaskType.ALPHA`: The alpha channel of the mask defines opacity.
*   `gfx.MaskType.LUMINANCE`: The brightness of the mask defines opacity (white = visible, black = transparent).

## 3.13 Filters

Filters are powerful post-processing effects applied to Groups. The API strictly follows the **SVG Filter Effects** specification.
A `Filter` object is a chain of primitives. Input images are identified by constants:
*   `gfx.FilterInput.SOURCE_GRAPHIC`: The rasterized content of the group.
*   `gfx.FilterInput.SOURCE_ALPHA`: The alpha channel of the group.
*   `gfx.FilterInput.PREVIOUS_RESULT`: The output of the previous filter stage.

To create a filter: `local f = gfx.Filter();`
Chain methods to add primitives.

### Supported Primitives

#### `gaussianBlur`
Performs a Gaussian blur on the input image.
*   `input`: Source (default: `PREVIOUS_RESULT`).
*   `stdDevX`, `stdDevY`: The standard deviation for the blur operation. If `stdDevY` is omitted, it equals `stdDevX`.

#### `offset`
Translates the image.
*   `input`: Source.
*   `dx`, `dy`: Distance to offset.

#### `colorMatrix`
Matrix transformation of RGBA values.
*   `input`: Source.
*   `type`:
    *   `"matrix"`: A 4x5 matrix (20 floats).
    *   `"saturate"`: Single value 0..1 (grayscale) to >1 (supersaturated).
    *   `"hueRotate"`: Angle in degrees.
    *   `"luminanceToAlpha"`: Converts brightness to alpha channel.

#### `composite`
Combines two input images pixel-wise.
*   `in1`, `in2`: The two source inputs.
*   `operator`: `OVER`, `IN`, `OUT`, `ATOP`, `XOR`, `ARITHMETIC`.
*   `k1, k2, k3, k4`: Coefficients for `ARITHMETIC` mode ($result = k1 \cdot i1 \cdot i2 + k2 \cdot i1 + k3 \cdot i2 + k4$).

#### `displacementMap`
Uses the pixel values from `in2` to spatially displace `in1`.
*   `in1`: The image to displace.
*   `in2`: The map image.
*   `scale`: Displacement magnitude.
*   `xChannelSelector`, `yChannelSelector`: Which channel (`R`,`G`,`B`,`A`) to use from the map.

#### `turbulence`
Generates Perlin or Simplex noise. Useful for clouds, marble, or dissolve effects.
*   `baseFrequencyX`, `baseFrequencyY`: Noise granularity.
*   `numOctaves`: Detail level.
*   `seed`: RNG seed.
*   `type`: `"fractalNoise"` or `"turbulence"`.

#### `morphology`
Erodes (thins) or dilates (fattens) the image.
*   `operator`: `"erode"` or `"dilate"`.
*   `radiusX`, `radiusY`: Kernel size.

#### `dropShadow` (Convenience)
A shorthand that combines blur, offset, and composite.
*   `dx`, `dy`: Offset.
*   `stdDev`: Blur amount.
*   `color`: Shadow colour.

### Example: A "Neon Glow" Filter
This filter creates a glow by blurring the source, colouring it, and compositing it behind the original.

```c
local neonFilter = gfx.Filter()
    // 1. Blur the source alpha
    .gaussianBlur({
        input=gfx.FilterInput.SOURCE_ALPHA,
        stdDevX=4, stdDevY=4
    })
    // 2. Turn the blurred alpha into a solid colour (e.g., Green)
    .flood({
        color=Color(0, 1, 0, 1)
    })
    // 3. Composite the flood color *into* the blur shape
    .composite({
        operator=gfx.CompositeOp.IN,
        in2=gfx.FilterInput.PREVIOUS_RESULT // The blur result
    })
    // 4. Merge the original drawing on top of the glow
    .merge({
        inputs=[gfx.FilterInput.PREVIOUS_RESULT, gfx.FilterInput.SOURCE_GRAPHIC]
    });

// Usage
scene.groupBegin({ filter=neonFilter });
    scene.drawText("NEON", 100, 100, myStyle);
scene.groupEnd();
```

## 3.14 Hit Testing

Since the engine retains geometry data, it provides high-performance hit testing against paths without requiring manual math.

| Method | Description |
| :--- | :--- |
| `gfx.hitTestFill(path, x, y, [rule])` | Returns `true` if point `(x,y)` is inside the filled area of `path`. |
| `gfx.hitTestStroke(path, x, y, strokeWidth)` | Returns `true` if point `(x,y)` is touching the stroke of `path`. |

*   **Note:** Coordinates `(x,y)` must be in the same space as the path. If you drew the path with a transform, you must transform the mouse coordinates by the inverse matrix before testing.

```c
// Example: Checking if mouse clicked a shape
local mouse = input.mouse();
// Inverse transform logic would go here if the shape was transformed
if (gfx.hitTestFill(myButtonPath, mouse.x, mouse.y)) {
    // Handle click
}
```