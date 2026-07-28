## Module: buzz-relay — moderation, admin & background workers (`crates/buzz-relay/src`)
### Aspect: Security

---

#### 1. Authorization on every privileged path

| # | Path | Transport auth | Handler authorization | Tenant fence | Ban re-check | Freshness |
|---|---|---|---|---|---|---|
| 1 | 9040 ban | NIP-42 / NIP-98 + `Scope::MessagesWrite` (`ingest.rs:216`) | `authorize_moderation_action(Ban)` (`moderation_commands.rs:156-166`) | `tenant.community()` on both role reads (`moderation_authz.rs:96-116`) | **yes** (`:103-108`) | ±120 s (`:113-121`) |
| 2 | 9041 unban | same | `Unban` (`:235-244`) | yes | yes | yes |
| 3 | 9042 timeout | same | `Timeout` (`:274-283`) | yes | yes | yes |
| 4 | 9043 untimeout | same | `Untimeout` (`:338-347`) | yes | yes | yes |
| 5 | 9044 resolve | same | `ResolveReport` (`:399-408`) + report must exist in-tenant (`:414-417`) | yes | yes | yes |
| 6 | `GET /moderation/reports` | NIP-98 + replay guard (`bridge.rs:2031-2033`) | `ViewQueue` (`bridge.rs:2036-2049`) | host-derived tenant | **no** | n/a |
| 7 | `GET /moderation/audit` | same | same | same | **no** | n/a |
| 8 | 9030 add member | NIP-42/98 + `Scope::AdminUsers` (`ingest.rs:251-256`) + channel-scoped-token rejection (`ingest.rs:1512-1516`) | inline `sender_role != "admin" && != "owner"` (`relay_admin.rs:177`) | `tenant.community()` (`:135`) | **NO** | ±120 s (`:125-130`) |
| 9 | 9031 remove member | same | inline (`:227`) + self-check (`:231`) + atomic role-conditional delete (`:243`) | yes | **NO** | yes |
| 10 | 9032 change role | same | inline `sender_role != "owner"` (`:286`) + self-check (`:290`) | yes | **NO** | yes |
| 11 | 9033 set workspace icon | same | inline (`:148`) | yes | **NO** | yes |
| 12 | 9035 archive identity | NIP-42/98 + `Scope::UsersWrite` (`ingest.rs:266`) | 3 consent paths (`identity_archive.rs:228-251`), NIP-70 `["-"]` required (`:156-167`) | `tenant.community()` on role + profile query (`:241`, `:279`) | **NO** (subject to ingest's own gate — see §3) | ±120 s (`:141-153`) |
| 13 | 9036 unarchive | same | same | yes | (as above) | yes |
| 14 | 30350 push lease | NIP-42/98, author-only kind (`kind.rs:120`) | none beyond authorship — the lease is self-owned | tenant-derived origin binding (`push_lease.rs:494`, `:585-596`) | subject to ingest's gate | expiration-only skew (`:138-143`) |
| 15 | 1984 report | NIP-42/98 + `Scope::MessagesWrite` (`ingest.rs:212`) | **none** — any authenticated principal may report | target must resolve in-tenant (`report.rs:56`, `:69`) | **deliberately bypassed** (`ingest.rs:1551-1562`) | none |
| 16 | 42000 feedback | same | **none** | tenant-scoped insert (`product_feedback.rs:57`) | **deliberately bypassed** (`ingest.rs:1538-1541`) | none |
| 17 | `POST /operator/communities` | NIP-98 against `RELAY_OPERATOR_API_ORIGIN` + replay guard (`api/operator.rs:104-135`, `:155-163`) | `RELAY_OPERATOR_PUBKEYS` allowlist, fail-closed on empty (`community_provisioning.rs:255-266`) | **above the tenant fence by design** (`:3-14`) | n/a | NIP-98 window |
| 18 | workflow `send_message` | none — internal sink | workflow owner must be a channel member unless the channel is open (`workflow_sink.rs:243-251`) | community comes from the run, never `config.relay_url` (`:190-210`) | **no** | n/a |
| 19 | push delivery | outbound NIP-98 signed by the relay key (`push_runtime.rs:551-565`) | 3-layer read authorization per wake (`:222-233`, `:250`) + membership revalidation at send (`:372-402`) | per-community claim (`:322-325`) | **no** | claim/generation fences |
| 20 | storage sweep | none — leader-gated background task | Postgres advisory lock (`main.rs:1414-1421`) | reads the whole bucket across all tenants | n/a | n/a |

##### 1.1 Findings

**S-1 (HIGH) — Relay-admin commands (9030–9033) have no ban re-check.** `ingest.rs:1613` exempts `is_relay_admin_kind` from the durable ban/timeout write gate, on the stated grounds that "relay-admin commands retain their separate authorization policy" (`ingest.rs:1594-1596`). That separate policy is `relay_admin.rs:133-142`, which reads only `relay_members.role` — grep for `restriction`, `banned`, or `moderation_restriction_state` in `relay_admin.rs` returns **zero hits**. The moderation handler pointedly does re-check, with an in-code rationale that HTTP NIP-98 requests and already-authenticated sockets can reach the handler without a fresh NIP-42 challenge (`moderation_commands.rs:97-101`). That exact reasoning applies verbatim to 9030–9033 and is not implemented. A banned owner or admin with a surviving socket, or issuing over NIP-98, can add members, remove members, change roles, and rewrite the workspace icon.

**S-2 (MEDIUM) — The "reject channel-scoped API tokens" contract for 9040–9044 is not enforced.** `moderation_commands.rs:50` pins it as part of the routing contract. Ingest rejects channel-scoped tokens only for relay-admin kinds (`ingest.rs:1512-1516`) and leave requests (`:1520-1523`). The generic "channel-scoped tokens cannot publish global events" gate sits at `ingest.rs:1721-1724`, **after** the moderation dispatch's early `return` at `:1582-1586`, so it never runs for 9040–9044. Combined with `required_scope_for_kind` granting them only `Scope::MessagesWrite` (`ingest.rs:216`), a legacy channel-scoped WS API token held by a community owner/admin can issue a community-wide ban. Privilege is not escalated beyond the token holder's community role, but the token's channel confinement is silently void.

**S-3 (MEDIUM) — Community moderation has no self-action prevention.** `moderation_commands.rs` never compares actor to target. An owner issuing 9040 against their own pubkey passes `decide_authority` (`moderation_authz.rs:149-150`), writes the ban row (`:169`), and then has their own sockets closed cluster-wide (`:195-200`). Recovery is blocked by the handler's own ban re-check (`:103-108`), so it requires a second owner/admin or direct DB access. Contrast `relay_admin.rs:231-233` (`cannot remove yourself`) and `:290-292` (`cannot change your own role`), which do implement the check. An admin is protected only incidentally, by the peer guard tripping on their own `admin` role (`moderation_authz.rs:165-167`).

**S-4 (MEDIUM) — Legacy provisioning mode can rotate an existing community's owner.** Without `create_only`, an operator-signed request runs `bootstrap_owner` on an existing community, demoting the previous owner to admin (`community_provisioning.rs:321-334`). This is documented and deliberate (`:236-247`) but it makes the operator allowlist a community-takeover capability, not just a creation capability. The mitigation ("clients provisioning on behalf of end users must use create_only", `:317-320`) is prose, not code.

**S-5 (LOW) — 1984 and 42000 accept writes from banned actors in the missed-disconnect window.** Both are dispatched before the restriction gate (`ingest.rs:1541`, `:1562` vs `:1613`). For reports this is explicitly intended and reasoned (`ingest.rs:1551-1559`: reports are non-actioning signals, mod-only visibility). For 42000 product feedback there is **no** stated rationale — a banned user can keep writing 32 KiB bodies plus 64 KiB of tags and `imeta`-verified media references into the operator feedback table indefinitely.

**S-6 (LOW) — No authorization at all on report submission.** Any authenticated principal in the tenant may report any in-tenant event, blob, or pubkey. Idempotency is per signed event id (`buzz-db/src/moderation.rs:39`), so distinct signed reports from one pubkey all land. See §7.

---

#### 2. Audit-trail coverage of privileged mutations

`buzz-audit` declares 11 actions (`buzz-audit/src/action.rs:5-29`). Production emits exactly two: `EventCreated` (`handlers/event.rs:560`) and `MediaUploaded` (`api/media.rs:428`). Confirmed by grep: **none of the 12 assigned files references `buzz_audit`, `AuditAction`, or `state.audit_tx`.**

> ARCHITECTURE.md:497 says "**10 audit actions**" and omits `MediaUploaded` — stale by one variant, and it enumerates declared actions rather than emitted ones.

##### 2.1 Unaudited privileged mutations (the finding)

| Mutation | `events` row | Hash chain | `moderation_actions` | Durable trail? |
|---|---|---|---|---|
| 9040 ban | no | **no** | yes (`moderation_commands.rs:180`) | `moderation_actions` only |
| 9041 unban | no | **no** | yes (`:256`) | `moderation_actions` only |
| 9042 timeout | no | **no** | yes (`:298`) | `moderation_actions` only |
| 9043 untimeout | no | **no** | yes (`:358`) | `moderation_actions` only |
| 9044 resolve | no | **no** | yes (`:453`) | `moderation_actions` only |
| **9030 add member** | no | **no** | **no** | **`tracing::info!` only** (`relay_admin.rs:203-209`) |
| **9031 remove member** | no | **no** | **no** | **`tracing::info!` only** (`:268-272`) |
| **9032 change role** | no | **no** | **no** | **`tracing::info!` only** (`:327-332`) |
| **9033 set workspace icon** | no | **no** | **no** | **`tracing::info!` only** (`:164`) |
| **operator provisioning / owner rotation** | n/a | **no** | **no** | **`tracing::info!` only** (`community_provisioning.rs:302-308`, `:336-343`) |
| 1984 report | no | **no** | no (own table) | `moderation_reports` |
| 42000 feedback | no | **no** | no (own table) | `product_feedback` |
| 30350 push lease | yes | **no** — ingest returns at `:2199` before the dispatch path | no | `push_leases` + event row |
| 9035/9036 identity archive | **yes** (fall-through, `ingest.rs:1909-1912`) | **YES** `EventCreated` | no | full |

**S-7 (HIGH) — 12 of the 14 privileged kinds this module owns produce no hash-chain entry.** Bans, unbans, timeouts, report resolutions, member additions/removals, role changes, and ownership-affecting provisioning are all invisible to `verify_chain()`. The only tamper-evident audit surface in the relay does not cover the relay's own privilege model.

**S-8 (HIGH) — Relay-admin and operator-provisioning mutations have no durable audit trail of any kind.** No event row, no hash-chain entry, no `moderation_actions` row. The record of "who made X an admin" or "who rotated community Y's owner" exists only in ephemeral process logs. `moderation_actions` at least gives moderation a queryable trail — membership and ownership changes get nothing.

**S-9 (MEDIUM) — `moderation_actions` is not hash-chained.** It is a plain table (`migrations/0006_moderation.sql`), so the moderation trail is not tamper-evident. `moderation_actions.matched_principal` — the field designed to record which NIP-OA principal an enforcement check matched (`buzz-db/src/moderation.rs:139`) and surfaced verbatim by the audit API (`api/bridge.rs:2166`) — is hard-coded `None` at its single writer (`moderation_commands.rs:540`) and is therefore **always NULL in production**. `reason_code`, `private_reason`, and `channel_id` are likewise always NULL (`:536-539`). Five of nine audit columns carry no data.

**S-10 (LOW) — Audit attribution for relay-signed events.** Moderation notice DMs are audited as `EventCreated` with the **relay pubkey** as actor (`moderation_notices.rs:178`), so the moderator behind the notice is not recoverable from the chain. Workflow messages correctly attribute the workflow owner rather than the relay key (`workflow_sink.rs:355`, rationale `handlers/event.rs:561-567`) — the moderation path does not follow that pattern.

---

#### 3. Self-ban and privilege-escalation prevention

| Vector | Prevented? | Mechanism |
|---|---|---|
| Admin bans the owner | **yes** | peer guard (`moderation_authz.rs:163-171`) |
| Admin bans a fellow admin | **yes** | same |
| Admin times out the owner/an admin | **yes** | same |
| Admin bans a non-member / plain member | allowed by design | guard trips on role, not on a missing row (`:165-167`, test `:246-266`) |
| Admin **unbans/untimeouts** the owner or an admin | **NOT prevented** | deliberately excluded from the guard (`:165`), documented as benign and owner-reversible (`:157-162`) |
| Admin promotes self to owner | **yes** | `cannot change your own role` (`relay_admin.rs:290-292`) and `cannot set role to owner` (`:300-302`) |
| Admin promotes another to admin | **yes** | owner-only (`relay_admin.rs:187-189`) |
| Anyone reaches owner via events | **yes** | 9030 refuses `role=owner` (`:184-186`), 9032 refuses `owner` (`:300-302`); the only paths are `RELAY_OWNER_PUBKEY` config or the operator endpoint |
| Admin removes the owner | **yes** | atomic `RemoveResult::IsOwner` (`:257-259`) |
| Admin removes another admin | **yes** | admin path is `remove_relay_member_if_role(…, "member")` (`:243`) → `RoleMismatch` (`:263-265`) |
| Promote-between-read-and-delete TOCTOU | **yes** | single conditional delete, explicitly to close that race (`:235-241`) |
| Channel owner/admin escalates to community actions | **yes** | `ChannelRole` authorizes only `DeleteMessage`/`Kick` (`moderation_authz.rs:174-178`, test `:294-310`) — and is unreachable anyway |
| **Owner self-bans** | **NO** | see S-3 |
| Actor bans by raw `p`-tag pubkey with no in-tenant existence check | allowed | `moderation_commands.rs:147` accepts any 64-hex; only 9044's report target is tenant-resolved (`:414`) |

**S-11 (LOW) — Denial reasons leak internal detail.** `authz_denial` wraps *any* `anyhow` error from the authorization seam, including `sqlx` errors from `get_relay_member`, as `restricted: {e}` (`moderation_commands.rs:548-550` + `moderation_authz.rs:99`, `:111`, `:127`). A DB failure therefore surfaces to an unauthorized client as a `restricted:` message containing database text. The HTTP moderation read path does this correctly, normalizing every denial to a fixed 403 string (`api/bridge.rs:2044-2049`).

---

#### 4. Ban cascade to NIP-OA agents; timeout non-cascade

##### 4.1 The cascade (auth seam only)

`handlers/auth.rs:107-186`, evaluated after signature verification and **before** the allowlist and relay-membership gates:

1. Read restriction state for the authenticated pubkey (`:120-131`).
2. Only if that is `Clear`, extract the cryptographically-proven NIP-OA owner from the self-proving auth tag with **no DB round-trip** (`:135-140`) and read the owner's restriction state (`:141-155`).
3. `Banned` ⇒ `blocked: you are banned from this community`, `AuthState::Failed`, reason frame on the **control** channel (not `send`, to avoid racing the cancel), then immediate socket close (`:158-186`).
4. `DbError` ⇒ deny with `error: internal error checking restriction state` — deliberately distinguished so a transient blip never tells an innocent user they are banned or pins `Failed` on a false premise (`:112-118`).

Semantics: **owner ban ⇒ agents banned; agent ban is agent-only** (`handlers/auth.rs:104-106`).

##### 4.2 The non-cascade (ingest write path)

`ingest.rs:1613-1650` checks `auth.pubkey()` **only**, with no owner resolution. Documented as deliberate (`ingest.rs:1598-1611`): for bans the cascade is *structural* — an agent whose owner is banned can never authenticate, so its socket never exists to reach ingest. Timeout has no auth-seam presence at all (it is write-block-only), so:

**S-12 (MEDIUM, accepted) — An owner timeout does not silence that owner's agents.** Stated as a "deliberate Phase-1 asymmetry" (`ingest.rs:1607-1608`), with the reason given as `IngestAuth` not carrying the self-proving auth tag and the follow-up named as a restriction-state cache (`:1608-1611`). Operationally: timing out a human leaves every agent they own free to post.

**S-13 (MEDIUM) — The cascade is absent on the moderation-command path.** `moderation_commands.rs:103-108` reads restriction state for `event.pubkey` only. An agent whose owner is banned but whose own pubkey is clear can, over HTTP NIP-98 (which never crosses the auth seam), reach `handle_moderation_command`. It will then be denied by `authorize_moderation_action` unless the agent pubkey itself holds `owner`/`admin` in `relay_members` — an unusual but not impossible configuration. The same gap applies to `relay_admin.rs` with no ban check at all (S-1).

##### 4.3 Live-enforcement split

Ban closes sockets cluster-wide (`moderation_commands.rs:195-200` → `state.rs:1018-1050`); timeout does not. The Redis publish is `tokio::spawn`ed fire-and-forget whose failure only warns (`state.rs:1043-1047`), justified because the durable row re-rejects at auth (`:1039-1042`) and the ingest gate backstops surviving sockets (`ingest.rs:1589-1596`).

**S-14 (MEDIUM) — A missed disconnect publish leaves read access open indefinitely.** The ingest write gate covers writes; nothing gates an established subscription's *reads*. On a pod that missed the Redis command, a banned user's open socket keeps receiving fan-out until it reconnects. `disconnect_pubkey_clusterwide` returns the local close count (`state.rs:1049`) and the caller discards it (`moderation_commands.rs:195`), so there is no signal that zero sockets were closed.

---

#### 5. Push-lease forgery and endpoint-hijack resistance

| Threat | Control | Site |
|---|---|---|
| Forged lease for another pubkey | 30350 is an author-only kind (`kind.rs:120`); ingest enforces `event.pubkey == auth.pubkey()` (`ingest.rs:1508-1512`); the event is signature-verified before routing | strong |
| Cross-tenant lease injection via the decrypted `origin` | `origin` must equal `canonical_origin(config.relay_url, tenant.host())` — scheme from config, authority from the **server-resolved** tenant (`push_lease.rs:494`, `:585-596`); module doc pins that tenant selection never comes from the plaintext (`:3-5`) | strong |
| Metadata leakage via public tags | only `d`/`expiration`/`exec`/`alt` permitted; anything else is a hard reject; duplicates rejected; every tag must have exactly 2 elements (`push_lease.rs:106-126`) | strong |
| Subscribing to another user's traffic | every positive filter must be narrowed (`:292-295`) and every `#p` value must equal the lease author (`:298-303`) | strong |
| Gift-wrap (1059) recipient-activity leak through wake timing | match-time counterpart of REQ's `#p` gate: the filter must be self-`#p` **and** the event must carry that author's `p` tag (`push_runtime.rs:292-310`, rationale `:287-290`); tested `:580-620` | strong |
| Reading events the lease author cannot see | `reader_authorized_for_event` (`push_runtime.rs:222-225`) + channel membership from the pre-loaded pair set (`:226-233`) + membership revalidation at send (`:372-402`) | strong |
| **Endpoint hijack** (claiming another device's APNs token) | DB-enforced: one active address per endpoint tuple ⇒ `EndpointAlreadyLeased` → `invalid: endpoint already leased` (`ingest.rs:2170-2174`); `endpoint_hash` is SHA-256 of the endpoint (`push_lease.rs:535`) | **first-claim-wins, no proof of possession** — see S-15 |
| Lease rollback / replay | NIP-01 addressable ordering (`StaleEvent`) + strictly-increasing generation watermark (`StaleGeneration`) (`ingest.rs:2163-2167`); generation must be a positive JSON-safe integer (`push_lease.rs:198-200`) | strong |
| Source-event reuse across addresses | `SourceEventCollision` (`ingest.rs:2179-2183`) | strong |
| Quota exhaustion | 16 active leases per author (`push_lease.rs:479`), 12 further hard-coded quotas (`:513-521`), 64 KiB ciphertext, 32 KiB plaintext (`:477-478`) | strong |
| JSON parser confusion (duplicate keys) | custom `DeserializeSeed` rejecting duplicates at every depth (`push_lease.rs:383-455`), tested at top level and nested (`:637-649`) | strong |
| Executor-key confusion | `exec` must equal `config.push_executor_key_id` (`:484-486`) | strong |
| Non-finite JSON numbers | rejected (`push_lease.rs:419`) | strong |
| Stale 410 killing a rotated lease | endpoint invalidation is generation-fenced (`push_runtime.rs:456-465`) | strong |
| Double-delivery via a fresh request id after timeout | request id is the durable wake row id, stable across retries, pinned by an HTTP test (`push_runtime.rs:486-490`, `:626-655`) | strong |

**S-15 (MEDIUM) — Endpoint ownership is first-claim-wins with no proof of possession.** The relay never validates that the submitting pubkey controls the APNs token; it accepts any 1..=4096-byte `endpoint` string, hashes it, and lets the DB unique constraint arbitrate (`push_lease.rs:533-535`, `:563-570`). A user who learns another device's token can register it first and thereafter receive that device's wakes — or, once registered, block the legitimate device with `endpoint already leased`. The wake payload itself is content-free (`DeliveryRequest` carries only `v`/`endpoint_grant`/`request_id`/`expires_at`, `push_runtime.rs:31-37`), so the leak is wake *timing* and *existence*, not message content — but timing plus the 3-layer read authorization means the attacker learns when the *victim's* pubkey has readable traffic, since the lease's own author gates matching. Net: the practical impact is denial of push to the real device plus a notification-timing oracle scoped to the attacker's own readable events.

**S-16 (LOW) — `endpoint_grant` is a misnomer that invites a false sense of opacity.** The DB field is documented as "Opaque endpoint grant issued by the stateless gateway" (`buzz-db/src/push.rs:94`), but the relay stores the client-supplied endpoint verbatim (`push_lease.rs:544`, `:555`) and forwards it unchanged (`push_runtime.rs:507-515`). The stored value is a raw APNs device token — a credential-adjacent secret — persisted in plaintext in `push_leases` and transmitted in every delivery body. It is never logged (verified: `push_runtime.rs` logs only `wake=%id`), which is the one saving grace.

**S-17 (LOW) — `urgent` class dead code creates a latent policy hole.** `URGENT_KINDS = &[]` (`push_lease.rs:16`) and `"urgent"` absent from `supported_classes` (`:509`) means the urgent-kind confinement check at `:281-283` never runs. If a future change adds `"urgent"` to `supported_classes` without also populating `URGENT_KINDS`, the confinement check would then reject *all* urgent leases — fail-closed, so the hole is benign today. `class_rank` in both copies still ranks `urgent` highest (`push_lease.rs:582`, `push_runtime.rs:574`).

---

#### 6. SSRF in push delivery

**Not exposed.** The destination is `state.config.push_gateway_delivery_url`, an operator-configured `url::Url` validated at boot to be:
- scheme exactly `https` (`config.rs:349`)
- host present (`:350`)
- no username, no password (`:351-352`)
- path exactly `/v1/deliveries/apns` (`:353`)
- no query, no fragment (`:354-355`)

Failure to satisfy any of these is a boot-time `ConfigError` (`config.rs:356-360`), i.e. fail-closed. The client-controlled `endpoint_grant` travels only in the JSON body (`push_runtime.rs:507-515`); no user-supplied value ever reaches the URL, headers, or method. The NIP-98 `u` tag is derived from the same validated URL (`push_runtime.rs:552-556`).

Residual risks:

**S-18 (LOW) — Redirects are not disabled.** `reqwest::Client::builder()` at `push_runtime.rs:313-316` sets only `.timeout(...)`; `redirect::Policy` is left at the default (follow up to 10). A compromised or misconfigured gateway host could redirect the delivery POST — including the NIP-98 `Authorization` header and the endpoint token — to an arbitrary host. `buzz-workflow`'s `call_webhook` explicitly disables redirects (per ARCHITECTURE.md:539); this path does not.

**S-19 (LOW) — The default destination is a hard-coded third-party host.** Unset `BUZZ_PUSH_GATEWAY_DELIVERY_URL` falls back to `https://push.buzz.xyz/v1/deliveries/apns` (`config.rs:339`, `:755-758`), which also **enables** the matcher and delivery worker (`main.rs:686-692`) and lease acceptance (`push_lease.rs:480-482`). A self-hosted relay that never configures push will still accept leases, run both loops, and attempt outbound HTTPS to `push.buzz.xyz` — leaking the existence of wakes (and the device token in the body) to a host the operator never chose. Disabling requires explicitly setting the variable to an *empty* string (`config.rs:753`), which is non-obvious.

---

#### 7. Report abuse surface

| Property | Value | Site |
|---|---|---|
| Who may report | any authenticated principal with `Scope::MessagesWrite` | `ingest.rs:212` |
| Rate limiting | **none** in this path. ARCHITECTURE.md:821 confirms the only `RateLimiter` implementation repo-wide is a test stub | — |
| Idempotency | per signed event id (`report_event_id`) | `buzz-db/src/moderation.rs:39` |
| Reportable while timed out | **yes**, by design | `ingest.rs:1551-1562` |
| Reportable while banned (missed-disconnect window) | **yes**, tolerated | `ingest.rs:1554-1559` |
| Target must exist in-tenant | yes for `e` (`report.rs:56-59`) and `x` (`:66-71`); **no** for `p`-only (`:73`) | |
| Note size cap | **none** — `event.content` is stored unbounded (`report.rs:222-228`); only the global event-size limit applies | |
| Self-reporting | not prevented | |
| Auto-action on report | **never**, by design (NIP-56) | `report.rs:3-6` |
| Reporter identity exposure | mod-queue only, never revealed to the author | `buzz-db/src/moderation.rs:42`; enforced by the notice-privacy invariant (`moderation_notices.rs:268-272`) |

**S-20 (MEDIUM) — Unbounded report flooding.** A single principal can generate arbitrarily many distinct signed 1984 events (each a fresh event id, so idempotency does not help) against any in-tenant target, each writing a `moderation_reports` row with an uncapped note. There is no per-pubkey rate limit, no per-target cap, and no dedup on `(reporter, target)`. The moderation read API caps at 500 rows per request (`api/bridge.rs:2053`), so a flood degrades the queue's usability for moderators. Contrast `product_feedback.rs:12-13`, which caps body at 32 KiB and tags at 64 KiB — the report path applies neither.

**S-21 (LOW) — Report note is relayed to the reporter but never sanitized on the resolution path.** 9044's `reason` tag is passed verbatim into the reporter's DM (`moderation_commands.rs:477-481` → `moderation_notices.rs:283-286`). The doc requires it be "safe for the reporter's eyes" (`moderation_commands.rs:44-47`); nothing enforces that. The notice is also indexed into full-text search via `dispatch_persistent_event` (`moderation_notices.rs:178`), so a moderator's `public_reason` becomes searchable content.

---

#### 8. Storage-sweep credential scope

| Property | Value | Site |
|---|---|---|
| Credentials | the **same** `BUZZ_S3_ACCESS_KEY` / `BUZZ_S3_SECRET_KEY` used for media upload and download | `config.rs:622-625`; shared `MediaStorage` handle at `main.rs:1459` |
| Additional IAM requirement | `s3:ListBucket` (or MinIO list) — surfaced in the failure log as an operator hint | `storage_sweep.rs:176-181` |
| Prefix scope | **empty prefix — whole bucket, every tenant** | `buzz-media/src/storage.rs:249-256` |
| Page size | 1000 objects per LIST | `main.rs:1466` |
| Cumulative cap | `BUZZ_STORAGE_SWEEP_MAX_OBJECTS`, default 1,000,000, checked before folding each page | `storage_sweep.rs:65-68`; `buzz-media/src/bucket_index.rs:392-395` |
| Data retained in memory | bounded per-sha/per-binding aggregates only, never the full listing | `bucket_index.rs:228-246` |
| Kill switch | `BUZZ_STORAGE_METRICS=off` — suppresses the sweep and every storage gauge including health ones | `main.rs:1454-1456` |

**S-22 (LOW) — No least-privilege separation for the sweep.** Enabling storage metrics requires adding `s3:ListBucket` to the relay's **read-write** media credential, over the whole bucket. There is no separate list-only credential and no per-tenant prefix restriction. A relay compromise therefore also yields a full cross-tenant object inventory. The kill switch is the only mitigation offered, and it is documented for that purpose (`storage_sweep.rs:42-45`, `deploy/charts/buzz/values.yaml:308-313`).

**S-23 (INFO) — Sweep output is cross-tenant by construction.** `per_community` maps community UUIDs to byte/object totals, and unmapped UUIDs roll into a global `buzz_storage_unmapped_community_bytes` gauge (`storage_sweep.rs:318-330`). Per-community series are gated by the same `EmissionScope` predicate as DB-derived gauges (`main.rs:1474-1477`), so scope exclusion does apply — but the aggregate totals (`buzz_total_storage_bytes`, orphan/anomaly counters) are unscoped and expose whole-deployment volume to any Prometheus scraper.

---

#### 9. Secret handling in logs

Reviewed all `tracing` calls in the 12 files. **No secret material is logged.**

| Candidate | Handling |
|---|---|
| APNs endpoint token (`endpoint_grant`) | never logged; `push_runtime.rs` logs only `wake=%uuid`, `?invalid_at`, and error text |
| Lease ciphertext / decrypted plaintext | `push_lease.rs` contains **zero** `tracing` calls |
| Relay secret key | used at `push_lease.rs:488` and for signing at `moderation_notices.rs:165`, `:198`, `workflow_sink.rs:304`, `push_runtime.rs:562`; never formatted into a log |
| Workspace icon (may be a base64 data URL up to 96 KB) | only its length is logged: `icon_len = icon.len()` (`relay_admin.rs:164`) |
| Moderation notice body / `public_reason` | never logged |
| Report note | never logged |
| Product-feedback body | never logged |
| NIP-98 `Authorization` header | constructed at `push_runtime.rs:551-565`, never logged |
| Pubkeys | logged as hex (`moderation_commands.rs:223`, `relay_admin.rs:203-209`, `identity_archive.rs:90-97`) — public data, appropriate |
| `DATABASE_URL` with an inline dev password | present as a **test-only** fallback literal `postgres://buzz:buzz_dev@localhost:5432/buzz` (`identity_archive.rs:439-440`); inside `#[cfg(test)]`, never logged |

**S-24 (LOW) — Database error text reaches clients.** Not a log issue but the same class of exposure: `error: database error: {e}` (`moderation_commands.rs:174` and 5 more), `database error: {e}` (`relay_admin.rs:137` and 6 more), and `restricted: {e}` wrapping a possible `sqlx` error (`moderation_commands.rs:549`). `push_lease.rs:572` is the only path that deliberately opaques the DB error — at the cost of losing the diagnostic entirely.

---

#### 10. Positive security controls worth preserving

| Control | Site |
|---|---|
| Single authorization seam for all moderation, with a pure exhaustively-tested policy core | `moderation_authz.rs:83-181`, tests `:184-330` |
| Ban re-checked at three independent layers (auth seam, ingest write gate, command handler) with fail-closed DB-error handling at each | `handlers/auth.rs:112-176`, `ingest.rs:1613-1650`, `moderation_commands.rs:103-108` |
| NIP-OA owner→agent ban cascade with no DB round-trip for owner extraction | `handlers/auth.rs:134-155` |
| DB-error denial distinguished from a real ban, so an innocent user is never told they are banned | `handlers/auth.rs:112-118` |
| Fail-closed operator allowlist, empty by default, plus a boot-time requirement that it be paired with an explicit API origin | `community_provisioning.rs:258-266`; `config.rs:577-580` |
| Canonical-authority host validation with a byte-identical round-trip requirement | `community_provisioning.rs:120-146` |
| NIP-98 replay guard on the operator endpoint, fail-closed when the guard itself errors | `api/operator.rs:104-135` |
| Closed-world push-lease validation: 4 permitted tags, 5 permitted filter members, duplicate-key rejection at every JSON depth | `push_lease.rs:106-126`, `:264`, `:383-455` |
| Push-eligible kinds pinned to the SQL trigger by an `include_str!` test | `push_lease.rs:696-710` |
| Gift-wrap wake-timing leak closed at match time | `push_runtime.rs:292-310` |
| Generation-fenced endpoint invalidation and claim-fenced completion | `push_runtime.rs:456-465`; `ClaimedWake.claim_id` (`buzz-db/src/push.rs:141`) |
| Exact-HTTPS-path gateway URL validation, fail-closed at boot | `config.rs:341-361` |
| NIP-IA live-revocation check: replacing your kind:0 auth tag invalidates outstanding owner-signed archive requests | `identity_archive.rs:270-296`, integration test `:515-578` |
| NIP-70 protected-event tag mandatory on archive requests | `identity_archive.rs:156-167` |
| Notice-DM privacy invariant: bodies built only from sanitized fields, never reporter identities or raw notes | `moderation_notices.rs:20-23`, `:268-272` |
| Workspace icon rejects `javascript:` and non-image data URLs, control chars, and whitespace | `relay_admin.rs:69-94`, tests `:441-450` |
| Workflow tenancy derived from the run's community, failing closed on an unmapped community — never re-derived from `config.relay_url` | `workflow_sink.rs:190-210` |
| Zero `unsafe`, zero production `unwrap()`, zero production `panic!` across 6,720 LOC | measured |
