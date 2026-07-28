## Module: buzz-relay — HTTP API surface (`crates/buzz-relay/src/api`)
### Aspect: Configuration

All values are read from `crate::config::Config` (constructed once in
`Config::from_env`, `crates/buzz-relay/src/config.rs`) and reached through
`state.config`. No file in this module group calls `std::env::var` outside
`#[cfg(test)]`.

---

#### 1. Env vars consumed by this module group

##### Authentication / admission

| Env var | Field | Default | Consumed at | Notes |
|---|---|---|---|---|
| `BUZZ_REQUIRE_AUTH_TOKEN` | `require_auth_token` | **`false`** | `bridge.rs:640`, `:908`, `:1341`, `:2033` | ⚠️ **Permissive.** `false` enables the unsigned `X-Pubkey` impersonation path (`bridge.rs:118-127`). Parsed at `config.rs:475-477`; startup warning at `config.rs:588-593`. **Not present in `.env.example`.** |
| `BUZZ_REQUIRE_RELAY_MEMBERSHIP` | `require_relay_membership` | **`false`** | `api/mod.rs:67`, `bridge.rs:808`, `operator.rs:435` | ⚠️ Open relay by default — any valid key may publish, read, and upload. `config.rs:483-485`; default asserted `config.rs:953-956`. |
| `BUZZ_ALLOW_NIP_OA_AUTH` | `allow_nip_oa_auth` | **`false`** | `api/mod.rs:81` | Gates NIP-OA owner delegation *for admission*. The owner-**backfill** path is intentionally unflagged (`api/mod.rs:151-156`). `config.rs:520-522`. |
| `BUZZ_RATE_LIMIT_HUMAN_API_CALLS_PER_MIN` | `auth.rate_limits.human_api_calls_per_min` | **`300`** | `bridge.rs:29` | Window is a fixed 60 s (`bridge.rs:35`). Default `buzz-auth/src/rate_limit.rs:113-115`; env parse `config.rs:291-294`; documented `.env.example:61`. |

##### Relay identity (used for scheme derivation and HMAC keys)

| Env var | Field | Default | Consumed at | Notes |
|---|---|---|---|---|
| `BUZZ_RELAY_URL` (see `config.rs`) | `relay_url` | — | `bridge.rs:635`, `:903`, `:1336`, `:2031`; `media.rs:404`, `:449`; `invites.rs:212`, `:267`; `nip05.rs:55`, `:107`; `operator.rs:219` | **Only its scheme prefix is used** for URL construction (`wss://`/`https://` ⇒ TLS, else plaintext). Its *host* is deliberately never used as an identity — see `bridge.rs:184-193`. `operator.rs:219` is the one exception: `relay_url_authority` extracts the deployment host to protect it from archival. |
| `BUZZ_RELAY_PRIVATE_KEY` | → `state.relay_keypair` | generated if absent (`config.rs:594`) | `bridge.rs:526`, `:1972`; `invites.rs:185`, `:259`, `:300` | Signs 39005/39006/20001 overlays **and** derives the invite/policy HMAC key (`invite_token.rs:112-117`). Rotating it invalidates every outstanding invite — the documented blast-radius control (`invite_token.rs:24-30`). |

##### Media

| Env var | Field | Default | Consumed at | Notes |
|---|---|---|---|---|
| `BUZZ_REQUIRE_MEDIA_GET_AUTH` | `require_media_get_auth` | **`false`** | `media.rs:491`, `:517`, `:626`, `:805` | ⚠️ **Permissive.** Off ⇒ `GET`/`HEAD /media/*` unauthenticated and `Cache-Control: public`. Accepts `true`/`1`/`yes`/`on` case-insensitively (`config.rs:682-689`). Default explicitly justified "for staged client rollout" (`config.rs:973-976`). Documented `.env.example:90`. |
| `BUZZ_MEDIA_UPLOADS_PER_MINUTE` | `media_uploads_per_minute` | `30` | `media.rs:96` | Per `(community, pubkey)`, per pod. Must be > 0 (`config.rs:676-680`). `.env.example:87`. |
| `BUZZ_MEDIA_MAX_CONCURRENT_UPLOADS` | `media_max_concurrent_uploads` | `8` | `state.rs:730` (`Semaphore`), read at `media.rs:117` | Global per pod. `config.rs:663-668`. `.env.example:85`. |
| `BUZZ_MEDIA_MAX_CONCURRENT_UPLOADS_PER_PUBKEY` | `media_max_concurrent_uploads_per_pubkey` | `2`, clamped to the global | `media.rs:126` | `config.rs:669-675`. `.env.example:86`. |
| `BUZZ_MAX_IMAGE_BYTES` | `media.max_image_bytes` | `50 MB` | `media.rs:363`; `router.rs:34` | Contributes to the buffered non-video ceiling **and** to the route body limit. |
| `BUZZ_MAX_VIDEO_BYTES` | `media.max_video_bytes` | `500 MB` | `router.rs:35` | Sets the media-router `RequestBodyLimitLayer` to `max(image, video)` = **500 MB** by default. |
| `BUZZ_MAX_FILE_BYTES` | `media.max_file_bytes` | `100 MB` | `media.rs:364` | Buffered non-video ceiling is `max(image, file)` = **100 MB** in RAM per upload. |
| `BUZZ_MEDIA_UPLOAD_RECORDS` | `media.upload_records_enabled` | **`false`** | `media.rs:247` | Off ⇒ `upload_attribution` returns `None` and no `_uploads/` record is written. `config.rs:651-653`. |
| `BUZZ_MEDIA_UPLOAD_IP_HEADER` | `media.upload_ip_header` | `None` | `media.rs:279` | Trusted-edge header name. **Fail-empty**: missing/malformed/non-public ⇒ nothing recorded; the socket address is never used (`media.rs:243-251`). `config.rs:654-657`. |
| `BUZZ_MEDIA_UPLOAD_PORT_HEADER` | `media.upload_port_header` | `None` | `media.rs:280` | Only kept alongside a valid IP. `config.rs:658-661`. |

##### Join policy (deployment-global, not per-community)

| Env var | Field | Default | Consumed at | Notes |
|---|---|---|---|---|
| `BUZZ_TERMS_OF_SERVICE_MARKDOWN` | `join_policy.terms_markdown` | `None` | `invites.rs:82`, `:98` | Size-capped by `MAX_POLICY_MARKDOWN_BYTES` (`config.rs:780-787`). |
| `BUZZ_PRIVACY_POLICY_MARKDOWN` | `join_policy.privacy_markdown` | `None` | `invites.rs:83`, `:107` | same |
| `BUZZ_AGE_ATTESTATION_REQUIRED` | `join_policy.age_attestation_required` | `false` | `invites.rs:84`, `:177` | `config.rs:792` |
| *(derived)* | `join_policy.version` | `sha256(terms ‖ 0x00 ‖ privacy ‖ 0x00‖age)` | `invites.rs:85`, `:178`, `:186`, `:319`, `:333` | Any edit invalidates every outstanding receipt (`config.rs:799-810`). |
| *(derived)* | `join_policy: Option<…>` | `None` when all three inputs are unset | `invites.rs:76`, `:169`, `:314` | `None` ⇒ the claim gate is skipped entirely and all three policy endpoints 404. Default asserted `config.rs:977-980`. **None of these three vars appear in `.env.example`.** |

##### Operator control plane

| Env var | Field | Default | Consumed at | Notes |
|---|---|---|---|---|
| `RELAY_OPERATOR_API_ORIGIN` | `relay_operator_api_origin` | `None` | `operator.rs:68` | Deliberately **un-prefixed** (shared relay-identity family, `config.rs:544-548`). Unset ⇒ every operator request **500s** (`operator.rs:69-72`). Parsed by `parse_operator_api_origin` (`config.rs:549-553`). |
| `RELAY_OPERATOR_PUBKEYS` | `relay_operator_pubkeys` | empty | `operator.rs:89-95` | Comma-separated 64-hex, lowercased, deduped. An invalid entry is a **hard config error** (unlike `RELAY_OWNER_PUBKEY`, which warns and ignores) because silently dropping an operator would silently disable provisioning (`config.rs:555-576`). Non-empty without the origin is also a hard error (`config.rs:577-581`). Default asserted `config.rs:961-964`. **Neither var appears in `.env.example`.** |

##### Deployment-admin surface

| Env var | Field | Default | Consumed at | Notes |
|---|---|---|---|---|
| `BUZZ_ADMIN_HOST` | `admin.host` | `None` | `admin/auth.rs:8-14`, `:18-21`; `router.rs:56-59`, `:264` | ⚠️ **This single value is the entire authorization boundary** for 5 cross-tenant read endpoints (see security SEC-02). Must be a bare authority — `/`, `\`, `@` are rejected (`config.rs:819-825`). Unset ⇒ the sub-router (and the only security-header middleware in the app) is never mounted. **Not in `.env.example`.** |
| `BUZZ_ADMIN_WEB_DIR` | `admin.web_dir` | `None` | `router.rs:52-55`, `:159-176`, `:266-275` | Validated at startup: must contain `index.html` (`config.rs:826-837`). |

##### Mesh demo

| Env var | Field | Default | Consumed at | Notes |
|---|---|---|---|---|
| `BUZZ_MESH` | `mesh.enabled` → `state.mesh()` | **`false`** | `mesh_demo.rs:64` | Strict parse: `on`/`true`/`1` only (`config.rs:509-512`). |
| `BUZZ_MESH_DEMO_ECHO` | `mesh_demo_echo` | **`false`** | `mesh_demo.rs:67` | Same strict parse (`config.rs:514-518`). Both must be on or the route 404s. **Not in `.env.example`.** |

##### Infrastructure reached indirectly

| Env var | Used by this module via | `file:line` |
|---|---|---|
| `REDIS_URL` / `redis_url` | `state.nip98_replay` (replay seen-set), `state.admission_rate_limiter`, `state.pubsub` (presence), mesh `SessionDirectory` | `state.rs:710-712`; `bridge.rs:141`, `:31`, `:1951` |
| `DATABASE_URL` / `database_url` | `state.db` — every handler | throughout |
| `BUZZ_S3_*` | `state.media_storage` | `media.rs:637-679` |
| `BUZZ_CORS_ORIGINS` | `build_cors_layer` — applies to **every** route including this module's | `router.rs:188`, `:373-397` |
| `BUZZ_WEB_DIR`, `BUZZ_SERVE_GIT_WEB_GUI` | SPA fallback that serves `/invite/{code}` | `router.rs:155-186`, `:191-198` |

---

#### 2. Permissive security-relevant defaults (flagged)

| # | Var | Default | Consequence | Cited at |
|---|---|---|---|---|
| 1 | `BUZZ_REQUIRE_AUTH_TOKEN` | `false` | Unsigned `X-Pubkey` header impersonates any pubkey on `/events`, `/query`, `/count`, and all three `/moderation/*` reads; NIP-98 replay protection is bypassed on that path. **Highest-impact default in the module.** | `config.rs:475-477`; `bridge.rs:118-127`, `:150-153` |
| 2 | `BUZZ_REQUIRE_RELAY_MEMBERSHIP` | `false` | Open relay: any valid key may publish, read, and upload media. Also means the unbounded media limiter map (`state.rs:774`) is keyed by an attacker-mintable identity. | `config.rs:483-485`; `api/mod.rs:67-69`; `media.rs:196-206` |
| 3 | `BUZZ_REQUIRE_MEDIA_GET_AUTH` | `false` | `GET`/`HEAD /media/*` unauthenticated; blob responses cached as `public`. `verify_blossom_get_auth` is dead code in this configuration. | `config.rs:682-689`; `media.rs:491-494`, `:517-523` |
| 4 | `BUZZ_CORS_ORIGINS` | empty ⇒ `CorsLayer::permissive()` | Any origin may read cross-origin responses from every route in this module, including `/query` results and `/moderation/*`. Mitigated for the *authenticated* routes by the NIP-98 requirement (no cookies/ambient credentials), but a real concern for `/media/*` when read auth is off. Note the failure path is correct: an unparseable explicit list falls back to **deny**, not permissive (`router.rs:383-392`). | `router.rs:373-397`; `config.rs:595-600` |
| 5 | `BUZZ_ADMIN_HOST` | `None` (safe) — but when **set**, it is the whole boundary | Setting it exposes 5 cross-tenant read endpoints with no credential; a spoofable `Host` header is sufficient. Safe-by-default, unsafe-once-enabled. | `admin/auth.rs:16-33`; `docs/admin/README.md:64-70` |
| 6 | `BUZZ_MAX_VIDEO_BYTES` = 500 MB | route body limit becomes `max(image, video)` | The media router's `RequestBodyLimitLayer` is **500 MB** — 10× the "50 MB limit" claimed in `ARCHITECTURE.md:623`. Bounded in practice by the pre-body auth extractor (`media.rs:29-38`) and the concurrency semaphore. | `router.rs:33-36` |
| 7 | `BUZZ_MAX_FILE_BYTES` = 100 MB | non-video buffered ceiling | Worst case per pod ≈ `max(image, file) × media_max_concurrent_uploads` = 100 MB × 8 = **800 MB** resident. | `media.rs:362-367`; `state.rs:730` |
| 8 | `join_policy` | `None` | The consent gate does not exist unless explicitly configured; `claim` accepts without any receipt. | `config.rs:794-799`; `invites.rs:314-316` |

**Non-permissive defaults worth noting as correct:** `mesh`/`mesh_demo_echo` off with a strict
`on`/`true`/`1` parse so typos fail closed (`config.rs:509-518`); `upload_records_enabled` off with
fail-empty IP capture (`media.rs:243-251`); `allow_nip_oa_auth` off; `relay_operator_pubkeys` empty
so provisioning is disabled until explicitly configured, with a startup hard-error if the pubkeys
are set without the origin (`config.rs:577-581`).

---

#### 3. Hard-coded constants (not configurable)

| Constant | Value | Purpose | `file:line` |
|---|---|---|---|
| `BRIDGE_FEED_MAX_LIMIT` | 100 | feed page cap | `bridge.rs:270` |
| `BRIDGE_THREAD_MAX_LIMIT` | 500 | thread page cap | `bridge.rs:271` |
| `BRIDGE_WINDOW_DEFAULT_LIMIT` / `_MAX_LIMIT` | 50 / 200 | channel-window rows | `bridge.rs:374-375` |
| aux-closure per-hop limit | 1000 | reaction/deletion closure | `bridge.rs:492` |
| FTS `per_page` ceiling | 500 | `limit.min(500)` | `bridge.rs:1665` |
| `REJECT_REASON_MAX_BYTES` | 256 | log truncation of attacker-controlled text | `bridge.rs:595` |
| `MODERATION_READ_LIMIT` | 500 | moderation row cap | `bridge.rs:2059` |
| `MAX_RANGE_CHUNK` | 16 MiB | 206 response cap | `media.rs:587` |
| `MEDIA_UPLOAD_RATE_WINDOW` | 60 s | upload limiter window | `media.rs:66` |
| upload-extractor Blossom window | 3600 s | pre-type-detection permissive window | `media.rs:177` |
| media-read Blossom window | 3600 s | `verify_blossom_get_auth` max age | `media.rs:502` |
| `SNIFF_BYTES` | 4096 | content-sniff prefix | `media.rs:321` |
| `is_safe_ext` bounds | 1–8 chars, `[a-z0-9]` | path-segment gate | `media.rs:533-535` |
| `CLAIM_RATE_WINDOW` / `CLAIM_RATE_LIMIT` / `CLAIM_RATE_CACHE_CAPACITY` | 60 s / 10 / 10 000 | invite-claim limiter | `invites.rs:34`, `:37`, `:41` |
| `DEFAULT_INVITE_TTL_SECS` / `MAX_INVITE_TTL_SECS` / mint TTL floor | 72 h / 30 d / 60 s | invite lifetime | `invite_token.rs:52`, `:55`, `:129` |
| `MAX_CODE_LEN` | 1024 | pre-parse input bound | `invite_token.rs:57` |
| policy-receipt max length / TTL | 2048 bytes / 10 min | receipt bounds | `invite_token.rs:369`, `:350` |
| `KEY_DERIVATION_LABEL` | `b"buzz-invite-v1"` | HMAC domain separation | `invite_token.rs:58` |
| `OPERATOR_REPLAY_SCOPE` | `"operator-management"` | replay namespace | `operator.rs:55` |
| admin `limit` range | 1–200, default 50 | report page cap | `admin/mod.rs:75-83` |
| admin feedback limit | **100, hard-coded** | no client control | `admin/mod.rs:155` |
| admin `summarize_body` cap | 240 chars | summary truncation | `admin/mod.rs:296` |
| admin body limit | 1024 bytes | `RequestBodyLimitLayer` | `admin/mod.rs:39` |
| `ECHO_TIMEOUT` | 10 s | mesh demo wait | `mesh_demo.rs:44` |
| `FILTER_QUERY_CONCURRENCY` | 4 (compile-time asserted 2..=8) | bounded-concurrent DB reads | `handlers/req.rs:35`, `:41` |
| bridge/api body limit | 1 MiB | `RequestBodyLimitLayer` | `router.rs:130` |

Candidates for promotion to config: `MODERATION_READ_LIMIT`, `MAX_RANGE_CHUNK`, the invite TTLs, and
the admin feedback limit — all are operational tuning knobs currently requiring a rebuild.

---

#### 4. `.env.example` coverage gap

Documented (`.env.example`): `BUZZ_RATE_LIMIT_HUMAN_API_CALLS_PER_MIN` (:61),
`BUZZ_MEDIA_MAX_CONCURRENT_UPLOADS` (:85), `BUZZ_MEDIA_MAX_CONCURRENT_UPLOADS_PER_PUBKEY` (:86),
`BUZZ_MEDIA_UPLOADS_PER_MINUTE` (:87), `BUZZ_REQUIRE_MEDIA_GET_AUTH` (:90).

**Undocumented but consumed by this module — 11 vars, including the four most
security-significant ones:**

| Var | Why it matters |
|---|---|
| `BUZZ_REQUIRE_AUTH_TOKEN` | the unsigned-`X-Pubkey` switch (SEC-01) |
| `BUZZ_REQUIRE_RELAY_MEMBERSHIP` | open-vs-closed relay |
| `BUZZ_ADMIN_HOST` | the sole authorization boundary for 5 cross-tenant read routes (SEC-02) |
| `BUZZ_ADMIN_WEB_DIR` | admin SPA |
| `RELAY_OPERATOR_API_ORIGIN` | operator NIP-98 audience |
| `RELAY_OPERATOR_PUBKEYS` | operator allowlist |
| `BUZZ_MESH_DEMO_ECHO` | unauthenticated, body-tenanted demo route (SEC-03) |
| `BUZZ_ALLOW_NIP_OA_AUTH` | agent→owner delegation |
| `BUZZ_TERMS_OF_SERVICE_MARKDOWN` | join-policy gate |
| `BUZZ_PRIVACY_POLICY_MARKDOWN` | join-policy gate |
| `BUZZ_AGE_ATTESTATION_REQUIRED` | join-policy gate |

An operator following `.env.example` alone ships a relay where `/events`, `/query`, `/count`, and the
moderation reads accept unsigned `X-Pubkey` impersonation, with no hint in the template that the
knob exists.

---

#### 5. Startup validation performed for this surface

| Check | Behaviour | `file:line` |
|---|---|---|
| `RELAY_OPERATOR_PUBKEYS` entry not 64-hex | **hard error**, refuses to start | `config.rs:565-570` |
| `RELAY_OPERATOR_PUBKEYS` set without `RELAY_OPERATOR_API_ORIGIN` | **hard error** | `config.rs:577-581` |
| `BUZZ_ADMIN_HOST` contains `/`, `\`, or `@` | **hard error** | `config.rs:819-825` |
| `BUZZ_ADMIN_WEB_DIR` without `index.html` | **hard error** | `config.rs:830-836` |
| policy Markdown over `MAX_POLICY_MARKDOWN_BYTES` | **hard error** | `config.rs:780-787` |
| `BUZZ_REQUIRE_AUTH_TOKEN=false` | warning only | `config.rs:588-593` |
| `BUZZ_CORS_ORIGINS` set but nothing parseable | error log + deny-all CORS (not permissive) | `router.rs:383-392` |
| media knobs ≤ 0 | silently fall back to the default via `.filter(\|&v\| v > 0)` | `config.rs:663-680` |
| `BUZZ_MESH*` typos (e.g. `BUZZ_MESH=yes`) | silently **off** | `config.rs:509-518` |
| `BUZZ_REQUIRE_MEDIA_GET_AUTH` accepts `yes`/`on` | more lenient than the other booleans, which take only `true`/`1` | `config.rs:682-689` vs `:475-477` |

The boolean-parsing inconsistency is worth noting: `require_media_get_auth` accepts four spellings,
`require_auth_token` / `require_relay_membership` / `allow_nip_oa_auth` accept two, and the mesh vars
accept three (case-insensitive `on` plus `true`/`1`). An operator writing `BUZZ_REQUIRE_AUTH_TOKEN=yes`
silently gets `false`.
