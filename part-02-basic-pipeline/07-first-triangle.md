# Drawing Your First Triangle

> **Build-Along Milestone:** You can explain a minimal Vulkan draw with `triangle.vert`/`triangle.frag` — or recognize the same steps inside Spark's full scene path.

**You will learn:**

- Record `vkCmdDraw` with zero vertex buffers when positions are generated in the vertex shader.
- Use `gl_VertexIndex` for a three-vertex triangle list.
- Verify the full chain: render pass → pipeline → draw → submit → present.

## 1. The Problem We Are Solving

### If you have never rendered a 3D frame before

A **video game image** is not painted by hand each frame. The computer repeats a loop:

```text
while the game is running:
    1. Read input (keyboard, mouse)
    2. Update game state (move player, animate enemies)
    3. Build a list of "things to draw" (meshes, lights, camera)
    4. Tell the GPU to draw those things into an image
    5. Show that image on the monitor
```

Step 4 is **rendering**. The GPU is a specialized processor that can color millions of pixels per second by running small programs called **shaders** on every pixel (and every vertex).

**Vulkan** is the API (application programming interface) you use to talk to the GPU on Windows, Linux, and Android. It is explicit: you create objects, record commands, and submit them. Nothing happens until you call `vkQueueSubmit`.

Before you draw a character, a tree, or a lit PBR scene, you draw **one triangle**. If the triangle fails, the problem is always in the infrastructure (device, swapchain, render pass, pipeline, sync) — not in your mesh loader or material system. That is why this chapter exists.

### What "draw" means in Vulkan

A **draw call** is an instruction recorded into a **command buffer**:

```cpp
vkCmdDraw(commandBuffer, 3, 1, 0, 0);
```

This tells the GPU: "Run the currently bound graphics pipeline on **3 vertices** to produce **1 instance** of a primitive." With triangle-list topology, 3 vertices = 1 triangle.

The GPU then:

1. Runs the **vertex shader** once per vertex (positions → clip space).
2. **Rasterizes** the triangle (figures out which pixels are inside).
3. Runs the **fragment shader** once per covered pixel (outputs a color).

You do not call these shaders yourself — binding the pipeline connects your SPIR-V bytecode to the hardware.

Everything in Part II converges on one API call:

```cpp
vkCmdDraw(commandBuffer, vertexCount, instanceCount, firstVertex, firstInstance);
```

Before cubes, textures, and ECS, you need **one successful draw** — proof that shaders, render pass, pipeline, and sync from Part I work together. Spark's `triangle.vert` / `triangle.frag` are the smallest programs that produce colored fragments on screen.

You may not run a separate triangle binary in Spark — the engine jumps to scene shaders quickly — but the **steps are identical** with fewer bindings. This chapter is the checklist you debug against when the screen stays black.

## 1b. Concept Deep Dive — Minimal Draw vs Spark Scene Path

### vkCmdDraw(3) vs vkCmdDrawIndexed — same GPU stages

A tutorial triangle uses `vkCmdDraw` with positions hard-coded in the vertex shader via `gl_VertexIndex`. Spark's production path binds real geometry and calls `vkCmdDrawIndexed`:

```text
Triangle bootstrap                    Spark scene path
──────────────────                    ────────────────
triangle.vert (gl_VertexIndex)        scene.vert (vertex attributes)
empty vertex input state              8-attribute interleaved layout
vkCmdDraw(3, 1, 0, 0)                 vkCmdDrawIndexed(count, 1, firstIndex, vertexOffset, 0)
empty pipeline layout                 layout + set 0 + push constants
present render pass                   HDR render pass → tonemap → present
```

Both paths require: active render pass, bound pipeline, correct sync semaphores, and a submitted command buffer.

### DrawFrame — where the draw actually executes

The triangle appears only after `DrawFrame` completes the full Part I shell — wait fence, acquire image, write UBO, record, submit, present:

```cpp
// src/spark/render/core/VulkanRenderer.cpp (lines 488–508)
    frameSync.ResetCurrentFrameFence(device());

    WriteUniformBuffer(currentFrame);

    if (vkResetCommandBuffer(commandBuffers[imageIndex], 0) != VK_SUCCESS) {
        throw std::runtime_error("vkResetCommandBuffer failed");
    }
    RecordSceneCommandBuffer(commandBuffers[imageIndex], imageIndex, currentFrame);

    frameSync.SubmitFrame(graphicsQueue(), commandBuffers[imageIndex], imageIndex);

    const VkResult presentResult =
            frameSync.PresentFrame(presentQueue(), presentSwapchain().khr, imageIndex);
    frameSync.AdvanceFrame();
```

If the screen stays the swapchain clear color, the problem is before `vkCmdDraw*` — sync, render pass, or pipeline bind.

### Inside the render pass — bind, optional bind, draw

Spark's opaque pass follows the canonical order. For Part II with builtin meshes, the chain is pipeline → descriptors → vertex/index buffers → push constants → indexed draw:

```cpp
// src/spark/render/scene/VulkanSceneOpaquePass.cpp (lines 90–98, 206)
    vkCmdBindPipeline(commandBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, ctx.pipelineLit);
    vkCmdBindVertexBuffers(commandBuffer, 0, 1, &ctx.meshBindings.staticVertexBuffer, &vbOffset);
    vkCmdBindIndexBuffer(commandBuffer, ctx.meshBindings.staticIndexBuffer, 0, VK_INDEX_TYPE_UINT32);
    vkCmdBindDescriptorSets(commandBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, ctx.pipelineLayout, 0, 1, &ctx.descriptorSet, 0, nullptr);
    // ... per draw: vkCmdPushConstants, then:
    vkCmdDrawIndexed(commandBuffer, range.indexCount, 1, range.firstIndex, range.vertexOffset, 0);
```

Reduce mentally to triangle complexity: if scene draws fail, temporarily strip descriptors and custom meshes to isolate infrastructure.

### Debugging checklist mapped to Spark files

| Symptom | Check in Spark |
|---------|----------------|
| Black screen, no validation | `sceneParamsValid`, empty `pendingScene.draws` |
| Validation on draw | Render pass active? Pipeline compatible? |
| Wrong colors | SPIR-V stale — rebuild `SparkShaders` |
| Flickering | Per-flight UBO index mismatch with fence wait |
| Nothing on screen but no error | Tonemap pass — scene may render to HDR offscreen |

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

### Beginner walkthrough: every object you need before vkCmdDraw

Think of Vulkan objects as a checklist. For one colored triangle, create them **in this order**:

| Step | Object | Plain English | Spark / your engine |
|------|--------|---------------|---------------------|
| 1 | `VkInstance` | "Hello Vulkan, I am an app" | `VulkanDeviceContext::CreateInstance` |
| 2 | `VkSurfaceKHR` | Connection to the OS window | GLFW creates surface |
| 3 | `VkPhysicalDevice` | Choose a GPU | `SelectPhysicalDevice` |
| 4 | `VkDevice` | Logical handle to that GPU | `CreateLogicalDevice` |
| 5 | `VkSwapchainKHR` | Ring of images the monitor displays | `CreateSwapchainImages` |
| 6 | `VkRenderPass` | Description of attachments (color, depth) | `VulkanPresentRenderPass` or HDR pass |
| 7 | `VkFramebuffer` | Actual images bound to the render pass | Per swapchain image |
| 8 | `VkShaderModule` ×2 | Compiled vertex + fragment programs | `triangle.vert.spv`, `triangle.frag.spv` |
| 9 | `VkPipelineLayout` | What descriptors/push constants exist | Empty for triangle |
| 10 | `VkPipeline` | Full graphics state + shaders | `vkCreateGraphicsPipelines` |
| 11 | `VkCommandPool` + buffer | CPU-side list of GPU commands | `CreateCommandPool` |
| 12 | Sync (fence, semaphores) | Wait until GPU finishes | `VulkanFrameSync` |

**Per frame**, the sequence is:

```text
Wait until previous frame finished (fence)
  → Acquire next swapchain image (semaphore)
  → Reset + record command buffer:
        BeginRenderPass
        BindPipeline
        Draw(3 vertices)
        EndRenderPass
  → Submit command buffer to graphics queue (semaphore)
  → Present image to screen
```

Part I built steps 1–7 and 12. This chapter completes steps 8–11 for the smallest possible geometry.

### What gl_VertexIndex does (line by line)

In `triangle.vert`, Spark (and Vulkan tutorials) often avoid vertex buffers at first:

```glsl
// shaders/triangle.vert (concept)
void main() {
    vec2 positions[3] = vec2[](
        vec2(-0.5, -0.5),
        vec2( 0.5, -0.5),
        vec2( 0.0,  0.5)
    );
    gl_Position = vec4(positions[gl_VertexIndex], 0.0, 1.0);
}
```

- `vkCmdDraw(..., 3, ...)` invokes the vertex shader **3 times**.
- `gl_VertexIndex` is `0`, then `1`, then `2` — built into the GPU.
- `gl_Position` is a `vec4` in **clip space**. When `x,y` are between -1 and +1 and `w = 1`, the point is visible on screen.
- The fragment shader runs for every pixel inside the triangle and writes `outColor`.

No `VkBuffer` is bound because positions live **inside the shader**. Chapter 8 adds buffers when meshes have thousands of vertices.

### Clip space in one paragraph

The GPU ends vertex processing with coordinates where **visible screen** maps to a cube from (-1,-1,-1) to (+1,+1,+1) before perspective divide. You do not need to understand projection matrices yet — the triangle tutorial uses positions already in that range. When you add a camera in Part V, the vertex shader multiplies by `viewProj` to move world objects into clip space.

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
