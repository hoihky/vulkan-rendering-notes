# SceneRenderParams Field Reference

> **Build-Along Milestone:** Trace any snapshot field from ECS to GPU consumer in under five grep steps.

**You will learn:**

- Navigate the CPU→GPU frame contract from one header.
- Know which render pass consumes each major field group.
- Extend the struct following submit → GPU plumbing order.

## 1. The Problem We Are Solving

`SceneRenderParams` is the **lingua franca** between simulation and GPU from Part II onward. During bug hunts you need a map: which array feeds which pass, which component fills which field — without rereading every prior part.

Spark's header is source of truth; your engine may rename types but should preserve **roles** documented here.


Appendix chapter 02 supports active development — bookmark it. When stuck mid-chapter on Parts XV–XVI, grep appendix first before re-reading entire prior part. Keep local notes column for your renamed types alongside Spark names.

Schedule 30-minute appendix refresh after any PR touching `SceneSubmit`, descriptor layouts, or `RecreateSwapchain`.


Keep `SceneRenderParams.hpp` open beside this appendix while debugging missing draws — field reference table maps header names to gather functions without re-grepping entire SceneSubmit.cpp.

## 2. Core Concepts

### CPU→GPU frame contract

`SceneRenderParams` is the **immutable-for-flight-slot** snapshot each frame:

```text
ECS + simulation  →  SceneSubmit Fill*  →  SceneRenderParams  →  VulkanRenderer
```

Gameplay never touches descriptors. Renderer reads arrays and writes UBO/SSBO/descriptor updates. Part II introduced the struct; Parts IV–XII extended fields — this appendix maps the result.

### Camera and sun block

Top-level fields drive scene UBO and lighting:

| Field | GPU consumer |
|-------|----------------|
| `viewProjection`, `cameraPositionWorld` | SceneUBO binding 0 |
| `lightDirectionWorld`, `lightColor`, `lightIntensity` | Sun term in scene.frag |
| `ambientColor`, `lightingProfile`, `timeOfDay` | Hemisphere / TOD |
| `directionalShadowsEnabled`, shadow bias fields | CSM path |

Filled by `TryFillSceneCameraFromWorld` and lighting resolve helpers at start of submit.

### Draw list arrays

| Array | Pass |
|-------|------|
| `draws` / `transparentDraws` | Forward scene opaque/trans (`SceneDrawListSorter` partitions transparent) |
| `meshInstancedBatches` / `meshInstances` | Rigid instanced path (`scene_instanced.vert`, SSBO binding 16) |
| `sprites` | HDR sprite pass |
| `tilemapTiles` + tilemap draws | Tilemap pass |
| `particles` | Particle pass |
| `pointLights` / `spotLights` | Cluster SSBO upload |
| `screenUi` | Post-tonemap UI |

Sort policies (`SceneSpriteSortMode`, `SceneTransparentSortMode`, per-draw `drawableSort` / `SceneRenderQueue`) live on params — submit sorts before record.

### Lit submit API (Spark main)

| Call | When |
|------|------|
| `world.Rendering().SubmitLit` | Default 3D demo render — fill + `SetSceneRenderParams` |
| `world.Rendering().SubmitLitWithCamera` | ECS main camera resolution |
| `world.Rendering().FillLit` | Editor: patch `SceneRenderParams` after gather |
| `FillStandardLitSceneFromWorld` | Legacy; same as `FillLit` internally |

See Part II Chapter 11 and Spark `docs/LIT_RENDER_API.md`.

### Texture array packing

`sceneTextures` — max 16 `SharedPtr<Texture2D>`. Each draw item carries `textureLayer` index into array. `FindOrAddSceneTexture` deduplicates during submit — same texture pointer → same layer.

Normal/ORM/emissive layers optional per `SceneDrawItem` — glTF PBR path from Part IV/VII.

### Post and screen-space fields

`ssaoEnabled`, radius, bias, strength — Part X screen-space pass reads from params or derived uniform pack. `exposure`, tonemap-related overrides connect to `HdrTonemapPass`.

Post-process **volumes** (Part X) may resolve into params flags before submit returns — not always 1:1 component draw items.

### Extension policy

Adding gameplay-visible rendering data:

```text
1. Add field to SceneRenderParams
2. Fill in SceneSubmit from component(s)
3. Extend VulkanSceneUniformWriter or pass record
4. Update GLSL UBO/SSBO if needed
```

Never add Vulkan types to params. Never skip step 1 and write GPU from gameplay — breaks tests and scripting boundary.

### Transparent draw sort

`transparentDraws` sorted by depth or material policy before record — incorrect sort shows see-through objects in wrong order. Opaque `draws` front-to-back for early-Z; transparent back-to-front typically.

When adding foliage cards, confirm which array they append to — wrong bucket breaks depth write assumptions.

### Time of day and lighting profile

`useTimeOfDay` and `lightingProfile` interact — outdoor preset enables sun arc; explicit `lightDirectionWorld` override still possible for cutscenes. Document which fields win in `ResolveSceneLightingFromParams` when both set.

### Fog and post toggles

`fogEnabled` data path may exist before GPU implements full fog — params field reserved; check shader actually consumes before enabling in demo. Same for exposure override vs preset.

When debugging black screen, log `draws.size()` and `sceneTextures.size()` — empty draws with valid camera implicates submit not params struct size.

### Copy vs reference semantics

`SetSceneRenderParams` copies entire struct — large vectors duplicate each Present call. Cost acceptable for learning scale (<500 draws); production may use double-buffered shared_ptr to immutable snapshot for multithreaded submit.

### SceneRenderParams field reference table

| Field group | Key fields | Filled by | Consumed by |
|-------------|------------|-----------|-------------|
| Camera | viewProjection, cameraPositionWorld, viewportSize | TryFillSceneCameraFromWorld | SceneUBO, all passes |
| Sun | lightDirectionWorld, lightColor, lightIntensity | ResolveSceneLighting | scene.frag |
| Ambient | ambientColor, lightingProfile, timeOfDay | Lighting resolve + TOD | scene.frag, sprites |
| Shadows | directionalShadowsEnabled, shadowBias, cascadeBlend | Submit + shadow pass | scene.frag binding 3 |
| Opaque draws | draws[] SceneDrawItem | Mesh gather | RecordSceneOpaque |
| Transparent | transparentDraws[] | Mesh gather + sort | RecordSceneTransparent |
| Sprites | sprites[], spriteSortMode | Sprite gather + sort | Sprite pass |
| Tilemaps | tilemapTiles[], tilemapDraws[] | Tilemap submitter | Tilemap pass |
| Particles | particles[] | Particle gather | Particle pass |
| Lights | pointLights[], spotLights[] | Light components | Cluster SSBO |
| Textures | sceneTextures[16] | FindOrAddSceneTexture | Binding 1 array |
| Post | ssaoEnabled, exposure, ssaoRadius | Post volumes + camera | SSAO, tonemap |
| Screen UI | screenUi[] | ScreenUi gather | ScreenUi pass |

**SceneDrawItem key fields:** modelMatrix, meshId, materialLayer, textureLayer, skinPaletteOffset, flags (shadow cast/receive).

Grep `struct SceneRenderParams` in header — appendix lags if you add fields without updating this table in same PR.

### SceneDrawItem field reference

Each `SceneDrawItem` in `params.draws[]` carries GPU record inputs:

| Field | Type role | Set by gather | Read by record |
|-------|-----------|---------------|----------------|
| modelMatrix | float4x4 | TransformComponent | Push constant |
| meshId | uint | MeshComponent registry | VB/IB bind |
| materialLayer | uint | MaterialComponent | Push / UBO |
| textureLayer | int | FindOrAddSceneTexture | Push binding 1 index |
| normalLayer | int | Material normal map | Push |
| ormLayer | int | Material ORM | Push |
| emissiveLayer | int | Material emissive | Push |
| skinPaletteOffset | int | SkinnedMesh + Animator | SSBO offset |
| flags | uint bitmask | Mesh + cast shadows | Cull + shadow pass |

**Flags bitmask (typical):**

```text
kSceneShadowCast   = 1 << 0
kSceneShadowReceive = 1 << 1
kSceneTransparent  = 1 << 2
```

Wrong bucket (`draws` vs `transparentDraws`) breaks sort — transparent in opaque array still depth-writes if material opaque.

Debug missing draw: log gather skip reason — null mesh, disabled component, failed frustum cull (Part XII), zero scale transform.

### Lighting resolve precedence table

When multiple sources set sun/sky, `ResolveSceneLightingFromParams` applies precedence:

| Priority | Source | Fields set |
|----------|--------|------------|
| 1 | Explicit directional component | lightDirectionWorld, lightColor |
| 2 | Time of day system | sun arc, ambient gradient |
| 3 | lightingProfile preset | Outdoor/Indoor defaults |
| 4 | Hardcoded fallback | ThreeDDemo default sun |

Cutscenes override TOD by writing directional component for duration — document override in scene script. Forgetting to clear override leaves night scene stuck at noon sun.

`useTimeOfDay` flag on params enables arc — disabling does not zero sun; explicit direction still applies. Grep resolve function when lighting "ignores" editor TOD slider — often explicit directional on camera rig entity.

Ambient scales separately from sun — indoor profile lowers ambientColor without changing shadow direction. SSAO strength interacts with ambient — dark ambient + strong SSAO crushes midtones; tune together in demo 17.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### SceneRenderParams.hpp single header

**Concept:** CPU→GPU frame contract

**Decision:** SceneRenderParams.hpp single header

**Why:** One grep target for all snapshot types — SceneDrawItem, SceneSpriteDraw, lights.

**In Spark:** `include/spark/engine/SceneRenderParams.hpp`.
### SceneSubmit Fill* module

**Concept:** Draw list arrays

**Decision:** SceneSubmit Fill* module

**Why:** Each subsystem gathers into params — renderer stays O(draws) not O(entities).

**In Spark:** `src/spark/scene/SceneSubmit.cpp`.
### TryFillSceneCameraFromWorld

**Concept:** Camera and sun block

**Decision:** TryFillSceneCameraFromWorld

**Why:** Central camera resolution — avoids each demo duplicating view/proj math.

**In Spark:** Called at start of standard lit fill.
### FindOrAddSceneTexture

**Concept:** Texture array packing

**Decision:** FindOrAddSceneTexture

**Why:** Stable layer indices across draws in one frame — required for bindless-style array.

**In Spark:** SceneSubmit.cpp lambda `findOrAddTexture`.
### No GPU handles in snapshot

**Concept:** Extension policy

**Decision:** No GPU handles in snapshot

**Why:** Keeps params serializable for replay debugging and future network snapshot experiments.

**In Spark:** Part XIII serializes components, not params.
### PR rule: update table when adding params field

**Concept:** SceneRenderParams field reference table

**Decision:** PR rule: update table when adding params field

**Why:** Stale appendix worse than none — field table in same PR as SceneSubmit.

**In Spark:** include/spark/scene/SceneRenderParams.hpp.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `include/spark/engine/SceneRenderParams.hpp`
- `src/spark/scene/SceneSubmit.cpp`
- `src/spark/render/scene/VulkanSceneUniformWriter.cpp`

### Step 1 — Skim SceneRenderParams top struct

Read fields from viewProjection through fog — note defaults.

*SceneRenderParams core fields and shadow/SSAO toggles*

```cpp
// include/spark/engine/SceneRenderParams.hpp (lines 299–340)
struct SceneRenderParams {
    static constexpr std::uint32_t MaxSceneTextures = 16;
    /** Must match GPU clustered lights SSBO capacity. */
    static constexpr std::uint32_t MaxPointLights = 256;

    Matrix4 viewProjection{};
    Array<SceneDrawItem> draws;
    /** Alpha-blended lit meshes (after opaque <c>draws</c>, before sprites). */
    Array<SceneDrawItem> transparentDraws;
    /** Textures packed into a GPU 2D array (same order as textureLayer indices in draws). Max 16. */
    Array<SharedPtr<Texture2D>> sceneTextures;
    /** Normalized; direction from a surface point toward the light (N·L). */
    Vector3 lightDirectionWorld{0.35F, 0.92F, 0.18F};
    Vector3 cameraPositionWorld{};
    Vector3 lightColor{1.0F, 0.97F, 0.9F};
    float lightIntensity = 0.92F;
    Vector3 ambientColor{0.05F, 0.055F, 0.07F};
    /**
     * When true, the renderer builds a directional shadow map from lightDirectionWorld and modulates sun BRDF.
     * When false, sun lighting is unshadowed (cheaper; useful for 2D-style ortho scenes).
     */
    bool directionalShadowsEnabled = true;
    /** When true, point/spot lights with <c>castsShadow</c> render shadow maps (capped per frame). */
    bool punctualShadowsEnabled = true;

    /** Screen-space ambient occlusion applied before tonemap (requires retained scene depth). */
    bool ssaoEnabled = true;
    float ssaoRadius = 0.35F;
    float ssaoBias = 0.02F;
    float ssaoStrength = 0.65F;
    /** Depth bias for shadow map compare (tune if you see acne). */
    float shadowBias = 0.0026F;
    /**
     * Extra shadow compare bias on grazing angles (meters-scale coefficient in scene.frag): adds
     * `shadowNormalBias * 0.22 * (1 - N·L)` to the depth bias. Does **not** move the world position before
     * `worldToShadowClip` (that broke UV vs depth consistency). Tune with `shadowBias` for acne vs Peter-panning.
     */
    float shadowNormalBias = 0.048F;
    /**
     * When true, scene.frag flips shadow-map V when sampling (`uv.y = 1 - uv.y`). Packed into GPU `viewportSize.w`
     * (xy remain framebuffer size). Enable on stacks where depth-compare sampling row order does not match NDC→UV
     * (often MoltenVK / Metal); if shadows look vertically mirrored or missing on the ground, toggle this.
```
### Step 2 — Read nested draw types

Understand SceneDrawItem PBR fields and SceneSpriteDraw sort keys.

*Lights, particles, sprite draw structs*

```cpp
// include/spark/engine/SceneRenderParams.hpp (lines 62–128)
struct ScenePointLight {
    Vector3 positionWorld{};
    float range = 8.0F;
    Vector3 color{1.0F, 1.0F, 1.0F};
    float intensity = 1.0F;
    bool castsShadow = false;
};

/**
 * Forward-rendered spotlight: position + range, axis direction (world, normalized toward lit cone center),
 * inner/outer cone as **full** cone angles in radians (outer >= inner; half-space used on GPU via cos(half)).
 */
struct SceneSpotLight {
    Vector3 positionWorld{};
    float range = 12.0F;
    Vector3 directionWorld{0.0F, 0.0F, -1.0F};
    /** Full cone opening angle in radians (smaller = tighter hotspot). */
    float innerConeRadians = 0.55F;
    /** Full cone opening angle in radians (must be >= inner). */
    float outerConeRadians = 0.85F;
    Vector3 color{1.0F, 1.0F, 1.0F};
    float intensity = 3.0F;
    bool castsShadow = false;
};

/** One billboard particle instance (CPU sim → Vulkan particle pass). */
struct SceneParticleInstance {
    Vector3 position{};
    float size = 0.1F;
    Vector4 color{1.0F, 1.0F, 1.0F, 1.0F};
};

/** One oriented decal projector instance (world pose + half-extents). */
struct SceneDecalDraw {
    Matrix4 projectorWorld = Matrix4::Identity;
    Vector3 halfExtents{0.5F, 0.5F, 0.25F};
    std::int32_t textureLayer = -1;
    float opacity = 1.0F;
};

/**
 * One alpha-blended textured quad in world space (unlit). Sorted before draw (see <c>SceneSpriteSortMode</c> on
 * <c>SceneRenderParams</c>).
 * Unit quad is XY [-0.5,0.5]² at z=0 in model space; transform scales/orients it.
 * textureLayer indexes SceneRenderParams::sceneTextures (-1 = tint-only, no texture sample).
 */
struct SceneSpriteDraw {
    Matrix4 model = Matrix4::Identity;
    Vector4 tint{1.0F, 1.0F, 1.0F, 1.0F};
    /** Atlas UV bounds (minU, minV, maxU, maxV) in normalized 0–1 space. */
    Vector4 uvRect{0.0F, 0.0F, 1.0F, 1.0F};
    std::int32_t textureLayer = -1;
    /** Matches sprite push constant layout (SpriteLighting2DMode). */
    SpriteLighting2DMode lightingMode = SpriteLighting2DMode::None;
    float lightingPad0 = 0.0F;
    float lightingPad1 = 0.0F;
    /** Shader-specific (see SpriteLighting2DMode). */
    Vector4 lightingParam0{1.0F, 1.0F, 1.0F, 1.0F};
    Vector4 lightingParam1{1.0F, 0.0F, 0.0F, 0.0F};
    /** Lower values draw first (behind); higher values draw on top within the sorting layer. */
    std::int32_t sortOrder = 0;
    /** Global sorting layer order from <c>RenderLayerRegistry</c> (lower draws behind). */
    std::int16_t sortingLayerOrder = 0;
    /** World Y anchor for <c>SceneSpriteSortMode::SortOrderThenWorldY</c> (sorting-group aware). */
    float sortWorldY = 0.0F;
    SceneBlendMode blendMode = kSceneBlendModeDefault;
};
```
### Step 3 — Trace rigid mesh gather

Follow frustum cull path into drawList push.

*Mesh gather and frustum cull region*

```cpp
// src/spark/scene/SceneSubmit.cpp (lines 242–260)
    Array<SceneDrawItem> drawList;
    drawList.Reserve(48);

    RigidDrawableSubmitSink rigidSink{drawList, params, findOrAddTexture, defaultShadowFlags};
    if (sceneForCulling != nullptr) {
        DispatchDrawableFrustumCull(
                *sceneForCulling, viewProjection, sceneForCulling->GetSpatialPartitionKind(), rigidSink);
    } else {
        world.ForEachActiveGameObject([&](GameObject* o) {
            if (o == nullptr) {
                return;
            }
            const MeshComponent* mc = o->GetComponent<MeshComponent>();
            if (mc == nullptr || !mc->GetMesh()) {
                return;
            }
            rigidSink.OnDrawable(o, *mc, o->GetComponent<MaterialComponent>(), o->GetWorldMatrix());
        });
    }
```
### Step 4 — Trace SceneSpriteDraw

Grep `SpriteComponent` → params.sprites push → VulkanSpritePass instance SSBO — document file:line in your wiki.
### Step 5 — Field → pass table

Copy into notes:

| params field | Record function |
|--------------|-----------------|
| draws | RecordScenePass |
| sprites | sprite pass in HDR |
| particles | RecordParticlePass |
| screenUi | screen UI after tonemap |
### Step 6 — Log snapshot once per second

```cpp
LOG("draws=%zu sprites=%zu lights=%zu",
    p.draws.size(), p.sprites.size(), p.pointLights.size());
```

Spike detection when accidental double submit.
### Step 7 — Immutability discipline

After `SetSceneRenderParams`, do not mutate vectors until frame fence completes — or double-buffer params if game mutates early.
### Step 8 — Design wind field exercise

Add `Vector3 windDirection` — choose UBO extension vs push constant; document std140 padding if UBO.

## 5. How This Connects to the Rest of Your Engine

Parts **IV–XII** extended fields incrementally. **Descriptor map** binds UBO fields. **Component reference** lists ECS sources. Part **XIII** serializes components that regenerate this struct each frame.


Appendix **descriptor map** binding 0 SceneUBO fields mirror camera/sun block here — cross-reference when UBO layout changes. Part **XI** sprite sort modes live on same params struct as mesh draws.

## 6. Pitfalls and Debugging

- **Bypassing snapshot** — gameplay calling Vulkan for one "quick" draw breaks layering.
- **Pointers in params** — use indices, SharedPtr to CPU assets only, never VkImageView.
- **std140 mismatch** — padding when extending UBO mirrors (Part III).
- **Mutating params mid-record** — data race with GPU reading prior frame slot.
- **Unsorted transparent draws** — wrong blend order if submit skips sort.


- **Adding params field without gather** — draw never appears despite component present.
- **Transparent sort wrong bucket** — foliage in opaque array breaks depth write.

## 7. Exercises

1. Copy SceneRenderParams field table; add one custom field end-to-end.
2. Trace transparent vs opaque bucket for one glTF asset.
3. Grep ResolveSceneLighting when timeOfDay and manual sun both set.

## Summary

SceneRenderParams holds camera, draws, lights, particles, 2D, UI, and post toggles each frame. Extend here first, then SceneSubmit, then GPU — use Spark header as reference. SceneDrawItem and lighting precedence tables complete the params reference alongside draw arrays and texture packing rules.

---

Next: [Descriptor Binding Map](03-descriptor-map.md).
