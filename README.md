# Fantasy Console — Technical Specifications

> **A next-generation fantasy console built on a Retained Mode architecture.**

## Overview

This repository contains the **official technical specifications** for the Fantasy Console — a paradigm shift bridging nostalgic retro computing with modern GPU-accelerated rendering. Unlike traditional fantasy consoles that rely on direct framebuffer manipulation, this system operates entirely on a **Retained Mode** architecture: you construct persistent scene graphs and declarative animations, and the runtime handles rasterization, occlusion, and compositing with mathematical precision.

Whether you're rendering 2D vector paths or complex 3D PBR meshes, you don't draw pixels directly — you orchestrate a visual symphony of nodes and resources.

## Core Features

| Feature | Description |
|:--------|:------------|
| **Video Modes** | 800×600 (4:3 SVGA) or 960×540 (16:9 qHD) — 32-bit RGBA, linear premultiplied alpha |
| **2D Engine** | Resolution-independent vector graphics: Bézier paths, gradients, patterns, SVG-style filters |
| **3D Engine** | Full PBR (Physically Based Rendering) pipeline with metallic-roughness workflow, glTF 2.0 compatible |
| **Text Rendering** | High-precision typography with shaping, kerning, ligatures, and bidirectional layout (RTL) |
| **Animation** | Declarative property animation with timelines, easing functions, and native Lottie support |
| **Audio** | Deterministic tracker modules (MOD/XM/S3M/IT) + 32-voice polyphonic SFX engine at 48kHz |
| **Input** | Up to 4 gamepads, mouse, and keyboard — unified polling API with deterministic sampling |
| **Scripting** | [Squirrel](http://www.squirrel-lang.org/) scripting language with hot-reload support |
| **Determinism** | Bit-perfect reproducibility for replays, rollback networking, and competitive scoring |

## Documentation Structure

| Chapter | File | Topic |
|:--------|:-----|:------|
| 0 | [00_Overview.md](00_Overview.md) | Introduction and philosophy |
| 1 | [01_HardwareSpecification.md](01_HardwareSpecification.md) | System architecture, video modes, memory budgets |
| 2 | [02_Lifecycle.md](02_Lifecycle.md) | Script lifecycle (`init`, `update`, `draw`), hot-reloading |
| 3 | [03a_Graphics2D_Core.md](03a_Graphics2D_Core.md) | 2D primitives: paths, paints, styles |
| 3 | [03b_Graphics2D_Advanced.md](03b_Graphics2D_Advanced.md) | Transforms, clipping, groups, filters |
| 4 | [04_TextAPI.md](04_TextAPI.md) | Typography, fonts, text layout |
| 5 | [05_AnimationInteractivity.md](05_AnimationInteractivity.md) | Animation system, timelines, Lottie |
| 6 | [06_Graphics3D.md](06_Graphics3D.md) | 3D scene graph, PBR materials, lights, cameras |
| 7 | [07_AudioAPI.md](07_AudioAPI.md) | Tracker music, SFX, mixer |
| 8 | [08_InputAPI.md](08_InputAPI.md) | Gamepads, mouse, keyboard, haptics |
| 9 | [09_TimeFilesystemDebug.md](09_TimeFilesystemDebug.md) | Time management, filesystem sandbox, debugging |
| 10 | [10_ManifestStructure.md](10_ManifestStructure.md) | Cartridge format, manifest configuration |
| 11 | [11_Examples.md](11_Examples.md) | Comprehensive code examples |

## Design Philosophy

1. **Retained Mode First** — Build the scene; the console draws it. Enables automatic batching, infinite zoom, and complex scene manipulations.

2. **Determinism** — Audio, input, and simulation are designed for bit-perfect reproducibility, enabling reliable replays and lockstep networking.

3. **Standard Compliance** — Normative language follows [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119). Defaults mirror [SVG 2.0](https://www.w3.org/TR/SVG2/), [glTF 2.0](https://www.khronos.org/gltf/), and [Lottie](https://airbnb.io/lottie/).

## Technical Specifications Summary

### Hardware Constraints

```
Cartridge Size      : ≤ 32 MiB (code + assets)
Script Source       : ≤ 1 MiB
Runtime Heap        : ≤ 64 MiB
Scene Nodes         : ≤ 50,000 per frame
Render Commands     : ≤ 200,000 per frame
3D Triangles        : ≤ 500,000 per frame
Audio Sample Rate   : 48 kHz stereo
SFX Channels        : 32 polyphonic voices
Controllers         : Up to 4 gamepads
```

### Scripting Language

The console uses **Squirrel** — a high-level, object-oriented scripting language with C-like syntax. Scripts define the standard entry points:

```c
function init()       // Called once at cartridge load
function update(dt)   // Fixed-step simulation (e.g., 60 Hz)
function draw()       // Variable-step presentation
```

## Related Repositories

- **Implementation** — The runtime implementation will be maintained in a separate repository.

## License

This specification documentation is licensed under the **GNU Affero General Public License v3.0 (AGPL-3.0)**.

See [LICENSE](LICENSE) for the full license text.

---

**Copyright © 2024 Michele Fabbri. All rights reserved.**
