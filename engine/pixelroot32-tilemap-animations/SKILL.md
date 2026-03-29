---
name: pixelroot32-tilemap-animations
description: Implement tile animations with O(1) frame resolution and zero dynamic allocations for ESP32
license: MIT
compatibility: opencode>=0.1.0
metadata:
  domain: engine
  language: cpp
  platform: esp32
---

## Overview

Generate tile animation code with zero dynamic allocations, O(1) frame lookup, and ESP32-optimized performance.

## Source of Truth

- `docs/TILE_ANIMATION_ARCHITECTURE_DIAGRAM.md` - Animation system
- `docs/ARCHITECTURE.md` - TileAnimationManager
- `docs/STYLE_GUIDE.md` - Sprite/tilemap guidelines

## Architecture

```
TileAnimation[] (PROGMEM)  →  TileAnimationManager  →  lookupTable[256] (RAM)
     4 bytes × N                  step()                    resolveFrame(tileIndex)
```

## TileAnimation Struct (PROGMEM)

Each animation is 4 bytes:
```cpp
struct TileAnimation {
    uint8_t baseTileIndex;   // First tile in sequence
    uint8_t frameCount;       // Number of frames
    uint8_t frameDuration;   // Ticks per frame
    uint8_t reserved;        // Padding
};
```

## Memory Budget

| Tileset Size | RAM Usage | % ESP32 DRAM |
|--------------|-----------|--------------|
| 64 tiles | 73 bytes | 0.02% |
| 128 tiles | 137 bytes | 0.04% |
| 256 tiles | 265 bytes | 0.08% |

**Recommendation**: Start with 64-128 tiles, increase only if needed.

## Animation Definition (Scene Header)

```cpp
// In scene header (PROGMEM)
PIXELROOT32_SCENE_FLASH_ATTR const pixelroot32::graphics::TileAnimation animations[] = {
    { 2, 4, 8, 0 },  // Water: tiles 2-5, 4 frames, 8 ticks/frame
    { 6, 2, 6, 0 },  // Lava: tiles 6-7, 2 frames, 6 ticks/frame
};
```

## Initialization

```cpp
// Global manager instance
pixelroot32::graphics::TileAnimationManager animManager(animations, 2, 64);

// Link to tilemap
pixelroot32::graphics::TileMap2bpp backgroundLayer = {
    // ... other fields ...
    nullptr,              // runtimeMask
    nullptr,              // paletteIndices
    &animManager          // animManager - enables animations
};
```

## Game Loop Integration

```cpp
void MyScene::update(unsigned long dt) {
    // Advance animations once per frame
    animManager.step();
    
    Scene::update(dt);
}

void MyScene::draw(pixelroot32::graphics::Renderer& r) {
    // Renderer automatically calls resolveFrame() for each visible tile
    r.drawTileMap(backgroundLayer, 0, 0);
    Scene::draw(r);
}
```

## Animation Speed Control

```cpp
void MyScene::update(unsigned long dt) {
    // Half speed: advance every 2 frames
    if (frameCount % 2 == 0) {
        animManager.step();
    }
    
    // Pause when game is paused
    if (!isPaused) {
        animManager.step();
    }
    
    // Speed up: call step() multiple times
    animManager.step();  // Normal
    animManager.step();   // Double speed
}
```

## Multiple Animation Managers

```cpp
pixelroot32::graphics::TileAnimationManager bgAnim(bgAnims, 2, 64);
pixelroot32::graphics::TileAnimationManager fgAnim(fgAnims, 4, 64);

backgroundLayer.animManager = &bgAnim;
foregroundLayer.animManager = &fgAnim;

void MyScene::update(unsigned long dt) {
    bgAnim.step();
    fgAnim.step();
}
```

## TileMap Rendering

```cpp
// Draw tilemap with animated tiles
renderer.drawTileMap(backgroundLayer, originX, originY);

// For scrolling games, combine with Camera2D
camera.setPosition(x, y);
renderer.setDisplayOffset(-camera.getX(), -camera.getY());
```

## Constraints

- Frames must be contiguous in tileset (tiles 2,3,4,5)
- All instances of a tile share same animation frame
- Global timing - all animations advance together
- Resolution O(1) via lookup table
- Zero dynamic allocations - all data in fixed arrays or PROGMEM
- Use `IRAM_ATTR` on hot path functions
