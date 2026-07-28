## Module: buzz-relay — huddle audio, tunnel & conformance seam (`crates/buzz-relay/src`)
### Aspect: Debt

---

#### 0. Baseline metrics

| Metric | Value | Source |
|---|---|---|
| Files | 12 | assignment |
| Total lines | 9,266 | `wc -l` |
| Production lines (before each file's `#[cfg(test)]`) | 5,565 | computed |
| Test lines | 3,701 (**40 %**) | computed |
| Tests | **82** | see §6 |
| `#[ignore]`d tests | **0** | grep |
| Tests that silently no-op without Redis | **11** | §6.2 |
| `unsafe` blocks | **0** | grep |
| `unwrap()` outside `#[cfg(test)]` | **0** | §5 |
| `expect(` outside `#[cfg(test)]` | **10** | §5 |
| `panic!`/`unreachable!` outside `#[cfg(test)]` | **1** | §5 |
| TODO/FIXME/XXX/HACK markers | **0** | grep |
| `tokio::spawn` sites | 22 total; **11 production** | §2.3 |
| `metrics::` calls | **1** (`mesh_fence_rejections_total`, `directory.rs:483`) | grep |
| Public items with zero production callers | **13** | §3 |
| Duplicated type names inside the crate | **5** | §4 |

---

#### 1. `audio/join.rs` and `audio/handler.rs` — complexity

##### 1.1 File sizes

| File | Total | Production | Test | Test share |
|---|---|---|---|---|
| `audio/join.rs` | 3,036 | 1,806 | 1,230 | 41 % |
| `audio/handler.rs` | 1,430 | 1,337 | 93 | 7 % |
| `tunnel/reliable.rs` | 950 | 658 | 292 | 31 % |
| `tunnel/directory.rs` | 922 | 576 | 346 | 38 % |
| `audio/room.rs` | 790 | 556 | 234 | 30 % |
| `mesh_boot.rs` | 750 | 521 | 229 | 31 % |
| `conformance/mod.rs` | 727 | 430 | 297 | 41 % |
| `audio/mesh.rs` | 393 | 284 | 109 | 28 % |
| `audio/wire.rs` | 168 | 88 | 80 | 48 % |
| `conformance/tracers.rs` | 73 | 73 | **0** | 0 % |
| `audio/mod.rs` | 19 | 19 | 0 | — |
| `tunnel/mod.rs` | 8 | 8 | 0 | — |

`join.rs` at 3,036 lines is the largest file, but **41 % of it is tests** — 1,806
production lines across 48 functions is dense, not bloated. The real complexity
problem is `handler.rs`, which is 93 % production code with 7 % tests.

##### 1.2 Longest functions (production code only)

| Function | Lines | Location | `if`/`match`/`while`/`for` | match arms |
|---|---|---|---|---|
| **`handle_active_audio_connection`** | **719** | `handler.rs:167-885` | **49** | **36** |
| `serve_control_loop` | 256 | `join.rs:1115-1370` | 25 | 30 |
| `boot_mesh` | 110 | `mesh_boot.rs:411-520` | 5 | 0 |
| `recv_loop` | 107 | `handler.rs:947-1053` | 12 | 12 |
| `emit_participant_event` | 100 | `handler.rs:1237-1336` | 6 | 9 |
| `validate_fenced_header` | 92 | `directory.rs:348-439` | 5 | 0 |
| `read_owner_control` | 86 | `join.rs:1527-1612` | 10 | 13 |
| `ensure_membership` | 83 | `handler.rs:1153-1235` | 8 | 0 |
| `spawn_lease_renewer_with_interval` | 78 | `reliable.rs:580-657` | 3 | 8 |
| `wire_mesh_consumers` | 76 | `mesh_boot.rs:224-299` | 4 | 2 |
| `spawn_huddle_renewer_with_interval` | 73 | `join.rs:490-562` | 3 | 8 |
| `attach_signals` | 60 | `join.rs:665-724` | 6 | 0 |
| `dial_remote_owner` | 59 | `join.rs:1666-1724` | 2 | 6 |
| `resolve_join` | 57 | `join.rs:323-379` | 3 | 4 |
| `add_peer_at_index` | 54 | `room.rs:281-334` | 7 | 0 |

`handle_active_audio_connection` is **2.8× the next-longest function** and carries
49 branch points in one linear scope. It holds 12 mutable locals threaded across the
whole body (`pending_remote`, `acquired_lease`, `remote_session`, `remote_stream`,
`remote_fence`, `owner_lost`, `owner_draining`, `owner_generation`, plus four task
handles), and interleaves at least 9 distinct responsibilities:

1. challenge/auth handshake (`:176-238`)
2. relay + channel membership (`:244-286`)
3. mesh ownership resolution / availability gate (`:300-378`)
4. room acquisition + archived re-check (`:379-413`)
5. protocol-version admission (`:415-441`)
6. cross-pod dial + registration (`:443-503`)
7. local admission + owner-lease install (`:505-604`)
8. task spawning + supervision (`:657-800`)
9. teardown: peer removal, `left`, three lifecycle-event emits, archive, lease release (`:803-884`)

Each of stages 3–7 has its own early-return path that must independently remember to
`send_clean_close` the mesh stream and `cleanup_if_empty` the room — **7
`cleanup_if_empty` sites (`:401`, `:408`, `:484`, `:501`, `:637`, `:849`, `:865`, of
which 5 are on early-return paths) and 3 `send_clean_close` sites (`:520`, `:530`,
`:544`) in one function**, with no RAII guard for either obligation. `:389-403` and
`:404-410` are two arms of one `match` that both call `cleanup_if_empty`; `:637` is a
third for the `joined`-send-failure path. A new early return that forgets either is a
silent room leak or a leaked remote peer on the owner pod.

`serve_control_loop` (256 lines, 25 branches) is the second offender: a `select!`
over four arms inside a `loop` whose `break` value is a `Result`, with a
`teardown_reason` latch mutated from three places (`:1160`, `:1164`, `:1220`) and a
`stream_community` latch read from four (`:1176`, `:1266`, `:1291`, `:1346`).

---

#### 2. Prioritized findings

##### P1 — Correctness / security risk

| # | Severity | Finding | Evidence |
|---|---|---|---|
| D-01 | **High** | **Media datagrams are not Redis-fenced and never check `owner_runtime_id`.** `on_media_datagram` gates on a purely local monotone counter, then delivers. A mesh peer can inject audio into any room on any peer pod, attribute it to any `peer_index`, or poison the floor with one high-generation frame and silence the legitimate owner | `mesh.rs:204-250`, `:102-128`; `wire.rs:90-92` calls the owner field "advisory" |
| D-02 | **High** | **The media datagram envelope carries no community**, so the host-derived tenant boundary does not hold on the media path. `get_unambiguous_by_channel` fails closed only when two *active* communities share the channel UUID — with one active community per UUID (the normal case) a peer addresses that community's room without naming it. Acknowledged in-code as a known limitation | `room.rs:519-541`, `mesh.rs:36-42`, `:221-227` |
| D-03 | **High** (when enabled) | **`POST /_mesh/demo/echo` is unauthenticated and takes `community_id` from the request body** — the only route in the relay that does not derive the tenant from the host. Grants arbitrary lease creation and unbounded generation-counter inflation in any community's Redis key space, including on a channel UUID that is a live huddle's `session_id` | `api/mesh_demo.rs:50-95`; `router.rs:123` |
| D-04 | **Medium** | **The huddle path ignores `lease.profile`.** `HuddleDirectory::owner_of` discards it (`join.rs:110-116`), so a `reliable-stream` lease is honoured as huddle ownership. `reliable.rs:99-105` does check — the asymmetry is what makes D-03 reach huddles | `join.rs:107-127` vs `reliable.rs:99-105` |
| D-05 | **Medium** | **Audio shares the global `conn_semaphore` and holds a permit for the whole 5 s pre-auth window.** An unauthenticated client can exhaust the relay's entire WebSocket budget through `/huddle/…/audio`. No per-IP rate limit, no separate audio budget | `handler.rs:90-99`, `:190`; `state.rs:727` |
| D-06 | **Medium** | **`GenerationFloor.seen` is an unbounded `DashMap` with no TTL sweep.** Entries are added by any inbound datagram and removed only by explicit `forget`. Reachable from within the mesh at will | `mesh.rs:90`, `:131-133` |
| D-07 | **Medium** | **No frame-rate or bitrate limit.** Only a 4096-byte per-frame cap. One authenticated peer produces 24× amplification in-pod and 24× on the network cross-pod | `handler.rs:961-964`; `room.rs:398-411`; `mesh.rs:262-283` |
| D-08 | **Medium** | **`remove_peer` silently does nothing on a poisoned lock** (`let Ok(mut g) = … else { return }`), leaking the peer and its index. Six other sites in the same file handle poisoning six different ways, including one that recovers correctly | `room.rs:337-340`; cf. `:193`, `:202`, `:229`, `:282`, `:363`, `:462` |
| D-09 | **Medium** | **Post-admission lease blind window.** Local WS peers have no per-frame fence (stated in-code), so a >30 s partition can leave two pods fanning out locally for up to ~10 s until the next renew tick | `handler.rs:568-575`; `join.rs:452`; `directory.rs:17` |
| D-10 | **Medium** | **`audio_forward_loop` re-introduces a control drop point** that `room.rs`'s 32-slot queue exists to avoid: it `try_send`s into the 8-slot WS control channel, and unlike `broadcast_control` it does not warn. A `joined`/`left` lost here silently desyncs the client's index→pubkey map | `handler.rs:1104-1108` vs `room.rs:441-446` |

##### P2 — Maintainability

| # | Severity | Finding | Evidence |
|---|---|---|---|
| D-11 | **Medium** | **`handle_active_audio_connection` is 719 lines / 49 branches** with 12 threaded mutable locals, 5 `cleanup_if_empty` sites and 3 `send_clean_close` sites and no RAII guard for either. See §1.2 | `handler.rs:167-885` |
| D-12 | **Medium** | **`handler.rs` has 1,337 production lines and 2 tests**, both peripheral (semaphore budget, parser size cap). The entire join sequence, all 13 WS error codes, teardown ordering, and the 4-step lifecycle-event pipeline are untested | `handler.rs:1341-1358`, `:1417-1427` |
| D-13 | **Medium** | **Two `AdmissionError` types in one crate.** `audio/room.rs:83-95` (`Ended`/`Full`/`VersionMismatch`) and `admission.rs:12` (`Exceeded`/`Unavailable`) are unrelated types with the same name, both live, neither renamed on import. A reader at `handler.rs:513` must resolve the path to know which one | `room.rs:83`; `admission.rs:12` |
| D-14 | **Medium** | **Three parallel `Option`s that are always `Some` together** (`remote_session`, `remote_stream`, `remote_fence`) force **4 of the 6 production `expect`s** in the group. One `Option<struct{…}>` removes all four | `handler.rs:445-503`, `:689-702` |
| D-15 | **Medium** | **Duplicated roster types with no conversion coverage.** `RosterSnapshot`/`RosterDelta` exist in both `room.rs` and `join.rs` with hand-written conversions at `join.rs:1414-1433`. Nothing catches field drift | `room.rs:64-81`; `join.rs:895-945` |
| D-16 | **Low-Medium** | **Duplicated renewer implementation.** `spawn_huddle_renewer_with_interval` (`join.rs:494-562`) and `spawn_lease_renewer_with_interval` (`reliable.rs:580-657`) are structurally identical (73 vs 78 lines, same select, same loss contract, same release-error nuance) over two different lease newtypes. The huddle version's doc even says "A mirror of `crate::tunnel::reliable::spawn_observable_renewer`" (`join.rs:476-481`) | both |
| D-17 | **Low-Medium** | **Two `Ownership` types** with identical fields in sibling modules — `join.rs:186-192` (live) and `mesh.rs:74-80` (attached to a dead trait) | both |
| D-18 | **Low-Medium** | **`ensure_membership` returns `Result<Uuid, String>`.** Four distinct denial reasons (archived, unlinked parent, non-member, DB error) collapse into one client-visible `not a member`, and the caller cannot distinguish them for logging or metrics | `handler.rs:1153-1158`, `:274-285` |
| D-19 | **Low** | **`ReliableStreamError` is `#[allow(missing_docs)]`** — the only doc escape hatch in the group, against the `AGENTS.md` rule "New public API must have doc comments" | `reliable.rs:531` |
| D-20 | **Low** | **Double `get_channel` per join.** The same row is fetched at `handler.rs:1164` (inside `ensure_membership`) and again at `:389`, uncached. Deliberate for the race, but two Postgres round trips on every join | `handler.rs:389`, `:1164` |
| D-21 | **Low** | **`heartbeat_loop` does not set `MissedTickBehavior`**, unlike both renewers. A runtime stall can fire several ticks in a row and trip `MAX_MISSED_PONGS` spuriously | `handler.rs:1131` vs `join.rs:502`, `reliable.rs:591` |
| D-22 | **Low** | **`peer_pubkeys()` is unsorted while `roster_snapshot()` sorts by index**, so a same-pod client's `joined.peers` array is in `DashMap` order and a cross-pod client's is sorted — an avoidable client-visible inconsistency | `room.rs:471` vs `:479-484` |
| D-23 | **Low** | **`V2_HEADER_LEN`/`FLAG_DTX` are duplicated** between `audio/wire.rs:29,33` and `desktop/src-tauri/src/huddle/wire.rs:48`. One protocol constant, two copies, no shared crate | both |
| D-24 | **Low** | **`mesh_stack_smoke.rs:31` requires manual sync** with `buzz_lib::mesh_llm::MESH_WORKER_STACK_SIZE` in the desktop crate — a cross-crate constant coupled by comment | `examples/mesh_stack_smoke.rs:31` |

##### P3 — Observability

| # | Severity | Finding | Evidence |
|---|---|---|---|
| D-25 | **Medium** | **Zero metrics on the entire audio path.** No gauge for active rooms or peers, no counter for dropped frames, join failures, admission rejections by reason, version mismatches, heartbeat deaths, or owner-loss events. The group's only metric is `mesh_fence_rejections_total` | grep: 1 `metrics::` call in 9,266 lines, `directory.rs:483` |
| D-26 | **Medium** | **Audio frame drops are completely invisible.** Three `try_send` sites drop silently with no counter and no log (`room.rs:409`, `:427`, `handler.rs:1115`). A room at capacity degrading to unusable audio produces no signal at all | those three lines |
| D-27 | **Low** | **`seq` is written and never read.** Two independent per-leg counters (`join.rs:1508`/`:1759`, `mesh.rs:264`/`:270`) exist for "loss/reorder observability" (`wire.rs:113-115`) and no consumer computes loss or reorder from either | both |
| D-28 | **Low** | **`GET /_mesh` is unauthenticated** on a router documented as having no auth, and returns peer runtime ids, addresses and drain state | `router.rs:222-224`, `:230`, `:399-406` |

##### P4 — Dead code (zero production callers)

Verified by grep across `crates/**` and `desktop/src-tauri/**`.

| # | Item | Line | Status |
|---|---|---|---|
| D-29 | `HuddleOwnerDirectory` trait + `mesh::Ownership` | `mesh.rs:67-80` | **Zero implementors, zero callers, zero tests.** Superseded by `join.rs`'s `HuddleDirectory`. `mesh.rs:14` still points readers at it |
| D-30 | `read_teardown_cause` | `join.rs:1623-1642` | Zero production callers; 4 test callers (`join.rs:2948`, `:2981`, `:3007`) plus 5 dedicated tests (`:2955-3016`). Superseded by `read_owner_control` (`join.rs:1527`), the only one `handler.rs:707` uses. ~20 production + ~65 test lines |
| D-31 | `Room::mark_ended` | `room.rs:192-199` | Zero production callers; one test (`room.rs:660`). Production ends rooms via `remove_peer_and_check_ended` |
| D-32 | `PeerCtrl::Close` | `room.rs:35` | **Zero producers.** Handled at `handler.rs:1112` but nothing ever sends it — the graceful per-peer shutdown path it implies is unwired |
| D-33 | `SessionDirectory::takeover` | `directory.rs:233-242` | Zero callers anywhere. A documented "distinct operation" that is a verbatim delegate to `acquire` |
| D-34 | `SessionDirectory::known_generation` | `directory.rs:324-339` | Zero production callers; 2 test callers |
| D-35 | `ReliableStreamRouter::spawn_renewer` / `spawn_observable_renewer` | `reliable.rs:179`, `:192` | Zero callers. `mesh_boot.rs:206-215` documents that renewal "lands with the first product session consumer" — which has not landed |
| D-36 | `ReliableMeshStream::with_community` | `reliable.rs:263-266` | Zero callers |
| D-37 | `HuddleOwnerRegistry::attach` | `join.rs:659-661` | Zero production callers (thin wrapper over `attach_signals`); 5 test callers |
| D-38 | `MeshAudioRouter::{new, fence, local_runtime_id}` | `mesh.rs:169`, `:196`, `:201` | Zero production callers. Production uses `with_fence` (`mesh_boot.rs:239`) |
| D-39 | `MeshHandle.membership` | `mesh_boot.rs:141`, populated `:501` | **Written, never read.** `/_mesh` reads the private `runtime` field instead (`mesh_boot.rs:172-174`) |
| D-40 | `conformance::step` | `conformance/mod.rs:121-123` | Zero callers |
| D-41 | **`JsonlTracer`** | `tracers.rs:30-72` | **Zero callers in the entire workspace.** The only impl that would make the conformance seam observable. `state.rs:795-797` promises "test helpers in `crates/buzz-test-client` once those land" — they have not. 43 production lines, 0 tests |
| D-42 | `buzz_conformance::NoopTracer` | `buzz-conformance/src/lib.rs:323` | Zero users — the relay defines its own (`tracers.rs:20`) |
| D-43 | `TraceAction::ReadHostFeedRows` | `buzz-conformance/src/lib.rs:250` | **No emitter in the relay.** Only referenced by the conformance crate's own proptest (`tests/proptest_checker.rs:164`, `:255`) and its transition function |
| D-44 | `HuddleLeaseRenewer.task` / `ReliableLeaseRenewer.task` | `join.rs:467`, `reliable.rs:202` | Public `JoinHandle` fields, never awaited in production — the renewer is detached at `join.rs:694` |
| D-45 | `SessionStreamHandler` inbound reliable consumer | `mesh_boot.rs:299-303` | Accepts the stream, logs "no session consumer wired — closing", closes. Fence validation runs, then the work is discarded |

**Whole-lane dead weight:** the reliable-stream tunnel (`tunnel/reliable.rs`, 950
lines) has **no product consumer at all**. Its only join-side caller is the
demo-gated `POST /_mesh/demo/echo`, and its owner side either echoes (demo) or
closes. `reliable.rs:1` describes it as "for berd ↔ goose-server sessions"; no such
wiring exists in the repo.

##### P5 — Documentation drift

| # | Doc claim | Line | Reality |
|---|---|---|---|
| D-46 | "Operators running multiple relay pods MUST set `BUZZ_HUDDLE_AUDIO_AVAILABLE=false`" | `config.rs:120-128` | **Stale and misleading.** With `BUZZ_MESH=on` the flag is never read — `handler.rs:306-378` only consults it on the mesh-`None` arm. An operator following this doc on a mesh deployment sets a flag that does nothing |
| D-47 | "`actor` is the lower 16 bytes of `blake3(pubkey_bytes)` as a hex string" | `conformance/mod.rs:51-53` | `actor_label` takes the first 16 hex chars of the pubkey; no hashing (`:70-74`). The rationale 10 lines later (`:63-69`) explains the real design, contradicting the doc above it |
| D-48 | "req.rs / event.rs: (held back as additive patch for Eva to apply onto Max's req.rs writes)" | `conformance/mod.rs:46-48` | The `req.rs` wire points **have** landed: `req.rs:144`, `:355`, `:671` |
| D-49 | "Zero-cost tracer used in production builds… the build can have the compiler eliminate them entirely behind a feature flag" | `tracers.rs:11-13`, `state.rs:615` | No such feature flag exists (`Cargo.toml` `[features]` = `dev` only). Every emit site constructs its `TraceAction` (allocating `String`s, cloning `AbstractState`) and discards it |
| D-50 | "JSON control message (joined/left/**speakers**)" | `room.rs:33` | No `speakers` message is ever produced anywhere in the relay |
| D-51 | "Today a huddle's audio only fans out within a single pod… This module removes that wall" | `mesh.rs:1-10` | Present tense on both sides of a change that has shipped; the "today" is the pre-mesh state |
| D-52 | "The `HuddleControl` stream path… is wired in a following change" | `mesh.rs:150-153` | Already wired (`mesh_boot.rs:255-275`) |
| D-53 | "the client frame's encrypted content is NIP-44 between client endpoints, so server-side plaintext of the media itself never exists" | `buzz-relay-mesh/src/wire.rs:118-121` | **No such encryption exists.** `desktop/src-tauri/src/huddle/relay_api.rs:303` builds a plain v2 frame; the relay and every mesh peer hold plaintext Opus |
| D-54 | "older versions stay supported indefinitely for staged rollouts" | `handler.rs:120-122` | True for v1 acceptance, but there is no way to *pin* a deployment below `CURRENT_PROTOCOL_VERSION` — no env var, no config field |
| D-55 | "48100 — A huddle (audio/**video** session) was started" | `buzz-core/src/kind.rs:453-454` | Nothing in the relay handles video |
| D-56 | ARCHITECTURE.md: "**Room state:** an admission guard synchronizes joins…; soft cap 25 peers (hard cap 255 via `u8` peer index)" | `ARCHITECTURE.md:566` | Accurate, except the hard cap is **255 peers using indices 0..=254** — `alloc()` refuses at `next_fresh == 255`, so index 255 is never issued (`room.rs:146-152`) |
| D-57 | ARCHITECTURE.md § Huddle Audio (`:560-568`) makes no mention of the mesh, cross-pod ownership, the Redis session directory, or the `roster` control message | `ARCHITECTURE.md:560-568` | Describes only the single-pod design. The 4,400-line cross-pod subsystem in `join.rs`+`mesh.rs`+`directory.rs` is undocumented at architecture level |
| D-58 | Zero env vars from this group appear in `.env.example` or `deploy/compose/.env.example` | verified by grep | `BUZZ_HUDDLE_AUDIO_AVAILABLE`, `BUZZ_MESH`, `BUZZ_MESH_BIND_ADDR`, `BUZZ_MESH_DEMO_ECHO`, `BUZZ_MESH_ADVERTISE_ADDR`, `POD_IP` are discoverable only from source |

---

#### 3. Dead-code line-count summary

| Category | Approx. production lines |
|---|---|
| `HuddleOwnerDirectory` + `mesh::Ownership` (D-29) | 20 |
| `read_teardown_cause` + its 5 tests (D-30) | 20 prod / 65 test |
| `JsonlTracer` (D-41) | 43 |
| `takeover`, `known_generation`, `with_community`, `spawn_renewer`, `spawn_observable_renewer`, `attach`, `mark_ended`, `step`, `MeshAudioRouter::{new,fence,local_runtime_id}` (D-31…D-40) | ~90 |
| Reliable-stream lane with no product consumer (D-45 + whole-lane note) | 658 |
| **Total unreachable-from-production** | **~830 production lines (15 % of the group)** |

---

#### 4. `expect` / `unreachable` inventory (production paths)

| Site | Rationale quality | Fix |
|---|---|---|
| `handler.rs:451` `pending_remote.expect("RemoteOwner matched above")` | Fragile — the `if let` at `:448-450` already matched, then the value is re-taken | restructure the match to bind the payload once |
| `handler.rs:457` `unreachable!("matched RemoteOwner above")` | Same pattern | as above |
| `handler.rs:689`, `:692`, `:696`, `:701` (×4) | Invariant across three parallel `Option`s, not type-enforced | D-14: collapse into one `Option<struct>` |
| `handler.rs:731` `state.mesh().expect(...)` | Sound — guarded at `:727` | acceptable |
| `reliable.rs:471` `bytes[2..18].try_into().expect(...)` | Infallible given the `len < 18` check at `:462` | use a fixed-size array pattern |
| `directory.rs:261`, `:291` (×2) | Trust the Lua contract (`:47`, `:65`) | restructure the script return so the type carries it |
| `conformance/mod.rs:246` `row.expect(...)` | Sound but avoidable | a `match` removes it |

Against the `AGENTS.md` rule "Do not introduce new `unwrap()` or `expect()` in
production paths": compliant on `unwrap()`, non-compliant on `expect()` in four
files. Five of the ten are removable by one refactor each.

---

#### 5. Recommended remediation order

1. **D-01 / D-02** — Redis-fence media datagrams (or, at minimum, check
   `fenced.owner_runtime_id` against the expected owner and cap floor advancement),
   and add a community label to the media envelope so `get_unambiguous_by_channel`
   can be replaced with the community-scoped `get`. These two are the load-bearing
   security gaps and they share a fix surface.
2. **D-03 / D-04** — Either delete `POST /_mesh/demo/echo` (its own doc says "the
   route stays demo-gated until it is deleted") or move it behind the admin host with
   auth and derive `community_id` from the `Host`. Independently, make
   `HuddleDirectory::owner_of` reject a lease whose `profile != HuddleControl`, as
   `reliable.rs:99-105` already does.
3. **D-25 / D-26** — Add metrics: active rooms/peers gauges, dropped-frame counter,
   admission-rejection counter by reason, owner-loss counter. Zero observability on
   a realtime path is the fastest-compounding operational debt here.
4. **D-11 / D-12** — Extract stages 1–7 of `handle_active_audio_connection` into
   named async functions returning a small state struct, wrap the
   `cleanup_if_empty`/`send_clean_close` obligations in a `Drop` guard, and add
   tests over the resulting seams. This unblocks D-14 and D-18.
5. **D-05 / D-06 / D-07** — Give the audio route its own connection budget or a
   per-IP limiter; add a TTL/LRU bound to `GenerationFloor.seen`; add a per-peer
   frame-rate cap.
6. **D-29 … D-45** — Delete the dead code. ~830 production lines, no behavioural
   risk, and it removes two of the five name collisions (`mesh::Ownership`, the
   unused `NoopTracer`).
7. **D-46 … D-58** — Correct the documentation drift. D-46 (the
   `BUZZ_HUDDLE_AUDIO_AVAILABLE` guidance) and D-53 (the false NIP-44 encryption
   claim) are the two that could cause an operator or auditor to make a wrong
   decision; fix those first.
8. **D-41 + configuration** — Either wire `JsonlTracer` behind an env var
   (`BUZZ_CONFORMANCE_TRACE=<path>`) so the conformance seam becomes operable, or
   delete it and the `EmitGuard` arming in `ingest_event` so the crate stops paying
   for a mechanism that is inert. The current middle state — armed in production,
   discarding every result — is the worst of both.
