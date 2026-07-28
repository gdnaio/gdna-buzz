## Module: buzz-db (`crates/buzz-db`)

### Security

#### 1. Memory safety

`#![deny(unsafe_code)]` at `crates/buzz-db/src/lib.rs:1`. A search for `unsafe`
across `crates/buzz-db/src/**` returns three hits, all non-code: the lint
attribute itself, a doc comment at `crates/buzz-db/src/thread.rs:342`, and a code
comment at `crates/buzz-db/src/workflow.rs:1674`. **No `unsafe` blocks exist.**

#### 2. SQL injection surface

No user value is ever concatenated into SQL. All 15 `sqlx::AssertSqlSafe` sites
interpolate only: (a) compile-time literals, (b) generated `$n` placeholder
indices, (c) values already validated by an allowlist/validator, or (d) values
whose Rust type cannot carry SQL. Ten sites are production code, five are
test-only.

| Site | Interpolated content | Why it is safe | Values still bound? |
|------|----------------------|----------------|---------------------|
| `crates/buzz-db/src/channel.rs:870` (`get_accessible_channels`) | `membership_clause` chosen from two string literals (`:822-826`); the base template; an appended `AND c.visibility::text = $3` | no runtime data reaches the string | yes — `$1` community, `$2` pubkey, `$3` visibility (`:871-878`) |
| `crates/buzz-db/src/channel.rs:957` (`get_users_bulk`) | `"$2, $3, …"` list generated from `2..(len+2)` (`:942-945`) | digits only, derived from a length | yes — every pubkey bound in a loop (`:958-960`) |
| `crates/buzz-db/src/channel.rs:1107` (`update_channel`) | SET fragments from fixed literals + generated indices (`:1080-1105`) | column names are literals; only indices vary | yes — `:1110-1122` |
| `crates/buzz-db/src/thread.rs:430` (`get_thread_replies`) | `${param_idx}` indices only (`:387-400`) | digits only | yes — `:431-448` |
| `crates/buzz-db/src/thread.rs:631` (`get_channel_window`) | `${param_idx}` indices **and** `kind IN ({list})` where `list` is `&[u32]` joined via `to_string()` (`:617-627`) | `u32` cannot encode SQL; this is the only caller-supplied data interpolated in production | partially — community/channel/cursor/limit bound (`:632-643`); the kind list is inlined as decimal integers |
| `crates/buzz-db/src/user.rs:148` (`update_user_profile`) | SET fragments + generated indices (`:112-142`) | as above | yes — `:149-160` |
| `crates/buzz-db/src/usage.rs:281` (`active_user_counts`) | `INTERVAL '{interval_sql}'` where `interval_sql: &'static str` (`:252`) | `&'static str` signature keeps it a compile-time literal; documented "must be a trusted literal" at `:249-251` and re-stated on the `Db` wrapper at `crates/buzz-db/src/lib.rs:621-623` | no other value; this query takes no binds |
| `crates/buzz-db/src/usage.rs:323` (`active_channel_counts`) | same `&'static str` interval | same | same |
| `crates/buzz-db/src/partition.rs:130` (`ensure_partition`) | `partition_name`, `table_name`, two date strings — DDL, which cannot be parameterized | allowlist + three validators re-checked immediately before, `:84-105` | n/a (DDL) |
| `crates/buzz-db/src/lib.rs:5235`, `:5256`, `:6009`, `:6014` | scratch database names from `Uuid::new_v4().simple()` | test-only (`#[cfg(test)]`), UUID hex | n/a |
| `crates/buzz-db/src/replica_fence.rs:613`, `:638` | `CREATE ROLE`/`DROP ROLE` with a UUID-derived role name | test-only | n/a |

Additional non-interpolating dynamic-SQL surfaces, all fully parameterized:
`QueryBuilder` with `push_bind`/`separated`/`push_values`
(`crates/buzz-db/src/event.rs:344`, `:591`, `:836`, `:957`;
`crates/buzz-db/src/feed.rs:91`, `:150`, `:207`; `crates/buzz-db/src/lib.rs:146`;
`crates/buzz-db/src/channel.rs:1337`), and the e-tag JSONB containment predicate
which binds a `serde_json::Value` rather than string-building JSON
(`crates/buzz-db/src/event.rs:410-419`).

**DDL construction** exists in exactly one production place — the partition
manager. Its three-layer defence:

1. Table allowlist `PARTITIONED_TABLES = ["events", "delivery_log"]`
   (`crates/buzz-db/src/partition.rs:12`), re-checked inside `ensure_partition`
   even though the only caller iterates the same constant (`:84-88`).
2. `validate_partition_suffix` — non-empty, ASCII digits and `_` only
   (`:58-60`), enforced at `:89-93`.
3. `validate_date_str` — exactly 10 bytes, `-` at indices 4 and 7, digits
   elsewhere (`:63-73`), enforced at `:94-105`.

Negative unit tests pin all three, including the explicit injection payloads
`"2026_03; DROP TABLE events--"` and `"2026-03-01; DROP TABLE events--"`
(`crates/buzz-db/src/partition.rs:156-174`) and that `api_tokens`/`users` are
**not** in the allowlist (`:176-181`).

LIKE-metacharacter escaping: `escape_like` escapes `\`, `%`, `_` and the query
uses `ESCAPE '\'`, so a search for `"%"` cannot become a full-table wildcard
(`crates/buzz-db/src/user.rs:207-222`, `:246-256`).

#### 3. Secret handling

| Secret | Storage form | Where |
|--------|--------------|-------|
| Workflow approval tokens | SHA-256 digest only; `create_approval` takes the **raw** token and hashes it internally, so plaintext never reaches the DB layer | `crates/buzz-db/src/workflow.rs:33-36`, `:915-946`; hash-only lookup/update paths `:973-994`, `:1059-1092` |
| API tokens | SHA-256 digest supplied by the caller; DB CHECK enforces 32 bytes | `crates/buzz-db/src/api_token.rs:8-9`; `migrations/0001_initial_schema.sql:488` |
| DM participant sets | SHA-256 over sorted, deduped pubkeys | `crates/buzz-db/src/dm.rs:48-60` |
| Push endpoint identity | `endpoint_hash` (SHA-256, 32 bytes) plus an opaque `endpoint_grant`; both are **copied from the stored lease** on wake enqueue so a caller cannot redirect a wake | `migrations/0012_push_leases.sql:11-12`; `crates/buzz-db/src/push.rs:502-511`, `:650-701` |
| APNs device tokens | never in a buzz-db-managed tenant table: `push_gateway_installations.token_ciphertext` + `token_fingerprint` are operator-global and have no Rust module here | `migrations/0015_push_gateway_authority.sql:12-26` |
| AUTH events (kind 22242) | **never stored** — they carry bearer tokens | `crates/buzz-db/src/error.rs:19-22`; `crates/buzz-db/src/event.rs:248-250` |
| Community signing key | `communities.signing_key BYTEA` — no read/write helper in this crate | `migrations/0001_initial_schema.sql:56` |
| Default DB URL literal | `postgres://buzz:buzz_dev@localhost:5432/buzz` in `DbConfig::default()` and in test constants, each annotated `// sadscan:disable np.postgres.1` where scanned | `crates/buzz-db/src/lib.rs:240`; `crates/buzz-db/src/replica_fence.rs:512` |

`ApiTokenRecord` intentionally carries `token_hash`, and the doc comment tells
callers to strip it before returning data to clients
(`crates/buzz-db/src/api_token.rs:201-206`).

Privacy-by-storage: NIP-44-encrypted / p-gated kinds are excluded from the FTS
vector so they are unsearchable at the storage layer, not just filtered at query
time (`migrations/0001_initial_schema.sql:207-226`,
`migrations/0005_agent_turn_metric_fts.sql:1-33`,
`migrations/0014_push_lease_fts.sql:1-9`). Reporter identity, private moderation
reasons, and `matched_principal` are documented as mod/audit-only and are never
placed on a public surface by this crate (`migrations/0006_moderation.sql:9-12`,
`:88-113`).

#### 4. TOCTOU / race safety

| Hazard | Mitigation | Evidence |
|--------|------------|----------|
| Inviter loses their role between the role check and the member insert | whole sequence in one transaction | `crates/buzz-db/src/channel.rs:362-456` |
| Actor's role changes between check and removal | one transaction; last-owner count also inside it | `crates/buzz-db/src/channel.rs:461-529` |
| Two concurrent reaction adds both see "absent" | single `INSERT … ON CONFLICT DO UPDATE … WHERE removed_at IS NOT NULL` | `crates/buzz-db/src/reaction.rs:66-112` |
| Two concurrent token mints both pass a count check | count is a subquery inside the INSERT | `crates/buzz-db/src/api_token.rs:91-126` |
| Two concurrent agent-owner mints | conditional `UPDATE … WHERE agent_owner_pubkey IS NULL` (first-mint-wins) | `crates/buzz-db/src/user.rs:283-303` |
| Concurrent replaceable-event writers | per-`(community, kind, pubkey, coordinate)` transaction advisory lock; ordering checked inside the same tx | `crates/buzz-db/src/lib.rs:3320-3336`, `:3654-3664` |
| Stale NIP-43 snapshot overwriting a newer one | advisory lock acquired **before** reading membership, so the whole read-build-write cycle is serialized | `crates/buzz-db/src/lib.rs:3498-3524` |
| Owner removal race (read role, then delete) | conditional `DELETE … AND role <> 'owner'` / `AND role = $3`; the follow-up read is only for error classification | `crates/buzz-db/src/relay_members.rs:196-285` |
| Stale-owner overwrite during ownership transfer | owner rows locked `FOR UPDATE` + `expected_owner_pubkey` verified in-tx → `OwnerConflict` | `crates/buzz-db/src/relay_members.rs:427-448` |
| Two transfers/creates to the same recipient both passing the 3-community cap | per-recipient FNV-1a advisory lock shared by both paths | `crates/buzz-db/src/relay_members.rs:384-391`, `:418-424`; `crates/buzz-db/src/lib.rs:869-874` |
| Two pods firing the same cron instant | unique claim insert `(community, workflow, scheduled_for)` | `crates/buzz-db/src/workflow.rs:496-534` |
| Two pods delivering the same reminder | `UPDATE … WHERE delivered_at IS NULL`; release is compare-and-clear on the pod's own stamp | `crates/buzz-db/src/event.rs:1370-1431` |
| Two decisions on one approval | `AND status = 'pending'` in the UPDATE | `crates/buzz-db/src/workflow.rs:1043-1051` |
| Repo-name claim race | `INSERT … ON CONFLICT DO NOTHING RETURNING`; a vanished conflicting row is treated as taken, never granted | `crates/buzz-db/src/git_repo.rs:96-139` |
| Lost push wake when a lease activates concurrently with an event insert | per-community advisory lock: events take it **shared**, activations **exclusive**; plus a 120 s `received_at` backfill inside the activation tx | `migrations/0023_push_match_gate.sql:1-42`; `crates/buzz-db/src/push.rs:15-66`, `:239-243`, `:464-468` |
| Stale TTL prefetch (permanent→ephemeral transition mid-ingest) | deferred trigger reads `ttl_seconds` under a shared per-channel advisory key; `update_channel` takes the same key exclusive | `migrations/0024_event_ttl_refresh_shared_lock.sql:25-57`; `crates/buzz-db/src/channel.rs:1131-1147` |
| Orphan mention after a concurrent NIP-RS hard delete | mention insert takes `FOR KEY SHARE` on the live event, skips if gone; the Rust delete path deletes event-then-mentions in a fixed order | `migrations/0009_nip_rs_database_guards.sql:111-137`; `crates/buzz-db/src/lib.rs:3759-3781` |
| NIP-RS resurrection window between an existence check and uniqueness enforcement | the watermark trigger returns NULL for an exact coordinate replay instead of probing the payload | `migrations/0010_nip_rs_exact_replay_guard.sql:34-47` |
| Old relay binary hard-deleting a NIP-RS coordinate | BEFORE DELETE trigger refuses unless the transaction opted in via a transaction-local GUC | `migrations/0011_nip_rs_exact_tag_cardinality.sql:45-60`; opt-in `crates/buzz-db/src/lib.rs:3742-3747` (verified transaction-local by test `crates/buzz-db/src/lib.rs:4194-4218`) |
| Below-fence row committed by a long-held transaction | deferred constraint trigger re-evaluates `clock_timestamp()` **inside commit**, so holding the tx open cannot outrun the floor | `migrations/0021_created_at_fence_floor.sql:20-42`, `:44-74` |
| An armed GUC with no enforcing trigger opening the fence | probe gated on catalog **and** behavioural verification; migration also fails closed | `crates/buzz-db/src/lib.rs:449-462`; `crates/buzz-db/src/replica_fence.rs:145-330`; `crates/buzz-db/src/migration.rs:25` |
| A session advisory lock leaking back into the pool | the lock-owning connection is `detach()`ed and closed on guard drop | `crates/buzz-db/src/lib.rs:511-535` |

Lock-ordering discipline is explicit: push takes address → author → gate
(`crates/buzz-db/src/push.rs:239-243`); the TTL key is acquired at commit, after
any push-gate shared lock, and no path takes both domains exclusively
(`migrations/0024_event_ttl_refresh_shared_lock.sql:20-24`); the NIP-RS delete
path fixes event-before-mentions ordering
(`crates/buzz-db/src/lib.rs:3759-3762`).

#### 5. Tenant isolation

Storage-level guarantees (schema): every non-allowlisted table has
`community_id NOT NULL` and every PK/UNIQUE/FK/unique index on it leads with
`community_id`, asserted over the concatenation of all 24 migrations by
`crates/buzz-db/src/migration.rs:635-670`. `channels.community_id` is immutable
(trigger `migrations/0001_initial_schema.sql:115-128`) and migrations may not
re-tenant it (`crates/buzz-db/src/migration.rs:527-556`, `:672-688`).

Query-level guarantees: `CommunityId` is a required parameter of essentially
every read/write. Notable belt-and-braces cases:

- `event_mentions` joins carry the **community tuple**
  (`e.community_id = m.community_id AND e.id = m.event_id`) — a bare
  `e.id = m.event_id` would leak cross-community mentions
  (`crates/buzz-db/src/event.rs:348-361`, `:576-586`;
  `crates/buzz-db/src/feed.rs:92-100`; schema note
  `migrations/0001_initial_schema.sql:281-284`).
- API-token lookup filters `community_id` in the WHERE clause **in addition to**
  the unique index, so the property holds even if storage uniqueness is relaxed
  (`crates/buzz-db/src/api_token.rs:129-158`; `crates/buzz-db/src/lib.rs:2337-2340`).
- Reminder claim/release bind `community_id` because the same event id may exist
  in two communities (`crates/buzz-db/src/event.rs:1359-1368`, `:1394-1400`).
- `communities_of_channels` deliberately **omits** unknown channels from its map
  so the relay's `MissingLookup → ImplBug → CoverageBreach` chain stays
  non-vacuous (`crates/buzz-db/src/lib.rs:1041-1049`, test `:5053-5085`).

Deliberate, documented cross-community reads (audited exceptions):

| Path | Scope | Justification in code |
|------|-------|----------------------|
| `admin_moderation::{list_reports, get_report, list_feedback, get_feedback}` | deployment-global | "the only moderation repository allowed to omit a `CommunityId`" — `crates/buzz-db/src/admin_moderation.rs:1-8` |
| `product_feedback::list` | deployment-global | operator inbox; `community_id` is provenance only — `crates/buzz-db/src/product_feedback.rs:3-6`, `migrations/0017_product_feedback.sql:23-24` |
| all of `usage.rs` (11 fns) | deployment-global aggregates | Prometheus rollups — `crates/buzz-db/src/usage.rs:1-14` |
| `Db::list_communities_owned_by` | filtered by pubkey only | "operator-plane helper… callers must gate it on deployment-level operator auth" — `crates/buzz-db/src/lib.rs:710-716` |
| owner-count checks (`SELECT count(*) FROM relay_members WHERE pubkey=$1 AND role='owner'`) | intentionally cross-community | the 3-community cap is a per-pubkey property — `crates/buzz-db/src/lib.rs:894-899`; `crates/buzz-db/src/relay_members.rs:459-467` |
| `Db::{community_of_channel, communities_of_channels}` | resolve tenancy from a row | inputs are internal channel ids; the result **is** the community label — `crates/buzz-db/src/lib.rs:1006-1077` |
| `Db::{archive,unarchive}_community_owned_by`, `lookup_community_by_host*`, `get/set_community_icon`, `ensure_configured_community`, `create_community_with_owner` | act on `communities` itself | operator-global registry table |
| `workflow::list_all_enabled_workflows` | scheduler scan | returns each row's `community_id` and filters archived communities — `crates/buzz-db/src/workflow.rs:449-478` |
| `workflow::prune_scheduled_workflow_fires_before` | global DELETE by `claimed_at` | janitor concern — `migrations/0001_initial_schema.sql:464-466` |
| `channel::reap_expired_ephemeral_channels` | global UPDATE | joins `communities`, skips archived, returns `(community_id, host, channel_id)` — `crates/buzz-db/src/channel.rs:1387-1417` |
| `event::query_due_reminders` | global scan | joins `communities`, skips archived, returns per-row `community_id` + `host` — `crates/buzz-db/src/event.rs:1293-1339` |
| `push::{claim_due_match_batch, reap_exhausted_matches}` | global | the claim CTE selects ONE community then scopes everything else — `crates/buzz-db/src/push.rs:840-882`, `:925-941` |
| `relay_members::backfill_from_allowlist` | community-scoped, but reads `information_schema` | startup migration helper — `crates/buzz-db/src/relay_members.rs:516-548` |
| `Db::backfill_d_tags` | **global `UPDATE events`, no community filter** | idempotent maintenance over legacy NIP-33 rows — `crates/buzz-db/src/lib.rs:2806-2824`. This is the one tenant-table **write** with no community predicate; it only fills a NULL derived column. |

Cross-tenant isolation is regression-tested with identical shapes in two
communities: events (`crates/buzz-db/src/event.rs:1560`), channels
(`crates/buzz-db/src/channel.rs:1553`), users
(`crates/buzz-db/src/channel.rs:1499`), reactions
(`crates/buzz-db/src/lib.rs:4890`), allowlist (`crates/buzz-db/src/lib.rs:4948`),
feeds (`crates/buzz-db/src/feed.rs:311`, `:365`, `:415`), relay membership
(`crates/buzz-db/src/relay_members.rs:600`), archived identities
(`crates/buzz-db/src/archived_identities.rs:150`), git repo names
(`crates/buzz-db/src/git_repo.rs:334`), reminders
(`crates/buzz-db/src/event.rs:2183`), and product feedback
(`crates/buzz-db/src/product_feedback.rs:125`).

#### 6. Least privilege

- The crate assumes the relay's role can create partitions
  (`CREATE TABLE … PARTITION OF`), set GUCs, and take advisory locks. It does
  **not** issue `GRANT`/`REVOKE`, create roles, or manage extensions beyond
  `CREATE EXTENSION IF NOT EXISTS pgcrypto` in migration 0001.
- The fence probe needs `pg_monitor`-level visibility into `pg_stat_activity`.
  An unprivileged role sees NULL `state`/`xact_start` (and NULL `backend_type`),
  and the probe **fails closed** with
  `MaskedActivity { masked }` rather than silently `MIN()`-ing hidden rows away
  — the classification is explicitly ordered so masked rows are detected
  *before* any `backend_type` filter (`crates/buzz-db/src/replica_fence.rs:369-374`,
  `:414-441`). A dedicated test creates an unprivileged login role and asserts
  the failure (`crates/buzz-db/src/replica_fence.rs:595-644`).
- A "replica" URL pointed at a primary is rejected: the replay-LSN check is
  gated on `pg_is_in_recovery()` and NULL is an error, not "fresh"
  (`crates/buzz-db/src/replica_fence.rs:441-463`, test `:648-660`).
- Operational bypasses of the floor guard are named and bounded: sessions without
  the `buzz.created_at_floor` GUC (pg_restore, manual backfills) and
  `session_replication_role = replica` restores are outside the proof by design
  and require holding the fence closed for their duration
  (`migrations/0021_created_at_fence_floor.sql:26-42`;
  `crates/buzz-db/src/replica_fence.rs:38-42`).

#### 7. Input validation

| Validated | Where |
|-----------|-------|
| pubkey length = 32 bytes (channels, members, DMs) | `crates/buzz-db/src/channel.rs:96-101`, `:184-189`, `:355-360`; `crates/buzz-db/src/dm.rs:118-124`; DB CHECK `migrations/0001_initial_schema.sql:171` |
| `p`-tag pubkeys must be 64 lowercase-able hex chars; malformed ones are dropped with a debug log rather than poisoning the batch | `crates/buzz-db/src/lib.rs:127-142` |
| Channel name canonicalised + non-empty | `crates/buzz-db/src/channel.rs:103-106`, `:1063-1068` |
| Nil channel UUID rejected | `crates/buzz-db/src/channel.rs:191-195` |
| DM participant count 2–9 | `crates/buzz-db/src/dm.rs:107-117`, `:366-370` |
| `channel_add_policy` vocabulary | `crates/buzz-db/src/user.rs:374-380` |
| Mutually exclusive / dependent query params (`before_id` needs `until`; `global_only` vs `channel_id`) | `crates/buzz-db/src/event.rs:304-317` |
| Stored `kind` must fit `u16` on read; otherwise `InvalidData` | `crates/buzz-db/src/event.rs:468-470` |
| Timestamps convertible from Unix seconds; otherwise `InvalidTimestamp` | `crates/buzz-db/src/event.rs:265-267`; `crates/buzz-db/src/lib.rs:3316-3318` |
| Enum/status strings parsed strictly | `crates/buzz-db/src/workflow.rs:61-71`, `:103-116`, `:148-160`; `crates/buzz-db/src/moderation.rs:580-590` |
| Huddle-link candidate content capped at 512 bytes and 32 rows before JSON parsing | `crates/buzz-db/src/event.rs:130-136`, `:210-217` |
| NIP-RS coordinate shape: lowercase-hex 32 chars, exactly one `d`, exactly one `["t","read-state"]` | `crates/buzz-db/src/lib.rs:3672-3687`; DB mirror `migrations/0011_nip_rs_exact_tag_cardinality.sql:66-88` |
| Byte-length and range CHECKs on push/moderation/feedback columns | `migrations/0012_push_leases.sql:5-21`, `migrations/0015_push_gateway_authority.sql:14-21`, `migrations/0006_moderation.sql:20-26`, `migrations/0017_product_feedback.sql:7-13` |

Known validation gaps (not defects per se, but the boundary is elsewhere):

1. **`d_tag` length is not enforced on write.** `D_TAG_MAX_LEN = 1024` is defined
   (`crates/buzz-db/src/event.rs:120-124`) but never compared against anything in
   this crate; `extract_d_tag` explicitly preserves the full value and defers
   length enforcement to the ingest layer
   (`crates/buzz-db/src/event.rs:2010-2019`).
2. **`max_active_leases` is caller-supplied**, so the push quota ceiling is a
   relay-side policy value, not a DB invariant
   (`crates/buzz-db/src/push.rs:213-221`).
3. **Git repo-name quota is enforced by the caller**, not by the module
   (`crates/buzz-db/src/git_repo.rs:88-92`).
4. **`usage_active_*_counts` trust their `&'static str` interval.** The type
   prevents runtime user input, but nothing rejects a malformed literal at
   compile time (`crates/buzz-db/src/usage.rs:249-253`).
5. **`api_tokens.scopes`/`channel_ids` are unconstrained JSONB** — schema has no
   CHECK; malformed JSON surfaces as `InvalidData` on read
   (`crates/buzz-db/src/lib.rs:3876-3899`).
6. **`get_channel_window`'s kind filter is inlined, not bound** — safe because
   the parameter is `&[u32]`, but it is the only production interpolation of
   caller data (`crates/buzz-db/src/thread.rs:617-627`).
7. **`Db::backfill_d_tags` has no community predicate** — a global maintenance
   write (`crates/buzz-db/src/lib.rs:2810-2824`).
8. **`api_token::list_tokens_by_owner` has no `LIMIT`** — unbounded result set
   for a single owner (`crates/buzz-db/src/api_token.rs:208-266`).
