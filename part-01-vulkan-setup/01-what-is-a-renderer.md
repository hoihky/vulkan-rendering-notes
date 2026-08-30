# What Is a Real-Time Rendering Engine?

> **Build-Along Milestone:** You can explain Spark's three-layer flow and locate the main loop, presenter interface, and snapshot copy — the mental map for every Part I build step.

**You will learn:**

- Separate simulation, scene description, and GPU presentation in your mental model.
- Explain why modern APIs like Vulkan expose explicit resource lifetimes.
- Trace Spark's per-frame path from `OnUpdate` through `PresentFrame`.

## 1. The Problem We Are Solving

In this chapter we orient ourselves inside **Spark** — the reference mini-engine you build and diff against throughout the book. Open the Spark repo, run `./cmake-build-debug/SparkDemo`, and keep `src/Engine.cpp` visible while you read.

You want pixels on screen at 60+ frames per second. A **real-time rendering engine** repeatedly:

1. **Simulates** the world — movement, animation, gameplay rules.
2. **Describes** what should appear in the next image — meshes, lights, camera.
3. **Executes** GPU work to turn that description into pixels and presents them.

Merge steps (2) and (3) — for example calling `vkCmdDraw` from gameplay — and you drown in platform details. Spark inserts a **render snapshot** (`SceneRenderParams`) that `VulkanRenderer` consumes without knowing game rules. Part I builds step (3)'s infrastructure; Part II connects the snapshot to draw calls.

### The rendering loop in one sentence

Every real-time game repeats forever:

```text
simulate world → describe what to draw → GPU draws it → show on screen → repeat
```

If you are completely new to graphics programming, memorize those four verbs. Everything in this book expands one of them:

| Verb | Parts in this book | Spark example |
|------|-------------------|---------------|
| Simulate | V, XIV, XV | `OnUpdate`, physics, input |
| Describe | VI–VII, XIII | `SceneSubmit`, components |
| Draw | I–III, IV, IX–XI | `VulkanRenderer`, shaders |
| Show | I, X | swapchain, tonemap, present |

Vulkan lives entirely in **draw**. C++ gameplay lives in **simulate** and **describe**. Keeping them separate is the single most important architectural decision you will make.

## 1b. Concept Deep Dive — The Snapshot Contract Between Game and GPU

### Why a plain struct beats a live scene pointer

Chapter 1's four verbs collapse into three code layers in Spark: gameplay mutates ECS state, `SceneSubmit` flattens that state into `SceneRenderParams`, and `VulkanRenderer` consumes the snapshot during command recording. The renderer never walks `GameObject` lists because the GPU timeline lags the CPU by one or more frames — if the backend held a pointer to a game-owned vector, a resize during `vkQueueSubmit` would be a use-after-free.

Spark therefore **copies** params once per frame into `pendingScene`. While the GPU executes frame *N*, the CPU may already be writing frame *N+1* into a different logical slot (Chapter 10 adds per-flight buffers). The snapshot is the **immutable contract** for the submission about to be recorded: draw items, lights, clear color, and texture layer indices are frozen at `SetSceneRenderParams` time.

The presenter interface keeps Vulkan types out of game headers. `IEngineContext::SetSceneRenderParams` forwards to `IFramePresenter::SetSceneRenderParams`; demos call the facade from `OnRender`, never from `OnUpdate`. That ordering guarantees simulation finished before the describe step runs.

### Data flow through Spark (one frame)

```text
OnUpdate (ECS)          OnRender / SceneSubmit           DrawFrame (Vulkan)
     │                         │                                │
     │  mutate transforms      │  FillStandardLitSceneFromWorld │
     │  run physics            │  → SceneRenderParams           │
     │                         │  → SetSceneRenderParams        │
     │                         │         │                      │
     │                         │         ▼                      │
     │                         │    pendingScene (copy) ───────►│ RecordSceneCommandBuffer
     │                         │                                │ vkQueueSubmit → Present
```

| Stage | Spark owner | Vulkan headers? |
|-------|-------------|-----------------|
| Simulate | `IGame::OnUpdate`, ECS | No |
| Describe | `SceneSubmit`, `OnRender` | No |
| Execute | `VulkanRenderer::DrawFrame` | Yes |

### Frontend submit — params then forward

Demos build the snapshot on the CPU side and push it through the engine facade; they never touch `VulkanRenderer` directly:

```cpp
// src/spark/scene/submit/SceneSubmit.cpp (lines 617–632)
    SceneRenderParams params{};
    FillStandardLitSceneFromWorld(world,
            context,
            viewProjection,
            cameraPositionWorld,
            lightDirectionWorld,
            lightColor,
            lightIntensity,
            ambientColor,
            enableParticles,
            particleCameraRight,
            particleCameraUp,
            sceneTimeSeconds,
            params,
            spriteSortMode);
    context.SetSceneRenderParams(params);
```

### Facade forwards without exposing the backend

```cpp
// src/spark/engine/EngineContext.cpp (lines 41–43)
void EngineContext::SetSceneRenderParams(const SceneRenderParams& params) {
    presenter.SetSceneRenderParams(params);
}
```

### Backend copy and lighting resolve

The renderer stores its own copy, merges texture layers, and pre-resolves lighting so record passes read stable data:

```cpp
// src/spark/render/core/VulkanRenderer.cpp (lines 579–590)
void VulkanRenderer::SetSceneRenderParams(const SceneRenderParams& params) {
    if (!SceneHasCustomMeshDraws(params)) {
        customMeshPool.ClearKnownMeshes();
    }
    mergedSceneTextures = params.sceneTextures;
    mergedSceneHdrTextures = params.sceneHdrTextures;
    pendingScene = params;
    pendingScene.sceneTextures = mergedSceneTextures;
    pendingScene.sceneHdrTextures = mergedSceneHdrTextures;
    SceneSubmitDetail::ResolveIblEnvironmentLayer(pendingScene);
    sceneParamsValid = true;
    resolvedLighting = SceneLightingResolver::Resolve(pendingScene);
}
```

Every Part I chapter builds infrastructure for the right side of this diagram; Part II connects real draw calls to `pendingScene`.

## 2. Core Concepts

### Frame budget and the 16 ms wall

At 60 FPS you have **16.7 ms** of wall-clock time per frame. Both CPU and GPU compete for that budget:

| Stage | Typical CPU/GPU share | Spark touchpoint |
|-------|----------------------|------------------|
| Input + gameplay | 25–40% CPU | `Engine::Run` → `game->OnUpdate` |
| Scene build | 5–15% CPU | `SceneSubmit` → `SceneRenderParams` |
| GPU rendering | 45–60% GPU | `VulkanRenderer::DrawFrame` |
| Present / compositor | < 5% | `vkQueuePresentKHR` |

The formula for sustainable frame rate:

```text
T_frame = T_sim + T_snapshot + T_record + T_gpu + T_present ≤ 16.7 ms
```

If `T_sim + T_snapshot` grows, either drop quality or miss v-sync. Profile both sides from day one — a "GPU bound" game can still miss frames because the CPU never submitted work in time.

### Three-layer architecture

Professional engines split responsibilities into three layers with **one-way data flow**:

```text
┌─────────────┐    SceneRenderParams     ┌──────────────────┐
│  Game / ECS │ ───────────────────────► │  VulkanRenderer  │
│  (simulate) │   plain C++ structs only │  (record + GPU)  │
└─────────────┘                          └──────────────────┘
       ▲                                           │
       │ input                                     ▼
  GlfwInput                                  swapchain pixels
```

Layer 1 never includes `<vulkan/vulkan.h>`. Layer 3 never queries entity components. The snapshot is the **contract** — immutable for the frame slot the GPU is reading.

### Forward rendering preview

This book builds a **forward** renderer: each draw evaluates lighting in the fragment shader immediately. **Deferred** renderers write material attributes to G-buffer textures and light in a second pass — better for hundreds of dynamic lights, harder to bootstrap.

| Approach | Draw cost | Light cost | Bootstrap difficulty |
|----------|-----------|------------|---------------------|
| Forward (Spark default) | O(objects × lights) with clustering | Per fragment in same pass | Low |
| Deferred | O(objects) to G-buffer | O(lights × pixels) | High |

Spark uses **clustered forward** (Part IV) to scale point/spot lights without a full G-buffer. You will not need deferred until open-world scale in later parts.

### Explicit APIs vs driver guessing

Vulkan exposes a **state machine** you drive explicitly:

| Object | You create | Driver does |
|--------|-----------|-------------|
| `VkInstance` | App metadata + extensions | Layer dispatch |
| `VkDevice` | Queues + features | Scheduling |
| `VkSwapchainKHR` | Image ring for display | Compositor handoff |
| Command buffers | Recorded draw commands | Execute on GPU |

OpenGL hid synchronization; Vulkan makes it visible. The upside: predictable performance, multi-threaded command recording, and validation layers that catch lifetime bugs at development time.

### Part I roadmap

Part I ends with **frame sync** — you can clear the swapchain every frame with correct fences and semaphores. Part II connects `SceneRenderParams` to `vkCmdDrawIndexed`. Use this map:

| Part I chapter | Spark milestone |
|----------------|-----------------|
| 5 Window | GLFW surface ready |
| 6 Instance | Validation active |
| 7 Device | Queues + swapchain ext |
| 8 Swapchain | Image views created |
| 9 DeviceContext | Central GPU owner |
| 10 Frame sync | Acquire/submit/present |

**Build-along:** Run SparkDemo after each chapter and confirm the previous milestone still works before adding the next Vulkan object.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Game code never includes Vulkan headers

**Concept:** Three-layer architecture

**Decision:** Game code never includes Vulkan headers

**Why:** Gameplay headers must compile on teams that unit-test simulation without a GPU. Vulkan types in game code create link-time coupling and circular dependencies when assets need render constants.

**In Spark:** `include/spark/engine/IEngineContext.hpp` — `SetSceneRenderParams` forwards to `IFramePresenter` with zero Vulkan types in the facade.
### Snapshot is copied once per frame, not referenced

**Concept:** One-way data flow

**Decision:** Snapshot is copied once per frame, not referenced

**Why:** If the renderer held a pointer to a game-owned vector, a resize during GPU execution causes use-after-free. Copying `SceneRenderParams` into `VulkanRenderer::pendingScene` isolates lifetimes.

**In Spark:** `src/spark/render/core/VulkanRenderer.cpp` — `SetSceneRenderParams` assigns `pendingScene = params`.
### Engine owns renderer construction, game owns simulation

**Concept:** Explicit APIs

**Decision:** Engine owns renderer construction, game owns simulation

**Why:** Construction order matters: window before surface, device before swapchain. Centralizing `VulkanRenderer` construction in `Engine::Impl` guarantees valid initialization order.

**In Spark:** `src/Engine.cpp` — `Impl` constructs `Window`, then `VulkanRenderer(window)`, then `EngineContext`.
### Update and render are separate virtual calls

**Concept:** Frame budget

**Decision:** Update and render are separate virtual calls

**Why:** Separating `OnUpdate` (simulation) from `OnRender` (snapshot build) lets you measure each stage independently and keeps input sampling before simulation.

**In Spark:** `src/Engine.cpp` — loop calls `OnUpdate` then `OnRender` then `PresentFrame`.
### Present is the only GPU entry from the loop

**Concept:** IFramePresenter abstraction

**Decision:** Present is the only GPU entry from the loop

**Why:** Games should not call `DrawFrame` directly. `PresentFrame()` is the stable contract; internal acquire/submit/present details stay in `VulkanRenderer`.

**In Spark:** `include/spark/engine/IFramePresenter.hpp` — `PresentFrame()` virtual; `src/spark/render/core/VulkanRenderer.cpp` implements via `DrawFrame()`.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `src/Engine.cpp`
- `include/spark/engine/IEngineContext.hpp`
- `include/spark/engine/IFramePresenter.hpp`
- `include/spark/engine/SceneRenderParams.hpp`
- `include/spark/render/core/VulkanRenderer.hpp`
- `src/spark/render/core/VulkanRenderer.cpp`

### Step 1 — Locate Spark's main loop

Every engine shares the same skeleton. In Spark, `Engine::Run` owns the while-loop. Follow along: open `src/Engine.cpp` and find the poll → update → render → present sequence.
*Spark's per-frame loop — your mini-engine will mirror this shape.*

```cpp
// src/Engine.cpp (lines 132–200)
        while (!window.ShouldClose()) {
            window.PollEvents();
            input->BeginFrame();

            if (input->IsKeyPressedThisFrame(GLFW_KEY_F12)) {
                char shotPath[512]{};
                const auto now = std::chrono::system_clock::now();
                const std::time_t nowSec = std::chrono::system_clock::to_time_t(now);
                std::tm localTm{};
#if defined(_WIN32)
                localtime_s(&localTm, &nowSec);
#else
                localtime_r(&nowSec, &localTm);
#endif
                std::snprintf(
                        shotPath,
                        sizeof(shotPath),
                        "%s/screenshots/spark_%04d%02d%02d_%02d%02d%02d.png",
                        SPARK_BUILD_ASSETS_DIR,
                        localTm.tm_year + 1900,
                        localTm.tm_mon + 1,
                        localTm.tm_mday,
                        localTm.tm_hour,
                        localTm.tm_min,
                        localTm.tm_sec);
                engineContext->GetFramePresenter().RequestScreenshotSave(shotPath);
            }

            if (input->IsKeyPressedThisFrame(GLFW_KEY_F3) || input->IsKeyPressedThisFrame(GLFW_KEY_F4)) {
                ToggleDemoFpsOverlayVisible();
            }

            if (input->IsKeyPressedThisFrame(GLFW_KEY_F9)) {
                IFramePresenter& presenter = engineContext->GetFramePresenter();
                if (presenter.IsVideoRecording()) {
                    StopVideoRecording(presenter, audio);
                } else {
                    const VideoRecordingSettings settings =
                            RecordingSettingsFromRunOptions(runOptions, BuildTimestampedRecordingPath());
                    (void)StartVideoRecording(presenter, audio, settings);
                }
            }

            if (autoRecordPending) {
                const VideoRecordingSettings settings =
                        RecordingSettingsFromRunOptions(runOptions, runOptions.autoRecordPath);
                if (StartVideoRecording(engineContext->GetFramePresenter(), audio, settings)) {
                    autoRecordPending = false;
                }
            }

            const auto now = Clock::now();
            const float deltaSeconds = std::chrono::duration<float>(now - previous).count();
            previous = now;
            totalSeconds += deltaSeconds;

            const FrameTiming timing{deltaSeconds, totalSeconds, frameIndex};

            game->OnUpdate(timing, *engineContext);
            audio.PumpFrame(timing.deltaTimeSeconds);

            Ui::UiSystem::Get().OnEnginePreRender(window, *input, timing.deltaTimeSeconds);

            game->OnRender(renderFrame, *engineContext);

            Ui::UiSystem::Get().OnEnginePostRender();

            renderer->PresentFrame();
            ++frameIndex;
```
### Step 2 — Identify the frame presenter interface

Before reading Vulkan details, understand what the loop expects from the GPU backend. `IFramePresenter` is deliberately small — present, resize notification, optional scene params.
*The presenter contract your renderer implements.*

```cpp
// include/spark/engine/IFramePresenter.hpp (lines 13–36)
class IFramePresenter {
public:
    virtual ~IFramePresenter() = default;

    /** Submits the current frame to the display (e.g. Vulkan present). */
    virtual void PresentFrame() = 0;

    /** Called when the window framebuffer size changes. */
    virtual void NotifySwapchainResize() = 0;

    /**
     * Pixel size of the drawable actually rendered (e.g. swapchain). When true, matches UI projection / hit tests.
     * Default: unknown — use Window / GLFW framebuffer size instead.
     */
    [[nodiscard]] virtual bool TryGetDrawableSize(int& outWidth, int& outHeight) const {
        (void)outWidth;
        (void)outHeight;
        return false;
    }

    /** Optional: Vulkan backend uses this for lit meshes; default is no-op. */
    virtual void SetSceneRenderParams(const SceneRenderParams& params) {
        (void)params;
    }
```
### Step 3 — Trace scene params into the renderer

Games never call Vulkan directly. They fill `SceneRenderParams` and pass it through `IEngineContext`. This is the snapshot pattern you will extend through Part VIII.
*Engine facade — note zero Vulkan includes.*

```cpp
// include/spark/engine/IEngineContext.hpp (lines 17–35)
class IEngineContext {
public:
    virtual ~IEngineContext() = default;

    [[nodiscard]] virtual Window& GetWindow() = 0;
    [[nodiscard]] virtual IFramePresenter& GetFramePresenter() = 0;
    [[nodiscard]] virtual IInput& GetInput() = 0;

    /** Drawable pixel size for rendering and UI (swapchain when available, else window framebuffer). */
    virtual void GetFramebufferSize(int& outWidth, int& outHeight) const = 0;

    /** Forwards to the frame presenter (Vulkan, etc.); no need to include IFramePresenter. */
    virtual void SetSceneRenderParams(const SceneRenderParams& params) = 0;

    /** Read-modify-write last scene params (Vulkan presenter). */
    [[nodiscard]] virtual bool TryGetMutableSceneRenderParams(SceneRenderParams*& outParams) noexcept {
        (void)outParams;
        return false;
    }
```
### Step 4 — See how the renderer stores the snapshot

`VulkanRenderer::SetSceneRenderParams` copies params into `pendingScene`. While the GPU reads frame *N*, the CPU may write frame *N+1* into a different slot — but must not mutate the copy still in flight.
*Snapshot copy — immutability per frame slot starts here.*

```cpp
// src/spark/render/core/VulkanRenderer.cpp (lines 521–525)
void VulkanRenderer::SetSceneRenderParams(const SceneRenderParams& params) {
    pendingScene = params;
    sceneParamsValid = true;
    resolvedLighting = SceneLightingResolver::Resolve(pendingScene);
}
```
### Step 5 — Map ownership to Spark classes

Draw this table and label Spark's owner for each row:

| Object | Owner in Spark | Lifetime |
|--------|---------------|----------|
| Window / surface | `Window` + `VulkanDeviceContext` | app |
| `VkDevice` | `VulkanDeviceContext` | app |
| Swapchain images | `VulkanPresentationSwapchain` | until resize |
| Command buffers | `VulkanRenderer::commandBuffers` | per swapchain image |
| Per-frame sync | `VulkanFrameSync` | ring of 2 frames |

Every Vulkan object you create in Part I must fit one row.
### Step 6 — Plan your Part I milestone

**Your milestone this part:** clear the swapchain each frame with working sync.

**Part II milestone:** draw a colored triangle via render pass + pipeline.

Run SparkDemo now — even before you understand every line, you have a target end state. Diff your work against Spark chapter by chapter.

## 5. How This Connects to the Rest of Your Engine

Part **II** implements real `vkCmdDrawIndexed` calls inside a render pass. Part **III** authors the shaders loaded into `VulkanScenePipeline`. Part **IV** extends `SceneDrawItem` with texture layers and lights. Part **V** fills `viewProjection` from camera components.

Memorize this arrow:

```text
[OnUpdate] → [OnRender / SceneSubmit] → SceneRenderParams → [DrawFrame] → [Present]
```

Part I builds everything right of the params arrow; Part II connects the left side.

## 6. Pitfalls and Debugging

- **Calling Vulkan from `OnUpdate()`** — couples simulation to the GPU timeline; validation errors become gameplay bugs.
- **Reusing one uniform buffer without fencing** — transforms flicker or tear; Spark uses per-flight buffers in `VulkanSceneDescriptors`.
- **Treating 16 ms as GPU-only** — `SceneSubmit` for 500 entities can exceed budget before the GPU starts.
- **Skipping the snapshot struct** — passing `Scene*` to the renderer creates circular includes within a week.
- **Copying Spark file-for-file** — learn the *pattern*; your engine may split classes differently.

## 7. Exercises

1. Run SparkDemo and press F3 — locate where the FPS overlay writes into `SceneRenderParams`.
2. Draw the three-layer diagram for SparkDemo's ThreeDDemo mode. Label ECS, submit, and Vulkan.
3. At 60 FPS with 200 `SceneDrawItem` entries, how many draw records per second?

## Summary

A rendering engine simulates, describes, executes on GPU, and presents. Spark's `SceneRenderParams` + `IFramePresenter` keep that description clean while Vulkan stays behind the presenter interface.

---

Next: [Engine Architecture at a Glance](02-engine-architecture.md).
