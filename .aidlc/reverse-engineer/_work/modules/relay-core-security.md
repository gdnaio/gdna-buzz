## Module: buzz-relay — core & bootstrap (`crates/buzz-relay/src`)
### Aspect: Security

---

#### 1. Auth enforcement architecture

**There is no authentication middleware anywhere in the router.** The only `middleware::from_fn` layers on the app router are `track_metrics` (`router.rs:188`) and, route-scoped, `require_localhost` on the git policy router (`api/git/mod.rs:64`) and `security_headers` on the admin router (`api/admin/mod.rs:38`). Every auth decision lives inside a handler body.

Consequences:
- Adding a route registration without adding an in-handler auth call yields a silently unauthenticated endpoint. Nothing structural prevents it.
- The relay-core files in this group contain **zero** auth checks of their own; they only bind tenancy (`router.rs:288`), refuse during shutdown (`router.rs:311`), cap frame size (`router.rs:324`), and cap concurrency (`state.rs:514-516`).

#### 2. Unauthenticated surface reachable from the routers

| Endpoint | Listener | Line | What it exposes |
|----------|----------|------|-----------------|
| `GET /` (NIP-11 branch) | app | `router.rs:63`, `:275-277`, `:333` | full NIP-11 doc incl. `self` pubkey, community `icon`, push descriptor with the relay's executor pubkey (`nip11.rs:207`) |
| `GET /info` | app | `router.rs:64` | identical document |
| `GET /.well-known/nostr.json` | app | `router.rs:65` | NIP-05 mapping |
| `GET /health` | app | `router.rs:67` | `"ok"` |
| `GET /_liveness` | app **and** health | `router.rs:68`, `:227` | `"ok"` |
| `GET /_readiness` | app **and** health | `router.rs:69`, `:228` | `{"status","postgres","redis"}` — **discloses which backing store is down** (`router.rs:369-372`) |
| `GET /_status` | health | `router.rs:229`, `:387-394` | service name, **exact build version** (`CARGO_PKG_VERSION` = `0.2.0`), uptime |
| `GET /_mesh` | health | `router.rs:230`, `:380-386` | full mesh peer table: `runtime_id`, **`endpoint_addrs`** (internal socket addresses), `proto_version`, `phi`, `load`, `record_version`, `last_heartbeat_millis`, per-peer counters (`crates/buzz-relay-mesh/src/status.rs:8-29`) |
| `GET /metrics` | metrics | `metrics.rs:73` | every gauge/counter/histogram, **including per-community series labelled by host** (`main.rs:1341-1359`, `:1546-1552`) — i.e. an enumeration of tenant hostnames |
| `GET /api/join-policy`, `/terms`, `/privacy` | app | `router.rs:96`, `:99-106` | operator ToS/privacy Markdown |
| `GET /media/{sha256_ext}`, `HEAD` same | app | `router.rs:41-43` | **media blobs — auth is off by default** (`require_media_get_auth`, `config.rs:196-197`, default `false` at `config.rs:682-689`) |
| SPA fallback `/invite/{code}`, `/assets/*` | app | `router.rs:145-183` | static bundle |
| `/` , `/repos`, `/repos/*` SPA | app | `router.rs:210-212` | only when `serve_git_web_gui` |

##### The three hard-coded `0.0.0.0` binds
- Health listener: `("0.0.0.0", config.health_port)` (`main.rs:1116`) — ignores `BUZZ_BIND_ADDR`'s IP.
- Metrics listener: `([0,0,0,0], port)` (`metrics.rs:74`) — same.
- Mesh default bind: `"0.0.0.0:3478"` (`config.rs:507`).

So `/_status`, `/_mesh`, and `/metrics` are exposed on **all** interfaces regardless of `BUZZ_BIND_ADDR`, with no auth, no CORS restriction, and no rate limit. `/_mesh` leaking `endpoint_addrs` is internal-topology disclosure; `/metrics` leaking per-community host labels is tenant enumeration. Both are normally shielded by K8s network policy, but neither is shielded by the application.

#### 3. Tenancy isolation enforcement

Strong points, all verified:

| Control | Cite |
|---------|------|
| Community is bound from the request `Host` **before** the WebSocket upgrade, so no frame is read on an unbound connection | `router.rs:279-300` |
| Empty/whitespace host fails closed **before** the resolver, guarding against a misconfigured `host = ''` row that the schema permits | `tenant.rs:81-88` |
| Unmapped host and lookup error produce a **single generic** `404`, never echoing the host — no probe oracle for which communities exist | `router.rs:292-297`, `tenant.rs:56-59` |
| No default/fallback community path exists anywhere in the seam | `tenant.rs:89-95` |
| Server-internal paths (git, hook, workflow sink, startup) resolve `relay_url` through the same fail-closed helper, not a bypass | `tenant.rs:120-133` |
| Background sweeps build a per-row `TenantContext` from the DB `RETURNING`, never a default | `main.rs:637-644`, `main.rs:741-746` |
| Every cache key is `(CommunityId, …)`-prefixed; predicate invalidation is community-scoped, with over-invalidation as the failure mode rather than serving stale access state | `state.rs:541-553`, `state.rs:881-895`, `state.rs:928-975` |
| Local-echo dedup is community-keyed so a publish in A cannot suppress a same-id event in B | `state.rs:530-540` |
| Ban-driven disconnects are fenced to the banning community | `state.rs:296-308`, `state.rs:328-334` |
| Fan-out compares the receiver-side community label recorded at handshake against the event label | `state.rs:52-54` |
| NIP-11 `build` is pinned by a compile-time const to a static/scalar-only signature so it can never grow an unscoped DB/search/audit input | `nip11.rs:307-335` |
| NIP-98 replay prevention is Redis-backed and community-keyed; the doc forbids replacing it with process-local caching | `state.rs:576-581`, `state.rs:710-711` |
| Admin host is short-circuited **first** in both the `/` handler and the SPA fallback, so it can never serve the public bundle, NIP-11, or the WebSocket endpoint | `router.rs:252-273`, `router.rs:157-168` |

##### Gaps in tenancy isolation
- `/metrics` publishes per-community gauges labelled with the tenant **host** (`main.rs:1341-1359`), which is exactly the information the `404` at `router.rs:292` refuses to disclose. `BUZZ_USAGE_METRICS_PER_COMMUNITY=off` removes it, but the default is `all` (`main.rs:65`).
- `GET /` NIP-11 is served fail-open before binding (deliberate, `router.rs:279-286`) but it also performs 1–3 Postgres queries per request (`nip11.rs:237`, `:246`, `:278`, `:283`), so the unauthenticated path is DB-coupled.

#### 4. Secret handling

##### `Config` is `Debug` with no redaction
`#[derive(Debug, Clone)]` at `config.rs:50` covers fields that hold secrets in cleartext:

| Secret field | Line |
|--------------|------|
| `database_url` (contains the PG password) | `config.rs:55` |
| `read_database_url` | `config.rs:58` |
| `redis_url` (may contain a password) | `config.rs:60` |
| `relay_private_key` | `config.rs:92` |
| `git_hook_hmac_secret` | `config.rs:239` |
| `media.s3_secret_key` / `s3_access_key` (via `buzz_media::MediaConfig`) | `config.rs:187`, populated `config.rs:622-625` |

Any `{:?}` / `tracing::debug!(?config)` would print all of them. Verified that the code does **not** currently do this: the startup log at `main.rs:128-136` selects 6 non-secret fields, and `AppState`'s manual `Debug` prints only `relay_url` and `max_connections` (`state.rs:1209-1215`). So the derive is a loaded gun, not a live leak. There is no `SecretString`/`Zeroize` wrapper anywhere in `config.rs`.

##### Secret-adjacent logging that does happen
- `RELAY_OWNER_PUBKEY` is echoed on invalid input: `warn!("RELAY_OWNER_PUBKEY is not a valid 64-char hex pubkey — ignoring. Got: {s:?}")` (`config.rs:538-542`). A pubkey is not secret, but a mis-set env var (e.g. an nsec pasted into the wrong variable) would be logged verbatim.
- `RELAY_OPERATOR_PUBKEYS` echoes the offending entry in the config error (`config.rs:568-570`).
- The dev relay pubkey is logged at `warn` (`main.rs:402-407`).

##### Weak-secret defaults
| Item | Behaviour | Cite |
|------|-----------|------|
| `BUZZ_GIT_HOOK_HMAC_SECRET` | unset ⇒ a fresh random 32-byte hex secret **per process**. In a multi-pod deployment each pod has a different hook secret. Explicitly-set values must be ≥32 chars | `config.rs:739-744`, `config.rs:862-871` |
| `BUZZ_RELAY_PRIVATE_KEY` | unset **and** `require_auth_token=false` ⇒ hard-coded `0000…0001` with a `warn` | `main.rs:396-408` |
| `BUZZ_S3_ACCESS_KEY` / `BUZZ_S3_SECRET_KEY` | default to `"buzz_dev"` / `"buzz_dev_secret"` | `config.rs:622-625` |
| `DATABASE_URL` | defaults to `postgres://buzz:buzz_dev@localhost:5432/buzz` (annotated `// sadscan:disable np.postgres.1`) | `config.rs:410-411` |

Every one of these is a *dev* default that silently activates in production if the env var is missing. `require_auth_token=false` is itself the default (`config.rs:475-477`), so the hard-coded relay key path is the default path for a bare deployment.

#### 5. CORS and response headers

| Condition | Result | Cite |
|-----------|--------|------|
| `BUZZ_CORS_ORIGINS` unset/empty (**the default**) | `CorsLayer::permissive()` over the entire app router — media, git, invites, operator, moderation, admin | `router.rs:410-412`, `config.rs:595-600` |
| Origins set, at least one parses | allow-list origins, `allow_methods(Any)`, `allow_headers(Any)`; **no `allow_credentials`** | `router.rs:428-431` |
| Origins set, none parse | bare `CorsLayer::new()` after an `error!` — refuses to degrade to permissive | `router.rs:419-426` |

`CorsLayer::permissive()` in `tower-http` sends `Access-Control-Allow-Origin: *` with `Any` methods/headers. Combined with the NIP-98-signed-request model this is not a session-hijack vector (no cookies, no ambient credentials), but it does let any web origin read `/_readiness`, `/info`, `/api/join-policy`, and — when GET auth is off — `/media/{sha256}` blobs cross-origin.

Security headers are applied on **exactly one** router: `api/admin/mod.rs:43-56` sets `Cache-Control: no-store`, `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, and a referrer policy. The public app router, the SPA fallback (`router.rs:214-219`, `router.rs:145-183`), the health router (`router.rs:225-232`), and `/metrics` have **no** security headers — no CSP, no HSTS, no `nosniff`, no frame protection. The SPA fallback serves `text/html` from `web_dir` (`router.rs:216`) with no CSP.

#### 6. Panic / DoS surface

| Surface | Assessment | Cite |
|---------|-----------|------|
| Frame size | capped at the parser (`max_message_size` + `max_frame_size`) before assembly, default 512 KiB | `router.rs:324-332`, `config.rs:15` |
| HTTP body size | 1 MB on the api router, `max(image,video)` on media (default 500 MB), `git_max_pack_bytes` on git (default 500 MB), 1 MB on git policy, 1 KB on admin | `router.rs:129`, `:44`, `api/git/transport.rs:1763`, `api/git/mod.rs:63`, `api/admin/mod.rs:39` |
| Concurrent connections | `conn_semaphore`, default 10,000, `try_acquire_owned` | `state.rs:729`, `config.rs:449-452` |
| Concurrent handlers | `handler_semaphore`, default 1,024 | `state.rs:730` |
| Slow-client backpressure | cancels after `slow_client_grace_limit` (default 15) consecutive full buffers | `state.rs:451-478` |
| Subscription count | 1,024 per connection | `handlers/req.rs:26/66` |
| Filters per REQ / sub-id length | 10 / 256 bytes | `protocol.rs:14`, `:11` |
| **Per-IP connection limiting** | **implemented but never called.** `RateLimiter::check_ip_connection` exists (`crates/buzz-auth/src/rate_limit.rs:188`) and `RedisRateLimiter` implements it (`crates/buzz-pubsub/src/rate_limiter.rs:112`), but the only other reference in the workspace is the test stub at `admission.rs:85`. There is no IP-level flood control | verified by grep |
| Client IP | derived in the router (`router.rs:236-240`) and stored on `ConnectionState` (`connection.rs:61`, `:170`) but **never read in production** — only test fixtures read it (`state.rs:1358`, `handlers/event.rs:1365`). Not used for limiting, logging, or audit | verified by grep |
| UDS clients have no IP at all | `main.rs:1179` uses `.into_make_service()` (no `ConnectInfo`), so `router.rs:239` falls back to `0.0.0.0:0` for every UDS connection | |
| Unbounded per-pubkey `DashMap`s | `observer_rate_limiter` (`state.rs:589`) and `media_upload_rate_limiter` (`state.rs:592`) have **no capacity cap, no TTL, no eviction** in this file. Keys are `(community, caller-chosen pubkey bytes)`. `invite_claim_rate_limiter` explicitly documents this exact risk and defends with a moka capacity cap (`state.rs:593-596`) — the same defence is missing on the other two | memory-growth DoS |
| Unauthenticated DB amplification | `GET /` and `GET /info` each issue 1–3 Postgres queries with no cache and no rate limit; push is enabled by default so the 3-query path is the default | `nip11.rs:237/246/278/283`, `config.rs:752-757` |
| Boot-time panics | 17 `expect` in `metrics.rs:79-145` (compile-time literals), `expect` on the rustls provider (`main.rs:90`), on SIGTERM install (`main.rs:1230`), on the git store (`state.rs:701`) and pack cache (`state.rs:708`); `panic!` at `main.rs:409` | all boot-time, not request-reachable |
| Request-path panic candidates | `state.rs:446` `WsUtf8Bytes::try_from(...).expect("relay fan-out frames are serialized UTF-8 JSON")` — panics inside the fan-out path if a caller ever passes non-UTF-8; `main.rs:1253` `expect("valid event JSON from DB row")` in the reminder scheduler — panics the scheduler task on a malformed DB row; `metrics.rs:180` `path.unwrap()` (guarded by the `None` arm above, safe) | |
| `debug_assert` in release | `nip11.rs:143` — the NIP-43-without-`self` inconsistency check is compiled out of release builds | |
| Hard exit | `std::process::exit(1)` at `main.rs:1153` bypasses the audit drain and OTEL flush | |

`#[deny(unsafe_code)]` at `lib.rs:1`, **0 unsafe blocks**, and **0 `TODO`/`FIXME`/`HACK`/`XXX` markers** across all 12 files.

#### 7. Auth-relevant configuration defaults (all permissive)

| Setting | Default | Effect at default | Cite |
|---------|---------|-------------------|------|
| `BUZZ_REQUIRE_AUTH_TOKEN` | `false` | REST bypasses token auth; a startup `warn` is emitted. Also unlocks the hard-coded dev relay key | `config.rs:475-477`, `:590-594`, `main.rs:396-408` |
| `BUZZ_PUBKEY_ALLOWLIST` | `false` | any NIP-42 pubkey may connect | `config.rs:479-481` |
| `BUZZ_REQUIRE_RELAY_MEMBERSHIP` | `false` | relay-level membership check is a no-op for all authenticated callers | `config.rs:483-485` |
| `BUZZ_REQUIRE_MEDIA_GET_AUTH` | `false` | media GET/HEAD served without Blossom auth or membership | `config.rs:682-689` |
| `BUZZ_CORS_ORIGINS` | unset | permissive CORS | `router.rs:410-412` |
| `BUZZ_ALLOW_NIP_OA_AUTH` | `false` | fail-closed (good default) | `config.rs:520-523` |
| `RELAY_OPERATOR_PUBKEYS` | empty | community provisioning **disabled** — fail-closed (good default) | `config.rs:164-168`, `:555-576` |
| `BUZZ_MESH` | off | nothing bound — fail-closed (good default) | `config.rs:492-509` |
| `BUZZ_AUTO_MIGRATE` | off | no schema mutation — fail-closed (good default) | `main.rs:25-33` |
| `BUZZ_AUDIT_ENABLED` | **`true`** | audit on by default (good default) | `config.rs:793` |
| `BUZZ_GIT_CONFORMANCE_PROBE` | on (`!= "false"`) | S3 CAS is proven before serving git — fail-closed (good default) | `main.rs:469-471` |

The five permissive defaults at the top of the table compose into a fully open relay from a bare `docker run` with only `DATABASE_URL`/`REDIS_URL` set: no REST token, no allowlist, no membership, unauthenticated media reads, `*` CORS, and a deterministic well-known signing key. Each individually has a rationale (dev ergonomics) and one emits a warning; the composition does not.

#### 8. Deliberate fail-closed behaviours worth preserving

| Behaviour | Cite |
|-----------|------|
| Rate-limit admission denies when Redis is unreachable | `admission.rs:29-33` |
| Host binding rejects on lookup error, not just unmapped | `tenant.rs:93-94` |
| Empty host rejected before the resolver | `tenant.rs:85-87` |
| Channel-visibility cache stores only `private`, never a permissive value, so the worst stale entry is over-restrictive | `state.rs:1106-1122` |
| Cache-invalidation-closure failure over-invalidates rather than serving stale access | `state.rs:886-894`, `:932-974` |
| Community-lifecycle revalidation retains sockets on lookup failure but keeps sweeping | `state.rs:1082-1085` |
| Push lease acceptance is disabled without an exact gateway URL, so no undeliverable work accumulates | `main.rs:684-691`, `config.rs:342-360` |
| Git policy endpoint 403s when `ConnectInfo` is absent (`unwrap_or(false)`) — so the UDS listener, which has no `ConnectInfo`, cannot reach it | `api/git/mod.rs:41-48`, `main.rs:1179` |
| Mesh demo echo requires **two** explicit opt-ins | `router.rs:121-123`, `config.rs:514-518` |
| Membership enforcement makes six otherwise-non-fatal startup steps fatal | `main.rs:201`, `:214`, `:242`, `:250`, `:276`, `:296` |
