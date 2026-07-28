## Module: buzz-relay — core & bootstrap (`crates/buzz-relay/src`)
### Aspect: Configuration

**93 distinct environment variables** are read by the files in this group: 74 in `config.rs`, 15 in `main.rs` (14 explicit + `RUST_LOG` via `EnvFilter`), 1 in `nip11.rs`, 3 in `telemetry.rs` (2 explicit + `OTEL_RESOURCE_ATTRIBUTES` via the SDK detector). A further 4 are read by `storage_sweep.rs` and consumed only from `main.rs` (§4).

There is no config file, no CLI flag parsing, and no config reload. Everything is env-only, read once at boot — except two vars read **per request/per tick** (§5).

---

#### 1. `config.rs` — every env var (74)

##### 1a. Listeners and identity

| Var | Type | Default | Read | Consumed |
|-----|------|---------|------|----------|
| `BUZZ_BIND_ADDR` | `SocketAddr` | `0.0.0.0:3000` | `config.rs:407` | `main.rs:1157` (TCP bind). Invalid ⇒ **hard error** `InvalidBindAddr` (`config.rs:265-268`) |
| `RELAY_URL` | `String` | `ws://localhost:3000` | `config.rs:428` | `main.rs:239` (deployment community), `main.rs:568`, `nip11.rs:241`, `tenant.rs:130`. **Never validated as a URL** |
| `BUZZ_PAIRING_RELAY_URL` | `Option<String>` | `None` | `config.rs:430` | `nip11.rs:243` only. Must be `ws`/`wss` with a host ⇒ else hard error (`config.rs:432-446`) |
| `BUZZ_UDS_PATH` | `Option<String>` | `None` | `config.rs:604` | `main.rs:1162-1187`; a `warn` on non-unix (`main.rs:1207-1211`) |
| `BUZZ_HEALTH_PORT` | `u16` | `8080` | `config.rs:609` | `main.rs:1116` — bound to hard-coded `0.0.0.0`, ignores `BUZZ_BIND_ADDR`'s IP |
| `BUZZ_METRICS_PORT` | `u16` | `9102` | `config.rs:614` | `main.rs:138` → `metrics.rs:74` — also hard-coded `0.0.0.0` |
| `BUZZ_RELAY_PRIVATE_KEY` | `Option<String>` | `None` | `config.rs:602` | `main.rs:393-394` (parse or abort), `nip11.rs:302` (gates `self` + NIP-43). **Not validated at config load** — only when `main` parses it |
| `RELAY_OWNER_PUBKEY` | `Option<String>` (64 hex) | `None` | `config.rs:526` | `main.rs:201`, `main.rs:292-294`. Invalid ⇒ **warn-and-ignore, value echoed in the log** (`config.rs:538-542`) |

##### 1b. Datastores

| Var | Type | Default | Read | Consumed |
|-----|------|---------|------|----------|
| `DATABASE_URL` | `String` | `postgres://buzz:buzz_dev@localhost:5432/buzz` | `config.rs:410` | `main.rs:146` (main pool), `main.rs:328` (audit pool), `main.rs:375` (search fallback) |
| `READ_DATABASE_URL` | `Option<String>` | `None`; blank/whitespace normalised to `None` (`config.rs:413-416`) | `config.rs:413` | `main.rs:147` (replica pool), `main.rs:373` (search pool preference), `main.rs:384` (log) |
| `REDIS_URL` | `String` | `redis://localhost:6379` | `config.rs:419` | `main.rs:337`, `main.rs:344` |
| `BUZZ_REDIS_POOL_SIZE` | `usize` >0 | `16` | `config.rs:421` | `main.rs:338`. Zero/junk ⇒ silent fallback to 16 (tested `config.rs:988-1011`) |

##### 1c. Connection limits

| Var | Type | Default | Read | Consumed |
|-----|------|---------|------|----------|
| `BUZZ_MAX_CONNECTIONS` | `usize` | `10_000` | `config.rs:449` | `state.rs:649` → `conn_semaphore` (`state.rs:729`) |
| `BUZZ_MAX_CONCURRENT_HANDLERS` | `usize` | `1_024` | `config.rs:454` | `state.rs:650` → `handler_semaphore` (`state.rs:730`) |
| `BUZZ_SEND_BUFFER` | `usize` | `1_000` | `config.rs:459` | `connection.rs:159` |
| `BUZZ_MAX_FRAME_BYTES` | `usize` >0 | `524_288` (`config.rs:15`) | `config.rs:464` | `router.rs:302` → `router.rs:330-331`, `nip11.rs:242` (advertised `max_message_length`) |
| `BUZZ_SLOW_CLIENT_GRACE_LIMIT` | `u8` | `15` | `config.rs:470` | `connection.rs:179`, `connection.rs:212` |

##### 1d. Auth / membership

| Var | Grammar | Default | Read | Consumed |
|-----|---------|---------|------|----------|
| `BUZZ_REQUIRE_AUTH_TOKEN` | `=="true"\|\|=="1"` | `false` | `config.rs:475` | `api/bridge.rs`, `main.rs:395` (dev-key gate), `main.rs:409` (panic gate). Emits a startup `warn` when false (`config.rs:590-594`) |
| `BUZZ_PUBKEY_ALLOWLIST` | `=="true"\|\|=="1"` | `false` | `config.rs:479` | `handlers/auth.rs:187` |
| `BUZZ_REQUIRE_RELAY_MEMBERSHIP` | `=="true"\|\|=="1"` | `false` | `config.rs:483` | 22 sites; `main.rs:201/214/242/250/276/296/506`, `nip11.rs:304` |
| `BUZZ_ALLOW_NIP_OA_AUTH` | `=="true"\|\|=="1"` | `false` | `config.rs:520` | `api/mod.rs:81` |
| `RELAY_OPERATOR_PUBKEYS` | CSV of 64-hex | empty (⇒ provisioning disabled) | `config.rs:555` | `api/operator.rs:91`, `handlers/community_provisioning.rs:261`. Invalid entry ⇒ **hard error, entry echoed** (`config.rs:566-571`); deduped |
| `RELAY_OPERATOR_API_ORIGIN` | http(s) origin, no creds/path/query/fragment | `None` | `config.rs:549` | `api/operator.rs:70`. **Required** when `RELAY_OPERATOR_PUBKEYS` is non-empty (`config.rs:577-582`) |

##### 1e. Web / CORS / admin / policy

| Var | Type | Default | Read | Consumed |
|-----|------|---------|------|----------|
| `BUZZ_CORS_ORIGINS` | CSV | empty ⇒ **permissive CORS** | `config.rs:595` | `router.rs:190` → `router.rs:409-432` |
| `BUZZ_WEB_DIR` | `Option<PathBuf>` | `None` | `config.rs:843` | `router.rs:145-183`, `router.rs:320-330`. Must contain `index.html` ⇒ else hard error (`config.rs:850-856`) |
| `BUZZ_SERVE_GIT_WEB_GUI` | `=="true"\|\|=="1"` | `false` | `config.rs:848` | `router.rs:143`, `router.rs:320` |
| `BUZZ_ADMIN_HOST` | exact authority | `None` ⇒ admin surface absent | `config.rs:814` | `router.rs:47`, `api/admin::is_admin_host`. Rejects `/ \ @` (`config.rs:819-823`) |
| `BUZZ_ADMIN_WEB_DIR` | `Option<PathBuf>` | `None` | `config.rs:826` | `router.rs:49-52`, `router.rs:265-271`. Must contain `index.html` (`config.rs:830-836`) |
| `BUZZ_TERMS_OF_SERVICE_MARKDOWN` | `Option<String>` ≤256 KiB | `None` | `config.rs:790` (helper `:776`) | `api/invites::join_policy*` via `config.join_policy` |
| `BUZZ_PRIVACY_POLICY_MARKDOWN` | `Option<String>` ≤256 KiB | `None` | `config.rs:791` | same |
| `BUZZ_AGE_ATTESTATION_REQUIRED` | `parse_bool` strict | `false` | `config.rs:792` (helper `:379`) | same; invalid ⇒ hard error (tested `config.rs:1074-1090`) |

##### 1f. Media

| Var | Type | Default | Read | Consumed |
|-----|------|---------|------|----------|
| `BUZZ_S3_ENDPOINT` | `String` | `http://localhost:9000` | `config.rs:620` | `state.rs:695` (git store), `buzz_media::MediaStorage` |
| `BUZZ_S3_ACCESS_KEY` | `String` | `buzz_dev` | `config.rs:622` | same |
| `BUZZ_S3_SECRET_KEY` | `String` | `buzz_dev_secret` | `config.rs:624` | same |
| `BUZZ_S3_BUCKET` | `String` | `buzz-media` | `config.rs:626` | same |
| `BUZZ_S3_REGION` | `String` | falls back to `AWS_REGION`, then `us-east-1` | `config.rs:627` | same |
| `AWS_REGION` | `String` | — (fallback only) | `config.rs:628` | region resolution |
| `BUZZ_MAX_IMAGE_BYTES` | `u64` | `52_428_800` (50 MB) | `config.rs:630` | `router.rs:34` (body limit), media validation |
| `BUZZ_MAX_GIF_BYTES` | `u64` | `10_485_760` (10 MB) | `config.rs:634` | media validation |
| `BUZZ_MAX_VIDEO_BYTES` | `u64` | `524_288_000` (500 MB) | `config.rs:638` | `router.rs:35`, media validation |
| `BUZZ_MAX_FILE_BYTES` | `u64` | `104_857_600` (100 MB) | `config.rs:642` | media validation |
| `BUZZ_MEDIA_BASE_URL` | `String` | `http://localhost:3000/media` | `config.rs:646` | media URL generation |
| `BUZZ_MEDIA_UPLOAD_RECORDS` | `=="true"\|\|=="1"` | `false` | `config.rs:651` | `MediaConfig::validate` coherence check (`main.rs:417`) |
| `BUZZ_MEDIA_UPLOAD_IP_HEADER` | `Option<String>` lowercased | `None` | `config.rs:654` | same |
| `BUZZ_MEDIA_UPLOAD_PORT_HEADER` | `Option<String>` lowercased | `None` | `config.rs:658` | same |
| `BUZZ_MEDIA_MAX_CONCURRENT_UPLOADS` | `usize` >0 | `8` | `config.rs:664` | `state.rs:693` → `media_upload_semaphore` |
| `BUZZ_MEDIA_MAX_CONCURRENT_UPLOADS_PER_PUBKEY` | `u32` >0 | `2`, clamped to the above | `config.rs:670` | `api/media.rs` |
| `BUZZ_MEDIA_UPLOADS_PER_MINUTE` | `u32` >0 | `30` | `config.rs:676` | `api/media.rs` |
| `BUZZ_REQUIRE_MEDIA_GET_AUTH` | `true\|1\|yes\|on` (ci) | `false` | `config.rs:682` | `api/media.rs` GET/HEAD gate |

##### 1g. Git

| Var | Type | Default | Read | Consumed |
|-----|------|---------|------|----------|
| `BUZZ_GIT_REPO_PATH` | `PathBuf` (created) | `./repos` | `config.rs:705` | `api/git/*`; `create_dir_all` failure ⇒ hard error (`config.rs:384-397`) |
| `BUZZ_GIT_PACK_CACHE_PATH` | `PathBuf` (created) | `<git_repo_path>/.pack-cache` | `config.rs:709` | `state.rs:704` |
| `BUZZ_GIT_MAX_PACK_BYTES` | `u64` | `524_288_000` (500 MB) | `config.rs:713` | `api/git/transport.rs:1757` (body limit) |
| `BUZZ_GIT_MAX_REPO_BYTES` | `u64` | `git_max_pack_bytes * 2` (1 GB) | `config.rs:717` | `api/git/hydrate.rs` |
| `BUZZ_GIT_PACK_CACHE_MAX_BYTES` | `u64` | `git_max_repo_bytes * 5` (5 GB) | `config.rs:721` | `state.rs:705` |
| `BUZZ_GIT_PACK_CACHE_MAX_CONCURRENT_POPULATIONS` | `usize` >0 | `2` | `config.rs:726` | `state.rs:706` |
| `BUZZ_GIT_MAX_REPOS_PER_PUBKEY` | `u32` | `100` | `config.rs:731` | `handlers/side_effects.rs:2487` |
| `BUZZ_GIT_MAX_CONCURRENT_OPS` | `usize` | `20` | `config.rs:735` | `state.rs:692` → `git_semaphore` |
| `BUZZ_GIT_HOOK_HMAC_SECRET` | `String` ≥32 when set | random 32-byte hex **per process** | `config.rs:739`, re-read `config.rs:865` | `api/git/policy.rs`, `api/git/hook.rs` |

##### 1h. Push (NIP-PL)

| Var | Type | Default | Read | Consumed |
|-----|------|---------|------|----------|
| `BUZZ_PUSH_EXECUTOR_KEY_ID` | `String` 1..=64 | `relay-v1` | `config.rs:746` | `nip11.rs:241`. Empty or >64 ⇒ hard error (`config.rs:747-751`) |
| `BUZZ_PUSH_GATEWAY_DELIVERY_URL` | exact `https://…/v1/deliveries/apns` | **`https://push.buzz.xyz/v1/deliveries/apns`** (`config.rs:332`) — explicit empty string disables | `config.rs:752` | `main.rs:686` (worker gate), `nip11.rs:244/251`, `push_runtime.rs` |
| `BUZZ_PUSH_GATEWAY_TIMEOUT_MS` | `u64` in `100..=10000` | `2000` | `config.rs:759` | `push_runtime.rs:314`. Out of range ⇒ hard error (`config.rs:763-771`) |

##### 1i. Mesh

| Var | Grammar | Default | Read | Consumed |
|-----|---------|---------|------|----------|
| `BUZZ_MESH` | `on` (ci) \| `true` \| `1` | off | `config.rs:498` | `mesh_boot::boot_mesh` (`main.rs:442`) |
| `BUZZ_MESH_BIND_ADDR` | `SocketAddr` | `0.0.0.0:3478` | `config.rs:501` | `boot_mesh`. Invalid ⇒ hard error (`config.rs:502-506`) |
| `BUZZ_MESH_DEMO_ECHO` | `on` (ci) \| `true` \| `1` | off | `config.rs:516` | `main.rs:456` (`wire_consumers`), `api/mesh_demo.rs` |

##### 1j. Misc

| Var | Type | Default | Read | Consumed |
|-----|------|---------|------|----------|
| `BUZZ_HUDDLE_AUDIO_AVAILABLE` | `!(=="false"\|\|=="0")` (**inverted**) | `true` | `config.rs:489` | `audio/handler.rs:357` |
| `BUZZ_AUDIT_ENABLED` | `parse_bool` strict | `true` | `config.rs:793` | `main.rs:130/139/323`, `state.rs:763`. Invalid ⇒ hard error (tested `config.rs:1056-1072`) |
| `BUZZ_EPHEMERAL_TTL_OVERRIDE` | `i32` >0 | `None` | `config.rs:691` | `handlers/ingest.rs:2099`, `handlers/side_effects.rs:1681`. Emits a startup `warn` when set (`config.rs:696-702`) |

##### 1k. Rate limits (7 vars → `buzz_auth::RateLimitConfig`)

Read by `rate_limit_config_from_env` (`config.rs:284-317`) via `positive_u64_from_env` (`config.rs:270-282`). **All 7 hard-error on zero, junk, or non-Unicode** — no silent fallback.

| Var | Read | `RateLimitConfig` field | Production consumer |
|-----|------|-------------------------|---------------------|
| `BUZZ_RATE_LIMIT_HUMAN_MESSAGES_PER_MIN` | `config.rs:288` | `human_messages_per_min` | `connection.rs:636` |
| `BUZZ_RATE_LIMIT_HUMAN_API_CALLS_PER_MIN` | `config.rs:292` | `human_api_calls_per_min` | `api/bridge.rs:29` |
| `BUZZ_RATE_LIMIT_HUMAN_WS_EVENTS_PER_SEC` | `config.rs:296` | `human_ws_events_per_sec` | `connection.rs:614` |
| `BUZZ_RATE_LIMIT_AGENT_STANDARD_MESSAGES_PER_MIN` | `config.rs:300` | `agent_standard_messages_per_min` | `connection.rs:634` |
| **`BUZZ_RATE_LIMIT_AGENT_STANDARD_API_CALLS_PER_MIN`** | `config.rs:304` | `agent_standard_api_calls_per_min` | **NONE — parsed, validated, stored, never read** |
| **`BUZZ_RATE_LIMIT_AGENT_ELEVATED_MESSAGES_PER_MIN`** | `config.rs:308` | `agent_elevated_messages_per_min` | **NONE** |
| **`BUZZ_RATE_LIMIT_AGENT_PLATFORM_MESSAGES_PER_MIN`** | `config.rs:312` | `agent_platform_messages_per_min` | **NONE** |

Confirmed by workspace grep across `crates/**` and `desktop/src-tauri/**`: the only occurrences of the three field names outside `config.rs` are their declarations (`crates/buzz-auth/src/rate_limit.rs:101,104,107`) and their default assignments (`:139,140,141`). No reader. **3 of 7 rate-limit vars are dead** — an operator setting them gets no effect and no warning. Two of the three name a tier ("elevated", "platform") for which no enforcement path exists at all.

---

#### 2. `main.rs` — 15 env vars read outside `Config`

These bypass `Config` entirely, so they are invisible to `Config`'s tests, invisible to the startup "Config loaded" log (`main.rs:128-136`), and not covered by any of `config.rs`'s validation helpers.

| Var | Type | Default | Read | Consumed | Invalid-value policy |
|-----|------|---------|------|----------|----------------------|
| `RUST_LOG` | filter directives | none (plus forced `buzz_relay=info`) | `main.rs:111` (`EnvFilter::from_default_env`) | log filtering | tracing-subscriber default |
| `BUZZ_AUTO_MIGRATE` | `true\|1\|yes\|on` trimmed/ci | off | `main.rs:162` (helper `main.rs:25-33`) | `main.rs:163-172` | anything else ⇒ off |
| `BUZZ_RECONCILE_CHANNELS` | **presence only** (`is_ok()`) | absent | `main.rs:549` | `main.rs:549-590` | any value incl. empty ⇒ on |
| `BUZZ_GIT_CONFORMANCE_PROBE` | `!= "false"` | on | `main.rs:469` | `main.rs:466-503` | anything but `false` ⇒ on |
| `BUZZ_GIT_PROBE_WRITERS` | `usize` | `32` | `main.rs:473` | `main.rs:485` | silent default |
| `BUZZ_GIT_PROBE_ROUNDS` | `usize` | `3` | `main.rs:477` | `main.rs:486` | silent default |
| `BUZZ_NIP43_RECONCILE_INTERVAL_SECS` | `u64` `.max(1)` | `60` | `main.rs:517` | `main.rs:523` | silent default |
| `BUZZ_REAPER_INTERVAL_SECS` | `u64` | `60` | `main.rs:609` | `main.rs:619` | silent default; **no floor — `0` yields a busy loop** |
| `SPROUT_REMINDER_SCHEDULER_INTERVAL_SECS` | `u64` | `10` | `main.rs:701` | `main.rs:721` | silent default; **no floor** |
| `SPROUT_REMINDER_SCHEDULER_BATCH_LIMIT` | `i64` | `100` | `main.rs:705` | `main.rs:727` | silent default |
| `BUZZ_COMMUNITY_REVALIDATE_INTERVAL_SECS` | `u64` `.clamp(1,300)` | `30` | `main.rs:882` | `main.rs:889` | silent default + clamp |
| `BUZZ_POOL_METRICS_INTERVAL_SECS` | `u64` `.max(1)` | `10` | `main.rs:945` | `main.rs:951` | silent default; floored because `interval` panics on zero (`main.rs:949`) |
| `BUZZ_USAGE_METRICS_INTERVAL_SECS` | `u64` `.max(5)` | `300` | `main.rs:1258` | `main.rs:133`, `main.rs:1013/1021` | silent default + floor |
| `BUZZ_USAGE_METRICS_IDLE_TIMEOUT_SECS` | `u64` | `900`, raised to `interval*3` | `main.rs:1267` | `main.rs:138` → `metrics.rs:77` | silent default |
| `BUZZ_USAGE_METRICS_PER_COMMUNITY` | `""\|all\|off` | `all` | `main.rs:58` | `main.rs:1002`, `main.rs:1335/1509/1467` | unknown ⇒ `warn` + `all` (`main.rs:66-72`) |

Two of these have **no lower bound** (`BUZZ_REAPER_INTERVAL_SECS`, `SPROUT_REMINDER_SCHEDULER_INTERVAL_SECS`), unlike the three neighbours that do — a `0` turns the corresponding task into a `sleep(0)` hot loop hammering Postgres. The two adjacent pollers were explicitly floored for exactly this reason (`main.rs:949`, `main.rs:1261`).

---

#### 3. `nip11.rs` and `telemetry.rs`

| Var | Type | Default | Read | Consumed | Note |
|-----|------|---------|------|----------|------|
| `SPROUT_MAX_NOT_BEFORE_DELTA` | `u64` | `31_536_000` (1 y) | `nip11.rs:97` | `nip11.rs:113` advertised `max_not_before_delta` | **read on every NIP-11 request**, not cached in `Config`; legacy `SPROUT_` prefix |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | presence | unset ⇒ tracing disabled | `telemetry.rs:81` | `telemetry.rs:79-90` | value itself is consumed by the OTLP SDK, not by this code |
| `OTEL_SERVICE_NAME` | non-empty `String` | `buzz-relay` | `telemetry.rs:50` | `telemetry.rs:56-59` | empty string falls back (tested `telemetry.rs:157-171`) |
| `OTEL_RESOURCE_ATTRIBUTES` | OTEL spec | — | `telemetry.rs:53` (`EnvResourceDetector::new()`) | overlaid **last**, so a `service.name` there wins over `OTEL_SERVICE_NAME` (`telemetry.rs:32`, `:48-52`) | |

**Documented-but-unimplemented (verified delta):** `telemetry.rs:23-24` lists `OTEL_TRACES_SAMPLER` (documented default `parentbased_always_on`) and `OTEL_TRACES_SAMPLER_ARG` as "Standard OTEL env vars honoured". No code in this crate reads either — `try_init_tracer` (`telemetry.rs:79-90`) and `classify_exporter_result` (`telemetry.rs:96-113`) build the provider with only `.with_resource(...)` and `.with_batch_exporter(...)`, never `.with_sampler(...)`. Whether they take effect therefore depends entirely on `opentelemetry_sdk` 0.32 internals (`Cargo.toml:78` → workspace `Cargo.toml:78`). I could not verify the SDK's behaviour offline; treat the doc claim as unproven and the sampler as effectively unconfigurable from this crate.

---

#### 4. Consumed by `main.rs` but declared in `storage_sweep.rs`

| Var | Type | Default | Read | Consumed |
|-----|------|---------|------|----------|
| `BUZZ_STORAGE_SWEEP_INTERVAL_SECS` | `u64` `.max(60)` | `3600` | `storage_sweep.rs:56` | `main.rs:1454` → `main.rs:1457` |
| `BUZZ_STORAGE_SWEEP_TIMEOUT_SECS` | `u64` | `120` | `storage_sweep.rs:61` | `main.rs:1458` |
| `BUZZ_STORAGE_SWEEP_MAX_OBJECTS` | `u64` | `1_000_000` | `storage_sweep.rs:65` | `main.rs:1454`, `main.rs:1462` |
| `BUZZ_STORAGE_METRICS` | `off` (trimmed, ci) disables; anything else incl. unset enables | enabled | `storage_sweep.rs:69` | `main.rs:1452` |

These live in a function-local `OnceLock` (`main.rs:1444-1451`) rather than on `Config`/`AppState`, with an explicit rationale: localized feature config, read once on the first leader tick, stable for the process lifetime.

---

#### 5. Config read outside boot (2 vars)

Everything else is read once. Two are not:

| Var | Frequency | Cite | Consequence |
|-----|-----------|------|-------------|
| `SPROUT_MAX_NOT_BEFORE_DELTA` | **every NIP-11 request** (`GET /`, `GET /info`) | `nip11.rs:96-100` | a `std::env::var` + parse per unauthenticated request; inconsistent with every other advertised limit, which comes from `Config` |
| `BUZZ_STORAGE_*` / `BUZZ_STORAGE_METRICS` | first leader tick only, then cached | `main.rs:1444-1451` | documented and intentional |

---

#### 6. Parsed-but-never-consumed summary (the known defect class)

| Item | Where parsed | Reader |
|------|--------------|--------|
| `BUZZ_RATE_LIMIT_AGENT_STANDARD_API_CALLS_PER_MIN` | `config.rs:303-306` | **none** |
| `BUZZ_RATE_LIMIT_AGENT_ELEVATED_MESSAGES_PER_MIN` | `config.rs:307-310` | **none** |
| `BUZZ_RATE_LIMIT_AGENT_PLATFORM_MESSAGES_PER_MIN` | `config.rs:311-314` | **none** |

All three are fields of `buzz_auth::RateLimitConfig` nested inside `Config.auth` (`config.rs:82`), so the dead surface is *inside* a live field — which is why a top-level field audit misses it.

At the top level, every one of `Config`'s **51** fields (`config.rs:53-262`) has at least one production consumer — verified individually by grepping each field name across `crates/**` and `desktop/src-tauri/**` and discarding `#[cfg(test)]` hits. Two are consumed only through a continuation-line access that a naive `config.X` grep misses: `relay_operator_api_origin` (`api/operator.rs:70`) and `relay_operator_pubkeys` (`api/operator.rs:91`, `handlers/community_provisioning.rs:261`); both are live.

The near-misses worth recording:

| Field | Only production consumer | Note |
|-------|--------------------------|------|
| `pairing_relay_url` | `nip11.rs:243` | advertisement only — the relay never dials it |
| `pubkey_allowlist_enabled` | `handlers/auth.rs:187` | single site |
| `huddle_audio_available` | `audio/handler.rs:357` | single site |
| `allow_nip_oa_auth` | `api/mod.rs:81` | single site |
| `git_max_repos_per_pubkey` | `handlers/side_effects.rs:2487` | single site |
| `push_gateway_timeout` | `push_runtime.rs:314` | single site |
| `started_at` (AppState) | `router.rs:388` | single site (`/_status`) |

---

#### 7. `.env.example` coverage — 5 of 93

`.env.example` (233 lines) declares only 12 variable names, of which just **5 are read by this module**: `BUZZ_BIND_ADDR`, `DATABASE_URL`, `REDIS_URL`, `RELAY_URL`, `RUST_LOG`. The rest are `PG*` helpers for local tooling plus:

**Two dead entries**: `TYPESENSE_API_KEY` (`.env.example:40`) and `TYPESENSE_URL` (`.env.example:41`). No Rust code in the workspace reads either — search moved to Postgres FTS (`main.rs:369-372`, `handlers/event.rs:482-483` confirm "The old Typesense `index_event` worker and its `search_index_tx` mpsc are gone"). The only remaining `Typesense` mentions are historical comments (`crates/buzz-search/src/query.rs:20,46`).

**88 of 93 vars are undocumented in `.env.example`**, including every security-relevant switch: `BUZZ_REQUIRE_AUTH_TOKEN`, `BUZZ_REQUIRE_RELAY_MEMBERSHIP`, `BUZZ_PUBKEY_ALLOWLIST`, `BUZZ_REQUIRE_MEDIA_GET_AUTH`, `BUZZ_CORS_ORIGINS`, `BUZZ_RELAY_PRIVATE_KEY`, `RELAY_OWNER_PUBKEY`, `RELAY_OPERATOR_PUBKEYS`, `RELAY_OPERATOR_API_ORIGIN`, `BUZZ_AUDIT_ENABLED`, `BUZZ_AUTO_MIGRATE`, `BUZZ_ADMIN_HOST`. An operator following `AGENTS.md`'s `cp .env.example .env` gets a relay with every permissive default active and no indication that the switches exist.
