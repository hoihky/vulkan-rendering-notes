# Material Submit

> **Build-Along Milestone:** Demo uses standard submit; draw and texture layer counts stable frame-to-frame.

**You will learn:**

- Call FillStandardLitSceneFromWorld to flatten ECS meshes and materials.
- Explain FindOrAddSceneTexture dedup and layer index assignment.
- Trace sky vs standard material paths in RigidDrawableSubmitSink.

## 1. The Problem We Are Solving

Components live in `GameWorld`; Vulkan records from **SceneRenderParams**. Submit is the once-per-frame bridge —
walk active entities, apply materials, register textures, sort draws, resolve IBL layer — immutable before GPU work.

Build along: switch a demo from manual drawList assembly to `FillStandardLitSceneFromWorld` and verify counts match.

## 2. Core Concepts

### Snapshot boundary

`FillStandardLitSceneFromWorld` clears prior frame data, gathers ECS drawables into `SceneDrawItem` arrays, registers textures, sorts, partitions opaque/transparent — **before** command recording.


### ApplyMaterialComponentToSceneDrawItem

Copies scalars (metallic, roughness, emissive, shading, toon, opacity) and resolves normal/ORM/emissive layers into `params.sceneTextures` via FindOrAddSceneTexture.


### Sky special case in submit sink

When `SkyComponent` is present, submit bypasses standard material apply for shadow flags and uses sky tint × material tint for equirect — still registers base color texture layer.


### ResolveIblEnvironmentLayer post-gather

After draws are collected, submit scans for first sky draw with texture to set IBL environment index.

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
FillStandardLitSceneFromWorld(world, context, viewProj, camPos,
    lightDir, lightColor, lightIntensity, ambient, …, params);
context.SetSceneRenderParams(params);
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

Replace hand-built drawList in a demo with `FillStandardLitSceneFromWorld` — scene should match prior visuals including materials and sky sort order.

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

FillStandardLitSceneFromWorld and ApplyMaterialComponentToSceneDrawItem flatten ECS materials into sorted draw items and deduplicated texture layers.

---

Next: [Material Showcase Demo](10-material-showcase.md).
