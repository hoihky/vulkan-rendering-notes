# Building a Real-Time Renderer with Vulkan

This book teaches you **how to design and implement** a real-time rendering engine with Vulkan — by building alongside **Spark**, a C++23 reference engine. Each chapter adds a subsystem to the same mini-engine: you read the theory, study Spark's code, then mirror the pattern in your own project.

## How to read this book

1. **Open Spark** — clone the repo, build with CMake, and keep the cited files visible while reading.
2. **Learn the concepts** — each chapter explains the theory (math, GPU behavior, engine architecture).
3. **Follow the design decisions** — every choice links back to a concept and explains *why* Spark structured it that way.
4. **Build along** — the implementation walkthrough cites real Spark source with line numbers; diff your code against it.
5. **Do the exercises** — reinforce each chapter with small tasks in your engine or Spark forks.

## Prerequisites

- C++17 or later, comfortable with pointers and RAII
- Linear algebra (vectors, matrices, dot/cross product)
- CMake basics and the [Vulkan SDK](https://vulkan.lunarg.com/)
- Optional: clone Spark to run demos while reading

## Parts

| Part | You will build / understand |
|------|-----------------------------|
| [I](part-01-vulkan-setup/index.md) | Window, instance, device, swapchain, frame loop |
| [II](part-02-basic-pipeline/index.md) | Render passes, buffers, pipelines, first draws |
| [III](part-03-shaders/index.md) | Shader toolchain, UBOs, push constants |
| [IV](part-04-textures-lighting/index.md) | Textures, samplers, lights, shading |
| [V](part-05-input-camera/index.md) | Input and camera matrices |
| [VI](part-06-models-animation/index.md) | Mesh loading and skinning |
| [VII](part-07-sky-materials/index.md) | Sky, materials, PBR |
| [VIII](part-08-particles/index.md) | Particle systems |
| [IX](part-09-shadows/index.md) | Shadow mapping |
| [X](part-10-hdr-post/index.md) | HDR and post-processing |
| [XI](part-11-2d-rendering/index.md) | 2D sprites, tilemaps, UI |
| [XII](part-12-terrain/index.md) | Terrain and culling |
| [XIII](part-13-scene-editor/index.md) | Scene serialization and tools |
| [XIV](part-14-physics/index.md) | Physics and rendering sync |
| [XV](part-15-audio-ai-scripting/index.md) | Adjacent engine systems |
| [XVI](part-16-production/index.md) | Shipping and hardening |
| [Appendix](appendix/index.md) | Quick reference |

Start with [Chapter 1 — What Is a Real-Time Rendering Engine?](part-01-vulkan-setup/01-what-is-a-renderer.md).
