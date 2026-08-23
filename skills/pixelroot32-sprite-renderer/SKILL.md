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
  engine_version: "1.9.0+unreleased"
  # No `feature_gate` key: the renderer subsystem itself is always compiled.
  # The optional sub-features it hosts are gated individually and documented in
  # the body: PIXELROOT32_ENABLE_DIRTY_REGIONS,
  # PIXELROOT32_ENABLE_STATIC_TILEMAP_FB_CACHE,
  # PIXELROOT32_ENABLE_STATIC_LAYER_SNAPSHOT, PIXELROOT32_TFT_12BIT_COLOR.
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

`TileMap` / `TileMap2bpp` / `TileMap4bpp` are aliases of `TileMapGeneric<T>`
(`include/graphics/Renderer.h`). The struct has **11 fields**, and **the first
eight have no default initializer** — `runtimeMask` included. Brace-init must
supply all eight or the members are left indeterminate.

```cpp
template <typename T>
struct TileMapGeneric {
    uint8_t*  indices;      // width * height cells
    uint8_t   width;
    uint8_t   height;
    const T*  tiles;
    uint8_t   tileWidth;
    uint8_t   tileHeight;
    uint16_t  tileCount;
    uint8_t*  runtimeMask;  // 1 bit per tile; nullptr = all tiles active
    // --- only these three default to nullptr ---
    TileAnimationManager* animManager    = nullptr;
    const uint8_t*        paletteIndices = nullptr;
    const uint8_t*        tileFootY      = nullptr;

    inline uint8_t footYFor(uint16_t index) const;
    void initRuntimeMask();
    bool isTileActive(int x, int y);
};
```

```cpp
// 1bpp, 2bpp, 4bpp tilemaps — all eight non-defaulted fields supplied
TileMap map = { indices, 32, 32, tiles, 8, 8, 256, nullptr };
TileMap2bpp map2bpp = { indices2, 32, 32, tiles2, 8, 8, 64, nullptr };
TileMap4bpp map4bpp = { indices4, 32, 32, tiles4, 8, 8, 256, nullptr };

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

**Per-cell background palette (`paletteIndices`)** — only meaningful for 2bpp/4bpp
multi-palette tilemaps. Array size is `width * height`, one byte per cell:
**bits 0-2 are the palette slot (0..7); bits 3-7 are reserved.** Extract with
`kTileCellPaletteMask`, which is a **namespace-scope constant, not a struct
member** (`Renderer.h:87-89`, value **`0x07`**); its sprite-side sibling is
`kSpritePaletteMask = 0x07` (`:92`).

```cpp
uint8_t slot = map4bpp.paletteIndices[y * map4bpp.width + x] & kTileCellPaletteMask;
```

**Per-tile foot row (`tileFootY`)** — array parallel to `tiles[]` with `tileCount`
entries, read through `footYFor(index)`. `nullptr` means plain top-left anchoring.
This field exists for the projected-rendering story (depth sorting against a
tile's ground contact row); the projected `drawTileMap` overloads, the cell↔screen
math, and the producer obligations are **not documented here** — see the
**`pixelroot32-projection`** skill.

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

**Header**: `include/graphics/DirtyGrid.h`

```cpp
bool intersectsPrevDirty(int x, int y, int w, int h) const;   // :84
```

Iterates the cells covered by the **half-open** rect `[x, x+w) × [y, y+h)` and
returns `true` if any of them was dirty in the previous frame. Exact edge cases:

- `fullDirty` is checked **before** the degenerate-rect guard, so on a full-dirty
  grid even `intersectsPrevDirty(0, 0, 0, 0)` returns `true`.
- Otherwise `w <= 0` or `h <= 0` returns `false`, as does a rect entirely outside
  the grid bounds.
- Safe to call on a freshly constructed grid (returns `false`).

### DrawSurface::flushPendingTransfers

**Header**: `include/graphics/DrawSurface.h`

```cpp
virtual void flushPendingTransfers() {}   // :246 — virtual, NOT pure; empty by default
```

Contrast `virtual void present() = 0;` (`:230`), which every driver must implement.
Rationale: some drivers defer the tail of a frame so its bus time overlaps the next
frame, and they assume a `present()` will follow. A scene reporting
`shouldRedrawFramebuffer() == false` breaks that assumption — without this call the
bus stays claimed and every other device on it is locked out. The Engine calls
`flushPendingTransfers()` on exactly the frames it skips presenting.
`TFT_eSPI_Drawer` overrides it with `waitForPendingDMA()`. A driver with nothing
pending correctly inherits the no-op.

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

### StaticLayerSnapshot

**Header**: `include/graphics/StaticLayerSnapshot.h`
**Namespace**: `pixelroot32::graphics`
**Feature gate**: `PIXELROOT32_ENABLE_STATIC_LAYER_SNAPSHOT` (default **0**)

```cpp
StaticLayerSnapshot() = default;                                    // :80
void clear();                                                       // :83
[[nodiscard]] bool allocateForLogicalSize(int width, int height);   // :107
[[nodiscard]] bool allocateForRenderer(const Renderer& renderer);   // :110
void invalidate();                                                  // :113
[[nodiscard]] bool isValid() const;                                 // :116
bool capture(Renderer& renderer);                                   // :130
bool restore(Renderer& renderer);                                   // :142
void adviseFramebufferBeforeBeginFrame(Renderer& renderer) const;   // :154
```

**Choosing between `StaticLayerSnapshot` and `StaticTilemapLayerCache`.** Both own
one heap pixel buffer; they differ on **who draws**:

| | `StaticTilemapLayerCache` | `StaticLayerSnapshot` |
|---|---|---|
| Draws the layers | **Yes** — owns `TileMap4bpp*` specs and issues the draw calls itself | **No** — never draws anything |
| Layer references | Holds `TileMap4bppDrawSpec` arrays (`:71-77`) | Holds none |
| Camera | Tracks a camera sample and re-keys on it | Tracks no camera |
| `adviseFramebufferBeforeBeginFrame` | 7 arguments | **1 argument** |
| Projection | Tilemap-shaped only | "projection-agnostic where the tilemap cache is not" (`:34-38`) |

Use the snapshot when the static layer is drawn **sprite-per-cell by game code** —
an isometric room, for example — so there is no `TileMap4bpp` to hand to the cache.
The contract is a plain two-verb handshake: the game says "the framebuffer now holds
my static layers" (`capture()`), and later "put them back" (`restore()`).
**`restore()` returning `false` is the signal to redraw the layers yourself and
`capture()` again.**

Hard contracts:

- **`capture()` must run before anything dynamic is drawn** (`:121-124`). Capture
  after the player is drawn and the player is baked into the snapshot permanently.
- **A failed re-allocation drops previously captured contents** (`:95-99`) — treat a
  `false` from `allocateFor*` as "snapshot is now empty", not "nothing happened".
- Restore has two speeds (`:47-53`): with `PIXELROOT32_ENABLE_DIRTY_REGIONS` only
  previously-dirtied cells are repainted, *roughly two orders of magnitude cheaper*
  than the full copy. Without dirty regions every restore is a full framebuffer copy.
- **Cost: one full logical framebuffer of heap — 57,600 B at 240×240** (`:42`).
  Budget it during `Scene::init()`, never in the game loop.
- The buffer is `std::unique_ptr<uint8_t, SnapshotBufferDeleter>` over
  `std::malloc` / `std::free` — the style guide forbids `operator new` here.

### Span Tables

**Header**: `include/graphics/SpanTable.h`
**Namespace**: `pixelroot32::graphics`
**Feature gate**: none — always compiled

```cpp
void computeSpanTable(Sprite4bpp& s, uint8_t* outMinX, uint8_t* outMaxX);   // :55
void computeSpanTable(Sprite2bpp& s, uint8_t* outMinX, uint8_t* outMaxX);   // :64
```

Both take the sprite by **non-const reference** and return **`void`**. The results
land in the two caller-owned arrays, which the sprite then points at:

```cpp
const uint8_t* rowMinX = nullptr;   ///< per-row opaque span start; nullptr = full bbox
const uint8_t* rowMaxX = nullptr;   ///< one past last col;         nullptr = full bbox
```
(`Renderer.h:65-68` for `Sprite2bpp`, `:81-84` for `Sprite4bpp` — identical text.)

Contracts:

- **Lifetime** (`:14-20`): the function writes into caller-provided `uint8_t[height]`
  arrays; the sprite holds **non-owning** pointers to them. The buffers must outlive
  every draw call that uses the sprite — never point at a stack array in a scope that
  ends before the frame does.
- **`computeSpanTable` does NOT wire the pointers in** (`:50-53`): "rowMinX/rowMaxX
  are NOT modified; the caller is responsible for assigning them after this call."
  Computing the table and forgetting the two assignments silently changes nothing.
- **Over-approximation** (`:20-25`): only leading and trailing transparent runs are
  skipped. Interior transparent pixels inside `[rowMinX, rowMaxX)` are still iterated.
- A fully transparent row yields `{ minX = 0, maxX = 0 }` (`:27-30`).
- **`flipX` bypasses the span limits entirely** — `src/graphics/Renderer.cpp:740-750`
  (4bpp) and `:630-636` (2bpp): the mirrored layout invalidates the precomputed
  min/max, so a flipped draw walks the full bounding box. Never budget a
  performance win for flipped sprites.

```cpp
static uint8_t minX[16], maxX[16];       // must outlive every draw of `spr`
computeSpanTable(spr, minX, maxX);
spr.rowMinX = minX;                       // <-- caller must do this
spr.rowMaxX = maxX;                       // <-- and this
```

**ESP32**: sprite structs are usually flash-resident and cannot be mutated in place.
`examples/iso_dungeon/src/IsoDungeonScene.cpp:60-78` uses `const_cast` and gates the
wiring for ESP32 (documented at `SpanTable.h:37-43`). Copy the sprite struct into RAM
or follow that gating pattern; do not assign `rowMinX`/`rowMaxX` on a flash object.

### 12-bit colour (Rgb444)

**Header**: `include/graphics/Rgb444.h`
**Namespace**: `pixelroot32::graphics`
**Feature gate**: the header itself has **none** — it is deliberately platform-neutral
so native tests can verify it. The *driver path* is gated by
`PIXELROOT32_TFT_12BIT_COLOR` (default **0**, `PlatformDefaults.h:244-246`, defined
**nested inside** `#if defined(PIXELROOT32_USE_TFT_ESPI_DRIVER)`).

```cpp
inline constexpr uint16_t packRgb565ToRgb444(uint16_t rgb565);           // :37
inline void packRgb444Pair(uint8_t* dst, uint16_t left, uint16_t right); // :61
```

`packRgb444Pair` writes **exactly three bytes**; `dst` must not be null. Bits above
the low 12 of either colour are ignored, so raw palette entries may be passed
unmasked.

**Experimental — not verified on hardware** (`docs/api/config.md:52`). It cuts 25% of
SPI bus time per frame and shrinks each DMA line buffer by 25%, but **the driver
silently keeps RGB565 when the physical width is not a multiple of 4** — enabling the
flag is not proof the path is active. The RGB332→RGB444 mapping is bijective but
**not bit-exact** (`docs/audits/performance-audit-esp32.md:102`): red and green shades
shift slightly, blue is exact.

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

**Related skills** — do not re-derive their material here:
- **`pixelroot32-projection`**: `ProjectionSpec`, cell↔screen math, depth sorting, the
  projected `drawTileMap` overloads and their producer obligations, `ScreenBounds` /
  `CameraBounds` / `cameraRangeFor`.
- **`pixelroot32-gameplay-framework`**: `GridSpec`/`GridMotion`, `RoomGraph`,
  `StateMachine`, `ObjectPool`, `EventBus`, `InteractionTracker` — the grid/room logic
  that decides *what* to draw per cell in a sprite-per-cell static layer.

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
- **StaticLayerSnapshot**: one full logical framebuffer of heap — **57,600 B at 240×240**. Allocate in `Scene::init()`; a failed re-allocation discards the captured contents.
- **Span tables**: the `uint8_t[height]` arrays are caller-owned and must outlive every draw. Flash-resident sprite structs cannot be mutated in place on ESP32 — see the `const_cast` gating in `examples/iso_dungeon`.
- **No `std::function`**: Callbacks use raw function pointers (`UIElementVoidCallback`).

## Gotchas

1. **Palette pointers must be static**: `setCustomPalette()`, `setBackgroundCustomPaletteSlot()` do not copy the palette array — the pointer must remain valid.
2. **`Color::Transparent` is not a real color**: Resolving it or using it in draw primitives is a no-op. It's a sentinel value (255).
3. **TileAttribute data is PROGMEM**: Always use `PIXELROOT32_STRCMP_P()` for key comparison and `PIXELROOT32_MEMCPY_P()` to read values.
4. **Sprite format**: `Sprite` data is packed 16-bit rows. Bit 0 is the leftmost pixel. Only the lowest `width` bits are used.
5. **LayerType::Static**: Marking a tilemap as static tells the dirty system not to mark individual cells — the layer is assumed to cover the full viewport.
6. **Dirty regions and stacked scenes**: When stacking scenes, at least one scene must NOT advise framebuffer clear suppression, or all stacked scenes must collectively cover the framebuffer.
7. **Dual palette**: Must call `enableDualPaletteMode(true)` before setting separate background/sprite palettes — otherwise both get the same palette.
8. **Sprites and tilemaps read different palette banks.** `drawSprite` resolves colour
   through `getSpritePaletteSlot`; `drawTileMap` through `getBackgroundPaletteSlot`,
   and `src/graphics/Color.cpp` keeps two separate static arrays. Converting a
   sprite-per-cell layer into a single `drawTileMap` call therefore **silently changes
   which bank it reads** — the colours shift even though the indices did not. Mirror
   the slot into the other bank before making that switch.
9. **`TileMapGeneric`'s first eight fields have no default initializer**, `runtimeMask`
   included. `TileMap4bpp map;` at block scope leaves them indeterminate — always
   brace-init all eight.
10. **`kTileCellPaletteMask` is namespace-scope, not a struct member** — write
    `kTileCellPaletteMask` (value `0x07`), never `map.kTileCellPaletteMask`.

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
TileMap4bpp map = { indices, 32, 32, tiles, 8, 8, 256, nullptr };
map.paletteIndices = perCellPalette;  // width*height bytes; bits 0-2 = slot, 3-7 reserved
// Each cell selects a palette slot (0-7) via `cell & kTileCellPaletteMask`
renderer.drawTileMap(map, 0, 0, LayerType::Static);
```

## Agent Constraints
- **Pointers:** Custom palettes passed to the renderer MUST be declared as `static` or `const` globally. NEVER pass a pointer to a stack-allocated array that will go out of scope. The same rule applies to span-table arrays assigned to `rowMinX`/`rowMaxX`.
- **Struct init:** NEVER default-construct a `TileMapGeneric` alias — brace-init all eight non-defaulted fields.
- **API surface:** NEVER invent renderer, snapshot, or span-table members. If a symbol is not listed above, verify it in the header before emitting it.
