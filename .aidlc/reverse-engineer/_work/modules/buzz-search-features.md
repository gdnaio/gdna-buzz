## Module: buzz-search (`crates/buzz-search`)

### Features

#### Capability inventory

| Capability | Present | Where |
|---|---|---|
| Community-scoped full-text search over `events.content` (via `search_tsv`) | yes | `query.rs:216-323` |
| Word/lexeme search with Postgres websearch grammar (`FullText`) | yes | `query.rs:142-146` |
| Typeahead prefix search on the trailing token (`Prefix`) | yes | `query.rs:147-176` |
| Relevance ranking (`ts_rank_cd`) surfaced to callers as `SearchHit.rank` | yes | `query.rs:236`, `query.rs:117` |
| Channel scoping with four explicit variants | yes | `query.rs:41-53`, `query.rs:248-264` |
| Kind pushdown | yes | `query.rs:267-273` |
| Author pushdown | yes | `query.rs:275-281` |
| `since` / `until` time-window pushdown | yes | `query.rs:283-293` |
| Soft-delete exclusion | yes | `query.rs:242` |
| Offset pagination with clamped page/per-page | yes | `query.rs:224-231`, `query.rs:295-298` |
| Empty-query short-circuit (no DB roundtrip) | yes | `query.rs:217-222` |
| Input hardening (NUL scrub, 4096-char cap) | yes | `query.rs:179-194` |
| Row-decode length validation for `id`/`pubkey` | yes | `query.rs:306-311` |

#### Deliberately absent

| Not present | Evidence |
|---|---|
| Any write/index/delete path | Only SQL verb in the crate is `SELECT` (`query.rs:234`); tests do their own `INSERT`/`UPDATE` directly (`tests/fts_integration.rs:107, 130, 690`). No `INSERT`/`UPDATE`/`DELETE` in `src/`. |
| Total result count / `found` | `SearchResult` has only `hits` and `page` (`query.rs:122-127`) |
| Keyset (cursor) pagination | only `LIMIT`/`OFFSET` (`query.rs:295-298`) |
| Highlighting / snippets (`ts_headline`) | not referenced anywhere in the crate |
| Faceting, aggregation, suggestions, synonyms, stemming | `'simple'` config = no stemming/stopwords (`query.rs:143`, `173`); no other features present |
| Tag (`#p`, `#e`, `#h`, `#d`) or `ids` pushdown | `SearchQuery` has no such fields (`query.rs:73-99`); caller re-filters (`crates/buzz-relay/src/api/bridge.rs:1582-1592`) |
| Authorization / membership enforcement | none in crate; see security doc |
| Retry, timeout, circuit-breaker, tracing/metrics | no `tracing`, `tokio::time`, or retry code in `src/` |
| Typesense client | no dependency (`Cargo.toml:10-15`); only two historical mentions in doc comments (`query.rs:20`, `query.rs:46`) |

#### Completeness assessment

The query side is functionally complete for the two callers in the repo (WS NIP-50
`REQ` at `crates/buzz-relay/src/handlers/req.rs:503-611` and the HTTP `/query`
bridge at `crates/buzz-relay/src/api/bridge.rs:1628-1698`). Both use the crate as
a candidate generator and then re-fetch + re-authorize, which is the documented
contract (`lib.rs:15-18`, `query.rs:3-9`).

Two rough edges that are visible in code rather than declared as gaps:

1. `per_page` handling has a redundant intermediate binding: `per_page` is clamped
   to `1..=500` at `query.rs:224`, then `per_page_actual` re-derives the value and
   substitutes `PER_PAGE_DEFAULT` when the *raw* input was `0`
   (`query.rs:225-229`). The clamp at `224` already maps `0 → 1`, so the `0` branch
   exists only because it inspects `query.per_page` again. Net behavior: `0 → 100`,
   `1..=500 → as-is`, `>500 → 500`.
2. The `search()` doc comment sketches the SQL as `FROM events, <tsquery> AS query`
   with `ts_rank_cd(search_tsv, query)` (`query.rs:196-207`), while the emitted SQL is
   `FROM events CROSS JOIN LATERAL (SELECT <tsquery> AS query) AS search_query` with
   `ts_rank_cd(search_tsv, search_query.query)` (`query.rs:233-242`). Same semantics,
   drifted text.

#### TODO / FIXME / HACK / XXX comments

A case-insensitive grep for `TODO`, `FIXME`, `HACK`, `XXX` across the whole crate
(`Cargo.toml`, `src/**`, `tests/**`) returns **zero matches**. The only
near-miss vocabulary is descriptive, not a marker:

| Text | File:line | Note |
|---|---|---|
| `the kind:0 content-flattening hack` | `crates/buzz-search/tests/fts_integration.rs:264-265` | prose in a test doc comment describing removed legacy behavior; not a `HACK:` marker |
| `xyzzy` / `qwerty` inside test token literals | `tests/fts_integration.rs:1109`, `1270`, `1366` | test fixtures, not markers |
