## Module: buzz-relay — core & bootstrap (`crates/buzz-relay/src`)
### Aspect: Business Rules

All rules below were read out of the code, not from docs. Where a doc comment and the code disagree, the delta is stated explicitly.

---

#### A. Startup ordering and preconditions (`main.rs:83-1060`)

`main()` is a single 977-line function. The order is load-bearing in several places (the code says so at `main.rs:176-183`, `main.rs:220-232`, `main.rs:265-269`, `main.rs:592-593`).

| # | Rule | Cite |
|---|------|------|
| **BR-RC-01** | The ring rustls `CryptoProvider` MUST be installed before any TLS use (`rediss://`, `wss://`, S3-over-TLS). Failure is a hard `expect` panic. | `main.rs:88-90` |
| **BR-RC-02** | The tracing subscriber is installed **after** `telemetry::try_init_tracer` returns but **before** any log is emitted; an exporter build failure is therefore logged only at `main.rs:118-120`, never inside `telemetry.rs` (documented `telemetry.rs:75-78`). | `main.rs:98-120` |
| **BR-RC-03** | Log filtering is `EnvFilter::from_default_env()` (i.e. `RUST_LOG`) **plus a forced `buzz_relay=info` directive**, so `buzz_relay` can never be filtered below `info` by env alone. | `main.rs:111` |
| **BR-RC-04** | Config load failure is fatal: `Config::from_env()` errors abort startup. | `main.rs:122-127` |
| **BR-RC-05** | The Prometheus exporter is installed **before** the DB connects, so a DB failure is still observable via `/metrics`. `metrics::install` panics if a recorder already exists or the port is taken. | `main.rs:138`, `metrics.rs:143-146` |
| **BR-RC-06** | Gauge idle-timeout MUST be at least 3× the usage-poll interval; the configured value is raised to `max(configured_or_900, interval*3)`. | `main.rs:1273-1277`, called `main.rs:136-138` |
| **BR-RC-07** | Usage-poll interval has a hard floor of 5 s (`.max(5)`) to prevent a busy loop; default 300 s. | `main.rs:1257-1262` |
| **BR-RC-08** | Postgres connect failure is fatal. Read-replica presence is logged but never required. | `main.rs:151-160` |
| **BR-RC-09** | Migrations run **only** when `BUZZ_AUTO_MIGRATE` ∈ {`true`,`1`,`yes`,`on`} (case-insensitive, trimmed). Opt-in, not opt-out. Migration failure is fatal. | `main.rs:25-33`, `main.rs:161-172` |
| **BR-RC-10** | Partition pre-creation (`ensure_future_partitions(3)`) failure is logged at `error` and **non-fatal** — the relay serves on with no future partitions. | `main.rs:173-176` |
| **BR-RC-11** | The replica freshness fence probe MUST start after the migration decision so the commit-time floor guard is verified against the *live* schema. Verification failure is loud but non-fatal: the fence stays closed and every cursor page routes to the writer. | `main.rs:177-198` |
| **BR-RC-12** | `require_relay_membership=true` requires a valid `RELAY_OWNER_PUBKEY`; otherwise abort. `config.rs:526-546` already warn-and-drops invalid values, so this catches the resulting `None`. | `main.rs:199-212` |
| **BR-RC-13** | `require_relay_membership=true` requires `BUZZ_RELAY_PRIVATE_KEY`; otherwise abort. Checked **before any DB mutation** (documented `main.rs:214-216`). | `main.rs:213-220` |
| **BR-RC-14** | The deployment's own community is seeded from `relay_url_authority(config.relay_url)` — the *same* normalization live request resolution uses — so the bootstrapped owner lands in the community requests actually resolve to. | `main.rs:238-240`, `tenant.rs:139`, `tenant.rs:80` |
| **BR-RC-15** | An empty derived authority is fatal **only when** `require_relay_membership=true`; otherwise it logs `error` and skips backfill/bootstrap. | `main.rs:241-256` |
| **BR-RC-16** | `ensure_configured_community` failure is fatal iff membership is enforced, else non-fatal error log. | `main.rs:247-263` |
| **BR-RC-17** | `pubkey_allowlist` → `relay_members` backfill MUST run **before** owner bootstrap, otherwise enabling membership locks every existing user out (documented `main.rs:265-269`). Failure fatal iff membership enforced. | `main.rs:270-289` |
| **BR-RC-18** | Owner bootstrap runs only when both a community and an owner pubkey exist. Failure fatal iff membership enforced. | `main.rs:290-311` |
| **BR-RC-19** | NIP-33 `d_tag` backfill runs unconditionally every startup and is always non-fatal (`error!` only). Idempotent. | `main.rs:312-320` |
| **BR-RC-20** | The audit service opens a **separate** Postgres pool (max 5, min 1) only when `audit_enabled`. Connect failure is fatal. | `main.rs:322-334` |
| **BR-RC-21** | Redis pool creation and `PubSubManager::new` failures are both fatal. Pool size = `config.redis_pool_size` (default 16; deadpool's own default of `CPU*2` is explicitly rejected as too small, `config.rs:61-65`). | `main.rs:335-348` |
| **BR-RC-22** | Search opens a **third** Postgres pool, preferring `read_database_url` when set (search is lag-tolerant). Connect failure is fatal. No pool sizing knobs are set — sqlx defaults apply. | `main.rs:369-386` |
| **BR-RC-23** | Relay signing keypair resolution, in strict priority: (a) `BUZZ_RELAY_PRIVATE_KEY` → parse or abort; (b) else if `!require_auth_token` → **hard-coded dev key `0x…01`** with a `warn`; (c) else `panic!`. | `main.rs:392-414` |
| **BR-RC-24** | Media config is validated (`config.media.validate()`) and storage constructed before `AppState`; both failures fatal. | `main.rs:415-422` |
| **BR-RC-25** | `AppState::new` spawns the audit worker eagerly even when audit is disabled; the disabled worker just awaits cancellation and returns (`state.rs:659-663`). `audit_tx` is `None` when disabled, so nothing can enqueue. | `state.rs:653-690`, `state.rs:763` |
| **BR-RC-26** | Mesh boot (`BUZZ_MESH` seam) returns `None` when off ⇒ **nothing bound, published or spawned**. When on, a bind/Redis failure is fatal (`?` at `main.rs:451`). Consumers are wired **before** the handle is published to `AppState.mesh`, so no peer can route traffic to an unwired dispatcher. | `main.rs:437-464` |
| **BR-RC-27** | The git object-store A3 conformance probe runs by default (opt-*out* via `BUZZ_GIT_CONFORMANCE_PROBE=false`) and its failure is **fatal** — a backend that cannot do linearizable conditional writes invalidates the manifest-pointer protocol. | `main.rs:466-503` |
| **BR-RC-28** | NIP-43 membership-snapshot reconciliation runs at startup **before the listener binds** and then periodically, only when `require_relay_membership=true`. Startup failure is `warn`-only. | `main.rs:505-546` |
| **BR-RC-29** | Channel-event reconciliation runs only when `BUZZ_RECONCILE_CHANNELS` is *present* (any value, including empty — `is_ok()`), is community-scoped via `bind_deployment_community`, and retries 24× at 5 s (≈2 min). An unmapped relay host skips it with a `warn`. | `main.rs:547-590` |
| **BR-RC-30** | The workflow action sink MUST be wired after `AppState` (which owns `sub_registry`/`conn_manager`) and **before** the cron loop starts. | `main.rs:592-600` |
| **BR-RC-31** | Both routers are built (`main.rs:939-940`) **before** the pool-metrics and usage-metrics pollers are spawned, and `serve()` is the last call. | `main.rs:939-1043` |

#### B. Tenancy resolution rules (`tenant.rs`)

| # | Rule | Cite |
|---|------|------|
| **BR-RC-32** | Row zero: `req.community = resolve_host(connection.host)`, bound at connection establishment, **before the WebSocket upgrade**, so no frame is ever read on an unbound connection. | `tenant.rs:1-9`, `router.rs:279-300` |
| **BR-RC-33** | The host is normalized exactly once, by `buzz_core::tenant::normalize_host`, before lookup — so `RELAY.EXAMPLE`, `relay.example.`, and `relay.example:443` all bind to one community (`tenant.rs:203-219`). Implementors of `HostResolver` MUST assume pre-normalized input (`tenant.rs:22-26`). | `tenant.rs:80` |
| **BR-RC-34** | An empty or whitespace-only host fails closed **before** the resolver is consulted, because the schema does not forbid a `host = ''` row in `communities`. The rejection reuses `UnmappedHost`, byte-identical to any other unmapped host, so a caller cannot probe for an empty-host row. | `tenant.rs:81-88` |
| **BR-RC-35** | Both "host maps to nothing" (`Ok(None)`) and "lookup failed" (`Err`) reject. There is deliberately **no** default/fallback community path. | `tenant.rs:89-95`, `tenant.rs:149-151` |
| **BR-RC-36** | The rejection surfaced to the caller is a single generic `404 "relay: no community is configured for this host"` that never echoes the host and never distinguishes unmapped from lookup error. | `router.rs:288-299` |
| **BR-RC-37** | The returned `TenantContext` carries the **normalized** host so downstream NIP-05 labelling, audit labelling, and the NIP-98 `u`-host check all see one canonical form. | `tenant.rs:74-77`, `tenant.rs:91` |
| **BR-RC-38** | Server-internal paths with no inbound `Host` (git Smart-HTTP, the localhost pre-receive hook, the workflow sink, startup tasks) resolve `config.relay_url`'s authority through the *same* fail-closed `bind_community`. Not a fallback: an unmapped `relay_url` errors identically. | `tenant.rs:120-133` |
| **BR-RC-39** | Background sweeps that cross communities MUST build a per-row `TenantContext` from the DB `RETURNING`, never a default tenant. Enforced by construction in the ephemeral reaper (`main.rs:637-644`) and the reminder scheduler (`main.rs:741-746`). | `main.rs:632-644`, `main.rs:707-746` |
| **BR-RC-40** | The relay's *own* deployment community is ensured with the same authority derivation used by live request resolution, so bootstrap and requests cannot diverge. | `main.rs:220-236` |
| **BR-RC-41** | Local-echo dedup keys include the community, because the same Nostr event id can legitimately exist in two communities; keying on the bare id would let a publish in A suppress a distinct event in B. | `state.rs:530-540`, `state.rs:821-824` |
| **BR-RC-42** | Cross-pod cache invalidation applies **only** the matching tenant-local drop; a mutation in A must not flush B's derived state. | `main.rs:856-867`, `state.rs:980-998` |
| **BR-RC-43** | Ban-driven disconnects are fenced to the banning community: the same pubkey live in A and B loses only A's sockets. | `state.rs:296-308`, `state.rs:328-334`, tests `state.rs:1734-1780` |
| **BR-RC-44** | Graceful-shutdown drain is deliberately **not** tenant-fenced — `drain_all` signals every connection across all communities. | `state.rs:336-366`, test `state.rs:1782-1836` |
| **BR-RC-45** | Community-scoped cache invalidation uses moka predicate invalidation; if invalidation closures are ever unavailable the code **over-invalidates** (full flush) rather than serving stale access state. | `state.rs:881-895`, `state.rs:928-975` |

#### C. Admission rules (`admission.rs`, `router.rs`, `state.rs`)

| # | Rule | Cite |
|---|------|------|
| **BR-RC-46** | Shared rate-limit admission is **fail-closed**: a limiter error (Redis unavailable) maps to `AdmissionError::Unavailable` and denies, after a `warn`. Confirmed. | `admission.rs:29-33`, test `admission.rs:141-158` |
| **BR-RC-47** | A denied counter returns `Exceeded { reset_in_secs }` taken from the limiter result, so callers can emit a retry hint. | `admission.rs:25-28` |
| **BR-RC-48** | WebSocket admission converts a per-second limit into a fixed 5-second window with 5× the budget, preserving the average rate while allowing desktop's burst of simultaneous subscriptions. Saturating multiply. | `admission.rs:6-10`, `admission.rs:39-44`, tests `:107-115` |
| **BR-RC-49** | Total concurrent connections are bounded by `conn_semaphore` = `config.max_connections` (default 10,000), acquired with `try_acquire_owned` (immediate refusal, no queuing) by both ordinary WS (`connection.rs:149`) and huddle audio (`audio/handler.rs:113`). | `state.rs:729`, `config.rs:449-452` |
| **BR-RC-50** | Concurrent message handlers are bounded by `handler_semaphore` = `config.max_concurrent_handlers` (default 1,024), also `try_acquire_owned`. | `state.rs:730`, `connection.rs:513/541/563` |
| **BR-RC-51** | A new WebSocket upgrade is refused with `503 "relay restarting"` once `shutting_down` is set, because readiness-503 only stops K8s routing and direct/in-flight upgrades still arrive during the pre-drain grace window. | `router.rs:305-313` |
| **BR-RC-52** | Inbound frame size is capped **at the parser**, before tungstenite assembles the message, by setting both `max_message_size` and `max_frame_size`; the application-level check in `recv_loop` remains as defence in depth. | `router.rs:324-332`, test `router.rs:504-518` |
| **BR-RC-53** | Slow-client backpressure: `grace_limit` consecutive full send buffers cancels the connection; any successful send resets the counter to 0. The counter is shared (`Arc<AtomicU8>`) between direct sends and fan-out, so a mixed pattern still trips the limit. Default grace 15 (`config.rs:470-473`). | `state.rs:451-478`, tests `state.rs:1447-1533` |
| **BR-RC-54** | Registration and drain are race-free by an insert-then-check / store-then-iterate pair: a connection that registers after the drain snapshot observes the sticky `draining` flag and self-signals its own 1012 close + cancel. The flag is one-way. | `state.rs:227-235`, `state.rs:353-356`, test `state.rs:1875-1930` |
| **BR-RC-55** | Long-lived-socket community admission ordering: register → durably revalidate → run. Archive-before-query is caught by the query; archive-after-registration is caught by the token. A guard deregisters on **every** exit path. | `state.rs:132-158`, tests `state.rs:1602-1683` |
| **BR-RC-56** | The periodic community revalidator only inspects communities that have **live local sockets** (`bound_communities`), never a global DB scan; a per-community lookup failure retains that community's sockets until the next tick and does not abort the sweep. | `state.rs:160-181`, `state.rs:1076-1087`, test `state.rs:1685-1730` |
| **BR-RC-57** | `check_ip_connection` (per-IP connection flood limiting) exists in `buzz-auth` (`crates/buzz-auth/src/rate_limit.rs:188`) and is implemented by `RedisRateLimiter` (`crates/buzz-pubsub/src/rate_limiter.rs:112`), but has **zero production callers** — the only other reference is the test stub at `admission.rs:85`. There is no IP-level admission rule in force. | verified by workspace grep |

#### D. NIP-11 advertisement rules (`nip11.rs`)

| # | Rule | Cite |
|---|------|------|
| **BR-RC-58** | `relay_self` (JSON `self`) is advertised iff the relay has a **stable** signing key (`relay_private_key.is_some()`). Ephemeral keys are excluded because they change on restart, leaving previously-signed NIP-29 group metadata unverifiable. | `nip11.rs:288-305` |
| **BR-RC-59** | NIP-43 is advertised iff stable key **AND** `require_relay_membership`. It is deliberately absent from the static `SUPPORTED_NIPS`; the desktop pairing probe keys off it, and advertising it on an open relay misroutes pairing peers to a non-existent `/pair` sidecar (`nip11.rs:475-484`). | `nip11.rs:17-21`, `nip11.rs:148-151`, `nip11.rs:304` |
| **BR-RC-60** | Advertising NIP-43 without `self` is a programmer error, caught by `debug_assert!` in debug/test builds only — a release build would ship the inconsistent document. | `nip11.rs:143-146`, test `nip11.rs:514-521` |
| **BR-RC-61** | `RelayInfo::build` MUST accept only static/scalar inputs. This is mechanically enforced: `_RELAY_INFO_BUILD_STATIC_INPUT_FENCE` (`nip11.rs:329-335`) is a `const` function pointer pinned to `build`'s exact signature — adding a `&Db`, `&AppState`, search, or audit handle breaks the build. | `nip11.rs:307-335` |
| **BR-RC-62** | NIP-11 is served **before** tenant binding and fails **open**: an unmapped host still gets the document with host-scoped fields (`icon`) simply absent, so the doc cannot leak which hosts are mapped. | `router.rs:275-286`, `nip11.rs:271-286` |
| **BR-RC-63** | The push descriptor is emitted iff `push_gateway_delivery_url.is_some()` **and** the request host binds to a community; the `origin` scheme is derived from whether `relay_url` starts with `wss://`. | `nip11.rs:183-233`, `nip11.rs:244-259`, test `nip11.rs:340-356` |
| **BR-RC-64** | `supported_nips` MUST be sorted — asserted by test (`nip11.rs:462-471`). |
| **BR-RC-65** | An empty-string icon is treated the same as absent and omitted from the JSON, so the document matches pre-icon output byte-for-byte. | `nip11.rs:154`, `nip11.rs:31-32`, test `nip11.rs:412-437` |
| **BR-RC-66** (delta) | `auth_required: true`, `restricted_writes: true`, and `due_delivery_mode: "push"` are **unconditional literals** (`nip11.rs:109/111/112`) — advertised even on a fully open relay and even when push delivery is disabled and no push worker was spawned (`main.rs:686-691`). |
| **BR-RC-67** (delta) | `max_limit: 10_000` (`nip11.rs:106`) is not enforced: `crates/buzz-db/src/event.rs:331` clamps a filter limit to `max_limit.unwrap_or(1000)`. |

#### E. Shutdown / graceful-drain rules (`main.rs:1113-1240`)

| # | Rule | Cite |
|---|------|------|
| **BR-RC-68** | The signal set is SIGTERM **or** Ctrl+C on unix; Ctrl+C only elsewhere. `signal(SIGTERM)` installation failure is an `expect` panic. | `main.rs:1226-1240` |
| **BR-RC-69** | Shutdown sequence: set `shutting_down=true` → readiness returns 503 → **sleep 5 s** so K8s stops routing → signal the listeners to drain → send a `1012 Service Restart` close frame to every live WebSocket → **sleep 30 s** → `std::process::exit(1)`. | `main.rs:1130-1154` |
| **BR-RC-70** | Every live WebSocket is explicitly closed with 1012 rather than left to ride the dying pod; without it a client would learn about the restart only from a TCP reset (or up to 60 s of stall-watchdog silence). | `state.rs:336-350`, `main.rs:1138-1149` |
| **BR-RC-71** | Close frames are delivered "queue on ctrl, then cancel": the send loop drains queued control frames before its `biased` cancel branch closes the socket. Best-effort — a full 1-slot control buffer still gets the cancel, just without the frame. | `state.rs:344-350`, `state.rs:357-362`, test `state.rs:1838-1873` |
| **BR-RC-72** | The health listener is spawned **without** graceful shutdown so `/_readiness` keeps answering 503 throughout the drain. | `main.rs:1120-1124` |
| **BR-RC-73** | The community revalidator is cancelled only *after* `serve()` returns, and only that loop — via its own dedicated `CancellationToken`, not the global flag. | `main.rs:1043-1045`, `state.rs:510` |
| **BR-RC-74** | The audit worker drain is signalled by a `CancellationToken` (not by dropping `Arc<AppState>`) so it works while background tasks still hold state clones; timeout 5 s. On cancel it closes the receiver, then drains everything already buffered. | `state.rs:664-681`, `state.rs:1182-1196`, `main.rs:1047-1051` |
| **BR-RC-75** | OTEL spans are flushed last, after the audit drain; a shutdown error is `warn`-only. | `main.rs:1053-1058` |
| **BR-RC-76** (delta) | The 30 s watchdog `std::process::exit(1)` (`main.rs:1153`) bypasses **both** BR-RC-74 and BR-RC-75 and exits non-zero. A shutdown that overruns the drain window therefore loses buffered audit entries and pending spans, and K8s records a failed termination. |
| **BR-RC-77** | On the UDS path, the UDS server task is `abort()`ed once the TCP server has drained; the function returns early and never falls through to the TCP-only branch. | `main.rs:1197-1205` |
| **BR-RC-78** | `BUZZ_UDS_PATH` pointing at an existing **non-socket** file is a fatal error; an existing socket is unlinked and rebound. | `main.rs:1163-1176` |
| **BR-RC-79** | On non-unix, a set `BUZZ_UDS_PATH` is a `warn` and silently ignored. | `main.rs:1208-1211` |

#### F. Metrics-emission rules (`main.rs:1279-1806`)

| # | Rule | Cite |
|---|------|------|
| **BR-RC-80** | The first usage tick is jittered by `rand::random::<u64>() % interval_secs` using true per-process randomness — PID-derived seeds are explicitly rejected because the relay is typically PID 1 in every pod. | `main.rs:1009-1017` |
| **BR-RC-81** | The usage interval uses `MissedTickBehavior::Skip` so a slow tick never schedules a catch-up burst. Same for the periodic-until-cancelled helper. | `main.rs:1021`, `main.rs:1089` |
| **BR-RC-82** | Exactly one pod owns the DB-derived snapshot, via a Postgres advisory lock (`USAGE_METRICS_LOCK_KEY`). Leadership is re-checked each tick with `is_live()`; a failed liveness check demotes and **skips** re-acquisition on that same tick (`!demoted` guard). | `main.rs:1417-1433` |
| **BR-RC-83** | All DB queries for a tick complete **before any gauge is set** ("collect then publish", C4). Any query error aborts the whole tick so a mixed fresh/stale snapshot is impossible. | `main.rs:1481-1516` |
| **BR-RC-84** | In-memory gauges are emitted by **every** pod (dashboard sums); DB gauges only by the leader (dashboard maxes). Documented `main.rs:989-993`. |
| **BR-RC-85** | Per-community series are zero-filled from the host map, not from query rows, so a community that drops to zero reports 0 instead of retaining its last non-zero value. Applied to users, channels, messages, relay members, workflows, git repos, active users, active channels. | `main.rs:1524-1806` |
| **BR-RC-86** | An unrecognised `channel_type` / `role` / `workflow status` row is **skipped with a `warn`**, not defaulted. | `main.rs:1573-1580`, `main.rs:1633-1640`, `main.rs:1676-1683` |
| **BR-RC-87** | A label key that disappears between ticks receives one final `0.0` before being forgotten; the key stores the resolved **host label** so a renamed/removed community can still be zeroed. | `main.rs:1316-1382`, test `main.rs:1861-1875` |
| **BR-RC-88** | Legacy lifecycle gauges (`buzz_ws_connections_active`, `buzz_subscriptions_active`) are kept "recent" for the exporter's idle-timeout policy by `increment(0.0)` rather than a snapshot `set()`, so a poller refresh cannot race the lifecycle inc/dec. | `main.rs:1306-1317`, tests `main.rs:1877-1927` |
| **BR-RC-89** | If the host-map query fails, the leader demotes, in-memory gauges are still emitted with `None` host map (totals only), and the tick returns `Err`. | `main.rs:1390-1403` |
| **BR-RC-90** | Fleet-wide `buzz_total_*` gauges always emit regardless of `EmissionScope`; only per-community series are gated. | `main.rs:44-47`, `main.rs:1506-1516` |
| **BR-RC-91** | The storage sweep has its own always-on config and a hard kill switch independent of `EmissionScope`; disabled means **no** storage-family gauge is touched, including health gauges, so a relay without `s3:ListBucket` can turn the feature off cleanly. | `main.rs:1436-1453` |
| **BR-RC-92** | An unknown `BUZZ_USAGE_METRICS_PER_COMMUNITY` value logs a `warn` and defaults to `all` (fail-open on cost, not on correctness). | `main.rs:57-73` |
| **BR-RC-93** | The pool-metrics interval is floored at 1 s because `tokio::time::interval` panics on `Duration::ZERO`. | `main.rs:945-949` |
| **BR-RC-94** | Replica-fence observability is reported as `buzz_db_replica_fence_open` 1/0 plus `..._lag_seconds`; a closed or stale fence reports `open=0` and leaves the lag gauge untouched. | `main.rs:967-978` |

#### G. Config validation rules (`config.rs`)

| # | Rule | Cite |
|---|------|------|
| **BR-RC-95** | `RELAY_OWNER_PUBKEY` must be exactly 64 hex chars; an invalid value is **warn-and-ignore** (lowercased, trimmed first). | `config.rs:526-546` |
| **BR-RC-96** | `RELAY_OPERATOR_PUBKEYS` entries must each be 64 hex chars; an invalid entry is a **hard config error** (asymmetric to BR-RC-95 by design — silently dropping an operator pubkey would silently disable provisioning). Duplicates are deduped, order preserved. | `config.rs:548-576` |
| **BR-RC-97** | A non-empty `RELAY_OPERATOR_PUBKEYS` **requires** `RELAY_OPERATOR_API_ORIGIN`; the origin must be `http`/`https` with a host and **no** credentials, path, query, or fragment. | `config.rs:577-582`, `config.rs:319-338` |
| **BR-RC-98** | Every `BUZZ_RATE_LIMIT_*` override must parse as a **positive** integer; zero and junk are hard errors (not silent fallbacks). | `config.rs:270-282`, tests `config.rs:1107-1121` |
| **BR-RC-99** | `BUZZ_PUSH_GATEWAY_DELIVERY_URL` must be an **exact** `https://…/v1/deliveries/apns` URL — no `http`, no trailing slash, no query, no fragment, no credentials. Explicit empty string disables push; **unset falls back to the hard-coded `https://push.buzz.xyz/v1/deliveries/apns`**, i.e. push is on by default. | `config.rs:340-360`, `config.rs:332`, `config.rs:752-758` |
| **BR-RC-100** | `BUZZ_PUSH_GATEWAY_TIMEOUT_MS` must be in `100..=10000`; out-of-range is a hard error, not a silent default. Unset ⇒ 2000 ms. | `config.rs:759-773` |
| **BR-RC-101** | `BUZZ_PUSH_EXECUTOR_KEY_ID` must be 1..=64 bytes. | `config.rs:745-751` |
| **BR-RC-102** | Policy Markdown documents are capped at 256 KiB each; `join_policy` is `None` unless at least one document or the age attestation is configured. Its `version` is `SHA-256(terms ‖ 0x00 ‖ privacy ‖ 0x00 ‖ age_flag)` — a content-derived id binding receipts to the exact revision. | `config.rs:775-810` |
| **BR-RC-103** | `BUZZ_ADMIN_HOST` must be an exact authority — rejecting any `/`, `\`, or `@`. `BUZZ_ADMIN_WEB_DIR` and `BUZZ_WEB_DIR` must each contain `index.html` or startup fails. | `config.rs:813-838`, `config.rs:850-859` |
| **BR-RC-104** | `BUZZ_GIT_HOOK_HMAC_SECRET`, when explicitly set, must be ≥32 chars. When unset a fresh random 32-byte hex secret is generated per process — so in a multi-pod deployment the hook secret differs per pod. | `config.rs:739-744`, `config.rs:862-871` |
| **BR-RC-105** | `BUZZ_PAIRING_RELAY_URL` must parse as `ws://` or `wss://` with a host; `https://` is rejected. | `config.rs:430-447`, test `config.rs:1287-1305` |
| **BR-RC-106** | `git_repo_path` and `git_pack_cache_path` are **created** (`create_dir_all`) at config load; an uncreatable path is a config error. `git_pack_cache_path` defaults to `<git_repo_path>/.pack-cache`. | `config.rs:377-398`, `config.rs:704-712` |
| **BR-RC-107** | Derived defaults chain: `git_max_repo_bytes = git_max_pack_bytes * 2`, `git_pack_cache_max_bytes = git_max_repo_bytes * 5` (saturating), so raising the pack limit silently raises both. | `config.rs:717-725` |
| **BR-RC-108** | `media_max_concurrent_uploads_per_pubkey` is clamped to `min(configured, media_max_concurrent_uploads)`. | `config.rs:669-675` |
| **BR-RC-109** | Boolean parsing is **inconsistent by design across three helpers**: `parse_bool` accepts `true/1/on` vs `false/0/off/""` and *errors* on anything else (`config.rs:363-377`); most flags use a bare `v == "true" \|\| v == "1"` and silently treat anything else as false (`config.rs:475`, `:479`, `:483`, `:520`, `:651`, `:848`); `huddle_audio_available` inverts (`!(v=="false"\|\|v=="0")`, `config.rs:489`); `BUZZ_MESH`/`BUZZ_MESH_DEMO_ECHO` accept `on` case-insensitively plus `true`/`1` (`config.rs:498`, `:516`); `BUZZ_REQUIRE_MEDIA_GET_AUTH` additionally accepts `yes`/`on` (`config.rs:682-689`); `BUZZ_AUTO_MIGRATE` accepts `true/1/yes/on` trimmed and case-insensitive (`main.rs:25-33`); `BUZZ_GIT_CONFORMANCE_PROBE` is `v != "false"` (`main.rs:469-471`); `BUZZ_RECONCILE_CHANNELS` is mere presence (`main.rs:549`). **Eight distinct boolean grammars.** |
| **BR-RC-110** | `!require_auth_token` emits a startup `warn` that REST bypasses token auth. `BUZZ_EPHEMERAL_TTL_OVERRIDE` emits a `warn` that it overrides client TTLs. | `config.rs:590-594`, `config.rs:696-702` |

#### H. CORS rules (`router.rs:409-432`)

| # | Rule | Cite |
|---|------|------|
| **BR-RC-111** | Empty `BUZZ_CORS_ORIGINS` ⇒ `CorsLayer::permissive()` — the **default**, since the var is unset by default (`config.rs:595-600`). | `router.rs:410-412` |
| **BR-RC-112** | If origins are configured but **none** parse as a `HeaderValue`, the layer refuses to fall back to permissive and returns a bare `CorsLayer::new()` (no CORS headers) after an `error!`. | `router.rs:419-426` |
| **BR-RC-113** | With valid origins the layer allows those origins with `Any` methods and `Any` headers — no credentials allowance, no method/header restriction. | `router.rs:428-431` |
