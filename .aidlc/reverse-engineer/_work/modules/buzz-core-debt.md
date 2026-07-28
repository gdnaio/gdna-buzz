## Module: buzz-core (`crates/buzz-core`)

### Aspect: Technical Debt

Findings are factual observations from the source. No severity ranking is implied by ordering.

---

### 1. Size and complexity hotspots

#### File sizes (`wc -l`, 20 `.rs` files, 7,577 lines total)

| File | Lines | Test module starts | Non-test lines (approx.) |
|---|---|---|---|
| `src/pairing/session.rs` | 1,315 | `:756` | ~755 |
| `src/engram.rs` | 1,049 | `:607` | ~606 |
| `src/git_perms.rs` | 1,003 | `:601` | ~600 |
| `src/kind.rs` | 784 | `:747` | ~746 |
| `src/pairing/qr.rs` | 588 | `:245` | ~244 |
| `src/agent_turn_metric.rs` | 508 | `:194` | ~193 |
| `src/pairing/crypto.rs` | 413 | `:131` | ~130 |
| `src/filter.rs` | 300 | `:106` | ~105 |
| `src/tenant.rs` | 275 | `:175` | ~174 |
| `src/pairing/types.rs` | 242 | `:98` | ~97 |
| `src/network.rs` | 200 | `:56` | ~55 |
| `src/channel.rs` | 198 | `:181` | ~180 |
| `src/observer.rs` | 159 | `:113` | ~112 |
| `src/relay.rs` | 121 | `:80` | ~79 |
| `src/pairing/mod.rs` | 80 | — | 80 |
| `src/lib.rs` | 75 | — | 75 |
| `src/event.rs` | 74 | `:53` | ~52 |
| `src/presence.rs` | 73 | `:37` | ~36 |
| `src/verification.rs` | 71 | `:34` | ~33 |
| `src/error.rs` | 20 | — | 20 |

Three files exceed 1,000 lines. The repo's per-file size guard (`mobile/scripts/check-file-sizes.mjs`, 1,000-line ceiling) applies to `mobile/`, `desktop/`, and `web/`; no equivalent guard was found for Rust crates, so these are not gate failures — just the largest units in the crate.

#### Longest non-test functions

| Lines | Function | Location | Notes |
|---|---|---|---|
| 117 | `decode_qr` | `src/pairing/qr.rs:104` | one function performs 9 distinct validations plus query parsing; each step is a separate guard clause |
| 85 | `parse_strict_json` | `src/engram.rs:283` | contains a nested `struct StrictValue` and two full trait impls (`DeserializeSeed`, `Visitor` with 11 visit methods) inside the function body |
| 71 | `evaluate_ref_update` | `src/git_perms.rs:508` | 4 sequential policy checks with distinct denial messages |
| 70 | `filter_match_one` | `src/filter.rs:35` | 6 field checks plus the nested `#h` fallback with 4 levels of `if` nesting (`:83-100`) |
| 70 | `validate_and_decrypt` | `src/engram.rs:488` | 8 envelope checks + decrypt + re-derivation |
| 59 | `parse_protection_tag_with_warnings` | `src/git_perms.rs:303` | |
| 58 | `RefPattern::parse` | `src/git_perms.rs:83` | one 60-line `for` loop with a 5-branch `if/else if` chain (`:97-144`) |
| 48 | `handle_offer` | `src/pairing/session.rs:149` | |
| 45 | `extract_refs` | `src/engram.rs:384` | hand-rolled byte scanner with two nested `while` loops and manual index arithmetic |

`kind.rs` is long but flat: 130 constants, 4 slices, 11 functions, and a 38-line compile-time assertion block — no control flow deeper than a `matches!`.

---

### 2. Panic-capable code in non-test paths

The repo rule is "do not introduce new `unwrap()`/`expect()` in production paths" (AGENTS.md). Current state:

| Construct | Location | Justification present in code? |
|---|---|---|
| `.expect("valid keys produce conversation key")` | `src/engram.rs:137` | no inline justification; relies on caller supplying valid keys |
| `.expect("HMAC-SHA256 is keyed-prefix MAC; new_from_slice cannot fail")` | `src/engram.rs:149` | yes — `:145-147` explains why it is infallible and why the error is not propagated |
| `unreachable!()` in `RefPattern::matches` | `src/git_perms.rs:163` and `:177` | no comment; relies on the invariant that `RecursiveWildcard` only ever occupies the last segment (enforced in `parse` at `:100-107`). A `RefPattern` constructed by any future path that violates that invariant would panic |
| `.expect("sign")` ×2 | `src/lib.rs:59`, `:68` | inside the `test-utils`-gated helper module, not compiled into a normal build |

Everything else in the crate returns `Result`. No `panic!`, `todo!`, or `unimplemented!` appears in `src/`.

---

### 3. Duplicated constants and hard-coded literals

| Duplication | Locations |
|---|---|
| NIP-44 plaintext cap `65_535` defined **three times** | `observer::OBSERVER_MAX_PLAINTEXT_LEN` (`src/observer.rs:25`), `engram::NIP44_PLAINTEXT_MAX` (`src/engram.rs:28`), inline literal in the pairing session (`src/pairing/session.rs:609`) |
| NIP-44 ciphertext window `132..=87472` defined twice | named constants `observer::NIP44_MIN_CONTENT_LEN` / `NIP44_MAX_CONTENT_LEN` (`src/observer.rs:21`, `:23`) vs an inline range literal in `PairingSession::decrypt_message` (`src/pairing/session.rs:595`) |
| Result-gated kind set defined twice | `kind::RESULT_GATED_KINDS` (`src/kind.rs:129`) vs two hard-coded `!=` comparisons in `reader_authorized_for_event` (`src/filter.rs:25`) — adding a kind to the constant will not extend the runtime gate |
| Pairing kind | `kind::KIND_PAIRING` (`src/kind.rs:329`) re-derived into a module-private `PAIRING_KIND: u16` (`src/pairing/session.rs:46`) |
| `KIND_*` → `Kind::Custom(x as u16)` casts | repeated at `src/engram.rs:468`, `:860`, `src/pairing/session.rs:46`, and in tests (`src/filter.rs:244`, `:276`, `src/observer.rs:131`, `:147`, plus bare numeric `Kind::Custom(44200)` at `src/agent_turn_metric.rs:239`, `:262`, `:493`) — no shared conversion helper despite `event_kind_u32`/`event_kind_i32` existing for the reverse direction (`src/kind.rs:696-704`) |
| Tag-kind string comparison idiom | `t.kind().to_string() == "d"` / `"p"` / `"h"` appears at `src/engram.rs:506`, `:525`, `:851`, `src/filter.rs:74`, `:84` — string allocation per tag per comparison, and no shared helper |

---

### 4. Registry consistency issues (`kind.rs`)

| Finding | Evidence |
|---|---|
| 3 of 130 kind constants are absent from `ALL_KINDS` with **no code comment explaining the omission**: `KIND_AUTH` (22242), `KIND_NOSTR_IDENTITY_BINDING` (24243), `KIND_PUSH_LEASE` (30350) | `src/kind.rs:77`, `:81`, `:109` vs the list at `:490-617` |
| The implied "never stored ⇒ excluded" rule does not hold: `KIND_BLOSSOM_AUTH` (`:78-79`) and `KIND_HTTP_AUTH` (`:82-83`) carry "not stored" doc comments yet appear in `ALL_KINDS` (`:551`, `:554`) | same |
| `no_duplicate_kind_values` only covers the 127 listed kinds; the 3 excluded constants are outside the duplicate check | `src/kind.rs:752-758` |
| Declaration order does not follow numeric order (e.g. `KIND_CHANNEL_METADATA` 41 at `:54` sits between 30030 (`:52`) and 5 (`:56`); `KIND_STREAM_MESSAGE` 9 at `:343` sits inside the 40000-series block), so a reader cannot scan for range collisions visually | `src/kind.rs:9-487` |
| Naming inconsistency: 4 admin command kinds use a `RELAY_ADMIN_` prefix while the other 126 use `KIND_` | `src/kind.rs:253-259` |
| `KIND_CHANNEL_METADATA` (41) is documented "Not used by Buzz today" yet is a live member of `is_replaceable` and `ALL_KINDS` | `src/kind.rs:53-54`, `:629`, `:502` |
| Legacy-migration notes remain in doc comments as historical record: "V1 used kind:10001 (replaceable range — wrong), then 40001" (`:338`), "V1 used kind:10002 (replaceable range — wrong)" (`:344`), "V1 used kind:10004 (replaceable range + NIP-51 collision — wrong)" (`:346`), "V1 used addressable range (30001–30003) — wrong" (`:412`) | |
| Self-identified scaling note: `AUTHOR_ONLY_KINDS` is "a tiny linear set… If this grows past ~4 kinds, convert to a compile-time bitset or sorted array" | `src/kind.rs:118-119` |
| An empty section header with no members: `// User groups (47000–47999)` | `src/kind.rs:448` |

---

### 5. Documentation contradictions

| Repo doc claim | Reality in code | Delta |
|---|---|---|
| `ARCHITECTURE.md:142`: "`buzz-core` defines all 81 kinds as `pub const KIND_*: u32`" | 130 kind constants (`src/kind.rs`, 134 `u32` consts minus 4 range bounds) | undercounts by 49 |
| `ARCHITECTURE.md:346`: "`pub const ALL_KINDS: &[u32]  // 80 entries (KIND_AUTH excluded — never stored)`" | `ALL_KINDS` has 127 entries (`src/kind.rs:490-617`) | undercounts by 47 |
| `ARCHITECTURE.md:346`: only `KIND_AUTH` named as excluded | 3 constants are excluded (`KIND_AUTH`, `KIND_NOSTR_IDENTITY_BINDING`, `KIND_PUSH_LEASE`) | 2 omissions; the `KIND_AUTH` exclusion itself is accurate |
| Task brief / module description: "~81 kinds" | 130 | same drift as ARCHITECTURE.md |
| Task brief / module description: "SSRF helpers" (plural) | exactly one function, `is_private_ip` (`src/network.rs:25`) | minor |
| `AGENTS.md` gotcha #1: "Kind `39000` for channel metadata, not `41` — kind 41 is NIP-01 (unused)" | matches the code (`src/kind.rs:53-54`, `:286`) | consistent |
| `AGENTS.md`: "All event kind integers are defined in `buzz-core/src/kind.rs`" | consistent for Rust; parallel mirrors exist outside this crate (`desktop/src/shared/constants/kinds.ts`, `mobile/.../nostr_models.dart` per AGENTS.md) with no automated cross-language drift check found in this crate | partial |
| `crates/buzz-core/Cargo.toml:29`: "NO tokio, NO sqlx, NO redis, NO axum" | true today, but nothing enforces it: root `deny.toml:90-92` `[bans]` has no per-crate bans, and no `[workspace.lints]`/`[lints]` table exists | convention only |

---

### 6. Test-coverage gaps

213 `#[test]` functions exist (static count), but coverage is uneven. Gaps identified by reading the test modules:

| Area | Gap | Evidence |
|---|---|---|
| `error.rs` | no tests at all; `VerificationError` `Display` strings are unverified | `src/error.rs` has no `#[cfg(test)]` block |
| `event.rs` | its single test (`tampered_signature_fails_verify`, `:56-66`) exercises `nostr::Event`, not `StoredEvent`. `StoredEvent::new`, `with_received_at`, and `is_verified()` have no direct test | `src/event.rs:53-67` |
| `channel.rs` | the only test covers `canonical_channel_name`. `MemberRole::permission_level`, `has_at_least`, `is_elevated`, all three `FromStr` impls, `as_str`, and `Display` have no direct tests in this module (the role ladder is exercised indirectly through `git_perms` tests) | `src/channel.rs:181-198` |
| `filter.rs` | 6 tests. No test for: `ids` **prefix** matching (only exact ids are used), `since`/`until` **boundary equality** (`created_at == since`), multiple generic-tag keys AND-ed together, a filter with no constraints matching everything, or a present-but-empty `kinds` list | `src/filter.rs:106-300` |
| `kind.rs` | 4 tests. No test asserts the `ALL_KINDS` **count**, no test asserts that the 3 excluded constants are intentionally absent, and `is_command_kind` / `is_relay_admin_kind` / `is_workflow_execution_kind` / `is_identity_archive_request_kind` / `is_ephemeral` / `event_kind_u32` / `event_kind_i32` have no unit tests (some are covered only by the compile-time assertion block) | `src/kind.rs:747-784` |
| `observer.rs` | 2 tests. `encrypt_observer_payload`'s `PlaintextTooLarge` path and the upper ciphertext bound (87,472) are untested; only the too-short case is covered | `src/observer.rs:113-158` |
| `relay.rs` | 3 tests. `MissingHost` and `InvalidUrl` variants are never asserted; only `InvalidScheme`, `Credentials`, `Fragment` are | `src/relay.rs:80-121` |
| `tenant.rs` | `TenantContext` equality/clone semantics and `CommunityId` ordering (`PartialOrd`/`Ord` are derived) are untested | `src/tenant.rs:175-275` |
| `engram.rs` | `select_head`'s **id tiebreak** branch is not actually exercised: the test named `select_head_lww_with_id_tiebreak` (`src/engram.rs:789`) builds two events with *different* `created_at` values (`1_700_000_000` and `1_700_000_001` at `:800-801`), so only the timestamp branch runs; its own comment claims three events and a tie check (`:790-791`). `Body::Core` round-trip through `build_event`/`validate_and_decrypt` and the `BodyTooLarge` boundary (exactly 65,535 vs 65,536) are also untested | `src/engram.rs:789-803` |
| `agent_turn_metric.rs` | `validate()` is well covered, but the documented `session_id`/`turn_seq`-with-`cumulative` requirement has no test because it has no implementation | `src/agent_turn_metric.rs:147-169` |
| `git_perms.rs` | `MAX_PROTECTION_RULES` (51-tag) path, `MAX_PATTERN_LENGTH` (257-char) path, and `parse_protection_tags` (the multi-tag entry point) have no tests — all 34 tests call `parse_protection_tag` on a single tag | `src/git_perms.rs:601-1003` |
| `pairing/session.rs` | no test for `sign_event`, `relay_urls`, or the ciphertext-length rejection path in `decrypt_message`; the 0–30 s jitter is untested | `src/pairing/session.rs:756-1315` |
| whole crate | no property-based tests (no `proptest`/`quickcheck` dependency or usage), despite several pure functions with clear algebraic properties (`normalize_host` idempotence, `Body` JSON round-trip, `is_private_ip` range partitioning, `select_head` total order) | `Cargo.toml:13-27` |
| whole crate | no integration test directory (`tests/`); all tests are unit tests inside the crate | crate root contains only `Cargo.toml` and `src/` |

Note on measurement: the crate was **not compiled** during this analysis (`cargo` is unavailable without the repo's Hermit activation in this environment), so all test counts are static reads of `#[test]` attributes and no pass/fail state is claimed.

---

### 7. Possibly-unused public surface

Measured by searching all `*.rs` files under `crates/` and `desktop/src-tauri/` (excluding `crates/buzz-core/` itself) for each symbol name. Caveats: constants may also be referenced by numeric literal, by the TypeScript/Dart kind mirrors, or via type inference (so a type can be *used* without its name appearing).

**24 of 130 kind constants have no by-name reference outside `buzz-core`:**

`KIND_CHANNEL_METADATA`, `KIND_FILE_METADATA`, `KIND_BLOSSOM_AUTH`, `KIND_HTTP_AUTH`, `KIND_NIP29_CREATE_INVITE`, `KIND_NIP43_MEMBER_ADDED`, `KIND_NIP43_MEMBER_REMOVED`, `KIND_NIP29_GROUP_ROLES`, `KIND_HUDDLE_REACTION`, `KIND_SYSTEM_MESSAGE`, `KIND_CHANNEL_SUMMARY`, `KIND_DM_CREATED`, `KIND_JOB_ACCEPTED`, `KIND_JOB_CANCEL`, `KIND_JOB_ERROR`, `KIND_WORKFLOW_STEP_STARTED`, `KIND_WORKFLOW_STEP_COMPLETED`, `KIND_WORKFLOW_STEP_FAILED`, `KIND_WORKFLOW_COMPLETED`, `KIND_WORKFLOW_FAILED`, `KIND_WORKFLOW_CANCELLED`, `KIND_WORKFLOW_APPROVAL_GRANTED`, `KIND_AUDIT_ENTRY`, `KIND_MEDIA_UPLOAD`.

Several of these are documented as relay-emitted or client-side kinds (e.g. `KIND_CHANNEL_SUMMARY` is relay-signed, `KIND_MEDIA_UPLOAD` is labelled "Internal kind for media upload audit entries. Not a relay event kind." at `src/kind.rs:465`), so absence of a Rust reference does not by itself mean the kind is dead.

**Other public items with no by-name reference outside `buzz-core`:**

| Item | Location | Note |
|---|---|---|
| `parse_protection_tag_with_warnings` | `src/git_perms.rs:303` | the `unknown_rules` reporting path exists so callers "can log warnings" (doc at `:376-377`) but no caller outside this crate reads it |
| `default_min_role` | `src/git_perms.rs:403` | used internally by the evaluator |
| `evaluate_ref_update` | `src/git_perms.rs:508` | consumers appear to use `evaluate_push` |
| `EffectiveRules` (and `for_ref`) | `src/git_perms.rs:432`, `:447` | used internally |
| `MAX_PROTECTION_RULES`, `MAX_PATTERN_LENGTH`, `MAX_WILDCARDS_PER_PATTERN` | `src/git_perms.rs:19-23` | limits not surfaced to callers for error messages |
| `NIP44_MIN_CONTENT_LEN`, `NIP44_MAX_CONTENT_LEN` | `src/observer.rs:21`, `:23` | see the duplication finding in §3 |
| `D_TAG_DOMAIN` | `src/engram.rs:24` | exported for spec traceability |
| `MemberRole::permission_level`, `MemberRole::has_at_least` | `src/channel.rs:142`, `:155` | used internally by `git_perms` |
| `Body::is_tombstone` | `src/engram.rs:183` | callers appear to match on `Body::Memory { value: None, .. }` directly |
| `validate_slug` | `src/engram.rs:67` | callers go through `normalize_slug` |
| `QrPayload` (type name) | `src/pairing/qr.rs:34` | the type **is** exercised by `crates/buzz-pairing-cli/src/main.rs:119-227` through `encode_qr`/`decode_qr` and inference — a naming-search artifact, not dead code |

---

### 8. Structural / consistency observations

| # | Observation | Evidence |
|---|---|---|
| D-1 | Orphaned comment fragment with no antecedent line, inside the engram test module: `//    vectors". Pinning these as CI invariants is the single best` — reads as a partially deleted doc block | `src/engram.rs:615` |
| D-2 | A `// SAFETY:` comment annotates a **safe** slice operation; the convention normally marks `unsafe` blocks, of which the crate has none | `src/engram.rs:410-413` |
| D-3 | `filter.rs` re-implements the result-gated kind check instead of consuming `kind::RESULT_GATED_KINDS`, creating two sources of truth for a security-relevant set | `src/filter.rs:25` vs `src/kind.rs:129` |
| D-4 | `git_perms::parse_protection_tags` checks the rule cap **before** parsing, so a malformed 51st tag surfaces `TooManyRules` instead of its parse error | `src/git_perms.rs:383-394` |
| D-5 | `PatternError::InvalidSegment(String)` is used for two unrelated conditions — a bad segment and "`**` must be the last segment" — the latter passing a *sentence* as the segment value | `src/git_perms.rs:101-106` vs `:123-137` |
| D-6 | `MAX_WILDCARDS_PER_PATTERN = 3` counts `*` and `**` together; the doc comment says "wildcard segments per pattern" without stating that `**` counts as one, so the effective limit for `refs/*/*/*` (3) vs `refs/*/*/**` (3) is only discoverable from code | `src/git_perms.rs:23`, `:100-116` |
| D-7 | `agent_turn_metric.rs` re-exports `ObserverPayloadError` under a second name (`AgentTurnMetricError`) while its own functions return the original type in their signatures, so both names appear in the public API for the same errors | `src/agent_turn_metric.rs:15` vs `:169`, `:185` |
| D-8 | `AgentTurnMetricPayload.timestamp` is an unvalidated `String` and `channel_id` is a `String` rather than `Uuid`, despite the crate already depending on `uuid` and `chrono` | `src/agent_turn_metric.rs:97` (`channel_id`), `:112` (`timestamp`) |
| D-9 | `TokenCounts` mixes two serde strategies: four fields serialize `null` when absent while `cache_read_tokens`/`cache_write_tokens` are skipped entirely — intentional per the "not reported vs zero" contract, but it means consumers must handle both shapes | `src/agent_turn_metric.rs:24-42`, test `:301-319` |
| D-10 | `StopReason` has a derived `Serialize` but a hand-written `Deserialize`, so the two are maintained independently; a new variant added to the enum will silently deserialize as `Unknown` until the manual match is updated | `src/agent_turn_metric.rs:49-77` |
| D-11 | `parse_strict_json` embeds a full serde `Visitor` implementation (11 methods) inside a function body, which keeps it private but makes it untestable in isolation and unavailable for reuse by other strict-JSON needs | `src/engram.rs:283-380` |
| D-12 | `extract_refs` is a hand-rolled byte scanner with manual index arithmetic and documented surprising cases (`[[[mem/x]]]` yields nothing) rather than a parser or regex; correctness rests entirely on its 15 tests | `src/engram.rs:384-430`, tests `:891-1047` |
| D-13 | Engram `d`-tag and `p`-tag comparisons are non-constant-time (`!=`, `eq_ignore_ascii_case`) while the pairing module uses `ct_eq` for analogous 32-byte comparisons — an internal inconsistency in comparison policy | `src/engram.rs:533`, `:551` vs `src/pairing/crypto.rs:126-129` |
| D-14 | `PairingSession` is split across four `impl` blocks (`:109`, `:282`, `:424`, `:546`) plus a `#[cfg(test)]` block (`:530-531`); role-specific methods are separated by block but nothing in the type system prevents calling a Target method on a Source session — enforcement is the runtime `expect_role` check | impl blocks at `src/pairing/session.rs:109`, `:282`, `:424`, `:531`, `:546`; `expect_role` at `:717-726` |
| D-15 | `SessionState` has 7 variants and transitions are asserted by string-formatted `UnexpectedMessage` errors rather than a typed transition table, so an illegal transition is only discoverable at runtime | `src/pairing/session.rs:59-78`, `:706-726` |
| D-16 | `PairingError::UnexpectedMessage` is overloaded for at least five distinct failure classes (wrong message type, wrong state, wrong role, bad content length, oversized plaintext), so callers cannot distinguish protocol-shape errors from size-limit errors | 11 construction sites: `src/pairing/session.rs:166` (version), `:272` (complete=false), `:432`/`:451` (terminal state), `:596` (content length), `:611` (plaintext size), `:639` (duplicate id), `:646` (kind), `:708` (state), `:719` (role), `:750` (message type) |
| D-17 | `QrPayload` derives `Clone` **and** implements `Drop` to zeroize its secret — a clone extends the lifetime of the secret beyond the original's drop, and a pairing test relies on that clone (`session.rs:990`) | `src/pairing/qr.rs:37`, `:56-60` |
| D-18 | `is_private_ip` mixes std helpers with hand-written bit masks; the std helpers' exact semantics (e.g. what `is_private()` covers) are only documented in the doc comment, not asserted by tests for each RFC1918 block boundary | `src/network.rs:26-40` |
| D-19 | The crate exposes both `normalize_host` (`tenant.rs:121`) and `normalize_relay_url` (`relay.rs:37`) and `relay_url_authority` (`tenant.rs:156`) — three overlapping normalizers with deliberately different rules; the differences are documented but a caller must read all three to pick correctly | `src/tenant.rs:106-172`, `src/relay.rs:20-78` |
| D-20 | `nostr` types are re-exported from the crate root (`lib.rs:42`), so consumers can depend on `buzz_core::Event` and be coupled to the `nostr` 0.44 major version through this crate without declaring it | `src/lib.rs:42`, `Cargo.toml:14` |

---

### 9. Deprecated API usage

No `#[deprecated]` attribute is declared in this crate, and no call site is annotated as using a deprecated API. Nothing in the crate references a `deprecated` item by name. The closest artefacts are the historical "V1 used kind:X — wrong" doc notes in `src/kind.rs:338-346`, `:412`, which record superseded kind numbers rather than deprecated Rust items.
