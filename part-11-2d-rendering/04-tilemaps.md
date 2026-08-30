# Tilemaps

> **Build-Along Milestone:** 64×64 tile map renders with O(visible layers) draw calls, not O(total tiles).

**You will learn:**

- Parse tile layers into visible tile instances with chunk frustum culling.
- Submit SceneTilemapDraw headers referencing shared tilemapTiles pool.
- Support multiple layers with sort order and autotile/animation components.

## 1. The Problem We Are Solving

Large 2D worlds are stored as **tile ID grids**, not as giant preauthored meshes. Drawing every cell every frame destroys CPU budget. Production engines **batch visible tiles** into instance pools, cull off-screen chunks, and draw with few instanced calls.

Spark's `SceneTilemapSubmitter` and `VulkanTilemapPass` implement this inside the HDR composite path alongside sprites. Milestone: render a multi-layer map in `TilemapShowcase2DDemo` with chunk culling — visible tile count drops when panning away, draw calls do not explode.

## 1b. Concept Deep Dive — Grid Logic on CPU, Instanced Quads on GPU

### Logical grid vs visible instance pool

A tilemap stores 2D tile IDs per layer — not pre-built meshes. Submit converts **visible** cells into `SceneTilemapTileInstance` records batched under `SceneTilemapDraw` headers with `[tileBegin, tileBegin + tileCount)` into a shared pool:

```cpp
// src/spark/scene/submit/SceneSpriteTileCull.cpp (lines 10–14)
SceneSpriteTileCull::SceneSpriteTileCull(const Matrix4& viewProjection) noexcept
    : frustum(Frustum::FromColumnMajorViewProjection(viewProjection)) {}

bool SceneSpriteTileCull::IsAxisAlignedBoxVisible(const Vector3& boxMin, const Vector3& boxMax) const noexcept {
    return frustum.IntersectsAxisAlignedBox(boxMin, boxMax);
}
```

`CollectVisibleTiles` iterates 16×16 chunks, tests chunk AABB against frustum, then emits only in-chunk non-empty cells — CPU cost scales with visible area, not map dimensions.

### Same unit quad as sprites

`VulkanTilemapPass` expands each tile instance into the shared sprite quad index range (`spriteQuadFirstIndex`). One layer with 200 visible cells becomes one instanced draw, not 200 pipeline binds. Tile ID → atlas UV resolves from tileset margin/spacing at submit.

### HDR composite pass ordering

Tilemaps record inside `composite2DPass` after opaque 3D geometry in the HDR subpass — they depth-test against terrain and write linear HDR color like world sprites. Pan TilemapShowcase2DDemo off-screen and log visible tile count dropping while draw call count stays flat.

## 2. Core Concepts

### Tile grid vs draw instances

A **tilemap** stores a 2D grid of **tile IDs** per layer — not pre-built meshes in the asset file. At submit, Spark converts visible cells into `SceneTilemapTileInstance` records `{gridX, gridY, tileId, transformFlags, tint}` batched under `SceneTilemapDraw` headers. Each draw references a contiguous range `[tileBegin, tileBegin + tileCount)` in the shared `params.tilemapTiles` pool.

Naively drawing one quad per tile per frame costs thousands of draw calls. Spark instead **collects visible tiles** into a pool, sorts them for layer/Y-order, and lets `VulkanTilemapPass` expand each instance into the shared unit quad (same indices as sprites). One tilemap layer with 200 visible cells becomes one instanced draw — not 200 pipeline binds.

SceneTilemapDraw tileBegin/tileCount index into shared pool — corrupt begin index scrambles UVs for whole layer. MaxTilemapTiles stderr warning means silent drop — budget pools for level size.

The logical map is a **2D array of tile IDs**; the GPU sees a flat pool of visible tile instances rebuilt each frame. Editing one cell marks a chunk dirty in editor builds, not the entire world's vertex buffer. Each visible tile becomes one instance with model matrix translating to world cell center, UV rect from tileset, and layer sort keys. Empty cells (ID zero in Tiled convention) never enter the pool — distinguish empty from "tile index zero in tileset" at import time. The indirection layer lets designers paint 4096×4096 maps while draw calls stay proportional to visible area, not total cell count.

Editor builds can maintain a **dirty chunk bitmask** so only touched 16×16 regions re-run collection after a single cell paint. Runtime gameplay rarely edits tiles; static maps amortize import cost at load. When streaming large worlds, each chunk file hydrates a subset of the logical grid — the draw pool still only sees visible instances, not the full world array. Compare memory: logical grid is compact uint16 IDs; instance pool is fat transforms and UVs.


### Tile ID to atlas UV

Tile IDs index into a **tileset** defining atlas cells. Spark resolves UVs in the tilemap shader from atlas grid dimensions, margin, and spacing pixels (`FillAtlasFields` in submit). TMX **gid** values may embed flip flags in high bits — mask with `0x1FFFFFFF` before lookup when importing Tiled maps.

Autotile components (`TilemapAutotileComponent`) rewrite neighbor-dependent IDs at runtime before collection — grass edges vs interior cells. Animated tiles use `TilemapTileAnimatorComponent` to supply `tileAnimationTimeSeconds` passed into `CollectVisibleTiles` for frame selection.

Autotile RebuildIfNeeded runs in submit walk — editing tiles without rebuild shows wrong autotile variants one frame. TMX flip bits in gid require mask before tileset lookup.

Tileset metadata maps **local tile index** to atlas UV bounds, accounting for margin and spacing fields from TMX. Flip flags in Tiled global IDs occupy high bits — strip them before tileset lookup or UVs mirror unpredictably. Autotile rules in Spark demos compute neighbor masks on CPU during map load, not per frame. When multiple tilesets attach to one layer, the submitter resolves texture layer per cell from the tileset handle registered through `findOrAddTexture`. Keep tileset PNGs in the same sRGB workflow as sprite atlases for consistent HDR response after decode.

When importing Tiled JSON, preserve **firstgid** per tileset so local indices map correctly — off-by-one gid bugs show as wrong texture cells, not missing draws. Spacing and margin in `FillAtlasFields` must match the PNG grid Spark artists export; a 2px margin in art with 0 margin in code shifts every UV by a fraction that bleeds on linear filtering. Keep a reference tileset with numbered cells for visual regression tests after importer changes.


### Chunk culling and visibility

`SceneSpriteTileCull::CollectVisibleTiles` iterates **16×16 tile chunks** (default `kDefaultTilemapChunkSize`). For each chunk it builds a world AABB, tests against the view frustum, then emits only in-chunk non-empty cells. Off-screen chunks never append to the tile pool — CPU cost scales with **visible** tiles, not map area.

The same cull helper serves sprite visibility (`IsSpriteVisible`) using transformed unit-quad bounds. Chapter 12 frustum culling extends the same `Frustum` class to 3D mesh draws; 2D tile culling is the first place you see chunk bounds pay off in Spark demos.

Default chunk size 16 — smaller chunks increase cull granularity at cost of loop overhead. Entire map off-screen returns appended=0 early without scanning cells.`SceneSpriteTileCull` builds frustum planes once from `viewProjection` and tests **16×16 tile chunks** before enumerating cells. A chunk's world AABB derives from tile size constants and chunk grid coordinates — misaligned tile world scale breaks cull bounds and pops tiles at screen edges. Sprites use the same cull helper with sprite model AABB for consistency when panning orthographic cameras. Log visible vs total tile counts while panning TilemapShowcase2DDemo; totals should drop smoothly, not stair-step unless chunk boundaries align with camera motion. Combine chunk cull with render layer sorting — culled tiles never reach sort, saving stable sort cost too.

Orthographic cameras produce **axis-aligned frustum tests** that align well with square tile chunks — perspective cameras may include corner chunks whose AABB intersects frustum even when most cells are off-screen, a acceptable conservative bias. Sprite AABB for cull should include non-uniform scale from transform — a flattened coin sprite still needs tight bounds. Log chunk coordinates that fail visibility to diagnose maps placed far from world origin with wrong tile size constants.


### Multiple layers and sort modes

`TilemapComponent` holds multiple named layers (ground, decor, overhead). Each visible layer produces one `SceneTilemapDraw` with sort keys from `DrawableSortResolver` and per-layer `TilemapLayerSortMode` (sort by grid Y for pseudo-3/4 perspective). `StableSortTilemapDraws` insertion-sorts draws so lower `sortingLayerOrder` renders first.

Ground layers typically carry collision via `TileCollisionShape` on tile definitions; decor layers may be non-colliding overlays. Keep **gameplay grid** (`TilemapGameplayGrid`) related to but not identical with render mesh — editing collision should not require rebuilding GPU buffers unless tile IDs change.

StableSortTilemapDraws uses insertion sort like sprites — stable for equal keys. sortWorldYAnchor breaks ties for pseudo-3/4 Y-sort modes.

TMX layers map to **separate submit passes** inside `SceneTilemapSubmitter`, each with its own sort mode and layer order. Background parallax layers may use lower `sortingLayerOrder` than foreground collision decor. Spark supports sort-by-world-Y within a layer for pseudo-depth in top-down maps. Layer visibility toggles in gameplay should skip submit entirely for hidden layers — do not push zero-alpha tiles as a substitute. When two layers share a tileset, texture layer indices still deduplicate through the shared scene texture array from Part IV.

Parallax scrolling often uses **different scroll factors** per layer implemented in gameplay by offsetting camera submission matrix per tilemap entity, not by shader magic. Collision layers should use distinct layer names in TMX and lower sort priority than visuals so debug draws can hide gameplay layers. When one layer uses autotile and another manual tiles, import order matters — autotile neighbor reads need finalized underlying terrain layer IDs.


### SceneTilemapSubmit pipeline

`SceneTilemapSubmitter::Submit` walks active `TilemapComponent` entities:

```text
For each layer:
  cull.

CollectVisibleTiles → append to tilemapTiles pool
  StableSortTilemapTileInstances (Y-order within layer)
  push SceneTilemapDraw header
StableSortTilemapDraws (layer order)
```

`VulkanTilemapPass` records inside `composite2DPass` — typically **before** sprites so character billboards draw on top of ground. Policy varies by game; document your ordering in the frame graph. Tilemap and sprite passes share HDR depth testing from Chapter 1.

TilemapTileAnimatorComponent supplies global time for animated water tiles — sync with gameplay pause by freezing animator time. Invisible layers skipped before collection.

The submitter walks active `TilemapComponent` entities, collects visible tiles per layer, fills `SceneTilemapDraw` headers with atlas dimensions, and pushes instance arrays on `SceneRenderParams`. It shares **composite2DPass** record with sprites — tilemaps draw first or interleave based on sort keys resolved per tile instance. `FillAtlasFields` copies margin, spacing, and tile pixel size so the vertex shader converts tile indices to UVs identically to offline tools. Max tile counts guard with `MaxTiles` — exceeding caps drops tiles silently; log in debug builds when approaching limits during large map streaming tests.

The submitter is stateless each frame — no retained GPU tile cache on CPU beyond component storage. Hot paths are chunk iteration and push_back into instance arrays; profile before micro-optimizing UV math. Tile draw headers carry atlas pixel dimensions so vertex shader converts tile index to UVs without hardcoding atlas sizes in SPIR-V. Multiple tilemap components on different entities merge into one pool sorted globally by layer keys.


### Limits and budgeting

`SceneRenderParams` caps `MaxTilemapDraws` and `MaxTilemapTiles` — exceeding them logs to stderr and stops submission. Large open maps need chunk streaming (Part XII roadmap) or aggressive culling, not larger static pools without bound. Profile `TilemapShowcase2DDemo` with pathfinding overlay enabled to see typical visible tile counts vs pool limits.

Reserve calls reduce realloc — large maps should pre-reserve tile pool in demo Load(). Hitting MaxTilemapDraws aborts layer walk mid-submit.

Budget **CPU tile collection** separately from GPU instance upload. A 256×256 visible region at one tile per instance is 65k instances — still one draw if within SSBO capacity, but memcpy cost hurts. Chunk culling exists to keep visible instances in the low thousands for 60 FPS on laptop integrated GPUs. Tile animation via `TilemapTileAnimatorComponent` updates cell IDs on a timer — only dirty chunks re-collect. Memory for collision grids (`TilemapGameplayGrid`) duplicates logical map data for physics; render pool rebuilds must not reallocate gameplay storage each frame.`MaxTiles` and `MaxSprites` caps protect mobile GPUs — desktop dev builds can raise limits but keep shipping values conservative. Instance SSBO size in bytes equals count times struct stride — calculate headroom against `minStorageBufferOffsetAlignment` and max SSBO range on target hardware. Tile animation that updates many cells per frame shifts cost from GPU to CPU collection — budget animated water regions as small overlays, not entire ocean layers.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Shared tilemapTiles pool

**Concept:** Tile grid vs draw instances

**Decision:** Shared tilemapTiles pool

**Why:** Many layers append into one GPU instance buffer — fewer allocations than per-layer vectors on GPU.

**In Spark:** `include/spark/engine/SceneRenderParams.hpp` — `SceneTilemapTileInstance`.

### FillAtlasFields at submit

**Concept:** Tile ID to atlas UV

**Decision:** FillAtlasFields at submit

**Why:** Tileset margin/spacing handled once per draw header — shader stays generic.

**In Spark:** `SceneTilemapSubmit.cpp` lines 18–36.

### SceneSpriteTileCull chunk loop

**Concept:** Chunk culling and visibility

**Decision:** SceneSpriteTileCull chunk loop

**Why:** Frustum reject entire 16×16 blocks before scanning cells — O(chunks) not O(map).

**In Spark:** `SceneSpriteTileCull.cpp` lines 75–80.

### TilemapLayerSort per layer

**Concept:** Multiple layers and sort modes

**Decision:** TilemapLayerSort per layer

**Why:** Y-sort tiles within layer for overlap; sort draws by registry layer order.

**In Spark:** `src/spark/scene/tilemap/TilemapLayerSort.cpp`.

### Submitter class + VulkanTilemapPass

**Concept:** SceneTilemapSubmit pipeline

**Decision:** Submitter class + VulkanTilemapPass

**Why:** Submit stays pure CPU; pass only records instanced quads.

**In Spark:** `src/spark/scene/SceneTilemapSubmit.cpp` — `Submit` from line 64.

### Fixed MaxTilemapTiles cap

**Concept:** Limits and budgeting

**Decision:** Fixed MaxTilemapTiles cap

**Why:** Prevents runaway memory on malformed maps; forces culling/streaming design early.

**In Spark:** `SceneRenderParams::MaxTilemapTiles` in header.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `src/spark/scene/SceneTilemapSubmit.cpp`
- `src/spark/scene/SceneSpriteTileCull.cpp`
- `src/spark/render/sprites2d/VulkanTilemapPass.cpp`
- `include/spark/demo/TilemapShowcase2DDemo.hpp`

### Step 1 — Run TilemapShowcase2DDemo

Baseline layers, autotile, pathfinding.

### Step 2 — Read FillAtlasFields

Atlas margin/spacing on draw header.

*FillAtlasFields*

```cpp
// src/spark/scene/SceneTilemapSubmit.cpp (lines 18–36)
void FillAtlasFields(const TilemapComponent& tilemap, SceneTilemapDraw& draw) noexcept {
    draw.atlasTilesU = tilemap.GetAtlasTilesU();
    draw.atlasTilesV = tilemap.GetAtlasTilesV();
    if (const SharedPtr<Tileset>& tileset = tilemap.GetTileset(); tileset) {
        draw.atlasMarginPixels = tileset->GetMarginPixels();
        draw.atlasSpacingPixels = tileset->GetSpacingPixels();
        draw.atlasTilePixelWidth = tileset->GetTilePixelWidth();
        draw.atlasTilePixelHeight = tileset->GetTilePixelHeight();
        if (const SharedPtr<Texture2D>& atlas = tileset->GetAtlas(); atlas) {
            if (tileset->GetImagePixelWidth() > 0U && tileset->GetImagePixelHeight() > 0U) {
                draw.atlasTextureWidth = tileset->GetImagePixelWidth();
                draw.atlasTextureHeight = tileset->GetImagePixelHeight();
            } else {
                draw.atlasTextureWidth = atlas->GetWidth();
                draw.atlasTextureHeight = atlas->GetHeight();
            }
        }
    }
}
```

### Step 3 — Trace Submit loop

Per-layer tile collection.

*SceneTilemapSubmit*

```cpp
// src/spark/scene/SceneTilemapSubmit.cpp (lines 102–150)
        for (std::uint32_t layerIndex = 0; layerIndex < tilemap->GetLayerCount(); ++layerIndex) {
            if (params.tilemaps.GetSize() >= SceneRenderParams::MaxTilemapDraws) {
                std::fprintf(
                        stderr,
                        "Spark: tilemap draw limit (%u) reached\n",
                        SceneRenderParams::MaxTilemapDraws);
                return;
            }

            const TilemapLayer& mapLayer = tilemap->GetLayer(layerIndex);
            if (!mapLayer.visible) {
                continue;
            }

            const std::uint32_t remainingTiles =
                    SceneRenderParams::MaxTilemapTiles -
                    static_cast<std::uint32_t>(params.tilemapTiles.GetSize());
            if (remainingTiles == 0) {
                std::fprintf(
                        stderr,
                        "Spark: tilemap tile pool limit (%u) reached\n",
                        SceneRenderParams::MaxTilemapTiles);
                return;
            }

            const std::uint32_t tileBegin = static_cast<std::uint32_t>(params.tilemapTiles.GetSize());
            const std::uint32_t appended = cull.CollectVisibleTiles(
                    worldMatrix,
                    tileWorldSize,
                    *tilemap,
                    layerIndex,
                    params.tilemapTiles,
                    remainingTiles,
                    tileAnimationTimeSeconds);
            if (appended == 0) {
                continue;
            }

            StableSortTilemapTileInstances(
                    params.tilemapTiles,
                    tileBegin,
                    appended,
                    mapLayer.sortMode,
                    worldMatrix,
                    tileWorldSize);

            SceneTilemapDraw draw{};
            draw.worldTransform = worldMatrix;
            draw.tileWorldSize = tileWorldSize;
```

### Step 4 — Study chunk frustum cull

16x16 chunk visibility.

*CollectVisibleTiles*

```cpp
// src/spark/scene/SceneSpriteTileCull.cpp (lines 46–80)
std::uint32_t SceneSpriteTileCull::CollectVisibleTiles(
        const Matrix4& world,
        const float tileWorldSize,
        const TilemapComponent& tilemap,
        const std::uint32_t layerIndex,
        Array<SceneTilemapTileInstance>& outTiles,
        const std::uint32_t maxTiles,
        const float tileAnimationTimeSeconds) const noexcept {
    if (layerIndex >= tilemap.GetLayerCount()) {
        return 0;
    }
    const TilemapLayer& layer = tilemap.GetLayer(layerIndex);
    if (!layer.visible) {
        return 0;
    }
    const std::uint32_t mw = tilemap.GetMapWidth();
    const std::uint32_t mh = tilemap.GetMapHeight();
    if (mw == 0 || mh == 0 || tileWorldSize <= 0.0F) {
        return 0;
    }

    Vector3 mapMin{};
    Vector3 mapMax{};
    ComputeTilemapWorldBounds(world, tileWorldSize, mw, mh, mapMin, mapMax);
    if (!IsAxisAlignedBoxVisible(mapMin, mapMax)) {
        return 0;
    }

    std::uint32_t appended = 0;
    const std::uint32_t chunkSize = kDefaultTilemapChunkSize;
    const std::uint32_t chunksX = (mw + chunkSize - 1U) / chunkSize;
    const std::uint32_t chunksY = (mh + chunkSize - 1U) / chunkSize;
    static constexpr float kChunkDepthPad = 0.05F;

    for (std::uint32_t cy = 0; cy < chunksY; ++cy) {
```

### Step 5 — Build procedural 32x32 map

Log visible tile count while panning.

### Step 6 — Add decor layer

Second layer higher sort order.

### Step 7 — Wire collision separately

TilemapGameplayGrid vs render pool.

### Step 8 — Milestone — O(chunks) not O(tiles)

Draw calls flat when panning.

## 5. How This Connects to the Rest of Your Engine

Tilemaps render in the **HDR scene pass** like sprites (Chapter 1). **2D lighting** applies if you assign lit sprites over tiles — tiles themselves use tilemap shaders. **Frustum cull** (Part XII ch4) shares `Frustum` math with `SceneSpriteTileCull`. **Platformer pipeline** (Chapter 8) submits tilemaps + sprites together.

**Frame graph reminder:** simulation and component updates happen in `OnUpdate`; snapshot fill in `OnRender` or dedicated submit; GPU work in `PresentFrame`. No chapter in Part XI or XII introduces new Vulkan calls from gameplay — only new arrays on the snapshot and record slots in `VulkanRenderer`. Cross-reference Part X HDR ordering whenever you add a draw type: world content belongs in the HDR subpass; display UI belongs after tonemap.

Tilemaps share composite2DPass with sprites (Chapter 1). Frustum chunk cull (this chapter) foreshadows Part XII mesh cull — same frustum math, different granularity. Collision grids feed Part XIV platformer movement — import TMX collision layer separately from visual layers.

## 6. Pitfalls and Debugging

- **Rebuilding entire map on one cell edit** — mark dirty chunks only in editor builds.
- **Tile ID 0 vs empty** — Tiled uses 0 for empty; Spark skips zero cells in collection.
- **Flipped gid bits** — mask flip flags before tileset lookup or UVs mirror wrong.
- **Same sort layer for ground and decor** — z-fighting and alpha fights between layers.
- **Exceeding MaxTilemapTiles** — silent truncation after stderr warning; budget pools for level size.

**Debugging workflow:** when visuals wrong, capture RenderDoc and list (1) pass name, (2) attachment format, (3) bound pipeline, (4) push/SSBO bytes for one failing draw. Compare with Spark capture on the same demo. Validation errors include VUID — search Khronos spec. CPU-side logging of snapshot array sizes (`sprites`, `tilemaps`, `draws`) often finds empty submit before GPU debugging.

Logging visible tile count during pan is the fastest sanity check for broken chunk bounds. Tiled flip bits on gids must strip before autotile neighbor lookup. Max tile cap silent drop is insidious — assert in debug when `PushBack` fails.

## 7. Exercises

1. Render 32×32 procedural map; log visible tile count while panning camera.
2. Add second decor layer; verify draw order via RenderDoc event order.
3. Enable autotile grass — change one wall cell and confirm neighbor variants update.
4. Compare tile draw count with cull disabled vs enabled (temporarily force frustum always visible).
5. Trace `SceneTilemapDraw` pool slice for one layer (`tileBegin`, `tileCount`).

## Summary

Tilemaps collect visible tile instances into a shared pool with chunk frustum culling, sort layers for draw order, and record via VulkanTilemapPass in the HDR composite subpass.

---

Next: [Render Layers](05-render-layers.md).
