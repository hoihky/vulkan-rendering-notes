# Render Layers

> **Build-Along Milestone:** Two overlapping alpha sprites consistently sort by layer every frame without flicker.

**You will learn:**

- Assign integer render layers via RenderLayerComponent and registry.
- Resolve sort keys with DrawableSortResolver including sorting groups.
- Stable-sort sprite and tilemap draw arrays before GPU record.

## 1. The Problem We Are Solving

Alpha-blended 2D draws need **predictable ordering**. Depth buffer alone fails for coplanar quads and transparent overlap. **Render layers** are integer priorities computed at submit, combined with optional world-Y tie-breaks, and applied via stable sorts before `VulkanSpritePass` records instances.

Spark centralizes resolution in `DrawableSortResolver` — your milestone: two overlapping sprites on different layers always composite the same way, with no frame-to-frame flicker.
## 2. Core Concepts

### Why sort keys exist

Without explicit **draw order**, alpha-blended sprites and tile layers intersect unpredictably — foreground props flicker behind the player, decals vanish when overlap angles change. Vulkan draw order within a pass is **submission order** unless you depth-sort (insufficient alone for alpha).

Spark assigns each 2D drawable a **sort key** at submit time: `sortingLayerOrder` (global layer from registry), `sortOrder` (order within layer), and optional `sortWorldY` for pseudo-depth in side-scrollers. Keys are resolved once per entity, stable-sorted, then recorded — gameplay never sorts during GPU work.

Undefined order shows as Z-fighting between alpha sprites coplanar in XY — separate with sortOrder or tiny Z bias. 3D meshes ignore sprite sort keys — pass order decides mesh vs sprite.

Transparent draws lack a single depth order that looks correct from all angles. **Sort keys** encode artist intent: background hills at layer 0, characters at 10, foreground vines at 20. Without keys, alpha-blended overlap flickers frame-to-frame when insertion order varies. Sort keys are integers computed at submit, not at entity creation — runtime gameplay can bump pickup sprites to layer 99 for one frame when collected. Depth buffer still resolves sprites against opaque terrain; sort keys only order among coplanar or similarly-depth sprites. Document layer ranges in a table shared between art and engineering.

Multiplayer replays and video capture require **deterministic overlap** — stable sort plus explicit layer integers make the same scene reproducible across machines. When two artists assign the same layer to unrelated objects, establish a secondary `sortOrder` tie-break rather than renumbering global layers mid-milestone. Document layer budget: background 0–9, gameplay 10–29, VFX 30–49, foreground 50+ as a starting convention for side-scrollers.


### DrawableSortResolver hierarchy

`DrawableSortResolver::Resolve` combines ECS components into `ResolvedDrawableSort`:

| Input | Effect |
|-------|--------|
| `RenderLayerComponent` | Named layer → `sortingLayerOrder` via registry |
| `SortingGroupComponent` on ancestor | Batches children at group order (Unity-style) |
| Native order (`SpriteComponent::sortOrder`) | Used when no explicit order-in-layer override |
| World Y anchor | Tie-break for `SceneSpriteSortMode::SortOrderThenWorldY` |

Nearest enabled sorting group wins when walking parent chain. This lets a “character” group keep hat, body, and weapon together while still sorting against the tilemap.

RenderLayerRegistry singleton loads named layers at startup — typo in layer id falls back to default order. SortingGroupComponent on parent batches children to group order plus local order.`DrawableSortResolver::Resolve` merges **SortingGroupComponent** parent overrides, per-component `sortOrder`, and registry defaults into a `ResolvedDrawableSort` key. Sorting groups let you batch a character's body, weapon, and hat without manual order micromanagement — children inherit group layer unless explicitly overridden. World-Y anchor breaks ties for top-down games where layer alone is insufficient. The resolver runs per entity during submit, not during render — record path reads flat arrays only. Adding a new drawable type requires calling the same resolver for consistent policy across sprites, tilemap tiles, and particles.

Parent **SortingGroupComponent** overrides child sortOrder when configured — children still contribute world-Y anchor for tie-breaks unless group policy says otherwise. Document precedence in a unit test: child sortOrder 100 under group layer 5 resolves to group layer, not 100. Resolver runs per drawable type with the same API — adding particles to custom sort policy means calling Resolve with particle component adapter if you extend the engine.


### Stable sort semantics

Spark uses **stable insertion sort** for sprite and tilemap arrays — equal keys preserve insertion order from the world walk. Unstable sorts cause **alpha flicker** frame-to-frame when two sprites share the same layer and sort order.

```cpp
SceneSubmitDetail::StableSortSprites(params.sprites, params.spriteSortMode);
StableSortTilemapDraws(params.tilemaps);
```

`SceneSpriteSortMode` selects whether world Y breaks ties after sort order — `TilemapShowcase2DDemo` passes `SortOrderThenWorldY` at submit. Document your project's mode in the submit call; hybrid 3D/2D games often use sort order only for UI-world sprites.

std::sort without stability causes flicker — Spark uses insertion sort helpers deliberately. SceneSpriteSortMode enum passed to StableSortSprites from demo Render.`StableSortSprites` preserves **relative order** among equal keys — critical when two rain particles share layer and Y but spawned in painter's order. Unstable sorts swap equal elements across frames, producing sparkle on overlapping alpha. Spark uses stable sort policy on sprite arrays after all instances are collected. Sort mode enum on `SceneRenderParams` selects ascending vs descending Y tie-break for different game genres. Never sort after GPU upload; SSBO order must match sorted CPU array when debugging overdraw in RenderDoc instance inspector.

Stable sort cost is O(n log n) with higher constant than unstable — acceptable for thousands of sprites, not millions of particles without radix alternatives. `spriteSortMode` on scene params selects Y ascending vs descending for top-down vs platformer conventions — set once per demo, not per entity. When debugging sort, log keys `(sortingLayerOrder, sortOrder, sortWorldY, stableIndex)` for overlapping pairs only to reduce log spam.


### RenderLayerRegistry policy

Named layers (`"Background"`, `"Entities"`, `"Foreground"`) map to integer `sortingLayerOrder` values at load. Leave **gaps** (0, 100, 200) so designers insert categories without renumbering everything. Components store layer id + optional explicit order-in-layer.

`RenderLayers2DDemo` documents default policy for hybrid scenes — ground tilemaps low, entities mid, VFX high. Part VIII particles often render after sprites in the HDR pass with the highest effective layer — additive flashes should not be blocked by accidental sort keys.

sortingLayerOrder is int16 — saturating arithmetic in resolver prevents overflow when combining group and local orders.

Central registry assigns **named layers** to integer priorities — "Background", "Default", "Foreground", "UIWorld". Registry prevents artists from typing magic numbers into fifty prefabs. Custom properties in TMX can reference registry names if you extend import — Spark demos use component fields directly. Layers do not cross render passes: a sprite at layer 100 still draws in HDR composite, not above opaque meshes, unless you also change pass order. Registry documentation should list which layers receive lighting modes from Chapter 2 and which stay unlit for performance.

Name-to-integer mapping should be **data-driven** from a JSON or header table generated at build time so designers add layers without recompiling C++. Invalid layer names at component attach time should fail loudly in development — silent fallback to default layer hides prefab misconfiguration until alpha overlap looks wrong weeks later.


### Separate pass ordering vs layer keys

Render layers control order **within** the 2D composite pass. The **frame graph** still orders pass types:

```text
opaque 3D → transparent 3D → tilemap → sprite → particles → tonemap → screen UI
```

A mesh with sort layer 999 still draws in opaque pass before any sprite — layers do not move draws across pass boundaries. When mixing 3D terrain (Part XII) with 2D overlays, terrain is opaque mesh; sprites sort among themselves above or below only if depth and submit order align with your 2.5D camera.

Particles after composite2D — VFX always on top of sprites unless custom pass inserted. Transparent meshes before 2D composite — glass occludes sprites via depth.

Pass order is **coarser** than layer keys: shadow → opaque → transparent mesh → 2D composite → particles → post. Layer keys sort within the 2D composite bucket only. Expecting a sprite layer to interleave with opaque mesh draws requires mesh transparency or mesh render queue changes — not sprite layer alone. Tilemap layers and sprite layers share sort key namespace when configured — align TMX layer order with sprite registry priorities in platformer scenes. Frame graph diagrams should show both pass boundaries and layer ranges to onboard new rendering engineers quickly.

Particles in Part VIII often render after composite2D — their sort keys operate in particle pass context, not sprite layer namespace, unless you unify policies deliberately. Transparent **meshes** between opaque terrain and sprites require mesh render queue or separate subpass — sprite layer 999 cannot draw between two mesh passes. Frame graph documentation should list pass boundaries as primary, layer table as secondary sort within pass.


### TMX custom properties

Tiled layer properties can encode `renderLayer` strings parsed at import into `TilemapLayer` sort fields. Autotile and animation components do not change sort keys — only layer metadata and `DrawableSortResolver` outputs do. Editor pipeline (Part XIII) should expose layer names matching `RenderLayerRegistry` entries to avoid silent defaults dumping decor into layer 0.

Import pipeline maps TMX layer properties to sort fields — document property names for level designers in template README.

Tiled custom properties export as strings on objects and tiles — Spark importers can map `sortingLayer` properties to component fields at load. Document required property names in your map template files. Autotile and collision layers often live on invisible layers with distinct sort priorities — set layer visibility off in Tiled but keep collision export enabled. Flip and rotation bits in gid affect neighbor autotile masks; apply transforms before autotile evaluation during import, not at render. Version TMX files with git-friendly line endings; large maps diff poorly — consider chunk-based map files for production.

Export scripts from Tiled can embed **spark:sortingLayer** strings on objects — map importers read these into `SpriteComponent` or tile object spawners. Version your TMX schema in repo docs when adding custom properties so level designers sync tool templates. Global tile IDs with flip bits set above 0x80000000 need masking before property lookup — handle in one shared ImportTiledGid helper.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Resolve at submit, not record

**Concept:** Why sort keys exist

**Decision:** Resolve at submit, not record

**Why:** CPU sort once per frame on plain arrays — GPU sees deterministic order.

**In Spark:** `SceneSubmit.cpp` sprite loop lines 351–371.

### Central static resolver

**Concept:** DrawableSortResolver hierarchy

**Decision:** Central static resolver

**Why:** One implementation shared by sprites and tilemaps — no duplicate sort logic.

**In Spark:** `src/spark/scene/DrawableSortResolver.cpp` lines 81–95.

### Insertion sort helpers

**Concept:** Stable sort semantics

**Decision:** Insertion sort helpers

**Why:** Deterministic equal-key ordering prevents alpha shimmer on stacked sprites.

**In Spark:** `SceneSubmitDetail::StableSortSprites`.

### Named layers with gaps

**Concept:** RenderLayerRegistry policy

**Decision:** Named layers with gaps

**Why:** Designers add ForegroundVFX at 250 without shifting Entities at 100.

**In Spark:** `include/spark/scene/RenderLayerRegistry.hpp`.

### Document frame graph separately

**Concept:** Separate pass ordering vs layer keys

**Decision:** Document frame graph separately

**Why:** Layers cannot override pass boundaries — avoids false expectation that layer 999 beats opaque terrain.

**In Spark:** `VulkanRenderer.cpp` HDR record order lines 299–328.

### Import-time layer mapping

**Concept:** TMX custom properties

**Decision:** Import-time layer mapping

**Why:** Sort policy authored in Tiled travels with map file — code defaults only as fallback.

**In Spark:** `TilemapLayerSort.cpp` sort mode enums.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `include/spark/scene/DrawableSortResolver.hpp`
- `src/spark/scene/DrawableSortResolver.cpp`
- `src/spark/scene/tilemap/TilemapLayerSort.cpp`
- `include/spark/demo/RenderLayers2DDemo.hpp`

### Step 1 — Read DrawableSortResolver API

Layer and sorting group rules.

*DrawableSortResolver*

```cpp
// include/spark/scene/DrawableSortResolver.hpp (lines 11–35)
/**
 * Resolves effective 2D draw order for sprites and tilemaps from the ECS hierarchy.
 *
 * - <c>RenderLayerComponent</c> selects a named layer and optional order-in-layer.
 * - <c>SortingGroupComponent</c> on an ancestor batches descendants at the group's order
 *   (Unity Sorting Group semantics: nearest enabled group wins).
 * - Drawable-native order (<c>SpriteComponent::sortOrder</c>, <c>TilemapComponent::sortOrderBase</c>)
 *   is used when no <c>RenderLayerComponent</c> overrides order-in-layer.
 */
class DrawableSortResolver {
public:
    [[nodiscard]] static ResolvedDrawableSort Resolve(const GameObject& drawable, std::int32_t nativeOrder) noexcept;

private:
    [[nodiscard]] static const SortingGroupComponent* FindNearestSortingGroup(const GameObject& drawable) noexcept;
    [[nodiscard]] static std::int16_t ResolveLayerSortingOrder(
            const GameObject& drawable,
            const SortingGroupComponent* sortingGroup) noexcept;
    [[nodiscard]] static std::int32_t ResolveLocalOrderInLayer(
            const GameObject& drawable,
            std::int32_t nativeOrder) noexcept;
    [[nodiscard]] static float ResolveWorldYAnchor(
            const GameObject& drawable,
            const SortingGroupComponent* sortingGroup) noexcept;
};
```

### Step 2 — Study Resolve implementation

sortingLayerOrder and sortOrder.

*Resolve*

```cpp
// src/spark/scene/DrawableSortResolver.cpp (lines 81–95)
ResolvedDrawableSort DrawableSortResolver::Resolve(const GameObject& drawable, const std::int32_t nativeOrder) noexcept {
    ResolvedDrawableSort resolved{};
    const SortingGroupComponent* sortingGroup = FindNearestSortingGroup(drawable);
    resolved.key.sortingLayerOrder = ResolveLayerSortingOrder(drawable, sortingGroup);
    const std::int32_t localOrder = ResolveLocalOrderInLayer(drawable, nativeOrder);
    if (sortingGroup != nullptr) {
        const std::int64_t combined =
                static_cast<std::int64_t>(sortingGroup->GetSortingOrder()) + static_cast<std::int64_t>(localOrder);
        resolved.key.sortingOrder = SaturateSortingOrder(combined);
    } else {
        resolved.key.sortingOrder = localOrder;
    }
    resolved.worldYAnchor = ResolveWorldYAnchor(drawable, sortingGroup);
    return resolved;
}
```

### Step 3 — Follow sprite submit sort fields

StableSortSprites after resolve.

*sprite sort*

```cpp
// src/spark/scene/SceneSubmit.cpp (lines 351–371)
        const ResolvedDrawableSort resolved = DrawableSortResolver::Resolve(*o, sc->GetSortOrder());
        sd.sortOrder = resolved.key.sortingOrder;
        sd.sortingLayerOrder = resolved.key.sortingLayerOrder;
        sd.sortWorldY = resolved.worldYAnchor;
        if (sc->GetTexture()) {
            sd.textureLayer = findOrAddTexture(sc->GetTexture());
        } else {
            sd.textureLayer = -1;
        }
        if (const SpriteLighting2DComponent* lit = o->GetComponent<SpriteLighting2DComponent>()) {
            sd.lightingMode = lit->GetMode();
            sd.lightingParam0 = lit->GetParam0();
            sd.lightingParam1 = lit->GetParam1();
        }
        sd.blendMode = SceneSubmitDetail::ResolveSpriteBlendMode(*o);
        params.sprites.PushBack(sd);
    });

    tilemapSubmitter.Submit(world, params, spriteTileCull, findOrAddTexture, SceneSubmitDetail::ResolveSpriteBlendMode);

    SceneSubmitDetail::StableSortSprites(params.sprites, params.spriteSortMode);
```

### Step 4 — Run RenderLayers2DDemo

Overlapping sprites different layers.

### Step 5 — Overlap test

Layer 10 vs 20 alpha overlap.

### Step 6 — Add SortingGroupComponent

Batch child sprites.

### Step 7 — Document frame graph and layer policy

Pass order + layer ranges table.

### Step 8 — Milestone — deterministic alpha overlap

No flicker over 5 minutes.

## 5. How This Connects to the Rest of Your Engine

Layers connect **sprites** (Chapter 1) and **tilemaps** (Chapter 4). **Particles** (Part VIII) typically highest HDR layer. **Screen UI** (Chapter 6) is separate pass after tonemap — registry layers do not apply there. **3D terrain** (Part XII) uses opaque pass, not sprite sort keys.

**Frame graph reminder:** simulation and component updates happen in `OnUpdate`; snapshot fill in `OnRender` or dedicated submit; GPU work in `PresentFrame`. No chapter in Part XI or XII introduces new Vulkan calls from gameplay — only new arrays on the snapshot and record slots in `VulkanRenderer`. Cross-reference Part X HDR ordering whenever you add a draw type: world content belongs in the HDR subpass; display UI belongs after tonemap.

Sort keys apply within HDR 2D composite — not across opaque terrain (Part XII). Particles (VIII) typically sort above sprites via layer policy. Screen UI (Chapter 6) ignores sprite layer registry entirely.

## 6. Pitfalls and Debugging

- **Transparent needs sort within layer** — same layer + same order → artist discipline required.
- **Unstable sort** — never use `std::sort` without equal-key tie policy on alpha draws.
- **Expecting layer to cross passes** — sprites cannot sort above opaque mesh purely via layer id.
- **Forgetting SortingGroup on root** — children may interleave with unrelated sprites mid-layer.
- **World Y mode with ortho top-down** — Y tie-break may not match design intent; pick sort mode explicitly.

**Debugging workflow:** when visuals wrong, capture RenderDoc and list (1) pass name, (2) attachment format, (3) bound pipeline, (4) push/SSBO bytes for one failing draw. Compare with Spark capture on the same demo. Validation errors include VUID — search Khronos spec. CPU-side logging of snapshot array sizes (`sprites`, `tilemaps`, `draws`) often finds empty submit before GPU debugging.

Stable sort is non-negotiable for alpha coplanar stacks. SortingGroup parent overrides confuse if child also sets high sortOrder — document precedence in DrawableSortResolver tests.

## 7. Exercises

1. Draw overlapping sprites on layers 10 vs 20 — capture video, verify no flicker.
2. Add TMX `renderLayer` property to a decor layer — confirm import sets `sortingLayerOrder`.
3. List full sort order: opaque mesh, transparent mesh, tilemap, sprite, particles, UI.
4. Disable stable sort temporarily — observe flicker, then revert.
5. Log `DrawableSortKey` for two sprites on same layer with different `sortOrder`.

## Summary

Render layers are integer priorities resolved by DrawableSortResolver at submit and applied via stable sorts before 2D pass recording. Pass-type ordering in the frame graph remains independent.

---

Next: [Screen UI](06-screen-ui.md).
