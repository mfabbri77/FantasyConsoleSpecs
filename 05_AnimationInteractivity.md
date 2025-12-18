# 5. Animation & Interactivity API

In a Retained Mode engine, animation is not achieved by calculating new coordinates every frame in a loop. Instead, you define **Animations**—declarative descriptions of how values change over time—and apply them to your scene objects.

The console features a robust animation system capable of interpolating numbers, vectors, colours, and even morphing paths. It also includes a native loader for **Lottie** (JSON) files, allowing you to import cinematic vector animations directly.

## 5.1 Concepts

*   **Track:** A timeline for a single property (e.g., "Rotation"). It contains a sequence of Keyframes.
*   **Keyframe:** A specific value at a specific time, combined with easing information (how to approach/leave this value).
*   **Animation:** A collection of Tracks targeting a specific object or group of objects.
*   **Timeline:** A master sequencer that controls multiple Animations, allowing for complex choreography.

## 5.2 Property Handles

To animate a value, the system needs a reference to it. Since Squirrel is a dynamic language, the console provides a **Property Handle** API to create type-safe references to nested fields.

```c
// Create a handle targeting the 'rotation' of a Transform
local rotHandle = anim.prop(myNode).transform.rotation;

// Create a handle targeting the 'alpha' of a specific colour in a gradient
local colorHandle = anim.prop(myPaint).gradient.stops[0].color;
```

Valid targets for handles include:
*   `transform` (position, scale, rotation, skew)
*   `style` (opacity, stroke width)
*   `paint` (color, gradient stops)
*   `camera` (fov, position, target)
*   `light` (intensity, color, direction)

## 5.3 Creating Animations

An animation is created via `gfx.anim.create()`. You then add tracks to it.

```c
local jumpAnim = gfx.anim.create("jump");
jumpAnim.duration(1.0); // Total duration in seconds

// Define the track
jumpAnim.track(anim.prop(playerNode).transform.position.y, [
    { time=0.0, value=0 },
    { time=0.5, value=-100, easing=gfx.Easing.easeOutQuad },
    { time=1.0, value=0,    easing=gfx.Easing.easeInQuad }
]);

// Play it
jumpAnim.play({ loop=false });
```

### Keyframe Structure
Each keyframe is a table:
*   `time`: Seconds from the start of the animation.
*   `value`: The value to reach. Must match the type of the property (float, Vec2, Color).
*   `easing`: (Optional) The curve used to interpolate *towards* the next keyframe.

## 5.4 Easing Functions

The `gfx.Easing` module provides standard interpolation curves.

| Function | Description |
| :--- | :--- |
| `linear` | Constant speed. |
| `easeInQuad`, `easeOutQuad`, `easeInOutQuad` | Quadratic curves (starts/ends slow). |
| `easeInCubic`, `easeOutCubic`, ... | Cubic curves (steeper acceleration). |
| `elastic`, `bounce`, `back` | Overshooting effects for "juicy" motion. |
| `step` | Instant jump to the value. |
| `bezier(x1, y1, x2, y2)` | Custom cubic Bézier definition (compatible with CSS/Lottie). |

## 5.5 Timelines

A **Timeline** allows you to sequence animations relative to each other.

```c
local introSequence = gfx.anim.timeline();

// Add animations. 'offset' is relative to the previous item's end.
introSequence.add(fadeInAnim);               // Starts at t=0
introSequence.add(logoSpinAnim, -0.5);       // Starts 0.5s before fade finishes
introSequence.add(textSlideAnim, 0.2);       // Starts 0.2s after logo finishes

introSequence.play();
```

## 5.6 Lottie Integration

The console includes a native parser and renderer for the **Lottie** format. This allows artists to work in tools like Adobe After Effects or Blender, export to JSON, and run the result with pixel-perfect vector fidelity.

### Loading Lottie
```c
// Load the asset
local heroAnim = gfx.anim.loadLottie("assets/hero_run.json");

// Configure
heroAnim.loop = true;
heroAnim.speed = 1.0;

// Render
// Lottie animations are drawn like any other drawable
scene.drawLottie(heroAnim, x, y, width, height);
```

### Controlling Lottie
You can manipulate playback programmatically:
*   `heroAnim.play()` / `heroAnim.pause()`
*   `heroAnim.seek(frame)`
*   `heroAnim.setSegment(startFrame, endFrame)`: Loops only a specific section (e.g., "Run Cycle" frames 0-30).

**Limitations:**
*   Expressions (Javascript inside JSON) are **not** supported.
*   Layer effects (Gaussian Blur within Lottie) are mapped to console Filters where possible, but heavy usage impacts performance.

## 5.7 Animation Events & Hooks

Animations are not just visual; they can drive game logic. You can attach callbacks to lifecycle events.

```c
// Trigger a sound when the jump reaches the peak
jumpAnim.at(0.5, function() {
    audio.playSfx(peakSound);
});

// Logic when animation completes
jumpAnim.onComplete(function() {
    state.isJumping = false;
});

// Logic on every frame of the animation
jumpAnim.onUpdate(function(progress) {
    // 'progress' is 0.0 to 1.0
});
```

## 5.8 Example: Interactive Button

This example creates a UI button that scales up when hovered and bounces when clicked, using the animation system.

```c
local btnNode = {
    scale = Vec2(1, 1),
    rotation = 0
};
local hoverAnim;
local clickAnim;

function init() {
    // 1. Define Hover Animation (Pulse)
    hoverAnim = gfx.anim.create("hover");
    hoverAnim.duration(1.0);
    hoverAnim.track(anim.prop(btnNode).scale, [
        { time=0.0, value=Vec2(1.0, 1.0) },
        { time=0.5, value=Vec2(1.1, 1.1), easing=gfx.Easing.easeInOutSine },
        { time=1.0, value=Vec2(1.0, 1.0), easing=gfx.Easing.easeInOutSine }
    ]);
    
    // 2. Define Click Animation (Bounce)
    clickAnim = gfx.anim.create("click");
    clickAnim.duration(0.3);
    clickAnim.track(anim.prop(btnNode).scale, [
        { time=0.0, value=Vec2(0.9, 0.9) },
        { time=0.3, value=Vec2(1.0, 1.0), easing=gfx.Easing.elasticOut }
    ]);
}

function update(dt) {
    local m = input.mouse();
    local isHover = (m.x > 300 && m.x < 500 && m.y > 250 && m.y < 350);
    
    if (isHover) {
        if (!hoverAnim.isPlaying()) hoverAnim.play({ loop=true });
        
        if (input.btnp(0, input.BTN_MOUSE_LEFT)) {
            clickAnim.play({ force=true }); // Force restart if already playing
            audio.playSfx(clickSound);
        }
    } else {
        hoverAnim.stop();
        btnNode.scale = Vec2(1, 1); // Reset
    }
}

function draw() {
    local scene = gfx.beginFrame();
    
    scene.save();
    // Apply the animated scale
    scene.transform(gfx.Transform.translate(400, 300)); // Pivot center
    scene.transform(gfx.Transform.scale(btnNode.scale.x, btnNode.scale.y));
    scene.transform(gfx.Transform.translate(-400, -300));
    
    // Draw Button
    local p = gfx.Path();
    p.rect(300, 250, 200, 100, 10, 10);
    scene.drawPath(p, gfx.Style().fill(gfx.Paint.color(0.2, 0.6, 1, 1)));
    
    scene.restore();
    gfx.drawScene(scene);
}
```