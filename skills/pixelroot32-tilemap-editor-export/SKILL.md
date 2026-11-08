---
name: pixelroot32-tilemap-editor-export
description: Consume the PixelRoot32 Tilemap Editor's C++ export — generated file shape, export options, isometric contract (projection, foot table, span tables, room maps) and the re-export cycle. Use when importing an exported scene or tileset, wiring its init(), drawing room maps, or re-exporting a changed map.
license: MIT
compatibility: opencode>=0.1.0
metadata:
  domain: engine
  subsystem: tooling
  module: core
  platform: cross-platform
  engine_version: "1.15.0"
---

## Overview

The PixelRoot32 Tilemap Editor emits **data, not code you maintain**: a scene
becomes `<namespace>_<scene>.h`, `<namespace>_<scene>.cpp` and
`<namespace>_tilemap_palette.h`, in namespace `<namespace>::<scene>`. Every
export overwrites those three files, so game logic lives in your own files and
the map is changed in the editor.

Three facts decide almost every question about an export:

1. **The export is an interface, not a suggestion.** The header states its own
   invariants with `static_assert`s that fire at *your* compile.
2. **Only what is resolvable is exported.** A producer cannot emit a rule, only
   its result — a checkerboard baked from `(x + y) & 1` at runtime is exported
   as the resolved index array. Layouts you keep for gameplay (walkability,
   spawns) stay your source of truth, and a test is what pins the two together.
3. **Orthogonal is unchanged.** Projection, foot table, span tables, cell
   stride and room maps are emitted **only for isometric scenes**; an orthogonal
   export is byte-for-byte what it always was.

This skill owns the export contract. It does not own projection math
(`pixelroot32-projection`), `RoomGraph` (`pixelroot32-gameplay-framework`) or
sprite bit layouts (`pixelroot32-sprite-renderer`).

## Source of Truth

- `docs/tools/tilemap-editor/overview.md` — what the editor is, formats, export summary
- `docs/tools/tilemap-editor/technical-reference.md` — file shape, export options, isometric sections, data formats
- `docs/tools/tilemap-editor/isometric-guide.md` — isometric workflow, foot table, palette install, 1bpp limit, rooms
- `docs/tools/tilemap-editor/usage-guide.md`, `quick-start.md` — authoring, rooms, export dialog
- `docs/architecture/projected-tilemap-producer-obligations.md` — what a projected tileset must satisfy at runtime
- Reference exports: `graphics/iso_dungeon` and `graphics/iso_tilemap_export` in PixelRoot32-Demo-Projects (each keeps its `.pr32scene.bin` and its export settings in its README)

## Export Options

| Option | Meaning | Notes |
|---|---|---|
| **C++ Namespace** | Namespace of the generated code | Becomes `<namespace>::<scene>` |
| **Color Depth (BPP)** | 1, 2 or 4 bpp | **Read-only**, auto-detected from the palette |
| **Store in Flash (ESP32)** | Puts data in PROGMEM | Recommended for every target |
| **Legacy Format** | Omits the flash attributes | Compatibility only |
| **Install palette in init()** | `None` (default), `Background` or `Dual` | Which palette bank the generated `init()` fills |
| **Custom screen origin** | Isometric only | Default is the map's own inset; tick it for a fixed one-screen room |

**Options are not stored in the project file.** The dialog starts from its
defaults on every export — set them again each time.

`Install palette in init()` adds one call at the top of `init()`:
`setBackgroundCustomPalette(P)` for *Background*, `setDualCustomPalette(P, P)`
for *Dual*. `drawTileMap` reads the **background** bank and `drawSprite` the
**sprite** bank, so *None* means the tiles draw with whatever background palette
happens to be installed. In multi-palette mode it installs the lowest slot's
palette before the per-slot calls, which still win. At 1bpp there is no RGB565
palette, so every option behaves as *None*.

## Generated Shape

Single-palette mode: one shared palette in the header, tileset and index arrays
in the `.cpp`, and one layer struct per layer (`TileMap`, `TileMap2bpp` or
`TileMap4bpp` by detected BPP). Multi-palette mode: per-slot palettes
(`PLATFORMS_PALETTE`, …), `setBackgroundCustomPaletteSlot(slot, palette)` calls
in `init()`, and one tileset per layer.

An isometric export splits the header into **two blocks of the same namespace**:

| Block | Holds | Guard |
|---|---|---|
| Geometry | `TILE_WIDTH`/`TILE_HEIGHT`, `MAP_WIDTH`/`MAP_HEIGHT`, `ISO_PROJECTION` + its `static_assert`s, `TILESET_MAX_TILE_HEIGHT`, spawn, room metadata, attribute and behaviour layers | **none** (room metadata keeps its own `PIXELROOT32_ENABLE_GAMEPLAY_ROOM`) |
| Tile data | palettes, `TILESET_TILE_COUNT`, `TILESET_SPRITES`, `TILESET_FOOT_Y`, layer and room maps, `init()`, animations | `#ifdef PIXELROOT32_ENABLE_4BPP_SPRITES`, or `PIXELROOT32_ENABLE_2BPP_SPRITES` at 2bpp; **none at 1bpp** (the engine has no 1bpp switch) |

The `.cpp` wraps its whole body in the same guard. Gameplay places actors on
cells where no tile is drawn, so a build without the sprite-format switch must
still compile the geometry — that is why the split exists.

Every isometric constant is `inline constexpr`, and `TILESET_SPRITES` (plus
`<LAYER>_TILESET_SPRITES` in multi-palette mode) has **external linkage**: the
`.cpp` defines it without `static`, so every translation unit sees one array in
flash. An orthogonal export keeps its `static const` arrays.

`TILESET_MAX_TILE_HEIGHT` is the height of the tallest tile in the pool (in any
layer's pool, in multi-palette mode). A tile overhangs its cell by up to
`TILESET_MAX_TILE_HEIGHT - TILE_HEIGHT` pixels.

**`TILE_WIDTH`/`TILE_HEIGHT` are the cell STRIDE, not the bitmap extent.** A
2:1 isometric cell is 32×16 while a wall tile stands 32×40 on it; the height
lives in the tile's own `Sprite4bpp::height` and in `TILESET_FOOT_Y`. Never grow
the stride to fit the art.

## Tileset Tables

**Foot anchors.** `TILESET_FOOT_Y[i]` is the bitmap row of tile `i` that lands on
its cell's diamond centre — what lets a tall sprite stand on a short cell. The
exported `init()` assigns it (`background.tileFootY = TILESET_FOOT_Y;`), so a
consumer that calls `init()` and draws through `ISO_PROJECTION` wires nothing by
hand. Orthogonal exports emit no foot table: their art fills its cell.

In multi-palette mode each layer has its own pool and therefore its own table —
`BACKGROUND_TILESET_FOOT_Y`, `LAYER_1_TILESET_FOOT_Y`, … — because `tileFootY`
must be parallel to the `tiles[]` it anchors.

**Span tables.** Isometric 4bpp and 2bpp exports precompute each tile's opaque
run per row into `TILESET_SPAN_MIN_X` / `TILESET_SPAN_MAX_X` in flash: row `y` of
tile `i` covers `[MIN_X, MAX_X)`, and a fully transparent row is `0/0` — exactly
what the engine's `computeSpanTable` writes. The renderer uses them to skip
transparent padding, so you draw straight from the `const` descriptors with no
RAM copy and **no `computeSpanTable` pass at startup**. Slot 0 has no rows and
leaves its span pointers `nullptr`. Orthogonal and 1bpp exports emit none
(`Sprite` has no span fields). A 2bpp table matches the 2bpp blit's bit order
(pixel 0 in the low bits of each byte) since engine 1.15.0.

**Slot 0.** Index 0 is the empty tile `drawTileMap` never draws. In an isometric
export it is **zero-extent** — `{ nullptr, PALETTE, 0, 0, … }`, no bytes in the
data pool, no span rows, foot 0 — because the projected path reads
`tiles[0 .. tileCount)` to size its cull window and a zero extent cannot widen
it. An orthogonal export keeps its full-size blank tile.

**Deduplicated pool.** The exporter collapses identical tiles, so the tileset you
imported and `TILESET_SPRITES` differ in length **and order**. Every count, the
foot table and the span tables are sized to the pool — the number `init()` writes
to `tileCount`, which `footYFor()` bounds-checks. A third-party writer of these
headers must remap the same way and accumulate each sprite's byte offset from its
own extents; `TILE_WIDTH * TILE_HEIGHT` is never a valid stride.

## Data Formats

| BPP | Per row | Colours | Bit order |
|---|---|---|---|
| 1 | one `uint16_t` (width ≤ 16) | 2 | bit `width - 1` = leftmost pixel |
| 2 | `(width * 2 + 7) / 8` bytes | 4 | pixel 0 in the low bits of each byte |
| 4 | `(width * 4 + 7) / 8` bytes | 16 | pixel 0 in the low nibble |

**1bpp width limit.** A 1bpp `Sprite` stores one `uint16_t` per row, so a 1bpp
tile is at most **16 px** wide. When a 1bpp isometric scene uses a wider tile the
export reports a warning naming the widest tile, writes the files anyway, and
those tiles do not draw as authored. Add a colour so the scene exports at 2bpp,
or narrow the art. How the renderer reads each format is
`pixelroot32-sprite-renderer`'s contract.

## Rooms

Room metadata is optional and additive: a project without rooms exports exactly
as before. The header wraps it in `#if PIXELROOT32_ENABLE_GAMEPLAY_ROOM`, and the
symbol prefix is the scene file's name in upper case (`level1.h` →
`LEVEL1_ROOMS`).

```cpp
inline constexpr pixelroot32::gameplay::RoomData LEVEL1_ROOMS[] = {
    // originCol, originRow, cols, rows, { Up, Down, Left, Right }
    {  0, 0, 20, 15, { 0xFFFF, 0xFFFF, 0xFFFF,      1 } },
};
inline constexpr pixelroot32::gameplay::RoomLayer LEVEL1_ROOM_LAYER = {
    LEVEL1_ROOMS, 1, 16, 16   // rooms, roomCount, tileWidth, tileHeight
};
```

Both arrays are `const`, so they cost **0 bytes of SRAM**.

An isometric scene with rooms differs in three ways:

- `ROOMS` and `ROOM_LAYER` are `inline constexpr` instead of `static const`.
- `ROOM_LAYER`'s `tileWidth`/`tileHeight` are the **cell stride**
  (`TILE_WIDTH`/`TILE_HEIGHT`, e.g. `32, 16`); room origins stay in scene cells.
- The export adds **one map per room**, inside the depth guard:
  `<layer>_room<r>` (local cells from `(0, 0)`, the room's `cols`×`rows`) plus
  `NUM_ROOM_LAYERS` and `ROOM_LAYERS[NUM_ROOM_LAYERS]`, indexed like `ROOMS`.

With several layers, `ROOM_LAYERS` holds the bottom layer's maps and the header
adds `LAYERS_PER_ROOM` and one `<LAYER>_ROOM_LAYERS[]` per layer — drawing a room
is one `drawTileMap` per layer, bottom to top. Room maps share the deduplicated
tileset, `tileCount`, foot table and span tables with the whole-scene map, which
is still exported. `init()` wires every room map.

**Behaviour and attribute layers are not cropped.** They stay in scene cells even
in a room-mapped scene, so add the room's `originCol`/`originRow` when querying
them. Query attributes with `get_tile_attribute(layer, x, y, "type")` and flags
with `getTileFlags(layer, x, y)` (`TILE_SOLID`, …).

An orthogonal scene exports room metadata as before, with no room maps.

## Wiring and Drawing

```cpp
#include "level1.h"
level1::init();                                            // palettes, tileCount, foot table, rooms
renderer.drawTileMap(level1::layer_foreground, 0, 0);       // orthogonal
```

Animated layers expose `<LayerName>AnimManager()`; step it before drawing (a
legacy `getAnimManager()` alias maps to the Details layer when present).

Rooms: build the graph from the exported layer in `Scene::init()` and check the
return. `0` means the whole layer was rejected and `enterRoom()` is then a silent
no-op.

```cpp
pixelroot32::gameplay::RoomGraph<8> rooms;
if (pixelroot32::gameplay::buildRoomGraph(level1::LEVEL1_ROOM_LAYER, rooms) > 0) {
    setRoomGraph(&rooms);
    rooms.enterRoom(0, &camera);
}
```

An **isometric** scene either draws one screen per room through the scene's own
`ISO_PROJECTION` with `enterRoom(r, nullptr)`, or places the rooms in a scrolling
scene: `buildRoomGraph(layer, graph, ISO_PROJECTION, perRoom)` plus one
`drawTileMap` per room through `roomProjection(ISO_PROJECTION, ROOMS[r])`. The
rect geometry, `perRoom` and the camera consequences belong to
`pixelroot32-projection`; the graph API to `pixelroot32-gameplay-framework`.

## Re-exporting

The exported files are output, not source. The cycle is:

1. **File → Open** the `.pr32scene.bin`, edit, **Save**.
2. **File → Export** (Ctrl+E). Set every option again — the project file does not
   store them.
3. Export to the same folder; the three files are overwritten in place.
4. Rebuild. The exported `static_assert`s and your own assertions against the
   exported constants catch a scene that no longer fits the game.

Keep game code out of the generated files: a re-export deletes it without a diff.

## ESP32 Constraints

- All exported data is `const`/`constexpr` and lands in flash; a room layer costs
  0 B SRAM.
- Isometric span tables remove a per-sprite `computeSpanTable` pass and the RAM
  copy it would need; they ship as `static const` arrays carrying
  `PIXELROOT32_SCENE_FLASH_ATTR`, so *Legacy Format* drops the flash attribute
  from every exported table at once.
- One tileset per palette slot means one pool, one foot table and one span-table
  pair **per layer** in multi-palette mode — budget flash per layer, not once.
- The projected draw's executed cost on `esp32dev` is +1,744 B flash / +80 B SRAM
  (measured on `iso_dungeon`); see `docs/architecture/memory-system.md`.

## Gotchas

- **Export requires a license.** Without a valid Ed25519 v3 key the *Upgrade
  Required* dialog blocks export; every other editor feature still works.
- **A rule cannot be exported — only its result.** Anything the producer resolves
  per cell at draw time is not in the file. Bake it at export time or compute it
  in your own code.
- **The pool is deduplicated and reordered.** Never assume `TILESET_SPRITES`
  matches the tilesheet's order or length.
- **`TILE_WIDTH`/`TILE_HEIGHT` are the cell stride.** Do not set them to the
  diamond's on-screen size or to the bitmap's extents.
- **`TILESET_FOOT_Y` is wired by `init()`.** Calling `drawTileMap` before
  `init()` anchors every tile top-left; orthogonal exports have no table at all.
- **Palette bank choice is silent.** `drawTileMap` reads the background bank and
  `drawSprite` the sprite bank; a scene exported with *None* draws with whatever
  background palette is installed.
- **1bpp isometric tiles cap at 16 px.** The warning does not stop the export.
- **Options are per-export.** Nothing about the dialog survives in the project.
- **Behaviour layers keep scene cells.** Add the room origin when querying a room.
- **Orthogonal exports are unchanged.** Do not look for projection, foot table,
  span tables or room maps there.

## Agent Constraints

**DO**

- Call the generated `init()` before drawing, and draw from the exported
  `const` descriptors directly.
- Check `buildRoomGraph`'s return, and keep room origins in scene cells.
- Read the three `static_assert`s as the export's contract; add your own against
  the exported constants.
- Point at `pixelroot32-projection` for the basis, `pixelroot32-gameplay-framework`
  for the graph and `pixelroot32-sprite-renderer` for bit orders.
- Re-export from the editor and re-read the warning list after every change.

**DON'T**

- Do not edit the generated `.h`/`.cpp`/palette header — a re-export deletes it.
- Do not hand-write, hand-patch or "fix up" an isometric export: the stride, foot
  table, zero-extent slot 0, span tables and guard split are already emitted.
- Do not invent symbols. If it is not in the generated header, it is not exported.
- Do not index the tile pool by `TILE_WIDTH * TILE_HEIGHT`.
- Do not copy the exported map into RAM to mutate it; these are flash-resident
  `const` arrays.
- Do not treat an orthogonal export as if it had the isometric extras, or
  vice-versa.
