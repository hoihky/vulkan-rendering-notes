# Graphics Pipeline Fundamentals

> **Build-Along Milestone:** You can list the fixed-function and programmable stages in `VkGraphicsPipelineCreateInfo` and locate Spark's scene pipeline creation.

**You will learn:**

- Assemble shader stages, vertex input, rasterization, and blend state into one pipeline object.
- Explain why pipelines are baked at creation time in Vulkan.
- Match vertex attributes to mesh layout and GLSL `location` qualifiers.

## 1. The Problem We Are Solving

In OpenGL you changed shaders and state between draws with loose calls. In Vulkan, most **graphics state is immutable** inside a `VkPipeline` object created up front. A draw is:

```text
bind pipeline → bind descriptors → bind vertex buffers → vkCmdDraw*
```

This chapter walks through what Spark packs into `VulkanScenePipeline::CreateGraphicsPipeline` — the same structure you will use for a minimal triangle pipeline in Chapter 7.

### What is a graphics pipeline? (complete beginner)

In everyday terms, a **graphics pipeline** is a **recipe** the GPU follows for every draw call. The recipe includes:

- **Which programs run** (vertex shader, fragment shader)
- **How to read vertex data** (stride, attribute locations)
- **How triangles are assembled** (triangle list, strip, etc.)
- **Whether back faces are culled**
- **Whether depth testing is on**
- **Whether colors are blended with what's already in the framebuffer**
- **Which render pass format** the output must match

In OpenGL (older API), you changed this recipe with hundreds of loose function calls. In **Vulkan**, almost the entire recipe is frozen into one object: `VkPipeline`. If you want different blend mode, you create a **second pipeline** — you do not flip a global "blend on" switch.

```text
Analogy: OpenGL = cooking with adjustable knobs on the stove
         Vulkan   = pre-printed recipe cards; swap the whole card to change the dish
```

Spark creates three recipe cards for the same shaders:

| Pipeline | When used | Key difference |
|----------|-----------|----------------|
| `pipelineLit` | Opaque meshes | Depth write ON, blending OFF |
| `pipelineSky` | Sky dome | Depth write OFF, no culling |
| `pipelineLitTransparent` | Glass, foliage | Depth write OFF, alpha blend ON |

### The GPU pipeline stages (simplified)

You only write **vertex** and **fragment** shaders in this book. The hardware still runs fixed stages between them:

```text
[Vertex shader]     ← you write this (move vertices, pass UVs)
       ↓
[Clip / perspective] ← fixed function (hardware)
       ↓
[Rasterization]      ← fixed function (triangle → pixels)
       ↓
[Fragment shader]   ← you write this (color, lighting, textures)
       ↓
[Depth test / blend] ← configured in VkPipeline, not in GLSL
       ↓
[Framebuffer color attachment]
```

**Vertex shader** runs once per vertex. **Fragment shader** runs once per pixel (roughly — GPUs optimize). Lighting and texturing happen in the fragment shader in Spark's forward renderer.

### Why pipelines are created after the render pass

Vulkan needs to know the **format of the color attachment** (e.g. `R16G16B16A16_SFLOAT` for HDR, `B8G8R8A8_UNORM` for swapchain) when compiling the pipeline. That format comes from the **render pass** definition.

Spark's lit meshes render into an **HDR offscreen buffer**, not directly into the window. Therefore:

```text
Create HDR render pass  →  Create scene pipelines compatible with HDR pass
Create present pass     →  Create tonemap/UI pipelines compatible with swapchain
```

If you create a pipeline against the wrong render pass, `vkCreateGraphicsPipelines` may succeed but validation layers scream on the first draw.

## 1b. Concept Deep Dive — VulkanScenePipeline Internals

### One SPIR-V pair, three pipeline objects

Spark avoids creating a pipeline per material. The same `scene.vert.spv` / `scene.frag.spv` modules feed three `VkPipeline` handles that differ only in rasterization, depth, and blend state:

| Pipeline | Depth write | Cull | Blend | Use case |
|----------|-------------|------|-------|----------|
| `pipelineLit` | ON | BACK | OFF | Opaque meshes |
| `pipelineSky` | OFF | NONE | OFF | Infinite sky shell |
| `pipelineLitTransparent` | OFF | BACK | alpha | Glass, fade |

```cpp
// src/spark/render/scene/VulkanScenePipeline.cpp (lines 228–247)
    if (vkCreateGraphicsPipelines(device, VK_NULL_HANDLE, 1, &pipelineCreateInfo, nullptr, &pipelineLit) !=
            VK_SUCCESS) {
        throw std::runtime_error("VulkanScenePipeline: vkCreateGraphicsPipelines failed");
    }

    pipelineCreateInfo.pDepthStencilState = &depthStencilSky;
    pipelineCreateInfo.pRasterizationState = &rasterizerSky;
    if (vkCreateGraphicsPipelines(device, VK_NULL_HANDLE, 1, &pipelineCreateInfo, nullptr, &pipelineSky) !=
            VK_SUCCESS) {
        throw std::runtime_error("VulkanScenePipeline: vkCreateGraphicsPipelines (sky) failed");
    }

    pipelineCreateInfo.pDepthStencilState = &depthStencilTransparent;
    pipelineCreateInfo.pColorBlendState = &transparentColorBlending;
    if (vkCreateGraphicsPipelines(
                device, VK_NULL_HANDLE, 1, &pipelineCreateInfo, nullptr, &pipelineLitTransparent) !=
            VK_SUCCESS) {
        throw std::runtime_error("VulkanScenePipeline: vkCreateGraphicsPipelines (transparent) failed");
    }
```

Per-draw variation (model matrix, albedo, texture layer) comes from **push constants**, not rebinding pipelines.

### Vertex input — eight attributes, one binding

The pipeline describes how the GPU reads each interleaved vertex. Every `location` must match GLSL `layout(location = N)` in `scene.vert`:

```cpp
// src/spark/render/scene/VulkanScenePipeline.cpp (lines 71–86)
    using VL = VulkanSceneVertexLayout;
    VkVertexInputBindingDescription binding{};
    binding.binding = 0;
    binding.stride = VL::kStrideBytes;
    binding.inputRate = VK_VERTEX_INPUT_RATE_VERTEX;

    VkVertexInputAttributeDescription attrs[8]{};
    attrs[0].binding = 0;
    attrs[0].location = 0;
    attrs[0].format = VK_FORMAT_R32G32B32_SFLOAT;
    attrs[0].offset = sizeof(float) * VL::kOffPosition;
    attrs[1].binding = 0;
    attrs[1].location = 1;
    attrs[1].format = VK_FORMAT_R32G32B32_SFLOAT;
    attrs[1].offset = sizeof(float) * VL::kOffNormal;
```

Part II demos use position, normal, and UV; joints/weights slots stay zero for rigid meshes.

### Pipeline layout — descriptor set 0 + ModelPushConstants

Before `vkCreateGraphicsPipelines`, Spark creates a layout referencing the scene descriptor set layout and a push constant range sized to `ModelPushConstants`:

```cpp
// src/spark/render/scene/VulkanScenePipeline.cpp (lines 197–225)
    VkPushConstantRange pushConstantRange{};
    pushConstantRange.stageFlags = VK_SHADER_STAGE_VERTEX_BIT | VK_SHADER_STAGE_FRAGMENT_BIT;
    pushConstantRange.offset = 0;
    pushConstantRange.size = sizeof(VulkanSceneOpaquePass::ModelPushConstants);

    VkPipelineLayoutCreateInfo pipelineLayoutCreateInfo{};
    pipelineLayoutCreateInfo.setLayoutCount = 1;
    pipelineLayoutCreateInfo.pSetLayouts = &sceneDescriptorSetLayout;
    pipelineLayoutCreateInfo.pushConstantRangeCount = 1;
    pipelineLayoutCreateInfo.pPushConstantRanges = &pushConstantRange;
    // ...
    pipelineCreateInfo.layout = pipelineLayout;
    pipelineCreateInfo.renderPass = hdrRenderPass;
    pipelineCreateInfo.subpass = 0;
```

The HDR render pass handle is mandatory — Vulkan validates attachment format compatibility at pipeline creation time.

### Bootstrap order in VulkanRenderer

Pipelines cannot exist until descriptors, render pass, and shader modules are ready:

```text
CreatePersistentSceneResources()
  → sceneDescriptors.CreateSetLayout + uniform buffers
  → CreateSceneGeometry()           (static VB/IB)
RecreateSwapchain()
  → hdrTonemapPass.CreateHdrRenderPass()
  → scenePipeline.CreateGraphicsPipeline(device, hdrRenderPass, layout, shaders)
```

Resize triggers `RecreateSwapchain` only — persistent resources (layout, static geometry) survive if formats unchanged.

## 2. Core Concepts

### Programmable stages

| Stage | Shader | Spark file |
|-------|--------|------------|
| Vertex | `VK_SHADER_STAGE_VERTEX_BIT` | `scene.vert.spv` |
| Fragment | `VK_SHADER_STAGE_FRAGMENT_BIT` | `scene.frag.spv` |

Optional stages (geometry, tessellation, mesh) are unused in Spark's forward path. Each stage is a `VkShaderModule` plus entry point name (`"main"`).

### Vertex input state

`VkPipelineVertexInputStateCreateInfo` describes:

- **Bindings** — stride and per-vertex vs per-instance rate
- **Attributes** — `location`, `format`, offset into each vertex

Spark's lit mesh layout uses **one binding**, stride from `VulkanSceneVertexLayout::kStrideBytes`, six attributes (position, normal, UV, tangent, joint indices, weights).

Mismatch between C++ interleaved data and shader `layout(location = N)` causes validation errors or garbage vertices.

### Vertex layout: how C++ memory becomes shader inputs

Every mesh vertex in Spark is a contiguous block of floats in RAM. The GPU does not understand C++ structs — you must describe the layout with `VkVertexInputBindingDescription` and `VkVertexInputAttributeDescription`.

Spark's scene mesh uses **one binding** (binding index 0) with stride `VulkanSceneVertexLayout::kStrideBytes`:

```cpp
// src/spark/render/scene/VulkanScenePipeline.cpp (lines 71–110)
    using VL = VulkanSceneVertexLayout;
    VkVertexInputBindingDescription binding{};
    binding.binding = 0;
    binding.stride = VL::kStrideBytes;           // bytes between consecutive vertices
    binding.inputRate = VK_VERTEX_INPUT_RATE_VERTEX;  // advance per vertex, not per instance

    VkVertexInputAttributeDescription attrs[8]{};
    attrs[0].location = 0;  // matches layout(location = 0) in scene.vert
    attrs[0].format = VK_FORMAT_R32G32B32_SFLOAT;
    attrs[0].offset = sizeof(float) * VL::kOffPosition;
    attrs[2].location = 2;  // UV
    attrs[2].format = VK_FORMAT_R32G32_SFLOAT;
    attrs[2].offset = sizeof(float) * VL::kOffTexCoord0;
    // ... normal, tangent, joints, weights, texCoord1, color
```

**How to read this table when debugging:**

| Field | Meaning |
|-------|---------|
| `location` | Must match GLSL `layout(location = N)` |
| `format` | How many floats/ints to read (`R32G32B32_SFLOAT` = 3 floats) |
| `offset` | Byte offset from start of vertex struct |
| `binding` | Which vertex buffer stream (Spark uses only 0) |

At draw time you bind a `VkBuffer` and tell Vulkan "binding 0 starts at offset 0." The pipeline's vertex input state tells the GPU how to interpret each 80-byte (or whatever stride) chunk as position, normal, UV, etc.

**Common beginner mistake:** `sizeof(Vertex)` in C++ does not match `stride` because of padding. Spark centralizes offsets in `VulkanSceneVertexLayout` so pipeline creation, mesh upload, and glTF import all agree.

### Step-by-step: creating pipelineLit in Spark

When `RecreateSwapchain` runs after a window resize, Spark calls `VulkanScenePipeline::CreateGraphicsPipeline`. The function performs these steps in order:

1. **Load SPIR-V** from disk (`scene.vert.spv`, `scene.frag.spv`).
2. **Create shader modules** (`vkCreateShaderModule`).
3. **Describe vertex input** (8 attributes, one binding).
4. **Set input assembly** to `TRIANGLE_LIST`.
5. **Enable dynamic viewport/scissor** (set actual viewport during record).
6. **Configure rasterization** — fill mode, back-face cull, CCW front face.
7. **Configure depth** — test ON, write ON for opaque.
8. **Configure blend** — OFF for opaque (`pipelineLit`).
9. **Create pipeline layout** — descriptor set 0 + push constant range for `ModelPushConstants`.
10. **Call `vkCreateGraphicsPipelines`** with `hdrRenderPass` handle.
11. **Repeat steps 7–8** with different depth/blend for sky and transparent pipelines.

Steps 9–10 are why you cannot draw until descriptors exist (Part II Chapter 6) even though shaders compiled successfully.

### Building a Vulkan renderer pipeline from scratch

Spark does not create a single "mega pipeline" at startup. The **bootstrap sequence** in `VulkanRenderer`'s constructor shows the dependency order every engine must respect:

```text
VulkanDeviceContext (instance → surface → device → queues)
  → CreateCommandPool
  → CreatePersistentSceneResources (descriptors, static meshes, shadow resources)
  → RecreateSwapchain (swapchain images, HDR render pass, ALL graphics pipelines)
  → frameSync.Create (fences + semaphores)
```

The constructor wires this explicitly:

```cpp
// src/spark/render/core/VulkanRenderer.cpp (lines 43–54)
VulkanRenderer::VulkanRenderer(Window& inAppWindow) : deviceContext(inAppWindow), boundWindow(&inAppWindow) {
    shaderLoader.SetDevice(device());

    CreateCommandPool();
    CreatePersistentSceneResources();
    RecreateSwapchain();
    frameSync.Create(device());
    // ...
}
```

**Why this order matters:**

| Step | Depends on | Creates |
|------|------------|---------|
| `VulkanDeviceContext` | GLFW window | `VkInstance`, `VkDevice`, queues |
| `CreatePersistentSceneResources` | logical device | descriptor layout, uniform buffers, static cube/plane geometry |
| `RecreateSwapchain` | surface + device | swapchain, **HDR render pass**, scene/sprite/particle pipelines |
| `frameSync` | device | per-flight fences, acquire/present semaphores |

Graphics pipelines cannot be created until you know the **render pass handle** and **attachment formats**. Spark's scene pipelines target `hdrTonemapPass.HdrRenderPass()` — not the swapchain present pass. That is why pipeline creation lives inside `RecreateSwapchain`, not in the constructor's first line.

Inside `RecreateSwapchain`, Spark creates **three** scene pipelines from one shader pair (`scene.vert` / `scene.frag`):

```text
pipelineLit           — depth write ON,  blend OFF  (opaque meshes)
pipelineSky           — depth write OFF, cull NONE   (infinite sky shell)
pipelineLitTransparent — depth write OFF, alpha blend ON (glass, fade)
```

All three share one `VkPipelineLayout` (descriptor set 0 + `ModelPushConstants` push range). Per-draw variation comes from push constants — metallic, texture layers, shading model — not from rebinding pipelines unless the draw is sky or transparent.

When you build your own engine from scratch, replicate this layering:

1. **Device bootstrap** — isolate in a `DeviceContext` class (Spark: `VulkanDeviceContext`).
2. **Swapchain-dependent resources** — isolate in `RecreateSwapchain` so resize only rebuilds what must change.
3. **Pipeline triple** — opaque / sky / transparent from one SPIR-V pair; avoid N pipelines per material.
4. **Record pass** — delegate mesh draws to `VulkanSceneOpaquePass::Record` so `VulkanRenderer` stays an orchestrator.

### Input assembly

`topology` — `VK_PRIMITIVE_TOPOLOGY_TRIANGLE_LIST` for meshes. `primitiveRestartEnable` — false for standard indexed triangles.

### Viewport and scissor

Vulkan allows dynamic viewport/scissor (Spark sets them in `VulkanSceneOpaquePass::Record`). Fixed viewport in pipeline creation is optional when dynamic state is enabled.

### Rasterization

| Field | Spark typical value | Effect |
|-------|---------------------|--------|
| `polygonMode` | `FILL` | Solid triangles |
| `cullMode` | `BACK` | Cull back faces |
| `frontFace` | `COUNTER_CLOCKWISE` | Winding order |

Double-sided materials disable cull in a separate pipeline variant (`pipelineLit` vs double-sided draws).

### Depth and blend

- **Depth test** `LESS`, depth write on for opaque pass
- **Blend** off for opaque; separate `pipelineLitTransparent` enables alpha blending

### Pipeline layout

`VkPipelineLayout` lists descriptor set layouts and push constant ranges. Scene draws use **set 0** (UBO, textures, lights) and **push constants** for per-draw `model` matrix — created before the graphics pipeline because the pipeline references the layout handle.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### VulkanScenePipeline owns modules and pipelines

**Concept:** Immutable pipeline state

**Decision:** One class loads SPIR-V, creates modules, builds lit/sky/transparent pipelines

**Why:** Destroy order is modules → pipelines → layout. Colocating creation avoids leaking `VkShaderModule` when reloading shaders.

**In Spark:** `VulkanScenePipeline::CreateGraphicsPipeline` / `DestroyGraphicsPipeline`.

### Multiple pipelines, one layout

**Concept:** Depth and blend

**Decision:** `pipelineLit`, `pipelineLitTransparent`, `pipelineSky` share `pipelineLayout`

**Why:** Same descriptor bindings; only blend and depth write differ. Fewer layout objects means fewer descriptor set compatibility rules to debug.

**In Spark:** `VulkanScenePipeline.cpp` — three `vkCreateGraphicsPipelines` calls.

### Vertex layout in dedicated header

**Concept:** Vertex input state

**Decision:** `VulkanSceneVertexLayout` centralizes stride and attribute offsets

**Why:** `CreateSceneGeometry`, custom mesh upload, and pipeline creation all read the same constants — one source of truth.

**In Spark:** `include/spark/render/scene/VulkanSceneVertexLayout.hpp`.

### Pipelines tied to HDR render pass

**Concept:** Pipeline compatibility

**Decision:** Pass `hdrRenderPass` handle into `vkCreateGraphicsPipelines`

**Why:** Vulkan validates attachment formats and sample counts at pipeline creation time.

**In Spark:** `VulkanScenePipeline::CreateGraphicsPipeline(device, hdrRenderPass, ...)`.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book; Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `include/spark/render/scene/VulkanScenePipeline.hpp`
- `src/spark/render/scene/VulkanScenePipeline.cpp`
- `include/spark/render/scene/VulkanSceneVertexLayout.hpp`

### Step 1 — Load shader modules

```cpp
// src/spark/render/scene/VulkanScenePipeline.cpp (lines 52–67)
    const Array<char> vertCode = shaders.ReadSpvFile("scene.vert.spv");
    const Array<char> fragCode = shaders.ReadSpvFile("scene.frag.spv");
    vertModule = shaders.CreateShaderModule(vertCode);
    fragModule = shaders.CreateShaderModule(fragCode);

    VkPipelineShaderStageCreateInfo vertShaderStage{};
    vertShaderStage.sType = VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO;
    vertShaderStage.stage = VK_SHADER_STAGE_VERTEX_BIT;
    vertShaderStage.module = vertModule;
    vertShaderStage.pName = "main";

    VkPipelineShaderStageCreateInfo fragShaderStage{};
    fragShaderStage.stage = VK_SHADER_STAGE_FRAGMENT_BIT;
    fragShaderStage.module = fragModule;
    fragShaderStage.pName = "main";
```

Chapter 4 details SPIR-V loading; here focus on how modules plug into the pipeline.

### Step 2 — Vertex input description

```cpp
// VulkanScenePipeline.cpp (lines 71–76, abbreviated)
    using VL = VulkanSceneVertexLayout;
    VkVertexInputBindingDescription binding{};
    binding.binding = 0;
    binding.stride = VL::kStrideBytes;
    binding.inputRate = VK_VERTEX_INPUT_RATE_VERTEX;

    VkVertexInputAttributeDescription attrs[6]{};
    attrs[0].location = 0;  // position
    attrs[0].format = VK_FORMAT_R32G32B32_SFLOAT;
    // ... normal, UV, tangent, joints, weights
```

Compare with `scene.vert` input locations when debugging attribute mismatches.

### Step 3 — Fixed-function state blocks

The create call chains:

```text
VkGraphicsPipelineCreateInfo
  ├── stages[2]
  ├── pVertexInputState
  ├── pInputAssemblyState
  ├── pViewportState (or dynamic)
  ├── pRasterizationState
  ├── pMultisampleState
  ├── pDepthStencilState
  ├── pColorBlendState
  ├── layout (pipelineLayout)
  └── renderPass (hdrRenderPass)
```

Read Spark's file for exact `VkPipelineDepthStencilStateCreateInfo` — depth test enabled, write on for opaque.

### Step 4 — Bind pipeline before draw

```cpp
// VulkanSceneOpaquePass.cpp
    vkCmdBindPipeline(commandBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, ctx.pipelineLit);
```

Changing shader or blend state mid-pass requires binding a **different pipeline object**, not a loose uniform.

### Step 5 — Build a minimal pipeline checklist

For your triangle (Chapter 7):

1. Shader modules from `triangle.vert.spv` / `triangle.frag.spv`
2. Empty vertex input (hard-coded positions in shader via `gl_VertexIndex`)
3. Triangle list topology
4. Single color attachment format matching render pass
5. Pipeline layout with **no** descriptors initially

## 5. How This Connects to the Rest of Your Engine

Chapter **4** feeds shader bytes into module creation. Chapter **5** provides vertex buffers matching vertex input state. Chapter **6** adds descriptor set layout to `pipelineLayout`.

Part **III** edits `scene.vert` / `scene.frag` — every change requires rebuilding SPIR-V and recreating pipelines that reference those modules.

## 6. Pitfalls and Debugging

- **Pipeline render pass mismatch** — creation fails or validation on draw; recreate pipeline after pass change.
- **Wrong front face winding** — mesh looks inside-out; flip cull mode or index order.
- **Vertex stride wrong** — geometry "explodes"; compare `kStrideBytes` with sizeof interleaved vertex.
- **Forgetting pipeline layout push constant range** — `vkCmdPushConstants` validation errors.

## 7. Exercises

1. List five fields in `VkGraphicsPipelineCreateInfo` and what breaks if each is wrong.
2. How many `VkPipeline` objects does `VulkanScenePipeline` create for opaque, transparent, and sky?
3. Why must `pipelineLayout` outlive `VkPipeline`?
4. Compare `triangle.vert` (no vertex input) vs `scene.vert` (six attributes).

## Summary

A Vulkan graphics pipeline bundles shader stages and fixed-function state for a specific render pass. Spark's scene pipeline loads `scene.vert`/`scene.frag`, describes interleaved vertex layout, and creates separate pipelines for opaque and transparent blending. Bind pipeline, then descriptors, then buffers, then draw.

---

Next: [SPIR-V Shader Loading](04-spirv-loading.md).
