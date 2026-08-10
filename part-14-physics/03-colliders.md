# Colliders

> **Build-Along Milestone:** Player box collider aligns with tilemap floor; ground raycast gates jump reliably.

**You will learn:**

- Author 2D and 3D collider components on entities.
- Understand bake pipeline from components to runtime collider arrays.
- Bake tilemap collision grids for platformers.
- Use physics queries for gameplay ground checks and overlaps.

## 1. The Problem We Are Solving

Physics simulation consumes **flat collider arrays**, not ECS components directly. Spark rebuilds those arrays every Simulate from a **bake pipeline** — boxes, circles, tilemap grids, 3D primitives. Materials tune contact response; queries answer gameplay questions (grounded? hit pickup?) without stepping full dynamics.

You will author colliders on player and tilemap, tune materials, and verify debug alignment with sprites.

Milestone: player box collider feet align with tilemap floor; ground ray gates jump reliably.

## 2. Core Concepts

### Collider components vs baked colliders

Authoring uses ECS components: `BoxCollider2DComponent`, `CircleCollider2DComponent`, `TilemapCollider2DComponent`, `BoxCollider3DComponent`, etc. Simulation uses flat `Collider2D` / `Collider3D` arrays rebuilt each step. **Bake pipeline** translates components → runtime colliders with world-space AABBs, categories, trigger flags, and material indices.

Editing a component in inspector affects next Simulate rebuild — no manual "rebake" button required.

### 2D primitive shapes

Box and circle colliders attach offsets and sizes in local space; bake applies entity transform. Dynamic bodies require matching rigidbody + at least one shape — integrator skips dynamics without collider.

Align box half-extents with sprite visual feet — platformer feel depends more on collider than mesh bounds.

### Tilemap collision bake

`TilemapCollider2DComponent` expands walkable/blocked tiles into many axis-aligned box colliders — efficient for platform levels versus spawning one entity per tile. `TilemapCollisionBake` transforms normalized cell corners through the tilemap entity's world matrix, builds a world-space AABB per solid cell, and appends box colliders to the bake context.

Tile flip/rotate flags adjust normalized corner positions before world projection — a flipped tile collision must match flipped render tile (Part XI) or players snag invisible lips at chunk boundaries.

### Physics materials

`PhysicsMaterial2DComponent` / `PhysicsMaterial3DComponent` store friction, restitution, and combine modes. Contact resolver reads material params at impact — tune ground vs ice vs rubber ball. PhysicsBallThrow3DDemo documents restitution combine rule between body and surface materials.

### Physics queries

`PhysicsQueryWorld2D` rebuilds static broadphase and exposes raycast, overlap circle, AABB overlap without full dynamics step. Platformer jump uses **ground check raycast** down short distance each frame.

Query layer masks use same `CollisionFilter2D::ShouldCollide` rules as simulation — consistent behavior between "what blocks movement" and "what ray hits".

| Need | API | Dynamics? |
|------|-----|-----------|
| Ground check | RaycastStatics | no |
| Pickup overlap | OverlapCircleStatics | no |
| Move + collide | Simulate2D | yes |

### Sensors vs solids

Trigger colliders participate in overlap queries and trigger signals but should not block motion unless your design intentionally uses solid triggers. `PhysicsQueryFilter2D::hitTriggers` and `hitSolids` flags separate ground checks from pickup overlap scans.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### ColliderBakePipeline2D/3D strategy registry

**Concept:** Collider components vs baked colliders

**Decision:** ColliderBakePipeline2D/3D strategy registry

**Why:** Each shape type registers IColliderBakeStrategy; Rebuild walks all active objects.

**In Spark:** `ColliderBakePipeline2D::Rebuild`

### Component-local offsets + half extents

**Concept:** 2D primitive shapes

**Decision:** Component-local offsets + half extents

**Why:** Artist-friendly authoring; bake applies world matrix.

**In Spark:** `BoxCollider2DComponent` / `CircleCollider2DComponent` headers

### TilemapCollisionBake module

**Concept:** Tilemap collision bake

**Decision:** TilemapCollisionBake module

**Why:** Static grid from tile layer — broadphase-friendly for large maps.

**In Spark:** `src/spark/physics/TilemapCollisionBake.cpp`

### PhysicsMaterial2D/3D components

**Concept:** Physics materials

**Decision:** PhysicsMaterial2D/3D components

**Why:** Per-entity tuning; resolver reads at contact time.

**In Spark:** `PhysicsMaterial3DComponent` on ground in BallThrow demo

### PhysicsQueryWorld2D on PhysicsSubsystem

**Concept:** Physics queries

**Decision:** PhysicsQueryWorld2D on PhysicsSubsystem

**Why:** Shared cell size with simulation broadphase.

**In Spark:** `include/spark/physics/PhysicsQueries2D.hpp`

### Collider trigger flag in bake output

**Concept:** Sensors vs solids

**Decision:** Collider trigger flag in bake output

**Why:** Simulation and queries respect filter bits consistently.

**In Spark:** `PhysicsQueryFilter2D` struct

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `src/spark/physics/colliders/ColliderBakePipeline2D.cpp`
- `src/spark/physics/TilemapCollisionBake.cpp`
- `include/spark/physics/PhysicsQueries2D.hpp`
- `include/spark/ecs/components/physics/2d/TilemapCollider2DComponent.hpp`

### Step 1 — Read bake pipeline Rebuild

Strategies iterate active objects; each Contributes + Bake appends colliders.

*ColliderBakePipeline2D::Rebuild*

```cpp
// src/spark/physics/colliders/ColliderBakePipeline2D.cpp (lines 21–41)
void ColliderBakePipeline2D::Rebuild(
        GameWorld& world, const float cellWorldSize,
        Array<Collider2D>& outColliders, SpatialHashGrid2D& outGrid) {
    outColliders.Clear();
    outGrid.Clear();
    outGrid.SetCellSize(cellWorldSize);
    ColliderBakeContext2D context{outColliders, outGrid};
    world.ForEachActiveGameObject([&](GameObject* object) {
        for (std::size_t i = 0; i < strategies.GetSize(); ++i) {
            const IColliderBakeStrategy2D& strategy = *strategies[i];
            if (strategy.Contributes(*object)) {
                strategy.Bake(*object, context);
            }
        }
    });
}
```

### Step 2 — Study tilemap corner to world AABB

Tile cell corners transform through world matrix for collision boxes.

*Local tile point to world AABB*

```cpp
// src/spark/physics/TilemapCollisionBake.cpp (lines 28–58)
void LocalTilePointToWorld(
        const Matrix4& worldMatrix, const float tileWorldSize,
        const std::uint32_t tileX, const std::uint32_t tileY,
        const Vector2& normalized, Vector4& outWorldH) noexcept {
    const float lx = (static_cast<float>(tileX) + normalized.x) * tileWorldSize;
    const float ly = (static_cast<float>(tileY) + normalized.y) * tileWorldSize;
    outWorldH = worldMatrix * Vector4(lx, ly, 0.0F, 1.0F);
}
```

### Step 3 — Add BoxCollider2D to platformer player

Size matches sprite feet; pair with Rigidbody2DComponent dynamic body.

### Step 4 — Bake tilemap on map entity

Add `TilemapCollider2DComponent` to tilemap entity from Part XI — same entity as render `TilemapComponent`.

### Step 5 — Tune landing material

Low restitution on player-ground contacts — reduce bounce on platform landing.

### Step 6 — Ground check raycast

```cpp
PhysicsQueryFilter2D filter{};
filter.hitSolids = true;
filter.hitTriggers = false;
// short downward ray; gate jump on hit
```

Match ray length to player capsule radius — too short misses ground on slope edges.

### Step 7 — Debug draw collider outlines

Optional line pass overlays AABBs — compare feet alignment before shipping level geometry.

### Step 8 — 3D ball materials

*Ground physics material restitution*

```cpp
// src/spark/demo/PhysicsBallThrow3DDemo.cpp (lines 48–51)
ground->AddComponent<Spark::BoxCollider3DComponent>(Spark::Vector3{hx, hy, hx}, Spark::Vector3{0.0F, hy, 0.0F});
ground->AddComponent<Spark::Rigidbody3DComponent>(Spark::RigidbodyBodyType3D::Static, 1.0F);
ground->AddComponent<Spark::PhysicsMaterial3DComponent>(0.62F, 0.48F, 0.58F);
```

## 5. How This Connects to the Rest of Your Engine

Part **XI** tilemaps supply render grid; same entity hosts TilemapCollider2D. Part **XIII** serializes collider dimensions in scene snapshots. Part **XV** scripting may expose raycast helpers via interop. BroadPhase2DDemo stress-tests many static colliders from bake output.

Empty tiles produce no collider — verify art export marks holes as empty, not invisible collision tiles.

## 6. Pitfalls and Debugging

- **Collider smaller than sprite** — visible float or interpenetration at feet.
- **Forgot tilemap rebake after edit** — walk through walls until next full reload.
- **Sensor vs solid confusion** — triggers should not block unless intended.
- **Query mask mismatch** — ray ignores ground layer bit; jump never enables.
- **3D box not centered on mesh** — offset parameter wrong in BoxCollider3DComponent.

**Debugging workflow:** debug-draw collider AABB after Simulate; compare sprite feet Y vs collider bottom Y; adjust collider offset until all three agree within 2px.

## 7. Exercises

1. Align box collider with sprite feet using debug draw; screenshot overlay aligned within 2px.
2. Implement ground raycast gating jump; log ray hit distance each airborne frame.
3. Compare ball restitution 0.2 vs 0.9 materials; measure apex height after three bounces.
4. Count baked colliders in a 32×32 solid tilemap region; report RebuildStatics ms.
5. Place trigger sensor coin; verify overlap signal fires without blocking player horizontal motion.

## Summary

Colliders are ECS components baked each physics step into runtime arrays. Spark supports primitives, tilemap grids, materials, and query API sharing broadphase settings with simulation.

---

Next: [Sync Transforms](04-physics-render.md).
