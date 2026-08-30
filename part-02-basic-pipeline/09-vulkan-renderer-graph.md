# VulkanRenderer Frame Graph

> **Build-Along Milestone:** You can list the major record steps inside `RecordSceneCommandBuffer` and explain why shadows run before the HDR scene pass.

**You will learn:**

- Read Spark's per-frame GPU ordering as a frame graph.
- Identify where uploads, scene pass, post, 2D, and UI fit.
- Know which steps are essential in Part II vs added in later parts.

## 1. The Problem We Are Solving

`VulkanRenderer::DrawFrame` is more than one render pass. Each frame records **multiple passes** into one command buffer (or few), with ordering constraints:

- Upload textures before sampling
- Shadow maps before lit shading reads them
- Scene HDR before tonemap
- Tonemap before UI overlay

This **frame graph** is the roadmap for the rest of the book. Part II focuses on the **HDR scene subpass** inside `RecordSceneCommandBuffer`; this chapter shows where that slice sits so you do not get lost in later parts.

### Frame graph for beginners

A **frame** is one image on your monitor. A **frame graph** is the **ordered list of GPU work** your engine records each frame. Think of it as a recipe card pinned above the GPU:

```text
Frame N:
  [Upload new texture pixels]
  [Draw shadow maps]
  [Draw 3D scene into HDR buffer]
  [Draw 2D sprites into same HDR buffer]
  [Tonemap HDR → window]
  [Draw UI on top]
  [Show on screen]
```

Order matters. You cannot tonemap before the scene is drawn. You cannot sample shadow maps before they are rendered. Spark encodes this order in one function: `VulkanRenderer::RecordSceneCommandBuffer` — not a fancy graph editor, but the same idea AAA engines use with more tooling.

### CPU vs GPU: who does what each frame

| Phase | Runs on | Spark entry point | What it does |
|-------|---------|-------------------|--------------|
| Gameplay | CPU | `game->OnUpdate` | Move entities, physics, input |
| Scene gather | CPU | `FillStandardLitSceneFromWorld` | Build draw list from components |
| Set snapshot | CPU | `SetSceneRenderParams` | Hand draw list to renderer |
| Record commands | CPU | `RecordSceneCommandBuffer` | Write vkCmd* into command buffer |
| Execute | GPU | Queue submit | Actually run draws |
| Present | GPU + OS | `vkQueuePresentKHR` | Show swapchain image |

Beginners often profile only the GPU and wonder why frames stutter — the CPU might still be building `SceneRenderParams` after the deadline. Both sides must finish before v-sync.

### The full Spark frame (expanded)

```text
Engine::Run loop (every frame):
│
├─ OnUpdate()                    ← simulation (CPU)
├─ OnRender() → SceneSubmit      ← build SceneRenderParams (CPU)
├─ PresentFrame()
│    └─ DrawFrame()
│         ├─ WaitForFence        ← don't overwrite GPU-in-use buffers
│         ├─ AcquireSwapchainImage
│         ├─ WriteUniformBuffer  ← camera, lights for this flight index
│         ├─ RecordSceneCommandBuffer:
│         │    ├─ Upload textures/meshes
│         │    ├─ Shadow passes
│         │    ├─ HDR scene (opaque → maybe transparent resume)
│         │    ├─ 2D composite + particles
│         │    ├─ End HDR
│         │    ├─ SSAO (optional)
│         │    ├─ Tonemap → swapchain
│         │    └─ UI + ImGui
│         ├─ QueueSubmit
│         └─ QueuePresent
└─ Advance frame index
```

When learning, pick **one horizontal slice** (e.g. "HDR scene opaque pass") and ignore the rest until it works. Use this diagram to see where your slice sits.

## 1b. Concept Deep Dive — RecordSceneCommandBuffer Ordering

### Full GPU timeline inside one command buffer

`RecordSceneCommandBuffer` is Spark's frame graph encoded as imperative C++. Order is fixed — later passes depend on earlier outputs:

```text
vkBeginCommandBuffer
  ├─ deferredUploadBatch.Record          (textures, UI atlases)
  ├─ customMeshPool.RecordUploads        (dynamic mesh staging)
  ├─ RecordShadowMapPass                 (directional + punctual)
  ├─ hdrTonemapPass barriers
  ├─ vkCmdBeginRenderPass (HDR)
  │    ├─ sceneOpaquePass.Record         ← Part II focus
  │    ├─ [transparent resume if needed]
  │    ├─ composite2DPass.Record         (tilemaps + sprites)
  │    └─ particlePass.Record
  ├─ vkCmdEndRenderPass (HDR)
  ├─ hdrTonemapPass.TransitionColorToShaderRead
  ├─ screenSpaceEffectsPass (SSAO, optional)
  ├─ hdrTonemapPass.RecordTonemap        (HDR → swapchain)
  └─ screenUi / ImGui overlay
vkEndCommandBuffer
```

Part II needs only the upload + HDR begin + opaque pass + end + tonemap slice; shadows and 2D layers can run empty when params have no shadow casters or sprites.

### Entry point — uploads before any render pass

The function opens with resource uploads and shadow recording, then transitions depth and begins the HDR pass:

```cpp
// src/spark/render/core/VulkanRenderer.cpp (lines 217–290)
void VulkanRenderer::RecordSceneCommandBuffer(
        VkCommandBuffer commandBuffer, std::uint32_t imageIndex, std::uint32_t frameIndex) {
    vkBeginCommandBuffer(commandBuffer, &beginInfo);

    deferredUploadBatch.Record(commandBuffer, device(), sceneTextureUploader, sceneHdrTextureUploader, screenUi);
    customMeshPool.RecordUploads(commandBuffer);
    RecordShadowMapPass(commandBuffer, frameIndex);

    hdrTonemapPass.BeginColorAttachmentBarrierIfNeeded(commandBuffer, frameIndex);
    // ... depth UNDEFINED → DEPTH_STENCIL_ATTACHMENT_OPTIMAL barrier ...
    vkCmdBeginRenderPass(commandBuffer, &renderPassBeginInfo, VK_SUBPASS_CONTENTS_INLINE);
```

Textures must upload **before** the fragment shader samples them. Shadow maps must render **before** lit shading reads them — same dependency rule as tonemap-after-scene.

### Opaque pass context — everything Part II wires together

Inside the HDR render pass, Spark assembles one struct carrying scene snapshot, pipelines, descriptors, and mesh bindings:

```cpp
// src/spark/render/core/VulkanRenderer.cpp (lines 292–320)
        const VulkanSceneOpaqueRecordContext opaqueCtx{
                .scene = &pendingScene,
                .sceneParamsValid = sceneParamsValid,
                .pipelineLit = scenePipeline.PipelineLit(),
                .pipelineLayout = scenePipeline.PipelineLayout(),
                .descriptorSet = sceneDescriptors.DescriptorSet(frameIndex),
                .meshBindings = { .staticVertexBuffer = vertexBuffer, /* ... */ },
        };
        sceneOpaquePass.Record(commandBuffer, opaqueCtx);
```

If `sceneParamsValid` is false or `draws` is empty, `Record` returns immediately — you get a cleared HDR buffer with no geometry.

### DrawFrame outer shell — CPU work before record

The frame graph's CPU half runs in `DrawFrame` before `RecordSceneCommandBuffer` is called:

```cpp
// src/spark/render/core/VulkanRenderer.cpp (lines 460–495)
    frameSync.WaitForSwapchainImageFence(device(), imageIndex);
    deferredUploadBatch.Prepare(/* texture upload scheduling */);
    customMeshPool.UpdateFromScene(pendingScene, submittedFrameCounter, kMaxFramesInFlight);
    WriteUniformBuffer(currentFrame);
    RecordSceneCommandBuffer(commandBuffers[imageIndex], imageIndex, currentFrame);
    frameSync.SubmitFrame(graphicsQueue(), commandBuffers[imageIndex], imageIndex);
```

Profiling stutter? Measure both `FillStandardLitSceneFromWorld` (CPU) and `RecordSceneCommandBuffer` (CPU record + GPU execute).

## 2. Core Concepts

### DrawFrame outer shell (Part I)

From Part I Chapter 10:

```text
Wait fence → Acquire image → Wait image fence → Reset fence
→ Uploads / uniform write → Record command buffer → Submit → Present → Advance frame
```

The frame graph lives **inside** record.

### RecordSceneCommandBuffer high-level order

Spark's scene command buffer roughly follows:

```text
1. Begin command buffer
2. Deferred texture / mesh uploads (barriers)
3. Shadow map passes (Part IX)
4. HDR color attachment barrier (if needed)
5. Depth image layout barrier (first use)
6. Begin HDR render pass (clear color + depth)
7. Scene opaque + transparent draws  ← Part II core
8. End HDR render pass
9. SSAO / post / tonemap (Part X)
10. Present pass / UI / sprites (Parts X–XI)
11. End command buffer
```

Part II: study steps 6–8 in detail; treat others as "reserved slots."

### Single primary command buffer per swapchain image

Spark resets and records `commandBuffers[imageIndex]` each frame. All passes for that frame append to the same buffer for simplicity — not a full DAG scheduler like AAA engines, but **explicit ordering** you can read top-to-bottom.

### Upload batch before draws

`deferredUploadBatch.Record` copies staging data to GPU images and buffers **before** draws that reference them. Same frame's draws must not sample textures mid-upload without barriers — batch collects work then records transfers first.

### Why shadows before scene

Lit fragment shader samples shadow maps. Depth attachments from shadow passes must complete before the main pass reads them as combined image samplers — hence shadow passes appear **earlier** in the command buffer.

### pendingScene consumption

`RecordSceneCommandBuffer` reads `pendingScene` and `sceneParamsValid` captured when the game called `SetSceneRenderParams` **before** `PresentFrame`. The snapshot is stable for the entire record/submit of that frame.

### Bootstrap: what exists before the first draw

Spark's renderer constructor builds GPU infrastructure in a fixed order. Treat this as the checklist when implementing your own Vulkan engine:

```cpp
// src/spark/render/core/VulkanRenderer.cpp (lines 43–54)
VulkanRenderer::VulkanRenderer(Window& inAppWindow) : deviceContext(inAppWindow), boundWindow(&inAppWindow) {
    shaderLoader.SetDevice(device());
    CreateCommandPool();
    CreatePersistentSceneResources();  // descriptors, static geometry, shadow atlases
    RecreateSwapchain();               // swapchain + HDR pass + all pipelines
    frameSync.Create(device());
}
```

`VulkanDeviceContext` (created as the first member) already owns instance, surface, physical/logical device, and queues. Swapchain images are created inside `RecreateSwapchain`, not in the device constructor — window size may still be settling at startup.

`CreatePersistentSceneResources` allocates resources that **do not** depend on swapchain extent: scene descriptor layout and pool, uniform buffers, skin SSBOs, static unit-cube vertex/index buffers, shadow map atlases. These survive resize without recreation.

`RecreateSwapchain` is the resize hub: depth buffers, HDR + present render passes, HDR flight framebuffers, scene/sprite/particle/tonemap pipelines, presentation framebuffers, and per-swapchain-image command buffers.

### Transparent draws and the HDR resume pass

When `pendingScene.transparentDraws` is non-empty, Spark **ends** the HDR render pass early, copies the opaque HDR color to `opaqueBackground`, then **resumes** HDR with `LOAD` (not `CLEAR`) so depth from opaque geometry is preserved:

```text
vkCmdBeginRenderPass (HDR, CLEAR)
  → sceneOpaquePass.Record (opaque + sky)
  → vkCmdEndRenderPass
  → copy opaque HDR color → sceneOpaqueColor texture (binding 13)
  → vkCmdBeginRenderPass (HDR resume, LOAD depth + color)
  → sceneOpaquePass.RecordTransparent
  → composite2D + particles
vkCmdEndRenderPass
```

Transmission and screen-space refraction in `gltf_pbr_extensions.glsl` sample `sceneOpaqueColor` — the copy must happen between opaque and transparent recording. This is why transparent rendering is not "just another draw call" in the same subpass; it is a deliberate render-pass split tied to PBR extensions.

### Descriptor set 0 at record time

Scene draws bind **one descriptor set per frame-in-flight index**:

```text
binding 0  — SceneUniformGpu UBO (viewProj, lights, exposure)
binding 1  — sceneTextures (sampler2DArray, LDR materials)
binding 11 — sceneHdrTextures (HDR emissive / float albedo)
binding 13 — sceneOpaqueColor (transmission background)
```

Per-draw material selection does **not** change descriptors — push constants carry `textureLayer`, `normalMapLayer`, `metallicRoughnessMapLayer`, and glTF extension fields. See Part IV binding chapter and Part VII material submit.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### RecordSceneCommandBuffer centralizes ordering

**Concept:** Frame graph

**Decision:** One method sequences uploads, shadows, HDR scene, post — called from `DrawFrame`

**Why:** Avoids scattered `vkCmd*` calls across demos; ordering bugs fixed in one file.

**In Spark:** `VulkanRenderer::RecordSceneCommandBuffer`.

### VulkanSceneOpaquePass as sub-recorder

**Concept:** HDR scene subpass

**Decision:** Scene mesh draws delegated to `sceneOpaquePass.Record`

**Why:** Keeps thousand-line renderer file manageable; scene pass testable with mock context.

**In Spark:** `VulkanSceneOpaquePass::Record` / `RecordTransparent`.

### HDR before present pass

**Concept:** Post ordering

**Decision:** Lit scene renders to HDR float target; tonemap writes swapchain

**Why:** Linear lighting and exposure in Part X need off-screen HDR buffer.

**In Spark:** `VulkanHdrTonemapPass` between scene and present.

### Per-image command buffers

**Concept:** Single primary buffer per image

**Decision:** `commandBuffers` sized to swapchain image count

**Why:** Acquired `imageIndex` selects which buffer to record — matches image lifetime.

**In Spark:** `VulkanRenderer` member `commandBuffers`.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book; Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `src/spark/render/core/VulkanRenderer.cpp` — `DrawFrame`, `RecordSceneCommandBuffer`
- `src/spark/render/scene/VulkanSceneOpaquePass.cpp`
- `include/spark/render/post/VulkanHdrTonemapPass.hpp`

### Step 1 — Entry from DrawFrame

```cpp
// VulkanRenderer.cpp (lines 451–458, abbreviated)
    frameSync.ResetCurrentFrameFence(device());
    WriteUniformBuffer(currentFrame);

    vkResetCommandBuffer(commandBuffers[imageIndex], 0);
    RecordSceneCommandBuffer(commandBuffers[imageIndex], imageIndex, currentFrame);

    frameSync.SubmitFrame(graphicsQueue(), commandBuffers[imageIndex], imageIndex);
```

Uniform write happens **after** fence wait — safe to touch per-flight UBO.

### Step 2 — Open RecordSceneCommandBuffer

```cpp
// VulkanRenderer.cpp (lines 201–213, abbreviated)
void VulkanRenderer::RecordSceneCommandBuffer(...) {
    vkBeginCommandBuffer(commandBuffer, &beginInfo);

    deferredUploadBatch.Record(commandBuffer, device(), sceneTextureUploader, screenUi);
    customMeshPool.RecordUploads(commandBuffer);
    RecordShadowMapPass(commandBuffer, frameIndex);
```

Part II: uploads may be no-ops without textures; shadow pass skips if disabled.

### Step 3 — HDR pass and scene draws

```cpp
    // depth barrier for sceneDepthResources[frameIndex] ...
    vkCmdBeginRenderPass(commandBuffer, &renderPassBeginInfo, VK_SUBPASS_CONTENTS_INLINE);

    if (frameIndex < sceneDescriptors.DescriptorSetCount()) {
        VulkanSceneOpaqueRecordContext opaqueCtx{ ... };
        sceneOpaquePass.Record(commandBuffer, opaqueCtx);
        sceneOpaquePass.RecordTransparent(commandBuffer, opaqueCtx, customDrawPackedTransparent);
    }

    vkCmdEndRenderPass(commandBuffer);
    // post passes follow ...
```

This is where Chapter 8's draws execute.

### Step 4 — Draw a simplified graph on paper

For Part II only, erase nodes you have not built yet:

```text
Acquire → Record: [ Clear HDR+Depth → Draw meshes ] → Submit → Present
```

Add shadow, SSAO, sprites nodes as you complete later parts.

### Step 5 — Trace one demo frame

Set breakpoint on `sceneOpaquePass.Record`. Run SparkDemo 3D mode. Confirm `pendingScene.draws` is non-empty when hook hits.

## 5. How This Connects to the Rest of Your Engine

| Part | Adds to frame graph |
|------|---------------------|
| IX | `RecordShadowMapPass` |
| X | SSAO, tonemap |
| XI | Sprite/tilemap passes |
| XI | Screen UI |

Part II's graph node is **HDR scene opaque pass**. Everything else hangs off the same command buffer recording discipline.

## 6. Pitfalls and Debugging

- **Recording before acquire** — wrong image index buffer.
- **Post before scene** — undefined HDR input to tonemap.
- **sceneParamsValid false** — early return; empty scene.
- **Missing EndRenderPass** — validation on submit.

## 7. Exercises

1. Print line numbers of each major section in `RecordSceneCommandBuffer`.
2. Which passes run when `pendingScene.draws` is empty but UI is active?
3. Why is `WriteUniformBuffer` called before record, not after?
4. Compare Spark frame graph to Part I's three-layer diagram — where does GPU fit?

## Summary

`VulkanRenderer` records a ordered sequence of passes each frame inside one command buffer. Part II cares about the HDR render pass and `VulkanSceneOpaquePass` — uploads before, post after. Use the full graph as a map when later parts add shadows, tonemap, and 2D.

---

Next: [Submitting a Scene from Code](10-submitting-scene.md).
