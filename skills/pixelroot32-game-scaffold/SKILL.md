---
name: pixelroot32-game-scaffold
description: Create the PlatformIO project scaffold for a new PixelRoot32 game — root platformio.ini, base lib/platformio.ini, platform entry points (native/ESP32), and a minimal Scene. Use when scaffolding, bootstrapping, initializing, starting, or creating a new game project.
license: MIT
compatibility: opencode>=0.1.0
metadata:
  domain: engine
  platform: cross-platform
  engine_version: "1.9.0+unreleased"
---

## Overview

Every PixelRoot32 game is a **self-contained PlatformIO project** (PC via SDL2 `native`, and ESP32-class boards). The scaffold is a fixed set of files, not freeform code. This skill produces that set exactly, so agents stop hallucinating the `platformio.ini`, the platform entry points, and the `Engine`/`DisplayConfig`/`InputConfig` wiring.

## Source of Truth

Derived from the engine's shipped examples (self-contained PlatformIO projects):

- `examples/hello_world/` — the **minimal** reference: engine boots → scene draws → input works. Read this first.
- `examples/snake/` — the audio variant: `Engine(config, inputConfig, audioConfig)` with an `SDL2_AudioBackend`.
- `examples/README.md` — catalogue of every example, its environments, and where each `PIXELROOT32_ENABLE_*` capability is demonstrated.
- Engine headers: `include/graphics/DisplayConfig.h`, `include/input/InputConfig.h`, `include/platforms/EngineConfig.h`, `include/core/Engine.h`.

## Directory Layout

```
<game-name>/
├── platformio.ini          # root: default_envs + per-environment build flags
├── .gitignore
├── README.md               # optional but recommended (mirror examples/README.md)
├── lib/
│   └── platformio.ini      # [base], [base_esp32], [base_native] — shared by all envs
└── src/
    ├── main.cpp            # #ifdef platform selector only — no game logic
    ├── <GameName>Scene.h
    ├── <GameName>Scene.cpp
    └── platforms/
        ├── native.h        # SDL2: DisplayConfig + InputConfig + Engine + main()
        ├── esp32_dev.h     # ST7735/ST7789: config + Engine + setup()/loop()
        └── esp32_s3.h      # ST7735 variant with S3 pins + Core 2.0.14 workaround
```

Use the game name as namespace (`namespace mygame {`) and `PascalCase` scene name (`MyGameScene`). Only `src/*.cpp` files are compiled; `.h` under `src/` are plain includes.

## Key Files

### `lib/platformio.ini` — base config (do NOT deviate)

```ini
[base]
build_unflags =
	-std=gnu++11        ; remove C++11 default, we use C++17
build_flags =
    -Wl,--gc-sections
	-std=gnu++17
    -fno-exceptions     ; engine is compiled -fno-exceptions

[base_esp32]
platform = espressif32
monitor_speed = 115200
build_unflags =
	${base.build_unflags}
build_flags =
    ${base.build_flags}
    -Os
    -flto
    -fuse-linker-plugin
    -ffunction-sections
    -fdata-sections
    -fno-lto
    -fno-rtti

[base_native]
platform = native
build_unflags =
	${base.build_unflags}
build_flags =
    ${base.build_flags}
    -DPLATFORM_NATIVE
    -DSDL_MAIN_HANDLED
    -lSDL2
```

### `platformio.ini` — root (engine dependency is a decision gate)

```ini
[platformio]
default_envs = native
extra_configs = lib/platformio.ini

[env:native]
extends = base_native
lib_deps =
	symlink://<relative-path-to-engine-root>
build_flags =
    ${base_native.build_flags}
    -D PHYSICAL_DISPLAY_WIDTH=240
    -D PHYSICAL_DISPLAY_HEIGHT=240
    -Isrc
    -Iinclude
    -I.pio/libdeps/native/PixelRoot32-Game-Engine/include
    -I.pio/libdeps/native/PixelRoot32-Game-Engine/src
    ; Windows (MSYS2/MinGW) — comment out on macOS (use -I/opt/homebrew/include -L/opt/homebrew/lib):
    -IC:/msys64/mingw64/include
    -LC:/msys64/mingw64/lib
    -O2
    -Wall
    -Wextra
    -mconsole
```

`env:esp32dev` / `env:esp32s3` add `extends = base_esp32`, `board`, `framework = arduino`, the same `symlink://` lib_deps, `-D PLATFORM_ESP32DEV` / `-D PLATFORM_ESP32S3`, the TFT driver flags, pins, and `-Isrc`.

### `src/main.cpp` — platform selector ONLY

```cpp
#ifdef PLATFORM_NATIVE
#include "platforms/native.h"
#elif defined(PLATFORM_ESP32S3)
#include "platforms/esp32_s3.h"
#else
#include "platforms/esp32_dev.h"
#endif
```

### `src/platforms/native.h`

```cpp
#ifdef PLATFORM_NATIVE
#include <SDL2/SDL.h>
#include <drivers/native/SDL2_Drawer.h>
#include <core/Engine.h>
#include <platforms/EngineConfig.h>
#include "MyGameScene.h"

namespace pr32 = pixelroot32;

pr32::graphics::DisplayConfig config(
    pr32::graphics::DisplayType::NONE,
    DISPLAY_ROTATION, PHYSICAL_DISPLAY_WIDTH, PHYSICAL_DISPLAY_HEIGHT,
    LOGICAL_WIDTH, LOGICAL_HEIGHT, X_OFF_SET, Y_OFF_SET);

pr32::input::InputConfig inputConfig(
    SDL_SCANCODE_UP, SDL_SCANCODE_DOWN, SDL_SCANCODE_LEFT, SDL_SCANCODE_RIGHT,
    SDL_SCANCODE_SPACE, SDL_SCANCODE_RETURN);   // 6 buttons: Up, Down, Left, Right, A, B

pr32::core::Engine engine(config, inputConfig);

mygame::MyGameScene scene;

int main(int argc, char* argv[]) {
    (void)argc; (void)argv;
    engine.init();
    engine.setScene(&scene);
    engine.run();
    return 0;
}
#endif
```

### `src/platforms/esp32_dev.h`

```cpp
#ifdef PLATFORM_ESP32DEV
#include <Arduino.h>
#include <drivers/esp32/TFT_eSPI_Drawer.h>
#include <core/Engine.h>
#include <platforms/EngineConfig.h>
#include "MyGameScene.h"

namespace pr32 = pixelroot32;

const int BTN_UP = 32, BTN_DOWN = 27, BTN_LEFT = 33;
const int BTN_RIGHT = 14, BTN_A = 13, BTN_B = 12;

pr32::graphics::DisplayConfig config(
    pr32::graphics::DisplayType::ST7735,
    DISPLAY_ROTATION, PHYSICAL_DISPLAY_WIDTH, PHYSICAL_DISPLAY_HEIGHT,
    LOGICAL_WIDTH, LOGICAL_HEIGHT, X_OFF_SET, Y_OFF_SET);

pr32::input::InputConfig inputConfig(BTN_UP, BTN_DOWN, BTN_LEFT, BTN_RIGHT, BTN_A, BTN_B);

pr32::core::Engine engine(config, inputConfig);

mygame::MyGameScene scene;

void setup() {
    engine.init();
    engine.setScene(&scene);
}
void loop() {
    engine.run();
}
#endif
```

`esp32_s3.h` is identical except pins (`BTN_UP=15, BTN_DOWN=37, BTN_LEFT=36, BTN_RIGHT=35, BTN_A=1, BTN_B=2`) and the guard `#ifdef PLATFORM_ESP32S3`.

### `src/MyGameScene.h` / `.cpp` — minimal scene

```cpp
#pragma once
#include <core/Scene.h>
#include <graphics/Renderer.h>

namespace mygame {

class MyGameScene : public pixelroot32::core::Scene {
public:
    void init() override;
    void update(unsigned long deltaTime) override;
    void draw(pixelroot32::graphics::Renderer& renderer) override;
};

}
```

```cpp
#include "MyGameScene.h"
#include <core/Engine.h>

namespace pr32 = pixelroot32;
extern pr32::core::Engine engine;   // provided by the platform file

namespace mygame {

void MyGameScene::init() {
    Scene::init();                   // ALWAYS call base first (resetState + scheduler init)
    // create entities / UI here; addEntity(ptr)
}

void MyGameScene::update(unsigned long deltaTime) {
    Scene::update(deltaTime);
    // game logic — zero allocation
}

void MyGameScene::draw(pr32::graphics::Renderer& renderer) {
    renderer.drawFilledRectangle(0, 0,
        renderer.getLogicalWidth(), renderer.getLogicalHeight(),
        pr32::graphics::Color::Black);
    Scene::draw(renderer);
}

}
```

## Decision Gates

| Fork | Options | Rule |
|------|---------|------|
| Engine dependency | inside engine repo vs standalone | Inside repo → `lib_deps = symlink://<relative path to engine root>` (examples use `../../` from `examples/<name>`). Standalone → `lib_deps = PixelRoot32-Game-Engine=<git ref or registry tag>`. |
| Platforms | native only / +esp32dev / +esp32s3 | Always include `native`; add ESP32 envs + matching `platforms/*.h` as needed. |
| Audio | with / without | With audio, add `SDL2_AudioBackend` + `Engine(config, inputConfig, audioConfig)` (see `examples/snake/src/platforms/native.h`). Without, `Engine(config, inputConfig)`. |
| Display | ST7735 (128×128) / ST7789 (240×240) / OLED | Pick `DisplayType` + TFT flags + `PHYSICAL_DISPLAY_WIDTH/HEIGHT`. `OLED_SSD1306`/`OLED_SH1106` use U8g2 (`-D PIXELROOT32_USE_U8G2_DRIVER`, I2C pin ctor — see `examples/flappy_bird`). |
| Resolution | physical vs logical | Logical ≤ physical. Set `-D LOGICAL_WIDTH/HEIGHT` to render smaller and let the engine nearest-neighbor scale (memory savings). |
| Feature gates | `PIXELROOT32_ENABLE_*` | Pass as `-D PIXELROOT32_ENABLE_<X>=1` on the env `build_flags`. Consult `examples/README.md` "Where each opt-in capability is demonstrated" before enabling. |

## ESP32 Constraints

- **Resolution macros** come from `include/platforms/EngineConfig.h` with defaults `240×240`: `PHYSICAL_DISPLAY_WIDTH/HEIGHT`, `LOGICAL_WIDTH/HEIGHT` (default = physical), `DISPLAY_ROTATION`, `X_OFF_SET`, `Y_OFF_SET`. Override with `-D` in `build_flags`.
- **TFT flags** (ST7735/ST7789 via TFT_eSPI) are mandatory on ESP32: `-D USER_SETUP_LOADED=1 -D ST7735_DRIVER -D TFT_WIDTH=… -D TFT_HEIGHT=…` plus pins (`TFT_MOSI`, `TFT_SCLK`, `TFT_DC`, `TFT_RST`, `TFT_CS`), `SPI_FREQUENCY`, `SPI_READ_FREQUENCY`.
- **ESP32-S3 freeze workaround**: pin Arduino Core to `2.0.14` via `platform_packages = framework-arduinoespressif32 @ https://github.com/espressif/arduino-esp32#2.0.14`.
- **`-fno-exceptions` + `-fno-rtti`** are on the ESP32 base; game code must not use `try`/`catch`, `throw`, `dynamic_cast`, `typeid`.
- **Zero-allocation game loop**: `Scene::update()`/`draw()` must not `new`/`malloc`/`std::vector`. UI layout trees are the exception — build them once in `init()`/`initUI()`.

## Gotchas

1. **`symlink://` path is relative to the PROJECT dir, not the engine.** `examples/hello_world` uses `symlink://../../` because it sits at `examples/<name>/` inside the engine repo. Compute the path from your project folder to the engine root; a wrong path fails at `pio run` with "library not found".
2. **`-I.pio/libdeps/native/PixelRoot32-Game-Engine/include` (and `/src`)** assume the dependency resolves to a folder named `PixelRoot32-Game-Engine`. True for the symlink and registry; a Git-branch `lib_deps` whose `library.json` names the lib differently breaks these `-I` lines — adjust the name to match.
3. **`DisplayConfig` field order is positional and fixed**: `(DisplayType, rotation, physicalW, physicalH, logicalW, logicalH, xOffset, yOffset, customSurface=nullptr)`. `logW==0` means "same as physical". `DisplayType::CUSTOM` needs a `DrawSurface*`.
4. **`DisplayType` enum is unscoped** (`ST7789, ST7735, ILI9341, ILI9341_2, OLED_SSD1306, OLED_SH1106, NONE, CUSTOM`) in `pixelroot32::graphics`. Native/SDL uses `NONE`.
5. **`InputConfig` is variadic** (max 16); native passes SDL scancodes, ESP32 passes GPIO pins, in the same 6-button order: Up, Down, Left, Right, A, B.
6. **`Engine` lives in the platform file, not the scene.** The scene uses `extern pixelroot32::core::Engine engine;`. Do NOT create a second `Engine` in the scene or `main.cpp`.
7. **Native entry is `main()`; ESP32 entry is `setup()`/`loop()`.** `engine.setScene(&scene)` before `engine.run()`. `Engine` does not own the scene — keep the scene alive for the whole run.
8. **`Scene::init()` is idempotent and must be called first** in every override (it runs `resetState()`). Release owned resources in a `resetState()` override BEFORE calling `Scene::resetState()`.
9. **Windows SDL2**: `native` needs the MSYS2/MinGW include/lib paths (`-IC:/msys64/mingw64/include -LC:/msys64/mingw64/lib`); macOS uses Homebrew paths instead. Keep both, one commented, per the examples.

## Common Patterns

### Add a second scene (menu → game)

```cpp
// native.h — two scenes, engine owns neither
mygame::MenuScene menuScene;
mygame::GameScene gameScene;

int main(int argc, char* argv[]) {
    (void)argc; (void)argv;
    engine.init();
    engine.setScene(&menuScene);
    engine.run();
    return 0;
}
```

Scene switching/transitions belong to `pixelroot32-scene-manager`.

### Optional feature flag in scene

```cpp
#include <platforms/EngineConfig.h>
#if PIXELROOT32_ENABLE_PHYSICS
    physicsScheduler.update(dt, collisionSystem);
#endif
```

Route subsystem code generation through the canonical routing table in `pixelroot32-cpp-code-generation`.

## Agent Constraints

- DO NOT invent a different file layout: the six files above (`platformio.ini`, `lib/platformio.ini`, `src/main.cpp`, `src/platforms/{native,esp32_dev,esp32_s3}.h`, scene `.h`/`.cpp`) are the contract.
- DO NOT change the base build flags (`-std=gnu++17`, `-fno-exceptions`, ESP32 `-fno-rtti`/`-flto`, native `-DPLATFORM_NATIVE -DSDL_MAIN_HANDLED -lSDL2`).
- DO NOT define `Engine`/`DisplayConfig`/`InputConfig` in the scene or `main.cpp` — they belong to the platform files.
- DO NOT add `new`/`malloc`/`std::vector` to `update()`/`draw()`.
- DO NOT omit `Scene::init()` at the top of the scene's `init()`.
- Verify every `-D PIXELROOT32_ENABLE_*` flag against `examples/README.md` before enabling; flags you guess silently no-op or bloat Flash.
