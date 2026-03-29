---
name: pixelroot32-testing
description: Write unit and integration tests using Unity framework with mocks, coverage analysis, and CI integration
license: MIT
compatibility: opencode>=0.1.0
metadata:
  domain: tooling
  language: cpp
  platform: cross-platform
---

## Overview

Generate tests using Unity framework with PlatformIO, mock implementations, and coverage analysis.

## Source of Truth

- `docs/TESTING_GUIDE.md` - Testing practices
- `test/test_config.h` - Shared utilities and macros
- `test/mocks/` - Mock implementations

## Running Tests

```bash
# Run all tests on native platform
pio test -e native_test

# Run specific test suite
pio test -e native_test -f test_physics_actor

# Run with verbose output
pio test -e native_test --verbose

# Generate coverage (Windows)
python scripts/coverage_win.py --report

# Generate coverage (Linux)
python scripts/coverage_linux.py --report
```

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
    ├── MockDrawSurface.h
    └── MockRenderer.h
```

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
    pixelroot32::audio::AudioConfig config;
    MockAudioBackend backend;
    pixelroot32::audio::AudioEngine engine(config);
    
    pixelroot32::audio::AudioEvent event = {
        pixelroot32::audio::WaveType::PULSE,
        440.0f,  // frequency
        0.5f,    // volume
        0.1f     // duration
    };
    
    // Act
    engine.playEvent(event);
    
    // Assert
    TEST_ASSERT_EQUAL(1, backend.getEventCount());
}
```

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

- Default env: `native_test`
- Use Unity framework
- Include path: `../../test_config.h` from unit tests
- Use mocks from `../../mocks/` directory
- Embedded tests marked with `test_ignore` in platformio.ini
