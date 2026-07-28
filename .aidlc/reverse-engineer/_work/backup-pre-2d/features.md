<!-- Analyzed: 2026-07-25T01:12:08Z | Scope: full project -->
# Feature Inventory

> Status: initialized in Phase 1. Feature inventory, user journeys, and completeness
> assessment are populated per-module during Phase 2 and consolidated in Phase 3.

## Summary

Scan-time signal: 28 desktop feature packages, 11 mobile feature packages, and 5 declared
preview features (`workflows`, `projects`, `pulse`, `forum`, `agentManagedProfiles`).

Batch 2a covers the foundation layer, which **enables** capabilities rather than exposing
user-facing surfaces: 17 capabilities in `buzz-core` (13 full, 4 partial), 25 capability
areas in `buzz-sdk`, 19 in `buzz-persona`, and a narrow connect→auth→publish→read flow in
`buzz-ws-client`.

**Zero TODO/FIXME/HACK/XXX markers exist across all four crates.** Deferred work is
expressed as prose doc comments instead, which makes it invisible to marker-based debt
tooling. The notable stubs found this way:

| Stubbed / absent | Where | Statement in code |
|---|---|---|
| Persona lifecycle hooks | `crates/buzz-persona/src/resolve.rs:57` | "parsed, not executed — reserved for future use, not yet wired" |
| Persona skill scoping | `crates/buzz-persona/src/pack.rs:249` | `resolve_skills` fully implemented + tested but **never called** by any production path |
| MCP `${VAR}` interpolation | `crates/buzz-persona/src/resolve.rs:274` | env values pass through as literals |
| `engines.buzz` version gate | `crates/buzz-persona/src/manifest.rs:39-40` | parsed, never evaluated |
| Full NIP-10 threading for kind 1 | `crates/buzz-sdk/src/builders.rs:732-737` | "Full NIP-10 threading … is deferred" |
| Typed `REQ`/`CLOSE`/`COUNT`, reconnect/retry | `crates/buzz-ws-client` | absent — callers hand-roll via `send_raw` |
| Filter `limit` + NIP-50 `search` | `crates/buzz-core/src/filter.rs:35-104` | not implemented in the matcher |

Batch 2b covers the service layer. The same pattern repeats and intensifies: **zero
TODO/FIXME/HACK markers in six of seven crates** (only `buzz-workflow` has one, WF-08),
while a substantial amount of functionality is present-but-unreachable. Marker-based
tooling sees none of it.

Capabilities that exist in code but cannot be reached in production:

| Non-functional / unreachable | Where | Nature |
|---|---|---|
| Workflow `send_dm`, `set_channel_topic` | `crates/buzz-workflow/src/executor.rs` | Return `NotImplemented` (documented) |
| Workflow `add_reaction` | `crates/buzz-workflow/src/executor.rs` | **Undocumented failure** — POSTs to `/api/messages/{id}/reactions`, which the relay never registers (`crates/buzz-relay/src/router.rs:39-125`) |
| Workflow approval gates | `crates/buzz-workflow/src/executor.rs:663` | Token generated, never persisted; runs marked `Failed`; **nothing ever writes `WaitingApproval`**, so the relay's resume path is unreachable and `Db::create_approval` has no caller |
| `buzz-auth::ChannelAccessChecker` | `crates/buzz-auth` | **Zero implementors repo-wide**; the trait doc's claim that `buzz-db` implements it is false |
| `Scope::all_non_admin()` (14 scopes) | `crates/buzz-auth` | Never called — NIP-42 grants `all_known()` (16) |
| `check_ip_connection` / `LimitType::IpConnections` | `crates/buzz-pubsub/src/rate_limiter.rs:112-120` | Fully implemented, no production caller — and it is the one control that would cover currently-unmetered pre-auth traffic |
| `verify_chain` / `get_entries` (audit) | `crates/buzz-audit` | No production caller — nothing verifies the hash chain in normal operation |
| `verify_blossom_get_auth` | `crates/buzz-media` | Never called from the crate; the relay gate defaults to off |
| `PubSubConfig::with_unsubscribe_debounce`, `topic_refcount` | `crates/buzz-pubsub/src/lib.rs:93`, `:248` | Knob and metric hook, both unwired |
| Media GC / orphan cleanup / storage quota | `crates/buzz-media` | Absent entirely |
| **Typing indicators** | `crates/buzz-pubsub` | **Advertised in four places, implemented nowhere.** `Cargo.toml:8`, an orphaned doc comment at `src/lib.rs:43`, `AGENTS.md`, and `ARCHITECTURE.md:82`, `:432`, `:434`, `:777`, `:801` — including a concrete ZADD/ZREMRANGEBYSCORE/EXPIRE design at `:452-456`. Repo-wide: zero sorted-set calls, zero `buzz:typing` keys. Real behaviour is kind-20002 ephemeral events (`crates/buzz-core/src/kind.rs:407`) produced by `buzz-acp` and fanned out generically |

Capabilities confirmed **working** in 2b, contrary to documentation that says otherwise:

| Capability | Doc claim | Reality |
|---|---|---|
| Rate limiting | `ARCHITECTURE.md:823` (§9 #2), `:390`, `:460` — "not enforced, test stub only" | Enforced before work on WS `EVENT`/`REQ`/`COUNT` and the HTTP bridge (`crates/buzz-pubsub/src/rate_limiter.rs:99`, wired `crates/buzz-relay/src/state.rs:712`) |
| Workflow `execute_from_step` | "exists for future use" | **3 live callers**; `execute_run` is the one with none |
| Media hash verification | implied absent | Verified server-side on upload (not on read) |

Also undocumented-but-present: a **5th** workflow trigger, `diff_posted`, and an **11th**
audit action, `MediaUploaded`.

User journeys and completeness against the README's "Works today / Being wired up"
table are assembled in Phase 3.

### Batch 2c features (relay, mesh, conformance)

Three feature families land in this batch, and in each one a headline capability is present in
code but not reachable in a deployed configuration.

- **Git hosting is real and broad** — smart HTTP, policy hooks, CAS publish, ref protection,
  and a hydrate path — but has no read authorization
  (`crates/buzz-relay/src/api/git/transport.rs:539-594`, `:786-827`), and pointer creation is
  ungated (a zero-ref-command push creates pointers, bypassing `git_repo_names` and quota).
- **Huddle audio and the tunnel** are hosted directly by the relay rather than a separate
  service (`crates/buzz-relay/src/` audio group).
- **`workflow_sink` implements 1 of the 7 workflow actions** the engine defines.
- **The mesh subsystem is complete enough to run and never runs.** `buzz-relay-mesh` is absent
  from `ARCHITECTURE.md` and `AGENTS.md`, and absent from `just test-unit`, so all 32 of its
  tests compile but never execute. `MeshRuntime::shutdown()` is never called.
- **The 5 `crates/buzz-relay/examples/mesh_*.rs` do not exercise `buzz-relay-mesh`** — they are
  MeshLLM (`mesh_llm_sdk`) smoke tests pulled in via git dev-dependencies
  (`crates/buzz-relay/Cargo.toml:84-85`).
- **The conformance gate is built but unassembled.** Emission is live at the ingest and REQ
  seams; `EmitGuard` is armed at exactly one seam
  (`crates/buzz-relay/src/handlers/ingest.rs:1408-1412`); row-community projection runs on both
  read lanes. But `check_trace` (`crates/buzz-conformance/src/checker.rs:74`) has zero
  production callers and `JsonlTracer` — the only persisting tracer
  (`crates/buzz-relay/src/conformance/tracers.rs:30-72`) — is never instantiated. Production
  binds `NoopTracer` (`crates/buzz-relay/src/state.rs:798`), the sole assignment to that field.
  So every `TraceStep` is constructed and discarded, and the relay→JSONL→checker pipeline the
  crate exists to run is not wired anywhere.

## Features

| Feature | Surfaces | Status | Modules | Source |
|---|---|---|---|---|
| _pending_ | | | | |

## Desktop Feature Packages (scan-level)

`agent-memory`, `agents`, `channel-templates`, `channels`, `chat`, `communities`,
`community-members`, `custom-emoji`, `forum`, `home`, `huddle`, `identity-archive`,
`local-archive`, `mesh-compute`, `messages`, `moderation`, `notifications`, `onboarding`,
`presence`, `profile`, `projects`, `pulse`, `reminders`, `search`, `settings`, `sidebar`,
`user-status`, `workflows`

## Mobile Feature Packages (scan-level)

`activity`, `channels`, `custom_emoji`, `forum`, `home`, `invites`, `pairing`, `profile`,
`pulse`, `search`, `settings`

## User Journeys

_Pending Phase 2._

## Completeness Assessment

_Pending Phase 2/3 — cross-check against README's "Works today / Being wired up /
Pending" table and ARCHITECTURE.md §9 Known Limitations._

---

# Phase 2 — Module Findings

## Module: buzz-core (`crates/buzz-core`)

### Aspect: Features

`buzz-core` is a foundation library — it enables capabilities in other crates rather than exposing user-facing surfaces itself. Completeness below is judged only against what this crate's own code does; a capability marked "partial" means part of the contract described in its own doc comments is not implemented here.

---

### Capability inventory

| # | Capability | What this crate provides | Completeness | Evidence |
|---|-----------|--------------------------|--------------|----------|
| F-1 | Nostr event verification | ID-hash recomputation + Schnorr signature check, with a typed error carrying computed vs claimed ID | **full** | `src/verification.rs:11-32`, `src/error.rs:2-20` |
| F-2 | Relay event metadata wrapper | `StoredEvent` binding an event to receive time, channel scope, and a private verified flag | **full** | `src/event.rs:11-48` |
| F-3 | Event-kind registry | 130 kind constants, 4 range bounds, 4 kind-set slices, `ALL_KINDS` (127), 9 classification predicates, 2 kind extractors, 25 compile-time range assertions | **full** | `src/kind.rs:9-820` |
| F-4 | NIP-01 subscription filter matching | kinds/authors/since/until/ids(prefix)/generic-tag matching with OR across filters, AND within a filter, plus the `#h` channel fallback | **full** for the fields it implements; **partial** vs NIP-01 overall — `limit` and NIP-50 `search` are not handled here (no reference to either in `filter.rs`) | `src/filter.rs:10-104` |
| F-5 | Per-viewer result-level read authorization | `reader_authorized_for_event` gate for DM-visibility and agent-turn-metric kinds | **full** (as a predicate; enforcement at delivery sites lives in `buzz-relay`, per doc `filter.rs:19-22`) | `src/filter.rs:23-33` |
| F-6 | SSRF address classification | `is_private_ip` covering 7 IPv4 and 9 IPv6 categories plus 3 recursive embedded-IPv4 paths (was "6 IPv6 categories plus IPv4-mapped recursion" — `c26bf594` added NAT64 local-use, Teredo and 6to4 to the direct checks, and NAT64 well-known + IPv4-translated + IPv4-compatible to the recursive paths) | **full** for the enumerated ranges; see security doc for ranges *not* covered | `src/network.rs:46-95` |
| F-7 | Multi-tenant community identity | `CommunityId` newtype, `TenantContext`, host normalization, relay-URL authority extraction | **partial by design** — the type system removes accidental client-controlled tenants; the deliberate path is closed by lint + review, stated explicitly at `tenant.rs:23-30` | `src/tenant.rs:37-172` |
| F-8 | Relay runtime identity canonicalization | `normalize_relay_url` folding loopback spellings, case, default ports, root path | **full** | `src/relay.rs:37-78` |
| F-9 | Channel + role vocabulary | visibility/type/role enums with string round-trip and a numeric permission ladder | **full** | `src/channel.rs:22-181` |
| F-10 | Channel-name canonicalization | strips leading `#`/whitespace, trims trailing space | **full** | `src/channel.rs:15-19` |
| F-11 | Presence vocabulary | curated `online`/`away`/`offline` enum for structured APIs | **full** | `src/presence.rs:11-35` |
| F-12 | Agent observer frame crypto | NIP-44 v2 encrypt/decrypt of arbitrary serde payloads with size envelopes and zeroization; frame/tag name constants | **full** | `src/observer.rs:13-110` |
| F-13 | NIP-AM agent turn metrics | payload type (`TokenCounts`, `StopReason`, `AgentTurnMetricPayload`), numeric validation, symmetric encrypt/decrypt | **partial** — `session_id`/`turn_seq` requirements and monotonicity documented at `agent_turn_metric.rs:97-108` are not enforced in code; only `cost_usd` is validated (`:147-169`) | `src/agent_turn_metric.rs:22-191` |
| F-14 | NIP-AE agent engrams | slug grammar, shorthand normalization, conversation key, d-tag HMAC, byte-exact body codec, strict JSON parsing, `[[ref]]` extraction, event build, envelope validation + decrypt, LWW head selection, monotonic clock rule, `Listing` wire type | **full** for the primitives; signature verification is delegated to the caller by contract (`engram.rs:478-482`) | `src/engram.rs:20-603` |
| F-15 | Git push permission engine | ref-pattern grammar + matcher, update classification, `buzz-protect` tag parsing with forward-compat warnings, effective-rule union, default role table, per-ref and whole-push evaluation with denial reasons | **full** for evaluation inputs; the `is_ancestor` fact and the Bot→Member promotion are supplied by callers (`git_perms.rs:208-210`, test note `:910-924`) | `src/git_perms.rs:19-597` |
| F-16 | NIP-AB device pairing | HKDF derivations (session id, SAS, transcript hash), 6-digit SAS formatting, constant-time compare, `nostrpair://` QR encode/decode, full 7-state session machine for both roles with dedup, expiry, abort handling, and secret zeroization | **full** as a pure protocol engine; relay I/O and user interaction are the caller's job (`session.rs:6-8`) | `src/pairing/` (5 files, 2,638 lines) |
| F-17 | Test fixtures for dependents | `make_event`, `make_event_with_keys`, `make_stored_event` behind the `test-utils` feature | **full** | `src/lib.rs:47-74`; consumed by `crates/buzz-relay/Cargo.toml:89` |

---

### TODO / FIXME / HACK / XXX comments

**Zero.** A recursive search of `crates/buzz-core/src` for `TODO`, `FIXME`, `HACK`, and `XXX` returns no matches. The crate carries no in-code deferral markers.

Related in-code forward-looking notes (not TODO-tagged, quoted verbatim):

| Note | file:line |
|------|-----------|
| "Currently a tiny linear set. If this grows past ~4 kinds, convert to a / compile-time bitset or sorted array with binary search for hot-path use." | `src/kind.rs:118-119` |
| "Forward-compatibility: unknown rules are skipped but reported." | `src/git_perms.rs:345` |
| "We still verify rather than `.expect()` so a future change to the serializer can't silently introduce a panic on the hot path." | `src/engram.rs:445-448` |
| "Residual transient copies that cannot be zeroized: 1. `serde_json::to_string` may create intermediate buffers during serialization 2. `nip44::encrypt` reads the plaintext but does not zero its internal copy" | `src/pairing/session.rs:556-558` |

One truncated/orphaned comment fragment sits in the engram test module with no preceding context line:

```
    //    vectors". Pinning these as CI invariants is the single best
```
— `src/engram.rs:615`. (Recorded in the debt doc as well.)

---

### Capabilities explicitly *not* in this crate

| Absent capability | Evidence |
|---|---|
| Any I/O, async runtime, DB, cache, or HTTP | `Cargo.toml:28` comment "NO tokio, NO sqlx, NO redis, NO axum — zero I/O dependencies"; no `tokio`/`sqlx`/`redis`/`axum` entries in `[dependencies]` (`Cargo.toml:13-27`) |
| Environment/config reading | no `env::var` or `std::env` occurrences anywhere in `src/` |
| Filter `limit` handling and NIP-50 `search` routing | `filter.rs:35-104` implements neither |
| NIP-42 AUTH URL equivalence | delegated to `buzz-auth` by explicit doc statement (`relay.rs:28-32`) |
| Signature verification inside engram decrypt | delegated to the caller (`engram.rs:478-482`) |
| Enforcement of the p-gate / author-only / result-gated read rules | this crate only *declares* the kind sets (`kind.rs:112-156`); enforcement is documented as living in the relay and search crates |


## Module: buzz-sdk (`crates/buzz-sdk`)

### Aspect: Features

The crate's stated scope: build validated, unsigned Nostr events for Buzz
operations; the caller signs and transports them
(`crates/buzz-sdk/src/lib.rs:5-13`). Every capability below is therefore
"produce the correct wire form", never "perform the operation".

---

### 1. Capabilities driveable through the SDK

| Capability area | Builders / functions | Kinds | Completeness | Evidence |
|---|---|---|---|---|
| Stream chat messaging | `build_message` | 9 | Full — threading, mentions, broadcast flag, imeta media | `builders.rs:219-241` |
| Forum discussions | `build_forum_post`, `build_forum_comment`, `build_vote` | 45001, 45003, 45002 | Full for post/comment/vote | `builders.rs:278-305`, `446-460` |
| Message editing | `build_edit` | 40003 | Full | `builders.rs:378-389` |
| Message deletion | `build_delete_message`, `build_delete_message_with_options`, `build_delete_compat` | 9005, 5 | Full, incl. moderation tombstone metadata | `builders.rs:403-443` |
| Reactions | `build_reaction`, `build_custom_emoji_reaction`, `build_remove_reaction` | 7, 5 | Full (NIP-25 + NIP-30) | `builders.rs:463-498` |
| Custom emoji palette | `build_custom_emoji_set`, `normalize_custom_emoji_shortcode`, `CUSTOM_EMOJI_SET_D_TAG` | 30030 | Partial — write-side only; the doc states add/remove is "read-own-set → mutate → rebuild", and the read/union step is left to callers | `builders.rs:503-527` |
| Code-diff sharing | `build_diff_message` + `DiffMeta` | 40008 | Full metadata surface (repo, commit, file, branch pair, PR number, language, truncation, alt text) | `builders.rs:308-375` |
| Canvas documents | `build_set_canvas` | 40100 | Partial — no content-length validation, unlike every other content builder | `builders.rs:529-532` |
| Profiles (NIP-01) | `build_profile` | 0 | Partial — only 5 fields (`display_name`, `name`, `picture`, `about`, `nip05`); other kind-0 fields cannot be set | `builders.rs:537-562` |
| Channel lifecycle | `build_create_channel`, `build_update_channel`, `build_set_topic`, `build_set_purpose`, `build_archive`, `build_unarchive`, `build_delete_channel` | 9007, 9002, 9008 | Full for name/about/visibility/type/ttl/topic/purpose/archive | `builders.rs:604-730` |
| Membership | `build_add_member`, `build_remove_member`, `build_join`, `build_leave` | 9000, 9001, 9021, 9022 | Full | `builders.rs:565-598`, `703-706` |
| Direct messages | `build_dm_open`, `build_dm_add_member` | 41010, 41011 | Partial — conversation setup only; no DM message-body or gift-wrap builder in this crate | `builders.rs:1544-1566` |
| Presence | `build_presence_update` | 20001 | Full for the three-state vocabulary | `builders.rs:1570-1585` |
| Global social notes | `build_note` | 1 | Partial by design — flat reply model; "Full NIP-10 threading (root + reply + p-tags) is deferred" | `builders.rs:732-748` |
| Contact lists (NIP-02) | `build_contact_list` | 3 | Full replacement semantics; deltas require caller read-before-write | `builders.rs:753-813` |
| Git repo announcement (NIP-34) | `build_repo_announcement`, `build_repo_announcement_with_tags` | 30617 | Full, incl. read-modify-write path preserving unknown tags | `builders.rs:828-963` |
| Git patches | `build_git_patch` + `GitPatchMeta` | 1617 | Full tag surface (euc, series markers, commit/parent, PGP sig, committer identity) | `builders.rs:1007-1069` |
| Git issues | `build_git_issue` + `GitIssueMeta` | 1621 | Full (subject, labels, recipients) | `builders.rs:1081-1111` |
| Git status transitions | `build_git_status`, `GitStatus`, `GitStatusMeta`, `GitAppliedPatchRef::parse` | 1630–1633 | Full, incl. merge-only fields and `q`-tag relay/pubkey hints | `builders.rs:1114-1299` |
| Git pull requests | `build_git_pull_request`, `build_git_pr_update` | 1618, 1619 | Full event shape; explicitly does **not** verify commit reachability or perform network work | `builders.rs:1301-1460` |
| Workflows | `build_workflow_def`, `build_workflow_update`, `build_workflow_delete`, `build_workflow_trigger`, `build_workflow_approval` | 30620, 5, 46020, 46030/46031 | Full CRUD + trigger + approve/deny | `builders.rs:1462-1541` |
| Community moderation | `build_moderation_ban`/`unban`/`timeout`/`untimeout`/`resolve_report` | 9040–9044 | Full command surface; status↔action pairing deliberately delegated to the relay | `builders.rs:1587-1690` |
| Identity archival (NIP-IA) | `build_archive_identity_request`, `build_unarchive_identity_request` | 9035, 9036 | Full request shape; consent-path selection is relay-side | `builders.rs:1692-1823` |
| Agent observer streaming | `build_agent_observer_frame` | 24200 | Partial — validates that content *looks like* NIP-44 ciphertext but does not encrypt; encryption lives in `buzz_core::observer` | `builders.rs:243-274`; `crates/buzz-core/src/observer.rs:53-55` |
| Mention resolution | `extract_at_names`, `extract_at_mentions_with_known`, `match_names_to_profiles`, `merge_mentions`, `normalize_mention_pubkeys`, `strip_code_regions`, `extract_nostr_uris` | n/a | Full pure-function pipeline; membership/profile querying is the caller's job | `mentions.rs:1-30`, `64-387` |
| NIP-OA owner attestation | `compute_auth_tag`, `verify_auth_tag`, `parse_auth_tag` | n/a (tag) | Full sign/verify/parse with spec test vector pinned | `nip_oa.rs:146-299`; vector `nip_oa.rs:313-333`, `nip_oa.rs:487-494` |
| Event reading | `extract_channel_id` | n/a | Minimal — the crate is write-oriented; this is the only inbound-event helper | `builders.rs:816-826` |

Kind coverage: **35 distinct event kinds** across 51 public builder functions
(kinds 0, 1, 3, 5, 7, 9, 1617, 1618, 1619, 1621, 1630, 1631, 1632, 1633, 9000,
9001, 9002, 9005, 9007, 9008, 9021, 9022, 9035, 9036, 9040, 9041, 9042, 9043,
9044, 20001, 24200, 30030, 30617, 30620, 40003, 40008, 40100, 41010, 41011,
45001, 45002, 45003, 46020, 46030, 46031 — 45 kind integers, 35 unique
kind *families* if the four Git status kinds and the two approval kinds are each
counted once).

---

### 2. Explicitly stubbed / deferred behavior

| Item | Statement in code | File:line |
|---|---|---|
| Full NIP-10 threading for kind 1 | "This is intentionally simpler than the full `ThreadRef` mechanism used for channel messages — social notes use a flat reply model for now. Full NIP-10 threading (root + reply + p-tags) is deferred." | `builders.rs:732-737` |
| PR tip reachability | "this builder does no network work and does not verify reachability" | `builders.rs:1311-1315` |
| NIP-OA verification depth in builders | "Structural check only — the relay performs full NIP-OA verification." | `builders.rs:1722-1723` |
| NIP-IA consent path | "The relay verifies; this builder's job is to produce a well-formed, signed request — the relay selects the consent path (self / admin / owner)." | `builders.rs:1697-1699` |
| Moderation status/action pairing | "(`dismiss` pairs with `dismissed`, everything else with `resolved` — the relay enforces the pairing)" | `builders.rs:1648-1651` |
| Emoji palette union | "The workspace palette shown in clients is the union of every member's set, deduped by `(shortcode, url)` on read." (read side not implemented here) | `builders.rs:508-510` |
| `parse_auth_tag` skips crypto | "This is the fast path used at MCP startup — no crypto is performed." | `nip_oa.rs:249-250` |
| `GitStatusMeta.recipients` defaulting | "GitStatusMeta.recipients is the caller's responsibility (the CLI defaults it)" | `builders.rs:3157-3159` (test comment) |

---

### 3. TODO / FIXME / HACK / XXX comments

A recursive search across `crates/buzz-sdk/` (all files, including tests and the
example) for `TODO`, `FIXME`, `HACK`, and `XXX` returned **zero matches**. There
are no such markers in this crate.

The deferred-work statements listed in section 2 are ordinary doc comments, not
tagged markers.

---

### 4. Feature-flag-gated capabilities

None. There is no `[features]` table in `crates/buzz-sdk/Cargo.toml` and no
`#[cfg(feature = ...)]` anywhere in the crate (verified by search across
`src/` and `examples/`). The only conditional compilation is `#[cfg(test)]` on
the three test modules (`builders.rs:1825`, `mentions.rs:389`,
`nip_oa.rs:301`).


## Module: buzz-persona (`crates/buzz-persona`)

### Aspect: Features

### Capability inventory

| # | Capability | Entry point | Completeness | Evidence |
|---|---|---|---|---|
| F1 | Parse a `.persona.md` (YAML frontmatter + markdown body) from a string | `persona::parse_persona_md` (`crates/buzz-persona/src/persona.rs:208`) | Full | 29 unit tests in `crates/buzz-persona/src/persona.rs:331-644` covering minimal, full-field, missing/empty fields, malformed YAML, size limits, delimiter edge cases |
| F2 | Parse a `.persona.md` from disk with a pre-read size guard | `persona::parse_persona_file` (`crates/buzz-persona/src/persona.rs:262`) | Full | size check at `:263-267`; no dedicated test for the `TooLarge` path (see debt doc) |
| F3 | Frontmatter splitting with strict own-line closing delimiter | `persona::split_frontmatter` (`crates/buzz-persona/src/persona.rs:277`) | Full | scanner loop `:287-306`; tests `:482-501` |
| F4 | `provider:model-id` splitting on first colon | `persona::split_model` (`crates/buzz-persona/src/persona.rs:324`) | Full | tests `:545-563` |
| F5 | Parse `plugin.json` (OPS-superset manifest) | `manifest::parse_manifest` / `parse_manifest_file` (`crates/buzz-persona/src/manifest.rs:152`, `:190`) | Full | 14 tests `crates/buzz-persona/src/manifest.rs:195-364` |
| F6 | Load a whole pack directory (manifest + personas + instructions + `.mcp.json` + skills-dir detection) | `pack::load_pack` (`crates/buzz-persona/src/pack.rs:125`) | Full for the paths it reads; **partial** in coverage — pack-level `hooks/hooks.json` is deliberately never loaded (`crates/buzz-persona/src/pack.rs:111-112`) | 14 tests `crates/buzz-persona/src/pack.rs:447-734` |
| F7 | Path-traversal / symlink-escape defense for manifest-declared paths | `safe_resolve` (`crates/buzz-persona/src/pack.rs:323`) | Full for `personas`, `pack_instructions`, `mcp_config`. **Not applied** to hook paths (`crates/buzz-persona/src/resolve.rs:339-346`) or persona `skills` paths | tests `crates/buzz-persona/src/pack.rs:561-607` |
| F8 | Behavioral-config precedence merge (levels 3–5) | `merge::merge_behavioral_config`, `merge::resolve_persona_config` (`crates/buzz-persona/src/merge.rs:47`, `:85`) | Full | 22 tests `crates/buzz-persona/src/merge.rs:200-464` including null/empty-container semantics and shallow-replacement |
| F9 | Full pack resolution to ACP-ready output | `resolve::resolve_pack`, `resolve_loaded_pack` (`crates/buzz-persona/src/resolve.rs:108`, `:118`) | Full | 26 tests `crates/buzz-persona/src/resolve.rs:400-892` |
| F10 | Single-persona lookup by name | `resolve::resolve_persona_by_name` (`crates/buzz-persona/src/resolve.rs:186`) | Full — note it re-reads and re-resolves the entire pack (`:187`) | tests `:806-861` |
| F11 | MCP server merge (pack shared + per-persona, deterministic order) | `merge_mcp_servers` (`crates/buzz-persona/src/resolve.rs:277`) | Partial — merge works; `${VAR}` interpolation is explicitly out of scope | doc `:274`; test `:558-572` asserts literals survive |
| F12 | Env-var projection per agent runtime (`goose` vs `buzz-agent`) | `runtime_env_vars` (`crates/buzz-persona/src/resolve.rs:365`) | Full for the two known runtimes; every other runtime silently falls into the GOOSE_* branch (`:380`) | tests `:602-737`, `crates/buzz-persona/tests/e2e_env_flow.rs:35-356` |
| F13 | Lifecycle hooks | `resolve_hooks` (`crates/buzz-persona/src/resolve.rs:347`) | **Stubbed** — parsed and carried only. Field comment: "Hooks (parsed, not executed — reserved for future use, not yet wired)" (`crates/buzz-persona/src/resolve.rs:57`) | no process-spawn code anywhere in the crate |
| F14 | Skill scoping (claimed vs shared) | `pack::resolve_skills` (`crates/buzz-persona/src/pack.rs:249`) | **Stubbed / orphaned** — implemented and tested, but never called by `load_pack` or `resolve_pack`; `ResolvedPersona.skills` carries raw declared paths instead (`crates/buzz-persona/src/resolve.rs:249`), and the field is marked "reserved for future use, not yet wired" (`:60`) | grep for `resolve_skills` finds only the definition, its own tests, and no production caller |
| F15 | Pack validation with error/warning severity and exit codes | `validate::validate_pack` (`crates/buzz-persona/src/validate.rs:143`) | Partial — see gaps table below | 22 tests `crates/buzz-persona/src/validate.rs:436-963` |
| F16 | Human-readable validation output | `Display for ValidationReport` (`crates/buzz-persona/src/validate.rs:74-96`) | Full | rendered by `crates/buzz-cli/src/commands/pack.rs:26-35` (which iterates diagnostics itself rather than using `Display`) |
| F17 | `engines.buzz` version negotiation | field only (`crates/buzz-persona/src/manifest.rs:39-40`) | **Not implemented** — parsed, never compared; no semver dependency in `crates/buzz-persona/Cargo.toml:10-14` | — |
| F18 | Pack integrity (sha256 / `pack.lock`), zip/git install, pack packaging | — | **Absent** — no such code in the crate; `PERSONA_PACK_SPEC.md` §11/§13 describe it as buzz-acp/CLI responsibility | no `sha2`, `zip`, or HTTP dependency in `crates/buzz-persona/Cargo.toml` |
| F19 | Prompt templating / variable substitution | — | **Absent** — `system_prompt` is a verbatim copy (`crates/buzz-persona/src/resolve.rs:200`) | no template engine dependency |

---

### Validation feature gaps (F15 detail)

Measured against the checks the crate itself declares or that
`PERSONA_PACK_SPEC.md` §16 (PF-2) lists as remaining:

| Gap | Evidence |
|---|---|
| `skills:` paths are never checked for existence | `advisory_check_skill_names` only inspects directories that already exist (`crates/buzz-persona/src/validate.rs:365-370`); no "declared skill missing" diagnostic exists |
| `hooks:` paths are never checked for existence | no hook handling in `validate.rs` (grep for `hook` in that file matches only the `hooks_config` entry of `KNOWN_MANIFEST_KEYS` at `crates/buzz-persona/src/validate.rs:115`) |
| `SKILL.md` missing `name:`/`description:` is not an error | only `name` is read (`crates/buzz-persona/src/validate.rs:420-422`); `description` never inspected |
| Only the **first** structural fault is reported | early `return` at `crates/buzz-persona/src/validate.rs:148-151`; test comment acknowledges this: "load_pack fails on the first missing required field ... a single error is emitted" (`crates/buzz-persona/tests/integration.rs:296-299`) |
| `temperature` range is not validated | no range check in `merge.rs` or `validate.rs`; matches spec §10 which states type-only checking |
| Unknown persona-frontmatter keys produce a serde error message, not a targeted diagnostic | K1 path — `Error("pack failed to load: invalid file ...: failed to parse YAML frontmatter: unknown field ...")`; test asserts only that the key name appears (`crates/buzz-persona/src/validate.rs:707-733`) |

---

### TODO / FIXME / HACK / XXX comments

**Zero.** A grep for `TODO`, `FIXME`, `HACK`, and `XXX` across the entire
`crates/buzz-persona` directory (all `.rs` files, `Cargo.toml`, and
`PERSONA_PACK_SPEC.md`) returns no matches.

Deferred work is instead expressed as prose in doc comments. Verbatim, with locations:

| Location | Comment (verbatim) |
|---|---|
| `crates/buzz-persona/src/resolve.rs:57` | `// Hooks (parsed, not executed — reserved for future use, not yet wired)` |
| `crates/buzz-persona/src/resolve.rs:60` | `// Skills (bare names — reserved for future use, not yet wired)` |
| `crates/buzz-persona/src/resolve.rs:274` | `/// Env values are passed through as literals — no `${VAR}` interpolation.` |
| `crates/buzz-persona/src/resolve.rs:341-346` | `/// Security: we intentionally do NOT resolve these to absolute paths.` … `/// Hook paths come from untrusted persona frontmatter and could contain` / `/// `../` traversal. Since hooks are not executed in this PR, we store` / `/// them as-is. The PR that wires execution MUST validate through` / `/// `safe_resolve()` before use.` |
| `crates/buzz-persona/src/resolve.rs:225-231` | `// Version: LoadedPersona has no per-persona version field — persona files` / `// don't declare a version in frontmatter. The pack version is used as-is.` / `// If per-persona versioning is added in the future, LoadedPersona should` / `// gain `version: Option<String>` and this line should become:` / `//   lp.version.clone().unwrap_or_else(|| pack_version.to_owned())` |
| `crates/buzz-persona/src/resolve.rs:178-179` | `// Pack-level description not yet wired through PackManifestData.` |
| `crates/buzz-persona/src/pack.rs:111-112` | `// hooks_config is intentionally omitted: hooks are a runtime concern loaded` / `// separately by buzz-acp, not a pack-parsing concern.` |
| `crates/buzz-persona/src/persona.rs:230` | `// Fix #1: enforce non-empty required strings` |
| `crates/buzz-persona/src/persona.rs:263` | `// Fix #4: check file size before reading to avoid large allocations` |
| `crates/buzz-persona/src/merge.rs:9` | `/// Levels 1–2 (operator env vars, desktop UI) are resolved at runtime.` |

Note on `crates/buzz-persona/src/resolve.rs:178-179`: the comment says the pack
description is "not yet wired", but the code on the next line does read it
(`loaded.manifest.description.clone().unwrap_or_default()` at `:180`) and
`PackManifestData.description` exists (`crates/buzz-persona/src/pack.rs:106`) and is
populated (`:143`). The comment appears stale.

Two `// Fix #N:` markers (`persona.rs:230`, `persona.rs:263`) reference an external
review-item numbering that is not defined anywhere in the repo.

---

### Spec-vs-implementation deltas

`PERSONA_PACK_SPEC.md` ships inside this crate and describes behavior that lives partly
outside it. Items the spec marks as implemented-elsewhere or planned, cross-checked
against this crate's code:

| Spec item | Spec location | State in this crate |
|---|---|---|
| Skill copying to `$AGENT_CWD/.agents/skills/` | `PERSONA_PACK_SPEC.md:359-361` ("planned for a future release") | absent; `resolve_skills` computes the mapping but nothing copies |
| MCP `${VAR}` interpolation | `PERSONA_PACK_SPEC.md:512-514` ("planned but not yet implemented") | confirmed absent (`crates/buzz-persona/src/resolve.rs:274`) |
| Hook execution | `PERSONA_PACK_SPEC.md:551-552` ("parsed and validated at pack load time but not yet executed") | parsed only; pack-level hooks not even parsed (`crates/buzz-persona/src/pack.rs:111-112`) |
| `engines.buzz` rejection of too-new packs | `PERSONA_PACK_SPEC.md:113-114` | not implemented (F17) |
| `buzz pack validate` — "Remaining: verify `skills:` and `hooks:` paths exist; error on `SKILL.md` missing `name:` or `description:`" | `PERSONA_PACK_SPEC.md:1139` (PF-2) | confirmed still missing (gap table above) |
| Unknown keys in `defaults` are validation warnings; unknown persona frontmatter keys are hard errors | `PERSONA_PACK_SPEC.md:803-807` | matches implementation (K6, A8) |
| V6 namespaced `buzz:` block is unsupported | `PERSONA_PACK_SPEC.md:1108` | consistent — `buzz` is not a `Frontmatter` field, so such a file fails `deny_unknown_fields` |


## Module: buzz-ws-client (`crates/buzz-ws-client`)

### Aspect: Features

Library crate, no binary targets (`crates/buzz-ws-client/Cargo.toml:1`–`7`, no
`[[bin]]`/`[features]` sections). Purpose per the module docs: connect, NIP-42
authenticate, publish, and read relay messages.

---

### 1. Capability matrix

| Capability | State | Evidence |
|---|---|---|
| Connect to a relay over WebSocket (ws/wss via `MaybeTlsStream`) | Full | `connection.rs:48`–`65`, `:8`, `:14` |
| NIP-42 authentication (await challenge → sign → send AUTH → await OK) | Full | `connection.rs:70`–`93`; `message.rs:174`–`190` |
| Connect + authenticate in one call | Full | `connection.rs:37`–`45` |
| Attach an extra authorization tag (documented as "NIP-OA") to the AUTH event | Full (single tag only) | `connection.rs:36`, `message.rs:169`–`186` |
| Publish a signed event and await its `OK` | Full | `connection.rs:96`–`101` |
| One-shot publish helper with overall timeout | Full | `connection.rs:277`–`294` |
| Read the next relay message with a caller-set timeout | Full | `connection.rs:104`–`112`, `:128`–`155` |
| Out-of-order message buffering while awaiting a specific frame | Full | `connection.rs:28`, `:205`, `:257`, `:259` |
| Parse the seven NIP-01/NIP-42/NIP-45 relay message types | Full for `EVENT`, `OK`, `EOSE`, `CLOSED`, `NOTICE`, `AUTH`, and `COUNT` (the seventh was added by the post-analysis sync; six at report time) | `message.rs:70`–`162` |
| WebSocket keepalive (Ping → Pong) | Full | `connection.rs:148`–`150`, `:208`–`:210`, `:262`–`:264` |
| Graceful close | Full | `connection.rs:115`–`118` |
| Send arbitrary JSON frames (escape hatch used for `REQ`/`CLOSE`/`COUNT`) | Full, but untyped | `connection.rs:121`–`126` |
| Subscription management (typed `REQ`/`CLOSE`, filter types, EOSE-driven collection) | **Absent** — no such API; callers hand-roll it (e.g. `crates/buzz-test-client/src/lib.rs:154`, `:160`) | no `REQ`/`CLOSE` literal anywhere in the crate |
| `COUNT` message support | **Status: resolved** by the post-analysis sync (was: absent — no `RelayMessage` variant, so a `COUNT` reply failed as `UnexpectedMessage`). A `Count { subscription_id, count }` variant (`message.rs:40`–`46`) and a `"COUNT"` parse arm reading `arr[2]["count"]` as `u64` (`:147`–`162`) now handle it; a malformed payload still falls back to `UnexpectedMessage`. | `message.rs:40`–`46`, `:147`–`162` |
| Reconnect / retry / backoff | **Absent** | no retry or backoff code in `connection.rs` (verified over the whole file) |
| Automatic re-auth on mid-session challenge | **Partial** — challenge is captured and buffered, but re-auth is not triggered; caller must call `authenticate` again | `connection.rs:255`–`258` vs `:161` |
| Rejection surfacing for published events | **Partial** — `accepted: false` is returned to the caller rather than raised as an error, and the `EventRejected` variant exists unused | `connection.rs:96`–`101`; `error.rs:40` |
| Binary frame handling | **Ignored by design** (`_ => {}`) | `connection.rs:152`, `:212`, `:266` |
| Metrics / instrumentation | **Minimal** — three `debug!` lines, no spans, no counters | `connection.rs:57`, `:91`, `:123` |
| Configuration surface (env vars, cargo features) | **Absent** — see the configuration aspect | no `env::var`, no `cfg(feature = …)` in the crate |

---

### 2. Completeness notes

- The public API is closed over a narrow flow: connect → auth → publish → read →
  close. Everything a NIP-01 subscriber needs beyond that is delegated to
  `send_raw` + `next_event` (`connection.rs:121`, `:104`).
- `next_event` re-checks the buffer immediately before delegating to `recv_one`,
  which checks it again (`connection.rs:108`–`111` then `:129`–`131`) — behaviourally
  redundant but harmless.
- The `Auth` challenge cap (1024 bytes) exists on only one of the three challenge
  intake paths (`connection.rs:198`; the paths at `:161` and `:170` are uncapped).

---

### 3. TODO / FIXME / HACK / XXX markers

**Zero.** A case-insensitive search for `TODO`, `FIXME`, `HACK`, `XXX`,
`unimplemented`, and `todo` across `crates/buzz-ws-client/` returned no matches
(exit code 1, no output). There are also no `#[deprecated]`, `#[allow(dead_code)]`,
or `unimplemented!()`/`todo!()` markers. The only lint attributes present are
`#![deny(unsafe_code)]` (`lib.rs:1`) and `#[allow(clippy::result_large_err)]`
(`message.rs:61`).

---

### 4. Tests shipped with the crate

Three unit tests, all compile-time assertions on the timeout floors — no I/O, no
mock relay, no parser tests:

| Test | Asserts | file:line |
|---|---|---|
| `auth_challenge_timeout_meets_floor` | `AUTH_CHALLENGE_TIMEOUT_SECS >= 20` | `connection.rs:300`–`303` |
| `auth_ok_timeout_meets_floor` | `AUTH_OK_TIMEOUT_SECS >= 20` | `connection.rs:305`–`308` |
| `publish_ok_timeout_meets_floor` | `PUBLISH_OK_TIMEOUT_SECS >= 30` | `connection.rs:310`–`313` |

Module gate at `connection.rs:296`–`298`. Each body uses an inline `const { assert!(…) }`
block, so the check is evaluated at compile time rather than at run time.


## Module: buzz-db (`crates/buzz-db`)

### Features

Capabilities the crate provides, with completeness read from the code (not from
docs). "Complete" means the module has both read and write paths and callers in
the workspace surface; "read-only", "partial", and "schema-only" are noted.

| # | Capability | State | Evidence |
|---|-----------|-------|----------|
| 1 | **Nostr event store** — insert (dedup), scoped query with 14 pushdown filters + the kind-30175 visibility clause added by `ab3af828` (one more optional predicate block: 11 → 12 `if let Some(…) = q.…` blocks in `query_events`), COUNT, scoped id lookup, batch id lookup, soft delete, coordinate delete | complete | `insert_event` `crates/buzz-db/src/event.rs:256`, `query_events` `:318`, `count_events` `:598`, `get_event_by_id` `:909`, `get_events_by_ids` `:989`, `soft_delete_event` `:744`, `soft_delete_by_coordinate` `:771` |
| 2 | **Monthly range partitioning** of `events` and `delivery_log` with startup/cron partition creation | complete for creation; **no drop/detach/rotation-out path exists** | `crates/buzz-db/src/partition.rs:15-73`; partitions declared `migrations/0001_initial_schema.sql:237-252`, `:343-354` |
| 3 | **Replaceable-event replacement** — NIP-16 addressable (`kind, pubkey, channel_id`) and NIP-33 parameterized (`kind, pubkey, d_tag`), each under an advisory lock with deterministic tie-breaks | complete | `crates/buzz-db/src/lib.rs:3306`, `:3628` |
| 4 | **NIP-RS read-state retention** — exact-cardinality classification, payload hard delete, durable ordering watermark, mixed-version DB triggers, hard-delete opt-in fence | complete | `crates/buzz-db/src/lib.rs:3672-3796`; `migrations/0007`, `0009`, `0010`, `0011` |
| 5 | **Mesh-status heartbeat retention** (kind 30003 `buzz-mesh-member-status:*`) — one physical row per coordinate | complete | `crates/buzz-db/src/lib.rs:3688-3695`; `migrations/0019_mesh_status_retention.sql` |
| 6 | **Channel CRUD + membership** with transactional role enforcement, soft delete, last-owner protection, archive/unarchive, topic/purpose, canvas | complete | `crates/buzz-db/src/channel.rs` (26 public fns) |
| 7 | **Ephemeral channels with TTL** — deadline init, in-commit refresh, advisory-lock-ordered TTL transitions, idempotent reaper | complete | `crates/buzz-db/src/channel.rs:1387`; `migrations/0022`, `0024` |
| 8 | **Threading** — parent/root/depth metadata, materialized `reply_count`/`descendant_count`/`last_reply_at`, keyset-paginated replies, channel window with server `has_more`, batched participants | complete | `crates/buzz-db/src/thread.rs` |
| 9 | **Reactions** — TOCTOU-free upsert with new/reactivate/duplicate semantics, soft delete, grouped and bulk reads, single-transaction kind:7 coupling | complete | `crates/buzz-db/src/reaction.rs`; `crates/buzz-db/src/event.rs:1242` |
| 10 | **DMs** — participant-hash identity, 2–9 participants, idempotent open, per-user hide/unhide, hidden-DM listing | complete | `crates/buzz-db/src/dm.rs` |
| 11 | **Home feed** — mentions / needs-action / activity over the normalized `event_mentions` index with a hard 100-row cap | complete | `crates/buzz-db/src/feed.rs` |
| 12 | **Mention index** (`event_mentions`) — multi-row upsert built from validated `p` tags, best-effort (failures logged, never block the event) | complete | `crates/buzz-db/src/lib.rs:97-165`, call sites `:1085-1090`, `:1391-1396`, `:1419-1428` |
| 13 | **User profiles** — upsert, profile update with empty→NULL semantics, NIP-05 lookup, LIKE-escaped search with ranking, agent ownership, channel-add policy | complete | `crates/buzz-db/src/user.rs` |
| 14 | **API tokens** — hashed storage, atomic 10-token quota, community-scoped lookup (incl. revoked), listing, single/bulk revoke, `last_used_at` touch | complete | `crates/buzz-db/src/api_token.rs`, `crates/buzz-db/src/lib.rs:2327-2461` |
| 15 | **Workflows / runs / approvals** — owner-and-channel-guarded upsert, bounded lists, run lifecycle with `started_at`/`completed_at` stamping, SHA-256 approval tokens, single-decision approval updates | complete; 5 functions are documented as having **no current callers** (`create_workflow`, `update_workflow`, `update_workflow_status`, `set_workflow_enabled`, `delete_workflow`) | `crates/buzz-db/src/workflow.rs:275`, `:619`, `:653`, `:683`, `:714` |
| 16 | **Cron fire claims** — at-most-once `(community, workflow, scheduled_for)` claim, DB-authoritative interval anchor, run-id attach, retention prune | complete | `crates/buzz-db/src/workflow.rs:496-611` |
| 17 | **NIP-ER reminders** — due-set query with per-tenant host provenance, exactly-once claim, stamped release | complete | `crates/buzz-db/src/event.rs:1334-1472` |
| 18 | **Relay membership (NIP-43)** — CRUD, invite claim with policy-acceptance evidence, owner-protected removal/role change, owner bootstrap, atomic ownership transfer with a 3-community cap, allowlist backfill, locked snapshot publication | complete | `crates/buzz-db/src/relay_members.rs`; snapshot at `crates/buzz-db/src/lib.rs:3438-3626` |
| 19 | **Pubkey allowlist** — membership check, enforcement-active check, add/remove/list | complete (inline on `Db`) | `crates/buzz-db/src/lib.rs:2826-2905` |
| 20 | **Archived identities (NIP-IA)** — community-scoped archive/unarchive/list, idempotent | complete | `crates/buzz-db/src/archived_identities.rs` |
| 21 | **Community moderation** — NIP-56 report ingest/queue/resolve, ban + timeout state with expiry-aware reads, audit action log | complete | `crates/buzz-db/src/moderation.rs` |
| 22 | **Deployment-admin moderation plane** — keyset-paginated global report list, global feedback list, single-row fetches | read-only by construction | `crates/buzz-db/src/admin_moderation.rs` |
| 23 | **Product feedback sidecar** — deployment-wide idempotent insert by event id, newest-first list | complete | `crates/buzz-db/src/product_feedback.rs` |
| 24 | **Git repo-name registry (NIP-34)** — atomic per-community reservation, owner classification, quota count, owner-scoped release | complete | `crates/buzz-db/src/git_repo.rs` |
| 25 | **NIP-PL push leases + wake outbox + match queue** — signed-lease acceptance with ordering/quota/endpoint rules, gate-guarded matcher enqueue, batched wake enqueue, fenced claim/complete/retry/fail, send-time revalidation, generation-scoped endpoint disable, pruning | complete for the relay-owned side | `crates/buzz-db/src/push.rs` |
| 26 | **Read-replica routing with a freshness fence** — commit-time floor guard, ordered LSN handshake, staleness expiry, two-part startup verification, per-query routing decisions | complete | `crates/buzz-db/src/replica_fence.rs`; `crates/buzz-db/src/lib.rs:2004`, `:2063` |
| 27 | **Per-community usage rollups** for Prometheus gauges (11 queries) plus a detached-session advisory leader lock | complete | `crates/buzz-db/src/usage.rs`; `crates/buzz-db/src/lib.rs:517-535` |
| 28 | **Embedded migrations + tenant-isolation lint harness** — 24 migrations, pre-flight ambiguity guard, post-migration floor-guard verification, SQL-parsing lint tests | complete | `crates/buzz-db/src/migration.rs` |
| 29 | **Pool observability** — `ping`, writer/replica pool stats | complete | `crates/buzz-db/src/lib.rs:485-511` |
| 30 | **Community lifecycle** — host resolve (active / management), icon get/set, ensure-configured, atomic create-with-owner, archive/unarchive, channel→community reverse resolution (single + batch) | complete | `crates/buzz-db/src/lib.rs:656-1077` |
| 31 | **d_tag backfill** maintenance for legacy NIP-33 rows | complete, idempotent, **unscoped by community** | `crates/buzz-db/src/lib.rs:2810-2824` |
| 32 | **Subscriptions / delivery_log / audit_log / rate_limit_violations** | **schema-only in this crate** — tables exist, no Rust read/write path | `migrations/0001_initial_schema.sql:304`, `:329`, `:498`, `:606`; no matching module (verified by search) |
| 33 | **push_gateway_* authority tables** (6) | **schema-only in this crate** — created by `migrations/0015`, referenced only by the lint allowlist in tests | `migrations/0015_push_gateway_authority.sql`; `crates/buzz-db/src/migration.rs:343-348` |

---

#### TODO / FIXME / HACK / XXX inventory

**Zero occurrences.** A case-insensitive search for `TODO`, `FIXME`, `HACK`, and
`XXX` across `crates/buzz-db/**/*.rs`, all 24 files in `migrations/`, and
`schema/schema.sql` returns no matches. Deferred work is instead expressed as
prose in doc comments; the concrete instances are:

| Marker text (verbatim) | File:line |
|------------------------|-----------|
| `/// creation path is [`upsert_workflow`] via event ingest. (No current callers.)` | `crates/buzz-db/src/workflow.rs:275` |
| `/// matching lags the change by up to the cache TTL. (No current callers.)` | `crates/buzz-db/src/workflow.rs:619` |
| `/// [`update_workflow`]. (No current callers.)` | `crates/buzz-db/src/workflow.rs:653` |
| `/// on [`update_workflow`]. (No current callers.)` | `crates/buzz-db/src/workflow.rs:683` |
| `/// `channel_id` needed for invalidation. (No current callers.)` | `crates/buzz-db/src/workflow.rs:714` |
| `/// `cursor` is reserved for future keyset pagination (currently unused).` | `crates/buzz-db/src/reaction.rs:279` (parameter `_cursor` at `:286`) |
| `/// NOTE: The primary increment path is inlined inside [`insert_thread_metadata`]'s` … `/// This standalone version exists for future use cases` (function carries `#[allow(dead_code)]`) | `crates/buzz-db/src/thread.rs:243-250` |
| `// Run one query per event. For typical message-list sizes (<=100 events)` … `// a single-query approach with dynamic IN clauses over` / `// composite keys can be added later if needed.` | `crates/buzz-db/src/reaction.rs:373-375` |
| `//  If query performance` / `// degrades, add a composite index like `(pubkey, kind, created_at DESC, id ASC)`.` | `crates/buzz-db/src/event.rs:532-534` |
| `//! At scale these can become recurring partition scans; if that becomes a` / `//! problem, move them to a maintained rollup table and drop the interval.` | `crates/buzz-db/src/usage.rs:8-10` |
| `/// For large channel sets, consider pagination.` | `crates/buzz-db/src/channel.rs:602` |
| `-- the search lane can` / `-- revisit the config behind evidence.` (FTS `'simple'` config) | `migrations/0001_initial_schema.sql:200-201` |


## Module: buzz-auth (`crates/buzz-auth`)

### Features / Capabilities

Completeness legend: **full** = implemented and exercised by tests; **partial** =
implemented but with a documented or observable gap; **interface-only** = trait or
type defined here, behaviour lives elsewhere; **stubbed** = test-only
implementation.

| # | Capability | Completeness | Evidence | Notes |
|---|-----------|--------------|----------|-------|
| 1 | NIP-42 challenge generation | full | `crates/buzz-auth/src/nip42.rs:38-41` | 32 CSPRNG bytes via `rand::random`, hex-encoded; test pins 64 chars + uniqueness (`:102-109`) |
| 2 | NIP-42 AUTH event verification (kind, sig, challenge, relay, timestamp) | full | `crates/buzz-auth/src/nip42.rs:47-86` | 5 rejection tests + 2 normalisation tests |
| 3 | Relay-URL normalisation for AUTH | full | `crates/buzz-auth/src/nip42.rs:19-33` | loopback aliasing + trailing-slash stripping; `ws` vs `wss` not aliased |
| 4 | `AuthService` async wrapper with `spawn_blocking` offload | full | `crates/buzz-auth/src/lib.rs:118-143` | 3 async tests (`:199-242`) |
| 5 | NIP-98 HTTP auth verification (8-step) | full | `crates/buzz-auth/src/nip98.rs:55-131` | 11 tests including payload-hash and loopback-distinctness |
| 6 | NIP-98 body-hash binding (`payload` tag) | partial | `crates/buzz-auth/src/nip98.rs:117-127` | only enforced when the tag **and** a body are both present; presence is the caller's responsibility (`crates/buzz-relay/src/api/bridge.rs:99-112`) |
| 7 | NIP-98 replay protection | interface-only | trait `crates/buzz-auth/src/nip98_replay.rs:64-104`; keys `:114-121`; constants `:46`, `:59` | production Redis impl is `crates/buzz-pubsub/src/nip98_replay.rs:34`; this crate ships only `AlwaysFreshReplayGuard` (`:126-139`) |
| 8 | Scope model + wire-format round-trip | full | `crates/buzz-auth/src/scope.rs:16-178` | 16 known variants + `Unknown` passthrough; 5 tests |
| 9 | Scope granting on NIP-42 | full (but coarse) | `crates/buzz-auth/src/lib.rs:136-142` | grants all 16 scopes unconditionally; no per-connection differentiation |
| 10 | Scope checking (`require_scope`, `has_scope`) | full | `crates/buzz-auth/src/access.rs:60-69`, `crates/buzz-auth/src/lib.rs:84-86` | exact-match only; no hierarchy/wildcards. `require_scope` has no external caller |
| 11 | Channel access checking | interface-only | trait `crates/buzz-auth/src/access.rs:31-57` | **no production implementor anywhere in the repo** (see debt aspect); only `MockAccessChecker` (`:135-151`) |
| 12 | Combined scope+membership helpers | full but unused | `crates/buzz-auth/src/access.rs:72-101` | 4 tests; zero callers outside this crate |
| 13 | Rate limiting | interface-only | trait `crates/buzz-auth/src/rate_limit.rs:168-194` | production Redis impl is `crates/buzz-pubsub/src/rate_limiter.rs:99-121`, wired in `buzz-relay` — see security aspect verdict |
| 14 | Rate-limit tier configuration | full (parse) / partial (consumption) | `crates/buzz-auth/src/rate_limit.rs:86-144` | all 7 fields deserialise with defaults; only 4 are read by any consumer (see configuration aspect) |
| 15 | Rate-limit / replay Redis key construction | full | `crates/buzz-auth/src/rate_limit.rs:201-215`, `crates/buzz-auth/src/nip98_replay.rs:114-121` | community-prefixed for pubkey/event keys, global for IP; lowercase invariant tested |
| 16 | Test doubles (`MockAccessChecker`, `AlwaysAllowRateLimiter`, `AlwaysFreshReplayGuard`) | stubbed by design | `crates/buzz-auth/src/access.rs:108`, `crates/buzz-auth/src/rate_limit.rs:219`, `crates/buzz-auth/src/nip98_replay.rs:127` | all `#[cfg(any(test, feature = "test-utils"))]`; none referenced outside this crate |
| 17 | Dev-only username→pubkey derivation | full but orphaned | `crates/buzz-auth/src/lib.rs:159-167` | `#[cfg(any(test, feature = "dev"))]`; no caller in the repo, and no test either |
| 18 | Error taxonomy | full | `crates/buzz-auth/src/error.rs:9-59` | 10 variants; 2 (`Nip98Replay`, `PubkeyMismatch`) are never constructed in this crate |

---

### Capabilities explicitly absent

| Absent capability | Evidence |
|-------------------|----------|
| JWT / OAuth token validation, token minting, refresh | crate doc "No JWT validation, no token management, no IdP runtime dependency" (`crates/buzz-auth/src/lib.rs:16`, `:98`); no such types in any of the 8 source files |
| Session or credential persistence | no `sqlx`, no DB dependency in `crates/buzz-auth/Cargo.toml:14-26` |
| Any network I/O | no HTTP/WS client dependency; see integrations aspect |
| Per-channel scope narrowing | `AuthContext.channel_ids` is documented "reserved for future per-channel access control" and always constructed as `None` (`crates/buzz-auth/src/lib.rs:69-72`, `:139`) |
| Constant-time comparison of challenge / payload hash | plain `!=` on `&str` (`crates/buzz-auth/src/nip42.rs:64`, `crates/buzz-auth/src/nip98.rs:122`); no `subtle` dependency |
| NIP-98 → `AuthContext` construction | `verify_nip98_event` returns a bare `PublicKey` (`crates/buzz-auth/src/nip98.rs:60`), contradicting the crate-doc invariant at `crates/buzz-auth/src/lib.rs:15` |

---

### TODO / FIXME / HACK / XXX comments

A case-insensitive search of the entire `crates/buzz-auth/` directory for
`TODO`, `FIXME`, `HACK`, and `XXX` returns **zero matches**. There are no
in-code work markers in this crate.

Deferred-work statements are instead written as prose in doc comments. The ones
that carry an explicit "not yet / future / v2 / deferred" claim, verbatim:

| Marker text (verbatim) | file:line |
|------------------------|-----------|
| `/// Channel restriction (reserved for future per-channel access control).` | `crates/buzz-auth/src/lib.rs:69` |
| `/// Reserved for future use. Not currently enforced by git HTTP routes —` / `/// those use NIP-98 auth directly. Will be enforced when collaborator` / `/// access (read-only, maintainer) is added in v2.` | `crates/buzz-auth/src/scope.rs:47-49` |
| `/// Enforced for kind:30617/30618 events via WebSocket ingest, but NOT` / `/// enforced by git HTTP push routes (which use NIP-98 + owner check).` / `/// Full enforcement deferred to v2 collaborator model.` | `crates/buzz-auth/src/scope.rs:53-55` |
| `//! ⚠️ Fixed windows allow up to 2× burst at boundaries. Upgrade to sliding` / `//! window or token bucket for strict limiting.` | `crates/buzz-auth/src/rate_limit.rs:6-7` (repeated at `:165-167`) |
| `/// ⚠️ The fixed-window algorithm used by the Redis implementation allows up to 2×` / `/// burst at window boundaries. Upgrade to a sliding window or token bucket if strict` / `/// per-second limiting is required.` | `crates/buzz-auth/src/rate_limit.rs:165-167` |
| `/// If per-(community, IP) caps are ever needed as a tenant-fairness signal, that` / `/// belongs in an additive \`LimitType\` keyed on \`(community, ip)\`, not in this trait.` | `crates/buzz-auth/src/rate_limit.rs:161-163` |
| `/// # ⚠️ SECURITY — Dev/test only` … `/// and **must never be compiled into a production release build**.` | `crates/buzz-auth/src/lib.rs:152-155` |
| `// Set later by relay membership gate if NIP-OA` | `crates/buzz-auth/src/lib.rs:141` |


## Module: buzz-pubsub (`crates/buzz-pubsub`)
### Aspect: Features

`buzz-pubsub` is a horizontal-scaling substrate, not a user-facing feature area.
It supplies six capabilities that make a multi-pod relay behave like one relay.

---

### F-PS-1 — Community-scoped realtime event fan-out

Cross-pod delivery of signed Nostr events. A pod publishes to
`buzz:{community}:channel:{id}` or `buzz:{community}:global` (`topic.rs:45-50`),
and every pod holding local interest re-broadcasts to its WebSocket sessions
through a `broadcast::channel(4096)` (`lib.rs:126`, forward at `subscriber.rs:165`).

Architecture as documented in the crate header (`lib.rs:8-16`): a
`deadpool-redis` pool serves all imperative commands, while a **dedicated**
`redis::aio::PubSub` connection — explicitly not from the pool because it is
stateful (`lib.rs:19-20`) — handles subscriptions. Obtained at
`subscriber.rs:85-87` and split into sink/stream so commands and messages can be
serviced from one `tokio::select!` (`subscriber.rs:109-171`).

Relay integration: `subscribe_local` consumers at `buzz-relay/src/main.rs:822` and
`handlers/event.rs:1644`; publishes flow through `PubSubManager::publish_event`
(`lib.rs:322-330`).

### F-PS-2 — Dynamic, refcounted subscription scoping

Rather than subscribing to a firehose, each pod subscribes only to topics with
live local interest. `retain_topic` / `release_topic` (`lib.rs:192`, `:215`)
maintain a refcount map (`subscriber.rs:21`) and emit `Subscribe` /
`UnsubscribeIfIdle` commands (`subscriber.rs:26-31`) to the pub/sub task.

Two properties make this safe:
- **Debounced teardown** (default 500 ms, `lib.rs:82`) with a re-check of the
  refcount at execution time (`subscriber.rs:123-130`), so subscription churn from
  a client re-issuing `REQ` doesn't thrash Redis.
- **Reconnect reconciliation** — the refcount map, not the connection, is the
  source of truth; a fresh connection re-subscribes from a snapshot before
  processing messages (`subscriber.rs:90-101`).

Wired into the relay's subscription lifecycle at `handlers/req.rs:251`, `:256`,
`handlers/close.rs:21`, `handlers/side_effects.rs:81`, `connection.rs:268`, and the
global-topic retains at `handlers/event.rs:1683`, `:1687`.

### F-PS-3 — Presence (online/away) with TTL

`SET buzz:{community}:presence:{pubkey_hex} <status> EX 90` (`presence.rs:36-43`),
TTL const `presence.rs:16`. Designed around a 30 s client heartbeat with 3× TTL
headroom so a single missed beat doesn't flap (`presence.rs:4-6`). Clean disconnect
deletes eagerly (`presence.rs:52-56`).

Reads: single (`presence.rs:62-72`) and bulk `MGET` returning `pubkey_hex → status`
for present keys only (`presence.rs:74-95`). The bulk path is what the relay
actually uses, at `buzz-relay/src/api/bridge.rs:1954`; multi-tenant isolation of
this exact call is pinned by conformance tests
(`crates/buzz-test-client/tests/conformance_multitenant.rs:2371`, `:2484`).
Writes come from the relay event handler (`handlers/event.rs:798`) and clears from
`connection.rs:280` / `handlers/event.rs:793`.

Capability limit: there is **no per-community presence index** (no `SET`/`ZSET` of
online members). `get_presence_bulk` requires the caller to already know the
candidate pubkey list, so "list everyone online in this community" is not
answerable from this crate.

### F-PS-4 — Cross-pod cache invalidation

Each pod keeps in-memory (moka) membership / accessible-channels / visibility
caches with a 10 s TTL; without this feature a membership change would stay stale on
every pod except the writer for up to that TTL (`cache_invalidation.rs:4-9`).

Four drop operations, each mirroring one relay-local `invalidate_*` call
(`cache_invalidation.rs:58-80`): `Membership { channel_id, pubkey }`,
`AccessibleAll`, `Visibility { channel_id }`, `ChannelDeleted`.

Delivery is a wildcard `psubscribe "buzz:*:cache-invalidate"`
(`cache_invalidation.rs:27`, `:139`) with per-message community demux
(`cache_invalidation.rs:38-51`, `:144-148`). Publisher: `buzz-relay/src/state.rs:970`.

The design invariant is that the message is a *pure key drop*, never an
authorization decision — the per-event access gate remains the enforcement point
and the next read re-fetches from the DB (`cache_invalidation.rs:11-14`).

### F-PS-5 — Cross-pod connection control (live ban enforcement)

Solves the "banned member's socket is on another pod" problem
(`conn_control.rs:3-6`). Two commands (`conn_control.rs:56-73`):
- `DisconnectPubkey { pubkey, event_id, reason }` — carries enough context to
  reproduce the same NIP-01 `OK` frame the origin pod sent, so the member learns
  why regardless of which pod held the socket (`conn_control.rs:62-65`).
- `DisconnectCommunity` — drops every socket bound to the carrying community.

Producers: `buzz-relay/src/state.rs:1034` (`DisconnectPubkey`), `:1066`
(`DisconnectCommunity`), both published via `publish_conn_control`
(`state.rs:1044`, `:1066`). Consumer: the subscriber spawned at
`buzz-relay/src/main.rs:366`, receiving on `subscribe_conn_control`
(`main.rs:903`) and dispatching both variants at `main.rs:908` and `:913`.

Deliberately a separate channel from F-PS-4 so that the cache channel's
idempotent-hint invariant is preserved (`conn_control.rs:12-18`). The DB ban row is
the durable backstop: a dropped message still results in refusal at the next auth
attempt (`conn_control.rs:18-21`).

### F-PS-6 — HA primitives borrowed by `buzz-auth` seams

Two trait implementations that let single-pod auth logic work across pods:

- **`RedisRateLimiter`** (`rate_limiter.rs:88-120`) — fixed-window `INCR`+`EXPIRE`
  via one atomic Lua script (`rate_limiter.rs:24-31`), with self-repair for
  TTL-less keys left by an older non-atomic implementation (`rate_limiter.rs:57-70`).
  Constructed at `buzz-relay/src/state.rs:712`, held as
  `admission_rate_limiter` (`state.rs:584`), and enforced for WS `EVENT`/`REQ`/`COUNT`
  at `buzz-relay/src/connection.rs:593-648`.
- **`RedisNip98ReplayGuard`** (`nip98_replay.rs:23-101`) — atomic set-if-absent
  seen-set for NIP-98 HTTP auth, described as the "§5 pre-build gate for
  multi-tenant HA replay protection" (`nip98_replay.rs:4-6`). Constructed at
  `buzz-relay/src/state.rs:711`; two-pod behaviour simulated in relay tests at
  `api/bridge.rs:2275-2276`.

### Documented-but-absent capability: typing indicators

The crate description advertises typing indicators —
`description = "Redis pub/sub fan-out, presence, and typing indicators for Buzz"`
(`Cargo.toml:8`) — and a doc comment reading "Typing indicator tracking in Redis."
sits at `lib.rs:43`. **No typing module, key, or function exists.** The module list
ends at `pub mod topic` (`lib.rs:42`), and the doc comment at `:43` is
mis-attached to `pub use error::PubSubError;` (`lib.rs:44`). A repo-wide grep for
typing-related Redis operations (`ZADD`, `ZREMRANGE`, `typing_key`, `mod typing`)
across `crates/**/*.rs` returns nothing. Feature F-PS-7 does not exist; see the
debt file.


## Module: buzz-search (`crates/buzz-search`)

### Features

#### Capability inventory

| Capability | Present | Where |
|---|---|---|
| Community-scoped full-text search over `events.content` (via `search_tsv`) | yes | `query.rs:216-323` |
| Word/lexeme search with Postgres websearch grammar (`FullText`) | yes | `query.rs:142-146` |
| Typeahead prefix search on the trailing token (`Prefix`) | yes | `query.rs:147-176` |
| Relevance ranking (`ts_rank_cd`) surfaced to callers as `SearchHit.rank` | yes | `query.rs:236`, `query.rs:117` |
| Channel scoping with four explicit variants | yes | `query.rs:41-53`, `query.rs:248-264` |
| Kind pushdown | yes | `query.rs:267-273` |
| Author pushdown | yes | `query.rs:275-281` |
| `since` / `until` time-window pushdown | yes | `query.rs:283-293` |
| Soft-delete exclusion | yes | `query.rs:242` |
| Offset pagination with clamped page/per-page | yes | `query.rs:224-231`, `query.rs:295-298` |
| Empty-query short-circuit (no DB roundtrip) | yes | `query.rs:217-222` |
| Input hardening (NUL scrub, 4096-char cap) | yes | `query.rs:179-194` |
| Row-decode length validation for `id`/`pubkey` | yes | `query.rs:306-311` |

#### Deliberately absent

| Not present | Evidence |
|---|---|
| Any write/index/delete path | Only SQL verb in the crate is `SELECT` (`query.rs:234`); tests do their own `INSERT`/`UPDATE` directly (`tests/fts_integration.rs:107, 130, 690`). No `INSERT`/`UPDATE`/`DELETE` in `src/`. |
| Total result count / `found` | `SearchResult` has only `hits` and `page` (`query.rs:122-127`) |
| Keyset (cursor) pagination | only `LIMIT`/`OFFSET` (`query.rs:295-298`) |
| Highlighting / snippets (`ts_headline`) | not referenced anywhere in the crate |
| Faceting, aggregation, suggestions, synonyms, stemming | `'simple'` config = no stemming/stopwords (`query.rs:143`, `173`); no other features present |
| Tag (`#p`, `#e`, `#h`, `#d`) or `ids` pushdown | `SearchQuery` has no such fields (`query.rs:73-99`); caller re-filters (`crates/buzz-relay/src/api/bridge.rs:1582-1592`) |
| Authorization / membership enforcement | none in crate; see security doc |
| Retry, timeout, circuit-breaker, tracing/metrics | no `tracing`, `tokio::time`, or retry code in `src/` |
| Typesense client | no dependency (`Cargo.toml:10-15`); only two historical mentions in doc comments (`query.rs:20`, `query.rs:46`) |

#### Completeness assessment

The query side is functionally complete for the two callers in the repo (WS NIP-50
`REQ` at `crates/buzz-relay/src/handlers/req.rs:504-611` and the HTTP `/query`
bridge at `crates/buzz-relay/src/api/bridge.rs:1628-1698`). Both use the crate as
a candidate generator and then re-fetch + re-authorize, which is the documented
contract (`lib.rs:15-18`, `query.rs:3-9`).

Two rough edges that are visible in code rather than declared as gaps:

1. `per_page` handling has a redundant intermediate binding: `per_page` is clamped
   to `1..=500` at `query.rs:224`, then `per_page_actual` re-derives the value and
   substitutes `PER_PAGE_DEFAULT` when the *raw* input was `0`
   (`query.rs:225-229`). The clamp at `224` already maps `0 → 1`, so the `0` branch
   exists only because it inspects `query.per_page` again. Net behavior: `0 → 100`,
   `1..=500 → as-is`, `>500 → 500`.
2. The `search()` doc comment sketches the SQL as `FROM events, <tsquery> AS query`
   with `ts_rank_cd(search_tsv, query)` (`query.rs:196-207`), while the emitted SQL is
   `FROM events CROSS JOIN LATERAL (SELECT <tsquery> AS query) AS search_query` with
   `ts_rank_cd(search_tsv, search_query.query)` (`query.rs:233-242`). Same semantics,
   drifted text.

#### TODO / FIXME / HACK / XXX comments

A case-insensitive grep for `TODO`, `FIXME`, `HACK`, `XXX` across the whole crate
(`Cargo.toml`, `src/**`, `tests/**`) returns **zero matches**. The only
near-miss vocabulary is descriptive, not a marker:

| Text | File:line | Note |
|---|---|---|
| `the kind:0 content-flattening hack` | `crates/buzz-search/tests/fts_integration.rs:264-265` | prose in a test doc comment describing removed legacy behavior; not a `HACK:` marker |
| `xyzzy` / `qwerty` inside test token literals | `tests/fts_integration.rs:1109`, `1270`, `1366` | test fixtures, not markers |


## Module: buzz-audit (`crates/buzz-audit`)

### Aspect: Features

### Capability inventory

| Capability | Where | Completeness |
|---|---|---|
| Append an entry to a per-community hash chain | `AuditService::log` (`crates/buzz-audit/src/service.rs:53-80`) + `log_inner` (`:82-152`) | Complete: assigns `seq`, links `prev_hash`, stamps `created_at`, computes hash, inserts in one transaction |
| Per-community write serialization across processes | `pg_advisory_lock(hashtextextended(...))` (`service.rs:56-62`, key at `:29`, `:58`) | Complete for the happy and error paths; unlock errors are discarded (`service.rs:71`) |
| Panic-safe lock release | `catch_unwind` + unconditional unlock + `resume_unwind` (`service.rs:67-79`) | Implemented; **no test exercises it** (no panic-inducing test in `service.rs:258-527`) |
| Deterministic hashing of an entry | `compute_hash` (`hash.rs:42-73`) | Complete; 9-field fixed order, presence tags, canonical JSON |
| Timestamp precision normalization | `to_storage_precision` (`hash.rs:22-24`), `log_timestamp` (`service.rs:21-23`), re-normalization in `compute_hash` (`hash.rs:47-51`) | Complete and double-enforced |
| Canonical JSON serialization | `canonical_json` (`hash.rs:80-116`) | Complete for object/array/scalar; recursion has no depth guard |
| Chain verification over a `seq` range | `verify_chain` (`service.rs:160-206`) | Partial by design: checks links + recomputed hashes inside the range; does not check range start against genesis, does not check `seq` contiguity, does not detect tail truncation |
| Paginated chain read | `get_entries` (`service.rs:212-235`) | Complete: `seq >= from_seq`, `ORDER BY seq ASC`, `LIMIT` |
| Action enum ↔ string round-trip | `as_str` (`action.rs:35-49`), `Display` (`:66-70`), `FromStr` (`:72-82`) | Complete for all 11 variants; unknown strings error |
| Error taxonomy | `AuditError` (`error.rs:12-41`) | 5 variants; see Conventions |
| Genesis sentinel | `GENESIS_HASH` (`hash.rs:9`) | Complete |

### Not implemented / absent (verified by reading, not inferred)

- **No DDL / migrations in the crate.** Stated at `lib.rs:17-18`; table lives in
  `migrations/0001_initial_schema.sql:606-619`.
- **No head/tip accessor.** There is no public `head()`/`latest_seq()`; head is read
  privately inside `log_inner` (`service.rs:94-101`).
- **No count/aggregate, no filter-by-action, no time-range query.** The only reads are
  `verify_chain` and `get_entries`, both keyed on `seq` (`service.rs:160`, `:212`).
- **No external anchoring, signing, or notarization.** No signature, HMAC, Merkle
  root, or export path exists in `crates/buzz-audit/src` (the crate's only crypto
  dependency use is `sha2` at `hash.rs:2`).
- **No retention, archival, or pruning.** No `DELETE`/`TRUNCATE` statement in the
  crate.
- **No batch append.** `log` takes a single `NewAuditEntry` (`service.rs:53`).
- **No retry/backoff on DB failure.** Errors are returned to the caller; the relay
  worker logs and drops them (`crates/buzz-relay/src/state.rs:1199-1207`).
- **No cargo features.** `crates/buzz-audit/Cargo.toml` has no `[features]` section.
- **No integration/`tests/` directory.** All tests are inline `#[cfg(test)]` modules.

### Actions defined vs. actions actually produced

11 variants are defined (`action.rs:8-31`), but repo-wide grep for `AuditAction::`
outside this crate finds only two producers:

| Action | Produced at |
|---|---|
| `EventCreated` | `crates/buzz-relay/src/handlers/event.rs:583` |
| `MediaUploaded` | `crates/buzz-relay/src/api/media.rs:428` |

The remaining 9 (`EventDeleted`, `ChannelCreated`, `ChannelUpdated`, `ChannelDeleted`,
`MemberAdded`, `MemberRemoved`, `AuthSuccess`, `AuthFailure`, `RateLimitExceeded`) have
no production call site in the repo; they appear only in this crate's own tests
(e.g. `service.rs:352`, `:356`, `:452`, `:455`).

### TODO / FIXME / HACK / XXX comments

A case-insensitive search for `todo`, `fixme`, `hack`, `xxx`, `unimplemented!`,
`todo!` across `crates/buzz-audit/src` returns **zero matches**. There are no
deferred-work markers in the crate.

### Documentation-vs-code drift found while reading

| ARCHITECTURE.md claim (line) | Code reality |
|---|---|
| "10 audit actions" (`ARCHITECTURE.md:503`) | 11 variants (`action.rs:8-31`); `MediaUploaded` is missing from the doc list |
| Hash covers `event_id`, `event_kind`, `channel_id` (`ARCHITECTURE.md:501`) | Those fields do not exist on `AuditEntry` (`entry.rs:14-37`); pre-image has `community_id`, `seq`, `created_at`, `action`, `actor_pubkey`, `object_id`, `detail`, `prev_hash` (`hash.rs:45-71`) |
| `AuditError::AuthEventForbidden` returned for `KIND_AUTH` (`ARCHITECTURE.md:505`) | No such variant (`error.rs:12-41`); the refusal is in the relay ingest path (`crates/buzz-relay/src/handlers/ingest.rs:1464-1468`) |
| `GENESIS_HASH` is "64 zeros" (`ARCHITECTURE.md:497`) | `[u8; 32]` of zeros, i.e. 32 bytes = 64 zero hex chars (`hash.rs:9`) |
| "`pg_advisory_lock` before each transaction" implying one lock (`ARCHITECTURE.md:501`) | Lock key is per-community (`service.rs:29`, `:58`) |


## Module: buzz-media (`crates/buzz-media`)

### Aspect: Features

Completeness is judged against what the code does, not against the Blossom spec's full endpoint set.

| Feature | Completeness | Evidence |
|---|---|---|
| Content-addressed blob upload (image path) — sniff, validate, hash, store, thumbnail, blurhash, sidecar | Full | `crates/buzz-media/src/upload.rs:207-236`, `crates/buzz-media/src/thumbnail.rs:15-50` |
| Generic-file upload path (documents, archives, text, data) | Full | `crates/buzz-media/src/upload.rs:245-274`, `crates/buzz-media/src/validation.rs:159-209` |
| Streaming video upload (never fully buffered in RAM) | Full | `crates/buzz-media/src/upload.rs:292-511` |
| Server-side SHA-256 + `x`-tag hash verification | Full | `crates/buzz-media/src/upload.rs:84-86`, `crates/buzz-media/src/auth.rs:190-196` |
| Blossom kind-24242 auth verification (upload + get verbs) | Full for the two verbs implemented; `delete`/`list`/`mirror` verbs absent | `crates/buzz-media/src/auth.rs:6-10`, `crates/buzz-media/src/auth.rs:31-236` |
| Per-community sidecar metadata as tenant read gate | Full | `crates/buzz-media/src/storage.rs:177-221` |
| Idempotent re-upload short-circuit | Full | `crates/buzz-media/src/upload.rs:97-128`, `crates/buzz-media/src/upload.rs:424-453` |
| Thumbnail generation (320px JPEG) + 4×3 blurhash | Full for images; **none for video** ("no thumbnail — desktop handles that") | `crates/buzz-media/src/thumbnail.rs:29-38`, `crates/buzz-media/src/upload.rs:437-448` |
| Metadata/EXIF policy enforcement for JPEG/PNG/WebP/GIF/MP4 | Full as a *rejection* policy; no stripping/sanitizing is performed server-side | `crates/buzz-media/src/validation.rs:487-928` |
| Image-bomb protection (25 MP pre-decode cap, fail-closed on unparseable geometry) | Full | `crates/buzz-media/src/validation.rs:264-273` |
| MP4 structural validation (fast-start, codec, duration, resolution, box allowlist) | Full | `crates/buzz-media/src/validation.rs:289-395`, `crates/buzz-media/src/validation.rs:408-484`, `crates/buzz-media/src/validation.rs:831-928` |
| S3 range reads and streaming reads (HTTP 206 / large video serving support) | Full | `crates/buzz-media/src/storage.rs:118-146` |
| HEAD with size metadata | Partial — only `content_length` is surfaced; no ETag, last-modified, or content-type | `crates/buzz-media/src/storage.rs:167-175`, `crates/buzz-media/src/storage.rs:379-381` |
| Per-upload moderation records (`_uploads/` side channel) with opt-in IP/port capture | Full, off by default | `crates/buzz-media/src/upload_record.rs:139-199`, `crates/buzz-media/src/config.rs:45-61` |
| Bucket key taxonomy + storage-accounting sweep (physical/logical/orphan/anomaly gauges) | Full as a pure fold; the S3 driver is one page-listing method and the scheduling lives in the relay | `crates/buzz-media/src/bucket_index.rs:54-411`, `crates/buzz-media/src/storage.rs:242-265` |
| AWS credential-chain support (IRSA / Pod Identity) alongside static keys | Full | `crates/buzz-media/src/storage.rs:25-65` |
| Blob deletion / GC / orphan sweeping (actual deletes) | **Stubbed** — only a raw `delete(key)` primitive; no GC job, no cascade, no retention | `crates/buzz-media/src/storage.rs:158-164`, `crates/buzz-media/src/upload.rs:122-127` |
| Blossom BUD-02 `/list`, `DELETE /{sha256}`, BUD-04 mirror | **Absent** in this crate | no such fn in `crates/buzz-media/src/lib.rs:17-28`; relay routes only upload/get/head (`crates/buzz-relay/src/router.rs:38-45`) |
| Signed/presigned URL generation | **Absent** — no `presign_*` call anywhere | `crates/buzz-media/src/storage.rs:73-265` |
| Multipart upload | **Absent** — `put_object_stream_with_content_type` is used; no explicit multipart API | `crates/buzz-media/src/storage.rs:96-101` |
| Audio support | Deliberately **not supported**: sniffed `audio/*` is rejected on the generic path "until Buzz has an explicit sanitizer and location-metadata validator for its container" | `crates/buzz-media/src/validation.rs:186-196` |
| PDF inline preview | Deliberately deferred — "inline PDF preview is a planned fast-follow" | `crates/buzz-media/src/validation.rs:211-215` |
| Transcoding / re-encoding of stored media | **Absent** by design (validation rejects non-canonical inputs instead) | `crates/buzz-media/src/validation.rs:487-491` |

---

### TODO / FIXME / HACK / XXX comments

**None.** A repo-relative scan of all 12 `.rs` files in the crate returns zero matches for `TODO`, `FIXME`, `HACK`, or `XXX` (verified across `crates/buzz-media/src/*.rs` and `crates/buzz-media/tests/static_creds_minio.rs`).

Deferred work is instead recorded in prose comments. The forward-looking ones, quoted verbatim:

- "A V2 background GC job can sweep blobs with no matching sidecar after a grace period." — `crates/buzz-media/src/upload.rs:126-127`
- "PDF is intentionally *not* inline yet — inline PDF preview is a planned fast-follow; until the renderer handles it, force download like any other file." — `crates/buzz-media/src/validation.rs:213-215`
- "audio is rejected until Buzz has an explicit sanitizer and location-metadata validator for its container." — `crates/buzz-media/src/validation.rs:188-190`
- "Revert to crates.io once #449 lands upstream." (workspace-level, about the `aws-creds` fork this crate depends on) — `Cargo.toml:169`

For contrast, the adjacent relay handler *does* carry a TODO for the quota gap this crate does not cover: "TODO(v2): Add persistent per-pubkey storage quotas. Admission limits below bound active parser/storage work, but they do not cap durable bytes stored." — `crates/buzz-relay/src/api/media.rs:302-304`.


## Module: buzz-workflow (`crates/buzz-workflow`)

### Aspect: Features

---

### 1. Capabilities

| Capability | Where | Status |
|---|---|---|
| YAML → validated `WorkflowDef` → canonical JSON | `schema.rs:262-268`, `lib.rs:163-165` | full |
| Definition validation (name, steps, step-id charset/uniqueness, cron/interval invariants) | `schema.rs:152-229` | full |
| Event-driven trigger matching with cache | `lib.rs:276-383` | full |
| Cron + interval scheduling with cross-pod at-most-once claim | `lib.rs:428-672` | full |
| Sequential step execution with per-step timeout | `executor.rs:1083-1217` | full |
| `if:` conditions via evalexpr with 4 custom string helpers | `executor.rs:224-384` | full |
| Template substitution with 2 filters | `executor.rs:70-201` | full (single-pass only) |
| Execution trace persistence + partial-progress on failure | `executor.rs:1108-1204`, `error.rs:9-15`, `lib.rs:175-263` | full |
| Concurrency admission control | `executor.rs:978-983`, `lib.rs:110-111` | full |
| Approval-gate suspension + resume plumbing | `executor.rs:650-668`, `executor.rs:1018-1075` | partial — token is generated but never persisted; runs end `Failed` |
| Side-effect abstraction (`ActionSink`) | `action_sink.rs:48-70` | partial — only `send_message` exists on the trait |
| Outbound HTTP with SSRF protection | `executor.rs:745-871` | full, gated behind the `reqwest` feature |

---

### 2. Trigger completeness

| Trigger | Status | Evidence |
|---|---|---|
| `message_posted` | **full** — kind-9 gate + optional evalexpr filter | `lib.rs:958`, `lib.rs:824-846` |
| `reaction_added` | **full** — kind-7 gate + exact emoji equality (no shortcode/unicode normalization) | `lib.rs:959`, `lib.rs:807-822` |
| `diff_posted` | **full** — kind-40008 gate + optional filter | `lib.rs:960`, `lib.rs:848-880` |
| `schedule` (`cron`) | **full** — window-based match, deterministic claim anchor | `lib.rs:526-534`, `lib.rs:688-706` |
| `schedule` (`interval`) | **full** — bucket-quantized anchor, restart-safe prefilter, ≥ 60 s enforced at parse | `lib.rs:535-538`, `lib.rs:719-797`, `schema.rs:212-225` |
| `webhook` | **schema + relay-side only** — this crate never fires it; the entry point is the relay's `POST /hooks/{id}` handler, which builds the `TriggerContext` and calls `execute_from_step` directly | `lib.rs:962`, `crates/buzz-relay/src/router.rs:120`, `crates/buzz-relay/src/api/bridge.rs:1777-1911` |

---

### 3. Action completeness (precise, as of the code read)

| Action | Status | Detail |
|---|---|---|
| `send_message` | **full** (requires a wired `ActionSink`) | Community-scoped run/workflow lookup, destination validation, owner attribution, real event emission through `RelayActionSink` (`executor.rs:530-578`; `crates/buzz-relay/src/workflow_sink.rs:159`). Without `set_action_sink` it fails with `InvalidDefinition` (`lib.rs:148-156`). |
| `send_dm` | **stubbed** | Returns `WorkflowError::NotImplemented("SendDm")` — the step and therefore the run fails (`executor.rs:580-584`). |
| `set_channel_topic` | **stubbed** | Returns `WorkflowError::NotImplemented("SetChannelTopic")` (`executor.rs:586-590`). |
| `add_reaction` | **partial / non-functional against the current relay** | Code path is complete under `feature = "reqwest"` (which the relay enables, `crates/buzz-relay/Cargo.toml:63`) but targets `POST {BUZZ_RELAY_BASE_URL}/api/messages/{message_id}/reactions` (`executor.rs:888-894`). No such route is registered in the relay router (`crates/buzz-relay/src/router.rs:39-125` — verified by grep for `reactions` and `/api/messages`), so the call returns a non-success status and the step fails with `WebhookError` (`executor.rs:914-922`). Without the `reqwest` feature it silently returns `{added:false, skipped:true}` (`executor.rs:606-616`). |
| `call_webhook` | **full** under `feature = "reqwest"` | SSRF guard, DNS pinning, redirects disabled, 10 s timeout, 1 MiB cap (`executor.rs:781-869`). Without the feature it is a no-op returning `{status:0, body:null, skipped:true}` (`executor.rs:636-647`). |
| `request_approval` | **partial / effectively non-functional** | Generates a token and suspends, but nothing persists an approval record or emits kind 46010 (`executor.rs:650-668`); `finalize_run` converts the suspension into `RunStatus::Failed` (`lib.rs:192-215`). |
| `delay` | **full, bounded** | Max 270 s; longer delays fail the step (`executor.rs:671-690`). |

Header note: the module doc comment still claims "Action dispatch uses placeholder implementations that log intent. Real event emission is wired in WF-07/08" (`executor.rs:9-10`) and `dispatch_action`'s doc says "For MVP, most actions log their intent and return a success output" (`executor.rs:521-522`). Both are stale relative to the `send_message`/`call_webhook` implementations.

---

### 4. Feature-flag effects

`feature = "reqwest"` (`Cargo.toml:28-29`) toggles `add_reaction` and `call_webhook` between real HTTP and skip-stubs. `buzz-relay` enables it (`crates/buzz-relay/Cargo.toml:63`); `buzz-admin` depends on the crate without it (`crates/buzz-admin/Cargo.toml:21`), so an admin-side build compiles the skip-stubs.

---

### 5. All TODO/FIXME/HACK/XXX comments (verbatim)

Repo-grep over `crates/buzz-workflow/` found exactly three markers, all `TODO`:

| Marker | Verbatim text | Location |
|---|---|---|
| TODO | `// TODO (WF-07): emit DM event.` | `crates/buzz-workflow/src/executor.rs:582` |
| TODO | `// TODO (WF-07): update channel topic via DB.` | `crates/buzz-workflow/src/executor.rs:588` |
| TODO | `// TODO (WF-08): create approval record in DB, emit kind:46010.` | `crates/buzz-workflow/src/executor.rs:663` |

No `FIXME`, `HACK`, or `XXX` comments exist in the crate.

Adjacent in-code work markers that are not TODO-tagged but state incomplete behaviour:
- `"// Approval gates are not yet implemented (WF-08). / Fail explicitly rather than creating unreachable WaitingApproval rows."` (`lib.rs:192-193`).
- `"Long delays (hours/days) should use the scheduled resume pattern (future work: WF-09)."` (`executor.rs:674-675`).
- `"In-memory only — lost on restart. Missed fires during downtime are not replayed (acceptable for MVP)."` (`lib.rs:85-86`).
- Numbered "Fix N" comments referencing prior review rounds: Fix 4 (`schema.rs:218`), Fix 2 (`lib.rs:465`), Fix 5 (`lib.rs:572`), Fix 6 (`lib.rs:638`), Fix 1 (`lib.rs:664`).


## Module: buzz-relay — core & bootstrap (`crates/buzz-relay/src`)
### Aspect: Features

---

#### 1. What bootstrap enables

`main()` (`main.rs:83-1060`) is the single composition root. Everything the relay can do is switched on here.

| Capability | Enabled by | Gate |
|-----------|-----------|------|
| NIP-01 WebSocket relay | `build_router` (`router.rs:32`) + `handle_connection` via `router.rs:316` | always |
| NIP-11 relay info (content-negotiated + `/info`) | `nip11.rs:235`, `router.rs:63-64` | always, unauthenticated |
| NIP-05 discovery | `router.rs:65` | always |
| Nostr-over-HTTP bridge (`/events`, `/query`, `/count`) | `router.rs:71-73` | always, NIP-98 |
| Blossom media upload/download | `media_router` (`router.rs:37-45`) | always; GET auth off by default (`config.rs:197`) |
| Git smart HTTP | `git_router` (`api/git/transport.rs:1756`) | always, but gated behind the fatal A3 probe (`main.rs:466-503`) |
| Git policy hook callback | `git_policy_router` (`api/git/mod.rs:60`) | always, localhost-only |
| Huddle audio WebSocket | `router.rs:125-128` | `huddle_audio_available` (default `true`, `config.rs:489-491`) checked at `audio/handler.rs:357` |
| Relay invites + join policy | `router.rs:95-111` | join-policy routes return content only when `config.join_policy.is_some()` (`config.rs:793-810`) |
| Operator community provisioning | `router.rs:74-93` | `relay_operator_pubkeys` non-empty (fail-closed default: disabled, `config.rs:161-169`) |
| Deployment-admin API + SPA | `router.rs:47-54` | `BUZZ_ADMIN_HOST` set (`config.rs:813-838`) |
| Public web bundle (invite landing / git browser) | SPA fallback `router.rs:145-183` | `BUZZ_WEB_DIR` set; git-browser routes additionally need `BUZZ_SERVE_GIT_WEB_GUI` |
| Moderation queue reads | `router.rs:113-118` | always, NIP-98 + mod-authz |
| Workflow webhooks | `router.rs:120` | always, secret-authenticated |
| Prometheus `/metrics` | `metrics::install` (`main.rs:138`) | always |
| Distributed tracing (OTLP) | `telemetry::try_init_tracer` (`main.rs:100`) | `OTEL_EXPORTER_OTLP_ENDPOINT` set (`telemetry.rs:80-83`) |
| Tamper-evident audit log | separate PG pool + worker (`main.rs:322-334`, `state.rs:653-690`) | `BUZZ_AUDIT_ENABLED` (default `true`, `config.rs:793`) |
| Inter-relay QUIC mesh | `mesh_boot::boot_mesh` (`main.rs:442`) | `BUZZ_MESH=on` (default off, `config.rs:492-509`) |
| Mesh reliable-stream echo probe | `handle.wire_consumers(..., mesh_demo_echo, ...)` (`main.rs:455`) + `router.rs:123` | `BUZZ_MESH_DEMO_ECHO=on` (`config.rs:514-518`) |
| NIP-PL push (matcher + delivery worker) | `main.rs:686-691` | `push_gateway_delivery_url.is_some()` — **on by default** (`config.rs:752-758`) |
| NIP-ER reminder scheduler | `main.rs:693-802` | always |
| NIP-43 relay membership enforcement + snapshot reconciliation | `main.rs:505-546` | `BUZZ_REQUIRE_RELAY_MEMBERSHIP` (default false) |
| NIP-OA owner-attested agent auth | `api/mod.rs:81` | `BUZZ_ALLOW_NIP_OA_AUTH` (default false, `config.rs:520-523`) |
| Ephemeral channel reaping | `main.rs:602-684` | always |
| Multi-node fan-out / cache invalidation / conn-control | `main.rs:350-367` (subscribers) + `main.rs:804-936` (consumers) | always |
| Community lifecycle durable revalidation | `main.rs:869-890` | always |
| Usage + storage metrics polling | `main.rs:987-1041` | always; per-community series gated by `BUZZ_USAGE_METRICS_PER_COMMUNITY`, storage sweep by `BUZZ_STORAGE_METRICS` (`storage_sweep.rs:69`) |
| Postgres FTS search | `main.rs:369-386` | always |
| Replica read routing behind a freshness fence | `main.rs:177-198` | `READ_DATABASE_URL` set + probe verifies the floor guard |
| Runtime conformance tracing | `state.rs:797` | production always binds `NoopTracer` (zero cost); only tests swap in `JsonlTracer` |
| UDS listener (service mesh sidecar) | `main.rs:1163-1206` | `BUZZ_UDS_PATH` set, unix only |
| Auto migrations | `main.rs:161-172` | `BUZZ_AUTO_MIGRATE` (default off) |
| Channel-event reconciliation (dev/CI) | `main.rs:547-590` | `BUZZ_RECONCILE_CHANNELS` present |
| Dev relay identity | `main.rs:396-408` | `BUZZ_RELAY_PRIVATE_KEY` unset **and** `!require_auth_token` |

#### 2. Background tasks — every `tokio::spawn`

**23 production spawn sites** in this file group (19 in `main.rs`, 3 in `state.rs`, 1 in `metrics.rs`). One additional spawn at `main.rs:1834` is inside `#[cfg(test)]`. `mesh_boot::boot_mesh` / `wire_consumers` spawn further tasks outside this group.

| # | Task | Spawn | Cadence / trigger | On error | Cancellable? |
|---|------|-------|-------------------|----------|--------------|
| 1 | Prometheus exporter HTTP server | `metrics.rs:146` | event-driven | none (future returns) | no |
| 2 | Audit worker | `state.rs:658` | event-driven on `audit_tx` | per-entry error → `buzz_audit_log_errors_total` + `error!` (`state.rs:1200-1206`); worker keeps going | **yes** — `AuditShutdownHandle` (`state.rs:1182-1196`), 5 s drain from `main.rs:1049` |
| 3 | Cache-invalidation publisher (fire-and-forget, per invalidation) | `state.rs:968` | per `invalidate_*` call | `warn!` and swallow; backstopped by ≤10 s TTL (`state.rs:960-963`) | no |
| 4 | Conn-control publisher (ban fan-out, per ban) | `state.rs:1043` | per ban | `warn!` and swallow; DB ban row is the durable backstop (`state.rs:1046-1050`) | no |
| 5 | Redis pub/sub event subscriber | `main.rs:354` | continuous | inside `run_subscriber` | no |
| 6 | Redis cache-invalidation subscriber | `main.rs:360` | continuous | inside | no |
| 7 | Redis conn-control subscriber | `main.rs:366` | continuous | inside | no |
| 8 | NIP-43 membership snapshot reconciler | `main.rs:522` | `BUZZ_NIP43_RECONCILE_INTERVAL_SECS`, default 60 s, `.max(1)`; first tick consumed immediately (`main.rs:524`) | `warn!`, loop continues | no |
| 9 | Channel-event reconciler (dev/CI) | `main.rs:551` | 24 attempts, 5 s apart (≈2 min) then exits | `warn!` per attempt | no |
| 10 | Workflow cron loop | `main.rs:599` | inside `WorkflowEngine::run` | inside | no |
| 11 | Ephemeral channel reaper | `main.rs:613` | `BUZZ_REAPER_INTERVAL_SECS`, default 60 s | tick error → `error!` + `continue`; per-channel side-effect errors logged individually (`main.rs:657/667`) | no |
| 12 | NIP-PL matcher | `main.rs:687` | inside `push_runtime::run_matcher` | inside | no |
| 13 | NIP-PL delivery worker | `main.rs:688` | inside `push_runtime::run_delivery_worker` | inside | no |
| 14 | NIP-ER reminder scheduler | `main.rs:709` | `SPROUT_REMINDER_SCHEDULER_INTERVAL_SECS`, default 10 s; batch `SPROUT_REMINDER_SCHEDULER_BATCH_LIMIT`, default 100 | query error → `error!` + `continue`; claim error → `warn!` + skip; publish error → release the claim (compare-and-clear on a per-attempt stamp), release failure → `warn!` **and the reminder stays claimed forever** (`main.rs:781-798`) | no |
| 15 | Multi-node fan-out consumer | `main.rs:823` | continuous `broadcast::recv` | `Lagged(n)` → `buzz_multinode_fanout_lag_total += n` + `warn!`; `Closed` → `error!` + **break (task dies, fan-out silently stops)** | no |
| 16 | Cache-invalidation consumer | `main.rs:853` | continuous | `Lagged` → counter + `warn!`; `Closed` → `error!` + break | no |
| 17 | Community lifecycle revalidator | `main.rs:888` | `BUZZ_COMMUNITY_REVALIDATE_INTERVAL_SECS`, default 30 s, `.clamp(1,300)` | per-community failure retains sockets, logged `warn!` (`state.rs:1082-1084`) | **yes** — `community_revalidator_cancel`, `main.rs:1045`; exits without waiting for the next tick (test `main.rs:1827-1849`) |
| 18 | Conn-control consumer | `main.rs:904` | continuous | `Lagged` → counter + `warn!`; `Closed` → `error!` + break | no |
| 19 | Pool metrics poller | `main.rs:950` | `BUZZ_POOL_METRICS_INTERVAL_SECS`, default 10 s, `.max(1)` | none — no fallible call in the loop | no |
| 20 | Usage metrics poller (+ storage sweep trigger) | `main.rs:1008` | `BUZZ_USAGE_METRICS_INTERVAL_SECS`, default 300 s, `.max(5)`; first tick jittered `rand % interval` | tick error → `error!` "skipping"; leader demotes | no |
| 21 | Health listener server | `main.rs:1122` | event-driven | `.ok()` — errors swallowed | no (deliberate, BR-RC-72) |
| 22 | Shutdown watchdog | `main.rs:1132` | one-shot on signal | ends in `std::process::exit(1)` after 30 s | no |
| 23 | UDS server (unix + `BUZZ_UDS_PATH`) | `main.rs:1181` | event-driven | `.ok()` | via `shutdown_tx` watch, then `abort()` (`main.rs:1197`) |
| — | Storage sweep body | `storage_sweep::maybe_spawn_sweep` called `main.rs:1456` | single-flight, `BUZZ_STORAGE_SWEEP_INTERVAL_SECS` | inside `storage_sweep` | no |

##### Failure-mode observations
- **Only 2 of 23 tasks are cancellable** (#2 audit, #17 revalidator). The other 21 are abandoned at process exit.
- Tasks #15, #16, #18 `break` out of their loop on `RecvError::Closed`. After that the pod keeps serving traffic but silently loses multi-node fan-out, cross-pod cache invalidation, or ban propagation respectively. There is no restart, no health-check coupling, and no gauge for "consumer alive" — only an `error!` line (`main.rs:838-841`, `main.rs:868-871`, `main.rs:929-932`).
- Task #14's release-failure branch (`main.rs:789-797`) leaves a reminder permanently claimed and never retried; the log explicitly says so.
- No task uses `JoinHandle` supervision or panic capture except #2 (`state.rs:1188` reports `Audit worker panicked`).

#### 3. Concurrency budget features

| Semaphore | Field | Bound | Default | Acquisition |
|-----------|-------|-------|---------|-------------|
| Connections | `conn_semaphore` (`state.rs:514`) | `max_connections` | 10,000 (`config.rs:449-452`) | `try_acquire_owned` — instant refusal |
| Handlers | `handler_semaphore` (`state.rs:516`) | `max_concurrent_handlers` | 1,024 (`config.rs:454-457`) | `try_acquire_owned` |
| Git subprocesses | `git_semaphore` (`state.rs:521`) | `git_max_concurrent_ops` | 20 (`config.rs:735-738`) | `api/git/transport.rs:322` |
| Media uploads | `media_upload_semaphore` (`state.rs:523`) | `media_max_concurrent_uploads` | 8 (`config.rs:663-668`) | `api/media.rs:119` |

`git_semaphore`'s doc is explicit that it bounds resources and is **not** writer serialization — that is the manifest-pointer CAS (`state.rs:517-520`).

#### 4. Observability features surfaced by this module

- **Framework HTTP metrics**: `http_requests_total`, `http_request_latency_ms` with `{code, caller, action}` (`metrics.rs:200-205`). `action` is the axum `MatchedPath` — never the raw URI, deliberately, to avoid unbounded cardinality from scanners (`metrics.rs:166-179`).
- **Per-metric histogram buckets**: 5 bucket families registered at `metrics.rs:76-141` — HTTP latency ms (11 buckets, `metrics.rs:37-39`), generic `_seconds` suffix (10, `metrics.rs:42`), git durations (13, `metrics.rs:45-47`), git bytes (9, `metrics.rs:50-60`), git pack counts (9, `metrics.rs:63`), fan-out recipients (9, `metrics.rs:66`).
- **Gauge idle-timeout**: `MetricKindMask::GAUGE` with the BR-RC-06 timeout (`metrics.rs:75-78`) so intentionally-stopped series disappear.
- **Pool gauges** (`main.rs:955-985`): `buzz_db_pool_{size,idle,active,max}`, `buzz_db_read_pool_{size,idle,active,max}`, `buzz_db_replica_fence_{open,lag_seconds}`, `buzz_redis_pool_{available,size,max,waiting}`. **The audit pool (`main.rs:325-329`) and the search pool (`main.rs:378-382`) are not instrumented at all.**
- **Usage gauges** (`main.rs:1481-1806`): fleet totals `buzz_total_{ws_connections,users_online_pod,subscriptions,users,channels,messages,relay_members,workflows,git_repos,active_users,active_channels}` + `buzz_communities_total`; per-community `buzz_community_{ws_connections,users_online_pod,subscriptions,users,channels,messages,relay_members,workflows,git_repos,active_users,active_channels}`.
- **Leader gauge**: `buzz_usage_poller_is_leader` (`main.rs:1032-1036`).
- **Lag counters**: `buzz_multinode_fanout_lag_total` (`main.rs:836`), `buzz_cache_invalidation_lag_total` (`main.rs:866`), `buzz_conn_control_lag_total` (`main.rs:927`).
- **Audit metrics**: `buzz_audit_enabled` gauge (`main.rs:139`), `buzz_audit_log_errors_total`, `buzz_audit_log_seconds` (`state.rs:1201-1205`).
- **Cache-hit metrics**: `buzz_membership_cache_{hits,misses}_total` (`state.rs:833/836`), `buzz_accessible_channels_cache_{hits,misses}_total` (`state.rs:1095/1098`). The other five caches have **no** hit/miss instrumentation.
- **Backpressure**: `buzz_ws_backpressure_disconnects_total` (`state.rs:466`).
- **Tracing**: JSON-flattened stdout always (`main.rs:109`); OTLP gRPC layer attached only when `OTEL_EXPORTER_OTLP_ENDPOINT` is set (`main.rs:101-107`, `telemetry.rs:79-90`), tracer name `"buzz-relay"` (`main.rs:104`).

#### 5. Feature flags / gates summary

Cargo features (`Cargo.toml:80-81`): exactly one — `dev = ["buzz-auth/dev"]`. No `#[cfg(feature = ...)]` in this file group; the flag only forwards dev key derivation into `buzz-auth`.

Compile-time cfg used: `#[cfg(unix)]` / `#[cfg(not(unix))]` for UDS (`main.rs:1162`, `main.rs:1207`) and the SIGTERM handler (`main.rs:1227`, `main.rs:1236`); `#[cfg(unix)]` on one config test (`config.rs:1329`).

Runtime kill switches that produce **byte-identical off behaviour** (explicit design intent, verified):
- `BUZZ_MESH` off ⇒ no UDP bind, no Redis registry write, no spawn (`config.rs:492-497`, `main.rs:437-441`).
- `BUZZ_MESH_DEMO_ECHO` off ⇒ inbound reliable streams accepted, logged, closed (`config.rs:139-143`).
- `EmissionScope::Off` ⇒ no per-community series; totals unchanged (`main.rs:76-78`).
- Storage sweep disabled ⇒ zero storage-family gauges including health (`main.rs:1436-1453`).
- Audit disabled ⇒ `audit_tx = None`, worker parked on cancellation (`state.rs:659-663`, `state.rs:763`).


## Module: buzz-relay — WebSocket protocol & subscriptions (`crates/buzz-relay/src`)
### Aspect: Features

---

#### 1. Capabilities delivered over WebSocket

| Capability | NIP | Implemented at | Notes |
|---|---|---|---|
| Relay-initiated challenge/response auth | NIP-42 | `connection.rs:182-186` + `handlers/auth.rs:44-296` | mandatory within 5 s (`connection.rs:27`) |
| Owner-attestation agent auth (agent→owner delegation) | NIP-OA | `auth.rs:28-42`, `:224-281` | tag extracted from the signed AUTH event, so it is signature-protected |
| Signed event submission | NIP-01 | `event.rs:585-736` | delegates persistent kinds to the shared ingest seam |
| Ephemeral event relay (20000–29999), never stored | NIP-16 range | `event.rs:762-897` | WS-only; HTTP explicitly rejects 1059 and 20001 (`ingest.rs:1448-1452`) |
| Presence set/clear | Buzz kind 20001 | `event.rs:772-800` | Redis-backed; falls through to global ephemeral fan-out |
| Encrypted agent observer frames (bidirectional) | Buzz kind 24200 | `event.rs:920-1069` | NIP-44 content, structural direction inference, own rate limiter |
| Filter subscriptions with historical backfill + EOSE | NIP-01 | `req.rs:44-417` | one DB query per filter, bounded concurrency 4 |
| Full-text search subscriptions | NIP-50 | `req.rs:505-727` | one-shot (no fan-out registration), paginated |
| Aggregate counts | NIP-45 | `count.rs:30-286` | exact-or-refuse (5000-candidate budget); since `ab3af828` any 30175-matching filter is forced off the exact SQL path (`count.rs:110`, `:174`, `:243`) |
| Subscription cancellation | NIP-01 | `close.rs:12-30` | idempotent |
| Channel (group) scoping via `#h` | NIP-29 | `req.rs:1009-1036`, `count.rs:17-26`, `subscription.rs:265-330` | no dedicated message types |
| Gift-wrapped DMs | NIP-17/59 | pubkey-mismatch exemption `event.rs:636-645`; read gate `req.rs:1038-1071` | WS-only ingest |
| Author-only-unless-shared persona catalog | NIP-AP (kind 30175) | gate `req.rs:388`, `:705`, `count.rs:202`/`:271`, fan-out `event.rs:154-175`; helper `req.rs:1222-1234` | **added by `ab3af828`**; SQL pre-filter `crates/buzz-db/src/event.rs:504-527`; ingest shape rule `ingest.rs:1042-1060` |
| Live cross-node delivery | — | `event.rs:282-332` ← `main.rs:818-845` | Redis pub/sub fan-in |
| Heartbeat / dead-peer detection | RFC 6455 | `connection.rs:378-405` | 30 s ping, 3 misses |
| Slow-client shedding | — | `connection.rs:88-113`, `state.rs:436-475` | 15-strike grace, shared counter |
| Live ban enforcement on open sockets | — | `state.rs:290-325` (called from moderation) + `auth.rs:105-183` | frame-then-close idiom |
| Graceful-restart signalling | RFC 6455 1012 | `state.rs:352-374` | sticky drain flag |
| Community archival closing live sockets | — | `state.rs:99-176`, `connection.rs:132-140` | plus a periodic durable revalidation backstop |
| Runtime conformance tracing on the read seam | — | `req.rs:110-116`, `:141-148`, `:332-366`, `:626-668` | `NoopTracer` in production (`state.rs:798`) |

---

#### 2. Realtime delivery path, end to end

##### 2.1 Ingest → fan-out (persistent event, same pod)

```
client ──["EVENT",e]──▶ recv_loop                     connection.rs:419-436
                        ├─ frame size check           connection.rs:421-434
                        ├─ ClientMessage::parse       protocol.rs:66-76
                        ├─ enforce_ws_admission       connection.rs:498-500 → :594-653
                        │    · WsEvents  (5 s window)
                        │    · Messages  (60 s window)
                        ├─ handler_semaphore permit   connection.rs:513-521
                        └─ tokio::spawn(handle_event) connection.rs:530-536
handle_event                                          event.rs:608
  ├─ auth read                                        event.rs:611-631
  ├─ pubkey match (1059 exempt)                       event.rs:636-645
  ├─ kind 22242 reject                                event.rs:647-655
  ├─ kind 24200 → observer branch                     event.rs:657-669
  ├─ ephemeral → ephemeral branch                     event.rs:675-696
  └─ ingest_event(state, tenant, event, IngestAuth)   event.rs:728
       └─ (verify, membership, DB insert, …)          handlers/ingest.rs:1393+
       └─ dispatch_persistent_event                   event.rs:349
            ├─ AWAITED: enqueue_event_created_audit   event.rs:358-366  (bounded chan, cap 1000)
            └─ spawn dispatch_persistent_event_inner  event.rs:374-393
                 ├─ mark_local_event                  event.rs:417
                 ├─ pubsub.publish_event(topic)       event.rs:418-427
                 ├─ sub_registry.fan_out_scoped       event.rs:429-431
                 ├─ filter_fanout_by_access(threaded) event.rs:432-439
                 │    └─ incl. kind-30175 shared gate event.rs:154-175
                 ├─ DM-visibility owner fence         event.rs:457-491
                 ├─ frame cache + send_to_text_bytes  event.rs:492-493
                 └─ spawn workflow_engine.on_event    event.rs:533-557
◀──["OK",id,true,""]── conn.send                      event.rs:719-723
◀──["EVENT",sub,e]──── per matching subscription      event.rs:76-98
```

The `OK` and the `EVENT` frames are on **independent** timelines: the `OK` is emitted as soon as `dispatch_persistent_event` returns (which is after the audit enqueue only), while fan-out happens in a spawned task (`event.rs:374`). A client can therefore observe its own event arriving on a subscription **before or after** its `OK`.

##### 2.2 Ingest → fan-out (ephemeral / observer, same pod)

Shorter: verify on `spawn_blocking` → optional membership check → `mark_local_event` → `publish_event` → `fan_out_event_to_local_subscribers` → `OK true`.

- channel-scoped ephemeral: `event.rs:831-865` (topic `Channel(ch)`)
- channel-less ephemeral: `event.rs:866-894` (topic `Global`)
- observer frame: `event.rs:1069-1089` (topic `Global`, always)

`fan_out_event_to_local_subscribers` (`event.rs:241-279`) is the canonical guarded local send: `fan_out_scoped` → `filter_fanout_by_access(…, None)` → serialise once → frame cache → `send_to_text_bytes`. It has six other production callers outside this group (`audio/handler.rs:1318`, `api/git/transport.rs:1703`, `side_effects.rs:803`/`:889`/`:2759`/`:2911`) — i.e. huddle audio events, git push notifications, and NIP-29/NIP-25 side effects all reach WS subscribers through this same gate.

##### 2.3 Where cross-node Redis events re-enter

Single re-entry point:

```
another pod ──▶ Redis PSUBSCRIBE buzz:{community}:{topic}
                  └─ buzz-pubsub subscriber task
                       └─ broadcast → pubsub.subscribe_local()
main.rs:818-845   loop { rx.recv() → fan_out_pubsub_event(&state, ev) }
event.rs:282-332  fan_out_pubsub_event
                    ├─ topic → Option<Uuid>              event.rs:287-290
                    ├─ StoredEvent::new(event, ch)       event.rs:292
                    ├─ local-echo dedup (community,id)   event.rs:295-304   ← consume-on-read
                    ├─ sub_registry.fan_out_scoped       event.rs:306
                    ├─ filter_fanout_by_access(…, None)  event.rs:307
                    ├─ buzz_multinode_fanout_total       event.rs:308
                    └─ frame cache + send_to_text_bytes  event.rs:311-330
```

Notes verified against the code:
- The **community label comes from the parsed Redis channel** (`channel_event.community_id`, `event.rs:291`), not from the event body — so a forged event body cannot relabel a delivery.
- The **`threaded` visibility argument is `None`** on this path (`event.rs:307`), so a cross-node private-channel delivery always pays a fresh visibility read (or a cached `private`).
- The **DM-visibility owner fence (BR-WS-110) does not exist on this path.** It lives only in `dispatch_persistent_event_inner` (`event.rs:457-491`). A kind 30622 / 44200 event that reaches a second pod via Redis is fanned out with only the `filter_fanout_by_access` fences applied — neither of those kinds is in `AUTHOR_ONLY_KINDS` (`buzz-core/src/kind.rs:120`), so the owner check is absent. The kind-30175 gate added by `ab3af828` **is** inside `filter_fanout_by_access` (`event.rs:154-175`) and therefore does reach this path — the same placement would fix the 30622/44200 case. See the security and debt aspects.
- Broadcast lag is counted (`buzz_multinode_fanout_lag_total`, `main.rs:834`) but **not** repaired — lagged events are lost, not re-fetched.

##### 2.4 Redis subscription demand is driven only by REQ

`retain_topic` is called from exactly **one** production site: `req.rs:255`, after a successful non-search registration. The three `release_topic` sites are `req.rs:250` (replaced subscription), `close.rs:21` (explicit CLOSE), `connection.rs:268` (per removed subscription at disconnect).

Consequence: a pod that only *publishes* never PSUBSCRIBEs. `handlers/event.rs:1706` and `:1687` do call `retain_topic(&tenant, EventTopic::Global)` — but both are inside `#[cfg(test)]` (the Redis round-trip test, module begins `event.rs:1158`), precisely because the test needed to force the demand-driven subscription that production only creates via REQ (see the comment at `event.rs:1715-1722`). This is correct behaviour, not a gap, but it means **local echo suppression on the publishing pod only matters when that pod also has a subscriber for the topic**.

---

#### 3. Fan-out index selection (the performance feature)

`fan_out_scoped` (`subscription.rs:265-330`) is sub-linear in total subscriptions:

| Event shape | Indexes consulted | Site |
|---|---|---|
| `channel_id = Some(ch)` | `channel_kind_index[(community,{ch,kind})]`, then `channel_wildcard_index[(community,ch)]` | `:271-289` |
| `channel_id = None` | `global_p_kind_index[{community,kind,p}]` for **each** distinct `p` tag, then `global_kind_index[(community,kind)]`, then `global_wildcard_index[community]` | `:290-318` |

Every candidate still runs the full `filters_match` predicate (`:369-387`), with a `seen` set suppressing duplicates when a subscription appears in two indexes. The `(kind, #p)` index exists specifically so an observer-frame or membership-notification broadcast does not scan every subscription of that kind — pinned by `subscription.rs:1218-1250`.

---

#### 4. Frame-serialisation optimisation

One `serde_json::to_string` per event per fan-out cycle (`event.rs:252-259`, `:288-295`, `:420-431`), then one `Arc<Bytes>` per distinct `sub_id` (`fanout_frame_cache`, `event.rs:63-74`), cloned per recipient without copying the body (`state.rs:444-448`). Byte-for-byte compatibility with the legacy `format!` output is pinned at `event.rs:1178-1189`, and cross-cycle `Arc` reuse is explicitly forbidden and tested at `event.rs:1168-1188`.

Outbound writes are batched: up to 64 data frames per `flush()` (`connection.rs:347-369`), with control frames always drained first (`connection.rs:322-325`).

---

#### 5. Observability emitted by this group

| Metric | Type | Site |
|---|---|---|
| `buzz_ws_connections_total{community}` | counter | `connection.rs:184-188` |
| `buzz_ws_connections_active` | gauge | `connection.rs:196`, `:286` |
| `buzz_ws_auth_timeouts_total` | counter | `connection.rs:243` |
| `buzz_ws_backpressure_disconnects_total` | counter | `connection.rs:101`, `state.rs:466` |
| `buzz_ws_send_batch_size` | histogram | `connection.rs:368` |
| `buzz_admission_rejections_total{transport,reason}` | counter | `connection.rs:663`, `:671` |
| `buzz_auth_attempts_total{method}` | counter | `auth.rs:84` |
| `buzz_auth_failures_total{reason}` | counter | `auth.rs:169-171`, `:210`, `:235`, `:288` — reasons `banned`, `ban_check_error`, `allowlist_denied`, `not_relay_member`, `nip42_invalid` |
| `buzz_subscriptions_active` | gauge | `subscription.rs:83`, `:170`, `:192` |
| `buzz_events_received_total{kind}` | counter | `event.rs:624` (kind label bounded by `bounded_kind_label`, `:35-53`) |
| `buzz_community_events_received_total{community}` | counter | `event.rs:628-632` |
| `buzz_event_processing_seconds` | histogram | `event.rs:740` |
| `buzz_fanout_recipients` | histogram | `event.rs:248`, `:418` |
| `buzz_multinode_fanout_total` | counter | `event.rs:308` |
| `buzz_post_commit_dispatch_scheduled_total` | counter | `event.rs:374` |
| `buzz_post_commit_dispatch_errors_total{stage}` | counter | `event.rs:449` |
| `buzz_audit_send_errors_total` | counter | `event.rs:599` |
| `buzz_req_global_access_resolution_skips_total{kind}` | counter | `req.rs:90` |
| `buzz_count_fallback_rejections_total` | counter | `count.rs:190`, `:259` |
| `buzz_workflow_runs_total{trigger,community}` | counter | `event.rs:545-550` |

Cardinality is deliberately managed: kind is fleet-wide only and community is kind-free, because `bounded_kind_label` passes through all 10 000 client-controlled values in 20000–29999 (rationale `event.rs:620-623`).

Tracing spans: `ws.auth` (`connection.rs:505`), `ws.event` (`:524-529`), `ws.req` (`:551`), `ws.count` (`:572`) — each captured *before* the `tokio::spawn` so context propagates (comment `:522-523`).

---

#### 6. Capabilities notably **absent** from the WS surface

| Missing | Evidence |
|---|---|
| NIP-01 `["CLOSED", sub, "…"]` on relay-side subscription eviction by the reaper | `side_effects.rs:62` removes registry entries; no `CLOSED` is emitted from this group's code for that path |
| Any per-IP connection cap | `check_ip_connection` never called (`buzz-pubsub/src/rate_limiter.rs:112` has no relay caller) |
| Subscription-level result limits beyond per-filter `limit` | only `MAX_HISTORICAL_LIMIT` per filter (`req.rs:881`) |
| NIP-45 `HLL` / approximate counts | `RelayMessage::count` emits only `{"count": N}` (`protocol.rs:208`) |
| Compression (permessage-deflate) | no negotiation anywhere in `router.rs:334-342` |
| Backfill resumption / cursors | EOSE is terminal; no `since`-cursor handshake |


## Module: buzz-relay — event ingest & side effects (`crates/buzz-relay/src/handlers`)
### Aspect: Features

Every product capability that writes state passes through `ingest_event`
(`ingest.rs:1367`). This aspect maps features to the branches that realize them, then
lists the mismatches: handled kinds with no reachable feature, and declared kinds ingest
refuses.

---

### 1. Feature → ingest branch map

| Feature | Kinds | Ingest branch | Side effects |
|---|---|---|---|
| **Channel messaging** | 9, 40002 | generic store (`ingest.rs:2417-2427`) + NIP-10 thread resolution (`:2220-2231`) | none; kind:39005 summary emitted for replies (`:2448-2455`) |
| **Message editing** | 40003 | `validate_edit_ownership` (`ingest.rs:1986-1990`) → generic store | none |
| **Message deletion (self / owner)** | 5 | `validate_standard_deletion_event` (`ingest.rs:1947-1951`) → store → `handle_standard_deletion_event` (`side_effects.rs:2108`) | soft-delete + counter decrement + reaction-row removal + kind:39005 |
| **Moderated deletion (channel admin)** | 9005 | `validate_admin_event` 9005 arm (`side_effects.rs:508`) → store → `handle_delete_event_side_effect` (`side_effects.rs:1560`) | soft-delete, counter decrement, kind:39005, kind:40099 tombstone carrying `action_id`/`reason_code`/`public_reason` |
| **Reactions** | 7 | inline transactional path (`ingest.rs:2271-2391`) | none (dedup + `reactions` row happen pre-storage) |
| **Threads (NIP-10) + live badge counts** | any `requires_h` kind with `e root`/`e reply` | `resolve_nip10_thread_meta` (`ingest.rs:564-717`) | `emit_live_thread_summary` → kind:39005 (`side_effects.rs:724`) |
| **Message pinning / bookmarking / scheduling / reminders** | 40004, 40005, 40006, 40007 | generic store, `h` required — **no validator, no side effect** | none |
| **Code-diff messages** | 40008 | `validate_diff_event` (`ingest.rs:896-963`) → generic store | none |
| **Canvas** | 40100 | generic store, `h` required | none |
| **Forums** | 45001, 45002, 45003 | generic store; 45002 gated by `validate_forum_vote_target` (`ingest.rs:844`) | none |
| **Channel lifecycle (create)** | 9007 | eager `create_channel_with_id` (`ingest.rs:2129`) → store → `handle_create_group` (`side_effects.rs:1660`) | kind:40099 `channel_created`, 39000/39001/39002 discovery, kind:44100 to the creator, `buzz_channels_created_total` metric |
| **Channel metadata (name/about/topic/purpose/visibility/ttl/archive)** | 9002 | `validate_admin_event` 9002 arm (`side_effects.rs:410`) → store → `handle_edit_metadata` (`side_effects.rs:1335`) | per-tag DB update, kind:40099 per change type, cache invalidation, subscription eviction on open→private, kind:44100 resubscribe fan-out on unarchive, discovery re-emit |
| **Channel deletion** | 9008 | 9008 arm (`side_effects.rs:625`) → store → `handle_delete_group` (`side_effects.rs:1783`) | soft-delete channel, soft-delete discovery events, cache invalidation, kind:40099 |
| **Channel membership (invite / remove)** | 9000, 9001 | 9000/9001 arms → store → `handle_put_user` / `handle_remove_user` | `channel_members` write, cache invalidation, subscription eviction (remove only), kind:40099, discovery re-emit, kind:44100/44101 |
| **Self-join / self-leave** | 9021, 9022 | `ingest.rs:2134-2154` / 9022 arm → `handle_join_request` / `handle_leave_request` | same shape as 9000/9001 |
| **Agent channel-add policy** | 10100 | store → `handle_agent_profile` (`side_effects.rs:1078`) | `ensure_user`, `set_channel_add_policy`; consumed by the 9000 third-party-add gate (`side_effects.rs:340-365`) |
| **Profiles + NIP-05** | 0 | JSON pre-check (`ingest.rs:2234`) → store → `handle_kind0_profile` (`side_effects.rs:1113`) | absolute-state sync to `users`; NIP-05 canonicalised against the tenant host and silently cleared if off-domain; UNIQUE collision retried without the handle |
| **Direct messages (NIP-59 payload)** | 1059 | store with `channel_id = None`; pubkey-match waived; **WS only** | none |
| **DM channel lifecycle** | 41010, 41011, 41012 | `command_executor.rs:310` / `:443` / `:580` | `open_dm`/`hide_dm`, kind:40099 `dm_created`, 39000/39002 discovery, kind:44100 per participant, kind:30622 NIP-DV snapshot |
| **Workflows (definition)** | 30620 | `command_executor.rs:653` | `upsert_workflow`, webhook-secret generation/preservation, workflow cache invalidation |
| **Workflows (manual trigger)** | 46020 | `command_executor.rs:809` | `create_workflow_run` + spawned `execute_from_step` |
| **Workflow approvals** | 46030, 46031 | `command_executor.rs:986` / `:1098` | approval status update; grant spawns `resume_workflow_after_approval` (`:1236`), deny cancels the run |
| **Workflow deletion** | 5 with an `a` tag `30620:…` | `handle_a_tag_deletion` (`side_effects.rs:1990-2049`) | `delete_workflow_for_owner` by UUID or owner+name, cache invalidation |
| **Community moderation (ban/timeout/report resolution)** | 9040–9044 | direct route (`ingest.rs:1598-1614`) | out of module (`moderation_commands.rs`) |
| **Abuse reporting** | 1984 | direct route (`ingest.rs:1560-1570`) | `moderation_reports` row only |
| **Ban/timeout write enforcement** | all except 9030–9033, 9040–9044 | `moderation_restriction_state` gate (`ingest.rs:1613-1642`) | — |
| **Relay membership admin (NIP-43)** | 9030–9033 | direct route (`ingest.rs:1834-1844`) | 8000/8001 delta + 13534 snapshot from `relay_admin.rs` |
| **Self-service relay leave** | 28936 | direct route (`ingest.rs:1846-1928`) | `publish_nip43_member_removed` + `publish_nip43_membership_list` (both fire-and-forget) |
| **Identity archival (NIP-IA)** | 9035, 9036 | pre-storage handler then normal storage (`ingest.rs:1941-1945`) | 8002/8003 delta + 13535 snapshot |
| **Agent memory / engrams (NIP-AE)** | 30174 | `validate_engram_envelope` (`ingest.rs:2002-2005`) → NIP-33 store | none |
| **Agent telemetry (NIP-AM)** | 44200 | envelope + owner check (`ingest.rs:1981-2016`) → store | none |
| **Personas / teams / managed agents (NIP-AP)** | 30175, 30176, 30177 | 30175 gets `validate_persona_envelope`; 30176/30177 only the `d`-length check | none |
| **Event reminders (NIP-ER)** | 30300 | `validate_event_reminder` (`ingest.rs:2044-2047`) → NIP-33 store | none |
| **Web push leases (NIP-PL)** | 30350 | direct route (`ingest.rs:2156-2204`) | `push_leases` table |
| **Media attachments** | any kind with `imeta` tags | `validate_imeta_tags` + `verify_imeta_blobs` (`ingest.rs:2232-2244`) | none — the blob was uploaded out-of-band via Blossom |
| **Git repos (NIP-34)** | 30617, 30618, 1617–1621, 1630–1633 | 30617 → store → `handle_git_repo_announcement` (`side_effects.rs:2412`); the rest are plain stores | name reservation, empty-manifest pointer seed/repair, initial kind:30618 |
| **Huddles (audio)** | 48100–48103, 48106 | generic store, `h` required — **no validator, no side effect** | none |
| **User status / read state / NIP-51 lists / NIP-65 / NIP-30 emoji** | 30315, 30078, 10000, 10001, 10002, 10003, 10030, 30000, 30003, 30030 | replaceable / NIP-33 store, forced global | none |
| **Long-form posts** | 30023 | NIP-33 store, forced global | deletion supported via the generic coordinate soft-delete (`side_effects.rs:2051-2088`, referencing block/sprout#714) |
| **Product feedback** | 42000 | direct route (`ingest.rs:1538-1558`) | private deployment table |
| **Text notes** | 1 | generic store, forced global | none |
| **Contact list** | 3 | replaceable store, forced global | none |
| **Operational reconciliation** | — | not ingest; `main.rs` startup/periodic tasks | `reconcile_nip43_membership_snapshots` (`side_effects.rs:2776`, called `main.rs:508`, `:527`), `reconcile_channel_events` (`:2948`, called `main.rs:577`), `evict_all_channel_subscriptions` (`:128`, called `main.rs:672` by the ephemeral-channel reaper) |

---

### 2. Handled kinds whose feature has **no other reachable surface**

| Kind | Situation |
|---|---|
| 40004 `STREAM_MESSAGE_PINNED`, 40005 `…_BOOKMARKED`, 40006 `…_SCHEDULED`, 40007 `STREAM_REMINDER` | Accepted and stored with only the `h`-tag gate (`ingest.rs:455-491`). No pre-storage validator, no side effect, no `is_side_effect_kind` membership. Semantics are entirely client-side convention; the relay cannot tell a pin from a scheduled message. Nothing enforces that the `e` target exists or is in the same channel — unlike 40003 (`ingest.rs:763`) and 45002 (`:844`), which do check. |
| 48100–48103, 48106 huddles | Same: stored, `h` required, zero relay-side validation. A client can post `HUDDLE_ENDED` for a huddle that never started. |
| 40100 `CANVAS` | Same. |
| 30176 `TEAM`, 30177 `MANAGED_AGENT` | Accepted with only the 1024-byte `d`-length check. 30175 `PERSONA` gets a strict slug grammar (`ingest.rs:1027`) precisely because an empty `d` causes LWW data loss — 30176/30177 share that exact addressing shape and have **no** such guard, so an empty `d` collapses every team/managed-agent into one slot. Asymmetric by omission, not by design. |
| 30618 `GIT_REPO_STATE` | Client-submittable with `ReposWrite` scope, but the relay also mints it itself (`side_effects.rs:2733`, and `api/git/manifest_event.rs` on push). A client can publish a competing 30618 for a repo it does not own — the `d`-tag coordinate includes the *submitter's* pubkey, so it cannot overwrite the relay-signed head, but it can pollute the kind. |
| 1617–1621, 1630–1633 git patches/PRs/issues/statuses | Stored with `MessagesWrite` and forced global. No validation that the referenced repo exists or that the author has any relationship to it. |

---

### 3. Declared kinds ingest **rejects** — feature reachability

47 of the 127 `ALL_KINDS` entries never pass `required_scope_for_kind`
(`ingest.rs:198-306`). Grouped by why:

| Group | Kinds | Assessment |
|---|---|---|
| **Relay-signed outputs of this very module** | 8000, 8001, 8002, 8003, 13534, 13535, 39000, 39001, 39002 | Correct to reject. But note only 13534 is in `is_relay_only_kind` (`buzz-core/src/kind.rs:758-769`); the other eight fall through to the generic `restricted: unknown event kind`. Same outcome, worse diagnostic. |
| **Relay-signed, correctly classified** | 30622, 39005, 39006, 40901, 40902 | `is_relay_only_kind` → `restricted: relay-only kind` (`ingest.rs:1481`). |
| **Relay-signed, special-cased** | 44100, 44101 | Own message: `invalid: membership notifications are relay-signed only` (`ingest.rs:1469`). |
| **Relay-signed, no guard at all** | 40099 `SYSTEM_MESSAGE` | Minted by `emit_system_message` (`side_effects.rs:677`) but absent from `is_relay_only_kind`. It appears in `is_side_effect_kind` (`side_effects.rs:36`), which is dead code since ingest rejects it. |
| **Ephemeral (handled upstream)** | 20001, 20002, 24134, 24200, 24810 | Dispatched in `handlers/event.rs` before ingest; `ephemeral_kinds_not_in_scope_allowlist` (`ingest.rs:2790-2793`) asserts the exclusion. |
| **Auth artefacts** | 24242 `BLOSSOM_AUTH`, 27235 `HTTP_AUTH` | Consumed by the media and NIP-98 paths, never ingested. Correct. |
| **Unimplemented features** | 9009 `NIP29_CREATE_INVITE`, 39003 `NIP29_GROUP_ROLES`, 41 `CHANNEL_METADATA`, 1063 `FILE_METADATA`, 41001 `DM_CREATED`, 43001–43006 job kinds (6), 46001–46012 workflow-execution kinds (10), 48001 `AUDIT_ENTRY`, 49001 `MEDIA_UPLOAD` | **26 declared kinds with no ingest path.** The job kinds (43001–43006) and workflow-execution kinds (46001–46012) are declared, documented in `buzz-core/src/kind.rs:456-522`, and unreachable: nothing can publish them. 48001 `AUDIT_ENTRY` is particularly notable — the audit log is Postgres-only (`buzz-audit/src/service.rs`), so this kind exists purely as a reservation. |

The single most concrete mismatch: **9009** has a live `handle_side_effects` arm
(`side_effects.rs:161-167`) that logs `"NIP-29 kind 9009 handler deferred to future
phase"` and returns `Ok`. It cannot be reached — 9009 is not in the allowlist. The arm is
provably dead.

---

### 4. Cross-crate feature mismatches found

| Mismatch | Evidence |
|---|---|
| **Workflow `add_reaction` action is broken.** `buzz-workflow` POSTs to `{base_url}/api/messages/{message_id}/reactions` (`buzz-workflow/src/executor.rs:883-889`). No such route exists — the relay registers exactly three `/api/*` paths (`router.rs:95`, `:96`, `:111`) and no `/api/messages` prefix anywhere. Meanwhile ingest **does** have a full native reaction path (kind:7, `ingest.rs:2271-2391`) that the workflow action does not use. Any workflow with an `add_reaction` step will 404 and surface `AddReaction: relay returned {status}` (`executor.rs:917`). | `buzz-workflow/src/executor.rs:883-917`; `crates/buzz-relay/src/router.rs` |
| **Reaction-triggered workflows can fire but cannot react back.** `reaction_added` is a valid workflow trigger (`buzz-workflow/src/schema.rs:290`), and kind:7 does reach `dispatch_persistent_event` → `workflow_engine.on_event` (`handlers/event.rs:525-554`), so the trigger works. Only the response action is dead. | as above |
| **9 of 11 audit actions have no producer.** `AuditAction` declares 11 variants (`buzz-audit/src/action.rs:8-31`). Production emits exactly two: `EventCreated` (`handlers/event.rs:583`) and `MediaUploaded` (`api/media.rs`). `EventDeleted`, `ChannelCreated`, `ChannelUpdated`, `ChannelDeleted`, `MemberAdded`, `MemberRemoved`, `AuthSuccess`, `AuthFailure`, `RateLimitExceeded` are never written — despite this module performing every one of those actions. | grep for `AuditAction::*` outside `crates/buzz-audit/` and `tests/` |
| **`is_side_effect_kind` claims two ranges that no kind can reach.** `41001..=41003` and `40099` (`side_effects.rs:36`) — 41001 is `DM_CREATED` (not in the allowlist), 41002/41003 are undefined, 40099 is relay-signed. The DM *command* kinds are 41010–41012 and are routed to `command_executor` long before this predicate runs (`ingest.rs:1560`). | `side_effects.rs:35-37` vs `ingest.rs:198-306` |


## Module: buzz-relay — HTTP API surface (`crates/buzz-relay/src/api`)
### Aspect: Features

---

#### 1. Product capability → endpoint map

| Capability | Endpoints | Handler `file:line` | Consumers (verified by grep) |
|---|---|---|---|
| **Send a message / publish any event over HTTP** | `POST /events` | `bridge.rs:613` | `buzz-cli` (`client.rs:874`, `:1025`), desktop Tauri (`huddle/pipeline.rs:321`, `commands/personas/snapshot/import.rs:550`), desktop E2E bridge (`e2eBridge.ts:5189`) |
| **Read messages / channels / profiles over HTTP** | `POST /query` | `bridge.rs:880` | `buzz-cli` (`client.rs:774`), mobile (`relay_session.dart:136`), desktop timeline via the bridge (`channelWindowResponse.ts:84`, `e2eBridge.ts:4219`, `:4368`, `:4527`) |
| **Unread / badge counts** | `POST /count` | `bridge.rs:1314` | `buzz-cli` (`client.rs:804`) |
| **Server-assembled channel timeline (read model)** | `POST /query` with `top_level` / `include_aux` / `include_summaries` | `bridge.rs:404-581` | desktop (`channelWindowResponse.ts`, `e2eBridge.ts:4527`) |
| **Threaded replies with keyset pagination** | `POST /query` with `depth_limit` + `thread_cursor` | `bridge.rs:1112-1183`, `:305-345` | desktop `get_thread_replies` paging (doc at `bridge.rs:307-315`); `buzz messages thread` |
| **Activity / mentions / needs-action feeds** | `POST /query` with `feed_types` | `bridge.rs:1029-1109` | desktop feed views |
| **Full-text search (NIP-50), prefix + paged** | `POST /query` with `search` | `bridge.rs:1616-1749` | desktop search; `buzz messages search` |
| **Presence (online/away)** | `POST /query` for kind:20001/40902 | `bridge.rs:1920-1985` | desktop/mobile presence; replaced the removed `GET /api/presence` (`mobile/test/features/profile/presence_cache_provider_test.dart:13`) |
| **Moderation queue, audit log, active restrictions** | `GET /moderation/reports\|audit\|restricted` | `bridge.rs:2092`, `:2100`, `:2118` | desktop (`shared/api/moderation.ts:340-373`, `features/settings/lib/moderationQueue.ts`), `buzz-cli` (`commands/moderation.rs:110`, `:120`, `:127`) |
| **Workflow automation triggered by external systems** | `POST /hooks/{id}` | `bridge.rs:1780` | external callers only — no in-repo client |
| **Media upload (images, video, generic files)** | `PUT /upload`, `PUT /media/upload` | `media.rs:305` | desktop Tauri (`commands/media.rs:487`, `:498` with legacy fallback), mobile (`relay_client.dart:23`, `media_upload.dart:19`), `buzz-cli` (`client.rs:1144`, `:1197`) |
| **Media download, thumbnails, video seeking** | `GET/HEAD /media/{sha256_ext}` | `media.rs:604`, `:798` | desktop markdown/video (`mediaEntry.ts:75`, `MarkdownVideoPlayer.tsx:36`, `VideoPlayer.tsx:96`), `buzz-cli` (`client.rs` `/media/{hash}.jpg`) |
| **Invite a new member to a closed relay** | `POST /api/invites` | `invites.rs:230` | desktop (`shared/api/invites.ts:183`, `InviteLinkSection.tsx:44`) |
| **Redeem an invite (self-service join)** | `POST /api/invites/claim` | `invites.rs:291` | desktop (`invites.ts:203`, `useClaimInvite.ts`), web (`invite-api` via `InvitePage.tsx:110`), mobile (`invite_join_provider.dart:248`) |
| **Terms/privacy consent gate before join** | `GET /api/join-policy`, `/terms`, `/privacy`, `POST /api/invites/accept-policy` | `invites.rs:75`, `:95`, `:104`, `:162` | desktop (`invites.ts:106`, `:129`, `:160`, `JoinPolicyNotice.tsx`), web (`InvitePage.tsx:69`, `:80`) |
| **Invite landing page (browser)** | SPA fallback `/invite/{code}` | `router.rs:191-194` (`is_invite_landing_path`) | `web/src/features/invite/ui/InvitePage.tsx` |
| **NIP-05 identity resolution** | `GET /.well-known/nostr.json` | `nip05.rs:26` | **no product client** — only E2E/conformance tests (`e2e_relay.rs:1061`, `:1106`; `conformance_multitenant.rs:1201`, `:1242`); external Nostr clients are the real audience |
| **Community provisioning / lifecycle (control plane)** | 6 × `/operator/communities*` | `operator.rs:149`, `:203`, `:265`, `:302`, `:354`, `:468` | **no in-repo client at all** — see §2 |
| **Deployment-wide moderation + product-feedback dashboard** | 5 × `/api/admin/v1/*` | `admin/mod.rs:93`, `:125`, `:151`, `:177`, `:191` | `admin-web/` SPA (`admin-web/src/api.ts:1`, `App.tsx:520`, `tests/routes.spec.ts`) |
| **Mesh reliable-stream smoke test** | `POST /_mesh/demo/echo` | `mesh_demo.rs:58` | **none** — self-described as "Not a product flow" (`mesh_demo.rs:21-23`) |

#### 2. Endpoints with no in-repo client

Grepped `desktop/src`, `desktop/src-tauri`, `web/src`, `mobile/lib`, `crates/buzz-cli`,
`crates/buzz-ws-client`, `crates/buzz-sdk`, `crates/buzz-admin`, `crates/buzz-pairing-cli`,
plus a repo-wide sweep.

| Endpoint | Status | Notes |
|---|---|---|
| `GET /operator/communities` | zero clients | Only relay code + its own `#[ignore]`d tests reference the path (`operator.rs:312`, `:769`). Intended for an out-of-repo control plane (likely the buzz.xyz console). |
| `POST /operator/communities` | zero clients | same |
| `POST /operator/communities/archive` | zero clients | same |
| `POST /operator/communities/unarchive` | zero clients | same |
| `GET /operator/communities/availability` | zero clients | same |
| `POST /operator/communities/transfer` | zero clients | same |
| `POST /_mesh/demo/echo` | zero clients | testbed-only, double-flag-gated |
| `GET /.well-known/nostr.json` | zero product clients | tests only, in-repo |
| `POST /hooks/{id}` | zero in-repo clients | by design (external webhook senders) |
| `GET /api/admin/v1/*` | in-repo client is `admin-web/`, a separate SPA bundle | not shipped inside desktop/mobile |

**Consequence:** 6 of the 30 method×path pairs in this module group (20%) are an unexercised
control-plane surface from this repository's perspective; their only coverage is 11 `#[ignore]`d
Postgres-backed tests in `operator.rs`.

#### 3. Features that exist in code but are unreachable

| Item | `file:line` | Why |
|---|---|---|
| `api/events.rs` re-export shim | `api/events.rs:1-5` | Zero references repo-wide; `router.rs:71-73` binds `api::bridge::*` directly. Its stated reason ("backward compatibility with router.rs") is false. |
| `webhook_secret::strip_secret` | `webhook_secret.rs:57` | Zero production callers. The documented invariant "the secret must never be embedded in a response body" therefore has no enforcement point in the HTTP surface. |
| `POST /api/messages/{id}/reactions` | absent from `router.rs:32-190` | `buzz-workflow`'s `add_reaction` action POSTs here (`buzz-workflow/src/executor.rs:889`), so that workflow action always fails against this relay. |
| `GET /api/presence` | absent | `ARCHITECTURE.md:824` describes it as existing; it was removed (`mobile/test/features/profile/presence_cache_provider_test.dart:13`). |
| `HttpAuthMethod::DevPubkey` | `handlers/ingest.rs:58` | Never constructed; `bridge.rs:830` always reports `Nip98`. |
| `IngestAuth::Http.auth_method` | `bridge.rs:830` | Never read by any consumer. |
| `relay_members::check_relay_membership` / `MembershipDecision` | `api/mod.rs:61`, `:46` | Single caller (`api/mod.rs:130`) inside the same module; the transport-neutral indirection has no second consumer. |
| `MediaError::DisallowedContentType` on `/upload` | `media.rs:384-388` | Only reachable via the **legacy** `/media/upload` alias; the modern `/upload` route accepts generic files. |

#### 4. Feature flags / staged rollouts visible here

| Flag | Default | Gated feature | `file:line` |
|---|---|---|---|
| `require_auth_token` | **false** | real NIP-98 vs `X-Pubkey` header on `/events`, `/query`, `/count`, `/moderation/*` | `bridge.rs:118-127`; `config.rs:475-477` |
| `require_relay_membership` | **false** | closed-relay admission on `/events`, `/query`, `/count`, uploads, git | `api/mod.rs:67-69`; `config.rs:483-485` |
| `allow_nip_oa_auth` | **false** | NIP-OA owner delegation *for membership admission* (the owner **backfill** path is unflagged because the signature is self-proving) | `api/mod.rs:81`, `:151-156`; `config.rs:520-522` |
| `require_media_get_auth` | **false** | Blossom read auth on `GET/HEAD /media/*`; also flips `Cache-Control` public→private | `media.rs:491-514`, `:517-523`; `config.rs:682-689` |
| `join_policy` (derived from 3 env vars) | `None` | the consent gate on `claim` and the three policy endpoints | `config.rs:794-811`; `invites.rs:314-322` |
| `admin` (`BUZZ_ADMIN_HOST`) | `None` | the entire `/api/admin/v1` sub-router **and** its security-header middleware | `router.rs:56-59`; `config.rs:813-841` |
| `relay_operator_pubkeys` + `relay_operator_api_origin` | empty / `None` | effective usability of the 6 operator routes (routes always registered; requests 403 or 500 when unconfigured) | `operator.rs:69-98`; `config.rs:549-581` |
| `mesh` + `mesh_demo_echo` | false / false | `/_mesh/demo/echo` (404 unless both on) | `mesh_demo.rs:64-70`; `config.rs:509-518` |
| `upload_records_enabled` | **false** | per-upload `_uploads/` attribution records incl. uploader display name and edge IP | `media.rs:246-249`; `config.rs:651-653` |
| `serve_git_web_gui` | false | SPA fallback for `/`, `/repos*` (invite landing is always served) | `router.rs:196-198`; `config.rs:848-851` |

#### 5. Explicitly deferred / stubbed

| Item | `file:line` | Statement |
|---|---|---|
| Persistent per-pubkey storage quotas for media | `media.rs:302-304` | `TODO(v2)` — admission limits bound *active* work but do not cap durable bytes stored. The only TODO marker in all 13 files. |
| Per-invite-code revocation | `invite_token.rs:43-46` | "Revocation is coarse: rotate the relay keypair… Per-code revocation requires the future `relay_invites` table increment." |
| `/media/upload` legacy alias | `media.rs:5-7`, `:57-63`, `:313-317` | "temporary media-only legacy alias"; usage tracked via `buzz_media_legacy_upload_route_total`. Still called by desktop as a 404/405 fallback (`desktop/src-tauri/src/commands/media.rs:498`) and `buzz-cli` (`client.rs:1183-1197`). |
| `/_mesh/demo/echo` | `mesh_demo.rs:21-23` | "The real join-side consumer (goose/berd session wiring) replaces this; the route stays demo-gated until it is deleted." |
| Timestamp-only thread cursor | `bridge.rs:316-320` | "still accepted and paginates on time alone (unsafe across same-second ties)" |


## Module: buzz-relay — git hosting (`crates/buzz-relay/src/api/git`)
### Aspect: Features

#### 1. What this module is

A git server with **no persistent per-repo filesystem**. Every request materializes an ephemeral bare repo from an object-store manifest, shells out to stock `git`, and drops the tree (`hydrate.rs:1-27`, doc §v1 deployment architecture). Refs commit by compare-and-swap on a single manifest pointer (`cas_publish.rs:1208-1226`).

| Capability | Status | Evidence |
|---|---|---|
| Smart HTTP clone / fetch | supported | `transport.rs:786-827` |
| Smart HTTP push | supported | `transport.rs:858-962` |
| Ref advertisement | supported, two implementations | fast path `transport.rs:471-537`; subprocess `:596-724` |
| Dumb HTTP (`objects/info/packs`, loose-object URLs) | **absent** — no routes registered | `transport.rs:1758-1763` |
| `git://` daemon, SSH transport | **absent** | no listener anywhere in the crate |
| Protocol v2 (`Git-Protocol: version=2`) | **not implemented server-side**; the request header is never read, and the fast-path advertisement is a v0 `# service=` stream. The subprocess path inherits whatever the installed `git upload-pack --advertise-refs` does without the env var set, i.e. v0. | `transport.rs:471-537`, `:645-653` |

#### 2. Git operations

| Operation | Supported | Mechanism / limit |
|---|---|---|
| Full clone | yes | subprocess `upload-pack`, streamed (`transport.rs:1414-1498`) |
| Fetch / incremental | yes | same path |
| **Shallow clone** (`--depth`) | yes, via subprocess. The fast-path advertisement offers `shallow deepen-since deepen-not deepen-relative` as a fixed string (`transport.rs:484-491`); the real negotiation happens against `git upload-pack` in the follow-up POST. Exercised in production by the web repo browser (`web/src/features/repos/git-client.ts:86-113`, `depth: 1`). |
| `--single-branch`, `--no-tags` | yes (client-side negotiation, server-transparent) | `web/src/features/repos/git-client.ts:100-112` |
| **Partial clone** (`filter=blob:none`, promisor) | **not offered**. `filter` is absent from the fast-path capability string (`transport.rs:484-494`); the subprocess path may advertise it if the installed git does, but no promisor remote/fetch-object protocol exists on the relay. | |
| **Thin pack** | offered (`thin-pack` in caps, `transport.rs:485`) | |
| `side-band` / `side-band-64k` | offered; report-status parsing explicitly handles the nested band-1 framing | `transport.rs:485`, `:1181-1208` |
| Branch create / fast-forward | yes | `git_perms.rs:403-428` defaults Member |
| **Force push** (non-fast-forward) | yes, Admin by default, deniable per-pattern via `no-force-push` | `git_perms.rs:419-427`, `:545-551`; e2e `crates/buzz-test-client/tests/e2e_git.rs:197` |
| **Branch deletion** | yes, Admin by default, deniable via `no-delete`. `capture_pack` handles delete-all (no positive tips ⇒ `None`) and refs-only deletes. | `git_perms.rs:428`, `:553-559`; `cas_publish.rs:507-511` |
| **Tags** | stored and served, but any `refs/tags/*` **disables the advertisement fast path** because the manifest cannot reproduce an annotated tag's `^{}` peel line. Tag *creation* defaults to Member; tag *move* to Admin. | `transport.rs:388-403`; `git_perms.rs:410-424` |
| Atomic push (`atomic` capability) | not offered by the fast path (receive-pack advertisement always goes through the subprocess, so whatever git offers applies). Policy evaluation is nonetheless all-or-nothing. | `transport.rs:549-551`; `git_perms.rs:584-600` |
| **Git LFS** | **absent** — no `objects/batch` endpoint, no `lfs` string anywhere in the module. | |
| **Submodules** | no special handling; gitlinks are ordinary objects inside packs. No recursive fetch support, no `.gitmodules` awareness. | |
| **Signed pushes** (`push-cert`) | not offered; `receive-pack` runs without `--signed`. Object-level signing is a separate concern handled by the `git-sign-nostr` crate. | `transport.rs:1019-1027` |
| **Server-side hooks other than pre-receive** | absent. Only `hooks/pre-receive` is installed; `update`, `post-receive`, `post-update` are never written. | `hook.rs:152-178` |
| **Repository deletion / rename** | absent. No protocol path deletes a pointer or a repo; the doc's "no deletion under the protocol" rule (doc §Axioms A1) is upheld by omission. | |
| Garbage collection / repack of stored objects | only proactive **pack compaction** during an accepted push; old packs are never deleted. Physical pruning is an out-of-band retention concern. | `cas_publish.rs:666-875`, doc §Theorem 2 Remark |
| SHA-256 repositories | advertised as derivable (`object-format=sha256` from oid width, `transport.rs:473-479`) and accepted by `is_hex_oid`, but **not usable**: `init_bare_repo` always creates SHA-1 (`hydrate.rs:181-184`) and `snapshot_workspace_state` drops non-40-hex oids (`cas_publish.rs:325-328`). |

#### 3. Performance / operability features

| Feature | Description | Site |
|---|---|---|
| Manifest advertisement fast path | Serves `info/refs` for `git-upload-pack` straight from the verified manifest — no hydrate, no subprocess, **no semaphore permit**. Gated by `fast_path_eligible`, which doubles as a safety re-check. Byte-compatible with `git upload-pack --advertise-refs` for the branches-only case. | `transport.rs:380-537`, `:552-577` |
| Streamed fetch | `upload-pack` stdout is streamed into the HTTP body instead of buffered; the child, the tempdir, the stdin pump, and the semaphore permit are all kept alive until EOF or client disconnect. | `transport.rs:1262-1332`, `:1414-1498` |
| Streaming deadline + metrics | 300 s in-band deadline that kills the subprocess; `buzz_git_upload_pack_{timeouts_total, stream_seconds, stream_bytes}`. | `transport.rs:1282-1391` |
| Bounded pack/index cache | Process-local, digest-keyed, byte-bounded, LRU-evicted, single-flight populated, hard-linked into workspaces. | `pack_cache.rs:66-456` |
| Cross-process cache adoption | A lost publication rename is resolved by adopting the winner's directory — designed for several relay processes sharing a scratch volume. | `pack_cache.rs:314-330` |
| Stale-session GC | Abandoned `session-*` cache dirs older than 10 min are removed at startup. | `pack_cache.rs:482-509` |
| Proactive pack compaction | At ≥ 96 parent packs an accepted push captures the full post-push closure into ≤ 128 bounded packs and replaces the pack list. | `cas_publish.rs:1046-1103`, `:666-875` |
| gzip request-body inflation | Git gzips large negotiation bodies; the relay inflates transparently with an independent decoded-byte cap. Without it the subprocess dies with `bad line length character`. | `transport.rs:726-784` |
| Startup conformance gate | Fatal A1/A3 probe against the configured bucket before the listener opens. | `store.rs:571-877`, `main.rs:466-503` |
| Observability | 8 counters, 11 histograms, 3 gauges under `buzz_git_*`. | see integrations aspect |

#### 4. Authorization features

| Feature | Description | Site |
|---|---|---|
| NIP-98 transport auth | Repo-root-scoped, tenant-host-bound, ±60 s window. Method and body-hash binding intentionally waived. | `transport.rs:76-231` |
| NIP-OA managed-agent delegation | An agent's attestation may travel in the signed NIP-98 event's `auth` tag (git cannot carry `x-auth-tag` through the credential helper). | `transport.rs:204-210` |
| Managed-agent owner authority | A verified NIP-OA *owner* of the repo-owner key gets `MemberRole::Owner` on that repo. | `policy.rs:343-368` |
| Channel-bound roles | Non-owners derive their role from the channel named by kind:30617's `buzz-channel` tag. | `policy.rs:308-395` |
| Bot promotion | `MemberRole::Bot` → `Member` for git pushes only. | `policy.rs:397-402` |
| Ref protection via `buzz-protect` tags | Glob patterns (≤ 3 wildcards, ≤ 256 chars, ≤ 50 rules) with `push:<role>`, `no-force-push`, `no-delete`, `require-patch`; unioned strictest-wins; explicit rules can never weaken defaults. | `git_perms.rs:244-262`, `:445-486`, `:534-550` |
| Archived-channel read-only | An archived bound channel denies **all** pushes, owner included. | `policy.rs:308-330` |
| Per-pubkey repo quota | Default 100, enforced at announce. | `handlers/side_effects.rs:2486-2492` |
| Repo-name reservation | `(community_id, repo_id)` unique in Postgres via `INSERT … ON CONFLICT DO NOTHING`. | `handlers/side_effects.rs:2441-2513` |
| **Read authorization** | **absent.** No per-repo, per-channel, or visibility check on `info/refs`/`upload-pack`. `transport.rs:60-66` claims "No public repos for v1", but in practice every repo is readable by any caller that clears the transport gate. | absence in `transport.rs:539-827` |

#### 5. Repo browser surface

The relay does not render repository content itself. Two cooperating surfaces:

| Surface | Mechanism | Site |
|---|---|---|
| Repo/branch discovery | `POST /query` for kinds 30617 (repos) and 30618 (ref state) — plain Nostr, not this module. The web client explicitly notes it must not trust arbitrary authors' 30618. | `web/src/features/repos/use-repos.ts:66`, `use-repo-refs.ts:54-57` |
| File/tree/blob browsing | **client-side**: `isomorphic-git` over LightningFS/IndexedDB does a `depth:1, singleBranch, noTags` clone of `/git/{owner}/{repo}.git` with a NIP-98 header, then reads trees/blobs locally. | `web/src/features/repos/git-client.ts:17-140` |
| SPA route gating | The relay serves `/`, `/repos`, `/repos/*` from the web bundle only when `BUZZ_SERVE_GIT_WEB_GUI` is truthy. | `router.rs:206-213`; config `config.rs:848-850` |

Consequences: no server-side tree/blob/commit API exists, browsing costs a real clone per repo per browser, and the browser's IndexedDB holds full repo content. There is no server-side diff, blame, or search over repository contents.

#### 6. Not implemented (explicit non-features)

- No repo delete/rename/transfer protocol.
- No `receive-pack` `--signed` / push certificates.
- No `update`/`post-receive` hooks, so no server-side CI trigger from git itself (workflows are driven by kind:30618 events instead).
- No mirror/replication, no cross-relay fetch.
- No pack-content policy: received packs are never inspected for object types, sizes, or path names beyond what `git index-pack` validates.
- No per-repo read ACL, no anonymous/public read mode (both directions of that missing knob: no anonymous access, and no way to *restrict* read below community membership).
- No liveness/latency guarantee — explicitly out of scope in the design (doc §Scope and Non-Goals).


## Module: buzz-relay — moderation, admin & background workers (`crates/buzz-relay/src`)
### Aspect: Features

---

#### 1. Feature inventory

| # | Subsystem | File(s) | Status | Entry point |
|---|---|---|---|---|
| F1 | Community moderation commands (ban/unban/timeout/untimeout/resolve) | `handlers/moderation_commands.rs` (768 LOC) | working | `ingest.rs:1606` |
| F2 | Moderation authorization capability seam | `handlers/moderation_authz.rs` (335) | working, **partly dead** | `moderation_commands.rs` ×5, `api/bridge.rs:2055` |
| F3 | Relay-signed moderation notice DMs | `handlers/moderation_notices.rs` (398) | working, **1 of 3 templates unused** | `moderation_commands.rs` ×3 |
| F4 | NIP-56 report intake | `handlers/report.rs` (337) | working | `ingest.rs:1588` |
| F5 | NIP-43 relay-membership admin + workspace icon | `handlers/relay_admin.rs` (468) | working | `ingest.rs:1835` |
| F6 | Operator community provisioning | `handlers/community_provisioning.rs` (445) | working | `api/operator.rs:171` |
| F7 | NIP-PL push-lease validation + acceptance | `handlers/push_lease.rs` (771) | working, **`urgent` class dead** | `ingest.rs:2182` |
| F8 | NIP-PL durable matcher + APNs delivery worker | `push_runtime.rs` (656) | working, **no leader election** | `main.rs:687-690` |
| F9 | NIP-IA identity archive/unarchive | `handlers/identity_archive.rs` (580) | working | `ingest.rs:1942` |
| F10 | Product feedback sidecar | `handlers/product_feedback.rs` (161) | working | `ingest.rs:1567` |
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

Because no caller passes a `channel_id` (all 6 sites pass `None`) and no caller requests `DeleteMessage`/`Kick`, the `get_member_role` lookup (`:120-131`) is unreachable and `ModerationAuthority::ChannelRole` (`:174-178`) can never be returned. The 9005-delete and 9001-kick paths use a **separate, unrelated** authorization function, `side_effects::validate_admin_event` (`ingest.rs:1929-1933`) — the very function the module doc says the seam is "the bridge … missing today" for (`moderation_authz.rs:73-75`). The bridge exists but is not connected.

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

**Missing relative to the moderation path:** no ban re-check. `relay_admin.rs` contains zero references to `moderation_restriction_state`, and `ingest.rs:1639` exempts relay-admin kinds from the write gate — so a banned owner/admin with a surviving socket can still mutate membership.

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

Requests deliberately fall through to normal storage so the delta's `["e", request_id]` audit reference resolves (`ingest.rs:1935-1938`) — meaning 9035/9036 are the **only** kinds in this module that produce an `events` row and therefore the only ones that reach the `buzz-audit` hash chain.

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


## Module: buzz-relay — huddle audio, tunnel & conformance seam (`crates/buzz-relay/src`)
### Aspect: Features

---

#### 1. Huddle audio — capability inventory

| Capability | Where | Completeness |
|---|---|---|
| WebSocket audio room per (community, channel) | `audio/room.rs:490-550`, route `router.rs:125-128` | Complete for single-pod |
| NIP-42 challenge/response auth on the audio socket | `handler.rs:176-238` | Complete — reuses `state.auth.verify_auth_event`, the same verifier as the main relay door |
| Channel-membership gate with ephemeral auto-add | `handler.rs:1153-1235` | Complete; auto-add restricted to TTL channels whose caller is a parent member |
| Opaque Opus fan-out with 1-byte `peer_index` prefix | `room.rs:391-411` | Complete; never decodes, never re-encodes |
| v2 frame header parse + telemetry clamp | `audio/wire.rs:29-88` | Complete; 4 tests pin byte order, short input, clamp-keeps-frame, reserved-bit passthrough |
| Room protocol-version pinning | `room.rs:110-127`, `:239-247` | Complete; 5 tests |
| Peer-index allocation with recycling | `room.rs:143-157` | Complete; 0..=254 |
| Separate audio / control queues with priority drain | `room.rs:23-27`, `handler.rs:1060-1086` | Complete |
| Heartbeat + missed-pong disconnect | `handler.rs:1127-1151` | Complete (30 s / 3 misses) |
| Auto-end + channel archive when the room empties | `room.rs:358-388`, `handler.rs:833-866` | Complete, with rollback on archive failure |
| Lifecycle events 48101 / 48102 / 48103 (relay-signed, persisted, fanned out locally, published to Redis) | `handler.rs:645-653`, `:822-831`, `:850-859`, `:1237-1332` | Complete |
| Ordered roster snapshot/delta model | `room.rs:55-81`, `:452-476` | Complete; broadcast cap 64 with snapshot recovery |
| Cross-pod huddle: owner-authoritative fan-out over the mesh | `audio/join.rs`, `audio/mesh.rs` | Implemented end-to-end; **only active when `BUZZ_MESH=on`** (default off) |
| Redis fenced-CAS ownership with generation monotonicity | `tunnel/directory.rs:20-79`, `:348-439` | Complete |
| Per-room owner-lease renewer with observable loss | `join.rs:452-562`, `:600-773` | Complete |
| Graceful drain of owned huddles on SIGTERM | `join.rs:750-773`, `mesh_boot.rs:481-496` | Complete |
| Ambiguity-safe media routing (fail closed on channel-UUID collision) | `room.rs:526-541`, `mesh.rs:221-227` | Complete |

---

#### 2. Absent from huddle audio (verified by reading, not inferred)

| Feature | Evidence of absence |
|---|---|
| **Recording** | No writer, no storage call, no media handle in `crates/buzz-relay/src/audio/`. `ARCHITECTURE.md:825` lists "Huddle recording/tracks not built" as known-absent; confirmed |
| **Transcription / STT** | Not in the relay. STT is client-side: `desktop/src-tauri/src/huddle/stt.rs` |
| **Video** | No video kind, no video track, no SDP. `buzz-core/src/kind.rs:530` calls 48100 "audio/video session" but nothing in the relay handles video |
| **Screen share** | No reference anywhere in `crates/buzz-relay/src` |
| **SFU mixing / transcoding** | The relay never decodes: `broadcast_frame` copies bytes with a 1-byte prefix (`room.rs:398-411`). Fan-out is N×(N−1) mesh, not a mixer. No Opus, WebRTC, or codec crate in `crates/buzz-relay/Cargo.toml` |
| **Echo cancellation / noise suppression / AGC** | None; no DSP dependency |
| **TURN / STUN / ICE** | None. Clients connect straight to the relay over WSS. The default `BUZZ_MESH_BIND_ADDR` port `3478` (`config.rs:507`) is the *STUN* port number but the protocol spoken there is iroh/QUIC, not STUN — a naming coincidence, not a TURN server |
| **Codec negotiation** | Nothing is negotiated except the integer `protocol_version` (`handler.rs:134-138`). Sample rate, channel count, frame duration, and bitrate are never exchanged. The 48 kHz assumption lives only in a field name (`wire.rs:44`) and the "20 ms/frame" comment behind the queue sizing (`room.rs:40`) |
| **Simulcast / layered audio / redundancy (RED/FEC)** | None |
| **Jitter buffer / packet reordering / loss concealment** | None relay-side; `seq`/`ts_48k` are parsed for `tracing::trace!` only (`handler.rs:996-1003`). Playout is client-side (`desktop/src-tauri/src/huddle/playout.rs`) |
| **Server-side mute / kick / moderation** | No mute state, no kick message, no moderator role on `Room` (`room.rs:157-170`) |
| **Active-speaker / dominant-talker election** | Explicitly forbidden from consuming `level_dbov` (`wire.rs:16-20`); done client-side (`playout.rs:220`) |
| **Per-room bitrate/frame-rate limits or admission-time quality control** | Only a byte cap (4096/frame) and a queue depth (8) |
| **Push-to-talk, hand-raise, reactions in-huddle** | None |
| **Huddle metrics** | Zero `metrics::` calls in `audio/` — no gauge for active rooms/peers, no counter for dropped frames, no join-failure counter. The only metric in the whole group is `mesh_fence_rejections_total` (`tunnel/directory.rs:483`) |
| **Bandwidth accounting / per-peer rate limiting** | None |
| **A `speakers` control message** | Promised by the doc comment at `room.rs:33`; never produced |
| **`PeerCtrl::Close`** | Variant exists (`room.rs:35`) and is handled (`handler.rs:1112`) but has **zero producers** — the graceful per-peer shutdown path it implies is not wired |
| **Session resumption / reconnect continuity** | A rejoin is a fresh handshake with a fresh peer UUID and (usually) a different index |
| **Kind 48106 (huddle guidelines)** | Defined `kind.rs:538`, allowlisted for labelling `handlers/event.rs:49`, but never produced or consumed by the relay. Client-side only (`desktop/src-tauri/src/huddle/agents.rs:31`) |
| **Kind 48100 (huddle started)** | The relay only *reads* it, to validate a claimed parent (`handler.rs:1176-1186`). The producer is the desktop client (`desktop/src-tauri/src/huddle/mod.rs:252`) |

---

#### 3. What the tunnel lane delivers

| Capability | Where | Completeness |
|---|---|---|
| Redis fenced session directory (acquire / renew / release / lookup / validate) with a non-expiring generation counter | `tunnel/directory.rs:180-439` | Complete and well-tested (7 tests, 5 of which need live Redis) |
| Typed fence-rejection taxonomy shared with `/_mesh` and the huddle wire | `directory.rs:348-439`; `join.rs:982-1017` | Complete |
| Reliable-stream join routing (own locally vs. open a fenced bi-stream to the owner) | `reliable.rs:78-126` | Complete |
| Owner-side inbound accept with structural validation | `reliable.rs:135-172` | Complete |
| 1 MiB-chunked ordered framing with a community-bearing inner header | `reliable.rs:274-294`, `:412-492` | Complete |
| Per-frame Redis fence validation that **fails the session** on mismatch | `reliable.rs:326-389` | Complete |
| Observable lease renewer | `reliable.rs:571-657` | Implemented; **never spawned** |

##### 3.1 What the tunnel lane does **not** deliver

- **No product consumer.** `mesh_boot.rs:299-303` accepts an inbound reliable stream,
  logs `"reliable stream accepted; no session consumer wired — closing"`, and closes
  it. `mesh_boot.rs:206-215` states plainly that renewal attaches on the join path
  "which lands with the first product session consumer".
- **No join-side product caller.** The only route reaching
  `ReliableStreamRouter::join` is the demo-gated `POST /_mesh/demo/echo`
  (`api/mesh_demo.rs:73`, `:95`), documented as "Not a product flow"
  (`api/mesh_demo.rs:21-22`).
- **No renewal in the demo path either** — `api/mesh_demo.rs:11-15` documents that
  the `Owned` arm deliberately lets the lease die with its 30 s TTL, and instructs
  operators to drive the second pod inside that window.
- **No retransmission, ACK, flow control, or backpressure signalling** beyond what
  QUIC provides. There is no seq/ack in `ReliableWireFrame` (`reliable.rs:412-422`).
- **No goose/berd bridge.** The module doc says it is "for berd ↔ goose-server
  sessions" (`reliable.rs:1`) but no such wiring exists in the repo.
- `SessionDirectory::takeover` (`directory.rs:233`) and
  `ReliableMeshStream::with_community` (`reliable.rs:263`) have **zero callers**;
  `SessionDirectory::known_generation` (`directory.rs:324`) has test-only callers.

---

#### 4. What the conformance seam delivers

| Capability | Where | Completeness |
|---|---|---|
| `Tracer` trait + two impls (`NoopTracer`, `JsonlTracer`) | `conformance/tracers.rs:14-72` | `NoopTracer` in use; `JsonlTracer` **never instantiated anywhere** |
| Label projection (community, host, actor, msg-id, channel) | `conformance/mod.rs:47-91` | Complete, with one documented delta (§4.2) |
| Claimed-vs-resolved community separation | `conformance/mod.rs:94-117` | Complete |
| Write-path emits (`WriteInsert`/`WriteInsertGlobal`/`WriteDuplicate`/`SanitizedError`/`AuthCheck`) | consumed from `handlers/ingest.rs:2218`, `:2334-2348`, `:2468-2485` | Wired on the ingest path |
| Read-path emits (`ReadMessageRows`, `ReadByIdRows`) with row-community projection | `conformance/mod.rs:265-330`, called `req.rs:355`, `:671` | Wired |
| `ReadHostFeedRows` | schema `buzz-conformance/src/lib.rs:250` | **No emitter in the relay** — only the conformance crate's own proptest references it |
| Row-lookup coverage guard (`MissingLookup` → `ImplBug`) | `conformance/mod.rs:216-255` | Complete; 4 tests |
| `EmitGuard` RAII coverage-breach guard | `conformance/mod.rs:334-414` | Complete; 2 self-tests (`:454-528`) |
| `IngestError → SanitizedReason` closed-alphabet mapping | `conformance/mod.rs:422-429` | Complete, exhaustive match |

##### 4.1 Is it active in production? No.

`state.rs:794-798` binds `Arc::new(crate::conformance::NoopTracer)` unconditionally
at `AppState` construction. There is no config flag, env var, or feature gate that
swaps it. `NoopTracer::record` is `fn record(&self, _step: TraceStep) {}`
(`tracers.rs:22-24`). Consequences:

- Every emit site still **constructs** its `TraceAction` (allocating `String`s for
  labels, cloning `AbstractState`) and then discards it. The claimed "zero cost"
  (`tracers.rs:11-13`, `state.rs:615`) is not achieved — the work is done, only the
  write is elided. There is no `#[cfg(feature = ...)]` anywhere in the module.
- The `EmitGuard`'s `ImplBug` on a coverage breach is recorded into `NoopTracer` and
  therefore **silently discarded** in production. The guard is a test-time
  mechanism that is armed in production for no observable benefit.
- `JsonlTracer` — the only impl that would make the seam observable — has **zero
  callers in the workspace**. `state.rs:795-797` promises "Conformance tests
  overwrite this with a JsonlTracer after construction (see test helpers in
  `crates/buzz-test-client` once those land)". Those helpers have not landed: grep
  for `JsonlTracer` finds only the definition and two doc comments.
- `conformance/mod.rs:47-48` documents wire points in `req.rs`/`event.rs` as
  "held back as additive patch for Eva to apply onto Max's req.rs writes — see
  thread `c882c9b1…`". The `req.rs` sites **have** landed (`req.rs:144`, `:355`,
  `:671`); the doc comment is stale.

##### 4.2 Documented deltas in the conformance module

| Doc claim | Line | Actual code |
|---|---|---|
| "`actor` is the lower 16 bytes of `blake3(pubkey_bytes)` as a hex string" | `conformance/mod.rs:51-53` | `actor_label` takes the **first 16 hex chars of the pubkey hex** — no hashing (`:70-74`). The rationale at `:63-69` explains why this is acceptable, but the doc comment 10 lines earlier still says blake3 |
| "The trace carries opaque labels … never … wall-clock timestamps" | `conformance/mod.rs:12-15` | Honoured: `TraceStep` has no timestamp field (`buzz-conformance/src/lib.rs:290-299`). `bound_host` is a full host string, not opaque (`:58`), and `channel_label` is a direct UUID wrap — both deliberate (`:87-88`) |
| "the build can have the compiler eliminate them entirely behind a feature flag" | `tracers.rs:11-13` | No such feature flag exists in `crates/buzz-relay/Cargo.toml` (`[features] dev = ["buzz-auth/dev"]` only) |

---

#### 5. Mesh boot seam — what it delivers

| Capability | Where |
|---|---|
| Single construction point for all mesh machinery, `None` when off | `mesh_boot.rs:411-520` |
| iroh endpoint bind with a boot-unique keypair → boot-unique `RuntimeId` | `mesh_boot.rs:423-431` |
| Relay-key-attested ready record published to Redis + readiness-gated heartbeat | `mesh_boot.rs:445-472` |
| Advertise-address preference chain (`BUZZ_MESH_ADVERTISE_ADDR` → `POD_IP`+bound port → all endpoint IPs) | `mesh_boot.rs:379-402` |
| Static capability advertisement (all three profiles in one binary) | `mesh_boot.rs:369-377` |
| Per-profile inbound dispatcher over a single transport slot | `mesh_boot.rs:44-130` |
| Immediate reconcile pass so seed peers are dialed at boot | `mesh_boot.rs:478` |
| Drain watcher: gossip `draining=true` then drain owned huddles | `mesh_boot.rs:481-496` |
| Shared single `GenerationFloor` between the datagram hot path and huddle teardown | `mesh_boot.rs:149-166`, `:236-245`; pinned `mesh_boot.rs:665-737` |
| `BUZZ_MESH_DEMO_ECHO` reliable-stream echo consumer | `mesh_boot.rs:307-364` |

##### 5.1 Absent from the boot seam

- **No mesh readiness gate on the huddle path.** `boot_mesh` returns a handle as
  soon as the endpoint binds and the ready record publishes; there is no wait for
  peer connectivity. A join that resolves `RemoteOwner` before the owner peer is
  dialed fails with `huddle_owner_unreachable` (`handler.rs:487-503`).
- **`MeshHandle.membership` is written but never read.** Populated at
  `mesh_boot.rs:501`, exposed at `:141`, with zero consumers. `/_mesh` reads the
  private `runtime` field instead (`mesh_boot.rs:172-174`).
- **No mesh peer count / session gauge.** `/_mesh` returns whatever
  `MeshStatus` carries, unauthenticated (`router.rs:399-406`).


## Module: buzz-relay-mesh (`crates/buzz-relay-mesh`)
### Aspect: Features

Crate description: "Inter-relay QUIC mesh: transport, membership, and the fenced
wire contract" (`Cargo.toml:3`). 3,169 LOC across 10 files; 32 unit tests, all
passing, none `#[ignore]`d (verified by running `cargo test -p buzz-relay-mesh --lib`:
`32 passed; 0 failed; 0 ignored`).

---

#### 1. Opt-in status: **off by default, hard no-op when off**

`BUZZ_MESH` must be an explicit `on`/`true`/`1`; absent, `off`, or any typo means
disabled (`config.rs:498-500`). When disabled, `boot_mesh` returns `Ok(None)` before
touching anything (`mesh_boot.rs:417-419`) — no UDP bind, no Redis write, no spawned
task. `AppState.mesh` stays an empty `OnceLock` (`state.rs:627`), so
`AppState::mesh()` returns `None` (`state.rs:812`) and every consumer takes its
single-instance path.

Two tests pin this: `mesh_off_boots_nothing` points the Redis pool at an unroutable
`redis://127.0.0.1:1` so any accidental Redis touch would fail
(`mesh_boot.rs:526-541`), and `mesh_defaults_off_when_env_absent`
(`mesh_boot.rs:544-555`) asserts the fail-safe reading.

Nothing in `.env.example`, `deploy/`, or the helm chart sets `BUZZ_MESH` — verified
by grep. **No deployment in this repo turns the mesh on.**

---

#### 2. What ships and works today

| Capability | Where | Evidence |
|---|---|---|
| iroh/QUIC endpoint bind with boot-unique ed25519 identity | `endpoint.rs:19-44` | test `two_endpoints_connect_with_alpn_and_authenticated_identity` (`endpoint.rs:180-187`) |
| ALPN-pinned, mutually-authenticated peer connections, iroh relays disabled | `endpoint.rs:35`, `peer.rs:50-55` | same test |
| Length-delimited reliable bi-streams carrying `MeshStreamFrame` | `peer.rs:135-192` | `reliable_stream_roundtrip_carries_mesh_stream_frame` (`endpoint.rs:189-227`) |
| QUIC datagrams carrying `MeshDatagram`, size-checked pre-send | `peer.rs:105-129`, `lib.rs:213-226` | `datagram_roundtrip_carries_mesh_datagram` (`endpoint.rs:229-237`), `oversized_datagram_is_rejected_before_send` (`endpoint.rs:239-254`) |
| Opus-sized datagram loss/order gate (64 frames, loopback) | — | `opus_sized_datagrams_clear_empirical_local_loss_gate` (`endpoint.rs:256-291`) |
| Datagram header-overhead budget (≤64 B) | — | `datagram_header_overhead_within_budget` (`wire.rs:271-284`) |
| Redis ready registry: publish / clear / scan, TTL 3× refresh | `registry.rs:182-257` | `ready_key_is_stable_and_namespaced` (`registry.rs:317-322`), `expiry_is_three_refreshes` (`:324-327`), `ready_record_roundtrips_json` (`:336-346`) |
| Relay-key schnorr attestation of the boot-unique endpoint key | `registry.rs:29-96` | `ready_record_attestation_verifies_and_binds_runtime_pubkey` (`:348-356`), `attestation_rejects_signature_for_other_runtime` (`:358-364`) |
| Deployment-anchored seed admission (foreign relay identity rejected, unanchored = fail-closed) | `membership.rs:85-118` | 4 tests, `membership.rs:425-471` |
| Scuttlebutt digest/delta membership gossip on a per-connection control stream | `membership.rs:208-247`, `runtime.rs:501-551` | `digest_delta_only_sends_newer_records` (`gossip.rs:238-251`), `warm_pair_connects_and_gossips_membership` (`runtime.rs:721-...`) |
| Last-version-wins record merge | `membership.rs:120-153` | `stale_gossip_record_is_ignored` (`membership.rs:474-479`), `apply_delta_ignores_stale_versions` (`gossip.rs:253-266`) |
| Accept-loop admission gate with one registry rescan | `runtime.rs:262-323` | exercised indirectly by `connected_pair` (`runtime.rs:645-670`) |
| Warm full-mesh reconcile/dial loop | `runtime.rs:285-355` | `warm_pair_connects_and_gossips_membership` |
| Deterministic simultaneous-dial tie-break | `runtime.rs:204-213` | `tie_break_is_symmetric` (`runtime.rs:846-856`), `simultaneous_dial_converges_to_one_connection` (`runtime.rs:858-...`) |
| Inbound fan-out to a single `InboundHandler` slot | `runtime.rs:196-198`, `:358-372`, `:461-470` | `transport_datagram_reaches_inbound_handler` (`runtime.rs:745-...`), `transport_session_stream_reaches_inbound_handler` (`runtime.rs:775-...`) |
| Typed error for sending to an unconnected peer | `runtime.rs:167-175` | `send_to_unconnected_peer_is_typed_error` (`runtime.rs:823-843`) |
| Drain: gossip `draining=true`, stop being dialed | `membership.rs:385-388`, `runtime.rs:302` | `begin_drain_updates_local_record` (`membership.rs:482-489`) |
| Readiness-gated registry heartbeat with clear-on-not-ready | `registry.rs:295-312`, `runtime.rs:594-608` | `heartbeat_starts_unpublished` (`registry.rs:329-334`) |
| Per-peer counters surfaced at `/_mesh` | `membership.rs:249-283`, `status.rs` | `counters_are_reflected_in_status` (`membership.rs:481-...`) |
| Phi-accrual-style suspicion (exponential approximation) | `gossip.rs:168-220` | `phi_rises_as_heartbeats_age` (`gossip.rs:268-278`) |
| Wire-version and gossip-payload-version rejection | `wire.rs:182-188`, `gossip.rs:66-77` | `unknown_version_rejected` (`wire.rs:246-257`), `gossip_payload_roundtrips` (`gossip.rs:...`) |

---

#### 3. What is stubbed, inert, or dead

| Item | Status | Location |
|---|---|---|
| `gossip::GossipState` (+ 7 methods) | complete duplicate of `MeshMembership`'s scuttlebutt logic; **zero callers** outside its own tests | `gossip.rs:81-166` |
| `PeerInfo.load` | structurally always `0.0` — no writer anywhere | `lib.rs:137-138`, `gossip.rs:35`, `runtime.rs:566` |
| `MeshCounters.stale_generation_rejections` | always 0 in production; sole writer's only caller is a test | `membership.rs:285-293`, test `:486` |
| `mesh_fence_rejections_total{reason=…}` metric | documented, does not exist | `lib.rs:102-109` |
| `MeshError::Disabled`, `MeshError::PeerDraining` | dead variants — zero constructors repo-wide | `lib.rs:94`, `lib.rs:119` |
| `RuntimeAttestation::verify` | zero callers | `registry.rs:48-50` |
| `ReadyHeartbeat::shutdown` / `record` / `published` | zero production callers | `registry.rs:287-312` |
| `MeshRuntime::shutdown` | zero production callers — loops leak to process exit | `runtime.rs:155-164` |
| `MeshRuntime::connected_peers` | zero callers outside in-crate tests | `runtime.rs:138-146` |
| `MeshEndpoint::endpoint()` | zero callers anywhere | `endpoint.rs:51-53` |
| `MeshPeer::counters()` / `PeerCounters` | atomics incremented, never read | `peer.rs:10-15`, `:73-75` |
| `MeshMembership::with_phi_suspect_threshold` | zero callers — 8.0 is unoverridable | `membership.rs:66-69` |
| `RelayMeshMembership::peers()` / `local_runtime_id()` | zero production callers; `MeshHandle.membership` is written (`mesh_boot.rs:501`) and never read | `membership.rs:359-383` |
| `proto_version` / `capabilities` negotiation | advertised, never compared | `registry.rs:110`, `mesh_boot.rs:371-377` |
| Peer eviction | none — membership table only grows | `membership.rs` (no `remove`/`retain`) |
| Reconnect backoff | none | `runtime.rs:328-355` |
| `hmac`, `futures-util` deps | declared, **zero `use` sites** | `Cargo.toml:20,25` |
| `proptest` dev-dep | declared, zero property tests | `Cargo.toml:29` |
| Peer selection / gossip fan-out sampling | none — gossip is 1:N to all connected peers | `runtime.rs:571-585` |

Also inert at the consumer boundary: **no product session consumer is wired**.
`wire_mesh_consumers` accepts a reliable stream, logs
`"reliable stream accepted; no session consumer wired — closing"`
(`mesh_boot.rs:288-292`), unless `BUZZ_MESH_DEMO_ECHO` is on. And
`mesh_boot.rs:216-220` notes owner-side lease renewal "lands with the first product
session consumer" — i.e. not yet.

---

#### 4. Delivered end-to-end paths (via `buzz-relay`)

Three inbound profiles are wired (`mesh_boot.rs:229-299`):

1. **`RealtimeMedia` datagrams → huddle audio fan-in.** `MeshAudioRouter` over a
   shared `GenerationFloor` (`mesh_boot.rs:243-254`). This is the one path with a
   real product consumer (cross-pod huddle audio, `audio/mesh.rs`, `audio/join.rs`).
2. **`HuddleControl` streams → `HuddleControlAcceptor::accept_inbound`**
   (`mesh_boot.rs:256-278`) — owner-side peer register/unregister.
3. **`ReliableStream` streams → `ReliableStreamRouter::accept_inbound`**
   (`mesh_boot.rs:280-299`) — accepted and fence-validated, then closed (or echoed
   under the demo flag). Goose/berd tunnels are the intended consumer and are absent.

Operator surface: `GET /_mesh` on the health listener (`router.rs:230`, handler
`router.rs:396-406`) and the double-gated `POST /_mesh/demo/echo`
(`router.rs:123`, `api/mesh_demo.rs`).

---

#### 5. The five `mesh_*` examples — **they do not exercise this crate**

The brief describes `crates/buzz-relay/examples/mesh_*.rs` as exercising
`buzz-relay-mesh`. **That is not the case.** All five import `mesh_llm_sdk` /
`mesh_llm_host_runtime` — the external **MeshLLM** peer-to-peer LLM-serving project
(git dev-deps, `crates/buzz-relay/Cargo.toml:84-85`, tag `v0.73.1`). Verified: grep
for `buzz_relay_mesh` across `crates/buzz-relay/examples/` returns **zero matches**.

Two unrelated things named "mesh" live in one crate, both spelled `mesh_*`:

| Example | LOC | What it actually proves | Subject |
|---|---|---|---|
| `mesh_admission_smoke.rs` | 452 | MeshLLM **owner-allowlist** admission: 3 subprocesses (serve + trusted client + non-member); possession of an invite token admits nobody, only allowlisted `OwnerKeypair` ids route inference (`mesh_admission_smoke.rs:1-30`, verdict matching `:262-283`) | mesh-llm |
| `mesh_agent_e2e.rs` | 421 | share-compute → agent-env preset → real `buzz-agent` over ACP stdio → inference; 4 permutations incl. a context-fit rejection (P3, `:106-134`) and `buzz-dev-mcp` tool use (P4, `:136-166`) | mesh-llm |
| `mesh_serve_client_smoke.rs` | 171 | MeshLLM serve node + client node joined by invite token, one completion routed client→serve (`:1-24`) | mesh-llm |
| `mesh_stack_smoke.rs` | 121 | tokio worker **stack-size** repro/fix: 2 MiB dies on the stack guard, 8 MiB completes a model download (`:1-30`); mirrors `buzz_lib::mesh_llm::MESH_WORKER_STACK_SIZE` (`:35-37`) | mesh-llm |
| `mesh_serve_smoke.rs` | 96 | single-node MeshLLM serve-and-self-consume; documents that `console_ui(true)` is load-bearing for `serve::start` readiness polling at mesh rev `bd16da4` (`:22-31`) | mesh-llm |

They are all explicitly **hardware-gated and not CI** — stated in each header
(`mesh_admission_smoke.rs:26-27`, `mesh_agent_e2e.rs:22-24`,
`mesh_serve_client_smoke.rs:25-26`, `mesh_stack_smoke.rs:26-27`) and reachable only
via `just mesh-e2e-hardware` (`justfile:327-331`), `just mesh-e2e-admission`
(`justfile:335-339`), `just mesh-e2e-confidence` (`justfile:342-349`).

##### CI-gate check

**No CI workflow gates any of the five examples, and none gates this crate's
tests.** Verified:

- Every `mesh` hit in `.github/workflows/{ci,release,signed-macos-canary,windows-canary}.yml`
  is MeshLLM build plumbing (resolving the `mesh-llm-sdk` rev from `Cargo.lock`,
  caching llama native libs, `--features mesh-llm` desktop builds) — e.g.
  `ci.yml:1015-1052`, `release.yml:141-181`.
- CI's Rust lint step is `just clippy` (`ci.yml:94`) = `cargo clippy --workspace
  --all-targets -- -D warnings` (`justfile:106-107`). `--all-targets` **compiles**
  the five examples (and therefore forces the git dev-deps to build) but runs
  nothing.
- CI's unit step is `just test-unit` (`ci.yml:116`), which runs only
  `-p buzz-core -p buzz-auth --lib`, `-p buzz-db --lib`, `-p buzz-conformance`,
  `-p buzz-push-gateway` (`justfile:275-295`). **`buzz-relay-mesh` is not in that
  list.**
- The backend-integration job archives `-p buzz-relay -p buzz-test-client --lib`
  (`ci.yml:336-342`) — also not this crate.

**Net: all 32 `buzz-relay-mesh` unit tests compile in CI and never execute.** They
pass locally (verified), including the four that stand up real loopback iroh
endpoints and the two multi-runtime gossip/tie-break tests.

---

#### 6. Test inventory (32 total, 0 `#[ignore]`d)

| File | Tests | Kind |
|---|---|---|
| `endpoint.rs` | 5 | `#[tokio::test]`, real loopback iroh endpoints |
| `membership.rs` | 7 | pure `#[test]`, real `nostr::Keys::generate()` signing |
| `registry.rs` | 6 | pure `#[test]`; `heartbeat_starts_unpublished` builds a pool but never dials |
| `runtime.rs` | 6 | 5 `#[tokio::test]` (2-runtime loopback meshes) + 1 pure |
| `gossip.rs` | 4 | pure |
| `wire.rs` | 4 | pure roundtrip/negative/budget |
| `lib.rs`, `peer.rs`, `status.rs` | 0 | — |

Untested behaviour of note: no test covers `is_known_peer`'s registry-rescan branch
(`runtime.rs:309-320`), the `remove_peer` path (`runtime.rs:267-281`), `dial_peer`'s
bad-addr/bad-id branches (`runtime.rs:328-348`), `ReadyRegistry::publish_ready`/
`clear_ready`/`scan_ready` against a live Redis, or `spawn_registry_heartbeat`
(`runtime.rs:594-608`). All Redis paths are effectively untested.


## Module: buzz-conformance (`crates/buzz-conformance`)
### Aspect: Features

Five capabilities, split cleanly by whether the relay actually drives them.

| # | Feature | Where it lives | Wired into the relay? |
|---|---|---|---|
| F1 | Trace emission (the seam) | `crates/buzz-relay/src/conformance/mod.rs`, `handlers/ingest.rs`, `handlers/req.rs` | **Yes** — but always into `NoopTracer` (`crates/buzz-relay/src/state.rs:798`) |
| F2 | Coverage-breach guard (`EmitGuard`) | `crates/buzz-relay/src/conformance/mod.rs:344-415` | **Yes** — armed at one seam only (`handlers/ingest.rs:1408-1412`) |
| F3 | Row-community projection | `crates/buzz-relay/src/conformance/mod.rs:186-333` | **Yes** — both read lanes in `req.rs` |
| F4 | Replay checking (`check_trace`) | `crates/buzz-conformance/src/checker.rs:74` | **No** — zero production callers |
| F5 | Test lanes: property tests + golden fixtures | `crates/buzz-conformance/tests/` | Test-only by construction |

---

### F1 — Trace emission

Nine actions in the vocabulary (`src/lib.rs:179-261`); the relay emits **six** of them.

| Action | Relay emit site(s) | Producer |
|---|---|---|
| `SanitizedError` | `handlers/ingest.rs:1436-1443` | terminal-error wrapper, one place for all `Err` returns |
| `AuthCheck` (write path) | `handlers/ingest.rs:1794-1802` | after `check_channel_membership` (`:1785-1787`) |
| `AuthCheck` (read path) | `handlers/req.rs:143-151` → `conformance/mod.rs:148-168` | after `db.is_member` (`req.rs:137-142`), only on the cache-miss branch (`:134-136`) |
| `WriteInsertGlobal` | `handlers/ingest.rs:139-146` (product feedback, called at `:1573`), `:2215-2222` (push lease), `:2506-2509` (trailing dispatch, channel-less arm) | |
| `WriteInsert` | `handlers/ingest.rs:2362-2366` (reaction lane), `:2470-2474` (trailing dispatch) | |
| `WriteDuplicate` | `handlers/ingest.rs:2368-2372` (reaction lane), `:2501-2505` (trailing dispatch) | |
| `ReadMessageRows` | `handlers/req.rs:355-361` → `conformance/mod.rs:265-298` | non-search REQ lane, per filter |
| `ReadByIdRows` | `handlers/req.rs:671-677` → `conformance/mod.rs:300-333` | NIP-50 search refetch lane |
| `ImplBug` | `conformance/mod.rs:404-414` (guard drop), `:284-296` / `:319-331` (projection miss) | |
| `ReadHostFeedRows` | **none** — grep across `crates/buzz-relay/` finds no producer | |

Emission is per-decision, synchronous, and untimed. There is no batching, no sampling, no
async queue; `emit` is `tracer.record(TraceStep::new(action, state))`
(`conformance/mod.rs:127-129`).

**One emit arm is unreachable.** `handlers/ingest.rs:2452-2458` returns early on
`!was_inserted`, so the trailing dispatch's `(Some(ch), false) => WriteDuplicate` arm
(`:2501-2505`) can never execute. Only the reaction lane's `WriteDuplicate` (`:2368`) is
reachable, and only when `buzz_db::ReactionEventInsertOutcome::Inserted` carries
`was_inserted: false` (`:2348-2351`) — the explicit `Duplicate` outcome returns earlier at
`:2341-2347` without emitting.

**Fan-out is deliberately not a feature.** `handlers/event.rs:405-410` documents the decision:
the spec has no fan-out action, and acceptance is already recorded at the ingest seam.

---

### F2 — Coverage-breach guard

`EmitGuard::arm` (`conformance/mod.rs:383-400`) returns a pair: the guard, and a
`CountingTracer`-wrapped `Arc<dyn Tracer>` the caller must use in place of the original. The
wrapper bumps a `Relaxed` `AtomicUsize` on each `record` (`:367-373`); `Drop` emits
`ImplBug { kind }` on the **inner** tracer when the count is still zero (`:403-415`).

Callers never disarm — the design note at `:337-343` is explicit that production paths just
call `record` as before.

**Armed at exactly one seam:** `ingest_event` (`handlers/ingest.rs:1408-1412`, `kind =
"ingest_event_exited_without_trace"`). The REQ path arms nothing; `handle_req` builds a
`trace_state` (`handlers/req.rs:116-118`) but no guard, so a REQ that returns before any read
emits produces no `ImplBug`. Six paths through `ingest_event_inner` return `Ok` without any
emit, so the guard fires `ImplBug` on each:

| Path | Line | Kind(s) |
|---|---|---|
| Command routing | `ingest.rs:1560-1562` | 7 kinds: 30620, 41010, 41011, 41012, 46020, 46030, 46031 (`buzz-core/src/kind.rs:743-754`) |
| NIP-56 report | `ingest.rs:1587-1596` | 1984 (`buzz-core/src/kind.rs:267`) |
| Moderation commands | `ingest.rs:1605-1614` | 5 kinds (`buzz-core/src/kind.rs:316-325`) |
| Relay-admin kinds | `ingest.rs:1834-1842` | 4 kinds (`buzz-core/src/kind.rs:723-731`) |
| NIP-43 leave request | `ingest.rs:1846-1928` | 28936 (`buzz-core/src/kind.rs:344`) |
| Reaction duplicate | `ingest.rs:2341-2347` | 7 (`buzz-core/src/kind.rs:58`) |
| Non-reaction duplicate | `ingest.rs:2452-2458` | all stored kinds |

That is seven, not six. Each returns `accepted: true` or `accepted: false` with no trace step,
so the guard's `ImplBug` is the *only* record for the request — which the checker classifies as
`CoverageBreach` (`src/transitions.rs:277-279`). The guard therefore works as designed; the
gap is that a legitimate accepted write is indistinguishable from a forgotten emit site.

---

### F3 — Row-community projection ("(B) strategy")

The read seam's non-tautology mechanism, documented at `conformance/mod.rs:170-185`:

- Channel-less rows (`row.channel_id == None`) → project as `resolved_community`
  (`:191`).
- Channel-scoped rows → look up the row's **own** `channel_id` in a map from
  `buzz_db::Buzz::communities_of_channels` (`:192-194`). A hit yields the looked-up label; a
  miss yields `None`.
- `project_row_communities` (`:234-262`) turns the first `None` into
  `RowCommunityProjection::MissingLookup { kind: "row_community_lookup_missing",
  first_missing_channel }` (`:246-253`), which the record helpers convert into `ImplBug`
  (`:284-296`, `:319-331`).

The key property is that the label comes from the row, not from the query's `WHERE` clause —
so a channel-scoped row cannot masquerade as channel-less to dodge the lookup
(`:182-185`). Two unit tests pin it: `project_row_communities_channel_scoped_uses_lookup_label`
asserts the foreign label survives and is *not* replaced by resolved
(`conformance/mod.rs:621-648`), and `project_row_communities_channel_scoped_missing_is_breach`
asserts a miss is a breach, not a substitution (`:650-673`).

**Cost:** the projection requires an extra `db.communities_of_channels` round-trip per REQ
filter (`handlers/req.rs:345`) and per search page (`:661`), gated on
`trace_state.is_some()` (`:334`, `:649`) — a condition independent of which tracer is bound.
See the debt doc.

---

### F4 — Replay checking

`check_trace` (`src/checker.rs:74-131`) walks a `Scenario` and returns the first
`TransitionError`. Stages, per the doc at `:63-73`: empty guard → schema check → bootstrap →
per-step `check_step` → coverage-set diff.

**No production wiring exists.** Repo-wide grep for `check_trace` returns hits only in the
crate's own source, its two test files, and its two markdown docs. Nothing constructs a
`Scenario` from a live relay, and `JsonlTracer` — the only tracer that persists anything —
has zero instantiation sites outside its own definition (`conformance/tracers.rs:30-45`;
grep for `JsonlTracer` returns only the definition, the re-export at `mod.rs:46`, two doc
comments at `state.rs:616`/`:795`, and two markdown mentions). So the end-to-end pipeline the
crate is built for (relay → JSONL → checker) is not assembled anywhere in the repo.
`crates/buzz-conformance/LIMITS.md:120-125` describes it as the "next ratchet".

---

### F5 — Test lanes

22 tests, all passing (`cargo test -p buzz-conformance`: 9 + 7 + 6).

**Unit lane — 9 tests in `src/checker.rs::tests` (`:135-337`).** One passing case
(`write_insert_then_read_with_only_resolved_rows_passes`, `:172-207`) plus one bite case per
failure mode. Helpers `cid`/`ch`/`state`/`step` at `:144-162`.

**Property lane — 7 proptest cases, 128 cases each
(`ProptestConfig::with_cases(128)`, `tests/proptest_checker.rs:193`).**

| ID | Test | Line | Asserts |
|---|---|---|---|
| P2 | `clean_trace_is_accepted` | `:199` | no false rejects on 1..=12 clean actions |
| P1 | `foreign_row_label_is_rejected` | `:213` | `NonInterference` across all three read variants |
| P3a | `auth_allow_foreign_claim_bites` | `:272` | `IllegalTransition` |
| P3b | `auth_deny_any_claim_is_ok` | `:304` | `Deny` never bites on the claim axis |
| P4 | `impl_bug_bites_coverage_breach` | `:328` | `CoverageBreach` |
| P5 | `state_flip_bites_state_mismatch` | `:354` | `StateMismatch` on any one flipped field |
| P6 | `check_trace_is_deterministic_and_total` | `:406` | two runs stringify identically; no panic |

The lane explicitly refuses a parallel oracle — rationale at `:9-25`: a reference
implementation "would just be a copy of the checker". Community/channel pools are 3 wide
(`POOL`, `:51`) so foreign-vs-resolved collisions are frequent (`:44-49`).

**Fixture lane — 6 tests, 4 committed JSONL files.** `assert_fixture_matches`
(`tests/replay_fixtures.rs:206-235`) serializes the typed builder, byte-compares against the
committed file, then re-parses and compares structurally (`:233-234`).
`BUZZ_CONFORMANCE_UPDATE=1` regenerates instead of comparing (`:210-214`).

| Fixture | Builder | Test | Verdict |
|---|---|---|---|
| `good.jsonl` | `:75-101` | `good_trace_passes_check` `:238-250` | `Ok(())`, with 3 required actions (`:241-244`) |
| `bad_host_channel_mismatch.jsonl` | `:107-129` | `:253-264` | `IllegalTransition` |
| `bad_coverage_breach.jsonl` | `:134-141` | `:267-278` | `CoverageBreach` |
| `bad_foreign_row_leak.jsonl` | `:154-168` | `:281-292` | `NonInterference` |

Two fixture-free tests round out the lane: `empty_trace_is_coverage_breach` (`:295-304`) and
`missing_required_action_is_coverage_breach` (`:307-320`).

**Relay-side lane — 9 tests in `crates/buzz-relay/src/conformance/mod.rs::tests`
(`:431-726`)**, covering the guard (`:467-495`, `:498-527`), the REQ AuthCheck verdict table
(`:530-570`, `:572-593`), and the four projection cases (`:596-618`, `:621-648`, `:650-673`,
`:675-696`, `:698-725`). Plus one in `handlers/ingest.rs:2530-2565`
(`feedback_success_action_satisfies_ingest_emit_guard`) proving the feedback path satisfies
the guard.

These relay-side tests are **not** in the unit gate: `just test-unit` runs `-p buzz-core
-p buzz-auth --lib`, `-p buzz-db --lib`, `-p buzz-conformance`, `-p buzz-push-gateway`
(`justfile:278-293`) and the shell fallback matches (`scripts/run-tests.sh:81-102`). Neither
list includes `buzz-relay`, so the emitter-side proofs run only via
`run_integration_tests`' catch-all `cargo test --test '*'` (`scripts/run-tests.sh:118-120`),
which matches integration targets, not `--lib` unit tests. `crates/buzz-conformance/LIMITS.md:109`
instructs `cargo test -p buzz-relay --lib conformance::` as a required CI surface; no justfile
recipe or workflow runs it.

