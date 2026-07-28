## Module: buzz-relay-mesh (`crates/buzz-relay-mesh`)
### Aspect: Configuration

The crate reads **no environment variables itself** — verified: `std::env` appears
nowhere in `crates/buzz-relay-mesh/src`. All configuration is resolved by
`buzz-relay` and passed in as `MeshConfig` (`lib.rs:53-64`) plus two direct env
reads in `mesh_boot.rs`.

---

#### 1. Environment variables

Complete set (grep for `BUZZ_MESH` + `POD_IP` across the repo, excluding the
unrelated MeshLLM `BUZZ_MESH_API_PORT`/`BUZZ_MESH_CONSOLE_PORT`/
`BUZZ_MESH_IROH_RELAYS` desktop vars, `desktop/src-tauri/src/mesh_llm/mod.rs:37-47`):

| Variable | Default | Accepted values | Resolved at | Consumed at |
|---|---|---|---|---|
| `BUZZ_MESH` | **off** | `on` (case-insensitive), `true`, `1` → enabled; **absent, `off`, or any other value → disabled** | `config.rs:498-500` | `mesh_boot.rs:417` |
| `BUZZ_MESH_BIND_ADDR` | `0.0.0.0:3478` | any parseable `SocketAddr`; parse failure is a **fatal `ConfigError::InvalidValue`** | `config.rs:501-507` | `MeshEndpoint::bind`, `mesh_boot.rs:383`; logged `mesh_boot.rs:394` |
| `BUZZ_MESH_ADVERTISE_ADDR` | unset | free-form string, trimmed; empty → ignored. **Not validated as a `SocketAddr` at boot** — a typo is only discovered when a *peer* fails to parse it (`runtime.rs:328-334`) | read inline, `mesh_boot.rs:384-389` | published in `ReadyRecord.endpoint_addrs` / `GossipRecord.endpoint_addrs` |
| `POD_IP` | unset | IP string, trimmed; used only when non-empty **and** a bound port was resolved | read inline, `mesh_boot.rs:398-403` | same |
| `BUZZ_MESH_DEMO_ECHO` | **off** | `on`/`true`/`1` only (same strict parser) | `config.rs:516-518` | `main.rs:456` → `wire_consumers(demo_echo)`; route gate `api/mesh_demo.rs`; owner-side echo `mesh_boot.rs:290-292` |

Two more relay-level vars materially affect the mesh's exposure:

| Variable | Default | Relevance |
|---|---|---|
| `BUZZ_HEALTH_PORT` (`config.health_port`) | 8080 | the listener carrying `GET /_mesh` (`router.rs:230`), bound to **hard-coded `0.0.0.0`** at `main.rs:1119`, ignoring `BUZZ_BIND_ADDR` |
| `BUZZ_RELAY_PRIVATE_KEY` (→ `state.relay_keypair`) | — | the secp256k1 key that signs and anchors every ready-record attestation (`mesh_boot.rs:445`, `:450`) |

##### Parser semantics (verified)

```rust
// config.rs:498-500
let mesh_enabled = std::env::var("BUZZ_MESH")
    .map(|v| v.eq_ignore_ascii_case("on") || v == "true" || v == "1")
    .unwrap_or(false);
```

Note the asymmetry: `on` is case-insensitive, `true`/`1` are exact. `TRUE`, `On `
(trailing space), `yes`, `enabled` all evaluate to **off**. Deliberate — rationale at
`config.rs:494-497`: "an image upgrade with untouched env must not bind a new UDP
port or write a new Redis key… anything else (absent, `off`, other values) keeps
exact single-instance behavior." `BUZZ_MESH_DEMO_ECHO` uses the identical parser
(`config.rs:514-518`, comment "same strict pattern as BUZZ_MESH — explicit
`on`/`true`/`1` only, anything else (absent, `off`, typos) is off"). The cost of the
strictness is silent misconfiguration: `BUZZ_MESH=ON` yields a meshless relay with
no warning.

`BUZZ_MESH_BIND_ADDR` uses `unwrap_or_else(|_| Ok(default))?` (`config.rs:502-507`),
so a **missing** var falls back to the default while a **malformed** var is fatal —
the correct split.

---

#### 2. `MeshConfig` fields

`lib.rs:53-64`, `#[derive(Clone, Debug)]`, no `Default`, no `Deserialize`.
Constructed only at `config.rs:508-512`; stored as `Config.mesh` (`config.rs:136`).

| Field | Type | Value source | Default | Read at |
|---|---|---|---|---|
| `enabled` | `bool` | `BUZZ_MESH` | `false` | `mesh_boot.rs:417` |
| `bind_addr` | `std::net::SocketAddr` | `BUZZ_MESH_BIND_ADDR` | `0.0.0.0:3478` | `mesh_boot.rs:383`, `:396` |
| `registry_refresh` | `std::time::Duration` | **hardcoded** | `15s` (`config.rs:511`) | `mesh_boot.rs:447` → `ReadyRegistry::new` |

**`registry_refresh` has no environment variable.** `config.rs:511` is a literal
`std::time::Duration::from_secs(15)`. The crate exposes
`DEFAULT_REGISTRY_REFRESH = 15s` (`registry.rs:20`) for exactly this purpose and the
relay does not use it (grep: zero external callers). Operators cannot tune heartbeat
cadence, and therefore cannot tune the derived **45 s registry TTL**
(`REGISTRY_EXPIRY_MULTIPLIER = 3`, `registry.rs:21`; `expiry_for`, `:154-156`) — the
window in which a crashed pod remains dialable.

##### Documented-default delta

`lib.rs:55-56` documents `BUZZ_MESH` as "`on` (default when replicas can exist) |
`off` kill switch." The implementation defaults it **off**, unconditionally, with no
replica detection. `config.rs:131-135` and `mesh_boot.rs:4-6` state the correct
behaviour, and `mesh_boot.rs:544-555` tests it
(`mesh_defaults_off_when_env_absent`). The `lib.rs` doc comment is stale — it
describes the design as originally proposed, before the review blocker noted at
`mesh_boot.rs:544-546` ("Blocker fix (Wren review of 8b077fdb): absent `BUZZ_MESH`,
the mesh is OFF"). Also, "kill switch" implies default-on semantics that no longer
apply.

---

#### 3. Compile-time constants (the real tuning surface)

None of these are configurable at runtime; all require a rebuild.

| Constant | Value | Location | Overridable? |
|---|---|---|---|
| `DEFAULT_GOSSIP_INTERVAL` | 2 s | `runtime.rs:44` | only via `MeshRuntime::start_with_intervals` (`:102`) — **zero external callers**, tests only |
| `DEFAULT_RECONCILE_INTERVAL` | 5 s | `runtime.rs:42` | same |
| `CONTROL_QUEUE_DEPTH` | 64 | `runtime.rs:46` | no — private const |
| `DEFAULT_PHI_SUSPECT_THRESHOLD` | 8.0 | `membership.rs:17` | only via `with_phi_suspect_threshold` (`:66`) — **zero callers**; the relay never overrides it (`mesh_boot.rs:444-445`) |
| `PhiAccrual` sample window | 100 | `gossip.rs:176` (`Default::new(100)`) | only via `PhiAccrual::new` — zero external callers |
| `DEFAULT_REGISTRY_REFRESH` | 15 s | `registry.rs:20` | unused by the relay (hardcoded duplicate at `config.rs:511`) |
| `REGISTRY_EXPIRY_MULTIPLIER` | 3 | `registry.rs:21` | no |
| `READY_KEY_PREFIX` | `"mesh:ready:"` | `registry.rs:19` | no |
| `ATTESTATION_CONTEXT` | `"buzz-relay-mesh-ready-v1"` | `registry.rs:22` | no |
| `GOSSIP_PAYLOAD_VERSION` | 1 | `gossip.rs:13` | no |
| `ALPN` | `b"buzz/mesh/1"` | `wire.rs:37` | no |
| `WIRE_VERSION` | 1 | `wire.rs:42` | no |
| `MAX_STREAM_FRAME` | 16 MiB | `wire.rs:46` | no |
| `PROTO_VERSION` (relay) | `WIRE_VERSION as u16` = 1 | `mesh_boot.rs:367` | no |
| capabilities list | `["reliable-stream","realtime-media","huddle-control"]` | `mesh_boot.rs:371-377` | no — static, "all three tunnel profiles ship in the same binary" |
| drain-watcher poll | 500 ms | `mesh_boot.rs:495` | no |
| demo-echo drain tick | 100 ms | `mesh_boot.rs:319` | no |
| demo-echo `ECHO_TIMEOUT` | 10 s | `api/mesh_demo.rs:~41` | no |

Derived timings worth recording: with the shipped values a peer is marked
`Suspect` after `8 × ln(10) × 2s ≈ 36.8 s` of gossip silence (BR-MESH-34), its
registry record expires after 45 s, and a dead peer is re-dialed every 5 s
indefinitely with no backoff.

---

#### 4. Address advertisement resolution

`advertise_addrs(&MeshEndpoint)` (`mesh_boot.rs:382-403`), documented at
`mesh_boot.rs:379-381`. Strict precedence, first match wins:

1. **`BUZZ_MESH_ADVERTISE_ADDR`** (`:384-389`) — trimmed; empty string falls through.
   Returns a single-element list. Intended for "explicit, classic-LB shapes."
2. **`POD_IP` + actual bound port** (`:391-403`) — the port comes from
   `endpoint.ip_addrs().first().port()`, i.e. the *real* bound port, so
   `BUZZ_MESH_BIND_ADDR=0.0.0.0:0` works. Requires `bound_port != 0`. Intended for
   k8s Downward API with "zero RBAC."
3. **All endpoint IP transport addrs** (`:403`) — `endpoint.ip_addrs()`
   (`endpoint.rs:61-71`, filters `TransportAddr::Ip`, drops relay paths). Dev/local
   fallback.

Consumed identically by both the Redis record (`ReadyRecord::new(…, addrs, …)`,
`mesh_boot.rs:448-453`) and the gossip record (`GossipRecord::new(runtime_id, addrs,
PROTO_VERSION)`, `mesh_boot.rs:439`).

Gaps:

- `BUZZ_MESH_ADVERTISE_ADDR` is **never parsed at boot**. It is stored as a `String`
  (`registry.rs:105-107` explains the string choice as layer independence) and only
  parsed by the *dialing peer* (`runtime.rs:329-335`), where a bad value produces
  `warn!("mesh: bad peer addr")` on every remote pod every 5 s. A typo is silently
  self-isolating.
- Path 3 can advertise loopback or a container-internal address in environments
  where neither env var is set, producing a mesh that forms only within one host.
- The advertised list is captured **once at boot** (`mesh_boot.rs:385`) and republished
  verbatim by the heartbeat (`ReadyRecord` is moved into `spawn_registry_heartbeat`,
  `mesh_boot.rs:467-471`). If the pod's IP changes, the record never updates.

---

#### 5. Deployment configuration state

- **No deployment in this repo enables the mesh.** grep for `BUZZ_MESH` across
  `.env.example`, `deploy/`, and the helm charts returns nothing. There is likewise
  no `3478` port declaration and no `POD_IP` Downward-API stanza anywhere in the repo.
  The k8s wiring the code anticipates (`lib.rs:59-61` "Excluded from istio sidecar
  capture in k8s"; `mesh_boot.rs:380` Downward API) lives in the external
  `squareup/block-coder-tf-stacks` repo, if at all.
- **`BUZZ_HUDDLE_AUDIO_AVAILABLE`** is the current answer to horizontal scaling:
  `config.rs:120-129` instructs operators running multiple relay pods to set it
  `false` "until the out-of-relay media/SFU service lands." That is the shipped
  multi-pod story; the mesh is the not-yet-enabled replacement.
- `BUZZ_MESH_BIND_ADDR` defaults to **`0.0.0.0`** — all interfaces. In any
  environment where the mesh is enabled without network policy, the QUIC endpoint is
  publicly reachable (see `-security.md` F-04, F-06).

---

#### 6. Configuration-related test coverage

| Test | Asserts | Location |
|---|---|---|
| `mesh_off_boots_nothing` | `boot_mesh` returns `None` and never touches Redis (pool points at unroutable `redis://127.0.0.1:1`) | `mesh_boot.rs:526-541` |
| `mesh_defaults_off_when_env_absent` | absent `BUZZ_MESH` ⇒ `config.mesh.enabled == false`; self-skips if the var is externally forced | `mesh_boot.rs:544-555` |
| `expiry_is_three_refreshes` | `expiry_for(15s) == 45s` | `registry.rs:324-327` |
| `ready_key_is_stable_and_namespaced` | `ready_key(rid(0xAB)) == "mesh:ready:" + "ab"*32` | `registry.rs:317-322` |

Untested: `BUZZ_MESH_BIND_ADDR` parse-failure fatality, `advertise_addrs`'
three-way precedence (all three branches), `BUZZ_MESH_DEMO_ECHO` gating, and the
`registry_refresh` value flowing through to the Redis TTL.

---

#### 7. Summary of configuration findings

1. `registry_refresh` (and hence the 45 s TTL) is unreachable from the environment
   despite `MeshConfig` exposing the field and `registry.rs:20` providing the default
   constant.
2. Gossip interval, reconcile interval, phi threshold, and phi sample window are all
   API-overridable but **have zero callers** — effectively compile-time constants.
3. `lib.rs:55-56` documents the wrong default for `BUZZ_MESH` (says on, is off).
4. `BUZZ_MESH_ADVERTISE_ADDR` accepts an unvalidated string; failures surface on
   remote pods, not at boot.
5. The strict-opt-in parsers silently ignore near-miss values (`ON`, `yes`) with no
   warning.
6. `/_mesh` rides a listener whose bind address is hard-coded `0.0.0.0`
   (`main.rs:1119`) and cannot be restricted by `BUZZ_BIND_ADDR`.
