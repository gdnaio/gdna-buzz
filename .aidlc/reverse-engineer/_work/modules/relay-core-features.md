## Module: buzz-relay — core & bootstrap (`crates/buzz-relay/src`)
### Aspect: Features

---

#### 1. What bootstrap enables

`main()` (`main.rs:83-1060`) is the single composition root. Everything the relay can do is switched on here.

| Capability | Enabled by | Gate |
|-----------|-----------|------|
| NIP-01 WebSocket relay | `build_router` (`router.rs:32`) + `handle_connection` via `router.rs:316` | always |
| NIP-11 relay info (content-negotiated + `/info`) | `nip11.rs:235`, `router.rs:63-64` | always, unauthenticated |
| NIP-05 discovery | `router.rs:65` | always |
| Nostr-over-HTTP bridge (`/events`, `/query`, `/count`) | `router.rs:71-73` | always, NIP-98 |
| Blossom media upload/download | `media_router` (`router.rs:37-45`) | always; GET auth off by default (`config.rs:197`) |
| Git smart HTTP | `git_router` (`api/git/transport.rs:1756`) | always, but gated behind the fatal A3 probe (`main.rs:466-503`) |
| Git policy hook callback | `git_policy_router` (`api/git/mod.rs:60`) | always, localhost-only |
| Huddle audio WebSocket | `router.rs:125-128` | `huddle_audio_available` (default `true`, `config.rs:489-491`) checked at `audio/handler.rs:357` |
| Relay invites + join policy | `router.rs:95-111` | join-policy routes return content only when `config.join_policy.is_some()` (`config.rs:793-810`) |
| Operator community provisioning | `router.rs:74-93` | `relay_operator_pubkeys` non-empty (fail-closed default: disabled, `config.rs:161-169`) |
| Deployment-admin API + SPA | `router.rs:47-54` | `BUZZ_ADMIN_HOST` set (`config.rs:813-838`) |
| Public web bundle (invite landing / git browser) | SPA fallback `router.rs:145-183` | `BUZZ_WEB_DIR` set; git-browser routes additionally need `BUZZ_SERVE_GIT_WEB_GUI` |
| Moderation queue reads | `router.rs:113-118` | always, NIP-98 + mod-authz |
| Workflow webhooks | `router.rs:120` | always, secret-authenticated |
| Prometheus `/metrics` | `metrics::install` (`main.rs:138`) | always |
| Distributed tracing (OTLP) | `telemetry::try_init_tracer` (`main.rs:100`) | `OTEL_EXPORTER_OTLP_ENDPOINT` set (`telemetry.rs:80-83`) |
| Tamper-evident audit log | separate PG pool + worker (`main.rs:322-334`, `state.rs:653-690`) | `BUZZ_AUDIT_ENABLED` (default `true`, `config.rs:793`) |
| Inter-relay QUIC mesh | `mesh_boot::boot_mesh` (`main.rs:442`) | `BUZZ_MESH=on` (default off, `config.rs:492-509`) |
| Mesh reliable-stream echo probe | `handle.wire_consumers(..., mesh_demo_echo, ...)` (`main.rs:455`) + `router.rs:123` | `BUZZ_MESH_DEMO_ECHO=on` (`config.rs:514-518`) |
| NIP-PL push (matcher + delivery worker) | `main.rs:686-691` | `push_gateway_delivery_url.is_some()` — **on by default** (`config.rs:752-758`) |
| NIP-ER reminder scheduler | `main.rs:693-802` | always |
| NIP-43 relay membership enforcement + snapshot reconciliation | `main.rs:505-546` | `BUZZ_REQUIRE_RELAY_MEMBERSHIP` (default false) |
| NIP-OA owner-attested agent auth | `api/mod.rs:81` | `BUZZ_ALLOW_NIP_OA_AUTH` (default false, `config.rs:520-523`) |
| Ephemeral channel reaping | `main.rs:602-684` | always |
| Multi-node fan-out / cache invalidation / conn-control | `main.rs:350-367` (subscribers) + `main.rs:804-936` (consumers) | always |
| Community lifecycle durable revalidation | `main.rs:869-890` | always |
| Usage + storage metrics polling | `main.rs:987-1041` | always; per-community series gated by `BUZZ_USAGE_METRICS_PER_COMMUNITY`, storage sweep by `BUZZ_STORAGE_METRICS` (`storage_sweep.rs:69`) |
| Postgres FTS search | `main.rs:369-386` | always |
| Replica read routing behind a freshness fence | `main.rs:177-198` | `READ_DATABASE_URL` set + probe verifies the floor guard |
| Runtime conformance tracing | `state.rs:797` | production always binds `NoopTracer` (zero cost); only tests swap in `JsonlTracer` |
| UDS listener (service mesh sidecar) | `main.rs:1163-1206` | `BUZZ_UDS_PATH` set, unix only |
| Auto migrations | `main.rs:161-172` | `BUZZ_AUTO_MIGRATE` (default off) |
| Channel-event reconciliation (dev/CI) | `main.rs:547-590` | `BUZZ_RECONCILE_CHANNELS` present |
| Dev relay identity | `main.rs:396-408` | `BUZZ_RELAY_PRIVATE_KEY` unset **and** `!require_auth_token` |

#### 2. Background tasks — every `tokio::spawn`

**23 production spawn sites** in this file group (19 in `main.rs`, 3 in `state.rs`, 1 in `metrics.rs`). One additional spawn at `main.rs:1834` is inside `#[cfg(test)]`. `mesh_boot::boot_mesh` / `wire_consumers` spawn further tasks outside this group.

| # | Task | Spawn | Cadence / trigger | On error | Cancellable? |
|---|------|-------|-------------------|----------|--------------|
| 1 | Prometheus exporter HTTP server | `metrics.rs:146` | event-driven | none (future returns) | no |
| 2 | Audit worker | `state.rs:658` | event-driven on `audit_tx` | per-entry error → `buzz_audit_log_errors_total` + `error!` (`state.rs:1200-1206`); worker keeps going | **yes** — `AuditShutdownHandle` (`state.rs:1182-1196`), 5 s drain from `main.rs:1049` |
| 3 | Cache-invalidation publisher (fire-and-forget, per invalidation) | `state.rs:968` | per `invalidate_*` call | `warn!` and swallow; backstopped by ≤10 s TTL (`state.rs:960-963`) | no |
| 4 | Conn-control publisher (ban fan-out, per ban) | `state.rs:1043` | per ban | `warn!` and swallow; DB ban row is the durable backstop (`state.rs:1046-1050`) | no |
| 5 | Redis pub/sub event subscriber | `main.rs:354` | continuous | inside `run_subscriber` | no |
| 6 | Redis cache-invalidation subscriber | `main.rs:360` | continuous | inside | no |
| 7 | Redis conn-control subscriber | `main.rs:366` | continuous | inside | no |
| 8 | NIP-43 membership snapshot reconciler | `main.rs:522` | `BUZZ_NIP43_RECONCILE_INTERVAL_SECS`, default 60 s, `.max(1)`; first tick consumed immediately (`main.rs:524`) | `warn!`, loop continues | no |
| 9 | Channel-event reconciler (dev/CI) | `main.rs:551` | 24 attempts, 5 s apart (≈2 min) then exits | `warn!` per attempt | no |
| 10 | Workflow cron loop | `main.rs:599` | inside `WorkflowEngine::run` | inside | no |
| 11 | Ephemeral channel reaper | `main.rs:613` | `BUZZ_REAPER_INTERVAL_SECS`, default 60 s | tick error → `error!` + `continue`; per-channel side-effect errors logged individually (`main.rs:657/667`) | no |
| 12 | NIP-PL matcher | `main.rs:687` | inside `push_runtime::run_matcher` | inside | no |
| 13 | NIP-PL delivery worker | `main.rs:688` | inside `push_runtime::run_delivery_worker` | inside | no |
| 14 | NIP-ER reminder scheduler | `main.rs:709` | `SPROUT_REMINDER_SCHEDULER_INTERVAL_SECS`, default 10 s; batch `SPROUT_REMINDER_SCHEDULER_BATCH_LIMIT`, default 100 | query error → `error!` + `continue`; claim error → `warn!` + skip; publish error → release the claim (compare-and-clear on a per-attempt stamp), release failure → `warn!` **and the reminder stays claimed forever** (`main.rs:781-798`) | no |
| 15 | Multi-node fan-out consumer | `main.rs:823` | continuous `broadcast::recv` | `Lagged(n)` → `buzz_multinode_fanout_lag_total += n` + `warn!`; `Closed` → `error!` + **break (task dies, fan-out silently stops)** | no |
| 16 | Cache-invalidation consumer | `main.rs:853` | continuous | `Lagged` → counter + `warn!`; `Closed` → `error!` + break | no |
| 17 | Community lifecycle revalidator | `main.rs:888` | `BUZZ_COMMUNITY_REVALIDATE_INTERVAL_SECS`, default 30 s, `.clamp(1,300)` | per-community failure retains sockets, logged `warn!` (`state.rs:1082-1084`) | **yes** — `community_revalidator_cancel`, `main.rs:1045`; exits without waiting for the next tick (test `main.rs:1827-1849`) |
| 18 | Conn-control consumer | `main.rs:904` | continuous | `Lagged` → counter + `warn!`; `Closed` → `error!` + break | no |
| 19 | Pool metrics poller | `main.rs:950` | `BUZZ_POOL_METRICS_INTERVAL_SECS`, default 10 s, `.max(1)` | none — no fallible call in the loop | no |
| 20 | Usage metrics poller (+ storage sweep trigger) | `main.rs:1008` | `BUZZ_USAGE_METRICS_INTERVAL_SECS`, default 300 s, `.max(5)`; first tick jittered `rand % interval` | tick error → `error!` "skipping"; leader demotes | no |
| 21 | Health listener server | `main.rs:1122` | event-driven | `.ok()` — errors swallowed | no (deliberate, BR-RC-72) |
| 22 | Shutdown watchdog | `main.rs:1132` | one-shot on signal | ends in `std::process::exit(1)` after 30 s | no |
| 23 | UDS server (unix + `BUZZ_UDS_PATH`) | `main.rs:1181` | event-driven | `.ok()` | via `shutdown_tx` watch, then `abort()` (`main.rs:1197`) |
| — | Storage sweep body | `storage_sweep::maybe_spawn_sweep` called `main.rs:1456` | single-flight, `BUZZ_STORAGE_SWEEP_INTERVAL_SECS` | inside `storage_sweep` | no |

##### Failure-mode observations
- **Only 2 of 23 tasks are cancellable** (#2 audit, #17 revalidator). The other 21 are abandoned at process exit.
- Tasks #15, #16, #18 `break` out of their loop on `RecvError::Closed`. After that the pod keeps serving traffic but silently loses multi-node fan-out, cross-pod cache invalidation, or ban propagation respectively. There is no restart, no health-check coupling, and no gauge for "consumer alive" — only an `error!` line (`main.rs:838-841`, `main.rs:868-871`, `main.rs:929-932`).
- Task #14's release-failure branch (`main.rs:789-797`) leaves a reminder permanently claimed and never retried; the log explicitly says so.
- No task uses `JoinHandle` supervision or panic capture except #2 (`state.rs:1188` reports `Audit worker panicked`).

#### 3. Concurrency budget features

| Semaphore | Field | Bound | Default | Acquisition |
|-----------|-------|-------|---------|-------------|
| Connections | `conn_semaphore` (`state.rs:514`) | `max_connections` | 10,000 (`config.rs:449-452`) | `try_acquire_owned` — instant refusal |
| Handlers | `handler_semaphore` (`state.rs:516`) | `max_concurrent_handlers` | 1,024 (`config.rs:454-457`) | `try_acquire_owned` |
| Git subprocesses | `git_semaphore` (`state.rs:521`) | `git_max_concurrent_ops` | 20 (`config.rs:735-738`) | `api/git/transport.rs:322` |
| Media uploads | `media_upload_semaphore` (`state.rs:523`) | `media_max_concurrent_uploads` | 8 (`config.rs:663-668`) | `api/media.rs:119` |

`git_semaphore`'s doc is explicit that it bounds resources and is **not** writer serialization — that is the manifest-pointer CAS (`state.rs:517-520`).

#### 4. Observability features surfaced by this module

- **Framework HTTP metrics**: `http_requests_total`, `http_request_latency_ms` with `{code, caller, action}` (`metrics.rs:200-205`). `action` is the axum `MatchedPath` — never the raw URI, deliberately, to avoid unbounded cardinality from scanners (`metrics.rs:166-179`).
- **Per-metric histogram buckets**: 5 bucket families registered at `metrics.rs:76-141` — HTTP latency ms (11 buckets, `metrics.rs:37-39`), generic `_seconds` suffix (10, `metrics.rs:42`), git durations (13, `metrics.rs:45-47`), git bytes (9, `metrics.rs:50-60`), git pack counts (9, `metrics.rs:63`), fan-out recipients (9, `metrics.rs:66`).
- **Gauge idle-timeout**: `MetricKindMask::GAUGE` with the BR-RC-06 timeout (`metrics.rs:75-78`) so intentionally-stopped series disappear.
- **Pool gauges** (`main.rs:955-985`): `buzz_db_pool_{size,idle,active,max}`, `buzz_db_read_pool_{size,idle,active,max}`, `buzz_db_replica_fence_{open,lag_seconds}`, `buzz_redis_pool_{available,size,max,waiting}`. **The audit pool (`main.rs:325-329`) and the search pool (`main.rs:378-382`) are not instrumented at all.**
- **Usage gauges** (`main.rs:1481-1806`): fleet totals `buzz_total_{ws_connections,users_online_pod,subscriptions,users,channels,messages,relay_members,workflows,git_repos,active_users,active_channels}` + `buzz_communities_total`; per-community `buzz_community_{ws_connections,users_online_pod,subscriptions,users,channels,messages,relay_members,workflows,git_repos,active_users,active_channels}`.
- **Leader gauge**: `buzz_usage_poller_is_leader` (`main.rs:1032-1036`).
- **Lag counters**: `buzz_multinode_fanout_lag_total` (`main.rs:836`), `buzz_cache_invalidation_lag_total` (`main.rs:866`), `buzz_conn_control_lag_total` (`main.rs:927`).
- **Audit metrics**: `buzz_audit_enabled` gauge (`main.rs:139`), `buzz_audit_log_errors_total`, `buzz_audit_log_seconds` (`state.rs:1201-1205`).
- **Cache-hit metrics**: `buzz_membership_cache_{hits,misses}_total` (`state.rs:833/836`), `buzz_accessible_channels_cache_{hits,misses}_total` (`state.rs:1095/1098`). The other five caches have **no** hit/miss instrumentation.
- **Backpressure**: `buzz_ws_backpressure_disconnects_total` (`state.rs:466`).
- **Tracing**: JSON-flattened stdout always (`main.rs:109`); OTLP gRPC layer attached only when `OTEL_EXPORTER_OTLP_ENDPOINT` is set (`main.rs:101-107`, `telemetry.rs:79-90`), tracer name `"buzz-relay"` (`main.rs:104`).

#### 5. Feature flags / gates summary

Cargo features (`Cargo.toml:80-81`): exactly one — `dev = ["buzz-auth/dev"]`. No `#[cfg(feature = ...)]` in this file group; the flag only forwards dev key derivation into `buzz-auth`.

Compile-time cfg used: `#[cfg(unix)]` / `#[cfg(not(unix))]` for UDS (`main.rs:1162`, `main.rs:1207`) and the SIGTERM handler (`main.rs:1227`, `main.rs:1236`); `#[cfg(unix)]` on one config test (`config.rs:1329`).

Runtime kill switches that produce **byte-identical off behaviour** (explicit design intent, verified):
- `BUZZ_MESH` off ⇒ no UDP bind, no Redis registry write, no spawn (`config.rs:492-497`, `main.rs:437-441`).
- `BUZZ_MESH_DEMO_ECHO` off ⇒ inbound reliable streams accepted, logged, closed (`config.rs:139-143`).
- `EmissionScope::Off` ⇒ no per-community series; totals unchanged (`main.rs:76-78`).
- Storage sweep disabled ⇒ zero storage-family gauges including health (`main.rs:1436-1453`).
- Audit disabled ⇒ `audit_tx = None`, worker parked on cancellation (`state.rs:659-663`, `state.rs:763`).
