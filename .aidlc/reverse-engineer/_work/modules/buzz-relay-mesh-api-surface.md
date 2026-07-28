## Module: buzz-relay-mesh (`crates/buzz-relay-mesh`)
### Aspect: API Surface

The crate exposes **8 public modules** (`lib.rs:21-28`) with **no `#[doc(hidden)]`
and no feature gates**. Everything is unconditionally public. `Cargo.toml` declares
no `[features]` section.

Public-item census (counting named types, traits, functions, consts, type aliases —
not trait-impl methods): **~120 items**. Breakdown below with verified caller data.

**Sole production consumer: `buzz-relay`** (`crates/buzz-relay/Cargo.toml:26`).
Verified — grep for `buzz_relay_mesh` across `crates/**` and `desktop/**` matches
exactly 8 files, all under `crates/buzz-relay/src/`: `mesh_boot.rs` (9),
`tunnel/reliable.rs` (6), `audio/join.rs` (5), `api/mesh_demo.rs` (4),
`config.rs` (2), `audio/mesh.rs` (2), `tunnel/directory.rs` (1),
`audio/handler.rs` (1).

---

#### 1. The boot-path handle and consumer-wiring API

The crate itself exposes **no boot function**. The handle consumers actually hold is
defined in `buzz-relay`, not here:

| Item | Where | Notes |
|---|---|---|
| `mesh_boot::boot_mesh(config, redis_pool, relay_keypair, shutting_down) -> anyhow::Result<Option<MeshHandle>>` | `crates/buzz-relay/src/mesh_boot.rs:412-420` | called once, `main.rs:442`; `Ok(None)` when `BUZZ_MESH` off (`mesh_boot.rs:417-419`) |
| `mesh_boot::MeshHandle` | `mesh_boot.rs:134-172` | published to `AppState.mesh: Arc<OnceLock<MeshHandle>>` (`state.rs:627`) at `main.rs:457`; `main.rs:460` is `unreachable!("mesh handle is set exactly once, right here")` |
| `MeshHandle::status()` | `mesh_boot.rs:172-174` | delegates to `MeshRuntime::membership().status()`; sole caller `router.rs:381` |
| `MeshHandle::wire_consumers(rooms, demo_echo, shutting_down)` | `mesh_boot.rs:183-199` | sole caller `main.rs:455-459`, i.e. **before** publication to `AppState` |
| `mesh_boot::wire_mesh_consumers(...)` (9 args) | `mesh_boot.rs:229-241` | `#[allow(clippy::too_many_arguments)]`; callers: `MeshHandle::wire_consumers` + one test (`mesh_boot.rs:686`) |
| `mesh_boot::MeshInboundDispatcher` + `register_huddle_control` / `register_reliable_stream` / `register_datagrams` | `mesh_boot.rs:57-59`, `:71,:78,:84` | the fan-out over the transport's single `set_inbound` slot; first registration wins (`OnceLock::set`) |
| `AppState::mesh() -> Option<&MeshHandle>` | `crates/buzz-relay/src/state.rs:812-814` | read only at `router.rs:381` |

`MeshHandle` fields (`mesh_boot.rs:136-171`): `directory` (Redis fenced session
directory), `transport: Arc<dyn RelayPeerTransport>`, `membership: Arc<dyn
RelayMeshMembership>`, `local_runtime_id`, `dispatcher`, `audio_fence`,
`runtime: MeshRuntime` (private), `owners`.

**Zero-reader field:** `MeshHandle.membership` is populated at `mesh_boot.rs:501`
and **never read**. grep for `.membership` field access on a `MeshHandle` across
`crates/buzz-relay/src` returns nothing; the three `.membership()` hits
(`mesh_boot.rs:173,488,501`) are all the `MeshRuntime` *method*, not the field.
Consequently `RelayMeshMembership::peers()` and `::local_runtime_id()` have **zero
production callers** (see §6).

---

#### 2. `MeshConfig` (`lib.rs:53-64`)

```rust
pub struct MeshConfig {
    pub enabled: bool,                          // lib.rs:57
    pub bind_addr: std::net::SocketAddr,        // lib.rs:60
    pub registry_refresh: std::time::Duration,  // lib.rs:63
}
```

`#[derive(Clone, Debug)]` (`lib.rs:52`). No `Default`, no builder, no
`Deserialize` — the relay hand-constructs it.

| Field | Constructed | Read |
|---|---|---|
| `enabled` | `config.rs:509` | `mesh_boot.rs:417` |
| `bind_addr` | `config.rs:510` | `mesh_boot.rs:383`, logged `mesh_boot.rs:396` |
| `registry_refresh` | `config.rs:511` (hardcoded 15 s) | `mesh_boot.rs:447` |

Struct field `Config.mesh` at `crates/buzz-relay/src/config.rs:136`.

---

#### 3. The two seam traits (the crate's documented contract)

`lib.rs:11-19` declares the relay consumes the crate "exclusively through two seams."
Verified: both are used, plus a third (`InboundHandler`) and the concrete
`MeshStream`/half traits, so the "exclusively two" claim is understated.

##### `RelayMeshMembership` (`lib.rs:144-151`)

| Method | Impl | Production callers |
|---|---|---|
| `peers() -> Vec<PeerInfo>` | `membership.rs:359-379` | **0** |
| `local_runtime_id() -> RuntimeId` | `membership.rs:381-383` | **0** |
| `begin_drain()` | `membership.rs:385-388` | 1 — `mesh_boot.rs:488` (drain watcher), reached via `MeshRuntime::membership()`, not via the trait object |

##### `RelayPeerTransport` (`lib.rs:158-175`)

| Method | Impl | Production callers |
|---|---|---|
| `send_datagram(to, dgram)` | `runtime.rs:167-175` | `audio/mesh.rs` media sink path (`audio/mesh.rs:37,56,260`); tests `mesh_boot.rs:660` |
| `open_session_stream(to, hello) -> BoxFuture<Result<MeshStream>>` | `runtime.rs:177-194` | `tunnel/reliable.rs:46,718`; `audio/join.rs:1002,1019,1483,1667`; `api/mesh_demo.rs:40,89` |
| `set_inbound(handler)` | `runtime.rs:196-198` | `mesh_boot.rs:509` (once, with `MeshInboundDispatcher`) |

Doc note at `lib.rs:161-163`: implementations do datagram-size + wire-version checks
but **not** generation fencing. Verified — `runtime.rs:167-194` touches neither
`generation` nor Redis.

##### `InboundHandler` (`lib.rs:177-180`)

`on_datagram(from, dgram)` / `on_session_stream(from, hello, stream)`.
Implemented by `MeshInboundDispatcher` (`mesh_boot.rs:95-133`) and by test doubles
(`runtime.rs:660` in-crate; `tunnel/reliable.rs:733`, `audio/join.rs:2198`,
`api/mesh_demo.rs:199`).

##### `MeshStream` + halves

`MeshStream` (`lib.rs:184-189`) is a concrete struct with `pub(crate)` fields; the
public constructor `MeshStream::new` lives in `peer.rs:199-201` and is documented as
existing so consumers can build in-memory streams in tests (`peer.rs:196-198`).
Verified consumers: `mesh_boot.rs:564` (`stub_stream`), `audio/join.rs:2083-2115`
(in-memory duplex over `tokio::sync::mpsc`).

`StreamSendHalf` (`lib.rs:191-194`: `send_frame`, `finish`) and `StreamRecvHalf`
(`lib.rs:196-198`: `recv_frame`) — production impls `IrohSendHalf`/`IrohRecvHalf`
(`peer.rs:132-192`); test impls at `mesh_boot.rs:566-580`, `audio/join.rs:2086-2109`.

`BoxFuture<'a, T>` (`lib.rs:141`) is public specifically so out-of-crate implementors
can name it (`lib.rs:138-140`); used at `mesh_boot.rs:566`, `audio/join.rs:2083`.

---

#### 4. `wire` — the frozen wire contract (`wire.rs`)

| Item | Location | External callers |
|---|---|---|
| `ALPN: &[u8] = b"buzz/mesh/1"` | `wire.rs:37` | none outside crate |
| `WIRE_VERSION: u8 = 1` | `wire.rs:42` | `mesh_boot.rs:367` (→ `proto_version`) |
| `MAX_STREAM_FRAME: u32 = 16 MiB` | `wire.rs:46` | `tunnel/reliable.rs:28` (doc), `:945` (test assert) |
| `RuntimeId` (+ `to_hex`, `Debug`, `Display`) | `wire.rs:62-81` | pervasive |
| `FencedHeader` | `wire.rs:85-93` | `tunnel/*`, `audio/*`, `api/mesh_demo.rs` |
| `Profile` (3 variants) | `wire.rs:97-109` | dispatcher routing `mesh_boot.rs:112-127` |
| `MeshDatagram` | `wire.rs:111-122` | `audio/mesh.rs`, `audio/join.rs` |
| `MeshStreamFrame` (4 variants) | `wire.rs:126-144` | `tunnel/reliable.rs`, `audio/join.rs` |
| `StreamRole` (2 variants) | `wire.rs:148-158` | `mesh_boot.rs:103`, `tunnel/reliable.rs`, `audio/join.rs` |
| `StreamHello` | `wire.rs:160-163` | as above |
| `GoodbyeReason` (3 variants) | `wire.rs:166-174` | `mesh_boot.rs:28`, `tunnel/reliable.rs`, `audio/join.rs` |
| `encode<T>` / `decode<T>` | `wire.rs:176-188` | in-crate only (`peer.rs:137,169,190`; test `tunnel/reliable.rs` uses the frame types, not the codec) |

**`#[non_exhaustive]` count: 0.** grep across `crates/buzz-relay-mesh/src` returns
nothing. Every public enum — including the "FROZEN" wire enums `MeshStreamFrame`,
`StreamRole`, `Profile`, `GoodbyeReason`, `GossipMessage`, `ConnectionState`, and
the 16-variant `MeshError` — is exhaustive. Adding a variant to any of them is a
breaking change for every downstream `match`, and the wire enums additionally have
no unknown-variant tolerance on decode: postcard rejects an out-of-range enum
discriminant, so a newer pod sending a 5th `MeshStreamFrame` variant produces
`MeshError::Decode` on an older pod, not a skip. The ALPN-bump discipline
(`wire.rs:34-36`) is the only mitigation, and it is a convention, not a mechanism.

---

#### 5. `endpoint` — transport bind/dial (`endpoint.rs`)

| Item | Location | Callers |
|---|---|---|
| `MeshEndpoint` | `endpoint.rs:12-15` | `mesh_boot.rs:25,383,422`; tests `tunnel/reliable.rs:663,782-785`, `api/mesh_demo.rs:152,272-273` |
| `MeshEndpoint::bind(SocketAddr)` | `endpoint.rs:19-21` | `mesh_boot.rs:383` — the only production bind |
| `MeshEndpoint::bind_with_secret_key(SecretKey, SocketAddr)` | `endpoint.rs:26-44` | tests only (in-crate `endpoint.rs:157-162`, `runtime.rs:627`; relay-side `tunnel/reliable.rs`, `api/mesh_demo.rs`) |
| `runtime_id()` | `endpoint.rs:47-49` | `mesh_boot.rs:386`; in-crate `runtime.rs:134,219,296,463` |
| `endpoint() -> iroh::Endpoint` | `endpoint.rs:51-53` | **0 callers anywhere** |
| `addr() -> EndpointAddr` | `endpoint.rs:55-57` | in-crate only (`endpoint.rs:64`, tests) |
| `ip_addrs() -> Vec<SocketAddr>` | `endpoint.rs:61-71` | `mesh_boot.rs:391-401` (`advertise_addrs`) |
| `accept()` / `connect(EndpointAddr)` | `endpoint.rs:73-93` | in-crate `runtime.rs:262,338`; relay tests |
| `runtime_id_from_public_key` | `endpoint.rs:96-98` | in-crate `endpoint.rs:38`, `peer.rs:58` |
| `public_key_from_runtime_id` | `endpoint.rs:100-102` | in-crate `endpoint.rs:105` |
| `direct_addr(RuntimeId, SocketAddr)` | `endpoint.rs:104-109` | in-crate `runtime.rs:333` |

`bind_with_secret_key` is documented "production should use `Self::bind`"
(`endpoint.rs:23-25`) — verified true.

---

#### 6. `membership` (`membership.rs`)

`MeshMembership` — 26 public methods. **Every one has zero callers outside the crate
except `status()` and `begin_drain()`**, both reached through
`MeshRuntime::membership()`:

| Method | Location | External callers |
|---|---|---|
| `new(GossipRecord)` | `:46-59` | `mesh_boot.rs:444` |
| `with_expected_relay_pubkey(String)` | `:61-64` | `mesh_boot.rs:445` |
| `with_phi_suspect_threshold(f64)` | `:66-69` | **0** — the 8.0 default is unoverridable in production |
| `local_record()` | `:71-75` | 0 (in-crate ×4) |
| `apply_ready_records(iter)` | `:85-118` | 0 (in-crate `runtime.rs:311,318`) |
| `apply_gossip_record(rec) -> bool` | `:120-153` | 0 (in-crate `runtime.rs:106,536`) |
| `mark_connection_state` | `:155-164` | 0 (in-crate ×4) |
| `update_local<F>` | `:166-178` | 0 (in-crate `runtime.rs:565`, `membership.rs:387`) |
| `is_draining()` | `:180-182` | 0 (the `is_draining` hits in `buzz-relay` are `HuddleOwnerRegistry`'s, `audio/join.rs:625`) |
| `has_peer(RuntimeId)` | `:187-192` | 0 (in-crate `runtime.rs:306,321`) |
| `records()` | `:195-206` | 0 (in-crate `runtime.rs:299`, `membership.rs:210,232`) |
| `digest()` | `:208-223` | 0 (in-crate `runtime.rs:567`) |
| `delta_for(&[GossipDigestEntry])` | `:225-247` | 0 (in-crate `runtime.rs:539`) |
| `record_stream_opened/received`, `record_datagram_sent/received`, `record_gossip_frame_sent/received` (6) | `:249-283` | 0 (in-crate, one call site each) |
| `record_stale_generation_rejection(Option<RuntimeId>)` | `:285-293` | **0 anywhere** — only the test at `:486` |
| `status() -> MeshStatus` | `:295-313` | `mesh_boot.rs:173` |

##### `PeerInfo` (`lib.rs:129-139`)

Referenced in `buzz-relay` only at `tunnel/reliable.rs:664` (test import) and
`:949` — a `#[allow(dead_code)] fn _peer_info_is_not_an_owner_signal(_peer: PeerInfo)`
that exists purely to document the fencing law. **No production code reads a
`PeerInfo`.**

---

#### 7. `runtime` (`runtime.rs`)

| Item | Location | Callers |
|---|---|---|
| `DEFAULT_RECONCILE_INTERVAL = 5s` | `:42` | in-crate `:97` |
| `DEFAULT_GOSSIP_INTERVAL = 2s` | `:44` | in-crate `:96` |
| `MeshRuntime` | `:77-80` | `mesh_boot.rs:29,161,218,473` |
| `MeshRuntime::start(endpoint, membership, Option<registry>)` | `:88-100` | `mesh_boot.rs:473` |
| `MeshRuntime::start_with_intervals(..., gossip, reconcile)` | `:102-127` | **0 external** (in-crate tests `:632`) |
| `membership() -> &MeshMembership` | `:129-131` | `mesh_boot.rs:173,488,501` |
| `local_runtime_id()` | `:133-135` | **0 external** (tests only) |
| `connected_peers() -> Vec<RuntimeId>` | `:138-146` | **0 anywhere outside in-crate tests** |
| `reconcile_now()` | `:150-152` | `mesh_boot.rs:478` |
| `shutdown()` | `:155-164` | **0 external** — the relay never calls it; only in-crate tests do |
| `spawn_registry_heartbeat(registry, record, ready)` | `:594-608` | `mesh_boot.rs:467` |

`MeshRuntime::shutdown()` being uncalled is a real leak: `MeshRuntime` doc
(`runtime.rs:75-76`) says "dropping all clones does NOT stop the loops — call
`shutdown()` for that." The relay's drain watcher (`mesh_boot.rs:481-497`) calls
`begin_drain()` + `owners.drain_all()` and returns; the accept/reconcile/gossip
loops and the registry heartbeat run until process exit. Registry cleanup relies on
`ReadyHeartbeat::tick(false)` (`registry.rs:299-302`) — and `ReadyHeartbeat::shutdown()`
(`registry.rs:306-312`) also has **zero callers**.

---

#### 8. `registry` (`registry.rs`)

| Item | Location | Callers |
|---|---|---|
| `READY_KEY_PREFIX`, `DEFAULT_REGISTRY_REFRESH`, `REGISTRY_EXPIRY_MULTIPLIER`, `ATTESTATION_CONTEXT` | `:19-22` | **0 external each**; `DEFAULT_REGISTRY_REFRESH` is not even used by the relay, which hardcodes 15 s at `config.rs:511` |
| `RuntimeAttestation` + `new` + `verify` | `:30-51` | `new` in-crate `:121`; **`verify` has 0 callers anywhere** |
| `attestation_preimage` | `:85-91` | in-crate `:94` only |
| `ReadyRecord` + `new` + `key` + `verify_attestation` | `:99-147` | `new` → `mesh_boot.rs:448`; `key`/`verify_attestation` in-crate |
| `ready_key(RuntimeId)` | `:150-152` | in-crate `:136`, `:200` |
| `expiry_for(Duration)` | `:154-156` | in-crate `:175` |
| `ReadyRegistry::new / refresh_interval / expiry / publish_ready / clear_ready / scan_ready / heartbeat` | `:166-262` | `new` → `mesh_boot.rs:447`; `publish_ready` → `mesh_boot.rs:459`; the rest in-crate (`runtime.rs:311,318,601,604`) |
| `ReadyHeartbeat::record / published / tick / shutdown` | `:287-312` | `tick` in-crate `:602`; `record`/`published`/`shutdown` → **0 production callers** |

---

#### 9. `gossip` (`gossip.rs`)

`GOSSIP_PAYLOAD_VERSION` (`:13`), `GossipRecord` + `new` (`:16-42`),
`GossipDigestEntry` (`:45-48`), `GossipMessage` (`:51-60`), `encode_message` /
`decode_message` (`:62-77`), `GossipState` + 7 methods (`:81-166`), `PhiAccrual` +
`new/observe/phi_at/mean_secs` (`:168-220`), `now_millis` (`:223-229`),
`system_time_from_millis` (`:231-233`).

External callers: **`GossipRecord` and `GossipRecord::new` only**
(`mesh_boot.rs:26,439`). Everything else in this module is in-crate or unused:

- `GossipState` and all 7 of its methods — **zero callers outside its own tests**.
- `PhiAccrual::new` (only `Default` is used, `membership.rs:137`),
  `PhiAccrual::mean_secs` (in-crate `:212` only), `GossipDigestEntry`,
  `GossipMessage`, `encode_message`/`decode_message`, `now_millis`,
  `system_time_from_millis` — all zero external callers.

---

#### 10. `status` (`status.rs`) and `lib` root

`MeshStatus`, `MeshPeerStatus`, `ConnectionState`, `MeshCounters`, `MeshPeerCounters`
— all re-exported (`lib.rs:46`). External use: `MeshStatus` is named at
`mesh_boot.rs:29,172` and serialized at `router.rs:381`. The other four have zero
direct external references (they arrive as JSON through `serde_json::to_value`).

`ConnectionState` appears to have 25 hits in `buzz-relay` — **all false positives**:
that is the relay's own WebSocket `ConnectionState` (`connection.rs:53`,
`handlers/auth.rs:17`, etc.), an unrelated type.

Root-level items: `MeshConfig` (§2), `MeshError` (16 variants, `lib.rs:65-127`),
`PeerInfo` (`lib.rs:129-139`), `BoxFuture` (`lib.rs:141`), the 4 traits, `MeshStream`,
and `encode_datagram_checked` (`lib.rs:213-226`) — the last has **zero callers
outside `peer.rs:110`** despite being documented as a "raw bytes helper used by
transport internals."

`peer::PeerCounters` (`peer.rs:10-15`) and `MeshPeer::counters()` (`peer.rs:73-75`)
are public with **zero callers anywhere** — the atomics are incremented
(`peer.rs:84,97,114,121`) and never read.

---

#### 11. Public-surface summary

| Category | Count | Zero-external-caller count |
|---|---|---|
| Public modules | 8 | 0 |
| Public traits | 4 (`RelayMeshMembership`, `RelayPeerTransport`, `InboundHandler`, `StreamSendHalf`+`StreamRecvHalf` = 5) | 0 |
| Public structs | 17 | 8 (`GossipState`, `PhiAccrual`, `GossipDigestEntry`, `RuntimeAttestation`, `ReadyHeartbeat`, `PeerCounters`, `MeshCounters`, `MeshPeerCounters`) |
| Public enums | 7 | 1 (`GossipMessage`) |
| Public consts | 9 | 7 |
| Free functions | 11 | 8 |
| `#[non_exhaustive]` | **0** | — |
| `#[deprecated]` | 0 | — |

Roughly **40% of the public surface has no caller outside the crate**, concentrated
in `gossip::GossipState` (a full duplicate of `MeshMembership`'s logic), the
registry's constant/helper layer, and the two parallel counter models.
