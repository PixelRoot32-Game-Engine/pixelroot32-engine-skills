---
name: pixelroot32-audio
description: NES-style 8-voice audio subsystem with 5 wave types (Pulse, Triangle, Noise, Sine, Saw), ADSR+LFO+sweep envelopes, Q15 no-FPU path, SPSC command queue, multi-track sequencer, and MusicPlayer. Use when implementing sound effects, music playback, or audio pipeline configuration.
license: MIT
compatibility: opencode>=0.1.0
metadata:
  domain: engine
  subsystem: audio
  module: core
  feature_gate: PIXELROOT32_ENABLE_AUDIO
  platform: cross-platform
---

## Overview

PixelRoot32 provides an NES-style audio subsystem with dynamic 8-voice pooling, 5 wave types, per-voice ADSR envelopes, LFO modulation, frequency sweep, and a multi-track music sequencer. The architecture separates synthesis (`ApuCore`) from scheduling (`AudioScheduler`) and exposes a facade (`AudioEngine`) for game code. A lock-free SPSC command queue mediates between game and audio threads.

## Key APIs

### AudioEngine (Facade)

**Header**: `include/audio/AudioEngine.h`
**Namespace**: `pixelroot32::audio`
**Feature gate**: `PIXELROOT32_ENABLE_AUDIO`

```cpp
AudioConfig config(backend, 22050);  // backend, sampleRate
AudioEngine engine(config, caps);
engine.init();

// One-shot sounds
engine.playEvent({WaveType::PULSE, 440.0f, 0.5f, 0.8f, 0.5f});

// Master control
engine.setMasterVolume(0.75f);
engine.setMasterBitcrush(4);  // 4-bit reduction

// Music transport
bool playing = engine.isMusicPlaying();
bool paused = engine.isMusicPaused();
```

### ApuCore (Synthesis)

**Header**: `include/audio/ApuCore.h`
**Namespace**: `pixelroot32::audio`

```cpp
ApuCore apu;
apu.init(22050);  // sample rate

// Command submission (thread-safe via SPSC queue)
apu.submitCommand(cmd);

// Generate samples (called by scheduler)
apu.generateSamples(stream, length);

// Music sequencer
apu.setSequencerNoteLimit(32);
apu.isMusicPlaying();
apu.isMusicPaused();

// Profiling
ApuCore::ProfileEntry stats[64];
uint8_t count = 64;
apu.getAndResetProfileStats(stats, count);
```

### AudioScheduler (Abstract)

**Header**: `include/audio/AudioScheduler.h`
**Namespace**: `pixelroot32::audio`

```cpp
class MyScheduler : public AudioScheduler {
    void init(AudioBackend* backend, int sampleRate,
              const PlatformCapabilities& caps, int blockSize) override;
    void submitCommand(const AudioCommand& cmd) override;
    void start() override;
    void stop() override;
    void generateSamples(int16_t* stream, int length) override;
    ApuCore& getApuCore() override;
};
```

### MusicPlayer (High-level sequencer)

**Header**: `include/audio/MusicPlayer.h`
**Namespace**: `pixelroot32::audio`

```cpp
MusicPlayer player(engine);

// Play a track (must remain in scope)
player.play(myTrack);

// Transport
player.stop();
player.pause();
player.resume();

// Tempo
player.setTempoFactor(1.5f);  // 150% speed
player.setBPM(180.0f);
```

### AudioCommandQueue (SPSC)

**Header**: `include/audio/AudioCommandQueue.h`
**Namespace**: `pixelroot32::audio`

```cpp
AudioCommandQueue queue;  // Capacity: 128 (configurable via AUDIO_COMMAND_QUEUE_CAPACITY)

// Producer (game thread)
queue.enqueue(cmd);  // Returns false if full

// Consumer (audio thread)
AudioCommand out;
queue.dequeue(out);

// Diagnostics
size_t dropped = queue.getDroppedCommands();
```

## Wave Types

| Wave | Enum | Characteristics |
|------|------|-----------------|
| **Pulse** | `WaveType::PULSE` | Variable duty cycle (12.5%, 25%, 50%, 75%), duty sweep |
| **Triangle** | `WaveType::TRIANGLE` | Smooth, softer timbre — good for bass, pads, leads |
| **Noise** | `WaveType::NOISE` | 15-bit LFSR, short/long mode (93 or 32767-step) |
| **Sine** | `WaveType::SINE` | Band-limited sine via LUT — pure tone |
| **Saw** | `WaveType::SAW` | Polyphonic saw from linear phase ramp — rich harmonics |

## AudioEvent Parameters

```cpp
AudioEvent event;
event.type = WaveType::PULSE;
event.frequency = 440.0f;     // Hz
event.duration = 0.5f;        // seconds
event.volume = 0.8f;          // 0.0 - 1.0
event.duty = 0.5f;            // Pulse duty cycle
event.noisePeriod = 0;        // NOISE: 0=auto, >0=direct LFSR period
event.preset = &INSTR_PULSE_LEAD;  // Instrument preset (ADSR, LFO)
event.sweepEndHz = 880.0f;    // Frequency sweep end
event.sweepDurationSec = 0.3f;// Sweep duration
```

## Instrument Presets

**Header**: `include/audio/AudioMusicTypes.h`

| Preset | Type | Duty | Use |
|--------|------|------|-----|
| `INSTR_PULSE_LEAD` | Pulse | 50% | Melody with vibrato |
| `INSTR_TRIANGLE_LEAD` | Triangle | — | Smooth lead |
| `INSTR_TRIANGLE_PAD` | Triangle | — | Atmospheric pad with tremolo |
| `INSTR_PULSE_PAD` | Pulse | 25% | Evolving pad with PWM |
| `INSTR_PULSE_HARMONY` | Pulse | 12.5% | Harmony with tremolo |
| `INSTR_TRIANGLE_BASS` | Triangle | — | Tight bass |
| `INSTR_PULSE_BASS` | Pulse | 25% | Punchy bass |
| `INSTR_KICK` | Noise | 0% | Percussion: kick drum |
| `INSTR_SNARE` | Noise | 0% | Percussion: snare (93-step) |
| `INSTR_HIHAT` | Noise | 0% | Percussion: hi-hat (93-step) |

## Music Track Format

```cpp
// Define notes
MusicNote melody[] = {
    makeNote(INSTR_PULSE_LEAD, Note::C, 5, 0.25f),
    makeNote(INSTR_PULSE_LEAD, Note::E, 5, 0.25f),
    makeNote(INSTR_PULSE_LEAD, Note::G, 5, 0.25f),
    makeRest(0.5f),
};

// Define track
MusicTrack track = {
    melody,                             // notes
    4,                                  // count
    true,                               // loop
    WaveType::PULSE,                    // channelType
    0.5f,                               // duty
    &secondVoice,                       // secondVoice (nullptr = disabled)
    nullptr,                            // thirdVoice
    &percussionTrack,                   // percussion
};

// Play
MusicPlayer player(engine);
player.play(track);
```

Note helpers: `makeNote(preset, note, octave, duration)`, `makeNote(preset, note, duration)`, `makeRest(duration)`.

## Composition Patterns

### Sound effect on collision

```
Entity::onCollision(other):
  ├── engine.playEvent({WaveType::NOISE, 200, 0.1, 0.5})
  └── engine.playEvent({WaveType::PULSE, 80, 0.15, 0.3, 0.5})
```

### Music with percussion

```
Scene::init():
  ├── player.setBPM(150)
  ├── player.play(mainTrack)    // main melody, harmony, bass
  └── // Track sub-voices and percussion play automatically
```

### Engine + MusicPlayer integration

```
Scene::update(dt):
  ├── if (!player.isPlaying() && nextLevel):
  │     player.play(nextTrack)
  └── engine.generateSamples(stream, length)  // Called by backend
```

## ESP32 Constraints

- **No FPU (ESP32-C3 RISC-V)**: Use Q15 fixed-point path — `EnvelopeState`, `LfoState`, `AudioChannel` have Q15/Q32 mirrors for all hot-path math.
- **IRAM_ATTR**: Mark `generateSamples()` and hot synthesis functions with `IRAM_ATTR` when using I2S DMA to avoid flash contention.
- **Block size**: 128 samples for no-FPU platforms (`platforms::config::HasFPU` controls default).
- **SPSC queue**: `AudioCommandQueue` capacity defaults to 128 (512 bytes). Increase via `AUDIO_COMMAND_QUEUE_CAPACITY` for high-throughput.
- **Sequencer note limit**: Default 32 notes/frame. Set via `setSequencerNoteLimit()` or `AUDIO_SEQUENCER_MAX_NOTES` to prevent audio starvation.
- **Blocking**: Never allocate or block in `generateSamples()`. All synthesis is pre-computed.

## Gotchas

1. **Preset pointers must outlive usage**: `AudioEvent::preset`, `MusicNote::preset`, and `MusicTrack::notes` must point to `static` or `constexpr` data — stack-local temporaries will dangle.
2. **SPSC queue drops**: When the queue is full, the *newest* command is dropped (not the oldest). Monitor `getDroppedCommands()` for backpressure.
3. **Noise channel**: `frequency` sets the LFSR clock rate, not pitch. Shorter `noisePeriod` → lower pitch. `noiseShortMode=true` gives a metallic 93-step LFSR.
4. **Block size alignment**: Must be a multiple of 128 for I2S DMA alignment (compile-time assertion).
5. **Bitcrush**: `setMasterBitcrush(bits)` with bits 0 disables the effect. Values 1-15 re-quantize the final int16 output.
6. **MusicTrack lifetime**: The `MusicTrack` and its `MusicNote` array must remain in scope for the duration of playback — the sequencer references them by pointer.
7. **Profile ring buffer**: `PROFILE_RING_SIZE` is 64 entries. Get-and-reset semantics: each call drains all pending entries.

## Common Patterns

### Retro explosion sound (frequency sweep + noise)
```cpp
engine.playEvent({WaveType::NOISE, 400, 0.3f, 0.6f, 0.0f});
engine.playEvent({WaveType::PULSE, 200, 0.2f, 0.5f, 0.5f, 0,
                  nullptr, 60.0f, 0.2f});  // sweep end Hz
```

### Player jump with preset
```cpp
AudioEvent jump = {WaveType::PULSE, 300, 0.15f, 0.4f, 0.5f, 0, &INSTR_PULSE_BASS};
engine.playEvent(jump);
```
