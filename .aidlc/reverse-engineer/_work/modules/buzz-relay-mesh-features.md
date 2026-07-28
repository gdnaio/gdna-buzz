## Module: buzz-relay-mesh (`crates/buzz-relay-mesh`)
### Aspect: Features

Crate description: "Inter-relay QUIC mesh: transport, membership, and the fenced
wire contract" (`Cargo.toml:3`). 3,169 LOC across 10 files; 32 unit tests, all
passing, none `#[ignore]`d (verified by running `cargo test -p buzz-relay-mesh --lib`:
`32 passed; 0 failed; 0 ignored`).

---

#### 1. Opt-in status: **off by default, hard no-op when off**

`BUZZ_MESH` must be an explicit `on`/`true`/`1`; absent, `off`, or any typo means
disabled (`config.rs:498-500`). When disabled, `boot_mesh` returns `Ok(None)` before
touching anything (`mesh_boot.rs:417-419`) — no UDP bind, no Redis write, no spawned
task. `AppState.mesh` stays an empty `OnceLock` (`state.rs:627`), so
`AppState::mesh()` returns `None` (`state.rs:812`) and every consumer takes its
single-instance path.

Two tests pin this: `mesh_off_boots_nothing` points the Redis pool at an unroutable
`redis://127.0.0.1:1` so any accidental Redis touch would fail
(`mesh_boot.rs:526-541`), and `mesh_defaults_off_when_env_absent`
(`mesh_boot.rs:544-555`) asserts the fail-safe reading.

Nothing in `.env.example`, `deploy/`, or the helm chart sets `BUZZ_MESH` — verified
by grep. **No deployment in this repo turns the mesh on.**

---

#### 2. What ships and works today

| Capability | Where | Evidence |
|---|---|---|
| iroh/QUIC endpoint bind with boot-unique ed25519 identity | `endpoint.rs:19-44` | test `two_endpoints_connect_with_alpn_and_authenticated_identity` (`endpoint.rs:180-187`) |
| ALPN-pinned, mutually-authenticated peer connections, iroh relays disabled | `endpoint.rs:35`, `peer.rs:50-55` | same test |
| Length-delimited reliable bi-streams carrying `MeshStreamFrame` | `peer.rs:135-192` | `reliable_stream_roundtrip_carries_mesh_stream_frame` (`endpoint.rs:189-227`) |
| QUIC datagrams carrying `MeshDatagram`, size-checked pre-send | `peer.rs:105-129`, `lib.rs:213-226` | `datagram_roundtrip_carries_mesh_datagram` (`endpoint.rs:229-237`), `oversized_datagram_is_rejected_before_send` (`endpoint.rs:239-254`) |
| Opus-sized datagram loss/order gate (64 frames, loopback) | — | `opus_sized_datagrams_clear_empirical_local_loss_gate` (`endpoint.rs:256-291`) |
| Datagram header-overhead budget (≤64 B) | — | `datagram_header_overhead_within_budget` (`wire.rs:271-284`) |
| Redis ready registry: publish / clear / scan, TTL 3× refresh | `registry.rs:182-257` | `ready_key_is_stable_and_namespaced` (`registry.rs:317-322`), `expiry_is_three_refreshes` (`:324-327`), `ready_record_roundtrips_json` (`:336-346`) |
| Relay-key schnorr attestation of the boot-unique endpoint key | `registry.rs:29-96` | `ready_record_attestation_verifies_and_binds_runtime_pubkey` (`:348-356`), `attestation_rejects_signature_for_other_runtime` (`:358-364`) |
| Deployment-anchored seed admission (foreign relay identity rejected, unanchored = fail-closed) | `membership.rs:85-118` | 4 tests, `membership.rs:425-471` |
| Scuttlebutt digest/delta membership gossip on a per-connection control stream | `membership.rs:208-247`, `runtime.rs:501-551` | `digest_delta_only_sends_newer_records` (`gossip.rs:238-251`), `warm_pair_connects_and_gossips_membership` (`runtime.rs:721-...`) |
| Last-version-wins record merge | `membership.rs:120-153` | `stale_gossip_record_is_ignored` (`membership.rs:474-479`), `apply_delta_ignores_stale_versions` (`gossip.rs:253-266`) |
| Accept-loop admission gate with one registry rescan | `runtime.rs:262-323` | exercised indirectly by `connected_pair` (`runtime.rs:645-670`) |
| Warm full-mesh reconcile/dial loop | `runtime.rs:285-355` | `warm_pair_connects_and_gossips_membership` |
| Deterministic simultaneous-dial tie-break | `runtime.rs:204-213` | `tie_break_is_symmetric` (`runtime.rs:846-856`), `simultaneous_dial_converges_to_one_connection` (`runtime.rs:858-...`) |
| Inbound fan-out to a single `InboundHandler` slot | `runtime.rs:196-198`, `:358-372`, `:461-470` | `transport_datagram_reaches_inbound_handler` (`runtime.rs:745-...`), `transport_session_stream_reaches_inbound_handler` (`runtime.rs:775-...`) |
| Typed error for sending to an unconnected peer | `runtime.rs:167-175` | `send_to_unconnected_peer_is_typed_error` (`runtime.rs:823-843`) |
| Drain: gossip `draining=true`, stop being dialed | `membership.rs:385-388`, `runtime.rs:302` | `begin_drain_updates_local_record` (`membership.rs:482-489`) |
| Readiness-gated registry heartbeat with clear-on-not-ready | `registry.rs:295-312`, `runtime.rs:594-608` | `heartbeat_starts_unpublished` (`registry.rs:329-334`) |
| Per-peer counters surfaced at `/_mesh` | `membership.rs:249-283`, `status.rs` | `counters_are_reflected_in_status` (`membership.rs:481-...`) |
| Phi-accrual-style suspicion (exponential approximation) | `gossip.rs:168-220` | `phi_rises_as_heartbeats_age` (`gossip.rs:268-278`) |
| Wire-version and gossip-payload-version rejection | `wire.rs:182-188`, `gossip.rs:66-77` | `unknown_version_rejected` (`wire.rs:246-257`), `gossip_payload_roundtrips` (`gossip.rs:...`) |

---

#### 3. What is stubbed, inert, or dead

| Item | Status | Location |
|---|---|---|
| `gossip::GossipState` (+ 7 methods) | complete duplicate of `MeshMembership`'s scuttlebutt logic; **zero callers** outside its own tests | `gossip.rs:81-166` |
| `PeerInfo.load` | structurally always `0.0` — no writer anywhere | `lib.rs:137-138`, `gossip.rs:35`, `runtime.rs:566` |
| `MeshCounters.stale_generation_rejections` | always 0 in production; sole writer's only caller is a test | `membership.rs:285-293`, test `:486` |
| `mesh_fence_rejections_total{reason=…}` metric | documented, does not exist | `lib.rs:102-109` |
| `MeshError::Disabled`, `MeshError::PeerDraining` | dead variants — zero constructors repo-wide | `lib.rs:94`, `lib.rs:119` |
| `RuntimeAttestation::verify` | zero callers | `registry.rs:48-50` |
| `ReadyHeartbeat::shutdown` / `record` / `published` | zero production callers | `registry.rs:287-312` |
| `MeshRuntime::shutdown` | zero production callers — loops leak to process exit | `runtime.rs:155-164` |
| `MeshRuntime::connected_peers` | zero callers outside in-crate tests | `runtime.rs:138-146` |
| `MeshEndpoint::endpoint()` | zero callers anywhere | `endpoint.rs:51-53` |
| `MeshPeer::counters()` / `PeerCounters` | atomics incremented, never read | `peer.rs:10-15`, `:73-75` |
| `MeshMembership::with_phi_suspect_threshold` | zero callers — 8.0 is unoverridable | `membership.rs:66-69` |
| `RelayMeshMembership::peers()` / `local_runtime_id()` | zero production callers; `MeshHandle.membership` is written (`mesh_boot.rs:501`) and never read | `membership.rs:359-383` |
| `proto_version` / `capabilities` negotiation | advertised, never compared | `registry.rs:110`, `mesh_boot.rs:371-377` |
| Peer eviction | none — membership table only grows | `membership.rs` (no `remove`/`retain`) |
| Reconnect backoff | none | `runtime.rs:328-355` |
| `hmac`, `futures-util` deps | declared, **zero `use` sites** | `Cargo.toml:20,25` |
| `proptest` dev-dep | declared, zero property tests | `Cargo.toml:29` |
| Peer selection / gossip fan-out sampling | none — gossip is 1:N to all connected peers | `runtime.rs:571-585` |

Also inert at the consumer boundary: **no product session consumer is wired**.
`wire_mesh_consumers` accepts a reliable stream, logs
`"reliable stream accepted; no session consumer wired — closing"`
(`mesh_boot.rs:288-292`), unless `BUZZ_MESH_DEMO_ECHO` is on. And
`mesh_boot.rs:216-220` notes owner-side lease renewal "lands with the first product
session consumer" — i.e. not yet.

---

#### 4. Delivered end-to-end paths (via `buzz-relay`)

Three inbound profiles are wired (`mesh_boot.rs:229-299`):

1. **`RealtimeMedia` datagrams → huddle audio fan-in.** `MeshAudioRouter` over a
   shared `GenerationFloor` (`mesh_boot.rs:243-254`). This is the one path with a
   real product consumer (cross-pod huddle audio, `audio/mesh.rs`, `audio/join.rs`).
2. **`HuddleControl` streams → `HuddleControlAcceptor::accept_inbound`**
   (`mesh_boot.rs:256-278`) — owner-side peer register/unregister.
3. **`ReliableStream` streams → `ReliableStreamRouter::accept_inbound`**
   (`mesh_boot.rs:280-299`) — accepted and fence-validated, then closed (or echoed
   under the demo flag). Goose/berd tunnels are the intended consumer and are absent.

Operator surface: `GET /_mesh` on the health listener (`router.rs:230`, handler
`router.rs:396-406`) and the double-gated `POST /_mesh/demo/echo`
(`router.rs:123`, `api/mesh_demo.rs`).

---

#### 5. The five `mesh_*` examples — **they do not exercise this crate**

The brief describes `crates/buzz-relay/examples/mesh_*.rs` as exercising
`buzz-relay-mesh`. **That is not the case.** All five import `mesh_llm_sdk` /
`mesh_llm_host_runtime` — the external **MeshLLM** peer-to-peer LLM-serving project
(git dev-deps, `crates/buzz-relay/Cargo.toml:84-85`, tag `v0.73.1`). Verified: grep
for `buzz_relay_mesh` across `crates/buzz-relay/examples/` returns **zero matches**.

Two unrelated things named "mesh" live in one crate, both spelled `mesh_*`:

| Example | LOC | What it actually proves | Subject |
|---|---|---|---|
| `mesh_admission_smoke.rs` | 452 | MeshLLM **owner-allowlist** admission: 3 subprocesses (serve + trusted client + non-member); possession of an invite token admits nobody, only allowlisted `OwnerKeypair` ids route inference (`mesh_admission_smoke.rs:1-30`, verdict matching `:262-283`) | mesh-llm |
| `mesh_agent_e2e.rs` | 421 | share-compute → agent-env preset → real `buzz-agent` over ACP stdio → inference; 4 permutations incl. a context-fit rejection (P3, `:106-134`) and `buzz-dev-mcp` tool use (P4, `:136-166`) | mesh-llm |
| `mesh_serve_client_smoke.rs` | 171 | MeshLLM serve node + client node joined by invite token, one completion routed client→serve (`:1-24`) | mesh-llm |
| `mesh_stack_smoke.rs` | 121 | tokio worker **stack-size** repro/fix: 2 MiB dies on the stack guard, 8 MiB completes a model download (`:1-30`); mirrors `buzz_lib::mesh_llm::MESH_WORKER_STACK_SIZE` (`:35-37`) | mesh-llm |
| `mesh_serve_smoke.rs` | 96 | single-node MeshLLM serve-and-self-consume; documents that `console_ui(true)` is load-bearing for `serve::start` readiness polling at mesh rev `bd16da4` (`:22-31`) | mesh-llm |

They are all explicitly **hardware-gated and not CI** — stated in each header
(`mesh_admission_smoke.rs:26-27`, `mesh_agent_e2e.rs:22-24`,
`mesh_serve_client_smoke.rs:25-26`, `mesh_stack_smoke.rs:26-27`) and reachable only
via `just mesh-e2e-hardware` (`justfile:327-331`), `just mesh-e2e-admission`
(`justfile:335-339`), `just mesh-e2e-confidence` (`justfile:342-349`).

##### CI-gate check

**No CI workflow gates any of the five examples, and none gates this crate's
tests.** Verified:

- Every `mesh` hit in `.github/workflows/{ci,release,signed-macos-canary,windows-canary}.yml`
  is MeshLLM build plumbing (resolving the `mesh-llm-sdk` rev from `Cargo.lock`,
  caching llama native libs, `--features mesh-llm` desktop builds) — e.g.
  `ci.yml:1015-1052`, `release.yml:141-181`.
- CI's Rust lint step is `just clippy` (`ci.yml:94`) = `cargo clippy --workspace
  --all-targets -- -D warnings` (`justfile:106-107`). `--all-targets` **compiles**
  the five examples (and therefore forces the git dev-deps to build) but runs
  nothing.
- CI's unit step is `just test-unit` (`ci.yml:116`), which runs only
  `-p buzz-core -p buzz-auth --lib`, `-p buzz-db --lib`, `-p buzz-conformance`,
  `-p buzz-push-gateway` (`justfile:275-295`). **`buzz-relay-mesh` is not in that
  list.**
- The backend-integration job archives `-p buzz-relay -p buzz-test-client --lib`
  (`ci.yml:336-342`) — also not this crate.

**Net: all 32 `buzz-relay-mesh` unit tests compile in CI and never execute.** They
pass locally (verified), including the four that stand up real loopback iroh
endpoints and the two multi-runtime gossip/tie-break tests.

---

#### 6. Test inventory (32 total, 0 `#[ignore]`d)

| File | Tests | Kind |
|---|---|---|
| `endpoint.rs` | 5 | `#[tokio::test]`, real loopback iroh endpoints |
| `membership.rs` | 7 | pure `#[test]`, real `nostr::Keys::generate()` signing |
| `registry.rs` | 6 | pure `#[test]`; `heartbeat_starts_unpublished` builds a pool but never dials |
| `runtime.rs` | 6 | 5 `#[tokio::test]` (2-runtime loopback meshes) + 1 pure |
| `gossip.rs` | 4 | pure |
| `wire.rs` | 4 | pure roundtrip/negative/budget |
| `lib.rs`, `peer.rs`, `status.rs` | 0 | — |

Untested behaviour of note: no test covers `is_known_peer`'s registry-rescan branch
(`runtime.rs:309-320`), the `remove_peer` path (`runtime.rs:267-281`), `dial_peer`'s
bad-addr/bad-id branches (`runtime.rs:328-348`), `ReadyRegistry::publish_ready`/
`clear_ready`/`scan_ready` against a live Redis, or `spawn_registry_heartbeat`
(`runtime.rs:594-608`). All Redis paths are effectively untested.
