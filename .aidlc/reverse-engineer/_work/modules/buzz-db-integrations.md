## Module: buzz-db (`crates/buzz-db`)

### Integrations

#### 1. Dependencies

Declared at `crates/buzz-db/Cargo.toml:11-25`; all versions come from the
workspace table in `Cargo.toml`.

| Crate | Workspace version / features | Used for |
|-------|------------------------------|----------|
| `buzz-core` | path `crates/buzz-core` (`Cargo.toml:124`) | `CommunityId`, `StoredEvent`, `kind::*` predicates and constants, `channel::{ChannelType, ChannelVisibility, MemberRole, canonical_channel_name}` |
| `sqlx` | `0.9`, features `runtime-tokio`, `tls-rustls`, `postgres`, `uuid`, `chrono`, `json` (`Cargo.toml:52-54`) | the only database driver |
| `tokio` | `1` (`Cargo.toml:43`) | `tokio::spawn` for the fence probe, `tokio::time::{interval, timeout}` |
| `serde` / `serde_json` | `1` (`Cargo.toml:64`, `:69`) | JSONB round-trips, `Serialize` on admin/feedback records, event reconstruction |
| `uuid` | `1`, `v4`+`serde` (`Cargo.toml:89`) | all UUID columns and `Uuid::new_v4()` id minting |
| `chrono` | `0.4`, `serde` (`Cargo.toml:90`) | `DateTime<Utc>` ↔ `TIMESTAMPTZ`, month arithmetic in the partition manager |
| `hex` | `0.4` (`Cargo.toml:97`) | pubkey/event-id hex encode for `event_mentions.pubkey_hex` and admin projections |
| `sha2` | `0.11` (`Cargo.toml:96`) | approval-token hashing (`crates/buzz-db/src/workflow.rs:33`), DM participant hash (`crates/buzz-db/src/dm.rs:48`), push advisory-lock key derivation (`crates/buzz-db/src/push.rs:221-230`) |
| `tracing` | `0.1` (`Cargo.toml:74`) | `info!` in the partition manager, `warn!`/`debug!` on best-effort paths |
| `thiserror` | `2` (`Cargo.toml:85`) | `DbError`, `ProbeError` |
| `nostr` | `0.44`, `nip44`+`nip98` (`Cargo.toml:61`) | `nostr::Event` in insert signatures; `EventBuilder`/`Keys`/`Tag`/`Kind` when the crate itself signs the NIP-43 snapshot |

Dev-dependencies: only `tokio` (`crates/buzz-db/Cargo.toml:23-24`). There is no
`[features]` section — the crate has **no cargo features**.

#### 2. Postgres / sqlx specifics

**Query construction.** Runtime `sqlx::query()` / `sqlx::query_as::<_, T>()` /
`sqlx::query_scalar::<_, T>()` only — there is **no** use of `sqlx::query!`,
`query_as!`, or `query_scalar!` anywhere in the crate, so no `.sqlx/` offline
cache is required (design note at `crates/buzz-db/src/lib.rs:10`). Dynamic
SQL uses either `sqlx::QueryBuilder` (`crates/buzz-db/src/event.rs:344`,
`:591`, `crates/buzz-db/src/feed.rs:91`, `crates/buzz-db/src/lib.rs:146`,
`crates/buzz-db/src/channel.rs:1337`, `crates/buzz-db/src/event.rs:836`, `:957`)
or `format!` + `sqlx::AssertSqlSafe` with all values still bound (15 sites; see
`buzz-db-security.md`).

**Pool configuration** (`crates/buzz-db/src/lib.rs:387-407`, defaults at `:236-249`):

| Knob | Default | Notes |
|------|---------|-------|
| `max_connections` | 20 | sized so 4 relay pods × (20 main + 5 audit) fit PG `max_connections=100` |
| `min_connections` | 2 | |
| `acquire_timeout_secs` | 3 | |
| `max_lifetime_secs` | 1800 | |
| `idle_timeout_secs` | 600 | |

The **writer** pool installs an `after_connect` hook that runs
`SELECT set_config('buzz.created_at_floor', $1, false)` with
`replica_fence::CREATED_AT_FLOOR_SECS` on every connection
(`crates/buzz-db/src/lib.rs:394-405`); the replica pool does not
(`arm_floor_guard = false` at `:363`).

**Read-replica handling.** `DbConfig::read_database_url` optionally connects a
second pool with identical sizing (`crates/buzz-db/src/lib.rs:222-234`, `:361-364`).
`Db::read()` returns the replica when configured, else the writer
(`:470-472`). The documented routing contract restricts replica use to
lag-tolerant reads; exactly two call sites route conditionally:
`get_thread_replies` (`:2004-2043`) and `get_channel_window` (`:2063-2077`).
A background probe (`replica_fence::run_probe`, spawned by
`Db::spawn_fence_probe` at `:449-467`) performs a writer→replica LSN handshake
every 5 s.

**Postgres features relied upon**

| Feature | Where |
|---------|-------|
| Declarative range partitioning + partition pruning | `migrations/0001_initial_schema.sql:235`, `:341` |
| Generated `STORED` columns | `events.search_tsv`, `migrations/0001_initial_schema.sql:222` |
| GIN indexes (`tsvector`, `jsonb_path_ops`) | `migrations/0001_initial_schema.sql:278`, `migrations/0004_events_tags_gin.sql:21` |
| Partial and expression indexes | e.g. `migrations/0001_initial_schema.sql:61`, `:102`, `:178`, `:269` |
| Enum types + `::text` / `::enum` casts | `migrations/0001_initial_schema.sql:28-37`; casts e.g. `crates/buzz-db/src/channel.rs:118` |
| `plpgsql` trigger functions, `CREATE CONSTRAINT TRIGGER … DEFERRABLE INITIALLY DEFERRED` | `migrations/0021_created_at_fence_floor.sql:70`, `migrations/0022_event_ttl_refresh.sql:37` |
| Session/transaction GUCs (`current_setting`, `set_config`) | `buzz.created_at_floor`, `buzz.nip_rs_hard_delete` |
| Advisory locks — transaction-scoped (`pg_advisory_xact_lock`, `_shared`) and session-scoped (`pg_try_advisory_lock`) | `crates/buzz-db/src/lib.rs:517-535`, `:3329`, `:3506`, `:3661`; `crates/buzz-db/src/push.rs:27`, `:232`, `:236`; `crates/buzz-db/src/relay_members.rs:420` |
| `hashtextextended(text, 0)` for lock keys derived in SQL | `migrations/0023_push_match_gate.sql:34-35`, `migrations/0024_…:31-32`, `crates/buzz-db/src/channel.rs:1132-1139` |
| `xmax = 0` upsert-winner detection | `crates/buzz-db/src/lib.rs:836` |
| `FOR UPDATE`, `FOR KEY SHARE`, `SKIP LOCKED` | `crates/buzz-db/src/relay_members.rs:434`, `crates/buzz-db/src/push.rs:259`, `:656`, `:864`, `:1044`; `migrations/0009_…:124` |
| `UNNEST(array…) AS t(...)` set-wise DML | `crates/buzz-db/src/push.rs:636-638`, `:673-681`, `:709-712` |
| `DISTINCT ON`, `FILTER (WHERE …)`, `ROW_NUMBER() OVER (PARTITION BY …)`, `json_agg`/`jsonb_build_object`, `jsonb_array_elements`, `array_position` | `crates/buzz-db/src/event.rs:1305`; `crates/buzz-db/src/usage.rs:47-48`; `crates/buzz-db/src/thread.rs:690-696`; `crates/buzz-db/src/channel.rs:900`; `crates/buzz-db/src/lib.rs:2813-2817`; `crates/buzz-db/src/channel.rs:840` |
| Catalog introspection (`pg_class`, `pg_namespace`, `pg_inherits`, `pg_trigger`, `pg_attrdef`, `information_schema.tables`, `to_regclass`) | `crates/buzz-db/src/partition.rs:108-121`; `crates/buzz-db/src/replica_fence.rs:147-172`; `migrations/0014_push_lease_fts.sql:15-21`; `crates/buzz-db/src/relay_members.rs:518-522`; `crates/buzz-db/src/migration.rs:36-38` |
| Replication views (`pg_stat_activity`, `pg_prepared_xacts`, `pg_current_wal_lsn`, `pg_last_wal_replay_lsn`, `pg_is_in_recovery`, `pg_lsn` casts) | `crates/buzz-db/src/replica_fence.rs:404-463` |
| `pgcrypto` extension for `gen_random_uuid()` | `migrations/0001_initial_schema.sql:24` |
| `LOCK TABLE … IN SHARE ROW EXCLUSIVE MODE` in migrations | `migrations/0007_nip_rs_retention.sql:12`, `migrations/0008_…:9` |
| `SET CONSTRAINTS ALL IMMEDIATE` (verification only) | `crates/buzz-db/src/replica_fence.rs:232` |

**TLS.** Supplied by sqlx's `tls-rustls` feature (`Cargo.toml:53`). The crate
itself never sets TLS options — mode is whatever the connection URL specifies.

**Migration runner.** `sqlx::migrate!("../../migrations")` embeds the SQL at
compile time (`crates/buzz-db/src/migration.rs:11`); `MIGRATOR.run(pool)` at
`:19`. Checksums are therefore frozen — every schema change must be a new
additive file, a constraint asserted throughout
`crates/buzz-db/src/migration.rs:559-830`.

#### 3. Non-Postgres I/O

None. The crate opens no sockets or files of its own, spawns no processes, and
makes no HTTP calls: the only network egress is the sqlx Postgres connection(s).
The only spawned task is `tokio::spawn(replica_fence::run_probe(...))`
(`crates/buzz-db/src/lib.rs:463-467`), which itself only talks to the two pools.
Environment variables are read **only** inside `#[cfg(test)]` modules
(see `buzz-db-configuration.md`).

#### 4. Upstream / downstream coupling

- **Upstream (consumed):** `buzz-core` only — no other Buzz crate is a
  dependency (`crates/buzz-db/Cargo.toml:11-25`).
- **Downstream (consumers of this crate):** `buzz-db` is declared in the
  workspace dependency table (`Cargo.toml:126`) and is imported by the relay and
  other service crates; per `ARCHITECTURE.md:97` the relay is the only
  orchestrator and sibling service crates do not call each other.
- **Cross-crate constant duplication:** the FTS exclusion/allowlist kind lists
  and the push-eligible kind allowlist are inlined in frozen SQL and must be
  kept in sync with `buzz_core::kind` by hand — stated at
  `migrations/0001_initial_schema.sql:214-221`,
  `migrations/0005_agent_turn_metric_fts.sql:20-24`, and
  `migrations/0018_push_match_queue.sql:22-24`. The moderation action vocabulary
  is duplicated in `crates/buzz-db/src/moderation.rs:104-118` and asserted
  against the SQL CHECK by a test at `crates/buzz-db/src/migration.rs:640-645`.
- **Sibling crate referenced from comments only:** `buzz-search`
  (`crates/buzz-search/tests/fts_integration.rs` is named as the place to add
  FTS regression tests — `migrations/0001_initial_schema.sql:220-221`).
