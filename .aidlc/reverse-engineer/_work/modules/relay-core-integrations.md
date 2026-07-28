## Module: buzz-relay — core & bootstrap (`crates/buzz-relay/src`)
### Aspect: Integrations

---

#### 1. External systems wired at startup, in order

The order below is the literal execution order of `main()` (`main.rs:83-1060`).

| Step | System | Call site | Fatal on failure? | Notes |
|------|--------|-----------|-------------------|-------|
| 1 | rustls `CryptoProvider` (ring) | `main.rs:88-90` | **yes** (`expect` panic) | Required before any TLS: `rediss://` to ElastiCache, `wss://`, S3 over TLS. Both `aws-lc-rs` and `ring` are in the tree transitively so rustls cannot auto-select (`main.rs:84-87`). |
| 2 | OTLP trace exporter (gRPC/tonic) | `telemetry.rs:85-88` via `main.rs:100` | **no** | Skipped entirely when `OTEL_EXPORTER_OTLP_ENDPOINT` is unset (`telemetry.rs:80-83`). A build failure returns `ExporterBuildFailed` and is logged as `warn` after the subscriber is up (`main.rs:118-120`). |
| 3 | Prometheus exporter HTTP listener | `metrics.rs:73`, spawned `metrics.rs:146` | **yes** (`expect` at `metrics.rs:143/145`) | Binds `0.0.0.0:metrics_port` (default 9102). Panics if the port is taken or a recorder already exists. |
| 4 | Postgres — main pool (+ optional read replica) | `main.rs:151-155` | **yes** | `DbConfig { database_url, read_database_url, ..default }` (`main.rs:145-149`) ⇒ max 20 / min 2 / acquire 3 s / lifetime 1800 s / idle 600 s (`crates/buzz-db/src/lib.rs:247-257`). Replica pool gets the *same* sizing (`crates/buzz-db/src/lib.rs:381-386`). |
| 5 | Postgres — migrations | `main.rs:164-168` | **yes**, but only runs when `BUZZ_AUTO_MIGRATE` is truthy (`main.rs:161-172`) | |
| 6 | Postgres — partition pre-creation (3 ahead) | `main.rs:173-176` | **no** — `error!` only | |
| 7 | Postgres — replica freshness fence probe | `main.rs:186-196` | **no** — `error!`, fence stays closed, all cursor reads go to the writer | Deliberately after step 5 so the commit-time floor guard is checked against the live schema (`main.rs:177-183`). |
| 8 | Postgres — community/owner/allowlist/d_tag bootstrap | `main.rs:220-320` | conditional: fatal iff `require_relay_membership` (steps for community, backfill, owner); `backfill_d_tags` always non-fatal | |
| 9 | Postgres — **audit pool** (separate) | `main.rs:325-329` | **yes** | `PgPoolOptions::new().max_connections(5).min_connections(1)`. No acquire timeout, lifetime, or idle timeout set. Only created when `audit_enabled`. |
| 10 | Redis — deadpool pool | `main.rs:336-341` | **yes** | `deadpool_redis::Config::from_url(redis_url)`, `PoolConfig::new(config.redis_pool_size)` (default 16). Cloned once for the readiness handler (`main.rs:333`, comment: cheap Arc clone). |
| 11 | Redis — `PubSubManager` (dedicated connection) | `main.rs:343-347` | **yes** | |
| 12 | Redis — 3 subscriber tasks | `main.rs:350-367` | n/a (spawned) | events, cache invalidation, conn-control. |
| 13 | Postgres — **search pool** (third pool) | `main.rs:378-382` | **yes** | `PgPoolOptions::new().connect(search_db_url)` — **no sizing knobs set at all**, sqlx defaults apply. Prefers `read_database_url` when set (`main.rs:373-377`). |
| 14 | Workflow engine (in-process, DB-backed) | `main.rs:389-390` | n/a | `WorkflowConfig::default()` — no env-driven workflow config is read here. |
| 15 | Relay signing keypair | `main.rs:392-414` | **yes** (`panic!` at `main.rs:409` when `require_auth_token` and no key) | Dev fallback uses hard-coded `0x…01` with a `warn` (`main.rs:396-408`). |
| 16 | S3 / MinIO — media storage | `main.rs:415-421` | **yes** | `config.media.validate()` then `MediaStorage::new`. |
| 17 | S3 / MinIO — git object store | inside `AppState::new`, `state.rs:694-701` | **yes** (`expect`, `state.rs:701`) | Reuses the same `media.s3_*` credentials/bucket/region. The `expect` message asserts media storage already validated the same config — true only because step 16 precedes `AppState::new`. |
| 18 | Local filesystem — git pack cache | `state.rs:702-709` | **yes** (`expect`, `state.rs:708`) | Directory itself was already `create_dir_all`'d during config load (`config.rs:390-397`). |
| 19 | Redis — NIP-98 replay guard | `state.rs:710-711` | n/a (lazy, uses the pool) | Doc is explicit: must stay Redis-backed and community-keyed; process-local caching would break cross-pod replay freshness (`state.rs:576-581`). |
| 20 | Redis — admission rate limiter | `state.rs:712` | n/a (lazy) | `RedisRateLimiter::new(redis_pool.clone())` — the real implementation lives at `crates/buzz-pubsub/src/rate_limiter.rs:88-99`. |
| 21 | Inter-relay QUIC mesh (UDP bind + Redis registry) | `main.rs:442-451` | **yes when enabled** (`?` at `main.rs:451`) | Off by default: `boot_mesh` returns `None` ⇒ nothing bound/published/spawned. Consumers wired at `main.rs:455-459` **before** the handle is published to `AppState.mesh`. |
| 22 | S3 / MinIO — A3 conformance probe | `main.rs:466-503` | **yes** | Runs by default (opt-out `BUZZ_GIT_CONFORMANCE_PROBE=false`). Races `BUZZ_GIT_PROBE_WRITERS` (default 32) writers over `BUZZ_GIT_PROBE_ROUNDS` (default 3) rounds against the pointer CAS. A backend that cannot do linearizable conditional writes aborts startup. |
| 23 | APNs push gateway (HTTPS, outbound) | matcher/worker spawned `main.rs:686-691`; timeout applied `push_runtime.rs:314` | **no** — the workers simply are not spawned when `push_gateway_delivery_url` is `None` | URL must be exactly `https://…/v1/deliveries/apns` (`config.rs:342-360`). Default when unset is the hard-coded `https://push.buzz.xyz/v1/deliveries/apns` (`config.rs:332`, `config.rs:752-757`) — **outbound push integration is on by default**. |
| 24 | TCP listeners: health then app | `main.rs:1116` then `main.rs:1157` | **yes** for both | Health binds first so probes answer as early as possible. |
| 25 | Unix domain socket (optional) | `main.rs:1178-1187` | **yes when configured** | Pre-existing non-socket file at the path is fatal (`main.rs:1168-1172`). |

#### 2. Postgres connection accounting

| Pool | Created | max | min | Instrumented? |
|------|---------|-----|-----|---------------|
| Main writer | `main.rs:151` → `crates/buzz-db/src/lib.rs:361` | 20 | 2 | yes (`main.rs:956-959`) |
| Read replica (if `READ_DATABASE_URL`) | `crates/buzz-db/src/lib.rs:363` | 20 | 2 | yes (`main.rs:962-966`) |
| Audit | `main.rs:325-329` | 5 | 1 | **no** |
| Search | `main.rs:378-382` | sqlx default (unset) | sqlx default | **no** |

**Verified doc drift:** `DbConfig::default()`'s doc says "At 20 main + 5 audit = 25/pod, four relay pods fit within the PG limit" (`crates/buzz-db/src/lib.rs:244-246`). `main.rs` opens a **third** pool for search (`main.rs:378-382`) whose size is never set, and a replica pool at the same 20. The actual per-pod ceiling is 20 + 5 + search-default, plus 20 more with a replica — not 25. The "four pods within PG max_connections=100" arithmetic therefore does not hold.

Note also: the search pool connects to the **replica** when one is configured (`main.rs:373-377`), deliberately bypassing the freshness fence because search is lag-tolerant (`main.rs:369-372`).

#### 3. Failure behaviour if each dependency is unavailable

| Dependency | At startup | At runtime |
|-----------|-----------|-----------|
| Postgres (main) | abort with `DB connection failed` (`main.rs:151-155`) | `/_readiness` → 503 with `{"postgres": false}` (`router.rs:352-373`); host→community binding fails closed ⇒ every new WS gets a generic 404 (`router.rs:288-299`); NIP-11 still served, `icon` omitted (`nip11.rs:277-286`) |
| Postgres (audit) | abort (`main.rs:329`) | per-entry `error!` + `buzz_audit_log_errors_total`; worker survives (`state.rs:1200-1206`); events are still accepted (audit is off the critical path) |
| Postgres (search) | abort (`main.rs:382`) | search REQ/`POST /query` fail; not surfaced in readiness |
| Redis | abort on pool creation or `PubSubManager::new` (`main.rs:336-347`) | `/_readiness` → 503 with `{"redis": false}` (`router.rs:353-373`); **rate-limit admission fails closed and denies** (`admission.rs:29-33`); NIP-98 replay guard unavailable ⇒ NIP-98 routes fail; cross-pod fan-out / cache invalidation / ban propagation silently stop |
| S3 / MinIO | abort — media storage init (`main.rs:419-421`), git store (`state.rs:701`), and A3 probe (`main.rs:488-495`) are all fatal | media + git request failures; the storage sweep is the only path with an explicit kill switch for missing `s3:ListBucket` (`main.rs:1436-1441`) |
| OTLP collector | never fatal (`telemetry.rs:80-90`) | batch exporter drops spans; shutdown error is `warn` (`main.rs:1054-1057`) |
| Mesh peers / Redis registry | fatal **only when `BUZZ_MESH=on`** (`main.rs:451`) | `/_mesh` reports peer state; fence-rejection counters exposed |
| APNs push gateway | not contacted at startup | per-request timeout `push_gateway_timeout` (default 2000 ms, `config.rs:759-773`) applied at `push_runtime.rs:314` |

#### 4. Retry / backoff inventory

There is **no exponential backoff anywhere in this file group**. Every retry is a fixed-interval loop or a single attempt.

| Path | Strategy | Cite |
|------|----------|------|
| Postgres connect (all 3 pools) | none — single attempt, abort | `main.rs:151`, `main.rs:329`, `main.rs:382` |
| Redis pool | deadpool acquires lazily per use; no relay-side retry | `main.rs:336-341` |
| Channel-event reconciliation | fixed 5 s × 24 attempts (≈2 min), then gives up silently | `main.rs:570-590` |
| NIP-43 snapshot reconciliation | fixed interval, default 60 s, indefinite | `main.rs:517-545` |
| Ephemeral reaper | fixed interval, default 60 s; failed tick `continue`s | `main.rs:609-620` |
| Reminder scheduler | fixed interval, default 10 s; failed publish releases the claim so the next tick retries — **unless the release itself fails, in which case the reminder is never retried** | `main.rs:701-798` |
| Community revalidator | fixed interval, default 30 s clamped to `1..=300`; a failed per-community lookup simply waits for the next tick | `main.rs:882-890`, `state.rs:1076-1087` |
| Pool metrics | fixed interval, default 10 s, `.max(1)` | `main.rs:945-949` |
| Usage metrics | fixed interval, default 300 s, `.max(5)`, first tick jittered by `rand % interval`; `MissedTickBehavior::Skip` | `main.rs:1009-1022` |
| Cross-pod publishes (cache invalidation, ban) | fire-and-forget, **no retry**; backstopped by ≤10 s cache TTL and the durable DB ban row respectively | `state.rs:964-978`, `state.rs:1039-1053` |
| Community-archive publish | **awaited** (not fire-and-forget) so the archive API can offer a retryable response; plus a periodic durable revalidation backstop | `state.rs:1055-1071`, `main.rs:869-890` |
| Redis broadcast consumers | `Lagged` is counted and tolerated; `Closed` **breaks the loop permanently** | `main.rs:834-843`, `:864-873`, `:925-934` |
| Storage sweep | single-flight with `BUZZ_STORAGE_SWEEP_TIMEOUT_SECS`; last cached snapshot re-emitted | `main.rs:1442-1477`, `storage_sweep.rs:56-72` |
| A3 conformance probe | `race_width` × `race_rounds` races, single overall attempt, fatal | `main.rs:472-502` |

#### 5. Crate-level dependency integration (`Cargo.toml`)

Internal crates depended on: `buzz-core`, `buzz-conformance`, `buzz-db`, `buzz-auth`, `buzz-pubsub`, `buzz-audit`, `buzz-search`, `buzz-relay-mesh`, `buzz-sdk`, `buzz-workflow` (with `reqwest` feature), `buzz-media` (`Cargo.toml:19-27`, `:66-68`).

Notable third-party pins that are **not** workspace-managed (each pinned locally):
- `rustls 0.23` with `default-features = false, features = ["ring","std"]` (`Cargo.toml:57`) — deliberate, paired with `main.rs:88-90`.
- `s3 = { version = "0.37", package = "rust-s3", features = ["tokio-rustls-tls","fail-on-err","tags"] }` (`Cargo.toml:61`).
- `async-trait 0.1` (`Cargo.toml:28`) — but `tenant.rs:29-30` explicitly says it uses native `async fn` in trait "no `async-trait` dependency", and `HostResolver` (`tenant.rs:31`) does. So the `async-trait` dep is used elsewhere in the crate, not by the tenancy seam.
- `base64 0.22`, `tempfile 3`, `bytes 1`, `infer 0.19`, `pulldown-cmark 0.13.4`, `async-compression 0.4.42` (`Cargo.toml:60`, `:62-64`, `:78-79`).
- `tokio-util` needs the extra `io` feature for git stdout streaming (`Cargo.toml:31-34`).

Dev-dependencies pull **two git-sourced crates from an external GitHub org**: `mesh-llm-sdk` and `mesh-llm-host-runtime`, both `git = "https://github.com/Mesh-LLM/mesh-llm.git", tag = "v0.73.1"` (`Cargo.toml:84-85`). These are test-only but make `cargo test -p buzz-relay` require network access to a third-party repo.

`buzz-relay` version is deliberately independent of the workspace: `0.2.0` (`Cargo.toml:7`, rationale `Cargo.toml:4-6`) and is what `NIP-11.version` reports (`nip11.rs:161`).

#### 6. Nothing depends on `buzz-relay` as a library

Verified: no crate in the workspace and nothing under `desktop/src-tauri/**` declares `buzz-relay` as a dependency (`buzz-admin`, `buzz-conformance`, `buzz-relay-mesh`, `git-sign-nostr` mention the name only in comments). The single external textual reference is a doc mention of `buzz_relay::handlers::event::tests::`. Consequence: the crate's entire `pub` surface exists solely for `main.rs` and in-crate consumers, and `lib.rs:53-55`'s three re-exports have no consumer at all.
