---
name: pixelroot32-cpp-code-generation
description: Generate C++17 code following PixelRoot32 engine conventions, naming patterns, and architectural practices
license: MIT
compatibility: opencode>=0.1.0
metadata:
  domain: engine
  language: cpp
  platform: cross-platform
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
  - `pixelroot32::audio` (when `PIXELROOT32_ENABLE_AUDIO=1`)

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
        player = std::make_unique<Player>(100, 100, 16, 16);
        addEntity(player.get());
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

## Memory Constraints

Generate code respecting ESP32 limits:
- Max entities per scene: 32 (configurable via `MAX_ENTITIES`)
- Physics contacts pool: 128 (fixed, no heap allocation)
- Spatial grid cell size: 32px default
- Avoid runtime allocations in `update()` and `draw()`

## Modular Compilation

Wrap subsystem code with conditional compilation:

```cpp
#if PIXELROOT32_ENABLE_AUDIO
class AudioManager {
    pixelroot32::audio::AudioEngine* audio;
};
#endif

#if PIXELROOT32_ENABLE_PHYSICS
class PhysicsObject : public pixelroot32::core::PhysicsActor {
    // Physics-enabled actor
};
#endif
```

## Logging Integration

Use unified logging when `PIXELROOT32_DEBUG_MODE` is defined:

```cpp
#include <core/Log.h>
using namespace pixelroot32::core::logging;

log("Player position: %d, %d", static_cast<int>(x), static_cast<int>(y));
log(LogLevel::Warning, "Low memory: %d bytes", freeRAM);
```

## Constraints

- Do NOT generate code that violates documented APIs
- Do NOT assume un documented behavior
- Do NOT use internal namespaces
- Do NOT create runtime allocations in game loop
- Always use fully-qualified names in headers
