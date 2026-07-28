## Module: buzz-db (`crates/buzz-db`)

### Technical Debt

#### 1. Schema drift: `migrations/` vs `schema/schema.sql`

`schema/schema.sql` (1016 lines) is described in its own header as the "source of
truth for fresh database setup" (`schema/schema.sql:3`), but nothing in the crate
reads it — the runner embeds `migrations/` only
(`crates/buzz-db/src/migration.rs:11`). It has drifted substantially. Verified
absences (grep count 0 in `schema/schema.sql`):

| Object | Created by | Missing from `schema/schema.sql` |
|--------|-----------|----------------------------------|
| table `git_repo_names` + `idx_git_repo_names_owner` | `migrations/0002_git_repo_names.sql:20`, `:29` | yes |
| column `communities.icon` | `migrations/0003_community_icon.sql:12` | yes |
| index `idx_events_tags_gin` (GIN `jsonb_path_ops`) | `migrations/0004_events_tags_gin.sql:21` | yes |
| table `parameterized_event_watermarks` | `migrations/0007_nip_rs_retention.sql:14` | yes |
| index `idx_event_mentions_community_event` | `migrations/0007_nip_rs_retention.sql:26` | yes |
| function+trigger `guard_nip_rs_watermark` / `trg_events_nip_rs_watermark` | `migrations/0009:11,70`, replaced `0010:4`, `0011:62` | yes |
| function+trigger `purge_soft_deleted_nip_rs` | `migrations/0009:77,104`, replaced `0011:123` | yes |
| function+trigger `guard_event_mention_live` | `migrations/0009:111,133` | yes |
| function+trigger `guard_nip_rs_hard_delete` | `migrations/0011:45,56` | yes |
| function+trigger `purge_soft_deleted_buzz_mesh_status` | `migrations/0019:21,41` | yes |
| table `product_feedback` (+2 indexes, + allowlist row) | `migrations/0017_product_feedback.sql:5` | yes |
| table `join_policy_acceptances` | `migrations/0020_join_policy_acceptances.sql:4` | yes |

`schema/schema.sql` also encodes an `events.search_tsv` expression that matches
**neither** end state produced by the migrations:

```sql
    search_tsv  TSVECTOR GENERATED ALWAYS AS (
        CASE WHEN kind IN (1059, 30300, 30350, 30622, 44100, 44101, 44200) THEN NULL::tsvector
             ELSE to_tsvector('simple', content)
        END
    ) STORED,
```
(`schema/schema.sql:196-200`, with the comment "Keep in sync with migrations
(final state: 0001 + 0005 + 0009)" at `:194` — the relevant migration is `0014`,
not `0009`). Migrations produce either the brownfield negative list
(`0001`+`0005`+`0014` ⇒ `{1059, 30300, 30622, 44100, 44101, 44200}` ∪ `{30350}`)
or, for an empty database, the positive allowlist from `0008`
(`{0, 9, 40002, 45001, 45003}` minus `30350`). The divergence between fresh and
brownfield installs is itself latent debt: the same relay binary can be running
two different search policies (asserted, not fixed, by
`crates/buzz-db/src/migration.rs:1075-1109`), and the remediation is an
out-of-band operator script named at
`migrations/0008_fresh_install_search_allowlist.sql:3-4`
(`scripts/maintenance/nip_rs_search_allowlist.sql`).

`schema/schema.sql` does contain a few things migrations also have (moderation
tables, push lease/queue tables, the `0021`/`0023`/`0024` functions), so it is a
partially-updated snapshot rather than a stale copy — which makes it more likely
to be trusted by mistake. There is no test asserting the two files agree.

#### 2. Complexity hotspots

File sizes (measured; prod = lines before `#[cfg(test)]`):

| File | Total | Prod | Test | Test share |
|------|------:|-----:|-----:|-----------:|
| `lib.rs` | 6106 | 3916 | 2190 | 36% |
| `event.rs` | 2465 | 1428 | 1037 | 42% |
| `push.rs` | 2388 | 1261 | 1127 | 47% |
| `workflow.rs` | 2276 | 1197 | 1079 | 47% |
| `channel.rs` | 1896 | 1415 | 481 | 25% |
| `thread.rs` | 1730 | 809 | 921 | 53% |
| `migration.rs` | 1248 | 97 | 1151 | 92% |
| `relay_members.rs` | 953 | 555 | 398 | 42% |
| `moderation.rs` | 895 | 629 | 266 | 30% |
| `feed.rs` | 825 | 248 | 577 | 70% |
| `usage.rs` | 723 | 356 | 367 | 51% |
| `user.rs` | 674 | 400 | 274 | 41% |
| `replica_fence.rs` | 662 | 507 | 155 | 23% |
| `dm.rs` | 558 | 518 | 40 | 7% |
| `api_token.rs` | 523 | 326 | 197 | 38% |
| `reaction.rs` | 419 | 419 | 0 | 0% |
| `git_repo.rs` | 381 | 181 | 200 | 52% |
| `admin_moderation.rs` | 231 | 231 | 0 | 0% |
| `archived_identities.rs` | 222 | 126 | 96 | 43% |
| `product_feedback.rs` | 189 | 119 | 70 | 37% |
| `partition.rs` | 183 | 151 | 32 | 17% |
| `error.rs` | 55 | 55 | 0 | 0% |
| **total** | **25602** | **14944** | **10658** | **42%** |

`lib.rs` at 6106 lines is the single biggest structural problem: it is
simultaneously the crate root, the `Db` façade with **215** delegating methods,
the community-registry repository, the replaceable-event write engine
(`replace_addressable_event`, `replace_parameterized_event`,
`publish_nip43_membership_locked`), the allowlist repository, the api-token
partial repository, and a 2190-line test module. Nothing in the crate's own
module convention explains why community/allowlist/token/replacement SQL lives
in the root rather than in `community.rs` / `allowlist.rs` / `api_token.rs` /
`replace.rs`.

Longest functions (brace-matched):

| Lines | Location | Notes |
|------:|----------|-------|
| 214 | `crates/buzz-db/src/lib.rs:3628` `replace_parameterized_event` | classification + two ordering sources + conditional hard/soft delete + watermark upsert in one body |
| 208 | `crates/buzz-db/src/event.rs:302` `query_events` | 14 optional predicates; `col_prefix` string threaded through 20 `format!` calls |
| 185 | `crates/buzz-db/src/thread.rs:565` `get_channel_window` | SQL assembly + `has_more` probe + cursor capture + batched participant hydration |
| 183 | `crates/buzz-db/src/push.rs:213` `accept_lease_event` | three locks + five rejection paths + two inserts |
| 170 | `crates/buzz-db/src/push.rs:619` `enqueue_wakes` | four-phase set-wise protocol with three intermediate hash maps |
| 168 | `crates/buzz-db/src/event.rs:1004` `insert_event_with_thread_metadata_tx` | 5-deep nesting (`if was_inserted` → `if let Some(meta)` → `if rows_affected` → `if let Some(pid)` → `if let Some(root_id)` → `if root_id != pid`) |
| 142 | `crates/buzz-db/src/thread.rs:345` `get_thread_replies` | manual positional-parameter bookkeeping (`param_idx`) mirrored in two places |
| 140 | `crates/buzz-db/src/event.rs:557` `count_events` | **near-duplicate** of `query_events`' predicate block |
| 125 | `crates/buzz-db/src/lib.rs:3306` `replace_addressable_event` | |
| 124 | `crates/buzz-db/src/thread.rs:116` `insert_thread_metadata` | **near-duplicate** of the `_tx` variant in `event.rs` |

Duplication that has to be maintained in lockstep:

1. `query_events` (`crates/buzz-db/src/event.rs:302`) and `count_events`
   (`:557`) repeat the same ~10 predicate blocks with the same `col_prefix`
   trick; `count_events` silently omits the composite-cursor branch and the
   `LIMIT/OFFSET`, so the two can drift.
2. `thread::insert_thread_metadata` (`crates/buzz-db/src/thread.rs:116-247`) and
   `event::insert_event_with_thread_metadata_tx`'s inner block
   (`crates/buzz-db/src/event.rs:1097-1173`) contain the same stub-insert +
   counter-bump logic, written twice.
3. `thread::increment_reply_count` (`crates/buzz-db/src/thread.rs:251-289`) is a
   third copy of just the counter half — and is dead (see §3).
4. `Db` wrappers repeat every module signature; 20 of them carry
   `#[allow(clippy::too_many_arguments)]` on both sides of the boundary
   (e.g. `crates/buzz-db/src/lib.rs:1437` and `crates/buzz-db/src/channel.rs:86`).
5. The push-eligible kind allowlist `{7, 9, 1059, 40007, 46010}` exists in three
   places: `migrations/0018_push_match_queue.sql:25`,
   `migrations/0023_push_match_gate.sql:26`, and
   `crates/buzz-db/src/push.rs:58` — with only a comment
   ("Keep this allowlist identical to the relay's validated NIP-PL descriptor")
   holding them together.
6. The FTS kind lists are inlined in four migrations plus `schema/schema.sql`,
   with the sync obligation stated in prose
   (`migrations/0001_initial_schema.sql:214-221`).
7. The NIP-RS exact-cardinality predicate is implemented once in Rust
   (`crates/buzz-db/src/lib.rs:3672-3687`) and three times in plpgsql
   (`migrations/0011_nip_rs_exact_tag_cardinality.sql:66-88`, `:126-148`, and the
   `DELETE` classifier at `:12-40`).
8. `ChannelRecord` is mapped by two separate row mappers with slightly different
   column expectations: `crates/buzz-db/src/channel.rs:983` (full projection) and
   `crates/buzz-db/src/dm.rs:481` (DM projection that never selects
   `ttl_seconds`/`ttl_deadline` and relies on `unwrap_or(None)`).

#### 3. Dead / unreachable code

| Item | Evidence |
|------|----------|
| `thread::increment_reply_count` | carries `#[allow(dead_code)]` and a doc comment saying the real path is inlined elsewhere — `crates/buzz-db/src/thread.rs:243-289` |
| `workflow::create_workflow` | "(No current callers.)" — `crates/buzz-db/src/workflow.rs:275` |
| `workflow::update_workflow` | "(No current callers.)" — `crates/buzz-db/src/workflow.rs:619` |
| `workflow::update_workflow_status` | "(No current callers.)" — `crates/buzz-db/src/workflow.rs:653` |
| `workflow::set_workflow_enabled` | "(No current callers.)" — `crates/buzz-db/src/workflow.rs:683` |
| `workflow::delete_workflow` | "(No current callers.)" — `crates/buzz-db/src/workflow.rs:714` |
| `reaction::get_reactions`' `_cursor` parameter | "reserved for future keyset pagination (currently unused)" — `crates/buzz-db/src/reaction.rs:279-286`. The public signature advertises pagination it does not implement. |
| `event::D_TAG_MAX_LEN` | declared but never compared against anything — `crates/buzz-db/src/event.rs:120-124` |
| `Db::update_token_last_used` | pure alias for `touch_api_token` — `crates/buzz-db/src/lib.rs:2377-2384` |
| `communities.signing_key` | column exists, no accessor in this crate — `migrations/0001_initial_schema.sql:56` |
| `channels.max_members`, `channels.nip29_group_id`, `users.okta_user_id`, `users.metadata_event_id`, `users.deactivated_at`, `users.agent_type`, `users.capabilities` | surfaced in records but never written by any function in this crate (only read) |
| `thread_metadata.parent_event_created_at` / `root_event_created_at` | written on insert but never used in a WHERE/JOIN by any read in this crate |

#### 4. Tables with schema but no code

`subscriptions`, `delivery_log`, `audit_log`, `rate_limit_violations`, and all six
`push_gateway_*` tables have DDL in `migrations/` but **no** read or write path in
buzz-db. For `delivery_log` this is doubly odd: the crate manages its partitions
(`crates/buzz-db/src/partition.rs:12`) and its PK is the single hand-maintained
exception in the tenant-isolation lint
(`crates/buzz-db/src/migration.rs:497-501`), yet nothing in the crate ever inserts
a row. `ARCHITECTURE.md:793` describes it as "(partitioned; Rust module pending)".
`subscriptions` carries a full filter/delivery model that nothing populates.

#### 5. Test coverage gaps

Overall the crate is heavily tested (203 test functions, 42% of its lines), but
the distribution is uneven and **121 of 122** async tests are `#[ignore]`-gated,
so `cargo test` with no database exercises only the 81 pure `#[test]`s plus one
lazy-pool wiring test.

| Gap | Evidence |
|-----|----------|
| `reaction.rs` has **no** `mod tests` at all — the three-way upsert semantics are covered only indirectly, from `event.rs` and `lib.rs` | `crates/buzz-db/src/reaction.rs` (419 lines, 0 tests) |
| `admin_moderation.rs` has **no** tests — including the keyset-cursor SQL and the `bounded_limit` clamp | `crates/buzz-db/src/admin_moderation.rs` (231 lines, 0 tests) |
| `error.rs` has no tests (low risk: pure `Display` impls) | `crates/buzz-db/src/error.rs` |
| `dm.rs` has 4 pure tests and **0** database tests; participant-hash collision behaviour, hide/unhide, and the 9-participant ceiling are unexercised against Postgres | `crates/buzz-db/src/dm.rs:520+` |
| `channel.rs` has the lowest test share of the big modules (25%); `add_member`'s role matrix has no test for the private-channel elevated-grant refusal, and `update_channel`'s dynamic-SET builder has no test | `crates/buzz-db/src/channel.rs:1417+` (7 tests) |
| `partition.rs`: validators are unit-tested but `ensure_future_partitions`/`ensure_partition` have **no** database test, including the `42P17` overlap-tolerance branch | `crates/buzz-db/src/partition.rs:153-182` |
| `usage.rs`'s `active_user_counts`/`active_channel_counts` interval interpolation has no test asserting the produced SQL | `crates/buzz-db/src/usage.rs:254-341` |
| `api_token.rs`: only 2 tests; the 10-token quota boundary and expiry-aware "active" definition are not directly asserted | `crates/buzz-db/src/api_token.rs:328+` |
| No test asserts `schema/schema.sql` matches the migration end state | — |
| No test asserts the three copies of the push kind allowlist agree (unlike the moderation vocabulary, which *is* asserted at `crates/buzz-db/src/migration.rs:640-645`) | — |
| Tests share one database by default and rely on unique hosts for isolation; one test explicitly documents that a shared fixed pubkey picks up rows leaked by sibling ignored tests | `crates/buzz-db/src/lib.rs:4879-4884` |

#### 6. Migration-history debt

- **24 checksum-frozen files.** Because `sqlx::migrate!` pins checksums, four of
  the 24 migrations exist purely to patch earlier ones:
  `0010` and `0011` rewrite `0009`'s function bodies; `0023` rewrites `0018`'s;
  `0024` rewrites `0022`'s. Reading the current behaviour of
  `guard_nip_rs_watermark`, `purge_soft_deleted_nip_rs`,
  `enqueue_push_match_job`, or `refresh_channel_ttl_after_event_insert` requires
  reading up to three files and knowing which wins.
- `events.search_tsv` is dropped and re-added **three** times after `0001`
  (`0005`, `0008` conditionally, `0014`), each time rebuilding the GIN index. On a
  populated database `0005` rewrites the whole heap column.
- `0007` performs irreversible `DELETE`s of user payloads during startup
  migration (`migrations/0007_nip_rs_retention.sql:88-140`), guarded only by the
  pre-flight ambiguity check in `crates/buzz-db/src/migration.rs:34-96`.
- `0008` behaves differently depending on whether the table is empty, permanently
  bifurcating the deployment population.
- `0011` deletes rows from `parameterized_event_watermarks`
  (`migrations/0011_nip_rs_exact_tag_cardinality.sql:7-43`).
- `0004` builds a GIN index on a partitioned parent without `CONCURRENTLY`, which
  the file itself flags as requiring a deploy window
  (`migrations/0004_events_tags_gin.sql:13-17`).
- The lint harness that guarantees tenant isolation lives in `#[cfg(test)]`
  (`crates/buzz-db/src/migration.rs:99-830`) — it is a test, not a runtime or
  CI-independent gate, and its "operator-global" allowlist is a **hard-coded Rust
  array** (`crates/buzz-db/src/migration.rs:334-352`) that must be updated
  alongside the `_operator_global_tables` INSERTs; a new allowlisted table that
  someone forgets to add to the array will make the lint fail closed (safe), but
  a table added to the array without the DB row will silently exempt itself.

#### 7. Performance debt (self-documented)

| Issue | Evidence |
|-------|----------|
| `get_reactions_bulk` runs **one query per event** | "For typical message-list sizes (<=100 events) this is acceptable; a single-query approach … can be added later if needed" — `crates/buzz-db/src/reaction.rs:371-375` |
| `list_dms_for_user` runs **one participants query per DM** | `crates/buzz-db/src/dm.rs:308-334` |
| `query_events`' `ORDER BY created_at DESC, id ASC` has no covering index for the trailing column; Postgres sorts in memory | "No existing index covers this trailing column… If query performance degrades, add a composite index" — `crates/buzz-db/src/event.rs:435-441` |
| `usage.rs` event-derived aggregates are recurring partition scans | "At scale these can become recurring partition scans; if that becomes a problem, move them to a maintained rollup table" — `crates/buzz-db/src/usage.rs:6-10` |
| `get_members_bulk` has no pagination | "For large channel sets, consider pagination." — `crates/buzz-db/src/channel.rs:600-604` |
| `get_accessible_channel_ids` has **no LIMIT** (deliberately — a test asserts 1001 rows are returned) so the result set grows with the community | `crates/buzz-db/src/channel.rs:638-667`; test `crates/buzz-db/src/channel.rs:1751-1783` |
| `api_token::list_tokens_by_owner` has no `LIMIT` | `crates/buzz-db/src/api_token.rs:208-266` |
| FTS uses the `'simple'` configuration (no stemming/stopwords) as a compatibility choice, flagged for revisit | `migrations/0001_initial_schema.sql:198-201` |
| Two prior perf regressions are documented as fixed but leave the shape fragile: unindexed `tags @>` containment cost ~1.7 s/page (`migrations/0004_events_tags_gin.sql:5-12`) and the `0022` TTL row lock took commit latency from 0.07 ms to ~15 ms at 200 QPS (`migrations/0024_event_ttl_refresh_shared_lock.sql:1-9`) |

#### 8. API-shape debt

- **215 delegating methods on one `Db` struct.** Every new module function needs a
  matching wrapper; several wrappers are already inconsistent — `api_token`
  functions take `Uuid` while their `Db` wrappers take `CommunityId` and
  dereference (`crates/buzz-db/src/lib.rs:2283-2292`), whereas every other module
  takes `CommunityId` directly.
- Naming collisions between generic `Db` methods and specific ones:
  `Db::archive`/`Db::unarchive`/`Db::is_archived`/`Db::list_archived` are about
  *identities* (`crates/buzz-db/src/lib.rs:3238-3279`), while
  `Db::archive_channel` and `Db::archive_community_owned_by` are about other
  entities — the unqualified names are the least specific.
- Return-type inconsistency for "not found": `get_channel` returns
  `Err(ChannelNotFound)` (`crates/buzz-db/src/channel.rs:294`) while
  `get_user`/`get_relay_member`/`get_ban` return `Ok(None)`; `get_workflow`
  returns `Err(NotFound)` while `find_by_owner_and_name` returns `Ok(None)`.
- `ApiTokenRecord` exposes `token_hash` to every caller and relies on a doc
  comment to prevent leaking it (`crates/buzz-db/src/api_token.rs:201-206`).
- `EventQuery` has 18 fields with several mutually exclusive combinations
  validated at runtime rather than by type
  (`crates/buzz-db/src/event.rs:21-105`, checks at `:304-317`).

#### 9. Deprecated API usage

None found. No `#[deprecated]` items are defined or consumed; no
`#[allow(deprecated)]` appears in the crate. Dependency versions are current for
the workspace (sqlx 0.9, nostr 0.44, thiserror 2, sha2 0.11 — `Cargo.toml:52`,
`:61`, `:85`, `:96`), and the code uses the post-0.8 sqlx `AssertSqlSafe` API
rather than the older implicit-`&str` path, which is itself evidence of a recent
upgrade having been done properly.

#### 10. Risk summary

| Risk | Severity | Why |
|------|----------|-----|
| `schema/schema.sql` mistaken for authoritative | high | it claims to be the source of truth, is missing 12 objects, and encodes a `search_tsv` expression no migration produces |
| Fresh-vs-brownfield FTS policy divergence | medium | same binary, two search behaviours, remediation is a manual script |
| Kind-allowlist triplication (push) and FTS lists in frozen SQL | medium | a new privacy-sensitive kind silently becomes searchable / a new push kind silently stops waking devices |
| `lib.rs` size and mixed responsibilities | medium | 3916 production lines in the crate root, including the most intricate write paths |
| `query_events`/`count_events` duplication | medium | a filter fixed in one and not the other changes COUNT vs REQ semantics |
| 121/122 async tests gated behind `#[ignore]` | medium | the correctness argument for locking, triggers, and tenancy only holds if CI actually runs the gated suite |
| `delivery_log`/`subscriptions` schema without code | low | dead weight, plus a hand-maintained lint exception for a table nothing writes |
| `reaction.rs` / `admin_moderation.rs` with zero tests | low–medium | `admin_moderation` is the one module allowed to skip tenant scoping |
