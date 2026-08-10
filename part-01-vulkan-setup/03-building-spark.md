# Building the Reference Project

> **Build-Along Milestone:** SparkDemo builds and runs on your machine; you know where shaders, assets, and the engine binary live.

**You will learn:**

- Install toolchain dependencies for Spark on macOS, Linux, or Windows.
- Configure CMake presets and understand output paths.
- Locate SPIR-V shaders and runtime asset directories the engine expects.

## 1. The Problem We Are Solving

You cannot build along with a book if the reference project does not compile. Before touching Vulkan theory, you need a **reproducible build** of Spark: compiler, Vulkan SDK, CMake, and network access for first-time dependency fetch.

This chapter is practical. Follow every step on your machine. When SparkDemo shows a window (even a blank or clearing color), Part I's Vulkan chapters have a running target to compare against.

## 2. Core Concepts

### Toolchain requirements

| Tool | Minimum | Used for |
|------|---------|----------|
| CMake | 3.28 | Presets, shader custom target |
| C++ compiler | C++23 | Spark engine code |
| Vulkan SDK | Latest stable | Loader, validation layers, `glslangValidator` |
| Git + network | — | GLFW FetchContent, fonts, sample assets on first configure |

Verify before configuring:

```bash
cmake --version
glslangValidator --version
vulkaninfo --summary   # or vkvia on some systems
```

If `glslangValidator` is missing, install the [Vulkan SDK](https://vulkan.lunarg.com/) and add its `bin` directory to `PATH`.

### CMake presets and build trees

Spark uses **CMake presets** so IDE and CLI builds share the same flags:

```bash
cmake --preset debug
cmake --build cmake-build-debug -j
./cmake-build-debug/SparkDemo
```

| Preset | Output dir | Main target |
|--------|------------|-------------|
| `debug` | `cmake-build-debug/` | `SparkDemo`, tests |
| `editor-debug` | `cmake-build-editor/` | `SparkEditor` only (slimmer) |

The debug preset defines macros such as `SPARK_BUILD_ASSETS_DIR` and `SPARK_SHADER_SPV_DIR` pointing into the build tree — your running binary loads shaders and fonts from there, not from source folders.

### Shader build pipeline

Spark does not ship precompiled SPIR-V in git for every platform. CMake builds a `SparkShaders` target that runs `glslangValidator` on every `.vert`, `.frag`, and included `.glsl` under `shaders/`. The engine executable depends on `SparkShaders`, so a shader edit triggers recompilation before link.

Runtime path: `SPARK_SHADER_SPV_DIR` → `VulkanSpvShaderLoader` → `VkShaderModule`. If you see a black screen in later parts, first check whether `.spv` files exist next to the binary's shader output directory.

### Assets and screenshots

Runtime assets (fonts, models, textures) copy into the build directory via CMake. Screenshots and recordings default under `SPARK_BUILD_ASSETS_DIR`:

- `screenshots/` — F12 in SparkDemo
- `recordings/` — F9 video capture

Knowing these paths helps when verifying Part I milestones ("did present actually run?").

### SparkDemo as launcher

`SparkDemo` is not a single game — it is a **launcher** for many `IGame` demos (3D maze, material showcase, 2D platformer, etc.). Part I only needs any mode that opens a window. Later parts reference specific demos by name; for now, run the default or pick "3D" modes from the menu.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### FetchContent for GLFW

**Concept:** Toolchain requirements

**Decision:** Download GLFW at configure time instead of vendoring a submodule

**Why:** Keeps the repo smaller and pins a known GLFW version across platforms. Requires network on first `cmake` run.

**In Spark:** Root `CMakeLists.txt` — `FetchContent_Declare(glfw ...)`.

### Shaders as a dedicated CMake target

**Concept:** Shader build pipeline

**Decision:** `add_custom_target(SparkShaders)` with explicit `.spv` outputs

**Why:** Ninja/Make need concrete output files to incrementalize. Listing `.glsl` includes as `DEPENDS` rebuilds SPIR-V when shared headers change.

**In Spark:** `CMakeLists.txt` — `SparkShaders` target; engine links via `add_dependencies`.

### Build-dir asset roots via compile definitions

**Concept:** Assets and screenshots

**Decision:** `SPARK_BUILD_ASSETS_DIR` and `SPARK_SHADER_SPV_DIR` are compile-time strings

**Why:** Running from the build tree works without setting environment variables. CI and teammates get identical paths relative to the executable.

**In Spark:** `include/spark/config.hpp` (generated or defined by CMake).

### Separate editor preset

**Concept:** CMake presets

**Decision:** `editor-debug` builds only editor targets

**Why:** Faster iteration when working on tools without compiling every demo and test.

**In Spark:** `CMakePresets.json` — `editor-debug` preset.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book; Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `CMakeLists.txt`
- `CMakePresets.json`
- `README.md`
- `shaders/` (sources)
- `cmake-build-debug/SparkDemo` (output binary)

### Step 1 — Clone and open the repo

```bash
git clone <your-spark-remote> spark
cd spark
```

Set `SPARK_ROOT` if you use scripts elsewhere; for this book, keeping a sibling folder next to `vulkan-notes` is fine:

```text
Projects/
  spark/
  vulkan-notes/
```

### Step 2 — Configure with the debug preset

```bash
cmake --preset debug
```

First run downloads GLFW and may fetch assets. If configure fails on validation layers, install the Vulkan SDK — Spark enables validation in Debug builds.

### Step 3 — Build SparkDemo

```bash
cmake --build cmake-build-debug -j$(nproc 2>/dev/null || sysctl -n hw.ncpu)
```

Confirm `SparkShaders` runs during build (you should see `glslangValidator` invocations on a clean build).

### Step 4 — Run and verify window

```bash
./cmake-build-debug/SparkDemo
```

You should get a GLFW window and a demo menu or scene. Press **F3** to toggle FPS overlay if available — that confirms the main loop from Chapter 1 is alive.

### Step 5 — Locate compiled SPIR-V

```bash
find cmake-build-debug -name 'scene.vert.spv' 2>/dev/null | head -1
```

Note the directory — that is what `SPARK_SHADER_SPV_DIR` points to. When Part II fails to load shaders, compare your path with this location.

### Step 6 — Capture a screenshot

Press **F12** in SparkDemo. Find the PNG under `cmake-build-debug/.../screenshots/` (path uses `SPARK_BUILD_ASSETS_DIR`). A saved screenshot proves the swapchain presented at least one frame.

## 5. How This Connects to the Rest of Your Engine

Every later chapter assumes you can rebuild after editing shaders or C++. Part **III** dives into `SparkShaders` CMake rules. Part **II** needs a running binary to show triangle draws. Keep a terminal tab with `cmake --build cmake-build-debug -j` ready throughout the book.

## 6. Pitfalls and Debugging

- **`glslangValidator` not found** — Vulkan SDK `bin` not on `PATH`; fix before debugging Vulkan code.
- **Running SparkDemo from wrong cwd** — assets are relative to build macros; always run from build output or use IDE run configs in `.run/`.
- **Stale SPIR-V** — if shader output looks wrong, touch the `.frag` file and rebuild; confirm `SparkShaders` ran.
- **Apple Silicon** — Spark sets portability instance flags; use SDK MoltenVK; see `SPARK_PLATFORM_APPLE` in device creation (Chapter 7).

## 7. Exercises

1. Build the `editor-debug` preset and list which targets exist vs full debug.
2. Change `shaders/scene.frag` albedo tint slightly, rebuild, run — confirm visible change.
3. Run `vulkaninfo` and note your GPU name and API version.
4. Read `docs/ARCHITECTURE_AND_DEVELOPER_GUIDE.md` table of contents — bookmark it for later parts.

## Summary

A working Spark build is the foundation of this book: CMake presets, `SparkShaders`, build-dir assets, and `SparkDemo` as the run target. You are not memorizing every CMake line — you are ensuring your machine can compile and run the reference engine before Vulkan details accumulate.

---

Next: [The Engine Loop](04-engine-loop.md).
