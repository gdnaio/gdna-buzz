## Module: buzz-db (`crates/buzz-db`)

### Features

Capabilities the crate provides, with completeness read from the code (not from
docs). "Complete" means the module has both read and write paths and callers in
the workspace surface; "read-only", "partial", and "schema-only" are noted.

| # | Capability | State | Evidence |
|---|-----------|-------|----------|
| 1 | **Nostr event store** — insert (dedup), scoped query with 14 pushdown filters, COUNT, scoped id lookup, batch id lookup, soft delete, coordinate delete | complete | `crates/buzz-db/src/event.rs:240`, `:302`, `:557`, `:868`, `:948`, `:703`, `:730` |
| 2 | **Monthly range partitioning** of `events` and `delivery_log` with startup/cron partition creation | complete for creation; **no drop/detach/rotation-out path exists** | `crates/buzz-db/src/partition.rs:15-73`; partitions declared `migrations/0001_initial_schema.sql:237-252`, `:343-354` |
| 3 | **Replaceable-event replacement** — NIP-16 addressable (`kind, pubkey, channel_id`) and NIP-33 parameterized (`kind, pubkey, d_tag`), each under an advisory lock with deterministic tie-breaks | complete | `crates/buzz-db/src/lib.rs:3306`, `:3628` |
| 4 | **NIP-RS read-state retention** — exact-cardinality classification, payload hard delete, durable ordering watermark, mixed-version DB triggers, hard-delete opt-in fence | complete | `crates/buzz-db/src/lib.rs:3672-3796`; `migrations/0007`, `0009`, `0010`, `0011` |
| 5 | **Mesh-status heartbeat retention** (kind 30003 `buzz-mesh-member-status:*`) — one physical row per coordinate | complete | `crates/buzz-db/src/lib.rs:3688-3695`; `migrations/0019_mesh_status_retention.sql` |
| 6 | **Channel CRUD + membership** with transactional role enforcement, soft delete, last-owner protection, archive/unarchive, topic/purpose, canvas | complete | `crates/buzz-db/src/channel.rs` (26 public fns) |
| 7 | **Ephemeral channels with TTL** — deadline init, in-commit refresh, advisory-lock-ordered TTL transitions, idempotent reaper | complete | `crates/buzz-db/src/channel.rs:1387`; `migrations/0022`, `0024` |
| 8 | **Threading** — parent/root/depth metadata, materialized `reply_count`/`descendant_count`/`last_reply_at`, keyset-paginated replies, channel window with server `has_more`, batched participants | complete | `crates/buzz-db/src/thread.rs` |
| 9 | **Reactions** — TOCTOU-free upsert with new/reactivate/duplicate semantics, soft delete, grouped and bulk reads, single-transaction kind:7 coupling | complete | `crates/buzz-db/src/reaction.rs`; `crates/buzz-db/src/event.rs:1201` |
| 10 | **DMs** — participant-hash identity, 2–9 participants, idempotent open, per-user hide/unhide, hidden-DM listing | complete | `crates/buzz-db/src/dm.rs` |
| 11 | **Home feed** — mentions / needs-action / activity over the normalized `event_mentions` index with a hard 100-row cap | complete | `crates/buzz-db/src/feed.rs` |
| 12 | **Mention index** (`event_mentions`) — multi-row upsert built from validated `p` tags, best-effort (failures logged, never block the event) | complete | `crates/buzz-db/src/lib.rs:97-165`, call sites `:1085-1090`, `:1391-1396`, `:1419-1428` |
| 13 | **User profiles** — upsert, profile update with empty→NULL semantics, NIP-05 lookup, LIKE-escaped search with ranking, agent ownership, channel-add policy | complete | `crates/buzz-db/src/user.rs` |
| 14 | **API tokens** — hashed storage, atomic 10-token quota, community-scoped lookup (incl. revoked), listing, single/bulk revoke, `last_used_at` touch | complete | `crates/buzz-db/src/api_token.rs`, `crates/buzz-db/src/lib.rs:2327-2461` |
| 15 | **Workflows / runs / approvals** — owner-and-channel-guarded upsert, bounded lists, run lifecycle with `started_at`/`completed_at` stamping, SHA-256 approval tokens, single-decision approval updates | complete; 5 functions are documented as having **no current callers** (`create_workflow`, `update_workflow`, `update_workflow_status`, `set_workflow_enabled`, `delete_workflow`) | `crates/buzz-db/src/workflow.rs:275`, `:619`, `:653`, `:683`, `:714` |
| 16 | **Cron fire claims** — at-most-once `(community, workflow, scheduled_for)` claim, DB-authoritative interval anchor, run-id attach, retention prune | complete | `crates/buzz-db/src/workflow.rs:496-611` |
| 17 | **NIP-ER reminders** — due-set query with per-tenant host provenance, exactly-once claim, stamped release | complete | `crates/buzz-db/src/event.rs:1293-1431` |
| 18 | **Relay membership (NIP-43)** — CRUD, invite claim with policy-acceptance evidence, owner-protected removal/role change, owner bootstrap, atomic ownership transfer with a 3-community cap, allowlist backfill, locked snapshot publication | complete | `crates/buzz-db/src/relay_members.rs`; snapshot at `crates/buzz-db/src/lib.rs:3438-3626` |
| 19 | **Pubkey allowlist** — membership check, enforcement-active check, add/remove/list | complete (inline on `Db`) | `crates/buzz-db/src/lib.rs:2826-2905` |
| 20 | **Archived identities (NIP-IA)** — community-scoped archive/unarchive/list, idempotent | complete | `crates/buzz-db/src/archived_identities.rs` |
| 21 | **Community moderation** — NIP-56 report ingest/queue/resolve, ban + timeout state with expiry-aware reads, audit action log | complete | `crates/buzz-db/src/moderation.rs` |
| 22 | **Deployment-admin moderation plane** — keyset-paginated global report list, global feedback list, single-row fetches | read-only by construction | `crates/buzz-db/src/admin_moderation.rs` |
| 23 | **Product feedback sidecar** — deployment-wide idempotent insert by event id, newest-first list | complete | `crates/buzz-db/src/product_feedback.rs` |
| 24 | **Git repo-name registry (NIP-34)** — atomic per-community reservation, owner classification, quota count, owner-scoped release | complete | `crates/buzz-db/src/git_repo.rs` |
| 25 | **NIP-PL push leases + wake outbox + match queue** — signed-lease acceptance with ordering/quota/endpoint rules, gate-guarded matcher enqueue, batched wake enqueue, fenced claim/complete/retry/fail, send-time revalidation, generation-scoped endpoint disable, pruning | complete for the relay-owned side | `crates/buzz-db/src/push.rs` |
| 26 | **Read-replica routing with a freshness fence** — commit-time floor guard, ordered LSN handshake, staleness expiry, two-part startup verification, per-query routing decisions | complete | `crates/buzz-db/src/replica_fence.rs`; `crates/buzz-db/src/lib.rs:2004`, `:2063` |
| 27 | **Per-community usage rollups** for Prometheus gauges (11 queries) plus a detached-session advisory leader lock | complete | `crates/buzz-db/src/usage.rs`; `crates/buzz-db/src/lib.rs:517-535` |
| 28 | **Embedded migrations + tenant-isolation lint harness** — 24 migrations, pre-flight ambiguity guard, post-migration floor-guard verification, SQL-parsing lint tests | complete | `crates/buzz-db/src/migration.rs` |
| 29 | **Pool observability** — `ping`, writer/replica pool stats | complete | `crates/buzz-db/src/lib.rs:485-511` |
| 30 | **Community lifecycle** — host resolve (active / management), icon get/set, ensure-configured, atomic create-with-owner, archive/unarchive, channel→community reverse resolution (single + batch) | complete | `crates/buzz-db/src/lib.rs:656-1077` |
| 31 | **d_tag backfill** maintenance for legacy NIP-33 rows | complete, idempotent, **unscoped by community** | `crates/buzz-db/src/lib.rs:2810-2824` |
| 32 | **Subscriptions / delivery_log / audit_log / rate_limit_violations** | **schema-only in this crate** — tables exist, no Rust read/write path | `migrations/0001_initial_schema.sql:304`, `:329`, `:498`, `:606`; no matching module (verified by search) |
| 33 | **push_gateway_* authority tables** (6) | **schema-only in this crate** — created by `migrations/0015`, referenced only by the lint allowlist in tests | `migrations/0015_push_gateway_authority.sql`; `crates/buzz-db/src/migration.rs:343-348` |

---

#### TODO / FIXME / HACK / XXX inventory

**Zero occurrences.** A case-insensitive search for `TODO`, `FIXME`, `HACK`, and
`XXX` across `crates/buzz-db/**/*.rs`, all 24 files in `migrations/`, and
`schema/schema.sql` returns no matches. Deferred work is instead expressed as
prose in doc comments; the concrete instances are:

| Marker text (verbatim) | File:line |
|------------------------|-----------|
| `/// creation path is [`upsert_workflow`] via event ingest. (No current callers.)` | `crates/buzz-db/src/workflow.rs:275` |
| `/// matching lags the change by up to the cache TTL. (No current callers.)` | `crates/buzz-db/src/workflow.rs:619` |
| `/// [`update_workflow`]. (No current callers.)` | `crates/buzz-db/src/workflow.rs:653` |
| `/// on [`update_workflow`]. (No current callers.)` | `crates/buzz-db/src/workflow.rs:683` |
| `/// `channel_id` needed for invalidation. (No current callers.)` | `crates/buzz-db/src/workflow.rs:714` |
| `/// `cursor` is reserved for future keyset pagination (currently unused).` | `crates/buzz-db/src/reaction.rs:279` (parameter `_cursor` at `:286`) |
| `/// NOTE: The primary increment path is inlined inside [`insert_thread_metadata`]'s` … `/// This standalone version exists for future use cases` (function carries `#[allow(dead_code)]`) | `crates/buzz-db/src/thread.rs:243-250` |
| `// Run one query per event. For typical message-list sizes (<=100 events)` … `// a single-query approach with dynamic IN clauses over` / `// composite keys can be added later if needed.` | `crates/buzz-db/src/reaction.rs:373-375` |
| `//  If query performance` / `// degrades, add a composite index like `(pubkey, kind, created_at DESC, id ASC)`.` | `crates/buzz-db/src/event.rs:437-441` |
| `//! At scale these can become recurring partition scans; if that becomes a` / `//! problem, move them to a maintained rollup table and drop the interval.` | `crates/buzz-db/src/usage.rs:8-10` |
| `/// For large channel sets, consider pagination.` | `crates/buzz-db/src/channel.rs:602` |
| `-- the search lane can` / `-- revisit the config behind evidence.` (FTS `'simple'` config) | `migrations/0001_initial_schema.sql:200-201` |
