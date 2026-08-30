# SPIR-V Shader Loading

> **Build-Along Milestone:** You can trace GLSL source → CMake `SparkShaders` → `VulkanSpvShaderLoader` → `VkShaderModule`.

**You will learn:**

- Explain why Vulkan uses offline SPIR-V instead of runtime GLSL compilation.
- Load `.spv` bytes from the build directory at runtime.
- Create and destroy `VkShaderModule` objects tied to pipeline lifetime.

## 1. The Problem We Are Solving

Part II needs GPU programs. Vulkan consumes **SPIR-V** — a binary intermediate representation — not GLSL source strings at runtime. Your workflow is:

```text
Author GLSL → compile offline (glslangValidator) → load bytes → vkCreateShaderModule
```

If this pipeline is slow or manual, you will avoid shader iteration. Spark automates compilation via CMake and loads from `SPARK_SHADER_SPV_DIR`. This chapter makes that path explicit so shader edits reliably show up in SparkDemo.

## 1b. Concept Deep Dive — From GLSL to VkShaderModule

### Offline compile, runtime load — why Spark ships .spv files

Vulkan drivers consume SPIR-V, not GLSL source. Spark's CMake `SparkShaders` target runs `glslangValidator -V` on every shader at build time; the engine reads binary bytes at startup. This keeps runtime fast and lets CI fail on shader syntax errors before any draw:

```text
shaders/scene.vert  ──glslangValidator──►  cmake-build/shaders/scene.vert.spv
shaders/scene.frag ──glslangValidator──►  cmake-build/shaders/scene.frag.spv
                                              │
                                              ▼
                              VulkanSpvShaderLoader::ReadSpvFile
                                              │
                                              ▼
                                    vkCreateShaderModule
                                              │
                                              ▼
                              VkPipelineShaderStageCreateInfo (stage + "main")
```

The `SparkEngine` target depends on `SparkShaders` — a clean build always produces fresh SPIR-V before linking.

### ReadSpvFile — absolute path from compile definition

`SPARK_SHADER_SPV_DIR` is baked in at compile time so the loader finds `.spv` regardless of working directory:

```cpp
// src/spark/render/gpu/VulkanSpvShaderLoader.cpp (lines 12–41)
Array<char> VulkanSpvShaderLoader::ReadSpvFile(const char* filename) const {
    Utf8String pathUtf(SPARK_SHADER_SPV_DIR);
    pathUtf.AppendUtf8("/");
    pathUtf.AppendUtf8(filename);
    FILE* file = std::fopen(pathUtf.CStr(), "rb");
    if (file == nullptr) {
        throw std::runtime_error(std::string("failed to open shader: ") + pathUtf.CStr());
    }
    // seek, read full file into Array<char>, fclose
    return buffer;
}
```

A `failed to open shader` exception almost always means `cmake --build` was not run after editing GLSL.

### CreateShaderModule — SPIR-V as uint32 word stream

Vulkan expects `codeSize` divisible by 4 and `pCode` pointing at the SPIR-V magic header (`0x07230203`):

```cpp
// src/spark/render/gpu/VulkanSpvShaderLoader.cpp (lines 44–53)
VkShaderModule VulkanSpvShaderLoader::CreateShaderModule(const Array<char>& code) const {
    VkShaderModuleCreateInfo shaderModuleCreateInfo{};
    shaderModuleCreateInfo.sType = VK_STRUCTURE_TYPE_SHADER_MODULE_CREATE_INFO;
    shaderModuleCreateInfo.codeSize = code.GetSize();
    shaderModuleCreateInfo.pCode = reinterpret_cast<const std::uint32_t*>(code.GetData());
    VkShaderModule shaderModule = VK_NULL_HANDLE;
    if (vkCreateShaderModule(device, &shaderModuleCreateInfo, nullptr, &shaderModule) != VK_SUCCESS) {
        throw std::runtime_error("vkCreateShaderModule failed");
    }
    return shaderModule;
}
```

Invalid SPIR-V from a corrupted file fails here; valid GLSL that fails glslang never reaches runtime.

### Modules wired into pipeline creation

`VulkanScenePipeline::CreateGraphicsPipeline` loads both stages, creates modules, then passes them to `vkCreateGraphicsPipelines`. Destroy order is pipelines first, then modules — encoded in `DestroyGraphicsPipeline`:

```cpp
// src/spark/render/scene/VulkanScenePipeline.cpp (lines 52–67)
    const Array<char> vertCode = shaders.ReadSpvFile("scene.vert.spv");
    const Array<char> fragCode = shaders.ReadSpvFile("scene.frag.spv");
    vertModule = shaders.CreateShaderModule(vertCode);
    fragModule = shaders.CreateShaderModule(fragCode);

    VkPipelineShaderStageCreateInfo vertShaderStage{};
    vertShaderStage.stage = VK_SHADER_STAGE_VERTEX_BIT;
    vertShaderStage.module = vertModule;
    vertShaderStage.pName = "main";
```

Chapter 7's `triangle.vert.spv` / `triangle.frag.spv` follow the identical loader path with empty vertex input state.

## 2. Core Concepts

### Why offline SPIR-V

| Approach | Pros | Cons |
|----------|------|------|
| Runtime GLSL compile | Flexible | Slow startup, large driver dependency |
| Offline SPIR-V | Fast load, CI can validate shaders | Extra build step |
| DXIL/Metallib (other APIs) | Platform-optimal | Not Vulkan's model |

Shipping `.spv` files (or embedding bytes) is standard for production Vulkan engines.

### glslangValidator invocation

Spark's CMake runs roughly:

```bash
glslangValidator -V shaders/scene.vert -o cmake-build-debug/shaders/scene.vert.spv \
  --target-env vulkan1.0 -Ishaders
```

`-Ishaders` resolves `#include "scene_ubo.glsl"`. The `SparkShaders` custom target lists every `.spv` output; the engine target depends on it.

### VkShaderModuleCreateInfo

SPIR-V is a `uint32_t` word stream:

```cpp
VkShaderModuleCreateInfo info{};
info.codeSize = bytes.size();
info.pCode = reinterpret_cast<const uint32_t*>(bytes.data());
vkCreateShaderModule(device, &info, nullptr, &module);
```

`codeSize` must be a multiple of 4. Destroy modules when destroying pipelines that reference them (or at shutdown).

### Shader entry points

`VkPipelineShaderStageCreateInfo::pName` is the GLSL `main` function unless you use `OpEntryPoint` names. Spark always uses `"main"`.

### Include dependencies

When you edit a shared header like `scene_ubo.glsl`, CMake must list it in `DEPENDS` so dependent `.vert`/`.frag` recompile. Missing DEPENDS causes stale SPIR-V — one of the hardest bugs to notice because only some uniforms change.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### VulkanSpvShaderLoader per device

**Concept:** VkShaderModule lifecycle

**Decision:** Small class holding `VkDevice`, methods `ReadSpvFile` and `CreateShaderModule`

**Why:** Centralizes path construction from `SPARK_SHADER_SPV_DIR` and error messages when files are missing.

**In Spark:** `include/spark/render/gpu/VulkanSpvShaderLoader.hpp`.

### Path from compile definition

**Concept:** Offline SPIR-V

**Decision:** `SPARK_SHADER_SPV_DIR` macro points at build output `shaders/`

**Why:** Running from IDE or CLI always finds the same relative layout without copying `.spv` manually.

**In Spark:** `spark/config.hpp` + `VulkanSpvShaderLoader::ReadSpvFile`.

### Modules owned by pipeline class

**Concept:** Destroy order

**Decision:** `VulkanScenePipeline` stores `vertModule` and `fragModule`, destroys in `DestroyGraphicsPipeline`

**Why:** Pipelines reference modules; destroy pipelines first, then modules — order encoded in one destructor path.

**In Spark:** `VulkanScenePipeline::DestroyGraphicsPipeline`.

### Separate triangle shaders for bootstrap

**Concept:** Iteration workflow

**Decision:** `triangle.vert` / `triangle.frag` in `shaders/` for minimal draws

**Why:** Smaller shaders compile faster while learning; scene shaders are large with includes.

**In Spark:** `shaders/triangle.vert`, `shaders/triangle.frag`.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book; Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `CMakeLists.txt` — `SparkShaders` target
- `src/spark/render/gpu/VulkanSpvShaderLoader.cpp`
- `shaders/triangle.vert`, `shaders/triangle.frag`
- `shaders/scene.vert`, `shaders/scene.frag`

### Step 1 — Minimal triangle vertex shader

Spark's tutorial triangle needs **no** vertex buffer — positions come from `gl_VertexIndex`:

```glsl
// shaders/triangle.vert
#version 450

vec2 positions[3] = vec2[](
    vec2(0.0, -0.5),
    vec2(0.5, 0.5),
    vec2(-0.5, 0.5)
);

void main() {
    gl_Position = vec4(positions[gl_VertexIndex], 0.0, 1.0);
}
```

```glsl
// shaders/triangle.frag
#version 450
layout(location = 0) out vec4 outColor;
void main() {
    outColor = vec4(0.15, 0.55, 0.95, 1.0);
}
```

### Step 2 — Read SPIR-V from disk

```cpp
// src/spark/render/gpu/VulkanSpvShaderLoader.cpp (lines 12–41)
Array<char> VulkanSpvShaderLoader::ReadSpvFile(const char* filename) const {
    Utf8String pathUtf(SPARK_SHADER_SPV_DIR);
    pathUtf.AppendUtf8("/");
    pathUtf.AppendUtf8(filename);
    FILE* file = std::fopen(pathUtf.CStr(), "rb");
    // seek, read full file into Array<char>, fclose
    return buffer;
}
```

If this throws `failed to open shader`, run `cmake --build` and confirm `.spv` exists under the build tree.

### Step 3 — Create shader module

```cpp
// VulkanSpvShaderLoader.cpp (lines 44–53)
VkShaderModule VulkanSpvShaderLoader::CreateShaderModule(const Array<char>& code) const {
    VkShaderModuleCreateInfo shaderModuleCreateInfo{};
    shaderModuleCreateInfo.sType = VK_STRUCTURE_TYPE_SHADER_MODULE_CREATE_INFO;
    shaderModuleCreateInfo.codeSize = code.GetSize();
    shaderModuleCreateInfo.pCode = reinterpret_cast<const std::uint32_t*>(code.GetData());
    VkShaderModule shaderModule = VK_NULL_HANDLE;
    if (vkCreateShaderModule(device, &shaderModuleCreateInfo, nullptr, &shaderModule) != VK_SUCCESS) {
        throw std::runtime_error("vkCreateShaderModule failed");
    }
    return shaderModule;
}
```

### Step 4 — Wire into pipeline creation

Scene pipeline loads both stages:

```cpp
const Array<char> vertCode = shaders.ReadSpvFile("scene.vert.spv");
const Array<char> fragCode = shaders.ReadSpvFile("scene.frag.spv");
vertModule = shaders.CreateShaderModule(vertCode);
fragModule = shaders.CreateShaderModule(fragCode);
```

### Step 5 — Verify iteration loop

1. Edit `triangle.frag` color
2. `cmake --build cmake-build-debug -j`
3. Run SparkDemo
4. Confirm color change — if not, `SparkShaders` did not rebuild; touch CMake or clean build.

## 5. How This Connects to the Rest of Your Engine

Chapter **3** consumed modules in `vkCreateGraphicsPipelines`. Part **III** explains GLSL details inside `scene.vert` and `scene.frag`. Part **VII** uses `triangle.*.spv` for the first draw.

Shader loading is not glamorous — but broken loading blocks every visual milestone.

## 6. Pitfalls and Debugging

- **Missing .spv after pull** — run build; SPIR-V often gitignored.
- **Wrong working directory** — `SPARK_SHADER_SPV_DIR` is absolute at compile time; run from build output paths.
- **Stale includes** — add `.glsl` to CMake DEPENDS.
- **Invalid SPIR-V** — glslang errors at compile time; fix before runtime.

## 7. Exercises

1. Find `SparkShaders` in root `CMakeLists.txt` — how many shader outputs are listed?
2. Hex-dump first 16 bytes of `scene.vert.spv` — recognize SPIR-V magic `0x07230203`.
3. Deliberately break GLSL syntax — confirm build fails at compile, not at `vkCreateShaderModule`.
4. Add a one-line comment to `triangle.frag` — measure rebuild time for iteration budget.

## Summary

Vulkan loads SPIR-V bytes into `VkShaderModule` objects. Spark compiles GLSL offline via CMake, reads `.spv` from `SPARK_SHADER_SPV_DIR`, and attaches modules to graphics pipelines. Establish a fast compile-run loop before tackling mesh drawing.

---

Next: [Vertex and Index Buffers](05-vertex-index-buffers.md).
