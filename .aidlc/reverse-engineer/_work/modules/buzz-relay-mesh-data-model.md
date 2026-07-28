## Module: buzz-relay-mesh (`crates/buzz-relay-mesh`)
### Aspect: Data Model

Scope: the 10 files of `crates/buzz-relay-mesh` (3,169 LOC). No SQL, no migrations,
no ORM — this crate's persistence surface is **one Redis key namespace** plus
in-memory tables. All types below verified by reading the source.

---

#### 1. Identity: `RuntimeId`

| Item | Location | Definition |
|---|---|---|
| `RuntimeId(pub [u8; 32])` | `wire.rs:62` | ed25519 public key of a **boot-unique** mesh keypair generated at process start |
| `to_hex()` | `wire.rs:65` | lowercase hex, 64 chars |
| `Debug` | `wire.rs:70-74` | truncates to `RuntimeId(xxxxxxxx…)` (first 8 hex chars) |
| `Display` | `wire.rs:76-80` | full 64-char hex — this is what lands in Redis keys and JSON |
| Derives | `wire.rs:61` | `Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize` |

Generation site: `SecretKey::generate()` → `runtime_id_from_public_key(secret_key.public())`
(`endpoint.rs:20`, `endpoint.rs:38`, `endpoint.rs:96-98`). The reverse map is
`public_key_from_runtime_id` (`endpoint.rs:100-102`).

**Deliberate non-identity** (documented at `wire.rs:52-60`): the RuntimeId is *not*
the deployment's Nostr/secp256k1 relay key, because the helm chart shares one
`BUZZ_RELAY_PRIVATE_KEY` across all pods — reusing it would collapse the ownership
plane. Verified: nothing in the crate derives a RuntimeId from a Nostr key.

The tuple field is `pub` (`wire.rs:62`), so any consumer can construct an arbitrary
`RuntimeId` value without going through iroh. Used that way in tests
(`membership.rs:394`, `runtime.rs:...`), and in `buzz-relay` tests
(`mesh_boot.rs:592`).

---

#### 2. Wire model (serialization codec = **postcard**, not JSON)

Codec: `postcard` v1 with `use-std` (`Cargo.toml:14`, workspace pin
`Cargo.toml:65`). Framing helpers:

| Function | Location | Format |
|---|---|---|
| `wire::encode<T>` | `wire.rs:176-179` | `[WIRE_VERSION: u8]` ++ `postcard(T)` |
| `wire::decode<T>` | `wire.rs:182-188` | version-byte check then `postcard::from_bytes` |
| `gossip::encode_message` | `gossip.rs:62-64` | **bare postcard, NO version byte** |
| `gossip::decode_message` | `gossip.rs:66-77` | postcard, then checks the in-struct `version` field |

Two different versioning schemes coexist: the outer frame uses a leading byte
(`WIRE_VERSION = 1`, `wire.rs:42`); the gossip payload nested inside
`MeshStreamFrame::Gossip{payload}` uses an in-struct `version: u8` field
(`GOSSIP_PAYLOAD_VERSION = 1`, `gossip.rs:13`). Documented as deliberate at
`wire.rs:139-141` ("opaque at this layer so gossip can evolve without a wire bump").

##### 2.1 Wire message type count

**Exactly 5 top-level wire payload types**, 2 of which are envelopes:

| # | Type | Location | Transport |
|---|---|---|---|
| 1 | `MeshDatagram` | `wire.rs:111-122` | one per QUIC datagram, no length prefix |
| 2 | `MeshStreamFrame` | `wire.rs:126-144` | u32-LE length + postcard, on QUIC bi-streams |
| 3 | `StreamHello` | `wire.rs:160-163` | carried inside `MeshStreamFrame::Hello` |
| 4 | `GossipMessage` | `gossip.rs:51-60` | carried inside `MeshStreamFrame::Gossip.payload` |
| 5 | `ReadyRecord` | `registry.rs:99-113` | **JSON** in Redis (`serde_json`, `registry.rs:185`) |

`MeshStreamFrame` has **4 variants** (`wire.rs:128,131,140,144`): `Hello`, `Data`,
`Goodbye`, `Gossip`. `GossipMessage` has **2 variants** (`gossip.rs:52,56`):
`Digest`, `Delta`. `StreamRole` has **2 variants** (`wire.rs:151,154`): `Control`,
`Session`. `Profile` has **3 variants** (`wire.rs:99,101,107`): `ReliableStream`,
`RealtimeMedia`, `HuddleControl`. `GoodbyeReason` has **3 variants**
(`wire.rs:168,170,172`): `SessionEnded`, `Draining`, `StaleGeneration`.

##### 2.2 `FencedHeader` — the fenced tuple

`wire.rs:85-93`: `{ session_id: Uuid, generation: u64, owner_runtime_id: RuntimeId }`.
Derives `Clone, Copy, Debug, PartialEq, Eq, Serialize, Deserialize` (`wire.rs:84`).

Present on `MeshDatagram.fenced`, `MeshStreamFrame::Data.fenced`,
`MeshStreamFrame::Goodbye.fenced`, and `StreamRole::Session.fenced`. **Absent from
`MeshStreamFrame::Gossip`** (`wire.rs:144-146`) and from `StreamRole::Control` —
control-plane traffic is unfenced by construction.

`generation` is documented (`wire.rs:87-89`) as the monotonic lease generation from
the Redis CAS; `owner_runtime_id` is documented as **advisory** (`wire.rs:90-92`).
No code in this crate reads or compares `generation` — the fence checks live in
`buzz-relay` (`tunnel/directory.rs:378,395,413,430`; `audio/join.rs:1081,1201,1888`).

##### 2.3 Size bounds

| Bound | Value | Location | Enforced at |
|---|---|---|---|
| `MAX_STREAM_FRAME` | 16 MiB (`16 * 1024 * 1024`) | `wire.rs:46` | send `peer.rs:142-147`, recv `peer.rs:178-183` |
| datagram max | `Connection::max_datagram_size()` (runtime value) | — | `encode_datagram_checked` `lib.rs:213-226`, `peer.rs:106-110` |
| datagram header overhead budget | ≤ 64 B (test-asserted, not runtime-enforced) | `wire.rs:271-274` | test `datagram_header_overhead_within_budget` |
| ALPN | `b"buzz/mesh/1"` | `wire.rs:37` | checked at `peer.rs:50-55` |

---

#### 3. Membership / peer model

##### 3.1 `GossipRecord` — the per-runtime replicated record

`gossip.rs:16-27`. Derives `Clone, Debug, PartialEq, Serialize, Deserialize`
(`gossip.rs:15`; note: **no `Eq`** — `load: f32`).

| Field | Type | Notes |
|---|---|---|
| `runtime_id` | `RuntimeId` | primary key of the record |
| `endpoint_addrs` | `Vec<String>` | dialable socket addrs as **strings**, parsed lazily at `runtime.rs:328` |
| `proto_version` | `u16` | set by relay to `WIRE_VERSION as u16` (`mesh_boot.rs:367`) |
| `load` | `f32` | advisory load factor; **never written** — see §7 |
| `draining` | `bool` | set only via `begin_drain` (`membership.rs:361-364`) |
| `capabilities` | `Vec<String>` | relay ships a static 3-item list (`mesh_boot.rs:371-377`) |
| `version` | `u64` | per-runtime monotonic; **only the owning runtime increments its own** (`gossip.rs:23-24`) |
| `heartbeat_millis` | `u64` | ms since UNIX_EPOCH, refreshed on every version bump |

Constructor `GossipRecord::new` (`gossip.rs:30-42`) seeds `version = 1`,
`load = 0.0`, `draining = false`, `capabilities = vec![]`, `heartbeat_millis = now_millis()`.

##### 3.2 `PeerState` (private) and the membership table

`membership.rs:20-26`: `{ record: GossipRecord, phi: PhiAccrual, connection_state:
ConnectionState, counters: MeshPeerCounters }`.

`MeshMembership` (`membership.rs:29-43`) is a cloneable handle over shared state:

| Field | Type | Location |
|---|---|---|
| `local_runtime_id` | `RuntimeId` (immutable copy) | `membership.rs:30` |
| `local_record` | `Arc<RwLock<GossipRecord>>` | `membership.rs:31` |
| `peers` | `Arc<RwLock<HashMap<RuntimeId, PeerState>>>` | `membership.rs:32` |
| `draining` | `Arc<AtomicBool>` | `membership.rs:33` |
| `stale_generation_rejections` | `Arc<AtomicU64>` | `membership.rs:34` |
| `foreign_relay_rejections` | `Arc<AtomicU64>` | `membership.rs:35` |
| `expected_relay_pubkey` | `Option<String>` (hex) | `membership.rs:41` — `None` is fail-closed |
| `phi_suspect_threshold` | `f64`, default `8.0` | `membership.rs:42`, const `membership.rs:17` |

**The local record is stored separately from `peers`** and re-appended on every
`records()` call (`membership.rs:195-206`), so `peers.len()` never counts self.

**There is no eviction path.** `grep` for `.remove(`/`retain`/`clear()` in
`membership.rs` returns nothing. Peers only ever transition
`connection_state → Disconnected` (`runtime.rs:275-280`); the entry, its
`endpoint_addrs`, and its counters persist for the process lifetime. See
`buzz-relay-mesh-debt.md` D-02 and `-security.md` S-03.

##### 3.3 `GossipState` — a second, parallel model

`gossip.rs:81-83`: `{ records: HashMap<RuntimeId, GossipRecord> }` with
`new/records/get/update_local/digest/delta_for/apply_delta`
(`gossip.rs:86,92,96,100,111,127,151`).

This is a **complete duplicate** of the scuttlebutt logic that `MeshMembership`
reimplements (`membership.rs:120,166,208,225`). `GossipState` has **zero callers
outside its own module's tests** (verified by grep across `crates/**`). It is the
purer of the two (no `local_runtime_id` special-casing, `apply_delta` returns changed
ids) but nothing uses it. See `-debt.md` D-01.

##### 3.4 Version / conflict-resolution semantics

| Rule | Location |
|---|---|
| Local version bump is `saturating_add(1)` + `heartbeat_millis = now_millis()` | `membership.rs:174-176`; mirror `gossip.rs:105-107` |
| Remote record applied only when `record.version > peer.record.version` | `membership.rs:127-135` (`<=` → `false`) |
| `GossipState` variant | `gossip.rs:154-157` (`is_none_or(|e| record.version > e.version)`) |
| Self-records rejected | `membership.rs:121-123` (`apply_gossip_record`), `membership.rs:87-89` (`apply_ready_records`) |
| Delta selection: send records the remote is missing or behind on | `membership.rs:229-243`; `gossip.rs:127-149` |
| Digest/Delta ordering: sorted by `runtime_id.to_hex()` | `membership.rs:216`, `membership.rs:241`; `gossip.rs:118`, `gossip.rs:144` |

Last-version-wins, no vector clocks, no tie-break on equal versions (equal → ignored).

---

#### 4. Failure detector state: `PhiAccrual`

`gossip.rs:168-172`: `{ samples: Vec<Duration>, last_heartbeat: Option<SystemTime>,
max_samples: usize }`. `Default` = `new(100)` (`gossip.rs:174-178`);
`max_samples` floored at 1 (`gossip.rs:185`).

`observe(at)` (`gossip.rs:189-201`): pushes `at - last_heartbeat` when positive and
non-zero, evicts front via `samples.remove(0)` when over `max_samples`
(`gossip.rs:195-197`, O(n) shift), then sets `last_heartbeat = at`.

`phi_at(now)` (`gossip.rs:203-215`) returns `None` when no heartbeat seen, when
`samples` is empty (i.e. exactly one heartbeat observed), or when `mean <= f64::EPSILON`;
otherwise:

```
phi = (elapsed_secs / mean_secs) / LN_10          // gossip.rs:214
```

**Delta vs the name.** This is *not* the Hayashibara phi-accrual detector: the
sample **variance is never used** (`mean_secs`, `gossip.rs:217-220`, is the only
statistic consumed). The doc comment at `gossip.rs:213` is accurate about what the
code does (`-log10(e^(-elapsed/mean))`), but the type name implies a
distribution-aware detector it does not implement. Concretely, with the default
`DEFAULT_GOSSIP_INTERVAL = 2s` (`runtime.rs:44`) the mean settles near 2s, so
`phi >= 8.0` requires `elapsed >= 8 * ln(10) * 2s ≈ 36.8s` of silence.

`observe` is called from exactly two places, both in `apply_gossip_record`
(`membership.rs:132`, `membership.rs:139`) — phi advances **only when a strictly
newer record arrives**, never from raw connection liveness.

---

#### 5. Redis registry model

| Constant | Value | Location |
|---|---|---|
| `READY_KEY_PREFIX` | `"mesh:ready:"` | `registry.rs:19` |
| key template | `mesh:ready:{runtime_id_hex}` | `ready_key`, `registry.rs:150-152` |
| `DEFAULT_REGISTRY_REFRESH` | 15 s | `registry.rs:20` |
| `REGISTRY_EXPIRY_MULTIPLIER` | 3 | `registry.rs:21` |
| TTL | `refresh * 3`, min 1 s | `expiry_for` `registry.rs:154-156`; `ttl_secs` `registry.rs:187` |
| `ATTESTATION_CONTEXT` | `"buzz-relay-mesh-ready-v1"` | `registry.rs:22` |

Commands used: `SET key payload EX ttl` (`registry.rs:188-194`),
`DEL key` (`registry.rs:201-204`), `SCAN cursor MATCH mesh:ready:* COUNT 100`
+ per-key `GET` (`registry.rs:217-228`). Value encoding is **JSON**
(`serde_json::to_string`, `registry.rs:185`) — the only non-postcard payload in
the crate. Key namespace is **global**: not community/tenant scoped and not
deployment-prefixed; cross-deployment isolation relies entirely on the
`relay_pubkey` anchor check (`membership.rs:90-102`).

##### 5.1 `ReadyRecord`

`registry.rs:99-113`, derives `Clone, Debug, PartialEq, Serialize, Deserialize`
(`registry.rs:98` — no `Eq`).

| Field | Type | Notes |
|---|---|---|
| `runtime_id` | `RuntimeId` | serialized by serde as a 32-byte array |
| `runtime_pubkey` | `String` | **explicit duplicate** of `runtime_id.to_hex()`; cross-checked at `registry.rs:140-145` |
| `relay_pubkey` | `String` | Nostr/secp256k1 hex |
| `relay_sig` | `String` | schnorr signature, `Signature::to_string()` |
| `endpoint_addrs` | `Vec<String>` | |
| `proto_version` | `u16` | |
| `capabilities` | `Vec<String>` | |

`RuntimeAttestation` (`registry.rs:30-34`) is the `{relay_pubkey, relay_sig}` pair
alone; it is embedded field-wise into `ReadyRecord` rather than nested
(`registry.rs:120-130`). `RuntimeAttestation` as a named type has **zero callers
outside `ReadyRecord::new`** (`registry.rs:121`) and its own `verify` method
(`registry.rs:48-50`) has zero callers at all.

Signed preimage (`registry.rs:85-91`) — textual and versioned to avoid JSON
key-order dependence:

```
buzz-relay-mesh-ready-v1\nruntime_pubkey={hex}\nrelay_pubkey={hex}
```

hashed with SHA-256 (`registry.rs:93-96`) then schnorr-signed/verified
(`registry.rs:41`, `registry.rs:78-80`).

##### 5.2 `ReadyHeartbeat`

`registry.rs:280-284`: `{ registry: ReadyRegistry, record: ReadyRecord, published:
bool }`. The `published` bool is the only state machine: `tick(ready)` publishes on
`ready`, clears on the `ready→not-ready` edge (`registry.rs:295-304`), `shutdown()`
clears if published (`registry.rs:306-312`).

`ReadyRegistry` itself (`registry.rs:160-163`) holds `{ pool: deadpool_redis::Pool,
refresh: Duration }` — no cursor, no cache, no local copy of scan results.

---

#### 6. Runtime model

`Inner` (`runtime.rs:65-73`): `{ endpoint: MeshEndpoint, membership: MeshMembership,
registry: Option<ReadyRegistry>, peers: RwLock<HashMap<RuntimeId, PeerEntry>>,
handler: Mutex<Option<Arc<dyn InboundHandler>>>, gossip_interval, reconcile_interval }`.

`PeerEntry` (`runtime.rs:50-55`): `{ peer: MeshPeer, control_tx:
Option<mpsc::Sender<MeshStreamFrame>>, tasks: Vec<JoinHandle<()>> }`.
`control_tx` is `None` on the accept side until a `Hello{Control}` arrives
(`runtime.rs:236-247` vs `runtime.rs:441-448`) — and `gossip_tick_loop` only targets
entries where `control_tx.is_some()` (`runtime.rs:576`).

`MeshRuntime` (`runtime.rs:77-80`): `{ inner: Arc<Inner>, loops:
Arc<Mutex<Vec<JoinHandle<()>>>> }`. **Two peer tables exist**: `Inner.peers`
(live QUIC connections, keyed by RuntimeId) and `MeshMembership.peers`
(gossip records). They are only loosely coupled via `mark_connection_state`
(`runtime.rs:250-252`, `:277-279`, `:314-316`, `:344-346`).

`CONTROL_QUEUE_DEPTH = 64` (`runtime.rs:46`) bounds queued control frames per peer;
overflow drops via `try_send` (`runtime.rs:556`).

`MeshPeer` (`peer.rs:38-43`): `{ _endpoint: iroh::Endpoint, conn:
iroh::endpoint::Connection, runtime_id: RuntimeId, counters: Arc<PeerCountersInner> }`.
`PeerCountersInner` (`peer.rs:19-24`) is 4 `AtomicU64`s snapshotted into
`PeerCounters` (`peer.rs:10-15`: `streams_opened, streams_accepted, datagrams_sent,
datagrams_received`).

**Two disjoint counter models.** `PeerCounters` (per-connection, atomics, in
`peer.rs`) and `MeshPeerCounters` (per-membership-entry, in `status.rs:51-61`) track
overlapping quantities with different field names (`streams_accepted` vs
`streams_received`) and are never reconciled. `MeshPeer::counters()` (`peer.rs:73`)
has zero callers; only the `MeshPeerCounters` set reaches `/_mesh`.

---

#### 7. `MeshStatus` — the `/_mesh` JSON shape

All in `status.rs`, all `Serialize`-only (no `Deserialize` — so nothing can
round-trip or contract-test the endpoint's own output).

```
MeshStatus                                        status.rs:7-15
  enabled: bool                                   status.rs:9   (hardcoded true, membership.rs:352)
  local_runtime_id: String                        status.rs:10  (full 64-hex)
  draining: bool                                  status.rs:11
  peer_count: usize                               status.rs:12
  peers: Vec<MeshPeerStatus>                      status.rs:13  (sorted by runtime_id, membership.rs:298)
  counters: MeshCounters                          status.rs:14

MeshPeerStatus                                    status.rs:17-29
  runtime_id: String, endpoint_addrs: Vec<String> status.rs:19-20
  proto_version: u16, draining: bool              status.rs:21-22
  connection_state: ConnectionState               status.rs:23
  phi: Option<f64>, load: f32                     status.rs:24-25
  record_version: u64                             status.rs:26
  last_heartbeat_millis: u64                      status.rs:27
  counters: MeshPeerCounters                      status.rs:28

ConnectionState (snake_case)                      status.rs:31-40
  disconnected (default) | connecting | connected | suspect

MeshCounters                                      status.rs:42-49
  stale_generation_rejections: u64                status.rs:44
  foreign_relay_rejections: u64                   status.rs:47
  peers: Vec<MeshPeerCounters>                    status.rs:48

MeshPeerCounters                                  status.rs:51-61
  runtime_id: String
  streams_opened, streams_received,
  datagrams_sent, datagrams_received,
  gossip_frames_sent, gossip_frames_received,
  stale_generation_rejections: u64
```

Observations, all verified:

- **`Suspect` is derived, never stored.** `peer_statuses` recomputes it per call from
  `phi >= phi_suspect_threshold` (`membership.rs:340-344`); the stored
  `PeerState.connection_state` is only ever Disconnected/Connecting/Connected.
- **`MeshCounters.peers` duplicates every `MeshPeerStatus.counters`** verbatim
  (`membership.rs:302`), so each peer's counters appear twice in the JSON.
- **`MeshStatus.enabled` is a constant `true`** at the source (`membership.rs:352`);
  the `false` case is synthesized by the relay handler (`router.rs:404`), not by
  this data model.
- **`stale_generation_rejections` is structurally always 0 in production.** The only
  writer is `record_stale_generation_rejection` (`membership.rs:285-293`), whose
  sole call site anywhere is the unit test at `membership.rs:486`. The relay's real
  fence rejections are raised in `tunnel/directory.rs` / `audio/join.rs` and never
  routed back here.
- `endpoint_addrs` is echoed verbatim from the gossip record — see `-security.md` S-05.

---

#### 8. `MeshConfig` (config-shaped data, not persisted)

`lib.rs:53-64`: `{ enabled: bool, bind_addr: SocketAddr, registry_refresh: Duration }`.
Constructed only in `buzz-relay` (`config.rs:508-512`). See `-configuration.md`.

---

#### 9. Error taxonomy as data (`MeshError`, `lib.rs:65-127`)

**16 variants.** Four carry the fence-rejection taxonomy with structured payloads
(`StaleGeneration` `lib.rs:96-101`; `NoActiveLease` `lib.rs:110-117`; `OwnerMismatch`
`lib.rs:118-124`; `FutureGeneration` `lib.rs:125-130` — line refs per the enum body
spanning `lib.rs:66-127`). `lib.rs:102-109` documents the intended counter surface
`mesh_fence_rejections_total{reason=stale_generation|no_active_lease|owner_mismatch|future_generation}`
and states none of the four are serialized (the wire signal stays
`GoodbyeReason::StaleGeneration`, `wire.rs:172`).

Verified: **that metric does not exist.** grep for `mesh_fence_rejections_total`
across the repo returns nothing outside this doc comment.

Constructor census (grep across `crates/**`):

| Variant | Constructed in mesh crate | Constructed in `buzz-relay` |
|---|---|---|
| `Encode` / `Decode` | `wire.rs:178,184`; `gossip.rs:63,67` | `audio/join.rs:970,975` |
| `UnknownWireVersion` / `EmptyFrame` | `wire.rs:185,186` | — |
| `FrameTooLarge` | `peer.rs:143,179` | — |
| `DatagramTooLarge` | `lib.rs:219` | — |
| `PeerNotConnected` | `runtime.rs:169,187` | — |
| `Transport` | 22 sites (`endpoint.rs`, `peer.rs`, `registry.rs`, `gossip.rs`) | — |
| `Redis` | implicit via `#[from]` + `?` on `query_async` (`registry.rs:194,204,225,228`) | — |
| `StaleGeneration` | **none** | `tunnel/directory.rs:378,870`; `audio/join.rs:2854` |
| `NoActiveLease` | **none** | `tunnel/directory.rs:395,914` |
| `OwnerMismatch` | **none** | `tunnel/directory.rs:430,824`; `audio/join.rs:1081,1201,1888` |
| `FutureGeneration` | **none** | `tunnel/directory.rs:413,842` |
| `PeerDraining` (`lib.rs:94`) | **none** | **none** — dead variant |
| `Disabled` (`lib.rs:119`) | **none** | **none** — dead variant |

The fencing law holds structurally: the crate that *defines* the fence errors never
raises them. `PeerInfo` even has a compile-time reminder of this in the consumer:
`fn _peer_info_is_not_an_owner_signal(_peer: PeerInfo)` (`tunnel/reliable.rs:949`,
`#[allow(dead_code)]`).

---

#### 10. Documentation deltas

- **ARCHITECTURE.md does not cover this crate at all.** `grep -i "mesh|iroh|quic"
  ARCHITECTURE.md` → zero hits across all 827 lines, including §6 "Crate Reference"
  (`ARCHITECTURE.md:330`). `buzz-relay-mesh` is likewise absent from `AGENTS.md`'s
  repo-structure table, `CONTRIBUTING.md`, and `README.md`.
- `lib.rs:55-57` calls `BUZZ_MESH` "`on` (default when replicas can exist)". The
  implementation defaults it **off** (`config.rs:498-500`). Documented delta —
  see `-configuration.md`.
- `lib.rs:129-139` describes `PeerInfo.load` as "advisory load factor gossiped by the
  peer (0.0..)". No code ever assigns a non-zero `load`: `GossipRecord::new` sets
  `0.0` (`gossip.rs:35`) and the only mutation hook, `update_local(|_| {})`
  (`runtime.rs:566`), is a no-op closure. The field is structurally always `0.0`.
- `lib.rs:184-189` says `MeshStream`'s halves are "placeholder halves … pre-transport";
  they are in fact the real iroh-backed halves (`peer.rs:132-133`, `peer.rs:135-192`).
  Stale comment.
