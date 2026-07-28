## Module: buzz-search (`crates/buzz-search`)

### Business Rules

All rules live in `query::search` (`crates/buzz-search/src/query.rs:216-323`) and its
two private helpers. `SearchService` adds no rules (`lib.rs:51-53`).

#### R1 — Community scoping is unconditional and first

| What it enforces | Site | Trigger |
|---|---|---|
| `WHERE community_id = $1` is the first predicate of every executed statement; the bound value is `*query.community.as_uuid()` | `query.rs:240-241` | every non-empty-`q` call |

`SearchQuery.community` is a non-`Option` `CommunityId` (`query.rs:76`), and
`CommunityId` can only be minted from a server-trusted UUID
(`crates/buzz-core/src/tenant.rs:43-45`). There is no code path in the crate that
builds the statement without this predicate — the builder is seeded with the
`FROM` clause and the `WHERE community_id = ` fragment before any optional
predicate is considered (`query.rs:233-242`).

#### R2 — Empty / whitespace-only search text short-circuits with no SQL

| What it enforces | Site | Trigger |
|---|---|---|
| Returns `SearchResult { hits: [], page: clamp(page,1,PAGE_MAX) }` without touching the pool | `query.rs:217-222` | `normalized_search_text(&q)` returns `None`, i.e. `q.trim()` empty, or NUL-scrub + trim leaves it empty (`query.rs:180-190`) |

#### R3 — Search-text normalization: trim, NUL→space, hard char cap

| What it enforces | Site | Trigger |
|---|---|---|
| `q.trim()` applied first | `query.rs:180` | always |
| Each `'\0'` replaced with `' '` (Postgres text-search input cannot contain NUL) | `query.rs:186` | any NUL in text |
| Truncation to the first `SEARCH_TEXT_MAX_CHARS = 4096` **chars** (not bytes) | `query.rs:134`, `query.rs:185` | longer input |
| Re-trim after cleaning; `None` if now empty | `query.rs:188-193` | trailing-whitespace-only remainder |

Stated rationale: bound Postgres text-search parser CPU/memory per request
(`query.rs:131-134`).

#### R4 — NIP-50 term handling is parameter-bound in both modes (no escaping by string surgery)

| What it enforces | Site | Trigger |
|---|---|---|
| `FullText`: term passed to `websearch_to_tsquery('simple', $n)` as a bind, so Postgres' own websearch grammar (quoted phrases, `or`, `-`) applies and no operator can escape into SQL text | `query.rs:143-145` | `mode == FullText` |
| `Prefix`: term bound once (`query.rs:168`) then split/normalized entirely inside SQL — `regexp_split_to_table($n, '\s+') WITH ORDINALITY`, `to_tsvector('simple', token)`, `tsvector_to_array`, `unnest ... WITH ORDINALITY` | `query.rs:154-176` | `mode == Prefix` |
| `Prefix`: each lexeme wrapped in `quote_literal(...)` before concatenation, explicitly to stop tsquery-syntax injection from punctuation/operators | `query.rs:150-153`, `query.rs:157` | `mode == Prefix` |
| `Prefix`: only the **trailing whitespace-delimited token** gets `:*`; completed tokens stay exact (`CASE WHEN token_ord = max_token_ord`) | `query.rs:157-158` | `mode == Prefix` |
| `Prefix`: lexemes joined with `' & '` ordered by `(token_ord, lex_ord)`; `COALESCE(..., '')::tsquery` so an all-stopword/empty tokenization degrades to the empty tsquery rather than an error | `query.rs:155-162` | `mode == Prefix` |

The `simple` text-search configuration is used on both the query side
(`query.rs:143`, `query.rs:173`) and the storage side
(`migrations/0001_initial_schema.sql:224`), so query normalization matches the
generated column by construction — this symmetry is asserted in the code comment
at `query.rs:149-152`.

#### R5 — `ChannelScope` semantics (one rule per variant)

| Variant | Predicate emitted | Site | Effect |
|---|---|---|---|
| `Any` | none | `query.rs:249-251` | no channel constraint; every community row (subject to other predicates) is a candidate, including channel-scoped rows the caller may not be able to read |
| `ChannelLessOnly` | `AND channel_id IS NULL` | `query.rs:252-254` | only channel-less/global events |
| `Channels(ids)` | `AND channel_id = ANY($n)` | `query.rs:255-259` | only the listed channels; `ids = []` → `= ANY('{}')` → zero rows (documented at `query.rs:33-36`, asserted at `tests/fts_integration.rs:645-668`) |
| `ChannelsOrChannelLess(ids)` | `AND (channel_id = ANY($n) OR channel_id IS NULL)` | `query.rs:260-264` | listed channels plus channel-less; `ids = []` is equivalent to `ChannelLessOnly` (`query.rs:36-39`) |

Caller-side mapping from the legacy `(accessible_channels, include_global)` pair
lives outside this crate in `crates/buzz-relay/src/handlers/req.rs:484-501`; the
`empty && !include_global` case returns `None` there and the caller EOSEs instead
of calling search (`req.rs:517-524`).

#### R6 — Soft-deleted events are never candidates

| What it enforces | Site | Trigger |
|---|---|---|
| `AND deleted_at IS NULL` in every statement | `query.rs:242` | every non-empty-`q` call |

Regression test: `tests/fts_integration.rs:674-714`.

#### R7 — Optional NIP-01 pushdowns treat `Some(empty)` as "no constraint"

| What it enforces | Site | Trigger |
|---|---|---|
| `kinds`: predicate added only if `Some` **and** non-empty | `query.rs:267-273` | `kinds = Some(vec![...])` |
| `authors`: predicate added only if `Some` **and** non-empty | `query.rs:275-281` | `authors = Some(vec![...])` |
| `since`: `AND created_at >= to_timestamp($n)` (inclusive) | `query.rs:283-287` | `since.is_some()` |
| `until`: `AND created_at <= to_timestamp($n)` (inclusive) | `query.rs:289-293` | `until.is_some()` |

No validation that `since <= until`; an inverted range simply yields zero rows.
No validation that author byte-vectors are 32 bytes long.

#### R8 — Which kinds are searchable is a **storage** decision, not a crate rule

| What it enforces | Site | Trigger |
|---|---|---|
| The crate applies no allow/deny kind list; it only forwards the caller's `kinds` filter | `query.rs:267-273` (only kind logic in the crate) | always |
| Exclusion is realized by `search_tsv` being `NULL` for excluded kinds, which `@@` can never match | `migrations/0001_initial_schema.sql:222-226`, `migrations/0005_agent_turn_metric_fts.sql:26-31`, `migrations/0008_fresh_install_search_allowlist.sql:15-20`, `migrations/0014_push_lease_fts.sql:28-32`, `schema/schema.sql:211-215` | row insert |

Effective policy depends on which migration path a database took:

| Path | `search_tsv` expression | Reference |
|---|---|---|
| Fresh schema file | `CASE WHEN kind IN (1059, 30300, 30350, 30622, 44100, 44101, 44200) THEN NULL ELSE to_tsvector('simple', content) END` | `schema/schema.sql:211-215` |
| Migration 0001 | excludes `1059, 30300, 30622, 44100, 44101` | `migrations/0001_initial_schema.sql:222-226` |
| + 0005 | excludes `1059, 30300, 30622, 44100, 44101, 44200` | `migrations/0005_agent_turn_metric_fts.sql:26-31` |
| + 0008, **only if `events` is empty** | replaced by positive allowlist `kind IN (0, 9, 40002, 45001, 45003)` | `migrations/0008_fresh_install_search_allowlist.sql:13-23` |
| + 0014 | wraps whatever exists: `CASE WHEN kind = 30350 THEN NULL ELSE (<previous>) END` | `migrations/0014_push_lease_fts.sql:28-32` |
| Brownfield opt-in to allowlist | operator runs `scripts/maintenance/nip_rs_search_allowlist.sql:11-18` out of band | that file |

#### R9 — Ordering is fixed: relevance, then recency, then id

| What it enforces | Site |
|---|---|
| `ORDER BY rank DESC, created_at DESC, id` where `rank = ts_rank_cd(search_tsv, search_query.query)` | `query.rs:236`, `query.rs:295` |

`id` (BYTEA, ascending) is the deterministic tiebreak, which is what makes
OFFSET pagination stable across pages.

#### R10 — Pagination and limit clamps

| What it enforces | Value | Site |
|---|---|---|
| `per_page` clamped to `1..=PER_PAGE_MAX` | `PER_PAGE_MAX = 500` | `query.rs:129`, `query.rs:224` |
| `per_page == 0` → `PER_PAGE_DEFAULT` | `PER_PAGE_DEFAULT = 100` | `query.rs:130`, `query.rs:225-229` |
| `page` clamped to `1..=PAGE_MAX` (both on the SQL path and the empty-query path) | `PAGE_MAX = 1000` | `query.rs:138`, `query.rs:220`, `query.rs:230` |
| `OFFSET = (page - 1) * per_page_actual`, computed in `i64` | — | `query.rs:231`, `query.rs:297-298` |
| Returned `SearchResult.page` is the clamped page, not the requested one | — | `query.rs:322` (test: `tests/fts_integration.rs:1044`, expects `1000` for `u32::MAX`) |

Stated reason for the page clamp: pages are server-generated today (WS iterates
`1..=MAX_SEARCH_PAGES`, bridge passes a page), but clamp anyway so future
untrusted input cannot produce a huge OFFSET (`query.rs:135-138`). Caller values:
`MAX_SEARCH_PAGES = 10` (`crates/buzz-relay/src/handlers/req.rs:421`), WS
`per_page = 100` (`req.rs:591`), bridge `per_page = limit.min(500)`
(`crates/buzz-relay/src/api/bridge.rs:1660`) and bridge page from the raw JSON
`page`/`search_page`/`searchPage` field, defaulting to 1
(`bridge.rs:345-353`).

#### R11 — Authorization is explicitly NOT applied here

| What it enforces | Site |
|---|---|
| No membership check, no role check, no `#p` check, no owner gate, no author-only filter anywhere in the crate — the only visibility-affecting predicates are `community_id`, `deleted_at IS NULL`, the caller-supplied `ChannelScope`, and the storage-level NULL tsvector | entire `query.rs:216-323`; documented as such at `lib.rs:19-22` and `query.rs:3-9` |

Where authorization *is* applied (caller side, outside this crate):

| Check | Site |
|---|---|
| Re-fetch canonical events scoped by `(community, ids)` before use | `crates/buzz-relay/src/api/bridge.rs:1727-1731`, `crates/buzz-relay/src/handlers/req.rs:601-621` |
| Full NIP-01 filter re-match + channel-accessibility + reader authorization per hit (`search_hit_accepted`) | `crates/buzz-relay/src/api/bridge.rs:1594-1625`, invoked at `bridge.rs:1732` |
| Author-only kinds dropped | `crates/buzz-relay/src/api/bridge.rs:1735-1737` |
| WS path equivalent post-filter + channel check | `crates/buzz-relay/src/handlers/req.rs:685-701` |

Consequence if a caller forgets: the crate will happily return hits from channels
the reader cannot access whenever `ChannelScope::Any` is passed, and hits whose
non-pushed-down filter constraints (`#p`, `#e`, `#d`, `ids`) were never evaluated.
The relay's own comment states this exact failure mode for the `/query` bridge
(`crates/buzz-relay/src/api/bridge.rs:1582-1592`).
