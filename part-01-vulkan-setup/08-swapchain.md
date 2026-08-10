# Swapchain and Presentation

> **Build-Along Milestone:** You understand swapchain image count, surface format choice, present modes, and how Spark creates image views.

**You will learn:**

- Query swapchain capabilities for a surface.
- Choose format, present mode, and extent.
- Create `VkSwapchainKHR` and `VkImageView` for each swapchain image.

## 1. The Problem We Are Solving

The GPU renders into **off-screen images**. The **swapchain** is a queue of images the presentation engine shows on screen — typically double or triple buffered. You acquire an image, render into it, then present it.

Swapchain creation is where window pixel size, display refresh rate, and GPU capabilities meet. Get extent wrong and the image stretches; pick `FIFO` vs `MAILBOX` and you trade latency for tearing. This chapter explains Spark's choices in `VulkanDeviceContext::CreateSwapchainImages`.

## 2. Core Concepts

### Swapchain support query

For each physical device + surface, ask:

| Query | Result used for |
|-------|-----------------|
| `vkGetPhysicalDeviceSurfaceCapabilitiesKHR` | min/max image count, extent, transforms |
| `vkGetPhysicalDeviceSurfaceFormatsKHR` | pixel format (e.g. `B8G8R8A8_UNORM`) |
| `vkGetPhysicalDevicePresentModesKHR` | `FIFO`, `MAILBOX`, `IMMEDIATE` |

Spark wraps these in `VulkanGpuSwapchain::QuerySwapchainSupport`.

### Image count

Use at least `minImageCount + 1` (often triple buffering), clamped to `maxImageCount` if the driver sets a maximum:

```cpp
uint32_t imageCount = capabilities.minImageCount + 1;
```

More images smooth frame pacing at the cost of VRAM.

### Surface format

Prefer `VK_FORMAT_B8G8R8A8_UNORM` with `VK_COLOR_SPACE_SRGB_NONLINEAR_KHR` when available — matches common monitors. HDR formats come in Part X.

### Present mode

| Mode | Behavior | Typical use |
|------|----------|-------------|
| `VK_PRESENT_MODE_FIFO_KHR` | VSync on — wait for vertical blank | Default, no tearing |
| `VK_PRESENT_MODE_MAILBOX_KHR` | Triple buffer, low latency if available | Games when supported |
| `VK_PRESENT_MODE_IMMEDIATE_KHR` | May tear | Benchmarks only |

Spark's `ChooseSwapPresentMode` prefers `MAILBOX` when available, else `FIFO`.

### Extent

`swapchainExtent` should match current framebuffer size, clamped to `minImageExtent` / `maxImageExtent`. Special case: when `currentExtent` is not `0xFFFFFFFF`, use it directly (some platforms fix extent).

### Image views

Swapchain gives you `VkImage` handles — not directly bindable as framebuffer attachments until you create **image views** (`VkImageView`) describing color aspect and mip levels.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### VulkanPresentationSwapchain struct

**Concept:** Swapchain support query

**Decision:** Store `khr`, `images`, `imageViews`, `imageFormat`, `extent` together

**Why:** Framebuffer creation (Part II) needs format + views + extent as a bundle; one struct avoids mismatched state.

**In Spark:** `include/spark/render/present/VulkanPresentationSwapchain.hpp`.

### TRANSFER_SRC on swapchain images

**Concept:** Image count / usage

**Decision:** `imageUsage` includes `COLOR_ATTACHMENT | TRANSFER_SRC`

**Why:** Screenshot and video capture read back from swapchain without an extra copy target (see `VulkanRenderer` capture paths).

**In Spark:** `CreateSwapchainImages` — `swapchainCreateInfo.imageUsage`.

### CONCURRENT sharing when queue families differ

**Concept:** Present mode / queues

**Decision:** `VK_SHARING_MODE_CONCURRENT` with both family indices when graphics ≠ present

**Why:** Swapchain images must be readable by both queues without explicit ownership transfers every frame.

**In Spark:** `CreateSwapchainImages` — sharing mode branch.

### RecreateSwapchain waits for idle

**Concept:** Extent changes on resize

**Decision:** `WaitDeviceIdle` before destroy/create on resize

**Why:** In-flight frames may still reference old swapchain images; idle is blunt but correct for learning.

**In Spark:** `VulkanDeviceContext::RecreateSwapchain`.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book; Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `src/spark/render/core/VulkanDeviceContext.cpp` — `CreateSwapchainImages`, `RecreateSwapchain`
- `include/spark/render/present/VulkanPresentationSwapchain.hpp`
- `src/spark/render/gpu/VulkanGpuSwapchain.cpp` — format/mode/extent helpers

### Step 1 — Query and choose parameters

```cpp
// VulkanDeviceContext.cpp (lines 229–235)
void VulkanDeviceContext::CreateSwapchainImages() {
    const VulkanRendererGpu::SwapchainSupportDetails swapchainSupport =
            VulkanRendererGpu::QuerySwapchainSupport(physicalDevice, surface);
    const VkSurfaceFormatKHR surfaceFormat = VulkanRendererGpu::ChooseSwapSurfaceFormat(swapchainSupport.formats);
    const VkPresentModeKHR presentMode = VulkanRendererGpu::ChooseSwapPresentMode(swapchainSupport.presentModes);
    const VkExtent2D extent =
            VulkanRendererGpu::ChooseSwapExtent(swapchainSupport.capabilities, boundWindow->Handle());
```

Log chosen format, mode, and extent once — invaluable when moving to a new monitor or VM.

### Step 2 — vkCreateSwapchainKHR

```cpp
    std::uint32_t imageCount = swapchainSupport.capabilities.minImageCount + 1;
    if (swapchainSupport.capabilities.maxImageCount > 0 && imageCount > swapchainSupport.capabilities.maxImageCount) {
        imageCount = swapchainSupport.capabilities.maxImageCount;
    }

    VkSwapchainCreateInfoKHR swapchainCreateInfo{};
    swapchainCreateInfo.sType = VK_STRUCTURE_TYPE_SWAPCHAIN_CREATE_INFO_KHR;
    swapchainCreateInfo.surface = surface;
    swapchainCreateInfo.minImageCount = imageCount;
    swapchainCreateInfo.imageFormat = surfaceFormat.format;
    swapchainCreateInfo.imageColorSpace = surfaceFormat.colorSpace;
    swapchainCreateInfo.imageExtent = extent;
    swapchainCreateInfo.imageArrayLayers = 1;
    swapchainCreateInfo.imageUsage = VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT | VK_IMAGE_USAGE_TRANSFER_SRC_BIT;
    // sharing mode, preTransform, compositeAlpha, presentMode, clipped ...
    vkCreateSwapchainKHR(device, &swapchainCreateInfo, nullptr, &swapchain.khr);
```

### Step 3 — Get images and create views

```cpp
    vkGetSwapchainImagesKHR(device, swapchain.khr, &imageCount, nullptr);
    swapchain.images.Resize(static_cast<std::size_t>(imageCount));
    vkGetSwapchainImagesKHR(device, swapchain.khr, &imageCount, swapchain.images.GetData());

    swapchain.imageFormat = surfaceFormat.format;
    swapchain.extent = extent;

    for (std::size_t i = 0; i < swapchain.images.GetSize(); ++i) {
        VkImageViewCreateInfo imageViewCreateInfo{};
        imageViewCreateInfo.sType = VK_STRUCTURE_TYPE_IMAGE_VIEW_CREATE_INFO;
        imageViewCreateInfo.image = swapchain.images[i];
        imageViewCreateInfo.viewType = VK_IMAGE_VIEW_TYPE_2D;
        imageViewCreateInfo.format = swapchain.imageFormat;
        imageViewCreateInfo.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
        // mip/layer counts = 1
        vkCreateImageView(device, &imageViewCreateInfo, nullptr, &swapchain.imageViews[i]);
    }
```

Part II attaches these views to framebuffers for render passes.

### Step 4 — Recreate on resize

```cpp
void VulkanDeviceContext::RecreateSwapchain() {
    int width = 0, height = 0;
    boundWindow->GetFramebufferSize(width, height);
    while (width == 0 || height == 0) {
        boundWindow->PollEvents();
        boundWindow->GetFramebufferSize(width, height);
    }
    WaitDeviceIdle();
    DestroySwapchain();
    CreateSwapchainImages();
}
```

Minimized windows report zero extent — spin until restored, as Vulkan cannot create zero-sized swapchains.

### Step 5 — Test resize

Run SparkDemo, drag window corners, minimize/restore. Swapchain recreation should not crash; brief flicker is acceptable in Part I.

## 5. How This Connects to the Rest of Your Engine

Chapter **9** places swapchain inside `VulkanDeviceContext` as the central GPU owner. Chapter **10** acquires swapchain images each frame. Part **II** creates **framebuffers** pairing swapchain views with a depth image.

Swapchain format and extent flow into projection matrices in Part V — keep them queryable from one place.

## 6. Pitfalls and Debugging

- **`OUT_OF_DATE` on acquire** — extent changed; recreate swapchain (Chapter 10).
- **Wrong extent (window vs framebuffer)** — blurry output on Retina; use `glfwGetFramebufferSize`.
- **Old swapchain not passed** — on recreate, set `oldSwapchain` for proper handoff (Spark sets null after idle destroy — acceptable when fully idle).
- **IMMEDIATE present everywhere** — tearing in production; prefer FIFO or MAILBOX.

## 7. Exercises

1. Log present mode at startup — is MAILBOX available on your GPU?
2. Resize to non-standard aspect ratio — does extent match framebuffer pixels?
3. Count swapchain images — is it min+1?
4. Read `ChooseSwapExtent` in Spark — when does it use `currentExtent` directly?

## Summary

The swapchain connects Vulkan rendering to the display: query support, choose format/mode/extent, create images and views, recreate on resize. Spark encapsulates storage in `VulkanPresentationSwapchain` and builds it from `VulkanDeviceContext` after logical device creation.

---

Next: [RenderDevice — Central GPU Owner](09-device-context.md).
