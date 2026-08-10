# Indexed Drawing — Cube and Ground

> **Build-Along Milestone:** You can submit one `SceneDrawItem` for `UnitCube` and one for `GroundPlane` and see both on screen in SparkDemo.

**You will learn:**

- Map `SceneMeshSlot` to index ranges in the static buffer.
- Use push constants for per-draw `model` matrix.
- Combine view-projection UBO with per-object transforms.

## 1. The Problem We Are Solving

A single triangle proved the pipeline works. Real scenes need **many objects** sharing GPU memory. Spark's static buffer holds a **unit cube** and a **ground plane** with separate index ranges. Each `SceneDrawItem` selects a slot; the opaque pass issues `vkCmdDrawIndexed` with the correct `firstIndex` and `indexCount`.

This chapter is the first time `SceneRenderParams` becomes **visible geometry** — the payoff for Chapters 1–6.

## 2. Core Concepts

### SceneMeshSlot routing

| Slot | Geometry | Typical use |
|------|----------|-------------|
| `UnitCube` | 1×1×1 cube centered at origin | Props, debug blocks |
| `GroundPlane` | Large XZ quad at y=0 | Floor reference |
| `Custom` | Dynamic VB from `Mesh` asset | Imported models (Part VI) |

Renderer stores counts from `CreateSceneGeometry`:

```cpp
std::uint32_t cubeIndexCount;
std::uint32_t planeIndexCount;
std::uint32_t planeFirstIndex;
std::uint32_t cubeVertexOffset;
std::uint32_t planeVertexOffset;
```

### Model matrix per draw

Each draw item carries `Matrix4 model`. Vertex shader computes:

```text
worldPos = model * localPos
clipPos  = viewProj * worldPos
```

Spark passes `model` via **push constants** to avoid rebinding descriptors per mesh.

### Ground plane scale

The plane vertices span `±groundExtent` on X and Z with normal +Y. Draw items usually leave `model` as identity for a floor at y=0, or translate/scaling for level layout.

### Albedo and lighting

Part II uses simple directional light from `SceneRenderParams`. Fragment shader multiplies albedo by Lambert term `max(N·L, 0)`. No textures required yet — set `textureLayer = -1`.

### Multiple draws per frame

```cpp
params.draws.PushBack(floorItem);
params.draws.PushBack(cubeItem);
```

Opaque pass loops all items in `pendingScene.draws` — order matters for opaque depth when overlapping (not an issue for floor vs cube above it).

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Builtin geometry at renderer init

**Concept:** SceneMeshSlot routing

**Decision:** `CreateSceneGeometry` during `VulkanRenderer` setup, not per demo

**Why:** Demos only fill `SceneDrawItem`; they never upload cube VB themselves.

**In Spark:** `VulkanRenderer::CreateSceneGeometry`.

### Push constants for model matrix

**Concept:** Model matrix per draw

**Decision:** `ModelPushConstants` struct in `scene.vert` push_constant block

**Why:** Updates per draw without descriptor set updates — lower CPU overhead for many objects.

**In Spark:** `VulkanSceneOpaquePass` — `vkCmdPushConstants` before each `vkCmdDrawIndexed`.

### CCW ground plane indices

**Concept:** Ground plane scale

**Decision:** Index order chosen so normal points +Y with back-face cull enabled

**Why:** Wrong winding culls entire plane — Spark comment documents fix in `CreateSceneGeometry`.

**In Spark:** `VulkanRenderer.cpp` plane index triplets.

### UnitCube default on SceneDrawItem

**Concept:** Multiple draws

**Decision:** `SceneDrawItem::mesh` defaults to `UnitCube`

**Why:** Quick debug draws without setting every field.

**In Spark:** `SceneRenderParams.hpp`.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book; Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `src/spark/render/core/VulkanRenderer.cpp` — `CreateSceneGeometry`
- `src/spark/render/scene/VulkanSceneOpaquePass.cpp`
- `shaders/scene.vert` — push constants + `ubo.viewProj`
- Any demo filling `SceneRenderParams` (e.g. `ThreeDDemo`)

### Step 1 — Fill scene params in a demo

```cpp
void MyDemo::OnRender(IRenderFrame&, IEngineContext& context) {
    SceneRenderParams params{};

    int w = 0, h = 0;
    context.GetFramebufferSize(w, h);
    const float aspect = w > 0 && h > 0 ? static_cast<float>(w) / static_cast<float>(h) : 16.0f / 9.0f;

    const Vector3 eye{4.0F, 3.0F, 6.0F};
    const Vector3 target{0.0F, 0.5F, 0.0F};
    params.viewProjection = Matrix4::PerspectiveVulkan(60.0F, aspect, 0.1F, 200.0F)
                          * Matrix4::LookAt(eye, target, Vector3::UnitY);
    params.cameraPositionWorld = eye;
    params.lightDirectionWorld = Vector3{0.35F, 0.92F, 0.18F}.Normalized();

    SceneDrawItem floor{};
    floor.mesh = SceneMeshSlot::GroundPlane;
    floor.albedo = Vector3{0.25F, 0.28F, 0.32F};
  params.draws.PushBack(floor);

    SceneDrawItem cube{};
    cube.mesh = SceneMeshSlot::UnitCube;
    cube.model = Matrix4::Translation(0.0F, 0.5F, 0.0F);
    cube.albedo = Vector3{0.9F, 0.3F, 0.2F};
    params.draws.PushBack(cube);

    context.SetSceneRenderParams(params);
}
```

Run SparkDemo with a 3D mode using similar logic — you should see a floor and cube.

### Step 2 — Opaque pass draw loop

```cpp
// VulkanSceneOpaquePass.cpp (conceptual)
    for (const SceneDrawItem& item : ctx.scene->draws) {
        MeshDrawRange range = ResolveBuiltinRange(item.mesh, ctx.meshBindings);
        PushModelConstants(commandBuffer, ctx.pipelineLayout, item);
        vkCmdDrawIndexed(commandBuffer, range.indexCount, 1, range.firstIndex, range.vertexOffset, 0);
    }
```

Read actual `ResolveBuiltinRange` / equivalent in Spark source for exact field names.

### Step 3 — Verify index ranges

From `CreateSceneGeometry`:

- Cube: `firstIndex = 0`, `indexCount = cubeIndexCount`
- Plane: `firstIndex = planeFirstIndex`, `indexCount = planeIndexCount`

Log these once at startup to confirm upload succeeded.

### Step 4 — RenderDoc capture

Capture a frame. You should see two indexed draw calls with the same vertex/index buffers, different push constant data.

### Step 5 — Move the cube

Animate `cube.model` in `OnUpdate` with `Matrix4::RotationY(totalTime)` — transforms update CPU-side; GPU reads new push constants each frame.

## 5. How This Connects to the Rest of Your Engine

Chapter **10** replaces hand-filled params with `SceneSubmit` from ECS. Part **IV** adds textures via `textureLayer`. Part **VI** uses `SceneMeshSlot::Custom` for glTF assets.

Cube + ground prove **indexed drawing + UBO + push constants** — the core forward mesh path.

## 6. Pitfalls and Debugging

- **Cube inside plane** — cube y translation must be ≥ 0.5 for unit cube sitting on floor.
- **Black screen with valid draws** — `viewProjection` wrong; camera inside geometry.
- **Missing sceneParamsValid** — forgot `SetSceneRenderParams`.
- **Plane not visible** — culled due to winding; check indices in `CreateSceneGeometry`.

## 7. Exercises

1. Stack three cubes with different `model` translations — how many draw calls?
2. Scale floor by non-uniform `model` matrix — does lighting still look correct?
3. Disable depth test in a local build — what happens when cube intersects plane?
4. Find which SparkDemo mode is simplest to test cube + ground.

## Summary

Indexed draws select subranges of shared vertex/index buffers using `SceneMeshSlot`. Per-draw `model` matrices go through push constants; per-frame `viewProjection` lives in `SceneUBO`. Fill `SceneRenderParams` with floor and cube items — Part II's visual milestone.

---

Next: [VulkanRenderer Frame Graph](09-vulkan-renderer-graph.md).
