## Module: buzz-search (`crates/buzz-search`)

### Integrations

#### Dependencies — `crates/buzz-search/Cargo.toml`

| Dependency | Declaration | Line | Workspace version/features |
|---|---|---|---|
| `buzz-core` | `{ workspace = true }` | `Cargo.toml:11` | path `crates/buzz-core` (root `Cargo.toml:124`) |
| `sqlx` | `{ workspace = true }` | `Cargo.toml:12` | `0.9`, features `runtime-tokio, tls-rustls, postgres, uuid, chrono, json` (root `Cargo.toml:52-54`) |
| `uuid` | `{ workspace = true }` | `Cargo.toml:13` | `1`, features `v4, serde` (root `Cargo.toml:89`) |
| `thiserror` | `{ workspace = true }` | `Cargo.toml:14` | `2` (root `Cargo.toml:85`) |
| `tokio` (dev only) | `{ workspace = true }` | `Cargo.toml:17` | `1`, multi-thread/macros/etc. (root `Cargo.toml:43`) |

Four runtime dependencies total. No HTTP client, no serde, no tracing, no Redis, no
S3. Package description: "Postgres full-text search for Buzz, scoped by community"
(`Cargo.toml:8`).

#### buzz-core usage

| Item | Import | Use |
|---|---|---|
| `buzz_core::CommunityId` | `query.rs:14`, re-exported at `lib.rs:29` | `SearchQuery.community` field type (`query.rs:76`); `as_uuid()` for the SQL bind (`query.rs:241`, definition at `crates/buzz-core/src/tenant.rs:47-49`) |

That is the entire coupling to `buzz-core` in `src/`. Tests additionally import
`buzz_core::kind::{AUTHOR_ONLY_KINDS, KIND_AGENT_TURN_METRIC, KIND_MEMBER_ADDED_NOTIFICATION, KIND_MEMBER_REMOVED_NOTIFICATION, P_GATED_KINDS}`
and `buzz_core::kind::is_ephemeral` (`tests/fts_integration.rs:9-16`, `1382`) as
drift tripwires against the schema's hard-coded exclusion list.

#### sqlx / Postgres usage

| Aspect | Detail | Line |
|---|---|---|
| Imports | `sqlx::{PgPool, QueryBuilder, Row}` in query, `sqlx::PgPool` in lib | `query.rs:15`, `lib.rs:33` |
| Query style | Runtime `QueryBuilder<sqlx::Postgres>` — **not** the compile-time `sqlx::query!` macros; no `.sqlx/` offline cache is used by this crate | `query.rs:233` |
| Binding | `push_bind` for every dynamic value: community UUID (`241`), prefix/fulltext term (`144`, `168`), channel id vec (`257`, `262`), kinds vec (`270`), authors vec (`278`), since (`285`), until (`291`), limit (`296`), offset (`298`) | as listed |
| Execution | `qb.build().fetch_all(pool).await?` — single round trip, no transaction, no prepared-statement caching directives | `query.rs:300` |
| Row decoding | `Row::try_get` by column name for all six columns | `query.rs:304-318` |
| Postgres-specific SQL | `to_timestamp`, `EXTRACT(EPOCH FROM ...)::bigint`, `websearch_to_tsquery`, `to_tsvector`, `tsvector_to_array`, `ts_rank_cd`, `@@`, `= ANY(...)`, `CROSS JOIN LATERAL`, `regexp_split_to_table ... WITH ORDINALITY`, `string_agg`, `quote_literal`, `::tsquery` | `query.rs:143`, `154-176`, `234-298` |
| Types crossing the boundary | `Uuid` ↔ `uuid`, `Vec<u8>` ↔ `BYTEA`, `i32` ↔ `INT`, `i64` ↔ `BIGINT`, `f32` ↔ `real` (`ts_rank_cd` result), `Option<Uuid>` ↔ nullable `uuid` | `query.rs:304-318` |

#### Pool handling

`SearchService` stores an owned `PgPool` clone (`lib.rs:41`, `lib.rs:46-48`). The
crate never creates, configures, closes, or resizes a pool — no `PgPoolOptions` in
`src/`. `PgPool` is internally `Arc`-based, so `#[derive(Clone)]` on
`SearchService` (`lib.rs:39`) shares the same pool. The relay wraps it in
`Arc<SearchService>` in `AppState` and constructs it from the relay's pool
(`crates/buzz-relay/src/state.rs:1273`).

#### Non-Postgres I/O

None. No filesystem access, no network client, no environment reads, no process
spawning, no clock or RNG use in `src/`. The only `std::env::var` calls in the
crate are in the test harness (`tests/fts_integration.rs:33`, `92`).

#### Typesense status — explicit check

| Question | Answer | Evidence |
|---|---|---|
| Typesense dependency in `Cargo.toml`? | No | `Cargo.toml:10-18` lists only buzz-core, sqlx, uuid, thiserror, tokio |
| Typesense client/HTTP code anywhere in the crate? | No | grep for `typesense` (case-insensitive) across the crate returns 2 hits, both prose in doc comments |
| Remaining mentions | Two, historical | `query.rs:20` ("the legacy … matrix from the Typesense relay"), `query.rs:46` ("what the legacy Typesense `channel_id:=__global__` sentinel meant") |

Related historical prose outside the crate (not code): the schema comment
"Full-text search vector (Typesense → Postgres FTS)"
(`migrations/0001_initial_schema.sql:200`, `schema/schema.sql:199`).

#### Test-harness integrations

The integration test file couples directly to migration SQL at compile time via
`include_str!` (`tests/fts_integration.rs:22-32`): migrations `0001`–`0008` plus
`0014`, applied in order into a per-test schema (`tests/fts_integration.rs:57-84`).
It creates and drops a uniquely-named schema through a separate one-connection
admin pool (`tests/fts_integration.rs:36-46`, `87-103`) and passes the schema via
the connection URL option `options=-c search_path=<schema>`
(`tests/fts_integration.rs:48`). Adding a future FTS-affecting migration requires
editing this list — noted in-file at `tests/fts_integration.rs:55-56`.
