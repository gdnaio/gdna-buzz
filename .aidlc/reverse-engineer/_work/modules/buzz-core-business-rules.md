## Module: buzz-core (`crates/buzz-core`)

### Aspect: Business Rules

Every rule below is read from the implementation. "Trigger" = the call path that evaluates the rule.

---

### 1. Event verification (`src/verification.rs`)

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| BR-1 | Event ID must equal the canonical NIP-01 hash of `(pubkey, created_at, kind, tags, content)`. On mismatch the recomputed ID and the claimed ID are both reported. | `verification.rs:12-25` | `verify_event()` — called by relay ingest (doc says call via `spawn_blocking`, `verification.rs:1-2`, `:9`) |
| BR-2 | Schnorr signature over the event ID must validate; checked **only after** the ID check passes. | `verification.rs:27-29` | same |
| BR-3 | Verification order is ID-then-signature, never the reverse. | sequential structure `verification.rs:12-31` | same |

Test evidence: `rejects_tampered_id` (`verification.rs:44-58`) and `rejects_tampered_signature` (`:60-68`); `event.rs:57-66` additionally asserts that tampering only `sig` leaves `verify_id()` true while `verify_signature()` is false.

---

### 2. NIP-01 filter matching (`src/filter.rs`)

Top-level semantics: multiple filters are OR-ed (`filters_match` uses `.any()`, `filter.rs:10-13`); fields inside one filter are AND-ed (sequential early-`return false` in `filter_match_one`, `filter.rs:35-104`).

| # | Rule | Enforced at | Notes / edge cases |
|---|------|-------------|--------------------|
| BR-4 | `kinds`: **absent** (`None`) → no kind constraint; **present** → event kind must be a member. | `filter.rs:36-40` | An explicitly **empty** `kinds` vector, if constructible, would make `contains` false and reject everything. The code does not special-case empty-vs-absent; only `Option` presence is checked. |
| BR-5 | `authors`: absent → unconstrained; present → `event.pubkey` must be a member. | `filter.rs:42-46` | Exact match on `PublicKey`; **no prefix matching for authors** (contrast BR-7). |
| BR-6 | `since`: reject when `event.created_at < since` (i.e. `since` is inclusive). | `filter.rs:48-52` | |
| BR-7 | `until`: reject when `event.created_at > until` (inclusive). | `filter.rs:54-58` | |
| BR-8 | `ids`: **prefix matching** — event id hex must `starts_with` at least one filter id hex. | `filter.rs:60-66` | Comment cites NIP-01 prefix allowance (`filter.rs:60`). Since `EventId::to_hex()` is always 64 chars, the practical effect is full-equality unless a shorter id type is supplied. |
| BR-9 | Generic tag filters (`#e`, `#p`, `#h`, …): for each tag key in the filter, at least one of the filter's values must equal the content of at least one event tag of that key. Different tag keys are AND-ed; values within one key are OR-ed. | `filter.rs:68-77` | Matching compares `tag.kind().to_string()` against the filter key string and `tag.content()` against the value. |
| BR-10 | `#h` fallback: if no h-tag matched **and** the event carries **no** h-tags at all, fall back to `StoredEvent.channel_id`; the filter's `#h` values must contain that UUID string, otherwise reject. If `channel_id` is `None`, reject. | `filter.rs:78-93` | Rationale in code: reactions (kind 7) and deletions (kind 5) derive channel from their target and carry no h-tag (`filter.rs:78-82`). |
| BR-11 | `#h` strictness: if the event **does** have h-tags but none matched, reject — the tag is authoritative and `channel_id` must not override it. | `filter.rs:94-100` | Test `h_tag_fallback_uses_stored_channel_id` (`filter.rs:188`) asserts this anti-leak property at `filter.rs:229-235`. |
| BR-12 | Empty filter list matches nothing (`[].iter().any(..) == false`). | `filter.rs:10-13` | Asserted in `or_semantics` at `filter.rs:173`. |
| BR-13 | A filter with no constraints at all matches every event (all `Option`s `None`, no generic tags → falls through to `true`). | `filter.rs:103` | Implicit; no test pins this. |

#### Result-level read gate

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| BR-14 | For `KIND_DM_VISIBILITY` (30622) and `KIND_AGENT_TURN_METRIC` (44200), the reader's pubkey hex must appear as the content of a `p` tag on the event; all other kinds are unconditionally authorized. | `filter.rs:23-33` | `reader_authorized_for_event()` — doc lists the three call sites it guards (WS historical pull, HTTP bridge, live fan-out) at `filter.rs:19-22` |
| BR-15 | The authoring agent of a turn metric is **not** authorized to read it back (only the `p`-tagged owner is). | consequence of `filter.rs:28-32`; asserted at `filter.rs:294-298` | same |

Note: the gated kind set in `reader_authorized_for_event` is hard-coded as two `!=` comparisons (`filter.rs:25`), duplicating the membership of `kind::RESULT_GATED_KINDS` (`kind.rs:129`) rather than reading it.

---

### 3. Kind classification rules (`src/kind.rs`)

| # | Rule | Enforced at |
|---|------|-------------|
| BR-16 | Ephemeral = `20000 ≤ kind ≤ 29999`; documented as never stored. | `kind.rs:621-623`, range constants `:321-323` |
| BR-17 | Replaceable = kinds `0`, `3`, `41`, or `10000..=19999` (NIP-01). Parameterized-replaceable kinds are explicitly *not* in this set. | `kind.rs:628-630` and doc `:625-627` |
| BR-18 | Parameterized replaceable = `30000 ≤ kind ≤ 39999`, keyed by `(pubkey, kind, d_tag)`, latest `created_at` wins. | `kind.rs:635-637` and doc `:632-634` |
| BR-19 | Replaceable and parameterized-replaceable sets must be disjoint over the whole u16 space. | test `replaceable_and_parameterized_are_disjoint` iterates `0..=65535` (`kind.rs:776-784`) |
| BR-20 | Workflow execution kinds `46001..=46012` must not trigger workflows (loop prevention, per doc). | `kind.rs:641-643`, doc `:639-640` |
| BR-21 | Relay-admin command kinds are exactly 9030–9033. | `kind.rs:647-656` |
| BR-22 | Identity-archive **requests** are only 9035/9036; relay-signed 8002/8003/13535 are deliberately excluded from the request classifier. | `kind.rs:662-664`, doc `:658-661` |
| BR-23 | Community moderation commands are exactly 9040–9044; `KIND_REPORT` (1984) is **not** a command. | `kind.rs:240-249`; compile-time assertions `:742-744` |
| BR-24 | Transactional "command kinds" requiring atomic execution: 30620, 41010, 41011, 41012, 46020, 46030, 46031. | `kind.rs:667-679` |
| BR-25 | Relay-only kinds (client submission must be rejected): 13534, 40901, 40902, 30622, 39005, 39006. | `kind.rs:682-693`; test `nip43_membership_snapshot_is_relay_only` (`kind.rs:760-763`) |
| BR-26 | No two entries in `ALL_KINDS` may share an integer value. | test `no_duplicate_kind_values` (`kind.rs:752-758`) — note: covers only the 127 listed kinds, not the 3 excluded ones |
| BR-27 | Compile-time invariants: new addressable kinds must be in the 30000–39999 range; all kind constants must fit `u16`; `KIND_AGENT_TURN_METRIC` must be neither ephemeral, replaceable, nor parameterized. | `const _: () = assert!(...)` block, `kind.rs:707-744` |
| BR-28 | Author-only read kinds: `KIND_EVENT_REMINDER` (30300) and `KIND_PUSH_LEASE` (30350) — relay must not reveal existence, count, tags, content, schedule, or search matches to non-authors (doc, enforced elsewhere). | `kind.rs:112-120` |
| BR-29 | `#p`-bound read kinds (`P_GATED_KINDS`): 24200, 44100, 44101, 1059, 30622, 44200 — a REQ that can match any of these is closed unless the filter's `#p` equals the reader's pubkey; stored kinds in the set also get a NULL `search_tsv`. | `kind.rs:122-156` (declaration + contract doc) |
| BR-30 | Result-gated kinds (30622, 44200) force the per-event fallback path in COUNT instead of fast SQL counting. | `kind.rs:122-129` |

---

### 4. Channel + role rules (`src/channel.rs`)

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| BR-31 | Canonical channel name strips **all** leading `#` characters and leading/trailing whitespace (interleaved), then trims the end. | `channel.rs:15-19` | `canonical_channel_name()`; test `channel_names_trim_whitespace_and_drop_all_leading_hashes` (`channel.rs:186-197`) pins `"### ###" → ""` and `"channel#topic"` unchanged |
| BR-32 | Role hierarchy is numeric: Owner 4 > Admin 3 > Member 2 > Guest 1 > Bot 0. | `channel.rs:142-150` | `permission_level()` |
| BR-33 | Bot never satisfies any role requirement at this layer (level 0 < every other level). | consequence of `channel.rs:142-157` | `has_at_least()`; the `git_perms` test `evaluate_bot_cannot_push_without_explicit_grant` documents that the policy layer promotes Bot→Member before evaluation (`git_perms.rs:910-924`) |
| BR-34 | Elevated roles (only grantable by owners/admins per doc) are Owner and Admin. | `channel.rs:134-136` | `is_elevated()` |
| BR-35 | `FromStr` for all three channel enums is strict: unknown strings error with a formatted message; there is no default variant. | `channel.rs:44-53`, `:88-99`, `:163-179` | parsing DB/Nostr tag values |

---

### 5. Presence rules (`src/presence.rs`)

| # | Rule | Enforced at |
|---|------|-------------|
| BR-36 | Structured (REST/MCP) presence accepts exactly `online`, `away`, `offline`; unknown values fail deserialization. | serde derive with `rename_all = "lowercase"` (`presence.rs:10-18`); test `serde_rejects_unknown_variant` (`presence.rs:55-59`) |
| BR-37 | `Offline` clears the presence entry (documented semantic). | doc comment `presence.rs:17` |
| BR-38 | `as_str()`, `Display`, and the serde form must agree. | `presence.rs:22-35`; tests `as_str_matches_serde` (`presence.rs:61-66`), `display_matches_as_str` (`presence.rs:68-73`) |

---

### 6. Tenant resolution rules (`src/tenant.rs`)

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| BR-39 | A request's community is resolved by the server from the connection host — never supplied or influenced by the client. Expressed in types by having no `Default`, no `Deserialize`, and no parse-from-input constructor. | `tenant.rs:10-30` (invariant doc), `:43-44` (no serde derives), `:74-78` | construction of `TenantContext` |
| BR-40 | Host normalization: trim whitespace → ASCII-lowercase → strip a `:443` or `:80` suffix → strip one trailing `.`. Non-default ports are preserved as distinct selectors. | `tenant.rs:121-139` | `normalize_host()` on both the inbound `Host` header and the stored `communities.host` |
| BR-41 | Order matters: the default-port strip happens **before** the trailing-dot strip, so `"Relay.Example.:443"` normalizes to `"relay.example"`. | `tenant.rs:128-138`; test `normalize_host_collapses_tenant_split_variants` (`tenant.rs:196-213`) | |
| BR-42 | Empty/whitespace-only host normalizes to `""`; resolution must fail closed rather than defaulting to a tenant. | `tenant.rs:117-119` (doc), test `normalize_host_empty_stays_empty` (`tenant.rs:229-234`) | |
| BR-43 | IPv6 literals are left intact (`[::1]` keeps brackets) because only an exact `:80`/`:443` suffix is stripped. | `tenant.rs:126-134`; test `normalize_host_leaves_ipv6_literal_intact` (`tenant.rs:222-227`) | |
| BR-44 | `relay_url_authority` must produce byte-identical output to `normalize_host` for the same deployment: host + explicit non-default port, IPv6 in brackets, empty string when unparseable (caller fails closed). | `tenant.rs:156-172`; tests `tenant.rs:236-274` | startup community seeding, `bind_deployment_community`, `buzz-admin` tenant resolution (per doc `:146-151`) |

---

### 7. Relay URL identity rules (`src/relay.rs`)

| # | Rule | Enforced at |
|---|------|-------------|
| BR-45 | Scheme must be `ws` or `wss`; anything else → `InvalidScheme`. | `relay.rs:40-42` |
| BR-46 | URLs carrying a username or password are rejected (`Credentials`). | `relay.rs:43-45` |
| BR-47 | URLs carrying a fragment are rejected (`Fragment`). | `relay.rs:46-48` |
| BR-48 | A host is required (`MissingHost`). | `relay.rs:50` |
| BR-49 | All loopback spellings (`localhost` case-insensitive, loopback IPv4, loopback IPv6) fold to `127.0.0.1`; other DNS hosts are ASCII-lowercased. | `relay.rs:51-64` |
| BR-50 | Default ports are dropped (`ws`→80, `wss`→443); a root path `/` is removed; trailing `/` is trimmed from the final string. Non-root paths and queries are preserved. | `relay.rs:66-77` |
| BR-51 | This normalizer is explicitly **not** the NIP-42 AUTH comparison helper; AUTH equivalence must stay narrower. | doc `relay.rs:28-32` |

---

### 8. NIP-AE engram rules (`src/engram.rs`)

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| BR-52 | Slug grammar: either the reserved `"core"`, or `mem/` + one-or-more `/`-separated segments; each segment 1–64 bytes, first byte `[a-z0-9]`, remaining bytes `[a-z0-9_-]`; whole slug ≤ 255 bytes. | `engram.rs:67-92` + `validate_segment` `:94-112` | `validate_slug()` from parse, build, extract-refs paths |
| BR-53 | Shorthand normalization: `core` passes through; anything not already prefixed gets `mem/` prepended; the result is then validated (so `"Foo"` still fails). | `engram.rs:123-131`; test `normalize_slug_adds_mem_prefix` (`engram.rs:773-778`) | `normalize_slug()` |
| BR-54 | Conversation key `K_c` is NIP-44 v2 ECDH-derived and symmetric: `derive(sk_a, pk_o) == derive(sk_o, pk_a)`. | `engram.rs:136-138`; test `conversation_key_matches_spec` pins both directions to one hex vector (`engram.rs:636-644`) | |
| BR-55 | `d` tag = lowercase hex of `HMAC-SHA256(K_c, "agent-memory/v1/d-tag" ‖ 0x00 ‖ slug)`. | `engram.rs:144-155` | `d_tag()` |
| BR-56 | Body JSON is byte-exact: slug first, whitespace-free, `{"slug":…,"value":…}` or `{"slug":"core","profile":…}`; `null` value = tombstone. | `engram.rs:189-217`; test `body_round_trips_byte_exact` (`engram.rs:656-693`) | `to_json_bytes()` |
| BR-57 | Body parsing rejects duplicate object member names **at any nesting depth**; repeats inside arrays are allowed. | `parse_strict_json` `engram.rs:283-380` (`visit_map` dup check `:353-370`); tests `body_parse_rejects_duplicate_keys` (`engram.rs:695`), `body_parse_accepts_arrays_of_repeated_strings` (`engram.rs:710`), `body_parse_rejects_duplicates_at_any_depth` (`engram.rs:739`) | `Body::from_json_bytes()` |
| BR-58 | Trailing data after the JSON document is rejected. | `engram.rs:377-378` | same |
| BR-59 | Body shape: top level must be an object; `slug` must be a present string and pass BR-52; `core` requires a string `profile`; non-core requires `value` as string or null; unknown fields are ignored. | `engram.rs:228-258`; test `body_parse_ignores_unknown_fields` (`engram.rs:703-708`) | same |
| BR-60 | Reference extraction: only literal `[[slug]]` substrings count; the first `]]` closes, no nesting; an inner `[[` restarts the scan; candidates failing `validate_slug` are silently dropped; results are first-occurrence-ordered and deduplicated. | `engram.rs:384-430` | `extract_refs()`; 15 tests spanning `engram.rs:891-1047` |
| BR-61 | Serialized body must not exceed 65,535 bytes (NIP-44 plaintext cap) — rejected at build time with `BodyTooLarge`. | `engram.rs:436-440`; test `body_too_large_rejected_at_build_time` (`engram.rs:877-889`) | `build_event()` |
| BR-62 | Built engram events carry exactly two tags — `["d", <hex>]` and `["p", <owner hex>]` — kind 30174, with a caller-supplied `created_at`. | `engram.rs:456-476` | `build_event()` |
| BR-63 | Envelope validation (head-selection rules 1 and 5): kind must be 30174; `event.pubkey` must equal the expected agent; exactly one `d` tag; `d` must be 64 lowercase hex chars; exactly one `p` tag; `p` must equal the expected owner (case-insensitive compare). | `engram.rs:489-536` | `validate_and_decrypt()` |
| BR-64 | After decryption, the body's slug must re-derive to the event's `d` tag (rule 4), else `InvalidEnvelope`. | `engram.rs:545-557` | same |
| BR-65 | Signature verification is the **caller's** responsibility before calling `validate_and_decrypt` (NIP-44 requires outer-signature-before-decrypt). | doc `engram.rs:478-482` | contract note, not enforced in this function |
| BR-66 | Head selection (LWW): greatest `created_at` wins; ties broken by **lowest** event id hex. | `engram.rs:564-583` | `select_head()` |
| BR-67 | Write monotonicity: `created_at = max(now, prior_head + 1)` with saturating add. | `engram.rs:588-593`; test `monotonic_clock_rule` (`engram.rs:781-786`) | `monotonic_created_at()` |

---

### 9. NIP-AM turn metric rules (`src/agent_turn_metric.rs`)

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| BR-68 | `cost_usd` (in `turn` and in `cumulative`) must be finite and ≥ 0 when present; NaN, ±inf, and negatives are rejected with `InvalidPayload`. | `agent_turn_metric.rs:147-169` | `validate()` |
| BR-69 | Token counts cannot be negative by construction (`Option<u64>`); `None` means "not reported", never zero. | type choice `:24-33` + doc `:18-20` | serde round-trip |
| BR-70 | `total_tokens` is provider-reported and must not be derived by summing input+output. | doc `:31-32` | consumer contract |
| BR-71 | Validation is symmetric: both `encrypt_agent_turn_metric` and `decrypt_agent_turn_metric` call `validate()`, so a payload written through the lower-level `encrypt_observer_payload` path is still rejected on read. | `:174` and `:189-190`; regression test `decrypt_agent_turn_metric_rejects_negative_cost_bypassing_encrypt` (`agent_turn_metric.rs:476-507`) | encrypt / decrypt |
| BR-72 | Unrecognized `stopReason` strings deserialize to `Unknown` instead of failing the payload; token counts survive. | hand-written `Deserialize` `:64-77`; test `unknown_stop_reason_maps_to_unknown_not_error` (`agent_turn_metric.rs:321-345`) | payload parse |
| BR-73 | `deltaReliable` defaults to `true` when absent on the wire. | `#[serde(default)]` `:126-127` + `:142-144`; test `delta_reliable_defaults_to_true_when_absent` (`agent_turn_metric.rs:275-283`) | payload parse |
| BR-74 | `session_id` and `turn_seq` are REQUIRED whenever `cumulative` is present; `turn_seq` must strictly increase within a `session_id`, and a publisher restart that loses the counter must start a new `session_id`. | doc only, `:97-108` — **not enforced in code** | documented publisher contract |

---

### 10. Observer payload rules (`src/observer.rs`)

| # | Rule | Enforced at | Trigger |
|---|------|-------------|---------|
| BR-75 | Ciphertext must be 132–87,472 chars to be treated as NIP-44 v2; out-of-range content is rejected **before** any decryption attempt. | `observer.rs:53-55`, `:85-90` | `decrypt_observer_payload()` |
| BR-76 | Plaintext JSON must be ≤ 65,535 bytes on both encrypt and decrypt; oversize plaintext is zeroized before the error returns. | `observer.rs:64-72` (encrypt), `:96-104` (decrypt) | both helpers |
| BR-77 | Plaintext is zeroized after use on every path — success, size failure, and parse failure. | `observer.rs:67`, `:74`, `:100`, `:107` | both helpers |
| BR-78 | Encryption always uses NIP-44 `Version::V2`. | `observer.rs:70` | `encrypt_observer_payload()` |
| BR-79 | Decryption uses `event.pubkey` as the peer key, so only the recipient of the event's ECDH pair can read it. | `observer.rs:92-95` | `decrypt_observer_payload()` |

---

### 11. Git permission rules (`src/git_perms.rs`)

#### Pattern grammar

| # | Rule | Enforced at |
|---|------|-------------|
| BR-80 | Pattern must be non-empty. | `git_perms.rs:84-86` |
| BR-81 | Pattern length ≤ 256 chars (`MAX_PATTERN_LENGTH`). | `:87-89` |
| BR-82 | Pattern must start with `refs/`. | `:90-92` |
| BR-83 | `**` is allowed only as the final segment. | `:100-107` |
| BR-84 | At most 3 wildcard segments (`*` and `**` both count). | `:108-111`, `:113-116` |
| BR-85 | Empty segments (e.g. `refs//x`) are rejected. | `:121-122` |
| BR-86 | Partial globs (`v*`, `?`, `[`, `]` inside a segment) are rejected. | `:123-131` |
| BR-87 | Literal segments are restricted to `[A-Za-z0-9._-]`. | `:132-137` |
| BR-88 | `*` matches exactly one path segment; non-recursive patterns require an exact segment-count match. | `:170-178` |
| BR-89 | `**` matches **one or more** remaining segments — `refs/heads/**` does not match `refs/heads`. | `:151-168`; test `pattern_recursive_requires_at_least_one_segment` (`git_perms.rs:676-681`) |

#### Update classification

| # | Rule | Enforced at |
|---|------|-------------|
| BR-90 | `old_oid == 40 zeros` → Create; else `new_oid == 40 zeros` → Delete; else `is_ancestor` → FastForward; else NonFastForward. Create is checked before Delete. | `git_perms.rs:206-221` |

#### Rule parsing

| # | Rule | Enforced at |
|---|------|-------------|
| BR-91 | A `buzz-protect` tag needs at least 2 values (pattern + ≥1 rule), else `TooFewValues`. | `:306-308` |
| BR-92 | `push:<role>` is parsed via `MemberRole::FromStr`; unknown roles → `InvalidRole`. | `:317-320` |
| BR-93 | `push:bot` and `push:guest` are rejected as nonsensical. | `:321-326`; test `parse_protection_tag_rejects_push_bot_and_guest` (`git_perms.rs:760-771`) |
| BR-94 | Multiple `push:` rules in one tag collapse to the **strictest** (highest permission level). | `:327-336` |
| BR-95 | Recognized flag rules are exactly `no-force-push`, `no-delete`, `require-patch`; unknown rules are skipped and reported for logging (forward compatibility). | `:337-346`; test `parse_protection_tag_unknown_rule_skipped` (`git_perms.rs:743-750`) |
| BR-96 | At most 50 protection rules per repo (`MAX_PROTECTION_RULES`); the 51st `buzz-protect` tag errors with `TooManyRules`. | `:379-395` |
| BR-97 | Non-`buzz-protect` tags are skipped rather than erroring. | `:384-386` |

#### Effective rules + evaluation

| # | Rule | Enforced at |
|---|------|-------------|
| BR-98 | `EffectiveRules` unions every matching pattern: strictest `push_role`, logical OR of `no_force_push` / `no_delete` / `require_patch`, and `has_explicit_match` if any matched. | `:447-475`; test `effective_rules_union_strictest_role` (`git_perms.rs:773-783`) |
| BR-99 | Built-in defaults when no rule matches: branch/tag Create → Member; other Create → Admin; branch FastForward → Member; tag FastForward (tag move) → Admin; other FastForward → Admin; NonFastForward → Admin; Delete → Admin. | `:403-425` |
| BR-100 | If no explicit rule matches, only the default minimum role is checked; the denial message states "using built-in defaults". | `:511-525` |
| BR-101 | `require-patch` denies **all** direct pushes regardless of role or update kind. | `:527-533`; doc `:248-253`; test `evaluate_require_patch_blocks_all` (`git_perms.rs:861-872`) |
| BR-102 | An explicit `push:<role>` can never *weaken* the built-in default: the effective minimum is `max(explicit, default)`. | `:535-552`; test `evaluate_push_member_cannot_weaken_destructive_defaults` (`git_perms.rs:943-978`) |
| BR-103 | A rule that sets only flags (no `push:`) still enforces the built-in default role — a Guest cannot slip through. | `:546-551`; regression test `evaluate_guest_denied_even_with_only_no_force_push_rule` (`git_perms.rs:926-941`) |
| BR-104 | `no-force-push` blocks NonFastForward updates for **everyone**, including Owner. | `:554-560`; test `evaluate_no_force_push_blocks_owner` (`git_perms.rs:816-830`) |
| BR-105 | `no-force-push` must not block FastForward updates. | `:559`; test `evaluate_no_force_push_allows_fast_forward` (`git_perms.rs:832-846`) |
| BR-106 | `no-delete` blocks Delete updates for everyone, including Admin. | `:562-568`; test `evaluate_no_delete_blocks_admin` (`git_perms.rs:848-859`) |
| BR-107 | Check order inside an explicit match: require-patch → role → no-force-push → no-delete. | sequential structure `:527-570` |
| BR-108 | A push is atomic: `evaluate_push` collects denials from every ref and fails the whole push if any ref is denied. | `:584-597`; doc `:580-582`; test `evaluate_push_multiple_refs_partial_deny` (`git_perms.rs:980-1002`) |
| BR-109 | Permission model statement: channel role = repo role; `buzz-protect` constraints apply to everyone including the owner. | doc `git_perms.rs:5-7` |

---

### 12. NIP-AB pairing rules

#### QR payload (`src/pairing/qr.rs`)

| # | Rule | Enforced at |
|---|------|-------------|
| BR-110 | URI length must not exceed 2048 characters. | `qr.rs:105-111` |
| BR-111 | URI must start with `nostrpair://` and contain a `?` query string. | `qr.rs:113-127` |
| BR-112 | Pubkey must be exactly 64 **lowercase** hex chars (uppercase rejected). | `qr.rs:129-137`, helper `is_lowercase_hex` `:240-242`; test `reject_uppercase_hex_in_pubkey` (`qr.rs:555-571`) |
| BR-113 | `secret` must be present and exactly 64 lowercase hex chars → 32 bytes. | `qr.rs:153-168` |
| BR-114 | An all-zeros `session_secret` is rejected. | `qr.rs:169-174`; test `reject_all_zeros_session_secret` (`qr.rs:538-553`) |
| BR-115 | At least one `relay` parameter is required. | `qr.rs:176-181` |
| BR-116 | Every relay URL must fully parse, use scheme `ws` or `wss`, and have a host. | `qr.rs:183-204` |
| BR-117 | Version defaults to 1 when `v=` is absent; any version ≠ 1 is rejected. | `qr.rs:145-151`; tests `reject_unsupported_version` (`qr.rs:517-527`) and `default_version_when_absent` (`qr.rs:529-536`) |
| BR-118 | Unknown query parameters are ignored. | `qr.rs:141` |
| BR-119 | Relay values are percent-encoded with `NON_ALPHANUMERIC` on encode and percent-decoded (lossy UTF-8) on decode. | `qr.rs:220-236` |
| BR-120 | The session secret is zeroized when a `QrPayload` drops. | `qr.rs:56-60` |

#### Key derivation (`src/pairing/crypto.rs`)

| # | Rule | Enforced at |
|---|------|-------------|
| BR-121 | `session_id = HKDF-SHA256(IKM=session_secret, salt=[], info="nostr-pair-session-id", L=32)`. | `crypto.rs:29-42`, `:54-56`, info const `:23` |
| BR-122 | `sas_input = HKDF-SHA256(IKM=ecdh_shared, salt=session_secret, info="nostr-pair-sas-v1", L=32)`; `sas_code = be_u32(sas_input[0..4]) mod 1_000_000`. | `crypto.rs:70-75`, info const `:24` |
| BR-123 | `transcript = session_id ‖ source_pubkey ‖ target_pubkey ‖ sas_input` (128 bytes, order-sensitive); `transcript_hash = HKDF-SHA256(IKM=transcript, salt=session_secret, info="nostr-pair-transcript-v1", L=32)`. | `crypto.rs:89-105`; order-sensitivity test `transcript_hash_sensitive_to_pubkey_order` (`crypto.rs:325-351`) |
| BR-124 | SAS is displayed zero-padded to exactly 6 digits. | `crypto.rs:116-118`; tests `format_sas_zero_padding` (`crypto.rs:353-360`) and `format_sas_always_six_chars` (`crypto.rs:362-369`) |
| BR-125 | Comparisons of 32-byte secret-derived values use constant-time equality. | `crypto.rs:126-129` |
| BR-126 | Pinned spec test vectors (pubkeys, ECDH, session id, sas_input, SAS `863346`, transcript hash) must not drift. | test `all_test_vectors` (`crypto.rs:272-323`) |

#### Session state machine (`src/pairing/session.rs`)

| # | Rule | Enforced at |
|---|------|-------------|
| BR-127 | Default session lifetime is 120 s; every handler calls `check_expired()` first and fails with `SessionExpired`. | `session.rs:43` (`DEFAULT_TIMEOUT = 120s`), applied at `:138`/`:309`; `check_expired` at `:698-703`; test `expired_session_rejects_operations` (`session.rs:1130-1145`) |
| BR-128 | Every handler asserts both the exact expected state and the expected role before doing work. | `expect_state` `:706-715`, `expect_role` `:717-726`; called at the top of each handler (e.g. `:150-152`, `:330-332`, `:392-394`) |
| BR-129 | Offer handling requires protocol `version == 1`. | `:169-175` |
| BR-130 | The offer's `session_id` must equal the locally derived session id, compared in constant time; mismatch → `InvalidSessionId`. | `:177-185`; test `reject_invalid_session_id` (`session.rs:986-1002`) |
| BR-131 | The source locks onto the offering pubkey as its peer; every later event must come from that exact pubkey. | `:187-188`, `validate_event_from_peer` `:681-696`; test `reject_event_from_wrong_pubkey` (`session.rs:1004-1043`) |
| BR-132 | The ECDH shared secret is zeroized immediately after SAS derivation. | `:186-191` (source, `ecdh.zeroize()` at `:189`), `:290-295` (target, `:295`) |
| BR-133 | Basic event validation: full `event.verify()` (id + sig, `:632-635`), duplicate-id check (`:638-643`), kind must equal 24134 (`:645-650`), and a `p` tag must match our ephemeral pubkey (`:653-666`). | `session.rs:631-674` |
| BR-134 | Duplicate event ids are rejected; ids are recorded **only after** a message is fully accepted, so speculative probes (e.g. `handle_abort` used as a detector) cannot poison the dedup set. | `record_event` `:676-678`, called at `:193`, `:264`, `:369`, `:404`, `:468`; tests `duplicate_event_id_is_rejected` (`session.rs:1148-1184`), `speculative_abort_does_not_poison_dedup` (`:1186-1212`), `wrong_type_message_not_recorded` (`:1214-1267`) |
| BR-135 | Target must obtain explicit user confirmation: `handle_sas_confirm` moves to `AwaitingConfirmation`, and only `confirm_target_sas()` unlocks `Transferring` so a payload can be accepted. | `:368`, `:376-382`; test `target_must_confirm_sas_before_payload` (`session.rs:1056-1089`) |
| BR-136 | Transcript-hash mismatch on the target aborts the session immediately (`state = Aborted`, `TranscriptMismatch`). | `:359-367` |
| BR-137 | Exactly one payload per session — state advances to `PayloadExchanged`, so a second send or receive fails. | `:249`, `:403`; test `reject_duplicate_payload` (`session.rs:1091-1119`) |
| BR-138 | `complete(success = false)` sets state to `Aborted`, returns an error, and does **not** record the event id. | `:267-278`; test `complete_failure_aborts_without_recording` (`session.rs:1269-1314`) |
| BR-139 | `abort()` is rejected from terminal states (`Completed`/`Aborted`) — a finished session cannot regress. | `:431-437`; test `local_abort_after_completed_is_rejected` (`session.rs:871-898`) |
| BR-140 | `abort()` with no known peer transitions to `Aborted` and returns `None` (nothing to encrypt). | `:438-441`; test `abort_without_peer_returns_none` (`session.rs:862-869`) |
| BR-141 | `handle_abort()` requires a known peer, so an anonymous relay observer cannot kill a session; late aborts in terminal states are ignored. | `:449-462`; tests `reject_handle_abort_before_peer_known` (`session.rs:900-931`) and `reject_abort_after_completed` (`session.rs:933-984`) |
| BR-142 | Outbound events carry a `created_at` reduced by random 0–30 s jitter for metadata privacy, exactly one `p` tag (the peer), kind 24134, NIP-44 v2 content. | `session.rs:561-590` (jitter at `:578-579`, single `p` tag at `:582`, NIP-44 V2 at `:566-571`) |
| BR-143 | Inbound content length must be 132–87,472 chars before decryption; decrypted plaintext must be ≤ 65,535 bytes, otherwise it is zeroized and rejected. | `session.rs:592-629` (length gate `:595`, plaintext cap `:609-615`) |
| BR-144 | Serialized plaintext is zeroized after encryption; decrypted plaintext is zeroized on both success and parse failure; the transient payload clone is zeroized in `send_payload`. | `:573` (serialized plaintext), `:610`/`:619` (decrypted plaintext), `:241-247` (payload clone) |
| BR-145 | `qr_uri()` returns `None` for a Target session (only the source displays a QR). | `session.rs:517-527` (`role != Source` → `None` at `:518-520`) |
| BR-146 | Session secret, session id, and SAS input are zeroized on drop. | `Drop` impl `session.rs:731-739` |

#### Message wire rules (`src/pairing/types.rs`)

| # | Rule | Enforced at |
|---|------|-------------|
| BR-147 | Messages are JSON tagged unions on `"type"` in kebab-case (`offer`, `sas-confirm`, `payload`, `complete`, `abort`). | `types.rs:18-63`; tests `:104-126`, `:147-155` |
| BR-148 | `Offer.version` defaults to 1 when absent (pre-versioned compatibility). | `types.rs:12-14`, `:28-31`; test `:117-131` |
| BR-149 | Unknown `abort.reason` strings deserialize to `Unknown` and are to be treated as `protocol_error`; callers must not emit `Unknown` outbound. | `types.rs:88-101`; tests `:196-206`, `:208-215` |

---

### 13. SSRF classification rules (`src/network.rs`)

Documented as: webhook targets must not resolve to these addresses (`network.rs:9-11`). Exact ranges in the security aspect doc; the rule itself:

| # | Rule | Enforced at |
|---|------|-------------|
| BR-150 | An IPv4-mapped IPv6 address is re-checked recursively against the IPv4 rule set, so `::ffff:10.0.0.1` is blocked. | `network.rs:45-48` |
| BR-151 | IPv4 blocked if loopback, RFC1918 private, link-local, first octet 0, broadcast, CGNAT `100.64.0.0/10`, or benchmarking `198.18.0.0/15`. | `network.rs:26-40` |
| BR-152 | IPv6 blocked if loopback, unspecified, ULA `fc00::/7`, link-local `fe80::/10`, multicast `ff00::/8`, or documentation `2001:db8::/32`. | `network.rs:49-56` |
