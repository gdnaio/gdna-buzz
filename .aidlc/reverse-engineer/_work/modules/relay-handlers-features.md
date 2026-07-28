## Module: buzz-relay — moderation, admin & background workers (`crates/buzz-relay/src`)
### Aspect: Features

---

#### 1. Feature inventory

| # | Subsystem | File(s) | Status | Entry point |
|---|---|---|---|---|
| F1 | Community moderation commands (ban/unban/timeout/untimeout/resolve) | `handlers/moderation_commands.rs` (768 LOC) | working | `ingest.rs:1580` |
| F2 | Moderation authorization capability seam | `handlers/moderation_authz.rs` (335) | working, **partly dead** | `moderation_commands.rs` ×5, `api/bridge.rs:2037` |
| F3 | Relay-signed moderation notice DMs | `handlers/moderation_notices.rs` (398) | working, **1 of 3 templates unused** | `moderation_commands.rs` ×3 |
| F4 | NIP-56 report intake | `handlers/report.rs` (337) | working | `ingest.rs:1562` |
| F5 | NIP-43 relay-membership admin + workspace icon | `handlers/relay_admin.rs` (468) | working | `ingest.rs:1809` |
| F6 | Operator community provisioning | `handlers/community_provisioning.rs` (445) | working | `api/operator.rs:171` |
| F7 | NIP-PL push-lease validation + acceptance | `handlers/push_lease.rs` (771) | working, **`urgent` class dead** | `ingest.rs:2156` |
| F8 | NIP-PL durable matcher + APNs delivery worker | `push_runtime.rs` (656) | working, **no leader election** | `main.rs:687-690` |
| F9 | NIP-IA identity archive/unarchive | `handlers/identity_archive.rs` (580) | working | `ingest.rs:1916` |
| F10 | Product feedback sidecar | `handlers/product_feedback.rs` (161) | working | `ingest.rs:1541` |
| F11 | Hourly S3 storage sweep + gauges | `storage_sweep.rs` (1090) | working | `main.rs:1460`, `:1474` |
| F12 | Workflow action sink | `workflow_sink.rs` (711) | **1 of 7 workflow actions implemented** | `main.rs:594` |

---

#### F1 — Community moderation commands

Five direct commands, never stored as events, each writing durable state plus a `moderation_actions` audit row.

| Kind | Delivers | Side effects actually implemented |
|---|---|---|
| 9040 ban | permanent or expiring community ban | `community_bans` upsert (`:169`), audit row (`:180`), cluster-wide live disconnect (`:195-200`), best-effort notice DM (`:204-220`) |
| 9041 unban | ban lift, refuses if not banned | `community_bans` clear (`:248`), audit row (`:256`) — **no notice DM** |
| 9042 timeout | mandatory-expiry write-block | `muted_until` upsert (`:287`), audit row (`:298`), best-effort notice DM (`:309-321`) — **no disconnect** |
| 9043 untimeout | mute clear, refuses if not timed out | `community_bans` clear (`:351`), audit row (`:358`) — **no notice DM** |
| 9044 resolve report | report status + decision audit + reporter DM | audit row (`:453`), `resolve_moderation_report` (`:461`), reporter notice (`:481-495`) |

**Declared but not delivered.** The module docs claim 9044's `delete`/`kick`/`ban` resolutions "fan out through the existing 9005/9001 + 9040 paths" (`moderation_commands.rs:20`, `:47-50`). Verified: `handle_resolve` performs **no** fan-out — it writes a `resolve:*` decision row and returns (`:453-499`). The docs do say the *client* composes the paired 9040-9043 (`:48-50`), so the relay-side behaviour is by design, but the summary table at `:14-21` reads as if the relay chains the enforcement. Nothing in the relay guarantees the enforcement ever happens.

**`escalated` report status has no producer.** 9044 accepts `action=escalate` but always stores `status=resolved` (`:380-385`); the `ReportRecord.status` doc advertises `escalated` (`buzz-db/src/moderation.rs:71`) and `ModerationNotice::body` has an `"escalated"` arm (`moderation_notices.rs:281`) that is unreachable.

---

#### F2 — Moderation authorization seam

Delivers an exhaustively unit-tested pure policy function (`decide_authority`, `moderation_authz.rs:146-181`) separated from its I/O, plus a 3-role capability grid.

**Non-functional in production:** the entire channel-local authority branch.

| Capability | Production requester | Status |
|---|---|---|
| `Ban`, `Unban`, `Timeout`, `Untimeout` | 9040–9043 | live |
| `ResolveReport` | 9044 | live |
| `ViewQueue` | `GET /moderation/reports`, `/moderation/audit` | live |
| **`DeleteMessage`** | **none** | dead — zero call sites |
| **`Kick`** | **none** | dead — zero call sites |

Because no caller passes a `channel_id` (all 6 sites pass `None`) and no caller requests `DeleteMessage`/`Kick`, the `get_member_role` lookup (`:120-131`) is unreachable and `ModerationAuthority::ChannelRole` (`:174-178`) can never be returned. The 9005-delete and 9001-kick paths use a **separate, unrelated** authorization function, `side_effects::validate_admin_event` (`ingest.rs:1903-1907`) — the very function the module doc says the seam is "the bridge … missing today" for (`moderation_authz.rs:73-75`). The bridge exists but is not connected.

The declared-and-recorded `ModerationAuthority` is also non-functional: computed on every call, discarded by every caller, never written to the audit row despite the doc at `:61`.

---

#### F3 — Relay-signed moderation notice DMs

Delivers a closed-loop notification primitive: one DM thread per (community, user), relay-authored, with a named `"{host} Moderation"` kind:0 profile, kind:39000 discovery, and per-source idempotency.

| Template | Production producer |
|---|---|
| `Restriction { kind: "ban" }` | 9040 (`moderation_commands.rs:208`) |
| `Restriction { kind: "timeout" }` | 9042 (`moderation_commands.rs:313`) |
| `ReportResolved` | 9044 (`moderation_commands.rs:485`) |
| **`ContentActioned`** | **none** — declared (`moderation_notices.rs:52-57`), rendered (`:288-292`), tested (`:388-397`), never constructed outside tests |

`ContentActioned` is documented as the "actioned-author" notice for the same primitive (`moderation_notices.rs:25-26`) and its body mirrors the delete tombstone's `public_reason` — but nothing in the 9005 delete path calls `send_moderation_notice`. Verified: the only 3 call sites are in `moderation_commands.rs`.

**Also non-delivered:** unban (9041) and untimeout (9043) send no notice at all, so a user learns they were restricted but never that the restriction was lifted. The `Restriction` body promises to render the expiry ("with expiry rendered into the message", `moderation_notices.rs:66`) — the actual body does **not** include any expiry (`:296-305`), only the action verb and reason.

Replies are non-replyable in v1 by intent (`moderation_notices.rs:27-28`); the profile `about` string tells the user replies are not monitored (`:191-192`).

---

#### F4 — NIP-56 report intake

Delivers tenant-fenced report intake for three target classes with idempotency on the signed event id, and remains available during a write-block so users can signal abuse while timed out (`ingest.rs:1551-1559`).

Working: `e` (event, in-tenant only), `x` (media blob via tenant-scoped sidecar), `p`-only (community-local pubkey report). 7 report types.

**Known Phase-1 limitation, documented in-code:** blob lookup cannot distinguish not-found from transient storage failure, so both surface as `invalid: report target blob not found` (`report.rs:66-70`).

**Dead field:** `ParsedReportTarget::{Event,Blob}.author_pubkey` is parsed and validated but never persisted (`report.rs:106-107`, `:112-113`; discarded at `:54`, `:65`).

**Dead public constant:** `REPORT_TYPES` is `pub` with zero external consumers (`report.rs:29`).

---

#### F5 — NIP-43 relay-membership admin

Delivers four operations with a documented permission matrix and TOCTOU-safe removal.

| Kind | Delivers |
|---|---|
| 9030 | idempotent member add (admin/owner); admins can only add `member` |
| 9031 | member remove; admin path is a conditional delete restricted to `member` targets; owner path refuses other owners |
| 9032 | role change (owner only); `owner` role unreachable by design |
| 9033 | workspace icon set/clear (admin/owner), http(s) URL or `data:image/*` |

Every mutation best-effort publishes NIP-43 announcement events (`relay_admin.rs:214-220`, `:274-279`, `:334-336`).

**Misleading behaviour:** 9030 with `role=owner` says "use kind:9032 to promote to owner" (`:185`) but 9032 refuses `owner` too (`:301`). There is no event-based path to owner promotion at all; it requires `RELAY_OWNER_PUBKEY` config or the operator endpoint.

**Missing relative to the moderation path:** no ban re-check. `relay_admin.rs` contains zero references to `moderation_restriction_state`, and `ingest.rs:1613` exempts relay-admin kinds from the write gate — so a banned owner/admin with a surviving socket can still mutate membership.

---

#### F6 — Operator community provisioning

Delivers deployment-root community creation over `POST /operator/communities`, gated by the `RELAY_OPERATOR_PUBKEYS` allowlist and NIP-98 with a replay guard, plus strict canonical-authority host validation (13 unit tests, `community_provisioning.rs:354-443`).

Two modes:
- `create_only=true` — atomic host+owner creation, refuses existing hosts, enforces a per-owner community limit.
- default convergence mode — idempotent on the host row and **can rotate an existing community's owner** (`:236-247`, `:321-334`). Documented as the price of retry convergence; clients acting for end users are *told* to use `create_only` but nothing enforces it.

Also exports two validators reused by 5 other operator endpoints: `validate_pubkey_hex` (`:71`) and `normalize_candidate_host` (`:180`).

---

#### F7 — NIP-PL push-lease validation

Delivers a genuinely strict, closed-world envelope + encrypted-plaintext validator: 4 permitted public tags, duplicate-key rejection at every JSON depth via a custom `DeserializeSeed`, active/inactive schema bifurcation, server-resolved origin binding, 5-member filter allowlist, self-`#p` confinement, v4-UUID `#h` validation, and 12 hard-coded quotas.

Strongest invariant in the module: a unit test asserts the SQL trigger allowlist in `migrations/0018_push_match_queue.sql` uses `PUSH_KINDS` **exactly** (`push_lease.rs:696-710`), so the Rust and DB views of push-eligible kinds cannot drift.

**Declared but non-functional: the `urgent` delivery class.** `URGENT_KINDS = &[]` (`:16`) and `supported_classes` omits `"urgent"` (`:509`), so a lease requesting `class: "urgent"` is rejected earlier by `class not supported` (`:246`) and the urgent-kind confinement check at `:281-283` is unreachable. `class_rank` still ranks it in both copies (`:582`, `push_runtime.rs:574`) and NIP-11 publicly advertises `urgent_kinds: []` (`nip11.rs:209`).

**Declared but unused public API:** 8 public items with zero external consumers — `validate_envelope`, `parse_plaintext`, `validate_plaintext`, `LeaseEnvelope`, `LeasePlaintext`, `LeaseLimits`, `AppProfile`, `MAX_SAFE_JSON_INTEGER`. The module presents itself as a reusable validation library (`:1-6`) but only `accept` is called.

**Naming vs behaviour:** `ActiveLease.endpoint_grant` is documented as an "opaque endpoint grant issued by the stateless gateway" (`buzz-db/src/push.rs:94`), but the relay stores the client-supplied `endpoint` string verbatim (`push_lease.rs:544`, `:555`) and forwards it as `endpoint_grant` in the delivery body (`push_runtime.rs:507-515`). The relay never mints a grant.

---

#### F8 — NIP-PL matcher + delivery worker

Delivers two always-on background loops (spawned as one unit, `main.rs:686-692`):

**Matcher** — batch-claims up to 64 accepted events per community, loads leases + membership pairs once per batch, evaluates matches purely (no DB in `match_job`), flushes all wakes in one transaction, and reaps poison jobs on a separate 30 s cadence. Per-batch cost is documented as constant regardless of match count (`push_runtime.rs:52-56`). Idle backoff 250 ms → 2 s.

**Delivery worker** — enumerates communities via `usage_community_hosts()`, claims 16 wakes each, and for every wake performs revalidate → membership → revalidate before a single NIP-98-signed HTTPS POST. Handles 6 distinct response cases with generation-fenced endpoint invalidation and exponential retry to 8 attempts. Idle backoff 500 ms → 2 s.

**Notable design guarantee delivered:** the gateway request id is the durable wake row id and is byte-stable across retries, pinned by an HTTP-level test that runs a real axum server (`push_runtime.rs:626-655`).

**Not delivered: leader election.** Unlike the storage sweep, both loops run on **every** pod with no advisory lock (`main.rs:686-692`). Correctness rests entirely on DB claim/fence tokens; the cost is N× the claim-scan load at N pods. The delivery worker's per-community scan is also unbounded — it iterates every community returned by `usage_community_hosts()` on every sweep (`push_runtime.rs:320-338`).

**Not delivered: SSRF exposure surface.** By construction the destination URL is operator config, validated to an exact HTTPS `/v1/deliveries/apns` path (`config.rs:341-361`) — user data travels only in the body. This is a positive: there is no user-controlled URL anywhere in the delivery path.

---

#### F9 — NIP-IA identity archive

Delivers three consent paths (self / community owner-or-admin / cryptographic NIP-OA key owner), a NIP-70 protected-event requirement, optional successor-key (`replaced-by`) recording, and relay-signed 8002/8003 deltas plus a 13535 snapshot.

The distinguishing feature is **live revocation**: an owner-signed archive request is re-verified against the target's *current* kind:0 auth tag, so replacing your kind:0 immediately invalidates outstanding owner requests (`identity_archive.rs:270-296`). This is the module's only real integration test (`:515-578`).

Requests deliberately fall through to normal storage so the delta's `["e", request_id]` audit reference resolves (`ingest.rs:1909-1912`) — meaning 9035/9036 are the **only** kinds in this module that produce an `events` row and therefore the only ones that reach the `buzz-audit` hash chain.

Delta and snapshot publication failures only warn (`:130-136`), so `archived_identities` can silently diverge from the event-backed view. Unarchive is a hard `DELETE` (`buzz-db/src/archived_identities.rs:83-91`) — the consent/actor/reason provenance is destroyed, not tombstoned.

---

#### F10 — Product feedback sidecar

Delivers private, operator-only feedback capture that bypasses event storage and fan-out entirely: optional 3-value category, 32 KiB body, 64 KiB tag array, and full `imeta` attachment validation + blob verification against tenant media (`product_feedback.rs:26-36`). Smallest file in the group (161 LOC) with 4 focused unit tests.

---

#### F11 — S3 storage sweep

Delivers a cadence-decoupled bucket sweep whose results are cached and re-published on every usage tick, so a transient S3 blip never blanks dashboards:

| Capability | Site |
|---|---|
| Single-flight, harvest-then-spawn under one lock | `storage_sweep.rs:151-256` |
| Warm-cache retention across failures | `:175-184`, test `:531-591` |
| Immediate respawn after failure (self-healing) | `:105-127`, `:89-103` |
| Whole-attempt timeout, converted to `SweepError::Timeout` | `:249-252` |
| Object cap enforced before folding each page | `buzz-media/src/bucket_index.rs:392-395` |
| Hard kill switch suppressing even health gauges | `main.rs:1454-1456` |
| Zeroing of per-community series on disappearance / rename / scope exclusion | `:333-345`, test `:857-1088` |
| Unmapped-community rollup gauge | `:318-323`, `:330` |
| Demoted-leader safety (unharvested snapshot never published) | test `:643-670` |

18 gauges emitted: 3 health (`sweep_ok`, `sweep_failures`, `sweep_duration_seconds`), 1 freshness (`sweep_age_seconds`), 4 totals (physical/logical × bytes/objects), 7 anomaly/orphan, 1 unmapped rollup, 2 per-community series.

By far the best-tested file in the group: 15 tests including paused-time (`#[tokio::test(start_paused = true)]`) coverage of stall/timeout and a 3-scenario state-machine regression for stale series.

---

#### F12 — Workflow action sink

**This is the module's largest gap.** `ActionSink` declares exactly **one** method (`buzz-workflow/src/action_sink.rs:44-64`) and `RelayActionSink` implements exactly that one (`workflow_sink.rs:172-179`).

Against the 7 action types ARCHITECTURE.md:533-542 advertises:

| Action | Where it executes | Wired end-to-end? | Evidence |
|---|---|---|---|
| `send_message` | `workflow_sink.rs:180-357` | **YES** | full validate → build → persist → dispatch path |
| `send_dm` | `buzz-workflow/src/executor.rs:575-579` | **NO** | `Err(NotImplemented("SendDm"))` at `:578`; `// TODO (WF-07)` at `:577` |
| `set_channel_topic` | `executor.rs:580-584` | **NO** | `Err(NotImplemented("SetChannelTopic"))` at `:583`; `// TODO (WF-07)` at `:582` |
| `add_reaction` | `executor.rs:585-607` → `add_reaction_impl` `:885-919` | **NO** | POSTs `{BUZZ_RELAY_BASE_URL}/api/messages/{id}/reactions` (`:886-888`). Verified: `router.rs` registers **zero** `reactions` routes and zero `/api/messages` routes. Every attempt returns `WebhookError("AddReaction: relay returned 404 …")` (`:903-908`). Without the `reqwest` feature it silently returns `{"added": false, "skipped": true}` (`:597-606`) |
| `call_webhook` | `executor.rs:608+` | yes (own HTTP client, outside this module) | — |
| `request_approval` | `executor.rs` → `StepResult::Suspended` | **NO** | ARCHITECTURE.md:826 (WF-08) |
| `delay` | `executor.rs` | yes (outside this module) | — |

**Verified answer to the ARCHITECTURE.md §9 question:** `workflow_sink` implements **1 of 7** workflow actions. ARCHITECTURE.md items 5 and 6 (`:826-827`) are accurate but incomplete — `add_reaction` is a third broken action not listed as a limitation, and ARCHITECTURE.md:541 presents it as working. Because the trait has a single method, `send_dm` and `set_channel_topic` cannot be implemented without widening `ActionSink`.

**What `send_message` does deliver, well:**
- Community-correct tenancy (never re-derives from `config.relay_url`, fails closed on an unmapped community, `:190-210`).
- Access control: owner must be a channel member unless the channel is open (`:243-251`).
- `@Name` → `p` tag mention resolution that *defines* the agent-wake contract, deliberately conservative (members-only, exact name, greedy-longest, ambiguity wakes nobody) with genuinely hard Unicode correctness — the case-folding walk works in original-char coordinates because `İ`(U+0130) lowercases to two code points (`:78-96`), regression-tested at `:490-503`, `:544-560`, `:590-604`.
- 18 unit tests for mention resolution; `resolve_mention_pubkeys` is the most thoroughly tested pure function in the group.

**What it does not deliver:** threading (always `depth: 0`, `:333`), so workflows cannot reply in-thread; and `dispatch_persistent_event`'s result is discarded (`:351`), so fan-out/search/audit failures are silent.

---

#### 2. Cross-feature summary of declared-but-non-functional items

| Item | Declared at | Production producers/consumers |
|---|---|---|
`ModerationAction::DeleteMessage` | `moderation_authz.rs:32` | 0 |
`ModerationAction::Kick` | `moderation_authz.rs:34` | 0 |
`ModerationAuthority::ChannelRole` | `moderation_authz.rs:68` | unreachable |
`ModerationAuthority` return value | `moderation_authz.rs:61` | computed, discarded by all |
`ModerationNotice::ContentActioned` | `moderation_notices.rs:53` | 0 |
`ReportRecord.status = "escalated"` | `buzz-db/src/moderation.rs:71` | 0 |
`ModerationNotice::body` `"escalated"` arm | `moderation_notices.rs:281` | unreachable |
`NewAction.matched_principal` | `buzz-db/src/moderation.rs:139` | always `None` (`moderation_commands.rs:540`) |
`NewAction.reason_code` / `.private_reason` / `.channel_id` | `:130-136` | always `None` (`moderation_commands.rs:536-539`) |
`MODERATION_ACTION_CHECK_VOCAB` `"delete_message"`, `"kick"` | `:105-106` | 0 writers |
`resolution_audit_action` `"resolve:unknown"` | `moderation_commands.rs:511` | unreachable, and not in the CHECK vocab |
`ParsedReportTarget::{Event,Blob}.author_pubkey` | `report.rs:106`, `:112` | parsed, never persisted |
`REPORT_TYPES` (pub) | `report.rs:29` | 0 external |
`URGENT_KINDS` / `class: "urgent"` | `push_lease.rs:16` | unreachable |
`push_lease` 8 public validation items | `push_lease.rs:19-81`, `:83`, `:149`, `:194` | 0 external |
`StorageEmittedKey` (`pub(crate)`) | `storage_sweep.rs:121` | 0 outside its own file |
`send_dm`, `set_channel_topic` workflow actions | `buzz-workflow/src/executor.rs:575`, `:580` | `NotImplemented` |
`add_reaction` workflow action | `buzz-workflow/src/executor.rs:585` | 404 — route not registered |
