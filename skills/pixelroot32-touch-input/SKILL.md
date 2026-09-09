---
name: pixelroot32-touch-input
description: Touch input pipeline with adapter pattern (XPT2046, GT911), state machine gesture detection, pull-based event dispatching, circular buffer for points, calibration, and ActorTouchController. Use when implementing touch input, gesture recognition, or multi-touch support.
license: MIT
compatibility: opencode>=0.1.0
metadata:
  domain: engine
  subsystem: input
  module: touch
  platform: cross-platform
  engine_version: "1.9.0+unreleased"
---

## Overview

PixelRoot32 provides a complete touch input pipeline: raw hardware → `TouchAdapter` → normalized `TouchPoint` → `TouchStateMachine` (gesture detection) → `TouchEventQueue` → `TouchEventDispatcher` (pull-based consumer API). The adapter pattern supports XPT2046 (single-touch resistive) and GT911 (5-point capacitive) with hardware abstraction. Also includes `InputManager` for physical buttons + keyboard + touch.

**See also**: `pixelroot32-ui-system` for `UIManager::processEvents` (the main consumer of `TouchEvent`); `pixelroot32-gameplay-framework` for wiring unconsumed gestures into game state; `pixelroot32-projection` for mapping screen-space touch coordinates onto world/isometric space.

## Key APIs

### TouchAdapter (Abstract)

**Header**: `include/input/TouchAdapter.h`
**Namespace**: `pixelroot32::input`

```cpp
class MyAdapter : public TouchAdapter {
    bool init() override;
    void update() override;
    uint8_t getTouchPoints(TouchPoint* points) override;  // Max 5
};
```

Concrete adapters are selected at compile time via `TOUCH_DRIVER_*`:
```cpp
// Defined in platformio.ini
// -D TOUCH_DRIVER_XPT2046  — SPI resistive, 1 point
// -D TOUCH_DRIVER_GT911    — I2C capacitive, 5 points
```

### TouchPoint (Normalized Data)

**Header**: `include/input/TouchPoint.h`
**Namespace**: `pixelroot32::input`

```cpp
TouchPoint tp(x, y, pressed, id, timestamp);

// Properties
int16_t x;           // 0 = left edge, clamped to display width
int16_t y;           // 0 = top edge
bool pressed;        // Touch active?
uint8_t id;          // 0-4 (touch identifier)
uint32_t ts;         // Monotonic timestamp in ms

// Validation
bool valid = tp.isValid(320, 240);
bool empty = tp.isEmpty();  // !pressed

// Max simultaneous points
constexpr uint8_t TOUCH_MAX_POINTS = 5;
```

### TouchEvent (Gesture Events)

**Header**: `include/input/TouchEvent.h`
**Namespace**: `pixelroot32::input`

```cpp
// Compact 12-byte event
TouchEvent event(TouchEventType::Click, touchId, x, y, timestamp, TouchEventFlags::Primary);

// Type detection
TouchEventType type = event.getType();
bool down = (type == TouchEventType::TouchDown);
bool click = (type == TouchEventType::Click);
bool drag = (type == TouchEventType::DragMove);

// Consumed flag (used by UI system)
event.consume();
bool consumed = event.isConsumed();
```

### TouchEventTypes

**Header**: `include/input/TouchEventTypes.h`

```cpp
// All gesture event types
TouchEventType::None        // No event
TouchEventType::TouchDown   // Press started
TouchEventType::TouchUp     // Release
TouchEventType::Click       // Quick press+release
TouchEventType::DoubleClick // Two clicks within 400ms
TouchEventType::LongPress   // Held >800ms
TouchEventType::DragStart   // Moved past 10px threshold
TouchEventType::DragMove    // Continuing drag
TouchEventType::DragEnd     // Released while dragging

// Timing constants (adjustable)
TouchTiming::CLICK_MAX_DURATION      = 300ms
TouchTiming::DOUBLE_CLICK_INTERVAL   = 400ms
TouchTiming::LONG_PRESS_THRESHOLD    = 800ms
TouchTiming::DRAG_THRESHOLD          = 10px
```

### TouchStateMachine

**Header**: `include/input/TouchStateMachine.h`
**Namespace**: `pixelroot32::input`

```cpp
TouchStateMachine machine;

// Feed raw touch state each frame
machine.update(touchId, pressed, x, y, timestamp, outputQueue);

// State queries
TouchState state = machine.getState(touchId);
// Idle → Pressed → LongPress | Dragging → Idle

bool active = machine.isActive();
uint32_t duration = machine.getPressDuration(touchId, now);

// Reset
machine.reset(touchId);
machine.resetAll();
```

#### State Transitions

```
Idle ──TouchDown──► Pressed ──release──► Idle (Click)
                       │
                       ├── hold >800ms ──► LongPress
                       │                      │
                       └── move >10px ──► Dragging
                                              │
                                    release ──► Idle (DragEnd)
```

### TouchEventDispatcher (Consumer API)

**Header**: `include/input/TouchEventDispatcher.h`
**Namespace**: `pixelroot32::input`
**Feature gate**: none — this header contains no `#if` at all.

```cpp
TouchEventDispatcher dispatcher;

// Feed raw input (per frame)
dispatcher.processTouch(touchId, pressed, x, y, timestamp);
// Or batch:
dispatcher.processTouchPoints(points, count, timestamp);

// Pull-based retrieval (consumes from queue)
TouchEvent events[16];
uint8_t count = dispatcher.getEvents(events, 16);

// Peek without consuming
count = dispatcher.peekEvents(events, 16);
bool hasEvents = dispatcher.hasEvents();
uint8_t pending = dispatcher.getEventCount();

// State
TouchState state = dispatcher.getTouchState(0);
bool touching = dispatcher.isTouchActive();

// Reset
dispatcher.clearEvents();
dispatcher.reset();  // Clear events + reset state machine
```

### InputManager (Buttons + Keyboard + Touch)

**Header**: `include/input/InputManager.h`
**Namespace**: `pixelroot32::input`
**Feature gate**: none — the embedded `TouchEventDispatcher` (`:18` include, `:152` member) and its accessors are compiled unconditionally, independent of `PIXELROOT32_ENABLE_TOUCH`.

```cpp
InputConfig config = { pins, BUTTON_COUNT };
InputManager input(config);
input.init();

// Update each frame
// Native:
input.update(dt, sdlKeyboardState);
// ESP32:
input.update(dt);

// Button state access
bool pressed = input.isButtonPressed(BUTTON_A);
bool down = input.isButtonDown(BUTTON_A);   // Just pressed this frame
bool released = input.isButtonReleased(BUTTON_A);
bool clicked = input.isButtonClicked(BUTTON_A);  // Press + release

// Touch accessors — always declared, even with PIXELROOT32_ENABLE_TOUCH=0.
// With the flag off these compile and link, but Engine never polls the
// hardware, so the queue stays permanently empty.
TouchEvent events[16];
uint8_t n = input.getTouchEvents(events, 16);   // :101
bool any  = input.hasTouchEvents();             // :107
```

### TouchManager (Adapter Integration)

**Header**: `include/input/TouchManager.h`
**Namespace**: `pixelroot32::input`
**Feature gate**: none. `TouchManager`'s `#if`s select a driver/platform (`#ifdef PLATFORM_NATIVE`, `#if defined(TOUCH_DRIVER_XPT2046)`) — they never test `PIXELROOT32_ENABLE_TOUCH`.

```cpp
TouchManager touchMgr(320, 240);  // Display bounds
touchMgr.init();
touchMgr.update(dt);

// Get active points
TouchPoint points[5];
uint8_t count = touchMgr.getTouchPoints(points);

// State
bool touching = touchMgr.isTouching();
uint8_t pointCount = touchMgr.getTouchCount();
```

### ActorTouchController

**Header**: `include/input/ActorTouchController.h`

```cpp
// Maps touch to actor control (drag, click, etc.)
// Integrates with TouchEventDispatcher for gesture → actor mapping
```

## Feature Gate

`PIXELROOT32_ENABLE_TOUCH` — **default `0`** (`include/platforms/EngineConfig.h:104-106`).

**This flag gates `Engine` ONLY.** It is not a whole-subsystem switch, which is why this skill deliberately carries **no `feature_gate` key in its metadata**: a blanket gate declaration would be misleading, since most of the touch pipeline compiles regardless of the flag.

**Removed when the flag is 0** — all in `Engine`:

| Location | What disappears |
| --- | --- |
| `include/core/Engine.h:22-25` | the three touch includes |
| `include/core/Engine.h:176` / `:184` | `getTouchDispatcher()` |
| `include/core/Engine.h:276-282` | `touchDispatcher`, `touchManager`, `wasTouchActive`, `lastTouchX`, `lastTouchY` |
| `src/core/Engine.cpp` | the per-frame touch polling |

**Still compiled when the flag is 0** — untouched by it:

- `InputManager` embeds a `TouchEventDispatcher` **unconditionally** (`include/input/InputManager.h:18` include, `:152` member) and still exposes `getTouchEvents()` (`:101`) and `hasTouchEvents()` (`:107`).
- `include/input/TouchEventDispatcher.h` contains **no `#if` at all**.
- `TouchManager` compiles too; its guards are **driver/platform selection only** — `#ifdef PLATFORM_NATIVE`, `#if defined(TOUCH_DRIVER_XPT2046)` — never `PIXELROOT32_ENABLE_TOUCH`.

So with the flag off, `Engine`'s touch surface vanishes but `InputManager`'s does not. Which concrete `TouchManager` driver you get is decided by `TOUCH_DRIVER_XPT2046` / `TOUCH_DRIVER_GT911` / `PLATFORM_NATIVE`, entirely independently of this flag.

## Event Pipeline Flow

```
[Hardware] → [TouchAdapter]
                → normalize coordinates
                → return TouchPoint[]
                    → [TouchManager / TouchEventDispatcher]
                        → TouchStateMachine.update()
                            → generate TouchEvent (gestures)
                                → [TouchEventQueue]
                                    → consumer: getEvents()
                                        → [Scene::processTouchEvents]
                                            → [UIManager::processEvents]
                                            → [Scene::onUnconsumedTouchEvent]
```

## Composition Patterns

### Touch input in scene
```
Scene::processTouchEvents(events, count):
  ├── Scene::processTouchEvents(events, count)  // UI first, then unconsumed
  └── // Or manually:
      ├── uiManager.processEvents(events, count)
      └── for each unconsumed: onUnconsumedTouchEvent(event)

Scene::onUnconsumedTouchEvent(event):
  ├── if (event.getType() == Click): handleTap(event.x, event.y)
  ├── if (event.getType() == DragMove): handleDrag(event.x, event.y)
  └── if (event.getType() == LongPress): showContextMenu()
```

### InputManager + Touch combined
```
Engine::runFrame():
  ├── input.update(dt)                    // Update buttons + touch
  ├── TouchEvent events[16];
  ├── uint8_t n = input.getTouchEvents(events, 16);
  ├── sceneManager.update(dt);
  └── scene.processTouchEvents(events, n);
```

## ESP32 Constraints

- **Touch queue size**: Fixed 16-events (`TOUCH_EVENT_QUEUE_SIZE` = 192 bytes). Adequate for most frame loads.
- **TouchPoint buffer**: `CIRCULAR_BUFFER_SIZE` = 8 entries. Manages recent history for gesture smoothing.
- **MAX_TOUCH_IDS**: 5 concurrent touch points (GT911), 1 on XPT2046.
- **No floating point in adapters**: Coordinate normalization uses integer math. No FPU needed for touch pipeline.
- **`TouchEvent` struct**: Compact 12 bytes, naturally aligned. No padding overhead.
- **Compile-time adapter selection**: Use `TOUCH_DRIVER_XPT2046` or `TOUCH_DRIVER_GT911` defines. Only one adapter compiled in.
- **`PIXELROOT32_ENABLE_TOUCH` saves `Engine` bytes only**: default `0`. Turning it off strips `Engine`'s dispatcher, manager, and cached touch state plus the polling code — it does NOT strip `InputManager`'s embedded `TouchEventDispatcher`, which is always linked in.

## Gotchas

1. **TouchEvent size**: `TouchEvent` is 12 bytes (4 + 2 + 2 + 1 + 1 + 1 + 1). The `_padding` byte is explicit — do not rely on implicit padding.
2. **Primary flag**: `TouchEventFlags::Primary` marks the first finger. Subsequent touches do not set this flag.
3. **Consumed flag**: Once set, the event should not be processed by downstream handlers. `UIManager::processEvents` sets this for UI-consumed events.
4. **Drag threshold**: `DRAG_THRESHOLD = 10px` Manhattan distance before drag starts. Tune per-game if needed (hard-coded in `TouchStateMachine`).
5. **Double-click detection**: Uses per-touch-ID last-click tracking (position + timestamp). Only fires if within 400ms and 10px of previous click.
6. **Long press fires once**: `longPressFired` flag prevents repeated long-press events for a single press-and-hold.
7. **Timestamps must be monotonic**: `TouchEventDispatcher` assumes monotonically increasing timestamps per touch ID.
8. **Adapter-specific calibration**: XPT2046 may need rotation/flip mapping (configurable in adapter init). GT911 typically provides pre-calibrated coordinates.
9. **`PIXELROOT32_ENABLE_TOUCH=0` compiles but does not poll**: reading touch through `InputManager::getTouchEvents()` **compiles and links** with the flag off — the dispatcher member is unconditional — but `Engine` is not polling the hardware, so the queue stays empty forever. The symptom is a silent dead control scheme, not a build error. A game that relies on `Engine`'s touch pipeline MUST set `-D PIXELROOT32_ENABLE_TOUCH=1`; the default is `0`.
10. **The flag is not the driver selector**: setting `PIXELROOT32_ENABLE_TOUCH=1` without a `TOUCH_DRIVER_*` define still gives you no hardware. Driver choice is `TOUCH_DRIVER_XPT2046` / `TOUCH_DRIVER_GT911` / `PLATFORM_NATIVE`, decided separately.
11. **Do NOT `#if`-guard `TouchEventDispatcher` or `TouchManager` uses**: neither header is gated by `PIXELROOT32_ENABLE_TOUCH`. Guarding them mirrors `Engine`'s gate onto code that does not have it. Guard only `Engine::getTouchDispatcher()` call sites.

## Common Patterns

### Drag-to-move actor
```cpp
void onUnconsumedTouchEvent(const TouchEvent& event) override {
    if (event.getType() == TouchEventType::DragMove) {
        player->setVelocity(toScalar(event.x - lastX) * 0.5f, 0);
    } else if (event.getType() == TouchEventType::DragEnd) {
        player->setVelocity(0, 0);
    }
    lastX = event.x;
    lastY = event.y;
}
```

### Tap to interact
```cpp
void onUnconsumedTouchEvent(const TouchEvent& event) override {
    if (event.getType() == TouchEventType::Click) {
        // Check if tap is on an interactable area
        checkInteraction(event.x, event.y);
    }
}
```

## Agent Constraints
- **Event Bleeding:** At the start of ANY auto-generated touch handler, YOU MUST insert `if (event.isConsumed()) return;` to prevent events from bleeding through UI into the game world.
- **Build flag:** When generating a game that consumes touch through `Engine`, YOU MUST add `-D PIXELROOT32_ENABLE_TOUCH=1` to `platformio.ini`. The default is `0`, and the code still compiles — it just never receives an event.
- **No false gates:** YOU MUST NOT wrap `TouchEventDispatcher`, `TouchManager`, or `InputManager::getTouchEvents()` in `#if PIXELROOT32_ENABLE_TOUCH`. None of them is gated by it.
