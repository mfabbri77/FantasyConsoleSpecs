# 4. Text Rendering API

Text in the Fantasy Console is not a second-class citizen. It is rendered using the same high-precision vector pipeline as shapes, supporting sub-pixel positioning, complex shaping (ligatures, kerning), and bidirectional layout (RTL).

To render text, you need two components:
1.  **`gfx.TextStyle`**: Defines the *typography* (Font, Size, Alignment, Spacing).
2.  **`gfx.Style`**: Defines the *appearance* (Fill Paint, Stroke, Render Order).

## 4.1 Loading Fonts

Fonts are immutable resources loaded from the `assets/` directory. The engine supports TrueType (`.ttf`), OpenType (`.otf`), and WOFF2 (`.woff2`) formats.

```c
// Load a font face. This operation is synchronous but cached.
local mainFont = gfx.Font.load("assets/fonts/Inter-Regular.ttf");
local boldFont = gfx.Font.load("assets/fonts/Inter-Bold.ttf");
```

### Fallbacks & Glyphs
If a font lacks a glyph for a specific character (e.g., an Emoji or a Kanji), the engine attempts to resolve it using a system-defined fallback chain. If no glyph is found, a "tofu" (box) symbol is drawn.

## 4.2 TextStyle (Typography)

The `TextStyle` object is an immutable descriptor of how text is laid out. It does **not** contain colour information. Use the builder pattern to construct it.

```c
local ts = gfx.TextStyle()
    .font(mainFont)             // The gfx.Font object
    .size(24.0)                 // Size in user units (pixels)
    .align(gfx.TextAlign.LEFT)  // LEFT, CENTER, RIGHT, JUSTIFY
    .baseline(gfx.TextBaseline.ALPHABETIC) // TOP, MIDDLE, ALPHABETIC, BOTTOM
    .letterSpacing(0.0)         // Extra space between characters
    .lineHeight(1.2);           // Multiplier of font size for multiline text
```

| Property | Description |
| :--- | :--- |
| **font** | The primary font face. Defaults to a built-in sans-serif if omitted. |
| **size** | The em-size of the font. |
| **align** | Horizontal alignment relative to the anchor point `(x,y)`. |
| **baseline** | Vertical alignment relative to the anchor point. `ALPHABETIC` is standard for Latin/European scripts. |
| **direction** | Explicit `LTR` or `RTL`. If `AUTO` (default), direction is inferred from the text content per paragraph. |

## 4.3 Drawing Text

To draw text, submit a command to the current scene. You must provide both the `TextStyle` (for layout) and the `Style` (for colour).

### Basic Drawing
```c
local redFill = gfx.Style().fill(gfx.Paint.color(1, 0, 0, 1));

// Draws "Hello" at (100, 100) using the defined typography and red fill
scene.drawText("Hello World", 100, 100, ts, redFill);
```

### Outlined Text
Since text uses the standard `gfx.Style`, you can stroke it just like a path.
```c
local outlineStyle = gfx.Style()
    .fill(gfx.Paint.color(1, 1, 1, 1))      // White Fill
    .stroke(gfx.Paint.color(0, 0, 0, 1))    // Black Stroke
    .strokeStyle(gfx.StrokeStyle().width(2))
    .order(gfx.PaintOrder.STROKE_THEN_FILL); // Stroke behind fill

scene.drawText("Title Screen", 400, 300, titleTs, outlineStyle);
```

### Text on Path
You can layout text along any `gfx.Path`. The text will follow the curvature, rotating characters to match the tangent.

```c
scene.drawTextOnPath(text, path, textStyle, style, offset);
```
*   **offset**: Distance along the path to start the text.

## 4.4 Measuring Text

For UI layout, you often need to know exactly how much space text will occupy before rendering it.

```c
local metrics = gfx.measureText("Score: 100", textStyle);
```

The returned `metrics` object contains:
*   `width`: The total horizontal advance.
*   `actualBoundingBox`: A `Rect` containing the tight bounds of the visible pixels.
*   `fontBoundingBoxAscent`: Distance from baseline to the top of the font's design box.
*   `fontBoundingBoxDescent`: Distance from baseline to the bottom.
*   `lines`: An array of line metrics (if the text contains newlines).

## 4.5 Advanced Shaping

The console uses a professional-grade shaping engine (similar to HarfBuzz). This means:

1.  **Ligatures:** Sequences like `fi`, `fl`, `==>` (if supported by the font) are automatically combined into single glyphs.
2.  **Kerning:** Character spacing is adjusted based on the font's kerning tables (e.g., `AV` is tighter than `A V`).
3.  **Bi-directional Text:** Mixing English (LTR) and Arabic/Hebrew (RTL) in the same string works correctly. The `align` property adapts: `LEFT` alignment becomes "Start" alignment (Right in RTL).

## 4.6 Example: Typewriter Effect

This example shows how to combine substring manipulation with measurement to create a typewriter effect that wraps correctly.

```c
local fullText = "Welcome to the Fantasy Console. This text is being typed out in real-time.";
local displayLen = 0;
local font;
local ts;
local style;

function init() {
    font = gfx.Font.load("assets/mono.ttf");
    ts = gfx.TextStyle().font(font).size(20).lineHeight(1.5);
    style = gfx.Style().fill(gfx.Paint.color(0, 1, 0, 1)); // Retro Green
}

function update(dt) {
    // Type 10 characters per second
    displayLen += dt * 10;
    if (displayLen > fullText.len()) displayLen = fullText.len();
}

function draw() {
    local scene = gfx.beginFrame();
    scene.clear(Color(0, 0, 0, 1));

    // Slice the string
    local currentStr = fullText.slice(0, displayLen.tointeger());
    
    // Draw wrapped text inside a box
    // Note: drawText automatically handles newlines in the string.
    // For automatic wrapping, we would use a layout helper (future API).
    scene.drawText(currentStr, 50, 50, ts, style);
    
    // Draw a blinking cursor at the end
    local m = gfx.measureText(currentStr, ts);
    if ((time.ticks() / 500000) % 2 == 0) { // Blink every 0.5s
        local cursorX = 50 + m.width; // Simplified (assumes single line for this snippet)
        local cursorP = gfx.Path();
        cursorP.rect(cursorX, 50 - m.fontBoundingBoxAscent, 10, 24);
        scene.drawPath(cursorP, style);
    }
    
    gfx.drawScene(scene);
}
```