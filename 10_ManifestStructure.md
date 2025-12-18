# 10. Manifest and Packaging

A **Cartridge** is a self-contained unit of software and data. Physically, it is a directory (during development) or a compressed `.pak` archive (for distribution). The brain of the cartridge is the `manifest.ini` file, which tells the runtime how to configure the virtual hardware and where to find the entry point.

## 10.1 Cartridge Layout

A standard project follows this directory structure:

```text
my_game/
├── manifest.ini          # Metadata and System Config
├── main.nut              # Main entry point script
├── src/                  # Additional source files
│   ├── player.nut
│   └── utils.nut
├── assets/               # Read-only assets
│   ├── sprites/
│   ├── fonts/
│   └── audio/
└── save/                 # Created by runtime (ignored by version control)
```

## 10.2 The manifest.ini File

The manifest uses the INI format and is divided into logical sections.

### [cart] - System Configuration
This section defines the virtual hardware parameters.

| Key | Values | Description |
| :--- | :--- | :--- |
| `title` | String | The name of your game. |
| `author` | String | Your name or studio. |
| `version` | String | Semantic version (e.g., `1.0.2`). |
| **`mode`** | `LORES_4_3` \| `HIRES_4_3` \| `LORES_16_9` \| `HIRES_16_9` | Sets the resolution (400x300, 800x600, 480x270, or 960x540). |
| **`fps`** | `30` \| `60` \| `120` | Target simulation frequency for `update()`. |
| `entry` | Path | Path to the primary script (default: `main.nut`). |
| **`determinism`** | `strict` \| `best` | `strict` enforces bit-perfect timing and RNG. |

### [assets] - Asset Mapping
Instead of hardcoding paths like `"assets/images/hero_final_v2.png"` in your scripts, you can map them to symbolic names. These names become available in the global `ASSETS` table.

```ini
[assets]
hero_sprite = "assets/sprites/hero.png"
main_theme  = "assets/audio/music.xm"
ui_font     = "assets/fonts/inter.ttf"
```

In your script:
```c
local img = gfx.Image.load(ASSETS.hero_sprite);
```

### [window] - Host Presentation
Settings that affect how the host OS displays the console window.

*   `width`, `height`: Initial window size on PC.
*   `resizable`: `true` or `false`.
*   `fullscreen`: `true` or `false`.

## 10.3 Example Manifest

```ini
[cart]
title = "Neon Racer 2000"
author = "CyberLabs"
version = "1.0.0"
mode = "HIRES_16_9"
fps = 60
entry = "main.nut"
determinism = "strict"

[assets]
# Graphics
car_mesh = "assets/models/car.obj"
road_tex = "assets/textures/asphalt.png"

# Audio
engine_sfx = "assets/sfx/engine.wav"
bg_music   = "assets/music/synthwave.it"

# Data
levels = "assets/data/levels.json"

[window]
width = 1280
height = 720
resizable = true
```

## 10.4 The Packaging Process

### Development Mode
During development, the runtime points to the project folder. You can add, remove, or edit files, and use **Hot Reloading** to see changes instantly.

### Distribution (.pak)
When you are ready to ship, the Editor can "Export" the project into a `.pak` file.
1.  **Compression:** All assets and scripts are compressed using a high-ratio algorithm.
2.  **Stripping:** Debug symbols, comments, and `dbg.*` calls are stripped from the scripts to optimize performance and protect source code.
3.  **Virtual Filesystem:** The `.pak` file is a mountable VFS. To the host OS, it's a single file; to the console, it's a complete hierarchy.

## 10.5 Script Inclusion

To keep your code organized, you can include other `.nut` files. The path is always relative to the cartridge root.

```c
// In main.nut
require("src/utils.nut");
require("src/player.nut");

local p = Player(); // Defined in player.nut
```

## 10.6 Versioning and Compatibility

The Fantasy Console runtime is backwards compatible. The `manifest.ini` can optionally include an `api_version` key.
*   If your cartridge specifies `api_version = 1`, and the user is running a version 2 runtime, the system will enable a **Legacy Compatibility Layer** to ensure the 2D/3D APIs behave exactly as they did in version 1.
*   This ensures that "fantasy" games remain playable for decades, regardless of how the underlying host technology evolves.

## 10.7 Security and Integrity

The `.pak` format includes a digital signature.
*   **Anti-Tamper:** If a user modifies an asset inside a signed `.pak`, the console will refuse to load it in "Strict" mode.
*   **Save Isolation:** Save data is never stored inside the `.pak`. It is stored in a system-defined user folder (e.g., `%APPDATA%/FantasyConsole/saves/[TitleID]/`). This ensures that updating a game does not delete the player's progress.