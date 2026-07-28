## Module: buzz-search (`crates/buzz-search`)

### Security

#### SQL injection surface

| Question | Finding | Evidence |
|---|---|---|
| Is the FTS term parameterized? | Yes, in both modes. | `FullText`: `qb.push("websearch_to_tsquery('simple', "); qb.push_bind(search_text); qb.push(")")` (`query.rs:143-145`). `Prefix`: the raw text is a single `push_bind` (`query.rs:168`) consumed by in-SQL `regexp_split_to_table(...)`. |
| Is any user input interpolated into SQL text? | No. Every dynamic value is `push_bind`; `push` receives only `&'static`-style literal fragments. | binds at `query.rs:144`, `168`, `241`, `257`, `262`, `270`, `278`, `285`, `291`, `296`, `298`; no `format!`/`write!`/`+`-built SQL exists in `src/` |
| Does the shape of the statement depend on user data? | Predicate *presence* depends on `Option`/enum discriminants (`query.rs:248-293`), never on user-supplied strings. Each branch pushes a fixed fragment plus a bind. | `query.rs:248-293` |
| Test-only exception | `tests/fts_integration.rs` interpolates a locally generated schema name into DDL, explicitly opting in via `sqlx::AssertSqlSafe`. Name is `fts_test_<Uuid::new_v4().simple()>` — not caller-controlled. | `tests/fts_integration.rs:37`, `43-46`, `99-102` |

Conclusion: no first-order SQL injection surface in this crate.

#### tsquery injection via search operators

| Mode | Exposure | Mechanism |
|---|---|---|
| `FullText` | The term is interpreted by Postgres' `websearch_to_tsquery`, which is the sanitizing parser: it never raises syntax errors and treats stray operators as text. User operators can influence *matching* (quoted phrases, `or`, leading `-`) but cannot construct arbitrary tsquery structure outside websearch grammar, and cannot alter SQL. | `query.rs:143-145` |
| `Prefix` | Lexemes are produced by `to_tsvector('simple', token)` and each is wrapped in `quote_literal(...)` before concatenation into a tsquery, then cast `::tsquery`. The stated intent is exactly this: "`quote_literal` prevents tsquery syntax injection from punctuation/operators in the raw topbar input". Only the trailing token receives `:*`; the `&` conjunction is fixed by the code, so a user cannot inject `|`, `!`, `<->`, or parentheses into the query tree. | `query.rs:150-153`, `155-176` |
| Regression coverage | A test feeds `"operators ' : & | ( ) ! alpha be"` through `Prefix` and asserts one clean hit rather than an error or a widened match. | `tests/fts_integration.rs:441-481` |

Residual DoS-ish considerations (not injection): `Prefix` builds a tsquery whose
term count grows with the number of tokens in the input; the only bound is the
4096-char cap (`query.rs:134`, `185`), so a 4096-char whitespace-dense input can
produce a many-term conjunction. `FullText` inherits Postgres' own websearch
parser limits. `tsquery` conjunctions are AND-only in `Prefix`, so the result set
narrows rather than widens as terms grow.

#### Tenant isolation

| Question | Finding | Evidence |
|---|---|---|
| Does every query filter `community_id`? | Yes. The crate contains exactly one executed statement, and `WHERE community_id = <bind>` is written into the builder before any conditional branch. | `query.rs:240-241`, `300` |
| Can a caller omit it? | No. `SearchQuery.community: CommunityId` is not `Option`, has no default, and `CommunityId` cannot be parsed from client input (`crates/buzz-core/src/tenant.rs:37-45`). | `query.rs:76` |
| Exceptions | None — there is no second query method, no count method, no admin/bypass path. | whole `src/` |
| Regression coverage | Event inserted under community A, queried as B → asserted zero hits. | `tests/fts_integration.rs:201-259` |

Index-level note (storage, not this crate): tenant filtering relies on
community-leading btree indexes BitmapAnd-ed with the single-column GIN
(`migrations/0001_initial_schema.sql:270-278`). Correctness comes from the
predicate, not the index.

#### Authorization boundary (caller responsibility)

The crate performs **no** authorization. The visibility-affecting predicates it
applies are only: `community_id` (`query.rs:241`), `deleted_at IS NULL`
(`query.rs:242`), the caller-supplied `ChannelScope` (`query.rs:248-264`), and the
caller's `kinds`/`authors`/time filters (`query.rs:267-293`). Documented explicitly
at `lib.rs:15-18` and `query.rs:3-9` ("Search is never the access boundary — it
cannot widen visibility").

Where the boundary actually is:

| Step | Site |
|---|---|
| Caller maps its accessible-channel set into `ChannelScope`, short-circuiting when the reader has nothing accessible | `crates/buzz-relay/src/handlers/req.rs:484-501`, `517-524`; bridge equivalent `crates/buzz-relay/src/api/bridge.rs:1642-1651` |
| Hits are re-fetched by `(community, ids)` through buzz-db | `crates/buzz-relay/src/handlers/req.rs:601-621`, `crates/buzz-relay/src/api/bridge.rs:1727-1731` |
| Per-hit re-authorization: full NIP-01 filter re-match, channel-accessibility, reader authorization | `search_hit_accepted` at `crates/buzz-relay/src/api/bridge.rs:1594-1625`, called at `bridge.rs:1732`; WS equivalent `crates/buzz-relay/src/handlers/req.rs:685-701` |
| Author-only kinds dropped | `crates/buzz-relay/src/api/bridge.rs:1735-1737` |

If a caller forgets: with `ChannelScope::Any`, the crate returns hits from any
channel in the community regardless of membership; with any scope, constraints the
crate cannot push down (`#p`, `#e`, `#d`, `ids`) are simply unevaluated. The relay
documents this exact leak scenario for `/query` — an authorized-looking
`{"kinds":[30174],"#p":[self]}` search would otherwise return text-matching
envelopes belonging to another owner (`crates/buzz-relay/src/api/bridge.rs:1582-1592`).

Backstops that survive a caller mistake (defense in depth, both outside this crate):

| Backstop | Effect | Evidence |
|---|---|---|
| Storage-level NULL `search_tsv` for privacy-sensitive kinds | `@@` can never match NULL, so those rows are not candidates at all | `migrations/0001_initial_schema.sql:222-226`, `migrations/0005_agent_turn_metric_fts.sql:26-31`, `migrations/0014_push_lease_fts.sql:28-32`, `schema/schema.sql:211-215` |
| Fresh-install positive allowlist (empty DBs only) | only kinds `0, 9, 40002, 45001, 45003` are indexed | `migrations/0008_fresh_install_search_allowlist.sql:13-23` |
| Tripwire tests over `AUTHOR_ONLY_KINDS` and persistent `P_GATED_KINDS` | fail if a Rust privacy constant gains a kind the schema does not exclude | `tests/fts_integration.rs:1256-1361`, `1338-1447` |

#### Information leakage

| Vector | Assessment | Evidence |
|---|---|---|
| Result counts | No total/`found` is returned — only the page's hits. A caller can still infer existence by page length, but only within its own community and channel scope. | `query.rs:122-127` |
| Ranking | `SearchHit.rank` (`ts_rank_cd`) is returned for every hit and is derived from the matched row's tsvector. If a caller forwarded a hit's rank without re-authorizing, that leaks a relevance signal about content the reader may not read. In-tree callers discard everything but `event_id` (`crates/buzz-relay/src/handlers/req.rs:625-626`, `crates/buzz-relay/src/api/bridge.rs:1725`). | `query.rs:117`, `236` |
| Metadata in hits | `kind`, `pubkey`, `channel_id`, `created_at` are returned pre-authorization — same caller-discipline caveat as `rank`. | `query.rs:104-118` |
| Error messages | `SearchError::Db` forwards the underlying sqlx message; decode errors embed only a byte length, no row data (`query.rs:307`, `310`). Whether the relay surfaces it is the caller's choice (bridge wraps it into an internal error string, `crates/buzz-relay/src/api/bridge.rs:1719-1722`). | `error.rs:7` |
| Timing | No constant-time requirements apply; search timing is inherently data-dependent. | — |

#### Denial-of-service controls present

| Control | Value | Site |
|---|---|---|
| Search text length cap | 4096 chars | `query.rs:134`, `185` |
| Empty query rejected before any DB roundtrip | — | `query.rs:217-222` |
| `per_page` cap | 500 (`0 → 100`) | `query.rs:129-130`, `224-229` |
| `page` cap (bounds OFFSET) | 1000 | `query.rs:138`, `230` |
| NUL scrub (prevents Postgres error-path churn from adversarial text) | — | `query.rs:186`, test `tests/fts_integration.rs:972-1012` |

Not present: per-caller rate limiting, statement timeout, query cancellation, or
concurrency limit — all left to the pool/caller.

#### `unsafe` code

`#![deny(unsafe_code)]` at `crates/buzz-search/src/lib.rs:1`; no `unsafe` block
appears in any of the four `.rs` files. Verified by reading all of them.

#### Input-validation gaps

| Gap | Consequence | Site |
|---|---|---|
| `authors` entries are `Vec<u8>` of arbitrary length — no 32-byte check | A malformed pubkey simply matches nothing (`pubkey = ANY`), so it is a correctness/telemetry gap rather than a security hole | `query.rs:88`, `275-281` |
| `kinds` values are unbounded `i32`; negative or out-of-range kinds are accepted | matches nothing | `query.rs:86`, `267-273` |
| No `since <= until` validation | inverted window yields zero rows silently | `query.rs:283-293` |
| No cap on the *number* of channel ids / kinds / authors bound | a caller could bind a very large array; only caller-side discipline bounds it | `query.rs:255-281` |
| Char cap counts `chars`, not tokens or bytes | a 4096-char multi-token input in `Prefix` mode still builds a large conjunction (see tsquery section) | `query.rs:185` |
| Crate cannot verify the caller re-authorizes | the entire authorization model is external; nothing in-crate can detect a forgetful caller | `lib.rs:15-18` |
