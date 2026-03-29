# pixelroot32-skills

Skills catalog for working with the **PixelRoot32 Game Engine** ecosystem.

## Description

Collection of specialized skills for game development with PixelRoot32, a game engine designed specifically for ESP32 and other embedded platforms. Each skill provides guides, code patterns, and best practices for different engine subsystems.

## Project Structure

```
pixelroot32-skills/
├── engine/                    # Engine skills
│   ├── pixelroot32-sprite-graphics/
│   ├── pixelroot32-testing/
│   ├── pixelroot32-memory-optimization/
│   ├── pixelroot32-ui-development/
│   ├── pixelroot32-tilemap-animations/
│   ├── pixelroot32-audio-music/
│   ├── pixelroot32-physics-collision/
│   └── pixelroot32-cpp-code-generation/
└── tool-suite/               # (Reserved for tools)
```

## Available Skills

### Engine

| Skill | Description | Platform |
|-------|-------------|----------|
| `pixelroot32-sprite-graphics` | Sprites, tilemaps and rendering (1bpp/2bpp/4bpp) with multi-palette support | ESP32 |
| `pixelroot32-testing` | Unit and integration tests with Unity, coverage analysis | Cross-platform |
| `pixelroot32-memory-optimization` | Memory optimization techniques, object pooling, zero-allocation patterns | ESP32 |
| `pixelroot32-ui-development` | User interfaces with layouts, D-pad navigation | ESP32 |
| `pixelroot32-tilemap-animations` | Tile animations with O(1) frame resolution, zero dynamic allocations | ESP32 |
| `pixelroot32-audio-music` | NES-style 4-channel audio subsystem | ESP32 |
| `pixelroot32-physics-collision` | Physics actors, collision detection, Flat Solver | ESP32 |
| `pixelroot32-cpp-code-generation` | C++17 code generation following engine conventions | Cross-platform |

## Usage

Each skill contains a `SKILL.md` file with:

- **Overview**: General subsystem description
- **Source of Truth**: Reference documentation
- **Code Patterns**: Usage examples
- **Constraints**: Restrictions and limitations
- **Memory Budgets**: Memory usage per feature

## Engine Key Features

### Graphics System

- Sprite formats: 1bpp (default), 2bpp, 4bpp
- Dual palette system
- 2D camera with smooth follow
- Render layers (Background, Gameplay, UI)
- Resolution scaling

### Physics

- Actor types: Static, Kinematic, Rigid, Sensor
- Flat Solver with 5-step pipeline
- Spatial grid for collision optimization
- One-way platforms

### Audio

- 4 NES-style channels: Pulse, Triangle, Noise
- Music sequencer with playback
- Instrument presets

### UI

- Layouts: Vertical, Horizontal, Grid, Anchor
- D-pad navigation
- Panels and containers system

## Modular Compilation

The engine allows disabling subsystems to save memory:

| Subsystem | RAM | Flash |
|-----------|-----|-------|
| Audio | ~8 KB | ~15 KB |
| Physics | ~12 KB | ~25 KB |
| UI | ~4 KB | ~20 KB |
| Particles | ~6 KB | ~10 KB |

## Requirements

- **Compatibility**: opencode >= 0.1.0
- **Language**: C++17
- **Primary platform**: ESP32
- **Test framework**: Unity (PlatformIO)

## License

MIT
