# Engine Architecture at a Glance

> **Build-Along Milestone:** You can name Spark's four layers, list what `VulkanRenderer` composes, and explain why `IEngineContext` exists.

**You will learn:**

- Map Spark modules to platform, game, render-frontend, and render-backend layers.
- Explain dependency inversion via `IEngineContext` and `IFramePresenter`.
- Trace one frame from GLFW poll to `vkQueuePresentKHR`.

## 1. The Problem We Are Solving

Chapter 1 introduced the three-layer mental model: simulate → describe → present. This chapter adds **module boundaries** — the folders and classes that enforce that model in code.

Without boundaries, a Vulkan tutorial collapses into a single `main.cpp` that mixes GLFW callbacks, gameplay, and `vkCreateGraphicsPipeline`. Every feature becomes a merge conflict. Spark avoids that by splitting responsibilities into small types with explicit includes. Your job in Part I is not to memorize every class, but to know **which layer owns which concern** so you can navigate the repo while building your own engine.

Open these two headers side by side while you read:

- `include/spark/engine/IEngineContext.hpp` — what game code may call
- `include/spark/render/core/VulkanRenderer.hpp` — what the GPU backend owns

The difference in their `#include` lists is the architecture.

### Four layers for beginners

Spark organizes code into layers you should memorize early:

```text
┌─────────────────────────────────────────────┐
│  Platform (GLFW, files, time)               │
├─────────────────────────────────────────────┤
│  Game / ECS (GameWorld, components, demos)  │
├─────────────────────────────────────────────┤
│  Render frontend (SceneSubmit, snapshots)   │
├─────────────────────────────────────────────┤
│  Render backend (VulkanRenderer, passes)    │
└─────────────────────────────────────────────┘
```

| Layer | May include Vulkan headers? | Example |
|-------|----------------------------|---------|
| Platform | No | `Window`, `GlfwInput` |
| Game | No | `CharacterCameraDemo`, `MeshComponent` |
| Render frontend | No | `SceneSubmit`, `SceneRenderParams` |
| Render backend | **Yes** | `VulkanRenderer`, `VulkanScenePipeline` |

**Dependency rule:** upper layers call down; the backend never includes game demo headers. `IEngineContext` is the narrow API surface games use to push `SceneRenderParams` without linking every Vulkan pass.

### One frame through the layers

```text
GLFW poll
  → Engine::Run
      → game->OnUpdate        (Game layer)
      → game->OnRender        (Game builds scene via Submit — frontend)
      → context.SetSceneRenderParams
      → renderer->PresentFrame (VulkanRenderer — backend)
          → DrawFrame → RecordSceneCommandBuffer
```

When you get lost in a 3,000-line file, ask: **which layer am I in?** If gameplay is calling `vkCmdDraw`, architecture has leaked.

## 1b. Concept Deep Dive — How Spark Enforces Layer Boundaries

### Interfaces as compile-time firewalls

Spark's four layers are not just folder names — they are **include-graph constraints**. Open `IGame.hpp` and `VulkanRenderer.hpp` side by side: the game header lists ECS and math types; the renderer header pulls in a dozen pass classes and `<vulkan/vulkan.h>`. If a demo `#include`s the renderer directly, every gameplay `.cpp` recompiles when you touch a shadow pass, and circular dependencies appear within days.

Dependency inversion solves this with two narrow ports: `IEngineContext` (what games call) and `IFramePresenter` (what the loop calls for GPU work). `EngineContext` is a **facade** that knows both `Window` and `IFramePresenter` but exposes neither Vulkan type nor internal pass objects. `VulkanRenderer` implements the presenter and owns composition — device context, frame sync, opaque pass, tonemap — rather than inheriting from game code.

Construction order in `Engine::Impl` mirrors the dependency graph from the bottom up: window first (surface extensions), then renderer (which bootstraps Vulkan internally), then input and context wired to both. No Vulkan calls appear in `Engine.cpp`; the renderer constructor is the single entry that triggers instance/device setup.

### One frame, four layers (includes implied)

```text
  Platform          Game              Frontend                 Backend
  ────────          ────              ────────                 ───────
  Window            IGame::OnUpdate   SceneSubmit              VulkanRenderer
  GlfwInput         components        SceneRenderParams        VulkanDeviceContext
       │                 │                   │                        │
       └─ PollEvents ────┴─ timing ─────────┴─ SetSceneRenderParams ─┴─ DrawFrame
```

| Layer | `#include <vulkan/vulkan.h>`? | Calls into |
|-------|------------------------------|------------|
| Platform | No (GLFW surface ext only at link) | OS / GLFW |
| Game | Never | `IEngineContext` |
| Frontend | Never | `SceneRenderParams`, submit helpers |
| Backend | Yes | `VkDevice`, passes, sync |

### Engine wires subsystems in dependency order

```cpp
// src/Engine.cpp (lines 98–112)
    explicit Impl(UniquePtr<IGame> ownedGame)
        : window{}
        , renderer(MakeUnique<VulkanRenderer>(window))
        , input(MakeUnique<GlfwInput>(window.Handle()))
        , engineContext(MakeUnique<EngineContext>(window, *renderer, *input, audio))
        , game(MoveTemp(ownedGame)) {
        input->WireToWindow(window);
        renderer->GetImGuiLayer().InstallPlatformCallbacks(window);
        engineContext->BindImGuiLayer(renderer->GetImGuiLayer());
        // ...
        window.SetFramebufferResizeCallback([this](int /*width*/, int /*height*/) {
            renderer->NotifySwapchainResize();
        });
    }
```

### Game contract — simulation vs snapshot

```cpp
// include/spark/engine/IGame.hpp
class IGame {
public:
    virtual ~IGame() = default;

    virtual void OnAttach(IEngineContext& context) = 0;
    virtual void OnDetach() = 0;
    virtual void OnUpdate(const FrameTiming& timing, IEngineContext& context) = 0;
    virtual void OnRender(IRenderFrame& frame, IEngineContext& context) = 0;
};
```

### Renderer composes passes; device context is a member

`VulkanRenderer` is not a monolith — it aggregates focused subsystems while delegating bootstrap to `VulkanDeviceContext`:

```cpp
// src/spark/render/core/VulkanRenderer.cpp (lines 43–49)
VulkanRenderer::VulkanRenderer(Window& inAppWindow) : deviceContext(inAppWindow), boundWindow(&inAppWindow) {
    shaderLoader.SetDevice(device());

    CreateCommandPool();
    CreatePersistentSceneResources();
    RecreateSwapchain();
    frameSync.Create(device());
```

When you add a feature later (particles, UI), add a pass object — do not grow a single renderer translation unit without bound.

## 2. Core Concepts

### Dependency inversion

High-level game code depends on **abstractions**, not Vulkan classes:

```text
  IGame ──uses──> IEngineContext ──uses──> IFramePresenter
                      │                         │
                      ▼                         ▼
                 EngineContext            VulkanRenderer
                 (facade)                 (implements)
```

`IGame::OnUpdate` never sees `VkDevice`. `VulkanRenderer` never queries `GameObject` components — it reads `SceneRenderParams` only. Both sides depend on interfaces declared under `include/spark/engine/`. That is the **Dependency Inversion Principle**: stable abstractions sit between volatile implementations.

### Spark folder layout

Spark mirrors the four-layer model in its repository:

| Layer | Spark paths | May include Vulkan? |
|-------|------------|---------------------|
| Platform | `render/platform/Window.hpp`, `engine/GlfwInput.hpp` | Surface extensions only |
| Game | `ecs/`, `scene/`, `demo/*Demo.cpp` | Never |
| Frontend | `engine/SceneRenderParams.hpp`, `scene/SceneSubmit.hpp` | Never |
| Backend | `render/core/`, `render/gpu/`, `render/present/` | Yes |

Rule for your engine: only files under a `render/backend/` tree (Spark uses `render/core/` and `render/gpu/`) may `#include <vulkan/vulkan.h>`.

### Frame phases

Each frame passes through distinct phases with measurable cost:

| Phase | Spark call | Typical work |
|-------|-----------|--------------|
| Poll | `window.PollEvents()` | GLFW/OS events |
| Input | `input->BeginFrame()` | Edge-triggered keys |
| Update | `game->OnUpdate(timing, ctx)` | Simulation |
| Render setup | `game->OnRender(frame, ctx)` | Build `SceneRenderParams` |
| GPU | `renderer->PresentFrame()` → `DrawFrame` | Record + submit + present |

```text
Poll → BeginFrame → OnUpdate → OnRender → PresentFrame
         ↑ simulation only    ↑ snapshot    ↑ all Vk* work
```

Spark records command buffers on the main thread. Vulkan allows worker-thread recording (Part XVI); until then, one thread keeps debugging simple.

### Initialization dependency graph

GPU objects must be created in dependency order:

```text
Window → VkInstance → VkSurfaceKHR → VkPhysicalDevice → VkDevice
    → VkSwapchainKHR → VkRenderPass → VkPipeline → VkCommandBuffer
```

Destroy in **reverse** order. Spark centralizes the left branch in `VulkanDeviceContext`'s constructor and swapchain recreation in `RecreateSwapchain()`. Validation errors like `VUID-vkDestroyDevice-device-00378` usually mean you violated this graph.

### Composition over monolith

`VulkanRenderer` is not one giant class that does everything. It **composes** focused subsystems:

| Subsystem | Responsibility |
|-----------|----------------|
| `VulkanDeviceContext` | Instance, device, swapchain images |
| `VulkanFrameSync` | Fences, semaphores, acquire/present |
| `VulkanPresentRenderPass` | Swapchain + HDR render passes |
| `VulkanSceneOpaquePass` | Lit mesh draws |
| `VulkanDirectionalShadowPass` | Shadow depth maps |

Each subsystem has a narrow public API. When you add a feature (particles, UI), you add a pass object — you do not grow a 10,000-line renderer file.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Game code depends on IEngineContext, not VulkanRenderer

**Concept:** Dependency inversion

**Decision:** Expose `IEngineContext` to `IGame` instead of `VulkanRenderer&`

**Why:** Demos and unit tests can substitute a null presenter. Game headers stay free of GPU types, so compile times for gameplay stay fast when you touch shaders.

**In Spark:** `include/spark/engine/IGame.hpp` — `OnUpdate` and `OnRender` take `IEngineContext&` only.

### EngineContext is a thin facade

**Concept:** Four-layer layout

**Decision:** `EngineContext` forwards `SetSceneRenderParams` to the presenter without game code including `VulkanRenderer.hpp`

**Why:** The facade is the only place that knows both `Window` and `IFramePresenter`. Games call one method; the backend receives the snapshot.

**In Spark:** `src/spark/engine/EngineContext.cpp` — implements `IEngineContext` by delegating to `VulkanRenderer`.

### VulkanRenderer implements IFramePresenter

**Concept:** Frame phases

**Decision:** `PresentFrame()` is the sole GPU entry from `Engine::Run`

**Why:** The loop should not call `DrawFrame` directly. Resize, screenshot, and video capture hooks live on the presenter interface — one stable contract for all demos.

**In Spark:** `VulkanRenderer` inherits `IFramePresenter`; `Engine::Impl` stores `UniquePtr<VulkanRenderer>` and passes it to `EngineContext`.

### DeviceContext separated from Renderer

**Concept:** Initialization dependency graph

**Decision:** `VulkanDeviceContext` owns instance/device/swapchain; `VulkanRenderer` owns passes and command buffers

**Why:** Swapchain recreation touches device-level resources frequently. Isolating `VulkanDeviceContext` keeps recreate logic out of scene-pass code.

**In Spark:** `VulkanRenderer` holds `VulkanDeviceContext deviceContext` as a member; constructor takes `Window&` and builds context first.

### Demos implement IGame, not Engine

**Concept:** Composition

**Decision:** `Engine` is closed; games implement `IGame` or inherit `Game`

**Why:** You never fork `Engine.cpp` for a new demo. `ThreeDDemo`, `SkyDemo`, etc. only override virtual hooks.

**In Spark:** `src/spark/demo/` — each demo is an `IGame` subclass launched from `SparkDemo` main.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book; Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `include/spark/engine/Engine.hpp`
- `src/Engine.cpp`
- `include/spark/engine/IEngineContext.hpp`
- `include/spark/engine/IFramePresenter.hpp`
- `include/spark/render/core/VulkanRenderer.hpp`
- `include/spark/render/core/VulkanDeviceContext.hpp`

### Step 1 — Read Engine construction order

`Engine::Impl` constructs subsystems in dependency order: window first, then renderer (which creates Vulkan), then input and context.

```cpp
// src/Engine.cpp (lines 98–113)
    explicit Impl(UniquePtr<IGame> ownedGame)
        : window{}
        , renderer(MakeUnique<VulkanRenderer>(window))
        , input(MakeUnique<GlfwInput>(window.Handle()))
        , engineContext(MakeUnique<EngineContext>(window, *renderer, *input, audio))
        , game(MoveTemp(ownedGame)) {
        input->WireToWindow(window);
        renderer->GetImGuiLayer().InstallPlatformCallbacks(window);
        engineContext->BindImGuiLayer(renderer->GetImGuiLayer());
        Ui::UiSystem::Get().BindImGuiLayer(&renderer->GetImGuiLayer());
        Ui::UiSystem::Get().RegisterSparkNativeBackend();
        Ui::UiSystem::Get().RegisterDearImguiBackend();
        window.SetFramebufferResizeCallback([this](int /*width*/, int /*height*/) {
            renderer->NotifySwapchainResize();
        });
    }
```

Notice: **no Vulkan calls in `Engine.cpp`**. The renderer constructor triggers device setup internally.

### Step 2 — Study IGame hooks

The game contract is four virtual methods. Simulation and snapshot building are separate on purpose.

```cpp
// include/spark/engine/IGame.hpp
class IGame {
public:
    virtual ~IGame() = default;

    virtual void OnAttach(IEngineContext& context) = 0;
    virtual void OnDetach() = 0;
    virtual void OnUpdate(const FrameTiming& timing, IEngineContext& context) = 0;
    virtual void OnRender(IRenderFrame& frame, IEngineContext& context) = 0;
};
```

**Build-along:** Open any file in `src/spark/demo/` and find where it calls `context.SetSceneRenderParams(...)`. That call belongs in `OnRender`, not `OnUpdate`.

### Step 3 — Trace presenter to DrawFrame

`PresentFrame` is the public name; `DrawFrame` is the internal implementation with acquire, record, submit, present.

```cpp
// src/spark/render/core/VulkanRenderer.cpp (lines 386–388, 398–415)
void VulkanRenderer::PresentFrame() {
    DrawFrame();
}

void VulkanRenderer::DrawFrame() {
    if (framebufferResized) {
        framebufferResized = false;
        RecreateSwapchain();
        return;
    }

    frameSync.WaitForCurrentFrameFence(device());

    const std::uint32_t currentFrame = frameSync.CurrentFrameIndex();
    // ... uploads, acquire image, record commands, submit, present ...
```

Part I focuses on the tail of this function (sync + present). Part II fills the middle (render pass + draw).

### Step 4 — Map VulkanRenderer members

Skim `VulkanRenderer.hpp` and label each member with a subsystem from Section 2:

| Member (conceptual) | Layer |
|---------------------|-------|
| `deviceContext` | Backend — device lifecycle |
| `frameSync` | Backend — per-frame sync |
| `sceneOpaquePass` | Backend — mesh rendering |
| `pendingScene` | Frontend snapshot copy |

You do not need every member on day one. The pattern — **compose passes, don't inherit everything** — is what matters.

### Step 5 — Draw your own layer diagram

On paper, draw four boxes: Platform, Game, Frontend, Backend. Add arrows for one frame:

```text
GlfwInput → IGame::OnUpdate → IGame::OnRender → SceneRenderParams
    → VulkanRenderer::SetSceneRenderParams → DrawFrame → swapchain
```

Keep this diagram visible while reading Parts I–II.

## 5. How This Connects to the Rest of Your Engine

Part **I** chapters 5–10 implement the Backend box bottom-up: window, instance, device, swapchain, sync. Part **II** connects `SceneRenderParams` to `vkCmdDrawIndexed`. Part **VII** shows how ECS feeds the frontend via `SceneSubmit`.

If you feel lost later, return to this chapter's layer diagram and ask: *which box does this file belong in?*

## 6. Pitfalls and Debugging

- **Including `VulkanRenderer.hpp` from demo code** — breaks layering; use `IEngineContext` only.
- **Calling `SetSceneRenderParams` from `OnUpdate`** — couples simulation to render data; hard to test and risks GPU races.
- **Putting gameplay in `Engine::Run`** — fork the engine instead of the demo; `Engine` should stay generic.
- **Monolithic renderer class** — if your renderer file exceeds ~800 lines in Part I, split device context and sync now.

## 7. Exercises

1. List every `#include` in `ThreeDDemo.cpp` (or any demo). Confirm none pull in `<vulkan/vulkan.h>`.
2. Count how many classes in `VulkanRenderer.hpp` are pass objects vs utilities.
3. Add a temporary log in `OnUpdate` and `OnRender` of a demo — verify `OnRender` runs after `OnUpdate` every frame.
4. Find `NotifySwapchainResize` — explain why resize goes through a flag instead of recreating inside the GLFW callback.

## Summary

Spark's architecture is four layers connected by interfaces: platform, game, render frontend, render backend. `Engine` wires them together; `IGame` stays ignorant of Vulkan; `VulkanRenderer` stays ignorant of ECS. Every Part I file you open should fit one box on the diagram.

---

Next: [Building the Reference Project](03-building-spark.md).
