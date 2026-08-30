# Submitting a Scene from Code

> **Build-Along Milestone:** You can fill `SceneRenderParams` manually or via `FillStandardLitSceneFromWorld` and see a lit 3D scene in SparkDemo.

**You will learn:**

- Submit draws without ECS using manual `SceneRenderParams`.
- Introduce `SceneSubmit` helpers that walk `GameWorld` for production paths.
- Connect `OnRender` to the full Part II pipeline end-to-end.

## 1. The Problem We Are Solving

Chapter 8 hand-authored floor and cube items. Real demos use **dozens of entities** with meshes, materials, and cameras. Duplicating that logic per demo is unmaintainable.

Spark centralizes ECS → snapshot translation in **`SceneSubmit`** (`FillStandardLitSceneFromWorld`, `SubmitStandardLitSceneFromWorld`). This chapter closes Part II by showing both paths:

1. **Manual** — learn the data contract
2. **Submit helper** — how Spark demos scale

Either way, the last line is `context.SetSceneRenderParams(params)` before `PresentFrame`.

### Your first end-to-end frame (beginner checklist)

By the end of Part II, one complete frame looks like this on the CPU:

```text
1. Engine::Run wakes up for frame N
2. game->OnUpdate(dt)           — move the camera, animate objects
3. game->OnRender(frame, ctx):
     a. Build SceneRenderParams params
     b. params.camera = ...      — view + projection matrices
     c. FillStandardLitSceneFromWorld(world, params, ctx)
        OR manually push SceneDrawItem entries
     d. ctx.SetSceneRenderParams(params)
4. renderer->PresentFrame()
     a. DrawFrame → RecordSceneCommandBuffer
     b. sceneOpaquePass draws each item in params.draws
     c. tonemap + present
5. Monitor shows frame N
```

If the screen is black, ask **which step failed**:

| Symptom | Likely broken step |
|---------|-------------------|
| Game runs, black screen | 3c empty `params.draws`, or 4b pipeline/render pass |
| Gray clear color only | 4b render pass clears but no draws recorded |
| Geometry wrong place | 3b camera matrices |
| White untextured mesh | Material/submit layers (Part IV/VII) |
| Validation error on submit | Part I sync or Part II pipeline |

### SceneDrawItem: the smallest unit of rendering

Each thing the GPU draws is one **draw item**. At minimum it carries:

| Field | Purpose |
|-------|---------|
| `model` | 4×4 world matrix (object → world space) |
| `mesh` / `customMesh` | Which geometry (unit cube slot or SharedPtr mesh) |
| `submeshIndex` | Which index range for multi-material meshes |
| `albedo` | Fallback tint when no texture |
| `textureLayer` | Index into `sceneTextures` array (-1 = none) |
| `metallic`, `roughness` | PBR scalars for fragment shader |

```cpp
// Building one item by hand (learning exercise)
SceneDrawItem item{};
item.model = transform.ToMatrix4();
item.mesh = SceneMeshSlot::UnitCube;
item.albedo = {0.8f, 0.2f, 0.2f};
item.metallic = 0.0f;
item.roughness = 0.5f;
params.draws.PushBack(item);
```

Production code never hand-assembles hundreds of these — `FillStandardLitSceneFromWorld` does it from components. Start manual to learn the contract, then switch to the helper.

## 1b. Concept Deep Dive — SceneSubmit ECS Translation

### Two paths, one struct type

Spark demos converge on `SceneRenderParams` whether you fill it by hand or via ECS helpers. The renderer cannot tell which path built the snapshot:

```text
Manual path:
  OnRender → build SceneDrawItem[] → SetSceneRenderParams

ECS path:
  OnRender → FillStandardLitSceneFromWorld(world, params)
          → SetSceneRenderParams
          (or SubmitStandardLitSceneFromWorld wraps both)
```

Both end with the same `VulkanRenderer::pendingScene` copy and the same opaque pass loop.

### SubmitStandardLitSceneFromWorld — fill then submit

The convenience wrapper allocates params, calls the fill helper with camera and lighting arguments, then forwards to the engine context:

```cpp
// src/spark/scene/submit/SceneSubmit.cpp (lines 603–633)
void SubmitStandardLitSceneFromWorld(
        GameWorld& world,
        IEngineContext& context,
        const Matrix4& viewProjection,
        const Vector3& cameraPositionWorld,
        /* lighting, particles, time ... */) {
    SceneRenderParams params{};
    FillStandardLitSceneFromWorld(world, context, viewProjection, cameraPositionWorld,
            lightDirectionWorld, lightColor, lightIntensity, ambientColor,
            enableParticles, particleCameraRight, particleCameraUp,
            sceneTimeSeconds, params, spriteSortMode);
    context.SetSceneRenderParams(params);
}
```

Demos with a `MainCamera` often call `SubmitStandardLitSceneFromWorldWithCamera`, which derives view/projection from ECS before filling.

### PushRigidMeshDraws — MeshComponent to SceneDrawItem

For each rigid mesh entity, SceneSubmit builds a base draw item from the world matrix and expands submeshes when multi-material slots exist:

```cpp
// src/spark/scene/submit/SceneSubmit.cpp (lines 77–98)
void PushRigidMeshDraws(
        Array<SceneDrawItem>& drawList,
        SceneDrawItem baseItem,
        const Mesh& mesh,
        const MaterialComponent* mat,
        const MultiMaterialComponent* multiMat,
        SceneRenderParams& params,
        const FindSceneTextureFn& findOrAddTexture) {
    const Array<MeshSubmesh>& submeshes = mesh.GetSubmeshes();
    if (submeshes.IsEmpty() || multiMat == nullptr) {
        SceneDrawItem item = baseItem;
        item.submeshIndex = kSceneDrawFullSubmesh;
        if (mat != nullptr) {
            ApplyMaterialComponentToSceneDrawItem(item, mat, &params);
            ApplyAlbedoTexture(item, mat->GetBaseColorTexture(), mat->GetTint(), findOrAddTexture);
        }
        drawList.PushBack(item);
        return;
    }
    // ... one SceneDrawItem per submesh with material slot ...
}
```

Builtin slots (`UnitCube`, `GroundPlane`) set `baseItem.mesh` from `MeshComponent::slot`; custom assets set `mesh = Custom` with `customMesh` shared pointer.

### Partition and sort — opaque vs transparent arrays

After gathering, SceneSubmit stable-sorts by material/state and partitions into `draws` vs `transparentDraws` based on opacity:

```cpp
// src/spark/scene/submit/SceneSubmit.cpp (lines 466–467)
    SceneSubmitDetail::StableSortDrawItems(drawList);
    PartitionSortedDrawItemsIntoSceneParams(drawList, params, cameraPositionWorld);
```

Transparent items sort back-to-front by camera distance — the opaque pass reads `params.draws` only; transparent resume uses `params.transparentDraws`.

### Complete Part II frame graph (CPU + GPU)

```text
Engine::Run
  game->OnUpdate(dt)
  game->OnRender:
    FillStandardLitSceneFromWorld OR manual params
    context.SetSceneRenderParams(params)
  PresentFrame → DrawFrame
    WriteUniformBuffer(resolvedLighting from params)
    RecordSceneCommandBuffer
      sceneOpaquePass.Record(params.draws)
    tonemap → vkQueuePresentKHR
```

Chapter 1's contract, chapters 2–7's Vulkan objects, and this chapter's ECS glue — together they produce a lit 3D frame in SparkDemo.

## 2. Core Concepts

### Manual vs ECS-driven submit

| Path | When to use |
|------|-------------|
| Manual `SceneRenderParams` | Prototypes, unit tests, this book's exercises |
| `FillStandardLitSceneFromWorld` | Demos with `GameObject` + `MeshComponent` |

Both produce the same struct type — renderer cannot tell the difference.

### SceneSubmit responsibilities

`FillStandardLitSceneFromWorld`:

- Resolves main camera → `viewProjection`, `cameraPositionWorld`
- Walks enabled `MeshComponent` / `MaterialComponent` pairs
- Fills `SceneDrawItem` with transforms, albedo, texture layers
- Gathers lights (directional from params or components)
- Optionally culls with scene frustum (Part XII preview)

It does **not** call Vulkan — only fills CPU arrays.

### Texture registration

When a draw references `Texture2D`, submit assigns `textureLayer` by appending to `params.sceneTextures` (max 16 in Part II scope). Duplicate textures share the same layer index.

### SubmitStandardLitSceneFromWorld convenience

Wraps fill + `context.SetSceneRenderParams` in one call for demos:

```cpp
SubmitStandardLitSceneFromWorld(world, context, ...);
```

Read signature in `SceneSubmit.hpp` for optional flags (particles, sprite sort mode).

### End-to-end Part II flow

```text
Engine::Run
  → OnUpdate (gameplay)
  → OnRender
       → build SceneRenderParams (manual or SceneSubmit)
       → context.SetSceneRenderParams
  → PresentFrame
       → DrawFrame
            → WriteUniformBuffer
            → RecordSceneCommandBuffer
                 → sceneOpaquePass.Record
  → display
```

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### SceneSubmit as free functions

**Concept:** SceneSubmit responsibilities

**Decision:** Namespace-level functions, not methods on `Scene`

**Why:** Avoids `Scene` depending on renderer; submit logic callable from editor and demos alike.

**In Spark:** `include/spark/scene/SceneSubmit.hpp`.

### ApplyMaterialComponentToSceneDrawItem

**Concept:** Texture registration

**Decision:** Shared helper copies material fields into `SceneDrawItem`

**Why:** MaterialComponent mapping used by ECS walk and editor preview — one mapping table.

**In Spark:** `SceneSubmit.hpp` — `ApplyMaterialComponentToSceneDrawItem`.

### Partition opaque vs transparent

**Concept:** Manual vs ECS submit

**Decision:** `PartitionSortedDrawItemsIntoSceneParams` splits by opacity after gather

**Why:** Opaque pass needs depth write; transparent pass sorted back-to-front.

**In Spark:** `SceneSubmit.hpp`.

### Camera from world

**Concept:** FillStandardLitSceneFromWorld

**Decision:** `TryFillSceneCameraFromWorld` prefers enabled camera components

**Why:** Demos do not compute matrices manually; consistent projection rules.

**In Spark:** `SceneSubmit.cpp` + `CameraComponent`.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book; Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `include/spark/scene/SceneSubmit.hpp`
- `src/spark/scene/SceneSubmit.cpp`
- `src/spark/demo/` — any 3D demo `OnRender`
- `include/spark/engine/IEngineContext.hpp`

### Step 1 — Read SceneSubmit API

```cpp
// include/spark/scene/SceneSubmit.hpp (abbreviated)
void FillStandardLitSceneFromWorld(
        GameWorld& world,
        IEngineContext& context,
        const Matrix4& viewProjection,
        const Vector3& cameraPositionWorld,
        const Vector3& lightDirectionWorld,
        const Vector3& lightColor,
        float lightIntensity,
        const Vector3& ambientColor,
        bool enableParticles,
        /* camera basis for particles */,
        float sceneTimeSeconds,
        SceneRenderParams& outParams,
        ...);

void SubmitStandardLitSceneFromWorld(/* similar args, calls SetSceneRenderParams */);
```

When starting out, ignore particle and sprite parameters — pass `false` and defaults.

### Step 2 — Manual submit (review Chapter 8)

Complete minimal demo without ECS — already shown in Chapter 8. Verify in RenderDoc: two draws, one UBO update per frame.

### Step 3 — Find a demo OnRender

Open a simple 3D demo (e.g. material showcase or maze). Pattern:

```cpp
void SomeDemo::OnRender(IRenderFrame&, IEngineContext& context) {
    SceneRenderParams params{};
    FillStandardLitSceneFromWorld(
            world,
            context,
            /* matrices from camera component */,
            lightDir, lightColor, lightIntensity, ambient,
            false, /* particles */
            ...);
    context.SetSceneRenderParams(params);
}
```

Or one-liner `SubmitStandardLitSceneFromWorld`.

### Step 4 — ECS rigid mesh sink (internals)

Inside `SceneSubmit.cpp`, frustum walk calls sinks like `RigidDrawableSubmitSink::OnDrawable`:

```cpp
        SceneDrawItem item{};
        item.mesh = SceneMeshSlot::Custom;  // or builtin if procedural
        item.model = worldM;
        ApplyMaterialComponentToSceneDrawItem(item, mat, &params);
        drawList.PushBack(item);
```

Builtin slots used when mesh matches unit cube / plane prototypes.

### Step 5 — Full Part II checklist

Run SparkDemo 3D mode and verify:

- [ ] Window opens (Part I)
- [ ] No validation errors (Part I–II)
- [ ] Floor + objects visible (Chapter 8+)
- [ ] Camera moves with input (Part V preview — optional)
- [ ] `SetSceneRenderParams` called every frame

## 5. How This Connects to the Rest of Your Engine

Part **III** improves shader quality on the same submit path. Part **VII** explains ECS components SceneSubmit reads. Part **X** adds post passes after scene record — submit path unchanged.

Part II ends at: **snapshot in, lit meshes out.**

## 6. Pitfalls and Debugging

- **Empty world** — no mesh components; scene clears to background only.
- **Camera disabled** — zero matrices; nothing visible.
- **Forgot SetSceneRenderParams** — GPU uses stale pendingScene.
- **Too many textures** — exceeds `MaxSceneTextures`; assert or clamp in debug.

## 7. Exercises

1. Pick one SparkDemo and trace `OnRender` to `vkCmdDrawIndexed`.
2. Replace `FillStandardLitSceneFromWorld` with manual params for one cube — same visual?
3. List five component types `SceneSubmit` reads (mesh, material, ...).
4. What happens if you call `SetSceneRenderParams` in `OnUpdate` instead of `OnRender`?

## Summary

Part II completes the path from game code to pixels: build `SceneRenderParams` manually or via `SceneSubmit`, push through `IEngineContext`, and let `VulkanRenderer` record indexed draws in the HDR pass. You now have the basic forward pipeline — Parts III onward improve shaders, materials, and frame graph nodes without changing this contract.

---

Next: [Part III — Shader Essentials](../part-03-shaders/index.md).
