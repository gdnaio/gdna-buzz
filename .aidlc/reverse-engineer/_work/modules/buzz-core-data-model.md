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
