# 9. Time, Filesystem & Debugging

This chapter covers the essential system utilities that glue your application together: managing the flow of time, persisting player data, generating random numbers, and debugging your code.

## 9.1 Time Management

The console maintains two distinct concepts of time. Distinguishing between them is critical for maintaining the **Deterministic** guarantee.

### Simulation Time (Deterministic)
Use these functions for game logic, physics, and replays. They advance in fixed steps relative to the `update()` loop.

| Function | Returns | Description |
| :--- | :--- | :--- |
| `time.dt()` | `float` | The delta time in seconds for the current update tick (e.g., `0.01666...` at 60Hz). Constant during a frame. |
| `time.frame()` | `int` | The number of frames processed since `init()`. |
| `time.ticks()` | `int` | The total simulation time in microseconds. |
| `time.fps()` | `float` | The current *simulation* framerate (game speed). |

### Wall-Clock Time (Non-Deterministic)
Use these functions **only** for UI (e.g., displaying a clock), debugging, or performance profiling. **Never** use them to drive gameplay mechanics.

| Function | Returns | Description |
| :--- | :--- | :--- |
| `time.now()` | `double` | High-precision seconds since app start. |
| `time.date()` | `Table` | System date: `{ year, month, day, hour, min, sec }`. |
| `time.realFps()` | `float` | The actual rendering framerate (V-Sync). |

## 9.2 Random Number Generation (RNG)

The `math` namespace provides a high-quality, seeded Pseudo-Random Number Generator (PRNG). To ensure that a specific seed always generates the exact same level layout or loot drop, you **MUST** use these functions instead of any host-specific randomness.

```c
// Initialize the RNG.
// In 'Strict' determinism mode, you MUST call this in init().
math.seed(123456789);

// Usage
local f = math.rand();       // Float [0.0, 1.0)
local i = math.range(1, 10); // Integer [1, 10] (Inclusive)
local c = math.coin();       // Boolean (50/50)
```

## 9.3 Filesystem Sandbox

For security and portability, the filesystem is strictly sandboxed. Paths use forward slashes `/`.

### 1. Assets (Read-Only)
The `assets/` folder contains your cartridge's immutable data (images, sounds, scripts).
*   **Access:** You generally don't read these files manually; you pass paths to loaders like `gfx.Image.load()`.
*   **Manual Read:** `fs.readText("assets/data.json")` is allowed if you need to parse custom data.

### 2. Save Data (Read/Write)
The `save/` folder is your private persistent storage.
*   **Quota:** 4 MiB max.
*   **Isolation:** You cannot see files from other cartridges.

### File Operations
All file I/O is **synchronous** to keep the API simple and deterministic.

| Function | Description |
| :--- | :--- |
| `fs.writeText(path, str)` | Writes a string to a file in `save/`. Overwrites if exists. |
| `fs.readText(path)` | Returns the file content as a string. Throws if missing. |
| `fs.exists(path)` | Returns `true` if the file exists. |
| `fs.delete(path)` | Deletes a file in `save/`. |
| `fs.list(dir)` | Returns an array of filenames in the directory. |
| `fs.jsonRead(path)` | Helper: Reads text and parses JSON into a table. |
| `fs.jsonWrite(path, tbl)` | Helper: Serializes a table to JSON and writes it. |

```c
// Example: Saving High Score
function saveScore(score) {
    local data = { highscore = score, timestamp = time.date() };
    fs.jsonWrite("save/score.json", data);
}
```

## 9.4 Debugging Tools

The console includes a suite of tools that are stripped or disabled in Release builds, allowing you to instrument your code heavily during development.

### Logging
The `log` namespace prints to the Editor's console panel.
*   `log.info(msg)`: Standard white text.
*   `log.warn(msg)`: Yellow text, adds a warning counter.
*   `log.error(msg)`: Red text, pops up the console.

### Visual Watch
Instead of spamming the log with `print(player.x)`, use `dbg.watch`. This adds a live-updating row to the Debug Inspector panel.

```c
function update(dt) {
    player.x += dt * 10;
    // Shows "Player X: 10.5" in the side panel
    dbg.watch("Player X", player.x);
    dbg.watch("State", currentStateName);
}
```

### Assertions
Enforce invariants in your code. If an assertion fails in Editor Mode, execution pauses at that line. In Player Mode, it is ignored.

```c
dbg.assert(health >= 0, "Health cannot be negative!");
```

### Profiling
Measure how long a block of code takes to execute. The result appears in the Profiler Flame Graph.

```c
dbg.profileBegin("Pathfinding");
    calculateRoute();
dbg.profileEnd("Pathfinding");
```

### Visual Debug Drawing
Draw shapes on top of everything for debugging physics or AI. These are automatically cleared next frame.

*   `dbg.drawLine(x1, y1, x2, y2, color)`
*   `dbg.drawRect(x, y, w, h, color)`
*   `dbg.drawCircle(x, y, r, color)`

## 9.5 Example: System Manager

A robust pattern for managing game settings using these APIs.

```c
local settings = {
    soundVolume = 1.0,
    firstRun = true
};

function init() {
    // 1. Load Settings
    if (fs.exists("save/config.json")) {
        local diskData = fs.jsonRead("save/config.json");
        // Merge disk data into settings
        foreach(k,v in diskData) settings[k] = v;
        log.info("Config loaded.");
    } else {
        log.warn("No config found, using defaults.");
    }

    // 2. Apply Settings
    audio.setMasterVolume(settings.soundVolume);
    
    // 3. Debug Setup
    if (sys.isDebug()) {
        log.info("Running in Debug Mode");
        // Seed with a fixed number for reproducible bugs
        math.seed(12345);
    } else {
        // Seed with system time for players
        math.seed(sys.timestamp());
    }
}

function onExit() {
    // Auto-save on quit
    fs.jsonWrite("save/config.json", settings);
}
```