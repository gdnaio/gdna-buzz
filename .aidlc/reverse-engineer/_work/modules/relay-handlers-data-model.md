## Module: buzz-relay — moderation, admin & background workers (`crates/buzz-relay/src`)
### Aspect: Data Model

Scope: 12 files, 6,720 LOC. None of these files own DDL; every table lives in `migrations/` and is reached through `buzz-db`. What follows is the *in-memory* model each handler builds plus the durable row shape it writes.

---

#### 1. Moderation state model

##### 1.1 Restriction state (bans + timeouts share one row)

`community_bans` is a single row per `(community_id, pubkey)` carrying **both** ban and timeout state — they are not separate tables.

| Field | Type | Source |
|---|---|---|
| `pubkey` | `Vec<u8>` (raw 32 bytes) | `buzz-db/src/moderation.rs:85` |
| `banned` | `bool` | `moderation.rs:87` |
| `ban_expires_at` | `Option<DateTime<Utc>>`; `None` while `banned` ⇒ **permanent** | `moderation.rs:89` |
| `ban_reason` | `Option<String>` (private) | `moderation.rs:91` |
| `muted_until` | `Option<DateTime<Utc>>`; `None` or past ⇒ not timed out | `moderation.rs:93` |
| `mute_reason` | `Option<String>` (private) | `moderation.rs:95` |
| `actor_pubkey` | `Vec<u8>` — last moderator to touch the row | `moderation.rs:97` |
| `updated_at` | `DateTime<Utc>` | `moderation.rs:99` |

The handler-facing projection is deliberately narrower — `RestrictionState { banned: bool, muted_until: Option<DateTime<Utc>> }` (`buzz-db/src/moderation.rs:432`), read at `moderation_commands.rs:103-108` and consumed by `ensure_actor_not_banned` (`moderation_commands.rs:135-142`). **`ban_expires_at` is not in the projection**, so the handler's ban check is on the raw `banned` boolean only; expiry interpretation is the DB layer's job.

Writes from this module:

| Kind | DB call | Site |
|---|---|---|
| 9040 | `ban_community_member(community, target, actor, reason, expires_at)` | `moderation_commands.rs:169-176` |
| 9041 | `unban_community_member(...) -> bool` | `moderation_commands.rs:248` |
| 9042 | `timeout_community_member(community, target, actor, muted_until, reason)` | `moderation_commands.rs:287-294` |
| 9043 | `untimeout_community_member(...) -> bool` | `moderation_commands.rs:351` |

`expires_at` for a ban is parsed from an optional `["expiration", <unix secs>]` tag (`extract_expiration`, `moderation_commands.rs:592-606`) — `Ok(None)` when absent ⇒ permanent (`moderation_commands.rs:148`). For a timeout the same tag is **required** (`moderation_commands.rs:270-271`), giving `muted_until`.

##### 1.2 Report model (`moderation_reports`)

Insert shape `NewReport` (`buzz-db/src/moderation.rs:37-51`), built at `report.rs:79-90`:

| Field | Constraint / provenance |
|---|---|
| `report_event_id: &[u8]` | signed kind:1984 event id; **idempotency key per community** (`moderation.rs:39`, `:170-171`) |
| `reporter_pubkey: &[u8]` | `event.pubkey.to_bytes()` (`report.rs:45`) |
| `target: ReportTarget` | exactly one of `Event(Vec<u8>)` / `Pubkey(Vec<u8>)` / `Blob(Vec<u8>)` (`moderation.rs:26-34`) |
| `channel_id: Option<Uuid>` | **only** inferred from an in-tenant target event row (`report.rs:60`); `None` for `Blob` (`report.rs:72`) and `Pubkey` (`report.rs:74`) targets |
| `report_type: &str` | one of 7 values in `REPORT_TYPES` (`report.rs:29-37`) |
| `note: Option<&str>` | `event.content` when non-empty (`report.rs:222-228`) |

Read-back `ReportRecord` (`moderation.rs:54-81`) adds a `status` lifecycle of **four** values documented as `open | resolved | dismissed | escalated` (`moderation.rs:71`), plus `resolved_by`, `resolved_at`, `action_id`.

> **Delta:** the 9044 handler only ever writes `resolved` or `dismissed` (`moderation_commands.rs:380-385`). `escalated` is a documented status with **no producer** — an `action=escalate` resolution still stores `status=resolved` and merely audits the action string as `escalate` (`moderation_commands.rs:503-505`). The `ModerationNotice::body` renderer has a dead `"escalated"` arm (`moderation_notices.rs:281`) that no production status can reach.

Parse-time intermediate (not persisted): `ParsedReport` / `ParsedReportTarget` (`report.rs:96-119`). Both `Event` and `Blob` variants carry an `author_pubkey: Vec<u8>` field explicitly annotated "Validation-shape only" (`report.rs:106-107`, `:112-113`) — it is **never inserted**; the destructuring at `report.rs:54` and `:65` discards it with `..`. Dead field kept for NIP-56 shape validation.

##### 1.3 Moderation audit row (`moderation_actions`)

`NewAction` has 9 fields (`buzz-db/src/moderation.rs:121-141`). The single production writer (`moderation_commands.rs:529-543`) populates only **5** of them:

| Field | Populated? | Site |
|---|---|---|
| `actor_pubkey` | yes | `moderation_commands.rs:532` |
| `action` | yes | `:533` |
| `target_pubkey` | yes (Option) | `:534` |
| `target_event_id` | yes (Option) | `:535` |
| `public_reason` | yes (Option) | `:538` |
| `channel_id` | **always `None`** | `:536` |
| `reason_code` | **always `None`** | `:537` |
| `private_reason` | **always `None`** | `:539` |
| `matched_principal` | **always `None`** | `:540` |

`matched_principal` is documented as recording the NIP-OA principal an enforcement check matched (`moderation.rs:139`) and is surfaced verbatim by the mod-audit read API (`api/bridge.rs:2166`) — it is therefore **always `null` in production**. Confirmed: `insert_moderation_action` has exactly one caller (`moderation_commands.rs:529`).

Action vocabulary is DB-CHECK-enforced with 12 values (`MODERATION_ACTION_CHECK_VOCAB`, `moderation.rs:104-117`). The mapping function `resolution_audit_action` (`moderation_commands.rs:502-513`) covers 6 of them; direct enforcement rows cover 4 more (`ban`/`unban`/`timeout`/`untimeout`, `moderation_commands.rs:180`, `:256`, `:298`, `:358`).

| Vocab value | Written in production? | Site |
|---|---|---|
| `ban`, `unban`, `timeout`, `untimeout` | yes | `moderation_commands.rs:180/256/298/358` |
| `dismiss_report`, `escalate`, `resolve:delete`, `resolve:kick`, `resolve:ban`, `resolve:timeout` | yes | `moderation_commands.rs:503-508` |
| **`delete_message`** | **no production writer** | — |
| **`kick`** | **no production writer** | — |

`resolution_audit_action` also has an unreachable `"resolve:unknown"` fallback (`moderation_commands.rs:511`) that is **not** in the CHECK vocabulary — if ever reached it would raise a constraint violation surfaced as `error: failed to write audit row: …` (`moderation_commands.rs:544`). It is guarded by the caller's own vocabulary validation at `moderation_commands.rs:386-392`.

##### 1.4 Authorization value model

`moderation_authz.rs` defines three enums, none persisted:

| Type | Variants | Production use |
|---|---|---|
| `ModerationAction` (`:30-48`) | 8: `DeleteMessage`, `Kick`, `Ban`, `Unban`, `Timeout`, `Untimeout`, `ResolveReport`, `ViewQueue` | **6 used**; `DeleteMessage` + `Kick` have ZERO production call sites |
| `ModerationTarget<'a>` (`:51-58`) | `Event(&[u8])`, `Pubkey(&[u8])`, `None` | all 3 constructed (`moderation_commands.rs:161/240/279/343/404`, `api/bridge.rs:2042`) |
| `ModerationAuthority` (`:62-69`) | `CommunityOwner`, `CommunityAdmin`, `ChannelRole` | `ChannelRole` is **unreachable in production** — see api-surface |

`ModerationAuthority` is returned to every caller and **discarded by all of them** (`moderation_commands.rs:165`, `api/bridge.rs:2044`). The doc comment claims it is "recorded in the audit row" (`moderation_authz.rs:61`) — it is not; `insert_audit` takes no authority parameter (`moderation_commands.rs:518-527`).

##### 1.5 Moderation notice model

`ModerationNotice` (`moderation_notices.rs:42-72`) — 3 variants, 2 with production constructors:

| Variant | Fields | Producer |
|---|---|---|
| `ReportResolved { report_id, status, summary }` | `:44-57` | `moderation_commands.rs:485-489` |
| `Restriction { action_id, kind, public_reason }` | `:60-68` | `moderation_commands.rs:208-212` (ban), `:313-317` (timeout) |
| **`ContentActioned { action_id, public_reason }`** | `:53-57` | **none** — zero production constructors |

Durable representation: a relay-signed **kind:9** (`KIND_STREAM_MESSAGE`) inserted into a two-party DM channel with exactly two tags (`moderation_notices.rs:160-163`):
- `["h", <dm_channel_id uuid>]`
- `["moderation_source", <source row UUID>]` — deliberately not `e`, because the value is an opaque DB row UUID not a 32-byte event id (`moderation_notices.rs:35-38`).

The idempotency key is `ModerationNotice::source_id()` (`moderation_notices.rs:259-265`) — the report row id for `ReportResolved`, the audit action id otherwise. Supporting rows: a DM channel from `open_dm` (participant-hash idempotent, `moderation_notices.rs:100-107`), a relay kind:0 profile named `"{host} Moderation"` (`moderation_notices.rs:189-194`), and kind:39000 discovery via `emit_group_discovery_events` (`moderation_notices.rs:155`).

---

#### 2. Relay-member model

`RelayMember` (`buzz-db/src/relay_members.rs:17`) keyed on `(community_id, pubkey)` where **pubkey is a 64-char hex string, not bytes** — unlike `community_bans`/`moderation_actions` which store raw bytes. Both representations coexist in the same request: `moderation_authz.rs:98` hex-encodes the actor for `get_relay_member` while `moderation_commands.rs:532` passes the same actor as raw bytes to `insert_moderation_action`.

Role is an untyped `String` compared against string literals throughout: `"owner"` / `"admin"` / `"member"` (`moderation_authz.rs:149-179`, `relay_admin.rs:148/177/185-191/227/286/301-304`, `identity_archive.rs:245`). There is **no Rust enum** for the role. `relay_admin.rs:142` maps a missing member row to the empty string `""`, so "not a member" and "role is empty" are indistinguishable downstream.

`RemoveResult` (`relay_members.rs:180`) is a 4-variant outcome enum consumed at `relay_admin.rs:256-266`: `Removed`, `IsOwner`, `NotFound`, `RoleMismatch`.

Community icon is a separate scalar on the community row, written by kind:9033 through `set_community_icon(community, Option<&str>)` (`relay_admin.rs:157-161`) — empty string ⇒ `None` ⇒ clear.

---

#### 3. Community-provisioning model

Request/response DTOs (`community_provisioning.rs:43-68`):

| `ProvisionCommunityRequest` | Type | Default |
|---|---|---|
| `host` | `String` | required |
| `initial_owner_pubkey` | `Option<String>` | `None` (`:49`) |
| `create_only` | `bool` | `false` (`:53`) |

| `ProvisionCommunityResponse` | Type | Notes |
|---|---|---|
| `community_id` | `String` (UUID) | `:60` |
| `host` | `String` | canonical stored host, `:62` |
| `status` | `&'static str` | `"created"` or `"existed"` (`:65`, set at `:310`/`:346`) |
| `owner_pubkey` | `Option<String>` | omitted when `None` (`:67`) |

Host constraint: `MAX_HOST_LEN = 255` matching `communities.host VARCHAR(255)` (`community_provisioning.rs:39`), plus a canonical-authority round-trip through `url::Url` (`:102-148`) and DNS label rules (max 253 total, max 63 per label, `:150-172`).

Outcome enum from the create-only path: `buzz_db::CreateCommunityWithOwnerResult::{Created, HostExists, LimitReached}` (`community_provisioning.rs:290-299`).

---

#### 4. Push-lease and endpoint-state model

##### 4.1 Wire model (`push_lease.rs`)

Public envelope, from the signed event's public tags (`push_lease.rs:24-29`):

| `LeaseEnvelope` | Source tag | Constraint |
|---|---|---|
| `installation_id: String` | `d` | 1..=64 bytes (`:139-141`) |
| `expiration: i64` | `expiration` | `> now - 120`, `<= now + 30 days` (`:143-150`, TTL const `:475`) |
| `executor_key_id: String` | `exec` | non-empty (`:151-154`); must equal `config.push_executor_key_id` (`:484-486`) |

Only 4 public tag names are permitted at all — `d`, `expiration`, `exec`, `alt` — and any other tag is a hard reject (`push_lease.rs:112-126`). `alt` is accepted-and-discarded.

Encrypted plaintext (`LeasePlaintext`, `push_lease.rs:32-41`) is `#[serde(deny_unknown_fields)]` **and** additionally key-checked against an explicit required/optional list that varies by `active` (`push_lease.rs:159-190`):

| Field | Active lease | Inactive lease |
|---|---|---|
| `v: u64` | required, must be `1` (`:195`) | required |
| `origin: String` | required, must equal `wss?://{tenant.host}` (`:200`, built at `:585-596`) | required |
| `generation: u64` | required, `1..=2^53-1` (`:198`, const `:21`) | required |
| `active: bool` | `true` | `false` |
| `app_profile`, `transport`, `endpoint`, `subscriptions` | required | **must be absent** (`:210-218`) |

Nested `Subscription` (`push_lease.rs:45-51`): `filter: Map<String,Value>`, `class: String`, `ignore: Vec<Map>` (default empty), `suppress: Option<Suppress>`. `Suppress` is a single `p_tags_max: u64` that must be `> 0` (`:55-58`, `:256-258`).

Filter members are restricted to exactly 5 keys — `kinds`, `authors`, `#p`, `#h`, `#e` (`push_lease.rs:264`). `kinds` is mandatory; `#h` values must be canonical **v4** UUIDs (`:305-311`); `#p` values must equal the lease author (`:300-303`); `authors`/`#e` must be exact lowercase 64-hex (`:365-374`).

Parsing rejects duplicate object keys at **every** nesting depth via a custom `DeserializeSeed` (`NoDuplicates` / `NoDuplicatesVisitor`, `push_lease.rs:383-455`) — `serde_json` otherwise last-write-wins.

Policy limits are hard-coded at the call site, not configurable (`push_lease.rs:503-522`): 16 subscriptions, 16 kinds, 20 authors, 50 `#h`, 20 generic tag values, 8 ignore filters, 4096-byte endpoint, 512-byte strings, 16 active leases per author (`:479`), 64 KiB ciphertext (`:477`), 32 KiB plaintext (`:478`).

##### 4.2 Durable model (`buzz-db/src/push.rs`)

| Struct | Purpose | Fields of note |
|---|---|---|
| `LeaseVersion<'a>` (`push.rs:74-85`) | NIP-01 addressable ordering + generation watermark | `source_event_id`, `source_created_at`, `generation`, `expires_at` |
| `ActiveLease<'a>` (`push.rs:87-100`) | effective push state | `app_profile`, `endpoint_hash` (SHA-256 of endpoint, computed at `push_lease.rs:535`), `endpoint_grant` (the plaintext endpoint itself, `push_lease.rs:544`), `max_class`, `subscriptions` |
| `MatchLease` (`push.rs:175-188`) | matcher input | `author` (raw bytes), `installation_id`, `generation`, `subscriptions: Value`, `expires_at: i64` |
| `WakeRequest` (`push.rs:562`) | matcher output | built at `push_runtime.rs:277-284` |
| `ClaimedWake` (`push.rs:137-164`) | one exclusively-claimed delivery job | 12 fields incl. `claim_id` fencing token and `attempt: i32` |

**Note the naming/semantics mismatch:** `ActiveLease.endpoint_grant` is documented as "Opaque endpoint grant issued by the stateless gateway" (`push.rs:94`) but the relay stores the client-supplied `endpoint` string verbatim into it (`push_lease.rs:544` `capability = endpoint.to_owned()`; `:555`). The relay never mints or validates a grant — it is a pass-through of the plaintext APNs token.

Outcome enums: `AcceptLeaseOutcome` 7 variants (`push.rs:190-208`), `ReplaceLeaseOutcome` 3 (`:102-110`), `EnqueueWakeOutcome` 3 (`:113-121`), `RevalidateWakeOutcome` 2 (`:166-172`).

`class_rank` orders 4 classes `silent < default < time_sensitive < urgent` and is **duplicated verbatim** in two files (`push_lease.rs:575-583`, `push_runtime.rs:567-575`) with no shared definition.

`max_class` is derived by taking the highest-ranked class across all subscriptions (`push_lease.rs:536-543`).

---

#### 5. Archived-identity model

`ArchivedIdentity` (`buzz-db/src/archived_identities.rs:17-32`), keyed `(community_id, pubkey)` with `ON CONFLICT DO NOTHING` — re-archiving is a no-op that does **not** update the existing row (`archived_identities.rs:66`, doc `:46-48`):

| Field | Type | Provenance in handler |
|---|---|---|
| `pubkey` | 64-char lowercase hex `String` | `extract_single_p_tag_hex` + `to_ascii_lowercase()` (`identity_archive.rs:56-58`) |
| `consent_path` | `String` — `"self"` \| `"owner"` \| `"admin"` | `ConsentPath::as_str()` (`identity_archive.rs:23-37`, resolved `:228-251`) |
| `actor` | hex `String` | `event.pubkey.to_hex()` (`identity_archive.rs:44`) |
| `reason` | `Option<String>` | `["reason", …]` tag (`identity_archive.rs:63`) |
| `replaced_by` | `Option<String>` | `["replaced-by", …]` tag; must be lowercase 64-hex, must differ from target, at most one (`identity_archive.rs:200-226`) |
| `request_event_id` | hex `String` | `event.id.to_hex()` (`identity_archive.rs:66`) |
| `archived_at` | `DateTime<Utc>` | DB default |

`ConsentPath` is a private 3-variant enum with no `Display`/serde — only `as_str()`. The enum ordering in `determine_consent_path` is `SelfSigned` → `Admin` (owner *or* admin role) → `Owner` (`identity_archive.rs:230-251`); note the naming inversion: a community **`owner` role** produces `ConsentPath::Admin` (`identity_archive.rs:245-247`), while `ConsentPath::Owner` means *NIP-OA key owner of the target agent*, verified cryptographically (`:250`).

Deletion is a hard `DELETE` (`archived_identities.rs:83-91`) — unarchiving destroys the consent/actor/reason provenance rather than tombstoning it.

`archive`/`unarchive` return `bool changed` (`identity_archive.rs:70-96`) which gates all downstream event emission (`identity_archive.rs:100-102`).

---

#### 6. Storage-sweep snapshot shape

Config value object, read once at boot (`storage_sweep.rs:34-46`): `StorageSweepConfig { interval: Duration, timeout: Duration, max_objects: u64, enabled: bool }`, `Copy`.

Runtime state (`storage_sweep.rs:130-141`), one instance in `AppState` behind `tokio::sync::Mutex` (`state.rs:561`):

| Field | Type | Meaning |
|---|---|---|
| `in_flight` | `Option<JoinHandle<SweepAttempt>>` | single-flight slot |
| `cached` | `Option<CachedSnapshot>` | last **successful** snapshot + harvest `Instant` (`:107-111`) |
| `last_attempt` | `Option<LastAttempt>` | `{ ok: bool, duration: Duration }` of the newest attempt regardless of success (`:98-102`) |
| `failures_total` | `u64` | process-local, resets on failover |
| `previously_emitted` | `HashSet<StorageEmittedKey>` | prior tick's per-community series, for zeroing |

`StorageEmittedKey` (`storage_sweep.rs:121-124`) is `Bytes(String)` \| `Objects(String)` keyed on the **resolved host label**, not the UUID, specifically so a host rename can still zero the old series (`storage_sweep.rs:114-119`).

The snapshot itself is `buzz_media::BucketSnapshot` (`buzz-media/src/bucket_index.rs:206-226`) — 11 scalar counters plus `per_community: HashMap<Uuid, CommunityStorage>`:

| Group | Fields |
|---|---|
| physical | `physical_bytes`, `physical_objects` |
| logical (bills a multi-bound blob N times, `:210-211`) | `logical_bytes`, `logical_objects` |
| orphans | `orphan_blob_bytes`, `orphan_blob_count`, `orphan_sidecar_count` |
| anomalies | `multi_variant_shas`, `multi_variant_bytes` |
| unclassified | `unknown_key_bytes`, `unknown_key_objects` |
| per-tenant | `per_community: HashMap<Uuid, CommunityStorage { bytes, objects }>` |

`CachedSnapshot.completed_at` is stamped at **harvest**, not sweep completion (`storage_sweep.rs:171-173`), so the exported `buzz_storage_sweep_age_seconds` under-reports staleness by up to one usage tick.

---

#### 7. Workflow-sink action payloads

`RelayActionSink` holds a single field: `state: Weak<AppState>` (`workflow_sink.rs:159-161`) — `Weak` specifically to break the `AppState → WorkflowEngine → ActionSink → AppState` `Arc` cycle (`:150-155`).

The sink implements exactly **one** action, `send_message` (`workflow_sink.rs:173-179`), whose input tuple is `(CommunityId, channel_id: &str, text: &str, author_pubkey: &str)` and whose output is the emitted event id hex.

Emitted event shape — relay-signed **kind:9** with a variable-length tag list (`workflow_sink.rs:259-296`):

| Tag | Cardinality | Meaning |
|---|---|---|
| `["p", <workflow owner hex>]` | exactly 1 | attribution (event is signed by the *relay* key, `:304`) |
| `["h", <canonical channel UUID>]` | exactly 1 | NIP-29 scope |
| `["buzz:workflow", "true"]` | exactly 1 | recursion guard |
| `["p", <mentioned member hex>]` | 0..n | one per `@Name` resolving to a channel member, author skipped (`:290-296`) |

Thread metadata is fixed (`workflow_sink.rs:325-335`): `depth: 0`, `parent_event_id: None`, `root_event_id: None`, `broadcast: false` — documented as "Workflow messages are always top-level" (`:322`). A workflow can therefore never reply into a thread.

The mention-resolution intermediate is `members: &[(String /*display_name*/, String /*pubkey hex*/)]` built from `get_members` + `get_users_bulk`, dropping members with no `display_name` (`workflow_sink.rs:274-289`). Ambiguity is modelled as `HashMap<String /*lowercased name*/, Option<String /*pubkey*/>>` where `None` means "≥2 distinct pubkeys share this name ⇒ tag nobody" (`workflow_sink.rs:47-64`).

`event_created_at` falls back to `Utc::now()` on an out-of-range timestamp (`workflow_sink.rs:311-314`) — silently, unlike `product_feedback.rs:47-49` which rejects the same condition.

---

#### 8. Product-feedback model

`NewProductFeedback` built at `product_feedback.rs:59-68`:

| Field | Constraint |
|---|---|
| `event_id` | `event.id.as_bytes()` |
| `submitter_pubkey` | raw 32 bytes |
| `category: Option<&str>` | 0 or 1 `category` tag from `["bug","praise","needs-work"]` (`:11`, `:78-92`) |
| `body: &str` | non-blank, ≤ 32 KiB (`:12`, `:94-104`) |
| `tags: serde_json::Value` | full tag array re-serialized, ≤ 64 KiB (`:13`, `:70-76`) |
| `event_created_at` | rejected if out of `DateTime` range (`:47-49`) |

`imeta` tags, when present, are validated and their blobs verified against tenant-scoped media before insert (`product_feedback.rs:26-36`).

---

#### 9. Cross-cutting data-model observations

| Observation | Evidence |
|---|---|
| Pubkey representation is split: `relay_members` + `archived_identities` use 64-hex `String`; `community_bans` + `moderation_actions` + `moderation_reports` use raw `Vec<u8>` | `relay_members.rs:41`, `archived_identities.rs:34` vs `moderation.rs:85`, `:122` |
| Two independent definitions of kind 30350 | `buzz-core/src/kind.rs:109` (`30350`) and `push_lease.rs:19` (`30_350`); ingest imports the `push_lease` copy (`ingest.rs:204`, `:450`, `:2156`) while `req.rs:1689` uses the `buzz-core` one |
| `class_rank` duplicated | `push_lease.rs:575`, `push_runtime.rs:567` |
| `extract_tag_value` duplicated (identical body, 3 copies) | `moderation_commands.rs:608`, `relay_admin.rs:49`, `identity_archive.rs:189` |
| `extract_p_tag_*` triplicated with divergent semantics — `moderation_commands.rs:561` returns *first* valid; `relay_admin.rs:33` returns *first* valid as hex; `identity_archive.rs:170` requires *exactly one* | see cited lines |
| ±120 s freshness constant re-declared 3× | named const `moderation_commands.rs:81`; bare literal `relay_admin.rs:125`; bare literal `identity_archive.rs:147`; a 4th as `ALLOWED_SKEW` `push_lease.rs:476` |
