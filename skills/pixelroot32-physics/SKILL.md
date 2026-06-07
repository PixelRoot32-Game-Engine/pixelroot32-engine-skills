---
name: pixelroot32-physics
description: Flat Solver 6-step physics pipeline with broad/narrow phase collision detection, CCD, one-way platforms, Fixed16 math, spatial grid with static/dynamic layers, and tile collision builder. Use when implementing physics simulation, collision detection, or tile-based level collision.
license: MIT
compatibility: opencode>=0.1.0
metadata:
  domain: engine
  subsystem: physics
  module: collision
  feature_gate: PIXELROOT32_ENABLE_PHYSICS
  platform: cross-platform
---

## Overview

PixelRoot32's physics system uses a "Flat Solver" 6-step pipeline: Update → Detect → Solve Velocity → Integrate Position → Solve Penetration → Trigger Callbacks. It features a spatial grid with separate static/dynamic layers, continuous collision detection (CCD) for fast-moving bodies, tile collision merging via `TileCollisionBuilder`, and support for 4 actor types (Static, Kinematic, Rigid, Sensor). All math uses the adaptable `Scalar` type (Fixed16 Q16.16 on no-FPU ESP32, float on PC).

## Key APIs

### CollisionSystem

**Header**: `include/physics/CollisionSystem.h`
**Namespace**: `pixelroot32::physics`

```cpp
CollisionSystem physics;

// Registration
physics.addEntity(entity);     // Entity must contain PhysicsActor
physics.removeEntity(entity);

// Full pipeline (called by PhysicsScheduler)
physics.update();
// Or manual step control:
physics.detectCollisions();     // Broad + narrow phase
physics.solveVelocity();        // Resolve velocities
physics.integratePositions();   // Move bodies
physics.solvePenetration();     // Separate overlapping bodies
physics.triggerCallbacks();     // Fire onCollision events

// Configuration constants
CollisionSystem::FIXED_DT;          // 1/60th second
CollisionSystem::SLOP;              // 0.02f penetration slop
CollisionSystem::VELOCITY_DAMPING;  // 0.999f
CollisionSystem::MAX_VELOCITY;      // 500.0f units/s
CollisionSystem::CCD_THRESHOLD;     // 3.0f (triggers CCD)
```

### SpatialGrid

**Header**: `include/physics/SpatialGrid.h`
**Namespace**: `pixelroot32::physics`

```cpp
SpatialGrid grid;

// Separate static/dynamic layers
grid.clearDynamic();              // Clear per-frame dynamic bodies
grid.clear();                     // Clear everything
grid.markStaticDirty();           // Rebuild static layer next frame
grid.rebuildStaticIfNeeded(entities, entityCount);
grid.insertDynamic(actor);        // Insert moving body

// Query
Actor* candidates[16];
int count = 0;
grid.getPotentialColliders(actor, candidates, count, 16);
```

### PhysicsScheduler (Fixed Timestep)

**Header**: `include/physics/PhysicsScheduler.h`
**Namespace**: `pixelroot32::physics`

```cpp
PhysicsScheduler scheduler;

scheduler.init();

// In game loop (called with real delta time)
uint8_t steps = scheduler.update(realDeltaMicros, collisionSystem);
// Returns steps executed (0-4)
// Normal: max 1 step per frame, Backlog: max 4 steps

// Diagnostics
uint8_t stepsExecuted = scheduler.getStepsExecuted();
uint32_t accumulator = scheduler.getAccumulator();
```

### CollisionTypes

**Header**: `include/physics/CollisionTypes.h`
**Namespace**: `pixelroot32::physics`

```cpp
// Collision layers (uint16_t bitmask)
using CollisionLayer = uint16_t;
DefaultLayers::kNone;   // 0
DefaultLayers::kAll;    // 0xFFFF

// Collision shapes
Circle c = { toScalar(50), toScalar(50), toScalar(10) };
Segment seg = { 0, 0, 100, 100 };

// Intersection tests
bool hit = intersects(circleA, circleB);
bool hit2 = intersects(circle, rect);
bool hit3 = intersects(segment, rect);

// CCD: sweep test
// circleStart -> circleEnd against rect
```

### Contact and KinematicCollision

```cpp
// Full contact info (Rigid-Rigid or Rigid-Static)
struct Contact {
    PhysicsActor* bodyA;
    PhysicsActor* bodyB;
    Vector2 normal;
    Vector2 contactPoint;
    Scalar penetration;
    Scalar restitution;
    bool isSensorContact;   // true if either body is sensor
};

// Kinematic collision result
struct KinematicCollision {
    Actor* collider;
    Vector2 normal;
    Vector2 position;
    Scalar travel;      // Distance traveled before collision
    Scalar remainder;   // Remaining distance
};
```

### TileCollisionBuilder

**Header**: `include/physics/TileCollisionBuilder.h`
**Namespace**: `pixelroot32::physics`

```cpp
TileCollisionBuilderConfig cfg(16, 16);  // tile width, tile height
TileCollisionBuilder builder(scene, cfg);

// Build from exported behavior layer
// Creates StaticActor for SOLID, SensorActor for SENSOR/DAMAGE/COLLECTIBLE
int created = builder.buildFromBehaviorLayer(layer, 0);

// Merge adjacent tiles into larger bodies (reduces entity count)
int merged = builder.buildMergedFromBehaviorLayer(layer, 0);
```

### TileAttributes (Physics Metadata)

**Header**: `include/physics/TileAttributes.h`

```cpp
// Behavior layer flags
uint8_t flags = getTileFlag(behaviorLayer, x, y);
if (flags & TileFlags::SOLID) { /* block movement */ }
if (flags & TileFlags::ONEWAY) { /* one-way platform */ }
if (flags & TileFlags::SENSOR) { /* trigger zone */ }
```

### KinematicActor (Character Controller)

**Header**: `include/physics/KinematicActor.h`
**Namespace**: `pixelroot32::physics`
**Inherits**: `pixelroot32::core::PhysicsActor`

Script-driven bodies with slide/snap collision. Use for player characters and other manually moved actors — not the same as setting `PhysicsBodyType::KINEMATIC` on a generic `PhysicsActor`.

```cpp
enum class SnapPolicy { None, Step, Continuous };

KinematicActor player(100, 100, 16, 16);
Vector2 velocity = player.getVelocity();

// dt is required (seconds). Assign the returned velocity.
velocity = player.moveAndSlide(
    velocity,
    toScalar(deltaTime * 0.001f),
    {0, -1},                          // upDirection
    SnapPolicy::Step,
    jumpThisFrame ? Vector2{} : Vector2{0, KinematicActor::MIN_SNAP},
    &floorBody                        // optional KINEMATIC floor pointer
);

// Current-frame contact state (no persistence across frames)
player.is_on_floor();
player.is_on_wall();
player.is_on_ceiling();

// Platform velocity inheritance (KINEMATIC floors)
const Vector2& floorVel = player.getFloorVelocity();
player.clearFloorVelocity();          // call on jump
```

**Do not use** `moveAndSlideWithSnap` — removed; snap is unified via `SnapPolicy` + `snapVector`.

## 6-Step Pipeline Order

```
1. detectCollisions()      — Broad phase (SpatialGrid) + Narrow phase (AABB/Circle)
2. solveVelocity()         — Apply impulse resolution, restitution, friction
3. integratePositions()    — Move bodies by velocity * dt
4. solvePenetration()      — Push overlapping bodies apart
5. triggerCallbacks()      — Call onCollision() on each actor
6. (reset per-frame state)
```

## Actor Type Integration

| Actor Type | Body Type | Spatial Grid | Physics Response | Collision Events |
|-----------|-----------|-------------|-----------------|------------------|
| `PhysicsActor` STATIC | Static | Static layer | Blocks others | onCollision |
| `PhysicsActor` KINEMATIC | Kinematic | Dynamic layer | Stops on contact | KinematicCollision |
| `KinematicActor` (class) | Kinematic | Dynamic layer | `moveAndSlide` / `moveAndCollide` | KinematicCollision + `is_on_*` |
| `PhysicsActor` RIGID | Rigid | Dynamic layer | Full resolution | Contact |
| `PhysicsActor` sensor=true | Sensor | Dynamic layer | None | onCollision (events only) |
| `Actor` (base) | — | Not added | None | onCollision |

## Composition Patterns

### Scene physics setup
```
Scene::init():                        // idempotent — calls resetState() internally
  ├── Scene::init()                   // resetState() + physicsScheduler.init()
  ├── TileCollisionBuilder builder(scene, {16, 16})
  ├── builder.buildMergedFromBehaviorLayer(wallLayer, 0)
  └── builder.buildFromBehaviorLayer(triggerLayer, 1)

Scene::update(dt):
  ├── physicsScheduler.update(dt * 1000, collisionSystem)
  └── // Check contact results after physics step
```

### Kinematic character controller
```
Player : public KinematicActor

Player::update(dt):
  ├── velocity.y += gravity * dt
  ├── velocity.x = inputX * speed
  ├── snap = jumpThisFrame ? Vector2{} : Vector2{0, MIN_SNAP}
  ├── velocity = moveAndSlide(velocity, toScalar(dt), {0,-1}, SnapPolicy::Step, snap)
  └── if (jump) clearFloorVelocity()
```

## ESP32 Constraints

- **Fixed16 math**: `Scalar` is `int32_t` (Q16.16) on no-FPU ESP32-C3. All physics math uses `toScalar()`/`toFloat()` wrappers. Never use raw `float` in hot physics paths.
- **SpatialGrid is fully static**: `staticCells`, `dynamicCells` are static arrays sized at compile time via `PlatformConfig`. Grid capacity is fixed — cannot grow at runtime.
- **Max entities**: Defined by `platforms::config::MaxEntities`. `TileCollisionBuilder` halves this for tile collision entities.
- **CCD threshold**: `CCD_THRESHOLD = 3.0f` — bodies moving faster than 3 units/frame trigger sweep-based CCD. Tune via `setMaxVelocity()`.
- **Velocity iterations**: Configurable at compile time via `platforms::config::VelocityIterations`.

## Gotchas

1. **Entity must be registered with CollisionSystem**: Adding an entity to the Scene is NOT sufficient — you must also call `collisionSystem.addEntity(entity)`.
2. **Static layer rebuild**: Call `grid.markStaticDirty()` when changing static actor positions. The grid only rebuilds on `rebuildStaticIfNeeded()`.
3. **KinematicCollision vs Contact**: Kinematic bodies produce `KinematicCollision` (travel/remainder). Rigid bodies produce `Contact` (penetration/normal). Check `triggerCallbacks()` for both.
4. **Sensor contacts**: When `isSensorContact` is true, no physics response is applied (no velocity resolution, no penetration resolution). Only callbacks fire.
5. **One-way platforms**: Rely on previous-position crossing detection. If teleporting an actor, call `setPosition()` to sync `previousPosition`.
6. **Accumulator is never clamped**: `PhysicsScheduler` preserves real time. If the game lags heavily, it will run multiple catch-up steps (max 4).
7. **TileCollisionBuilder entity count**: Returns -1 if the entity limit is exceeded. Check the return value.
8. **`moveAndSlide` requires explicit `dt`**: Pass delta time in seconds (`deltaTime * 0.001f`). There is no default parameter.
9. **Assign `moveAndSlide` return value**: The method returns resolved velocity; always assign back to the caller's velocity variable.
10. **Disable snap on jump explicitly**: Pass `snapVector = Vector2{}` on jump frames — upward velocity does not auto-disable snap.
11. **`SnapPolicy::Continuous` is reserved**: Currently behaves as `None` (debug builds assert once).
12. **`MIN_SNAP` threshold**: Snap is skipped when `snapVector` magnitude is below `KinematicActor::MIN_SNAP` (4.0).

## Common Patterns

### Simple rigid body
```cpp
PhysicsActor crate(100, 100, 16, 16);
crate.setBodyType(PhysicsBodyType::RIGID);
crate.setMass(5.0f);
crate.setCollisionLayer(WORLD_LAYER);
crate.setCollisionMask(WALL_LAYER | PLAYER_LAYER);
```

### One-way platform (stand through from below)
```cpp
PhysicsActor platform(50, 150, 64, 8);
platform.setBodyType(PhysicsBodyType::STATIC);
platform.setOneWay(true);
platform.setCollisionLayer(PLATFORM_LAYER);
```

### CCD for fast projectiles
```cpp
PhysicsActor bullet(0, 100, 4, 4);
bullet.setVelocity(400.0f, 0);  // Exceeds CCD_THRESHOLD
// CCD automatically sweeps against static bodies
```

## Agent Constraints
- **Data Types:** NEVER use raw `float` for physics calculations. YOU MUST use `toScalar()` and the `Scalar` type to maintain ESP32 no-FPU compatibility.
- **Testing:** Any change to the collision solver or physics logic REQUIRES an update to the unit tests in `pixelroot32-testing`.
