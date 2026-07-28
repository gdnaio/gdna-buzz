## Module: buzz-db (`crates/buzz-db`)

### Conventions

#### 1. Crate-level lints and module layout

- `#![deny(unsafe_code)]` and `#![warn(missing_docs)]` at
  `crates/buzz-db/src/lib.rs:1-2`; every public item carries a doc comment.
- One module per persistence concern, each declared with a doc comment in
  `lib.rs` (`crates/buzz-db/src/lib.rs:12-51`): `admin_moderation`, `api_token`,
  `archived_identities`, `channel`, `dm`, `error`, `event`, `feed`, `git_repo`,
  `migration`, `moderation`, `partition`, `product_feedback`, `push`, `reaction`,
  `relay_members`, `replica_fence`, `thread`, `usage`, `user`, `workflow`
  (21 modules + `lib.rs`).
- Two-layer API: module-level free functions take `pool: &PgPool` (or
  `tx: &mut Transaction<'_, Postgres>`) plus a `CommunityId`; `Db` methods are
  thin delegating wrappers. Inline SQL on `Db` is the exception, not the rule.
- Section banners inside larger modules: `// -- Public structs ---`,
  `// -- Write operations ---`, `// -- Read operations ---`, `// -- Row mappers ---`
  (e.g. `crates/buzz-db/src/reaction.rs:10`, `:64`, `:276`;
  `crates/buzz-db/src/thread.rs:16`, `:108`, `:339`).
- Re-export policy: enums shared with the SDK live in `buzz-core` and are
  re-exported, not redefined — `pub use buzz_core::channel::{ChannelType, ChannelVisibility, MemberRole}`
  with the rationale in the comment (`crates/buzz-db/src/channel.rs:13-17`).

#### 2. Naming

| Pattern | Convention | Examples |
|---------|-----------|----------|
| Reads returning a single row | `get_*` / `find_*` (`find_*` when absence is normal) | `get_channel`, `get_event_by_id`, `find_dm_by_participants`, `find_by_owner_and_name` |
| Reads returning many rows | `list_*` / `query_*` (`query_*` when filters are involved) | `list_channels`, `list_relay_members`, `query_events`, `query_mentions` |
| Batched variants | `*_bulk` suffix | `get_members_bulk`, `get_users_bulk`, `get_member_counts_bulk`, `get_last_message_at_bulk`, `get_reactions_bulk` |
| Predicates | `is_*` / `has_*` | `is_member`, `is_relay_member`, `is_archived`, `is_agent_owner`, `has_allowlist_entries`, `has_join_policy_acceptance`, `has_read_pool` |
| Idempotent create | `ensure_*` | `ensure_user`, `ensure_configured_community`, `ensure_future_partitions` |
| Upsert with ordering semantics | `replace_*` / `upsert_*` / `accept_*` | `replace_addressable_event`, `replace_parameterized_event`, `upsert_workflow`, `accept_lease_event` |
| Queue/lease lifecycle | `claim_* / complete_* / retry_* / fail_* / release_* / reap_* / prune_*` | `claim_due_wakes`, `complete_match_batch`, `retry_wake`, `fail_wake`, `release_due_reminder`, `reap_exhausted_matches`, `prune_wake_outbox` |
| Row → struct mappers | private `row_to_*` | `row_to_stored_event`, `row_to_channel_record`, `row_to_member_record`, `row_to_report`, `row_to_ban`, `row_to_action`, `row_to_workflow_record`, `row_to_run_record`, `row_to_approval_record`, `row_to_claimed_wake`, `row_to_archived_identity`, `row_to_feedback` |
| Transaction-aware twins | `*_tx` suffix | `get_active_role_tx`, `get_channel_tx`, `add_reaction_tx`, `insert_event_with_thread_metadata_tx` |
| Insert parameter bags | `New*` / `*Params` structs instead of long argument lists | `NewReport`, `NewAction`, `NewProductFeedback`, `NewWake`, `ThreadMetadataParams`, `CreateApprovalParams`, `ChannelUpdate` |
| Outcome enums instead of booleans when >2 states | `*Outcome` / `*Result` | `ReserveOutcome`, `AcceptLeaseOutcome`, `ReplaceLeaseOutcome`, `EnqueueWakeOutcome`, `RevalidateWakeOutcome`, `ReactionEventInsertOutcome`, `RemoveResult`, `TransferResult`, `CreateCommunityWithOwnerResult` |
| Advisory-lock key namespaces | `'<domain>:'` prefixes, documented as mutually distinct | `'buzz_push_gate:'` (`crates/buzz-db/src/push.rs:21`), `'buzz_channel_ttl:'` (`migrations/0024_…:20`), `'buzz_audit:'` (referenced at `migrations/0023_…:20`) |
| SQL identifiers | `snake_case`; indexes `idx_<table>_<cols>` in `0001`–`0007`/`0017`, bare `<table>_<purpose>` for push tables in `0012`/`0015`/`0018` | `idx_events_community_channel_created` vs `push_wake_outbox_due` |

Boolean returns are consistently "did this call change state": `rows_affected() > 0`
(e.g. `crates/buzz-db/src/event.rs:701`, `crates/buzz-db/src/reaction.rs:110`,
`crates/buzz-db/src/git_repo.rs:180`) or `== 1` where exactly one row is the
contract (`crates/buzz-db/src/push.rs:1148`, `crates/buzz-db/src/event.rs:1430`).

#### 3. Error handling

Single error enum `DbError` (`crates/buzz-db/src/error.rs:7-52`) with
`thiserror`, plus `pub type Result<T> = std::result::Result<T, DbError>` at `:51`.

| Variant | Source | Meaning |
|---------|--------|---------|
| `Sqlx(#[from] sqlx::Error)` | `:11` | driver-level failure |
| `Migrate(#[from] sqlx::migrate::MigrateError)` | `:15` | migration failure |
| `AuthEventRejected` | `:21` | kind 22242 must not be stored |
| `EphemeralEventRejected(u16)` | `:25` | kinds 20000–29999 must not be stored |
| `ChannelNotFound(uuid::Uuid)` | `:29` | |
| `MemberNotFound(uuid::Uuid)` | `:33` | |
| `NotFound(String)` | `:37` | generic |
| `AccessDenied(String)` | `:41` | permission/state refusal |
| `Serde(#[from] serde_json::Error)` | `:45` | JSON round-trip |
| `InvalidData(String)` | `:49` | malformed input or malformed stored value |
| `InvalidTimestamp(i64)` | `:53` | timestamp could not be interpreted |

Separate probe-only error type: `replica_fence::ProbeError`
(`crates/buzz-db/src/replica_fence.rs:363-380`) with `Writer`,
`MaskedActivity { masked }`, `ReplicaLsnUnavailable`; all three are treated
identically (fail closed) by the probe loop at `:492-502`.

Conventions:
- **No `unwrap()`/`expect()` on fallible DB results in production paths.** Row
  decoding uses `row.try_get(...)?`. The only non-test `expect`/`unwrap` calls are
  infallible-slice conversions in lock-key derivation
  (`crates/buzz-db/src/push.rs:224`, `:230`), a `expect("one outcome per request")`
  on a locally guaranteed vector length (`crates/buzz-db/src/push.rs:601`), and
  `expect("length checked")` after an explicit length check
  (`crates/buzz-db/src/thread.rs:328`).
- Postgres error codes are matched explicitly where behaviour depends on them:
  `42P17` overlap in the partition manager (`crates/buzz-db/src/partition.rs:139-141`),
  `23505` + constraint name and the generic `23xxx` family in push
  (`crates/buzz-db/src/push.rs:392-410`), `23514` check-violation in the fence
  verifier (`crates/buzz-db/src/replica_fence.rs:206`).
- Best-effort side indexes never fail the caller: mention inserts are logged and
  swallowed (`crates/buzz-db/src/lib.rs:1086-1089`, `:1392-1395`, `:1424-1427`,
  `:3428-3431`, `:3610-3613`, `:3812-3815`).
- Enum/status strings are parsed with `FromStr` returning
  `DbError::InvalidData(format!("unknown … : {other}"))` — never a silent default
  (`crates/buzz-db/src/workflow.rs:61-71`, `:103-116`, `:148-160`).
- `try_get(...).unwrap_or(None)` is used deliberately for columns that may be
  absent from a given projection, and is documented as such
  (`crates/buzz-db/src/channel.rs:990-999`).

#### 4. Query-construction style

| Style | When used | Examples |
|-------|-----------|----------|
| Static SQL string + positional `.bind()` | the default | most of the crate |
| `sqlx::QueryBuilder` with `push_bind` / `separated(", ")` / `push_values` | variable-length `IN (…)` lists and optional predicates | `crates/buzz-db/src/event.rs:344-449`, `:591-698`; `crates/buzz-db/src/feed.rs:91-119`; `crates/buzz-db/src/lib.rs:146-163`; `crates/buzz-db/src/channel.rs:1337-1349`; `crates/buzz-db/src/event.rs:836-848`, `:957-966` |
| `format!` + `sqlx::AssertSqlSafe`, all values still bound | dynamic SET/ORDER/placeholder shapes that `QueryBuilder` can't express, and DDL | 15 sites: `crates/buzz-db/src/channel.rs:870`, `:957`, `:1107`; `crates/buzz-db/src/thread.rs:430`, `:631`; `crates/buzz-db/src/user.rs:148`; `crates/buzz-db/src/usage.rs:281`, `:323`; `crates/buzz-db/src/partition.rs:130`; `crates/buzz-db/src/lib.rs:5235`, `:5256`, `:6009`, `:6014`; `crates/buzz-db/src/replica_fence.rs:613`, `:638` (last five are test-only) |
| `sqlx::query_as::<_, (…tuple…)>` | small fixed projections | `crates/buzz-db/src/user.rs:61`, `crates/buzz-db/src/usage.rs:43` and siblings, `crates/buzz-db/src/lib.rs:3345` |
| `sqlx::query_scalar::<_, T>` | single-value reads | `crates/buzz-db/src/lib.rs:519`, `:687`; `crates/buzz-db/src/push.rs:299`; `crates/buzz-db/src/relay_members.rs:429` |
| `ANY($n)` array binds | fixed-arity list predicates | `crates/buzz-db/src/channel.rs:565`, `:625`; `crates/buzz-db/src/push.rs:912`, `:980` |
| Nullable-filter idiom `($n::type IS NULL OR col = $n)` | optional filters without dynamic SQL | `crates/buzz-db/src/admin_moderation.rs:106-112`; `crates/buzz-db/src/moderation.rs:222` |
| Two static query variants instead of one dynamic string | when only a single optional predicate exists | `crates/buzz-db/src/channel.rs:669-708`; `crates/buzz-db/src/dm.rs:239-306` |

Ordering/pagination conventions: `ORDER BY created_at DESC, id ASC` for event
reads (`crates/buzz-db/src/event.rs:435-443`); composite keyset cursors rather
than OFFSET for channel windows and thread pages
(`crates/buzz-db/src/thread.rs:380-386`, `:595-602`); a `LIMIT n+1` probe rather
than a second COUNT for `has_more` (`crates/buzz-db/src/thread.rs:640-643`).
Every list query has a bound: an explicit `LIMIT` literal (1000), a clamped
parameter, or a constant (`FEED_MAX_LIMIT`, `LIST_MAX_LIMIT`, `MAX_PAGE_SIZE`).

#### 5. Transactions and locking

- `pool.begin()` … `tx.commit()` / `tx.rollback()`: **33** `begin()` sites and
  **30** `commit()` sites in `crates/buzz-db/src/**` — the difference is
  early-return paths that `rollback()` deliberately
  (e.g. `crates/buzz-db/src/lib.rs:3366`, `crates/buzz-db/src/event.rs:1222`,
  `crates/buzz-db/src/relay_members.rs:437`).
- Transactions are used wherever two writes must agree: channel create +
  owner bootstrap, event + thread metadata + counters, reaction + kind:7 event,
  membership + policy acceptance, community + owner, lease + source event.
- Advisory locks are acquired **first** inside a transaction, in one documented
  global order per subsystem (`crates/buzz-db/src/push.rs:239-243`,
  `migrations/0024_event_ttl_refresh_shared_lock.sql:20-24`).
- `Db::begin_transaction` (`crates/buzz-db/src/lib.rs:648-650`) exposes a
  `Transaction<'static, Postgres>` to callers, justified by `PgPool` being
  `Arc`-backed.
- Session-scoped advisory locks are held on a **detached** connection so a
  locked session is never returned to the pool
  (`crates/buzz-db/src/lib.rs:511-535`, guard type at `:203-219`).

#### 6. Row mapping

No `#[derive(sqlx::FromRow)]` anywhere in the crate (zero matches). Every row is
decoded manually, which keeps enum columns readable as `::text` and lets
projections vary per query. Common shapes:

```rust
rows.into_iter().map(row_to_report).collect()                 // moderation.rs:236
row.map(row_to_report).transpose()                            // moderation.rs:260
row.map(|row| { Ok(Record { … row.try_get("x")? }) }).transpose()  // lib.rs:673
```

Byte columns are `Vec<u8>` in structs and hex-encoded only at the presentation
boundary (`crates/buzz-db/src/admin_moderation.rs:172-176`,
`crates/buzz-db/src/product_feedback.rs:112-113`). `pubkey_hex` in
`event_mentions` is the one place hex is the storage form, always lowercased on
write (`crates/buzz-db/src/lib.rs:140`) and on read predicates
(`crates/buzz-db/src/event.rs:361`).

#### 7. Testing patterns

Counts (all tests live in in-file `#[cfg(test)] mod tests`; there is no
`tests/` directory in the crate):

| Metric | Count |
|--------|-------|
| `#[test]` (pure, no infrastructure) | **81** |
| `#[tokio::test]` | **122** |
| `#[ignore]`-gated | **121** |
| Non-ignored `#[tokio::test]` | **1** (`read_falls_back_to_writer_when_no_replica_configured`, `crates/buzz-db/src/lib.rs:5361`, uses `connect_lazy` so it never touches the network) |
| Files with a `mod tests` | 19 of 22 |

Per file:

| File | `#[test]` | `#[tokio::test]` | `#[ignore]` |
|------|-----------|------------------|-------------|
| `workflow.rs` | 24 | 7 | 7 |
| `feed.rs` | 22 | 3 | 3 |
| `event.rs` | 14 | 12 | 12 |
| `migration.rs` | 7 | 3 | 3 |
| `user.rs` | 5 | 8 | 8 |
| `dm.rs` | 4 | 0 | 0 |
| `partition.rs` | 3 | 0 | 0 |
| `replica_fence.rs` | 2 | 3 | 3 |
| `lib.rs` | 0 | 25 | 24 |
| `push.rs` | 0 | 14 | 14 |
| `relay_members.rs` | 0 | 10 | 10 |
| `thread.rs` | 0 | 10 | 10 |
| `usage.rs` | 0 | 8 | 8 |
| `channel.rs` | 0 | 7 | 7 |
| `git_repo.rs` | 0 | 4 | 4 |
| `moderation.rs` | 0 | 4 | 4 |
| `api_token.rs` | 0 | 2 | 2 |
| `archived_identities.rs` | 0 | 1 | 1 |
| `product_feedback.rs` | 0 | 1 | 1 |
| `admin_moderation.rs`, `error.rs`, `reaction.rs` | 0 | 0 | 0 |

Conventions:
- Infra tests are gated `#[ignore = "requires Postgres"]` (or
  `"requires migrated Postgres"` at `crates/buzz-db/src/product_feedback.rs:124`).
- Test DB URL resolution: a `const TEST_DB_URL` default plus
  `BUZZ_TEST_DATABASE_URL` → `DATABASE_URL` (most modules) or `TEST_DATABASE_URL`
  (`lib.rs`, `replica_fence.rs`) — see `buzz-db-configuration.md`.
- Every infra test mints its own community via a local
  `make_test_community` / `make_community` helper with a UUID-derived host, so
  tests are isolated on a shared database (e.g.
  `crates/buzz-db/src/event.rs:1443-1454`).
- Cross-tenant isolation tests deliberately use **identical** ids/shapes in two
  communities (`crates/buzz-db/src/event.rs:1560`,
  `crates/buzz-db/src/channel.rs:1553`, `crates/buzz-db/src/lib.rs:4890`).
- Replica-routing tests create two scratch databases with **divergent** fixtures
  so every assertion observes which pool actually served the query
  (`crates/buzz-db/src/lib.rs:5216-5262`, `:5379`, `:5464`), and include
  explicit "counterfactual" assertions that pin the hazard an over-open fence
  would cause (`:5647-5661`, `:5741-5755`).
- Concurrency tests use `tokio::spawn` + `sleep` + `is_finished()` to assert a
  statement *blocks*, then `tokio::time::timeout` to assert it completes
  (`crates/buzz-db/src/lib.rs:4351-4380`, `crates/buzz-db/src/event.rs:1503-1533`).
- The migration lint tests hand-roll a small SQL parser (statement splitting
  respecting `'` and `$$`, paren matching, top-level CSV split) and assert
  tenant-isolation properties over the concatenation of **all** migrations
  (`crates/buzz-db/src/migration.rs:120-370`, `:635-688`).
- Pure unit tests concentrate on the code that has no DB dependency: validators
  (`partition.rs:153-181`), tag extraction (`event.rs:1927-2040`), hashing/ordering
  (`dm.rs:520+`), SQL-shape assertions built from `QueryBuilder`
  (`feed.rs:766-861`), and enum round-trips (`workflow.rs:1199+`).
