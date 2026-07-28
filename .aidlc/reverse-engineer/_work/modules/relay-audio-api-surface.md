## Module: buzz-relay — huddle audio, tunnel & conformance seam (`crates/buzz-relay/src`)
### Aspect: API Surface

---

#### 1. HTTP / WebSocket routes owned by this group

| Route | Method | Handler | Registered | Auth |
|---|---|---|---|---|
| `/huddle/{channel_id}/audio` | GET (WS upgrade) | `audio::handler::ws_audio_handler` | `router.rs:125-128` | host→tenant bind, then in-band NIP-42 |
| `/_mesh/demo/echo` | POST | `api::mesh_demo::demo_echo` | `router.rs:123` | **none** — gated only on `BUZZ_MESH=on` + `BUZZ_MESH_DEMO_ECHO=on`, else 404 |
| `/_mesh` | GET | `mesh_status_handler` (`router.rs:399-406`) | `router.rs:230` (health router) | **none** — health router has "no metrics middleware, no auth, no CORS, no body limit" (`router.rs:222-224`) |

`/huddle/…/audio` sits on `api_router`, which carries
`RequestBodyLimitLayer::new(1024*1024)` (`router.rs:130`) — irrelevant to a WS
upgrade — plus the shared metrics/trace/CORS layers (`router.rs:186-189`).

The `{channel_id}` path segment is extracted as `Path<Uuid>` (`handler.rs:67`), so a
non-UUID segment is rejected by axum with 400 before the handler runs.

---

#### 2. `GET /huddle/{channel_id}/audio` — full frame inventory

##### 2.1 Pre-upgrade rejections (HTTP status, no WebSocket)

| Status | Body | Condition | Line |
|---|---|---|---|
| 404 | `relay: no community is configured for this host` | `tenant::bind_community(&state.db, Host)` errs — unmapped host or DB failure. Deliberately generic so an anonymous caller cannot enumerate communities (`handler.rs:69-73`) | `handler.rs:80-88` |
| 503 | `relay: connection limit reached` | `conn_semaphore.try_acquire_owned()` fails — **shared with ordinary relay WebSockets** | `handler.rs:90-99` |
| — | (silent return) | `state.db.is_community_active(community)` false, via `run_registered_community_connection` | `handler.rs:156-164` |

Parser bounds installed **before** upgrade: `max_message_size` and `max_frame_size`
both `MAX_WEBSOCKET_MESSAGE_BYTES = 8192` (`handler.rs:52`, applied
`handler.rs:116-119`, `:105`). Pinned by
`audio_websocket_parser_rejects_oversized_messages_before_handler_reads_them`
(`handler.rs:1417-1427`).

##### 2.2 Inbound frames (client → relay)

| Frame | Shape / condition | Handling | Line |
|---|---|---|---|
| Text `{"type":"auth", …}` | Required first; anything else is ignored while the 5 s `AUTH_TIMEOUT` runs. `>8192` chars → warn + `continue` (still inside the window) | `AuthMsg` deserialize | `handler.rs:126-142`, `:188-214` |
| Text `{"type":"leave"}` | any valid JSON with `type == "leave"` | breaks `recv_loop` → clean teardown | `handler.rs:1030-1036` |
| Text other | valid or invalid JSON, `≤8192` | **silently ignored** (no error frame) | `handler.rs:1030-1036` |
| Text `>8192` (post-auth) | | warn + drop; connection survives | `handler.rs:1026-1029` |
| Binary | `≤ MAX_AUDIO_FRAME_BYTES = 4096` (`handler.rs:44`). If room pin `≥2`: must be `> 8` bytes and `FrameHeader::parse` must yield a non-empty payload | forwarded opaquely (owner path: `room.broadcast_frame`; ingress path: `session.forward_media`) | `handler.rs:960-1023` |
| Binary `>4096` | | warn + drop; connection survives | `handler.rs:961-964` |
| Ping | | `Pong` echoed via the **priority** control channel (`try_send`, dropped if that 8-slot channel is full) | `handler.rs:1041-1044` |
| Pong | | resets `missed_pongs` to 0 | `handler.rs:1038-1040` |
| Close / stream end | | breaks `recv_loop` | `handler.rs:1045` |

`AuthMsg` (`handler.rs:126-139`):

| Field | Type | Required | Default |
|---|---|---|---|
| `type` | String | yes, must equal `"auth"` | — |
| `event` | `nostr::Event` | yes | — |
| `parent_channel_id` | `Option<Uuid>` | only for TTL channels | `None` |
| `protocol_version` | u8 | no | **1** (`handler.rs:141-142`) |

Note the size window: the parser admits 8192-byte binary frames but the handler
drops anything over 4096 — so 4097..8192 binary bytes are accepted by tungstenite,
buffered, then discarded.

##### 2.3 Outbound frames (relay → client)

Every outbound text frame is a JSON object with a `type`. **6 distinct `type`
values**: `challenge`, `error`, `joined`, `left`, `roster`, plus WS control frames.

| `type` | Fields | Emitted when | Line |
|---|---|---|---|
| `challenge` | `challenge: String` (`buzz_auth::generate_challenge`) | first frame after upgrade, unconditionally | `handler.rs:176-186` |
| `joined` | `pubkey`, `peer_index`, `peers: [{pubkey, peer_index}]` | after successful admission. **Owner/local path broadcasts to the whole room** (`handler.rs:641`); ingress path sends **only to the joining socket** (`handler.rs:628-632`) | `handler.rs:620-643` |
| `left` | `pubkey`, `peer_index` | on disconnect, owner/local path only (`remote_session.is_none()`, `handler.rs:818-820`); owner also fans `left` for remote peers on `UnregisterPeer` (`join.rs:1301-1310`) and on control-stream teardown (`join.rs:1354-1366`) | `handler.rs:812-820` |
| `joined` (revision-bearing) | `revision`, `pubkey`, `peer_index`, `peers` | ingress path only, forwarded from an owner `RosterDelta` | `join.rs:1570-1576` |
| `left` (revision-bearing) | `revision`, `pubkey`, `peer_index` | ingress path only, from an owner `RosterDelta` | `join.rs:1577-1581` |
| `roster` | `revision`, `peers: [{pubkey, peer_index}]` | ingress path only, from an owner `RosterSnapshot` (initial or post-resync) | `join.rs:1552-1558` |

##### 2.4 Outbound `error` frames — exact conditions

Eleven distinct error emissions; only 8 carry a machine-readable `code`.

| `code` | `message` | Condition | Line |
|---|---|---|---|
| *(none)* | `auth failed` | `state.auth.verify_auth_event` rejects (bad sig, wrong challenge, wrong relay URL, allowlist/token policy) | `handler.rs:226-236` |
| *(none)* | `restricted: not a relay member` | `api::relay_members::enforce_relay_membership` errs — **no-op when `require_relay_membership=false`, the default** (`api/mod.rs:67`, `:130-131`) | `handler.rs:244-262` |
| *(none)* | `not a member` | `ensure_membership` errs: DB error, archived channel, missing/unlinked `parent_channel_id`, non-member of a non-open channel | `handler.rs:265-286` |
| *(none)* | `huddle has ended` | post-`get_or_create` re-check found `archived_at.is_some()` | `handler.rs:389-403` |
| `huddle_relay_draining` | `relay is draining; reconnect` | mesh on and `HuddleOwnerRegistry::is_draining()` | `handler.rs:308-320` |
| `join_rejected` | `huddle join rejected` | `resolve_join_owner_ready` errs (fence rejection, Redis error, or the 25-attempt owner-ready loop exhausting) | `handler.rs:342-355` |
| `huddle_audio_unavailable` | `huddle audio unavailable in this deployment` | mesh **off** and `huddle_audio_available == false` | `handler.rs:356-375` |
| `unsupported_version` | `huddle audio protocol v{n} not supported; relay max is v2`, plus `current_version` | `requested_version == 0 \|\| > 2` | `handler.rs:417-441` |
| `huddle_owner_unreachable` | `could not reach the huddle owner` | `DialError::Mesh` from `dial_remote_owner` | `handler.rs:487-503` |
| `room_full` | `peer index space exhausted` | `AdmissionError::Full` (soft cap 25 **or** index exhaustion) | `handler.rs:513-522`; cross-pod mirror `handler.rs:917-921` |
| `room_ended` | `huddle has ended` | `AdmissionError::Ended` | `handler.rs:523-533`; mirror `handler.rs:922-924` |
| `upgrade_required` | `this huddle is using audio protocol v{pinned}; your client requested v{requested}`, plus `pinned_version`, `requested_version` | `AdmissionError::VersionMismatch` | `handler.rs:534-548`; mirror `handler.rs:925-932` |
| `join_rejected` + `fence_reason` | `huddle join rejected`, `fence_reason ∈ {stale_generation, no_active_lease, owner_mismatch, future_generation}` | owner replied `RegisterRejected(Fenced(..))` | `handler.rs:933-937` |

`remote_rejection_ws_error` (`handler.rs:915-939`) is the cross-pod mirror: it
reproduces the same `code`s a same-pod join emits, so a client cannot tell which
pod owned the room.

##### 2.5 Silent teardowns (no `error` frame)

| Condition | Line |
|---|---|
| challenge send fails | `handler.rs:180-185` |
| 5 s auth timeout or disconnect before auth | `handler.rs:207-214` |
| `get_channel` pre-join check returns `Err` (fail-closed) | `handler.rs:404-410` |
| `joined` send fails on the ingress path | `handler.rs:628-636` |
| 3 missed pongs (`MAX_MISSED_PONGS`, 30 s `HEARTBEAT_INTERVAL`) → `cancel` → `send_loop` sends bare `Close(None)` | `handler.rs:1127-1151`, `:1066-1069` |
| owner lost/draining (owner path) → `cancel` + `fence.forget` | `handler.rs:727-772` |
| owner `Goodbye`/stream close (ingress path) → `teardown_remote_huddle` | `handler.rs:707-714`, `:897-910` |
| community deactivated mid-connection | `state.rs` connection registry via `handler.rs:156` |

---

#### 3. Huddle event kinds

| Kind | Constant | Producer in this group | Not produced here |
|---|---|---|---|
| 48100 `KIND_HUDDLE_STARTED` | `buzz-core/src/kind.rs:454` | **none** — the relay only *reads* it, via `db.huddle_started_link_exists` (`handler.rs:1176-1186`) to validate a claimed parent. Produced by the desktop client (`desktop/src-tauri/src/huddle/mod.rs:252`) | |
| 48101 `KIND_HUDDLE_PARTICIPANT_JOINED` | `kind.rs:456` | `handler.rs:645-653` — after successful admission, on **every** path (owner and ingress) | |
| 48102 `KIND_HUDDLE_PARTICIPANT_LEFT` | `kind.rs:458` | `handler.rs:822-831` — after peer removal, unconditionally | |
| 48103 `KIND_HUDDLE_ENDED` | `kind.rs:460` | `handler.rs:850-859` — only when `should_auto_end && archive_channel` succeeded | |
| 48106 `KIND_HUDDLE_GUIDELINES` | `kind.rs:462` | **none in this group.** Only relay reference is the kind-label allowlist `handlers/event.rs:49`. Produced client-side (`desktop/src-tauri/src/huddle/agents.rs:31`) | |

All three relay-emitted events are signed with `state.relay_keypair` (`handler.rs:1268`),
so their author is the relay, not the participant. Tags:
`h = parent_channel_id`, `p = participant_pubkey` (`handler.rs:1240-1256`); content
is `{"ephemeral_channel_id": "<uuid>"}` (`handler.rs:1238`). The `parent_channel_id`
is the ephemeral channel's *parent* for TTL channels and the channel itself
otherwise (`handler.rs:1170-1194`).

Emission is a 4-step pipeline (`handler.rs:1274-1332`): persist
(`db.insert_event`) → `mark_local_event` → `fan_out_event_to_local_subscribers` →
`pubsub.publish_event`. On a duplicate insert, fan-out is **skipped entirely**
(`handler.rs:1285-1295`). On an insert error, the event is still fanned out from an
in-memory `StoredEvent::new` (`handler.rs:1296-1307`) — live subscribers see it,
late joiners never will. On a publish error, `local_event_ids` is invalidated
(`handler.rs:1326-1330`).

---

#### 4. `HuddleControl` mesh stream API

Opened by `dial_remote_owner` (`join.rs:1660-1724`) with
`StreamHello{sender: local_runtime_id, role: Session{fenced, profile: HuddleControl}}`.
Accepted by `HuddleControlAcceptor::accept_inbound` (`join.rs:1054-1093`).

##### 4.1 `accept_inbound` structural validation (before any state touch)

| Check | Rejection | Line |
|---|---|---|
| `hello.sender == from` (authenticated peer) | `MeshError::Transport` | `join.rs:1060-1065` |
| role is `Session` | `MeshError::Transport` | `join.rs:1066-1070` |
| `profile == HuddleControl` | `MeshError::Transport` | `join.rs:1071-1075` |
| `fenced.owner_runtime_id == self.local_runtime_id` | `MeshError::OwnerMismatch` | `join.rs:1079-1086` |

Deliberately **not** Redis-fenced here — the fence key needs the community, which
only arrives on the first `RegisterPeer` (`join.rs:1032-1041`).

##### 4.2 `serve_control_loop` frame handling (`join.rs:1115-1370`)

| Inbound | Precondition | Effect |
|---|---|---|
| `Data{fenced: f, payload}` where `f != fenced` | — | break `Err(OwnerMismatch)` (`join.rs:1191-1198`) |
| `RegisterPeer{community_id, pubkey, protocol_version}` | community latched on first frame; a later frame naming a different community → break `Err(Transport)` (`join.rs:1200-1209`) | `is_draining()` → `Goodbye(Draining)` (`:1219-1222`); else `get_or_create` room, `subscribe_roster`, then **`directory.validate(community, fenced)` before `add_peer`** (`:1231-1245`). Fence error classifiable → `RegisterRejected(Fenced(..))`; non-fence error tears the stream down |
| `UnregisterPeer{pubkey}` | pubkey present in this stream's `registered` map | `room.remove_peer` + `left` fan-out to local peers. **No fence** — cannot touch another community's room (`join.rs:1264-1288`) |
| `RosterResync` | community latched | replies `RosterSnapshot` (`join.rs:1290-1300`) |
| `PeerRegistered` / `RosterSnapshot` / `RosterDelta` / `RegisterRejected` | — | break `Err(Transport)` — owner→non-owner replies are a protocol violation on the accept side (`join.rs:1302-1310`) |
| `Goodbye{..}` or clean close | — | break `Ok(())` (`join.rs:1183`) |
| any other frame (`Hello`, `Gossip`) | — | break `Err(Transport)` (`join.rs:1184-1188`) |

Owner-initiated arms: `draining` fires → `Goodbye(Draining)`; `lost` fires →
`Goodbye(StaleGeneration)` (`join.rs:1157-1166`, sent at `join.rs:1315-1321`).
A `roster_rx` `Lagged` recovers by sending a fresh `RosterSnapshot`
(`join.rs:1174-1182`); `Closed` breaks the loop.

Teardown always drops every peer this stream registered, regardless of exit path
(`join.rs:1345-1367`).

##### 4.3 Ingress-side reader `read_owner_control` (`join.rs:1527-1612`)

| Owner frame | Ingress action |
|---|---|
| `Goodbye{reason}` | return `HuddleTeardownCause::from_goodbye` (`join.rs:1497-1504`) |
| `RosterSnapshot{revision, peers}` | set `revision`, emit WS `roster` |
| `RosterDelta` where `next == revision+1` | advance, emit WS `joined`/`left` |
| `RosterDelta` where `next <= revision` | ignore (`join.rs:1583`) |
| `RosterDelta` with a gap | send `RosterResync` upstream, **do not apply** (`join.rs:1584-1597`) |
| any other decoded msg | ignore (`join.rs:1598`) |
| decode error | `debug!` and keep reading (`join.rs:1599`) |
| clean close / transport error | `StreamClosed` |

Pinned end-to-end by `roster_revision_gap_requests_resync_before_forwarding_new_state`
(`join.rs:2113-2183`).

---

#### 5. Public Rust API and its callers

##### 5.1 `audio` module

| Item | Line | Production callers |
|---|---|---|
| `ws_audio_handler` | `handler.rs:64` | `router.rs:127` |
| `AudioRoomManager::{new, get_or_create, get, get_unambiguous_by_channel, cleanup_if_empty}` | `room.rs:496-550` | `state.rs:768`; `handler.rs:380`, `:401`, `:408`, `:484`, `:501`, `:637`, `:849`, `:865`; `join.rs:1176`, `:1229`, `:1266`, `:1291`, `:1347`; `mesh.rs:221` |
| `Room::{add_peer, add_peer_at_index, remove_peer, remove_peer_and_check_ended, broadcast_frame, deliver_prefixed, broadcast_control, subscribe_roster, roster_snapshot, peer_pubkeys, is_empty, clear_ended}` | `room.rs:228-487` | all reached from `handler.rs` / `join.rs` / `mesh.rs` |
| `Room::mark_ended` | `room.rs:192` | **zero production callers** — only `room.rs:660` (test). Production ends rooms via `remove_peer_and_check_ended` |
| `FrameHeader::{parse, is_dtx}`, `V2_HEADER_LEN`, `FLAG_DTX` | `wire.rs:29-88` | `handler.rs:948`, `:983`, `:986`, `:1001`. `FLAG_DTX` itself is referenced only in `wire.rs` (relay side) — the desktop has its own copy (`desktop/src-tauri/src/huddle/wire.rs:48`) |
| `GenerationFloor::{new, check, forget}`, `FenceVerdict` | `mesh.rs:93-146` | `mesh_boot.rs:516`; `mesh.rs:214`; `handler.rs:755`, `:763`, `:909` |
| `MeshAudioRouter::{with_fence, on_media_datagram}` | `mesh.rs:180-250` | `mesh_boot.rs:239-245` |
| `MeshAudioRouter::{new, fence, local_runtime_id}` | `mesh.rs:169`, `:196`, `:201` | **zero production callers** (`new` only in `mesh.rs` tests) |
| `HuddleOwnerDirectory` trait + `mesh::Ownership` | `mesh.rs:67-80` | **zero implementors, zero callers anywhere** — fully dead |
| `spawn_remote_peer_sink` | `mesh.rs:262` | `join.rs:1391` |

##### 5.2 `audio::join` module

| Item | Line | Production callers |
|---|---|---|
| `HuddleDirectory` trait (5 methods) | `join.rs:66-101` | impl for `SessionDirectory` at `join.rs:107-183`; used by `resolve_join`, the renewer, `HuddleControlAcceptor` |
| `resolve_join` | `join.rs:317` | `join.rs:426` only (via `resolve_join_owner_ready`) — no direct production caller |
| `resolve_join_owner_ready` | `join.rs:416` | `handler.rs:322` |
| `dial_remote_owner` | `join.rs:1660` | `handler.rs:459` |
| `send_clean_close` | `join.rs:1770` | `handler.rs:520`, `:530`, `:544`, `:716` |
| `read_owner_control` | `join.rs:1527` | `handler.rs:707` |
| `read_teardown_cause` | `join.rs:1623` | **zero production callers** — 4 test callers only (`join.rs:2948`, `:2981`, `:3007`). Superseded by `read_owner_control` |
| `HuddleOwnerRegistry::{new, is_draining, lost_for, drain_for, attach_signals, release, drain, drain_all}` | `join.rs:636-773` | `mesh_boot.rs:474`; `handler.rs:308`, `:582`, `:588`, `:589`, `:876`; `join.rs:1089`, `:1090`, `:1219`; `mesh_boot.rs:489` |
| `HuddleOwnerRegistry::attach` | `join.rs:659` | **zero production callers** — thin wrapper over `attach_signals`; 5 test callers |
| `HuddleOwnerRegistry::drain` | `join.rs:750` | reached only through `drain_all` (`join.rs:772`) |
| `spawn_observable_huddle_renewer` | `join.rs:482` | `join.rs:674`, `:688`, `:695` (all inside `attach_signals`) — no external caller |
| `HuddleLeaseRenewer` (`.task`, `.lost`) | `join.rs:464-471` | `.lost` cloned at `join.rs:694`; **`.task` is never awaited in production** — the struct is dropped, detaching the task |
| `HuddleControlAcceptor::{new, accept_inbound}` | `join.rs:1013`, `:1054` | `mesh_boot.rs:262-268`, `:277` |
| `encode_control` / `decode_control` | `join.rs:1006`, `:1011` | `join.rs` internally (7 sites) |
| `FenceRejection::{from_mesh_error, code}` | `join.rs:996`, `:1013` | `join.rs:1239`; `handler.rs:936` |
| `JoinOutcome::fenced_header` | `join.rs:272` | `handler.rs:452` |
| `RemoteHuddleSession::{peer_index, roster, fenced, pubkey, forward_media}` | `join.rs:1728-1766` | `handler.rs:508`, `:609`, `:520`, `:694`, `:1019` |
| `HUDDLE_CONTROL_PROFILE` | `join.rs:1027` | `join.rs:143` |
| `HUDDLE_SESSION_ENDED` | `join.rs:1451` | `join.rs:1781` |

##### 5.3 `tunnel` module

`tunnel/mod.rs:1-8` exports exactly two submodules: `directory`, `reliable`.

| Item | Line | Production callers |
|---|---|---|
| `SessionDirectory::{new, with_lease_ttl, acquire, renew, release, lookup, validate_fenced_header}` | `directory.rs:180-439` | `mesh_boot.rs:512`; `join.rs:110-183`; `reliable.rs:87`, `:383` |
| `SessionDirectory::takeover` | `directory.rs:233` | **zero callers anywhere** (delegates to `acquire`) |
| `SessionDirectory::known_generation` | `directory.rs:324` | **zero production callers** — 2 test callers (`directory.rs:695`, `:764`) |
| `SessionLease::fenced_header` | `directory.rs:444` | `reliable.rs:117`; tests |
| `ReliableStreamRouter::{new, directory, local_runtime_id, join, accept_inbound}` | `reliable.rs:50-172` | `mesh_boot.rs:283-287`, `:290`; `api/mesh_demo.rs:73`, `:95`, `:239`+ |
| `ReliableStreamRouter::{spawn_renewer, spawn_observable_renewer}` | `reliable.rs:179`, `:192` | **zero callers.** `mesh_boot.rs:212-215` explicitly documents that renewal is not wired yet |
| `ReliableMeshStream::{new, fenced, send_bytes, send_goodbye, finish, recv_validated, community_id}` | `reliable.rs:243-359` | `mesh_boot.rs:330-357`; `api/mesh_demo.rs:110`, `:116` |
| `ReliableMeshStream::{new_inbound, with_community}` | `reliable.rs:253`, `:263` | `new_inbound` from `reliable.rs:170`; **`with_community` has zero callers** |
| `MAX_RELIABLE_PAYLOAD_BYTES` | `reliable.rs:31` | `reliable.rs:280` |
| `ReliableJoin`, `ReliableInbound`, `ReliableFrame`, `ReliableStreamError` | `reliable.rs:207-568` | `mesh_boot.rs`, `api/mesh_demo.rs` |

##### 5.4 `mesh_boot` module

| Item | Line | Production callers |
|---|---|---|
| `boot_mesh` | `mesh_boot.rs:411` | `main.rs:442` |
| `MeshHandle::wire_consumers` | `mesh_boot.rs:180` | `main.rs:454-458` |
| `MeshHandle::status` | `mesh_boot.rs:172` | `router.rs:401` |
| `MeshHandle` public fields (`directory`, `transport`, `membership`, `local_runtime_id`, `dispatcher`, `audio_fence`, `owners`) | `mesh_boot.rs:137-168` | `handler.rs`, `api/mesh_demo.rs` |
| `MeshHandle.membership` | `mesh_boot.rs:141` | **zero readers** — populated at `mesh_boot.rs:501` but never used by any consumer; `/_mesh` goes through the private `runtime` field instead |
| `wire_mesh_consumers` | `mesh_boot.rs:224` | `mesh_boot.rs:183` + 1 test |
| `MeshInboundDispatcher::register_{huddle_control,reliable_stream,datagrams}` | `mesh_boot.rs:70-88` | `mesh_boot.rs:242`, `:269`, `:288` |
| `run_demo_echo` (`pub(crate)`) | `mesh_boot.rs:307` | `mesh_boot.rs:294`; `api/mesh_demo.rs:294` (test) |
| `SessionStreamHandler`, `DatagramHandler` type aliases | `mesh_boot.rs:39`, `:42` | internal |

`main.rs:455-459` wires consumers **before** `state.mesh.set(handle)`, so inbound
mesh traffic can be served before any local join can resolve ownership.
`main.rs:459` is `unreachable!("mesh handle is set exactly once, right here")`.

##### 5.5 `conformance` module — the `Tracer` surface

`Tracer` (`buzz-conformance/src/lib.rs:314-318`) is a single-method trait:
`fn record(&self, step: TraceStep)`. It is `Send + Sync`, takes `&self`, returns
nothing, and cannot fail — so an emit can never propagate an error into a request
path, and cannot apply backpressure.

| Impl | Line | Behaviour |
|---|---|---|
| `NoopTracer` | `tracers.rs:20-24` | discards; **this is the production binding** (`state.rs:798`) |
| `JsonlTracer` | `tracers.rs:30-72` | truncate-on-create, one JSON line per step, `flush()` per record, `Mutex` poisoning recovered via `into_inner` |
| `CountingTracer` (private) | `conformance/mod.rs:357-372` | increments then delegates |
| `buzz_conformance::NoopTracer` | `buzz-conformance/src/lib.rs:323` | **zero users** |

**`JsonlTracer` has zero callers in the entire workspace** — no production site, no
test, no integration harness. The only references are its own definition and two
doc comments (`state.rs:616`, `:795`) promising that "conformance tests bind" it.

Public helpers and their callers:

| Helper | Line | Callers |
|---|---|---|
| `state_for_request` | `:55` | `ingest.rs:145`, `:1381`, `:1801`, `:2195`, `:2348`, `:2485`, `:2547`; `req.rs:118` |
| `msg_id_label` | `:78` | `ingest.rs:142`, `:2192`, `:2337`, `:2343`, `:2471`, `:2476`, `:2481` |
| `channel_label` | `:89` | `conformance/mod.rs:154`, `:279`, `:314`; tests |
| `claimed_community_from_event` | `:101` | `ingest.rs:143`, `:1788`, `:2193`, `:2334`, `:2468` |
| `step` | `:121` | **zero callers** |
| `emit` | `:127` | `ingest.rs:1414`, `:2348`, `:2485` |
| `record_req_authcheck` | `:148` | `req.rs:144` |
| `project_row_communities` | `:234` | `conformance/mod.rs:281`, `:316`; tests |
| `record_read_message_rows` | `:265` | `req.rs:355` |
| `record_read_by_id_rows` | `:300` | `req.rs:671` |
| `EmitGuard::arm` | `:383` | `ingest.rs:1382`, `:2550` (test) |
| `sanitized_reason_for` | `:422` | `ingest.rs:1411` |
| `RowCommunityProjection` | `:216` | internal + tests |

The `EmitGuard` contract: `arm(tracer, state, kind)` returns
`(guard, counting_tracer)`; the caller **must** thread the returned wrapper
downstream. `ingest_event` does exactly this (`ingest.rs:1382-1387`), passing
`&tracer` into `ingest_event_inner`. On `Drop` with zero records, the guard emits
`ImplBug{kind}` onto the *original* tracer — which in production is `NoopTracer`,
so the breach is discarded.

---

#### 6. What this group deliberately does **not** expose

- No HTTP endpoint returns huddle room state. There is no `GET /huddle/{id}/peers`,
  no participant count, no roster REST surface. Room state is only observable by
  joining the WebSocket or by replaying kinds 48101/48102/48103.
- No admin/operator surface for huddles: no force-end, no kick, no mute, no
  capacity override. `buzz-admin` has no huddle subcommand.
- No `buzz-cli` subcommand touches `/huddle/…/audio` (grep across `crates/buzz-cli`
  finds no huddle route).
- The tunnel lane has **no product-facing API**: the only route reaching
  `ReliableStreamRouter::join` is the demo-gated `POST /_mesh/demo/echo`
  (`api/mesh_demo.rs:73`), documented at `mesh_boot.rs:206-215` as "no product
  session consumer is wired yet".
