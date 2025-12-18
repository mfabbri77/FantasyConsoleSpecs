# 2. Lifecycle of a Cartridge

A cartridge is a Squirrel script loaded and executed by the runtime. Unlike simple game loops that just run code linearly, the Fantasy Console enforces a structured lifecycle to guarantee determinism, separate simulation from rendering, and enable powerful debugging tools like time-travel and hot-swapping.

This chapter details the entry points, the retained-mode rendering loop, and event handling strategies.

## 2.1 The Core Loop

The runtime drives the execution flow. You do not write a `while(true)` loop; instead, you define specific functions that the runtime calls at the appropriate times.

| Function | Signature | Frequency | Role |
| :--- | :--- | :--- | :--- |
| **`init`** | `init()` | Once | Called immediately after the cartridge is loaded and the VM is spun up. Use this to initialise global state, load assets, and seed the deterministic RNG. |
| **`update`** | `update(dt)` | Fixed (e.g. 60Hz) | The **Simulation Step**. All game logic, physics integration, and input handling **MUST** happen here. `dt` is the fixed delta time in seconds. |
| **`draw`** | `draw()` | Variable (V-Sync) | The **Presentation Step**. This function is responsible for assembling the scene graph for the current frame. **Do not** mutate gameplay state here. |

### The Separation of Concerns
Because the console uses a **Retained Mode** renderer, the distinction between `update` and `draw` is stricter than in traditional systems:

*   **In `update(dt)`**: You modify the *mathematical model* of your world (e.g., `player.x += velocity`). You do not touch graphics objects here.
*   **In `draw()`**: You map your mathematical model to visual nodes (e.g., `playerNode.setPosition(player.x, player.y)`). You are constructing a "frame packet" that the GPU will render *after* this function returns.

## 2.2 Optional Event Callbacks

While polling input in `update()` is preferred for movement and continuous actions, event callbacks are superior for "trigger" actions (like typing or menu clicks) and system notifications.

| Callback | Signature | Description |
| :--- | :--- | :--- |
| `onKey` | `onKey(code, isDown)` | Triggered when a physical keyboard key is pressed or released. `code` is a hardware-agnostic scancode. |
| `onMouse` | `onMouse(x, y, buttons, wheel)` | Triggered on mouse movement, click, or scroll. Coordinates are in the current Video Mode resolution. |
| `onPad` | `onPad(player, button, isDown)` | Triggered when a gamepad button (digital or analog threshold) changes state. |
| `onPause` | `onPause(isPaused)` | Called when the runtime gains or loses focus (e.g., user alt-tabs or opens the system overlay). Pause your audio/logic if needed. |
| `onError` | `onError(message, stacktrace)` | Called when an uncaught exception occurs. Returning `true` suppresses the default "Blue Screen of Death", allowing for custom error screens. |

## 2.3 Determinism and The Loop

To maintain the **Strict Determinism** profile (see Chapter 1):

1.  **Logic Isolation:** Never put game logic in `draw()`. The runtime may call `draw()` multiple times per logic tick (interpolation) or skip it entirely (frame skipping) depending on host performance. Only `update()` is guaranteed to run exactly once per simulation step.
2.  **Clock Hygiene:**
    *   **DO NOT** call `time.now()` (system clock) inside `update()` or `draw()`.
    *   **DO** use `time.dt()`, `time.frame()`, and `time.ticks()` which are derived from the simulation step count.
3.  **RNG Safety:** Seed the RNG in `init()` using `math.seed()`. Never use Lua/Squirrel's iteration order on tables (`foreach`) for game logic, as the hashing order is implementation-defined. Sort keys before iterating if order matters.

## 2.4 Hot Reloading (Live Coding)

The console supports aggressive Hot Reloading, allowing you to edit code while the game runs. This is controlled via the Editor interface.

### Soft Reload (Inject)
*   **Action:** The runtime recompiles the script and replaces the function definitions (e.g., `update`, `draw`) in memory.
*   **State:** The global heap (variables, tables, active scene nodes) is **preserved**.
*   **Use Case:** Tweaking jump physics, adjusting colours, fixing a bug in `draw()` logic.
*   **Risk:** If you change the structure of a class or a global variable's type, the old data in memory might clash with the new code.

### Hard Reload (Reset)
*   **Action:** The VM is torn down, memory is cleared, script is recompiled, and `init()` is called again.
*   **State:** Everything is reset to zero.
*   **Use Case:** Changing `init()` logic, adding new assets, or when the state has become corrupted by a Soft Reload.

## 2.5 Error Handling

When an error occurs:
1.  **Editor Mode:** The simulation pauses. The offending line is highlighted, and the stack trace appears in the console. You can fix the code and trigger a Soft Reload to resume execution from exactly where it crashed.
2.  **Player Mode:** The runtime calls `onError`. If unhandled, it stops the cartridge and displays a system error message.

## 2.6 Example: The Complete Lifecycle

This example demonstrates the separation of simulation (`state`) and presentation (`nodes`).

```c
// 1. GLOBAL STATE (The Model)
local state = {
    t = 0.0,
    pos = Vec2(400, 300), // Centre of 800x600
    color = Color(1, 1, 1, 1)
};

// 2. RESOURCES (The View)
local resources = {
    font = null,
    music = null
};

// 3. INIT (Setup)
function init() {
    // Seed RNG for determinism
    math.seed(1337);
    
    // Load immutable assets
    resources.font = gfx.Font.load("assets/interface.ttf");
    resources.music = audio.loadModule("assets/bgm.xm");
    
    audio.play(resources.music);
    log.info("Cartridge Initialised");
}

// 4. UPDATE (Simulation - Fixed Step)
function update(dt) {
    state.t += dt;
    
    // Simple harmonic motion logic
    state.pos.x = 400 + math.sin(state.t * 2) * 200;
    state.pos.y = 300 + math.cos(state.t * 3) * 100;
    
    // Input handling (Poll)
    if (input.btn(0, input.BTN_A)) {
        state.color = Color(1, 0, 0, 1); // Red when held
    } else {
        state.color = Color(1, 1, 1, 1); // White default
    }
}

// 5. DRAW (Presentation - Variable Step)
function draw() {
    // 1. Begin a new frame scene
    local scene = gfx.beginFrame();
    
    // 2. Clear background
    scene.clear(Color(0.1, 0.1, 0.15, 1));
    
    // 3. Draw shapes based on State
    // Note: We are creating a path description, not rasterising pixels yet.
    local p = gfx.Path();
    p.circle(state.pos.x, state.pos.y, 32);
    
    local style = gfx.Style()
        .fill(gfx.Paint.color(state.color))
        .stroke(gfx.Paint.color(0, 0, 0, 0.5))
        .strokeWidth(4);
        
    scene.drawPath(p, style);
    
    // 4. Draw UI
    local textStyle = gfx.TextStyle(resources.font, 20);
    scene.drawText("Time: " + state.t, 10, 580, textStyle);
    
    // 5. Submit the scene to the GPU
    gfx.drawScene(scene);
}