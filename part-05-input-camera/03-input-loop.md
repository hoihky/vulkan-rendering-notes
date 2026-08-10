# Input in the Loop

> **Build-Along Milestone:** You can diagram Spark's frame order from `PollEvents` through `OnUpdate` to `PresentFrame` and locate where input is snapshotted.

**You will learn:**

- Place input polling and `BeginFrame` correctly in the main loop.
- Pass `IEngineContext` into game update and render hooks.
- Handle engine-level hotkeys before game logic.
- Integrate UI input after simulation, before render.

## 1. The Problem We Are Solving

Input is not a background service — it is **synchronized to frames**. If you read keys before polling events, you see stale state. If UI reads input before the game consumes capture, focus fights erupt. If render runs before update, you display last frame's world with this frame's camera.

Spark's `Engine::Run` loop fixes a canonical order used across all demos. This chapter is the **temporal contract** every camera and controller in Part V assumes.

## 2. Core Concepts

### One frame timeline

```text
┌─────────────────────────────────────────────────────────────────┐
│ 1. PollEvents()          ← OS → GLFW queue                      │
│ 2. input.BeginFrame()    ← snapshot curr/prev, mouse delta      │
│ 3. Engine hotkeys        ← F12 screenshot, F3/F4 FPS overlay    │
│ 4. game->OnUpdate()      ← simulation, cameras, gameplay      │
│ 5. audio.PumpFrame()                                                 │
│ 6. UiSystem::OnEnginePreRender()  ← UI input + layout           │
│ 7. game->OnRender()      ← build SceneRenderParams              │
│ 8. UiSystem::OnEnginePostRender()                                 │
│ 9. renderer->PresentFrame()                                       │
└─────────────────────────────────────────────────────────────────┘
```

Input for gameplay is read in step 4. UI may read again in step 6 with updated focus state.

### Engine construction wiring

```cpp
Impl::Impl(UniquePtr<IGame> ownedGame)
    : window{}
    , renderer(MakeUnique<VulkanRenderer>(window))
    , input(MakeUnique<GlfwInput>(window.Handle()))
    , engineContext(MakeUnique<EngineContext>(window, *renderer, *input, audio))
    , game(MoveTemp(ownedGame)) {
    input->WireToWindow(window);
}
```

`EngineContext` holds references to window, renderer, **input**, and audio. Lifetime: input dies with engine impl.

### FrameTiming passed to game

```cpp
struct FrameTiming {
    float deltaTimeSeconds;
    float totalTimeSeconds;
    std::uint64_t frameIndex;
};
```

Cameras multiply movement by `deltaTimeSeconds`. Mouse look often **ignores** dt (raw sensitivity per pixel) but may scale for consistency.

`frameIndex > 0` guard avoids first-frame mouse spike after capture toggle — demos check before `AddLook`.

### IGame contract

```cpp
class IGame {
public:
    virtual void OnUpdate(const FrameTiming& timing, IEngineContext& context) = 0;
    virtual void OnRender(const RenderFrame& frame, IEngineContext& context) = 0;
};
```

- **OnUpdate** — input, physics, ECS ticks, camera integration
- **OnRender** — fill `SceneRenderParams`, no simulation

Split keeps simulation deterministic relative to input snapshot for that frame.

### Engine hotkeys (before game)

`Engine.cpp` handles global keys:

| Key | Action |
|-----|--------|
| F12 | Screenshot |
| F3 / F4 | FPS overlay toggle |
| F9 | Video recording toggle |

Processed with `IsKeyPressedThisFrame` after `BeginFrame`, before `OnUpdate` — games cannot accidentally consume these if engine checks first (same input, order matters).

### UI input pass

`UiSystem::OnEnginePreRender(window, input, dt)` runs **after** game update so:

- Game owns capture for FPS modes first
- UI can still process clicks when capture off
- Layout uses latest window size

Document for chapters combining editor UI + fly camera.

### Delta time source

```cpp
const double now = glfwGetTime();
const float deltaSeconds = static_cast<float>(now - lastTime);
lastTime = now;
```

Clamp extreme dt on debugger breakpoints — optional production hardening not in all demos.

### ShouldClose and exit

Loop condition `!window.ShouldClose()` — ESC or window close sets flag via GLFW callback. Game need not poll ESC unless menu requires it.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Poll then BeginFrame strictly

**Concept:** Frame timeline

**Decision:** Never interleave; documented in FlyCamera header

**Why:** Edge detection and mouse delta depend on fresh poll.

**In Spark:** `Engine.cpp` while loop top.

### Update before render

**Concept:** IGame contract

**Decision:** `OnUpdate` then `OnRender` same frame

**Why:** Render snapshot reflects post-simulation transforms and camera.

**In Spark:** All demos.

### UI after game update

**Concept:** UI input pass

**Decision:** `OnEnginePreRender` between update and render

**Why:** Gameplay capture priority; UI still receives events when uncaptured.

**In Spark:** `UiSystem.cpp` hook.

### Context bundles input

**Concept:** Engine construction

**Decision:** `EngineContext` exposes `GetInput()`

**Why:** Single injection point for games; no global.

**In Spark:** `EngineContext.hpp`.

### frameIndex for first-frame guards

**Concept:** FrameTiming

**Decision:** Monotonic counter from loop start

**Why:** Skip spurious mouse delta on capture enable frame.

**In Spark:** `ThreeDDemo`, `CharacterCameraDemo`.

## 4. Implementation Walkthrough

### Spark Files We Touch This Chapter

- `src/Engine.cpp`
- `include/spark/engine/Engine.hpp`
- `include/spark/engine/IGame.hpp`
- `include/spark/engine/FrameTiming.hpp`
- `src/spark/ui/runtime/UiSystem.cpp`

### Step 1 — Trace one iteration

Breakpoint at `PollEvents`, `BeginFrame`, `OnUpdate`, `PresentFrame`. Single-step one frame.

### Step 2 — Break order experiment

Temporarily call `BeginFrame` before `PollEvents` — observe one-frame input lag.

### Step 3 — Measure update vs render time

Log durations — input/camera should live in update, not render.

### Step 4 — Dual consumer test

Enable FPS capture + UI overlay — verify who receives mouse when capture toggles.

### Step 5 — Empty game stub

Minimal `IGame` that logs `IsKeyDown(W)` — confirms loop wiring without Vulkan complexity.

## 5. How This Connects to the Rest of Your Engine

Chapters **5–9** run inside `OnUpdate`. Chapter **10** writes camera matrices in `OnRender` or submit helper. Part **I** frame sync (`PresentFrame`) completes the loop after render.

The loop order is **infrastructure** — changing it breaks every camera chapter.

## 6. Pitfalls and Debugging

- **Input lag one frame** — BeginFrame before PollEvents.
- **Simulation in OnRender** — violates contract; camera stutters with vsync.
- **Double update** — calling game update from render path.
- **Dt explosion** — breakpoint pause; clamp delta.
- **UI steals keys** — focus order; check when UI pre-render runs.

## 7. Exercises

1. Draw ASCII timeline of Spark frame with your game's extra systems added.
2. Add engine hotkey F5 to reload shaders — where in loop should it go?
3. Why is audio `PumpFrame` after update, not before?
4. Implement frame time graph using `FrameTiming` only in update.
5. Compare single-threaded loop vs input thread — pros/cons paragraph.

## Summary

Spark's main loop polls GLFW events, snapshots input with `BeginFrame`, runs engine hotkeys, updates the game with `FrameTiming` and `IEngineContext`, pumps audio, processes UI input, renders, and presents. Games read input only in `OnUpdate` after `BeginFrame`. Preserve this order when adding cameras, physics, or networking.

---

Next: [Camera Math](04-camera-math.md).
