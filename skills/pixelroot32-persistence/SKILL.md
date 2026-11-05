---
name: pixelroot32-persistence
description: The PixelRoot32 persistence module — IKeyValueStore/ISlotStore storage models, StoreResult error model, RecordCodec 16-byte CRC-32 envelope, PersistenceService roles and record helpers, and the ESP32 NVS, host-file and in-memory adapters. Use when saving settings, high scores, unlocks or save slots, migrating a record schema, or testing save code without hardware.
license: MIT
compatibility: opencode>=0.1.0
metadata:
  domain: engine
  subsystem: persistence
  module: persistence
  feature_gate: PIXELROOT32_ENABLE_PERSISTENCE
  platform: cross-platform
  engine_version: "1.14.0"
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
   |-- HostFileKeyValueStore / HostFileSlotStore   drivers/native  (PLATFORM_NATIVE)
   `-- MemoryKeyValueStore   / MemorySlotStore     persistence/    (any target)
```

Game logic talks only to `IKeyValueStore&` / `ISlotStore&`; which medium sits behind
them is decided once, where the stores are constructed. A new medium (SD, EEPROM) adds an
adapter and touches neither the interfaces nor game code. Adapters move opaque bytes;
`RecordCodec` is the one place a record is validated.

**`Engine` owns none of it.** The game constructs its stores and the service, calls each
adapter's `init()`, and binds the stores at setup. **The engine never writes on its own:**
every set, erase and `commit()` is a call the game makes. Reference implementation:
`examples/persistence` (one settings record + one save slot, every `StoreResult` printed,
same scene on ESP32 and native).

**One flag gates the whole module**: `PIXELROOT32_ENABLE_PERSISTENCE`, default `0`. With
it off every header body compiles to nothing — no symbols, no `Engine` bytes, no `Scene`
vtable change. There are no other macros: capacities are constructor arguments or
template parameters. Mirrors: `pixelroot32::platforms::config::EnablePersistence`
(`include/platforms/EngineConfig.h`) and `pixelroot32::modules::Persistence`
(`include/core/EngineModules.h`).

```ini
build_flags =
    -D PIXELROOT32_ENABLE_PERSISTENCE=1
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
  the previous record or the new one, never a mix.
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

### Adapters

| Adapter | Header | Extra guard | Medium | `commit()` |
|---|---|---|---|---|
| `Esp32NvsKeyValueStore`, `Esp32NvsSlotStore` | `include/drivers/esp32/Esp32NvsStore.h` | `defined(ESP32)` | One ESP-IDF NVS namespace | `nvs_commit` |
| `HostFileKeyValueStore`, `HostFileSlotStore` | `include/drivers/native/HostFileStore.h` | `defined(PLATFORM_NATIVE)` | One file per key / slot | `Ok` (writes already in place) |
| `MemoryKeyValueStore<Entries, BlobBytes>`, `MemorySlotStore<Slots, RecordBytes>` | `include/persistence/MemoryStores.h` | — | Fixed in-RAM tables | Counts, `Ok` |

All live in `pixelroot32::persistence` behind `PIXELROOT32_ENABLE_PERSISTENCE`.

```cpp
// ESP32 NVS — namespace/prefix must pass isValidKey(); limits must be non-zero
Esp32NvsKeyValueStore(const char* nameSpace, size_t maxBlobBytes,
                      const char* partition = kNvsDefaultPartition) noexcept;   // "nvs"
Esp32NvsSlotStore(const char* nameSpace, const char* prefix, uint16_t slots,
                  size_t maxRecordBytes, const char* partition = kNvsDefaultPartition) noexcept;
StoreResult init() noexcept;           // both; Ok also when already open; NEVER erases
size_t maxBlobSize() const noexcept;   // KV only

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
(`test_persist_codec`, `_contract`, `_service`, `_hostfile`, `_flags`); NVS is exercised on
the board through `tools/persist_nvs_smoke`.

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
  data. Large saves need a future raw-partition/SD adapter (not shipped).
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
  type is undefined; the NVS header additionally needs `ESP32`, the host-file header
  `PLATFORM_NATIVE` — include them under matching `#if defined(...)` guards.
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
- **Host-file POSIX path** (`rename`, `fsync`, `ENOSPC`) has not yet been run outside
  Windows.
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
  game's `platformio.ini` (default `0`), and guard adapter includes with
  `defined(ESP32)` / `defined(PLATFORM_NATIVE)`.
- **Never invent APIs.** There is no `IFileStore`, no float/string/`int64` getter or
  setter, no automatic migration, no auto-save/background flush, no `Engine` accessor for
  persistence, no `PersistenceService::commit()`, no SD/EEPROM/raw-partition adapter, and
  no limit macros (capacities are constructor/template arguments).
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
