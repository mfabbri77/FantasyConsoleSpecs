# Fantasy Console Documentation

Welcome to the definitive developer documentation for the **Fantasy Console**.

This platform represents a paradigm shift in the concept of "fantasy" hardware. It bridges the gap between the nostalgic charm of retro computing and the architectural robustness of modern game engines. Unlike traditional fantasy consoles that rely on direct framebuffer manipulation or immediate-mode blitting, this console is built entirely upon a **Retained Mode** architecture.

Whether you are rendering 2D vector paths or complex 3D meshes, you do not draw pixels directly. Instead, you construct persistent scenes, manipulate object hierarchies, and declare animations. The runtime—powered by a modern GPU renderer—handles the rasterisation, occlusion, and compositing with mathematical precision.

## Core Philosophy

1.  **Retained Mode First:** You build the scene; the console draws it. This allows for optimisations like automatic batching, infinite resolution independent zooming, and complex scene graph manipulations that would be impossible in a pure retro-pixel environment.
2.  **Determinism:** The audio, input, and simulation steps are designed for bit-perfect reproducibility, enabling reliable replays and lockstep networking.
3.  **Standard Compliance:** Normative language in this documentation follows [RFC 2119]. Keywords such as **MUST**, **SHOULD**, and **MAY** indicate requirements. Defaults mirror the behaviour of [SVG 2.0], [glTF 2.0], and the [Lottie] animation format.

## Documentation Structure

Each chapter lives in its own Markdown file. The documentation has been reorganised to follow the logical flow of building a retained-mode application: strictly defining hardware limits, understanding the lifecycle, building visual assets (2D/Text), animating them, expanding into 3D, and finally handling audio and input.

| Chapter | File | Description |
| :--- | :--- | :--- |
| 1 | **01_HardwareSpecification.md** | **Hardware & Architecture:** System constraints, the 800×600 and 960×540 video modes, colour pipelines, and memory budgets. |
| 2 | **02_Lifecycle.md** | **Script Lifecycle:** The game loop (`init`, `update`, `draw`), event callbacks, hot-reloading, and error handling strategies. |
| 3 | **03_Graphics2D.md** | **2D Vector Engine:** The retained-mode API for paths, paints, strokes, and compositing. Defines the scene graph for 2D. |
| 4 | **04_TextAPI.md** | **Typography:** Advanced text rendering, font loading, shaping, and layout within the retained scene. |
| 5 | **05_AnimationInteractivity.md** | **Animation System:** Timeline management, property interpolation, easing functions, and Lottie integration. |
| 6 | **06_Graphics3D.md** | **3D Rendering:** The 3D scene graph, PBR materials, lights, cameras, and integration with the animation system. |
| 7 | **07_AudioAPI.md** | **Audio Subsystem:** Deterministic tracker module playback and the transient sound effect (SFX) system. |
| 8 | **08_InputAPI.md** | **Input Devices:** Polling and event handling for gamepads, mouse, and keyboard, including analogue mapping. |
| 9 | **09_TimeFilesystemDebug.md** | **System & Debug:** Time management, the filesystem sandbox, RNG, and debugging tools. |
| 10 | **10_ManifestStructure.md** | **Cartridge Format:** Packaging, manifest configuration, and asset definitions. |
| 11 | **11_Examples.md** | **Cookbook:** Comprehensive examples combining all systems to build interactive experiences. |

## Getting Started

Before writing code, understand that the console's *raison d'être* is to provide a constrained but high-level canvas. You are not pushing bytes to a VRAM address; you are orchestrating a visual symphony of nodes and resources.

Spend time exploring **Chapter 1** to understand the resolution and color space, then move to **Chapter 3** and **Chapter 6** to grasp the visual capabilities. We hope this platform inspires you to build the next generation of fantasy experiences.