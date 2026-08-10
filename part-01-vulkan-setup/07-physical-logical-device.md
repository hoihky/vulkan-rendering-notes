# Physical and Logical Device

> **Build-Along Milestone:** You can explain how Spark picks a GPU, finds queue families, and creates a logical device with the swapchain extension.

**You will learn:**

- Enumerate physical devices and test suitability for presentation.
- Find graphics and present queue family indices.
- Create `VkDevice` with required extensions and retrieve queues.

## 1. The Problem We Are Solving

A machine may have **multiple GPUs** — integrated, discrete, software rasterizer. Vulkan exposes each as `VkPhysicalDevice`. You must pick one that can **draw** and **present to your window's surface**, then create a **logical device** (`VkDevice`) that opens the queues you need.

This chapter is the bridge from instance-level setup to device-level resources. Without a logical device, you cannot create swapchains, buffers, or pipelines.

## 2. Core Concepts

### Physical vs logical device

| Handle | Role |
|--------|------|
| `VkPhysicalDevice` | Query-only — properties, memory heaps, queue families |
| `VkDevice` | Create resources, submit work, own queues |

You never destroy a physical device — it disappears when the instance is destroyed. You **do** destroy logical devices with `vkDestroyDevice`.

### Queue families

GPUs expose one or more **queues** grouped into families. Each family supports a subset of operations (`VK_QUEUE_GRAPHICS_BIT`, `VK_QUEUE_COMPUTE_BIT`, etc.).

For windowed rendering you need:

1. A family with **graphics** support (drawing commands).
2. A family that supports **presentation** to your surface (`vkGetPhysicalDeviceSurfaceSupportKHR`).

They may be the same index (common on discrete GPUs) or different (some laptops).

### Device suitability

Spark's `IsDeviceSuitable` checks:

- Complete graphics + present queue families
- Non-empty swapchain formats and present modes
- `VK_KHR_swapchain` extension available
- On Apple: `VK_KHR_portability_subset`

Production engines also score VRAM, discrete vs integrated, and ray-tracing features. Spark picks the **first** suitable device — fine for learning; you may add GPU preference later.

### Logical device creation

`vkCreateDevice` takes:

- One or two `VkDeviceQueueCreateInfo` (same family → one create info; different → two)
- Enabled device extensions (`VK_KHR_swapchain`)
- Optional enabled layers (validation, mirroring instance)
- `VkPhysicalDeviceFeatures` / feature chains (Spark enables extended dynamic state)

After creation:

```cpp
vkGetDeviceQueue(device, graphicsFamily, 0, &graphicsQueue);
vkGetDeviceQueue(device, presentFamily, 0, &presentQueue);
```

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### First suitable device wins

**Concept:** Device suitability

**Decision:** Loop enumerated physical devices; break on first `IsDeviceSuitable`

**Why:** Predictable on dev machines with one GPU; avoids complex scoring until you need multi-GPU (Part XVI).

**In Spark:** `VulkanDeviceContext::SelectPhysicalDevice`.

### Queue family helper struct

**Concept:** Queue families

**Decision:** `QueueFamilyIndices` with `Complete()` predicate

**Why:** Centralizes the graphics+present check; swapchain creation reuses the same indices.

**In Spark:** `VulkanGpuPhysicalDevice.hpp` — `FindQueueFamilies`.

### Swapchain extension required at device level

**Concept:** Logical device creation

**Decision:** Always enable `VK_KHR_SWAPCHAIN_EXTENSION_NAME`

**Why:** Without it, `vkCreateSwapchainKHR` is unavailable even if instance created successfully.

**In Spark:** `VulkanDeviceContext::CreateLogicalDevice` — `deviceExtensions` array.

### Duplicate queue create infos when families differ

**Concept:** Queue families

**Decision:** Two `VkDeviceQueueCreateInfo` when graphics ≠ present family

**Why:** Vulkan requires explicit queue creation per family used; `VK_SHARING_MODE_CONCURRENT` on swapchain handles cross-family images later.

**In Spark:** `CreateLogicalDevice` branches on `graphicsFamily == presentFamily`.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book; Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `include/spark/render/gpu/VulkanGpuPhysicalDevice.hpp`
- `src/spark/render/gpu/VulkanGpuPhysicalDevice.cpp`
- `src/spark/render/core/VulkanDeviceContext.cpp`

### Step 1 — Enumerate physical devices

```cpp
// VulkanDeviceContext.cpp (lines 140–159)
void VulkanDeviceContext::SelectPhysicalDevice() {
    std::uint32_t deviceCount = 0;
    vkEnumeratePhysicalDevices(instance, &deviceCount, nullptr);
    if (deviceCount == 0) {
        throw std::runtime_error("no Vulkan-capable GPU found");
    }

    Array<VkPhysicalDevice> devices;
    devices.Resize(static_cast<std::size_t>(deviceCount));
    vkEnumeratePhysicalDevices(instance, &deviceCount, devices.GetData());
    for (std::size_t di = 0; di < devices.GetSize(); ++di) {
        VkPhysicalDevice dev = devices[di];
        if (VulkanRendererGpu::IsDeviceSuitable(dev, surface)) {
            physicalDevice = dev;
            break;
        }
    }
    if (physicalDevice == VK_NULL_HANDLE) {
        throw std::runtime_error("no suitable Vulkan physical device");
    }

    queueFamilies = VulkanRendererGpu::FindQueueFamilies(physicalDevice, surface);
}
```

**Build-along:** Log `deviceCount` and property names with `vkGetPhysicalDeviceProperties` in a scratch program.

### Step 2 — Find queue families

```cpp
// VulkanGpuPhysicalDevice.cpp (lines 21–35, abbreviated)
    for (std::uint32_t i = 0; i < count; ++i) {
        if (props[i].queueFlags & VK_QUEUE_GRAPHICS_BIT) {
            indices.graphicsFamily = i;
            indices.hasGraphicsFamily = true;
        }
        VkBool32 presentSupport = VK_FALSE;
        vkGetPhysicalDeviceSurfaceSupportKHR(physicalDevice, i, surface, &presentSupport);
        if (presentSupport) {
            indices.presentFamily = i;
            indices.hasPresentFamily = true;
        }
        if (indices.Complete()) {
            break;
        }
    }
```

### Step 3 — Suitability including swapchain

```cpp
// VulkanGpuPhysicalDevice.cpp (lines 55–72)
bool VulkanGpuPhysicalDevice::IsDeviceSuitable(VkPhysicalDevice physicalDevice, VkSurfaceKHR surface) {
    const QueueFamilyIndices queueFamilies = FindQueueFamilies(physicalDevice, surface);
    if (!queueFamilies.Complete()) {
        return false;
    }
    const SwapchainSupportDetails swapchainSupport = VulkanGpuSwapchain::QuerySwapchainSupport(physicalDevice, surface);
    if (swapchainSupport.formats.IsEmpty() || swapchainSupport.presentModes.IsEmpty()) {
        return false;
    }
    if (!DeviceExtensionSupported(physicalDevice, VK_KHR_SWAPCHAIN_EXTENSION_NAME)) {
        return false;
    }
    // Apple portability check ...
    return true;
}
```

### Step 4 — Create logical device

```cpp
// VulkanDeviceContext.cpp (lines 164–226, structure)
void VulkanDeviceContext::CreateLogicalDevice() {
    // Build queueCreateInfos (one or two families)
    // Enable VK_KHR_SWAPCHAIN_EXTENSION_NAME
    // Optional validation layers on device
    vkCreateDevice(physicalDevice, &deviceCreateInfo, nullptr, &device);
    vkGetDeviceQueue(device, queueFamilies.graphicsFamily, 0, &graphicsQueue);
    vkGetDeviceQueue(device, queueFamilies.presentFamily, 0, &presentQueue);
}
```

Spark also chains `VkPhysicalDeviceExtendedDynamicStateFeaturesEXT` for pipeline state flexibility in later parts.

### Step 5 — Verify queues

After SparkDemo runs, you have a valid `VkDevice` even if you never see draws yet. Validation should stay silent through device creation on a supported GPU.

## 5. How This Connects to the Rest of Your Engine

Chapter **8** creates the swapchain on this device using `queueFamilies` and swapchain support queries introduced here. All buffer and image creation in Part II calls `device` from `VulkanDeviceContext`.

If device creation fails, fix suitability before debugging swapchain code.

## 6. Pitfalls and Debugging

- **No suitable device on laptop** — ensure discrete GPU is used; update drivers; check surface not null.
- **Forgetting swapchain extension** — `vkCreateSwapchainKHR` entry point null.
- **Same queue priority 1.0** — fine for tutorials; real engines may prioritize async compute lower.
- **Selecting integrated GPU** — first suitable may be iGPU; add preference for discrete if both exist.

## 7. Exercises

1. Print all physical device names before selection in a local Spark build.
2. On your machine, are graphics and present the same family index?
3. List swapchain formats with `vkGetPhysicalDeviceSurfaceFormatsKHR` in a test harness.
4. What happens if you skip `VK_KHR_swapchain` in device extensions?

## Summary

Pick a physical device that can render and present, record queue family indices, and create a logical device with swapchain support. Spark wraps enumeration and suitability in `VulkanGpuPhysicalDevice` and sequences creation in `VulkanDeviceContext` — the pattern every Vulkan app follows after instance and surface setup.

---

Next: [Swapchain and Presentation](08-swapchain.md).
