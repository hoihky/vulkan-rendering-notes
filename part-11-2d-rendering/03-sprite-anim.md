# Sprite Animation

> **Build-Along Milestone:** Character idle loop plays smoothly with correct atlas UVs and no bleeding.

**You will learn:**

- Store flipbook clips as UV rectangles in a uniform-grid texture atlas.
- Advance animation time on CPU in SpriteAnimatorComponent each frame.
- Wire optional Sprite2DCharacterAnimFsmComponent for platformer locomotion clips.

## 1. The Problem We Are Solving

Static sprites suffice for props; characters need **flipbook animation** — cycling atlas cells over time. CPU owns clip selection and frame timing; GPU only receives the current UV bounds in instance data, keeping the same instanced draw path as Chapter 1.

Spark's `SpriteAnimatorComponent` demonstrates uniform-grid atlases; `Sprite2DCharacterAnimFsmComponent` connects clips to 2D physics velocity for platformer feel. Milestone: a 4-frame idle loop on a Kenney atlas with clean edges and jump clip switching on input.
## 2. Core Concepts

### Flipbook clips as UV rectangles

**Sprite animation** does not swap meshes — it changes which **atlas sub-rectangle** the shader samples each frame. Spark stores clips in `SpriteAnimationClip`:

```cpp
struct SpriteAnimationClip {
    std::uint32_t firstFrame, frameCount;
    float framesPerSecond;
    bool loop;
};
```

CPU advances a frame index over time; GPU receives only the current `uvRect` on `SpriteComponent`. The vertex shader remaps unit-quad UVs: `mix(uvMin, uvMax, inQuadUV)` with optional half-texel inset in `sprite.vert`. This separation keeps the render path identical for static and animated sprites — submit always reads `sc->GetUvRect()` regardless of whether an animator wrote it this frame.

Clip firstFrame offsets into atlas grid — idle might start at frame 4 of sheet. frameCount can be less than total grid cells. Non-loop clips hold last frame until SetClipIndex changes.

A clip is a **time range** plus ordered frame indices, not a separate texture. `SpriteAnimationClip` stores start/end frame numbers and frames-per-second; `SpriteAnimatorComponent` converts elapsed time into a frame index and then into a UV rectangle. Non-uniform clips (8-frame run, 2-frame jump) coexist in one atlas because UV math is per-frame, not per-clip texture. When exporting from Aseprite or TexturePacker, align clip boundaries to grid cells so `ComputeUniformGridUv` needs no custom per-frame tables. For one-off cinematic frames outside the grid, use explicit UV rects on `SpriteComponent` without the animator.

Export pipelines should emit **grid-aligned cells** with consistent pixel dimensions so `AddClip` frame ranges map 1:1 to atlas coordinates. When mixing clip lengths in one atlas, pad shorter clips with duplicate last frames rather than leaving empty grid slots that confuse UV math. Runtime clip blending (cross-fade between walk and run) stays outside Spark baseline — swap clip index on FSM transition instead. Memory stays flat because only UV rect changes per frame, not vertex buffers.


### Uniform grid atlas convention

`SpriteAnimatorComponent` assumes a **regular grid** atlas: `columns × rows` cells of equal size. Frame index 0 is **bottom-left**, increasing along +X then +Y — matching many sprite sheet tools when exported with bottom origin:

```cpp
Vector4 ComputeUniformGridUv(columns, rows, linearFrame);
// returns (u0, v0, u1, v1)
```

Kenney platformer sheets in Spark's `Platformer2DDemo` use this layout. Non-uniform frames (variable-width glyph atlases) need a separate clip table of explicit UV rects per frame — the same clip struct works; only the UV lookup function differs. Store normalized 0–1 coords so resolution changes do not break animation.

ComputeUniformGridUv uses bottom-left origin — Tiled top-left exports need Y flip at import. columns/rows zero guard to 1 prevents divide by zero.

Spark's grid origin is **bottom-left** in UV space: frame (0,0) sits at the atlas corner with increasing column index moving +U and row index moving +V. Tools that export top-left origins require a row flip at import — document this once in your asset pipeline README. `SetUniformGrid(cols, rows)` divides `[0,1]²` evenly; non-square cells in the PNG still work if the grid metadata matches pixel layout. Padding between cells in the PNG should exceed GPU linear filtering footprint; combine padding with half-texel inset in `sprite.vert` for crisp pixel art at 1080p without shimmer at 4K.


### Time integration and loop policy

`OnUpdate` accumulates `timeInClipSeconds += deltaTime`. For looping clips, Spark wraps with `fmod(time, clipDuration)`. For one-shots (attack, hurt), time clamps at the last frame until the FSM switches clips:

```cpp
localFrame = min(localFrame, frameCount - 1);  // non-loop
```

Cap delta-time spikes after a debugger pause — otherwise one frame may advance many indices. `SpriteAnimatorComponent` runs at `ComponentUpdatePriority::AnimatorPlayback`, after physics but before render snapshot. The sibling `SpriteComponent` UV is overwritten every update; submit reads the final rect during `OnRender`.

SetClipIndex resets timeInClipSeconds — switching run→jump restarts clip timing intentionally. fmod loop prevents drift accumulation over long run times.

Animation time uses **delta time** from the engine clock, not fixed 60 Hz steps. After a frame hitch, clamp maximum advance per tick to avoid skipping attack frames in combat systems. Loop policy is per-clip: idle loops wrap with modulo; death clips hold on last frame until gameplay resets state. When switching clips, decide whether to preserve `timeInClipSeconds` — jump transitions usually reset to zero for snappy starts; directional run blends may preserve foot cycle phase. Submit reads only the current UV rect; it never sees clip names or FSM state, preserving the snapshot boundary.


### Sprite2DCharacterAnimFsmComponent

Higher-level **locomotion FSM** selects clip indices from gameplay state. `Sprite2DCharacterAnimFsmComponent` reads `Rigidbody2DComponent` velocity (when present) and switches idle vs move clips on `SpriteAnimatorComponent`. Combat overlays (attack, hurt) temporarily override locomotion clips via blackboard ints or one-shot APIs.

Add the FSM **before** `SpriteAnimatorComponent` on the same `GameObject` so `OnUpdate` priority runs the driver first, then the animator advances the selected clip. Platformer demos wire idle/run/jump/attack indices once at load — gameplay never touches UV math directly.

UpdatePriority AnimationDriver runs before AnimatorPlayback — component order on GameObject matters if priorities tie. LocomotionSource HorizontalAbsVelX suits side-scrollers; SpeedSq suits top-down.

The FSM maps **physics signals** — horizontal speed, grounded flag, jump pressed — into clip indices on `SpriteAnimatorComponent`. Platformer2DDemo wires `HorizontalAbsVelX` thresholds so walk/run thresholds feel responsive without mirroring velocity into shader code. Keep FSM transitions in C++ where unit tests can assert clip changes; do not encode state machines in JSON until Part XIII editor tooling exists. When air control modifies velocity mid-jump, the FSM should hysteresis run/walk thresholds to avoid flicker at zero crossing. Facing direction typically flips `TransformComponent` scale X rather than mirroring UVs — separate render concern from animation frame selection.


### Atlas bleeding and half-texel inset

Neighboring atlas pixels bleed when linear filtering samples across cell boundaries. Spark applies half-texel inset in **both** CPU grid UVs and the vertex shader (`kLayerSize = 512` assumption in `sprite.vert`). When importing from Tiled or TexturePacker, inset at **clip build time** rather than per-frame in gameplay.

Symptoms of bleeding: ghost pixels from adjacent frames, shimmer during animation. Fix by shrinking UV bounds inward by `0.5 / atlasWidth` on each side. Verify with an ImGui frame scrubber (Chapter 7) that pauses on each cell boundary.

sprite.vert half-texel inset uses kLayerSize 512 — mismatch with actual atlas size causes inset error; prefer CPU-side inset from real pixel dimensions.

Half-texel inset shrinks the sampled UV rectangle inward by `0.5 / atlasSize` on each edge so linear filtering never fetches neighboring cells. In `sprite.vert`, remapped corners apply inset before passing varyings to the fragment shader. Verify with a debug mip: zoom a single frame until edge colors stabilize. For pixel-art with `NEAREST` filtering, inset still helps when artists leave single-pixel gutters insufficient. Neighboring cells with wildly different hues show bleeding as colored halos — fix in art first, then confirm inset in shader.


### Events and non-looping clips

`IsCurrentClipFinished()` returns true once a non-looping clip reaches its last frame — use this to return the FSM to locomotion after an attack animation. Fire gameplay events (footstep sound, damage frame) on **frame index change**, not on accum overflow loops, or double-triggers occur when dt spikes.

Export clip definitions as JSON or embedded constexpr tables for hot reload in editor builds (Part XIII). The render path stays unchanged: only `SpriteComponent::SetUvRect` mutates per frame.

IsCurrentClipFinished compares time to clipDur — use for attack recovery to idle. Fire gameplay VFX on frame index change edge detect, not every frame while frame holds.

Non-looping clips signal completion via **component callbacks** or gameplay polling of `IsClipFinished()` rather than GPU fence reads. Attack combos chain by listening for clip end events in the same frame before input buffer consumes the next attack. Hold-last-frame clips (death, victory) keep the final UV rect until `SetClipIndex` explicitly changes state. When despawn happens mid-clip, submit simply omits the entity — no GPU teardown per sprite. Looping ambient animations (torches) should use low-frequency clip changes to avoid SSBO churn; only UV rect changes per frame, not pipeline or texture rebinding.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### SpriteAnimatorComponent owns timing

**Concept:** Flipbook clips as UV rectangles

**Decision:** SpriteAnimatorComponent owns timing

**Why:** Animation is simulation — updates in OnUpdate, never in Vulkan record.

**In Spark:** `include/spark/ecs/components/animation/SpriteAnimatorComponent.hpp`.

### ComputeUniformGridUv static helper

**Concept:** Uniform grid atlas convention

**Decision:** ComputeUniformGridUv static helper

**Why:** Pure function testable without GPU — maps linear index to normalized rect.

**In Spark:** `SpriteAnimatorComponent.cpp` lines 11–27.

### fmod for loop, clamp for one-shot

**Concept:** Time integration and loop policy

**Decision:** fmod for loop, clamp for one-shot

**Why:** Predictable playback for combat vs idle without separate clock types.

**In Spark:** `SpriteAnimatorComponent::OnUpdate` lines 89–106.

### Higher UpdatePriority than animator

**Concept:** Sprite2DCharacterAnimFsmComponent

**Decision:** Higher UpdatePriority than animator

**Why:** Driver selects clip index; animator only advances time within that clip.

**In Spark:** `include/spark/ecs/components/animation/Sprite2DCharacterAnimFsmComponent.hpp`.

### Inset in sprite.vert UV remap

**Concept:** Atlas bleeding and half-texel inset

**Decision:** Inset in sprite.vert UV remap

**Why:** GPU-side safety net even if CPU clip import forgot inset.

**In Spark:** `shaders/sprite.vert` lines 25–29.

### IsCurrentClipFinished query

**Concept:** Events and non-looping clips

**Decision:** IsCurrentClipFinished query

**Why:** Gameplay polls completion instead of hard-coded timers that desync from fps.

**In Spark:** `SpriteAnimatorComponent.hpp` line 52.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `include/spark/ecs/components/animation/SpriteAnimatorComponent.hpp`
- `src/spark/ecs/components/animation/SpriteAnimatorComponent.cpp`
- `include/spark/ecs/components/animation/Sprite2DCharacterAnimFsmComponent.hpp`

### Step 1 — Read clip struct and grid API

SpriteAnimationClip and grid UV helper.

*SpriteAnimatorComponent*

```cpp
// include/spark/ecs/components/animation/SpriteAnimatorComponent.hpp (lines 15–55)
struct SpriteAnimationClip {
    std::uint32_t firstFrame = 0;
    std::uint32_t frameCount = 1;
    float framesPerSecond = 8.0F;
    bool loop = true;
};

/**
 * Advances 2D sprite frame time and updates SpriteComponent UV for a uniform grid atlas (cols × rows).
 */
class SpriteAnimatorComponent final : public GameComponent {
public:
    static constexpr ComponentKind TypeKind = ComponentKind::SpriteAnimator;

    [[nodiscard]] ComponentKind Kind() const noexcept override { return TypeKind; }

    [[nodiscard]] int UpdatePriority() const noexcept override {
        return ComponentUpdatePriority::AnimatorPlayback;
    }

    SpriteAnimatorComponent() = default;

    void OnUpdate(const FrameTiming& timing, GameObject& owner, IEngineContext& context) override;

    /** Each cell is 1/(cols) × 1/(rows); frame 0 is bottom-left, increasing along +X then +Y. */
    void SetUniformGrid(std::uint32_t columns, std::uint32_t rows) noexcept;

    void ClearClips() noexcept;
    void AddClip(const SpriteAnimationClip& clip);

    void SetClipIndex(std::uint32_t index) noexcept;
    [[nodiscard]] std::uint32_t GetClipIndex() const noexcept { return currentClipIndex; }
    [[nodiscard]] std::uint32_t GetClipCount() const noexcept {
        return static_cast<std::uint32_t>(clips.GetSize());
    }

    /** For the current clip: when <c>loop == false</c>, true once playback reaches the last frame. */
    [[nodiscard]] bool IsCurrentClipFinished() const noexcept;

    [[nodiscard]] static Vector4 ComputeUniformGridUv(
            std::uint32_t columns, std::uint32_t rows, std::uint32_t linearFrame) noexcept;
```

### Step 2 — Study ComputeUniformGridUv

Bottom-left grid origin.

*ComputeUniformGridUv*

```cpp
// src/spark/ecs/components/animation/SpriteAnimatorComponent.cpp (lines 11–27)
Vector4 SpriteAnimatorComponent::ComputeUniformGridUv(
        std::uint32_t columns, std::uint32_t rows, std::uint32_t linearFrame) noexcept {
    if (columns == 0 || rows == 0) {
        return {0.0F, 0.0F, 1.0F, 1.0F};
    }
    const std::uint32_t cells = columns * rows;
    linearFrame %= cells;
    const std::uint32_t col = linearFrame % columns;
    const std::uint32_t rowFromBottom = linearFrame / columns;
    const float colsF = static_cast<float>(columns);
    const float rowsF = static_cast<float>(rows);
    const float u0 = static_cast<float>(col) / colsF;
    const float u1 = static_cast<float>(col + 1U) / colsF;
    const float v0 = static_cast<float>(rowFromBottom) / rowsF;
    const float v1 = static_cast<float>(rowFromBottom + 1U) / rowsF;
    return {u0, v0, u1, v1};
}
```

### Step 3 — Walk OnUpdate integration

Advance clip; SetUvRect on sprite.

*OnUpdate*

```cpp
// src/spark/ecs/components/animation/SpriteAnimatorComponent.cpp (lines 68–112)
void SpriteAnimatorComponent::OnUpdate(const FrameTiming& timing, GameObject& owner, IEngineContext& /*context*/) {
    SpriteComponent* sprite = owner.GetComponent<SpriteComponent>();
    if (sprite == nullptr) {
        return;
    }
    if (clips.IsEmpty() || currentClipIndex >= clips.GetSize()) {
        return;
    }
    const SpriteAnimationClip& clip = clips[currentClipIndex];
    if (clip.frameCount == 0U) {
        return;
    }
    const std::uint32_t cells = columns * rows;
    if (cells == 0U) {
        return;
    }

    const float fps = (clip.framesPerSecond > 1.0e-6F) ? clip.framesPerSecond : 8.0F;
    const float frameDur = 1.0F / fps;
    const float clipDur = frameDur * static_cast<float>(clip.frameCount);

    timeInClipSeconds += timing.deltaTimeSeconds;
    if (clip.loop) {
        if (clipDur > 1.0e-6F) {
            timeInClipSeconds = std::fmod(timeInClipSeconds, clipDur);
        }
    } else {
        timeInClipSeconds = std::min(timeInClipSeconds, std::max(clipDur - 1.0e-4F, 0.0F));
    }

    std::uint32_t localFrame = 0U;
    if (frameDur > 1.0e-8F) {
        localFrame = static_cast<std::uint32_t>(timeInClipSeconds / frameDur);
    }
    if (!clip.loop) {
        localFrame = std::min(localFrame, clip.frameCount - 1U);
    } else {
        localFrame %= clip.frameCount;
    }

    std::uint32_t linear = clip.firstFrame + localFrame;
    linear %= cells;

    sprite->SetUvRect(ComputeUniformGridUv(columns, rows, linear));
}
```

### Step 4 — Define a 4-frame idle clip

SetUniformGrid(4,1); AddClip loop.

### Step 5 — Wire FSM for run/jump

Sprite2DCharacterAnimFsmComponent clip indices.

### Step 6 — ImGui frame scrubber

Debug panel for frame index and UV rect.

### Step 7 — Switch run to jump on input

SetClipIndex without resetting transform.

### Step 8 — Milestone — idle loop without bleeding

Smooth loop; no atlas bleed.

## 5. How This Connects to the Rest of Your Engine

Animation feeds **sprite submit** (Chapter 1) via `SpriteComponent::uvRect`. **2D lighting** (Chapter 2) applies independently of frame index. **Tilemaps** (Chapter 4) animate via `TilemapTileAnimatorComponent` instead of flipbooks. Part **XIV** physics supplies velocity to the anim FSM.

**Frame graph reminder:** simulation and component updates happen in `OnUpdate`; snapshot fill in `OnRender` or dedicated submit; GPU work in `PresentFrame`. No chapter in Part XI or XII introduces new Vulkan calls from gameplay — only new arrays on the snapshot and record slots in `VulkanRenderer`. Cross-reference Part X HDR ordering whenever you add a draw type: world content belongs in the HDR subpass; display UI belongs after tonemap.

Animation UV updates happen in `OnUpdate` before submit reads `SpriteComponent::uvRect`. Physics (Part XIV) may drive FSM transitions; render remains unaware. Tile animation (Chapter 4) uses cell ID changes instead of flipbooks — pick one system per moving object type.

## 6. Pitfalls and Debugging

- **Atlas bleeding** — inset UVs by half texel; verify in sprite.vert remap.
- **Delta time spikes** — clamp max advance steps after hitch; cap `timeInClipSeconds` jump.
- **Wrong grid dimensions** — `SetUniformGrid(cols, rows)` must match PNG layout or frames slide diagonally.
- **FSM after animator in component order** — driver must run first; check UpdatePriority.
- **Event on accum overflow** — fire footstep on frame index change, not while loop wraps same frame twice.

**Debugging workflow:** when visuals wrong, capture RenderDoc and list (1) pass name, (2) attachment format, (3) bound pipeline, (4) push/SSBO bytes for one failing draw. Compare with Spark capture on the same demo. Validation errors include VUID — search Khronos spec. CPU-side logging of snapshot array sizes (`sprites`, `tilemaps`, `draws`) often finds empty submit before GPU debugging.

When frames skip after breakpoint, clamp delta advance to avoid finishing entire clip in one frame. Verify grid dimensions against atlas PNG in image editor overlay. FSM clip index out of range falls back to idle in well-tested demos — assert in your engine during development.

## 7. Exercises

1. Build 4-frame idle loop from one horizontal strip PNG (4×1 grid).
2. Switch run→jump on spacebar; confirm non-loop clip holds last frame until landing FSM.
3. Export clip JSON and reload at runtime without restarting the demo.
4. Log UV rect each frame — verify monotonic cell changes at 8 fps within 1% tolerance.
5. Document `SpriteAnimatorComponent` update order vs `TransformComponent`.

## Summary

Sprite animation selects atlas UV rectangles over time on CPU via SpriteAnimatorComponent and optional 2D anim FSM, passing current rect to submit without changing the GPU instancing path.

---

Next: [Tilemaps](04-tilemaps.md).
