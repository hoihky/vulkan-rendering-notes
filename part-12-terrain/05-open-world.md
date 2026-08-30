# Streaming Roadmap

> **Build-Along Milestone:** Written chunk/LOD plan and baseline TerrainDemo perf numbers in design notes.

**You will learn:**

- Outline clipmap or chunk streaming architecture for large terrains.
- Identify LOD stitching, async load, and physics alignment integration points.
- Keep TerrainDemo as the non-streaming teaching and regression baseline.

## 1. The Problem We Are Solving

Single-patch **TerrainMeshGenerator** suffices for demos; shipping open worlds need **streaming** height tiles, LOD rings, and async IO. This chapter maps the production extension path without requiring full implementation in the learning engine.

Milestone: written chunk/LOD plan plus baseline vertex count and RegenerateMesh timing from TerrainDemo.

## 1b. Concept Deep Dive — From Single Patch to Streaming Design

### TerrainDemo is the non-streaming regression baseline

Spark ships one `TerrainMeshGenerator` patch — sufficient for Parts IX–XII integration. Memory and vertex cost grow with subdiv²; a 2048² grid is millions of triangles in one draw. Open worlds partition XZ into **fixed-size chunks**, each with independent height arrays and draw items:

```text
Per chunk (production extension):
  heightSamples[] for cell resolution
  MeshHandle / Custom slot registration
  disk path (optional compressed R16)
  lastUsed frame for LRU eviction
```

Master single-patch brush/regenerate/cull in TerrainDemo before implementing clipmaps — streaming doubles debug surface area.

### LOD rings need skirt geometry

Discrete chunk LOD (near 128², far 32² vertices) or GPU clipmap rings both require **skirt indices** or vertical strips between mismatched edge vertex counts — without skirts, T-junction cracks leak sunlight at ring boundaries. Load radius vs unload radius hysteresis prevents chunk thrashing at cell borders.

### Async IO stays off the gameplay thread

Height tile decode on a worker thread → main thread registers mesh into Custom pool → GPU sees new draw item next frame. Physics colliders must align with streamed height — rebasing world origin when the camera roams far from spawn avoids floating-point jitter at large coordinates.

## 2. Core Concepts

### Single-patch vs open world

Spark **TerrainDemo** is one **TerrainMeshGenerator** patch — sufficient for learning Parts IX–XII integration. Shipping open worlds cannot hold full-resolution heightfields for entire continents in memory. **Streaming** loads height **tiles** near the player and unloads distant tiles under a memory budget.

Single-patch memory and vertex cost grow with subdiv squared — a 2048² grid is millions of triangles in one draw. Open worlds partition XZ into **fixed-size chunks**, each with independent height arrays and draw items. Generator code from Chapter 1 runs per chunk with shared settings but different world-origin offsets.

This chapter is a **roadmap** — design on paper and measure baseline perf from TerrainDemo before implementing clipmaps or chunk IO. Master single-patch brush/regenerate/cull first; streaming doubles debug surface area.

### Chunk grid world partition

Split world XZ into fixed-size cells (e.g. **256 m × 256 m**). Each cell owns:

```text
heightSamples[] for cell resolution
MeshHandle or Custom mesh slot registration
disk file path (optional compressed R16)
lastUsed frame for LRU eviction
```

Camera cell coordinate updates each frame; load missing neighbors within **load radius**; unload cells beyond **unload radius** (use hysteresis — unload radius > load radius to avoid thrashing).

World coordinates map to **chunk indices** via floor division on XZ. Load radius 2 implies up to 25 active patches in a 5×5 ring. Neighbor chunks share skirt geometry or stitched edge indices to hide T-junction cracks. Chunk origin at corner vs center affects floating precision — rebasing world origin when the camera roams far from spawn is standard open-world practice.

### LOD rings and clipmaps

**Geometric clipmap** (GPU Gems terrain chapter) centers concentric grids on camera XZ with **high subdiv near**, **low subdiv far**. Alternative: discrete chunk LOD levels per cell distance — near cell 128², mid 64², far 32² vertices.

**LOD stitching** requires **skirt geometry** or vertical strips between mismatched edge vertex counts — without skirts, **T-junction cracks** show sunlight leaks at ring boundaries. Budget skirt indices in the mesh builder when adjacent cells differ LOD.

Transition pops if LOD switches without geomorph — advanced implementations blend heights over time. Clipmap toroidal buffer shifts each frame are powerful for flight sims; chunk LOD rings are simpler for RPG-scale worlds.

### Async IO and main thread

Decode height tiles on **worker threads** (Part XVI async patterns); the main thread registers mesh when ready — **no stall in PresentFrame**. `SceneManager::BeginLoadSceneAsync` demonstrates async scene load Spark may extend for terrain tiles.

Worker threads return byte buffers; main thread parses and calls `RegenerateMesh` to keep Vulkan submission thread-safe. Limit in-flight loads to queue depth 2–3 to avoid memory spikes when the camera teleports. Failed loads should leave collision blocking or a flat placeholder chunk — never a walkable hole without gameplay handling.

The main thread must not read half-written height buffers — use a ready flag or immutable swap after decode completes.

### Physics alignment

**Collision heightfield** must match visual LOD **near the player** — simplified far collision is optional if gameplay allows. Part XIV 3D physics heightfield shapes should sample the highest-resolution loaded cell under the character's feet.

Player falling through an unloaded chunk is a classic streaming bug — keep collider loaded **one cell beyond** visual unload radius or block movement at stream boundaries. Heightfield collision samples bilinear between grid vertices when using the same heights array as render — divergence causes foot sliding.

Large world coordinates lose float precision — periodic origin shifts must move physics and rendering roots together.

### Baseline perf from TerrainDemo

Before designing streaming, log from the current generator:

```text
vertexCount = GridVertexCountX × GridVertexCountZ
triangleCount ≈ 2 × (subdivX × subdivZ)
RegenerateMesh ms per brush stroke
Opaque draw GPU ms with SSAO/shadows
```

Use **TerrainMeshGenerator::HeightSampleCount** and grid helpers for budgeting. If a single patch already exceeds frame budget, streaming LOD is mandatory, not optional. Record baseline opaque-pass ms, shadow-pass ms, and visible drawable ratio before the streaming design review — numbers anchor scope conversations.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Keep TerrainDemo as teaching baseline

**Concept:** Single-patch vs open world

**Decision:** Keep TerrainDemo as teaching baseline

**Why:** Non-streaming path remains regression target while streaming develops behind flag.

**In Spark:** `include/spark/demo/TerrainDemo.hpp`.

### Fixed world cell size

**Concept:** Chunk grid world partition

**Decision:** Fixed world cell size

**Why:** Predictable memory: peak loaded = (2×loadRadius+1)² cells × bytes per height.

**In Spark:** Roadmap extension — not required in baseline Spark.

### Industry clipmap reference

**Concept:** LOD rings and clipmaps

**Decision:** Industry clipmap reference

**Why:** Concentric grids centered on camera XZ — proven outdoor engine pattern.

**In Spark:** GPU Gems terrain clipmaps chapter (reference reading).

### Part XVI worker decode

**Concept:** Async IO and main thread

**Decision:** Part XVI worker decode

**Why:** Pair terrain tile load with existing async asset infrastructure.

**In Spark:** `SceneManager` async load patterns.

### Near-field full-res collider

**Concept:** Physics alignment

**Decision:** Near-field full-res collider

**Why:** Gameplay truth matches highest LOD ring only — far simplified acceptable.

**In Spark:** Part XIV PhysicsWorld3D heightfield when added.

### GridVertexCount helpers for budgeting

**Concept:** Baseline perf from TerrainDemo

**Decision:** GridVertexCount helpers for budgeting

**Why:** Quantify vertex/triangle counts before designing chunk sizes.

**In Spark:** `TerrainMeshGenerator.cpp` lines 58–70.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `include/spark/demo/TerrainDemo.hpp`
- `src/spark/scene/TerrainMeshGenerator.cpp`
- `include/spark/scene/TerrainGeneratorSettings.hpp`

### Step 1 — Measure current terrain cost

Grid vertex count helpers.

*budget helpers*

```cpp
// src/spark/scene/TerrainMeshGenerator.cpp (lines 58–70)
std::int32_t TerrainMeshGenerator::GridVertexCountX(const TerrainGeneratorSettings& settings) noexcept {
    return (std::max)(std::int32_t{2}, settings.subdivX) + 1;
}

std::int32_t TerrainMeshGenerator::GridVertexCountZ(const TerrainGeneratorSettings& settings) noexcept {
    return (std::max)(std::int32_t{2}, settings.subdivZ) + 1;
}

std::size_t TerrainMeshGenerator::HeightSampleCount(const TerrainGeneratorSettings& settings) noexcept {
    const std::int32_t nx = GridVertexCountX(settings);
    const std::int32_t nz = GridVertexCountZ(settings);
    return static_cast<std::size_t>(nx) * static_cast<std::size_t>(nz);
}
```

### Step 2 — Time RegenerateMesh

Profile brush rebuild ms at demo subdiv.

### Step 3 — Design chunk grid on paper

Cell size and load radius for a 4×4 km world.

### Step 4 — Camera cell tracking spec

Load/unload pseudocode with hysteresis gap.

### Step 5 — Skirt index plan

LOD crack prevention at ring boundaries.

### Step 6 — Async IO integration points

Worker decode, main thread mesh upload.

### Step 7 — Keep demo path simple

Single-patch TerrainDemo baseline behind `enableTerrainStreaming` flag.

### Step 8 — Milestone — written streaming plan

Chunk LOD plan documented with perf numbers.

## 5. How This Connects to the Rest of Your Engine

Part **XVI** async assets supply worker decode. Part **XIII** scene format may reference terrain tile file paths. Part **IX** CSM applies per visible chunk. Chapter 4 **frustum cull** reduces work per loaded chunk.

**Frame graph reminder:** simulation and component updates happen in `OnUpdate`; snapshot fill in `OnRender` or dedicated submit; GPU work in `PresentFrame`. No chapter in Part XI or XII introduces new Vulkan calls from gameplay — only new arrays on the snapshot and record slots in `VulkanRenderer`. Cross-reference Part X HDR ordering whenever you add a draw type: world content belongs in the HDR subpass; display UI belongs after tonemap.

Streaming extends TerrainComponent `RegenerateMesh` and frustum cull — no replacement renderer. Part XI tilemaps already chunk — borrow that mental model for terrain chunks.

## 6. Pitfalls and Debugging

- **Implementing streaming before single patch works** — debug two systems at once.
- **LOD cracks without skirts** — visible seams at ring boundaries in outdoor sun.
- **Physics vs render mismatch** — player falls through unloaded chunk collider.
- **Loading on main thread** — hitches PresentFrame; decode off-thread.
- **Unload radius == load radius** — thrashing at boundary; hysteresis gap required.

**Debugging workflow:** when visuals wrong, capture RenderDoc and list (1) pass name, (2) attachment format, (3) bound pipeline, (4) push/SSBO bytes for one failing draw. Compare with Spark capture on the same demo. Validation errors include VUID — search Khronos spec. CPU-side logging of snapshot array sizes (`sprites`, `tilemaps`, `draws`) often finds empty submit before GPU debugging.

## 7. Exercises

1. Sketch chunk grid for 4×4 km world with 256 m cells.
2. List data each chunk needs on disk vs in GPU memory.
3. Estimate peak memory for 3×3 loaded cells at two LOD levels.
4. Compare `RegenerateMesh` ms at subdiv 64 vs 256 — extrapolate streaming need.
5. Document skirt geometry purpose at LOD boundaries.

## Summary

Spark's single-patch TerrainDemo is the teaching baseline; production open worlds add chunk streaming, LOD rings, async IO, and physics alignment — plan on paper before implementing.

---

Continue to [Part XIII — Scene and Editor](../part-13-scene-editor/index.md).
