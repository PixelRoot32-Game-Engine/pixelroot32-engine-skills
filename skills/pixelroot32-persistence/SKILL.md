---
name: pixelroot32-persistence
description: The PixelRoot32 persistence module — IKeyValueStore/ISlotStore storage models, StoreResult error model, RecordCodec 16-byte CRC-32 envelope, PersistenceService roles and record helpers, and the ESP32 NVS, microSD, host-file and in-memory adapters. Use when saving settings, high scores, unlocks or save slots, including on a microSD card, migrating a record schema, or testing save code without hardware.
license: MIT
compatibility: opencode>=0.1.0
metadata:
  domain: engine
  subsystem: persistence
  module: persistence
  feature_gate: PIXELROOT32_ENABLE_PERSISTENCE
  platform: cross-platform
  engine_version: "1.16.0"
---

## Overview

`pixelroot32::persistence` keeps small game data between runs: settings, high scores,
unlocks and small save slots. It separates **storage models** from **media**:

```text
Game code
   |  settings(), saves()                (roles, bound at setup)
PersistenceService  ------------------  RecordCodec (envelope + CRC-32)
   |-- IKeyValueStore    typed ints, blobs, erase, commit
   `-- ISlotStore        N numbered records, replace-or-keep per slot
          ^
Adapters (one per medium)
   |-- Esp32NvsKeyValueStore / Esp32NvsSlotStore   drivers/esp32   (ESP32)
   |-- SdSlotStore                                 drivers/esp32   (ESP32 + SD)
   |-- HostFileKeyValueStore / HostFileSlotStore   drivers/native  (PLATFORM_NATIVE)
   `-- MemoryKeyValueStore   / MemorySlotStore     persistence/    (any target)
```

Game logic talks only to `IKeyValueStore&` / `ISlotStore&`; which medium sits behind
them is decided once, where the stores are constructed. A new medium adds an adapter and
touches neither the interfaces nor game code. Adapters move opaque bytes; `RecordCodec`
is the one place a record is validated.

**`Engine` owns none of it.** The game constructs its stores and the service, calls each
adapter's `init()`, and binds the stores at setup. **The engine never writes on its own:**
every set, erase and `commit()` is a call the game makes. Reference implementation:
`examples/persistence` (one settings record + one save slot, every `StoreResult` printed,
same scene on ESP32 and native); its `esp32sd` environment keeps settings in NVS and moves
the save slots to a microSD card.

**Two flags gate the module.** `PIXELROOT32_ENABLE_PERSISTENCE`, default `0`, is the
module gate: with it off every header body compiles to nothing — no symbols, no `Engine`
bytes, no `Scene` vtable change. `PIXELROOT32_ENABLE_SD_STORAGE`, default `0`, adds the
microSD adapter only; it requires the module flag and an ESP32 target, and the build fails
with a named `#error` when either is missing. Capacities stay constructor arguments or
template parameters. Mirrors: `pixelroot32::platforms::config::EnablePersistence` /
`EnableSdStorage` (`include/platforms/EngineConfig.h`) and
`pixelroot32::modules::Persistence` (`include/core/EngineModules.h`).

```ini
build_flags =
    -D PIXELROOT32_ENABLE_PERSISTENCE=1
    ; ESP32 only, and only when the saves live on a microSD card:
    -D PIXELROOT32_ENABLE_SD_STORAGE=1
```

## Key APIs

### StoreTypes — results and the key rule

**Header** `include/persistence/StoreTypes.h` · **Namespace** `pixelroot32::persistence` ·
**Feature gate** `PIXELROOT32_ENABLE_PERSISTENCE`

```cpp
enum class StoreResult : uint8_t {
    Ok,              // completed
    NotFound,        // key/slot holds no record; outputs untouched
    TooLarge,        // data exceeds the caller's buffer or the medium's limit
    NoSpace,         // medium full
    Corrupt,         // envelope, length or CRC check failed
    IoError,         // unclassified medium failure
    Unavailable,     // no store bound, or medium not initialised
    InvalidKey,      // key/namespace breaks isValidKey()
    InvalidArgument  // null pointer, out-of-range slot, other bad parameter
};
inline constexpr size_t kMaxKeyLength = 15;   // the ESP32 NVS limit, applied on EVERY medium
inline bool isValidKey(const char* key);      // 1..15 bytes, each in [A-Za-z0-9_-]; nullptr -> false
```

Every store, adapter and codec call returns a `StoreResult` — never a bare `bool`, never
an exception. Namespaces and slot prefixes follow the same key rule. An invalid key
answers `InvalidKey` before the medium is touched.

### IKeyValueStore — small named values

**Header** `include/persistence/KeyValueStore.h` · **Namespace** `pixelroot32::persistence` ·
**Feature gate** `PIXELROOT32_ENABLE_PERSISTENCE`

```cpp
class IKeyValueStore {
public:
    virtual StoreResult getI32(const char* key, int32_t& out) = 0;
    virtual StoreResult setI32(const char* key, int32_t value) = 0;
    virtual StoreResult getU32(const char* key, uint32_t& out) = 0;
    virtual StoreResult setU32(const char* key, uint32_t value) = 0;
    virtual StoreResult getBlob(const char* key, uint8_t* out, size_t capacity,
                                size_t& stored) = 0;   // capacity == 0 is the size query
    virtual StoreResult setBlob(const char* key, const uint8_t* data, size_t size) = 0;
    virtual StoreResult erase(const char* key) = 0;
    virtual StoreResult commit() = 0;                  // the durability point
};
```

- **One value per key, last write wins.** A set of any type replaces whatever the key
  held, so a typed get on a key that now holds another type answers `NotFound`.
- Missing key → `NotFound`, every output untouched. On a failed set the previous value
  is kept.
- `getBlob()` never writes past `capacity`; a larger blob answers `TooLarge`, writes
  nothing, and sets `stored` to its size. `out` may be `nullptr` only when `capacity == 0`.
- Typed values are `int32_t`, `uint32_t` and blobs only. Strings and floats go through
  the game's own record.
- A set or erase may stay volatile until `commit()`; adapters without a write-back stage
  return `Ok` from `commit()`.

### ISlotStore — numbered save slots

**Header** `include/persistence/SlotStore.h` · **Namespace** `pixelroot32::persistence` ·
**Feature gate** `PIXELROOT32_ENABLE_PERSISTENCE`

```cpp
class ISlotStore {
public:
    virtual uint16_t slotCount() const = 0;        // valid slots: 0..slotCount()-1
    virtual size_t maxRecordSize() const = 0;      // largest record write() accepts
    virtual StoreResult query(uint16_t slot, size_t& stored) = 0;   // used? + size, no read
    virtual StoreResult read(uint16_t slot, uint8_t* out, size_t capacity, size_t& stored) = 0;
    virtual StoreResult write(uint16_t slot, const uint8_t* data, size_t size) = 0;
    virtual StoreResult erase(uint16_t slot) = 0;  // NotFound if already empty
};
```

- Out-of-range slot → `InvalidArgument` before the medium is touched.
- Empty slot → `NotFound` with `stored = 0`. A zero-size record **occupies** its slot.
- **`write()` is replace-or-keep**: after a failed or interrupted write, `read()` returns
  the previous record or the new one, never a mix. The one exception the interface allows
  is a medium that cannot replace atomically: on FAT (`SdSlotStore`) a power loss between
  the remove and the rename can also leave the slot empty (`NotFound`), while the record
  envelope rejects a torn file as `Corrupt`.
- `write()` above `maxRecordSize()` → `TooLarge`. There is no `commit()` on slots.

### RecordCodec — the self-checking envelope

**Header** `include/persistence/RecordCodec.h` · **Namespace** `pixelroot32::persistence` ·
**Feature gate** `PIXELROOT32_ENABLE_PERSISTENCE`

```cpp
uint32_t crc32(const uint8_t* data, size_t size, uint32_t crc = 0);  // CRC-32/IEEE, chainable
struct RecordInfo { uint16_t schemaVersion; size_t payloadSize; };
class RecordCodec {
public:
    static constexpr size_t  kHeaderSize = 16;
    static constexpr uint8_t kEnvelopeVersion = 1;
    static StoreResult encode(uint16_t schema, const uint8_t* payload, size_t size,
                              uint8_t* out, size_t capacity, size_t& written);
    static StoreResult decode(const uint8_t* in, size_t size, const uint8_t*& payload,
                              RecordInfo& info);   // payload points INTO in
};
```

| Bytes | Field |
|---|---|
| 0-3 | Magic, ASCII `P` `R` `S` `T` |
| 4 | Envelope version (`kEnvelopeVersion`, `1`) |
| 5 | Reserved, `0` |
| 6-7 | Schema version (`uint16_t`, game-owned) |
| 8-11 | Payload length (`uint32_t`) |
| 12-15 | CRC-32 over bytes 0..11, then the payload |
| 16.. | Payload |

Little-endian, written byte by byte. CRC-32/IEEE (zlib/PNG: reflected polynomial
`0xEDB88320`, check value `0xCBF43926` for `"123456789"`), 16-entry nibble table = 64 B
of flash. Covering the header means a flipped schema or length field is caught.
`decode()` answers `Corrupt` for a truncated record, wrong magic, unknown envelope
version, non-zero reserved byte, length disagreeing with `size`, or CRC mismatch; outputs
are untouched on failure. **The schema version is reported, never judged.** `encode()`
answers `TooLarge` when `capacity < kHeaderSize + size` and writes nothing.

### PersistenceService — two roles plus record helpers

**Header** `include/persistence/PersistenceService.h` · **Namespace**
`pixelroot32::persistence` · **Feature gate** `PIXELROOT32_ENABLE_PERSISTENCE`

```cpp
class PersistenceService {
public:
    void bindSettings(IKeyValueStore* store);   // nullptr unbinds; store NOT owned
    void bindSaves(ISlotStore* store);          // nullptr unbinds; store NOT owned
    IKeyValueStore& settings();                 // never null: unbound -> NullKeyValueStore
    ISlotStore& saves();                        // never null: unbound -> NullSlotStore (0 slots)
    bool hasSettings() const;  bool hasSaves() const;

    StoreResult saveRecord(uint16_t slot, uint16_t schema, const uint8_t* payload, size_t size,
                           uint8_t* scratch, size_t scratchCapacity);
    StoreResult loadRecord(uint16_t slot, uint8_t* scratch, size_t scratchCapacity,
                           const uint8_t*& payload, RecordInfo& info);
    StoreResult saveSettingRecord(const char* key, uint16_t schema, const uint8_t* payload,
                                  size_t size, uint8_t* scratch, size_t scratchCapacity);
    StoreResult loadSettingRecord(const char* key, uint8_t* scratch, size_t scratchCapacity,
                                  const uint8_t*& payload, RecordInfo& info);
};
```

- **Never returns null.** An unbound role answers through a built-in null store
  (`NullKeyValueStore`, `NullSlotStore`) whose every call is `Unavailable`; the null slot
  store reports `slotCount() == 0` and `maxRecordSize() == 0`. A game keeps running on
  its defaults.
- Save check order: `Unavailable` (role unbound) → `InvalidArgument` (null `scratch`) →
  codec result (`TooLarge` when `scratchCapacity < kHeaderSize + size`; store untouched)
  → the store's `write()`/`setBlob()` result, unchanged.
- Load check order: `Unavailable` → `InvalidArgument` → the store's `read()`/`getBlob()`
  result, unchanged (`NotFound`, `TooLarge`, …) → `Corrupt` when decoding fails.
- **Scratch is caller-owned**; the service holds no buffer and allocates nothing. On
  `Ok`, `payload` points inside `scratch` — copy fields out before `scratch` dies.
- **Never calls `commit()`** — `saveSettingRecord()` may leave the blob volatile until the
  game calls `settings().commit()`. Size: four pointers, 16 B on ESP32.

### SdConfig — microSD mount config

**Header** `include/drivers/esp32/SdConfig.h` · **Namespace**
`pixelroot32::persistence` · **Feature gates** `PIXELROOT32_ENABLE_PERSISTENCE` +
`PIXELROOT32_ENABLE_SD_STORAGE` + `defined(ESP32)`

```cpp
struct SdConfig {
    SPIClass* spi = nullptr;      // required: &SPI, &HSPI, &VSPI, ...
    int sck = -1;                 // -1 = bus default
    int miso = -1;
    int mosi = -1;
    int cs = -1;                  // required on most boards
    uint32_t frequencyHz = 4000000;
};
```

Plain data: the adapter owns no board pin and no board assumption. `-1` means the bus
default for that signal; `spi` and `cs` must be real values. On a CYD-class board the
panel and the card need **different SPI hosts** (the example moves the panel to HSPI);
`SD.begin()` reconfigures the host it is given.

### SdSlotStore — save slots on a microSD card

**Header** `include/drivers/esp32/SdStore.h` · **Namespace**
`pixelroot32::persistence` · **Feature gates** `PIXELROOT32_ENABLE_PERSISTENCE` +
`PIXELROOT32_ENABLE_SD_STORAGE` + `defined(ESP32)`

```cpp
SdSlotStore(const char* rootDir, const char* prefix, uint16_t slots,
            size_t maxRecordBytes) noexcept;   // strings must outlive the object
StoreResult init(const SdConfig& config) noexcept;  // mounts or re-mounts; never formats
```

- **Files**: one per slot at `<rootDir>/<prefix>/<i>.slot` (`2.slot`); a missing file is
  an empty slot. `rootDir` is relative to the Arduino SD mountpoint (`/sd` by default),
  non-empty and starts with `/`; `prefix` must pass `isValidKey()`. Paths are held by
  pointer, so string literals are the intended use; `kSdMaxPathBytes` is `160`.
- **`init(config)`** brings up the bus, mounts the card and creates the directories
  (`mkdir` on an existing directory is not an error). It **never formats, erases or
  repairs** the card. Until it returns `Ok`, every other call answers `Unavailable`, so a
  game boots on defaults when the card is missing (`init()` itself answers
  `InvalidArgument`/`InvalidKey` for bad arguments).
- **Reinsert**: calling `init()` again after the card was removed releases the dead volume
  and mounts again (`SD.end()` first, because `SDFS::begin()` is a no-op while a volume is
  registered), so no reboot is needed. A removal itself is not visible up front: the
  Arduino `SD` library caches the card type at mount, so the first access after pulling
  the card answers `IoError` (or `NotFound` for a missing file).
- **Errors**: missing file `NotFound`; full card `NoSpace`; a file that fails to open or
  read, or any other I/O failure, `IoError` (never a silent `NotFound`); uninitialised
  store `Unavailable`; out-of-range slot or null buffer with non-zero capacity
  `InvalidArgument`; record above `maxRecordSize()` `TooLarge`. A torn but readable file
  opens `Ok` and is rejected later by the codec as `Corrupt`.
- **No `commit()`**: every write is flushed and in place.
- **Replace-or-keep on FAT**: write to `<file>.tmp`, flush (`fsync`), close, remove the old
  file, `rename` the temp. A power loss keeps the old record, leaves the new one, leaves a
  torn record the codec rejects as `Corrupt`, or — between the removal and the rename —
  leaves an empty slot (`NotFound`).
- **Timestamps**: the adapter never reads one. FAT has no date before 1980, so card files
  show 1980 until the game calls `configTime()` or copies the time from an RTC.
- **Memory**: no heap of its own, fixed stack buffers per call; the FATFS driver the
  Arduino `SD` library brings in costs several KB of RAM.

### Adapters

| Adapter | Header | Extra guard | Medium | `commit()` |
|---|---|---|---|---|
| `Esp32NvsKeyValueStore`, `Esp32NvsSlotStore` | `include/drivers/esp32/Esp32NvsStore.h` | `defined(ESP32)` | One ESP-IDF NVS namespace | `nvs_commit` |
| `SdSlotStore` | `include/drivers/esp32/SdStore.h` | `PIXELROOT32_ENABLE_SD_STORAGE && defined(ESP32)` | One FAT file per slot | — (no `commit()`) |
| `HostFileKeyValueStore`, `HostFileSlotStore` | `include/drivers/native/HostFileStore.h` | `defined(PLATFORM_NATIVE)` | One file per key / slot | `Ok` (writes already in place) |
| `MemoryKeyValueStore<Entries, BlobBytes>`, `MemorySlotStore<Slots, RecordBytes>` | `include/persistence/MemoryStores.h` | — | Fixed in-RAM tables | Counts, `Ok` |

All live in `pixelroot32::persistence` behind `PIXELROOT32_ENABLE_PERSISTENCE`; the SD row
also needs `PIXELROOT32_ENABLE_SD_STORAGE` and an ESP32 target.

```cpp
// ESP32 NVS — namespace/prefix must pass isValidKey(); limits must be non-zero
Esp32NvsKeyValueStore(const char* nameSpace, size_t maxBlobBytes,
                      const char* partition = kNvsDefaultPartition) noexcept;   // "nvs"
Esp32NvsSlotStore(const char* nameSpace, const char* prefix, uint16_t slots,
                  size_t maxRecordBytes, const char* partition = kNvsDefaultPartition) noexcept;
StoreResult init() noexcept;           // both; Ok also when already open; NEVER erases
size_t maxBlobSize() const noexcept;   // KV only

// microSD — ESP32 only; needs both flags; strings must outlive the object
SdSlotStore(const char* rootDir, const char* prefix, uint16_t slots,
            size_t maxRecordBytes) noexcept;
StoreResult init(const SdConfig& config) noexcept;   // mounts or re-mounts; never formats

// Native host files — strings are copied; init() creates the directories
HostFileKeyValueStore(const char* rootDir, const char* nameSpace, size_t maxBlobBytes) noexcept;
HostFileSlotStore(const char* rootDir, const char* prefix, uint16_t slots,
                  size_t maxRecordBytes) noexcept;
StoreResult init() noexcept;
void setFailBeforeReplace(bool fail) noexcept;   // test hook: crash between temp write and move

// Memory — no init(), no heap; test hooks outside the interfaces
void setFailWrites(bool fail);                      // later sets/erases/commit -> IoError
StoreResult corruptByte(const char* key, size_t index);   // MemoryKeyValueStore
StoreResult corruptByte(uint16_t slot, size_t index);     // MemorySlotStore
uint32_t writes() const;   uint32_t commits() const;      // commits(): key-value only
```

- **ESP32 NVS**: integers are NVS `I32`/`U32` items, blobs are NVS blobs. Slot `i` is the
  blob under key `<prefix><i>` (`save0`, `save1`, …); `init()` rejects a prefix whose
  longest key exceeds 15 chars. Until `init()` returns `Ok` every call (commit included,
  even an out-of-range slot) answers `Unavailable` without touching flash. A partition
  NVS cannot use (`NO_FREE_PAGES`, `NEW_VERSION_FOUND`) answers `Corrupt` and is left
  alone — erasing it (`nvs_flash_erase_partition`) wipes every namespace and is the
  game's decision. Not copyable.
- **Host files**: KV files live in `<rootDir>/<nameSpace>/`, slot files in
  `<rootDir>/<prefix>/` (`2.slot`). Uppercase key letters are escaped as `^` + lowercase
  (`Ab_C` → `^ab_^c.kv`) so case-insensitive file systems keep keys distinct. Writes go to
  `<file>.tmp`, then one replace-existing move (`MoveFileExA` on Windows, `rename()` on
  POSIX). A malformed file reads as `IoError`, never a silent `NotFound`. Paths live in
  fixed buffers of `kHostFileMaxPathBytes` (260).
- **microSD**: `rootDir`/`prefix` are held by pointer and must outlive the store. `init()`
  mounts and re-mounts; a failed mount leaves the role unbound and the game on defaults.
  A pulled card is only visible on the next access (`IoError`/`NotFound`). The adapter
  never formats and never reads a timestamp. FAT's non-atomic replace is the documented
  exception to replace-or-keep.

## Wiring a Game

Mirrors `examples/persistence`: the scene owns the stores and the service, picks the
adapter per platform, and binds only the stores whose `init()` succeeded.

```cpp
#include <persistence/PersistenceService.h>
#if defined(ESP32)
#include <drivers/esp32/Esp32NvsStore.h>
#elif defined(PLATFORM_NATIVE)
#include <drivers/native/HostFileStore.h>
#endif
namespace persist = pixelroot32::persistence;

class MyScene : public pixelroot32::core::Scene {
    persist::PersistenceService persistence_;
#if defined(ESP32)
    persist::Esp32NvsKeyValueStore settingsStore_{"mygame", 64};        // blobs <= 64 B
    persist::Esp32NvsSlotStore     saveStore_{"mygame", "save", 3, 256}; // save0..save2
#elif defined(PLATFORM_NATIVE)
    persist::HostFileKeyValueStore settingsStore_{"save_data", "mygame", 64};
    persist::HostFileSlotStore     saveStore_{"save_data", "save", 3, 256};
#endif
};

void MyScene::init() {
    Scene::init();
    if (settingsStore_.init() == persist::StoreResult::Ok) persistence_.bindSettings(&settingsStore_);
    if (saveStore_.init() == persist::StoreResult::Ok)     persistence_.bindSaves(&saveStore_);
    loadSettings();   // NotFound/Corrupt/Unavailable -> keep defaults
}
```

Only the store definitions differ per platform; everything else goes through the service.

On an ESP32 board with a card, keep settings in NVS and move only the save slots to the
card (`examples/persistence`'s `esp32sd` environment):

```cpp
#if defined(ESP32) && PIXELROOT32_ENABLE_SD_STORAGE
#include <drivers/esp32/SdConfig.h>
#include <drivers/esp32/SdStore.h>

persist::Esp32NvsKeyValueStore settingsStore_{"mygame", 64};
persist::SdSlotStore           saveStore_{"/pr32", "save", 3, 256};
const persist::SdConfig        sdConfig_{&SPI, -1, -1, -1, /* cs */ 5, 4000000};

void MyScene::init() {
    Scene::init();
    if (settingsStore_.init() == persist::StoreResult::Ok) persistence_.bindSettings(&settingsStore_);
    if (saveStore_.init(sdConfig_) == persist::StoreResult::Ok) persistence_.bindSaves(&saveStore_);
}
#endif
```

## Schema Versioning

The schema version (`uint16_t`) belongs to the game. The codec stores and reports it; it
never rejects a record for its version and there is **no automatic migration**.

1. Write payloads **field by field** — never `memcpy` a struct (padding and layout differ
   between the PC and the ESP32). Encode multi-byte fields little-endian by hand.
2. On load, gate on **both** `info.schemaVersion` and `info.payloadSize`.
3. To change a layout: bump the constant, keep a branch for every older version still
   accepted (fill new fields with defaults), fall back to defaults otherwise, and re-save
   in the new version at the next safe point.

```cpp
if (r == persist::StoreResult::Ok) {
    if (info.schemaVersion == 2 && info.payloadSize == 4)      { /* current layout */ }
    else if (info.schemaVersion == 1 && info.payloadSize == 3) { /* old: default the new field */ }
    // anything else: defaults
}
```

| Load result | Game reaction |
|---|---|
| `Ok` | Check schema + size, then copy fields out of `payload` |
| `NotFound` | First run: defaults |
| `Corrupt` | Damaged record: defaults (re-save at the next safe point) |
| `TooLarge` | Scratch smaller than the stored record — enlarge it |
| `Unavailable` | No store bound: defaults |
| `IoError`, `NoSpace` | Medium problem: report it, keep running |

## Composition Patterns

**Scene-owned stores (`pixelroot32-scene-manager`).** Construct stores and the service as
scene (or game-wide) members, `init()` + bind in `Scene::init()`, load once there. Stores
must outlive every binding — the service never owns them. A game-wide save system should
live outside any single scene so a scene switch does not destroy the bound stores.

**Settings / options screen (`pixelroot32-ui-system`, `pixelroot32-audio`).** Edit values in
RAM while the menu is open; on leaving the screen do one `saveSettingRecord()` + one
`settings().commit()`, then apply volume/brightness. Never save on every slider tick.

**Settings in NVS, saves on the card (`examples/persistence` `esp32sd`).** Bind the NVS
key-value store for settings and `SdSlotStore` for saves. The card can fail to mount and
the game still runs: settings load from NVS and the saves role stays unbound
(`Unavailable`), so the slot menu shows empty slots instead of crashing.

**Continue / slot menu.** `saves().query(slot, size)` tells used vs empty per slot without
reading; `saveRecord()` / `loadRecord()` for the content; `saves().erase(slot)` for
"Delete".

**Single counters.** A lone high score can skip the envelope:
`settings().setU32("hiscore", v)` + `commit()`. Anything with several fields gets a record.

**Testing (`pixelroot32-testing`).** Bind `MemoryKeyValueStore<8, 64>` /
`MemorySlotStore<3, 256>` to the same service the game uses; `setFailWrites(true)` proves
the game survives `IoError`, `corruptByte()` proves it falls back on `Corrupt`, and
`writes()` / `commits()` pin that it saves — and commits — exactly when it should. The
engine's suites run with `pio test -e native_test_persistence`
(`test_persist_codec`, `_contract`, `_service`, `_hostfile`, `_flags`). NVS is exercised on
the board through `tools/persist_nvs_smoke`; the microSD adapter has no native test
environment (ESP32 only), so its shared contract suite and the hot-swap path run on
hardware through `tools/persist_sd_smoke` (`pio test -e smoke` for the contract suite,
`pio run -e smoke -t upload` for the hot swap), with `examples/persistence`'s `esp32sd`
build demonstrating the adapter end to end.

## ESP32 Constraints

- **NVS writes stall both cores.** Any NVS write or page erase disables the flash cache on
  both cores: render loop, non-IRAM audio code and memory-mapped flash reads stall until it
  ends. Measured on the CYD (Arduino-ESP32 2.0.17 / IDF 4.4.7): `setBlob` 8–64 B ≈ 4 ms
  mean, up to ≈ 13 ms; `commit()` ≈ 12 µs (max 21 µs). NVS has no write-back cache — the
  cost is in the set/erase, not the commit. One set ≈ a quarter of a 60 FPS frame on
  average, most of one at worst.
- **Save at safe points only**: a menu, pause screen, "Saving…" screen or level
  transition, triggered by a discrete event. Never per frame, never from an audio
  callback or timer, never in the steady-state `update()`/`draw()` path.
- **Static RAM**: `PersistenceService` 16 B, `Esp32NvsKeyValueStore` 24 B,
  `Esp32NvsSlotStore` 28 B; no heap on the device path. Two NVS KV stores linked on the
  CYD: `.dram0.bss` +48 B, flash +2,760 B text, +680 B rodata. Flag off: 0 B.
- **NVS is for small records** — the CYD's 24 KB `nvs` partition holds about 20 KB of
  data. Large saves belong on the microSD adapter (`SdSlotStore`, 1.16.0); a raw flash
  partition adapter is not shipped.
- **microSD costs**: the FATFS driver the Arduino `SD` library brings in costs several KB
  of RAM, and `SD.begin()` reconfigures the SPI host it is given — keep the display and
  the card on separate hosts (the CYD example moves the panel to HSPI). The adapter
  itself has no heap and holds only two pointers plus the limits. The default clock is
  4 MHz; raise `frequencyHz` only after the wiring supports it.
- **Removable media**: a missing card is `Unavailable` and leaves the saves role unbound,
  so the game boots on defaults. Calling `init()` again re-mounts after reinsertion; a
  removal is only seen on the next access. Never format from engine-style code.
- **`init()` never erases** the partition; an unusable one answers `Corrupt`.
- **`Preferences` compatibility**: a blob written by Arduino `Preferences::putBytes(key, …)`
  in namespace N reads back byte-identically through `getBlob()` over namespace N, and the
  other way round — the migration path: read the old blob once, rewrite it enveloped.
- **`boot_app0.bin` at `0xE000`**: the Arduino PlatformIO build flashes the `otadata`
  image there on every upload. A custom partition table with no `otadata` whose `nvs`
  covers `0xE000` (e.g. `nvs` at `0x9000`, size `0x6000`) loses its last NVS page on every
  reflash. End `nvs` before `0xE000`, add an `otadata` row there, or drop the image with
  `tools/persist_nvs_smoke/keep_nvs.py`. Default tables are unaffected.
- Verified on Arduino-ESP32 2.0.17; Arduino-ESP32 3.x has not been built.

## Gotchas

- **Missing flag = empty headers.** Without `-D PIXELROOT32_ENABLE_PERSISTENCE=1` every
  type is undefined; the NVS header additionally needs `ESP32`, the microSD header needs
  `ESP32` plus `PIXELROOT32_ENABLE_SD_STORAGE`, the host-file header `PLATFORM_NATIVE` —
  include them under matching `#if defined(...)` guards.
- **Forgetting `init()`** on an NVS or host-file store makes every call `Unavailable`.
  Memory stores have no `init()`.
- **`settings().commit()` is your job.** The service never commits.
- **Load scratch must fit the stored record** (`kHeaderSize` + payload), or the load is
  `TooLarge`; the save needs `kHeaderSize + size`.
- **`payload` is a pointer into `scratch`** — dangling once the scratch array goes out of
  scope.
- **Cross-type writes on NVS are not atomic**: changing a key from integer to blob (or
  back) erases the old item first; a power loss in between loses the key. Never change a
  key's type.
- **A typed get on a key holding another type is `NotFound`**, not a conversion.
- **String lifetimes on ESP32**: the NVS namespace/partition strings must stay valid until
  `init()` has run, and `Esp32NvsSlotStore`'s prefix must outlive the object — use string
  literals. Host-file stores copy their strings.
- **Share no namespace** with unrelated data, and keep `<prefix><slots-1>` within 15 chars.
- **Host-file POSIX path** (`rename`, `fsync`, `ENOSPC`) runs in the Linux CI
  `native_test_persistence` suite and builds on macOS since 1.16.0; Windows remains the
  platform with the most mileage.
- **microSD strings must outlive the store**: `rootDir` and `prefix` are held by pointer;
  use string literals, not locals. `rootDir` must start with `/`.
- **A pulled card is not detected up front**: the first access answers `IoError`/
  `NotFound`; call `init()` again to re-mount.
- **FAT timestamps start at 1980**: call `configTime()` or copy the RTC time if the game
  shows dates.
- **FAT replace is not atomic**: a power loss in the remove-then-rename window can leave
  the slot empty (`NotFound`), not only the old or new record.
- **Never format the card** and never share the SPI host with the display unless the board
  wires it that way.
- **Not thread-safe.** Every call comes from the game-loop task; a game saving from
  another task must serialise access itself.

## Common Patterns

```cpp
// Save a settings record at a safe point (options screen closed)
constexpr uint16_t kSettingsSchema = 1;
persist::StoreResult saveSettings(uint8_t volume, uint8_t brightness, bool invertY) {
    const uint8_t payload[3] = {volume, brightness, uint8_t(invertY ? 1 : 0)};
    uint8_t scratch[persist::RecordCodec::kHeaderSize + sizeof(payload)];
    const auto r = persistence_.saveSettingRecord("cfg", kSettingsSchema, payload,
                                                  sizeof(payload), scratch, sizeof(scratch));
    return r != persist::StoreResult::Ok ? r : persistence_.settings().commit();
}

// Load with defaults on anything but a known schema
uint8_t scratch[persist::RecordCodec::kHeaderSize + 32];
const uint8_t* p = nullptr;  persist::RecordInfo info{};
if (persistence_.loadSettingRecord("cfg", scratch, sizeof(scratch), p, info) == persist::StoreResult::Ok
    && info.schemaVersion == kSettingsSchema && info.payloadSize == 3) {
    volume_ = p[0];  brightness_ = p[1];  invertY_ = p[2] != 0;
}

// Save slot: little-endian u16 written by hand, no separate commit for slots
const uint8_t slot[3] = {level_, uint8_t(presses_ & 0xFF), uint8_t(presses_ >> 8)};
uint8_t slotScratch[persist::RecordCodec::kHeaderSize + sizeof(slot)];
persistence_.saveRecord(0, kSlotSchema, slot, sizeof(slot), slotScratch, sizeof(slotScratch));

// Slot menu: used or empty, without reading
size_t size = 0;
const bool used = persistence_.saves().query(i, size) == persist::StoreResult::Ok;
```

## Agent Constraints

- **Never write persistence code without `-D PIXELROOT32_ENABLE_PERSISTENCE=1`** in the
  game's `platformio.ini` (default `0`), add `-D PIXELROOT32_ENABLE_SD_STORAGE=1` only for
  the microSD slot store on ESP32, and guard adapter includes with `defined(ESP32)` /
  `defined(PLATFORM_NATIVE)`.
- **microSD is ESP32-only and needs both flags.** Never enable
  `PIXELROOT32_ENABLE_SD_STORAGE` without `PIXELROOT32_ENABLE_PERSISTENCE=1`; the build
  fails with a named `#error` otherwise. Never format or erase the card, and never assume
  a timestamp.
- **Never invent APIs.** There is no `IFileStore`, no float/string/`int64` getter or
  setter, no automatic migration, no auto-save/background flush, no `Engine` accessor for
  persistence, no `PersistenceService::commit()`, no EEPROM/raw-partition adapter, and no
  limit macros (capacities are constructor/template arguments). The microSD adapter is
  `SdSlotStore`, ESP32 only, behind `PIXELROOT32_ENABLE_SD_STORAGE`.
- **Never write, erase or commit from the per-frame path**, audio callbacks or timers.
  Saves are discrete, player-driven events at safe points; group the sets of one save and
  `commit()` once.
- **Zero allocation**: scratch buffers are stack or member arrays sized
  `RecordCodec::kHeaderSize + payload`; never `new`/`malloc`/`std::vector`/`std::string`
  for records.
- **Never `memcpy` a struct into a record.** Serialize field by field, little-endian, and
  always store multi-field data through `saveRecord()`/`saveSettingRecord()` with a schema
  version.
- **Handle every `StoreResult`** — load paths fall back to defaults on `NotFound`,
  `Corrupt`, `Unavailable` and an unknown schema; the game must keep running when `init()`
  fails (role left unbound). No exceptions (`-fno-exceptions`), no RTTI.
- **Keys and namespaces**: 1..15 bytes of `[A-Za-z0-9_-]`, one namespace per game, never
  erase the NVS partition from engine-style code — that is an explicit game decision.
- **Cross-reference, do not duplicate**: `pixelroot32-scene-manager` (`Scene::init`
  ownership), `pixelroot32-ui-system` (options/slot menus), `pixelroot32-testing` (Unity,
  `native_test_persistence`), `pixelroot32-memory-optimization` (RAM/flash budgets),
  `pixelroot32-game-scaffold` (`platformio.ini` flags), and
  `pixelroot32-cpp-code-generation` (routing table).
