# Exposure Volumes

> **Build-Along Milestone:** Walking through a volume boundary changes brightness smoothly without rebuilding GPU resources.

**You will learn:**

- Define volumes that override post settings by camera position.
- Resolve exposure on CPU each frame into tonemap parameters.
- Handle priority when multiple post volumes overlap.
- Separate exposure grading from direct light intensity.

## 1. The Problem We Are Solving

Global exposure works for uniform outdoor scenes; **interiors, caves, and cinematic beats** need local brightness control. Constantly rebalancing every light entity is fragile — artists want **invisible regions** that push exposure toward target values when the camera enters them.

**Post-process volumes** solve this on the CPU: test camera position against authored bounds, pick the winning volume, write `SceneRenderParams.exposure`, and pass the resolved value into `RecordTonemap`. Spark uses `PostProcessVolumeComponent` and `ApplyRegionalRenderVolumes` — no extra render pass required.

## 1b. Concept Deep Dive — CPU Volume Resolve, GPU Tonemap Multiply

### Volumes mutate SceneRenderParams, not Vulkan objects

Gameplay never calls Vulkan for exposure tweaks. Each frame `ApplyRegionalRenderVolumes` walks ECS, tests camera position against oriented box/sphere bounds, and writes overrides into the render snapshot before uniform upload:

```cpp
// src/spark/scene/volume/RenderVolumes.cpp (lines 88–114)
    world.ForEachActiveGameObject([&](GameObject* o) {
        const PostProcessVolumeComponent* post = o->GetComponent<PostProcessVolumeComponent>();
        if (post == nullptr || !post->IsEnabled()) return;
        if (!PointInsideVolume(cameraPositionWorld, *o, post->GetShape(), post->GetHalfExtents())) return;
        if (!postFound || post->GetPriority() > postPriority) {
            postFound = true;
            postPriority = post->GetPriority();
            if (post->HasExposureOverride()) {
                params.exposure = post->GetExposure();
            }
            if (post->HasSsaoOverride()) {
                params.ssaoEnabled = post->GetSsaoEnabled();
            }
        }
    });
```

Strict **priority winner** — highest number beats lower on overlap, not distance-weighted blend.

### Exposure scales tonemap input, not sun intensity

`PostProcessVolumeComponent` can also override `ambientScale` — that affects scene UBO ambient terms. Exposure alone multiplies HDR radiance in `tonemap.frag` via `resolvedLighting.exposure` passed to `RecordTonemap`. Artists darken cave **display** without rebalancing every directional light entity.

### Shared volume math with fog

`PointInsideVolume` in `VolumeRegions.cpp` serves both post and fog volume components — same transform-to-local-space test, independent priority counters per volume type. Camera position must be final for the frame (after physics, before submit) when resolve runs.

## 2. Core Concepts

### Post volume data model

**Post-process volumes** are invisible regions that override tonemap-related settings when the camera enters them:

```cpp
// Conceptual — see PostProcessVolumeComponent
VolumeShape shape;          // Box or sphere
Vector3 halfExtents;      // From transform scale
float exposure;             // Override multiplier
bool ssaoEnabled;           // Optional SSAO toggle
int priority;               // Winner on overlap
```

Artists place volumes in the editor (Part XIII) — dark cave lowers exposure, bright exterior restores default. No extra render pass is required for basic exposure; CPU resolve writes the active value each frame.


### CPU resolve each frame

```text
For each active PostProcessVolumeComponent:
  if camera inside bounds:
    track highest priority volume
Apply overrides → SceneRenderParams.exposure / ssaoEnabled
Resolve lighting → resolvedLighting.exposure
RecordTonemap(..., exposure)
```

Gameplay code never calls Vulkan — volumes affect **parameters**, not GPU objects. `ApplyRegionalRenderVolumes` in `RenderVolumes.cpp` walks ECS components after camera position is known.


### Point-in-volume tests

Spark supports **box** and **sphere** shapes oriented with the GameObject transform:

```text
Box:    |localCam| <= halfExtents per axis (in volume space)
Sphere: |localCam| <= radius
```

`PointInsideVolume` transforms camera world position to volume local space using inverse transform. Oriented boxes allow corridor-shaped regions without mesh colliders.


### Priority overlap

When multiple volumes contain the camera, **highest priority wins** — not blend by default in Spark's regional resolve:

| Volume A | Volume B | Result |
|----------|----------|--------|
| priority 0, exp 0.5 | priority 10, exp 1.2 | exposure = 1.2 |

For smooth blends at boundaries, implement **edge fade** by distance to volume faces (future enhancement) or use non-overlapping authored volumes with hand-tuned transitions.


### Exposure vs light intensity

Exposure scales **tonemap input** — it does not multiply sun `lightColor` in the scene pass unless you also wire ambient scale overrides. This separation lets artists darken the **display** without rebalancing every light entity.

`PostProcessVolumeComponent` also exposes `ambientScale` override — affects scene UBO ambient terms when resolved, not just tonemap.


### Serialization and authoring

Volume bounds derive from **TransformComponent** scale × halfExtents. Saving the scene (Part XIII) must persist:

- enabled flag, shape, halfExtents, priority
- which overrides are active (`hasExposure`, `hasSsao`)
- override values

Reloading a level should reproduce the same exposure behavior without re-placing volumes manually.


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


### Volume shapes and transforms

Box volumes use oriented half-extents from component × GameObject transform scale. Sphere volumes use max scale component as radius. `PointInsideVolume` transforms camera to local space — same helper serves fog (Chapter 5).

Priority is **strict greater-wins** — not distance-weighted blend. Design levels so special volumes have higher priority numbers than global defaults.


### Ambient scale override

Post volumes can override `ambientScale` in addition to exposure — affects scene UBO ambient terms, not just tonemap. Dark cave can reduce both display exposure **and** ambient fill for mood without moving sun entity.


### Resolve timing

`ApplyRegionalRenderVolumes` runs during submit before GPU uniform write — gameplay never touches Vulkan. Camera position must be final for the frame (after physics, before render) when resolve executes.


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


### Exposure animation for gameplay

Flashbang or cave exit sequences can animate `PostProcessVolumeComponent` exposure over time on CPU — no shader recompile. Interpolate exposure during resolve or lerp component field before `ApplyRegionalRenderVolumes`. Keep UI post-tonemap so HUD stays readable during brightness punch.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### PostProcessVolumeComponent

**Concept:** Post volume data model

**Decision:** PostProcessVolumeComponent

**Why:** ECS component on empty GameObject; transform scale defines world bounds. Keeps post authoring in the same entity model as lights and fog.

**In Spark:** `include/spark/ecs/components/rendering/PostProcessVolumeComponent.hpp`.

### RenderVolumes module

**Concept:** CPU resolve each frame

**Decision:** RenderVolumes module

**Why:** Single function `ApplyRegionalRenderVolumes` mutates `SceneRenderParams` during submit — before GPU uniform write.

**In Spark:** `src/spark/scene/RenderVolumes.cpp` lines 55–115.

### Shared VolumeRegions helper

**Concept:** Point-in-volume tests

**Decision:** Shared VolumeRegions helper

**Why:** Same point-in-volume math serves fog volumes (Chapter 5) — one implementation, two component types.

**In Spark:** `src/spark/scene/VolumeRegions.cpp` (used by RenderVolumes).

### Strict priority winner

**Concept:** Priority overlap

**Decision:** Strict priority winner

**Why:** Predictable for designers — higher number beats lower. Document in level design guide; avoid huge default volumes swallowing specials.

**In Spark:** Priority compare in `ApplyRegionalRenderVolumes` post loop.

### Tonemap reads resolved exposure only

**Concept:** Exposure vs light intensity

**Decision:** Tonemap reads resolved exposure only

**Why:** Decouples display grading from physical light energies — `RecordTonemap` last parameter.

**In Spark:** `VulkanRenderer.cpp` line 362 `resolvedLighting.exposure`.

### Component fields serialized in scene files

**Concept:** Serialization and authoring

**Decision:** Component fields serialized in scene files

**Why:** Round-trip through editor ensures cave/dungeon beats survive load/save.

**In Spark:** Part XIII scene serialization hooks for PostProcessVolumeComponent.

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

- `include/spark/ecs/components/rendering/PostProcessVolumeComponent.hpp`
- `src/spark/scene/RenderVolumes.cpp`
- `src/spark/render/core/VulkanRenderer.cpp`
- `include/spark/engine/SceneRenderParams.hpp`
- `src/spark/scene/VolumeRegions.cpp`

### Step 1 — Read PostProcessVolumeComponent

Note fields: shape, halfExtents, priority, optional SSAO and exposure overrides with `has*` guard flags.

*Post volume component API*

```cpp
// include/spark/ecs/components/rendering/PostProcessVolumeComponent.hpp (lines 9–60)
/** Regional post-process overrides (SSAO, exposure) blended into <c>SceneRenderParams</c>. */
class PostProcessVolumeComponent final : public GameComponent {
public:
    static constexpr ComponentKind TypeKind = ComponentKind::PostProcessVolume;

    [[nodiscard]] ComponentKind Kind() const noexcept override { return TypeKind; }

    [[nodiscard]] bool IsEnabled() const noexcept { return enabled; }
    void SetEnabled(const bool e) noexcept { enabled = e; }

    [[nodiscard]] VolumeShape GetShape() const noexcept { return shape; }
    void SetShape(const VolumeShape s) noexcept { shape = s; }

    [[nodiscard]] const Vector3& GetHalfExtents() const noexcept { return halfExtents; }
    void SetHalfExtents(const Vector3& e) noexcept { halfExtents = e; }

    [[nodiscard]] std::int32_t GetPriority() const noexcept { return priority; }
    void SetPriority(const std::int32_t p) noexcept { priority = p; }

    [[nodiscard]] bool HasSsaoOverride() const noexcept { return hasSsao; }
    [[nodiscard]] bool GetSsaoEnabled() const noexcept { return ssaoEnabled; }
    void SetSsaoEnabled(const bool e) noexcept {
        hasSsao = true;
        ssaoEnabled = e;
    }

    [[nodiscard]] bool HasExposureOverride() const noexcept { return hasExposure; }
    [[nodiscard]] float GetExposure() const noexcept { return exposure; }
    void SetExposure(const float e) noexcept {
        hasExposure = true;
        exposure = e;
    }

    [[nodiscard]] bool HasAmbientScaleOverride() const noexcept { return hasAmbientScale; }
    [[nodiscard]] float GetAmbientScale() const noexcept { return ambientScale; }
    void SetAmbientScale(const float s) noexcept {
        hasAmbientScale = true;
        ambientScale = s;
    }

private:
    bool enabled = true;
    VolumeShape shape = VolumeShape::Box;
    Vector3 halfExtents{6.0F, 4.0F, 6.0F};
    std::int32_t priority = 0;
    bool hasSsao = false;
    bool ssaoEnabled = true;
    bool hasExposure = false;
    float exposure = 1.0F;
    bool hasAmbientScale = false;
    float ambientScale = 1.0F;
};
```

### Step 2 — Trace RenderVolumes resolve

`ApplyRegionalRenderVolumes` tests camera against fog and post volumes independently — post block sets `params.exposure` and `params.ssaoEnabled` when overrides present.

*Post-process volume priority resolve*

```cpp
// src/spark/scene/RenderVolumes.cpp (lines 88–114)
    bool postFound = false;
    std::int32_t postPriority = 0;
    world.ForEachActiveGameObject([&](GameObject* o) {
        if (o == nullptr) {
            return;
        }
        const PostProcessVolumeComponent* post = o->GetComponent<PostProcessVolumeComponent>();
        if (post == nullptr || !post->IsEnabled()) {
            return;
        }
        if (!PointInsideVolume(cameraPositionWorld, *o, post->GetShape(), post->GetHalfExtents())) {
            return;
        }
        if (!postFound || post->GetPriority() > postPriority) {
            postFound = true;
            postPriority = post->GetPriority();
            if (post->HasSsaoOverride()) {
                params.ssaoEnabled = post->GetSsaoEnabled();
            }
            if (post->HasExposureOverride()) {
                params.exposure = post->GetExposure();
            }
            if (post->HasAmbientScaleOverride()) {
                params.ambientScale = post->GetAmbientScale();
            }
        }
    });
```

### Step 3 — Follow exposure to tonemap

Resolved lighting carries exposure into `RecordTonemap`. Grep `resolvedLighting.exposure` to see full CPU chain from params to push constants.

*Tonemap record with exposure argument*

```cpp
// src/spark/render/core/VulkanRenderer.cpp (lines 355–362)
    hdrTonemapPass.RecordTonemap(
            commandBuffer,
            imageIndex,
            frameIndex,
            presentRenderPass.vkPass,
            presentSwapchain().extent,
            presentationFramebuffers,
            resolvedLighting.exposure);
```

### Step 4 — Place test volumes

Author dark cave volume (exposure 0.4) and bright outdoor default (1.0) — walk camera through boundary and watch brightness ramp in HDR/tonemap output.

### Step 5 — Log blended exposure

ImGui label during tuning — confirm value changes smoothly when crossing volume bounds, not frame-snapping unless priority switches abruptly.

### Step 6 — Overlap test

Two overlapping volumes with different priority — verify winner matches strict priority rule. Document for level designers.

### Step 7 — SSAO override volume

Disable SSAO inside a bright outdoor volume while keeping it on in forest — uses `HasSsaoOverride` path in resolve."

### Step 8 — Serialize and reload

Save scene with volume bounds, reload, confirm exposure override reproduces without manual re-setup (Part XIII integration).

## 5. How This Connects to the Rest of Your Engine

Volumes affect **tonemap and optional SSAO**, not shadow pass order. Part **XII** outdoor → indoor transitions rely on this. Part **XI** screen UI ignores volumes — it draws post-tonemap in display space. Chapter 5 fog volumes share the same `RenderVolumes` resolve pass.


Part **III** std140 UBO rules govern cascade arrays. Part **VIII** particles typically skip shadow receive. Validation layers should stay enabled until shadow and HDR passes are stable.

## 6. Pitfalls and Debugging

- **Instant snap at boundary** — add edge fade if artists report harsh transitions (Spark uses strict inside/outside).
- **Huge default volume swallowing specials** — priority ordering and smaller interior volumes fix this.
- **Exposure documented as EV vs linear** — Spark uses linear multiplier; document for designers (2× = one stop brighter).
- **Expecting volume to darken sun without ambient override** — exposure is tonemap-side unless ambient scale also overridden.

- **Skipping RenderDoc pass order check** — hours lost to matrix bugs when the real issue is pass sequencing.
- **Transposed cascade matrix upload** — shadows slide horizontally when the camera strafes.

## 7. Exercises

1. Place two overlapping volumes — verify priority winner for exposure.
2. Animate volume exposure keyframes for flashbang recovery sequence.
3. Serialize volume bounds to scene file and reload — confirm override persists.
4. Compare screenshot inside vs outside volume at same sun angle.
5. Document linear exposure slider range your artists should use (0.25–2.0).

## Summary

Exposure volumes are CPU-resolved regions that write post settings into SceneRenderParams each frame. Spark uses PostProcessVolumeComponent, strict priority on overlap, and passes resolved exposure to RecordTonemap.

---

Next: [Fog Volumes](05-fog.md).
