## Module: buzz-relay — moderation, admin & background workers (`crates/buzz-relay/src`)
### Aspect: Debt

Scope: 12 files, 6,720 LOC, 110 tests, 1 `#[ignore]`d test, 0 `unsafe`, 0 production `unwrap()`, 7 production `.expect()`, 0 `TODO`/`FIXME`/`HACK` markers.

Severity: **CRITICAL** = security or data-integrity hole reachable today · **HIGH** = correctness/operability gap with real blast radius · **MEDIUM** = latent risk or meaningful maintenance drag · **LOW** = hygiene · **INFO** = observation.

---

#### Prioritized findings

##### D-01 · HIGH · Relay-admin commands (9030–9033) never re-check the durable ban

`ingest.rs:1613` exempts `is_relay_admin_kind` from the ban/timeout write gate on the stated grounds that relay-admin "retains its separate authorization policy" (`ingest.rs:1594-1596`). That policy is `relay_admin.rs:133-142`, which reads only `relay_members.role`. Grep for `restriction` / `banned` / `moderation_restriction_state` in `relay_admin.rs`: **zero hits**.

The moderation handler *does* re-check, with rationale that HTTP NIP-98 requests and already-authenticated sockets bypass the fresh NIP-42 challenge (`moderation_commands.rs:97-101`). That reasoning applies verbatim to 9030–9033 and is not implemented. A banned owner/admin with a surviving socket — or over NIP-98, which never touches the auth seam — can add members, remove members, change roles, and rewrite the workspace icon.

Fix: mirror `moderation_commands.rs:103-108` at the top of `handle_relay_admin_event`.

---

##### D-02 · HIGH · 12 of 14 privileged kinds produce no hash-chain audit entry

`buzz-audit` declares 11 actions (`buzz-audit/src/action.rs:5-29`); production emits exactly two — `EventCreated` (`handlers/event.rs:560`) and `MediaUploaded` (`api/media.rs:428`). None of the 12 assigned files references `buzz_audit`. Confirmed.

| Unaudited in the hash chain | Any durable trail? |
|---|---|
| 9040 ban, 9041 unban, 9042 timeout, 9043 untimeout, 9044 resolve | `moderation_actions` only — a plain, non-hash-chained table |
| **9030 add member, 9031 remove member, 9032 change role, 9033 set icon** | **none** — `tracing::info!` only (`relay_admin.rs:164`, `:203-209`, `:268-272`, `:327-332`) |
| **operator community provisioning / owner rotation** | **none** — `tracing::info!` only (`community_provisioning.rs:302-308`, `:336-343`) |
| 1984 report, 42000 feedback | own sidecar tables |
| 30350 push lease | event row, but ingest returns at `:2199` before the dispatch that audits |

Only 9035/9036 reach the chain, and only because they deliberately fall through to storage (`ingest.rs:1909-1912`).

The tamper-evident audit surface does not cover the relay's own privilege model. Membership and ownership changes — "who made X an admin", "who rotated community Y's owner" — exist only in ephemeral process logs.

Also: ARCHITECTURE.md:497 says "**10 audit actions**" and omits `MediaUploaded`; the enum has 11.

---

##### D-03 · MEDIUM · Community moderation has no self-action prevention

`moderation_commands.rs` never compares actor to target. An owner issuing 9040 against their own pubkey passes `decide_authority` (`moderation_authz.rs:149-150`), commits the ban (`:169`), then closes their own sockets cluster-wide (`:195-200`). Self-unban is blocked by the handler's own ban re-check (`:103-108`), so recovery needs a second owner/admin or direct DB access.

`relay_admin.rs` implements exactly this check for its own operations — `cannot remove yourself` (`:231-233`), `cannot change your own role` (`:290-292`). The moderation path does not. An admin is protected only incidentally, because the peer guard trips on their own `admin` role (`moderation_authz.rs:165-167`).

---

##### D-04 · MEDIUM · The "reject channel-scoped API tokens" contract for 9040–9044 is not enforced

Pinned at `moderation_commands.rs:50` as part of the routing contract. Ingest rejects channel-scoped tokens only for relay-admin kinds (`ingest.rs:1512-1516`) and leave requests (`:1520-1523`); the generic global-event token gate at `ingest.rs:1721-1724` is **unreachable** for 9040–9044 because the moderation dispatch returns at `:1582-1586`. Combined with `Scope::MessagesWrite` (`ingest.rs:216`), a legacy channel-scoped WS API token held by a community admin can issue a community-wide ban.

---

##### D-05 · MEDIUM · `push_runtime` has 2 tests for 656 LOC and zero metrics

Coverage: `gift_wrap_match_requires_self_p_filter_and_recipient` (`:580-620`) and `gateway_retries_send_the_same_request_id_over_http` (`:626-655`). Untested: `deliver_one`'s 10-branch response state machine (`:349-505`), `retry_or_fail`'s exponential backoff (`:531-550`), `process_match_batch`'s completed/retry/pending partitioning (`:125-214`), `load_match_context` (`:98-123`), and `match_job`'s class/suppress/ignore logic (`:216-290`).

Observability is equally thin: **zero** `metrics::` calls in the file. No wake-enqueued counter, no delivery-outcome counter by HTTP status, no gateway-latency histogram, no matcher-backlog gauge. Push delivery health is observable only through `warn!`/`error!` lines. Compare `storage_sweep.rs`, which emits 18 gauges for a strictly less critical subsystem.

---

##### D-06 · MEDIUM · Every DB call in the push delivery path discards its result

8 sites use `let _ = state.db.…` : `push_runtime.rs:361-364`, `:380-383`, `:410-413`, `:436-440`, `:442-446`, `:454-462`, `:469-472`, `:492-495`, `:533-536`, `:539-546`. A failed `complete_push_wake` leaves the row claimed; recovery depends on the 30 s claim lease expiring and the wake being re-claimed. That is safe (the stable request id makes redelivery idempotent) but completely invisible — combined with D-05 there is no way to detect a persistently failing completion path.

---

##### D-07 · MEDIUM · The delivery worker dies permanently if the HTTP client fails to build

`reqwest::Client::builder().timeout(…).build().expect("push HTTP client")` at `push_runtime.rs:313-316`, inside a `tokio::spawn`ed loop (`main.rs:688-690`). A panic here kills the worker while the matcher keeps enqueuing wakes — an outbox with no consumer, no restart, and (per D-05) no metric to notice.

---

##### D-08 · MEDIUM · The push wire contract is duplicated, not shared

`DeliveryRequest` / `DeliveryResponse` (`push_runtime.rs:31-51`) and the gateway's own model (`buzz-push-gateway/src/model.rs`, `grant.rs:162`, `http.rs:151`) are independent definitions with no shared crate and no cross-artifact test. Contrast the excellent `include_str!` migration test that pins `PUSH_KINDS` to the SQL trigger (`push_lease.rs:696-710`) — that pattern is applied once and not here.

---

##### D-09 · MEDIUM · Push lease DB errors are undiagnosable

`push_lease.rs:571-572`: `.map_err(|_| AcceptError::Internal("lease persistence failed"))`. The underlying `DbError` — constraint name, deadlock, connection failure — is discarded entirely and never logged (the file has zero `tracing` calls). Operators see a 500 with a fixed string and nothing else.

---

##### D-10 · MEDIUM · `LeasePlaintext` models mandatory fields as `Option`, forcing 5 production `.expect()`

`push_lease.rs:32-41` uses `Option<T>` for `app_profile`/`transport`/`endpoint`/`subscriptions`, which are mandatory when `active == true`. `accept` then re-unwraps them 5× with justification comments (`:534`, `:539`, `:543`, `:548`, `:552`). Five of the module's seven production `.expect()` calls come from this one modelling choice. An `enum LeasePlaintext { Active { … }, Inactive { … } }` would make them unrepresentable.

Note the workaround already in place: because `serde(deny_unknown_fields)` cannot express a conditional required-set, the code adds a *second*, hand-rolled key-presence check (`:159-190`) on top of serde. Two validation layers for one schema.

---

##### D-11 · MEDIUM · `push_runtime` runs on every pod with no leader election

`main.rs:686-692` spawns both loops unconditionally when the gateway URL is set (which is the default). `storage_sweep` is protected by a real Postgres advisory lock (`main.rs:1414-1421`); `push_runtime` is not. Correctness rests entirely on DB claim/fence tokens, which is sound, but the cost is N× claim-scan load at N pods. The delivery worker additionally iterates **every** community from `usage_community_hosts()` on every sweep (`push_runtime.rs:320-338`) with no bounding — that scan grows linearly with tenant count per pod per 500 ms at minimum.

---

##### D-12 · MEDIUM · Dead capability surface in `moderation_authz`

| Item | Production consumers |
|---|---|
| `ModerationAction::DeleteMessage` (`:32`) | **0** |
| `ModerationAction::Kick` (`:34`) | **0** |
| `ModerationAuthority::ChannelRole` (`:68`) | unreachable — all 6 call sites pass `channel_id: None` |
| `get_member_role` lookup (`:120-131`) | unreachable |
| `ModerationAuthority` return value | computed on every call, discarded by every caller |

The module doc says the seam is "the bridge `validate_admin_event` is missing today" (`:73-75`). The 9005-delete and 9001-kick paths still use `side_effects::validate_admin_event` (`ingest.rs:1903-1907`). The bridge was built and never connected — ~25% of the file (the channel-role branch plus its 2 tests) is production-dead.

The doc also claims the matched authority is "recorded in the audit row" (`:61`). `insert_audit` takes no authority parameter (`moderation_commands.rs:518-527`). **Documentation delta.**

---

##### D-13 · MEDIUM · 5 of 9 `moderation_actions` columns are always NULL

The single production writer hard-codes `channel_id: None`, `reason_code: None`, `private_reason: None`, `matched_principal: None` (`moderation_commands.rs:536-540`). `matched_principal` is documented as recording the NIP-OA principal an enforcement check matched (`buzz-db/src/moderation.rs:139`) and is surfaced verbatim by the mod-audit read API (`api/bridge.rs:2166`) — so that API field is **always `null`**.

Additionally, `MODERATION_ACTION_CHECK_VOCAB` includes `"delete_message"` and `"kick"` (`buzz-db/src/moderation.rs:105-106`) with zero production writers, and `resolution_audit_action` has an unreachable `"resolve:unknown"` fallback (`moderation_commands.rs:511`) that is **not** in the CHECK vocabulary — if reached it would raise a constraint violation.

---

##### D-14 · MEDIUM · No atomicity across the moderation write set

`handle_ban` performs restriction read (`:105`), ban write (`:169`), audit write (`:180`), disconnect (`:195`), notice (`:204`) with **no enclosing transaction**. An audit-write failure returns `error: failed to write audit row: …` (`:544`) for a ban that is already durable — the client sees failure for an action that took effect. Same shape for timeout (`:287` → `:298`).

`handle_resolve` inverts the order deliberately (audit first at `:453`, resolve at `:461`) and documents the residual race as tolerated (`:419-425`) — but that also means a failed resolve leaves an orphan audit row.

---

##### D-15 · MEDIUM · Unbounded report flooding, with no note size cap

Any authenticated principal may submit arbitrarily many distinct signed 1984 events. Idempotency is per event id (`buzz-db/src/moderation.rs:39`), so fresh signatures always land. There is no per-pubkey rate limit (ARCHITECTURE.md:821 confirms the only repo-wide `RateLimiter` impl is a test stub), no per-target cap, no `(reporter, target)` dedup, and the note is `event.content` stored **uncapped** (`report.rs:222-228`).

Contrast `product_feedback.rs:12-13`, which caps body at 32 KiB and tags at 64 KiB. Same threat model, opposite treatment. The mod queue read caps at 500 rows (`api/bridge.rs:2053`), so a flood degrades moderator usability.

---

##### D-16 · MEDIUM · Legacy provisioning mode is a community-takeover capability

Without `create_only`, an operator-signed request runs `bootstrap_owner` on an **existing** community, demoting the previous owner to admin (`community_provisioning.rs:321-334`). Documented and deliberate (`:236-247`), but the mitigation — "clients provisioning on behalf of end users must use create_only" (`:317-320`) — is prose, not code. Combined with D-02 (no audit trail) an owner rotation is both possible and untraceable.

---

##### D-17 · MEDIUM · `add_reaction` workflow action targets an unregistered route

`buzz-workflow/src/executor.rs:886-888` POSTs to `{BUZZ_RELAY_BASE_URL}/api/messages/{message_id}/reactions`. Verified: `router.rs` registers **zero** `reactions` routes and zero `/api/messages` routes. Every attempt returns `WorkflowError::WebhookError("AddReaction: relay returned 404 …")` (`:903-908`). Without the `reqwest` feature it silently reports success-with-skip (`:597-606`).

ARCHITECTURE.md:541 lists `add_reaction` as a working action; §9 (`:826-827`) lists only `send_dm`/`set_channel_topic` and approval gates. **`add_reaction` is an undocumented third broken action.**

---

##### D-18 · MEDIUM · `ActionSink` has one method, so 3 of 7 workflow actions are structurally unimplementable

`buzz-workflow/src/action_sink.rs:44-64` declares exactly `send_message`; `RelayActionSink` implements exactly that (`workflow_sink.rs:172-179`). `send_dm` and `set_channel_topic` return `NotImplemented` (`executor.rs:578`, `:583`) and cannot be fixed without widening the trait. `add_reaction` bypasses the sink entirely via HTTP (D-17) rather than using it.

All 6 `ActionSinkError` variants also collapse into a single `WorkflowError::WebhookError` (`action_sink.rs:31-34`), so channel-not-found, archived-channel, access-denied, and a DB outage are indistinguishable in run output.

---

##### D-19 · MEDIUM · The `identity_archive` integration test passes green without Postgres

`identity_archive.rs:515-527` is a `#[tokio::test]` with three silent `return` bailouts: `test_pool()` → `None` (`:517-519`), a probe `SELECT` failing (`:520-526`), `test_state()` → `None` (`:527-529`). `test_state` itself is `Option`-returning with `.ok()?` chains (`:442-472`).

This is the module's **only** coverage of the live-kind:0 revocation rule (BR-MOD-87) — arguably its most security-relevant behaviour — and it silently no-ops in any environment without Postgres. `workflow_sink.rs:613` handles the same situation correctly with `#[ignore = "requires Postgres"]`.

---

##### D-20 · MEDIUM · `BUZZ_PUSH_GATEWAY_DELIVERY_URL` defaults to a hard-coded third-party host

Unset ⇒ `Some("https://push.buzz.xyz/v1/deliveries/apns")` (`config.rs:339`, `:755-758`), which enables lease acceptance (`push_lease.rs:480-482`), spawns both workers (`main.rs:686-692`), advertises push in NIP-11 (`nip11.rs:208`), and attempts outbound HTTPS carrying the device token. Disabling requires an explicitly **empty** value (`config.rs:753`) — unsetting does the opposite of the intuitive thing. A self-hosted relay that never configures push still does all of the above.

---

##### D-21 · LOW · Push endpoint ownership is first-claim-wins with no proof of possession

The relay accepts any 1..=4096-byte `endpoint` string, hashes it, and lets a DB unique constraint arbitrate (`push_lease.rs:533-535`, `:563-570`, outcome `EndpointAlreadyLeased` at `ingest.rs:2170-2174`). Registering a token you learned denies push to the real device. The wake body is content-free (`push_runtime.rs:31-37`), so the leak is timing/existence only, scoped by the lease author's own read authorization.

Related: `ActiveLease.endpoint_grant` is documented as "opaque endpoint grant issued by the stateless gateway" (`buzz-db/src/push.rs:94`) but stores the client-supplied token verbatim (`push_lease.rs:544`, `:555`). Misleading name for a credential-adjacent value persisted in plaintext.

---

##### D-22 · LOW · Redirects are not disabled on the push gateway client

`push_runtime.rs:313-316` sets only `.timeout(…)`; `redirect::Policy` defaults to following up to 10 hops. A compromised or misconfigured gateway host could redirect the POST — including the NIP-98 `Authorization` header and the device token — elsewhere. `buzz-workflow`'s `call_webhook` explicitly disables redirects (ARCHITECTURE.md:539); this path does not.

---

##### D-23 · LOW · `moderation_notices` idempotency is not concurrency-safe

`notice_already_sent` is query-then-insert (`moderation_notices.rs:227-252`); the comment states two simultaneous deliveries for the same source can both miss the pre-query and names hard per-source serialization as a follow-up (`:132-138`). The scan limit is hard-coded to 1000 with the reasoning that 1000 notices to one user is a practical ceiling (`:222-226`) — beyond it, crash-retry can duplicate.

---

##### D-24 · LOW · Notice-DM failures are logged at `info!`, not `warn!`

`moderation_commands.rs:217`, `:319`, `:493`. A user who was banned and never told is an `info`; a failed NIP-43 announcement is a `warn` (`relay_admin.rs:215`). Inverted severity for the more user-visible failure.

---

##### D-25 · LOW · Unban and untimeout send no notice at all

9041 (`moderation_commands.rs:227-261`) and 9043 (`:330-364`) have no `send_moderation_notice` call. A user learns they were restricted but never that the restriction was lifted. `ModerationNotice::Restriction`'s doc also promises "with expiry rendered into the message" (`moderation_notices.rs:66`) — the body renders only the verb and reason (`:296-305`), never the expiry.

---

##### D-26 · LOW · `ModerationNotice::ContentActioned` has zero production constructors

Declared (`moderation_notices.rs:52-57`), rendered (`:288-292`), tested (`:388-397`), documented as the "actioned-author" notice for the delete path (`:25-26`) — and never constructed outside tests. The 9005 delete path does not call `send_moderation_notice`.

---

##### D-27 · LOW · `escalated` report status has no producer

9044 always stores `resolved` or `dismissed` (`moderation_commands.rs:380-385`), yet `ReportRecord.status` advertises `escalated` (`buzz-db/src/moderation.rs:71`) and `ModerationNotice::body` has an unreachable `"escalated"` arm (`moderation_notices.rs:281`). An `action=escalate` resolution stores `status=resolved`.

Related: the module doc's summary table implies the relay fans `delete`/`kick`/`ban` resolutions out through 9005/9001/9040 (`moderation_commands.rs:20`). `handle_resolve` performs no fan-out (`:453-499`); the body text does say the *client* composes the paired command (`:48-50`), but nothing guarantees the enforcement ever happens.

---

##### D-28 · LOW · The `urgent` push class is dead

`URGENT_KINDS = &[]` (`push_lease.rs:16`) and `"urgent"` absent from `supported_classes` (`:509`) means `class not supported` (`:246`) fires before the urgent-kind confinement check at `:281-283` can run. `class_rank` still ranks it in both copies (`:582`, `push_runtime.rs:574`); NIP-11 advertises `urgent_kinds: []` (`nip11.rs:209`).

---

##### D-29 · LOW · 8 public items in `push_lease.rs` have no external consumer

`validate_envelope` (`:83`), `parse_plaintext` (`:149`), `validate_plaintext` (`:194`), `LeaseEnvelope` (`:24`), `LeasePlaintext` (`:32`), `LeaseLimits` (`:65`), `AppProfile` (`:60`), `MAX_SAFE_JSON_INTEGER` (`:21`). The module presents itself as a reusable validation library (`:1-6`) but only `accept` is called. Also dead-public: `REPORT_TYPES` (`report.rs:29`); over-scoped: `StorageEmittedKey` is `pub(crate)` but used only inside its own file (`storage_sweep.rs:121`).

---

##### D-30 · LOW · Two competing definitions of `KIND_PUSH_LEASE`

`buzz-core/src/kind.rs:109` (`30350`) and `handlers/push_lease.rs:19` (`30_350`). Ingest imports the `push_lease` copy (`ingest.rs:204`, `:450`, `:2156`) and so does `side_effects.rs:2004`, `:2130`; `req.rs:1689-1710` imports the `buzz-core` one. AGENTS.md requires all kind integers live in `buzz-core/src/kind.rs`. Two sources of truth for one wire constant.

---

##### D-31 · LOW · Three competing error-prefix strategies across six handlers

| Strategy | Handlers | Effect |
|---|---|---|
| self-prefixing | `moderation_commands` (helpers `:548-558`), `report`, `product_feedback` (inline literals) | correct |
| unprefixed, ingest wraps `invalid: {e}` | `relay_admin` (`ingest.rs:1811`), `identity_archive` (`ingest.rs:1917`) | authorization failures surface as `invalid: actor not authorized: …` — semantically wrong |
| typed error enum | `push_lease` → `AcceptError` (`ingest.rs:187-195`) | correct 400/500 split |

Only `moderation_commands` has a test pinning its prefixes (`:669-680`). `report.rs`/`product_feedback.rs` inline the same prefixes at 20+ sites with no helper.

---

##### D-32 · LOW · Duplicated helpers across the handler set

| Duplicate | Copies |
|---|---|
| `extract_tag_value` (identical body) | 3 — `moderation_commands.rs:608`, `relay_admin.rs:49`, `identity_archive.rs:189` |
| p-tag extraction with 3 different contracts | `moderation_commands.rs:561` (bytes, first-valid), `relay_admin.rs:33` (hex, first-valid), `identity_archive.rs:170` (hex, **exactly-one**) |
| 64-hex validation | 4 inline + 2 typed (`report.rs:211`, `push_lease.rs:365` — the latter alone rejects uppercase) |
| ±120 s freshness (3 sites) + `ALLOWED_SKEW` | `moderation_commands.rs:81`, `relay_admin.rs:125`, `identity_archive.rs:147`, `push_lease.rs:476` |
| Freshness error string, verbatim | `moderation_commands.rs:117-120`, `relay_admin.rs:126-129`, `identity_archive.rs:148-151` |
| `blocked: you are banned from this community`, verbatim | `moderation_commands.rs:139`, `:199`, `ingest.rs:1622`, `handlers/auth.rs:171` |
| `class_rank` | `push_lease.rs:575`, `push_runtime.rs:567` |
| DB-state test-harness construction (~30 LOC each) | `identity_archive.rs:442-472`, `workflow_sink.rs:574-599` |

---

##### D-33 · LOW · Six handler signatures, three argument orders, three naming schemes

`(tenant, state, event)` for `moderation_commands`/`relay_admin`/`identity_archive`; `(tenant, event, state)` for `report`/`product_feedback`; `(tenant, state, event, now)` for `push_lease`. Names: `handle_*_event` ×4, `handle_*_command` ×1, bare `handle` ×1. Only `push_lease` injects a clock, so the other five freshness checks are untestable without wall-clock manipulation.

---

##### D-34 · LOW · Pubkey representation split across the same request

`relay_members` and `archived_identities` key on 64-hex `String` (`relay_members.rs:41`, `archived_identities.rs:34`); `community_bans`, `moderation_actions`, `moderation_reports` use raw `Vec<u8>` (`moderation.rs:85`, `:122`). A single 9040 hex-encodes the actor for `get_relay_member` (`moderation_authz.rs:98`) and passes the same actor as bytes to `insert_moderation_action` (`moderation_commands.rs:532`). Role is an untyped `String` compared against literals at 10+ sites with no enum, and `relay_admin.rs:142` collapses "no member row" to `""`.

---

##### D-35 · LOW · Best-effort side effects can silently desync durable state from the event view

| Path | Failure handling |
|---|---|
| NIP-43 member add/remove/list (5 calls) | `warn!` and continue (`relay_admin.rs:214-220`, `:274-279`, `:334-336`) |
| NIP-IA delta + archival list | `warn!` and continue (`identity_archive.rs:130-136`) |
| Provisioning membership snapshot | `warn!` and continue (`community_provisioning.rs:218-227`) |
| Moderation kind:0 profile | `warn!` and continue (`moderation_notices.rs:152-154`) |
| Workflow `dispatch_persistent_event` | result discarded with `let _ =` (`workflow_sink.rs:351`) |
| Ban disconnect Redis publish | `tokio::spawn`ed, `warn!` on failure (`state.rs:1043-1047`) |

Each is individually reasoned, but the aggregate is that `relay_members` / `archived_identities` / `communities` can diverge from their event-backed views with only a log line, and no reconciliation job for the membership/archive views.

---

##### D-36 · LOW · A missed ban-disconnect publish leaves read access open indefinitely

The ingest gate (`ingest.rs:1613-1650`) covers writes; nothing gates an established subscription's reads. On a pod that missed the Redis command, a banned user's socket keeps receiving fan-out until reconnect. `disconnect_pubkey_clusterwide` returns the local close count (`state.rs:1049`) and the caller discards it (`moderation_commands.rs:195`), so zero-sockets-closed is not even observable.

---

##### D-37 · LOW · An owner timeout does not cascade to that owner's NIP-OA agents

Bans cascade at the auth seam (`handlers/auth.rs:134-155`); the ingest write gate checks the authoring pubkey only (`ingest.rs:1598-1611`). Because timeout has no auth-seam presence, timing out a human leaves every agent they own free to post. Documented as a deliberate Phase-1 asymmetry with the follow-up named as a restriction-state cache (`ingest.rs:1607-1611`).

---

##### D-38 · LOW · Storage-sweep config silently absorbs invalid values and is read once, late

The four `BUZZ_STORAGE_*` vars use `.ok().and_then(parse).unwrap_or(default)` (`storage_sweep.rs:56-73`) — a typo is invisible. Every `Config::from_env` field by contrast hard-fails at boot (`config.rs:747-751`, `:764-771`). They are also read only on the **first leader tick** behind a function-local `OnceLock` (`main.rs:1447-1453`), so a non-leader pod never validates them.

`TIMEOUT_SECS` and `MAX_OBJECTS` have no floor: `TIMEOUT_SECS=0` yields a sweep that times out immediately and respawns every usage tick forever, because `should_spawn` returns `true` unconditionally on `!ok` (`storage_sweep.rs:110`). Only `INTERVAL_SECS` is floored (`.max(60)`, `:59-60`).

`BUZZ_STORAGE_METRICS` uses bespoke `off`-only parsing rather than the crate's `parse_bool` (`config.rs:364-378`), so `false`/`0`/`disabled` all **enable** the feature (pinned by a test that documents the asymmetry, `storage_sweep.rs:379-397`).

---

##### D-39 · LOW · Storage sweep requires `s3:ListBucket` on the read-write media credential, whole-bucket

The sweep shares `MediaStorage` and therefore `BUZZ_S3_ACCESS_KEY`/`BUZZ_S3_SECRET_KEY` (`config.rs:622-625`), calling `list_page` with an **empty prefix** (`buzz-media/src/storage.rs:249-256`). No list-only credential, no per-tenant prefix. Enabling metrics widens the blast radius of a relay compromise to a full cross-tenant object inventory. The kill switch is the only mitigation offered (`storage_sweep.rs:42-45`).

---

##### D-40 · LOW · S3 outage is reported to reporters as "blob not found"

`report.rs:66-70` maps every `get_sidecar` failure — including transient storage errors — to `invalid: report target blob not found`. Documented as a known Phase-1 limitation (`:66-69`), but a MinIO/S3 outage tells reporters their evidence does not exist.

---

##### D-41 · LOW · DB error text reaches clients on multiple paths

`error: database error: {e}` (`moderation_commands.rs:174` + 5), `database error: {e}` (`relay_admin.rs:137` + 6), and `restricted: {e}` wrapping a possible `sqlx` error from the authorization seam (`moderation_commands.rs:549` + `moderation_authz.rs:99`). The HTTP moderation read path does this correctly, normalizing every denial to a fixed 403 (`api/bridge.rs:2044-2049`); the event paths do not. `identity_archive.rs:324` similarly surfaces `buzz-sdk` internal error text.

---

##### D-42 · LOW · Workflow messages can never be threaded

`workflow_sink.rs:325-335` hard-codes `depth: 0`, `parent_event_id: None`, `root_event_id: None`, documented as "Workflow messages are always top-level" (`:322`). A workflow triggered by a message cannot reply in that message's thread. `event_created_at` also silently falls back to `Utc::now()` on an out-of-range timestamp (`:311-314`), where `product_feedback.rs:47-49` rejects the same condition.

---

##### D-43 · LOW · Moderation success logs omit the actor

`moderation_commands.rs:223`, `:258`, `:325`, `:362` log `target` but not `actor`. `relay_admin.rs:203-209` logs both `sender` and `target`. Combined with D-02 (no hash-chain entry), a ban is not attributable from logs alone — only from `moderation_actions`.

---

##### D-44 · INFO · Unused validation fields kept for shape-checking

`ParsedReportTarget::{Event,Blob}.author_pubkey` is parsed and annotated "Validation-shape only" (`report.rs:106-107`, `:112-113`) and discarded with `..` at `:54`, `:65`. Deliberate, documented, but a reader must trace two hops to learn the reported `p` tag is not authority for authorship.

---

##### D-45 · INFO · `storage_sweep.rs` is 1090 LOC with 730 of them tests

Production code is `:1-357` (357 LOC); tests are `:359-1090` (731 LOC), a 2:1 test-to-code ratio — the healthiest in the group and the right model for the rest of it. Note also that `emit_storage_metrics` publishes only gauges (even sweep duration, `:300`), so cross-sweep percentile analysis is impossible.

---

#### Debt summary by file

| File | LOC | Tests | Findings | Weight |
|---|---|---|---|---|
| `push_runtime.rs` | 656 | 2 | D-05, D-06, D-07, D-08, D-11, D-22, D-28, D-30, D-32 | **heaviest** — lowest coverage + zero metrics on the most operationally exposed loop |
| `handlers/relay_admin.rs` | 468 | 15 | D-01, D-02, D-31, D-32, D-33, D-35, D-43 | **highest severity** — D-01 + D-02 |
| `handlers/moderation_commands.rs` | 768 | 10 | D-02, D-03, D-04, D-13, D-14, D-24, D-25, D-27, D-31, D-32, D-33, D-36, D-41, D-43 | most findings, but most are policy gaps, not code defects |
| `handlers/push_lease.rs` | 771 | 10 | D-09, D-10, D-21, D-28, D-29, D-30 | typed-modelling debt |
| `handlers/moderation_authz.rs` | 335 | 7 | D-12, D-41 | ~25% production-dead |
| `handlers/identity_archive.rs` | 580 | 6 | D-19, D-32, D-33, D-35, D-41 | D-19 is the important one |
| `workflow_sink.rs` | 711 | 18 | D-17, D-18, D-35, D-42 | gaps are in `buzz-workflow`, not the sink |
| `handlers/report.rs` | 337 | 6 | D-15, D-31, D-33, D-40, D-44 | D-15 is the important one |
| `handlers/moderation_notices.rs` | 398 | 4 | D-23, D-25, D-26, D-35 | low coverage for a user-facing path |
| `handlers/community_provisioning.rs` | 445 | 13 | D-02, D-16, D-35 | best-validated file; the gap is policy |
| `storage_sweep.rs` | 1090 | 15 | D-38, D-39, D-45 | exemplary engineering; config hygiene only |
| `handlers/product_feedback.rs` | 161 | 4 | D-31 | cleanest file in the group |

#### Verified counts

| Metric | Value |
|---|---|
| Inbound event kinds owned | 14 (1984, 9030–9033, 9035, 9036, 9040–9044, 30350, 42000) |
| Tests (`#[test]` + `#[tokio::test]`) | 110 |
| `#[ignore]`d tests | 1 (`workflow_sink.rs:613`) |
| Tests that silently pass without Postgres | 1 (`identity_archive.rs:515`) + 1 env-gated skip (`storage_sweep.rs:381`) |
| `unsafe` | 0 |
| `unwrap()` outside `#[cfg(test)]` | 0 |
| `.expect()` outside `#[cfg(test)]` | 7 — `push_lease.rs:534/539/543/548/552`, `push_runtime.rs:316/514` |
| `panic!` outside `#[cfg(test)]` | 0 |
| `todo!` / `unimplemented!` | 0 |
| `TODO` / `FIXME` / `HACK` / `XXX` markers | 0 (the 2 `TODO (WF-07)` live in `buzz-workflow/src/executor.rs:577`, `:582`) |
| `#[allow(…)]` attributes | 0 |
| Env vars read directly | 4 (all `BUZZ_STORAGE_*` in `storage_sweep.rs`) |
| `SPROUT_*` vars read | 0 |
| Privileged mutations with no hash-chain entry | 12 of 14 kinds |
| Privileged mutations with no durable audit trail at all | 5 (9030, 9031, 9032, 9033, operator provisioning) |
| Public items with zero external callers | 10 (8 in `push_lease.rs`, `REPORT_TYPES`, `StorageEmittedKey`) |
| Declared-but-unconstructed enum variants | 3 (`DeleteMessage`, `Kick`, `ContentActioned`) |
| Workflow actions implemented by `workflow_sink` | 1 of 7 |
