# Frame Synchronization

> **Build-Along Milestone:** You can diagram acquire → wait → record → submit → present and explain Spark's two frames in flight.

**You will learn:**

- Use fences to wait for GPU completion before reusing CPU-written data.
- Use semaphores to synchronize acquire, render, and present queue operations.
- Implement double-buffered frames in flight without tearing or validation errors.

## 1. The Problem We Are Solving

The CPU records commands **faster** than the GPU executes them. The display shows **previous** frames while you write the next. Without synchronization you will:

- Reuse uniform buffers while the GPU still reads them (flickering transforms).
- Present an image still being written (tearing).
- Submit work that references a swapchain image another frame owns.

This chapter completes **Part I**: after it, Spark clears or renders to the swapchain every frame with correct Vulkan sync. Everything in Part II adds draws inside the same sync skeleton.

## 2. Core Concepts

### CPU–GPU timeline

```text
Frame N (CPU):   write UBO → record CB → submit ───────────────►
Frame N (GPU):                    wait ... execute ... present ─►
Frame N+1 (CPU):                              write UBO → ...
```

The CPU must not overwrite UBO slot `i` until the fence for the submission that used slot `i` signals.

### Fences vs semaphores

| Primitive | CPU waits? | GPU waits? | Spark usage |
|-----------|------------|------------|-------------|
| `VkFence` | `vkWaitForFences` | signals when work done | Per frame in flight |
| `VkSemaphore` | optional | `vkQueueSubmit` / present wait | acquire → render → present |

Fences are **CPU-GPU** sync. Semaphores are **GPU-GPU** sync between queue operations.

### Frames in flight

Spark uses **`kMaxFramesInFlight = 2`**: two parallel CPU recordings while GPU works on the previous submission. More frames increase latency; one frame simplifies debugging but stalls CPU on every present.

### Swapchain image vs frame index

Swapchain may have **3 images** while you have **2 frames in flight**. Spark tracks:

- `currentFrame` — indexes per-flight fences and `imageAvailable` semaphores
- `imageIndex` — which swapchain image was acquired this submission
- `renderFinishedSemaphores[imageIndex]` — one semaphore **per swapchain image** for present wait

This avoids signaling the wrong present semaphore when image count ≠ frames in flight.

### Acquire → submit → present

```text
vkAcquireNextImageKHR → imageAvailable semaphore
vkQueueSubmit         → wait imageAvailable, signal renderFinished[imageIndex], signal fence
vkQueuePresentKHR     → wait renderFinished[imageIndex]
```

`OUT_OF_DATE` from acquire or present triggers swapchain recreation (Chapter 8).

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### VulkanFrameSync owns all sync objects

**Concept:** Fences vs semaphores

**Decision:** Dedicated class; renderer calls `WaitForCurrentFrameFence`, `SubmitFrame`, `PresentFrame`

**Why:** `DrawFrame` is already long — isolating sync prevents copy-paste errors across demos.

**In Spark:** `include/spark/render/core/VulkanFrameSync.hpp`.

### Fences created signaled

**Concept:** Frames in flight

**Decision:** `VK_FENCE_CREATE_SIGNALED_BIT` on create

**Why:** First frame's `WaitForFences` would otherwise block forever — no prior submission signaled the fence.

**In Spark:** `VulkanFrameSync::Create` — fence create info flags.

### imagesInFlight per swapchain image

**Concept:** Swapchain image vs frame index

**Decision:** `WaitForSwapchainImageFence` before writing to acquired image

**Why:** Same swapchain image must not be acquired twice while GPU still uses it from an older submission.

**In Spark:** `VulkanFrameSync::TrackSwapchainImageInFlight`.

### Reset fence after wait, before record

**Concept:** CPU–GPU timeline

**Decision:** `ResetCurrentFrameFence` after waits, before command buffer record

**Why:** Next submission to the same slot needs an unsignaled fence to attach to `vkQueueSubmit`.

**In Spark:** `VulkanRenderer::DrawFrame` — after uploads, before `vkResetCommandBuffer`.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book; Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `include/spark/render/core/VulkanFrameSync.hpp`
- `src/spark/render/core/VulkanFrameSync.cpp`
- `src/spark/render/core/VulkanRenderer.cpp` — `DrawFrame`

### Step 1 — Create per-flight sync

```cpp
// VulkanFrameSync.cpp (lines 7–23)
void VulkanFrameSync::Create(VkDevice device) {
    imageAvailableSemaphores.Resize(kMaxFramesInFlight);
    inFlightFences.Resize(kMaxFramesInFlight);

    VkSemaphoreCreateInfo semaphoreCreateInfo{};
    semaphoreCreateInfo.sType = VK_STRUCTURE_TYPE_SEMAPHORE_CREATE_INFO;

    VkFenceCreateInfo fenceCreateInfo{};
    fenceCreateInfo.sType = VK_STRUCTURE_TYPE_FENCE_CREATE_INFO;
    fenceCreateInfo.flags = VK_FENCE_CREATE_SIGNALED_BIT;

    for (std::size_t i = 0; i < kMaxFramesInFlight; ++i) {
        vkCreateSemaphore(device, &semaphoreCreateInfo, nullptr, &imageAvailableSemaphores[i]);
        vkCreateFence(device, &fenceCreateInfo, nullptr, &inFlightFences[i]);
    }
}
```

### Step 2 — Recreate per-swapchain semaphores

When swapchain image count changes, rebuild `renderFinishedSemaphores` and `imagesInFlight`:

```cpp
void VulkanFrameSync::RecreateSwapchainSync(VkDevice device, std::size_t swapchainImageCount) {
    DestroySwapchainSync(device);
    renderFinishedSemaphores.Resize(swapchainImageCount);
    // create one semaphore per image ...
    imagesInFlight.Resize(swapchainImageCount);
    for (auto& fence : imagesInFlight) {
        fence = VK_NULL_HANDLE;
    }
}
```

### Step 3 — DrawFrame sync sequence

```cpp
// VulkanRenderer.cpp (lines 398–426, 451–471, abbreviated)
void VulkanRenderer::DrawFrame() {
    if (framebufferResized) {
        RecreateSwapchain();
        return;
    }

    frameSync.WaitForCurrentFrameFence(device());

    std::uint32_t imageIndex = 0;
    const VkResult acquireResult =
            frameSync.AcquireNextImage(device(), presentSwapchain().khr, imageIndex);

    if (acquireResult == VK_ERROR_OUT_OF_DATE_KHR) {
        RecreateSwapchain();
        return;
    }

    frameSync.WaitForSwapchainImageFence(device(), imageIndex);
    frameSync.TrackSwapchainImageInFlight(imageIndex);

  frameSync.ResetCurrentFrameFence(device());

    // record command buffer ...

    frameSync.SubmitFrame(graphicsQueue(), commandBuffers[imageIndex], imageIndex);

    const VkResult presentResult =
            frameSync.PresentFrame(presentQueue(), presentSwapchain().khr, imageIndex);

    if (presentResult == VK_ERROR_OUT_OF_DATE_KHR || presentResult == VK_SUBOPTIMAL_KHR) {
        RecreateSwapchain();
    }

    frameSync.AdvanceFrame();
}
```

### Step 4 — Submit and present helpers

```cpp
// VulkanFrameSync.cpp — SubmitFrame (lines 127–146)
void VulkanFrameSync::SubmitFrame(
        VkQueue graphicsQueue,
        VkCommandBuffer commandBuffer,
        std::uint32_t imageIndex) const {
    VkSemaphore waitSemaphores[] = {imageAvailableSemaphores[currentFrame]};
    constexpr VkPipelineStageFlags waitStages[] = {VK_PIPELINE_STAGE_ALL_GRAPHICS_BIT};

    VkSubmitInfo submitInfo{};
    submitInfo.waitSemaphoreCount = 1;
    submitInfo.pWaitSemaphores = waitSemaphores;
    submitInfo.pWaitDstStageMask = waitStages;
    submitInfo.commandBufferCount = 1;
    submitInfo.pCommandBuffers = &commandBuffer;
    submitInfo.signalSemaphoreCount = 1;
    submitInfo.pSignalSemaphores = &renderFinishedSemaphores[imageIndex];

    vkQueueSubmit(graphicsQueue, 1, &submitInfo, inFlightFences[currentFrame]);
}

// PresentFrame (lines 153–165)
VkResult VulkanFrameSync::PresentFrame(
        VkQueue presentQueue,
        VkSwapchainKHR swapchain,
        std::uint32_t imageIndex) const {
    VkPresentInfoKHR presentInfo{};
    presentInfo.waitSemaphoreCount = 1;
    presentInfo.pWaitSemaphores = &renderFinishedSemaphores[imageIndex];
    presentInfo.swapchainCount = 1;
    presentInfo.pSwapchains = &swapchain;
    presentInfo.pImageIndices = &imageIndex;
    return vkQueuePresentKHR(presentQueue, &presentInfo);
}
```

### Step 5 — Part I milestone check

Run SparkDemo. Resize, minimize, restore. No validation sync errors in console means your skeleton matches Spark's. Part II will add draw calls inside `RecordSceneCommandBuffer` without changing this outer sync.

## 5. How This Connects to the Rest of Your Engine

Part **II** records draw commands between reset and submit. Part **IV** writes uniform buffers after `WaitForCurrentFrameFence`. Part **XVI** may add triple buffering and async compute queues — same fence/semaphore vocabulary.

Memorize the acquire/submit/present diagram; it appears on every Vulkan engine interview.

## 6. Pitfalls and Debugging

- **Fence not created signaled** — first frame deadlocks in `WaitForFences`.
- **Resetting fence before wait** — CPU races GPU; always wait then reset.
- **Present without waiting on renderFinished** — flicker or `VUID-VkPresentInfoKHR-pImageIndices-01430`.
- **Reusing UBO before fence signals** — classic transform flicker; tie buffer index to `currentFrame`.
- **Ignoring OUT_OF_DATE** — infinite failed acquire loop; call `RecreateSwapchain`.

## 7. Exercises

1. Count fences, semaphores, and command buffers in a running Spark build — multiply by frames in flight.
2. Temporarily remove present wait semaphore (revert after!) — describe what you see.
3. Compare FPS with FIFO vs MAILBOX if your GPU reports both present modes.
4. Draw a timeline diagram for two frames in flight with three swapchain images.

## Summary

Frame synchronization connects CPU recording to GPU execution and display: fences gate per-flight data, semaphores chain acquire → render → present. Spark's `VulkanFrameSync` implements the pattern `DrawFrame` uses every frame — **Part I complete** when this runs stable with a clearing or empty pass.

---

Next: [Part II — Basic Rendering Pipeline](../part-02-basic-pipeline/index.md).
