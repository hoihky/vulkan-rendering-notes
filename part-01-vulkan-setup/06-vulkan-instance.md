# Vulkan Instance and Validation Layers

> **Build-Along Milestone:** You can explain what `VkInstance` does, which extensions Spark enables, and how validation messages reach your console.

**You will learn:**

- Create a `VkInstance` with application info and required extensions.
- Enable Khronos validation layers in debug builds.
- Install a debug messenger to print validation errors.

## 1. The Problem We Are Solving

`VkInstance` is the **root handle** of every Vulkan program. It loads function pointers, enables extensions and layers, and outlives individual devices. Create it incorrectly — missing surface extension, wrong API version — and nothing downstream works.

This chapter covers instance creation only. You still will not draw geometry. The milestone is a valid instance plus (in debug) a debug messenger that prints helpful errors when you make mistakes in later chapters.

## 2. Core Concepts

### What VkInstance represents

| Responsibility | Details |
|----------------|---------|
| Entry point | `vkCreateInstance`, global extensions/layers |
| No GPU yet | Physical devices enumerated *after* instance |
| Lifetime | Created first; destroyed after all devices and surfaces |

Think of instance as **bootstrapping the API**, not **using the GPU**.

### Application info

`VkApplicationInfo` tells the driver your app and engine names and the Vulkan API version you target:

```cpp
appInfo.apiVersion = VK_API_VERSION_1_3;
```

Drivers may warn if you request features above what they support. Spark targets 1.3 for modern dynamic state and synchronization2 features used later.

### Instance extensions

Minimum for GLFW windowing:

- `VK_KHR_surface`
- Platform surface extension (from GLFW)

Debug builds add `VK_EXT_debug_utils` when validation is enabled — required for the debug messenger callback.

On macOS, Spark also sets:

```cpp
createInfo.flags |= VK_INSTANCE_CREATE_ENUMERATE_PORTABILITY_BIT_KHR;
```

for MoltenVK portability subset compliance.

### Validation layers

The Khronos validation layer (`VK_LAYER_KHRONOS_validation`) intercepts Vulkan calls and reports:

- Lifetime violations (use-after-free of handles)
- Synchronization mistakes (missing barriers)
- Render pass attachment mismatches

Spark enables it when `!NDEBUG`:

```cpp
static constexpr bool kEnableValidationLayers = true;  // debug only
```

Ship release builds without layers — they add CPU overhead.

### Debug messenger

`VkDebugUtilsMessengerEXT` registers a callback receiving message severity, type, and text. Spark routes warnings and errors to `stderr` via `DefaultDebugMessengerCallback`.

Without a messenger, layers may only log to a platform-specific debugger you never see.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### VulkanGpuInstance static helpers

**Concept:** What VkInstance represents

**Decision:** Put layer/extension queries in `VulkanRendererGpu::VulkanGpuInstance` namespace

**Why:** Keeps `VulkanDeviceContext` readable — one line calls `GetRequiredInstanceExtensions` instead of duplicating GLFW merge logic.

**In Spark:** `include/spark/render/gpu/VulkanGpuInstance.hpp`.

### constexpr validation toggle

**Concept:** Validation layers

**Decision:** `kEnableValidationLayers` is compile-time false in Release

**Why:** Zero runtime cost in shipping builds; no `#ifdef` scattered through create calls — `if constexpr` strips layer code.

**In Spark:** `VulkanGpuInstance.hpp` — `#ifndef NDEBUG`.

### CreateDebugMessenger after instance

**Concept:** Debug messenger

**Decision:** Separate `CreateDebugMessenger()` called from `VulkanDeviceContext` constructor

**Why:** Messenger needs valid `VkInstance`; failure to create instance should not leave half-initialized debug state.

**In Spark:** `VulkanDeviceContext.cpp` — after `CreateInstance()`.

### Extensions from Window + debug utils

**Concept:** Instance extensions

**Decision:** `GetRequiredInstanceExtensions(const Window&)` merges GLFW list with debug extension

**Why:** Single function used at create time — impossible to forget surface extension on one platform.

**In Spark:** `src/spark/render/gpu/VulkanGpuInstance.cpp`.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book; Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `include/spark/render/gpu/VulkanGpuInstance.hpp`
- `src/spark/render/gpu/VulkanGpuInstance.cpp`
- `src/spark/render/core/VulkanDeviceContext.cpp`

### Step 1 — Validation layer check

Before creating instance, Spark verifies layers exist (debug only):

```cpp
// VulkanDeviceContext.cpp (lines 10–15)
VulkanDeviceContext::VulkanDeviceContext(Window& window) : boundWindow(&window) {
    if constexpr (VulkanRendererGpu::kEnableValidationLayers) {
        if (!VulkanRendererGpu::CheckValidationLayerSupport()) {
            throw std::runtime_error("Vulkan validation layers requested but not available");
        }
    }
    CreateInstance();
    // ...
```

Install the Vulkan SDK if this throws on a dev machine.

### Step 2 — vkCreateInstance

```cpp
// VulkanDeviceContext.cpp (lines 84–112)
void VulkanDeviceContext::CreateInstance() {
    VkApplicationInfo appInfo{};
    appInfo.sType = VK_STRUCTURE_TYPE_APPLICATION_INFO;
    appInfo.pApplicationName = "Spark";
    appInfo.applicationVersion = VK_MAKE_VERSION(0, 1, 0);
    appInfo.pEngineName = "Spark";
    appInfo.engineVersion = VK_MAKE_VERSION(0, 1, 0);
    appInfo.apiVersion = VK_API_VERSION_1_3;

    const Array<const char*> extensions = VulkanRendererGpu::GetRequiredInstanceExtensions(*boundWindow);

    VkInstanceCreateInfo createInfo{};
    createInfo.sType = VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO;
    createInfo.pApplicationInfo = &appInfo;
    createInfo.enabledExtensionCount = static_cast<std::uint32_t>(extensions.GetSize());
    createInfo.ppEnabledExtensionNames = extensions.GetData();
#ifdef SPARK_PLATFORM_APPLE
    createInfo.flags |= VK_INSTANCE_CREATE_ENUMERATE_PORTABILITY_BIT_KHR;
#endif
    if constexpr (VulkanRendererGpu::kEnableValidationLayers) {
        const char* const validationLayerNames[] = {VulkanRendererGpu::kKhronosValidationLayerName};
        createInfo.enabledLayerCount = 1;
        createInfo.ppEnabledLayerNames = validationLayerNames;
    }

    if (vkCreateInstance(&createInfo, nullptr, &instance) != VK_SUCCESS) {
        throw std::runtime_error("vkCreateInstance failed");
    }
}
```

**Build-along:** Copy the structure into your engine; swap application name strings.

### Step 3 — Debug messenger

```cpp
// VulkanDeviceContext.cpp (lines 115–131)
void VulkanDeviceContext::CreateDebugMessenger() {
    if constexpr (!VulkanRendererGpu::kEnableValidationLayers) {
        return;
    }

    VkDebugUtilsMessengerCreateInfoEXT debugCreateInfo{};
    debugCreateInfo.sType = VK_STRUCTURE_TYPE_DEBUG_UTILS_MESSENGER_CREATE_INFO_EXT;
    debugCreateInfo.messageSeverity = VK_DEBUG_UTILS_MESSAGE_SEVERITY_WARNING_BIT_EXT |
                                      VK_DEBUG_UTILS_MESSAGE_SEVERITY_ERROR_BIT_EXT;
    debugCreateInfo.messageType = VK_DEBUG_UTILS_MESSAGE_TYPE_GENERAL_BIT_EXT |
                                  VK_DEBUG_UTILS_MESSAGE_TYPE_VALIDATION_BIT_EXT |
                                  VK_DEBUG_UTILS_MESSAGE_TYPE_PERFORMANCE_BIT_EXT;
    debugCreateInfo.pfnUserCallback = VulkanRendererGpu::DefaultDebugMessengerCallback;
    // CreateDebugUtilsMessenger(...)
}
```

When you see `VUID-...` in the console, search the Vulkan spec for that ID — validation is teaching you the contract.

### Step 4 — Destroy order in destructor

```cpp
// VulkanDeviceContext destructor (abbreviated)
    WaitDeviceIdle();
    DestroySwapchain();
    vkDestroyDevice(device, nullptr);
    DestroyDebugUtilsMessenger(instance, debugMessenger, nullptr);
    vkDestroySurfaceKHR(instance, surface, nullptr);
    vkDestroyInstance(instance, nullptr);
```

Instance is last after surface — matches Chapter 2's dependency graph.

## 5. How This Connects to the Rest of Your Engine

Chapter **7** selects a physical device from this instance. Chapter **5** (window) supplied the extensions passed here. Keep validation on through Part III — shader and barrier mistakes are easier to fix with messenger text than with a black screen.

## 6. Pitfalls and Debugging

- **Missing surface extension** — `vkCreateInstance` succeeds but `glfwCreateWindowSurface` fails; check GLFW extension list.
- **Layers not installed** — SDK provides them; LunarG installer adds layer JSON to system path.
- **API version too high** — lower `apiVersion` to `VK_API_VERSION_1_2` if targeting older hardware.
- **Ignoring validation warnings** — warnings become errors on other GPUs; fix root cause early.

## 7. Exercises

1. Intentionally omit one extension in a local branch — read the validation or error message.
2. List layers with `vkEnumerateInstanceLayerProperties` in a tiny test program.
3. Change messenger to also print `INFO` severity — how noisy is one frame?
4. Document destroy order as a checklist in your engine notes.

## Summary

`VkInstance` bootstraps Vulkan with app metadata, GLFW surface extensions, and optional validation layers. Spark centralizes extension merging in `VulkanGpuInstance` and wires a debug messenger in debug builds. Instance creation is the first Vulkan call that must succeed for the rest of Part I.

---

Next: [Physical and Logical Device](07-physical-logical-device.md).
