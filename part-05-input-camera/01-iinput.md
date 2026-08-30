# Input Abstraction

> **Build-Along Milestone:** Gameplay code queries `IInput&` through `IEngineContext::GetInput()` without including GLFW headers.

**You will learn:**

- Define a platform-agnostic input interface for keyboard and mouse.
- Distinguish held keys from edge-triggered presses and releases.
- Inject input through engine context for testability and porting.
- Map cursor coordinates to framebuffer pixels for HiDPI correctness.

## 1. The Problem We Are Solving

Camera controllers, character movement, and UI all need to know what the player pressed this frame. Calling `glfwGetKey` directly from every system couples gameplay to GLFW, blocks headless unit tests, and makes console ports painful.

Spark inverts the dependency with **`IInput`** — a small virtual interface games depend on. Desktop builds use **`GlfwInput`**; tests can stub keys; future platforms swap implementations without touching `FlyCamera` or demos.

This chapter defines the contract: what queries exist, what "this frame" means, and how context exposes input to `IGame::OnUpdate`.

### Input abstraction for beginners

**Problem:** if every system calls GLFW directly, you cannot:

- Run headless tests that fake "W pressed"
- Port to console without rewriting every camera
- Swap input backends (SDL, raw X11) without touching gameplay

**Solution:** `IInput` is a thin **interface** — gameplay asks questions, platform code answers:

| Query | Meaning |
|-------|---------|
| `IsKeyDown(Key::W)` | W is held **right now** |
| `WasKeyPressed(Key::Space)` | W went down **this frame only** (edge) |
| `GetMouseDelta()` | Pixel movement since last frame |
| `GetCursorPosition()` | Mouse in framebuffer pixels |

**Edge vs held:** jump on `WasKeyPressed(Space)` — if you use `IsKeyDown`, holding space triggers jump every frame.

```text
Player code:
  if (input.WasKeyPressed(Key::Space)) velocity.y = jumpSpeed;

Engine provides:
  IInput& input = ctx.GetInput();  // GlfwInput on desktop
```

You never `#include <GLFW/glfw3.h>` in `FlyCamera` — only in `GlfwInput.cpp`.

## 1b. Concept Deep Dive — IInput Virtual Contract

### Minimal surface area for gameplay

```cpp
// include/spark/engine/IInput.hpp
class IInput {
public:
    virtual ~IInput() = default;
    [[nodiscard]] virtual bool IsKeyDown(int keyCode) const = 0;
    [[nodiscard]] virtual bool IsKeyPressedThisFrame(int keyCode) const = 0;
    [[nodiscard]] virtual float GetMouseDeltaX() const { return 0.0F; }
    [[nodiscard]] virtual float GetMouseDeltaY() const { return 0.0F; }
    virtual void SetCursorCaptured(bool capture) { (void)capture; }
    virtual void GetCursorFramebufferPixels(float& outX, float& outY, int w, int h) const;
};
```

Key codes are **GLFW values** when using `GlfwInput` — `FlyCamera.cpp` includes `GLFW/glfw3.h` only for `GLFW_KEY_W` constants, not for polling. A headless test stub can hard-code `IsKeyDown` return values without linking GLFW.

### Edge queries require BeginFrame

`IsKeyPressedThisFrame` compares current vs previous snapshot — meaningless until the platform backend runs `BeginFrame()` after `PollEvents`. Documented order in Chapter 3; violating it produces sticky keys or missed first presses.

### HiDPI cursor in drawable pixels

`GetCursorFramebufferPixels` accepts swapchain width/height so picking and UI align with Vulkan framebuffer pixels on Retina displays — window-point coordinates would miss by 2× on scaled monitors.

### Engine context injection

`IEngineContext::GetInput()` returns the same `IInput&` for the frame. Demos take `IInput& input = ctx.GetInput()` in `OnUpdate` — no global singleton. UI (`UiInputRouter`) and gameplay share the reference but UI reads input later in the frame (after game update).

## 2. Core Concepts

### Adapter pattern for input

```text
OS / GLFW  →  GlfwInput (concrete)  →  IInput (interface)  →  FlyCamera, UI, demos
```

Gameplay includes `spark/engine/IInput.hpp` only — never `GLFW/glfw3.h`. Key codes are documented as GLFW values when using `GlfwInput`, but the interface itself is integer codes with no GLFW type in the signature.

### Level vs edge queries

| Query | True when | Use case |
|-------|-----------|----------|
| `IsKeyDown(code)` | Key held this frame | Movement (WASD), sprint |
| `IsKeyPressedThisFrame(code)` | Down now, up last frame | Toggle capture, one-shot actions |
| `IsMouseButtonDown(btn)` | Button held | Aim, drag |
| `IsMouseButtonPressedThisFrame(btn)` | Click start | Fire weapon once |
| `IsMouseButtonReleasedThisFrame(btn)` | Click end | Release drag |

**Level** = state. **Edge** = transition. Jump on press, not while held — edge query. Move while W held — level query.

Edge detection requires the backend to snapshot previous frame state in `BeginFrame()` (Chapter 2).

### IInput surface area

```cpp
class IInput {
public:
    virtual bool IsKeyDown(int keyCode) const = 0;
    virtual bool IsKeyPressedThisFrame(int keyCode) const = 0;

    virtual float GetMouseDeltaX() const { return 0.0F; }
    virtual float GetMouseDeltaY() const { return 0.0F; }
    virtual bool IsMouseButtonDown(int button) const { return false; }
    virtual bool IsMouseButtonPressedThisFrame(int button) const { return false; }
    virtual bool IsMouseButtonReleasedThisFrame(int button) const { return false; }
    virtual float GetScrollDeltaY() const { return 0.0F; }

    virtual void GetCursorFramebufferPixels(float& outX, float& outY,
        int drawableWidth = 0, int drawableHeight = 0) const;

    virtual void SetCursorCaptured(bool capture);
    virtual bool IsCursorCaptured() const { return false; }

    virtual bool TryGetClipboardUtf8(Utf8String& out) const { return false; }
    virtual void SetClipboardUtf8(const Utf8String& text) const;
};
```

Pure virtual only for keyboard — mouse, scroll, clipboard have **default no-op** implementations so minimal stubs compile.

### Injection via IEngineContext

```cpp
class IEngineContext {
public:
    virtual IInput& GetInput() = 0;
    virtual void GetFramebufferSize(int& w, int& h) const = 0;
    // presenter, audio, scene params...
};
```

Demos receive context in `OnUpdate`:

```cpp
void ThreeDDemo::Simulate(const FrameTiming& timing, IEngineContext& context) {
    IInput& in = context.GetInput();
    if (in.IsKeyPressedThisFrame(GLFW_KEY_F1)) { /* toggle capture */ }
}
```

`EngineContext` wires real `GlfwInput` at startup. Tests inject a fake `IInput` subclass.

### Cursor in framebuffer pixels

`GetCursorFramebufferPixels` returns drawable pixel coordinates — top-left origin, Y down — matching swapchain extent. On Retina displays, window coordinates ≠ framebuffer pixels. Pass `drawableWidth/Height` from `GetFramebufferSize` for UI hit-testing and picking aligned with rendered image.

### Cursor capture for FPS look

`SetCursorCaptured(true)` hides cursor and enables relative mouse motion (implementation in `GlfwInput`). Games check `IsCursorCaptured()` before applying `GetMouseDeltaX/Y` — editor modes may use free cursor when capture is off.

### Key code convention

When using GLFW backend, key codes are `GLFW_KEY_*` constants (e.g. `GLFW_KEY_W = 87`). Document this in your game layer — do not magic-number unless you wrap constants in an enum.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Virtual interface, not callbacks

**Concept:** Adapter pattern

**Decision:** Poll-based `IInput` queries each frame

**Why:** Matches Spark's single-threaded update loop; simpler than event queues for teaching engine.

**In Spark:** `IInput.hpp`; games poll in `OnUpdate`.

### Defaults for optional features

**Concept:** IInput surface

**Decision:** Mouse/scroll/clipboard default to zero/false

**Why:** Headless test doubles implement only `IsKeyDown` / `IsKeyPressedThisFrame`.

**In Spark:** Inline defaults in `IInput.hpp`.

### Context facade, not global input

**Concept:** Injection via context

**Decision:** `context.GetInput()` not singleton

**Why:** Multiple engine instances in tests; explicit dependency in `OnUpdate` signature.

**In Spark:** `IEngineContext`, `EngineContext.hpp`.

### Framebuffer pixel cursor API

**Concept:** HiDPI cursor

**Decision:** Separate from raw GLFW window coords

**Why:** UI and Vulkan viewport use drawable size; mismatches cause offset clicks on Mac.

**In Spark:** `GetCursorFramebufferPixels` doc comment.

### GLFW codes as lingua franca

**Concept:** Key code convention

**Decision:** Document GLFW values for desktop; no parallel enum yet

**Why:** Zero translation layer in `GlfwInput`; demos copy GLFW constants.

**In Spark:** `IInput.hpp` header comment.

## 4. Implementation Walkthrough

### Spark Files We Touch This Chapter

- `include/spark/engine/IInput.hpp`
- `include/spark/engine/IEngineContext.hpp`
- `include/spark/engine/EngineContext.hpp`
- `src/spark/demo/ThreeDDemo.cpp` (consumer example)

### Step 1 — Define a test double

```cpp
class StubInput : public IInput {
    bool wDown = false;
public:
    bool IsKeyDown(int code) const override { return code == GLFW_KEY_W && wDown; }
    bool IsKeyPressedThisFrame(int) const override { return false; }
    void SetW(bool v) { wDown = v; }
};
```

Unit-test `FlyCamera::ProcessMovement` without a window.

### Step 2 — Trace demo usage

Open `ThreeDDemo::Simulate` — list every `IInput` call. Categorize level vs edge.

### Step 3 — Framebuffer cursor

Log `GetCursorFramebufferPixels` vs raw GLFW `glfwGetCursorPos` on Retina — observe scale factor.

### Step 4 — Capture toggle

F1 toggles `SetCursorCaptured` in demo — note mouse delta only meaningful when captured.

### Step 5 — Avoid GLFW in camera module

Confirm `FlyCamera.hpp` includes only `IInput.hpp`, not GLFW.

## 5. How This Connects to the Rest of Your Engine

Chapter **2** implements `GlfwInput`. Chapter **3** orders `PollEvents` and `BeginFrame`. Chapters **5–7** consume input in camera controllers. UI system (Part XI) shares the same `IInput&` after game update.

Input abstraction is the **boundary** between OS and simulation.

## 6. Pitfalls and Debugging

- **Edge never fires** — forgot `BeginFrame` snapshot (Chapter 2).
- **Held key stutters** — using `PressedThisFrame` for movement instead of `IsKeyDown`.
- **Wrong key constant** — ASCII `'W'` vs `GLFW_KEY_W`.
- **UI clicks offset** — cursor in window coords instead of framebuffer pixels.
- **Global GLFW in gameplay** — breaks headless CI; route through `IInput`.

## 7. Exercises

1. Implement `RecordingInput` that wraps `IInput` and logs all edge events.
2. List which `IInput` methods `FlyCamera` uses (grep Spark).
3. Design a gamepad extension to `IInput` without breaking existing games.
4. Why are mouse methods optional with defaults but keyboard pure virtual?
5. Write a one-paragraph argument for poll vs event-driven input in game loops.

## Summary

`IInput` decouples gameplay from GLFW with level and edge queries for keys and mouse, optional scroll and clipboard, and framebuffer-aware cursor position. Games access it through `IEngineContext::GetInput()`. Implement `GlfwInput` for desktop and stubs for tests — never call the window API from camera or ECS code.

---

Next: [GlfwInput](02-glfw-input.md).
