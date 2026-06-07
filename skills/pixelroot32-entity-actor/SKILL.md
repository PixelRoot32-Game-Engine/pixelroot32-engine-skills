---
name: pixelroot32-entity-actor
description: Core entity/actor hierarchy with Entity as base class, Actor with collision layers/masks, PhysicsActor with 4 body types (Static, Kinematic, Rigid, Sensor), KinematicActor for character controllers, Fixed16 math, and component model. Use when creating game objects, defining collision behavior, or extending the entity system.
license: MIT
compatibility: opencode>=0.1.0
metadata:
  domain: engine
  subsystem: core
  module: entity-actor
  platform: cross-platform
---

## Overview

PixelRoot32 uses a Godot-inspired hierarchy: `Entity` → `Actor` → `PhysicsActor` → `KinematicActor`, with specialized actor types for physics. `Entity` is the abstract base for all game objects with position/size, lifecycle (update/draw), and layer-based rendering. `Actor` adds collision layers and masks. `PhysicsActor` adds velocity, gravity, body types, and collision shapes. `KinematicActor` adds script-driven movement with `moveAndSlide` for character controllers. The system uses adaptable `Scalar` type (float on PC, Fixed16 Q16.16 on ESP32).

## Key APIs

### Entity (Base Class)

**Header**: `include/core/Entity.h`
**Namespace**: `pixelroot32::core`

```cpp
// Abstract base — all game objects derive from Entity
class MyEntity : public Entity {
    MyEntity(Vector2 pos, int w, int h)
        : Entity(pos, w, h, EntityType::GENERIC) {}

    void update(unsigned long deltaTime) override { /* logic */ }
    void draw(Renderer& renderer) override { /* render */ }
};

// Properties
entity.position.x, entity.position.y  // Scalar coordinates
entity.width, entity.height           // Dimensions (int)
entity.isVisible = true;              // Controls draw() invocation
entity.isEnabled = true;              // Controls update() invocation
entity.setRenderLayer(3);             // 0 to MaxLayers-1
entity.getRenderLayer();
```

### Actor (Collision-aware Entity)

**Header**: `include/core/Actor.h`
**Namespace**: `pixelroot32::core`

```cpp
class MyActor : public Actor {
    MyActor(float x, float y, int w, int h) : Actor(x, y, w, h) {
        setCollisionLayer(DefaultLayers::kAll);
        setCollisionMask(DefaultLayers::kAll);
    }

    Rect getHitBox() override { return {position, width, height}; }
    void onCollision(Actor* other) override {
        // Notification only — no physics changes
        if (other->isInLayer(PLAYER_LAYER)) { /* hit player */ }
    }
};

// Collision configuration
actor.setCollisionLayer(0x0001);          // This actor's layer
actor.setCollisionMask(0x0002);           // Layers it collides with
actor.isInLayer(0x0001);                  // Query layer membership
actor.isPhysicsBody();                    // Returns false for Actor
```

### PhysicsActor (Full Physics Body)

**Header**: `include/core/PhysicsActor.h`
**Namespace**: `pixelroot32::core`

```cpp
PhysicsActor actor(100, 100, 16, 16);

// Body type
actor.setBodyType(PhysicsBodyType::KINEMATIC);
PhysicsBodyType type = actor.getBodyType();

// Velocity
actor.setVelocity(100.0f, 0.0f);        // Move right
actor.setVelocity(Scalar(100), Scalar(0));
pixelroot32::math::Vector2 vel = actor.getVelocity();

// Physics properties
actor.setMass(2.0f);
actor.setGravityScale(toScalar(1.0f));
actor.setRestitution(toScalar(0.5f));    // Bounciness
actor.setFriction(toScalar(0.1f));

// World bounds
actor.setWorldBounds(240, 240);
actor.setLimits(0, 0, 240, 240);         // Custom LimitRect
WorldCollisionInfo info = actor.getWorldCollisionInfo();

// Collision shape
actor.setShape(CollisionShape::CIRCLE);
actor.setRadius(toScalar(8.0f));         // Auto-sets width/height

// Flags
actor.setSensor(true);                   // Trigger (events only)
actor.setOneWay(true);                   // One-way platform
actor.setBounce(true);                   // Bounce on collision

// User data
actor.setUserData((void*)tileData);
void* data = actor.getUserData();
```

### KinematicActor (Character Controller)

**Header**: `include/physics/KinematicActor.h`
**Namespace**: `pixelroot32::physics`
**Inherits**: `pixelroot32::core::PhysicsActor`

Use for player characters and script-driven movement. **Do not** use a generic `PhysicsActor` with `setBodyType(KINEMATIC)` for platformer-style slide/snap — derive from `KinematicActor` instead.

```cpp
class Player : public pixelroot32::physics::KinematicActor {
    Player() : KinematicActor(0, 0, 16, 16) {
        setCollisionLayer(PLAYER_LAYER);
        setCollisionMask(WALL_LAYER | ENEMY_LAYER);
    }

    void update(unsigned long deltaTime) override {
        float dt = deltaTime * 0.001f;
        Vector2 vel = getVelocity();
        vel.y += toScalar(gravity * dt);
        vel.x = toScalar(inputX * speed);

        bool jumpThisFrame = wantsJump && is_on_floor();
        if (jumpThisFrame) {
            vel.y = toScalar(-jumpImpulse);
            clearFloorVelocity();
        }

        Vector2 snap = jumpThisFrame ? Vector2{} : Vector2{0, MIN_SNAP};
        vel = moveAndSlide(vel, toScalar(dt), {0, -1}, SnapPolicy::Step, snap);
        setVelocity(vel.x, vel.y);
    }
};

// Contact queries (current frame only)
actor.is_on_floor();
actor.is_on_wall();
actor.is_on_ceiling();
actor.getFloorVelocity();    // KINEMATIC floor platform velocity
actor.clearFloorVelocity();  // call on jump
```

### Physics Body Types

| BodyType | Gravity | Forces | Manual Move | Collision Response |
|----------|---------|--------|-------------|-------------------|
| `STATIC` | No | No | No | Blocks others |
| `KINEMATIC` | No | No | Via `setVelocity` | Stops at obstacles |
| `RIGID` | Yes | Yes | Yes | Full physics simulation |
| Sensor flag | No | No | Yes | Events only, no response |

**Note**: `PhysicsBodyType::KINEMATIC` is a body-type flag on `PhysicsActor`. For character controllers with slide/snap, use the **`KinematicActor` subclass** instead.

### CollisionShape

| Shape | Use | Notes |
|-------|-----|-------|
| `AABB` | Default | Axis-Aligned Bounding Box |
| `CIRCLE` | Circular bodies | Set radius; auto-sets width/height to diameter |

### Log

**Header**: `include/core/Log.h`
**Namespace**: `pixelroot32::core`

```cpp
PIXELROOT32_LOG("Player position: %d, %d", (int)pos.x, (int)pos.y);
PIXELROOT32_LOG_ERROR("Entity not found: %d", entityId);
```

## Composition Patterns

### Custom game actor with physics
```
class Player : public KinematicActor {
    Player() : KinematicActor(0, 0, 16, 16) {
        setCollisionLayer(PLAYER_LAYER);
        setCollisionMask(WALL_LAYER | ENEMY_LAYER);
        setWorldBounds(levelWidth, levelHeight);
    }

    void update(dt) override {
        float dtSec = dt * 0.001f;
        Vector2 vel = getVelocity();
        vel.y += toScalar(gravity * dtSec);
        vel.x = toScalar(inputX * speed);
        vel = moveAndSlide(vel, toScalar(dtSec), {0, -1}, SnapPolicy::Step,
                           jumpThisFrame ? Vector2{} : Vector2{0, MIN_SNAP});
        setVelocity(vel.x, vel.y);
    }

    void onCollision(Actor* other) override {
        if (other->isInLayer(COIN_LAYER)) collectCoin();
    }
};
```

### Sensor trigger zone
```
class TriggerZone : public PhysicsActor {
    TriggerZone() : PhysicsActor(x, y, w, h) {
        setSensor(true);                    // No physics response
        setCollisionLayer(TRIGGER_LAYER);
        setCollisionMask(PLAYER_LAYER);
    }
    void onCollision(Actor* other) override {
        activateEvent();
    }
};
```

## ESP32 Constraints

- **No `float` on no-FPU cores**: `Scalar` maps to `int32_t` (Fixed16 Q16.16) on ESP32-C3. Use `toScalar(float)`, `toFloat(Scalar)` for conversion at non-critical boundaries.
- **`!std::is_same<Scalar, float>` constructors**: Additional constructors exist for `float` arguments on fixed-point builds only (to avoid ambiguity when Scalar IS float).
- **No RTTI**: Use `EntityType` enum for type discrimination. Avoid `dynamic_cast`.
- **Packed flags**: `PhysicsActor::physicsFlags` packs sensor/one-way/bounce into a single `uint8_t`. Use `setSensor()`, `setOneWay()`, `setBounce()` — never manipulate bits directly.
- **User data**: Engine does NOT manage `userData` lifetime. The caller is fully responsible.

## Gotchas

1. **Previous position sync**: When setting position manually, call `setPosition(Vector2)` (not `position = ...`) to sync `previousPosition` for crossing detection.
2. **World collision reset**: `resetWorldCollisionInfo()` must be called at the start of each frame (done automatically by PhysicsActor::update()).
3. **Sensor actors**: `isPhysicsBody()` returns `true` for all PhysicsActors, even sensors. Distinguish via `isSensor()`.
4. **One-way platforms**: Only block from one direction (above → below pass-through). Use `setOneWay(true)` on static bodies.
5. **World bounds vs LimitRect**: If both are set, `LimitRect` takes priority. World bounds are used only when no custom LimitRect is set.
6. **Bounce flag**: `setBounce(true)` reflects velocity on static contact. `setBounce(false)` zeroes velocity on contact.
7. **Render layer 1**: Default for Entity. UI elements default to layer 2. Higher layers draw on top.
8. **Character controllers use `KinematicActor`**: Do not call `PhysicsActor::update()` for slide/snap movement — use `moveAndSlide()` and assign the returned velocity.
9. **`is_on_*` is frame-local**: `is_on_floor()`, `is_on_wall()`, `is_on_ceiling()` reflect only the last `moveAndSlide` call in the current frame.

## Common Patterns

### Static platform
```cpp
PhysicsActor platform(100, 150, 32, 8);
platform.setBodyType(PhysicsBodyType::STATIC);
platform.setCollisionLayer(WALL_LAYER);
platform.setCollisionMask(PLAYER_LAYER);
```

### Rigid body with bounce
```cpp
PhysicsActor ball(50, 50, 8, 8);
ball.setBodyType(PhysicsBodyType::RIGID);
ball.setShape(CollisionShape::CIRCLE);
ball.setRadius(toScalar(4));
ball.setRestitution(toScalar(0.8f));
ball.setBounce(true);
```

## Agent Constraints
- **Allocation:** NEVER use `new` or `malloc` to instantiate Actors in the game loop. YOU MUST use `SceneArena` or fixed arrays.
- **Testing:** If you modify `update` or core logic, YOU MUST run a cross-test or request the user to test the logic.
