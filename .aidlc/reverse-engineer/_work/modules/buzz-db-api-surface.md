## Module: buzz-db (`crates/buzz-db`)

### API Surface

Shape of the crate's public API:

| Kind | Count | Where |
|------|-------|-------|
| `Db` inherent methods | 215 | `crates/buzz-db/src/lib.rs:360`–`:3628` |
| Free functions in `lib.rs` | 1 (`insert_mentions`) | `crates/buzz-db/src/lib.rs:97` |
| Module-level free functions | 187 | the 19 sibling modules |
| `ReplicaFence` methods | 5 | `crates/buzz-db/src/replica_fence.rs:88`–`:140` |
| `UsageMetricsLeader::is_live` | 1 | `crates/buzz-db/src/lib.rs:215` |
| Public constants | 12 | see table at end |

Two-layer design: nearly every `Db` method is a thin delegate that passes
`&self.pool` (or `self.read()`) plus a server-resolved `CommunityId` to a
module-level free function. Exceptions — statements written inline on `Db` —
are called out below.

Public module list and doc intent: `crates/buzz-db/src/lib.rs:12-51`.
Re-exports: `pub use error::{DbError, Result}` and
`pub use event::{EventQuery, ReactionEventInsertOutcome}` — `crates/buzz-db/src/lib.rs:53-54`.

---

#### `lib.rs` — `Db` handle, community registry, replaceable-event writes

| Fn | Operation | Tables touched |
|----|-----------|----------------|
| `insert_mentions` `:97` | multi-row `QueryBuilder` `INSERT … ON CONFLICT DO NOTHING`; extracts + validates lowercase-hex 64-char `p` tags | `event_mentions` |
| `Db::new` `:360` / `connect_pool` `:387` | `PgPoolOptions` connect; writer pool sets `SELECT set_config('buzz.created_at_floor', $1, false)` on every connection | — |
| `Db::from_pool` `:403`, `from_pools` `:419` | construct from existing pools (tests) | — |
| `Db::fence` `:429`, `read` `:470`, `has_read_pool` `:475`, `pool_stats` `:494`, `read_pool_stats` `:503` | accessors | — |
| `Db::spawn_fence_probe` `:449` | runs `verify_floor_guard_catalog` + `verify_floor_guard_behavior`, then spawns `replica_fence::run_probe` | `pg_trigger`/`pg_class` catalog; scratch rows in `communities`+`events` (rolled back) |
| `Db::migrate` `:480` | `migration::run_migrations` | `_sqlx_migrations` + all |
| `Db::ping` `:485` | `SELECT 1` | — |
| `Db::try_lock_usage_metrics` `:517` | `SELECT pg_try_advisory_lock($1)` on a detached connection | — (advisory lock) |
| `Db::admin_list_reports` `:537`, `admin_get_report` `:563`, `admin_list_feedback` `:571`, `admin_get_feedback` `:579` | delegate to `admin_moderation` (deployment-global) | `moderation_reports`, `product_feedback`, `communities` |
| `Db::usage_*` (11 methods) `:587`–`:640` | delegate to `usage` | `communities`, `users`, `channels`, `events`, `relay_members`, `workflows`, `git_repo_names` |
| `Db::begin_transaction` `:648` | `pool.begin()` → `Transaction<'static, Postgres>` | — |
| `Db::lookup_community_by_host` `:656` | `SELECT id, host FROM communities WHERE lower(host)=lower($1) AND archived_at IS NULL` | `communities` |
| `Db::is_community_active` `:685` | `SELECT EXISTS(… archived_at IS NULL)` | `communities` |
| `Db::lookup_community_by_host_for_management` `:696` | same lookup **without** the archived filter | `communities` |
| `Db::list_communities_owned_by` `:717` | `communities JOIN relay_members … WHERE rm.pubkey=$1 AND rm.role='owner'` (no community filter — operator plane) | `communities`, `relay_members` |
| `Db::lookup_community_host` `:762` | `SELECT host … WHERE id=$1 AND archived_at IS NULL` | `communities` |
| `Db::get_community_icon` `:786` / `set_community_icon` `:806` | `SELECT icon` / `UPDATE … SET icon` | `communities` |
| `Db::ensure_configured_community` `:830` | `INSERT … ON CONFLICT (lower(host)) DO UPDATE SET host = communities.host RETURNING id, host, (xmax = 0) AS created` | `communities` |
| `Db::create_community_with_owner` `:862` | tx: `pg_advisory_xact_lock(owner_count_advisory_lock_key)` → `INSERT communities … ON CONFLICT DO NOTHING` → owner count check → `INSERT relay_members … 'owner'` | `communities`, `relay_members` |
| `Db::archive_community_owned_by` `:947` | `UPDATE communities … FROM relay_members … SET archived_at = COALESCE(archived_at, now())`, refuses the protected deployment host | `communities`, `relay_members` |
| `Db::unarchive_community_owned_by` `:980` | `UPDATE communities SET archived_at = NULL FROM relay_members …` | `communities`, `relay_members` |
| `Db::community_of_channel` `:1012` | `SELECT community_id FROM channels WHERE id=$1 AND deleted_at IS NULL` (resolves tenancy — deliberately unscoped) | `channels` |
| `Db::communities_of_channels` `:1050` | `SELECT id, community_id … WHERE id = ANY($1) AND deleted_at IS NULL`; missing ids are absent from the map | `channels` |
| `Db::insert_event` `:1079` | `event::insert_event`, then best-effort `insert_mentions` when inserted | `events`, `event_mentions` |
| `Db::query_events` `:1095`, `count_events` `:1100` | delegate to `event` | `events` (+`event_mentions` on `#p`) |
| `Db::huddle_started_link_exists` `:1106` | delegate | `events` |
| `Db::get_latest_global_replaceable` `:1128`, `get_event_by_id` `:1140`, `get_event_by_id_including_deleted` `:1149`, `get_events_by_ids` `:1215` | delegate | `events` |
| `Db::soft_delete_event` `:1158`, `soft_delete_by_coordinate` `:1168`, `soft_delete_event_and_update_thread` `:1179` | delegate | `events`, `thread_metadata` |
| `Db::get_last_message_at` `:1197`, `_bulk` `:1206` | delegate | `events` |
| `Db::claim_due_push_match_batch` `:1224` … `disable_push_endpoint` `:1338`, `accept_push_lease_event` `:1357` (14 methods) | delegate to `push` | `push_match_queue`, `push_leases`, `push_wake_outbox`, `events` |
| `Db::insert_event_with_thread_metadata` `:1379` | delegate + best-effort `insert_mentions` | `events`, `thread_metadata`, `event_mentions` |
| `Db::insert_reaction_event_with_thread_metadata` `:1404` | delegate + best-effort `insert_mentions` | `events`, `reactions`, `thread_metadata`, `event_mentions` |
| `Db::create_channel` `:1438` … `reap_expired_ephemeral_channels` `:1725` (26 methods) | delegate to `channel` | `channels`, `channel_members`, `users`, `communities` |
| `Db::query_due_reminders` `:1732`, `claim_due_reminder` `:1741`, `claim_due_reminder_with_stamp` `:1751`, `release_due_reminder` `:1769` | delegate to `event` | `events`, `communities` |
| `Db::ensure_user` `:1791` … `set_channel_add_policy` `:1877` (9 methods) | delegate to `user` | `users` |
| `Db::find_dm_by_participants` `:1887` … `list_hidden_dms` `:1950` (7 methods) | delegate to `dm` | `channels`, `channel_members`, `users` |
| `Db::insert_thread_metadata` `:1960`, `get_thread_summary` `:2044`, `get_thread_metadata_by_event` `:2079`, `decrement_reply_count` `:2088` | delegate to `thread` | `thread_metadata`, `events` |
| `Db::get_thread_replies` `:2004` | **replica routing**: cursor page tried on `read()` when fence open; re-run on writer unless page is full AND tail ≤ fence | `thread_metadata`, `events` |
| `Db::get_channel_window` `:2063` | **replica routing**: `read()` only when `fence.covers(cursor_ts)` | `events`, `thread_metadata` |
| `Db::add_reaction` `:2099` … `get_reactions_bulk` `:2212` (7 methods) | delegate to `reaction` | `reactions` |
| `Db::query_feed_mentions` `:2221`, `query_feed_needs_action` `:2241`, `query_feed_activity` `:2261` | delegate to `feed` | `events`, `event_mentions` |
| `Db::create_api_token` `:2273`, `create_api_token_if_under_limit` `:2298`, `get_api_token_by_hash_including_revoked` `:2352`, `list_tokens_by_owner` `:2421`, `revoke_token` `:2430`, `revoke_all_tokens` `:2448` | delegate to `api_token` | `api_tokens` |
| `Db::get_api_token_by_hash` `:2327` | **inline** `SELECT … WHERE community_id=$1 AND token_hash=$2 AND revoked_at IS NULL` | `api_tokens` |
| `Db::touch_api_token` `:2366` / `update_token_last_used` `:2378` (alias) | **inline** `UPDATE api_tokens SET last_used_at = NOW() WHERE community_id=$1 AND token_hash=$2` | `api_tokens` |
| `Db::list_active_tokens` `:2387` | **inline** `SELECT … WHERE community_id=$1 AND revoked_at IS NULL ORDER BY created_at DESC LIMIT 1000` | `api_tokens` |
| `Db::create_workflow` `:2464` … `update_approval_by_stored_hash` `:2782` (27 methods) | delegate to `workflow` | `workflows`, `workflow_runs`, `workflow_approvals`, `scheduled_workflow_fires` |
| `Db::ensure_future_partitions` `:2802` | delegate to `partition` | DDL on `events`, `delivery_log` |
| `Db::backfill_d_tags` `:2810` | **inline** `UPDATE events SET d_tag = COALESCE((SELECT elem->>1 FROM jsonb_array_elements(tags) …), '') WHERE kind BETWEEN 30000 AND 39999 AND d_tag IS NULL` — **no community filter** | `events` |
| `Db::is_pubkey_allowed` `:2826`, `has_allowlist_entries` `:2839`, `add_to_allowlist` `:2850`, `remove_from_allowlist` `:2871`, `list_allowlist` `:2886` | **inline** count/insert/delete/select, all `WHERE community_id = $1` | `pubkey_allowlist` |
| `Db::is_relay_member` `:2907` … `backfill_from_allowlist` `:3027` (12 methods) | delegate to `relay_members` | `relay_members`, `join_policy_acceptances`, `pubkey_allowlist` |
| `Db::insert_product_feedback` `:3032`, `list_product_feedback` `:3041` | delegate to `product_feedback` | `product_feedback` |
| `Db::insert_moderation_report` `:3049` … `list_moderation_actions` `:3185` (15 methods) | delegate to `moderation` | `moderation_reports`, `community_bans`, `moderation_actions` |
| `Db::repo_name_owner` `:3195`, `reserve_repo_name` `:3207`, `count_repos_for_owner` `:3217`, `release_repo_name` `:3228` | delegate to `git_repo` | `git_repo_names` |
| `Db::is_archived` `:3238`, `archive` `:3244`, `unarchive` `:3268`, `list_archived` `:3273` | delegate to `archived_identities` | `archived_identities` |
| `Db::soft_delete_discovery_events` `:3281` | **inline** `UPDATE events SET deleted_at=NOW() WHERE community_id=$1 AND channel_id=$2 AND pubkey=$3 AND deleted_at IS NULL AND kind IN (39000,39001,39002)` | `events` |
| `Db::replace_addressable_event` `:3306` | **inline tx**: `pg_advisory_xact_lock(fnv1a(community,kind,pubkey,channel))` → newest-live probe (`ORDER BY created_at DESC, id ASC LIMIT 1`) → stale reject → soft-delete old → `INSERT … ON CONFLICT DO NOTHING` → commit → best-effort mentions | `events`, `event_mentions` |
| `Db::nip43_membership_snapshot_needs_reconciliation` `:3438` | `query_events` for the relay-signed kind:13534 snapshot + `list_relay_members`, compares sorted `(pubkey, role)` sets | `events`, `relay_members` |
| `Db::publish_nip43_membership_locked` `:3488` | **inline tx**: advisory lock → `SELECT pubkey, role FROM relay_members WHERE community_id=$1 ORDER BY created_at ASC` → build+sign kind:13534 → soft-delete prior snapshots → insert | `relay_members`, `events`, `event_mentions` |
| `Db::replace_parameterized_event` `:3628` | **inline tx**: advisory lock → classify NIP-RS / buzz-mesh-status → probe live head + `parameterized_event_watermarks` → stale reject → hard-DELETE (with `set_config('buzz.nip_rs_hard_delete','on',true)` for NIP-RS) or soft-delete → insert → upsert watermark | `events`, `parameterized_event_watermarks`, `event_mentions` |

Private helper: `event_replacement_lock_key` `crates/buzz-db/src/lib.rs:63` —
FNV-1a over `(community_id, kind, pubkey, optional coordinate)` → `i64`
advisory-lock key.

---

#### `event.rs`

| Fn | Operation | Tables |
|----|-----------|--------|
| `extract_d_tag` `:141` | pure; NIP-33 kinds only, missing `d` ⇒ `Some("")` | — |
| `extract_not_before` `:166` | pure; kind 30300 only | — |
| `huddle_started_link_exists` `:199` | `SELECT content … WHERE community_id/channel_id/kind=48100/pubkey AND octet_length(content)<=512 AND content ILIKE $6 ORDER BY created_at DESC, id ASC LIMIT 32`, then JSON check in Rust | `events` |
| `insert_event` `:240` | rejects `KIND_AUTH`/ephemeral, then single `INSERT … ON CONFLICT DO NOTHING` (12 columns) | `events` |
| `query_events` `:302` | `QueryBuilder` SELECT with 14 optional predicates; `INNER JOIN event_mentions` when `p_tag_hex` set; `ORDER BY created_at DESC, id ASC LIMIT/OFFSET` | `events`, `event_mentions` |
| `count_events` `:557` | same predicate set, `SELECT COUNT(*)` (no cursor/order/limit) | `events`, `event_mentions` |
| `soft_delete_event` `:703` | `UPDATE … SET deleted_at=NOW() WHERE community_id AND id AND deleted_at IS NULL` | `events` |
| `soft_delete_by_coordinate` `:730` | same on `(kind, pubkey, d_tag)` | `events` |
| `soft_delete_event_and_update_thread` `:756` | tx: soft-delete + `reply_count = GREATEST(reply_count-1,0)` + `descendant_count = GREATEST(…-1,0)` | `events`, `thread_metadata` |
| `get_last_message_at` `:806` / `_bulk` `:831` | `MAX(created_at)` / `GROUP BY channel_id` | `events` |
| `get_event_by_id` `:868`, `get_event_by_id_including_deleted` `:924` | scoped id lookup, `ORDER BY created_at DESC LIMIT 1` | `events` |
| `get_latest_global_replaceable` `:894` | `… channel_id IS NULL AND deleted_at IS NULL ORDER BY created_at DESC, id ASC LIMIT 1` | `events` |
| `get_events_by_ids` `:948` | `QueryBuilder` `id IN (…)`; `debug_assert!(ids.len() <= 500)` | `events` |
| `insert_event_with_thread_metadata` `:1180` (+ private `_tx` `:1017`) | tx: event insert → `thread_metadata` insert → parent/root stub inserts → `reply_count+1`, `last_reply_at=NOW()`, `descendant_count+1` | `events`, `thread_metadata` |
| `insert_reaction_event_with_thread_metadata` `:1201` | tx: resolve live target → `reaction::add_reaction_tx` → short-circuit on active duplicate → event+thread insert | `events`, `reactions`, `thread_metadata` |
| `query_due_reminders` `:1293` | `SELECT DISTINCT ON (community_id, pubkey, d_tag) … JOIN communities … WHERE kind=30300 AND not_before<=$2 AND deleted_at IS NULL AND delivered_at IS NULL AND c.archived_at IS NULL ORDER BY …, created_at DESC, id ASC LIMIT $3` | `events`, `communities` |
| `claim_due_reminder` `:1344` / `_with_stamp` `:1370` | `UPDATE events SET delivered_at=$1 WHERE community_id AND created_at AND id AND delivered_at IS NULL` | `events` |
| `release_due_reminder` `:1402` | compare-and-clear `… AND delivered_at = $4` | `events` |
| `row_to_stored_event` (crate-private) `:451` | rebuilds `nostr::Event` from JSON; unparseable rows are skipped with a warn | — |

---

#### `channel.rs`

| Fn | Operation | Tables |
|----|-----------|--------|
| `create_channel` `:87` | validates 32-byte pubkey + canonical non-empty name; tx: insert channel (`ttl_deadline = NOW() + ($8 || ' seconds')::interval`) → upsert creator as `'owner'` → read back | `channels`, `channel_members` |
| `create_channel_with_id` `:175` | same, client UUID, rejects nil UUID, `ON CONFLICT (community_id, id) DO NOTHING`; returns `(record, was_created)` | `channels`, `channel_members` |
| `get_channel` `:271` | scoped select, `deleted_at IS NULL`, else `ChannelNotFound` | `channels` |
| `get_canvas` `:298` / `set_canvas` `:315` | select / update `canvas` | `channels` |
| `add_member` `:346` | tx: read channel → role enforcement (private needs elevated inviter unless creator bootstrap; elevated roles need owner/admin granter) → `INSERT … ON CONFLICT DO UPDATE SET removed_at=NULL, removed_by=NULL, role=EXCLUDED.role` → read back | `channels`, `channel_members` |
| `remove_member` `:459` | tx: actor role check (or self-remove, or `user::is_agent_owner` on the **pool**) → last-owner guard (`COUNT(*) … role='owner' AND removed_at IS NULL <= 1`) → `UPDATE SET removed_at=NOW(), removed_by=$1` | `channel_members`, `channels`, `users` |
| `is_member` `:531` | `COUNT(*)` with `JOIN channels … deleted_at IS NULL` | `channel_members`, `channels` |
| `membership_pairs` `:554` | one statement, `channel_id = ANY($2) AND pubkey = ANY($3)` | `channel_members`, `channels` |
| `get_members` `:581` (LIMIT 1000) / `get_members_bulk` `:610` (`ANY($2)`) | active members joined to live channels | `channel_members`, `channels` |
| `get_accessible_channel_ids` `:638` | membership `UNION` all `visibility='open'` channels (no LIMIT) | `channel_members`, `channels` |
| `list_channels` `:669` | two static variants (with/without `visibility::text = $2`), `LIMIT 1000` | `channels` |
| `get_accessible_channels` `:828` | `format!`-built SQL (`AssertSqlSafe`) with a fixed membership clause + optional `$3` visibility bind; `LEFT JOIN channel_members`; DM rows hidden when `cm.hidden_at IS NOT NULL`; `LIMIT 1000` | `channels`, `channel_members` |
| `get_bot_members` `:894` | `json_agg(DISTINCT jsonb_build_object('name', c.name, 'id', c.id::text))` grouped per bot; `LIMIT 1000` | `channel_members`, `users`, `channels` |
| `get_users_bulk` `:938` | `format!`-built `$n` placeholder list (`AssertSqlSafe`), all values bound | `users` |
| `update_channel` `:1050` | dynamic SET list (`format!` + `AssertSqlSafe`, positional binds); TTL changes run in a tx that first takes `pg_advisory_xact_lock(hashtextextended('buzz_channel_ttl:<community>:<channel>'))` | `channels` |
| `set_topic` `:1155` / `set_purpose` `:1179` | update + `*_set_by`/`*_set_at` | `channels` |
| `archive_channel` `:1206` | read state → `AccessDenied` if already archived → `UPDATE SET archived_at=NOW()` | `channels` |
| `unarchive_channel` `:1248` | read state → `AccessDenied` if not archived → clear `archived_at` and renew `ttl_deadline` when `ttl_seconds IS NOT NULL` | `channels` |
| `soft_delete_channel` `:1292` | `UPDATE SET deleted_at=NOW() … deleted_at IS NULL` | `channels` |
| `get_member_count` `:1309` / `get_member_counts_bulk` `:1328` | `COUNT(*)` / `QueryBuilder` `GROUP BY channel_id` | `channel_members` |
| `get_member_role` `:1363` | scoped role read joined to live channel | `channel_members`, `channels` |
| `reap_expired_ephemeral_channels` `:1387` | global `UPDATE channels … FROM communities WHERE ttl_deadline < NOW() AND archived_at IS NULL AND deleted_at IS NULL AND c.archived_at IS NULL RETURNING community_id, host, id` | `channels`, `communities` |

Private: `get_active_role_tx` `:697`, `get_channel_tx` `:718`,
`row_to_channel_record` `:983`, `row_to_member_record` `:1027`.

---

#### `thread.rs`

| Fn | Operation | Tables |
|----|-----------|--------|
| `insert_thread_metadata` `:116` | tx: insert row `ON CONFLICT DO NOTHING` → parent/root stub inserts → `reply_count+1`/`last_reply_at`/`descendant_count+1` (only when the row was actually inserted) | `thread_metadata` |
| `increment_reply_count` `:251` | `#[allow(dead_code)]`, documented as unused; standalone counter bump | `thread_metadata` |
| `decrement_reply_count` `:292` | `GREATEST(x-1, 0)` on parent and root | `thread_metadata` |
| `get_thread_replies` `:345` | `format!`-built SQL (`AssertSqlSafe`) joining `thread_metadata ⋈ events` on `(community_id, created_at, id)`; composite `(event_created_at, event_id) > ($ts,$id)` keyset, legacy 8-byte timestamp-only cursor supported; `ORDER BY event_created_at ASC, event_id ASC LIMIT $n` | `thread_metadata`, `events` |
| `get_thread_summary` `:489` | counters read + top-10 participants (`DISTINCT` pubkey, `MAX(created_at)` order) | `thread_metadata`, `events` |
| `get_channel_window` `:565` | `format!`-built SQL (`AssertSqlSafe`); top-level predicate `tm.depth IS NULL OR tm.depth=0 OR (tm.depth=1 AND tm.broadcast)`; `kind IN (...)` list built from `u32::to_string()`; `LIMIT $n+1` probe drives `has_more`; batched per-root participants via `ROW_NUMBER() OVER (PARTITION BY root_event_id …) WHERE rn <= 10` | `events`, `thread_metadata` |
| `get_thread_metadata_by_event` `:755` | single scoped row | `thread_metadata` |

---

#### `feed.rs`

| Fn | Operation | Tables |
|----|-----------|--------|
| `query_mentions` `:125` (builder `:87`) | `INNER JOIN event_mentions m ON e.community_id=m.community_id AND e.id=m.event_id`, `m.pubkey_hex = $`, kinds `{KIND_STREAM_MESSAGE, _V2, KIND_FORUM_POST, KIND_FORUM_COMMENT}`, visible-channel filter, `ORDER BY m.event_created_at DESC LIMIT min(limit, 100)` | `events`, `event_mentions` |
| `query_needs_action` `:186` (builder `:148`) | same join, kinds `{KIND_WORKFLOW_APPROVAL_REQUESTED, KIND_STREAM_REMINDER}` | `events`, `event_mentions` |
| `query_activity` `:235` (builder `:207`) | no join; kinds `{stream msg, v2, forum post, job request/progress/result}` | `events` |
| private `push_visible_channel_filter` `:59` | empty accessible list ⇒ `channel_id IS NULL` only | — |

---

#### `reaction.rs`

| Fn | Operation | Tables |
|----|-----------|--------|
| `add_reaction` `:82` / `add_reaction_tx` (crate-private) `:114` | shared `ADD_REACTION_SQL` `:66`: `INSERT … ON CONFLICT (community_id, event_created_at, event_id, pubkey, emoji) DO UPDATE SET created_at=NOW(), removed_at=NULL, reaction_event_id=COALESCE(EXCLUDED…, reactions…) WHERE reactions.removed_at IS NOT NULL` | `reactions` |
| `remove_reaction` `:140` | `UPDATE SET removed_at=NOW() … removed_at IS NULL` | `reactions` |
| `remove_reaction_by_source_event_id` `:174` | same keyed on `reaction_event_id` | `reactions` |
| `get_active_reaction_record` `:197` | scoped single-row select | `reactions` |
| `set_reaction_event_id` `:238` | backfill `reaction_event_id` on the active row | `reactions` |
| `get_reactions` `:280` | two-step: inner `DISTINCT emoji … LIMIT $4` then all rows for those groups; grouping in Rust; `_cursor` parameter is unused | `reactions` |
| `get_reactions_bulk` `:366` | **one query per input pair** (`GROUP BY emoji`) | `reactions` |

---

#### `dm.rs`

| Fn | Operation | Tables |
|----|-----------|--------|
| `compute_participant_hash` `:48` | pure: sort + dedup pubkeys, SHA-256, no separator | — |
| `find_dm_by_participants` `:65` | `… participant_hash=$2 AND channel_type='dm' AND deleted_at IS NULL LIMIT 1` | `channels` |
| `create_dm` `:101` | validates 2–9 participants and 32-byte keys; tx: idempotency probe → insert `channel_type='dm', visibility='private'` → insert every participant as `'member'` → read back | `channels`, `channel_members` |
| `list_dms_for_user` `:226` | `limit.min(200)`; cursor resolved to `updated_at`; then **one participants query per DM** | `channels`, `channel_members`, `users` |
| `open_dm` `:356` | merges `created_by`, enforces ≤9, fast-path find (+`unhide_dm`), else `create_dm` | `channels`, `channel_members` |
| `hide_dm` `:397` | `UPDATE channel_members SET hidden_at=NOW() … removed_at IS NULL`; 0 rows ⇒ `NotFound` | `channel_members` |
| `unhide_dm` `:429` | clears `hidden_at` (no-op safe) | `channel_members` |
| `list_hidden_dms` `:454` | `hidden_at IS NOT NULL` + live DM channel | `channel_members`, `channels` |

---

#### `user.rs`

| Fn | Operation | Tables |
|----|-----------|--------|
| `ensure_user` `:42` | `INSERT INTO users (community_id, pubkey) … ON CONFLICT DO NOTHING`; returns `rows_affected == 1` | `users` |
| `get_user` `:58` | `query_as` tuple select | `users` |
| `update_user_profile` `:103` | dynamic SET list (`format!` + `AssertSqlSafe`, positional binds); empty string ⇒ NULL | `users` |
| `get_user_by_nip05` `:169` | `LOWER(nip05_handle) = LOWER($2) LIMIT 1` | `users` |
| `search_users` `:224` | `escape_like` `:214` escapes `\ % _`, `LIKE … ESCAPE '\'` over display name / handle / `encode(pubkey,'hex')`; 6-tier `CASE` ranking; `limit.clamp(1, 500)` | `users` |
| `set_agent_owner` `:291` | conditional `UPDATE … WHERE agent_owner_pubkey IS NULL` (first-mint-wins), then existence probe to distinguish not-found from already-owned | `users` |
| `get_agent_channel_policy` `:330` | `channel_add_policy::text` + `agent_owner_pubkey` | `users` |
| `is_agent_owner` `:354` | `SELECT agent_owner_pubkey = $3 … AND agent_owner_pubkey IS NOT NULL` | `users` |
| `set_channel_add_policy` `:374` | Rust-side vocabulary check then `$1::channel_add_policy` cast | `users` |

---

#### `api_token.rs`

| Fn | Operation | Tables |
|----|-----------|--------|
| `create_api_token` `:15` | plain insert of a caller-supplied SHA-256 hash | `api_tokens` |
| `create_api_token_if_under_limit` `:69` | `INSERT … SELECT … WHERE (SELECT COUNT(*) … community_id AND owner_pubkey AND revoked_at IS NULL AND (expires_at IS NULL OR expires_at > NOW())) < 10`; `created_by_self_mint = TRUE` | `api_tokens` |
| `get_api_token_by_hash_including_revoked` `:144` | `WHERE community_id=$1 AND token_hash=$2` | `api_tokens` |
| `list_tokens_by_owner` `:208` | all tokens incl. revoked, `ORDER BY created_at DESC` (**no LIMIT**) | `api_tokens` |
| `revoke_token` `:272` | `UPDATE … WHERE community_id AND id AND owner_pubkey AND revoked_at IS NULL` | `api_tokens` |
| `revoke_all_tokens` `:303` | same without `id` | `api_tokens` |

---

#### `workflow.rs`

| Fn | Operation | Tables |
|----|-----------|--------|
| `create_workflow` `:276` | insert with `'active'`/`TRUE`; documented "No current callers" | `workflows` |
| `upsert_workflow` `:313` | `ON CONFLICT (community_id, id) DO UPDATE … WHERE workflows.owner_pubkey = EXCLUDED.owner_pubkey AND workflows.channel_id IS NOT DISTINCT FROM EXCLUDED.channel_id RETURNING id`; `None` ⇒ `AccessDenied` | `workflows` |
| `get_workflow` `:363` | scoped select | `workflows` |
| `list_channel_workflows` `:389` | `limit.clamp(1, LIST_MAX_LIMIT)`, `offset.max(0)` | `workflows` |
| `list_enabled_channel_workflows` `:425` | `status='active' AND enabled` , `LIMIT LIST_MAX_LIMIT` | `workflows` |
| `list_all_enabled_workflows` `:457` | global scan `JOIN communities … WHERE definition->'trigger'->>'on' = 'schedule' AND c.archived_at IS NULL LIMIT 1000` | `workflows`, `communities` |
| `claim_scheduled_workflow_fire` `:496` | `INSERT … SELECT w.community_id, w.id, $3 FROM workflows w WHERE community_id AND id ON CONFLICT DO NOTHING RETURNING …` | `scheduled_workflow_fires`, `workflows` |
| `latest_scheduled_workflow_fire` `:537` | `MAX(scheduled_for)` | `scheduled_workflow_fires` |
| `attach_scheduled_workflow_run` `:563` | `UPDATE … WHERE workflow_run_id IS NULL` | `scheduled_workflow_fires` |
| `prune_scheduled_workflow_fires_before` `:597` | global `DELETE … WHERE claimed_at < $1` | `scheduled_workflow_fires` |
| `update_workflow` `:620`, `update_workflow_status` `:654`, `set_workflow_enabled` `:684`, `delete_workflow` `:715` | scoped update/delete; 0 rows ⇒ `NotFound`; all documented "No current callers" | `workflows` |
| `delete_workflow_for_owner` `:738` | `DELETE … AND owner_pubkey=$3 RETURNING channel_id` | `workflows` (cascades) |
| `create_workflow_run` `:767` | insert `'pending'`, `execution_trace='[]'` | `workflow_runs` |
| `get_workflow_run` `:795`, `list_workflow_runs` `:818` (`limit.min(1000)`) | scoped reads | `workflow_runs` |
| `update_workflow_run` `:850` | sets status/step/trace/error, stamps `started_at` when the **bind** = `'running'` and it is NULL, `completed_at` for terminal states | `workflow_runs` |
| `create_approval` `:918` | hashes the raw token with SHA-256 (`hash_approval_token` `:33`) then inserts | `workflow_approvals` |
| `get_approval` `:956` / `get_approval_by_stored_hash` `:973` | scoped `(community_id, token)` lookup | `workflow_approvals` |
| `get_run_approvals` `:996` | `ORDER BY step_index, created_at` | `workflow_approvals` |
| `update_approval` `:1031` / `update_approval_by_stored_hash` `:1059` | `UPDATE … WHERE community_id AND token AND status='pending'`; stamps `granted_at`/`denied_at` | `workflow_approvals` |
| `find_by_owner_and_name` `:1169` | `WHERE community_id AND owner_pubkey AND name LIMIT 1` | `workflows` |

---

#### `push.rs`

| Fn | Operation | Tables |
|----|-----------|--------|
| `accept_lease_event` `:213` | tx: SHA-256-derived address lock + author lock (+ push-gate lock when activating) → source-event collision probe → `FOR UPDATE` ordering probe → expire stale active leases → quota + endpoint-uniqueness checks → soft-delete prior kind:30350 → insert event → upsert lease → `backfill_push_match_jobs` | `push_leases`, `events`, `push_match_queue` |
| `replace_active_lease` `:419` / `revoke_lease` `:439` / private `replace_lease` `:447` | upsert whose `ON CONFLICT … WHERE` clause **is** the ordering state machine (`source_created_at`/`source_event_id` then `generation`) | `push_leases`, `push_match_queue` |
| `enqueue_wake` `:580` | batch-of-one wrapper over `enqueue_wakes` | as below |
| `enqueue_wakes` `:619` | tx: `FOR UPDATE` lock of distinct lease rows via `UNNEST` in deterministic order → per-request generation match → one multi-row `INSERT … ON CONFLICT (community_id, endpoint_hash, event_id) DO NOTHING RETURNING …` → set-wise duplicate lookup | `push_leases`, `push_wake_outbox` |
| `claim_due_match_batch` `:819` (+ `_with_loader` `:833`) | CTE picks ONE community, `FOR UPDATE OF q SKIP LOCKED LIMIT $4`, `attempts+1`; then loads events; jobs with no live event are deleted | `push_match_queue`, `events` |
| `reap_exhausted_matches` `:933` | global `DELETE … WHERE attempts >= 8 AND (pending OR expired lease)` | `push_match_queue` |
| `active_match_leases` `:945` | active + endpoint-enabled + unexpired leases for one community | `push_leases` |
| `complete_match_batch` `:970` / `retry_match_batch` `:993` | fenced `DELETE` / `UPDATE` by `claim_id` | `push_match_queue` |
| `claim_due_wakes` `:1021` | CTE joins outbox ⋈ lease (community, author, installation, generation, endpoint_hash) ⋈ live event; `FOR UPDATE OF o SKIP LOCKED`; `attempts+1` | `push_wake_outbox`, `push_leases`, `events` |
| `revalidate_wake_for_send` `:1085` | the load-bearing send-time re-join (`state='sending'`, `lease_until >= now()`, unexpired, active, endpoint-enabled) | `push_wake_outbox`, `push_leases`, `events` |
| `complete_wake` `:1132` / `retry_wake` `:1152` / `fail_wake` `:1174` | fenced state transitions requiring `claim_id` match | `push_wake_outbox` |
| `disable_endpoint_generation` `:1197` | `UPDATE … WHERE generation=$4 AND active AND endpoint_enabled` | `push_leases` |
| `prune_wake_outbox` `:1223` | `DELETE … terminal/expired AND NOT EXISTS (matching push_match_queue row)` | `push_wake_outbox`, `push_match_queue` |
| private `acquire_push_gate_lock` `:24`, `backfill_push_match_jobs` `:52`, `constraint_acceptance_outcome` `:392`, `row_to_claimed_wake` `:1245` | — | — |

---

#### `moderation.rs`

| Fn | Operation | Tables |
|----|-----------|--------|
| `insert_report` `:172` | `ON CONFLICT (community_id, report_event_id) DO UPDATE SET report_event_id = EXCLUDED.report_event_id RETURNING id` (idempotent) | `moderation_reports` |
| `list_reports` `:213` | `($2::text IS NULL OR status = $2)`, `ORDER BY created_at DESC LIMIT $3` | `moderation_reports` |
| `get_report` `:240` / `get_report_by_event` `:263` | scoped single-row reads | `moderation_reports` |
| `resolve_report` `:287` | `UPDATE … AND status='open'` | `moderation_reports` |
| `ban_member` `:314` / `timeout_member` `:371` | upserts keyed `(community_id, pubkey)` | `community_bans` |
| `unban_member` `:347` | `… AND banned = true` | `community_bans` |
| `untimeout_member` `:403` | `… AND muted_until > now()` | `community_bans` |
| `restriction_state` `:441` | computes `banned AND (ban_expires_at IS NULL OR ban_expires_at > now())` and future-only `muted_until`; missing row ⇒ default | `community_bans` |
| `get_ban` `:470` / `list_restricted` `:494` | same expiry-aware projection | `community_bans` |
| `insert_action` `:518` | plain insert `RETURNING id` | `moderation_actions` |
| `list_actions` `:549` | `ORDER BY created_at DESC LIMIT $2` | `moderation_actions` |

---

#### `admin_moderation.rs` — deployment-global reads (documented exception)

| Fn | Operation | Tables |
|----|-----------|--------|
| `list_reports` `:85` | keyset `(created_at, id) < ($7,$8)`, optional community/status/type/target/time filters, `bounded_limit` clamps to `MAX_PAGE_SIZE=200` | `moderation_reports`, `communities` |
| `get_report` `:132` | `WHERE r.id = $1` (no community predicate) | `moderation_reports`, `communities` |
| `list_feedback` `:181` / `get_feedback` `:200` | newest-first / by row id | `product_feedback`, `communities` |

---

#### `product_feedback.rs`

| Fn | Operation | Tables |
|----|-----------|--------|
| `insert` `:59` | `ON CONFLICT (event_id) DO UPDATE SET event_id = EXCLUDED.event_id RETURNING id` — deployment-wide idempotency, first community keeps provenance | `product_feedback` |
| `list` `:89` | `ORDER BY received_at DESC, id LIMIT $1` — **no community filter** | `product_feedback` |

---

#### `relay_members.rs`

| Fn | Operation | Tables |
|----|-----------|--------|
| `is_relay_member` `:31`, `get_relay_member` `:41`, `list_relay_members` `:69` | scoped reads | `relay_members` |
| `add_relay_member` `:97` | `ON CONFLICT (community_id, pubkey) DO NOTHING` | `relay_members` |
| `claim_relay_membership` `:122` | tx: member insert + optional `join_policy_acceptances` insert | `relay_members`, `join_policy_acceptances` |
| `has_join_policy_acceptance` `:160` | existence probe | `join_policy_acceptances` |
| `remove_relay_member` `:196` | `DELETE … AND role <> 'owner'`, then one probe to distinguish `IsOwner`/`NotFound` | `relay_members` |
| `remove_relay_member_if_role` `:242` | `DELETE … AND role = $3`, then probe → `IsOwner`/`RoleMismatch`/`NotFound` | `relay_members` |
| `update_relay_member_role` `:287` | `UPDATE … AND role <> 'owner'` | `relay_members` |
| `bootstrap_owner` `:320` | tx: upsert owner → demote other owners to `'admin'`; does **not** enforce the per-owner limit | `relay_members` |
| `owner_count_advisory_lock_key` `:384` | pure FNV-1a over the hex pubkey | — |
| `transfer_ownership` `:411` | tx: advisory lock on transferee → `SELECT … role='owner' FOR UPDATE` → expected-owner check → cross-community owner count vs `MAX_COMMUNITIES_PER_OWNER` → upsert new owner → demote others to `'member'` | `relay_members` |
| `backfill_from_allowlist` `:516` | `information_schema` existence probe → empty-target guard → `INSERT … SELECT encode(pubkey,'hex') FROM pubkey_allowlist WHERE community_id=$1` | `pubkey_allowlist`, `relay_members` |

---

#### `archived_identities.rs`, `git_repo.rs`

| Fn | Operation | Tables |
|----|-----------|--------|
| `archived_identities::is_archived` `:34` | `SELECT 1 …` | `archived_identities` |
| `archive` `:50` | `ON CONFLICT (community_id, pubkey) DO NOTHING` | `archived_identities` |
| `unarchive` `:83` | scoped `DELETE` | `archived_identities` |
| `list_archived` `:95` | `ORDER BY archived_at ASC` | `archived_identities` |
| `git_repo::repo_name_owner` `:47` | scoped owner read | `git_repo_names` |
| `reserve_repo_name` `:81` | `INSERT … ON CONFLICT (community_id, repo_id) DO NOTHING RETURNING owner_pubkey`, then classification read | `git_repo_names` |
| `count_repos_for_owner` `:142` | `COUNT(*) … owner_pubkey=$2` (quota input) | `git_repo_names` |
| `release_repo_name` `:164` | owner-scoped `DELETE` | `git_repo_names` |

---

#### `usage.rs` — operator rollups (all cross-community by design)

| Fn | Operation | Tables |
|----|-----------|--------|
| `community_count` `:20` | `COUNT(*)` | `communities` |
| `user_counts` `:41` | `COUNT(*) FILTER (…)` split human/agent, `WHERE deactivated_at IS NULL GROUP BY community_id` | `users` |
| `channel_counts` `:79` | `GROUP BY community_id, channel_type WHERE deleted_at IS NULL` | `channels` |
| `message_counts` `:113` | `WHERE kind = 9 AND deleted_at IS NULL GROUP BY community_id` | `events` |
| `relay_member_counts` `:146` | `GROUP BY community_id, role` | `relay_members` |
| `workflow_counts` `:179` | `GROUP BY community_id, status` | `workflows` |
| `git_repo_counts` `:210` | `GROUP BY community_id` | `git_repo_names` |
| `active_user_counts` `:254` | `format!`-interpolated `INTERVAL '{interval_sql}'` (`&'static str`) + `AssertSqlSafe`; 3-way human/agent/unknown classification | `events`, `users` |
| `active_channel_counts` `:308` | same interval interpolation; `COUNT(DISTINCT channel_id) WHERE kind = 9` | `events` |
| `community_hosts` `:347` | `SELECT id, host FROM communities` | `communities` |

---

#### `partition.rs`, `migration.rs`, `replica_fence.rs`

| Fn | Operation | Tables |
|----|-----------|--------|
| `partition::ensure_future_partitions` `:15` | loops months, computes suffix/date strings, calls private `ensure_partition` `:76` | `pg_catalog.pg_class`/`pg_namespace`; DDL `CREATE TABLE … PARTITION OF` on `events` and `delivery_log` |
| `migration::run_migrations` `:14` | pre-flight `reject_legacy_nip_rs_cardinality_ambiguity` `:34` → `MIGRATOR.run(pool)` → `verify_floor_guard_catalog` | `_sqlx_migrations`, `events`, all |
| `replica_fence::verify_floor_guard_catalog` `:145` | catalog shape check on `events` + every partition (`pg_trigger` tgtype bits) | `pg_inherits`, `pg_class`, `pg_trigger` |
| `replica_fence::verify_floor_guard_behavior` `:199` | one rolled-back tx: `SHOW buzz.created_at_floor`, `SET CONSTRAINTS ALL IMMEDIATE`, 5 adversarial inserts/updates under savepoints | `communities`, `events` (rolled back) |
| `replica_fence::probe_once` `:466` (private `sample_writer` `:404`, `replica_covers` `:449`) | writer: `clock_timestamp()`, then `pg_stat_activity`/`pg_prepared_xacts` scan, then `pg_current_wal_lsn()`; replica: `pg_last_wal_replay_lsn() >= $1` gated on `pg_is_in_recovery()` | system views only |
| `replica_fence::run_probe` `:488` | 5 s loop, closes the fence on any error | — |
| `ReplicaFence::new` `:88`, `close` `:97`, `verified_through` `:114`, `covers` `:129`, `force_open_for_tests` `:136` | in-memory atomics | — |

---

#### Public constants

| Constant | Value | File:line |
|----------|-------|-----------|
| `admin_moderation::MAX_PAGE_SIZE` | 200 | `crates/buzz-db/src/admin_moderation.rs:15` |
| `event::D_TAG_MAX_LEN` | 1024 | `crates/buzz-db/src/event.rs:124` |
| `feed::FEED_MAX_LIMIT` | 100 | `crates/buzz-db/src/feed.rs:29` |
| `moderation::MODERATION_ACTION_CHECK_VOCAB` | 12 strings | `crates/buzz-db/src/moderation.rs:104` |
| `push::MAX_MATCH_ATTEMPTS` | 8 | `crates/buzz-db/src/push.rs:70` |
| `relay_members::MAX_COMMUNITIES_PER_OWNER` | 3 | `crates/buzz-db/src/relay_members.rs:379` |
| `replica_fence::CREATED_AT_FLOOR_SECS` | 960 | `crates/buzz-db/src/replica_fence.rs:51` |
| `replica_fence::FENCE_CLOCK_MARGIN_SECS` | 5 | `crates/buzz-db/src/replica_fence.rs:59` |
| `replica_fence::PROBE_INTERVAL` | 5 s | `crates/buzz-db/src/replica_fence.rs:62` |
| `replica_fence::FENCE_STALENESS` | 30 s | `crates/buzz-db/src/replica_fence.rs:66` |
| `workflow::LIST_DEFAULT_LIMIT` | 100 | `crates/buzz-db/src/workflow.rs:25` |
| `workflow::LIST_MAX_LIMIT` | 1000 | `crates/buzz-db/src/workflow.rs:27` |
