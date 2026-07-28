## Module: buzz-relay — event ingest & side effects (`crates/buzz-relay/src/handlers`)
### Aspect: Security

This is the relay's only write door. Everything below is verified against source, not
against the doc comments.

---

### 1. Headline findings

| # | Finding | Severity | Evidence |
|---|---|---|---|
| **S-01** | **The per-kind scope gate is inert in production.** Both transports construct `IngestAuth` with `Scope::all_known()` — all 16 scopes. So `!auth.scopes().contains(&required)` at `ingest.rs:1525` can never be true, and the entire 81-row scope mapping in `required_scope_for_kind` (`ingest.rs:198-306`) is documentation only. `AdminUsers` on 9030–9033 and `AdminChannels` on 9000/9001/9008 provide **zero** enforcement; real authorization for those kinds lives in `relay_admin.rs` and `validate_admin_event`. | **High** (defence-in-depth absent, not directly exploitable) | `buzz-auth/src/lib.rs:133-141`; `api/bridge.rs:826-829`; `buzz-auth/src/scope.rs:68-87` |
| **S-02** | **All channel-scoped-token logic is dead.** `AuthContext.channel_ids` is hard-coded `None` on the WS path and absent from `IngestAuth::Http`, so `auth.channel_ids()` never returns `Some`. Four gates never fire: `ingest.rs:1512` (relay-admin vs channel token), `:1520` (28936 vs channel token), `:1721-1728` (channel token publishing a global event), and `check_token_channel_access` (`:525-532`). The code's own doc admits it (`ingest.rs:113-115`). | **Medium** (dead code masquerading as a control) | `buzz-auth/src/lib.rs:138`; `ingest.rs:117-126` |
| **S-03** | **Privileged writes are not audited.** `buzz-audit` declares 11 actions; production emits exactly 2: `EventCreated` (`handlers/event.rs:560`) and `MediaUploaded`. Every privileged mutation this group performs — role grants, member removal, channel deletion, visibility flips, relay-membership changes, identity archival — produces **only** a generic `EventCreated` row, and the 4 kinds that are *never stored* (9030–9033) produce **no audit row at all**. `EventDeleted`, `ChannelCreated`, `ChannelUpdated`, `ChannelDeleted`, `MemberAdded`, `MemberRemoved`, `AuthSuccess`, `AuthFailure`, `RateLimitExceeded` have zero producers workspace-wide. | **High** | `buzz-audit/src/action.rs:8-31`; grep for `AuditAction::*` outside `crates/buzz-audit/` |
| **S-04** | **Relay-admin commands (9030–9033) leave no durable trace whatsoever.** They are handled directly and never stored (`ingest.rs:1808-1818`), so no `events` row exists → `dispatch_persistent_event` is never called → no `EventCreated` audit entry. Membership and role changes at the *relay* level are the highest-privilege operation in the system and are the only privileged path with neither an event row nor an audit row. Compare 9035/9036, which deliberately *do* fall through to storage precisely so the audit reference resolves (`ingest.rs:1914-1919`). | **High** | `ingest.rs:1808-1818` vs `:1915-1919` |
| **S-05** | **Community moderation (9040–9044) has no `buzz-audit` entry either.** Also never stored (`ingest.rs:1579-1588`). The module header's "audit row" (`moderation_commands.rs:11-14`) refers to a `moderation_actions` domain table, not the hash-chained `buzz-audit` log — so ban/timeout/unban are outside the tamper-evident chain. | **Medium** | `ingest.rs:1579-1588`; `moderation_commands.rs:11-14`; no `buzz_audit` reference in that file |
| **S-06** | **Side-effect failure is invisible to the client and to the audit log.** A rejected `add_member`, a failed `update_channel`, a failed `remove_member` — all leave the command event committed, fanned out, and reported as `accepted: true`. Clients render the change; the DB never made it. | **High** | `ingest.rs:2434-2441` then `:2489` |
| **S-07** | **9000 on an *open* channel gets no relay-layer authorization.** `validate_admin_event`'s elevated-role check sits inside `if channel.visibility == "private"` (`side_effects.rs:296-338`). On an open channel, any authenticated principal may submit a 9000 with `role=owner`. The escalation is blocked only by `buzz_db::channel::add_member`'s granter check (`buzz-db/src/channel.rs:391-410`). Not currently exploitable, but the relay-layer gate reads as if it were the control and is not, and a DB refusal manifests as S-06 (event stored, no change, success reported). | **Medium** | `side_effects.rs:296-338` vs `buzz-db/src/channel.rs:391-410` |
| **S-08** | **`is_global_only_kind` does not neutralize a stray `h` tag on the read side.** `channel_id` is nulled at write (`ingest.rs:1709-1711`), but the signed tag remains and `filter.rs` treats explicit `h` tags as authoritative, so a global-only event can still match `#h` queries for a channel the author cannot read. Self-documented as a known limitation at `ingest.rs:369-377`. Affects all **44** global-only kinds. | **Medium** | `ingest.rs:369-377` |
| **S-09** | **4 `expect()` calls in the ingest hot path.** `ingest.rs:1998`, `:2000` (kind:44200 `p` tag / hex decode), `:2338`, `:2344` (`channel_id.expect("reaction path has channel")`). Each is justified by an earlier check in the same function, but all four are ~340–570 lines away from the invariant they rely on, and a panic in `ingest_event_inner` aborts the request path. AGENTS.md forbids new `unwrap()`/`expect()` in production paths. | **Medium** | `ingest.rs:1998`, `:2000`, `:2338`, `:2344` |
| **S-10** | **2 `unwrap()` calls in `validate_admin_event`.** `role_str.parse().unwrap()` (`side_effects.rs:311`) and `actor_member.unwrap()` (`side_effects.rs:314`) — both inside the 9000 private-channel branch. The parse is pre-validated at `:288-291` and the `Option` is checked at `:299-303`, so both are currently sound, but they sit in an authorization path. | **Medium** | `side_effects.rs:311`, `:314` |

---

### 2. Tenancy enforcement — full write-path audit

Every DB call in the group takes `tenant.community()`, resolved server-side from the
request host. **No write path derives the community from event content.** Verified
call-by-call:

| Write path | Community argument | Site |
|---|---|---|
| generic insert | `tenant.community()` | `ingest.rs:2394` |
| replaceable / NIP-33 replace | `tenant.community()` | `ingest.rs:2371`, `:2385` |
| reaction insert | `tenant.community()` | `ingest.rs:2298` |
| channel pre-create (9007) | `tenant.community()` | `ingest.rs:2103` |
| compensating soft-delete | `tenant.community()` | `ingest.rs:2408` |
| restriction lookup | `tenant.community()` | `ingest.rs:1616` |
| membership check | `tenant.community()` | `ingest.rs:501` (via `is_member_cached`) |
| relay-member removal | `tenant.community()` | `ingest.rs:1857` |
| command-event raw INSERT | `tenant.community().as_uuid()` bound as `$1` | `command_executor.rs:207` |
| command coordinate lock + SELECT + supersede | `tenant.community().as_uuid()` | `command_executor.rs:157-195` |
| all side-effect writes | `tenant.community()` | `side_effects.rs` — 60+ sites, no exceptions found |

Two places explicitly refuse to re-derive the community from client input, with the
attack spelled out in the comment:
- 30620: "`community_of_channel(channel_id)` is ambiguous when the same channel UUID exists
  in two communities and could mint the workflow under the wrong tenant"
  (`command_executor.rs:759-771`), followed by a community-scoped `get_channel` to fail
  closed;
- 46020: "a bare-id lookup could load B's workflow and then satisfy the membership check
  below against B's colliding channel, letting B trigger A's workflow"
  (`command_executor.rs:826-831`).

The conformance trace records `claimed_community_from_event(&event)` alongside every write
and auth verdict, but as a **claim only** — the verdict basis is always
`tenant.community()`, stated at `ingest.rs:1777-1782`.

`ThreadedChannelVisibility` (`ingest.rs:1750-1760`) bundles the resolved visibility with
the `(community_id, channel_id)` it was resolved under, so fan-out cannot apply it to the
wrong pair. `None` means "fan-out does its own fail-closed lookup", never "assume open" —
documented as fence 1 at `ingest.rs:1742-1749`.

---

### 3. Authorization matrix — who may write what

| Kind(s) | Required identity | Enforced where | Bypass paths |
|---|---|---|---|
| 0, 1, 3, 10000–10003, 10030, 10100, 30000, 30003, 30023, 30030, 30078, 30174–30177, 30300, 30315, 30350 | self (pubkey-match, `ingest.rs:1499`) | envelope | — |
| 1059 | **any authenticated principal**, pubkey-match deliberately waived | `ingest.rs:1498-1503` | by design (NIP-59 ephemeral keys) |
| 9, 40002, 40004–40008, 40100, 45001, 45003, 48100–48106 | channel member, or any principal if the channel is `open` | `check_channel_membership` `ingest.rs:493-523` | — |
| 40003 | target's effective author, or the NIP-OA owner of the agent that authored it; author path re-gated on membership | `validate_edit_ownership` `ingest.rs:763-842` | **skips** the generic membership gate (`ingest.rs:1772`) |
| 45002 | channel member + target is a 45001/45003 in the same channel | `ingest.rs:844-894` | — |
| 5 | target's effective author, or the NIP-OA owner of that author | `validate_standard_deletion_event` `side_effects.rs:179-238` | — |
| 7 | channel member of the *target's* channel (channel derived from target, `ingest.rs:1645`) | generic gate | — |
| 9000 | private channel: existing member; elevated role needs owner/admin. **Open channel: nothing** | `side_effects.rs:284-372` | S-07 |
| 9001 | self (not last owner), or owner/admin, or member + NIP-OA owner of target | `side_effects.rs:373-409` | non-members refused even for their own agent (`:365-367`) |
| 9002 | privileged tags: owner/admin, or owner of any active owner-role agent. topic/purpose: any member | `side_effects.rs:410-624` | **skips** the generic gate (`ingest.rs:1773`) |
| 9005 | target author (unless moderation metadata present), or owner/admin, or NIP-OA owner of the agent author | `side_effects.rs:508-624` | **skips** the generic gate (`ingest.rs:1774`) |
| 9007 | any authenticated principal — creator becomes owner | `ingest.rs:2029-2132` | **skips** the generic gate (no channel yet) |
| 9008 | owner, or owner of any active owner-role agent | `side_effects.rs:625-644` | **skips** the generic gate (`ingest.rs:1775`) |
| 9021 | any authenticated principal, `open` channels only | `ingest.rs:2134-2154` + `side_effects.rs:1846-1854` | **skips** the generic gate by design |
| 9022 | active member, not last owner | `side_effects.rs:645-663` | — |
| 9030–9033 | relay owner/admin — decided entirely inside `relay_admin.rs` | out of module | scope gate is inert (S-01) |
| 9035/9036 | self, admin role, or owner-via-live-kind:0 — decided in `identity_archive.rs`. Scope deliberately `UsersWrite` not `AdminUsers`, with the rationale at `ingest.rs:265-273` | out of module | — |
| 9040–9044 | moderator capability, decided in `moderation_commands.rs`; ban re-checked there (`:103-108`) | out of module | **exempt** from the ingest ban/timeout gate (`ingest.rs:1613`) |
| 28936 | self; relay owner refused | `ingest.rs:1820-1902` | — |
| 30617 | self; name must be unowned or self-owned; per-pubkey quota | `side_effects.rs:2464-2510` | — |
| 30618 | any principal with `ReposWrite` (i.e. everyone, S-01) — no repo-ownership check | `ingest.rs:294` + generic store | can publish a competing 30618 under its own `d` coordinate |
| 1617–1621, 1630–1633 | any authenticated principal; no repo relationship checked | generic store | — |
| 41010 | self | `command_executor.rs:310` | — |
| 41011/41012 | member of the DM; channel must be `channel_type == "dm"` | `command_executor.rs:518`, `:611` | — |
| 30620 | channel member; on update, must match existing owner **and** channel | `command_executor.rs:713-735` | — |
| 46020 | workflow **owner** only — "channel membership alone is insufficient: a member could otherwise invoke another user's webhook or message actions" (`command_executor.rs:838-840`) | `command_executor.rs:842-847` | — |
| 46030/46031 | `approver_spec` = `""`/`"any"` (anyone) or an exact pubkey; **fails closed** on role-based or unrecognised specs | `check_approver_spec` `command_executor.rs:961-984` | ⚠ `""`/`"any"` means *any authenticated principal in the community* may approve a workflow gate — including one they have no relationship to. That is the documented design, but it is a broad default. |
| 44200 | `event.pubkey` must be a registered agent and `p` must be its DB-registered owner | `ingest.rs:1981-2016` | — |
| 1984, 42000 | any authenticated principal | `ingest.rs:1540`, `:1561` | 1984 is deliberately allowed while timed out and in the post-ban window (`ingest.rs:1551-1559`) |

---

### 4. Tag injection / spoofing resistance

| Vector | Defence | Site |
|---|---|---|
| Forged author via `event.pubkey` | pubkey must equal the authenticated principal (except 1059) | `ingest.rs:1499` |
| Forged author via relay-signed attribution | `effective_message_author` only trusts `actor`/`p` tags **when `event.pubkey == relay_pubkey`** — a user-signed event can never claim a different author | `ingest.rs:729-761`; duplicate impl `side_effects.rs:2271-2298` |
| Stray `h` tag on a global kind | `channel_id` forced `None` (write side only — see S-08) | `ingest.rs:1709-1711` |
| Missing `h` on a channel kind | hard reject | `ingest.rs:1713-1717` |
| Cross-channel edit (h=A, e=event-in-B) | `validate_edit_ownership` compares the edit's channel to the target's | `ingest.rs:790-807` |
| Cross-channel delete (h=A, e=event-in-B) | checked twice — pre-storage `validate_admin_event` (`side_effects.rs:560-573`) and again in the side effect (`:1583-1598`), with the attack named in the comment | `side_effects.rs:1580-1583` |
| Cross-channel vote | `validate_forum_vote_target` compares channels | `ingest.rs:880-892` |
| Cross-channel reply | parent's `channel_id` must equal the reply's; a parent with no channel is refused | `ingest.rs:620-628` |
| Multi-target deletion ambiguity | `e_count + a_count` must be exactly 1; malformed `e` tags **count** | `ingest.rs:1946-1958`, `count_e_tags` `:719-726`, regression test `:3033-3043` |
| Malformed `e` + valid `a` routing to the coordinate soft-delete | routing keys on `has_e_tag` (presence, not validity) | `side_effects.rs:2296-2302` |
| Case-folding attack on `#p` visibility | 30174 and 44200 both require **lowercase** hex in `d`/`p`/`agent`, because an uppercase head wins NIP-33 replacement then becomes invisible to lowercase `#p` readers | `ingest.rs:1004-1019`, `:1178-1214`; regression test `:2612-2620` |
| Empty-`d` slot collapse | 30175 requires a non-empty slug (`ingest.rs:1027-1082`); 30300 requires exactly one non-empty `d` (`:1287-1296`). ⚠ 30176/30177 have no such guard | `ingest.rs:1027`, `:1287` |
| Garbage ciphertext superseding a valid head | NIP-44 v2 shape check (base64 alphabet, length ≡ 0 mod 4, decoded ≥99 B, `0x02` prefix) on 30174 and 44200 | `ingest.rs:1084-1149` |
| Channel identity smuggled in a 44200 envelope | an `h` tag is a hard reject | `ingest.rs:1170-1174` |
| `imeta` path traversal via `filename` | rejects `/`, `\`, control chars; explicitly "display-only and must never influence storage keys, which are content-addressed" | `imeta.rs:138-155` |
| `imeta` pointing at an external host | `is_local_media_url` requires `/media/` or the tenant's own base URL; rejects `?`, `#`, `%` | `imeta.rs:373-418`; test `:456-462` |
| `imeta` claiming a blob it does not own | sidecar must exist and MIME/size/duration must match | `imeta.rs:246-278` |
| `imeta` MIME deny-list bypass | `m` must equal the stored sidecar MIME, and a sidecar exists only for content that passed the upload validator — reasoned at `imeta.rs:71-76` | `imeta.rs:259-266` |
| `imeta` thumbnail masquerading as a primary | `url` and `image` reject `.thumb.`; `thumb` must end `.thumb.jpg` | `imeta.rs:62-66`, `:97-105`, `:123-128` |
| Back-dated event injection | ±900 s symmetric fence on all kinds; ±120 s on 28936 and 9040–9044 | `ingest.rs:1480-1487`, `:1830-1841` |
| Same-second replacement race on relay-signed addressables | `created_at` forced strictly past any existing event of the same coordinate, so the random-event-id NIP-16 tiebreaker cannot strand a stale snapshot | `side_effects.rs:905-928`, `:3092-3115` |
| Thread-depth exhaustion | depth ≤ 100 | `ingest.rs:645-649` |
| Content exhaustion | 256 KiB global, 60 KiB for 40008, 64 chars for kind:7 emoji | `ingest.rs:1489`, `:898`, `:2283` |
| SQL injection in the one raw-SQL module | all values bound as parameters; the advisory-lock key is a computed `i64` | `command_executor.rs:157-232` |
| Log injection / reflected event content | `Rejected` messages can embed event-controlled values (raw `visibility`, `channel_type`, tag pubkeys); the HTTP bridge truncates before logging but returns the full string to the client | `api/bridge.rs:842-851` |
| Revoked access to a live subscription | eviction on 9001/9022/9002-flip/archive, plus per-event fan-out membership re-check as the cluster-wide backstop | `side_effects.rs:39-140`, `:1428-1438` |
| Banned socket that missed the disconnect fan-out | durable re-check on every write — the write-path gate is explicitly "the durable backstop the fan-out's best-effort delivery relies on" | `ingest.rs:1589-1601` |

---

### 5. Privilege-escalation surfaces

| Surface | Assessment |
|---|---|
| **Role grant via 9000** | Private channels: gated (member + elevated-granter). Open channels: relay layer does nothing; only `buzz-db` blocks it (S-07). Self-add always bypasses the `channel_add_policy` check (`side_effects.rs:334-338`) — correct for joining, but it means the policy cannot stop an agent from adding *itself*. |
| **Role change via 9000 upsert** | `add_member`'s `ON CONFLICT … DO UPDATE SET role = EXCLUDED.role` (`buzz-db/src/channel.rs:412-421`) means 9000 doubles as a role-*change* command with no separate authorization. On an open channel, `effective_role` gating applies; on a private channel, `validate_admin_event`'s elevated check applies. No path grants owner without an existing owner/admin granter — but the two checks are in different layers and neither is named "change role". |
| **Last-owner orphaning** | Guarded in four places: `validate_admin_event` 9001 (`side_effects.rs:379-389`) and 9022 (`:647-661`), plus a second guard inside `handle_remove_user` (`:1273-1287`) and `handle_leave_request` (`:1919-1930`). Redundant but consistent. |
| **NIP-OA owner acting on private channels without membership** | Deliberate: 40003, 9002, 9005, 9008 skip the membership gate so an owning human can act on their agent's private channels ("OQ1 decision", `ingest.rs:1763-1770`). `actor_owns_any_owner_agent` (`side_effects.rs:240-257`) checks **all** active owners, not just the first, which is correct under co-ownership. Note the asymmetry: 9001 explicitly refuses this for non-members (`:365-367`) while 9002/9008 explicitly allow it (`:598-608`, `:632-640`) — three different answers to "can a non-member owner act?" across four kinds. |
| **Timeout does not cascade owner→agent** | Documented Phase-1 asymmetry (`ingest.rs:1602-1612`): a timed-out owner's agents can still write. Ban cascades structurally at the auth seam; timeout has no auth-seam presence. |
| **Moderation commands exempt from the ban/timeout gate** | Intentional so a timed-out admin can lift a timeout (`ingest.rs:1572-1578`). The handler independently re-checks the durable ban (`moderation_commands.rs:103-108`), so a banned actor cannot use the exemption. |
| **Relay-admin commands exempt from the ban/timeout gate** | `ingest.rs:1613` also excludes `is_relay_admin_kind`, with only "Relay-admin commands retain their separate authorization policy" as justification (`ingest.rs:1600-1601`). Unlike moderation commands, there is no visible statement that `relay_admin.rs` re-checks the community ban. This exemption is the weaker of the two. |
| **Approval `approver_spec = ""` / `"any"`** | Any authenticated principal may approve any workflow gate with an open spec (`command_executor.rs:965-968`). Role-based specs fail closed (`:979-983`), so the only two states are "one exact pubkey" or "anyone". |
| **kind:1059 pubkey-match waiver** | Any authenticated principal may publish a gift wrap signed by an arbitrary ephemeral key. Required by NIP-59; the accounting/metrics path deliberately classifies by the authenticated principal rather than the envelope signer (`ingest.rs:1370-1372`, test `:2934-2949`). |

---

### 6. Kinds that mutate state with no audit entry

| Kind(s) | State mutated | `events` row? | `EventCreated` audit? |
|---|---|---|---|
| 9030–9033 | `relay_members` (add / remove / role change), workspace profile | **no** | **no** |
| 9040–9044 | `community_bans`, `muted_until`, report resolution | **no** | **no** |
| 1984 | `moderation_reports` | **no** | **no** |
| 42000 | product-feedback table | **no** | **no** |
| 30350 | `push_leases` | **no** | **no** |
| 28936 | `relay_members` (self-removal) | **no** | **no** |
| 41010–41012, 30620, 46020, 46030, 46031 | DMs, workflows, runs, approvals | yes (raw INSERT, `command_executor.rs:196`) | ⚠ **no** — `handle_command` returns without ever calling `dispatch_persistent_event`, so command events are stored but never audited and never fanned out |
| 9000–9008, 9021, 9022 | membership, channel metadata, deletions | yes | yes, but only as generic `EventCreated` |
| 9035/9036 | `archived_identities` | yes (deliberately) | generic `EventCreated` |

**Six kind groups mutate durable state with no audit row of any kind**, and the seven
command kinds are stored-but-unaudited. Given the hash-chain design of `buzz-audit`
(`buzz-audit/src/hash.rs`), the chain's coverage of privileged actions is materially
narrower than its API suggests.

---

### 7. Controls that work well

- **Verification before everything.** Signature and event-id verification precede all
  routing except the four kind-classification rejects (`ingest.rs:1460-1478`), and the
  command-routing comment pins the invariant: routed "AFTER signature verification,
  timestamp check, pubkey/auth match, and scope validation — never before"
  (`ingest.rs:1532-1533`, echoed `command_executor.rs:7-12`).
- **Fail-closed restriction lookup.** A DB error refuses the write rather than allowing it
  (`ingest.rs:1633-1641`).
- **Durable ban re-check on every write** as a backstop for missed live disconnects
  (`ingest.rs:1589-1601`).
- **Community-scoped everything**, with two explicit refusals to re-derive tenancy from
  client input (`command_executor.rs:759-771`, `:826-831`).
- **Cross-channel target checks** on all four target-bearing kinds (5, 9005, 40003, 45002).
- **Sidecar-authoritative media validation** — the event's claims are checked against
  storage, never trusted (`imeta.rs:259-278`).
- **Lowercase-hex enforcement** on `#p`-queried tags, closing a real invisibility attack
  with a regression test for it (`ingest.rs:3139-3152`).
- **NIP-70 `["-"]` tag required on 28936** (`ingest.rs:1843-1852`) and set on every
  relay-signed NIP-43/NIP-IA event.
- **`no unsafe`** — zero occurrences across all 8 911 lines.

---

### 8. Counted metrics

| Metric | ingest.rs | side_effects.rs | command_executor.rs | imeta.rs |
|---|---|---|---|---|
| `unsafe` blocks | 0 | 0 | 0 | 0 |
| `unwrap()` outside `#[cfg(test)]` | 0 | **2** (`:311`, `:314`) | 0 | 0 |
| `expect()` outside `#[cfg(test)]` | **4** (`:1998`, `:2000`, `:2338`, `:2344`) | 0 | 0 | 0 |
| `TODO`/`FIXME`/`XXX`/`HACK` | 0 | 0 | 0 | 0 |
| `#[test]` | 95 | 5 | **0** | 11 |
| `#[tokio::test]` | 0 | 0 | 0 | 0 |
| `#[ignore]` | 0 | 0 | 0 | 0 |

Total: 111 unit tests, **0 ignored**, 0 async tests. `command_executor.rs` (1 327 lines,
7 command handlers, the only raw-SQL module, the only LWW implementation outside
`buzz-db`) has **no `#[cfg(test)]` module at all**.
