## Module: buzz-relay — event ingest & side effects (`crates/buzz-relay/src/handlers`)
### Aspect: Integrations

---

### 1. Crate dependency fan-out

| Crate | Reached from this group | How |
|---|---|---|
| `buzz-core` | all four files | kind constants + classification predicates (`ingest.rs:11-45`), `verify_event` (`:1465`), `TenantContext`, `CommunityId`, `StoredEvent`, `canonical_channel_name` (`ingest.rs:2043`, `side_effects.rs:466`) |
| `buzz-auth` | `ingest.rs:8` | `Scope` only — used by `required_scope_for_kind` (`:198-306`) and the (unreachable) gate at `:1525` |
| `buzz-db` | all except `imeta.rs` | 60+ distinct methods via `state.db` — see §2 |
| `buzz-media` | `imeta.rs` | `MediaStorage::get_sidecar` (`imeta.rs:246`, `:308`), `MediaStorage::head` (`:252`, `:290`, `:328`), `validation::mime_to_ext` (`imeta.rs:5`, used `:185`) |
| `buzz-pubsub` | `side_effects.rs` | `EventTopic` (`:26`); `publish_event` (`:701`, `:794`, `:873`); `release_topic` (`:81`) |
| `buzz-workflow` | `command_executor.rs` | `WorkflowEngine::parse_yaml` (`:719`), `TriggerDef::Webhook` match (`:738`), `WorkflowDef` deserialize (`:903`, `:1272`), `executor::execute_from_step` (`:925`, `:1317`), `engine.finalize_run` (`:934`, `:1325`), `engine.invalidate_channel_workflows` (`:783`); `side_effects.rs:2017`, `:2038` for the a-tag delete path |
| `buzz-conformance` | `ingest.rs` | `Tracer` trait (`:1381`), `TraceAction`, `EmitGuard`, `Verdict` via `crate::conformance` (`:47-50`) |
| `buzz-audit` | **not directly** | reached transitively through `dispatch_persistent_event` → `enqueue_event_created_audit` (`handlers/event.rs:336`, `:534-577`) |
| `buzz-search` | **never** | Postgres FTS is a generated `tsvector` column populated by the `insert_event` write itself; the old Typesense worker and `search_index_tx` are gone (`handlers/event.rs:478-484`). This module makes **zero** `buzz_search` calls. |
| `sqlx` | `command_executor.rs` | raw SQL — `pg_advisory_xact_lock` (`:170`), coordinate SELECT (`:176`), coordinate soft-delete (`:196`), `events` INSERT (`:196-232`). The only place in this group that bypasses the `buzz-db` API. |
| `metrics` | `ingest.rs`, `side_effects.rs`, `command_executor.rs` | 8 series — see configuration.md §4 |
| `nostr` | all | `Event`, `EventBuilder`, `Kind`, `Tag`, `Keys` |

`buzz-relay` internal reach-out: `crate::api::media::media_base_url_for_tenant`
(`ingest.rs:2212`) and `is_safe_ext` (`imeta.rs:379`); `crate::api::nip05::canonicalize_nip05`
(`side_effects.rs:1157`); `crate::api::git::{manifest, store, manifest_event}`
(`side_effects.rs:2616-2618`, `:2705`, `:2738`); `crate::protocol::RelayMessage`
(`side_effects.rs:23`); `crate::webhook_secret` (`command_executor.rs:27`);
`crate::handlers::{event, relay_admin, identity_archive, report, product_feedback,
moderation_commands, push_lease}`.

---

### 2. `buzz-db` call inventory (by concern)

| Concern | Methods called | Sites |
|---|---|---|
| Event read | `get_event_by_id`, `get_event_by_id_including_deleted`, `query_events` | `ingest.rs:361`, `:634`, `:665`, `:690`, `:790`, `:869`, `:2245`; `side_effects.rs:203`, `:552`, `:1600`, `:2114`, `:2201`, `:931`, `:2969`, `:3080` |
| Event write | `insert_event`, `insert_event_with_thread_metadata`, `insert_reaction_event_with_thread_metadata`, `replace_addressable_event`, `replace_parameterized_event` | `ingest.rs:2298`, `:2371`, `:2385`, `:2394`; `side_effects.rs:692`, `:868`, `:940`, `:2752`, `:2911`, `:3036`, `:3123`, `:3186` |
| Event delete | `soft_delete_event_and_update_thread`, `soft_delete_by_coordinate`, `soft_delete_discovery_events` | `side_effects.rs:1624`, `:2147`, `:2069`, `:1798` |
| Thread | `get_thread_metadata_by_event`, `get_thread_summary` | `ingest.rs:608`; `side_effects.rs:1616`, `:2131`, `:735` |
| Reaction | `remove_reaction_by_source_event_id`, `remove_reaction` | `side_effects.rs:2175`, `:2216` |
| Channel | `get_channel`, `create_channel`, `create_channel_with_id`, `update_channel`, `set_topic`, `set_purpose`, `archive_channel`, `unarchive_channel`, `soft_delete_channel`, `list_channels`, `open_dm`, `hide_dm`, `list_hidden_dms` | `ingest.rs:1739`, `:812`, `:2103`, `:2408`; `side_effects.rs:271`, `:545`, `:1345`, `:1372`, `:1387`, `:1416`, `:1466`, `:1485`, `:1499`, `:1789`, `:1846`, `:2952`, `:3062`; `command_executor.rs:398`, `:497`, `:534`, `:611`, `:633`, `:766` |
| Membership | `get_members`, `add_member`, `remove_member`, `is_agent_owner`, `get_agent_channel_policy` | `side_effects.rs:100`, `:298`, `:376`, `:391`, `:520`(via members), `:627`, `:647`, `:1216`, `:1275`, `:1293`, `:1526`, `:1932`; `ingest.rs:829`, `:1339`, `:2002`; `command_executor.rs:518` |
| User | `ensure_user`, `update_user_profile`, `set_channel_add_policy` | `side_effects.rs:1096`, `:1140`, `:1163`, `:1182`, `:1105`; `command_executor.rs:49` |
| Relay membership | `remove_relay_member`, `publish_nip43_membership_locked`, `nip43_membership_snapshot_needs_reconciliation`, `usage_community_hosts` | `ingest.rs:1857`; `side_effects.rs:2866`, `:2789`, `:2777` |
| Archived identities | `list_archived` | `side_effects.rs:3009` |
| Moderation | `moderation_restriction_state` | `ingest.rs:1616` |
| Workflow | `get_workflow`, `upsert_workflow`, `create_workflow_run`, `get_workflow_run`, `update_workflow_run`, `delete_workflow_for_owner`, `find_workflow_by_owner_and_name`, `get_approval_by_stored_hash`, `update_approval_by_stored_hash` | `command_executor.rs:724`, `:775`, `:918`, `:1250`, `:1177`, `:1276`, `:1290`, `:1305`, `:1041`, `:1153`; `side_effects.rs:2000`, `:2011`, `:2026` |
| Git | `repo_name_owner`, `count_repos_for_owner`, `reserve_repo_name`, `release_repo_name` | `side_effects.rs:2470`, `:2491`, `:2500`, `:2543` |
| Transaction | `begin_transaction` | `command_executor.rs:106` |

---

### 3. Transaction boundaries and partial-failure semantics

**The core answer: ingest is *not* atomic beyond the single storage call.**

#### 3a. What *is* atomic

| Unit | Contents | Where |
|---|---|---|
| Generic event insert | `events` row + `thread_metadata` row + parent/root stubs + `reply_count`/`last_reply_at`/`descendant_count` updates | `buzz-db/src/event.rs:1004-1191`; the rationale ("could leave reply counters inconsistent if one succeeded and the other failed") is at `:1173-1177` |
| Reaction insert | `reactions` upsert + `events` row + `thread_metadata`, in a load-bearing order so an active duplicate returns before a duplicate kind:7 is stored | `buzz-db/src/event.rs:1201-1275`; ordering note `ingest.rs:2294-2297` |
| Soft delete | `events.deleted_at` + `reply_count`/`descendant_count` decrements | `soft_delete_event_and_update_thread`, called `side_effects.rs:1624`, `:2147` |
| Replaceable / NIP-33 replace | old-row supersede + new insert | inside `replace_addressable_event` / `replace_parameterized_event` |
| Relay-member removal | NotFound / IsOwner classification + delete | `remove_relay_member` (`ingest.rs:1857`) — the comment at `:1855` states it "handles both the NotFound and IsOwner cases atomically" |
| Git name reservation | `INSERT … ON CONFLICT DO NOTHING` as the TOCTOU race guard | `side_effects.rs:2500`, rationale `:2437-2447` |
| NIP-43 snapshot publish | read members + build event + replace, all inside one per-community advisory lock so a stale snapshot cannot win by arrival order | `publish_nip43_membership_locked` (`side_effects.rs:2866`), rationale `:2814-2824` |
| Command coordinate LWW | `pg_advisory_xact_lock` + head read + supersede + insert | `command_executor.rs:170-232` |

#### 3b. What is **not** atomic — the failure matrix

| Failure point | Event state | Domain state | Client sees | Evidence |
|---|---|---|---|---|
| `handle_side_effects` returns `Err` | **committed and fanned out** | not applied | `accepted: true, message: ""` | `ingest.rs:2434-2441` — `warn!` only, then `dispatch_persistent_event` at `:2489` runs unconditionally |
| 9000 `add_member` rejected by the DB's elevated-role guard (`buzz-db/src/channel.rs:385`, `:400`) | kind:9000 committed + fanned out | no membership change | success | same |
| 9002 `update_channel` fails mid-loop | kind:9002 committed; **earlier tags in the same event already applied** | partial metadata update | success | `side_effects.rs:1339-1552` — the per-tag loop uses `?`, so tag *n* failing leaves tags 1..n−1 applied |
| 9007 event insert fails after `create_channel_with_id` | not stored | channel soft-deleted by compensation | `Internal` | `ingest.rs:2404-2414` — manual compensation, itself `warn!`-only on failure |
| 30617 pointer seed fails | kind:30617 committed | name reservation released **only if this attempt created it** | success | `side_effects.rs:2528-2555` |
| 30617 kind:30618 emission fails | committed | pointer exists, subscribers miss the "repo exists" signal | success | `side_effects.rs:2588-2601` — explicitly "Non-fatal" |
| `emit_live_thread_summary` fails | committed, counters correct | live badge counts stale until the next page fetch | success | `side_effects.rs:724-815`, spawned task, `warn!` on every failure branch |
| `emit_system_message` insert fails | committed | no kind:40099 tombstone / notice | success (side effect returns `Ok`) | `side_effects.rs:690-697` — insert failure is `warn!`-ed and the function still returns `Ok(())` |
| `emit_membership_notification` fails | committed | agent never learns to resubscribe | success | `side_effects.rs:1248-1256`, `:1319-1327`, `:1766-1774` |
| `emit_group_discovery_events` fails | committed | 39000/39001/39002 stale | success | `side_effects.rs:1244`, `:1315`, `:1553`, `:1762`, `:1875` |
| Redis `publish_event` fails | committed | `local_event_ids` echo-dedupe entry **rolled back** so a later Redis retry is not swallowed | success | `side_effects.rs:790-800`, `:869-879`; same pattern in `handlers/event.rs:390` |
| Audit enqueue channel closed | committed | audit entry lost | success | `handlers/event.rs:574-577` — `error!` + `buzz_audit_send_errors_total` |
| Command mutation succeeds, `tx.commit()` fails | **not** stored | mutation persisted | `Internal("error: commit transaction: …")` | `command_executor.rs:92-98` documents this exact window; safety rests on `open_dm` / `hide_dm` / `update_approval` / `upsert_workflow` being idempotent so a retry converges |
| 28936 NIP-43 publishes fail | member already removed | 8001/13534 not published | `accepted: true, "info: you have left this relay"` | `ingest.rs:1883-1896` |

#### 3c. Fail-closed vs fail-open

Fail-**closed** (error propagates, write refused):
- restriction-state lookup (`ingest.rs:1633-1641`, comment: "a DB error must not let a
  banned/timed-out actor write");
- 9021 membership check (`side_effects.rs:1861-1866`, "Fail closed on DB errors rather
  than falling through to add_member");
- 9005 target lookup (`side_effects.rs:552-558`, "Fail closed: missing target → reject");
- 9002 `ttl` parse in the *side effect* (`side_effects.rs:1454-1464`, "a parse failure must
  reject, never silently clear the TTL to permanent");
- 30617 pointer establishment (`side_effects.rs:2528-2559`).

Fail-**open** (error swallowed):
- every side effect after storage (BR-IN-69);
- `insert_mentions` (`buzz-db/src/lib.rs:1394-1399`);
- `ensure_user` in the command executor (`command_executor.rs:60-63` — `warn!` and
  continue, even though the comment at `:44-46` says it exists to satisfy a foreign-key
  requirement; a subsequent FK violation would then surface as `Internal`);
- kind:0 NIP-05 UNIQUE collision, which retries the profile write **without** the handle
  rather than failing (`side_effects.rs:1174-1195`);
- kind:0 off-domain NIP-05, silently cleared because "the event is already persisted and
  can't be rejected" (`side_effects.rs:1150-1153`).

---

### 4. Fan-out path (via `handlers/event.rs`)

`dispatch_persistent_event` (`handlers/event.rs:326-367`) is called from
`ingest.rs:2351` (reaction), `:2489` (generic), and from `side_effects.rs:940`, `:2872`,
`:3045`, `:3132`, `:3195`. It:
1. awaits `enqueue_event_created_audit` (bounded mpsc, capacity 1000 — deliberate
   backpressure, `handlers/event.rs:540-548`);
2. spawns a task that marks the event local, publishes to Redis, then fans out to local
   subscribers, then spawns workflow evaluation.

Consequence: **NIP-01 `OK` is returned before Redis publish, local fan-out, or workflow
triggering complete** — stated explicitly at `handlers/event.rs:320-325`.

Workflow triggering is skipped for `is_workflow_execution_kind` (46001–46012),
`is_command_kind` (7 kinds), relay-signed events tagged `buzz:workflow`, and kind 1059
(`handlers/event.rs:497-502`). The workflow lookup is community-scoped, with the rationale
that a colliding channel UUID in another community must not trigger this community's
workflows (`handlers/event.rs:511-517`).

Three emitters deliberately bypass `dispatch_persistent_event` and call
`fan_out_event_to_local_subscribers` directly, skipping audit and workflow evaluation:
`emit_membership_notification` (`side_effects.rs:882-885`), `publish_nip43_delta`
(`:2905-2907`), `emit_initial_ref_state` (`:2755-2762`). The first documents this as
"Fan-out only — skip search indexing and workflow evaluation" (`side_effects.rs:855`).
`emit_live_thread_summary` does the same for a never-stored event (`:801-808`).

---

### 5. Subscription lifecycle coupling

`side_effects.rs` reaches directly into the connection and subscription registries — the
only non-transport module that does:

| Function | Effect | Trigger |
|---|---|---|
| `evict_live_channel_subscriptions` `:39` | closes a specific pubkey's channel subs across all their connections | 9001 (`:1295`), 9022 (`:1934`) |
| `evict_conn_channel_subscriptions` `:56` | removes from `sub_registry`, removes from the conn's local map, `release_topic`, sends `CLOSED restricted: channel access revoked` | the three above |
| `evict_non_member_channel_subscriptions` `:95` | closes subs for connections whose pubkey is not a current member | 9002 open→private (`:1437`) |
| `evict_all_channel_subscriptions` `:128` | closes every sub on a channel | ephemeral-channel reaper (`main.rs:672`) |

The reason string `channel access revoked` is chosen because it is in the client's
drop-set, so a connected agent drops one channel without reconnecting
(`side_effects.rs:118-125`).

---

### 6. Media integration (`imeta.rs`)

Read-only against `buzz_media::MediaStorage`: 3 `get_sidecar` reads and 3 `head` calls per
imeta tag worst case (`imeta.rs:246`, `:252`, `:290`, `:308`, `:328`). No writes, no
deletes, no retention interaction. Uploads happen out-of-band via Blossom; ingest only
proves the blob already exists and that the claimed metadata matches the sidecar.

Trust boundary: the sidecar is authoritative for `ext`, `mime_type`, `size`, and
`duration_secs`; the event's claims are checked *against* it, never the reverse
(`imeta.rs:259-278`). The upload validator's deny-list is therefore the real content gate,
and `validate_imeta_tags` only needs a structural MIME check — reasoned out at
`imeta.rs:71-76`.

Per-tenant media base URL: `media_base_url_for_tenant(&state.config.relay_url,
tenant.host())` (`ingest.rs:2211-2212`), so a tenant-host URL is accepted only when it
matches that tenant's base (tested at `imeta.rs:438-449`).

---

### 7. Git object store integration

`side_effects.rs` is the only handler that touches `state.git_store`:
`put_manifest` (`:2642`), `put_pointer(Precond::IfNoneMatchStar)` (`:2652`),
`get_pointer` (`:2664`, `:2714`). CAS semantics: `CasOutcome::Won` → success;
`CasOutcome::LostRace` → success **only if** the existing pointer names the same empty
manifest digest, otherwise a hard error rather than silently accepting a stale pointer
(`:2670-2691`). `ensure_manifest_pointer` (`:2704-2731`) is the tolerant re-announce
variant: any existing pointer is left untouched, an absent one is repaired.

The invariant maintained is "repo announced ⟺ pointer exists", so the read path can treat
pointer-absent as never-announced and keep `info_refs`'s fail-closed 404 unambiguous
(`side_effects.rs:2557-2571`).

---

### 8. Conformance-trace integration

`ingest_event` (`ingest.rs:1367`) wraps the pipeline in `EmitGuard::arm`
(`:1381-1386`), passes the counting tracer down, then maps terminal errors to a single
`SanitizedError` action (`:1409-1417`). Emitted actions: `AuthCheck` (`:1791-1799`),
`WriteInsert` / `WriteDuplicate` (`:2327-2350`, `:2467-2486`), `WriteInsertGlobal`
(`:2186-2193`, `:2481`, plus `emit_product_feedback_success` `:133-154`).
Production tracer is `NoopTracer` (`state.rs:798`), so this is inert outside conformance
tests. Spec reference: `docs/spec/MultiTenantRelay.tla`, cited at `ingest.rs:1364`.
