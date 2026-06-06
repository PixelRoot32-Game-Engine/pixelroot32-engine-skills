---
name: pixelroot32-sprite-renderer
description: Graphics rendering system with 3 sprite formats (1bpp, 2bpp, 4bpp), MultiSprite layering, 3 tilemap types, Dirty Regions, static cache, palette management (single/dual/multi-slot), step-based animation, and text/font rendering. Use when implementing rendering logic, sprites, tilemaps, or visual effects.
license: MIT
compatibility: opencode>=0.1.0
metadata:
  domain: engine
  subsystem: graphics
  module: renderer
  platform: cross-platform
---

## Overview

PixelRoot32's Renderer provides a unified drawing API for shapes, sprites, tilemaps, and text. It supports 3 sprite formats (1bpp monochrome, 2bpp 4-color, 4bpp 16-color), multi-layer sprites, three tilemap types (1bpp/2bpp/4bpp), dirty region optimization (8×8 cell grid), and a palette system with single, dual, and multi-slot modes. The rendering pipeline uses a logical framebuffer (8bpp) with hardware-accelerated draw surfaces.

## Key APIs

### Renderer (Main API)

**Header**: `include/graphics/Renderer.h`
**Namespace**: `pixelroot32::graphics`

```cpp
Renderer renderer(DisplayConfig{});
renderer.init();

// Frame lifecycle
renderer.beginFrame();
// ... draw calls ...
renderer.endFrame();

// Display configuration
renderer.setDisplaySize(240, 240);       // Logical resolution
renderer.setDisplayOffset(camX, camY);   // Camera scroll
renderer.setOffsetBypass(true);          // HUD elements (ignore camera)
renderer.setContrast(200);               // Brightness 0-255
```

### Sprite Formats

**Header**: `include/graphics/Renderer.h`

```cpp
// 1bpp Monochrome (16-bit row packing, bit 0 = leftmost pixel)
struct Sprite mySprite = { data, 8, 8 };

// 2bpp 4-color (per-pixel palette index)
Sprite2bpp sprite2 = { data, palette, 16, 16, 4 };

// 4bpp 16-color
Sprite4bpp sprite4 = { data, palette, 16, 16, 16 };

// Multi-layer (1bpp layers composed)
MultiSprite multi = { 16, 16, layers, 3 };  // 3 layers
```

### Drawing

```cpp
// 1bpp sprites
renderer.drawSprite(mySprite, 100, 100, Color::White, false);
renderer.drawSprite(mySprite, 100, 100, 2.0f, 2.0f, Color::White);  // Scaled

// 2bpp/4bpp sprites (with palette slot)
renderer.drawSprite(sprite2, 100, 100, 0, false);  // paletteSlot=0
renderer.drawSprite(sprite4, 100, 100, 1);          // paletteSlot=1

// Multi-layer sprites
renderer.drawMultiSprite(multi, 100, 100);
renderer.drawMultiSprite(multi, 100, 100, 1.5f, 1.5f);  // Scaled

// Shapes
renderer.drawPixel(10, 10, Color::Red);
renderer.drawLine(0, 0, 100, 100, Color::White);
renderer.drawRectangle(10, 10, 50, 50, Color::Cyan);
renderer.drawFilledRectangle(10, 10, 50, 50, Color::Blue);
renderer.drawFilledRectangleW(10, 10, 50, 50, 0x001F);  // RGB565
renderer.drawCircle(50, 50, 20, Color::Yellow);
renderer.drawFilledCircle(50, 50, 20, Color::Green);

// Bitmaps
renderer.drawBitmap(0, 0, 240, 240, bitmap, Color::White);
```

### Tilemaps

```cpp
// 1bpp, 2bpp, 4bpp tilemaps
TileMap map = { indices, 32, 32, tiles, 8, 8, 256 };
TileMap2bpp map2bpp;
TileMap4bpp map4bpp;

// Drawing
renderer.drawTileMap(map, originX, originY, Color::White, LayerType::Dynamic);
renderer.drawTileMap(map2bpp, originX, originY, LayerType::Static);
renderer.drawTileMap(map4bpp, originX, originY, LayerType::Dynamic);

// Runtime tile activation mask
map.initRuntimeMask();
map.setTileActive(5, 3, false);  // Hide tile at (5,3)
bool active = map.isTileActive(5, 3);
map.cleanupRuntimeMask();         // Manual cleanup
```

### Tile Attributes (metadata)

```cpp
// Query metadata from exported tilemaps
const char* value = get_tile_attribute(layerAttributes, numLayers, 0, 10, 5, "solid");
if (value && strcmp_P(value, "true") == 0) { /* solid */ }

// Check for attributes
bool has = tile_has_attributes(layerAttributes, numLayers, 0, 10, 5);

// Get entry (efficient for multiple queries)
const TileAttributeEntry* entry = get_tile_entry(layerAttributes, numLayers, 0, 10, 5);
```

### Text Rendering

```cpp
// Default font
renderer.drawText("Score: 100", 10, 10, Color::White, 2);

// Custom font
renderer.drawText("Hello", 10, 10, Color::Cyan, 2, &myFont);

// Centered
renderer.drawTextCentered("GAME OVER", 120, Color::Red, 3);
```

### Dirty Regions

**Feature gate**: `PIXELROOT32_ENABLE_DIRTY_REGIONS`

```cpp
// Automatic: beginFrame uses DirtyGrid to clear only dirty 8x8 cells
// Manual overrides
renderer.forceFullRedraw();
renderer.setDebugDirtyCellOverlay(true);  // Visualize dirty cells (debug)
```

### Static Tilemap Cache

**Feature gate**: `PIXELROOT32_ENABLE_STATIC_TILEMAP_FB_CACHE`

```cpp
// For 4bpp tilemap static layers - avoids redrawing every frame
StaticTilemapLayerCache cache;
cache.allocateForLogicalSize(240, 240);

// In scene update
cache.draw(renderer, -camX, -camY, staticLayers, numStatic, dynamicLayers, numDynamic);

// Force rebuild
cache.invalidate();

// Must advise before beginFrame for dirty region alignment
// Call from Scene::adviseFramebufferBeforeBeginFrame()
cache.adviseFramebufferBeforeBeginFrame(renderer, -camX, -camY, staticLayers, numStatic, dynamicLayers, numDynamic);
```

## Sprite Animation

**Header**: `include/graphics/Renderer.h`

```cpp
SpriteAnimation anim = { frames, 4, 0 };  // 4 frames, start at 0

// In update (step-based)
if (moveTrigger) anim.step();

// In draw
const Sprite* s = anim.getCurrentSprite();
if (s) renderer.drawSprite(*s, x, y, Color::White);

const MultiSprite* ms = anim.getCurrentMultiSprite();
if (ms) renderer.drawMultiSprite(*ms, x, y);
```

## Palette System

**Header**: `include/graphics/Color.h`

```cpp
// Single palette mode (default)
setPalette(PaletteType::NES);
setCustomPalette(myRGB565Palette);  // 16 uint16_t values

// Dual palette mode (backgrounds vs sprites)
enableDualPaletteMode(true);
setBackgroundPalette(PaletteType::GB);
setSpritePalette(PaletteType::PICO8);
// Or helper:
setDualPalette(PaletteType::NES, PaletteType::GB);

// Multi-palette tilemaps (8 slots, 0-7)
initBackgroundPaletteSlots();
setBackgroundPaletteSlot(1, PaletteType::GBC);
setBackgroundCustomPaletteSlot(2, myCustomPal);

// Color resolution
uint16_t rgb = resolveColor(Color::Red);                                  // Single mode
uint16_t rgb2 = resolveColor(Color::Red, PaletteContext::Sprite);         // Context-aware
uint16_t rgb3 = resolveColorWithPalette(Color::Red, myPalette);           // Explicit
```

## Composition Patterns

### Scene draw pipeline
```
Scene::draw(renderer):
  ├── renderer.setDisplayOffset(-camX, -camY)
  ├── renderer.drawTileMap(bgMap, 0, 0, LayerType::Static)
  ├── renderer.drawSprite(playerSprite, px, py, Color::White)
  ├── renderer.setOffsetBypass(true)           // HUD
  ├── renderer.drawText("HP", 5, 5, Color::Red, 1)
  └── renderer.setOffsetBypass(false)
```

### Palette-context sprite batch
```
renderer.setSpritePaletteSlotContext(2);
renderer.drawSprite(enemy1, x, y);   // Uses slot 2
renderer.drawSprite(enemy2, x, y);   // Uses slot 2
renderer.setSpritePaletteSlotContext(0xFF);  // Disable context
```

## ESP32 Constraints

- **Logical framebuffer**: 8bpp index buffer (not RGB565). Palette resolution done at flush time.
- **PROGMEM strings**: All tile attribute keys/values stored in flash via `PIXELROOT32_MEMCPY_P`/`PIXELROOT32_STRCMP_P`. Never copy to RAM.
- **Runtime mask**: Tile activation uses bit-packed mask: `(width * height + 7) / 8` bytes. Allocate with `initRuntimeMask()`.
- **Dirty regions**: 8×8 cell grid reduces per-frame clear cost. Static layers (`LayerType::Static`) suppress per-cell dirty marking.
- **StaticTilemapLayerCache**: Pre-allocate during `Scene::init()` — avoid heap allocation in game loop.
- **No `std::function`**: Callbacks use raw function pointers (`UIElementVoidCallback`).

## Gotchas

1. **Palette pointers must be static**: `setCustomPalette()`, `setBackgroundCustomPaletteSlot()` do not copy the palette array — the pointer must remain valid.
2. **`Color::Transparent` is not a real color**: Resolving it or using it in draw primitives is a no-op. It's a sentinel value (255).
3. **TileAttribute data is PROGMEM**: Always use `PIXELROOT32_STRCMP_P()` for key comparison and `PIXELROOT32_MEMCPY_P()` to read values.
4. **Sprite format**: `Sprite` data is packed 16-bit rows. Bit 0 is the leftmost pixel. Only the lowest `width` bits are used.
5. **LayerType::Static**: Marking a tilemap as static tells the dirty system not to mark individual cells — the layer is assumed to cover the full viewport.
6. **Dirty regions and stacked scenes**: When stacking scenes, at least one scene must NOT advise framebuffer clear suppression, or all stacked scenes must collectively cover the framebuffer.
7. **Dual palette**: Must call `enableDualPaletteMode(true)` before setting separate background/sprite palettes — otherwise both get the same palette.

## Common Patterns

### Animated character with multi-sprite
```cpp
// Setup: 3 layers: body, outline, highlight
const SpriteLayer layers[] = {
    { bodyData, Color::Green },
    { outlineData, Color::Black },
    { highlightData, Color::LightGreen },
};
MultiSprite hero = { 16, 16, layers, 3 };

// Draw at world position (offset by camera)
renderer.drawMultiSprite(hero, px - camX, py - camY);
```

### Tilemap with per-cell palette
```cpp
TileMap4bpp map;
map.paletteIndices = perCellPalette;  // width*height array of slot indices
// Each cell uses a different palette slot (0-7)
renderer.drawTileMap(map, 0, 0, LayerType::Static);
```

## Agent Constraints
- **Pointers:** Custom palettes passed to the renderer MUST be declared as `static` or `const` globally. NEVER pass a pointer to a stack-allocated array that will go out of scope.
