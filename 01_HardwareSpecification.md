# 1. Hardware and System Architecture

This chapter defines the specifications of the virtual machine on which all cartridges execute. 

The console is designed as a **Retained Mode** system. Unlike traditional fantasy consoles that expose a direct framebuffer for pixel manipulation, this architecture exposes a high-performance scene graph. The hardware budget is therefore defined not just by raw bytes, but by the complexity of the scene (number of nodes, paths, and transforms) that the GPU can resolve per frame.

## 1.1 Video Modes

The console supports two fixed high-fidelity resolutions. A cartridge **MUST** declare its target mode in the manifest (see Chapter 10) and **MUST NOT** change it at runtime. All coordinates passed to drawing functions are expressed in the native pixels of the selected mode.

| Mode Name | Resolution | Aspect Ratio | Colour Depth | Notes |
| :--- | :--- | :--- | :--- | :--- |
| **MODE_4_3** | **800 × 600** | 4:3 | 32‑bit RGBA | Classic SVGA feel; ideal for retro-PC styled interfaces and games. |
| **MODE_16_9** | **960 × 540** | 16:9 | 32‑bit RGBA | qHD resolution; perfect integer scaling (×2) to Full HD (1920×1080). |

### Presentation & Scaling
When presenting to the host display, the runtime **SHOULD** use integer scaling (nearest-neighbour or sharp bilinear) to preserve crisp edges. The runtime **MUST** letter-box or pillar-box the content to maintain the strictly defined aspect ratio. Non‑uniform stretching is strictly prohibited to ensure artistic integrity.

## 1.2 Colour Pipeline

To achieve professional-grade blending and filtering, the rendering pipeline operates entirely in **Linear Colour Space** using **Premultiplied Alpha**. This is a significant departure from typical retro consoles that operate in sRGB/Gamma space.

### The Pipeline Steps
1.  **Input:** When you define a colour in scripts (e.g., `Color(1.0, 0.5, 0.0, 0.5)`), the runtime interprets these as **sRGB** values.
2.  **Conversion:** The values are immediately converted to **Linear Space**.
3.  **Premultiplication:** The RGB components are multiplied by the Alpha component.
    *   *Example:* sRGB Red at 50% opacity is defined as `(1.0, 0.0, 0.0, 0.5)`. The pipeline converts this to linear `(1.0, 0.0, 0.0, 0.5)` and then premultiplies to store `(0.5, 0.0, 0.0, 0.5)`.
4.  **Compositing:** All blending, gradient interpolation, and filter effects (blur, bloom) happen in this linear, premultiplied state. This ensures physically correct light transport and avoids "dark halos" around semi-transparent objects.
5.  **Output:** At the very end of the frame, the final framebuffer is converted back to **sRGB** (Gamma 2.2) for display on the monitor.

**Developer Requirement:** Scripts **SHOULD** supply premultiplied values if manually manipulating raw pixel buffers, but standard `Color()` constructors handle the heavy lifting for you automatically.

## 1.3 Determinism and Timing

Determinism is the bedrock of the console, enabling features like instant replays, rollback networking, and shared high-score verification.

### Profiles
The console defines two **determinism profiles**:

*   **Strict Deterministic (Default):**
    *   Frame updates (`update`) and rendering (`draw`) **MUST** produce identical results given the same input stream and random seed.
    *   **Prohibited:** `time.now()`, OS timers, unseeded RNG, arbitrary filesystem queries, or threading.
    *   **Enforced:** The runtime feeds a fixed delta time (`dt`) to `update()`. For example, at 60Hz, `dt` is always exactly `0.016666...`.
    *   *Use case:* Gameplay logic, physics simulations.

*   **Best Effort Deterministic:**
    *   Minor non-determinism is tolerated (e.g., floating-point variances across different CPU architectures).
    *   **Allowed:** `time.now()` for real-time clock displays, reading external configs.
    *   *Use case:* Development tools, level editors, non-competitive interactive art.

The manifest field `determinism` selects the profile. Scripts **MUST** treat `dt` passed to `update()` as authoritative.

## 1.4 Memory and Performance Budgets

Because the console uses a **Retained Mode** renderer, performance limits are defined by *scene complexity* rather than just CPU cycles. Exceeding these limits acts as a hardware constraint and will trigger a fatal error or dropping of draw commands.

### Storage & Heap
*   **Cartridge Size:** ≤ 32 MiB (Code + Assets). *Increased from standard retro specs to accommodate higher-res assets.*
*   **Script Source:** ≤ 1 MiB (Uncompressed text).
*   **Runtime Heap:** ≤ 64 MiB (Squirrel objects, tables, arrays).
*   **String Pool:** ≤ 16 MiB.
*   **Persistent Save Data:** ≤ 4 MiB in `save/`.

### Scene Graph & Rendering Budgets (Per Frame)
The "Scene" is the collection of all 2D paths, 3D meshes, and text objects active in the current frame.

*   **Render Commands:** ≤ 200,000 instructions per frame.
*   **Live Scene Nodes:** ≤ 50,000 active nodes (2D or 3D) in the hierarchy.
*   **Path Complexity:** ≤ 100,000 tessellated segments.
*   **Clip Stack Depth:** ≤ 64 levels of nested masking/clipping.
*   **Offscreen Surfaces:** ≤ 16 surfaces of max size 2048 × 2048.
*   **3D Geometry:** ≤ 500,000 triangles per frame.

### Performance Cost Categories
To help developers optimize, the integrated profiler breaks down frame time into:

| Category | Description | Optimization Strategy |
| :--- | :--- | :--- |
| `scene_update` | Time spent calculating local transforms (TRS) and hierarchy propagation. | Reduce hierarchy depth; flatten static groups. |
| `tessellation` | Converting vector paths/strokes into GPU geometry. | Reuse immutable `Path` objects; avoid rebuilding paths every frame. |
| `raster` | GPU fill rate usage (transparency, overdraw). | Minimize large semi-transparent overlaps; use simple bounding boxes for culling. |
| `filters` | Post-processing effects (Blur, Bloom). | Reduce filter kernel sizes or number of passes. |

## 1.5 Audio Subsystem

Audio follows the deterministic philosophy. The system guarantees that audio synthesis happens in lockstep with the game logic.

*   **Sample Rate:** 48 kHz Stereo.
*   **Buffer Strategy:** Double-buffered, 512 frames per callback (approx 10.6ms latency).
*   **Capabilities:**
    *   1 × Active Tracker Module (XM, MOD, S3M, IT).
    *   32 × Polyphonic SFX Channels (WAV, OGG).
    *   Global Master Mixer with deterministic limiting/ducking.

## 1.6 Input Architecture

The system abstracts physical hardware into virtual devices to ensure portability.

*   **Controllers:** Up to 4 simultaneous virtual Gamepads.
*   **Layout:** "Retropad" style + Dual Analog Sticks + Analog Triggers.
*   **Polling:** Input is sampled once per frame, immediately before `update()`. This ensures that input state remains constant throughout the entire logic tick (no race conditions).
*   **Mouse/Keyboard:** Fully supported as first-class input devices, with coordinate mapping automatically adjusted to the 800x600 or 960x540 viewport.

## 1.7 Filesystem Sandbox

Each cartridge operates in a strict sandbox:
1.  **Read-Only Assets:** The `assets/` folder is immutable at runtime.
2.  **Private Save:** The `save/` folder is the **only** writable location.
3.  **Isolation:** No access to the host OS filesystem, network sockets, or inter-process communication is permitted, ensuring cartridge security and portability.