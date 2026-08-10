# Game Templates

> **Build-Along Milestone:** Retail-config build launches forked game without ImGui or validation layers.

**You will learn:**

- Fork a Spark demo into a single-entry shipping game template.
- Apply quality presets at runtime without recompile.
- Strip validation, ImGui, and scripting for retail builds.

## 1. The Problem We Are Solving

The book built a **capability stack** — production needs a **shipping-shaped** project: one game entry, quality presets, developer tools optional, smoke-tested resize/load/shutdown.

Spark's demo shell proves subsystems work; this chapter compresses that into **MyGame.exe** — same renderer, fewer menus, packaged assets, retail CMake flags.

Milestone: build `MyGame` target, launch from install directory (not source tree), play 60 s with validation off.

## 2. Core Concepts

### Demo vs shipping shape

SparkDemo shell launches **20 modes** — each validates a book part. Shipping game replaces selector with **one** `IGame` entry, fixed default scene, packaged assets.

```text
SparkShellDemo  →  menu picks BroadPhase2DDemo, ThreeDDemo, ...
MyGame.exe      →  MyGame : IGame only
```

Fork the demo closest to your genre — faster than empty `main.cpp` with missing Init order.

### Quality presets

Expose tunables without recompile:

```cpp
struct QualityPreset {
    uint32_t shadowMapSize;
    bool ssao;
    uint32_t maxParticles;
    float shadowDistanceMax;
};
```

Load from JSON or CLI args at startup — map to `SceneRenderParams` and renderer init flags. Low preset saves VRAM from Chapter 2 budget math.

| Setting | Init-only | Runtime toggle |
|---------|-----------|----------------|
| CSM resolution | Yes — recreate shadow maps | Requires shadow pass destroy |
| SSAO enable | Yes — creates SSAO images | Recreate post stack |
| Exposure / tonemap | No | Uniform write |

Changing shadow resolution mid-level equals swapchain-scale recreate cost — schedule at next level load or show a loading bar.

### Developer tools stripping

Retail builds disable:

- Validation layers (`VulkanGpuInstance::kEnableValidationLayers`)
- ImGui debug panels and F3 overlay
- F9 video capture / F12 screenshot hotkeys (optional)
- Scripting host (Part XV)

```cpp
#ifndef NDEBUG
    static constexpr bool kEnableValidationLayers = true;
#else
    static constexpr bool kEnableValidationLayers = false;
#endif
```

Use CMake compile definitions — `#ifdef SPARK_DEVELOPER_TOOLS` wraps overlays and verbose logging. Link-time removal beats runtime `if` for attack surface and binary size.

### Asset packaging

Ship `assets/` relative to binary or embed pak — scene files use paths from Part XIII (`assets/scenes/...`). Missing mesh path fails on user machine while working in dev tree — test from **install directory**, not source dir only.

```cmake
add_custom_command(TARGET MyGame POST_BUILD
  COMMAND ${CMAKE_COMMAND} -E copy_directory
    "${SPARK_ASSETS_DIR}/assets" "$<TARGET_FILE_DIR:MyGame>/assets")
```

### Smoke test checklist

Before milestone tag:

```text
Boot cold → play 60 s → resize window → load save/scene → toggle quality → shutdown
```

Validation **must** be clean on dev shutdown — memory chapter leak audit. Retail build runs on clean VM without Vulkan SDK installed.

```cpp
if (HasArg("--smoke-test")) {
    game.LoadDefaultLevel();
    for (int i = 0; i < 120; ++i) engine.Tick(1.f/60.f);
    return 0;
}
```

### Retail vs dev CMake diff

| Item | Dev (SparkDemo) | Retail (MyGame) |
|------|-----------------|-----------------|
| Validation layers | On (`!NDEBUG`) | Off |
| ImGui | Linked | Stripped |
| Demo selector | 20 modes | One game mode |
| Asset paths | Source tree | POST_BUILD copy |
| GPU timestamps | On | Off |

Run `nm` or `dumpbin` on retail binary — `ImGui::` symbols should be absent. File size drop 20–40% typical.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Fork Platformer or ThreeDDemo

**Concept:** Demo vs shipping shape

**Decision:** Copy closest genre demo, delete unused modes from CMake.

**Why:** Proven Init order and subsystem list — faster than greenfield `main`.

**In Spark:** Appendix demo catalog.

### Runtime struct applied at renderer init

**Concept:** Quality presets

**Decision:** `ApplyQualityPreset` before creating shadow resources.

**Why:** Artists iterate shadow size without programmer rebuild — preset drives `CreateImage` sizes.

**In Spark:** Renderer init or dedicated preset function.

### NDEBUG gates validation layers

**Concept:** Developer tools stripping

**Decision:** `kEnableValidationLayers` false in release.

**Why:** Validation doubles CPU cost and surfaces driver-specific errors to players.

**In Spark:** `VulkanGpuInstance.hpp` lines 19–23.

### Separate MyGame CMake target

**Concept:** Retail vs dev CMake diff

**Decision:** Second executable, not `#ifdef` in twenty demo files.

**Why:** Single target with all demos bloats retail binary and link time.

**In Spark:** `CMakeLists.txt` pattern for game targets.

### POST_BUILD asset copy

**Concept:** Asset packaging

**Decision:** Copy `assets/` next to executable on build.

**Why:** Mirrors `SoundFileLoader` and scene path resolution at runtime.

**In Spark:** CMake POST_BUILD on game target.

### Smoke test CLI flag

**Concept:** Smoke test checklist

**Decision:** `--smoke-test` runs 120 frames headless-friendly.

**Why:** Automates minimum regression without full GPU CI farm.

**In Spark:** Implement in your forked game entry.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `src/spark/demo/SparkShellDemo.cpp`
- `CMakeLists.txt`
- `include/spark/render/gpu/VulkanGpuInstance.hpp`

### Step 1 — Pick template demo

ThreeDDemo for 3D action, Platformer2DDemo for 2D — copy Game subclass files to `games/MyGame/`.

### Step 2 — Study shell selector you are not shipping

Retail `main` constructs `Engine(MakeUnique<MyGame>())` directly:

*Launcher menu — reference only for shipping fork*

```cpp
// src/spark/demo/SparkShellDemo.cpp (lines 38–59)
static constexpr const char* kLauncherDemoLabels[] = {
        "1  - Basic 3D scene",
        "2  - Sky: box, dome, plane",
        // ... 20 demo entries ...
        "20 - Dear ImGui tools (docking)",
};
```

### Step 3 — Define QualityPreset JSON

```json
{"shadowMapSize":2048,"ssao":true,"maxParticles":8192}
```

Parse in `OnAttach`, pass to renderer facade setter before shadow resources create.

### Step 4 — Strip developer tools in CMake

```cmake
add_executable(MyGame src/MyGame.cpp ...)
target_link_libraries(MyGame PRIVATE spark_engine ...)
target_compile_definitions(MyGame PRIVATE SPARK_DEVELOPER_TOOLS=0 NDEBUG)
```

Verify ImGui and validation compile out — binary size drops measurably.

### Step 5 — Confirm validation layer gate

*Release builds disable validation*

```cpp
// include/spark/render/gpu/VulkanGpuInstance.hpp (lines 19–23)
#ifndef NDEBUG
    static constexpr bool kEnableValidationLayers = true;
#else
    static constexpr bool kEnableValidationLayers = false;
#endif
```

### Step 6 — Package assets POST_BUILD

Copy `assets/` tree to `$<TARGET_FILE_DIR:MyGame>/assets`. Run game from install dir — load scene and footstep WAV.

### Step 7 — Run smoke checklist

60 s play, drag resize, async scene load if enabled, exit — validation silent on dev build.

### Step 8 — Tag milestone release

Git tag `v0.1-shipping-template` with known-good baseline JSON from profiling chapter — future renderer work compares against it.

## 5. How This Connects to the Rest of Your Engine

Parts **I–XV** feed the template. Appendix **demo catalog** picks fork source. Appendix **lifecycle** gates clean shutdown in smoke test. Profiling baselines attach to milestone tags.

Part **XVI** Chapters 1–4 hardening (resize, memory, async, profiling) are prerequisites for a credible shipping milestone gate.

## 6. Pitfalls and Debugging

- **Shipping with validation layers** — 2× CPU cost and user-visible errors on driver updates.
- **Hardcoded scene in renderer** — keep scene path in game layer JSON.
- **Dev-only asset paths** — test installed build on clean machine.
- **Leaving 19 demos linked** — bloated binary and confusing entry symbols.
- **No quality Low preset** — min-spec laptops OOM on default shadow size.
- **Assets only in source tree** — POST_BUILD copy missing on installed build.

## 7. Exercises

1. Diff dev vs retail compile definitions; list stripped subsystems.
2. Implement `--smoke-test` 120 frames; wire into CI or local `smoke.sh`.
3. Run retail binary on clean VM without SDK installed.
4. Document which quality settings require restart vs immediate apply.
5. Fork ThreeDDemo into `MyGame` — measure binary size before and after removing shell menu.

## Summary

Game templates fork one Spark demo with quality presets, packaged assets, and stripped developer tools. CMake flags gate validation, ImGui, capture hotkeys, and optional scripting. Retail shipping requires POST_BUILD asset packaging and smoke automation — dev SparkDemo stays the CI regression harness.

---

Next: [Appendices](../appendix/index.md).
