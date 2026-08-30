# C# Scripting

> **Build-Along Milestone:** You can document safe interop call order and locate component glue in SparkInteropComponents.cpp.

**You will learn:**

- Explain SparkInterop C ABI and opaque handle discipline.
- Map managed OnUpdate to native subsystem call order.
- Identify safe facades vs forbidden Vulkan exposure.

## 1. The Problem We Are Solving

Recompiling C++ for every gameplay tweak slows iteration. **Scripting** hosts managed code behind Spark's **C interop** — the same facades native demos use (`ProcessSoundCues`, physics simulate, component mutation) without exporting ECS memory layouts or GPU handles.

You may not ship C# in your fork yet — understanding the boundary prevents accidental `vk*` calls from gameplay and prepares you to enable `spark_script_host_run` when ready.

Milestone: document OnUpdate call order mirroring native demos and list ten allowed vs ten forbidden interop APIs.

## 1b. Concept Deep Dive — SparkInterop C ABI

### Stable C header, opaque handles

Managed gameplay cannot include C++ ECS headers that change layout every commit. Spark exposes a **C API** in `SparkInterop.h` — P/Invoke friendly, parsed by ClangSharp for codegen:

```cpp
// include/spark/scripting/SparkInterop.h
typedef struct SparkEngineContext SparkEngineContext;
typedef struct SparkGameWorld SparkGameWorld;
typedef struct SparkGameObject SparkGameObject;

typedef struct SparkFrameTiming {
    float deltaTimeSeconds;
    // ...
} SparkFrameTiming;
```

Handles are opaque pointers — scripts never see `GameObject` memory layout or vtables.

### Allowed: subsystem facades native demos use

```text
spark_world_create_game_object
spark_transform_set_translation
spark_physics_simulate_2d
spark_process_sound_cues
spark_game_get_world
```

These route through the same code paths as `Platformer2DDemo` — physics, audio, component mutation without GPU exposure.

### Forbidden: anything touching Vulkan or ECS internals

```text
vkCreateBuffer          — never exported
GameComponent* raw      — layout unstable
SceneRenderParams*      — per-frame renderer state
DescriptorSet handles   — GPU resource lifetime
Mesh vertex buffer ptr  — bypasses asset cache
```

`SparkInteropComponents.cpp` implements safe mutation wrappers; `SparkInteropScene.cpp` covers spawn/destroy. Regenerate C# bindings after header changes: `tools/generate-csharp-bindings.sh`.

## 2. Core Concepts

### Why a C ABI boundary

Managed gameplay (C# via CoreCLR) cannot safely include C++ ECS headers that change layout every commit. Spark exposes a **stable C API** in `SparkInterop.h` — P/Invoke friendly, parsed by ClangSharp for codegen.

```text
C# game assembly  →  P/Invoke  →  SparkInterop (C)
                                      ↓
                                 C++ EngineContext, GameWorld, SoundEngine
```

Version the **interop surface**, not internal `GameObject` memory layout. Regenerate bindings when `SparkInterop.h` changes (`tools/generate-csharp-bindings.sh`).

### Opaque handles and SparkFrameTiming

Interop exports opaque pointers:

```cpp
typedef struct SparkEngineContext SparkEngineContext;
typedef struct SparkGameObject SparkGameObject;
```

Managed code never dereferences them — only passes back to exported functions. `SparkFrameTiming` mirrors native `FrameTiming` (delta, total, frameIndex) for deterministic `OnUpdate`.

### Managed game callbacks

`SparkManagedGameCallbacks` registers script lifecycle:

```cpp
typedef void (*SparkGameOnUpdateFn)(void* userData, const SparkFrameTiming* timing, SparkEngineContext* context);
```

Host calls native `OnUpdate` equivalent after wiring `SparkHostApi::registerManagedGame`. Entry: `spark_script_host_run(argc, argv)` bootstraps CLR and loads game DLL.

### Subsystem parity through interop

Interop re-exports native subsystem order as functions:

| Native | Interop |
|--------|---------|
| `ProcessSoundCues` | `spark_world_process_sound_cues` |
| `Simulate2D` | `spark_world_physics_simulate_2d` |
| AI simulate | `spark_world_simulate_game_ai` |

Scripts must call the same sequence as C++ demos — physics before render snapshot, sound cues after listener update.

### SparkInteropComponents glue

`SparkInteropComponents.cpp` implements component attach/get/set for dozens of `SparkComponentKind` values. Each function validates handle, casts internally, mutates C++ component.

Adding interop for a new component means: enum value in `SparkInterop.h`, C API functions, registration in components cpp, regenerated C# stubs — **not** exposing the C++ class directly.

### Safety boundary

**Allowed:** transform set, sound cue queue, scene load hooks, input query, physics via approved wrappers.

**Forbidden:** `VkDevice`, `VkCommandBuffer`, raw pointers to GPU mapped memory, calling `PresentFrame` from script thread.

Document forbidden list in `docs/CSHARP_SCRIPTING.md`. Same separation as Part I — gameplay layer never includes `vulkan.h`.

### Managed Update parity table

| Order | Native (C++ demo) | Interop export |
|-------|-------------------|--------------|
| 1 | Input poll | spark_input_* |
| 2 | Physics Simulate2D | spark_world_physics_simulate_2d |
| 3 | AI SimulateGameAi | spark_world_simulate_game_ai |
| 4 | ProcessSoundCues | spark_world_process_sound_cues |
| 5 | Gameplay logic | managed OnUpdate body |
| 6 | SceneSubmit snapshot | automatic before PresentFrame |

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### SparkInterop.h as single contract

**Concept:** Why a C ABI boundary

**Decision:** SparkInterop.h as single contract

**Why:** C linkage name stability across compilers; ClangSharp one-to-one codegen.

**In Spark:** `include/spark/scripting/SparkInterop.h`.

### Forward-declared structs

**Concept:** Opaque handles and SparkFrameTiming

**Decision:** Forward-declared structs

**Why:** Prevents managed reimplementation of C++ layout; timing struct is blittable.

**In Spark:** SparkInterop.h lines 27–40.

### SparkManagedGameCallbacks + SparkHostApi

**Concept:** Managed game callbacks

**Decision:** SparkManagedGameCallbacks + SparkHostApi

**Why:** Mirrors IGame virtuals without exporting C++ vtables to CLR.

**In Spark:** SparkInterop.h lines 83–99.

### Explicit spark_world_* simulate functions

**Concept:** Subsystem parity through interop

**Decision:** Explicit spark_world_* simulate functions

**Why:** Forces script authors to respect native ordering instead of implicit magic.

**In Spark:** SparkInterop.h — physics and sound exports.

### Per-component C API in SparkInteropComponents.cpp

**Concept:** SparkInteropComponents glue

**Decision:** Per-component C API in SparkInteropComponents.cpp

**Why:** Centralizes validation; one place to audit pointer lifetimes on destroy.

**In Spark:** `src/spark/scripting/SparkInteropComponents.cpp`.

### No GPU handles in SparkInterop.h

**Concept:** Safety boundary

**Decision:** No GPU handles in SparkInterop.h

**Why:** Security and stability — scripts cannot submit arbitrary command buffers.

**In Spark:** `docs/CSHARP_SCRIPTING.md`.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `include/spark/scripting/SparkInterop.h`
- `src/spark/scripting/SparkInteropComponents.cpp`
- `docs/CSHARP_SCRIPTING.md`

### Step 1 — Read interop header surface

*Handles, timing, managed game callbacks*

```cpp
// include/spark/scripting/SparkInterop.h (lines 27–99)
typedef struct SparkEngineContext SparkEngineContext;
typedef struct SparkGameWorld SparkGameWorld;
typedef struct SparkGameObject SparkGameObject;

typedef struct SparkFrameTiming {
    float deltaTimeSeconds;
    float totalTimeSeconds;
    uint64_t frameIndex;
} SparkFrameTiming;

typedef void (*SparkGameOnUpdateFn)(void* userData, const SparkFrameTiming* timing, SparkEngineContext* context);

typedef struct SparkManagedGameCallbacks {
    void* userData;
    SparkGameOnAttachFn onAttach;
    SparkGameOnDetachFn onDetach;
    SparkGameOnUpdateFn onUpdate;
    SparkGameOnRenderFn onRender;
} SparkManagedGameCallbacks;
```

### Step 2 — Find world subsystem exports

*Scene/world lifecycle and subsystem simulate exports*

```cpp
// include/spark/scripting/SparkInterop.h (lines 135–150)
SPARK_SCRIPT_API void spark_world_process_sound_cues(SparkGameWorld* world, SparkEngineContext* context);
SPARK_SCRIPT_API void spark_world_simulate_game_ai(
        SparkGameWorld* world, const SparkFrameTiming* timing, SparkEngineContext* context);
SPARK_SCRIPT_API void spark_world_physics_simulate_2d(
        SparkGameWorld* world, const SparkFrameTiming* timing, const SparkPhysicsWorld2DSettings* settings);
SPARK_SCRIPT_API void spark_world_physics_simulate_3d(SparkGameWorld* world, const SparkFrameTiming* timing);
```

### Step 3 — Read CSHARP_SCRIPTING.md

Follow Spark doc for host setup, assembly load path, and `SparkScriptEntryInitialize` convention.

### Step 4 — Trace component get/set

*Example interop — get object name as UTF-8*

```cpp
// src/spark/scripting/SparkInteropComponents.cpp (lines 65–78)
int spark_object_get_name(const SparkGameObject* object, char* outUtf8, const std::uint32_t outCapacity) {
    if (object == nullptr || outUtf8 == nullptr || outCapacity == 0) {
        return 0;
    }
    const char* name = ToObject(const_cast<SparkGameObject*>(object))->GetName().CStr();
    const std::size_t len = std::strlen(name);
    const std::size_t copy = (len < outCapacity - 1) ? len : (outCapacity - 1);
    std::memcpy(outUtf8, name, copy);
    outUtf8[copy] = '\0';
    return static_cast<int>(copy);
}
```

### Step 5 — Sketch PlaySound interop path

Managed gameplay should call `spark_world_process_sound_cues` after queuing via `SoundCueComponent` interop — not a raw "play WAV" that bypasses spatial listener order.

### Step 6 — Document OnUpdate order

```text
input → gameplay → physics → AI → sound cues → (native OnRender builds snapshot)
```

Managed `OnUpdate` runs inside host hook **before** native render — never call Vulkan from here.

### Step 7 — Handle entity destroy

When `spark_world_destroy_game_object` runs, native side must invalidate handles held in managed dictionaries — prevent use-after-free if script caches `SparkGameObject*`.

### Step 8 — Milestone — allowed/forbidden API list

Table ten approved interop calls vs ten forbidden (any `vk*`, direct descriptor access, manual SceneRenderParams memcpy without codegen).

## 5. How This Connects to the Rest of Your Engine

Part **XIII** scene load interop enables designer pipelines. Part **XIV** physics stays native — scripts apply forces via `spark_world_physics_simulate_*` wrappers only. Part **XVI** retail builds may `#ifdef` out scripting and validation together.

Chapters 1–3 audio/AI subsystems are reachable through world/process functions — scripts orchestrate, native code executes heavy work.

## 6. Pitfalls and Debugging

- **Exposing VkDevice to scripts** — catastrophic stability and security risk.
- **Calling interop from render callback** — race with GPU thread; Update only.
- **Manual SceneRenderParams layout in C#** — struct mismatch corrupts GPU; use components.
- **Skipping binding regeneration** — silent ABI drift after header edit.
- **GC allocations in hot OnUpdate** — combat stutter; pool script objects.

## 7. Exercises

1. Complete ABI review checklist on SparkInterop.h; note any padding risks.
2. Diagram script host boot sequence with native vs managed ownership boxes.
3. Write minimal P/Invoke test moving Transform via opaque handle.
4. Compare managed Update order table to `Platformer2DDemo::OnUpdate` line-by-line.
5. Grep `SparkInterop.h` for `Vk` — confirm zero GPU symbols in interop surface.

## Summary

C# scripting uses SparkInterop C API with opaque handles and managed callbacks mirroring IGame lifecycle. Native engine owns rendering; scripts mutate ECS and call subsystem exports from Update.

---

Continue to [Part XVI — Production](../part-16-production/index.md).
