# Material Submit

> **Build-Along Milestone:** Demo uses standard submit; draw and texture layer counts stable frame-to-frame.

**Spark API note:** Production demos call **`world.Rendering().SubmitLit`** with **`LitSceneSubmitOptions`** (Part II Chapter 11). The legacy free function **`FillStandardLitSceneFromWorld`** still exists and delegates to the same **`LitSceneSubmitter`** gather — snippets below use that name when tracing internal steps.

**You will learn:**

- Call lit scene submit to flatten ECS meshes and materials (`world.Rendering().SubmitLit` in demos).
- Explain FindOrAddSceneTexture dedup and layer index assignment.
- Trace sky vs standard material paths in RigidDrawableSubmitSink.

## 1. The Problem We Are Solving

Components live in `GameWorld`; Vulkan records from **SceneRenderParams**. Submit is the once-per-frame bridge —
walk active entities, apply materials, register textures, sort draws, resolve IBL layer — immutable before GPU work.

Build along: switch a demo from manual drawList assembly to `world.Rendering().SubmitLit` and verify counts match.

### What is "submit"? (the bridge every beginner must understand)

Your game stores objects in an **Entity Component System** (`GameWorld`, `GameObject`, components). Vulkan has never heard of `GameObject`. Vulkan only understands:

- Command buffers with `vkCmdDrawIndexed`
- Bound pipelines, descriptor sets, vertex buffers
- Push constants with matrices and material indices

**Submit** is the translation layer. Once per frame, Spark walks the ECS and builds a flat, GPU-friendly struct: `SceneRenderParams`.

```text
GameWorld (ECS)  ──submit──►  SceneRenderParams  ──record──►  Vulkan commands
   OOP, components              plain structs, arrays           vkCmd*
```

Rules every engine should follow:

1. **Submit runs on CPU before GPU record** — never call `vkCmd*` from `OnUpdate`.
2. **SceneRenderParams is a snapshot** — copy or build fresh each frame; do not mutate while GPU reads last frame's data.
3. **No Vulkan types in SceneRenderParams** — only `SceneDrawItem`, texture layer ints, matrices.

`world.Rendering().SubmitLit` is the main entry point demos call after filling the camera (legacy `FillStandardLitSceneFromWorld` forwards to the same code):

```cpp
// Typical demo OnRender pattern (concept)
LitSceneSubmitOptions opts{};
opts.WithDirectionalLight(sunDir, sunColor, sunIntensity).WithAmbient(ambientColor);
world.Rendering().SubmitLit(context, viewProjection, cameraPositionWorld, opts);
// SubmitLit calls SetSceneRenderParams internally — no manual params publish needed
```

After this call, `params.draws` contains every opaque mesh, `params.transparentDraws` contains sorted transparent meshes, and `params.sceneTextures` lists unique textures with assigned layers.

### Inside FillStandardLitSceneFromWorld (conceptual steps)

1. **Clear** last frame's draw arrays and texture registry.
2. **Resolve lighting** — sun direction, ambient, exposure from components and volumes.
3. **Frustum cull** (optional) — skip off-screen objects.
4. **For each visible `MeshComponent`:**
   - Read transform → world matrix.
   - Read `MaterialComponent` or `MultiMaterialComponent`.
   - Call `PushRigidMeshDraws` (one or many `SceneDrawItem`s).
   - Register textures via `FindOrAddSceneTexture`.
5. **Repeat** for skinned meshes, sky, sprites, etc.
6. **Sort** draws (sky first, opaque by material, transparent back-to-front).
7. **Partition** opaque vs transparent arrays.

You do not call these steps manually in gameplay code — one function does the walk. Your job when adding a new drawable type is to extend the sink that handles it, not to bypass submit.

## 1b. Concept Deep Dive — FindOrAddSceneTexture and Sky Submit

### Texture deduplication by pointer identity

```cpp
// src/spark/scene/submit/SceneSubmitMaterial.cpp (lines 68–124)
std::int32_t FindOrAddSceneTexture(SceneRenderParams& params, const SharedPtr<Texture2D>& tex, ...) {
    if (!tex) return -1;
    SharedPtr<Texture2D> resolved = tex->ResolveAtlasUv(uvScale, uvOffset);

    const bool isHdr = resolved->IsHdrFloatPixels();
    Array<SharedPtr<Texture2D>>& targetArray =
            isHdr ? params.sceneHdrTextures : params.sceneTextures;
    const uint32_t maxLayers = isHdr ? MaxSceneHdrTextures : MaxSceneTextures;  // 8 / 64

    for (size_t i = 0; i < targetArray.GetSize(); ++i) {
        if (targetArray[i].Get() == resolved.Get())
            return static_cast<std::int32_t>(i);  // dedupe hit
    }
    if (targetArray.GetSize() >= maxLayers) {
        std::fprintf(stderr, "Spark: scene texture limit reached\n");
        return -1;
    }
    const std::int32_t layer = static_cast<std::int32_t>(targetArray.GetSize());
    targetArray.PushBack(resolved);
    return layer;
}
```

Same `SharedPtr` address → same layer index across all draws that frame. HDR float pixels route to `sceneHdrTextures` (binding 11); LDR sRGB to `sceneTextures` (binding 1).

### PopulateSkyDrawItem — separate from standard material path

```cpp
// src/spark/scene/submit/SceneSubmitMaterial.cpp (concept)
void PopulateSkyDrawItem(SceneDrawItem& item, const SkyComponent& sky,
        const MaterialComponent* mat, const Matrix4& world, SceneRenderParams& params) {
    item.skyMode = sky.GetSkyMode();
    item.albedo = sky.GetTint();
    item.model = world;
    item.mesh = SceneMeshSlot::Custom;
    if (mat && mat->GetBaseColorTexture()) {
        item.textureLayer = FindOrAddSceneTexture(params, mat->GetBaseColorTexture(), ...);
        item.textureIsHdrLinear = isHdr;
    }
}
```

Sky tint multiplies equirect sample in fragment shader; no metallic/roughness on sky draws.

### FillStandardLitSceneFromWorld orchestration

```cpp
// src/spark/scene/submit/SceneSubmit.cpp (concept)
sceneForCulling->ForEachSky([&](..., const SkyComponent& sk, ...) {
    SceneDrawItem item{};
    PopulateSkyDrawItem(item, sk, mc, mat, world, params);
    drawList.PushBack(item);
});
// ... rigid meshes, skinned meshes ...
ResolveIblEnvironmentLayer(params);  // after all draws collected
```

### Push constants map layers to shader

```glsl
// shaders/scene_push.glsl (concept)
layout(push_constant) uniform Push {
    int textureLayer;              // base color / sky equirect
    int normalMapLayer;
    int metallicRoughnessMapLayer;
    int emissiveMapLayer;
    int albedoHdrLinear;           // route sample to sceneHdrTextures
    // ...
} push;
```

## 2. Core Concepts

### Snapshot boundary

`FillStandardLitSceneFromWorld` clears prior frame data, gathers ECS drawables into `SceneDrawItem` arrays, registers textures, sorts, partitions opaque/transparent — **before** command recording.


### ApplyMaterialComponentToSceneDrawItem

Copies scalars (metallic, roughness, emissive, shading, toon, opacity) and resolves normal/ORM/emissive layers into `params.sceneTextures` via FindOrAddSceneTexture.


### Sky special case in submit sink

When `SkyComponent` is present, submit bypasses standard material apply for shadow flags and uses sky tint × material tint for equirect — still registers base color texture layer.


### ResolveIblEnvironmentLayer post-gather

After draws are collected, submit scans for first sky draw with texture to set IBL environment index.

### Multi-material submit: one entity, many draws

`PushRigidMeshDraws` is the function that connects `MultiMaterialComponent` + `MeshSubmesh` to flat `SceneDrawItem` arrays:

```cpp
// src/spark/scene/submit/SceneSubmit.cpp (lines 77–115)
void PushRigidMeshDraws(
        Array<SceneDrawItem>& drawList,
        SceneDrawItem baseItem,
        const Mesh& mesh,
        const MaterialComponent* mat,
        const MultiMaterialComponent* multiMat,
        SceneRenderParams& params,
        const std::function<std::int32_t(...)>& findOrAddTexture) {
    const Array<MeshSubmesh>& submeshes = mesh.GetSubmeshes();
    if (submeshes.IsEmpty() || multiMat == nullptr) {
        // single draw — full index buffer or MaterialComponent only
        // ...
        return;
    }

    for (std::size_t si = 0; si < submeshes.GetSize(); ++si) {
        SceneDrawItem item = baseItem;
        item.submeshIndex = static_cast<std::uint32_t>(si);
        const MeshSubmesh& sm = submeshes[si];
        const std::uint32_t materialIndex = sm.ResolveMaterialIndex(multiMat->GetActiveVariantIndex());
        if (materialIndex < multiMat->GetSlotCount()) {
            const MultiMaterialComponent::Slot& slot = multiMat->GetSlot(materialIndex);
            ApplyMultiMaterialSlotToSceneDrawItem(item, slot, &params);
            ApplyAlbedoTexture(item, slot.baseColor, slot.tint, findOrAddTexture);
        }
        drawList.PushBack(item);
    }
}
```

`ApplyMultiMaterialSlotToSceneDrawItem` copies metallic, roughness, emissive, shading model, glTF extension factors, and registers normal/ORM/emissive maps into `params.sceneTextures` via `FindOrAddSceneTexture`. Each unique `SharedPtr<Texture2D>` gets a stable **layer index** in the `sampler2DArray` uploaded before draws.

### GPU record: push constants carry material state

The Vulkan pass does not rebind descriptors per material. `VulkanSceneOpaquePass::Record` writes one `ModelPushConstants` struct per draw:

```cpp
// src/spark/render/scene/VulkanSceneOpaquePass.cpp (lines 149–183)
        push.textureLayer = d.textureLayer;
        push.metallic = d.metallic;
        push.roughness = d.roughness;
        push.normalMapLayer = d.normalMapLayer;
        push.metallicRoughnessMapLayer = d.metallicRoughnessMapLayer;
        push.emissiveMapLayer = d.emissiveMapLayer;
        FillMaterialExtensionPush(push, d.gltfExtensions);
        // ...
        vkCmdPushConstants(commandBuffer, layout, VK_SHADER_STAGE_VERTEX_BIT | VK_SHADER_STAGE_FRAGMENT_BIT,
                           0, sizeof(push), &push);
        vkCmdDrawIndexed(commandBuffer, indexCount, 1, firstIndex, vertexOffset, 0);
```

`firstIndex` and `indexCount` come from `VulkanCustomMeshPool::ResolveRigidDrawSlice`, which uses `item.submeshIndex` to select the `MeshSubmesh` range inside the packed custom index buffer.

**Material pipeline summary:**

```text
ECS MaterialComponent / MultiMaterialComponent (CPU textures)
  → SceneSubmit (layer indices + PBR scalars in SceneDrawItem)
  → VulkanSceneTextureUploader (RGBA8 / RGBA32F → GPU array layers)
  → VulkanSceneDescriptors binding 1 + 11
  → Push constants per draw
  → scene.frag GGX + ORM + normal mapping
  → HDR RGBA16F attachment (Part X)
```

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Single gather entry point

**Concept:** FillStandardLitSceneFromWorld

**Decision:** Single gather entry point

**Why:** Demos call one function after camera fill — avoids duplicating ECS walks.

**In Spark:** `src/spark/scene/SceneSubmit.cpp`

### Pointer equality on SharedPtr

**Concept:** FindOrAddSceneTexture dedup

**Decision:** Pointer equality on SharedPtr

**Why:** Stable layer indices when same texture used on many draws; respects MaxSceneTextures cap.

**In Spark:** `SceneSubmitMaterial.cpp`

### Sky first, transparent split

**Concept:** StableSortDrawItems then partition

**Decision:** Sky first, transparent split

**Why:** Opaque pass gets sorted draws; transparent copied to separate array with depth sort.

**In Spark:** `SceneSubmitDrawPartition.cpp`

### Callback per visible mesh

**Concept:** RigidDrawableSubmitSink

**Decision:** Callback per visible mesh

**Why:** Frustum cull reuses same sink interface — material apply identical for culled path.

**In Spark:** `SceneSubmit.cpp` RigidDrawableSubmitSink

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `src/spark/scene/SceneSubmit.cpp`
- `src/spark/scene/SceneSubmitMaterial.cpp`
- `include/spark/scene/SceneSubmit.hpp`

### Step 1 — Open ApplyMaterialComponentToSceneDrawItem

*Scalar copy + texture layer registration for normal/ORM/emissive.*

```cpp
// src/spark/scene/SceneSubmitMaterial.cpp (lines 14–95)
void ApplyMaterialComponentToSceneDrawItemImpl(SceneDrawItem& item, const MaterialComponent* mat) noexcept {
    if (mat == nullptr) {
        return;
    }
    item.metallic = mat->GetMetallic();
    item.roughness = mat->GetRoughness();
    item.metallicFactor = mat->GetMetallicFactor();
    item.roughnessFactor = mat->GetRoughnessFactor();
    item.occlusionStrength = mat->GetOcclusionStrength();
    item.emissiveColor = mat->GetEmissiveColor();
    item.emissiveIntensity = mat->GetEmissiveIntensity();
    item.emissiveFactor = mat->GetEmissiveFactor();
    item.shadingModel = mat->GetShadingModel();
    item.toonDiffuseBands = mat->GetToonDiffuseBands();
    item.toonRimIntensity = mat->GetToonRimIntensity();
    item.toonRimPower = mat->GetToonRimPower();
    item.doubleSided = mat->IsDoubleSided();
    item.opacity = mat->GetOpacity();
}

}  // namespace

namespace SceneSubmitDetail {

std::int32_t FindOrAddSceneTexture(SceneRenderParams& params, const SharedPtr<Texture2D>& tex) {
    if (!tex) {
        return -1;
    }
    for (std::size_t i = 0; i < params.sceneTextures.GetSize(); ++i) {
        if (params.sceneTextures[i].Get() == tex.Get()) {
            return static_cast<std::int32_t>(i);
        }
    }
    if (params.sceneTextures.GetSize() >= SceneRenderParams::MaxSceneTextures) {
        std::fprintf(
                stderr,
                "Spark: scene texture limit (%u) reached; dropping \"%s\"\n",
                SceneRenderParams::MaxSceneTextures,
                tex->GetName().CStr());
        return -1;
    }
    params.sceneTextures.PushBack(tex);
    return static_cast<std::int32_t>(params.sceneTextures.GetSize() - 1U);
}

void ResolveIblEnvironmentLayer(SceneRenderParams& params) noexcept {
    if (params.iblEnvironmentLayer >= 0) {
        return;
    }
    for (std::size_t i = 0; i < params.draws.GetSize(); ++i) {
        const SceneDrawItem& d = params.draws[i];
        if (d.skyMode != SceneSkyMode::None && d.textureLayer >= 0) {
            params.iblEnvironmentLayer = d.textureLayer;
            return;
        }
    }
}

}  // namespace SceneSubmitDetail

void ApplyMaterialComponentToSceneDrawItem(
        SceneDrawItem& item,
        const MaterialComponent* mat,
        SceneRenderParams* resolveTextures) noexcept {
    ApplyMaterialComponentToSceneDrawItemImpl(item, mat);
    if (mat == nullptr || resolveTextures == nullptr) {
        return;
    }
    item.normalMapLayer = -1;
    item.metallicRoughnessMapLayer = -1;
    item.emissiveMapLayer = -1;
    if (mat->GetNormalTexture()) {
        item.normalMapLayer = SceneSubmitDetail::FindOrAddSceneTexture(*resolveTextures, mat->GetNormalTexture());
    }
    if (mat->GetMetallicRoughnessTexture()) {
        item.metallicRoughnessMapLayer =
                SceneSubmitDetail::FindOrAddSceneTexture(*resolveTextures, mat->GetMetallicRoughnessTexture());
    }
    if (mat->GetEmissiveTexture()) {
        item.emissiveMapLayer = SceneSubmitDetail::FindOrAddSceneTexture(*resolveTextures, mat->GetEmissiveTexture());
    }
}
```

### Step 2 — Trace mesh + material gather

*RigidDrawableSubmitSink — sky branch vs standard material pairing.*

```cpp
// src/spark/scene/SceneSubmit.cpp (lines 64–108)
    void OnDrawable(
            GameObject* o,
            const MeshComponent& mc,
            const MaterialComponent* mat,
            const Matrix4& worldM) override {
        if (o == nullptr || !mc.GetMesh()) {
            return;
        }
        const SkyComponent* sky = o->GetComponent<SkyComponent>();
        if (sky != nullptr && sky->IsSkyEnabled()) {
            SceneDrawItem item{};
            item.mesh = SceneMeshSlot::Custom;
            item.skyMode = sky->GetSkyMode();
            item.model = worldM;
            item.customMesh = mc.GetMesh();
            item.albedo = sky->GetTint();
            item.textureLayer = -1;
            item.metallic = 0.0F;
            item.roughness = 1.0F;
            if (mat != nullptr && mat->GetBaseColorTexture()) {
                const Vector3& t = mat->GetTint();
                item.albedo = {item.albedo.x * t.x, item.albedo.y * t.y, item.albedo.z * t.z};
                item.textureLayer = findOrAddTexture(mat->GetBaseColorTexture());
            }
            item.shadowFlags = 0;
            drawList.PushBack(item);
            return;
        }
        SceneDrawItem item{};
        item.model = worldM;
        item.mesh = mc.GetSlot();
        if (mc.GetSlot() == SceneMeshSlot::Custom) {
            item.customMesh = mc.GetMesh();
        }
        Vector3 alb = mc.GetAlbedo();
        item.textureLayer = -1;
        if (mat != nullptr) {
            ApplyMaterialComponentToSceneDrawItem(item, mat, &params);
            if (mat->GetBaseColorTexture()) {
                const Vector3& t = mat->GetTint();
                alb = {alb.x * t.x, alb.y * t.y, alb.z * t.z};
                item.textureLayer = findOrAddTexture(mat->GetBaseColorTexture());
            }
        }
        item.albedo = alb;
```

### Step 3 — Wire demo OnRender

```cpp
LitSceneSubmitOptions opts{};
opts.WithDirectionalLight(lightDir, lightColor, lightIntensity).WithAmbient(ambient);
world.Rendering().SubmitLit(context, viewProj, camPos, opts);
// Or world.Rendering().FillLit(...) if you patch SceneRenderParams before record
```

Renderer reads snapshot only — no GameWorld access during record.

### Step 4 — Log texture dedup

```cpp
// After fill: params.sceneTextures.GetSize() vs count of unique SharedPtrs
```

Two cubes sharing brick texture should produce **one** layer index.

### Step 5 — Hide entity test

`SetActive(false)` on meshed entity — draw count drops next frame; texture count may stay if sky still references same atlas.

### Step 6 — Milestone — full gather without custom demo loop

Replace hand-built drawList in a demo with `world.Rendering().SubmitLit` — scene should match prior visuals including materials and sky sort order.

## 5. How This Connects to the Rest of Your Engine

Part **IV** renderer consumes `params.draws` and `sceneTextures`. **IBL Basics** uses ResolveIblEnvironmentLayer
at end of fill. **Material Showcase Demo** is the visual regression target for this gather path.

## 6. Pitfalls and Debugging

- **Forgetting Clear() on draws/textures** — duplicate geometry every frame.",
- **Submit after record started** — race on snapshot; gather before SetSceneRenderParams only once per frame.",
- **Texture limit exceeded** — FindOrAddSceneTexture returns −1 and logs; drops maps silently.",
- **Material on inactive entity** — skipped by ForEachActiveGameObject — expected, not a leak.

## 7. Exercises

1. Log `params.draws` size vs `MeshComponent` count — explain gaps (cull, null mesh, sky).
2. Share one ORM texture across five props — confirm one layer index in RenderDoc.
3. Diff `SceneDrawItem` fields before/after `ApplyMaterialComponentToSceneDrawItem`.
4. Call two-arg apply without `&params` — confirm map layers stay `−1`.
5. Trace `ResolveIblEnvironmentLayer` when sky has no texture bound.

## Summary

`LitSceneSubmitter` (via `world.Rendering().SubmitLit` or legacy `FillStandardLitSceneFromWorld`) and `ApplyMaterialComponentToSceneDrawItem` flatten ECS materials into sorted draw items and deduplicated texture layers.

---

Next: [Material Showcase Demo](10-material-showcase.md).
