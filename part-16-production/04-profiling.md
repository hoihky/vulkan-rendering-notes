# Profiling

> **Build-Along Milestone:** ImGui overlay shows FPS and section timings; RenderDoc groups passes by debug label.

**You will learn:**

- Instrument CPU subsystems with scoped timers around submit and record.
- Group GPU passes with debug utils labels for RenderDoc.
- Build ImGui overlay and regression JSON baselines.

## 1. The Problem We Are Solving

Optimization without measurement is folklore. **Profiling** splits CPU time (physics, SceneSubmit, command record) and GPU time (shadow, scene, SSAO, tonemap) so you fix the real bottleneck.

Spark passes `FrameTiming` everywhere and integrates ImGui overlays in demos — you will add labels and timers aligned with the frame graph from Parts IX–X, then save baselines before renderer refactors.

Milestone: F3 overlay shows FPS; one RenderDoc capture with pass labels matching the Part X frame graph.

## 1b. Concept Deep Dive — FrameTiming Instrumentation

### FrameTiming travels with every subsystem tick

```cpp
// include/spark/engine/FrameTiming.hpp
struct FrameTiming {
    float deltaTimeSeconds = 0.0F;
    float totalTimeSeconds = 0.0F;
    std::uint64_t frameIndex = 0;
};
```

Engine computes timing once per frame; `OnUpdate`, physics, editor panels, and demos receive the same struct — consistent denominators for scoped timers and FPS overlays.

### DemoFoundation FPS overlay pattern

Spark demos toggle an ImGui FPS overlay via F3 (`ToggleDemoFpsOverlayVisible`). Wrap expensive sections with scoped timers logged to the overlay:

```text
Frame N:
  physics.Simulate2D     0.4 ms
  FillSceneFromWorld     1.2 ms
  VulkanRenderer::DrawFrame  8.1 ms
```

Save JSON baselines before renderer refactors — compare shadow pass ms after Part IX changes, not gut feel.

### RenderDoc pass grouping

Name command buffer regions to match the Part X frame graph — shadow → opaque → particles → SSAO → tonemap → UI. Without labels, GPU captures show anonymous `vkCmdDrawIndexed` blobs; with labels, you see which pass regressed after a pipeline change.

CPU timers tell you **where** to look; GPU captures tell you **what** to fix inside the renderer.

## 2. Core Concepts

### Measure before optimize

Guessing bottlenecks wastes weeks. Split frame cost:

```text
T_frame ≈ T_physics + T_submit + T_record + T_gpu + T_present
```

Optimize the **largest** term first — disable SSAO when CPU submit dominates, reduce shadow resolution when GPU scene pass dominates. Spark separates physics (Part XIV) from `FillSceneFromWorld` from `RecordSceneCommandBuffer` from GPU passes.

### FrameTiming as shared clock

```cpp
struct FrameTiming {
    float deltaTimeSeconds = 0.0F;
    float totalTimeSeconds = 0.0F;
    std::uint64_t frameIndex = 0;
};
```

`Engine.cpp` constructs `FrameTiming` each tick and passes it to `OnUpdate`, audio `PumpFrame`, and UI — one clock source for simulation and instrumentation.

### CPU scoped timers

Wrap subsystems with scoped timers writing into a frame stats struct:

```cpp
struct FrameStats {
    float simulate2DMs, submitMs, recordMs;
};
```

Log rolling average every 60 frames to reduce spam. Name scopes to match gather function names so flame graphs map to code.

### GPU timestamp queries

`VkQueryPool` `VK_QUERY_TYPE_TIMESTAMP` inserts at pass boundaries in the command buffer. Convert delta ticks to ms using `vkGetPhysicalDeviceProperties` timestamp period.

**Latency:** results read **one frame late** after fence — label this in the overlay. Gate with a cvar `enableGpuTiming` so retail pays zero query overhead.

### Debug utils labels

`VK_EXT_debug_utils` `vkCmdBeginDebugLabelEXT` groups commands in the RenderDoc capture tree — label names match book pass names: Shadow, HDR Scene, SSAO, Tonemap, UI.

Insert at the start of each `Record*Pass` function — no-op when extension missing.

### ImGui stats overlay

`Engine.cpp` toggles demo FPS overlay on F3/F4. Extend with CPU sections, draw counts, and particle totals. Zero shipping cost when developer tools compile out.

### CPU vs GPU bound decision tree

```text
if (submitMs + recordMs > 12 ms && gpuMs < 8 ms) → CPU bound (cull/submit)
else if (gpuSceneMs > 15 ms) → GPU scene (shadows, draws)
else if (gpuPostMs > 5 ms) → post stack
else → present or driver overhead
```

### Regression baselines

Save JSON snapshot per demo camera pose:

```json
{
  "demo": "TimeOfDay",
  "demoIndex": 17,
  "resolution": "1920x1080",
  "quality": "High",
  "cpu_ms": {"submit": 0.82, "record": 1.14},
  "gpu_ms": {"shadow": 2.1, "scene": 4.8, "post": 1.6, "total": 9.2},
  "counts": {"visibleDraws": 142}
}
```

Compare after renderer refactor — 20% regression without feature change signals accidental sync insert or descriptor churn. Include `quality` and `resolution` in every baseline.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### FrameTiming passed to all subsystems

**Concept:** Measure before optimize

**Decision:** Single timing struct from engine loop.

**Why:** Consistent dt for sim, audio, and profiling — no duplicate clocks.

**In Spark:** `include/spark/engine/FrameTiming.hpp` and `Engine.cpp` line 188.

### ScopedTimer → stats fields

**Concept:** CPU scoped timers

**Decision:** RAII scoped timers writing into `FrameStats`.

**Why:** Low overhead pattern encourages wide instrumentation without macro soup.

**In Spark:** Add to game `Update` and `VulkanRenderer` record paths in your fork.

### Optional query pool per renderer

**Concept:** GPU timestamp queries

**Decision:** Disabled by default in retail builds.

**Why:** Mobile TBDR drivers vary timestamp support; retail should pay zero query cost.

**In Spark:** Gate `Record*Pass` timestamp writes behind developer tools.

### Label = pass name

**Concept:** Debug utils labels

**Decision:** RenderDoc tree matches Part IX–X frame graph names.

**Why:** Faster student debugging — pass boundaries visible without guessing.

**In Spark:** Insert in `VulkanRenderer` `Record*` functions.

### JSON per demo milestone

**Concept:** Regression baselines

**Decision:** Store baseline JSON tagged to git commit.

**Why:** Optional CI step — fail build on >15% GPU regression on reference machine.

**In Spark:** Time of Day demo default camera.

### F3 overlay toggle in Engine

**Concept:** ImGui stats overlay

**Decision:** Hotkey toggles overlay without recompile.

**Why:** Demos stay instrumented; retail strips the code path entirely.

**In Spark:** `Engine.cpp` lines 160–162.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `include/spark/engine/FrameTiming.hpp`
- `src/Engine.cpp`
- `src/spark/render/core/VulkanRenderer.cpp`

### Step 1 — Read FrameTiming struct

Minimal today — extend with a stats sidecar in your fork.

*Per-frame timing passed to subsystems*

```cpp
// include/spark/engine/FrameTiming.hpp (lines 7–11)
struct FrameTiming {
    float deltaTimeSeconds = 0.0F;
    float totalTimeSeconds = 0.0F;
    std::uint64_t frameIndex = 0;
};
```

### Step 2 — Trace Engine loop timing construction

*FrameTiming built each tick*

```cpp
// src/Engine.cpp (lines 183–191)
            const auto now = Clock::now();
            const float deltaSeconds = std::chrono::duration<float>(now - previous).count();
            previous = now;
            totalSeconds += deltaSeconds;

            const FrameTiming timing{deltaSeconds, totalSeconds, frameIndex};

            game->OnUpdate(timing, *engineContext);
            audio.PumpFrame(timing.deltaTimeSeconds);
```

### Step 3 — Add ScopedTimer to SceneSubmit

```cpp
ScopedTimer t(&frameStats.submitMs);
FillStandardLitSceneFromWorld(...);
```

Log top 3 sections once per second during TerrainDemo.

### Step 4 — Insert debug labels

Wrap shadow and tonemap record:

```cpp
VkDebugUtilsLabelEXT label{VK_STRUCTURE_TYPE_DEBUG_UTILS_LABEL_EXT};
label.pLabelName = "HDR Scene";
vkCmdBeginDebugLabelEXT(cmd, &label);
// ... passes ...
vkCmdEndDebugLabelEXT(cmd);
```

### Step 5 — Wire F3 overlay toggle

Spark already toggles demo FPS overlay:

```cpp
// src/Engine.cpp (lines 160–162)
            if (input->IsKeyPressedThisFrame(GLFW_KEY_F3) || input->IsKeyPressedThisFrame(GLFW_KEY_F4)) {
                ToggleDemoFpsOverlayVisible();
            }
```

Extend the overlay with `submitMs`, `recordMs`, and visible draw count.

### Step 6 — Enable timestamp queries (optional)

Create query pool `2 * numPasses * framesInFlight`. Write at pass start/end; read after fence completes for the **previous** frame.

### Step 7 — Capture labeled frame in RenderDoc

Run Time of Day demo, capture one frame — expand debug label tree, confirm pass order matches Part X diagram. Capture with validation off for representative GPU timing.

### Step 8 — Save baseline JSON

Store stats after known-good commit — attach to PR when changing `VulkanRenderer` record order.

## 5. How This Connects to the Rest of Your Engine

Part **VIII** particle perf used pooling metrics. Part **XII** cull affects visible draw count in overlay. Part **XIV** physics ms reported separately. Part **XVI** Chapter 3 async work validates via max frame ms.

Combined RenderDoc capture + F3 overlay tells the full story — overlay points to the subsystem; capture confirms draw counts and pass boundaries.

## 6. Pitfalls and Debugging

- **GPU timestamps without timestamp period** — wrong ms values; query properties first.
- **Profiling Debug build only** — opt-in cvar in RelWithDebInfo for representative numbers.
- **Optimizing cold paths** — profile ThreeDDemo + Time of Day, not empty scene.
- **Timer recursion double-count** — one timer per scope level.
- **Reading timestamps same frame** — GPU not finished; wait on fence.
- **Optimizing gpuPostMs when submitMs dominates** — use decision tree before shader work.
- **Baseline capture wrong camera pose** — comparison invalid; document xyz+yaw in notes.

## 7. Exercises

1. Label each F3 overlay field you add using the glossary table in Section 2.
2. Save baseline vs branch RenderDoc pair for demo 17; write `notes.md` diff.
3. Export profiler JSON; identify CPU vs GPU bound for your fork.
4. Toggle SSAO in demo 17 — measure delta `gpuPostMs` and document in baseline JSON.
5. Add scoped timers to physics simulate and compare to submit ms — which dominates in your demo?

## Summary

Profiling combines CPU scoped timers, optional GPU timestamps, debug labels, and ImGui overlay. Spark uses `FrameTiming` as the shared clock for simulation and instrumentation. JSON baselines prevent optimizing the wrong subsystem.

---

Next: [Game Templates](05-shipping.md).
