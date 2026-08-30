# Scene File Format

> **Build-Along Milestone:** Save file with three entities shows correct magic, ids, parent links, and component lines.

**You will learn:**

- Design a versioned scene format separating gameplay data from GPU resources.
- Use SceneDocument as interchange between disk and GameWorld.
- Explain Capture/Write and Read/Apply symmetry with line-oriented v4 text.
- Plan migration when format version increments.

## 1. The Problem We Are Solving

Until Part XII you authored worlds by recompiling C++ or spawning entities in demo `Load()` functions. Production pipelines persist **ECS descriptions** to disk and rebuild GPU state on load. Spark's `SceneSerializer` captures `GameWorld` into `SceneDocument` and writes `spark_scene_v4` text files — you will follow that contract end-to-end.

The core rule: GPU objects are never portable. Save stable ids, component fields, and project-relative asset paths. Let `GameWorldAssetLoader` and `SceneSubmit` reconstruct visuals the same way runtime spawning does. Treat `SceneRenderParams` as a per-frame derivative, not a save target.

Milestone: create Ground + Sun + one cube in the editor, save, and verify `scene.txt` lists three `E` lines with transform and mesh/light `C` lines.

## 1b. Concept Deep Dive — SceneDocument vs GPU State

### Versioned interchange, not a GPU dump

`SceneDocument` is the in-memory contract between disk and `GameWorld`. The magic string and entity records carry **only** gameplay data:

```cpp
// include/spark/scene/serialization/SceneDocument.hpp
struct SceneDocument {
    static constexpr const char* kMagic = "spark_scene_v4";
    SceneDocumentHeader header;
    Array<EntityRecord> entities;
};

struct EntityRecord {
    std::uint64_t id = 0;
    Utf8String name;
    std::int64_t parentId = -1;
    Array<ComponentRecord> components;  // kind tag + opaque payload
};
```

Each `ComponentRecord` is one `C` line on disk — `transform`, `mesh`, `gltf_scene`, etc. No `VkBuffer`, no descriptor set handles, no per-frame render params.

### Capture/Apply symmetry through SceneSerializer

`SceneSerializer::Capture` walks live entities and asks the registry for payloads; `SceneDeserializer::Apply` creates `GameObject`s, restores components, then links parents in a third pass:

```cpp
// include/spark/scene/serialization/SceneSerializer.hpp
class SceneSerializer {
    [[nodiscard]] SceneDocument Capture(
            const GameWorld& world,
            const SceneCaptureContext& ctx,
            const std::function<bool(const GameObject*)>& includeEntity) const;
    [[nodiscard]] bool WriteToFile(const SceneDocument& document, const char* path) const;
};

class SceneDeserializer {
    [[nodiscard]] bool Apply(
            const SceneDocument& document,
            GameWorld& world,
            const SceneApplyContext& ctx,
            HashMap<std::uint64_t, GameObject*>* outIdToObject = nullptr) const;
};
```

**Why three-pass Apply?** Parent links need stable entity ids. Pass 1 creates objects; pass 2 restores components (some need asset loader); pass 3 wires `SetParent` from `parentId`.

### assetsRoot ties relative paths to runtime roots

The header stores a project-relative assets root. On load, `SceneApplyContext::assetsRoot` resolves `"models/chair.glb"` against the same directory the editor used at save time — so capture and apply share one path vocabulary regardless of where the executable runs from.

## 2. Core Concepts

### What to serialize

Scene persistence stores **gameplay state** — entity hierarchy, transforms, mesh asset paths, light parameters, collider dimensions. It never stores **GPU resources**: `VkBuffer`, pipeline handles, descriptor sets, or per-frame `SceneRenderParams`.

| Save | Do not save |
|------|-------------|
| Entity id, name, parent id | VkImage, VkPipeline |
| Component fields (POD + paths) | GPU buffer offsets |
| Relative asset paths | Frame index, camera viewProj |

When a scene loads, `MeshComponent` paths trigger `GameWorldAssetLoader`; the renderer uploads geometry through the same custom-mesh and static-slot paths as runtime spawning.

### Version header and magic line

Spark scene files begin with a **magic string** identifying format generation. Current writers emit `spark_scene_v4`; readers also accept `spark_scene_v3` for backward compatibility.

```text
spark_scene_v4
3
H name "Level01"
E 42 -1 "Ground"
C transform 1 1 1 0 0 0 0 0 0 1
```

Unknown magic → reject immediately. Known older magic → parse with v3 rules or run migration transforms. Never silently reinterpret bytes — corrupt worlds are worse than a clear load error.

### SceneDocument as interchange

`SceneDocument` is the in-memory envelope between disk bytes and live `GameWorld`. It holds `SceneDocumentHeader` metadata plus an `Array<EntityRecord>`. Each entity record stores stable `uint64_t id`, quoted name, `parentId` (-1 for roots), and an array of `ComponentRecord` lines.

Capture walks ECS → builds document. Apply walks document → spawns ECS. WriteToFile and ReadFromFile translate document ↔ text. This symmetry lets you unit-test serialization without touching disk and lets async loaders parse on workers while keeping Apply on the main thread.

### Line-oriented entity format

Spark chose a **human-readable line protocol** over JSON or binary blobs for classroom debugging.

```text
E <id> <parentId> "<name>"
C <kind_tag> <space-separated payload>
```

Payloads are opaque to the core loader — only the matching `IComponentSnapshotHandler` parses them. Quoted names escape backslash and double-quote. Component kind tags are stable snake_case strings (`transform`, `mesh`, `point_light`) registered alongside `ComponentKind` enum values.

### Deterministic capture order

`SceneSerializer::Capture` iterates a fixed `kCaptureOrder[]` array of `ComponentKind` values for each entity. Order matters when Apply adds components that depend on others — Transform before Mesh, Mesh before Material.

An **include predicate** filters which entities enter the document. The editor saves only placed props and user lights, not internal bootstrap entities like `EditorGui`. Runtime games can save the whole world or a streaming chunk by swapping the predicate.

### Authoritative ECS, derived render snapshot

`SceneRenderParams` is rebuilt every frame from components — it is not saved. The authoritative chain is:

```text
edit ECS → Capture → disk → Apply → ECS → SceneSubmit → GPU
```

This mirrors production engines (Unity YAML for components, not draw calls). Any temptation to "snapshot the renderer" is an architecture smell.

### Capture payload size math

Estimate scene file growth before shipping editor saves to artists:

| Component kind | Approx bytes per entity | Notes |
|----------------|-------------------------|-------|
| transform | ~90 | ten floats at six decimals |
| mesh | ~40–120 | slot tag + quoted path |
| point_light | ~60 | position + color + intensity |
| box_collider_3d | ~50 | half extents + offset |

For N entities with average K components, expect roughly `N × (80 + K×70)` bytes plus header. Terrain height arrays (Part XII) dominate size and belong in chunked sidecar files, not inline `C` lines.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### SceneSerializer + SceneDeserializer pair

**Concept:** What to serialize

**Decision:** SceneSerializer + SceneDeserializer pair

**Why:** Symmetric Capture/Apply API keeps read and write paths aligned; GPU rebuilds through existing importers.

**In Spark:** `include/spark/scene/serialization/SceneSerializer.hpp`

### SceneDocument structs

**Concept:** SceneDocument as interchange

**Decision:** SceneDocument structs (header + EntityRecord + ComponentRecord)

**Why:** Pure data layer decouples parsing, migration, and async staging from GameObject mutation.

**In Spark:** `include/spark/scene/serialization/SceneDocument.hpp`

### kMagic / kMagicV3 constants

**Concept:** Version header and magic line

**Decision:** kMagic / kMagicV3 constants on SceneDocument

**Why:** ReadFromString branches on magic before parsing body; writers always emit v4.

**In Spark:** `SceneDocument::kMagic` in SceneDocument.hpp

### E/C line prefixes with sscanf parsing

**Concept:** Line-oriented entity format

**Decision:** E/C line prefixes with sscanf parsing

**Why:** Diff-friendly text files; students inspect saves without hex dumps.

**In Spark:** `ParseEntityHeader` / `ParseComponentLine` in SceneSerializer.cpp

### kCaptureOrder[] in SceneSerializer.cpp

**Concept:** Deterministic capture order

**Decision:** kCaptureOrder[] in SceneSerializer.cpp

**Why:** Stable component ordering across platforms; handlers stay independent.

**In Spark:** Lines 18–55 of SceneSerializer.cpp

### Capture walks GameWorld, not SceneRenderParams

**Concept:** Authoritative ECS, derived render snapshot

**Decision:** Capture walks GameWorld, not SceneRenderParams

**Why:** Inspector and gameplay always edit components; submit derives visuals.

**In Spark:** `SceneSerializer::Capture` entity walk

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `include/spark/scene/serialization/SceneSerializer.hpp`
- `src/spark/scene/serialization/SceneSerializer.cpp`
- `include/spark/scene/serialization/SceneDocument.hpp`

### Step 1 — Read the serializer public API

Study the two-class split: `SceneSerializer` for Capture/Write, `SceneDeserializer` for Read/Apply.

*SceneSerializer and SceneDeserializer declarations*

```cpp
// include/spark/scene/serialization/SceneSerializer.hpp (lines 14–60)
class SceneSerializer {
public:
    explicit SceneSerializer(const ComponentSnapshotRegistry& registry = ComponentSnapshotRegistry::Default());

    [[nodiscard]] SceneDocument Capture(
            const GameWorld& world,
            const SceneCaptureContext& ctx,
            const std::function<bool(const GameObject*)>& includeEntity) const;

    [[nodiscard]] bool WriteToFile(const SceneDocument& document, const char* path) const;
};

class SceneDeserializer {
public:
    [[nodiscard]] bool ReadFromFile(const char* path, SceneDocument& out) const;
    [[nodiscard]] bool ReadFromString(const char* text, SceneDocument& out) const;

    [[nodiscard]] bool Apply(
            const SceneDocument& document,
            GameWorld& world,
            const SceneApplyContext& ctx,
            HashMap<std::uint64_t, GameObject*>* outIdToObject = nullptr) const;
};
```

### Step 2 — Inspect SceneDocument layout

Confirm magic constants, header fields, and entity/component record shapes.

*ComponentRecord, EntityRecord, SceneDocument*

```cpp
// include/spark/scene/serialization/SceneDocument.hpp (lines 10–45)
struct ComponentRecord {
    Utf8String kind;
    Utf8String payload;
};

struct EntityRecord {
    std::uint64_t id = 0;
    Utf8String name;
    std::int64_t parentId = -1;
    Array<ComponentRecord> components;
};

struct SceneDocument {
    static constexpr const char* kMagic = "spark_scene_v4";
    static constexpr const char* kMagicV3 = "spark_scene_v3";

    SceneDocumentHeader header;
    Array<EntityRecord> entities;
};
```

### Step 3 — Trace Capture entity walk

Capture skips entities with zero components. Follow `kCaptureOrder` emission per object.

*Capture loop over GameWorld*

```cpp
// src/spark/scene/serialization/SceneSerializer.cpp (lines 356–388)
SceneDocument SceneSerializer::Capture(
        const GameWorld& world,
        const SceneCaptureContext& ctx,
        const std::function<bool(const GameObject*)>& includeEntity) const {
    SceneDocument doc;
    world.ForEachGameObject([&](const GameObject* object) {
        if (object == nullptr || (includeEntity && !includeEntity(object))) {
            return;
        }
        EntityRecord entity;
        entity.id = object->GetId();
        entity.name = object->GetName();
        entity.parentId = object->GetParent() != nullptr
                ? static_cast<std::int64_t>(object->GetParent()->GetId()) : -1;

        for (const ComponentKind kind : kCaptureOrder) {
            const IComponentSnapshotHandler* handler = registry.Find(kind);
            if (handler == nullptr || object->TryGetComponentByKind(kind) == nullptr) {
                continue;
            }
            ComponentRecord record;
            if (handler->TryCapture(*object, ctx, record)) {
                entity.components.PushBack(MoveTemp(record));
            }
        }
        if (!entity.components.IsEmpty()) {
            doc.entities.PushBack(MoveTemp(entity));
        }
    });
    return doc;
}
```

### Step 4 — Study WriteToFile emission

Magic line, entity count, optional H headers, then E/C blocks.

*WriteToFile text emission*

```cpp
// src/spark/scene/serialization/SceneSerializer.cpp (lines 308–334)
bool SceneSerializer::WriteToFile(const SceneDocument& document, const char* path) const {
    std::FILE* f = std::fopen(path, "w");
    if (f == nullptr) {
        return false;
    }
    std::fprintf(f, "%s\n", SceneDocument::kMagic);
    std::fprintf(f, "%zu\n", document.entities.GetSize());
    WriteHeaderLinesToFile(f, document);
    for (std::size_t ei = 0; ei < document.entities.GetSize(); ++ei) {
        const EntityRecord& e = document.entities[ei];
        std::fprintf(f, "E %llu %lld ", static_cast<unsigned long long>(e.id), e.parentId);
        WriteEscapedName(f, e.name);
        std::fputc('\n', f);
        for (std::size_t ci = 0; ci < e.components.GetSize(); ++ci) {
            std::fprintf(f, "C %s %s\n", e.components[ci].kind.CStr(), e.components[ci].payload.CStr());
        }
    }
    std::fclose(f);
    return true;
}
```

### Step 5 — Parse and reject bad magic

ReadFromString rejects unknown magic before parsing entities.

*Magic validation on read*

```cpp
// src/spark/scene/serialization/SceneSerializer.cpp (lines 336–365)
bool SceneDeserializer::ReadFromString(const char* text, SceneDocument& out) const {
    out.entities.Clear();
    const bool isV4 = std::strcmp(magic, SceneDocument::kMagic) == 0;
    const bool isV3 = std::strcmp(magic, SceneDocument::kMagicV3) == 0;
    if (!isV4 && !isV3) {
        return false;
    }
    // ... entity count and E/C parse ...
}
```

### Step 6 — Round-trip empty bootstrap scene

Save before placing props — expect minimal entity count when `includeEntity` filters bootstrap.

### Step 7 — Audit for GPU leakage

Grep handlers for `Vk` handles or pointer serialization — GPU state must never appear in text payloads.

### Step 8 — Milestone — three-entity save file

Create Ground + Sun + one cube, save, and verify file lists three `E` lines with correct `C` lines.

## 5. How This Connects to the Rest of Your Engine

Chapter 2 registers per-component snapshot handlers. Chapter 3 async load uses the same document envelope. Part **VI** mesh import reloads paths restored by `mesh` handlers. Part **IX–X** lighting and post volumes serialize through extended handlers. Part **XIV** physics components appear in `kCaptureOrder` after terrain.

**Frame graph reminder:** serialization is CPU-only — no Vulkan calls. Accidentally serializing GPU handles into mesh payloads will compile yet fail on other machines. Grep handlers before review.

When extending `kCaptureOrder` for new component kinds, append physics and audio entries after visual components so Apply restores transforms and meshes before colliders that depend on world pose.

## 6. Pitfalls and Debugging

- **Saving SceneRenderParams** — stale derived data; always Capture from ECS.
- **Silent version mismatch** — corrupt loads; check magic and fail fast.
- **Absolute asset paths** — break on other machines; prefer assets_root + relative path.
- **Missing include filter** — editor saves internal UI entities; use predicate like SceneEditor3DDemo.
- **Wrong capture order** — Apply may fail if Material restores before Mesh exists.

**Debugging workflow:** when round-trip fails, work through: (1) magic + entity count match `E` line count, (2) parent ids reference existing `E` ids, (3) each `C` kind tag has registered handler, (4) Apply returns true, (5) RenderDoc mesh draw count ≥ entities with mesh after Pump.

Serialization has no VUID — failures are Apply returning false or silent handler skip. Log `TryRestore` false with kind tag and entity name.

## 7. Exercises

1. Save/load scene with 3 entities; verify parent hierarchy in file and world.
2. Change magic to `spark_scene_v99`; confirm ReadFromFile returns false.
3. List five component types from kCaptureOrder you would serialize first for your game.
4. Diff a v3 file against v4 — note header line additions.
5. Hand-estimate scene.txt size for 200 entities with 3 components using the payload table.

## Summary

Scene files store versioned ECS documents in line-oriented v4 text, not GPU handles. Spark uses SceneSerializer Capture/Apply with SceneDocument and kCaptureOrder for deterministic component emission.

---

Next: [Component Snapshots](02-snapshots.md).
