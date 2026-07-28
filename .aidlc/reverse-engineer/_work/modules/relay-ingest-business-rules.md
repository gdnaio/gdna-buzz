## Module: buzz-relay — event ingest & side effects (`crates/buzz-relay/src/handlers`)
### Aspect: Business Rules

The ingest pipeline is one linear function, `ingest_event_inner`
(`ingest.rs:1427-2505`, **1 079 lines**), preceded by a tracing wrapper
`ingest_event` (`ingest.rs:1367-1425`). Rules below are numbered in **exact source
execution order**. Anything marked ⚠ is a verified deviation from the doc comment or from
the intent stated in the surrounding code.

---

### Phase A — envelope gates (kind-independent, pre-crypto)

| ID | Rule | Enforced at | Failure |
|---|---|---|---|
| **BR-IN-01** | kind 22242 (`KIND_AUTH`) may never be submitted through ingest. | `ingest.rs:1438-1441` | `Rejected("invalid: AUTH events cannot be submitted")` |
| **BR-IN-02** | kinds 44100 / 44101 (membership notifications) are relay-signed only. | `ingest.rs:1443-1447` | `Rejected("invalid: membership notifications are relay-signed only")` |
| **BR-IN-03** | Over HTTP, kinds 1059 (gift wrap) and 20001 (presence) are refused; they are WebSocket-only. | `ingest.rs:1449-1453` | `Rejected("invalid: kind {n} is only accepted via WebSocket")` |
| **BR-IN-04** | `is_relay_only_kind` (13534, 30622, 39005, 39006, 40901, 40902) is refused from any client. | `ingest.rs:1455-1457`; predicate `buzz-core/src/kind.rs:682-693` | `Rejected("restricted: relay-only kind")` |

⚠ BR-IN-01..04 all run **before** signature verification (BR-IN-05). An unauthenticated
malformed event can therefore be classified by kind without paying the Schnorr cost —
intentional, but it means these four rules provide no cryptographic assurance about the
claimed kind.

---

### Phase B — cryptographic and size gates

| ID | Rule | Enforced at | Failure |
|---|---|---|---|
| **BR-IN-05** | Event id and Schnorr signature must verify. Verification runs on `spawn_blocking`; the event is shared by `Arc` to avoid deep-cloning up to 256 KiB of content. | `ingest.rs:1460-1478`; `buzz_core::verification::verify_event` | `Rejected("invalid: {e}")`; task panic → `Internal("error: internal verification error")` |
| **BR-IN-06** | `\|created_at − server_now\| ≤ 900 s` (±15 min), `MAX_TIMESTAMP_DRIFT_SECS`. Applies to **all** kinds. | `ingest.rs:1480-1487` | `Rejected("invalid: event timestamp too far from server time")` |
| **BR-IN-07** | `content.len() ≤ 262 144` bytes (256 KiB), `MAX_EVENT_CONTENT_BYTES`. | `ingest.rs:1489-1497` | `Rejected("invalid: content exceeds maximum size of 262144 bytes (got n)")` |
| **BR-IN-08** | The event's signing pubkey must equal the authenticated principal — **except** for kind 1059 (NIP-59 gift wrap), which deliberately uses an unrelated ephemeral key. | `ingest.rs:1498-1503` | `AuthFailed("invalid: event pubkey does not match authenticated identity")` |

BR-IN-06 is a **symmetric** fence: events more than 15 minutes in the *past* are also
rejected, which makes back-dated import impossible through ingest. Two kinds tighten it
further to ±120 s: 28936 (`ingest.rs:1830-1841`) and 9040–9044 (in
`moderation_commands.rs:116`).

---

### Phase C — kind allowlist and scope

| ID | Rule | Enforced at | Failure |
|---|---|---|---|
| **BR-IN-09** | The kind must appear in `required_scope_for_kind`'s match — an **81-kind allowlist**. Unknown kinds (including 47 of the 127 `ALL_KINDS` entries) are refused. This is the real kind gate. | `ingest.rs:198-306`; raised `:1505-1508` | `Rejected("restricted: unknown event kind")` |
| **BR-IN-10** | kind 9002's required scope is *content-dependent*: `AdminChannels` if any `archived` tag is present, otherwise `ChannelsWrite`. | `ingest.rs:276-287` | — |
| **BR-IN-11** | Relay-admin kinds (9030–9033) may not be issued by a channel-scoped token. | `ingest.rs:1509-1518` | `AuthFailed("restricted: relay admin commands require a global token, not a channel-scoped token")` |
| **BR-IN-12** | kind 28936 may not be issued by a channel-scoped token. | `ingest.rs:1519-1524` | `AuthFailed("restricted: leave requests require a global token")` |
| **BR-IN-13** | The connection's scopes must contain the kind's required scope. | `ingest.rs:1525-1530` | `AuthFailed("restricted: insufficient scope (need {scope})")` |

⚠ **BR-IN-13 is unreachable in production.** Both transports construct `IngestAuth` with
`buzz_auth::Scope::all_known()` — WS at `buzz-auth/src/lib.rs:134-141`, HTTP at
`api/bridge.rs:827`. `Scope::all_known()` returns all 16 scopes
(`buzz-auth/src/scope.rs:68-87`). No production code path produces a reduced scope set, so
`!auth.scopes().contains(&required)` is always `false`. The 81-row scope column in the
dispatch table is documentation, not enforcement.

⚠ **BR-IN-11 and BR-IN-12 are also unreachable.** `AuthContext.channel_ids` is
hard-coded `None` on the WS auth path (`buzz-auth/src/lib.rs:138`), and
`IngestAuth::Http` has no `channel_ids` field, so `auth.channel_ids()`
(`ingest.rs:117-126`) never returns `Some`. The code's own doc comment admits this:
"In pure Nostr mode this always returns None" (`ingest.rs:113-115`).

---

### Phase D — early routing (bypasses generic storage)

Order is load-bearing. Each of these returns before the write-block gate or before storage.

| ID | Rule | Enforced at |
|---|---|---|
| **BR-IN-14** | Command kinds (30620, 41010–41012, 46020, 46030, 46031) are routed to `command_executor::handle_command`. The comment at `ingest.rs:1532-1533` pins the invariant: routed **after** signature, timestamp, pubkey-match, and scope — never before. | `ingest.rs:1534-1536` |
| **BR-IN-15** | kind 42000 (product feedback) is sidecarred to a private deployment table; it never enters `events` and never fans out. | `ingest.rs:1538-1558` |
| **BR-IN-16** | kind 1984 (NIP-56 report) is persisted only to `moderation_reports`; not stored publicly, never fanned out. Reports remain submittable while the author is timed out **and** in the missed-disconnect window after a ban, because they are non-actioning signals. | `ingest.rs:1560-1570` |
| **BR-IN-17** | Moderation commands 9040–9044 execute directly and are never stored or fanned out. Routed **before** the ban/timeout gate so a timed-out admin can lift a timeout; the handler re-checks the durable ban itself (`moderation_commands.rs:103-108`). | `ingest.rs:1572-1588` |

---

### Phase E — community restriction (ban / timeout write-block)

| ID | Rule | Enforced at | Failure |
|---|---|---|---|
| **BR-IN-18** | For every kind **except** 9040–9044 and 9030–9033, a durable `moderation_restriction_state` lookup runs. `banned = true` → refuse. | `ingest.rs:1613-1624` | `AuthFailed("blocked: you are banned from this community")` |
| **BR-IN-19** | `muted_until > now` → refuse the write (connection stays open so the client can render a countdown). | `ingest.rs:1625-1632` | `AuthFailed("restricted: you are timed out until {unix_ts}")` |
| **BR-IN-20** | A DB error in the restriction lookup **fails closed**. | `ingest.rs:1633-1641` | `Internal("error: internal error checking restriction state: {e}")` |
| **BR-IN-21** | The gate checks the *authoring* pubkey only — no NIP-OA owner→agent cascade. Documented as a deliberate Phase-1 asymmetry: ban cascades structurally at the auth seam (a banned owner's agent can never authenticate), timeout has no auth-seam presence and therefore does **not** cascade. | `ingest.rs:1602-1612` (rationale) | — |

This gate is one DB round-trip on **every** write. The code names the intended fix
(a restriction-state cache) at `ingest.rs:1609-1612`.

---

### Phase F — channel resolution and tenancy

| ID | Rule | Enforced at | Failure |
|---|---|---|---|
| **BR-IN-22** | kind 7: `channel_id` is derived from the **last** `e` tag's target event's `channel_id`, not from an `h` tag (`derive_reaction_channel`). Missing target → reject; missing/invalid `e` → reject; DB error → `Internal`. | `ingest.rs:331-377`, `:1644-1663` | `invalid: reaction target event not found` / `invalid: reaction must reference a target event via e tag` / `error: internal error looking up reaction target: {e}` |
| **BR-IN-23** | kind 1059: `channel_id` forced `None`. | `ingest.rs:1664-1665` | — |
| **BR-IN-24** | kind 5: `channel_id` derived from the **first** `e` tag's target's channel; absent target ⇒ `None` and deferred to BR-IN-33. | `ingest.rs:1666-1706` | `invalid: malformed deletion target id`; `error: looking up deletion target: {e}` |
| **BR-IN-25** | All other kinds: `channel_id` = first parseable UUID in an `h` tag. | `extract_channel_id` `ingest.rs:308-319`, called `:1707` |  — |
| **BR-IN-26** | `is_global_only_kind` (44 kinds, `ingest.rs:379-453`) forces `channel_id = None` regardless of any `h` tag. | `ingest.rs:1709-1711` | — |
| **BR-IN-27** | `requires_h_channel_scope` (23 kinds, `ingest.rs:455-491`) must yield a channel. | `ingest.rs:1713-1717` | `Rejected("invalid: channel-scoped events must include an h tag")` |
| **BR-IN-28** | The two sets are provably disjoint — asserted by an exhaustive test over all 65 536 kind values. | test `global_only_and_channel_scoped_are_disjoint` `ingest.rs:2753-2762` | — |
| **BR-IN-29** | Token-scoped channel access check (dead — see BR-IN-11 note). A channel-scoped token may not publish a global event. | `check_token_channel_access` `ingest.rs:525-532`; `:1719-1728` | `AuthFailed("restricted: channel-scoped tokens cannot publish global events")` |
| **BR-IN-30** | **Tenancy fence.** Every DB access in the pipeline is `tenant.community()`-scoped, resolved server-side from the request host — never from the event. Verified: `get_channel` `:1739`, `is_member_cached` (via `check_channel_membership` `:501`), `get_event_by_id` `:1690`/`:2245`, all inserts `:2298`/`:2371`/`:2385`/`:2394`. `claimed_community_from_event(&event)` is recorded in the trace **only as a claim**, never used as an authorization basis (`ingest.rs:1777-1782` documents this explicitly). | throughout | — |

⚠ **BR-IN-26 leaks through the read path.** The doc comment at `ingest.rs:369-377`
concedes it: the raw `h` tag stays on the signed event, and `filter.rs` treats explicit
`h` tags as authoritative, so a stray `h` on a global-only kind still matches `#h`
queries. Nulling `channel_id` fixes the write side only. Flagged in the source as
"a known limitation … should be addressed in the filter layer as a follow-up".

---

### Phase G — membership / authorization

| ID | Rule | Enforced at | Failure |
|---|---|---|---|
| **BR-IN-31** | If a channel is resolved, the author must be an active member **or** the channel's `visibility == "open"`. Uses the request's already-fetched `channel_row` to avoid a second SELECT. | `check_channel_membership` `ingest.rs:493-523`; called `:1785` | `Rejected("restricted: not a channel member")` |
| **BR-IN-31b** | A membership-lookup DB failure yields `error: database error: {e}` classified as **`Rejected`**, not `Internal`. ⚠ Inconsistent with BR-IN-20's fail-closed-as-`Internal` convention; over HTTP this surfaces as 400 instead of 500. | `ingest.rs:501`, mapped `:1802` | `Rejected("error: database error: …")` |
| **BR-IN-32** | Six kinds **skip** BR-IN-31 and rely on their own validator: 9021 (join needs no prior membership), 9007 (channel does not exist yet), 40003, 9002, 9005, 9008 (per-kind validators are the authority, so an agent's owning human can act without being a member — "OQ1 decision", `ingest.rs:1763-1770`). | `ingest.rs:1770-1775` | — |
| **BR-IN-32b** | A conformance `AuthCheck` verdict (`Allow`/`Deny`) is emitted at the call site, carrying the *claimed* community from the event alongside the server-resolved verdict basis. | `ingest.rs:1776-1801` | — |

---

### Phase H — kinds handled directly (post-authorization, pre-storage)

| ID | Rule | Enforced at |
|---|---|---|
| **BR-IN-33** | Relay-admin kinds 9030–9033 mutate `relay_members` and return; **never stored**. | `ingest.rs:1808-1818` |
| **BR-IN-34** | kind 28936 requires `require_relay_membership = true`, a ±120 s freshness window, and a NIP-70 `["-"]` tag. `remove_relay_member` handles NotFound / IsOwner atomically; the relay owner may not leave. Then two fire-and-forget publishes (8001 delta + 13534 snapshot), whose failures are `warn!`-logged only. **Never stored.** | `ingest.rs:1820-1902` |

---

### Phase I — per-kind pre-storage validators

| ID | Rule | Enforced at |
|---|---|---|
| **BR-IN-35** | `is_admin_kind` = `9000..=9022` → `validate_admin_event`. 9007 returns `Ok` immediately (no channel yet). All others must carry a resolvable `h` tag, and the channel must not be archived unless the event is a 9002 `archived=false` unarchive. | `side_effects.rs:26-28`, `:259-283`; called `ingest.rs:1905-1913` |
| **BR-IN-36** | 9000 (PUT_USER): `role` tag must parse. On **private** channels the actor must be an existing member, and only owner/admin may grant an elevated role. Self-add is always allowed. Third-party add consults the target's `channel_add_policy`: `owner_only` → only the registered owner; `nobody` → refuse; `anyone`/unknown → allow. ⚠ On **open** channels no relay-layer authorization runs at all — the only elevated-role gate is inside `buzz_db::channel::add_member` (`buzz-db/src/channel.rs:391-410`). | `side_effects.rs:284-372` |
| **BR-IN-37** | 9001 (REMOVE_USER): self-removal requires active membership and forbids removing the last owner. Removing others requires owner/admin, or member + NIP-OA ownership of the target. Non-members are refused outright — the code explicitly declines to check `is_agent_owner` for non-members ("you must be in the channel to remove anyone, even your own bot", `side_effects.rs:365-367`). | `side_effects.rs:373-409` |
| **BR-IN-38** | 9002 (EDIT_METADATA): ≥1 recognised tag from {name, about, archived, topic, purpose, visibility, ttl}. `archived` ∈ {true,false}; `name` non-empty post-canonicalisation; `visibility` ∈ {open,private}; `ttl` = `""` (clear) or a positive `i32`. Privileged set {name, about, archived, visibility, ttl} needs owner/admin **or** owner-of-any-active-owner-agent; {topic, purpose} needs only membership. | `side_effects.rs:410-624` |
| **BR-IN-39** | 9005 (DELETE_EVENT): optional `action_id` must be a UUID; `e` tag required; target must exist and belong to the h-tag channel (fail-closed on missing target — prevents an admin of A deleting events in B). Author self-delete path is re-gated on membership/open-visibility, and is **disabled** when moderation metadata (`action_id`/`reason_code`/`public_reason`) is present, forcing the owner/admin path. Fallback: owner/admin, or NIP-OA owner of the message's agent author. | `side_effects.rs:508-624`, `author_delete_can_use_self_delete_path` `:2353-2355` |
| **BR-IN-40** | 9008 (DELETE_GROUP): owner only, or owner-of-any-active-owner-agent. | `side_effects.rs:625-644` |
| **BR-IN-41** | 9022 (LEAVE_REQUEST): active member, not the last owner. | `side_effects.rs:645-663` |
| **BR-IN-42** | 9035/9036 (NIP-IA): `handle_identity_archive_event` verifies the consent path (self / admin-role / owner-via-live-kind:0) and mutates `archived_identities`, emitting the relay-signed 8002/8003 delta and 13535 snapshot. The request then **falls through to normal storage** so the delta's `["e", request_id]` resolves. | `ingest.rs:1915-1919` |
| **BR-IN-43** | kind 5: `validate_standard_deletion_event`. `a`-tag path checks the coordinate's pubkey equals the actor or the actor owns that agent. `e`-tag path resolves each target *including deleted* and requires the same. | `side_effects.rs:179-238`; called `ingest.rs:1921-1925` |
| **BR-IN-44** | If a channel was resolved, `channel.archived_at IS NOT NULL` refuses the write — unless the event is 9002 with an `archived=false` tag. | `ingest.rs:1927-1943` |
| **BR-IN-45** | kinds 5 and 9005: `e_count + a_count` must be **exactly 1**. `count_e_tags` counts *all* `e` tags including malformed ones, so a valid + malformed pair is rejected (regression-tested at `ingest.rs:3066-3081`). | `ingest.rs:1946-1958`, `count_e_tags` `:719-726` |
| **BR-IN-46** | kind 40003: `validate_edit_ownership` — target exists, same channel, actor is the target's *effective* author (relay-signed events resolve via `actor` then `p` tag) or the agent's owning human. Author path re-gates membership so a removed private-channel member cannot mutate old messages. | `ingest.rs:763-842`, `effective_message_author` `:729-761`; called `:1960-1964` |
| **BR-IN-47** | kind 45002: target must exist, be kind 45001 or 45003, and live in the same channel. | `ingest.rs:844-894`; called `:1966-1970` |
| **BR-IN-48** | kind 40008: content ≤ 61 440 B; `repo` tag required and `http(s)`; `commit` tag required and ≥7 hex chars; `parent-commit` ≥7 hex; `branch` needs both source and target; `pr` a positive integer. | `ingest.rs:896-963`; called `:1972-1974` |
| **BR-IN-49** | kind 30174: exactly one 64-**lowercase**-hex `d`, exactly one 64-lowercase-hex `p`. Uppercase is refused because readers query `#p` lowercase and Nostr tag matching is byte-exact — an uppercase head would win NIP-33 replacement then be invisible (`ingest.rs:1010-1013`). Content must be a plausible NIP-44 v2 payload. | `ingest.rs:965-1025`; called `:1976-1979` |
| **BR-IN-50** | NIP-44 v2 content shape: non-empty; standard base64 alphabet only; length a multiple of 4; padding only in the final two positions, ≤2 chars; decoded length ≥ 99 bytes; first decoded byte `0x02`. Envelope sanity only — the MAC is never checked at the relay. | `validate_engram_nip44_content` `ingest.rs:1084-1149` |
| **BR-IN-51** | kind 44200: exactly one 64-lc-hex `p`, exactly one 64-lc-hex `agent` equal to `event.pubkey`, **no** `h` tag (channel identity lives inside the ciphertext), NIP-44 v2 content. Then a DB check that `p` is the registered owner of `event.pubkey`. | `ingest.rs:1151-1221`; called `:1981-2016` |
| **BR-IN-52** | kind 30300: exactly one non-empty `d`; at most one `not_before`; `not_before` must be canonical decimal (no sign, whitespace, decimal point, or leading zero except `"0"`), ≤ 9 007 199 254 740 991, and ≤ `now + SPROUT_MAX_NOT_BEFORE_DELTA` (default 31 536 000 s). When both are present, `expiration > not_before`. A malformed `expiration` is ignored (NIP-40's concern). | `validate_not_before` `ingest.rs:1223-1250`, `validate_event_reminder` `:1252-1326`; called `:2018-2021` |
| **BR-IN-53** | kind 30175: exactly one `d`, non-empty, ≤64 chars, matching `^[a-z0-9][a-z0-9_-]{0,63}$`. Rationale: an empty `d` collapses every persona into the `(pubkey, 30175, "")` slot — LWW data loss (`ingest.rs:1022-1024`). | `ingest.rs:1027-1082`; called `:2023-2026` |
| **BR-IN-54** | kind 9007: `name` required and non-empty after `canonical_channel_name`; `visibility` (default `open`) and `channel_type` (default `stream`) must parse — validated for **all** 9007 events, with or without an `h` tag, so invalid enums are refused pre-storage. With an `h` tag, the channel is created eagerly via `create_channel_with_id`; `was_created == false` returns `accepted:false, "duplicate: channel already exists"`. | `ingest.rs:2029-2132` |
| **BR-IN-55** | kind 9021: an `h` tag is mandatory; the channel must exist and be `open`. | `ingest.rs:2134-2154` |
| **BR-IN-56** | kind 30350: `push_lease::accept` returns one of 7 outcomes; 6 map to distinct `Rejected` strings and `Accepted` returns immediately. Infrastructure failures map to `Internal`, never `invalid:` — regression-tested at `ingest.rs:2732-2751`. | `map_push_accept_error` `ingest.rs:186-195`; `:2156-2204` |
| **BR-IN-57** | Every `imeta` tag is validated structurally, then every referenced blob is verified to exist in storage with matching MIME / size / duration, plus thumbnails and poster frames. | `imeta.rs:11-335`; called `ingest.rs:2206-2218` |
| **BR-IN-58** | `requires_h_channel_scope` kinds resolve NIP-10 ancestry: `e root` / `e reply` markers (4-element tags only); parent must exist and be in the same channel; a parent with no channel is refused; the client's `root` must match the resolved ancestry; **depth ≤ 100**. | `resolve_nip10_thread_meta` `ingest.rs:564-717`; called `:2220-2231` |
| **BR-IN-59** | kind 0 content must parse as JSON, checked *before* storage so the profile-sync side effect cannot silently fail on a stored event. | `ingest.rs:2233-2239` |
| **BR-IN-60** | kind 7 emoji: `content` defaults to `"+"` when empty; `chars().count() ≤ 64` (mirrors the SDK's `check_emoji_len` so raw clients cannot bypass it). | `ingest.rs:2274-2292` |

---

### Phase J — persistence and replacement semantics

| ID | Rule | Enforced at |
|---|---|---|
| **BR-IN-61** | kind 7 uses a single transaction that upserts the reaction row (`ON CONFLICT` dedup) with `reaction_event_id` already set, then stores the kind:7 event. Ordering is load-bearing: an **active duplicate returns before** a duplicate kind:7 event is stored. Three outcomes: `TargetMissing` → reject, `Duplicate` → `accepted:false`, `Inserted` → continue. | `ingest.rs:2294-2325`; `buzz-db/src/event.rs:1201-1246` |
| **BR-IN-62** | `is_replaceable` (0, 3, 41, 10000–19999) → `replace_addressable_event` with NIP-16 stale-write protection. | `ingest.rs:2367-2374` |
| **BR-IN-63** | `is_parameterized_replaceable` (30000–39999) → `d_tag` extracted, length ≤ `D_TAG_MAX_LEN` = 1024 (`buzz-db/src/event.rs:124`), then `replace_parameterized_event` keyed `(kind, pubkey, d_tag)`. | `ingest.rs:2375-2390` |
| **BR-IN-64** | Everything else → `insert_event_with_thread_metadata`. On DB failure, if a 9007 channel was pre-created it is compensated by `soft_delete_channel` + cache invalidation; `DbError::AuthEventRejected` maps to `invalid: AUTH events cannot be stored`, all else to `Internal`. | `ingest.rs:2391-2424` |
| **BR-IN-65** | **NIP-33/NIP-16 LWW conflict signal.** When the storage call reports `was_inserted == false` — the write lost the LWW race or the id already existed — ingest returns `accepted: true, message: "duplicate:"`. This is the relay-side conflict detection; `buzz-cli` converts it to `CliError::Conflict` → **exit code 5** (`buzz-cli/src/commands/mem.rs:105-108`, `commands/notes.rs:560-563`, `buzz-cli/src/error.rs:103`). | `ingest.rs:2426-2432` |
| **BR-IN-66** | Command kinds get their own LWW: for a NIP-33 `d_tag`, `persist_command_event` takes an FNV-1a-derived `pg_advisory_xact_lock` over `(community, kind, pubkey, d_tag)`, reads the current head, and treats `created_at < existing` or `created_at == existing && incoming_id >= existing_id` as **dominated** → `PersistResult::Duplicate`. Otherwise it soft-deletes the old row and inserts. | `command_executor.rs:135-224` |
| **BR-IN-67** | Command-event dedup: raw `INSERT … ON CONFLICT DO NOTHING`; `rows_affected() == 0` → `Duplicate`, transaction rolled back on drop. | `command_executor.rs:196-232` |

---

### Phase K — post-storage side effects

| ID | Rule | Enforced at |
|---|---|---|
| **BR-IN-68** | `is_side_effect_kind` gates `handle_side_effects`. Predicate: `0 \| 5 \| 9000..=9022 \| 30617 \| 10100 \| 41001..=41003 \| 40099`. | `side_effects.rs:35-37`; called `ingest.rs:2434-2441` |
| **BR-IN-69** | ⚠ **Side-effect failure does not fail the write.** `handle_side_effects` errors are `warn!`-logged and discarded; the event is already committed and is fanned out immediately afterwards. This is the module's core partial-failure semantic. | `ingest.rs:2434-2441`, then `:2489` |
| **BR-IN-70** | A freshly-inserted reply triggers a fire-and-forget relay-signed kind:39005 thread summary (`emit_live_thread_summary`), which **re-reads** counts from `thread_metadata` post-commit rather than incrementing, so the emitted summary is exact under concurrency. Fan-out only — never stored. | `ingest.rs:2443-2455`; `side_effects.rs:724-815` |
| **BR-IN-71** | kind 7 is deliberately **excluded** from `is_side_effect_kind` because dedup + DB writes already happened pre-storage (`side_effects.rs:31-34`). The old `handle_reaction()` is gone — noted at `side_effects.rs:1974-1976`. | `side_effects.rs:31-37` |
| **BR-IN-72** | The relay-signed system message (kind 40099) emitted by most NIP-29 side effects is best-effort: an insert failure is `warn!`-logged and fan-out proceeds. | `emit_system_message` `side_effects.rs:677-722` |
| **BR-IN-73** | Discovery events (39000/39001/39002) force `created_at > any existing event of the same (kind, pubkey, channel)` so two updates in the same second cannot lose to the random-event-id NIP-16 tiebreaker. Same guard in `publish_dm_visibility_snapshot`. | `emit_addressable_discovery_event` `side_effects.rs:902-960`; `:3092-3115` |
| **BR-IN-74** | 9002 `visibility` flips invalidate the accessible-channels and channel-visibility caches **before** any later event for that channel fans out, and an open→private flip eagerly evicts non-member subscriptions (with per-event fan-out membership re-checking as the cluster-wide backstop). | `side_effects.rs:1413-1447`; `evict_non_member_channel_subscriptions` `:95-121` |
| **BR-IN-75** | Deletions (kind 5 and 9005) look up thread metadata and call `soft_delete_event_and_update_thread`, which soft-deletes and **decrements** `reply_count`/`descendant_count` in one transaction, then emits a fresh 39005 so live badges count down. `deleted == false` short-circuits with no system message, to avoid false audit records. | `side_effects.rs:1615-1645`, `:2138-2168` |
| **BR-IN-76** | kind 5 targeting a kind:7 also removes the reaction row: first by `reaction_event_id`, falling back to `(target, actor, emoji)` derived from the reaction event itself. | `side_effects.rs:2170-2231` |
| **BR-IN-77** | kind 5 with no `e` tag at all routes to `handle_a_tag_deletion`. Routing keys on the **presence** of any `e` tag (not on decoded-target count) so a malformed `e` alongside an `a` can never silently soft-delete a coordinate. | `has_e_tag` `side_effects.rs:2300-2302`; `:2113-2118` |
| **BR-IN-78** | `a`-tag deletion dispatch: 30350 is ignored (revocation is a higher-generation replacement only); 30620 deletes the workflow by UUID or owner+name and invalidates the workflow cache; any other parameterized-replaceable kind is soft-deleted by coordinate; non-NIP-33 kinds are a no-op. | `side_effects.rs:1979-2106` |
| **BR-IN-79** | kind 30617: repo id `[a-zA-Z0-9._-]{1,64}`, no leading dot, no `..`. Same-owner re-announce is idempotent; a name held by another pubkey is a hard collision. Per-pubkey quota (`git_max_repos_per_pubkey`, default 100) is checked **before** claiming. Only a fresh `Reserved` claim may be rolled back on pointer failure; only a fresh claim emits the initial empty-refs kind:30618 (re-emitting would shadow real pushed refs under NIP-16). | `side_effects.rs:2391-2604` |
| **BR-IN-80** | 9001/9022 both re-check the last-owner guard inside their side-effect handler, duplicating the pre-storage check in `validate_admin_event`. | `side_effects.rs:1273-1287`, `:1919-1930` |
| **BR-IN-81** | 9021 join short-circuits when the actor is already a member, and fails closed on DB error rather than falling through to `add_member`. | `side_effects.rs:1856-1866` |
| **BR-IN-82** | Post-unarchive (9002 `archived=false`), a kind:44100 notification is re-emitted to **every** member purely as a resubscribe trigger, because archiving evicted their live subscriptions and unarchive otherwise emits no resubscribe signal. Documented known limitation: four sub-second archive/unarchive toggles by the same actor could collide event ids and skip a fan-out. | `side_effects.rs:1508-1546` |

---

### Phase L — command-executor rules

| ID | Rule | Enforced at |
|---|---|---|
| **BR-IN-83** | `handle_command` calls `ensure_user` first (FK requirement); failure is `warn!`-logged and execution continues. | `command_executor.rs:44-64` |
| **BR-IN-84** | 41010: 1–8 `p` tags; participant set = self + others deduplicated; then `open_dm`. A re-open of an existing DM clears the caller's `hidden_at` and republishes their NIP-DV snapshot. | `command_executor.rs:310-441` |
| **BR-IN-85** | 41011: caller must be a member; channel must be `channel_type == "dm"`; merged set ≤9; creates a **new** DM because DM participant sets are immutable. | `command_executor.rs:443-578` |
| **BR-IN-86** | 41012: caller must be a member of a `dm` channel; `hide_dm`; then republish the NIP-DV snapshot. | `command_executor.rs:580-651` |
| **BR-IN-87** | 30620: caller must be a channel member; the YAML must parse; an existing workflow at the same UUID must have the same owner **and** channel; the webhook secret is preserved across updates and returned only on first creation; the definition hash is computed **after** secret injection; the workflow's community is always `tenant.community()`, never re-derived from the client-supplied channel id, and the channel is re-verified inside that community. | `command_executor.rs:653-807` |
| **BR-IN-88** | 46020: the workflow lookup is community-scoped (a bare-id lookup could load another community's workflow and then satisfy membership against *its* colliding channel). Only the workflow **owner** may trigger — channel membership is explicitly insufficient because manual triggers run with owner authority. The trigger event is persisted under `workflow.channel_id` so workflow ids do not leak to unrelated relay members. | `command_executor.rs:809-959` |
| **BR-IN-89** | 46030/46031: approval must be `Pending` and unexpired; `check_approver_spec` accepts `""`/`"any"` (anyone) or an exact 64-hex pubkey (case-insensitive compare) and **fails closed** on role-based or unrecognised specs. `update_approval_by_stored_hash` returning `false` is treated as a lost race. Grant resumes the run at `step_index + 1`; deny cancels it. Both guard on `RunStatus::WaitingApproval` before acting. | `command_executor.rs:961-1234`, `resume_workflow_after_approval` `:1236-1327` |
| **BR-IN-90** | ⚠ Commands are **not atomic**. The doc header claims "validate → begin tx → insert event → execute mutations → commit" (`command_executor.rs:3-4`), but the implementation note at `:92-98` admits domain mutations execute on the pool, not in the transaction: "if a mutation succeeds but commit fails, the mutation persists without the event record". Safety rests on the mutations being idempotent. | `command_executor.rs:92-98` |

---

### Retention / TTL

There is **no retention or TTL enforcement on events** in this module. The only TTL is
per-channel: `resolve_ttl` (`handlers/mod.rs:42-62`) reads a `ttl` tag and lets
`BUZZ_EPHEMERAL_TTL_OVERRIDE` clobber it; the value is stored on the channel row at 9007
creation (`ingest.rs:2099`, `side_effects.rs:1681`) or updated by 9002
(`side_effects.rs:1449-1481`). Actual expiry is done by an out-of-module reaper
(`main.rs:646-680`). NIP-40 `expiration` tags are read only for the 30300 ordering check
(BR-IN-52) and are never acted upon at ingest.

---

### Conformance-trace coverage rule

**BR-IN-91**: `ingest_event` arms an `EmitGuard` (`ingest.rs:1381-1386`) whose `Drop`
records an `ImplBug` if the request emitted **no** trace record. Terminal errors are
mapped centrally (`ingest.rs:1409-1417`), and the success paths emit
`WriteInsert`/`WriteInsertGlobal`/`WriteDuplicate` at their dispatch points
(`:2327-2350`, `:2457-2487`, `:2186-2193`) plus an explicit
`emit_product_feedback_success` for 42000 (`:1546`, helper `:133-154`).

⚠ At least **six** success exit paths return `Ok` without any emit and without a prior
`AuthCheck` (which only fires for channel-bearing kinds): kind 1984
(`ingest.rs:1565-1569`), moderation commands (`:1583-1587`), relay-admin
(`:1812-1816`), 28936 (`:1898-1902`), the 9007 duplicate-channel branch (`:2118-2123`),
and all seven command kinds via `handle_command`. Under a recording tracer this is a
CoverageBreach. Production impact is nil because `AppState::tracer` is `NoopTracer`
(`state.rs:798`), so the guard is inert outside conformance tests.
