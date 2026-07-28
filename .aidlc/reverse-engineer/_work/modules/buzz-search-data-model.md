## Module: buzz-search (`crates/buzz-search`)

### Data Model

The crate defines no persistent schema of its own. It declares five public types
(one service handle, two enums, three structs) and reads one table (`events`).
There are no `sqlx::FromRow` derives; rows are decoded field-by-field via
`Row::try_get` (`crates/buzz-search/src/query.rs:304-319`).

#### Files

| File | LOC | Contents |
|---|---|---|
| `crates/buzz-search/src/lib.rs` | 54 | crate lints, re-exports, `SearchService` |
| `crates/buzz-search/src/query.rs` | 352 | all query types, SQL construction, `search()`, 3 unit tests |
| `crates/buzz-search/src/error.rs` | 9 | `SearchError` |
| `crates/buzz-search/tests/fts_integration.rs` | 1448 | 18 Postgres-gated integration tests |

---

### `SearchService` — `crates/buzz-search/src/lib.rs:39-42`

| Field | Type | Visibility | Notes |
|---|---|---|---|
| `pool` | `sqlx::PgPool` | private | Only field. `#[derive(Debug, Clone)]` at `lib.rs:39`. |

Doc comment states it "Holds nothing the pool itself doesn't already own" and
exists as an injection point for the relay's `AppState`
(`crates/buzz-search/src/lib.rs:35-38`).

---

### `SearchQuery` — `crates/buzz-search/src/query.rs:73-99`

`#[derive(Debug, Clone)]` (`query.rs:73`). All fields `pub`.

| Field | Type | Line | Semantics as documented/used |
|---|---|---|---|
| `community` | `buzz_core::CommunityId` | `query.rs:76` | Server-resolved tenant. Required at the type level (no `Option`). Bound as the first SQL predicate (`query.rs:241`). |
| `q` | `String` | `query.rs:79` | NIP-50 search text. Empty/whitespace short-circuits before SQL (`query.rs:217-222`). |
| `channel_scope` | `ChannelScope` | `query.rs:84` | Channel constraint; see enum below. |
| `kinds` | `Option<Vec<i32>>` | `query.rs:86` | `None` **or empty vec** = no kind constraint (`query.rs:267-273`). |
| `authors` | `Option<Vec<Vec<u8>>>` | `query.rs:88` | 32-byte pubkeys as raw bytes. `None`/empty = no constraint (`query.rs:275-281`). Type does not enforce the 32-byte length. |
| `since` | `Option<i64>` | `query.rs:90` | Unix seconds; inclusive lower bound (`>=`, `query.rs:284`). |
| `until` | `Option<i64>` | `query.rs:92` | Unix seconds; inclusive upper bound (`<=`, `query.rs:290`). |
| `page` | `u32` | `query.rs:94` | 1-indexed; clamped to `1..=1000` (`query.rs:230`). |
| `per_page` | `u32` | `query.rs:96` | Clamped to `1..=500`; `0` becomes `100` (`query.rs:224-229`). |
| `mode` | `SearchMode` | `query.rs:98` | Selects the tsquery construction (`query.rs:239`, `140-177`). |

No `Default`, no builder, no constructor fn — callers struct-literal every field
(e.g. `crates/buzz-relay/src/handlers/req.rs:599-611`,
`crates/buzz-relay/src/api/bridge.rs:1687-1698`).

---

### `SearchHit` — `crates/buzz-search/src/query.rs:104-118`

`#[derive(Debug, Clone)]`. All fields `pub`.

| Field | Type | Line | Source column |
|---|---|---|---|
| `event_id` | `[u8; 32]` | `query.rs:107` | `events.id` (BYTEA), length-checked to 32 (`query.rs:306-308`) |
| `kind` | `i32` | `query.rs:109` | `events.kind` (`query.rs:314`) |
| `pubkey` | `[u8; 32]` | `query.rs:111` | `events.pubkey` (BYTEA), length-checked to 32 (`query.rs:309-311`) |
| `channel_id` | `Option<Uuid>` | `query.rs:113` | `events.channel_id`; `None` = channel-less (`query.rs:316`) |
| `created_at` | `i64` | `query.rs:115` | `EXTRACT(EPOCH FROM created_at)::bigint AS created_at_s` (`query.rs:235`, `317`) |
| `rank` | `f32` | `query.rs:117` | `ts_rank_cd(search_tsv, search_query.query) AS rank` (`query.rs:236`, `318`) |

Notably absent: `content`, `tags`, `sig`, `d_tag`. The struct is documented as
"just enough to drive that fetch and preserve relevance ordering"
(`query.rs:101-103`).

---

### `SearchResult` — `crates/buzz-search/src/query.rs:121-127`

| Field | Type | Line | Notes |
|---|---|---|---|
| `hits` | `Vec<SearchHit>` | `query.rs:124` | Page of hits, relevance-then-recency ordered |
| `page` | `u32` | `query.rs:126` | Clamped page actually used (`query.rs:230`, `322`); also set on the empty-query early return (`query.rs:220`) |

There is **no total-count / `found` field** — the crate returns no result cardinality
beyond the page itself.

---

### `ChannelScope` — `crates/buzz-search/src/query.rs:41-53`

`#[derive(Debug, Clone)]` (no `Copy`, no `PartialEq`).

| Variant | Line | Payload | Emitted SQL fragment | Emit site |
|---|---|---|---|---|
| `Any` | `query.rs:44` | — | *(nothing)* | `query.rs:249-251` |
| `ChannelLessOnly` | `query.rs:47` | — | `AND channel_id IS NULL` | `query.rs:252-254` |
| `Channels` | `query.rs:49` | `Vec<Uuid>` | `AND channel_id = ANY($n)` | `query.rs:255-259` |
| `ChannelsOrChannelLess` | `query.rs:52` | `Vec<Uuid>` | `AND (channel_id = ANY($n) OR channel_id IS NULL)` | `query.rs:260-264` |

Empty vectors are deliberately not special-cased: `Channels(vec![])` yields
zero rows and `ChannelsOrChannelLess(vec![])` is equivalent to
`ChannelLessOnly` (`query.rs:33-39`).

---

### `SearchMode` — `crates/buzz-search/src/query.rs:56-66`

`#[derive(Debug, Clone, Copy, PartialEq, Eq)]` (`query.rs:56`).

| Variant | Line | tsquery construction |
|---|---|---|
| `FullText` | `query.rs:59` | `websearch_to_tsquery('simple', $n)` (`query.rs:142-146`) |
| `Prefix` | `query.rs:65` | In-SQL token pipeline: `regexp_split_to_table($n, '\s+') WITH ORDINALITY` → `to_tsvector('simple', token)` → `tsvector_to_array` → `unnest ... WITH ORDINALITY` → `string_agg(quote_literal(lexeme) || CASE WHEN token_ord = max_token_ord THEN ':*' ELSE '' END, ' & ' ORDER BY token_ord, lex_ord)::tsquery`, `COALESCE`d to `''` (`query.rs:147-176`) |

---

### `SearchError` — `crates/buzz-search/src/error.rs:4-9`

| Variant | Line | Shape | `Display` |
|---|---|---|---|
| `Db` | `error.rs:8` | `Db(#[from] sqlx::Error)` | `"database error: {0}"` (`error.rs:7`) |

Single-variant enum; row-decode length failures are folded into it as
`sqlx::Error::Decode` (`query.rs:306-311`).

---

### SQL shape queried against `events`

One statement, built with `sqlx::QueryBuilder` (`query.rs:233-298`). Literal
text as constructed:

```sql
SELECT id, kind, pubkey, channel_id,
       EXTRACT(EPOCH FROM created_at)::bigint AS created_at_s,
       ts_rank_cd(search_tsv, search_query.query) AS rank
FROM events CROSS JOIN LATERAL (SELECT <mode tsquery> AS query) AS search_query
WHERE community_id = $1
  AND deleted_at IS NULL
  AND search_tsv @@ search_query.query
  [AND <channel scope>] [AND kind = ANY($n)] [AND pubkey = ANY($n)]
  [AND created_at >= to_timestamp($n)] [AND created_at <= to_timestamp($n)]
ORDER BY rank DESC, created_at DESC, id
LIMIT $n OFFSET $n
```

Columns referenced on `events`:

| Column | Role | Reference |
|---|---|---|
| `id` | selected, tiebreak in `ORDER BY` | `query.rs:234`, `295` |
| `kind` | selected + optional predicate | `query.rs:234`, `269` |
| `pubkey` | selected + optional predicate | `query.rs:234`, `277` |
| `channel_id` | selected + channel-scope predicate | `query.rs:234`, `253-263` |
| `created_at` | selected (epoch cast), since/until predicates, ordering | `query.rs:235`, `284`, `290`, `295` |
| `search_tsv` | rank input + `@@` probe | `query.rs:236`, `242` |
| `community_id` | tenant predicate (first) | `query.rs:240-241` |
| `deleted_at` | soft-delete exclusion | `query.rs:242` |

Underlying storage (read for confirmation only, not owned by this crate):
`events` is `PARTITION BY RANGE (created_at)` with PK
`(community_id, created_at, id)` (`migrations/0001_initial_schema.sql:191-238`),
`search_tsv` is `TSVECTOR GENERATED ALWAYS ... STORED`
(`migrations/0001_initial_schema.sql:222-226`), and the access path is
`CREATE INDEX idx_events_search_tsv ON events USING GIN (search_tsv)`
(`migrations/0001_initial_schema.sql:278`).
