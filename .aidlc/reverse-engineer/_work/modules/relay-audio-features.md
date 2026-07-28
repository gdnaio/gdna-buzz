## Module: buzz-relay — huddle audio, tunnel & conformance seam (`crates/buzz-relay/src`)
### Aspect: Features

---

#### 1. Huddle audio — capability inventory

| Capability | Where | Completeness |
|---|---|---|
| WebSocket audio room per (community, channel) | `audio/room.rs:490-550`, route `router.rs:125-128` | Complete for single-pod |
| NIP-42 challenge/response auth on the audio socket | `handler.rs:176-238` | Complete — reuses `state.auth.verify_auth_event`, the same verifier as the main relay door |
| Channel-membership gate with ephemeral auto-add | `handler.rs:1153-1235` | Complete; auto-add restricted to TTL channels whose caller is a parent member |
| Opaque Opus fan-out with 1-byte `peer_index` prefix | `room.rs:391-411` | Complete; never decodes, never re-encodes |
| v2 frame header parse + telemetry clamp | `audio/wire.rs:29-88` | Complete; 4 tests pin byte order, short input, clamp-keeps-frame, reserved-bit passthrough |
| Room protocol-version pinning | `room.rs:110-127`, `:239-247` | Complete; 5 tests |
| Peer-index allocation with recycling | `room.rs:143-157` | Complete; 0..=254 |
| Separate audio / control queues with priority drain | `room.rs:23-27`, `handler.rs:1060-1086` | Complete |
| Heartbeat + missed-pong disconnect | `handler.rs:1127-1151` | Complete (30 s / 3 misses) |
| Auto-end + channel archive when the room empties | `room.rs:358-388`, `handler.rs:833-866` | Complete, with rollback on archive failure |
| Lifecycle events 48101 / 48102 / 48103 (relay-signed, persisted, fanned out locally, published to Redis) | `handler.rs:645-653`, `:822-831`, `:850-859`, `:1237-1332` | Complete |
| Ordered roster snapshot/delta model | `room.rs:55-81`, `:452-476` | Complete; broadcast cap 64 with snapshot recovery |
| Cross-pod huddle: owner-authoritative fan-out over the mesh | `audio/join.rs`, `audio/mesh.rs` | Implemented end-to-end; **only active when `BUZZ_MESH=on`** (default off) |
| Redis fenced-CAS ownership with generation monotonicity | `tunnel/directory.rs:20-79`, `:348-439` | Complete |
| Per-room owner-lease renewer with observable loss | `join.rs:452-562`, `:600-773` | Complete |
| Graceful drain of owned huddles on SIGTERM | `join.rs:750-773`, `mesh_boot.rs:481-496` | Complete |
| Ambiguity-safe media routing (fail closed on channel-UUID collision) | `room.rs:526-541`, `mesh.rs:221-227` | Complete |

---

#### 2. Absent from huddle audio (verified by reading, not inferred)

| Feature | Evidence of absence |
|---|---|
| **Recording** | No writer, no storage call, no media handle in `crates/buzz-relay/src/audio/`. `ARCHITECTURE.md:825` lists "Huddle recording/tracks not built" as known-absent; confirmed |
| **Transcription / STT** | Not in the relay. STT is client-side: `desktop/src-tauri/src/huddle/stt.rs` |
| **Video** | No video kind, no video track, no SDP. `buzz-core/src/kind.rs:454` calls 48100 "audio/video session" but nothing in the relay handles video |
| **Screen share** | No reference anywhere in `crates/buzz-relay/src` |
| **SFU mixing / transcoding** | The relay never decodes: `broadcast_frame` copies bytes with a 1-byte prefix (`room.rs:398-411`). Fan-out is N×(N−1) mesh, not a mixer. No Opus, WebRTC, or codec crate in `crates/buzz-relay/Cargo.toml` |
| **Echo cancellation / noise suppression / AGC** | None; no DSP dependency |
| **TURN / STUN / ICE** | None. Clients connect straight to the relay over WSS. The default `BUZZ_MESH_BIND_ADDR` port `3478` (`config.rs:507`) is the *STUN* port number but the protocol spoken there is iroh/QUIC, not STUN — a naming coincidence, not a TURN server |
| **Codec negotiation** | Nothing is negotiated except the integer `protocol_version` (`handler.rs:134-138`). Sample rate, channel count, frame duration, and bitrate are never exchanged. The 48 kHz assumption lives only in a field name (`wire.rs:44`) and the "20 ms/frame" comment behind the queue sizing (`room.rs:40`) |
| **Simulcast / layered audio / redundancy (RED/FEC)** | None |
| **Jitter buffer / packet reordering / loss concealment** | None relay-side; `seq`/`ts_48k` are parsed for `tracing::trace!` only (`handler.rs:996-1003`). Playout is client-side (`desktop/src-tauri/src/huddle/playout.rs`) |
| **Server-side mute / kick / moderation** | No mute state, no kick message, no moderator role on `Room` (`room.rs:157-170`) |
| **Active-speaker / dominant-talker election** | Explicitly forbidden from consuming `level_dbov` (`wire.rs:16-20`); done client-side (`playout.rs:220`) |
| **Per-room bitrate/frame-rate limits or admission-time quality control** | Only a byte cap (4096/frame) and a queue depth (8) |
| **Push-to-talk, hand-raise, reactions in-huddle** | None |
| **Huddle metrics** | Zero `metrics::` calls in `audio/` — no gauge for active rooms/peers, no counter for dropped frames, no join-failure counter. The only metric in the whole group is `mesh_fence_rejections_total` (`tunnel/directory.rs:483`) |
| **Bandwidth accounting / per-peer rate limiting** | None |
| **A `speakers` control message** | Promised by the doc comment at `room.rs:33`; never produced |
| **`PeerCtrl::Close`** | Variant exists (`room.rs:35`) and is handled (`handler.rs:1112`) but has **zero producers** — the graceful per-peer shutdown path it implies is not wired |
| **Session resumption / reconnect continuity** | A rejoin is a fresh handshake with a fresh peer UUID and (usually) a different index |
| **Kind 48106 (huddle guidelines)** | Defined `kind.rs:462`, allowlisted for labelling `handlers/event.rs:49`, but never produced or consumed by the relay. Client-side only (`desktop/src-tauri/src/huddle/agents.rs:31`) |
| **Kind 48100 (huddle started)** | The relay only *reads* it, to validate a claimed parent (`handler.rs:1176-1186`). The producer is the desktop client (`desktop/src-tauri/src/huddle/mod.rs:252`) |

---

#### 3. What the tunnel lane delivers

| Capability | Where | Completeness |
|---|---|---|
| Redis fenced session directory (acquire / renew / release / lookup / validate) with a non-expiring generation counter | `tunnel/directory.rs:180-439` | Complete and well-tested (7 tests, 5 of which need live Redis) |
| Typed fence-rejection taxonomy shared with `/_mesh` and the huddle wire | `directory.rs:348-439`; `join.rs:982-1017` | Complete |
| Reliable-stream join routing (own locally vs. open a fenced bi-stream to the owner) | `reliable.rs:78-126` | Complete |
| Owner-side inbound accept with structural validation | `reliable.rs:135-172` | Complete |
| 1 MiB-chunked ordered framing with a community-bearing inner header | `reliable.rs:274-294`, `:412-492` | Complete |
| Per-frame Redis fence validation that **fails the session** on mismatch | `reliable.rs:326-389` | Complete |
| Observable lease renewer | `reliable.rs:571-657` | Implemented; **never spawned** |

##### 3.1 What the tunnel lane does **not** deliver

- **No product consumer.** `mesh_boot.rs:299-303` accepts an inbound reliable stream,
  logs `"reliable stream accepted; no session consumer wired — closing"`, and closes
  it. `mesh_boot.rs:206-215` states plainly that renewal attaches on the join path
  "which lands with the first product session consumer".
- **No join-side product caller.** The only route reaching
  `ReliableStreamRouter::join` is the demo-gated `POST /_mesh/demo/echo`
  (`api/mesh_demo.rs:73`, `:95`), documented as "Not a product flow"
  (`api/mesh_demo.rs:21-22`).
- **No renewal in the demo path either** — `api/mesh_demo.rs:11-15` documents that
  the `Owned` arm deliberately lets the lease die with its 30 s TTL, and instructs
  operators to drive the second pod inside that window.
- **No retransmission, ACK, flow control, or backpressure signalling** beyond what
  QUIC provides. There is no seq/ack in `ReliableWireFrame` (`reliable.rs:412-422`).
- **No goose/berd bridge.** The module doc says it is "for berd ↔ goose-server
  sessions" (`reliable.rs:1`) but no such wiring exists in the repo.
- `SessionDirectory::takeover` (`directory.rs:233`) and
  `ReliableMeshStream::with_community` (`reliable.rs:263`) have **zero callers**;
  `SessionDirectory::known_generation` (`directory.rs:324`) has test-only callers.

---

#### 4. What the conformance seam delivers

| Capability | Where | Completeness |
|---|---|---|
| `Tracer` trait + two impls (`NoopTracer`, `JsonlTracer`) | `conformance/tracers.rs:14-72` | `NoopTracer` in use; `JsonlTracer` **never instantiated anywhere** |
| Label projection (community, host, actor, msg-id, channel) | `conformance/mod.rs:47-91` | Complete, with one documented delta (§4.2) |
| Claimed-vs-resolved community separation | `conformance/mod.rs:94-117` | Complete |
| Write-path emits (`WriteInsert`/`WriteInsertGlobal`/`WriteDuplicate`/`SanitizedError`/`AuthCheck`) | consumed from `handlers/ingest.rs:2192`, `:2334-2348`, `:2468-2485` | Wired on the ingest path |
| Read-path emits (`ReadMessageRows`, `ReadByIdRows`) with row-community projection | `conformance/mod.rs:265-330`, called `req.rs:355`, `:671` | Wired |
| `ReadHostFeedRows` | schema `buzz-conformance/src/lib.rs:250` | **No emitter in the relay** — only the conformance crate's own proptest references it |
| Row-lookup coverage guard (`MissingLookup` → `ImplBug`) | `conformance/mod.rs:216-255` | Complete; 4 tests |
| `EmitGuard` RAII coverage-breach guard | `conformance/mod.rs:334-414` | Complete; 2 self-tests (`:454-528`) |
| `IngestError → SanitizedReason` closed-alphabet mapping | `conformance/mod.rs:422-429` | Complete, exhaustive match |

##### 4.1 Is it active in production? No.

`state.rs:794-798` binds `Arc::new(crate::conformance::NoopTracer)` unconditionally
at `AppState` construction. There is no config flag, env var, or feature gate that
swaps it. `NoopTracer::record` is `fn record(&self, _step: TraceStep) {}`
(`tracers.rs:22-24`). Consequences:

- Every emit site still **constructs** its `TraceAction` (allocating `String`s for
  labels, cloning `AbstractState`) and then discards it. The claimed "zero cost"
  (`tracers.rs:11-13`, `state.rs:615`) is not achieved — the work is done, only the
  write is elided. There is no `#[cfg(feature = ...)]` anywhere in the module.
- The `EmitGuard`'s `ImplBug` on a coverage breach is recorded into `NoopTracer` and
  therefore **silently discarded** in production. The guard is a test-time
  mechanism that is armed in production for no observable benefit.
- `JsonlTracer` — the only impl that would make the seam observable — has **zero
  callers in the workspace**. `state.rs:795-797` promises "Conformance tests
  overwrite this with a JsonlTracer after construction (see test helpers in
  `crates/buzz-test-client` once those land)". Those helpers have not landed: grep
  for `JsonlTracer` finds only the definition and two doc comments.
- `conformance/mod.rs:47-48` documents wire points in `req.rs`/`event.rs` as
  "held back as additive patch for Eva to apply onto Max's req.rs writes — see
  thread `c882c9b1…`". The `req.rs` sites **have** landed (`req.rs:144`, `:355`,
  `:671`); the doc comment is stale.

##### 4.2 Documented deltas in the conformance module

| Doc claim | Line | Actual code |
|---|---|---|
| "`actor` is the lower 16 bytes of `blake3(pubkey_bytes)` as a hex string" | `conformance/mod.rs:51-53` | `actor_label` takes the **first 16 hex chars of the pubkey hex** — no hashing (`:70-74`). The rationale at `:63-69` explains why this is acceptable, but the doc comment 10 lines earlier still says blake3 |
| "The trace carries opaque labels … never … wall-clock timestamps" | `conformance/mod.rs:12-15` | Honoured: `TraceStep` has no timestamp field (`buzz-conformance/src/lib.rs:290-299`). `bound_host` is a full host string, not opaque (`:58`), and `channel_label` is a direct UUID wrap — both deliberate (`:87-88`) |
| "the build can have the compiler eliminate them entirely behind a feature flag" | `tracers.rs:11-13` | No such feature flag exists in `crates/buzz-relay/Cargo.toml` (`[features] dev = ["buzz-auth/dev"]` only) |

---

#### 5. Mesh boot seam — what it delivers

| Capability | Where |
|---|---|
| Single construction point for all mesh machinery, `None` when off | `mesh_boot.rs:411-520` |
| iroh endpoint bind with a boot-unique keypair → boot-unique `RuntimeId` | `mesh_boot.rs:423-431` |
| Relay-key-attested ready record published to Redis + readiness-gated heartbeat | `mesh_boot.rs:445-472` |
| Advertise-address preference chain (`BUZZ_MESH_ADVERTISE_ADDR` → `POD_IP`+bound port → all endpoint IPs) | `mesh_boot.rs:379-402` |
| Static capability advertisement (all three profiles in one binary) | `mesh_boot.rs:369-377` |
| Per-profile inbound dispatcher over a single transport slot | `mesh_boot.rs:44-130` |
| Immediate reconcile pass so seed peers are dialed at boot | `mesh_boot.rs:478` |
| Drain watcher: gossip `draining=true` then drain owned huddles | `mesh_boot.rs:481-496` |
| Shared single `GenerationFloor` between the datagram hot path and huddle teardown | `mesh_boot.rs:149-166`, `:236-245`; pinned `mesh_boot.rs:665-737` |
| `BUZZ_MESH_DEMO_ECHO` reliable-stream echo consumer | `mesh_boot.rs:307-364` |

##### 5.1 Absent from the boot seam

- **No mesh readiness gate on the huddle path.** `boot_mesh` returns a handle as
  soon as the endpoint binds and the ready record publishes; there is no wait for
  peer connectivity. A join that resolves `RemoteOwner` before the owner peer is
  dialed fails with `huddle_owner_unreachable` (`handler.rs:487-503`).
- **`MeshHandle.membership` is written but never read.** Populated at
  `mesh_boot.rs:501`, exposed at `:141`, with zero consumers. `/_mesh` reads the
  private `runtime` field instead (`mesh_boot.rs:172-174`).
- **No mesh peer count / session gauge.** `/_mesh` returns whatever
  `MeshStatus` carries, unauthenticated (`router.rs:399-406`).
