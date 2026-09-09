---
name: pixelroot32-camera2d
description: Camera2D viewport management, follow-target scrolling, bounds clamping, and camera effects (shake, punch, offset). Use when implementing camera logic, parallax scrolling, screen shake, or camera transitions in PixelRoot32 games.
license: MIT
compatibility: opencode>=0.1.0
metadata:
  domain: engine
  subsystem: graphics
  module: camera2d
  platform: cross-platform
  engine_version: "1.9.0+unreleased"
  # No single feature_gate: the subsystem is only partially gated. Camera2D is
  # always compiled; CameraEffectsSystem is behind PIXELROOT32_ENABLE_CAMERA_EFFECTS
  # and CameraTween behind PIXELROOT32_ENABLE_CAMERA_TWEEN. Each Key API entry
  # states its own gate.
---

## Overview

PixelRoot32 provides a 2D camera system via `Camera2D` and `CameraEffectsSystem`. The camera manages the viewable area with bounds clamping and smooth follow. Effects (shake, punch, offset) are composed on top via a separate system.

## Key APIs

### Camera2D

**Header**: `include/graphics/Camera2D.h`
**Namespace**: `pixelroot32::graphics`
**Feature gate**: none — `Camera2D` is always compiled

The public surface is exactly these 12 members. Nothing else exists: there is no
`update()`, no `setSmoothing()`, no `hasMoved()`, and no viewport getters.

```cpp
Camera2D(int viewportWidth, int viewportHeight);

// Bounds (unbounded if min > max on that axis)
void setBounds(math::Scalar minX, math::Scalar maxX);          // Horizontal
void setVerticalBounds(math::Scalar minY, math::Scalar maxY);  // Vertical

// Position
void setPosition(math::Vector2 position);
void followTarget(math::Scalar targetX);   // Horizontal only
void followTarget(math::Vector2 target);   // Both axes

// Queries
math::Scalar  getX() const;
math::Scalar  getY() const;
math::Vector2 getPosition() const;

// Apply to renderer
void apply(Renderer& renderer) const;                                   // Without effects
void apply(Renderer& renderer, const math::Vector2& effectOffset) const; // With effects

// Viewport
void setViewportSize(int width, int height);
```

**Projected (isometric / non-orthogonal) maps**: derive the clamp range from the
map's screen extents with `graphics::cameraRangeFor(...)` and feed the resulting
`CameraBounds` into `setBounds` / `setVerticalBounds`. That helper, `ScreenBounds`,
and the projection math live behind `PIXELROOT32_ENABLE_TILEMAP_PROJECTION` — see
the **`pixelroot32-projection`** skill.

### CameraTween

**Header**: `include/graphics/CameraTween.h`
**Namespace**: `pixelroot32::graphics`
**Feature gate**: `PIXELROOT32_ENABLE_CAMERA_TWEEN` (default **0**; mirror `pixelroot32::platforms::config::EnableCameraTween`)

```cpp
// Declared OUTSIDE the feature guard — available in both flag states.
enum class TweenEasing : uint8_t { Linear = 0, EaseInQuad, EaseOutQuad, EaseInOutQuad };

template <uint8_t N = 4>   // N = max simultaneously active tweens
class CameraTween {
    static constexpr uint8_t kMaxTweens     = N;
    static constexpr uint8_t kInvalidSlotId = 0xFF;

    CameraTween(const CameraTween&)            = delete;  // non-copyable
    CameraTween& operator=(const CameraTween&) = delete;

    uint8_t startTween(pixelroot32::math::Vector2 from,
                       pixelroot32::math::Vector2 to,
                       uint16_t durationMs,
                       TweenEasing easing);
    void    update(uint16_t deltaTimeMs, Camera2D* camera);  // pointer, not reference
    bool    isComplete(uint8_t slotId) const;
    uint8_t activeCount() const;
    void    cancel(uint8_t slotId);
};
```

- `from` / `to` are `math::Vector2` **by value**; durations are `uint16_t` ms; slot
  ids are `uint8_t`. **No parameter is `math::Scalar`.**
- `startTween` returns a slot id, or `kInvalidSlotId` (`0xFF`) when no slot is free.
- With `N = 0`, `startTween` always returns `kInvalidSlotId` and `activeCount()` is
  always 0.

### CameraEffectsSystem

**Header**: `include/graphics/CameraEffects.h`
**Namespace**: `pixelroot32::graphics`
**Feature gate**: `PIXELROOT32_ENABLE_CAMERA_EFFECTS`

```cpp
CameraEffectsSystem effects;

// Trigger effects (4 slots, round-robin)
effects.triggerShake(amplitude, durationMs);
effects.triggerPunch(amplitude, durationMs, direction);
effects.triggerOffset(amplitude, durationMs);

// Update and apply
effects.update(deltaTimeMs);
if (effects.hasActiveEffects()) {
    Vector2 offset = effects.getOffset();
    camera.apply(renderer, offset);
}

effects.cancelAll();
```

## Composition Patterns

```
Scene::update(dt)
  ├── target->update(dt)        // Move player/entity
  ├── camera.followTarget(target) // Track target
  ├── effects.update(dt)         // Advance effects
  └── camera.apply(renderer, effects.getOffset()) // Push transform
```

## Effect Types

| Type | Behavior | Use Case |
|------|----------|----------|
| **Shake** | Random oscillation via Xorshift32 | Explosions, impacts |
| **Punch** | Directional impulse with linear decay | Hit feedback, dash |
| **Offset** | Constant displacement for duration | Cutscene pans |

## ESP32 Constraints

- **No float in effects**: Shake uses `Xorshift32` (pure integer random). Never replace with `rand()` or float-based RNG.
- **Zero heap**: `std::array<EffectSlot, 4>` — fixed 4 slots, no allocation.
- **IRAM_ATTR**: Consider marking `computeShakeOffset()` if called in tight loops.

## Gotchas

1. Camera coordinates are **negated** for Renderer offset — content scrolls "under" the camera.
2. Bounds are per-axis; unset bounds (min > max) mean unclamped.
3. Effects use **round-robin** slot allocation — if all 4 slots are active, the oldest is overwritten.
4. `CameraEffectsSystem` is a **stub** when `PIXELROOT32_ENABLE_CAMERA_EFFECTS=0` — all methods are no-ops.
5. **`CameraTween::isComplete()` is asymmetric across the flag.** The flag-off stub
   returns **`true`** for any slot id; the live implementation returns **`false`**
   for a slot that was never started. Code that loops "until complete" behaves
   oppositely in the two builds — never treat `isComplete()` as a portable
   "did this tween exist" probe.
6. **`TweenEasing` is declared outside the `PIXELROOT32_ENABLE_CAMERA_TWEEN` guard**,
   so the enum compiles in both flag states. Only the `CameraTween` class template is
   swapped for a same-signature stub (`startTween` → `kInvalidSlotId`, `activeCount()`
   → 0, `update`/`cancel` empty).
7. **`CameraTween::update()` takes a `Camera2D*` pointer**, not a reference — pass
   `&camera`. It is also the only place the tween writes the camera position.
8. `CameraTween` is **non-copyable** (copy ctor and copy-assign are `= delete`). Store
   it by value in the scene or hold it by reference; never pass it by value.

## Common Patterns

### Side-scrolling follow with bounds
```cpp
camera.setBounds(0, mapWidth - viewportWidth);
camera.followTarget(player.getX());
camera.apply(renderer);
```

### Screen shake on impact
```cpp
effects.triggerShake(toScalar(4.0f), 200);
// In update loop:
effects.update(dt);
camera.apply(renderer, effects.getOffset());
```

### Parallax (manual offset)
```cpp
// Parallax layers shift at different rates
Scalar parallaxOffset = camera.getX() * parallaxFactor;
renderer.drawSprite(bgSprite, static_cast<int>(parallaxOffset), 0);
```

## Agent Constraints
- **Memory:** NEVER use `new` to allocate camera effects dynamically.
