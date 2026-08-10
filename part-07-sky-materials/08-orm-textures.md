# ORM Textures

> **Build-Along Milestone:** ORM texture drives visible chrome vs matte regions on one mesh.

**You will learn:**

- Unpack occlusion, roughness, metallic from a packed ORM texture.
- Apply `metallicFactor` and `roughnessFactor` after sampling.
- Register `metallicRoughnessMapLayer` at submit and feed IBL specular roughness.

## 1. The Problem We Are Solving

Separate grayscale textures waste array slots and break glTF delivery. **ORM packing** stores three material scalars in one image — your shader must unpack channels, multiply by material factors, and feed PBR + IBL with per-pixel roughness.

Build along: bind an ORM map on `MaterialComponent` and tune **Q/E/R/F** keys in `MaterialShowcase3DDemo` to see metal vs roughness respond independently of albedo.

## 2. Core Concepts

### glTF ORM packing

One texture stores **O**cclusion (R), **R**oughness (G), **M**etallic (B) — glTF metallic-roughness workflow. Spark names the layer `metallicRoughnessMapLayer` on `SceneDrawItem`.

| Channel | glTF role | Spark usage |
|---------|-----------|-------------|
| R | Ambient occlusion | Multiplies indirect/IBL when `occlusionStrength > 0` |
| G | Roughness | Per-pixel roughness × `roughnessFactor` |
| B | Metallic | Per-pixel metallic × `metallicFactor` |

### Factor multiply order

```text
met  = clamp(orm.b * metallicFactor, 0, 1)   // when map bound
rough = clamp(orm.g * roughnessFactor, 0.04, 1)
```

Scalar `metallic` / `roughness` on the draw item apply when **no** map is bound. Factors still scale sampled channels when map is present.

### Occlusion on ambient

Sampled R modulates `occlusion` — multiplies indirect/ambient terms and IBL when `occlusionStrength > 0`. Setting strength to `0` skips AO read (returns `0.0` occlusion multiplier path). Direct sun is **not** multiplied by AO in Spark — caves may look bright in direct light; AO primarily darkens fill.

### Linear data

ORM channels are linear — no sRGB decode, same as normal maps. Author and export as linear PNG or EXR.

### IBL roughness coupling

`sparkPrefilterEnvironment` in `ibl.glsl` uses per-pixel `rough` from ORM — chrome (low G) gets sharp reflections; rubber (high G) blurs environment specular on metals.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### One texture() call for three scalars

**Concept:** Single ORM sample

**Decision:** One `vec3 orm` read per fragment

**Why:** Saves sampler slots vs three grayscale textures — matches glTF delivery.

**In Spark:** `scene.frag` `metallicRoughnessMapLayer` block

### One SharedPtr on MaterialComponent

**Concept:** `GetMetallicRoughnessTexture`

**Decision:** `SetMetallicRoughnessTexture` + `MeshDirty`

**Why:** Runtime swaps ORM atlas without replacing mesh.

**In Spark:** `MaterialComponent.hpp`

### clamp roughness floor 0.04

**Concept:** GGX singularity guard

**Decision:** `clamp(..., 0.04, 1.0)` after sample

**Why:** Prevents specular fireflies on perfectly smooth pixels.

**In Spark:** `scene.frag` rough clamp

### MaxSceneTextures includes ORM

**Concept:** Shared 16-layer budget

**Decision:** Dedup by pointer identity

**Why:** Many props sharing one ORM atlas must map to one layer index.

**In Spark:** `SceneSubmitMaterial.cpp` `FindOrAddSceneTexture`

## 4. Implementation Walkthrough

### Spark Files We Touch This Chapter

- `shaders/scene.frag`
- `shaders/ibl.glsl`
- `include/spark/ecs/components/rendering/MaterialComponent.hpp`
- `src/spark/scene/SceneSubmitMaterial.cpp`

### Step 1 — Read ORM unpack in shader

```glsl
// shaders/scene.frag (lines 436–447)
    float met = clamp(vMetallic * push.metallicFactor, 0.0, 1.0);
    float rough = clamp(vRoughness * push.roughnessFactor, 0.04, 1.0);
    float occlusion = push.occlusionStrength;
    if (push.metallicRoughnessMapLayer >= 0) {
        int mrl = clamp(push.metallicRoughnessMapLayer, 0, 15);
        vec3 orm = texture(sceneTextures, vec3(vTexCoord, float(mrl))).rgb;
        rough = clamp(orm.g * push.roughnessFactor, 0.04, 1.0);
        met = clamp(orm.b * push.metallicFactor, 0.0, 1.0);
        occlusion = push.occlusionStrength > 0.0
            ? clamp(orm.r * push.occlusionStrength, 0.08, 1.0)
            : 0.0;
    }
```

Note: vertex attributes `vMetallic` / `vRoughness` carry scalar defaults; ORM overrides when layer bound.

### Step 2 — Assign ORM on material

```cpp
mat->SetMetallicRoughnessTexture(ormTex);
mat->SetMetallicFactor(1.0f);
mat->SetRoughnessFactor(1.0f);
mat->SetOcclusionStrength(1.0f);
```

### Step 3 — Trace layer registration

```cpp
// src/spark/scene/SceneSubmitMaterial.cpp
    if (mat->GetMetallicRoughnessTexture()) {
        item.metallicRoughnessMapLayer =
                SceneSubmitDetail::FindOrAddSceneTexture(*resolveTextures, mat->GetMetallicRoughnessTexture());
    }
```

Two-argument `ApplyMaterialComponentToSceneDrawItem` without `&params` leaves all map layers at `−1`.

### Step 4 — Verify per-pixel roughness on IBL

High roughness in green channel should blur specular IBL on metals — rotate roughness factor in MaterialShowcase (**R/F** keys) while ORM map stays bound.

### Step 5 — Paint debug channels

Temporarily output channels as color:

```glsl
// outColor = vec4(orm, 1.0); return;  // after ORM sample
```

Confirm R darkens crevices in ambient-only views, G controls gloss width, B toggles metal response.

### Step 6 — Milestone — ORM-driven chrome vs rubber

One ORM texture on two UV regions: B≈1, G≈0 reads chrome; B≈0, G≈1 reads rubber — same albedo, different ORM.

## 5. How This Connects to the Rest of Your Engine

**MaterialComponent** holds the ORM `SharedPtr`. **IBL Basics** specular prefilter uses per-pixel roughness from ORM when bound. **Normal Mapping** runs in the same fetch preamble. **Material Submit** registers the layer once per unique texture pointer.

## 6. Pitfalls and Debugging

- **Swapped G and B** — dielectrics look metallic; verify glTF channel order.
- **sRGB ORM upload** — wrong roughness response; use linear UNORM.
- **AO on direct sun only** — Spark applies AO primarily on ambient/IBL path.
- **metallicFactor 0 with B=1 in map** — still zero metal if factor zero — understand multiply order.
- **ORM layer −1 but expecting map** — forgot `&params` overload at submit.

## 7. Exercises

1. Visualize each ORM channel alone as grayscale.
2. Set `roughnessFactor 0.5` with flat G=1 — explain resulting roughness 0.5.
3. Count unique `sceneTextures` layers with and without ORM dedup on repeated props.
4. Paint a gradient in G only — observe IBL blur change along sphere.
5. Set `occlusionStrength = 0` — confirm R channel ignored.

## Summary

ORM packs AO, roughness, metallic in RGB; `scene.frag` samples once, applies factors, and feeds PBR punctual lights and IBL specular roughness.

---

Next: [Material Submit](09-material-submit.md).
