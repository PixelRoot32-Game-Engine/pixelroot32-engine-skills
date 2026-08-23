---
name: pixelroot32-ui-system
description: Touch UI system with layout containers (Anchor, Grid, Horizontal, Vertical), widgets (Button, Checkbox, Label, Panel, Slider), hit testing, touch event routing via UIManager, and fixed-position HUD support. Use when implementing menus, HUDs, touch controls, or in-game UI panels.
license: MIT
compatibility: opencode>=0.1.0
metadata:
  domain: engine
  subsystem: graphics
  module: ui
  feature_gate: PIXELROOT32_ENABLE_UI_SYSTEM
  platform: cross-platform
  engine_version: "1.9.0+unreleased"
---

## Overview

PixelRoot32's UI system provides entity-based UI elements with touch input support. `UIElement` inherits from `Entity` and integrates with the scene graph. Layouts (`UIAnchorLayout`, `UIGridLayout`, `UIHorizontalLayout`, `UIVerticalLayout`) automate positioning. Touch widgets (`UITouchButton`, `UITouchCheckbox`, `UITouchSlider`) provide interactive controls driven by `UIManager`, which routes touch events, manages hover/pressed/captured states, and maintains consumed-event flags. Sprite widgets (`UISprite`, `UISpriteRow`) added in 1.9.0 render flash-resident sprite art as UI, backed by the non-owning `UISpriteRef` handle.

**See also**: `pixelroot32-touch-input` for the event pipeline feeding `UIManager::processEvents`; `pixelroot32-gameplay-framework` for the game state a HUD (`UILabel`, `UISpriteRow`) renders; `pixelroot32-projection` for converting world coordinates to the screen space that fixed-position UI lives in.

## Key APIs

### UIManager (Touch Event Router)

**Header**: `include/graphics/ui/UIManager.h`
**Namespace**: `pixelroot32::graphics::ui`
**Feature gate**: `PIXELROOT32_ENABLE_UI_SYSTEM`

**Two pointer types, not one.** Registry/storage is `UITouchElement*`; the three cached-widget accessors and one `removeElement` overload are `UITouchWidget*`. Match the declaration exactly.

```cpp
UIManager ui;

// Storage (non-owning pointers — widgets must outlive the manager)
static constexpr uint8_t MAX_ELEMENTS = 16;      // :63
UITouchElement* elementPointers[MAX_ELEMENTS];   // :66

// Registration
bool addElement(UITouchElement* element);
bool removeElement(uint8_t id);
bool removeElement(UITouchWidget* widget);       // UITouchWidget, NOT UITouchElement
void clear();

// Element access
UITouchElement* getElement(uint8_t id) const;
UITouchElement* getElementAt(uint8_t index) const;
UITouchElement** getElements();
UITouchElement* const* getElements() const;
uint8_t getElementCount() const;

// Touch event processing (returns consumed count)
uint8_t processEvents(pixelroot32::input::TouchEvent* events, uint8_t count);
bool    processEvent(pixelroot32::input::TouchEvent& event);   // non-const reference

// Hover/active tracking — these four return UITouchWidget*
UITouchWidget* getActiveWidget() const;
UITouchWidget* getHoverWidget() const;
UITouchWidget* getCapturedWidget() const;
void releaseCapture();
void updateHover(int16_t x, int16_t y);
void clearConsumeFlags();

// Capacity
uint8_t getMaxElements() const;   // 16
bool    isFull() const;

// DEPRECATED no-ops — both bodies do nothing but (void) their argument
void update(unsigned long deltaTime);                      // :115 @deprecated
void draw(pixelroot32::graphics::Renderer& renderer);      // :116 @deprecated
```

### UIElement (Base Widget)

**Header**: `include/graphics/ui/UIElement.h`
**Namespace**: `pixelroot32::graphics::ui`
**Feature gate**: `PIXELROOT32_ENABLE_UI_SYSTEM`

```cpp
// Inherits from Entity with EntityType::UI_ELEMENT
UIElement element(position, width, height, type);
// Or with x, y:
UIElement element(x, y, w, h, UIElementType::GENERIC);

// Types
enum UIElementType { GENERIC, BUTTON, LABEL, CHECKBOX, LAYOUT };

// Callback typedefs — raw function pointers, no std::function (memory efficiency)
using UIElementVoidCallback = void(*)();
using UIElementBoolCallback = void(*)(bool);

// Fixed position (HUD — ignores camera scroll)
element.setFixedPosition(true);
element.isFixedPosition();

// Focus
bool focusable = element.isFocusable();  // Override in subclasses

// Preferred size (for layouts)
element.getPreferredSize(prefWidth, prefHeight);
```

### UITouchButton

**Header**: `include/graphics/ui/UITouchButton.h`
**Namespace**: `pixelroot32::graphics::ui`
**Feature gate**: `PIXELROOT32_ENABLE_UI_SYSTEM`

```cpp
// UITouchButton(std::string_view t, math::Vector2 position, math::Vector2 size,
//               UIElementVoidCallback callback = nullptr,
//               TextAlignment textAlign = TextAlignment::CENTER, int fontSize = 2);
UITouchButton button("Start", {100, 100}, {80, 24}, onStartClick);

// Legacy constructor — takes NO callback:
//   UITouchButton(std::string_view t, int16_t x, int16_t y, uint16_t w, uint16_t h);

// Callbacks — raw function pointers (UIElementVoidCallback = void(*)())
button.setOnDown(callback);
button.setOnUp(callback);
button.setOnClick(callback);  // void setOnClick(UIElementVoidCallback callback);

// Styling
button.setColors(normalColor, pressedColor, disabledColor);
button.setFontSize(2);
button.setTextAlignment(TextAlignment::CENTER);

// Label
button.setLabel("Play");
std::string_view label = button.getLabel();

// Auto-sizing
button.autoSize(4);  // pad around text

// State
button.reset();
```

### UILabel

**Header**: `include/graphics/ui/UILabel.h`
**Namespace**: `pixelroot32::graphics::ui`
**Feature gate**: `PIXELROOT32_ENABLE_UI_SYSTEM`

```cpp
UILabel label("Score: 0", {10, 10}, Color::White, 2);

label.setText("Score: 100");
label.setVisible(false);
label.centerX(screenWidth);
```

### UIButton (Physical Input Version)

**Header**: `include/graphics/ui/UIButton.h`
**Namespace**: `pixelroot32::graphics::ui`
**Feature gate**: `PIXELROOT32_ENABLE_UI_SYSTEM`

```cpp
UIButton btn("Start", 0, {100, 100}, {80, 24}, onPress);
btn.handleInput(inputManager);

// Selection (for D-pad navigation)
btn.setSelected(true);
btn.getSelected();

// Styling
btn.setStyle(Color::White, Color::Blue, true);
```

### Layout System

#### UIAnchorLayout (Fixed Positions)

**Header**: `include/graphics/ui/UIAnchorLayout.h`
**Namespace**: `pixelroot32::graphics::ui`
**Feature gate**: `PIXELROOT32_ENABLE_UI_SYSTEM`

```cpp
UIAnchorLayout layout(0, 0, screenW, screenH);

// Elements positioned at anchors
layout.addElement(&scoreLabel, Anchor::TOP_LEFT);
layout.addElement(&pauseBtn, Anchor::TOP_RIGHT);
layout.addElement(&healthBar, Anchor::BOTTOM_LEFT);
layout.addElement(&dialogPanel, Anchor::CENTER);

layout.setScreenSize(240, 240);
layout.updateLayout();  // Recalculate positions
```

Anchors: `TOP_LEFT`, `TOP_RIGHT`, `BOTTOM_LEFT`, `BOTTOM_RIGHT`, `CENTER`, `TOP_CENTER`, `BOTTOM_CENTER`, `LEFT_CENTER`, `RIGHT_CENTER`.

#### UIGridLayout (Matrix)

**Header**: `include/graphics/ui/UIGridLayout.h`
**Namespace**: `pixelroot32::graphics::ui`
**Feature gate**: `PIXELROOT32_ENABLE_UI_SYSTEM`

```cpp
UIGridLayout grid(10, 10, 220, 220);
grid.addElement(&btn1);  // Auto-positions in row-major order
grid.addElement(&btn2);
grid.updateLayout();
```

#### UIVerticalLayout (Stack Top→Bottom)

**Header**: `include/graphics/ui/UIVerticalLayout.h`
**Namespace**: `pixelroot32::graphics::ui`
**Feature gate**: `PIXELROOT32_ENABLE_UI_SYSTEM`

```cpp
// class UIVerticalLayout : public UILayout — two constructor forms:
UIVerticalLayout(math::Scalar x, math::Scalar y, int w, int h);
UIVerticalLayout(math::Vector2 position, int w, int h);

UIVerticalLayout column(toScalar(10), toScalar(10), 100, 200);
column.addElement(&btn1);
column.addElement(&btn2);
column.updateLayout();
```

#### UIHorizontalLayout (Row Left→Right)

**Header**: `include/graphics/ui/UIHorizontalLayout.h`
**Namespace**: `pixelroot32::graphics::ui`
**Feature gate**: `PIXELROOT32_ENABLE_UI_SYSTEM`

```cpp
// class UIHorizontalLayout : public UILayout — two constructor forms:
UIHorizontalLayout(math::Scalar x, math::Scalar y, int w, int h);
UIHorizontalLayout(math::Vector2 position, int w, int h);

UIHorizontalLayout row({10, 220}, 220, 24);
row.addElement(&leftBtn);
row.addElement(&rightBtn);
row.updateLayout();
```

`UIGridLayout` and `UIAnchorLayout` expose the same two-form constructor pattern (`Scalar x, Scalar y, int w, int h` and `Vector2 position, int w, int h`).

#### UIPaddingContainer (Single-Child Wrapper)

**Header**: `include/graphics/ui/UIPaddingContainer.h`
**Namespace**: `pixelroot32::graphics::ui`
**Feature gate**: `PIXELROOT32_ENABLE_UI_SYSTEM`

**Not a layout.** `UIPaddingContainer` derives from `UIElement`, not from `UILayout`, and holds exactly one child: `UIElement* child = nullptr;`. It has no element vector and no `addElement()` — do not treat it as a multi-child container.

#### UILayout (Base)

**Header**: `include/graphics/ui/UILayout.h`
**Namespace**: `pixelroot32::graphics::ui`
**Feature gate**: `PIXELROOT32_ENABLE_UI_SYSTEM` — the whole header is wrapped in `#if PIXELROOT32_ENABLE_UI_SYSTEM`

Storage is a `std::vector<UIElement*> elements;` (protected). `UIAnchorLayout` adds a second vector, `std::vector<std::pair<UIElement*, Anchor>> anchoredElements;`. There is **no max-children constant on layouts** — child storage is heap-backed and unbounded.

```cpp
// Shared properties for all layouts
layout.setPadding(toScalar(4));
layout.setSpacing(toScalar(8));
layout.setScrollingEnabled(true);

// Scroll behavior
enum ScrollBehavior { NONE, SCROLL, CLAMP };

// Element access
layout.getElement(index);
layout.getElementCount();
layout.clearElements();
```

### UIHitTest

**Header**: `include/graphics/ui/UIHitTest.h`
**Namespace**: `pixelroot32::graphics::ui`
**Feature gate**: `PIXELROOT32_ENABLE_UI_SYSTEM`

```cpp
// Hit testing utility for touch events
// Checks if a point is within an element's bounds
// Used internally by UIManager for event routing
```

### UITouchWidget (Abstract Touch Base)

**Header**: `include/graphics/ui/UITouchWidget.h`
**Namespace**: `pixelroot32::graphics::ui`
**Feature gate**: `PIXELROOT32_ENABLE_UI_SYSTEM`

```cpp
// Base class for touch-interactive widgets
// Provides processEvent(), draw(), state management
```

### UITouchCheckbox

**Header**: `include/graphics/ui/UITouchCheckbox.h`
**Namespace**: `pixelroot32::graphics::ui`
**Feature gate**: `PIXELROOT32_ENABLE_UI_SYSTEM`

```cpp
UITouchCheckbox checkbox("Sound", {10, 50});

checkbox.setChecked(true);
checkbox.isChecked();

// void setOnChanged(UIElementBoolCallback callback);  — there is NO setOnToggle()
checkbox.setOnChanged(onSoundChanged);   // void(*)(bool)
```

The constructor's last parameter is `UIElementBoolCallback callback = nullptr`, so the callback can also be passed at construction.

### UITouchSlider

**Header**: `include/graphics/ui/UITouchSlider.h`
**Namespace**: `pixelroot32::graphics::ui`
**Feature gate**: `PIXELROOT32_ENABLE_UI_SYSTEM`

`class UITouchSlider : public UITouchElement`. **Value domain is 0–100, not 0–255** (class doc: "Value range: 0-100"). Pixel geometry is `int16_t`/`uint16_t` — **not** `Scalar`.

```cpp
static constexpr uint8_t MIN_VALUE = 0;      // :55
static constexpr uint8_t MAX_VALUE = 100;    // :56
using SliderCallback = void(*)(uint8_t);     // :37 — uint8_t, NOT float

// Only constructor (no Vector2 form):
explicit UITouchSlider(int16_t x, int16_t y, uint16_t w, uint16_t h, uint8_t initialValue);  // :67

UITouchSlider slider(10, 100, 200, 16, 75);   // initialValue in [MIN_VALUE, MAX_VALUE]

// Callbacks — void setOnValueChanged(SliderCallback);  there is NO setOnChange()
void setOnValueChanged(SliderCallback callback);   // :80  void(*)(uint8_t)
void setOnDragStart(UIElementVoidCallback);        // :86
void setOnDragEnd(UIElementVoidCallback);          // :92

// Value access
uint8_t getValue() const;            // :134
void    setValue(uint8_t newValue);  // :140
uint8_t getPreviousValue() const;    // :146
bool    hasValueChanged() const;     // :152
```

### UIPanel

**Header**: `include/graphics/ui/UIPanel.h`
**Namespace**: `pixelroot32::graphics::ui`
**Feature gate**: `PIXELROOT32_ENABLE_UI_SYSTEM`

```cpp
UIPanel panel({50, 50}, {140, 140});
// Container widget with background rendering
```

### UISpriteRef (new in 1.9.0)

**Header**: `include/graphics/ui/UISpriteRef.h`
**Namespace**: `pixelroot32::graphics::ui`
**Feature gate**: `PIXELROOT32_ENABLE_UI_SYSTEM` — **no stub classes**; with the flag off these declarations simply vanish.

A plain aggregate `struct`, **no base class**. It is a non-owning handle to one sprite in one of three formats.

```cpp
enum class UISpriteFormat : uint8_t { None = 0, Mono, Bpp2, Bpp4 };   // :41-46

struct UISpriteRef {                                                  // :53-65
    union Storage { const Sprite* mono; const Sprite2bpp* bpp2; const Sprite4bpp* bpp4; };
    Storage storage{nullptr};
    UISpriteFormat format = UISpriteFormat::None;
    Color tint = Color::White;      // Mono only; ignored otherwise
    uint8_t paletteSlot = 0;        // 2bpp/4bpp only; ignored otherwise
};

// Factories — one per format
inline UISpriteRef makeUISpriteRef(const Sprite& sprite, Color tint = Color::White);      // :68
inline UISpriteRef makeUISpriteRef(const Sprite2bpp& sprite, uint8_t paletteSlot = 0);    // :77
inline UISpriteRef makeUISpriteRef(const Sprite4bpp& sprite, uint8_t paletteSlot = 0);    // :86

// Free functions
bool uiSpriteRefIsSet(const UISpriteRef& ref);    // :100
int  uiSpriteRefWidth(const UISpriteRef& ref);    // :103
int  uiSpriteRefHeight(const UISpriteRef& ref);   // :106
void drawUISpriteRef(Renderer& renderer, const UISpriteRef& ref,
                     int x, int y, bool flipX);   // :114 — flipX has NO default
```

**Ownership** (`:31-32`): a `UISpriteRef` **never owns** the sprite. Sprites are `constexpr`/flash-resident asset data that outlives any UI element pointing at it.

### UISprite (new in 1.9.0)

**Header**: `include/graphics/ui/UISprite.h`
**Namespace**: `pixelroot32::graphics::ui`
**Feature gate**: `PIXELROOT32_ENABLE_UI_SYSTEM` — no stub class.

`class UISprite : public UIElement` (:41). Position is `Scalar`/`Vector2` (an `Entity`-derived element), unlike `UITouchSlider`.

```cpp
explicit UISprite(pixelroot32::math::Vector2 position);                 // :49
UISprite(pixelroot32::math::Scalar x, pixelroot32::math::Scalar y);     // :54

// One setter per format — sizes the element to the sprite
void setSprite(const Sprite& sprite, Color tint = Color::White);        // :63
void setSprite(const Sprite2bpp& sprite, uint8_t paletteSlot = 0);      // :70
void setSprite(const Sprite4bpp& sprite, uint8_t paletteSlot = 0);      // :77
void clearSprite();                                                     // :85 — returns the element to 0x0

// Queries
UISpriteFormat getFormat() const;      // :88
bool    hasSprite() const;             // :91
Color   getTint() const;               // :94
uint8_t getPaletteSlot() const;        // :97

// Horizontal flip — a property of the ELEMENT, survives a later setSprite()
void setFlipX(bool flip);              // :105
bool getFlipX() const;                 // :108

void update(unsigned long deltaTime) override;   // :114 — documented no-op
void draw(Renderer& renderer) override;          // :120
```

### UISpriteRow (new in 1.9.0)

**Header**: `include/graphics/ui/UISpriteRow.h`
**Namespace**: `pixelroot32::graphics::ui`
**Feature gate**: `PIXELROOT32_ENABLE_UI_SYSTEM` — no stub class.

`class UISpriteRow : public UIElement` (:59). This is the heart/ammo/lives row: `unitsPerIcon` lets one icon show partial states (e.g. half-hearts), so `value` counts **units**, not icons.

```cpp
static constexpr int kMaxStates = 5;                                    // :67

explicit UISpriteRow(pixelroot32::math::Vector2 position);              // :75
UISpriteRow(pixelroot32::math::Scalar x, pixelroot32::math::Scalar y);  // :80

// Per-state artwork — stateIndex out of range is IGNORED, not clamped (:86-88)
void setStateSprite(int stateIndex, const Sprite& sprite, Color tint = Color::White);    // :92
void setStateSprite(int stateIndex, const Sprite2bpp& sprite, uint8_t paletteSlot = 0);  // :100
void setStateSprite(int stateIndex, const Sprite4bpp& sprite, uint8_t paletteSlot = 0);  // :108

void    setUnitsPerIcon(uint8_t units);  uint8_t getUnitsPerIcon() const;  // :114 / :117 — CLAMPS to [1, kMaxStates - 1]
void    setCapacity(uint8_t iconCount);  uint8_t getCapacity() const;      // :124 / :127
void    setValue(int filledUnits);       int     getValue() const;         // :136 / :139 — int, UNCLAMPED
void    setSpacing(int pixels);          int     getSpacing() const;       // :142 / :145
void    setRowSpacing(int pixels);       int     getRowSpacing() const;    // :148 / :151
void    setIconsPerRow(uint8_t n);       uint8_t getIconsPerRow() const;   // :157 / :160 — 0 DISABLES wrapping

// Geometry helpers — both return/emit 0 for an index outside [0, capacity)
int  stateIndexAt(int iconIndex) const;                                 // :171
void iconOffsetAt(int iconIndex, int& outOffsetX, int& outOffsetY) const;  // :182
```

## Layout System Hierarchy

```
UIAnchorLayout (screen-relative anchors)
  ├── UITouchButton (TOP_LEFT)
  ├── UILabel (TOP_CENTER)
  ├── UIGridLayout (CENTER)
  │     ├── UITouchButton
  │     ├── UITouchButton
  │     └── UITouchCheckbox
  └── UIPanel (BOTTOM_RIGHT)
        ├── UILabel
        └── UITouchSlider
```

## Event Routing Pipeline

```
TouchEvent from hardware
  └── UIManager::processEvents()
        ├── For each event:
        │     ├── Update hover state (updateHover)
        │     ├── Hit test against registered elements
        │     ├── Route to matched UITouchWidget::processEvent()
        │     └── If consumed: set Consumed flag
        └── Return consumed count
```

## Composition Patterns

### Scene with full UI
```
class MenuScene : public Scene {
    UITouchButton playBtn, settingsBtn;
    UILabel title;

    void initUI() override {
        title = UILabel("PIXELROOT32", {0, 20}, Color::White, 3);
        title.centerX(240);
        playBtn = UITouchButton("Play", {80, 100}, {80, 24}, onPlay);
        settingsBtn = UITouchButton("Settings", {80, 130}, {80, 24}, onSettings);

        getUIManager().addElement(&playBtn);
        getUIManager().addElement(&settingsBtn);
    }
};
```

### HUD with fixed-position elements
```
class GameHUD {
    UILabel scoreLabel, hpLabel;
    UITouchButton pauseBtn;

    void setup(UIManager& ui) {
        scoreLabel = UILabel("Score: 0", {5, 5}, Color::White, 1);
        hpLabel = UILabel("HP: 3", {5, 20}, Color::Red, 1);
        pauseBtn = UITouchButton("II", {210, 5}, {24, 24}, onPause);

        ui.addElement(&pauseBtn);
        // scoreLabel and hpLabel are not touchable — no UI registration needed
    }
};
```

## ESP32 Constraints

- **MAX_ELEMENTS**: `UIManager` IS bounded — `static constexpr uint8_t MAX_ELEMENTS = 16;` backing `UITouchElement* elementPointers[MAX_ELEMENTS]` and `bool slotInUse[MAX_ELEMENTS]`. Check `isFull()` before adding.
- **No std::function**: Callbacks use raw C function pointers (`void(*)()`, `void(*)(bool)`, `void(*)(uint8_t)` for sliders).
- **Layouts are the documented exception to the zero-allocation rule**: `UILayout` stores children in `std::vector<UIElement*>` (and `UIAnchorLayout` in a second `std::vector<std::pair<UIElement*, Anchor>>`). That storage is heap-backed and **unbounded** — there is no max-children constant. Build the whole layout tree once during `Scene::init()`/`initUI()`, and NEVER add or remove layout children inside `update()`/`draw()`. The element objects themselves are still scene-owned; the vector only holds pointers.
- **Sprite widgets are allocation-free**: `UISpriteRef` is a POD aggregate holding a pointer union — it copies by value and never allocates. `UISpriteRow` is bounded by `static constexpr int kMaxStates = 5;` state slots, so its art table is fixed-size. Sprite pixel data stays in flash (`constexpr`), never copied to RAM.
- **Feature gate**: Entire UI system is gated by `PIXELROOT32_ENABLE_UI_SYSTEM`. All UI headers must be guarded with `#if`. The 1.9.0 sprite widgets declare **no stub classes** — with the flag off the declarations vanish entirely, so guard call sites too.
- **UIManager in Scene**: `UIManager` is a member of `Scene`. Only available when the feature gate is enabled.

## Gotchas

1. **Non-owning pointers**: `UIManager` does NOT delete widgets. Call `removeElement()` BEFORE destroying a widget to prevent dangling pointers. `capturedWidget` is automatically cleared on `removeElement()`.
2. **Consumed flags**: Once `processEvents()` marks an event as consumed, downstream scene code should skip it. Check `event.isConsumed()`.
3. **Layout reflow**: Call `updateLayout()` after adding/removing elements in layout containers. Anchor layouts auto-position but do not auto-scale.
4. **Fixed position**: Widgets with `setFixedPosition(true)` ignore camera scroll. Works even without `UIManager` — it's an `Entity` property.
5. **UIManager neither updates nor draws**: `UIManager::draw()` AND `UIManager::update()` are both deprecated no-ops — their bodies only `(void)` the argument. Widgets must be updated and drawn by their owning scene/entity via `widget.update(dt)` / `widget.draw(renderer)`. Calling `ui.update(dt)` and expecting widget ticks is a silent no-op.
6. **`UIElementBoolCallback`**: Used by `UITouchCheckbox::setOnChanged()`. Signature: `void(*)(bool checked)`.
7. **Slot-based allocation**: `UIManager` uses slot reuse (`slotInUse[]`). Adding/removing elements does not compact slots — `getElementAt(index)` may return nullptr for freed slots.
8. **Capturing lambdas do NOT convert**: every callback type is a raw function pointer (`UIElementVoidCallback`, `UIElementBoolCallback`, `SliderCallback`). A lambda **with a capture list** will not compile. Pass a free function, a `static` member function, or a **captureless** lambda (`[](){ ... }`, `[](bool b){ ... }`, `[](uint8_t v){ ... }`) that reaches state through globals or statics.
9. **Slider domain is 0–100, NOT 0–255**: `SliderCallback` is `void(*)(uint8_t)` and the constructor takes `uint8_t initialValue`, but the range is bounded by `MIN_VALUE = 0` / `MAX_VALUE = 100`. There is no float/normalized domain — scale yourself, and divide by `UITouchSlider::MAX_VALUE` (100), **never by 255**. Dividing by `255.0f` silently caps the control at ~39% of the target range.
10. **Real setter names**: `UITouchCheckbox::setOnChanged` (NOT `setOnToggle`) and `UITouchSlider::setOnValueChanged` (NOT `setOnChange`). Those two misspellings do not exist in the engine.
11. **Slider geometry is not `Scalar`**: `UITouchSlider` takes `int16_t x, y` and `uint16_t w, h` — raw pixels. Do NOT wrap them in `toScalar()`. `UISprite`/`UISpriteRow` are the opposite: they derive from `UIElement` and take `Scalar`/`Vector2`.
12. **`UISpriteRow::setStateSprite` out of range is IGNORED, not clamped**: a `stateIndex` outside `[0, kMaxStates)` silently does nothing — the icon keeps whatever art it had. No error, no assert.
13. **`setUnitsPerIcon` clamps, `setValue` does not**: `setUnitsPerIcon` clamps its argument into `[1, kMaxStates - 1]`. `setValue(int)` is **unclamped** — below 0 renders fully empty, above `capacity * unitsPerIcon` renders fully full. Clamp game state before passing it in if you need to read the value back.
14. **`setIconsPerRow(0)` disables wrapping**: zero is not "no icons" and not an error — it means a single unwrapped row. `stateIndexAt` returns `0` for an index outside `[0, capacity)`, and `iconOffsetAt` sets **both** outputs to `0` for the same — an out-of-range query looks exactly like icon 0.
15. **`UISprite::setFlipX` survives `setSprite()`**: the flip is a property of the element (a left-facing slot), not of the sprite currently in it. Swapping art does not reset it. `clearSprite()` returns the element to 0x0 but the flip flag persists.
16. **`drawUISpriteRef`'s `flipX` has no default**: all five arguments are required — `drawUISpriteRef(renderer, ref, x, y, false)`. Omitting the last one will not compile.
17. **A `UISpriteRef` never owns its sprite**: it is a raw pointer union into `constexpr`/flash-resident asset data. Never point one at a stack temporary or a heap sprite you might free — the asset must outlive every element referencing it.

## Common Patterns

### Tap button to transition scene
```cpp
// Captureless lambda — converts to UIElementVoidCallback. A capturing lambda would NOT.
UITouchButton startBtn("Start", {80, 140}, {80, 32}, [](){
    // Transition to game scene via global reference
    sceneManager.transitionToScene(&gameScene, TransitionType::Fade, 300);
});
```

### Volume slider
```cpp
// x, y, w, h, initialValue — all integral pixels; value domain is 0-100
UITouchSlider volSlider(40, 180, 160, 16, 75);

// SliderCallback = void(*)(uint8_t). Free/static function or captureless lambda only.
volSlider.setOnValueChanged([](uint8_t v) {
    // Divide by MAX_VALUE (100) — NOT 255.
    audioEngine.setMasterVolume(static_cast<float>(v) / UITouchSlider::MAX_VALUE);
});

// Read back / drive programmatically
uint8_t current = volSlider.getValue();
volSlider.setValue(50);
if (volSlider.hasValueChanged()) {
    uint8_t delta = current - volSlider.getPreviousValue();
}
```

### Half-heart health row
```cpp
// 3 hearts, each icon showing 2 units (empty / half / full) => value is in HALVES.
UISpriteRow hearts({4, 4});
hearts.setStateSprite(0, heartEmptySprite);   // state 0 = 0 units filled
hearts.setStateSprite(1, heartHalfSprite);    // state 1 = 1 unit
hearts.setStateSprite(2, heartFullSprite);    // state 2 = 2 units
hearts.setUnitsPerIcon(2);                    // clamped into [1, kMaxStates - 1]
hearts.setCapacity(3);                        // 3 icons => max value 6
hearts.setSpacing(2);
hearts.setIconsPerRow(0);                     // 0 = no wrapping, single row

// setValue is UNCLAMPED — clamp game state yourself
int units = player.healthHalves;
if (units < 0) units = 0;
if (units > 3 * 2) units = 3 * 2;
hearts.setValue(units);

hearts.draw(renderer);   // the scene draws it; UIManager::draw() is a no-op
```

## Agent Constraints
- **Consumption:** YOU MUST call `event.consume()` when handling a touch event in a UI widget.
- **Ownership:** YOU MUST call `ui.removeElement()` BEFORE destroying a local widget to prevent dangling pointers.
- **Slider range:** YOU MUST treat `UITouchSlider` values as 0–100 (`MIN_VALUE`/`MAX_VALUE`). NEVER scale a slider value by 255.
- **Sprite lifetime:** YOU MUST back every `UISpriteRef` / `UISprite` / `UISpriteRow` with `constexpr` flash-resident sprite data. NEVER point one at a local or freed sprite — the ref does not own it.
- **Unclamped value:** YOU MUST clamp the argument to `UISpriteRow::setValue()` yourself; the engine does not.
