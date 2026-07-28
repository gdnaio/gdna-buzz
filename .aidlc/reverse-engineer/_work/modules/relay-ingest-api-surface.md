## Module: buzz-relay — event ingest & side effects (`crates/buzz-relay/src/handlers`)
### Aspect: API Surface

This module group has **no HTTP routes of its own**. Its API is (a) the Rust entry point
`ingest_event`, called by exactly two transports, and (b) the **kind → handler dispatch
table** that entry point implements.

---

### 1. Rust entry points and their callers

| Item | Signature | file:line | Production callers |
|---|---|---|---|
| `ingest_event` | `async fn(&Arc<AppState>, &TenantContext, Event, IngestAuth) -> Result<IngestResult, IngestError>` | `ingest.rs:1367-1425` | WS `EVENT` (`handlers/event.rs`, via `IngestAuth::Nip42`); HTTP `POST /events` (`api/bridge.rs:830`, via `IngestAuth::Http`) |
| `ingest_event_inner` | private, same signature + `&Arc<dyn Tracer>` | `ingest.rs:1427-2505` | `ingest_event` only |
| `reject_with_transport` | `fn(&'static str, &'static str)` | `ingest.rs:156-164` | `api/bridge.rs:783,851,859,867`; `handlers/event.rs:31` |
| `handle_command` | `async fn(&TenantContext, &Arc<AppState>, Event, IngestAuth) -> Result<IngestResult, IngestError>` | `command_executor.rs:36-78` | `ingest.rs:1535` only |
| `handle_side_effects` | `async fn(&TenantContext, u32, &Event, &Arc<AppState>) -> anyhow::Result<()>` | `side_effects.rs:142-177` | `ingest.rs:2436` only |
| `validate_admin_event` | `async fn(&TenantContext, u32, &Event, &Arc<AppState>) -> anyhow::Result<()>` | `side_effects.rs:259-676` | `ingest.rs:1906` only |
| `validate_standard_deletion_event` | `async fn(&TenantContext, &Event, &Arc<AppState>) -> anyhow::Result<()>` | `side_effects.rs:179-238` | `ingest.rs:1922` only |
| `validate_imeta_tags` | `fn(&[Vec<String>], &str) -> Result<(), String>` | `imeta.rs:11-208` | `ingest.rs:2213` (re-exported via `crate::api`) |
| `verify_imeta_blobs` | `async fn(&TenantContext, &[Vec<String>], &MediaStorage) -> Result<(), String>` | `imeta.rs:209-335` | `ingest.rs:2215` |

#### Public types

| Type | Variants / fields | file:line |
|---|---|---|
| `HttpAuthMethod` | `Nip98`, `DevPubkey` | `ingest.rs:54-59` |
| `IngestAuth` | `Nip42 { pubkey, scopes, channel_ids, conn_id }`, `Http { pubkey, scopes, auth_method }` | `ingest.rs:63-86` |
| `IngestResult` | `event_id: String`, `accepted: bool`, `message: String` | `ingest.rs:166-173` |
| `IngestError` | `Rejected(String)` → WS `OK false` / HTTP 400; `AuthFailed(String)` → HTTP 403; `Internal(String)` → HTTP 500 | `ingest.rs:177-184` |
| `ReactionChannelResult` | `Channel(Uuid)`, `NoChannel`, `NotFound`, `NoTarget`, `DbError(String)` | `ingest.rs:322-328` |
| `ThreadMetadataOwned` | 9 fields; `as_params()` → `buzz_db::event::ThreadMetadataParams` | `ingest.rs:535-561` |
| `PersistResult` (private) | `Duplicate`, `Inserted(sqlx::Transaction)` | `command_executor.rs:80-85` |

`IngestAuth` accessors: `pubkey()` `:88`, `principal_pubkey_bytes()` `:95`, `scopes()` `:100`,
`conn_id()` `:107`, `channel_ids()` `:117`, `is_http()` `:128`.

---

### 2. Kind acceptance census (verified)

| Measure | Count | Evidence |
|---|---|---|
| Kind constants in `buzz-core` | 130 | `crates/buzz-core/src/kind.rs` |
| `ALL_KINDS` entries | 127 | `kind.rs:490-617` |
| Distinct kinds accepted by `required_scope_for_kind` | **81** | `ingest.rs:198-306` (enumerated below) |
| …of which are in `ALL_KINDS` | 80 | `KIND_PUSH_LEASE` (30350) is accepted but absent from `ALL_KINDS` |
| `ALL_KINDS` entries **rejected** by ingest | **47** | see §5 |
| Kinds routed away from generic storage entirely | 15 | 7 command kinds + 5 moderation commands + 1984 + 42000 + 30350 |
| Kinds that get a bespoke pre-storage validator | 12 | 0, 5, 9002/9005/9000/9001/9008/9022 (via `validate_admin_event`), 9035/9036, 40003, 40008, 44200, 45002, 30174, 30175, 30300, 9007, 9021 |
| Side-effect dispatch arms in `handle_side_effects` | **13** + `_` | `side_effects.rs:143-176` |
| `validate_admin_event` match arms | **7** + `_` | `side_effects.rs:284-675` |
| `handle_command` match arms | **7** + `_` | `command_executor.rs:66-77` |

---

### 3. THE DISPATCH TABLE — every kind ingest accepts

Legend for **Route**: `store` = generic storage + fan-out; `store+SE` = stored then
`handle_side_effects`; `direct` = handled and returned, never stored; `store+direct` =
handler runs pre-storage *and* the event is stored.
`Scope` is the value returned by `required_scope_for_kind` (`ingest.rs:198`) — but see
security.md BR: in production **every** authenticated principal holds
`Scope::all_known()` (`buzz-auth/src/lib.rs:137`; `api/bridge.rs:827`), so the scope column
documents intent, not enforcement. `h`: `R` = required (`requires_h_channel_scope`
`ingest.rs:455`), `N` = forced-null (`is_global_only_kind` `ingest.rs:379`),
`O` = optional/derived.

| Kind | Constant | Scope | `h` | Extra required tags / pre-storage validator | Route | Distinct reject strings |
|---|---|---|---|---|---|---|
| 0 | `KIND_PROFILE` | UsersWrite | N | content must parse as JSON (`ingest.rs:2234`) | store+SE (`handle_kind0_profile` `side_effects.rs:1113`) | `invalid: kind:0 content must be valid JSON` |
| 1 | `KIND_TEXT_NOTE` | MessagesWrite | N | — | store | — |
| 3 | `KIND_CONTACT_LIST` | UsersWrite | N | — | store (replaceable) | — |
| 5 | `KIND_DELETION` | MessagesWrite | O (derived from target) | exactly one `e`+`a` target (`ingest.rs:1946`); `validate_standard_deletion_event` (`ingest.rs:1922`) | store+SE (`handle_standard_deletion_event` `side_effects.rs:2108`) | `invalid: malformed deletion target id`; `invalid: deletion events must reference exactly one target via e or a tag (got e=…, a=…)`; `invalid: must be event author`; `invalid: target event not found`; `invalid: missing e or a tag for target`; `invalid: invalid a-tag format`; `invalid: invalid pubkey in a-tag` |
| 7 | `KIND_REACTION` | MessagesWrite | O (derived from target's channel, `ingest.rs:1645`) | ≥1 `e` tag w/ 64-hex; emoji ≤64 chars (`ingest.rs:2283`) | **inline transactional** (`insert_reaction_event_with_thread_metadata` `ingest.rs:2298`) — deliberately excluded from `is_side_effect_kind` (`side_effects.rs:31-34`) | `invalid: reaction must reference a target event via e tag`; `invalid: reaction target event not found`; `invalid: malformed reaction target id`; `invalid: reaction emoji exceeds 64 characters (got n)`; OK-but-rejected `duplicate: reaction already exists` |
| 9 | `KIND_STREAM_MESSAGE` | MessagesWrite | **R** | NIP-10 `e root`/`e reply` resolved (`ingest.rs:2220`) | store | `invalid: channel-scoped events must include an h tag`; `invalid: reply parent not found`; `invalid: parent event belongs to a different channel`; `invalid: parent event has no channel association`; `invalid: root tag does not match thread ancestry`; `invalid: thread depth limit exceeded` |
| 1059 | `KIND_GIFT_WRAP` | MessagesWrite | forced `None` (`ingest.rs:1664`) | pubkey-match check waived (`ingest.rs:1499`) | store (**WS only** — `ingest.rs:1449`) | `invalid: kind 1059 is only accepted via WebSocket` |
| 1617 | `KIND_GIT_PATCH` | MessagesWrite | N | — | store | — |
| 1618 | `KIND_GIT_PULL_REQUEST` | MessagesWrite | N | — | store | — |
| 1619 | `KIND_GIT_PR_UPDATE` | MessagesWrite | N | — | store | — |
| 1621 | `KIND_GIT_ISSUE` | MessagesWrite | N | — | store | — |
| 1630–1633 | `KIND_GIT_STATUS_{OPEN,MERGED,CLOSED,DRAFT}` | MessagesWrite | N | — | store | — |
| 1984 | `KIND_REPORT` | MessagesWrite | — | `report::handle_report_event` (`ingest.rs:1562`) | **direct** → `moderation_reports` only; never stored, never fanned out | whatever `handle_report_event` returns, wrapped verbatim as `Rejected` |
| 9000 | `KIND_NIP29_PUT_USER` | AdminChannels | **R** | `p` tag; optional `role`; `validate_admin_event` 9000 arm (`side_effects.rs:284`) | store+SE (`handle_put_user` `side_effects.rs:1203`) | `invalid: missing p tag`; `invalid: invalid role: X`; `invalid: actor not authorized`; `invalid: only owners/admins may grant elevated roles`; `invalid: policy:owner_only — only the agent owner can add this agent`; `invalid: policy:owner_only — agent has no owner set`; `invalid: policy:nobody — this agent has disabled external channel additions` |
| 9001 | `KIND_NIP29_REMOVE_USER` | AdminChannels | **R** | `p` tag; 9001 arm (`side_effects.rs:373`) | store+SE (`handle_remove_user` `side_effects.rs:1265`) | `invalid: missing p tag`; `invalid: actor is not an active member`; `invalid: cannot remove the last owner`; `invalid: actor not authorized` |
| 9002 | `KIND_NIP29_EDIT_METADATA` | AdminChannels if an `archived` tag is present, else ChannelsWrite (`ingest.rs:276-287`) | **R** | ≥1 of `name/about/archived/topic/purpose/visibility/ttl` (`side_effects.rs:410`) | store+SE (`handle_edit_metadata` `side_effects.rs:1335`) | `invalid: kind:9002 must include at least one metadata tag (…)`; `invalid: invalid archived value: X (must be "true" or "false")`; `invalid: archived tag must have a value`; `invalid: channel name is required`; `invalid: invalid visibility value: X`; `invalid: visibility tag must have a value`; `invalid: invalid ttl value: X`; `invalid: ttl tag must have a value (…)`; `invalid: actor not authorized for name/about/archived/visibility/ttl changes`; `invalid: not a member` |
| 9005 | `KIND_NIP29_DELETE_EVENT` | MessagesWrite | **R** | exactly one `e`+`a` (`ingest.rs:1946`); 9005 arm (`side_effects.rs:508`) | store+SE (`handle_delete_event_side_effect` `side_effects.rs:1560`) | `invalid: missing e tag for target event`; `invalid: invalid action_id tag`; `invalid: target event not found`; `invalid: target event belongs to a different channel`; `invalid: target event has no channel`; `invalid: must be event author or channel owner/admin` |
| 9007 | `KIND_NIP29_CREATE_GROUP` | ChannelsWrite | O (client-chosen UUID) | `name` non-empty after canonicalisation; `visibility`∈{open,private} default open; `channel_type` default stream (`ingest.rs:2031-2085`) | pre-create channel (`ingest.rs:2103`) then store+SE (`handle_create_group` `side_effects.rs:1660`) | `invalid: channel name is required`; `invalid visibility: X`; `invalid channel_type: X`; OK-but-rejected `duplicate: channel already exists` |
| 9008 | `KIND_NIP29_DELETE_GROUP` | AdminChannels | **R** | 9008 arm — owner only or owner-of-owner-agent (`side_effects.rs:625`) | store+SE (`handle_delete_group` `side_effects.rs:1783`) | `invalid: only owner can delete group` |
| 9021 | `KIND_NIP29_JOIN_REQUEST` | ChannelsRead | O (but rejected if absent, `ingest.rs:2136`) | channel must exist and be `open` (`ingest.rs:2141`) | store+SE (`handle_join_request` `side_effects.rs:1835`) | `invalid: join request must include an h tag`; `restricted: channel is private`; `invalid: channel not found`; SE-only: `channel is private — request an invitation` |
| 9022 | `KIND_NIP29_LEAVE_REQUEST` | ChannelsRead | **R** | 9022 arm — active member, not last owner (`side_effects.rs:645`) | store+SE (`handle_leave_request` `side_effects.rs:1913`) | `invalid: actor is not an active member`; `invalid: cannot remove the last owner` |
| 9030 | `RELAY_ADMIN_ADD_MEMBER` | AdminUsers | N | delegated to `relay_admin::handle_relay_admin_event` (`ingest.rs:1809`) | **direct** — mutates `relay_members`, not stored | `restricted: relay admin commands require a global token, not a channel-scoped token`; `invalid: {handler error}` |
| 9031 | `RELAY_ADMIN_REMOVE_MEMBER` | AdminUsers | N | ditto | direct | ditto |
| 9032 | `RELAY_ADMIN_CHANGE_ROLE` | AdminUsers | N | ditto | direct | ditto |
| 9033 | `RELAY_ADMIN_SET_WORKSPACE_PROFILE` | AdminUsers | N | ditto | direct | ditto |
| 9035 | `KIND_IA_ARCHIVE_REQUEST` | **UsersWrite** (deliberately not AdminUsers — `ingest.rs:265-273`) | N | `identity_archive::handle_identity_archive_event` (`ingest.rs:1916`) | **store+direct** — handler runs pre-storage, then the request itself is stored so the 8002 delta's `["e", request_id]` resolves | `invalid: {handler error}` |
| 9036 | `KIND_IA_UNARCHIVE_REQUEST` | UsersWrite | N | ditto | store+direct | `invalid: {handler error}` |
| 9040 | `KIND_MODERATION_BAN` | MessagesWrite | N | `moderation_commands::handle_moderation_command` (`ingest.rs:1580`) | **direct** — exempt from the ban/timeout write-block gate (`ingest.rs:1613`) | handler string verbatim (`restricted: moderator access required`, `invalid: event timestamp out of range: …`, …) |
| 9041–9044 | `KIND_MODERATION_{UNBAN,TIMEOUT,UNTIMEOUT,RESOLVE_REPORT}` | MessagesWrite | N | ditto | direct | ditto |
| 10000 | `KIND_MUTE_LIST` | UsersWrite | N | — | store (replaceable) | — |
| 10001 | `KIND_PIN_LIST` | UsersWrite | N | — | store (replaceable) | — |
| 10002 | `KIND_NIP65_RELAY_LIST_METADATA` | UsersWrite | N | — | store (replaceable) | — |
| 10003 | `KIND_BOOKMARK_LIST` | UsersWrite | N | — | store (replaceable) | — |
| 10030 | `KIND_EMOJI_LIST` | UsersWrite | N | — | store (replaceable) | — |
| 10100 | `KIND_AGENT_PROFILE` | UsersWrite | N | — | store+SE (`handle_agent_profile` `side_effects.rs:1078`) | SE-only: `kind:10100 content parse error: …`; `kind:10100 missing channel_add_policy field` |
| 28936 | `KIND_NIP43_LEAVE_REQUEST` | ChannelsRead | N | `require_relay_membership=true`; ±120 s freshness; NIP-70 `["-"]` tag (`ingest.rs:1820-1856`) | **direct** — removes from `relay_members`, not stored | `restricted: leave requests require a global token`; `invalid: relay membership is not enabled`; `invalid: leave request timestamp out of range (delta=Ns, max ±120s)`; `invalid: leave request must include NIP-70 protected event tag ["-"]`; `invalid: you are not a relay member`; `invalid: relay owner cannot leave`; success msg `info: you have left this relay` |
| 30000 | `KIND_FOLLOW_SET` | UsersWrite | N | `d` ≤1024 B | store (NIP-33) | `invalid: d tag too long (n bytes, max 1024)` |
| 30003 | `KIND_BOOKMARK_SET` | UsersWrite | N | ditto | store (NIP-33) | ditto |
| 30023 | `KIND_LONG_FORM` | MessagesWrite | N | ditto | store (NIP-33) | ditto |
| 30030 | `KIND_EMOJI_SET` | UsersWrite | N | ditto | store (NIP-33) | ditto |
| 30078 | `KIND_READ_STATE` | UsersWrite | N | ditto | store (NIP-33) | ditto |
| 30174 | `KIND_AGENT_ENGRAM` | UsersWrite | N | exactly one 64-lc-hex `d`; exactly one 64-lc-hex `p`; content = NIP-44 v2 shape (`ingest.rs:965-1025`) | store (NIP-33) | `invalid: agent-engram event must have exactly one \`d\` tag (got n)`; `…one \`p\` tag (got n)`; `invalid: agent-engram \`d\` tag must be 64 lowercase hex chars`; `invalid: agent-engram \`p\` tag must be 64 lowercase hex chars (pubkey)`; `invalid: agent-engram content must not be empty (NIP-44 ciphertext)`; `invalid: agent-engram content is not valid base64 (length)`; `invalid: agent-engram content is not valid base64`; `invalid: agent-engram content too short for NIP-44 v2`; `invalid: agent-engram content is not NIP-44 v2 (expected 0x02 version prefix)` |
| 30175 | `KIND_PERSONA` | UsersWrite | N | exactly one non-empty `d` matching `^[a-z0-9][a-z0-9_-]{0,63}$` (`ingest.rs:1027-1082`) | store (NIP-33) | `invalid: persona event must have exactly one \`d\` tag (got n)`; `invalid: persona event \`d\` tag must not be empty`; `invalid: persona event \`d\` tag too long (n chars, max 64)`; `invalid: persona event \`d\` tag must start with a lowercase letter or digit`; `invalid: persona event \`d\` tag must match [a-z0-9_-] after the first character` |
| 30176 | `KIND_TEAM` | UsersWrite | N | `d` ≤1024 B only | store (NIP-33) | `invalid: d tag too long …` |
| 30177 | `KIND_MANAGED_AGENT` | UsersWrite | N | ditto | store (NIP-33) | ditto |
| 30300 | `KIND_EVENT_REMINDER` | UsersWrite | N | exactly one non-empty `d`; ≤1 `not_before` (canonical decimal ≤ 2^53−1, ≤ now+`SPROUT_MAX_NOT_BEFORE_DELTA`); `expiration > not_before` when both present (`ingest.rs:1252-1326`) | store (NIP-33) | `invalid: malformed not_before`; `invalid: missing d tag`; `invalid: duplicate d tag`; `invalid: empty d tag`; `invalid: not_before too far in future`; `invalid: expiration before not_before` |
| 30315 | `KIND_USER_STATUS` | UsersWrite | N | `d` ≤1024 B | store (NIP-33) | `invalid: d tag too long …` |
| 30350 | `push_lease::KIND_PUSH_LEASE` | UsersWrite | N | `push_lease::accept` (`ingest.rs:2157`) | **direct** — `push_leases` table, not the events table | `invalid: stale replacement`; `invalid: stale generation`; `invalid: endpoint already leased`; `invalid: lease quota exceeded`; `invalid: source event collision`; `invalid: lease constraint violation`; `invalid: {validation}`; `Internal({reason})` |
| 30617 | `KIND_GIT_REPO_ANNOUNCEMENT` | **ReposWrite** | N | `d` = repo id `[a-zA-Z0-9._-]{1,64}`, no leading dot, no `..` (`side_effects.rs:2391`) | store+SE (`handle_git_repo_announcement` `side_effects.rs:2412`) | SE-only (logged, not returned): `kind:30617 missing d tag`; `invalid repo identifier: …`; `repo name 'X' already taken by another owner`; `repo limit exceeded: n >= m`; `failed to ensure manifest pointer: …` |
| 30618 | `KIND_GIT_REPO_STATE` | ReposWrite | N | `d` ≤1024 B | store (NIP-33) | `invalid: d tag too long …` |
| 30620 | `KIND_WORKFLOW_DEF` | MessagesWrite | required by handler (`h`) | `h` UUID; `d` = workflow UUID; caller must be channel member; YAML parses (`command_executor.rs:653`) | **command executor** (own tx) | `invalid: missing h tag (channel_id)`; `invalid: bad channel_id format`; `invalid: missing d tag (workflow_id)`; `invalid: bad workflow_id format`; `forbidden: not a member of this channel`; `invalid: workflow YAML parse error: …`; `forbidden: workflow belongs to a different owner or channel`; `invalid: workflow channel not found`; `invalid: d tag too long …`; `duplicate: already processed` |
| 40002 | `KIND_STREAM_MESSAGE_V2` | MessagesWrite | **R** | NIP-10 resolution | store | same thread errors as kind 9 |
| 40003 | `KIND_STREAM_MESSAGE_EDIT` | MessagesWrite | **R** | `validate_edit_ownership` (`ingest.rs:763`) — target exists, same channel, actor is effective author **or** the agent's owning human; author path re-gates membership | store | `invalid: missing e tag for edit target`; `invalid: invalid target event ID`; `invalid: edit target event not found`; `invalid: target event belongs to a different channel`; `invalid: target event has no channel`; `invalid: must be event author to edit`; `invalid: restricted: not a channel member`; `invalid: db error…` |
| 40004 | `KIND_STREAM_MESSAGE_PINNED` | MessagesWrite | **R** | — | store | h-tag error only |
| 40005 | `KIND_STREAM_MESSAGE_BOOKMARKED` | MessagesWrite | **R** | — | store | h-tag error only |
| 40006 | `KIND_STREAM_MESSAGE_SCHEDULED` | MessagesWrite | **R** | — | store | h-tag error only |
| 40007 | `KIND_STREAM_REMINDER` | MessagesWrite | **R** | — | store | h-tag error only |
| 40008 | `KIND_STREAM_MESSAGE_DIFF` | MessagesWrite | **R** | `validate_diff_event` (`ingest.rs:896`) — content ≤60 KiB, `repo` http(s), `commit` ≥7 hex | store | `invalid: diff content exceeds 60KB limit (got n bytes)`; `invalid: repo URL must be http or https`; `invalid: commit SHA must be at least 7 hex characters`; `invalid: parent-commit SHA must be at least 7 hex characters`; `invalid: branch tag requires both source and target`; `invalid: pr number must be a positive integer`; `invalid: diff event requires a repo tag`; `invalid: diff event requires a commit tag` |
| 40100 | `KIND_CANVAS` | ChannelsWrite | **R** | — | store | h-tag error only |
| 41010 | `KIND_DM_OPEN` | MessagesWrite | — | 1–8 `p` tags (9 total max) (`command_executor.rs:310`) | command executor | `invalid: pubkeys must contain at least 1 other participant`; `invalid: pubkeys may contain at most 8 other participants (9 total)`; `invalid: bad pubkey hex: X`; `invalid: pubkey must be 32 bytes: X`; success `response:{"channel_id":…,"created":bool}`; `duplicate: already processed` |
| 41011 | `KIND_DM_ADD_MEMBER` | MessagesWrite | `h` required by handler | `h` UUID + ≥1 `p`; caller must be a member; channel must be `dm`; total ≤9 (`command_executor.rs:443`) | command executor (creates a **new** DM — sets are immutable) | `invalid: missing h tag (channel_id)`; `invalid: bad channel_id format`; `invalid: must specify at least 1 new participant in p tags`; `forbidden: not a member of this DM`; `invalid: DM not found`; `invalid: channel is not a DM`; `invalid: DM supports at most 9 participants`; success `response:{"channel_id":…}` |
| 41012 | `KIND_DM_HIDE` | MessagesWrite | `h` required by handler | `h` UUID; member; channel is `dm` (`command_executor.rs:580`) | command executor | `invalid: missing h tag (channel_id)`; `invalid: bad channel_id format`; `forbidden: not a member of this DM`; `invalid: DM not found`; `invalid: channel is not a DM`; success message `{}` |
| 42000 | `KIND_PRODUCT_FEEDBACK` | MessagesWrite | — | `product_feedback::handle` (`ingest.rs:1541`) | **direct** → private deployment table; no storage, no fan-out | handler string verbatim |
| 44200 | `KIND_AGENT_TURN_METRIC` | MessagesWrite | N (an explicit `h` tag is a **hard reject**) | exactly one 64-lc-hex `p`; exactly one 64-lc-hex `agent` == `event.pubkey`; no `h`; NIP-44 v2 content; `p` must be the DB-registered owner of `event.pubkey` (`ingest.rs:1151-1221`, `:1981-2016`) | store | ``invalid: agent-turn-metric event must not have an `h` tag (…)``; ``invalid: agent-turn-metric event must have exactly one `p` tag (got n)``; ``invalid: agent-turn-metric `p` tag must be 64 lowercase hex chars``; ``invalid: agent-turn-metric event must have exactly one `agent` tag (got n)``; ``invalid: agent-turn-metric `agent` tag must be 64 lowercase hex chars``; ``invalid: agent-turn-metric `agent` tag must equal event pubkey``; `invalid: agent-turn-metric content …`; ``restricted: agent-turn-metric `p` tag must be the registered owner of this agent`` |
| 45001 | `KIND_FORUM_POST` | MessagesWrite | **R** | NIP-10 resolution | store | h-tag + thread errors |
| 45002 | `KIND_FORUM_VOTE` | MessagesWrite | **R** | `validate_forum_vote_target` (`ingest.rs:844`) — target exists, kind ∈ {45001, 45003}, same channel | store | `invalid: missing e tag for vote target`; `invalid: invalid target event ID`; `invalid: vote target event not found`; `invalid: vote target must be a forum post or comment`; `invalid: target event belongs to a different channel`; `invalid: target event has no channel` |
| 45003 | `KIND_FORUM_COMMENT` | MessagesWrite | **R** | NIP-10 resolution | store | h-tag + thread errors |
| 46020 | `KIND_WORKFLOW_TRIGGER` | MessagesWrite | — | `d` or `e` = workflow UUID; caller must be workflow **owner** (`command_executor.rs:809`) | command executor; persisted under `workflow.channel_id` | `invalid: missing workflow reference (d or e tag)`; `invalid: bad workflow_id format`; `invalid: workflow not found`; `forbidden: not authorized to trigger this workflow`; success `response:{"run_id":…}` |
| 46030 | `KIND_APPROVAL_GRANT` | MessagesWrite | — | `d`/`e` = token hash hex; approval pending + unexpired; `check_approver_spec` (`command_executor.rs:961`) | command executor | `invalid: missing approval reference (d or e tag)`; `invalid: bad approval token hash hex`; `invalid: approval not found`; `invalid: approval already {status}`; `invalid: approval token has expired`; `forbidden: not the designated approver for this request`; `forbidden: approver spec 'X' is not yet supported`; `invalid: approval already acted on (race)`; success `response:{"status":"granted","run_id":…}` |
| 46031 | `KIND_APPROVAL_DENY` | MessagesWrite | — | ditto | command executor | ditto, success `response:{"status":"denied","run_id":…}` |
| 48100–48103 | `KIND_HUDDLE_{STARTED,PARTICIPANT_JOINED,PARTICIPANT_LEFT,ENDED}` | ChannelsWrite | **R** | — | store | h-tag error only |
| 48106 | `KIND_HUDDLE_GUIDELINES` | ChannelsWrite | **R** | — | store | h-tag error only |

---

### 4. Cross-cutting reject strings (apply to every kind)

Emitted before per-kind dispatch, in this order:

| # | Condition | Variant | String | file:line |
|---|---|---|---|---|
| 1 | kind == 22242 | Rejected | `invalid: AUTH events cannot be submitted` | `ingest.rs:1438` |
| 2 | kind ∈ {44100, 44101} | Rejected | `invalid: membership notifications are relay-signed only` | `ingest.rs:1443` |
| 3 | HTTP + kind ∈ {1059, 20001} | Rejected | `invalid: kind {n} is only accepted via WebSocket` | `ingest.rs:1449` |
| 4 | `is_relay_only_kind` (13534, 30622, 39005, 39006, 40901, 40902) | Rejected | `restricted: relay-only kind` | `ingest.rs:1455` |
| 5 | signature/id verify fail | Rejected | `invalid: {VerificationError}` | `ingest.rs:1469` |
| 5b | verify task panic | Internal | `error: internal verification error` | `ingest.rs:1473` |
| 6 | \|created_at − now\| > 900 s | Rejected | `invalid: event timestamp too far from server time` | `ingest.rs:1483` |
| 7 | content > 262 144 B | Rejected | `invalid: content exceeds maximum size of 262144 bytes (got n)` | `ingest.rs:1490` |
| 8 | `event.pubkey != auth.pubkey` and not gift-wrap | **AuthFailed** | `invalid: event pubkey does not match authenticated identity` | `ingest.rs:1499` |
| 9 | kind not in the 81-kind allowlist | Rejected | `restricted: unknown event kind` | `ingest.rs:305`, raised at `:1507` |
| 10 | relay-admin kind + channel-scoped token | AuthFailed | `restricted: relay admin commands require a global token, not a channel-scoped token` | `ingest.rs:1512` |
| 11 | kind 28936 + channel-scoped token | AuthFailed | `restricted: leave requests require a global token` | `ingest.rs:1520` |
| 12 | missing scope | AuthFailed | `restricted: insufficient scope (need {scope})` | `ingest.rs:1525` |
| 13 | actor banned (all kinds except 9040–9044 and 9030–9033) | AuthFailed | `blocked: you are banned from this community` | `ingest.rs:1621` |
| 14 | actor timed out | AuthFailed | `restricted: you are timed out until {unix_ts}` | `ingest.rs:1627` |
| 15 | restriction lookup DB error (fail-closed) | Internal | `error: internal error checking restriction state: {e}` | `ingest.rs:1637` |
| 16 | requires `h`, none present | Rejected | `invalid: channel-scoped events must include an h tag` | `ingest.rs:1713` |
| 17 | channel-scoped token + global event | AuthFailed | `restricted: channel-scoped tokens cannot publish global events` | `ingest.rs:1726` |
| 18 | not a member and channel not `open` | Rejected | `restricted: not a channel member` | `ingest.rs:521` |
| 19 | membership lookup failed | Rejected | `error: database error: {e}` (note: **`Rejected`**, not `Internal`) | `ingest.rs:501` |
| 20 | channel `archived_at IS NOT NULL` (except 9002 `archived=false`) | Rejected | `invalid: channel is archived` | `ingest.rs:1938` |
| 21 | replaceable/NIP-33 write lost the LWW race | **Ok** `accepted=true` | `duplicate:` | `ingest.rs:2427-2431` |

`skip_membership` set (`ingest.rs:1770-1775`): 9021, 9007, 40003, 9002, 9005, 9008 — six
kinds bypass gate #18 and rely on their per-kind validator instead.

---

### 5. `ALL_KINDS` entries ingest **rejects** (47)

| Reject reason | Count | Kinds |
|---|---|---|
| `restricted: relay-only kind` (`ingest.rs:1455`) | 6 | 13534, 30622, 39005, 39006, 40901, 40902 |
| `invalid: membership notifications are relay-signed only` | 2 | 44100, 44101 |
| `restricted: unknown event kind` (no allowlist arm) | 39 | 41, 1063, 8000, 8001, 8002, 8003, 9009, 13535, 20001, 20002, 24134, 24200, 24242, 24810, 27235, 39000, 39001, 39002, 39003, 40099, 41001, 43001–43006, 46001–46007, 46010–46012, 48001, 49001 |

Notes:
- 20001 (`KIND_PRESENCE_UPDATE`) and 20002 are ephemeral and handled in
  `handlers/event.rs` before ingest — they never reach `required_scope_for_kind`.
  The HTTP guard at `ingest.rs:1449` names 20001 explicitly even though the kind can
  never pass the allowlist anyway (belt-and-braces, or vestigial).
- 8000/8001/8002/8003/13535/39000–39002 are relay-signed *outputs* of this very module
  (`side_effects.rs:2881`, `:3008`, `:3142`, `:962`) but are **not** in
  `is_relay_only_kind`, so their rejection message is the generic
  `restricted: unknown event kind` rather than `restricted: relay-only kind`.
- 9009 (`KIND_NIP29_CREATE_INVITE`) has a live `handle_side_effects` arm
  (`side_effects.rs:161-167`) that can never be reached — see debt.md D-08.

---

### 6. Success-response shapes

| Route | `accepted` | `message` |
|---|---|---|
| Generic store, new | `true` | `""` (`ingest.rs:2501`) |
| Generic store, LWW-dominated / dup id | `true` | `"duplicate:"` (`ingest.rs:2430`) |
| Reaction, new | `true` | `""` (`ingest.rs:2362`) |
| Reaction, active duplicate | **`false`** | `"duplicate: reaction already exists"` (`ingest.rs:2318`) |
| 9007 channel already exists | **`false`** | `"duplicate: channel already exists"` (`ingest.rs:2120`) |
| Direct handlers (9030–9033, 9040–9044, 1984, 42000, 30350) | `true` | `""` |
| 28936 leave | `true` | `"info: you have left this relay"` (`ingest.rs:1900`) |
| Command kinds, first time | `true` | `"response:{json}"` or `"{}"` (41012) |
| Command kinds, replay | `true` | `"duplicate: already processed"` |

The `accepted=true` + `message="duplicate:"` shape is the **NIP-33 LWW write-conflict
signal**: `buzz-cli` translates it to `CliError::Conflict` → exit code 5
(`crates/buzz-cli/src/commands/mem.rs:105-108`, `commands/notes.rs:560-563`,
`crates/buzz-cli/src/error.rs:103`).

---

### 7. imeta sub-API (`imeta.rs`)

`validate_imeta_tags` allows 13 keys (`imeta.rs:11-14`), of which 12 are singletons
(`imeta.rs:15-18`); `url`, `m`, `x`, `size` are all mandatory (`imeta.rs:163`).
`verify_imeta_blobs` performs 5 storage checks (`imeta.rs:246`, `:252`, `:262`, `:290`,
`:305`). Private helpers: `is_well_formed_mime` `:340`,
`extract_hash_from_media_url` `:351`, `extract_ext_from_media_url` `:362`,
`is_local_media_url` `:373`.

| Reject string | Line |
|---|---|
| `only imeta tags allowed in media_tags` | `imeta.rs:32` |
| `disallowed imeta key: {k}` | `imeta.rs:52` |
| `duplicate imeta key: {k}` | `imeta.rs:55` |
| `imeta url must be a local /media/ path` | `imeta.rs:61` |
| `imeta url must not be a thumbnail path; use thumb field` | `imeta.rs:64` |
| `imeta m must be a valid MIME type` | `imeta.rs:77` |
| `imeta x must be a 64-char lowercase hex SHA-256` | `imeta.rs:86` |
| `imeta size must be a positive integer` | `imeta.rs:94` |
| `imeta thumb must be a local .thumb.jpg path` | `imeta.rs:103` |
| `imeta duration must be a positive finite number` / `… must be a valid float` | `imeta.rs:110`, `:113` |
| `imeta bitrate must be a positive integer` | `imeta.rs:117` |
| `imeta image must be a local /media/ path` | `imeta.rs:122` |
| `imeta image must reference a standalone poster frame, not a thumbnail` | `imeta.rs:126` |
| `imeta image must reference an image file (jpg, png, gif, webp), not video` | `imeta.rs:133` |
| `imeta filename must be 1–255 chars` | `imeta.rs:144` |
| `imeta filename must not contain path separators or control characters` | `imeta.rs:151` |
| `imeta tag must include url, m, x, and size` | `imeta.rs:164` |
| `imeta {key} is only valid for video/mp4, not {m}` | `imeta.rs:172` |
| `imeta url hash does not match x` | `imeta.rs:181` |
| `imeta url extension does not match m` | `imeta.rs:189` |
| `imeta thumb hash does not match x` | `imeta.rs:198` |
| `imeta references nonexistent blob: {x}` | `imeta.rs:249` |
| `imeta blob object missing in storage: {x}` | `imeta.rs:257` |
| `storage error checking blob {x}: {e}` | `imeta.rs:255` |
| `imeta m ({m}) does not match stored MIME ({stored})` | `imeta.rs:262` |
| `imeta size ({n}) does not match stored size ({stored})` | `imeta.rs:268` |
| `imeta duration ({d}) does not match stored duration ({stored})` | `imeta.rs:275` |
| `imeta thumb references missing thumbnail: {x}` | `imeta.rs:293` |
| `imeta image URL has no extractable hash: {url}` | `imeta.rs:306` |
| `imeta image references nonexistent poster: {hash}` | `imeta.rs:311` |
| `imeta image poster MIME must be image type, got {m}` | `imeta.rs:315` |
| `imeta image extension ({url_ext}) does not match stored extension ({sidecar})` | `imeta.rs:322` |
| `imeta image references missing poster frame: {hash}` | `imeta.rs:332` |

All 33 strings reach the client prefixed with `invalid: ` (`ingest.rs:2214`, `:2217`).

---

### 8. Doc/code delta

| Claim | Location | Reality |
|---|---|---|
| "Both WebSocket `["EVENT", ...]` and HTTP `POST /events` feed into `ingest_event` — two doors, one room." | `ingest.rs:3-4` | **Accurate**, with two documented asymmetries: HTTP rejects kinds 1059 and 20001 (`ingest.rs:1449`), and `IngestAuth::Http` has no `channel_ids` field at all. |
| "Command kinds (41010–41012, 30620, 46020, 46030–46031) are processed transactionally: validate → begin tx → insert event → execute mutations → commit." | `command_executor.rs:3-4` | Kind list **matches** `is_command_kind` (`buzz-core/src/kind.rs:667-679`). The atomicity claim is contradicted five lines later by the module's own note that "Domain mutations … execute on the connection pool, NOT inside this transaction" (`command_executor.rs:92-98`). |
| `ARCHITECTURE.md` mentions ingest exactly once | `ARCHITECTURE.md:620` | No ingest pipeline, kind-dispatch table, or side-effect contract is documented anywhere in `ARCHITECTURE.md`. This table has no doc counterpart. |
