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
  engine_version: "1.9.0+unreleased"
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

### Spatial Queries

**Header**: `include/physics/CollisionSystem.h`
**Namespace**: `pixelroot32::physics`
**Feature gate**: `PIXELROOT32_ENABLE_SPATIAL_QUERY` — default **0**, mirrored as `pixelroot32::platforms::config::EnableSpatialQuery`. **Hard dependency**: `PIXELROOT32_ENABLE_SPATIAL_QUERY=1` requires `PIXELROOT32_ENABLE_PHYSICS=1`; the combination is enforced by an `#error` in `include/platforms/PlatformDefaults.h`.

```cpp
int queryRadius(pixelroot32::math::Vector2 center,
                pixelroot32::math::Scalar radius,
                CollisionLayer mask,
                pixelroot32::core::Actor** outArray,
                int maxCount);

int queryBox(const pixelroot32::core::Rect& box,
             CollisionLayer mask,
             pixelroot32::core::Actor** outArray,
             int maxCount);
```

Buffer convention: the **caller allocates** the `Actor*` array and passes its capacity as `maxCount`. The return value is the **number of actors written** into `outArray` (never more than `maxCount`). No allocation happens inside the query.

```cpp
#if PIXELROOT32_ENABLE_SPATIAL_QUERY
    pixelroot32::core::Actor* hits[8];
    int found = physics.queryRadius(player.getPosition(), toScalar(24.0f),
                                    ENEMY_LAYER, hits, 8);
    for (int i = 0; i < found; ++i) { /* hits[i] */ }
#endif
```

### Interaction Triggers

**Header**: `include/physics/CollisionSystem.h`
**Namespace**: `pixelroot32::physics`
**Feature gate**: `PIXELROOT32_ENABLE_INTERACTION_TRIGGERS` — default **0**. **Hard dependency**: also requires `PIXELROOT32_ENABLE_PHYSICS=1` (same `#error` in `include/platforms/PlatformDefaults.h`).

```cpp
void setInteractionTracker(pixelroot32::gameplay::InteractionTracker* tracker);
```

The pointer is **non-owning**; the member defaults to `nullptr`. **The wiring is not automatic** — a tracker that is constructed but never handed to `CollisionSystem::setInteractionTracker()` does nothing at all. The caller keeps the tracker alive for as long as the `CollisionSystem` holds the pointer.

`InteractionComponent` / `InteractionTracker` themselves are documented in **`pixelroot32-gameplay-framework`**; this skill only owns the `CollisionSystem` wiring call.

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

### Per-Pixel Tile Collision

**Header**: `include/physics/TilePixelCollision.h`
**Namespace**: `pixelroot32::physics`
**Feature gate**: **none** — this header contains no `#if` of any kind. That is a real asymmetry against every neighbouring physics addition (spatial query, interaction triggers, depth sort are all flag-gated); per-pixel tile collision is always compiled in.

```cpp
bool isTilePixelSolid(const graphics::Sprite4bpp* tile, int px, int py,
                      int erodePx = 0) noexcept;

bool isWorldPixelSolid(const graphics::TileMap4bpp* tilemap,
                       const TileBehaviorLayer& layer,
                       int tileX, int tileY,
                       int px, int py,
                       uint8_t requiredFlags = TILE_SOLID,
                       int erodePx = 0) noexcept;
```

**`erodePx` semantics** — morphological erosion radius, default `0` = off. With `erodePx = k > 0` a pixel counts as solid only if the entire `(2*k+1)^2` square centred on it is opaque; a single transparent neighbour — **or a neighbour past the tile edge** — voids the pixel. It **shrinks the tile's collision silhouette, not the moving body**, so 1px branches and stray opaque pixels stop blocking passage while the tile's core stays solid.

**Cost** — O(1) at `erodePx = 0`; `(2k+1)^2` lookups at `erodePx = k`.

**Scope limit** — erosion is **per-tile**: it never reads neighbouring tiles. A wall built from several tiles therefore erodes 1px *at each tile seam*, producing seams the body can nick.

**`CollisionMode` / `kCollisionMode` are NOT engine API.** They are **example-owned**, defined in `examples/legend_of_clone/src/GameConstants.h` inside namespace `legend_of_clone`, with zero occurrences anywhere under `include/` or `src/`. Never emit them as engine symbols. The pattern they demonstrate — compile-time mode selection with `if constexpr` plus a game-owned `kTileSolidErosionPx = 1` — is a valid usage reference; `examples/legend_of_clone` is the place to read it.

### Multi-Hit Tile Consumption

**Header**: `include/physics/TileConsumptionHelper.h`
**Namespace**: `pixelroot32::physics`

`TileConsumptionConfig`, complete field list in declaration order:

```cpp
bool    updateTilemap       = true;   // update runtimeMask to hide consumed tiles
bool    logConsumption      = true;
bool    validateCoordinates = true;
uint8_t requiredHits        = 1;      // 1 = single-shot; >1 requires multiple applyHit() calls
```

```cpp
bool applyHit(pixelroot32::core::Actor* tileActor, uint16_t tileX, uint16_t tileY,
              uint8_t& remainingHits);

inline bool applyHitFromUserData(pixelroot32::core::Actor* tileActor, uintptr_t packedUserData,
                                 pixelroot32::core::Scene& scene, void* tilemap,
                                 uint8_t& remainingHits,
                                 const TileConsumptionConfig& config = TileConsumptionConfig());

inline bool applyHitFromCollision(pixelroot32::core::Actor* tileActor, uintptr_t packedUserData,
                                  pixelroot32::core::Scene& scene, void* tilemap,
                                  uint8_t& remainingHits,
                                  const TileConsumptionConfig& config = TileConsumptionConfig());
```

**Critical contract — the caller owns the hit counter.** `applyHit` decrements and reports the remaining count through the `remainingHits` out-param. **The engine does NOT store per-tile hit state anywhere.** When `remainingHits` reaches `0` the caller SHOULD invoke `consumeTile()` to finalize; nothing consumes the tile automatically. Persisting the counter per tile coordinate is entirely the game's job.

`tilemap` is `void*` in every signature above — it is a `TileMapGeneric*` passed as `void*` to avoid template instantiation in this header. Do not "fix" it to a typed pointer.

### KinematicActor (Character Controller)

**Header**: `include/physics/KinematicActor.h`
**Namespace**: `pixelroot32::physics`
**Inherits**: `pixelroot32::core::PhysicsActor`

> This skill is the **canonical owner of `KinematicActor`**. `pixelroot32-entity-actor` places the class in the hierarchy and defers here for the API — do not duplicate this section elsewhere.

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

// Top-surface floor validation (default true)
player.setStrictTopSurfaceFloor(false);  // opt out for custom edge-anchor mechanics
bool strict = player.isStrictTopSurfaceFloor();
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
- **Physics/grid limits**: `#ifndef` macros in `include/platforms/EngineConfig.h`, each mirrored under `pixelroot32::platforms::config::`. Defaults: `SPATIAL_GRID_CELL_SIZE` 32, `SPATIAL_GRID_MAX_ENTITIES_PER_CELL` 24, `SPATIAL_GRID_MAX_STATIC_PER_CELL` 12, `SPATIAL_GRID_MAX_DYNAMIC_PER_CELL` 12, `PHYSICS_MAX_ENTITIES` 64, `PHYSICS_MAX_CONTACTS` 128, `PHYSICS_MAX_PAIRS` 128 (alongside `MAX_ENTITIES` 64, `MAX_SCENES` 8, `MAX_LAYERS` 4).
- **ESP32 lowers exactly four**: `[base_esp32]` in `platformio.ini` overrides `SPATIAL_GRID_MAX_STATIC_PER_CELL` → 4, `SPATIAL_GRID_MAX_DYNAMIC_PER_CELL` → 4, `PHYSICS_MAX_CONTACTS` → 64, `PHYSICS_MAX_PAIRS` → 64. Every other limit keeps its default on ESP32.

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
13. **Top-surface floor validation**: With `strictTopSurfaceFloor` (default true), floor state and kinematic carry require horizontal overlap on the platform's top face — actors detach when walking off the top edge. Narrow hitboxes standing on platform edges (including one-way edges) are supported without being pushed off. Call `setStrictTopSurfaceFloor(false)` only for custom edge-anchor mechanics.
14. **Spatial query mask is one-directional**: `queryRadius`/`queryBox` test the caller-supplied `mask` against **`other->layer` only**. This is NOT the bidirectional layer/mask test `checkCollision()` performs — an actor whose own mask excludes the querier is still returned. Filter further yourself if you need symmetry.
15. **Spatial queries can cross scenes**: a query **may return static geometry registered by another simultaneously-alive scene's `CollisionSystem`/`SpatialGrid`**. This is documented, pre-existing, and unguarded — nothing scopes results to the querying scene. Validate the returned actors when more than one scene is alive.
16. **Spatial query and interaction triggers require physics**: `PIXELROOT32_ENABLE_SPATIAL_QUERY=1` and `PIXELROOT32_ENABLE_INTERACTION_TRIGGERS=1` both hard-require `PIXELROOT32_ENABLE_PHYSICS=1`; `include/platforms/PlatformDefaults.h` raises an `#error` otherwise. Both default to `0`.
17. **`setInteractionTracker` wiring is not automatic**: constructing an `InteractionTracker` is not enough. Without the explicit `collisionSystem.setInteractionTracker(&tracker)` call the tracker never sees a contact. The pointer is non-owning — outlive it.
18. **`TilePixelCollision.h` has no feature flag**: unlike every neighbouring physics addition, it is unconditionally compiled. There is no `PIXELROOT32_ENABLE_*` macro to guard calls with — do not invent one.
19. **`erodePx` erodes the tile, not the body**: erosion shrinks the *tile's* solid silhouette. It is per-tile only, so a multi-tile wall erodes at every tile seam. Cost is `(2k+1)^2` lookups per query at `erodePx = k`.
20. **The caller owns the multi-hit counter**: `applyHit` only decrements the `remainingHits` out-param. The engine stores no per-tile hit state and never calls `consumeTile()` for you — do that yourself when `remainingHits` hits 0.

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
