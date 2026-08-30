# Camera to UBO

> **Build-Along Milestone:** `SceneRenderParams::viewProjection` and `cameraPositionWorld` reach `SceneUniformGpu` via `VulkanSceneUniformWriter`, powering `scene.vert`, sky rays, and specular in `scene.frag`.

**You will learn:**

- Place camera matrices on the render snapshot.
- Upload `viewProj`, `cameraPos`, and `invViewProj` to the scene UBO.
- Trace camera data from any rig type to GPU shaders.
- Avoid common mismatches between view matrix and camera position.

## 1. The Problem We Are Solving

Chapters 5–9 produced camera matrices on the CPU — fly cam, character rig, ECS resolve, 2D ortho. The GPU still reads **`ubo.viewProj`** and **`ubo.cameraPos`** from binding 0. The handoff is:

```text
Camera rig / resolver  →  SceneRenderParams  →  VulkanSceneUniformWriter  →  SceneUBO  →  shaders
```

This chapter closes Part V by documenting every field, who writes it, and how shaders consume it — connecting input and camera work to Part III–IV shader code.

### From camera on CPU to `ubo` on GPU (beginner)

Shaders cannot read your `FlyCamera` struct. Every frame you **copy** matrices into GPU memory:

```text
FlyCamera / CameraComponent
  → view, proj matrices multiplied → viewProj
  → SceneRenderParams.viewProjection
  → VulkanSceneUniformWriter::Write
  → VkBuffer (UBO binding 0)
  → scene.vert: gl_Position = ubo.viewProj * model * vec4(position, 1)
```

**Critical fields:**

| Field | Shader use |
|-------|------------|
| `viewProj` | Transform vertices to clip space |
| `cameraPos` | Specular, fog, rim lighting in `scene.frag` |
| `invViewProj` | Reconstruct world position from depth (sky, debug) |

**Common bugs:**

| Symptom | Likely cause |
|---------|--------------|
| Mesh off-center | View matrix inverted (eye vs target swapped) |
| Specular wrong direction | `cameraPos` not updated when using manual view matrix |
| Sky doesn't match scene | `invViewProj` stale or wrong column-major upload |

**Column-major:** GLM and GLSL expect column vectors — `viewProj * vec4(pos,1)`, not `vec4 * viewProj`.

Part V ends here: input → camera matrices → UBO → Part III shaders draw the frame.

## 1b. Concept Deep Dive — SceneUniformGpu Camera Fields

### std140 layout (656 bytes)

```cpp
// include/spark/render/scene/VulkanSceneUniformGpu.hpp
struct SceneUniformGpu {
    float viewProj[16]{};
    float lightDir[4]{};
    float cameraPos[4]{};
    float lightColor[4]{};
    float ambientColor[4]{};
    float invViewProj[16]{};
    float viewportSize[4]{};
    // shadow cascades, cluster grid, IBL params follow...
};
static_assert(sizeof(SceneUniformGpu) == 656);
```

Binding 0 in scene descriptor set 0 — updated every frame per swapchain image via host-visible or persistently mapped memory.

### Writer copies from SceneRenderParams

```cpp
// src/spark/render/scene/VulkanSceneUniformWriter.cpp (excerpt)
std::memcpy(u.viewProj, scene.viewProjection.m, sizeof(u.viewProj));
u.cameraPos[0] = scene.cameraPositionWorld.x;
u.cameraPos[1] = scene.cameraPositionWorld.y;
u.cameraPos[2] = scene.cameraPositionWorld.z;
Matrix4 invVp;
if (scene.viewProjection.TryInvert(invVp)) {
    std::memcpy(u.invViewProj, invVp.m, sizeof(u.invViewProj));
}
```

`cameraPositionWorld` must match the eye used to build `viewProj` — specular, fog, and rim in `scene.frag` read `ubo.cameraPos`, not a recomputed inverse view translation.

### Shader consumption map

| UBO field | Consumer |
|-----------|----------|
| `viewProj` | `scene.vert` clip transform |
| `cameraPos` | `scene.frag` view vector V for specular |
| `invViewProj` | Sky, SSR-style debug, world reconstruction |
| `viewportSize` | Screen-space effects, cluster indexing |

Any camera rig — `FlyCamera`, `CharacterCameraRig`, ECS `CameraComponent`, `Camera2D` — ends at the same two CPU fields: `SceneRenderParams::viewProjection` and `cameraPositionWorld`. Part V input work is complete once those fields are correct before `PresentFrame`.

## 2. Core Concepts

### SceneRenderParams camera fields

```cpp
struct SceneRenderParams {
    Matrix4 viewProjection{};
    Vector3 cameraPositionWorld{};
    // draws, lights, textures...
};
```

| Field | Source examples |
|-------|-----------------|
| `viewProjection` | `proj * view` from any camera path |
| `cameraPositionWorld` | Fly `position`, rig `CameraWorldPosition()`, ECS transform translation |

**Both required** — view matrix alone does not reliably encode eye position for shaders after projection multiply.

### Setting the snapshot

Manual demo:

```cpp
params.viewProjection = proj * view;
params.cameraPositionWorld = camera.position;
context.SetSceneRenderParams(params);
```

Auto path:

```cpp
SubmitStandardLitSceneFromWorldWithCamera(world, context, ...);
// internally TryFillSceneCameraFromWorld + gather
```

Renderer reads snapshot when recording commands for that frame slot.

### SceneUniformGpu layout (camera slice)

```cpp
struct SceneUniformGpu {
    float viewProj[16];
    float lightDir[4];
    float cameraPos[4];
    // ...
    float invViewProj[16];
    float viewportSize[4];  // xy = extent, used with invViewProj for sky
};
```

656-byte std140 block — Chapter 4 Part III. Camera fields at start and `invViewProj` offset 160.

### VulkanSceneUniformWriter::Write

```cpp
std::memcpy(u.viewProj, scene.viewProjection.m, sizeof(u.viewProj));

Matrix4 invVp;
if (scene.viewProjection.TryInvert(invVp))
    std::memcpy(u.invViewProj, invVp.m, sizeof(u.invViewProj));
else
    std::memcpy(u.invViewProj, Matrix4::Identity.m, sizeof(u.invViewProj));

u.cameraPos[0] = scene.cameraPositionWorld.x;
u.cameraPos[1] = scene.cameraPositionWorld.y;
u.cameraPos[2] = scene.cameraPositionWorld.z;
u.cameraPos[3] = 0.0f;

u.viewportSize[0] = static_cast<float>(swapchainExtent.width);
u.viewportSize[1] = static_cast<float>(swapchainExtent.height);
```

Runs per frame-in-flight slot after scene submit frozen.

### Shader consumption — vertex stage

```glsl
// scene.vert
vec4 clip = ubo.viewProj * vec4(worldPos, 1.0);
gl_Position = clip;  // or sky Z hack
```

Transforms **world-space** vertex positions from model/skinning to clip. Wrong `viewProj` — entire scene wrong; wrong `model` push — single object wrong.

### Shader consumption — fragment stage

**View vector for lighting:**

```glsl
vec3 V = normalize(ubo.cameraPos.xyz - vWorldPos);
```

**Sky ray reconstruction (equirect):**

```glsl
vec3 cam = ubo.cameraPos.xyz;
vec2 ndc = vec2(sx * 2.0 - 1.0, sy * 2.0 - 1.0);
vec4 farH = ubo.invViewProj * vec4(ndc, 1.0, 1.0);
vec3 farW = farH.xyz / max(farH.w, 1e-5);
vec3 dir = normalize(farW - cam);
```

`invViewProj` **not** stored in `SceneRenderParams` — computed at upload to save snapshot size and ensure consistency with uploaded `viewProj`.

### Clustered lights use same camera

CPU cluster builder (`VulkanClusteredForwardLights`) takes `viewProj` and `cameraPos` for tile assignment — punctual lighting consistent with visible frustum (Chapter 9 Part IV).

### Shadow passes

Directional shadow matrices use scene camera frustum — `worldToShadowClip` in UBO from shadow pass using same snapshot camera (Part V shadows book).

### 2D camera same UBO

Orthographic `viewProj` uploads identically — sprite shaders multiply same `ubo.viewProj`. `cameraPos` may be less critical for pure 2D lighting but still filled for consistency.

### End-to-end checklist

```text
□ aspect from framebuffer size this frame
□ PerspectiveVulkan / OrthographicVulkan used
□ viewProj = proj * view (not reversed)
□ cameraPositionWorld matches eye used to build view
□ SetSceneRenderParams before renderer record
□ Uniform writer runs for correct frame index
□ invViewProj invert succeeded (non-singular matrix)
```

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### viewProj on snapshot not UBO-only

**Concept:** SceneRenderParams

**Decision:** CPU snapshot carries matrix; writer copies

**Why:** Shadow/cluster CPU passes read same matrix before GPU upload.

**In Spark:** `SceneRenderParams.hpp`.

### cameraPos separate field

**Concept:** Shader view vector

**Decision:** Explicit Vector3, not extracted from view matrix

**Why:** View is inverse camera transform; eye in world is translation column of **camera world** matrix, not trivial from view.

**In Spark:** All demo fill paths set both.

### invViewProj at upload time

**Concept:** Sky rays

**Decision:** Invert once per frame in writer

**Why:** Avoid stale inverse if only viewProj cached; handle singular fail.

**In Spark:** `VulkanSceneUniformWriter.cpp`.

### viewportSize in same UBO

**Concept:** Sky NDC reconstruction

**Decision:** Extent packed with camera-related fields

**Why:** `gl_FragCoord` / drawable size needed with `invViewProj` for per-pixel rays.

**In Spark:** `scene.frag` sky block.

### One UBO for all scene draws

**Concept:** End-to-end

**Decision:** Single binding 0 per frame

**Why:** All meshes share one camera per frame — forward renderer baseline.

**In Spark:** `VulkanSceneDescriptors`.

## 4. Implementation Walkthrough

### Spark Files We Touch This Chapter

- `include/spark/engine/SceneRenderParams.hpp`
- `include/spark/render/scene/VulkanSceneUniformGpu.hpp`
- `src/spark/render/scene/VulkanSceneUniformWriter.cpp`
- `shaders/scene_ubo.glsl`
- `shaders/scene.vert`, `shaders/scene.frag`

### Step 1 — Log snapshot

Each frame log `viewProj.m[12..14]` translation part of composite — not eye position! Log `cameraPositionWorld` separately.

### Step 2 — RenderDoc UBO

Capture draw — binding 0 buffer hex matches CPU `SceneUniformGpu` after write.

### Step 3 — Break cameraPos

Set `cameraPositionWorld` to zero intentionally — specular and sky break; mesh still transforms (uses viewProj only).

### Step 4 — Break invViewProj

Upload identity inverse — sky swims; mesh OK.

### Step 5 — Compare manual vs ECS resolve

Same scene two ways — UBO bytes should match if cameras equivalent.

### Step 6 — Resize window

Resize swapchain — aspect, viewProj, viewportSize all update same frame.

## 5. How This Connects to the Rest of Your Engine

Part **V** ends at GPU boundary. Part **VI** skinned meshes use same `viewProj`. Part **IV** lighting uses `cameraPos`. Part **VIII** tonemap may use exposure from lighting resolve, not camera.

Camera mistakes surface in **every** shader using UBO — verify upload early with sky or normal debug.

## 6. Pitfalls and Debugging

- **Specular wrong, position OK** — `cameraPos` wrong but `viewProj` right.
- **Sky wrong, mesh OK** — `invViewProj` or `viewportSize` stale.
- **One frame behind** — snapshot from last frame's camera after resize.
- **Extracting eye from view matrix** — use world transform translation instead.
- **Double SetSceneRenderParams** — game overwrites after auto-submit.

## 7. Exercises

1. Add debug mode: visualize `length(cam - vWorldPos)` as heat in fragment shader.
2. Prove with math that view matrix translation column ≠ camera world position.
3. Trace `CharacterCameraDemo` from `ViewMatrix()` to UBO bytes in one diagram.
4. Measure cost of `TryInvert` 4×4 per frame — negligible?
5. List all Spark shaders reading `ubo.viewProj` or `ubo.cameraPos` (grep).

## Summary

Camera matrices flow from rigs or ECS resolution into `SceneRenderParams::viewProjection` and `cameraPositionWorld`, then `VulkanSceneUniformWriter` copies `viewProj`, inverts to `invViewProj`, and sets `cameraPos` in the scene UBO. Vertex shaders transform world positions; fragment shaders build view vectors and sky rays. Set both matrix and eye position every frame, use framebuffer aspect, and validate in RenderDoc — Part V input and camera work exists to feed this upload path.

---

Continue to [Part VI — Models and Animation](../part-06-models-animation/index.md).
