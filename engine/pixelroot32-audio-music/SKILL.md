---
name: pixelroot32-audio-music
description: Implement audio effects and music playback using the NES-style 4-channel audio subsystem
license: MIT
compatibility: opencode>=0.1.0
metadata:
  domain: engine
  language: cpp
  platform: esp32
---

## Overview

Generate audio code using PixelRoot32's NES-style audio system with Pulse, Triangle, and Noise channels, plus music sequencing.

## Source of Truth

- `docs/AUDIO_NES_SUBSYSTEM_REFERENCE.md` - Audio engine details
- `docs/MUSIC_PLAYER_GUIDE.md` - Music playback
- `docs/API_REFERENCE.md` - AudioEngine, MusicPlayer APIs

## Audio Architecture

```
AudioEngine (Facade)
└── AudioScheduler (Strategy)
    ├── AudioCommandQueue
    ├── Channel Generators (Pulse, Triangle, Noise)
    └── Mixer with LUT
```

## Modular Compilation

Audio is conditionally compiled:

```cpp
#if PIXELROOT32_ENABLE_AUDIO
// Audio code here
#endif

// Disable to save ~8KB RAM, ~15KB Flash
```

## Wave Types

| Type | Description | Use Case |
|------|-------------|----------|
| PULSE | Square wave with variable duty | SFX, melodies |
| TRIANGLE | Triangle wave (fixed volume) | Bass, flute |
| NOISE | Pseudo-random noise | Explosions, drums |

## Playing Sound Effects

```cpp
#include <audio/AudioEngine.h>

pixelroot32::audio::AudioEvent evt{};
evt.type = pixelroot32::audio::WaveType::PULSE;
evt.frequency = 1500.0f;  // Hz
evt.duration = 0.12f;      // seconds
evt.volume = 0.8f;         // 0.0 to 1.0
evt.duty = 0.5f;           // Pulse duty cycle

audioEngine.playEvent(evt);
```

## Music Track Definition

```cpp
#include <audio/AudioMusicTypes.h>
#include <audio/MusicPlayer.h>

using namespace pixelroot32::audio;

// Define melody using instrument presets
static const MusicNote MELODY[] = {
    makeNote(INSTR_PULSE_LEAD, Note::C, 0.20f),
    makeNote(INSTR_PULSE_LEAD, Note::E, 0.20f),
    makeNote(INSTR_PULSE_LEAD, Note::G, 0.25f),
    makeRest(0.10f),
};

static const MusicTrack GAME_MUSIC = {
    MELODY,
    sizeof(MELODY) / sizeof(MusicNote),
    true,   // loop
    WaveType::PULSE,
    0.5f    // duty
};
```

## Instrument Presets

| Preset | Description |
|--------|-------------|
| `INSTR_PULSE_LEAD` | Main lead pulse, octave 4 |
| `INSTR_PULSE_BASS` | Bass pulse, octave 3 |
| `INSTR_PULSE_CHIP_HIGH` | High-pitched chiptune, octave 5 |
| `INSTR_TRIANGLE_PAD` | Soft triangle pad, octave 4 |

## Note Enum

```cpp
enum Note {
    C, Cs, D, Ds, E, F, Fs, G, Gs, A, As, B, Rest
};

// Convert to frequency
float freq = noteToFrequency(Note::C, 4);  // C4 = 261.63 Hz
```

## MusicPlayer Usage

```cpp
void MyScene::init() override {
    engine.getMusicPlayer().play(GAME_MUSIC);
}

void MyScene::update(unsigned long dt) override {
    if (paused) {
        engine.getMusicPlayer().pause();
    } else {
        engine.getMusicPlayer().resume();
    }
}
```

## Tempo Control

```cpp
musicPlayer.setTempoFactor(2.0f);   // Double speed
musicPlayer.setTempoFactor(0.5f);   // Half speed
musicPlayer.setTempoFactor(1.0f);   // Normal
```

## Audio Configuration

```cpp
pixelroot32::audio::AudioConfig config;
config.sampleRate = 22050;  // Default
config.backend = &myBackend;  // Platform-specific (I2S, DAC, SDL2)
```

## Duty Cycle Values

Standard NES duty cycles:
- 0.125 (12.5%) - High pitch sounds
- 0.25 (25%) - Classic NES sound
- 0.5 (50%) - Square wave
- 0.75 (75%) - Softer tone

## Constraints

- Audio subsystem requires `PIXELROOT32_ENABLE_AUDIO=1`
- Audio does NOT use Scalar - uses internal integer mixing
- Use instrument presets for consistent sound
- Memory: ~8KB RAM, ~15KB Flash when enabled
- Multi-core on ESP32: audio on core 0, game on core 1
