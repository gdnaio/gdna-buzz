## Module: buzz-relay — huddle audio, tunnel & conformance seam (`crates/buzz-relay/src`)
### Aspect: Conventions

---

#### 1. Task / loop patterns

##### 1.1 Spawned tasks — exact inventory

| File | `tokio::spawn` sites | Which are production |
|---|---|---|
| `audio/join.rs` | 9 | **1** — the renewer at `join.rs:498` (reached via `attach_signals` at `:674`, `:688`, `:695`). The other 8 are in the test module |
| `audio/handler.rs` | 6 | **4** per connection: `send_loop` (`:663`), `heartbeat_loop` (`:667`), `audio_forward_loop` (`:670`), plus conditionally the owner-control reader (`:704`) and the owner-teardown watcher (`:733`). 1 test spawn |
| `audio/mesh.rs` | 1 | **1** — `spawn_remote_peer_sink` (`mesh.rs:272`), one per remote peer on the owner pod |
| `tunnel/reliable.rs` | 3 | **1** — the lease renewer (`reliable.rs:588`), which no production site invokes. 2 test spawns |
| `mesh_boot.rs` | 3 | **3** — the huddle-control accept task (`:271`), the reliable accept task (`:291`), and the drain watcher (`:485`) |
| `audio/room.rs`, `audio/wire.rs`, `tunnel/directory.rs`, `conformance/*` | 0 | — |

**Per audio connection: 4–6 tasks** (3 unconditional + reader + owner watcher).
At the 25-peer soft cap × N rooms this is the dominant task cost.

##### 1.2 Structured-concurrency discipline in `handler.rs`

Consistently applied: one root `CancellationToken` per connection
(`handler.rs:145`), child tokens for the loops that must die first
(`send_cancel` `:662`, `fwd_cancel` `:669`), and the parent cloned for loops that
must observe global cancellation (`hb_cancel` `:665`, `reader_cancel` `:688`,
`owner_cancel` `:732`). After `recv_loop` returns, `cancel.cancel()` then **every**
task is awaited (`handler.rs:787-800`) before cleanup runs — including the reader
task, so its clean-close reaches the owner before the peer is removed.

Deviation: the huddle-lease renewer is **detached**. `attach_signals` keeps only
`renewer.lost` (`join.rs:694`) and drops the `HuddleLeaseRenewer` struct, so its
`task: JoinHandle<()>` is never awaited. The struct exposes `.task` publicly
(`join.rs:467`) but no production code reads it. The same shape exists in the
reliable lane (`reliable.rs:200-205`).

Deviation: `spawn_remote_peer_sink` (`mesh.rs:272-283`) is fully detached — it
terminates only when its `mpsc::Receiver<Bytes>` closes, i.e. when the owner's
`Room` drops the peer. The teardown path relies on that implicit chain
(documented `join.rs:1341-1344`).

##### 1.3 `select!` conventions

`biased;` is used wherever cancellation or control must win over data:

| Site | Ordering |
|---|---|
| `handler.rs:187-193` | cancel → auth timeout |
| `handler.rs:952-956` | cancel → `ws_recv.next()` |
| `handler.rs:1071-1085` | cancel → control → data |
| `handler.rs:1099-1123` | cancel → peer control → audio |

Not biased: `heartbeat_loop` (`handler.rs:1130-1148`, tick vs. cancel — either order
is safe), the renewer loops (`join.rs:506-533`, `reliable.rs:592-621`), and
`serve_control_loop` (`join.rs:1156-1190`, where drain/lost arms are listed first
so the compiler's random polling still reaches them promptly).

A recurring idiom for an optional `select!` arm — a `pending()` future when the
token is absent, so the arm degenerates instead of the branch being duplicated:

```rust
let lost_fired = async {
    match &lost { Some(t) => t.cancelled().await, None => std::future::pending().await }
};
```

Used at `join.rs:1145-1155`, `join.rs:1167-1172` (roster), and
`handler.rs:735-748`. This is the module's signature pattern.

##### 1.4 Interval conventions

Both renewers set `MissedTickBehavior::Delay` (`join.rs:502`, `reliable.rs:591`) so
a stalled Redis call does not produce a burst of catch-up renewals.
`heartbeat_loop` (`handler.rs:1131`) does **not** set a missed-tick behaviour, so a
long stall can fire several ticks in a row and trip `MAX_MISSED_PONGS` spuriously.
The demo-echo drain poll uses a 100 ms interval (`mesh_boot.rs:315`); the mesh drain
watcher uses `sleep(500ms)` in a loop rather than an interval (`mesh_boot.rs:495`).

---

#### 2. Error handling

##### 2.1 No `unwrap` / `expect` / `panic` in most of the group

Counted over production code only (everything before the file's `#[cfg(test)]` mod):

| File | `unwrap()` | `expect(` | `panic!`/`unreachable!` | `unsafe` |
|---|---|---|---|---|
| `audio/join.rs` (1..1806) | 0 | 0 | 0 | 0 |
| `audio/room.rs` (1..556) | 0 | 0 | 0 | 0 |
| `audio/mesh.rs` (1..284) | 0 | 0 | 0 | 0 |
| `audio/wire.rs` (1..88) | 0 | 0 | 0 | 0 |
| `audio/mod.rs` | 0 | 0 | 0 | 0 |
| `tunnel/mod.rs` | 0 | 0 | 0 | 0 |
| `mesh_boot.rs` (1..521) | 0 | 0 | 0 | 0 |
| `conformance/tracers.rs` | 0 | 0 | 0 | 0 |
| **`audio/handler.rs` (1..1337)** | 0 | **6** | **1** `unreachable!` | 0 |
| **`tunnel/reliable.rs` (1..658)** | 0 | **1** | 0 | 0 |
| **`tunnel/directory.rs` (1..576)** | 0 | **2** | 0 | 0 |
| **`conformance/mod.rs` (1..430)** | 0 | **1** | 0 | 0 |
| **Total** | **0** | **10** | **1** | **0** |

The 10 `expect`s and the `unreachable!`:

| Site | Justification quality |
|---|---|
| `handler.rs:451` `pending_remote.expect("RemoteOwner matched above")` | Sound but fragile — the `if let` at `:448-450` already matched, then the value is re-taken |
| `handler.rs:457` `unreachable!("matched RemoteOwner above")` | Same pattern; a `let-else` re-destructure of a value proven to be `RemoteOwner` |
| `handler.rs:689` `remote_fence.expect("remote_fence set whenever remote_stream is")` | Invariant across three `Option`s set together at `:466-469`; not type-enforced |
| `handler.rs:692`, `:696`, `:701` `remote_session.expect(...)` ×3 | Same invariant, asserted three more times |
| `handler.rs:731` `state.mesh().expect("owner teardown watcher only exists when mesh owner state exists")` | Sound — guarded by `owner_lost.is_some() \|\| owner_draining.is_some()` at `:727` |
| `reliable.rs:471` `bytes[2..18].try_into().expect("16 byte community id slice")` | Infallible given the `len < 18` check at `:462`; could be `TryInto` on a fixed array |
| `directory.rs:261` `current.expect("renewed returns lease")` | Trusts the Lua contract — `renewed` always returns a non-empty value (`directory.rs:47`) |
| `directory.rs:291` `current.expect("released returns lease")` | Same for `released` (`directory.rs:65`) |
| `conformance/mod.rs:246` `row.expect("project_row_community returns None only for Some(ch)")` | Sound; a `match` would remove it |

Four of the six `handler.rs` `expect`s exist because `remote_session`,
`remote_stream`, and `remote_fence` are three parallel `Option`s that are always
`Some` together (`handler.rs:445-503`). A single `Option<struct{…}>` would make all
four unnecessary — this is the clearest local refactor available.

Repo rule from `AGENTS.md`: "Do not introduce new `unwrap()` or `expect()` in
production paths". The group is compliant on `unwrap()`, non-compliant on `expect()`
in four files.

##### 2.2 Lock-poisoning convention (inconsistent)

`audio/room.rs` uses three different strategies for the same mutex:

| Site | Strategy |
|---|---|
| `mark_ended` `:193` | `if let Ok(mut g)` → else return `false` |
| `clear_ended` `:202` | `if let Ok(mut g)` → else silently do nothing |
| `add_peer` `:229-231` | `.map_err(\|_\| AdmissionError::Ended)` — "poisoned ≈ shutting down" |
| `add_peer_at_index` `:282` | same |
| `remove_peer` `:338-340` | `let Ok(mut g) = … else { return }` — **the peer is never removed and its index leaks** |
| `remove_peer_and_check_ended` `:363` | `.ok()?` — returns `None`, caller treats it as "not ended" |
| `roster_snapshot` `:462` | `.unwrap_or_else(\|e\| e.into_inner())` — the only site that recovers through poisoning |

`conformance/tracers.rs:57-60` also recovers via `into_inner`, with an explicit
rationale (`:56`). Since nothing in the group can panic while holding the guard
(no user code runs inside the critical sections), poisoning is unreachable in
practice — but seven different handlings of one impossible case is noise, and two of
them (`remove_peer`, `clear_ended`) fail silently in a way that leaks state.

##### 2.3 Error type conventions

- Two `thiserror` enums: `DirectoryError` (6 variants, `directory.rs:141-176`) and
  `ReliableStreamError` (10 variants, `reliable.rs:529-568`). Both carry structured
  context (community, session, raw value) rather than strings.
- `ReliableStreamError` is annotated `#[allow(missing_docs)]` (`reliable.rs:531`) —
  the only doc-comment escape hatch in the group, against the `AGENTS.md` rule "New
  public API must have doc comments".
- Every `HuddleDirectory` method flattens `DirectoryError` into
  `MeshError::Transport(e.to_string())` (`join.rs:114`, `:139`, `:158`, `:172`),
  **losing the typed variant** — so a malformed lease and a Redis outage are
  indistinguishable to the huddle join path. Only `validate` preserves typed fence
  errors (`join.rs:180`), which is exactly what `FenceRejection::from_mesh_error`
  needs (`join.rs:996-1005`).
- `ensure_membership` returns `Result<Uuid, String>` (`handler.rs:1153-1158`) —
  stringly-typed, so the caller cannot distinguish "archived", "not linked",
  "not a member", and "db error"; all four collapse into the same WS
  `{"type":"error","message":"not a member"}` (`handler.rs:274-285`).
- `DialError` (`join.rs:1646-1653`) correctly splits a clean owner rejection from a
  transport failure, and the handler maps them to different WS errors
  (`handler.rs:474-503`).

##### 2.4 Fail-closed convention

Applied consistently at tenant and ownership boundaries:

| Decision | Fail-closed choice |
|---|---|
| Unmapped host | 404 with a generic message, never a default tenant (`handler.rs:69-88`) |
| Pre-join `get_channel` DB error | silent teardown, not admission (`handler.rs:404-410`) |
| Owner-ready loop exhaustion | transient error, never an ownerless owner peer (`join.rs:443-447`) |
| Ambiguous channel UUID on the media path | drop, never cross-tenant delivery (`room.rs:526-541`) |
| Missing row-community lookup | `ImplBug`, never substitute the resolved label (`conformance/mod.rs:245-254`) |
| Mesh bind / registry publish failure with mesh on | fatal boot (`mesh_boot.rs:423-463`) |

##### 2.5 Logging conventions

Structured `tracing` with `%`/`?` sigils and `channel_id`/`peer_id`/`session_id`
fields throughout. Production-code counts (excluding test modules):

| File | error | warn | info | debug | trace |
|---|---|---|---|---|---|
| `audio/handler.rs` | 1 | 22 | 7 | 6 | 1 |
| `audio/join.rs` | 0 | 4 | 0 | 4 | 0 |
| `audio/room.rs` | 0 | 1 | 0 | 0 | 0 |
| `audio/mesh.rs` | 0 | 1 | 0 | 3 | 0 |
| `tunnel/reliable.rs` | 0 | 4 | 0 | 0 | 0 |
| `tunnel/directory.rs` | 0 | 3 | 0 | 0 | 0 |
| `mesh_boot.rs` | 0 | 13 | 10 | 0 | 0 |
| **Total** | **1** | **48** | **17** | **13** | **1** |

`error!` is used exactly once, for a genuine invariant violation
(`handler.rs:590-600`), which matches the repo's severity discipline. The one
`trace!` is the per-frame v2 header dump (`handler.rs:996-1003`) — correctly at
`trace` so it is off by default.

Pubkeys are logged in full hex (`handler.rs:255`, `:283`, `:419`, …), consistent with
the rest of the relay and with the rationale in `conformance/mod.rs:66-68`.

---

#### 3. Backpressure conventions

The group has **two deliberately opposite** policies, and they are documented as
such:

| Lane | Policy | Sites |
|---|---|---|
| Realtime audio | **never queue, drop on full** | `try_send` at `room.rs:409`, `:427`, `handler.rs:1115` (`data_tx`), `handler.rs:1043` (Pong) |
| State-bearing control | **never drop; size generously and warn if it happens** | `mpsc::channel(32)` (`room.rs:45`), warning at `room.rs:441-446` |

Detail:

- Per-peer audio queue is 8 slots ≈ 160 ms (`room.rs:38-40`). A slow WS peer loses
  audio but never stalls the room.
- The WS-side data channel is 16 slots (`handler.rs:659`), the WS control channel 8
  (`handler.rs:660`).
- `audio_forward_loop` (`handler.rs:1093-1125`) bridges room→WS using `try_send` on
  **both** channels — so a full 8-slot WS control channel silently drops a
  `joined`/`left` message that the room-level 32-slot channel deliberately protected.
  The room's warning (`room.rs:441-446`) does not fire for this second drop point.
  This is the one inconsistency in an otherwise coherent policy.
- Roster deltas use `tokio::sync::broadcast` (cap 64, `room.rs:179`) with the
  `Lagged` → full-snapshot recovery pattern (`join.rs:1174-1182`), which is the
  correct shape for a lossy-but-recoverable ordered stream.
- Mesh media send is fire-and-forget with `debug!` on error
  (`join.rs:1762-1765`, `mesh.rs:277-282`) — no queue, no retry.
- Reliable-stream sends are `await`ed, so QUIC's own flow control provides
  backpressure (`reliable.rs:283-291`).
- The huddle-control accept task and reliable accept task are `tokio::spawn`ed per
  inbound stream (`mesh_boot.rs:270-274`, `:290-298`) with **no concurrency bound** —
  the dispatcher doc says handlers "must hand off promptly (spawn) rather than
  block" (`mesh_boot.rs:36-37`), and they do, but nothing caps how many.
- The datagram handler runs **inline on the accept task** (`mesh_boot.rs:236-245`),
  justified because `on_media_datagram` is synchronous and non-blocking. Verified:
  it is a `DashMap` lookup plus `try_send`s (`mesh.rs:204-250`).

---

#### 4. Test conventions

##### 4.1 Counts

| File | tests | `#[ignore]`d |
|---|---|---|
| `audio/join.rs` | 34 | 0 |
| `audio/room.rs` | 9 | 0 |
| `tunnel/directory.rs` | 7 | 0 |
| `audio/mesh.rs` | 6 | 0 |
| `tunnel/reliable.rs` | 6 | 0 |
| `mesh_boot.rs` | 5 | 0 |
| `conformance/mod.rs` | 9 | 0 |
| `audio/wire.rs` | 4 | 0 |
| `audio/handler.rs` | 2 | 0 |
| `audio/mod.rs`, `tunnel/mod.rs`, `conformance/tracers.rs` | 0 | 0 |
| **Total** | **82** | **0** |

All 82 are inline `#[cfg(test)] mod tests`. No `tests/` directory exists for this
group; no `#[ignore]` anywhere.

##### 4.2 Silent-skip on missing Redis — the dominant convention

`redis_directory_if_available()` pings Redis and returns `Option<SessionDirectory>`;
every integration test opens with `let Some(directory) = … else { return; }`:

- `tunnel/directory.rs:592-604`, used by 5 tests (`:650`, `:712`, `:779`, `:882`)
- `tunnel/reliable.rs:707-719`, used by 4 tests
- `api/mesh_demo.rs:169-179`, used by 2 tests

These tests **pass vacuously** when Redis is absent. `just test-unit` therefore
reports green while never executing the Lua scripts, generation monotonicity,
or the fence taxonomy. Only `just test` (Postgres + Redis) exercises them. An
`#[ignore]` + explicit opt-in, or a hard failure when `REDIS_URL` is set,
would make the gap visible.

##### 4.3 Test doubles

| Double | Where | Covers |
|---|---|---|
| `FakeDir` | `join.rs:1821-1900` | scripted `HuddleDirectory`: queued `owner_of`/`acquire`/`validate` results, a `VecDeque` of renew outcomes, and call counters. This is the reason 34 tests run without Redis |
| `ChanSend`/`ChanRecv` + `stream_pair()` | `join.rs:2088-2110` | an in-memory `MeshStream` pair over `mpsc::unbounded_channel`, built through the **public** `MeshStream::new` seam — drives `accept_inbound` end-to-end without iroh |
| `NullTransport` | `join.rs:2064-2079` | no-op `send_datagram`, erroring `open_session_stream` |
| `NoopTransport` / `DirectTransport` | `reliable.rs:734-757`, `:824-848`; `api/mesh_demo.rs:181-224` | `unreachable!()` on unexpected calls; `DirectTransport` wraps a real `MeshPeer` |
| `StubSend`/`StubRecv` | `mesh_boot.rs:552-568` | minimal stream for dispatcher routing tests |
| `VecTracer` | `conformance/mod.rs:441-450` | collects `TraceStep`s so `EmitGuard` Drop behaviour is observable |
| `install_for_test` | `join.rs:782-795` | `#[cfg(test)]`-gated registry entry with a caller-supplied `lost` token and **no renewer**, isolating fan-out from renewer timing |

`await_release_calls` (`join.rs:2440-2456`) polls a counter with a 2 s timeout
because the registry owns the renewer task and exposes no `JoinHandle` — a
sound workaround, and the comment says so.

##### 4.4 Naming and documentation convention

Test names are full sentences describing the invariant:
`admit_full_wins_over_version_mismatch`, `registry_release_is_generation_fenced`,
`owner_ready_waits_for_winner_install_then_reuses`,
`parse_clamps_out_of_range_level_keeps_frame`,
`wired_datagram_consumer_shares_the_handle_fence`. Most carry a doc comment
explaining *why* the invariant matters, several citing the reviewer who asked for
it (`room.rs:759-766` "Per Sami/Perci's review", `room.rs:663-665` "Per Max's
review checklist", `mesh_boot.rs:543-545` "Blocker fix (Wren review of 8b077fdb)").

`mesh_boot.rs:546-556` shows a good convention: rather than asserting a lie when the
environment is forced, the test skips —
`if std::env::var("BUZZ_MESH").is_ok() { return; }` with the comment
"externally forced — skip rather than assert a lie".

##### 4.5 Testing gaps (structural, not stylistic)

- `audio/handler.rs` has 1,337 production lines and **2 tests**, both covering
  peripheral concerns (semaphore budget `:1341-1358`, parser size limit
  `:1417-1427`). The 719-line `handle_active_audio_connection` — every WS error
  code, the whole join sequence, teardown ordering — is untested at unit level.
- No test asserts the emitted **JSON shape** of any WS frame. The `code`/`message`
  strings the desktop client branches on (`desktop/src-tauri/src/huddle/relay_api.rs:130-150`)
  are unpinned on the relay side.
- No test covers `emit_participant_event`'s four-step pipeline, including the
  duplicate-skip and insert-error-but-fan-out-anyway branches
  (`handler.rs:1285-1307`).
- `conformance/tracers.rs` has **zero tests** — `JsonlTracer`'s truncate-on-create,
  serialization, and poison recovery are unverified (and it has no callers either).
- No test exercises `wire_mesh_consumers`' huddle-control or reliable-stream arms;
  only the datagram arm is covered (`mesh_boot.rs:665-737`).

---

#### 5. Documentation conventions

Module-level docs are unusually rich and carry ASCII flow diagrams
(`audio/mod.rs:6-9`, `handler.rs:3-12`, `room.rs:3-6`) plus explicit
"why not the other way" reasoning: `mesh.rs:26-35` (the payload invariant),
`join.rs:22-38` (Redis is the arbiter, mesh is a hint), `room.rs:216-227`
(error precedence as an information-leak defence), `wire.rs:12-20` (threat-model
invariant). `mesh_boot.rs:404-410` even argues with itself in prose about whether a
misconfigured mesh should be fatal.

Two conventions worth naming:

1. **Invariants are pinned by a named test, and the doc comment cites it.**
   `room.rs:124-127` → "See `version_pin_persists_across_peer_churn` for the test
   that pins this behavior"; `mesh_boot.rs:667-673` → the load-bearing shared-fence
   test.
2. **Deliberate non-features are documented as such**, so a reader does not
   mistake absence for oversight: `join.rs:1281-1284` (why `UnregisterPeer` needs no
   fence), `conformance/mod.rs:128-146` (why `claimed_community` is `None` on the
   read path), `mesh_boot.rs:206-215` (why no renewer is wired).

Stale docs found: `conformance/mod.rs:46-48` says the `req.rs`/`event.rs` wire
points are "held back as additive patch" when `req.rs:144`/`:355`/`:671` have
landed; `conformance/mod.rs:51-53` claims a blake3 actor label that the code does
not compute; `room.rs:33` lists a `speakers` control message that is never emitted;
`mesh.rs:1-10` opens with "Today a huddle's audio only fans out within a single
pod … This module removes that wall", present tense on both sides of a change that
has already shipped.
