---
name: pixelroot32-networking
description: The PixelRoot32 ESP-NOW multiplayer module — NetSession discovery and reliable delivery, the Scene::onNetworkEvent hook, NetTransport with its ESP32EspNowTransport and LoopbackTransport backends, the NetCounter convergent shared value, and tagged custom messages. Use when writing router-free peer-to-peer multiplayer, sending game messages between boards, sharing a score, or testing netcode on native.
license: MIT
compatibility: opencode>=0.1.0
metadata:
  domain: engine
  subsystem: networking
  module: networking
  feature_gate: PIXELROOT32_ENABLE_NETWORKING
  platform: cross-platform
  engine_version: "1.14.0"
---

## Overview

`pixelroot32::networking` is a **router-free peer-to-peer link over ESP-NOW**. The radio
runs as a Wi-Fi station that never joins an access point: no router, no IP, no DHCP, no
sockets. Every peer must sit on the **same channel (1..13)**. The module is layered and
every layer is borrowed, never owned, by the one above it:

| Layer | Owns | Never does |
|---|---|---|
| `NetTransport` (`ESP32EspNowTransport` / `LoopbackTransport`) | link, channel, peer addresses, staging rings | parse headers, sequence, ACK, retry |
| `NetSession` | HELLO/BYE discovery, fixed peer table, offline timeout, reliable delivery (ACK, bounded retry, 64-bit duplicate window, duplicate re-ACK), event queue | own or shut down the transport, interpret payloads |
| `Engine` + `Scene::onNetworkEvent` | one `session.update()` per frame and event dispatch | own, init or shut down the session |
| `NetCounter` | convergent signed accumulator, snapshot resync | touch the transport |

**One flag gates everything**: `PIXELROOT32_ENABLE_NETWORKING`, default `0`. Off, the module
contributes no symbols, no `Engine` bytes and no `Scene` vtable slot. On, it is **only
supported on ESP32 and native** — any other target hits a named `#error`. ESP-NOW and `WiFi`
come from the Arduino-ESP32 core: no `lib_deps`. Reference projects: `examples/esp_now`
(shared counter), `examples/esp_now_dice` (commit/reveal), `examples/esp_now_ping` (tagged
custom messages, application RTT).

**Scope: convergence, not lockstep.** The same applied operations give the same value on
every peer whatever the arrival order; two screens may differ for a few frames while an
operation is in flight.

## Key APIs

### NetTypes — addresses, results, events

**Header** `include/networking/NetTypes.h` · **Namespace** `pixelroot32::networking` ·
**Feature gate** `PIXELROOT32_ENABLE_NETWORKING`

```cpp
inline constexpr uint8_t kNetAddressSize = 6;
struct NetAddress { uint8_t bytes[kNetAddressSize]{};          // MAC, trivially copyable
    bool operator==(const NetAddress&) const noexcept;  bool operator!=(...) const noexcept;
    bool isBroadcast() const noexcept;                          // all 0xFF
    bool isZero() const noexcept; };                            // all 0x00 — never a valid peer
enum class NetResult : uint8_t { Ok = 0, NotInitialized, InvalidArgument, PayloadTooLarge,
    PeerNotFound, PeerTableFull, NoMemory, Busy, TransportError, Unsupported };
enum class NetDelivery : uint8_t { Unreliable = 0, Reliable = 1 };
enum class NetEventType : uint8_t { Message = 0, PeerJoined = 1, PeerLeft = 2, SendFailed = 3 };
struct NetEvent {                       // 76 B at the default 64 B payload; never allocates
    NetEventType type; uint8_t peerId; uint8_t nodeId; uint32_t sequence; uint16_t length;
    uint8_t payload[platforms::config::NetMaxPayload]{};       // inline copy
};
struct NetPeerInfo { NetAddress address; uint8_t nodeId; uint16_t bootId; uint16_t rttMs;
                     uint32_t lastSeenMs; uint32_t uptimeMs; bool online; };
struct NetStats { uint32_t txMessages, rxMessages, txRetries, txFailed, rxDropped,
                  rxDuplicates, txBusy; uint16_t lastRttMs; };   // monotonic — difference two samples
```

### NetSession — discovery, peer table, reliability

**Header** `include/networking/NetSession.h` · **Namespace** `pixelroot32::networking` ·
**Feature gate** `PIXELROOT32_ENABLE_NETWORKING`

```cpp
struct NetSessionConfig {
    uint8_t  channel = 1;          // 1..13, identical on every peer
    uint8_t  nodeId = 0;           // application label; 0 derives it from the MAC
    uint16_t bootId = 0;           // 0 generates one per boot
    bool     discovery = true;     // false: no HELLO; only addPeer()-seeded addresses get in
    uint16_t helloIntervalMs = platforms::config::NetHelloIntervalMs;  // 1000
    uint16_t peerTimeoutMs   = platforms::config::NetPeerTimeoutMs;    // 3000
    uint16_t ackTimeoutMs    = platforms::config::NetAckTimeoutMs;     // 20
    uint8_t  maxRetries      = platforms::config::NetMaxRetries;       // 3
};
class NetSession {
public:
    static constexpr uint8_t  kMaxPeers = platforms::config::NetMaxPeers;
    static constexpr uint16_t kRxQueueCapacity = platforms::config::NetRxQueueCapacity;
    explicit NetSession(NetTransport& transportRef);  // non-copyable, non-movable; ref must outlive it
    NetResult init(const NetSessionConfig& config);   // transport must already be ready
    void shutdown();                                  // best-effort BYE; transport left running
    void update(uint32_t nowMs);                      // the session's ONLY clock; drains <= 8 frames
    NetResult send(uint8_t peerId, NetDelivery delivery, const void* data, uint16_t length);
    NetResult addPeer(const NetPeerConfig& peer);     // pre-seed: stored offline, no event
    bool pollEvent(NetEvent& out);                    // FIFO
    bool peerInfo(uint8_t peerId, NetPeerInfo& out) const;
    uint8_t peerCount() const;                        // online peers only
    const NetStats& stats() const;
    uint8_t pendingCount() const;                     // reliable messages awaiting ACK
    uint8_t localNodeId() const;  uint16_t localBootId() const;
};
```

`init()` validates before touching the link: `NotInitialized` when the transport is not
ready, `InvalidArgument` for a channel outside 1..13 or a zero interval/timeout/retry limit.
It applies the channel to the transport and registers broadcast when discovery is on.
`addPeer()` entries survive a re-init; pending events do not.

**`send()` results** — `Ok` means *the session accepted it*, not that the peer received it:

| Result | Meaning |
|---|---|
| `Ok` | Reliable: queued (maybe still waiting for transport room). Unreliable: handed to the transport. |
| `Busy` | Reliable pending queue full, **or** an unreliable send found no transport room (nothing queued). |
| `PeerNotFound` | Slot exists but the peer is offline — data never creates or revives a peer. |
| `PayloadTooLarge` | `length > NetMaxPayload`. |
| `InvalidArgument` | Out-of-range or unused slot, or null `data` with non-zero `length`. |
| `NotInitialized` | Before `init()`. |

**Reliable delivery**: each peer has a monotonic `uint32_t` sequence (0 skipped on wrap). The
whole encoded frame is kept in the pending queue, so a retry is byte-identical. After
`ackTimeoutMs` it retries; after `maxRetries` retransmissions it is dropped, `txFailed`
increments and a `SendFailed` event is queued. A busy **transport** does not fail a reliable
send — the entry stays unsent with its sequence and is attempted on a later `update()`,
without spending retry budget. The receiver keeps `rxHighestSeq` plus a **64-bit window** of
older sequences (serial arithmetic); a duplicate is **not delivered again but is ACKed
again**, because the lost frame may have been the ACK. A new `bootId` resets that peer's
window and fails its pending messages. Reliable is **effectively-once, not ordered**.

### Scene hook and Engine dispatch

**Header** `include/core/Scene.h`, `include/core/Engine.h` · **Namespace**
`pixelroot32::core` · **Feature gate** `PIXELROOT32_ENABLE_NETWORKING`

```cpp
// Scene — default no-op; scenes that ignore networking pay only the vtable slot
virtual void onNetworkEvent(const pixelroot32::networking::NetEvent& event);
// Engine — borrows the pointer; never inits, configures or shuts the session down
void setNetworkSession(pixelroot32::networking::NetSession* session);  // nullptr disables
pixelroot32::networking::NetSession* getNetworkSession();
```

Each `Engine::update()`: input → `session->update(currentMillis)` → drain `pollEvent()` into
`onNetworkEvent()` of the scene **current at drain time** → `Scene::update()` → touch. A scene
swap re-routes queued events instead of discarding them (a reliable message may already be
ACKed). With the session registered, a `send()` from `Scene::update()` is correctly stamped.

### NetTransport — the frame-level backend contract

**Header** `include/networking/NetTransport.h` · **Namespace** `pixelroot32::networking` ·
**Feature gate** `PIXELROOT32_ENABLE_NETWORKING`

```cpp
inline constexpr uint16_t kNetTransportFrameSize =
    platforms::config::NetHeaderSize + platforms::config::NetMaxPayload;  // 80 B at defaults, static_assert <= 250
struct NetTransportConfig { uint8_t channel = 1; uint8_t nodeId = 0; bool manageWifi = true; };
struct NetPeerConfig { NetAddress address{}; uint8_t nodeId = 0; bool encrypted = false; };  // encrypted: reserved, ignored
struct NetRxInfo { NetAddress from{}; uint16_t length = 0; uint8_t data[kNetTransportFrameSize]{}; };
struct NetTxResult { NetAddress to{}; bool success = false; };
class NetTransport {   // every method main-loop only; pollRx/pollTxResult never block or allocate
public:
    virtual NetResult init(const NetTransportConfig& config) = 0;   // idempotent
    virtual void shutdown() = 0;                                    // safe if never initialized
    virtual NetResult addPeer(const NetPeerConfig& peer) = 0;
    virtual NetResult removePeer(const NetAddress& address) = 0;
    virtual NetResult send(const NetAddress& to, const uint8_t* frame, uint16_t length) = 0;
    virtual bool pollRx(NetRxInfo& out) = 0;
    virtual bool pollTxResult(NetTxResult& out) = 0;                // strict FIFO
    virtual NetAddress localAddress() const = 0;
    virtual NetResult setChannel(uint8_t channel) = 0;              // InvalidArgument outside 1..13
    virtual bool isReady() const = 0;
};
```

**`ESP32EspNowTransport`** — **Header** `include/drivers/esp32/ESP32EspNowTransport.h` ·
compiled only when `PIXELROOT32_ENABLE_NETWORKING && defined(ESP32)`. Default-constructed,
non-copyable. `init()` claims the radio, brings the station up without an AP (when
`manageWifi`), starts ESP-NOW; returns `Busy` if another instance owns the radio,
`TransportError` when a Wi-Fi/ESP-NOW/MAC step fails, and unwinds on failure. Peers follow the
interface channel, so `setChannel()` moves them without re-registration. ESP-NOW callbacks
copy into `kRxRingCapacity = 4` / `kTxResultRingCapacity = 4` SPSC rings with atomic indices;
a full ring drops the **newest** entry. `ESP_ERR_ESPNOW_NO_MEM` maps to `Busy`.

**`LoopbackTransport` / `LoopbackBus`** — **Header** `include/drivers/native/LoopbackTransport.h` ·
compiled only when `PIXELROOT32_ENABLE_NETWORKING && defined(PLATFORM_NATIVE)`.

```cpp
struct LoopbackFaults {           // applied per destination: loss, then duplication, then reorder
    uint8_t lossPercent = 0, duplicatePercent = 0, reorderPercent = 0;   // 0..100
    uint16_t delayMs = 0;         // added to the bus clock
    bool sendBusy = false;        // every send() returns NetResult::Busy
    uint32_t seed = 0x12345678u;  // xorshift32 — same profile, same decisions
};
class LoopbackBus { public: static constexpr uint8_t kMaxTransports = 8;
    void setNowMs(uint32_t nowMs) noexcept;  uint32_t nowMs() const noexcept;   // the ONLY clock
    void setFaults(const LoopbackFaults& faults) noexcept;   // also reseeds
    const LoopbackFaults& faults() const noexcept;  void reset() noexcept; };
class LoopbackTransport final : public NetTransport { public:
    static constexpr uint8_t kRxQueueCapacity = 8, kTxResultCapacity = 4;
    LoopbackTransport(LoopbackBus& bus, const NetAddress& localAddress); };  // addPeer/removePeer: always Ok (no-op)
```

### NetCounter — convergent shared value

**Header** `include/networking/NetCounter.h` · **Namespace** `pixelroot32::networking` ·
**Feature gate** `PIXELROOT32_ENABLE_NETWORKING`

```cpp
inline constexpr uint8_t  kCounterOpTag = 1, kCounterSnapshotTag = 2, kCounterSnapshotRequestTag = 3;
inline constexpr uint16_t kCounterUptimeTieMs = 1000;      // closer uptimes: lower nodeId exports
inline constexpr uint16_t kCounterResyncTimeoutMs = 3000;  // then isOutOfSync() is raised
inline constexpr uint8_t  kCounterResyncBufferCapacity = 4;// ops buffered during resync, drop-newest
class NetCounter {
public:
    static constexpr uint8_t kMaxSlots = platforms::config::NetMaxPeers;  // local slot included
    NetResult init(NetSession& session);   // always Ok; session must already be initialized
    void shutdown();
    NetResult add(int16_t delta);          // local first, then one Reliable op to every online peer
    void onEvent(const NetEvent& event);   // feed EVERY event, not only Message
    void update(uint32_t nowMs);           // once per frame, before the session drains
    bool isResyncing() const;  bool isOutOfSync() const;
    int32_t value() const;                 // sum of slots, saturates at int32 limits
    uint32_t revision() const;             // count of real value changes
};
```

Idempotency comes from the session's duplicate window, and addition is commutative, so
arrival order never matters. `add()` returns `Busy` while `isResyncing()` (drop or queue the
input), never rolls back a failed send, and `add(0)` sends nothing. On `PeerJoined` the
better-established node (higher uptime, lower `nodeId` within 1 s) sends a snapshot; the
younger node enters resync, buffers incoming ops, imports, and replays them. A reboot faster
than the survivor's peer timeout is covered by the **explicit snapshot request** (tag 3). No
snapshot within 3 s → resume with `isOutOfSync()` raised.

## Messages and Wire Format

`NetProtocol.h` (`kNetMagic = 0x5052`, `kNetHeaderSize = 16`, `kNetMaxFrameSize = 250`,
`kNetMaxAppPayload = 234`, `NetPacketType { Data = 1, Hello = 2, Bye = 3, Ack = 4,
Snapshot = 5 }`) is the session's codec — game code never builds frames. Game messages are
opaque `Data` payloads with **your own one-byte tag**:

| Rule | Why |
|---|---|
| Byte 0 is the message tag | one event stream, routed by tag |
| Tags 1, 2, 3 belong to `NetCounter`; app tags start at 10 | dice uses 10–12, ping uses 16–17 |
| Fixed little-endian fields, explicit encode/decode | padding and endianness never reach the wire |
| Check `event.length` exactly before reading | foreign or corrupt frames never read past the payload |
| Carry a round/tick/version when order or schema matters | no ordering, no schema negotiation |

```cpp
inline constexpr uint8_t kPingTag = 16;               // from examples/esp_now_ping
inline constexpr uint16_t kPingSize = 7;              // tag + u16 seq + u32 stampMs
uint16_t encodePing(uint8_t* out, uint16_t cap, uint16_t seq, uint32_t stampMs) {
    if (out == nullptr || cap < kPingSize) return 0;
    out[0] = kPingTag;
    out[1] = static_cast<uint8_t>(seq & 0xFF);  out[2] = static_cast<uint8_t>(seq >> 8);
    for (uint8_t i = 0; i < 4; ++i) out[3 + i] = static_cast<uint8_t>(stampMs >> (8 * i));
    return kPingSize;
}
uint8_t buf[pixelroot32::platforms::config::NetMaxPayload];
const uint16_t len = encodePing(buf, sizeof(buf), ++seq, nowMs);
if (len != 0) session.send(peerId, NetDelivery::Reliable, buf, len);
```

**Delivery per message**: `Reliable` for things applied exactly once (shot, score, turn,
choice); `Unreliable` for continuous state the next packet supersedes (position, cursor).
There is **no fragmentation**: a message above `NetMaxPayload` needs a bigger flag or
application chunking.

## Configuration

Macros in `include/platforms/PlatformDefaults.h`, mirrored as `constexpr` in
`pixelroot32::platforms::config` (`include/platforms/EngineConfig.h`); module switch
`pixelroot32::modules::Networking` in `include/core/EngineModules.h`.

| Macro | Default | Mirror | Compile-time limit |
|---|---|---|---|
| `PIXELROOT32_ENABLE_NETWORKING` | `0` | `EnableNetworking` | ESP32 or native only (`#error`) |
| `PIXELROOT32_NET_MAX_PEERS` | `4` | `NetMaxPeers` | 1..20 (ESP-NOW hardware limit) |
| `PIXELROOT32_NET_MAX_PAYLOAD` | `64` | `NetMaxPayload` | > 0 and payload + header <= 250 (max 234) |
| `PIXELROOT32_NET_RX_QUEUE_CAPACITY` | `8` | `NetRxQueueCapacity` | power of two >= 2 |
| `PIXELROOT32_NET_TX_QUEUE_CAPACITY` | `8` | `NetTxQueueCapacity` | power of two >= 2, <= 255 |
| `PIXELROOT32_NET_ACK_TIMEOUT_MS` | `20` | `NetAckTimeoutMs` | — |
| `PIXELROOT32_NET_MAX_RETRIES` | `3` | `NetMaxRetries` | >= 1 |
| `PIXELROOT32_NET_PEER_TIMEOUT_MS` | `3000` | `NetPeerTimeoutMs` | — |
| `PIXELROOT32_NET_HELLO_INTERVAL_MS` | `1000` | `NetHelloIntervalMs` | — |
| `PIXELROOT32_NET_PROTOCOL_VERSION` | `1` | `NetProtocolVersion` | wire format; mismatched peers reject |
| `PIXELROOT32_NET_HEADER_SIZE` | `16` | `NetHeaderSize` | must stay 16 (`static_assert`) |

`NetCounter.h` adds one more: `2 + NetMaxPeers * 11 <= NetMaxPayload` (the snapshot must fit
one payload). At the default 64 B payload that caps `NetMaxPeers` at **5** when the counter
is included.

## Composition Patterns

**ESP32 setup order** (`examples/esp_now`): transport → session → counter → Engine.

```cpp
#include <core/Engine.h>
#include <drivers/esp32/ESP32EspNowTransport.h>
#include <networking/NetCounter.h>
#include <networking/NetSession.h>
using namespace pixelroot32::networking;

namespace { ESP32EspNowTransport transport; NetSession session(transport); NetCounter counter; GameScene scene; }

void setup() {
    engine.init();
    NetTransportConfig tc; tc.channel = 1; tc.nodeId = 1; tc.manageWifi = true;
    if (transport.init(tc) != NetResult::Ok) { /* no radio: keep running offline */ }
    NetSessionConfig sc;  sc.channel = 1; sc.nodeId = 1;   // nodeId unique per board
    if (session.init(sc) != NetResult::Ok) { /* bad config or transport not ready */ }
    counter.init(session);
    engine.setScene(&scene);
    engine.setNetworkSession(&session);   // Engine borrows; app still owns shutdown()
}
```

**Scene integration** (see **`pixelroot32-scene-manager`**): route by event type, then tag.

```cpp
void GameScene::update(unsigned long deltaTime) {
    counter.update(static_cast<uint32_t>(millis()));       // once per frame
    if (engine.getInputManager().isButtonPressed(kApplyButton)) {
        (void)counter.add(1);                              // Busy while resyncing: drop or queue
    }
}
void GameScene::onNetworkEvent(const NetEvent& event) {
    counter.onEvent(event);                                 // every event, lifecycle included
    switch (event.type) {
        case NetEventType::PeerJoined: /* map event.peerId -> player, resend its state */ break;
        case NetEventType::PeerLeft:   /* mark offline, cancel waits on that player */   break;
        case NetEventType::SendFailed: /* a reliable message gave up */                 break;
        case NetEventType::Message:    if (event.length > 0) routeByTag(event);         break;
    }
}
```

Use `peerId` (session slot, stable across timeout/revive) for `send()`; use `nodeId` (HELLO
label) for game roles and score signs. Pumping a session by hand keeps the order explicit:
`counter.update(now)` → `session.update(now)` → `while (session.pollEvent(e)) counter.onEvent(e);`.

**Gameplay** (see **`pixelroot32-gameplay-framework`**): a networked turn or round is a
small state machine whose transitions are driven from `onNetworkEvent` and a timeout, as in
`examples/esp_now_dice` (`Committing`/`Revealing` expire after 3 s).

**Testing on native** (see **`pixelroot32-testing`**): two sessions on one `LoopbackBus`,
faults injected, clock advanced explicitly. Run with `pio test -e native_test_networking`
(`native_test` is the flag-off contract and compiles none of these assertions).

```cpp
#include <drivers/native/LoopbackTransport.h>
LoopbackBus bus;
NetAddress a{}, b{};  a.bytes[5] = 1;  b.bytes[5] = 2;     // unique, non-zero, non-broadcast
LoopbackTransport ta(bus, a), tb(bus, b);
NetSession sa(ta), sb(tb);
NetTransportConfig tc;  ta.init(tc);  tb.init(tc);
NetSessionConfig cA; cA.nodeId = 1;  NetSessionConfig cB; cB.nodeId = 2;
sa.init(cA);  sb.init(cB);
LoopbackFaults f;  f.lossPercent = 30;  f.duplicatePercent = 20;  f.reorderPercent = 20;
f.delayMs = 5;  f.seed = 42;  bus.setFaults(f);
for (uint32_t t = 0; t < 5000; t += 10) { bus.setNowMs(t); sa.update(t); sb.update(t); /* poll + assert */ }
```

One `Engine` pumps one session: the native `examples/esp_now` registers the primary session
and drives the simulated second node from `Scene::update()`. The loopback proves protocol
and logic, not RF range, interference or Wi-Fi timing.

## ESP32 Constraints

- **Frame**: ESP-NOW moves at most 250 B; header 16 B; default payload 64 B (80 B frame);
  hard application ceiling 234 B. Every queue entry carries the payload inline, so RAM scales
  linearly with `NET_MAX_PAYLOAD`.
- **Peers**: default table 4, hardware ceiling 20 (6 encrypted, unused in v1). Slots are
  never freed — a timed-out peer keeps its index.
- **Queues**: session RX events 8, reliable pending 8; ESP32 staging rings 4 RX / 4 TX
  results. All drop-newest; a full pending queue makes `send()` return `Busy`.
- **Static RAM**: the module budgets ~2 KB at defaults (`NetEvent` 76 B × 8 ≈ 608 B,
  pending frames ≈ 672 B, peer table ≈ 128 B, ESP32 staging ≈ 288 B, counter ≈ 56 B).
  Engine bench on esp32dev: **+1,800 B RAM / +4,836 B flash** for the module alone.
- **Wi-Fi stack**: `ESP32EspNowTransport` pulls in the Wi-Fi/lwIP stack (**~580 KB flash**,
  excluded from that bench). `examples/esp_now` on a C3 is 42,492 B RAM / 749,824 B flash
  example-wide. On a single-core C3 disable audio, particles, physics and UI if unused.
- **Arduino-ESP32**: 2.0.17 and 3.3.12 verified; a compile-time shim
  (`ESP_ARDUINO_VERSION_MAJOR >= 3`) covers the 3.x callback signatures.
- **Main loop cost**: `update()` drains at most 8 frames per call; ACK timeout 20 ms fits
  3 retries inside a 60 ms stall.

## Gotchas

- **No encryption, no authentication in v1.** Every peer is registered unencrypted and
  `NetPeerConfig::encrypted` is ignored. With discovery on, anyone in range with a free slot
  joins via HELLO. Locked deployments: `discovery = false` + `addPeer()` on both sides (still
  unencrypted). Never send secrets.
- **Only ESP32 and native targets.** Enabling the flag elsewhere fails the build.
  `ESP32EspNowTransport` exists only under `ESP32`, `LoopbackTransport` only under
  `PLATFORM_NATIVE` — guard includes per target.
- **Callbacks run in the Wi-Fi task**, not the main loop. They only copy into the staging
  rings and bump drop counters; frames reach the session through `pollRx()`. Never call
  session, counter or scene code from an ESP-NOW callback, and never call any networking
  method off the main loop — the session queue is deliberately not atomic.
- **Channel mismatch is silent.** Both boards run fine and show 0 peers. Never join an AP:
  the association moves the channel and drops the link.
- **`nodeId` collisions are not detected** — the MAC is the identity. Give each board a
  unique, compile-time `nodeId` and rebuild both when one changes.
- **`Ok` from `send()` is acceptance, not delivery.** Handle `SendFailed`.
- **`send()` before the first `session.update()`** stamps the pending entry at 0, skewing the
  first ACK timeout. Engine-registered sessions are already updated before `Scene::update()`.
- **Data never creates a peer.** Messages from an unknown MAC are dropped; `send()` to an
  offline slot is `PeerNotFound`.
- **One `NetCounter` per session** — its tags 1–3 are fixed, so a second instance would eat
  the first one's operations.
- **Raising `NET_MAX_PEERS` above 5 with the default payload breaks the `NetCounter`
  snapshot `static_assert`** — raise `NET_MAX_PAYLOAD` with it.
- **No persistence.** If both boards reboot the shared value resets to 0; store it yourself
  with the persistence module if needed.
- **One `ESP32EspNowTransport` per process**; a second `init()` answers `Busy`.
- **The flag changes header contents** — define it for the whole library build, not one file.

## Common Patterns

```cpp
// Shared score with a sign convention (examples/esp_now_dice): value() holds A - B
const bool isA = session.localNodeId() < otherNodeId;
counter.add(isA ? +1 : -1);            // winner moves the shared score once per round
if (counter.revision() != lastRevision) { lastRevision = counter.revision(); redraw(); }
if (counter.isOutOfSync()) { /* surface "desync" — value may have diverged */ }

// Commit/reveal round (examples/esp_now_dice): no peer can pick after seeing the other
// 1. send Reliable Commit{round, commitment(seed)}  2. after both commits, send Reveal{round, seed}
// 3. verify reveal against commitment (mismatch -> CHEAT), derive the result from
//    (seedA, seedB, round) on both boards — never transmit the result itself
// 4. 3 s in Committing/Revealing -> TIMEOUT; PeerLeft -> back to Idle
// The dice commitment is a 32-bit mix, NOT cryptographic.

// Automatic reply (examples/esp_now_ping): answer the sender's slot, never a guessed one
case NetEventType::Message:
    if (event.length == kPingSize && event.payload[0] == kPingTag) {
        session.send(event.peerId, NetDelivery::Reliable, reply, replyLen);
    }
    break;

// Locked pairing: no discovery, both sides pre-seeded
NetSessionConfig sc; sc.discovery = false;  session.init(sc);
NetPeerConfig peer; peer.address = kOtherBoardMac; peer.nodeId = 2;  session.addPeer(peer);
```

## Agent Constraints

- **Never write networking code without `-D PIXELROOT32_ENABLE_NETWORKING=1`** in the
  target environments only; the default build stays at `0`. Never enable it for a target
  other than ESP32 or native.
- **Never invent APIs.** There is no `NetSession::broadcast()`, no `connect()`/`listen()`,
  no sockets/IP/TCP/UDP, no encryption or `setKey()`, no ordered channel, no fragmentation,
  no RPC or object replication, no generic serializer, no `Engine`-owned session, no
  transport-level wire `Snapshot` used by v1 (the counter snapshot is an app tag).
- **Ownership is explicit.** The app owns transport, session and counter as static/global
  objects; init order is transport → session → counter → `setNetworkSession()`, and the app
  calls their `shutdown()`. Never construct them per frame or on the heap.
- **Zero allocation in the loop**: payload buffers are stack/static fixed arrays of at most
  `platforms::config::NetMaxPayload`; no `new`, `malloc`, `std::vector`, `std::string` or
  `std::function` in netcode.
- **Every fallible call returns `NetResult`** — check it; never use exceptions (`-fno-exceptions`).
- **Every app message has a tag >= 10, a fixed little-endian layout, an exact length check
  before reading, and a round/version when order or schema matters.** Never `memcpy` a
  struct onto the wire.
- **Feed every event to `NetCounter::onEvent()`**, call `counter.update()` once per frame,
  and treat `add()` → `Busy` as "resyncing", not an error.
- **Main loop only.** No networking calls from ISRs, ESP-NOW callbacks or other tasks.
- **C++17, PascalCase types, camelCase methods, no `m_` prefixes**, Doxygen in headers.
- **Cross-reference, do not duplicate**: `pixelroot32-scene-manager` (`Scene` lifecycle,
  `onNetworkEvent` placement), `pixelroot32-gameplay-framework` (round/turn state machines),
  `pixelroot32-testing` (`native_test_networking`, Unity), `pixelroot32-memory-optimization`
  (RAM budgets on C3), `pixelroot32-cpp-code-generation` (routing table, naming).
