# The Render Data Contract

> **Build-Along Milestone:** You can describe `SceneRenderParams` and `SceneDrawItem` fields and explain why game code never passes `VkBuffer` handles.

**You will learn:**

- Define CPU-side structures that describe a frame without Vulkan types.
- Explain why the snapshot must be immutable while the GPU reads it.
- Plumb draw commands from game code to `VulkanRenderer::pendingScene`.

## 1. The Problem We Are Solving

Part I gave you a clearing window with working sync. Part II answers: **what does the GPU draw?**

Gameplay knows about entities, meshes, and lights. Vulkan knows about buffers, pipelines, and descriptor sets. You need a **contract** — plain C++ structs — that the game fills and the renderer reads. Spark calls this contract `SceneRenderParams`.

Without this boundary, every new feature leaks `VkBuffer` into demo headers. With it, you can unit-test snapshot building, swap render backends, and reason about one frame's data as a single document handed to the GPU.

Open `include/spark/engine/SceneRenderParams.hpp` while you read.

## 2. Core Concepts

### Snapshot vs simulation state

| Layer | Owns | Example |
|-------|------|---------|
| Simulation | Mutable ECS / game objects | `TransformComponent`, velocity |
| Snapshot | Immutable per-frame copy | `SceneRenderParams` |
| GPU backend | Vulkan resources | `VkBuffer`, pipelines |

The game **writes** the snapshot once per frame in `OnRender`. The renderer **reads** it during `DrawFrame`. Simulation must not mutate arrays the GPU still consumes from the previous frame.

### SceneDrawItem — one draw call's worth of data

Each opaque or transparent mesh draw is a `SceneDrawItem`:

| Field | Role |
|-------|------|
| `mesh` | Built-in slot (`UnitCube`, `GroundPlane`) or `Custom` |
| `model` | World transform matrix |
| `albedo` | Color multiplier before textures |
| `textureLayer` | Index into `sceneTextures` (-1 = none) |
| `opacity` | 1 = opaque; &lt; 1 routes to `transparentDraws` |

Later parts add PBR channels, skinning, and shadow flags — the pattern stays the same: **flat struct, no pointers to Vulkan**.

### SceneRenderParams — the full frame description

Core 3D fields you use in Part II:

```cpp
struct SceneRenderParams {
    Matrix4 viewProjection{};
    Array<SceneDrawItem> draws;
    Array<SceneDrawItem> transparentDraws;
    Array<SharedPtr<Texture2D>> sceneTextures;
    Vector3 lightDirectionWorld;
    Vector3 cameraPositionWorld;
    Vector3 lightColor;
    float lightIntensity;
    Vector3 ambientColor;
    // ... lighting, sprites, UI — added in later parts
};
```

`viewProjection` is written once per frame from the camera. `draws` is append-only during scene build. The renderer copies the whole struct:

```cpp
void VulkanRenderer::SetSceneRenderParams(const SceneRenderParams& params) {
    pendingScene = params;
    sceneParamsValid = true;
}
```

### Built-in mesh slots

Spark packs a unit cube and ground plane into one shared vertex/index buffer at startup (`CreateSceneGeometry`). Draw items reference them by slot:

```cpp
enum class SceneMeshSlot : std::uint8_t {
    UnitCube = 0,
    GroundPlane = 1,
    Custom = 2,
};
```

Part II chapter 8 draws the cube and plane without loading external files.

### Texture array indexing

`sceneTextures` is a CPU array of `Texture2D` assets. Each draw stores an **index** (`textureLayer`), not a `VkImageView`. The renderer uploads textures and binds them as a 2D array in a descriptor set (Part IV). Keeping indices in the snapshot decouples asset resolution from draw recording.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Plain structs in engine headers

**Concept:** Snapshot vs simulation

**Decision:** `SceneRenderParams` lives in `include/spark/engine/` with no Vulkan includes

**Why:** Game code and tests compile without linking the Vulkan loader. The struct is serializable documentation of one frame.

**In Spark:** `SceneRenderParams.hpp` — only math, arrays, and scene asset types.

### Copy into pendingScene, not pointer

**Concept:** Immutability during GPU work

**Decision:** `SetSceneRenderParams` assigns by value into `VulkanRenderer::pendingScene`

**Why:** If the renderer held a pointer to game-owned memory, a `draws.Clear()` during GPU execution causes use-after-free. Copying trades memory for safety — acceptable for hundreds of draws per frame.

**In Spark:** `VulkanRenderer.cpp` — `SetSceneRenderParams`.

### Separate opaque and transparent arrays

**Concept:** SceneDrawItem routing

**Decision:** `draws` for opaque; `transparentDraws` sorted back-to-front when needed

**Why:** Opaque passes use depth write; transparent passes change blend state and sort order. Splitting arrays keeps the opaque loop simple in Part II.

**In Spark:** `PartitionSortedDrawItemsIntoSceneParams` in `SceneSubmit.hpp`.

### Mesh slot enum instead of raw buffer handles

**Concept:** Built-in mesh slots

**Decision:** `SceneMeshSlot` indexes into renderer-owned static geometry

**Why:** Demos draw cubes without knowing vertex buffer offsets. Custom meshes use `SharedPtr<Mesh>` on the draw item only when `mesh == Custom`.

**In Spark:** `VulkanSceneOpaquePass` — `cubeIndexCount`, `planeFirstIndex` bindings.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book; Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `include/spark/engine/SceneRenderParams.hpp`
- `include/spark/engine/IEngineContext.hpp`
- `src/spark/render/core/VulkanRenderer.cpp`
- `include/spark/render/core/VulkanRenderer.hpp`

### Step 1 — Read SceneDrawItem

```cpp
// include/spark/engine/SceneRenderParams.hpp (lines 170–218, abbreviated)
struct SceneDrawItem {
    SceneMeshSlot mesh = SceneMeshSlot::UnitCube;
    SceneSkyMode skyMode = SceneSkyMode::None;
    Matrix4 model = Matrix4::Identity;
    Vector3 albedo{1.0F, 1.0F, 1.0F};
    std::int32_t textureLayer = -1;
    // ... PBR, skinning, shadows — used in later parts
    float opacity = 1.0F;
};
```

**Build-along:** Add a comment in your engine listing the minimum fields you need for Part II: mesh id, model matrix, color.

### Step 2 — Read SceneRenderParams core fields

```cpp
// include/spark/engine/SceneRenderParams.hpp (lines 299–315)
struct SceneRenderParams {
    static constexpr std::uint32_t MaxSceneTextures = 16;

    Matrix4 viewProjection{};
    Array<SceneDrawItem> draws;
    Array<SceneDrawItem> transparentDraws;
    Array<SharedPtr<Texture2D>> sceneTextures;
    Vector3 lightDirectionWorld{0.35F, 0.92F, 0.18F};
    Vector3 cameraPositionWorld{};
    Vector3 lightColor{1.0F, 0.97F, 0.9F};
    float lightIntensity = 0.92F;
    Vector3 ambientColor{0.05F, 0.055F, 0.07F};
    // ...
};
```

### Step 3 — Trace SetSceneRenderParams from game code

Demos call through the facade:

```cpp
// Typical demo OnRender pattern
SceneRenderParams params{};
params.viewProjection = ...;
params.draws.PushBack(item);
context.SetSceneRenderParams(params);
```

`IEngineContext::SetSceneRenderParams` forwards to `IFramePresenter` — game never includes `VulkanRenderer.hpp`.

### Step 4 — See renderer storage

```cpp
// src/spark/render/core/VulkanRenderer.cpp
void VulkanRenderer::SetSceneRenderParams(const SceneRenderParams& params) {
    pendingScene = params;
    sceneParamsValid = true;
    resolvedLighting = SceneLightingResolver::Resolve(pendingScene);
}
```

`sceneParamsValid` gates drawing — empty or unset params skip mesh passes safely.

### Step 5 — Build a minimal snapshot by hand

In a demo's `OnRender`, before using ECS helpers:

```cpp
SceneRenderParams params{};
params.viewProjection = Matrix4::PerspectiveVulkan(60.0f, aspect, 0.1f, 100.0f)
                      * Matrix4::LookAt(eye, target, up);
params.cameraPositionWorld = eye;

SceneDrawItem cube{};
cube.mesh = SceneMeshSlot::UnitCube;
cube.model = Matrix4::Translation(0, 0.5f, 0);
cube.albedo = Vector3{1, 0.2f, 0.2f};
params.draws.PushBack(cube);

context.SetSceneRenderParams(params);
```

You will not see the cube until Part II finishes pipelines and buffers — but the data path is correct.

## 5. How This Connects to the Rest of Your Engine

Chapter **2** creates render passes that clear and draw into HDR targets. Chapter **5** uploads mesh data the slots reference. Chapter **10** replaces hand-filled params with `SceneSubmit` from ECS.

Part I's arrow now has concrete types:

```text
OnRender → SceneRenderParams → SetSceneRenderParams → RecordSceneCommandBuffer
```

## 6. Pitfalls and Debugging

- **Storing `VkBuffer` in SceneDrawItem** — breaks layering; use mesh slots or opaque asset ids.
- **Mutating `draws` after submit** — race the GPU; rebuild params each frame or double-buffer snapshots.
- **Forgetting `viewProjection`** — geometry renders but outside clip space — black screen.
- **textureLayer without sceneTextures entry** — validation or garbage sampling in Part IV.

## 7. Exercises

1. List ten fields in `SceneRenderParams` you will use in Part II vs fields you can ignore until Part IV.
2. Draw a diagram: demo `OnRender` → `pendingScene` → opaque pass loop.
3. Why is `SharedPtr<Mesh>` acceptable on a draw item but `VkBuffer` is not?
4. Count `sizeof` impact of copying 200 `SceneDrawItem` structs — is copy still reasonable?

## Summary

`SceneRenderParams` is the render snapshot: view/projection, draw lists, lights, and texture indices as plain data. Spark copies it into the renderer each frame, keeping Vulkan out of game headers. Every subsequent Part II chapter implements one step of turning that snapshot into pixels.

---

Next: [Render Passes and Framebuffers](02-render-passes.md).
