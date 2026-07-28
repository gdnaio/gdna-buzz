## Module: buzz-db (`crates/buzz-db`)

### Data Model

buzz-db owns the entire Postgres schema. The authoritative source is
`migrations/0001_initial_schema.sql` … `migrations/0024_event_ttl_refresh_shared_lock.sql`
(24 files, embedded at compile time by `sqlx::migrate!("../../migrations")` —
`crates/buzz-db/src/migration.rs:11`). `schema/schema.sql` is a separate
"desired-state" file that is **not** applied by the crate and has drifted (see
`buzz-db-debt.md`).

Counts in the end state produced by the migrations: **37 base tables**
(31 tenant-scoped + 6 push-gateway operator-global tables, of which 9 are
registered operator-global), **14 declarative partition children** (8 on
`events`, 6 on `delivery_log`), **10 enum types**, **63 distinct explicitly
created indexes** (66 `CREATE INDEX` statements, 3 of which re-create
`idx_events_search_tsv`), **9 trigger functions**, **9 triggers**.

---

#### 1. Enum types (`migrations/0001_initial_schema.sql:28-37`)

| Type | Values | Line |
|------|--------|------|
| `channel_type` | `stream`, `forum`, `dm`, `workflow` | `migrations/0001_initial_schema.sql:28` |
| `channel_visibility` | `open`, `private` | `migrations/0001_initial_schema.sql:29` |
| `member_role` | `owner`, `admin`, `member`, `guest`, `bot` | `migrations/0001_initial_schema.sql:30` |
| `workflow_status` | `active`, `disabled`, `archived` | `migrations/0001_initial_schema.sql:31` |
| `run_status` | `pending`, `running`, `waiting_approval`, `completed`, `failed`, `cancelled` | `migrations/0001_initial_schema.sql:32` |
| `approval_status` | `pending`, `granted`, `denied`, `expired` | `migrations/0001_initial_schema.sql:33` |
| `delivery_method` | `webhook`, `websocket` | `migrations/0001_initial_schema.sql:34` |
| `subscription_status` | `active`, `paused`, `deleted` | `migrations/0001_initial_schema.sql:35` |
| `pause_reason` | `user`, `system`, `rate_limit` | `migrations/0001_initial_schema.sql:36` |
| `channel_add_policy` | `anyone`, `owner_only`, `nobody` | `migrations/0001_initial_schema.sql:37` |

Extension: `pgcrypto` (`migrations/0001_initial_schema.sql:24`) for `gen_random_uuid()`.

CHECK-constraint vocabularies (not Postgres enums):

| Table.column | Allowed values | Line |
|--------------|----------------|------|
| `relay_members.role` | `owner`, `admin`, `member` | `migrations/0001_initial_schema.sql:577` |
| `archived_identities.consent_path` | `self`, `owner`, `admin` | `migrations/0001_initial_schema.sql:592` |
| `moderation_reports.target_kind` | `event`, `pubkey`, `blob` | `migrations/0006_moderation.sql:22` |
| `moderation_reports.status` | `open`, `resolved`, `dismissed`, `escalated` | `migrations/0006_moderation.sql:35-36` |
| `moderation_actions.action` | `delete_message`, `kick`, `ban`, `unban`, `timeout`, `untimeout`, `dismiss_report`, `escalate`, `resolve:delete`, `resolve:kick`, `resolve:ban`, `resolve:timeout` | `migrations/0006_moderation.sql:97-101` |
| `moderation_actions.matched_principal` | `self`, `owner` (or NULL) | `migrations/0006_moderation.sql:113` |
| `product_feedback.category` | `bug`, `praise`, `needs-work` (or NULL) | `migrations/0017_product_feedback.sql:9` |
| `push_leases.max_class`, `push_wake_outbox.class` | `silent`, `default`, `time_sensitive`, `urgent` | `migrations/0012_push_leases.sql:13`, `:38` |
| `push_wake_outbox.state` | `pending`, `sending`, `delivered`, `failed` | `migrations/0012_push_leases.sql:40` |
| `push_match_queue.state` | `pending`, `matching` | `migrations/0018_push_match_queue.sql:8` |
| `push_gateway_installations.app_profile` | `buzz-ios-production`, `buzz-ios-sandbox` | `migrations/0015_push_gateway_authority.sql:17` |

---

#### 2. `communities` — operator-global tenant registry

`migrations/0001_initial_schema.sql:53-59`; extended by `0003` and `0016`.

| Column | Type | Null | Default | Notes |
|--------|------|------|---------|-------|
| `id` | UUID | NOT NULL | `gen_random_uuid()` | PK; **is** the community key |
| `host` | VARCHAR(255) | NOT NULL | — | stored pre-normalized |
| `signing_key` | BYTEA | NULL | — | |
| `created_at` | TIMESTAMPTZ | NOT NULL | `NOW()` | |
| `icon` | TEXT | NULL | — | added `migrations/0003_community_icon.sql:12` (NIP-11 `icon`) |
| `archived_at` | TIMESTAMPTZ | NULL | — | added `migrations/0016_community_archival.sql:3` |

- PK: `(id)` — `migrations/0001_initial_schema.sql:54`
- CHECK `chk_communities_id_not_nil`: `id <> '00000000-…'::uuid` — `:58`
- UNIQUE INDEX `idx_communities_host` on `(lower(host))` — `:61`

---

#### 3. `channels`

`migrations/0001_initial_schema.sql:72-99`

| Column | Type | Null | Default |
|--------|------|------|---------|
| `id` | UUID | NOT NULL | `gen_random_uuid()` |
| `community_id` | UUID | NOT NULL | FK → `communities(id)` |
| `name` | VARCHAR(255) | NOT NULL | — |
| `channel_type` | `channel_type` | NOT NULL | `'stream'` |
| `visibility` | `channel_visibility` | NOT NULL | `'open'` |
| `description` | TEXT | NULL | — |
| `canvas` | TEXT | NULL | — |
| `created_by` | BYTEA | NOT NULL | — |
| `created_at` | TIMESTAMPTZ | NOT NULL | `NOW()` |
| `updated_at` | TIMESTAMPTZ | NOT NULL | `NOW()` |
| `archived_at` | TIMESTAMPTZ | NULL | — |
| `deleted_at` | TIMESTAMPTZ | NULL | — (soft delete) |
| `nip29_group_id` | VARCHAR(255) | NULL | — |
| `topic_required` | BOOLEAN | NOT NULL | `FALSE` |
| `max_members` | INT | NULL | — |
| `topic` | TEXT | NULL | — |
| `topic_set_by` | BYTEA | NULL | — |
| `topic_set_at` | TIMESTAMPTZ | NULL | — |
| `purpose` | TEXT | NULL | — |
| `purpose_set_by` | BYTEA | NULL | — |
| `purpose_set_at` | TIMESTAMPTZ | NULL | — |
| `participant_hash` | BYTEA | NULL | — (DM identity, SHA-256) |
| `ttl_seconds` | INT | NULL | — |
| `ttl_deadline` | TIMESTAMPTZ | NULL | — |

- PK `(community_id, id)` — `:97`
- CHECK `chk_channels_id_not_nil` — `:98`
- UNIQUE `idx_channels_nip29_group (community_id, nip29_group_id) WHERE nip29_group_id IS NOT NULL` — `:102`
- UNIQUE `idx_channels_dm_hash (community_id, participant_hash) WHERE participant_hash IS NOT NULL` — `:104`
- `idx_channels_community_type (community_id, channel_type)` — `:106`
- `idx_channels_community_visibility (community_id, visibility)` — `:107`
- `idx_channels_created_by (community_id, created_by)` — `:108`
- `idx_channels_ttl_expiry (ttl_deadline) WHERE ttl_seconds IS NOT NULL AND archived_at IS NULL AND deleted_at IS NULL` — `:109` (**not** community-leading; it is the reaper's global scan index)
- Trigger `trg_channels_community_id_immutable` BEFORE UPDATE FOR EACH ROW → `channels_community_id_immutable()` raises `check_violation` when `community_id` changes — `:115-128`

---

#### 4. `channel_members`

`migrations/0001_initial_schema.sql:132-145`

| Column | Type | Null | Default |
|--------|------|------|---------|
| `community_id` | UUID | NOT NULL | FK → `communities(id)` |
| `channel_id` | UUID | NOT NULL | — |
| `pubkey` | BYTEA | NOT NULL | — |
| `role` | `member_role` | NOT NULL | `'member'` |
| `joined_at` | TIMESTAMPTZ | NOT NULL | `NOW()` |
| `invited_by` | BYTEA | NULL | — |
| `removed_at` | TIMESTAMPTZ | NULL | — (soft delete) |
| `removed_by` | BYTEA | NULL | — |
| `hidden_at` | TIMESTAMPTZ | NULL | — (per-user DM hide) |

- PK `(community_id, channel_id, pubkey)` — `:142`
- FK `(community_id, channel_id)` → `channels(community_id, id) ON DELETE CASCADE` — `:143`
- `idx_channel_members_pubkey (community_id, pubkey) WHERE removed_at IS NULL` — `:147`

---

#### 5. `users`

`migrations/0001_initial_schema.sql:154-175`

| Column | Type | Null | Default |
|--------|------|------|---------|
| `community_id` | UUID | NOT NULL | FK → `communities(id)` |
| `pubkey` | BYTEA | NOT NULL | — |
| `nip05_handle` | VARCHAR(255) | NULL | — |
| `display_name` | VARCHAR(255) | NULL | — |
| `avatar_url` | TEXT | NULL | — |
| `about` | TEXT | NULL | — |
| `agent_type` | VARCHAR(255) | NULL | — |
| `capabilities` | JSONB | NULL | — |
| `okta_user_id` | VARCHAR(255) | NULL | — |
| `created_at` | TIMESTAMPTZ | NOT NULL | `NOW()` |
| `updated_at` | TIMESTAMPTZ | NOT NULL | `NOW()` |
| `deactivated_at` | TIMESTAMPTZ | NULL | — |
| `metadata_event_id` | BYTEA | NULL | — |
| `agent_owner_pubkey` | BYTEA | NULL | — |
| `channel_add_policy` | `channel_add_policy` | NOT NULL | `'anyone'` |

- PK `(community_id, pubkey)` — `:170`
- CHECK `chk_users_pubkey_len`: `LENGTH(pubkey) = 32` — `:171`
- Self-FK `(community_id, agent_owner_pubkey)` → `users(community_id, pubkey) ON DELETE SET NULL` — `:173`
- UNIQUE `idx_users_nip05 (community_id, lower(nip05_handle)) WHERE nip05_handle IS NOT NULL` — `:178`
- UNIQUE `idx_users_okta (community_id, okta_user_id) WHERE okta_user_id IS NOT NULL` — `:180`

---

#### 6. `events` — monthly range-partitioned

`migrations/0001_initial_schema.sql:190-235`

| Column | Type | Null | Default | Notes |
|--------|------|------|---------|-------|
| `community_id` | UUID | NOT NULL | FK → `communities(id)` | |
| `id` | BYTEA | NOT NULL | — | 32-byte Nostr event id |
| `pubkey` | BYTEA | NOT NULL | — | |
| `created_at` | TIMESTAMPTZ | NOT NULL | — | partition key |
| `kind` | INT | NOT NULL | — | Nostr kind widened to i32 |
| `tags` | JSONB | NOT NULL | — | |
| `content` | TEXT | NOT NULL | — | |
| `search_tsv` | TSVECTOR | — | GENERATED ALWAYS … STORED | see below |
| `sig` | BYTEA | NOT NULL | — | |
| `received_at` | TIMESTAMPTZ | NOT NULL | `NOW()` | |
| `channel_id` | UUID | NULL | — | **no FK** (partitioned table) |
| `deleted_at` | TIMESTAMPTZ | NULL | — | soft delete |
| `d_tag` | TEXT | NULL | — | NIP-33 coordinate |
| `not_before` | BIGINT | NULL | — | NIP-ER reminder due time |
| `delivered_at` | BIGINT | NULL | — | reminder delivery claim stamp |

- PK `(community_id, created_at, id)` — `:234`; `PARTITION BY RANGE (created_at)` — `:235`
- Partitions declared in `0001`: `events_p_past` (MINVALUE→2026-01-01), `events_p2026_01` … `events_p2026_06`, `events_p_future` (2026-07-01→MAXVALUE) — `:237-252`

**`search_tsv` generated column (verbatim, `migrations/0001_initial_schema.sql:222-226`):**

```sql
    search_tsv  TSVECTOR GENERATED ALWAYS AS (
        CASE WHEN kind IN (1059, 30300, 30622, 44100, 44101) THEN NULL::tsvector
             ELSE to_tsvector('simple', content)
        END
    ) STORED,
```

Kinds excluded in `0001`: 1059 (KIND_GIFT_WRAP), 30300 (KIND_EVENT_REMINDER),
30622 (KIND_DM_VISIBILITY), 44100 (KIND_MEMBER_ADDED_NOTIFICATION),
44101 (KIND_MEMBER_REMOVED_NOTIFICATION) — rationale at `:207-221`.

The column is then rewritten three times:

1. `migrations/0005_agent_turn_metric_fts.sql:29-33` — drops and re-adds, adding **44200** (agent turn metrics):
```sql
ALTER TABLE events ADD COLUMN search_tsv TSVECTOR GENERATED ALWAYS AS (
    CASE WHEN kind IN (1059, 30300, 30622, 44100, 44101, 44200) THEN NULL::tsvector
         ELSE to_tsvector('simple', content)
    END
) STORED;
```
2. `migrations/0008_fresh_install_search_allowlist.sql:11-22` — **only when `events` is empty**, replaces the negative exclusion list with a positive allowlist:
```sql
        ALTER TABLE events ADD COLUMN search_tsv TSVECTOR GENERATED ALWAYS AS (
            CASE WHEN kind IN (0, 9, 40002, 45001, 45003)
                 THEN to_tsvector('simple', content)
                 ELSE NULL::tsvector
            END
        ) STORED;
```
3. `migrations/0014_push_lease_fts.sql:11-33` — reads the *current* expression from `pg_attrdef` and wraps it so kind **30350** (NIP-PL lease) always yields NULL:
```sql
        'ALTER TABLE events ADD COLUMN search_tsv TSVECTOR GENERATED ALWAYS AS (CASE WHEN kind = 30350 THEN NULL::tsvector ELSE (%s) END) STORED',
```

Net effect: a **fresh** install ends with allowlist `{0, 9, 40002, 45001, 45003}`
minus `30350`; a **brownfield** install ends with the negative exclusion list
`{1059, 30300, 30622, 44100, 44101, 44200}` plus `30350`. This divergence is
asserted by the test at `crates/buzz-db/src/migration.rs:1075-1109`.

**Indexes on `events`:**

| Index | Columns / type | Line |
|-------|----------------|------|
| `idx_events_community_id` | btree `(community_id, id, created_at DESC)` | `migrations/0001_initial_schema.sql:257` |
| `idx_events_community_channel_created` | btree `(community_id, channel_id, created_at DESC, id)` | `:259` |
| `idx_events_community_pubkey_kind_created` | btree `(community_id, pubkey, kind, created_at DESC, id)` | `:261` |
| `idx_events_community_kind_created` | btree `(community_id, kind, created_at DESC, id)` | `:263` |
| `idx_events_community_deleted` | btree `(community_id, deleted_at)` | `:265` |
| `idx_events_addressable` | btree `(community_id, kind, pubkey, channel_id, deleted_at)` | `:267` |
| `idx_events_parameterized` | btree `(community_id, kind, pubkey, d_tag, created_at DESC, id) WHERE d_tag IS NOT NULL AND deleted_at IS NULL` | `:269` |
| `idx_events_not_before` | btree `(community_id, not_before) WHERE not_before IS NOT NULL AND deleted_at IS NULL AND delivered_at IS NULL` | `:272` |
| `idx_events_search_tsv` | **GIN** `(search_tsv)` | `:278`; re-created at `0005:33`, `0008:21`, `0014:33` |
| `idx_events_tags_gin` | **GIN** `(tags jsonb_path_ops)` | `migrations/0004_events_tags_gin.sql:21` |

**Triggers on `events`** (all row-level):

| Trigger | Timing | Function | Line |
|---------|--------|----------|------|
| `trg_events_nip_rs_watermark` | BEFORE INSERT | `guard_nip_rs_watermark()` | `migrations/0009_nip_rs_database_guards.sql:70`; body replaced at `0010:4` and `0011:62` |
| `trg_events_purge_soft_deleted_nip_rs` | AFTER UPDATE OF `deleted_at` | `purge_soft_deleted_nip_rs()` | `0009:104`; body replaced at `0011:123` |
| `trg_events_guard_nip_rs_hard_delete` | BEFORE DELETE `WHEN (OLD.kind = 30078 AND OLD.d_tag ~ '^read-state:[0-9a-f]{32}$')` | `guard_nip_rs_hard_delete()` | `migrations/0011_nip_rs_exact_tag_cardinality.sql:56-60` |
| `events_enqueue_push_match` | AFTER INSERT | `enqueue_push_match_job()` | `migrations/0018_push_match_queue.sql:36`; body replaced at `0023:22` |
| `trg_events_purge_soft_deleted_buzz_mesh_status` | AFTER UPDATE OF `deleted_at` | `purge_soft_deleted_buzz_mesh_status()` | `migrations/0019_mesh_status_retention.sql:41` |
| `events_created_at_floor` | CONSTRAINT, AFTER INSERT OR UPDATE OF `created_at, channel_id`, DEFERRABLE INITIALLY DEFERRED | `events_created_at_floor_guard()` | `migrations/0021_created_at_fence_floor.sql:70-74` |
| `events_refresh_channel_ttl` | CONSTRAINT, AFTER INSERT, DEFERRABLE INITIALLY DEFERRED | `refresh_channel_ttl_after_event_insert()` | `migrations/0022_event_ttl_refresh.sql:37-39`; body replaced at `0024:25` |

---

#### 7. `event_mentions`

`migrations/0001_initial_schema.sql:286-294`

| Column | Type | Null |
|--------|------|------|
| `community_id` | UUID | NOT NULL (FK → `communities(id)`) |
| `pubkey_hex` | VARCHAR(64) | NOT NULL |
| `event_id` | BYTEA | NOT NULL |
| `event_created_at` | TIMESTAMPTZ | NOT NULL |
| `channel_id` | UUID | NULL |
| `event_kind` | INT | NULL |

- PK `(community_id, pubkey_hex, event_id)` — `:293`
- `idx_event_mentions_pubkey_created (community_id, pubkey_hex, event_created_at DESC)` — `:296`
- `idx_event_mentions_pubkey_kind_created (community_id, pubkey_hex, event_kind, event_created_at DESC)` — `:298`
- `idx_event_mentions_community_event (community_id, event_id)` — `migrations/0007_nip_rs_retention.sql:26`
- Trigger `trg_event_mentions_require_live_event` BEFORE INSERT → `guard_event_mention_live()`: for `event_kind = 30078` only, takes `FOR KEY SHARE` on the live `events` row and returns NULL (silently skips) if it is gone — `migrations/0009_nip_rs_database_guards.sql:111-137`
- No FK to `events` (partitioned parent) — denormalized index, `migrations/0007_nip_rs_retention.sql:85-87`

---

#### 8. `subscriptions`

`migrations/0001_initial_schema.sql:304-323`. **No Rust module in this crate reads or writes it.**

| Column | Type | Null | Default |
|--------|------|------|---------|
| `community_id` | UUID | NOT NULL | FK → `communities(id)` |
| `id` | VARCHAR(255) | NOT NULL | — |
| `owner_pubkey` | BYTEA | NOT NULL | — |
| `filter_kinds` / `filter_authors` / `filter_channel_ids` | JSONB | NULL | — |
| `filter_since` / `filter_until` | TIMESTAMPTZ | NULL | — |
| `delivery_method` | `delivery_method` | NOT NULL | `'webhook'` |
| `delivery_url` | TEXT | NULL | — |
| `status` | `subscription_status` | NOT NULL | `'active'` |
| `pause_reason` | `pause_reason` | NULL | — |
| `delivered_count` / `error_count` | BIGINT | NOT NULL | `0` |
| `created_at` / `updated_at` | TIMESTAMPTZ | NOT NULL | `NOW()` |

- PK `(community_id, id)` — `:321`; FK `(community_id, owner_pubkey)` → `users` — `:322`

---

#### 9. `delivery_log` — monthly range-partitioned

`migrations/0001_initial_schema.sql:329-341`. **No Rust module in this crate reads or writes it**; only `partition.rs` manages its partitions.

| Column | Type | Null | Default |
|--------|------|------|---------|
| `community_id` | UUID | NOT NULL | FK → `communities(id)` |
| `id` | BIGINT | NOT NULL | `GENERATED ALWAYS AS IDENTITY` |
| `subscription_id` | VARCHAR(255) | NULL | — |
| `event_id` | BYTEA | NULL | — |
| `method` | `delivery_method` | NULL | — |
| `delivered_at` | TIMESTAMPTZ | NOT NULL | `NOW()` (partition key) |
| `success` | BOOLEAN | NULL | — |
| `http_status` | INT | NULL | — |
| `error_message` | TEXT | NULL | — |
| `attempt_number` | INT | NULL | `1` |

- PK `(delivered_at, id)` — `:340` — the **only** tenant-scoped PK not led by
  `community_id`; explicitly allowlisted by the lint at
  `crates/buzz-db/src/migration.rs:497-501` (`is_allowed_partition_primary_key_exception`).
- Partitions: `delivery_log_p_past`, `delivery_log_p2026_03` … `_p2026_06`, `_p_future` — `:343-354`
- `idx_delivery_log_community_sub (community_id, subscription_id)` — `:356`

---

#### 10. `workflows`, `workflow_runs`, `workflow_approvals`, `scheduled_workflow_fires`

`workflows` — `migrations/0001_initial_schema.sql:362-377`

| Column | Type | Null | Default |
|--------|------|------|---------|
| `community_id` | UUID | NOT NULL | FK → `communities(id)` |
| `id` | UUID | NOT NULL | `gen_random_uuid()` |
| `name` | VARCHAR(255) | NOT NULL | — |
| `owner_pubkey` | BYTEA | NOT NULL | — |
| `channel_id` | UUID | NULL | — |
| `definition` | JSONB | NOT NULL | — |
| `definition_hash` | BYTEA | NOT NULL | — |
| `status` | `workflow_status` | NOT NULL | `'active'` |
| `enabled` | BOOLEAN | NOT NULL | `TRUE` |
| `created_at` / `updated_at` | TIMESTAMPTZ | NOT NULL | `NOW()` |

PK `(community_id, id)` `:374`; FK → `users(community_id, owner_pubkey)` `:375`;
FK → `channels(community_id, channel_id)` `:376`.
Indexes: `idx_workflows_channel_active (community_id, channel_id, status, enabled)` `:379`;
`idx_workflows_enabled (enabled, status) WHERE enabled` `:382` (deliberately **not** community-leading — scheduler scan).

`workflow_runs` — `migrations/0001_initial_schema.sql:386-402`: `community_id`,
`id` (default `gen_random_uuid()`), `workflow_id`, `status run_status NOT NULL DEFAULT 'pending'`,
`trigger_event_id BYTEA`, `current_step INT NOT NULL DEFAULT 0`,
`execution_trace JSONB NOT NULL DEFAULT '[]'`, `trigger_context JSONB`,
`started_at`, `completed_at`, `error_message`, `created_at NOT NULL DEFAULT NOW()`.
PK `(community_id, id)` `:399`; FK `(community_id, workflow_id)` → `workflows … ON DELETE CASCADE` `:400`.
Indexes `idx_workflow_runs_workflow (community_id, workflow_id)` `:404`,
`idx_workflow_runs_status (community_id, status)` `:405`.

`workflow_approvals` — `migrations/0001_initial_schema.sql:411-431`: `community_id`,
`token BYTEA NOT NULL` (SHA-256 hash, **PK component**), `workflow_id`, `run_id`,
`step_id VARCHAR(64)`, `step_index INT`, `approver_spec TEXT`,
`status approval_status NOT NULL DEFAULT 'pending'`, `approver_pubkey BYTEA`,
`note TEXT`, `granted_at`, `denied_at`, `expires_at TIMESTAMPTZ NOT NULL`,
`created_at NOT NULL DEFAULT NOW()`.
PK `(community_id, token)` `:426`; FKs to `workflows` and `workflow_runs`, both
`ON DELETE CASCADE` `:427-430`. Indexes `idx_workflow_approvals_workflow`,
`_run`, `_status` (all community-leading) `:433-435`.

`scheduled_workflow_fires` — `migrations/0001_initial_schema.sql:451-462`:
`community_id`, `workflow_id`, `scheduled_for TIMESTAMPTZ NOT NULL`,
`claimed_at NOT NULL DEFAULT NOW()`, `workflow_run_id UUID NULL`.
PK `(community_id, workflow_id, scheduled_for)` `:457` — the at-most-once cron claim.
FK → `workflows … ON DELETE CASCADE` `:458`; FK → `workflow_runs … ON DELETE NO ACTION` `:460`.
Index `idx_scheduled_fires_claimed_at (claimed_at)` `:466` (global; janitor prune).

---

#### 11. `api_tokens`

`migrations/0001_initial_schema.sql:472-489`

| Column | Type | Null | Default |
|--------|------|------|---------|
| `community_id` | UUID | NOT NULL | FK → `communities(id)` |
| `id` | UUID | NOT NULL | `gen_random_uuid()` |
| `token_hash` | BYTEA | NOT NULL | — (SHA-256) |
| `owner_pubkey` | BYTEA | NOT NULL | — |
| `name` | VARCHAR(255) | NOT NULL | — |
| `scopes` | JSONB | NOT NULL | — |
| `channel_ids` | JSONB | NULL | — |
| `created_at` | TIMESTAMPTZ | NOT NULL | `NOW()` |
| `expires_at` / `last_used_at` / `revoked_at` | TIMESTAMPTZ | NULL | — |
| `revoked_by` | BYTEA | NULL | — |
| `created_by_self_mint` | BOOLEAN | NOT NULL | `FALSE` |

PK `(community_id, id)` `:486`; FK `(community_id, owner_pubkey)` → `users` `:487`;
CHECK `chk_api_tokens_hash_len`: `LENGTH(token_hash) = 32` `:488`;
UNIQUE `idx_api_tokens_hash (community_id, token_hash)` `:491`.

---

#### 12. `rate_limit_violations` — operator-global

`migrations/0001_initial_schema.sql:498-507`. **No Rust module in this crate touches it.**
`id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY`, `community_id UUID NULL`
(attribution label only), `pubkey BYTEA`, `violation_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`,
`limit_type VARCHAR(64)`, `limit_value INT`, `actual_value INT`, `action_taken VARCHAR(64)`.

---

#### 13. `thread_metadata`

`migrations/0001_initial_schema.sql:512-528`

| Column | Type | Null | Default |
|--------|------|------|---------|
| `community_id` | UUID | NOT NULL | FK → `communities(id)` |
| `event_created_at` | TIMESTAMPTZ | NOT NULL | — |
| `event_id` | BYTEA | NOT NULL | — |
| `channel_id` | UUID | NOT NULL | — |
| `parent_event_id` | BYTEA | NULL | — |
| `parent_event_created_at` | TIMESTAMPTZ | NULL | — |
| `root_event_id` | BYTEA | NULL | — |
| `root_event_created_at` | TIMESTAMPTZ | NULL | — |
| `depth` | INT | NOT NULL | `0` |
| `reply_count` | INT | NOT NULL | `0` (materialized counter) |
| `descendant_count` | INT | NOT NULL | `0` (materialized counter) |
| `last_reply_at` | TIMESTAMPTZ | NULL | — |
| `broadcast` | BOOLEAN | NOT NULL | `FALSE` |

PK `(community_id, event_created_at, event_id)` `:526`; FK `(community_id, channel_id)` → `channels` `:527`.
Indexes: `idx_thread_metadata_parent (community_id, parent_event_id)` `:530`,
`_root (community_id, root_event_id)` `:531`,
`_channel_depth (community_id, channel_id, depth, event_created_at)` `:532`,
`_event_id (community_id, event_id)` `:534`.

---

#### 14. `reactions`

`migrations/0001_initial_schema.sql:539-549`

| Column | Type | Null | Default |
|--------|------|------|---------|
| `community_id` | UUID | NOT NULL | FK → `communities(id)` |
| `event_created_at` | TIMESTAMPTZ | NOT NULL | — |
| `event_id` | BYTEA | NOT NULL | — |
| `pubkey` | BYTEA | NOT NULL | — |
| `emoji` | VARCHAR(64) | NOT NULL | — |
| `created_at` | TIMESTAMPTZ | NOT NULL | `NOW()` |
| `removed_at` | TIMESTAMPTZ | NULL | — (soft delete) |
| `reaction_event_id` | BYTEA | NULL | — (source kind:7 id) |

PK `(community_id, event_created_at, event_id, pubkey, emoji)` `:548`.
Indexes: `idx_reactions_event (community_id, event_id, event_created_at)` `:551`,
`idx_reactions_pubkey (community_id, pubkey)` `:552`,
UNIQUE `idx_reactions_source_event (community_id, reaction_event_id) WHERE reaction_event_id IS NOT NULL` `:554`.

---

#### 15. `pubkey_allowlist`, `relay_members`, `archived_identities`, `join_policy_acceptances`

`pubkey_allowlist` — `migrations/0001_initial_schema.sql:561-568`:
`community_id` (FK), `pubkey BYTEA`, `added_by BYTEA NULL`,
`added_at NOT NULL DEFAULT NOW()`, `note TEXT`. PK `(community_id, pubkey)` `:567`.

`relay_members` — `migrations/0001_initial_schema.sql:574-582`:
`community_id` (FK), `pubkey TEXT` (lowercase hex),
`role TEXT NOT NULL CHECK (role IN ('owner','admin','member'))` `:577`,
`added_by TEXT NULL`, `created_at`/`updated_at NOT NULL DEFAULT now()`.
PK `(community_id, pubkey)` `:581`; index `idx_relay_members_role (community_id, role)` `:584`.

`archived_identities` — `migrations/0001_initial_schema.sql:589-599`:
`community_id` (FK), `pubkey TEXT`,
`consent_path TEXT NOT NULL CHECK (… IN ('self','owner','admin'))` `:592`,
`actor TEXT NOT NULL`, `reason TEXT`, `replaced_by TEXT`,
`request_event_id TEXT NOT NULL`, `archived_at NOT NULL DEFAULT now()`.
PK `(community_id, pubkey)` `:598`.

`join_policy_acceptances` — `migrations/0020_join_policy_acceptances.sql:4-11`:
`community_id UUID NOT NULL` (no direct FK to `communities`),
`pubkey TEXT NOT NULL`, `policy_version TEXT NOT NULL CHECK (length(policy_version) = 64)`,
`accepted_at NOT NULL DEFAULT now()`. PK `(community_id, pubkey, policy_version)` `:9`;
FK `(community_id, pubkey)` → `relay_members(community_id, pubkey) ON DELETE CASCADE` `:10-11`.

---

#### 16. `audit_log`

`migrations/0001_initial_schema.sql:606-617`. **No Rust module in this crate touches it**
(owned by `buzz-audit`).
`community_id` (FK), `seq BIGINT NOT NULL`, `hash BYTEA NOT NULL`,
`prev_hash BYTEA NULL`, `action VARCHAR(64) NOT NULL`, `actor_pubkey BYTEA NULL`,
`object_id TEXT NULL`, `detail JSONB NULL`, `created_at NOT NULL DEFAULT NOW()`.
PK `(community_id, seq)` `:616`; UNIQUE `idx_audit_log_hash (community_id, hash)` `:619`.

---

#### 17. `_operator_global_tables` — lint allowlist registry

`migrations/0001_initial_schema.sql:628-638`: `table_name TEXT PRIMARY KEY`,
`reason TEXT NOT NULL`. Seeded with `communities`, `rate_limit_violations`,
`_operator_global_tables` `:633-636`; extended by
`migrations/0015_push_gateway_authority.sql:68-74` (six `push_gateway_*` tables)
and `migrations/0017_product_feedback.sql:23-24` (`product_feedback`).
Total registered operator-global tables: **10**.

---

#### 18. `git_repo_names`

`migrations/0002_git_repo_names.sql:20-26`: `community_id` (FK), `repo_id TEXT NOT NULL`,
`owner_pubkey TEXT NOT NULL`, `created_at NOT NULL DEFAULT now()`.
PK `(community_id, repo_id)` `:25`; index `idx_git_repo_names_owner (community_id, owner_pubkey)` `:29`.

---

#### 19. Moderation tables

`moderation_reports` — `migrations/0006_moderation.sql:15-51`

| Column | Type | Null | Default / CHECK |
|--------|------|------|-----------------|
| `community_id` | UUID | NOT NULL | FK → `communities(id)` |
| `id` | UUID | NOT NULL | `gen_random_uuid()` |
| `report_event_id` | BYTEA | NOT NULL | `length = 32` |
| `reporter_pubkey` | BYTEA | NOT NULL | `length = 32` |
| `target_kind` | TEXT | NOT NULL | `IN ('event','pubkey','blob')` |
| `target_event_id` | BYTEA | NULL | NULL or `length = 32` |
| `target_pubkey` | BYTEA | NULL | NULL or `length = 32` |
| `target_blob_sha256` | BYTEA | NULL | NULL or `length = 32` |
| `channel_id` | UUID | NULL | — |
| `report_type` | TEXT | NOT NULL | — |
| `note` | TEXT | NULL | — |
| `status` | TEXT | NOT NULL | `'open'`, `IN ('open','resolved','dismissed','escalated')` |
| `resolved_by` | BYTEA | NULL | — |
| `resolved_at` | TIMESTAMPTZ | NULL | — |
| `action_id` | UUID | NULL | — |
| `created_at` | TIMESTAMPTZ | NOT NULL | `now()` |

- PK `(community_id, id)` `:39`
- Table CHECK enforcing exactly one target class per row — `:42-46`
- FK `(community_id, channel_id)` → `channels(community_id, id)` `:49`
- FK `(community_id, action_id)` → `moderation_actions(community_id, id)` added by
  `ALTER TABLE` at `migrations/0006_moderation.sql:128-130`
- Indexes: `idx_moderation_reports_status (community_id, status, created_at DESC)` `:53`;
  `_target_event (community_id, target_event_id) WHERE target_event_id IS NOT NULL` `:56`;
  `_target_pubkey (community_id, target_pubkey) WHERE target_pubkey IS NOT NULL` `:59`;
  UNIQUE `idx_moderation_reports_event (community_id, report_event_id)` `:63`

`community_bans` — `migrations/0006_moderation.sql:72-87`: `community_id` (FK),
`pubkey BYTEA NOT NULL CHECK length=32`, `banned BOOLEAN NOT NULL DEFAULT false`,
`ban_expires_at TIMESTAMPTZ NULL` (NULL + banned ⇒ permanent), `ban_reason TEXT`,
`muted_until TIMESTAMPTZ NULL`, `mute_reason TEXT`,
`actor_pubkey BYTEA NOT NULL CHECK length=32`, `created_at`/`updated_at DEFAULT now()`.
PK `(community_id, pubkey)` `:86`. No secondary indexes.

`moderation_actions` — `migrations/0006_moderation.sql:94-118`: `community_id` (FK),
`id UUID DEFAULT gen_random_uuid()`, `actor_pubkey` (32B),
`action TEXT NOT NULL CHECK (…12 values…)` `:97-101`,
`target_pubkey`/`target_event_id` (nullable, 32B), `channel_id UUID NULL`,
`reason_code TEXT`, `public_reason TEXT`, `private_reason TEXT`,
`matched_principal TEXT CHECK (NULL OR IN ('self','owner'))` `:113`,
`created_at DEFAULT now()`. PK `(community_id, id)` `:116`;
FK `(community_id, channel_id)` → `channels` `:117`.
Indexes: `_created (community_id, created_at DESC)` `:120`;
`_target_pubkey (community_id, target_pubkey) WHERE target_pubkey IS NOT NULL` `:122`.

---

#### 20. `parameterized_event_watermarks`

`migrations/0007_nip_rs_retention.sql:14-22`: `community_id` (FK), `kind INT NOT NULL`,
`pubkey BYTEA NOT NULL`, `d_tag TEXT NOT NULL`, `created_at TIMESTAMPTZ NOT NULL`,
`event_id BYTEA NOT NULL`. PK `(community_id, kind, pubkey, d_tag)` `:21`.
Compact ordering high-water mark that survives payload purge, so a stale NIP-RS
event can never be resurrected.

---

#### 21. `product_feedback` — operator-global sidecar

`migrations/0017_product_feedback.sql:5-16`

| Column | Type | Null | Default / CHECK |
|--------|------|------|-----------------|
| `id` | UUID | NOT NULL | `gen_random_uuid()`, PK |
| `community_id` | UUID | NOT NULL | FK → `communities(id)` (provenance only) |
| `event_id` | BYTEA | NOT NULL | `length = 32`, `UNIQUE (event_id)` `:15` |
| `submitter_pubkey` | BYTEA | NOT NULL | `length = 32` |
| `category` | TEXT | NULL | `IN ('bug','praise','needs-work')` |
| `body` | TEXT | NOT NULL | `length(btrim(body)) > 0` |
| `tags` | JSONB | NOT NULL | `'[]'::jsonb`, `jsonb_typeof(tags) = 'array'` |
| `event_created_at` | TIMESTAMPTZ | NOT NULL | — |
| `received_at` | TIMESTAMPTZ | NOT NULL | `now()` |

Indexes: `idx_product_feedback_received (received_at DESC, id)` `:18`;
`idx_product_feedback_community_received (community_id, received_at DESC, id)` `:20`.
Note: the PK is `(id)` alone and the UNIQUE is `(event_id)` alone — permitted only
because the table is registered operator-global at `:23-24`.

---

#### 22. Push tables

`push_leases` — `migrations/0012_push_leases.sql:3-22`, plus `endpoint_enabled` from `0013:3-4`

| Column | Type | Null | CHECK |
|--------|------|------|-------|
| `community_id` | UUID | NOT NULL | FK → `communities(id)` |
| `author` | BYTEA | NOT NULL | `length = 32` |
| `installation_id` | TEXT | NOT NULL | `octet_length BETWEEN 1 AND 64` |
| `source_event_id` | BYTEA | NOT NULL | `length = 32` |
| `source_created_at` | BIGINT | NOT NULL | — |
| `generation` | BIGINT | NOT NULL | `> 0` |
| `active` | BOOLEAN | NOT NULL | — |
| `endpoint_enabled` | BOOLEAN | NOT NULL DEFAULT true | `migrations/0013_push_endpoint_state.sql:3` |
| `app_profile` | TEXT | NULL | — |
| `endpoint_hash` | BYTEA | NULL | NULL or `length = 32` |
| `endpoint_grant` | TEXT | NULL | — |
| `max_class` | TEXT | NULL | NULL or `IN ('silent','default','time_sensitive','urgent')` |
| `subscriptions` | JSONB | NULL | — |
| `expires_at` | BIGINT | NOT NULL | — |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | — |

PK `(community_id, author, installation_id)` `:18`;
UNIQUE `(community_id, source_event_id)` `:19`;
table CHECK coupling `active` to all five effective columns being non-NULL (and
all NULL when inactive) `:20-21`;
UNIQUE INDEX `push_leases_endpoint_unique (community_id, author, app_profile, endpoint_hash) WHERE active` `:23`;
index `push_leases_expiry (community_id, expires_at) WHERE active` `:26`.

`push_wake_outbox` — `migrations/0012_push_leases.sql:28-52`: `community_id` (FK),
`id UUID DEFAULT gen_random_uuid()`, `author` (32B), `installation_id TEXT`,
`lease_generation BIGINT CHECK > 0`, `endpoint_hash` (32B), `event_id` (32B),
`class TEXT CHECK IN (4 classes)`, `expires_at BIGINT`,
`state TEXT DEFAULT 'pending' CHECK IN ('pending','sending','delivered','failed')`,
`attempts INTEGER DEFAULT 0 CHECK >= 0`, `next_attempt_at TIMESTAMPTZ DEFAULT now()`,
`lease_until TIMESTAMPTZ NULL`, `claim_id UUID NULL`, `created_at DEFAULT now()`.
PK `(community_id, id)` `:44`; FK `(community_id, author, installation_id)` → `push_leases` `:45-46`;
UNIQUE `(community_id, endpoint_hash, event_id)` `:47` (the wake dedup key).
Indexes `push_wake_outbox_due (community_id, next_attempt_at) WHERE state='pending'` `:49`;
`_recovery (community_id, lease_until) WHERE state='sending'` `:51`.

`push_match_queue` — `migrations/0018_push_match_queue.sql:5-15`: `community_id` (FK),
`event_id BYTEA CHECK length=32`, `state TEXT DEFAULT 'pending' CHECK IN ('pending','matching')`,
`attempts INTEGER DEFAULT 0 CHECK >= 0`, `next_attempt_at DEFAULT now()`,
`lease_until TIMESTAMPTZ NULL`, `claim_id UUID NULL`, `created_at DEFAULT now()`.
PK `(community_id, event_id)` `:14`. Indexes
`push_match_queue_due (next_attempt_at, created_at) WHERE state='pending'` `:16`
and `_recovery (lease_until) WHERE state='matching'` `:18` — both deliberately
**not** community-leading (global due-scan).

`push_gateway_*` (six tables, `migrations/0015_push_gateway_authority.sql:4-66`) —
deployment-global, **no `community_id` column, and no Rust module in buzz-db**:

| Table | PK | Notable constraints / indexes |
|-------|-----|------------------------------|
| `push_gateway_challenges` | `id` | `challenge_hash` 32B; `_expiry (expires_at)` |
| `push_gateway_installations` | `id` | `app_attest_key_id` UNIQUE, 1–128B; `app_attest_public_key` 33–256B; `assertion_counter BETWEEN 0 AND 4294967295`; `app_profile IN ('buzz-ios-production','buzz-ios-sandbox')`; `token_ciphertext` 1–2048B; `token_fingerprint` 32B; `endpoint_epoch > 0`; UNIQUE `(app_profile, token_fingerprint)`; partial `_expiry` |
| `push_gateway_delegations` | `id` | FK → `push_gateway_installations(id)`; `relay_pubkey` 32B; `endpoint_epoch > 0`; `generation > 0`; UNIQUE `(installation_id, relay_pubkey)`; CHECK `not_before < expires_at` |
| `push_gateway_endpoint_quotas` | `token_fingerprint` (32B) | `admitted >= 0`; `_updated (updated_at)` |
| `push_gateway_delivery_auth_replays` | `(relay_pubkey, auth_event_id)` | both 32B; `_expiry` |
| `push_gateway_delivery_request_replays` | `(relay_pubkey, request_id)` | `relay_pubkey` 32B; `_expiry` |

---

#### 23. FK graph (tenant-scoped)

```
communities(id)
 ├── channels(community_id) ............... PK (community_id, id)
 │    ├── channel_members(community_id, channel_id)      ON DELETE CASCADE
 │    ├── thread_metadata(community_id, channel_id)
 │    ├── workflows(community_id, channel_id)
 │    ├── moderation_reports(community_id, channel_id)
 │    └── moderation_actions(community_id, channel_id)
 ├── users(community_id) ................. PK (community_id, pubkey)
 │    ├── users(community_id, agent_owner_pubkey)  self-FK, ON DELETE SET NULL
 │    ├── subscriptions(community_id, owner_pubkey)
 │    ├── workflows(community_id, owner_pubkey)
 │    └── api_tokens(community_id, owner_pubkey)
 ├── relay_members(community_id, pubkey)
 │    └── join_policy_acceptances(community_id, pubkey)  ON DELETE CASCADE
 ├── workflows(community_id, id)
 │    ├── workflow_runs(...)              ON DELETE CASCADE
 │    ├── workflow_approvals(...)         ON DELETE CASCADE
 │    └── scheduled_workflow_fires(...)   ON DELETE CASCADE
 ├── workflow_runs(community_id, id)
 │    ├── workflow_approvals(community_id, run_id)       ON DELETE CASCADE
 │    └── scheduled_workflow_fires(community_id, workflow_run_id)  NO ACTION
 ├── push_leases(community_id, author, installation_id)
 │    └── push_wake_outbox(...)
 ├── moderation_actions(community_id, id)
 │    └── moderation_reports(community_id, action_id)
 ├── events / event_mentions / reactions / thread_metadata /
 │   pubkey_allowlist / archived_identities / audit_log /
 │   git_repo_names / parameterized_event_watermarks /
 │   push_match_queue / delivery_log / product_feedback  →  community_id only
 └── (no FK from any table to `events` — partitioned parent, see
      `migrations/0007_nip_rs_retention.sql:85-87`)
```

`join_policy_acceptances.community_id` has **no** direct FK to `communities`; it
inherits the reference transitively through `relay_members`
(`migrations/0020_join_policy_acceptances.sql:10-11`).

---

#### 24. Rust types mapping to tables

None of these use `#[derive(sqlx::FromRow)]`; every row is decoded by hand with
`row.try_get(...)` (see `buzz-db-conventions.md`).

| Rust type | File:line | Backing table | Fields (type) |
|-----------|-----------|---------------|---------------|
| `CommunityRecord` | `crates/buzz-db/src/lib.rs:244` | `communities` | `id: CommunityId`, `host: String` |
| `EnsuredCommunityRecord` | `crates/buzz-db/src/lib.rs:253` | `communities` | + `created: bool` (from `xmax = 0`) |
| `CreatedCommunityRecord` | `crates/buzz-db/src/lib.rs:264` | `communities` | `id`, `host` |
| `CreateCommunityWithOwnerResult` | `crates/buzz-db/src/lib.rs:272` | — | `Created(..)` \| `HostExists` \| `LimitReached` |
| `OwnedCommunityRecord` | `crates/buzz-db/src/lib.rs:282` | `communities` + `relay_members` | `id`, `host`, `created_at`, `archived_at: Option<..>` |
| `ArchivedCommunityRecord` / `UnarchivedCommunityRecord` | `crates/buzz-db/src/lib.rs:294`, `:305` | `communities` | `id`, `host`, (`archived_at`) |
| `TokenSummary` | `crates/buzz-db/src/lib.rs:314` | `api_tokens` | `id: Uuid`, `name`, `owner_pubkey: Vec<u8>`, `scopes: Vec<String>`, `created_at`, `expires_at` |
| `ApiTokenRecord` | `crates/buzz-db/src/lib.rs:3838` | `api_tokens` | `id`, `token_hash: Vec<u8>`, `owner_pubkey`, `name`, `scopes: Vec<String>`, `channel_ids: Option<Vec<Uuid>>`, `created_at`, `expires_at`, `last_used_at`, `revoked_at` |
| `AllowlistEntry` | `crates/buzz-db/src/lib.rs:3863` | `pubkey_allowlist` | `pubkey`, `added_by`, `added_at`, `note` |
| `DbConfig` / `Db` / `DbPoolStats` | `crates/buzz-db/src/lib.rs:222`, `:167`, `:191` | — | pool config/handles |
| `EventQuery` | `crates/buzz-db/src/event.rs:21` | `events` (+`event_mentions`) | 18 filter fields incl. `community_id`, `kinds`, `authors`, `ids`, `e_tags`, `d_tags`, `before_id`, `channel_ids`, `global_only`, `max_limit` |
| `ReactionEventInsertOutcome` | `crates/buzz-db/src/event.rs:107` | — | `TargetMissing` \| `Duplicate` \| `Inserted{stored_event, was_inserted}` |
| `ThreadMetadataParams<'a>` | `crates/buzz-db/src/event.rs:1000` | `thread_metadata` | insert params |
| `DueReminder` | `crates/buzz-db/src/event.rs:1263` | `events` + `communities` | `community_id`, `host`, `id`, `pubkey`, `created_at`, `kind`, `tags`, `content`, `sig`, `channel_id` |
| `ChannelRecord` | `crates/buzz-db/src/channel.rs:20` | `channels` | 24 fields, 1:1 with the column list (enums surfaced as `String` via `::text`) |
| `MemberRecord` | `crates/buzz-db/src/channel.rs:70` | `channel_members` | `channel_id`, `pubkey`, `role: String`, `joined_at`, `invited_by`, `removed_at` |
| `ChannelUpdate` | `crates/buzz-db/src/channel.rs:1042` | `channels` | `name`, `description`, `visibility`, `ttl_seconds: Option<Option<i32>>` |
| `AccessibleChannel` | `crates/buzz-db/src/channel.rs:781` | `channels` + `channel_members` | `channel`, `is_member: bool` |
| `BotMemberRecord` / `BotChannelEntry` | `crates/buzz-db/src/channel.rs:757`, `:729` | `channel_members`+`users`+`channels` | `pubkey`, `display_name`, `agent_type`, `capabilities: Value`, `channels: Vec<BotChannelEntry>` |
| `UserRecord` | `crates/buzz-db/src/channel.rs:770` | `users` | `pubkey`, `display_name`, `avatar_url`, `nip05_handle` |
| `ReapedEphemeralChannel` | `crates/buzz-db/src/channel.rs:738` | `channels`+`communities` | `community_id`, `host`, `channel_id` |
| `ChannelType`/`ChannelVisibility`/`MemberRole` | re-exported at `crates/buzz-db/src/channel.rs:17` from `buzz_core::channel` | `channel_type`/`channel_visibility`/`member_role` enums | mapped by `as_str()` + `$n::enum` cast, parsed back via `FromStr` |
| `UserProfile` / `UserSearchProfile` | `crates/buzz-db/src/user.rs:9`, `:25` | `users` | pubkey + profile columns |
| `DmRecord` / `DmParticipant` | `crates/buzz-db/src/dm.rs:20`, `:32` | `channels`+`channel_members`+`users` | conversation view |
| `ThreadReply` | `crates/buzz-db/src/thread.rs:19` | `thread_metadata` ⋈ `events` | `event_id`, `parent_event_id`, `root_event_id`, `channel_id`, `pubkey`, `tags`, `content`, `stored_event`, `depth`, `created_at`, `broadcast` |
| `ThreadSummary` | `crates/buzz-db/src/thread.rs:47` | `thread_metadata` | `reply_count: i32`, `descendant_count: i32`, `last_reply_at`, `participants: Vec<Vec<u8>>` |
| `ChannelWindow` / `ChannelWindowRow` | `crates/buzz-db/src/thread.rs:74`, `:62` | `events` ⋈ `thread_metadata` | rows + `has_more` + `next_cursor` |
| `ThreadMetadataRecord` | `crates/buzz-db/src/thread.rs:85` | `thread_metadata` | full row |
| `ReactionGroup` / `ReactionUser` / `ReactionSummary` / `BulkReactionEntry` / `ActiveReactionRecord` | `crates/buzz-db/src/reaction.rs:12`, `:23`, `:44`, `:33`, `:53` | `reactions` | grouped views |
| `WorkflowRecord` | `crates/buzz-db/src/workflow.rs:163` | `workflows` | `id`, `community_id`, `name`, `owner_pubkey`, `channel_id`, `definition: Value`, `definition_hash`, `status: WorkflowStatus`, `enabled`, `created_at`, `updated_at` |
| `WorkflowRunRecord` | `crates/buzz-db/src/workflow.rs:190` | `workflow_runs` | full row, `status: RunStatus` |
| `ApprovalRecord` | `crates/buzz-db/src/workflow.rs:239` | `workflow_approvals` | `token: Vec<u8>` (hash), `workflow_id`, `run_id`, `step_id`, `step_index`, `approver_spec`, `status: ApprovalStatus`, `approver_pubkey`, `note`, `expires_at`, `created_at` |
| `ScheduledWorkflowFireClaim` | `crates/buzz-db/src/workflow.rs:222` | `scheduled_workflow_fires` | `community_id`, `workflow_id`, `scheduled_for`, `claimed_at` |
| `WorkflowStatus` / `RunStatus` / `ApprovalStatus` | `crates/buzz-db/src/workflow.rs:41`, `:76`, `:122` | matching enums | `Display` + `FromStr`, cast via `$n::workflow_status` etc. |
| `NewReport` / `ReportRecord` / `ReportTarget` | `crates/buzz-db/src/moderation.rs:36`, `:58`, `:27` | `moderation_reports` | `ReportTarget` is the Rust encoding of `target_kind` + the three target columns |
| `BanRecord` | `crates/buzz-db/src/moderation.rs:84` | `community_bans` | `banned` is the **computed** `banned AND (ban_expires_at IS NULL OR ban_expires_at > now())` |
| `RestrictionState` | `crates/buzz-db/src/moderation.rs:437` | `community_bans` | `banned: bool`, `muted_until: Option<..>` |
| `NewAction` / `ActionRecord` | `crates/buzz-db/src/moderation.rs:126`, `:158` | `moderation_actions` | full row |
| `MODERATION_ACTION_CHECK_VOCAB` | `crates/buzz-db/src/moderation.rs:106` | `moderation_actions.action` CHECK | 12-value mirror of the SQL CHECK |
| `AdminReport` / `AdminFeedback` | `crates/buzz-db/src/admin_moderation.rs:22`, `:63` | `moderation_reports`/`product_feedback` ⋈ `communities` | hex-encoded, `serde(rename_all="camelCase")` |
| `NewProductFeedback` / `ProductFeedbackRecord` | `crates/buzz-db/src/product_feedback.rs:16`, `:33` | `product_feedback` | hex-encoded ids on read |
| `RelayMember` | `crates/buzz-db/src/relay_members.rs:17` | `relay_members` | `pubkey: String`, `role: String`, `added_by: Option<String>`, `created_at`, `updated_at` |
| `RemoveResult` / `TransferResult` | `crates/buzz-db/src/relay_members.rs:172`, `:335` | — | `Removed`\|`IsOwner`\|`NotFound`\|`RoleMismatch`; `Transferred{previous_owner}`\|`AlreadyOwner`\|`NoOwner`\|`OwnerConflict`\|`LimitReached` |
| `ArchivedIdentity` | `crates/buzz-db/src/archived_identities.rs:12` | `archived_identities` | full row, all hex strings |
| `ReserveOutcome` | `crates/buzz-db/src/git_repo.rs:28` | `git_repo_names` | `Reserved`\|`AlreadyOwned`\|`TakenByOther` |
| `LeaseVersion` / `ActiveLease` / `MatchLease` | `crates/buzz-db/src/push.rs:71`, `:85`, `:176` | `push_leases` | ordering + effective fields |
| `ReplaceLeaseOutcome` / `AcceptLeaseOutcome` | `crates/buzz-db/src/push.rs:100`, `:191` | — | `Accepted`\|`StaleEvent`\|`StaleGeneration` (+ `EndpointAlreadyLeased`, `LeaseQuotaExceeded`, `SourceEventCollision`, `ConstraintViolation`) |
| `NewWake` / `WakeRequest` / `ClaimedWake` / `EnqueueWakeOutcome` / `RevalidateWakeOutcome` | `crates/buzz-db/src/push.rs:121`, `:490`, `:135`, `:110`, `:166` | `push_wake_outbox` ⋈ `push_leases` ⋈ `events` | outbox job views |
| `ClaimedMatchBatch` / `BatchedMatch` | `crates/buzz-db/src/push.rs:717`, `:727` | `push_match_queue` ⋈ `events` | claimed batch |
| `CommunityUserCounts`, `CommunityChannelCount`, `CommunityMessageCount`, `CommunityMemberCount`, `CommunityWorkflowCount`, `CommunityGitRepoCount`, `CommunityActiveUsers`, `CommunityActiveChannels`, `CommunityHost` | `crates/buzz-db/src/usage.rs:28`, `:69`, `:104`, `:137`, `:170`, `:203`, `:239`, `:305`, `:328` | rollups over `users`/`channels`/`events`/`relay_members`/`workflows`/`git_repo_names`/`communities` | Prometheus gauge inputs |
| `ReplicaFence` | `crates/buzz-db/src/replica_fence.rs:69` | — | in-memory `AtomicI64` fence, not a table |
| `DbError` | `crates/buzz-db/src/error.rs:7` | — | 11 variants (see conventions) |

Tables with **no** Rust representation in this crate: `subscriptions`,
`delivery_log`, `audit_log`, `rate_limit_violations`, `_operator_global_tables`,
and all six `push_gateway_*` tables.
