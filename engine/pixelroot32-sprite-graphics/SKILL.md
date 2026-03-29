---
name: pixelroot32-sprite-graphics
description: Implement sprites, tilemaps, and rendering using 1bpp/2bpp/4bpp formats with multi-palette support
license: MIT
compatibility: opencode>=0.1.0
metadata:
  domain: engine
  language: cpp
  platform: esp32
---

## Overview

Generate graphics code using PixelRoot32's rendering system with sprite formats, tilemaps, multi-palette support, and camera management.

## Source of Truth

- `docs/STYLE_GUIDE.md` - Sprite/Graphics guidelines
- `docs/API_REFERENCE.md` - Renderer, Camera2D, Color
- `docs/ARCHITECTURE.md` - Renderer layer

## Sprite Formats

| Format | Flag | Memory | Use Case |
|--------|------|--------|----------|
| 1bpp | Default | w×h/8 | Core gameplay sprites |
| 2bpp | `PIXELROOT32_ENABLE_2BPP_SPRITES` | w×h/4 | Basic color |
| 4bpp | `PIXELROOT32_ENABLE_4BPP_SPRITES` | w×h/2 | Full color |

## 1bpp Sprite Definition

```cpp
#include <graphics/Sprite.h>

static const uint16_t PLAYER_SPRITE_BITMAP[] = {
    0b01100110,  // Row 0
    0b11111110,  // Row 1
    // ... more rows (one uint16_t per row)
};

static const pixelroot32::graphics::Sprite playerSprite = {
    PLAYER_SPRITE_BITMAP,
    8,   // width
    8,   // height
    pixelroot32::graphics::Color::White,
    pixelroot32::graphics::Color::Transparent
};
```

## Rendering Sprites

```cpp
// Convert Scalar to int at render time
renderer.drawSprite(
    playerSprite,
    static_cast<int>(x),
    static_cast<int>(y),
    pixelroot32::graphics::Color::White
);
```

## Layered Sprites (Multi-Color)

```cpp
static const pixelroot32::graphics::SpriteLayer PLAYER_LAYERS[] = {
    {outlineBitmap, pixelroot32::graphics::Color::Black},
    {bodyBitmap, pixelroot32::graphics::Color::White},
};

static const pixelroot32::graphics::MultiSprite playerMulti = {
    PLAYER_LAYERS,
    2,  // layerCount
    8,  // width
    8   // height
};
```

## Multi-Palette System

Enable dual palette mode:
```cpp
void MyScene::init() override {
    pixelroot32::graphics::Color::enableDualPaletteMode(true);
    
    // Initialize slot banks
    pixelroot32::graphics::initBackgroundPaletteSlots();
    pixelroot32::graphics::initSpritePaletteSlots();
    
    // Assign palettes
    pixelroot32::graphics::setSpritePaletteSlot(0, 
        pixelroot32::graphics::PaletteType::PR32);
    pixelroot32::graphics::setSpritePaletteSlot(1, 
        pixelroot32::graphics::PaletteType::NES);
}
```

Draw with palette slot:
```cpp
renderer.drawSprite(enemySprite, x, y, paletteSlot, flipX);
```

Context for batch rendering:
```cpp
renderer.setSpritePaletteSlotContext(1);  // Set once
for (auto& bullet : bullets) {
    renderer.drawSprite(bulletSprite, bullet.x, bullet.y, 0, false);
}
renderer.setSpritePaletteSlotContext(0xFF);  // Reset
```

## Tilemap Rendering

```cpp
// Tile indices in compact uint8_t array
static const uint8_t TILE_MAP_DATA[] = {
    1, 1, 1, 1, 1,
    1, 0, 0, 0, 1,
    // ...
};

static const pixelroot32::graphics::TileMap2bpp LEVEL = {
    TILE_MAP_DATA,
    5,   // width in tiles
    2,   // height in tiles
    8,   // tile width
    8,   // tile height
    tileBitmaps,   // tileset
    16,            // tileset size
    nullptr,       // runtimeMask
    nullptr,       // paletteIndices
    nullptr        // animManager
};

renderer.drawTileMap(LEVEL, originX, originY);
```

## Camera2D

```cpp
#include <graphics/Camera2D.h>

pixelroot32::graphics::Camera2D camera;

void MyScene::update(unsigned long dt) {
    camera.follow(player.get());
    camera.smoothFollow(0.1f);  // Smooth interpolation
}

void MyScene::draw(pixelroot32::graphics::Renderer& r) {
    r.setDisplayOffset(
        -camera.getX(), 
        -camera.getY()
    );
    // Draw world...
}
```

## Render Layers

| Layer | Use |
|-------|-----|
| 0 | Background (tilemaps, stars) |
| 1 | Gameplay (player, enemies, bullets) |
| 2 | UI (labels, menus, HUD) |

```cpp
playerActor.setRenderLayer(1);  // Gameplay
backgroundEntity.setRenderLayer(0);  // Background
scoreLabel.setRenderLayer(2);  // UI
```

## Resolution Scaling

```cpp
pixelroot32::graphics::DisplayConfig config;
config.logicalWidth = 128;
config.logicalHeight = 128;
config.physicalWidth = 240;
config.physicalHeight = 240;
```

## Constraints

- Use 1bpp for gameplay-critical sprites
- Use 2bpp/4bpp sparingly on ESP32
- Convert Scalar to int only at render time
- Use layered sprites for multi-color instead of higher bpp
- Render layers in ascending order (0, 1, 2)
- Tilemap indices in compact uint8_t array to minimize RAM
