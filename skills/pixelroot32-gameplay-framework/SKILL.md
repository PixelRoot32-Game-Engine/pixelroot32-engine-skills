---
name: pixelroot32-gameplay-framework
description: The pixelroot32::gameplay primitives — GridSpec/GridMotion, StateMachine, ObjectPool, GameplayEventBus, InteractionTracker, RoomGraph/RoomLayout and DepthCompare. Use when building top-down, grid-locked, room-based or pooled-entity gameplay, or when wiring any PIXELROOT32_ENABLE_GAMEPLAY_* feature flag.
license: MIT
compatibility: opencode>=0.1.0
metadata:
  domain: engine
  subsystem: gameplay
  module: gameplay-framework
  platform: cross-platform
  engine_version: "1.9.0+unreleased"
---

## Overview

`pixelroot32::gameplay` is a family of independent, zero-heap, opt-in primitives shipped in
engine 1.9.0. Composition, not inheritance: none adds a virtual or a member to `Entity`/`Actor`,
and only `RoomGraph` and `InteractionTracker` have engine-side wiring. This is where the
engine's **top-down** story lives — `examples/bomberbot`, `examples/2048`,
`examples/legend_of_clone`, `examples/room_screen` and `examples/midway_clone` are built from
these pieces.

**Every flag in this family defaults to `0`.** Code using `GridSpec`, `GridMotion`,
`StateMachine`, `ObjectPool`, `RoomGraph` or `GameplayEventBus` without the matching
`-D ..._=1` does not compile — each header is an `#if` that otherwise leaves an empty
translation unit. Adding the flag is part of writing the code, not a follow-up.

## Key APIs

### GridSpace — cell ⇄ world conversion

**Header** `include/gameplay/GridSpace.h` · **Namespace** `pixelroot32::gameplay` ·
**Feature gate** `PIXELROOT32_ENABLE_GAMEPLAY_GRID_SPACE`

```cpp
struct GridSpec {                       // six plain ints — NOT Scalar
    int originX = 0, originY = 0;       // world pixels of cell (0,0)'s top-left
    int cellWidth = 1, cellHeight = 1;  // MUST be >= 1
    int cols = 0, rows = 0;             // extent, for containsCell()
};
constexpr bool gridSpecIsValid(const GridSpec& spec);
constexpr int           cellToWorldX(int cellX, const GridSpec& spec);   // never divides
constexpr int           cellToWorldY(int cellY, const GridSpec& spec);
constexpr math::Vector2 cellToWorld(int cellX, int cellY, const GridSpec& spec);
constexpr int worldToCellX(int worldX, const GridSpec& spec);            // FLOOR semantics
constexpr int worldToCellY(int worldY, const GridSpec& spec);
inline    int worldToCellX(math::Scalar worldX, const GridSpec& spec);   // not constexpr
inline    int worldToCellY(math::Scalar worldY, const GridSpec& spec);
int worldToCellX(double, const GridSpec&) = delete;   // hard error on both targets
int worldToCellY(double, const GridSpec&) = delete;
constexpr bool containsCell(int cellX, int cellY, const GridSpec& spec);

// Declare inline constexpr and assert at the declaration site — the aggregate cannot
// assert its own arguments under -fno-exceptions. No snapToGrid, no runtime grid object.
inline constexpr gameplay::GridSpec kBoardGrid{
    kBoardOriginX, kBoardOriginY, kCellSize, kCellSize, kCols, kRows};
static_assert(gameplay::gridSpecIsValid(kBoardGrid),
              "kBoardGrid exceeds Scalar's range or has an invalid cell size.");
```

### GridMotion — one actor's cell-to-cell step

**Header** `include/gameplay/GridMotion.h` · **Namespace** `pixelroot32::gameplay` ·
**Feature gate** `PIXELROOT32_ENABLE_GAMEPLAY_GRID_SPACE` (shared; no `..._GRID_MOTION` flag)

```cpp
struct GridMotion {                 // five plain ints — NOT Scalar. 20 B per moving actor.
    int cellX = 0, cellY = 0;       // LOGICAL cell — what every gameplay rule reads
    int toX = 0,   toY = 0;         // target cell; == cellX/cellY at rest
    int progress = 0;               // 0 == at rest; 1..stepsPerCell-1 in flight
};
constexpr bool isMoving(const GridMotion& motion);
constexpr void placeAt(GridMotion& motion, int cellX, int cellY);       // spawn/teleport
constexpr void beginStep(GridMotion& motion, int toCellX, int toCellY); // no validation
constexpr bool tickStep(GridMotion& motion, int stepsPerCell);          // true == arrival EDGE
inline math::Vector2 interpolatedWorld(const GridMotion& motion, int stepsPerCell,
                                       const GridSpec& spec);
```

Enterability, direction choice, arrival reactions and input buffering are **game code**;
`GridMotion` owns only the mechanics. A second `interpolatedWorld(const GridMotion&, int,
const ProjectionSpec&)` overload lives in a nested `#if PIXELROOT32_ENABLE_PROJECTION` for
isometric/oblique layouts and is guarded on both flags — see **`pixelroot32-projection`**.

### StateMachine — FSM over a caller-owned const table

**Header** `include/gameplay/StateMachine.h` · **Namespace** `pixelroot32::gameplay` ·
**Feature gate** `PIXELROOT32_ENABLE_GAMEPLAY_STATE_MACHINE`

```cpp
using StateId = uint8_t;
class StateMachine {
public:
    static constexpr StateId kInvalidStateId = 0xFF;
    using EnterFn  = void (*)(void* owner, StateId fromState);
    using UpdateFn = void (*)(void* owner, unsigned long deltaTime, uint32_t timeInStateMs);
    using ExitFn   = void (*)(void* owner, StateId toState);
    struct State { EnterFn onEnter; UpdateFn onUpdate; ExitFn onExit; StateId id; };

    void     configure(void* owner, const State* table, uint8_t stateCount);  // NOT copied
    void     start(StateId initialState);
    void     update(unsigned long deltaTime);
    bool     requestState(StateId nextState);   // false == no row with that id
    void     restartState();                    // genuine onExit/onEnter re-entry
    void     reset();                           // un-started; fires NO callback
    StateId  getCurrentState() const;
    StateId  getPreviousState() const;
    uint32_t getTimeInState() const;            // saturating ms — never Scalar
    bool     isRunning() const;
    uint8_t  getTransitionOverflowCount() const;
};
```

`kMaxChainedTransitions` is **private** (`StateMachine.h:158`, value `8`) — game code observes
overflow only through `getTransitionOverflowCount()`. The table must outlive the machine; the
convention is a class-static `const` array in flash, configured once
(`examples/metroidvania/src/PlayerActor.cpp:56`, `:69`):
`stateMachine.configure(this, kPlayerStates, 4); stateMachine.start(kIdleId);`

### ObjectPool — fixed-capacity placement-new slots

**Header** `include/gameplay/ObjectPool.h` · **Namespace** `pixelroot32::gameplay` ·
**Feature gate** `PIXELROOT32_ENABLE_GAMEPLAY_OBJECT_POOL`

```cpp
template <typename T, uint16_t N>
class ObjectPool {                    // copy ctor and copy assignment are deleted
public:
    static constexpr uint16_t kCapacity = N;
    static constexpr uint16_t kEnd = 0xFFFFu;   // nextLive()/indexOf() sentinel
    template <typename... Args> T* acquire(Args&&... args);  // nullptr when full
    bool     release(T* object);                 // false == null/foreign/already free
    bool     releaseAt(uint16_t index);
    void     reset();                            // ~T() over every live slot
    uint16_t capacity() const;
    uint16_t size() const;
    bool     isFull() const;
    bool     isLive(uint16_t index) const;
    T*       at(uint16_t index);                 // nullptr when out of range or dead
    uint16_t indexOf(const T* object) const;     // kEnd when not found
    uint16_t nextLive(uint16_t from) const;      // kEnd when none remains
};
// Iterate with nextLive() against kEnd, never for (i = 0; i < N; ++i). T needs no
// default constructor — acquire() forwards its arguments to T's.
```

### GameplayEventBus — Engine-owned fixed-capacity FIFO

**Header** `include/gameplay/GameplayEventBus.h` (+ `GameplayEvent.h`, `GameplayEventType.h`) ·
**Namespace** `pixelroot32::gameplay` · **Feature gate** `PIXELROOT32_ENABLE_GAMEPLAY_EVENTS`

```cpp
enum class GameplayEventType : uint8_t {
    None = 0, TriggerEnter, TriggerExit, Interact, UserBase = 64  // game codes start at 64
};
struct GameplayEvent {
    void*             userData  = nullptr;  // never owned or freed by the engine
    math::Scalar      value{};              // Scalar — build with math::toScalar()
    uint16_t          entityIdA = 0, entityIdB = 0;
    GameplayEventType type      = GameplayEventType::None;
};
class GameplayEventBus {
public:
    static constexpr uint16_t kCapacity =
        static_cast<uint16_t>(platforms::config::GameplayEventQueueCapacity);
    bool     publish(const GameplayEvent& event);  // false == DROPPED (buffer full)
    bool     consume(GameplayEvent& outEvent);     // FIFO
    void     clear();                              // does NOT reset the drop counter
    uint32_t getDroppedCount() const;              // monotonic
    uint16_t size() const;  bool isEmpty() const;  bool isFull() const;
};
```

One instance, owned by `Engine`: `engine.getGameplayEventBus()` (`include/core/Engine.h:266`).
`SceneManager` drains it on every `SceneSwap`, but **not** on `pushScene()`/`popScene()`.
Non-atomic — publishing from an ISR or the audio task is unsupported and corrupts the ring.

### InteractionComponent / InteractionTracker — trigger enter/exit edges

**Header** `include/gameplay/InteractionComponent.h`, `include/gameplay/InteractionTracker.h` ·
**Namespace** `pixelroot32::gameplay` · **Feature gate**
`PIXELROOT32_ENABLE_INTERACTION_TRIGGERS` (**requires** `PIXELROOT32_ENABLE_PHYSICS=1`)

```cpp
struct InteractionComponent {
    using TriggerFn  = void (*)(void* owner, core::Actor* other);
    using InteractFn = void (*)(void* owner, core::Actor* instigator);
    void*      owner      = nullptr;
    TriggerFn  onEnter    = nullptr;
    TriggerFn  onExit     = nullptr;
    InteractFn onInteract = nullptr;   // manual-call only — the engine NEVER fires it
    void invokeInteract(core::Actor* instigator);
};
class InteractionTracker {
public:
    void registerActor(core::Actor* actor, InteractionComponent* comp);  // entityId != 0
    void unregisterActor(core::Actor* actor);
    void setEventBus(GameplayEventBus* bus);   // only under PIXELROOT32_ENABLE_GAMEPLAY_EVENTS
    void beginFrame();                         // these three are driven by CollisionSystem::
    void recordPair(core::Actor* a, core::Actor* b);   // triggerCallbacks(), never game code
    void endFrame();
    void reset();
};

collisionSystem.setInteractionTracker(&tracker_);   // include/physics/CollisionSystem.h:131
tracker_.registerActor(&door_, &door_.interaction); // AFTER addEntity assigned entityId
```

### DepthCompare — ready-made paint-order comparators

**Header** `include/gameplay/DepthCompare.h` · **Namespace** `pixelroot32::gameplay` ·
**Feature gate** none for `compareByBottomY`; `PIXELROOT32_ENABLE_DEPTH_SORT` for
`compareByDepthKey`

```cpp
inline bool compareByBottomY(core::Entity* a, core::Entity* b);   // UNGUARDED
#if PIXELROOT32_ENABLE_DEPTH_SORT
inline bool compareByDepthKey(core::Entity* a, core::Entity* b);
#endif
```

`compareByBottomY` orders by `position.y + math::toScalar(height)` — the **top-down default**.
The `Scene` members it plugs into (`depthComparator`, `depthSortEnabled`, `Scene.h:277-280`)
are themselves behind `PIXELROOT32_ENABLE_DEPTH_SORT`, so the assignment needs the flag even
though the function does not. `compareByDepthKey` belongs to **`pixelroot32-projection`**.

### RoomGraph / RoomLayout — screen-by-screen rooms

**Header** `include/gameplay/RoomGraph.h`, `include/gameplay/RoomLayout.h` ·
**Namespace** `pixelroot32::gameplay` · **Feature gate** `PIXELROOT32_ENABLE_GAMEPLAY_ROOM`

```cpp
enum class RoomDir : uint8_t { Up = 0, Down = 1, Left = 2, Right = 3 };
struct Room {                                    // camera bounds ARE Scalar
    math::Scalar cameraMinX, cameraMinY, cameraMaxX, cameraMaxY;
    int16_t tileOriginCol, tileOriginRow, tileCols, tileRows;   // -1 / 0 when unused
    bool    hasTileWindow;
    static constexpr int INVALID_ROOM = 0xFFFF;
};
class RoomGraphBase {   // abstract; what Scene stores, type-erased over N
    virtual void enterRoom(uint16_t idx, graphics::Camera2D* camera) = 0;
    virtual uint16_t currentRoomIndex() const = 0;
    virtual bool isValidIdx(uint16_t idx) const = 0;
};
template <uint16_t N> class RoomGraph : public RoomGraphBase {
    uint16_t    addRoom(math::Scalar minX, math::Scalar minY,
                        math::Scalar maxX, math::Scalar maxY);   // 0xFFFF when full
    void        setTileWindow(uint16_t roomIdx, int16_t originCol, int16_t originRow,
                              int16_t cols, int16_t rows);
    void        connect(uint16_t fromIdx, uint16_t toIdx, RoomDir dir);
    void        enterRoom(uint16_t idx, graphics::Camera2D* camera) override;
    void        setOnEnter(void (*fn)(int fromIdx, int toIdx, void* userData),
                           void* userData = nullptr);
    uint16_t    currentRoomIndex() const override;   // 0xFFFF before the first enterRoom()
    const Room& getRoom(uint16_t idx) const;         // NOT bounds-checked
    uint16_t    roomCount() const;
    bool        isValidIdx(uint16_t idx) const override;
    uint8_t     getConnections(uint16_t idx, int out[], uint8_t maxOut) const;
};
// RoomLayout.h
inline constexpr uint16_t kNoRoomConnection  = 0xFFFF;  // :59 — the wall sentinel
inline constexpr uint16_t kGraphFull         = 0xFFFF;  // addRoom()'s "full" return
inline constexpr int      kMaxRoomWorldCoord = 32767;
struct RoomData { uint16_t originCol, originRow, cols, rows, connections[4]; };  // 16 B, TILES
struct RoomLayer { const RoomData* rooms; uint16_t roomCount; uint8_t tileWidth, tileHeight; };
template <uint16_t N> uint16_t buildRoomGraph(const RoomLayer& layer, RoomGraph<N>& graph);

// What the Tilemap Editor's room export emits, inside #if PIXELROOT32_ENABLE_GAMEPLAY_ROOM:
static const pixelroot32::gameplay::RoomData ROOMSCREEN_MAIN_SCENE_ROOMS[] = {
    //  originCol, originRow, cols, rows, { Up, Down, Left, Right }
    {  0,  0, 15, 15, { 0xFFFF,      2, 0xFFFF,      1 } },
    { 15,  0, 15, 15, { 0xFFFF,      3,      0, 0xFFFF } },
};
static const pixelroot32::gameplay::RoomLayer ROOMSCREEN_MAIN_SCENE_ROOM_LAYER = {
    ROOMSCREEN_MAIN_SCENE_ROOMS, 4, 16, 16 };   // rooms, roomCount, tileWidth, tileHeight
```

`buildRoomGraph` validates the whole layer first (one out-of-range room rejects everything and
returns `0`), **appends** rooms, and remaps layer-local connection indices into graph space.
`Scene::onRoomEnter(int fromIdx, int toIdx)` (`include/core/Scene.h:157`) is the hook, also
under `GAMEPLAY_ROOM` — but see Gotchas: it is not wired for you.

## Feature Flags

Defined in `include/platforms/PlatformDefaults.h`, mirrored as `constexpr bool` in
`pixelroot32::platforms::config` (`include/platforms/EngineConfig.h`). **All default `0`.**
Prefix every name below with `PIXELROOT32_ENABLE_`.

| Flag | Enables | Default | Mirror constant |
|---|---|---|---|
| `..._GAMEPLAY_EVENTS` (`:102`) | `GameplayEventBus`, `Engine::getGameplayEventBus()` | `0` | `config::EnableGameplayEvents` |
| `..._INTERACTION_TRIGGERS` (`:106`) | `InteractionTracker`, `CollisionSystem::setInteractionTracker()` | `0` | `config::EnableInteractionTriggers` |
| `..._SPATIAL_QUERY` (`:110`) | `CollisionSystem::queryRadius/queryBox` | `0` | `config::EnableSpatialQuery` |
| `..._DEPTH_SORT` (`:114`) | `Scene::depthComparator`/`depthSortEnabled`, `Entity::depthKey`, `compareByDepthKey` | `0` | `config::EnableDepthSort` |
| `..._GAMEPLAY_STATE_MACHINE` (`:118`) | `StateMachine` | `0` | `config::EnableGameplayStateMachine` |
| `..._GAMEPLAY_OBJECT_POOL` (`:122`) | `ObjectPool<T,N>` | `0` | `config::EnableGameplayObjectPool` |
| `..._GAMEPLAY_GRID_SPACE` (`:126`) | `GridSpec` **and** `GridMotion` | `0` | `config::EnableGameplayGridSpace` |
| `..._GAMEPLAY_ROOM` (`:130`) | `RoomGraph<N>`, `RoomLayout`, `Scene::onRoomEnter`/`setRoomGraph` | `0` | `config::EnableGameplayRoom` |
| `..._PROJECTION` (`:142`) | `ProjectionSpec` + the `interpolatedWorld` overload | `0` | `config::EnableProjection` |

Two **require `PIXELROOT32_ENABLE_PHYSICS=1`** and fail the build with an `#error` in
`PlatformDefaults.h` otherwise: `INTERACTION_TRIGGERS` and `SPATIAL_QUERY` — `CollisionSystem`
and `SpatialGrid` only exist when physics is on. `SPATIAL_QUERY`'s API is documented in
**`pixelroot32-physics`**; it is named here only as part of the family. Sizing constants:
`GAMEPLAY_EVENT_QUEUE_CAPACITY` (default `32`, `config::GameplayEventQueueCapacity`),
`GAMEPLAY_MAX_INTERACTIVE_ACTORS` (default `16`, `config::GameplayMaxInteractiveActors`).

## Composition Patterns

**Grid-locked top-down movement — `examples/bomberbot`** (`GRID_SPACE=1`, `DEPTH_SORT=1`):
13x11 board of 16 px cells; `inline constexpr GridSpec kBoardGrid` folds the status band into
`originY` so `containsCell()` rejects it for free. One `GridMotion mv` per actor
(`PlayerActor.h:134`), `kPlayerStepsPerCell = 12`, `kEnemyStepsPerCell = 20`. Logic runs on a
fixed 20 ms accumulator (`kLogicStepMs`, clamped to 4 catch-up steps), so `tickStep()` counts
steps, never milliseconds. Bomberbot writes its own bottom-edge `drawLowerLast` comparator
rather than using `compareByBottomY`, but the rule is identical.

**Screen-by-screen rooms — `examples/legend_of_clone`, `examples/room_screen`**
(`GAMEPLAY_ROOM=1`): both own a `RoomGraph<N>`, build it from an exported `RoomLayer`, register
a static trampoline, hand the graph to the `Scene`, then enter the start room.

```cpp
rooms_ = gameplay::RoomGraph<kMaxRooms>{};        // buildRoomGraph APPENDS — reset first
const uint16_t built = gameplay::buildRoomGraph(DUNGEON_ROOM_LAYER, rooms_);
if (built == 0) { return; }                        // a rejected layer builds nothing
rooms_.setOnEnter(&TopDownScene::onRoomEnterCallback, this);
setRoomGraph(&rooms_);
rooms_.enterRoom(startRoom, &camera_);
snapCameraToRoom(startRoom);                       // enterRoom only CLAMPS bounds
```

`legend_of_clone` layers scrolling transitions on top (widen bounds during the slide,
`enterRoom()` on arrival to snap back). `room_screen` is the minimal reference: a 2x2 grid of
15x15-tile rooms in a `RoomGraph<4>` built from a Tilemap-Editor-exported layer, camera snapped
with `camera.setPosition(math::Vector2(room.cameraMinX, room.cameraMinY))`.

**Pooled projectiles — `examples/midway_clone`** (`OBJECT_POOL=1`): four `Scene`-member pools
(`ObjectPool<Bullet,8>`, `ObjectPool<Bullet,12>`, `ObjectPool<Enemy,10>`,
`ObjectPool<Explosion,6>`), all `reset()` from the scene's `init()` *after* `Scene::init()` ran.
`acquire()` returning `nullptr` is a normal outcome the game handles (a full pool drops the
spawn rather than stalling the wave queue); a `static_assert` pins capacity against the largest
wave. Waves are keyed to camera Y, not to a clock.

**Grid math without motion — `examples/2048`** (`GRID_SPACE=1` only): `GridSpec` +
`cellToWorldX/Y` + `gridSpecIsValid`, no `GridMotion`, no depth sort. Swipes arrive through
`Scene::onUnconsumedTouchEvent` (see **`pixelroot32-touch-input`**).

## ESP32 Constraints

Figures from `docs/architecture/memory-system.md` (Gameplay Framework Phase 1/2/3 sections):

- `GridSpec`: `sizeof == 24 B`, but `constexpr` puts it in `.rodata` — **0 B SRAM**. A runtime
  `GridSpec` costs 24 B; no shipped consumer has one.
- `GridMotion`: `sizeof == 20 B` in `.bss` per moving actor — 480 B worst case at the ESP32-C3
  ceiling of 24 entities.
- `StateMachine`: 20 B/instance (C3) / 32 B (native), plus one shared `const` table in flash.
- `ObjectPool<T,N>`: `N * sizeof(T)` + `uint32_t liveWords_[(N+31)/32]` + two `uint16_t`
  counters, identical on both targets.
- `GameplayEventBus`: ~512 B (C3) / ~768 B (native) at the default 32 slots.
- `InteractionTracker`: ~704 B on ESP32 with `PHYSICS_MAX_CONTACTS=64`; ~1.2 KB native (128).
- `DEPTH_SORT`: ~8 B per `Scene` **plus 4 B per `Entity`** on 32-bit (`depthKey`).
- `RoomData`/`RoomLayer`: `static const` => flash. A 2-room layer is 40 B flash, 0 B SRAM.
- Any flag left at `0` contributes **0 B** — the header is an empty `#if` block.

`interpolatedWorld()` costs one integer division per axis per call (`stepsPerCell` is a runtime
value) — call it once per frame and cache. `worldToCellX/Y` cost exactly one `div` and no
`rem`; `cellToWorld*` never divides.

## Gotchas

- **`Scene::onRoomEnter` is not wired for you.** `setRoomGraph()` only stores the pointer
  (`Scene.h:180`); nothing in `Scene.cpp` touches `onEnter`. Call
  `graph.setOnEnter(&MyScene::trampoline, this)` with a static trampoline forwarding to the
  virtual, or `onRoomEnter()` stays permanently silent.
- **`enterRoom()` does not move the camera** — it only sets `setBounds`/`setVerticalBounds` and
  fires `onEnter`. Without a follow-up `camera.setPosition(...)` the previous room stays up.
- **`InteractionTracker` does nothing until `CollisionSystem::setInteractionTracker()` is
  called** — with no tracker set, `triggerCallbacks()` behaves as with the flag off.
- **Register interactive actors only after `addEntity`** — `registerActor()` asserts
  `actor->entityId != 0`, and the `CollisionSystem` is what assigns that id.
- **A callback's `other` is null unless both sides are registered.** Bus events fire for every
  contact pair; component callbacks resolve `other` through the registry.
- **`GameplayEventBus` drops on overflow (drop-newest) and never blocks.** `publish()`
  returning `false` is the only synchronous signal; `getDroppedCount()` is the diagnostic and
  survives `clear()`.
- **`StateMachine` chained transitions cap at 8.** Chains requested from `onEnter`/`onExit` are
  drained iteratively by the initiating `requestState()`, never recursively. Past the cap the
  drain stops, the queued request is discarded, and `getTransitionOverflowCount()` (saturating
  `uint8_t`, not cleared by `reset()`) increments — the machine sits in the last state it
  actually entered, so a self-retriggering enter/exit pair freezes there silently.
- **`update()` credits the frame's delta to the state being LEFT.** Transitioning from inside
  `onUpdate` returns with `getTimeInState() == 0`; code ported from a hand-rolled
  `changeState()` loses one frame per transition.
- **`requestState(getCurrentState())` is a no-op** — use `restartState()` for a real re-entry.
  **`reset()` fires no `onExit`**; it is teardown only.
- **A `double` into `worldToCellX/Y` is a compile error, not a silent narrowing** — those
  overloads are `= delete` because `double` resolves differently on native (`Scalar == float`)
  than on the C3.
- **Never put an `ObjectPool` in arena memory.** `Scene::resetState()` rewinds the bump offset
  without running destructors, so `~T()` never runs and two live objects alias one address.
  Make it a plain member or a file-scope `static`, and `reset()` it from `init()` **after**
  `Scene::init()`.
- **`getRoom()` does not bounds-check** — call `isValidIdx()` first. **`currentRoomIndex()` is
  `0xFFFF` before the first `enterRoom()`**, and the first `onRoomEnter` reports
  `fromIdx == 0xFFFF`.
- **`buildRoomGraph` appends.** Re-running `init()` on a scene swap without
  `graph = RoomGraph<N>{};` silently adds nothing and returns `0`.
- **Game event codes start at `GameplayEventType::UserBase` (64)** — below that is reserved.

## Common Patterns

```cpp
// Grid step with a game-owned enterability rule (examples/bomberbot)
if (gameplay::isMoving(mv)) {
    if (gameplay::tickStep(mv, kPlayerStepsPerCell)) { onArriveAtCell(); }
} else {
    const int nx = mv.cellX + dirX_, ny = mv.cellY + dirY_;
    if (gameplay::containsCell(nx, ny, kBoardGrid) && board_->isWalkable(nx, ny)) {
        gameplay::beginStep(mv, nx, ny);
    }
}
position = gameplay::interpolatedWorld(mv, kPlayerStepsPerCell, kBoardGrid);

// Pool iteration — releasing the current slot mid-loop is safe
using BulletPool = gameplay::ObjectPool<Bullet, kMaxPlayerBullets>;
for (uint16_t i = pool.nextLive(0); i != BulletPool::kEnd; i = pool.nextLive(i + 1)) {
    if (offScreen(*pool.at(i))) { pool.releaseAt(i); }
}

// Top-down depth ordering, in Scene::init()
#if PIXELROOT32_ENABLE_DEPTH_SORT
    depthComparator  = &gameplay::compareByBottomY;
    depthSortEnabled = true;   // required: actors move every frame
#endif

// Draining and publishing on the bus
auto& bus = engine.getGameplayEventBus();
gameplay::GameplayEvent ev;
while (bus.consume(ev)) { /* dispatch on ev.type / ev.entityIdA / ev.entityIdB */ }

gameplay::GameplayEvent hit;
hit.type  = static_cast<gameplay::GameplayEventType>(
                static_cast<uint8_t>(gameplay::GameplayEventType::UserBase) + 3);
hit.value = math::toScalar(damage);   // Scalar field — never a raw float literal
if (!bus.publish(hit)) { /* dropped; getDroppedCount() was bumped */ }
```

## Agent Constraints

- **Never write a `gameplay::` symbol without also adding its `-D` flag** to the example's
  `platformio.ini` (`lib/platformio.ini` in the shipped examples). All flags default `0`.
- **Never invent APIs.** There is no `snapToGrid`, no runtime `GridSpace` object, no
  `StateMachine::getStateCount()`, no `ObjectPool::begin()/end()`, no
  `RoomGraph::getRoomByDir()`, and no engine-driven `onInteract()`.
- **Never iterate an `ObjectPool` with a raw index loop** — use `nextLive()` / `kEnd`.
- **Never allocate in `update()`/`draw()`**: no `new`, `malloc`, `std::vector`,
  `std::shared_ptr` or `std::function`. Every primitive here is fixed-capacity by design.
- **Respect the `Scalar`-vs-`int` split.** `GridSpec` (6 fields) and `GridMotion` (5 fields)
  are **all plain `int`** — never wrap them in `math::toScalar()`.
  `Room::cameraMinX/MinY/MaxX/MaxY`, `RoomGraph::addRoom()`'s four arguments and
  `GameplayEvent::value` **are** `math::Scalar` — build those with `math::toScalar()`, never a
  raw `float`. `Room::tileOriginCol/Row`/`tileCols`/`tileRows` are `int16_t`; every `RoomData`
  field is `uint16_t`; pool and room indices are `uint16_t`; `StateId` is `uint8_t`.
- **Keep flag pairings honest.** `INTERACTION_TRIGGERS` and `SPATIAL_QUERY` require
  `PIXELROOT32_ENABLE_PHYSICS=1`. The `ProjectionSpec` overload of `interpolatedWorld` requires
  `GAMEPLAY_GRID_SPACE` **and** `PROJECTION`. `InteractionTracker::setEventBus()` requires
  `GAMEPLAY_EVENTS`.
- **C++17, `-fno-exceptions`, no RTTI.** PascalCase types, camelCase methods.
- **Cross-reference, do not duplicate**: `pixelroot32-projection` (isometric/oblique,
  `ProjectionSpec`, `compareByDepthKey`), `pixelroot32-physics` (`CollisionSystem`,
  `queryRadius`/`queryBox`), `pixelroot32-camera2d` (`setBounds`, `setVerticalBounds`),
  `pixelroot32-scene-manager` (`Scene::init`/`resetState`, scene swaps),
  `pixelroot32-entity-actor` (`Actor`, `entityId`, `renderLayer`),
  `pixelroot32-sprite-renderer` (drawing what these primitives position).
