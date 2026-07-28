## Module: buzz-search (`crates/buzz-search`)

### Technical Debt

#### Markers

Zero `TODO` / `FIXME` / `HACK` / `XXX` markers exist in the crate (grep over
`Cargo.toml`, `src/**`, `tests/**`). Everything below is inferred from reading code,
not from declared debt.

#### Complexity hotspots

| Hotspot | Location | Nature |
|---|---|---|
| Prefix-mode tsquery construction | `src/query.rs:154-176` (~23 lines of SQL inside two `push` calls) | A nested subselect combining `regexp_split_to_table ... WITH ORDINALITY`, a window `max(token_ord) OVER ()`, `CROSS JOIN LATERAL unnest(tsvector_to_array(to_tsvector(...))) WITH ORDINALITY`, `quote_literal`, `string_agg` with `ORDER BY`, `COALESCE`, and a `::tsquery` cast — expressed as a Rust string literal with `\` continuations. Not unit-testable in isolation; correctness depends entirely on 3 ignored integration tests (`tests/fts_integration.rs:306-539`). |
| `search()` length | `src/query.rs:216-323` (108 lines) | Single function does normalization dispatch, clamping, SQL assembly across 9 optional fragments, execution, and row decoding. No sub-functions besides `push_tsquery`. |
| Manual row decoding | `src/query.rs:302-320` | Six `try_get` calls plus two `try_into` length checks with hand-written error strings; no `FromRow` derive, so column-name drift is a runtime failure. |

#### Redundant / effectively dead logic

| Item | Location | Detail |
|---|---|---|
| `per_page` intermediate binding | `src/query.rs:224-229` | `let per_page = query.per_page.clamp(1, PER_PAGE_MAX)` already maps `0 → 1`; the following `if query.per_page == 0 { PER_PAGE_DEFAULT }` re-reads the raw field to override that. The `per_page` binding is used only in the `else` arm, so the clamp's `0 → 1` result is unreachable. Behavior is fine (`0 → 100`), the code path is redundant. |
| `SearchHit.rank` / `kind` / `pubkey` / `channel_id` for in-tree callers | `src/query.rs:107-117` vs `crates/buzz-relay/src/handlers/req.rs:625-626`, `crates/buzz-relay/src/api/bridge.rs:1725` | Both callers immediately map hits to `event_id` only; the other four fields are selected, decoded, and discarded. `SELECT` of `kind`/`pubkey` and the `ts_rank_cd` computation are still needed for ordering (`rank` is in `ORDER BY`), but the decode work is unused. |
| `SearchResult.page` | `src/query.rs:126` | Returned but not read by either in-tree caller (WS drives pages in its own loop, `crates/buzz-relay/src/handlers/req.rs:588-604`). Its only consumer is a test assertion (`tests/fts_integration.rs:1044`). |
| `pub use query::search` free function | `src/lib.rs:31`, `src/query.rs:216` | Both the free fn and the `SearchService` method are public; all in-tree callers use the service. Two public entry points for one behavior. |

#### Documentation drift

| Drift | Location | Detail |
|---|---|---|
| SQL sketch vs emitted SQL | `src/query.rs:196-207` vs `src/query.rs:233-242` | Doc says `FROM events, <mode-specific tsquery> AS query` and `ts_rank_cd(search_tsv, query)`; code emits `FROM events CROSS JOIN LATERAL (SELECT ... AS query) AS search_query` and `ts_rank_cd(search_tsv, search_query.query)`. |
| Crate doc quotes an outdated `search_tsv` definition | `src/lib.rs:6-8` | States the column is `to_tsvector('simple', content)`; the actual expression is a `CASE` with a kind exclusion list (`migrations/0001_initial_schema.sql:222-226`) or, on fresh installs, a positive allowlist (`migrations/0008_fresh_install_search_allowlist.sql:15-20`). The privacy behavior the crate relies on is not described in its own doc. |
| ARCHITECTURE.md exclusion list is stale | `ARCHITECTURE.md:808-810` | Lists `CASE WHEN kind IN (1059, 30300, 30622)`; current schema is `(1059, 30300, 30350, 30622, 44100, 44101, 44200)` (`schema/schema.sql:212`). |
| `SearchQuery.per_page` doc | `src/query.rs:95` | Says "Clamped at 500 internally" without mentioning the `0 → 100` substitution. |

#### Test coverage gaps

| Gap | Detail |
|---|---|
| 18 of 21 tests require infrastructure | Every behavior test is `#[ignore = "requires Postgres"]` (`tests/fts_integration.rs`, 18 occurrences). `cargo test -p buzz-search` in a plain CI run executes only the 3 `normalized_search_text` unit tests (`src/query.rs:329-351`). |
| No SQL-string assertions | `QueryBuilder` output is never inspected in a unit test, so predicate presence/absence (`kinds = Some(vec![])`, `authors = Some(vec![])`, `since` only, `until` only, `ChannelScope::Any`) has no infra-free coverage. |
| `authors` filter untested | No integration test sets `authors: Some(...)` — every call site in `tests/fts_integration.rs` passes `authors: None`. The `pubkey = ANY` branch (`src/query.rs:275-281`) is uncovered. |
| `per_page` clamping untested | No test passes `per_page: 0` or `per_page > 500`; only `page: u32::MAX` is exercised (`tests/fts_integration.rs:1014-1055`). |
| `mode: Prefix` combined with `authors`/`since`/`until` untested | Prefix tests only vary `kinds` and content. |
| Rank ordering untested | No test asserts that a higher-`ts_rank_cd` document precedes a lower one; ordering coverage is limited to counts and `created_at` (`tests/fts_integration.rs:748-810`). |
| Decode-error branches untested | The "N bytes, expected 32" paths (`src/query.rs:306-311`) have no test — they require a malformed row. |
| `SearchError` surface untested | No test exercises a DB failure. |
| Migration list is manually maintained | `tests/fts_integration.rs:22-32`, `57-84` enumerate migrations by hand; a future FTS-affecting migration that is not added silently makes the privacy tripwires test a stale schema. The file notes the obligation at `tests/fts_integration.rs:55-56`. |
| Allowlist vs exclusion-list divergence is untested | Tests replay `0001…0008 + 0014` against an *empty* `events` table, so migration 0008's emptiness branch fires and the tests actually validate the **positive allowlist** path (`migrations/0008_fresh_install_search_allowlist.sql:13-23`). The brownfield exclusion-list expression (`migrations/0005_agent_turn_metric_fts.sql:26-31`) is therefore not the expression under test, even though the test doc comments describe the exclusion CASE (`tests/fts_integration.rs:1085-1104`). |

#### Coupling and structural debt

| Item | Detail |
|---|---|
| Privacy policy split across three languages/locations | Rust constants (`buzz_core::kind::AUTHOR_ONLY_KINDS`, `P_GATED_KINDS`), inlined SQL literals in each migration, and `schema/schema.sql` must be kept in sync by hand; the migrations say so explicitly (`migrations/0001_initial_schema.sql:216-221`, `migrations/0005_agent_turn_metric_fts.sql:19-22`). Tripwire tests exist but are ignored by default. |
| Divergent effective search policy per deployment | Fresh installs get an allowlist; populated installs keep an exclusion list until an operator runs `scripts/maintenance/nip_rs_search_allowlist.sql`. Two databases can answer the same query differently, and nothing in this crate can detect which policy is active. |
| Tests compile-time-depend on `migrations/` paths | `include_str!("../../../migrations/...")` (`tests/fts_integration.rs:22-32`) breaks the crate's test build if a migration file is renamed. |
| Test file is 1448 LOC vs 415 LOC of source | 3.5:1 test-to-source ratio concentrated in one file with a repeated 12-field `SearchQuery` literal (18 occurrences); no builder/helper for query construction, so adding a `SearchQuery` field requires touching every test. |
| No offline sqlx verification | Runtime `QueryBuilder` only (`src/query.rs:233`); column names and types are unverified at compile time. Matches the repo-wide limitation recorded at `ARCHITECTURE.md:823`. |
| Unqualified table name | `FROM events` relies on `search_path` (`src/query.rs:237`), which is what lets the test harness redirect to a per-test schema — but it also means the crate cannot be pointed at a schema-qualified table without changing the pool's `search_path`. |

#### Deprecated API usage

None found. `sqlx 0.9` `QueryBuilder`/`Row::try_get` are current; `thiserror 2`
attribute style is current; `sqlx::AssertSqlSafe` (`tests/fts_integration.rs:44`,
`100`) is the sqlx 0.9-era explicit opt-in for non-literal SQL, not a deprecated
path. No `#[deprecated]` items are referenced.

#### Observability gap

No `tracing`/`log` dependency and no instrumentation in `src/` — slow searches,
clamped pages, truncated search text, and empty-query short-circuits are all
invisible to operators from inside this crate. Callers log only failures
(`crates/buzz-relay/src/handlers/req.rs:614`, `crates/buzz-relay/src/api/bridge.rs:1719-1722`).
