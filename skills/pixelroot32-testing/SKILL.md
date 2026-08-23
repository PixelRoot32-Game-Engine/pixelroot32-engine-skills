---
name: pixelroot32-testing
description: Write unit and integration tests using Unity framework with mocks, coverage analysis, and CI integration. Use when adding or running engine tests, selecting a PlatformIO test environment, or debugging a suite that passes without asserting anything.
license: MIT
compatibility: opencode>=0.1.0
metadata:
  domain: tooling
  language: cpp
  platform: cross-platform
  engine_version: "1.9.0+unreleased"
---

## Overview

Generate tests using Unity framework with PlatformIO, mock implementations, and coverage analysis.

## Source of Truth

- `docs/TESTING_GUIDE.md` - Testing practices
- `test/test_config.h` - Shared utilities and macros
- `test/mocks/` - Mock implementations

## Running Tests

`platformio.ini` defines **exactly two buildable test environments**: `[env:native_test]` and
`[env:native_test_gameplay]`. Every other section (`[base]`, `[base_esp32]`, `[base_native]`,
`[profile_*]`, `[native_*]`, `[esp32_*]`) is a **template without the `env:` prefix and cannot be
passed to `-e`**.

```bash
# Run the flags-off suite set (default contract)
pio test -e native_test

# Run the gameplay/projection capability suites (flags ON) — see "Flag-Gated Suites"
pio test -e native_test_gameplay

# Run a single suite: -f / --filter, matching the `test_filter` key
pio test -e native_test -f "test_physics_actor"
pio test -e native_test -f "test_camera2d"
pio test -e native_test_gameplay -f "test_gameplay_grid_space"

# Run with verbose output
pio test -e native_test --verbose

# Generate coverage (Windows)
python scripts/coverage_win.py --report

# Generate coverage (Linux)
python scripts/coverage_linux.py --report
```

> **`-t` is NOT a suite selector.** `AGENTS.md:144` documents
> `pio test -e native_test -t "test_<module>_<function>"`, but `-t` is PlatformIO's `--target`
> (a *build* target), not a test filter. Use `-f` / `--filter`, which matches the `test_filter`
> key in `platformio.ini`. Likewise `AGENTS.md:8-11` documents `pio run -e esp32_full` and
> `-e native_full`; those names carry **no `env:` prefix** in `platformio.ini` and are therefore
> not buildable as written. Treat both as documentation defects, not as usage to copy.

## Flag-Gated Suites

**The single most important testing fact in this repo.** Most gameplay/projection capabilities are
compiled out by default (`PIXELROOT32_ENABLE_*` default `0`). Their test files are written as
`#if <FLAG> ... #else <stub> ... #endif`, so under `native_test` they compile only the `#else`
stub branch — **the suite passes without asserting anything (a vacuous pass).**

| Environment | Purpose | Gameplay/projection flags |
|-------------|---------|---------------------------|
| `[env:native_test]` (`platformio.ini:96`) | Proves the **flags-off contract**: no behavior change for existing examples | all default (off) |
| `[env:native_test_gameplay]` (`platformio.ini:141`) | `extends = env:native_test`; **the only place the gameplay capability assertions actually execute** | 12 flags forced to `1` |

`[env:native_test_gameplay]` adds exactly these 12 `-D` flags, all `=1`:

```
PIXELROOT32_ENABLE_GAMEPLAY_EVENTS      PIXELROOT32_ENABLE_GAMEPLAY_OBJECT_POOL
PIXELROOT32_ENABLE_INTERACTION_TRIGGERS PIXELROOT32_ENABLE_GAMEPLAY_GRID_SPACE
PIXELROOT32_ENABLE_SPATIAL_QUERY        PIXELROOT32_ENABLE_GAMEPLAY_ROOM
PIXELROOT32_ENABLE_DEPTH_SORT           PIXELROOT32_ENABLE_CAMERA_TWEEN
PIXELROOT32_ENABLE_GAMEPLAY_STATE_MACHINE PIXELROOT32_ENABLE_PROJECTION
                                        PIXELROOT32_ENABLE_STATIC_LAYER_SNAPSHOT
                                        PIXELROOT32_ENABLE_TILEMAP_PROJECTION
```

**These suites MUST be run under `-e native_test_gameplay` or they pass vacuously:**

- `test_camera_tween`
- `test_spatial_query`
- `test_scene_depth_sort`
- `test_static_layer_snapshot`
- every `test_gameplay_*` suite (8 of them)
- `test_tilemap_projected_draw`, `test_tilemap_projected_dirty_skip`

Rationale is stated in-repo at `platformio.ini:135-140`. A green `native_test` run is **not**
evidence that a flag-gated capability works — always re-run the affected suite under
`native_test_gameplay` before claiming a gameplay/projection change is covered.

## Test Structure

```
test/
├── test_config.h              # Shared utilities
├── unit/                      # Unit tests
│   ├── test_actor/
│   ├── test_physics_actor/
│   ├── test_collision_system/
│   └── ...
├── test_engine_integration/   # Integration tests
├── test_game_loop/           # Game loop tests
└── mocks/                    # Mock implementations
    ├── MockAudioBackend.h
    ├── MockAudioScheduler.h
    ├── MockDisplay.h
    ├── MockEntity.h
    ├── MockDrawSurface.h
    ├── MockScene.h
    └── MockRenderer.h
```

`test/unit/` currently holds **83 suite directories**. `[env:native_test]` selects them plus a
fixed set of integration suites (`platformio.ini:107-108`):

```ini
test_filter = unit/*, test_player_jump_integration, test_background_palette_render_integration,
              test_engine_integration, test_tile_collection_integration,
              test_user_data_integration, test_game_loop
test_ignore = test_tile_performance_integration, test_user_data_esp32_performance
```

`test_framework = unity`, `test_build_src = true`. The two `test_ignore` suites are performance
benchmarks and are excluded from the default run.

## Test File Naming

- Folder: `test/unit/test_<module>/`
- Source: `test_<module>.cpp`

```cpp
// Function naming: test_<module>_<function>_<scenario>
test_mathutil_lerp_basic
test_physics_actor_set_velocity_float
test_ui_button_click
```

## Basic Test Template

```cpp
#include <unity.h>
#include "module/Header.h"
#include "../../test_config.h"

using namespace pixelroot32::module;

void setUp(void) {
    test_setup();
}

void tearDown(void) {
    test_teardown();
}

void test_module_function_basic(void) {
    // Arrange
    Scalar a = toScalar(0.0f);
    Scalar b = toScalar(10.0f);
    Scalar t = toScalar(0.5f);
    
    // Act
    Scalar result = lerp(a, b, t);
    
    // Assert
    TEST_ASSERT_EQUAL_FLOAT(5.0f, toFloat(result));
}

int main(int argc, char **argv) {
    (void)argc;
    (void)argv;
    UNITY_BEGIN();
    RUN_TEST(test_module_function_basic);
    return UNITY_END();
}
```

## Testing with Mocks

```cpp
#include <unity.h>
#include "audio/AudioEngine.h"
#include "../../mocks/MockAudioBackend.h"
#include "../../test_config.h"

void test_audio_engine_play_event(void) {
    // Arrange
    MockAudioBackend backend;
    // AudioConfig takes a BACKEND POINTER — pass &backend, not backend.
    // A default-constructed AudioConfig has backend == nullptr, so the mock would
    // never observe the event and getEventCount() could never reach 1.
    pixelroot32::audio::AudioConfig config(&backend, 22050);
    // caps is defaulted: AudioEngine(config) and AudioEngine(config, caps) are both valid
    pixelroot32::audio::AudioEngine engine(config);
    
    // Aggregate init follows DECLARATION order — see pixelroot32-audio for all 15 fields:
    //   type, frequency, duration, volume, duty, noisePeriod, preset, sweepEndHz,
    //   sweepDurationSec, loop, sweepCurve, dutySteps, dutyStepCount,
    //   pitchEnvelope, pitchEnvelopeCount
    pixelroot32::audio::AudioEvent event = {
        pixelroot32::audio::WaveType::PULSE,
        440.0f,  // frequency (Hz)
        0.1f,    // duration (seconds)
        0.5f,    // volume (0.0 - 1.0)
        0.5f     // duty (pulse only)
    };
    
    // Act
    engine.playEvent(event);
    
    // Assert
    TEST_ASSERT_EQUAL(1, backend.getEventCount());
}
```

> **Field order is `frequency, duration, volume` — not `frequency, volume, duration`.** The struct
> lives in the external APU library (`PixelRoot32-APU/include/pixelroot32/apu/AudioTypes.h`,
> namespace `pixelroot32::audio`). Only the first five fields have no default initializer, so a
> 5-element brace-init is the safe positional form. `preset` is **field 7**, between `noisePeriod`
> and `sweepEndHz` — a positional init that trails `preset` at the end, or that skips
> `noisePeriod`, compiles and produces silently wrong audio. Prefer named field assignment
> (`event.preset = &INSTR_...;`) for anything past `duty`.

## Integration Test Example

```cpp
#include <unity.h>
#include "core/Engine.h"
#include "../mocks/MockDrawSurface.h"
#include "../test_config.h"

void test_engine_scene_lifecycle(void) {
    auto mock = std::make_unique<MockDrawSurface>();
    pixelroot32::graphics::DisplayConfig config = 
        PIXELROOT32_CUSTOM_DISPLAY(mock.release(), 240, 240);
    
    pixelroot32::core::Engine engine(config);
    auto scene = std::make_unique<MockScene>();
    
    engine.setScene(scene.get());
    TEST_ASSERT_TRUE(engine.getCurrentScene().has_value());
}
```

## Test Utilities (test_config.h)

Available helpers:
- `test_setup()` / `test_teardown()` - Initialize/cleanup
- `test_data::PI`, `test_data::SCREEN_WIDTH` - Common constants
- `float_eq()` - Float comparison
- `TEST_ASSERT_FLOAT_EQUAL(expected, actual)` - Float assertion

## Coverage Targets

| Metric | Target |
|--------|--------|
| Line Coverage | ≥80% |
| Function Coverage | ≥90% |
| Branch Coverage | ≥70% (optional) |

## Coverage Scripts

```bash
# Windows with gcovr
python scripts/coverage_win.py --report

# Linux with lcov
python scripts/coverage_linux.py --report

# Skip tests, only generate report
python scripts/coverage_win.py --no-tests --report
```

## Platform-Specific Tests

```cpp
#ifdef ESP32
void test_esp32_audio_dac(void) {
    TEST_ASSERT_TRUE(ESP.getChipModel() == CHIP_ESP32);
    // ESP32-specific assertions
}
#endif
```

## Best Practices

1. **Naming**: Be descriptive (`test_physics_gravity_acceleration`)
2. **Independence**: Each test runs alone, no order dependencies
3. **Coverage**: Test happy path AND edge cases
4. **Performance**: Keep tests fast (<100ms each)
5. **Mocks**: Use mocks for external dependencies

## Constraints

- Default env: `native_test`; only other buildable env is `native_test_gameplay`
- Flag-gated suites MUST also run under `native_test_gameplay` (see Flag-Gated Suites)
- Single suite selection is `-f` / `--filter`, never `-t`
- Use Unity framework
- Include path: `../../test_config.h` from unit tests
- Use mocks from `../../mocks/` directory
- Embedded tests marked with `test_ignore` in platformio.ini

## Subsystem-Specific Testing

For testing patterns specific to each subsystem, refer to the specialized skills:

Suites marked **†** are flag-gated and pass vacuously outside `-e native_test_gameplay`.

| Subsystem | Skill | Test Suites |
|-----------|-------|-------------|
| Camera | `pixelroot32-camera2d` | `test_camera2d`, `test_camera_effects`, `test_camera_tween`† |
| Audio | `pixelroot32-audio` | `test_audio`, `test_audio_command_queue`, `test_audio_engine`, `test_audio_music_types`, `test_audio_scheduler`, `test_apu_core`, `test_music_player` |
| Rendering | `pixelroot32-sprite-renderer` | `test_graphics`, `test_color`, `test_rgb444`, `test_compute_span_table`, `test_dirty_grid`, `test_dirty_grid_intersects_prev_dirty`, `test_sprite4bpp_framebuffer`, `test_static_tilemap_layer_cache`, `test_static_layer_snapshot`†, `test_font_manager`, `test_display_config`, `test_tile_animation`, `test_tile_attributes`, `test_tile_attributes_unit`, `test_tile_mask`, `test_tile_consumption_helper` |
| Physics | `pixelroot32-physics` | `test_collision_primitives`, `test_collision_system`, `test_collision_types`, `test_physics_actor`, `test_physics_expansion`, `test_physics_scheduler`, `test_kinematic_actor`, `test_rigid_actor`, `test_sensor_actor`, `test_tile_collision_builder`, `test_tile_pixel_collision`, `test_rect` |
| UI | `pixelroot32-ui-system` | `test_ui`, `test_ui_sprite`, `test_ui_touchwidget`, `test_uimanager` |
| Scenes | `pixelroot32-scene-manager` | `test_scene`, `test_scene_manager`, `test_scene_transition`, `test_transition_effect`, `test_diagonal_wipe`, `test_directional_iris`, `test_engine` |
| Input | `pixelroot32-touch-input` | `test_TouchEventDispatcher`, `test_TouchEventQueue`, `test_TouchStateMachine`, `test_touch_calibration`, `test_xpt2046_adapter`, `test_input_config`, `test_input_manager` |
| Particles | `pixelroot32-particles` | `test_particle_emitter` |
| Entities | `pixelroot32-entity-actor` | `test_actor`, `test_entity`, `test_static_actor`, `test_actor_touch_controller` |
| Projection | `pixelroot32-projection` | `test_math_projection`, `test_gameplay_projection`†, `test_projected_map_bounds`, `test_scene_depth_sort`†, `test_tilemap_projected_draw`†, `test_tilemap_projected_dirty_skip`†, `test_tilemap_foot_anchor`, `test_iso_dungeon_projected_conversion` |
| Gameplay framework | `pixelroot32-gameplay-framework` | `test_gameplay_event_bus`†, `test_gameplay_grid_motion`†, `test_gameplay_grid_space`†, `test_gameplay_object_pool`†, `test_gameplay_projection`†, `test_gameplay_room_graph`†, `test_gameplay_room_layout`†, `test_gameplay_state_machine`†, `test_spatial_query`†, `test_interaction_tracker`, `test_room_scene_int` |
| Core / platform | *(no dedicated skill)* | `test_math`, `test_platforms`, `test_platform_capabilities`, `test_platform_log` |

## Agent Constraints
- **Mocks:** Any new unit test testing a dependent module MUST include and use the corresponding Mocks to isolate the logic.
