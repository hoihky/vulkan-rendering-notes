# Part 16: Production Hardening

Harden swapchain recreation, GPU memory, async asset loading, profiling, and shipping templates so your renderer survives real players and real hardware.

| # | Chapter |
|---|---------|
| 1 | [Swapchain Recreation](01-swapchain-recreate.md) |
| 2 | [GPU Memory Strategy](02-memory.md) |
| 3 | [Async Assets](03-async-assets.md) |
| 4 | [Profiling](04-profiling.md) |
| 5 | [Game Templates](05-shipping.md) |

**Arc:** `RecreateSwapchain` debounce + OUT_OF_DATE handling → `VulkanGpuBufferImage` rings and `DeferredUploadBatch` → `GameWorldAssetLoader` worker + `SceneManager::Pump` → `FrameTiming` scopes and RenderDoc labels → retail `MyGame` target with quality presets.

Continue to [Appendices](../appendix/index.md).
