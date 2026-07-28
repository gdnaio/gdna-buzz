## Module: buzz-relay — event ingest & side effects (`crates/buzz-relay/src/handlers`)
### Aspect: Features

Every product capability that writes state passes through `ingest_event`
(`ingest.rs:1367`). This aspect maps features to the branches that realize them, then
lists the mismatches: handled kinds with no reachable feature, and declared kinds ingest
refuses.

---

### 1. Feature → ingest branch map

| Feature | Kinds | Ingest branch | Side effects |
|---|---|---|---|
| **Channel messaging** | 9, 40002 | generic store (`ingest.rs:2391-2401`) + NIP-10 thread resolution (`:2220-2231`) | none; kind:39005 summary emitted for replies (`:2448-2455`) |
| **Message editing** | 40003 | `validate_edit_ownership` (`ingest.rs:1960-1964`) → generic store | none |
| **Message deletion (self / owner)** | 5 | `validate_standard_deletion_event` (`ingest.rs:1921-1925`) → store → `handle_standard_deletion_event` (`side_effects.rs:2108`) | soft-delete + counter decrement + reaction-row removal + kind:39005 |
| **Moderated deletion (channel admin)** | 9005 | `validate_admin_event` 9005 arm (`side_effects.rs:508`) → store → `handle_delete_event_side_effect` (`side_effects.rs:1560`) | soft-delete, counter decrement, kind:39005, kind:40099 tombstone carrying `action_id`/`reason_code`/`public_reason` |
| **Reactions** | 7 | inline transactional path (`ingest.rs:2245-2365`) | none (dedup + `reactions` row happen pre-storage) |
| **Threads (NIP-10) + live badge counts** | any `requires_h` kind with `e root`/`e reply` | `resolve_nip10_thread_meta` (`ingest.rs:564-717`) | `emit_live_thread_summary` → kind:39005 (`side_effects.rs:724`) |
| **Message pinning / bookmarking / scheduling / reminders** | 40004, 40005, 40006, 40007 | generic store, `h` required — **no validator, no side effect** | none |
| **Code-diff messages** | 40008 | `validate_diff_event` (`ingest.rs:896-963`) → generic store | none |
| **Canvas** | 40100 | generic store, `h` required | none |
| **Forums** | 45001, 45002, 45003 | generic store; 45002 gated by `validate_forum_vote_target` (`ingest.rs:844`) | none |
| **Channel lifecycle (create)** | 9007 | eager `create_channel_with_id` (`ingest.rs:2103`) → store → `handle_create_group` (`side_effects.rs:1660`) | kind:40099 `channel_created`, 39000/39001/39002 discovery, kind:44100 to the creator, `buzz_channels_created_total` metric |
| **Channel metadata (name/about/topic/purpose/visibility/ttl/archive)** | 9002 | `validate_admin_event` 9002 arm (`side_effects.rs:410`) → store → `handle_edit_metadata` (`side_effects.rs:1335`) | per-tag DB update, kind:40099 per change type, cache invalidation, subscription eviction on open→private, kind:44100 resubscribe fan-out on unarchive, discovery re-emit |
| **Channel deletion** | 9008 | 9008 arm (`side_effects.rs:625`) → store → `handle_delete_group` (`side_effects.rs:1783`) | soft-delete channel, soft-delete discovery events, cache invalidation, kind:40099 |
| **Channel membership (invite / remove)** | 9000, 9001 | 9000/9001 arms → store → `handle_put_user` / `handle_remove_user` | `channel_members` write, cache invalidation, subscription eviction (remove only), kind:40099, discovery re-emit, kind:44100/44101 |
| **Self-join / self-leave** | 9021, 9022 | `ingest.rs:2134-2154` / 9022 arm → `handle_join_request` / `handle_leave_request` | same shape as 9000/9001 |
| **Agent channel-add policy** | 10100 | store → `handle_agent_profile` (`side_effects.rs:1078`) | `ensure_user`, `set_channel_add_policy`; consumed by the 9000 third-party-add gate (`side_effects.rs:340-365`) |
| **Profiles + NIP-05** | 0 | JSON pre-check (`ingest.rs:2234`) → store → `handle_kind0_profile` (`side_effects.rs:1113`) | absolute-state sync to `users`; NIP-05 canonicalised against the tenant host and silently cleared if off-domain; UNIQUE collision retried without the handle |
| **Direct messages (NIP-59 payload)** | 1059 | store with `channel_id = None`; pubkey-match waived; **WS only** | none |
| **DM channel lifecycle** | 41010, 41011, 41012 | `command_executor.rs:310` / `:443` / `:580` | `open_dm`/`hide_dm`, kind:40099 `dm_created`, 39000/39002 discovery, kind:44100 per participant, kind:30622 NIP-DV snapshot |
| **Workflows (definition)** | 30620 | `command_executor.rs:653` | `upsert_workflow`, webhook-secret generation/preservation, workflow cache invalidation |
| **Workflows (manual trigger)** | 46020 | `command_executor.rs:809` | `create_workflow_run` + spawned `execute_from_step` |
| **Workflow approvals** | 46030, 46031 | `command_executor.rs:986` / `:1098` | approval status update; grant spawns `resume_workflow_after_approval` (`:1236`), deny cancels the run |
| **Workflow deletion** | 5 with an `a` tag `30620:…` | `handle_a_tag_deletion` (`side_effects.rs:1990-2049`) | `delete_workflow_for_owner` by UUID or owner+name, cache invalidation |
| **Community moderation (ban/timeout/report resolution)** | 9040–9044 | direct route (`ingest.rs:1572-1588`) | out of module (`moderation_commands.rs`) |
| **Abuse reporting** | 1984 | direct route (`ingest.rs:1560-1570`) | `moderation_reports` row only |
| **Ban/timeout write enforcement** | all except 9030–9033, 9040–9044 | `moderation_restriction_state` gate (`ingest.rs:1613-1642`) | — |
| **Relay membership admin (NIP-43)** | 9030–9033 | direct route (`ingest.rs:1808-1818`) | 8000/8001 delta + 13534 snapshot from `relay_admin.rs` |
| **Self-service relay leave** | 28936 | direct route (`ingest.rs:1820-1902`) | `publish_nip43_member_removed` + `publish_nip43_membership_list` (both fire-and-forget) |
| **Identity archival (NIP-IA)** | 9035, 9036 | pre-storage handler then normal storage (`ingest.rs:1915-1919`) | 8002/8003 delta + 13535 snapshot |
| **Agent memory / engrams (NIP-AE)** | 30174 | `validate_engram_envelope` (`ingest.rs:1976-1979`) → NIP-33 store | none |
| **Agent telemetry (NIP-AM)** | 44200 | envelope + owner check (`ingest.rs:1981-2016`) → store | none |
| **Personas / teams / managed agents (NIP-AP)** | 30175, 30176, 30177 | 30175 gets `validate_persona_envelope`; 30176/30177 only the `d`-length check | none |
| **Event reminders (NIP-ER)** | 30300 | `validate_event_reminder` (`ingest.rs:2018-2021`) → NIP-33 store | none |
| **Web push leases (NIP-PL)** | 30350 | direct route (`ingest.rs:2156-2204`) | `push_leases` table |
| **Media attachments** | any kind with `imeta` tags | `validate_imeta_tags` + `verify_imeta_blobs` (`ingest.rs:2206-2218`) | none — the blob was uploaded out-of-band via Blossom |
| **Git repos (NIP-34)** | 30617, 30618, 1617–1621, 1630–1633 | 30617 → store → `handle_git_repo_announcement` (`side_effects.rs:2412`); the rest are plain stores | name reservation, empty-manifest pointer seed/repair, initial kind:30618 |
| **Huddles (audio)** | 48100–48103, 48106 | generic store, `h` required — **no validator, no side effect** | none |
| **User status / read state / NIP-51 lists / NIP-65 / NIP-30 emoji** | 30315, 30078, 10000, 10001, 10002, 10003, 10030, 30000, 30003, 30030 | replaceable / NIP-33 store, forced global | none |
| **Long-form posts** | 30023 | NIP-33 store, forced global | deletion supported via the generic coordinate soft-delete (`side_effects.rs:2051-2088`, referencing block/sprout#714) |
| **Product feedback** | 42000 | direct route (`ingest.rs:1538-1558`) | private deployment table |
| **Text notes** | 1 | generic store, forced global | none |
| **Contact list** | 3 | replaceable store, forced global | none |
| **Operational reconciliation** | — | not ingest; `main.rs` startup/periodic tasks | `reconcile_nip43_membership_snapshots` (`side_effects.rs:2776`, called `main.rs:508`, `:527`), `reconcile_channel_events` (`:2948`, called `main.rs:577`), `evict_all_channel_subscriptions` (`:128`, called `main.rs:672` by the ephemeral-channel reaper) |

---

### 2. Handled kinds whose feature has **no other reachable surface**

| Kind | Situation |
|---|---|
| 40004 `STREAM_MESSAGE_PINNED`, 40005 `…_BOOKMARKED`, 40006 `…_SCHEDULED`, 40007 `STREAM_REMINDER` | Accepted and stored with only the `h`-tag gate (`ingest.rs:455-491`). No pre-storage validator, no side effect, no `is_side_effect_kind` membership. Semantics are entirely client-side convention; the relay cannot tell a pin from a scheduled message. Nothing enforces that the `e` target exists or is in the same channel — unlike 40003 (`ingest.rs:763`) and 45002 (`:844`), which do check. |
| 48100–48103, 48106 huddles | Same: stored, `h` required, zero relay-side validation. A client can post `HUDDLE_ENDED` for a huddle that never started. |
| 40100 `CANVAS` | Same. |
| 30176 `TEAM`, 30177 `MANAGED_AGENT` | Accepted with only the 1024-byte `d`-length check. 30175 `PERSONA` gets a strict slug grammar (`ingest.rs:1027`) precisely because an empty `d` causes LWW data loss — 30176/30177 share that exact addressing shape and have **no** such guard, so an empty `d` collapses every team/managed-agent into one slot. Asymmetric by omission, not by design. |
| 30618 `GIT_REPO_STATE` | Client-submittable with `ReposWrite` scope, but the relay also mints it itself (`side_effects.rs:2733`, and `api/git/manifest_event.rs` on push). A client can publish a competing 30618 for a repo it does not own — the `d`-tag coordinate includes the *submitter's* pubkey, so it cannot overwrite the relay-signed head, but it can pollute the kind. |
| 1617–1621, 1630–1633 git patches/PRs/issues/statuses | Stored with `MessagesWrite` and forced global. No validation that the referenced repo exists or that the author has any relationship to it. |

---

### 3. Declared kinds ingest **rejects** — feature reachability

47 of the 127 `ALL_KINDS` entries never pass `required_scope_for_kind`
(`ingest.rs:198-306`). Grouped by why:

| Group | Kinds | Assessment |
|---|---|---|
| **Relay-signed outputs of this very module** | 8000, 8001, 8002, 8003, 13534, 13535, 39000, 39001, 39002 | Correct to reject. But note only 13534 is in `is_relay_only_kind` (`buzz-core/src/kind.rs:682-693`); the other eight fall through to the generic `restricted: unknown event kind`. Same outcome, worse diagnostic. |
| **Relay-signed, correctly classified** | 30622, 39005, 39006, 40901, 40902 | `is_relay_only_kind` → `restricted: relay-only kind` (`ingest.rs:1455`). |
| **Relay-signed, special-cased** | 44100, 44101 | Own message: `invalid: membership notifications are relay-signed only` (`ingest.rs:1443`). |
| **Relay-signed, no guard at all** | 40099 `SYSTEM_MESSAGE` | Minted by `emit_system_message` (`side_effects.rs:677`) but absent from `is_relay_only_kind`. It appears in `is_side_effect_kind` (`side_effects.rs:36`), which is dead code since ingest rejects it. |
| **Ephemeral (handled upstream)** | 20001, 20002, 24134, 24200, 24810 | Dispatched in `handlers/event.rs` before ingest; `ephemeral_kinds_not_in_scope_allowlist` (`ingest.rs:2764-2767`) asserts the exclusion. |
| **Auth artefacts** | 24242 `BLOSSOM_AUTH`, 27235 `HTTP_AUTH` | Consumed by the media and NIP-98 paths, never ingested. Correct. |
| **Unimplemented features** | 9009 `NIP29_CREATE_INVITE`, 39003 `NIP29_GROUP_ROLES`, 41 `CHANNEL_METADATA`, 1063 `FILE_METADATA`, 41001 `DM_CREATED`, 43001–43006 job kinds (6), 46001–46012 workflow-execution kinds (10), 48001 `AUDIT_ENTRY`, 49001 `MEDIA_UPLOAD` | **26 declared kinds with no ingest path.** The job kinds (43001–43006) and workflow-execution kinds (46001–46012) are declared, documented in `buzz-core/src/kind.rs:380-446`, and unreachable: nothing can publish them. 48001 `AUDIT_ENTRY` is particularly notable — the audit log is Postgres-only (`buzz-audit/src/service.rs`), so this kind exists purely as a reservation. |

The single most concrete mismatch: **9009** has a live `handle_side_effects` arm
(`side_effects.rs:161-167`) that logs `"NIP-29 kind 9009 handler deferred to future
phase"` and returns `Ok`. It cannot be reached — 9009 is not in the allowlist. The arm is
provably dead.

---

### 4. Cross-crate feature mismatches found

| Mismatch | Evidence |
|---|---|
| **Workflow `add_reaction` action is broken.** `buzz-workflow` POSTs to `{base_url}/api/messages/{message_id}/reactions` (`buzz-workflow/src/executor.rs:883-889`). No such route exists — the relay registers exactly three `/api/*` paths (`router.rs:95`, `:96`, `:111`) and no `/api/messages` prefix anywhere. Meanwhile ingest **does** have a full native reaction path (kind:7, `ingest.rs:2245-2365`) that the workflow action does not use. Any workflow with an `add_reaction` step will 404 and surface `AddReaction: relay returned {status}` (`executor.rs:917`). | `buzz-workflow/src/executor.rs:883-917`; `crates/buzz-relay/src/router.rs` |
| **Reaction-triggered workflows can fire but cannot react back.** `reaction_added` is a valid workflow trigger (`buzz-workflow/src/schema.rs:290`), and kind:7 does reach `dispatch_persistent_event` → `workflow_engine.on_event` (`handlers/event.rs:502-531`), so the trigger works. Only the response action is dead. | as above |
| **9 of 11 audit actions have no producer.** `AuditAction` declares 11 variants (`buzz-audit/src/action.rs:8-31`). Production emits exactly two: `EventCreated` (`handlers/event.rs:560`) and `MediaUploaded` (`api/media.rs`). `EventDeleted`, `ChannelCreated`, `ChannelUpdated`, `ChannelDeleted`, `MemberAdded`, `MemberRemoved`, `AuthSuccess`, `AuthFailure`, `RateLimitExceeded` are never written — despite this module performing every one of those actions. | grep for `AuditAction::*` outside `crates/buzz-audit/` and `tests/` |
| **`is_side_effect_kind` claims two ranges that no kind can reach.** `41001..=41003` and `40099` (`side_effects.rs:36`) — 41001 is `DM_CREATED` (not in the allowlist), 41002/41003 are undefined, 40099 is relay-signed. The DM *command* kinds are 41010–41012 and are routed to `command_executor` long before this predicate runs (`ingest.rs:1534`). | `side_effects.rs:35-37` vs `ingest.rs:198-306` |
