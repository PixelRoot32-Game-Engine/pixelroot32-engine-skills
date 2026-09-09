---
name: pixelroot32-scene-manager
description: Scene lifecycle management with stack-based scene navigation, Fade/Iris transitions, entity management with arena allocation, and CameraEffects integration. Use when structuring game screens, managing scene transitions, or organizing entity lifecycles.
license: MIT
compatibility: opencode>=0.1.0
metadata:
  domain: engine
  subsystem: core
  module: scene-manager
  engine_version: "1.9.0+unreleased"
  platform: cross-platform
---

## Overview

PixelRoot32 provides a Scene-based game architecture inspired by Godot Engine. `SceneManager` manages a stack of scenes with push/pop navigation and transition effects (Fade, Iris, DiagonalWipe). Each `Scene` owns an array of `Entity` instances with layer-based rendering, optional physics integration, UI system integration, and camera effects. The scene lifecycle: `init()` (idempotent) → `update()`/`draw()` loop → `resetState()` on re-init.

## Key APIs

### Scene (Base Class)

**Header**: `include/core/Scene.h`
**Namespace**: `pixelroot32::core`

```cpp
class GameScene : public Scene {
    void init() override {
        Scene::init();   // resetState() + physicsScheduler.init() — always call base first
        // Create entities, init arena, etc.
    }

    void resetState() noexcept override {
        // Release owned resources BEFORE base reset (unique_ptr, external refs)
        player.reset();
        Scene::resetState();   // clears entities, arena, collisionSystem
    }

    void update(unsigned long deltaTime) override {
        // Update entities, physics, game logic
        for (int i = 0; i < entityCount; i++) {
            if (entities[i]->isEnabled)
                entities[i]->update(deltaTime);
        }
    }

    void draw(Renderer& renderer) override {
        // Draw entities in sorted order (primary key = render layer)
        renderer.beginFrame();
        for (int i = 0; i < entityCount; i++) {
            if (entities[i]->isVisible)
                entities[i]->draw(renderer);
        }
        renderer.endFrame();
    }

    // Optional hooks
    void initUI() override { /* Add UI elements */ }
    void updateUI(unsigned long dt) override { /* Update UI */ }
    bool shouldRedrawFramebuffer() const override { return true; }

private:
    std::unique_ptr<Player> player;
};
```

### Entity Management

```cpp
// Add/remove entities
scene.addEntity(entity);
scene.removeEntity(entity);
scene.clearEntities();           // Remove all entities

// Entity array (sorted — render layer is the PRIMARY key; see Entity Sorting below)
int count = scene.entityCount;   // Current entity count
Entity* e = scene.entities[i];   // Direct array access

// Arena allocation (placement new for zero-heap scenes) — see SceneArena below
scene.arena.init(memoryPool, POOL_SIZE);
Entity* player = arenaNew<Player>(scene.arena, args...);  // nullptr if exhausted/gate off
```

### Entity Sorting

**Header**: `include/core/Scene.h`
**Namespace**: `pixelroot32::core`
**Feature gate**: **none** — the four members below sit in an **unguarded `protected` region** of `Scene`. They are **NOT** behind `PIXELROOT32_ENABLE_DEPTH_SORT`; only `Entity::depthKey` is flag-gated.

```cpp
using DepthComparator = bool (*)(Entity* a, Entity* b);   // raw function pointer, NOT std::function
DepthComparator depthComparator = nullptr;
bool depthSortEnabled = false;                            // when true, draw() re-sorts EVERY frame
void sortEntities();
```

The ordering predicate:

```cpp
inline bool shouldPrecede(Entity* key, Entity* at) const {
    const unsigned char la = at->getRenderLayer();
    const unsigned char lk = key->getRenderLayer();
    if (la != lk) return la > lk;                 // identical to previous behavior
    if (depthComparator == nullptr) return false; // identical to previous behavior
    return depthComparator(key, at);
}
```

Rules that follow from it:

- **`renderLayer` is still the primary key.** Nothing about layer ordering changed.
- **The comparator only breaks ties *within* a render layer.** It is consulted solely when two entities share a layer; it can never move an entity across layers.
- **With `depthComparator == nullptr` the behavior is byte-identical to before** the feature existed.
- **`depthSortEnabled = true` forces a full re-sort every frame in `draw()`, regardless of `needsSorting`.** Leave it `false` for static scenes and pay the sort only when entity depth actually changes per frame.
- **All four are `protected`.** A game sets them from inside its own `Scene`-derived `init()` — not from outside the scene.

```cpp
class IsoScene : public pixelroot32::core::Scene {
    static bool byDepth(Entity* a, Entity* b) { /* game-owned */ }

    void init() override {
        Scene::init();
        depthComparator = &IsoScene::byDepth;   // protected member, set from inside
        depthSortEnabled = true;                // re-sort every frame
    }
};
```

Choosing a comparator — and the `Entity::depthKey` values it reads — is projection work: see **`pixelroot32-projection`**. The `depthKey` field itself (guard and memory cost) is documented in **`pixelroot32-entity-actor`**.

### Room Transitions

**Header**: `include/core/Scene.h`
**Namespace**: `pixelroot32::core`
**Feature gate**: `PIXELROOT32_ENABLE_GAMEPLAY_ROOM` — default **0**.

```cpp
virtual void onRoomEnter(int fromIdx, int toIdx) { (void)fromIdx; (void)toIdx; }
```

Virtual hook with an empty default body; override it in a `Scene` subclass to react to room changes. **`fromIdx` is `0xFFFF` on first entry** — the sentinel is documented as `0xFFFF` while both parameters are declared plain `int`, so compare against `0xFFFF` explicitly rather than assuming a negative value.

**The engine NEVER calls `onRoomEnter`.** Overriding it is not enough — the override silently never fires. `Scene::setRoomGraph()` only stores the pointer, and `src/core/Scene.cpp` contains no reference to `onRoomEnter` or the room-graph member. The header comment in `include/core/Scene.h` claiming the base invokes the hook through the graph's `onEnter` callback **is stale and wrong**; trust this skill, not that comment.

The game wires it manually with a static trampoline:

```cpp
class RoomScene : public pixelroot32::core::Scene {
    static void onRoomEnterCallback(int fromIdx, int toIdx, void* ctx) {
        static_cast<RoomScene*>(ctx)->onRoomEnter(fromIdx, toIdx);
    }

    void init() override {
        Scene::init();
        rooms_.setOnEnter(onRoomEnterCallback, this);   // REQUIRED — nothing fires without it
    }

    void onRoomEnter(int fromIdx, int toIdx) override {
        if (fromIdx == 0xFFFF) { /* first entry */ }
    }
};
```

Shipped references for this wiring: `examples/legend_of_clone/src/TopDownScene.cpp` and `examples/room_screen/src/RoomScreenScene.cpp`.

`RoomGraph`, `RoomData`, `RoomLayer`, `buildRoomGraph` and `setOnEnter` live in **`pixelroot32-gameplay-framework`**; this skill only owns the `Scene` hook and the fact that it must be wired by hand.

### SceneArena

**Header**: `include/core/Scene.h` — `SceneArena` is declared **inside `Scene.h`**; there is **no `include/core/SceneArena.h`**
**Namespace**: `pixelroot32::core`
**Feature gate**: `PIXELROOT32_ENABLE_SCENE_ARENA` — tested with `#ifdef`, **never defined by the engine or `platformio.ini`** (default OFF). The game must pass `-D PIXELROOT32_ENABLE_SCENE_ARENA` on its own build line. With the gate off, `init()`/`reset()` are no-ops, `allocate()` always returns `nullptr`, and `arenaNew<T>()` always returns `nullptr`.

`struct SceneArena` has public members `unsigned char* buffer; std::size_t capacity; std::size_t offset;` and exactly one constructor: the default (zero-init) one.

```cpp
// Full public API
void  init(void* memory, std::size_t size);
void  reset();
void* allocate(std::size_t size, std::size_t alignment);   // alignment is REQUIRED — no default arg

// Free function template in the same header
template<typename T, typename... Args>
T* arenaNew(SceneArena& arena, Args&&... args);   // allocate(sizeof(T), alignof(T)) + placement new

// Usage
alignas(alignof(max_align_t)) unsigned char pool[4096];

SceneArena arena;                    // default ctor ONLY — no SceneArena(buffer, size) form
arena.init(pool, sizeof(pool));

void* raw = arena.allocate(sizeof(Player), alignof(Player));  // there is no alloc()
Player* p = arenaNew<Player>(arena, 50, 50);                  // nullptr when exhausted
if (p == nullptr) { /* out of arena space */ }

arena.reset();   // rewinds offset to 0
```

`Scene::arena` is a plain `SceneArena` member of `Scene`, not wrapped in any `#if` — the gate only changes runtime behavior. **Destructors are NEVER run**: `reset()` just rewinds the offset, so only trivially destructible types are safe to arena-allocate.

### SceneManager

**Header**: `include/core/SceneManager.h`
**Namespace**: `pixelroot32::core`

```cpp
SceneManager manager;

// Direct scene switching
manager.setCurrentScene(&menuScene);
manager.pushScene(&pauseScene);   // Push overlay
manager.popScene();               // Pop back to previous

// Query
Scene* current = manager.getCurrentScene().value_or(nullptr);
int count = manager.getSceneCount();
bool empty = manager.isEmpty();
bool redraw = manager.aggregateShouldRedrawFramebuffer();

// Update/draw loop
manager.adviseFramebufferBeforeBeginFrame(renderer);
manager.update(deltaTime);
manager.draw(renderer);
```

### Scene Transitions

```cpp
// Fade transition (no center needed)
manager.transitionToScene(&gameScene, TransitionType::Fade, 500);

// Iris transition (with direction-specific centers)
manager.transitionToScene(&gameScene, TransitionType::Iris, 300,
                          120, 120,   // Iris out center
                          120, 120);  // Iris in center

// DiagonalWipe transition
manager.transitionToScene(&gameScene, TransitionType::DiagonalWipe, 400);

// State query
bool transitioning = manager.isTransitioning();
TransitionState state = manager.getTransitionState();
// Idle → FadingOut → SceneSwap → FadingIn → Idle
```

### Touch Event Pipeline

```cpp
// In Engine loop (Scene provides deterministic pipeline)
void Scene::processTouchEvents(TouchEvent* events, uint8_t count) {
    // 1. UI: UIManager::processEvents — marks consumed
    // 2. For unconsumed: onUnconsumedTouchEvent()
}

// Override in scene
void onUnconsumedTouchEvent(const TouchEvent& event) override {
    if (event.getType() == TouchEventType::Click) {
        handleTap(event.x, event.y);
    }
}
```

### Camera Effects Integration

```cpp
// Scene provides access to CameraEffects
#if PIXELROOT32_ENABLE_CAMERA_EFFECTS
    // Access via Scene member
    cameraEffects.triggerShake(toScalar(4.0f), 200);
    cameraEffects.triggerPunch(toScalar(3.0f), 100, Vector2::RIGHT());

    // Get offset for renderer
    Vector2 offset = getCameraEffectOffset();
    renderer.setDisplayOffset(-camX + (int)offset.x, -camY + (int)offset.y);
#endif
```

### Framebuffer Optimization

```cpp
// Scene controls framebuffer redraw
bool shouldRedrawFramebuffer() const override {
    // Return false when nothing has changed
    return camera.hasMoved() || animFrameChanged;
}

// Advise before beginFrame for dirty region alignment
void adviseFramebufferBeforeBeginFrame(Renderer& renderer) override {
    staticCache.adviseFramebufferBeforeBeginFrame(renderer, ...);
}
```

## Composition Patterns

### Full game loop integration
```
Engine::runFrame():
  ├── sceneManager.adviseFramebufferBeforeBeginFrame(renderer)
  ├── renderer.beginFrame()
  ├── sceneManager.update(deltaTime)
  ├── sceneManager.draw(renderer)
  └── renderer.endFrame()
```

### Scene with physics + UI + camera effects
```
class LevelScene : public Scene {
    void init() override {
        Scene::init();         // resetState() + physicsScheduler.init()
        initUI();              // Set up UI elements
        arena.init(pool, size);
        player = arenaNew<Player>(arena, x, y);
        addEntity(player);
    }

    void resetState() noexcept override {
        player = nullptr;      // arena allocations invalidated by base reset
        Scene::resetState();
    }

    void update(dt) override {
        // Update entities first
        Scene::update(dt);
        // Then physics
        #if PIXELROOT32_ENABLE_PHYSICS
            physicsScheduler.update(dt * 1000, collisionSystem);
        #endif
        // Then camera
        camera.followTarget(player->getPosition());
        #if PIXELROOT32_ENABLE_CAMERA_EFFECTS
            cameraEffects.update(dt);
        #endif
        updateUI(dt);
    }
};
```

### Pause overlay via scene stack
```
gameScene → pushScene(pauseOverlay)
  ├── pauseOverlay.init(): Set up semi-transparent menu
  ├── pauseOverlay.update(dt): Handle resume/quit buttons
  └── gameScene.popScene(): Resume
```

## ESP32 Constraints

- **Entity array is fixed**: `Scene::entities[]` is a fixed-size array (`MaxEntities` from `EngineConfig`). Check `entityCount` before adding.
- **Core limits**: `#ifndef` macros in `include/platforms/EngineConfig.h`, mirrored under `pixelroot32::platforms::config::` — `MAX_SCENES` 8, `MAX_LAYERS` 4, `MAX_ENTITIES` 64. ESP32 (`[base_esp32]` in `platformio.ini`) does **not** lower any of these three; it only lowers four physics/grid limits (see `pixelroot32-physics`).
- **Arena allocation**: `SceneArena` (declared in `include/core/Scene.h`) provides placement-new allocation from a pre-allocated buffer. No heap fragmentation. Call `arena.init(memory, size)` during `Scene::init()`. Requires `-D PIXELROOT32_ENABLE_SCENE_ARENA` on the game's build line — the engine never defines it, and without it `arenaNew<T>()` returns `nullptr`.
- **Scene stack is fixed**: `SceneManager::sceneStack[]` has fixed `MaxScenes` capacity (from `EngineConfig`). `pushScene()` silently fails when full.
- **`TransitionEffect` is Engine-owned**: The Engine creates and owns one `TransitionEffect`. `SceneManager` holds a non-owning pointer set via `setTransitionEffect()`.
- **`std::optional<Scene*>`**: `getCurrentScene()` returns `std::optional` — always check `value_or(nullptr)` or `has_value()`.

## Gotchas

1. **Scene pointer lifetime**: `setCurrentScene()`, `pushScene()` do NOT take ownership. Scenes must remain valid until `popScene()` or replacement.
2. **Transition state machine**: During `FadingOut`/`FadingIn`, the scene's `update()` is skipped (input blocking). `draw()` still runs so the transition effect has content to post-process.
3. **UI Manager lifecycle**: `UIManager` is a member of `Scene`. Widgets registered with `addElement()` must be unregistered before destruction to avoid dangling pointers.
4. **Physics in Scene**: `CollisionSystem` and `PhysicsScheduler` are conditionally compiled (`PIXELROOT32_ENABLE_PHYSICS`). Always guard with `#if`.
5. **Entity sorting**: `needsSorting` triggers a sort whose **primary key is the render layer** — it is no longer *purely* layer-based. Set `needsSorting` when adding entities or changing render layers. A `depthComparator` breaks ties only between entities that share a layer.
6. **Sorting members are unguarded and `protected`**: `DepthComparator`, `depthComparator`, `depthSortEnabled` and `sortEntities()` are NOT behind `PIXELROOT32_ENABLE_DEPTH_SORT` (only `Entity::depthKey` is) and are `protected` — set them from inside a `Scene`-derived `init()`, never from outside. `DepthComparator` is a raw function pointer (`bool (*)(Entity*, Entity*)`), so no capturing lambda can be assigned to it.
7. **`depthSortEnabled` re-sorts every frame**: when `true`, `draw()` sorts regardless of `needsSorting`. That is a per-frame cost on a fixed 64-entity array — enable it only for scenes whose depth order genuinely changes each frame.
8. **`onRoomEnter` first-entry sentinel**: `fromIdx` is `0xFFFF` on the first entry, even though the parameters are plain `int`. Test for `0xFFFF`; do not assume `-1` or `< 0`. The hook only exists under `PIXELROOT32_ENABLE_GAMEPLAY_ROOM` (default 0).
9. **`onRoomEnter` is NEVER invoked by the engine**: overriding it alone does nothing — the override silently never fires. `Scene::setRoomGraph()` only assigns the pointer, and `src/core/Scene.cpp` contains zero references to `onRoomEnter` or the room-graph member. The game MUST wire it itself via `RoomGraph::setOnEnter(callback, context)` with a static trampoline that forwards to the scene. **The engine's own header comment claiming "the base invokes `onRoomEnter` via the graph's `onEnter` callback" is stale and wrong** — do not "correct" this skill back from that comment. Working references: `examples/legend_of_clone/src/TopDownScene.cpp` and `examples/room_screen/src/RoomScreenScene.cpp`, both calling `rooms_.setOnEnter(onRoomEnterCallback, this)`.
10. **Framebuffer optimization**: If ALL stacked scenes return `false` from `shouldRedrawFramebuffer()`, the engine skips `draw()` and `present()` for that frame.
11. **Iris centers reset**: `TransitionEffect::init()` resets centers to -1. `SceneManager` stores and reapplies direction-specific centers.
12. **Idempotent `init()`**: `Scene::init()` calls `resetState()` first. N invocations must not leak resources or leave dangling pointers. Always call `Scene::init()` in overrides — never call `physicsScheduler.init()` directly.
13. **`resetState()` for owned resources**: Scenes with `unique_ptr` or external references must override `resetState()`, release owned resources **before** calling `Scene::resetState()` as the last operation.

## Common Patterns

### Splash → Menu → Game flow
```cpp
// Initialize
SceneManager manager;
SplashScene splash;
MenuScene menu;
GameScene game;

manager.setCurrentScene(&splash);

// After splash timer:
manager.transitionToScene(&menu, TransitionType::Fade, 300);

// On "Start" button:
manager.transitionToScene(&game, TransitionType::Iris, 500, 120, 120, 120, 120);
```

### Arena-allocated entity in scene
```cpp
// Reserve pool in scene
alignas(alignof(max_align_t)) unsigned char pool[4096];

void MyScene::init() {
    Scene::init();
    arena.init(pool, sizeof(pool));
    Player* p = arenaNew<Player>(arena, 50, 50);   // nullptr if exhausted or gate off
    if (p == nullptr) return;
    addEntity(p);
}

void MyScene::resetState() noexcept {
    Scene::resetState();   // arena.reset() + clearEntities() — invalidates arena allocations
}
```

## Agent Constraints
- **CRITICAL OWNERSHIP WARNING:** `SceneManager` does NOT own scene pointers. ALWAYS clean up the `SceneArena` and local objects when doing `popScene()` or destroying a scene to prevent memory leaks and dangling pointers.
