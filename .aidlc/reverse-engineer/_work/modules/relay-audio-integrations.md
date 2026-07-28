## Module: buzz-relay — huddle audio, tunnel & conformance seam (`crates/buzz-relay/src`)
### Aspect: Integrations

---

#### 1. Dependency map

| Integration | Reached via | Call sites in this group | Failure mode |
|---|---|---|---|
| `buzz-relay-mesh` (iroh/QUIC) | `MeshHandle.transport: Arc<dyn RelayPeerTransport>`, `MeshRuntime`, `MeshEndpoint` | `mesh_boot.rs:423`, `:475`, `:502`, `:505`; `join.rs:1675`, `:1762`; `mesh.rs:277`; `reliable.rs:120` | §2 |
| `buzz-pubsub` / Redis pub-sub | `state.pubsub.publish_event` | `handler.rs:1322-1331` | §3.1 |
| Redis (deadpool) — session directory | `SessionDirectory` over `deadpool_redis::Pool` | `directory.rs:214`, `:249`, `:283`, `:313`, `:331`, `:359` | §3.2 |
| Redis — mesh ready registry | `ReadyRegistry` | `mesh_boot.rs:445`, `:461`, `:468` | §3.3 |
| `buzz-db` (Postgres) | `state.db.*` | `handler.rs:158`, `:389`, `:838`, `:1164`, `:1179`, `:1197`, `:1212`, `:1219`, `:1281` | §4 |
| `buzz-auth` | `generate_challenge`, `state.auth.verify_auth_event` | `handler.rs:176`, `:222` | §5 |
| `buzz-core` | `CommunityId`, `TenantContext`, `StoredEvent` | throughout | compile-time only |
| `buzz-conformance` | `Tracer`, `TraceStep`, `TraceAction`, `AbstractState`, … | `conformance/mod.rs:38-44`; `tracers.rs:12` | §6 |
| `nostr` crate | `EventBuilder`, `Kind`, `Tag`, `Keys` | `handler.rs:1240-1272`; `mesh_boot.rs:415`, `:452` | signing failure → `warn!` + skip (`handler.rs:1270-1273`) |
| `postcard` | `HuddleControlMsg` codec | `join.rs:1007`, `:1012` | `MeshError::Encode`/`Decode` |
| `dashmap` | rooms, peers, generation floor, owner registry | `room.rs:490`, `:163`; `mesh.rs:90`; `join.rs:601` | none (in-process) |
| `metrics` | one counter | `directory.rs:483` | none |
| `mesh-llm-sdk` / `mesh-llm-host-runtime` | **dev-dependencies only** | `Cargo.toml:87-88`; consumed by `examples/mesh_agent_e2e.rs:25,48`, `examples/mesh_serve_client_smoke.rs:29,44-45,53`, `examples/mesh_serve_smoke.rs:13`, `examples/mesh_stack_smoke.rs:53` | §7 |

**No audio codec or media library is linked.** `crates/buzz-relay/Cargo.toml` has no
`opus`, `webrtc`, `cpal`, `dasp`, `symphonia`, `stun`, or `turn` entry — verified by
grep. The relay treats every audio byte as opaque.

---

#### 2. `buzz-relay-mesh` — the deepest coupling

##### 2.1 Surfaces consumed

| Type / fn | Where used |
|---|---|
| `RelayPeerTransport::{send_datagram, open_session_stream, set_inbound}` | `join.rs:1675`, `:1762`; `mesh.rs:277`; `reliable.rs:120`; `mesh_boot.rs:505` |
| `InboundHandler::{on_datagram, on_session_stream}` | implemented by `MeshInboundDispatcher`, `mesh_boot.rs:91-130` |
| `MeshStream::{send_frame, recv_frame, finish}` | `join.rs:1010`, `:1179`, `:1783`; `reliable.rs:283`, `:317`, `:334` |
| `MeshStreamFrame` (4 variants) | `join.rs`, `reliable.rs`, `mesh_boot.rs` |
| `FencedHeader`, `MeshDatagram`, `Profile`, `GoodbyeReason`, `RuntimeId`, `StreamHello`, `StreamRole`, `MeshError` | throughout |
| `MeshEndpoint::bind`, `endpoint.ip_addrs()`, `endpoint.runtime_id()` | `mesh_boot.rs:423`, `:392`, `:432` |
| `MeshRuntime::{start, reconcile_now, membership, clone}` | `mesh_boot.rs:475`, `:478`, `:173`, `:501-502` |
| `MeshMembership::{new, with_expected_relay_pubkey}`, `RelayMeshMembership`, `MeshStatus` | `mesh_boot.rs:441-443`, `:501`, `:172-174` |
| `ReadyRegistry`, `ReadyRecord`, `GossipRecord`, `spawn_registry_heartbeat` | `mesh_boot.rs:445-472` |
| `WIRE_VERSION`, `wire::MAX_STREAM_FRAME` | `mesh_boot.rs:367`; `reliable.rs:945` |

`MeshHandle` is the sole gateway: `AppState::mesh()` returns `Option<&MeshHandle>`
(`state.rs:812-814`), and every consumer branches on that `Option`
(`handler.rs:306`, `:449`, `:577`, `:875`).

##### 2.2 Trust boundary between the two crates

- Inbound mesh connections are gated on `is_known_peer`, which requires a
  Redis ready-registry record (`buzz-relay-mesh/src/runtime.rs:275-283`, `:309-320`).
  Records are attested against the relay signing key
  (`MeshMembership::with_expected_relay_pubkey`, `mesh_boot.rs:442-443`).
- The `from: RuntimeId` handed to every handler is the **authenticated QUIC peer
  identity** (`runtime.rs:392-399`, `:412`), which is what lets
  `accept_inbound` assert `hello.sender == from` (`join.rs:1060-1065`,
  `reliable.rs:143-148`).
- The mesh layer itself does **no** Redis fencing. Every fence check in this group
  is performed by the *consumer*: `join.rs:1231-1245` (control frames),
  `reliable.rs:381-385` (reliable frames), `mesh.rs:212-220` (media, local floor
  only — see security).

##### 2.3 Failure modes

| Failure | Behaviour |
|---|---|
| `MeshEndpoint::bind` fails with mesh on | **Fatal at boot** — `anyhow` error propagated from `mesh_boot.rs:423-431` to `main.rs:442` |
| Peer unreachable when dialing an owner | `DialError::Mesh` → WS `huddle_owner_unreachable`; the joining client gets a clean error, and `cleanup_if_empty` runs (`handler.rs:487-503`) |
| `send_datagram` fails (disconnected peer, oversize) | `debug!` and continue — audio drop-on-error (`join.rs:1762-1765`, `mesh.rs:277-282`) |
| `send_frame` on the control stream fails | breaks the serve loop with `Err`, then unconditional peer teardown (`join.rs:1245-1254`, `:1345-1367`) |
| Owner pod dies mid-call | ingress reader sees a bare close → `StreamClosed` → cancel + `fence.forget` (`join.rs:1604-1610`, `handler.rs:707-714`) |
| Owner drains (SIGTERM) | `Goodbye(Draining)` → ingress rejoins; local owner clients closed by the drain watcher (`join.rs:1157-1161`, `handler.rs:735-748`) |
| Traffic arrives before a profile handler is registered | logged and dropped; the peer's fenced retry is safe (`mesh_boot.rs:52-55`, `:92-100`, `:122-129`) |
| `RealtimeMedia` arrives as a stream | rejected without routing (`mesh_boot.rs:113-121`) |
| MTU overflow on a media datagram | the sink drops it with a `debug!`; the comment explicitly says MTU prevention "is the ship-gate's job" (`mesh.rs:278-281`) — i.e. **no runtime MTU check exists** |

---

#### 3. Redis — three independent uses

##### 3.1 `buzz-pubsub` (lifecycle-event fan-out)

Single call: `state.pubsub.publish_event(tenant, EventTopic::Channel(parent), &event)`
(`handler.rs:1322-1325`). Topic is the **lifecycle parent channel**, not the
ephemeral huddle channel.

Failure: the event is already persisted and already fanned out locally, so a publish
error only means other pods miss the live delivery. `local_event_ids` is invalidated
so a later echo is not suppressed (`handler.rs:1326-1330`), then `warn!`.

##### 3.2 Session directory (ownership arbiter)

`deadpool_redis::Pool`, shared with the rest of the relay
(`state.redis_pool.clone()` → `mesh_boot.rs:442`, `:512`). Four Lua scripts
(`directory.rs:20-79`) + two plain `GET`s (`directory.rs:313`, `:331`).

| Failure | Behaviour |
|---|---|
| pool checkout fails | `DirectoryError::Pool` → `MeshError::Transport` at every `HuddleDirectory` boundary (`join.rs:114`, `:139`, `:158`, `:172`) |
| Redis unreachable during `resolve_join` | join fails → WS `join_rejected` (`handler.rs:342-355`). **Huddles become unjoinable when Redis is down and mesh is on** |
| Redis unreachable during renewal | `Err` → treated as **owner loss**, `lost` fires, every local owner client is closed for rejoin (`join.rs:521-529`, `handler.rs:756-765`). A Redis blip therefore drops every cross-pod huddle on the pod |
| Redis unreachable during owner-side `validate` | non-fence error → the whole `HuddleControl` stream is torn down (`join.rs:1240-1244`) |
| Malformed lease value in Redis | `MalformedLease` → `Transport` error → join failure; never a silent default (`directory.rs:495-531`) |
| Lease key expires while the pod still serves peers | Redis stops naming the pod; the next renew returns `Lost`. Between expiry and the next renew tick (up to 10 s) the pod keeps fanning out with a dead lease — local WS peers have **no per-frame fence** |

##### 3.3 Mesh ready registry

`ReadyRegistry::new(redis_pool, config.mesh.registry_refresh)` (`mesh_boot.rs:445`),
first `publish_ready` at `:461`, then `spawn_registry_heartbeat` gated on
`!shutting_down` (`mesh_boot.rs:466-472`).

Failure: the **first** publish failing is fatal to boot (`mesh_boot.rs:459-463`) —
"if Redis can't take the attested record, peers can never find us". Later heartbeat
failures are internal to `buzz-relay-mesh`.

---

#### 4. `buzz-db` (Postgres)

| Call | Line | Purpose | Failure mode |
|---|---|---|---|
| `is_community_active(community)` | `handler.rs:158` | community lifecycle gate | closure result drives `run_registered_community_connection`; a DB error there rejects the connection |
| `get_channel(community, channel)` | `handler.rs:1164` (in `ensure_membership`) | load channel, reject archived | `Err` → `"db error: {e}"` → WS `not a member` |
| `get_channel(community, channel)` | `handler.rs:389` | post-`get_or_create` archived re-check | `Err` → **fail closed**, silent teardown (`handler.rs:404-410`) |
| `huddle_started_link_exists(community, parent, channel, created_by)` | `handler.rs:1179-1186` | verify a creator-signed kind-48100 link before trusting a client-supplied parent | `Err` → `"db error"`; `false` → `"ephemeral channel is not linked to claimed parent"` |
| `is_member_cached(community, channel, pubkey)` ×2 | `handler.rs:1197`, `:1212` | membership fast path + parent check | `Err` → `"db error"` |
| `add_member(community, channel, pubkey, Member, Some(created_by))` | `handler.rs:1219-1227` | ephemeral auto-add | `Err` → `"auto-add failed: {e}"` → join refused |
| `invalidate_membership(tenant, channel, pubkey)` | `handler.rs:1228` | cache coherence after auto-add | infallible |
| `archive_channel(community, channel)` | `handler.rs:838` | auto-end | `Err` → `clear_ended()`, huddle stays alive, no 48103 (`handler.rs:840-845`) |
| `insert_event(community, &event, Some(parent))` | `handler.rs:1281-1284` | persist lifecycle event | duplicate → skip fan-out; `Err` → fan out from memory anyway (`handler.rs:1285-1307`) |

Note the **double `get_channel`** on every join (`:1164` and `:389`) — two round
trips for the same row, deliberate to close a race but uncached.

---

#### 5. `buzz-auth`

- `generate_challenge()` (`handler.rs:33`, `:176`) — the challenge nonce.
- `state.auth.verify_auth_event(event, &challenge, &relay_url)` (`handler.rs:220-238`)
  — full NIP-42 verification, identical to the main relay door. The returned
  `auth_ctx.pubkey` is the only identity used downstream (`handler.rs:240-242`).
- `crate::handlers::auth::extract_auth_tag_json(&event)` (`handler.rs:217`) —
  NIP-OA tag pulled out *before* the event is consumed by the verifier.
- `crate::api::bridge::nip42_expected_relay_url(&state.config.relay_url, &tenant)`
  (`handler.rs:219`) — per-tenant expected relay URL.
- `crate::api::relay_members::enforce_relay_membership` (`handler.rs:244-262`) —
  no-op unless `require_relay_membership` is on (`api/mod.rs:67`, `:130-131`).

Failure: any verifier rejection → WS `{"type":"error","message":"auth failed"}` and
close. No retry, no second challenge.

---

#### 6. `buzz-conformance`

Consumed as a pure schema + trait crate. `conformance/mod.rs:38-44` re-exports 11
items; `tracers.rs:12` imports `TraceStep` and `Tracer`.

| Direction | Detail |
|---|---|
| relay → crate | `AppState.tracer: Arc<dyn buzz_conformance::Tracer>` (`state.rs:620`); every `record` call |
| crate → relay | nothing — the checker (`buzz-conformance/src/checker.rs`) consumes JSONL offline |

Failure modes: **none can reach a request path.** `Tracer::record` returns `()`
(`buzz-conformance/src/lib.rs:317`), so an emit cannot fail, cannot block, and
cannot apply backpressure. `JsonlTracer` swallows every I/O error
(`tracers.rs:63-71`) and recovers a poisoned mutex via `into_inner`
(`tracers.rs:57-60`). Because production binds `NoopTracer` (`state.rs:798`), the
integration is inert at runtime — see features §4.1.

One duplication: `buzz_conformance::NoopTracer` (`buzz-conformance/src/lib.rs:323`)
exists and has **zero users**; the relay defines and uses its own
(`tracers.rs:20-24`).

---

#### 7. `mesh-llm` — a name collision, not a mesh integration

`crates/buzz-relay/Cargo.toml:87-88` pins two **dev-dependencies** to
`git = "https://github.com/Mesh-LLM/mesh-llm.git", tag = "v0.73.1"`:
`mesh-llm-sdk` (features `client`, `serving`) and `mesh-llm-host-runtime`
(feature `dynamic-native-runtime`).

Consumers are exactly the five files in `crates/buzz-relay/examples/`:

| Example | Uses |
|---|---|
| `mesh_agent_e2e.rs` | `mesh_llm_sdk::{serve, MeshDiscoveryMode}` (`:25`), `mesh_llm_host_runtime::initialize_host_runtime` (`:48`) |
| `mesh_serve_client_smoke.rs` | `mesh_llm_sdk::{client, serve, MeshDiscoveryMode}` (`:29`), `native_runtime_cache` / `CURRENT_MESH_VERSION` (`:44-45`), `initialize_host_runtime` (`:53`) |
| `mesh_serve_smoke.rs` | `mesh_llm_sdk::{serve, MeshDiscoveryMode}` (`:13`) |
| `mesh_stack_smoke.rs` | `mesh_llm_host_runtime::models::download_model_ref_with_progress_details` (`:53`) |
| `mesh_admission_smoke.rs` | process-global mesh-llm state note (`:16`); no direct import |

This is **local LLM inference / model serving**, entirely unrelated to
`buzz-relay-mesh` (the inter-relay QUIC mesh in this group). Two different things
called "mesh" inside one crate, both spelled `mesh_*` in file names. Risk profile:

- A `git`-pinned dependency by **tag** (not commit SHA) — tags are mutable, so the
  build is not reproducible against a retagged upstream.
- Present in `[dev-dependencies]`, so it does not ship in the relay binary, but it
  **does** enter the dev/CI dependency graph and lockfile for anyone running
  `cargo test -p buzz-relay`.
- `mesh_stack_smoke.rs:31` requires manual sync with
  `buzz_lib::mesh_llm::MESH_WORKER_STACK_SIZE` in the desktop crate — a
  cross-crate constant duplicated by comment, not by code.

---

#### 8. Inbound/outbound integration matrix for one cross-pod huddle frame

```
Client A (pod 1, non-owner)                        Client B (pod 2, OWNER)
   │ WS binary [v2 hdr][Opus]                          
   ├──────────────────────────────► handler.rs:1019 forward_media
   │                                 join.rs:1758 media_datagram
   │                                 [ownerIdx][v2 hdr][Opus] + FencedHeader
   │                                 transport.send_datagram ──► iroh/QUIC
   │                                                              │
   │                        mesh_boot.rs:242 dispatcher.on_datagram
   │                        mesh.rs:212 GenerationFloor::check   ◄─┘
   │                        mesh.rs:221 get_unambiguous_by_channel
   │                        mesh.rs:247 room.deliver_prefixed
   │                                     └─► B's audio_tx ─► WS binary
   │
   │  B speaks: room.broadcast_frame (room.rs:398) puts the prefixed frame
   │  into A's *remote* AudioPeer.audio_tx, drained by
   │  spawn_remote_peer_sink (mesh.rs:262) ─► datagram ─► pod 1
   │  ─► mesh.rs:247 deliver_prefixed ─► A's WS
```

Redis is consulted **only** at join (`resolve_join_owner_ready` → `owner_of` /
`acquire` / `validate`), at owner-side `RegisterPeer` (`validate`), and every 10 s
by the renewer. It is **never** consulted per media frame.
