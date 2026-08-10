# Vulkan Particle Pass

> **Build-Along Milestone:** RenderDoc shows particle draw with correct vertex stride and additive pipeline.

**You will learn:**

- Describe VulkanParticlePass resource creation: shaders, UBO, host-visible vertex buffer.
- Explain CPU-side quad expansion from SceneParticleInstance to vertex data.
- Identify early-out conditions and buffer capacity limits in Record.

## 1. The Problem We Are Solving

The lit mesh pipeline assumes rigid transforms and material descriptors. Particles need **billboard expansion,
additive blend, and a lightweight vertex format** — bundled into `VulkanParticlePass` instead of special-casing scene draws.

Build along: trace Record from `scene.particles` through scratchVertices to `vkCmdDraw`.

## 2. Core Concepts

### Host-visible vertex ring

Spark maps a 10 MB host-coherent vertex buffer once. Record memcpy's expanded quads each frame — no staging copy or device-local buffer for particles.


### CPU quad expansion

Each SceneParticleInstance becomes six vertices (two triangles). Each vertex carries center position, color, size, and corner offset (-1..1 billboard corner).


### 10-float vertex layout

position(3) + color(4) + size(1) + corner(2) = 10 floats per vertex — matches VkVertexInputAttributeDescription.


### Per-frame UBO

ParticleUniformGpu holds viewProj, cameraPos, cameraRight, cameraUp — one descriptor set per frame in flight.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### HOST_VISIBLE | HOST_COHERENT VB

**Concept:** Host-visible vertex ring

**Decision:** HOST_VISIBLE | HOST_COHERENT VB

**Why:** Particle count varies wildly; memcpy is simpler than ring-buffered device uploads at this scale.

**In Spark:** `VulkanParticlePass::CreateGpuResources` vertex buffer

### Expand at Record not in shader

**Concept:** CPU quad expansion

**Decision:** Expand at Record not in shader

**Why:** Procedural VS with SSBO is future work; Spark chooses explicit expansion for clarity and MoltenVK safety.

**In Spark:** `VulkanParticlePass::Record` scratchVertices loop

### Four attribute locations

**Concept:** 10-float vertex layout

**Decision:** Four attribute locations

**Why:** Corner offset lets VS reconstruct billboard without instancing API — one vkCmdDraw for all verts.

**In Spark:** `kParticleFloatsPerVertex = 10`

### Descriptor set per frame in flight

**Concept:** Per-frame UBO

**Decision:** Descriptor set per frame in flight

**Why:** Avoids GPU race on uniform writes — same pattern as scene descriptors.

**In Spark:** `uniformBuffers[frameIndex]` in Record

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `include/spark/render/sprites2d/VulkanParticlePass.hpp`
- `src/spark/render/sprites2d/VulkanParticlePass.cpp`

### Step 1 — Read pass public API

*CreateGpuResources, CreateGraphicsPipeline, Record.*

```cpp
// include/spark/render/sprites2d/VulkanParticlePass.hpp (lines 13–31)
/** GPU particle billboards rendered into the HDR pass. */
class VulkanParticlePass {
public:
    void CreateGpuResources(
            VkPhysicalDevice physicalDevice,
            VkDevice device,
            std::uint32_t framesInFlight,
            const VulkanSpvShaderLoader& shaders);
    void DestroyGpuResources(VkDevice device);

    void CreateGraphicsPipeline(VkDevice device, VkRenderPass hdrRenderPass);
    void DestroyGraphicsPipeline(VkDevice device);

    void Record(
            VkCommandBuffer commandBuffer,
            std::uint32_t frameIndex,
            VkExtent2D extent,
            const SceneRenderParams& scene,
            bool sceneParamsValid) const;
```

### Step 2 — Create GPU resources

*Descriptor layout, pool, UBO buffers per frame.*

```cpp
// src/spark/render/sprites2d/VulkanParticlePass.cpp (lines 19–40)
void VulkanParticlePass::CreateGpuResources(
        const VkPhysicalDevice physicalDevice,
        const VkDevice device,
        const std::uint32_t framesInFlight,
        const VulkanSpvShaderLoader& shaders) {
    if (device == VK_NULL_HANDLE) {
        return;
    }

    VkDescriptorSetLayoutBinding uboBinding{};
    uboBinding.binding = 0;
    uboBinding.descriptorType = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
    uboBinding.descriptorCount = 1;
    uboBinding.stageFlags = VK_SHADER_STAGE_VERTEX_BIT;

    VkDescriptorSetLayoutCreateInfo dslInfo{};
    dslInfo.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO;
    dslInfo.bindingCount = 1;
    dslInfo.pBindings = &uboBinding;
    if (vkCreateDescriptorSetLayout(device, &dslInfo, nullptr, &descriptorSetLayout) != VK_SUCCESS) {
        throw std::runtime_error("vkCreateDescriptorSetLayout (particle) failed");
    }
```

### Step 3 — Load particle shaders

*Read particle.vert.spv and particle.frag.spv from shader loader.*

```cpp
// src/spark/render/sprites2d/VulkanParticlePass.cpp (lines 119–122)
    const Array<char> pv = shaders.ReadSpvFile("particle.vert.spv");
    const Array<char> pf = shaders.ReadSpvFile("particle.frag.spv");
    vertModule = shaders.CreateShaderModule(pv);
    fragModule = shaders.CreateShaderModule(pf);
```

### Step 4 — Study Record early-out and cap

*Skip if empty; clamp count by MaxParticles and vertex buffer capacity.*

```cpp
// src/spark/render/sprites2d/VulkanParticlePass.cpp (lines 322–344)
void VulkanParticlePass::Record(
        const VkCommandBuffer commandBuffer,
        const std::uint32_t frameIndex,
        const VkExtent2D extent,
        const SceneRenderParams& scene,
        const bool sceneParamsValid) const {
    if (!sceneParamsValid || pipeline == VK_NULL_HANDLE || vertexMapped == nullptr ||
        frameIndex >= descriptorSets.GetSize() || scene.particles.IsEmpty()) {
        return;
    }

    const std::size_t maxByBuffer =
            static_cast<std::size_t>(vertexCapacityBytes / (sizeof(float) * kParticleFloatsPerVertex)) / 4U;
    std::size_t nPart = scene.particles.GetSize();
    if (nPart > static_cast<std::size_t>(SceneRenderParams::MaxParticles)) {
        nPart = static_cast<std::size_t>(SceneRenderParams::MaxParticles);
    }
    if (nPart > maxByBuffer) {
        nPart = maxByBuffer;
    }
    if (nPart == 0) {
        return;
    }
```

### Step 5 — Trace quad expansion and draw

*Build scratchVertices, memcpy to mapped VB, bind pipeline, vkCmdDraw.*

```cpp
// src/spark/render/sprites2d/VulkanParticlePass.cpp (lines 362–412)
    scratchVertices.Clear();
    const float corners[4][2] = {{-1.0F, -1.0F}, {1.0F, -1.0F}, {1.0F, 1.0F}, {-1.0F, 1.0F}};
    const std::uint32_t idx0[] = {0, 1, 2, 0, 2, 3};
    for (std::size_t pi = 0; pi < nPart; ++pi) {
        const SceneParticleInstance& p = scene.particles[pi];
        for (int t = 0; t < 6; ++t) {
            const int k = static_cast<int>(idx0[t]);
            scratchVertices.PushBack(p.position.x);
            scratchVertices.PushBack(p.position.y);
            scratchVertices.PushBack(p.position.z);
            scratchVertices.PushBack(p.color.x);
            scratchVertices.PushBack(p.color.y);
            scratchVertices.PushBack(p.color.z);
            scratchVertices.PushBack(p.color.w);
            scratchVertices.PushBack(p.size);
            scratchVertices.PushBack(corners[k][0]);
            scratchVertices.PushBack(corners[k][1]);
        }
    }

    const VkDeviceSize vbBytes = sizeof(float) * static_cast<VkDeviceSize>(scratchVertices.GetSize());
    std::memcpy(vertexMapped, scratchVertices.GetData(), static_cast<std::size_t>(vbBytes));

    VkViewport viewport{};
    viewport.x = 0.0F;
    viewport.y = 0.0F;
    viewport.width = static_cast<float>(extent.width);
    viewport.height = static_cast<float>(extent.height);
    viewport.minDepth = 0.0F;
    viewport.maxDepth = 1.0F;
    vkCmdSetViewport(commandBuffer, 0, 1, &viewport);

    VkRect2D fullScissor{};
    VulkanScreenUiClip::BindScenePassScissor(commandBuffer, &scene, extent, fullScissor);

    vkCmdBindPipeline(commandBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, pipeline);
    vkCmdBindDescriptorSets(
            commandBuffer,
            VK_PIPELINE_BIND_POINT_GRAPHICS,
            pipelineLayout,
            0,
            1,
            &descriptorSets[frameIndex],
            0,
            nullptr);

    const VkDeviceSize vbOff = 0;
    vkCmdBindVertexBuffers(commandBuffer, 0, 1, &vertexBuffer, &vbOff);

    const std::uint32_t vertCount = static_cast<std::uint32_t>(scratchVertices.GetSize() / kParticleFloatsPerVertex);
    vkCmdDraw(commandBuffer, vertCount, 1, 0, 0);
```

### Step 6 — RenderDoc capture

Confirm one draw call after sprites; vertex buffer bound with stride 40 bytes (10 floats).

## 5. How This Connects to the Rest of Your Engine

**Collecting Particles** fills the input array. **Particle Shaders** interpret the 10-float vertex layout.
**Blend and Depth State** configured in CreateGraphicsPipeline. **Frame Graph** calls Record inside HDR subpass.

## 6. Pitfalls and Debugging

- **sceneParamsValid false** — Record returns immediately; check submit ran.
- **Vertex buffer overflow** — nPart clamped by buffer bytes / (10 floats × 6 verts per particle).
- **Pipeline null after resize** — recreate pipeline when HDR render pass changes.
- **Forgetting scissor restore** — Record uses VulkanScreenUiClip; must restore after draw.

## 7. Exercises

1. Calculate max particles fit in 10 MB buffer at 6 verts × 10 floats each.
2. Log `vertCount` passed to `vkCmdDraw` for one heavy emitter frame.
3. Capture RenderDoc: verify particle pipeline distinct from scene lit pipeline.
4. Force `scene.particles.IsEmpty()` — confirm `Record` early-outs.
5. List contents of one expanded vertex (10 floats) in order.

## Summary

VulkanParticlePass maps a host-visible VB, expands each particle to six vertices at Record, writes per-frame UBO, and issues one vkCmdDraw into the HDR subpass.

---

Next: [Particle Shaders](05-particle-shaders.md).
