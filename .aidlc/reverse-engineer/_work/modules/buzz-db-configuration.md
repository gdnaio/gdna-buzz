## Module: buzz-db (`crates/buzz-db`)

### Configuration

#### 1. Environment variables

The crate reads **no environment variables in production code**. Connection
details arrive as a `DbConfig` struct from the caller
(`crates/buzz-db/src/lib.rs:222-234`); the doc comments name the variables the
*relay* is expected to source them from, but the lookup happens outside this
crate:

| Variable | Referenced as | Where documented |
|----------|---------------|------------------|
| `DATABASE_URL` | source of `DbConfig::database_url` | `crates/buzz-db/src/lib.rs:225` |
| `READ_DATABASE_URL` | source of `DbConfig::read_database_url` (e.g. an Aurora `cluster-ro-` endpoint) | `crates/buzz-db/src/lib.rs:227-230` |
| `BUZZ_AUTO_MIGRATE` | relay-side switch that decides whether `Db::migrate` runs; affects fence-probe gating | `crates/buzz-db/src/lib.rs:437-441`; test at `:5946-5951` |

Test-only environment reads (all inside `#[cfg(test)] mod tests`):

| Variable | Precedence | Files |
|----------|-----------|-------|
| `BUZZ_TEST_DATABASE_URL` → `DATABASE_URL` → const default | first match wins | `crates/buzz-db/src/event.rs:1437-1439`, `crates/buzz-db/src/feed.rs:258-260`, `crates/buzz-db/src/git_repo.rs:190-192`, `crates/buzz-db/src/moderation.rs:639-641`, `crates/buzz-db/src/thread.rs:822-824`, `crates/buzz-db/src/push.rs:1270-1272`, `crates/buzz-db/src/workflow.rs:1702-1704`, `crates/buzz-db/src/migration.rs:1028-1030`, `crates/buzz-db/src/relay_members.rs:564-566`, `crates/buzz-db/src/product_feedback.rs:127-129` |
| `BUZZ_TEST_DATABASE_URL` → const default | no `DATABASE_URL` fallback | `crates/buzz-db/src/channel.rs:1794` |
| `TEST_DATABASE_URL` → const default | different name from the above | `crates/buzz-db/src/lib.rs:3936`, `:4655`, `:5228`; `crates/buzz-db/src/replica_fence.rs:515` |
| (none) | hard-coded const only | `crates/buzz-db/src/archived_identities.rs:130-135`, `crates/buzz-db/src/user.rs:406-408`, `crates/buzz-db/src/usage.rs:365-369`, `crates/buzz-db/src/api_token.rs` test setup |

The default test URL literal is
`postgres://buzz:buzz_dev@localhost:5432/buzz`, repeated as a
`const TEST_DB_URL` in 16 test modules.

#### 2. Cargo features

None. `crates/buzz-db/Cargo.toml` has no `[features]` section
(`crates/buzz-db/Cargo.toml:1-25`). All dependency feature selection is inherited
from the workspace table — most relevantly
`sqlx = { version = "0.9", features = ["runtime-tokio", "tls-rustls", "postgres", "uuid", "chrono", "json"] }`
(`Cargo.toml:52-54`). Package metadata (`version`, `edition`, `rust-version`,
`license`, `repository`) is all `.workspace = true`
(`crates/buzz-db/Cargo.toml:3-8`); the workspace pins `edition = "2021"` and
`rust-version = "1.88.0"` (`Cargo.toml:36-37`).

#### 3. Pool tunables (`DbConfig`)

Defined at `crates/buzz-db/src/lib.rs:222-234`, defaults at `:236-249`, applied
at `:387-393`.

| Field | Type | Default | Applied as |
|-------|------|---------|-----------|
| `database_url` | `String` | `postgres://buzz:buzz_dev@localhost:5432/buzz` | writer pool target |
| `read_database_url` | `Option<String>` | `None` (replica routing disabled) | replica pool target |
| `max_connections` | `u32` | `20` | `PgPoolOptions::max_connections` |
| `min_connections` | `u32` | `2` | `PgPoolOptions::min_connections` |
| `acquire_timeout_secs` | `u64` | `3` | `PgPoolOptions::acquire_timeout` |
| `max_lifetime_secs` | `u64` | `1800` | `PgPoolOptions::max_lifetime` |
| `idle_timeout_secs` | `u64` | `600` | `PgPoolOptions::idle_timeout` |

Sizing rationale recorded in the `Default` doc comment: staging measured 51 idle
+ 1 active out of 50, so 20 main + 5 audit = 25 connections per pod lets four
relay pods fit a Postgres `max_connections = 100`
(`crates/buzz-db/src/lib.rs:236-239`). Both pools use the **same** sizing
(`crates/buzz-db/src/lib.rs:361-364`), and `DbPoolStats::max` reports the
configured ceiling for either pool (`:494-511`).

`Db::from_pool` / `Db::from_pools` derive `max_connections` from
`pool.options().get_max_connections()` instead of a config value
(`crates/buzz-db/src/lib.rs:404-427`) and do **not** arm the floor guard, so
callers that build pools themselves are unguarded unless they set the GUC.

#### 4. Session GUCs (runtime configuration in Postgres)

| GUC | Set by | Scope | Effect |
|-----|--------|-------|--------|
| `buzz.created_at_floor` | writer-pool `after_connect` hook, value = `CREATED_AT_FLOOR_SECS` (`crates/buzz-db/src/lib.rs:394-405`) | session (`is_local = false`) | arms the deferred commit-time floor guard from `migrations/0021_created_at_fence_floor.sql:44-68`; unset/blank ⇒ guard is a no-op |
| `buzz.nip_rs_hard_delete` | `set_config(..., true)` inside the NIP-RS replacement transaction (`crates/buzz-db/src/lib.rs:3742-3747`) and inside the purge trigger (`migrations/0011_nip_rs_exact_tag_cardinality.sql:143`) | **transaction-local** | authorises the BEFORE DELETE fence at `migrations/0011_…:45-60`; verified not to leak across commit or rollback by test `crates/buzz-db/src/lib.rs:4194-4218` |

Tests arm the floor guard transaction-locally with
`set_config('buzz.created_at_floor', $1, true)`
(`crates/buzz-db/src/lib.rs:5811-5815`, `:5991-5995`).

#### 5. Compile-time constants

| Constant | Value | File:line | Meaning |
|----------|-------|-----------|---------|
| `replica_fence::CREATED_AT_FLOOR_SECS` | `960` | `crates/buzz-db/src/replica_fence.rs:51` | seconds of `created_at` history the commit-time guard tolerates; must exceed the relay's ±900 s ingest envelope. The GUC value and the fence subtrahend must never diverge. |
| `replica_fence::FENCE_CLOCK_MARGIN_SECS` | `5` | `crates/buzz-db/src/replica_fence.rs:59` | extra margin subtracted from the fence |
| `replica_fence::PROBE_INTERVAL` | `5 s` | `crates/buzz-db/src/replica_fence.rs:62` | probe cadence |
| `replica_fence::FENCE_STALENESS` | `30 s` | `crates/buzz-db/src/replica_fence.rs:66` | fence age after which it is treated as closed |
| `replica_fence::CLOSED` (private) | `i64::MIN` | `crates/buzz-db/src/replica_fence.rs:69` | closed sentinel |
| `feed::FEED_MAX_LIMIT` | `100` | `crates/buzz-db/src/feed.rs:29` | hard cap on every feed query |
| `workflow::LIST_DEFAULT_LIMIT` | `100` | `crates/buzz-db/src/workflow.rs:25` | default workflow list page |
| `workflow::LIST_MAX_LIMIT` | `1000` | `crates/buzz-db/src/workflow.rs:27` | workflow list ceiling |
| `admin_moderation::MAX_PAGE_SIZE` | `200` | `crates/buzz-db/src/admin_moderation.rs:15` | admin page clamp |
| `relay_members::MAX_COMMUNITIES_PER_OWNER` | `3` | `crates/buzz-db/src/relay_members.rs:379` | per-pubkey community ownership cap |
| `push::MAX_MATCH_ATTEMPTS` | `8` | `crates/buzz-db/src/push.rs:70` | matcher-job attempt ceiling |
| `push::PUSH_GATE_LOCK_NAMESPACE` (private) | `"buzz_push_gate:"` | `crates/buzz-db/src/push.rs:21` | advisory-lock key domain; must match `migrations/0023_push_match_gate.sql:34` |
| `push::PUSH_GATE_BACKFILL_SECS` (private) | `120` | `crates/buzz-db/src/push.rs:41` | activation backfill window over `received_at` |
| `event::D_TAG_MAX_LEN` | `1024` | `crates/buzz-db/src/event.rs:124` | documented ceiling, **not enforced in this crate** |
| `event::HUDDLE_LINK_CONTENT_MAX_BYTES` (private) | `512` | `crates/buzz-db/src/event.rs:130` | max candidate content size for huddle-link resolution |
| `event::HUDDLE_LINK_CANDIDATE_LIMIT` (private) | `32` | `crates/buzz-db/src/event.rs:136` | max candidate rows inspected |
| `partition::PARTITIONED_TABLES` (private) | `["events", "delivery_log"]` | `crates/buzz-db/src/partition.rs:12` | DDL allowlist |
| `moderation::MODERATION_ACTION_CHECK_VOCAB` | 12 strings | `crates/buzz-db/src/moderation.rs:104` | mirror of the SQL CHECK |

Hard-coded limits embedded directly in SQL rather than named constants:

| Limit | Value | Where |
|-------|-------|-------|
| Default query limit / clamp | 100 / 1000 | `crates/buzz-db/src/event.rs:334-336` |
| `list_channels`, `get_members`, `get_bot_members`, `get_accessible_channels` | `LIMIT 1000` | `crates/buzz-db/src/channel.rs:696`, `:713`, `:588`, `:903`, `:840` |
| `list_active_tokens` | `LIMIT 1000` | `crates/buzz-db/src/lib.rs:2396` |
| Active-token quota per (community, owner) | `< 10` | `crates/buzz-db/src/api_token.rs:119` |
| Thread/window participant cap | 10 | `crates/buzz-db/src/thread.rs:520`, `:698` |
| `list_dms_for_user` limit cap | 200 | `crates/buzz-db/src/dm.rs:233` |
| `search_users` limit clamp | `[1, 500]` | `crates/buzz-db/src/user.rs:233` |
| `list_workflow_runs` limit cap | 1000 | `crates/buzz-db/src/workflow.rs:824` |
| DM participant bounds | 2–9 | `crates/buzz-db/src/dm.rs:107-117` |
| `get_events_by_ids` debug bound | 500 | `crates/buzz-db/src/event.rs:955` |
| `UsageMetricsLeader::is_live` ping timeout | 5 s | `crates/buzz-db/src/lib.rs:216` |
| Push-eligible kinds (trigger allowlist) | `7, 9, 1059, 40007, 46010` | `migrations/0018_push_match_queue.sql:25`, `migrations/0023_push_match_gate.sql:26`, mirrored in Rust at `crates/buzz-db/src/push.rs:58` |
| FTS excluded / allowlisted kinds | see data-model §6 | `migrations/0001:223`, `0005:31`, `0008:15`, `0014:29` |
| TTL-refresh exempt kind | `9007` | `migrations/0024_event_ttl_refresh_shared_lock.sql:30` |
| NIP-29 discovery kinds soft-deleted together | `39000, 39001, 39002` | `crates/buzz-db/src/lib.rs:3287` |
| Declared partition ranges | 2026-01 … 2026-07 plus `_p_past`/`_p_future` | `migrations/0001_initial_schema.sql:237-252`, `:343-354` |

#### 6. Operational configuration expectations

- `ensure_future_partitions(months_ahead)` is meant to run on startup and monthly
  via cron; the caller chooses `months_ahead`
  (`crates/buzz-db/src/partition.rs:1-4`, `:15`).
- `spawn_fence_probe` must be called **after** the migration decision, otherwise
  an armed GUC with an unapplied migration 0021 could open a fence over an
  unenforced floor (`crates/buzz-db/src/lib.rs:434-448`).
- The probe role needs `pg_monitor`; without it the fence stays closed
  (`crates/buzz-db/src/replica_fence.rs:369-374`).
- Backfills of channel-bearing `events` rows must run on a connection **without**
  `buzz.created_at_floor` and with the replica breaker held closed
  (`migrations/0021_created_at_fence_floor.sql:26-42`).
- `prune_scheduled_workflow_fires_before`'s cutoff must be older than the largest
  supported interval schedule (`crates/buzz-db/src/workflow.rs:589-596`).
- Migration `0004` builds a GIN index on a partitioned parent without
  `CONCURRENTLY` (unsupported there), so brownfield application takes a share
  lock per partition and should run in a deploy window
  (`migrations/0004_events_tags_gin.sql:13-17`).
