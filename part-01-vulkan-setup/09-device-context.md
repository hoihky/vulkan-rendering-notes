# RenderDevice — Central GPU Owner

> **Build-Along Milestone:** You can list everything `VulkanDeviceContext` owns and explain why `VulkanRenderer` delegates device lifecycle to it.

**You will learn:**

- Centralize instance, surface, device, queues, and swapchain in one class.
- Separate device lifecycle from render-pass orchestration.
- Call `RecreateSwapchain` safely after resize or `OUT_OF_DATE`.

## 1. The Problem We Are Solving

By now you have seen instance creation, device selection, and swapchain setup as separate steps. In a growing engine, those handles scatter across files and recreate logic duplicates. Spark introduces **`VulkanDeviceContext`** as the **single owner** of GPU device lifetime resources.

`VulkanRenderer` then focuses on passes, command buffers, and scene drawing — it **uses** the device context but does not recreate instances on every resize. This chapter teaches that split so your mini-engine stays maintainable through Part XVI.

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
