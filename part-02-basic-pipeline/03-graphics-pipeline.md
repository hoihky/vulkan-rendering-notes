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
