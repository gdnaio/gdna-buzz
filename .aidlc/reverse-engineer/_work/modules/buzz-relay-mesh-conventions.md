## Module: buzz-relay-mesh (`crates/buzz-relay-mesh`)
### Aspect: Conventions

---

#### 1. Module layout and ownership annotation

Eight flat modules, no submodule nesting (`lib.rs:21-28`). The crate carries an
explicit **lane-ownership map in code** (`lib.rs:30-36`):

```
endpoint.rs, peer.rs                              — Mari (transport core)
registry.rs, gossip.rs, membership.rs, status.rs  — Max (membership + /_mesh)
```
with a note that the session directory and tunnel routing live relay-side (Perci)
and huddle fan-out in `buzz-relay`'s audio module (Dawn). This is an unusual
convention for this repo — no other crate embeds per-file human owners in source.
There is **no corresponding `.github/CODEOWNERS` entry** (verified: no mesh path in
CODEOWNERS), so the annotation is advisory only.

Each module opens with a `//!` header stating its contract and its *limits*, and the
limit is always the same one: "this cannot elect owners." See `membership.rs:1-6`,
`gossip.rs:1-5`, `registry.rs:1-7`, `runtime.rs:21-23`. That repetition is the
crate's dominant documentation convention.

#### 2. Error handling

- **One error enum, `thiserror`-derived.** `MeshError` (`lib.rs:65-127`), 16 variants,
  `#[derive(Debug, thiserror::Error)]`. No `anyhow` inside the crate — `anyhow`
  appears only at the consumer boundary (`mesh_boot.rs:412`).
- **Typed variants for every fence-visible reject.** Explicit policy at
  `lib.rs:102-109`: "every fence-visible reject is a typed variant, never a generic
  `Transport`, so live kill-9 / partition / replay evidence is unambiguous."
  Honoured for the four fence variants; *not* honoured for the rest — see below.
- **`MeshError::Transport(String)` is the catch-all and it is overused.** 22
  construction sites collapse ~8 distinct causes into one untyped string:
  iroh bind/connect/stream/datagram errors (`endpoint.rs:38,39,79,91,101`;
  `peer.rs:82,95,113,123,151,155,163,174,187`), attestation parse and verify
  failures (`registry.rs:56-83`), key/pubkey mismatch (`registry.rs:141`), JSON
  encode (`registry.rs:186`), Redis pool acquisition (`registry.rs:271`), missing
  datagram support (`peer.rs:109`), and unknown gossip payload version
  (`gossip.rs:72`). The last one is notable: the *outer* frame version gets a typed
  `UnknownWireVersion` (`wire.rs:185`) while the *inner* gossip version gets a
  string, so the two version channels are not observably symmetric.
- **`err.to_string()` discards iroh's structured errors** at all 12 transport sites,
  so callers cannot distinguish timeout from refused from TLS failure.
- **`#[from]` used exactly once**, for `redis::RedisError` (`lib.rs:126`); everything
  else is explicit `map_err`.
- **Errors are propagated, never swallowed silently** — with two deliberate
  exceptions, both commented: `try_send` on the gossip queue
  (`runtime.rs:556`, "dropping a frame under backpressure is strictly better than
  blocking a recv loop") and `encode_message(&delta)` failures being skipped
  (`runtime.rs:540`, `if let Ok(payload)`).

##### `unwrap` / `expect` policy

AGENTS.md forbids new `unwrap()`/`expect()` in production paths. **22 `expect()`
outside `#[cfg(test)]`, zero `unwrap()`:**

| File | Count | All of the form |
|---|---|---|
| `runtime.rs` | 13 (`:142,156,159,168,183,197,202,222,270,349,444,553,573`) | `.expect("peer lock poisoned")` / `"loop lock poisoned"` / `"handler lock poisoned"` |
| `membership.rs` | 9 (`:74,126,159,173,190,199,322,332,363`) | `.expect("membership lock poisoned")` / `"local record lock poisoned"` |

Every one is a `std::sync` lock-poison unwrap — the conventional accepted case (a
poisoned mutex means another thread already panicked while holding it). Still a
literal deviation from the stated rule, and it means a panic anywhere inside a
membership or peer critical section escalates to a panic in **every** subsequent
mesh operation, including the `/_mesh` handler. `parking_lot` or explicit
`unwrap_or_else(|e| e.into_inner())` recovery would remove the escalation.

`0` `unsafe` (verified: grep for `unsafe` in `src/` returns nothing) — compliant.
`0` `TODO`/`FIXME`/`XXX`/`HACK`/`todo!`/`unimplemented!` markers (verified). All
known gaps are expressed as prose in doc comments instead, which makes them
invisible to grep-based debt tooling.

#### 3. Concurrency and task patterns

**Nine `tokio::spawn` sites in production code**, all in `runtime.rs`:

| # | Task | Site | Lifetime |
|---|---|---|---|
| 1 | `accept_loop` | `:119` | tracked in `MeshRuntime.loops` |
| 2 | `reconcile_loop` | `:120` | tracked in `loops` |
| 3 | `gossip_tick_loop` | `:121` | tracked in `loops` |
| 4 | `datagram_recv_loop` (per peer) | `:233` | tracked in `PeerEntry.tasks` |
| 5 | `stream_accept_loop` (per peer) | `:234` | tracked in `PeerEntry.tasks` |
| 6 | `open_control_stream` (per dialed peer) | `:240` | tracked in `PeerEntry.tasks` |
| 7 | `control_stream_exchange` (accept side) | `:449` | **untracked — detached** |
| 8 | control-stream send pump | `:509` | held locally, aborted on recv-loop exit (`:549`) |
| 9 | registry heartbeat | `:599` | `JoinHandle` returned and **dropped** by the caller (`mesh_boot.rs:467`) |

Two more spawns exist in tests (`endpoint.rs:162`, `:198`).

Patterns and their gaps:

- **`JoinHandle` bookkeeping + explicit abort** is the intended discipline
  (`PeerEntry::abort`, `runtime.rs:56-62`; `MeshRuntime::shutdown`, `:155-164`).
  Broken in two places: task #7 is spawned without being pushed onto
  `PeerEntry.tasks`, so a peer removal aborts the recv loops but leaves the accept
  side's control exchange running until its stream errors; task #9's handle is
  discarded at the call site so the heartbeat can never be stopped.
- **`shutdown()` is never called in production** (`-api-surface.md` §7) — every loop
  above runs until process exit.
- **Infinite `loop { … sleep }` with no jitter and no backoff**: `reconcile_loop`
  (`:285-293`), `gossip_tick_loop` (`:563-587`), heartbeat (`:600-606`), and the
  relay-side drain watcher (`mesh_boot.rs:484-496`). All pods started together will
  gossip and rescan in lockstep.
- **Lock hygiene is careful about await points.** `send_datagram` explicitly
  `drop(peers)` before touching membership (`runtime.rs:172-173`);
  `open_session_stream` clones the `MeshPeer` out of a scoped read guard before
  awaiting (`:182-189`); `install_peer` drops the write guard before
  `mark_connection_state` (`:249-252`). No `std::sync` guard is ever held across an
  `.await` — verified by reading all guard scopes.
- **Two lock kinds by intent**: `RwLock` for read-mostly tables
  (`Inner.peers` `:69`, membership's two `:31-32`), `Mutex` for the write-once
  handler slot (`:70`) and the loop vector (`:79`). `Arc<AtomicU64>`/`AtomicBool` for
  counters and the draining flag (`membership.rs:33-35`), all `Ordering::Relaxed`
  (`membership.rs:181,286,309,310`; `peer.rs:29-34`) — appropriate for pure counters.
- **Bounded channels only.** `mpsc::channel(CONTROL_QUEUE_DEPTH = 64)`
  (`runtime.rs:46`, `:237`, `:442`) with `try_send` drop-on-full (`:556`). No
  unbounded channel anywhere.
- **Handles are cheap-clone `Arc` facades**: `MeshRuntime` (`:77-80`),
  `MeshMembership` (`:29-43`), `MeshPeer` (`peer.rs:38-43`), `ReadyRegistry`
  (`registry.rs:160-163`) all `#[derive(Clone)]` over shared inner state. Documented
  at `runtime.rs:75-76`.

#### 4. Wire-compatibility discipline

Codified in `wire.rs:1-32`:

- `wire.rs` is declared a **FROZEN surface**; "changes here require a post in the
  mesh thread **before** the edit" (`wire.rs:10-13`), with the stated failure mode
  being two lanes compiling against different frame layouts. Unenforced by tooling.
- **ALPN carries the version** — `buzz/mesh/1` (`wire.rs:37`) — so a version bump
  gets a new ALPN and "old and new pods never half-speak to each other during a
  rolling deploy" (`wire.rs:34-36`). This is the primary compatibility mechanism.
- **Version byte first, reject unknown loudly** (`wire.rs:38-41`, enforced
  `wire.rs:183-186`). "Receivers MUST reject unknown versions loudly (count it, log
  it) rather than guessing" — the *log* happens at the call site
  (`runtime.rs:406`, `:549`); the *count* does not exist.
- **Nested opacity for evolution**: gossip rides as `Vec<u8>` inside
  `MeshStreamFrame::Gossip` with its own in-struct version so the gossip lane can
  evolve without a wire bump (`wire.rs:139-141`, `gossip.rs:13`). Same trick for
  huddle control on the consumer side (`audio/join.rs:797-801`).
- **Documented invariants are stated as MUSTs in prose**: "first frame MUST be
  `Hello`" (`wire.rs:29-31`), "senders MUST check the encoded size … never truncate"
  (`wire.rs:21-22`), "receivers MUST reject stale generations at every hop"
  (`wire.rs:22-24`).
- **No `#[non_exhaustive]` on any public enum** (verified: 0 occurrences). The
  convention is "bump the ALPN," not "tolerate unknown variants" — internally
  consistent but leaves zero forward compatibility within a version.
- **Header-size budget pinned by test**, not by a const:
  `datagram_header_overhead_within_budget` asserts ≤64 B so it "can't silently grow
  past the budget" (`wire.rs:266-284`). Nice pattern; note it is one of the 32 tests
  CI never runs.

#### 5. Naming and API-shape conventions

- **`Mesh*` prefix** for crate-owned types (`MeshEndpoint`, `MeshPeer`, `MeshRuntime`,
  `MeshStream`, `MeshDatagram`, `MeshStreamFrame`, `MeshStatus`, `MeshError`,
  `MeshConfig`, `MeshMembership`). **`Relay*` prefix** for the two consumer-facing
  seam traits (`RelayMeshMembership`, `RelayPeerTransport`, `lib.rs:144`, `:158`) —
  the prefix marks "this is the relay's view," which is a genuinely useful signal.
- **Builder-by-`with_`** on `MeshMembership` (`with_expected_relay_pubkey`,
  `with_phi_suspect_threshold`, `membership.rs:61`, `:66`), consuming `mut self`.
- **`record_*` for counter mutation** (6 methods, `membership.rs:249-283`) all funnel
  through one private `update_peer_counters` (`:315-326`) using `saturating_add`.
- **`*_now` for "do the periodic thing immediately"** — `reconcile_now`
  (`runtime.rs:150`), used as the boot fast-path.
- **`*_with_*` for the test/explicit variant of a production default**:
  `bind_with_secret_key` (`endpoint.rs:26`), `start_with_intervals`
  (`runtime.rs:102`), each documented as "production should use the plain one"
  (`endpoint.rs:23-25`, `runtime.rs:84-87`).
- **Time as `u64` millis on the wire, `SystemTime`/`Duration` in memory**, with
  explicit converters `now_millis` (`gossip.rs:223-229`) and
  `system_time_from_millis` (`gossip.rs:231-233`). `now_millis` clamps to
  `u64::MAX` and uses `unwrap_or_default()` on a pre-epoch clock (`gossip.rs:225-228`)
  — no panic path.
- **Addresses as `String` at the model boundary** (`GossipRecord.endpoint_addrs`,
  `ReadyRecord.endpoint_addrs`) explicitly "so this layer does not depend on
  transport internals" (`registry.rs:105-106`), parsed lazily at dial time
  (`runtime.rs:328-336`). Trades type safety for layer independence, and moves
  parse failures from boot to runtime.
- **Boxed futures instead of `async_trait`** (`BoxFuture`, `lib.rs:141`), public
  precisely so out-of-crate implementors can name it (`lib.rs:138-140`).
- **Concrete type over trait where lanes must share an implementation**:
  `MeshStream` is a struct, not a trait, "so lanes share one framing
  implementation" (`lib.rs:183`).

#### 6. Logging conventions

28 `tracing` sites. Consistent shape: structured fields first, then a
`"mesh: <event>"` message.

- Prefix is `mesh:` in `runtime.rs` (`:246`, `:265`, `:272`, `:279`, `:290`, `:317`,
  `:331`, `:342`, `:360`, `:380`, `:404`, `:409`, `:548`, `:602`) and
  `"mesh membership …"` / `"mesh ready registry …"` in the membership/registry lanes
  (`membership.rs:96`, `:105`; `registry.rs:236`, `:241`, `:246`).
- `peer = %runtime_id` (full 64-hex via `Display`) is the standard correlation field.
- Level discipline: `info!` for lifecycle (peer connected/disconnected `:255`, `:280`;
  endpoint closed `:272`), `warn!` for rejected/malformed input and dial failures,
  `debug!` for expected loop termination (`:360`, `:380`, `:548`).
- Rejections log enough to diagnose without leaking secrets — e.g. the
  foreign-relay warn logs `record_relay_pubkey` and `anchored`
  (`membership.rs:96-101`), never a private key.
- Gap: no rate limiting on `warn!`. `"rejected inbound connection from unattested
  runtime id"` (`runtime.rs:265-269`) and the dial-failure warn
  (`runtime.rs:342-346`, every 5 s per dead peer) are both attacker- or
  drift-triggerable log floods.

#### 7. Test conventions

32 tests, `#[cfg(test)] mod tests` at the bottom of 6 of 9 files; **no `tests/`
directory**, no integration-test target. All 32 pass; **0 `#[ignore]`d** —
notable, since this repo uses `#[ignore]` heavily for infra-dependent tests
(`justfile:277-285` explains the buzz-db pattern). Here the Redis-dependent paths
simply have no tests at all rather than ignored ones.

- **Deterministic identities in tests** via `SecretKey::from_bytes(&[n; 32])`
  (`endpoint.rs:157-162`, `runtime.rs:627-631`) and `RuntimeId([byte; 32])` helpers
  named `rid(byte)` — the same helper name is repeated in `membership.rs:394`,
  `registry.rs:319`, `gossip.rs:240`, `runtime.rs:...`, and even in the consumer
  (`mesh_boot.rs:582`). Convention by copy-paste, not by a shared test-utils module
  (this crate exposes no `test-utils` feature, unlike `buzz-core`).
- **Real transport in unit tests.** Five `endpoint.rs` tests and five `runtime.rs`
  tests stand up genuine loopback iroh endpoints and connect them
  (`endpoint_pair`/`connected_pair`, `endpoint.rs:155-176`; `runtime`/`seed`/
  `connected_pair`, `runtime.rs:626-670`). No mocking of QUIC. Whole suite still runs
  in 0.25 s.
- **Poll-with-timeout instead of sleep-and-hope**: every async assertion is
  `timeout(Duration::from_secs(5), async { loop { … sleep(20ms) } })`
  (`runtime.rs:655-668`, `:724-740`, `:756-768`, `:788-800`). Consistent and correct.
- **Explicit teardown** — every multi-runtime test calls `a.shutdown(); b.shutdown();`
  (`runtime.rs:742-743` etc.), which is the only place `shutdown()` is exercised.
- **Negative tests are first-class**: unknown wire version (`wire.rs:246`),
  oversize datagram (`endpoint.rs:239`), tampered attestation (`registry.rs:348`,
  `:358`; `membership.rs:437`), foreign relay key (`membership.rs:451`), unanchored
  fail-closed (`membership.rs:465`), stale gossip (`membership.rs:474`,
  `gossip.rs:253`), unconnected peer (`runtime.rs:823`).
- **Tests as executable specs for physical budgets**: the Opus-sized loss/order gate
  (`endpoint.rs:256-291`) and the 64-byte header budget (`wire.rs:266-284`).
- `tokio = { features = ["test-util"] }` is declared (`Cargo.toml:29`) but **no test
  uses paused time** (`tokio::time::pause` appears nowhere) — the phi tests hand
  `SystemTime` values in directly instead (`gossip.rs:268-278`), which is why they are
  fast and deterministic.
- Test-only doc comments explain *why* a setup is shaped as it is, e.g.
  `runtime.rs:647-650` explaining that with no registry the acceptor's admission gate
  requires pre-seeded membership "(production gets this from the attested ready
  registry)".

#### 8. Documentation conventions

- Every public item has a doc comment (AGENTS.md rule) — spot-checked across all 9
  files; the only bare items are the `IrohSendHalf`/`IrohRecvHalf` private structs
  (`peer.rs:132-133`).
- Doc comments record **rationale and rejected alternatives**, often naming the
  reviewer: "Wren's contract-review blocker" (`wire.rs:57`), "Wren's chaos-gate
  ruling" (`lib.rs:103`), "Dawn huddle peer_index" (`endpoint.rs:259`). Valuable
  archaeology; also means the source is the only record — none of it is in
  `ARCHITECTURE.md`, which does not mention this crate at all (verified: zero
  `mesh`/`iroh`/`quic` hits in all 827 lines).
- **Six doc comments are now stale or wrong** — see `-business-rules.md` §K for the
  full list (notably `lib.rs:55-56` claiming `BUZZ_MESH` defaults on, `lib.rs:186-188`
  calling the real iroh stream halves "placeholder", and `lib.rs:102-109`
  specifying a metric that does not exist).

#### 9. Deviations from repo-wide conventions

| AGENTS.md / repo convention | This crate |
|---|---|
| No `unsafe` | ✅ zero |
| No new `unwrap()`/`expect()` in production paths | ⚠️ 22 `expect()` (all lock-poison) |
| New public API must have doc comments | ✅ |
| Prefer Nostr events over new HTTP endpoints | n/a (crate has no HTTP surface; the consumer adds `GET /_mesh` and `POST /_mesh/demo/echo`) |
| Event kinds in `buzz-core/src/kind.rs` | n/a — this crate has no `buzz-core` dep and defines its own postcard wire, entirely outside the Nostr event model |
| Channel scoping via `h` tags | n/a — no tenant identifier on the mesh wire at all (`-integrations.md` §4) |
| Crate listed in AGENTS.md repo structure | ❌ absent |
| Crate documented in ARCHITECTURE.md §6 Crate Reference | ❌ absent |
| Unit tests run by `just test-unit` | ❌ absent from the list (`justfile:275-295`) |
| `#[ignore]` for infra-dependent tests | ❌ Redis paths untested rather than ignored |
