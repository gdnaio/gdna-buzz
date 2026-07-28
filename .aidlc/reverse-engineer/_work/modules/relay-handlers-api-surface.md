## Module: buzz-relay — moderation, admin & background workers (`crates/buzz-relay/src`)
### Aspect: API Surface

---

#### 1. Event kinds owned by this module

**14 inbound kinds** across 6 handlers. Counts verified against `buzz-core/src/kind.rs` and the ingest dispatch chain.

| Kind | Owner | Routed at | Stored as an event? | Direct DB effect |
|---|---|---|---|---|
| 1984 | `report::handle_report_event` | `ingest.rs:1560-1570` | **no** | `moderation_reports` insert |
| 9030 | `relay_admin::handle_relay_admin_event` | `ingest.rs:1808-1818` | **no** | `relay_members` insert |
| 9031 | same | same | **no** | `relay_members` delete |
| 9032 | same | same | **no** | `relay_members` role update |
| 9033 | same | same | **no** | `communities.icon` update |
| 9035 | `identity_archive::handle_identity_archive_event` | `ingest.rs:1913-1919` | **yes** (falls through) | `archived_identities` insert |
| 9036 | same | same | **yes** (falls through) | `archived_identities` delete |
| 9040 | `moderation_commands::handle_moderation_command` | `ingest.rs:1574-1587` | **no** | `community_bans` + `moderation_actions` |
| 9041 | same | same | **no** | `community_bans` + `moderation_actions` |
| 9042 | same | same | **no** | `community_bans` + `moderation_actions` |
| 9043 | same | same | **no** | `community_bans` + `moderation_actions` |
| 9044 | same | same | **no** | `moderation_reports` + `moderation_actions` |
| 30350 | `push_lease::accept` | `ingest.rs:2155-2199` | **yes** (atomic with lease) | `push_leases` + source event |
| 42000 | `product_feedback::handle` | `ingest.rs:1538-1552` | **no** | `product_feedback` insert |

**Outbound (relay-signed) kinds emitted by this module:**

| Kind | Emitter | Site |
|---|---|---|
| 0 (`Metadata`) | `moderation_notices::publish_moderation_profile` — `"{host} Moderation"` | `moderation_notices.rs:186-213` |
| 9 (`KIND_STREAM_MESSAGE`) | moderation notice DM | `moderation_notices.rs:160-178` |
| 9 (`KIND_STREAM_MESSAGE`) | workflow `send_message` | `workflow_sink.rs:302-352` |
| 39000 | DM discovery via `emit_group_discovery_events` | `moderation_notices.rs:155` |
| 8000-series NIP-43 member add/remove + membership list | `relay_admin.rs:214-220`, `:274-279`, `:334-336` (delegated to `side_effects.rs:2826/2923/2932`) | |
| 8002 / 8003 / 13535 (NIP-IA) | `identity_archive.rs:104-136` (delegated to `side_effects.rs:3206/3234/3008`) | |
| `HttpAuth` (27235, NIP-98) | outbound gateway auth header, never stored | `push_runtime.rs:551-565` |

##### Routing-order facts (verified)

- **9040–9044 are dispatched *before* the ban/timeout write-block gate** (`ingest.rs:1574-1587` vs the gate at `:1613-1650`), so a timed-out admin can lift a timeout. The gate explicitly exempts both moderation-command and relay-admin kinds (`ingest.rs:1613`).
- The handler re-checks the durable ban itself at `moderation_commands.rs:103-108` → `ensure_actor_not_banned` (`:135-142`). **`relay_admin.rs` performs no such re-check** — grep for `restriction`/`banned` in `relay_admin.rs` returns zero hits. A banned owner/admin whose live disconnect was missed can still issue 9030–9033.
- All five moderation kinds are listed in `is_global_only_kind` (`ingest.rs:429-433`), as are 9030–9033 (`:436-439`), 9035/9036 (`:445-446`) and 30350 (`:450`). 1984 and 42000 are **not** in that list.
- 9035/9036 deliberately fall through to normal storage so the 8002/8003 delta's `["e", request_id]` resolves (`ingest.rs:1909-1912`).

##### Required scopes (`required_scope_for_kind`)

| Kinds | Scope | Site |
|---|---|---|
| 1984, 42000 | `MessagesWrite` | `ingest.rs:212` |
| 9040–9044 | `MessagesWrite` | `ingest.rs:216` |
| 9030–9033 | `AdminUsers` | `ingest.rs:251-256` |
| 9035, 9036 | `UsersWrite` | `ingest.rs:266` |
| 30350 | (author-only kind) | `kind.rs:120` |

> **Contract delta:** `moderation_commands.rs:50` pins "reject channel-scoped API tokens" for 9040–9044. Ingest rejects channel-scoped tokens only for relay-admin kinds (`ingest.rs:1512-1516`) and leave requests (`:1520-1523`); the generic global-event token gate (`ingest.rs:1721-1724`) sits *after* the moderation dispatch's early `return` at `:1582-1586` and is therefore unreachable for 9040–9044. Combined with `Scope::MessagesWrite`, a legacy channel-scoped WS API token held by a community admin can issue a community-wide ban.

---

#### 2. Exact accept/reject strings

##### 2.1 `moderation_commands` (9040–9044)

Three prefix helpers: `invalid()` → `"invalid: {msg}"` (`:552-554`), `error()` → `"error: {msg}"` (`:556-558`), `authz_denial()` → `"restricted: {e}"` (`:548-550`). Pinned by a unit test (`:669-680`).

| Reject string | Site |
|---|---|
| `blocked: you are banned from this community` (no prefix) | `:139` |
| `invalid: event timestamp out of range: created_at=…, now=…, delta=…s (max ±120s)` | `:117-120` |
| `invalid: unexpected moderation command kind: {other}` | `:130-132` |
| `invalid: missing or invalid p tag` | `:147`, `:228`, `:264`, `:331` |
| `invalid: timeout requires an expiration tag` | `:271` |
| `invalid: invalid expiration tag: {raw}` / `invalid: expiration out of range: {secs}` | `:598`, `:602` |
| `invalid: member is not banned` | `:252` |
| `invalid: member is not timed out` | `:355` |
| `invalid: missing or invalid report tag (expect 64-hex event id)` | `:368` |
| `invalid: missing status tag` / `invalid: missing action tag` | `:369`, `:370` |
| `invalid: invalid status: {status} (expect resolved\|dismissed)` | `:381-383` |
| `invalid: invalid action: {action} (expect delete\|kick\|ban\|timeout\|dismiss\|escalate)` | `:389-391` |
| `` invalid: action `dismiss` pairs only with status `dismissed` `` | `:394-396` |
| `invalid: report not found in this community` | `:417` |
| `invalid: report is not open (already resolved or dismissed)` | `:427-429`, `:471-473` |
| `restricted: moderator access required` | via `:549` + `moderation_authz.rs:178` |
| `restricted: an admin cannot ban or time out a community owner or fellow admin` | via `:549` + `moderation_authz.rs:167` |
| `error: database error: {e}` | `:174`, `:250`, `:292`, `:353`, `:416`, `:469` |
| `error: database error checking restriction state: {e}` | `:108` |
| `error: failed to write audit row: {e}` | `:544` |

Accept: `Ok(())`; ingest converts to `IngestResult { accepted: true, message: "" }` (`ingest.rs:1583-1586`).

##### 2.2 `relay_admin` (9030–9033)

Returns **unprefixed** strings; ingest wraps every one as `format!("invalid: {e}")` (`ingest.rs:1811`). This means an authorization failure surfaces to clients as `invalid: actor not authorized: …` rather than `restricted: …` — inconsistent with the moderation and REQ paths.

| Reject string | Site |
|---|---|
| `event timestamp out of range: created_at=…, now=…, delta=…s (max ±120s)` | `:126-129` |
| `database error: {e}` | `:137`, `:201`, `:246`, `:253`, `:311`, `:319`, `:321` |
| `actor not authorized: must be admin or owner` | `:148`, `:177`, `:227` |
| `actor not authorized: must be owner` | `:286` |
| `actor not authorized: only owner can grant admin role` | `:188` |
| `actor not authorized: admins can only remove members` | `:264` |
| `icon contains invalid characters` | `:72` |
| `icon data URL too long: {n} bytes (max 98304)` | `:78-81` |
| `icon must be an http(s) URL or data:image/* URL` | `:84` |
| `icon URL too long: {n} bytes (max 2048)` | `:89-92` |
| `failed to store workspace icon: {e}` | `:162` |
| `missing or invalid p tag` | `:168` |
| `invalid role: use kind:9032 to promote to owner` | `:185` |
| `invalid role: {role}` | `:191`, `:304` |
| `cannot remove yourself` | `:232` |
| `cannot remove the relay owner` | `:258` |
| `member not found: {target_hex}` | `:261`, `:323` |
| `cannot change your own role` | `:291` |
| `cannot set role to owner` | `:301` |
| `cannot change the relay owner's role` | `:322` |
| `missing role tag` | `:295` |
| `unexpected relay admin kind: {other}` | `:340` |

##### 2.3 `report` (1984)

Self-prefixes (unlike relay_admin), so ingest's `IngestError::Rejected` passes them through verbatim (`ingest.rs:1563`).

| String | Site |
|---|---|
| `invalid: report must include a p tag` | `:123` |
| `invalid: report must include exactly one p tag` | `:126` |
| `invalid: report must target only one of e or x` | `:135` |
| `invalid: report must include at most one e tag` | `:138` |
| `invalid: report must include at most one x tag` | `:141` |
| `invalid: report target tag missing report type` | `:202` |
| `invalid: unsupported report type: {value}` | `:207` |
| `invalid: malformed {label}` (labels: `p tag pubkey`, `e tag event id`, `x tag sha256`) | `:213`, `:215`, `:217` |
| `invalid: report target event not found` | `:59` |
| `invalid: report target blob not found` | `:72` |
| `error: database error resolving report target: {e}` | `:58` |
| `error: database error inserting report: {e}` | `:88` |

##### 2.4 `product_feedback` (42000)

| String | Site |
|---|---|
| `invalid: unsupported feedback category` | `:88` |
| `invalid: feedback must include at most one category tag` | `:89` |
| `invalid: feedback body must not be empty` | `:96` |
| `invalid: feedback body exceeds maximum size of 32768 bytes` | `:98-100` |
| `invalid: feedback tags exceed maximum size of 65536 bytes` | `:73-75` |
| `invalid: feedback timestamp is out of range` | `:49` |
| `error: failed to serialize feedback tags: {e}` | `:71` |
| `error: failed to deserialize feedback tags: {e}` | `:77` |
| `error: database error inserting product feedback: {e}` | `:70` (via `:66`) |
| imeta errors | delegated to `crate::api::validate_imeta_tags` / `verify_imeta_blobs` (`:32-35`) |

##### 2.5 `identity_archive` (9035/9036)

Unprefixed; ingest wraps as `invalid: {e}` (`ingest.rs:1917`).

| String | Site |
|---|---|
| `unexpected identity archive kind: {kind}` | `:52` |
| `event timestamp out of range: … (max ±120s)` | `:148-151` |
| `request must include exactly one NIP-70 protected event tag ["-"] (got {n})` | `:158-160` |
| `missing or invalid p tag` | `:57` |
| `replaced-by is not valid on unarchive requests` | `:61` |
| `invalid replaced-by tag` / `invalid replaced-by pubkey` | `:209`, `:216` |
| `replaced-by must differ from target` | `:219` |
| `multiple replaced-by tags` | `:222` |
| `database error: {e}` | `:78`, `:88`, `:242`, `:280` |
| `auth tag must have exactly four elements` / `multiple auth tags` / `missing auth tag` | `:305`, `:308`, `:314` |
| `invalid request auth tag: {e}` | `:262` |
| `request auth owner must equal request signer` | `:264` |
| `request auth time bound not satisfied: created_at {n} >= {bound}` / `<= {bound}` | `:339-341`, `:348-350` |
| `invalid created_at< bound: {b}` / `invalid created_at> bound: {b}` | `:336`, `:345` |
| `target has no live kind:0 profile` | `:284` |
| `live kind:0 author did not match target` | `:288` |
| `invalid live kind:0 auth tag: {e}` | `:292` |
| `live kind:0 no longer attests to request signer` | `:294` |
| `invalid target pubkey: {e}` | `:278`, `:322` |
| `failed to encode auth tag: {e}` | `:317` |
| `invalid auth tag json: {e}` / `auth tag missing conditions` | `:330`, `:332` |

##### 2.6 `push_lease` (30350)

Two-channel error type `AcceptError::{Validation(String), Internal(String)}` (`push_lease.rs:457-461`), mapped by `map_push_accept_error` (`ingest.rs:187-195`): `Validation` → `invalid: {reason}` (400/OK-false), `Internal` → `IngestError::Internal` (500). `From<String>` defaults to `Validation` (`:463-467`).

Validation strings: `push not supported` (`:481`), `wrong event kind` (`:88`), `content too long` (`:91`), `empty public tag` (`:99`), `{name} tag must have exactly one value` (`:101`), `duplicate {name} tag` (`:114`), `unexpected public tag: {name}` (`:116`), `missing d tag` (`:129`), `invalid d tag length` (`:131`), `missing expiration tag` (`:133`), `expiration must be integer Unix seconds` (`:136`), `lease already expired` (`:139`), `lease ttl too long` (`:142`), `missing exec tag` (`:144`), `empty exec tag` (`:146`), `unknown executor key` (`:485`), `invalid encrypted content` (`:491`), `plaintext too long` (`:151`), `lease plaintext must be an object` (`:158`), `missing active` / `active must be a boolean` (`:161`, `:163`), `missing {key}` (`:186`), `unknown field: {key}` (`:190`), `invalid lease schema: {e}` (`:192`), `unsupported version` (`:196`), `generation must be a positive safe integer` (`:199`), `origin mismatch` (`:201`), `inactive lease must use minimal schema` (`:215`), `missing app_profile`/`transport`/`endpoint`/`subscriptions` (`:220-223`), `app profile not supported` (`:228`), `transport mismatch` (`:231`), `invalid endpoint length` (`:234`), `invalid string length` (`:378`), `subscription quota exceeded` (`:240`), `class not supported` (`:246`), `ignore quota exceeded` (`:250`), `p_tags_max must be positive` (`:257`), `filter member not permitted: {key}` (`:266`), `kind must be an integer` (`:274`), `kind not push-eligible` (`:279`), `class not permitted for kind` (`:282`), `lease filter not narrowed` (`:294`), `p-tag must be self` (`:302`), `invalid h tag` (`:308`, `:310`), `non-exact match value for {label}` (`:371`), `invalid {key} count` (`:334`, `:352`), `{key} must be an array` (`:331`, `:348`), `{key} values must be strings` (`:357`), `missing {key}` (`:325`), `internal optional-array misuse` (`:327`), `invalid relay URL` / `invalid tenant host` (`:591`, `:594`), `invalid generation` (`:558`), `invalid subscriptions` (`:550`), `duplicate object key: {key}` (`:441`), `non-finite number` (`:419`).

Internal string: exactly one — `lease persistence failed` (`:572`), which deliberately swallows the underlying `DbError`.

Outcome strings produced by **ingest**, not the handler (`ingest.rs:2160-2186`): `invalid: stale replacement`, `invalid: stale generation`, `invalid: endpoint already leased`, `invalid: lease quota exceeded`, `invalid: source event collision`, `invalid: lease constraint violation`.

##### 2.7 `community_provisioning` (HTTP)

| String | HTTP status mapped at `api/operator.rs` | Site |
|---|---|---|
| `actor not authorized: not a relay operator` | 403 (`operator.rs:180`) | `:265` |
| `community already exists` | 409 (`operator.rs:183`) | `:292` |
| `limit_reached: owner already owns the maximum number of communities` | 409 | `:295-298` |
| `initial_owner_pubkey is required when create_only is true` | 400 (`operator.rs:194`) | `:281-283` |
| `invalid initial_owner_pubkey: expected 64-char hex pubkey` | 400 | `:275-277` |
| `host is empty` / `host too long: {n} bytes (max 255)` | 400 | `:85`, `:88-90`, `:183`, `:185-188` |
| `host is not normalized: expected {…}` | 400 | `:93-96` |
| `host contains invalid characters` | 400 | `:107`, `:191` |
| `host must be a bare authority (no scheme, path, query, or userinfo)` | 400 | `:114-116`, `:195-197` |
| `host is not a valid authority` | 400 | `:124`, `:127` |
| `host is not a canonical authority: expected {…}` | 400 | `:141-143` |
| `domain name too long` / `domain contains an empty label` / `domain label too long` / `domain label contains invalid characters` | 400 | `:152`, `:156`, `:159`, `:168` |
| `failed to create community: {e}` | 500 (`operator.rs:186-192`) | `:288`, `:325` |
| `community provisioned but owner bootstrap failed: {e}` | 500 | `:332` |

---

#### 3. Public Rust items and their callers

##### 3.1 `handlers/moderation_commands.rs`
| Item | Visibility | Production callers |
|---|---|---|
| `handle_moderation_command` | `pub` | 1 — `ingest.rs:1580` |

Everything else is private (`handle_ban/unban/timeout/untimeout/resolve`, `resolution_audit_action`, `insert_audit`, `authz_denial`, `invalid`, `error`, `extract_*`, `ensure_actor_not_banned`).

##### 3.2 `handlers/moderation_notices.rs`
| Item | Visibility | Production callers |
|---|---|---|
| `ModerationNotice` (enum) | `pub` | `moderation_commands.rs` only |
| `ModerationNotice::ReportResolved` | — | `moderation_commands.rs:485` |
| `ModerationNotice::Restriction` | — | `moderation_commands.rs:208`, `:313` |
| **`ModerationNotice::ContentActioned`** | — | **ZERO production constructors** (only `moderation_notices.rs:390` in tests) |
| `send_moderation_notice` | `pub` | 3 — `moderation_commands.rs:204`, `:309`, `:481` |

##### 3.3 `handlers/moderation_authz.rs`
| Item | Visibility | Production callers |
|---|---|---|
| `authorize_moderation_action` | `pub` | 6 — `moderation_commands.rs:156/235/274/338/399`, `api/bridge.rs:2037` |
| `ModerationAction` | `pub` | 6 of 8 variants used |
| ├ `Ban`, `Unban`, `Timeout`, `Untimeout`, `ResolveReport` | — | `moderation_commands.rs:162/241/280/344/405` |
| ├ `ViewQueue` | — | `api/bridge.rs:2043` |
| ├ **`DeleteMessage`** | — | **ZERO** — only `moderation_authz.rs:123/175/190/282/298` (own match arms + tests) |
| └ **`Kick`** | — | **ZERO** — same |
| `ModerationTarget<'a>` | `pub` | all 3 variants constructed |
| `ModerationAuthority` | `pub` | returned by every call, **discarded by every caller** |

**Consequence:** because `DeleteMessage`/`Kick` are never requested, and all 6 call sites pass `channel_id: None`, the `channel_role` lookup at `moderation_authz.rs:120-131` always evaluates to `None`, and the `ModerationAuthority::ChannelRole` arm (`:174-178`) is **dead in production**. The whole channel-local-authority feature is exercised by unit tests only (`:277-310`).

##### 3.4 `handlers/relay_admin.rs`
| Item | Visibility | Production callers |
|---|---|---|
| `handle_relay_admin_event` | `pub` | 1 — `ingest.rs:1809` |

##### 3.5 `handlers/community_provisioning.rs`
| Item | Visibility | Production callers |
|---|---|---|
| `ProvisionCommunityRequest` | `pub` | `api/operator.rs:164` |
| `ProvisionCommunityResponse` | `pub` | `api/operator.rs:174` (serialized to JSON) |
| `provision_community` | `pub` | 1 — `api/operator.rs:171` |
| `validate_pubkey_hex` | `pub(crate)` | 5 — `api/operator.rs:225/280/318/383/391` |
| `normalize_candidate_host` | `pub(crate)` | 3 — `api/operator.rs:216/278/484` |

HTTP route: `POST /operator/communities` (`router.rs:76`).

##### 3.6 `handlers/push_lease.rs`
| Item | Visibility | Production callers outside this file |
|---|---|---|
| `accept` | `pub` | 1 — `ingest.rs:2156` |
| `AcceptError` | `pub` | `ingest.rs:187-194` |
| `KIND_PUSH_LEASE` | `pub` | `ingest.rs:204/450/2156`, `side_effects.rs:2004/2130` |
| `PUSH_KINDS` | `pub(crate)` | `nip11.rs:208`, `:353` |
| `URGENT_KINDS` | `pub(crate)` | `nip11.rs:209` |
| `Subscription` | `pub` | `push_runtime.rs:13`, `:250` |
| `Suppress` | `pub` | reachable only through `Subscription.suppress` |
| **`validate_envelope`** | `pub` | **ZERO external** |
| **`parse_plaintext`** | `pub` | **ZERO external** |
| **`validate_plaintext`** | `pub` | **ZERO external** |
| **`LeaseEnvelope`** | `pub` | **ZERO external** |
| **`LeasePlaintext`** | `pub` | **ZERO external** |
| **`LeaseLimits<'a>`** | `pub` | **ZERO external** |
| **`AppProfile<'a>`** | `pub` | **ZERO external** (the `AppProfile` hits in `buzz-push-gateway` are a *different* type, `buzz-push-gateway/src/model.rs:14`) |
| **`MAX_SAFE_JSON_INTEGER`** | `pub` | **ZERO external** — used only at `push_lease.rs:198` |

8 public items with no consumer outside this module. `validate_envelope`/`parse_plaintext`/`validate_plaintext` are documented as a public validation API (`push_lease.rs:1-6`) that nothing calls except `accept` and tests.

##### 3.7 `handlers/identity_archive.rs`
| Item | Visibility | Production callers |
|---|---|---|
| `handle_identity_archive_event` | `pub` | 1 — `ingest.rs:1916` |

##### 3.8 `handlers/report.rs`
| Item | Visibility | Production callers |
|---|---|---|
| `handle_report_event` | `pub` | 1 — `ingest.rs:1562` |
| **`REPORT_TYPES`** | `pub` | **ZERO external** — only `report.rs:202` |

##### 3.9 `handlers/product_feedback.rs`
| Item | Visibility | Production callers |
|---|---|---|
| `handle` | `pub` | 1 — `ingest.rs:1541` |

##### 3.10 `push_runtime.rs`
| Item | Visibility | Production callers |
|---|---|---|
| `run_matcher` | `pub` | 1 — `main.rs:687` |
| `run_delivery_worker` | `pub` | 1 — `main.rs:688-690` |

Both spawned only when `state.config.push_gateway_delivery_url.is_some()` (`main.rs:686`) — which is the **default**, because an unset `BUZZ_PUSH_GATEWAY_DELIVERY_URL` falls back to the hard-coded `https://push.buzz.xyz/v1/deliveries/apns` (`config.rs:339`, `:752-758`). Only an explicitly-empty value disables it (`config.rs:753`).

##### 3.11 `storage_sweep.rs`
| Item | Visibility | Production callers |
|---|---|---|
| `StorageSweepConfig` | `pub` | `main.rs:1447`, `:1453` |
| `StorageSweepConfig::from_env` | `pub` | 1 — `main.rs:1453` (behind a function-local `OnceLock`) |
| `StorageSweepState` | `pub` | `state.rs:561`, `:764` |
| `maybe_spawn_sweep` | `pub` | 1 — `main.rs:1460-1471` |
| `emit_storage_metrics` | `pub` | 1 — `main.rs:1474-1477` |
| `StorageEmittedKey` | `pub(crate)` | **used only inside `storage_sweep.rs`** — could be private |

Both entry points are called only from the leader-only branch of the usage tick (`main.rs:1423-1430`).

##### 3.12 `workflow_sink.rs`
| Item | Visibility | Production callers |
|---|---|---|
| `RelayActionSink` | `pub` | `main.rs:594` |
| `RelayActionSink::new` | `pub` | 1 — `main.rs:594` |
| `impl ActionSink for RelayActionSink::send_message` | trait impl | `buzz-workflow/src/executor.rs:567-569` |
| `resolve_mention_pubkeys` | private | `workflow_sink.rs:291` |

---

#### 4. Workflow-action surface actually implemented by `workflow_sink`

The `ActionSink` trait declares **exactly one** method (`buzz-workflow/src/action_sink.rs:44-64`): `send_message`. `RelayActionSink` implements it (`workflow_sink.rs:172-179`) and nothing else.

Against the 7 workflow action types documented in `ARCHITECTURE.md:533-542`:

| Workflow action | Path | Reaches `workflow_sink`? | Working end-to-end? |
|---|---|---|---|
| `send_message` | `executor.rs:566-569` → `ActionSink::send_message` | **yes** | **yes** |
| `send_dm` | `executor.rs:575-579` | no | **no** — `Err(WorkflowError::NotImplemented("SendDm"))` at `executor.rs:578` |
| `set_channel_topic` | `executor.rs:580-584` | no | **no** — `Err(NotImplemented("SetChannelTopic"))` at `executor.rs:583` |
| `add_reaction` | `executor.rs:585-607` → `add_reaction_impl` HTTP POST | no | **no** — POSTs to `{BUZZ_RELAY_BASE_URL}/api/messages/{id}/reactions` (`executor.rs:886-888`); no such route is registered in `router.rs` (verified: zero `reactions` and zero `api/messages` matches). Returns `WorkflowError::WebhookError("AddReaction: relay returned 404 …")` |
| `call_webhook` | `executor.rs:608+` (own HTTP client) | no | yes (outside this module) |
| `request_approval` | `executor.rs` → `StepResult::Suspended` | no | **no** — ARCHITECTURE.md:826 (WF-08) |
| `delay` | `executor.rs` | no | yes (outside this module) |

**Verified from the sink side:** `workflow_sink` implements 1 of 7 workflow actions. ARCHITECTURE.md items 5 and 6 (`:826-827`) are accurate as written but **understate** the gap: `add_reaction` is a third broken action not listed there, and the relay-side sink surface is a single method, so `send_dm`/`set_channel_topic` cannot be fixed without widening the trait.

---

#### 5. HTTP surface touched by this module

| Route | Method | Handler | Auth |
|---|---|---|---|
| `/operator/communities` | POST | `api::operator::provision_community` → `community_provisioning::provision_community` | NIP-98 against `RELAY_OPERATOR_API_ORIGIN` + replay guard (`operator.rs:104-135`) + `RELAY_OPERATOR_PUBKEYS` allowlist (`community_provisioning.rs:258-266`) |
| `/moderation/reports` | GET | `api::bridge::moderation_reports` | NIP-98 + `ModerationAction::ViewQueue` (`bridge.rs:2036-2049`) |
| `/moderation/audit` | GET | `api::bridge::moderation_audit` | same |

Read cap: `MODERATION_READ_LIMIT = 500` (`bridge.rs:2053`).

No handler in this module registers a route itself; all three are wired in `router.rs:76`, `:113`, `:114`.

---

#### 6. Outbound HTTP surface

One only: the push gateway `POST` (`push_runtime.rs:517-528`).

Request body (`DeliveryRequest`, `push_runtime.rs:31-37`): `{ "v": 1, "endpoint_grant": <string>, "request_id": <uuid>, "expires_at": <i64> }`. `request_id` is the durable wake row id and is **deliberately stable across retries** (`push_runtime.rs:488-490`), pinned by an HTTP-level test (`push_runtime.rs:626-655`).

Headers: `Authorization: Nostr <base64(kind-27235 event)>` (`push_runtime.rs:551-565`) and `Content-Type: application/json`.

Response contract (`DeliveryResponse`, `push_runtime.rs:39-51`), an internally-tagged enum on `status`:

| HTTP | Body `status` | Relay action | Site |
|---|---|---|---|
| 2xx | `accepted` | `complete_push_wake` | `:434-441` |
| 2xx | anything else / unparseable | `fail_push_wake` | `:442-447` |
| 410 | `invalid_endpoint { generation, invalid_at }` | `disable_push_endpoint` **iff** generation matches, then `fail` | `:448-473` |
| 503 | `retry { retry_after_seconds }` | `retry_or_fail`, delay clamped to `>0` else 2 s | `:474-484` |
| 429 | (ignored) | `retry_or_fail(2)` | `:485-487` |
| 404 **and** `attempt > 1` | (ignored) | `complete_push_wake` — treats a replayed timed-out attempt as delivered | `:491-497` |
| timeout / connect error | — | `retry_or_fail(2)` | `:498` |
| anything else | — | `fail_push_wake` | `:499-503` |
