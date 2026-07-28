<!-- Analyzed: 2026-07-25T01:12:08Z | Scope: full project -->
# Data Model

> Status: initialized in Phase 1. Entities, schemas, relationships, and access patterns
> are populated per-module during Phase 2 and consolidated in Phase 3.

## Summary

PostgreSQL 17 is the primary store: 24 forward-only migrations, a desired-state
`schema/schema.sql`, monthly range partitioning on `events` and `delivery_log`, and a
generated `search_tsv` tsvector column backing full-text search.

Batch 2a (foundation crates) contributes **no persistence model** — these are the
in-memory domain types every other crate builds on:

- `buzz-core` — `StoredEvent` (event + received_at + channel_id + private `verified`
  flag), `CommunityId`/`TenantContext` (deliberately non-`Deserialize` tenant fence),
  channel/role/presence enums with a numeric permission ladder (Owner 4 → Bot 0), the
  NIP-AE engram `Body` codec, NIP-AM turn-metric payloads, the git ref-protection model,
  and the NIP-AB pairing session state machine (7 states, 12 private fields, zeroized on drop).
- `buzz-sdk` — builder input structs and the **event wire shapes** (tags + content) for
  45 kind integers. No persistence.
- `buzz-persona` — a four-stage on-disk pack schema (`plugin.json` → `PersonaConfig` →
  `LoadedPersona` → `ResolvedPersona`), merged through an untyped `serde_json::Value` layer.
- `buzz-ws-client` — connection/message types only; auth state is implicit (no state enum).

Batch 2b (service crates) contributes the actual persistence model:

- **`buzz-db`** — **37 tables, 63 indexes**, 409 public API items, 136 business rules,
  0 `unsafe`, 0 TODOs. `lib.rs` alone is 6,106 lines. Monthly range partitioning on
  `events` and `delivery_log` as described in Phase 1.
- **`buzz-audit`** — owns no DDL; the `audit_log` table belongs to migration `0001`.
  Its data model is the hash-chain entry plus a per-community advisory lock.
- **`buzz-search`** — owns no DDL; reads the `search_tsv` generated column and issues a
  single query shape with `community_id` always as the first predicate.
- **`buzz-media`** — object-store keys and validation metadata, no relational schema.
- **`buzz-workflow`** — workflow/run/step/approval rows via `buzz-db`; the only 2b crate
  that depends on `buzz-db`.
- **`buzz-auth`** — session/scope types; no owned tables.
- **`buzz-pubsub`** — **no SQL at all**; its "data model" is a Redis keyspace (8 key
  families, all community-prefixed except IP rate limits) plus three unversioned JSON
  wire payloads.

Two schema-integrity findings surfaced (detail in the `buzz-db` section and in `debt.md`):

- **`schema/schema.sql` is not actually the source of truth it claims to be** — 12 objects
  present in the migration end-state are missing from it, and its `search_tsv` definition
  matches no migration end-state.
- **Migration 0008 makes fresh and brownfield installs diverge permanently.** It grants a
  positive FTS allowlist only to *empty* databases, so two deployments of the same version
  run different search policies forever.

One tenant-scoping exception exists across all tenant-table writes: `Db::backfill_d_tags`
(`crates/buzz-db/src/lib.rs:2810`) has no community predicate.

Test posture worth noting for anyone relying on this layer: **121 of 122 async tests in
`buzz-db` are `#[ignore]`d**, so the default unit-test gate exercises essentially none of
the data-access code.

### Batch 2c data model (relay, mesh, conformance)

The relay adds **no persistent tables** — batch 2b's 37-table Postgres schema is the whole
durable model. What 2c contributes is (a) a large in-process mutable state object, (b) two
wire vocabularies (WebSocket protocol frames, mesh gossip records), and (c) a serde-only
trace schema.

- **`AppState` is the de-facto god object** — every subsystem hangs off one struct
  (`crates/buzz-relay/src/state.rs`), including the DB pool, Redis pub/sub, rate limiter,
  media store, audit service, workflow engine, mesh runtime, and the conformance tracer
  (`state.rs:620`). Its in-memory caches (channel→community, membership, ban, NIP-05) are the
  only tenancy-resolution fast path, and each has its own eviction discipline.
- **Tenancy is a two-type fence**: `CommunityId` (non-`Deserialize`, so it cannot be conjured
  from client input) plus `TenantContext` (`crates/buzz-relay/src/tenant.rs`), host-derived on
  every request. `Inv_RowZero` names this seam (`tenant.rs:76`, `:291`, `:312`).
- **Mesh introduces an unauthenticated gossip record model** — peer records carry
  `endpoint_addrs` and are accepted without signature or origin proof
  (`crates/buzz-relay-mesh/src/membership.rs:120-153`), and there is no eviction path, so the
  peer table only grows.
- **Conformance has no persistent model at all** — its "data model" is a JSONL wire schema
  (`TraceStep` = `schema_version` + `action` + `state_after`,
  `crates/buzz-conformance/src/lib.rs:290-299`) plus an in-memory `ModelState`
  (`src/transitions.rs:105-118`) carried across one trace. The abstract state is three fields:
  `resolved_community`, `bound_host`, `actor` (`src/lib.rs:150-175`).
- **The trace alphabet is narrower than the spec it mirrors.** `SanitizedReason` has 3
  variants (`src/lib.rs:132-140`) against the 9 declared in
  `docs/spec/MultiTenantRelay.cfg:26`, and `TraceAction` models 8 of the 23 `Next` actions in
  `docs/spec/MultiTenantRelay.tla:933-956`.
- **Documented wire keys do not match the code** — `TRACE_SCHEMA.md:37-46` documents
  `{"schema":…, "state":…}`; serde emits `schema_version` / `state_after`
  (`src/lib.rs:292`, `:298`).

## Entities

| Entity | Table / Type | Module | Key Fields | Source |
|---|---|---|---|---|
| _pending_ | | | | |

## Relationships

_Pending Phase 2._

## Access Patterns

| Pattern | Query Shape | Module | Index Support | Source |
|---|---|---|---|---|
| _pending_ | | | | |

## Migrations

| # | File | Purpose |
|---|---|---|
| 0001 | `migrations/0001_initial_schema.sql` | Consolidated initial schema |
| 0002 | `migrations/0002_git_repo_names.sql` | Git repo naming |
| 0003 | `migrations/0003_community_icon.sql` | Community icon |
| 0004 | `migrations/0004_events_tags_gin.sql` | GIN index on event tags |
| 0005 | `migrations/0005_agent_turn_metric_fts.sql` | Agent turn metric FTS |
| 0006 | `migrations/0006_moderation.sql` | Moderation |
| 0007 | `migrations/0007_nip_rs_retention.sql` | NIP-RS retention |
| 0008 | `migrations/0008_fresh_install_search_allowlist.sql` | Search allowlist |
| 0009 | `migrations/0009_nip_rs_database_guards.sql` | DB guards |
| 0010 | `migrations/0010_nip_rs_exact_replay_guard.sql` | Exact replay guard |
| 0011 | `migrations/0011_nip_rs_exact_tag_cardinality.sql` | Tag cardinality |
| 0012 | `migrations/0012_push_leases.sql` | Push leases |
| 0013 | `migrations/0013_push_endpoint_state.sql` | Push endpoint state |
| 0014 | `migrations/0014_push_lease_fts.sql` | Push lease FTS |
| 0015 | `migrations/0015_push_gateway_authority.sql` | Push gateway authority |
| 0016 | `migrations/0016_community_archival.sql` | Community archival |
| 0017 | `migrations/0017_product_feedback.sql` | Product feedback |
| 0018 | `migrations/0018_push_match_queue.sql` | Push match queue |
| 0019 | `migrations/0019_mesh_status_retention.sql` | Mesh status retention |
| 0020 | `migrations/0020_join_policy_acceptances.sql` | Join-policy acceptances |
| 0021 | `migrations/0021_created_at_fence_floor.sql` | created_at fence floor |
| 0022 | `migrations/0022_event_ttl_refresh.sql` | Event TTL refresh |
| 0023 | `migrations/0023_push_match_gate.sql` | Push match gate |
| 0024 | `migrations/0024_event_ttl_refresh_shared_lock.sql` | TTL refresh shared lock |

---

# Phase 2 — Module Findings

## Module: buzz-core (`crates/buzz-core`)

### Aspect: Data Model

All types below were read directly from source. 20 `.rs` files (15 in `src/`, 5 in `src/pairing/`), no `tests/` directory, no `bin/` targets — the crate is a pure library (`crates/buzz-core/src/lib.rs:9-45`).

---

### 1. Event wrapper

#### `StoredEvent` — `crates/buzz-core/src/event.rs:11-19`

| Field | Type | Visibility | Notes |
|-------|------|-----------|-------|
| `event` | `nostr::Event` | `pub` | the underlying Nostr event (`event.rs:13`) |
| `received_at` | `chrono::DateTime<Utc>` | `pub` | relay wall-clock receive time (`event.rs:15`) |
| `channel_id` | `Option<uuid::Uuid>` | `pub` | channel scope; `None` = global/DM (`event.rs:17`) |
| `verified` | `bool` | **private** | read accessor `is_verified()` at `event.rs:33-35` |

Derives: `Debug, Clone` (`event.rs:10`). **No `Serialize`/`Deserialize`.**

Constructors:
- `StoredEvent::new(event, channel_id)` → `received_at = Utc::now()`, `verified = false` (`event.rs:23-30`, flag at `:28`).
- `StoredEvent::with_received_at(event, received_at, channel_id, verified)` → all four fields explicit (`event.rs:38-49`).

Invariant carried by construction: `new()` always yields an unverified wrapper (`event.rs:28`), so a `verified = true` wrapper can only be produced deliberately via `with_received_at` (`event.rs:38-49`).

---

### 2. Errors as domain types

| Type | Declared at | Variants (with payload) |
|------|-------------|------------------------|
| `VerificationError` | `src/error.rs:3` (derive `:2`) | `InvalidId { computed: String, got: String }` (`:6-11`, fields `:8`, `:10`), `InvalidSignature` (`:15`), `Secp(#[from] nostr::secp256k1::Error)` (`:19`) |
| `EngramError` | `src/engram.rs:38` (derive `:37`) | `InvalidSlug(String)` `:41`, `InvalidBody(String)` `:44`, `InvalidEnvelope(String)` `:47`, `Decrypt` `:50`, `Encrypt(String)` `:53`, `BodyTooLarge(usize)` `:56`, `Sign(String)` `:59` |
| `ObserverPayloadError` | `src/observer.rs:29` (derive `:28`) | `Nip44(#[from] nip44::Error)` `:32`, `Json(#[from] serde_json::Error)` `:35`, `InvalidCiphertextLength(usize)` `:38`, `PlaintextTooLarge { max: usize, got: usize }` `:41-46`, `InvalidPayload(String)` `:49` |
| `PairingError` | `src/pairing/mod.rs:35` (derive `:34`) | `InvalidQr(String)`, `InvalidSessionId`, `SasMismatch`, `TranscriptMismatch`, `UnexpectedMessage { expected: String, got: String }`, `SessionExpired`, `Nip44(#[from] …)`, `Json(#[from] …)`, `InvalidPubkey(String)` `:75`, `SigningError(String)` `:79` |
| `NormalizeRelayUrlError` | `src/relay.rs:8` (derive `:7`, includes `PartialEq, Eq`) | `InvalidUrl(String)`, `InvalidScheme`, `Credentials`, `Fragment`, `MissingHost` |
| `PatternError` | `src/git_perms.rs:52` (derive `:51`) | `Empty`, `TooLong`, `MissingRefsPrefix`, `InvalidSegment(String)`, `TooManyWildcards` — hand-written `Display` (`:65-78`) + `impl std::error::Error` (`:79`) |
| `RuleParseError` | `src/git_perms.rs:264` (derive `:263`) | `TooFewValues`, `TooManyRules`, `InvalidPattern(PatternError)`, `UnknownRule(String)`, `InvalidRole(String)` — hand-written `Display` (`:277-287`) + `impl std::error::Error` (`:289`) |

`AgentTurnMetricError` is a re-export alias of `ObserverPayloadError` (`src/agent_turn_metric.rs:15`).

---

### 3. Tenant identity

#### `CommunityId` — `crates/buzz-core/src/tenant.rs:37`

| Field | Type | Visibility |
|-------|------|-----------|
| `.0` | `uuid::Uuid` | **private** (tuple newtype) |

Derives: `Debug, Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord` (`tenant.rs:36`). **Deliberately no `Serialize`/`Deserialize`** — the "fence" rationale is at `tenant.rs:10-30`.
Accessors: `from_uuid(Uuid) -> Self` (const, `tenant.rs:45-47`), `as_uuid(&self) -> &Uuid` (const, `tenant.rs:50-52`), `Display` delegating to the inner UUID (`tenant.rs:55-59`).

#### `TenantContext` — `crates/buzz-core/src/tenant.rs:68`

| Field | Type | Visibility | Accessor |
|-------|------|-----------|----------|
| `community` | `CommunityId` | **private** (`tenant.rs:69`) | `community()` (const) `tenant.rs:87-89` |
| `host` | `String` | **private** (`tenant.rs:70`) | `host()` `tenant.rs:95-97` |

Derives: `Debug, Clone, PartialEq, Eq` (`tenant.rs:67`). No `Default`, no `Deserialize` (stated at `tenant.rs:20-21`). Single constructor `TenantContext::resolved(community, host: impl Into<String>)` (`tenant.rs:79-84`), documented as callable only from the host-resolution path (`tenant.rs:73-78`). The module explicitly labels this a "lint-and-review fence, not a compiler fence" (`tenant.rs:23-30`).

---

### 4. Channel / membership enums

| Type | Declared at | Variants | String form |
|------|-------------|----------|-------------|
| `ChannelVisibility` | `src/channel.rs:22` (derive `:21`) | `Open`, `Private` | `as_str()` → `"open"`/`"private"` (`:31-37`) |
| `ChannelType` | `src/channel.rs:59` (derive `:58`) | `Stream`, `Forum`, `Dm`, `Workflow` | `as_str()` (`:72-79`) |
| `MemberRole` | `src/channel.rs:108` (derive `:107`) | `Owner`, `Admin`, `Member`, `Guest`, `Bot` | `as_str()` (`:123-131`) |

All three derive `Debug, Clone, Copy, PartialEq, Eq` and implement `Display` + `FromStr` with `Err = String`: `ChannelVisibility` (`:39`, `:45`), `ChannelType` (`:82`, `:88`), `MemberRole` (`:160`, `:166`). **No serde derives** — string conversion is manual and shared with DB enums and Nostr tags (doc comments `channel.rs:30`, `:71`, `:122`).

`MemberRole` numeric model (`channel.rs:142-150`):

| Role | `permission_level()` | `is_elevated()` (`:134-136`) |
|------|---------------------|------------------------------|
| Owner | 4 | true |
| Admin | 3 | true |
| Member | 2 | false |
| Guest | 1 | false |
| Bot | 0 | false |

`has_at_least(required)` = `self.permission_level() >= required.permission_level()` (`channel.rs:155-157`). Bot is documented as outside the linear hierarchy (`channel.rs:104-106`).

---

### 5. Presence

`PresenceStatus` — `src/presence.rs:11`: `Online | Away | Offline`. Derives `Debug, Clone, Copy, PartialEq, Eq, Deserialize, Serialize` (`presence.rs:9`) with `#[serde(rename_all = "lowercase")]` (`:10`). `as_str()` (`:22-28`) and `Display` (`:31-35`) both yield the same lowercase strings. Doc notes the WebSocket path (kind 20001) accepts arbitrary strings while this enum is the curated REST/MCP set (`presence.rs:7-8`).

---

### 6. NIP-AM agent turn metrics (`src/agent_turn_metric.rs`)

#### `TokenCounts` — `agent_turn_metric.rs:23`
Derives `Debug, Clone, PartialEq, Serialize, Deserialize` (`:21`) with `#[serde(rename_all = "camelCase")]` (`:22`).

| Field | Type | Declared | Serde |
|-------|------|----------|-------|
| `input_tokens` | `Option<u64>` | `:25` | `inputTokens`, serialized as `null` when `None` |
| `output_tokens` | `Option<u64>` | `:28` | `outputTokens`, `null` when `None` |
| `total_tokens` | `Option<u64>` | `:32` | `totalTokens`; doc: provider-reported, NOT derived (`:30-31`) |
| `cost_usd` | `Option<f64>` | `:35` | `costUsd`; must be finite + non-negative when present (`:34`) |
| `cache_read_tokens` | `Option<u64>` | `:39` | `#[serde(skip_serializing_if = "Option::is_none")]` (`:38`) |
| `cache_write_tokens` | `Option<u64>` | `:43` | same skip attribute (`:42`) |

Semantic invariant documented at `:18-20`: `None` means "not reported", not zero. Test `null_token_counts_round_trip` (`:301-319`) asserts `inputTokens`/`outputTokens` serialize as literal `null`.

#### `StopReason` — `agent_turn_metric.rs:53`
`EndTurn | MaxTokens | Cancelled | Error | Unknown`; `Serialize` derived (`:51`) with `rename_all = "snake_case"` (`:52`), **hand-written `Deserialize`** (`:66-77`) mapping any unrecognized string to `Unknown`.

#### `AgentTurnMetricPayload` — `agent_turn_metric.rs:89`
Derives `Debug, Clone, PartialEq, Serialize, Deserialize` (`:87`) with `#[serde(rename_all = "camelCase")]` (`:88`).

| Field | Type | Declared | Required per doc | Notes |
|-------|------|----------|------------------|-------|
| `harness` | `String` | `:91` | REQUIRED (`:90`) | e.g. `"goose"` |
| `model` | `Option<String>` | `:94` | no | |
| `channel_id` | `Option<String>` | `:97` | no | `channelId`, encrypted inside payload (`:96`) |
| `session_id` | `Option<String>` | `:100` | required when `cumulative` present (`:99`) | |
| `turn_id` | `Option<String>` | `:103` | no | |
| `turn_seq` | `Option<u64>` | `:109` | required when `cumulative` present; strictly increasing per session (`:105-108`) | |
| `timestamp` | `String` | `:112` | REQUIRED, RFC 3339 (`:111`) | unparsed string |
| `turn` | `Option<TokenCounts>` | `:115` | no | computed delta |
| `cumulative` | `Option<TokenCounts>` | `:118` | no | session-cumulative |
| `delta_reliable` | `bool` | `:124` | `#[serde(default = "default_delta_reliable")]` (`:123`) → `true` (`:130-132`) | |
| `stop_reason` | `Option<StopReason>` | `:127` | no | |

Relationships: `AgentTurnMetricPayload` embeds two `TokenCounts` (turn + cumulative) and one `StopReason`; `validate()` (`:140-169`) is the only place invariants on `cost_usd` are checked.

---

### 7. NIP-AE engrams (`src/engram.rs`)

#### `Body` — `engram.rs:158`
Enum discriminated by slug; derives `Debug, Clone, PartialEq, Eq` (`:157`). No serde derives — encoding is hand-rolled for byte-exact spec conformance (`:190-192`).

| Variant | Declared | Fields |
|---------|----------|--------|
| `Core { profile: String }` | `:160` | `profile` at `:162`; used when `slug == "core"` |
| `Memory { slug: String, value: Option<String> }` | `:165` | `slug` at `:167`, `value` at `:169`; `value = None` is a tombstone (`:164`) |

Methods: `slug()` (`:175-180`), `is_tombstone()` (`:183-185`), `to_json_bytes()` (`:189-212`, emits `{"slug":…,"value":…}` / `{"slug":"core","profile":…}` whitespace-free with slug first), `from_json_bytes()` (`:216-259`).

#### `Listing` — `engram.rs:598`
Wire type for `buzz mem ls`; derives `Debug, Clone, Serialize, Deserialize` (`:597`).

| Field | Type | Declared |
|-------|------|----------|
| `slug` | `String` | `:600` |
| `event_id` | `String` | `:602` |
| `created_at` | `u64` | `:604` |

External type reused: `nostr::nips::nip44::v2::ConversationKey` as the `K_c` carrier, returned by `conversation_key()` (`engram.rs:136-138`).

---

### 8. Git permission model (`src/git_perms.rs`)

#### `RefPattern` — `git_perms.rs:32` (derive `:31`)

| Field | Type | Visibility |
|-------|------|-----------|
| `raw` | `String` | **private** (`:34`) — accessor `as_str()` (`:181-183`) + `Display` (`:186-190`) |
| `segments` | `Vec<PatternSegment>` | **private** (`:36`) |

`PatternSegment` (`:41`, derive `:40`) is a **private** enum: `Literal(String)`, `Wildcard`, `RecursiveWildcard`. Invariants enforced in `RefPattern::parse` (`:83-146`): non-empty, ≤ `MAX_PATTERN_LENGTH`, must start with `refs/`, `**` only as final segment, ≤ `MAX_WILDCARDS_PER_PATTERN` wildcards, segment charset `[A-Za-z0-9._-]`.

#### `UpdateKind` — `git_perms.rs:196` (derive `:195`)
`Create | FastForward | NonFastForward | Delete`; classified by `classify(old_oid, new_oid, is_ancestor)` (`:212-221`) against a function-local 40-zero OID constant (`:213`).

#### `RefUpdate` — `git_perms.rs:228` (derive `:227`)

| Field | Type | Declared |
|-------|------|----------|
| `ref_name` | `String` | `:230` |
| `kind` | `UpdateKind` | `:232` |
| `old_oid` | `String` (zero OID for creates) | `:234` |
| `new_oid` | `String` (zero OID for deletes) | `:236` |

#### `ProtectionRule` — `git_perms.rs:244` (derive `:243`)

| Field | Type | Declared | Notes |
|-------|------|----------|-------|
| `pattern` | `RefPattern` | `:246` | |
| `push_role` | `Option<MemberRole>` | `:248` | strictest wins when merged (`:327-336`) |
| `no_force_push` | `bool` | `:250` | |
| `no_delete` | `bool` | `:252` | |
| `require_patch` | `bool` | `:259` | doc: blocks ALL update kinds, not just FF (`:254-258`) |

#### `ParsedProtection` — `git_perms.rs:365` (derive `:364`)
`rules: Vec<ProtectionRule>` (`:367`) + `unknown_rules: Vec<String>` (`:370`, forward-compat reporting).

#### `EffectiveRules` — `git_perms.rs:432` (derive `:431`)
`push_role: Option<MemberRole>` (`:434`), `no_force_push: bool` (`:436`), `no_delete: bool` (`:438`), `require_patch: bool` (`:440`), `has_explicit_match: bool` (`:442`). Computed by `for_ref(ref_name, rules)` (`:447-475`) as a union across matching rules.

#### `Denial` — `git_perms.rs:492` (derive `:491`)
`ref_name: String` (`:494`), `reason: String` (`:496`); `Display` → `"{ref_name}: {reason}"` (`:499-503`).

Relationship chain (documented at `git_perms.rs:9-16`): kind:30617 tags → `parse_protection_tags` → `Vec<ProtectionRule>` → `EffectiveRules::for_ref` → `evaluate_ref_update` / `evaluate_push` → `Denial`s. `ProtectionRule` and the evaluator both consume `MemberRole` imported at `git_perms.rs:15`, making `git_perms` the only intra-crate cross-module type dependency besides `kind`.

---

### 9. NIP-AB pairing types (`src/pairing/`)

#### `QrPayload` — `pairing/qr.rs:34` (derive `:33`)

| Field | Type | Declared | Notes |
|-------|------|----------|-------|
| `source_pubkey` | `nostr::PublicKey` | `:36` | |
| `session_secret` | `[u8; 32]` | `:40` | zeroized on drop (`:52-56`) |
| `relays` | `Vec<String>` | `:42` | ≥ 1 required on decode (`qr.rs:176-181`) |
| `version` | `u32` | `:47` | `v=1`; absent → 1, values > 1 rejected (`qr.rs:145-151`) |

Derives `Debug, Clone` (`qr.rs:33`) and implements `Drop` calling `session_secret.zeroize()` (`qr.rs:52-56`). No serde — the wire form is the `nostrpair://` URI.

#### `PairingMessage` — `pairing/types.rs:21`
Derives `Debug, Clone, PartialEq, Eq, Serialize, Deserialize` (`:19`) with `#[serde(tag = "type", rename_all = "kebab-case")]` (`:20`).

| Variant | Declared | Fields |
|---------|----------|--------|
| `Offer` | `:23` | `session_id: String` (`:25`, hex 32B), `version: u32` (`:30`) with `#[serde(default = "default_version")]` (`:29`) → `default_version()` at `:9-11` |
| `SasConfirm` | `:34` | `transcript_hash: String` (`:36`, hex 32B) |
| `Payload` | `:40` | `payload_type: PayloadType` (`:42`), `payload: String` (`:44`) |
| `Complete` | `:48` | `success: bool` (`:50`) |
| `Abort` | `:54` | `reason: AbortReason` (`:56`) |

#### `PayloadType` — `pairing/types.rs:63`
`Nsec | Bunker | Connect | Custom`; derive `:61`, `rename_all = "snake_case"` (`:62`).

#### `AbortReason` — `pairing/types.rs:81`
`SasMismatch | UserDenied | Timeout | ProtocolError | Unknown`; derive `:79`, `rename_all = "snake_case"` (`:80`). `Unknown` carries `#[serde(other)]` (`:94`) so unrecognized strings deserialize to it (`:95`). Doc states callers MUST NOT emit `Unknown` outbound (`:91-93`).

#### `Role` / `SessionState` — `pairing/session.rs:50` and `:59`
`Role`: `Source | Target` (derive `:49`). `SessionState`: `Waiting | Confirming | AwaitingConfirmation | Transferring | PayloadExchanged | Completed | Aborted` (derive `:58`).

#### `PairingSession` — `pairing/session.rs:82`
All 12 fields are **private**; no derives at all (not even `Debug`).

| Field | Type | Declared | Accessor |
|-------|------|----------|----------|
| `role` | `Role` | `:83` | `role()` (`:486-488`) |
| `state` | `SessionState` | `:84` | `state()` (`:481-483`) |
| `keys` | `nostr::Keys` | `:86` | `pubkey()` (`:491-493`), `sign_event()` (`:510-515`) |
| `session_secret` | `[u8; 32]` | `:88` | none (zeroized on drop, `:733`) |
| `relay_urls` | `Vec<String>` | `:90` | `relay_urls()` (`:496-498`) |
| `peer_pubkey` | `Option<PublicKey>` | `:93` | none |
| `session_id` | `[u8; 32]` | `:95` | none (zeroized, `:734`) |
| `sas_code` | `Option<u32>` | `:97` | `sas_code()` → `Option<String>` (`:501-503`) |
| `sas_input` | `Option<[u8; 32]>` | `:99` | none (zeroized, `:735-737`) |
| `processed_ids` | `HashSet<[u8; 32]>` | `:102` | test-only `has_processed()` (`:536-538`) |
| `created_at` | `std::time::Instant` | `:104` | via `is_expired()` (`:476-478`) |
| `timeout` | `Duration` | `:106` | test-only `set_timeout()` (`:541-543`) |

`Drop` impl zeroizes `session_secret`, `session_id`, `sas_input` (`session.rs:731-739`).

---

### Cross-type relationships (summary)

- `StoredEvent` (`event.rs:11`) is the input to all filter matching (`filter.rs:10-12`, `:35`).
- `filter.rs` reads kind constants from `kind.rs` (`filter.rs:24-25`).
- `git_perms.rs` imports `channel::MemberRole` (`git_perms.rs:15`).
- `agent_turn_metric.rs` reuses `observer.rs` crypto + error types (`agent_turn_metric.rs:12`, `:15`).
- `engram.rs` and `pairing/session.rs` both consume `kind.rs` constants (`engram.rs:18`, `session.rs:46`).
- `tenant.rs`, `relay.rs`, `network.rs`, `presence.rs`, `channel.rs` have no intra-crate type dependencies.


## Module: buzz-sdk (`crates/buzz-sdk`)

### Aspect: Data Model

The crate defines **no persistence model**. Its "data model" is (a) the input
parameter structs consumed by builder functions and (b) the Nostr event wire
shapes those builders emit. Every builder returns `nostr::EventBuilder`
(unsigned); the caller signs (`crates/buzz-sdk/src/lib.rs:11-13`).

---

### 1. Input types declared in `lib.rs`

| Type | Kind | Fields (type) | File:line |
|---|---|---|---|
| `ThreadRef` | struct | `root_event_id: nostr::EventId`, `parent_event_id: nostr::EventId` | `crates/buzz-sdk/src/lib.rs:28-33` |
| `DiffMeta` | struct | `repo_url: String`, `commit_sha: String`, `file_path: Option<String>`, `parent_commit: Option<String>`, `branch: Option<(String, String)>`, `pr_number: Option<u32>`, `language: Option<String>`, `description: Option<String>`, `truncated: bool`, `alt_text: Option<String>` | `crates/buzz-sdk/src/lib.rs:36-57` |
| `VoteDirection` | enum (`Debug, Clone, Copy, PartialEq, Eq`) | `Up`, `Down` | `crates/buzz-sdk/src/lib.rs:60-66` |
| `CustomEmoji` | struct (`Debug, Clone, PartialEq, Eq`) | `shortcode: String`, `url: String` | `crates/buzz-sdk/src/lib.rs:69-75` |
| `SdkError` | enum (`thiserror::Error`) | see error table below | `crates/buzz-sdk/src/lib.rs:87-113` |

Re-exported from `buzz-core` (no local definition):

| Local alias | Source | Variants → tag string | File:line |
|---|---|---|---|
| `ChannelKind` | `buzz_core::channel::ChannelType` | `Stream`→`stream`, `Forum`→`forum`, `Dm`→`dm`, `Workflow`→`workflow` | `crates/buzz-sdk/src/lib.rs:80`; values `crates/buzz-core/src/channel.rs:59-79` |
| `Visibility` | `buzz_core::channel::ChannelVisibility` | `Open`→`open`, `Private`→`private` | `crates/buzz-sdk/src/lib.rs:82`; values `crates/buzz-core/src/channel.rs:22-36` |
| `MemberRole` | `buzz_core::channel::MemberRole` | `Owner`/`Admin`/`Member`/`Guest`/`Bot` → `owner`/`admin`/`member`/`guest`/`bot` | `crates/buzz-sdk/src/lib.rs:84`; values `crates/buzz-core/src/channel.rs:108-131` |
| `canonical_channel_name` | `buzz_core::channel::canonical_channel_name` | strips leading `#`/whitespace, trims trailing ws | `crates/buzz-sdk/src/lib.rs:78`; impl `crates/buzz-core/src/channel.rs:15-18` |
| `kind` module | `buzz_core::kind` | all kind integers | `crates/buzz-sdk/src/lib.rs:22` |

`SdkError` variants:

| Variant | Payload | Display text | File:line |
|---|---|---|---|
| `ContentTooLarge` | `{ max: usize, got: usize }` | `content exceeds maximum size of {max} bytes (got {got})` | `crates/buzz-sdk/src/lib.rs:89-96` |
| `InvalidTag` | `String` | `invalid tag: {0}` | `crates/buzz-sdk/src/lib.rs:97-99` |
| `EmojiTooLong` | — | `emoji exceeds maximum length of 64 characters` | `crates/buzz-sdk/src/lib.rs:100-102` |
| `TooManyMentions` | — | `too many mentions (max 50)` | `crates/buzz-sdk/src/lib.rs:103-105` |
| `InvalidDiffMeta` | `String` | `invalid diff metadata: {0}` | `crates/buzz-sdk/src/lib.rs:106-108` |
| `InvalidInput` | `String` | `invalid input: {0}` | `crates/buzz-sdk/src/lib.rs:109-112` |

---

### 2. Input types declared in `builders.rs`

| Type | Kind | Fields (type) | File:line |
|---|---|---|---|
| `DeleteMessageOptions<'a>` | struct (`Debug, Clone, Default`) | `action_id: Option<Uuid>`, `reason_code: Option<&'a str>`, `public_reason: Option<&'a str>` | `crates/buzz-sdk/src/builders.rs:392-400` |
| `GitRepoCoord` | struct | `owner: String` (64-hex pubkey), `id: String` (`d`-tag) | `crates/buzz-sdk/src/builders.rs:964-973` |
| `GitPatchMeta` | struct (`Default`) | `euc: Option<String>`, `recipients: Vec<String>`, `reply_to: Option<String>`, `root: bool`, `root_revision: bool`, `commit: Option<String>`, `parent_commit: Option<String>`, `commit_pgp_sig: Option<String>`, `committer: Option<(String, String, String, String)>` | `crates/buzz-sdk/src/builders.rs:984-1005` |
| `GitIssueMeta` | struct (`Default`) | `labels: Vec<String>`, `recipients: Vec<String>` | `crates/buzz-sdk/src/builders.rs:1072-1078` |
| `GitStatus` | enum (`Debug, Clone, Copy, PartialEq, Eq`) | `Open`, `AppliedOrResolved`, `Closed`, `Draft` | `crates/buzz-sdk/src/builders.rs:1114-1124` |
| `GitAppliedPatchRef` | struct (`Debug, Clone, PartialEq, Eq`) | `id: String`, `relay: Option<String>`, `pubkey: Option<String>` | `crates/buzz-sdk/src/builders.rs:1132-1141` |
| `GitStatusMeta` | struct (`Default`) | `root_event: String`, `accepted_revision_root: Option<String>`, `repo: Option<GitRepoCoord>`, `euc: Option<String>`, `recipients: Vec<String>`, `applied_patches: Vec<GitAppliedPatchRef>`, `merge_commit: Option<String>`, `applied_as_commits: Vec<String>` | `crates/buzz-sdk/src/builders.rs:1200-1219` |
| `GitPullRequestMeta` | struct (`Default`) | `euc: Option<String>`, `recipients: Vec<String>`, `channel_id: Option<String>`, `subject: String`, `labels: Vec<String>`, `commit: String`, `clone_urls: Vec<String>`, `branch_name: Option<String>`, `merge_base: Option<String>`, `revision_of: Option<String>` | `crates/buzz-sdk/src/builders.rs:1302-1327` |
| `GitPrUpdateMeta` | struct (`Default`) | `euc: Option<String>`, `recipients: Vec<String>`, `pr_event: String`, `pr_author: String`, `commit: String`, `clone_urls: Vec<String>`, `merge_base: Option<String>` | `crates/buzz-sdk/src/builders.rs:1396-1411` |
| `CUSTOM_EMOJI_SET_D_TAG` | `pub const &str` = `"buzz:custom-emoji"` | — | `crates/buzz-sdk/src/builders.rs:503` |
| `MAX_CONTACTS` | private `const usize` = `10_000` | — | `crates/buzz-sdk/src/builders.rs:751` |
| `MAX_REASON_BYTES` | private `const usize` = `64` | — | `crates/buzz-sdk/src/builders.rs:1704` |

`GitStatus` → kind mapping (`crates/buzz-sdk/src/builders.rs:1187-1196`):
`Open`→1630, `AppliedOrResolved`→1631, `Closed`→1632, `Draft`→1633.

`GitRepoCoord::to_a_tag_value` renders `30617:<owner>:<id>`
(`crates/buzz-sdk/src/builders.rs:975-982`).

---

### 3. Input types declared in `mentions.rs`

| Type | Kind | Fields | File:line |
|---|---|---|---|
| `MentionProfile<'a>` | struct (`Debug, Clone, Copy`) | `pubkey: &'a str` (lowercase hex), `content_json: &'a str` (raw kind-0 `content`) | `crates/buzz-sdk/src/mentions.rs:45-51` |
| `MENTION_CAP` | `pub const usize` = `50` | — | `crates/buzz-sdk/src/mentions.rs:38` |

`match_names_to_profiles` reads only `display_name`, falling back to `name`
when `display_name` is absent (`crates/buzz-sdk/src/mentions.rs:186-190`).

---

### 4. Event wire shapes produced (tags + content)

Content column is the exact `event.content` payload the builder sets.

| Kind | Builder | Tags emitted (in order) | Content shape |
|---|---|---|---|
| 9 | `build_message` | `["h",uuid]`, NIP-10 e-tags, `["p",hex]`*, `["broadcast","1"]`?, raw imeta tags | plain text, ≤64 KiB (`builders.rs:219-241`) |
| 24200 | `build_agent_observer_frame` | `["p",recipient]`, `["agent",agent_pk]`, `["frame","telemetry"\|"control"]` | NIP-44 v2 ciphertext (`builders.rs:245-274`) |
| 45001 | `build_forum_post` | `["h",uuid]`, `["p",hex]`*, imeta | text ≤64 KiB (`builders.rs:278-289`) |
| 45003 | `build_forum_comment` | `["h",uuid]`, NIP-10 e-tags, `["p",hex]`*, imeta | text ≤64 KiB (`builders.rs:292-305`) |
| 40008 | `build_diff_message` | `["h",uuid]`, `["repo",url]`, `["commit",sha]`, `["file",p]`?, `["parent-commit",sha]`?, `["branch",src,tgt]`?, `["pr",n]`?, `["l",lang]`?, `["description",d]`?, `["truncated","true"]`?, `["alt",t]`?, e-tags? | diff text ≤60 KiB (`builders.rs:308-375`) |
| 40003 | `build_edit` | `["h",uuid]`, `["e",target]` | replacement text ≤64 KiB (`builders.rs:378-389`) |
| 9005 | `build_delete_message` / `_with_options` | `["h",uuid]`, `["e",target]`, `["action_id",uuid]`?, `["reason_code",s]`?, `["public_reason",s]`? | empty string (`builders.rs:403-431`) |
| 5 | `build_delete_compat` | `["h",uuid]`, `["e",target]` | empty (`builders.rs:434-443`) |
| 5 | `build_remove_reaction` | `["e",reaction_id]` | empty (`builders.rs:495-498`) |
| 5 | `build_workflow_delete` | `["a","30620:<pubkey>:<workflow_uuid>"]` | empty (`builders.rs:1498-1508`) |
| 45002 | `build_vote` | `["h",uuid]`, `["e",target]` | `"+"` or `"-"` (`builders.rs:446-460`) |
| 7 | `build_reaction` | `["e",target]` | emoji string, ≤64 chars (`builders.rs:463-471`) |
| 7 | `build_custom_emoji_reaction` | `["e",target]`, `["emoji",shortcode,url]` | `":shortcode:"` (`builders.rs:479-492`) |
| 30030 | `build_custom_emoji_set` | `["d","buzz:custom-emoji"]`, then `["emoji",shortcode,url]`* | empty (`builders.rs:511-527`) |
| 40100 | `build_set_canvas` | `["h",uuid]` | canvas body, **no length check** (`builders.rs:529-532`) |
| 0 | `build_profile` | none | JSON object with only the `Some` keys among `display_name`, `name`, `picture`, `about`, `nip05` (`builders.rs:537-562`) |
| 9000 | `build_add_member` | `["h",uuid]`, `["p",hex]`, `["role",role]`? | empty (`builders.rs:565-579`) |
| 9001 | `build_remove_member` | `["h",uuid]`, `["p",hex]` | empty (`builders.rs:582-592`) |
| 9022 | `build_leave` | `["h",uuid]` | empty (`builders.rs:595-598`) |
| 9002 | `build_update_channel` | `["h",uuid]`, `["name",n]`?, `["about",a]`?, `["visibility",v]`?, `["ttl",secs\|""]`? | empty (`builders.rs:604-649`) |
| 9002 | `build_set_topic` | `["h",uuid]`, `["topic",t]` | empty (`builders.rs:652-658`) |
| 9002 | `build_set_purpose` | `["h",uuid]`, `["purpose",p]` | empty (`builders.rs:661-667`) |
| 9002 | `build_archive` / `build_unarchive` | `["h",uuid]`, `["archived","true"\|"false"]` | empty (`builders.rs:709-724`) |
| 9007 | `build_create_channel` | `["h",uuid]`, `["name",n]`, `["visibility",v]`?, `["channel_type",t]`?, `["about",a]`?, `["ttl",secs]`? | empty (`builders.rs:674-700`) |
| 9021 | `build_join` | `["h",uuid]` | empty (`builders.rs:703-706`) |
| 9008 | `build_delete_channel` | `["h",uuid]` | empty (`builders.rs:727-730`) |
| 1 | `build_note` | `["e",id,"","reply"]`? | text ≤64 KiB (`builders.rs:738-748`) |
| 3 | `build_contact_list` | `["p",hex,relay_or_"",petname_or_""]`* | empty (`builders.rs:764-813`) |
| 30617 | `build_repo_announcement` | `["d",repo_id]`, `["name",n]`?, `["description",d]`?, `["clone",url…]`?, `["web",url]`?, `["relays",url…]`? | empty (`builders.rs:834-949`) |
| 30617 | `build_repo_announcement_with_tags` | caller tags with all `d` tags removed, then `["d",repo_id]` inserted at index 0 | caller-supplied `content` (`builders.rs:952-965`) |
| 1617 | `build_git_patch` | `["a",coord]`, `["r",euc,"euc"]`?, `["p",owner]`, `["p",recipient]`*, `["e",prev,"","reply"]`?, `["t","root"]`?, `["t","root-revision"]`?, `["commit",c]`+`["r",c]`?, `["parent-commit",p]`?, `["commit-pgp-sig",s]`?, `["committer",name,email,ts,tz]`? | verbatim `git format-patch` output, ≤60 KiB, non-blank (`builders.rs:1013-1069`) |
| 1621 | `build_git_issue` | `["a",coord]`, `["p",owner]`, `["p",recipient]`*, `["subject",s]`, `["t",label]`* | markdown ≤64 KiB (`builders.rs:1081-1111`) |
| 1630/1631/1632/1633 | `build_git_status` | `["e",root,"","root"]`, `["e",rev,"","reply"]`?, `["p",recipient]`*, `["a",coord]`?, `["r",euc]`?, `["q",id[,relay[,pubkey]]]`*, `["merge-commit",c]`+`["r",c]`?, `["applied-as-commits",c…]`+`["r",c]`* | markdown ≤64 KiB (may be empty) (`builders.rs:1222-1299`) |
| 1618 | `build_git_pull_request` | `["a",coord]`, `["r",euc]`?, `["p",owner]`, `["p",recipient]`*, `["subject",s]`, `["t",label]`*, `["c",commit]`, `["h",uuid]`? (UUID-validated and canonically re-rendered — test `builders.rs:3491-3517`), `["clone",url…]`, `["branch-name",b]`?, `["merge-base",c]`?, `["e",patch]`? | markdown ≤64 KiB (`builders.rs:1330-1393`) |
| 1619 | `build_git_pr_update` | `["a",coord]`, `["r",euc]`?, `["p",owner]`, `["p",recipient]`*, `["E",pr_event]`, `["P",pr_author]`, `["c",commit]`, `["clone",url…]`, `["merge-base",c]`? | markdown ≤64 KiB (`builders.rs:1416-1460`) |
| 30620 | `build_workflow_def` / `build_workflow_update` | `["d",workflow_uuid]`, `["h",channel_uuid]` | workflow YAML ≤64 KiB (`builders.rs:1463-1494`) |
| 46020 | `build_workflow_trigger` | `["d",workflow_uuid]` | empty (`builders.rs:1511-1514`) |
| 46030 / 46031 | `build_workflow_approval` | `["d",token_hash]` | free-text note (no length check) (`builders.rs:1522-1541`) |
| 41010 | `build_dm_open` | `["p",hex]` ×1–8 | empty (`builders.rs:1544-1556`) |
| 41011 | `build_dm_add_member` | `["h",uuid]`, `["p",hex]` | empty (`builders.rs:1559-1566`) |
| 20001 | `build_presence_update` | `["status",s]` | duplicate of the status string (`builders.rs:1570-1585`) |
| 9040 | `build_moderation_ban` | `["p",hex]`, `["expiration",unix]`?, `["reason",r]`? | empty (`builders.rs:1597-1611`) |
| 9041 | `build_moderation_unban` | `["p",hex]` | empty (`builders.rs:1614-1620`) |
| 9042 | `build_moderation_timeout` | `["p",hex]`, `["expiration",unix]`, `["reason",r]`? | empty (`builders.rs:1623-1637`) |
| 9043 | `build_moderation_untimeout` | `["p",hex]` | empty (`builders.rs:1640-1646`) |
| 9044 | `build_moderation_resolve_report` | `["report",id]`, `["status",s]`, `["action",a]`, `["reason",r]`? | empty (`builders.rs:1654-1690`) |
| 9035 | `build_archive_identity_request` | `["-"]`, `["p",target]`, `["reason",code]`?, `["replaced-by",pk]`?, `["auth",owner,conditions,sig]`? | optional human text ≤64 KiB (`builders.rs:1788-1802`, tags `1739-1786`) |
| 9036 | `build_unarchive_identity_request` | `["-"]`, `["p",target]`, `["reason",code]`?, `["auth",…]`? (no `replaced-by`) | optional human text ≤64 KiB (`builders.rs:1810-1823`) |

`*` = repeatable, `?` = conditional.

Notable serde/JSON shapes:
- Kind 0 content is assembled as a `serde_json::Map` and stringified, so absent
  options are **omitted keys**, not `null` (`crates/buzz-sdk/src/builders.rs:542-561`).
- The NIP-OA `auth` tag is passed to builders as a `&[String; 4]` fixed array
  (`crates/buzz-sdk/src/builders.rs:1723-1737`), and produced/consumed by
  `nip_oa` as a **JSON array string** `["auth",owner_hex,conditions,sig_hex]`
  (`crates/buzz-sdk/src/nip_oa.rs:146-176`, `252-299`).

---

### 5. Builder ↔ kind relationships worth noting

- Kind 9002 is emitted by five distinct builders (`build_update_channel`,
  `build_set_topic`, `build_set_purpose`, `build_archive`, `build_unarchive`) —
  they differ only in tag vocabulary (`builders.rs:604-724`).
- Kind 5 is emitted by three builders with three different targeting schemes:
  `h`+`e` (`builders.rs:434`), `e` only (`builders.rs:495`), and `a` coordinate
  (`builders.rs:1498`).
- `build_workflow_def` and `build_workflow_update` are byte-identical in body
  (`builders.rs:1463-1494`).
- `extract_channel_id(&nostr::Event) -> Option<Uuid>` is the only reader
  (inverse) helper in the crate (`crates/buzz-sdk/src/builders.rs:816-826`).


## Module: buzz-persona (`crates/buzz-persona`)

### Aspect: Data Model

Scope note: this crate ships **no persona data assets**. There are no `.persona.md`,
`.toml`, `.yaml`, or `.json` files in the crate, and no `include_str!` /
`include_dir!` / `include_bytes!` macros anywhere (verified by repo-wide grep over
`crates/buzz-persona`). The only non-Rust file is `crates/buzz-persona/PERSONA_PACK_SPEC.md`
(prose spec, not compiled in). The crate is a **parser/loader/resolver library over
on-disk pack directories** — personas are data supplied by the caller, not baked in.

---

### 1. Pack-on-disk schema (what the loader reads)

`load_pack()` expects this layout, documented at `crates/buzz-persona/src/pack.rs:1-19`
and enforced by the code paths cited below:

| Path (pack-relative) | Required | Read at | Notes |
|---|---|---|---|
| `.plugin/plugin.json` | yes | `crates/buzz-persona/src/pack.rs:132-137` | Missing → `PackError::ManifestNotFound` |
| paths listed in manifest `personas[]` | yes (each must exist) | `crates/buzz-persona/src/pack.rs:155-164` | Missing → `PackError::PersonaNotFound` |
| `pack_instructions` path, else `instructions.md` | no | `crates/buzz-persona/src/pack.rs:167-188` | Explicit path missing → hard error; implicit fallback missing → `None` |
| `mcp_config` path, else `.mcp.json` | no | `crates/buzz-persona/src/pack.rs:191-220` | Explicit path missing → hard error; parsed as free-form `serde_json::Value` |
| `skills/` directory | no | `crates/buzz-persona/src/pack.rs:223-230` | Only presence is recorded (`skills_dir`); contents are not parsed at load |
| `hooks_config` path | parsed in manifest only | `crates/buzz-persona/src/manifest.rs:116` | Deliberately dropped by the loader — see `crates/buzz-persona/src/pack.rs:111-112` |

---

### 2. `PersonaConfig` — typed `.persona.md` (frontmatter + body)

`crates/buzz-persona/src/persona.rs:99-170`. Derives `Debug, Clone, Serialize, Deserialize`
with `#[serde(rename_all = "snake_case")]` (`crates/buzz-persona/src/persona.rs:99-100`).

| Field | Type | Required | serde attr | Line |
|---|---|---|---|---|
| `name` | `String` | yes | — | `persona.rs:103` |
| `display_name` | `String` | yes | — | `persona.rs:106` |
| `avatar` | `Option<String>` | no | `skip_serializing_if = "Option::is_none"` | `persona.rs:110` |
| `description` | `String` | yes | — | `persona.rs:113` |
| `version` | `Option<String>` | no | skip-if-none | `persona.rs:116` |
| `author` | `Option<String>` | no | skip-if-none | `persona.rs:119` |
| `skills` | `Vec<String>` | no | `#[serde(default)]` | `persona.rs:123` |
| `mcp_servers` | `Vec<McpServerConfig>` | no | `#[serde(default)]` | `persona.rs:127` |
| `subscribe` | `Option<Vec<String>>` | no | `default`, skip-if-none | `persona.rs:135` |
| `triggers` | `Option<RespondTo>` | no | skip-if-none | `persona.rs:139` |
| `model` | `Option<String>` | no | skip-if-none; `"provider:model-id"` | `persona.rs:143` |
| `runtime` | `Option<String>` | no | skip-if-none; e.g. `'goose'`, `'claude'` | `persona.rs:148` |
| `temperature` | `Option<f64>` | no | skip-if-none | `persona.rs:151` |
| `max_context_tokens` | `Option<u64>` | no | skip-if-none | `persona.rs:154` |
| `thread_replies` | `Option<bool>` | no | skip-if-none | `persona.rs:158` |
| `broadcast_replies` | `Option<bool>` | no | skip-if-none | `persona.rs:162` |
| `hooks` | `Option<Hooks>` | no | skip-if-none | `persona.rs:165` |
| `prompt` | `String` | — (populated from body, never from YAML) | `#[serde(default)]` | `persona.rs:169` |

Tri-state semantics for `subscribe` are documented inline at
`crates/buzz-persona/src/persona.rs:129-134`: `None` = omitted/null → fall through to
pack default; `Some(vec![])` = intentional "subscribe to nothing"; `Some([...])` = explicit.

#### Private deserialization shadow: `Frontmatter`

`crates/buzz-persona/src/persona.rs:174-196` — a private struct with
`#[serde(rename_all = "snake_case", deny_unknown_fields)]`. All identity fields are
`Option<...>` so the parser can emit `PersonaError::MissingField` instead of a serde
path error. `triggers` carries `#[serde(alias = "respond_to")]`
(`crates/buzz-persona/src/persona.rs:186`) — the legacy key alias. `prompt` is **absent**
from this struct, so a persona file cannot inject a prompt via frontmatter; the body is
the only source (`crates/buzz-persona/src/persona.rs:257`).

#### Nested persona types

`RespondTo` — `crates/buzz-persona/src/persona.rs:53-65`:

| Field | Type | serde | Line |
|---|---|---|---|
| `mentions` | `Option<bool>` | skip-if-none | `persona.rs:56` |
| `keywords` | `Vec<String>` | `#[serde(default)]` | `persona.rs:60` |
| `all_messages` | `Option<bool>` | skip-if-none | `persona.rs:64` |

`McpServerConfig` — `crates/buzz-persona/src/persona.rs:70-79`:

| Field | Type | serde | Line |
|---|---|---|---|
| `name` | `String` | required | `persona.rs:71` |
| `command` | `String` | required | `persona.rs:72` |
| `args` | `Vec<String>` | `default` | `persona.rs:75` |
| `env` | `HashMap<String, String>` | `default` | `persona.rs:78` |

`Hooks` — `crates/buzz-persona/src/persona.rs:84-93`, with
`deny_unknown_fields` (`crates/buzz-persona/src/persona.rs:83`):
`on_start: Option<String>` (`:86`), `on_stop: Option<String>` (`:89`),
`on_message: Option<String>` (`:92`). Doc comment states paths are pack-relative
(`crates/buzz-persona/src/persona.rs:81`).

#### Size constants

`MAX_FRONTMATTER_BYTES = 1_048_576` (1 MiB) — `crates/buzz-persona/src/persona.rs:21`.
`MAX_BODY_BYTES = 262_144` (256 KiB) — `crates/buzz-persona/src/persona.rs:24`.

---

### 3. `PackManifest` — typed `plugin.json`

`crates/buzz-persona/src/manifest.rs:79-121`, `rename_all = "snake_case"`
(`crates/buzz-persona/src/manifest.rs:78`).

| Field | Type | Required | Line |
|---|---|---|---|
| `id` | `String` | yes | `manifest.rs:80` |
| `name` | `String` | yes | `manifest.rs:81` |
| `version` | `String` | yes | `manifest.rs:82` |
| `description` | `Option<String>` | no | `manifest.rs:85` |
| `author` | `Option<String>` | no | `manifest.rs:88` |
| `license` | `Option<String>` | no | `manifest.rs:91` |
| `homepage` | `Option<String>` | no | `manifest.rs:94` |
| `keywords` | `Vec<String>` | no (`default`) | `manifest.rs:97` |
| `engines` | `Option<Engines>` | no | `manifest.rs:100` |
| `personas` | `Vec<String>` | no (`default` → empty) | `manifest.rs:104` |
| `pack_instructions` | `Option<String>` | no | `manifest.rs:108` |
| `mcp_config` | `Option<String>` | no | `manifest.rs:112` |
| `hooks_config` | `Option<String>` | no | `manifest.rs:116` |
| `defaults` | `Option<BehavioralDefaults>` | no | `manifest.rs:120` |

`Engines` — `crates/buzz-persona/src/manifest.rs:37-41`: single field
`buzz: Option<String>` with `alias = "buzz"` (`crates/buzz-persona/src/manifest.rs:39`).
No semver comparison logic exists in this crate.

`BehavioralDefaults` — `crates/buzz-persona/src/manifest.rs:49-70`, described as
"same shape as the persona behavioral config fields"
(`crates/buzz-persona/src/manifest.rs:44-46`):
`model` (`:51`), `temperature` (`:54`), `max_context_tokens` (`:57`),
`subscribe` (`:60`), `triggers: Option<RespondTo>` with
`alias = "respond_to"` (`:62-63`), `thread_replies` (`:66`),
`broadcast_replies` (`:69`).

`RawManifest` — private permissive mirror at `crates/buzz-persona/src/manifest.rs:132-149`.
Explicitly **no** `deny_unknown_fields`; the rationale (OPS superset may carry foreign
fields such as `ops_category`, `marketplace_tags`) is documented at
`crates/buzz-persona/src/manifest.rs:123-130`.

---

### 4. Intermediate model: `LoadedPack` / `LoadedPersona`

`LoadedPack` — `crates/buzz-persona/src/pack.rs:64-73`:

| Field | Type | Line |
|---|---|---|
| `manifest` | `PackManifestData` | `pack.rs:65` |
| `personas` | `Vec<LoadedPersona>` | `pack.rs:66` |
| `pack_instructions` | `Option<String>` (file contents) | `pack.rs:68` |
| `shared_mcp_config` | `Option<serde_json::Value>` (raw `.mcp.json`) | `pack.rs:70` |
| `skills_dir` | `Option<PathBuf>` | `pack.rs:72` |

`LoadedPersona` — `crates/buzz-persona/src/pack.rs:77-98`. This is the post-merge shape:
behavioral fields have already had pack defaults applied, so `subscribe`,
`thread_replies`, `broadcast_replies` are no longer `Option`.

| Field | Type | Line | Origin |
|---|---|---|---|
| `source_path` | `PathBuf` | `pack.rs:78` | absolute path of the `.persona.md` |
| `name` / `display_name` / `description` | `String` | `pack.rs:79-81` | frontmatter, verbatim |
| `avatar` | `Option<String>` | `pack.rs:82` | frontmatter |
| `model` | `Option<String>` | `pack.rs:83` | merged (still `provider:id` form) |
| `runtime` | `Option<String>` | `pack.rs:85` | frontmatter only, **not merged** (`pack.rs:428`) |
| `temperature` | `Option<f64>` | `pack.rs:86` | merged |
| `max_context_tokens` | `Option<u64>` | `pack.rs:87` | merged |
| `subscribe` | `Vec<String>` | `pack.rs:88` | merged, `unwrap_or_default()` at `pack.rs:432` |
| `triggers` | `Option<TriggersData>` | `pack.rs:89` | merged |
| `thread_replies` | `bool` | `pack.rs:90` | merged (default true) |
| `broadcast_replies` | `bool` | `pack.rs:91` | merged (default false) |
| `skills` | `Vec<String>` | `pack.rs:92` | frontmatter, raw paths |
| `mcp_servers` | `Vec<serde_json::Value>` | `pack.rs:94` | typed configs re-serialized to JSON (`pack.rs:415-419`) |
| `hooks` | `Option<HooksData>` | `pack.rs:95` | frontmatter |
| `prompt` | `String` | `pack.rs:97` | markdown body |

`PackManifestData` — `crates/buzz-persona/src/pack.rs:102-115`. A reduced manifest
projection: `id`, `name`, `version`, `description`, `personas`, `pack_instructions`,
`mcp_config`, and `defaults` re-encoded as `Option<serde_json::Value>` (`pack.rs:114`)
so the JSON-based merge layer can consume it. `hooks_config` is intentionally omitted
with an inline comment at `crates/buzz-persona/src/pack.rs:111-112`.

---

### 5. Merge-layer model (`merge.rs`)

Plain (non-serde) data structs used to carry merged values:

| Type | Fields | Line |
|---|---|---|
| `TriggersData` | `mentions: bool`, `keywords: Vec<String>`, `all_messages: bool` | `merge.rs:11-15` |
| `HooksData` | `on_start`/`on_stop`/`on_message`: `Option<String>` | `merge.rs:18-22` |
| `ResolvedConfig` | `model: Option<String>`, `temperature: Option<f64>`, `max_context_tokens: Option<u64>`, `subscribe: Option<Vec<String>>`, `triggers: Option<TriggersData>`, `thread_replies: bool`, `broadcast_replies: bool` | `merge.rs:25-36` |

`ResolvedConfig::subscribe` tri-state is documented at `crates/buzz-persona/src/merge.rs:29-31`.
Built-in defaults as constants: `DEFAULT_THREAD_REPLIES = true`
(`crates/buzz-persona/src/merge.rs:38`), `DEFAULT_BROADCAST_REPLIES = false`
(`crates/buzz-persona/src/merge.rs:39`). `TriggersData` sub-field built-ins are inline
literals in `parse_triggers`: `mentions` → `true` (`merge.rs:181`), `keywords` → empty
(`merge.rs:191`), `all_messages` → `false` (`merge.rs:196`).

---

### 6. Output model: `ResolvedPack` / `ResolvedPersona` (the ACP-facing contract)

Module doc states the shape is "designed backward from ACP's `Config`"
(`crates/buzz-persona/src/resolve.rs:1-14`).

`ResolvedPersona` — `crates/buzz-persona/src/resolve.rs:23-65`:

| Field | Type | Line | Derivation |
|---|---|---|---|
| `name`, `display_name`, `description` | `String` | `resolve.rs:25-27` | copied from `LoadedPersona` |
| `avatar` | `Option<String>` | `resolve.rs:28` | copied |
| `version` | `String` | `resolve.rs:29` | always the **pack** version (`resolve.rs:225-231`) |
| `system_prompt` | `String` | `resolve.rs:32` | persona markdown body only (`resolve.rs:200`) |
| `pack_instructions` | `Option<String>` | `resolve.rs:34` | trimmed; empty → `None` (`resolve.rs:201-204`) |
| `model` | `Option<String>` | `resolve.rs:37` | model id **after** colon split |
| `llm_provider` | `Option<String>` | `resolve.rs:40` | colon prefix of `model` |
| `runtime` | `Option<String>` | `resolve.rs:43` | persona `runtime` passthrough |
| `temperature` | `Option<f64>` | `resolve.rs:44` | merged value |
| `max_context_tokens` | `Option<u64>` | `resolve.rs:45` | merged value |
| `subscribe` | `Vec<String>` | `resolve.rs:48` | merged list |
| `triggers` | `ResolvedTriggers` (non-optional) | `resolve.rs:50` | `None` collapses to built-ins (`resolve.rs:255-268`) |
| `thread_replies`, `broadcast_replies` | `bool` | `resolve.rs:51-52` | merged |
| `mcp_servers` | `Vec<ResolvedMcpServer>` | `resolve.rs:55` | pack `.mcp.json` + persona list, name-keyed |
| `hooks` | `Option<ResolvedHooks>` | `resolve.rs:58` | "parsed, not executed — reserved for future use, not yet wired" (`resolve.rs:57`) |
| `skills` | `Vec<String>` | `resolve.rs:61` | "reserved for future use, not yet wired" (`resolve.rs:60`); value is `lp.skills.clone()` (`resolve.rs:249`), i.e. raw declared paths, not bare names |
| `runtime_env_vars` | `Vec<(String, String)>` | `resolve.rs:64` | projection from model/temperature/context |

`ResolvedMcpServer` — `crates/buzz-persona/src/resolve.rs:69-74`: `name`, `command`,
`args: Vec<String>`, `env: Vec<(String, String)>` (env as ordered pairs, not a map).
Doc comment records that env values are literals with no `${VAR}` interpolation
(`crates/buzz-persona/src/resolve.rs:68`).

`ResolvedHooks` — `crates/buzz-persona/src/resolve.rs:78-82`: same three optional paths.

`ResolvedTriggers` — `crates/buzz-persona/src/resolve.rs:86-90`: `mentions: bool`,
`keywords: Vec<String>`, `all_messages: bool`.

`ResolvedPack` — `crates/buzz-persona/src/resolve.rs:94-100`: `id`, `name`, `version`,
`description: String` (empty string when the manifest omits it — `resolve.rs:180`),
`personas: Vec<ResolvedPersona>`.

---

### 7. Validation report model

`ValidationDiagnostic` — `crates/buzz-persona/src/validate.rs:19-22`: enum with
`Error(String)` and `Warning(String)`; `Display` renders `ERROR: ` / `WARN:  `
prefixes (`crates/buzz-persona/src/validate.rs:24-33`).
`ValidationReport` — `crates/buzz-persona/src/validate.rs:35-37`: single field
`diagnostics: Vec<ValidationDiagnostic>`, `#[derive(Debug, Default)]`.

---

### 8. Data-flow relationship: persona → agent

The crate performs a four-stage transformation; each stage is a distinct type:

```
plugin.json  ──parse_manifest──▶ PackManifest ──▶ PackManifestData
.persona.md  ──parse_persona_md─▶ PersonaConfig
                                      │  (serialized to serde_json::Value, pack.rs:406-409)
                                      ▼
                     resolve_persona_config ──▶ ResolvedConfig ──▶ LoadedPersona
                                                                       │
                                                       resolve_one_persona (resolve.rs:194)
                                                                       ▼
                                                                ResolvedPersona
                                                          (system_prompt + env vars + MCP)
```

Notable mechanic: the merge layer is JSON-based, so `PersonaConfig` is round-tripped
through `serde_json::to_value` before merging (`crates/buzz-persona/src/pack.rs:406-409`).
This is why `PersonaConfig` carries `skip_serializing_if = "Option::is_none"` on every
optional behavioral field — omitted fields must not appear as `null` keys in the merge
input.

Relationship to agents: a persona does **not** reference an agent identity, keypair, or
Nostr pubkey anywhere in this crate. The only agent-runtime coupling is
(a) `runtime: Option<String>` selecting an env-var naming scheme
(`crates/buzz-persona/src/resolve.rs:365-397`), and (b) `runtime_env_vars`, the
key/value pairs a harness is expected to inject into the agent subprocess.


## Module: buzz-ws-client (`crates/buzz-ws-client`)

### Aspect: Data Model

Scope of files read: `Cargo.toml` (17 lines), `src/lib.rs` (9), `src/error.rs` (51),
`src/message.rs` (167), `src/connection.rs` (314). There is no `tests/` directory,
no `build.rs`, no `benches/`. No persistence layer, no serde-derived DTOs: all
types are in-memory only and constructed from/into JSON arrays by hand.

---

### 1. Connection type

`NostrWsConnection` — the only stateful type in the crate.

| Field | Type | Visibility | Purpose (as coded) | file:line |
|---|---|---|---|---|
| `ws` | `WsStream` | private | The live WebSocket stream | `crates/buzz-ws-client/src/connection.rs:27` |
| `buffer` | `VecDeque<RelayMessage>` | private | Out-of-order relay messages parked while waiting for a specific message | `crates/buzz-ws-client/src/connection.rs:28` |
| `pending_challenge` | `Option<String>` | private | Last NIP-42 `AUTH` challenge string observed on the socket | `crates/buzz-ws-client/src/connection.rs:29` |
| `relay_url` | `String` | private | The URL string passed to `connect`, reused verbatim as the `relay` tag source when building the AUTH event | `crates/buzz-ws-client/src/connection.rs:30` |

Declaration: `crates/buzz-ws-client/src/connection.rs:26`–`31`. The struct is `pub`
but **every field is private** — external mutation is only possible through the
methods in `impl NostrWsConnection` (`crates/buzz-ws-client/src/connection.rs:33`).
No `Debug`, `Clone`, `Default`, or `Drop` impl is derived or written for it.

Private type alias for the transport:

```rust
type WsStream = WebSocketStream<MaybeTlsStream<tokio::net::TcpStream>>;
```
`crates/buzz-ws-client/src/connection.rs:14` (not exported — it does not appear in
`crates/buzz-ws-client/src/lib.rs:7`).

Ownership note: `disconnect` takes `mut self` by value
(`crates/buzz-ws-client/src/connection.rs:115`), so closing consumes the
connection; all other methods take `&mut self`.

---

### 2. Connection / auth state representation

There is **no explicit state enum and no state-machine type** in this crate
(verified: no `enum` other than `RelayMessage` in `message.rs:8` and
`WsClientError` in `error.rs:5`). Auth/connection state is represented implicitly:

| Implicit state | How represented | file:line |
|---|---|---|
| Connected, challenge not yet seen | `pending_challenge: None` after construction | `crates/buzz-ws-client/src/connection.rs:62` |
| Challenge seen but not yet consumed | `pending_challenge = Some(challenge)` set opportunistically while reading | `crates/buzz-ws-client/src/connection.rs:144`, `crates/buzz-ws-client/src/connection.rs:256` |
| Challenge consumed | `pending_challenge.take()` at the top of `wait_for_auth_challenge` | `crates/buzz-ws-client/src/connection.rs:161` |
| Authenticated | Not recorded anywhere; `authenticate` returns `Ok(())` and stores no flag | `crates/buzz-ws-client/src/connection.rs:70`–`93` |

Consequence visible in code: nothing prevents `authenticate` being called twice, and
no method checks an "is authenticated" predicate before sending
(`send_event` at `crates/buzz-ws-client/src/connection.rs:96` sends unconditionally).

---

### 3. Relay message model (inbound wire types)

`RelayMessage` — `#[derive(Debug, Clone)]` (`crates/buzz-ws-client/src/message.rs:7`),
declared `crates/buzz-ws-client/src/message.rs:8`–`47`. Variant fields are named
(struct-style) except `Ok`.

| Variant | Fields (name: type) | Wire form parsed from | file:line |
|---|---|---|---|
| `Event` | `subscription_id: String`, `event: Box<Event>` | `["EVENT", <sub_id>, <event object>]` | decl `message.rs:10`–`15`; parse `message.rs:71`–`86` |
| `Ok` | `OkResponse` (tuple variant) | `["OK", <event_id>, <bool>, <message>]` | decl `message.rs:17`; parse `message.rs:87`–`104` |
| `Eose` | `subscription_id: String` | `["EOSE", <sub_id>]` | decl `message.rs:19`–`22`; parse `message.rs:105`–`114` |
| `Closed` | `subscription_id: String`, `message: String` | `["CLOSED", <sub_id>, <message>]` | decl `message.rs:24`–`29`; parse `message.rs:115`–`130` |
| `Notice` | `message: String` | `["NOTICE", <message>]` | decl `message.rs:31`–`34`; parse `message.rs:131`–`138` |
| `Auth` | `challenge: String` | `["AUTH", <challenge>]` | decl `message.rs:36`–`39`; parse `message.rs:139`–`146` |
| `Count` (**added after the original analysis**) | `subscription_id: String`, `count: u64` | `["COUNT", <sub_id>, {"count": <u64>}]` | decl `message.rs:40`–`46`; parse `message.rs:147`–`162` |

`event` is boxed (`Box<Event>`, `crates/buzz-ws-client/src/message.rs:14`;
`Box::new(event)` at `crates/buzz-ws-client/src/message.rs:84`) — the only
indirection in the enum.

Not modelled (no variant exists): `NEG-*`, or any other NIP message type.
**`COUNT` no longer belongs on that list** — the post-analysis sync added a
`Count` variant (`crates/buzz-ws-client/src/message.rs:40`–`46`) and a `"COUNT"`
parse arm (`:147`–`162`) that reads `arr[2]["count"]` as `u64`, so the crate now
models 7 inbound message types rather than 6.
Anything else is rejected as `WsClientError::UnexpectedMessage("unknown message
type: {other}")` (`crates/buzz-ws-client/src/message.rs:163`–`165`).

---

### 4. `OkResponse`

`#[derive(Debug, Clone)]` (`crates/buzz-ws-client/src/message.rs:50`), declared
`crates/buzz-ws-client/src/message.rs:51`–`58`. This is the only type with public
fields.

| Field | Type | Semantics per doc comment / parse code | file:line |
|---|---|---|---|
| `event_id` | `String` | Hex-encoded ID of the acknowledged event; taken from `arr[1]` as a string, required | decl `message.rs:53`; parse `message.rs:88`–`92` |
| `accepted` | `bool` | Relay acceptance flag; taken from `arr[2]`, **defaults to `false`** when absent or non-boolean (`unwrap_or(false)`) | decl `message.rs:55`; parse `message.rs:93` |
| `message` | `String` | Human-readable reason; taken from `arr[3]`, **defaults to `""`** when absent | decl `message.rs:57`; parse `message.rs:94`–`98` |

Matching of an `OK` to a request is by exact string comparison of `event_id`
against the hex id of the locally built/sent event
(`crates/buzz-ws-client/src/connection.rs:227`, `:254`).

---

### 5. Outbound wire values

No dedicated request/DTO types exist. Outbound frames are built ad hoc with
`serde_json::json!` and sent as text:

| Outbound value | Built at | Sent via |
|---|---|---|
| `["AUTH", <signed event>]` | `crates/buzz-ws-client/src/connection.rs:82` | `send_raw` → `Message::Text` (`connection.rs:121`–`125`) |
| `["EVENT", <signed event>]` | `crates/buzz-ws-client/src/connection.rs:98` | same |
| `Message::Pong(data)` | echo of an inbound `Ping` | `connection.rs:149`, `:209`, `:263` |
| Arbitrary `serde_json::Value` (e.g. `REQ` / `CLOSE`, composed by callers) | caller-supplied | `send_raw` (`connection.rs:121`) |

Serialization of the `nostr::Event` inside those arrays is delegated to
`serde_json` via the `nostr` crate's own `Serialize` impl
(`crates/buzz-ws-client/src/connection.rs:82`, `:98`).

---

### 6. Error model (summary; detail in the conventions/API aspects)

`WsClientError` — `#[derive(Debug, Error)]` (`crates/buzz-ws-client/src/error.rs:4`),
10 variants at `crates/buzz-ws-client/src/error.rs:5`–`45`:
`WebSocket` (`:8`, `#[from]` tungstenite), `Json` (`:12`, `#[from]` serde_json),
`EventBuilder(String)` (`:16`), `Url(String)` (`:20`), `Timeout` (`:24`),
`ConnectionClosed` (`:28`), `UnexpectedMessage(String)` (`:32`),
`AuthFailed(String)` (`:36`), `EventRejected(String)` (`:40`),
`NoAuthChallenge` (`:44`). Plus a manual `From<nostr::event::builder::Error>`
(`crates/buzz-ws-client/src/error.rs:47`–`51`).

---

### 7. Domain types borrowed from `nostr` (not defined here)

| Type | Used as | file:line |
|---|---|---|
| `nostr::Event` | payload in `RelayMessage::Event`, argument to `send_event` / `publish_event`, return of `build_auth_event` | `message.rs:1`, `message.rs:14`, `connection.rs:96`, `connection.rs:279` |
| `nostr::Keys` | signing material for the AUTH event | `connection.rs:39`, `message.rs:177` |
| `nostr::Tag` | optional extra authorization tag on the AUTH event | `connection.rs:40`, `message.rs:178` |
| `nostr::EventBuilder` | AUTH event construction | `message.rs:181` |
| `nostr::RelayUrl` | parsed relay URL embedded in the AUTH event | `message.rs:180` |

The AUTH event's kind and its standard tag set are produced by
`nostr::EventBuilder::auth` (`crates/buzz-ws-client/src/message.rs:181`) — **not
visible in this crate**, so the concrete kind integer cannot be confirmed from
these files. (For cross-reference only, outside this module: the relay side
validates `Kind::Authentication` at `crates/buzz-auth/src/nip42.rs:52` and
`crates/buzz-core/src/kind.rs:77` defines `KIND_AUTH: u32 = 22242`.)


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
| `EventQuery` | `crates/buzz-db/src/event.rs:22` | `events` (+`event_mentions`) | **19** filter fields (was 18 — `ab3af828` added `persona_reader: Option<Vec<u8>>` at `:88`) incl. `community_id`, `kinds`, `authors`, `ids`, `e_tags`, `d_tags`, `before_id`, `channel_ids`, `global_only`, `max_limit`, `persona_reader` |
| `ReactionEventInsertOutcome` | `crates/buzz-db/src/event.rs:124` | — | `TargetMissing` \| `Duplicate` \| `Inserted{stored_event, was_inserted}` |
| `ThreadMetadataParams<'a>` | `crates/buzz-db/src/event.rs:1024` | `thread_metadata` | insert params |
| `DueReminder` | `crates/buzz-db/src/event.rs:1306` | `events` + `communities` | `community_id`, `host`, `id`, `pubkey`, `created_at`, `kind`, `tags`, `content`, `sig`, `channel_id` |
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


## Module: buzz-auth (`crates/buzz-auth`)

### Data Model

All types below are defined in this crate. There is no persistence layer here —
no SQL, no ORM types. Types are either in-memory auth results, config structs
(serde-deserialised by the caller), or key-format helpers for Redis strings
produced elsewhere.

---

### `AuthMethod` — enum (`crates/buzz-auth/src/lib.rs:55-60`)

Derives: `Debug, Clone, PartialEq, Eq` (`crates/buzz-auth/src/lib.rs:54`).

| Variant | Meaning (from doc comment) | file:line |
|---------|----------------------------|-----------|
| `Nip42` | NIP-42 challenge/response — Schnorr signature over kind:22242 | `crates/buzz-auth/src/lib.rs:57` |
| `Nip98` | NIP-98 HTTP Auth — Schnorr signature over kind:27235 | `crates/buzz-auth/src/lib.rs:59` |

Note: `AuthMethod::Nip98` is never constructed inside this crate — the only
producer of `AuthContext` here (`AuthService::verify_auth_event`) hardcodes
`AuthMethod::Nip42` (`crates/buzz-auth/src/lib.rs:140`). `verify_nip98_event`
returns a bare `nostr::PublicKey`, not an `AuthContext`
(`crates/buzz-auth/src/nip98.rs:60`).

---

### `AuthContext` — struct (`crates/buzz-auth/src/lib.rs:64-80`)

Derives: `Debug, Clone` (`crates/buzz-auth/src/lib.rs:63`). Not `PartialEq`, not
`Serialize`/`Deserialize`.

| Field | Type | Semantics | file:line |
|-------|------|-----------|-----------|
| `pubkey` | `nostr::PublicKey` | The authenticated Nostr public key | `crates/buzz-auth/src/lib.rs:66` |
| `scopes` | `Vec<Scope>` | Permission scopes granted to this connection | `crates/buzz-auth/src/lib.rs:68` |
| `channel_ids` | `Option<Vec<uuid::Uuid>>` | Channel restriction; doc says "reserved for future per-channel access control", `None` means unrestricted | `crates/buzz-auth/src/lib.rs:72` |
| `auth_method` | `AuthMethod` | How the connection was authenticated | `crates/buzz-auth/src/lib.rs:74` |
| `agent_owner_pubkey` | `Option<nostr::PublicKey>` | NIP-OA verified owner pubkey; `None` for direct relay members / non-NIP-OA paths. Doc: "Set by the relay membership gate when NIP-OA fallback succeeds." | `crates/buzz-auth/src/lib.rs:79` |

Inherent method: `has_scope(&self, scope: &Scope) -> bool` — a `Vec::contains`
lookup (`crates/buzz-auth/src/lib.rs:84-86`).

`channel_ids` is set to `None` at every construction site found in the repo:
`crates/buzz-auth/src/lib.rs:139`, `crates/buzz-auth/src/lib.rs:191` (test), and
`crates/buzz-relay/src/handlers/event.rs:1370`.

---

### `Scope` — enum (`crates/buzz-auth/src/scope.rs:16-61`)

Derives: `Debug, Clone, PartialEq, Eq, Hash` (`crates/buzz-auth/src/scope.rs:15`).
Not `Serialize`/`Deserialize`; wire form goes through `as_str`/`FromStr`.

17 variants total: 16 known + 1 catch-all. Wire strings from `Scope::as_str`
(`crates/buzz-auth/src/scope.rs:114-134`) and `FromStr`
(`crates/buzz-auth/src/scope.rs:146-166`) — the two tables are exact inverses.

| # | Variant | Wire string | In `all_known()` | In `all_non_admin()` | Variant decl |
|---|---------|-------------|------------------|----------------------|--------------|
| 1 | `MessagesRead` | `messages:read` | yes | yes | `scope.rs:18` |
| 2 | `MessagesWrite` | `messages:write` | yes | yes | `scope.rs:20` |
| 3 | `ChannelsRead` | `channels:read` | yes | yes | `scope.rs:22` |
| 4 | `ChannelsWrite` | `channels:write` | yes | yes | `scope.rs:24` |
| 5 | `AdminChannels` | `admin:channels` | yes | **no** | `scope.rs:26` |
| 6 | `UsersRead` | `users:read` | yes | yes | `scope.rs:28` |
| 7 | `UsersWrite` | `users:write` | yes | yes | `scope.rs:30` |
| 8 | `AdminUsers` | `admin:users` | yes | **no** | `scope.rs:32` |
| 9 | `JobsRead` | `jobs:read` | yes | yes | `scope.rs:34` |
| 10 | `JobsWrite` | `jobs:write` | yes | yes | `scope.rs:36` |
| 11 | `SubscriptionsRead` | `subscriptions:read` | yes | yes | `scope.rs:38` |
| 12 | `SubscriptionsWrite` | `subscriptions:write` | yes | yes | `scope.rs:40` |
| 13 | `FilesRead` | `files:read` | yes | yes | `scope.rs:42` |
| 14 | `FilesWrite` | `files:write` | yes | yes | `scope.rs:44` |
| 15 | `ReposRead` | `repos:read` | yes | yes | `scope.rs:50` |
| 16 | `ReposWrite` | `repos:write` | yes | yes | `scope.rs:56` |
| 17 | `Unknown(String)` | the inner string verbatim | no | no | `scope.rs:60` |

Counts are pinned by tests: `all_known()` = **16**
(`crates/buzz-auth/src/scope.rs:237`), `all_non_admin()` = **14**
(`crates/buzz-auth/src/scope.rs:208`).

`ReposRead` / `ReposWrite` carry doc caveats about non-enforcement:
`ReposRead` — "Not currently enforced by git HTTP routes"
(`crates/buzz-auth/src/scope.rs:47-49`); `ReposWrite` — "Enforced for
kind:30617/30618 events via WebSocket ingest, but NOT enforced by git HTTP push
routes" (`crates/buzz-auth/src/scope.rs:52-55`). Both are crate-doc claims about
code in `buzz-relay`, not verified here.

Trait impls: `fmt::Display` delegates to `as_str`
(`crates/buzz-auth/src/scope.rs:137-141`); `FromStr` with
`type Err = std::convert::Infallible` (`crates/buzz-auth/src/scope.rs:143-167`)
— unknown strings become `Unknown(_)` rather than an error.

Persistence note from the doc comment: "Scopes are stored as `TEXT[]` in the
database so new variants can be added without schema migrations"
(`crates/buzz-auth/src/scope.rs:12-14`). No DB code in this crate confirms that.

---

### `AuthConfig` — struct (`crates/buzz-auth/src/lib.rs:91-95`)

Derives: `Debug, Clone, Default, serde::Serialize, serde::Deserialize`
(`crates/buzz-auth/src/lib.rs:90`).

| Field | Type | serde | file:line |
|-------|------|-------|-----------|
| `rate_limits` | `RateLimitConfig` | `#[serde(default)]` | `crates/buzz-auth/src/lib.rs:93-94` |

Doc: "typically loaded from the relay's TOML config file"
(`crates/buzz-auth/src/lib.rs:89`). In practice `buzz-relay` builds it from env
vars (`crates/buzz-relay/src/config.rs:585`).

---

### `AuthService` — struct (`crates/buzz-auth/src/lib.rs:100-102`)

Derives `Debug, Clone` (`crates/buzz-auth/src/lib.rs:99`). Single private field
`config: AuthConfig` (`crates/buzz-auth/src/lib.rs:101`). Holds no keys, no
connections, no caches.

---

### `RateLimitConfig` — struct (`crates/buzz-auth/src/rate_limit.rs:86-108`)

Derives: `Debug, Clone, Serialize, Deserialize`
(`crates/buzz-auth/src/rate_limit.rs:85`). Manual `Default` impl
(`crates/buzz-auth/src/rate_limit.rs:132-144`) — values identical to the serde
default fns.

Four tiers (human, agent-standard, agent-elevated, agent-platform) across seven
fields:

| Tier | Field | Type | Default | Default fn | Field decl |
|------|-------|------|---------|-----------|------------|
| human | `human_messages_per_min` | `u64` | 60 | `default_human_msg` (`rate_limit.rs:110-112`) | `rate_limit.rs:89` |
| human | `human_api_calls_per_min` | `u64` | 300 | `default_human_api` (`rate_limit.rs:113-115`) | `rate_limit.rs:92` |
| human | `human_ws_events_per_sec` | `u64` | 10 | `default_human_ws` (`rate_limit.rs:116-118`) | `rate_limit.rs:95` |
| agent-standard | `agent_standard_messages_per_min` | `u64` | 120 | `default_agent_std_msg` (`rate_limit.rs:119-121`) | `rate_limit.rs:98` |
| agent-standard | `agent_standard_api_calls_per_min` | `u64` | 600 | `default_agent_std_api` (`rate_limit.rs:122-124`) | `rate_limit.rs:101` |
| agent-elevated | `agent_elevated_messages_per_min` | `u64` | 300 | `default_agent_elev_msg` (`rate_limit.rs:125-127`) | `rate_limit.rs:104` |
| agent-platform | `agent_platform_messages_per_min` | `u64` | 600 | `default_agent_plat_msg` (`rate_limit.rs:128-130`) | `rate_limit.rs:107` |

Every field carries `#[serde(default = "...")]`, so a partial TOML/JSON section
deserialises successfully with per-field fallbacks.

---

### `LimitType` — enum (`crates/buzz-auth/src/rate_limit.rs:58-67`)

Derives: `Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize` with
`#[serde(rename_all = "snake_case")]`
(`crates/buzz-auth/src/rate_limit.rs:56-57`).

| Variant | serde string | `key_suffix()` | Doc meaning | file:line |
|---------|--------------|----------------|-------------|-----------|
| `Messages` | `messages` | `msg` | Nostr message events sent via WebSocket | `rate_limit.rs:60` / suffix `:72` |
| `ApiCalls` | `api_calls` | `api` | HTTP REST API calls | `rate_limit.rs:62` / suffix `:73` |
| `WsEvents` | `ws_events` | `ws` | All WebSocket events (broader than `Messages`) | `rate_limit.rs:64` / suffix `:74` |
| `IpConnections` | `ip_connections` | `conn` | Concurrent WS connections from a single IP | `rate_limit.rs:66` / suffix `:75` |

`key_suffix(&self) -> &'static str` at `crates/buzz-auth/src/rate_limit.rs:71-78`.

---

### `RateLimitResult` — struct (`crates/buzz-auth/src/rate_limit.rs:19-28`)

Derives `Debug, Clone, PartialEq, Eq` (`crates/buzz-auth/src/rate_limit.rs:18`).

| Field | Type | Meaning | file:line |
|-------|------|---------|-----------|
| `allowed` | `bool` | permitted (`true`) vs reject (`false`) | `rate_limit.rs:21` |
| `current` | `u64` | counter value after this increment | `rate_limit.rs:23` |
| `limit` | `u64` | configured limit for this window | `rate_limit.rs:25` |
| `reset_in_secs` | `u64` | seconds until window reset | `rate_limit.rs:27` |

Constructors: `RateLimitResult::allowed(current, limit, reset_in_secs)`
(`crates/buzz-auth/src/rate_limit.rs:32-39`) and
`RateLimitResult::denied(...)` (`crates/buzz-auth/src/rate_limit.rs:42-49`).

---

### `AuthError` — enum (`crates/buzz-auth/src/error.rs:9-59`)

See the conventions aspect for the full variant/message table. Derives
`Debug, thiserror::Error` (`crates/buzz-auth/src/error.rs:8`). One variant is
structured: `InsufficientScope { required: String, have: Vec<String> }`
(`crates/buzz-auth/src/error.rs:45-50`).

---

### Challenge / token types

There is **no** challenge struct and **no** token type in this crate.

- The NIP-42 challenge is a bare `String`: `generate_challenge() -> String`,
  32 CSPRNG bytes hex-encoded (`crates/buzz-auth/src/nip42.rs:38-41`). The test
  pins 64 hex chars (`crates/buzz-auth/src/nip42.rs:103-109`). Challenge state
  lives in the caller (`AuthState::Pending { challenge }` in
  `crates/buzz-relay/src/connection.rs:44` / `handlers/auth.rs:48`).
- No JWT, opaque token, refresh token, or API-token struct exists. The crate doc
  states this explicitly: "No JWT validation, no token management, no IdP runtime
  dependency" (`crates/buzz-auth/src/lib.rs:16`, repeated at
  `crates/buzz-auth/src/lib.rs:98`).
- The NIP-98 credential is a raw JSON string parsed into `nostr::Event` inside
  `verify_nip98_event` (`crates/buzz-auth/src/nip98.rs:62`); the crate never
  defines a wrapper type for it.

---

### Replay-guard data (`crates/buzz-auth/src/nip98_replay.rs`)

No struct — the seen-set is modelled entirely as the `Nip98ReplayGuard` trait
plus two key-format functions and two constants.

| Constant | Value | Purpose | file:line |
|----------|-------|---------|-----------|
| `DEFAULT_REPLAY_TTL_SECS` | `120` | Floor for the replay window (2× the ±60s NIP-98 tolerance) | `nip98_replay.rs:46` |
| `MAX_REPLAY_TTL_SECS` | `3600` | Ceiling; implementations MUST clamp down to it | `nip98_replay.rs:59` |

Key formats:

| Function | Format | file:line |
|----------|--------|-----------|
| `nip98_replay_key(ctx, event_id)` | delegates to the scope form with `ctx.community().to_string()` | `nip98_replay.rs:114-116` |
| `nip98_replay_key_for_scope(scope, event_id)` | `buzz:{scope}:nip98:{event_id_hex}` | `nip98_replay.rs:119-121` |
| `rate_limit_key(ctx, pubkey, limit_type)` | `buzz:{community}:ratelimit:{pubkey_hex}:{suffix}` | `rate_limit.rs:201-208` |
| `ip_rate_limit_key(ip)` | `buzz:ratelimit:ip:{ip}:conn` (no community prefix — operator-global) | `rate_limit.rs:213-215` |

`MockAccessChecker` holds one in-memory field:
`allowed: HashSet<(uuid::Uuid, String, Uuid)>` keyed on
`(community_id, pubkey_hex, channel_id)`
(`crates/buzz-auth/src/access.rs:108-110`, insert at `:123-125`) — cfg-gated to
`any(test, feature = "test-utils")` (`crates/buzz-auth/src/access.rs:107`).

---

### External types referenced (not defined here)

| Type | Origin | Used for | file:line |
|------|--------|----------|-----------|
| `nostr::PublicKey` | `nostr` | authenticated identity | `lib.rs:66`, `nip98.rs:60` |
| `nostr::Event` | `nostr` | AUTH / HTTP-auth events | `nip42.rs:48`, `nip98.rs:62` |
| `nostr::EventId` | `nostr` | replay seen-set key | `nip98_replay.rs:37` |
| `nostr::Timestamp` | `nostr` | clock-skew checks | `nip42.rs:78`, `nip98.rs:78` |
| `buzz_core::TenantContext` | `buzz-core` | community scoping on access + rate-limit + replay | `access.rs:9`, `rate_limit.rs:11`, `nip98_replay.rs:36` |
| `uuid::Uuid` | `uuid` | channel ids | `access.rs:11`, `lib.rs:72` |
| `std::net::IpAddr` | std | IP-keyed limits | `rate_limit.rs:9` |


## Module: buzz-pubsub (`crates/buzz-pubsub`)
### Aspect: Data Model

Source of truth read: `crates/buzz-pubsub/Cargo.toml` (26 lines) plus all 10 `.rs`
files — `src/lib.rs` (629), `src/subscriber.rs` (205), `src/cache_invalidation.rs`
(251), `src/conn_control.rs` (229), `src/presence.rs` (209), `src/nip98_replay.rs`
(202), `src/topic.rs` (197), `src/rate_limiter.rs` (121), `src/error.rs` (38),
`src/publisher.rs` (37). Total 2,144 lines (2,118 Rust).

The crate owns **no SQL, no migrations, and no persistent schema**. Its data model
is (1) a Redis keyspace, (2) in-process routing state, and (3) three JSON wire
payloads. It declares no `buzz-db` dependency (`Cargo.toml:11-24`).

---

### 1. Redis keyspace owned by this crate

All keys are prefixed by `BUZZ_PREFIX = "buzz"` (`topic.rs:13`). Community scoping
is achieved by embedding `ctx.community()` as the second path segment — key naming
is the *only* tenant separation mechanism (no separate Redis db index, no ACL).

| Key / channel | Redis type | TTL | Built at | Notes |
|---|---|---|---|---|
| `buzz:{community}:channel:{channel_uuid}` | pub/sub channel | n/a | `topic.rs:45-48` | Exact-channel event fan-out |
| `buzz:{community}:global` | pub/sub channel | n/a | `topic.rs:49` | Community-global events not routed to one channel |
| `buzz:{community}:cache-invalidate` | pub/sub channel | n/a | `cache_invalidation.rs:30-35` | Cross-pod cache-key drops |
| `buzz:{community}:conn-control` | pub/sub channel | n/a | `conn_control.rs:33-35` | Cross-pod disconnect commands |
| `buzz:{community}:presence:{pubkey_hex}` | string | `EX 90` | `presence.rs:19-25`, TTL const `presence.rs:16` | Value is a free-form status string |
| `buzz:{community}:ratelimit:{pubkey_hex}:{suffix}` | integer counter | `EX window_secs` | key from `buzz_auth::rate_limit::rate_limit_key` (`rate_limiter.rs:110`); format documented `rate_limiter.rs:82-84` | Fixed-window counter |
| `buzz:ratelimit:ip:{ip}:conn` | integer counter | `EX window_secs` | `buzz-auth/src/rate_limit.rs:213`, asserted `:312`; called from `rate_limiter.rs:118` | **Not community-scoped** — operator-global by design (`buzz-auth/src/rate_limit.rs:158`) |
| `buzz:{community}:nip98:{event_id_hex}` | string `"1"` | `EX ttl` (clamped) | `nip98_replay.rs:19`, key via `nip98_replay_key_for_scope` (`nip98_replay.rs:81`) | Replay seen-set marker |

Two subscribers use **cross-tenant wildcard patterns**, not per-community
subscriptions:
- `CACHE_INVALIDATION_PATTERN = "buzz:*:cache-invalidate"` (`cache_invalidation.rs:27`), `psubscribe` at `cache_invalidation.rs:139`
- `CONN_CONTROL_PATTERN = "buzz:*:conn-control"` (`conn_control.rs:30`), `psubscribe` at `conn_control.rs:130`

So every pod receives every community's invalidation and disconnect traffic and
demultiplexes by parsing the channel name (`parse_cache_invalidation_channel`
`cache_invalidation.rs:38-51`; `parse_conn_control_channel` `conn_control.rs:38-51`).
Event fan-out is the opposite: dynamically scoped `subscribe`/`unsubscribe` on the
exact topic keys with local interest (`subscriber.rs:96-100`, `:116-131`).

### 2. Key-parsing contract

`EventTopicKey::parse_redis_channel` (`topic.rs:53-99`) is a strict 3-or-4 segment
parser: prefix must equal `buzz` (`:58-63`), segment 2 must be a parseable UUID
(`:68-71`), segment 3 must be exactly `global` (no 4th segment) or `channel` plus a
UUID (no 5th segment) (`:77-95`). Anything else yields
`PubSubError::InvalidChannelKey`. Notably it **rejects** `presence:` keys — asserted
explicitly (`topic.rs:175`), which is what keeps the presence keyspace from ever
being interpreted as an event topic.

### 3. In-process structures

| Type | Definition | Shape / notes |
|---|---|---|
| `EventTopic` | `topic.rs:17-22` | `Channel(Uuid)` \| `Global`. `Copy`, `Hash`, `Eq` (`topic.rs:16`) |
| `EventTopicKey` | `topic.rs:26-31` | `{ community_id: CommunityId, topic: EventTopic }` — the composite routing identity |
| `ChannelEvent` | `lib.rs:62-69` | `{ community_id, topic, event: nostr::Event }` — the broadcast payload |
| `PubSubConfig` | `lib.rs:73-78` | `{ redis_url: String, unsubscribe_debounce: Duration }` |
| `PubSubManager` | `lib.rs:100-113` | 9 fields: pool, `redis_url`, `unsubscribe_debounce`, `desired_topics`, `subscription_tx`, `subscription_rx: Mutex<Option<..>>`, and 3 broadcast senders |
| `DesiredTopics` | `subscriber.rs:21` | `Arc<Mutex<HashMap<EventTopicKey, usize>>>` — refcount map, declared "source of truth across Redis reconnects" (`lib.rs:108`) |
| `SubscriptionCommand` | `subscriber.rs:26-31` | `pub(crate)`; `Subscribe(EventTopicKey)` \| `UnsubscribeIfIdle(EventTopicKey)` |
| `active_topics` | `subscriber.rs:88` | `HashSet<String>` of Redis channel names, **connection-local** — rebuilt from `desired_topics` on every reconnect (`subscriber.rs:90-101`) |
| `RedisRateLimiter` | `rate_limiter.rs:88-90` | Single field: `pool` |
| `RedisNip98ReplayGuard` | `nip98_replay.rs:23-25` | Single field: `pool` |

Channel capacities are all hardcoded to 4096 with no configuration knob: three
`broadcast::channel(4096)` (`lib.rs:126-128`) and one `mpsc::channel(4096)`
(`lib.rs:129`).

`PubSubManager` derives nothing and is **not `Clone`** (`lib.rs:100`); the three
`run_*_subscriber` methods take `self: Arc<Self>` (`lib.rs:148`, `:165`, `:175`), so
shared ownership is via `Arc` only.

### 4. Wire payloads

| Payload | Serialization | Definition |
|---|---|---|
| Event fan-out | `nostr::Event` JSON via `event.as_json()` (`publisher.rs:31`), decoded by `nostr::Event::from_json` (`subscriber.rs:151`) | Full signed Nostr event, not a delta |
| `CacheInvalidation` | `serde_json`, internally tagged `#[serde(tag = "op")]` (`cache_invalidation.rs:57`) | 4 variants: `Membership { channel_id: Uuid, pubkey: Vec<u8> }`, `AccessibleAll`, `Visibility { channel_id }`, `ChannelDeleted` (`cache_invalidation.rs:58-80`) |
| `ConnControl` | `serde_json`, internally tagged `#[serde(tag = "op")]` (`conn_control.rs:55`) | 2 variants: `DisconnectCommunity`, `DisconnectPubkey { pubkey: Vec<u8>, event_id: String, reason: String }` (`conn_control.rs:56-73`) |

Both control enums are wrapped for delivery with their parsed community:
`ScopedCacheInvalidation { community_id, invalidation }` (`cache_invalidation.rs:83-88`)
and `ScopedConnControl { community_id, command }` (`conn_control.rs:75-80`).

Neither control payload carries a version field, a timestamp, a nonce, or an
origin-pod id. Unknown `op` values fail deserialization and are skipped with a
`warn` (`cache_invalidation.rs:159-165`, `conn_control.rs:150-156`); a test pins
that an unknown command does not poison later messages (`conn_control.rs:209-217`).
`pubkey` is declared as "32 raw bytes" in the doc (`conn_control.rs:60`) but typed
`Vec<u8>` with **no length validation** on either send or receive.

### 5. Error model

`PubSubError` (`error.rs:5-28`) — 6 variants: `Redis` (from `redis::RedisError`),
`Pool` (from `deadpool_redis::PoolError`), `Serialization` (from
`serde_json::Error`), `BroadcastLagged(u64)`, `SubscriberStopped`,
`InvalidChannelKey(String)`. A `From<broadcast::error::RecvError>` impl maps
`Lagged(n) → BroadcastLagged(n)` and `Closed → SubscriberStopped`
(`error.rs:31-38`). The rate limiter and replay guard do **not** use this type —
they return `buzz_auth::error::AuthError` to satisfy the borrowed traits
(`rate_limiter.rs:35`, `nip98_replay.rs:37`).


## Module: buzz-search (`crates/buzz-search`)

### Data Model

The crate defines no persistent schema of its own. It declares five public types
(one service handle, two enums, three structs) and reads one table (`events`).
There are no `sqlx::FromRow` derives; rows are decoded field-by-field via
`Row::try_get` (`crates/buzz-search/src/query.rs:304-319`).

#### Files

| File | LOC | Contents |
|---|---|---|
| `crates/buzz-search/src/lib.rs` | 54 | crate lints, re-exports, `SearchService` |
| `crates/buzz-search/src/query.rs` | 352 | all query types, SQL construction, `search()`, 3 unit tests |
| `crates/buzz-search/src/error.rs` | 9 | `SearchError` |
| `crates/buzz-search/tests/fts_integration.rs` | 1448 | 18 Postgres-gated integration tests |

---

### `SearchService` — `crates/buzz-search/src/lib.rs:39-42`

| Field | Type | Visibility | Notes |
|---|---|---|---|
| `pool` | `sqlx::PgPool` | private | Only field. `#[derive(Debug, Clone)]` at `lib.rs:39`. |

Doc comment states it "Holds nothing the pool itself doesn't already own" and
exists as an injection point for the relay's `AppState`
(`crates/buzz-search/src/lib.rs:35-38`).

---

### `SearchQuery` — `crates/buzz-search/src/query.rs:73-99`

`#[derive(Debug, Clone)]` (`query.rs:73`). All fields `pub`.

| Field | Type | Line | Semantics as documented/used |
|---|---|---|---|
| `community` | `buzz_core::CommunityId` | `query.rs:76` | Server-resolved tenant. Required at the type level (no `Option`). Bound as the first SQL predicate (`query.rs:241`). |
| `q` | `String` | `query.rs:79` | NIP-50 search text. Empty/whitespace short-circuits before SQL (`query.rs:217-222`). |
| `channel_scope` | `ChannelScope` | `query.rs:84` | Channel constraint; see enum below. |
| `kinds` | `Option<Vec<i32>>` | `query.rs:86` | `None` **or empty vec** = no kind constraint (`query.rs:267-273`). |
| `authors` | `Option<Vec<Vec<u8>>>` | `query.rs:88` | 32-byte pubkeys as raw bytes. `None`/empty = no constraint (`query.rs:275-281`). Type does not enforce the 32-byte length. |
| `since` | `Option<i64>` | `query.rs:90` | Unix seconds; inclusive lower bound (`>=`, `query.rs:284`). |
| `until` | `Option<i64>` | `query.rs:92` | Unix seconds; inclusive upper bound (`<=`, `query.rs:290`). |
| `page` | `u32` | `query.rs:94` | 1-indexed; clamped to `1..=1000` (`query.rs:230`). |
| `per_page` | `u32` | `query.rs:96` | Clamped to `1..=500`; `0` becomes `100` (`query.rs:224-229`). |
| `mode` | `SearchMode` | `query.rs:98` | Selects the tsquery construction (`query.rs:239`, `140-177`). |

No `Default`, no builder, no constructor fn — callers struct-literal every field
(e.g. `crates/buzz-relay/src/handlers/req.rs:599-611`,
`crates/buzz-relay/src/api/bridge.rs:1687-1698`).

---

### `SearchHit` — `crates/buzz-search/src/query.rs:104-118`

`#[derive(Debug, Clone)]`. All fields `pub`.

| Field | Type | Line | Source column |
|---|---|---|---|
| `event_id` | `[u8; 32]` | `query.rs:107` | `events.id` (BYTEA), length-checked to 32 (`query.rs:306-308`) |
| `kind` | `i32` | `query.rs:109` | `events.kind` (`query.rs:314`) |
| `pubkey` | `[u8; 32]` | `query.rs:111` | `events.pubkey` (BYTEA), length-checked to 32 (`query.rs:309-311`) |
| `channel_id` | `Option<Uuid>` | `query.rs:113` | `events.channel_id`; `None` = channel-less (`query.rs:316`) |
| `created_at` | `i64` | `query.rs:115` | `EXTRACT(EPOCH FROM created_at)::bigint AS created_at_s` (`query.rs:235`, `317`) |
| `rank` | `f32` | `query.rs:117` | `ts_rank_cd(search_tsv, search_query.query) AS rank` (`query.rs:236`, `318`) |

Notably absent: `content`, `tags`, `sig`, `d_tag`. The struct is documented as
"just enough to drive that fetch and preserve relevance ordering"
(`query.rs:101-103`).

---

### `SearchResult` — `crates/buzz-search/src/query.rs:121-127`

| Field | Type | Line | Notes |
|---|---|---|---|
| `hits` | `Vec<SearchHit>` | `query.rs:124` | Page of hits, relevance-then-recency ordered |
| `page` | `u32` | `query.rs:126` | Clamped page actually used (`query.rs:230`, `322`); also set on the empty-query early return (`query.rs:220`) |

There is **no total-count / `found` field** — the crate returns no result cardinality
beyond the page itself.

---

### `ChannelScope` — `crates/buzz-search/src/query.rs:41-53`

`#[derive(Debug, Clone)]` (no `Copy`, no `PartialEq`).

| Variant | Line | Payload | Emitted SQL fragment | Emit site |
|---|---|---|---|---|
| `Any` | `query.rs:44` | — | *(nothing)* | `query.rs:249-251` |
| `ChannelLessOnly` | `query.rs:47` | — | `AND channel_id IS NULL` | `query.rs:252-254` |
| `Channels` | `query.rs:49` | `Vec<Uuid>` | `AND channel_id = ANY($n)` | `query.rs:255-259` |
| `ChannelsOrChannelLess` | `query.rs:52` | `Vec<Uuid>` | `AND (channel_id = ANY($n) OR channel_id IS NULL)` | `query.rs:260-264` |

Empty vectors are deliberately not special-cased: `Channels(vec![])` yields
zero rows and `ChannelsOrChannelLess(vec![])` is equivalent to
`ChannelLessOnly` (`query.rs:33-39`).

---

### `SearchMode` — `crates/buzz-search/src/query.rs:56-66`

`#[derive(Debug, Clone, Copy, PartialEq, Eq)]` (`query.rs:56`).

| Variant | Line | tsquery construction |
|---|---|---|
| `FullText` | `query.rs:59` | `websearch_to_tsquery('simple', $n)` (`query.rs:142-146`) |
| `Prefix` | `query.rs:65` | In-SQL token pipeline: `regexp_split_to_table($n, '\s+') WITH ORDINALITY` → `to_tsvector('simple', token)` → `tsvector_to_array` → `unnest ... WITH ORDINALITY` → `string_agg(quote_literal(lexeme) || CASE WHEN token_ord = max_token_ord THEN ':*' ELSE '' END, ' & ' ORDER BY token_ord, lex_ord)::tsquery`, `COALESCE`d to `''` (`query.rs:147-176`) |

---

### `SearchError` — `crates/buzz-search/src/error.rs:4-9`

| Variant | Line | Shape | `Display` |
|---|---|---|---|
| `Db` | `error.rs:8` | `Db(#[from] sqlx::Error)` | `"database error: {0}"` (`error.rs:7`) |

Single-variant enum; row-decode length failures are folded into it as
`sqlx::Error::Decode` (`query.rs:306-311`).

---

### SQL shape queried against `events`

One statement, built with `sqlx::QueryBuilder` (`query.rs:233-298`). Literal
text as constructed:

```sql
SELECT id, kind, pubkey, channel_id,
       EXTRACT(EPOCH FROM created_at)::bigint AS created_at_s,
       ts_rank_cd(search_tsv, search_query.query) AS rank
FROM events CROSS JOIN LATERAL (SELECT <mode tsquery> AS query) AS search_query
WHERE community_id = $1
  AND deleted_at IS NULL
  AND search_tsv @@ search_query.query
  [AND <channel scope>] [AND kind = ANY($n)] [AND pubkey = ANY($n)]
  [AND created_at >= to_timestamp($n)] [AND created_at <= to_timestamp($n)]
ORDER BY rank DESC, created_at DESC, id
LIMIT $n OFFSET $n
```

Columns referenced on `events`:

| Column | Role | Reference |
|---|---|---|
| `id` | selected, tiebreak in `ORDER BY` | `query.rs:234`, `295` |
| `kind` | selected + optional predicate | `query.rs:234`, `269` |
| `pubkey` | selected + optional predicate | `query.rs:234`, `277` |
| `channel_id` | selected + channel-scope predicate | `query.rs:234`, `253-263` |
| `created_at` | selected (epoch cast), since/until predicates, ordering | `query.rs:235`, `284`, `290`, `295` |
| `search_tsv` | rank input + `@@` probe | `query.rs:236`, `242` |
| `community_id` | tenant predicate (first) | `query.rs:240-241` |
| `deleted_at` | soft-delete exclusion | `query.rs:242` |

Underlying storage (read for confirmation only, not owned by this crate):
`events` is `PARTITION BY RANGE (created_at)` with PK
`(community_id, created_at, id)` (`migrations/0001_initial_schema.sql:191-238`),
`search_tsv` is `TSVECTOR GENERATED ALWAYS ... STORED`
(`migrations/0001_initial_schema.sql:222-226`), and the access path is
`CREATE INDEX idx_events_search_tsv ON events USING GIN (search_tsv)`
(`migrations/0001_initial_schema.sql:278`).


## Module: buzz-audit (`crates/buzz-audit`)

### Aspect: Data Model

Source of truth read: `crates/buzz-audit/Cargo.toml`, `src/lib.rs`, `src/action.rs`,
`src/entry.rs`, `src/error.rs`, `src/hash.rs`, `src/service.rs` (all 6 `.rs` files,
1,114 non-blank / 1,136 total lines). The crate ships **no DDL** — the `audit_log`
table is owned by migration `0001` (`crates/buzz-audit/src/lib.rs:17-18`).

---

### 1. `AuditEntry` — materialised stored row (`crates/buzz-audit/src/entry.rs:14-37`)

Derives `Debug, Clone, PartialEq, Eq, Serialize, Deserialize` (`entry.rs:13`).

| Field | Rust type | Nullability | Notes (file:line) |
|---|---|---|---|
| `community_id` | `uuid::Uuid` (raw, **not** `CommunityId`) | required | "Leads the primary key" — `entry.rs:16` |
| `seq` | `i64` | required | monotonic **within** `community_id`, starts at 1 — `entry.rs:18` |
| `hash` | `Vec<u8>` | required | SHA-256 of this entry incl. `community_id` + `prev_hash` — `entry.rs:20` |
| `prev_hash` | `Option<Vec<u8>>` | nullable | `None` for the community's first entry; hashed as `GENESIS_HASH` — `entry.rs:21-23` |
| `action` | `AuditAction` | required | `entry.rs:25` |
| `actor_pubkey` | `Option<Vec<u8>>` | nullable | raw pubkey bytes, not hex — `entry.rs:27` |
| `object_id` | `Option<String>` | nullable | generic object identifier (event id hex, channel UUID, media sha256) — `entry.rs:28-31` |
| `detail` | `serde_json::Value` | required (type has no `Option`) | included in the hash via canonical JSON — `entry.rs:32-34` |
| `created_at` | `chrono::DateTime<Utc>` | required | `entry.rs:36` |

There is **no** `event_id`, `event_kind`, or `channel_id` field on this struct.
The relay puts those two values inside `detail` instead
(`crates/buzz-relay/src/handlers/event.rs:566-569`).

### 2. `NewAuditEntry` — append input (`crates/buzz-audit/src/entry.rs:52-72`)

Derives `Debug, Clone, PartialEq, Eq` only — deliberately **not**
`Serialize`/`Deserialize`, documented as a provenance fence so no client blob can
become a `NewAuditEntry` (`entry.rs:46-51`).

| Field | Rust type | Notes (file:line) |
|---|---|---|
| `community_id` | `buzz_core::CommunityId` (newtype over `Uuid`) | server-resolved tenant only — `entry.rs:53-57`; newtype defined at `crates/buzz-core/src/tenant.rs:37` |
| `action` | `AuditAction` | `entry.rs:59` |
| `actor_pubkey` | `Option<Vec<u8>>` | `entry.rs:61` |
| `object_id` | `Option<String>` | `entry.rs:63` |
| `detail` | `serde_json::Value` | doc-comment forbids token/secret material; opaque and persisted verbatim — `entry.rs:64-71` |

`seq`, `prev_hash`, `hash`, `created_at` are assigned by
`AuditService::log` (`entry.rs:39-40`; assignment at `service.rs:110-126`).

### 3. `AuditAction` enum (`crates/buzz-audit/src/action.rs:8-31`)

`#[serde(rename_all = "snake_case")]` (`action.rs:7`); `as_str()` supplies the
stable string used in both hashing and DB storage (`action.rs:34-49`).
**11 variants** (not 10):

| # | Variant | `as_str()` string | Declared (file:line) | String mapping (file:line) |
|---|---|---|---|---|
| 1 | `EventCreated` | `event_created` | `action.rs:10` | `action.rs:37` |
| 2 | `EventDeleted` | `event_deleted` | `action.rs:12` | `action.rs:38` |
| 3 | `ChannelCreated` | `channel_created` | `action.rs:14` | `action.rs:39` |
| 4 | `ChannelUpdated` | `channel_updated` | `action.rs:16` | `action.rs:40` |
| 5 | `ChannelDeleted` | `channel_deleted` | `action.rs:18` | `action.rs:41` |
| 6 | `MemberAdded` | `member_added` | `action.rs:20` | `action.rs:42` |
| 7 | `MemberRemoved` | `member_removed` | `action.rs:22` | `action.rs:43` |
| 8 | `AuthSuccess` | `auth_success` | `action.rs:24` | `action.rs:44` |
| 9 | `AuthFailure` | `auth_failure` | `action.rs:26` | `action.rs:45` |
| 10 | `RateLimitExceeded` | `rate_limit_exceeded` | `action.rs:28` | `action.rs:46` |
| 11 | `MediaUploaded` | `media_uploaded` | `action.rs:30` | `action.rs:47` |

Private `const ALL: &'static [Self]` mirrors all 11 variants (`action.rs:51-63`) and
backs both `FromStr` (`action.rs:72-82`) and the round-trip test (`action.rs:89-94`).
`Display` delegates to `as_str()` (`action.rs:66-70`). `FromStr::Err = String`
(`action.rs:73`), message `unknown audit action: {s:?}` (`action.rs:80`).

Note the `serde` representation and `as_str()` happen to agree (snake_case), but
they are two independent mappings — `serde` is derived (`action.rs:7`), storage/hash
uses `as_str()` (`action.rs:35`).

### 4. Database row shape — `audit_log` (`migrations/0001_initial_schema.sql:606-619`)

| Column | SQL type | Constraint | Line |
|---|---|---|---|
| `community_id` | `UUID NOT NULL` | `REFERENCES communities(id)`; part of PK | `0001_initial_schema.sql:607` |
| `seq` | `BIGINT NOT NULL` | part of PK | `:608` |
| `hash` | `BYTEA NOT NULL` | unique per community (index below) | `:609` |
| `prev_hash` | `BYTEA` | nullable | `:610` |
| `action` | `VARCHAR(64) NOT NULL` | free-text; no CHECK/enum | `:611` |
| `actor_pubkey` | `BYTEA` | nullable | `:612` |
| `object_id` | `TEXT` | nullable | `:613` |
| `detail` | `JSONB` | **nullable in SQL** while the Rust field is non-`Option` | `:614` |
| `created_at` | `TIMESTAMPTZ NOT NULL DEFAULT NOW()` | value always supplied by the crate | `:615` |

- `PRIMARY KEY (community_id, seq)` — `0001_initial_schema.sql:616`
- `CREATE UNIQUE INDEX idx_audit_log_hash ON audit_log (community_id, hash)` — `:619`
- No `DELETE`/`UPDATE` statement exists anywhere in the crate; the only write is
  the `INSERT` at `service.rs:130-147`.

Column ↔ field correspondence is 1:1 for both the insert bind order
(`service.rs:137-145`) and the row-decode path (`service.rs:245-255`). `community_id`
is decoded as `Uuid` (`service.rs:246`), i.e. the typed `CommunityId` fence exists
only on the input struct, not on rows read back.

### 5. `GENESIS_HASH` (`crates/buzz-audit/src/hash.rs:9`)

```rust
pub const GENESIS_HASH: [u8; 32] = [0u8; 32];
```

**32 zero bytes** (= 64 zero hex characters when rendered). It is a hashing-time
sentinel only: the first entry of a community stores `prev_hash = NULL`
(`hash.rs:7-8`, `service.rs:108`) and `compute_hash` substitutes `GENESIS_HASH` when
`prev_hash` is `None` (`hash.rs:68-71`). `GENESIS_HASH` is re-exported at the crate
root (`lib.rs:34`).

### 6. Error type shape (`crates/buzz-audit/src/error.rs:12-41`)

`AuditError` is the only error type; see the Conventions aspect for the full variant
table. Data-carrying variants hold only `seq: i64` (`error.rs:22-25`, `29-32`) plus
wrapped `sqlx::Error` / `serde_json::Error` (`error.rs:15`, `:40`). No variant carries
a `community_id` (`error.rs:7-10`).

### 7. Public type exports (`crates/buzz-audit/src/lib.rs:31-35`)

`AuditAction`, `AuditEntry`, `NewAuditEntry`, `AuditError`, `compute_hash`,
`GENESIS_HASH`, `AuditService`. `hash::to_storage_precision` is public but not
re-exported at the root (`hash.rs:22`, absent from `lib.rs:34`).


## Module: buzz-media (`crates/buzz-media`)

### Aspect: Data Model

Scope: all 11 `src/*.rs` files + `tests/static_creds_minio.rs` read in full. The crate has **no database access** — `buzz-db` is not a dependency (`crates/buzz-media/Cargo.toml:11-35`). All persistence is S3 objects. Tenancy types (`CommunityId`, `TenantContext`) come from `buzz-core` (`crates/buzz-media/src/storage.rs:6`).

---

### 1. Public types

#### `BlobDescriptor` — Blossom BUD-02 upload response (`crates/buzz-media/src/types.rs:7-30`)

Derives `Debug, Clone, Serialize, Deserialize` (`crates/buzz-media/src/types.rs:6`).

| Field | Type | Serde | Source |
|---|---|---|---|
| `url` | `String` | — | `crates/buzz-media/src/types.rs:9` |
| `sha256` | `String` (64 hex) | — | `crates/buzz-media/src/types.rs:11` |
| `size` | `u64` | — | `crates/buzz-media/src/types.rs:13` |
| `mime_type` | `String` | `rename = "type"` | `crates/buzz-media/src/types.rs:15-16` |
| `uploaded` | `i64` (unix secs) | — | `crates/buzz-media/src/types.rs:18` |
| `dim` | `Option<String>` (`"WxH"`) | `skip_serializing_if = "Option::is_none"` | `crates/buzz-media/src/types.rs:20-21` |
| `blurhash` | `Option<String>` | skip-if-none | `crates/buzz-media/src/types.rs:23-24` |
| `thumb` | `Option<String>` (URL) | skip-if-none | `crates/buzz-media/src/types.rs:26-27` |
| `duration` | `Option<f64>` (secs, video only) | skip-if-none | `crates/buzz-media/src/types.rs:29-30` |

Empty-string `BlobMeta` fields are converted to `None` when building the descriptor, so they are omitted from JSON rather than serialized as `""` (`crates/buzz-media/src/upload.rs:539-560`).

#### `BlobMeta` — persisted sidecar JSON (`crates/buzz-media/src/storage.rs:385-403`)

Derives `Debug, Clone, Default, Serialize, Deserialize` (`crates/buzz-media/src/storage.rs:384`). Stored at `_meta/{community}/{sha256}.json` (`crates/buzz-media/src/storage.rs:183-185`).

| Field | Type | Serde | Notes |
|---|---|---|---|
| `dim` | `String` | — | `"WxH"`; empty for generic files (`crates/buzz-media/src/upload.rs:265`) |
| `blurhash` | `String` | — | empty for video/files |
| `thumb_url` | `String` | — | empty for video/files |
| `ext` | `String` | — | canonical extension, e.g. `jpg` |
| `mime_type` | `String` | — | sniffed MIME |
| `size` | `u64` | — | byte length |
| `uploaded_at` | `i64` | `#[serde(default)]` | `crates/buzz-media/src/storage.rs:399-400` |
| `duration_secs` | `Option<f64>` | `default`, skip-if-none | `crates/buzz-media/src/storage.rs:401-403` |

#### `BlobHeadMeta` — HEAD result (`crates/buzz-media/src/storage.rs:379-381`)

| Field | Type | Source |
|---|---|---|
| `size` | `u64` (from S3 `content_length`, `unwrap_or(0)`) | `crates/buzz-media/src/storage.rs:167-172` |

Note: no derives on this struct (`crates/buzz-media/src/storage.rs:378-381`).

#### `MediaStorage` — the single storage backend type (`crates/buzz-media/src/storage.rs:19-21`)

| Field | Type | Visibility |
|---|---|---|
| `bucket` | `Box<s3::Bucket>` | private |

There is **no storage trait** in this crate — `MediaStorage` is a concrete struct over `rust-s3`'s `Bucket`; the only pluggable seam is the page-fetching closure in `fold_bucket_listing` (`crates/buzz-media/src/bucket_index.rs:377-383`).

#### `ByteStream` — streaming read alias (`crates/buzz-media/src/storage.rs:16`)

`pub type ByteStream = Pin<Box<dyn futures_core::Stream<Item = Result<Bytes, MediaError>> + Send>>`

#### `MediaConfig` (`crates/buzz-media/src/config.rs:17-62`)

Derives `Debug, Clone, serde::Deserialize` (`crates/buzz-media/src/config.rs:16`). Full field list in the Configuration aspect; fields: `s3_endpoint`, `s3_access_key`, `s3_secret_key`, `s3_bucket`, `s3_region` (`String`); `max_image_bytes`, `max_gif_bytes`, `max_video_bytes`, `max_file_bytes` (`u64`); `public_base_url` (`String`); `upload_records_enabled` (`bool`); `upload_ip_header`, `upload_port_header` (`Option<String>`).

#### `VideoMeta` — MP4 parse result (`crates/buzz-media/src/validation.rs:222-231`)

| Field | Type | Source |
|---|---|---|
| `duration_secs` | `f64` (from `mvhd`/track timescale, not edit lists) | `crates/buzz-media/src/validation.rs:224` |
| `width` | `u32` | `crates/buzz-media/src/validation.rs:226` |
| `height` | `u32` | `crates/buzz-media/src/validation.rs:228` |
| `has_audio` | `bool` | `crates/buzz-media/src/validation.rs:230` |

#### `BlossomVerb` — auth verb enum (`crates/buzz-media/src/auth.rs:6-10`)

`Upload` → `"upload"`, `Get` → `"get"` (`crates/buzz-media/src/auth.rs:13-18`). Derives `Debug, Clone, Copy, PartialEq, Eq`.

#### `MediaError` — 35 variants (`crates/buzz-media/src/error.rs:8-86`)

`thiserror::Error`; full variant table in the Conventions aspect.

#### Upload-record types (`crates/buzz-media/src/upload_record.rs`)

`UploadRecord` (`crates/buzz-media/src/upload_record.rs:56-97`), persisted at `_uploads/{community}/{sha256}/{event_id}.json`:

| Field | Type | Serde |
|---|---|---|
| `version` | `u32` (= `UPLOAD_RECORD_VERSION` = 1, `crates/buzz-media/src/upload_record.rs:52`) | — |
| `event_id` | `String` (ULID) | — |
| `sha256` | `String` | — |
| `ext` | `String` | — |
| `mime_type` | `String` | — |
| `size` | `u64` | — |
| `uploaded_at` | `i64` | — |
| `community_id` | `String` (UUID) | — |
| `community_host` | `String` | — |
| `uploader_id` | `String` (hex pubkey) | — |
| `uploader_npub` | `String` (bech32) | — |
| `uploader_name` | `Option<String>` | skip-if-none (`crates/buzz-media/src/upload_record.rs:85-86`) |
| `ip` | `Option<String>` | skip-if-none (`crates/buzz-media/src/upload_record.rs:89-90`) |
| `port` | `Option<u16>` | skip-if-none (`crates/buzz-media/src/upload_record.rs:94-95`) |

`UploadNetworkInfo` (`crates/buzz-media/src/upload_record.rs:100-105`): `ip: Option<IpAddr>`, `port: Option<u16>`; derives `Debug, Clone, Copy, Default, PartialEq, Eq`.

`UploadAttribution` (`crates/buzz-media/src/upload_record.rs:110-115`): `uploader_name: Option<String>`, `net: UploadNetworkInfo`; derives `Debug, Clone, Default`.

`UploadEventFacts<'a>` (`crates/buzz-media/src/upload_record.rs:119-130`): `sha256: &'a str`, `ext: &'a str`, `mime: &'a str`, `size: u64`, `uploaded_at: i64`; derives `Debug, Clone, Copy`.

#### Bucket-sweep types (`crates/buzz-media/src/bucket_index.rs`)

`KeyClass` enum (`crates/buzz-media/src/bucket_index.rs:34-49`), derives `Debug, Clone, PartialEq, Eq`:

| Variant | Payload | Key shape |
|---|---|---|
| `Thumb` | `sha256: String` | `{sha256}.thumb.jpg` |
| `Blob` | `sha256: String, ext: String` | `{sha256}.{ext}` |
| `Sidecar` | `community: Uuid, sha256: String` | `_meta/{community}/{sha256}.json` |
| `Auxiliary` | `community: Uuid, sha256: String, event_id: String` | `_uploads/{community}/{sha256}/{ulid}.json` |
| `Unknown` | — | everything else |

`CommunityStorage` (`crates/buzz-media/src/bucket_index.rs:197-200`): `bytes: u64`, `objects: u64`.

`BucketSnapshot` (`crates/buzz-media/src/bucket_index.rs:206-229`): `physical_bytes`, `physical_objects`, `logical_bytes`, `logical_objects` (`u64`), `per_community: HashMap<Uuid, CommunityStorage>`, `orphan_blob_bytes`, `orphan_blob_count`, `orphan_sidecar_count`, `multi_variant_shas`, `multi_variant_bytes`, `unknown_key_bytes`, `unknown_key_objects` (all `u64`).

`BucketAggregate` (`crates/buzz-media/src/bucket_index.rs:232-246`) — private fields: `blob_variant_bytes: HashMap<String, Vec<u64>>`, `thumb_bytes: HashMap<String, u64>`, `sidecar_bindings: HashMap<(Uuid, String), u64>`, plus `physical_bytes`, `physical_objects`, `unknown_bytes`, `unknown_objects`.

`Page` (`crates/buzz-media/src/bucket_index.rs:364-368`): `objects: Vec<(String, u64)>`, `next_continuation_token: Option<String>`, `is_truncated: bool`.

`SweepError` (`crates/buzz-media/src/bucket_index.rs:341-362`): `CapExceeded { seen, cap }`, `Storage(#[from] MediaError)`, `Timeout(Duration)`, `MalformedPage`.

#### Private pipeline types (`crates/buzz-media/src/upload.rs`)

`BufferedUploadInput<'a>` (`crates/buzz-media/src/upload.rs:45-52`): `storage`, `config`, `ctx`, `auth_event`, `body: Bytes`, `attribution: Option<UploadAttribution>`.
`MetadataInput` (`crates/buzz-media/src/upload.rs:195-201`): `sha256`, `ext`, `mime` (`String`), `body: Bytes`, `uploaded_at: i64`.

---

### 2. Content-addressing scheme

| Element | Value | file:line |
|---|---|---|
| Hash algorithm | SHA-256, lowercase hex | `crates/buzz-media/src/upload.rs:84` (buffered), `crates/buzz-media/src/upload.rs:397` (streamed) |
| Hash crate | `sha2::Sha256` + `hex::encode` | `crates/buzz-media/src/upload.rs:4`, `crates/buzz-media/Cargo.toml:19-20` |
| Blob key | `{sha256}.{ext}` | `crates/buzz-media/src/upload.rs:94`, `crates/buzz-media/src/upload.rs:421` |
| Thumbnail key | `{sha256}.thumb.jpg` | `crates/buzz-media/src/upload.rs:531` |
| Sidecar key | `_meta/{community_uuid}/{sha256}.json` | `crates/buzz-media/src/storage.rs:183-185` |
| Upload-record key | `_uploads/{community_uuid}/{sha256}/{ulid}.json` | `crates/buzz-media/src/upload_record.rs:181-183` |
| Public blob URL | `{public_base_url}/{sha256}.{ext}` | `crates/buzz-media/src/upload.rs:548` |
| Public thumb URL | `{public_base_url}/{sha256}.thumb.jpg` | `crates/buzz-media/src/thumbnail.rs:44` |
| ULID source | `ulid::Ulid::new().to_string()` (uppercase Crockford base32) | `crates/buzz-media/src/upload_record.rs:150` |

Raw blob bytes are **globally shared CAS** (no community segment); the *sidecar* carries the tenant scoping and is described in code as "the tenant read gate" (`crates/buzz-media/src/storage.rs:177-182`). Two communities uploading identical bytes share one blob object but get distinct sidecars (`crates/buzz-media/src/storage.rs:352-377`).

Extension is derived from the sniffed MIME, never from a client filename: images via `mime_to_ext` (`crates/buzz-media/src/validation.rs:930-939`), generic files via `file_mime_to_ext` with fallback to `infer`'s extension then `"bin"` (`crates/buzz-media/src/validation.rs:99-156`, `crates/buzz-media/src/validation.rs:203-206`), video hardcoded `"mp4"` (`crates/buzz-media/src/upload.rs:420`).

---

### 3. Persisted metadata inventory (S3 objects only)

| Object | Payload type | Content-Type | Writer |
|---|---|---|---|
| `{sha}.{ext}` | raw uploaded bytes | sniffed MIME | `crates/buzz-media/src/upload.rs:129`, `crates/buzz-media/src/upload.rs:434` |
| `{sha}.thumb.jpg` | JPEG thumbnail (≤320px) | `image/jpeg` | `crates/buzz-media/src/upload.rs:531-532` |
| `_meta/{community}/{sha}.json` | `BlobMeta` JSON | `application/json` | `crates/buzz-media/src/storage.rs:210-221` |
| `_uploads/{community}/{sha}/{ulid}.json` | `UploadRecord` JSON | `application/json` | `crates/buzz-media/src/upload_record.rs:174-176` |

No DB rows, no Redis keys, no local filesystem persistence (temp files are deleted after the S3 put — `crates/buzz-media/src/upload.rs:435`).


## Module: buzz-workflow (`crates/buzz-workflow`)

### Aspect: Data Model

Source files read in full: `Cargo.toml`, `src/lib.rs` (1564 LOC), `src/schema.rs` (878), `src/executor.rs` (1834), `src/error.rs` (66), `src/action_sink.rs` (69).

---

### 1. Definition schema root — `WorkflowDef`

`crates/buzz-workflow/src/schema.rs:13-27`. Derives `Debug, Clone, Serialize, Deserialize`. No `deny_unknown_fields` — unknown keys in the stored JSON are silently ignored (this is how the relay's `_webhook_secret` key survives round-trips; see security doc).

| Field | Type | Serde attribute | Line |
|---|---|---|---|
| `name` | `String` | (required) | `schema.rs:16` |
| `description` | `Option<String>` | `#[serde(default)]` | `schema.rs:18-19` |
| `trigger` | `TriggerDef` | (required) | `schema.rs:21` |
| `steps` | `Vec<Step>` | (required) | `schema.rs:23` |
| `enabled` | `bool` | `#[serde(default = "default_true")]` → `true` (`schema.rs:29-31`) | `schema.rs:25-26` |

---

### 2. `TriggerDef` — internally tagged on `on`

`crates/buzz-workflow/src/schema.rs:36-68`: `#[serde(tag = "on", rename_all = "snake_case")]` (`schema.rs:37`). Five variants; fields live inline in the same YAML/JSON mapping as `on:`.

| YAML `on:` value | Rust variant | Fields (type, serde) | Line |
|---|---|---|---|
| `message_posted` | `MessagePosted` | `filter: Option<String>` — `#[serde(default)]` | `schema.rs:40-44` |
| `reaction_added` | `ReactionAdded` | `emoji: Option<String>` — `#[serde(default)]` | `schema.rs:46-50` |
| `diff_posted` | `DiffPosted` | `filter: Option<String>` — `#[serde(default)]` | `schema.rs:52-56` |
| `schedule` | `Schedule` | `cron: Option<String>` and `interval: Option<String>`, both `#[serde(default)]` | `schema.rs:58-65` |
| `webhook` | `Webhook` | unit variant, no fields | `schema.rs:67` |

Round-trip is asserted for `diff_posted` at `schema.rs:855-871`.

---

### 3. `Step`

`crates/buzz-workflow/src/schema.rs:71-87`.

| Field | Type | Serde attribute | Line |
|---|---|---|---|
| `id` | `String` | (required) | `schema.rs:74` |
| `name` | `Option<String>` | `#[serde(default)]` | `schema.rs:76-77` |
| `if_expr` | `Option<String>` | `#[serde(rename = "if", default)]` — YAML key is `if:` | `schema.rs:79-80` |
| `timeout_secs` | `Option<u64>` | `#[serde(default)]` | `schema.rs:82-83` |
| `action` | `ActionDef` | `#[serde(flatten)]` — action tag + fields inline on the step map | `schema.rs:85-86` |

---

### 4. `ActionDef` — internally tagged on `action`

`crates/buzz-workflow/src/schema.rs:90-147`: `#[serde(tag = "action", rename_all = "snake_case")]` (`schema.rs:91`). Seven variants.

| YAML `action:` value | Variant | Fields (type, serde) | Line |
|---|---|---|---|
| `send_message` | `SendMessage` | `text: String`; `channel: Option<String>` `#[serde(default)]` | `schema.rs:94-100` |
| `send_dm` | `SendDm` | `to: String`; `text: String` | `schema.rs:102-107` |
| `set_channel_topic` | `SetChannelTopic` | `topic: String` | `schema.rs:109-112` |
| `add_reaction` | `AddReaction` | `emoji: String` | `schema.rs:114-117` |
| `call_webhook` | `CallWebhook` | `url: String`; `method: Option<String>` `#[serde(default)]`; `headers: Option<HashMap<String,String>>` `#[serde(default)]`; `body: Option<String>` `#[serde(default)]` | `schema.rs:119-131` |
| `request_approval` | `RequestApproval` | `from: String`; `message: String`; `timeout: Option<String>` `#[serde(default)]` | `schema.rs:133-141` |
| `delay` | `Delay` | `duration: String` | `schema.rs:143-146` |

Because both enums are internally tagged and `ActionDef` is additionally `#[serde(flatten)]`ed into `Step`, a step is a flat mapping: `{id, name?, if?, timeout_secs?, action, <action fields>}`.

---

### 5. Runtime / execution types

| Type | Kind | Fields | Line |
|---|---|---|---|
| `WorkflowConfig` | struct, `Clone, Debug` | `max_concurrent: usize`, `default_timeout_secs: u64` | `lib.rs:57-63` |
| `WorkflowEngine` | struct (no derive) | `db: Db` (`lib.rs:76`), `config: WorkflowConfig` (`77`), `run_semaphore: Arc<Semaphore>` (`79`), `last_fired: DashMap<(CommunityId, Uuid), DateTime<Utc>>` (`87`), `action_sink: OnceLock<Arc<dyn ActionSink>>` (`90`), `workflow_cache: moka::sync::Cache<(CommunityId, Uuid), Arc<Vec<buzz_db::workflow::WorkflowRecord>>>` (`104-105`) | `lib.rs:75-106` |
| `TriggerContext` | struct, `Debug, Clone, Default, Serialize, Deserialize` | `text: String`, `author: String`, `channel_id: String`, `timestamp: String`, `emoji: String`, `message_id: String`, `webhook_fields: HashMap<String,String>` | `executor.rs:26-42` |
| `StepResult` | enum, `Debug` | `Completed(JsonValue)`, `Suspended { approval_token: String }`, `Skipped` | `executor.rs:455-465` |
| `ExecutionResult` | struct, `Debug` | `approval_token: Option<String>`, `step_index: usize`, `step_outputs: HashMap<String, JsonValue>`, `trace: Vec<JsonValue>` | `executor.rs:941-952` |
| `PartialProgress` | struct, `Debug, Default` | `step_index: usize`, `trace: Vec<serde_json::Value>` | `error.rs:9-15` |
| `WorkflowError` | enum, `Debug, Error` | 9 variants — see conventions doc | `error.rs:18-60` |
| `ActionSinkError` | enum, `Debug, thiserror::Error` | `InvalidInput(String)`, `ChannelNotFound(String)`, `ChannelArchived(String)`, `EventBuild(String)`, `Database(String)`, `EmptyContent` | `action_sink.rs:12-32` |
| `ActionSink` | trait, `Send + Sync` | one method `send_message` returning `Pin<Box<dyn Future<Output = Result<String, ActionSinkError>> + Send + '_>>` | `action_sink.rs:48-70` |

Trace entry shapes written into `execution_trace` (a JSON array):

| Shape | Written when | Line |
|---|---|---|
| `{"step_id": …, "status": "skipped"}` | `if:` evaluated false | `executor.rs:1108-1111` |
| `{"step_id": …, "status": "completed", "output": <JsonValue>}` | step dispatched successfully | `executor.rs:1179-1183` |
| `{"step_id": …, "status": "skipped"}` | `StepResult::Skipped` returned | `executor.rs:1202-1205` |

Per-action `output` payloads (become `steps.ID.output.*`):

| Action | Output JSON | Line |
|---|---|---|
| `send_message` | `{"sent": true, "event_id": <hex>}` | `executor.rs:574-577` |
| `add_reaction` (with `reqwest`) | `{"added": true, "status": <u16>, "response": <json>}` | `executor.rs:928-932` |
| `add_reaction` (no `reqwest`) | `{"added": false, "skipped": true}` | `executor.rs:613-615` |
| `call_webhook` (with `reqwest`) | `{"status": <u16>, "body": <string>}` | `executor.rs:865-868` |
| `call_webhook` (no `reqwest`) | `{"status": 0, "body": null, "skipped": true}` | `executor.rs:642-646` |
| `delay` | `{"slept_secs": <u64>}` | `executor.rs:685-687` |
| `send_dm`, `set_channel_topic` | none — return `WorkflowError::NotImplemented` | `executor.rs:583`, `executor.rs:589` |
| `request_approval` | none — returns `StepResult::Suspended { approval_token }` | `executor.rs:665-667` |

---

### 6. DB row shapes consumed via `buzz-db`

The crate does not define SQL; it uses `buzz_db::Db` methods and the record structs in `crates/buzz-db/src/workflow.rs`.

**Read: `WorkflowRecord`** (`crates/buzz-db/src/workflow.rs:165-190`) — `id: Uuid`, `community_id: CommunityId`, `name: String`, `owner_pubkey: Vec<u8>`, `channel_id: Option<Uuid>`, `definition: serde_json::Value`, `definition_hash: Vec<u8>`, `status: WorkflowStatus`, `enabled: bool`, `created_at`, `updated_at`.
Fields actually touched by this crate: `definition` (deserialized into `WorkflowDef` at `lib.rs:331`, `lib.rs:459`), `enabled` (`lib.rs:335`, `lib.rs:474`), `id` (`lib.rs:339`, `lib.rs:346`), `community_id` (`lib.rs:455`), `channel_id` (`lib.rs:478`, `executor.rs:553`), `owner_pubkey` (hex-encoded at `executor.rs:558`).

**Read: `WorkflowRunRecord`** (`crates/buzz-db/src/workflow.rs:192-229`) — `id`, `community_id`, `workflow_id`, `status: RunStatus`, `trigger_event_id: Option<Vec<u8>>`, `current_step: i32`, `execution_trace: serde_json::Value`, `trigger_context: Option<serde_json::Value>`, `started_at`, `completed_at`, `error_message`, `created_at`. This crate reads `workflow_id` (`executor.rs:546`) and `execution_trace` (`executor.rs:1038`).

**Write path:**

| Db method | Args written by this crate | Call site |
|---|---|---|
| `create_workflow_run(community_id, workflow_id, trigger_event_id: Option<&[u8]>, trigger_context: Option<&Value>)` → `Uuid` | event path passes the trigger event id bytes + serialized `TriggerContext`; cron path passes `None` event id | `lib.rs:344-355`, `lib.rs:590-600` |
| `update_workflow_run(community_id, id, status: RunStatus, current_step: i32, trace: &Value, error: Option<&str>)` | `Running` at start (`executor.rs:985-994`, `executor.rs:1047-1056`); `Completed`/`Failed` at finalize (`lib.rs:199-215`, `lib.rs:218-238`, `lib.rs:242-261`) | as listed |
| `claim_scheduled_workflow_fire(community_id, workflow_id, scheduled_for)` → `Option<ScheduledWorkflowFireClaim>` | durable at-most-once cron claim | `lib.rs:547-568` |
| `attach_scheduled_workflow_run(community_id, workflow_id, scheduled_for, run_id)` → `bool` | best-effort link claim→run | `lib.rs:617-628` |
| `latest_scheduled_workflow_fire(community_id, workflow_id)` → `Option<DateTime<Utc>>` | restart anchor for interval triggers | `lib.rs:500-517` |
| `list_enabled_channel_workflows(community_id, channel_id)` → `Vec<WorkflowRecord>` | per-event lookup (cached) | `lib.rs:301-306` |
| `list_all_enabled_workflows()` → `Vec<WorkflowRecord>` | cron scan (not community-scoped; carries `community_id` per row) | `lib.rs:436` |
| `get_workflow_run(community_id, run_id)` / `get_workflow(community_id, workflow_id)` | destination validation + attribution for `send_message`; trace restore for resume | `executor.rs:535-556`, `executor.rs:1037` |

**Status enums referenced** (defined in `crates/buzz-db/src/workflow.rs`): `RunStatus { Pending, Running, WaitingApproval, Completed, Failed, Cancelled }` (`workflow.rs:78-91`); `WorkflowStatus { Active, Disabled, Archived }` (`workflow.rs:42-50`); `ApprovalStatus { Pending, Granted, Denied, Expired }` (`workflow.rs:124-133`). This crate writes only `Running`, `Completed`, `Failed` (`executor.rs:989`, `executor.rs:1051`, `lib.rs:204`, `lib.rs:223`, `lib.rs:247`) — it never writes `WaitingApproval`, `Pending`, or `Cancelled`.

**Approval row shape it does *not* write:** `ApprovalRecord` (`crates/buzz-db/src/workflow.rs:244-268`) — `token: Vec<u8>` (SHA-256 hash of the raw token, `workflow.rs:33-35`), `workflow_id`, `run_id`, `step_id: String`, `step_index: i32`, `approver_spec: String`, `status: ApprovalStatus`, `approver_pubkey: Option<Vec<u8>>`, `note: Option<String>`, `expires_at`, `created_at`. No code path in the repository calls `Db::create_approval` (`crates/buzz-db/src/lib.rs:2729`) outside buzz-db itself — verified by repo-wide grep.


## Module: buzz-relay — core & bootstrap (`crates/buzz-relay/src`)
### Aspect: Data Model

Scope: `main.rs` (1940), `state.rs` (1932), `config.rs` (1346), `router.rs` (530), `nip11.rs` (523), `protocol.rs` (458), `tenant.rs` (333), `telemetry.rs` (214), `metrics.rs` (207), `admission.rs` (158), `lib.rs` (56), `error.rs` (50), `Cargo.toml`. Total 7,747 LOC. No `unsafe` (`lib.rs:1` `#![deny(unsafe_code)]`); `#![warn(missing_docs)]` at `lib.rs:2`.

---

#### 1. `AppState` — every field

`AppState` is declared `state.rs:488-628` (141 lines, 39 fields), `#[derive(Clone)]` at `state.rs:487` (cheap: every non-`Copy` field is `Arc`/`Db`/`Pool`). Constructed once in `AppState::new` (`state.rs:636-798`) and wrapped `Arc<AppState>` at `main.rs:435`.

| # | Field | Line | Type | Populated by | Read by (production) |
|---|-------|------|------|--------------|----------------------|
| 1 | `config` | `state.rs:490` | `Arc<Config>` | `state.rs:717` from `main.rs:425` | pervasive (101 `state.config.*` sites) |
| 2 | `db` | `state.rs:492` | `Db` | `state.rs:719` ← `main.rs:151` | pervasive; `main.rs:954`, `router.rs:352`, `nip11.rs:246/278` |
| 3 | `redis_pool` | `state.rs:494` | `deadpool_redis::Pool` | `state.rs:720` ← `main.rs:333` (`redis_health_pool` clone) | `router.rs:353` (readiness), `main.rs:979`, `mesh_boot` via `main.rs:445` |
| 4 | `audit` | `state.rs:496` | `Option<Arc<AuditService>>` | `state.rs:718` | **ZERO readers anywhere.** See §5 |
| 5 | `pubsub` | `state.rs:498` | `Arc<PubSubManager>` | `state.rs:721` ← `main.rs:343` | `main.rs:784/822/852/903`, `state.rs:966/1039`, `handlers/*`, `connection.rs:267` |
| 6 | `auth` | `state.rs:500` | `Arc<AuthService>` | `state.rs:722` ← `main.rs:368` | `api/bridge.rs:29`, `connection.rs:614/634/636`, `handlers/auth.rs` |
| 7 | `search` | `state.rs:502` | `Arc<SearchService>` | `state.rs:723` (`search_arc`, `state.rs:652`) | `handlers/req.rs`, `api/bridge.rs` |
| 8 | `sub_registry` | `state.rs:504` | `Arc<SubscriptionRegistry>` | `state.rs:724` | `main.rs:1327`, `handlers/req.rs`, `handlers/close.rs` |
| 9 | `conn_manager` | `state.rs:506` | `Arc<ConnectionManager>` | `state.rs:725` | 17 sites incl. `main.rs:918/1130/1145`, `handlers/event.rs:146/184/460` |
| 10 | `community_connections` | `state.rs:508` | `Arc<CommunityConnectionRegistry>` | `state.rs:726` | `connection.rs:127`, `audio/handler.rs:153`, `main.rs:910`, `state.rs:1077` |
| 11 | `community_revalidator_cancel` | `state.rs:510` | `CancellationToken` | `state.rs:727` | `main.rs:886` (task), `main.rs:1045` (cancel post-serve) |
| 12 | `community_disconnect_publish_attempts` | `state.rs:512` | `Arc<AtomicU64>` | `state.rs:728` | written `state.rs:1063`; read **only** at `api/operator.rs:1010,1020`, both inside `#[cfg(test)]` (`api/operator.rs:500`) → **test-only telemetry** |
| 13 | `conn_semaphore` | `state.rs:514` | `Arc<Semaphore>` | `state.rs:729` (`max_connections`, `state.rs:649`) | `connection.rs:149`, `audio/handler.rs:90/113` |
| 14 | `handler_semaphore` | `state.rs:516` | `Arc<Semaphore>` | `state.rs:730` (`max_concurrent_handlers`, `state.rs:650`) | `connection.rs:513/541/563` |
| 15 | `git_semaphore` | `state.rs:521` | `Arc<Semaphore>` | `state.rs:731` (`git_max_concurrent_ops`, `state.rs:692`) | `api/git/transport.rs:322` |
| 16 | `media_upload_semaphore` | `state.rs:523` | `Arc<Semaphore>` | `state.rs:732` (`media_max_concurrent_uploads`, `state.rs:693`) | `api/media.rs:119` |
| 17 | `workflow_engine` | `state.rs:526` | `Arc<WorkflowEngine>` | `state.rs:733` ← `main.rs:390` | `workflow_sink.rs`, `api/bridge.rs` |
| 18 | `relay_keypair` | `state.rs:528` | `nostr::Keys` | `state.rs:734` ← `main.rs:392-414` | 47 sites; `nip11.rs:242/303`, `mesh_boot` via `main.rs:444` |
| 19 | `local_event_ids` | `state.rs:541` | `Arc<moka::sync::Cache<(CommunityId,[u8;32]),()>>` | `state.rs:736-741` | `handlers/event.rs:301/279/401/832/860/1053`, `side_effects.rs:798/877`, `audio/handler.rs:1328` |
| 20 | `membership_cache` | `state.rs:545` | `Arc<Cache<(CommunityId,Uuid,Vec<u8>),bool>>` | `state.rs:742-748` | via `is_member_cached` (`state.rs:827`); direct at `handlers/event.rs:2171/2151/2267/2270` |
| 21 | `accessible_channels_cache` | `state.rs:549` | `Arc<Cache<(CommunityId,Vec<u8>),Vec<Uuid>>>` | `state.rs:749-755` | **only via `AppState` methods** (`state.rs:1089`, `868`, `882`, `937`); no external direct reader |
| 22 | `channel_visibility_cache` | `state.rs:552` | `Arc<Cache<(CommunityId,Uuid),String>>` | `state.rs:756-762` | `state.rs:1124`, `handlers/event.rs:2142/2142/2228` |
| 23 | `audit_tx` | `state.rs:555` | `Option<mpsc::Sender<NewAuditEntry>>` | `state.rs:763` — `audit_enabled.then_some(audit_tx)` (`state.rs:716`) | `handlers/event.rs:581+`, `api/media.rs` — **this, not `audit`, is the real audit path** |
| 24 | `media_storage` | `state.rs:557` | `Arc<MediaStorage>` | `state.rs:764` ← `main.rs:419` | `main.rs:1455`, `api/media.rs` (11 sites) |
| 25 | `storage_sweep` | `state.rs:561` | `Arc<tokio::sync::Mutex<StorageSweepState>>` | `state.rs:765-767` | `main.rs:1461`, `main.rs:1474` |
| 26 | `git_store` | `state.rs:565` | `GitStore` (not `Arc`) | `state.rs:768` ← built in-constructor `state.rs:694-701` | `main.rs:491`, `api/git/*` |
| 27 | `git_pack_cache` | `state.rs:569` | `Arc<GitPackCache>` | `state.rs:769` ← `state.rs:702-709` | `api/git/transport.rs`, `api/git/hydrate.rs` |
| 28 | `audio_rooms` | `state.rs:571` | `Arc<AudioRoomManager>` | `state.rs:770` | `main.rs:455`, `audio/handler.rs` (8 sites) |
| 29 | `shutting_down` | `state.rs:573` | `Arc<AtomicBool>` | `state.rs:771` | `router.rs:281` (WS refuse), `router.rs:337` (readiness 503), `main.rs:449/457` (mesh), written `main.rs:1133` |
| 30 | `started_at` | `state.rs:575` | `Instant` | `state.rs:772` | `router.rs:388` (`/_status` uptime) — single reader |
| 31 | `nip98_replay` | `state.rs:582` | `Arc<dyn Nip98ReplayGuard>` | `state.rs:710-711` (`RedisNip98ReplayGuard`) | `api/invites.rs`, `api/operator.rs`, `api/bridge.rs` |
| 32 | `admission_rate_limiter` | `state.rs:584` | `Arc<RedisRateLimiter>` | `state.rs:712` | `api/bridge.rs:31`, `connection.rs:616`, `connection.rs:639` |
| 33 | `observer_rate_limiter` | `state.rs:589` | `Arc<ScopedRateLimiter>` (`DashMap`) | `state.rs:773` | `handlers/event.rs:924` — single reader |
| 34 | `media_upload_rate_limiter` | `state.rs:592` | `Arc<ScopedRateLimiter>` (`DashMap`) | `state.rs:774` | `api/media.rs:97` — single reader |
| 35 | `invite_claim_rate_limiter` | `state.rs:597` | `Arc<Cache<ScopedPubkeyKey,Arc<AtomicU32>>>` | `state.rs:775-780` | `api/invites.rs:380` |
| 36 | `media_uploads_in_flight` | `state.rs:600` | `Arc<DashMap<ScopedPubkeyKey,u32>>` | `state.rs:781` | `api/media.rs:70-71` |
| 37 | `observer_owner_cache` | `state.rs:607` | `Arc<Cache<(CommunityId,Vec<u8>,Vec<u8>),bool>>` | `state.rs:782-787` | `handlers/event.rs` (6 sites) |
| 38 | `author_type_cache` | `state.rs:613` | `Arc<Cache<(CommunityId,Vec<u8>),bool>>` | `state.rs:788-793` | `handlers/ingest.rs:1360/1342`, `api/mod.rs:220` |
| 39 | `tracer` | `state.rs:620` | `Arc<dyn buzz_conformance::Tracer>` | `state.rs:797` — always `NoopTracer` in production | `handlers/*` emit sites |
| 40 | `mesh` | `state.rs:627` | `Arc<OnceLock<MeshHandle>>` | `state.rs:798` empty; set at `main.rs:459` | via `AppState::mesh()` (`state.rs:812`) → `router.rs:381` |

Count: 40 fields (the table numbering matches; the struct body spans `state.rs:488-628`).

##### Verified deltas
- The `Clone` on `AppState` (`state.rs:487`) is real but *unused as a `Clone`*: every consumer takes `&AppState` or `Arc<AppState>`. `git_store` (`state.rs:565`) is the only non-`Arc`, non-pool field and is `Clone`-by-value on each `AppState` clone.
- `AppState`'s `Debug` impl (`state.rs:1209-1215`) prints only `relay_url` and `max_connections` — no secret leakage, but also no diagnostic value beyond config echo.

#### 2. In-memory caches and their eviction

| Cache | Line | Key | Value | Max capacity | TTL | Invalidation closures |
|-------|------|-----|-------|--------------|-----|-----------------------|
| `local_event_ids` | `state.rs:736-741` | `(CommunityId,[u8;32])` | `()` | 10,000 | 60 s | no |
| `membership_cache` | `state.rs:742-748` | `(CommunityId,Uuid,Vec<u8>)` | `bool` | 10,000 | 10 s | **yes** (`state.rs:746`) |
| `accessible_channels_cache` | `state.rs:749-755` | `(CommunityId,Vec<u8>)` | `Vec<Uuid>` | 10,000 | 10 s | **yes** (`state.rs:753`) |
| `channel_visibility_cache` | `state.rs:756-762` | `(CommunityId,Uuid)` | `String` | 10,000 | 10 s | **yes** (`state.rs:760`) |
| `invite_claim_rate_limiter` | `state.rs:775-780` | `ScopedPubkeyKey` | `Arc<AtomicU32>` | `api::invites::CLAIM_RATE_CACHE_CAPACITY` | `api::invites::CLAIM_RATE_WINDOW` | no |
| `observer_owner_cache` | `state.rs:782-787` | `(CommunityId,Vec<u8>,Vec<u8>)` | `bool` | 1,000 | 300 s | no |
| `author_type_cache` | `state.rs:788-793` | `(CommunityId,Vec<u8>)` | `bool` | 10,000 | 300 s | no |

Unbounded structures (no capacity cap, no TTL, no reaper):

| Structure | Line | Key | Growth bound |
|-----------|------|-----|--------------|
| `observer_rate_limiter` | `state.rs:589`, alias `state.rs:39` | `(CommunityId,[u8;32])` → `(u32, Instant)` | **none** — `DashMap` entries are never removed in this file |
| `media_upload_rate_limiter` | `state.rs:592` | same | **none** |
| `media_uploads_in_flight` | `state.rs:600` | `ScopedPubkeyKey` → `u32` | **none** in this file |
| `ConnectionManager::connections` | `state.rs:183` | `Uuid` → `ConnEntry` | bounded by `conn_semaphore` (`state.rs:514`) |
| `CommunityConnectionRegistry::connections` | `state.rs:67` | `Uuid` → `(CommunityId, CancellationToken)` | bounded by the RAII guard `state.rs:120-130` |

The doc comment on `invite_claim_rate_limiter` (`state.rs:593-596`) explicitly justifies a hard capacity because "pre-membership callers can cheaply generate fresh Nostr keys". The same argument applies to `observer_rate_limiter` and `media_upload_rate_limiter` (both keyed on caller-chosen pubkey bytes) and is *not* applied there — see debt.

#### 3. Connection-tracking types

| Type | Line | Shape | Notes |
|------|------|-------|-------|
| `ScopedPubkeyKey` | `state.rs:37` | `(CommunityId, [u8;32])` | `pub(crate)`; re-used by `api/invites.rs:386`, `api/media.rs:70` |
| `SlidingWindowCounter` | `state.rs:38` | `(u32, Instant)` | private |
| `ScopedRateLimiter` | `state.rs:39` | `DashMap<ScopedPubkeyKey, SlidingWindowCounter>` | private alias, two `AppState` fields use it |
| `ConnEntry` | `state.rs:42-63` | `tx`, `ctrl_tx`, `cancel`, `community_id`, `backpressure_count: Arc<AtomicU8>`, `subscriptions`, `authenticated_pubkey: Arc<RwLock<Option<Vec<u8>>>>`, `grace_limit: u8` | private struct, 8 fields |
| `ConnectionManager` | `state.rs:182-188` | `connections: DashMap<Uuid,ConnEntry>` + `draining: AtomicBool` | sticky one-way drain flag (`state.rs:184-187`) |
| `CommunityConnectionRegistry` | `state.rs:65-68` | `Arc<DashMap<Uuid,(CommunityId,CancellationToken)>>` | shared by ordinary WS *and* huddle audio |
| `CommunityConnectionGuard` | `state.rs:120-123` | RAII deregister (`Drop` at `state.rs:125-130`) | |
| `ThreadedChannelVisibility` | `state.rs:1162-1170` | `community_id`, `channel_id`, `visibility: String` | consumed at `handlers/event.rs:120/332/380/2232/2273/2303`, `handlers/ingest.rs:1781` |
| `AuditShutdownHandle` | `state.rs:1177-1180` | `cancel: CancellationToken`, `handle: JoinHandle<()>` | returned from `AppState::new`, drained `main.rs:1049` |

Note the *authenticated pubkey* is `Vec<u8>` inside `Arc<RwLock<Option<..>>>` (`state.rs:60`) rather than a typed `PublicKey` — every comparison is a byte-slice compare (`state.rs:271-275`).

#### 4. `Config` struct shape

`Config` at `config.rs:51-263`: **51 public fields**, no nesting except three embedded sub-configs.

| Group | Fields (line) |
|-------|---------------|
| Listeners | `bind_addr:53`, `uds_path:95`, `health_port:98`, `metrics_port:100`, `relay_url:68`, `pairing_relay_url:70` |
| Datastores | `database_url:55`, `read_database_url:58`, `redis_url:60`, `redis_pool_size:66` |
| Connection limits | `max_connections:72`, `max_concurrent_handlers:74`, `send_buffer_size:76`, `max_frame_bytes:78`, `slow_client_grace_limit:80` |
| Auth / membership | `auth:82` (`buzz_auth::AuthConfig`), `require_auth_token:85`, `pubkey_allowlist_enabled:106`, `require_relay_membership:112`, `relay_private_key:92`, `relay_owner_pubkey:149`, `relay_operator_api_origin:157`, `relay_operator_pubkeys:170`, `allow_nip_oa_auth:184` |
| Web / CORS | `cors_origins:89`, `web_dir:259`, `serve_git_web_gui:262`, `admin:254` (`AdminConfig`), `join_policy:251` (`JoinPolicyConfig`) |
| Media | `media:187` (`buzz_media::MediaConfig`), `media_max_concurrent_uploads:189`, `media_max_concurrent_uploads_per_pubkey:191`, `media_uploads_per_minute:193`, `require_media_get_auth:197` |
| Git | `git_repo_path:218`, `git_pack_cache_path:220`, `git_max_pack_bytes:222`, `git_max_repo_bytes:227`, `git_pack_cache_max_bytes:230`, `git_pack_cache_max_concurrent_populations:232`, `git_max_repos_per_pubkey:234`, `git_max_concurrent_ops:236`, `git_hook_hmac_secret:239` |
| Push (NIP-PL) | `push_executor_key_id:242`, `push_gateway_delivery_url:245`, `push_gateway_timeout:247` |
| Mesh | `mesh:136` (`buzz_relay_mesh::MeshConfig`), `mesh_demo_echo:144` |
| Misc | `huddle_audio_available:129`, `audit_enabled:202`, `ephemeral_ttl_override:209` |

Sub-structs owned by this module:
- `AdminConfig` (`config.rs:29-34`): `host: String`, `web_dir: Option<PathBuf>`.
- `JoinPolicyConfig` (`config.rs:37-47`): `terms_markdown`, `privacy_markdown`, `age_attestation_required`, `version: String` (SHA-256 over the three, `config.rs:797-801`).

`Config` derives `Debug, Clone` (`config.rs:50`) — so `{:?}` on `Config` **prints `database_url`, `redis_url`, `relay_private_key`, `git_hook_hmac_secret`, `media.s3_secret_key` in cleartext**. No `Debug` redaction anywhere in `config.rs`. See security.

`ConfigError` (`config.rs:19-27`), 2 variants: `InvalidBindAddr(String)`, `InvalidValue(String)`. Both constructed; `InvalidBindAddr` only from `parse_bind_addr` (`config.rs:265-268`).

`DEFAULT_MAX_FRAME_BYTES = 512 * 1024` (`config.rs:15`), the only public const; used at `config.rs:468` and 8 test sites in `nip11.rs`.

#### 5. Dead / write-only data

| Item | Line | Evidence |
|------|------|----------|
| `AppState::audit` | `state.rs:496` | Written `state.rs:718`. Grep for `.audit` across `crates/**` + `desktop/src-tauri/**` returns **no read**. The `AuditService` stays alive only because the worker closure captured `audit_for_worker` (`state.rs:656`), and the field's `is_some()` is consumed via the separate local `audit_enabled` (`state.rs:716`). The field is pure retained state. |
| `AppState::community_disconnect_publish_attempts` | `state.rs:512` | Incremented `state.rs:1063` on every archive publish; read only at `api/operator.rs:1010,1020` — both after `#[cfg(test)]` (`api/operator.rs:500`). Test-only → counts as zero production readers. Its own doc says "Test/telemetry counter". |
| `RelayError` — 9 of 10 variants | `error.rs:8-48` | Only `InvalidMessage` (`error.rs:44`) is ever constructed, all in `protocol.rs:43-171`. `WebSocket:11`, `Json:15` (`#[from]`), `Database:19` (`#[from]`), `Auth:23` (`#[from]`), `PubSub:27` (`#[from]`), `ConnectionLimitReached:31`, `RateLimitExceeded:35`, `NotAuthenticated:39`, `Internal:47` have zero constructors in `crates/buzz-relay/**`. (Matches in `buzz-acp` are a *different* `RelayError` type.) The four `#[from]` impls are therefore unreachable. |
| `lib.rs` re-exports | `lib.rs:53-55` | `pub use config::Config; pub use error::{RelayError, Result}; pub use state::AppState;`. No crate depends on `buzz-relay` as a library (only `buzz-relay` itself; `buzz-admin`/`buzz-conformance`/`git-sign-nostr` only mention it in comments). `main.rs:17-24` imports via full module paths (`buzz_relay::config::Config`, `buzz_relay::state::AppState`), never the re-exports. All three re-exports are unused. |
| `CommunityConnectionRegistry::bound_communities` | `state.rs:111` | `pub`, but the only non-test caller is `revalidate_registered_communities` in the same file (`state.rs:174`). Should be `pub(crate)` or private. |
| `ConnectionManager::pubkey_for` vs `pubkey_for_conn` | `state.rs:425-430` vs `state.rs:286-291` | **Byte-identical bodies and signatures.** Both live: `pubkey_for` ← `handlers/event.rs:483`; `pubkey_for_conn` ← `handlers/event.rs:146/184`, `handlers/side_effects.rs:108`. Two names, one behaviour. |
| `ConnectionState::remote_addr` | `connection.rs:61` | Populated from the router's `ConnectInfo` (`router.rs:236-240` → `connection.rs:170`). No production read; the only reads are test fixtures (`state.rs:1358`, `handlers/event.rs:1388`). The client IP is captured and discarded. |

#### 6. Protocol message types (`protocol.rs`)

`ClientMessage` (`protocol.rs:23-42`), 5 variants, all parsed and all matched in production:

| Variant | Line | Payload | Parse guard |
|---------|------|---------|-------------|
| `Event(Event)` | `protocol.rs:25` | `nostr::Event` | `arr.len() >= 2` (`protocol.rs:59`) |
| `Req { sub_id, filters }` | `protocol.rs:27-32` | `String`, `Vec<Filter>` | non-empty sub_id (`:80`), `len <= 256` (`:86`), `filters <= 10` (`:93`) |
| `Close(String)` | `protocol.rs:34` | `String` | `arr.len() >= 2` (`:147`) — **no length or emptiness check** (unlike REQ/COUNT) |
| `Count { sub_id, filters }` | `protocol.rs:36-41` | `String`, `Vec<Filter>` | same guards as REQ (`:120`, `:125`, `:131`) |
| `Auth(Event)` | `protocol.rs:42` | `nostr::Event` | `arr.len() >= 2` (`:159`) |

Constants: `MAX_SUB_ID_LENGTH = 256` (`protocol.rs:11`), `MAX_FILTERS_PER_REQ = 10` (`protocol.rs:14`). Both are **hard-coded duplicates** of the NIP-11 advertised values at `nip11.rs:107` and `nip11.rs:105` — no shared const.

`RelayMessage` (`protocol.rs:176`) is a unit struct namespace with 7 associated fns, all returning `String`. All 7 have production callers: `auth_challenge:179`, `event:184`, `notice:191`, `eose:196`, `ok:201`, `closed:206`, `count:211` (→ `handlers/count.rs:285`, single caller).

#### 7. Tenancy types (`tenant.rs`)

| Item | Line | Shape |
|------|------|-------|
| `HostResolver` trait | `tenant.rs:31-49` | assoc `type Error`; native `async fn resolve_host(&self, normalized_host:&str) -> Result<Option<CommunityId>, Self::Error>` — no `async-trait`, no `dyn` |
| `BindError<E>` | `tenant.rs:52-62` | `UnmappedHost` \| `Lookup(E)` — 2 variants, both constructed (`tenant.rs:89`, `:93`, `:94`) |
| `impl HostResolver for buzz_db::Db` | `tenant.rs:141-152` | `Error = buzz_db::DbError`; adapts `lookup_community_by_host` → `CommunityId` |
| `relay_url_authority` | `tenant.rs:139` | `pub use buzz_core::tenant::relay_url_authority` — a re-export, not a local fn |

The canonical carrier is `buzz_core::tenant::TenantContext` (imported `tenant.rs:17`), constructed only via `TenantContext::resolved(community, host)` at `tenant.rs:91` (request path), `main.rs:641` (reaper, per-row), `main.rs:743` (reminder scheduler, per-row). No `TenantContext::default()` path exists in this module.

`BindError` intentionally has **no** distinct `EmptyHost` variant — the empty/whitespace host short-circuit at `tenant.rs:85-87` reuses `UnmappedHost` so the rejection is byte-identical (documented `tenant.rs:78-84`).

#### 8. Metrics-poller data types (`main.rs`)

| Type | Line | Shape | Note |
|------|------|-------|------|
| `EmissionScope` | `main.rs:50-53` | `All` \| `Off` | `allows(&self, _community_id: &Uuid)` (`main.rs:76-78`) **ignores its argument** — the documented `top:<k>` mode (`main.rs:44-47`) is unimplemented |
| `InMemoryMetricKey` | `main.rs:1279-1284` | `WsConnections(String)` \| `UsersOnline(String)` \| `Subscriptions(String)` — carries the *host label*, not the UUID | enables final-zero emission for removed communities (`main.rs:1371-1373`) |
| `USAGE_METRICS_LOCK_KEY` | `main.rs:80` | `i64 = 0x4255_5A5A_4D45_5452` (ASCII `BUZZMETR`) | pg advisory-lock key, `main.rs:1428` |
| `SWEEP_CONFIG` | `main.rs:1444-1445` | function-local `OnceLock<StorageSweepConfig>` | deliberately *not* on `Config`/`AppState` (`main.rs:1446-1449`) |

#### 9. NIP-11 document types (`nip11.rs`)

`RelayInfo` (`nip11.rs:25-59`), 13 fields; `RelayLimitation` (`nip11.rs:62-92`), 13 fields. See api-surface for the field-by-field values. `SUPPORTED_NIPS: &[u32] = &[1,2,10,11,16,17,23,25,29,33,38,42,50,56]` (`nip11.rs:15`, 14 entries) plus conditional `NIP_RELAY_MEMBERSHIP = 43` (`nip11.rs:21`).


## Module: buzz-relay — WebSocket protocol & subscriptions (`crates/buzz-relay/src`)
### Aspect: Data Model

Scope: `connection.rs` (893), `subscription.rs` (1562), `handlers/{mod,event,req,count,close,auth}.rs` (62 / 2461 / 1946 / 281 / 35 / 350) = 7,590 LOC.

---

#### 1. `AuthState` — per-connection NIP-42 state machine

`connection.rs:37-48`. Three states, no others; `#[derive(Debug, Clone)]` (`:36`).

| State | Payload | Set at | Exit transitions |
|---|---|---|---|
| `Pending { challenge: String }` | the random challenge string sent to the client | `connection.rs:171-173` (at construction, before any frame is read) | → `Authenticated` (`auth.rs:278`), → `Failed` (`auth.rs:172`, `:206`, `:230`, `:287`) |
| `Authenticated(AuthContext)` | `buzz_auth::AuthContext` | `auth.rs:278` | **terminal** — a second AUTH is refused (`auth.rs:49-57`) |
| `Failed` | none | `auth.rs:172` (ban / ban-check error), `:206` (allowlist), `:230` (relay membership), `:287` (NIP-42 verify failure) | **terminal** — further AUTH refused (`auth.rs:58-66`) |

`Failed` is *sticky for the life of the socket*: `handle_auth` reads the state first (`auth.rs:45-74`) and only proceeds from `Pending`. There is no retry path. The ban-seam code comments call this out explicitly at `auth.rs:98-112` ("without … pinning `Failed` for the connection's life on a false premise") — hence the deliberate `Banned` vs `DbError` split at `auth.rs:113-117`.

`AuthContext` fields the WS path consumes (`crates/buzz-auth/src/lib.rs:64-79`):

| Field | Type | Read by |
|---|---|---|
| `pubkey` | `nostr::PublicKey` | `event.rs:640`, `req.rs:63`, `count.rs:41`, `connection.rs:607`, `connection.rs:277` |
| `scopes` | `Vec<Scope>` | `event.rs:641` (→ `:658`, `:676`), `req.rs:54` |
| `channel_ids` | `Option<Vec<Uuid>>` | `event.rs:642`, `req.rs:74`, `count.rs:41` |
| `auth_method` | `AuthMethod` | `auth.rs:191` (allowlist gate applies only to `Nip42`) |
| `agent_owner_pubkey` | `Option<PublicKey>` | `event.rs:989` (observer fast path), `connection.rs:607` (agent-vs-human rate tier) |

**Delta vs doc.** `channel_ids` is documented as "reserved for future per-channel access control" (`buzz-auth/src/lib.rs:69`). It is **not** reserved — it is load-bearing today: it narrows `accessible_channels` at `req.rs:87-88`, `count.rs:95-97`, and bounds the request-local repair at `req.rs:137-139` / `count.rs:142-149`.

---

#### 2. `ConnectionState` — every field

`connection.rs:53-80`. Deliberately split by access pattern (doc `:50-52`).

| Field | Type | Lock | Purpose / notes |
|---|---|---|---|
| `conn_id` | `Uuid` | — | `Uuid::new_v4()` at `connection.rs:127`; primary key into `ConnectionManager` and `SubscriptionRegistry` |
| `tenant` | `TenantContext` | — | resolved from the HTTP `Host` **before** the upgrade (`router.rs:286-300`); doc `connection.rs:57-60` says "never overridable by client-supplied input" — verified: no write path exists after construction |
| `remote_addr` | `SocketAddr` | — | stored but **only** used for the two `info!` log lines (`connection.rs:183`, `:289`). Never used for authorization or rate limiting |
| `auth_state` | `RwLock<AuthState>` | tokio `RwLock` | read on every EVENT/REQ/COUNT/AUTH; written only by `handle_auth` |
| `subscriptions` | `Arc<Mutex<HashMap<String, Vec<Filter>>>>` | tokio `Mutex` | alias `ConnectionSubscriptions` (`connection.rs:30`). **Second copy** of the filters already in `sub_registry` |
| `send_tx` | `mpsc::Sender<WsMessage>` | — | capacity = `config.send_buffer_size` (`connection.rs:159`), default **1000** (`config.rs:459-462`) |
| `ctrl_tx` | `mpsc::Sender<WsMessage>` | — | capacity hard-coded **8** (`connection.rs:162`). Pong / Close / ban-reason / restart-close |
| `cancel` | `CancellationToken` | — | created in `handle_connection` (`connection.rs:128`), shared with `ConnEntry` |
| `backpressure_count` | `Arc<AtomicU8>` | atomic | shared with `ConnEntry.backpressure_count` (`state.rs:54`, `:212`) so direct sends and fan-out sends decrement one counter |
| `grace_limit` | `u8` | — | copied from `config.slow_client_grace_limit` (`connection.rs:179`), default **15** (`config.rs:470-473`) |

Not stored on `ConnectionState`: the authenticated pubkey (it lives separately in `ConnEntry.authenticated_pubkey`, `state.rs:56`, set at `auth.rs:279-281`). Fan-out reads the `ConnEntry` copy (`event.rs:146`, `:184`, `:460`), never `ConnectionState`.

Task-local state not on the struct:
- `missed_pongs: Arc<AtomicU8>` — `connection.rs:218`, shared between `heartbeat_loop` (`:389`) and `recv_loop` (`:462`).
- The owned semaphore permit — `connection.rs:149`, dropped at `:287`.

---

#### 3. `SubscriptionRegistry` — six concurrent maps

`subscription.rs:44-59`. `#[derive(Debug, Default)]`; `new()` (`:66`) is `Self::default()`.

##### 3.1 Authoritative store

```
subs: DashMap<ConnId, HashMap<SubId, SubEntry>>          // subscription.rs:49
SubEntry = (Vec<Filter>, CommunityId, Option<Uuid>)      // subscription.rs:16
ConnId   = Uuid                                          // subscription.rs:12
SubId    = String                                        // subscription.rs:14
```

`SubEntry` carries the **server-resolved** community and the derived channel scope alongside the filters. Doc `:47-48` claims this "enables O(1) targeted index removal and gives lifecycle code the exact Redis topic to release" — verified at `remove_from_index` (`:391-479`) and at the three `release_topic` sites (`req.rs:251`, `close.rs:21`, `connection.rs:268`).

##### 3.2 Five fan-out indexes

| Index | Key | Value | Populated when | Sites |
|---|---|---|---|---|
| `channel_kind_index` | `(CommunityId, IndexKey{channel_id, kind})` | `Vec<(ConnId, SubId)>` | `channel_id.is_some()` and every filter has a non-empty `kinds` | insert `:107-111`, read `:278`, remove `:426-433` |
| `channel_wildcard_index` | `(CommunityId, Uuid)` | `Vec<(ConnId, SubId)>` | `channel_id.is_some()` and ≥1 filter has **no** `kinds` | insert `:90-93`, read `:284`, remove `:405-411` |
| `global_kind_index` | `(CommunityId, Kind)` | `Vec<(ConnId, SubId)>` | `channel_id.is_none()`, kinds present, **and** the p-kind key extraction returned `None` | insert `:136-140`, read `:305-308`, remove `:459-467` |
| `global_p_kind_index` | `GlobalPKindIndexKey{community_id, kind, p}` (`:27-32`, private) | `Vec<(ConnId, SubId)>` | `channel_id.is_none()` and **every** filter has non-empty `kinds` **and** a non-empty `#p` (`:486-520`) | insert `:120-123`, read `:294-302`, remove `:442-450` |
| `global_wildcard_index` | `CommunityId` | `Vec<(ConnId, SubId)>` | `channel_id.is_none()`, ≥1 filter kindless, no p-kind key | insert `:128-131`, read `:314`, remove `:452-458` |

`IndexKey` is `pub` (`:19-25`) — `{channel_id: Uuid, kind: Kind}`, `Hash + Eq`. `GlobalPKindIndexKey` is private (`:28`). `RemovedSubscription` is `pub` (`:35-41`) — `{community_id, channel_id: Option<Uuid>}`, `Copy`.

##### 3.3 Index-selection rule (`extract_kinds_from_filters`, `:546-567`)

Tri-state over the whole filter *set* (NIP-01 OR semantics, doc `:539-544`):

| Return | Meaning | Index used |
|---|---|---|
| `None` | at least one filter omits `kinds` → subscription is a wildcard | `*_wildcard_index` |
| `Some(vec![])` | every filter had `kinds: []` | **none** — deliberately unindexed (`:95-100`, `:415-418`); matches nothing |
| `Some(kinds)` | union of all kinds across filters | one entry per kind in `*_kind_index` |

`extract_global_p_kind_index_keys` (`:486-520`) returns `None` (falling back to the generic global indexes) if **any** filter lacks `kinds`, or lacks a `#p` tag; it `continue`s past a filter with empty `kinds` (`:494-496`) — so a `[{kinds:[]}, {kinds:[9], #p:[x]}]` pair still lands only on the p-kind index. `p` values are stored as raw `String`, not normalized (no lowercasing), while `event_p_tag_values` (`:522-537`) also returns raw strings — so p-index matching is **case-sensitive** on hex.

##### 3.4 No refcounting in the registry

There is **no** per-topic refcount inside `SubscriptionRegistry`. Refcounting lives one layer out, in `buzz_pubsub`: `desired_topics: HashMap<EventTopicKey, usize>` with `retain_topic` (`buzz-pubsub/src/lib.rs:192-208`) and `release_topic` (`:215-232`). The registry's contribution is returning `Option<RemovedSubscription>` / `Vec<RemovedSubscription>` so callers can issue exactly one matching `release_topic`.

`Vec<(ConnId, SubId)>` index buckets are linear-scanned on removal (`retain`, `:407`, `:429`, `:446`, `:455`, `:464`) and emptied buckets are removed from the DashMap (`:409-410`, `:431-432`, `:448-449`, `:456-457`, `:465-466`) — so no empty-bucket leak, but a channel+kind with N subscribers costs O(N) per removal.

---

#### 4. Per-connection limits and counters

| Limit | Value | Definition | Enforced at | Configurable |
|---|---|---|---|---|
| Subscriptions per connection | **1024** | `req.rs:26` `MAX_SUBSCRIPTIONS` | `req.rs:66` (against `conn.subscriptions.len()`) | no |
| Filters per REQ / COUNT | **10** | `protocol.rs:14` `MAX_FILTERS_PER_REQ` | `protocol.rs:93-98`, `:135-140` | no |
| Sub-id length | **256 B** | `protocol.rs:11` `MAX_SUB_ID_LENGTH` | `protocol.rs:87-91`, `:128-133` | no |
| Inbound frame bytes | **524288** (512 KiB) | `config.rs:14` `DEFAULT_MAX_FRAME_BYTES` | parser `router.rs:340-342`; app-level re-check `connection.rs:421`, `:440` | `BUZZ_MAX_FRAME_BYTES` |
| Historical rows per filter | **2000** | `req.rs:25` `MAX_HISTORICAL_LIMIT` | `req.rs:881-882`, search `:538-539` | no |
| Concurrent per-REQ DB queries | **4** | `req.rs:35` `FILTER_QUERY_CONCURRENCY` (compile-time bounded 2..=8 at `:41`) | `req.rs:318` | no |
| Search pages per filter | **10** × 100 rows | `req.rs:421` `MAX_SEARCH_PAGES`, per-page `:589` | `req.rs:594` | no |
| COUNT fallback candidate rows | **5000** (+1 probe) | `req.rs:753` `COUNT_FALLBACK_CANDIDATE_LIMIT` | `req.rs:752-761`, `count.rs:186`/`:255` | no |
| Outbound data-frame batch | **64** | `connection.rs:33` `MAX_WS_SEND_BATCH` | `connection.rs:351` | no |
| Send-buffer depth | **1000** msgs | `config.rs:459-462` | `connection.rs:159` | `BUZZ_SEND_BUFFER` |
| Control-buffer depth | **8** msgs | hard-coded `connection.rs:162` | — | no |
| Backpressure strikes | **15** | `config.rs:470-473` | `connection.rs:100` / `state.rs:464` | `BUZZ_SLOW_CLIENT_GRACE_LIMIT` |
| Missed pongs | **3** | `connection.rs:389-393` | same | no |
| Auth grace | **5 s** | `connection.rs:27` `AUTH_TIMEOUT` | `connection.rs:232` | no |
| Observer telemetry | **100/s per (community, agent)** | `event.rs:917-939` | `event.rs:1036` | no |
| Observer freshness | **±300 s** | `event.rs:952` | same | no |

Counters: `backpressure_count` (`AtomicU8`, reset on any successful send — `connection.rs:92`, `state.rs:456`), `missed_pongs` (`AtomicU8`, reset on Pong — `connection.rs:462`), `observer_rate_limiter` (`DashMap<(CommunityId,[u8;32]), (u32, Instant)>`, `state.rs:589`, ctor `:773`).

**`AtomicU8` overflow risk:** `grace_limit` is a `u8` from an unvalidated `parse()` (`config.rs:470-473`). Setting `BUZZ_SLOW_CLIENT_GRACE_LIMIT` > 255 makes the parse fail and silently fall back to 15; setting it to `0` makes `count >= grace_limit` true on the *first* full buffer (`connection.rs:100`) — instant disconnect. There is no `>0` filter, unlike `max_frame_bytes` (`config.rs:467`).

---

#### 5. How filters are stored for matching

Filters are `nostr::Filter` values, stored **twice**:

1. `ConnectionState.subscriptions: HashMap<SubId, Vec<Filter>>` — written at `req.rs:236-239`, removed at `close.rs:16`. Its **only** read is the `len()`/`contains_key` cap check at `req.rs:66`. `ConnectionManager::subscriptions_for` (`state.rs:383`) hands the same `Arc` to `side_effects.rs:71`.
2. `SubscriptionRegistry.subs[conn_id][sub_id].0` — written at `subscription.rs:79-82`, and this is the copy actually used for matching (`push_match` → `filters_match`, `:377`).

Matching is a **full linear predicate evaluation** per candidate: the index narrows candidates by `(community, channel, kind)` or `(community, kind, #p)`, then `buzz_core::filter::filters_match` (`buzz-core/src/filter.rs:11-13` → `filter_match_one` `:35-103`) re-checks kinds, authors, since, until, `ids` (**prefix** match, `:64-68`), and every generic tag. There is no precompiled/normalized filter representation.

`filter_match_one`'s `#h` handling (`buzz-core/src/filter.rs:69-102`) has an `h`-specific fallback: if the event carries no `h` tag at all, `StoredEvent.channel_id` is used instead — so reactions/deletions that derive their channel match `#h` filters. If the event *does* carry `h` tags and none match, it is strictly rejected (`:98-100`).

Per-filter clones: `register_scoped` clones the whole `Vec<Filter>` (`subscription.rs:81`) and each `sub_id` once per index key (`:92`, `:110`, `:122`, `:130`, `:138`). `req.rs` clones the filter vec twice more (`:238`, `:245`). Memory per subscription therefore scales with `filters × (1 + copies) + kinds × sub_id_len`, with `sub_id_len` up to 256 B and `kinds` unbounded within one filter.

---

#### 6. Shared registries this group writes into (`state.rs`)

| Structure | Type | Written by this group |
|---|---|---|
| `ConnectionManager.connections` | `DashMap<Uuid, ConnEntry>` (`state.rs:183`) | `register` `connection.rs:199-212`; `deregister` `:271`; `set_authenticated_pubkey` `auth.rs:279-281` |
| `ConnEntry` | `state.rs:41-58` — `{tx, ctrl_tx, cancel, community_id, backpressure_count, subscriptions, authenticated_pubkey: Arc<std::sync::RwLock<Option<Vec<u8>>>>, grace_limit}` | as above |
| `CommunityConnectionRegistry.connections` | `DashMap<Uuid, (CommunityId, CancellationToken)>` (`state.rs:66`) | via `run_registered_community_connection` `connection.rs:132-140` |
| `local_event_ids` | `moka::sync::Cache<(CommunityId,[u8;32]), ()>`, cap 10 000, TTL 60 s (`state.rs:734-739`) | `mark_local_event` `event.rs:417`, `:824`, `:852`, `:1046`; read+invalidate `event.rs:301-303` |
| `observer_rate_limiter` | `DashMap<(CommunityId,[u8;32]), (u32, Instant)>` (`state.rs:589`, ctor `:773`) | `event.rs:917-939` |
| `observer_owner_cache` | `moka::sync::Cache<(CommunityId, Vec<u8>, Vec<u8>), bool>`, cap 1000, TTL 300 s (`state.rs:782-787`) | `event.rs:1018-1036` |

**Unbounded structure:** `observer_rate_limiter` is a plain `DashMap` with **no capacity bound and no eviction** (`state.rs:589`, ctor `:773`, `event.rs:920-924` only ever calls `entry().or_insert()`). One entry per distinct `(community, agent pubkey)` accumulates for process lifetime — 40 B key + 24 B value per observed agent key, never reclaimed. Compare `local_event_ids` / `observer_owner_cache`, which are moka caches with explicit caps.


## Module: buzz-relay — event ingest & side effects (`crates/buzz-relay/src/handlers`)
### Aspect: Data Model

This group owns no tables. It is the **write orchestrator**: it decides which rows are
written, in what order, and inside which transaction boundary. Below: what gets written,
the in-memory types that carry it, and the materialized-counter contract.

---

### 1. Tables written, by write path

| Table | Written by | Transaction boundary |
|---|---|---|
| `events` | `insert_event_with_thread_metadata` (`ingest.rs:2420`), `replace_addressable_event` (`ingest.rs:2398`), `replace_parameterized_event` (`ingest.rs:2413`), `insert_reaction_event_with_thread_metadata` (`ingest.rs:2324`), `insert_event` (`side_effects.rs:692`, `:868`, `:2752`, `:2911`, `:3186`), raw `INSERT` (`command_executor.rs:196-232`) | one tx per call, except the command path (see §6) |
| `thread_metadata` | same tx as the `events` insert, via `insert_event_with_thread_metadata_tx` (`buzz-db/src/event.rs:1045`) | atomic with the event |
| `reactions` | `insert_reaction_event_with_thread_metadata` (`ingest.rs:2324`); removal in `side_effects.rs:2175`, `:2216` | atomic with the kind:7 event |
| `channels` | `create_channel_with_id` (`ingest.rs:2129`), `create_channel` (`side_effects.rs:1692`, `:1719`), `update_channel` (`side_effects.rs:1345`, `:1360`, `:1424`, `:1466`), `archive_channel` / `unarchive_channel` (`:1485`, `:1499`), `soft_delete_channel` (`ingest.rs:2434`, `side_effects.rs:1789`), `open_dm` (`command_executor.rs:398`, `:534`) | separate tx per call |
| `channel_members` | `add_member` (`side_effects.rs:1216`, `:1868`), `remove_member` (`:1293`, `:1932`) | separate tx |
| `users` | `ensure_user` (`side_effects.rs:1096`, `:1140`, `command_executor.rs:49`), `update_user_profile` (`side_effects.rs:1163`, `:1182`), `set_channel_add_policy` (`side_effects.rs:1105`) | separate tx |
| `relay_members` | `remove_relay_member` (`ingest.rs:1883`); 9030–9033 delegate to `relay_admin` | separate tx |
| `moderation_reports` | `report::handle_report_event` (`ingest.rs:1588`) | out of module |
| product-feedback table | `product_feedback::handle` (`ingest.rs:1567`) | out of module |
| `push_leases` | `push_lease::accept` (`ingest.rs:2183`) | out of module |
| `archived_identities` | `identity_archive::handle_identity_archive_event` (`ingest.rs:1942`) | out of module |
| `workflows` | `upsert_workflow` (`command_executor.rs:775`), `delete_workflow_for_owner` (`side_effects.rs:2000`, `:2026`) | pool, **outside** the command tx |
| `workflow_runs` | `create_workflow_run` (`command_executor.rs:918`), `update_workflow_run` (`:1177`, `:1276`, `:1305`) | pool |
| `workflow_approvals` | `update_approval_by_stored_hash` (`command_executor.rs:1041`, `:1153`) | pool |
| `git_repo_names` | `reserve_repo_name` (`side_effects.rs:2500`), `release_repo_name` (`side_effects.rs:2543`) | separate tx; `ON CONFLICT DO NOTHING` is the TOCTOU guard |
| object store (git) | `put_manifest` / `put_pointer` / `get_pointer` (`side_effects.rs:2642-2726`) | CAS, not SQL |
| `audit_log` | indirectly, via `dispatch_persistent_event` → `enqueue_event_created_audit` (`handlers/event.rs:359`, `:534-577`) | bounded mpsc, capacity 1000 |
| `mentions` | indirectly, inside `Db::insert_event_with_thread_metadata` (`buzz-db/src/lib.rs:1394-1399`) — best-effort, `warn!` on failure | same call, non-transactional |

---

### 2. Ingest write order (the happy path, kind 9 with a reply)

1. `get_channel` prefetch — one SELECT reused by three later gates (`ingest.rs:1764-1767`).
2. `channel_visibility_cached` — resolves fan-out visibility once, bundled with
   `(community, channel)` so it cannot be applied to the wrong pair
   (`ingest.rs:1776-1786`, type `crate::state::ThreadedChannelVisibility`).
3. `is_member_cached` (+ open-visibility fallback) — authorization read (`ingest.rs:1811`).
4. Blob existence reads for every `imeta` tag (`ingest.rs:2241`).
5. `get_event_by_id` + `get_thread_metadata_by_event` for the reply parent, issued
   **concurrently** via `tokio::join!` (`ingest.rs:606-611`); possibly a third
   `get_event_by_id` for the root (`:634`, `:665`).
6. **Write**: `insert_event_with_thread_metadata` — one transaction containing the
   `events` row, the `thread_metadata` row, parent/root stub rows, `reply_count + 1` on
   the parent and `descendant_count + 1` on the root
   (`buzz-db/src/event.rs:1090-1191`).
7. `insert_mentions` — same call, **not** in the transaction, `warn!` on failure
   (`buzz-db/src/lib.rs:1394-1399`).
8. `handle_side_effects` — separate, non-transactional, failure only `warn!`-logged
   (`ingest.rs:2460-2467`).
9. `emit_live_thread_summary` — spawned task; re-reads counters and publishes a
   relay-signed kind:39005 (`ingest.rs:2474-2481`).
10. `enqueue_event_created_audit` — awaited (backpressure by design)
    (`handlers/event.rs:336-343`).
11. `dispatch_persistent_event` — spawns Redis publish + local fan-out + workflow trigger
    (`ingest.rs:2513-2521`).

Steps 6 and 7 are the only durable writes to `events`; steps 8–11 are all
post-commit and best-effort.

---

### 3. In-memory types

#### `IngestAuth` (`ingest.rs:63-86`)

| Variant | Fields | Populated by |
|---|---|---|
| `Nip42` | `pubkey: nostr::PublicKey`, `scopes: Vec<Scope>`, `channel_ids: Option<Vec<Uuid>>`, `conn_id: Uuid` | `handlers/event.rs:720-725` |
| `Http` | `pubkey`, `scopes: Vec<Scope>`, `auth_method: HttpAuthMethod` | `api/bridge.rs:826-829` |

Verified population reality: `scopes` is always `Scope::all_known()` (16 scopes) on both
paths (`buzz-auth/src/lib.rs:137`, `api/bridge.rs:827`); `channel_ids` is always `None`
(`buzz-auth/src/lib.rs:138`); `auth_method` is always `Nip98` (`api/bridge.rs:828`) —
`HttpAuthMethod::DevPubkey` (`ingest.rs:57`) is never constructed in production.

#### `IngestResult` / `IngestError` (`ingest.rs:166-184`)

`IngestResult { event_id: String (hex), accepted: bool, message: String }`. `message` is a
tagged union encoded as a string prefix: `""` (plain success), `"duplicate:"` /
`"duplicate: …"` (conflict or replay), `"info: …"` (28936), `"response:{json}"` (command
kinds), `"{}"` (41012). ⚠ There is no type for this — callers string-match
(`buzz-cli/src/commands/mem.rs:105`).

`IngestError` has 3 variants mapped per transport at `api/bridge.rs:842-871`:
`Rejected` → 400, `AuthFailed` → 403, `Internal` → 500.

#### `ThreadMetadataOwned` (`ingest.rs:535-561`)

Owned mirror of `buzz_db::event::ThreadMetadataParams<'_>`, needed because the borrowed
form cannot outlive the resolution step. Fields: `event_id`, `event_created_at`,
`channel_id`, `parent_event_id`, `parent_event_created_at`, `root_event_id`,
`root_event_created_at`, `depth: i32`, `broadcast: bool`. `as_params()` (`:548`) always
sets the parent/root `Option`s to `Some` — this type only exists for actual replies.

`broadcast` is derived from a `["broadcast","1"]` tag (`ingest.rs:695-698`); `depth` is
`parent_meta.depth + 1`, or the heuristic 1/2 when the parent has no `thread_metadata`
row yet (`ingest.rs:660`).

#### `ReactionChannelResult` (`ingest.rs:322-328`)

Five outcomes distinguishing "target is global" (`NoChannel` → store globally) from
"target missing" (`NotFound` → reject) — the distinction that makes a reaction to a
global kind:1 legal while a reaction to a nonexistent id is not.

#### `PersistResult` (`command_executor.rs:80-85`)

`Duplicate` | `Inserted(sqlx::Transaction<'static, Postgres>)`. The **open transaction is
the return value** — the caller must commit it after running domain mutations. Dropping it
rolls back.

---

### 4. The imeta metadata model (`imeta.rs`)

An `imeta` tag is a Nostr tag whose elements after `"imeta"` are `"key value"` strings
(single-space split, `splitn(2, ' ')` — `imeta.rs:46-49`), i.e. a flattened key/value map
inside a tag array.

| Key | Type / grammar | Mandatory | Singleton | Validated at |
|---|---|---|---|---|
| `url` | local `/media/{64-hex}.{ext}` or `{media_base_url}/{64-hex}.{ext}`; never a `.thumb.` path | ✅ | ✅ | `imeta.rs:59-68` |
| `m` | well-formed `type/subtype`, ≤255 chars, no whitespace/control | ✅ | ✅ | `imeta.rs:69-81`, `is_well_formed_mime` `:340-349` |
| `x` | 64-char lowercase hex SHA-256 | ✅ | ✅ | `imeta.rs:82-89` |
| `size` | positive `u64` (0 rejected) | ✅ | ✅ | `imeta.rs:90-96` |
| `dim` | opaque (allowed, unvalidated) | ❌ | ✅ | allowlist only |
| `blurhash` | opaque | ❌ | ✅ | allowlist only |
| `alt` | opaque | ❌ | ✅ | allowlist only |
| `thumb` | local `.thumb.jpg` path | ❌ | ✅ | `imeta.rs:97-105` |
| `fallback` | opaque — **the only allowed non-singleton key** | ❌ | ❌ | `imeta.rs:11-18` |
| `duration` | positive finite `f64`; video/mp4 only | ❌ | ✅ | `imeta.rs:106-115`, `:167-176` |
| `bitrate` | positive `u64`; video/mp4 only | ❌ | ✅ | `imeta.rs:116-118` |
| `image` | local `/media/` path with ext ∈ {jpg,png,gif,webp}, not a thumbnail; video/mp4 only | ❌ | ✅ | `imeta.rs:119-137` |
| `filename` | 1–255 chars, no `/`, `\`, or control chars — display-only, never influences the content-addressed storage key | ❌ | ✅ | `imeta.rs:138-155` |

13 allowed keys, 12 singletons (`imeta.rs:11-18`).

**Internal consistency rules** (`imeta.rs:178-200`):
- hash embedded in `url` must equal `x`;
- for the 5 "previewable" MIMEs (`image/jpeg`, `image/png`, `image/gif`, `image/webp`,
  `video/mp4` — `imeta.rs:24-30`) the url extension must equal `mime_to_ext(m)` exactly;
- for any other MIME the ext is not derivable, so equality is deferred to the sidecar
  cross-check;
- `thumb`'s embedded hash must equal `x`.

**Storage cross-check** (`verify_imeta_blobs` `imeta.rs:209-335`) — five reads per tag:
1. sidecar must exist for `x`;
2. `HEAD {x}.{sidecar.ext}` must exist;
3. claimed `m` == `sidecar.mime_type`, claimed `size` == `sidecar.size`, claimed
   `duration` within 0.1 s of `sidecar.duration_secs`;
4. if `thumb` claimed, `HEAD {x}.thumb.jpg`;
5. if `image` claimed, resolve its sidecar (MIME must be an image type, ext must match)
   and `HEAD` the poster blob.

The sidecar is the authority for `ext`; `is_local_media_url` (`imeta.rs:373-418`) only
performs a structural gate, rejecting `?`, `#`, and `%` and accepting either
`{64hex}.{safe_ext}` or `{64hex}.thumb.jpg`. `is_safe_ext` is shared with the serve path
so the predicate cannot drift (`imeta.rs:377-379`).

---

### 5. Command-executor input/output types

| Kind | Inputs read from the event | Output `message` |
|---|---|---|
| 41010 | 1–8 `p` tags → `Vec<Vec<u8>>` participants, deduped with self prepended (`command_executor.rs:325-345`) | `response:{"channel_id":"<uuid>","created":<bool>}` (`:432-440`) |
| 41011 | `h` → `Uuid`; `p` tags merged into existing member set; ≤9 total | `response:{"channel_id":"<uuid>"}` (`:571-577`) |
| 41012 | `h` → `Uuid` | `{}` (`:648`) |
| 30620 | `h` → `Uuid`; `d` → workflow `Uuid`; optional `name` tag; `content` = workflow YAML → `(WorkflowDef, definition_json)`; SHA-256 of the **post-secret-injection** JSON as `definition_hash` (`:306-308`, `:747-751`) | `response:{"workflow_id":"<uuid>"[,"webhook_secret":"…"]}` (`:797-805`) |
| 46020 | `d` or `e` → workflow `Uuid`; `content` (if a JSON object) → `TriggerContext.webhook_fields: HashMap<String,String>`, non-string values stringified (`:889-902`) | `response:{"run_id":"<uuid>"}` (`:951-957`) |
| 46030 / 46031 | `d` or `e` → hex → `token_hash: Vec<u8>`; `content` → optional approver note | `response:{"status":"granted"\|"denied","run_id":"<uuid>"}` (`:1088-1095`, `:1315-1322`) |

Tag extractors (all "first match wins", all comparing `tag.kind().to_string()`):
`extract_p_tags` `:235`, `extract_h_tag` `:250`, `extract_d_tag` `:261`,
`extract_e_tag` `:272`, `extract_tag` `:283`. `decode_pubkey` `:294` enforces exactly
32 bytes. `compute_definition_hash` `:306` is raw SHA-256 bytes.

`persist_command_event` writes 11 columns of `events` by hand
(`command_executor.rs:196-217`): `community_id, id, pubkey, created_at, kind, tags,
content, sig, received_at, channel_id, d_tag`. ⚠ It bypasses
`Db::insert_event_with_thread_metadata`, so command events get **no** `thread_metadata`
row and **no** `mentions` row. Harmless today (no command kind is threaded or
mention-bearing), but it is a schema-drift hazard: a new `events` column with a NOT NULL
default handled inside `buzz-db` would not be applied here.

---

### 6. Transaction boundaries — precise

| Path | Atomic set | Non-atomic tail |
|---|---|---|
| Generic insert | `events` + `thread_metadata` + counter updates (`buzz-db/src/event.rs:1004-1191`) | `mentions`, side effects, audit, fan-out |
| Reaction | `reactions` upsert + `events` insert + `thread_metadata` (`buzz-db/src/event.rs:1201-1275`) | `mentions`, fan-out |
| Replaceable / NIP-33 | old-row soft-delete + new insert inside `replace_*_event` | everything after |
| 9007 create-group | ⚠ **two** transactions: `create_channel_with_id` (`ingest.rs:2129`) then the event insert (`:2394`). Compensated manually — an insert failure triggers `soft_delete_channel` + `invalidate_channel_deleted` (`ingest.rs:2430-2440`) | side effects |
| Command kinds | `events` row only (`command_executor.rs:196`) | ⚠ **all domain mutations** — `open_dm`, `hide_dm`, `upsert_workflow`, `create_workflow_run`, `update_approval_by_stored_hash` all run on the pool before `tx.commit()`; the module documents this at `:92-98` |
| Deletion | soft-delete + counter decrement (`soft_delete_event_and_update_thread`) | reaction-row removal, system message, 39005 |
| 30617 announce | ⚠ **three** stores: Postgres name reservation, object-store manifest, object-store pointer. Manual compensation: only a fresh `Reserved` claim releases the name on pointer failure (`side_effects.rs:2528-2555`) | kind:30618 emission |

---

### 7. Materialized thread counters — full audit

Columns: `thread_metadata.reply_count`, `thread_metadata.descendant_count`,
`thread_metadata.last_reply_at` (`buzz-db/src/thread.rs:49-51`).

**Semantics** (`buzz-db/src/thread.rs:241-249`): `reply_count` counts *direct children*;
`descendant_count` counts *all* descendants at every depth. The root's
`descendant_count` is bumped even when `root == parent`.

#### Every insert path, and whether it touches the counters

| Insert path | Passes thread metadata? | Counters updated? | Correct? |
|---|---|---|---|
| `ingest.rs:2420` `insert_event_with_thread_metadata` (generic) | yes, when `thread_meta.is_some()` | ✅ same tx (`buzz-db/src/event.rs:1045-1229`, stub-insert `:1131`) | ✅ the only reply-bearing path in ingest |
| `ingest.rs:2324` `insert_reaction_event_with_thread_metadata` | `thread_params` passed but always `None` in practice — kind 7 is not in `requires_h_channel_scope` (`ingest.rs:455-491`), so `thread_meta` is `None` at `:2220-2231` | n/a | ✅ reactions are not replies |
| `ingest.rs:2397` `replace_addressable_event` | **no such parameter** | never | ✅ today — no `is_replaceable` kind is in `requires_h_channel_scope` |
| `ingest.rs:2411` `replace_parameterized_event` | **no such parameter** | never | ✅ today — no 30000–39999 kind is in `requires_h_channel_scope` |
| `command_executor.rs:196` raw `INSERT` | no | never | ✅ no command kind is threaded |
| `side_effects.rs:692` `emit_system_message` → `insert_event` | no | never | ✅ kind 40099 is a channel-level notice, never a reply |
| `side_effects.rs:868`, `:2752`, `:2911`, `:3186` `insert_event` (44100/44101, 30618, 8000/8001, 8002/8003) | no | never | ✅ all global (`channel_id = None`) |
| `workflow_sink.rs:331` `insert_event_with_thread_metadata` | yes, with `depth: 0`, `parent: None`, `root: None` and an explicit comment "Workflow messages are always top-level" (`workflow_sink.rs:318-329`) | no increment (guarded on `parent_event_id.is_some()`) | ✅ |

#### Decrement paths

| Path | Function | Counters |
|---|---|---|
| kind 9005 | `side_effects.rs:1624` `soft_delete_event_and_update_thread` | ✅ same tx |
| kind 5 (per target) | `side_effects.rs:2147` same | ✅ same tx |

Both re-emit a kind:39005 afterwards so live badge counts move **down**
(`side_effects.rs:1638-1645`, `:2158-2168`).

#### Verdict

**Every reply-insert path reachable in production updates the counters, and every
delete path decrements them.** The AGENTS.md invariant holds today.

Two latent traps:
1. `replace_addressable_event` and `replace_parameterized_event` have **no thread-metadata
   parameter at all**. `thread_meta` is computed at `ingest.rs:2246-2257` and then
   *silently dropped* on those two branches (`:2367-2390`). Adding any replaceable kind to
   `requires_h_channel_scope` would lose thread ancestry with no compile error and no
   warning. There is no test asserting the two predicates are disjoint from the
   replaceable ranges — only that `is_global_only_kind` and `requires_h_channel_scope` are
   disjoint from each other (`ingest.rs:2753-2762`).
2. `buzz_db::thread::increment_reply_count` (`buzz-db/src/thread.rs:251-287`) has **zero
   callers anywhere in the workspace**, and `Db::decrement_reply_count`
   (`buzz-db/src/lib.rs:2088-2095`) likewise. `Db::insert_thread_metadata`
   (`buzz-db/src/lib.rs:1973`) is only reached from `#[cfg(test)]` code
   (`buzz-db/src/thread.rs:1315`, inside the test module that starts at `:810`). Three
   dead counter-mutation entry points remain available for a future caller to pick up and
   use *outside* a transaction, which is exactly the inconsistency the transactional
   variants exist to prevent (the rationale is spelled out at
   `buzz-db/src/thread.rs:111-114`).

#### Thread summary read shape

`get_thread_summary` (`buzz-db/src/lib.rs:2049`) yields `reply_count`,
`descendant_count`, `last_reply_at`, `participants: Vec<Vec<u8>>`, rendered as the
kind:39005 content JSON at `side_effects.rs:751-756`:

```json
{"reply_count":n,"descendant_count":n,"last_reply_at":<unix|null>,"participants":["<hex>",...]}
```

with tags `["e",root]`, `["d",root]`, `["h",channel]` (`side_effects.rs:757-761`). Same
contract as the channel-window page overlay in `api/bridge.rs` — documented as "one
contract, two delivery doors" (`side_effects.rs:748-750`), matching
`docs/nips/NIP-CW.md:129-133` and `docs/bridge-channel-window.md:99`.

---

### 8. Relay-signed events this module mints

| Kind | Emitter | Storage | Replacement key |
|---|---|---|---|
| 39000 group metadata | `emit_group_discovery_events` `side_effects.rs:962` | channel-scoped | `(kind, relay_pubkey, channel_id)` addressable |
| 39001 group admins | same | channel-scoped | same |
| 39002 group members | same | channel-scoped | same |
| 39005 thread summary | `emit_live_thread_summary` `side_effects.rs:724` | **never stored** — fan-out only | n/a |
| 40099 system message | `emit_system_message` `side_effects.rs:677` | channel-scoped, plain insert | n/a |
| 44100 / 44101 membership notification | `emit_membership_notification` `side_effects.rs:817` | **global** (`channel_id = None`) so global subscribers see it; `h` tag carries the channel as metadata | n/a |
| 8000 / 8001 NIP-43 delta | `publish_nip43_delta` `side_effects.rs:2881` | global, NIP-70 `["-"]` | n/a |
| 13534 NIP-43 membership list | `publish_nip43_membership_locked` (DB-side) `side_effects.rs:2866` | global addressable | per-community advisory lock covers read+build+write |
| 8002 / 8003 NIP-IA delta | `publish_nipia_delta` `side_effects.rs:3142` | global; tags `["-"]`, `["p",target]`, `["consent",path,actor]`, `["e",request_id]`, optional `reason` / `replaced-by` | n/a |
| 13535 NIP-IA archived list | `publish_nipia_archival_list` `side_effects.rs:3008` | global addressable | `(kind, relay_pubkey)` |
| 30622 NIP-DV DM visibility | `publish_dm_visibility_snapshot` `side_effects.rs:3058` | global, `d` = viewer hex, `p` = viewer (so the `#p`-gated read path scopes it to its owner), one `h` per hidden DM | `(kind, relay_pubkey, d=viewer)` |
| 30618 git ref state | `emit_initial_ref_state` `side_effects.rs:2733` | global | `(kind, relay_pubkey, d=repo_id)` |

Note the inversion: 39000–39002 are stored **channel-scoped** so private member lists
inherit channel access control, at the documented cost that live global subscriptions
(`{kinds:[39000]}`) never receive them (`side_effects.rs:952-960`). 44100/44101 go the
other way — stored **global** precisely so an agent's always-live global subscription can
learn about a new channel.

---

### 9. Caches invalidated (module-level consistency surface)

| Cache | Invalidator | Trigger |
|---|---|---|
| membership | `state.invalidate_membership` | 9000 `side_effects.rs:1217`, 9001 `:1294`, 9007 `:1745`, 9021 `:1878`, 9022 `:1933`, DM open `command_executor.rs:383`, DM add `:543` |
| accessible channels | `state.invalidate_all_accessible_channels` | 9002 visibility flip `side_effects.rs:1426`, 9007 open channel `:1749` |
| channel visibility | `state.invalidate_channel_visibility` | 9002 visibility flip `side_effects.rs:1427` |
| channel-deleted (both membership + accessible) | `state.invalidate_channel_deleted` | 9008 `side_effects.rs:1811`, 9007 insert-failure compensation `ingest.rs:2438` |
| workflow-by-channel | `workflow_engine.invalidate_channel_workflows` | 30620 upsert `command_executor.rs:783`, a-tag workflow delete `side_effects.rs:2004`, `:2030` |
| `author_type_cache` | never invalidated | populated at `ingest.rs:1348-1354`; metric-labelling only, explicitly "never used for authorization" (`ingest.rs:1349-1351`) |
| `local_event_ids` (echo dedupe) | `mark_local_event` then `invalidate` on publish failure | `side_effects.rs:784-798`, `:862-878` |


## Module: buzz-relay — HTTP API surface (`crates/buzz-relay/src/api`)
### Aspect: Data Model

This module owns no persistent schema. Everything below is either a wire DTO, a
cryptographic token structure, or a projection of a `buzz-db` row into JSON.

---

#### 1. Request DTOs (`serde::Deserialize`)

| Type | Fields | Notes | `file:line` |
|---|---|---|---|
| `WebhookQuery` | `secret: Option<String>` | `?secret=` fallback for `/hooks/{id}`; header preferred | `bridge.rs:1770-1775` |
| `ModerationReadQuery` | `status: Option<String>`, `limit: Option<i64>` | `#[derive(Default)]`; `limit` clamped by `clamp_limit` to `1..=500` | `bridge.rs:2080-2090` |
| `MintInviteRequest` | `ttl_secs: Option<u64>` | `#[serde(default)]`; empty body accepted (`invites.rs:249-258`) | `invites.rs:44-50` |
| `ClaimInviteRequest` | `code: String`, `policy_receipt: Option<String>` | `code` is required | `invites.rs:53-59` |
| `AcceptPolicyRequest` | `code: String`, `policy_version: String`, `age_confirmed: bool` (`default`) | `age_confirmed` only consulted when `age_attestation_required` | `invites.rs:62-71` |
| `ListCommunitiesQuery` | `owner_pubkey: String` | required query param | `operator.rs:28-31` |
| `CommunityAvailabilityQuery` | `host: String` | required query param | `operator.rs:34-37` |
| `TransferCommunityRequest` | `community_id: String`, `new_owner_pubkey: String`, `expected_owner_pubkey: String` | private; all three required — `expected_owner_pubkey` is the CAS token | `operator.rs:39-44` |
| `ArchiveCommunityRequest` | `host: String`, `owner_pubkey: String` | shared by archive and unarchive | `operator.rs:196-200` |
| `ProvisionCommunityRequest` | `host: String`, `initial_owner_pubkey: Option<String>`, `create_only: bool` | defined in `handlers/community_provisioning.rs:43-54`, deserialized here | `operator.rs:161` |
| `ReportQuery` | `community_id: Option<Uuid>`, `status`, `report_type`, `target_kind`, `before`/`after: Option<DateTime<Utc>>`, `limit: Option<i64>` | `rename_all="camelCase"` + **`deny_unknown_fields`** — the only strict-schema DTO in the module | `admin/mod.rs:63-73` |
| `Nip05Query` | `name: Option<String>` | lowercased before lookup (`nip05.rs:48`) | `nip05.rs:17-21` |
| `DemoEchoRequest` | `community_id: Uuid`, `session_id: Uuid`, `payload: String` | **`community_id` is client-supplied**, not host-derived | `mesh_demo.rs:48-56` |

`nostr::Filter` (plus the raw `serde_json::Value` shadow) is the de facto request DTO for
`/query` and `/count`; the bridge-only extension fields are enumerated in the api-surface aspect.

#### 2. Response DTOs

| Type | Shape | `file:line` |
|---|---|---|
| `TransferCommunityResponse` | `{community_id, new_owner_pubkey, status: &'static str, previous_owner?}` (`skip_serializing_if`) | `operator.rs:46-53` |
| `ProvisionCommunityResponse` | `{community_id, host, status: "created"\|"existed", owner_pubkey?}` | `handlers/community_provisioning.rs:58-69` |
| `FeedbackSummary` | camelCase `{id, communityId, communityHost, submitterPubkey, category?, bodySummary, receivedAt}` | `admin/mod.rs:139-149` |
| `ErrorEnvelope` / `ErrorBody` | `{"error":{"code","message","requestId"}}`; `requestId` is a **freshly generated** `Uuid::new_v4()` per response, not a correlated trace id | `admin/error.rs:16-28`, `:60-77` |
| `BlobDescriptor` | `{url, sha256, size, mime_type, uploaded, dim?, blurhash?, thumb?, duration?}` — owned by `buzz-media`, mutated here by `rewrite_descriptor_urls_for_tenant` | `media.rs:458-472` |
| `AdminReport`, `AdminFeedback` | `buzz_db::admin_moderation::*`, serialized verbatim (no field filtering) | `admin/mod.rs:93-98`, `:177-182` |

Everything else is ad-hoc `serde_json::json!` — see §5.

#### 3. Invite-token structure and signing (`invite_token.rs`)

```text
code    = base64url_nopad(payload_json) "." base64url_nopad(hmac_sha256(key, payload_json))
key     = sha256(relay_secret_key_bytes ‖ b"buzz-invite-v1")
```

| Element | Detail | `file:line` |
|---|---|---|
| `InvitePayload` | `{c: String (community UUID), r: String (role), e: u64 (unix expiry), n: String (b64 of 16 random bytes)}` | `invite_token.rs:64-74` |
| Key derivation | `derive_invite_key` = `sha256(secret_key ‖ KEY_DERIVATION_LABEL)`; label `b"buzz-invite-v1"` at `:58` | `invite_token.rs:112-117` |
| Signing | `Hmac<Sha256>` over the exact serialized payload bytes | `invite_token.rs:119-123` |
| Mint | `ttl.clamp(60, MAX_INVITE_TTL_SECS)`; `MAX_INVITE_TTL_SECS = 30 d` (`:55`), `DEFAULT_INVITE_TTL_SECS = 72 h` (`:52`); role hard-coded `"member"` | `invite_token.rs:128-149` |
| Verify order | length ≤ `MAX_CODE_LEN`(1024) → split on `.` → b64 decode both → **constant-time `mac.verify_slice`** → deserialize → expiry → community → `r == "member"` | `invite_token.rs:156-192` |
| `InviteError` | `Malformed`, `BadSignature`, `Expired`, `WrongCommunity`, `InvalidRole` | `invite_token.rs:79-92` |
| Nonce | `rand::random::<[u8;16]>()` | `invite_token.rs:133` |

Properties the code asserts and tests: multi-use until expiry (no server-side "used" bit,
`invite_token.rs:32-34`), community-scoped (`:36-38`, test `:227-235`), role-capped even for a
correctly-signed elevated payload (`:39-42`, test `:311-331`), revocation only by rotating the
relay keypair (`:43-46`).

##### Policy-acceptance receipt (same key, second payload type)

| Element | Detail | `file:line` |
|---|---|---|
| `PolicyAcceptancePayload` | `{c: hex(sha256(invite_code)), v: policy version, e: unix expiry}` | `invite_token.rs:335-343` |
| Mint | expiry = `now + 10 min`; same `sign_payload` key | `invite_token.rs:346-359` |
| Verify | length ≤ 2048 → MAC → expiry → `c` must equal `hex(sha256(code))` **and** `v == version`, else `Malformed` | `invite_token.rs:362-390` |

**Domain-separation gap:** both payload types are HMAC'd with the *same* derived key and carry no
purpose label inside the signed bytes. Cross-type confusion is currently blocked only by serde's
missing-field strictness (`InvitePayload` needs `r`+`n`; `PolicyAcceptancePayload` needs `v`), not by
an explicit tag. Adding an optional field to either struct would open the confusion.

#### 4. Webhook-secret model (`webhook_secret.rs`)

| Element | Detail | `file:line` |
|---|---|---|
| Storage location | inside the workflow definition JSON under key `"_webhook_secret"` — no separate column | `webhook_secret.rs:3-5`, `:34-41` |
| Value | `Uuid::new_v4().to_string()` — 122 bits, hyphenated, always 36 chars | `webhook_secret.rs:22-28` |
| Hash-ordering contract | `inject_secret` **must** run before `definition_hash` is computed, else the stored hash never matches | `webhook_secret.rs:7-22` |
| Read | `extract_secret` — `None` when absent or non-string | `webhook_secret.rs:44-50` |
| Redaction helper | `strip_secret` — **zero production callers** | `webhook_secret.rs:52-68` |
| Compare | `verify_secret`: length check (non-constant-time, justified at `:74-79`) then XOR-fold over all bytes | `webhook_secret.rs:70-90` |
| Consumers | mint/inject at `handlers/command_executor.rs:713-718`; verify at `bridge.rs:1823-1844` | — |

#### 5. Admin auth model

There is **no admin principal, token, or session**. The model is:

| Element | Detail | `file:line` |
|---|---|---|
| `AdminConfig` | `{host: String, web_dir: Option<PathBuf>}` | `config.rs:27-34` |
| Enablement | router exists iff `config.admin.is_some()` (i.e. `BUZZ_ADMIN_HOST` non-empty) | `router.rs:57-59`, `config.rs:813-841` |
| `is_admin_host` | exact string equality between the `Host` header and `config.admin.host` | `admin/auth.rs:6-14` |
| `authorize` | `admin` config present (else 404) → `is_admin_host` (else 403) → **if `Origin` present**, `origin_matches_host` (else 403) | `admin/auth.rs:16-33` |
| `origin_matches_host` | strips `https://` then `http://`, compares remainder | `admin/auth.rs:35-40` |
| Tenancy | none — `admin_list_reports`/`admin_list_feedback`/`admin_get_*` are deployment-wide; `communityId` is an *optional filter*, not a boundary | `admin/mod.rs:100-110`, `:151-175` |

The trust boundary is documented as the private ingress, not the application
(`docs/admin/README.md:7-9`, `:64-70`).

#### 6. Media sidecar / read model

The serve path treats **storage as non-authoritative**; the sidecar is the source of truth.

| Element | Detail | `file:line` |
|---|---|---|
| Sidecar reads | `read_sidecar_mime(tenant, hash)` for content-type; `get_sidecar(tenant, hash)` for `.ext` | `media.rs:635-660`, `:810-836` |
| Tenancy | sidecars are tenant-scoped; blobs are shared content-addressed objects, so the sidecar lookup is the per-community gate | `media.rs:632-634` |
| Path grammar | 1–3 dot segments: `{64-hex}` \| `{64-hex}.{ext}` \| `{64-hex}.thumb.jpg`; `ext` must satisfy `is_safe_ext` (1–8 chars, `[a-z0-9]`) | `media.rs:527-583` |
| Ext agreement | for `{hash}.{ext}`, `requested_ext` must equal `sidecar.ext` or 404 | `media.rs:646-658`, `:820-834` |
| Key resolution | bare hash → `{hash}.{sidecar.ext}`, re-validated through `is_safe_ext` | `media.rs:864-882` |
| Thumbnails | always `.thumb.jpg`, content-type forced to `image/jpeg`, parent sidecar must exist | `media.rs:636-643`, `:811-818` |
| Response headers | `Content-Disposition` `inline` iff `buzz_media::serve_inline(mime)` else `attachment`; always `CSP: default-src 'none'`, `X-Content-Type-Options: nosniff`, `Accept-Ranges: bytes` | `media.rs:663-668`, `:678-687`, `:736-744` |
| Cache-Control | `private, max-age=31536000, immutable` when `require_media_get_auth`, else `public, …` | `media.rs:517-523` |
| `UploadAttribution` | `{uploader_name: Option<String>, net: UploadNetworkInfo{ip, port}}`; only built when `upload_records_enabled`; IP read from `config.media.upload_ip_header` and validated public, fail-empty | `media.rs:238-283` |
| URL rewrite | descriptor `url`/`thumb` rewritten to `{scheme}://{tenant_host}/media/{sha256}.{ext}`; ext falls back to `bin` | `media.rs:447-476` |
| `AuthenticatedUpload` | private extractor: `{auth_event, tenant, route_mode, _upload_permit}` | `media.rs:33-41` |
| `UploadPermit` | RAII: global `OwnedSemaphorePermit` + a DashMap in-flight counter decremented on `Drop` | `media.rs:68-86` |
| `ScopedPubkeyKey` | `(CommunityId, [u8; 32])` — the key type for every per-pubkey limiter in this module | `state.rs:37` |

#### 7. Row → JSON projections (moderation reads)

Hand-written, not `Serialize` impls — all byte fields hex-encoded.

| Function | Emits | `file:line` |
|---|---|---|
| `report_json` | `{id, report_event_id, reporter_pubkey, target_kind: "event"\|"pubkey"\|"blob", target, channel_id, report_type, note, status, resolved_by?, resolved_at, action_id, created_at}` | `bridge.rs:2150-2171` |
| `action_json` | `{id, actor_pubkey, action, target_pubkey?, target_event_id?, channel_id, reason_code, public_reason, **private_reason**, matched_principal, created_at}` | `bridge.rs:2173-2188` |
| `ban_json` | `{pubkey, banned, ban_expires_at, ban_reason, muted_until, mute_reason, actor_pubkey, updated_at}` | `bridge.rs:2190-2202` |

`private_reason` is exposed to every ViewQueue-authorized reader (owner/admin) — by design, but note
it is the only field whose name declares it non-public.

#### 8. Standard error envelopes

Two incompatible shapes coexist:

| Shape | Used by | `file:line` |
|---|---|---|
| `{"error": "<message string>"}` | bridge, media (via `MediaError: IntoResponse`), invites, operator, mesh demo | `api/mod.rs:19-21`; `buzz-media/src/error.rs:107-162` |
| `{"error":{"code","message","requestId"}}` | admin only | `admin/error.rs:16-28` |

`internal_error` (`api/mod.rs:23-26`) logs the detail via `tracing::error!` and returns the fixed
string `"internal server error"` — the one place error text is deliberately not reflected.


## Module: buzz-relay — git hosting (`crates/buzz-relay/src/api/git`)
### Aspect: Data Model

#### 1. Storage tiers

Git state lives in **three** places. None of them is a persistent per-repo filesystem.

| Tier | Physical form | Mutability | Authority | Code |
|---|---|---|---|---|
| Object store (S3/MinIO) | `pointer`, `manifests/<sha256>`, `packs/<sha256>`, `idx/<pack-sha256>` | pointer mutable (CAS); rest create-only | **source of truth** | `store.rs:170-908` |
| Postgres | `git_repo_names` (name registry + quota), events (kind 30617/30618) | mutable | name allocation only | `handlers/side_effects.rs:2441-2480` |
| Local disk | per-request bare-repo tempdir; process-lifetime pack cache | ephemeral | pure cache/scratch | `hydrate.rs:293-379`, `pack_cache.rs:66-426` |

`store.rs:22-24` states the read-side rule explicitly: bytes are verified against the key digest, so deviation is *detectable*, not silent.

#### 2. Object-store key scheme

| Key | Producer | Body | Content type | Precondition |
|---|---|---|---|---|
| `repos/<community-uuid>/<owner-hex>/<repo>/pointer` | `manifest.rs:181-184` | 64-hex manifest digest, bare (no prefix, no JSON) | `application/json` (mislabeled — body is bare hex) | `If-Match: <etag>` or `If-None-Match: *` (`store.rs:481-514`) |
| `manifests/<sha256(bytes)>` | `store.rs:337-340` via `put_immutable` (`:254-280`) | canonical manifest JSON | `application/json` | `If-None-Match: *` |
| `packs/<sha256(bytes)>` | `store.rs:282-285` | raw git packfile | `application/x-git-pack` | `If-None-Match: *` |
| `idx/<pack-sha256>` | `store.rs:294-321` | `.idx` sidecar | `application/x-git-index` | `If-None-Match: *` |
| `probe/pointer-<uuid>`, `probe/inm-race/<sha256>`, `probes/pack…` | `store.rs:589-877` | probe payloads | mixed | mixed |

Three properties worth naming:

- **The pointer namespace is community-scoped; the CAS namespace is not.** `pointer_key` interpolates `CommunityId` (`manifest.rs:181-184`, pinned by `manifest.rs:491-523`), but `manifests/`, `packs/`, and `idx/` are global. Two communities pushing byte-identical packs share one object. That is intentional (content addressing) and safe, but it means **cross-community existence of a pack digest is observable to anyone with bucket read access**.
- **The idx key is *not* content-addressed on its own bytes** — it is keyed by the *pack* digest (`store.rs:227-241`). The doc comment says this keeps the manifest schema unchanged so readers can derive `idx/<pack_digest>`. Consequence: idx bytes are unverified on read (`hydrate.rs:388-408` writes them, then validates via `git verify-pack`, regenerating on failure). This is the only object read where digest verification is replaced by a subprocess check.
- **`.git` suffix is stripped at the key boundary** (`manifest.rs:182`), so `/git/o/r` and `/git/o/r.git` address the same pointer.

#### 3. Manifest format (`manifest.rs:49-73`)

```json
{"version":1,"head":"refs/heads/main","refs":{"refs/heads/main":"aaaa…"},"packs":["packs/<64hex>"],"parent":null}
```

| Field | Type | Meaning | Constraint | Enforced |
|---|---|---|---|---|
| `version` | `u32` | schema version, currently `1` | must `== MANIFEST_VERSION` on read | `manifest.rs:263-272`, const `:34` |
| `head` | `String` | symbolic HEAD target, **unprefixed** (no `ref: `) | non-empty + `is_safe_refname` | `manifest.rs:216-221` |
| `refs` | `BTreeMap<String,String>` | refname → hex oid | ≤ `MAX_MANIFEST_REFS` = 10 000 (`:47`); each key `is_safe_refname`, each value `is_hex_oid` | `manifest.rs:210-232` |
| `packs` | `Vec<String>` | full store keys, sorted+deduped | ≤ `MAX_MANIFEST_PACKS` = 128 (`:39`); each `is_pack_key` | `manifest.rs:205-209`, `:233-237` |
| `parent` | `Option<String>` | **bare** 64-hex digest of superseded manifest | `is_manifest_digest` (lowercase hex only) | `manifest.rs:238-242` |

Canonicalization (`manifest.rs:255-261`): clone → `packs.sort()` → `packs.dedup()` → `serde_json::to_vec`. Determinism rests on (a) `BTreeMap` iteration order, (b) declaration field order, (c) serde's no-whitespace output. Pinned byte-for-byte by `manifest.rs:550-568` and `:367-385`.

Notable asymmetries:

- `canonical_bytes` deliberately does **not** call `validate()` (`manifest.rs:251-254`) so the validation seam is visible at the write site (`cas_publish.rs:1177-1190`).
- `is_hex_oid` accepts 40 **and** 64 hex (`manifest.rs:156-158`) but `is_manifest_digest` (`:165-167`) accepts only lowercase 64. So a ref oid may be uppercase hex while a parent digest may not.
- `head` is stored in `refs`-key form and *not* required to be present in `refs`. `manifest.rs:367-385` pins that the announce seed is `head="refs/heads/main"` with `refs:{}` — a legal manifest whose HEAD dangles.

##### Derived constants

| Const | Value | Derivation | Site |
|---|---|---|---|
| `MANIFEST_VERSION` | 1 | literal | `manifest.rs:34` |
| `MAX_MANIFEST_PACKS` | 128 | literal | `manifest.rs:39` |
| `PACK_COMPACTION_THRESHOLD` | 96 | `MAX_MANIFEST_PACKS * 3 / 4` | `manifest.rs:45` |
| `MAX_MANIFEST_REFS` | 10 000 | literal | `manifest.rs:47` |
| `MAX_MANIFEST_BYTES` (read cap) | 4 MiB | literal | `hydrate.rs:45` |
| `MAX_REF_SNAPSHOT_BYTES` | 4 MiB | literal | `cas_publish.rs:270` |

#### 4. Pointer / CAS model

```
pointer(repo) = (ETag e, digest d)      store.rs:455-479 — one GET returns both
manifest(d)   = get_verified("manifests/"+d, d)   hydrate.rs:262-266
packs         = manifest.packs                    hydrate.rs:302-329
```

Types:

| Type | Shape | Site |
|---|---|---|
| `ETag(pub String)` | opaque, quoting preserved verbatim | `store.rs:37` |
| `Precond` | `IfNoneMatchStar \| IfMatch(ETag)` | `store.rs:41-46` |
| `CasOutcome` | `Won(ETag) \| LostRace` — 412 is a *value*, not an error | `store.rs:58-64` |
| `ParentState` | `{ if_match: Option<ETag>, parent_digest: Option<String>, parent: Manifest }` | `cas_publish.rs:215-228` |
| `CasSuccess` | `{ manifest: Manifest, manifest_key: String }` | `cas_publish.rs:192-197` |

`ParentState::fresh()` (`cas_publish.rs:232-243`) models the first push as `if_match=None`, `parent_digest=None`, and an **empty manifest with `head = ""`** — the only place a manifest with empty `head` exists in memory; `validate()` rejects it (`manifest.rs:216-218`), which is exactly the guard `cas_publish.rs:1885-1904` pins.

**One GET, not HEAD-then-GET.** `get_pointer` extracts ETag and body from the *same* response (`store.rs:455-479`) and errors if the ETag header is absent (`:462-471`). `classify_cas` similarly fails closed when a 2xx PUT omits the ETag (`store.rs:516-537`) rather than returning `ETag("")`.

#### 5. Ref-state representation across layers

The same ref state has four distinct on-the-wire shapes. Divergence between them is the module's main data-model hazard.

| Layer | HEAD form | Ref form | Site |
|---|---|---|---|
| Manifest (storage) | bare `refs/heads/main` | `BTreeMap<refname, oid>` | `manifest.rs:53-73` |
| Hydrated bare repo | `HEAD` file = `ref: refs/heads/main\n` | loose file per ref, body `oid\n` | `hydrate.rs:355-371` |
| pkt-line advertisement | `<oid> HEAD\0…symref=HEAD:<ref>` | `<oid> <refname>\n`, BTreeMap order | `transport.rs:471-537` |
| kind:30618 event | tag `["HEAD","ref: refs/heads/main"]` | tag `[<refname>, <oid>]` per ref | `manifest_event.rs:88-104` |

Filters differ per layer, and they are **not** the same predicate:

| Predicate | Accepts | Site |
|---|---|---|
| `is_safe_refname` | any `refs/…`, alphabet `[A-Za-z0-9/_.-]`, no `..`/`//`/leading-or-trailing `/` | `manifest.rs:142-154` |
| `is_emittable_ref` (30618) | only `refs/heads/*` and `refs/tags/*` | `manifest_event.rs:117-127` |
| `fast_path_eligible` | no `refs/tags/*` at all; `head ∈ refs`; refname ≤ 4096 | `transport.rs:401-419` |
| policy-endpoint ref check | `refs/…` prefix, ≤ 256 bytes, no `..`, no byte ≤ 0x20 or 0x7f | `policy.rs:225-233` |

So `refs/notes/*`, `refs/stash`, `refs/pull/*` can be stored in a manifest and hydrated, but are **silently dropped** from kind:30618 (`manifest_event.rs:82-93`, pinned `:255-289`). Subscribers reading only 30618 see an incomplete ref set. Same for oids that are not 40/64 hex (`manifest_event.rs:129-131`, `:310-335`) — skipped, not failed.

##### Snapshot → manifest conversion is 40-hex only

`snapshot_workspace_state` (`cas_publish.rs:268-352`) parses `git for-each-ref --format=%(refname) %(objectname)` and **skips any ref whose oid is not exactly 40 hex** with a `warn!` (`cas_publish.rs:325-328`). Every other layer accepts 64-hex (`manifest.rs:156`, `manifest_event.rs:129`, `transport.rs:475-479` even derives `object-format=sha256` from a 64-char oid). A SHA-256 repository would therefore snapshot to `refs = {}` and publish a manifest that **silently deletes every ref**. Unreachable today because `init_bare_repo` (`hydrate.rs:181-184`) always creates a SHA-1 bare repo, but the failure mode is silent-drop rather than fail-closed.

#### 6. Pack-cache on-disk layout (`pack_cache.rs`)

```
<BUZZ_GIT_PACK_CACHE_PATH>/                     # rejected if a symlink — pack_cache.rs:118-126
  session-<rand>/                               # one per process — pack_cache.rs:127-132
    .heartbeat                                  # mtime touched every 60s — pack_cache.rs:20, :133-146
    <d0><d1>/                                   # 2-hex shard of the pack digest — pack_cache.rs:271
      .staging-<rand>/                           # pack + idx built here — pack_cache.rs:274-279
      <64-hex-digest>/                           # published by atomic rename — pack_cache.rs:311-331
        pack-<digest>.pack
        pack-<digest>.idx
```

In-memory index (`pack_cache.rs:53-64`): `HashMap<digest, CacheRecord{ entry: Arc<CachedPack>, last_used: u64 }>` plus `total_bytes` and a monotonic `tick` used as a logical LRU clock (`:241-251`). `CachedPack` (`:24-51`) records `pack_bytes`, `total_bytes = pack + idx`, and an optional `_temporary: TempDir` that keeps an *over-capacity* (bypassed) entry alive without ever entering the index (`:303-313`).

Installation into a request workspace is **hard-link first, copy fallback** (`pack_cache.rs:428-456`), so cache and workspace share inodes; a copy fallback increments `buzz_git_pack_cache_copy_fallbacks_total`.

Cross-process publication races are resolved by re-reading `CachedPack::from_dir` after a failed rename (`pack_cache.rs:314-330`) — the design assumes multiple relay processes may share the scratch volume.

Startup GC deletes any sibling `session-*` directory whose `.heartbeat` (falling back to the dir) mtime is older than `STALE_SESSION_AGE` = 10 min (`pack_cache.rs:21`, `:482-509`).

#### 7. Hydrated workspace layout (`hydrate.rs:293-379`)

```
<BUZZ_GIT_REPO_PATH>/.tmpXXXXXX/     # TempDir, dropped with HydratedRepo — hydrate.rs:294-299
  HEAD                               # "ref: <manifest.head>\n" — written LAST, hydrate.rs:373-375
  objects/pack/pack-<digest>.{pack,idx}   # phase 1 — hydrate.rs:302-329
  refs/…                             # loose, one file per manifest ref — phase 2, hydrate.rs:355-371
  hooks/pre-receive                  # write path only, 0o755 — hook.rs:152-178
```

`HydratedRepo` (`hydrate.rs:51-77`) carries `hydrated_bytes` and `hydrated_packs`, which flow into `PublishLimits.parent_hydrated_bytes` (`cas_publish.rs:148-155`) and the `buzz_git_hydrate_bytes`/`_packs` histograms (`hydrate.rs:135-136`).

**The phase boundary is the data-model invariant**: packs are all present and indexed before any ref file exists, so a failed hydrate yields "no refs" rather than "refs pointing at missing objects" (`hydrate.rs:288-292`, `:300-304`, `:351-353`).

#### 8. Relation of kind 30617 / 30618 to stored objects

| Event | Author | `d` tag | Role w.r.t. object store |
|---|---|---|---|
| kind **30617** (NIP-34 repo announcement) | repo **owner** | `repo_id` | authorization input only. Read by the policy endpoint to resolve protection rules + channel binding (`policy.rs:283-302`). Its arrival triggers name reservation + pointer seeding (`handlers/side_effects.rs:2405-2578`). |
| kind **30618** (NIP-34 repo state) | **relay** keypair | `repo_id` (must equal 30617's `d`) | *derived notification*, never the commit (`transport.rs:1662-1746`; doc §System Model). Built from `CasSuccess.manifest` — the bytes that landed. |

Mapping details:

- The relay signs 30618, not the pusher; the pusher/owner rides in a `p` tag as a Buzz extension (`manifest_event.rs:106-107`, doc-commented `:17-22`).
- `d` must be the **stripped** repo id. `PushContext.repo_id` is set from `validate_repo_id`'s return (`transport.rs:860`, `:948`), while `PushContext.repo` keeps the raw URL segment for `pointer_key` (`:947`, `:1521-1526`). A mismatch would make 30618 un-joinable to 30617 — pinned by `manifest_event.rs:382-393`.
- Announce seeds the pointer with the canonical empty manifest **before** publishing 30618 (`handlers/side_effects.rs:2606-2694`, then `:2733`). All empty manifests across all repos and communities are byte-identical, hence one shared `manifests/<digest>` object (`handlers/side_effects.rs:2624-2627`; pinned `manifest.rs:373-384`).
- `DEFAULT_HEAD = "refs/heads/main"` is pinned once (`handlers/side_effects.rs:2611`) and shared by the seed manifest and the initial 30618 so they cannot drift.
- Emission is skipped when `parent_digest == committed_digest` (`transport.rs:1677-1683`), and the DB dedups a repeat (`transport.rs:1712-1719`).

#### 9. HTTP-layer data shapes

`HookCallbackRequest` (`policy.rs:53-86`) / `HookRefUpdate` (`:73-86`) are the only JSON bodies the module accepts:

```json
{"repo_id":"…","repo_owner":"<64hex>","community_id":"<uuid>","pusher_pubkey":"<64hex>",
 "ref_updates":[{"old_oid":"<40hex>","new_oid":"<40hex>","ref_name":"refs/…","is_ancestor":true}],
 "timestamp":<u64>,"signature":"<hex hmac-sha256>"}
```

Response: `HookCallbackResponse { allowed: bool, denials: Vec<DenialResponse{ref_name, reason}> }` (`policy.rs:87-104`), `denials` omitted when empty (`:91-92`).

HMAC pre-image (`policy.rs:113-157`) is length-prefixed and `|`-separated:
`len(repo_id):repo_id | repo_owner | community_id | pusher_pubkey | (old_oid ‖ new_oid ‖ len(ref):ref ‖ "1"/"0")* sorted by ref_name | timestamp`.

#### 10. Deltas found against documentation

| Claim | Source | Reality |
|---|---|---|
| "manifest … containing `m.packs` … `m.refs`" | doc §System Model | Manifest also carries `version`, `head`, `parent` — `head` and `parent` are load-bearing (`manifest.rs:53-73`) |
| `ParentState` at `cas_publish.rs:154`; `cas_publish` at `:410`; `CasError::Conflict` at `:92` | doc §Implementation Correspondence table | Actual: `:215`, `:997`, `:105`. The doc warns line numbers are pinned at landing time. |
| "`build_git_response` (sole `Body::from(stdout)` site)" … "has two call sites" shared with read paths | doc §Implementation Correspondence | **Stale.** Both call sites are inside `finalize_push` (`transport.rs:1574`, `:1748`). Read paths now use `stream_git_read` (`:1414`) and an inline builder in `info_refs_subprocess` (`:715-721`). Push-side uniqueness is now trivially true. |
| "pointer-absence means never announced" | doc §Implementation Correspondence | Not enforced on the write path: `hydrate_for_write` creates a fresh workspace for an absent pointer (`hydrate.rs:227-244`) and `cas_publish` will `IfNoneMatchStar` a pointer into existence (`cas_publish.rs:1230-1237`) without consulting kind:30617 or `git_repo_names`. See security aspect. |
| `store.rs` module comment "wired in by the push path in a follow-up commit" beside `#![allow(dead_code)]` | `store.rs:25` | Stale — the push path is wired in (`cas_publish.rs:826`, `:1194`, `:1235`); the blanket allow now masks real dead code. |
| `hydrate.rs:24-30` "We narrow `#[allow(dead_code)]` to those specific items" | `hydrate.rs:24-30` | Stale — there is no `#[allow(dead_code)]` anywhere in `hydrate.rs`. |


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

`matched_principal` is documented as recording the NIP-OA principal an enforcement check matched (`moderation.rs:139`) and is surfaced verbatim by the mod-audit read API (`api/bridge.rs:2184`) — it is therefore **always `null` in production**. Confirmed: `insert_moderation_action` has exactly one caller (`moderation_commands.rs:529`).

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
| `ModerationTarget<'a>` (`:51-58`) | `Event(&[u8])`, `Pubkey(&[u8])`, `None` | all 3 constructed (`moderation_commands.rs:161/240/279/343/404`, `api/bridge.rs:2060`) |
| `ModerationAuthority` (`:62-69`) | `CommunityOwner`, `CommunityAdmin`, `ChannelRole` | `ChannelRole` is **unreachable in production** — see api-surface |

`ModerationAuthority` is returned to every caller and **discarded by all of them** (`moderation_commands.rs:165`, `api/bridge.rs:2062`). The doc comment claims it is "recorded in the audit row" (`moderation_authz.rs:61`) — it is not; `insert_audit` takes no authority parameter (`moderation_commands.rs:518-527`).

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
| Two independent definitions of kind 30350 | `buzz-core/src/kind.rs:109` (`30350`) and `push_lease.rs:19` (`30_350`); ingest imports the `push_lease` copy (`ingest.rs:204`, `:450`, `:2156`) while `req.rs:1734` uses the `buzz-core` one |
| `class_rank` duplicated | `push_lease.rs:575`, `push_runtime.rs:567` |
| `extract_tag_value` duplicated (identical body, 3 copies) | `moderation_commands.rs:608`, `relay_admin.rs:49`, `identity_archive.rs:189` |
| `extract_p_tag_*` triplicated with divergent semantics — `moderation_commands.rs:561` returns *first* valid; `relay_admin.rs:33` returns *first* valid as hex; `identity_archive.rs:170` requires *exactly one* | see cited lines |
| ±120 s freshness constant re-declared 3× | named const `moderation_commands.rs:81`; bare literal `relay_admin.rs:125`; bare literal `identity_archive.rs:147`; a 4th as `ALLOWED_SKEW` `push_lease.rs:476` |


## Module: buzz-relay — huddle audio, tunnel & conformance seam (`crates/buzz-relay/src`)
### Aspect: Data Model

Scope: `audio/{mod,wire,room,mesh,join,handler}.rs`, `tunnel/{mod,directory,reliable}.rs`, `mesh_boot.rs`, `conformance/{mod,tracers}.rs` (9,266 LOC verified line-by-line).

---

#### 1. Audio room state — in-process only

Nothing in this module is persisted. Every structure below lives in relay process
memory and is lost on restart. The only durable artefacts are the four Nostr
lifecycle events (`48101`/`48102`/`48103`, see api-surface) written through
`buzz-db`, and the Redis lease/generation keys in §5.

##### 1.1 Registry → Room → Peer

| Type | Definition | Key / shape | Notes |
|---|---|---|---|
| `AudioRoomManager` | `audio/room.rs:490-492` | `DashMap<(CommunityId, Uuid), Arc<Room>>` | Single global instance, `state.rs:571`, constructed `state.rs:768`. Key is **(community, channel)** — `room.rs:503-509` doc states channel UUIDs are only unique inside a community |
| `Room` | `audio/room.rs:157-170` | `community_id`, `channel_id`, `peers: DashMap<Uuid, AudioPeer>`, `guard: std::sync::Mutex<AdmissionGuard>`, `roster_tx: broadcast::Sender<RosterDelta>` (cap **64**, `room.rs:179`) | Room UUID key is the *channel* id; peer map key is a fresh `Uuid::new_v4()` per connection (`room.rs:255`, `:314`) |
| `AudioPeer` | `audio/room.rs:20-30` | `pubkey: String` (hex), `audio_tx: mpsc::Sender<Bytes>`, `ctrl_tx: mpsc::Sender<PeerCtrl>`, `peer_index: u8` | Two channels per peer so control is never starved by audio (`room.rs:24-27`) |
| `PeerCtrl` | `audio/room.rs:31-36` | `Json(String)` \| `Close` | **`Close` has zero producers** in the whole workspace; consumed only at `audio/handler.rs:1112` |
| `AdmissionGuard` | `audio/room.rs:97-131` | `next_fresh: u8`, `free: Vec<u8>`, `ended: bool`, `pinned_version: Option<u8>`, `roster_revision: u64` | The single mutex that serialises admission, ending, version pinning and roster revisioning |

##### 1.2 Capacities (all hard-coded)

| Constant | Value | Line | Meaning |
|---|---|---|---|
| `AUDIO_CHANNEL_CAPACITY` | 8 | `room.rs:40` | 160 ms at 20 ms/frame; `try_send`, drop-on-full |
| `CTRL_CHANNEL_CAPACITY` | 32 | `room.rs:45` | state-bearing; overflow logs a warning (`room.rs:441-446`) |
| `MAX_PEERS_PER_ROOM` | 25 | `room.rs:50` | soft cap; the doc math is `N×(N−1)` copies/tick → 600 at 25 |
| index space | 0..=254 usable | `room.rs:146-152` | `alloc()` returns `None` once `next_fresh == 255`, so index **255 is never issued** |
| `roster_tx` broadcast | 64 | `room.rs:179` | lag is recoverable via `roster_snapshot` |

##### 1.3 Room lifecycle state machine

State is the tuple `(peers.len(), guard.ended, guard.pinned_version, guard.roster_revision)`.

| From | Event | To | Code |
|---|---|---|---|
| absent | `get_or_create` | `(0, false, None, 0)` | `room.rs:503-509` |
| `(n, false, pin, r)` | `add_peer(v)` ok | `(n+1, false, pin∨v, r+1)` | `room.rs:228-276` |
| `(n, false, pin, r)` | `add_peer_at_index(v,i)` ok | `(n+1, false, pin∨v, r+1)` | `room.rs:281-334` |
| `(n, _, _, r)` | `remove_peer` | `(n−1, unchanged, pin, r+1)` | `room.rs:337-355` |
| `(1, false, …, r)` | `remove_peer_and_check_ended` | `(0, **true**, pin, r+1)` + `should_end=true` | `room.rs:362-388` |
| `(n, false, …)` | `mark_ended` | `(n, true, …)`, returns `peers.is_empty()` | `room.rs:192-199` |
| `(n, true, …)` | `clear_ended` (archive rollback) | `(n, false, …)` | `room.rs:201-205`, called `handler.rs:843` |
| `(0, *, *, *)` | `cleanup_if_empty` | absent (pin cleared) | `room.rs:545-550` |

`ended` is **sticky except via `clear_ended`**. `pinned_version` survives peer churn
(pinned by test `room.rs:730-757`) and is only cleared by eviction. `roster_revision`
uses `wrapping_add(1)` (`room.rs:257`, `:322`, `:342`, `:369`) — it can wrap to 0
after 2^64 mutations, at which point the ingress gap detector
(`join.rs:1568` `next == revision.wrapping_add(1)`) still matches, so the wrap is
consistent, not a bug.

##### 1.4 Roster value types (two parallel definitions)

| Room-local | Wire | Fields |
|---|---|---|
| `RosterPeer` `room.rs:55-61` | `RosterEntry` `join.rs:930-936` | `pubkey: String`, `peer_index: u8` |
| `RosterSnapshot` `room.rs:64-70` | `RosterSnapshot` `join.rs:938-945` | `revision: u64`, `peers: Vec<…>` |
| `RosterDelta` `room.rs:73-81` | `HuddleControlMsg::RosterDelta` `join.rs:895-903` | `revision`, `joined: Option<…>`, `left: Option<…>` |

Only `RosterPeer → RosterEntry` has a conversion (`join.rs:947-954`); the
snapshot/delta conversions are hand-written at `join.rs:1414-1433`. The room-local
types are **not** `Serialize`; the wire types are. This duplication is deliberate
(the room never learns about the mesh, `mesh.rs:38-42`) but is uncovered by any
test that would catch field drift between the two.

`roster_snapshot()` sorts by `peer_index` (`room.rs:471`); `peer_pubkeys()` does
**not** sort (`room.rs:479-484`) — so the `joined.peers` array a same-pod client
receives is in `DashMap` iteration order, while a cross-pod client's is sorted.

---

#### 2. Audio wire frame format

##### 2.1 Client → relay binary frame (protocol v2)

`audio/wire.rs:22-33` defines the v2 header; `V2_HEADER_LEN = 8`, `FLAG_DTX = 0x01`.

| Offset | Width | Field | Encoding | Trust |
|---|---|---|---|---|
| 0..=1 | u16 | `seq` | big-endian, wraps at 2^16 | client-authored, unvalidated |
| 2..=5 | u32 | `ts_48k` | big-endian, 48 kHz media clock | client-authored, unvalidated |
| 6 | i8 | `level_dbov` | canonical `-127..=0` | **untrusted telemetry**; clamped to `-127` when out of range (`wire.rs:73-77`) |
| 7 | u8 | `flags` | bit 0 = DTX, rest reserved & passed through | client-authored |
| 8.. | var | Opus payload | fully opaque | never decoded |

`FrameHeader` (`wire.rs:36-48`) is `Copy`. `parse` (`wire.rs:65-88`) returns
`Option<(FrameHeader, &[u8])>`; `None` only on `len < 8`. A bad `level_dbov` never
drops the frame — the module's stated threat-model invariant (`wire.rs:14-20`) and
its pinning test (`wire.rs:129-147`).

##### 2.2 Relay → client binary frame

`[peer_index: u8][client frame verbatim]`. Built in `Room::broadcast_frame`
(`room.rs:398-402`) with `BytesMut::with_capacity(1 + frame.len())`. The relay
never re-encodes: `prefixed` is cloned per receiver (`room.rs:409`), so a frame in
an N-peer room allocates once and refcount-clones N−1 times.

`deliver_prefixed` (`room.rs:422-429`) takes an already-prefixed buffer and skips
by `peer_index` rather than peer UUID — the cross-pod path, where the author is
remote and has no local peer id.

##### 2.3 Mesh media datagram

`MeshDatagram` (`buzz-relay-mesh/src/wire.rs:111-122`): `{ fenced: FencedHeader,
seq: u64, payload: Vec<u8> }`. For huddles the payload is
`[peer_index][v2 header][Opus]` — **peer_index is always payload byte 0, both
directions** (`mesh.rs:30-35`). Built by `media_datagram` (`join.rs:1795-1809`) on
the ingress→owner leg and by `spawn_remote_peer_sink` (`mesh.rs:262-283`) on the
owner→ingress leg. Two independent `seq` counters exist per session leg
(`RemoteHuddleSession.seq` `join.rs:1508`, and the sink's local `seq`
`mesh.rs:264`), both `wrapping_add(1)`, both never read by any consumer.

An empty payload after a valid fence is dropped with a warning (`mesh.rs:229-232`).
`media_datagram` will happily emit a 1-byte payload (index only, no audio) —
pinned by `join.rs:2914-2916`.

---

#### 3. `HuddleControl` stream schema (postcard)

`HuddleControlMsg` (`join.rs:846-928`) — **7 variants**, postcard-encoded into
`MeshStreamFrame::Data.payload` (`join.rs:1010-1013`, `:1018-1020`).

| # | Variant | Direction | Payload |
|---|---|---|---|
| 1 | `RegisterPeer` | non-owner → owner | `community_id: Uuid`, `pubkey: String`, `protocol_version: u8` |
| 2 | `PeerRegistered` | owner → non-owner | `pubkey`, `peer_index: u8`, `roster: RosterSnapshot` |
| 3 | `RosterSnapshot` | owner → non-owner | `revision: u64`, `peers: Vec<RosterEntry>` |
| 4 | `RosterDelta` | owner → non-owner | `revision`, `joined: Option<RosterEntry>`, `left: Option<RosterEntry>` |
| 5 | `RosterResync` | non-owner → owner | unit |
| 6 | `RegisterRejected` | owner → non-owner | `pubkey`, `reason: RegisterRejection` |
| 7 | `UnregisterPeer` | non-owner → owner | `pubkey: String` |

`community_id` rides as a raw `Uuid`, not a `CommunityId`, **because `CommunityId`
is deliberately non-`Deserialize`** (`join.rs:851-865`); the owner re-mints it with
`CommunityId::from_uuid` at `join.rs:1211`.

`RegisterRejection` (`join.rs:960-975`) — 4 variants: `RoomFull`, `RoomEnded`,
`VersionMismatch{pinned,requested}`, `Fenced(FenceRejection)`.
`FenceRejection` (`join.rs:982-992`) — 4 variants mirroring the non-`Serialize`
`MeshError` fence arms 1:1: `StaleGeneration`, `NoActiveLease`, `OwnerMismatch`,
`FutureGeneration`; wire codes at `join.rs:1010-1017`.

`HUDDLE_CONTROL_PROFILE = Profile::HuddleControl` (`join.rs:1027`).
`HUDDLE_SESSION_ENDED = GoodbyeReason::SessionEnded` (`join.rs:1451`).

---

#### 4. Ownership / join value types

| Type | Line | Shape |
|---|---|---|
| `Ownership` | `join.rs:186-192` | `{ owner_runtime_id: RuntimeId, generation: u64 }` |
| `HuddleLease` | `join.rs:202` | newtype over `SessionLease`, `pub(crate)` field |
| `AcquireOutcome` | `join.rs:239-246` | `Acquired(HuddleLease)` \| `Held(Ownership)` |
| `HuddleRenewOutcome` | `join.rs:216-222` | `Renewed(HuddleLease)` \| `Lost` |
| `HuddleReleaseOutcome` | `join.rs:226-232` | `Released` \| `NotOwner` |
| `JoinOutcome` | `join.rs:249-268` | `LocalOwner{generation}` \| `RemoteOwner{owner_runtime_id, generation}` |
| `ResolvedJoin` | `join.rs:305-311` | `{ outcome, acquired: Option<HuddleLease> }` — `Some` **only** on the CAS-win arm |
| `HuddleTeardownCause` | `join.rs:1483-1495` | `OwnerLost` \| `OwnerDraining` \| `SessionEnded` \| `StreamClosed` |
| `DialError` | `join.rs:1646-1653` | `Rejected(RegisterRejection)` \| `Mesh(MeshError)` |
| `RemoteHuddleSession` | `join.rs:1489-1510` | `peer_index`, `roster`, `fenced`, `owner`, `pubkey`, `transport`, `seq` |

##### 4.1 `HuddleOwnerRegistry` — per-pod owner state

`join.rs:600-604`: `entries: DashMap<Uuid /*session_id*/, HuddleOwnerEntry>` plus a
sticky `draining: AtomicBool`.
`HuddleOwnerEntry` (`join.rs:606-623`): `lost`, `draining`, `cancel`
(`CancellationToken` ×3) and `generation: u64` — the fence token for
`release`/`drain` (`join.rs:734-742`, `:750-761`).
`HuddleOwnerSignals` (`join.rs:627-633`): `{ lost, draining }`, returned atomically
by `attach_signals` so a CAS winner cannot miss a concurrent drain.

##### 4.2 `GenerationFloor` — local stale-frame guard

`mesh.rs:89-91`: `seen: DashMap<Uuid /*session_id*/, u64 /*highest generation*/>`.
Monotone-only: `check` accepts `>= floor`, advances on `>`, rejects `<`
(`mesh.rs:102-128`). `FenceVerdict` (`mesh.rs:132-146`): `Accept{advanced: bool}` \|
`RejectStale{known: u64}`. `forget` deletes the entry (`mesh.rs:131-133`).
**Unbounded map** — one entry per session ever seen, only removed by `forget`
(called from `handler.rs:755`, `:763`, `:909`), never by a TTL sweep.

---

#### 5. Session directory — Redis keys, leases, fences

`tunnel/directory.rs` is the sole Redis surface in this group.

##### 5.1 Key space

`SessionKeys::new` (`tunnel/directory.rs:453-461`):

| Key | Pattern | TTL | Purpose |
|---|---|---|---|
| lease | `buzz:{community_id}:tunnel:{session_id}:lease` | `PX ttl_ms`, default **30 s** (`directory.rs:17`) | current owner, expires |
| generation | `buzz:{community_id}:tunnel:{session_id}:generation` | **none — never expires** | monotone `INCR` counter |

Shape pinned by test `directory.rs:625-637`. For huddles, `session_id == channel_id`
(`join.rs:20`, `handler.rs:324`), so the huddle lease key is
`buzz:{community}:tunnel:{channel_uuid}:lease`.

##### 5.2 Lease value encoding

`{owner_hex}|{generation}|{profile_wire}` — a `|`-delimited string, written in Lua
(`directory.rs:31`) and parsed by `parse_lease` (`directory.rs:495-531`).
`profile_wire ∈ {reliable-stream, realtime-media, huddle-control}`
(`directory.rs:473-479` / `:486-493`). Validation: exactly 3 parts, `generation != 0`,
owner must be 64 hex chars → `[u8; 32]`. Rejections pinned at `directory.rs:639-648`.

##### 5.3 `SessionLease` and results

| Type | Line | Fields |
|---|---|---|
| `SessionLease` | `directory.rs:88-101` | `community_id`, `session_id`, `owner_runtime_id`, `generation: u64`, `profile` |
| `AcquireResult` | `directory.rs:105-111` | `Acquired(SessionLease)` \| `Exists(SessionLease)` |
| `RenewResult` | `directory.rs:115-124` | `Renewed(SessionLease)` \| `Lost{current: Option<SessionLease>, known_generation: Option<u64>}` |
| `ReleaseResult` | `directory.rs:128-137` | `Released(SessionLease)` \| `NotOwner{current, known_generation}` |
| `DirectoryError` | `directory.rs:141-176` | 6 variants: `Pool`, `Redis`, `MalformedLease`, `MalformedGeneration`, `UnexpectedScriptStatus`, `InvalidLeaseTtl` |

##### 5.4 Four Lua scripts (atomic units)

| Script | Lines | Returns `{status, value, known_generation}` |
|---|---|---|
| `ACQUIRE_SCRIPT` | `:20-34` | `exists` (no INCR) \| `acquired` (INCR then SET PX) |
| `RENEW_SCRIPT` | `:36-52` | `missing` \| `renewed` (PEXPIRE) \| `lost` |
| `RELEASE_SCRIPT` | `:54-70` | `missing` \| `released` (DEL) \| `lost` |
| `VALIDATE_SCRIPT` | `:72-79` | 2-tuple `{lease, known_generation}` — read-only |

The generation counter is INCR'd **only** inside `ACQUIRE_SCRIPT` when no live
lease exists (`:26-33`), which is what makes generation monotone across owner death
even though the lease key expires (pinned `directory.rs:712-777`).

##### 5.5 Fence-token semantics (`validate_fenced_header`, `directory.rs:348-439`)

`FencedHeader` (`buzz-relay-mesh/src/wire.rs:85-93`) = `{session_id, generation,
owner_runtime_id}`. Verdict order:

1. `known = max(lease.generation, counter)` (`:375-380`)
2. `known > 0 && frame.generation < known` → `StaleGeneration` (`:382-389`)
3. no live lease → `NoActiveLease` (`:391-406`)
4. `frame.generation != lease.generation` → `FutureGeneration` (`:408-422`)
5. `frame.owner != lease.owner` → `OwnerMismatch` (`:424-437`)

Each rejection increments `mesh_fence_rejections_total{reason}` (`:481-483`) —
the **only** metric in this entire 9,266-line group.

---

#### 6. Reliable-stream data structures

| Type | Line | Shape |
|---|---|---|
| `ReliableStreamRouter<T: ?Sized>` | `reliable.rs:36-41` | `directory`, `transport: Arc<T>`, `local_runtime_id` |
| `ReliableJoin` | `reliable.rs:207-218` | `Owned{lease}` \| `Forwarded{lease, stream}` |
| `ReliableInbound` | `reliable.rs:221-228` | `fenced`, `from`, `stream` |
| `ReliableMeshStream` | `reliable.rs:231-235` | `fenced`, `stream: MeshStream`, `community_id: Option<CommunityId>` (latched) |
| `ReliableWireFrame` (private) | `reliable.rs:412-422` | `Data{community_id, payload}` \| `Goodbye{community_id, reason}` |
| `ReliableFrame` (public) | `reliable.rs:521-527` | `Data(Vec<u8>)` \| `Goodbye(GoodbyeReason)` |
| `ReliableStreamError` | `reliable.rs:529-568` | 10 variants |

##### 6.1 Reliable inner frame — a hand-rolled binary format

`encode`/`decode` at `reliable.rs:434-492`:

```
byte 0      VERSION = 1                (reliable.rs:423)
byte 1      kind: DATA=1 | GOODBYE=2   (reliable.rs:424-425)
bytes 2..18 community_id UUID (16B, raw big-endian bytes)
bytes 18..  Data: opaque payload  |  Goodbye: 1 byte reason
```

`GoodbyeReason` ↔ byte: `SessionEnded=1`, `Draining=2`, `StaleGeneration=3`
(`reliable.rs:501-518`). `decode` rejects `len < 18`, wrong version, unknown kind,
and a `Goodbye` whose length ≠ 19. Note `bytes[2..18].try_into().expect(...)` at
`reliable.rs:471` is infallible given the length check but is still an `expect` in
a production path.

`MAX_RELIABLE_PAYLOAD_BYTES = 1 MiB` (`reliable.rs:31`), chunked by
`send_bytes` (`reliable.rs:280`); the mesh wire cap is 16 MiB
(`buzz-relay-mesh/src/wire.rs:46`), asserted at `reliable.rs:945`. There is **no**
minimum-size, rate, or total-bytes bound.

---

#### 7. Boot / dispatch structures

| Type | Line | Shape |
|---|---|---|
| `MeshHandle` | `mesh_boot.rs:135-169` | `directory`, `transport`, `membership`, `local_runtime_id`, `dispatcher`, `audio_fence: Arc<GenerationFloor>`, private `runtime: MeshRuntime`, `owners: Arc<HuddleOwnerRegistry>` |
| `MeshInboundDispatcher` | `mesh_boot.rs:57-60` | `slots: Arc<DispatcherSlots>` |
| `DispatcherSlots` | `mesh_boot.rs:62-67` | 3 × `OnceLock`: `huddle_control`, `reliable_stream`, `datagrams` |
| `SessionStreamHandler` | `mesh_boot.rs:39` | `Box<dyn Fn(RuntimeId, StreamHello, MeshStream) + Send + Sync>` |
| `DatagramHandler` | `mesh_boot.rs:42` | `Box<dyn Fn(RuntimeId, MeshDatagram) + Send + Sync>` |

`AppState.mesh: Arc<OnceLock<MeshHandle>>` (`state.rs:627`), published once at
`main.rs:458`; `AppState::mesh()` returns `Option<&MeshHandle>` (`state.rs:812-814`).

---

#### 8. Conformance abstract model

Re-exported wholesale from `buzz-conformance` at `conformance/mod.rs:38-44`.

| Type | Definition | Shape |
|---|---|---|
| `AbstractState` | `buzz-conformance/src/lib.rs:150-161` | `resolved_community: CommunityLabel`, `bound_host: HostLabel`, `actor: ActorLabel` |
| `TraceStep` | `buzz-conformance/src/lib.rs:290-299` | `schema_version: u32`, `action: TraceAction`, `state_after: AbstractState` |
| `TraceAction` | `buzz-conformance/src/lib.rs:179-260` | **9 variants** (below) |
| `Verdict` | re-exported `conformance/mod.rs:39` | `Allow` \| `Deny` |
| `SanitizedReason` | re-exported | `Invalid` \| `Restricted` \| `ServerError` (mapping `conformance/mod.rs:422-429`) |

`TraceAction` variants: `WriteInsert`, `WriteInsertGlobal`, `WriteDuplicate`,
`SanitizedError`, `AuthCheck`, `ReadMessageRows`, `ReadByIdRows`,
`ReadHostFeedRows`, `ImplBug`. **`ReadHostFeedRows` has no emitter anywhere in the
relay** — its only occurrences outside the schema are the conformance crate's own
proptest (`buzz-conformance/tests/proptest_checker.rs:164`, `:255`).

##### 8.1 Label projections

| Label | Source | Line | Fidelity |
|---|---|---|---|
| `resolved_community` | `TenantContext::community()` | `conformance/mod.rs:57` | server-resolved only |
| `bound_host` | `TenantContext::host()` | `conformance/mod.rs:58` | full host string, **not** hashed |
| `actor` | first 16 hex chars of the pubkey | `conformance/mod.rs:70-74` | **not** blake3 — the doc comment at `conformance/mod.rs:51-53` claims "lower 16 bytes of `blake3(pubkey_bytes)`"; the code is a plain hex prefix. Documented delta, rationale given at `:64-69` |
| `msg_id` | first 8 bytes of the event id, hex | `conformance/mod.rs:78-85` | truncation, not hashing |
| `channel` | direct `Uuid` wrap | `conformance/mod.rs:89-91` | not opaque |
| `claimed_community` | first `h` tag parsed as UUID | `conformance/mod.rs:101-117` | `None` when absent or unparseable; recorded **separately** from resolved so the M2 bite survives |

##### 8.2 Row-community projection

`RowCommunityProjection` (`conformance/mod.rs:216-231`): `Ok(Vec<CommunityLabel>)` \|
`MissingLookup{kind: &'static str, first_missing_channel: Uuid}`.
`project_row_community` (`:186-197`): channel-less row → resolved label;
channel-scoped row → lookup in `HashMap<Uuid, CommunityId>` or `None`. A `None`
becomes `MissingLookup` and is emitted as `ImplBug{kind:
"row_community_lookup_missing"}` (`:286-294`, `:321-329`) — never silently
substituted. Pinned by four tests (`:614-696`).

##### 8.3 `EmitGuard` / `CountingTracer`

`EmitGuard` (`conformance/mod.rs:344-353`): `inner: Arc<dyn Tracer>`, `state`,
`counter: Arc<AtomicUsize>`, `kind: &'static str`.
`CountingTracer` (`:357-360`) wraps the real tracer and `fetch_add(1, Relaxed)` per
record (`:368-372`).
`Drop` (`:403-414`): if `counter == 0`, records
`TraceAction::ImplBug{kind}` on the **inner** tracer.
`arm` returns `(Self, Arc<dyn Tracer>)` (`:383-400`) — callers must pass the
returned wrapper downstream, or the counter never moves.

---

#### 9. Type-name collisions inside one crate (verified)

| Name | Definition A | Definition B | Distinct? |
|---|---|---|---|
| `AdmissionError` | `audio/room.rs:83-95` — `Ended`/`Full`/`VersionMismatch{pinned,requested}` | `admission.rs:12` — `Exceeded`/`Unavailable` | **Yes, two unrelated types.** Both are live: the audio one at `handler.rs:513`/`:524`/`:535` and `join.rs:1441-1449`; the other in `crate::admission`. Neither imports the other; nothing renames on import |
| `Ownership` | `audio/join.rs:186-192` (used by `HuddleDirectory`) | `audio/mesh.rs:74-80` (used by the dead `HuddleOwnerDirectory`) | Yes — identical fields, two definitions in sibling modules |
| `NoopTracer` | `conformance/tracers.rs:20` (production binding, `state.rs:798`) | `buzz-conformance/src/lib.rs:323` (**zero users**) | Yes |
| `RosterSnapshot` / `RosterDelta` | `audio/room.rs:64`,`:73` | `audio/join.rs:938` / `HuddleControlMsg::RosterDelta` | Yes — §1.4 |
| "mesh" | `buzz-relay-mesh` (inter-relay QUIC) | `mesh-llm-sdk` dev-dep (`Cargo.toml:87-88`), consumed only by `examples/mesh_*.rs` | Two unrelated meanings in one crate |


## Module: buzz-relay-mesh (`crates/buzz-relay-mesh`)
### Aspect: Data Model

Scope: the 10 files of `crates/buzz-relay-mesh` (3,169 LOC). No SQL, no migrations,
no ORM — this crate's persistence surface is **one Redis key namespace** plus
in-memory tables. All types below verified by reading the source.

---

#### 1. Identity: `RuntimeId`

| Item | Location | Definition |
|---|---|---|
| `RuntimeId(pub [u8; 32])` | `wire.rs:62` | ed25519 public key of a **boot-unique** mesh keypair generated at process start |
| `to_hex()` | `wire.rs:65` | lowercase hex, 64 chars |
| `Debug` | `wire.rs:70-74` | truncates to `RuntimeId(xxxxxxxx…)` (first 8 hex chars) |
| `Display` | `wire.rs:76-80` | full 64-char hex — this is what lands in Redis keys and JSON |
| Derives | `wire.rs:61` | `Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize` |

Generation site: `SecretKey::generate()` → `runtime_id_from_public_key(secret_key.public())`
(`endpoint.rs:20`, `endpoint.rs:38`, `endpoint.rs:96-98`). The reverse map is
`public_key_from_runtime_id` (`endpoint.rs:100-102`).

**Deliberate non-identity** (documented at `wire.rs:52-60`): the RuntimeId is *not*
the deployment's Nostr/secp256k1 relay key, because the helm chart shares one
`BUZZ_RELAY_PRIVATE_KEY` across all pods — reusing it would collapse the ownership
plane. Verified: nothing in the crate derives a RuntimeId from a Nostr key.

The tuple field is `pub` (`wire.rs:62`), so any consumer can construct an arbitrary
`RuntimeId` value without going through iroh. Used that way in tests
(`membership.rs:394`, `runtime.rs:...`), and in `buzz-relay` tests
(`mesh_boot.rs:592`).

---

#### 2. Wire model (serialization codec = **postcard**, not JSON)

Codec: `postcard` v1 with `use-std` (`Cargo.toml:14`, workspace pin
`Cargo.toml:65`). Framing helpers:

| Function | Location | Format |
|---|---|---|
| `wire::encode<T>` | `wire.rs:176-179` | `[WIRE_VERSION: u8]` ++ `postcard(T)` |
| `wire::decode<T>` | `wire.rs:182-188` | version-byte check then `postcard::from_bytes` |
| `gossip::encode_message` | `gossip.rs:62-64` | **bare postcard, NO version byte** |
| `gossip::decode_message` | `gossip.rs:66-77` | postcard, then checks the in-struct `version` field |

Two different versioning schemes coexist: the outer frame uses a leading byte
(`WIRE_VERSION = 1`, `wire.rs:42`); the gossip payload nested inside
`MeshStreamFrame::Gossip{payload}` uses an in-struct `version: u8` field
(`GOSSIP_PAYLOAD_VERSION = 1`, `gossip.rs:13`). Documented as deliberate at
`wire.rs:139-141` ("opaque at this layer so gossip can evolve without a wire bump").

##### 2.1 Wire message type count

**Exactly 5 top-level wire payload types**, 2 of which are envelopes:

| # | Type | Location | Transport |
|---|---|---|---|
| 1 | `MeshDatagram` | `wire.rs:111-122` | one per QUIC datagram, no length prefix |
| 2 | `MeshStreamFrame` | `wire.rs:126-144` | u32-LE length + postcard, on QUIC bi-streams |
| 3 | `StreamHello` | `wire.rs:160-163` | carried inside `MeshStreamFrame::Hello` |
| 4 | `GossipMessage` | `gossip.rs:51-60` | carried inside `MeshStreamFrame::Gossip.payload` |
| 5 | `ReadyRecord` | `registry.rs:99-113` | **JSON** in Redis (`serde_json`, `registry.rs:185`) |

`MeshStreamFrame` has **4 variants** (`wire.rs:128,131,140,144`): `Hello`, `Data`,
`Goodbye`, `Gossip`. `GossipMessage` has **2 variants** (`gossip.rs:52,56`):
`Digest`, `Delta`. `StreamRole` has **2 variants** (`wire.rs:151,154`): `Control`,
`Session`. `Profile` has **3 variants** (`wire.rs:99,101,107`): `ReliableStream`,
`RealtimeMedia`, `HuddleControl`. `GoodbyeReason` has **3 variants**
(`wire.rs:168,170,172`): `SessionEnded`, `Draining`, `StaleGeneration`.

##### 2.2 `FencedHeader` — the fenced tuple

`wire.rs:85-93`: `{ session_id: Uuid, generation: u64, owner_runtime_id: RuntimeId }`.
Derives `Clone, Copy, Debug, PartialEq, Eq, Serialize, Deserialize` (`wire.rs:84`).

Present on `MeshDatagram.fenced`, `MeshStreamFrame::Data.fenced`,
`MeshStreamFrame::Goodbye.fenced`, and `StreamRole::Session.fenced`. **Absent from
`MeshStreamFrame::Gossip`** (`wire.rs:144-146`) and from `StreamRole::Control` —
control-plane traffic is unfenced by construction.

`generation` is documented (`wire.rs:87-89`) as the monotonic lease generation from
the Redis CAS; `owner_runtime_id` is documented as **advisory** (`wire.rs:90-92`).
No code in this crate reads or compares `generation` — the fence checks live in
`buzz-relay` (`tunnel/directory.rs:378,395,413,430`; `audio/join.rs:1081,1201,1888`).

##### 2.3 Size bounds

| Bound | Value | Location | Enforced at |
|---|---|---|---|
| `MAX_STREAM_FRAME` | 16 MiB (`16 * 1024 * 1024`) | `wire.rs:46` | send `peer.rs:142-147`, recv `peer.rs:178-183` |
| datagram max | `Connection::max_datagram_size()` (runtime value) | — | `encode_datagram_checked` `lib.rs:213-226`, `peer.rs:106-110` |
| datagram header overhead budget | ≤ 64 B (test-asserted, not runtime-enforced) | `wire.rs:271-274` | test `datagram_header_overhead_within_budget` |
| ALPN | `b"buzz/mesh/1"` | `wire.rs:37` | checked at `peer.rs:50-55` |

---

#### 3. Membership / peer model

##### 3.1 `GossipRecord` — the per-runtime replicated record

`gossip.rs:16-27`. Derives `Clone, Debug, PartialEq, Serialize, Deserialize`
(`gossip.rs:15`; note: **no `Eq`** — `load: f32`).

| Field | Type | Notes |
|---|---|---|
| `runtime_id` | `RuntimeId` | primary key of the record |
| `endpoint_addrs` | `Vec<String>` | dialable socket addrs as **strings**, parsed lazily at `runtime.rs:328` |
| `proto_version` | `u16` | set by relay to `WIRE_VERSION as u16` (`mesh_boot.rs:367`) |
| `load` | `f32` | advisory load factor; **never written** — see §7 |
| `draining` | `bool` | set only via `begin_drain` (`membership.rs:361-364`) |
| `capabilities` | `Vec<String>` | relay ships a static 3-item list (`mesh_boot.rs:371-377`) |
| `version` | `u64` | per-runtime monotonic; **only the owning runtime increments its own** (`gossip.rs:23-24`) |
| `heartbeat_millis` | `u64` | ms since UNIX_EPOCH, refreshed on every version bump |

Constructor `GossipRecord::new` (`gossip.rs:30-42`) seeds `version = 1`,
`load = 0.0`, `draining = false`, `capabilities = vec![]`, `heartbeat_millis = now_millis()`.

##### 3.2 `PeerState` (private) and the membership table

`membership.rs:20-26`: `{ record: GossipRecord, phi: PhiAccrual, connection_state:
ConnectionState, counters: MeshPeerCounters }`.

`MeshMembership` (`membership.rs:29-43`) is a cloneable handle over shared state:

| Field | Type | Location |
|---|---|---|
| `local_runtime_id` | `RuntimeId` (immutable copy) | `membership.rs:30` |
| `local_record` | `Arc<RwLock<GossipRecord>>` | `membership.rs:31` |
| `peers` | `Arc<RwLock<HashMap<RuntimeId, PeerState>>>` | `membership.rs:32` |
| `draining` | `Arc<AtomicBool>` | `membership.rs:33` |
| `stale_generation_rejections` | `Arc<AtomicU64>` | `membership.rs:34` |
| `foreign_relay_rejections` | `Arc<AtomicU64>` | `membership.rs:35` |
| `expected_relay_pubkey` | `Option<String>` (hex) | `membership.rs:41` — `None` is fail-closed |
| `phi_suspect_threshold` | `f64`, default `8.0` | `membership.rs:42`, const `membership.rs:17` |

**The local record is stored separately from `peers`** and re-appended on every
`records()` call (`membership.rs:195-206`), so `peers.len()` never counts self.

**There is no eviction path.** `grep` for `.remove(`/`retain`/`clear()` in
`membership.rs` returns nothing. Peers only ever transition
`connection_state → Disconnected` (`runtime.rs:275-280`); the entry, its
`endpoint_addrs`, and its counters persist for the process lifetime. See
`buzz-relay-mesh-debt.md` D-02 and `-security.md` S-03.

##### 3.3 `GossipState` — a second, parallel model

`gossip.rs:81-83`: `{ records: HashMap<RuntimeId, GossipRecord> }` with
`new/records/get/update_local/digest/delta_for/apply_delta`
(`gossip.rs:86,92,96,100,111,127,151`).

This is a **complete duplicate** of the scuttlebutt logic that `MeshMembership`
reimplements (`membership.rs:120,166,208,225`). `GossipState` has **zero callers
outside its own module's tests** (verified by grep across `crates/**`). It is the
purer of the two (no `local_runtime_id` special-casing, `apply_delta` returns changed
ids) but nothing uses it. See `-debt.md` D-01.

##### 3.4 Version / conflict-resolution semantics

| Rule | Location |
|---|---|
| Local version bump is `saturating_add(1)` + `heartbeat_millis = now_millis()` | `membership.rs:174-176`; mirror `gossip.rs:105-107` |
| Remote record applied only when `record.version > peer.record.version` | `membership.rs:127-135` (`<=` → `false`) |
| `GossipState` variant | `gossip.rs:154-157` (`is_none_or(|e| record.version > e.version)`) |
| Self-records rejected | `membership.rs:121-123` (`apply_gossip_record`), `membership.rs:87-89` (`apply_ready_records`) |
| Delta selection: send records the remote is missing or behind on | `membership.rs:229-243`; `gossip.rs:127-149` |
| Digest/Delta ordering: sorted by `runtime_id.to_hex()` | `membership.rs:216`, `membership.rs:241`; `gossip.rs:118`, `gossip.rs:144` |

Last-version-wins, no vector clocks, no tie-break on equal versions (equal → ignored).

---

#### 4. Failure detector state: `PhiAccrual`

`gossip.rs:168-172`: `{ samples: Vec<Duration>, last_heartbeat: Option<SystemTime>,
max_samples: usize }`. `Default` = `new(100)` (`gossip.rs:174-178`);
`max_samples` floored at 1 (`gossip.rs:185`).

`observe(at)` (`gossip.rs:189-201`): pushes `at - last_heartbeat` when positive and
non-zero, evicts front via `samples.remove(0)` when over `max_samples`
(`gossip.rs:195-197`, O(n) shift), then sets `last_heartbeat = at`.

`phi_at(now)` (`gossip.rs:203-215`) returns `None` when no heartbeat seen, when
`samples` is empty (i.e. exactly one heartbeat observed), or when `mean <= f64::EPSILON`;
otherwise:

```
phi = (elapsed_secs / mean_secs) / LN_10          // gossip.rs:214
```

**Delta vs the name.** This is *not* the Hayashibara phi-accrual detector: the
sample **variance is never used** (`mean_secs`, `gossip.rs:217-220`, is the only
statistic consumed). The doc comment at `gossip.rs:213` is accurate about what the
code does (`-log10(e^(-elapsed/mean))`), but the type name implies a
distribution-aware detector it does not implement. Concretely, with the default
`DEFAULT_GOSSIP_INTERVAL = 2s` (`runtime.rs:44`) the mean settles near 2s, so
`phi >= 8.0` requires `elapsed >= 8 * ln(10) * 2s ≈ 36.8s` of silence.

`observe` is called from exactly two places, both in `apply_gossip_record`
(`membership.rs:132`, `membership.rs:139`) — phi advances **only when a strictly
newer record arrives**, never from raw connection liveness.

---

#### 5. Redis registry model

| Constant | Value | Location |
|---|---|---|
| `READY_KEY_PREFIX` | `"mesh:ready:"` | `registry.rs:19` |
| key template | `mesh:ready:{runtime_id_hex}` | `ready_key`, `registry.rs:150-152` |
| `DEFAULT_REGISTRY_REFRESH` | 15 s | `registry.rs:20` |
| `REGISTRY_EXPIRY_MULTIPLIER` | 3 | `registry.rs:21` |
| TTL | `refresh * 3`, min 1 s | `expiry_for` `registry.rs:154-156`; `ttl_secs` `registry.rs:187` |
| `ATTESTATION_CONTEXT` | `"buzz-relay-mesh-ready-v1"` | `registry.rs:22` |

Commands used: `SET key payload EX ttl` (`registry.rs:188-194`),
`DEL key` (`registry.rs:201-204`), `SCAN cursor MATCH mesh:ready:* COUNT 100`
+ per-key `GET` (`registry.rs:217-228`). Value encoding is **JSON**
(`serde_json::to_string`, `registry.rs:185`) — the only non-postcard payload in
the crate. Key namespace is **global**: not community/tenant scoped and not
deployment-prefixed; cross-deployment isolation relies entirely on the
`relay_pubkey` anchor check (`membership.rs:90-102`).

##### 5.1 `ReadyRecord`

`registry.rs:99-113`, derives `Clone, Debug, PartialEq, Serialize, Deserialize`
(`registry.rs:98` — no `Eq`).

| Field | Type | Notes |
|---|---|---|
| `runtime_id` | `RuntimeId` | serialized by serde as a 32-byte array |
| `runtime_pubkey` | `String` | **explicit duplicate** of `runtime_id.to_hex()`; cross-checked at `registry.rs:140-145` |
| `relay_pubkey` | `String` | Nostr/secp256k1 hex |
| `relay_sig` | `String` | schnorr signature, `Signature::to_string()` |
| `endpoint_addrs` | `Vec<String>` | |
| `proto_version` | `u16` | |
| `capabilities` | `Vec<String>` | |

`RuntimeAttestation` (`registry.rs:30-34`) is the `{relay_pubkey, relay_sig}` pair
alone; it is embedded field-wise into `ReadyRecord` rather than nested
(`registry.rs:120-130`). `RuntimeAttestation` as a named type has **zero callers
outside `ReadyRecord::new`** (`registry.rs:121`) and its own `verify` method
(`registry.rs:48-50`) has zero callers at all.

Signed preimage (`registry.rs:85-91`) — textual and versioned to avoid JSON
key-order dependence:

```
buzz-relay-mesh-ready-v1\nruntime_pubkey={hex}\nrelay_pubkey={hex}
```

hashed with SHA-256 (`registry.rs:93-96`) then schnorr-signed/verified
(`registry.rs:41`, `registry.rs:78-80`).

##### 5.2 `ReadyHeartbeat`

`registry.rs:280-284`: `{ registry: ReadyRegistry, record: ReadyRecord, published:
bool }`. The `published` bool is the only state machine: `tick(ready)` publishes on
`ready`, clears on the `ready→not-ready` edge (`registry.rs:295-304`), `shutdown()`
clears if published (`registry.rs:306-312`).

`ReadyRegistry` itself (`registry.rs:160-163`) holds `{ pool: deadpool_redis::Pool,
refresh: Duration }` — no cursor, no cache, no local copy of scan results.

---

#### 6. Runtime model

`Inner` (`runtime.rs:65-73`): `{ endpoint: MeshEndpoint, membership: MeshMembership,
registry: Option<ReadyRegistry>, peers: RwLock<HashMap<RuntimeId, PeerEntry>>,
handler: Mutex<Option<Arc<dyn InboundHandler>>>, gossip_interval, reconcile_interval }`.

`PeerEntry` (`runtime.rs:50-55`): `{ peer: MeshPeer, control_tx:
Option<mpsc::Sender<MeshStreamFrame>>, tasks: Vec<JoinHandle<()>> }`.
`control_tx` is `None` on the accept side until a `Hello{Control}` arrives
(`runtime.rs:236-247` vs `runtime.rs:441-448`) — and `gossip_tick_loop` only targets
entries where `control_tx.is_some()` (`runtime.rs:576`).

`MeshRuntime` (`runtime.rs:77-80`): `{ inner: Arc<Inner>, loops:
Arc<Mutex<Vec<JoinHandle<()>>>> }`. **Two peer tables exist**: `Inner.peers`
(live QUIC connections, keyed by RuntimeId) and `MeshMembership.peers`
(gossip records). They are only loosely coupled via `mark_connection_state`
(`runtime.rs:250-252`, `:277-279`, `:314-316`, `:344-346`).

`CONTROL_QUEUE_DEPTH = 64` (`runtime.rs:46`) bounds queued control frames per peer;
overflow drops via `try_send` (`runtime.rs:556`).

`MeshPeer` (`peer.rs:38-43`): `{ _endpoint: iroh::Endpoint, conn:
iroh::endpoint::Connection, runtime_id: RuntimeId, counters: Arc<PeerCountersInner> }`.
`PeerCountersInner` (`peer.rs:19-24`) is 4 `AtomicU64`s snapshotted into
`PeerCounters` (`peer.rs:10-15`: `streams_opened, streams_accepted, datagrams_sent,
datagrams_received`).

**Two disjoint counter models.** `PeerCounters` (per-connection, atomics, in
`peer.rs`) and `MeshPeerCounters` (per-membership-entry, in `status.rs:51-61`) track
overlapping quantities with different field names (`streams_accepted` vs
`streams_received`) and are never reconciled. `MeshPeer::counters()` (`peer.rs:73`)
has zero callers; only the `MeshPeerCounters` set reaches `/_mesh`.

---

#### 7. `MeshStatus` — the `/_mesh` JSON shape

All in `status.rs`, all `Serialize`-only (no `Deserialize` — so nothing can
round-trip or contract-test the endpoint's own output).

```
MeshStatus                                        status.rs:7-15
  enabled: bool                                   status.rs:9   (hardcoded true, membership.rs:352)
  local_runtime_id: String                        status.rs:10  (full 64-hex)
  draining: bool                                  status.rs:11
  peer_count: usize                               status.rs:12
  peers: Vec<MeshPeerStatus>                      status.rs:13  (sorted by runtime_id, membership.rs:298)
  counters: MeshCounters                          status.rs:14

MeshPeerStatus                                    status.rs:17-29
  runtime_id: String, endpoint_addrs: Vec<String> status.rs:19-20
  proto_version: u16, draining: bool              status.rs:21-22
  connection_state: ConnectionState               status.rs:23
  phi: Option<f64>, load: f32                     status.rs:24-25
  record_version: u64                             status.rs:26
  last_heartbeat_millis: u64                      status.rs:27
  counters: MeshPeerCounters                      status.rs:28

ConnectionState (snake_case)                      status.rs:31-40
  disconnected (default) | connecting | connected | suspect

MeshCounters                                      status.rs:42-49
  stale_generation_rejections: u64                status.rs:44
  foreign_relay_rejections: u64                   status.rs:47
  peers: Vec<MeshPeerCounters>                    status.rs:48

MeshPeerCounters                                  status.rs:51-61
  runtime_id: String
  streams_opened, streams_received,
  datagrams_sent, datagrams_received,
  gossip_frames_sent, gossip_frames_received,
  stale_generation_rejections: u64
```

Observations, all verified:

- **`Suspect` is derived, never stored.** `peer_statuses` recomputes it per call from
  `phi >= phi_suspect_threshold` (`membership.rs:340-344`); the stored
  `PeerState.connection_state` is only ever Disconnected/Connecting/Connected.
- **`MeshCounters.peers` duplicates every `MeshPeerStatus.counters`** verbatim
  (`membership.rs:302`), so each peer's counters appear twice in the JSON.
- **`MeshStatus.enabled` is a constant `true`** at the source (`membership.rs:352`);
  the `false` case is synthesized by the relay handler (`router.rs:404`), not by
  this data model.
- **`stale_generation_rejections` is structurally always 0 in production.** The only
  writer is `record_stale_generation_rejection` (`membership.rs:285-293`), whose
  sole call site anywhere is the unit test at `membership.rs:486`. The relay's real
  fence rejections are raised in `tunnel/directory.rs` / `audio/join.rs` and never
  routed back here.
- `endpoint_addrs` is echoed verbatim from the gossip record — see `-security.md` S-05.

---

#### 8. `MeshConfig` (config-shaped data, not persisted)

`lib.rs:53-64`: `{ enabled: bool, bind_addr: SocketAddr, registry_refresh: Duration }`.
Constructed only in `buzz-relay` (`config.rs:508-512`). See `-configuration.md`.

---

#### 9. Error taxonomy as data (`MeshError`, `lib.rs:65-127`)

**16 variants.** Four carry the fence-rejection taxonomy with structured payloads
(`StaleGeneration` `lib.rs:96-101`; `NoActiveLease` `lib.rs:110-117`; `OwnerMismatch`
`lib.rs:118-124`; `FutureGeneration` `lib.rs:125-130` — line refs per the enum body
spanning `lib.rs:66-127`). `lib.rs:102-109` documents the intended counter surface
`mesh_fence_rejections_total{reason=stale_generation|no_active_lease|owner_mismatch|future_generation}`
and states none of the four are serialized (the wire signal stays
`GoodbyeReason::StaleGeneration`, `wire.rs:172`).

Verified: **that metric does not exist.** grep for `mesh_fence_rejections_total`
across the repo returns nothing outside this doc comment.

Constructor census (grep across `crates/**`):

| Variant | Constructed in mesh crate | Constructed in `buzz-relay` |
|---|---|---|
| `Encode` / `Decode` | `wire.rs:178,184`; `gossip.rs:63,67` | `audio/join.rs:970,975` |
| `UnknownWireVersion` / `EmptyFrame` | `wire.rs:185,186` | — |
| `FrameTooLarge` | `peer.rs:143,179` | — |
| `DatagramTooLarge` | `lib.rs:219` | — |
| `PeerNotConnected` | `runtime.rs:169,187` | — |
| `Transport` | 22 sites (`endpoint.rs`, `peer.rs`, `registry.rs`, `gossip.rs`) | — |
| `Redis` | implicit via `#[from]` + `?` on `query_async` (`registry.rs:194,204,225,228`) | — |
| `StaleGeneration` | **none** | `tunnel/directory.rs:378,870`; `audio/join.rs:2854` |
| `NoActiveLease` | **none** | `tunnel/directory.rs:395,914` |
| `OwnerMismatch` | **none** | `tunnel/directory.rs:430,824`; `audio/join.rs:1081,1201,1888` |
| `FutureGeneration` | **none** | `tunnel/directory.rs:413,842` |
| `PeerDraining` (`lib.rs:94`) | **none** | **none** — dead variant |
| `Disabled` (`lib.rs:119`) | **none** | **none** — dead variant |

The fencing law holds structurally: the crate that *defines* the fence errors never
raises them. `PeerInfo` even has a compile-time reminder of this in the consumer:
`fn _peer_info_is_not_an_owner_signal(_peer: PeerInfo)` (`tunnel/reliable.rs:949`,
`#[allow(dead_code)]`).

---

#### 10. Documentation deltas

- **ARCHITECTURE.md does not cover this crate at all.** `grep -i "mesh|iroh|quic"
  ARCHITECTURE.md` → zero hits across all 827 lines, including §6 "Crate Reference"
  (`ARCHITECTURE.md:330`). `buzz-relay-mesh` is likewise absent from `AGENTS.md`'s
  repo-structure table, `CONTRIBUTING.md`, and `README.md`.
- `lib.rs:55-57` calls `BUZZ_MESH` "`on` (default when replicas can exist)". The
  implementation defaults it **off** (`config.rs:498-500`). Documented delta —
  see `-configuration.md`.
- `lib.rs:129-139` describes `PeerInfo.load` as "advisory load factor gossiped by the
  peer (0.0..)". No code ever assigns a non-zero `load`: `GossipRecord::new` sets
  `0.0` (`gossip.rs:35`) and the only mutation hook, `update_local(|_| {})`
  (`runtime.rs:566`), is a no-op closure. The field is structurally always `0.0`.
- `lib.rs:184-189` says `MeshStream`'s halves are "placeholder halves … pre-transport";
  they are in fact the real iroh-backed halves (`peer.rs:132-133`, `peer.rs:135-192`).
  Stale comment.


## Module: buzz-conformance (`crates/buzz-conformance`)
### Aspect: Data Model

The crate has **no persistent data model** — no database, no migrations, no ORM. Its "data model" is a
serde wire schema (`TraceStep` JSONL) plus an in-memory `ModelState` the replay checker carries across
one trace. Everything below is a value type.

Crate size: 1,029 LOC of Rust across 3 src files + 755 LOC of tests
(`Cargo.toml` 35, `src/lib.rs` 327, `src/checker.rs` 337, `src/transitions.rs` 330,
`tests/proptest_checker.rs` 431, `tests/replay_fixtures.rs` 324). Two undeclared docs also live in the
crate: `LIMITS.md` (125) and `TRACE_SCHEMA.md` (163).

---

### 1. Opaque label newtypes

All five are `#[serde(transparent)]` newtypes — the JSON carries a bare scalar, not an object.

| Type | Definition | Inner | Derives | Relay-side source |
|---|---|---|---|---|
| `CommunityLabel` | `src/lib.rs:66` | `uuid::Uuid` | `Debug, Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord, Serialize, Deserialize` (`src/lib.rs:65`) | `CommunityLabel::from_uuid(*tenant.community().as_uuid())` — `buzz-relay/src/conformance/mod.rs:57` |
| `OpaqueId` | `src/lib.rs:93` | `String` | same minus `Copy` (`src/lib.rs:92`) | first 16 hex chars of the event id — `buzz-relay/src/conformance/mod.rs:78-85` |
| `HostLabel` | `src/lib.rs:100` | `String` | `src/lib.rs:99` | `tenant.host().to_string()` — `buzz-relay/src/conformance/mod.rs:58` |
| `ChannelLabel` | `src/lib.rs:106` | `uuid::Uuid` | `src/lib.rs:105` | direct wrap of the channel UUID — `buzz-relay/src/conformance/mod.rs:89-91` |
| `ActorLabel` | `src/lib.rs:112` | `String` | `src/lib.rs:111` | first 16 hex chars of the authed pubkey — `buzz-relay/src/conformance/mod.rs:71-75` |

`CommunityLabel::from_uuid` is a public `const fn` (`src/lib.rs:74`). `Display` is implemented only for
`CommunityLabel` (`src/lib.rs:79-83`).

### Documentation deltas on the label types (verified)

1. **`ActorLabel` doc vs. implementation.** `src/lib.rs:110` documents the actor label as *"the lower 16
   bytes of `blake3(pubkey)`. Stable, non-reversible, secret-free."* The only producer,
   `buzz-relay/src/conformance/mod.rs:71-75`, takes `actor.to_hex()[..16]` — a **prefix of the raw
   pubkey**, no hashing at all. The relay-side doc comment (`mod.rs:64-70`) is honest about this and
   explains why; the schema doc comment in this crate is stale. `TRACE_SCHEMA.md:31` also says "first 16
   hex chars of authed pubkey", agreeing with the code and contradicting `lib.rs:110`.
2. **`HostLabel` doc vs. implementation.** `src/lib.rs:96-98` says the label is *"produced by the relay
   from the bound `Host` header via a configured registry, never the raw `Host` string."* There is no
   registry: `mod.rs:58` stores `tenant.host().to_string()` directly. The committed fixture confirms a
   raw hostname on the wire — `tests/fixtures/good.jsonl` line 1 carries `"bound_host":"a.example.test"`.
3. **`OpaqueId` doc** (`src/lib.rs:88-91`) says "derived from an event id or other secret material …
   Implementations pick a hash" — the implementation is a hex prefix (`mod.rs:78-85`), not a hash.

---

### 2. Closed enums

| Type | Variants | Line | Wire form |
|---|---|---|---|
| `Verdict` | `Allow`, `Deny` | `src/lib.rs:117-123` | `snake_case` (`src/lib.rs:116`) → `"allow"` / `"deny"` |
| `SanitizedReason` | `Restricted`, `Invalid`, `ServerError` | `src/lib.rs:132-140` | `snake_case` → `"restricted"` / `"invalid"` / `"server_error"` |

`Verdict` is derived from the TLA+ `AuthCheck` verdict (`docs/spec/MultiTenantRelay.tla:801`, `verdict ==
IF allowed THEN "Allow" ELSE "Deny"`) — faithful, 2-for-2.

**`SanitizedReason` is a 3-way collapse of a 9-element spec alphabet.** `docs/spec/MultiTenantRelay.cfg:26`
sets `SanitizedErrors = {"auth-required", "restricted", "invalid", "duplicate", "pow", "rate-limited",
"blocked", "error", "frame-too-large"}`. The Rust enum has 3 variants. The doc comment at
`src/lib.rs:125-129` calls the set "the sanitized error alphabet (spec `Inv_SanitizedErrors`, M6
mutation)" and `TRACE_SCHEMA.md:110-116` calls it "closed", but closure is asserted against
`IngestError`'s 3 variants (`buzz-relay/src/conformance/mod.rs:422-430`), not against the spec's 9. A
mis-bucketed reason (e.g. relay maps a rate-limit to `Invalid`) is undetectable — see business-rules
BR-CONF-05.

---

### 3. `AbstractState` — the per-request abstract state

`src/lib.rs:150-175`. Three fields, all required, `Debug + Clone + PartialEq + Eq + Serialize +
Deserialize` (`src/lib.rs:149`):

| Field | Type | Line | Maps to relay concept | Spec counterpart |
|---|---|---|---|---|
| `resolved_community` | `CommunityLabel` | `src/lib.rs:155` | `TenantContext::community()` — host-derived, server-resolved | `o.community` on every observation (`MultiTenantRelay.tla:985`) |
| `bound_host` | `HostLabel` | `src/lib.rs:158` | `TenantContext::host()` | the `host` bound variable in `WriteInsert`/`AuthCheck`/reads; `HostCommunity[host]` |
| `actor` | `ActorLabel` | `src/lib.rs:160` | the NIP-42/NIP-98 authenticated pubkey | `o.actor` |

### What the abstract state deliberately omits — and the coverage consequence

The doc comment (`src/lib.rs:147-148`) says the state carries "deliberately … not raw payloads, pubkey
bytes, signatures, or wall-clock timestamps." Verified accurate. But the omissions also remove spec
state the checker would need:

| Spec state variable (`MultiTenantRelay.tla`) | Present in `AbstractState`? | Consequence |
|---|---|---|
| `messages` (row set) | No | `Inv_AcceptedWritesPersist` (`tla:1104`), `Inv_MessageKeyUnique` (`tla:1110`) uncheckable |
| `projections` | No | `Inv_ProjectionDerived` (`tla:1121`) uncheckable |
| `memberships` / `admittedMembers` | No | `Inv_AdmissionFence` (`tla:1071`) uncheckable — acknowledged at `src/transitions.rs:222-224` and `:253-256` |
| `HostCommunity[_]` (the full map) | No — only the one bound host | acknowledged at `src/transitions.rs:110-114`: the checker "does NOT know `HostCommunity[_]` at large" |
| `ChannelCommunity(_)` (channel→community map) | No | the checker cannot recompute a channel's real community, so `WriteInsert`/`WriteDuplicate` legality is unverifiable (BR-CONF-01/03) |
| `createdChannels`, `openChannels`, `auditHeads`, `queryFaults`, `feedReads`, `auxReads`, `authRegistrations` | No | `Inv_HostBindingFence` (`tla:1038`), `Inv_ChannelCommunityImmutable` (`tla:1057`), `Inv_NoTenantContextFailsClosed` (`tla:1116`) uncheckable |
| `worker` / observation ordering | No — explicitly rejected at `src/checker.rs:26-28` | fine: the spec models `observations` as an unordered set |

`src/transitions.rs:24-30` states the modeling reduction honestly: "a runtime trace covers ONE worker
handling ONE request — so the model state we carry is much smaller."

---

### 4. `TraceAction` — 9 variants (all of them)

`src/lib.rs:179-261`. Serde representation is internally-tagged: `#[serde(tag = "type", rename_all =
"snake_case")]` (`src/lib.rs:178`), so each JSON object carries a `"type"` discriminant.

| # | Variant | Line | Fields | `kind()` string | Spec action + line cited in code | Actual spec line |
|---|---|---|---|---|---|---|
| 1 | `WriteInsert` | `src/lib.rs:181-194` | `msg_id: OpaqueId`, `channel: ChannelLabel`, `claimed_community: Option<CommunityLabel>` | `write_insert` (`:267`) | `WriteInsert` 514–550 (`src/lib.rs:186`) | `tla:514` ✅ |
| 2 | `WriteInsertGlobal` | `src/lib.rs:195-203` | `msg_id`, `claimed_community` | `write_insert_global` (`:268`) | 559–595 (`src/lib.rs:187`) | `tla:559` ✅ |
| 3 | `WriteDuplicate` | `src/lib.rs:204-213` | `msg_id`, `channel`, `claimed_community` | `write_duplicate` (`:269`) | 606–637 (`src/lib.rs:188`) | `tla:606` ✅ |
| 4 | `SanitizedError` | `src/lib.rs:214-220` | `reason: SanitizedReason` | `sanitized_error` (`:270`) | 778 (`src/lib.rs:189`) | `tla:778` ✅ |
| 5 | `AuthCheck` | `src/lib.rs:221-230` | `channel`, `claimed_community`, `verdict: Verdict` | `auth_check` (`:271`) | 794 (`src/lib.rs:190`) | `tla:794` ✅ |
| 6 | `ReadMessageRows` | `src/lib.rs:231-240` | `channel: Option<ChannelLabel>`, `row_communities: Vec<CommunityLabel>` | `read_message_rows` (`:272`) | 643 (`src/lib.rs:191`) | `tla:643` ✅ |
| 7 | `ReadByIdRows` | `src/lib.rs:241-249` | `channel: Option<…>`, `row_communities: Vec<…>` | `read_by_id_rows` (`:273`) | 681 (`src/lib.rs:192`) | `tla:681` ✅ |
| 8 | `ReadHostFeedRows` | `src/lib.rs:250-255` | `row_communities: Vec<…>` | `read_host_feed_rows` (`:274`) | "line ~720" (`src/lib.rs:193`) | `tla:703` ⚠️ off by 17 |
| 9 | `ImplBug` | `src/lib.rs:256-261` | `kind: String` | `impl_bug` (`:275`) | not a spec action (`src/lib.rs:194`) | n/a ✅ |

`TraceAction::kind()` (`src/lib.rs:266-277`) is the canonical string used by `Scenario`'s coverage set.
`TraceAction::is_critical()` (`src/lib.rs:283-285`) is a `const fn` that returns `true` unconditionally.

### Spec-line-citation drift in the prose docs

`TRACE_SCHEMA.md` cites different lines than the Rust for the same actions: `WriteInsertGlobal` "line 562"
(`TRACE_SCHEMA.md:57`) vs. actual 559; `WriteDuplicate` "line 612" (`TRACE_SCHEMA.md:69`) vs. actual 606.
The invariant citations in `src/transitions.rs` also drift: `Inv_NonInterference` "line ~983"
(`src/transitions.rs:53`, `:287`) vs. actual `tla:985`; `Inv_ReadConfinement` "line ~1003"
(`src/transitions.rs:54`) vs. actual `tla:999`. All are annotated `~`, so they are approximations, not
false claims — but the un-tilde'd `WriteInsert` range "514–550" and `WriteInsertGlobal` "559–595" are
exact and correct.

### The `claimed_community` / `resolved_community` split

Three of the four write/auth variants carry `claimed_community: Option<CommunityLabel>` separately from
`state_after.resolved_community`. This is the load-bearing non-normalization documented at
`src/lib.rs:188-192` and `buzz-relay/src/conformance/mod.rs:17-19`. Producer:
`claimed_community_from_event` (`buzz-relay/src/conformance/mod.rs:101-119`) reads the event's first `h`
tag and parses it as a UUID; `None` when absent or unparseable.

**Semantic hazard, verified:** the `h` tag on the Buzz write path carries a **channel** UUID, not a
community UUID (`AGENTS.md` "Channel scoping: Channels use `h` tags (NIP-29 group tag)"; and the relay's
own kind:9007 handler treats `h` as the channel id — see `crates/buzz-test-client/tests/conformance_multitenant.rs:319`
building `Tag::parse(["h", &channel_uuid])`). `mod.rs:103-105` admits the ambiguity in a comment ("or
channel uuid, ambiguous"). So `claimed_community` is in practice populated with a **channel** UUID, which
will essentially never equal `resolved_community`. The single rule that consumes it — the AuthCheck M2/M8
bite (`src/transitions.rs:233-242`) — would therefore fire on nearly every real ingest, if a recording
tracer were ever enabled. See debt D-CONF-02.

On the read path `claimed_community` is unconditionally `None` by design, with a rationale doc comment at
`buzz-relay/src/conformance/mod.rs:135-145`.

### `row_communities` is a `Vec`, not a `Set` — deliberately

`src/lib.rs:236-239` and `src/transitions.rs:281-292` both state the reason: the checker must see every
leaked label, and de-duping would let a buggy emitter hide a leak. `check_row_labels`
(`src/transitions.rs:294-310`) walks the slice and fails on the first non-matching entry.

---

### 5. `TraceStep` — the JSONL record

`src/lib.rs:290-299`:

| Field | Type | Line |
|---|---|---|
| `schema_version` | `u32` | `src/lib.rs:292` |
| `action` | `TraceAction` | `src/lib.rs:294` |
| `state_after` | `AbstractState` | `src/lib.rs:298` |

`TraceStep::new` (`src/lib.rs:303-309`) stamps `SCHEMA_VERSION` (= `1`, `src/lib.rs:86`) automatically;
there is no constructor that lets a caller set a different version, so the only way to produce a
version-mismatched step is hand-written JSON.

### Replay-fixture format (JSONL)

One `TraceStep` per line, no envelope, no trailing metadata. Serializer:
`tests/replay_fixtures.rs:178-186` (`to_jsonl`); parser: `tests/replay_fixtures.rs:190-198` (`from_jsonl`,
skips blank lines, panics with the 1-based line number on a parse error). Four fixtures on disk:

| Fixture | Steps | Expected verdict | Builder | Test |
|---|---|---|---|---|
| `tests/fixtures/good.jsonl` | 3 (`auth_check` → `write_insert` → `read_message_rows`) | `Ok(())` | `tests/replay_fixtures.rs:74-101` | `:237-249` |
| `tests/fixtures/bad_host_channel_mismatch.jsonl` | 2 | `IllegalTransition` | `:107-129` | `:251-264` |
| `tests/fixtures/bad_coverage_breach.jsonl` | 1 (`impl_bug`) | `CoverageBreach` | `:134-141` | `:266-278` |
| `tests/fixtures/bad_foreign_row_leak.jsonl` | 1 | `NonInterference` | `:155-168` | `:280-292` |

The actual on-disk key names are `schema_version`, `action`, `state_after` — e.g.
`tests/fixtures/bad_coverage_breach.jsonl:1`. **`TRACE_SCHEMA.md:37-46` documents the record as
`{"schema": 1, "action": …, "state": {…}}`** — three of the four top-level keys are wrong in the doc
(`schema`/`state` vs. `schema_version`/`state_after`). Anyone hand-writing a fixture from the doc gets a
serde error.

Fixtures are byte-compared against the typed builders (`assert_fixture_matches`,
`tests/replay_fixtures.rs:205-235`) and regenerated with `BUZZ_CONFORMANCE_UPDATE=1`
(`tests/replay_fixtures.rs:210`), so a schema change forces a deliberate re-commit. This round-trip
guard works and is the crate's strongest anti-drift mechanism.

---

### 6. `ModelState` — the checker's in-memory model

`src/transitions.rs:105-118`. Fields mirror `AbstractState` 1:1 (`resolved_community`
`:108`, `bound_host` `:115`, `actor` `:117`) and are populated from the **first** trace step by
`ModelState::bootstrap` (`src/transitions.rs:123-129`). The model is **immutable for the lifetime of a
trace** — `check_step` takes `&ModelState` and the doc at `src/transitions.rs:131-132` says it "Updates
nothing." Consequence: the checker verifies *self-consistency across one request*, not evolution of relay
state. It cannot express any spec action that mutates state (`AdmitMember`, `CreateChannel`,
`RebuildProjections`, …).

Because the model is bootstrapped **from the trace itself**, a relay that consistently reported the wrong
resolved community for a whole request would pass every state check. The checker's independence
(`Cargo.toml:8-24`) is about not sharing *code*, not about independently deriving the tenant.

---

### 7. Dead type: `Verdict_`

`src/transitions.rs:53-56` declares `pub enum Verdict_ { Ok }`, documented "Reserved — internal
placeholder." Zero constructors, zero matches, zero references anywhere in `crates/**` (verified by
repo-wide grep). It survives `cargo clippy -p buzz-conformance --all-targets` with no warning because it
is `pub` in a library and the trailing underscore does not trip `non_camel_case_types`.

