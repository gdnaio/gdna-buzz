## Module: buzz-search (`crates/buzz-search`)

### API Surface

The crate exposes **two callable entry points** (one free fn, one method) plus one
constructor. Everything else public is data.

#### Crate root exports — `crates/buzz-search/src/lib.rs`

| Item | Line | Kind |
|---|---|---|
| `pub mod error` | `lib.rs:25` | module |
| `pub mod query` | `lib.rs:27` | module |
| `pub use buzz_core::CommunityId` | `lib.rs:29` | re-export |
| `pub use error::SearchError` | `lib.rs:30` | re-export |
| `pub use query::{search, ChannelScope, SearchHit, SearchMode, SearchQuery, SearchResult}` | `lib.rs:31` | re-export |
| `pub struct SearchService` | `lib.rs:40` | type |

Crate lints: `#![deny(unsafe_code)]` (`lib.rs:1`), `#![warn(missing_docs)]`
(`lib.rs:2`).

---

#### Functions and methods (complete list)

| Signature | Line | Public? |
|---|---|---|
| `impl SearchService { pub fn new(pool: PgPool) -> Self }` | `lib.rs:46-48` | yes |
| `impl SearchService { pub async fn search(&self, query: &SearchQuery) -> Result<SearchResult, SearchError> }` | `lib.rs:51-53` | yes |
| `pub async fn query::search(pool: &PgPool, query: &SearchQuery) -> Result<SearchResult, SearchError>` | `query.rs:216` | yes |
| `fn push_tsquery(qb: &mut QueryBuilder<sqlx::Postgres>, mode: SearchMode, search_text: &str)` | `query.rs:140` | private |
| `fn normalized_search_text(q: &str) -> Option<String>` | `query.rs:179` | private |

`SearchService::search` is a one-line delegate to the free function:
`query::search(&self.pool, query).await` (`lib.rs:52`). There is no other method
on `SearchService`, no `delete`, no `index`, no `count`.

No trait impls are hand-written; only derives (`Debug`, `Clone` on
`SearchService` at `lib.rs:39`; see data-model doc for the rest) and
`thiserror`'s `Error`/`Display`/`From<sqlx::Error>` on `SearchError`
(`error.rs:4-8`).

---

#### Construction

`SearchService::new(pool: PgPool)` takes ownership of a clone of the caller's
pool (`lib.rs:46-48`). No pool options, timeouts, or config are read here — the
pool is fully configured by the caller. Callers in-tree construct it from the
relay's pool, e.g. `crates/buzz-relay/src/state.rs:1273` and
`crates/buzz-relay/src/api/operator.rs:597`; the relay stores it as
`Arc<SearchService>` in `AppState` (`ARCHITECTURE.md:586`).

---

#### fn → SQL predicate shape → return type

| Fn | SQL emitted | Predicates | Returns |
|---|---|---|---|
| `SearchService::new` | none | — | `SearchService` |
| `SearchService::search` | delegates to `query::search` | see below | `Result<SearchResult, SearchError>` |
| `query::search` (empty/whitespace/NUL-only `q`) | **none — no roundtrip** (`query.rs:217-222`) | — | `Ok(SearchResult { hits: [], page: clamp(page,1,1000) })` |
| `query::search` (non-empty `q`) | single `SELECT` over `events` (`query.rs:233-298`) | `community_id = $1` **always** (`query.rs:240-241`); `deleted_at IS NULL` **always** (`query.rs:242`); `search_tsv @@ search_query.query` **always** (`query.rs:242`); then optional `channel_id` scope (`query.rs:248-264`), `kind = ANY` (`query.rs:267-273`), `pubkey = ANY` (`query.rs:275-281`), `created_at >= to_timestamp` (`query.rs:283-287`), `created_at <= to_timestamp` (`query.rs:289-293`) | `Result<SearchResult, SearchError>` |
| `push_tsquery` (private) | `websearch_to_tsquery('simple', $n)` for `FullText` (`query.rs:143-145`); token-split/`quote_literal`/`:*` aggregation subselect for `Prefix` (`query.rs:154-176`) | — | `()` (mutates the builder) |
| `normalized_search_text` (private) | none | — | `Option<String>` — `None` for empty-after-trim or empty-after-NUL-scrub (`query.rs:180-190`) |

Projection and ordering are fixed, not caller-selectable:
`SELECT id, kind, pubkey, channel_id, EXTRACT(EPOCH FROM created_at)::bigint AS created_at_s, ts_rank_cd(...) AS rank`
(`query.rs:234-236`) and
`ORDER BY rank DESC, created_at DESC, id LIMIT $n OFFSET $n`
(`query.rs:295-298`).

---

#### Error surface

Every failure path returns `SearchError::Db` (`error.rs:8`):

| Failure | Mechanism | Line |
|---|---|---|
| Query execution / connection error | `?` on `fetch_all` → `From<sqlx::Error>` | `query.rs:300` |
| Column type mismatch / missing column | `?` on `row.try_get` | `query.rs:304-305`, `314-318` |
| `id` not exactly 32 bytes | mapped to `sqlx::Error::Decode("event id column is N bytes, expected 32")` | `query.rs:306-308` |
| `pubkey` not exactly 32 bytes | mapped to `sqlx::Error::Decode("pubkey column is N bytes, expected 32")` | `query.rs:309-311` |

No panics in `src/`: no `unwrap()`, `expect()`, or indexing that can panic appears
in `lib.rs`, `query.rs`, or `error.rs` (verified by reading all three files).
