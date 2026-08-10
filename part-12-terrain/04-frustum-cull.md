# Frustum Culling

> **Build-Along Milestone:** Visible draw count drops when camera faces away from terrain on TerrainDemo.

**You will learn:**

- Build view frustum from camera view-projection matrix.
- Cull drawable AABBs during SceneSubmit before GPU record.
- Apply shared frustum culling to sprites and tilemap chunks.

## 1. The Problem We Are Solving

Large outdoor terrain and prop counts submit too many draws if every mesh reaches the GPU. **Frustum culling** tests axis-aligned bounds against the camera frustum during **SceneSubmit** — keeping GPU work proportional to visible content, not total placed objects.

Spark builds the frustum once from the same column-major view-projection matrix used for rendering, then tests each drawable AABB before filling `SceneRenderParams`. Sprites and tilemaps reuse that frustum through `SceneSpriteTileCull` so mesh and 2D visibility stay consistent at screen edges.

Milestone: log visible draw count on TerrainDemo — count drops when camera faces sky away from terrain; tilemap visible tile count drops when panning map off-screen.

## 2. Core Concepts

### View frustum from view-projection

**Frustum culling** removes draw items whose bounds fall outside the camera view before snapshot fill. Spark builds `Frustum::FromColumnMajorViewProjection(viewProjection)` once per submit, then tests axis-aligned boxes against six clip planes.

CPU cost is bounded per object (AABB test) instead of paying GPU vertex transform for off-screen meshes. For terrain with high subdiv, culling the entire terrain AABB when looking at sky saves opaque pass work — though conservative bounds may still include large hills when partially visible.

Near and far planes cull objects behind the camera and beyond far clip. Orthographic 2D cameras still have near/far — tilemaps behind the camera should not submit.

### Drawable records and sinks

`DispatchDrawableFrustumCull` gathers **DrawableRecord** entries `{object, mesh, material, world, aabbMin, aabbMax}` then dispatches cull strategy. Visible records flow into **RigidDrawableSubmitSink**, which builds **SceneDrawItem** entries — same packing as the non-culled path.

Skinned meshes use parallel `DispatchSkinnedDrawableFrustumCull` with a separate record stream (Part VI). Policy component **SceneSpatialPolicyComponent** selects partition kind per scene — brute force vs BVH/grid acceleration.

Each record carries world AABB built at submit from mesh bounds transformed by entity matrix. Terrain AABB spans the full heightfield extent — rarely culled unless the camera looks away entirely.

### Brute-force baseline

**BruteForceCull** loops all records testing `frustum.IntersectsAxisAlignedBox` — O(n) simple and correct:

```cpp
if (frustum.IntersectsAxisAlignedBox(r.aabbMin, r.aabbMax)) {
    sink.OnDrawable(r.object, *r.mesh, r.material, r.world);
}
```

Understand brute force before enabling BVH — when object count is below a few hundred, acceleration may not pay for build cost. TerrainDemo with modest prop count often stays on the brute path. Use brute results as golden reference when implementing BVH — visible set must match or be a superset, never miss a visible object.

### Spatial acceleration options

**SceneSpatialCull** implements BVH/grid partitions for static meshes — hide complexity behind `DispatchDrawableFrustumCull` uniform API. Static outdoor props benefit; dynamic skinned crowds may use a separate policy.

False negatives (object pops out while partially visible) mean **AABB too tight** — expand bounds or disable aggressive cull temporarily. Skinned meshes need **conservative** bounds encompassing pose extremes, not rest pose only.

### Sprite and tilemap cull reuse

**SceneSpriteTileCull** shares frustum math with mesh cull — sprites test transformed unit-quad AABB; tilemaps iterate **16×16 chunks** (Part XI). Submit constructs one cull helper from the same `viewProjection` matrix passed to mesh dispatch — matrix mismatch causes pop-in at screen edges.

Log visible vs total tile instances while panning TilemapShowcase2DDemo — off-screen chunks append zero tiles.

### Cull timing in submit

Frustum cull runs **during SceneSubmit** before arrays copy to renderer — not during GPU record, not after `SetSceneRenderParams`. Cull one frame late and GPU still draws last frame's off-screen objects.

Cull after animation update so bounds reflect current pose; cull before sort to save sort cost on invisible sprites. Shadow pass may need a separate cull frustum from light view — do not reuse camera frustum for shadow casters unless intentionally aligned.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Frustum::FromColumnMajorViewProjection

**Concept:** View frustum from view-projection

**Decision:** Frustum::FromColumnMajorViewProjection

**Why:** Single matrix source shared by mesh, sprite, and tile cull — consistent pop-in behavior.

**In Spark:** Used in SceneSpriteTileCull and SceneSpatialCull.

### Sink pattern for submit packing

**Concept:** Drawable records and sinks

**Decision:** Sink pattern for submit packing

**Why:** Cull and non-cull paths share SceneDrawItem builder — no duplicate submit logic.

**In Spark:** `SceneSubmit.cpp` lines 245–260 rigidSink + DispatchDrawableFrustumCull.

### BruteForceCull in SceneSpatialCull.cpp

**Concept:** Brute-force baseline

**Decision:** BruteForceCull in SceneSpatialCull.cpp

**Why:** Reference implementation for validation and small scenes.

**In Spark:** `SceneSpatialCull.cpp` lines 125–132.

### ScenePartitionKind dispatch

**Concept:** Spatial acceleration options

**Decision:** ScenePartitionKind dispatch

**Why:** Scenes opt into BVH via policy component without changing submit call sites.

**In Spark:** `src/spark/scene/SceneSpatialCull.cpp`.

### SceneSpriteTileCull shared helper

**Concept:** Sprite and tilemap cull reuse

**Decision:** SceneSpriteTileCull shared helper

**Why:** One frustum per submit for all 2D visibility — pairs with Part XI chunking.

**In Spark:** `src/spark/scene/SceneSpriteTileCull.cpp`.

### Before params handoff to renderer

**Concept:** Cull timing in submit

**Decision:** Before params handoff to renderer

**Why:** GPU never sees culled items — record cost scales with visible count.

**In Spark:** `DispatchDrawableFrustumCull` call at SceneSubmit line 247.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `src/spark/scene/SceneSubmit.cpp`
- `src/spark/scene/SceneSpatialCull.cpp`
- `src/spark/scene/SceneSpriteTileCull.cpp`

### Step 1 — Find DispatchDrawableFrustumCull call

Frustum cull in submit — locate where rigid draws enter the cull dispatch before `SceneDrawItem` packing. Note `sceneForCulling` guard: null scene skips cull and walks all drawables for debugging comparisons.

*DispatchDrawableFrustumCull*

```cpp
// src/spark/scene/SceneSubmit.cpp (lines 245–260)
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

### Step 2 — Study BruteForceCull

AABB vs frustum loop — the teaching baseline.

*BruteForceCull*

```cpp
// src/spark/scene/SceneSpatialCull.cpp (lines 125–132)
void BruteForceCull(const Array<DrawableRecord>& records, const Frustum& frustum, DrawableFrustumSink& sink) {
    for (std::size_t i = 0; i < records.GetSize(); ++i) {
        const DrawableRecord& r = records[i];
        if (frustum.IntersectsAxisAlignedBox(r.aabbMin, r.aabbMax)) {
            sink.OnDrawable(r.object, *r.mesh, r.material, r.world);
        }
    }
}
```

### Step 3 — Read sprite frustum helper

SceneSpriteTileCull ctor — same viewProj as mesh cull.

*SceneSpriteTileCull*

```cpp
// src/spark/scene/SceneSpriteTileCull.cpp (lines 10–28)
SceneSpriteTileCull::SceneSpriteTileCull(const Matrix4& viewProjection) noexcept
    : frustum(Frustum::FromColumnMajorViewProjection(viewProjection)) {}

bool SceneSpriteTileCull::IsAxisAlignedBoxVisible(const Vector3& boxMin, const Vector3& boxMax) const noexcept {
    return frustum.IntersectsAxisAlignedBox(boxMin, boxMax);
}

void SceneSpriteTileCull::ComputeSpriteWorldBounds(
        const Matrix4& spriteModel, Vector3& outMin, Vector3& outMax) noexcept {
    static constexpr Vector3 kLocalMin{-0.5F, -0.5F, 0.0F};
    static constexpr Vector3 kLocalMax{0.5F, 0.5F, 0.0F};
    AxisAlignedBox::EncapsulateTransformedLocal(kLocalMin, kLocalMax, spriteModel, outMin, outMax);
}

bool SceneSpriteTileCull::IsSpriteVisible(const Matrix4& spriteModel) const noexcept {
    Vector3 bmin{};
    Vector3 bmax{};
    ComputeSpriteWorldBounds(spriteModel, bmin, bmax);
    return IsAxisAlignedBoxVisible(bmin, bmax);
```

### Step 4 — Enable cull logging on TerrainDemo

Visible count vs total — log rigid draws while orbiting. Terrain should stay visible from most angles; props behind the camera should drop from the visible set immediately.

### Step 5 — Skinned cull path

Conservative skinned AABB — expand bounds beyond rest pose if limbs pop out during run cycles.

### Step 6 — Tilemap chunk cull test

Off-screen tiles skipped — pan TilemapShowcase2DDemo until chunks leave view.

### Step 7 — False negative debug

Expand AABB if popping — temporarily inflate bounds on suspect props.

### Step 8 — Milestone — visible count drops

Cull works on TerrainDemo — capture visible/total logs facing sky vs facing hills.

## 5. How This Connects to the Rest of Your Engine

Part **XI** tilemap chunks provide natural cull bounds via `SceneSpriteTileCull`. Part **XII** terrain benefits from mesh-level cull when hills leave view. Part **XVI** profiling compares cull on/off CPU and GPU cost.

Mesh cull complements sprite/tile chunk cull — publish one viewProj matrix per frame before all helpers run. Brute-force cull is the correctness oracle: accelerated partitions in `SceneSpatialCull` must never drop visible objects. Open-world Chapter 5 adds chunk-level granularity; until then TerrainDemo props validate frustum math with modest drawable counts.

## 6. Pitfalls and Debugging

- **AABB too tight on skinned mesh** — limbs culled while torso visible; use conservative bounds.
- **Cull after snapshot copy** — one frame stale visibility; cull only during submit.
- **viewProj mismatch between cull and submit** — pop-in at screen edges; single matrix per frame.
- **Forgetting ortho 2D frustum depth** — tiles behind camera may still submit if depth range wrong.
- **Disabling cull temporarily in shipping** — GPU cost scales with whole world object count.

**Debugging workflow:** capture RenderDoc and compare visible draw count against logged submit totals. Tight AABB on skinned meshes causes pop-in — expand conservative bounds. Cull using stale viewProj pops entire terrain on camera cuts.

## 7. Exercises

1. Log visible/total draws while rotating camera on TerrainDemo.
2. Disable cull temporarily — compare GPU frame time in profiler.
3. Pan 2D tilemap off-screen — verify tile pool append goes to zero.
4. Draw frustum planes debug overlay in ImGui (advanced).
5. Trace `Frustum::FromColumnMajorViewProjection` input matrix source.

## Summary

Frustum culling removes off-screen draw items during SceneSubmit via Frustum AABB tests, with brute-force and accelerated paths plus shared 2D chunk culling.

---

Next: [Streaming Roadmap](05-open-world.md).
