# Render Passes and Framebuffers

> **Build-Along Milestone:** You understand `VkRenderPass`, attachment descriptions, and how Spark binds swapchain or HDR images to framebuffers.

**You will learn:**

- Describe color and depth attachments in a render pass.
- Create framebuffers that pair image views with a render pass.
- Begin and end a render pass inside a command buffer.

## 1. The Problem We Are Solving

Vulkan does not draw to the screen directly. You **begin a render pass** on a **framebuffer** — a set of image views (color + optional depth) compatible with a **render pass** definition that describes load/store operations and layout transitions.

If the render pass and framebuffer formats disagree, `vkCreateFramebuffer` fails. If you skip the render pass, no `vkCmdDraw` is valid. This chapter explains the objects Spark creates before any mesh draw — the canvas your pipelines paint on.

## 2. Core Concepts

### Render pass = contract for attachments

`VkRenderPass` is a template describing:

| Attachment | Typical load op | Store op | Final layout |
|------------|-----------------|----------|--------------|
| Color | `CLEAR` or `LOAD` | `STORE` | `COLOR_ATTACHMENT_OPTIMAL` or `PRESENT_SRC` |
| Depth | `CLEAR` | `STORE` (if needed for SSAO) | `DEPTH_STENCIL_ATTACHMENT_OPTIMAL` |

Subpasses reference which attachments are color vs depth. Spark's HDR scene pass uses **one subpass** with color + depth for simplicity in Part II.

### Framebuffer = image views for one frame

A `VkFramebuffer` binds concrete `VkImageView` handles to attachment indices defined in the render pass. Swapchain has **N images** → **N framebuffers** (one per swapchain image). Depth may be per-swapchain-image or per-frame-in-flight depending on engine design.

```text
RenderPass (template)  +  ImageViews (instances)  →  Framebuffer
```

### Load and clear

When you `vkCmdBeginRenderPass` with clear values, the color attachment becomes your clear color and depth becomes 1.0 (far plane in Vulkan). Spark clears HDR color to black and depth to `1.0f` each frame before opaque draws.

### Spark's two-tier targets

Spark does not always draw directly to the swapchain image:

1. **HDR render pass** — scene draws into floating-point color + depth (`VulkanHdrTonemapPass`)
2. **Present render pass** — tonemap + UI composite into swapchain (`VulkanPresentRenderPass`)

Part II focuses on the HDR pass where `VulkanSceneOpaquePass` records draws. The pattern — render pass + framebuffer + `vkCmdBeginRenderPass` — is identical for both tiers.

### Attachment layout transitions

Render pass definitions include `initialLayout` and `finalLayout` for each attachment. Vulkan inserts barriers when you begin/end the pass. Manual barriers outside the pass are still needed when copying between HDR and swapchain — Spark handles those in tonemap and present passes.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Separate VulkanPresentRenderPass class

**Concept:** Render pass as template

**Decision:** `VulkanPresentRenderPass` owns only `VkRenderPass` for swapchain-format color (+ optional depth)

**Why:** Render pass lifetime exceeds individual framebuffers. Destroying swapchain images does not destroy the pass object if formats stay the same.

**In Spark:** `include/spark/render/present/VulkanPresentRenderPass.hpp`.

### Framebuffers in dedicated class

**Concept:** Framebuffer instances

**Decision:** `VulkanPresentationFramebuffers` creates one framebuffer per swapchain image view

**Why:** Recreate on resize destroys framebuffers but may reuse render pass when format unchanged.

**In Spark:** `VulkanPresentationFramebuffers::Create(device, renderPass, extent, colorImageViews, depthView)`.

### HDR pass for scene geometry

**Concept:** Two-tier targets

**Decision:** Scene draws target HDR color attachment, not swapchain directly

**Why:** Linear lighting, tonemap, and SSAO need floating-point color. Swapchain is often `sRGB` or `UNORM` unsuitable for accumulation.

**In Spark:** `RecordSceneCommandBuffer` uses `hdrTonemapPass.HdrRenderPass()` and per-flight HDR framebuffer.

### Clear color and depth each frame

**Concept:** Load and clear

**Decision:** Two `VkClearValue` entries passed to `vkCmdBeginRenderPass`

**Why:** Avoids ghosting from previous frame without full-screen quad overdraw. Depth clear enables correct occlusion for 3D meshes.

**In Spark:** `VulkanRenderer::RecordSceneCommandBuffer` — `clearColors[0]` color, `clearColors[1]` depth.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book; Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `include/spark/render/present/VulkanPresentRenderPass.hpp`
- `include/spark/render/present/VulkanPresentationFramebuffers.hpp`
- `src/spark/render/core/VulkanRenderer.cpp` — `RecordSceneCommandBuffer`
- `include/spark/render/post/VulkanHdrTonemapPass.hpp`

### Step 1 — Present render pass API

```cpp
// include/spark/render/present/VulkanPresentRenderPass.hpp
class VulkanPresentRenderPass {
public:
    VkRenderPass vkPass = VK_NULL_HANDLE;

    /** When depthFormat is VK_FORMAT_UNDEFINED, the pass is color-only (tonemap + UI). */
    void Create(VkDevice device, VkFormat swapchainImageFormat, VkFormat depthFormat = VK_FORMAT_UNDEFINED);
    void Destroy(VkDevice device) noexcept;
};
```

Depth is optional for passes that only blit a full-screen triangle to the swapchain.

### Step 2 — Framebuffer creation signature

```cpp
// include/spark/render/present/VulkanPresentationFramebuffers.hpp
class VulkanPresentationFramebuffers {
public:
    Array<VkFramebuffer> buffers;

    void Create(
            VkDevice device,
            VkRenderPass renderPass,
            VkExtent2D extent,
            const Array<VkImageView>& colorImageViews,
            VkImageView depthImageView = VK_NULL_HANDLE);
};
```

Each entry in `buffers` matches one swapchain image index.

### Step 3 — Begin render pass in command buffer

```cpp
// src/spark/render/core/VulkanRenderer.cpp (lines 247–269, abbreviated)
    VkClearValue clearColors[2]{};
    clearColors[0].color = { {0.0F, 0.0F, 0.0F, 1.0F} };
    clearColors[1].depthStencil = {1.0F, 0};

    VkRenderPassBeginInfo renderPassBeginInfo{};
    renderPassBeginInfo.sType = VK_STRUCTURE_TYPE_RENDER_PASS_BEGIN_INFO;
    renderPassBeginInfo.renderPass = hdrTonemapPass.HdrRenderPass();
    renderPassBeginInfo.framebuffer = hdrTonemapPass.Flight(frameIndex).framebuffer;
    renderPassBeginInfo.renderArea.offset = {0, 0};
    renderPassBeginInfo.renderArea.extent = presentSwapchain().extent;
    renderPassBeginInfo.clearValueCount = 2;
    renderPassBeginInfo.pClearValues = clearColors;

    vkCmdBeginRenderPass(commandBuffer, &renderPassBeginInfo, VK_SUBPASS_CONTENTS_INLINE);
```

All `vkCmdDraw` calls for the scene happen **between** this begin and the matching `vkCmdEndRenderPass`.

### Step 4 — Match formats when creating pipeline

Graphics pipelines are created with a **compatible render pass** handle. Changing HDR format requires recreating pipelines — another reason render pass is created early and kept stable.

### Step 5 — Mental model diagram

```text
Swapchain image view[i] ──► Present framebuffer[i] ──► Present render pass (tonemap/UI)
HDR color view[frame]   ──► HDR framebuffer[frame]  ──► HDR render pass (scene draws)
Depth view[frame]       ──► attached to HDR FB
```

## 5. How This Connects to the Rest of Your Engine

Chapter **3** creates a graphics pipeline compatible with the HDR render pass. Chapter **7** issues draws inside the pass begun here. Part **X** adds post-processing subpasses after the scene subpass ends.

Part I gave you swapchain **images**; this chapter binds them (or HDR equivalents) into **framebuffers** for drawing.

## 6. Pitfalls and Debugging

- **Render pass format ≠ image view format** — framebuffer creation fails; check `VkFormat` enums match.
- **Drawing outside begin/end** — validation error `VUID-vkCmdDraw-renderPass-02684`.
- **Wrong framebuffer index** — presenting image 2 while rendering to framebuffer 0 causes corruption.
- **Forgetting depth attachment in pass** — depth test silently disabled or validation complains about image layout.

## 7. Exercises

1. Sketch attachment descriptions for a pass with one color and one depth attachment.
2. How many framebuffers does Spark need for 3 swapchain images and HDR per-flight buffers?
3. What changes in `RenderPassBeginInfo` when the window resizes?
4. Read Vulkan spec on `initialLayout` — why must UNDEFINED images clear on first use?

## Summary

Render passes define how attachments are used; framebuffers bind specific image views to that template. Spark begins an HDR render pass each frame, clears color and depth, then records scene draws. Master begin/end before worrying about shader details.

---

Next: [Graphics Pipeline Fundamentals](03-graphics-pipeline.md).
