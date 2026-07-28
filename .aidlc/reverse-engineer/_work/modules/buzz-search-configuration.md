## Module: buzz-search (`crates/buzz-search`)

### Configuration

#### Environment variables

| Variable | Read in production code? | Where | Default | Purpose |
|---|---|---|---|---|
| `BUZZ_TEST_DATABASE_URL` | No — test harness only | `tests/fts_integration.rs:33`, `tests/fts_integration.rs:92` | `TEST_DB_URL` = `postgres://buzz:buzz_dev@localhost:5432/buzz` (`tests/fts_integration.rs:21`) | Postgres connection for the ignored integration tests |

**No environment variable is read anywhere in `src/`.** All runtime configuration
reaches the crate through the `PgPool` the caller passes to
`SearchService::new(pool)` (`crates/buzz-search/src/lib.rs:46-48`) — connection
string, pool sizing, TLS, and timeouts are entirely the caller's concern
(constructed by the relay, e.g. `crates/buzz-relay/src/state.rs:1273`).

#### Cargo features

| Feature | Status |
|---|---|
| Crate-defined features | None — `Cargo.toml` has no `[features]` section (`crates/buzz-search/Cargo.toml:1-18`) |
| `#[cfg(feature = ...)]` in code | None in any `.rs` file |
| `#[cfg(test)]` | One, gating the unit-test module (`src/query.rs:325`) |
| Inherited sqlx features (from workspace) | `runtime-tokio`, `tls-rustls`, `postgres`, `uuid`, `chrono`, `json` (root `Cargo.toml:52-54`) |
| Inherited uuid features | `v4`, `serde` (root `Cargo.toml:89`) |

Package metadata is fully workspace-inherited (`version`, `edition`,
`rust-version`, `license`, `repository`) — `crates/buzz-search/Cargo.toml:3-7`.

#### Compile-time constants (complete list)

| Constant | Value | Type | Line | Effect |
|---|---|---|---|---|
| `PER_PAGE_MAX` | `500` | `u32` | `src/query.rs:129` | Upper clamp on `SearchQuery.per_page` → `LIMIT` (`query.rs:224`) |
| `PER_PAGE_DEFAULT` | `100` | `u32` | `src/query.rs:130` | Substituted when `per_page == 0` (`query.rs:225-229`) |
| `SEARCH_TEXT_MAX_CHARS` | `4096` | `usize` | `src/query.rs:134` | Hard char cap on search text before it reaches Postgres' text-search parsers (`query.rs:185`) |
| `PAGE_MAX` | `1000` | `u32` | `src/query.rs:138` | Upper clamp on `page`, bounding `OFFSET = (page-1) * per_page` (`query.rs:220`, `230-231`) |

Effective caps composed with those constants: maximum reachable `OFFSET` is
`999 * 500 = 499_500` rows, and maximum rows returned per call is 500.

Test-file constants (not production configuration):

| Constant | Value | Line |
|---|---|---|
| `TEST_DB_URL` | `postgres://buzz:buzz_dev@localhost:5432/buzz` | `tests/fts_integration.rs:21` |
| `MIGRATION_0001_SQL` … `MIGRATION_0008_SQL`, `MIGRATION_0014_SQL` | `include_str!` of the corresponding `migrations/*.sql` | `tests/fts_integration.rs:22-32` |

#### Hard-coded behavioral settings (not configurable)

| Setting | Value | Line |
|---|---|---|
| Text-search configuration | `'simple'` (no stemming, no stopwords) — used for both `websearch_to_tsquery` and the prefix-mode `to_tsvector` | `query.rs:143`, `query.rs:173` |
| Ranking function | `ts_rank_cd` (no weights, no normalization flag) | `query.rs:236` |
| Ordering | `rank DESC, created_at DESC, id` | `query.rs:295` |
| Prefix-mode token split | `regexp_split_to_table(<term>, '\s+')` | `query.rs:167-170` |
| Prefix-mode conjunction | `' & '` | `query.rs:157` |
| Table queried | `events` (unqualified; resolved via the connection's `search_path`) | `query.rs:237` |

#### Caller-side settings that shape this crate's inputs (for reference)

| Setting | Value | Site |
|---|---|---|
| `MAX_SEARCH_PAGES` (WS pages iterated) | `10` | `crates/buzz-relay/src/handlers/req.rs:421` |
| WS `per_page` | `100` fixed | `crates/buzz-relay/src/handlers/req.rs:591` |
| Bridge `per_page` | `filter.limit.unwrap_or(100).min(500)` | `crates/buzz-relay/src/api/bridge.rs:1660` |
| Bridge `page` | raw `page` / `search_page` / `searchPage`, `> 0`, default `1` | `crates/buzz-relay/src/api/bridge.rs:345-353` |
| Bridge `mode` | `"prefix"` → `SearchMode::Prefix`, else `FullText` | `crates/buzz-relay/src/api/bridge.rs:337-345` |
