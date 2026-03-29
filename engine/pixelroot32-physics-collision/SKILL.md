---
name: pixelroot32-physics-collision
description: Implement physics actors, collision detection, and collision response using the Flat Solver system
license: MIT
compatibility: opencode>=0.1.0
metadata:
  domain: engine
  language: cpp
  platform: esp32
---

## Overview

Generate physics code using PixelRoot32's Flat Solver system. This skill covers collision detection, actor types, and the simulation pipeline.

## Source of Truth

- `docs/PHYSICS_SYSTEM_REFERENCE.md` - Physics system details
- `docs/ARCHITECTURE.md` - Actor hierarchy and CollisionSystem
- `docs/API_REFERENCE.md` - PhysicsActor API

## Actor Types Hierarchy

```
Entity
└── Actor
    └── PhysicsActor (Base)
        ├── StaticActor    (Immovable; static grid layer)
        │   └── SensorActor (Trigger; setSensor(true))
        ├── KinematicActor (Logic-driven movement)
        └── RigidActor     (Full simulation with gravity)
```

## Flat Solver Pipeline

Execute physics in strict order every frame:

1. **Detect Collisions** - Rebuild static grid if dirty; clear dynamic; insert RIGID/KINEMATIC
2. **Solve Velocity** - Impulse-based response (2 iterations); sensor contacts skipped
3. **Integrate Positions** - p = p + v * FIXED_DT (RIGID only)
4. **Solve Penetration** - Baumgarte stabilization + SLOP; sensor contacts skipped
5. **Trigger Callbacks** - onCollision notifications

## Key Constants

```cpp
static constexpr Scalar FIXED_DT = toScalar(1.0f / 60.0f);  // Fixed timestep
static constexpr Scalar SLOP = toScalar(0.02f);              // Ignore penetration < 2cm
static constexpr Scalar BIAS = toScalar(0.2f);               // 20% correction per frame
static constexpr Scalar VELOCITY_THRESHOLD = toScalar(0.5f);  // Zero restitution below
static constexpr int VELOCITY_ITERATIONS = 2;                 // Impulse solver iterations
static constexpr Scalar CCD_THRESHOLD = toScalar(3.0f);      // CCD activation
```

## Spatial Grid Configuration

- **Static layer**: Contains STATIC bodies. Rebuilt only when entities added/removed.
- **Dynamic layer**: Contains RIGID/KINEMATIC. Cleared/refilled every frame.
- **Cell size**: 32px default (configurable via `SPATIAL_GRID_CELL_SIZE`)
- **Max per cell**: 12 static, 12 dynamic

## Collision Shapes

| Shape | API |
|-------|-----|
| AABB | `setShape(CollisionShape::AABB)` |
| Circle | `setShape(CollisionShape::CIRCLE)` + `setRadius(r)` |

## Sensor Implementation

Use sensors for triggers, collectibles, checkpoints:

```cpp
class Coin : public pixelroot32::core::SensorActor {
public:
    Coin(Scalar x, Scalar y) : SensorActor(x, y, 16, 16) {
        setCollisionLayer(Layer::kCollectible);
        setCollisionMask(Layer::kPlayer);
    }
    
    void onCollision(pixelroot32::core::Actor* other) override {
        if (other->isInLayer(Layer::kPlayer)) {
            collect();
        }
    }
};
```

## One-Way Platforms

Use for platforms passable from below, solid from above:

```cpp
platform->setOneWay(true);
// Validation uses previousPosition to detect spatial crossing
// Rejects horizontal collisions
```

## Collision Layers

```cpp
enum DefaultLayers {
    kNone = 0,
    kPlayer = 1 << 0,
    kEnemy = 1 << 1,
    kProjectile = 1 << 2,
    kWall = 1 << 3,
};
```

## PhysicsActor Configuration

```cpp
class Player : public pixelroot32::core::KinematicActor {
public:
    Player(Scalar x, Scalar y) : KinematicActor(x, y, 16, 16) {
        setCollisionLayer(kPlayer);
        setCollisionMask(kWall | kEnemy);
        setShape(CollisionShape::AABB);
    }
    
    void update(unsigned long dt) override {
        // Move using moveAndSlide() or moveAndCollide()
        Vector2 input = getInput();
        setVelocity(input * moveSpeed);
        moveAndSlide(dt);
    }
};
```

## Tile Collision Builder

Generate physics bodies from tilemaps:

```cpp
#include <physics/TileCollisionBuilder.h>

void buildLevelPhysics(Scene& scene, const TileMap2bpp& map, 
                       const TileBehaviorLayer& behaviors) {
    pixelroot32::physics::TileCollisionBuilder builder;
    builder.build(scene, map, behaviors, Layer::kWall, Layer::kSensor);
}
```

## Tile Metadata

Pack/unpack tile data for gameplay:

```cpp
#include <physics/TileAttributes.h>

void onCollision(Actor* other) override {
    if (other->getUserData()) {
        auto [x, y, flags] = pixelroot32::physics::unpackTileData(other->getUserData());
        if (flags & TILE_COLLECTIBLE) {
            collectTile(x, y);
        }
    }
}
```

## Constraints

- Use Fixed16 on non-FPU platforms (ESP32-C3/C6)
- Use `Scalar` for all physics math (NOT float/double directly)
- Never allocate in physics hot path - use fixed contact pool
- Call `updatePreviousPosition()` at frame start for one-way detection
- onCollision is for notifications ONLY - solver handles response
