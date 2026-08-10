# Gathering Lights

> **Build-Along Milestone:** Each frame, `SceneSubmit` fills `SceneRenderParams` with sun, punctual lights, and resolved profile settings before GPU upload.

**You will learn:**

- Trace ECS components to `SceneRenderParams` light fields.
- Apply `SceneLightingResolver` to merge profiles and overrides.
- Pack resolved lighting into the scene UBO via `VulkanSceneUniformWriter`.
- Order submit steps so textures and lights are ready for the same frame slot.

## 1. The Problem We Are Solving

Lights originate in gameplay — sun entity, torches, flashlights. The GPU reads **flat structs** in UBO/SSBO, not ECS queries. **Gathering** is the CPU pass that:

1. Walks the world each frame
2. Writes `SceneRenderParams`
3. Resolves lighting profiles and time-of-day
4. Hands off to Vulkan writers

If gathering runs after GPU record, or clears arrays mid-frame, lights pop or lag one frame. Spark centralizes this in **scene submit** — the same pipeline that builds draw lists and texture arrays.

## 2. Core Concepts

### SceneRenderParams as lighting snapshot

Key fields beyond draws:

```cpp
Vector3 lightDirectionWorld;
Vector3 lightColor;
float lightIntensity;
bool directionalShadowsEnabled;

Array<ScenePointLight> pointLights;
Array<SceneSpotLight> spotLights;

SceneLightingProfile lightingProfile;
bool useTimeOfDay;
float timeOfDay;
float exposure;
// shadow cascade overrides, ambient overrides...
```

Immutable for the GPU frame slot once submit completes — same contract as mesh draws.

### Submit pipeline order (simplified)

```text
FillStandardLitSceneFromWorld:
  1. Clear sceneTextures, pointLights, spotLights
  2. ApplyEcsDirectionalLight(world, params)
  3. ProcessTimeOfDayDrivers (if animating clock)
  4. SceneLightingResolver::Resolve(params) → ResolvedSceneLighting
  5. ForEachActiveGameObject → point lights, spot lights, meshes, materials
  6. FindOrAddSceneTexture per material
```

Order matters: directional first, then resolver may **override sun** from time-of-day, then punctual collection.

### ApplyEcsDirectionalLight

```cpp
world.ForEachActiveGameObject([&](GameObject* object) {
    const DirectionalLightComponent* directional = ...;
    if (!directional || !directional->IsEnabled()) return;
    params.lightDirectionWorld = worldMatrix.TransformVector({0,0,1}).Normalized();
    params.lightColor = directional->GetColor();
    params.lightIntensity = directional->GetIntensity();
    applied = true;  // first only
});
```

Gameplay places a rotated empty entity — +Z points toward sun.

### Punctual gather in SceneSubmit

For each active object with `PointLightComponent`:

```cpp
ScenePointLight gpu{};
gpu.positionWorld = worldMat.TranslationVector();
gpu.range = pl->GetRange();
gpu.color = pl->GetColor();
gpu.intensity = pl->GetIntensity();
gpu.castsShadow = pl->CastsShadow();
params.pointLights.PushBack(gpu);
```

Spot lights similarly read `SpotLightComponent` — direction from transform **-Z** in world space, cone angles from component.

### SceneLightingResolver

```cpp
ResolvedSceneLighting SceneLightingResolver::Resolve(SceneRenderParams& params) {
    ResolvedSceneLighting resolved = ResolveSceneLightingFromParams(
        params.lightingProfile, params.exposure, ...);

    if (resolved.timeOfDayApplied || params.useTimeOfDay) {
        ApplyTimeOfDayLighting(tod, profile, resolved,
            &params.lightDirectionWorld, &params.lightColor, &params.lightIntensity);
    }
  return resolved;
}
```

Merges:

- **Profile preset** (Outdoor, Interior, …)
- **Per-scene overrides** (exposure, cascade far, fade distance)
- **Time-of-day** analytic sun/ambient (Chapter 10)

Returns `ResolvedSceneLighting` for renderer — exposure, shadow flags, ambient colors, whether ToD ran.

### ResolvedSceneLighting contents

| Field | Used for |
|-------|----------|
| `exposure` | Tonemap (Part VIII) |
| `shadowCascadeNear/Far` | CSM + cluster depth range |
| `shadowDistanceFadeEnd` | Sun shadow fade in shader |
| `directionalShadowsEnabled` | Skip shadow passes |
| `ambient` | UBO `ambientColor/Sky/Probe` |
| `ambientScale` | Multiplier on hemisphere |
| `timeOfDayApplied` | Debug / UI |

### VulkanSceneUniformWriter

After resolve, GPU frame writes UBO:

```cpp
void VulkanSceneUniformWriter::Write(mappedUbo, scene, lighting, extent, shadowPass, frameIndex, ...) {
    memcpy(u.viewProj, scene.viewProjection.m, ...);
    u.lightDir[0..2] = normalized scene.lightDirectionWorld;
    u.lightColor[0..2] = scene.lightColor;
    u.lightColor[3] = scene.lightIntensity;
    u.ambientColor = lighting.ambient.groundColor * lighting.ambientScale;
    // shadow matrices, cluster grid metadata...
    memcpy(mappedUbo, &u, sizeof(u));
}
```

`static_assert` offsets — Chapter 4 Part III. Lighting gather must finish **before** this write.

### VulkanClusteredForwardLights::BuildAndUpload

Parallel path for SSBO binding 4–5:

- Copy `pointLights` / `spotLights` arrays into `ClusterLightsGpu`
- Build cluster grid (Chapter 9)
- Map SSBO, memcpy, ready for descriptor bind

Called from renderer draw frame after submit snapshot is frozen.

### Demos vs ECS

Some demos push lights manually into `params.pointLights` without ECS — same structs, same GPU path. Production uses ECS; demos teach the snapshot contract.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Clear lights each submit

**Concept:** Submit order

**Decision:** `pointLights.Clear()` at frame start

**Why:** Stale lights from destroyed entities otherwise persist.

**In Spark:** `SceneSubmit.cpp` beginning.

### Resolver after ECS directional

**Concept:** SceneLightingResolver

**Decision:** ECS sun first, ToD may replace direction/color

**Why:** Gameplay can place sun entity; profile animates day/night over it.

**In Spark:** Resolver call order in submit.

### Resolved struct side channel

**Concept:** ResolvedSceneLighting

**Decision:** Return resolved values separate from mutating params

**Why:** Renderer needs shadow tuning without re-parsing profile enum.

**In Spark:** `SceneLightingResolver.hpp`.

### Single writer for UBO lighting fields

**Concept:** VulkanSceneUniformWriter

**Decision:** All `lightDir` / ambient packing in one class

**Why:** Avoid duplicate UBO writes from shadow and scene passes.

**In Spark:** `VulkanSceneUniformWriter.cpp`.

### Shared snapshot for textures + lights

**Concept:** SceneRenderParams

**Decision:** One struct per frame for entire scene pass

**Why:** Texture upload and light SSBO read same frame index.

**In Spark:** `VulkanDeferredUploadBatch` + clustered lights.

## 4. Implementation Walkthrough

### Spark Files We Touch This Chapter

- `src/spark/scene/SceneSubmit.cpp`
- `src/spark/scene/SceneSubmitLighting.cpp`
- `include/spark/render/lighting/SceneLightingResolver.hpp`
- `src/spark/render/lighting/SceneLightingResolver.cpp`
- `src/spark/render/scene/VulkanSceneUniformWriter.cpp`
- `src/spark/render/scene/VulkanClusteredForwardLights.cpp`

### Step 1 — Log submit counts

Each frame log `pointLights.GetSize()`, `lightIntensity`, profile enum.

### Step 2 — Disable ECS sun

Remove directional component — defaults or ToD still may set sun via profile.

### Step 3 — Override exposure

Set `params.exposure` on snapshot — verify tonemap changes (Part VIII) while UBO writer receives resolved value.

### Step 4 — Break order experiment

Call resolver before ECS directional (wrong) — observe double sun or wrong ToD base.

### Step 5 — Frame delay test

Modify light position after submit but before GPU finish — should not affect current frame (proves snapshot isolation).

### Step 6 — RenderDoc UBO

Compare `lightDir` bytes to CPU `lightDirectionWorld` after submit.

## 5. How This Connects to the Rest of Your Engine

Chapter **7** defined light structs. Chapter **9** consumes gathered arrays for SSBO. Chapter **10** drives profile and ToD in resolver. Part **I–II** frame sync ensures submit completes before command buffer record.

Gathering is the **CPU front door** for all lighting data.

## 6. Pitfalls and Debugging

- **Lights one frame late** — writing params after snapshot copied.
- **Duplicate lights** — forgot `Clear()` on punctual arrays.
- **ToD ignores ECS sun** — resolver overwrites; intentional for Outdoor profile.
- **Ambient override** — `params.ambientColor` nonzero replaces profile ground color.
- **Invalid sceneParamsValid** — GPU skips UBO write and uploads.

## 7. Exercises

1. Trace one point light from `PointLightComponent` to SSBO byte offset.
2. List all fields `ResolveSceneLightingFromParams` reads from `SceneRenderParams`.
3. Add debug print when `pointLights.size() > 64` — when does perf drop?
4. Compare `TimeOfDayDemo` vs manual sun entity — who wins?
5. Diagram submit → upload → draw on a timeline with fences.

## Summary

Spark gathers lighting each frame in scene submit: ECS directional and punctual components fill `SceneRenderParams`, `SceneLightingResolver` merges profiles and time-of-day into `ResolvedSceneLighting`, and `VulkanSceneUniformWriter` plus `VulkanClusteredForwardLights` pack GPU resources. Clear punctual arrays every frame, resolve after ECS sun, and treat the snapshot as immutable for the in-flight GPU slot.

---

Next: [Clustered Forward](09-clustered-lights.md).
