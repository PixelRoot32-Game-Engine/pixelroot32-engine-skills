---
name: pixelroot32-memory-optimization
description: Apply memory optimization techniques for ESP32 using modular compilation, object pooling, and zero-allocation patterns
license: MIT
compatibility: opencode>=0.1.0
metadata:
  domain: engine
  language: cpp
  platform: esp32
---

## Overview

Generate memory-efficient code using PixelRoot32's modular compilation, object pooling, and fixed-size patterns for ESP32 constraints.

## Source of Truth

- `docs/MEMORY_MANAGEMENT_GUIDE.md` - Memory management
- `docs/STYLE_GUIDE.md` - Best practices
- `docs/ARCHITECTURE.md` - Subsystem details

## Memory Savings by Subsystem

| Flag Disabled | RAM Savings | Flash Savings |
|--------------|-------------|--------------|
| `PIXELROOT32_ENABLE_AUDIO=0` | ~8 KB | ~15 KB |
| `PIXELROOT32_ENABLE_PHYSICS=0` | ~12 KB | ~25 KB |
| `PIXELROOT32_ENABLE_UI_SYSTEM=0` | ~4 KB | ~20 KB |
| `PIXELROOT32_ENABLE_PARTICLES=0` | ~6 KB | ~10 KB |
| **All disabled** | **~30 KB** | **~70 KB** |

## Modular Compilation Pattern

```cpp
// File-level guard
#if PIXELROOT32_ENABLE_AUDIO
#include <audio/AudioEngine.h>
#endif

#if PIXELROOT32_ENABLE_PHYSICS
#include <physics/CollisionSystem.h>
#endif

// Constructor initialization
Engine::Engine(...)
#if PIXELROOT32_ENABLE_AUDIO
    : audioEngine(audioConfig, capabilities),
#endif
      renderer(std::move(displayConfig)) {}

// Runtime initialization
void Engine::init() {
    renderer.init();
#if PIXELROOT32_ENABLE_PHYSICS
    collisionSystem.init();
#endif
}
```

## Build Profiles

```ini
[profile_minimal]
build_flags =
    -DPIXELROOT32_ENABLE_AUDIO=0
    -DPIXELROOT32_ENABLE_PHYSICS=0
    -DPIXELROOT32_ENABLE_PARTICLES=0
    -DPIXELROOT32_ENABLE_UI_SYSTEM=0
    -DMAX_ENTITIES=16

[profile_arcade]
build_flags =
    -DPIXELROOT32_ENABLE_AUDIO=1
    -DPIXELROOT32_ENABLE_PHYSICS=1
    -DPIXELROOT32_ENABLE_PARTICLES=1
    -DPIXELROOT32_ENABLE_UI_SYSTEM=0
```

## Object Pooling Pattern

```cpp
class BulletManager {
    static constexpr size_t MAX_BULLETS = 50;
    
    struct Bullet {
        pixelroot32::core::Actor actor;
        bool isActive = false;
    };
    
    Bullet bullets[MAX_BULLETS];
    
public:
    Bullet* spawn() {
        for (auto& b : bullets) {
            if (!b.isActive) {
                b.isActive = true;
                return &b;
            }
        }
        return nullptr;  // Pool exhausted
    }
    
    void despawn(Bullet* b) {
        b->isActive = false;
    }
};
```

## Zero-Allocation Game Loop

```cpp
// BAD - allocations in update()
void Player::update(unsigned long dt) {
    std::string msg = "Position: " + std::to_string(x);  // allocates!
    log(msg.c_str());
}

// GOOD - no allocations
void Player::update(unsigned long dt) {
    char buf[32];
    snprintf(buf, sizeof(buf), "Position: %d", static_cast<int>(x));
    log(buf);
}
```

## Smart Pointers

```cpp
// Owning - use unique_ptr
std::unique_ptr<pixelroot32::core::Scene> scene = 
    std::make_unique<GameScene>();

// Non-owning - pass raw pointer
sceneManager.setCurrentScene(scene.get());

// Transfer ownership
std::unique_ptr<Player> player = std::make_unique<Player>(100, 100);
std::unique_ptr<Player> transferred = std::move(player);
```

## Scene Arena (Optional)

For strict zero-allocation guarantees:

```cpp
#if PIXELROOT32_ENABLE_SCENE_ARENA
#include <core/SceneArena.h>

// Pre-allocated buffer
alignas(4) char arenaBuffer[4096];
pixelroot32::core::SceneArena arena(arenaBuffer, sizeof(arenaBuffer));

void* allocated = arena.alloc(sizeof(MyEntity));
#endif
```

## Entity Limits

| Limit | Default | Configurable |
|-------|---------|--------------|
| Max Entities | 32 | `MAX_ENTITIES` |
| Max Physics Contacts | 128 | `PHYSICS_MAX_CONTACTS` |
| Max Render Layers | 3 | `MAX_LAYERS` |
| Spatial Grid Cell | 32px | `SPATIAL_GRID_CELL_SIZE` |

## Per-Entity Costs

| Component | Memory Cost |
|-----------|-------------|
| Base Entity | ~32 bytes |
| Actor | ~64 bytes |
| PhysicsActor | ~128 bytes |
| KinematicActor | ~144 bytes |
| Sprite (1bpp) | (w×h/8) bytes |
| Sprite (2bpp) | (w×h/4) bytes |

## Framebuffer by Resolution

| Resolution | Framebuffer | Total |
|------------|------------|-------|
| 128×128 | ~16 KB | ~17 KB |
| 160×160 | ~25 KB | ~27 KB |
| 240×240 | ~57 KB | ~59 KB |

## Constraints

- NEVER use `new` or `malloc` in game loop
- Use fixed-size arrays instead of `std::vector` with push_back
- Pre-allocate entities in `init()`
- Use pooling for high-rotation objects (bullets, particles)
- Pass `std::string_view` not `std::string` to avoid copies
- Use `snprintf` with stack buffers for formatting
