# GlfwInput

> **Build-Along Milestone:** `GlfwInput::BeginFrame()` snapshots keyboard and mouse state; edge queries work after `PollEvents` each tick.

**You will learn:**

- Implement `IInput` with GLFW window handles.
- Snapshot previous and current key state for edge detection.
- Support cursor capture with re-centering and raw mouse motion.
- Accumulate scroll wheel delta and map HiDPI cursor coordinates.

## 1. The Problem We Are Solving

`IInput` defines **what** games ask; `GlfwInput` defines **how** GLFW answers. Each frame the backend must:

1. Poll OS events (outside `BeginFrame`, in the engine loop)
2. Copy GLFW state into `curr` arrays
3. Compare `curr` vs `prev` for edges
4. Compute mouse delta — especially tricky with captured cursor

Getting ordering wrong produces sticky keys, double jumps, or camera spikes. This chapter implements the desktop path Spark ships in every demo.

## 2. Core Concepts

### Construction and wiring

```cpp
GlfwInput input(window.Handle());
input.WireToWindow(window);  // scroll callback; cursor callback on non-Apple
```

`WireToWindow` registers GLFW callbacks on Spark's `Window` wrapper. Scroll accumulates into `scrollAccumY` between frames.

### BeginFrame lifecycle

```cpp
void GlfwInput::BeginFrame() {
    scrollDeltaThisFrame = static_cast<float>(scrollAccumY);
    scrollAccumY = 0.0;

    for (int k = 0; k <= GLFW_KEY_LAST; ++k) {
        prev[k] = curr[k];
        curr[k] = (glfwGetKey(window, k) == GLFW_PRESS || glfwGetKey(window, k) == GLFW_REPEAT);
    }
    for (int b = 0; b <= GLFW_MOUSE_BUTTON_LAST; ++b) {
        mousePrev[b] = mouseCurr[b];
        mouseCurr[b] = (glfwGetMouseButton(window, b) == GLFW_PRESS);
    }
    // mouse delta + cursor position...
}
```

| Phase | Responsibility |
|-------|----------------|
| `PollEvents` | GLFW fills OS event queue |
| `BeginFrame` | Snapshot into curr/prev, compute deltas |
| `OnUpdate` | Game reads stable per-frame state |

### Edge query implementation

```cpp
bool GlfwInput::IsKeyPressedThisFrame(int keyCode) const {
    return curr[keyCode] && !prev[keyCode];
}

bool GlfwInput::IsKeyDown(int keyCode) const {
    return curr[keyCode];
}
```

Same pattern for mouse buttons with `mouseCurr` / `mousePrev`.

### Free cursor vs captured cursor

**Free cursor:** delta from cursor position change since last frame (or callback on non-Apple).

**Captured cursor (`GLFW_CURSOR_DISABLED`):**

```cpp
glfwGetCursorPos(window, &mx, &my);
mouseDeltaX = static_cast<float>(mx - lastMouseX);
mouseDeltaY = static_cast<float>(my - lastMouseY);
// re-center cursor to window midpoint
glfwSetCursorPos(window, midX, midY);
lastMouseX = midX;
lastMouseY = midY;
```

Re-centering gives unlimited rotation without hitting screen edges. Small dead zone (`kCapturedMouseDeadZone = 0.12f`) filters jitter when cursor snaps back.

### Raw mouse motion

When supported:

```cpp
if (glfwRawMouseMotionSupported())
    glfwSetInputMode(window, GLFW_RAW_MOUSE_MOTION, GLFW_TRUE);
```

Bypasses OS acceleration — preferred for FPS camera feel on Windows/Linux.

### Platform: Apple vs others

`SPARK_GLFW_USE_CURSOR_CALLBACK` is **0 on Apple**, **1 elsewhere**:

- **Apple:** poll cursor in `BeginFrame` (MoltenVK / Cocoa quirks)
- **Others:** cursor position callback updates delta

Document platform branch when debugging Mac-specific mouse issues.

### Scroll accumulation

Scroll callback adds to `scrollAccumY`. `BeginFrame` moves accumulated value to `scrollDeltaThisFrame` and resets accumulator — one frame's worth of scroll per `GetScrollDeltaY()`.

### GetCursorFramebufferPixels

Maps `glfwGetCursorPos` through window-to-framebuffer scale:

```text
fbX = (windowX - windowOriginX) * (drawableWidth / windowWidth)
```

When `drawableWidth/Height` arguments are zero, uses `glfwGetFramebufferSize` internally. Matches swapchain pixel grid for UI.

### Clipboard

`TryGetClipboardUtf8` / `SetClipboardUtf8` wrap `glfwGetClipboardString` / `glfwSetClipboardString` with UTF-8 `Utf8String` — used by UI text fields (Part XI).

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Full key array snapshot

**Concept:** BeginFrame lifecycle

**Decision:** Loop `0..GLFW_KEY_LAST` every frame

**Why:** Simple, fast enough; edge detection O(1) per query.

**In Spark:** `GlfwInput.cpp` `BeginFrame`.

### REPEAT counts as down

**Concept:** IsKeyDown

**Decision:** `GLFW_PRESS || GLFW_REPEAT`

**Why:** Held key continues firing for text entry; movement stays active.

**In Spark:** Key snapshot loop.

### Capture re-center pattern

**Concept:** Captured cursor

**Decision:** Teleport to window center after reading delta

**Why:** Standard FPS pattern; infinite yaw without cursor hit edge.

**In Spark:** `SetCursorCaptured`, `BeginFrame` captured branch.

### Scroll batch per frame

**Concept:** Scroll accumulation

**Decision:** Accumulate in callback, consume in BeginFrame

**Why:** Multiple scroll events between frames sum correctly.

**In Spark:** `WireToWindow` scroll callback.

### Apple polling path

**Concept:** Platform split

**Decision:** No cursor callback on Apple

**Why:** Avoided historical GLFW/Cocoa issues with hidden cursor.

**In Spark:** `#if SPARK_GLFW_USE_CURSOR_CALLBACK`.

## 4. Implementation Walkthrough

### Spark Files We Touch This Chapter

- `include/spark/engine/GlfwInput.hpp`
- `src/spark/engine/GlfwInput.cpp`
- `src/Engine.cpp` (construction, `WireToWindow`)
- `include/spark/render/platform/Window.hpp`

### Step 1 — Verify loop order

In `Engine::Run`:

```cpp
window.PollEvents();
input->BeginFrame();
game->OnUpdate(timing, *engineContext);
```

Never swap — edges depend on prev from **last** frame's `BeginFrame`.

### Step 2 — Log edge detection

Press Space once — `IsKeyPressedThisFrame` true exactly one frame; `IsKeyDown` true while held.

### Step 3 — Capture test

Toggle F1 in `ThreeDDemo`. Move mouse — delta non-zero. Escape capture — delta behavior changes.

### Step 4 — Scroll test

Log `GetScrollDeltaY` while scrolling UI zoom — values ±1 typical per notch.

### Step 5 — Retina cursor

Compare framebuffer cursor to window coords on MacBook — scale ≈ 2.0.

## 5. How This Connects to the Rest of Your Engine

Chapter **1** defined the interface this class implements. Chapter **3** embeds it in the main loop. Chapter **5** `FlyCamera` reads deltas when captured. UI consumes same input after game update.

`GlfwInput` is the **desktop reference implementation** — port by replacing this class, not gameplay.

## 6. Pitfalls and Debugging

- **All edges false** — `BeginFrame` not called or called before `PollEvents`.
- **Camera jump on capture** — apply look only when `frameIndex > 0` after toggle.
- **Sticky mouse button** — focus loss without `PollEvents` — rare on desktop.
- **Zero scroll** — forgot `WireToWindow` scroll callback.
- **Wrong delta on Mac** — verify polling path vs callback path for your platform define.

## 7. Exercises

1. Add `GetMouseDelta` length log — detect spikes after capture toggle.
2. Implement `IsKeyReleasedThisFrame` (symmetric to press).
3. Measure `BeginFrame` cost with full key loop — still negligible?
4. Trace scroll from callback to zoom in `SceneEditor3DDemo`.
5. What breaks if you call `BeginFrame` twice per frame?

## Summary

`GlfwInput` implements `IInput` by snapshotting GLFW key and mouse state each `BeginFrame` after `PollEvents`, supporting edge queries, captured cursor with re-center and raw motion, per-frame scroll delta, and framebuffer-scaled cursor position. Wire callbacks at init, respect Apple polling vs callback split, and never read GLFW directly from gameplay.

---

Next: [Input in the Loop](03-input-loop.md).
