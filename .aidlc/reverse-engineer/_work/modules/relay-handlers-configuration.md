## Module: buzz-relay — moderation, admin & background workers (`crates/buzz-relay/src`)
### Aspect: Configuration

---

#### 1. Environment variables read *directly* by the 12 assigned files

Exactly **4** env vars are read inside this module's own code, all in `storage_sweep.rs`. Every other configuration input arrives pre-parsed on `state.config`.

| Variable | Parse | Default | Floor/validation | Read at | Consumed at |
|---|---|---|---|---|---|
| `BUZZ_STORAGE_SWEEP_INTERVAL_SECS` | `u64`, unparseable ⇒ default | `3600` | `.max(60)` — a misconfigured value cannot become a listing busy-loop (`:36-38`) | `storage_sweep.rs:56-60` | `main.rs:1462` → `maybe_spawn_sweep(interval)` → `should_spawn` (`storage_sweep.rs:105-127`) |
| `BUZZ_STORAGE_SWEEP_TIMEOUT_SECS` | `u64`, unparseable ⇒ default | `120` | none | `storage_sweep.rs:61-64` | `main.rs:1463` → `tokio::time::timeout` (`storage_sweep.rs:249-252`) |
| `BUZZ_STORAGE_SWEEP_MAX_OBJECTS` | `u64`, unparseable ⇒ default | `1_000_000` | none | `storage_sweep.rs:65-68` | `main.rs:1458`, `:1464-1467` → `buzz_media::fold_bucket_listing` cap check (`bucket_index.rs:392-395`) |
| `BUZZ_STORAGE_METRICS` | trimmed + lowercased; `"off"` ⇒ disabled, anything else **including unset** ⇒ enabled | enabled | none | `storage_sweep.rs:69-73` | `main.rs:1454-1456` — early return suppressing the sweep *and every* storage gauge including health ones |

##### 1.1 Read timing and caching

All four are read **once**, on the first leader tick, behind a function-local `OnceLock` in `main.rs`:

```
static SWEEP_CONFIG: OnceLock<StorageSweepConfig> = OnceLock::new();   // main.rs:1447-1448
let config = *SWEEP_CONFIG.get_or_init(StorageSweepConfig::from_env);   // main.rs:1453
```

The `OnceLock` is deliberately function-local rather than on `Config`/`AppState`, with the in-code rationale that this is localized feature config with a single consumer, stable for the process lifetime because env is immutable (`main.rs:1449-1452`). Consequence: these four vars are **not** validated at boot — a typo is silently absorbed into the default, and nothing is discoverable until the first leader tick.

`StorageSweepConfig` is `Copy` (`storage_sweep.rs:33`), so the deref-copy at `main.rs:1453` is cheap.

##### 1.2 Silent-default behaviour

All four use the `.ok().and_then(parse).unwrap_or(default)` idiom, so `BUZZ_STORAGE_SWEEP_TIMEOUT_SECS=abc` yields 120 with **no warning**. This is the opposite of the crate's `Config::from_env` convention, where an invalid value is a hard `ConfigError` (`config.rs:764-771` for `BUZZ_PUSH_GATEWAY_TIMEOUT_MS`, `:747-751` for `BUZZ_PUSH_EXECUTOR_KEY_ID`, `:563-568` for `RELAY_OPERATOR_PUBKEYS`). Two config subsystems, two error philosophies.

##### 1.3 Kill-switch semantics

Only the exact (trimmed, case-insensitive) string `off` disables (`storage_sweep.rs:69-73`). `BUZZ_STORAGE_METRICS=false`, `=0`, `=disabled` all **enable** the feature. This differs from `config.rs`'s `parse_bool` helper, which accepts `true|1|on` / `false|0|off|""` (`config.rs:364-378`) — and `storage_sweep` does not use it. A unit test pins the asymmetry (`storage_sweep.rs:379-397`, helper `:395-397`).

Documented for operators at `deploy/charts/buzz/values.yaml:308-313`.

---

#### 2. `Config` fields consumed by this module

Read from `state.config`, parsed at boot in `crates/buzz-relay/src/config.rs`.

| Field | Env var | Default | Boot validation | Consumed in this module |
|---|---|---|---|---|
| `relay_operator_pubkeys: Vec<String>` | `RELAY_OPERATOR_PUBKEYS` (comma-separated 64-hex, deduped, lowercased) | **empty ⇒ provisioning disabled** (`config.rs:962-963`) | each entry must be valid 64-hex (`config.rs:563-568`); **must be paired with `RELAY_OPERATOR_API_ORIGIN` or boot fails** (`config.rs:577-580`) | `community_provisioning.rs:258-266` — the operator gate |
| `require_relay_membership: bool` | `REQUIRE_RELAY_MEMBERSHIP` | — | — | `community_provisioning.rs:215` — gates NIP-43 snapshot publication; also overridden to `false` in two test helpers (`workflow_sink.rs:577`) |
| `push_gateway_delivery_url: Option<url::Url>` | `BUZZ_PUSH_GATEWAY_DELIVERY_URL` | **`https://push.buzz.xyz/v1/deliveries/apns`** (`config.rs:339`, `:755-758`) | exact HTTPS + host + no userinfo + path `/v1/deliveries/apns` + no query + no fragment (`config.rs:341-361`) — else boot error | `push_lease.rs:480-482` (gates lease acceptance), `push_runtime.rs:422-424` (delivery target), `main.rs:686` (gates both worker spawns) |
| `push_gateway_timeout: Duration` | `BUZZ_PUSH_GATEWAY_TIMEOUT_MS` | `2000` ms (`config.rs:771`) | integer in `100..=10_000`, else boot error (`config.rs:759-770`) | `push_runtime.rs:314` — whole-request `reqwest` timeout |
| `push_executor_key_id: String` | `BUZZ_PUSH_EXECUTOR_KEY_ID` | `"relay-v1"` (`config.rs:745-746`) | 1..=64 bytes, else boot error (`config.rs:747-751`) | `push_lease.rs:484-486` — must equal the lease's `exec` tag |
| `relay_url: String` | `RELAY_URL` | — | — | `push_lease.rs:494` → `canonical_origin` scheme derivation (`:585-596`); `product_feedback.rs:29` → tenant media base |
| `media: MediaConfig` | `BUZZ_S3_*`, `BUZZ_MEDIA_*` | see §3 | — | via `state.media_storage` in `report.rs:69`, `product_feedback.rs:35`, and the sweep's `list_page` (`main.rs:1462-1470`) |
| `database_url`, `redis_url`, `auth` | — | — | — | **test-only** construction in `identity_archive.rs:441-448` and `workflow_sink.rs:576-596` |

##### 2.1 The push-gateway default is the most consequential setting in this module

Because unset falls back to a concrete third-party URL rather than `None`:

| Consequence | Site |
|---|---|
| Lease acceptance is **on** by default | `push_lease.rs:480-482` returns `push not supported` only when the URL is `None` |
| The matcher **and** delivery worker are spawned by default | `main.rs:686-692` — "enabled as one unit" |
| NIP-11 advertises push support by default | `nip11.rs:208-209` |
| Outbound HTTPS to `push.buzz.xyz` is attempted by default | `push_runtime.rs:422-428` |

Disabling requires setting the variable to an **empty** string (`config.rs:753` matches `Ok(raw) if raw.trim().is_empty() => None`). Unsetting it does the opposite of what an operator would expect. A unit test pins both behaviours (`config.rs:1189-1212`).

---

#### 3. Indirect configuration (consumed via `state`, not read here)

| Setting | Env var | Default | Why this module cares |
|---|---|---|---|
| Usage-tick cadence | `BUZZ_USAGE_METRICS_INTERVAL_SECS` | `300`, `.max(5)` (`main.rs:1257-1262`) | **This is the storage sweep's real retry cadence after a failure** — `should_spawn` returns `true` unconditionally on `!ok` (`storage_sweep.rs:110`), so a permanently failing sweep retries every tick, not every `interval` (documented `:89-103`) |
| Gauge idle timeout | `BUZZ_USAGE_METRICS_IDLE_TIMEOUT_SECS` | `900` (`main.rs:1264-1269`) | the reason `emit_storage_metrics` explicitly zeroes disappearing per-community series instead of relying on idle eviction (`storage_sweep.rs:284-291`) |
| S3 endpoint / credentials / bucket / region | `BUZZ_S3_ENDPOINT`, `BUZZ_S3_ACCESS_KEY`, `BUZZ_S3_SECRET_KEY`, `BUZZ_S3_BUCKET` (default `buzz-media`), `BUZZ_S3_REGION` (falls back to `AWS_REGION`) | `config.rs:620-629` | the sweep lists the whole bucket with these credentials — needs `s3:ListBucket` added to the read-write media key |
| Relay keypair | `RELAY_PRIVATE_KEY` (random if unset, `config.rs:740-744`) | — | signs notice DMs (`moderation_notices.rs:165`, `:198`), workflow messages (`workflow_sink.rs:304`), the outbound NIP-98 header (`push_runtime.rs:562`), and decrypts lease plaintext (`push_lease.rs:488`) |
| Relay owner bootstrap | `RELAY_OWNER_PUBKEY` | — | the **only** supported path to owner promotion, since 9030 and 9032 both refuse `role=owner` (`relay_admin.rs:184-186`, `:300-302`); referenced in-code at `relay_admin.rs:298` and `community_provisioning.rs:38`, `:245` |
| Operator API origin | `RELAY_OPERATOR_API_ORIGIN` | — | NIP-98 URL binding for `POST /operator/communities` (`api/operator.rs:155-163`); required whenever the allowlist is non-empty (`config.rs:577-580`) |
| `BUZZ_RELAY_BASE_URL`, `BUZZ_API_TOKEN`, `BUZZ_RELAY_PUBKEY` | — | `http://localhost:3000` | read by `buzz-workflow`'s `add_reaction_impl` (`buzz-workflow/src/executor.rs:886`, `:895`, `:897`), **not** by `workflow_sink`. That action targets an unregistered route, so these three vars configure a permanently broken path |

---

#### 4. Hard-coded constants that behave like configuration

None of these are env-tunable. They are policy that requires a rebuild to change.

##### 4.1 Freshness / skew

| Constant | Value | Site |
|---|---|---|
| `MAX_COMMAND_SKEW_SECS` (9040–9044) | `120` | `moderation_commands.rs:81` |
| relay-admin skew (bare literal) | `120` | `relay_admin.rs:125` |
| identity-archive skew (bare literal) | `120` | `identity_archive.rs:147` |
| `ALLOWED_SKEW` (lease expiration) | `120` | `push_lease.rs:476` |

Four independent declarations of the same policy value; only one is named.

##### 4.2 Push-lease policy

| Constant | Value | Site |
|---|---|---|
| `MAX_LEASE_TTL` | 30 days (`30*24*60*60`) | `push_lease.rs:475` |
| `MAX_CONTENT` (ciphertext) | 65,536 bytes | `:477` |
| `MAX_PLAINTEXT` | 32,768 bytes | `:478` |
| `MAX_ACTIVE_LEASES` per author | 16 | `:479` |
| `PUSH_KINDS` | `[7, 9, 1059, 40007, 46010]` | `:15` — pinned to `migrations/0018_push_match_queue.sql` by a test (`:696-710`) |
| `URGENT_KINDS` | `[]` (empty) | `:16` — advertised in NIP-11 (`nip11.rs:209`) |
| `MAX_SAFE_JSON_INTEGER` | `2^53 - 1` | `:21` |
| app profiles | `buzz-ios-production`, `buzz-ios-sandbox` (both `apns`) | `:504-512` |
| supported classes | `silent`, `default`, `time_sensitive` | `:509` |
| `max_subscriptions` / `max_kinds` | 16 / 16 | `:513-514` |
| `max_authors` / `max_h` / `max_tag_values` / `max_ignore` | 20 / 50 / 20 / 8 | `:515-518` |
| `max_endpoint_len` / `max_string_len` | 4096 / 512 | `:519-520` |
| `d`-tag length bound | 1..=64 | `:139-141` |

##### 4.3 Push runtime

| Constant | Value | Site |
|---|---|---|
| `CLAIM_SECS` | 30 | `push_runtime.rs:15` |
| `EVENT_USEFUL_SECS` | 3600 | `:16` |
| `MAX_ATTEMPTS` (delivery) | 8 | `:17` |
| `MATCH_BATCH_LIMIT` | 64 | `:20` |
| `IDLE_POLL_FLOOR` / `IDLE_POLL_CEILING` (matcher) | 250 ms / 2 s | `:24-25` |
| delivery-worker idle floor | 500 ms (inline) | `:317`, `:341` |
| `REAP_INTERVAL` | 30 s | `:29` |
| wake claim batch size | 16 (inline) | `:325` |
| default retry delay | 2 s (inline, 4 sites) | `:392`, `:481`, `:486`, `:498` |
| backoff shift clamp | `2^6` | `:544` |
| `MAX_MATCH_ATTEMPTS` | 8 (`buzz-db/src/push.rs:70`) | referenced `:172`, `:190` |
| batch-retry delay | 2 s | `:141`, `:210` |
| claim-error sleep | 2 s | `:84` |

##### 4.4 Moderation / admin / feedback limits

| Constant | Value | Site |
|---|---|---|
| `MAX_WORKSPACE_ICON_URL_LEN` | 2048 | `relay_admin.rs:60` |
| `MAX_WORKSPACE_ICON_DATA_URL_LEN` | 98,304 (~72 KB image) | `relay_admin.rs:64` |
| `MAX_HOST_LEN` | 255 (matches `communities.host VARCHAR(255)`) | `community_provisioning.rs:39` |
| DNS total / label length | 253 / 63 | `community_provisioning.rs:151`, `:158` |
| notice idempotency scan limit | 1000 (the query clamp) | `moderation_notices.rs:240`, rationale `:222-226` |
| `MAX_BODY_BYTES` (feedback) | 32,768 | `product_feedback.rs:12` |
| `MAX_TAGS_BYTES` (feedback) | 65,536 | `product_feedback.rs:13` |
| feedback categories | `bug`, `praise`, `needs-work` | `product_feedback.rs:11` |
| `REPORT_TYPES` | 7 NIP-56 values | `report.rs:29-37` |
| report note size | **uncapped** | `report.rs:222-228` |
| `MODERATION_READ_LIMIT` | 500 | `api/bridge.rs:2053` |
| `MODERATION_SOURCE_TAG` | `"moderation_source"` | `moderation_notices.rs:38` |
| `MODERATION_ACTION_CHECK_VOCAB` | 12 values, must match `migrations/0006_moderation.sql` | `buzz-db/src/moderation.rs:104-117`, pinned by `moderation_commands.rs:659-667` |

**Notable gaps:** ban/timeout durations have no configurable maximum (any `expiration` in `i64` range is accepted, `moderation_commands.rs:592-606`); the report note has no size cap while feedback does; the push quotas are the only limits documented as "advertised descriptor policy" yet none of them are actually advertised via NIP-11 except `push_kinds`/`urgent_kinds`.

---

#### 5. Legacy `SPROUT_*` names

**Zero `SPROUT_*` variables are read by any of the 12 assigned files.** Verified by grep.

For completeness, the legacy names surviving elsewhere in `buzz-relay`:

| Variable | Read at | Purpose |
|---|---|---|
| `SPROUT_MAX_NOT_BEFORE_DELTA` | `nip11.rs:97`, `ingest.rs:1299` | advertised in NIP-11 and enforced at ingest |
| `SPROUT_REMINDER_SCHEDULER_INTERVAL_SECS` | `main.rs:701` | NIP-ER reminder scheduler cadence |
| `SPROUT_REMINDER_SCHEDULER_BATCH_LIMIT` | `main.rs:705` | NIP-ER batch size |

All three belong to other module groups. This module group is fully on the `BUZZ_*` prefix.

---

#### 6. Configuration-related findings

| # | Severity | Finding |
|---|---|---|
| C-1 | **HIGH** | `BUZZ_PUSH_GATEWAY_DELIVERY_URL` unset ⇒ `Some("https://push.buzz.xyz/v1/deliveries/apns")` (`config.rs:755-758`), which enables lease acceptance (`push_lease.rs:480`), spawns both background workers (`main.rs:686-692`), advertises push in NIP-11 (`nip11.rs:208`), and attempts outbound HTTPS carrying the device token to a host the operator never chose. Disabling requires an explicitly **empty** value (`config.rs:753`) — unsetting does the opposite of the intuitive thing. |
| C-2 | MEDIUM | The four `BUZZ_STORAGE_*` vars silently absorb invalid values into defaults (`storage_sweep.rs:56-73`), unlike every `Config::from_env` field which hard-fails at boot (`config.rs:747-751`, `:764-771`, `:563-568`). A typo is undiscoverable. |
| C-3 | MEDIUM | Those same four vars are read only on the **first leader tick** behind a function-local `OnceLock` (`main.rs:1447-1453`), so a non-leader pod never validates them and a misconfiguration surfaces late and only on one pod. |
| C-4 | MEDIUM | The storage sweep's effective retry cadence after a failure is `BUZZ_USAGE_METRICS_INTERVAL_SECS` (default 300 s), not `BUZZ_STORAGE_SWEEP_INTERVAL_SECS` (`storage_sweep.rs:105-127`). The coupling is documented in-code (`:89-103`) but the variable name implies otherwise. |
| C-5 | LOW | `BUZZ_STORAGE_METRICS` uses bespoke `off`-only parsing rather than the crate's `parse_bool` (`config.rs:364-378`), so `false`/`0`/`disabled` all **enable** the feature. Pinned by a test that documents the asymmetry (`storage_sweep.rs:379-397`). |
| C-6 | LOW | `BUZZ_STORAGE_SWEEP_TIMEOUT_SECS` and `_MAX_OBJECTS` have no floor or ceiling. `TIMEOUT_SECS=0` yields an immediately-timing-out sweep that respawns on every usage tick forever (`should_spawn` returns `true` on `!ok`, `storage_sweep.rs:110`). Only `INTERVAL_SECS` is floored. |
| C-7 | LOW | Owner promotion has no event-based path — 9030 and 9032 both refuse `owner` (`relay_admin.rs:184-186`, `:300-302`) — so it is config-only via `RELAY_OWNER_PUBKEY` or the operator endpoint. 9030's error message "use kind:9032 to promote to owner" (`:185`) points at a path that also refuses. |
| C-8 | LOW | 12 push-lease quotas plus 13 push-runtime timing constants are compile-time only (`push_lease.rs:475-521`, `push_runtime.rs:15-29`). Tuning batch size, claim lease, retry ceiling, or per-author lease quota for a differently-sized deployment requires a rebuild. |
| C-9 | LOW | `moderation_notices.rs:240` hard-codes the idempotency scan limit to 1000 with the reasoning that "1000 moderation notices to one user in one community is a practical ceiling" (`:225-226`). Beyond that, crash-retry can re-send a duplicate notice. |
| C-10 | INFO | Only `BUZZ_STORAGE_*` appears in `deploy/charts/buzz/values.yaml` (`:308-313`) among this module's direct env surface. The push-gateway vars are not documented in the chart. |
