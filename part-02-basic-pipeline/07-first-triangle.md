# Drawing Your First Triangle

> **Build-Along Milestone:** You can explain a minimal Vulkan draw with `triangle.vert`/`triangle.frag` — or recognize the same steps inside Spark's full scene path.

**You will learn:**

- Record `vkCmdDraw` with zero vertex buffers when positions are generated in the vertex shader.
- Use `gl_VertexIndex` for a three-vertex triangle list.
- Verify the full chain: render pass → pipeline → draw → submit → present.

## 1. The Problem We Are Solving

Everything in Part II converges on one API call:

```cpp
vkCmdDraw(commandBuffer, vertexCount, instanceCount, firstVertex, firstInstance);
```

Before cubes, textures, and ECS, you need **one successful draw** — proof that shaders, render pass, pipeline, and sync from Part I work together. Spark's `triangle.vert` / `triangle.frag` are the smallest programs that produce colored fragments on screen.

You may not run a separate triangle binary in Spark — the engine jumps to scene shaders quickly — but the **steps are identical** with fewer bindings. This chapter is the checklist you debug against when the screen stays black.

## 2. Core Concepts

### Hard-coded vertices in the vertex shader

```glsl
vec2 positions[3] = vec2[]( ... );
void main() {
    gl_Position = vec4(positions[gl_VertexIndex], 0.0, 1.0);
}
```

`gl_VertexIndex` is 0, 1, 2 for the three vertices of `vkCmdDraw(3, ...)`. No vertex buffer binding required — `VkPipelineVertexInputStateCreateInfo` has zero bindings.

### Triangle list topology

`vkCmdDraw(3, 1, 0, 0)` draws one primitive with three vertices. Input assembly must be `VK_PRIMITIVE_TOPOLOGY_TRIANGLE_LIST`.

### Minimal pipeline layout

First bootstrap pipeline:

- No descriptor sets (empty `VkPipelineLayoutCreateInfo`)
- No push constants
- Fragment shader outputs solid color to `location = 0`

Add UBO when you need a camera matrix — not before you see **any** color.

### Where the draw lives in the command buffer

Order inside an active render pass:

```text
vkCmdBeginRenderPass
  vkCmdBindPipeline
  // optional: vkCmdBindDescriptorSets, vkCmdBindVertexBuffers
  vkCmdDraw
vkCmdEndRenderPass
```

Submit command buffer with semaphores from Part I. Present. Triangle appears.

### Spark's equivalent path

Spark's production path uses `vkCmdDrawIndexed` with scene geometry inside `VulkanSceneOpaquePass`. The triangle lesson maps as:

| Triangle tutorial | Spark scene path |
|-------------------|------------------|
| `triangle.vert.spv` | `scene.vert.spv` |
| No vertex buffer | `vertexBuffer` + index buffer |
| `vkCmdDraw(3,...)` | `vkCmdDrawIndexed(indexCount,...)` |
| Empty layout | `scenePipeline.PipelineLayout()` + descriptors |

If scene draws fail, reduce mentally to triangle complexity — isolate pipeline vs data.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Keep triangle shaders in repo

**Concept:** Hard-coded vertices

**Decision:** `shaders/triangle.vert` and `triangle.frag` remain in `SparkShaders` target

**Why:** Fast regression test when porting to new hardware; minimal repro for shader toolchain bugs.

**In Spark:** `shaders/triangle.vert`, `shaders/triangle.frag`.

### NDC coordinates directly

**Concept:** Triangle list topology

**Decision:** Positions already in clip space (-1..1) with `w = 1`

**Why:** No MVP matrix needed for first pixel — removes UBO from critical path.

**In Spark:** Triangle vertex shader positions array.

### Solid color fragment shader

**Concept:** Minimal pipeline layout

**Decision:** `outColor = vec4(0.15, 0.55, 0.95, 1.0)`

**Why:** Confirms fragment stage runs; lighting bugs isolated later.

**In Spark:** `triangle.frag`.

### Inline render pass contents

**Concept:** Command buffer order

**Decision:** `VK_SUBPASS_CONTENTS_INLINE` — all commands in primary buffer

**Why:** Spark records draws directly in `RecordSceneCommandBuffer`; secondary buffers unused in Part II.

**In Spark:** `vkCmdBeginRenderPass(..., VK_SUBPASS_CONTENTS_INLINE)`.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book; Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `shaders/triangle.vert`
- `shaders/triangle.frag`
- `src/spark/render/core/VulkanRenderer.cpp` — command buffer record (compare)
- Part I — `VulkanFrameSync` submit/present

### Step 1 — Study triangle shaders

Already in repo (Chapter 4). Confirm compile output:

```bash
ls cmake-build-debug/shaders/triangle.*.spv
```

### Step 2 — Create minimal pipeline (your engine or scratch)

Pseudocode matching Vulkan tutorial:

```cpp
// After Part I device + swapchain + render pass exist:
VkShaderModule vert = LoadSpv("triangle.vert.spv");
VkShaderModule frag = LoadSpv("triangle.frag.spv");

VkPipelineLayoutCreateInfo layoutInfo{}; // zero bindings
vkCreatePipelineLayout(device, &layoutInfo, nullptr, &pipelineLayout);

// Graphics pipeline: no vertex input, triangle list, one color attachment
vkCreateGraphicsPipelines(device, ..., &pipeline);

// Per swapchain image: allocate command buffer
```

Match color attachment format to your render pass (HDR in Spark, or swapchain for simplest test harness).

### Step 3 — Record one frame

```cpp
vkBeginCommandBuffer(cmd, &beginInfo);
vkCmdBeginRenderPass(cmd, &renderPassBegin, VK_SUBPASS_CONTENTS_INLINE);
vkCmdBindPipeline(cmd, VK_PIPELINE_BIND_POINT_GRAPHICS, pipeline);
vkCmdDraw(cmd, 3, 1, 0, 0);
vkCmdEndRenderPass(cmd);
vkEndCommandBuffer(cmd);
// Submit + Present (Part I frame sync)
```

### Step 4 — Compare Spark's indexed draw

When scene params are empty, opaque pass returns early. Add one `SceneDrawItem` with `UnitCube` (Chapter 8) — Spark uses:

```cpp
vkCmdDrawIndexed(commandBuffer, range.indexCount, 1, range.firstIndex, range.vertexOffset, 0);
```

Same bind pipeline → draw pattern, different draw command.

### Step 5 — Debug black screen checklist

| Check | Question |
|-------|----------|
| Pass begun? | Validation on `vkCmdDraw` |
| Pipeline bound? | Compatible render pass |
| Submit signaled? | Fence completes |
| Present waited on renderFinished? | Part I semaphores |
| Triangle inside clip volume? | NDC z in 0..w |

## 5. How This Connects to the Rest of Your Engine

Chapter **8** replaces `vkCmdDraw(3)` with indexed cube draws using scene shaders. Part **III** replaces triangle shaders with `scene.vert` for transforms and lighting.

The triangle is the **smallest proof** your Part I + Part II infrastructure works.

## 6. Pitfalls and Debugging

- **Drawing outside render pass** — validation error.
- **Viewport zero size** — after minimize; wait for nonzero extent.
- **Wrong swapchain image** — clear wrong buffer; verify `imageIndex`.
- **Culling enabled with wrong winding** — triangle culled; disable cull for debug.

## 7. Exercises

1. Change triangle color in `triangle.frag` — confirm build + run loop.
2. Draw twice with offset — what does `firstVertex` do vs `gl_VertexIndex`?
3. Capture frame in RenderDoc — how many draw calls for one triangle?
4. List five Vulkan objects that must exist before `vkCmdDraw` succeeds.

## Summary

Your first draw is `vkCmdDraw(3)` with triangle shaders and no vertex buffer, inside a begun render pass, after Part I sync. Spark's production path uses indexed scene draws with the same record-submit-present skeleton. Prove the skeleton with a triangle before debugging lit meshes.

---

Next: [Indexed Drawing — Cube and Ground](08-indexed-cube.md).
