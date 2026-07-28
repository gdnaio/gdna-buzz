## Module: buzz-relay — core & bootstrap (`crates/buzz-relay/src`)
### Aspect: Debt

Severity scale: **S1** = correctness/security bug reachable today; **S2** = latent bug, operational hazard, or misleading documentation that would cause a wrong decision; **S3** = maintainability / consistency.

---

#### S1 — Live defects

##### D-01 (S1) Workflow `add_reaction` POSTs to a route that does not exist
`crates/buzz-workflow/src/executor.rs:889` constructs `POST {BUZZ_RELAY_BASE_URL}/api/messages/{message_id}/reactions`. No such route is registered in `router.rs:37-131`, `api/git/transport.rs:1760-1762`, `api/git/mod.rs:62`, or `api/admin/mod.rs:30-37`. The request falls through to the SPA fallback's 404 (`router.rs:178`) or a bare axum 404, so every workflow that adds a reaction fails silently at the HTTP layer. The only two mentions of `api/messages` in the workspace are the doc comment at `crates/buzz-workflow/src/executor.rs:883` and the URL at `:889`. Confirmed.

Also worth noting: this whole path violates the repo's own "prefer Nostr events over new HTTP endpoints" rule (AGENTS.md § Key Patterns) — the fix is a kind, not a route.

##### D-02 (S1) Three rate-limit env vars are parsed, validated, stored, and never read
`config.rs:303-306`, `:307-310`, `:311-314` parse `BUZZ_RATE_LIMIT_AGENT_STANDARD_API_CALLS_PER_MIN`, `..._AGENT_ELEVATED_MESSAGES_PER_MIN`, `..._AGENT_PLATFORM_MESSAGES_PER_MIN` and hard-error on invalid values, giving an operator every signal that they take effect. Verified across `crates/**` and `desktop/src-tauri/**`: the field names appear only in their declarations (`crates/buzz-auth/src/rate_limit.rs:101,104,107`) and default assignments (`:139,140,141`). Zero readers. Two of the three name enforcement tiers ("elevated", "platform") that have no code path at all. The four live siblings are consumed at `api/bridge.rs:29`, `connection.rs:614`, `connection.rs:634`, `connection.rs:636`.

##### D-03 (S1) Two background intervals have no lower bound → busy-loop on `0`
`BUZZ_REAPER_INTERVAL_SECS` (`main.rs:609-612`) and `SPROUT_REMINDER_SCHEDULER_INTERVAL_SECS` (`main.rs:701-704`) parse straight to `u64` with no floor. Set to `0`, `tokio::time::sleep(Duration::ZERO)` turns each task into a hot loop issuing `reap_expired_ephemeral_channels` / `query_due_reminders` continuously against Postgres. The two neighbouring pollers were explicitly floored for exactly this hazard: `main.rs:949` (`.max(1)` with the comment "tokio::time::interval panics on Duration::ZERO") and `main.rs:1261` (`.max(5)` "with a floor that prevents a busy loop"). `BUZZ_NIP43_RECONCILE_INTERVAL_SECS` has `.max(1)` (`main.rs:521`) and `BUZZ_COMMUNITY_REVALIDATE_INTERVAL_SECS` has `.clamp(1,300)` (`main.rs:886`). Two of six lack the guard.

##### D-04 (S1) Redis broadcast consumers die permanently on `RecvError::Closed`
`main.rs:838-841` (multi-node fan-out), `main.rs:868-871` (cache invalidation), `main.rs:929-932` (conn-control) all `break` out of their loop on `Closed`, log one `error!`, and end the task. The pod keeps serving traffic and keeps reporting `/_readiness: ready` (readiness only pings Postgres + Redis, `router.rs:352-373`) while:
- events from other pods are no longer fanned out to local subscribers,
- membership/visibility changes on other pods no longer drop local caches,
- bans recorded on other pods no longer close local sockets.

No restart, no supervision, no gauge for "consumer alive". The cache case degrades to a ≤10 s TTL wait (tolerable, `state.rs:960-963`); the fan-out and ban cases degrade to silent cross-pod incoherence.

##### D-05 (S1) Reminder scheduler can permanently orphan a reminder
`main.rs:781-798`: after winning the claim, if the Redis publish fails **and** the compensating `release_due_reminder` also fails, the code logs `warn!("… reminder stays claimed, will not retry")` and moves on. The `delivered_at` sentinel stays set, the partial index excludes the row, and no later tick or pod will ever pick it up. The comment states the outcome plainly; there is no dead-letter path, no metric, and no reconciliation sweep for stuck claims.

##### D-06 (S1) `expect` in the fan-out hot path
`state.rs:444-447`: `send_to_text_bytes` does `WsUtf8Bytes::try_from(...).expect("relay fan-out frames are serialized UTF-8 JSON")`. The doc says "Callers must only pass valid UTF-8 bytes" — an unenforceable contract on a `pub fn` taking `Arc<Bytes>`. A single non-UTF-8 payload panics the fan-out task. A `Result` or a `debug_assert` + drop would be equivalent in the happy path and non-fatal in the bad one.

Similarly `main.rs:1253` `serde_json::from_value(event_json).expect("valid event JSON from DB row")` panics the whole reminder-scheduler task on one malformed row, taking down reminder delivery for the pod.

##### D-07 (S1) Hard `process::exit(1)` bypasses the audit drain and span flush
`main.rs:1153` exits with code 1 after a 30 s drain timeout. The audit drain (`main.rs:1049`) and OTEL shutdown (`main.rs:1053`) are downstream of `serve()` returning, so an overrunning drain loses every buffered audit entry (up to 1,000, `state.rs:654`) and every pending span. It also reports a *failed* termination to Kubernetes for what is a graceful-shutdown overrun. A 30 s drain that is still holding connections is exactly the case where the audit trail matters most.

##### D-08 (S1) `/_mesh` and `/metrics` are unauthenticated on hard-coded `0.0.0.0`
`main.rs:1116` binds the health router to `("0.0.0.0", config.health_port)` and `metrics.rs:74` binds the exporter to `([0,0,0,0], port)` — both ignore `BUZZ_BIND_ADDR`'s address. `GET /_mesh` (`router.rs:230` → `router.rs:380-386`) serialises the full `MeshStatus`, including every peer's `endpoint_addrs` (`crates/buzz-relay-mesh/src/status.rs:20`) — internal socket addresses of the relay fleet. `GET /metrics` publishes per-community gauges labelled with the tenant **host** (`main.rs:1341-1359`, `:1546-1552`) — precisely the enumeration the `404` at `router.rs:292-297` is designed to prevent. `GET /_readiness` additionally discloses which backing store is down (`router.rs:369-372`). No auth, no CORS restriction, no rate limit, no way to bind them to loopback.

##### D-09 (S1) Two unbounded per-pubkey `DashMap`s
`observer_rate_limiter` (`state.rs:589`) and `media_upload_rate_limiter` (`state.rs:592`) are `DashMap<(CommunityId,[u8;32]), (u32, Instant)>` with **no capacity cap, no TTL, and no eviction anywhere in `state.rs`**. Keys are attacker-chosen pubkeys. The sibling field three lines below spells out the exact threat model and defends against it: `invite_claim_rate_limiter`'s doc says "the cache has a hard capacity because pre-membership callers can cheaply generate fresh Nostr keys" (`state.rs:593-596`) and uses a bounded moka cache (`state.rs:775-780`). The same reasoning applies verbatim to the two `DashMap`s; the defence was not applied. `media_uploads_in_flight` (`state.rs:600`) has the same shape (decremented by `api/media.rs`, so bounded in the happy path only).

##### D-10 (S1) Per-IP connection limiting is implemented and never called
`RateLimiter::check_ip_connection` is declared at `crates/buzz-auth/src/rate_limit.rs:188` and implemented by `RedisRateLimiter` at `crates/buzz-pubsub/src/rate_limiter.rs:112`. The only other reference in the workspace is the test stub at `admission.rs:85`. **Zero production callers.** The relay therefore has no IP-level connection-flood control; the only bound is `conn_semaphore` at 10,000 (`config.rs:449-452`).

Compounding it: the client IP *is* derived (`router.rs:236-240`), passed to `handle_connection` (`router.rs:316`), and stored on `ConnectionState::remote_addr` (`connection.rs:61`, populated `:170`) — and then **never read in production**. The only reads are test fixtures (`state.rs:1358`, `handlers/event.rs:1365`). The IP is captured and discarded: not logged, not audited, not limited on.

##### D-11 (S1) `/metrics` per-community host labels leak tenant identity by default
Same code path as D-08, but the *default* matters independently: `EmissionScope::from_env` defaults to `All` (`main.rs:65`), so a fresh deployment publishes one labelled series per community per metric. `BUZZ_USAGE_METRICS_PER_COMMUNITY=off` disables it, but nothing in `.env.example` mentions the variable exists.

---

#### S2 — Latent bugs, operational hazards, doc drift

##### D-12 (S2) `AppState::audit` is a completely unread field
`state.rs:496` declares `pub audit: Option<Arc<AuditService>>`, written at `state.rs:718`. Grep across `crates/**` and `desktop/src-tauri/**`: **no read anywhere**. All audit writes go through the separate `audit_tx` channel (`state.rs:555`, `:763`), and the `is_some()` gate uses the local `audit_enabled` (`state.rs:716`), not the field. The `AuditService` stays alive only because the worker closure captured `audit_for_worker` (`state.rs:656`). Pure retained state on a struct that is cloned into every request.

##### D-13 (S2) `RelayError` — 9 of 10 variants dead, and it is unusable by HTTP handlers
`error.rs:8-48` declares 10 variants. Only `InvalidMessage` (`error.rs:44`) is ever constructed, exclusively in `protocol.rs:43-171`. Dead: `WebSocket:11`, `Json:15`, `Database:19`, `Auth:23`, `PubSub:27`, `ConnectionLimitReached:31`, `RateLimitExceeded:35`, `NotAuthenticated:39`, `Internal:47`. The four `#[from]` conversions (`error.rs:15,19,23,27`) are therefore unreachable. `crate::error::Result` has exactly one importer (`protocol.rs:6`). There is no `impl IntoResponse for RelayError`, which is why HTTP handlers invented a third error style (raw `Response` tuples). The type is 50 lines of scaffolding for one parse-error variant.

##### D-14 (S2) All three `lib.rs` re-exports are unused, and nothing depends on the crate as a library
`lib.rs:53-55` re-exports `Config`, `RelayError`, `Result`, `AppState`. No workspace crate declares `buzz-relay` as a dependency (`buzz-admin`, `buzz-conformance`, `buzz-relay-mesh`, `git-sign-nostr` mention it only in comments), and `main.rs:17-24` imports through full module paths (`buzz_relay::config::Config`, `buzz_relay::state::AppState`). The entire 21-module `pub` surface exists to serve one in-crate binary, which means `#![warn(missing_docs)]` is doing documentation work for an audience of zero and every `pub` should probably be `pub(crate)`.

##### D-15 (S2) Postgres pool accounting in `DbConfig`'s doc is wrong
`crates/buzz-db/src/lib.rs:244-246` states "At 20 main + 5 audit = 25/pod, four relay pods fit within the PG limit" (PG `max_connections=100`). `main.rs` opens a **third** pool for search (`main.rs:378-382`) with **no sizing knobs set at all** — sqlx defaults apply — and a fourth (replica) at the same 20 when `READ_DATABASE_URL` is set (`crates/buzz-db/src/lib.rs:363` → `:381-386`). Real ceiling per pod: 20 + 5 + search-default (+20 with a replica). The four-pod arithmetic does not hold, and the two extra pools are **not instrumented** — `main.rs:955-985` reports only `db.pool_stats()`, `db.read_pool_stats()`, and Redis. A search-pool exhaustion is invisible.

##### D-16 (S2) ARCHITECTURE.md claims rate limiting does not exist — three places
- `ARCHITECTURE.md:390`: "No Redis-backed rate limiter exists anywhere in the codebase — rate limiting is not currently enforced."
- `ARCHITECTURE.md:459`: buzz-pubsub "**Does NOT:** implement the rate limiter."
- `ARCHITECTURE.md:823` (§9 Known Limitations #2, header at `:816`): "Only implementation is `AlwaysAllowRateLimiter` (test stub) … none are enforced." §9 is prefaced by "These are verified gaps in the current implementation — not design aspirations."

All three are false. `RedisRateLimiter` lives at `crates/buzz-pubsub/src/rate_limiter.rs:88-99` (i.e. in the very crate `:459` says does not implement it), is constructed at `state.rs:712`, held at `state.rs:584`, and enforced at `api/bridge.rs:31`, `connection.rs:616`, and `connection.rs:639` — fail-closed on limiter error (`admission.rs:29-33`). A reader trusting §9 would conclude rate limiting needs building from scratch and would likely build a second, competing limiter.

The *accurate* residual gap is narrower and is D-02 + D-10: 3 of 7 configured tiers have no reader, and per-IP limiting is implemented but never called.

##### D-17 (S2) NIP-11 `max_limit` over-advertises by 10×
`nip11.rs:106` advertises `max_limit: Some(10_000)`. `crates/buzz-db/src/event.rs:331-332`: `let clamp = q.max_limit.unwrap_or(1000); let limit_val = q.limit.unwrap_or(100).min(clamp);`. Ordinary REQ handling does not raise `max_limit` (the only setter is the COUNT-fallback path, `handlers/req.rs:759`). A client honouring the advertised limit and requesting 10,000 events silently receives 1,000.

##### D-18 (S2) NIP-45 (COUNT) and NIP-98 are implemented but not advertised
`SUPPORTED_NIPS` (`nip11.rs:15`) omits 45 despite `ClientMessage::Count` (`protocol.rs:36`), `RelayMessage::count` (`protocol.rs:211`), `handlers/count.rs:280`, and `POST /count` (`router.rs:73`) all existing. It also omits 98 despite NIP-98 auth being mandatory on 12 routes (`api/bridge.rs:111`, `api/invites.rs:193`, `api/operator.rs`). Nostr clients use `supported_nips` for capability negotiation; both omissions cause clients to avoid features the relay supports.

##### D-19 (S2) `due_delivery_mode: "push"` advertised unconditionally
`nip11.rs:112` hard-codes `due_delivery_mode: Some("push")`. When `push_gateway_delivery_url` is `None` the matcher and delivery worker are never spawned (`main.rs:684-691`), so the relay advertises push delivery it cannot perform. `restricted_writes: true` (`nip11.rs:111`) is likewise unconditional even on a fully open relay (`require_relay_membership=false`, `pubkey_allowlist_enabled=false`).

##### D-20 (S2) The NIP-43/`self` consistency check is compiled out of release builds
`nip11.rs:143-146` uses `debug_assert!` to catch `advertise_nip43=true` without `relay_self`. A release build ships the inconsistent document — clients would be told the relay speaks NIP-43 with no key to verify NIP-43 events against. The test `nip11.rs:513-517` proves the assert fires in debug only. Given `nip11.rs:139-142` calls this "a programmer error", a runtime `if` that drops NIP-43 from the list would be strictly safer.

##### D-21 (S2) Stale `#[ignore]` instructions in `tenant.rs`
`tenant.rs:225-236` and `:238-243` instruct the reader to "Delete this `#[ignore]` when the fix lands" and give a `cargo test --include-ignored` invocation. The fix landed (`tenant.rs:81-88`) and the attributes were removed — there are **zero** `#[ignore]` attributes anywhere in this file group. The instructions now describe a state that does not exist and imply the tests are still red gates.

##### D-22 (S2) `disconnect_pubkey_clusterwide`'s "unrepresentable" claim is false
`state.rs:1026-1030`: "Callers must not invoke the pod-local `conn_manager.disconnect_pubkey` directly … Pairing both halves here makes that mistake unrepresentable." `ConnectionManager::disconnect_pubkey` is `pub` (`state.rs:310`) and is called directly at `main.rs:918`. That call is *correct* (it is the receiver side of the cross-pod command, which must not re-publish), but the doc neither carves out the exception nor makes the mistake unrepresentable. `pub(crate)` plus a named receiver-side wrapper would make the claim true.

##### D-23 (S2) Weak-by-default secrets that silently activate in production
| Item | Default | Cite |
|------|---------|------|
| relay signing key | hard-coded `0000…0001` whenever `BUZZ_RELAY_PRIVATE_KEY` is unset **and** `require_auth_token=false` (itself the default) | `main.rs:396-408`, `config.rs:475-477` |
| `BUZZ_GIT_HOOK_HMAC_SECRET` | fresh random per **process** ⇒ different secret on every pod in a multi-pod deployment | `config.rs:739-744` |
| `BUZZ_S3_ACCESS_KEY` / `BUZZ_S3_SECRET_KEY` | `buzz_dev` / `buzz_dev_secret` | `config.rs:622-625` |
| `DATABASE_URL` | `postgres://buzz:buzz_dev@localhost:5432/buzz` | `config.rs:410-411` |

Each is individually reasonable for dev; composed with the five permissive auth defaults (`require_auth_token=false`, `pubkey_allowlist_enabled=false`, `require_relay_membership=false`, `require_media_get_auth=false`, permissive CORS) a bare deployment is fully open with a well-known signing key. Only one of the five emits a warning (`config.rs:590-594`).

##### D-24 (S2) `Config` derives `Debug` over six secret-bearing fields with no redaction
`config.rs:50` `#[derive(Debug, Clone)]` covers `database_url:55`, `read_database_url:58`, `redis_url:60`, `relay_private_key:92`, `git_hook_hmac_secret:239`, and `media.s3_secret_key`/`s3_access_key` (populated `config.rs:622-625`). No `SecretString`, no `Zeroize`, no manual `Debug`. Verified that no current call site prints it (`main.rs:128-136` selects 6 safe fields; `AppState`'s manual `Debug` prints 2, `state.rs:1209-1215`) — so this is a loaded gun, not a live leak. One `tracing::debug!(?config)` added during future debugging exfiltrates the PG password, the relay private key, the S3 secret, and the git hook HMAC into the log pipeline.

Related, and live: `config.rs:538-542` echoes the raw `RELAY_OWNER_PUBKEY` value into a `warn!` when it fails validation — an nsec pasted into the wrong variable would be logged verbatim.

##### D-25 (S2) `.env.example` documents 5 of 93 vars and 2 that no longer exist
`.env.example` (233 lines) names 12 variables, only 5 of which this module reads. Missing: every security switch (`BUZZ_REQUIRE_AUTH_TOKEN`, `BUZZ_REQUIRE_RELAY_MEMBERSHIP`, `BUZZ_PUBKEY_ALLOWLIST`, `BUZZ_REQUIRE_MEDIA_GET_AUTH`, `BUZZ_CORS_ORIGINS`, `BUZZ_RELAY_PRIVATE_KEY`, `RELAY_OWNER_PUBKEY`, `RELAY_OPERATOR_PUBKEYS`, `RELAY_OPERATOR_API_ORIGIN`, `BUZZ_AUDIT_ENABLED`, `BUZZ_AUTO_MIGRATE`, `BUZZ_ADMIN_HOST`). Present but dead: `TYPESENSE_API_KEY` (`.env.example:40`), `TYPESENSE_URL` (`.env.example:41`) — no Rust code reads either; search is Postgres FTS (`main.rs:369-372`, `handlers/event.rs:482-483`). AGENTS.md's onboarding step is `cp .env.example .env`, so the documented path produces a fully permissive relay with no hint the switches exist.

##### D-26 (S2) No security headers outside the admin router
`api/admin/mod.rs:43-56` sets `Cache-Control: no-store`, `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, and a referrer policy — on that router only. The public app router (`router.rs:185-191`), the SPA fallback that serves `text/html` from `web_dir` (`router.rs:214-219`), the health router (`router.rs:225-232`), and `/metrics` have no CSP, no HSTS, no `nosniff`, no frame protection.

##### D-27 (S2) Unauthenticated NIP-11 costs up to 3 Postgres queries per request
`nip11.rs:235-263`: `workspace_icon_for_host` does `bind_community` (`:278`) + `get_community_icon` (`:283`), then `nip11_document` does a **second** `bind_community` (`:246`) whenever push is configured — which is the default (`config.rs:332`, `:752-757`). No cache, no rate limit, on `GET /` and `GET /info`. The two `bind_community` calls resolve the identical host and could trivially share one result.

##### D-28 (S2) `SPROUT_MAX_NOT_BEFORE_DELTA` is read from the environment on every NIP-11 request
`nip11.rs:96-100` calls `std::env::var` + `parse` inside `relay_limitation`, which runs per request. Every other advertised limit comes from `Config`. Inconsistent, and a per-request syscall on an unauthenticated path.

##### D-29 (S2) Legacy `SPROUT_` prefix survives on three env vars
`SPROUT_REMINDER_SCHEDULER_INTERVAL_SECS` (`main.rs:701`), `SPROUT_REMINDER_SCHEDULER_BATCH_LIMIT` (`main.rs:705`), `SPROUT_MAX_NOT_BEFORE_DELTA` (`nip11.rs:97`). Everything else in the repo has been renamed to `buzz`/`BUZZ_`. Renaming is a breaking change for deployed manifests, so this needs an alias-with-deprecation, not a silent rename.

##### D-30 (S2) `AppState::community_disconnect_publish_attempts` is test-only telemetry on a production struct
`state.rs:512` is incremented on every community-archive publish (`state.rs:1063`) but read only at `api/operator.rs:1010,1020`, both inside `#[cfg(test)]` (module opens `api/operator.rs:500`). Its own doc admits "Test/telemetry counter". A production `AtomicU64` field and a fetch_add on a live code path to serve assertions.

##### D-31 (S2) Eight distinct boolean grammars for env vars
Within one config surface: strict `parse_bool` (`true/1/on` vs `false/0/off/""`, error otherwise — `config.rs:363-377`); bare `=="true"||=="1"` silently false otherwise (`config.rs:475,479,483,520,651,848`); **inverted** `!(=="false"||=="0")` (`config.rs:489`); case-insensitive `on` plus `true`/`1` (`config.rs:498,516`); `true|1|yes|on` (`config.rs:682-689`); trimmed case-insensitive `true|1|yes|on` (`main.rs:25-33`); `!= "false"` (`main.rs:469-471`); mere presence via `is_ok()` (`main.rs:549`). An operator cannot predict whether `BUZZ_X=yes` works without reading the source for that specific variable.

Matching split on numerics: hard error for `BUZZ_RATE_LIMIT_*` and `BUZZ_PUSH_GATEWAY_TIMEOUT_MS`; silent default for ~25 others, one of which has a test *asserting* the silent fallback (`config.rs:988-1011`).

---

#### S3 — Maintainability

##### D-32 (S3) File and function sizes
`main.rs` is 1,940 lines with a **977-line `main()`** (`main.rs:83-1060`) that inlines 19 `tokio::spawn` bodies, three multi-branch DB bootstrap sequences, and the full startup narrative. `state.rs` is 1,932 lines. `config.rs` is 1,346 lines with a **~470-line `Config::from_env()`** (`config.rs:405-874`). Extractable units are obvious and self-labelling: the metrics poller (`main.rs:1279-1806`, ~530 lines) is already a coherent module; the NIP-43 bootstrap block (`main.rs:199-320`), the pub/sub consumer trio (`main.rs:804-936`), and the reminder scheduler (`main.rs:693-802`) each stand alone. The mobile app enforces a 1,000-line ceiling (`mobile/scripts/check-file-sizes.mjs`, AGENTS.md); the relay has no equivalent guard.

##### D-33 (S3) Duplicated methods and duplicated constants
- `ConnectionManager::pubkey_for` (`state.rs:425-430`) and `ConnectionManager::pubkey_for_conn` (`state.rs:286-291`) are **byte-identical** in signature, body, and intent. Both live: `pubkey_for` ← `handlers/event.rs:460`; `pubkey_for_conn` ← `handlers/event.rs:146/184`, `handlers/side_effects.rs:108`.
- `health_handler` (`router.rs:295-297`) and `liveness_handler` (`router.rs:299-301`) return identical `(StatusCode::OK, "ok")`.
- `/_liveness` and `/_readiness` are registered on **both** routers (`router.rs:68-69` and `:227-228`) with no shared helper.
- Three NIP-11 limits are hard-coded twice with no shared const: `1024` at `nip11.rs:104` and `handlers/req.rs:26`; `10` at `nip11.rs:105` and `protocol.rs:14`; `256` at `nip11.rs:107` and `protocol.rs:11`. D-17 is exactly what happens when one copy drifts.

##### D-34 (S3) Two distinct `AdmissionError` types
`crate::admission::AdmissionError` (`admission.rs:12`, `Exceeded`/`Unavailable`) and `crate::audio::room::AdmissionError` (`audio/room.rs:83`, `Full`/`Ended`/`VersionMismatch`). Both are referenced from sibling call sites (`connection.rs:657`, `audio/handler.rs:515`), so every use must be path-qualified to stay unambiguous.

##### D-35 (S3) 26 production `unwrap`/`expect` against the repo's own rule
AGENTS.md: "Do not introduce new `unwrap()` or `expect()` in production paths." Count outside `#[cfg(test)]` in this group: **26** — `metrics.rs` 17 (`:79,84,89,94,99,104,109,114,119,124,129,134,136,141,143,145,180`), `main.rs` 4 (`:90,401,1230,1253`), `state.rs` 3 (`:446,701,708`), `config.rs` 1 (`:507`), `protocol.rs` 1 (`:189`). Plus `panic!` (`main.rs:409`), `unreachable!` (`main.rs:460`), `debug_assert!` (`nip11.rs:143`), `process::exit` (`main.rs:1153`).

Most are boot-time over compile-time literals and defensible; two are request/task-path (D-06). `main.rs:409`'s `panic!` is also stylistically inconsistent with the two adjacent preconditions that `return Err(anyhow!(…))` (`main.rs:206-211`, `:216-219`).

##### D-36 (S3) `metrics.rs` has zero tests
207 lines, 17 `expect`s, the `install` function that panics the process if a recorder already exists or the port is taken, and `track_metrics` with a hand-rolled `caller`-header validator (`metrics.rs:187-196`) — no test coverage at all. Every other file in the group has 4–22 tests (92 total). The header filter (length ≤64, `[A-Za-z0-9_-]`) is the one piece of untrusted-input handling in the file and is untested.

##### D-37 (S3) `admission.rs` is the only module with no doc comment
`lib.rs:5` `mod admission;` is the crate's only private module and `admission.rs:1` starts with a `use`, not `//!`. Every other file in the group opens with module docs (`state.rs:1`, `config.rs:1`, `router.rs:1`, `nip11.rs:1`, `protocol.rs:1`, `tenant.rs:1-15`, `telemetry.rs:1-32`, `metrics.rs:1-19`, `error.rs:1`). Relatedly, `lib.rs:43` `pub mod storage_sweep;` is the only `pub mod` in `lib.rs` without a `///` line.

##### D-38 (S3) Cache instrumentation covers 2 of 7 caches
Hit/miss counters exist for `membership_cache` (`state.rs:833/836`) and `accessible_channels_cache` (`state.rs:1095/1098`). `local_event_ids`, `channel_visibility_cache`, `observer_owner_cache`, `author_type_cache`, and `invite_claim_rate_limiter` have none — so their capacity (10,000 / 10,000 / 1,000 / 10,000) cannot be tuned from telemetry. `channel_visibility_cached` (`state.rs:1124`) is on the fan-out access-gate path.

##### D-39 (S3) Only 2 of 23 background tasks are cancellable
Cancellable: the audit worker (`AuditShutdownHandle`, `state.rs:1177-1196`) and the community revalidator (`community_revalidator_cancel`, `state.rs:510` → `main.rs:1045`). The other 21 (`main.rs:354,360,366,522,551,599,613,687,688,709,823,853,904,950,1008,1122,1132,1181`; `state.rs:968,1043`; `metrics.rs:146`) are abandoned at process exit with in-flight work. The reaper (#11) and reminder scheduler (#14) both perform DB writes.

##### D-40 (S3) `EmissionScope::allows` ignores its argument
`main.rs:76-78`: `fn allows(&self, _community_id: &Uuid) -> bool { matches!(self, Self::All) }`. The parameter exists solely for the planned `top:<k>` mode documented at `main.rs:44-47`. Honest and documented, but the signature promises per-community discrimination that does not exist, and three call sites pass a UUID that is discarded (`main.rs:1335`, `:1467/1509`, `main.rs:1475`).

##### D-41 (S3) `parse_optional_bool` is a single-caller one-liner
`config.rs:379-381` wraps `parse_bool(name, false)` and has exactly one caller (`config.rs:792`). Two names for one behaviour in a file that already has three overlapping parse helpers.

##### D-42 (S3) `bound_communities` is `pub` with no external caller
`state.rs:111` is `pub` but is only used by `revalidate_registered_communities` in the same file (`state.rs:174`) and by tests. Given D-14 (nothing consumes the crate as a library), `pub(crate)` or private is correct.

##### D-43 (S3) `AppState` derives `Clone` that is never used as a clone
`state.rs:487`. Every consumer takes `&AppState` or `Arc<AppState>`. The derive forces `git_store` (`state.rs:565`, the only non-`Arc` non-pool field) to stay `Clone` and invites accidental deep copies of a 40-field struct.

##### D-44 (S3) Test-only Postgres/Redis coupling in unit tests
`state.rs:1257-1290` (`test_state`) and the five tests using it, plus `config.rs`'s 22 env-mutating tests, need live infrastructure or a specific env. `test_state` points Redis at `redis://127.0.0.1:1` deliberately (`state.rs:1259`) but connects Postgres lazily to the real `database_url` (`state.rs:1260`). `just test-unit` will therefore behave differently depending on the developer's environment. There is no `crates/buzz-relay/tests/` directory to hold these.

##### D-45 (S3) Dev-dependencies fetch two crates from a third-party GitHub repo
`Cargo.toml:84-85` pins `mesh-llm-sdk` and `mesh-llm-host-runtime` to `git = "https://github.com/Mesh-LLM/mesh-llm.git", tag = "v0.73.1"`. `cargo test -p buzz-relay` therefore requires network access to an external org, and the supply-chain surface of the test build is outside the workspace lockfile's registry provenance.

##### D-46 (S3) Unreachable branch in `track_metrics`
`metrics.rs:170` skips `p == "/metrics"`. `/metrics` is served by the Prometheus exporter's own listener (`metrics.rs:73-74`), never registered on the app router, so it can never be a `MatchedPath`. Harmless, but it implies a routing arrangement that does not exist.

---

#### Positives worth preserving

Recorded so a future refactor does not remove them:

- `#![deny(unsafe_code)]` (`lib.rs:1`), **0 unsafe blocks**, **0 TODO/FIXME/HACK/XXX markers** across all 12 files, **0 `#[ignore]`d tests**, 92 tests.
- The compile-time conformance fence at `nip11.rs:329-335` — a documented invariant turned into a build break. The pattern deserves reuse (e.g. for the tenancy seam).
- The row-zero tenancy seam (`tenant.rs`) is trait-based, database-free at the call site, fail-closed on both `None` and `Err`, and has a red-team test module with a negative control (`tenant.rs:249-332`).
- The drain race is closed by an explicit insert-then-check / store-then-iterate pair with a sticky one-way flag and a test that names the interleaving (`state.rs:227-235`, `:353-356`, `:1875-1930`).
- `channel_visibility_cached` caches only the restrictive value so the worst stale entry is over-restrictive, never a leak (`state.rs:1106-1122`).
- Cache-invalidation-closure failure over-invalidates rather than serving stale access state (`state.rs:886-894`, `:932-974`).
- "Collect all DB results before emitting any gauge" prevents mixed fresh/stale metric snapshots (`main.rs:1481-1516`).
- Per-community gauge zero-fill from the host map, with a final `0.0` for disappeared label keys that preserves the host label so renames can be zeroed (`main.rs:1316-1382`).
- Usage-poll jitter uses true per-process randomness with a written rationale about PID 1 in containers (`main.rs:1009-1017`).
- Background sweeps build per-row `TenantContext` from the DB `RETURNING` rather than a default tenant (`main.rs:637-644`, `:741-746`).
- Claim-before-publish with a per-attempt stamp and compare-and-clear release in the reminder scheduler (`main.rs:748-798`) — correct except for D-05's terminal branch.
- Six startup steps become fatal precisely when membership enforcement is on, so a membership relay cannot boot into an unadministrable state (`main.rs:201,214,242,250,276,296`).
