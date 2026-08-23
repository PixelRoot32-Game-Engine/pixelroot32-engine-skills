---
name: pixelroot32-projection
description: Isometric, oblique and mirrored layouts through one integer 2x2 basis (ProjectionSpec), plus projected tilemap drawing, depth sorting and projected camera bounds. Use when a PixelRoot32 game is isometric or non-axis-aligned, or when screen position stops equaling world position.
license: MIT
compatibility: opencode>=0.1.0
metadata:
  domain: engine
  subsystem: math
  module: projection
  feature_gate: PIXELROOT32_ENABLE_PROJECTION
  platform: cross-platform
  engine_version: "1.9.0+unreleased"
---

## Overview

PixelRoot32 has **no isometric mode**. A projection is six integers in a `ProjectionSpec`: the screen anchor of cell `(0,0)` plus the two screen-space axis vectors of the cell grid. Orthogonal, isometric 2:1, isometric 1:1, oblique and mirrored layouts are all *values* of that one type. There is no isometric function, enum or template parameter anywhere in the engine — do not write one, and never hand-roll diamond math.

Two catalog-wide assumptions are **false** here, and they are why this skill exists:

- "Screen position equals world position." A `+1` step along one cell axis moves **both** screen axes; two cells at different world Ys can share a screen row.
- "Draw order is `renderLayer`." Under a projection, depth is per **cell**, not per layer, and a flat integer cannot express it.

Everything is opt-in; a `constexpr` spec costs zero SRAM.

## Key APIs

### ProjectionSpec and conversion math

**Header**: `include/math/Projection.h`
**Namespace**: `pixelroot32::math`
**Feature gate**: `PIXELROOT32_ENABLE_PROJECTION` (default `0`, wraps the whole file; config mirror `pixelroot32::platforms::config::EnableProjection`, `EngineConfig.h:582`)

```cpp
struct ProjectionSpec {
    int originX = 0;  // Screen X of cell (0,0)'s anchor
    int originY = 0;  // Screen Y of cell (0,0)'s anchor
    int axisXx  = 1;  // Screen X delta per +1 cellX
    int axisXy  = 0;  // Screen Y delta per +1 cellX
    int axisYx  = 0;  // Screen X delta per +1 cellY
    int axisYy  = 1;  // Screen Y delta per +1 cellY
};

constexpr int projectionDet(const ProjectionSpec&);              // axisXx*axisYy - axisXy*axisYx
constexpr int cellToScreenX(int cellX, int cellY, const ProjectionSpec&);   // multiply/add only
constexpr int cellToScreenY(int cellX, int cellY, const ProjectionSpec&);
constexpr int screenToCellX(int screenX, int screenY, const ProjectionSpec&); // one floor div
constexpr int screenToCellY(int screenX, int screenY, const ProjectionSpec&);

// Deleted: a double argument is a hard compile error, never a silent narrowing.
int cellToScreenX(double, double, const ProjectionSpec&) = delete;   // same for the other three

constexpr bool projectionSpecIsValid(const ProjectionSpec&, int cols, int rows);
constexpr bool rowMajorIsPainterOrder(const ProjectionSpec&);     // axisXy > 0 && axisYy > 0

struct CellRange { int startCol = 0; int endCol = 0; int startRow = 0; int endRow = 0; };

[[nodiscard]] constexpr CellRange cellRangeForScreenRect(
    const ProjectionSpec&, int screenX, int screenY, int screenW, int screenH, int cols, int rows);

namespace detail { constexpr int projectionFloorDiv(int value, int divisor); }  // pre: divisor >= 1
```

| Layout | Cell stride | Value | det |
|---|---|---|---|
| Orthogonal | 16x16 | `{0, 0, 16, 0, 0, 16}` | 256 |
| Isometric 2:1 | 32x16 | `{0, 0, 16, 8, -16, 8}` | 256 |
| Isometric 1:1 | 32x32 | `{0, 0, 16, 16, -16, 16}` | 512 |
| Oblique | 16x16 | `{0, 0, 16, 0, 8, 16}` | 256 |

The forward mapping never divides. The inverse uses Cramer's rule plus **exactly one** floor division per axis, flooring toward negative infinity — so a touch one pixel outside the map lands in the cell *outside* it instead of being folded into cell `(0,0)`. With a `constexpr` spec the determinant folds to a constant and GCC strength-reduces the division to a shift.

`projectionSpecIsValid` requires `projectionDet >= 1` (zero is degenerate; negative violates `projectionFloorDiv`'s precondition — express a left-handed layout by swapping the two axis columns) and checks all **four** projected corners against Scalar's +/-32767 range. `cols`/`rows` are parameters, not fields: one spec serves several maps.

`cellRangeForScreenRect` inverts the rect's four corners using its **inclusive last pixel** and returns a half-open `[startCol, endCol) x [startRow, endRow)`, clamped asymmetrically (start low, end high). It is exact for cell *parallelograms*, not for tile *sprites* — padding for art that overhangs its cell is the caller's job.

### The gameplay:: forwarding header

**Header**: `include/gameplay/Projection.h`
**Namespace**: `pixelroot32::gameplay`
**Feature gate**: `PIXELROOT32_ENABLE_PROJECTION` (same guard)

```cpp
using ProjectionSpec = pixelroot32::math::ProjectionSpec;   // an ALIAS, never a copy
using pixelroot32::math::projectionDet;
using pixelroot32::math::cellToScreenX;   // each carries its deleted double overload
using pixelroot32::math::cellToScreenY;
using pixelroot32::math::screenToCellX;
using pixelroot32::math::screenToCellY;
using pixelroot32::math::projectionSpecIsValid;
using pixelroot32::math::rowMajorIsPainterOrder;
namespace detail { using pixelroot32::math::detail::projectionFloorDiv; }
```

**Asymmetry, and it is a real trap**: `CellRange` and `cellRangeForScreenRect` are **NOT** forwarded — they exist only in `math::`. `gameplay::cellRangeForScreenRect(...)` does not compile.

## Depth Sorting

**Header**: `include/gameplay/DepthCompare.h`, `include/core/Entity.h`, `include/core/Scene.h`
**Namespace**: `pixelroot32::gameplay`, `pixelroot32::core`
**Feature gate**: `PIXELROOT32_ENABLE_DEPTH_SORT` for `depthKey` and `compareByDepthKey` (config mirror `config::EnableDepthSort`, `EngineConfig.h:522`); `compareByBottomY` is **unguarded**.

```cpp
int16_t depthKey = 0;                                             // Entity.h:107, inside the guard

inline bool compareByBottomY(core::Entity*, core::Entity*);       // DepthCompare.h:29, UNGUARDED
                                                                  //   position.y + toScalar(height)
inline bool compareByDepthKey(core::Entity*, core::Entity*);      // DepthCompare.h:61, guarded
                                                                  //   a->depthKey < b->depthKey

// Install the comparator from inside a Scene subclass:
depthComparator  = &gameplay::compareByDepthKey;
depthSortEnabled = true;
```

The `Scene`-side plumbing — `DepthComparator`, `depthComparator`, `depthSortEnabled`, `sortEntities()` and the `shouldPrecede` predicate, all `protected` and none behind the depth-sort flag — is owned by **`pixelroot32-scene-manager`**. Read it there; this skill covers only which comparator is correct under a projection.

**The rule to learn.** `compareByBottomY` orders by world Y, which is the correct paint order only while screen depth is a monotone function of world Y — true for axis-aligned top-down, **false under any non-identity projection**. Under a projection use `compareByDepthKey` and have the game write `depthKey` from its projected anchor; the engine never assigns or derives it (`core` must not know about a projection).

Render layer stays the primary sort key — the comparator only breaks ties **within** a layer. Equal keys compare `false`, preserving insertion-relative order. Cost: `Entity` 28 -> 32 bytes on 32-bit, 256 B at `MAX_ENTITIES = 64`, paid only by opted-in builds.

## Projected Tilemaps

**Header**: `include/graphics/Renderer.h` (guard opens at `:1263`)
**Namespace**: `pixelroot32::graphics`
**Feature gate**: `PIXELROOT32_ENABLE_TILEMAP_PROJECTION` (default `0`) — **requires** `PIXELROOT32_ENABLE_PROJECTION=1`; violating that is a hard `#error` at `PlatformDefaults.h:178-179`. There is **no** `config::EnableTilemapProjection`; this flag is preprocessor-only.

```cpp
void drawTileMap(const TileMap4bpp& map, int originX, int originY,
                 LayerType layerType, const pixelroot32::math::ProjectionSpec& projection);
void drawTileMap(const TileMap2bpp& map, int originX, int originY,
                 LayerType layerType, const pixelroot32::math::ProjectionSpec& projection);
void drawTileMap(const TileMap& map, int originX, int originY, Color color,
                 LayerType layerType, const pixelroot32::math::ProjectionSpec& projection);

const uint8_t* tileFootY = nullptr;             // TileMapGeneric<T>, Renderer.h:158
inline uint8_t footYFor(uint16_t index) const;  // :176 — tileFootY[index], else 0
```

All three take a **reference** (no null-map form) and give `layerType` **no default**, unlike the orthogonal 4bpp overload. Per-cell anchor used by the implementation:

```
drawX = cellToScreenX(tx, ty, drawSpec) - tile.width / 2
drawY = cellToScreenY(tx, ty, drawSpec) - map.footYFor(index)
```

The path iterates row-major and **never sorts**. Row-major is a valid back-to-front order only when `rowMajorIsPainterOrder(projection)` holds — a predicate that is *sufficient, not necessary*: it is `false` for the Orthogonal and Oblique rows above (`axisXy == 0`) and both still paint correctly because their art fills its cell. `false` means "unproven", not "invalid".

1bpp limits: `Sprite::width` caps at 16 px so a 2:1 diamond caps at 16x8, and `color` is one value for the whole map — shaded 1bpp diamonds cannot exist.

## Producer Obligations

From `docs/architecture/projected-tilemap-producer-obligations.md`. None is guessable from the API surface, and every one fails at **runtime**, not compile time.

1. **Tile ids are 1-based.** `index == 0` is skipped in every overload; a 0-based export silently loses every cell of its first tile type.
2. **Slot 0 must exist and be readable though it is never drawn.** The cull-padding scan reads `tiles[0 .. tileCount)`. A zero-extent sentinel (`{nullptr, nullptr, 0, 0, 0}` for `Sprite4bpp`) is correct; a null slot 0 is not. Never duplicate a real tile into slot 0 — that hides an index-0-skip regression.
3. **`tileFootY` is per-tile, not per-map**: parallel to `tiles[]`, `tileCount` entries; `nullptr` means top-left anchoring. One tileset legitimately mixes heights (`iso_dungeon` uses `8` for floors, `32` for walls and doors in one 7-slot tileset).
4. **The projection is a draw parameter, never map data.** A per-layer copy of the basis is a per-layer chance for two layers to disagree.
5. **`tileWidth`/`tileHeight` keep their orthogonal meaning.** They feed `computeTilemapDirtyTracking`. They are the cell **stride**, not the diamond's on-screen size.
6. **Row-major must be a valid painter's order.** Check `rowMajorIsPainterOrder(spec)`; enforce caller-side with a `static_assert` at the spec's declaration site (an aggregate cannot assert its own initializers under `-fno-exceptions`).

**Palette-bank obligation.** `drawSprite` resolves colour through `getSpritePaletteSlot`, `drawTileMap` through `getBackgroundPaletteSlot` — two separate static arrays in `src/graphics/Color.cpp`. Converting a sprite-per-cell layer to `drawTileMap` therefore changes which bank it reads, silently, with no art change. Confirm both banks agree first; `iso_dungeon` gets away with it only via `setDualCustomPalette(PAL, PAL)`.

**Props are not tiles.** Every tile layer draws before every entity and a `drawTileMap` call is atomic, so no layer arrangement expresses per-cell depth. Art a mover can pass *behind* must be a depth-sorted entity. The test is reachability, not size: `iso_dungeon`'s 40 px wall is a tile (nothing reachable sits behind it); its 30 px altar is an entity (the hero walks past both sides).

## Camera Bounds

**Header**: `include/graphics/ProjectedMapBounds.h`
**Namespace**: `pixelroot32::graphics`
**Feature gate**: `PIXELROOT32_ENABLE_TILEMAP_PROJECTION` (whole file)

```cpp
struct ScreenBounds { int left=0; int top=0; int right=0; int bottom=0; bool valid=false; };  // HALF-OPEN
struct CameraBounds { int minX=0; int maxX=0; int minY=0; int maxY=0; bool valid=false; };    // CLOSED

void expandProjectedMapBounds(ScreenBounds&, const TileMap&,     const math::ProjectionSpec&);
void expandProjectedMapBounds(ScreenBounds&, const TileMap2bpp&, const math::ProjectionSpec&);
void expandProjectedMapBounds(ScreenBounds&, const TileMap4bpp&, const math::ProjectionSpec&);

[[nodiscard]] CameraBounds cameraRangeFor(const ScreenBounds& world, int viewWidth, int viewHeight);
```

Flow: seed one `ScreenBounds{}` (its default `valid = false` is deliberate), call `expandProjectedMapBounds` once per layer at **scene init**, convert once with `cameraRangeFor`, feed `Camera2D::setBounds(minX, maxX)` and `setVerticalBounds(minY, maxY)`, then `apply(renderer)`. `apply()` pushes the negated camera position into `Renderer::setDisplayOffset`, so the following `drawTileMap` must pass a **zero** origin or the camera is double-counted.

`ScreenBounds` is half-open (`right`/`bottom` are one past the last covered pixel, matching a blit's `[drawX, drawX + width)`); `CameraBounds` is **closed** — every field is a position the camera may occupy. They share a field count and nothing else; conflating them is an off-by-one that only surfaces at a map edge.

Tile index 0 is **excluded** from this overhang scan (counting the never-blitted sentinel would let the camera scroll into empty space), whereas the renderer's cull-padding scan *includes* it because over-padding a cull rect is harmless. Do not harmonize the two loops.

Centre-collapse is required, not polish: when an axis's world is smaller than the viewport, `cameraRangeFor` collapses it to `(lo + hi) / 2` — plain truncation toward zero, deliberately **not** `projectionFloorDiv` — because `Camera2D::setPosition` applies min then max unconditionally and an inverted range would jam the camera against `maxX`. Axes collapse independently. Negative `left`/`top` are normal (`axisYx` is negative under an isometric basis), not an edge case.

## Grid Motion Under a Projection

**Header**: `include/gameplay/GridMotion.h:208-209`
**Namespace**: `pixelroot32::gameplay`
**Feature gate**: outer `PIXELROOT32_ENABLE_GAMEPLAY_GRID_SPACE`, nested `#if PIXELROOT32_ENABLE_PROJECTION`

```cpp
inline math::Vector2 interpolatedWorld(const GridMotion& motion, int stepsPerCell,
                                       const ProjectionSpec& spec);
// pre: stepsPerCell >= 1 and projectionSpecIsValid(spec, ...)
```

Projects both endpoints and lerps. `GridMotion` state (`placeAt`, `beginStep`, `tickStep`, `isMoving`) is projection-blind and needs no variant — this overload is why an isometric game does not reimplement cell-to-cell navigation. One cell-axis step moves **both** screen axes, which the `GridSpec` overload cannot express. The lerp truncates toward zero on purpose; do not "fix" it to the flooring division the inverse mapping uses. See `pixelroot32-gameplay-framework` for `GridMotion` and room graphs.

## Composition Patterns

- **Declare the spec once, beside the map, with its invariants**, and emit it *outside* any sprite-format guard so a 4bpp-off build still has geometry (`examples/iso_dungeon/src/assets/IsoDungeonRoomTileMap.h:112`).
- **Projection + depth sort**: tiles paint row-major unsorted; movers and reachable props are entities on one shared `renderLayer`, each writing `depthKey`, with `depthComparator` and `depthSortEnabled` set in the scene's `init()`. See `pixelroot32-scene-manager`.
- **Projection + camera**: `expandProjectedMapBounds` -> `cameraRangeFor` -> `Camera2D::setBounds`/`setVerticalBounds`, then draw at origin `(0,0)`. See `pixelroot32-camera2d`.
- **Projection + input**: `screenToCellX/Y` turns a touch into a cell. See `pixelroot32-touch-input`.
- **Projection + orthogonal rendering**: span tables, dirty regions and static snapshots apply unchanged. See `pixelroot32-sprite-renderer`.

## ESP32 Constraints

- A `constexpr ProjectionSpec` costs **zero SRAM**; the determinant is computed, never stored.
- Forward: 4 multiplies + 2 adds, no division. Inverse: one `div` per axis, strength-reduced to a shift for any `constexpr` spec with a power-of-two determinant (256/512 for the documented layouts) — that is why `iso_dungeon` `static_assert`s `projectionDet == 256`.
- Never reach for `Fixed16::operator/` here: a 64-bit shift plus a `__divdi3` libgcc call on a 32-bit core. Nothing in this subsystem does it.
- `depthKey` costs +4 bytes per `Entity` on 32-bit, 256 B at 64 entities.
- `cellRangeForScreenRect` over-approximates under a non-orthogonal basis (~2.25x for 2:1 iso on 240x240). Each rejected cell is one bounds compare — far cheaper than a sprite decode. Intended overdraw.
- `expandProjectedMapBounds` is `O(tileCount + 4)` and heap-free — scene init only, never per frame.
- Flash-resident `Sprite4bpp` descriptors are read-only on ESP32: `iso_dungeon` gates its `computeSpanTable` + `const_cast` span wiring behind `#if !defined(ESP32)` (`IsoDungeonScene.cpp:45`), because writing `rowMinX`/`rowMaxX` into flash panics with `LoadStoreError` on first draw.

## Gotchas

- `PIXELROOT32_ENABLE_GAMEPLAY_PROJECTION` was **renamed** to `PIXELROOT32_ENABLE_PROJECTION`; the old spelling is a hard `#error` (`PlatformDefaults.h:137-139`).
- `TILEMAP_PROJECTION=1` without `PROJECTION=1` is a hard `#error` (`PlatformDefaults.h:178-179`).
- `CellRange`/`cellRangeForScreenRect` live **only** in `math::`; `gameplay::` does not forward them.
- There is no `config::EnableTilemapProjection`. `config::EnableProjection` (`:582`) and `config::EnableDepthSort` (`:522`) do exist.
- A `double` argument to any of the four conversion functions is a **deleted overload** — a compile error, deliberately, because on the C3 it would otherwise silently resolve to the `int` overload.
- `screenToCellX/Y` **floor**; they do not clamp. Outside the map they return negative or out-of-range indices. Range-check yourself.
- `rowMajorIsPainterOrder == false` does not mean the spec is invalid; a guard that rejects `false` wrongly rejects the Orthogonal and Oblique layouts.
- `tileWidth`/`tileHeight` are the cell stride; a tile bitmap is routinely taller than its cell.
- `ScreenBounds` is half-open, `CameraBounds` is closed. Never cross-assign.
- Projected `drawTileMap` overloads take a **reference** and give `layerType` **no default**.
- Draw at origin `(0,0)` after `Camera2D::apply()`; the display offset is already live in the renderer.
- `depthComparator` and `depthSortEnabled` are `protected` `Scene` members — set them inside a `Scene`-derived `init()`.
- `examples/iso_dungeon` declares **no `GridSpec`** at all. It still needs `PIXELROOT32_ENABLE_GAMEPLAY_GRID_SPACE=1` because `GridMotion` shares that flag. Do not invent a `GridSpec` for an isometric game.
- The `iso_dungeon` README says `gameplay::ProjectionSpec` in prose while the asset header uses `pixelroot32::math::ProjectionSpec`. Same type (a `using` alias); prefer the `math::` spelling in new code.
- **The Tilemap Editor exports isometric scenes.** Per `docs/tools/tilemap-editor/isometric-guide.md`, the C++ export emits `ISO_PROJECTION`, the `TILESET_FOOT_Y` table, a rectangular `TILE_WIDTH`/`TILE_HEIGHT` cell stride and `inline constexpr` dimensions, with `static_assert`s that fire at the consuming project's compile. An isometric scene round-trips — do not hand-write the export. `docs/architecture/projected-tilemap-producer-obligations.md` still calls this an unbuilt "phase-2" generator delta; **that passage is stale**. The obligations themselves remain binding: any `TileMapGeneric` built by hand or by another tool must still satisfy all seven.

## Common Patterns

### Declaring a validated isometric spec

```cpp
#include "math/Projection.h"

namespace iso_dungeon {
inline constexpr uint8_t TILE_WIDTH  = 32;   // CELL STRIDE, not bitmap size
inline constexpr uint8_t TILE_HEIGHT = 16;
inline constexpr uint8_t MAP_WIDTH   = 7;
inline constexpr uint8_t MAP_HEIGHT  = 7;

inline constexpr pixelroot32::math::ProjectionSpec ISO_PROJECTION{
    120, 88,   // cell (0,0)'s diamond CENTRE
     16,  8,   // +1 cellX -> right and down
    -16,  8};  // +1 cellY -> left  and down

static_assert(pixelroot32::math::projectionSpecIsValid(ISO_PROJECTION, MAP_WIDTH, MAP_HEIGHT), "...");
static_assert(pixelroot32::math::projectionDet(ISO_PROJECTION) == 256, "...");
static_assert(pixelroot32::math::rowMajorIsPainterOrder(ISO_PROJECTION), "...");
}  // emitted OUTSIDE any #ifdef PIXELROOT32_ENABLE_4BPP_SPRITES guard
```

### Drawing one projected layer, and wiring projection-safe depth sort

```cpp
renderer.drawTileMap(*map_, 0, 0, gfx::LayerType::Static, ISO_PROJECTION);

void IsoDungeonScene::init() {
    Scene::init();
    gfx::setDualCustomPalette(TILEMAP_PALETTE_DATA, TILEMAP_PALETTE_DATA);
    depthComparator  = &gameplay::compareByDepthKey;
    depthSortEnabled = true;
}

depthKey = static_cast<int16_t>(position.y);   // HeroActor.cpp:185 — projected anchor
depthKey = static_cast<int16_t>(centreY_);     // PropEntity.cpp:36
depthKey = INT16_MIN;                          // always draw first
```

### Scroll bounds from projected layers, and touch picking

```cpp
gfx::ScreenBounds world{};
for (uint8_t i = 0; i < NUM_ROOM_LAYERS; ++i) {
    gfx::expandProjectedMapBounds(world, *ROOM_LAYERS[i], ISO_PROJECTION);
}
const gfx::CameraBounds cam = gfx::cameraRangeFor(world, viewW, viewH);
if (cam.valid) {
    camera.setBounds(math::toScalar(cam.minX), math::toScalar(cam.maxX));
    camera.setVerticalBounds(math::toScalar(cam.minY), math::toScalar(cam.maxY));
}

const int cx = pr32::math::screenToCellX(touchX, touchY, ISO_PROJECTION);
const int cy = pr32::math::screenToCellY(touchX, touchY, ISO_PROJECTION);
if (cx < 0 || cy < 0 || cx >= MAP_WIDTH || cy >= MAP_HEIGHT) return;  // floors, never clamps
```

## Agent Constraints

**DO**

- Represent every non-axis-aligned layout as a `ProjectionSpec` value.
- Emit the `static_assert`s next to the spec: `projectionSpecIsValid`, the determinant, and `rowMajorIsPainterOrder` when tiles can overhang.
- Qualify `CellRange`/`cellRangeForScreenRect` with `pixelroot32::math::`.
- Use `compareByDepthKey` + a game-written `depthKey` for any projected game; reserve `compareByBottomY` for axis-aligned top-down.
- Pass the projection as a draw argument, and pass origin `(0,0)` when a camera is applied.
- Provide a zero-extent, readable slot 0 and keep tile ids 1-based.
- Set `tileWidth`/`tileHeight` to the cell stride.
- Add `PIXELROOT32_ENABLE_PROJECTION=1` (plus `..._TILEMAP_PROJECTION=1`, `..._DEPTH_SORT=1`, `..._GAMEPLAY_GRID_SPACE=1` as needed) to `platformio.ini` build flags.

**DON'T**

- Do not hand-roll diamond math, `(x - y) * halfW` formulas, or an isometric helper class/enum/template. The engine has exactly one model.
- Do not invent APIs: there is no `IsometricSpec`, no `Projection` object, no `setProjection()`, no `config::EnableTilemapProjection`, no `gameplay::cellRangeForScreenRect`.
- Do not use `PIXELROOT32_ENABLE_GAMEPLAY_PROJECTION`, and do not enable `TILEMAP_PROJECTION` without `PROJECTION`.
- Do not pass `double`/`float` coordinates to the conversion functions.
- Do not store the projection inside a `TileMapGeneric`, or add a determinant field to `ProjectionSpec`.
- Do not treat `rowMajorIsPainterOrder == false` as an error.
- Do not sort projected tiles; sort entities instead.
- Do not model reachable props as tiles or as extra tile layers.
- Do not convert a sprite-per-cell layer to `drawTileMap` without checking that the sprite and background palette banks agree.
- Do not swap `ScreenBounds` and `CameraBounds`, and do not change `cameraRangeFor`'s truncation to floor division.
- Do not write to flash-resident sprite descriptors on ESP32.
