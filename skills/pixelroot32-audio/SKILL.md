---
name: pixelroot32-audio
description: NES-style 8-voice audio subsystem (4 music + 4 SFX voice partition) with 5 wave types (Pulse, Triangle, Noise, Sine, Saw), ADSR+LFO envelopes, Linear/Exponential frequency sweeps, duty/pitch breakpoint tables, looping SFX, SFX bank playback, Q15 no-FPU path, SPSC command queue, multi-track sequencer, and MusicPlayer. Use when implementing sound effects, music playback, or audio pipeline configuration.
license: MIT
compatibility: opencode>=0.1.0
metadata:
  domain: engine
  subsystem: audio
  module: core
  feature_gate: PIXELROOT32_ENABLE_AUDIO
  platform: cross-platform
  engine_version: "1.9.0+unreleased"
---

## Overview

PixelRoot32 provides an NES-style audio subsystem with an 8-voice pool partitioned 4+4 (slots 0–3 for sequencer music tracks, slots 4–7 for SFX and percussion), 5 wave types, per-voice ADSR envelopes, LFO modulation, Linear/Exponential frequency sweeps, duty/pitch breakpoint automation, looping SFX, and a multi-track music sequencer. The architecture separates synthesis (`ApuCore`) from scheduling (`AudioScheduler`) and exposes a facade (`AudioEngine`) for game code. A lock-free SPSC command queue mediates between game and audio threads.

## Key APIs

### AudioEngine (Facade)

**Header**: `include/audio/AudioEngine.h`
**Namespace**: `pixelroot32::audio`
**Feature gate**: `PIXELROOT32_ENABLE_AUDIO`

```cpp
// struct AudioConfig { AudioBackend* backend = nullptr; int sampleRate = 22050;
//                      int blockSize = platforms::config::HasFPU ? 256 : 128; };
//   AudioConfig(AudioBackend* backend = nullptr, int sampleRate = 22050,
//               int blockSize = platforms::config::HasFPU ? 256 : 128);
// `backend` is a POINTER and may be nullptr for headless configs.
// blockSize must be a multiple of 128 (static_assert, I2S alignment).
AudioConfig config(&backend, 22050);  // pass the ADDRESS of the backend

// Single constructor:
//   AudioEngine(const AudioConfig& config,
//               const platforms::PlatformCapabilities& caps = platforms::PlatformCapabilities());
// Both forms are valid — `caps` is a defaulted parameter:
//   AudioEngine engine(config);          // caps = PlatformCapabilities()
AudioEngine engine(config, caps);         // explicit capabilities
engine.init();

// Re-sync the APU when the backend grants a different device rate
// (e.g. SDL with SDL_AUDIO_ALLOW_FREQUENCY_CHANGE)
engine.reinitSampleRate(confirmedDeviceHz);

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

**Header**: `<pixelroot32/apu/ApuCore.h>` (external library)
**Legacy header**: `include/audio/ApuCore.h` — transitional re-export shim
**Namespace**: `pixelroot32::audio` (unchanged across the boundary)

Since engine **1.8.0** the APU core no longer lives in the engine: it was extracted to the external library `gperez88/PixelRoot32-APU` (`library.json` declares `"gperez88/PixelRoot32-APU": "^2.0.0"`). `include/audio/ApuCore.h` is now a re-export shim containing only `#pragma once` + `#include <pixelroot32/apu/ApuCore.h>`, so **existing `#include "audio/ApuCore.h"` keeps working unchanged**. NEW code should include `<pixelroot32/apu/ApuCore.h>` directly.

The namespace, the class-scoped constants and `ApuCore::ProfileEntry` are unchanged — only the physical header location moved.

```cpp
ApuCore apu;
apu.init(22050);  // sample rate

// Voice partition constants (MAX_VOICES = 8)
ApuCore::MUSIC_VOICE_BASE;   // 0 — first sequencer track slot (trackIdx maps 1:1)
ApuCore::MUSIC_VOICE_COUNT;  // 4 (= MAX_MUSIC_TRACKS)
ApuCore::SFX_VOICE_BASE;     // 4 — first PLAY_EVENT / percussion slot
ApuCore::SFX_VOICE_COUNT;    // 4 (= MAX_VOICES - SFX_VOICE_BASE)
ApuCore::TICKS_PER_BEAT;     // 4
ApuCore::PROFILE_RING_SIZE;  // 64

// Command submission (thread-safe via SPSC queue)
apu.submitCommand(cmd);

// Generate samples (called by scheduler)
apu.generateSamples(stream, length);

// Music sequencer
apu.setSequencerNoteLimit(32);
apu.isMusicPlaying();
apu.isMusicPaused();

// Profiling
// struct ApuCore::ProfileEntry { uint64_t audioTimeSamples; float peak; bool clipped; };
ApuCore::ProfileEntry stats[64];
uint8_t count = 64;
apu.getAndResetProfileStats(stats, count);
```

### Engine-owned vs APU library headers

Know which side of the 1.8.0 boundary a symbol lives on before including it.

**Re-export shims in `include/audio/` (real definitions in `gperez88/PixelRoot32-APU@^2.0.0`)** — each is only a comment + `#pragma once` + one include:

| Legacy shim | Re-exports |
|---|---|
| `include/audio/ApuCore.h` | `<pixelroot32/apu/ApuCore.h>` |
| `include/audio/AudioCommandQueue.h` | `<pixelroot32/apu/AudioCommandQueue.h>` |
| `include/audio/AudioMixerLUT.h` | `<pixelroot32/apu/AudioMixerLUT.h>` |
| `include/audio/AudioMusicTypes.h` | `<pixelroot32/apu/AudioMusicTypes.h>` |
| `include/audio/AudioOscLUT.h` | `<pixelroot32/apu/AudioOscLUT.h>` |
| `include/audio/AudioTypes.h` | `<pixelroot32/apu/AudioTypes.h>` |

**Still engine-owned in `include/audio/`**: `AudioBackend.h`, `AudioConfig.h`, `AudioEngine.h`, `AudioScheduler.h`, `DefaultAudioScheduler.h`, `MusicPlayer.h`, `SfxBankPlayback.h`.

**Which side owns which type** — these four are all defined in the external `PixelRoot32-APU` library, *not* in the engine:

| Type | Real definition | Reached through |
|---|---|---|
| `AudioEvent` | `PixelRoot32-APU/include/pixelroot32/apu/AudioTypes.h` | `<pixelroot32/apu/AudioTypes.h>` (shim: `audio/AudioTypes.h`) |
| `SweepCurve` | `PixelRoot32-APU/include/pixelroot32/apu/AudioTypes.h` | same |
| `SfxBreakpoint` | `PixelRoot32-APU/include/pixelroot32/apu/AudioTypes.h` | same |
| `InstrumentPreset` | `PixelRoot32-APU` (`pixelroot32/apu/AudioMusicTypes.h`) | `<pixelroot32/apu/AudioMusicTypes.h>` (shim: `audio/AudioMusicTypes.h`) |

All four keep namespace `pixelroot32::audio` on both sides of the boundary. `AudioEvent` is
**forward-declared** against `struct InstrumentPreset` inside `AudioTypes.h` — the full preset
definition comes from `AudioMusicTypes.h`.

Old includes of the shim paths keep compiling unchanged; new code should include the `<pixelroot32/apu/...>` path directly. The namespace stays `pixelroot32::audio` on both sides.

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

Strictly one producer + one consumer — concurrent multi-producer use is not supported (the enqueue path has no CAS retry).

### SFX Bank Playback (Helper)

**Header**: `include/audio/SfxBankPlayback.h`
**Namespace**: `pixelroot32::audio`

Header-only helper to play Tool Suite–style SFX bank entries: layers fire at t=0, timed sequence steps are delegated to a game-owned scheduler. Zero-allocation; the helper does not own timing.

```cpp
// Bank must expose a static API:
//   static uint8_t layerCount(SfxId);
//   static AudioEvent layerEvent(SfxId, uint8_t);
//   static uint8_t sequenceStepCount(SfxId);
//   static SequenceStep sequenceStep(SfxId, uint8_t);  // {float delaySec; AudioEvent event;}

playSfxBank<MySfxBank>(engine, SfxId::Coin, myScheduler);

// Scheduler implementations (subclass SfxDelayScheduler):
NullSfxDelayScheduler noDelays;             // banks with simultaneous layers only
ImmediateSfxDelayScheduler now(engine);     // test/stub: ignores delays
// Game code: implement schedule(delaySec, event) with a scene timer
```

## Voice Allocation (4+4 Partition)

- **Slots 0–3 (music)**: Reserved for sequencer melodic tracks; `trackIdx` maps 1:1 to slot in O(1). Melodic tracks never steal from each other, and SFX cannot interrupt melodic notes.
- **Slots 4–7 (SFX)**: Shared subpool for `PLAY_EVENT` effects and sequencer percussion hits. Voice stealing is confined to this subpool, and the steal ranking prefers reclaiming looped voices first.
- **Percussion overflow**: When the SFX subpool is saturated, a percussion hit may borrow an *idle* music voice — it never interrupts an active melodic note.

## Wave Types

| Wave | Enum | Characteristics |
|------|------|-----------------|
| **Pulse** | `WaveType::PULSE` | Variable duty cycle (12.5%, 25%, 50%, 75%), duty sweep |
| **Triangle** | `WaveType::TRIANGLE` | Smooth, softer timbre — good for bass, pads, leads |
| **Noise** | `WaveType::NOISE` | 15-bit LFSR, short/long mode (93 or 32767-step) |
| **Sine** | `WaveType::SINE` | Band-limited sine via LUT — pure tone |
| **Saw** | `WaveType::SAW` | Polyphonic saw from linear phase ramp — rich harmonics |

## AudioEvent Parameters

### Declaration Order (15 fields — authoritative)

`struct AudioEvent` is defined in the external APU library at
`PixelRoot32-APU/include/pixelroot32/apu/AudioTypes.h:720-771`, namespace `pixelroot32::audio`.
Aggregate (brace) initialization follows this exact order:

| # | Type | Field | Default |
|---|------|-------|---------|
| 1 | `WaveType` | `type` | *(none)* |
| 2 | `float` | `frequency` | *(none)* |
| 3 | `float` | `duration` | *(none)* — seconds |
| 4 | `float` | `volume` | *(none)* — 0.0–1.0 |
| 5 | `float` | `duty` | *(none)* — pulse only |
| 6 | `uint8_t` | `noisePeriod` | `0` |
| 7 | `const InstrumentPreset*` | `preset` | `nullptr` |
| 8 | `float` | `sweepEndHz` | `0.0f` |
| 9 | `float` | `sweepDurationSec` | `0.0f` |
| 10 | `bool` | `loop` | `false` |
| 11 | `SweepCurve` | `sweepCurve` | `SweepCurve::Linear` |
| 12 | `const SfxBreakpoint*` | `dutySteps` | `nullptr` |
| 13 | `uint8_t` | `dutyStepCount` | `0` |
| 14 | `const SfxBreakpoint*` | `pitchEnvelope` | `nullptr` |
| 15 | `uint8_t` | `pitchEnvelopeCount` | `0` |

Only fields **1–5 have no default initializer** — a 5-element positional brace-init is the safe
short form. `sweepCurve` sits at position 11 by design: its in-source comment reads *"additive at
end of struct for brace-init safety"*, i.e. it was appended after the sweep fields so older
brace-inits kept compiling.

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
event.loop = false;           // true = continuous until STOP_CHANNEL (or steal)
event.sweepCurve = SweepCurve::Linear;  // or SweepCurve::Exponential
event.dutySteps = nullptr;    // PULSE: stepped duty table (SfxBreakpoint*)
event.dutyStepCount = 0;      // max kMaxSfxDutySteps (4)
event.pitchEnvelope = nullptr;// multi-breakpoint pitch table (SfxBreakpoint*)
event.pitchEnvelopeCount = 0; // max kMaxSfxPitchPoints (4); >= 2 to activate
```

### Frequency Sweep

Active iff `sweepDurationSec > 0` and `sweepEndHz > 0`. Works on **all wave types**: melodic waves interpolate `frequency → sweepEndHz`; NOISE interpolates the LFSR clock Hz (and thus the period). `SweepCurve::Exponential` is geometric in Hz (falls back to Linear if start/end are not both > 0). Duration is clamped to note length for one-shots; looping voices run the full `sweepDurationSec`.

### Breakpoint Tables (SfxBreakpoint)

```cpp
struct SfxBreakpoint { float timeSec; float value; };  // time from voice start; non-decreasing

static constexpr SfxBreakpoint kLaserDuty[] = {{0.0f, 0.50f}, {0.05f, 0.25f}, {0.10f, 0.125f}};
event.dutySteps = kLaserDuty;         // PULSE only; hold semantics between points
event.dutyStepCount = 3;              // ignores InstrumentPreset::dutySweep while active

static constexpr SfxBreakpoint kFallPitch[] = {{0.0f, 880.0f}, {0.1f, 440.0f}, {0.3f, 110.0f}};
event.pitchEnvelope = kFallPitch;     // >= 2 points supersede the single-segment sweep
event.pitchEnvelopeCount = 3;
```

Tables MUST point to `static`/`constexpr` data (the voice holds pointer + count). FPU and Q15 paths supported; no heap allocation in `generateSamples()`.

### Looping SFX and Stop

```cpp
event.loop = true;            // voice stays enabled — no auto-disable
engine.playEvent(event);

AudioCommand stop{};
stop.type = AudioCommandType::STOP_CHANNEL;
stop.channelIndex = voiceSlot;        // SFX slots are 4-7
engine.submitCommand(stop);
```

When `loop == false`, `duration <= 0` disables the voice immediately — it never leaves a hanging voice.

## Instrument Presets

**Header**: `<pixelroot32/apu/AudioMusicTypes.h>` (external APU library; `include/audio/AudioMusicTypes.h` is the re-export shim)
**Namespace**: `pixelroot32::audio`

| Preset | Type | Duty | Use |
|--------|------|------|-----|
| `INSTR_PULSE_LEAD` | Pulse | 50% | Melody with vibrato |
| `INSTR_TRIANGLE_LEAD` | Triangle | — | Smooth lead |
| `INSTR_TRIANGLE_PAD` | Triangle | — | Atmospheric pad with tremolo |
| `INSTR_PULSE_PAD` | Pulse | 25% | Evolving pad with PWM |
| `INSTR_PULSE_HARMONY` | Pulse | 12.5% | Harmony with tremolo |
| `INSTR_TRIANGLE_BASS` | Triangle | — | Tight bass |
| `INSTR_PULSE_BASS` | Pulse | 25% | Punchy bass |
| `INSTR_KICK` | Noise | 0% | Percussion: kick drum (noisePeriod 60) |
| `INSTR_SNARE` | Noise | 0% | Percussion: snare (93-step, noisePeriod 15) |
| `INSTR_HIHAT` | Noise | 0% | Percussion: hi-hat (93-step) |

`InstrumentPreset` also supports an optional pitch sweep: set `pitchSweepEndHz` and `pitchSweepDurationSec` (both > 0 to activate) — materialized into the event's `sweep*` fields at note-on. When defining custom presets with brace-init, these are the last two fields.

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

### Note Duration Semantics

- `MusicNote::duration` is the sequencer advance in **beats** (quarter note = 1.0; `ApuCore::TICKS_PER_BEAT` = 4).
- `duration == 0.0` fires a **stacked hit** without advancing tempo — use it to layer Kick/Snare/Hi-Hat on the same step of the NOISE track.
- On the percussion track, only **`Rest` + noise preset** counts as a drum hit; a plain `Rest` (no preset) remains silence and a melodic `Rest` is a track-scoped note-off (it only releases that track's voice).
- When `tempoFactor > 1` truncates a note to 0 ticks, the sequencer clamps to 1 tick minimum.

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
8. **Looping voices never auto-stop**: `loop = true` keeps the voice enabled until `STOP_CHANNEL` or a steal. Always pair a looping `playEvent` with an explicit stop path (e.g. on scene exit).
9. **Breakpoint tables dangle like presets**: `dutySteps` and `pitchEnvelope` are pointer+count — the tables must be `static`/`constexpr`, with non-decreasing `timeSec`.
10. **SFX voice stealing is subpool-local**: Effects can only steal slots 4–7 (looped voices are reclaimed first). If 4 non-looping SFX are active, a new effect steals one of them — melodic music voices are never interrupted.
11. **Sequencer track slots are fixed**: Track N always plays on voice slot N (0–3). `STOP_CHANNEL` with `channelIndex` 0–3 kills a music track's voice.
12. **The APU is an external dependency**: since engine 1.8.0 the synthesis core ships as `gperez88/PixelRoot32-APU@^2.0.0`. A game's `platformio.ini` must resolve that dependency (`lib_deps`) or every `audio/ApuCore.h` shim include fails to compile.
13. **`preset` is field 7, NOT the last field**: it sits between `noisePeriod` (6) and `sweepEndHz` (8). A positional brace-init that trails `preset` at the end — `{type, frequency, duration, volume, duty, &INSTR_X}` — silently assigns the preset pointer to `noisePeriod`'s slot (or fails to compile), and omitting `noisePeriod` while positionally initializing `preset` shifts every field after it. Both produce **silently wrong audio, not a build error**. Past `duty`, prefer named assignment (`event.preset = &INSTR_PULSE_BASS;`). The correct positional form is `{type, frequency, duration, volume, duty, 0, &INSTR_PULSE_BASS}`.
14. **Field order is `frequency, duration, volume`** — not `frequency, volume, duration`. Swapping 3 and 4 compiles cleanly (both `float`) and yields a note of the wrong length at the wrong loudness.

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

### Looping engine hum (stop on scene exit)
```cpp
AudioEvent hum{};
hum.type = WaveType::TRIANGLE;
hum.frequency = 55.0f;
hum.volume = 0.3f;
hum.loop = true;
engine.playEvent(hum);

// Later (e.g. Scene::resetState or onExit):
AudioCommand stop{};
stop.type = AudioCommandType::STOP_CHANNEL;
stop.channelIndex = slot;  // SFX subpool: 4-7
engine.submitCommand(stop);
```

### Falling pitch with exponential curve
```cpp
AudioEvent fall{};
fall.type = WaveType::PULSE;
fall.frequency = 880.0f;
fall.duration = 0.4f;
fall.volume = 0.6f;
fall.duty = 0.5f;
fall.sweepEndHz = 110.0f;
fall.sweepDurationSec = 0.4f;
fall.sweepCurve = SweepCurve::Exponential;
engine.playEvent(fall);
```

## Agent Constraints
- **Feature Gate:** ALWAYS wrap AudioEngine usage in `#if PIXELROOT32_ENABLE_AUDIO`.
- **Testing:** If you modify audio components, YOU MUST read `pixelroot32-testing` and update/create the corresponding unit tests using Mocks.
