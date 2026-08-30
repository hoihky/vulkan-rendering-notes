# RenderDevice — Central GPU Owner

> **Build-Along Milestone:** You can list everything `VulkanDeviceContext` owns and explain why `VulkanRenderer` delegates device lifecycle to it.

**You will learn:**

- Centralize instance, surface, device, queues, and swapchain in one class.
- Separate device lifecycle from render-pass orchestration.
- Call `RecreateSwapchain` safely after resize or `OUT_OF_DATE`.

## 1. The Problem We Are Solving

By now you have seen instance creation, device selection, and swapchain setup as separate steps. In a growing engine, those handles scatter across files and recreate logic duplicates. Spark introduces **`VulkanDeviceContext`** as the **single owner** of GPU device lifetime resources.

`VulkanRenderer` then focuses on passes, command buffers, and scene drawing — it **uses** the device context but does not recreate instances on every resize. This chapter teaches that split so your mini-engine stays maintainable through Part XVI.

### Why split DeviceContext from Renderer?

Beginners often put **everything** in one giant `VulkanRenderer` class. That works for tutorials but collapses when:

- Window resizes require recreating swapchain **without** tearing down shadow atlases.
- Unit tests want a headless device without a full renderer.
- You add a second window or offline capture path.

Spark's split:

```text
VulkanDeviceContext  — "I own the GPU connection"
  ├── instance, surface, physical/logical device
  ├── queues
  └── swapchain + image views

VulkanRenderer       — "I own how we draw each frame"
  ├── render passes, pipelines, descriptors
  ├── command buffer recording
  └── scene / post / UI passes
```

**Rule of thumb:** if it survives window resize unchanged, it probably belongs in renderer persistent resources or a dedicated pool — not in DeviceContext. If it **must** match swapchain extent, it recreates in `RecreateSwapchain`.

### Constructor order in Spark (full Part I recap)

```cpp
// VulkanDeviceContext constructor (conceptual)
CreateInstance();
CreateDebugMessenger();
CreateSurface(window);
SelectPhysicalDevice();
CreateLogicalDevice();
// swapchain deferred until renderer calls RecreateSwapchain

// VulkanRenderer constructor
CreateCommandPool();
CreatePersistentSceneResources();
RecreateSwapchain();  // calls deviceContext.RecreateSwapchain() inside
frameSync.Create();
```

When the user resizes the window, only `RecreateSwapchain` and dependents rebuild — not the entire instance or logical device.

## 1b. Concept Deep Dive — DeviceContext as the GPU Service Boundary

### Splitting bootstrap from frame orchestration

By Chapter 8 you have seen instance, device, and swapchain creation as sequential steps scattered across walkthrough sections. Production engines consolidate those handles into one **device service** so recreate logic, getter access, and destroy order live in a single place. Spark's `VulkanDeviceContext` answers three questions every pass needs: *which `VkDevice`*, *which queues*, and *which swapchain images exist right now*.

`VulkanRenderer` owns how frames are drawn — render passes, pipelines, command pools, scene passes — but delegates lifecycle to `deviceContext`. When the window resizes, `RecreateSwapchain` waits for idle, destroys image views and the old `VkSwapchainKHR`, re-queries framebuffer extent (spinning on zero size while minimized), and rebuilds images. The logical device and instance survive; shadow atlases and shader modules stay loaded. That scope distinction is the main reason the split exists.

`TryGetDrawableSize` exposes swapchain extent to UI and camera code through `EngineContext`, falling back to GLFW framebuffer size when the swapchain is not ready yet — one pixel-size source for projection matrices and hit tests.

### Ownership split

```text
VulkanDeviceContext                    VulkanRenderer
─────────────────────                  ────────────────
VkInstance, VkSurfaceKHR               Command pool + buffers
VkPhysicalDevice, VkDevice             Render passes + pipelines
VkQueue (graphics, present)            Scene / shadow / post passes
VulkanPresentationSwapchain            VulkanFrameSync
                                       pendingScene snapshot
```

| Resource | Survives swapchain recreate? | Owner |
|----------|------------------------------|-------|
| `VkDevice` | Yes | `VulkanDeviceContext` |
| Swapchain image views | No — rebuilt | `VulkanPresentationSwapchain` |
| `VkRenderPass` (scene) | Yes (until format change) | `VulkanRenderer` |
| Depth buffer | No — matches new extent | `VulkanRenderer` |

### Constructor — bootstrap without swapchain yet

Swapchain creation waits until the renderer has render-pass format compatibility; the constructor stops after logical device:

```cpp
// src/spark/render/core/VulkanDeviceContext.cpp (lines 10–22)
VulkanDeviceContext::VulkanDeviceContext(Window& window) : boundWindow(&window) {
    if constexpr (VulkanRendererGpu::kEnableValidationLayers) {
        if (!VulkanRendererGpu::CheckValidationLayerSupport()) {
            throw std::runtime_error("Vulkan validation layers requested but not available");
        }
    }

    CreateInstance();
    CreateDebugMessenger();
    CreateSurface();
    SelectPhysicalDevice();
    CreateLogicalDevice();
}
```

### Recreate — idle, destroy, rebuild images

```cpp
// src/spark/render/core/VulkanDeviceContext.cpp (lines 57–72)
void VulkanDeviceContext::RecreateSwapchain() {
    if (boundWindow == nullptr || device == VK_NULL_HANDLE) {
        return;
    }

    int width = 0;
    int height = 0;
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

### Renderer constructs context first, then frame resources

```cpp
// src/spark/render/core/VulkanRenderer.cpp (lines 43–49)
VulkanRenderer::VulkanRenderer(Window& inAppWindow) : deviceContext(inAppWindow), boundWindow(&inAppWindow) {
    shaderLoader.SetDevice(device());

    CreateCommandPool();
    CreatePersistentSceneResources();
    RecreateSwapchain();
    frameSync.Create(device());
```

Pass code calls `device()` and `presentSwapchain()` forwarded from `deviceContext` — never duplicate `VkDevice` storage in pass classes.

### Drawable size for UI and cameras

```cpp
// src/spark/render/core/VulkanDeviceContext.cpp (lines 75–79)
bool VulkanDeviceContext::TryGetDrawableSize(int& outWidth, int& outHeight) const noexcept {
    if (swapchain.khr == VK_NULL_HANDLE || swapchain.extent.width == 0 || swapchain.extent.height == 0) {
        return false;
    }
    outWidth = static_cast<int>(swapchain.extent.width);
```

`EngineContext::GetFramebufferSize` tries the presenter first, then falls back to `Window::GetFramebufferSize`.

## 2. Core Concepts

### What DeviceContext owns

| Resource | Created in |
|----------|------------|
| `VkInstance` | `CreateInstance` |
| `VkDebugUtilsMessengerEXT` | `CreateDebugMessenger` |
| `VkSurfaceKHR` | `CreateSurface` |
| `VkPhysicalDevice` | `SelectPhysicalDevice` |
| `VkDevice` | `CreateLogicalDevice` |
| `VkQueue` graphics + present | `CreateLogicalDevice` |
| `VulkanPresentationSwapchain` | `CreateSwapchainImages` |

Public getters expose handles to renderer subsystems without making them global.

### What DeviceContext does not own

Render passes, pipelines, descriptor pools, and per-frame command buffers live on **`VulkanRenderer`** (or dedicated pass classes). Device context answers: *which GPU, which queues, which swapchain images exist right now*.

### Construction order

Constructor runs the full chain from Chapter 2's graph:

```text
CreateInstance → CreateDebugMessenger → CreateSurface
    → SelectPhysicalDevice → CreateLogicalDevice
```

Swapchain is created when the renderer finishes render-pass setup (needs compatible formats), but recreation always goes through `RecreateSwapchain()`.

### TryGetDrawableSize

UI and cameras need pixel dimensions matching the swapchain:

```cpp
bool TryGetDrawableSize(int& outWidth, int& outHeight) const;
```

Returns swapchain extent when valid; `EngineContext` falls back to GLFW framebuffer size when swapchain not ready.

### Idle before destroy

`WaitDeviceIdle` before destroying swapchain or device ensures no submitted work references images being freed. Heavy-handed but correct — production engines use fences per frame (Chapter 10) instead of global idle except on resize/shutdown.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Separate class from VulkanRenderer

**Concept:** What DeviceContext owns

**Decision:** `VulkanDeviceContext` is a member, not a base class of renderer

**Why:** Tests can mock device context; renderer code paths stay readable; device recreate does not recompile shader modules.

**In Spark:** `VulkanRenderer` holds `VulkanDeviceContext deviceContext`.

### Window pointer for recreate

**Concept:** RecreateSwapchain

**Decision:** Store `Window* boundWindow` for extent queries and event poll during zero-size wait

**Why:** Recreate needs live framebuffer size without pulling GLFW into every pass class.

**In Spark:** `VulkanDeviceContext::RecreateSwapchain` — polls window while minimized.

### Swapchain as sub-object

**Concept:** Swapchain ownership

**Decision:** `VulkanPresentationSwapchain swapchain` member with `Destroy()` method

**Why:** Image views must be destroyed before swapchain; bundling prevents orphan views.

**In Spark:** `swapchain.Destroy(device)` in `DestroySwapchain`.

### Getters return const references for swapchain

**Concept:** API surface

**Decision:** `GetSwapchain()` const/non-const overloads

**Why:** Framebuffer setup reads format/extent/views; const-correctness catches accidental mutation during record.

**In Spark:** `VulkanDeviceContext.hpp`.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book; Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `include/spark/render/core/VulkanDeviceContext.hpp`
- `src/spark/render/core/VulkanDeviceContext.cpp`
- `include/spark/render/core/VulkanRenderer.hpp`

### Step 1 — Read the public API

```cpp
// include/spark/render/core/VulkanDeviceContext.hpp (abbreviated)
class VulkanDeviceContext {
public:
    explicit VulkanDeviceContext(Window& window);
    ~VulkanDeviceContext();

    [[nodiscard]] VkInstance GetInstance() const noexcept;
    [[nodiscard]] VkPhysicalDevice GetPhysicalDevice() const noexcept;
    [[nodiscard]] VkDevice GetDevice() const noexcept;
    [[nodiscard]] VkSurfaceKHR GetSurface() const noexcept;
    [[nodiscard]] VkQueue GetGraphicsQueue() const noexcept;
    [[nodiscard]] VkQueue GetPresentQueue() const noexcept;

    [[nodiscard]] VulkanPresentationSwapchain& GetSwapchain() noexcept;
    [[nodiscard]] const VulkanPresentationSwapchain& GetSwapchain() const noexcept;

    void WaitDeviceIdle() const;
    void DestroySwapchain() noexcept;
    void RecreateSwapchain();
    [[nodiscard]] bool TryGetDrawableSize(int& outWidth, int& outHeight) const noexcept;
};
```

**Build-along:** Sketch the same interface for your engine — name it `RenderDevice` or `GpuContext` if you prefer.

### Step 2 — Constructor chain

```cpp
// VulkanDeviceContext.cpp (lines 10–22)
VulkanDeviceContext::VulkanDeviceContext(Window& window) : boundWindow(&window) {
    if constexpr (VulkanRendererGpu::kEnableValidationLayers) {
        if (!VulkanRendererGpu::CheckValidationLayerSupport()) {
            throw std::runtime_error("Vulkan validation layers requested but not available");
        }
    }

    CreateInstance();
    CreateDebugMessenger();
    CreateSurface();
    SelectPhysicalDevice();
    CreateLogicalDevice();
}
```

No swapchain yet — renderer may need render pass format first.

### Step 3 — Destructor order

```cpp
VulkanDeviceContext::~VulkanDeviceContext() {
    if (device == VK_NULL_HANDLE) {
        return;
    }
    WaitDeviceIdle();
    DestroySwapchain();
    vkDestroyDevice(device, nullptr);
    // debug messenger, surface, instance ...
}
```

Matches reverse of creation — memorize for interviews and debugging leaks.

### Step 4 — How renderer uses context

`VulkanRenderer` forwards common queries:

```cpp
VkDevice device() const { return deviceContext.GetDevice(); }
VkQueue graphicsQueue() const { return deviceContext.GetGraphicsQueue(); }
const VulkanPresentationSwapchain& presentSwapchain() const { return deviceContext.GetSwapchain(); }
```

Pass code never stores duplicate `VkDevice` handles — single source of truth.

### Step 5 — Recreate entry point

When `DrawFrame` sees resize or `OUT_OF_DATE`, it calls `RecreateSwapchain()` on the context, which re-runs `CreateSwapchainImages` after idle destroy. Framebuffers and depth images (Part II) must rebuild afterward — `VulkanRenderer::RecreateSwapchain` orchestrates that outer layer.

## 5. How This Connects to the Rest of Your Engine

Chapter **10** uses device, queues, and swapchain from this class for acquire/present. Part **II** creates framebuffers from `GetSwapchain().imageViews`. Part **XVI** discusses finer-grained sync than `WaitDeviceIdle` on every resize.

If you add a second window later, you typically add a second surface and swapchain — still owned by a device context abstraction.

## 6. Pitfalls and Debugging

- **Two classes destroying the same `VkDevice`** — only `VulkanDeviceContext` destructor calls `vkDestroyDevice`.
- **Recreating swapchain without rebuilding framebuffers** — validation error on next render pass; renderer must hook recreate.
- **Calling GPU work after destructor started** — shutdown order: stop loop → `OnDetach` → destroy renderer → window last.
- **Leaking image views** — always `DestroySwapchain` before device destroy.

## 7. Exercises

1. Grep `deviceContext.` in `VulkanRenderer.cpp` — how many call sites?
2. Trace `RecreateSwapchain` from resize callback to new image views — list every function on the path.
3. When is `TryGetDrawableSize` false — what does UI code do instead?
4. Draw a UML-style box diagram: DeviceContext vs VulkanRenderer responsibilities.

## Summary

`VulkanDeviceContext` centralizes Vulkan bootstrap and swapchain storage. `VulkanRenderer` consumes it for drawing. Keeping device lifecycle in one class prevents recreate bugs and mirrors how production engines separate "GPU device service" from "frame renderer."

---

Next: [Frame Synchronization](10-frame-sync.md).
