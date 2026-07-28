## Module: buzz-relay — huddle audio, tunnel & conformance seam (`crates/buzz-relay/src`)
### Aspect: Data Model

Scope: `audio/{mod,wire,room,mesh,join,handler}.rs`, `tunnel/{mod,directory,reliable}.rs`, `mesh_boot.rs`, `conformance/{mod,tracers}.rs` (9,266 LOC verified line-by-line).

---

#### 1. Audio room state — in-process only

Nothing in this module is persisted. Every structure below lives in relay process
memory and is lost on restart. The only durable artefacts are the four Nostr
lifecycle events (`48101`/`48102`/`48103`, see api-surface) written through
`buzz-db`, and the Redis lease/generation keys in §5.

##### 1.1 Registry → Room → Peer

| Type | Definition | Key / shape | Notes |
|---|---|---|---|
| `AudioRoomManager` | `audio/room.rs:490-492` | `DashMap<(CommunityId, Uuid), Arc<Room>>` | Single global instance, `state.rs:571`, constructed `state.rs:768`. Key is **(community, channel)** — `room.rs:503-509` doc states channel UUIDs are only unique inside a community |
| `Room` | `audio/room.rs:157-170` | `community_id`, `channel_id`, `peers: DashMap<Uuid, AudioPeer>`, `guard: std::sync::Mutex<AdmissionGuard>`, `roster_tx: broadcast::Sender<RosterDelta>` (cap **64**, `room.rs:179`) | Room UUID key is the *channel* id; peer map key is a fresh `Uuid::new_v4()` per connection (`room.rs:255`, `:314`) |
| `AudioPeer` | `audio/room.rs:20-30` | `pubkey: String` (hex), `audio_tx: mpsc::Sender<Bytes>`, `ctrl_tx: mpsc::Sender<PeerCtrl>`, `peer_index: u8` | Two channels per peer so control is never starved by audio (`room.rs:24-27`) |
| `PeerCtrl` | `audio/room.rs:31-36` | `Json(String)` \| `Close` | **`Close` has zero producers** in the whole workspace; consumed only at `audio/handler.rs:1112` |
| `AdmissionGuard` | `audio/room.rs:97-131` | `next_fresh: u8`, `free: Vec<u8>`, `ended: bool`, `pinned_version: Option<u8>`, `roster_revision: u64` | The single mutex that serialises admission, ending, version pinning and roster revisioning |

##### 1.2 Capacities (all hard-coded)

| Constant | Value | Line | Meaning |
|---|---|---|---|
| `AUDIO_CHANNEL_CAPACITY` | 8 | `room.rs:40` | 160 ms at 20 ms/frame; `try_send`, drop-on-full |
| `CTRL_CHANNEL_CAPACITY` | 32 | `room.rs:45` | state-bearing; overflow logs a warning (`room.rs:441-446`) |
| `MAX_PEERS_PER_ROOM` | 25 | `room.rs:50` | soft cap; the doc math is `N×(N−1)` copies/tick → 600 at 25 |
| index space | 0..=254 usable | `room.rs:146-152` | `alloc()` returns `None` once `next_fresh == 255`, so index **255 is never issued** |
| `roster_tx` broadcast | 64 | `room.rs:179` | lag is recoverable via `roster_snapshot` |

##### 1.3 Room lifecycle state machine

State is the tuple `(peers.len(), guard.ended, guard.pinned_version, guard.roster_revision)`.

| From | Event | To | Code |
|---|---|---|---|
| absent | `get_or_create` | `(0, false, None, 0)` | `room.rs:503-509` |
| `(n, false, pin, r)` | `add_peer(v)` ok | `(n+1, false, pin∨v, r+1)` | `room.rs:228-276` |
| `(n, false, pin, r)` | `add_peer_at_index(v,i)` ok | `(n+1, false, pin∨v, r+1)` | `room.rs:281-334` |
| `(n, _, _, r)` | `remove_peer` | `(n−1, unchanged, pin, r+1)` | `room.rs:337-355` |
| `(1, false, …, r)` | `remove_peer_and_check_ended` | `(0, **true**, pin, r+1)` + `should_end=true` | `room.rs:362-388` |
| `(n, false, …)` | `mark_ended` | `(n, true, …)`, returns `peers.is_empty()` | `room.rs:192-199` |
| `(n, true, …)` | `clear_ended` (archive rollback) | `(n, false, …)` | `room.rs:201-205`, called `handler.rs:843` |
| `(0, *, *, *)` | `cleanup_if_empty` | absent (pin cleared) | `room.rs:545-550` |

`ended` is **sticky except via `clear_ended`**. `pinned_version` survives peer churn
(pinned by test `room.rs:730-757`) and is only cleared by eviction. `roster_revision`
uses `wrapping_add(1)` (`room.rs:257`, `:322`, `:342`, `:369`) — it can wrap to 0
after 2^64 mutations, at which point the ingress gap detector
(`join.rs:1568` `next == revision.wrapping_add(1)`) still matches, so the wrap is
consistent, not a bug.

##### 1.4 Roster value types (two parallel definitions)

| Room-local | Wire | Fields |
|---|---|---|
| `RosterPeer` `room.rs:55-61` | `RosterEntry` `join.rs:930-936` | `pubkey: String`, `peer_index: u8` |
| `RosterSnapshot` `room.rs:64-70` | `RosterSnapshot` `join.rs:938-945` | `revision: u64`, `peers: Vec<…>` |
| `RosterDelta` `room.rs:73-81` | `HuddleControlMsg::RosterDelta` `join.rs:895-903` | `revision`, `joined: Option<…>`, `left: Option<…>` |

Only `RosterPeer → RosterEntry` has a conversion (`join.rs:947-954`); the
snapshot/delta conversions are hand-written at `join.rs:1414-1433`. The room-local
types are **not** `Serialize`; the wire types are. This duplication is deliberate
(the room never learns about the mesh, `mesh.rs:38-42`) but is uncovered by any
test that would catch field drift between the two.

`roster_snapshot()` sorts by `peer_index` (`room.rs:471`); `peer_pubkeys()` does
**not** sort (`room.rs:479-484`) — so the `joined.peers` array a same-pod client
receives is in `DashMap` iteration order, while a cross-pod client's is sorted.

---

#### 2. Audio wire frame format

##### 2.1 Client → relay binary frame (protocol v2)

`audio/wire.rs:22-33` defines the v2 header; `V2_HEADER_LEN = 8`, `FLAG_DTX = 0x01`.

| Offset | Width | Field | Encoding | Trust |
|---|---|---|---|---|
| 0..=1 | u16 | `seq` | big-endian, wraps at 2^16 | client-authored, unvalidated |
| 2..=5 | u32 | `ts_48k` | big-endian, 48 kHz media clock | client-authored, unvalidated |
| 6 | i8 | `level_dbov` | canonical `-127..=0` | **untrusted telemetry**; clamped to `-127` when out of range (`wire.rs:73-77`) |
| 7 | u8 | `flags` | bit 0 = DTX, rest reserved & passed through | client-authored |
| 8.. | var | Opus payload | fully opaque | never decoded |

`FrameHeader` (`wire.rs:36-48`) is `Copy`. `parse` (`wire.rs:65-88`) returns
`Option<(FrameHeader, &[u8])>`; `None` only on `len < 8`. A bad `level_dbov` never
drops the frame — the module's stated threat-model invariant (`wire.rs:14-20`) and
its pinning test (`wire.rs:129-147`).

##### 2.2 Relay → client binary frame

`[peer_index: u8][client frame verbatim]`. Built in `Room::broadcast_frame`
(`room.rs:398-402`) with `BytesMut::with_capacity(1 + frame.len())`. The relay
never re-encodes: `prefixed` is cloned per receiver (`room.rs:409`), so a frame in
an N-peer room allocates once and refcount-clones N−1 times.

`deliver_prefixed` (`room.rs:422-429`) takes an already-prefixed buffer and skips
by `peer_index` rather than peer UUID — the cross-pod path, where the author is
remote and has no local peer id.

##### 2.3 Mesh media datagram

`MeshDatagram` (`buzz-relay-mesh/src/wire.rs:111-122`): `{ fenced: FencedHeader,
seq: u64, payload: Vec<u8> }`. For huddles the payload is
`[peer_index][v2 header][Opus]` — **peer_index is always payload byte 0, both
directions** (`mesh.rs:30-35`). Built by `media_datagram` (`join.rs:1795-1809`) on
the ingress→owner leg and by `spawn_remote_peer_sink` (`mesh.rs:262-283`) on the
owner→ingress leg. Two independent `seq` counters exist per session leg
(`RemoteHuddleSession.seq` `join.rs:1508`, and the sink's local `seq`
`mesh.rs:264`), both `wrapping_add(1)`, both never read by any consumer.

An empty payload after a valid fence is dropped with a warning (`mesh.rs:229-232`).
`media_datagram` will happily emit a 1-byte payload (index only, no audio) —
pinned by `join.rs:2914-2916`.

---

#### 3. `HuddleControl` stream schema (postcard)

`HuddleControlMsg` (`join.rs:846-928`) — **7 variants**, postcard-encoded into
`MeshStreamFrame::Data.payload` (`join.rs:1010-1013`, `:1018-1020`).

| # | Variant | Direction | Payload |
|---|---|---|---|
| 1 | `RegisterPeer` | non-owner → owner | `community_id: Uuid`, `pubkey: String`, `protocol_version: u8` |
| 2 | `PeerRegistered` | owner → non-owner | `pubkey`, `peer_index: u8`, `roster: RosterSnapshot` |
| 3 | `RosterSnapshot` | owner → non-owner | `revision: u64`, `peers: Vec<RosterEntry>` |
| 4 | `RosterDelta` | owner → non-owner | `revision`, `joined: Option<RosterEntry>`, `left: Option<RosterEntry>` |
| 5 | `RosterResync` | non-owner → owner | unit |
| 6 | `RegisterRejected` | owner → non-owner | `pubkey`, `reason: RegisterRejection` |
| 7 | `UnregisterPeer` | non-owner → owner | `pubkey: String` |

`community_id` rides as a raw `Uuid`, not a `CommunityId`, **because `CommunityId`
is deliberately non-`Deserialize`** (`join.rs:851-865`); the owner re-mints it with
`CommunityId::from_uuid` at `join.rs:1211`.

`RegisterRejection` (`join.rs:960-975`) — 4 variants: `RoomFull`, `RoomEnded`,
`VersionMismatch{pinned,requested}`, `Fenced(FenceRejection)`.
`FenceRejection` (`join.rs:982-992`) — 4 variants mirroring the non-`Serialize`
`MeshError` fence arms 1:1: `StaleGeneration`, `NoActiveLease`, `OwnerMismatch`,
`FutureGeneration`; wire codes at `join.rs:1010-1017`.

`HUDDLE_CONTROL_PROFILE = Profile::HuddleControl` (`join.rs:1027`).
`HUDDLE_SESSION_ENDED = GoodbyeReason::SessionEnded` (`join.rs:1451`).

---

#### 4. Ownership / join value types

| Type | Line | Shape |
|---|---|---|
| `Ownership` | `join.rs:186-192` | `{ owner_runtime_id: RuntimeId, generation: u64 }` |
| `HuddleLease` | `join.rs:202` | newtype over `SessionLease`, `pub(crate)` field |
| `AcquireOutcome` | `join.rs:239-246` | `Acquired(HuddleLease)` \| `Held(Ownership)` |
| `HuddleRenewOutcome` | `join.rs:216-222` | `Renewed(HuddleLease)` \| `Lost` |
| `HuddleReleaseOutcome` | `join.rs:226-232` | `Released` \| `NotOwner` |
| `JoinOutcome` | `join.rs:249-268` | `LocalOwner{generation}` \| `RemoteOwner{owner_runtime_id, generation}` |
| `ResolvedJoin` | `join.rs:305-311` | `{ outcome, acquired: Option<HuddleLease> }` — `Some` **only** on the CAS-win arm |
| `HuddleTeardownCause` | `join.rs:1483-1495` | `OwnerLost` \| `OwnerDraining` \| `SessionEnded` \| `StreamClosed` |
| `DialError` | `join.rs:1646-1653` | `Rejected(RegisterRejection)` \| `Mesh(MeshError)` |
| `RemoteHuddleSession` | `join.rs:1489-1510` | `peer_index`, `roster`, `fenced`, `owner`, `pubkey`, `transport`, `seq` |

##### 4.1 `HuddleOwnerRegistry` — per-pod owner state

`join.rs:600-604`: `entries: DashMap<Uuid /*session_id*/, HuddleOwnerEntry>` plus a
sticky `draining: AtomicBool`.
`HuddleOwnerEntry` (`join.rs:606-623`): `lost`, `draining`, `cancel`
(`CancellationToken` ×3) and `generation: u64` — the fence token for
`release`/`drain` (`join.rs:734-742`, `:750-761`).
`HuddleOwnerSignals` (`join.rs:627-633`): `{ lost, draining }`, returned atomically
by `attach_signals` so a CAS winner cannot miss a concurrent drain.

##### 4.2 `GenerationFloor` — local stale-frame guard

`mesh.rs:89-91`: `seen: DashMap<Uuid /*session_id*/, u64 /*highest generation*/>`.
Monotone-only: `check` accepts `>= floor`, advances on `>`, rejects `<`
(`mesh.rs:102-128`). `FenceVerdict` (`mesh.rs:132-146`): `Accept{advanced: bool}` \|
`RejectStale{known: u64}`. `forget` deletes the entry (`mesh.rs:131-133`).
**Unbounded map** — one entry per session ever seen, only removed by `forget`
(called from `handler.rs:755`, `:763`, `:909`), never by a TTL sweep.

---

#### 5. Session directory — Redis keys, leases, fences

`tunnel/directory.rs` is the sole Redis surface in this group.

##### 5.1 Key space

`SessionKeys::new` (`tunnel/directory.rs:453-461`):

| Key | Pattern | TTL | Purpose |
|---|---|---|---|
| lease | `buzz:{community_id}:tunnel:{session_id}:lease` | `PX ttl_ms`, default **30 s** (`directory.rs:17`) | current owner, expires |
| generation | `buzz:{community_id}:tunnel:{session_id}:generation` | **none — never expires** | monotone `INCR` counter |

Shape pinned by test `directory.rs:625-637`. For huddles, `session_id == channel_id`
(`join.rs:20`, `handler.rs:324`), so the huddle lease key is
`buzz:{community}:tunnel:{channel_uuid}:lease`.

##### 5.2 Lease value encoding

`{owner_hex}|{generation}|{profile_wire}` — a `|`-delimited string, written in Lua
(`directory.rs:31`) and parsed by `parse_lease` (`directory.rs:495-531`).
`profile_wire ∈ {reliable-stream, realtime-media, huddle-control}`
(`directory.rs:473-479` / `:486-493`). Validation: exactly 3 parts, `generation != 0`,
owner must be 64 hex chars → `[u8; 32]`. Rejections pinned at `directory.rs:639-648`.

##### 5.3 `SessionLease` and results

| Type | Line | Fields |
|---|---|---|
| `SessionLease` | `directory.rs:88-101` | `community_id`, `session_id`, `owner_runtime_id`, `generation: u64`, `profile` |
| `AcquireResult` | `directory.rs:105-111` | `Acquired(SessionLease)` \| `Exists(SessionLease)` |
| `RenewResult` | `directory.rs:115-124` | `Renewed(SessionLease)` \| `Lost{current: Option<SessionLease>, known_generation: Option<u64>}` |
| `ReleaseResult` | `directory.rs:128-137` | `Released(SessionLease)` \| `NotOwner{current, known_generation}` |
| `DirectoryError` | `directory.rs:141-176` | 6 variants: `Pool`, `Redis`, `MalformedLease`, `MalformedGeneration`, `UnexpectedScriptStatus`, `InvalidLeaseTtl` |

##### 5.4 Four Lua scripts (atomic units)

| Script | Lines | Returns `{status, value, known_generation}` |
|---|---|---|
| `ACQUIRE_SCRIPT` | `:20-34` | `exists` (no INCR) \| `acquired` (INCR then SET PX) |
| `RENEW_SCRIPT` | `:36-52` | `missing` \| `renewed` (PEXPIRE) \| `lost` |
| `RELEASE_SCRIPT` | `:54-70` | `missing` \| `released` (DEL) \| `lost` |
| `VALIDATE_SCRIPT` | `:72-79` | 2-tuple `{lease, known_generation}` — read-only |

The generation counter is INCR'd **only** inside `ACQUIRE_SCRIPT` when no live
lease exists (`:26-33`), which is what makes generation monotone across owner death
even though the lease key expires (pinned `directory.rs:712-777`).

##### 5.5 Fence-token semantics (`validate_fenced_header`, `directory.rs:348-439`)

`FencedHeader` (`buzz-relay-mesh/src/wire.rs:85-93`) = `{session_id, generation,
owner_runtime_id}`. Verdict order:

1. `known = max(lease.generation, counter)` (`:375-380`)
2. `known > 0 && frame.generation < known` → `StaleGeneration` (`:382-389`)
3. no live lease → `NoActiveLease` (`:391-406`)
4. `frame.generation != lease.generation` → `FutureGeneration` (`:408-422`)
5. `frame.owner != lease.owner` → `OwnerMismatch` (`:424-437`)

Each rejection increments `mesh_fence_rejections_total{reason}` (`:481-483`) —
the **only** metric in this entire 9,266-line group.

---

#### 6. Reliable-stream data structures

| Type | Line | Shape |
|---|---|---|
| `ReliableStreamRouter<T: ?Sized>` | `reliable.rs:36-41` | `directory`, `transport: Arc<T>`, `local_runtime_id` |
| `ReliableJoin` | `reliable.rs:207-218` | `Owned{lease}` \| `Forwarded{lease, stream}` |
| `ReliableInbound` | `reliable.rs:221-228` | `fenced`, `from`, `stream` |
| `ReliableMeshStream` | `reliable.rs:231-235` | `fenced`, `stream: MeshStream`, `community_id: Option<CommunityId>` (latched) |
| `ReliableWireFrame` (private) | `reliable.rs:412-422` | `Data{community_id, payload}` \| `Goodbye{community_id, reason}` |
| `ReliableFrame` (public) | `reliable.rs:521-527` | `Data(Vec<u8>)` \| `Goodbye(GoodbyeReason)` |
| `ReliableStreamError` | `reliable.rs:529-568` | 10 variants |

##### 6.1 Reliable inner frame — a hand-rolled binary format

`encode`/`decode` at `reliable.rs:434-492`:

```
byte 0      VERSION = 1                (reliable.rs:423)
byte 1      kind: DATA=1 | GOODBYE=2   (reliable.rs:424-425)
bytes 2..18 community_id UUID (16B, raw big-endian bytes)
bytes 18..  Data: opaque payload  |  Goodbye: 1 byte reason
```

`GoodbyeReason` ↔ byte: `SessionEnded=1`, `Draining=2`, `StaleGeneration=3`
(`reliable.rs:501-518`). `decode` rejects `len < 18`, wrong version, unknown kind,
and a `Goodbye` whose length ≠ 19. Note `bytes[2..18].try_into().expect(...)` at
`reliable.rs:471` is infallible given the length check but is still an `expect` in
a production path.

`MAX_RELIABLE_PAYLOAD_BYTES = 1 MiB` (`reliable.rs:31`), chunked by
`send_bytes` (`reliable.rs:280`); the mesh wire cap is 16 MiB
(`buzz-relay-mesh/src/wire.rs:46`), asserted at `reliable.rs:945`. There is **no**
minimum-size, rate, or total-bytes bound.

---

#### 7. Boot / dispatch structures

| Type | Line | Shape |
|---|---|---|
| `MeshHandle` | `mesh_boot.rs:135-169` | `directory`, `transport`, `membership`, `local_runtime_id`, `dispatcher`, `audio_fence: Arc<GenerationFloor>`, private `runtime: MeshRuntime`, `owners: Arc<HuddleOwnerRegistry>` |
| `MeshInboundDispatcher` | `mesh_boot.rs:57-60` | `slots: Arc<DispatcherSlots>` |
| `DispatcherSlots` | `mesh_boot.rs:62-67` | 3 × `OnceLock`: `huddle_control`, `reliable_stream`, `datagrams` |
| `SessionStreamHandler` | `mesh_boot.rs:39` | `Box<dyn Fn(RuntimeId, StreamHello, MeshStream) + Send + Sync>` |
| `DatagramHandler` | `mesh_boot.rs:42` | `Box<dyn Fn(RuntimeId, MeshDatagram) + Send + Sync>` |

`AppState.mesh: Arc<OnceLock<MeshHandle>>` (`state.rs:627`), published once at
`main.rs:458`; `AppState::mesh()` returns `Option<&MeshHandle>` (`state.rs:812-814`).

---

#### 8. Conformance abstract model

Re-exported wholesale from `buzz-conformance` at `conformance/mod.rs:38-44`.

| Type | Definition | Shape |
|---|---|---|
| `AbstractState` | `buzz-conformance/src/lib.rs:150-161` | `resolved_community: CommunityLabel`, `bound_host: HostLabel`, `actor: ActorLabel` |
| `TraceStep` | `buzz-conformance/src/lib.rs:290-299` | `schema_version: u32`, `action: TraceAction`, `state_after: AbstractState` |
| `TraceAction` | `buzz-conformance/src/lib.rs:179-260` | **9 variants** (below) |
| `Verdict` | re-exported `conformance/mod.rs:39` | `Allow` \| `Deny` |
| `SanitizedReason` | re-exported | `Invalid` \| `Restricted` \| `ServerError` (mapping `conformance/mod.rs:422-429`) |

`TraceAction` variants: `WriteInsert`, `WriteInsertGlobal`, `WriteDuplicate`,
`SanitizedError`, `AuthCheck`, `ReadMessageRows`, `ReadByIdRows`,
`ReadHostFeedRows`, `ImplBug`. **`ReadHostFeedRows` has no emitter anywhere in the
relay** — its only occurrences outside the schema are the conformance crate's own
proptest (`buzz-conformance/tests/proptest_checker.rs:164`, `:255`).

##### 8.1 Label projections

| Label | Source | Line | Fidelity |
|---|---|---|---|
| `resolved_community` | `TenantContext::community()` | `conformance/mod.rs:57` | server-resolved only |
| `bound_host` | `TenantContext::host()` | `conformance/mod.rs:58` | full host string, **not** hashed |
| `actor` | first 16 hex chars of the pubkey | `conformance/mod.rs:70-74` | **not** blake3 — the doc comment at `conformance/mod.rs:51-53` claims "lower 16 bytes of `blake3(pubkey_bytes)`"; the code is a plain hex prefix. Documented delta, rationale given at `:64-69` |
| `msg_id` | first 8 bytes of the event id, hex | `conformance/mod.rs:78-85` | truncation, not hashing |
| `channel` | direct `Uuid` wrap | `conformance/mod.rs:89-91` | not opaque |
| `claimed_community` | first `h` tag parsed as UUID | `conformance/mod.rs:101-117` | `None` when absent or unparseable; recorded **separately** from resolved so the M2 bite survives |

##### 8.2 Row-community projection

`RowCommunityProjection` (`conformance/mod.rs:216-231`): `Ok(Vec<CommunityLabel>)` \|
`MissingLookup{kind: &'static str, first_missing_channel: Uuid}`.
`project_row_community` (`:186-197`): channel-less row → resolved label;
channel-scoped row → lookup in `HashMap<Uuid, CommunityId>` or `None`. A `None`
becomes `MissingLookup` and is emitted as `ImplBug{kind:
"row_community_lookup_missing"}` (`:286-294`, `:321-329`) — never silently
substituted. Pinned by four tests (`:614-696`).

##### 8.3 `EmitGuard` / `CountingTracer`

`EmitGuard` (`conformance/mod.rs:344-353`): `inner: Arc<dyn Tracer>`, `state`,
`counter: Arc<AtomicUsize>`, `kind: &'static str`.
`CountingTracer` (`:357-360`) wraps the real tracer and `fetch_add(1, Relaxed)` per
record (`:368-372`).
`Drop` (`:403-414`): if `counter == 0`, records
`TraceAction::ImplBug{kind}` on the **inner** tracer.
`arm` returns `(Self, Arc<dyn Tracer>)` (`:383-400`) — callers must pass the
returned wrapper downstream, or the counter never moves.

---

#### 9. Type-name collisions inside one crate (verified)

| Name | Definition A | Definition B | Distinct? |
|---|---|---|---|
| `AdmissionError` | `audio/room.rs:83-95` — `Ended`/`Full`/`VersionMismatch{pinned,requested}` | `admission.rs:12` — `Exceeded`/`Unavailable` | **Yes, two unrelated types.** Both are live: the audio one at `handler.rs:513`/`:524`/`:535` and `join.rs:1441-1449`; the other in `crate::admission`. Neither imports the other; nothing renames on import |
| `Ownership` | `audio/join.rs:186-192` (used by `HuddleDirectory`) | `audio/mesh.rs:74-80` (used by the dead `HuddleOwnerDirectory`) | Yes — identical fields, two definitions in sibling modules |
| `NoopTracer` | `conformance/tracers.rs:20` (production binding, `state.rs:798`) | `buzz-conformance/src/lib.rs:323` (**zero users**) | Yes |
| `RosterSnapshot` / `RosterDelta` | `audio/room.rs:64`,`:73` | `audio/join.rs:938` / `HuddleControlMsg::RosterDelta` | Yes — §1.4 |
| "mesh" | `buzz-relay-mesh` (inter-relay QUIC) | `mesh-llm-sdk` dev-dep (`Cargo.toml:87-88`), consumed only by `examples/mesh_*.rs` | Two unrelated meanings in one crate |
