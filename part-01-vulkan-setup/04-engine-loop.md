# The Engine Loop

> **Build-Along Milestone:** You can describe Spark's `Engine::Run` line-by-line and explain why `PresentFrame` comes after `OnRender`.

**You will learn:**

- Implement poll → update → render → present in the correct order.
- Measure delta time and pass `FrameTiming` to gameplay.
- Wire framebuffer resize to deferred swapchain recreation.

## 1. The Problem We Are Solving

The **engine loop** is the heartbeat of every real-time application. Games are not callback-driven like web pages — they run a `while` loop that advances simulation time and submits frames until the user closes the window.

Get the order wrong and you get subtle bugs: input one frame late, GPU reading scene data before it is written, or swapchain images recreated mid-record. Spark's `Engine::Run` is the canonical reference. This chapter walks through it so your mini-engine matches the same shape before Part I adds more Vulkan objects.

Open `src/Engine.cpp` and keep it visible.

## 2. Core Concepts

### Canonical loop shape

Every engine shares the same skeleton:

```text
while (!window.ShouldClose()) {
    PollEvents();
    BeginFrame();          // input edge detection
    OnUpdate(dt);          // simulation — no Vulkan
    OnRender();            // build SceneRenderParams
    PresentFrame();        // GPU + display
}
```

Delta time: `dt = now - previous` using a **steady clock** (`std::chrono::steady_clock`), not `system_clock`. Frame index increments for profiling, replay hooks, and deterministic noise seeds.

### IGame lifecycle

| Hook | When | Must not |
|------|------|----------|
| `OnAttach` | once, before loop | Call Vulkan |
| `OnUpdate` | each frame | Fill final GPU buffers directly |
| `OnRender` | each frame, after update | Run heavy physics unrelated to visuals |
| `OnDetach` | once, after loop | — |

Spark calls `OnAttach` after `EngineContext` and renderer exist, so demos can query drawable size and register input.

### Resize handling

GLFW fires `framebuffer size` callbacks on a different thread on some platforms. Spark sets a **flag** instead of recreating the swapchain inside the callback:

```text
GLFW resize callback → NotifySwapchainResize() → framebufferResized = true
Next DrawFrame()     → RecreateSwapchain() if flag set
```

Recreating inside the callback risks deadlocking with the main thread mid-frame.

### Subsystem hooks in the loop

Spark inserts fixed points for audio and UI without polluting game code:

| Call | Purpose |
|------|---------|
| `audio.PumpFrame(dt)` | Stream audio after simulation |
| `UiSystem::OnEnginePreRender` | UI layout before scene submit |
| `UiSystem::OnEnginePostRender` | UI after game render hook |

Your engine can start without these; add stub calls so later parts slot in cleanly.

### FrameTiming struct

Passed to `OnUpdate`:

```cpp
struct FrameTiming {
    float deltaTimeSeconds;
    float totalTimeSeconds;
    std::uint64_t frameIndex;
};
```

Use `deltaTimeSeconds` for movement (`position += velocity * dt`). Use `totalTimeSeconds` for periodic effects. Do **not** call `glfwGetTime()` inside random components — centralize time in the loop.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Single while-loop in Engine::Run

**Concept:** Canonical loop shape

**Decision:** One loop owns poll, update, render, present

**Why:** Multiple loops (e.g. fixed timestep inner + render outer) desync input sampling and present unless carefully designed. Spark uses variable timestep for simplicity in demos.

**In Spark:** `src/Engine.cpp` — `Impl::Run` while-loop.

### Virtual hooks on IGame

**Concept:** IGame lifecycle

**Decision:** Demos override `IGame`, not `Engine`

**Why:** `Engine` stays closed; new games ship as separate translation units. Tests can inject a mock `IGame`.

**In Spark:** `include/spark/engine/IGame.hpp`.

### Callback → NotifySwapchainResize

**Concept:** Resize handling

**Decision:** Defer recreate to next `DrawFrame`

**Why:** Vulkan requires `vkDeviceWaitIdle` before swapchain destroy; safe point is start of frame after fence wait.

**In Spark:** `src/Engine.cpp` lines 110–112 — resize callback; `VulkanRenderer::DrawFrame` checks `framebufferResized`.

### Present after OnRender

**Concept:** Frame phases

**Decision:** `renderer->PresentFrame()` is last in the iteration

**Why:** `OnRender` must finish writing `SceneRenderParams` before GPU recording reads them.

**In Spark:** `src/Engine.cpp` — `OnRender` then `PresentFrame`.

### Steady clock for dt

**Concept:** FrameTiming

**Decision:** `std::chrono::steady_clock` for delta

**Why:** Monotonic — NTP adjustments do not shrink or explode `dt`.

**In Spark:** `src/Engine.cpp` — `using Clock = std::chrono::steady_clock`.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book; Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `src/Engine.cpp`
- `include/spark/engine/IGame.hpp`
- `include/spark/engine/FrameTiming.hpp`
- `include/spark/render/core/VulkanRenderer.hpp`

### Step 1 — Read the full loop header

Before the `while`, Spark attaches the game and initializes timing state:

```cpp
// src/Engine.cpp (lines 115–130)
    void Run() {
        if (!game) {
            return;
        }
        autoRecordPending = !runOptions.autoRecordPath.IsEmpty();
        EmptyRenderFrame renderFrame;
        if (ISceneProvider* sceneHost = dynamic_cast<ISceneProvider*>(game.Get())) {
            engineContext->SetSceneBinding(&sceneHost->GetScene());
        }
        game->OnAttach(*engineContext);
        (void)audio.Startup();

        using Clock = std::chrono::steady_clock;
        auto previous = Clock::now();
        float totalSeconds = 0.0F;
        std::uint64_t frameIndex = 0;
```

`OnAttach` is the right place to spawn entities, load scenes, and cache `IEngineContext` pointers — not the constructor of your demo class.

### Step 2 — Core iteration body

The essential sequence (trimmed of screenshot/recording shortcuts):

```cpp
// src/Engine.cpp (lines 132–200, abbreviated)
        while (!window.ShouldClose()) {
            window.PollEvents();
            input->BeginFrame();

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
        }
```

**Build-along:** Set a breakpoint on `PresentFrame` and confirm it hits once per vertical blank cycle (with VSync).

### Step 3 — Trace resize wiring

Constructor registers the callback:

```cpp
        window.SetFramebufferResizeCallback([this](int /*width*/, int /*height*/) {
            renderer->NotifySwapchainResize();
        });
```

`DrawFrame` handles it at the top:

```cpp
    if (framebufferResized) {
        framebufferResized = false;
        RecreateSwapchain();
        return;
    }
```

Skipping the early `return` would record commands into destroyed swapchain images.

### Step 4 — Write a minimal loop in your engine

Even before Vulkan, stub:

```cpp
while (!window.ShouldClose()) {
    window.PollEvents();
    float dt = clock.Tick();
    game.OnUpdate(dt);
    game.OnRender();
    renderer.PresentFrame(); // no-op until Part I complete
}
```

Match Spark's order before adding complexity.

### Step 5 — Measure frame time

Log `timing.deltaTimeSeconds` every 60 frames. At 60 FPS, expect ~0.0167. Spikes above 0.033 mean you missed a v-sync period — start profiling `OnUpdate` vs `PresentFrame` early.

## 5. How This Connects to the Rest of Your Engine

Part **I** chapters 5–10 implement what happens inside `PresentFrame`. Part **II** makes `OnRender` fill draws that appear on screen. Part **V** reads input inside `OnUpdate` via `IEngineContext::GetInput()`.

The loop order never changes — only the work inside each hook grows.

## 6. Pitfalls and Debugging

- **Gameplay in GLFW callbacks** — breaks order; queue events and handle in `OnUpdate`.
- **Double `PresentFrame`** — tears or validation errors; exactly one per loop iteration.
- **Zero delta first frame** — guard division in physics if needed; Spark still passes small dt on frame 1.
- **Recreating swapchain in callback** — use deferred flag pattern from Spark.

## 7. Exercises

1. Resize the window during SparkDemo — confirm no crash; explain the two-step flag path.
2. Log `frameIndex` and verify it increments by 1 per frame.
3. Temporarily swap `OnUpdate` and `OnRender` in a local build — describe what breaks visually.
4. Find where `ISceneProvider` is used — which demos inherit it?

## Summary

`Engine::Run` is poll → input → update → render → present, with deferred swapchain recreation and centralized `FrameTiming`. Copy this shape into your engine before learning Vulkan details — the loop is the spine everything else hangs on.

---

Next: [Window Creation with GLFW](05-window-glfw.md).
