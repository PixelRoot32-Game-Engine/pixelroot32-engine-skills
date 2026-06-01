---
name: pixelroot32-docs
description: >
  Standard documentation guidelines for PixelRoot32 Game Engine using Doxygen style.
  Trigger: When the user asks to document code, add comments, improve documentation, or write Doxygen comments.
license: MIT
metadata:
  author: gentleman-programming
  version: "1.0"
---

## When to Use

- When writing new classes or methods in the engine.
- When asked to improve or update existing documentation.
- When fixing missing parameter or return documentation.

## Critical Patterns

- **Doxygen Format**: Use `/** ... */` block comments for documentation.
- **Classes/Structs**: Always include `@class` (or `@struct`) and `@brief` for the class description.
- **Methods**: Always include `@brief`, and explicitly document every parameter with `@param` and the return value with `@return` if the return type is not `void`.
- **Language**: Use clear, concise English for the documentation, even if the user prompts in Spanish, unless explicitly requested otherwise.
- **Location**: Place documentation in the header files (`.h`), not in the implementation files (`.cpp`), to serve as the single source of truth.

## Code Examples

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

## Commands

```bash
# To run doxygen generation (if configured)
doxygen Doxyfile
```

## Resources

- **Documentation**: See existing header files like `include/graphics/DrawSurface.h` for reference.

## Subsystem-Specific Documentation

For documenting subsystem APIs, refer to the specialized skills which include API details and composition patterns:

| Subsystem | Skill |
|-----------|-------|
| Camera | `pixelroot32-camera2d` |
| Audio | `pixelroot32-audio` |
| Rendering | `pixelroot32-sprite-renderer` |
| Physics | `pixelroot32-physics` |
| UI | `pixelroot32-ui-system` |
| Scenes | `pixelroot32-scene-manager` |
| Input | `pixelroot32-touch-input` |
| Particles | `pixelroot32-particles` |
| Entities | `pixelroot32-entity-actor` |
