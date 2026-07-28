## Module: buzz-relay-mesh (`crates/buzz-relay-mesh`)
### Aspect: Security

**Default posture: everything below is OFF by default.** `BUZZ_MESH` must be an
explicit `on`/`true`/`1` (`config.rs:498-500`); absent/`off`/typo → `boot_mesh`
returns `Ok(None)` before binding a socket, writing Redis, or spawning a task
(`mesh_boot.rs:417-419`). No config in this repo enables it (grep of `.env.example`,
`deploy/`, helm: zero `BUZZ_MESH`). So the findings below are latent, not live —
except **F-01 (`GET /_mesh`)**, which is registered unconditionally.

---

#### 1. Trust model between relay pods

Three layers, in decreasing strength:

| Layer | Mechanism | Strength |
|---|---|---|
| **Transport identity** | iroh/QUIC mutual TLS; peer identity = ed25519 node key; `RuntimeId` derived from `conn.remote_id()` (`peer.rs:58`) | strong — a peer cannot claim a RuntimeId it lacks the key for |
| **Deployment authorization** | relay-key schnorr attestation on Redis ready records, anchored to *our* relay pubkey (`membership.rs:90-103`) | strong for the registry path |
| **Membership propagation** | scuttlebutt gossip records | **none — unauthenticated** (F-02) |

The intended chain (`wire.rs:52-60`): all pods share one secp256k1
`BUZZ_RELAY_PRIVATE_KEY`, which would collapse the ownership plane if used as the
mesh identity, so each pod generates a fresh ed25519 mesh keypair at boot and the
shared relay key *signs* a binding published to Redis. Peers accept mesh connections
only from endpoint ids present in attested records.

##### Does a peer prove identity? Yes — twice, unevenly.

1. **Cryptographically, at the transport.** QUIC/TLS with ALPN `buzz/mesh/1`
   (`endpoint.rs:35`, `:88`); ALPN re-checked defensively (`peer.rs:50-55`); the
   RuntimeId is *read out of the authenticated connection*, never taken from a frame
   (`peer.rs:58`). `StreamHello.sender` (`wire.rs:161`) is attacker-controlled and
   **is never compared against the authenticated `remote_id`** — verified: the accept
   loop passes `hello` straight to the handler (`runtime.rs:461-464`) without
   cross-checking `hello.sender == remote`. Downstream consumers use the
   `from: RuntimeId` argument (authenticated), so this is currently latent, but the
   contract at `wire.rs:160-163` implies `sender` is meaningful and nothing enforces
   it.
2. **Authorization, at admission.** `is_known_peer` (`runtime.rs:305-323`) requires
   presence in the membership table. Entries reach that table via two paths with
   *asymmetric* verification (F-02).

---

#### 2. Findings

##### F-01 — `GET /_mesh` is unauthenticated and leaks the mesh topology · **HIGH**

Registered on the **health** router (`crates/buzz-relay/src/router.rs:230`), handler
`router.rs:396-406`, reading `AppState::mesh()` (`state.rs:812`) and serialising
`MeshStatus` directly (`router.rs:381`).

The health router is served by a listener bound to a **hard-coded `0.0.0.0`** on
`config.health_port` (default 8080) — `main.rs:1119-1121` — which **ignores
`BUZZ_BIND_ADDR`**. So an operator who binds the app listener to a private
interface still exposes the health listener on all interfaces. The health router
(`router.rs:225-232`) has **no auth layer, no NIP-42, no NIP-98, no operator-pubkey
check** — just four `get` routes.

Disclosed by `MeshStatus` (`status.rs:7-61`):

| Field | Site | Value to an attacker |
|---|---|---|
| `peers[].endpoint_addrs` | `status.rs:20` | **the exact IP:port of every mesh UDP socket in the fleet** — the dial targets for F-03/F-06 |
| `local_runtime_id`, `peers[].runtime_id` | `status.rs:10`, `:19` | full 64-hex ed25519 node keys; the allowlist contents |
| `peer_count`, `draining` | `status.rs:12`, `:11` | replica count and live rollout state (useful for timing an attack at a drain window) |
| `phi`, `connection_state` | `status.rs:24`, `:23` | which pods are currently suspect/partitioned |
| `proto_version`, `record_version`, `last_heartbeat_millis` | `status.rs:21`, `:26`, `:27` | build/version fingerprinting and liveness timing |
| per-peer counters (7 fields) | `status.rs:51-61` | traffic volumetrics per pod pair |

`MeshStatus` derives only `Serialize` (`status.rs:7`), and nothing redacts. Note
`mesh_boot.rs` deliberately keeps `runtime: MeshRuntime` private and exposes only
`status()` (`mesh_boot.rs:169-174`) — the encapsulation is fine; the exposure is the
route's lack of auth.

**Mitigation:** move `/_mesh` off the health listener onto the operator-authenticated
surface (the relay already has `RELAY_OPERATOR_PUBKEYS` + NIP-98 machinery,
`config.rs:~530`), or gate it behind `config.mesh.enabled` **and** operator auth, and
drop `endpoint_addrs` from the default projection.

##### F-02 — Gossip records are unauthenticated, so any admitted peer can inject arbitrary membership · **HIGH**

The two entry paths into `MeshMembership.peers` verify differently:

| Path | Verification |
|---|---|
| `apply_ready_records` (`membership.rs:85-118`) | relay-pubkey anchor match (`:90-92`) **then** schnorr verify (`:103`) |
| `apply_gossip_record` (`membership.rs:120-153`) | **none** — version comparison only |

`MeshStreamFrame::Gossip { payload }` (`wire.rs:144-146`) carries no `FencedHeader`
and no signature. `control_stream_exchange` decodes a `Delta` and applies every
record verbatim (`runtime.rs:534-538`). A record's `runtime_id`, `endpoint_addrs`,
`version`, `draining`, `capabilities`, and `heartbeat_millis` are all
attacker-chosen.

Consequences for a single compromised (or malicious-insider) pod that has been
admitted once:

1. **Admission-list poisoning.** Injecting `GossipRecord::new(attacker_runtime_id,
   …)` makes `has_peer(attacker_runtime_id)` true on every other pod
   (`membership.rs:187-192`), so `is_known_peer` (`runtime.rs:305-307`) admits inbound
   connections from a runtime that **never had a relay-key attestation**. The whole
   point of the attestation (`wire.rs:56-60`, `membership.rs:36-40`) is bypassed
   transitively. Redis is the "only way *into* the mesh" (`registry.rs:3`) only if
   gossip is trusted, and it is not verified.
2. **Outbound dial redirection / SSRF-by-UDP.** `reconcile_once` dials every
   non-draining record (`runtime.rs:296-326`) and `dial_peer` parses
   `endpoint_addrs` and connects (`runtime.rs:328-348`). An attacker sets
   `endpoint_addrs` to arbitrary internal `host:port` values; every pod then sends
   QUIC Initial packets there every 5 s, forever (no eviction, no backoff). The
   handshake will fail against a non-iroh target, but the *packets* reach arbitrary
   internal endpoints, repeatedly, from every pod.
3. **Address hijack of a legitimate peer.** Because the merge is
   version-greater-wins with no signature, a forged record for an existing
   `runtime_id` with `version = u64::MAX - 1` permanently pins that peer's
   `endpoint_addrs` (BR-MESH-24: the version-1 registry seed can never correct it
   again) and can set `draining = true` to make the fleet stop dialing a healthy pod.
4. **Unbounded table growth.** No eviction path exists (verified: no
   `remove`/`retain`/`clear` in `membership.rs`), so injected records are permanent
   for the process lifetime and each carries a `Vec<String>` of arbitrary length.

**Mitigation:** carry the `{relay_pubkey, relay_sig}` attestation *inside*
`GossipRecord` and run the same anchor+verify check in `apply_gossip_record` that
`apply_ready_records` already runs. The attestation preimage
(`registry.rs:85-91`) already binds exactly the right tuple.

##### F-03 — Admission is monotonically permissive; there is no revocation · **MEDIUM-HIGH**

Once a `runtime_id` enters the table it stays forever: `has_peer`
(`membership.rs:187-192`) has no TTL, no attestation re-check, and no eviction.
Registry records expire after 45 s (`registry.rs:187`, BR-MESH-08) but that has **no
effect on the in-memory allowlist**. Concretely:

- A retired/compromised pod's runtime key remains admissible on every surviving pod
  until those pods restart.
- Suspicion (`phi >= 8.0`) does not revoke: `peers()` filters it
  (`membership.rs:366-369`) but `has_peer` does not, and `peers()` has **zero
  production readers** (`-api-surface.md` §6), so suspicion has no security effect at
  all today.
- Draining does not revoke either — it only stops *outbound* dials
  (`runtime.rs:302`).

There is no key-revocation mechanism anywhere in the crate (no CRL, no epoch, no
"attestation valid until"). The attestation preimage (`registry.rs:85-91`) has **no
expiry, no nonce, and no timestamp**, so a captured `ReadyRecord` is valid forever
(see F-05).

##### F-04 — Unauthenticated inbound connections trigger a full Redis scan + N signature verifies · **MEDIUM**

`is_known_peer` (`runtime.rs:305-323`): unknown RuntimeId → `registry.scan_ready()`.
That is a `SCAN`+`GET`-per-key loop (serial, no `MGET`, `registry.rs:217-231`) with
one **secp256k1 schnorr verification per record** (`registry.rs:233-238` →
`registry.rs:70-83`).

Cost to the attacker: generate a fresh ed25519 keypair (microseconds), complete a
QUIC/TLS handshake to `0.0.0.0:3478`. Cost to the pod: one full Redis registry scan
plus N schnorr verifies, then a rejection. Repeat at line rate. Additionally each
rejection emits an unrate-limited `warn!` (`runtime.rs:265-269`) → log-volume
amplification.

Aggravating factors: the accept loop is **single-tasked and serial**
(`runtime.rs:259-283`) — one slow `scan_ready` head-of-line-blocks all legitimate
inbound connections; and the scan shares the relay's main `deadpool_redis` pool
(`mesh_boot.rs:447`), so the amplification lands on the same Redis connections the
event pipeline uses.

**Mitigation:** rate-limit or coalesce the rescan (one in-flight scan, shared
result, with a minimum interval), and cache negative admission decisions.

##### F-05 — Registry records are replayable; the attestation has no freshness binding · **MEDIUM**

Preimage: `buzz-relay-mesh-ready-v1\nruntime_pubkey={hex}\nrelay_pubkey={hex}`
(`registry.rs:85-91`). It covers **only** the two keys — **not** `endpoint_addrs`,
`proto_version`, or `capabilities`, and carries no timestamp/nonce/expiry.

Therefore anyone who can write to Redis (or MITM a non-TLS Redis link) can:

- **Rewrite `endpoint_addrs` on a validly-attested record.** `verify_attestation`
  (`registry.rs:139-146`) checks `runtime_pubkey == runtime_id.to_hex()` and the
  signature over the two keys; the address list is unsigned. This is a full
  redirection primitive against the *registry* path, matching F-02's gossip path.
- **Replay an old record** for a runtime id that has since restarted (its key is
  gone, so no handshake succeeds — the impact is dial churn, not admission) or
  resurrect a drained pod's record.

The record→key binding (BR-MESH-09, `registry.rs:233`) prevents key substitution but
not field tampering. Redis access control is the only real defence, and the crate
receives an externally-configured pool with no TLS assertion of its own.

**Mitigation:** sign the whole record (canonical serialization or add the fields to
the textual preimage) and add an issued-at/expires-at that verifiers check against
the TTL.

##### F-06 — Frame-size and stream-count amplification from an admitted peer · **MEDIUM**

- `MAX_STREAM_FRAME = 16 MiB` (`wire.rs:46`). The receive path reads a 4-byte
  attacker-controlled length and immediately allocates `vec![0u8; len]`
  (`peer.rs:184`) **before** any body has arrived. A peer can force a 16 MiB
  allocation per frame with 4 bytes sent, then stall. The relay's own reliable-stream
  consumer self-limits to 1 MiB chunks (`tunnel/reliable.rs:26-31`) but that is a
  sender-side convention; the receiver honours the full 16 MiB.
- `stream_accept_loop` (`runtime.rs:386-472`) accepts bi-streams in an unbounded
  loop; each session stream is handed to a consumer that spawns a task
  (`mesh_boot.rs:267-277`, `:285-298`). No per-peer stream cap, no concurrency
  semaphore.
- `datagram_recv_loop` (`runtime.rs:358-372`) decodes and dispatches inline with **no
  rate limit and no per-peer quota**.
- Gossip is the one bounded channel (depth 64, drop-on-full, `runtime.rs:46`, `:556`).

Blast radius is limited to already-admitted peers — but per F-02 that set is
attacker-extensible.

##### F-07 — `POST /_mesh/demo/echo` bypasses the host-derived tenant boundary · **MEDIUM** (double-gated)

`crates/buzz-relay/src/api/mesh_demo.rs`, route `router.rs:123`. Requires **both**
`BUZZ_MESH=on` and `BUZZ_MESH_DEMO_ECHO=on`, else 404 (`mesh_demo.rs:10-12`,
`config.rs:516-518`). It takes `community_id: Uuid` **from the request body**
(`mesh_demo.rs:47-48`) — the only relay route that does not derive the tenant from
the HTTP `Host`. It also acquires a real fenced lease and deliberately spawns no
renewer, so the lease lives out its Redis TTL (~30 s) (`mesh_demo.rs:12-16`).

Correctly gated and honestly documented as "not a product flow … the route stays
demo-gated until it is deleted" (`mesh_demo.rs:21-22`). The residual risk is
operator error: `BUZZ_MESH_DEMO_ECHO=on` left set in an environment that also has
mesh on yields an unauthenticated tenant-selecting endpoint. (Verified the gate is
strict-opt-in with the same parser as `BUZZ_MESH`, `config.rs:516-518`.)

##### F-08 — Lock poisoning escalates to fleet-wide mesh failure, including `/_mesh` · **LOW-MEDIUM**

22 `.expect("… lock poisoned")` outside `#[cfg(test)]` (`membership.rs:74,126,159,
173,190,199,322,332,363`; `runtime.rs:142,156,159,168,183,197,202,222,270,349,444,
553,573`). A single panic inside any membership or peer critical section poisons the
`RwLock`, after which **every** mesh operation panics — including
`MeshHandle::status()` (`mesh_boot.rs:173` → `membership.rs:296-313`), which is
called from the axum health handler. An attacker-triggerable panic in a critical
section would therefore turn `/_mesh` into a panicking route on the health listener.
No panic path was found inside those sections (all arithmetic is `saturating_*`,
no indexing, no `unwrap` on user data), so this is a robustness rather than an
exploitable finding.

##### F-09 — Transport error detail is discarded · **LOW**

All 12 iroh error sites flatten to `MeshError::Transport(err.to_string())`
(`endpoint.rs:38,39,79,91,101`; `peer.rs:82,95,113,123,151,155,163,174,187`), and the
five attestation failure causes collapse into the same variant
(`registry.rs:56-83`). Security consequence: a verifier cannot programmatically
distinguish "malformed hex" from "signature forgery attempt," so the two cannot be
alerted on differently. Only the anchor-mismatch case gets a counter
(`foreign_relay_rejections`, `membership.rs:93-94`); signature failures are logged
and forgotten (`membership.rs:104-109`).

---

#### 3. Positive security properties (verified, worth preserving)

- **Off by default, hard no-op.** Two tests pin it: `mesh_off_boots_nothing` uses an
  unroutable Redis URL so any accidental touch fails (`mesh_boot.rs:526-541`), and
  `mesh_defaults_off_when_env_absent` (`mesh_boot.rs:544-555`).
- **Fail-closed when unanchored.** `expected_relay_pubkey: None` rejects every ready
  record (`membership.rs:93`, documented `:36-40`, test `:465-471`) — the unanchored
  state is not accept-any.
- **Authorization checked before signature validity**, deliberately, so "signed by
  some relay key" is never mistaken for authorization (`membership.rs:80-84`, `:90-92`;
  test `ready_records_from_foreign_relay_identity_are_rejected`, `:451-462`).
- **Boot-unique identity** means a restarted pod cannot inherit a dead pod's
  ownership claims (`endpoint.rs:19-21`, `wire.rs:47-50`).
- **The mesh identity is deliberately not the shared deployment key** — reviewer-blocked
  design decision recorded in-source (`wire.rs:52-60`).
- **Transport encryption is mandatory**: QUIC/TLS 1.3 via `tls-ring`
  (`Cargo.toml:68`), mutual node-key authentication, ALPN-pinned. No plaintext mode
  exists; `RelayMode::Disabled` (`endpoint.rs:36`) means no third-party relay ever
  sees the traffic.
- **Media plaintext never exists server-side.** The datagram payload's client frame is
  NIP-44 between client endpoints; only the routing `peer_index` byte is
  relay-visible (`wire.rs:117-121`).
- **No ownership grant anywhere in the crate.** No Redis `SET NX`, `INCR`, `EVAL`, or
  CAS — verified; the only commands are `SET…EX`/`DEL`/`SCAN`/`GET` on
  `mesh:ready:*` (`registry.rs:188-228`). Forged frames cannot cause takeover, only
  rejection, because the fence lives in Redis (`tunnel/directory.rs`).
- **Self-record immutability by peers.** `apply_gossip_record` and
  `apply_ready_records` both drop records about self (`membership.rs:121-123`, `:87-89`),
  so no peer can rewrite our advertised addresses or set our draining flag.
- **Key/record binding in Redis** blocks key substitution (`registry.rs:233`) and
  `runtime_pubkey`/`runtime_id` cross-check blocks record splicing
  (`registry.rs:140-145`).
- **Loud, non-truncating size rejection** for frames and datagrams
  (`peer.rs:142-147`, `:178-183`, `lib.rs:218-223`).
- **One malformed registry entry cannot deny bootstrap** (`registry.rs:233-247`).
- **Zero `unsafe`**, zero `unwrap()` in production paths (only lock-poison
  `expect`s).
- **No secrets logged.** The foreign-relay warn logs the public key and an `anchored`
  boolean only (`membership.rs:96-101`); the private mesh key never leaves
  `iroh::SecretKey`.

---

#### 4. Forged-input resistance summary

| Attack | Outcome | Why |
|---|---|---|
| Forge a `RuntimeId` in a `StreamHello` | ineffective for routing — consumers use the authenticated `from` | `peer.rs:58`, `runtime.rs:461`; but `hello.sender` is never validated (§1) |
| Forge a `FencedHeader` generation | rejected by the Redis lease at the consumer | `tunnel/directory.rs:378,395,413,430` |
| Forge a `ReadyRecord` without the relay key | rejected — anchor + schnorr | `membership.rs:90-103`, `registry.rs:233-238` |
| Tamper `endpoint_addrs` on a valid `ReadyRecord` | **succeeds** — field not covered by the signature | F-05 |
| Forge a `GossipRecord` (any field, any runtime id) | **succeeds** — no verification at all | F-02 |
| Connect with a random keypair | rejected at admission, but costs a full Redis scan + N verifies | F-04 |
| Replay an old `ReadyRecord` | accepted (no freshness) — impact is dial churn / stale allowlist | F-05, F-03 |
| Replay a captured mesh frame | prevented by QUIC/TLS (per-connection AEAD + packet numbers); `seq` is not a replay guard (`wire.rs:114-116`) | — |
| Poison Redis with millions of `mesh:ready:*` keys | amplifies every scan; requires Redis write access | F-04, F-05 |
| Read `GET /_mesh` | **succeeds unauthenticated** — full topology + dial targets | F-01 |
