## Module: buzz-search (`crates/buzz-search`)

### Conventions

#### Module organization

| Module | File | Role |
|---|---|---|
| crate root | `src/lib.rs` (54 LOC) | lints, module decls, re-exports, `SearchService` handle |
| `error` | `src/error.rs` (9 LOC) | `SearchError` only |
| `query` | `src/query.rs` (352 LOC) | all query types + SQL construction + execution + unit tests |
| integration tests | `tests/fts_integration.rs` (1448 LOC) | Postgres-backed behavior tests |

Both modules are declared `pub mod` with doc comments on the declaration itself
(`lib.rs:24-27`), and their contents are flattened into the crate root by a single
`pub use` line each (`lib.rs:29-31`). Callers use both spellings in-tree:
`buzz_search::SearchService` (`crates/buzz-relay/src/state.rs:28`) and
`buzz_search::ChannelScope` / `SearchQuery` / `SearchMode` via the root
(`crates/buzz-relay/src/api/bridge.rs:1665`, `1687`, `1697`).

#### Lints

| Lint | Line |
|---|---|
| `#![deny(unsafe_code)]` | `lib.rs:1` |
| `#![warn(missing_docs)]` | `lib.rs:2` |

`missing_docs` is honored throughout: every public item, field, and enum variant
carries a doc comment (`query.rs:43-52`, `75-98`, `106-117`, `123-126`;
`error.rs:3-6`; `lib.rs:35-51`). One `#[allow]` appears in the crate, in tests
only: `#[allow(clippy::too_many_arguments)]` on the `insert_event` helper
(`tests/fts_integration.rs:118`).

#### Naming

| Pattern | Examples |
|---|---|
| Types: `Search*` prefix for the public surface | `SearchService`, `SearchQuery`, `SearchHit`, `SearchResult`, `SearchMode`, `SearchError` |
| Enum variants read as constraints, not flags | `Any`, `ChannelLessOnly`, `Channels`, `ChannelsOrChannelLess` (`query.rs:44-52`) |
| Private helpers: verb-first (`push_*`) or noun-phrase for pure fns | `push_tsquery` (`query.rs:140`), `normalized_search_text` (`query.rs:179`) |
| Constants: SCREAMING_SNAKE with the bound in the name | `PER_PAGE_MAX`, `PER_PAGE_DEFAULT`, `SEARCH_TEXT_MAX_CHARS`, `PAGE_MAX` (`query.rs:129-138`) |
| SQL aliases spelled out | `created_at_s`, `search_query.query`, `prefix_terms`, `raw_token`, `normalized` (`query.rs:235-237`, `168-176`) |
| Test names assert behavior, not method names | `search_does_not_return_other_community_events`, `channel_less_only_excludes_per_channel_events`, `excluded_kinds_are_storage_level_unsearchable` |

#### Error handling

Single-variant `thiserror` enum (`error.rs:4-9`):

| Variant | Attributes | Message |
|---|---|---|
| `Db(sqlx::Error)` | `#[from]`, `#[error("database error: {0}")]` | `error.rs:7-8` |

Conventions observed:
- All fallible steps use `?` — no `unwrap()`/`expect()`/`panic!` anywhere in `src/`
  (checked in all three source files), matching the repo rule against new
  `unwrap()`/`expect()` in production paths.
- Domain-shaped decode failures are expressed by re-wrapping into
  `sqlx::Error::Decode` with a message that names the column and the observed
  length, rather than adding an enum variant (`query.rs:306-311`).
- Degenerate input is handled by returning a valid empty result rather than an
  error (`query.rs:217-222`).

#### Query style

- One statement, assembled with `QueryBuilder` in strict order: projection + `FROM`
  seed (`query.rs:233-238`), tenant predicate (`240-241`), always-on predicates
  (`242`), optional predicates (`248-293`), `ORDER BY`/`LIMIT`/`OFFSET`
  (`295-298`).
- Every dynamic value goes through `push_bind`; `push` is used only for fixed SQL
  text (see integrations doc for the full bind list).
- Vectors are `.clone()`d into the bind because the builder needs owned values
  (`query.rs:257`, `262`, `270`, `278`).
- Multi-line SQL uses trailing `\` line continuations inside one string literal
  (`query.rs:234-237`, `154-176`).
- Non-obvious SQL carries an adjacent rationale comment (prefix-mode design at
  `query.rs:148-153`; channel-scope mapping at `query.rs:244-247`).
- Doc comments state the invariant in prose next to the code that enforces it
  ("`community_id = $ctx` is the first predicate and is non-negotiable",
  `query.rs:209-210`).

#### Testing patterns

| Metric | Count | Where |
|---|---|---|
| `#[test]` (sync unit tests) | 3 | `query.rs:329`, `338`, `346` inside `#[cfg(test)] mod tests` (`query.rs:325-352`) |
| `#[tokio::test]` (async integration tests) | 18 | `tests/fts_integration.rs` |
| `#[ignore = "requires Postgres"]` | 18 | every integration test; none of the 3 unit tests is ignored |

So 21 tests total, 18 infra-gated. Unit tests cover only
`normalized_search_text` (trim/reject-empty, NUL replacement, char cap).

Integration-test conventions:
- Per-test isolated schema named `fts_test_<uuid-simple>`, created and dropped
  around each test, declared parallel-safe (`tests/fts_integration.rs:6-8`,
  `35-46`, `87-103`).
- Full migration chain replayed so the test schema matches production
  (`tests/fts_integration.rs:55-84`).
- Test-only DDL string interpolation is explicitly marked with
  `sqlx::AssertSqlSafe` (`tests/fts_integration.rs:44`, `100`).
- Fixture helpers: `mk_community`, `insert_event`, `rand_bytes32`
  (`tests/fts_integration.rs:105-153`).
- Deterministic timestamps in the `1_700_000_000` family, unique content tokens
  per test to avoid cross-test coupling.
- Several tests document their own mutation-kill argument — flip the predicate and
  the assertion goes red (`tests/fts_integration.rs:877-887`, `1100-1104`).
- Two tests are explicit drift tripwires that iterate Rust constants
  (`AUTHOR_ONLY_KINDS`, persistent subset of `P_GATED_KINDS`) against the schema's
  inlined exclusion list (`tests/fts_integration.rs:1256-1265`, `1338-1361`).
- Run instruction is documented at the top of the file, including the
  `BUZZ_TEST_DATABASE_URL` override and `-- --include-ignored`
  (`tests/fts_integration.rs:3`).
