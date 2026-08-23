---
name: pixelroot32-memory-optimization
description: Apply memory optimization techniques for ESP32 using modular compilation, object pooling, and zero-allocation patterns. Use when sizing a RAM/flash budget, choosing compile-time feature flags, or removing allocations from the game loop.
license: MIT
compatibility: opencode>=0.1.0
metadata:
  domain: engine
  language: cpp
  platform: esp32
  engine_version: "1.9.0+unreleased"
---

## Overview

Generate memory-efficient code using PixelRoot32's modular compilation, object pooling, and fixed-size patterns for ESP32 constraints.

## Source of Truth

- `docs/MEMORY_MANAGEMENT_GUIDE.md` - Memory management
- `docs/STYLE_GUIDE.md` - Best practices
- `docs/ARCHITECTURE.md` - Subsystem details

## Memory Savings by Subsystem

Measured on ESP32 (`esp32dev`) with `scripts/measure_modular_size.py`, engine 1.9.0+unreleased,
APU dependency `gperez88/PixelRoot32-APU@^2.0.0`. Each row is the delta between `measure_full`
and that flag's `measure_no_*` build. RAM = `data + bss`, Flash = `text + data`.

| Flag Disabled | RAM Savings | Flash Savings |
|--------------|-------------|--------------|
| `PIXELROOT32_ENABLE_AUDIO=0` | ~17 KB (16,936 B) | ~15 KB (15,712 B) |
| `PIXELROOT32_ENABLE_PHYSICS=0` | ~9 KB (8,744 B) | ~7 KB (6,988 B) |
| `PIXELROOT32_ENABLE_UI_SYSTEM=0` | ~9 KB (8,784 B) | ~12 KB (12,368 B) |
| `PIXELROOT32_ENABLE_PARTICLES=0` | ~2 KB (2,328 B) | ~5 KB (4,956 B) |
| **All four disabled** | **~36 KB (37,036 B)** | **~40 KB (41,240 B)** |

Baseline for reference: `measure_full` is RAM 103,289 B / Flash 348,297 B; `measure_all_off`
is RAM 82,221 B / Flash 307,057 B.

**Audio is the biggest RAM win, not physics.** The dominant cost is the scheduler's own
buffers — the probe reports `sizeof(ESP32AudioScheduler) = 15,968` B, next to
`sizeof(CollisionSystem) = 4,884`, `sizeof(ParticleEmitter) = 1,264`, `sizeof(UIManager) = 96`.
The 1.8.0 APU extraction moved the synthesis core into an external library but **did not** move
that RAM out of the firmware: the library still links in.

> Deltas are not additive. Summing the four rows gives 36,792 B RAM / 40,024 B Flash, while
> disabling all four actually saves 37,036 B / 41,240 B — slightly *more*, because some shared
> support code survives every single-flag build and only drops out once nothing references it.
> Budget against the `all four disabled` row, never against the sum.
>
> Re-measure with `scripts/measure_modular_size.py` after any change that moves code across the
> engine/APU boundary or changes a driver. The script shells out to `pio`, so the PlatformIO on
> `PATH` must be a working install.

## Compile-Time Flag Inventory

Complete list of the engine's `PIXELROOT32_*` build flags, grouped by default value. Type-safe
`constexpr` mirrors live in namespace `pixelroot32::platforms::config`
(`include/platforms/EngineConfig.h`) — prefer the mirror in C++ expressions, the macro only in
preprocessor context.

**Subsystem gates — default `1` (ON, `#if`-tested):**

`PIXELROOT32_ENABLE_AUDIO`, `_PHYSICS`, `_UI_SYSTEM`, `_PARTICLES`, `_CAMERA_EFFECTS`,
`_SCENE_TRANSITIONS`, `_STATIC_TILEMAP_FB_CACHE`

**Default `0` (OFF, `#if`-tested) — opt in with `-D<FLAG>=1`:**

`PIXELROOT32_ENABLE_STATIC_LAYER_SNAPSHOT`, `_GAMEPLAY_EVENTS`, `_INTERACTION_TRIGGERS`,
`_SPATIAL_QUERY`, `_DEPTH_SORT`, `_GAMEPLAY_STATE_MACHINE`, `_GAMEPLAY_OBJECT_POOL`,
`_GAMEPLAY_GRID_SPACE`, `_GAMEPLAY_ROOM`, `_CAMERA_TWEEN`, `_PROJECTION`, `_TILEMAP_PROJECTION`,
`PIXELROOT32_ENABLE_TOUCH`, `_DIRTY_REGIONS`, `_DIRTY_REGION_PROFILING`,
`PIXELROOT32_TFT_12BIT_COLOR`

**`#ifdef`-tested — undefined by default, therefore effectively off.** A value of `0` still
enables them; define or omit, never `=0`:

`PIXELROOT32_ENABLE_2BPP_SPRITES`, `_4BPP_SPRITES`, `_SCENE_ARENA`, `_TILE_ANIMATIONS`,
`_PROFILING`, `_ENABLE_DEBUG_OVERLAY`

**Driver / DMA knobs:** `PIXELROOT32_TFT_ESPI_LINES_PER_BLOCK` (default 60),
`PIXELROOT32_TFT_ESPI_LINES_PER_BLOCK_FALLBACK` (default 30).

### Cross-Flag Rules (hard `#error` in `include/platforms/PlatformDefaults.h`)

| Rule | Location |
|------|----------|
| `_INTERACTION_TRIGGERS` or `_SPATIAL_QUERY` **require** `PIXELROOT32_ENABLE_PHYSICS=1` | `:171-173` |
| `_TILEMAP_PROJECTION` **requires** `PIXELROOT32_ENABLE_PROJECTION` | `:178-179` |
| `PIXELROOT32_ENABLE_GAMEPLAY_PROJECTION` was **renamed** to `PIXELROOT32_ENABLE_PROJECTION`; the old spelling is now a hard `#error` tripwire | `:137-139` |

There is **no** `EnableTilemapProjection` constant in `pixelroot32::platforms::config` —
`PIXELROOT32_ENABLE_TILEMAP_PROJECTION` is preprocessor-only and has no type-safe mirror.

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

> `[profile_*]` sections carry **no `env:` prefix**, so they are templates to `extends` from — they
> cannot be passed to `pio run -e` / `pio test -e` directly.

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

For strict zero-allocation guarantees. `SceneArena` is declared **inside `include/core/Scene.h`** — there is **no `include/core/SceneArena.h`**. Namespace `pixelroot32::core`; it is a `struct` with public members `unsigned char* buffer; std::size_t capacity; std::size_t offset;`.

**The gate is never defined by the engine.** `PIXELROOT32_ENABLE_SCENE_ARENA` is tested with `#ifdef` (`include/platforms/EngineConfig.h`, mirror `pixelroot32::platforms::config::EnableSceneArena`) and is defined **nowhere** in the engine or in `platformio.ini` — default OFF. The game MUST pass `-D PIXELROOT32_ENABLE_SCENE_ARENA` on its own build line. With the gate off, `init()`/`reset()` are no-ops, `allocate()` always returns `nullptr`, and therefore `arenaNew<T>()` always returns `nullptr`.

```cpp
#ifdef PIXELROOT32_ENABLE_SCENE_ARENA
#include <core/Scene.h>   // SceneArena lives here

// Game-owned backing buffer
alignas(alignof(max_align_t)) unsigned char arenaBuffer[4096];

pixelroot32::core::SceneArena arena;           // default ctor ONLY — no (buffer, size) form
arena.init(arenaBuffer, sizeof(arenaBuffer));  // void init(void* memory, std::size_t size)

// Raw allocation — alignment is REQUIRED (no default argument, and there is no alloc())
void* raw = arena.allocate(sizeof(MyEntity), alignof(MyEntity));

// Typed allocation + placement new; nullptr when exhausted (or when the gate is off)
MyEntity* e = pixelroot32::core::arenaNew<MyEntity>(arena, 50, 50);
if (e == nullptr) { /* out of arena space */ }

arena.reset();   // rewinds offset to 0
#endif
```

- `Scene::arena` is a plain `SceneArena` member of `Scene` (`include/core/Scene.h`), not wrapped in any `#if` — the gate only changes the arena's runtime behavior.
- **Destructors are NEVER run.** `reset()` only rewinds the offset, so only trivially destructible types are safe to arena-allocate.

## Entity Limits

All of these are `#ifndef`-guarded macros in `include/platforms/EngineConfig.h`, each mirrored by a `constexpr` constant in namespace `pixelroot32::platforms::config`. Because they are `#ifndef`-guarded, a `-D` on the build line wins. `include/platforms/PlatformDefaults.h` defines **none** of them.

| Limit | Macro | Default | ESP32 (`[base_esp32]`) | `constexpr` mirror |
|-------|-------|---------|------------------------|--------------------|
| Max scenes | `MAX_SCENES` | 8 | 8 | `config::MaxScenes` |
| Max render layers | `MAX_LAYERS` | 4 | 4 | `config::MaxLayers` |
| Max entities | `MAX_ENTITIES` | 64 | 64 | `config::MaxEntities` |
| Spatial grid cell | `SPATIAL_GRID_CELL_SIZE` | 32 | 32 | `config::SpatialGridCellSize` |
| Entities per cell | `SPATIAL_GRID_MAX_ENTITIES_PER_CELL` | 24 | 24 | `config::SpatialGridMaxEntitiesPerCell` |
| Static per cell | `SPATIAL_GRID_MAX_STATIC_PER_CELL` | 12 | **4** | `config::SpatialGridMaxStaticPerCell` |
| Dynamic per cell | `SPATIAL_GRID_MAX_DYNAMIC_PER_CELL` | 12 | **4** | `config::SpatialGridMaxDynamicPerCell` |
| Max physics entities | `PHYSICS_MAX_ENTITIES` | 64 | 64 | `config::PhysicsMaxEntities` |
| Max physics contacts | `PHYSICS_MAX_CONTACTS` | 128 | **64** | `config::PhysicsMaxContacts` |
| Max physics pairs | `PHYSICS_MAX_PAIRS` | 128 | **64** | `config::PhysicsMaxPairs` |

**ESP32 overrides**: `platformio.ini` `[base_esp32] build_flags` lowers exactly four of them — `SPATIAL_GRID_MAX_STATIC_PER_CELL=4`, `SPATIAL_GRID_MAX_DYNAMIC_PER_CELL=4`, `PHYSICS_MAX_CONTACTS=64`, `PHYSICS_MAX_PAIRS=64`. Every other limit keeps its default on hardware. Size hardware budgets against the ESP32 column, not the defaults.

Both spellings are valid in game code: the macro (`MAX_ENTITIES`) in preprocessor context, and the `constexpr` mirror (`pixelroot32::platforms::config::MaxEntities`) in C++ expressions — prefer the mirror.

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

### StaticLayerSnapshot Heap Cost

`PIXELROOT32_ENABLE_STATIC_LAYER_SNAPSHOT` allocates **one additional full logical framebuffer** on
the heap — **57,600 bytes at 240×240** (`240 × 240 × 1 byte/px`). That is a second framebuffer-sized
allocation on top of the renderer's own, which is why the flag **defaults to `0`** and must never be
enabled on an ESP32 budget without first re-measuring free heap. Budget it as
`logicalWidth × logicalHeight` bytes at any other resolution.

## Constraints

- NEVER use `new` or `malloc` in game loop
- Use fixed-size arrays instead of `std::vector` with push_back
- Pre-allocate entities in `init()`
- Use pooling for high-rotation objects (bullets, particles)
- Pass `std::string_view` not `std::string` to avoid copies
- Use `snprintf` with stack buffers for formatting

## Subsystem-Specific Constraints

For memory patterns specific to each subsystem, refer to the specialized skills:

| Subsystem | Skill | Key Constraints |
|-----------|-------|-----------------|
| Camera | `pixelroot32-camera2d` | 4 effect slots, zero-heap, Xorshift32 |
| Audio | `pixelroot32-audio` | 8-voice pool, Q15 no-FPU, SPSC queue |
| Rendering | `pixelroot32-sprite-renderer` | Dirty grid, static cache, palette |
| Physics | `pixelroot32-physics` | 64 contacts / 64 pairs on ESP32 (128 is the desktop default), Fixed16, spatial grid |
| UI | `pixelroot32-ui-system` | Widget pool, hit test cache |
| Particles | `pixelroot32-particles` | Emitter pool, preset configs |
| Projection | `pixelroot32-projection` | `_PROJECTION` / `_TILEMAP_PROJECTION` default 0; `_TILEMAP_PROJECTION` requires `_PROJECTION` |
| Gameplay framework | `pixelroot32-gameplay-framework` | All `_GAMEPLAY_*` gates default 0; `_INTERACTION_TRIGGERS` / `_SPATIAL_QUERY` require `_PHYSICS=1` |

## Agent Constraints
- **Strict Adherence:** The recommendations here are ABSOLUTE rules for Agents.
- **Forbidden:** NO `std::vector` with dynamic growth. NO `std::string` passed by value (use `std::string_view`). NO `std::function` (use raw function pointers). NO `std::shared_ptr`.
