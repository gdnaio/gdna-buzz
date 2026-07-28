## Module: buzz-relay — moderation, admin & background workers (`crates/buzz-relay/src`)
### Aspect: Business Rules

All rules below are read out of code, not documentation. Where a doc comment or `ARCHITECTURE.md` disagrees, the delta is called out inline.

---

#### A. Moderation authorization (BR-MOD-01 … BR-MOD-12)

**BR-MOD-01 — Single authorization seam.** Every moderation decision routes through `authorize_moderation_action` (`moderation_authz.rs:83-144`); no moderation handler performs an inline role check. Verified: `moderation_commands.rs` contains zero `get_relay_member`/`get_member_role` calls; all 5 command paths call the seam (`:156`, `:235`, `:274`, `:338`, `:399`).

**BR-MOD-02 — Community owner is unconditionally authorized.** `actor_role == "owner"` ⇒ `Ok(CommunityOwner)` for all 8 actions with **no guard rail**, including against another owner or admin target (`moderation_authz.rs:149-150`; test `:207-217`).

**BR-MOD-03 — Community admin holds every capability except restricting a peer.** `actor_role == "admin"` ⇒ `Ok(CommunityAdmin)` unless the action is `Ban` or `Timeout` **and** the target's `relay_members.role` is `owner` or `admin`, in which case: `an admin cannot ban or time out a community owner or fellow admin` (`moderation_authz.rs:163-171`).

**BR-MOD-04 — The peer guard trips on role, never on a missing row.** A target with no `relay_members` row (`target_role == None`) is bannable by an admin (`moderation_authz.rs:165-167` requires `Some("owner") | Some("admin")`; test `:246-266`). Rationale in-code: a drive-by spammer who already left must stay bannable (`moderation_authz.rs:157-159`).

**BR-MOD-05 — Restriction *lifting* is deliberately unguarded at the role seam.** `Unban` and `Untimeout` are excluded from the peer guard (`moderation_authz.rs:165`), so an admin may lift another admin's or the owner's restriction. Documented as intentional and "benign, audited, and owner-reversible" (`moderation_authz.rs:158-162`); test `:269-291`.

**BR-MOD-06 — Channel role grants only `DeleteMessage`/`Kick`, and only within `channel_id`.** `moderation_authz.rs:174-179`. **This rule is unreachable in production:** all 6 call sites pass `channel_id: None` and none request `DeleteMessage`/`Kick`, so `channel_role` is always `None` (`moderation_authz.rs:120-131`). Exercised by unit tests only (`:294-310`).

**BR-MOD-07 — Everyone else is denied with `moderator access required`.** Plain channel members and strangers fail every action (`moderation_authz.rs:178`; test `:313-327`).

**BR-MOD-08 — Authority never crosses the tenant fence.** Both role reads are keyed on `tenant.community()` (`moderation_authz.rs:96-131`). The module states callers must pre-resolve the target inside the same tenant (`:15-19`); `moderation_commands.rs` satisfies this for the report path (`:414`) but for pubkey targets there is nothing to resolve — a raw `p`-tag pubkey is accepted without any in-tenant existence check (`:147`, `:228`, `:264`, `:331`).

**BR-MOD-09 — Target-role lookup is conditional, not unconditional.** The second `get_relay_member` fires only for `(admin, Ban|Timeout, Pubkey)` (`moderation_authz.rs:105-116`). Owner and channel-role paths stay at one query. Side effect: an admin banning an *event* target would skip the guard entirely — currently unreachable because `Ban` is only ever paired with `ModerationTarget::Pubkey`.

**BR-MOD-10 — There is no Moderator tier in v1.** Roles are exactly community `owner`/`admin` plus channel `owner`/`admin` (`moderation_authz.rs:5-10`). Adding a tier is a change to `decide_authority` only.

**BR-MOD-11 — `ViewQueue` gates the read APIs.** `GET /moderation/reports` and `/moderation/audit` both require `ViewQueue` under `ModerationTarget::None` (`api/bridge.rs:2036-2049`), which by BR-MOD-02/03/07 means community owner or admin only. Denial is normalised to a fixed 403 `restricted: moderator access required` regardless of the underlying reason (`bridge.rs:2044-2049`).

**BR-MOD-12 — `ModerationAuthority` is computed and thrown away.** The doc comment says the matched authority is "recorded in the audit row" (`moderation_authz.rs:61`), but `insert_audit` accepts no authority argument (`moderation_commands.rs:518-527`) and both callers discard the return value (`moderation_commands.rs:165`, `bridge.rs:2044`). **Documentation delta.**

---

#### B. Self-action and last-owner protection (BR-MOD-13 … BR-MOD-18)

**BR-MOD-13 — Community moderation has NO self-action prevention.** `moderation_commands.rs` never compares `actor` to `target`. An owner can self-ban (9040 with own `p` tag) — `decide_authority` returns `CommunityOwner` (`moderation_authz.rs:149`), the ban row is written (`:169`), and the actor's own live sockets are then closed by `disconnect_pubkey_clusterwide` (`:195-200`). Recovery requires another owner/admin or direct DB access, because the ban re-check at `moderation_commands.rs:103-108` blocks the self-unban. An admin cannot self-ban only incidentally: the peer guard trips on their own `admin` role (`moderation_authz.rs:165-167`).

**BR-MOD-14 — Relay-admin removal has explicit self-protection.** `cannot remove yourself` (`relay_admin.rs:231-233`) and `cannot change your own role` (`relay_admin.rs:290-292`). Note the asymmetry with BR-MOD-13.

**BR-MOD-15 — The relay owner cannot be removed or demoted.** Owner removal is refused atomically in the DB, surfaced as `RemoveResult::IsOwner` → `cannot remove the relay owner` (`relay_admin.rs:257-259`). Role change on an owner fails the update and is disambiguated by a second read into `cannot change the relay owner's role` vs `member not found` (`relay_admin.rs:313-325`). This is the last-owner protection: there is no owner-count check, the owner row is simply immutable through 9031/9032.

**BR-MOD-16 — Ownership transfer via 9032 is blocked by design.** `cannot set role to owner` (`relay_admin.rs:300-302`) with an in-code DESIGN note that transfer is high-risk and must go through `RELAY_OWNER_PUBKEY` config (`relay_admin.rs:296-299`). 9030 likewise refuses `role=owner` with `invalid role: use kind:9032 to promote to owner` (`relay_admin.rs:184-186`) — a **misleading message**, since 9032 also refuses it.

**BR-MOD-17 — Admins cannot mint admins.** Only an owner may add a member at `role=admin` (`relay_admin.rs:187-189`). Role vocabulary is closed to `admin`/`member` (`relay_admin.rs:190-192`).

**BR-MOD-18 — Admin removal is TOCTOU-safe by construction.** Admins use `remove_relay_member_if_role(…, "member")` — a single conditional delete — rather than read-then-delete, explicitly to close the promote-between-read-and-delete race (`relay_admin.rs:235-249`). Owners use the unconditional `remove_relay_member` which itself refuses owners.

---

#### C. Freshness window (BR-MOD-19 … BR-MOD-21)

**BR-MOD-19 — ±120 s signed-timestamp window on every direct command.** Applied identically in four places with three different spellings:

| Path | Constant | Site |
|---|---|---|
| 9040–9044 | `MAX_COMMAND_SKEW_SECS = 120` | `moderation_commands.rs:81`, check `:113-121` |
| 9030–9033 | bare literal `120` | `relay_admin.rs:125-130` |
| 9035/9036 | bare literal `120` | `identity_archive.rs:141-153` |
| 30350 | `ALLOWED_SKEW = 120` (only against `expiration`, not `created_at`) | `push_lease.rs:476`, `:138` |

Rationale: these events are never stored, so replay of a captured command is the only threat and a tight window is the mitigation (`moderation_commands.rs:78-80`).

**BR-MOD-20 — The window is checked *after* the ban re-check for moderation, *after* nothing for relay-admin.** `moderation_commands.rs` reads restriction state at `:103-108` and only then checks freshness at `:113`. `relay_admin.rs` checks freshness first (`:122-131`) then reads the member row (`:133-137`).

**BR-MOD-21 — Clock-read failure fails closed.** `SystemTime::now().duration_since(UNIX_EPOCH)` errors map to `now = 0` (`moderation_commands.rs:115`, `relay_admin.rs:124`, `identity_archive.rs:146`), which makes `|event_ts - 0| > 120` true for any real timestamp ⇒ universal rejection. Correct direction, but it produces a confusing `now=0` in the error string.

---

#### D. Ban vs timeout semantics (BR-MOD-22 … BR-MOD-29)

**BR-MOD-22 — A ban is an admission boundary; a timeout is a write-block.** Ban is enforced at the NIP-42 auth seam (`handlers/auth.rs:107-186`) *and* as a durable backstop on the ingest write path (`ingest.rs:1616-1650`). Timeout has **no auth-seam presence at all** — it appears only on the ingest write path (`ingest.rs:1637-1643`), producing `restricted: you are timed out until <unix ts>` so the desktop can render a countdown (`ingest.rs:1584-1588` comment; `:1639-1641`).

**BR-MOD-23 — Ban closes live sockets cluster-wide, timeout does not.** 9040 calls `disconnect_pubkey_clusterwide` (`moderation_commands.rs:195-200`), which closes this pod's sockets synchronously and fire-and-forget-publishes a Redis `ConnControl::DisconnectPubkey` to every other pod (`state.rs:1018-1050`). 9042 has **no** disconnect call — a timed-out user keeps their socket and continues receiving events, only writes are refused.

**BR-MOD-24 — The live ban fan-out is best-effort; the durable row is the guarantee.** The Redis publish is `tokio::spawn`ed and its failure only logs (`state.rs:1043-1047`), justified in-code because the durable ban row rejects the member again at auth (`state.rs:1039-1042`). The ingest write-path gate exists precisely to cover a missed disconnect (`ingest.rs:1589-1596`).

**BR-MOD-25 — Ban expiry is permitted but not surfaced to the handler.** 9040 accepts an optional `expiration` tag ⇒ `None` means permanent (`moderation_commands.rs:148`, doc `:29`). The handler's own re-check reads only `RestrictionState.banned` (`moderation_commands.rs:135-142`), which omits `ban_expires_at` (`buzz-db/src/moderation.rs:432-436`); expiry semantics live entirely in the DB query.

**BR-MOD-26 — Timeout requires an expiration.** `invalid: timeout requires an expiration tag` (`moderation_commands.rs:270-271`). Unlike a ban, an unbounded timeout is not expressible.

**BR-MOD-27 — Moderation commands are exempt from the ban/timeout write gate so a timeout cannot disarm the tool that lifts it.** `ingest.rs:1613` excludes both `is_moderation_command_kind` and `is_relay_admin_kind`; the handler then re-imposes only the *ban* half (`moderation_commands.rs:103-108`), letting a timed-out admin issue 9043. Pinned by a unit test (`moderation_commands.rs:623-641`).

**BR-MOD-28 — The ban cascades from NIP-OA owner to agent; the timeout does not.** At the auth seam, if the authenticated pubkey is clear, the cryptographically-proven NIP-OA owner is checked and an owner ban denies the agent (`handlers/auth.rs:134-155`). The ingest write gate checks the **authoring pubkey only, with no cascade** (`ingest.rs:1598-1611`) — explicitly documented as a deliberate Phase-1 asymmetry because `IngestAuth` does not carry the self-proving auth tag (`ingest.rs:1606-1611`). Net effect: **an owner timeout does not silence that owner's agents.**

**BR-MOD-29 — Ban-state DB errors deny, but are distinguished from real bans.** Auth seam: `BanOutcome::DbError` ⇒ `error: internal error checking restriction state` rather than claiming a ban (`handlers/auth.rs:112-176`). Ingest gate: `IngestError::Internal("error: internal error checking restriction state: {e}")` (`ingest.rs:1645-1650`). Handler: `error: database error checking restriction state: {e}` (`moderation_commands.rs:108`). Three different strings for the same condition.

---

#### E. Report and resolution rules (BR-MOD-30 … BR-MOD-40)

**BR-MOD-30 — Reports are signals, never triggers.** The relay persists to the tenant queue and never auto-actions or fans out publicly (`report.rs:1-6`; not stored as an event — `ingest.rs:1551-1570`).

**BR-MOD-31 — Reports stay submittable while timed out and in the post-ban window.** 1984 is dispatched at `ingest.rs:1562`, before the restriction gate at `:1613`. Documented as intentional: users must be able to signal abuse during a write-block, and a banned actor in a missed-disconnect window is tolerated because reports are non-actioning and mod-only (`ingest.rs:1551-1559`).

**BR-MOD-32 — Exactly one `p` tag, at most one of `e` or `x`, never both.** `report.rs:122-142`. The `p` tag is mandatory even for `e`/`x` targets (NIP-56 shape) but its value is **not trusted** for authorship — the annotation says e-target author truth comes from the stored row (`report.rs:106-107`) and blob authorship is not inserted at all (`:112-113`).

**BR-MOD-33 — `e` targets must resolve in-tenant or the report is rejected.** `get_event_by_id(tenant.community(), …)` ⇒ `invalid: report target event not found` (`report.rs:55-60`); never a cross-tenant search (`report.rs:11-14`).

**BR-MOD-34 — `x` blob targets must have a tenant-scoped sidecar.** `media_storage.get_sidecar(tenant, sha_hex)` (`report.rs:66-71`), so a bare SHA-256 shared across tenants grants no cross-tenant visibility (`report.rs:15-18`). Known limitation: all lookup failures — including transient storage errors — collapse to `invalid: report target blob not found` (`report.rs:67-70`).

**BR-MOD-35 — `channel_id` is inferred only from an `e` target.** `Blob` and `Pubkey` targets store `channel_id = None` (`report.rs:72`, `:74`), which means a pubkey-only report is never visible in a channel-scoped queue view.

**BR-MOD-36 — Report type must be one of 7 NIP-56 values, read from the *target tag's third element*.** `REPORT_TYPES` (`report.rs:29-37`); extracted at `report.rs:186` and validated at `:200-209`. Note the type comes from the `e`/`x`/`p` tag itself, not a separate tag.

**BR-MOD-37 — Report insertion is idempotent on the signed event id.** `report_event_id` is the per-community idempotency key (`buzz-db/src/moderation.rs:39`, `:170-171`), so re-submitting the same signed 1984 is a no-op returning the existing row.

**BR-MOD-38 — 9044 vocabulary is re-validated server-side.** Status ∈ `{resolved, dismissed}` (`moderation_commands.rs:380-385`); action ∈ `{delete, kick, ban, timeout, dismiss, escalate}` (`:386-392`); `dismiss` ⇔ `dismissed` is an exclusive-or check (`:393-397`). In-code rationale: the SDK validates at build time but the relay must not trust the client (`:377-379`).

**BR-MOD-39 — 9044 records a *decision*, not an *enforcement*.** `resolve:*` prefixed audit actions distinguish the moderator's one-click resolution from the paired 9040/9042 that writes the unprefixed enforcement row (`moderation_commands.rs:502-513`, rationale `:443-450`). `dismiss` → `dismiss_report`, `escalate` → `escalate` — both unprefixed so escalations stay queryable for platform safety (`:503-504`, doc `:52-54`).

**BR-MOD-40 — A closed report cannot be re-resolved, with an accepted residual race.** An early `report.status != "open"` check (`moderation_commands.rs:426-430`) prevents an orphan audit row; the real guard is the DB's `WHERE status='open'` inside `resolve_moderation_report` (`:461-468`). The comment concedes a window where the row flips between read and write, yielding an audit row plus a failed resolve — explicitly tolerated (`moderation_commands.rs:419-425`).

**Additional 9044 facts:** the audit row is written **before** the resolve (`:453-460` then `:461`), so a failed resolve leaves an audit row; and the audit row inherits the *report's* target (`:434-439`), with `ReportTarget::Blob` mapping to `(None, None)` — a blob resolution audits with no target at all.

---

#### F. Notice-delivery rules (BR-MOD-41 … BR-MOD-46)

**BR-MOD-41 — Notice delivery is best-effort and never fails the command.** All three call sites log and continue (`moderation_commands.rs:204-220`, `:309-321`, `:481-495`), with in-code rationale that the enforcement has already landed and been audited (`:214-216`).

**BR-MOD-42 — One DM thread per (community, user), created on first use.** `open_dm` is participant-hash idempotent (`moderation_notices.rs:97-107`).

**BR-MOD-43 — A hidden moderation thread is force-resurfaced.** `unhide_dm` is called on every send because `open_dm` only clears `hidden_at` for `created_by` (the relay key) — otherwise a user who hid the thread would never see a later ban notice (`moderation_notices.rs:121-129`).

**BR-MOD-44 — Idempotency is crash-safe but not concurrency-safe.** `notice_already_sent` is a query-then-insert keyed on the `moderation_source` tag (`moderation_notices.rs:227-252`); the comment states two simultaneous deliveries for the same source can both miss the pre-query and that hard serialization is a noted follow-up (`:132-138`). The scan limit is pinned to the query clamp of 1000 rather than 1 or the default 100, because matching happens post-query in Rust (`:222-226`).

**BR-MOD-45 — Discovery and profile are re-emitted on every send, deliberately.** Not gated on `was_created`, because a first-delivery discovery failure would otherwise leave the thread permanently undiscoverable (`moderation_notices.rs:141-151`). The kind:0 profile failure is downgraded to a warning (`:152-154`) while discovery failure is `?`-propagated (`:155`) — so a discovery failure *does* abort the notice.

**BR-MOD-46 — Privacy invariant: notices never name reporters or quote report notes.** Bodies are built only from status/summary/`public_reason` (`moderation_notices.rs:267-306`, invariant `:268-272`, module doc `:20-23`). The reporter-facing summary defaults to a generic sentence when no `reason` tag is present (`moderation_commands.rs:477-481`). Note that `reason` from a 9044 tag is relayed verbatim to the reporter (`:479`) — the doc requires it be "safe for the reporter's eyes" (`moderation_commands.rs:44-47`) but nothing sanitizes it. A 32-byte recipient length check guards the DM (`moderation_notices.rs:82-87`) and a self-DM to the relay key is silently skipped (`:94-96`).

---

#### G. Relay-membership admin rules (BR-MOD-47 … BR-MOD-51)

**BR-MOD-47 — Permission matrix.** 9030 add / 9031 remove / 9033 workspace-profile require `admin`|`owner`; 9032 change-role requires `owner` (`relay_admin.rs:20-29` doc table; enforced `:148`, `:177`, `:227`, `:286`).

**BR-MOD-48 — 9030 is silently idempotent and never overwrites a role.** `add_relay_member` returns `was_inserted`; an existing member at any role is a no-op with `Ok(())` (`relay_admin.rs:194-212`, doc `:194-196`). NIP-43 announcements are suppressed on a no-op re-add to avoid spurious kind:8000 events (`:212-221`).

**BR-MOD-49 — 9033 is handled before p-tag extraction because it targets the relay, not a member.** `relay_admin.rs:144-166`. Missing or empty `icon` tag clears the icon (`:152`, `:157-161`).

**BR-MOD-50 — Workspace-icon policy.** Empty ⇒ clear; no control or whitespace characters; `data:image/*` up to 98,304 bytes; otherwise must start `https://` or `http://` and be ≤ 2048 bytes (`relay_admin.rs:69-94`). **`http://` is accepted** (`:83`), so an icon can be a plaintext URL. `data:text/html` and `javascript:` are rejected (test `:441-444`).

**BR-MOD-51 — NIP-43 publication failures never fail the command.** All 5 publish calls warn-and-continue (`relay_admin.rs:214-220`, `:274-279`, `:334-336`), so `relay_members` and the event-backed membership view can diverge silently.

---

#### H. Community-provisioning rules (BR-MOD-52 … BR-MOD-58)

**BR-MOD-52 — Provisioning authority is deployment-level, not community-level.** The gate is the `RELAY_OPERATOR_PUBKEYS` allowlist, deliberately **not** a `relay_members` lookup, because the effect of the operation is the creation of tenancy itself (`community_provisioning.rs:3-14`, `:255-266`).

**BR-MOD-53 — Empty allowlist fails closed.** An empty `relay_operator_pubkeys` (the default, `config.rs:962-963`) rejects everyone (`community_provisioning.rs:258-266`). Config additionally refuses to boot if the allowlist is set without `RELAY_OPERATOR_API_ORIGIN` (`config.rs:577-580`).

**BR-MOD-54 — `create_only=true` requires an owner and refuses an existing host.** `initial_owner_pubkey is required when create_only is true` (`:281-283`); `HostExists` ⇒ `community already exists`, `LimitReached` ⇒ `limit_reached: …` (`:290-299`).

**BR-MOD-55 — Legacy convergence mode can *rotate an existing community's owner*.** Without `create_only`, `ensure_configured_community` + `bootstrap_owner` runs even when the community already existed, demoting any previous owner to admin (`community_provisioning.rs:236-247`, `:321-334`). Documented as the price of retry convergence and as the reason the allowlist is "deployment-root authority, not create-only authority" (`:243-247`). Clients acting for end users are told to use `create_only` (`:317-320`) — this is guidance, not enforcement.

**BR-MOD-56 — Create requires a pre-normalized canonical host; availability checks do not.** `validate_host` insists `normalize_host(host) == host` so the stored value is byte-identical to request-time lookup (`community_provisioning.rs:75-100`). `normalize_candidate_host` normalizes first, accepting uppercase/trailing-dot/default-port, for read-only endpoints (`:174-207`).

**BR-MOD-57 — Host must be a canonical bare authority.** No scheme/path/query/fragment/userinfo (`:106-118`); parsed via `Url::parse("http://{authority}/")` and required to round-trip byte-identically (`:120-146`); DNS labels are alphanumeric-or-hyphen, non-empty, ≤63, not hyphen-terminal, total ≤253 (`:150-172`). Underscores are rejected (test `:409`), IPv6 brackets accepted (test `:421-424`).

**BR-MOD-58 — Membership snapshot publication is best-effort and only when membership is required.** Skipped entirely if `!require_relay_membership` (`:210-217`); a failure warns because provisioning is already committed and turning a stored success into an HTTP failure would make create-only retries report a misleading conflict (`:202-208`, `:218-227`).

---

#### I. Push-lease rules (BR-MOD-59 … BR-MOD-72)

**BR-MOD-59 — Lease acceptance is disabled without a gateway URL.** First check in `accept`: `push not supported` (`push_lease.rs:480-482`). Because the URL defaults to `https://push.buzz.xyz/v1/deliveries/apns` (`config.rs:339`, `:752-758`), leases are accepted **by default** on any deployment that does not explicitly set the var to empty.

**BR-MOD-60 — Only 4 public tags are permitted and duplicates are fatal.** `d`, `expiration`, `exec`, `alt`; every other tag ⇒ `unexpected public tag: {name}`; a repeat ⇒ `duplicate {name} tag`; any tag with ≠2 elements ⇒ rejected (`push_lease.rs:106-126`). This is the metadata-leak guard: nothing about subscriptions is public.

**BR-MOD-61 — Lease TTL is 30 days max, 120 s skew tolerance.** `expiration <= now - 120` ⇒ `lease already expired`; `expiration > now + 2_592_000` ⇒ `lease ttl too long` (`push_lease.rs:138-143`, consts `:475-476`).

**BR-MOD-62 — The executor key id must match deployment config exactly.** `unknown executor key` (`push_lease.rs:484-486`), default `relay-v1` (`config.rs:745-746`).

**BR-MOD-63 — Plaintext is NIP-44-decrypted with the relay's own secret key.** `nip44::decrypt(state.relay_keypair.secret_key(), &event.pubkey, &event.content)` (`push_lease.rs:487-492`) — any failure collapses to `invalid encrypted content`.

**BR-MOD-64 — Origin must equal the server-resolved tenant authority, never the decrypted claim.** `canonical_origin(config.relay_url, tenant.host())` derives the scheme from the relay URL and the authority from the tenant (`push_lease.rs:585-596`); `body.origin != expected` ⇒ `origin mismatch` (`:200-202`). Module doc pins that tenant selection never comes from the decrypted `origin` (`:3-5`).

**BR-MOD-65 — Generation must be a positive JSON-safe integer and strictly advance.** `1..=2^53-1` at validation (`push_lease.rs:198-200`); the strict-advance watermark is enforced in the DB, surfacing as `AcceptLeaseOutcome::StaleGeneration` → `invalid: stale generation` (`ingest.rs:2165-2167`).

**BR-MOD-66 — Duplicate JSON keys are rejected at every depth.** Custom `DeserializeSeed` (`push_lease.rs:383-455`); pinned by a test that covers both top-level and nested-filter duplicates (`:637-649`).

**BR-MOD-67 — An inactive lease must use the minimal schema.** Only `v`/`origin`/`generation`/`active` are permitted; presence of `app_profile`/`transport`/`endpoint`/`subscriptions` ⇒ `inactive lease must use minimal schema` (`push_lease.rs:210-218`), and the key-set check rejects them earlier as `unknown field: {key}` (`:159-190`; test `:651-663`).

**BR-MOD-68 — Only 2 app profiles and 3 classes are advertised, and transport must match the profile.** Profiles `buzz-ios-production` / `buzz-ios-sandbox`, both `apns` (`push_lease.rs:503-512`); classes `silent`/`default`/`time_sensitive` (`:509`). `transport mismatch` at `:230-232`.

**BR-MOD-69 — The `urgent` class is structurally unreachable.** `URGENT_KINDS` is empty (`push_lease.rs:16`) and `"urgent"` is absent from `supported_classes` (`:509`), so `class not supported` (`:245-247`) fires before the urgent-kind confinement check at `:281-283` can ever run. `class_rank` still ranks `urgent` highest in both copies (`:582`, `push_runtime.rs:574`), and NIP-11 advertises `urgent_kinds: []` (`nip11.rs:209`).

**BR-MOD-70 — Every positive lease filter must be narrowed and self-scoped.** A positive filter needs at least one of `authors`/`#p`/`#h` ⇒ else `lease filter not narrowed` (`push_lease.rs:292-295`); every `#p` value must equal the lease author ⇒ `p-tag must be self` (`:298-303`). `ignore` filters are exempt from narrowing and from urgent confinement because they can only subtract (`:251-255`, `:263` with `require_narrowing=false`).

**BR-MOD-71 — Push-eligible kinds are a closed set of 5.** `PUSH_KINDS = [7, 9, 1059, 40007, 46010]` (`push_lease.rs:15`); any other kind ⇒ `kind not push-eligible` (`:277-280`). A unit test asserts the DB trigger allowlist in `migrations/0018_push_match_queue.sql` uses this exact list (`push_lease.rs:696-710`) — a genuinely strong cross-artifact invariant.

**BR-MOD-72 — Quotas are hard-coded, not configurable.** 16 active leases per author (`push_lease.rs:479`), 16 subscriptions, 16 kinds, 20 authors, 50 `#h`, 20 generic tag values, 8 ignore filters, 4096-byte endpoint, 512-byte strings (`:513-521`), 64 KiB ciphertext (`:477`), 32 KiB plaintext (`:478`). Note `max_h = 50` is deliberately independent of `max_tag_values = 20` (test `:734-741`).

**Lease replacement rules** (enforced in the DB transaction, surfaced by ingest): NIP-01 addressable ordering (`StaleEvent`), generation watermark (`StaleGeneration`), one active address per endpoint tuple (`EndpointAlreadyLeased`), per-author quota (`LeaseQuotaExceeded`), source-event uniqueness (`SourceEventCollision`) — `ingest.rs:2159-2186`, outcomes at `buzz-db/src/push.rs:190-208`.

---

#### J. Push matcher / delivery rules (BR-MOD-73 … BR-MOD-83)

**BR-MOD-73 — Batch claim with a 30 s lease and a 64-job ceiling.** `CLAIM_SECS = 30`, `MATCH_BATCH_LIMIT = 64`, bounded under `get_events_by_ids`' 500-id contract (`push_runtime.rs:15`, `:18-20`, claim `:69-74`).

**BR-MOD-74 — Idle backoff 250 ms → 2 s, reset on any work.** `IDLE_POLL_FLOOR`/`IDLE_POLL_CEILING` (`push_runtime.rs:22-25`); doubling at `:80-81`, reset at `:76`. Delivery worker uses its own 500 ms floor with the same 2 s ceiling (`:317`, `:341-345`).

**BR-MOD-75 — Poison-job reaping runs off the claim path, every 30 s.** `REAP_INTERVAL` (`push_runtime.rs:26-29`), because the reap scan is not served by the due partial index and would slow claims exactly when a backlog needs them fastest (`:26-28`); executed at `:60-68`.

**BR-MOD-76 — Whole-batch retry when shared context fails to load.** No job was evaluated, so all ids are released with a 2 s delay (`push_runtime.rs:127-149`).

**BR-MOD-77 — 8 match attempts then discard, not infinite retry.** `job.attempt >= buzz_db::push::MAX_MATCH_ATTEMPTS` (=8, `buzz-db/src/push.rs:70`) ⇒ marked complete so a poison event cannot retry forever or pin outbox retention (`push_runtime.rs:168-176`).

**BR-MOD-78 — Match evaluation is pure; the wake enqueue is one transaction per batch.** `match_job` does no DB access (`push_runtime.rs:216-290`); all wakes flush in a single `enqueue_push_wakes` (`:184-198`). Enqueue failure sends only the contributing jobs back for an idempotent rematch, absorbed by the outbox dedup key (`:179-183`).

**BR-MOD-79 — Three authorization layers per (event, lease) pair.** `reader_authorized_for_event` on the lease author (`push_runtime.rs:222-225`); channel membership from the pre-loaded `(channel, author)` pair set (`:226-233`); and per-filter `push_filter_authorized_for_event` (`:250`).

**BR-MOD-80 — Gift wraps (kind 1059) may only match a lease whose filter is self-`#p` AND whose event is addressed to that author.** `push_runtime.rs:292-310` — a match-time counterpart of REQ's `#p` gate, because 1059 is globally stored and leaks recipient activity through wake timing (`:287-290`). All non-1059 kinds pass this check unconditionally (`:296-298`). Tested at `:580-620`.

**BR-MOD-81 — Wake class is the max across matching subscriptions; suppression and ignore-filters subtract.** Highest `class_rank` wins (`push_runtime.rs:268-270`); a matching `ignore` filter or a `p` tag count exceeding `suppress.p_tags_max` skips the subscription (`:255-267`).

**BR-MOD-82 — Wake deadline is `min(lease expiry, event created_at + 3600 s)`, and an already-expired wake is dropped.** `EVENT_USEFUL_SECS = 3600` (`push_runtime.rs:16`); computed `:272-276`.

**BR-MOD-83 — Delivery revalidates three times, with membership sandwiched between two generation fences.** `revalidate_push_wake` (`push_runtime.rs:354-370`) → channel membership (`:372-402`) → `revalidate_push_wake` again as the final DB op before transport (`:403-420`), because membership I/O can race lease replacement (`:403-404`). `Suppressed` ⇒ `fail_push_wake`; a revalidation *error* returns without touching the row, letting the claim lease expire.

**Retry schedule:** `retry_or_fail` backs off `delay * 2^(attempt-1)` clamped to `2^6`, failing permanently at `MAX_ATTEMPTS = 8` (`push_runtime.rs:531-550`, const `:17`).

**Endpoint invalidation is generation-fenced:** a 410 only disables the endpoint when the gateway-reported generation equals the wake's (`push_runtime.rs:452-466`) — so a stale 410 cannot kill a freshly rotated lease.

**404 after the first attempt is treated as success.** A timed-out terminal attempt burns the stable request id; its replay is indistinguishable from an invalid-grant 404, and sending a fresh id would double-deliver (`push_runtime.rs:488-497`). This deliberately swallows genuine 404s on retry.

---

#### K. Identity-archive consent rules (BR-MOD-84 … BR-MOD-91)

**BR-MOD-84 — Three consent paths, evaluated in a fixed order.** Self (`actor == target`) → community `owner`|`admin` role → cryptographic NIP-OA owner (`identity_archive.rs:228-251`). Note the naming inversion: a community owner/admin yields `ConsentPath::Admin` (`:245-247`); `ConsentPath::Owner` means *key owner of the target agent* (`:250`).

**BR-MOD-85 — The scope gate is deliberately weak because the handler owns authorization.** 9035/9036 require only `Scope::UsersWrite`, not `AdminUsers`, since self and owner-of-agent paths are open to ordinary users (`ingest.rs:258-266`).

**BR-MOD-86 — Owner consent requires a valid NIP-OA auth tag on the *request*, owned by the request signer.** Exactly one 4-element `auth` tag (`identity_archive.rs:300-318`); `verify_auth_tag(tag, target_pubkey)` must return the actor (`:260-266`).

**BR-MOD-87 — Owner consent additionally requires the target's *live* kind:0 to still attest the same owner.** A global-only kind:0 query for the target (`identity_archive.rs:270-284`), author cross-check (`:286-289`), and a second `verify_auth_tag` whose owner must equal the actor ⇒ else `live kind:0 no longer attests to request signer` (`:290-295`). This is the revocation path: replacing your kind:0 auth tag immediately invalidates outstanding owner-signed archive requests. Covered by the module's one integration test (`:515-578`).

**BR-MOD-88 — Auth-tag `created_at<`/`created_at>` conditions are enforced against the request's own timestamp.** `identity_archive.rs:328-355`; unknown clauses such as `kind=1` are ignored (test `:424-431`).

**BR-MOD-89 — Exactly one NIP-70 `["-"]` protected tag is mandatory.** Zero or ≥2 ⇒ rejected (`identity_archive.rs:156-167`).

**BR-MOD-90 — Exactly one valid `p` tag; `replaced-by` must be lowercase 64-hex, distinct from the target, and singular.** `extract_single_p_tag_hex` returns `None` on a second `p` tag (`:170-187`); `replaced-by` rules at `:200-226`. `replaced-by` on a 9036 unarchive is rejected (`:60-62`).

**BR-MOD-91 — Archiving is idempotent and event emission is gated on actual change.** `archive` uses `ON CONFLICT DO NOTHING` and does not mutate an existing row (`buzz-db/src/archived_identities.rs:66`); `!changed` ⇒ early `Ok(())` with no delta or snapshot published (`identity_archive.rs:100-102`). Delta/snapshot publish failures only warn (`:130-136`), so `archived_identities` can diverge from the 8002/8003/13535 event view.

**Archiving is explicitly not a ban** — it does not affect membership, relay access, or repository permissions (`buzz-db/src/archived_identities.rs:3-6`).

---

#### L. Product-feedback rules (BR-MOD-92 … BR-MOD-94)

**BR-MOD-92 — Feedback is sidecarred out of the event store entirely.** No `events` row, no fan-out (`ingest.rs:1536-1552`).

**BR-MOD-93 — At most one category from a closed 3-value set; category is optional.** `["bug","praise","needs-work"]` (`product_feedback.rs:11`, `:78-92`).

**BR-MOD-94 — Body must be non-blank and ≤32 KiB; the full tag array is stored and capped at 64 KiB.** `:94-104`, `:70-76`. `imeta` tags, if present, are URL-validated against the tenant media base and their blobs verified before insert (`:26-36`).

---

#### M. Storage-sweep rules (BR-MOD-95 … BR-MOD-101)

**BR-MOD-95 — Leader-only, single-flight, never awaited by the tick.** Both entry points are called from the leader-only branch of `run_usage_metrics_tick` (`main.rs:1423-1430`); `maybe_spawn_sweep` returns immediately if `in_flight` is unfinished (`storage_sweep.rs:161-165`). This is the module's leader election — a real Postgres advisory lock via `buzz_db::UsageMetricsLeader` (`main.rs:1414-1421`).

**BR-MOD-96 — Harvest and spawn share one lock acquisition, by design.** Splitting them would let a tick observe a freshly-emptied `in_flight` from a harvest that has not yet updated `cached`, and spawn a redundant sweep (`storage_sweep.rs:143-149`).

**BR-MOD-97 — Cadence rule: spawn iff nothing in flight AND (never attempted OR last attempt failed OR cache older than `interval`).** `should_spawn` (`storage_sweep.rs:105-127`). A failed attempt respawns on the very next usage tick (default 300 s, `main.rs:1257-1262`) rather than waiting the full hour — documented as intentional because a permission failure costs one cheap LIST and the sweep self-heals when fixed (`storage_sweep.rs:89-103`).

**BR-MOD-98 — A warm cache survives later failures; a cold cache stays cold.** Failure increments `failures_total` and updates `last_attempt` but never clears `cached` (`storage_sweep.rs:175-184`); tests `:531-591`, `:493-528`.

**BR-MOD-99 — Task panic is counted as a failure, not propagated.** `JoinError` ⇒ `failures_total += 1`, `last_attempt = { ok: false, duration: ZERO }` (`storage_sweep.rs:194-202`).

**BR-MOD-100 — The kill switch suppresses *all* storage gauges including health ones.** `BUZZ_STORAGE_METRICS=off` ⇒ `run_storage_sweep_tick` returns before touching anything (`main.rs:1454-1456`), so a relay without `s3:ListBucket` can disable the feature cleanly (`storage_sweep.rs:42-45`).

**BR-MOD-101 — A completed-but-unharvested sweep publishes nothing.** Harvest only happens inside `maybe_spawn_sweep`, which a demoted leader stops calling — so its snapshot sits in `in_flight` forever (`storage_sweep.rs:643-670` test; rationale `:611-628`). Per-community series that disappear from the snapshot (removed, host-renamed, or scope-excluded) are explicitly zeroed rather than left stale (`storage_sweep.rs:333-345`; test `:857-1088`). Unmapped community UUIDs roll into `buzz_storage_unmapped_community_bytes` instead of a per-community series (`:318-323`, `:330`).

**Budget:** `interval` floored at 60 s (`storage_sweep.rs:59-60`), whole-attempt `timeout` (`:249-252`), cumulative `max_objects` cap checked before folding each page (`buzz-media/src/bucket_index.rs:392-395`), 1000 objects per LIST page (`main.rs:1466`). Every `SweepError` variant means "keep the old snapshot", never a partial one (`bucket_index.rs:337-338`).

---

#### N. Workflow-action authorization (BR-MOD-102 … BR-MOD-108)

**BR-MOD-102 — The run's community is authoritative; the tenant is never re-derived from config.** `lookup_community_host(community_id)` reads the host back for labelling only, and fails closed if the community no longer maps to a host (`workflow_sink.rs:190-210`). In-code rationale: re-deriving from `config.relay_url` would post a community-B workflow's output into the deployment default under N>1 (`:191-199`).

**BR-MOD-103 — Empty/whitespace-only text is refused.** `ActionSinkError::EmptyContent` (`workflow_sink.rs:212-214`).

**BR-MOD-104 — Channel must exist, be a canonical UUID, and not be archived.** `Uuid::parse_str` then canonicalized (`:217-220`); `ChannelNotFound` mapped from `DbError::ChannelNotFound|NotFound` (`:222-231`); `ChannelArchived` at `:232-236`.

**BR-MOD-105 — The workflow *owner* must be a member unless the channel is open.** `!is_member && channel.visibility != "open"` ⇒ `workflow owner does not have access to destination channel` (`workflow_sink.rs:243-251`). This is the only authorization on the action; the event is then signed by the relay key (`:304`) with a `p` tag attributing the owner (`:260-261`).

**BR-MOD-106 — Mention resolution is deliberately conservative and defines the wake contract.** Members only, exact display name, case-insensitive, greedy-longest non-overlapping, ambiguous names wake nobody (`workflow_sink.rs:22-43`, impl `:45-140`). Boundary rule: `@` must sit at start/whitespace/`(` and the name must not be followed by an alphanumeric or `_` (`:98-101`). Case folding is done on the fly in original-char coordinates because `İ`(U+0130) lowercases to two code points (`:78-96`) — regression-tested at `:490-503` and `:544-560`.

**BR-MOD-107 — Workflow messages are always top-level.** `depth: 0`, no parent, no root, `broadcast: false` (`workflow_sink.rs:318-335`). A workflow can never reply into a thread.

**BR-MOD-108 — `buzz:workflow` tag prevents recursive triggering; post-persist side effects only fire on real insert.** Tag at `workflow_sink.rs:264-266`; `if was_inserted` guard at `:349`. The `dispatch_persistent_event` result is discarded with `let _ =` (`:351`), so fan-out/search/audit failures are invisible.

> **Verified against ARCHITECTURE.md §9:** items 5 (`:826`, approval gates) and 6 (`:827`, `send_dm`/`set_channel_topic` `NotImplemented`) are accurate. From the sink side the gap is wider — `ActionSink` declares exactly one method (`buzz-workflow/src/action_sink.rs:44-64`), so only `send_message` is wired end-to-end, and `add_reaction` is a **third** broken action (POSTs to `/api/messages/{id}/reactions`, `buzz-workflow/src/executor.rs:886-888`, which is not registered in `router.rs`). ARCHITECTURE.md:541 lists `add_reaction` as a working action with no caveat.
