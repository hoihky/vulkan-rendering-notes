# 2D Lighting

> **Build-Along Milestone:** Lit sprite brightens and darkens when rotating the directional light in RenderLayers2DDemo.

**You will learn:**

- Apply simplified lighting modes to sprites via SpriteLighting2DComponent.
- Reuse scene UBO directional light and clustered point lights in sprite.frag.
- Choose when rim/pulse HDR modes are appropriate vs full unlit sprites.

## 1. The Problem We Are Solving

Pure texture blits look flat in side-scrollers and ARPG-style games. **2D lighting** modulates sprite albedo using the same directional sun and optional point lights as the 3D scene — without paying for full BRDF, normal maps, or shadow map compares on every billboard.

Spark implements five modes in `sprite.frag`, driven by per-instance data gathered from `SpriteLighting2DComponent` at submit. Your milestone: run `RenderLayers2DDemo`, toggle lit mode on a character atlas, and rotate the sun until the sprite shading matches your expectation.
## 2. Core Concepts

### SpriteLighting2DMode taxonomy

Spark encodes per-sprite shading in `SpriteLighting2DMode`, stored on `SceneSpriteDraw::lightingMode` and passed to the GPU as a flat `int` varying. Modes in `sprite.frag`:

| Mode | Name | Fragment behavior |
|------|------|-------------------|
| 0 | None / Unlit | `tex × tint`, no lighting |
| 1 | Lit | Ambient + directional dot in 2D plane |
| 2 | Rim | Edge glow from radial distance in local XY |
| 3 | Pulse | Time-varying emissive using `ubo.timeGlobal` |
| 4 | Clustered | Adds point lights from cluster SSBO |

Modes 2–3 can push HDR values above 1.0 — tonemap (Part X) displays them correctly. Mode 4 reuses the same clustered light buffer as `scene.frag`, keeping one upload path for punctual lights. Artists attach `SpriteLighting2DComponent` beside `SpriteComponent`; submit copies mode and two `Vector4` param blocks into the instance record.

Mode integers must stay in sync between C++ SpriteLighting2DMode enum and GLSL branches — adding mode 5 requires shader and FillSpriteInstanceGpu padding review. RenderLayers2DDemo cycles presets; diff param0/param1 when copying a look.

When you add a sixth lighting mode later, treat it as a **shader contract change**: update the C++ enum, `FillSpriteInstanceGpu`, GLSL branches, and any ImGui preset tables in the same commit. Spark keeps modes as sequential integers rather than bitfields so the fragment shader can early-return without a lookup table. In `RenderLayers2DDemo`, each row of sprites uses a different mode with shared atlas UVs — diff the instance buffer in RenderDoc to see how only `lightingMode` and param vectors change while draw signature stays identical. This is the pattern for stylized 2D: one instanced pipeline, many looks via instance data.


### 2D directional lighting math

Lit mode (1) projects lighting onto the sprite **billboard plane** using `vLocalXY` from the unit quad corners and `ubo.lightDir.xy` from the scene UBO:

```glsl
vec2 l2 = normalize(ubo.lightDir.xy);
vec2 n2 = normalize(vLocalXY);
float ndl = max(0.0, dot(n2, l2));
vec3 lit = base.rgb * (amb + dif * ndl);
```

This is intentionally cheaper than full PBR — no normal maps, no shadow map compare for sprites in the baseline path. `lightingParam0.x/y` carry artist-tuned ambient and diffuse scales (`SpriteLighting2DComponent::param0`). Side-scrollers treat the quad normal as facing +Z; rotating the entity rotates the lit gradient with it. For top-down games, consider whether `HorizontalAbsVelX` FSM (Chapter 3) aligns with your chosen light direction.

Lit mode uses lightDir.xy only — 3D sun elevation still affects XY projection when sun low on horizon. Zero-length lightDir.xy triggers normalize epsilon in shader. Rotate sun in demo and verify gradient direction on upright character sprite.

Compare lit sprite output against a **lit mesh** under the same sun rotation in ThreeDDemo. The mesh uses tangent-space normals and shadow maps; the sprite uses a fake 2D normal from quad corner positions. That approximation breaks for strongly rotated billboards viewed from above — acceptable for side-scrollers, questionable for isometric games without adjusting light direction in level design. `lightingParam0.x` and `.y` are artist knobs: raise ambient for night scenes without changing the global UBO, or boost diffuse for high-contrast comic shading. Document default param presets per character archetype so gameplay code only swaps mode integers at runtime.


### Rim and pulse stylized modes

**Rim mode (2)** measures distance from the quad center in local space: `length(vLocalXY) * 2.0`, then raises `(1 - r)` to a power from `lightingParam0.w`. The result adds a colored fringe — torches, magic auras, selection outlines. **Pulse mode (3)** calls `pulse01(hz)` using `sin(ubo.timeGlobal.x * hz * 2π)` and mixes emissive color from `lightingParam0.rgb` with optional base darkening via `lightingParam1.y`.

Both modes output linear HDR radiance before tonemap. They are ideal for gameplay feedback (damage flash, collectible shimmer) without authoring separate animation frames. Tune `param0.w` (rim power) and `param1.x` (strength) in ImGui (Chapter 7) while the demo runs.

Pulse frequency from param0.w is Hz — values near zero still clamp to 0.01 in shader. Rim power param0.w below 0.05 clamps similarly. HDR pulse can exceed display without tonemap — always evaluate in final swapchain image.

Rim mode treats the quad as a **radial mask** in local space, not a physical light response. Selection rings in strategy games and damage flashes in action games both map cleanly to modes 2 and 3 without new textures. When stacking rim on top of lit mode is required, Spark chooses separate modes rather than combinatorial flags — if you need rim+lit simultaneously, add a dedicated mode branch instead of nesting conditionals ad hoc. Profile pulse frequency against seizure-safety guidelines for accessibility; `param0.w` above 8 Hz on large fullscreen sprites can distract. Capture HDR values before tonemap to verify pulse peaks stay below firefly-clamp thresholds if you enable aggressive bloom.


### Clustered point lights on sprites

Mode 4 loops clustered point lights when `ubo.clusterDepth.w >= 0.5` — the same gate `scene.frag` uses for punctual PBR. Each sprite accumulates:

```glsl
acc += base.rgb * lc * intens * sparkPointAttenuation(d, range) * difs;
```

Sprites use **world position** (`vWorldPos`) for distance, not the 2D plane approximation. This lets a point light near a character affect multiple sprites consistently with nearby meshes. Cost is O(lights × sprites) for every sprite in mode 4 — profile before marking an entire tile layer as clustered-lit. `RenderLayers2DDemo` showcases multiple presets side by side.

Clustered mode respects clusterDepth.w gate — disabling clusters in UBO makes mode 4 behave like ambient-only from param0.x. Point light count min() caps loop; excess lights truncated silently.

Mode 4 is the **expensive** sprite path: every instance executes a loop up to `kMaxClusteredPointLights`. Reserve it for hero characters, boss glow auras, and interactive props near dynamic lights — not for background foliage tiles converted to sprites. The gate `ubo.clusterDepth.w >= 0.5` matches `scene.frag` so disabling clustered lights in the scene UBO automatically degrades sprites to ambient-only without a separate toggle. Distance uses `vWorldPos` from the vertex shader, which includes the entity's full 3D transform; a sprite elevated on a platform receives point lights differently than one at ground Y. If light counts spike, verify the cluster upload path deduplicates lights identically for meshes and sprites.


### Reuse scene UBO and headers

`sprite.frag` includes the same headers as the 3D path:

```glsl
#include "scene_ubo.glsl"
#include "clustered_lights.glsl"
#include "color_space.glsl"
```

Directional `lightDir`, `lightColor`, exposure-related fields, and cluster metadata arrive in **set 0 binding 0** — no duplicate uniform buffer for 2D. When you rotate the sun in ThreeDDemo, lit sprites and meshes respond to the same vector. Keep lighting math in **linear space** after sRGB decode; never apply gamma in intermediate multiplies. The tonemap pass owns display encoding.

Sprite pass uses same DescriptorSet as scene — changing UBO layout in scene_ubo.glsl requires rebuilding sprite and scene pipelines together. timeGlobal.x drives pulse — ensure WriteUniformBuffer updates each frame.

Duplicating a second UBO for 2D would double upload bandwidth and desync sun direction when designers tune lighting in ThreeDDemo. Spark's include graph for `sprite.frag` deliberately mirrors `scene.frag` headers so struct layout changes fail at compile time across both modules. When extending `scene_ubo.glsl` with new fields — wind direction, time-of-day gradient — sprite shaders inherit them for free even if unused today. Exposure and fog fields in the UBO affect sprites only indirectly through tonemap and post; sprites do not apply fog in the baseline fragment path. Keep that separation in mind when outdoor scenes combine terrain fog (Part XII) with foreground sprite characters.


### ECS wiring at submit time

`SpriteLighting2DComponent` is optional. During `SceneSubmit`, if present, submit copies:

```cpp
sd.lightingMode = lit->GetMode();
sd.lightingParam0 = lit->GetParam0();
sd.lightingParam1 = lit->GetParam1();
```

If absent, mode stays `None` (unlit). Gameplay adjusts params at runtime — e.g., increase pulse strength when the player is invulnerable. Do not branch on gameplay flags inside `sprite.frag`; pass tunables through param vectors so one SPIR-V module serves all modes. Split pipelines per mode only after profiling proves branch cost dominates.

Sprites without SpriteLighting2DComponent default to mode None — zero GPU cost branch early-out. Adding component mid-gameplay requires no pipeline rebuild; only instance fields change.

Submit-time copying keeps **gameplay mutable** while the GPU snapshot stays immutable for the frame. A invulnerability power-up can lerp `lightingParam1.x` pulse strength in `OnUpdate`; submit writes the final values once before `PresentFrame`. Avoid caching `SpriteLighting2DComponent` pointers into `SceneRenderParams` — entities may despawn between submit and record on multi-threaded futures. The optional component pattern means unlit props pay zero extra SSBO bytes beyond default mode zero. When batching sprites for sort, lighting mode does not affect sort keys unless you explicitly add a material bucket — Spark sorts by layer, order, and Y anchor first.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Integer mode in instance SSBO

**Concept:** SpriteLighting2DMode taxonomy

**Decision:** Integer mode in instance SSBO

**Why:** One draw path serves unlit HUD-style sprites and stylized lit characters without separate renderer objects.

**In Spark:** `shaders/sprite.frag` — mode branches lines 41–93.

### SpriteLighting2DComponent param0 amb/dif

**Concept:** 2D directional lighting math

**Decision:** SpriteLighting2DComponent param0 amb/dif

**Why:** Artists tune contrast without recompiling shaders; submit forwards params unchanged.

**In Spark:** `include/spark/ecs/components/rendering/SpriteLighting2DComponent.hpp`.

### Time from scene UBO timeGlobal

**Concept:** Rim and pulse stylized modes

**Decision:** Time from scene UBO timeGlobal

**Why:** Pulse stays synchronized with gameplay systems reading the same clock — no duplicate timers.

**In Spark:** `sprite.frag` `pulse01()` lines 30–33.

### Shared cluster SSBO binding

**Concept:** Clustered point lights on sprites

**Decision:** Shared cluster SSBO binding

**Why:** One light upload feeds meshes and sprites; disabling clusters via UBO gate zeros cost instantly.

**In Spark:** `#include "clustered_lights.glsl"` in sprite.frag.

### Same descriptor set as scene pass

**Concept:** Reuse scene UBO and headers

**Decision:** Same descriptor set as scene pass

**Why:** Sprite pass binds `sceneDescriptors.DescriptorSet(frameIndex)` — no second UBO alloc.

**In Spark:** `VulkanSpriteRecordContext` in `VulkanRenderer.cpp` lines 315–325.

### Optional component on sprite entity

**Concept:** ECS wiring at submit time

**Decision:** Optional component on sprite entity

**Why:** Unlit sprites pay zero extra component memory; lit sprites opt in explicitly.

**In Spark:** `SceneSubmit.cpp` sprite loop lines 360–364.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `shaders/sprite.frag`
- `include/spark/ecs/components/rendering/SpriteLighting2DComponent.hpp`
- `src/spark/scene/SceneSubmit.cpp`
- `include/spark/demo/RenderLayers2DDemo.hpp`

### Step 1 — Read SpriteLighting2DComponent

Optional lit modes on sprite entities.

*SpriteLighting2DComponent*

```cpp
// include/spark/ecs/components/rendering/SpriteLighting2DComponent.hpp (lines 13–37)
class SpriteLighting2DComponent final : public GameComponent {
public:
    static constexpr ComponentKind TypeKind = ComponentKind::SpriteLighting2D;

    [[nodiscard]] ComponentKind Kind() const noexcept override { return TypeKind; }

    explicit SpriteLighting2DComponent(
            SpriteLighting2DMode modeIn = SpriteLighting2DMode::None,
            Vector4 param0In = {1.0F, 1.0F, 1.0F, 1.0F},
            Vector4 param1In = {1.0F, 0.0F, 0.0F, 0.0F}) noexcept
            : mode(modeIn), param0(param0In), param1(param1In) {}

    [[nodiscard]] SpriteLighting2DMode GetMode() const noexcept { return mode; }
    [[nodiscard]] const Vector4& GetParam0() const noexcept { return param0; }
    [[nodiscard]] const Vector4& GetParam1() const noexcept { return param1; }

    void SetMode(SpriteLighting2DMode m) noexcept { mode = m; }
    void SetParam0(const Vector4& v) noexcept { param0 = v; }
    void SetParam1(const Vector4& v) noexcept { param1 = v; }

private:
    SpriteLighting2DMode mode = SpriteLighting2DMode::None;
    Vector4 param0{1.0F, 1.0F, 1.0F, 1.0F};
    Vector4 param1{1.0F, 0.0F, 0.0F, 0.0F};
};
```

### Step 2 — Study lit and rim branches

Modes 1-3 in sprite.frag.

*lit rim pulse*

```glsl
// shaders/sprite.frag (lines 41–74)
    if (vLightingMode == 0) {
        outColor = base;
        return;
    }

    if (vLightingMode == 1) {
        vec2 l2 = normalize(ubo.lightDir.xy + vec2(1e-5));
        vec2 n2 = normalize(vLocalXY + vec2(1e-5));
        float ndl = max(0.0, dot(n2, l2));
        float amb = clamp(vLightingA.x, 0.0, 1.0);
        float dif = max(vLightingA.y, 0.0);
        vec3 lit = base.rgb * (amb + dif * ndl);
        outColor = vec4(lit, base.a);
        return;
    }

    if (vLightingMode == 2) {
        float r = length(vLocalXY) * 2.0;
        float rim = pow(max(0.0, 1.0 - r), max(0.05, vLightingA.a));
        vec3 rimc = vLightingA.rgb * rim * max(0.0, vLightingB.x);
        outColor = vec4(base.rgb + rimc, base.a);
        return;
    }

    if (vLightingMode == 3) {
        float hz = max(0.01, vLightingA.w);
        float pulse = pulse01(hz);
        float str = max(0.0, vLightingB.x);
        float mixb = clamp(vLightingB.y, 0.0, 1.0);
        vec3 emit = vLightingA.rgb * pulse * str;
        vec3 col = base.rgb * mix(1.0, 0.55 + 0.45 * pulse, mixb) + emit;
        outColor = vec4(col, base.a);
        return;
    }
```

### Step 3 — Read clustered sprite loop

Mode 4 point lights.

*clustered sprites*

```glsl
// shaders/sprite.frag (lines 76–93)
    if (vLightingMode == 4) {
        vec3 acc = base.rgb * clamp(vLightingA.x, 0.0, 1.0);
        float difs = max(0.0, vLightingA.y);
        if (ubo.clusterDepth.w >= 0.5) {
            uint numPt = min(clusterLights.numPointLights, kMaxClusteredPointLights);
            for (uint i = 0u; i < numPt; ++i) {
                vec3 p = clusterLights.pointPositionRange[i].xyz;
                float range = max(clusterLights.pointPositionRange[i].w, 1e-3);
                vec3 lc = clusterLights.pointColorIntensity[i].xyz;
                float intens = clusterLights.pointColorIntensity[i].w;
                float d = distance(vWorldPos, p);
                float att = sparkPointAttenuation(d, range);
                acc += base.rgb * lc * intens * att * difs;
            }
        }
        outColor = vec4(acc, base.a);
        return;
    }
```

### Step 4 — Trace submit wiring

Copy lighting fields at submit.

*SpriteLighting2D submit*

```cpp
// src/spark/scene/SceneSubmit.cpp (lines 360–366)
        if (const SpriteLighting2DComponent* lit = o->GetComponent<SpriteLighting2DComponent>()) {
            sd.lightingMode = lit->GetMode();
            sd.lightingParam0 = lit->GetParam0();
            sd.lightingParam1 = lit->GetParam1();
        }
        sd.blendMode = SceneSubmitDetail::ResolveSpriteBlendMode(*o);
        params.sprites.PushBack(sd);
```

### Step 5 — Run RenderLayers2DDemo

Compare unlit, lit, rim, pulse presets.

### Step 6 — Toggle unlit vs lit

Same atlas frame; rotate sun.

### Step 7 — Profile branch cost

Fullscreen unlit vs clustered lit.

### Step 8 — Milestone — sun direction affects lit sprite

Lit sprite responds to directional light.

## 5. How This Connects to the Rest of Your Engine

2D lighting reads **scene UBO** and **cluster SSBO** from Part IV. **HDR tonemap** (Part X) displays emissive pulse/rim modes above 1.0. **Sprite animation** (Chapter 3) updates UV rects independently of lighting mode. **Platformer pipeline** (Chapter 8) combines lit player sprites with unlit UI.

**Frame graph reminder:** simulation and component updates happen in `OnUpdate`; snapshot fill in `OnRender` or dedicated submit; GPU work in `PresentFrame`. No chapter in Part XI or XII introduces new Vulkan calls from gameplay — only new arrays on the snapshot and record slots in `VulkanRenderer`. Cross-reference Part X HDR ordering whenever you add a draw type: world content belongs in the HDR subpass; display UI belongs after tonemap.

Frame graph reminder for lit sprites: shadow maps (Part IX) may exclude sprite draws in baseline Spark, but directional light vector in scene UBO still drives mode 1. Tonemap (Part X) is mandatory evaluation surface for rim/pulse HDR peaks. When adding normal-mapped 2D in a fork, branch as new lighting mode rather than overloading mode 1 plane dot — keeps teaching path stable.

## 6. Pitfalls and Debugging

- **Treating sprites like 3D meshes** — no tangent-space normal maps in baseline sprite.frag; use mode 1 plane dot.
- **Lighting in gamma space** — always decode sRGB albedo first via `sparkSrgbToLinear`.
- **Mode 4 on every background tile** — cluster loop per sprite is expensive; reserve for hero characters.
- **Forgetting param0 amb/dif defaults** — zero ambient makes lit mode look harsh black on back faces.
- **Rotating lightDir without updating UBO** — sprite and mesh will disagree; one upload per frame in WriteUniformBuffer.

**Debugging workflow:** when visuals wrong, capture RenderDoc and list (1) pass name, (2) attachment format, (3) bound pipeline, (4) push/SSBO bytes for one failing draw. Compare with Spark capture on the same demo. Validation errors include VUID — search Khronos spec. CPU-side logging of snapshot array sizes (`sprites`, `tilemaps`, `draws`) often finds empty submit before GPU debugging.

Debug checklist: (1) confirm `lightingMode` in instance SSBO, (2) verify sRGB decode ran before lighting multiply, (3) compare `ubo.lightDir.xy` against mesh shading, (4) profile mode 4 only on hero sprites. Validation rarely catches wrong mode integers — visual compare against RenderLayers2DDemo row is faster.

## 7. Exercises

1. Map each `SpriteLighting2DMode` enum value to the `if (vLightingMode == N)` branch in `sprite.frag`.
2. Tune rim power (`param0.w`) on a torch sprite until the fringe reads at game resolution.
3. Disable clustered lights via UBO gate and confirm mode 4 matches mode 0 visually.
4. Compare RenderDoc shader invocations: fullscreen unlit vs fullscreen clustered-lit.
5. List which `lightingParam0` fields each mode reads.

## Summary

2D lighting modulates sprite albedo through per-instance modes in sprite.frag, reusing scene UBO and clustered lights. SpriteLighting2DComponent supplies mode and param vectors at submit time.

---

Next: [Sprite Animation](03-sprite-anim.md).
