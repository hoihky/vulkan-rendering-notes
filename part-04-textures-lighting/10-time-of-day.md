# Lighting Profiles

> **Build-Along Milestone:** `SceneLightingProfile::Outdoor` with `useTimeOfDay` drives sun direction, color, intensity, and hemisphere ambient through `ApplyTimeOfDayLighting`.

**You will learn:**

- Select among Default, Outdoor, Interior, and NightInterior presets.
- Resolve profile settings with per-scene overrides via `SceneLightingResolver`.
- Animate normalized time-of-day from midnight to noon to sunset.
- Pack ambient hemisphere and exposure into the scene UBO.

## 1. The Problem We Are Solving

Hard-coding one sun direction and gray ambient works for a single demo. Games need **context**:

- Outdoor vistas with long shadow fade and cool sky fill
- Interior combat spaces with tighter cascades and bright probe fill
- Night dungeons with dim sun and strong ambient probe
- **Time of day** — sunrise orange, noon white, sunset red, night blue

Spark encodes these as **`SceneLightingProfile` presets** plus optional **`useTimeOfDay`** animation. `SceneLightingResolver` merges preset, overrides on `SceneRenderParams`, and analytic sun/ambient curves into `ResolvedSceneLighting` and mutates sun fields on the snapshot.

This chapter closes Part IV — you can now texture meshes and light them with profile-aware forward shading.

### Time of day in plain language (beginner)

Games often fake a 24-hour cycle without simulating astronomy. **Time of day** is usually:

1. A **normalized clock** `t ∈ [0,1]` (0 = midnight, 0.25 = sunrise, 0.5 = noon).
2. A function that maps `t` → **sun direction**, **sun color**, **ambient color**, **fog density**.

```text
t = 0.35 (morning)
  → sunDir = lerp(eastHorizon, zenith, curve)
  → sunColor = warm orange
  → ambient = cool blue-gray (fill from sky)
  → ApplyTimeOfDayLighting writes SceneRenderParams sun fields
```

**No new shader needed** — same directional light UBO, different values each frame.

**Lighting profiles** bundle defaults:

| Profile | Typical use |
|---------|-------------|
| `Outdoor` | Sun + time-of-day, wide shadow fade |
| `Interior` | Dim sun, stronger fill ambient |
| `NightInterior` | Moon-tinted ambient, low exposure |

`SceneLightingResolver` merges preset + per-scene overrides so one outdoor level and one dungeon level share the same `scene.frag` but different resolved sun/ambient each frame.

**Shadows follow sun:** when `lightDir` rotates, shadow map ortho frustum must update (Part IX) — otherwise shadows slide incorrectly at dusk.

## 1b. Concept Deep Dive — SceneLightingProfile and ApplyTimeOfDayLighting

### Preset enum drives defaults

```cpp
// include/spark/render/lighting/SceneLightingProfile.hpp
enum class SceneLightingProfile : std::uint8_t {
    Default = 0,
    Outdoor = 1,        // sun arc, long shadow fade, cool sky fill
    Interior = 2,       // tighter cascades, brighter probe
    NightInterior = 3,  // dim moon sun, strong ambient probe
};
```

`LightingProfileSettingsFor(profile)` returns exposure, cascade near/far, ambient hemisphere colors, and whether time-of-day is enabled on the preset row.

### Analytic sun from normalized clock

`ApplyTimeOfDayLighting` maps `normalizedTime ∈ [0,1]` to sun direction, color, and intensity:

```cpp
// src/spark/render/lighting/SceneLightingProfile.cpp (concept)
void ApplyTimeOfDayLighting(
        const float normalizedTime,
        const SceneLightingProfile profile,
        ResolvedSceneLighting& out,
        Vector3* outSunDirectionWorld,
        Vector3* outSunColor,
        float* outSunIntensity) noexcept;
```

Outdoor profiles rotate azimuth with `t`; elevation uses `SunElevation01(t)` — warm twilight tints blend between night blue and noon white. Night interior caps sun intensity near 0.08–0.4 while boosting probe ambient.

### SceneSubmit integrates time drivers

Inside `FillStandardLitSceneFromWorld`, `ProcessTimeOfDayDrivers(world, dt)` advances `SceneTimeOfDayComponent` clocks before `SceneLightingResolver::Resolve`. `TimeOfDayDemo` sets `params.lightingProfile = SceneLightingProfile::Outdoor` and watches sun color shift without shader changes.

### Resolved lighting mutates the snapshot

Resolver output adjusts `params.lightDirectionWorld`, `lightColor`, `lightIntensity`, and ambient hemisphere fields on `SceneRenderParams` — the same UBO path as manual sun setup. Shadow cascade far/near and distance fade come from `ResolvedSceneLighting` into `VulkanSceneUniformWriter` and the directional shadow pass.

## 2. Core Concepts

### Profile enum

```cpp
enum class SceneLightingProfile : uint8_t {
    Default = 0,
    Outdoor = 1,
    Interior = 2,
    NightInterior = 3,
};
```

Set on snapshot:

```cpp
params.lightingProfile = SceneLightingProfile::Outdoor;
params.useTimeOfDay = true;
params.timeOfDay = 0.42f;  // normalized phase
```

### Preset comparison table

| Profile | Cascade far | Shadow fade | Ambient character | ToD default |
|---------|-------------|-------------|-------------------|-------------|
| **Default** | 400 m | Off | Moderate ground/sky | Off |
| **Outdoor** | 400 m | 150 m end | Cool sky, warm ground | **On** (0.42) |
| **Interior** | 75 m | 45 m end | Brighter probe fill | Off |
| **NightInterior** | 60 m | 38 m end | Dim + strong probe | **On** (0.04 night) |

`LightingProfileSettingsFor(profile)` returns full `SceneLightingProfileSettings` row — exposure, `shadowCascadeNear/Far`, `ambientScale`, shadow flags, `SceneTimeOfDaySettings`.

### SceneLightingProfileSettings structure

```cpp
struct SceneLightingProfileSettings {
    float exposure;
    float shadowCascadeNear;
    float shadowCascadeFar;
    float ambientScale;
    SceneShadowSettings shadow;
    SceneAmbientSettings ambient;
    SceneTimeOfDaySettings timeOfDay;
};
```

Shadow settings include directional/punctual enable, distance fade end, cascade blend fraction, default cast/receive on draws.

### Ambient hemisphere model

```cpp
struct SceneAmbientSettings {
    Vector3 groundColor;
    Vector3 skyColor;
    Vector3 probeColor;
    float probeWeight;
};
```

Shader mixes by surface normal up-component:

```glsl
float hemi = clamp(N.y * 0.5 + 0.5, 0.0, 1.0);
vec3 ambTint = mix(ambGround, ambSkyBase, hemi);
```

Probe adds uniform fill for interiors (`ambientProbe` in UBO).

### Normalized time-of-day

```cpp
struct SceneTimeOfDaySettings {
    bool enabled;
    float normalizedTime;  // 0=midnight, 0.25=sunrise, 0.5=noon, 0.75=sunset
};
```

`SunElevation01(t) = saturate(sin((t - 0.25) * 2π))` — zero at midnight, peak at noon.

### ApplyTimeOfDayLighting

Analytic curves (no texture sky required):

```cpp
void ApplyTimeOfDayLighting(float normalizedTime, SceneLightingProfile profile,
    ResolvedSceneLighting& resolved,
    Vector3* outSunDir, Vector3* outSunColor, float* outSunIntensity) {
    const float elev = SunElevation01(t);
  // azimuth rotates with t for Outdoor
    Vector3 sunDir{ sinAz * cosEl, sinEl, cosAz * cosEl };
    Vector3 sunColor = Lerp3(nightColor, noonColor, elev);
    // twilight warm band near horizon
    float sunIntensity = base + elev * scale;
    // update hemisphere ambient colors and exposure in resolved
    if (outSunDir) *outSunDir = sunDir;
    // ...
}
```

Mutates **resolved** ambient and optionally **params** sun direction/color/intensity when `useTimeOfDay` is true.

### Resolver merge order

```text
1. Load preset from params.lightingProfile
2. Apply numeric overrides on params (exposure, cascade far, fade distance, ...)
3. If useTimeOfDay or preset.timeOfDay.enabled → ApplyTimeOfDayLighting
4. If params.ambientColor manually set → override ground ambient
5. Return ResolvedSceneLighting to renderer
```

ECS directional sun may be **replaced** by ToD sun when profile enables it — know your scene setup.

### TimeOfDayDriverComponent

Gameplay can advance clock:

```cpp
// ProcessTimeOfDayDrivers in submit
timeDriver->Advance(deltaSeconds);
params.timeOfDay = timeDriver->GetNormalizedTime();
```

`TimeOfDayDemo` sets Outdoor profile + animated driver — reference for build-along.

### Exposure and tonemap

`resolved.exposure` flows to HDR tonemap pass (Part VIII), not directly to `scene.frag` sun math. Profile tuning separates **scene linear radiance** from **display mapping**.

### Shadow distance fade

Outdoor `distanceFadeEnd = 150` meters — shader reads `ubo.viewportSize.z` as fade end:

```glsl
float fade = smoothstep(fadeStart, shadowFadeEnd, distCam);
sunShadow = mix(sunShadow, 1.0, fade);
```

Shadows dissolve at distance — hide cascade limits in open worlds.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Preset enum not JSON per scene

**Concept:** Profile enum

**Decision:** Four built-in rows in C++ switch

**Why:** Fast, type-safe, no asset load for defaults; games can wrap with data later.

**In Spark:** `SceneLightingProfile.cpp` `LightingProfileSettingsFor`.

### Overrides on SceneRenderParams

**Concept:** Resolver merge

**Decision:** `exposure`, `shadowCascadeFar`, etc. on snapshot override preset when > 0

**Why:** Per-level tuning without new enum values.

**In Spark:** `ResolveSceneLightingFromParams`.

### Analytic ToD not sky texture

**Concept:** ApplyTimeOfDayLighting

**Decision:** Math sun + ambient; optional equirect sky separate

**Why:** Works without art; sky mesh can still use texture layer.

**In Spark:** `SceneLightingProfile.cpp`.

### Outdoor enables ToD by default

**Concept:** Preset table

**Decision:** `timeOfDay.enabled = true`, `normalizedTime = 0.42`

**Why:** Morning-ish lighting out of box for open scenes.

**In Spark:** Outdoor case in `LightingProfileSettingsFor`.

### Ambient probe for interiors

**Concept:** Hemisphere ambient

**Decision:** NightInterior high `probeWeight`

**Why:** Dark sun, visible fill in enclosed spaces without fake point lights everywhere.

**In Spark:** NightInterior preset row.

## 4. Implementation Walkthrough

### Spark Files We Touch This Chapter

- `include/spark/render/lighting/SceneLightingProfile.hpp`
- `src/spark/render/lighting/SceneLightingProfile.cpp`
- `src/spark/render/lighting/SceneLightingResolver.cpp`
- `src/spark/demo/TimeOfDayDemo.cpp`
- `src/spark/render/scene/VulkanSceneUniformWriter.cpp` (ambient pack)

### Step 1 — Profile sweep

Run same scene with Default, Outdoor, Interior, NightInterior — screenshot shadow distance and ambient.

### Step 2 — Animate time

`useTimeOfDay = true`, sweep `timeOfDay` 0 → 0.5 → 1.0 — sun arcs, color warms at sunrise/sunset.

### Step 3 — Disable ToD on Outdoor

Keep Outdoor profile but `useTimeOfDay = false` — preset static sun at 0.42 phase values unless ECS directional set.

### Step 4 — Manual ambient override

Set `params.ambientColor` — ground hemisphere shifts; verify in shader debug.

### Step 5 — Exposure only

Change `params.exposure` — tonemap brightness changes, not necessarily sun vector.

### Step 6 — TimeOfDayDemo diff

Run `TimeOfDayDemo`, log `lightDirectionWorld` each second — smooth rotation.

## 5. How This Connects to the Rest of Your Engine

Part **IV** complete: textures + lighting profiles. Part **V** uses profile cascade ranges for shadow matrices. Part **VIII** uses `resolved.exposure`. Part **X** gameplay may set profile per level load.

Profiles are **data-driven mood** — same shaders, different numbers in UBO and shadow passes.

## 6. Pitfalls and Debugging

- **Sun jumps at midnight** — `normalizedTime` wrap; use `floor` in analytic functions.
- **ToD fights ECS sun** — disable one source; document scene setup.
- **Interior still has long shadows** — wrong profile or cascade far override.
- **Flat ambient** — `ambientScale` too low or sky/ground colors too similar.
- **Night too bright** — check `sunIntensity` floor in NightInterior ToD curve.

## 7. Exercises

1. Plot `SunElevation01(t)` for t ∈ [0,1].
2. Add a fifth profile `Dusk` with custom colors — what fields change?
3. Document override precedence when both ECS sun and ToD enabled.
4. Match a photograph's sun angle by tuning `timeOfDay` manually.
5. List all UBO fields filled from `ResolvedSceneLighting` in uniform writer.

## Summary

Spark lights scenes through `SceneLightingProfile` presets (Default, Outdoor, Interior, NightInterior) resolved by `SceneLightingResolver` with optional per-scene overrides. Time-of-day animates sun direction, color, intensity, and hemisphere ambient via analytic curves on normalized phase. Ambient ground/sky/probe and exposure feed the UBO and tonemap path. Choose profile per level, animate `timeOfDay` for day/night cycles, and override individual fields when presets are close but not exact.

---

Continue to [Part V — Input and Camera](../part-05-input-camera/index.md).
