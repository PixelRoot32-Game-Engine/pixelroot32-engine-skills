---
name: pixelroot32-entity-actor
description: Core entity/actor hierarchy with Entity as base class, Actor with collision layers/masks, PhysicsActor with 4 body types (Static, Kinematic, Rigid, Sensor), Fixed16 math, and component model. Use when creating game objects, defining collision behavior, or extending the entity system.
license: MIT
compatibility: opencode>=0.1.0
metadata:
  domain: engine
  subsystem: core
  module: entity-actor
  platform: cross-platform
---

## Overview

PixelRoot32 uses a Godot-inspired hierarchy: `Entity` → `Actor` → `PhysicsActor`, with specialized actor types for physics. `Entity` is the abstract base for all game objects with position/size, lifecycle (update/draw), and layer-based rendering. `Actor` adds collision layers and masks. `PhysicsActor` adds velocity, gravity, body types, and collision shapes. The system uses adaptable `Scalar` type (float on PC, Fixed16 Q16.16 on ESP32).

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

### Physics Body Types

| BodyType | Gravity | Forces | Manual Move | Collision Response |
|----------|---------|--------|-------------|-------------------|
| `STATIC` | No | No | No | Blocks others |
| `KINEMATIC` | No | No | Via `setVelocity` | Stops at obstacles |
| `RIGID` | Yes | Yes | Yes | Full physics simulation |
| Sensor flag | No | No | Yes | Events only, no response |

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
class Player : public PhysicsActor {
    Player() : PhysicsActor(0, 0, 16, 16) {
        setBodyType(PhysicsBodyType::KINEMATIC);
        setCollisionLayer(PLAYER_LAYER);
        setCollisionMask(WALL_LAYER | ENEMY_LAYER);
        setWorldBounds(levelWidth, levelHeight);
    }

    void update(dt) override {
        // Manual velocity for KINEMATIC
        setVelocity(inputX * speed, velocity.y);
        PhysicsActor::update(dt);
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
