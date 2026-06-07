---
name: pixelroot32-scene-manager
description: Scene lifecycle management with stack-based scene navigation, Fade/Iris transitions, entity management with arena allocation, and CameraEffects integration. Use when structuring game screens, managing scene transitions, or organizing entity lifecycles.
license: MIT
compatibility: opencode>=0.1.0
metadata:
  domain: engine
  subsystem: core
  module: scene-manager
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
        // Draw entities sorted by render layer
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

// Entity array (sorted by render layer)
int count = scene.entityCount;   // Current entity count
Entity* e = scene.entities[i];   // Direct array access

// Arena allocation (placement new for zero-heap scenes)
scene.arena.init(memoryPool, POOL_SIZE);
Entity* player = arenaNew<Player>(scene.arena, args...);
```

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
- **Arena allocation**: `SceneArena` provides placement-new allocation from a pre-allocated buffer. No heap fragmentation. Call `arena.init()` during `Scene::init()`.
- **Scene stack is fixed**: `SceneManager::sceneStack[]` has fixed `MaxScenes` capacity (from `EngineConfig`). `pushScene()` silently fails when full.
- **`TransitionEffect` is Engine-owned**: The Engine creates and owns one `TransitionEffect`. `SceneManager` holds a non-owning pointer set via `setTransitionEffect()`.
- **`std::optional<Scene*>`**: `getCurrentScene()` returns `std::optional` — always check `value_or(nullptr)` or `has_value()`.

## Gotchas

1. **Scene pointer lifetime**: `setCurrentScene()`, `pushScene()` do NOT take ownership. Scenes must remain valid until `popScene()` or replacement.
2. **Transition state machine**: During `FadingOut`/`FadingIn`, the scene's `update()` is skipped (input blocking). `draw()` still runs so the transition effect has content to post-process.
3. **UI Manager lifecycle**: `UIManager` is a member of `Scene`. Widgets registered with `addElement()` must be unregistered before destruction to avoid dangling pointers.
4. **Physics in Scene**: `CollisionSystem` and `PhysicsScheduler` are conditionally compiled (`PIXELROOT32_ENABLE_PHYSICS`). Always guard with `#if`.
5. **Entity sorting**: `needsSorting` flag triggers layer-based sorting. Set it when adding/changing entity render layers.
6. **Framebuffer optimization**: If ALL stacked scenes return `false` from `shouldRedrawFramebuffer()`, the engine skips `draw()` and `present()` for that frame.
7. **Iris centers reset**: `TransitionEffect::init()` resets centers to -1. `SceneManager` stores and reapplies direction-specific centers.
8. **Idempotent `init()`**: `Scene::init()` calls `resetState()` first. N invocations must not leak resources or leave dangling pointers. Always call `Scene::init()` in overrides — never call `physicsScheduler.init()` directly.
9. **`resetState()` for owned resources**: Scenes with `unique_ptr` or external references must override `resetState()`, release owned resources **before** calling `Scene::resetState()` as the last operation.

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
    Player* p = arenaNew<Player>(arena, 50, 50);
    addEntity(p);
}

void MyScene::resetState() noexcept {
    Scene::resetState();   // arena.reset() + clearEntities() — invalidates arena allocations
}
```

## Agent Constraints
- **CRITICAL OWNERSHIP WARNING:** `SceneManager` does NOT own scene pointers. ALWAYS clean up the `SceneArena` and local objects when doing `popScene()` or destroying a scene to prevent memory leaks and dangling pointers.
