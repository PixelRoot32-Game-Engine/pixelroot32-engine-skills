---
name: pixelroot32-particles
description: Particle system with configurable emitters, built-in presets library, particle lifecycle (spawn → update → fade → die), velocity/gravity/friction physics, color interpolation, and angle-based emission. Use when implementing visual effects like fire, explosions, smoke, sparks, or dust.
license: MIT
compatibility: opencode>=0.1.0
metadata:
  domain: engine
  subsystem: graphics
  module: particles
  feature_gate: PIXELROOT32_ENABLE_PARTICLES
  platform: cross-platform
  engine_version: "1.9.0+unreleased"
---

## Overview

PixelRoot32's particle system provides lightweight, entity-based particle emitters with configurable behavior. `ParticleEmitter` inherits from `Entity` and manages a fixed-size pool (max 50 particles). Each `Particle` has position, velocity, lifetime, and color interpolation. A `ParticleConfig` struct controls all behavior parameters. Built-in presets (`ParticlePresets`) provide ready-to-use effects: Fire, Explosion, Smoke, Sparks, and Dust.

## Key APIs

### ParticleEmitter

**Header**: `include/graphics/particles/ParticleEmitter.h`
**Namespace**: `pixelroot32::graphics::particles`
**Feature gate**: `PIXELROOT32_ENABLE_PARTICLES`

```cpp
ParticleEmitter emitter({100, 100}, config);

// Burst particles from a position
emitter.burst({120, 100}, 20);  // Emit 20 particles at (120, 100)

// Entity lifecycle (called by Scene)
emitter.update(deltaTime);
emitter.draw(renderer);
```

### ParticleConfig

**Header**: `include/graphics/particles/ParticleConfig.h`

```cpp
struct ParticleConfig {
    Color startColor;                  // Color at birth
    Color endColor;                    // Color at death (if fadeColor=true)

    pixelroot32::math::Scalar minSpeed;      // Min initial speed
    pixelroot32::math::Scalar maxSpeed;      // Max initial speed

    pixelroot32::math::Scalar gravity;       // Y-axis gravity per frame (+down, -up)
    pixelroot32::math::Scalar friction;      // Air resistance (0.0-1.0) applied to velocity

    uint8_t minLife;                   // Min lifetime in frames
    uint8_t maxLife;                   // Max lifetime in frames

    bool fadeColor;                    // Interpolate startColor → endColor over life

    pixelroot32::math::Scalar minAngleDeg;   // Min emission angle (0 = right)
    pixelroot32::math::Scalar maxAngleDeg;   // Max emission angle
};
```

**No field is `float`.** The SIX `Scalar` fields — `minSpeed`, `maxSpeed`, `gravity`, `friction`, `minAngleDeg`, `maxAngleDeg` — MUST be assigned with `toScalar(...)`. `minLife`/`maxLife` are plain `uint8_t`, `fadeColor` is a plain `bool`, and the two color fields are plain `Color` values.

### Particle

**Header**: `include/graphics/particles/Particle.h`

```cpp
struct Particle {
    math::Vector2 position; // Current position
    math::Vector2 velocity; // Current velocity

    uint16_t color;         // Current color (RGB565)
    Color startColor;       // Initial color
    Color endColor;         // Final color

    uint8_t life;           // Remaining life in frames
    uint8_t maxLife;        // Total lifespan

    bool active;            // Currently in use?
};
```

`Particle` has **no `Scalar` fields** — position and velocity are `math::Vector2`, life counters are `uint8_t`.

### ParticlePresets

**Header**: `include/graphics/particles/ParticlePresets.h`

| Preset | Colors | Speed | Gravity | Life | Use |
|--------|--------|-------|---------|------|-----|
| `ParticlePresets::Fire` | Red → DarkRed | 0.5-1.5 | -0.02 (up) | 20-40 frames | Campfire, torch |
| `ParticlePresets::Explosion` | Yellow → Black | 2.0-4.0 | +0.1 (down) | 10-20 frames | Explosion burst |
| `ParticlePresets::Sparks` | White → Yellow | 1.5-3.0 | +0.15 (down) | 8-15 frames | Impact sparks |
| `ParticlePresets::Smoke` | DarkGray → Black | 0.2-0.6 | -0.01 (up) | 40-80 frames | Smoke plume |
| `ParticlePresets::Dust` | LightGray → DarkGray | 0.3-1.0 | +0.08 (down) | 12-25 frames | Footstep dust |

## Particle Update Loop

For each active particle every frame:
```
1. velocity.y += gravity          // Apply gravity
2. velocity *= friction           // Apply air resistance
3. position += velocity           // Integrate position
4. life -= 1                      // Decrement life
5. if (fadeColor): interpolate color from startColor to endColor
6. if (life == 0): active = false // Mark for reuse
```

## Composition Patterns

### Explosion on collision
```
void Player::onCollision(Actor* other):
  ├── if (other->isInLayer(ENEMY_LAYER)):
  │     emitter.burst(position, 30)              // 30 particles
  │     emitter.setConfig(ParticlePresets::Explosion)
  └── // ParticleEmitter update/draw in scene loop
```

### Continuous fire from torch
```
Scene::init():
  ├── torchEmitter = ParticleEmitter(torchPos, ParticlePresets::Fire)
  └── addEntity(&torchEmitter)

Scene::update(dt):
  ├── torchEmitter.update(dt)
  └── if (frame % 3 == 0):  // Every 3 frames
        torchEmitter.burst(torchPos, 2)  // Steady stream
```

### Custom particle config
```
ParticleConfig conf;
conf.startColor = Color::Cyan;             // Color — plain value
conf.endColor = Color::Blue;               // Color — plain value
conf.minSpeed = toScalar(1.0f);            // Scalar
conf.maxSpeed = toScalar(3.0f);            // Scalar
conf.gravity = toScalar(-0.05f);           // Scalar — negative: float upward
conf.friction = toScalar(0.95f);           // Scalar
conf.minLife = 15;                         // uint8_t — plain value
conf.maxLife = 30;                         // uint8_t — plain value
conf.fadeColor = true;                     // bool — plain value
conf.minAngleDeg = toScalar(45.0f);        // Scalar — cone spread
conf.maxAngleDeg = toScalar(135.0f);       // Scalar

ParticleEmitter magicEmitter({160, 120}, conf);
magicEmitter.burst({160, 120}, 25);
```

## ESP32 Constraints

- **MAX_PARTICLES_PER_EMITTER**: a **preprocessor macro defined in `include/graphics/particles/ParticleEmitter.h`**, value **50** — it is NOT in `ParticleConfig.h` or `Particle.h`. Use multiple emitters for more particles. Never change this in hot code.
- **No heap allocation**: `Particle particles[MAX_PARTICLES_PER_EMITTER]` is a fixed array inside the emitter. No runtime allocation.
- **`uint16_t` for color**: Particles store color as packed RGB565 (not the `Color` enum). The `lerpColor()` function converts at interpolation time.
- **Lifetime in frames**: `minLife`/`maxLife` are measured in frames (not seconds). At 60fps, 60 life = 1 second.
- **Gravity direction**: Positive gravity = downward. Negative gravity = upward. Use `toScalar()` values consistent with Scalar type.

## Gotchas

1. **Burst position vs emitter position**: `burst(position, count)` takes an explicit position parameter — particles emit from there, not from the emitter's `position`.
2. **Life is frames, not seconds**: At 60fps, `maxLife = 60` gives 1 second of particle life. Adjust based on target framerate.
3. **Friction is multiplicative**: Applied as `velocity *= friction` each frame. `friction = 0.90f` means 10% velocity loss per frame. `friction = 1.0f` means no slowdown.
4. **Angle range**: `minAngleDeg` and `maxAngleDeg` define the emission cone. `0°` = right, `90°` = down, `180°` = left. For full 360°: `min=0, max=360`.
5. **Color interpolation**: Only active when `fadeColor = true`. Otherwise particles keep `startColor` for their full life.
6. **`lerpColor` is private**: Color interpolation is handled internally by `ParticleEmitter`. Game code cannot call it directly.
7. **Inactive particle reuse**: Particles with `active = false` are recycled by `burst()`. No allocation happens at burst time.

## Common Patterns

### Footstep dust
```cpp
void Player::onStep() {
    dustEmitter.burst({position.x, position.y + height}, 3);
}
```

### Death explosion
```cpp
void Enemy::kill() {
    // Three simultaneous emitter types
    explosionEmitter.burst(position, 30);
    sparkEmitter.burst(position, 15);
    smokeEmitter.burst(position, 10);

    setVisible(false);  // Hide enemy
    // Emitters continue independently (added to scene)
}
```

### Rain/Snow effect
```cpp
// Heavy snow: burst from top of screen every frame
// Emitter at (120, 0)
ParticleConfig snow;
snow.startColor = Color::White;
snow.endColor = Color::LightGray;
snow.minSpeed = toScalar(0.5f);
snow.maxSpeed = toScalar(2.0f);
snow.gravity = toScalar(0.5f);
snow.friction = toScalar(1.0f);   // No air resistance
snow.minLife = 100;               // uint8_t
snow.maxLife = 200;               // uint8_t
snow.fadeColor = false;
snow.minAngleDeg = toScalar(260); // Falling left
snow.maxAngleDeg = toScalar(280); // Falling right

ParticleEmitter snowEmitter({120, -10}, snow);
// In Scene::update:
snowEmitter.burst({static_cast<int>(random(0, 240)), -5}, 3);
```

## Agent Constraints
- **Constants:** NEVER attempt to alter or override `MAX_PARTICLES_PER_EMITTER` in hot code. It is a strict hardware boundary.
