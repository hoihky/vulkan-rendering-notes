# Shader Build Pipeline

> **Build-Along Milestone:** You can trace a GLSL edit from `shaders/scene.vert` through CMake `SparkShaders` to `vkCreateShaderModule` at runtime.

**You will learn:**

- Explain why Vulkan uses offline SPIR-V instead of runtime GLSL compilation.
- Configure CMake so shader edits rebuild automatically.
- Diagnose missing `.spv` files and stale include dependencies.

## 1. The Problem We Are Solving

Part II loaded SPIR-V bytes and created pipelines. Part III is where you **author** those programs. If the compile loop is painful — wrong paths, forgotten rebuilds, silent stale headers — you will avoid shader work and the renderer stalls visually.

Spark's workflow is the industry-standard pattern:

```text
Edit GLSL → glslangValidator (-V) → .spv in build tree → VkShaderModule → VkPipeline
```

This chapter makes every step explicit. Open `CMakeLists.txt` near line 294 and `shaders/scene.vert` side by side. Your goal is a reliable **edit-compile-run** cycle measured in seconds, not minutes.

### What is a shader? (complete beginner)

A **shader** is a small program that runs on the **GPU**, not the CPU. Spark uses two kinds for 3D meshes:

| Shader | Runs how many times | Job |
|--------|---------------------|-----|
| **Vertex** | Once per vertex | Move corner to screen; pass UV/normal to next stage |
| **Fragment** | Once per pixel (approx.) | Pick final color; lighting, textures, shadows |

You write shaders in **GLSL** (OpenGL Shading Language) with Vulkan-specific rules. The CPU never executes this code directly — it is compiled to **SPIR-V** bytecode, loaded into `VkShaderModule`, and bound to a `VkPipeline`.

```text
You edit scene.frag (GLSL text)
  → CMake runs glslangValidator
  → scene.frag.spv (binary SPIR-V)
  → vkCreateShaderModule at startup / pipeline create
  → GPU executes during vkCmdDrawIndexed
```

### Why not compile GLSL at runtime?

OpenGL compiled shaders when you called `glCompileShader`. Vulkan **requires** offline SPIR-V because:

1. **Faster startup** — no driver compiler in the hot path.
2. **Deterministic errors** — compile fails at build time, not when player launches.
3. **Cross-vendor consistency** — SPIR-V is the contract; drivers optimize from there.

If `scene.vert.spv` is missing, Spark throws at pipeline creation — check your CMake build ran `SparkShaders` target.

### Your first shader edit workflow

1. Open `shaders/triangle.frag` (or `scene.frag`).
2. Change `outColor = vec4(1, 0, 0, 1)` to green.
3. Rebuild project (`cmake --build` — triggers SPIR-V regen).
4. Run SparkDemo — confirm color change.
5. If unchanged, you are loading stale `.spv` from wrong build directory.

## 1b. Concept Deep Dive — SparkShaders CMake Graph and Entry Points

### The compile loop in production

Spark's root `CMakeLists.txt` locates `glslangValidator`, emits `.spv` files into `${CMAKE_BINARY_DIR}/shaders/`, and wires the aggregate target `SparkShaders` so `SparkEngine` always builds shaders first:

```cmake
# CMakeLists.txt (lines 404–443)
set(SHADER_DIR ${CMAKE_CURRENT_SOURCE_DIR}/shaders)
set(SPV_DIR ${CMAKE_CURRENT_BINARY_DIR}/shaders)

set(SHADER_SOURCES
        ${SHADER_DIR}/scene.vert
        ${SHADER_DIR}/scene.frag
        ${SHADER_DIR}/shadow_depth.vert
        ${SHADER_DIR}/particle.vert
        ${SHADER_DIR}/particle.frag
        # sprite, text, UI, tonemap, post_process ...
)
foreach (SHADER ${SHADER_SOURCES})
    add_custom_command(
            OUTPUT ${SPV}
            COMMAND ${GLSLANG_VALIDATOR} -V --target-env vulkan1.0 -I${SHADER_DIR} ${SHADER} -o ${SPV}
            DEPENDS ${SHADER} ${SHADER_DIR}/scene_ubo.glsl ${SHADER_DIR}/scene_push.glsl ...
    )
endforeach ()
add_custom_target(SparkShaders DEPENDS ${SPV_FILES})
```

The `-I${SHADER_DIR}` flag is what makes `#include "scene_ubo.glsl"` resolve — without it, even a valid `scene.vert` fails offline compile.

### scene.vert and scene.frag — the primary SPIR-V pair

Both entry shaders enable the Google include extension and pull shared layouts before stage-specific code:

```glsl
#version 450
#extension GL_GOOGLE_include_directive : require

#include "scene_ubo.glsl"
#include "scene_push.glsl"
```

`scene.vert` declares eight vertex locations (position through vertex color), the skin SSBO at binding 2, and eleven varyings. `scene.frag` mirrors those inputs, adds `sampler2DArray` at binding 1, and includes lighting headers (`clustered_lights.glsl`, `ibl.glsl`, …). `VulkanScenePipeline` loads the resulting `scene.vert.spv` and `scene.frag.spv` at runtime — every other file in `SHADER_SOURCES` follows the same offline compile path for its pass.

### Shared headers must appear in DEPENDS

Editing `scene_ubo.glsl` or `scene_push.glsl` without listing them in the custom command `DEPENDS` line leaves stale SPIR-V on disk. Ninja only rebuilds outputs whose listed dependencies changed. When a visual change does not appear after a GLSL edit, check whether the modified file is in the DEPENDS list before suspecting runtime logic.

## 2. Core Concepts

### GLSL 450 and Vulkan semantics

Spark shaders start with:

```glsl
#version 450
#extension GL_GOOGLE_include_directive : require
```

Vulkan GLSL differs from OpenGL GLSL:

| OpenGL habit | Vulkan replacement |
|--------------|-------------------|
| `gl_FragColor` | `layout(location = 0) out vec4 outColor` |
| Implicit uniforms | `layout(set = N, binding = M)` |
| `texture2D()` legacy | `sampler2DArray` + explicit sets |
| Runtime compile | Offline SPIR-V only |

`GL_GOOGLE_include_directive` enables `#include "scene_ubo.glsl"` — shared UBO declarations without copy-paste.

### SPIR-V as the GPU ISA front-end

SPIR-V is a **binary intermediate** — not machine code, not human-editable. The driver JITs SPIR-V to GPU instructions. Benefits:

- **Fast startup** — no driver shader compiler at launch
- **Deterministic CI** — compile failures happen at build time
- **Cross-vendor** — same SPIR-V blob on NVIDIA, AMD, Apple (via MoltenVK)

Tradeoff: you must re-run `glslangValidator` after every GLSL change.

### glslangValidator flags Spark uses

```bash
glslangValidator -V --target-env vulkan1.0 -Ishaders scene.vert -o scene.vert.spv
```

| Flag | Meaning |
|------|---------|
| `-V` | Emit SPIR-V (not SPIR-V assembly text) |
| `--target-env vulkan1.0` | Vulkan memory model and builtins |
| `-Ishaders` | Include search path for `#include` |
| `-o` | Output `.spv` path |

Wrong `--target-env` can change `gl_Position` conventions or disable extensions.

### CMake custom commands and SparkShaders target

Each shader source gets one `add_custom_command` producing a `.spv` in `${CMAKE_BINARY_DIR}/shaders/`. The aggregate target `SparkShaders` depends on all outputs. `SparkEngine` depends on `SparkShaders` — linking the engine **forces** shaders to build first.

Critical: `DEPENDS` must list **included** `.glsl` files. Editing `scene_ubo.glsl` without listing it in DEPENDS means Ninja does not recompile `scene.vert.spv` — one of the most frustrating stale-build bugs.

### Runtime path SPARK_SHADER_SPV_DIR

At compile time Spark defines `SPARK_SHADER_SPV_DIR` to the build output shaders folder. `VulkanSpvShaderLoader::ReadSpvFile("scene.vert.spv")` opens:

```text
<build-dir>/shaders/scene.vert.spv
```

Running the executable from the wrong working directory does not matter — the path is baked in at CMake configure time (for the preset you used).

### Module vs pipeline lifetime

`VkShaderModule` can be destroyed after `vkCreateGraphicsPipelines` **if** you keep the pipeline alive — the driver has copied the relevant bits. Spark keeps modules until `DestroyGraphicsPipeline` for simpler reload debugging. Both patterns are valid; know which your engine uses.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Offline compile in CMake, never at runtime

**Concept:** SPIR-V as ISA front-end

**Decision:** No runtime GLSL compiler in shipping or dev builds

**Why:** Predictable errors, faster iteration once CMake is correct, smaller executable.

**In Spark:** Root `CMakeLists.txt` — `find_program(GLSLANG_VALIDATOR)` fails configure if missing.

### Explicit include DEPENDS on shared headers

**Concept:** CMake custom commands

**Decision:** `DEPENDS` lists `scene_ubo.glsl`, `clustered_lights.glsl`, `ibl.glsl`, etc.

**Why:** Header-only edits must invalidate all including stages.

**In Spark:** `foreach (SHADER ...)` custom command DEPENDS line 334.

### VulkanSpvShaderLoader abstraction

**Concept:** Runtime SPARK_SHADER_SPV_DIR

**Decision:** Single class reads bytes + creates modules for all passes

**Why:** Path logic and error messages live once; scene, sprite, and UI passes share it.

**In Spark:** `src/spark/render/gpu/VulkanSpvShaderLoader.cpp`.

### Separate .vert and .frag SPIR-V files

**Concept:** Module vs pipeline

**Decision:** Two modules per graphics pipeline, not one combined blob

**Why:** Vulkan pipeline creation expects per-stage modules; reuse vertex across fragment variants is rare.

**In Spark:** `VulkanScenePipeline` loads `scene.vert.spv` and `scene.frag.spv`.

### target-env vulkan1.0 pinned

**Concept:** glslangValidator flags

**Decision:** Match `VkApplicationInfo::apiVersion` policy (Spark targets 1.3 instance but shaders use vulkan1.0 env)

**Why:** glslang feature set stays conservative; avoids surprises on older validation.

**In Spark:** CMake `-V --target-env vulkan1.0`.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book; Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `CMakeLists.txt` (lines 294–341)
- `src/spark/render/gpu/VulkanSpvShaderLoader.cpp`
- `shaders/scene.vert`, `shaders/scene.frag`
- `shaders/scene_ubo.glsl`
- `src/spark/render/scene/VulkanScenePipeline.cpp`

### Step 1 — Verify toolchain

```bash
which glslangValidator
glslangValidator --version
```

If missing, install [Vulkan SDK](https://vulkan.lunarg.com/) and ensure `VULKAN_SDK/bin` is on `PATH`.

### Step 2 — Read CMake shader loop

```cmake
# CMakeLists.txt (lines 306–339, structure)
set(SHADER_SOURCES
        ${SHADER_DIR}/scene.vert
        ${SHADER_DIR}/scene.frag
        # shadow, particle, sprite, UI, tonemap ...
)
foreach (SHADER ${SHADER_SOURCES})
    # scene.vert → shaders/scene.vert.spv
    add_custom_command(
            OUTPUT ${SPV}
            COMMAND ${GLSLANG_VALIDATOR} -V --target-env vulkan1.0 -I${SHADER_DIR} ${SHADER} -o ${SPV}
            DEPENDS ${SHADER} ${SHADER_DIR}/scene_ubo.glsl ...
    )
endforeach ()
add_custom_target(SparkShaders DEPENDS ${SPV_FILES})
add_dependencies(SparkEngine SparkShaders)
```

**Build-along:** Add a comment inside `scene.frag` and run `cmake --build cmake-build-debug --target SparkShaders`. Confirm only affected `.spv` rebuilds.

### Step 3 — Manual compile (debugging)

When CMake paths confuse you, compile one file by hand:

```bash
cd /path/to/spark
glslangValidator -V --target-env vulkan1.0 -Ishaders shaders/scene.vert -o /tmp/scene.vert.spv
```

Syntax errors print with line numbers — faster than running the full engine.

### Step 4 — Runtime load

```cpp
// VulkanSpvShaderLoader.cpp (lines 12–53)
Array<char> VulkanSpvShaderLoader::ReadSpvFile(const char* filename) const {
    Utf8String pathUtf(SPARK_SHADER_SPV_DIR);
    pathUtf.AppendUtf8("/");
    pathUtf.AppendUtf8(filename);
    // fopen, fread entire file ...
}

VkShaderModule VulkanSpvShaderLoader::CreateShaderModule(const Array<char>& code) const {
    VkShaderModuleCreateInfo info{};
    info.codeSize = code.GetSize();
    info.pCode = reinterpret_cast<const uint32_t*>(code.GetData());
    vkCreateShaderModule(device, &info, nullptr, &shaderModule);
}
```

SPIR-V magic first word: `0x07230203` — useful sanity check in a hex dump.

### Step 5 — Attach to pipeline

```cpp
// VulkanScenePipeline.cpp (lines 52–67)
const Array<char> vertCode = shaders.ReadSpvFile("scene.vert.spv");
const Array<char> fragCode = shaders.ReadSpvFile("scene.frag.spv");
vertModule = shaders.CreateShaderModule(vertCode);
fragModule = shaders.CreateShaderModule(fragCode);
// VkPipelineShaderStageCreateInfo for VERTEX and FRAGMENT stages ...
```

Changing GLSL requires: rebuild SPIR-V → recreate **pipeline** (modules can be new or same handles after reload).

### Step 6 — Iteration checklist

1. Edit `shaders/scene.frag`
2. `cmake --build cmake-build-debug -j`
3. Run SparkDemo (no reconfigure unless you added new shader files to `SHADER_SOURCES`)
4. RenderDoc capture if change not visible — confirm pipeline timestamp

## 5. How This Connects to the Rest of Your Engine

Chapter **2** dissects `scene.vert` vertex processing. Chapter **4** aligns CPU `SceneUniformGpu` with `scene_ubo.glsl`. Part **IV** adds texture sampling in `scene.frag` binding 1.

The build pipeline is infrastructure — invest once, use for every pass (shadow, particles, UI, tonemap).

## 6. Pitfalls and Debugging

- **`failed to open shader`** — SPIR-V never built or wrong `SPARK_SHADER_SPV_DIR`; rebuild `SparkShaders`.
- **Edited .glsl, no visual change** — missing DEPENDS; touch `.vert` or clean build.
- **`vkCreateShaderModule` fail** — corrupt file; recompile; check git did not commit broken SPIR-V.
- **Added new.shader not in SHADER_SOURCES** — CMake never compiles it.
- **Include path wrong** — `-Ishaders` required for `#include "scene_ubo.glsl"`.

## 7. Exercises

1. Add `shaders/debug_magenta.frag` to `SHADER_SOURCES`, compile, load in a test harness.
2. Break `scene.vert` syntax deliberately — capture glslang error output.
3. Hex-dump first 16 bytes of `scene.vert.spv` — locate magic and version words.
4. List all `.glsl` files in DEPENDS — which are missing that you use in includes?
5. Measure wall time: full rebuild vs single shader rebuild with Ninja.

## Summary

Spark authors GLSL 450, compiles to SPIR-V via CMake `SparkShaders`, loads bytes with `VulkanSpvShaderLoader`, and attaches modules when creating graphics pipelines. Master DEPENDS on includes and `SPARK_SHADER_SPV_DIR` — the rest of Part III assumes this loop works reliably.

---

Next: [Vertex Shader Fundamentals](02-vertex-shader.md).
