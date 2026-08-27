---
name: pixelroot32-cpp-code-generation
description: Generate C++17 code following PixelRoot32 engine conventions, naming patterns, architectural practices, and Doxygen documentation standards. Use when writing or refactoring engine or game C++ source, choosing namespaces and include paths, documenting classes and methods, or routing to a subsystem-specific skill.
license: MIT
compatibility: opencode>=0.1.0
metadata:
  domain: engine
  language: cpp
  platform: cross-platform
  engine_version: "1.9.0+unreleased"
---

## Overview

Generate C++17 code for PixelRoot32 Game Engine following documented conventions. This skill ensures code consistency with the engine's architecture, style, and patterns.

## Source of Truth

All code generation must reference:
- `docs/STYLE_GUIDE.md` - Primary coding conventions
- `docs/ARCHITECTURE.md` - Layer hierarchy and component relationships
- `docs/API_REFERENCE.md` - Public API definitions

## Naming Conventions

### Required Rules

- **Classes/structs**: PascalCase (e.g., `PlayerActor`, `GameScene`)
- **Methods/functions**: camelCase (e.g., `update()`, `drawSprite()`)
- **Variables/members**: camelCase (e.g., `deltaTime`, `playerSprite`)
- **No Hungarian notation**: Do not use type prefixes
- **No member prefixes**: Avoid `m_` or `_` prefixes for members

### Namespace Usage

- **Public API namespaces** (stable):
  - `pixelroot32::core`
  - `pixelroot32::graphics`
  - `pixelroot32::graphics::ui`
  - `pixelroot32::input`
  - `pixelroot32::physics`
  - `pixelroot32::math`
  - `pixelroot32::gameplay`
  - `pixelroot32::audio` (when `PIXELROOT32_ENABLE_AUDIO=1`)

Notes:
- `pixelroot32::math` also carries `ProjectionSpec`, `CellRange` and the projection free functions (see `pixelroot32-projection`).
- `pixelroot32::graphics::particles` exists for emitters and presets (see `pixelroot32-particles`).
- `pixelroot32::gameplay` holds the reusable gameplay building blocks (see `pixelroot32-gameplay-framework`).

- **Internal namespaces** (do NOT use directly):
  - `pixelroot32::platform`
  - `pixelroot32::internal`
  - `pixelroot32::detail`

- **Alias recommended** for internal implementation:
  ```cpp
  namespace pr32 = pixelroot32;
  ```

## C++17 Requirements

### Mandatory Patterns

- Use `std::unique_ptr` for exclusive ownership
- Use `std::string_view` for non-owning string parameters (NOT `const std::string&`)
- Use `std::optional` for values that may not exist
- Use `[[nodiscard]]` for functions where return value must not be ignored
- Use `constexpr` for compile-time constants
- Use `if constexpr` for compile-time branching

### Forbidden Patterns

- Avoid RTTI and exceptions
- Avoid `new` or `malloc` in game loop (`update` or `draw`)
- Avoid `std::string` copies; use `std::string_view`
- Avoid dynamic allocation in hot paths

## Class Structure Order

Always order class members as:
1. Public members
2. Protected members
3. Private members

## Include Guidelines

- User code includes headers ONLY from `include/`
- Headers in `include/` may include `src/` headers
- Source files in `src/` must NOT include `include/` headers

## Code Generation Examples

### Entity Definition

```cpp
// Correct
class Player : public pixelroot32::core::Actor {
public:
    void update(unsigned long deltaTime) override {
        // Movement logic
    }
    
    void draw(pixelroot32::graphics::Renderer& renderer) override {
        renderer.drawSprite(playerSprite, static_cast<int>(x), static_cast<int>(y), Color::White);
    }
    
private:
    pixelroot32::graphics::Sprite playerSprite;
};
```

### Scene Definition

```cpp
class GameScene : public pixelroot32::core::Scene {
public:
    void init() override {
        Scene::init();   // idempotent — resetState() + physicsScheduler.init()
        player = std::make_unique<Player>(100, 100, 16, 16);
        addEntity(player.get());
    }

    void resetState() noexcept override {
        player.reset();          // release BEFORE base reset
        Scene::resetState();
    }

private:
    std::unique_ptr<Player> player;
};
```

### Scalar Usage

```cpp
// CORRECT - Use Scalar for all math
#include <math/Scalar.h>
using namespace pixelroot32::math;

Scalar speed = toScalar(2.5f);
Vector2 velocity = {toScalar(0.0f), toScalar(0.0f)};

// Convert to int only at render time
renderer.drawSprite(sprite, static_cast<int>(x), static_cast<int>(y), Color::White);
```

## Subsystem Skills

This table is the **canonical routing table** for the skill catalog. Other cross-cutting skills (`pixelroot32-memory-optimization`, `pixelroot32-testing`) must point here instead of restating it.

For subsystem-specific code generation, use the specialized skills:

| Subsystem | Skill | Use When |
|-----------|-------|----------|
| Camera | `pixelroot32-camera2d` | Camera logic, effects, parallax |
| Audio | `pixelroot32-audio` | Sound, music, voice pooling |
| Rendering | `pixelroot32-sprite-renderer` | Sprites, tilemaps, dirty regions |
| Physics | `pixelroot32-physics` | Collision, actors, spatial grid |
| UI | `pixelroot32-ui-system` | Layouts, widgets, touch integration |
| Scenes | `pixelroot32-scene-manager` | Scene lifecycle, transitions |
| Input | `pixelroot32-touch-input` | Touch events, state machine |
| Particles | `pixelroot32-particles` | Emitters, presets |
| Entities | `pixelroot32-entity-actor` | Actor types, lifecycle |
| Projection | `pixelroot32-projection` | Isometric, oblique or any non-axis-aligned view: `ProjectionSpec`, cell↔screen math, depth sorting, projected `drawTileMap`, projected camera bounds |
| Gameplay framework | `pixelroot32-gameplay-framework` | `pixelroot32::gameplay` building blocks: `GridSpec`/`GridMotion` grid-locked movement, `RoomGraph`/`RoomData`/`RoomLayer`/`buildRoomGraph` screen-by-screen rooms, `StateMachine`, `ObjectPool<T,N>`, `GameplayEventBus`, `InteractionComponent`/`InteractionTracker` |

For memory patterns and ESP32 constraints, see `pixelroot32-memory-optimization`. For creating a new game project from scratch (PlatformIO scaffold, platform entry points, minimal scene), see `pixelroot32-game-scaffold`.

## Logging Integration

Use unified logging when `PIXELROOT32_DEBUG_MODE` is defined:

```cpp
#include <core/Log.h>
using namespace pixelroot32::core::logging;

log("Player position: %d, %d", static_cast<int>(x), static_cast<int>(y));
log(LogLevel::Warning, "Low memory: %d bytes", freeRAM);
```

## Documentation (Doxygen)

Documentation is part of code generation: every new or edited public class, struct or method ships with its Doxygen block.

### Rules

- **Location**: documentation lives in the header (`.h`), never in the implementation (`.cpp`). The header is the single source of truth.
- **Format**: `/** ... */` block comments.
- **Classes/structs**: `@class` (or `@struct`) plus `@brief`.
- **Methods**: `@brief` always, one `@param` per parameter, and `@return` whenever the return type is not `void`.
- **Language**: documentation is written in English even when the prompt is in another language, unless explicitly requested otherwise.
- Defaulted virtuals (e.g. an empty-by-default hook such as `flushPendingTransfers()`) are documented like any other method — state the contract for overriders; subsystem semantics belong to the owning skill (for `DrawSurface`, `pixelroot32-sprite-renderer`).

### Example

Reference header: `include/graphics/DrawSurface.h`.

```cpp
/**
 * @class DrawSurface
 * @brief Abstract interface for platform-specific drawing operations.
 *
 * This class defines the contract for any graphics driver.
 */
class DrawSurface {
public:
    /**
     * @brief Draws a filled rectangle.
     * @param x Top-left X coordinate.
     * @param y Top-left Y coordinate.
     * @param width Width of the rectangle.
     * @param height Height of the rectangle.
     * @param color The fill color in RGB565 format.
     */
    virtual void drawFilledRectangle(int x, int y, int width, int height, uint16_t color) = 0;

    /**
     * @brief Checks if the surface is initialized.
     * @return true if initialized, false otherwise.
     */
    virtual bool isInitialized() const = 0;
};
```

### Generating the docs

```bash
# Run Doxygen generation (if configured)
doxygen Doxyfile
```

For subsystem API details and composition patterns while documenting, route through the canonical routing table above.

## Constraints

- Do NOT generate code that violates documented APIs
- Do NOT assume un documented behavior
- Do NOT use internal namespaces
- Do NOT create runtime allocations in game loop
- Always use fully-qualified names in headers

## Agent Constraints
- **Language Restrictions:** YOU MUST NOT use C++ exceptions (`try`, `catch`, `throw`). The `-fno-exceptions` flag is strictly enforced.
- **RTTI:** YOU MUST NOT use RTTI (`dynamic_cast`, `typeid`).
- **Documentation Rules:** Every generated comment MUST explain the *WHY* (the intent behind the code), not just the *WHAT* (what the code does).
