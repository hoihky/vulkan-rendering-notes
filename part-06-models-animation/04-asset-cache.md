# Asset Caching

> **Build-Along Milestone:** Ten identical props share one cached Mesh pointer; disk load happens once.

**You will learn:**

- Explain GameWorldAssetCache map types and path keys.
- Share Mesh and SkinnedGltfAsset across multiple entities safely.
- Use Register* APIs for async or editor-driven preload.

## 1. The Problem We Are Solving

Reloading the same rock mesh for every instance wastes I/O and duplicates CPU memory. Without a cache,
fifty props referencing one `.glb` parse the file fifty times and fragment the heap.

Build along: load one glTF twice through `GameWorld`, confirm pointer equality, spawn many instances with
distinct transforms.

### Asset cache for beginners

Without a cache, spawning 50 trees each calling `LoadGltf("tree.glb")` would:

- Read the file from disk 50 times
- Parse JSON/binary 50 times
- Allocate 50 copies of the same vertex data in RAM

**Cache = dictionary from path → loaded asset:**

```text
LoadMesh("props/rock.glb")
  if map contains path → return existing Mesh*
  else parse file, store in map, return pointer

Instance 1, 2, 3... all share same Mesh* — only Transform differs per entity
```

**Pointer equality test:** `LoadMesh(path) == LoadMesh(path)` should be **true** — same address means cache hit.

**Thread safety:** Spark's `GameWorldAssetCache` is used from main thread during load; async preload uses `Register*` APIs to warm cache before spawn.

**When to invalidate:** editor re-export of `.glb` changes file on disk — dev builds may need hot-reload or cache clear; shipping builds load once at level start.

## 1b. Concept Deep Dive — Cache Architecture in Spark

### Five parallel hash maps, one path key

`GameWorldAssetCache` owns separate maps per asset type — all keyed by `Utf8String` path:

```cpp
// include/spark/scene/assets/GameWorldAssetCache.hpp (lines 132–144)
    HashMap<Utf8String, SharedPtr<Mesh>, ...> meshCache;
    HashMap<Utf8String, GltfAsset, ...> gltfCache;
    HashMap<Utf8String, GltfSceneDocument, ...> gltfSceneCache;
    HashMap<Utf8String, SkinnedGltfAsset, ...> skinnedGltfCache;
    HashMap<Utf8String, SharedPtr<Texture2D>, ...> textureCache;
```

`LoadGltf("chair.glb")` and `LoadMesh("chair.glb")` hit **different** maps — the rigid glTF path stores a full `GltfAsset` with materials; `LoadMesh` on a `.glb` delegates to `LoadGltf(path).mesh`.

### SkinnedGltfAsset — shared skeleton + mesh + locomotion hints

```cpp
// include/spark/scene/assets/GameWorldAssetCache.hpp
struct SkinnedGltfAsset {
    SharedPtr<SkinnedMesh> mesh;
    SharedPtr<Skeleton> skeleton;
    Array<GltfMaterialDesc> materials;
    std::uint32_t walkClipIndex = 0;
    Quaternion bindUpAlignment{Quaternion::Identity};
    float bindFacingYawOffset = 0.0F;
};
```

`LoadSkinnedGltf` parses once, auto-detects walk clip by name (`"walk"` substring), and caches the whole struct. Ten NPCs share one `SkinnedGltfAsset` pointer; each entity gets its own `AnimatorComponent` with independent time.

### Register* APIs for async loading

`GameWorldAssetLoader` decodes on a worker thread, then calls `RegisterGltf` / `RegisterSkinnedGltf` on the main thread:

```text
RequestGltf(path)  →  worker: parse file  →  main: RegisterGltf(asset, path)
                                                      ↓
                                              gltfCache[path] = asset
```

Gameplay code polling `IsGltfReady(path)` can spawn entities only after registration — avoids half-loaded meshes on frame 1.

### Retain/release for streaming

`RetainAsset` / `ReleaseAsset` refcount cache entries by `CachedAssetKind`. When count hits zero, the entry can be evicted — editor streaming and level unload use this without leaking `SharedPtr` cycles.

## 2. Core Concepts

### Path-keyed caches

`GameWorldAssetCache` owns separate maps for meshes, glTF bundles, skinned glTF, and textures — keyed by `Utf8String` path. First load hits disk; subsequent loads return the same `SharedPtr`.


### Shared ownership semantics

`SharedPtr<const Mesh>`-style sharing: **immutable CPU data** after load. Transforms and materials vary per entity; mutating a cached mesh corrupts every instance.


### SkinnedGltfAsset bundle

Skinned import returns mesh + skeleton + walkClipIndex + bind alignment hints in one struct — spawn code does not re-parse the file for animation metadata.


### Register APIs for async loaders

`RegisterGltf` / `RegisterSkinnedGltf` let background jobs insert results — same keys as synchronous Load* paths so gameplay code stays uniform.


### TryGet* without loading

Lookups like `TryGetCachedSkinnedGltf` and `TryGetMeshByKeyOrPath` support editor previews and demos that preload assets without blocking the first spawn frame.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### GameWorldAssetCache on GameWorld

**Concept:** Path-keyed caches

**Decision:** GameWorldAssetCache on GameWorld

**Why:** Single Responsibility: asset I/O, not entity graph. Demos call `world.LoadGltf(path)`.

**In Spark:** `include/spark/scene/GameWorldAssetCache.hpp`

### SharedPtr return from Load*

**Concept:** Shared ownership semantics

**Decision:** SharedPtr return from Load*

**Why:** Reference counting frees CPU memory when last entity drops; no manual cache eviction in v1.

**In Spark:** `LoadMesh` / `LoadGltf` implementations

### Separate skinnedGltfCache map

**Concept:** SkinnedGltfAsset bundle

**Decision:** Separate skinnedGltfCache map

**Why:** Skeleton and clips stay tied to mesh fingerprint; AnimatorComponent references SharedPtr<Skeleton>.

**In Spark:** `SkinnedGltfAsset` struct — GameWorldAssetCache.hpp

### RegisterGltf / RegisterSkinnedGltf

**Concept:** Register APIs for async loaders

**Decision:** RegisterGltf / RegisterSkinnedGltf

**Why:** GameWorldAssetLoader can populate cache off-thread; main thread spawns from cached pointers.

**In Spark:** `src/spark/scene/GameWorldAssetLoader.cpp`

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `include/spark/scene/GameWorldAssetCache.hpp`
- `src/spark/scene/GameWorldAssetCache.cpp`
- `src/spark/scene/GameWorldAssetLoader.cpp`

### Step 1 — Read cache public API

*LoadMesh, LoadGltf, LoadSkinnedGltf, Register*, TryGet*.*

```cpp
// include/spark/scene/GameWorldAssetCache.hpp (lines 46–71)
/**
 * Path-keyed mesh / glTF / texture caches (Single Responsibility: asset I/O, not entity graph).
 * Owned by <c>GameWorld</c> and shared with async loaders via register APIs.
 */
class GameWorldAssetCache {
public:
    [[nodiscard]] SharedPtr<Mesh> LoadMesh(const char* path);
    [[nodiscard]] GltfAsset LoadGltf(const char* path);
    void RegisterGltf(const GltfAsset& asset, const char* cacheKey);
    [[nodiscard]] SkinnedGltfAsset LoadSkinnedGltf(const char* path);
    void RegisterSkinnedGltf(const SkinnedGltfAsset& asset, const char* cacheKey);
    [[nodiscard]] SharedPtr<Texture2D> LoadTexture(const char* path);
    SharedPtr<Mesh> RegisterMesh(const SharedPtr<Mesh>& mesh, const char* cacheKey = nullptr);
    SharedPtr<Texture2D> RegisterTexture(const SharedPtr<Texture2D>& texture, const char* cacheKey = nullptr);
    [[nodiscard]] SharedPtr<Mesh> TryGetMeshByKeyOrPath(const char* keyOrPath) const;
    [[nodiscard]] SharedPtr<Texture2D> TryGetTextureByKeyOrPath(const char* keyOrPath) const;
    [[nodiscard]] bool TryGetCachedSkinnedGltf(const char* path, SkinnedGltfAsset& out) const;
    [[nodiscard]] bool TryGetAxisAlignedBoundsForKeyOrPath(const char* keyOrPath, Vector3& outMin, Vector3& outMax)
            const;

private:
    HashMap<Utf8String, SharedPtr<Mesh>, Detail::Utf8StringHasher> meshCache;
    HashMap<Utf8String, GltfAsset, Detail::Utf8StringHasher> gltfCache;
    HashMap<Utf8String, SkinnedGltfAsset, Detail::Utf8StringHasher> skinnedGltfCache;
    HashMap<Utf8String, SharedPtr<Texture2D>, Detail::Utf8StringHasher> textureCache;
};
```

### Step 2 — Trace LoadMesh cache miss path

*Path lookup → TryLoadFromObj or delegate to LoadGltf for .glb/.gltf.*

```cpp
// src/spark/scene/GameWorldAssetCache.cpp (lines 47–63)
SharedPtr<Mesh> GameWorldAssetCache::LoadMesh(const char* path) {
    if (path == nullptr || path[0] == '\0') {
        return SharedPtr<Mesh>();
    }
    if (IsGltfPath(path)) {
        return LoadGltf(path).mesh;
    }
    const Utf8String key(path);
    if (const SharedPtr<Mesh>* cached = meshCache.Find(key)) {
        return *cached;
    }
    auto mesh = MakeShared<Mesh>(Utf8String(path));
    if (!Mesh::TryLoadFromObj(path, *mesh)) {
        return SharedPtr<Mesh>();
    }
    meshCache.Add(key, mesh);
    return mesh;
```

### Step 3 — Follow LoadSkinnedGltf

*TryLoadSkinnedCharacterFromGltf → SharedPtr bundle in skinnedGltfCache.*

```cpp
// src/spark/scene/GameWorldAssetCache.cpp (lines 97–123)
SkinnedGltfAsset GameWorldAssetCache::LoadSkinnedGltf(const char* path) {
    SkinnedGltfAsset empty{};
    if (path == nullptr || path[0] == '\0') {
        return empty;
    }
    const Utf8String key(path);
    if (const SkinnedGltfAsset* cached = skinnedGltfCache.Find(key)) {
        return *cached;
    }
    SkinnedMesh mesh;
    Skeleton skeleton;
    SharedPtr<Texture2D> tex;
    std::uint32_t walkClip = 0;
    Quaternion bindUp{};
    float facingYaw = 0.0F;
    if (!TryLoadSkinnedCharacterFromGltf(path, mesh, skeleton, &tex, &walkClip, &bindUp, &facingYaw)) {
        return empty;
    }
    SkinnedGltfAsset asset{};
    asset.mesh = SharedPtr<SkinnedMesh>(new SkinnedMesh(MoveTemp(mesh)));
    asset.skeleton = SharedPtr<Skeleton>(new Skeleton(MoveTemp(skeleton)));
    asset.baseColorTexture = tex;
    asset.walkClipIndex = walkClip;
    asset.bindUpAlignment = bindUp;
    asset.bindFacingYawOffset = facingYaw;
    skinnedGltfCache.Add(key, asset);
    return asset;
```

### Step 4 — Use Register for tooling

*RegisterSkinnedGltf — editor/async path inserts preloaded bundle.*

```cpp
// src/spark/scene/GameWorldAssetCache.cpp (lines 126–134)
void GameWorldAssetCache::RegisterSkinnedGltf(const SkinnedGltfAsset& asset, const char* cacheKey) {
    if (!asset.mesh || !asset.skeleton || cacheKey == nullptr || cacheKey[0] == '\0') {
        return;
    }
    skinnedGltfCache.Add(Utf8String(cacheKey), asset);
    if (asset.baseColorTexture) {
        textureCache.Add(Utf8String(cacheKey), asset.baseColorTexture);
    }
}
```

### Step 5 — Share one mesh across instances

```cpp
auto tree = world.LoadGltf("assets/models/Tree_1.glb");
for (int i = 0; i < 5; ++i) {
    auto* t = world.CreateGameObject();
    t->AddComponent<MeshComponent>(tree.mesh, SceneMeshSlot::Custom, Vector3::One);
}
```

Log `tree.mesh.Get()` address — identical across spawns.

### Step 6 — Preload in demo init

CharacterCameraDemo caches avatar and tree assets at level load so `ApplyAvatarModel` never blocks on disk I/O mid-gameplay.

### Step 7 — Milestone check — duplicate spawn, one load

Add logging on cache miss only. Spawning ten identical glTF props should log one import and reuse SharedPtr addresses.

## 5. How This Connects to the Rest of Your Engine

**Custom Mesh Pool** deduplicates GPU geometry by mesh pointer fingerprint, not path — cache sharing on CPU
automatically reduces unique custom meshes in the scene snapshot. Part VII texture cache mirrors the same pattern.

## 6. Pitfalls and Debugging

- **Mutating cached mesh** — one edit corrupts all instances; treat loaded data as immutable.
- **Different import options, same path** — Spark keys by path only; fork path or extend key if you add flip flags.
- **Reload without invalidating GPU pool** — CPU updated but stale VkBuffer; bump fingerprint or restart renderer.
- **Empty SharedPtr from failed load** — always check before AddComponent.

## 7. Exercises

1. Load one glTF twice; assert SharedPtr address equality.
2. Spawn 10 trees; log meshCache size stays 1.
3. Sketch a hot-reload key that includes file modification time.

## Summary

GameWorldAssetCache deduplicates CPU meshes, glTF bundles, and skinned assets by path with SharedPtr sharing.

---

Next: [Custom Mesh Pool](05-custom-mesh-pool.md).
