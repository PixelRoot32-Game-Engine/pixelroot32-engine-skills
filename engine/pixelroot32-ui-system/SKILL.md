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
---

## Overview

PixelRoot32's UI system provides entity-based UI elements with touch input support. `UIElement` inherits from `Entity` and integrates with the scene graph. Layouts (`UIAnchorLayout`, `UIGridLayout`, `UIHorizontalLayout`, `UIVerticalLayout`) automate positioning. Touch widgets (`UITouchButton`, `UITouchCheckbox`, `UITouchSlider`) provide interactive controls driven by `UIManager`, which routes touch events, manages hover/pressed/captured states, and maintains consumed-event flags.

## Key APIs

### UIManager (Touch Event Router)

**Header**: `include/graphics/ui/UIManager.h`
**Namespace**: `pixelroot32::graphics::ui`
**Feature gate**: `PIXELROOT32_ENABLE_UI_SYSTEM`

```cpp
UIManager ui;

// Registration (non-owning pointers — widgets must outlive)
ui.addElement(&myButton);
ui.removeElement(myButton);
ui.clear();

// Touch event processing (returns consumed count)
uint8_t consumed = ui.processEvents(events, count);
bool processed = ui.processEvent(event);

// Hover/active tracking
UITouchWidget* active = ui.getActiveWidget();
UITouchWidget* hover = ui.getHoverWidget();
UITouchWidget* captured = ui.getCapturedWidget();
ui.releaseCapture();
ui.updateHover(x, y);
ui.clearConsumeFlags();

// Capacity
uint8_t max = ui.getMaxElements();     // 16
bool full = ui.isFull();
```

### UIElement (Base Widget)

**Header**: `include/graphics/ui/UIElement.h`
**Namespace**: `pixelroot32::graphics::ui`

```cpp
// Inherits from Entity with EntityType::UI_ELEMENT
UIElement element(position, width, height, type);
// Or with x, y:
UIElement element(x, y, w, h, UIElementType::GENERIC);

// Types
enum UIElementType { GENERIC, BUTTON, LABEL, CHECKBOX, LAYOUT };

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

```cpp
UITouchButton button("Start", {100, 100}, {80, 24}, onStartClick);
// Legacy constructor: UITouchButton("Start", x, y, w, h);

// Callbacks
button.setOnDown(callback);
button.setOnUp(callback);
button.setOnClick(callback);  // Or pass to constructor

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

```cpp
UILabel label("Score: 0", {10, 10}, Color::White, 2);

label.setText("Score: 100");
label.setVisible(false);
label.centerX(screenWidth);
```

### UIButton (Physical Input Version)

**Header**: `include/graphics/ui/UIButton.h`
**Namespace**: `pixelroot32::graphics::ui`

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

```cpp
UIGridLayout grid(10, 10, 220, 220);
grid.addElement(&btn1);  // Auto-positions in row-major order
grid.addElement(&btn2);
grid.updateLayout();
```

#### UILayout (Base)

**Header**: `include/graphics/ui/UILayout.h`

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

```cpp
// Hit testing utility for touch events
// Checks if a point is within an element's bounds
// Used internally by UIManager for event routing
```

### UITouchWidget (Abstract Touch Base)

**Header**: `include/graphics/ui/UITouchWidget.h`

```cpp
// Base class for touch-interactive widgets
// Provides processEvent(), draw(), state management
```

### UITouchCheckbox

**Header**: `include/graphics/ui/UITouchCheckbox.h`

```cpp
UITouchCheckbox checkbox("Sound", {10, 50});

checkbox.setChecked(true);
checkbox.isChecked();
checkbox.setOnToggle(callback);  // UIElementBoolCallback(bool)
```

### UITouchSlider

**Header**: `include/graphics/ui/UITouchSlider.h`

```cpp
UITouchSlider slider({10, 100}, {200, 16});

slider.setValue(0.5f);
slider.getValue();
slider.setOnChange(callback);  // void(*)(float)
```

### UIPanel

**Header**: `include/graphics/ui/UIPanel.h`

```cpp
UIPanel panel({50, 50}, {140, 140});
// Container widget with background rendering
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

- **MAX_ELEMENTS**: UIManager holds a fixed array of 16 non-owning pointers. Check `isFull()` before adding.
- **No std::function**: Callbacks use raw C function pointers (`void(*)()`, `void(*)(bool)`, `void(*)(float)`).
- **No dynamic allocation**: Layouts use `std::vector<UIElement*>`, but element objects are owned by the scene. Layouts only reorder pointers.
- **Feature gate**: Entire UI system is gated by `PIXELROOT32_ENABLE_UI_SYSTEM`. All UI headers must be guarded with `#if`.
- **UIManager in Scene**: `UIManager` is a member of `Scene`. Only available when the feature gate is enabled.

## Gotchas

1. **Non-owning pointers**: `UIManager` does NOT delete widgets. Call `removeElement()` BEFORE destroying a widget to prevent dangling pointers. `capturedWidget` is automatically cleared on `removeElement()`.
2. **Consumed flags**: Once `processEvents()` marks an event as consumed, downstream scene code should skip it. Check `event.isConsumed()`.
3. **Layout reflow**: Call `updateLayout()` after adding/removing elements in layout containers. Anchor layouts auto-position but do not auto-scale.
4. **Fixed position**: Widgets with `setFixedPosition(true)` ignore camera scroll. Works even without `UIManager` — it's an `Entity` property.
5. **UI not drawn by UIManager**: `UIManager::draw()` is a deprecated no-op. Widgets must be drawn by their owning scene/entity via `widget.draw(renderer)`.
6. **`UIElementBoolCallback`**: Used by checkbox toggle. Signature: `void(*)(bool checked)`.
7. **Slot-based allocation**: `UIManager` uses slot reuse (`slotInUse[]`). Adding/removing elements does not compact slots — `getElementAt(index)` may return nullptr for freed slots.

## Common Patterns

### Tap button to transition scene
```cpp
UITouchButton startBtn("Start", {80, 140}, {80, 32}, [](){
    // Transition to game scene via global reference
    sceneManager.transitionToScene(&gameScene, TransitionType::Fade, 300);
});
```

### Volume slider
```cpp
UITouchSlider volSlider({40, 180}, {160, 16});
volSlider.setValue(0.75f);
volSlider.setOnChange([](float v) {
    audioEngine.setMasterVolume(v);
});
```
