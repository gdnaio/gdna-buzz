## Module: buzz-relay — huddle audio, tunnel & conformance seam (`crates/buzz-relay/src`)
### Aspect: Configuration

---

#### 1. Environment variables consumed (directly or via `Config`)

| Env var | Parse rule | Default | Read at | Consumed in this group at |
|---|---|---|---|---|
| `BUZZ_HUDDLE_AUDIO_AVAILABLE` | `!(v == "false" \|\| v == "0")` — i.e. **anything except `false`/`0` is true**, including typos | **`true`** | `config.rs:487-491` | `handler.rs:357` |
| `BUZZ_MESH` | `eq_ignore_ascii_case("on") \|\| v == "true" \|\| v == "1"` | **`false`** (absent, `off`, or a typo → off) | `config.rs:494-500` | `mesh_boot.rs:418-421` |
| `BUZZ_MESH_BIND_ADDR` | `SocketAddr::parse`; a parse failure is a **hard `ConfigError::InvalidValue`**; an *absent* var falls back to the default | `0.0.0.0:3478` | `config.rs:501-508` | `mesh_boot.rs:423-431`, logged `:436` |
| `BUZZ_MESH_DEMO_ECHO` | same strict `on`/`true`/`1` as `BUZZ_MESH` | **`false`** | `config.rs:514-518` | `mesh_boot.rs:293-297`; gate `api/mesh_demo.rs:70-72`; passed at `main.rs:456` |
| `BUZZ_MESH_ADVERTISE_ADDR` | trimmed; empty string ignored | unset | **read directly from the environment**, not via `Config` | `mesh_boot.rs:384-389` |
| `POD_IP` | trimmed; used only when a bound port is known | unset | **read directly** | `mesh_boot.rs:394-399` |
| `BUZZ_REQUIRE_RELAY_MEMBERSHIP` | `v == "true" \|\| v == "1"` | **`false`** | `config.rs:481-483` | reached via `enforce_relay_membership` at `handler.rs:244`; the gate itself short-circuits at `api/mod.rs:67` |
| `REDIS_URL` | used only by test helpers | `redis://127.0.0.1:6379` | `directory.rs:593`, `reliable.rs:708`, `api/mesh_demo.rs:170` | test-only |

`BUZZ_MESH_ADVERTISE_ADDR` and `POD_IP` are the only two env vars in this group read
with a bare `std::env::var` instead of going through `Config`
(`mesh_boot.rs:384`, `:394`). They are therefore invisible to `Config::from_env`'s
validation and to any config-dump/diagnostic surface.

##### 1.1 Advertise-address preference chain (`mesh_boot.rs:379-402`)

1. `BUZZ_MESH_ADVERTISE_ADDR` if set and non-empty → used verbatim, **no
   `SocketAddr` validation** (`mesh_boot.rs:384-389`). A malformed value is
   published to the ready registry and silently breaks peer dialing.
2. `POD_IP` + the actual bound port, when both are usable
   (`mesh_boot.rs:392-399`). Comment: "k8s Downward API, zero RBAC".
3. Every IP transport address the endpoint reports (`mesh_boot.rs:401`).

The bound port is taken from `ip_addrs().first()` (`mesh_boot.rs:392-393`); if that
is empty the port is `0` and branch 2 is skipped, falling through to an empty
vector from branch 3 — publishing a ready record with **no addresses** and no error.

---

#### 2. `Config` fields consumed

| Field | Type | Default | Definition | Read at |
|---|---|---|---|---|
| `huddle_audio_available` | `bool` | `true` | `config.rs:114-129` | `handler.rs:357` |
| `mesh: buzz_relay_mesh::MeshConfig` | struct | see §2.1 | `config.rs:131-136` | `mesh_boot.rs:418`, `:423`, `:445` |
| `mesh_demo_echo` | `bool` | `false` | `config.rs:138-144` | `main.rs:456` → `mesh_boot.rs:293` |
| `relay_url` | `String` | — | (elsewhere) | `handler.rs:219` via `nip42_expected_relay_url` |
| `require_relay_membership` | `bool` | `false` | `config.rs:110-112` | indirectly, `api/mod.rs:67` |

##### 2.1 `MeshConfig` (`buzz-relay-mesh/src/lib.rs:54-63`)

| Field | Source | Default | Tunable? |
|---|---|---|---|
| `enabled` | `BUZZ_MESH` | `false` | yes |
| `bind_addr` | `BUZZ_MESH_BIND_ADDR` | `0.0.0.0:3478` | yes |
| `registry_refresh` | **hard-coded** `Duration::from_secs(15)` | 15 s | **no** — `config.rs:511` writes a literal |

`registry_refresh` is a `MeshConfig` field with no env var behind it. It controls the
ready-registry heartbeat cadence (`mesh_boot.rs:445`), so ready-record staleness on
a mesh deployment is not operator-tunable.

---

#### 3. Other `AppState` inputs this group depends on

| Input | Where set | Consumed |
|---|---|---|
| `conn_semaphore: Arc<Semaphore>` sized by `max_connections` | `state.rs:727` | `handler.rs:90` — **shared with ordinary relay WebSockets**, so `max_connections` is the only operator-tunable audio limit |
| `community_connections: Arc<CommunityConnectionRegistry>` | `state.rs:724` | `handler.rs:153-164` |
| `audio_rooms: Arc<AudioRoomManager>` | `state.rs:768` | throughout |
| `relay_keypair: nostr::Keys` | `state.rs` | `handler.rs:1268` (lifecycle-event signing); `mesh_boot.rs:441-443`, `:452` (mesh attestation) |
| `redis_pool: deadpool_redis::Pool` | `state.rs` | `main.rs:444` → `mesh_boot.rs:442`, `:512` |
| `shutting_down: Arc<AtomicBool>` | `state.rs:769` | `main.rs:445`, `:457` → `mesh_boot.rs:466-472`, `:481-496`, `:317` |
| `tracer: Arc<dyn Tracer>` | `state.rs:794-798` | **unconditionally `NoopTracer`** |
| `mesh: Arc<OnceLock<MeshHandle>>` | `state.rs:627`, set `main.rs:458` | `state.rs:812-814` |

---

#### 4. `BUZZ_HUDDLE_AUDIO_AVAILABLE` — verifying the documented single-pod constraint

##### 4.1 What the doc says

`config.rs:114-129`:

> Huddle audio frames are relayed peer-to-peer *within a single pod*
> (`AudioRoomManager` is an in-process map; only huddle lifecycle events cross pods
> via Redis). Under horizontal scaling … two peers in the same huddle can land on
> different pods and never hear each other. … Operators running multiple relay pods
> MUST set `BUZZ_HUDDLE_AUDIO_AVAILABLE=false` until the out-of-relay media/SFU
> service lands.

##### 4.2 What actually breaks with more than one pod, mesh off

Verified accurate. `AudioRoomManager` is a per-process `DashMap`
(`room.rs:490-492`), constructed once per `AppState` (`state.rs:768`). Frame
fan-out (`room.rs:398-411`) iterates only that map. Nothing crosses pods except the
three lifecycle events, which go through `pubsub.publish_event`
(`handler.rs:1322-1325`). So two peers on different pods:

- each create their own `Room` for the same `(community, channel)` key;
- each independently pin a protocol version;
- each independently allocate `peer_index` from 0, so indices collide across pods;
- see a `joined` roster containing only their own pod's peers
  (`handler.rs:614-619` from `room.peer_pubkeys()`);
- hear nothing from the other pod;
- **each become "the last peer"** on leave and each run
  `remove_peer_and_check_ended` → `archive_channel` (`handler.rs:833-866`), so the
  channel is archived while the other pod's peer is still connected, and two 48103
  events are emitted (the second de-duplicated only if the event ids collide, which
  they will not — different `created_at`/content ordering).

The failure is worse than "never hear each other": the auto-end/archive path is
actively wrong under multi-pod. So the `MUST` in the doc is well-founded.

##### 4.3 Does the mesh path change that? Yes — and it also bypasses the gate

`handler.rs:306-378` is a two-arm match on `state.mesh()`:

```rust
match state.mesh() {
    Some(mesh) => { /* drain check, resolve_join_owner_ready */ }
    None => { if !state.config.huddle_audio_available { /* reject */ } }
}
```

**`huddle_audio_available` is only consulted on the `None` arm.** With `BUZZ_MESH=on`,
setting `BUZZ_HUDDLE_AUDIO_AVAILABLE=false` has **no effect at all** — joins are
accepted and routed through the mesh. This is intentional (the comment at
`handler.rs:289-296` says "When the mesh is off, we keep today's behavior exactly —
including the `huddle_audio_available=false` rejection under a non-mesh horizontal
deployment") but the *field documentation* at `config.rs:120-128` was written before
the mesh landed and still says operators "MUST set
`BUZZ_HUDDLE_AUDIO_AVAILABLE=false`" with no mention of the mesh. **Documented
delta.** An operator following `config.rs` on a mesh-enabled multi-pod deployment
would set a flag that does nothing.

With the mesh on, multi-pod huddles work correctly: one pod owns the room (Redis
CAS, `join.rs:317-379`), that pod is the sole `peer_index` allocator
(`mesh.rs:18-24`), and ingress pods never archive (`handler.rs:803-810`). So the
mesh does remove the single-pod constraint — the config doc has simply not caught up.

##### 4.4 The `false` value is nearly unreachable

The parse rule is `!(v == "false" || v == "0")` (`config.rs:489-491`). Only the exact
lowercase `false` or `0` disables it. `False`, `FALSE`, `no`, `off`, `disabled`, and
an empty string all evaluate to **true**. Contrast `BUZZ_MESH`, which uses
`eq_ignore_ascii_case("on")` (`config.rs:498-500`) — three flags in the same file
with three different parse conventions:

| Flag | Convention | Case-insensitive? |
|---|---|---|
| `BUZZ_HUDDLE_AUDIO_AVAILABLE` | deny-list (`false`/`0` → off) | no |
| `BUZZ_MESH` | allow-list (`on`/`true`/`1` → on) | `on` only |
| `BUZZ_MESH_DEMO_ECHO` | allow-list (`on`/`true`/`1` → on) | `on` only |
| `BUZZ_REQUIRE_RELAY_MEMBERSHIP` | allow-list (`true`/`1` → on) | no |

##### 4.5 Test coverage of the defaults

| Assertion | Line |
|---|---|
| `huddle_audio_available` defaults true | `config.rs:980-984` |
| `BUZZ_HUDDLE_AUDIO_AVAILABLE=false` disables it | `config.rs:1256-1265` |
| `BUZZ_MESH` absent → mesh off | `mesh_boot.rs:546-556` (skips if externally forced) |
| mesh off boots nothing (unroutable Redis proves it) | `mesh_boot.rs:527-541` |

---

#### 5. `BUZZ_MESH` / `BUZZ_MESH_BIND_ADDR` gating — verified behaviour

| Condition | Behaviour |
|---|---|
| `BUZZ_MESH` absent/off | `boot_mesh` returns `Ok(None)` at `mesh_boot.rs:418-421` before touching anything. `state.mesh()` stays `None`, so every consumer takes its single-pod branch (`handler.rs:306`, `:449`, `:577`, `:875`). `/_mesh` returns `{"enabled": false}` (`router.rs:404`). `/_mesh/demo/echo` 404s (`api/mesh_demo.rs:66-68`). No UDP port bound, no Redis key written, no task spawned — pinned by `mesh_boot.rs:527-541` using an unroutable Redis pool |
| `BUZZ_MESH=on`, bind succeeds, publish succeeds | endpoint bound on `bind_addr`, boot-unique `RuntimeId` (`mesh_boot.rs:423-432`), ready record published (`:459-464`), heartbeat spawned (`:466-472`), `MeshRuntime` started (`:475`), immediate `reconcile_now()` (`:478`), drain watcher spawned (`:481-496`), dispatcher installed as the single inbound slot (`:505`) |
| `BUZZ_MESH=on`, bind fails | **fatal** — `anyhow!("mesh endpoint bind on {} failed: {e}")` propagates to `main.rs:442` and aborts startup (`mesh_boot.rs:423-431`) |
| `BUZZ_MESH=on`, first ready-registry publish fails | **fatal** — `anyhow!("mesh ready-registry publish failed: {e}")` (`mesh_boot.rs:459-463`). Rationale at `:457-458`: "if Redis can't take the attested record, peers can never find us — fail loudly now, not quietly forever" |
| `BUZZ_MESH_BIND_ADDR` malformed | **fatal at config load** — `ConfigError::InvalidValue` (`config.rs:501-508`) |
| `BUZZ_MESH_BIND_ADDR` absent | default `0.0.0.0:3478` (`config.rs:508`) — binds on **all interfaces**. §7 |

`main.rs:454-459` wires consumers, then publishes the handle, then logs. The
`OnceLock::set` failure is `unreachable!("mesh handle is set exactly once, right
here")` (`main.rs:459`) — a genuine single-writer invariant, not a swallowed error.

Ordering note: consumers are registered **before** `state.mesh.set(handle)`, so
inbound mesh traffic can be served before any local join on this pod can resolve
ownership. That is the correct order (a peer must not hit an unregistered slot), and
`mesh_boot.rs:47-55` documents the residual boot-window drop as bounded and safe
because of fencing.

---

#### 6. Hard-coded values that should be tunable

Grouped by operational consequence.

##### 6.1 Capacity / QoS — no env var exists

| Constant | Value | Line | Why it should be tunable |
|---|---|---|---|
| `MAX_PEERS_PER_ROOM` | 25 | `room.rs:50` | The doc itself calls it "the soft one" and shows the `N×(N−1)` math (`room.rs:47-50`). A deployment with fatter pods or smaller huddles has no way to move it. This is the single most likely value an operator would want to change |
| `AUDIO_CHANNEL_CAPACITY` | 8 (≈160 ms) | `room.rs:40` | Jitter tolerance vs. latency tradeoff; hardware- and network-dependent |
| `CTRL_CHANNEL_CAPACITY` | 32 | `room.rs:45` | Sized by a comment ("even 30 simultaneous join/leave events fit") that is tied to `MAX_PEERS_PER_ROOM`; if the peer cap moved, this would need to move with it — and neither can |
| `MAX_AUDIO_FRAME_BYTES` | 4096 | `handler.rs:44` | Caps Opus bitrate implicitly. A higher-fidelity or multi-channel configuration would need this raised |
| WS data / control queue depths | 16 / 8 | `handler.rs:659-660` | Same tradeoff, second layer |
| `roster_tx` broadcast capacity | 64 | `room.rs:179` | Determines how much roster churn a slow control stream can absorb before a full resync |

##### 6.2 Timing — no env var exists

| Constant | Value | Line | Consequence |
|---|---|---|---|
| `HEARTBEAT_INTERVAL` | 30 s | `handler.rs:55` | With `MAX_MISSED_PONGS=3` this fixes dead-peer detection at 60–90 s. Behind an aggressive LB idle timeout this is too slow; on a lossy mobile link, too aggressive |
| `MAX_MISSED_PONGS` | 3 | `handler.rs:58` | as above |
| `AUTH_TIMEOUT` | 5 s | `handler.rs:61` | Also the window during which an unauthenticated client holds a shared `conn_semaphore` permit (see security §6.1) — the one timing constant with a direct DoS consequence |
| `DEFAULT_LEASE_TTL` | 30 s | `directory.rs:17` | `with_lease_ttl` exists (`directory.rs:187-189`) and tests use it (`directory.rs:601`), but production calls `SessionDirectory::new` (`mesh_boot.rs:512`), so the TTL is fixed. TTL is the failover-latency dial for cross-pod huddles |
| `DEFAULT_HUDDLE_RENEW_INTERVAL` | 10 s | `join.rs:452` | Must stay ≪ TTL; coupled to a value that is also fixed |
| `DEFAULT_RENEW_INTERVAL` (reliable) | 10 s | `reliable.rs:34` | duplicate of the above, in a second file |
| `OWNER_READY_RETRY_INTERVAL` / `_MAX_ATTEMPTS` | 20 ms / 25 | `join.rs:387-388` | Bounds a join's worst-case latency at ~500 ms and its Redis round trips at 25 |
| `registry_refresh` | 15 s | `config.rs:511` | A `MeshConfig` field with no env var — see §2.1 |
| drain-watcher poll | 500 ms | `mesh_boot.rs:495` | SIGTERM→drain latency |
| demo-echo drain poll | 100 ms | `mesh_boot.rs:315` | demo only |
| `ECHO_TIMEOUT` | 10 s | `api/mesh_demo.rs:45` | demo only |

##### 6.3 Protocol / wire — arguably correct to hard-code

| Constant | Value | Line | Note |
|---|---|---|---|
| `CURRENT_PROTOCOL_VERSION` | 2 | `handler.rs:123-124` | Correct as a compile-time constant, but there is **no way to pin a deployment to v1** during a staged rollout even though the doc says "older versions stay supported indefinitely for staged rollouts" (`handler.rs:120-122`). A `BUZZ_HUDDLE_MAX_PROTOCOL_VERSION` would make that claim actionable |
| `default_protocol_version()` | 1 | `handler.rs:140-142` | Correct — backwards compatibility |
| `V2_HEADER_LEN` / `FLAG_DTX` | 8 / `0x01` | `wire.rs:29`, `:33` | Wire format; must not be tunable. Duplicated in `desktop/src-tauri/src/huddle/wire.rs:48` — two copies of one protocol constant |
| `MAX_RELIABLE_PAYLOAD_BYTES` | 1 MiB | `reliable.rs:31` | Justified against goose's 50 MiB bodies (`reliable.rs:26-30`) and asserted against the 16 MiB wire cap (`reliable.rs:945`) |
| `ReliableWireFrame::VERSION` | 1 | `reliable.rs:423` | wire format |
| `PROTO_VERSION` | `WIRE_VERSION as u16` | `mesh_boot.rs:367` | derived, correct |
| `capabilities()` | all three profiles, static | `mesh_boot.rs:369-377` | Correct — "All three tunnel profiles ship in the same binary". But it means a pod cannot advertise *not* serving huddles, so there is no capability-based way to keep huddle ownership off a given pod |

##### 6.4 Conformance — no configuration at all

| Item | Status |
|---|---|
| Tracer selection | **unconditional** `NoopTracer` at `state.rs:794-798`. No env var, no config field, no cargo feature. `crates/buzz-relay/Cargo.toml` `[features]` contains only `dev = ["buzz-auth/dev"]` |
| `JsonlTracer` output path | `create<P: AsRef<Path>>` (`tracers.rs:36`) takes a path — but nothing calls it, so no path is ever configured |
| Feature-gated elision | `tracers.rs:11-13` says "the build can have the compiler eliminate them entirely behind a feature flag if the cost ever shows up in benches". No such flag exists, so the emit arguments are constructed and discarded on every ingest and REQ |
| Seam names | `"ingest_event_exited_without_trace"` (`ingest.rs:1385`), `"row_community_lookup_missing"` (`conformance/mod.rs:251`) — string literals, correctly not configurable |

Making the tracer selectable (`BUZZ_CONFORMANCE_TRACE=<path>`) is the single change
that would turn this module from inert scaffolding into an operable diagnostic.

---

#### 7. Deployment-facing configuration notes

- **`0.0.0.0:3478` binds all interfaces by default.** `config.rs:508`. On a
  mesh-enabled deployment the mesh QUIC endpoint is reachable from anywhere the pod
  is. Admission is gated on the Redis ready registry
  (`buzz-relay-mesh/src/runtime.rs:275-283`), so an unattested dialer is rejected,
  but the port is open and the default is not a loopback or pod-network address.
  `3478` is the IANA STUN port — coincidental, and potentially confusing to a
  network operator reading a port scan.
- **`/_mesh/demo/echo` is on the public router** (`router.rs:123`), not the admin
  host, and is unauthenticated when both mesh flags are on. See security §5.
- **`/_mesh` is on the health router** (`router.rs:230`) which
  `router.rs:222-224` documents as having "No metrics middleware, no auth, no CORS,
  no body limit". It returns peer runtime ids and addresses.
- **`max_connections` is the only operator dial that affects huddle capacity**, and
  it is shared with every other relay WebSocket (`state.rs:727`, `handler.rs:90`).
  There is no way to reserve or cap the audio share of that budget.
- **Nothing in `.env.example` or `deploy/compose/.env.example`** was found to
  document `BUZZ_HUDDLE_AUDIO_AVAILABLE`, `BUZZ_MESH`, `BUZZ_MESH_BIND_ADDR`,
  `BUZZ_MESH_DEMO_ECHO`, `BUZZ_MESH_ADVERTISE_ADDR`, or `POD_IP` — operators
  discover them only from `config.rs` doc comments and `mesh_boot.rs`.
- **Redis is a hard dependency for mesh-enabled huddles**, not a cache: joins fail
  (`join_rejected`) when the directory is unreachable, and a renewal failure is
  treated as owner loss, closing every local owner client on that pod
  (`join.rs:521-529` → `handler.rs:756-765`).
