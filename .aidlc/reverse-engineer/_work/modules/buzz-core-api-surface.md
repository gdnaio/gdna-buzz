## Module: buzz-core (`crates/buzz-core`)

### Aspect: API Surface

`buzz-core` is a library-only crate (no `main.rs`, no `src/bin/`). Its API is the `pub` surface of `crates/buzz-core/src/lib.rs` plus the 15 public modules it declares. Crate-level lints: `#![deny(unsafe_code)]` and `#![warn(missing_docs)]` (`crates/buzz-core/src/lib.rs:1-2`).

---

### 1. Public modules (`crates/buzz-core/src/lib.rs:9-38`)

| Module | Declared at | Purpose (per doc comment) |
|--------|-------------|---------------------------|
| `agent_turn_metric` | `lib.rs:9` | NIP-AM turn metric payload + encrypt/decrypt |
| `channel` | `lib.rs:11` | channel + membership enums |
| `engram` | `lib.rs:14` | NIP-AE slug grammar, conversation key, d-tag, body parse, envelope, head selection |
| `error` | `lib.rs:16` | relay-side error types |
| `event` | `lib.rs:18` | event wrapper with verification tracking |
| `filter` | `lib.rs:20` | NIP-01 subscription filter matching |
| `git_perms` | `lib.rs:22` | git ref patterns, protection rules, policy evaluation |
| `kind` | `lib.rs:24` | Buzz kind-number registry |
| `network` | `lib.rs:26` | SSRF-safe IP classification |
| `observer` | `lib.rs:28` | agent observer frame helpers |
| `pairing` | `lib.rs:30` | NIP-AB device pairing |
| `presence` | `lib.rs:32` | presence status types |
| `relay` | `lib.rs:34` | canonical relay runtime identities |
| `tenant` | `lib.rs:36` | tenant identity (community key) |
| `verification` | `lib.rs:38` | Schnorr signature + event id verification |

### 2. Crate-root re-exports (`lib.rs:40-45`)

| Re-export | Source |
|-----------|--------|
| `VerificationError` | `error::VerificationError` (`lib.rs:40`) |
| `StoredEvent` | `event::StoredEvent` (`lib.rs:41`) |
| `Event`, `EventId`, `Filter`, `Keys`, `Kind`, `PublicKey` | **re-exported from `nostr`** (`lib.rs:42`) |
| `PresenceStatus` | `presence::PresenceStatus` (`lib.rs:43`) |
| `normalize_host`, `CommunityId`, `TenantContext` | `tenant::…` (`lib.rs:44`) |
| `verify_event` | `verification::verify_event` (`lib.rs:45`) |

Feature-gated module `test_helpers` (`lib.rs:47-74`), active under `#[cfg(any(test, feature = "test-utils"))]`: `make_event(Kind)` (`lib.rs:55`), `make_event_with_keys(&Keys, Kind)` (`lib.rs:64`), `make_stored_event(Kind, Option<Uuid>)` (`lib.rs:72`).

---

### 3. Kind registry — complete constant table

**Verified counts (from `crates/buzz-core/src/kind.rs`):**

| Measure | Actual value | Evidence |
|---------|--------------|----------|
| `pub const … : u32` declarations in `kind.rs` | **134** | `grep "^pub const .*: u32 = "` over `kind.rs` |
| of those, range-boundary constants (not kinds) | **4** | `PARAM_REPLACEABLE_KIND_MIN` `kind.rs:316`, `PARAM_REPLACEABLE_KIND_MAX` `kind.rs:318`, `EPHEMERAL_KIND_MIN` `kind.rs:321`, `EPHEMERAL_KIND_MAX` `kind.rs:323` |
| **actual `KIND_*` / `RELAY_ADMIN_*` kind constants** | **130** | 134 − 4 |
| **entries in `ALL_KINDS`** | **127** | `kind.rs:490-617` (list body) |
| kind constants **absent** from `ALL_KINDS` | **3** | `KIND_AUTH` (22242, `kind.rs:77`), `KIND_NOSTR_IDENTITY_BINDING` (24243, `kind.rs:81`), `KIND_PUSH_LEASE` (30350, `kind.rs:109`) |
| duplicate names in `ALL_KINDS` | 0 | set comparison of the list body |
| duplicate integer values across all 130 constants | 0 | value-frequency check over the constant declarations |

Every kind constant, sorted by integer value. "In `ALL_KINDS`" = whether the name appears in the `ALL_KINDS` slice literal (`kind.rs:490-617`).

| Constant | Value | In `ALL_KINDS` | Declared at |
|---|---|---|---|
| `KIND_PROFILE` | 0 | yes | src/kind.rs:9 |
| `KIND_TEXT_NOTE` | 1 | yes | src/kind.rs:11 |
| `KIND_CONTACT_LIST` | 3 | yes | src/kind.rs:13 |
| `KIND_DELETION` | 5 | yes | src/kind.rs:56 |
| `KIND_REACTION` | 7 | yes | src/kind.rs:58 |
| `KIND_STREAM_MESSAGE` | 9 | yes | src/kind.rs:343 |
| `KIND_CHANNEL_METADATA` | 41 | yes | src/kind.rs:54 |
| `KIND_GIFT_WRAP` | 1059 | yes | src/kind.rs:60 |
| `KIND_FILE_METADATA` | 1063 | yes | src/kind.rs:62 |
| `KIND_GIT_PATCH` | 1617 | yes | src/kind.rs:473 |
| `KIND_GIT_PULL_REQUEST` | 1618 | yes | src/kind.rs:475 |
| `KIND_GIT_PR_UPDATE` | 1619 | yes | src/kind.rs:477 |
| `KIND_GIT_ISSUE` | 1621 | yes | src/kind.rs:479 |
| `KIND_GIT_STATUS_OPEN` | 1630 | yes | src/kind.rs:481 |
| `KIND_GIT_STATUS_MERGED` | 1631 | yes | src/kind.rs:483 |
| `KIND_GIT_STATUS_CLOSED` | 1632 | yes | src/kind.rs:485 |
| `KIND_GIT_STATUS_DRAFT` | 1633 | yes | src/kind.rs:487 |
| `KIND_REPORT` | 1984 | yes | src/kind.rs:191 |
| `KIND_NIP43_MEMBER_ADDED` | 8000 | yes | src/kind.rs:264 |
| `KIND_NIP43_MEMBER_REMOVED` | 8001 | yes | src/kind.rs:266 |
| `KIND_IA_ARCHIVED` | 8002 | yes | src/kind.rs:278 |
| `KIND_IA_UNARCHIVED` | 8003 | yes | src/kind.rs:280 |
| `KIND_NIP29_PUT_USER` | 9000 | yes | src/kind.rs:199 |
| `KIND_NIP29_REMOVE_USER` | 9001 | yes | src/kind.rs:201 |
| `KIND_NIP29_EDIT_METADATA` | 9002 | yes | src/kind.rs:203 |
| `KIND_NIP29_DELETE_EVENT` | 9005 | yes | src/kind.rs:205 |
| `KIND_NIP29_CREATE_GROUP` | 9007 | yes | src/kind.rs:207 |
| `KIND_NIP29_DELETE_GROUP` | 9008 | yes | src/kind.rs:209 |
| `KIND_NIP29_CREATE_INVITE` | 9009 | yes | src/kind.rs:211 |
| `KIND_NIP29_JOIN_REQUEST` | 9021 | yes | src/kind.rs:213 |
| `KIND_NIP29_LEAVE_REQUEST` | 9022 | yes | src/kind.rs:215 |
| `RELAY_ADMIN_ADD_MEMBER` | 9030 | yes | src/kind.rs:253 |
| `RELAY_ADMIN_REMOVE_MEMBER` | 9031 | yes | src/kind.rs:255 |
| `RELAY_ADMIN_CHANGE_ROLE` | 9032 | yes | src/kind.rs:257 |
| `RELAY_ADMIN_SET_WORKSPACE_PROFILE` | 9033 | yes | src/kind.rs:259 |
| `KIND_IA_ARCHIVE_REQUEST` | 9035 | yes | src/kind.rs:272 |
| `KIND_IA_UNARCHIVE_REQUEST` | 9036 | yes | src/kind.rs:274 |
| `KIND_MODERATION_BAN` | 9040 | yes | src/kind.rs:222 |
| `KIND_MODERATION_UNBAN` | 9041 | yes | src/kind.rs:224 |
| `KIND_MODERATION_TIMEOUT` | 9042 | yes | src/kind.rs:227 |
| `KIND_MODERATION_UNTIMEOUT` | 9043 | yes | src/kind.rs:229 |
| `KIND_MODERATION_RESOLVE_REPORT` | 9044 | yes | src/kind.rs:234 |
| `KIND_MUTE_LIST` | 10000 | yes | src/kind.rs:17 |
| `KIND_PIN_LIST` | 10001 | yes | src/kind.rs:22 |
| `KIND_NIP65_RELAY_LIST_METADATA` | 10002 | yes | src/kind.rs:27 |
| `KIND_BOOKMARK_LIST` | 10003 | yes | src/kind.rs:32 |
| `KIND_EMOJI_LIST` | 10030 | yes | src/kind.rs:34 |
| `KIND_AGENT_PROFILE` | 10100 | yes | src/kind.rs:87 |
| `KIND_NIP43_MEMBERSHIP_LIST` | 13534 | yes | src/kind.rs:262 |
| `KIND_IA_ARCHIVED_LIST` | 13535 | yes | src/kind.rs:282 |
| `KIND_PRESENCE_UPDATE` | 20001 | yes | src/kind.rs:327 |
| `KIND_TYPING_INDICATOR` | 20002 | yes | src/kind.rs:331 |
| `KIND_AUTH` | 22242 | **NO** | src/kind.rs:77 |
| `KIND_PAIRING` | 24134 | yes | src/kind.rs:329 |
| `KIND_AGENT_OBSERVER_FRAME` | 24200 | yes | src/kind.rs:333 |
| `KIND_BLOSSOM_AUTH` | 24242 | yes | src/kind.rs:79 |
| `KIND_NOSTR_IDENTITY_BINDING` | 24243 | **NO** | src/kind.rs:81 |
| `KIND_HUDDLE_REACTION` | 24810 | yes | src/kind.rs:336 |
| `KIND_HTTP_AUTH` | 27235 | yes | src/kind.rs:83 |
| `KIND_NIP43_LEAVE_REQUEST` | 28936 | yes | src/kind.rs:268 |
| `KIND_FOLLOW_SET` | 30000 | yes | src/kind.rs:39 |
| `KIND_BOOKMARK_SET` | 30003 | yes | src/kind.rs:43 |
| `KIND_LONG_FORM` | 30023 | yes | src/kind.rs:66 |
| `KIND_EMOJI_SET` | 30030 | yes | src/kind.rs:52 |
| `KIND_READ_STATE` | 30078 | yes | src/kind.rs:75 |
| `KIND_AGENT_ENGRAM` | 30174 | yes | src/kind.rs:94 |
| `KIND_PERSONA` | 30175 | yes | src/kind.rs:165 |
| `KIND_TEAM` | 30176 | yes | src/kind.rs:174 |
| `KIND_MANAGED_AGENT` | 30177 | yes | src/kind.rs:183 |
| `KIND_EVENT_REMINDER` | 30300 | yes | src/kind.rs:102 |
| `KIND_USER_STATUS` | 30315 | yes | src/kind.rs:70 |
| `KIND_PUSH_LEASE` | 30350 | **NO** | src/kind.rs:109 |
| `KIND_GIT_REPO_ANNOUNCEMENT` | 30617 | yes | src/kind.rs:469 |
| `KIND_GIT_REPO_STATE` | 30618 | yes | src/kind.rs:471 |
| `KIND_WORKFLOW_DEF` | 30620 | yes | src/kind.rs:306 |
| `KIND_DM_VISIBILITY` | 30622 | yes | src/kind.rs:313 |
| `KIND_NIP29_GROUP_METADATA` | 39000 | yes | src/kind.rs:286 |
| `KIND_NIP29_GROUP_ADMINS` | 39001 | yes | src/kind.rs:288 |
| `KIND_NIP29_GROUP_MEMBERS` | 39002 | yes | src/kind.rs:290 |
| `KIND_NIP29_GROUP_ROLES` | 39003 | yes | src/kind.rs:292 |
| `KIND_THREAD_SUMMARY` | 39005 | yes | src/kind.rs:299 |
| `KIND_WINDOW_BOUNDS` | 39006 | yes | src/kind.rs:303 |
| `KIND_STREAM_MESSAGE_V2` | 40002 | yes | src/kind.rs:345 |
| `KIND_STREAM_MESSAGE_EDIT` | 40003 | yes | src/kind.rs:347 |
| `KIND_STREAM_MESSAGE_PINNED` | 40004 | yes | src/kind.rs:349 |
| `KIND_STREAM_MESSAGE_BOOKMARKED` | 40005 | yes | src/kind.rs:351 |
| `KIND_STREAM_MESSAGE_SCHEDULED` | 40006 | yes | src/kind.rs:353 |
| `KIND_STREAM_REMINDER` | 40007 | yes | src/kind.rs:355 |
| `KIND_STREAM_MESSAGE_DIFF` | 40008 | yes | src/kind.rs:357 |
| `KIND_SYSTEM_MESSAGE` | 40099 | yes | src/kind.rs:361 |
| `KIND_CANVAS` | 40100 | yes | src/kind.rs:359 |
| `KIND_CHANNEL_SUMMARY` | 40901 | yes | src/kind.rs:365 |
| `KIND_PRESENCE_SNAPSHOT` | 40902 | yes | src/kind.rs:367 |
| `KIND_DM_CREATED` | 41001 | yes | src/kind.rs:377 |
| `KIND_DM_OPEN` | 41010 | yes | src/kind.rs:371 |
| `KIND_DM_ADD_MEMBER` | 41011 | yes | src/kind.rs:373 |
| `KIND_DM_HIDE` | 41012 | yes | src/kind.rs:375 |
| `KIND_PRODUCT_FEEDBACK` | 42000 | yes | src/kind.rs:195 |
| `KIND_JOB_REQUEST` | 43001 | yes | src/kind.rs:382 |
| `KIND_JOB_ACCEPTED` | 43002 | yes | src/kind.rs:384 |
| `KIND_JOB_PROGRESS` | 43003 | yes | src/kind.rs:386 |
| `KIND_JOB_RESULT` | 43004 | yes | src/kind.rs:388 |
| `KIND_JOB_CANCEL` | 43005 | yes | src/kind.rs:390 |
| `KIND_JOB_ERROR` | 43006 | yes | src/kind.rs:392 |
| `KIND_MEMBER_ADDED_NOTIFICATION` | 44100 | yes | src/kind.rs:396 |
| `KIND_MEMBER_REMOVED_NOTIFICATION` | 44101 | yes | src/kind.rs:400 |
| `KIND_AGENT_TURN_METRIC` | 44200 | yes | src/kind.rs:409 |
| `KIND_FORUM_POST` | 45001 | yes | src/kind.rs:414 |
| `KIND_FORUM_VOTE` | 45002 | yes | src/kind.rs:416 |
| `KIND_FORUM_COMMENT` | 45003 | yes | src/kind.rs:418 |
| `KIND_WORKFLOW_TRIGGERED` | 46001 | yes | src/kind.rs:428 |
| `KIND_WORKFLOW_STEP_STARTED` | 46002 | yes | src/kind.rs:430 |
| `KIND_WORKFLOW_STEP_COMPLETED` | 46003 | yes | src/kind.rs:432 |
| `KIND_WORKFLOW_STEP_FAILED` | 46004 | yes | src/kind.rs:434 |
| `KIND_WORKFLOW_COMPLETED` | 46005 | yes | src/kind.rs:436 |
| `KIND_WORKFLOW_FAILED` | 46006 | yes | src/kind.rs:438 |
| `KIND_WORKFLOW_CANCELLED` | 46007 | yes | src/kind.rs:440 |
| `KIND_WORKFLOW_APPROVAL_REQUESTED` | 46010 | yes | src/kind.rs:442 |
| `KIND_WORKFLOW_APPROVAL_GRANTED` | 46011 | yes | src/kind.rs:444 |
| `KIND_WORKFLOW_APPROVAL_DENIED` | 46012 | yes | src/kind.rs:446 |
| `KIND_WORKFLOW_TRIGGER` | 46020 | yes | src/kind.rs:422 |
| `KIND_APPROVAL_GRANT` | 46030 | yes | src/kind.rs:424 |
| `KIND_APPROVAL_DENY` | 46031 | yes | src/kind.rs:426 |
| `KIND_AUDIT_ENTRY` | 48001 | yes | src/kind.rs:452 |
| `KIND_HUDDLE_STARTED` | 48100 | yes | src/kind.rs:454 |
| `KIND_HUDDLE_PARTICIPANT_JOINED` | 48101 | yes | src/kind.rs:456 |
| `KIND_HUDDLE_PARTICIPANT_LEFT` | 48102 | yes | src/kind.rs:458 |
| `KIND_HUDDLE_ENDED` | 48103 | yes | src/kind.rs:460 |
| `KIND_HUDDLE_GUIDELINES` | 48106 | yes | src/kind.rs:462 |
| `KIND_MEDIA_UPLOAD` | 49001 | yes | src/kind.rs:466 |

#### Range-boundary constants

| Constant | Value | Declared at |
|---|---|---|
| `PARAM_REPLACEABLE_KIND_MIN` | 30000 | src/kind.rs:316 |
| `PARAM_REPLACEABLE_KIND_MAX` | 39999 | src/kind.rs:318 |
| `EPHEMERAL_KIND_MIN` | 20000 | src/kind.rs:321 |
| `EPHEMERAL_KIND_MAX` | 29999 | src/kind.rs:323 |

#### Kind-set constants (`&[u32]` slices)

| Constant | Members | Declared at |
|---|---|---|
| `AUTHOR_ONLY_KINDS` | `KIND_EVENT_REMINDER` (30300), `KIND_PUSH_LEASE` (30350) | src/kind.rs:120 |
| `RESULT_GATED_KINDS` | `KIND_DM_VISIBILITY` (30622), `KIND_AGENT_TURN_METRIC` (44200) | src/kind.rs:129 |
| `P_GATED_KINDS` | `KIND_AGENT_OBSERVER_FRAME` (24200), `KIND_MEMBER_ADDED_NOTIFICATION` (44100), `KIND_MEMBER_REMOVED_NOTIFICATION` (44101), `KIND_GIFT_WRAP` (1059), `KIND_DM_VISIBILITY` (30622), `KIND_AGENT_TURN_METRIC` (44200) — 6 entries | src/kind.rs:146-156 |
| `ALL_KINDS` | 127 entries (see table above) | src/kind.rs:490-617 |

#### Classification / helper functions (`kind.rs`)

| Function | Signature | Rule as coded | file:line |
|---|---|---|---|
| `is_moderation_command_kind` | `const fn(u32) -> bool` | matches 9040, 9041, 9042, 9043, 9044 | `kind.rs:240-249` |
| `is_ephemeral` | `const fn(u32) -> bool` | `kind >= 20000 && kind <= 29999` | `kind.rs:621-623` |
| `is_replaceable` | `const fn(u32) -> bool` | `matches!(kind, 0 \| 3 \| 41 \| 10000..=19999)` | `kind.rs:628-630` |
| `is_parameterized_replaceable` | `const fn(u32) -> bool` | `kind >= 30000 && kind <= 39999` | `kind.rs:635-637` |
| `is_workflow_execution_kind` | `const fn(u32) -> bool` | `46001 ..= 46012` (`KIND_WORKFLOW_TRIGGERED..=KIND_WORKFLOW_APPROVAL_DENIED`) | `kind.rs:641-643` |
| `is_relay_admin_kind` | `const fn(u32) -> bool` | matches 9030, 9031, 9032, 9033 | `kind.rs:647-656` |
| `is_identity_archive_request_kind` | `const fn(u32) -> bool` | matches 9035, 9036 only (relay-signed 8002/8003/13535 excluded per doc `kind.rs:658-661`) | `kind.rs:662-664` |
| `is_command_kind` | `const fn(u32) -> bool` | matches `KIND_WORKFLOW_DEF` 30620, `KIND_DM_OPEN` 41010, `KIND_DM_ADD_MEMBER` 41011, `KIND_DM_HIDE` 41012, `KIND_WORKFLOW_TRIGGER` 46020, `KIND_APPROVAL_GRANT` 46030, `KIND_APPROVAL_DENY` 46031 | `kind.rs:667-679` |
| `is_relay_only_kind` | `const fn(u32) -> bool` | matches 13534, 40901, 40902, 30622, 39005, 39006 | `kind.rs:682-693` |
| `event_kind_u32` | `fn(&nostr::Event) -> u32` | `event.kind.as_u16() as u32` | `kind.rs:696-698` |
| `event_kind_i32` | `fn(&nostr::Event) -> i32` | `event.kind.as_u16() as i32` | `kind.rs:702-704` |

Compile-time assertions in the module body (not functions, but part of the contract): `kind.rs:707-744` — range membership for `KIND_AGENT_PROFILE`, `KIND_PERSONA`, `KIND_TEAM`, `KIND_MANAGED_AGENT`, `KIND_WORKFLOW_DEF`, `KIND_EVENT_REMINDER`, `KIND_DM_VISIBILITY`, `KIND_THREAD_SUMMARY`, `KIND_WINDOW_BOUNDS`, the two NIP-34 addressable kinds, u16 fits for `KIND_AUTH`/`KIND_CANVAS`/`KIND_HUDDLE_GUIDELINES`/`KIND_AGENT_TURN_METRIC`/`KIND_REPORT`/`KIND_MODERATION_RESOLVE_REPORT`, and the moderation-command classification.

---

### 4. Verification + event API

| Item | Signature | file:line |
|---|---|---|
| `verify_event` | `fn(&Event) -> Result<(), VerificationError>` | `src/verification.rs:11-32` |
| `StoredEvent::new` | `fn(nostr::Event, Option<Uuid>) -> Self` | `src/event.rs:23-30` |
| `StoredEvent::with_received_at` | `fn(nostr::Event, DateTime<Utc>, Option<Uuid>, bool) -> Self` | `src/event.rs:38-48` |
| `StoredEvent::is_verified` | `fn(&self) -> bool` | `src/event.rs:33-35` |

### 5. Filter API (`src/filter.rs`)

| Item | Signature | file:line |
|---|---|---|
| `filters_match` | `fn(&[Filter], &StoredEvent) -> bool` | `filter.rs:10-13` |
| `reader_authorized_for_event` | `fn(&nostr::Event, &str) -> bool` | `filter.rs:23-33` |
| `filter_match_one` | private | `filter.rs:35-104` |

### 6. Network API (`src/network.rs`)

`is_private_ip(&std::net::IpAddr) -> bool` (`network.rs:25-53`) — the crate's only network helper; no other public items.

### 7. Tenant API (`src/tenant.rs`)

| Item | Signature | file:line |
|---|---|---|
| `CommunityId::from_uuid` | `const fn(Uuid) -> Self` | `tenant.rs:45-47` |
| `CommunityId::as_uuid` | `const fn(&self) -> &Uuid` | `tenant.rs:50-52` |
| `TenantContext::resolved` | `fn(CommunityId, impl Into<String>) -> Self` | `tenant.rs:79-84` |
| `TenantContext::community` | `const fn(&self) -> CommunityId` | `tenant.rs:87-89` |
| `TenantContext::host` | `fn(&self) -> &str` | `tenant.rs:95-97` |
| `normalize_host` | `#[must_use] fn(&str) -> String` | `tenant.rs:121-139` |
| `relay_url_authority` | `#[must_use] fn(&str) -> String` | `tenant.rs:156-172` |

### 8. Relay identity API (`src/relay.rs`)

`normalize_relay_url(&str) -> Result<String, NormalizeRelayUrlError>` (`relay.rs:37-78`). Doc explicitly states it is **not** the NIP-42 AUTH comparison helper (that lives in `buzz-auth`) — `relay.rs:28-32`.

### 9. Channel API (`src/channel.rs`)

`canonical_channel_name(&str) -> &str` (`channel.rs:15-19`); `ChannelVisibility`, `ChannelType`, `MemberRole` with `as_str`, `Display`, `FromStr` on each, plus `MemberRole::is_elevated` (`:134-136`), `permission_level` (`:142-150`), `has_at_least` (`:155-157`).

### 10. Presence API (`src/presence.rs`)

`PresenceStatus` + `as_str(&self) -> &'static str` (`presence.rs:22-28`) + `Display` (`:31-35`).

### 11. Observer API (`src/observer.rs`)

| Item | Value / signature | file:line |
|---|---|---|
| `OBSERVER_AGENT_TAG` | `"agent"` | `observer.rs:13` |
| `OBSERVER_FRAME_TAG` | `"frame"` | `observer.rs:15` |
| `OBSERVER_FRAME_TELEMETRY` | `"telemetry"` | `observer.rs:17` |
| `OBSERVER_FRAME_CONTROL` | `"control"` | `observer.rs:19` |
| `NIP44_MIN_CONTENT_LEN` | `132` | `observer.rs:21` |
| `NIP44_MAX_CONTENT_LEN` | `87_472` | `observer.rs:23` |
| `OBSERVER_MAX_PLAINTEXT_LEN` | `65_535` | `observer.rs:25` |
| `content_looks_like_nip44` | `fn(&str) -> bool` | `observer.rs:53-55` |
| `encrypt_observer_payload` | `fn<T: Serialize>(&Keys, &PublicKey, &T) -> Result<String, ObserverPayloadError>` | `observer.rs:58-81` |
| `decrypt_observer_payload` | `fn<T: DeserializeOwned>(&Keys, &Event) -> Result<T, ObserverPayloadError>` | `observer.rs:84-110` |

### 12. Agent turn metric API (`src/agent_turn_metric.rs`)

| Item | Signature | file:line |
|---|---|---|
| `AgentTurnMetricError` | alias of `ObserverPayloadError` | `agent_turn_metric.rs:15` |
| `AgentTurnMetricPayload::validate` | `fn(&self) -> Result<(), ObserverPayloadError>` | `:140-169` |
| `encrypt_agent_turn_metric` | `fn(&Keys, &PublicKey, &AgentTurnMetricPayload) -> Result<String, ObserverPayloadError>` | `:169-176` |
| `decrypt_agent_turn_metric` | `fn(&Keys, &Event) -> Result<AgentTurnMetricPayload, ObserverPayloadError>` | `:185-191` |

### 13. Engram API (`src/engram.rs`)

| Item | Value / signature | file:line |
|---|---|---|
| `CORE_SLUG` | `"core"` | `engram.rs:20` |
| `D_TAG_DOMAIN` | `b"agent-memory/v1/d-tag"` | `engram.rs:24` |
| `NIP44_PLAINTEXT_MAX` | `65_535` | `engram.rs:28` |
| `SLUG_MAX_LEN` | `255` | `engram.rs:31` |
| `validate_slug` | `fn(&str) -> Result<(), EngramError>` | `:67-92` |
| `normalize_slug` | `fn(&str) -> Result<String, EngramError>` | `:123-131` |
| `conversation_key` | `fn(&SecretKey, &PublicKey) -> ConversationKey` | `:136-138` |
| `d_tag` | `fn(&ConversationKey, &str) -> String` | `:144-155` |
| `Body::slug` / `is_tombstone` / `to_json_bytes` / `from_json_bytes` | see data-model doc | `:175`, `:183`, `:189`, `:216` |
| `extract_refs` | `fn(&str) -> Vec<String>` | `:384-430` |
| `build_event` | `fn(&Keys, &PublicKey, &Body, u64) -> Result<Event, EngramError>` | `:435-476` |
| `validate_and_decrypt` | `fn(&Event, &PublicKey, &PublicKey, &SecretKey, &PublicKey) -> Result<Body, EngramError>` | `:488-558` |
| `select_head` | `fn<I: IntoIterator<Item = Event>>(I) -> Option<Event>` | `:564-583` |
| `monotonic_created_at` | `fn(u64, Option<u64>) -> u64` | `:588-593` |
| `Listing` | struct | `:598-603` |

Private helpers (not exported): `validate_segment` (`:94-112`), `is_lower_alnum` (`:114-116`), `write_json_string` (`:263-281`), `parse_strict_json` (`:283-380`).

### 14. Git permissions API (`src/git_perms.rs`)

| Item | Value / signature | file:line |
|---|---|---|
| `MAX_PROTECTION_RULES` | `50` | `git_perms.rs:19` |
| `MAX_PATTERN_LENGTH` | `256` | `git_perms.rs:21` |
| `MAX_WILDCARDS_PER_PATTERN` | `3` | `git_perms.rs:23` |
| `RefPattern::parse` | `fn(&str) -> Result<Self, PatternError>` | `:83-146` |
| `RefPattern::matches` | `fn(&self, &str) -> bool` | `:147-178` |
| `RefPattern::as_str` | `fn(&self) -> &str` | `:181-183` |
| `UpdateKind::classify` | `fn(&str, &str, bool) -> Self` | `:212-221` |
| `parse_protection_tag` | `fn(&[&str]) -> Result<ProtectionRule, RuleParseError>` | `:297-300` |
| `parse_protection_tag_with_warnings` | `fn(&[&str]) -> Result<(ProtectionRule, Vec<String>), RuleParseError>` | `:303-362` |
| `parse_protection_tags` | `fn(&[Vec<String>]) -> Result<ParsedProtection, RuleParseError>` | `:379-399` |
| `default_min_role` | `fn(&str, UpdateKind) -> MemberRole` | `:403-425` |
| `EffectiveRules::for_ref` | `fn(&str, &[ProtectionRule]) -> Self` | `:447-475` |
| `evaluate_ref_update` | `fn(&RefUpdate, MemberRole, &[ProtectionRule]) -> Result<(), Denial>` | `:508-578` |
| `evaluate_push` | `fn(&[RefUpdate], MemberRole, &[ProtectionRule]) -> Result<(), Vec<Denial>>` | `:584-597` |

### 15. Pairing API (`src/pairing/`)

Re-exports from `pairing/mod.rs:27-29`: `QrPayload`, `PairingSession`, `Role`, `SessionState`, `AbortReason`, `PairingMessage`, `PayloadType`; plus `PairingError` (`mod.rs:35-71`) and submodules `crypto`, `qr`, `session`, `types` (`mod.rs:22-25`).

`pairing::crypto` (all pure):

| Function | Signature | file:line |
|---|---|---|
| `derive_session_id` | `fn(&[u8; 32]) -> [u8; 32]` | `crypto.rs:54-56` |
| `derive_sas` | `fn(&[u8; 32], &[u8; 32]) -> (u32, [u8; 32])` | `crypto.rs:70-75` |
| `derive_transcript_hash` | `fn(&[u8;32], &[u8;32], &[u8;32], &[u8;32], &[u8;32]) -> [u8;32]` | `crypto.rs:89-105` |
| `format_sas` | `fn(u32) -> String` (zero-padded 6 digits; doctest at `:110-114`) | `crypto.rs:116-118` |
| `ct_eq` | `fn(&[u8;32], &[u8;32]) -> bool` (constant-time) | `crypto.rs:126-129` |

`pairing::qr`: `encode_qr(&QrPayload) -> String` (`qr.rs:79-93`, doctest `:66-77`), `decode_qr(&str) -> Result<QrPayload, PairingError>` (`qr.rs:104-206`).

`pairing::session::PairingSession` — 22 public methods:

| Method | Role | Signature | file:line |
|---|---|---|---|
| `new_source` | Source | `fn(String) -> (Self, QrPayload)` | `session.rs:112-146` |
| `handle_offer` | Source | `fn(&mut self, &Event) -> Result<String, PairingError>` | `:149-197` |
| `confirm_sas` | Source | `fn(&mut self) -> Result<Event, PairingError>` | `:200-224` |
| `send_payload` | Source | `fn(&mut self, PayloadType, Zeroizing<String>) -> Result<Event, PairingError>` | `:227-251` |
| `handle_complete` | Source | `fn(&mut self, &Event) -> Result<(), PairingError>` | `:254-281` |
| `new_target` | Target | `fn(&QrPayload) -> Result<(Self, Event), PairingError>` | `:286-326` |
| `handle_sas_confirm` | Target | `fn(&mut self, &Event) -> Result<String, PairingError>` | `:329-373` |
| `confirm_target_sas` | Target | `fn(&mut self) -> Result<(), PairingError>` | `:376-382` |
| `handle_payload` | Target | `fn(&mut self, &Event) -> Result<(PayloadType, Zeroizing<String>), PairingError>` | `:388-409` |
| `send_complete` | Target | `fn(&mut self) -> Result<Event, PairingError>` | `:412-421` |
| `abort` | both | `fn(&mut self, AbortReason) -> Result<Option<Event>, PairingError>` | `:430-445` |
| `handle_abort` | both | `fn(&mut self, &Event) -> Result<AbortReason, PairingError>` | `:448-473` |
| `is_expired` / `state` / `role` / `pubkey` / `relay_urls` / `sas_code` / `sign_event` / `qr_uri` | accessors | — | `:476`, `:481`, `:486`, `:491`, `:496`, `:501`, `:510`, `:517` |

Test-only methods in a `#[cfg(test)] impl PairingSession` block (`session.rs:530-544`): `has_processed` (`:536`), `set_timeout` (`:541`) — not part of the published API. The type is split across four `impl` blocks: source-side (`:109`), target-side (`:282`), shared/accessors (`:424`), and private helpers (`:546`).

---

### 16. ARCHITECTURE.md verification

| Doc claim | Location | Actual code | Verdict |
|---|---|---|---|
| "`buzz-core` defines all 81 kinds as `pub const KIND_*: u32`" | `ARCHITECTURE.md:142` | 130 kind constants (134 `u32` consts − 4 range bounds) in `crates/buzz-core/src/kind.rs` | **stale** — undercounts by 49 |
| "`pub const ALL_KINDS: &[u32]  // 80 entries (KIND_AUTH excluded — never stored)`" | `ARCHITECTURE.md:346` | `ALL_KINDS` has **127** entries (`kind.rs:490-617`) | **stale** count |
| "KIND_AUTH excluded" | `ARCHITECTURE.md:346` | Correct: `KIND_AUTH` (22242, `kind.rs:77`) is **not** in `ALL_KINDS` | **accurate**, but incomplete — `KIND_NOSTR_IDENTITY_BINDING` (24243, `kind.rs:81`) and `KIND_PUSH_LEASE` (30350, `kind.rs:109`) are also excluded and are not mentioned |

No code comment in `kind.rs` explains why `KIND_NOSTR_IDENTITY_BINDING` or `KIND_PUSH_LEASE` are omitted from `ALL_KINDS`; their doc comments say "ephemeral, not stored" (`kind.rs:80`) and "parameterized replaceable, author-only" (`kind.rs:104-108`) respectively. Note that `KIND_BLOSSOM_AUTH` (24242) and `KIND_HTTP_AUTH` (27235) carry similar "not stored" doc comments (`kind.rs:78`, `:82`) yet **are** present in `ALL_KINDS` (`kind.rs:550`, `:553`), so "never stored" is not the discriminating rule in the code as written.
