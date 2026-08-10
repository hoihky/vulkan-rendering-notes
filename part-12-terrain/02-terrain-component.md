# TerrainComponent

> **Build-Along Milestone:** Runtime brush editing updates lit terrain mesh in TerrainDemo.

**You will learn:**

- Attach TerrainComponent with generator settings and pair MaterialComponent.
- Apply height brushes and regenerate mesh through RegenerateMesh.
- Raycast against terrain for editor and runtime sculpt tooling.

## 1. The Problem We Are Solving

Chapter 1 generates a static heightfield mesh. Games need **runtime ownership** — a height buffer you can sculpt, a rebuild trigger after edits, and ray picks for cursor placement. **TerrainComponent** wraps `TerrainMeshGenerator`, owns the height samples, pushes results into the Custom mesh slot, and exposes brush and raycast APIs used by TerrainDemo.

Milestone: raise a hill with the world-space brush in TerrainDemo and see lighting update on the new slope immediately.

## 2. Core Concepts

### Component ownership model

**TerrainComponent** owns three things:

```text
TerrainGeneratorSettings  — subdiv, extents, noise params
Array<float> heightSamples — one scalar per grid vertex
Vector3 meshAlbedo         — fallback tint when no MaterialComponent
```

Geometry and height data live on the component; shading parameters live on sibling **MaterialComponent** (Part VII pattern). Artists add `MaterialComponent` for grass/rock ORM maps; brush APIs mutate heights only. This mirrors static meshes: `MeshComponent` + `MaterialComponent` on one GameObject.

Serialization should store **heights + generator settings**, not the generated mesh — reload regenerates topology on boot. Multiple terrain entities are supported; each carries independent height buffers and draw items.

### RegenerateMesh pipeline

Single rebuild entry point after any height change:

```text
EnsureHeightBuffer → BuildIntoFromHeights → MeshComponent SetMesh(Custom)
```

`EnsureHeightBuffer` refills procedural fBM only when sample count mismatches settings subdiv. Every brush stroke calls `RegenerateMesh` in demo resolution — acceptable for learning; shipping open worlds chunk regeneration (Chapter 5).

`RegenerateMesh` builds a fresh `Mesh`, calls `TerrainMeshGenerator::BuildIntoFromHeights`, then either adds `MeshComponent` with `SceneMeshSlot::Custom` or updates an existing one. `OnAttach` triggers the first build so the entity is drawable the frame it enters the world.

### Height brush APIs

**`ApplyHeightBrushLocal`** modifies heights in the object XZ plane with radius in local units. **`ApplyHeightBrushWorld`** transforms center and radius from world space using the object's horizontal scale — use when picking from a camera raycast.

Falloff uses smoothstep by distance from brush center:

```cpp
const float t = 1.0F - dist / radiusXZ;
const float w = t * t * (3.0F - 2.0F * t);  // smoothstep
heightSamples[idx] += deltaY * w;
```

Positive `deltaY` raises hills; negative carves valleys. Multiple strokes accumulate. World brush scales radius by `max(sx, sz)` from the object transform columns so a 10 m world radius stays circular on scaled terrain.

### Raycast for tooling

**`TryRaycastWorld`** intersects a world ray against heightfield triangles built from current height samples and the object transform. The ray is transformed to local space; each quad is tested as two triangles with Möller–Trumbore intersection. Returns the closest hit within `maxDistance`.

Without raycast, brush center floats at wrong XZ — visible as circular edits offset from the cursor on slopes. TerrainDemo uses camera forward when the cursor is captured, or screen-to-world ray when the cursor is free.

### Reset and procedural fallback

**`ResetHeightsToProcedural`** clears samples and reruns `FillProceduralHeights` — undoes experimental sculpts during demo iteration (bound to **R** in TerrainDemo). **`GetHeightSamples`** exposes the read-only buffer for serialization (Part XIII scene format may store a binary height chunk per terrain entity).

Large arrays should not JSON-serialize naively — plan compressed binary or external `.r16` tile files for production maps.

### Material pairing on entity

TerrainDemo spawns an entity with **TerrainComponent** + **MaterialComponent**:

```cpp
terrainObject->AddComponent<TerrainComponent>(ts);
MaterialComponent* m = terrainObject->AddComponent<MaterialComponent>(...);
```

Generator supplies positions, normals, and planar UVs; material supplies texture layers, metallic/roughness, and normal strength. Shadow cast/receive flags come from draw item defaults — enable receive so props cast shadows onto ground (Chapter 3).

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### TerrainComponent OnAttach regenerate

**Concept:** Component ownership model

**Decision:** TerrainComponent OnAttach regenerate

**Why:** Guarantees valid mesh first frame entity is active — no lazy-init footgun in demos.

**In Spark:** `TerrainComponent.cpp` lines 20–22.

### Single rebuild function

**Concept:** RegenerateMesh pipeline

**Decision:** Single rebuild function

**Why:** Brush, reset, and attach all share upload path — consistent custom pool behavior.

**In Spark:** `TerrainComponent::RegenerateMesh` lines 37–49.

### Local vs world variants

**Concept:** Height brush APIs

**Decision:** Local vs world variants

**Why:** World API matches editor camera picks; local API matches procedural tools in object space.

**In Spark:** `TerrainComponent.hpp` lines 47–51.

### Triangle walk on height grid

**Concept:** Raycast for tooling

**Decision:** Triangle walk on height grid

**Why:** Accurate on slopes without analytical heightfield ray-march approximations.

**In Spark:** `TerrainComponent::TryRaycastWorld` in cpp.

### ResetHeightsToProcedural

**Concept:** Reset and procedural fallback

**Decision:** ResetHeightsToProcedural

**Why:** One-call return to demo baseline after sculpt experiments.

**In Spark:** `TerrainComponent.hpp` line 31.

### Separate MaterialComponent

**Concept:** Material pairing on entity

**Decision:** Separate MaterialComponent

**Why:** Same split as all meshes — terrain is not a special shading case.

**In Spark:** TerrainDemo entity setup.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `include/spark/ecs/components/rendering/TerrainComponent.hpp`
- `src/spark/ecs/components/rendering/TerrainComponent.cpp`
- `include/spark/demo/TerrainDemo.hpp`

### Step 1 — Read TerrainComponent public API

Brush, raycast, reset API.

*TerrainComponent API*

```cpp
// include/spark/ecs/components/rendering/TerrainComponent.hpp (lines 17–51)
class TerrainComponent final : public GameComponent {
public:
    static constexpr ComponentKind TypeKind = ComponentKind::Terrain;

    [[nodiscard]] ComponentKind Kind() const noexcept override { return TypeKind; }

    explicit TerrainComponent(TerrainGeneratorSettings settings, Vector3 meshAlbedo = Vector3{0.42F, 0.55F, 0.36F});

    void OnAttach(GameObject& owner) override;

    [[nodiscard]] const TerrainGeneratorSettings& GetSettings() const noexcept { return settings; }
    [[nodiscard]] const Array<float>& GetHeightSamples() const noexcept { return heightSamples; }

    /** Re-samples fBM into the height buffer and rebuilds the mesh. */
    void ResetHeightsToProcedural(GameObject& owner);

    /** Rebuilds mesh from the current height buffer (call after brush edits). */
    void RegenerateMesh(GameObject& owner);

    /**
     * Ray vs heightfield triangles in object-local space (terrain lies in XZ; Y is up).
     * @param rayDirWorld should be a direction (any non-zero length; it is normalized).
     */
    [[nodiscard]] bool TryRaycastWorld(
            const GameObject& owner,
            Vector3 rayOriginWorld,
            Vector3 rayDirWorld,
            float maxDistance,
            Vector3& outHitWorld) const;

    /** Smooth additive brush on the height grid (local XZ plane; radius in local units). */
    void ApplyHeightBrushLocal(GameObject& owner, Vector2 centerXZ, float radiusXZ, float deltaY);

    /** Transforms @p centerWorld to local XZ; scales radius from world using max horizontal scale of the object. */
    void ApplyHeightBrushWorld(GameObject& owner, Vector3 centerWorld, float radiusWorld, float deltaY);
```

### Step 2 — Study OnAttach and RegenerateMesh

Height ensure and mesh rebuild.

*RegenerateMesh*

```cpp
// src/spark/ecs/components/rendering/TerrainComponent.cpp (lines 20–49)
void TerrainComponent::OnAttach(GameObject& owner) {
    RegenerateMesh(owner);
}

void TerrainComponent::EnsureHeightBuffer(GameObject& /*owner*/) {
    const std::size_t need = TerrainMeshGenerator::HeightSampleCount(settings);
    if (heightSamples.GetSize() != need) {
        TerrainMeshGenerator::FillProceduralHeights(settings, heightSamples);
    }
}

void TerrainComponent::ResetHeightsToProcedural(GameObject& owner) {
    heightSamples.Clear();
    TerrainMeshGenerator::FillProceduralHeights(settings, heightSamples);
    RegenerateMesh(owner);
}

void TerrainComponent::RegenerateMesh(GameObject& owner) {
    EnsureHeightBuffer(owner);
    SharedPtr<Mesh> mesh = MakeShared<Mesh>(Utf8String("ProceduralTerrain"));
    TerrainMeshGenerator::BuildIntoFromHeights(*mesh, settings, heightSamples);

    MeshComponent* mc = owner.GetComponent<MeshComponent>();
    if (mc == nullptr) {
        owner.AddComponent<MeshComponent>(mesh, SceneMeshSlot::Custom, meshAlbedo);
    } else {
        mc->SetMesh(mesh);
        mc->SetAlbedo(meshAlbedo);
    }
}
```

### Step 3 — Study brush falloff

Smoothstep kernel in local brush.

*ApplyHeightBrushLocal*

```cpp
// src/spark/ecs/components/rendering/TerrainComponent.cpp (lines 149–179)
void TerrainComponent::ApplyHeightBrushLocal(GameObject& owner, Vector2 centerXZ, float radiusXZ, float deltaY) {
  // ...
            const float t = 1.0F - dist / radiusXZ;
            const float w = t * t * (3.0F - 2.0F * t);
            const std::size_t idx = static_cast<std::size_t>(iz * nx + ix);
            heightSamples[idx] += deltaY * w;
  // ...
    if (changed) {
        RegenerateMesh(owner);
    }
}
```

### Step 4 — Spawn terrain entity

TerrainComponent + MaterialComponent in TerrainDemo `Load`.

### Step 5 — Test brush edit

`ApplyHeightBrushWorld` at raycast hit — LMB raises, RMB lowers in TerrainDemo `Simulate`.

### Step 6 — Raycast from camera

`TryRaycastWorld` drives edit cursor position and brush center.

### Step 7 — ResetHeightsToProcedural

Press **R** to return to fBM baseline.

### Step 8 — Milestone — runtime sculpt

Brush updates lit mesh live.

## 5. How This Connects to the Rest of Your Engine

Part **VII** materials apply to terrain draws. Part **XIII** editor may call brush/raycast from viewport. Part **IX** shadows require cast/receive on terrain entity. Chapter 3 integrates terrain into the standard lit opaque pass.

**Frame graph reminder:** simulation and component updates happen in `OnUpdate`; snapshot fill in `OnRender` or dedicated submit; GPU work in `PresentFrame`. No chapter in Part XI or XII introduces new Vulkan calls from gameplay — only new arrays on the snapshot and record slots in `VulkanRenderer`. Cross-reference Part X HDR ordering whenever you add a draw type: world content belongs in the HDR subpass; display UI belongs after tonemap.

`RegenerateMesh` triggers the same Custom slot upload as Chapter 1 initial build. Material from Part VII determines final look — geometry alone is gray in the scene pass. Brush sculpting is synchronous in Spark baseline — treat `RegenerateMesh` milliseconds as a hard UX budget when planning editor features.

## 6. Pitfalls and Debugging

- **Regenerating without updating MeshComponent** — stale CPU mesh until RegenerateMesh completes.
- **Brush radius in wrong space** — world vs local mismatch offsets edit disc.
- **Missing MaterialComponent** — flat albedo only from component default color.
- **Full mesh rebuild every stroke at high subdiv** — frame hitch; plan chunking for production.
- **Raycast maxDistance too short** — misses far terrain under shallow grazing angles.

**Debugging workflow:** when visuals wrong, capture RenderDoc and list (1) pass name, (2) attachment format, (3) bound pipeline, (4) push/SSBO bytes for one failing draw. Compare with Spark capture on the same demo. Validation errors include VUID — search Khronos spec. CPU-side logging of snapshot array sizes (`sprites`, `tilemaps`, `draws`) often finds empty submit before GPU debugging.

Log `heightSamples.GetSize()` vs `HeightSampleCount(settings)` after every brush release in development — silent early-return from size mismatch looks like a broken brush when code is correct but counts diverged.

## 7. Exercises

1. Sculpt a hill with brush; verify lighting updates on slopes.
2. `ResetHeightsToProcedural` after edits.
3. Raycast from camera click; log hit world position.
4. Time `RegenerateMesh` for 256×256 subdiv — record ms for Chapter 5 budget.
5. List what `OnAttach` triggers on first spawn.

## Summary

TerrainComponent owns height samples and mesh regeneration, exposing local/world brushes, raycast, and reset while pairing with MaterialComponent for shading.

---

Next: [Terrain in Scene Pass](03-terrain-lighting.md).
