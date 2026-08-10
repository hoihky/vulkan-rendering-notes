# Fog Volumes

> **Build-Along Milestone:** Distant terrain fades into horizon color matching the sky on TerrainDemo.

**You will learn:**

- Implement height-based fog in linear HDR space.
- Drive fog from ECS fog volumes and global settings.
- Composite fog in forward scene shader before post sampling.
- Match fog color to sky horizon for seamless draw distance.

## 1. The Problem We Are Solving

Open worlds expose **draw distance**, z-fighting on distant meshes, and harsh geometry pop-in. **Distance fog** blends pixels toward a horizon color so the far clip plane disappears artistically. **Height fog** adds ground mist; **fog volumes** let designers override density and color in swamps, caves, or toxic zones.

Spark resolves fog parameters on CPU via `FogVolumeComponent` and `SceneRenderParams` — your build-along step wires the exponential falloff into `scene.frag` before tonemap consumes HDR color.

## 2. Core Concepts

### Exponential distance fog

**Fog** hides draw distance limits and sets mood by blending shaded color toward **fog color** with distance from camera:

```glsl
fogAmount = 1.0 - exp(-density * max(distance - fogStart, 0))
color = mix(litColor, fogColor, fogAmount)
```

| Parameter | Effect |
|-----------|--------|
| fogDensity | Higher → faster fade |
| fogStart / fogEnd | Artist-friendly linear falloff alternative |
| fogColor | Should match sky horizon (Part VII) |

Apply in **linear HDR** at end of `scene.frag` — before HDR buffer is sampled for SSAO/tonemap.


### Height-based falloff

Ground mist uses **world Y** to concentrate fog near the floor:

```glsl
heightFactor = exp(-max(worldY - groundLevel, 0) * heightFalloff)
fogAmount *= heightFactor;
```

Combine with distance term for valleys and shorelines. Terrain demos (Part XII) benefit strongly — distant hills dissolve into horizon color.


### Fog volumes

`FogVolumeComponent` mirrors post volume machinery:

```text
Camera inside fog volume → override fogEnabled, fogColor, fogDensity, fogStart, fogEnd
Highest priority wins (same as post volumes)
```

`ApplyRegionalRenderVolumes` fog loop runs before post loop — independent priority counters. Regional fog enables swamp pockets in otherwise clear levels.


### Forward vs post fog

| Approach | Pros | Cons |
|----------|------|------|
| Forward (scene.frag) | Per-material skip; uses lit color | Runs for every opaque pixel |
| Post (depth only) | Single fullscreen pass | Uniform on emissive/glass |

Spark stores fog parameters on `SceneRenderParams` and expects **forward application** in the scene shader — unlit/emissive materials can skip fog via flags when wired. GPU path comment notes data may exist before shader consumes it.


### Sky color matching

Seams appear when fog color ≠ sky horizon. **TimeOfDayDemo** ties fog tint to time-of-day sky parameters — as sun sets, fog warms with the sky gradient.

```text
fogColor ≈ lerp(groundAmbient, skyAmbient, horizonBlend)
```

Author fog color from a screenshot of the sky at the same time-of-day setting, not from an arbitrary gray.


### Fog with shadows and SSAO

Fog multiplies **after** direct lighting and shadow visibility — shadows remain visible through thin fog; heavy fog obscures contrast. SSAO darkens ambient **before** fog in the forward path — tune both together on outdoor scenes or ground contact looks overly dark in mist.


### Linear workflow audit

Trace one texel from albedo file to monitor:

```text
sRGB PNG → sparkSrgbToLinear → BRDF → RGBA16F → × exposure → ACES → gamma → UNORM swapchain
```

Any extra `pow(..., 2.2)` or missing decode step washes out contrast. HDR headroom only helps if sun values **exceed 1.0** in the float buffer before tonemap — verify in RenderDoc texture inspect.


### Post pass isolation

Tonemap and SSAO share the **fullscreen triangle** pattern but target different render passes and input views. Keeping them in separate classes prevents scene code from accidentally binding swapchain during world draw. Each post pass owns its push constants — exposure vs ssaoRadius — avoiding UBO reallocation per frame.


### Operational depth and demo validation

### Operational depth

Spark's reference demos (**ThreeDDemo**, **TerrainDemo**, **TimeOfDayDemo**) exist to validate each rendering contract in isolation before you merge subsystems in your engine. Run the demo that matches this chapter's topic after every implementation step — if Spark looks correct and your port does not, the bug is almost always struct layout, pass order, or matrix convention rather than a missing Vulkan feature.

**Descriptor set 0** carries the scene UBO, texture arrays, shadow maps, and punctual shadow SSBO. When debugging binding errors, use RenderDoc's pipeline state view to compare Spark's bound descriptors against yours for the same draw index. A common regression when adding shadows or HDR is recreating the descriptor set layout without re-binding shadow map views after swapchain recreate.

**Frame-in-flight indexing** must stay consistent: CPU writes UBO slot *i* while GPU may read slot *i* from the previous frame's submit. Shadow matrices uploaded in `WriteUniformBuffer(frameIndex)` must match the `frameIndex` passed to `RecordShadowMapPass` and the scene pass descriptor bind — mixing indices produces one-frame-late shadows that look like temporal filtering bugs.

When extending Spark's pattern, prefer **pure CPU modules** (cascade math, render volume resolve) that you can unit-test without Vulkan instances. GPU record functions should consume precomputed arrays — not re-derive frustum corners inside tight draw loops.


### Fog parameter semantics

`fogStart` / `fogEnd` provide artist-friendly linear ramp endpoints; `fogDensity` drives exponential term. Regional `FogVolumeComponent` overrides all fields when camera inside — highest priority wins, same as post volumes.

Spark stores params on CPU; **GPU wiring in scene.frag** is the build-along step — multiply lit linear color toward `fogColor` using view distance before HDR store.


### Horizon matching

Set `fogColor` from sky horizon color at current time of day — not neutral gray. TimeOfDayDemo pairs fog tint with sky parameters. Mismatch shows obvious **color seam** where terrain meets sky dome.


### Forward fog ordering

Apply fog **after** direct lighting and shadow visibility, **before** HDR buffer consumed by SSAO/tonemap. Fog on emissive-only materials may need skip flag — glowing particles through heavy fog look incorrect if fog multiplies emissive equally with diffuse.


### HDR debugging in RenderDoc

After capturing **ThreeDDemo**, select the HDR color attachment before tonemap. Texture inspect on sun-facing pixels — linear float values should exceed 1.0. If all values ≤ 1.0, either exposure is too low, sun intensity is clamped in scene shader, or you are viewing the wrong attachment (swapchain instead of RGBA16F).

Compare the same pixel after tonemap pass — display values should sit in 0–1 without channel clipping to exactly 1.0 on all RGB unless intentionally blown out.

SSAO scratch (when enabled) should show modulated HDR — not identical to raw scene unless strength is zero.


### Post stack ordering invariants

These invariants must hold every frame:

```text
1. Shadow passes complete before HDR scene begin
2. HDR scene end → color in SHADER_READ layout
3. SSAO (if on) reads HDR + copied depth → scratch RGBA16F
4. Tonemap reads scratch OR raw HDR → swapchain
5. Screen UI + ImGui after tonemap (display space)
```

Violating order 4 with UI before tonemap makes widgets exposure-scaled. Violating order 2 causes validation errors when SSAO or tonemap sample HDR.


### TimeOfDayDemo fog authoring

TimeOfDayDemo sets global fog parameters alongside sky colors — use as reference for horizon-matched fog tint. When sun color warms at sunset, fog color should warm similarly or terrain silhouette shows blue-gray cutout against orange sky.


### scene_ubo fog wiring build-along

Extend `scene_ubo.glsl` with `vec4 fogParams` (density, start, end, enabled) and `vec3 fogColor` padded to std140. `VulkanSceneUniformWriter` copies from `SceneRenderParams` after `ApplyRegionalRenderVolumes` resolves fog volumes. In `scene.frag` main() before HDR out:

```glsl
if (fogParams.w > 0.5) {
    float dist = length(vWorldPos - ubo.cameraPos.xyz);
    float fogAmt = clamp((dist - fogParams.y) / max(fogParams.z - fogParams.y, 1e-3), 0.0, 1.0);
    fogAmt = 1.0 - exp(-fogParams.x * dist);
    color = mix(color, fogColor, fogAmt);
}
```

Stay linear — fog color should match sky horizon from Part VII.


### FogVolumeComponent fields

`FogVolumeComponent` mirrors post volume shape API: box/sphere, halfExtents, priority, fogColor, density, start, end. Enabled flag allows designers to disable without removing component. Resolve writes into `SceneRenderParams` identically to global fog fields — shader cannot distinguish regional vs global once params uploaded.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### SceneRenderParams fog fields

**Concept:** Exponential distance fog

**Decision:** SceneRenderParams fog fields

**Why:** Global fog parameters gathered at submit — density, color, start/end available to scene UBO writer.

**In Spark:** `SceneRenderParams` lines 377–382 fog fields.

### Shader-side height term

**Concept:** Height-based falloff

**Decision:** Shader-side height term

**Why:** When wired in scene.frag, uses world Y from varyings — no extra texture.

**In Spark:** Forward fog block at end of scene shading (implement alongside Spark params).

### FogVolumeComponent + RenderVolumes

**Concept:** Fog volumes

**Decision:** FogVolumeComponent + RenderVolumes

**Why:** Regional overrides share point-in-volume tests with post volumes.

**In Spark:** `RenderVolumes.cpp` fog loop lines 64–86.

### Forward fog in scene shader (target)

**Concept:** Forward vs post fog

**Decision:** Forward fog in scene shader (target)

**Why:** Per-material control — emissive particles may skip fog when unlit path flagged.

**In Spark:** Params on CPU today; GPU wiring is build-along exercise.

### TimeOfDayDemo integration

**Concept:** Sky color matching

**Decision:** TimeOfDayDemo integration

**Why:** Fog color driven from sky/ambient settings for outdoor demos.

**In Spark:** `src/spark/demo/TimeOfDayDemo.cpp` fog params with time of day.

### Fog after lighting in scene.frag

**Concept:** Fog with shadows and SSAO

**Decision:** Fog after lighting in scene.frag

**Why:** Preserves shadow silhouette in thin fog; SSAO tuned separately.

**In Spark:** Document ordering: BRDF → shadows → fog → HDR out.

### color_space.glsl include

**Concept:** Linear workflow audit

**Decision:** color_space.glsl include

**Why:** Single sRGB decode function shared across scene and UI paths that sample albedo.

**In Spark:** `shaders/color_space.glsl` sparkSrgbToLinear.

### Separate VulkanHdrTonemapPass and VulkanScreenSpaceEffectsPass

**Concept:** Post pass isolation

**Decision:** Separate VulkanHdrTonemapPass and VulkanScreenSpaceEffectsPass

**Why:** Scene module never imports swapchain present pass — only HDR handle.

**In Spark:** `include/spark/render/post/` module split.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `src/spark/scene/RenderVolumes.cpp`
- `include/spark/ecs/components/rendering/FogVolumeComponent.hpp`
- `include/spark/engine/SceneRenderParams.hpp`
- `src/spark/demo/TimeOfDayDemo.cpp`
- `shaders/scene.frag`

### Step 1 — Locate fog fields in SceneRenderParams

Spark documents fog as regional data path — fields exist on the render snapshot even before GPU consumes them.

*Fog enable, color, density, start/end*

```cpp
// include/spark/engine/SceneRenderParams.hpp (lines 377–382)
    /** Regional fog from <c>FogVolumeComponent</c> (data path; GPU may ignore until wired). */
    bool fogEnabled = false;
    Vector3 fogColor{0.72F, 0.78F, 0.86F};
    float fogDensity = 0.0F;
    float fogStart = 8.0F;
    float fogEnd = 96.0F;
```

### Step 2 — Trace fog volume resolve

When camera enters `FogVolumeComponent` bounds, params override global fog. Same priority pattern as post volumes.

*Fog volume priority resolve*

```cpp
// src/spark/scene/RenderVolumes.cpp (lines 64–86)
    bool fogFound = false;
    std::int32_t fogPriority = 0;
    world.ForEachActiveGameObject([&](GameObject* o) {
        if (o == nullptr) {
            return;
        }
        const FogVolumeComponent* fog = o->GetComponent<FogVolumeComponent>();
        if (fog == nullptr || !fog->IsEnabled()) {
            return;
        }
        if (!PointInsideVolume(cameraPositionWorld, *o, fog->GetShape(), fog->GetHalfExtents())) {
            return;
        }
        if (!fogFound || fog->GetPriority() > fogPriority) {
            fogFound = true;
            fogPriority = fog->GetPriority();
            params.fogEnabled = true;
            params.fogColor = fog->GetFogColor();
            params.fogDensity = fog->GetFogDensity();
            params.fogStart = fog->GetFogStart();
            params.fogEnd = fog->GetFogEnd();
        }
    });
```

### Step 3 — Wire fog into scene UBO

Extend `scene_ubo.glsl` and uniform writer to carry fog parameters — match std140 alignment from Part III.

### Step 4 — Implement forward fog in scene.frag

At end of main(), compute view distance, fog amount, `mix(litRadiance, fogColor, fogAmount)` — stay linear. Skip for sky draws (`push.skyMode`).

### Step 5 — Tune on TerrainDemo

Increase density until distant hills fade into horizon — match fog color to sky gradient from Part VII.

### Step 6 — Ground mist with height falloff

Add height term so fog concentrates near terrain floor — verify on slopes and valleys.

### Step 7 — Fog + SSAO interaction

Enable both on outdoor scene — reduce SSAO strength slightly in foggy volumes if ground contact looks muddy.

### Step 8 — Regional fog volume test

Place high-density fog box in one region — walk in/out and confirm override via RenderVolumes priority.

### Step 9 — Read FogVolumeComponent header

Regional fog authoring component — same volume shapes as post volumes.

*Fog volume component fields*

```cpp
// include/spark/ecs/components/rendering/FogVolumeComponent.hpp (lines 9–38)
/** Regional fog overrides collected into <c>SceneRenderParams</c> each submit. */
class FogVolumeComponent final : public GameComponent {
public:
    static constexpr ComponentKind TypeKind = ComponentKind::FogVolume;

    [[nodiscard]] ComponentKind Kind() const noexcept override { return TypeKind; }

    [[nodiscard]] bool IsEnabled() const noexcept { return enabled; }
    void SetEnabled(const bool e) noexcept { enabled = e; }

    [[nodiscard]] VolumeShape GetShape() const noexcept { return shape; }
    void SetShape(const VolumeShape s) noexcept { shape = s; }

    [[nodiscard]] const Vector3& GetHalfExtents() const noexcept { return halfExtents; }
    void SetHalfExtents(const Vector3& e) noexcept { halfExtents = e; }

    [[nodiscard]] Vector3 GetFogColor() const noexcept { return fogColor; }
    void SetFogColor(const Vector3& c) noexcept { fogColor = c; }

    [[nodiscard]] float GetFogDensity() const noexcept { return fogDensity; }
    void SetFogDensity(const float d) noexcept { fogDensity = d; }

    [[nodiscard]] float GetFogStart() const noexcept { return fogStart; }
    void SetFogStart(const float s) noexcept { fogStart = s; }

    [[nodiscard]] float GetFogEnd() const noexcept { return fogEnd; }
    void SetFogEnd(const float e) noexcept { fogEnd = e; }

    [[nodiscard]] std::int32_t GetPriority() const noexcept { return priority; }
    void SetPriority(const std::int32_t p) noexcept { priority = p; }
```

## 5. How This Connects to the Rest of Your Engine

Fog interacts with Part **VII** sky color. Part **IX** shadows apply before fog in forward ordering. Part **XII** terrain distance culling pairs visually with fog. Chapter 4 post volumes share `RenderVolumes.cpp` resolve infrastructure.


Part **III** std140 UBO rules govern cascade arrays. Part **VIII** particles typically skip shadow receive. Validation layers should stay enabled until shadow and HDR passes are stable.

## 6. Pitfalls and Debugging

- **Fog after tonemap** — wrong space; banding and wrong color temperature.
- **Fog on emissive particles** — may need fog skip on unlit materials.
- **Depth-based post fog on transparent** — sort order issues; prefer forward fog for opaque world.
- **Fog color unrelated to sky** — visible horizon seam on TerrainDemo.

- **Skipping RenderDoc pass order check** — hours lost to matrix bugs when the real issue is pass sequencing.
- **Transposed cascade matrix upload** — shadows slide horizontally when the camera strafes.

## 7. Exercises

1. Implement ground mist with height falloff on TerrainDemo.
2. Drive fog color from directional light color at sunset in TimeOfDayDemo.
3. Compare forward fog vs hypothetical post-depth fog — list tradeoffs.
4. Place regional fog volume — verify priority over global fog settings.
5. Screenshot with fog disabled/enabled at same camera — measure far hill visibility.

## Summary

Fog blends distant pixels toward fog color using depth and height in linear HDR. Spark resolves volume overrides on CPU in RenderVolumes; forward application in scene.frag is the build-along GPU step.

---

Next: [Capture Tools](06-capture.md).
