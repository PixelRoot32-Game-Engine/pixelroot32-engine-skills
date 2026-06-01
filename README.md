# pixelroot32-skills

Skills catalog for working with the **PixelRoot32 Game Engine** ecosystem.

## Description

Collection of specialized skills for game development with PixelRoot32, a game engine designed specifically for ESP32 and other embedded platforms. Each skill provides guides, code patterns, and best practices for different engine subsystems.

## Project Structure

```
pixelroot32-skills/
├── engine/                        # Engine skills
│   ├── pixelroot32-audio/
│   ├── pixelroot32-camera2d/
│   ├── pixelroot32-cpp-code-generation/
│   ├── pixelroot32-docs/
│   ├── pixelroot32-entity-actor/
│   ├── pixelroot32-memory-optimization/
│   ├── pixelroot32-particles/
│   ├── pixelroot32-physics/
│   ├── pixelroot32-scene-manager/
│   ├── pixelroot32-sprite-renderer/
│   ├── pixelroot32-testing/
│   ├── pixelroot32-touch-input/
│   └── pixelroot32-ui-system/
└── tool-suite/                   # (Reserved for tools)
```

## Available Skills

### Engine

| Skill | Description | Platform |
|-------|-------------|----------|
| `pixelroot32-sprite-renderer` | Rendering system: 1bpp/2bpp/4bpp sprites, MultiSprite layering, tilemaps, dirty regions, palette management (single/dual/multi-slot), text rendering | Cross-platform |
| `pixelroot32-testing` | Unit and integration tests with Unity framework, coverage analysis, mocks | Cross-platform |
| `pixelroot32-memory-optimization` | Memory optimization: modular compilation, object pooling, zero-allocation patterns for ESP32 | ESP32 |
| `pixelroot32-ui-system` | Touch UI: layouts (Anchor, Grid, Horizontal, Vertical), widgets (Button, Checkbox, Label, Panel, Slider), hit testing, UIManager | Cross-platform |
| `pixelroot32-audio` | NES-style 8-voice audio: 5 wave types, ADSR+LFO+sweep envelopes, Q15 no-FPU path, SPSC queue, multi-track sequencer | Cross-platform |
| `pixelroot32-physics` | Flat Solver 6-step pipeline, broad/narrow phase collision, CCD, one-way platforms, spatial grid, tile collision builder | Cross-platform |
| `pixelroot32-entity-actor` | Entity/Actor/PhysicsActor hierarchy, collision layers/masks, 4 body types (Static, Kinematic, Rigid, Sensor), Fixed16 math | Cross-platform |
| `pixelroot32-camera2d` | Camera2D viewport management, follow-target scrolling, bounds clamping, camera effects (shake, punch, offset) | Cross-platform |
| `pixelroot32-particles` | Particle system: configurable emitters, built-in presets (Fire, Explosion, Smoke, Sparks, Dust), velocity/gravity/friction physics, color interpolation | Cross-platform |
| `pixelroot32-scene-manager` | Scene lifecycle, stack-based navigation, Fade/Iris transitions, entity arena allocation, CameraEffects integration | Cross-platform |
| `pixelroot32-touch-input` | Touch input pipeline: adapter pattern (XPT2046, GT911), state machine gesture detection, pull-based event dispatching, calibration | Cross-platform |
| `pixelroot32-cpp-code-generation` | C++17 code generation following engine conventions, naming patterns, and architectural practices | Cross-platform |
| `pixelroot32-docs` | Doxygen documentation guidelines for engine APIs | Cross-platform |

## Usage

Each skill contains a `SKILL.md` file with:

- **Overview**: General subsystem description
- **Key APIs**: Header references, namespaces, and code examples
- **Composition Patterns**: Integration examples across subsystems
- **ESP32 Constraints**: Platform-specific limitations and gotchas
- **Common Patterns**: Ready-to-use code snippets

## Engine Key Features

### Graphics System

- Sprite formats: 1bpp (monochrome), 2bpp (4-color), 4bpp (16-color)
- MultiSprite layering (compose multiple 1bpp layers)
- 3 tilemap types with runtime activation mask
- Dirty region optimization (8×8 cell grid)
- Static tilemap cache for non-dynamic layers
- Palette system: single, dual (background vs sprites), multi-slot (8 slots)
- Step-based sprite animation
- Text rendering with custom fonts

### Camera

- 2D camera with smooth follow-target
- Per-axis bounds clamping
- Camera effects: shake (Xorshift32), punch (directional impulse), offset
- 4 effect slots with round-robin allocation
- Parallax scrolling support

### Physics

- Flat Solver 6-step pipeline: Detect → Solve Velocity → Integrate → Solve Penetration → Callbacks
- Actor types: Static, Kinematic, Rigid, Sensor
- Continuous Collision Detection (CCD) for fast bodies
- Spatial grid with separate static/dynamic layers
- One-way platforms
- Tile collision builder with merge optimization

### Audio

- 8-voice dynamic pooling
- 5 wave types: Pulse, Triangle, Noise, Sine, Saw
- Per-voice ADSR envelopes, LFO modulation, frequency sweep
- 10 instrument presets
- Multi-track music sequencer with MusicPlayer
- Lock-free SPSC command queue
- Q15 fixed-point path for no-FPU cores

### UI

- Entity-based UI elements integrated with scene graph
- Layouts: Anchor, Grid, Horizontal, Vertical
- Touch widgets: Button, Checkbox, Slider, Panel, Label
- UIManager for touch event routing and hover/pressed/captured state
- Fixed-position HUD support (ignores camera scroll)

### Particles

- Entity-based emitters with fixed-size pool (50 particles max)
- Built-in presets: Fire, Explosion, Smoke, Sparks, Dust
- Configurable: speed, gravity, friction, angle-based emission, color interpolation
- Zero heap allocation

### Scenes

- Stack-based scene navigation (push/pop)
- Transition effects: Fade, Iris
- Entity arena allocation (placement new, zero-heap)
- Framebuffer optimization (skip draw when unchanged)
- Touch event pipeline: UI → unconsumed scene handling

### Input

- Touch adapter pattern: XPT2046 (resistive, 1-point), GT911 (capacitive, 5-point)
- State machine gesture detection: Click, DoubleClick, LongPress, DragStart/Move/End
- Pull-based event dispatching
- InputManager for physical buttons + keyboard + touch
- ActorTouchController for touch-to-actor mapping

### Entities

- Godot-inspired hierarchy: Entity → Actor → PhysicsActor
- Collision layers and masks (uint16_t bitmask)
- Adaptable Scalar type: float on PC, Fixed16 Q16.16 on ESP32
- Component model with lifecycle hooks (update/draw)

## Modular Compilation

The engine allows disabling subsystems to save memory:

| Subsystem | RAM | Flash |
|-----------|-----|-------|
| Audio | ~8 KB | ~15 KB |
| Physics | ~12 KB | ~25 KB |
| UI | ~4 KB | ~20 KB |
| Particles | ~6 KB | ~10 KB |
| **All disabled** | **~30 KB** | **~70 KB** |

## Requirements

- **Compatibility**: opencode >= 0.1.0
- **Language**: C++17
- **Primary platform**: ESP32 (also runs on native/desktop)
- **Test framework**: Unity (PlatformIO)
- **Touch drivers**: XPT2046 (SPI resistive) or GT911 (I2C capacitive)

## License

MIT
