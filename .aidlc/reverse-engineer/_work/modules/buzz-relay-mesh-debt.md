## Module: buzz-relay-mesh (`crates/buzz-relay-mesh`)
### Aspect: Technical Debt

Prioritized. Severity reflects blast radius **if the mesh is enabled** — today
`BUZZ_MESH` defaults off (`config.rs:498-500`) and nothing in this repo turns it on,
so most items are latent. D-04 is the exception: it is live regardless.

Baseline counts (all verified):

| Metric | Value |
|---|---|
| LOC | 3,169 across 10 files |
| Unit tests | **32**, all passing, **0 `#[ignore]`d** |
| Tests executed in CI | **0** |
| `unsafe` | **0** |
| `TODO`/`FIXME`/`XXX`/`HACK`/`todo!`/`unimplemented!` | **0** |
| `unwrap()` outside `#[cfg(test)]` | **0** |
| `expect()` outside `#[cfg(test)]` | **22** (all lock-poison) |
| `tokio::spawn` in production paths | **9** (all in `runtime.rs`) |
| Wire payload types | **5** (2 envelopes, 14 variants total across 6 enums) |
| `#[non_exhaustive]` public enums | **0 of 7** |
| Public items with zero external callers | ~40% of the surface |
| Unused declared dependencies | **3** (`hmac`, `futures-util`, `proptest`) |

---

#### CRITICAL

##### D-01 — Gossip records are applied without any authentication · CRITICAL (security)

`apply_gossip_record` (`membership.rs:120-153`) performs a version comparison and
nothing else, while its sibling `apply_ready_records` (`membership.rs:85-118`) does a
relay-pubkey anchor match plus a schnorr verify. `MeshStreamFrame::Gossip`
(`wire.rs:144-146`) carries no signature and no `FencedHeader`, and
`control_stream_exchange` applies every record in a `Delta` verbatim
(`runtime.rs:534-538`).

Any single admitted peer can therefore (a) insert arbitrary `runtime_id`s that then
pass `has_peer` admission on every other pod (`membership.rs:187-192` →
`runtime.rs:305-307`), bypassing the attestation the design exists to enforce
(`wire.rs:56-60`); (b) redirect every pod's 5-second dial loop at arbitrary
`host:port` (`runtime.rs:328-348`); (c) permanently pin a legitimate peer's addresses
or `draining` flag with a high `version`, which the version-1 registry seed can never
correct (BR-MESH-24).

**Fix:** embed `{relay_pubkey, relay_sig}` in `GossipRecord` and run the same
anchor+verify in `apply_gossip_record`. The preimage
(`registry.rs:85-91`) already binds the right tuple; this is a wire change, so it
needs an ALPN bump (`wire.rs:34-36`).

---

#### HIGH

##### D-02 — Membership table has no eviction, no expiry, and no revocation · HIGH

Verified: no `remove`/`retain`/`clear` anywhere in `membership.rs`. Entries persist
for the process lifetime. Consequences compound:

- `has_peer` admission is **monotonically permissive** — a retired or compromised
  runtime key stays admissible until every peer pod restarts. Registry TTL (45 s,
  `registry.rs:187`) has no effect on the in-memory allowlist.
- `reconcile_once` (`runtime.rs:296-326`) dials every non-draining record forever, so
  a scaled-down fleet leaves permanent dial churn.
- Unbounded memory growth: each entry holds a `Vec<String>` of arbitrary length plus a
  100-sample `PhiAccrual`, and per D-01 entries are attacker-insertable.
- Suspicion (`phi >= 8.0`) filters `peers()` (`membership.rs:366-369`) but not
  `has_peer` — and `peers()` has **zero production readers** (D-06), so suspicion has
  no effect at all today.

**Fix:** evict on `phi` beyond a hard threshold or on registry-record absence for
N scans; re-check attestation freshness periodically.

##### D-03 — Zero CI coverage: 32 tests compile and never run · HIGH

Verified chain:

- `just clippy` (`ci.yml:94`) = `cargo clippy --workspace --all-targets -- -D warnings`
  (`justfile:106-107`) — compiles the crate and its tests, runs nothing.
- `just test-unit` (`ci.yml:116`) runs only `-p buzz-core -p buzz-auth --lib`,
  `-p buzz-db --lib`, `-p buzz-conformance`, `-p buzz-push-gateway`
  (`justfile:275-295`). `buzz-relay-mesh` is absent.
- The backend-integration job archives `-p buzz-relay -p buzz-test-client --lib`
  (`ci.yml:336-342`). Also absent.
- `scripts/run-tests.sh` (the non-nextest fallback) covers the same four crates
  (`run-tests.sh:82-102`).

So the tie-break invariant, the attestation/anchor rejection tests, the wire
roundtrips, the 64-byte datagram header budget, and the Opus loss/order gate are all
regression-unprotected. They pass locally in 0.25 s — this is a one-line fix in
`justfile:278`.

##### D-04 — `GET /_mesh` is unauthenticated on an all-interfaces listener · HIGH (live today)

Route registered unconditionally at `crates/buzz-relay/src/router.rs:230`; handler
`router.rs:396-406` serialises `MeshStatus` whole (`router.rs:381`). The health
router (`router.rs:225-232`) has no auth layer, and its listener is bound to
**hard-coded `0.0.0.0:health_port`** (`main.rs:1119-1121`) — `BUZZ_BIND_ADDR` cannot
restrict it.

Leaks `peers[].endpoint_addrs` (`status.rs:20`) — the exact IP:port of every mesh UDP
socket in the fleet — plus all runtime ids, replica count, drain state, per-peer phi,
and 7 traffic counters per peer pair (`status.rs:17-61`).

Unlike the rest of this list, the *route* exists whether or not the mesh is enabled
(it returns `{"enabled": false}`, `router.rs:404`), so the surface is live now and
becomes an information leak the moment the mesh is switched on.

**Fix:** move it behind operator NIP-98 auth (the relay already has
`RELAY_OPERATOR_PUBKEYS` machinery), or drop `endpoint_addrs` from the default
projection.

##### D-05 — `iroh` dependency is declared with a pre-release requirement string · HIGH (supply chain)

`Cargo.toml:68`: `iroh = { version = "1.0.0-rc.0", default-features = false,
features = ["tls-ring"] }`.

**Nuance the brief should record:** `Cargo.lock` resolves iroh to **`1.0.2` from
crates.io** (`Cargo.lock:3902-3905`), because `^1.0.0-rc.0` admits stable 1.0.x. So
the shipped artifact is *not* on a release candidate. The manifest string is
nevertheless wrong in two ways: it advertises an rc dependency that is not what
builds, and pre-release requirements have surprising semver semantics (they opt the
requirement into matching other `1.0.0-*` pre-releases). It should read `"1.0"`.

Residual risk that is real: iroh 1.0.x is very new, `buzz-relay-mesh` is its only
consumer in the workspace, and the consumed surface is broad —
`endpoint::presets::Minimal`, `EndpointAddr::from_parts`, `TransportAddr`,
`Connection::remote_id/alpn/max_datagram_size`, `ReadExactError::FinishedEarly`
(`endpoint.rs:3,33,65-68,105-108`; `peer.rs:50,58,69,172`). A 1.1 minor will be
picked up by `cargo update` with no signal.

**Fix:** change the requirement to `"1.0"` (or pin `"=1.0.2"`), and add a
`cargo-deny`/`cargo-audit` gate — there is none in `ci.yml` today.

##### D-06 — The membership seam is wired and never read · HIGH (dead architecture)

`MeshHandle.membership: Arc<dyn RelayMeshMembership>` is populated at
`mesh_boot.rs:501` and **never read** — verified, no `.membership` field access on a
`MeshHandle` exists in `crates/buzz-relay/src`. Therefore:

- `RelayMeshMembership::peers()` (`membership.rs:359-379`) — 0 production callers.
- `RelayMeshMembership::local_runtime_id()` (`membership.rs:381-383`) — 0 callers.
- `PeerInfo` (`lib.rs:129-139`) is read nowhere; its only appearance in the consumer
  is `#[allow(dead_code)] fn _peer_info_is_not_an_owner_signal(_peer: PeerInfo)`
  (`tunnel/reliable.rs:949`), a comment-in-code.

Consequence: the entire "who is alive / draining / dialable?" seam — one of the two
seams `lib.rs:11-19` calls the crate's contract — has no consumer. No routing
decision anywhere in the relay consults liveness, load, or drain state of a peer.
Peer selection for a session is entirely the Redis directory's business. Combined
with D-02 this means phi accrual, the suspect threshold, and `load` are all
computation with no effect.

---

#### MEDIUM

##### D-07 — `MeshRuntime::shutdown()` is never called; loops and the heartbeat leak to process exit · MEDIUM

`runtime.rs:75-76` documents "dropping all clones does NOT stop the loops — call
`shutdown()`." Zero production callers (`runtime.rs:155-164`) — only in-crate tests.
The relay's drain watcher (`mesh_boot.rs:481-497`) calls `begin_drain()` +
`owners.drain_all()` then `return`s.

So after SIGTERM the accept loop keeps admitting connections, the reconcile loop keeps
dialing every 5 s, the gossip loop keeps bumping the local record every 2 s, and
`spawn_registry_heartbeat`'s task (`runtime.rs:598-607`) keeps running — its
`JoinHandle` is discarded at the call site (`mesh_boot.rs:467`) so it cannot be
stopped at all. Registry cleanup depends on that task getting one post-flag tick
inside its 15 s interval before the process exits (30 s graceful window,
`main.rs:1108`) — usually true, not guaranteed. `ReadyHeartbeat::shutdown()`
(`registry.rs:306-312`), which exists for exactly this, has zero callers.

##### D-08 — No reconnect backoff, and dials are serialized inside the reconcile loop · MEDIUM

`dial_peer` (`runtime.rs:328-355`) has no jitter, no exponential delay, and no failure
counter. `reconcile_once` awaits each `dial_peer` sequentially (`runtime.rs:317-326`),
so N unreachable peers serialize N connect timeouts into one pass, delaying dials to
healthy peers. With D-02 (records never evicted) a scaled-down or crashed fleet
produces permanent 5-second dial storms plus an unrate-limited `warn!` per attempt
(`runtime.rs:342-346`). All loops also lack jitter, so a fleet started together
gossips and rescans in lockstep (`runtime.rs:285-293`, `:563-587`, `:600-606`).

##### D-09 — Unauthenticated inbound connections trigger a full Redis scan + N signature verifies · MEDIUM

`is_known_peer` (`runtime.rs:305-323`) rescans the whole registry for any unknown
runtime id: serial `SCAN`+`GET`-per-key (`registry.rs:217-231`, no `MGET`, no
pipelining) with one secp256k1 schnorr verify per record (`registry.rs:233-238`). The
attacker cost is one generated ed25519 keypair plus a QUIC handshake; the defender
cost is a Redis round-trip per key plus N verifies — on the relay's **main**
`deadpool_redis` pool (`mesh_boot.rs:447`). The accept loop is single-tasked
(`runtime.rs:259-283`), so a slow scan head-of-line-blocks legitimate inbound
connections.

**Fix:** single-flight the rescan with a minimum interval, and negatively cache
rejected ids.

##### D-10 — `gossip::GossipState` is a complete unused duplicate of `MeshMembership`'s logic · MEDIUM

`gossip.rs:81-166`: `new/records/get/update_local/digest/delta_for/apply_delta`.
`MeshMembership` reimplements all of it (`membership.rs:120-153` ≈ `apply_delta`,
`:166-178` ≈ `update_local`, `:208-223` ≈ `digest`, `:225-247` ≈ `delta_for`).
`GossipState` has **zero callers outside its own tests** (verified across
`crates/**`), yet it carries 3 of the 4 gossip tests (`gossip.rs:238-266`) — so those
tests validate code that never runs, while the code that *does* run
(`MeshMembership`) is covered by only one merge test (`membership.rs:474-479`).

The two implementations have already drifted: `MeshMembership::apply_gossip_record`
sets `connection_state = Connected` on any accepted record (`membership.rs:133`),
which `GossipState` does not model — see D-13.

**Fix:** delete `GossipState` and move its two merge tests onto `MeshMembership`, or
make `MeshMembership` wrap it.

##### D-11 — 16 MiB frame length is honoured before the body arrives · MEDIUM

`peer.rs:178-186`: read a 4-byte attacker-controlled length, bounds-check against
`MAX_STREAM_FRAME` (16 MiB, `wire.rs:46`), then `vec![0u8; len as usize]` and
`read_exact`. Four bytes buy a 16 MiB allocation, and the peer can then stall.
`stream_accept_loop` (`runtime.rs:386-472`) accepts streams in an unbounded loop with
no per-peer cap or semaphore; `datagram_recv_loop` (`runtime.rs:358-372`) dispatches
inline with no rate limit or quota. The relay's own sender self-limits to 1 MiB
(`tunnel/reliable.rs:26-31`), but that is a convention, not an enforced receive
bound. Gossip is the only bounded path (depth 64, drop-on-full, `runtime.rs:46,556`).

##### D-12 — `ReadyRecord` attestation covers only the two keys · MEDIUM

Preimage (`registry.rs:85-91`) = `ATTESTATION_CONTEXT` + `runtime_pubkey` +
`relay_pubkey`. **Not signed:** `endpoint_addrs`, `proto_version`, `capabilities`.
**No nonce, no issued-at, no expiry.** So anyone with Redis write access can rewrite
a validly-attested record's dial addresses (a redirection primitive on the registry
path, mirroring D-01 on the gossip path), and a captured record is replayable
forever. Redis ACLs are the only defence, and the crate asserts nothing about the
pool it is handed.

##### D-13 — `apply_gossip_record` lies about connection state · MEDIUM

`membership.rs:133` and `:146` set `connection_state = ConnectionState::Connected`
for **any** accepted record, including records about *third parties* learned second-hand
through a `Delta`. A pod therefore reports peers it has never connected to as
`"connected"` on `/_mesh` (`status.rs:23`), and a genuinely disconnected peer flips
back to `Connected` the moment any neighbour gossips a newer record about it —
overwriting the `Disconnected` that `remove_peer` just set (`runtime.rs:277-279`).
`/_mesh`'s connection_state field is not trustworthy.

##### D-14 — Documented observability does not exist · MEDIUM

| Claim | Reality |
|---|---|
| `mesh_fence_rejections_total{reason=stale_generation\|no_active_lease\|owner_mismatch\|future_generation}` (`lib.rs:102-109`, attributed to a chaos-gate ruling) | **no such metric anywhere in the repo**; the crate has no `metrics` dependency at all |
| `MeshCounters.stale_generation_rejections` (`status.rs:43`) | structurally always 0 — the only writer, `record_stale_generation_rejection` (`membership.rs:285-293`), has no caller but the test at `:486`. The relay's real fence rejects are raised in `tunnel/directory.rs:378,395,413,430` and never routed back |
| "reject unknown versions loudly (count it, log it)" (`wire.rs:39-41`) | logged (`runtime.rs:406,549`), never counted |

Net: a cross-pod fencing incident produces log lines and a `/_mesh` page reading all
zeros. No spans either — `tracing` is used only for events, no `#[instrument]`, so a
cross-pod session cannot be traced end to end.

##### D-15 — `MeshError::Transport(String)` swallows 8 distinct failure classes · MEDIUM

22 construction sites. iroh bind/connect/stream/datagram errors are flattened with
`err.to_string()` at 12 of them (`endpoint.rs:38,39,79,91,101`;
`peer.rs:82,95,113,123,151,155,163,174,187`), discarding iroh's structured error;
five attestation failure causes collapse into the same variant
(`registry.rs:56-83`), so "malformed hex" and "signature forgery" are
indistinguishable to a caller and cannot be alerted on separately. The unknown
*gossip* payload version is a `Transport` string (`gossip.rs:72`) while the
unknown *frame* version gets a typed variant (`wire.rs:185`) — asymmetric.

This directly contradicts the crate's own stated policy at `lib.rs:102-105`: "every
fence-visible reject is a typed variant, never a generic `Transport`."

---

#### LOW

##### D-16 — Six stale or incorrect doc comments · LOW

| Location | Claim | Reality |
|---|---|---|
| `lib.rs:55-56` | `BUZZ_MESH` is "`on` (default when replicas can exist)" | defaults **off** (`config.rs:498-500`), tested `mesh_boot.rs:544-555` |
| `lib.rs:186-188` | `MeshStream` halves are "placeholder … pre-transport" | they are the real iroh halves (`peer.rs:132-192`) |
| `lib.rs:11-19` | relay consumes the crate "exclusively through two seams" | also uses `InboundHandler`, `MeshStream`, both half traits, `MeshEndpoint`, `MeshPeer`, `GossipRecord`, `ReadyRecord`/`ReadyRegistry`, `MeshRuntime`, `spawn_registry_heartbeat` |
| `lib.rs:137-138` | `PeerInfo.load` is "gossiped by the peer (0.0..)" | structurally always `0.0` — no writer exists (`gossip.rs:35`, `runtime.rs:566` is a no-op closure) |
| `wire.rs:31` | non-`Hello` first frame ⇒ "the stream is reset" | dropped, not reset (`runtime.rs:404-406`); no `reset()` call exists in the crate |
| `gossip.rs:168` (type name `PhiAccrual`) | implies the Hayashibara phi-accrual detector | variance is never used; `mean_secs` (`:217-220`) is the only statistic — exponential approximation only (`:213-214`) |

##### D-17 — `StreamHello.sender` is never validated against the authenticated peer · LOW

`wire.rs:161` declares `sender: RuntimeId`; nothing compares it to `conn.remote_id()`.
The accept loop passes `hello` straight through (`runtime.rs:461-464`) and consumers
use the authenticated `from` argument, so this is latent — but the field is
attacker-controlled and its presence invites a future consumer to trust it. Also,
`wire.rs:29-31` says the `Hello` contract holds "in both directions," yet nothing
reads the peer's `Hello` on a stream we opened (`runtime.rs:190-191`).

##### D-18 — Three declared dependencies are unused · LOW

`hmac` (`Cargo.toml:20`) and `futures-util` (`:25`) have **zero `use` sites** in
`src/`; `proptest` (`:29`) is a dev-dependency with **zero property tests** despite
the workspace convention of using it (`Cargo.toml:117`). `tokio`'s `test-util`
feature is enabled (`:29`) but `tokio::time::pause` is never used. Removing the three
cuts compile time and audit surface; a proptest for the postcard wire roundtrip and
the scuttlebutt merge would be a natural fit.

##### D-19 — Two parallel counter models, one of them write-only · LOW

`peer::PeerCounters` (`peer.rs:10-15`, atomics at `:19-24`, incremented at
`:84,97,114,121`) and `status::MeshPeerCounters` (`status.rs:51-61`, incremented via
`membership.rs:249-283`) track overlapping quantities with different field names
(`streams_accepted` vs `streams_received`) and are never reconciled.
`MeshPeer::counters()` (`peer.rs:73-75`) has **zero callers** — the per-connection
atomics are pure overhead. Additionally `MeshCounters.peers` duplicates every
`MeshPeerStatus.counters` verbatim in the `/_mesh` JSON (`membership.rs:302`).

##### D-20 — 22 lock-poison `expect()`s escalate a single panic to fleet-wide mesh failure · LOW

`membership.rs:74,126,159,173,190,199,322,332,363`;
`runtime.rs:142,156,159,168,183,197,202,222,270,349,444,553,573`. AGENTS.md forbids
new `expect()` in production paths; these are the conventional lock-poison case, but
one panic inside any critical section makes every subsequent mesh call panic —
including `MeshHandle::status()` (`mesh_boot.rs:173`), i.e. the axum health handler.
`parking_lot` or `unwrap_or_else(|e| e.into_inner())` removes the escalation. No
panic path was found inside those sections (all arithmetic is `saturating_*`, no
indexing), so this is robustness, not an exploit.

##### D-21 — One detached task and one discarded `JoinHandle` break the abort discipline · LOW

The crate's pattern is "track every `JoinHandle`, abort on removal"
(`PeerEntry::abort`, `runtime.rs:56-62`). Two of nine spawns escape it:
`control_stream_exchange` on the accept side (`runtime.rs:449`) is spawned without
being pushed onto `PeerEntry.tasks`, so `remove_peer` aborts the recv loops and
leaves it running until its stream errors; `spawn_registry_heartbeat`'s handle
(`runtime.rs:598`) is dropped by the caller (`mesh_boot.rs:467`).

##### D-22 — Registry seeds are permanently version 1, so a stale gossiped address can never be corrected · LOW

`apply_ready_records` builds `GossipRecord::new(...)` (version 1, `gossip.rs:38`) and
routes it through the version-greater-wins merge (`membership.rs:110-116`). Once any
gossip has been received for a peer, its version exceeds 1 forever, so a *corrected*
Redis record — e.g. after a pod IP change — can never be applied. Combined with D-01
this makes a forged high-version record permanently authoritative.

##### D-23 — No capability or protocol-version negotiation · LOW

`proto_version` (`registry.rs:110`, `gossip.rs:19`) and `capabilities`
(`mesh_boot.rs:371-377`) are advertised and **never compared** — verified: every
occurrence is an assignment or a `/_mesh` echo. `open_session_stream` will open a
`HuddleControl` stream to a peer that never advertised it; the only backstop is the
receiver's dispatcher (`mesh_boot.rs:112-134`). Combined with zero
`#[non_exhaustive]` on any of the 7 public enums, forward compatibility within one
ALPN is nil; the design intent (bump ALPN with the wire version, `wire.rs:34-36`) is
sound but unenforced.

##### D-24 — `git`-sourced dev-dependencies on a third-party project · LOW→MEDIUM (build/supply chain)

`crates/buzz-relay/Cargo.toml:84-85`:

```toml
mesh-llm-sdk          = { git = "https://github.com/Mesh-LLM/mesh-llm.git", tag = "v0.73.1", … }
mesh-llm-host-runtime = { git = "https://github.com/Mesh-LLM/mesh-llm.git", tag = "v0.73.1", … }
```

Not a dependency of `buzz-relay-mesh`, but it lands squarely in this module's blast
radius: these dev-deps exist **only** to build the five `mesh_*` examples, all of
which are MeshLLM smoke tests unrelated to the inter-relay mesh (see
`-features.md` §5). Because CI runs `cargo clippy --workspace --all-targets`
(`ci.yml:94`, `justfile:106-107`), **every CI run must fetch and build a
git-sourced third-party LLM runtime** — which is why `ci.yml:1015-1052`,
`release.yml:141-181`, and `signed-macos-canary.yml:100-139` all carry llama
native-library build/cache plumbing.

Specific risks: tag references are mutable (a retagged `v0.73.1` changes the build,
though `Cargo.lock` pins the rev, resolved at `ci.yml:1019`); the upstream repo is
a hard availability dependency of CI; and the C++/Metal runtime it pulls in is the
source of the teardown crashes the examples work around
(`mesh_serve_client_smoke.rs:129-136`, `mesh_agent_e2e.rs:180-193`).

**Fix:** move the five examples behind a cargo feature so `--all-targets` skips them
by default, or relocate them to a separate crate excluded from the root workspace
(the pattern already used for `desktop/src-tauri`).

##### D-25 — Naming collision: two unrelated "mesh" subsystems in one crate · LOW

`crates/buzz-relay` contains both `mesh_boot.rs`/`api/mesh_demo.rs` (this
inter-relay QUIC mesh) and `examples/mesh_*.rs` (MeshLLM shared compute). The
desktop app adds a third spelling (`BUZZ_MESH_API_PORT`, `BUZZ_MESH_CONSOLE_PORT`,
`BUZZ_MESH_IROH_RELAYS`, `desktop/src-tauri/src/mesh_llm/mod.rs:37-47`), and
`justfile` targets `mesh-e2e`, `mesh-e2e-hardware`, `mesh-e2e-admission`,
`mesh-e2e-confidence` (`justfile:304-349`) all mean MeshLLM, not this crate. Both
subsystems even use iroh. This has already produced at least one mis-scoped
assumption in this very analysis brief. Rename one side (`relay-mesh` vs
`shared-compute`) before adding more surface.

---

#### Documentation debt

##### D-26 — The crate is absent from every repo-level document · MEDIUM

Verified by grep:

| Document | `buzz-relay-mesh` | `mesh` / `iroh` / `quic` |
|---|---|---|
| `ARCHITECTURE.md` (827 lines, §6 "Crate Reference" at `:330`) | absent | **zero hits anywhere in the file** |
| `AGENTS.md` repo-structure table | absent | only mesh-llm-adjacent text |
| `CONTRIBUTING.md` | absent | — |
| `README.md` | absent | — |
| `.env.example`, `deploy/`, helm charts | absent | no `BUZZ_MESH`, no `3478`, no `POD_IP` |
| `.github/CODEOWNERS` | absent | — |

So a 3,169-LOC crate that binds a new UDP port, writes a new Redis key namespace,
introduces a second serialization codec (postcard), a second identity system
(ed25519 node keys alongside secp256k1 Nostr keys), and an unauthenticated HTTP
status route is documented **only in its own source comments**. Those comments are
good — they record rationale and named review blockers (`wire.rs:52-60`,
`lib.rs:102-109`, `mesh_boot.rs:544-546`) — but they are the sole record, and six of
them are already stale (D-16).

The `lib.rs:30-36` per-file human-owner map ("Mari", "Max", "Perci", "Dawn") has no
CODEOWNERS backing, so it will silently rot.

---

#### Prioritized remediation order

1. **D-03** — add `-p buzz-relay-mesh` to `justfile:278`. One line; unlocks
   regression protection for everything below.
2. **D-04** — authenticate or narrow `GET /_mesh`. Live today, independent of
   `BUZZ_MESH`.
3. **D-01** — attest gossip records (needs an ALPN bump; do it before any other wire
   change).
4. **D-02 / D-09** — evict membership entries and single-flight the admission rescan.
5. **D-05 / D-24** — fix the `iroh` requirement string, add `cargo-deny`, feature-gate
   the MeshLLM examples out of `--all-targets`.
6. **D-06 / D-10 / D-14** — decide the membership seam's fate, delete `GossipState`,
   and either implement the documented metrics or delete the claims.
7. **D-07 / D-08 / D-11 / D-21** — lifecycle: call `shutdown()`, add backoff, cap
   per-peer streams, track the detached task.
8. **D-16 / D-26** — correct the six stale doc comments and add the crate to
   `ARCHITECTURE.md` §6 and `AGENTS.md`.
