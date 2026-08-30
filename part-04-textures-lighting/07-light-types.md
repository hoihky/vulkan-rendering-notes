# Light Types

> **Build-Along Milestone:** You can describe how directional, point, and spot lights flow from ECS components to UBO vs SSBO and into `scene.frag`.

**You will learn:**

- Model directional sun light in the frame UBO.
- Pack point and spot lights into GPU SSBO layouts matching GLSL.
- Compute spot cone attenuation with inner/outer cosines.
- Understand range attenuation for point lights.

## 1. The Problem We Are Solving

Real scenes mix light sources:

| Type | Behavior | Real-world analog |
|------|----------|-------------------|
| **Directional** | Parallel rays, infinite distance | Sun/moon |
| **Point** | Omnidirectional from a position | Bulb, flame |
| **Spot** | Cone from a position + axis | Flashlight, stage light |

Each type needs a **CPU representation** (ECS → `SceneRenderParams`) and a **GPU representation** (UBO or SSBO fields). Mismatch between C++ struct layout and GLSL causes lights to drift, flicker, or vanish.

Spark uses **one directional** in the UBO and up to **256 point + 128 spot** lights in SSBO binding 4. This chapter defines both sides of that contract.

### Three light types in plain English

**Directional light (sun):**

- Rays are **parallel** everywhere in the scene.
- No position — only direction and color.
- Cheap: one calculation per pixel, stored in frame UBO.
- Shadows use orthographic projection from sun direction (Part IX).

**Point light (bulb):**

- Emits in **all directions** from a point in space.
- Gets dimmer with distance (attenuation).
- Stored in SSBO array — loop in fragment shader.

**Spot light (flashlight):**

- Like a point light but limited to a **cone** direction.
- Inner/outer cone angles control soft edge falloff.
- Also in SSBO array.

```text
Analogy:
  Directional = sun at infinity — same direction at your feet and rooftops
  Point       = candle in a room — brightness falls off with distance
  Spot        = stage spotlight — cone of light from one position
```

**ECS → GPU path:**

```text
DirectionalLightComponent on sun entity
  → SceneSubmit fills ubo.lightDir, lightColor, intensity

PointLightComponent / SpotLightComponent on lamps
  → SceneSubmit appends to params.pointLights / spotLights
  → VulkanSceneUniformWriter copies to SSBO binding 4
  → scene.frag loops punctual lights
```

## 1b. Concept Deep Dive — ECS Components to GPU Structs

### CPU snapshot structs mirror shader SSBO layout

```cpp
// include/spark/engine/SceneRenderParams.hpp
struct ScenePointLight {
    Vector3 positionWorld{};
    float range = 8.0F;
    Vector3 color{1.0F, 1.0F, 1.0F};
    float intensity = 1.0F;
    bool castsShadow = false;
};

struct SceneSpotLight {
    Vector3 positionWorld{};
    float range = 12.0F;
    Vector3 directionWorld{0.0F, 0.0F, -1.0F};
    float innerConeRadians = 0.55F;
    float outerConeRadians = 0.85F;
    Vector3 color{1.0F, 1.0F, 1.0F};
    float intensity = 3.0F;
    bool castsShadow = false;
};
```

Field order and padding must match `VulkanClusteredLightGpu` / GLSL std430 structs — reordering a member without updating both sides makes lights drift or disappear.

### ECS components on GameObjects

| Component | GPU destination | Key fields |
|-----------|-----------------|------------|
| `DirectionalLightComponent` | UBO `lightDir`, `lightColor` | direction, color, intensity |
| `PointLightComponent` | `params.pointLights[]` | color, intensity, range |
| `SpotLightComponent` | `params.spotLights[]` | cone angles, axis from transform |

Spot axis comes from the owner's world matrix: `-Z` local axis transformed to world space — rotating the lamp entity aims the cone.

### Capacity constants

`SceneRenderParams::MaxPointLights` (256) and `MaxSpotLights` (128) cap SSBO size. Submit silently stops adding lights when full — excess lamps in crowded scenes need artist culling or priority (future work).

### Attenuation on GPU

Point lights use range-based falloff in the fragment shader — `range` on the struct defines the cutoff radius. Spot lights add cosine cone tests using **full** cone angles in radians; inner/outer define a soft penumbra between hotspot and edge.

## 2. Core Concepts

### Directional light (UBO path)

No position — only direction and radiance. ECS `DirectionalLightComponent` on an entity:

```cpp
// Transform +Z in world = direction TOWARD the light source
Vector3 towardLight = worldMatrix.TransformVector(Vector3{0, 0, 1});
params.lightDirectionWorld = towardLight.Normalized();
params.lightColor = directional->GetColor();
params.lightIntensity = directional->GetIntensity();
```

GLSL mirror in `scene_ubo.glsl`:

```glsl
vec4 lightDir;   // xyz = toward light, w unused
vec4 lightColor; // rgb = color, w = intensity
```

Fragment shader:

```glsl
vec3 Ld = normalize(ubo.lightDir.xyz);
vec3 sunRad = ubo.lightColor.rgb * ubo.lightColor.w;
```

**First enabled directional wins** — `ApplyEcsDirectionalLight` stops after one match.

### Point light (SSBO path)

CPU struct:

```cpp
struct ScenePointLight {
    Vector3 positionWorld;
    float range;
    Vector3 color;
    float intensity;
    bool castsShadow;
};
```

GPU packed arrays in `ClusterLightsGpu`:

```glsl
vec4 pointPositionRange[256];   // xyz = position, w = range
vec4 pointColorIntensity[256];  // rgb = color, w = intensity
```

Shader loop:

```glsl
vec3 toL = clusterLights.pointPositionRange[i].xyz - vWorldPos;
float dist = length(toL);
vec3 Lp = toL / max(dist, 1e-4);
float att = sparkPointAttenuation(dist, range);
vec3 rad = color * intensity * att;
Lo += evalBRDF(N, V, Lp, base, met, rough, rad, 0.0);
```

### Point attenuation function

```glsl
float sparkPointAttenuation(float dist, float range) {
    float att = max(1.0 - dist / max(range, 1e-3), 0.0);
    return att * att;  // squared linear falloff
}
```

Smooth zero at `dist = range`. Not physically inverse-square — artist-friendly for game ranges.

### Spot light (SSBO path)

CPU struct:

```cpp
struct SceneSpotLight {
    Vector3 positionWorld;
    float range;
    Vector3 directionWorld;  // emission axis (world)
    float innerConeRadians;    // full cone angles
    float outerConeRadians;
    Vector3 color;
    float intensity;
    bool castsShadow;
};
```

GPU packs direction with **cos(half outer angle)**:

```glsl
vec4 spotPositionRange[128];
vec4 spotDirectionCosOuter[128];  // xyz = emission dir, w = cos(outer/2)
vec4 spotColorIntensity[128];
vec4 spotCosInner[128];           // x = cos(inner/2)
```

Angular falloff:

```glsl
float sparkSpotAngularAttenuation(vec3 toLightDir, vec3 spotEmissionDir,
                                  float cosOuter, float cosInner) {
    float cosTheta = dot(normalize(toLightDir), normalize(-spotEmissionDir));
    return smoothstep(cosOuter, cosInner, cosTheta);
}
```

Note: `toLightDir` is fragment → light; emission dir points **away** from fixture — hence `-spotEmissionDir`.

### Caps and limits

| Type | Max count | Overflow behavior |
|------|-----------|-------------------|
| Directional | 1 effective | Second ignored |
| Point | 256 | Submit should clamp |
| Spot | 128 | Submit should clamp |

`SceneRenderParams::MaxPointLights` / `MaxSpotLights` match shader constants — `static_assert` in `VulkanClusteredLightGpu.hpp`.

### Shadow flags per light

`castsShadow` on CPU drives shadow map allocation (Part V). Fragment side uses `push.shadowFlags` per mesh for receive, and punctual shadow SSBO for per-light shadow slots.

### ECS components

| Component | Maps to |
|-----------|---------|
| `DirectionalLightComponent` | UBO sun fields |
| `PointLightComponent` + transform | `ScenePointLight` |
| `SpotLightComponent` + transform | `SceneSpotLight` (local -Z = shine direction) |

Submit walks active objects and appends to `params.pointLights` / `params.spotLights` arrays.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Directional in UBO, punctual in SSBO

**Concept:** Directional vs point storage

**Decision:** Sun frame-global; local lights in std430 SSBO

**Why:** UBO size limited; 256+ lights need SSBO packing.

**In Spark:** Bindings 0 vs 4 in scene layout.

### Squared linear falloff

**Concept:** Point attenuation

**Decision:** `(1 - d/r)²` not `1/r²`

**Why:** Controllable range cutoff without infinite tail.

**In Spark:** `clustered_lights.glsl` `sparkPointAttenuation`.

### Full cone angles on CPU

**Concept:** Spot cones

**Decision:** Store radians on CPU; convert to cos(half) for GPU

**Why:** Artists think in cone degrees; GPU compares cosines efficiently.

**In Spark:** `VulkanClusteredForwardLights::BuildAndUpload` packing.

### First directional only

**Concept:** ECS directional

**Decision:** Single sun in `ApplyEcsDirectionalLight`

**Why:** One shadow cascade set; multiple suns rare in forward baseline.

**In Spark:** `SceneSubmitLighting.cpp`.

### intensity in vec4.w

**Concept:** Point/spot color packing

**Decision:** RGB color × scalar intensity

**Why:** Matches directional UBO pattern; HDR punctual lights.

**In Spark:** `pointColorIntensity[i].w`.

## 4. Implementation Walkthrough

### Spark Files We Touch This Chapter

- `include/spark/engine/SceneRenderParams.hpp`
- `src/spark/scene/SceneSubmit.cpp` (point/spot gather)
- `src/spark/scene/SceneSubmitLighting.cpp`
- `include/spark/render/scene/VulkanClusteredLightGpu.hpp`
- `shaders/clustered_lights.glsl`

### Step 1 — Sun only scene

One directional entity. Verify `lightDir` rotates with entity rotation (+Z toward sun).

### Step 2 — Add point light

Place point near mesh. Intensity 5, range 8. Falloff to zero at range boundary.

### Step 3 — Add spot light

Aim spot at mesh. Widen `outerConeRadians` — cone visible on floor. Narrow inner for hotspot.

### Step 4 — RenderDoc SSBO

Inspect binding 4 buffer — `numPointLights`, first `pointPositionRange` matches CPU.

### Step 5 — Exceed cap test

Spawn 300 point lights — confirm clamp at 256 without crash.

### Step 6 — Disable punctual

Set `ubo.clusterDepth.w = 0` on CPU — `accumulatePunctualPbr` early-outs (Chapter 9).

## 5. How This Connects to the Rest of Your Engine

Chapter **8** gathers ECS → `SceneRenderParams`. Chapter **9** uploads SSBO and builds cluster grid. Chapter **6** `evalBRDF` evaluates each punctual contribution. Part **V** adds shadow maps per `castsShadow`.

Light **types** define data layout; **gathering** defines who fills it each frame.

## 6. Pitfalls and Debugging

- **Spot points wrong way** — emission dir vs `toLightDir` sign in angular atten.
- **Cone in degrees vs radians** — Spark uses radians on CPU.
- **Inner > outer** — undefined falloff; validate on submit.
- **Light at mesh origin** — dist ≈ 0, normalize stable but attenuation maxed.
- **Wrong SSBO std430 padding** — spot arrays misaligned; use mirrored C++ struct.

## 7. Exercises

1. Draw debug gizmo for spot cone using `innerConeRadians` / `outerConeRadians`.
2. Compare inverse-square vs Spark squared linear falloff visually.
3. Document byte offset of `spotCosInner[0]` in `ClusterLightsGpu`.
4. Add second directional — confirm only first affects scene (current Spark).
5. Implement a `SceneAreaLight` stub — where would it live, UBO or SSBO?

## Summary

Spark's directional sun lives in the frame UBO (`lightDir`, `lightColor`). Point and spot lights pack into SSBO binding 4 with position, range, color, intensity, and spot cone cosines. Attenuation uses squared linear range for points and `smoothstep` on cosines for spots. ECS components feed `SceneRenderParams` arrays consumed by the clustered forward upload path.

---

Next: [Gathering Lights](08-gather-lights.md).
