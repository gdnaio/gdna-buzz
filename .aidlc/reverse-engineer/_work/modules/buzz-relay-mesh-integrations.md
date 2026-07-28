## Module: buzz-relay-mesh (`crates/buzz-relay-mesh`)
### Aspect: Integrations

Five external couplings: **iroh/QUIC**, **Redis**, **nostr/secp256k1**,
**postcard**, and **`buzz-relay`** (the only consumer). Notably **`buzz-core` is
NOT a dependency** — see §4.

Full dependency list (`Cargo.toml:11-26`): `tokio`, `serde`, `serde_json`,
`postcard`, `iroh`, `redis`, `deadpool-redis`, `thiserror`, `tracing`, `uuid`,
`hmac`, `sha2`, `hex`, `nostr`, `bytes`, `futures-util`.
Dev-deps (`:28-30`): `tokio` (test-util), `proptest`.

Verified usage census (`use`-site count per crate):

| Dep | `use` sites | Status |
|---|---|---|
| `tokio` | 36 | used |
| `tracing` | 28 | used |
| `iroh` | 12 | used |
| `nostr` | 11 | used |
| `uuid` | 7 | used |
| `postcard` | 6 | used |
| `redis` / `deadpool-redis` | 5 / 5 | used |
| `serde` / `serde_json` | 4 / 4 | used (`serde_json` only in `registry.rs`) |
| `sha2`, `hex`, `bytes`, `thiserror` | 1 each | used |
| **`hmac`** | **0** | **unused dependency** (`Cargo.toml:20`) |
| **`futures-util`** | **0** | **unused dependency** (`Cargo.toml:25`) |
| **`proptest`** | **0** | **unused dev-dependency** (`Cargo.toml:29`) |

---

#### 1. iroh / QUIC

##### Version — delta against the brief

The workspace **requirement** is `iroh = { version = "1.0.0-rc.0",
default-features = false, features = ["tls-ring"] }` (`Cargo.toml:68`, comment
`:67` "Inter-relay mesh transport (buzz-relay-mesh)"). The crate takes it via
`workspace = true` (`crates/buzz-relay-mesh/Cargo.toml:15`).

**`Cargo.lock` resolves iroh to `1.0.2` from crates.io**
(`Cargo.lock:3902-3905`, checksum `5fca9b4b462c…`), not to the rc. Because
`^1.0.0-rc.0` admits `1.0.2`, the pre-release string in the manifest is a
*floor*, not a pin: the built artifact uses a stable 1.0.x release. The manifest
string is nonetheless misleading and should be `"1.0"` — see `-debt.md` D-05.

`iroh` is the **only** crate in the workspace using it (grep: no other
`Cargo.toml` references it).

##### Surface consumed

| iroh item | Used at |
|---|---|
| `Endpoint`, `Endpoint::builder`, `.bind()` | `endpoint.rs:3`, `:33-41` |
| `iroh::endpoint::presets::Minimal` | `endpoint.rs:33` |
| `SecretKey::generate()` / `SecretKey::from_bytes` | `endpoint.rs:20`; tests `endpoint.rs:158` |
| `PublicKey::as_bytes` / `from_bytes` | `endpoint.rs:97`, `:101` |
| `RelayMode::Disabled` | `endpoint.rs:36` |
| `.alpns(vec![ALPN.to_vec()])` | `endpoint.rs:35` |
| `.bind_addr(SocketAddr)` | `endpoint.rs:37` |
| `EndpointAddr`, `EndpointAddr::from_parts`, `TransportAddr::Ip` | `endpoint.rs:3`, `:65-68`, `:105-108` |
| `endpoint.accept()` → `Incoming` → `Connection` | `endpoint.rs:74-81` |
| `endpoint.connect(addr, ALPN)` | `endpoint.rs:88` |
| `Connection::alpn / remote_id / max_datagram_size / open_bi / accept_bi / send_datagram / read_datagram` | `peer.rs:50`, `:58`, `:69`, `:80`, `:93`, `:112`, `:120` |
| `endpoint::SendStream::write_all / finish`, `RecvStream::read_exact` | `peer.rs:148-165`, `:171-188` |
| `endpoint::ReadExactError::FinishedEarly` | `peer.rs:172` |

##### Configuration choices with consequences

- **`RelayMode::Disabled`** (`endpoint.rs:36`): no iroh relay servers, no hole
  punching via relays, no DERP fallback. Peers must be **directly IP-reachable** —
  which is why the deployment story is pod-to-pod inside one cluster
  (`advertise_addrs` prefers `POD_IP`, `mesh_boot.rs:398-403`).
- **`presets::Minimal`** (`endpoint.rs:33`): the leanest iroh endpoint preset — no
  discovery services.
- **`tls-ring`** (`Cargo.toml:68`) with `default-features = false`: ring rather than
  aws-lc-rs; consistent with the rest of the workspace's `tls-ring` choices
  (`Cargo.toml:57` redis, `:79` otlp).
- **Identity = iroh node key.** `RuntimeId` *is* `PublicKey::as_bytes()`
  (`endpoint.rs:96-98`), so iroh's TLS peer authentication and the mesh's identity
  model are the same thing. `MeshPeer::from_connection` derives the remote's
  RuntimeId from `conn.remote_id()` (`peer.rs:58`) — an attacker cannot claim a
  RuntimeId they do not hold the key for.

##### Failure modes

| Failure | Behaviour | Site |
|---|---|---|
| bind fails (port in use, bad addr) | `MeshError::Transport` → **fatal relay boot** | `endpoint.rs:38-41`; `mesh_boot.rs:383-390`; `main.rs:442` `?` |
| inbound handshake fails | warn `"mesh: inbound connection failed"`, accept loop continues | `runtime.rs:277-279` |
| `endpoint.accept()` returns `None` (endpoint closed) | accept loop logs and **returns** — no restart, mesh is permanently deaf | `runtime.rs:271-274` |
| dial fails for an addr | warn, try next addr; all exhausted → mark Disconnected, retry in 5 s **with no backoff** | `runtime.rs:340-354` |
| ALPN mismatch on an established conn | `MeshError::Transport("unexpected mesh ALPN …")`, peer not installed | `peer.rs:50-55` |
| peer lacks datagram support | `Transport("peer does not support QUIC datagrams")` on every send | `peer.rs:109` |
| datagram or stream read error | `remove_peer` → tasks aborted, `ConnectionState::Disconnected`; membership entry **kept** | `runtime.rs:359-363`, `:379-383`, `:267-281` |
| oversize frame/datagram | typed `FrameTooLarge` / `DatagramTooLarge`, never truncated | `peer.rs:142-147`, `:178-183`, `lib.rs:218-223` |
| iroh error detail | **flattened to `String`** via `err.to_string()` at 12 sites — the structured iroh error type is discarded, so callers cannot match on cause | `endpoint.rs:38,39,79,91,101`; `peer.rs:82,95,113,123,151,155,163,174,187` |

`iroh` being a pre-1.0-string dependency with a 1.0.x lock resolution means an
`iroh` 1.1 release will be picked up silently by `cargo update`; the API surface
consumed here (`presets::Minimal`, `EndpointAddr`, `TransportAddr`,
`remote_id()`) is broad enough that this is a real upgrade-risk area.

---

#### 2. Redis (ready registry)

Clients: `redis` 1.0 (`Cargo.toml:57`, features `tokio-comp`,
`connection-manager`, `tokio-rustls-comp`) + `deadpool-redis` 0.23 (`:58`).
The crate never creates a pool — it receives one
(`ReadyRegistry::new(pool, refresh)`, `registry.rs:166-168`), and the relay passes
`state.redis_pool.clone()` (`mesh_boot.rs:447`, from `main.rs:443`). Same pool the
rest of the relay uses (`buzz-pubsub`, session directory), so mesh registry traffic
competes for the same connections.

| Operation | Command | Site | Cadence |
|---|---|---|---|
| publish ready | `SET mesh:ready:{id} <json> EX <ttl>` | `registry.rs:188-194` | once at boot (`mesh_boot.rs:459`) then every 15 s (`runtime.rs:602`) |
| clear ready | `DEL mesh:ready:{id}` | `registry.rs:201-204` | on ready→not-ready edge only (`registry.rs:299-302`) |
| scan | `SCAN <cur> MATCH mesh:ready:* COUNT 100` + `GET` per key | `registry.rs:217-228` | every 5 s (`runtime.rs:311`) **plus once per unknown inbound connection** (`runtime.rs:318`) |

Value codec is **JSON** (`serde_json::to_string`, `registry.rs:185` /
`from_str`, `registry.rs:232`) — the only non-postcard payload in the crate,
chosen for operator legibility.

Key namespace `mesh:ready:` (`registry.rs:19`) is **global**: not community-scoped,
not deployment-prefixed. Multi-deployment isolation rests entirely on the
`relay_pubkey` anchor check in `apply_ready_records` (`membership.rs:90-102`),
which counts rejects into `foreign_relay_rejections` (`status.rs:47`).

##### Failure modes

| Failure | Behaviour | Site |
|---|---|---|
| pool `get()` fails | `MeshError::Transport("redis pool: …")` | `registry.rs:270-272` |
| **first publish fails** | **fatal relay boot** — `anyhow!("mesh ready-registry publish failed: …")` | `mesh_boot.rs:456-463` |
| heartbeat publish fails | warn `"mesh: registry heartbeat tick failed"`, loop continues; the record then TTL-expires after 45 s and peers stop dialing us | `runtime.rs:601-603` |
| reconcile scan fails | warn `"mesh: registry scan failed"`, then dial from the (stale, never-evicted) membership table | `runtime.rs:288-292`, `:307-312` |
| inbound-admission scan fails | warn `"mesh: registry rescan on inbound failed"`, fall through to `has_peer` re-check → connection rejected | `runtime.rs:315-320` |
| malformed / key-mismatched / unattested entry | warn + skip, scan continues | `registry.rs:233-247` |
| Redis transport error mid-scan | whole scan aborts with `MeshError::Redis` (`#[from]`) | `registry.rs:225`, `:228` |
| clear fails on shutdown | error propagates from `tick`, logged as a heartbeat warn; record lingers up to TTL | `registry.rs:300`, `runtime.rs:601` |

Redis outage semantics: existing warm connections keep working (transport is
independent of Redis), but (a) no new peers can be discovered, (b) our record
TTL-expires so *new* pods can't find us, and (c) every unknown inbound connection
costs a failed scan. Nothing in this crate degrades gracefully to a static peer
list.

##### CPU/IO amplification

`scan_ready` performs one `GET` **per key** in a serial loop (`registry.rs:230-231`)
— no `MGET`, no pipelining — and one **secp256k1 schnorr verify per record**
(`registry.rs:233-238` → `registry.rs:70-80`). At 5 s cadence with N pods that is
N verifies per pod per 5 s, plus a full extra scan+verify pass for **every inbound
connection from an unrecognised runtime id** (`runtime.rs:309-320`). See
`-security.md` S-04.

---

#### 3. nostr / secp256k1 (attestation)

`nostr` 0.44 (`Cargo.toml:61`, features `nip44`, `nip98`) is used only in
`registry.rs`:

| Item | Site |
|---|---|
| `nostr::Keys::sign_schnorr` | `registry.rs:41` |
| `nostr::PublicKey::from_hex` / `.to_hex()` / `.xonly()` | `registry.rs:57`, `:39`, `:63` |
| `nostr::secp256k1::{Message, XOnlyPublicKey}`, `schnorr::Signature::from_str` | `registry.rs:11-12`, `:68` |
| `nostr::secp256k1::SECP256K1.verify_schnorr` | `registry.rs:81-82` |
| `sha2::Sha256::digest` over the textual preimage | `registry.rs:94` |

The signing key is the relay's Nostr identity, injected from the consumer:
`boot_mesh(..., relay_keypair: &nostr::Keys, ...)` (`mesh_boot.rs:415`) sourced from
`state.relay_keypair` (`main.rs:445`). The same key anchors acceptance
(`mesh_boot.rs:445` → `membership.rs:61-64`).

Failure modes: every parse/convert/verify error becomes a
`MeshError::Transport(format!(...))` string (`registry.rs:56-83`) — five distinct
failure causes collapse into one untyped variant, so a caller cannot distinguish
"bad hex" from "signature forged." Rejections are logged with `runtime_id`
(`membership.rs:96-101`, `:105-108`; `registry.rs:236-246`) and counted only for the
anchor-mismatch case (`foreign_relay_rejections`), not for signature failures.

---

#### 4. `buzz-core` — **not a dependency**

Verified: `crates/buzz-relay-mesh/Cargo.toml` has no `buzz-*` dependency at all, and
no `buzz_core` import exists in `src/`. The mesh crate is deliberately
Buzz-domain-free: no `CommunityId`, no event kinds, no NIP types, no tenant concept.

The tenant boundary is applied **outside** this crate — consumers thread
`buzz_core::CommunityId` through their own layers (`tunnel/reliable.rs:13`,
`audio/join.rs:41`, `api/mesh_demo.rs:29`) and the fenced session directory scopes
by community. Consequence: **the mesh wire format carries no tenant identifier.**
`FencedHeader` is `{session_id, generation, owner_runtime_id}` (`wire.rs:85-93`) —
community scoping is entirely inside the opaque `payload`, recovered by
`recv_validated`/`community_id()` on the relay side
(`mesh_boot.rs:334-341`, `tunnel/reliable.rs`). Cross-tenant isolation on the mesh
therefore depends on consumer discipline, not on the transport contract.

---

#### 5. postcard

`postcard` 1 with `default-features = false, features = ["use-std"]`
(`Cargo.toml:65`). Used at 6 sites: `wire.rs:178` (`to_extend`), `wire.rs:184`
(`from_bytes`), `gossip.rs:63`, `gossip.rs:67`, and the two error `#[source]`
bindings (`lib.rs:68`, `:70`).

Integration risk: postcard's enum encoding is a varint discriminant with no
unknown-variant escape hatch, and **no wire enum is `#[non_exhaustive]`**
(verified: zero occurrences in `src/`). Combined with the ALPN-per-version rule
(`wire.rs:34-36`) the design is "never mix versions," which is sound but leaves the
`WIRE_VERSION` byte and `GOSSIP_PAYLOAD_VERSION` field as belt-and-braces only.
The gossip payload is doubly-encoded (postcard `GossipMessage` inside a postcard
`MeshStreamFrame::Gossip.payload: Vec<u8>`), which costs a length prefix + a copy
per gossip frame but buys independent evolution (`wire.rs:139-141`).

---

#### 6. `buzz-relay` — how the consumer wires it

Declared at `crates/buzz-relay/Cargo.toml:26` (`buzz-relay-mesh = { workspace = true }`),
path-mapped at root `Cargo.toml:135`, member at `Cargo.toml:27`.

Boot sequence (`crates/buzz-relay/src/mesh_boot.rs:412-521`):

1. `config.mesh.enabled` false → `Ok(None)` (`:417-419`).
2. `MeshEndpoint::bind(config.mesh.bind_addr)` (`:383`) — fatal on error.
3. `advertise_addrs` (`:382-403`): `BUZZ_MESH_ADVERTISE_ADDR` → `POD_IP` + actual
   bound port → all endpoint IP addrs.
4. `GossipRecord::new(runtime_id, addrs, PROTO_VERSION)` + static capabilities
   (`:439-441`).
5. `MeshMembership::new(record).with_expected_relay_pubkey(relay_pubkey_hex)`
   (`:444-445`).
6. `ReadyRegistry::new(pool, registry_refresh)`; `ReadyRecord::new(...)`
   (`:447-453`).
7. `registry.publish_ready(&record)` — **fatal on error** (`:456-463`).
8. `spawn_registry_heartbeat(registry.clone(), record, || !shutting_down)`
   (`:467-471`).
9. `MeshRuntime::start(endpoint, membership, Some(registry))` (`:473`) — spawns 3
   loops.
10. `runtime.reconcile_now().await` (`:478`) — dial seeds immediately.
11. Drain watcher task: polls `shutting_down` every 500 ms, then `begin_drain()` +
    `owners.drain_all()` and returns (`:481-497`).
12. `Arc<dyn RelayMeshMembership>` (`:501`, never read) and
    `Arc<dyn RelayPeerTransport>` (`:502`) erased from the runtime.
13. `transport.set_inbound(Box::new(dispatcher.clone()))` (`:509`).
14. Return `MeshHandle` (`:511-520`).

Then `main.rs:455-459` calls `handle.wire_consumers(...)` **before**
`state.mesh.set(handle)` (`main.rs:457`), so the three profile consumers are
registered before the handle is observable; `main.rs:460` is
`unreachable!("mesh handle is set exactly once, right here")`.

Read path: `AppState::mesh()` (`state.rs:812`) — one caller, `router.rs:381`.

Consumer-side integration points:

| Consumer file | Uses |
|---|---|
| `mesh_boot.rs` | `MeshEndpoint`, `GossipRecord`, `ReadyRecord`/`ReadyRegistry`, `MeshMembership`, `MeshRuntime`, `spawn_registry_heartbeat`, `MeshStatus`, both seams, `InboundHandler`, `Profile`, `StreamRole`, `GoodbyeReason`, `WIRE_VERSION` |
| `audio/mesh.rs` | `FencedHeader`, `MeshDatagram`, `RelayPeerTransport`, `RuntimeId` (`audio/mesh.rs:37,56,260`) |
| `audio/join.rs` | full wire set + `MeshStream`, both half traits, `MeshError` fence variants; defines `HuddleControlMsg` as the opaque `Data` payload (`join.rs:797-801`) |
| `tunnel/reliable.rs` | wire set + `MeshStream`; chunks at 1 MiB against the 16 MiB cap (`reliable.rs:26-31`, assert `:945`) |
| `tunnel/directory.rs` | `MeshError` fence variants only (`:378,395,413,430,824,842,870,914`) |
| `api/mesh_demo.rs` | `RelayPeerTransport` + `MeshEndpoint`/`MeshPeer` in tests |
| `config.rs` | `MeshConfig` (`:136`, `:508`) |
| `audio/handler.rs` | reaches the handle via `AppState` (`:308`, `:446`) |

##### Consumer-side failure modes

- Handle absent (`mesh` off) → every consumer path is a no-op by contract
  (`state.rs:624-626`, `mesh_boot.rs:19-20`).
- Traffic arriving before a dispatcher slot is registered → logged and dropped
  (`mesh_boot.rs:99-104`, `:128-134`); documented as a bounded boot-window race made
  safe by the peer's fenced retry (`mesh_boot.rs:53-55`).
- Second registration on a slot → warn + ignored, first handler wins
  (`mesh_boot.rs:72-74`, `:79-81`, `:85-87`).
- `Profile::RealtimeMedia` arriving as a *stream* → rejected as a protocol violation
  (`mesh_boot.rs:118-126`).
- Mesh runtime loops are never shut down (`MeshRuntime::shutdown` uncalled), so on
  SIGTERM the accept/reconcile/gossip loops and the heartbeat task run until process
  exit.

---

#### 7. Not integrated (absences worth recording)

- **No metrics exporter.** The crate has no `metrics` dependency; the documented
  `mesh_fence_rejections_total` (`lib.rs:102-109`) does not exist. All mesh
  observability is the ad-hoc counters in `MeshStatus` plus `tracing` (28 sites).
- **No tracing spans / OTel.** `tracing` is used only for `info!/warn!/debug!`
  events; no `#[instrument]`, no span propagation across the mesh, so a cross-pod
  session cannot be traced end to end.
- **No `buzz-audit`.** Peer admission and rejection decisions are not audited, only
  logged.
- **No k8s API client.** Peer discovery is Redis-only; `POD_IP` comes from the
  Downward API via env, explicitly to need "zero RBAC" (`mesh_boot.rs:380-381`).
- **No `buzz-db` / Postgres.** The mesh is Redis-only.
