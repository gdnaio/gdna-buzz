## Module: buzz-db (`crates/buzz-db`)

### Business Rules

Rules are grouped by concern. "Trigger" is the call/statement path that
evaluates the rule.

---

#### 1. Event admission

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| 1.1 | Kind 22242 (`KIND_AUTH`) is never stored — it carries bearer tokens | `crates/buzz-db/src/event.rs:248-250` (also `:1024-1026` in the tx path) → `DbError::AuthEventRejected` (`crates/buzz-db/src/error.rs:20-22`) | `insert_event`, `insert_event_with_thread_metadata` |
| 1.2 | Ephemeral kinds 20000–29999 are never stored (`is_ephemeral`, from buzz-core) | `crates/buzz-db/src/event.rs:251-253`, `:1027-1029` → `DbError::EphemeralEventRejected(kind)` (`crates/buzz-db/src/error.rs:24-26`) | same |
| 1.3 | Event dedup is `ON CONFLICT DO NOTHING` on `(community_id, created_at, id)`; the caller learns whether the row was new via `was_inserted` | `crates/buzz-db/src/event.rs:270-274`, `:294-299` | every insert path |
| 1.4 | Dedup is per community — the same signed event may exist in two communities | PK `(community_id, created_at, id)` `migrations/0001_initial_schema.sql:234`; test `crates/buzz-db/src/event.rs:1560-1607` | insert into two communities |
| 1.5 | `d_tag` is materialized only for NIP-33 kinds (30000–39999); a missing `d` tag stores `""` | `crates/buzz-db/src/event.rs:141-162` | every insert |
| 1.6 | `not_before` is materialized only for kind 30300 and only when the tag parses as `i64` | `crates/buzz-db/src/event.rs:166-181` | every insert |
| 1.7 | Rows whose stored JSON cannot be rebuilt into a `nostr::Event` are **skipped** (warn), not fatal | `crates/buzz-db/src/event.rs:494-500`; same policy in `thread.rs:454-457`, `:657-660` | any read path |
| 1.8 | A channel-bearing `events` row whose `created_at` is more than `buzz.created_at_floor` seconds before **commit** time aborts the transaction (`check_violation`). Channel-NULL rows are structurally exempt; sessions without the GUC are unguarded | `migrations/0021_created_at_fence_floor.sql:44-68` (function), `:70-74` (deferred constraint trigger); GUC armed at `crates/buzz-db/src/lib.rs:394-405` | COMMIT of any insert/update of `created_at`/`channel_id` |

#### 2. Replaceable-event ordering (NIP-16 / NIP-33)

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| 2.1 | Replacement is serialized per `(community, kind, pubkey, [coordinate])` by a transaction-scoped advisory lock keyed with FNV-1a | `crates/buzz-db/src/lib.rs:63-89`, used at `:3320-3336`, `:3502-3512`, `:3654-3664` | `replace_addressable_event`, `publish_nip43_membership_locked`, `replace_parameterized_event` |
| 2.2 | Canonical ordering: newer `created_at` wins; on a same-second tie the **lowest** event id wins. Incoming events that are dominated are rejected with `(event, false)` and the tx rolls back | `crates/buzz-db/src/lib.rs:3358-3372` (addressable), `:3719-3737` (parameterized) | both replace paths |
| 2.3 | Addressable replacement keys on `(kind, pubkey, channel_id)` using `IS NOT DISTINCT FROM` for NULL safety | `crates/buzz-db/src/lib.rs:3345-3352`, `:3378-3386` | `replace_addressable_event` |
| 2.4 | NIP-33 replacement keys on `(kind, pubkey, d_tag)` **globally** — `channel_id` is stored for query scoping only and is not part of the replacement key | documented and implemented at `crates/buzz-db/src/lib.rs:3607-3626`, `:3697-3706` | `replace_parameterized_event` |
| 2.5 | If the insert hits `ON CONFLICT` after the old row was already soft-deleted, the whole tx rolls back so the previous replaceable event is not lost | `crates/buzz-db/src/lib.rs:3407-3417`, `:3798-3805` | duplicate id replay |
| 2.6 | Reads of the latest replaceable row use the same `ORDER BY created_at DESC, id ASC LIMIT 1` as the write path (defensive against historical duplicate survivors) | `crates/buzz-db/src/event.rs:894-921`; comment at `crates/buzz-db/src/lib.rs:3338-3340` | `get_latest_global_replaceable` |

#### 3. NIP-RS (kind 30078 read-state) retention

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| 3.1 | An event is classified NIP-RS only under **exact** tag cardinality: kind 30078, exactly one `d` tag whose value equals the coordinate, `d_tag` matching `^read-state:[0-9a-f]{32}$` (lowercase only), and exactly one `["t","read-state"]` tag | `crates/buzz-db/src/lib.rs:3672-3687`; DB mirror `migrations/0011_nip_rs_exact_tag_cardinality.sql:66-88` | `replace_parameterized_event` and every `events` INSERT |
| 3.2 | Superseded NIP-RS payloads are **hard-deleted** (not soft-deleted), and their `event_mentions` rows are deleted after the event row (fixed lock order to avoid deadlock with the mention fence) | `crates/buzz-db/src/lib.rs:3739-3782` | NIP-RS replacement |
| 3.3 | A compact ordering watermark `(community, kind, pubkey, d_tag) → (created_at, event_id)` survives payload deletion and blocks stale resurrection even after a NIP-09 coordinate delete | table `migrations/0007_nip_rs_retention.sql:14-22`; Rust probe `crates/buzz-db/src/lib.rs:3709-3723`; upsert `:3784-3796`; DB trigger `migrations/0011_…:89-121` | replacement + any raw INSERT |
| 3.4 | Exact replay of an already-watermarked coordinate is a silent no-op (`RETURN NULL`) rather than an error — so concurrent physical deletion cannot open a resurrection window | `migrations/0010_nip_rs_exact_replay_guard.sql:34-47`; carried forward at `migrations/0011_…:104-116` | BEFORE INSERT trigger |
| 3.5 | Any other stale NIP-RS insert raises `check_violation` with `'stale NIP-RS event rejected by durable watermark'` | `migrations/0011_nip_rs_exact_tag_cardinality.sql:118-119` | BEFORE INSERT trigger |
| 3.6 | A legacy soft-delete of a NIP-RS row is converted to a physical purge (event + mentions) by an AFTER UPDATE trigger, covering pre-fix relay binaries during rolling deploys | `migrations/0009_nip_rs_database_guards.sql:77-108`; corrected body `migrations/0011_…:123-160` | `UPDATE events SET deleted_at` |
| 3.7 | Direct `DELETE` of a read-state-coordinate row is **refused** unless the transaction has opted in with `set_config('buzz.nip_rs_hard_delete','on',true)` | `migrations/0011_nip_rs_exact_tag_cardinality.sql:45-60`; opt-in at `crates/buzz-db/src/lib.rs:3742-3747` | any `DELETE FROM events` on a matching row |
| 3.8 | Mention inserts for kind 30078 take `FOR KEY SHARE` on the live event and are silently dropped if the event is already gone | `migrations/0009_nip_rs_database_guards.sql:111-137` | `INSERT INTO event_mentions` |
| 3.9 | Migration `0007` refuses to run when a pre-0007 database holds kind-30078 rows with ambiguous `d`/`t` cardinality; the operator must repair first | `crates/buzz-db/src/migration.rs:34-96` (`reject_legacy_nip_rs_cardinality_ambiguity`) | `run_migrations` on a DB at version < 7 |

#### 4. Mesh-status retention (kind 30003, `buzz-mesh-member-status:*`)

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| 4.1 | A kind:30003 coordinate whose `d_tag` starts with `buzz-mesh-member-status:` and which carries `["k","buzz-mesh-status"]` hard-deletes its superseded payload (heartbeat, no historical value) | `crates/buzz-db/src/lib.rs:3688-3695`, `:3739-3782` | `replace_parameterized_event` |
| 4.2 | Legacy soft-deletes of those rows are purged physically by trigger | `migrations/0019_mesh_status_retention.sql:21-45` | `UPDATE events SET deleted_at` |

#### 5. Channel membership and roles

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| 5.1 | `created_by` / member `pubkey` must be exactly 32 bytes | `crates/buzz-db/src/channel.rs:96-101`, `:184-189`, `:355-360`; DB CHECK on `users.pubkey` `migrations/0001_initial_schema.sql:171` | `create_channel*`, `add_member` |
| 5.2 | Channel name is canonicalised (`buzz_core::channel::canonical_channel_name`) and must be non-empty | `crates/buzz-db/src/channel.rs:103-106`, `:191-194`, `:1063-1068` | `create_channel*`, `update_channel` |
| 5.3 | A client-supplied channel id may not be the nil UUID (reserved for global fan-out) | `crates/buzz-db/src/channel.rs:191-195` | `create_channel_with_id` |
| 5.4 | The creator is bootstrapped as `'owner'` in the **same transaction** as the channel insert | `crates/buzz-db/src/channel.rs:110-152` and `:220-241` | `create_channel*` |
| 5.5 | Private channel joins require an `invited_by` who is an active member; the creator adding themselves is the only bootstrap exception | `crates/buzz-db/src/channel.rs:368-385` | `add_member` |
| 5.6 | Elevated roles (`owner`, `admin`) may only be granted by an active `owner`/`admin` — checked for both private and open channels; open-channel self-join always yields `Member` | `crates/buzz-db/src/channel.rs:392-397` (private), `:399-417` (open) | `add_member` |
| 5.7 | The entire read-role → insert sequence runs in one transaction (TOCTOU-safe: the inviter cannot be removed between check and insert) | `crates/buzz-db/src/channel.rs:362`, `:453-456` | `add_member` |
| 5.8 | Removing another member requires an elevated actor **or** being the target's `agent_owner_pubkey`; self-removal always allowed | `crates/buzz-db/src/channel.rs:470-489` | `remove_member` |
| 5.9 | The **last owner** of a channel cannot be removed, regardless of caller ("transfer ownership first") | `crates/buzz-db/src/channel.rs:491-510` | `remove_member` |
| 5.10 | Member removal is a **soft delete** (`removed_at`, `removed_by`); re-adding reverses it via `ON CONFLICT … DO UPDATE SET removed_at=NULL, removed_by=NULL, role=EXCLUDED.role` | `crates/buzz-db/src/channel.rs:512-527` (remove) and `:419-433` (re-add) | `remove_member` / `add_member` |
| 5.11 | Every membership read joins `channels … deleted_at IS NULL`, so a soft-deleted channel has no members | `crates/buzz-db/src/channel.rs:531-552`, `:581-608`, `:610-636`, `:1363-1385` | membership reads |
| 5.12 | "Accessible channels" = active memberships ∪ all `visibility='open'` channels | `crates/buzz-db/src/channel.rs:638-667`; DM rows additionally require `cm.hidden_at IS NULL` at `:822` | REQ filter resolution |
| 5.13 | `channels.community_id` is immutable — a BEFORE UPDATE trigger raises `check_violation` on any change | `migrations/0001_initial_schema.sql:115-128` | any `UPDATE channels` |

#### 6. Channel TTL / ephemeral channels

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| 6.1 | `ttl_deadline` is initialised to `NOW() + ttl_seconds` at create, and reset on any TTL change | `crates/buzz-db/src/channel.rs:117-119`, `:1099-1105` | `create_channel*`, `update_channel` |
| 6.2 | Every durable channel-scoped event **except kind 9007** refreshes `ttl_deadline` for ephemeral channels, inside the commit of that event | `migrations/0024_event_ttl_refresh_shared_lock.sql:25-57` (replacing `0022`) | deferred AFTER INSERT on `events` |
| 6.3 | TTL refresh is synchronized with TTL transitions via a per-channel advisory lock: **shared** on the event path, **exclusive** in `update_channel`. Permanent channels take no row lock at all | `migrations/0024_…:31-33` (shared); `crates/buzz-db/src/channel.rs:1131-1147` (exclusive) | event commit vs TTL change |
| 6.4 | A TTL-refresh failure must never reject an otherwise valid event — the trigger swallows exceptions into a `WARNING` | `migrations/0024_…:48-53` | trigger body |
| 6.5 | Expired ephemeral channels are archived idempotently (`archived_at IS NULL` guard), skipping archived communities; the reaper returns `(community_id, host, channel_id)` so side effects run under the right tenant | `crates/buzz-db/src/channel.rs:1387-1417` | `reap_expired_ephemeral_channels` |
| 6.6 | Unarchiving an expired ephemeral channel renews its lease so the reaper does not immediately re-archive it | `crates/buzz-db/src/channel.rs:1276-1288` | `unarchive_channel` |
| 6.7 | `archive_channel` rejects an already-archived channel and `unarchive_channel` rejects a non-archived one, both with `AccessDenied` | `crates/buzz-db/src/channel.rs:1219-1230`, `:1261-1271` | those calls |

#### 7. Thread counter materialization

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| 7.1 | `reply_count` counts **direct** children; `descendant_count` counts all descendants — the root is bumped even when root == parent | `crates/buzz-db/src/thread.rs:236-249` (increment), `:292-333` (decrement) | reply insert/delete |
| 7.2 | Counters are only bumped when the `thread_metadata` row was actually inserted (`rows_affected > 0` after `ON CONFLICT DO NOTHING`) | `crates/buzz-db/src/thread.rs:152-154`; `crates/buzz-db/src/event.rs:1113-1115` | reply insert |
| 7.3 | Missing parent/root rows are stubbed (`depth=0, broadcast=false`) so the counter `UPDATE` has a target | `crates/buzz-db/src/thread.rs:158-207`; `crates/buzz-db/src/event.rs:1116-1170` | reply insert |
| 7.4 | Insert + all counter updates share one transaction, so a crash cannot desynchronise counters from rows | `crates/buzz-db/src/thread.rs:130`, `:245` (commit); `crates/buzz-db/src/event.rs:1180-1199` | reply insert |
| 7.5 | Decrements floor at zero (`GREATEST(x - 1, 0)`) | `crates/buzz-db/src/thread.rs:296`, `:319`; `crates/buzz-db/src/event.rs:777`, `:789` | reply delete |
| 7.6 | Soft-deleting an event and decrementing counters is atomic | `crates/buzz-db/src/event.rs:756-803` | `soft_delete_event_and_update_thread` |

#### 8. Reactions

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| 8.1 | One reaction per `(community, target event, pubkey, emoji)`; the PK is the uniqueness | `migrations/0001_initial_schema.sql:548` | insert |
| 8.2 | `add_reaction` is a single upsert whose `DO UPDATE … WHERE reactions.removed_at IS NOT NULL` gives three outcomes: new (1 row), reactivation (1 row), active duplicate (0 rows) — no TOCTOU window | `crates/buzz-db/src/reaction.rs:66-112` | `add_reaction`, `add_reaction_tx` |
| 8.3 | `reaction_event_id` is only ever filled in, never cleared, on reactivation (`COALESCE(EXCLUDED…, reactions…)`) | `crates/buzz-db/src/reaction.rs:71` | reactivation |
| 8.4 | Reaction removal is a soft delete (`removed_at`), so history is retained | `crates/buzz-db/src/reaction.rs:140-172`, `:174-195` | `remove_reaction*` |
| 8.5 | A kind:7 reaction event is stored only after the reaction row is confirmed new/reactivated; an active duplicate short-circuits **before** the event insert and rolls back | `crates/buzz-db/src/event.rs:1230-1244` | `insert_reaction_event_with_thread_metadata` |
| 8.6 | A reaction whose target does not exist (or is soft-deleted) **in this community** commits nothing and returns `TargetMissing` | `crates/buzz-db/src/event.rs:1213-1228` | same |
| 8.7 | `get_reactions`' `limit` bounds **emoji groups**, not raw rows, so one busy emoji cannot hide other groups | `crates/buzz-db/src/reaction.rs:283-315` | `get_reactions` |

#### 9. Feeds

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| 9.1 | Hard cap `FEED_MAX_LIMIT = 100`: every feed query applies `limit.min(FEED_MAX_LIMIT)` before building SQL | `crates/buzz-db/src/feed.rs:29`, applied at `:96`, `:157`, `:209` | all three feed queries |
| 9.2 | An **empty** accessible-channel list means "community-global events only", never "all channels" | `crates/buzz-db/src/feed.rs:59-74`; unit tests `:766-786` | all three feed queries |
| 9.3 | Mentions are restricted to kinds `{9, 40002 (v2), forum post, forum comment}`; needs-action to `{workflow approval requested, stream reminder}`; activity to `{stream msg, v2, forum post, job request/progress/result}` — workflow execution kinds are deliberately excluded from activity | `crates/buzz-db/src/feed.rs:104-107`, `:165-167`, `:216-219`; assertions `:882-935` | feed queries |
| 9.4 | Feed reads never surface soft-deleted events (`e.deleted_at IS NULL`) | `crates/buzz-db/src/feed.rs:103`, `:164`, `:214` | feed queries |

#### 10. Query scoping and pagination

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| 10.1 | `before_id` requires `until`; `global_only` and `channel_id` are mutually exclusive — both are `InvalidData` errors | `crates/buzz-db/src/event.rs:304-317` | `query_events` |
| 10.2 | An explicitly empty `kinds`/`authors`/`ids`/`e_tags` list means "match nothing": early `Ok(vec![])` / `Ok(0)` | `crates/buzz-db/src/event.rs:320-332`, `:559-571` | `query_events`, `count_events` |
| 10.3 | `Some(empty)` `channel_ids` means "no channel access" → only `channel_id IS NULL` rows (explicitly annotated `SECURITY`) | `crates/buzz-db/src/event.rs:373-378`, `:601-604` | `query_events`, `count_events` |
| 10.4 | Default limit 100, clamped to `max_limit` (default 1000) | `crates/buzz-db/src/event.rs:334-336` | `query_events` |
| 10.5 | Channel-access filtering is pushed into SQL so `LIMIT` applies to the **visible** set (exact exhaustion semantics) | `crates/buzz-db/src/event.rs:371-390`; regression test `:1740-1789` | `query_events` |
| 10.6 | `has_more` for a channel window comes from an internal `LIMIT n+1` probe evaluated after all predicates; the sentinel row never leaves the module and callers must not re-derive exhaustion from row counts | `crates/buzz-db/src/thread.rs:556-563`, `:645-664` | `get_channel_window` |
| 10.7 | The channel-window cursor is captured from the last **raw** row before event reconstruction, so a row that fails to reconstruct cannot stall the cursor | `crates/buzz-db/src/thread.rs:650-663` | `get_channel_window` |
| 10.8 | Thread pagination uses a composite `(event_created_at, event_id)` keyset; the legacy 8-byte timestamp-only cursor is still accepted and documented as unsafe across same-second ties | `crates/buzz-db/src/thread.rs:337-344`, `:365-379`, `:400-419` | `get_thread_replies` |
| 10.9 | A channel window's "top-level" set is `depth IS NULL OR depth = 0 OR (depth = 1 AND broadcast)` | `crates/buzz-db/src/thread.rs:589-593` | `get_channel_window` |
| 10.10 | `get_events_by_ids` expects callers to bound batches (`debug_assert!(ids.len() <= 500)`) | `crates/buzz-db/src/event.rs:955` | `get_events_by_ids` |

#### 11. Read-replica routing (replica fence)

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| 11.1 | The fence starts **closed**; a closed or stale (> 30 s) fence routes every cursor page to the writer | `crates/buzz-db/src/replica_fence.rs:89-93`, `:114-127` | any routed read |
| 11.2 | Head fetches (`cursor: None`) always read the writer | `crates/buzz-db/src/lib.rs:2004-2043` (thread), `:2063-2077` (channel window) | routed reads |
| 11.3 | A channel-window cursor page may use the replica only when `fence.covers(cursor_ts)` | `crates/buzz-db/src/lib.rs:2070-2073` | `get_channel_window` |
| 11.4 | A thread cursor page from the replica is accepted only when it is **full** (≥ limit) **and** its tail `created_at` is ≤ fence; otherwise it is re-run on the writer (prevents a lag-truncated false EOF and mid-page holes) | `crates/buzz-db/src/lib.rs:2016-2032` | `get_thread_replies` |
| 11.5 | Fence value = `min(oldest_open_xact_start, sampled_at) − CREATED_AT_FLOOR_SECS(960) − FENCE_CLOCK_MARGIN_SECS(5)`, and only after the replica has replayed past the sampled writer LSN | `crates/buzz-db/src/replica_fence.rs:466-486` | `probe_once` |
| 11.6 | The handshake must sample in order S → activity scan → L, as three separately awaited statements on one pinned connection | `crates/buzz-db/src/replica_fence.rs:404-447` | `sample_writer` |
| 11.7 | Fails closed on: any probe error, masked `pg_stat_activity` rows, NULL/absent replica replay LSN, or a target that is not in recovery | `crates/buzz-db/src/replica_fence.rs:378-395`, `:449-463`, `:492-502` | probe loop |
| 11.8 | The probe is only spawned after **two** verifications pass: catalog shape of the floor-guard trigger on the parent and every partition, and observed behaviour through the armed pool | `crates/buzz-db/src/lib.rs:449-462`; `crates/buzz-db/src/replica_fence.rs:145-196`, `:199-330` | `spawn_fence_probe` |
| 11.9 | `run_migrations` itself fails closed if the floor-guard trigger is missing or mis-shaped anywhere | `crates/buzz-db/src/migration.rs:25` | `run_migrations` |

#### 12. Community / tenant lifecycle

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| 12.1 | Host → community resolution is case-insensitive and **excludes archived** communities; the caller turns `None` into a fail-closed connection error | `crates/buzz-db/src/lib.rs:656-683` | `lookup_community_by_host` |
| 12.2 | An operator-plane variant deliberately resolves archived communities too | `crates/buzz-db/src/lib.rs:696-715` | `lookup_community_by_host_for_management` |
| 12.3 | `MAX_COMMUNITIES_PER_OWNER = 3`, enforced inside the same transaction that holds a per-owner FNV-1a advisory lock, for both create and transfer | `crates/buzz-db/src/relay_members.rs:379`, `:384-391`; create `crates/buzz-db/src/lib.rs:869-916`; transfer `crates/buzz-db/src/relay_members.rs:418-467` | `create_community_with_owner`, `transfer_ownership` |
| 12.4 | Limit rejection **rolls back** the freshly inserted `communities` row | `crates/buzz-db/src/lib.rs:900-903`; test `:4767-4795` | `create_community_with_owner` |
| 12.5 | An identical create retry by the same owner returns the original row; a different owner on the same host gets `HostExists` | `crates/buzz-db/src/lib.rs:918-940` | `create_community_with_owner` |
| 12.6 | Archival is idempotent (`COALESCE(archived_at, now())`), owner-gated, and refuses the protected deployment host | `crates/buzz-db/src/lib.rs:947-978` | `archive_community_owned_by` |
| 12.7 | `bootstrap_owner` upserts the configured owner and demotes other owners to `'admin'`; `transfer_ownership` demotes them to `'member'` | `crates/buzz-db/src/relay_members.rs:320-360` vs `:469-489`; the difference is called out at `:406-408` | startup vs user transfer |
| 12.8 | Ownership transfer requires the caller's `expected_owner_pubkey` to match a current owner row locked `FOR UPDATE`; otherwise `OwnerConflict` and the caller must re-read | `crates/buzz-db/src/relay_members.rs:427-448` | `transfer_ownership` |
| 12.9 | `relay_members` removal never deletes an owner (`role <> 'owner'` in the DELETE), and the role-conditional variant distinguishes `RoleMismatch` from `IsOwner` | `crates/buzz-db/src/relay_members.rs:196-240`, `:242-285` | `remove_relay_member*` |
| 12.10 | Relay-member role updates cannot touch an owner (`role <> 'owner'`) | `crates/buzz-db/src/relay_members.rs:287-303` | `update_relay_member_role` |
| 12.11 | Invite-claimed membership persists the accepted policy version **in the same transaction** — membership cannot be granted without its acceptance record | `crates/buzz-db/src/relay_members.rs:122-158` | `claim_relay_membership` |
| 12.12 | Allowlist backfill runs only when the target community has **no** `relay_members` rows, so intentionally removed members are not resurrected | `crates/buzz-db/src/relay_members.rs:527-537` | `backfill_from_allowlist` |
| 12.13 | Archiving an identity is not a ban: it does not affect membership, relay access, or repo permissions; re-archiving is idempotent and does not mutate the existing row | `crates/buzz-db/src/archived_identities.rs:3-6`, `:44-79` | `archive` |

#### 13. API tokens

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| 13.1 | Only SHA-256 hashes are stored; the hash is 32 bytes (DB CHECK) | `migrations/0001_initial_schema.sql:488`; callers pass the hash (`crates/buzz-db/src/api_token.rs:8-9`) | insert |
| 13.2 | 10 active tokens per `(community, owner)`, enforced by a subquery inside the INSERT (`… WHERE (SELECT COUNT(*) …) < 10`) — no count-then-insert race | `crates/buzz-db/src/api_token.rs:91-126` | `create_api_token_if_under_limit` |
| 13.3 | "Active" for the quota means `revoked_at IS NULL AND (expires_at IS NULL OR expires_at > NOW())` | `crates/buzz-db/src/api_token.rs:113-118` | same |
| 13.4 | Lookup is keyed on `(community_id, token_hash)`, not hash alone — a token minted in A can never authorize in B | `crates/buzz-db/src/lib.rs:2337-2340`; `crates/buzz-db/src/api_token.rs:155-158` and rationale `:129-142` | auth path |
| 13.5 | Revocation is scoped to `(community, id, owner)` and skips already-revoked rows (idempotent) | `crates/buzz-db/src/api_token.rs:272-301`, `:303-329` | `revoke_token`, `revoke_all_tokens` |
| 13.6 | Self-minted tokens are flagged (`created_by_self_mint = TRUE`) | `crates/buzz-db/src/api_token.rs:100` | `create_api_token_if_under_limit` |

#### 14. Workflows, runs, approvals, cron claims

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| 14.1 | A NIP-33 workflow upsert may only update a row with the **same owner and same channel**; otherwise `AccessDenied` — a learned workflow UUID is not a cross-user overwrite primitive | `crates/buzz-db/src/workflow.rs:320-352` | `upsert_workflow` |
| 14.2 | Owner-scoped delete keeps the owner predicate inside the DELETE (`… AND owner_pubkey = $3`), so no check-then-delete race | `crates/buzz-db/src/workflow.rs:738-763` | `delete_workflow_for_owner` |
| 14.3 | List queries are bounded: `LIST_DEFAULT_LIMIT = 100`, clamped to `LIST_MAX_LIMIT = 1000`; run lists clamp to 1000 | `crates/buzz-db/src/workflow.rs:25-27`, `:395`, `:441`, `:466`, `:824` | list calls |
| 14.4 | The scheduler scan only returns `status='active' AND enabled AND definition->'trigger'->>'on' = 'schedule'` in non-archived communities, and carries each row's `community_id` | `crates/buzz-db/src/workflow.rs:457-478` | `list_all_enabled_workflows` |
| 14.5 | At-most-once cron fire: the first pod to insert `(community_id, workflow_id, scheduled_for)` wins; everyone else gets `None` and must not create a run. The claim binds **both** community and id because workflow UUIDs are not globally unique | `crates/buzz-db/src/workflow.rs:496-534`; PK `migrations/0001_initial_schema.sql:457` | `claim_scheduled_workflow_fire` |
| 14.6 | Interval anchoring reads `MAX(scheduled_for)` from the **claim** table, not `workflow_runs` | `crates/buzz-db/src/workflow.rs:528-560` | `latest_scheduled_workflow_fire` |
| 14.7 | A won claim whose run creation fails keeps `workflow_run_id` NULL on purpose — the instant stays claimed and must not fire twice | `crates/buzz-db/src/workflow.rs:556-561` | `attach_scheduled_workflow_run` |
| 14.8 | Claim retention prune must stay older than the largest supported interval or interval workflows lose their anchor (documented operational constraint) | `crates/buzz-db/src/workflow.rs:589-596` | `prune_scheduled_workflow_fires_before` |
| 14.9 | `create_approval` receives the **raw** token and hashes it with SHA-256 internally; the plaintext never reaches the database | `crates/buzz-db/src/workflow.rs:33-36`, `:915-946` | `create_approval` |
| 14.10 | Approval grant/deny includes `AND status = 'pending'` in the WHERE clause, so two concurrent decisions cannot both succeed (`false` ⇒ conflict / HTTP 409) | `crates/buzz-db/src/workflow.rs:1043-1051`, `:1074-1087` | `update_approval*` |
| 14.11 | Approval lookups bind `(community_id, token)` so an approval action for A/X can never act on B/X | `crates/buzz-db/src/workflow.rs:983-991` | `get_approval*`, `update_approval*` |
| 14.12 | `started_at` is stamped from the **bind parameter** (`$5 = 'running' AND started_at IS NULL`), fixing the earlier post-SET read; `completed_at` is stamped for `completed`/`failed`/`cancelled` | `crates/buzz-db/src/workflow.rs:843-880` | `update_workflow_run` |
| 14.13 | Status/enum strings are parsed back with strict `FromStr`; unknown values are `InvalidData`, never silently coerced | `crates/buzz-db/src/workflow.rs:61-71`, `:103-116`, `:148-160` | every row mapping |

#### 15. DMs

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| 15.1 | DM identity is `SHA-256(sorted, deduped participant pubkeys)` with no separator (fixed-width inputs) — order-independent | `crates/buzz-db/src/dm.rs:48-60`; UNIQUE index `migrations/0001_initial_schema.sql:104` | `create_dm`, `open_dm` |
| 15.2 | 2 ≤ participants ≤ 9, each exactly 32 bytes | `crates/buzz-db/src/dm.rs:107-124`, `:365-370` | `create_dm`, `open_dm` |
| 15.3 | DMs are `channel_type='dm', visibility='private'`, participants added as `'member'` | `crates/buzz-db/src/dm.rs:160-190` | `create_dm` |
| 15.4 | `create_dm` is idempotent: the participant-hash probe runs inside the transaction | `crates/buzz-db/src/dm.rs:126-155` | `create_dm` |
| 15.5 | `open_dm` always adds the caller to the participant set | `crates/buzz-db/src/dm.rs:361-364` | `open_dm` |
| 15.6 | Hiding a DM is per-user (`channel_members.hidden_at`), never a delete; re-opening the same participant set unhides it | `crates/buzz-db/src/dm.rs:377-381`, `:390-424`, `:429-446` | `hide_dm`, `open_dm` |
| 15.7 | Hidden DMs are filtered out of channel listings (`c.channel_type != 'dm' OR cm.hidden_at IS NULL`) and of `list_dms_for_user` | `crates/buzz-db/src/channel.rs:822`; `crates/buzz-db/src/dm.rs:257`, `:283` | listings |
| 15.8 | `list_dms_for_user` caps `limit` at 200 | `crates/buzz-db/src/dm.rs:233` | `list_dms_for_user` |

#### 16. Moderation

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| 16.1 | Report ingest is idempotent per `(community, report_event_id)` | `crates/buzz-db/src/moderation.rs:186-193`; UNIQUE `migrations/0006_moderation.sql:63` | `insert_report` |
| 16.2 | Exactly one target class per report row; `target_kind` is authoritative and the DB CHECK forbids mixed rows. An unrecognised `target_kind` on read is `InvalidData` | `migrations/0006_moderation.sql:42-46`; `crates/buzz-db/src/moderation.rs:580-590` | insert / read |
| 16.3 | Only an `open` report can be resolved/dismissed/escalated (`AND status='open'`) | `crates/buzz-db/src/moderation.rs:298-300` | `resolve_report` |
| 16.4 | Ban expiry is computed on read: `banned AND (ban_expires_at IS NULL OR ban_expires_at > now())`; NULL expiry with `banned` ⇒ permanent | `crates/buzz-db/src/moderation.rs:447-451`, `:476-479`, `:497-508` | enforcement reads |
| 16.5 | `muted_until` is only surfaced when in the future | `crates/buzz-db/src/moderation.rs:450` | `restriction_state` |
| 16.6 | `unban_member` only affects a currently banned row; `untimeout_member` only a currently active timeout | `crates/buzz-db/src/moderation.rs:359`, `:415` | those calls |
| 16.7 | A report may only be resolved by an action row **in its own community** (composite FK) | `migrations/0006_moderation.sql:128-130` | `resolve_report` |
| 16.8 | The 12-value action vocabulary is duplicated in Rust and asserted against the SQL CHECK by a test | `crates/buzz-db/src/moderation.rs:104-118`; assertion `crates/buzz-db/src/migration.rs:640-645` | build/test |
| 16.9 | The deployment-admin plane is the **only** moderation repository allowed to omit a `CommunityId`, and it clamps pages to 200 | `crates/buzz-db/src/admin_moderation.rs:1-8`, `:15-18` | admin reads |

#### 17. Push (NIP-PL)

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| 17.1 | Lease acceptance takes three locks in a fixed global order: address lock → author lock → per-community push-gate lock (activations only) | `crates/buzz-db/src/push.rs:216-243` | `accept_lease_event` |
| 17.2 | A signed lease event wins only on `source_created_at` DESC with lowest `source_event_id` as tiebreak, **and** strictly increasing `generation`; otherwise `StaleEvent` / `StaleGeneration` | `crates/buzz-db/src/push.rs:266-280`; upsert predicate `:475-483` | `accept_lease_event`, `replace_lease` |
| 17.3 | Reusing a `source_event_id` under a different `(author, installation)` is `SourceEventCollision` | `crates/buzz-db/src/push.rs:245-259`, `:398-402` | `accept_lease_event` |
| 17.4 | Expired active leases are deactivated before quota/endpoint checks so they cannot hold quota or endpoint uniqueness forever | `crates/buzz-db/src/push.rs:285-296` | `accept_lease_event` |
| 17.5 | Per-author active-lease quota (`max_active_leases`, caller-supplied) and per-`(author, app_profile, endpoint_hash)` uniqueness are checked in-transaction, backed by a partial unique index | `crates/buzz-db/src/push.rs:298-323`; index `migrations/0012_push_leases.sql:23-25` |`accept_lease_event` |
| 17.6 | Any `23xxx` integrity violation on a validated lease is mapped to a protocol outcome, not a 500 | `crates/buzz-db/src/push.rs:392-410` | `accept_lease_event` |
| 17.7 | `push_match_queue` enqueue happens in the event's own transaction, only for kinds `{7, 9, 1059, 40007, 46010}`, and **only** when the community has an active + endpoint-enabled + unexpired lease | `migrations/0023_push_match_gate.sql:22-42` | AFTER INSERT on `events` |
| 17.8 | The lost-wake race is closed by a per-community advisory lock: event inserts take `'buzz_push_gate:<community>'` **shared**, lease activations take it **exclusive** | `migrations/0023_…:34-35`; `crates/buzz-db/src/push.rs:15-33`, `:239-243`, `:464-468` | insert vs activation |
| 17.9 | On activation, recent gate-skipped events (`received_at > now() - 120s`, relay clock, same kind allowlist) are backfilled into the queue **inside** the activation transaction | `crates/buzz-db/src/push.rs:35-66`, `:373-375`, `:496-499` | activation |
| 17.10 | Wake enqueue copies `endpoint_hash` and `endpoint_grant` from the current lease — a caller cannot redirect a wake | `crates/buzz-db/src/push.rs:502-511`, `:650-701` | `enqueue_wakes` |
| 17.11 | Wake dedup key is `(community_id, endpoint_hash, event_id)`; the first request to claim an inserted key reports `Enqueued`, later same-key requests report `Duplicate` | `migrations/0012_push_leases.sql:47`; `crates/buzz-db/src/push.rs:703-763` | `enqueue_wakes` |
| 17.12 | Lease rows are locked `FOR UPDATE` in `(author, installation_id)` order so batches and single-row replacements cannot deadlock | `crates/buzz-db/src/push.rs:624-660` | `enqueue_wakes` |
| 17.13 | An enqueue whose generation no longer matches the current active lease returns `InactiveLease` | `crates/buzz-db/src/push.rs:667-675` | `enqueue_wakes` |
| 17.14 | Matcher claims are one-community-at-a-time, `SKIP LOCKED`, `attempts < MAX_MATCH_ATTEMPTS(8)`, and increment `attempts` | `crates/buzz-db/src/push.rs:70`, `:840-882` | `claim_due_match_batch` |
| 17.15 | A claimed job whose source event is absent or soft-deleted is deleted — a deliberate privacy-preserving terminal outcome | `crates/buzz-db/src/push.rs:899-916` | `claim_due_match_batch` |
| 17.16 | Exhausted jobs are reaped by a **separate** periodic sweep, never on the claim path | `crates/buzz-db/src/push.rs:925-941` | `reap_exhausted_matches` |
| 17.17 | Every completion/retry/fail requires the fencing `claim_id` and `state='sending'`/`'matching'`; stale workers are no-ops | `crates/buzz-db/src/push.rs:970-1017`, `:1132-1195` | those calls |
| 17.18 | The load-bearing authorization gate is `revalidate_wake_for_send`, re-joining community + generation + endpoint_hash + live event immediately before the transport call; claim-time checks are only optimizations | `crates/buzz-db/src/push.rs:1080-1130` | send path |
| 17.19 | Endpoint disable only applies to the exact current generation (`AND generation=$4 AND active AND endpoint_enabled`), making stale gateway responses no-ops | `crates/buzz-db/src/push.rs:1193-1221` | `disable_endpoint_generation` |
| 17.20 | Outbox pruning skips rows whose event still has a matcher job | `crates/buzz-db/src/push.rs:1223-1243` | `prune_wake_outbox` |
| 17.21 | Structural coupling: an `active` lease must have all five effective columns non-NULL; an inactive lease must have them all NULL | `migrations/0012_push_leases.sql:20-21` | any write |

#### 18. Reminders (NIP-ER, kind 30300)

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| 18.1 | Due set = latest head per `(community, pubkey, d_tag)` with `not_before <= now`, not deleted, not delivered, in a non-archived community | `crates/buzz-db/src/event.rs:1293-1339` | `query_due_reminders` |
| 18.2 | Exactly one pod wins a reminder: `UPDATE … SET delivered_at=$1 WHERE … delivered_at IS NULL` | `crates/buzz-db/src/event.rs:1370-1400`; race test `:2065-2107` | `claim_due_reminder*` |
| 18.3 | Release is compare-and-clear on the pod's own stamp (`AND delivered_at = $4`), so one pod cannot roll back another's later claim | `crates/buzz-db/src/event.rs:1402-1431` | `release_due_reminder` |
| 18.4 | Claim and release are community-scoped, because the same event id may exist in two communities | `crates/buzz-db/src/event.rs:1359-1368`, `:1394-1400`; test `:2183-2263` | claim/release |

#### 19. Git repo names

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| 19.1 | Names are unique **within a community**, enforced atomically by `INSERT … ON CONFLICT (community_id, repo_id) DO NOTHING RETURNING owner_pubkey` (replaces the old filesystem `create_dir` race guard) | `crates/buzz-db/src/git_repo.rs:96-113`; PK `migrations/0002_git_repo_names.sql:25` | `reserve_repo_name` |
| 19.2 | Same-owner re-announce is `AlreadyOwned` (no quota re-check); different owner is `TakenByOther` | `crates/buzz-db/src/git_repo.rs:115-140` | `reserve_repo_name` |
| 19.3 | A conflicting row that vanished between INSERT and the classification read is treated as `TakenByOther` — a name is never granted without an atomic claim | `crates/buzz-db/src/git_repo.rs:134-139` | race |
| 19.4 | Per-pubkey quota is a caller responsibility using `count_repos_for_owner`; this module returns no quota error | `crates/buzz-db/src/git_repo.rs:88-92`, `:136-158` | announce handler |
| 19.5 | Rollback release is owner-scoped, so it can never delete another owner's concurrent reservation | `crates/buzz-db/src/git_repo.rs:160-180` | seed failure |

#### 20. Users / agents

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| 20.1 | Agent ownership is first-mint-wins, set by a conditional `UPDATE … WHERE agent_owner_pubkey IS NULL`; `false` means an owner already exists, an error means the agent row is missing | `crates/buzz-db/src/user.rs:283-325` | `set_agent_owner` |
| 20.2 | `agent_owner_pubkey` must reference a user in the **same** community (composite self-FK, `ON DELETE SET NULL`) | `migrations/0001_initial_schema.sql:173-174` | insert/update |
| 20.3 | Because `agent_owner_pubkey` is immutable after mint, `remove_member` may read it outside its transaction | documented at `crates/buzz-db/src/channel.rs:451-456`, `:483-485` | `remove_member` |
| 20.4 | `channel_add_policy` values are validated in Rust before the enum cast | `crates/buzz-db/src/user.rs:374-380` | `set_channel_add_policy` |
| 20.5 | Empty profile strings are stored as NULL — required for kind:0 absolute-state semantics and to avoid `nip05_handle` uniqueness collisions on `''` | `crates/buzz-db/src/user.rs:95-101`, `:135-140` | `update_user_profile` |
| 20.6 | User search escapes LIKE metacharacters and clamps `limit` to `[1, 500]` | `crates/buzz-db/src/user.rs:207-222`, `:233` | `search_users` |

#### 21. Retention / TTL / pruning summary

| Object | Policy | Where |
|--------|--------|-------|
| Superseded NIP-RS payloads | hard delete + watermark retained | `crates/buzz-db/src/lib.rs:3739-3796`, `migrations/0007`, `0009`, `0011` |
| Superseded mesh-status payloads | hard delete | `crates/buzz-db/src/lib.rs:3688-3695`, `migrations/0019` |
| Other superseded replaceables | soft delete (`deleted_at`) — history retained | `crates/buzz-db/src/lib.rs:3752-3757` |
| Events, channels, channel members, reactions | soft delete only; no purge path in this crate | `deleted_at` / `removed_at` / `removed_at` columns |
| Ephemeral channels | archived when `ttl_deadline < NOW()` | `crates/buzz-db/src/channel.rs:1387-1417` |
| `scheduled_workflow_fires` | operator-driven `DELETE … claimed_at < cutoff` | `crates/buzz-db/src/workflow.rs:597-611` |
| `push_wake_outbox` | `DELETE` terminal/expired rows older than a cutoff, unless a matcher job still references the event | `crates/buzz-db/src/push.rs:1223-1243` |
| `push_match_queue` | `DELETE` when `attempts >= 8`, or on load-miss | `crates/buzz-db/src/push.rs:899-941` |
| `events` / `delivery_log` partitions | created ahead; **no drop/detach path exists in this crate** | `crates/buzz-db/src/partition.rs:15-73` |
| `audit_log`, `delivery_log`, `subscriptions`, `rate_limit_violations` | no retention code in buzz-db | — |

#### 22. Partition management rules

| # | Rule | Enforced at |
|---|------|-------------|
| 22.1 | Only `events` and `delivery_log` may be partition-managed (`PARTITIONED_TABLES` allowlist), re-checked inside `ensure_partition` | `crates/buzz-db/src/partition.rs:12`, `:84-88` |
| 22.2 | Partition suffix must be digits/underscores only | `crates/buzz-db/src/partition.rs:58-60`, `:89-93` |
| 22.3 | Boundary dates must be exactly `YYYY-MM-DD` (length 10, digits, dashes at 4 and 7) | `crates/buzz-db/src/partition.rs:63-73`, `:94-104` |
| 22.4 | Existence is checked against `pg_catalog` with `relispartition = true` and `current_schema()` before any DDL | `crates/buzz-db/src/partition.rs:108-127` |
| 22.5 | A `42P17` "would overlap partition" error is treated as "already ensured" (the right-edge `*_p_future` catch-all covers the month) rather than failing startup | `crates/buzz-db/src/partition.rs:136-148` |
| 22.6 | Month arithmetic rolls the year over and rejects impossible dates with `InvalidData` | `crates/buzz-db/src/partition.rs:18-49` |

#### 23. Tenant scoping (community_id) as a rule

| # | Rule | Enforced at |
|---|------|-------------|
| 23.1 | Every tenant-scoped table carries `community_id NOT NULL`; asserted over the concatenated SQL of **all** migrations | `crates/buzz-db/src/migration.rs:635-651` (`all_non_operator_global_tables_have_not_null_community_id`) |
| 23.2 | Every PK / UNIQUE / FK / unique index on a tenant-scoped table must lead with `community_id`; the sole exception is `delivery_log`'s partition PK `(delivered_at, id)` | `crates/buzz-db/src/migration.rs:495-512`, `:653-670` |
| 23.3 | "Tenant-scoped" is defined negatively: any table not listed in `_operator_global_tables` | `crates/buzz-db/src/migration.rs:330-360` |
| 23.4 | Migrations may not re-tenant `channels`: no `UPDATE … SET community_id`, no drop/alter/rename of the column, no drop/disable of the guard trigger, no `DROP TABLE channels` | `crates/buzz-db/src/migration.rs:527-556`, `:672-688` |
| 23.5 | Cross-community reads are allowed only in the explicitly documented operator/scheduler paths (see `buzz-db-security.md` §tenant isolation) | `crates/buzz-db/src/admin_moderation.rs:1-8`, `crates/buzz-db/src/usage.rs:1-14`, `crates/buzz-db/src/workflow.rs:449-456` |
