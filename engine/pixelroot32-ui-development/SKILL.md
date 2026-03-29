---
name: pixelroot32-ui-development
description: Create user interfaces using the layout system with automatic positioning, navigation, and ESP32-optimized rendering
license: MIT
compatibility: opencode>=0.1.0
metadata:
  domain: engine
  language: cpp
  platform: esp32
---

## Overview

Generate UI code using PixelRoot32's layout system with automatic element positioning, D-pad navigation, and embedded-optimized rendering.

## Source of Truth

- `docs/API_REFERENCE.md` - UIElement, layouts, buttons
- `docs/ARCHITECTURE.md` - UI System hierarchy
- `docs/STYLE_GUIDE.md` - UI Layout guidelines

## Modular Compilation

```cpp
#if PIXELROOT32_ENABLE_UI_SYSTEM
// UI code here
#endif

// Disable to save ~4KB RAM, ~20KB Flash
```

## UI Element Hierarchy

```
Entity
└── UIElement
    ├── UILabel
    ├── UIButton
    ├── UICheckbox
    └── UIPanel
        └── UILayout
            ├── UIHorizontalLayout
            ├── UIVerticalLayout
            ├── UIGridLayout
            ├── UIAnchorLayout
            └── UIPaddingContainer
```

## Layout Selection Guide

| Layout | Use Case |
|--------|----------|
| `UIVerticalLayout` | Vertical lists, menus |
| `UIHorizontalLayout` | Horizontal menus, bars |
| `UIGridLayout` | Inventories, level selection |
| `UIAnchorLayout` | HUDs, fixed-position elements |
| `UIPaddingContainer` | Spacing around elements |

## Basic Button

```cpp
#include <graphics/ui/UIButton.h>

pixelroot32::graphics::ui::UIButton playButton{
    "PLAY",
    0,  // index
    pixelroot32::math::Vector2{80, 100},
    pixelroot32::math::Vector2{80, 24},
    []() {
        // Callback - start game
        gameScene.init();
    },
    pixelroot32::graphics::ui::TextAlignment::CENTER,
    2  // fontSize
};
```

## Grid Layout (Inventory)

```cpp
#include <graphics/ui/UIGridLayout.h>

pixelroot32::graphics::ui::UIGridLayout inventory{
    pixelroot32::math::Vector2{20, 20},
    200, 160
};
inventory.setColumns(4);
inventory.setPadding(4);
inventory.setSpacing(4);

for (int i = 0; i < 16; i++) {
    inventory.addElement(&itemButtons[i]);
}
```

## Vertical Menu

```cpp
#include <graphics/ui/UIVerticalLayout.h>

pixelroot32::graphics::ui::UIVerticalLayout menu{
    pixelroot32::math::Vector2{60, 60},
    120, 140
};
menu.setPadding(8);
menu.setSpacing(12);

menu.addElement(&startButton);
menu.addElement(&optionsButton);
menu.addElement(&quitButton);
```

## Panel with Layout

```cpp
#include <graphics/ui/UIPanel.h>

pixelroot32::graphics::ui::UIPanel dialog{
    pixelroot32::math::Vector2{40, 60},
    160, 100
};
dialog.setBackgroundColor(pixelroot32::graphics::Color::DarkBlue);
dialog.setBorderColor(pixelroot32::graphics::Color::White);
dialog.setBorderWidth(2);

dialog.setChild(&verticalLayout);
```

## Anchor Layout (HUD)

```cpp
#include <graphics/ui/UIAnchorLayout.h>

pixelroot32::graphics::ui::UIAnchorLayout hud{
    pixelroot32::math::Vector2{0, 0},
    240, 240
};
hud.addElement(&scoreLabel);
hud.addElementAtAnchor(&scoreLabel, pixelroot32::graphics::ui::Anchor::TopLeft);
hud.addElementAtAnchor(&livesLabel, pixelroot32::graphics::ui::Anchor::TopRight);
```

## Input Handling

```cpp
void MyScene::update(unsigned long dt) {
    menu.handleInput(inputManager);
}

void handleInput(const pixelroot32::input::InputManager& input) {
    if (input.isButtonPressed(pixelroot32::input::Button::BTN_A)) {
        button.press();
    }
    if (input.isButtonPressed(pixelroot32::input::Button::BTN_UP)) {
        layout.selectPrevious();
    }
    if (input.isButtonPressed(pixelroot32::input::Button::BTN_DOWN)) {
        layout.selectNext();
    }
}
```

## Navigation Mapping

```cpp
// Vertical layout: UP/DOWN navigation
// Horizontal layout: LEFT/RIGHT navigation
// Grid layout: 4-direction navigation

gridLayout.setNavigationButtons(
    pixelroot32::input::Button::BTN_UP,    // upButton
    pixelroot32::input::Button::BTN_DOWN,  // downButton
    pixelroot32::input::Button::BTN_LEFT,  // leftButton
    pixelroot32::input::Button::BTN_RIGHT   // rightButton
);
```

## Render Layer

UI elements should use render layer 2 (UI layer):

```cpp
uiElement.setRenderLayer(2);  // UI on top of gameplay
```

## Constraints

- UI system requires `PIXELROOT32_ENABLE_UI_SYSTEM=1`
- Use layouts over manual positioning
- Use anchors for HUD elements
- Layouts use viewport culling for performance
- Memory: ~4KB RAM when enabled
- Elements in layer 2 render above gameplay (layer 1) and background (layer 0)
