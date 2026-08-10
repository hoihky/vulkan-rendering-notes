# Window Creation with GLFW

> **Build-Along Milestone:** You understand why Spark uses `GLFW_NO_API`, how surface extensions are queried, and how resize callbacks reach the renderer.

**You will learn:**

- Create a Vulkan-capable window without an OpenGL context.
- Query required instance extensions from GLFW.
- Create a `VkSurfaceKHR` tied to the native window handle.

## 1. The Problem We Are Solving

Vulkan does not create windows — it renders **to surfaces** provided by the OS. GLFW is Spark's cross-platform bridge: it opens a native window, forwards input, and exposes `glfwCreateWindowSurface` after you create a `VkInstance`.

This chapter covers the **platform layer** only. No `VkDevice` yet. The milestone is a window that can later host a swapchain, with correct hints and extension queries so Chapter 6's instance creation succeeds.

## 2. Core Concepts

### GLFW_NO_API

OpenGL and Vulkan compete for the same window unless you opt out of GLFW's default OpenGL context:

```cpp
glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
```

Without this hint, `glfwCreateWindow` may create an OpenGL context you never use, wasting resources and sometimes conflicting with MoltenVK on macOS.

### Framebuffer vs window size

On Retina displays, **window size** (logical points) differs from **framebuffer size** (pixels). Vulkan swapchain extent uses framebuffer pixels:

```cpp
glfwGetFramebufferSize(window, &width, &height);
```

Spark's `Window::GetFramebufferSize` wraps this. UI hit-testing and projection matrices must use the same pixel dimensions as the swapchain.

### Required instance extensions

GLFW tells you which instance extensions are mandatory for surface creation on the current platform — typically `VK_KHR_surface` plus a platform-specific extension (`VK_KHR_win32_surface`, `VK_MVK_macos_surface`, `VK_KHR_xcb_surface`, etc.):

```cpp
uint32_t count;
const char** ext = glfwGetRequiredInstanceExtensions(&count);
```

Spark merges these with optional debug extensions in `VulkanGpuInstance::GetRequiredInstanceExtensions`.

### User pointer pattern

GLFW callbacks are C function pointers. Spark stores `this` on the window:

```cpp
glfwSetWindowUserPointer(glfwWindow, this);
// static callback → cast user pointer → member function
```

This keeps resize, cursor, and scroll handling in `Window` methods instead of globals.

### Surface creation

After `vkCreateInstance`, you call:

```cpp
glfwCreateWindowSurface(instance, glfwWindow, nullptr, &surface);
```

The surface ties the Vulkan instance to the drawable region. It must be destroyed before the instance (`vkDestroySurfaceKHR` before `vkDestroyInstance`).

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Window class owns GLFW lifetime

**Concept:** GLFW_NO_API

**Decision:** `Window` constructor calls `glfwInit`; destructor destroys window and `glfwTerminate`

**Why:** Single owner prevents double-init and use-after-terminate when multiple subsystems might otherwise call GLFW.

**In Spark:** `src/spark/render/platform/Window.cpp` — constructor/destructor.

### Default 1280×720 resizable window

**Concept:** Framebuffer vs window size

**Decision:** `DefaultWidth` / `DefaultHeight` constants; `GLFW_RESIZABLE` true

**Why:** Large enough for demos; resize exercises swapchain recreation path from Chapter 4.

**In Spark:** `include/spark/render/platform/Window.hpp` — `DefaultWidth = 1280`.

### RequiredVulkanInstanceExtensions on Window

**Concept:** Required instance extensions

**Decision:** Method on `Window` wraps `glfwGetRequiredInstanceExtensions`

**Why:** Instance creation needs the window's platform extensions; colocating the query on `Window` keeps platform knowledge in the platform module.

**In Spark:** `Window::RequiredVulkanInstanceExtensions()` in `Window.cpp`.

### Resize callback indirection

**Concept:** User pointer pattern

**Decision:** `SetFramebufferResizeCallback` stores `std::function`; GLFW trampoline calls `HandleFramebufferResize`

**Why:** `Engine` registers `NotifySwapchainResize` without GLFW types in `Engine.hpp`.

**In Spark:** `Window::SetFramebufferResizeCallback` + `Engine::Impl` constructor.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book; Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `include/spark/render/platform/Window.hpp`
- `src/spark/render/platform/Window.cpp`
- `src/spark/render/gpu/VulkanGpuInstance.cpp`

### Step 1 — Constructor hints and window creation

```cpp
// src/spark/render/platform/Window.cpp (lines 11–34)
Window::Window() {
    if (!glfwInit()) {
        throw std::runtime_error("glfwInit failed");
    }
    glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
    glfwWindowHint(GLFW_RESIZABLE, GLFW_TRUE);
#if defined(__APPLE__)
    glfwWindowHint(GLFW_COCOA_RETINA_FRAMEBUFFER, GLFW_TRUE);
#endif

    glfwWindow = glfwCreateWindow(DefaultWidth, DefaultHeight, "Spark", nullptr, nullptr);
    if (glfwWindow == nullptr) {
        glfwTerminate();
        throw std::runtime_error("glfwCreateWindow failed");
    }
    glfwSetWindowUserPointer(glfwWindow, this);
    glfwSetFramebufferSizeCallback(
            glfwWindow,
            +[](GLFWwindow* w, int width, int height) {
                auto* self = static_cast<Window*>(glfwGetWindowUserPointer(w));
                if (self != nullptr) {
                    self->HandleFramebufferResize(width, height);
                }
            });
}
```

**Build-along:** Compare with your engine — if you use SDL or Win32 native, map the same three responsibilities: no GL context, pixel size query, resize callback.

### Step 2 — Poll and close queries

```cpp
void Window::PollEvents() const { glfwPollEvents(); }

bool Window::ShouldClose() const { return glfwWindowShouldClose(glfwWindow) != 0; }
```

The main loop from Chapter 4 calls these every iteration.

### Step 3 — Extension query

```cpp
// src/spark/render/platform/Window.cpp (lines 95–107)
Array<const char*> Window::RequiredVulkanInstanceExtensions() const {
    std::uint32_t count = 0;
    const char** names = glfwGetRequiredInstanceExtensions(&count);
    if (names == nullptr || count == 0) {
        throw std::runtime_error("glfwGetRequiredInstanceExtensions returned no extensions");
    }
    Array<const char*> out;
    out.Reserve(static_cast<std::size_t>(count));
    for (std::uint32_t i = 0; i < count; ++i) {
        out.PushBack(names[i]);
    }
    return out;
}
```

Log `count` and extension names once during development — saves hours when instance creation fails on a new OS.

### Step 4 — Surface creation

```cpp
// src/spark/render/platform/Window.cpp (lines 109–115)
void Window::CreateVulkanSurface(void* instancePtr, void* outSurface) const {
    const auto vkInstance = static_cast<VkInstance>(instancePtr);
    auto* surfaceOut = static_cast<VkSurfaceKHR*>(outSurface);
    if (glfwCreateWindowSurface(vkInstance, glfwWindow, nullptr, surfaceOut) != VK_SUCCESS) {
        throw std::runtime_error("glfwCreateWindowSurface failed");
    }
}
```

`VulkanDeviceContext::CreateSurface` calls this immediately after instance creation (Chapter 9).

### Step 5 — Verify on your machine

Run SparkDemo, resize the window, minimize and restore. No crash means callbacks and `PollEvents` cooperate with the deferred recreate path.

## 5. How This Connects to the Rest of Your Engine

Chapter **6** passes `GetRequiredInstanceExtensions(window)` into `vkCreateInstance`. Chapter **8** uses framebuffer size for swapchain extent. Chapter **5** in Part V wires `GlfwInput` to the same `GLFWwindow*`.

The window is the root of the initialization graph from Chapter 2.

## 6. Pitfalls and Debugging

- **Forgetting GLFW_NO_API** — silent OpenGL context creation; weird failures on macOS.
- **Using window size for swapchain** — blurry or clipped rendering on Retina.
- **Calling `glfwTerminate` while instance alive** — undefined behavior; destroy Vulkan surface/device first.
- **Global resize callback** — use user pointer; multiple windows need per-window state.

## 7. Exercises

1. Print required extensions on your OS — how many, and which platform surface extension?
2. Maximize SparkDemo — log framebuffer size before and after.
3. Sketch the destroy order: window, surface, instance, GLFW terminate.
4. Read `VulkanGpuInstance::GetRequiredInstanceExtensions` — what does it add beyond GLFW's list?

## Summary

Spark's `Window` wraps GLFW with Vulkan-friendly hints, extension queries, surface creation, and resize callbacks. Master this file before instance creation — every Vulkan object downstream assumes a valid surface tied to a live native window.

---

Next: [Vulkan Instance and Validation Layers](06-vulkan-instance.md).
