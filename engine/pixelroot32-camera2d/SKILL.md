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
---

## Overview

PixelRoot32 provides a 2D camera system via `Camera2D` and `CameraEffectsSystem`. The camera manages the viewable area with bounds clamping and smooth follow. Effects (shake, punch, offset) are composed on top via a separate system.

## Key APIs

### Camera2D

**Header**: `include/graphics/Camera2D.h`
**Namespace**: `pixelroot32::graphics`

```cpp
Camera2D camera(viewportWidth, viewportHeight);

// Bounds (unbounded if min > max on that axis)
camera.setBounds(minX, maxX);        // Horizontal
camera.setVerticalBounds(minY, maxY); // Vertical

// Position
camera.setPosition({x, y});
camera.followTarget(targetX);        // Horizontal only
camera.followTarget(target);         // Both axes

// Apply to renderer
camera.apply(renderer);              // Without effects
camera.apply(renderer, effectOffset); // With effects
```

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

## Composition Pattern

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
| **Punch** | Directional impulse with linear decay | Hit反馈, dash |
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
