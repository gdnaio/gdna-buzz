<!-- Analyzed: 2026-07-25T01:12:08Z | Scope: full project -->
# Code Conventions

> Status: initialized in Phase 1. Patterns, naming, error handling, and testing
> conventions are populated per-module during Phase 2 and consolidated in Phase 3.

## Summary

House rules are documented in `AGENTS.md` / `CONTRIBUTING.md`: no `unsafe`, no new
`unwrap()`/`expect()` in production paths, doc comments on new public API, prefer new event
kinds over new HTTP endpoints, rem-based text sizing in desktop, no `StatefulWidget` in
Flutter, 1000-line file ceiling enforced by scripts.

Re-verified after the post-analysis `CONTRIBUTING.md` rewrite: the rules still
stand, but they now live only in prose — `#![deny(unsafe_code)]` / "do not add
unsafe blocks" at `CONTRIBUTING.md:222-223` and the `unwrap()`/`expect()` ban at
`:230-231`, with `just ci` required at `:183-187` and `:277`. The machine-readable
**PR checklist** that previously restated all of them as tick-boxes was deleted
by that rewrite, so a contributor now has to read the prose sections; the
checklist is no longer available to copy into a PR body.

Batch 2a compliance against those rules:

| Rule | `buzz-core` | `buzz-sdk` | `buzz-persona` | `buzz-ws-client` |
|---|---|---|---|---|
| No `unsafe` | ✅ `#![deny(unsafe_code)]`, 0 blocks | ✅ same | ✅ 0 blocks (no lint attr) | ✅ `#![deny(unsafe_code)]` |
| No `unwrap()`/`expect()` in production | ⚠️ 2 `expect` + 2 `unreachable!` (1 justified inline) | ✅ 0 | ✅ 0 | ⚠️ **2 `unwrap()` + 2 `unreachable!()`** (`connection.rs:170`, `:229`) |
| Doc comments on public API | ✅ `#![warn(missing_docs)]` | ✅ same | ⚠️ no lint; `lib.rs` has no crate doc | ⚠️ convention only, no lint |
| File-size discipline | ⚠️ 3 files >1,000 lines | ⚠️ `builders.rs` **3,824 lines** | ⚠️ `validate.rs` 1,070 | ✅ largest 314 |

The 1,000-line guard (`check-file-sizes.mjs`) covers only `desktop/`, `web/`, and `mobile/`
— **there is no equivalent guard for Rust crates**, which is why `builders.rs` at 3,824
lines holds 51 of 61 public functions without tripping any gate.

Shared idioms across all four crates: one `thiserror` enum per boundary (no `anyhow` in
libraries), guard-clause validation with early `return Err`, `as_str()` + `Display` +
`FromStr` triads on string-mapped enums, `normalize_*`/`canonical_*` for transformers,
inline `#[cfg(test)] mod tests` (no `tests/` dir except `buzz-persona`), and pinned spec
test vectors as module constants. Test-to-code ratio is high: 213 tests in `buzz-core`,
235 in `buzz-sdk`, 145 in `buzz-persona`, but only **3** in `buzz-ws-client` (all
compile-time constant-floor assertions). No property-based tests anywhere in batch 2a.

### Batch 2b compliance (service crates)

| Rule | Status across `buzz-db`, `buzz-auth`, `buzz-pubsub`, `buzz-search`, `buzz-audit`, `buzz-media`, `buzz-workflow` |
|---|---|
| No `unsafe` | ✅ zero blocks in all seven |
| No `unwrap()`/`expect()` in production | ✅ compliant — occurrences are confined to `#[cfg(test)]` modules |
| Doc comments on public API | ⚠️ `missing_docs` is `warn`, not `deny`, where present (e.g. `crates/buzz-pubsub/src/lib.rs:2`), so coverage can regress silently — and did: an orphaned doc comment at `crates/buzz-pubsub/src/lib.rs:43` advertises a module that does not exist |
| No TODO/FIXME/HACK markers | ✅ six of seven clean; `buzz-workflow` carries TODO WF-08 (`src/executor.rs:663`) |
| File-size discipline | ❌ **`crates/buzz-db/src/lib.rs` is 6,106 lines** — larger than `buzz-sdk/src/builders.rs` (3,824) and the biggest Rust file found so far |

The missing Rust file-size guard is now the clearest cross-cutting gap: two files exceed
3,800 lines with nothing to catch them, while `desktop/`, `web/`, and `mobile/` are held to
1,000.

Additional 2b conventions observed:

- **Redis/SQL keys and channel names are always built through helpers**, never
  string-formatted at the call site, with the prefix as a single shared constant
  (`crates/buzz-pubsub/src/topic.rs:13`). Paired parse/format functions sit adjacent and are
  round-trip tested.
- **Every key-building path takes `&TenantContext` rather than a raw `CommunityId`**, so a
  caller cannot fabricate a tenant. This is the most consistently applied convention in the
  service layer.
- **"Never let one bad message kill a loop"** — every Redis subscriber handles malformed
  input with `warn` + `continue` (`crates/buzz-pubsub/src/subscriber.rs:132-157` and
  mirrors).
- **Contract obligations written into telemetry, not just docs** — e.g. "caller MUST fail
  closed" appears in the `warn!` payloads themselves
  (`crates/buzz-pubsub/src/nip98_replay.rs:55`, `:76`).
- **Known-limitation callouts are inline** with a `⚠️` marker and a stated upgrade path
  (`crates/buzz-pubsub/src/rate_limiter.rs:9-10`), rather than tracked in an issue.

Testing-convention regressions in 2b:

- **`buzz-db`: 121 of 122 async tests are `#[ignore]`d**, so the default gate exercises
  essentially none of the data layer. Combined with runtime `sqlx::query()` (no compile-time
  validation), the layer has neither static nor default dynamic verification.
- **`buzz-pubsub`: 11 of 34 tests are `#[ignore = "requires Redis"]`** — including the two
  most valuable ones (cross-community topic isolation, replay-TTL clamping).
- **`crates/buzz-pubsub/src/rate_limiter.rs` has zero tests** despite being a live security
  control; the relay's tests substitute a stub that never touches Redis
  (`crates/buzz-relay/src/admission.rs:65-90`).
- Fixture duplication is common: `buzz-pubsub` defines the same three-line `ctx()` helper
  six times and duplicates `test_presence_set_and_get` across two files
  (`src/lib.rs:477`, `src/presence.rs:138`).
- Still **no property-based tests anywhere** in 2a or 2b.

### Batch 2c conventions (relay, mesh, conformance)

- **`AppState`-threading is the relay's dominant convention** — handlers take
  `State<Arc<AppState>>` and reach subsystems through it rather than receiving narrow
  dependencies, which is why `crates/buzz-relay/src/state.rs` is the file every feature
  touches.
- **Host-derived tenancy is applied uniformly** across WebSocket, HTTP bridge, git, and
  Blossom paths — `TenantContext` is constructed once per request from the `Host` header
  (`crates/buzz-relay/src/tenant.rs`). The single deviation is `/_mesh/demo/echo`, which reads
  `community_id` from the request body.
- **No `unsafe` anywhere in this batch**, consistent with 2a/2b.
- **Conformance mirrors TLA+ naming character-for-character** — every `TraceAction` variant is
  named after its spec action (`crates/buzz-conformance/src/lib.rs:181-250` against
  `docs/spec/MultiTenantRelay.tla:514`, `:559`, `:606`, `:643`, `:681`, `:703`, `:778`, `:794`),
  with `ImplBug` (`src/lib.rs:256`) the one variant having no counterpart.
- **`Inv_*` names exist only in comments, never as Rust identifiers** — the mapping from
  invariant to enforcing predicate lives in prose
  (`crates/buzz-conformance/src/transitions.rs:53-54`, `:296-297`, `src/lib.rs:238`).
- **`M1`…`M8` mutation IDs are a second comment-only vocabulary with no legend in the repo** —
  used at `crates/buzz-conformance/src/lib.rs:127`, `:190`, `:238`;
  `src/transitions.rs:218-221`; `crates/buzz-relay/src/conformance/mod.rs:18-19`;
  `crates/buzz-relay/src/handlers/ingest.rs:1779`. They are inherited from an external
  mutation-testing plan that is not in the repo.
- **Named-reviewer and thread-hash comments couple source to conversations not in the repo**
  (`crates/buzz-relay/src/conformance/mod.rs:37-38`,
  `crates/buzz-conformance/tests/replay_fixtures.rs:19-20`), and several are now stale.
- **Test names encode the assertion, not the target** — `*_bites_*` for expected failures,
  `*_is_fine` / `*_passes` for expected successes
  (`crates/buzz-conformance/src/checker.rs:210`, `:228`, `:247`, `:290`).
- **Observability code never breaks the request** — the `Tracer` trait returns `()`
  (`crates/buzz-conformance/src/lib.rs:317`), `JsonlTracer` recovers from a poisoned mutex and
  swallows IO errors (`crates/buzz-relay/src/conformance/tracers.rs:59-71`), and the read seam
  logs `warn!` and continues on DB failure (`handlers/req.rs:347-353`, `:663-669`).

## Tooling-Enforced Conventions (scan-level)

| Convention | Enforcement | Config |
|---|---|---|
| Rust formatting | `cargo fmt --all -- --check` | `just fmt-check`, CI rust-lint |
| Rust linting | `cargo clippy --workspace --all-targets -- -D warnings` | `just clippy` |
| JS/TS lint + format | Biome | `biome.json`, `just desktop-check` / `web-check` |
| File-size ceiling | `check-file-sizes.mjs` (desktop, web, mobile) | `pnpm check:file-sizes`, `just mobile-check` |
| No arbitrary px/rem text sizes | `check-px-text.mjs` | `pnpm check:px-text` |
| Pubkey truncation policy | `check-pubkey-truncation.mjs` | `pnpm check:pubkey-truncation` |
| Dart format + analyze | `dart format --set-exit-if-changed`, `flutter analyze` | `just mobile-check` |
| Dependency policy | `cargo-deny check` | `deny.toml`, CI security job |
| Dead API-token guard | grep guard in CI | `ci.yml` dead-token-guard |
| Git hooks | lefthook (pre-commit fix + re-stage, pre-push clippy/tests) | `lefthook.yml` |

## Naming Conventions

_Pending Phase 2._

## Error Handling Patterns

_Pending Phase 2 (`thiserror` per-crate error enums, `anyhow` at binaries, `?`
propagation)._

## Testing Conventions

_Pending Phase 2 (unit tests in-crate, `#[ignore]` infra-dependent E2E, cargo-nextest,
Playwright projects `smoke`/`integration`, Flutter widget tests)._

## Architectural Conventions

_Pending Phase 2 (kind-first feature addition, `h`-tag channel scoping, community fencing,
module-level singleton resets on community switch)._

---

# Phase 2 — Module Findings

## Module: buzz-core (`crates/buzz-core`)

### Aspect: Conventions

---

### 1. Module organization

Flat module tree with one nested module. All 15 top-level modules are declared and documented inline in `src/lib.rs:9-38` — each `pub mod` carries a one-line `///` doc *above the declaration* rather than relying solely on the module's own `//!` header:

```
src/
  lib.rs                 crate root: lints, module decls, re-exports, test_helpers
  error.rs               VerificationError only (20 lines — smallest module)
  event.rs               StoredEvent
  verification.rs        verify_event
  filter.rs              NIP-01 matching + reader gate
  kind.rs                kind registry (784 lines, constants + predicates)
  network.rs             is_private_ip
  channel.rs             ChannelVisibility / ChannelType / MemberRole
  presence.rs            PresenceStatus
  relay.rs               normalize_relay_url
  tenant.rs              CommunityId / TenantContext / host normalization
  observer.rs            NIP-44 payload helpers + constants
  agent_turn_metric.rs   NIP-AM payload types (built on observer.rs)
  engram.rs              NIP-AE primitives
  git_perms.rs           git ref protection model
  pairing/
    mod.rs               re-exports + PairingError
    crypto.rs            HKDF derivations
    types.rs             wire messages
    qr.rs                nostrpair:// URI codec
    session.rs           state machine
    NIP-AB.md            spec text kept beside the code
    NIP-AB.spthy         Tamarin formal model kept beside the code
```

Convention: one cohesive concern per file; the only module that composes another is `agent_turn_metric.rs`, which reuses `observer.rs` crypto and error types (`agent_turn_metric.rs:12-15`), and `git_perms.rs`, which imports `channel::MemberRole` (`git_perms.rs:15`).

`pairing/mod.rs` is a facade: submodule declarations (`:22-25`), curated re-exports (`:27-29`), and the shared error enum (`:35-71`). It also documents its own layout in a markdown table inside the module doc comment (`pairing/mod.rs:15-20`).

---

### 2. Naming patterns

| Pattern | Convention as used | Examples |
|---|---|---|
| Event kind constants | `KIND_<DOMAIN>[_<DETAIL>]: u32`, grouped by protocol with section comments | `KIND_STREAM_MESSAGE_EDIT` (`kind.rs:423`), `KIND_NIP29_PUT_USER` (`kind.rs:275`) |
| Exception to that pattern | relay-admin commands use a `RELAY_ADMIN_` prefix instead of `KIND_` | `RELAY_ADMIN_ADD_MEMBER` (`kind.rs:329`) … `RELAY_ADMIN_SET_WORKSPACE_PROFILE` (`kind.rs:335`) |
| Kind sets | plural `*_KINDS: &[u32]` | `ALL_KINDS` (`kind.rs:566`), `P_GATED_KINDS` (`kind.rs:146`), `AUTHOR_ONLY_KINDS` (`kind.rs:120`), `RESULT_GATED_KINDS` (`kind.rs:129`) |
| Range bounds | `<NAME>_KIND_MIN` / `_MAX` | `EPHEMERAL_KIND_MIN/MAX` (`kind.rs:397-399`) |
| Predicates | `is_*(kind: u32) -> bool`, `const fn` wherever possible | `is_ephemeral`, `is_replaceable`, `is_relay_only_kind` (`kind.rs:697-769`) |
| Limits | `MAX_*` / `*_MAX` / `*_MIN` constants rather than inline literals | `MAX_PROTECTION_RULES` (`git_perms.rs:19`), `NIP44_MAX_CONTENT_LEN` (`observer.rs:23`), `SLUG_MAX_LEN` (`engram.rs:31`) |
| Enum → string | `as_str(&self) -> &'static str` plus `Display` delegating to it | `channel.rs:31-56`, `:72-105`, `:123-155`; `presence.rs:22-35` |
| String → enum | `FromStr` with `type Err = String` and a `format!` message | `channel.rs:44-53`, `:88-99`, `:163-179` |
| Constructors that assert provenance | verb-named, not `new` | `TenantContext::resolved` (`tenant.rs:79`), `CommunityId::from_uuid` (`tenant.rs:45`), `StoredEvent::with_received_at` (`event.rs:38`) |
| Crypto derivations | `derive_*` returning fixed-size arrays | `derive_session_id`, `derive_sas`, `derive_transcript_hash` (`pairing/crypto.rs:54-105`) |
| Protocol handlers | `handle_*` consumes an inbound event; `send_*`/`confirm_*` produces an outbound event | `pairing/session.rs:149`, `:227`, `:254`, `:329`, `:388`, `:412` |
| HKDF domain separation strings | `INFO_*` private consts | `pairing/crypto.rs:22-25` |
| Normalizers | `normalize_*` / `canonical_*` | `normalize_host` (`tenant.rs:121`), `normalize_relay_url` (`relay.rs:37`), `normalize_slug` (`engram.rs:123`), `canonical_channel_name` (`channel.rs:15`) |

---

### 3. Error handling

Every error type is an enum; five of seven use `thiserror`, two are hand-rolled.

| Error type | Style | Variants | file:line |
|---|---|---|---|
| `VerificationError` | `thiserror` | 3 (incl. `#[from] nostr::secp256k1::Error`) | `error.rs:2-20` |
| `EngramError` | `thiserror` | 7 | `engram.rs:37-59` |
| `ObserverPayloadError` | `thiserror` | 5 (2 `#[from]`) | `observer.rs:23-46` |
| `PairingError` | `thiserror` | 10 (2 `#[from]`) | `pairing/mod.rs:34-71` |
| `NormalizeRelayUrlError` | `thiserror` + `PartialEq, Eq` (so tests can compare) | 5 | `relay.rs:7-24` |
| `PatternError` | manual `Display` + `impl std::error::Error` | 5 | `git_perms.rs:52-82` |
| `RuleParseError` | manual `Display` + `impl std::error::Error` | 5 (one wraps `PatternError`) | `git_perms.rs:264-301` |
| `Denial` | not an `Error` impl — a value type with `Display` | struct | `git_perms.rs:479-490` |
| `FromStr` errors on channel enums | plain `String` | — | `channel.rs:45`, `:89`, `:164` |

Conventions observed:
- Fallible operations return `Result<_, E>`; no panicking public API.
- No `unwrap()`/`expect()` in production paths **except two documented infallibility cases**, both with a justification comment: `ConversationKey::derive(...).expect("valid keys produce conversation key")` (`engram.rs:137`) and the HMAC key-length `expect` (`engram.rs:147-149`, justified at `:145-147`). Every other `unwrap`/`expect` in the crate is inside `#[cfg(test)]` code or the `test-utils`-gated helpers (`lib.rs:59`, `:68`).
- Error messages embed the offending value with `{x:?}` for round-trippable diagnostics (`channel.rs:52`, `engram.rs:79`, `git_perms.rs:76`).
- Guard-clause style: validation functions early-`return Err(...)` rather than nesting (e.g. `git_perms.rs:84-92`, `qr.rs:105-181`, `session.rs:150-152`).
- Fail-closed defaults are stated in doc comments where a caller must cooperate (`tenant.rs:117-119`, `tenant.rs:151`).

---

### 4. Doc-comment practice

- `#![warn(missing_docs)]` at `lib.rs:2` — every public item carries a doc comment; spot-checking the pub-item inventory found no undocumented public item.
- Module headers use `//!` with a short summary and, where a protocol is involved, an ASCII diagram: derivation flow (`pairing/crypto.rs:12-28`), protocol sequence (`pairing/session.rs:19-40`), permission pipeline (`git_perms.rs:11-16`).
- Private fields and private helpers are also documented (e.g. `PatternSegment` variants `git_perms.rs:41-49`, `PairingSession` fields `session.rs:86-112`).
- Doc comments carry the *rationale*, not just the description — several are effectively design records: the tenant "fence" (`tenant.rs:17-30`), why `normalize_relay_url` is not the AUTH comparator (`relay.rs:28-32`), why `require_patch` blocks all update kinds (`git_perms.rs:254-258`), why the `Unknown` abort reason must not be sent outbound (`pairing/types.rs:95-98`), residual un-zeroizable copies (`session.rs:551-559`).
- Spec citations are inline and section-level: `NIP-AB §Duplicate Event Handling` (`session.rs:530-533`), `NIP-AM §Numeric validity` (`agent_turn_metric.rs:138`), `NIP-AE *Head selection* rule (3)` (`engram.rs:222-224`), `RFC 6598` (`network.rs:32`).
- Cross-references use rustdoc links (`[`StoredEvent`]`, `[`crate::engram`]`) — e.g. `lib.rs:5-6`, `kind.rs:97-98`.
- Two doctests exist (both in pairing): `format_sas` (`pairing/crypto.rs:110-114`) and `encode_qr` (`pairing/qr.rs:66-77`).

---

### 5. Testing patterns

All tests are inline `#[cfg(test)] mod tests` blocks — there is **no `tests/` directory** and no proptest/quickcheck usage anywhere in the crate (searched: no `proptest`, no `quickcheck` in `src/` or `Cargo.toml`).

Static count of `#[test]` functions per file (grep of `#[test]` attributes):

| File | `#[test]` fns | Test module starts at |
|---|---|---|
| `src/network.rs` | 35 (was 29; `c26bf594` added 6 SSRF prefix tests) | `:97` (was `:56`) |
| `src/engram.rs` | 34 | `:607` |
| `src/git_perms.rs` | 34 | `:601` |
| `src/pairing/qr.rs` | 27 | `:245` |
| `src/pairing/session.rs` | 18 | `:756` (a separate `#[cfg(test)] impl` block sits at `:530`) |
| `src/kind.rs` | 14 (was 4; `07d0265c` added 10 persona-visibility tests) | `:823` (was `:747`) |
| `src/pairing/crypto.rs` | 14 | `:131` |
| `src/agent_turn_metric.rs` | 14 | `:194` |
| `src/pairing/types.rs` | 10 | `:98` |
| `src/tenant.rs` | 10 | `:175` |
| `src/filter.rs` | 6 | `:106` |
| `src/presence.rs` | 4 | `:37` |
| `src/relay.rs` | 3 | `:80` |
| `src/observer.rs` | 2 | `:113` |
| `src/verification.rs` | 2 | `:34` |
| `src/channel.rs` | 1 | `:181` |
| `src/event.rs` | 1 | `:53` |
| `src/error.rs`, `src/lib.rs`, `src/pairing/mod.rs` | 0 | — |
| **Total** | **229** (was **213** at `b8510ede`; +6 `network.rs`, +10 `kind.rs`) | |

(Counts are static — obtained by reading the source. The crate was not compiled during this analysis: `cargo` is not on the PATH in this environment without the repo's Hermit activation, so no test-run count is claimed.)

Recurring test conventions:

| Pattern | Example |
|---|---|
| Shared fixture fns at the top of the test module | `make_event()` (`event.rs:57-63`), `stored_with_tag()` (`filter.rs:113-121`), `sample_payload()` (`agent_turn_metric.rs:199-228`), `make_payload_with_turn_cost()` (`:351`), `make_payload_with_cumulative_cost()` (`:374`), `make_payload()` (`qr.rs:250-258`), `keys_from_hex()` (`engram.rs:611-613`) |
| Pinned spec test vectors as consts | `engram.rs:617-627` (SECKEY/PUBKEY/K_C/D_* vectors), `pairing/crypto.rs:136-155` (session secret + ephemeral key bytes) |
| Single test asserting a whole vector suite | `all_test_vectors` (`pairing/crypto.rs:272-323`), `d_tags_match_spec` (`engram.rs:646-654`) |
| Exhaustive-range invariant tests | `replaceable_and_parameterized_are_disjoint` loops `0..=65535` (`kind.rs:852-859`, message at `:856`) |
| Boundary-pair tests (just-inside / just-outside) | CGNAT and benchmarking range tests (`network.rs:299-339`), and — added in `c26bf594` — the NAT64 local-use / Teredo / 6to4 prefix-boundary tests (`network.rs:254-297`), `parameterized_replaceable_range` (`kind.rs:842-850`) |
| Round-trip serde tests per wire type | `pairing/types.rs:102-241`, `presence.rs:41-53`, `agent_turn_metric.rs:230-251` |
| Regression tests labelled as such with the scenario in a comment | `filter.rs:216-236` (explicit h-tag authority), `git_perms.rs:926-941` (guest bypass), `agent_turn_metric.rs:476-507` (validation bypass via lower-level encrypt) |
| Negative/anti-property tests | `reject_*` naming across `qr.rs` (12 `reject_*` tests) and `session.rs` (`reject_out_of_order_operations`, `reject_invalid_session_id`, `reject_event_from_wrong_pubkey`, …) |
| Full-protocol happy-path test driving both peers in one process | `happy_path_full_protocol` (`session.rs:762`) |
| Test-only inherent methods behind `#[cfg(test)]` instead of widening the API | `session.rs:530-544` (`has_processed`, `set_timeout`) |
| Compile-time tests | `const _: () = assert!(...)` block, `kind.rs:783-820` (25 assertions) |
| `assert!` messages explaining the invariant, not just the failure | `filter.rs:234`, `kind.rs:856`, `session.rs:782` |

---

### 6. Other code conventions

- `const fn` is used wherever the computation allows it (`kind.rs:316`, `:697-769`; `tenant.rs:45`, `:50`, `:87`; `engram.rs:114`).
- `#[must_use]` on the two pure string transformers (`tenant.rs:120` for `normalize_host`, `:155` for `relay_url_authority`).
- All kind integers are `u32` with a stated rationale (NIP-01 unsigned integer, u32 covers the range) at `kind.rs:3-5`; conversion to `u16`/`i32` is centralized in `event_kind_u32`/`event_kind_i32` (`kind.rs:772-780`) rather than scattered casts.
- Secret material is handled with a consistent trio: `Zeroizing<String>` in signatures, explicit `.zeroize()` after use, and `Drop` impls (`pairing/session.rs:227`, `:573`, `:731-739`; `pairing/qr.rs:56-60`; `observer.rs:66-109`).
- Hand-rolled serialization is used only where byte-exactness is a spec requirement, with the reason stated (`engram.rs:194-197`, `:261-262`).
- Newtype + private field + accessor is the standard way this crate expresses "provenance matters" (`CommunityId`, `TenantContext`, `RefPattern`, `StoredEvent.verified`).


## Module: buzz-sdk (`crates/buzz-sdk`)

### Aspect: Conventions

---

### 1. Module organization

| File | LOC | Role |
|---|---|---|
| `crates/buzz-sdk/src/lib.rs` | 112 | crate docs, lints, module declarations, shared input types, `SdkError`, `buzz-core` re-exports |
| `crates/buzz-sdk/src/builders.rs` | 3 824 | all 51 event builders + private validation helpers + 162 tests |
| `crates/buzz-sdk/src/mentions.rs` | 820 | pure mention-resolution helpers + 51 tests |
| `crates/buzz-sdk/src/nip_oa.rs` | 595 | NIP-OA auth-tag compute/verify/parse + 22 tests |
| `crates/buzz-sdk/examples/compute_auth_tag.rs` | 29 | example binary wrapping `nip_oa::compute_auth_tag` |

Flat two-level layout — no submodule directories. Types shared by more than one
builder live in `lib.rs`; types used by a single builder family live next to
that family in `builders.rs` (e.g. `GitPatchMeta` at `builders.rs:985`,
`DeleteMessageOptions` at `builders.rs:393`).

`pub use builders::*` (`lib.rs:19`) flattens the builder namespace so consumers
write `buzz_sdk::build_message`, while `mentions` and `nip_oa` stay
namespace-qualified (`lib.rs:15-17`).

Section banners with `// ---` rules mark thematic groups inside `builders.rs`
(`builders.rs:1587-1595` for moderation, `1692-1701` for NIP-IA,
`builders.rs:3286` and `3577` for test groups using `// ──` box-drawing rules).

---

### 2. Naming patterns

| Pattern | Convention | Examples |
|---|---|---|
| Event builders | `build_<operation>` | `build_message`, `build_forum_post`, `build_git_pr_update` (`builders.rs:219`, `278`, `1416`) |
| Variant builders | base name + `_with_<what>` suffix | `build_delete_message_with_options` (`builders.rs:411`), `build_repo_announcement_with_tags` (`builders.rs:952`) |
| Compatibility shims | `_compat` suffix | `build_delete_compat` (kind 5 vs native 9005) (`builders.rs:434`) |
| Private validators | `check_<subject>` returning `Result<(), SdkError>` or `Result<String, SdkError>` | `check_content`, `check_hex_len`, `check_commit_hex`, `check_pubkey_hex`, `check_hex_exact`, `check_repo_id`, `check_custom_emoji_url`, `check_reason`, `check_auth_tag_shape` (`builders.rs:35-170`, `1706-1737`) |
| Public normalizers | `normalize_<subject>` | `normalize_custom_emoji_shortcode` (`builders.rs:127`), `normalize_mention_pubkeys` (`mentions.rs:228`) |
| Tag emitters | `<subject>_tags(…, tags: &mut Vec<Tag>)` | `thread_tags`, `mention_tags`, `imeta_tags`, `identity_archive_tags` (`builders.rs:173`, `188`, `203`, `1739`) |
| Extractors | `extract_<subject>` | `extract_channel_id` (`builders.rs:816`), `extract_at_names`, `extract_nostr_uris` (`mentions.rs:64`, `353`) |
| Input bundles | `<Domain><Thing>Meta` / `Options` | `GitPatchMeta`, `GitIssueMeta`, `GitStatusMeta`, `GitPullRequestMeta`, `GitPrUpdateMeta`, `DeleteMessageOptions` |
| Constants | SCREAMING_SNAKE with unit in the name where relevant | `MENTION_CAP` (`mentions.rs:38`), `MAX_CONTACTS` (`builders.rs:751`), `MAX_REASON_BYTES` (`builders.rs:1704`), `CUSTOM_EMOJI_SET_D_TAG` (`builders.rs:503`) |
| `nip_oa` functions | verb-first, NIP-term nouns | `compute_auth_tag`, `verify_auth_tag`, `parse_auth_tag` |

Local shorthand inside tests: `ev`, `cid`, `eid`, `pk`, `wid`, `rid`
(`builders.rs:1958-1975`).

---

### 3. Builder-pattern style

There is **no fluent/typestate builder struct**. The style is:

1. Free function taking positional parameters, plus one `&Meta` struct when the
   optional-tag count grows large (Git family, `builders.rs:1013`, `1081`,
   `1222`, `1330`, `1416`).
2. Validate everything first, returning early on error.
3. Accumulate `Vec<Tag>` in wire order — `h`/`a`/`d` first, then optional tags.
4. Return `EventBuilder::new(Kind::Custom(n), content).tags(tags)` as the final
   expression. Signing is deliberately excluded (`lib.rs:11-13`).

Optionality is expressed through the type system rather than setters:
`Option<&str>` for optional strings, `Option<MemberRole>` for enums,
`Option<Option<i32>>` for the tri-state TTL (`builders.rs:604-609`), `bool` flags
for presence-only tags (`broadcast`, `truncated`, `root`, `root_revision`).

`#[derive(Default)]` on the five Git meta structs enables `..Default::default()`
partial initialization, which is how tests and the CLI construct them
(`builders.rs:984`, `1072`, `1200`, `1302`, `1396`).

Every kind integer reaches nostr through `Kind::Custom(... as u16)` — 26 sites
use `buzz_core::kind::KIND_*` constants (`builders.rs:6-19`), the rest use bare
literals.

---

### 4. Error handling

- Single crate-level error enum, `SdkError`, derived with `thiserror::Error`
  (`lib.rs:87-113`); six variants, three of which carry a free-form `String`
  message (`InvalidTag`, `InvalidDiffMeta`, `InvalidInput`) and one structured
  (`ContentTooLarge { max, got }`).
- Uniform return type: every fallible public function returns
  `Result<_, SdkError>`. There is no `Box<dyn Error>`, no `anyhow`, and no `From`
  impl chain — third-party errors are stringified at the boundary
  (`builders.rs:31`, `1372`; `nip_oa.rs:126`, `209`, `220`, `242`).
- **No `unwrap()`/`expect()` in non-test code** (verified by scan of `src/*.rs`
  outside `#[cfg(test)]`). The one near-miss is
  `parts.next().unwrap_or_default()` in `GitAppliedPatchRef::parse`
  (`builders.rs:1145`), which cannot panic.
- Guard-clause style: validation blocks sit at the top of each function and
  `return Err(...)` immediately; the tag-building section is unconditional
  afterwards (e.g. `builders.rs:314-340`, `840-919`).
- Error messages embed the offending value or measured size for diagnosis:
  `"{field} must be at least {min_len} hex characters (got {:?})"`
  (`builders.rs:47-49`), `"repo_id exceeds 64 characters (got {})"`
  (`builders.rs:97-100`).
- The example binary is the only place that panics on bad input, via `.expect()`
  on argument parsing (`examples/compute_auth_tag.rs:21-27`).

---

### 5. Doc-comment practice

- `#![warn(missing_docs)]` (`lib.rs:2`) — every public item is documented,
  including struct fields (e.g. `DiffMeta`'s ten fields each carry a `///`
  line, `lib.rs:37-56`) and enum variants (`lib.rs:62-65`).
- Crate doc opens with an ASCII "Mental Model" pipeline diagram
  (`lib.rs:5-13`); `mentions.rs:1-30` and `nip_oa.rs:1-19` do the same for their
  modules (pipeline diagram and tag-format/preimage blocks respectively).
- Builder docs state the kind number in prose ("Build a stream message (kind 9)",
  `builders.rs:211`) and enumerate parameters as a bullet list when there are
  more than three (`builders.rs:211-218`, `1465-1469`).
- Rationale comments cite the governing NIP and, frequently, the exact
  cross-repo file being mirrored: `desktop/src-tauri/src/events.rs:624-743`
  (`builders.rs:1700`), `desktop/src-tauri/src/events.rs:635-647`
  (`builders.rs:1702-1704`), `moderation_commands.rs` (`builders.rs:1593-1594`),
  `NIP-IA.md §Vector 1` (`builders.rs:3625`).
- Deviations from a NIP are called out inline rather than hidden — e.g. "The `h`
  tag is non-standard for NIP-09 but is required so channel-scoped subscriptions
  observe the delete." (`builders.rs:432-433`).
- `# Errors` sections appear in `nip_oa` public functions
  (`nip_oa.rs:140-144`, `173-177`, `245-250`).

---

### 6. Testing patterns

| Aspect | Convention | Evidence |
|---|---|---|
| Location | one `#[cfg(test)] mod tests` per source file; no `tests/` directory in the crate | `builders.rs:1825`, `mentions.rs:389`, `nip_oa.rs:301` |
| Volume | 235 `#[test]` functions total: 162 in `builders.rs`, 51 in `mentions.rs`, 22 in `nip_oa.rs` | counted per file |
| Framework | stock `#[test]` only — no proptest, no rstest, no async tests | no `proptest`/`quickcheck` reference anywhere in the crate |
| Test helpers | small fixture fns at the top of the module: `keys()`, `sign()`, `event_id()`, `uuid()`, `tag_values()`, `has_tag()` | `builders.rs:1830-1873` |
| Domain fixtures | per-family fixture fns: `good_diff_meta()` (`builders.rs:2054`), `pr_repo()` (`builders.rs:3396`), `full_clone_tag()` (`builders.rs:3403`), `profile()` (`mentions.rs:545`) |
| Naming | `<subject>_<scenario>`: `*_happy_path`, `*_minimal`, `*_all_fields`, `*_rejects_*`, `*_too_large` | `builders.rs:1874`, `2821`, `2145`, `2872` |
| Assertion style | sign the builder, then assert on `ev.kind.as_u16()`, `ev.content`, and tag presence via `has_tag`/`tag_values`; positional tag layout asserted where wire order matters | `builders.rs:3707-3735`, `3744-3771` |
| Error assertions | `assert!(matches!(result, Err(SdkError::Variant { .. })))` | `builders.rs:2080-2087`, `2177-2185` |
| Boundary pairs | max-allowed and max+1 tested together | `builders.rs:2009-2020` (64 KiB), `builders.rs:2260-2273` (64-char emoji), `builders.rs:3773-3788` (64-byte reason) |
| Regression comments | tests carry a comment naming the bug or cross-component contract they pin | `builders.rs:3101-3105` (whitespace-only patch), `builders.rs:3613-3615` (relay `extract_p_tag_bytes`), `builders.rs:3688-3690` (relay `extract_report_tag`) |
| Spec vectors | NIP-OA spec pubkeys/conditions/signature and NIP-IA target/owner hex are hoisted into module `const`s and asserted against | `nip_oa.rs:303-310`, `builders.rs:3701-3705` |
| Unicode/panic-safety tests | explicit non-panic tests for multi-byte input | `mentions.rs:520-531`, `mentions.rs:787-800` |
| Cross-client pinning | test comments state that the assertions mirror `desktop/src-tauri/src/events.rs`'s own tests so both clients stay on one wire form | `builders.rs:3696-3699` |


## Module: buzz-persona (`crates/buzz-persona`)

### Aspect: Conventions

### Module organization

`crates/buzz-persona/src/lib.rs:1-6` declares six flat public modules with no re-exports
and no crate-level `//!` doc. The layout mirrors a linear pipeline, one module per stage:

| Module | Role | LOC |
|---|---|---|
| `persona` | `.persona.md` parse (leaf — no intra-crate deps) | 645 |
| `manifest` | `plugin.json` parse (imports `persona::RespondTo` at `crates/buzz-persona/src/manifest.rs:20`) | 365 |
| `merge` | precedence resolution over `serde_json::Value` (leaf — no intra-crate deps) | 464 |
| `pack` | directory loader; composes `manifest` + `persona` + `merge` (`crates/buzz-persona/src/pack.rs:22-24`) | 734 |
| `resolve` | final projection; composes `merge` + `pack` + `persona` (`crates/buzz-persona/src/resolve.rs:18-21`) | 892 |
| `validate` | diagnostics layer over `pack` (`crates/buzz-persona/src/validate.rs:15`) | 1070 |

Dependency direction is strictly one-way (`persona`/`merge` → `manifest` → `pack` →
`resolve`/`validate`); no cycles.

Each stage owns a distinct type family, and types are **re-declared rather than reused**
across stages: `RespondTo` (serde-facing, `crates/buzz-persona/src/persona.rs:53`) →
`TriggersData` (merged, `crates/buzz-persona/src/merge.rs:11`) → `ResolvedTriggers`
(output, `crates/buzz-persona/src/resolve.rs:86`); likewise `Hooks` → `HooksData` →
`ResolvedHooks` and `PersonaConfig` → `LoadedPersona` → `ResolvedPersona`.

### Naming patterns

| Pattern | Examples |
|---|---|
| `parse_*` for string/file → typed | `parse_persona_md`, `parse_persona_file` (`crates/buzz-persona/src/persona.rs:208`, `:262`), `parse_manifest`, `parse_manifest_file` (`crates/buzz-persona/src/manifest.rs:152`, `:190`), `parse_triggers` (`crates/buzz-persona/src/merge.rs:177`), `parse_mcp_server_config` (`crates/buzz-persona/src/resolve.rs:311`) |
| `load_*` for directory → aggregate | `load_pack` (`crates/buzz-persona/src/pack.rs:125`) |
| `resolve_*` for merge/projection | `resolve_persona_config` (`crates/buzz-persona/src/merge.rs:85`), `resolve_pack`, `resolve_loaded_pack`, `resolve_persona_by_name`, `resolve_one_persona`, `resolve_triggers`, `resolve_hooks` (`crates/buzz-persona/src/resolve.rs:108`–`:357`), `resolve_skills` (`crates/buzz-persona/src/pack.rs:249`) |
| `merge_*` for two-sided combination | `merge_behavioral_config` (`crates/buzz-persona/src/merge.rs:47`), `merge_mcp_servers` (`crates/buzz-persona/src/resolve.rs:277`) |
| `split_*` for string decomposition | `split_frontmatter` (`crates/buzz-persona/src/persona.rs:277`), `split_model` (`:324`) |
| `validate_*` public / `*_check_*` private | `validate_pack` (`crates/buzz-persona/src/validate.rs:143`), `validate_persona_name` (`:167`); `semantic_check_personas` (`:187`), `advisory_check_manifest_keys` (`:302`), `advisory_check_respond_to_types` (`:210`), `advisory_check_skill_names` (`:354`) |
| `Loaded*` / `Resolved*` type prefixes marking pipeline stage | `LoadedPack`, `LoadedPersona`, `PackManifestData`; `ResolvedConfig`, `ResolvedPack`, `ResolvedPersona`, `ResolvedMcpServer`, `ResolvedHooks`, `ResolvedTriggers` |
| `*Data` suffix for plain (non-serde) carriers | `TriggersData`, `HooksData` (`crates/buzz-persona/src/merge.rs:11`, `:18`), `PackManifestData` (`crates/buzz-persona/src/pack.rs:102`) |
| `Raw*` / bare shadow structs for permissive deserialization | `RawManifest` (`crates/buzz-persona/src/manifest.rs:132`), `Frontmatter` (`crates/buzz-persona/src/persona.rs:176`) |
| `MAX_*` / `DEFAULT_*` / `KNOWN_*` constant prefixes | `MAX_FRONTMATTER_BYTES`, `MAX_BODY_BYTES` (`crates/buzz-persona/src/persona.rs:21`, `:24`), `MAX_NAME_LEN` (`crates/buzz-persona/src/validate.rs:168`), `DEFAULT_THREAD_REPLIES`/`DEFAULT_BROADCAST_REPLIES` (`crates/buzz-persona/src/merge.rs:38-39`), `KNOWN_MANIFEST_KEYS`/`KNOWN_BEHAVIORAL_KEYS`/`KNOWN_RESPOND_TO_KEYS` (`crates/buzz-persona/src/validate.rs:99`, `:121`, `:133`) |

Note the one naming collision: private `pack::parse_persona_file`
(`crates/buzz-persona/src/pack.rs:392`) shares a name with public
`persona::parse_persona_file` (`crates/buzz-persona/src/persona.rs:262`) while having a
different signature and return type.

### Error handling

Three `thiserror`-derived enums, one per boundary, with no shared error type and no
`anyhow`:

| Enum | Location | Variant count | `#[from]` conversions |
|---|---|---|---|
| `PersonaError` | `crates/buzz-persona/src/persona.rs:26-48` | 7 | `std::io::Error` (`:29`), `serde_yaml::Error` (`:44`) |
| `ManifestError` | `crates/buzz-persona/src/manifest.rs:22-31` | 3 | `std::io::Error` (`:26`), `serde_json::Error` (`:29`) |
| `PackError` | `crates/buzz-persona/src/pack.rs:25-54` | 8 | none automatic; manual `From<ManifestError>` at `:56-60` |

Conventions observed:

- Error messages are lowercase, no trailing period, with the offending value interpolated:
  `"path traversal rejected: {0}"` (`crates/buzz-persona/src/pack.rs:46`),
  `"missing required field: {0}"` (`crates/buzz-persona/src/persona.rs:46`).
- Constants are interpolated into `#[error]` strings:
  `#[error("frontmatter exceeds {MAX_FRONTMATTER_BYTES} bytes")]`
  (`crates/buzz-persona/src/persona.rs:34`).
- Structured variants carry a `path: PathBuf` plus a `reason: String` for file-scoped
  faults (`PackError::Io`, `FileParse`, `McpConfigParse` —
  `crates/buzz-persona/src/pack.rs:30-53`), and `#[source]` is used to preserve the IO
  cause chain (`:33`).
- Required-field checks are done manually against an `Option`-shaped shadow struct so the
  error is a clean `MissingField` instead of a serde path error — rationale stated at
  `crates/buzz-persona/src/manifest.rs:123-125` and `crates/buzz-persona/src/persona.rs:172-173`.
- Cross-boundary errors are **flattened to strings**, losing the source chain:
  `PackError::ManifestParse(e.to_string())` (`crates/buzz-persona/src/pack.rs:58`) and
  `.map_err(|e| PackError::FileParse { reason: e.to_string(), .. })`
  (`crates/buzz-persona/src/pack.rs:393-396`).
- `validate_pack` never returns `Err` — it converts all failures into
  `ValidationDiagnostic::Error` strings (`crates/buzz-persona/src/validate.rs:145-152`).
- No `unwrap()` / `expect()` / `panic!` in any non-test code path (verified by grep — all
  matches are inside `#[cfg(test)]` modules or `tests/`). No `unsafe` anywhere.
- Silent-drop is a deliberate recurring pattern for malformed sub-values:
  `filter_map(...)` on MCP entries (`crates/buzz-persona/src/pack.rs:415-419`,
  `crates/buzz-persona/src/resolve.rs:294-299`), `?` on missing `command`
  (`crates/buzz-persona/src/resolve.rs:312`), `.ok()?`/`continue` chains in the skill
  advisory check (`crates/buzz-persona/src/validate.rs:392-418`).

### Doc-comment practice

- Every module opens with a `//!` header **except** `pack.rs` and `merge.rs`, which use
  `///` on the first item instead — `crates/buzz-persona/src/pack.rs:1-19` and
  `crates/buzz-persona/src/merge.rs:1-9` are outer doc comments attached to the following
  item rather than module-level docs. `lib.rs` has no doc at all.
- Module docs carry a literal example: a `.persona.md` sample in
  `crates/buzz-persona/src/persona.rs:5-14`, a `plugin.json` sample in
  `crates/buzz-persona/src/manifest.rs:8-15`, a directory tree in
  `crates/buzz-persona/src/pack.rs:6-16`.
- Public functions document semantics, and `load_pack` documents its algorithm as a
  numbered list matching in-body step comments (`crates/buzz-persona/src/pack.rs:117-124`
  vs. the `// 1.` … `// 6.` markers at `:131`, `:151`, `:166`, `:190`, `:222`).
- Non-obvious decisions get a rationale block rather than a bare statement — e.g. the
  three-step defense-in-depth explanation on `safe_resolve`
  (`crates/buzz-persona/src/pack.rs:317-322`), the security note on `resolve_hooks`
  (`crates/buzz-persona/src/resolve.rs:339-346`), and the permissiveness rationale on
  `RawManifest` (`crates/buzz-persona/src/manifest.rs:123-130`).
- Tri-state field semantics are documented on the field itself with a bullet list —
  `PersonaConfig::subscribe` (`crates/buzz-persona/src/persona.rs:129-134`),
  `ResolvedConfig::subscribe` (`crates/buzz-persona/src/merge.rs:29-31`).
- `# Limits` is used as a doc heading on `parse_persona_md`
  (`crates/buzz-persona/src/persona.rs:205-207`).
- No `#[doc(hidden)]`, no `#![deny(missing_docs)]`, no doctests (all fenced blocks are
  ```text` or ```json`, e.g. `crates/buzz-persona/src/persona.rs:5`,
  `crates/buzz-persona/src/manifest.rs:8`).

### Testing patterns

| Convention | Evidence |
|---|---|
| Per-module `#[cfg(test)] mod tests { use super::*; }` | `crates/buzz-persona/src/manifest.rs:195-197`, `merge.rs:200-202`, `pack.rs:447-449`, `persona.rs:331-333`, `resolve.rs:400-404`, `validate.rs:436-438` |
| Test counts | `persona.rs` 29, `resolve.rs` 26, `merge.rs` 22, `validate.rs` 22, `manifest.rs` 14, `pack.rs` 14; `tests/integration.rs` 13, `tests/e2e_env_flow.rs` 5 — **145 total** |
| Descriptive snake_case test names encoding the expectation | `unknown_frontmatter_keys_error`, `closing_delimiter_with_trailing_junk_is_not_valid` (`crates/buzz-persona/src/persona.rs:403`, `:483`), `triggers_shallow_replacement` (`crates/buzz-persona/src/merge.rs:352`) |
| Fixture builders instead of inline setup | `minimal()` (`crates/buzz-persona/src/persona.rs:334`), `minimal_json()` (`crates/buzz-persona/src/manifest.rs:198`), `make_pack()` (`crates/buzz-persona/src/pack.rs:451`), `make_loaded_persona()` (`crates/buzz-persona/src/pack.rs:609`), `stub_persona()` (`crates/buzz-persona/src/resolve.rs:864`), `create_test_pack()` (`crates/buzz-persona/tests/integration.rs:14`) |
| Named fixture constant for the canonical persona | `SIMPLE_PERSONA` (`crates/buzz-persona/src/pack.rs:481-487`) |
| `tempfile::TempDir` / `tempfile::tempdir()` for on-disk pack fixtures | `crates/buzz-persona/src/pack.rs:450`, `crates/buzz-persona/src/validate.rs:462` |
| Error assertions via `matches!` on the variant, with the error echoed in the failure message | `assert!(matches!(&err, PersonaError::MissingField(f) if f == "name"), "got: {err}")` — `crates/buzz-persona/src/persona.rs:433-437` |
| Assertion messages state the *rule*, not the mechanics | `"pack keywords should be lost under shallow replacement"` (`crates/buzz-persona/src/merge.rs:361-364`); `"built-in default for mentions is true"` (`:379`) |
| Regression tests annotated with the bug they lock in | `crates/buzz-persona/src/merge.rs:400-403` — "Critical regression test … This was broken when BehavioralDefaults serialized the field as \"respond_to\" but merge looked for \"triggers\""; `crates/buzz-persona/tests/integration.rs:158-159` |
| Platform-gated tests | `#[cfg(unix)] #[test] fn symlink_escape_rejected` (`crates/buzz-persona/src/pack.rs:589-607`) |
| Integration tests exercise the documented pipeline end-to-end and are described in a module doc | `crates/buzz-persona/tests/integration.rs:1-5`, `crates/buzz-persona/tests/e2e_env_flow.rs:1-9` |
| Table-driven loop for a family of rejections | `for field in ["idle_timeout", "max_turn_duration", ...]` — `crates/buzz-persona/tests/integration.rs:637-650` |
| Local test helper for readable multiline fixtures | `fn indoc(s: &str) -> &str` (`crates/buzz-persona/src/persona.rs:642-644`) — hand-rolled rather than pulling the `indoc` crate |

No property-based testing, no snapshot testing, no mocking framework, no `#[should_panic]`.

### Asset embedding

There is none. No `include_str!` / `include_dir!` / `include_bytes!`, no `build.rs`, no
`[build-dependencies]`, no asset directory (`crates/buzz-persona/Cargo.toml:1-18` and the
full file listing confirm this). Personas are read from caller-supplied directories at
runtime via `std::fs`. The only bundled non-Rust file is
`crates/buzz-persona/PERSONA_PACK_SPEC.md`, which is documentation and not compiled in.

The convention for "packaged" data is therefore **path-relative directory conventions**
rather than embedding: `.plugin/plugin.json`, `instructions.md`, `.mcp.json`, `skills/`
are located by joining onto the canonicalized pack root
(`crates/buzz-persona/src/pack.rs:132`, `:183`, `:209`, `:224`).

### Other crate-level conventions

- Dependencies are pinned inline with explicit versions rather than
  `{ workspace = true }` (`crates/buzz-persona/Cargo.toml:10-14`), and the package
  declares its own `version`, `edition`, `license`, `repository` rather than inheriting
  from the workspace (`crates/buzz-persona/Cargo.toml:1-8`) — the opposite of the pattern
  in e.g. `crates/buzz-acp/Cargo.toml:1-8`. The declared `repository` is
  `https://github.com/block/sprout` (`crates/buzz-persona/Cargo.toml:7`), the pre-rename
  repo name.
- `edition = "2021"` (`crates/buzz-persona/Cargo.toml:4`) with no `rust-version` key.
- Import style: `use std::...` first, blank line, then external crates, then `use crate::...`
  (`crates/buzz-persona/src/pack.rs:20-24`, `crates/buzz-persona/src/resolve.rs:16-21`).
  `persona.rs` and `manifest.rs` follow the same order (`crates/buzz-persona/src/persona.rs:15-19`).
- Fully-qualified inline paths are used in places instead of top-level imports
  (`std::collections::HashSet::new()` at `crates/buzz-persona/src/pack.rs:264`,
  `std::path::Path::new` at `:254`), while the same types are imported at the top of other
  modules — the style is not uniform.
- Nested helper functions are used where a helper is single-use: `normalize_skill_name`
  declared inside `resolve_skills` (`crates/buzz-persona/src/pack.rs:253-260`), and a
  closure `parse_mcp` inside `load_pack` (`crates/buzz-persona/src/pack.rs:192-197`).
- Inline captured-identifier format strings throughout (`format!("...{e}")`,
  `write!(f, "ERROR: {msg}")` — `crates/buzz-persona/src/validate.rs:28`), consistent with
  modern clippy preferences.


## Module: buzz-ws-client (`crates/buzz-ws-client`)

### Aspect: Conventions

---

### 1. Module organization

| Module | Role | file:line |
|---|---|---|
| `lib.rs` | Crate lint gate + module declarations + flat re-exports; contains no logic | `lib.rs:1`–`9` |
| `connection.rs` | Transport, handshake, timeouts, read loops, one-shot helper, unit tests | `connection.rs:1`–`314` |
| `message.rs` | Wire types + parser + AUTH-event builder (pure functions, no I/O) | `message.rs:1`–`190` |
| `error.rs` | Single error enum + one manual `From` impl | `error.rs:1`–`51` |

Conventions observed: one concern per file; pure logic (`message.rs`) separated from
I/O (`connection.rs`); errors centralized in one enum rather than per-module error
types; re-exports flattened at the root so callers can use
`buzz_ws_client::{NostrWsConnection, RelayMessage, …}` (`lib.rs:7`–`9`) while module
paths remain available.

---

### 2. Naming

| Pattern | Examples | file:line |
|---|---|---|
| Types in `UpperCamelCase`, protocol-flavoured names | `NostrWsConnection`, `RelayMessage`, `OkResponse`, `WsClientError` | `connection.rs:26`, `message.rs:8`, `message.rs:51`, `error.rs:5` |
| Enum variants mirror wire message names (`OK`→`Ok`, `EOSE`→`Eose`) | `Event`, `Ok`, `Eose`, `Closed`, `Notice`, `Auth` | `message.rs:10`–`39` |
| Constants `SCREAMING_SNAKE_CASE` with unit suffix `_SECS` | `AUTH_CHALLENGE_TIMEOUT_SECS`, `AUTH_OK_TIMEOUT_SECS`, `PUBLISH_OK_TIMEOUT_SECS` | `connection.rs:17`, `:20`, `:23` |
| Verb-first fn names; `wait_for_*` for blocking waits, `recv_*`/`send_*` for I/O | `connect`, `connect_authenticated`, `authenticate`, `send_event`, `send_raw`, `next_event`, `disconnect`, `recv_one`, `wait_for_auth_challenge`, `wait_for_ok` | `connection.rs:37`–`269` |
| Parameter naming avoids shadowing the imported `timeout` fn | `timeout_dur` (not `timeout`) | `connection.rs:106`, `:159`, `:220` |
| Builder/parse free functions named after the artifact | `build_auth_event`, `parse_relay_message` | `message.rs:174`, `:62` |
| Test names assert an invariant | `auth_challenge_timeout_meets_floor` | `connection.rs:301` |

---

### 3. Error handling

- One crate-wide `thiserror` enum, `WsClientError`, `#[derive(Debug, Error)]`
  (`error.rs:4`–`5`), with a `#[error("…")]` display string on every variant
  (`error.rs:7`, `:11`, `:15`, `:19`, `:23`, `:27`, `:31`, `:35`, `:39`, `:43`).
- Automatic conversions where the source type is owned by a dependency:
  `#[from] tokio_tungstenite::tungstenite::Error` (`error.rs:8`) and
  `#[from] serde_json::Error` (`error.rs:12`). A hand-written
  `From<nostr::event::builder::Error>` stringifies instead of wrapping
  (`error.rs:47`–`51`), and the same stringify is duplicated inline at
  `message.rs:189`.
- Foreign errors that are not `#[from]` are stringified into `Url(String)` /
  `EventBuilder(String)` via `map_err(|e| … e.to_string())`
  (`connection.rs:51`, `message.rs:180`, `message.rs:189`) — source chains are not
  preserved for those.
- `?` propagation throughout; every fallible fn returns
  `Result<_, WsClientError>` (`connection.rs:41`, `:48`, `:74`, `:96`, `:107`, `:115`,
  `:121`, `:283`; `message.rs:62`, `:179`).
- Guard-clause style: early `return Err(...)` for rule violations
  (`connection.rs:88`, `:184`, `:199`, `:241`) and early `return Ok(...)` for cache
  hits (`connection.rs:109`, `:130`, `:162`, `:171`, `:203`, `:230`, `:254`).
- Repo rule "no new `unwrap()`/`expect()` in production paths"
  (`AGENTS.md`, Quality Gates) is **not fully honoured**: two `unwrap()` calls exist
  on `VecDeque::remove` immediately after a successful `position` lookup
  (`connection.rs:170`, `:229`), paired with `_ => unreachable!()` arms
  (`connection.rs:172`, `:231`). No `expect()` anywhere.
- `#[allow(clippy::result_large_err)]` is applied to `parse_relay_message`
  (`message.rs:61`) — the only lint suppression in the crate.

---

### 4. Async patterns

- All I/O methods are `async fn` on an owned `WebSocketStream`; there is no
  `Arc`/`Mutex`, no spawned task, no channel — the connection is single-owner and
  `&mut self`-driven (`connection.rs:70`, `:96`, `:104`, `:121`).
- Timeouts are applied with `tokio::time::timeout` around a single `.next()` await
  (`connection.rs:134`, `:187`, `:244`) or around a whole async block
  (`connection.rs:284`).
- Absolute deadlines with per-iteration recomputation for multi-frame waits
  (`connection.rs:176`–`185`, `:222`, `:236`–`242`) — avoids the timeout being reset
  by unrelated frames.
- Message buffering via `VecDeque` (`connection.rs:28`) with `push_back` for
  deferral and `pop_front` for delivery (`connection.rs:108`, `:129`, `:205`, `:257`).
- `disconnect(mut self)` consumes the connection so use-after-close is a compile
  error (`connection.rs:115`).
- Sink/stream traits are brought in explicitly rather than via a prelude:
  `use futures_util::{SinkExt, StreamExt};` (`connection.rs:4`).
- The three read loops share an identical `match raw { Text | Ping | Close | _ }`
  shape (`connection.rs:140`–`153`, `:193`–`:213`, `:250`–`:267`) — a deliberate
  copy of the same frame-dispatch skeleton in each waiter.

---

### 5. Documentation conventions

- Every public item has a `///` doc comment: constants (`connection.rs:16`, `:19`,
  `:22`), the struct (`:25`), each public method (`:34`–`36`, `:47`, `:67`–`69`,
  `:95`, `:103`, `:114`, `:120`), the free functions (`:272`–`276`; `message.rs:60`,
  `:169`–`173`), both public message types (`message.rs:6`, `:49`) and every enum
  variant *and* variant field (`message.rs:9`–`46`, now including the `Count`
  variant and its two fields at `:40`–`45`), and every error variant
  (`error.rs:3`, `:6`, `:10`, `:14`, `:18`, `:22`, `:26`, `:30`, `:34`, `:38`, `:42`).
  This satisfies the repo rule that new public API carries doc comments.
- Intra-doc links are used for cross-references: `[`RelayMessage`]`
  (`message.rs:60`), `[`crate::NostrWsConnection`]` (`error.rs:3`).
- Multi-paragraph docs explain the *why* on the two most subtle items: the
  `auth_tag` parameter (`message.rs:169`–`173`) and the bounded one-shot helper
  (`connection.rs:272`–`276`).
- No `//!` module-level docs on any file, and no `#![warn(missing_docs)]` — the
  doc-comment discipline is convention, not enforced here.

---

### 6. Lints and safety posture

- `#![deny(unsafe_code)]` at the crate root (`lib.rs:1`) — matches the repo's
  "no `unsafe` code" rule; no `unsafe` block exists in the crate (verified).
- No `#![forbid(...)]`, no `clippy.toml`, no crate-level `#![allow]`.
- Dependencies are all `{ workspace = true }` (`Cargo.toml:10`–`17`), so versions and
  features are centralized in the root manifest rather than pinned per crate;
  package metadata is likewise inherited (`Cargo.toml:3`–`7`).

---

### 7. Testing conventions

- Inline `#[cfg(test)] mod tests` with `use super::*;` at the bottom of the file
  under test (`connection.rs:296`–`298`) — no `tests/` directory, no dev-dependencies
  (`Cargo.toml` has no `[dev-dependencies]` section).
- Tests are compile-time invariant assertions rather than behavioural tests:
  `const { assert!(CONST >= floor) }` inside `#[test] fn` bodies
  (`connection.rs:302`, `:307`, `:312`) — encoding timeout *floors* so a future edit
  that lowers a timeout fails to build.
- No `#[tokio::test]`, no mock relay, no parser unit tests, no property tests.


## Module: buzz-db (`crates/buzz-db`)

### Conventions

#### 1. Crate-level lints and module layout

- `#![deny(unsafe_code)]` and `#![warn(missing_docs)]` at
  `crates/buzz-db/src/lib.rs:1-2`; every public item carries a doc comment.
- One module per persistence concern, each declared with a doc comment in
  `lib.rs` (`crates/buzz-db/src/lib.rs:12-51`): `admin_moderation`, `api_token`,
  `archived_identities`, `channel`, `dm`, `error`, `event`, `feed`, `git_repo`,
  `migration`, `moderation`, `partition`, `product_feedback`, `push`, `reaction`,
  `relay_members`, `replica_fence`, `thread`, `usage`, `user`, `workflow`
  (21 modules + `lib.rs`).
- Two-layer API: module-level free functions take `pool: &PgPool` (or
  `tx: &mut Transaction<'_, Postgres>`) plus a `CommunityId`; `Db` methods are
  thin delegating wrappers. Inline SQL on `Db` is the exception, not the rule.
- Section banners inside larger modules: `// -- Public structs ---`,
  `// -- Write operations ---`, `// -- Read operations ---`, `// -- Row mappers ---`
  (e.g. `crates/buzz-db/src/reaction.rs:10`, `:64`, `:276`;
  `crates/buzz-db/src/thread.rs:16`, `:108`, `:339`).
- Re-export policy: enums shared with the SDK live in `buzz-core` and are
  re-exported, not redefined — `pub use buzz_core::channel::{ChannelType, ChannelVisibility, MemberRole}`
  with the rationale in the comment (`crates/buzz-db/src/channel.rs:13-17`).

#### 2. Naming

| Pattern | Convention | Examples |
|---------|-----------|----------|
| Reads returning a single row | `get_*` / `find_*` (`find_*` when absence is normal) | `get_channel`, `get_event_by_id`, `find_dm_by_participants`, `find_by_owner_and_name` |
| Reads returning many rows | `list_*` / `query_*` (`query_*` when filters are involved) | `list_channels`, `list_relay_members`, `query_events`, `query_mentions` |
| Batched variants | `*_bulk` suffix | `get_members_bulk`, `get_users_bulk`, `get_member_counts_bulk`, `get_last_message_at_bulk`, `get_reactions_bulk` |
| Predicates | `is_*` / `has_*` | `is_member`, `is_relay_member`, `is_archived`, `is_agent_owner`, `has_allowlist_entries`, `has_join_policy_acceptance`, `has_read_pool` |
| Idempotent create | `ensure_*` | `ensure_user`, `ensure_configured_community`, `ensure_future_partitions` |
| Upsert with ordering semantics | `replace_*` / `upsert_*` / `accept_*` | `replace_addressable_event`, `replace_parameterized_event`, `upsert_workflow`, `accept_lease_event` |
| Queue/lease lifecycle | `claim_* / complete_* / retry_* / fail_* / release_* / reap_* / prune_*` | `claim_due_wakes`, `complete_match_batch`, `retry_wake`, `fail_wake`, `release_due_reminder`, `reap_exhausted_matches`, `prune_wake_outbox` |
| Row → struct mappers | private `row_to_*` | `row_to_stored_event`, `row_to_channel_record`, `row_to_member_record`, `row_to_report`, `row_to_ban`, `row_to_action`, `row_to_workflow_record`, `row_to_run_record`, `row_to_approval_record`, `row_to_claimed_wake`, `row_to_archived_identity`, `row_to_feedback` |
| Transaction-aware twins | `*_tx` suffix | `get_active_role_tx`, `get_channel_tx`, `add_reaction_tx`, `insert_event_with_thread_metadata_tx` |
| Insert parameter bags | `New*` / `*Params` structs instead of long argument lists | `NewReport`, `NewAction`, `NewProductFeedback`, `NewWake`, `ThreadMetadataParams`, `CreateApprovalParams`, `ChannelUpdate` |
| Outcome enums instead of booleans when >2 states | `*Outcome` / `*Result` | `ReserveOutcome`, `AcceptLeaseOutcome`, `ReplaceLeaseOutcome`, `EnqueueWakeOutcome`, `RevalidateWakeOutcome`, `ReactionEventInsertOutcome`, `RemoveResult`, `TransferResult`, `CreateCommunityWithOwnerResult` |
| Advisory-lock key namespaces | `'<domain>:'` prefixes, documented as mutually distinct | `'buzz_push_gate:'` (`crates/buzz-db/src/push.rs:21`), `'buzz_channel_ttl:'` (`migrations/0024_…:20`), `'buzz_audit:'` (referenced at `migrations/0023_…:20`) |
| SQL identifiers | `snake_case`; indexes `idx_<table>_<cols>` in `0001`–`0007`/`0017`, bare `<table>_<purpose>` for push tables in `0012`/`0015`/`0018` | `idx_events_community_channel_created` vs `push_wake_outbox_due` |

Boolean returns are consistently "did this call change state": `rows_affected() > 0`
(e.g. `crates/buzz-db/src/event.rs:742`, `crates/buzz-db/src/reaction.rs:110`,
`crates/buzz-db/src/git_repo.rs:180`) or `== 1` where exactly one row is the
contract (`crates/buzz-db/src/push.rs:1148`, `crates/buzz-db/src/event.rs:1471`).

#### 3. Error handling

Single error enum `DbError` (`crates/buzz-db/src/error.rs:7-52`) with
`thiserror`, plus `pub type Result<T> = std::result::Result<T, DbError>` at `:51`.

| Variant | Source | Meaning |
|---------|--------|---------|
| `Sqlx(#[from] sqlx::Error)` | `:11` | driver-level failure |
| `Migrate(#[from] sqlx::migrate::MigrateError)` | `:15` | migration failure |
| `AuthEventRejected` | `:21` | kind 22242 must not be stored |
| `EphemeralEventRejected(u16)` | `:25` | kinds 20000–29999 must not be stored |
| `ChannelNotFound(uuid::Uuid)` | `:29` | |
| `MemberNotFound(uuid::Uuid)` | `:33` | |
| `NotFound(String)` | `:37` | generic |
| `AccessDenied(String)` | `:41` | permission/state refusal |
| `Serde(#[from] serde_json::Error)` | `:45` | JSON round-trip |
| `InvalidData(String)` | `:49` | malformed input or malformed stored value |
| `InvalidTimestamp(i64)` | `:53` | timestamp could not be interpreted |

Separate probe-only error type: `replica_fence::ProbeError`
(`crates/buzz-db/src/replica_fence.rs:363-380`) with `Writer`,
`MaskedActivity { masked }`, `ReplicaLsnUnavailable`; all three are treated
identically (fail closed) by the probe loop at `:492-502`.

Conventions:
- **No `unwrap()`/`expect()` on fallible DB results in production paths.** Row
  decoding uses `row.try_get(...)?`. The only non-test `expect`/`unwrap` calls are
  infallible-slice conversions in lock-key derivation
  (`crates/buzz-db/src/push.rs:224`, `:230`), a `expect("one outcome per request")`
  on a locally guaranteed vector length (`crates/buzz-db/src/push.rs:601`), and
  `expect("length checked")` after an explicit length check
  (`crates/buzz-db/src/thread.rs:328`).
- Postgres error codes are matched explicitly where behaviour depends on them:
  `42P17` overlap in the partition manager (`crates/buzz-db/src/partition.rs:139-141`),
  `23505` + constraint name and the generic `23xxx` family in push
  (`crates/buzz-db/src/push.rs:392-410`), `23514` check-violation in the fence
  verifier (`crates/buzz-db/src/replica_fence.rs:206`).
- Best-effort side indexes never fail the caller: mention inserts are logged and
  swallowed (`crates/buzz-db/src/lib.rs:1086-1089`, `:1392-1395`, `:1424-1427`,
  `:3428-3431`, `:3610-3613`, `:3812-3815`).
- Enum/status strings are parsed with `FromStr` returning
  `DbError::InvalidData(format!("unknown … : {other}"))` — never a silent default
  (`crates/buzz-db/src/workflow.rs:61-71`, `:103-116`, `:148-160`).
- `try_get(...).unwrap_or(None)` is used deliberately for columns that may be
  absent from a given projection, and is documented as such
  (`crates/buzz-db/src/channel.rs:990-999`).

#### 4. Query-construction style

| Style | When used | Examples |
|-------|-----------|----------|
| Static SQL string + positional `.bind()` | the default | most of the crate |
| `sqlx::QueryBuilder` with `push_bind` / `separated(", ")` / `push_values` | variable-length `IN (…)` lists and optional predicates | `crates/buzz-db/src/event.rs:360-465`, `:591-698`; `crates/buzz-db/src/feed.rs:91-119`; `crates/buzz-db/src/lib.rs:146-163`; `crates/buzz-db/src/channel.rs:1337-1349`; `crates/buzz-db/src/event.rs:877-889`, `:957-966` |
| `format!` + `sqlx::AssertSqlSafe`, all values still bound | dynamic SET/ORDER/placeholder shapes that `QueryBuilder` can't express, and DDL | 15 sites: `crates/buzz-db/src/channel.rs:870`, `:957`, `:1107`; `crates/buzz-db/src/thread.rs:430`, `:631`; `crates/buzz-db/src/user.rs:148`; `crates/buzz-db/src/usage.rs:281`, `:323`; `crates/buzz-db/src/partition.rs:130`; `crates/buzz-db/src/lib.rs:5235`, `:5256`, `:6009`, `:6014`; `crates/buzz-db/src/replica_fence.rs:613`, `:638` (last five are test-only) |
| `sqlx::query_as::<_, (…tuple…)>` | small fixed projections | `crates/buzz-db/src/user.rs:61`, `crates/buzz-db/src/usage.rs:43` and siblings, `crates/buzz-db/src/lib.rs:3345` |
| `sqlx::query_scalar::<_, T>` | single-value reads | `crates/buzz-db/src/lib.rs:519`, `:687`; `crates/buzz-db/src/push.rs:299`; `crates/buzz-db/src/relay_members.rs:455` |
| `ANY($n)` array binds | fixed-arity list predicates | `crates/buzz-db/src/channel.rs:565`, `:625`; `crates/buzz-db/src/push.rs:912`, `:980` |
| Nullable-filter idiom `($n::type IS NULL OR col = $n)` | optional filters without dynamic SQL | `crates/buzz-db/src/admin_moderation.rs:106-112`; `crates/buzz-db/src/moderation.rs:222` |
| Two static query variants instead of one dynamic string | when only a single optional predicate exists | `crates/buzz-db/src/channel.rs:669-708`; `crates/buzz-db/src/dm.rs:239-306` |

Ordering/pagination conventions: `ORDER BY created_at DESC, id ASC` for event
reads (`crates/buzz-db/src/event.rs:535-537`, rationale `:529-534`); composite keyset cursors rather
than OFFSET for channel windows and thread pages
(`crates/buzz-db/src/thread.rs:380-386`, `:595-602`); a `LIMIT n+1` probe rather
than a second COUNT for `has_more` (`crates/buzz-db/src/thread.rs:640-643`).
Every list query has a bound: an explicit `LIMIT` literal (1000), a clamped
parameter, or a constant (`FEED_MAX_LIMIT`, `LIST_MAX_LIMIT`, `MAX_PAGE_SIZE`).

#### 5. Transactions and locking

- `pool.begin()` … `tx.commit()` / `tx.rollback()`: **33** `begin()` sites and
  **30** `commit()` sites in `crates/buzz-db/src/**` — the difference is
  early-return paths that `rollback()` deliberately
  (e.g. `crates/buzz-db/src/lib.rs:3366`, `crates/buzz-db/src/event.rs:1263`,
  `crates/buzz-db/src/relay_members.rs:463`).
- Transactions are used wherever two writes must agree: channel create +
  owner bootstrap, event + thread metadata + counters, reaction + kind:7 event,
  membership + policy acceptance, community + owner, lease + source event.
- Advisory locks are acquired **first** inside a transaction, in one documented
  global order per subsystem (`crates/buzz-db/src/push.rs:239-243`,
  `migrations/0024_event_ttl_refresh_shared_lock.sql:20-24`).
- `Db::begin_transaction` (`crates/buzz-db/src/lib.rs:648-650`) exposes a
  `Transaction<'static, Postgres>` to callers, justified by `PgPool` being
  `Arc`-backed.
- Session-scoped advisory locks are held on a **detached** connection so a
  locked session is never returned to the pool
  (`crates/buzz-db/src/lib.rs:511-535`, guard type at `:203-219`).

#### 6. Row mapping

No `#[derive(sqlx::FromRow)]` anywhere in the crate (zero matches). Every row is
decoded manually, which keeps enum columns readable as `::text` and lets
projections vary per query. Common shapes:

```rust
rows.into_iter().map(row_to_report).collect()                 // moderation.rs:236
row.map(row_to_report).transpose()                            // moderation.rs:260
row.map(|row| { Ok(Record { … row.try_get("x")? }) }).transpose()  // lib.rs:673
```

Byte columns are `Vec<u8>` in structs and hex-encoded only at the presentation
boundary (`crates/buzz-db/src/admin_moderation.rs:172-176`,
`crates/buzz-db/src/product_feedback.rs:112-113`). `pubkey_hex` in
`event_mentions` is the one place hex is the storage form, always lowercased on
write (`crates/buzz-db/src/lib.rs:140`) and on read predicates
(`crates/buzz-db/src/event.rs:377`).

#### 7. Testing patterns

Counts (all tests live in in-file `#[cfg(test)] mod tests`; there is no
`tests/` directory in the crate):

| Metric | Count |
|--------|-------|
| `#[test]` (pure, no infrastructure) | **81** |
| `#[tokio::test]` | **122** |
| `#[ignore]`-gated | **121** |
| Non-ignored `#[tokio::test]` | **1** (`read_falls_back_to_writer_when_no_replica_configured`, `crates/buzz-db/src/lib.rs:5361`, uses `connect_lazy` so it never touches the network) |
| Files with a `mod tests` | 19 of 22 |

Per file:

| File | `#[test]` | `#[tokio::test]` | `#[ignore]` |
|------|-----------|------------------|-------------|
| `workflow.rs` | 24 | 7 | 7 |
| `feed.rs` | 22 | 3 | 3 |
| `event.rs` | 14 | 12 | 12 |
| `migration.rs` | 7 | 3 | 3 |
| `user.rs` | 5 | 8 | 8 |
| `dm.rs` | 4 | 0 | 0 |
| `partition.rs` | 3 | 0 | 0 |
| `replica_fence.rs` | 2 | 3 | 3 |
| `lib.rs` | 0 | 25 | 24 |
| `push.rs` | 0 | 14 | 14 |
| `relay_members.rs` | 0 | 10 | 10 |
| `thread.rs` | 0 | 10 | 10 |
| `usage.rs` | 0 | 8 | 8 |
| `channel.rs` | 0 | 7 | 7 |
| `git_repo.rs` | 0 | 4 | 4 |
| `moderation.rs` | 0 | 4 | 4 |
| `api_token.rs` | 0 | 2 | 2 |
| `archived_identities.rs` | 0 | 1 | 1 |
| `product_feedback.rs` | 0 | 1 | 1 |
| `admin_moderation.rs`, `error.rs`, `reaction.rs` | 0 | 0 | 0 |

Conventions:
- Infra tests are gated `#[ignore = "requires Postgres"]` (or
  `"requires migrated Postgres"` at `crates/buzz-db/src/product_feedback.rs:124`).
- Test DB URL resolution: a `const TEST_DB_URL` default plus
  `BUZZ_TEST_DATABASE_URL` → `DATABASE_URL` (most modules) or `TEST_DATABASE_URL`
  (`lib.rs`, `replica_fence.rs`) — see `buzz-db-configuration.md`.
- Every infra test mints its own community via a local
  `make_test_community` / `make_community` helper with a UUID-derived host, so
  tests are isolated on a shared database (e.g.
  `crates/buzz-db/src/event.rs:1484-1495`).
- Cross-tenant isolation tests deliberately use **identical** ids/shapes in two
  communities (`crates/buzz-db/src/event.rs:1601`,
  `crates/buzz-db/src/channel.rs:1553`, `crates/buzz-db/src/lib.rs:4890`).
- Replica-routing tests create two scratch databases with **divergent** fixtures
  so every assertion observes which pool actually served the query
  (`crates/buzz-db/src/lib.rs:5216-5262`, `:5379`, `:5464`), and include
  explicit "counterfactual" assertions that pin the hazard an over-open fence
  would cause (`:5647-5661`, `:5741-5755`).
- Concurrency tests use `tokio::spawn` + `sleep` + `is_finished()` to assert a
  statement *blocks*, then `tokio::time::timeout` to assert it completes
  (`crates/buzz-db/src/lib.rs:4351-4380`, `crates/buzz-db/src/event.rs:1544-1574`).
- The migration lint tests hand-roll a small SQL parser (statement splitting
  respecting `'` and `$$`, paren matching, top-level CSV split) and assert
  tenant-isolation properties over the concatenation of **all** migrations
  (`crates/buzz-db/src/migration.rs:120-370`, `:635-688`).
- Pure unit tests concentrate on the code that has no DB dependency: validators
  (`partition.rs:153-181`), tag extraction (`event.rs:1968-2081`), hashing/ordering
  (`dm.rs:520+`), SQL-shape assertions built from `QueryBuilder`
  (`feed.rs:766-861`), and enum round-trips (`workflow.rs:1199+`).


## Module: buzz-auth (`crates/buzz-auth`)

### Conventions

### Module organisation

8 source files, 1,877 LOC total. One file per concern; `lib.rs` holds the
aggregate types plus re-exports.

| File | LOC | Responsibility |
|------|-----|----------------|
| `crates/buzz-auth/src/lib.rs` | 243 | crate docs, module decls, re-exports, `AuthMethod`/`AuthContext`/`AuthConfig`/`AuthService`, dev key derivation |
| `crates/buzz-auth/src/access.rs` | 251 | `ChannelAccessChecker`, `require_scope`, read/write helpers, `MockAccessChecker` |
| `crates/buzz-auth/src/error.rs` | 59 | `AuthError` only |
| `crates/buzz-auth/src/nip42.rs` | 183 | challenge gen + AUTH verification + relay-URL normalisation |
| `crates/buzz-auth/src/nip98.rs` | 317 | HTTP auth verification + URL normalisation |
| `crates/buzz-auth/src/nip98_replay.rs` | 249 | `Nip98ReplayGuard`, key formats, TTL constants |
| `crates/buzz-auth/src/rate_limit.rs` | 326 | `RateLimiter`, `RateLimitConfig`, `LimitType`, `RateLimitResult`, key formats |
| `crates/buzz-auth/src/scope.rs` | 249 | `Scope`, `parse_scopes` |

Structural conventions observed:

- Every module has a `//!` module-level doc block; several encode the algorithm
  as a numbered list before the code (`crates/buzz-auth/src/nip42.rs:1-7`,
  `crates/buzz-auth/src/nip98.rs:13-24`).
- Every module ends with `#[cfg(test)] mod tests { use super::*; ... }` except
  `error.rs`, which has no tests (`crates/buzz-auth/src/error.rs` — 59 lines, no
  test module).
- Test doubles live in the same file as the trait they implement, gated on
  `#[cfg(any(test, feature = "test-utils"))]`, and are re-exported from `lib.rs`
  under the same gate (`crates/buzz-auth/src/lib.rs:46-51`).
- Private helpers are placed immediately adjacent to their only caller
  (`normalize_relay_url` above `verify_nip42_event`,
  `crates/buzz-auth/src/nip42.rs:19-33`; `normalize_url` below
  `verify_nip98_event`, `crates/buzz-auth/src/nip98.rs:145-153`).
- Crate lints are declared at the top of `lib.rs`: `#![deny(unsafe_code)]`,
  `#![warn(missing_docs)]` (`crates/buzz-auth/src/lib.rs:1-2`). Every public item
  carries a doc comment, including individual enum variants and struct fields.

---

### Naming conventions

| Category | Convention | Examples |
|----------|-----------|----------|
| Verification fns | `verify_<spec>_event` | `verify_nip42_event` (`nip42.rs:47`), `verify_nip98_event` (`nip98.rs:55`) |
| Redis key builders | `<subject>_key` / `<subject>_key_for_scope` | `rate_limit_key` (`rate_limit.rs:201`), `ip_rate_limit_key` (`rate_limit.rs:213`), `nip98_replay_key` (`nip98_replay.rs:114`), `nip98_replay_key_for_scope` (`nip98_replay.rs:119`) |
| Check fns | `check_<subject>` (async, returns `Result<(), AuthError>`) or `require_<subject>` (sync) | `check_read_access`/`check_write_access` (`access.rs:72`, `:88`), `require_scope` (`access.rs:60`) |
| Trait methods that mutate shared state | verb-first, `try_`-prefixed when they can legitimately decline | `try_mark`, `try_mark_in_scope` (`nip98_replay.rs:97`, `:66`); `check_and_increment` (`rate_limit.rs:174`) |
| Traits | noun-agent (`-er` / `-Checker` / `-Guard`) | `RateLimiter`, `ChannelAccessChecker`, `Nip98ReplayGuard` |
| Test doubles | `Always<Behaviour><Trait>` or `Mock<Subject>` | `AlwaysAllowRateLimiter` (`rate_limit.rs:219`), `AlwaysFreshReplayGuard` (`nip98_replay.rs:127`), `MockAccessChecker` (`access.rs:108`) |
| Constants | SCREAMING_SNAKE with unit suffix | `TIMESTAMP_TOLERANCE_SECS` (`nip42.rs:35`, `nip98.rs:32`), `DEFAULT_REPLAY_TTL_SECS` / `MAX_REPLAY_TTL_SECS` (`nip98_replay.rs:46`, `:59`) |
| serde default fns | `default_<tier>_<metric>`, abbreviated | `default_human_msg`, `default_agent_std_api`, `default_agent_plat_msg` (`rate_limit.rs:110-130`) |
| Enum-to-wire mapping | `as_str()` returning `&str` (or `&'static str`) | `Scope::as_str` (`scope.rs:114`), `LimitType::key_suffix` (`rate_limit.rs:71`) |
| Scope wire strings | `resource:action` lowercase, `admin:resource` for admin | `messages:read`, `admin:channels` (`scope.rs:116-131`) |
| Redis key namespace | `buzz:` prefix, community segment second, subject last | `buzz:{community}:ratelimit:{pubkey}:{suffix}` (`rate_limit.rs:203`), `buzz:{scope}:nip98:{id}` (`nip98_replay.rs:120`) |

---

### Error handling

Single crate-wide error enum, `thiserror`-derived, no `anyhow`, no boxed errors,
no custom `Display` impls (`crates/buzz-auth/src/error.rs:8-9`).

| Variant | `#[error(...)]` message | Constructed at |
|---------|-------------------------|----------------|
| `InvalidSignature` | `invalid signature or malformed auth event` | `nip42.rs:53` (wrong kind), `nip42.rs:56` (sig/id failure) |
| `ChallengeMismatch` | `challenge mismatch` | `nip42.rs:62` (tag missing), `nip42.rs:65` (mismatch) |
| `RelayUrlMismatch` | `relay url mismatch` | `nip42.rs:72` (tag missing), `nip42.rs:75` (mismatch) |
| `EventExpired` | `auth event timestamp outside ±60s window` | `nip42.rs:82` |
| `Nip98Invalid(String)` | `NIP-98 HTTP Auth verification failed: {0}` | `nip98.rs:63`, `:67`, `:75`, `:82`, `:95`, `:98`, `:108`, `:111`, `:123` (9 sites) |
| `Nip98Replay` | `NIP-98 replay: event id already seen within window` | **never constructed in this crate** — intended for callers of `Nip98ReplayGuard` (usage example at `nip98_replay.rs:24`) |
| `PubkeyMismatch` | `pubkey mismatch: event pubkey does not match authenticated identity` | **never constructed in this crate** |
| `InsufficientScope { required: String, have: Vec<String> }` | `insufficient scope: required {required}, have {have:?}` | `access.rs:64-67` |
| `ChannelAccessDenied` | `channel access denied` | `access.rs:83`, `access.rs:99` |
| `Internal(String)` | `internal auth error: {0}` | `lib.rs:132` (spawn_blocking panic), `lib.rs:165` (dev key derivation) |

Handling patterns:

- **Guard clauses with early return**, not nested conditionals — both verifiers
  are flat sequences of `if ... { return Err(...) }`
  (`crates/buzz-auth/src/nip42.rs:52-84`, `crates/buzz-auth/src/nip98.rs:66-127`).
- **Coarsening on the NIP-42 path**: the underlying `VerificationError` is
  discarded via `.map_err(|_| AuthError::InvalidSignature)`
  (`crates/buzz-auth/src/nip42.rs:56`) and wrong-kind reuses the same variant
  (`:53`) — a caller cannot distinguish "wrong kind" from "bad signature".
- **Descriptive on the NIP-98 path**: each failure carries a formatted string,
  including the offending values for URL/method mismatches
  (`crates/buzz-auth/src/nip98.rs:98-100`, `:111-113`). The doc explicitly says
  the message is "safe for server logs but should not be forwarded verbatim to
  clients" (`crates/buzz-auth/src/nip98.rs:53-54`).
- **Error-hygiene rule stated on the enum**: "Do **not** include raw token
  values, database contents, or stack traces in error messages"
  (`crates/buzz-auth/src/error.rs:5-7`).
- **Double-`?` for nested Results**: `spawn_blocking` join error and the inner
  verification error are both propagated on one line with `??`
  (`crates/buzz-auth/src/lib.rs:132`).
- **`Infallible` instead of a fallible parse**: `FromStr for Scope` cannot fail;
  unknown input becomes `Scope::Unknown(_)`
  (`crates/buzz-auth/src/scope.rs:143-166`).
- **One production-path `expect`**: `parse_scopes` calls
  `.expect("infallible: Scope::from_str cannot fail")`
  (`crates/buzz-auth/src/scope.rs:175`) — statically unreachable given the
  `Infallible` error type, but it is the only non-test `expect`/`unwrap` in the
  crate. Every other `unwrap`/`expect` occurrence is inside a `#[cfg(test)]`
  module.
- **No `unsafe`**: `#![deny(unsafe_code)]` (`crates/buzz-auth/src/lib.rs:1`); the
  only occurrence of the token `unsafe` in the whole crate is that attribute.
- **No logging on error paths**: despite `tracing` being a declared dependency
  (`crates/buzz-auth/Cargo.toml:20`), nothing in `src/` emits a log or span.

---

### Async conventions

- Traits use **RPITIT** (`-> impl Future<Output = ...> + Send`) for
  `ChannelAccessChecker` (`crates/buzz-auth/src/access.rs:35-39`, `:46-51`) and
  `RateLimiter` (`crates/buzz-auth/src/rate_limit.rs:174-181`, `:188-193`) —
  these are not dyn-compatible.
- `Nip98ReplayGuard` deliberately uses `Pin<Box<dyn Future + Send + 'a>>`
  instead (`crates/buzz-auth/src/nip98_replay.rs:66-71`, `:97-103`) so it can be
  stored as `Arc<dyn Nip98ReplayGuard>` by the relay
  (`crates/buzz-relay/src/state.rs:582`). Two different async-trait styles
  coexist in one crate, chosen per dyn-dispatch need.
- Default trait method bodies use `async move { ... }` blocks
  (`crates/buzz-auth/src/access.rs:52-55`) or `Box::pin(async move { ... })`
  (`crates/buzz-auth/src/nip98_replay.rs:101`).
- Free async fns are plain `pub async fn` (`crates/buzz-auth/src/access.rs:72`,
  `:88`).
- CPU-bound crypto is offloaded with `spawn_blocking` and the values are cloned
  into the closure first (`crates/buzz-auth/src/lib.rs:125-132`); the sync
  verifier documents the requirement in its own doc comment
  (`crates/buzz-auth/src/nip42.rs:46`).

---

### Configuration conventions

- Config structs derive `Serialize + Deserialize + Default`, and **every**
  `RateLimitConfig` field carries `#[serde(default = "fn")]` so partial input
  deserialises (`crates/buzz-auth/src/rate_limit.rs:85-108`).
- The manual `Default` impl duplicates the same seven `default_*` fns rather
  than using `#[derive(Default)]` with literals, keeping one source of truth for
  each number (`crates/buzz-auth/src/rate_limit.rs:132-144`).
- Nested config uses `#[serde(default)]` on the field
  (`crates/buzz-auth/src/lib.rs:93`).
- Enum wire forms use `#[serde(rename_all = "snake_case")]`
  (`crates/buzz-auth/src/rate_limit.rs:57`).

---

### Testing patterns

Totals by attribute (grep of `crates/buzz-auth/src/*.rs`):

| File | `#[test]` | `#[tokio::test]` | Total |
|------|-----------|------------------|-------|
| `crates/buzz-auth/src/nip98.rs` | 11 | 0 | 11 |
| `crates/buzz-auth/src/nip42.rs` | 8 | 0 | 8 |
| `crates/buzz-auth/src/nip98_replay.rs` | 6 | 1 | 7 |
| `crates/buzz-auth/src/rate_limit.rs` | 4 | 1 | 5 |
| `crates/buzz-auth/src/scope.rs` | 5 | 0 | 5 |
| `crates/buzz-auth/src/access.rs` | 0 | 5 | 5 |
| `crates/buzz-auth/src/lib.rs` | 1 | 3 | 4 |
| `crates/buzz-auth/src/error.rs` | 0 | 0 | 0 |
| **Total** | **35** | **10** | **45** |

There is no `crates/buzz-auth/tests/` directory — all 45 tests are in-file unit
tests.

Patterns:

- **Builder helpers per file**: `make_auth_event(keys, challenge, relay_url)`
  (`crates/buzz-auth/src/nip42.rs:95-100`, duplicated at
  `crates/buzz-auth/src/lib.rs:174-179`), `make_nip98_event(...)` with optional
  payload/timestamp params (`crates/buzz-auth/src/nip98.rs:163-186`).
- **`fixture_ctx(host)` deterministic tenant fixtures**: SHA-256 of the host name
  → first 16 bytes → `Uuid` → `CommunityId`, so key-prefix assertions are stable
  (`crates/buzz-auth/src/rate_limit.rs:253-260`,
  `crates/buzz-auth/src/nip98_replay.rs:149-155`). `access.rs` uses a simpler
  random-UUID variant (`crates/buzz-auth/src/access.rs:159-161`).
- **`assert!(matches!(result, Err(AuthError::X)))`** as the standard rejection
  assertion (`crates/buzz-auth/src/nip42.rs:124-127`,
  `crates/buzz-auth/src/access.rs:188-191`, `crates/buzz-auth/src/lib.rs:227`).
- **Real keypairs, never fixtures**: every test calls `Keys::generate()` and
  signs with `EventBuilder`, so signature verification is genuinely exercised
  (e.g. `crates/buzz-auth/src/nip98.rs:190-191`).
- **Const-drift tripwires**: assertions deliberately made over constants, with
  `#[allow(clippy::assertions_on_constants)]` and a comment explaining why
  (`crates/buzz-auth/src/nip98_replay.rs:210-230`,
  `crates/buzz-auth/src/nip98_replay.rs:232-237`).
- **Property-style invariant tests over key formats**: all-lowercase character
  scan (`crates/buzz-auth/src/rate_limit.rs:290-306`,
  `crates/buzz-auth/src/nip98_replay.rs:194-208`), cross-community
  distinctness (`crates/buzz-auth/src/rate_limit.rs:275-288`,
  `crates/buzz-auth/src/nip98_replay.rs:178-192`,
  `crates/buzz-auth/src/access.rs:225-251`).
- **Set-cardinality + no-duplicate + no-`Unknown` assertions** on the scope
  constructors (`crates/buzz-auth/src/scope.rs:205-248`).
- **Security-rationale comments inside tests** explaining what the assertion
  protects (`crates/buzz-auth/src/nip98.rs:290-294`,
  `crates/buzz-auth/src/rate_limit.rs:277-278`,
  `crates/buzz-auth/src/access.rs:227-230`).
- Test-only `unwrap`/`expect` is used freely; no test-helper crate, no
  `proptest`/`quickcheck`, no `mockall` — hand-written doubles only.


## Module: buzz-pubsub (`crates/buzz-pubsub`)
### Aspect: Conventions

---

### 1. Lint posture and repo-rule compliance

| Rule | Status | Evidence |
|---|---|---|
| No `unsafe` | Enforced at crate level | `#![deny(unsafe_code)]` `lib.rs:1`; zero `unsafe` blocks |
| Public API documented | Enforced at crate level | `#![warn(missing_docs)]` `lib.rs:2` — a `warn`, not `deny`, so a missing doc does not fail the build |
| No `unwrap()`/`expect()` in production paths | **Compliant** | Every `unwrap`/`expect` occurrence is inside `#[cfg(test)]` modules (`lib.rs:369-627`, `presence.rs:105-208`, `nip98_replay.rs:103-201`, `topic.rs:117-196`, `cache_invalidation.rs:183-250`, `conn_control.rs:169-228`, `subscriber.rs:190-204`). The single production `expect` is in `#[cfg(test)] mod test_util` (`lib.rs:374`) |
| Workspace-inherited manifest fields | Compliant | `version`/`edition`/`rust-version`/`license`/`repository` all `.workspace = true` (`Cargo.toml:3-7`) |
| Workspace dependency pinning | Compliant | All 13 deps use `{ workspace = true }` (`Cargo.toml:11-23`); no local version literals |
| No TODO/FIXME/HACK markers | Compliant | none present in any of the 10 files |

### 2. Error-handling conventions

- One `thiserror` enum per crate, `PubSubError` (`error.rs:5-28`), with `#[from]`
  conversions for the three foreign error types (`error.rs:8`, `:12`, `:16`) and a
  hand-written `From<broadcast::error::RecvError>` that maps lag and closure onto
  domain variants (`error.rs:31-38`).
- **Trait-imposed exception:** the two `buzz-auth` trait impls return
  `AuthError` instead, because the trait signatures demand it (`rate_limiter.rs:35`,
  `nip98_replay.rs:37`). Foreign errors are flattened into
  `AuthError::Internal(format!(...))` at `rate_limiter.rs:45`, `:52`, `:66` and
  `nip98_replay.rs:57`, `:79`, `:95` — a deliberate choice to keep the user-facing
  category string bounded, called out in a comment at `nip98_replay.rs:50-51`.
- **Resilience convention: never let one bad message kill a loop.** Every subscriber
  handles a malformed input with `tracing::warn!` + `continue`
  (`subscriber.rs:132-157`, `cache_invalidation.rs:144-165`,
  `conn_control.rs:135-156`).
- **Fire-and-forget convention:** where the DB is the durable backstop, send results
  are discarded with `let _ =` and the rationale is documented at the call site
  (`lib.rs:203-207`, `:236-243`; doc at `lib.rs:266-271`, `:285-291`).

### 3. Naming and structural conventions

- All Redis keys and channels are built through a helper, never inline-formatted at
  the call site; the prefix is a single constant `BUZZ_PREFIX` (`topic.rs:13`) reused
  by every module (`presence.rs:14`, `cache_invalidation.rs:20`, `conn_control.rs:23`).
- Key layout is uniformly `buzz:{community}:{kind}[:{id}]`. The one deliberate
  exception, IP rate limits, is documented as such (`rate_limiter.rs:84-85`).
- Channel-suffix constants are exported so producer and subscriber cannot drift:
  `CACHE_INVALIDATION_SUFFIX`/`_PATTERN` (`cache_invalidation.rs:23`, `:27`),
  `CONN_CONTROL_SUFFIX`/`_PATTERN` (`conn_control.rs:26`, `:30`).
- Every construct-a-key path takes `&TenantContext` rather than a raw
  `CommunityId`, so a caller cannot fabricate a tenant
  (`topic.rs:35`, `:103`, `:108`, `presence.rs:19`, `cache_invalidation.rs:30`,
  `conn_control.rs:33`).
- Paired parse/format functions live beside each other and are round-trip tested
  (`topic.rs:43`/`:53` tested `topic.rs:150-177`;
  `cache_invalidation.rs:30`/`:38` tested `:201-207`;
  `conn_control.rs:33`/`:38` tested `:183-188`).
- Builder pattern with a `DEFAULT_*` associated const for tunables
  (`lib.rs:82`, `:93-96`).
- Backoff constants are duplicated per module rather than shared:
  `BACKOFF_INITIAL_SECS`/`BACKOFF_MAX_SECS` appear three times
  (`subscriber.rs:16-19`, `cache_invalidation.rs:91-94`, `conn_control.rs:81-84`)
  with identical values.

### 4. Concurrency conventions

- `Arc<Self>` receivers for infinite background loops (`lib.rs:148`, `:165`, `:175`);
  `PubSubManager` is intentionally not `Clone` (`lib.rs:100`).
- Take-once initialisation guard: the `mpsc::Receiver` is stored as
  `Mutex<Option<..>>` and `take()`n so a second `run_subscriber` is a logged no-op
  rather than a panic or a silent double-consume (`lib.rs:107`, `:149-152`).
- Locks are held for the shortest possible span: the refcount mutations compute a
  boolean inside a block, then the lock drops before any `await` on the channel
  (`lib.rs:194-207`, `:217-231`) — avoids holding a `tokio::Mutex` across `await`.
- `tokio::select!` for multiplexing commands against the message stream, with an
  `else` arm for total-shutdown detection (`subscriber.rs:110-171`).

### 5. Documentation conventions

- Crate-level ASCII architecture diagram (`lib.rs:8-16`) plus explicit "why" notes
  for non-obvious choices: why the pub/sub connection is not pooled (`lib.rs:19-20`),
  why cache-invalidation and conn-control are separate channels
  (`conn_control.rs:12-18`), why topics are not an isolation boundary
  (`topic.rs:3-6`, `lib.rs:305-320`), why the TTL is 3× the heartbeat
  (`presence.rs:4-6`).
- Known-limitation callouts are inline rather than in a separate doc: the fixed-window
  2× burst warning (`rate_limiter.rs:9-10`), the "upgrade to sliding window" note
  (same lines), and the `⚠️` marker convention.
- Contract obligations are written into log messages, not just docs — e.g. "caller
  MUST fail closed" appears in the `warn!` payloads at `nip98_replay.rs:55`, `:76`.

### 6. Testing conventions

34 test functions across 8 of 10 files. 11 require live Redis and are gated
`#[ignore = "requires Redis"]` — a consistent, uniform gate string
(`lib.rs:400`, `:438`, `:476`, `:511`; `presence.rs:137`, `:160`, `:187`;
`nip98_replay.rs:128`, `:145`, `:165`, `:180`).

| File | Tests | Redis-gated |
|---|---|---|
| `lib.rs` | 6 | 4 |
| `topic.rs` | 6 | 0 |
| `conn_control.rs` | 6 | 0 |
| `cache_invalidation.rs` | 5 | 0 |
| `presence.rs` | 5 | 3 |
| `nip98_replay.rs` | 4 | 4 |
| `subscriber.rs` | 2 | 0 |
| `error.rs`, `publisher.rs`, `rate_limiter.rs` | 0 | — |

Conventions observed:
- A shared `#[cfg(test)] mod test_util` provides `make_test_pool()` (`lib.rs:369-377`)
  reused by `presence.rs:107`; `nip98_replay.rs` instead builds its own pool honouring
  a `REDIS_URL` env override (`nip98_replay.rs:110-116`) — an inconsistency in an
  otherwise uniform pattern.
- Every module defines an identical local `fn ctx(id, host)` helper
  (`lib.rs:388`, `topic.rs:119`, `presence.rs:119`, `cache_invalidation.rs:185`,
  `conn_control.rs:171`, and inline at `subscriber.rs:191`) — six copies of the same
  three-line fixture.
- Negative-path tables: malformed inputs are asserted as a `for` loop over a literal
  array (`topic.rs:179-195`, `cache_invalidation.rs:222-233`).
- Serde round-trip tests for every wire enum variant (`cache_invalidation.rs:235-249`,
  `conn_control.rs:202-207`, `:219-227`) plus a forward-compat test asserting an
  unknown `op` is rejected without poisoning subsequent messages
  (`conn_control.rs:209-217`).
- Intent-documenting test names and comments: `lib.rs:556-557` explains that the test
  exists to catch a channel-id-only keying bug; `nip98_replay.rs:184-190` explains why
  clamping beats propagating an error.
- `test_presence_set_and_get` is **duplicated** — defined at both `lib.rs:477` and
  `presence.rs:138` with overlapping assertions.

`rate_limiter.rs` has **zero tests** despite implementing a security control; its
behaviour is covered only indirectly through `buzz-relay/src/admission.rs`'s
`StubLimiter` (`admission.rs:65-90`), which stubs out the Redis logic entirely — so
the Lua script, the `count <= limit` boundary (`rate_limiter.rs:74`), and the TTL
repair path (`rate_limiter.rs:57-70`) are untested anywhere in the repo.


## Module: buzz-search (`crates/buzz-search`)

### Conventions

#### Module organization

| Module | File | Role |
|---|---|---|
| crate root | `src/lib.rs` (54 LOC) | lints, module decls, re-exports, `SearchService` handle |
| `error` | `src/error.rs` (9 LOC) | `SearchError` only |
| `query` | `src/query.rs` (352 LOC) | all query types + SQL construction + execution + unit tests |
| integration tests | `tests/fts_integration.rs` (1448 LOC) | Postgres-backed behavior tests |

Both modules are declared `pub mod` with doc comments on the declaration itself
(`lib.rs:24-27`), and their contents are flattened into the crate root by a single
`pub use` line each (`lib.rs:29-31`). Callers use both spellings in-tree:
`buzz_search::SearchService` (`crates/buzz-relay/src/state.rs:28`) and
`buzz_search::ChannelScope` / `SearchQuery` / `SearchMode` via the root
(`crates/buzz-relay/src/api/bridge.rs:1665`, `1687`, `1697`).

#### Lints

| Lint | Line |
|---|---|
| `#![deny(unsafe_code)]` | `lib.rs:1` |
| `#![warn(missing_docs)]` | `lib.rs:2` |

`missing_docs` is honored throughout: every public item, field, and enum variant
carries a doc comment (`query.rs:43-52`, `75-98`, `106-117`, `123-126`;
`error.rs:3-6`; `lib.rs:35-51`). One `#[allow]` appears in the crate, in tests
only: `#[allow(clippy::too_many_arguments)]` on the `insert_event` helper
(`tests/fts_integration.rs:118`).

#### Naming

| Pattern | Examples |
|---|---|
| Types: `Search*` prefix for the public surface | `SearchService`, `SearchQuery`, `SearchHit`, `SearchResult`, `SearchMode`, `SearchError` |
| Enum variants read as constraints, not flags | `Any`, `ChannelLessOnly`, `Channels`, `ChannelsOrChannelLess` (`query.rs:44-52`) |
| Private helpers: verb-first (`push_*`) or noun-phrase for pure fns | `push_tsquery` (`query.rs:140`), `normalized_search_text` (`query.rs:179`) |
| Constants: SCREAMING_SNAKE with the bound in the name | `PER_PAGE_MAX`, `PER_PAGE_DEFAULT`, `SEARCH_TEXT_MAX_CHARS`, `PAGE_MAX` (`query.rs:129-138`) |
| SQL aliases spelled out | `created_at_s`, `search_query.query`, `prefix_terms`, `raw_token`, `normalized` (`query.rs:235-237`, `168-176`) |
| Test names assert behavior, not method names | `search_does_not_return_other_community_events`, `channel_less_only_excludes_per_channel_events`, `excluded_kinds_are_storage_level_unsearchable` |

#### Error handling

Single-variant `thiserror` enum (`error.rs:4-9`):

| Variant | Attributes | Message |
|---|---|---|
| `Db(sqlx::Error)` | `#[from]`, `#[error("database error: {0}")]` | `error.rs:7-8` |

Conventions observed:
- All fallible steps use `?` — no `unwrap()`/`expect()`/`panic!` anywhere in `src/`
  (checked in all three source files), matching the repo rule against new
  `unwrap()`/`expect()` in production paths.
- Domain-shaped decode failures are expressed by re-wrapping into
  `sqlx::Error::Decode` with a message that names the column and the observed
  length, rather than adding an enum variant (`query.rs:306-311`).
- Degenerate input is handled by returning a valid empty result rather than an
  error (`query.rs:217-222`).

#### Query style

- One statement, assembled with `QueryBuilder` in strict order: projection + `FROM`
  seed (`query.rs:233-238`), tenant predicate (`240-241`), always-on predicates
  (`242`), optional predicates (`248-293`), `ORDER BY`/`LIMIT`/`OFFSET`
  (`295-298`).
- Every dynamic value goes through `push_bind`; `push` is used only for fixed SQL
  text (see integrations doc for the full bind list).
- Vectors are `.clone()`d into the bind because the builder needs owned values
  (`query.rs:257`, `262`, `270`, `278`).
- Multi-line SQL uses trailing `\` line continuations inside one string literal
  (`query.rs:234-237`, `154-176`).
- Non-obvious SQL carries an adjacent rationale comment (prefix-mode design at
  `query.rs:148-153`; channel-scope mapping at `query.rs:244-247`).
- Doc comments state the invariant in prose next to the code that enforces it
  ("`community_id = $ctx` is the first predicate and is non-negotiable",
  `query.rs:209-210`).

#### Testing patterns

| Metric | Count | Where |
|---|---|---|
| `#[test]` (sync unit tests) | 3 | `query.rs:329`, `338`, `346` inside `#[cfg(test)] mod tests` (`query.rs:325-352`) |
| `#[tokio::test]` (async integration tests) | 18 | `tests/fts_integration.rs` |
| `#[ignore = "requires Postgres"]` | 18 | every integration test; none of the 3 unit tests is ignored |

So 21 tests total, 18 infra-gated. Unit tests cover only
`normalized_search_text` (trim/reject-empty, NUL replacement, char cap).

Integration-test conventions:
- Per-test isolated schema named `fts_test_<uuid-simple>`, created and dropped
  around each test, declared parallel-safe (`tests/fts_integration.rs:6-8`,
  `35-46`, `87-103`).
- Full migration chain replayed so the test schema matches production
  (`tests/fts_integration.rs:55-84`).
- Test-only DDL string interpolation is explicitly marked with
  `sqlx::AssertSqlSafe` (`tests/fts_integration.rs:44`, `100`).
- Fixture helpers: `mk_community`, `insert_event`, `rand_bytes32`
  (`tests/fts_integration.rs:105-153`).
- Deterministic timestamps in the `1_700_000_000` family, unique content tokens
  per test to avoid cross-test coupling.
- Several tests document their own mutation-kill argument — flip the predicate and
  the assertion goes red (`tests/fts_integration.rs:877-887`, `1100-1104`).
- Two tests are explicit drift tripwires that iterate Rust constants
  (`AUTHOR_ONLY_KINDS`, persistent subset of `P_GATED_KINDS`) against the schema's
  inlined exclusion list (`tests/fts_integration.rs:1256-1265`, `1338-1361`).
- Run instruction is documented at the top of the file, including the
  `BUZZ_TEST_DATABASE_URL` override and `-- --include-ignored`
  (`tests/fts_integration.rs:3`).


## Module: buzz-audit (`crates/buzz-audit`)

### Aspect: Conventions

### Module organization

| File | Lines | Role | Declared in `lib.rs` |
|---|---|---|---|
| `src/lib.rs` | 35 | crate docs, lints, module declarations, re-exports | — |
| `src/action.rs` | 100 | `AuditAction` enum + string mapping + `FromStr`/`Display` | `lib.rs:21` |
| `src/entry.rs` | 72 | `AuditEntry` (stored) and `NewAuditEntry` (input) | `lib.rs:23` |
| `src/error.rs` | 108 | `AuditError` | `lib.rs:25` |
| `src/hash.rs` | 272 | `GENESIS_HASH`, `to_storage_precision`, `compute_hash`, `canonical_json` | `lib.rs:27` |
| `src/service.rs` | 527 | `AuditService` (append/verify/read) + row decode | `lib.rs:29` |

One concept per file; no `mod.rs` nesting, no `tests/` directory. Every module gets a
doc comment at its declaration site (`lib.rs:20-29`) in addition to its own file-level
items.

Crate-level lints: `#![deny(unsafe_code)]` and `#![warn(missing_docs)]`
(`lib.rs:1-2`). Public items are documented throughout (e.g. `action.rs:5`, `:9-30`,
`:34`; `entry.rs:8-12`, `:15-36`; `service.rs:31-36`, `:42`, `:47-51`).

Selective re-export at the root rather than a blanket `pub use`: `AuditAction`,
`AuditEntry`, `NewAuditEntry`, `AuditError`, `compute_hash`, `GENESIS_HASH`,
`AuditService` (`lib.rs:31-35`). `to_storage_precision` stays module-qualified
(`hash.rs:22`).

### Naming

- Types: `UpperCamelCase` — `AuditService`, `AuditEntry`, `NewAuditEntry`,
  `AuditAction`, `AuditError`. The `New*` prefix marks the pre-persistence input
  (`entry.rs:52`).
- Functions: `snake_case` verbs — `log`, `verify_chain`, `get_entries`, `compute_hash`,
  `canonical_json`, `to_storage_precision`, `log_timestamp`, `row_to_audit_entry`.
  `*_inner` marks the private continuation of a public wrapper (`service.rs:82`).
- Constants: `SCREAMING_SNAKE_CASE` — `GENESIS_HASH` (`hash.rs:9`),
  `AUDIT_LOCK_NAMESPACE` (`service.rs:29`), private `ALL` (`action.rs:51`).
- DB action strings are snake_case of the variant name (`action.rs:37-47`).
- Row-decode helper is a free function, not a `From` impl, because it must return
  `Result` (`service.rs:238`).
- Aliased import to avoid trait-name pollution: `use futures_util::FutureExt as _`
  (`service.rs:2`).

### Error handling

Single crate error enum `AuditError`, `thiserror`-derived (`error.rs:11-41`):

| Variant | Payload | `#[error]` message | Line | Constructed at |
|---|---|---|---|---|
| `Database` | `#[from] sqlx::Error` | `database error: {0}` | `error.rs:14-15` | implicitly by `?` on every sqlx call (`service.rs:54`, `:62`, `:101`, `:147`, `:149`, `:179`, `:232`) |
| `ChainViolation` | `{ seq: i64 }` | `hash chain integrity violation at seq {seq}: prev_hash does not match preceding entry` | `error.rs:19-25` | `service.rs:193` |
| `HashMismatch` | `{ seq: i64 }` | `hash mismatch at seq {seq}: stored hash does not match recomputed hash` | `error.rs:28-32` | `service.rs:199` |
| `UnknownAction` | none | `unknown audit action in database` | `error.rs:35-36` | `service.rs:242` |
| `Serialization` | `#[from] serde_json::Error` | `serialization error: {0}` | `error.rs:39-40` | via `?` on `canonical_json` (`hash.rs:67`) |

Conventions observed:
- `?` propagation everywhere in production paths; **no `unwrap()`/`expect()`/`panic!`/
  `unimplemented!()` outside `#[cfg(test)]`** (grep confirms all 38 hits are inside test
  modules, e.g. `service.rs:304`, `hash.rs:155`).
- `#[from]` used only for foreign error types (`error.rs:15`, `:40`); domain variants
  are constructed explicitly with named fields.
- Deliberate error-sanitization rule: no variant carries a `community_id` or constraint
  name, documented at `error.rs:3-10` and pinned by a test that asserts the rendered
  text contains neither the community UUID (both hyphenated and simple forms) nor the
  strings `community_id`, `audit_log_pkey`, `constraint`, `communities`
  (`error.rs:58-107`).
- One intentional error suppression: the advisory unlock result is discarded with
  `let _ = ...` (`service.rs:71`).
- `FromStr for AuditAction` uses `Err = String` (`action.rs:73`, message at `:80`); the
  service discards that string and substitutes `UnknownAction` after a `warn!`
  (`service.rs:240-243`).

### Tracing / observability conventions

- `#[instrument(skip(self, entry), fields(action = %entry.action))]` on `log`
  (`service.rs:52`) — the entry itself is skipped so `detail` never lands in a span.
- `#[instrument(skip(self))]` on `verify_chain` (`service.rs:159`) and `get_entries`
  (`service.rs:211`), so `community`/`seq` args *are* recorded.
- `debug!(seq, "writing audit entry")` (`service.rs:128`); `warn!("unknown action in
  audit log")` without the offending value (`service.rs:241`).
- No metrics emitted from this crate; counters/histograms live in the relay
  (`crates/buzz-relay/src/state.rs:1201-1206`).

### Documentation conventions

Doc comments carry rationale, not just description — e.g. why the pre-image order is
frozen (`hash.rs:28-30`), why timestamps are truncated (`hash.rs:11-21`), why
`NewAuditEntry` is not `Deserialize` (`entry.rs:46-51`), why the lock is per-community
(`service.rs:25-28`), why `detail` must not hold tokens (`entry.rs:64-71`). Intra-doc
links are used (`[`crate::hash::GENESIS_HASH`]` at `entry.rs:22`,
`[`AuditService::log_inner`]` at `service.rs:19`).

### Testing patterns

Counts across `crates/buzz-audit/src`:

| Metric | Count | Locations |
|---|---|---|
| `#[cfg(test)] mod tests` blocks | 4 | `action.rs:84`, `error.rs:43`, `hash.rs:118`, `service.rs:258` |
| `#[test]` (sync) | 13 | `action.rs:88,96`; `error.rs:58`; `hash.rs:152,159,167,184,201,216,226,256,266`; `service.rs:283` |
| `#[tokio::test]` (async) | 6 | `service.rs:318,338,376,437,475,512` |
| **Total tests** | **19** | — |
| `#[ignore = "requires Postgres"]` | 6 | `service.rs:319,339,377,438,476,513` — exactly the 6 async tests |
| Tests runnable without infra | 13 | all `#[test]` |

Patterns:
- Fixture builders instead of literals: `sample_entry()` (`hash.rs:125-141`),
  `new_entry()` (`service.rs:307-315`), `make_community()` which inserts an FK-satisfying
  `communities` row with a unique host (`service.rs:296-305`).
- Intent-naming helpers that document the scenario: `nanosecond_instant()`
  (`hash.rs:145-147`), `after_database_round_trip()` (`hash.rs:150-152`).
- Infra tests degrade gracefully rather than fail when Postgres is absent:
  `PgPool::connect(...).await.ok()` then `let Some(pool) = ... else { return; }`
  (`service.rs:275-280`, used at `:321-323` etc.).
- Shared-table serialization via a `static OnceLock<tokio::sync::Mutex<()>>` guard taken
  at the top of each DB test (`service.rs:263-267`, `_g = db_lock().lock().await` at
  `:320`, `:340`, `:378`, `:439`, `:477`, `:514`).
- Tampering is simulated with raw SQL `UPDATE`/`INSERT` against the table
  (`service.rs:459-465`, `:492-505`) — testing detection, not the API.
- `matches!` assertions on error variants (`service.rs:472`, `:509`).
- Test doc comments state the property under test (`service.rs:281-282`, `:369-372`,
  `:475-477`).
- One unit test exists specifically so a regression is caught by `just test-unit`
  rather than only by the ignored tests (`service.rs:281-292`).

### Naming/typing convention at the tenant boundary

Input uses the newtype (`NewAuditEntry.community_id: CommunityId`, `entry.rs:57`);
storage and read-back use the raw `Uuid` (`entry.rs:16`, `service.rs:246`); the
conversion happens once, explicitly, at the DB boundary with a comment marking it
(`service.rs:89-91`). Query methods take `CommunityId` and bind `community.as_uuid()`
(`service.rs:162`, `:175`, `:214`, `:228`).


## Module: buzz-media (`crates/buzz-media`)

### Aspect: Conventions

### 1. Module organization

Flat, one concern per file, all modules `pub` and re-exported selectively (`crates/buzz-media/src/lib.rs:5-28`):

| File | LOC | Concern |
|---|---|---|
| `src/lib.rs` | 29 | module declarations + curated re-exports |
| `src/types.rs` | 31 | wire response type (`BlobDescriptor`) |
| `src/thumbnail.rs` | 51 | sync CPU-bound derived artifacts |
| `src/config.rs` | 187 | config struct + startup validation |
| `src/error.rs` | 198 | error enum + HTTP mapping |
| `src/storage.rs` | 404 | S3 client and key builders |
| `src/upload_record.rs` | 419 | moderation side-channel records |
| `src/auth.rs` | 552 | Blossom kind-24242 verification |
| `src/upload.rs` | 732 | upload pipelines (orchestration) |
| `src/bucket_index.rs` | 755 | key taxonomy + pure accounting fold |
| `src/validation.rs` | 2594 | all content validation |
| `tests/static_creds_minio.rs` | 75 | live MinIO round-trip (`#[ignore]`) |

Layering is one-directional: `upload.rs` orchestrates and depends on `auth`, `config`, `error`, `storage`, `thumbnail`, `types`, `upload_record`, `validation` (`crates/buzz-media/src/upload.rs:1-20`); `validation.rs` and `bucket_index.rs` have no S3 dependency at all (`crates/buzz-media/src/bucket_index.rs:4-6`).

Notable structural convention: `bucket_index.rs` is explicitly written as I/O-free pure logic with an injected page-fetcher so it can be driven by synthetic listings in tests (`crates/buzz-media/src/bucket_index.rs:4-14`).

---

### 2. Naming

| Pattern | Examples |
|---|---|
| `validate_*` for fail-closed checks returning `Result` | `validate_content`, `validate_file_content`, `validate_video_file`, `validate_image_metadata_free`, `validate_jpeg_metadata_free`, `validate_mp4_metadata_free` (`crates/buzz-media/src/validation.rs:159`, `:238`, `:289`, `:492`, `:502`, `:831`) |
| `looks_like_*` for boolean structural probes | `looks_like_iso_bmff`, `looks_like_mp4_iso_bmff` (`crates/buzz-media/src/validation.rs:48`, `:52`) |
| `verify_blossom_*` for auth | `verify_blossom_auth_event_for_verb`, `verify_blossom_auth_event`, `verify_blossom_upload_auth`, `verify_blossom_get_auth` (`crates/buzz-media/src/auth.rs:31`, `:147`, `:175`, `:207`) |
| `process_*_upload` for pipelines | `process_upload`, `process_file_upload`, `process_video_upload` (`crates/buzz-media/src/upload.rs:207`, `:245`, `:292`) |
| `*_key` for object-key builders | `sidecar_key`, `ctx_sidecar_key`, `upload_record_key` (`crates/buzz-media/src/storage.rs:183`, `:188`, `crates/buzz-media/src/upload_record.rs:181`) |
| `parse_*` returning `Option` for lenient parses | `parse_public_ip`, `parse_port`, `parse_thumb_key`, `parse_blob_key`, `parse_sidecar_key`, `parse_auxiliary_key`, `parse_canonical_uuid` (`crates/buzz-media/src/upload_record.rs:191`, `:197`, `crates/buzz-media/src/bucket_index.rs:129`-`:172`, `:112`) |
| `is_*` predicates | `is_sha256`, `is_blob_ext`, `is_ulid_charset`, `is_public_ip`, `is_snapshot_text_chunk` (`crates/buzz-media/src/bucket_index.rs:75`, `:84`, `:93`, `crates/buzz-media/src/upload_record.rs:207`, `crates/buzz-media/src/validation.rs:584`) |
| `_sync` suffix marks CPU-bound functions meant for `spawn_blocking` | `generate_image_metadata_sync` (`crates/buzz-media/src/thumbnail.rs:15`) |
| SCREAMING_SNAKE consts for policy tables and bounds | `ALLOWED_MIME_TYPES`, `BLOCKED_FILE_MIME_TYPES`, `MP4_BRANDS`, `PNG_SNAPSHOT_KEYWORDS`, `MAX_PIXELS`, `MAX_ATOMS`, `MAX_BOXES`, `MAX_BOX_DEPTH`, `MIN_SNIFF_BYTES`, `BUF`, `UPLOAD_RECORD_VERSION` |

---

### 3. Error handling

Single crate error enum `MediaError` (`crates/buzz-media/src/error.rs:8-86`), `thiserror`-derived, 35 variants, plus a separate `SweepError` for the accounting fold (`crates/buzz-media/src/bucket_index.rs:341-362`).

| Variant | Message | HTTP status |
|---|---|---|
| `UnknownContentType` | `unknown content type` | 415 |
| `DisallowedContentType(String)` | `disallowed content type: {0}` | 415 |
| `FileTooLarge { size: u64, max: u64 }` | `file too large: {size} bytes (max {max})` | 413 |
| `ImageTooLarge` | `image dimensions too large` | 413 |
| `InvalidImage` | `invalid image data` | 422 |
| `MetadataForbidden` | `media contains metadata or a non-canonical metadata channel` | 422 |
| `InvalidSignature` | `invalid signature` | 401 (generic) |
| `InvalidAuthKind` | `invalid auth event kind` | 401 |
| `InvalidAuthVerb` | `invalid auth verb` | 401 |
| `MissingTag(&'static str)` | `missing required tag: {0}` | 401 |
| `HashMismatch` | `hash mismatch` | 401 |
| `ServerMismatch` | `server mismatch` | 401 |
| `TokenExpired` | `token expired` | 401 |
| `TimestampOutOfWindow` | `timestamp out of window` | 401 |
| `StorageError(String)` | `storage error: {0}` | 500 (body flattened) |
| `Internal` | `internal error` | 500 |
| `NotFound` | `not found` | 404 |
| `MissingAuth` | `missing authorization header` | 401 |
| `InvalidAuthScheme` | `invalid authorization scheme` | 401 |
| `InvalidBase64` | `invalid base64 encoding` | 401 |
| `InvalidAuthEvent` | `invalid auth event` | 401 |
| `Unauthorized` | `unauthorized` | 401 |
| `InsufficientScope` | `insufficient scope` | 403 |
| `RelayMembershipRequired` | `relay membership required` | 403 |
| `TokenRevoked` | `token revoked` | 401 |
| `PubkeyMismatch` | `pubkey mismatch` | 401 |
| `UploadRateLimitExceeded` | `upload rate limit exceeded` | 429 |
| `UploadConcurrencyLimitReached` | `upload concurrency limit reached` | 429 |
| `WrongCodec` | `unsupported media codec: only H.264 video and AAC audio are accepted` | 415 |
| `DurationTooLong` | `video too long: duration exceeds 600 seconds` | 422 |
| `ResolutionTooHigh` | `video resolution too high: maximum is 3840x2160` | 422 |
| `MoovNotAtFront` | `moov atom not at front of file (not fast-start)` | 422 |
| `UnsupportedContainer` | `unsupported container: only MP4 is accepted` | 415 |
| `InvalidVideo` | `invalid video data` | 422 |
| `Io(String)` | `io error: {0}` | 500 |

Conventions visible in the mapping (`crates/buzz-media/src/error.rs:106-160`):
- All 15 authentication-failure variants collapse to a single `401 "authentication failed"` body, explicitly "to prevent oracle enumeration"; `InsufficientScope` stays 403 because it is authorization, not authentication (`crates/buzz-media/src/error.rs:120-146`).
- 5xx bodies are flattened to `"internal error"` and logged at `error` (`crates/buzz-media/src/error.rs:154-158`).
- Errors are converted, never `unwrap`ped: three `From` impls (`image::ImageError` → `InvalidImage`, `S3Error` → `StorageError`, `serde_json::Error` → `StorageError`) at `crates/buzz-media/src/error.rs:88-104`.
- `MediaConfig::validate` returns `Result<(), String>` (plain strings, not `MediaError`) because it is a startup check (`crates/buzz-media/src/config.rs:66`).
- Some variants are declared here but not constructed in this crate (`Unauthorized`, `TokenRevoked`, `PubkeyMismatch`, `RelayMembershipRequired`, `MissingAuth`, `InvalidAuthScheme`, `InvalidBase64`, `UploadRateLimitExceeded`, `UploadConcurrencyLimitReached`) — they exist for relay handlers that share the type; see the Debt aspect.

---

### 4. Async patterns

| Pattern | Usage |
|---|---|
| CPU-bound work always inside `tokio::task::spawn_blocking` | validation+hash+auth (`crates/buzz-media/src/upload.rs:79-89`), video auth (`crates/buzz-media/src/upload.rs:410-414`), MP4 validation (`crates/buzz-media/src/upload.rs:416-419`), thumbnail (`crates/buzz-media/src/upload.rs:518-524`) |
| Join errors mapped, never unwrapped | `.map_err(\|_\| MediaError::Internal)??` (double `?` over join + inner result) at `crates/buzz-media/src/upload.rs:87-88`, `:414`, `:419`, `:524` |
| Owned inputs cloned into blocking closures; `Bytes` clones are refcount bumps (documented) | `crates/buzz-media/src/upload.rs:193-201` |
| Generic closure injection instead of trait objects for the two variable steps of the buffered pipeline | `crates/buzz-media/src/upload.rs:54-63` |
| Streaming rather than buffering for large payloads | `StreamReader` + 64 KiB chunks to temp file (`crates/buzz-media/src/upload.rs:325-395`), 8 MiB `BufReader` upload (`crates/buzz-media/src/storage.rs:91-101`), `ByteStream` download (`crates/buzz-media/src/storage.rs:131-146`) |
| Sync-blocking `std::fs` used deliberately inside blocking contexts | `crates/buzz-media/src/validation.rs:295`, `:415`, `:921` |
| Async page fetcher expressed as `FnMut(Option<String>) -> Future` | `crates/buzz-media/src/bucket_index.rs:377-383` |

---

### 5. Documentation conventions

- Every file opens with a `//!` module doc (`crates/buzz-media/src/lib.rs:1-3`, `crates/buzz-media/src/upload_record.rs:1-48`, `crates/buzz-media/src/bucket_index.rs:1-19`).
- All public items carry `///` doc comments; several document *why* a rule exists, including consumer contracts (`crates/buzz-media/src/upload_record.rs:29-48`) and threat rationale (`crates/buzz-media/src/validation.rs:66-86`).
- A markdown table inside module docs describes the key taxonomy (`crates/buzz-media/src/bucket_index.rs:12-19`).
- Spec references are inline: BUD-01 (`crates/buzz-media/src/auth.rs:201-205`, `crates/buzz-media/src/storage.rs:378`), BUD-02 (`crates/buzz-media/src/types.rs:1`), BUD-11 §5/§6 (`crates/buzz-media/src/auth.rs:47`, `:124`, `:189`).

---

### 6. Testing patterns

| Metric | Count |
|---|---|
| `#[test]` (sync) in `src/` | **98** |
| `#[tokio::test]` | **6** (5 in `crates/buzz-media/src/bucket_index.rs:538-661`, 1 in `crates/buzz-media/tests/static_creds_minio.rs:44`) |
| Total tests | **104** |
| `#[ignore]` | **1** — `crates/buzz-media/tests/static_creds_minio.rs:45` (`"requires a live MinIO (docker compose up -d minio minio-init)"`) |
| Integration test files | 1 (`tests/static_creds_minio.rs`) |
| Binary fixtures | 12 PNG/JPEG files under `crates/buzz-media/tests/fixtures/{android,ios}` |

Per-file distribution: `validation.rs` 47, `bucket_index.rs` 16 + 5 async, `auth.rs` 14, `upload_record.rs` 7, `config.rs` 4, `storage.rs` 4, `upload.rs` 4, `error.rs` 2; `lib.rs`, `types.rs`, `thumbnail.rs` have **zero**.

Patterns:
- `#[cfg(test)] mod tests` at the bottom of each file with a local `test_config()`/`valid_config()`/`storage_config()` builder (`crates/buzz-media/src/validation.rs:945-983`, `crates/buzz-media/src/config.rs:128-145`, `crates/buzz-media/src/storage.rs:281-301`, `crates/buzz-media/src/upload.rs:566-584`).
- Hand-built binary fixtures as consts (`TINY_JPEG`, `TINY_PNG`, `TINY_GIF`, `MP4_FTYP_MAGIC`, `TINY_PDF`, `TINY_ZIP`) plus ~30 MP4 box-builder helpers (`crates/buzz-media/src/validation.rs:1651-2150`).
- Real-device fixtures compiled in with `include_bytes!` to pin the mobile-sanitizer contract (`crates/buzz-media/src/validation.rs:1164-1280`).
- "Prove the fixture" pattern: an independent parser asserts the fixture really contains GPS EXIF before the validator is exercised (`crates/buzz-media/src/validation.rs:1044-1075`).
- Negative-first assertions via `matches!(result, Err(MediaError::X))` (`crates/buzz-media/src/validation.rs:1298-1305`).
- Mutation-resistance tests named for the property they defend, e.g. `same_sha_sidecars_do_not_bleed_between_communities` (`crates/buzz-media/src/storage.rs:352-377`) and `malformed_uploads_key_is_unknown_not_auxiliary` (`crates/buzz-media/src/bucket_index.rs:490-506`).
- Async tests drive the sweep fold through canned `Page`s and an `Arc<Mutex<Vec<Page>>>` script (`crates/buzz-media/src/bucket_index.rs:551-600`).
- Env-overridable integration config (`BUZZ_S3_ENDPOINT`/`ACCESS_KEY`/`SECRET_KEY`/`BUCKET`) in the ignored test only (`crates/buzz-media/tests/static_creds_minio.rs:22-34`).

---

### 7. Repo-guideline compliance

| Guideline (AGENTS.md) | Status in this crate |
|---|---|
| No `unsafe` | Satisfied — zero occurrences of `unsafe` across all 12 files |
| No new `unwrap()`/`expect()` in production paths | Mostly satisfied: 9 remaining non-test `.unwrap()` calls, all `try_into()` on fixed-size slices behind explicit length checks (`crates/buzz-media/src/validation.rs:604`, `:605`, `:673`, `:674`, `:697`, `:706`, `:707`, `:885`, `:886`); plus one `unwrap_or_default()` that intentionally swallows blurhash failure (`crates/buzz-media/src/thumbnail.rs:37`) |
| Public API must have doc comments | Satisfied for all public fns/types reviewed |


## Module: buzz-workflow (`crates/buzz-workflow`)

### Aspect: Conventions

---

### 1. Module organization

| File | LOC | Role |
|---|---|---|
| `src/lib.rs` | 1564 | crate root: `WorkflowConfig`, `WorkflowEngine`, event hook, scheduler loop, trigger matching, `build_trigger_context` |
| `src/schema.rs` | 878 | definition types + validation + YAML parsing |
| `src/executor.rs` | 1834 | templates, conditions, action dispatch, HTTP impls, step loop |
| `src/error.rs` | 66 | `WorkflowError`, `PartialProgress` |
| `src/action_sink.rs` | 69 | `ActionSink` trait + `ActionSinkError` |

Flat module layout — four sibling modules, all declared `pub` at `lib.rs:33-36`, with the commonly used types re-exported at the crate root (`lib.rs:38-41`) so callers write `buzz_workflow::WorkflowDef` rather than `buzz_workflow::schema::WorkflowDef`.

Crate-level lints: `#![deny(unsafe_code)]` and `#![warn(missing_docs)]` (`lib.rs:1-2`); every public item carries a doc comment, and `///` docs are also used on private helpers and enum fields.

---

### 2. Naming

| Convention | Examples |
|---|---|
| Types `UpperCamelCase`, suffix conveys role | `WorkflowDef`, `TriggerDef`, `ActionDef`, `TriggerContext`, `ExecutionResult`, `StepResult`, `PartialProgress`, `WorkflowError`, `ActionSinkError` |
| Serde variant naming | `rename_all = "snake_case"` on both tagged enums so Rust `SendChannelTopic`-style variants map to YAML `set_channel_topic` (`schema.rs:37`, `schema.rs:91`) |
| Keyword collisions | Rust field `if_expr` renamed to YAML `if` (`schema.rs:79-80`) |
| Verb-first functions | `parse_yaml`, `validate_cron`, `normalize_cron`, `resolve_template`, `resolve_variable`, `resolve_step_templates`, `build_eval_context`, `build_trigger_context`, `evaluate_condition`, `dispatch_action`, `execute_run`, `execute_steps`, `execute_from_step`, `finalize_run` |
| Predicate functions prefixed `is_`/`should_`/`_matches_` | `interval_should_fire`, `interval_prefilter_should_fire`, `should_fire_workflow`, `trigger_matches_event` |
| `_impl` suffix for feature-gated internals | `call_webhook_impl`, `add_reaction_impl` (`executor.rs:781`, `executor.rs:888`) |
| Constants `SCREAMING_SNAKE_CASE`, unit in the name | `EVAL_TIMEOUT`, `MAX_EXPR_LEN`, `MAX_DELAY_SECS`, `WEBHOOK_MAX_RESPONSE_BYTES` |
| Deliberately unused params prefixed `_` | `generate_approval_token(_run_id, _step_id)` (`executor.rs:698`) |
| evalexpr variable mangling | dots → underscores: `trigger_text`, `steps_{id}_output_{field}` (documented as a table in the fn doc, `executor.rs:207-217`) |

---

### 3. Error handling

Single crate error enum `WorkflowError` (`error.rs:18-60`), `thiserror`-derived, all variants documented:

| Variant | Payload | `#[error]` message | Line |
|---|---|---|---|
| `InvalidYaml` | `#[from] serde_yaml::Error` | `invalid YAML: {0}` | `error.rs:20-22` |
| `InvalidDefinition` | `String` | `invalid definition: {0}` | `error.rs:24-26` |
| `ConditionError` | `String` | `condition evaluation error: {0}` | `error.rs:28-30` |
| `TemplateError` | `String` | `template error: {0}` | `error.rs:32-34` |
| `StepTimeout` | `{ step_id: String, timeout_secs: u64 }` | `step '{step_id}' timed out after {timeout_secs}s` | `error.rs:36-43` |
| `WebhookError` | `String` | `webhook error: {0}` | `error.rs:45-47` |
| `CapacityExceeded` | — | `capacity exceeded` | `error.rs:49-51` |
| `Database` | `String` | `database error: {0}` | `error.rs:53-55` |
| `NotImplemented` | `String` | `action not implemented: {0}` | `error.rs:57-59` |

Companion `ActionSinkError` (`action_sink.rs:12-32`) has 6 variants (`InvalidInput`, `ChannelNotFound`, `ChannelArchived`, `EventBuild`, `Database`, `EmptyContent`) and collapses into `WorkflowError::WebhookError` (`action_sink.rs:34-38`).

Patterns:
- `?` with `map_err` closures everywhere; no `unwrap()`/`expect()` in production paths except `LazyLock` client construction `expect("HTTP client build must succeed")` (`executor.rs:882`) and `parts.next().unwrap_or("")` style safe fallbacks (`executor.rs:99`).
- Fallible operations that must not abort a batch use "log-and-continue": `tracing::warn!`/`error!` then `continue` (`lib.rs:333-336`, `lib.rs:466-469`, `lib.rs:604-610`).
- Partial results are first-class: `Result<ExecutionResult, (WorkflowError, PartialProgress)>` is the executor's return type so trace data survives failure (`executor.rs:975`, `executor.rs:1088`).
- One deliberate panic: double `set_action_sink` (`lib.rs:139-143`), documented with `# Panics`.
- `WorkflowError::WebhookError` is overloaded — it also carries `send_message` DB-lookup failures (`executor.rs:539-541`, `executor.rs:548-551`) and SSRF/DNS failures (`executor.rs:757-763`).

---

### 4. Async patterns

| Pattern | Usage |
|---|---|
| Detached background execution | `tokio::spawn` for each triggered run, both event and cron paths (`lib.rs:371-381`, `lib.rs:649-661`) |
| Non-blocking admission | `Semaphore::try_acquire()` (no `acquire().await`), permit held in a `_permit` binding for the run's lifetime (`executor.rs:978`, `executor.rs:1028`) |
| CPU/blocking isolation | `spawn_blocking` for evalexpr evaluation (`executor.rs:372`) and for the blocking DNS resolver (`executor.rs:747-755`) |
| Timeouts | `tokio::time::timeout` around expression evaluation (`executor.rs:370`) and around each `dispatch_action` (`executor.rs:1139-1151`) |
| Sleep-based loop | `loop { sleep(60s).await; … }` scheduler with sleep-first ordering (`lib.rs:430-432`) |
| `Arc<Self>` receivers | `on_event(self: &Arc<Self>)` and `run(self: &Arc<Self>)` so spawned tasks can clone the engine without `'static` on `&self` (`lib.rs:276-279`, `lib.rs:428`) |
| Late init without `Mutex` | `OnceLock<Arc<dyn ActionSink>>` (`lib.rs:90`) |
| dyn-compatible async trait | manual `Pin<Box<dyn Future … + Send + '_>>` return instead of `async_trait` (`action_sink.rs:60-70`) |
| Lock-free shared state | `DashMap` for interval anchors, `moka::sync::Cache` for workflow lookups (`lib.rs:87`, `lib.rs:104`) |
| Sync-in-async caution | `evaluate_condition` is `async` purely to host the timeout; `resolve_template`/`build_eval_context` stay synchronous |

---

### 5. Testing patterns

All tests are inline `#[cfg(test)] mod tests` blocks: `schema.rs:270`, `lib.rs:966`, `executor.rs:1219`. No `tests/` directory, no fixtures directory, no mocking crate.

| File | `#[test]` | `#[tokio::test]` | Total |
|---|---|---|---|
| `schema.rs` | 50 | 0 | 50 |
| `lib.rs` | 38 | 0 | 38 |
| `executor.rs` | 39 | 22 | 61 |
| `error.rs` | 0 | 0 | 0 |
| `action_sink.rs` | 0 | 0 | 0 |
| **Total** | **127** | **22** | **149** |

Conventions observed:
- YAML fixtures are inline `&str` literals built with `concat!` or `\n`-escaped strings, with a comment explaining why raw strings are avoided (`schema.rs:275-276`, `schema.rs:326-328`).
- Error assertions use `matches!(err, WorkflowError::Variant(_))` plus substring checks on the message (`schema.rs:404-407`, `schema.rs:428-436`).
- Deterministic time: fixed RFC-3339 instants parsed for cron/interval tests (`lib.rs:969-985`, `lib.rs:1140-1166`) alongside `Utc::now()`-relative tests for elapsed-interval logic (`lib.rs:1168-1235`).
- Pure-function extraction for testability: `interval_prefilter_should_fire` is a free function over `&DashMap` explicitly "so it is unit-testable without a `Db`/Postgres" (`lib.rs:777-782`).
- Shared builders instead of a framework: `make_trigger()` (`executor.rs:1223-1233`), `make_message_event()` (`lib.rs:1338-1347`), `make_reaction_event()` (`lib.rs:1350-1371`).
- Regression tests carry intent comments naming the bug they lock down (`lib.rs:1211-1216`, `executor.rs:1812-1814`, `schema.rs:756-758`).
- Nothing that requires Postgres or an `ActionSink` is unit-tested — no test constructs a `WorkflowEngine`, so `on_event`, `run`, `execute_run`, `execute_from_step`, `execute_steps`, `dispatch_action`, and `finalize_run` have zero unit coverage in this crate.

---

### 6. Documentation conventions

- Module-level `//!` headers with a Responsibilities list (`executor.rs:1-11`) and an Architecture/Usage section including a `rust,ignore` example (`lib.rs:3-31`).
- Markdown tables inside doc comments to specify mappings (`executor.rs:207-217`).
- Long rationale comments attached to consistency-critical fields — the workflow cache's cross-pod invalidation trade-off (`lib.rs:92-103`) and the interval cold-start liveness argument (`lib.rs:385-399`).
- Ticket-tagged deferrals: `WF-07`, `WF-08`, `WF-09` (`executor.rs:582`, `:588`, `:663`, `:675`; `lib.rs:192`).
- Numbered "Fix N" comments preserving review history (`schema.rs:218`, `lib.rs:465`, `lib.rs:572`, `lib.rs:638`, `lib.rs:664`).


## Module: buzz-relay — core & bootstrap (`crates/buzz-relay/src`)
### Aspect: Conventions

---

#### 1. Crate-level lint policy

| Attribute | Line | Effect |
|-----------|------|--------|
| `#![deny(unsafe_code)]` | `lib.rs:1` | zero `unsafe` blocks in the crate; verified — the only textual `unsafe` in this group is a comment (`main.rs:1013`) |
| `#![warn(missing_docs)]` | `lib.rs:2` | warn, not deny — an undocumented public item compiles |
| crate `//!` doc | `lib.rs:3` | placed after the inner attributes (valid) |

Local `#[allow]` count in this group: **6**, all narrowly scoped, no blanket allows.

| Allow | Line | Justification present? |
|-------|------|------------------------|
| `clippy::too_many_arguments` on `ConnectionManager::register` | `state.rs:204` | yes — `state.rs:202-203` explains a params struct would only relocate the fields |
| `clippy::type_complexity` on `membership_cache` | `state.rs:544` | no |
| `clippy::type_complexity` on `accessible_channels_cache` | `state.rs:548` | no |
| `clippy::type_complexity` on `observer_owner_cache` | `state.rs:606` | no |
| `clippy::too_many_arguments` on `AppState::new` (10 params) | `state.rs:636` | no |
| `clippy::type_complexity` on the NIP-11 fence const | `nip11.rs:328` | implicit (the whole const is a deliberate signature pin) |

#### 2. Module layout convention

`lib.rs` declares 21 modules. Every `pub mod` carries a one-line `///` doc **except** `storage_sweep` (`lib.rs:43`) — the only inconsistency. `storage_sweep.rs` does have `//!` inner docs (`storage_sweep.rs:1-5`) so `missing_docs` is satisfied, but the `lib.rs` listing reads as an oversight.

Exactly **one** private module: `mod admission;` (`lib.rs:5`). It is also the only file in the group with **no module-level doc comment at all** — `admission.rs:1` is a `use` statement. Every other file starts with `//!`:

| File | Module doc | Line |
|------|-----------|------|
| `state.rs` | "Shared application state — Arc-wrapped, shared across all connections." | `:1` |
| `config.rs` | "Relay configuration from environment variables." | `:1` |
| `router.rs` | "axum routers — app (WebSocket + REST), health (K8s probes), metrics (Prometheus)." | `:1` |
| `nip11.rs` | "NIP-11 relay information document." | `:1` |
| `protocol.rs` | "NIP-01 client/relay message parsing and formatting." | `:1` |
| `tenant.rs` | 15-line conformance narrative ("row zero") | `:1-15` |
| `telemetry.rs` | ASCII architecture diagram + honoured env vars | `:1-32` |
| `metrics.rs` | ASCII architecture diagram | `:1-19` |
| `error.rs` | "Error types for the relay crate." | `:1` |
| `admission.rs` | **none** | — |

**ASCII-diagram convention**: three files open with a boxed ASCII diagram of their data flow — `metrics.rs:3-12`, `telemetry.rs:3-15`, and the `serve` fn doc (`main.rs:1097-1112`). This is a distinctive house style in the relay core.

#### 3. Error-handling convention — three coexisting styles

| Layer | Style | Cite |
|-------|-------|------|
| `main()` | `anyhow::Result` with `.map_err(|e| anyhow!("…: {e}"))` and an `error!` log immediately before returning | `main.rs:83`, `main.rs:122-127`, `main.rs:151-155` |
| `config.rs` | `thiserror` enum `ConfigError` with 2 variants | `config.rs:19-27` |
| `protocol.rs` | `thiserror` enum `RelayError` + `type Result<T>` alias | `error.rs:8-49`, used only at `protocol.rs:6` |
| handlers (outside group) | `axum::response::Response` / `IntoResponse` tuples, no shared error type | `router.rs:295-301`, `router.rs:304-375` |

`RelayError` is the crate's nominal error type but is effectively single-purpose: only `InvalidMessage` is constructed, only `protocol.rs` imports it (see debt). There is no `impl IntoResponse for RelayError`, so HTTP handlers cannot use it — that is why the third style exists.

Fallibility conventions actually followed:
- Fatal-at-boot: `.map_err(anyhow!)?` (11 sites in `main.rs`).
- Non-fatal-at-boot: `match … { Err(e) => error!(…) }` with an explicit "(non-fatal…)" phrase in the message — a consistent, greppable convention (`main.rs:174`, `:190-195`, `:253`, `:286`, `:305`, `:319`).
- Conditional fatality is always written as `if config.require_relay_membership { … return Err(…) } else { error!(…) }` (`main.rs:250-262`, `:275-288`, `:295-309`).

#### 4. `unwrap`/`expect`/panic conventions

**26 production `unwrap()`/`expect()` calls** in this file group (outside `#[cfg(test)]`), plus 1 `panic!`, 1 `unreachable!`, 1 `debug_assert!`, 1 `std::process::exit`.

| File | Count | Lines |
|------|-------|-------|
| `metrics.rs` | 17 | `:79, 84, 89, 94, 99, 104, 109, 114, 119, 124, 129, 134, 136, 141, 143, 145` (`expect`) + `:180` (`unwrap`) |
| `main.rs` | 4 | `:90, 401, 1230, 1253` |
| `state.rs` | 3 | `:446, 701, 708` |
| `config.rs` | 1 | `:507` |
| `protocol.rs` | 1 | `:189` |
| `router.rs`, `nip11.rs`, `tenant.rs`, `telemetry.rs`, `admission.rs`, `lib.rs`, `error.rs` | 0 | — |

Convention: `expect` messages are written as **assertions about an invariant**, not as error text — `"hardcoded dev key is valid"` (`main.rs:401`), `"relay fan-out frames are serialized UTF-8 JSON"` (`state.rs:446`), `"media storage was already constructed with this S3 config"` (`state.rs:701`), `"metrics exporter must build exactly once"` (`metrics.rs:143`), `"SAFETY: nostr::Event serialization is infallible for well-formed events"` (`protocol.rs:189`). Two carry an inline `// safe:` comment (`metrics.rs:180`, `config.rs:507`).

This contradicts AGENTS.md's "Do not introduce new `unwrap()` or `expect()` in production paths". The 17 in `metrics.rs` are all boot-time bucket registration where the argument is a compile-time literal array, so they are provably-infallible; still, they are production `expect`s.

Other panic-shaped exits:
- `panic!` at `main.rs:409` when `BUZZ_REQUIRE_AUTH_TOKEN=true` and no relay key — a deliberate hard stop, but inconsistent with the surrounding `return Err(anyhow!(…))` style used for the two neighbouring preconditions (`main.rs:206-211`, `:216-219`).
- `unreachable!("mesh handle is set exactly once, right here")` at `main.rs:460`.
- `debug_assert!` at `nip11.rs:143` — release builds skip the NIP-43/`self` consistency check.
- `std::process::exit(1)` at `main.rs:1153` — the only non-`main`-return process exit.

#### 5. Logging / tracing conventions

| Convention | Cite |
|-----------|------|
| JSON-only structured logs, `flatten_event(true)` | `main.rs:109` |
| Filter is `RUST_LOG` **plus** a forced `buzz_relay=info` directive | `main.rs:111` |
| OTLP layer attached only when the endpoint env var is set | `main.rs:101-107` |
| Structured fields preferred over interpolation: `info!(bind_addr = %…, relay_url = %…, …, "Config loaded")` | `main.rs:128-136` |
| `%` for `Display`, `?` for `Debug` — used consistently (`error = %e`, `error = ?e`) | `main.rs:119`, `main.rs:558`, `state.rs:886` |
| Sentence-case, no trailing period, human-readable message last | `main.rs:157`, `main.rs:197`, `state.rs:1185` |
| Startup progress logged as a linear narrative ("Postgres connected", "Redis pub/sub connected", "Media storage connected", "…listener started") | `main.rs:157-159`, `:348`, `:421`, `:1117`, `:1155` |
| Two logging import styles coexist: `use tracing::{error, info, warn}` at `main.rs:5` **and** fully-qualified `tracing::warn!`/`tracing::info!`/`tracing::error!` at `main.rs:404`, `:483`, `:513`, `:837`, `:1152` and throughout `state.rs` | mixed |

`state.rs` never imports the tracing macros; it always fully-qualifies (`state.rs:463`, `:472`, `:886`, `:1185`, …). `main.rs` does both. No lint enforces either.

#### 6. Test conventions

**Test counts in this file group** (`#[test]` + `#[tokio::test]`):

| File | Tests | `#[ignore]` | Notes |
|------|-------|-------------|-------|
| `config.rs` | 22 | 0 | serialized by a module-level `ENV_MUTEX` (`config.rs:931-934`) with an explicit rationale about flakiness |
| `state.rs` | 18 | 0 | 5 need a live Postgres/Redis via `test_state()` (`state.rs:1257-1290`) |
| `nip11.rs` | 15 | 0 | deliberately test the `SUPPORTED_NIPS` **constant** rather than `Config::from_env()` to avoid the env race (`nip11.rs:358-360`) |
| `tenant.rs` | 10 | 0 | includes a `redteam_attack2` sub-module (`tenant.rs:249-332`) |
| `protocol.rs` | 7 | 0 | table-driven with `Box<dyn Fn>` case tables and `type ParseCase`/`FormatCase` aliases to dodge `clippy::type_complexity` (`protocol.rs:230-232`, `:378-380`) |
| `main.rs` | 7 | 0 | includes `#[tokio::test(start_paused = true)]` for the cancellation loop (`main.rs:1827`) |
| `telemetry.rs` | 5 | 0 | `ENV_LOCK` mutex (`telemetry.rs:119`) |
| `router.rs` | 4 | 0 | one spins a real TCP + tungstenite client (`router.rs:463-501`) |
| `admission.rs` | 4 | 0 | stub `RateLimiter` impl (`admission.rs:64-95`) |
| `metrics.rs` | 0 | 0 | **no tests at all** |
| `lib.rs`, `error.rs` | 0 | 0 | no logic |
| **Total** | **92** | **0** | |

Conventions observed:
- **Env-mutation serialization**: two independent static mutexes, `config.rs:934` (`ENV_MUTEX`) and `telemetry.rs:119` (`ENV_LOCK`), both with a comment explaining process-global env races. `nip11.rs` avoids the problem structurally instead.
- **Save/restore around env mutation**: `let previous = std::env::var_os(…); … if let Some(v) = previous { set_var } else { remove_var }` — used at `config.rs:990-1000`, `:1015-1030`, `:1043-1055`, `:1058-1070`, `:1076-1086`, `:1192-1210`. Not used in `config.rs:1092-1105` or `:1109-1113` (rate-limit tests just `remove_var` afterwards) — inconsistent.
- **Red-team gate tests**: `tenant.rs:249-332` names its module `redteam_attack2`, documents the RED-then-GREEN cycle, and includes a *negative control* (`tenant.rs:326-331`). Notable convention.
- **Named-case tables**: `protocol.rs:234-273` and `:382-448` use `&[(name, Box<dyn Fn>)]` slices.
- **Behaviour-named test fns**: e.g. `disconnect_pubkey_is_fenced_to_the_banning_community` (`state.rs:1737`), `register_after_drain_self_signals_restart_close_and_cancel` (`state.rs:1881`), `relay_websocket_parser_rejects_oversized_messages_before_handler_reads_them` (`router.rs:504`). Long, assertion-shaped names are the norm.
- **Assertion messages**: nearly every `assert!`/`assert_eq!` carries an explanatory third argument (`state.rs:1454`, `:1770`, `:1795`, `nip11.rs:432`, `tenant.rs:320`).
- **`#[should_panic(expected = …)]`** used once, to pin the `debug_assert` (`nip11.rs:513-517`).
- **Zero `#[ignore]`d tests** in the group. However `tenant.rs:225-236` and `:238-243` still instruct the reader to "Delete this `#[ignore]` when the fix lands" — the attributes were already removed and the fix landed (`tenant.rs:81-88`). Stale instructions (see debt).
- Integration-style tests that need infrastructure are inlined into `#[cfg(test)]` modules rather than `tests/`: `crates/buzz-relay/tests/` **does not exist**.

#### 7. Naming conventions

| Pattern | Examples |
|---------|----------|
| Env vars: `BUZZ_` prefix, except deliberate exceptions | `RELAY_URL`, `RELAY_OWNER_PUBKEY`, `RELAY_OPERATOR_PUBKEYS`, `RELAY_OPERATOR_API_ORIGIN` — rationale given at `config.rs:524-525` and `:548-551` ("relay-identity config that may be shared across multiple services"); plus `DATABASE_URL`/`READ_DATABASE_URL`/`REDIS_URL`/`AWS_REGION`/`RUST_LOG`/`OTEL_*` |
| Env vars: **`SPROUT_` legacy prefix survives in 3 vars** | `SPROUT_REMINDER_SCHEDULER_INTERVAL_SECS` (`main.rs:701`), `SPROUT_REMINDER_SCHEDULER_BATCH_LIMIT` (`main.rs:705`), `SPROUT_MAX_NOT_BEFORE_DELTA` (`nip11.rs:97`) — the repo has otherwise renamed to `buzz`/`BUZZ_` |
| Metrics: `buzz_` prefix, `_total` counters, `_seconds` histograms | `state.rs:466`, `:1201`, `:1205`, `main.rs:836` |
| Framework metrics keep the CAKE names without the `buzz_` prefix | `http_requests_total`, `http_request_latency_ms` (`metrics.rs:204-205`) |
| Cross-pod method pairs: public `X` publishes, `pub(crate) X_local` applies a received drop (so a received drop is never re-published) | `state.rs:850/862`, `:876/881`, `:899/906`, `:921/928` |
| Cache accessors: `*_cached` | `is_member_cached` (`state.rs:827`), `get_accessible_channel_ids_cached` (`state.rs:1089`), `channel_visibility_cached` (`state.rs:1124`) |
| Cluster-wide operations: `*_clusterwide` | `disconnect_pubkey_clusterwide` (`state.rs:1018`), `disconnect_community_clusterwide` (`state.rs:1056`) |
| Health/internal routes prefixed `_` | `/_liveness`, `/_readiness`, `/_status`, `/_mesh`, `/_mesh/demo/echo` — and `metrics.rs:170` skips `/_*` on that basis |

#### 8. Doc-comment conventions

- Long "why" narratives on invariant-bearing items are the norm, not the exception: `state.rs:296-308` (tenant fence on ban), `state.rs:336-350` (drain rationale), `state.rs:530-540` (community-keyed dedup), `state.rs:1106-1122` (fail-safe visibility caching), `config.rs:114-128` (huddle single-pod constraint), `nip11.rs:307-327` (static-input fence), `tenant.rs:1-15` (row zero), `main.rs:36-48` (metric-cardinality cost lever).
- Spec/plan cross-references are embedded in doc comments: `COMMUNITY_MODERATION_PLAN.md §0 decision 4` (`state.rs:301`), `docs/spec/MultiTenantRelay.tla` (`state.rs:616-619`, `lib.rs:14-18`), `docs/git-on-object-storage.md` (`state.rs:563`), `PLANS/S3_STORAGE_METRICS_PLAN.md` (`storage_sweep.rs:4`), `E1 §4.8` (`state.rs:1116`, `state.rs:1155`), `spec §Push step 7, Inv_NoFork` (`state.rs:519-520`), `plan §4 fork B` / `§5b` (`config.rs:118-122`).
- Invariant names are used as identifiers in prose: `Inv_RowZero` (`tenant.rs:70`, `:251`), `Inv_LabelPropagation` (`state.rs:1159`), `Inv_NoFork` (`state.rs:520`), `A3` (`main.rs:467`), `C4`/`K1` (`main.rs:1486`, `:1506`).
- **Compile-time doc enforcement**: `nip11.rs:329-335` turns a documented conformance obligation into a type-level fence. This is the only instance of the pattern in the group and is explicitly described as "the same way a deny-lint would" (`nip11.rs:322-324`).

#### 9. Config-parsing conventions (inconsistent — see business-rules BR-RC-109)

Three helper functions exist (`positive_u64_from_env` `config.rs:270`, `parse_bool` `config.rs:363`, `parse_optional_bool` `config.rs:379`) but most fields bypass them with an inline `std::env::var(...).ok().and_then(|v| v.parse().ok()).unwrap_or(default)` chain. Result: **eight distinct boolean grammars** and two distinct numeric-invalid policies (silent default vs hard error) inside one file. `parse_optional_bool` (`config.rs:379-381`) is a one-line wrapper that just calls `parse_bool(name, false)` and has a single caller (`config.rs:792`).

Numeric-invalid policy split:
- Hard error: `BUZZ_RATE_LIMIT_*` (`config.rs:270-282`), `BUZZ_PUSH_GATEWAY_TIMEOUT_MS` (`config.rs:759-773`).
- Silent default: `BUZZ_REDIS_POOL_SIZE`, `BUZZ_MAX_CONNECTIONS`, `BUZZ_MAX_CONCURRENT_HANDLERS`, `BUZZ_SEND_BUFFER`, `BUZZ_MAX_FRAME_BYTES`, `BUZZ_SLOW_CLIENT_GRACE_LIMIT`, `BUZZ_HEALTH_PORT`, `BUZZ_METRICS_PORT`, all `BUZZ_GIT_*` numerics, all `BUZZ_MAX_*_BYTES`, all `BUZZ_MEDIA_*` numerics, and every interval var read directly in `main.rs`. `redis_pool_size` has an explicit test asserting the silent-fallback behaviour (`config.rs:988-1011`).


## Module: buzz-relay — WebSocket protocol & subscriptions (`crates/buzz-relay/src`)
### Aspect: Conventions

---

#### 1. Handler signatures

Every WS message handler follows one shape — `(payload…, Arc<ConnectionState>, Arc<AppState>)`, returning `()`, replying through `conn.send`:

| Handler | Signature | Site |
|---|---|---|
| AUTH | `async fn handle_auth(event: nostr::Event, conn: Arc<ConnectionState>, state: Arc<AppState>)` | `auth.rs:44-49` |
| EVENT | `async fn handle_event(event: Event, conn: Arc<ConnectionState>, state: Arc<AppState>)` | `event.rs:608` |
| REQ | `async fn handle_req(sub_id: String, filters: Vec<Filter>, conn: Arc<ConnectionState>, state: Arc<AppState>)` | `req.rs:44-49` |
| COUNT | `async fn handle_count(sub_id: String, filters: Vec<Filter>, conn: Arc<ConnectionState>, state: Arc<AppState>)` | `count.rs:30-35` |
| CLOSE | `async fn handle_close(sub_id: String, conn: Arc<ConnectionState>, state: Arc<AppState>)` | `close.rs:12` |

Conventions that hold across all five:

1. **No `Result`.** Errors are terminal side effects (a `conn.send` + `return`), never propagated. There is not one `?` in a handler's top-level body.
2. **`Arc` by value, not `&`.** Uniform even for the two handlers that are awaited inline (`auth`, `close`) and never need `'static` — so a handler can be moved into a spawn without a signature change.
3. **Auth is extracted into locals inside a scoped block**, so the `RwLock` guard drops before any `.await` on I/O. `event.rs:634-654`, `req.rs:50-87`, `count.rs:37-51`, `auth.rs:45-74` all use the same `let (…) = { let auth = conn.auth_state.read().await; match &*auth { … } };` shape.
4. **Internal helper naming**: `handle_*` for message entry points, `handle_*_event` for sub-branches (`handle_ephemeral_event` `event.rs:762`, `handle_agent_observer_event` `event.rs:943`, `handle_search_req` `req.rs:504`), `*_authorized` for boolean filter gates, `filter_can_match_*` for capability predicates, `extract_*` for tag/field readers.
5. **Tracing**: `#[tracing::instrument(skip_all, fields(...))]` with `tracing::field::Empty` placeholders recorded once the values exist (`event.rs:607`, `:591-594`; `auth.rs:45`, `:77-80`). The dispatcher captures the span *before* `tokio::spawn` and attaches it with `.instrument(span)`, with an explicit comment on why (`connection.rs:522-523`).
6. **`#[allow(clippy::too_many_arguments)]`** is used rather than introducing param structs, with a stated rationale where non-obvious (`req.rs:504`, `state.rs:201-204`).

Free-function helper conventions:
- `pub(crate)` for cross-module helpers inside the crate (`req.rs:448`, `:483`, `:993`, `:1042`, `:1099`, `:1137`, `:1154`, `:1172`, `:1186`, `:1204`; `event.rs:35`, `:218`, `:326`).
- `pub` only where the HTTP bridge needs it (`req.rs:733` `build_event_query_from_filter`, `req.rs:773` `filter_fully_pushable`, `event.rs:115` `filter_fanout_by_access`, `event.rs:282` `fan_out_pubsub_event`).
- Private for file-local (`req.rs:829`, `:842`, `:860`, `:1013`, `:1225`; `event.rs:55`, `:59`, `:63`, `:76`, `:894`, `:1071`, `:1111`, `:1117`).

---

#### 2. Error-to-wire mapping

The rule is **message-kind determines frame kind**:

| Inbound | Rejection frame | Rationale / site |
|---|---|---|
| `EVENT` | `["OK", <event_id>, false, "<prefix>: <detail>"]` | NIP-20 acknowledgement — `event.rs:669`, `:639`, `:649`, `:660`, `:678`, `:733` |
| `REQ` | `["CLOSED", <sub_id>, "<prefix>: <detail>"]` | `req.rs:56`, `:67`, `:80`, `:101`, `:167`, `:185`, `:192`, `:199`, `:214` |
| `COUNT` | `["CLOSED", <sub_id>, …]` | `count.rs:44`, `:57`, `:64`, `:71`, `:86`, `:179` |
| parse failure (no sub_id yet) | `["NOTICE", "invalid message: …"]` | `connection.rs:493` |
| pre-dispatch throttle | `CLOSED` if a sub_id exists, else `NOTICE` | `request_rejection_message`, `connection.rs:587-592` |

Reason strings use the NIP-01 machine-readable prefix set, consistently:

| Prefix | Meaning as used here | Examples |
|---|---|---|
| `auth-required:` | not authenticated, or an auth-state error | `event.rs:649`, `req.rs:78`/`:82`, `count.rs:47`, `auth.rs:54`/`:63`/`:210`/`:291` |
| `restricted:` | authenticated but not permitted | `event.rs:686`, `:681`, `:1027`; `req.rs:170`, `:187`, `:194`, `:201`; `auth.rs:234` |
| `invalid:` | malformed / semantically rejected | `event.rs:665`, `:652`, `:757`, `:956`, `:1073`, `:1096` |
| `blocked:` | moderation ban | `auth.rs:160` |
| `rate-limited:` | throttled | `connection.rs:517`, `:546`, `:567`, `:666`, `:674`; `event.rs:1063` |
| `error:` | server-side fault or a protocol-level cap | `event.rs:788`, `:1012`; `req.rs:69`, `:103`, `:216`; `count.rs:86`, `:179` |

Two deliberate escalations beyond the frame:
- **Ban** — frame queued on `ctrl_tx`, then `cancel()` so the socket closes immediately (`auth.rs:173-182`). The "queue on ctrl, then cancel" idiom is named as a reusable convention at `connection.rs:328-336` and pinned by `connection.rs:856-882`.
- **Oversized frame** — `NOTICE` then `break` out of `recv_loop` (`connection.rs:428-433`, `:447-452`).

##### Sanitisation convention (not uniformly applied)

`handle_event` sanitises `IngestError::Internal` to `error: internal server error` with an explicit comment (`event.rs:749-754`). `handle_count` does the opposite: four sites forward the raw error with `format!("error: {e}")` (`count.rs:179`, `:209`, `:249`, `:278`). The `req.rs` historical path forwards nothing (it emits a bare `EOSE`, `:327`). Three different postures for the same class of failure.

##### Metrics-on-rejection convention

`event.rs` funnels every rejection through `reject(reason)` (`event.rs:30-32`) → `reject_with_transport("ws", reason)` (`ingest.rs:156`) with a **bounded** reason label — the four values used are `"auth"`, `"invalid"`, `"scope"`, `"error"` (`event.rs:645`, `:638`, `:648`, `:659`, `:677`, `:732`, `:969`, `:1022`). `req.rs`, `count.rs`, and `close.rs` do **not** emit a rejection counter at all, so REQ/COUNT denials are invisible in metrics.

---

#### 3. Locking conventions

| Convention | Evidence |
|---|---|
| `auth_state` is a `RwLock` because it is read-heavy after auth; `subscriptions` is a `Mutex` because it is write-heavy during REQ/CLOSE. Stated in the struct doc. | `connection.rs:50-52` |
| Guards are scoped so they drop before I/O `await`s. | `event.rs:634-654`, `req.rs:50-87`, `count.rs:37-51` |
| Only one nested acquisition exists, always in the same order: `auth_state` (read) → `subscriptions` (lock). No reverse ordering exists, so no deadlock. | `req.rs:51` → `:65` |
| DashMap guards are explicitly `drop`ped before a `remove` on the same map, to avoid self-deadlock. | `subscription.rs:408-410`, `:430-432`, `:447-449`, `:456-457`, `:465-466` |
| `authenticated_pubkey` uses `std::sync::RwLock` (not tokio) because reads are non-async and lock-poisoning is handled with `.ok()?`. | `state.rs:56`, `:246-256`, `:286-290` |

---

#### 4. Concurrency conventions

| Convention | Evidence |
|---|---|
| `try_send` / `try_acquire_owned` everywhere on the hot path — never `send().await` or `acquire().await`, so the recv loop cannot be blocked by a slow peer or a saturated handler pool. | `connection.rs:89`, `:149`, `:513`, `:541`, `:563`; `state.rs:453` |
| One documented exception: the audit enqueue uses `send().await` **on purpose**, with a written rationale. | `event.rs:574` (rationale `:551-557`) |
| Semaphore permits are `Owned` and dropped explicitly at the end of the spawned body. | `connection.rs:533`, `:555`, `:576` |
| CPU-bound signature verification always goes to `spawn_blocking`. | `event.rs:772`, `:927`; `ingest.rs:1488` |
| Long delivery loops yield cooperatively every 100 items. | `req.rs:401-404` |
| Ordered concurrency uses `buffered`, never `buffer_unordered`, when downstream ordering is semantically load-bearing — with the reason spelled out. | `req.rs:318` (doc `:299-303`, and the constant's own doc `:28-34`) |
| Concurrency bounds are compile-time asserted where a wrong value would be silently harmful. | `req.rs:37-41` (`const _: () = assert!(…)`) |
| `biased;` in `select!` where starvation would break a safety property. | `connection.rs:326-327` |

---

#### 5. Test conventions

Counts (all in-file `#[cfg(test)] mod tests`):

| File | Tests | `#[ignore]` | Module starts |
|---|---|---|---|
| `connection.rs` | 5 | 0 | `:689` |
| `subscription.rs` | 29 | 0 | `:574` |
| `handlers/event.rs` | 24 | 0 | `:1135` |
| `handlers/req.rs` | 45 | 0 | `:1233` |
| `handlers/auth.rs` | 3 | 0 | `:298` |
| `handlers/count.rs` | **0** | — | — |
| `handlers/close.rs` | **0** | — | — |
| `handlers/mod.rs` | **0** | — | — |
| **total** | **106** | **0** | |

Conventions observed:

1. **Infra-optional tests skip, not `#[ignore]`.** Instead of the `#[ignore = "requires Postgres"]` style used elsewhere in the crate (e.g. `api/operator.rs:706`), this group probes availability at runtime and returns early with an `eprintln!`: `event.rs:1676-1679` (Redis), `:1636-1639` and `:1735-1738` (Postgres+Redis via `audit_state()`), `:1642-1649` (`redis_url_if_available`). Net effect: `just test-unit` never fails on a missing dependency, but it also never reports that a test was skipped as a test outcome.
2. **Lazy-pool test state.** `fanout_access::test_state()` (`event.rs:2041-2043`) builds a full `AppState` with `PgPool::connect_lazy` and an intentionally dead Redis (`redis://127.0.0.1:1`, `event.rs:1997-2002`), so pure fan-out logic is testable with no infrastructure. Cache pre-seeding substitutes for DB reads (`event.rs:2123-2126`, `:2143-2152`).
3. **Fail-closed proved by omission.** `threaded_visibility_open_passes_through` (`event.rs:2298-2321`) relies on the lazy pool erroring a fresh lookup: pass-through therefore *proves* the threaded value was consulted. The reasoning is written into the test doc comment (`:2291-2294`).
4. **Test-only single-tenant wrappers** keep pre-multi-tenant tests readable: `register`, `remove_channel_subscriptions`, `channel_subscriber_conns`, `fan_out`, all `#[cfg(test)]` and delegating to the `*_scoped` form with `test_community()` = nil UUID (`subscription.rs:154-160`, `:228-233`, `:257-260`, `:333-336`, `:568-571`).
5. **Security regressions are named as such** and carry the invariant they pin in the doc comment: `test_global_sub_does_not_receive_channel_events` (`subscription.rs:996-1032`), `channel_less_event_must_drop_recipient_in_different_community` (`event.rs:2458-2481`), `local_echo_suppression_is_scoped_to_its_community` (`event.rs:1576-1617`).
6. **Red-team module convention.** `event.rs:2346-2483` is a `mod redteam` (declaration at `:2395`) whose ~50-line header cites the TLA+ spec (`docs/spec/MultiTenantRelay.tla`), the invariant, the mutation class, the exact code sites, the required structural fix, and the ownership routing. It also documents a self-deleting pattern ("MUST be deleted in the same change that fixes the leak") — though the companion "documents the current broken behavior" test it refers to at `:2383-2385` **is not present**, so the header is now partly stale (see the debt aspect).
7. **Byte-compatibility pinning** for anything performance-refactored: `fanout_event_frame_matches_legacy_format_byte_for_byte` (`event.rs:1177-1189`) and the `Arc`-must-not-escape-a-cycle assertion (`:1168-1188`).
8. **Truth-table tests** for boolean gates, one test per row: `resolve_request_local_access` gets all four rows (`req.rs:1299-1360`); `d_tag` pushdown gets five (`req.rs:1594-1648`); `p_gated_filters_authorized` for 44200 gets four numbered cases in one test with case comments (`req.rs:1490-1546`).
9. **Assertion messages carry the invariant**, not the mechanics: `"kinds:[] sub must NOT be in the wildcard index"` (`subscription.rs:962`), `"Inv_NonInterference: a connection bound to community A must not receive a community-B event"` (`event.rs:2475-2478`).
10. **Mock sink over a real socket** for send-loop tests: `MockSink` implements `Sink<WsMessage>` with a scripted `fail_after_flushes` so `send_loop_inner` terminates deterministically (`connection.rs:692-757`). This is why `send_loop` is a thin wrapper over the generic `send_loop_inner` (`connection.rs:296-306`).

Notable coverage gaps in convention terms: `count.rs` (281 LOC) and `close.rs` (35 LOC) have **no** in-file tests at all, and no test in this group drives `handle_req`, `handle_count`, or `handle_close` end-to-end — only their extracted helpers. `handle_agent_observer_event` is the single message handler with an end-to-end unit test (`event.rs:1318-1404`).

---

#### 6. Documentation conventions

| Convention | Evidence |
|---|---|
| Module-level `//!` stating the pipeline in one line | `connection.rs:1`, `subscription.rs:1`, `event.rs:1`, `req.rs:1`, `count.rs:1`, `auth.rs:1-13` |
| Every `pub` item has a doc comment (crate enforces `missing_docs`; the one opt-out is `#[allow(dead_code, missing_docs)] pub mod push_lease` at `handlers/mod.rs:23`) | throughout |
| Invariants are written as prose next to the code that enforces them, and cross-reference the plan/spec section | `event.rs:196-217`, `:100-114`; `req.rs:423-447`; `auth.rs:92-112` |
| Rejected alternatives are recorded inline rather than dropped | `event.rs:574-580` (why `send().await`), `req.rs:28-34` (why per-filter queries), `state.rs:1107-1116` (why only `private` is cached) |
| Numbered fences referencing an external design doc | `event.rs:184-190` ("Fence 3 (§4.8 phase-2)", "Fence 1"), `auth.rs:95-97` ("COMMUNITY_MODERATION_PLAN.md §0 decision 4", "MOD-7/M20") |
| Stale-comment risk is high because comments name line numbers and revisions | `event.rs:2368` ("this file, line 62" — `filter_fanout_by_access` is now at `:115`), `event.rs:2361` (cites `state.rs:30-44` for `ConnEntry`, which is now `:41-58`) |

---

#### 7. Metrics naming convention

`buzz_<subsystem>_<noun>_<unit>` with `_total` for counters, `_seconds` for duration histograms, bare noun for gauges:

- counters: `buzz_ws_connections_total`, `buzz_ws_auth_timeouts_total`, `buzz_ws_backpressure_disconnects_total`, `buzz_admission_rejections_total`, `buzz_auth_attempts_total`, `buzz_auth_failures_total`, `buzz_events_received_total`, `buzz_community_events_received_total`, `buzz_multinode_fanout_total`, `buzz_post_commit_dispatch_scheduled_total`, `buzz_post_commit_dispatch_errors_total`, `buzz_audit_send_errors_total`, `buzz_req_global_access_resolution_skips_total`, `buzz_count_fallback_rejections_total`
- gauges: `buzz_ws_connections_active`, `buzz_subscriptions_active`
- histograms: `buzz_event_processing_seconds`, `buzz_fanout_recipients`, `buzz_ws_send_batch_size`

Label cardinality is explicitly bounded: `bounded_kind_label` (`event.rs:35-53`) collapses unknown kinds to `"other"`, and kind × community is deliberately never crossed (rationale `event.rs:620-627`). Rejection reasons are `&'static str` by type (`event.rs:30`), so the label set cannot grow at runtime.

---

#### 8. Code-style facts (quality-gate relevant)

| Check | Result |
|---|---|
| `unsafe` blocks in the 8 files | **0** |
| `unwrap()` / `expect()` outside `#[cfg(test)]` | **1** — `event.rs:88` `.expect("fan-out frame cache covers every recipient subscription id")` |
| `TODO` / `FIXME` / `XXX` / `HACK` markers | **0** |
| `#[ignore]`d tests | **0** |
| `#[allow(dead_code)]` | 1, on the out-of-group `push_lease` module (`handlers/mod.rs:23`) |
| Files over 1000 lines | 3 of 8 — `event.rs` 2461, `req.rs` 1946, `subscription.rs` 1562 |
| Production-code share | `connection.rs` 688/893 (77%), `subscription.rs` 573/1562 (37%), `event.rs` 1134/2461 (46%), `req.rs` 1232/1946 (63%), `auth.rs` 297/350 (85%) |


## Module: buzz-relay — event ingest & side effects (`crates/buzz-relay/src/handlers`)
### Aspect: Conventions

---

### 1. Handler shape

Three distinct shapes coexist, distinguished by return type and error style.

**A. The ingest pipeline** — one linear function, no sub-handlers.

```rust
async fn ingest_event_inner(
    state: &Arc<AppState>,
    tracer: &Arc<dyn buzz_conformance::Tracer>,
    tenant: &TenantContext,
    event: Event,
    auth: IngestAuth,
) -> Result<IngestResult, IngestError>          // ingest.rs:1453
```

Argument order is stable across the group: `tenant` first (or after `state`), then
`state`, then `event`, then auth. Two orderings are actually in use —
`(state, tenant, …)` in `ingest.rs` and `(tenant, state, …)` in `side_effects.rs` and
`command_executor.rs`. No file mixes them internally.

**B. Side-effect handlers** — `anyhow::Result<()>`, private, one per kind.

```rust
async fn handle_put_user(
    tenant: &TenantContext,
    event: &Event,
    state: &Arc<AppState>,
) -> anyhow::Result<()>                          // side_effects.rs:1203
```

All 13 follow this exactly: `handle_kind0_profile` `:1113`, `handle_agent_profile` `:1078`,
`handle_put_user` `:1203`, `handle_remove_user` `:1265`, `handle_edit_metadata` `:1335`,
`handle_delete_event_side_effect` `:1560`, `handle_create_group` `:1660`,
`handle_delete_group` `:1783`, `handle_join_request` `:1835`, `handle_leave_request` `:1913`,
`handle_a_tag_deletion` `:1979`, `handle_standard_deletion_event` `:2108`,
`handle_git_repo_announcement` `:2412`.

**C. Command handlers** — `Result<IngestResult, IngestError>`, taking `&IngestAuth`.

```rust
async fn handle_dm_open(
    tenant: &TenantContext,
    state: &Arc<AppState>,
    event: &Event,
    auth: &IngestAuth,
) -> Result<IngestResult, IngestError>           // command_executor.rs:310
```

All 7 identical (`:310`, `:443`, `:580`, `:653`, `:809`, `:986`, `:1098`). Each follows the
same 6-step comment skeleton — `// 1. Extract`, `// 2. Validate`, `// Persist the command
event`, `// 4. Execute`, `// Commit`, `// 5./6. Side effects` + `// Return response` —
which makes the group readable as a template. `handle_dm_open` `:310-441` is the reference
implementation.

---

### 2. Validation-function conventions

| Convention | Detail |
|---|---|
| Naming | `validate_*` for pre-storage gates. `validate_edit_ownership` `ingest.rs:763`, `validate_forum_vote_target` `:844`, `validate_diff_event` `:896`, `validate_engram_envelope` `:965`, `validate_persona_envelope` `:1027`, `validate_engram_nip44_content` `:1084`, `validate_agent_turn_metric_envelope` `:1151`, `validate_not_before` `:1223`, `validate_event_reminder` `:1252`, `validate_admin_event` `side_effects.rs:259`, `validate_standard_deletion_event` `:179`, `validate_imeta_tags` `imeta.rs:11`, `validate_repo_id` `side_effects.rs:2391`. |
| Error type | Pure/synchronous validators return `Result<(), String>`; the two `&'static str` returners (`validate_not_before`, `validate_event_reminder`) are the only exceptions, and their strings are closed-set wire values the spec pins for client matching (`ingest.rs:1285-1287`). Async DB-touching validators in `side_effects.rs` return `anyhow::Result<()>`. |
| Error prefix | Validators return **bare** messages; the ingest call site adds the prefix: `.map_err(\|e\| IngestError::Rejected(format!("invalid: {e}")))`. 12 sites use this exact line (`ingest.rs:1934`, `:1918`, `:1924`, `:1962`, `:1968`, `:1973`, `:1978`, `:1983`, `:2020`, `:2025`, `:2214`, `:2217`). ⚠ This produces `invalid: restricted: not a channel member` for edit-ownership failures (`ingest.rs:838`), i.e. a double prefix. |
| Predicate naming | `is_*` for classification: `is_global_only_kind` `ingest.rs:379`, `requires_h_channel_scope` `:455`, `is_admin_kind` `side_effects.rs:26`, `is_side_effect_kind` `:35`, `is_local_media_url` `imeta.rs:373`, `is_well_formed_mime` `:340`, `has_e_tag` `side_effects.rs:2300`, `actor_is_channel_owner_or_admin` `:2357`, `author_delete_can_use_self_delete_path` `:2353`. |
| Position | Every validator runs **pre-storage**. The only post-storage logic is `handle_side_effects`, whose failures are non-fatal by design (`ingest.rs:2460-2467`). |

---

### 3. Error-string wire format

A three-token prefix vocabulary, matched by clients:

| Prefix | Meaning | Typical mapping |
|---|---|---|
| `invalid: ` | client-side protocol/data error | `Rejected` → HTTP 400 |
| `restricted: ` | authorization refusal | either `Rejected` or `AuthFailed`; HTTP 400 or 403 |
| `blocked: ` | community ban | `AuthFailed` → 403 |
| `error: ` | server-side failure | mostly `Internal` → 500 |
| `duplicate: ` | idempotent no-op | `Ok` |
| `info: ` | successful non-storage action | `Ok` |
| `response:` | JSON payload follows (command kinds) | `Ok` |
| `forbidden: ` | **command-executor only** | `Rejected` → 400 |

⚠ Three inconsistencies:
1. `forbidden: ` appears only in `command_executor.rs` (`:509`, `:625`, `:711`, `:845`,
   `:975`, `:982`) and always as `Rejected`, so an authorization failure on a command kind
   returns HTTP **400**, while the same class of failure on any other kind returns **403**.
2. `restricted: ` maps to `Rejected` in some places (`ingest.rs:1482`, `:1507`, `:521`) and
   `AuthFailed` in others (`:1513`, `:1521`, `:1526`, `:1726`, `:2012`).
3. `error: database error: {e}` from `check_channel_membership` (`ingest.rs:501`) is
   surfaced as `Rejected` (`:1802`), giving a 400 for a server fault.

The prefix set is not centralised anywhere. `crate::conformance::sanitized_reason_for`
(`ingest.rs:1411`) is the only place that classifies `IngestError` variants into a closed
alphabet, and it is for the trace, not the wire.

---

### 4. Tag access convention

Every tag read in this group goes through `tag.kind().to_string() == "name"` and
`tag.content()`, or through `tag.as_slice()` for positional access. Two families:

| Helper | Semantics | Copies |
|---|---|---|
| first-match extractor | returns the first matching tag's content | `extract_channel_id` `ingest.rs:308`; `extract_h_tag_channel` `side_effects.rs:2237`; `extract_p_tag` `:2251`; `extract_tag_value` `:2325`; `extract_h_tag` `command_executor.rs:250`; `extract_d_tag` `:261`; `extract_e_tag` `:272`; `extract_tag` `:283` |
| all-match collector | `extract_target_event_ids` `side_effects.rs:2304`; `extract_p_tags` `command_executor.rs:235`; `count_e_tags` `ingest.rs:719` |

⚠ **`e`-tag selection direction is inconsistent and load-bearing.** Reactions take the
**last** `e` tag (`.rev()` — `ingest.rs:334`, `:2251`, `side_effects.rs:2192`), per NIP-25.
Edits (`ingest.rs:766`), votes (`:847`), deletion channel derivation (`:1670`), and 9005
(`side_effects.rs:531`) take the **first**. Nothing names or documents the rule; it must be
read off each call site.

⚠ **Duplicated helpers.** `effective_message_author` exists twice with identical bodies —
`ingest.rs:729-761` (`pub(crate)`) and `side_effects.rs:2271-2298` (private). The
`side_effects.rs` copy uses `extract_tag_value(event, "actor")` where the `ingest.rs` copy
inlines the same loop. `side_effects.rs:2195` then reaches back for
`super::ingest::effective_message_author`, so both copies are live in the same file's call
graph. Similarly `extract_channel_id` (`ingest.rs:308`) and `extract_h_tag_channel`
(`side_effects.rs:2237`) are byte-equivalent, and `command_executor.rs:250` has a third
variant returning `Option<String>` instead of `Option<Uuid>`.

---

### 5. How a new kind is added (the actual sequence)

Derived from the code, not from docs — no doc describes this.

1. Add the constant to `crates/buzz-core/src/kind.rs` and to `ALL_KINDS`
   (`kind.rs:566-693`). Add a compile-time range assertion if the kind is
   replaceable/parameterized (`kind.rs:783-820`).
2. Add a match arm to `required_scope_for_kind` (`ingest.rs:198-306`). **Without this the
   kind is rejected with `restricted: unknown event kind`.** This is the real gate.
3. Add it to exactly one of `is_global_only_kind` (`ingest.rs:379-453`) or
   `requires_h_channel_scope` (`:455-491`), or neither if the channel is derived some other
   way. The disjointness test (`:2753-2762`) will catch getting both.
4. If the kind needs pre-storage validation, write a `validate_*` returning
   `Result<(), String>` and call it in the `ingest_event_inner` gauntlet
   (`ingest.rs:1986-2052` is where the per-kind validators cluster).
5. If it needs post-storage effects, add it to `is_side_effect_kind`
   (`side_effects.rs:35-37`) **and** add a `handle_side_effects` arm
   (`:143-176`). Both, or the kind is silently ignored.
6. If it is a transactional command, add it to `buzz_core::kind::is_command_kind`
   (`kind.rs:743-755`), then a `handle_command` arm (`command_executor.rs:66-77`) plus a
   handler following shape C.
7. If it is relay-signed only, add it to `is_relay_only_kind` (`kind.rs:758-769`) so the
   reject message is `restricted: relay-only kind` rather than
   `restricted: unknown event kind`. ⚠ Eight relay-minted kinds skip this step today
   (8000, 8001, 8002, 8003, 13535, 39000, 39001, 39002, 40099) — see features.md §3.
8. Add unit tests to the `ingest.rs` test module asserting scope, global-only, and
   `requires_h` classification. The existing suite has one test per property per kind
   family (e.g. `:2683-2718` for NIP-51 lists, `:2909-2927` for teams/managed agents), and
   `per_kind_scope_allowlist_covers_all_migrated_kinds` (`:2822-2879`) is the running
   checklist — 44 kinds are listed there, out of 81 accepted, so it is not exhaustive.

There is **no** single registry or trait. Adding a kind touches 3–6 disjoint `match`
statements across 2 crates, none of which are exhaustive over `ALL_KINDS`. Nothing fails to
compile if a step is skipped.

---

### 6. Test conventions

| Convention | Detail |
|---|---|
| Location | `#[cfg(test)] mod tests` at the bottom of the file: `ingest.rs:2532`, `side_effects.rs:3266`, `imeta.rs:419`. `command_executor.rs` has none. |
| Style | Pure-function unit tests only. **Zero** `#[tokio::test]`, so no handler with a DB dependency is tested in-file. All 111 tests are synchronous. |
| Builders | `ingest.rs` has a small builder set: `make_dummy_event()` `:3045`, `make_event_with_tags(kind, content, &[&[&str]])` `:3053`, then kind-specific wrappers `make_engram` `:3083`, `make_reminder` `:3283`, `make_persona` `:3421`, `make_agent_turn_metric` `:3541`, plus `fake_nip44_v2()` `:3090` producing a shape-valid 99-byte NIP-44 v2 payload. |
| Assertion style | Property assertions loop over kind arrays with a message naming the kind: `assert!(is_global_only_kind(kind), "kind {kind} must be global-only")` (`ingest.rs:2597`, `:2707`, `:2865`, …). Error assertions match on substrings, not equality: `assert!(err.contains("`p` tag"), "got: {err}")` (`ingest.rs:3137`). |
| Exhaustive properties | One brute-force test over the whole kind space: `global_only_and_channel_scoped_are_disjoint` iterates `0..=65535` (`ingest.rs:2779-2788`). |
| Regression documentation | Regressions carry a doc comment explaining the failure they prevent, e.g. the uppercase-`p` invisibility bug (`ingest.rs:2612-2616`) and the non-base64 replacement bug (`:3169-3174`). |
| Metrics tests | `reject_with_transport_labels_http_and_ws_as_separate_series` (`ingest.rs:3752-3793`) uses `metrics_util::debugging::DebuggingRecorder` + `with_local_recorder` to assert label cardinality. The only metrics test in the group. |
| Conformance tests | `feedback_success_action_satisfies_ingest_emit_guard` (`ingest.rs:2557-2591`) arms a real `EmitGuard` against a `VecTracer` (`:2545-2555`) and asserts exactly one `WriteInsertGlobal`. The pattern exists for one kind only. |
| `side_effects.rs` tests | All 5 cover pure helpers: `delete_tombstone_content` (a `#[cfg(test)]`-only function at `:2363-2391` that duplicates the production tombstone builder at `:1650-1656`), `author_delete_can_use_self_delete_path`, `actor_is_channel_owner_or_admin`. Nothing touches the 418-line `validate_admin_event`. |
| Integration coverage | Behaviour of this group is covered out-of-file in `crates/buzz-test-client/tests/` — e.g. `e2e_human_edit_agent_content.rs:5-6` names `validate_standard_deletion_event` and the `validate_admin_event` 9005 branch as its subjects. Per AGENTS.md these need Postgres + Redis (`just test`). |

---

### 7. Logging conventions

| Convention | Detail |
|---|---|
| Levels | `debug!` for pipeline entry (`ingest.rs:1458`); `info!` for accepted writes and completed side effects (`ingest.rs:2385`, `:2499`); `warn!` for every swallowed side-effect failure (32 sites in `side_effects.rs`); `error!` for genuine bugs (`ingest.rs:1498` spawn panic, 8 sites in `command_executor.rs` for spawned-task failures). |
| Structured fields | `event_id = %hex`, `kind = u32`, `channel = %uuid`, `target = %hex`, `pubkey = %hex`, `error = %e`. Consistent throughout. |
| Sensitive values | Pubkeys are logged as hex — acceptable (they are public). Event content is never logged. Reject reasons that may embed event-controlled data are truncated **at the transport**, not here (`api/bridge.rs:842-851`). |
| Success/failure symmetry | `ingest.rs` logs one `info!` per accepted event (`:2359` reaction, `:2499` generic) but nothing on rejection — rejection telemetry is the `buzz_events_rejected_total` counter, emitted by the transport (`reject_with_transport` `:156`), not by ingest. |

---

### 8. Comment conventions

This group is unusually heavily commented, and the comments carry design decisions rather
than restating code. Recurring patterns:

- **Decision references.** `(OQ1 decision …)` `ingest.rs:1794`; `(E1 within-request
  threading; correctness ruling §4.8)` `:485`; `(§4.8 phase-2 addendum)` `:1742`;
  `(COMMUNITY_MODERATION_PLAN.md §0 decision 4)` `:1589`; `(C5)` double-count analysis
  `side_effects.rs:1670-1679`; `(F9)` `buzz-db/src/thread.rs:114`;
  `(spec line 794)` `ingest.rs:1776`.
- **Load-bearing-ordering markers.** "Ordering is load-bearing" `ingest.rs:2320`;
  "Redis before local fan-out so subscribers on other relay pods receive it too"
  `side_effects.rs:783`; "Listed after the workflow branch so workflow's bespoke deletion
  … takes precedence" `:2051-2056`.
- **Known-limitation blocks.** `ingest.rs:369-377` (stray `h` on the read path);
  `side_effects.rs:952-960` (channel-scoped discovery vs live global subs);
  `:1516-1524` (four sub-second archive toggles);
  `command_executor.rs:92-98` (non-atomic command mutations).
- **Negative rationale** — why something is *not* done: "we intentionally do NOT check
  is_agent_owner for non-members" `side_effects.rs:365-367`; "Not reachable in practice …
  so we don't engineer around it" `:1522-1524`; "diverges from kind:9001 intentionally"
  `:598-600`, `:632-634`.
- **Unreachable-arm annotations.** `RemoveResult::RoleMismatch` is documented as
  "unreachable but exhaustiveness requires it" (`ingest.rs:1901-1902`).

---

### 9. Metrics conventions

Label cardinality is reasoned about explicitly. Fleet-wide counters carry `kind` but
**no** `community`, because `bounded_kind_label` passes through all 10 000 ephemeral kind
values and crossing kind × community "would produce up to millions of series"
(`handlers/event.rs:629-632`). Per-community counters carry `community` but no `kind`.
`author_type` is accepted as a label only because it is 2-valued and "merely doubles the
kind series" (`ingest.rs:1391-1394`). `reject_with_transport`'s `reason` is documented as
"one of a small closed set … bounded, no cardinality risk" (`ingest.rs:150-155`).

---

### 10. AGENTS.md compliance

| Rule | Status |
|---|---|
| No `unsafe` | ✅ 0 occurrences in 8 911 lines |
| No new `unwrap()`/`expect()` in production paths | ❌ 4 `expect()` (`ingest.rs:2024`, `:2000`, `:2338`, `:2344`) + 2 `unwrap()` (`side_effects.rs:311`, `:314`) |
| New public API must have doc comments | ✅ every `pub` item in all four files is documented; `IngestAuth`'s fields are individually documented (`ingest.rs:64-85`) |
| Channels use `h` tags, not `e` tags | ✅ `extract_channel_id` reads only `h` (`ingest.rs:308-319`) |
| Event kinds defined in `buzz-core/src/kind.rs` | ⚠ mostly — `KIND_PUSH_LEASE` is defined in **both** `buzz-core/src/kind.rs:109` and `handlers/push_lease.rs:19`, and `ingest.rs` imports the `push_lease` copy (`ingest.rs:216`, `:451`, `:2156`) rather than the `buzz-core` one. Two sources of truth for one integer. |
| Prefer Nostr events over new HTTP endpoints | ✅ this group adds no HTTP surface |
| Thread counters updated on every reply-insert path | ✅ verified — see data-model.md §7 |
| 1 000-line file ceiling | ❌ not applicable to Rust (guard is JS/mobile only), but `ingest.rs` (3 686) and `side_effects.rs` (3 347) are 3.3–3.7× over the ceiling the repo applies elsewhere — see debt.md D-01 |


## Module: buzz-relay — HTTP API surface (`crates/buzz-relay/src/api`)
### Aspect: Conventions

---

#### 1. Handler signature conventions

Three distinct return-type dialects coexist in one module tree:

| Dialect | Return type | Used by | `file:line` |
|---|---|---|---|
| **Bridge/invites/operator** | `Result<Json<Value>, (StatusCode, Json<Value>)>` | 12 handlers | `bridge.rs:616`, `:883`, `:1317`, `:2077`, `:2103`, `:2121`; `invites.rs:233`, `:294`, `:165`; `operator.rs:152`, `:206`, `:268`, `:305`, `:357`, `:471` |
| **Media** | `Result<Json<BlobDescriptor>, MediaError>` / `Result<Response, MediaError>` — a domain error type with its own `IntoResponse` | 3 handlers | `media.rs:305-310`, `:604-608`, `:798-802` |
| **Admin** | `Result<Json<T>, ApiError>` with a typed error struct | 5 handlers | `admin/mod.rs:93-97`, `:125-129`, `:151-154`, `:177-181`, `:191-195` |

Two more one-offs: `workflow_webhook` returns `Result<(StatusCode, Json<Value>), (StatusCode, Json<Value>)>`
so it can emit 202 (`bridge.rs:1766`); `demo_echo` and `nostr_nip05` return bare `Response`
(`mesh_demo.rs:61`, `nip05.rs:29`), i.e. they are infallible by construction.

##### Extractor ordering

Consistent and load-bearing: `State` first, then `Path`, then `RawQuery`, then `Query`, then
`HeaderMap`, then the body (`Bytes` / `Body` / `Json`).

- `bridge.rs:2096-2099` — `State`, `HeaderMap`, `RawQuery`, `Query`: `RawQuery` **and** `Query` are
  both taken because NIP-98 signs the verbatim query string while the handler wants typed params.
  Same pattern at `operator.rs:305-309`, `:471-475`.
- `media.rs:306-309` — `State`, `AuthenticatedUpload` (a `FromRequestParts` extractor), `HeaderMap`,
  `Body`. The auth extractor runs before the body extractor by axum's rules, which is the whole
  point (`media.rs:29-38`).
- `mesh_demo.rs:59-62` — `State`, `Json<DemoEchoRequest>`. This is the one place where extractor
  ordering **defeats** the handler's intent: the `Json` rejection fires before the feature-flag 404.

##### Body handling

Handlers that need the raw bytes for NIP-98 payload verification take `axum::body::Bytes` and
`serde_json::from_slice` manually (`bridge.rs:618`, `:885`, `:1319`; `invites.rs:167`, `:236`;
`operator.rs:155`). Only `mesh_demo.rs:61` uses the `Json<T>` extractor — and it is the only handler
with no NIP-98 requirement, so nothing is lost there except the 404-indistinguishability property.

##### Tenant-binding preamble

Every tenant-scoped handler repeats the same 8-line block verbatim: read `Host`, `to_str().ok()`,
`unwrap_or("")`, `bind_community`, `map_err` to a fixed 404. Copied at `bridge.rs:621-633`,
`:888-901`, `:1321-1334`, `:1777-1786`, `:2013-2025`; `invites.rs:198-207`; `media.rs:154-166`,
`:477-487`; `nip05.rs:31-40`. Only `media.rs` factors it (`bind_media_read_tenant`, `media.rs:478`).

##### Post-auth helper split

`/events`, `/query`, `/count` each split into a thin routed wrapper plus an `_authed` helper, so the
wrapper can own exactly one terminal attribution log for every outcome:

| Wrapper | Helper | `file:line` |
|---|---|---|
| `submit_event` | `submit_event_authed` | `bridge.rs:613` / `:750` |
| `query_events` | `query_events_authed` | `bridge.rs:880` / `:947` |
| `count_events` | `count_events_authed` | `bridge.rs:1316` / `:1378` |

`submit_event` additionally routes outcomes through a `SubmitOutcome` enum carrying both log fields
and the HTTP response (`bridge.rs:706-747`), with `into_response()` collapsing it (`:722-729`).

#### 2. Error-to-status mapping

##### Envelope helpers (`api/mod.rs`)

```rust
api_error(status, msg)  -> (status, Json({"error": msg}))     // mod.rs:19-21
internal_error(msg)     -> tracing::error!; api_error(500, "internal server error")  // mod.rs:23-26
not_found(msg)          -> api_error(404, msg)                // mod.rs:28-31
```

`internal_error` is the only helper that deliberately withholds detail from the client while keeping
it in logs. It is used 30+ times across the module for every DB/serialize failure.

##### Status conventions actually observed

| Condition | Status | Body |
|---|---|---|
| Unmapped `Host` | 404 | fixed `"relay: no community is configured for this host"` (never echoes the host) |
| Missing/invalid NIP-98, replay detected, replay guard down | 401 | `"missing Nostr auth"` / `"NIP-98: {e}"` / `"NIP-98: replay detected"` / `"NIP-98: replay check unavailable"` |
| Read-gate violation (p-gate / engram / author-only) | **403** | `"restricted: …"` — note the WS sibling sends a `CLOSED` frame with an analogous string (`handlers/req.rs:186-204`) |
| Not a relay member | 403 | `{"error":"relay_membership_required","message":…}` — the **only** two-key error body in the non-admin surface (`api/mod.rs:134-139`) |
| Malformed filter/cursor/JSON | 400 | reflects the `serde_json` message |
| Rate limited | 429 | `"rate-limited: quota exceeded; retry in {n}s"` |
| Rate limiter unavailable | 503 | `"rate-limited: shared admission unavailable"` |
| Any DB/serialize error | 500 | fixed `"internal server error"` |

##### Media (`buzz-media/src/error.rs:107-162`)

All 15 authentication-ish variants (`MissingAuth`, `InvalidBase64`, `HashMismatch`, `ServerMismatch`,
`MissingTag`, `TokenExpired`, …) collapse to one 401 `"authentication failed"` explicitly "to prevent
oracle enumeration" (`error.rs:118-121`). `InsufficientScope` and `RelayMembershipRequired` are 403;
size 413; content-type 415; validation 422; rate/concurrency 429; IO/storage 500 with a generic body.

##### Admin (`admin/error.rs`)

Four constructors only — `bad_request(code, message)`, `forbidden()`, `not_found()`, `internal()` —
with `&'static str` code/message, so **no dynamic text can leak** through this surface by
construction. `From<DbError>` maps unconditionally to `internal()` (`admin/error.rs:79-83`).

##### Operator error mapping by string prefix

`operator.rs:180-199` matches on the `String` error returned by `community_provisioning`:
`"actor not authorized"`→403, `"community already exists"` / `"limit_reached:"`→409,
`"failed to create community:"` / `"community provisioned but owner bootstrap failed:"`→500 (generic
body + `tracing::error!`), fallthrough→400 with the message passed through. This is stringly-typed
control flow — a wording change in the provisioning module silently reclassifies the HTTP status.

#### 3. JSON shape conventions

| Convention | Where | Deviations |
|---|---|---|
| snake_case keys | bridge, invites, operator, media, mesh demo | admin uses `camelCase` (`admin/mod.rs:64`, `:140`, `:22`) |
| Reads return a bare JSON **array** | `/query`, `/moderation/*` | `/count` returns `{count}`; NIP-05 returns an object |
| Writes return `{event_id, accepted, message}` | `POST /events` (`bridge.rs:834-838`) | invites return `{code,…}` / `{status,…}`; operator returns `{community_id,…}` |
| Hex-encode all byte fields | `report_json`/`action_json`/`ban_json` (`bridge.rs:2132-2184`) | — |
| `#[serde(skip_serializing_if = "Option::is_none")]` for optional response fields | `operator.rs:51`, `handlers/community_provisioning.rs:67` | most ad-hoc `json!` bodies emit `null` instead |
| Ad-hoc `serde_json::json!` for response bodies | dominant style — 20+ sites | only 4 typed `Serialize` response structs exist (`TransferCommunityResponse`, `ProvisionCommunityResponse`, `FeedbackSummary`, `ErrorEnvelope`) |
| Empty-result shape is `[]` / `{}` , never 404 | `/query`, `/count`, NIP-05, `/api/join-policy` | — |
| Request DTOs are permissive | 12 of 13 DTOs ignore unknown fields | `ReportQuery` alone uses `deny_unknown_fields` (`admin/mod.rs:64`) |

#### 4. Tenancy conventions

- **Row zero**: bind the community from `Host` before anything else. The phrase "Row zero" appears
  as a literal comment marker at `bridge.rs:621`, `:888`, `:1321`, `:1773`; `media.rs:145`;
  `nip05.rs:32`; `router.rs:280`. Grepping for it is the fastest way to audit door coverage.
- **Never derive identity from `config.relay_url`'s host** — only its scheme. Three helper pairs
  encode this: `nip98_expected_url` (`bridge.rs:195-206`), `nip42_expected_relay_url`
  (`bridge.rs:225-231`), `media_base_url_for_tenant` (`media.rs:447-455`),
  `relay_url_for_tenant_host` (`nip05.rs:105-111`). Each has a paired test asserting the config host
  does **not** influence the output (`bridge.rs:2636-2654`, `:2749-2771`; `media.rs:1272-1280`;
  `nip05.rs:143-152`).
- Two deliberate exceptions, both documented in place: operator routes authenticate against
  `relay_operator_api_origin` and never bind a tenant (`operator.rs:57-60`); the admin surface is
  deployment-wide by design (`docs/admin/README.md:1-9`).

#### 5. Comment / documentation conventions

- Doc comments on handlers state method + path + auth mechanism as the first line
  (`bridge.rs:612`, `:877`, `:1310`; `media.rs:589-601`; `invites.rs:225-229`).
- Security-relevant decisions are argued inline at length, often citing the attack they close and
  the PR/review that found it: `bridge.rs:184-193` (NIP-98 host binding), `:208-224` (NIP-42
  sibling), `:1582-1594` (search post-filter), `:595-611` (log truncation), `media.rs:145-156`
  (bind-before-verify ordering), `invites.rs:36-43` (limiter capacity rationale),
  `invite_token.rs:24-46` (security properties **and non-properties**).
- Tests carry "bites if …" statements naming the regression they detect
  (`bridge.rs:2326-2328`, `:2414-2416`, `:2524-2528`, `:2706-2707`, `:3396-3400`).
- Module headers enumerate routes (`media.rs:3-8`, `invites.rs:3-13`, `mesh_demo.rs:1-23`).
- `// sadscan:disable np.postgres.1` suppresses the hardcoded-credential scanner on test DB URLs
  (`bridge.rs:3283`, `invites.rs:429`, `operator.rs:589`).

#### 6. Test conventions

**Counts (all 13 assigned files):** 159 test functions, **28** `#[ignore]`d, 0 `unsafe` blocks
(one occurrence of the word in a doc comment at `bridge.rs:303`), **1** TODO marker
(`media.rs:303`), **0** `unwrap()` outside `#[cfg(test)]`, **5** `expect()` outside `#[cfg(test)]`
(all in `invite_token.rs`: `:119`, `:139`, `:172`, `:349`, `:374` — every one an
infallible-by-construction HMAC/serialize call).

| File | tests | `#[ignore]` |
|---|---|---|
| `bridge.rs` | 64 | 8 |
| `media.rs` | 33 | 0 |
| `invites.rs` | 14 | 9 |
| `operator.rs` | 11 | 11 |
| `admin/mod.rs` | 10 | 0 |
| `webhook_secret.rs` | 10 | 0 |
| `invite_token.rs` | 9 | 0 |
| `api/mod.rs` | 3 | 0 |
| `nip05.rs` | 2 | 0 |
| `mesh_demo.rs` | 2 | 0 |
| `admin/auth.rs` | 1 | 0 |
| `admin/error.rs`, `events.rs` | 0 | 0 |

Established patterns:

1. **`#[ignore = "requires Postgres"` / `"requires Redis"`** is the gate for anything touching real
   infrastructure — 28 tests. Every `operator.rs` test is ignored, so the operator surface has
   **zero** coverage in `just test-unit`.
2. **Router-level `oneshot`** via `tower::ServiceExt` drives real HTTP through `build_router`, so
   route registration + extractor order + middleware are all in scope:
   `bridge.rs:3372-3390`, `invites.rs:598-620`, `operator.rs:635-660`, `admin/mod.rs:375-391`,
   `media.rs:1000-1010`.
3. **Injected `Nip98ReplayGuard` doubles** instead of Redis — four distinct doubles:
   `AlwaysErrGuard` (`bridge.rs:2348`), `AlwaysFreshReplayGuard` (`bridge.rs:3286`,
   `invites.rs:415`, `operator.rs:551`), `SeenOnceReplayGuard` (`invites.rs:1103`). The fail-closed
   test needs **no infrastructure** because the double supplies the error (`bridge.rs:2321-2377`).
4. **Positive controls beside every negative test** so a cross-host rejection test cannot pass
   vacuously (`bridge.rs:2477-2504`, `:2731-2747`).
5. **`current_thread` runtime + `metrics::with_local_recorder`** for metric assertions, because the
   recorder is a thread-local that a multi-thread scheduler would lose — reasoning spelled out at
   `bridge.rs:3255-3262`.
6. **Custom `tracing` capture writer** to assert the exactly-one-attribution-line invariant
   (`bridge.rs:3512-3584`).
7. **Test state builders** named `*_test_state` that mutate `Config::from_env()` then override
   `state.nip98_replay`: `bridge_handler_test_state` (`bridge.rs:3304`),
   `invite_test_state` (`invites.rs:441`), `operator_test_state` (`operator.rs:591`),
   `test_state_with_media_get_auth` (`media.rs:951`), `test_state` (`admin/mod.rs:335`).
8. **Silent skip vs panic is inconsistent**: `invites.rs:664-666` and `operator.rs:686-688` do
   `let Some(state) = … else { return; }` (test passes vacuously when Postgres is absent), while
   `bridge.rs:3423-3425` and `invites.rs:1074-1076` `panic!`/`expect` with an actionable message.
9. **Community isolation asserted explicitly** for every per-pubkey limiter
   (`media.rs:1120-1161`, `invites.rs:481-503`) and for the replay seen-set (`bridge.rs:2290-2292`).
10. Helper `nip98_auth_header(keys, url, method, body)` is **duplicated** in `invites.rs:505-519`
    and `operator.rs:596-616` with near-identical bodies, plus a third variant
    `build_nip98_event_json` + `nip98_auth_headers` in `bridge.rs:2380-2404`.

#### 7. Naming conventions

| Pattern | Examples |
|---|---|
| `verify_*` — cryptographic check returning `Result` | `verify_bridge_auth`, `verify_secret`, `verify_invite`, `verify_policy_acceptance` |
| `enforce_*` — check that returns an HTTP error tuple | `enforce_http_admission`, `enforce_relay_membership` |
| `authorize_*` — auth prelude returning the authenticated principal or tenant | `authorize_moderation_read`, `authorize_operator_request`, `authorize` (admin) |
| `extract_*` — pull an optional value out of raw JSON / headers | `extract_before_id`, `extract_depth_limit`, `extract_feed_types`, `extract_thread_cursor`, `extract_search_mode`, `extract_search_page`, `extract_page_offset`, `extract_channel_from_filter`, `extract_blossom_auth`, `extract_secret`, `extract_nip_oa_owner`, `extract_domain` |
| `*_json` — hand-rolled row→`Value` projection | `report_json`, `action_json`, `ban_json` |
| `handle_*_filter` / `handle_bridge_*` — one `/query` dispatch branch | `handle_channel_window_filter`, `handle_bridge_search` |
| `*_authed` — post-authentication continuation | `submit_event_authed`, `query_events_authed`, `count_events_authed` |
| `is_*` — pure boolean predicate | `is_safe_ext`, `is_sha256`, `is_admin_host`, `is_member_tag` |
| SCREAMING_SNAKE consts co-located with the code that reads them | `BRIDGE_FEED_MAX_LIMIT`, `MODERATION_READ_LIMIT`, `MAX_RANGE_CHUNK`, `CLAIM_RATE_LIMIT`, `MAX_INVITE_TTL_SECS`, `ECHO_TIMEOUT` |

#### 8. Convention violations / inconsistencies

| Issue | `file:line` |
|---|---|
| Three incompatible handler error dialects in one module tree (tuple / `MediaError` / `ApiError`) | see §1 |
| Two incompatible error-envelope JSON shapes | `api/mod.rs:19-21` vs `admin/error.rs:16-28` |
| camelCase only in the admin sub-tree | `admin/mod.rs:64`, `:140` |
| Stringly-typed status classification in the operator provisioning path | `operator.rs:180-199` |
| Tenant-binding preamble copy-pasted 9 times; factored only in `media.rs` | `media.rs:478-488` vs the rest |
| Stale `#[allow(dead_code)]` on a live function | `api/mod.rs:28` |
| `#[allow(private_interfaces)]` placed **between** two doc-comment blocks, splitting the doc | `media.rs:299-304` |
| `let _pubkey = …` discards the acting operator on transfer; archive/unarchive discard it entirely | `operator.rs:355`, `:209`, `:271` |
| `nostr_nip05` folds DB errors and misses into the same 200 via a catch-all `_ =>` arm | `nip05.rs:64` |
| `deny_unknown_fields` used on exactly one DTO | `admin/mod.rs:64` |
| `serve_blob_for_tenant` re-runs `validate_media_path` that its two callers already ran | `media.rs:604`/`:619`, `:630` |


## Module: buzz-relay — git hosting (`crates/buzz-relay/src/api/git`)
### Aspect: Conventions

#### 1. Module layout

| File | LOC | Layer | Depends on |
|---|---|---|---|
| `mod.rs` | 66 | wiring + `require_localhost` | `policy`, `transport` |
| `transport.rs` | 2 288 | HTTP handlers, subprocess runners, fence | all others |
| `cas_publish.rs` | 1 891 | commit protocol | `manifest`, `store`, `transport::harden_git_env` |
| `store.rs` | 1 164 | object-store primitives + conformance probe | none (leaf) |
| `hydrate.rs` | 893 | read/write materialization | `manifest`, `store`, `pack_cache`, `cas_publish::ParentState` |
| `policy.rs` | 775 | HMAC callback endpoint | `buzz_core::git_perms`, `buzz_db` |
| `pack_cache.rs` | 686 | bounded local cache | `hydrate` (`pub(super)` helpers), `store` |
| `manifest.rs` | 570 | schema + predicates | leaf except `CommunityId` |
| `manifest_event.rs` | 395 | manifest → kind:30618 | `nostr` only (pure) |
| `hook.rs` | 207 | bash hook literal + installer | none |

Dependency direction is mostly clean, with two documented back-edges: `pack_cache` → `hydrate` (`hydrate.rs:272`, `:381` are `pub(super)` for exactly this) and `cas_publish`/`hydrate` → `transport::harden_git_env` (a `pub(crate)` helper living in the HTTP layer, which is where it least belongs).

#### 2. Error handling

Three distinct idioms, one per layer:

| Layer | Error type | Idiom |
|---|---|---|
| HTTP handlers | `Result<Response, Response>` | errors *are* responses, built at the failure site. `#[allow(clippy::result_large_err)]` is applied where needed (`transport.rs:265`, `:317`, `:1413`) |
| Protocol layers | `thiserror` enums | `CasError` (7 variants, `cas_publish.rs:90-145`), `HydrateError` (5, `hydrate.rs:96-121`), `StoreError` (6, `store.rs:65-102`), `ManifestError` (9, `manifest.rs:74-131`), `BuildError` (2, `manifest_event.rs:44-52`) |
| Hook installer | `anyhow::Result` | the only `anyhow` in the module (`hook.rs:152`) |

Conventions worth naming:

- **Variants encode HTTP class, not just cause.** `CasError::Conflict` is deliberately separate from `Backend` so `?`-bubbling cannot turn a 412 into a 500 (`cas_publish.rs:105-124`); `ManifestInvalid` is 4xx-class while `ManifestReadFailed` is 5xx-class (`:126-141`). `finalize_push` maps all seven (`transport.rs:1601-1657`).
- **"Not found" is a type, not a variant.** `hydrate_for_read` returns `Result<Option<HydratedRepo>>` so 404-vs-5xx is enforced by the type system (`hydrate.rs:94-97`).
- **Semantic non-errors are values.** `CasOutcome::LostRace` for HTTP 412 (`store.rs:52-64`); a 412 on a content-addressed PUT is folded into `Ok(key)` (`store.rs:271-275`).
- **Fail closed on ambiguity.** A 2xx CAS without an ETag becomes an error rather than `ETag("")` (`store.rs:522-536`); `get_pointer` errors on a missing ETag (`:462-471`); the policy endpoint returns 403 for *every* failure including DB errors (`policy.rs:277-282`).
- **Non-fatal degradations are explicit `warn!` + continue**, never swallowed: idx sidecar write (`cas_publish.rs:1148-1152`, `:828-844`), idx validation failure (`hydrate.rs:398-406`), kind:30618 build/insert (`transport.rs:1712-1735`), compaction fallback (`cas_publish.rs:1096-1103`).
- **Error strings are opaque to clients.** All git 5xx bodies are the literal `"git error"` (`transport.rs:630-700`, `:1004-1124`); detail goes to `tracing`. Policy denials leak only role/rule text.
- **`?` is used freely inside protocol layers; handlers use explicit `match` + `map_err`** so each arm can pick a status.

#### 3. Panic policy

AGENTS.md forbids new `unwrap()`/`expect()` in production paths. Current production count (outside `#[cfg(test)]`): **17**.

| File | Count | Lines | Assessment |
|---|---|---|---|
| `transport.rs` | 10 | 97, 108, 334, 572, 721, 1037, 1442, 1469, 1493, 1507 | 7 are `Response::builder().body(..).unwrap()` (infallible); 3 are `child.stdin/stdout.take()` immediately after `Stdio::piped()` (infallible by construction, one of them written as `.unwrap()` rather than `.expect()`) |
| `store.rs` | 5 | 262, 297, 490, 892 (`"*".parse().unwrap()` — const header value), 718 (`new_etag.expect("winner exists")` — guarded by `winners == 1` two lines above) | all structurally infallible |
| `cas_publish.rs` | 1 | 410 — `pack_path.to_str().unwrap()` | path is `tempdir/pack-<hex>.pack`; the sibling compaction path handles the same case with a real error (`:702-705`), so this one is an inconsistency |
| `policy.rs` | 1 | 126 — `Hmac::new_from_slice(..).expect("HMAC can take key of any size")` | infallible for HMAC |
| `hydrate.rs`, `pack_cache.rs`, `manifest.rs`, `manifest_event.rs`, `hook.rs`, `mod.rs` | 0 | — | clean |

Two additional panic-avoidance conventions:

- `pack_cache` never `unwrap()`s a poisoned mutex: `self.state.lock().unwrap_or_else(|e| e.into_inner())` at `:242`, `:335`, `:350`, `:367`.
- `pkt_line` degrades an over-long payload to an empty `0004` frame + `error!` rather than panicking or truncating a length prefix, "non-panicking in every build profile" (`transport.rs:424-454`, pinned `:2231-2237`).

`unsafe`: **0** occurrences anywhere in the module (the only matches for the string are `unsafe_refname` identifiers and error text).

`TODO`/`FIXME`/`XXX`/`HACK`: **0** occurrences.

#### 4. Streaming patterns

Two mutually exclusive strategies, chosen by whether the operation mutates published state (`transport.rs:1244-1260`, `:1405-1412`):

**Read paths — stream.** `stream_git_read` (`transport.rs:1414-1498`) composes four layers:
1. `ReaderStream` over `ChildStdout`.
2. `TimedByteStream` — hard deadline, byte/duration histograms in `Drop` (`:1282-1391`).
3. `StreamingGit` — parks the `Child` and the `HydratedRepo` to extend their lifetimes past the last byte, aborts the stdin pump on `Drop`, and kills the child when it observes a `TimedOut` item (`:1226-1332`).
4. `GitPermitStream` — holds the semaphore permit to EOF (`:1293-1310`).

**Write path — buffer.** `run_git_at` returns an owned `PackOutput` (`transport.rs:971-991`) precisely so no `Response` can exist before the CAS. The rationale is documented at the type (`:963-970`) and at the streaming helper (`:1244-1260`).

Request bodies are always pumped by a **spawned task**, never awaited inline, so stdin backpressure cannot deadlock against stdout reads (`transport.rs:1039-1064`, `:1442-1467`). Both pumps log body/decode errors at `warn` and then close stdin so git sees EOF rather than an opaque hang.

Body decoding is a stream transformer with a running counter, not a buffer-then-check (`transport.rs:766-783`).

#### 5. Temp-file and cleanup discipline

| Rule | Practice |
|---|---|
| Scratch location | Every tempdir/tempfile is created with `*_in(scratch_dir)` so it lands on the mounted volume, never `/tmp`. `cas_publish` derives its scratch as `repo_path.parent()` rather than taking another argument (`cas_publish.rs:1028-1030`). |
| Subprocess output | `NamedTempFile::new_in(...)` + `.reopen()` → `Stdio::from(file)`. The `NamedTempFile` handle stays in scope so `Drop` unlinks it; the reopened descriptor is what the child writes. Repeated verbatim at 8 sites (`transport.rs:622-644`, `:996-1018`; `cas_publish.rs:272-284`, `:511-523`, `:595-607`, `:704-712`). |
| Metadata-before-read | Every buffered read is `tokio::fs::metadata(...).len()` → size check → `tokio::fs::read(...)`, never read-then-check (`transport.rs:680-701`, `:1085-1126`; `cas_publish.rs:307-315`, `:552-568`). |
| Bounded stderr | `read_log_prefix` / `read_prefix` cap at 64 KiB using `AsyncReadExt::take`, returning `"<stderr unavailable>"` on failure — duplicated in two files (`transport.rs:1141-1155`, `cas_publish.rs:877-891`). |
| Lifetime-as-cleanup | `HydratedRepo` owns its `TempDir` (`hydrate.rs:51-56`); the explicit `drop(repo)` / `drop(ctx.repo_handle)` calls at `transport.rs:707`, `:1576`, `:1750` are load-bearing ordering, and are commented as such. |
| `kill_on_drop(true)` | On 6 of 10 subprocess sites; absent on the four `.output()`/`.status()` sites that already await completion (`cas_publish.rs:284`, `:337`, `:409`) — except `hydrate::run_git`, which sets it despite using `.output()` (`hydrate.rs:453`). |
| Cache publication | staging `TempDir` → atomic `rename` into the final digest directory; on rename failure the winner's directory is adopted (`pack_cache.rs:274-333`). |
| Process-lifetime GC | Startup sweep of stale `session-*` dirs plus a 60 s heartbeat writer aborted in `Drop` (`pack_cache.rs:127-146`, `:420-426`, `:482-509`). |
| Bash hook | `WORK_DIR=$(mktemp -d)` with `trap 'rm -rf "$WORK_DIR"' EXIT` (`hook.rs:49-53`). |

#### 6. Concurrency conventions

| Mechanism | Scope | Site |
|---|---|---|
| `git_semaphore` | global, `try_acquire_owned` (never blocks), 503 on exhaustion | `transport.rs:318-338` |
| `PACK_COMPACTION_SEMAPHORE` | process-global `const_new(1)`, `acquire` with 300 s timeout | `cas_publish.rs:86`, `:576-586` |
| `population_semaphore` | per-cache, bounds concurrent object-store pack fetches | `pack_cache.rs:110`, `:253-269` |
| Per-digest single-flight | `DashMap<String, Arc<PopulationFlight>>` with an `AtomicUsize` refcount and an RAII `FlightParticipant` that deregisters on the last drop | `pack_cache.rs:76-104`, `:186-238`, `:392-399` |
| **No per-repo lock** | deliberate; the CAS is the only writer serialization, with a 14-line justification comment | `transport.rs:865-877` |
| Sync mutex in async | `std::sync::Mutex` for cache index (never held across `await`) | `pack_cache.rs:63`, `:241-390` |

#### 7. Documentation conventions

- Every file opens with a `//!` module doc that maps code to the spec by section (`cas_publish.rs:1-49` walks §Push steps 1–8; `hydrate.rs:1-27` walks §Read).
- Invariant names from the TLA+ model (`Inv_NoFork`, `Inv_Closed`, `Inv_RefEffectApplied`, `Inv_RefDerivedFromParent`) are cited inline at the code that establishes them (`cas_publish.rs:200-213`, `:897-906`; `transport.rs:832-856`).
- Negative documentation is a first-class pattern: "What this function deliberately does *not* do" (`cas_publish.rs:34-49`), the `SECURITY:` note explaining why method binding is tautological (`transport.rs:174-183`), the "no advisory lock — by design" block (`transport.rs:865-877`).
- Named-reviewer attribution appears in production comments ("Max's blocker", "Eva's call, on record in #proj-git-on-s3", "Sami #2 / Max / Dawn") — `transport.rs:530-537`, `:875-877`; `cas_publish.rs:1171-1176`. This ages badly and encodes decisions in people's names rather than in the argument.
- All public items carry doc comments, satisfying the AGENTS.md rule.
- Two module doc comments are **stale**: `store.rs:25` ("wired in … in a follow-up commit") and `hydrate.rs:24-30` (describes an `#[allow(dead_code)]` that no longer exists).

#### 8. Test conventions

Totals: **99 unit tests** in-module (76 `#[test]`, 23 `#[tokio::test]`), **0** `#[ignore]`d. Plus 2 `#[ignore]`d live E2E tests in `crates/buzz-test-client/tests/e2e_git.rs:195-475`.

| File | `#[test]` | `#[tokio::test]` | Notes |
|---|---|---|---|
| `manifest.rs` | 23 | 0 | schema + predicates + byte-pinning |
| `transport.rs` | 20 | 7 | gzip decode, pkt-line framing, report-status de-framing, NIP-98 host binding, fast-path eligibility |
| `cas_publish.rs` | 16 | 2 | pure composition + 2 that shell out to real `git` |
| `policy.rs` | 14 | 0 | 12 HMAC unit + 2 cross-language bash/Rust |
| `manifest_event.rs` | 9 | 0 | tag shape and filtering |
| `store.rs` | 6 | 4 | 2 pure (`classify_cas`) + 2 config + 4 live-MinIO probes |
| `pack_cache.rs` | 5 | 3 | eviction, symlink guard, flight coalescing |
| `hydrate.rs` | 2 | 6 | predicates + 4 live-MinIO roundtrips |
| `hook.rs` | 1 | 0 | Dockerfile parse assertion |
| `mod.rs` | 0 | 0 | **`require_localhost` is untested** |

Conventions:

- **Environment-gated live tests instead of `#[ignore]`.** `probe_enabled()` reads `BUZZ_GIT_S3_PROBE == "1"` and returns early otherwise, so the tests always "pass" in CI. Duplicated in three files (`store.rs:996-998`, `hydrate.rs:579-581`, `cas_publish.rs:1578-1580`). Cost: a silently skipped test is indistinguishable from a passing one in CI output — only `store.rs:1020-1022` prints a skip notice.
- **Named mutation-bite tests.** Tests are written to fail under a specific regression and say so: `git_nip98_rejects_token_signed_for_wrong_community_host` (`transport.rs:2101-2122`), `same_owner_repo_pointers_do_not_bleed_between_communities` (`manifest.rs:505-524`), `validate_invoked_between_compose_and_put_manifest` (`cas_publish.rs:1833-1878`).
- **Byte-pinning.** Canonical manifest bytes (`manifest.rs:544-568`, `:367-385`) and pkt-line framing against a stated git-2.51 oracle (`transport.rs:2239-2277`) are asserted literally.
- **Cross-language verification.** The bash HMAC is re-implemented inside the test and diffed against Rust (`policy.rs:592-773`) — the only test of the module's most security-critical contract, and it depends on `bash` + `openssl` being present.
- **Real-`git` helpers.** `run_test_git` asserts success and applies `harden_git_env` (`cas_publish.rs:1530-1541`); `build_source_repo` constructs a real repo and extracts its pack (`hydrate.rs:595-640`).
- **Test-only accessors** are gated: `GitPackCache::flight` is `#[cfg(test)]` (`pack_cache.rs:401-418`).
- Test module names are mostly `tests`, except `transport.rs`'s `track_c_tests` (`:1768`) and `store.rs`'s second module `probe` (`:984`), which document *why* they exist.

#### 9. Naming and style conventions

| Convention | Example |
|---|---|
| Spec vocabulary in identifiers | `ParentState`, `CasSuccess`, `CasOutcome`, `Precond`, `PublishLimits` |
| `m_before` / `m_after` mirroring the spec | `cas_publish.rs:1105`, `:1177` |
| Predicate functions named `is_*` | `is_safe_refname`, `is_hex_oid`, `is_pack_key`, `is_manifest_digest`, `is_emittable_ref`, `is_valid_oid`, `fast_path_eligible`, `should_compact`, `compacted_pack_set_is_usable` |
| `*_inner` for the metric-wrapped body | `hydrate_for_read` → `hydrate_for_read_inner` (`hydrate.rs:124-168`); `cas_publish` → `cas_publish_inner` (`:997-1021`) |
| Limit constants `SCREAMING_SNAKE` with a doc paragraph explaining the number | `transport.rs:42-59`, `cas_publish.rs:81-86`, `manifest.rs:34-47` |
| `_`-prefixed fields that exist only for lifetime | `_tempdir`, `_repo`, `_permit`, `_temporary`, `_session_dir` |
| `#[derive(Debug, Clone, Copy)]` on small option structs | `PublishLimits`, `PublishOptions`, `HydrationOptions` |
| Options bundled into a struct rather than long argument lists | `HydrationOptions` (`hydrate.rs:79-89`), `PublishLimits` (`cas_publish.rs:147-155`) |

#### 10. Convention violations and inconsistencies

| # | Issue | Site |
|---|---|---|
| 1 | 17 production `unwrap()`/`expect()` against the AGENTS.md rule (all structurally infallible, but the rule is absolute) | §3 above |
| 2 | Two distinct git-env hardening implementations that claim to match but do not (`GIT_CONFIG_GLOBAL` missing in one) | `transport.rs:294-310` vs `hydrate.rs:451-465` |
| 3 | `read_log_prefix` and `read_prefix` are the same 14-line function in two files | `transport.rs:1141-1155`, `cas_publish.rs:877-891` |
| 4 | `probe_enabled()` triplicated | `store.rs:996`, `hydrate.rs:579`, `cas_publish.rs:1578` |
| 5 | `tenant()` test helper duplicated three ways | `hydrate.rs:536`, `cas_publish.rs:1603`, `transport.rs:1902` |
| 6 | Path-to-`&str` handled inconsistently: `.unwrap()` in one place, a typed error two functions away | `cas_publish.rs:410` vs `:702-705` |
| 7 | Module-wide `#![allow(dead_code)]` in `store.rs` will hide any future genuinely-dead item | `store.rs:25` |
| 8 | `harden_git_env` lives in the HTTP transport module but is consumed by two storage layers | `transport.rs:302` |
| 9 | `stream_git_read` carries an unused `extra_args` parameter | `transport.rs:1418`, called with `&[]` at `:824` |
| 10 | `transport.rs` at 2 288 lines mixes auth extractor, pkt-line codec, three handlers, two subprocess runners, four stream adapters, and the fence | whole file |


## Module: buzz-relay — moderation, admin & background workers (`crates/buzz-relay/src`)
### Aspect: Conventions

---

#### 1. Handler signature shapes

Three distinct shapes coexist across 6 event handlers. Argument order is **not** consistent.

| Handler | Signature | Site |
|---|---|---|
| `moderation_commands::handle_moderation_command` | `(tenant, state, event) -> Result<(), String>` | `:91-95` |
| `relay_admin::handle_relay_admin_event` | `(tenant, state, event) -> Result<(), String>` | `:108-112` |
| `identity_archive::handle_identity_archive_event` | `(tenant, state, event) -> Result<(), String>` | `:40-44` |
| `report::handle_report_event` | `(tenant, event, state) -> Result<(), String>` — **state last** | `:44-48` |
| `product_feedback::handle` | `(tenant, event, state) -> Result<(), String>` — **state last** | `:19-23` |
| `push_lease::accept` | `(tenant, state, event, now: i64) -> Result<AcceptLeaseOutcome, AcceptError>` — **typed error, typed outcome, injected clock** | `:469-474` |

`push_lease::accept` is the only handler with a typed error and the only one taking an injected `now`. The other five call `SystemTime::now()` internally, making their freshness checks untestable without wall-clock manipulation.

Naming is also inconsistent: four handlers use `handle_<domain>_event`, `moderation_commands` uses `handle_<domain>_command`, `product_feedback` uses the bare `handle`.

`state` is always `&Arc<AppState>` except in `push_runtime`, which takes `&AppState` for helper functions (`push_runtime.rs:98`, `:125`, `:349`, `:531`) and `Arc<AppState>` by value for the two spawned loops (`:57`, `:312`).

---

#### 2. Error-string conventions

##### 2.1 Three competing prefix strategies

| Strategy | Handlers | Effect at the wire |
|---|---|---|
| **Self-prefixing** with `invalid:` / `error:` / `restricted:` | `moderation_commands` (via helpers `:548-558`), `report` (inline literals), `product_feedback` (inline literals) | ingest passes through verbatim |
| **Unprefixed**, ingest wraps as `invalid: {e}` | `relay_admin` (`ingest.rs:1837`), `identity_archive` (`ingest.rs:1943`) | authorization failures surface as `invalid: actor not authorized: …` — semantically wrong |
| **Typed error enum**, ingest maps | `push_lease` → `AcceptError::{Validation, Internal}` (`ingest.rs:187-195`) | correct 400/500 split |

`moderation_commands` is the only handler with named prefix helpers and the only one with a **test that pins the prefixes** (`moderation_commands.rs:669-680`):

```rust
fn authz_denial(e: anyhow::Error) -> String { format!("restricted: {e}") }   // :548
fn invalid(message: impl Into<String>) -> String { format!("invalid: {}", …) } // :552
fn error(message: impl Into<String>) -> String { format!("error: {}", …) }     // :556
```

`report.rs` and `product_feedback.rs` inline the same prefixes as string literals at 20+ sites, with no helper and no test pinning them.

##### 2.2 The `blocked:` prefix is unique and unprefixed

`blocked: you are banned from this community` appears at `moderation_commands.rs:139` (returned bare, not via `invalid()`), `:199` (as the disconnect reason), `ingest.rs:1648`, and `handlers/auth.rs:171` — four independent literals of the same string with no shared constant.

##### 2.3 Freshness error text is duplicated verbatim three times

```
"event timestamp out of range: created_at={event_ts}, now={now}, delta={}s (max ±120s)"
```
`moderation_commands.rs:117-120` (interpolating the named const), `relay_admin.rs:126-129` (hard-coded `±120s`), `identity_archive.rs:148-151` (hard-coded `±120s`).

##### 2.4 DB error text reaches clients

`error: database error: {e}` (`moderation_commands.rs:174` and 5 more), `database error: {e}` (`relay_admin.rs:137` and 6 more), `restricted: {e}` where `e` may be a `sqlx` error (`moderation_commands.rs:549` wrapping `moderation_authz.rs:99`). Only `push_lease.rs:572` deliberately opaques it — and in doing so discards the diagnostic entirely.

---

#### 3. Tag-extraction conventions

Four near-duplicate helper families, all private, all reimplemented per file.

| Helper | Copies | Divergence |
|---|---|---|
| `extract_tag_value(event, name) -> Option<String>` | 3 — `moderation_commands.rs:608`, `relay_admin.rs:49`, `identity_archive.rs:189` | bodies are functionally identical; `identity_archive` uses `find_map`, the others use a `for` loop |
| p-tag extraction | 3 — `extract_p_tag_bytes` (`moderation_commands.rs:561`, returns `Vec<u8>`), `extract_p_tag_hex` (`relay_admin.rs:33`, returns `String`), `extract_single_p_tag_hex` (`identity_archive.rs:170`, returns `String` **and rejects a second `p` tag**) | three different contracts for the same tag |
| 64-hex validation | 4 inline copies — `moderation_commands.rs:567`/`:582`, `relay_admin.rs:41`, `identity_archive.rs:178`, plus typed variants `report.rs:211-220` (`decode_32_byte_hex`) and `push_lease.rs:365-374` (`check_exact_hex`, lowercase-only) | `push_lease` is the only one rejecting uppercase hex |
| tag-name matching | two idioms: `tag.as_slice().first().map(\|s\| s.as_str()) == Some("p")` (`moderation_commands.rs:564`, `relay_admin.rs:36`, `identity_archive.rs:173`) vs `tag.kind().to_string() == "imeta"` (`product_feedback.rs:24`, `:81`, `push_runtime.rs:263`) | the second allocates a `String` per tag per call |

Case handling also diverges: `relay_admin.rs:169` and `identity_archive.rs:58` lowercase the target pubkey with `to_ascii_lowercase()`; `moderation_commands.rs:561-574` does not (it accepts mixed-case hex and `hex::decode`s it, so the bytes normalize anyway).

---

#### 4. Logging conventions

##### 4.1 Level usage

| Level | Convention | Examples |
|---|---|---|
| `info!` | one line per successful privileged mutation | `moderation_commands.rs:223` (`"community ban applied"`), `:258`, `:325`, `:362`, `:497`; `relay_admin.rs:164`, `:203-209`, `:268-272`, `:327-332`; `identity_archive.rs:90-97`; `community_provisioning.rs:302-308`, `:336-343`; `workflow_sink.rs:316-321` |
| `warn!` | best-effort side effect failed | `relay_admin.rs:215`, `:218`, `:275`, `:278`, `:335`; `identity_archive.rs:131`, `:135`; `moderation_notices.rs:153`; `community_provisioning.rs:220-226`; `push_runtime.rs:63`, `:130`, `:187`, `:367`, `:386`, `:415`, `:427`, `:464`, `:467` |
| `error!` | loop-level failure that will retry | `push_runtime.rs:65`, `:83`, `:337`; `storage_sweep.rs:176-181`, `:195` |
| `info!` for a *failed* side effect | **inconsistent** — notice-DM failures use `info!`, not `warn!` | `moderation_commands.rs:217`, `:319`, `:493` |

The notice-DM failure at `info!` level is the outlier: a user who was banned and never told is an `info`, while a failed NIP-43 announcement is a `warn`.

##### 4.2 Structured field conventions

Consistent field names across the group: `sender`, `target`, `actor`, `operator`, `community`, `host`, `role`, `new_role`, `kind`, `error`, `wake`, `event_id`, `attempt`, `reaped`, `changed`. `%`-sigil display formatting is used for hex/UUID values (`moderation_commands.rs:223`, `push_runtime.rs:171`).

Notably **absent**: `moderation_commands` logs `target` but never the `actor` on success (`:223`, `:258`, `:325`, `:362`) — `relay_admin` logs both `sender` and `target` (`:203-209`). The moderation success lines are therefore not attributable from logs alone.

##### 4.3 Secret handling in logs

No file logs a pubkey secret, token, or ciphertext. Verified specifics:
- `relay_admin.rs:164` logs `icon_len`, not the icon value.
- `push_runtime.rs` never logs `endpoint_grant`; the closest is `wake=%claimed.id` (a UUID).
- `push_lease.rs` logs nothing at all — zero `tracing` calls in the file.
- `moderation_notices.rs` never logs the notice body.

---

#### 5. Concurrency and background-loop conventions

| Convention | Followed by | Site |
|---|---|---|
| `loop { claim → work → backoff }` with exponential idle backoff capped at 2 s | `push_runtime::run_matcher` (`:57-90`), `run_delivery_worker` (`:312-347`) | both reset the delay on finding work |
| Off-claim-path periodic sweep using `tokio::time::Instant::elapsed()` rather than a second task | `push_runtime.rs:59-68` | rationale at `:26-28` |
| Single-flight via a stored `JoinHandle` + `is_finished()` | `storage_sweep.rs:161-165` | harvest and spawn deliberately share one lock (`:143-149`) |
| Leader election via Postgres advisory lock | `storage_sweep` (through `main.rs:1414-1430`) | **not** followed by `push_runtime`, which runs on every pod |
| `Weak<AppState>` to break `Arc` cycles | `workflow_sink.rs:159-161` | rationale `:150-155` |
| Function-local `OnceLock` for localized feature config | `main.rs:1447-1453` | rationale `:1448-1451` |
| Cross-tick state in `AppState` behind `tokio::sync::Mutex` | `storage_sweep` (`state.rs:561`) | pattern documented at `storage_sweep.rs:128-130` |

Pure/impure separation is applied consistently in the two most-tested files: `decide_authority` is factored out of `authorize_moderation_action` "so it is exhaustively unit-testable" (`moderation_authz.rs:137-139`), `match_job` is documented as "Pure match evaluation: no DB access" (`push_runtime.rs:216-218`), `should_spawn` is a pure cadence predicate (`storage_sweep.rs:105-127`), and `resolve_mention_pubkeys` is a pure function (`workflow_sink.rs:45`).

---

#### 6. Test conventions

##### 6.1 Counts

| File | LOC | Tests | `#[ignore]` | Test-mod start |
|---|---|---|---|---|
| `handlers/moderation_commands.rs` | 768 | 10 | 0 | `:619` |
| `handlers/moderation_notices.rs` | 398 | 4 | 0 | `:310` |
| `handlers/moderation_authz.rs` | 335 | 7 | 0 | `:184` |
| `handlers/relay_admin.rs` | 468 | 15 | 0 | `:348` |
| `handlers/community_provisioning.rs` | 445 | 13 | 0 | `:354` |
| `handlers/push_lease.rs` | 771 | 10 | 0 | `:600` |
| `handlers/identity_archive.rs` | 580 | 6 | 0 | `:360` |
| `handlers/report.rs` | 337 | 6 | 0 | `:231` |
| `handlers/product_feedback.rs` | 161 | 4 | 0 | `:100` |
| `push_runtime.rs` | 656 | **2** | 0 | `:578` |
| `storage_sweep.rs` | 1090 | 15 | 0 | `:360` |
| `workflow_sink.rs` | 711 | 18 | **1** (`:613`) | `:368` + `integration_tests` `:560` |
| **Total** | **6,720** | **110** | **1** | |

Test density is wildly uneven: `storage_sweep` has 15 tests for 1090 LOC (~1 per 73), `push_runtime` has **2** for 656 LOC (~1 per 328) — and one of those two is an HTTP-level test of request-id stability, not of the delivery state machine. `deliver_one`'s 10-branch response handling and `retry_or_fail`'s backoff have zero coverage.

##### 6.2 Test-module structure

Standard: `#[cfg(test)] mod tests { use super::*; … }`. `workflow_sink.rs` is the only file with a second module — `#[cfg(test)] mod integration_tests` (`:560`) with a module-level doc comment naming the commits it regresses and the exact command to run it (`:561-567`).

##### 6.3 Event-builder helper convention

Every event-handling file defines a local `make_*` helper, all slightly different:

| Helper | Signature | Site |
|---|---|---|
| `make_event(kind: u16, created_at_secs: u64, tags: Vec<Vec<String>>)` | includes a timestamp | `moderation_commands.rs:646-657` |
| `make_test_event(kind: u16, tags: Vec<Vec<&'static str>>)` | no timestamp; needs `Box::leak` at call sites (`:391`, `:369`) | `relay_admin.rs:355-365`, `identity_archive.rs:363-373` |
| `report_with_tags(tags: &[&[&str]])` | `&str` slices | `report.rs:234-245` |
| `feedback(tags: Vec<Tag>)` | pre-built `Tag`s | `product_feedback.rs:107-115` |
| `event(tags: Vec<Tag>)` | fixed `created_at = 1000` | `push_lease.rs:604-611` |

The `Box::leak(hex.clone().into_boxed_str())` idiom (`relay_admin.rs:391`, `identity_archive.rs:369`) is a deliberate test-only leak to satisfy the `&'static str` parameter — a smell caused by the helper's signature, not by the test.

##### 6.4 Postgres-gated tests: three different strategies

| Strategy | File | Behaviour without Postgres |
|---|---|---|
| `#[ignore = "requires Postgres"]` | `workflow_sink.rs:613` | **skipped and reported as ignored** — correct |
| Silent `return` on connect failure | `identity_archive.rs:515-527` | **passes green** — three bailouts: `test_pool()` returns `None` (`:517-519`), a probe `SELECT` fails (`:520-526`), `test_state()` returns `None` (`:527-529`) |
| n/a | all others | pure unit tests |

`identity_archive.rs:515` is a false-green: the module's only integration test — and the only coverage of the live-kind:0 revocation rule — silently no-ops in CI without Postgres.

`storage_sweep.rs:381-397` uses a fourth variant: it returns early if any of the four `BUZZ_STORAGE_*` env vars is externally set (`:386-394`), with an honest in-code comment "externally forced — skip rather than assert a lie" (`:392`).

##### 6.5 Cross-artifact invariant tests (the strongest convention here)

Three tests assert Rust constants match non-Rust artifacts:

| Test | Asserts | Site |
|---|---|---|
| `resolve_audit_actions_are_allowed_by_db_check_vocabulary` | every 9044 action maps into `MODERATION_ACTION_CHECK_VOCAB`, with a failure message naming `migrations/0006_moderation.sql` | `moderation_commands.rs:659-667` |
| `migration_trigger_allowlist_matches_advertised_push_kinds` | `include_str!("../../../../migrations/0018_push_match_queue.sql")` contains the literal `NEW.kind IN (7, 9, 1059, 40007, 46010)` | `push_lease.rs:696-710` |
| `command_error_prefix_helpers_preserve_machine_readable_token` | the three wire prefixes | `moderation_commands.rs:669-680` |

The `include_str!` migration test is the most valuable pattern in the group — it makes Rust/SQL drift a compile-adjacent failure. It is applied once.

##### 6.6 Async-test conventions in `storage_sweep`

- `#[tokio::test(start_paused = true)]` for timeout and multi-tick behaviour (`:686`, `:743`).
- `tokio::task::yield_now()` between `maybe_spawn_sweep` calls, with an in-code explanation that harvest happens on the *next* call so two failures need three calls (`:507-513`).
- `async { panic!("must not spawn …") }` as an assertion that a future is never polled — used 5× (`:493`, `:546`, `:671`, `:760`, `:781`), relying on the documented guarantee that an unpolled async value has not started its body (`:151-153`).
- `metrics_util::debugging::DebuggingRecorder` + `metrics::with_local_recorder` + `futures::executor::block_on` to capture gauges (`:652-656`, snapshot helpers `:800-812`, `:820-838`).
- A bounded `for _ in 0..50 { … if finished { break } yield_now() }` poll loop instead of a fixed poll count, with rationale (`:697-711`).

##### 6.7 Documentation-in-tests convention

Test names encode the rule (`admin_cannot_ban_or_timeout_owner_or_fellow_admin`, `lowercase_expansion_does_not_shift_later_mentions`, `a_completed_but_unharvested_sweep_never_emits_its_snapshot`), and several tests carry multi-paragraph rationale comments naming the reviewer or counterexample that motivated them: "Wren's redteam counterexample" (`workflow_sink.rs:490-497`), "Quinn's re-review" (`:526-528`), "Wren's L5 lesson: never a UUID where an event id belongs" (`moderation_commands.rs:717`), "Rev 3 required tests" (`storage_sweep.rs:602-628`).

Two tests document *why a test was dropped* rather than deleting silently: `workflow_sink.rs:526-528` explains the two `ẞ→ss`-premised cases were vacuous because `ẞ` lowercases to one char.

---

#### 7. Documentation conventions

##### 7.1 Module-doc structure

Every file opens with `//!` docs. The moderation files follow a house style unique to this group: a summary table of kinds → operations → side effects, then an explicitly labelled pinned-contract section, then a lane-ownership footer.

| Convention | Examples |
|---|---|
| Markdown table of kinds/permissions in the module doc | `moderation_commands.rs:14-21`, `relay_admin.rs:9-16` |
| `## Routing (pinned — …)` / `## Tag vocabulary (pinned — …)` sections with a date and reviewer | `moderation_commands.rs:23-27`, `:29-55` |
| `Lane ownership: L<n> (<name>)` footer naming the owning engineer | `moderation_commands.rs:57-60`, `moderation_notices.rs:30`, `moderation_authz.rs:21`, `report.rs:23-25`, `buzz-db/src/moderation.rs:15-16` |
| Cross-lane coordination instruction | `moderation_commands.rs:58-60` ("coordinate, don't edit ingest.rs") |
| Reference to a `PLANS/` design doc | `moderation_authz.rs:4-5`, `storage_sweep.rs:5-7`, `:35`, `moderation_notices.rs:3` |
| Reference to a TLA+ spec | `report.rs:11`, `buzz-db/src/moderation.rs:12-13` |
| `## Privacy` invariant section | `moderation_notices.rs:20-23` |
| `DESIGN:` inline marker for a deliberate refusal | `relay_admin.rs:296-299` |
| Named-thread / event-id citation for a pinned decision | `moderation_commands.rs:33` ("thread event `86f46207`"), `workflow_sink.rs:561` ("`e3661764` / `7899c1a8`") |

This is unusually good provenance documentation. The cost is that several pinned claims have drifted from the code (see the api-surface and features aspects: the "reject channel-scoped API tokens" claim at `moderation_commands.rs:50`, the "recorded in the audit row" claim at `moderation_authz.rs:61`, and the "fan out through the existing 9005/9001 + 9040 paths" claim at `moderation_commands.rs:20`).

##### 7.2 Rationale-comment convention

Long inline comments explaining *why*, including accepted tradeoffs stated explicitly rather than hidden. Representative examples:
- accepted residual race, named as tolerated — `moderation_commands.rs:419-425`
- crash-safe but not concurrency-safe, with the follow-up named — `moderation_notices.rs:132-138`
- why discovery is re-emitted unconditionally — `moderation_notices.rs:141-151`
- why `limit: Some(1000)` and not `Some(1)` — `moderation_notices.rs:222-226`
- why the operator allowlist is deployment-root, not create-only, authority — `community_provisioning.rs:236-247`
- why the storage sweep respawns on every tick after a failure — `storage_sweep.rs:89-103`
- why harvest and spawn share one lock — `storage_sweep.rs:143-149`
- why the gateway request id must be stable across retries — `push_runtime.rs:486-490`
- why case folding must run in original-char coordinates — `workflow_sink.rs:78-96`
- why the deliberately non-standard `moderation_source` tag is not an `e` tag — `moderation_notices.rs:35-38`

##### 7.3 Public-API doc coverage

All `pub` items carry doc comments, satisfying the AGENTS.md rule. Verified across `push_lease.rs` (every `pub` struct field documented, `:24-81`), `storage_sweep.rs` (`:34-46`, `:105-127`, `:143-153`, `:258-282`), `moderation_authz.rs` (`:28-69`), `buzz-db/src/moderation.rs` (every field, `:37-170`).

---

#### 8. Rust-hygiene conventions (measured)

| Rule | Count in the 12 files | Detail |
|---|---|---|
| `unsafe` | **0** | none anywhere |
| `unwrap()` in production paths | **0** | all 49 occurrences are inside `#[cfg(test)]` modules |
| `.expect()` in production paths | **7** | `push_lease.rs:534`, `:539`, `:543`, `:548`, `:552` (all justified by prior validation — "validated active endpoint" etc.); `push_runtime.rs:316` (`"push HTTP client"`), `:514` (`"closed delivery body"`) |
| `panic!` in production paths | **0** | 6 occurrences, all test assertions (`storage_sweep.rs:493/546/671/760/781`, `workflow_sink.rs:631`) |
| `todo!` / `unimplemented!` | **0** | — |
| `TODO` / `FIXME` / `HACK` / `XXX` markers | **0** | the two `// TODO (WF-07)` markers live in `buzz-workflow/src/executor.rs:577`, `:582` — outside this module |
| `#[allow(…)]` | **0** in these 12 files | `#[allow(clippy::too_many_arguments)]` appears on the DB functions they call (`buzz-db/src/push.rs:210`, `archived_identities.rs:49`) |
| `unwrap_or(0)` on `SystemTime` | 3 | `moderation_commands.rs:115`, `relay_admin.rs:124`, `identity_archive.rs:146` — fails closed but produces `now=0` in the error string |

The 5 `.expect()` calls in `push_lease.rs:530-556` are a direct consequence of `LeasePlaintext` using `Option<T>` for fields that are mandatory when `active == true`. A `LeasePlaintext::Active { … } | Inactive { … }` enum would make them unrepresentable. This is the module's clearest type-modelling debt against the "no new `expect()` in production paths" rule.

---

#### 9. Deviations from repo-wide conventions (AGENTS.md)

| AGENTS.md rule | Compliance |
|---|---|
| No `unsafe` | ✅ 0 |
| No new `unwrap()`/`expect()` in production paths | ⚠️ 7 `.expect()` (5 in `push_lease.rs`, 2 in `push_runtime.rs`) |
| New public API must have doc comments | ✅ all `pub` items documented |
| Event kinds defined in `buzz-core/src/kind.rs` | ⚠️ `KIND_PUSH_LEASE` is defined **twice** — `buzz-core/src/kind.rs:109` and `push_lease.rs:19`; ingest imports the `push_lease` copy (`ingest.rs:204`, `:450`, `:2156`) while `req.rs:1734` imports the `buzz-core` one |
| Channels scoped by `h` tags, not `e` | ✅ `workflow_sink.rs:262-263`, `moderation_notices.rs:161`; `moderation_notices.rs:35-38` explicitly refuses to abuse `e` for a row UUID |
| Prefer Nostr events over new HTTP endpoints | ✅ 13 of 14 operations are event kinds; the one HTTP endpoint (`/operator/communities`) is justified in-code as necessarily above the tenant fence (`community_provisioning.rs:3-14`) |
| Thread counters (`reply_count`/`descendant_count`) updated by reply inserters | n/a — `workflow_sink` inserts only top-level (`depth: 0`, `workflow_sink.rs:333`) and `moderation_notices` inserts via `insert_event` with no thread metadata (`:174`) |


## Module: buzz-relay — huddle audio, tunnel & conformance seam (`crates/buzz-relay/src`)
### Aspect: Conventions

---

#### 1. Task / loop patterns

##### 1.1 Spawned tasks — exact inventory

| File | `tokio::spawn` sites | Which are production |
|---|---|---|
| `audio/join.rs` | 9 | **1** — the renewer at `join.rs:498` (reached via `attach_signals` at `:674`, `:688`, `:695`). The other 8 are in the test module |
| `audio/handler.rs` | 6 | **4** per connection: `send_loop` (`:663`), `heartbeat_loop` (`:667`), `audio_forward_loop` (`:670`), plus conditionally the owner-control reader (`:704`) and the owner-teardown watcher (`:733`). 1 test spawn |
| `audio/mesh.rs` | 1 | **1** — `spawn_remote_peer_sink` (`mesh.rs:272`), one per remote peer on the owner pod |
| `tunnel/reliable.rs` | 3 | **1** — the lease renewer (`reliable.rs:588`), which no production site invokes. 2 test spawns |
| `mesh_boot.rs` | 3 | **3** — the huddle-control accept task (`:271`), the reliable accept task (`:291`), and the drain watcher (`:485`) |
| `audio/room.rs`, `audio/wire.rs`, `tunnel/directory.rs`, `conformance/*` | 0 | — |

**Per audio connection: 4–6 tasks** (3 unconditional + reader + owner watcher).
At the 25-peer soft cap × N rooms this is the dominant task cost.

##### 1.2 Structured-concurrency discipline in `handler.rs`

Consistently applied: one root `CancellationToken` per connection
(`handler.rs:145`), child tokens for the loops that must die first
(`send_cancel` `:662`, `fwd_cancel` `:669`), and the parent cloned for loops that
must observe global cancellation (`hb_cancel` `:665`, `reader_cancel` `:688`,
`owner_cancel` `:732`). After `recv_loop` returns, `cancel.cancel()` then **every**
task is awaited (`handler.rs:787-800`) before cleanup runs — including the reader
task, so its clean-close reaches the owner before the peer is removed.

Deviation: the huddle-lease renewer is **detached**. `attach_signals` keeps only
`renewer.lost` (`join.rs:694`) and drops the `HuddleLeaseRenewer` struct, so its
`task: JoinHandle<()>` is never awaited. The struct exposes `.task` publicly
(`join.rs:467`) but no production code reads it. The same shape exists in the
reliable lane (`reliable.rs:200-205`).

Deviation: `spawn_remote_peer_sink` (`mesh.rs:272-283`) is fully detached — it
terminates only when its `mpsc::Receiver<Bytes>` closes, i.e. when the owner's
`Room` drops the peer. The teardown path relies on that implicit chain
(documented `join.rs:1341-1344`).

##### 1.3 `select!` conventions

`biased;` is used wherever cancellation or control must win over data:

| Site | Ordering |
|---|---|
| `handler.rs:187-193` | cancel → auth timeout |
| `handler.rs:952-956` | cancel → `ws_recv.next()` |
| `handler.rs:1071-1085` | cancel → control → data |
| `handler.rs:1099-1123` | cancel → peer control → audio |

Not biased: `heartbeat_loop` (`handler.rs:1130-1148`, tick vs. cancel — either order
is safe), the renewer loops (`join.rs:506-533`, `reliable.rs:592-621`), and
`serve_control_loop` (`join.rs:1156-1190`, where drain/lost arms are listed first
so the compiler's random polling still reaches them promptly).

A recurring idiom for an optional `select!` arm — a `pending()` future when the
token is absent, so the arm degenerates instead of the branch being duplicated:

```rust
let lost_fired = async {
    match &lost { Some(t) => t.cancelled().await, None => std::future::pending().await }
};
```

Used at `join.rs:1145-1155`, `join.rs:1167-1172` (roster), and
`handler.rs:735-748`. This is the module's signature pattern.

##### 1.4 Interval conventions

Both renewers set `MissedTickBehavior::Delay` (`join.rs:502`, `reliable.rs:591`) so
a stalled Redis call does not produce a burst of catch-up renewals.
`heartbeat_loop` (`handler.rs:1131`) does **not** set a missed-tick behaviour, so a
long stall can fire several ticks in a row and trip `MAX_MISSED_PONGS` spuriously.
The demo-echo drain poll uses a 100 ms interval (`mesh_boot.rs:315`); the mesh drain
watcher uses `sleep(500ms)` in a loop rather than an interval (`mesh_boot.rs:495`).

---

#### 2. Error handling

##### 2.1 No `unwrap` / `expect` / `panic` in most of the group

Counted over production code only (everything before the file's `#[cfg(test)]` mod):

| File | `unwrap()` | `expect(` | `panic!`/`unreachable!` | `unsafe` |
|---|---|---|---|---|
| `audio/join.rs` (1..1806) | 0 | 0 | 0 | 0 |
| `audio/room.rs` (1..556) | 0 | 0 | 0 | 0 |
| `audio/mesh.rs` (1..284) | 0 | 0 | 0 | 0 |
| `audio/wire.rs` (1..88) | 0 | 0 | 0 | 0 |
| `audio/mod.rs` | 0 | 0 | 0 | 0 |
| `tunnel/mod.rs` | 0 | 0 | 0 | 0 |
| `mesh_boot.rs` (1..521) | 0 | 0 | 0 | 0 |
| `conformance/tracers.rs` | 0 | 0 | 0 | 0 |
| **`audio/handler.rs` (1..1337)** | 0 | **6** | **1** `unreachable!` | 0 |
| **`tunnel/reliable.rs` (1..658)** | 0 | **1** | 0 | 0 |
| **`tunnel/directory.rs` (1..576)** | 0 | **2** | 0 | 0 |
| **`conformance/mod.rs` (1..430)** | 0 | **1** | 0 | 0 |
| **Total** | **0** | **10** | **1** | **0** |

The 10 `expect`s and the `unreachable!`:

| Site | Justification quality |
|---|---|
| `handler.rs:451` `pending_remote.expect("RemoteOwner matched above")` | Sound but fragile — the `if let` at `:448-450` already matched, then the value is re-taken |
| `handler.rs:457` `unreachable!("matched RemoteOwner above")` | Same pattern; a `let-else` re-destructure of a value proven to be `RemoteOwner` |
| `handler.rs:689` `remote_fence.expect("remote_fence set whenever remote_stream is")` | Invariant across three `Option`s set together at `:466-469`; not type-enforced |
| `handler.rs:692`, `:696`, `:701` `remote_session.expect(...)` ×3 | Same invariant, asserted three more times |
| `handler.rs:731` `state.mesh().expect("owner teardown watcher only exists when mesh owner state exists")` | Sound — guarded by `owner_lost.is_some() \|\| owner_draining.is_some()` at `:727` |
| `reliable.rs:471` `bytes[2..18].try_into().expect("16 byte community id slice")` | Infallible given the `len < 18` check at `:462`; could be `TryInto` on a fixed array |
| `directory.rs:261` `current.expect("renewed returns lease")` | Trusts the Lua contract — `renewed` always returns a non-empty value (`directory.rs:47`) |
| `directory.rs:291` `current.expect("released returns lease")` | Same for `released` (`directory.rs:65`) |
| `conformance/mod.rs:246` `row.expect("project_row_community returns None only for Some(ch)")` | Sound; a `match` would remove it |

Four of the six `handler.rs` `expect`s exist because `remote_session`,
`remote_stream`, and `remote_fence` are three parallel `Option`s that are always
`Some` together (`handler.rs:445-503`). A single `Option<struct{…}>` would make all
four unnecessary — this is the clearest local refactor available.

Repo rule from `AGENTS.md`: "Do not introduce new `unwrap()` or `expect()` in
production paths". The group is compliant on `unwrap()`, non-compliant on `expect()`
in four files.

##### 2.2 Lock-poisoning convention (inconsistent)

`audio/room.rs` uses three different strategies for the same mutex:

| Site | Strategy |
|---|---|
| `mark_ended` `:193` | `if let Ok(mut g)` → else return `false` |
| `clear_ended` `:202` | `if let Ok(mut g)` → else silently do nothing |
| `add_peer` `:229-231` | `.map_err(\|_\| AdmissionError::Ended)` — "poisoned ≈ shutting down" |
| `add_peer_at_index` `:282` | same |
| `remove_peer` `:338-340` | `let Ok(mut g) = … else { return }` — **the peer is never removed and its index leaks** |
| `remove_peer_and_check_ended` `:363` | `.ok()?` — returns `None`, caller treats it as "not ended" |
| `roster_snapshot` `:462` | `.unwrap_or_else(\|e\| e.into_inner())` — the only site that recovers through poisoning |

`conformance/tracers.rs:57-60` also recovers via `into_inner`, with an explicit
rationale (`:56`). Since nothing in the group can panic while holding the guard
(no user code runs inside the critical sections), poisoning is unreachable in
practice — but seven different handlings of one impossible case is noise, and two of
them (`remove_peer`, `clear_ended`) fail silently in a way that leaks state.

##### 2.3 Error type conventions

- Two `thiserror` enums: `DirectoryError` (6 variants, `directory.rs:141-176`) and
  `ReliableStreamError` (10 variants, `reliable.rs:529-568`). Both carry structured
  context (community, session, raw value) rather than strings.
- `ReliableStreamError` is annotated `#[allow(missing_docs)]` (`reliable.rs:531`) —
  the only doc-comment escape hatch in the group, against the `AGENTS.md` rule "New
  public API must have doc comments".
- Every `HuddleDirectory` method flattens `DirectoryError` into
  `MeshError::Transport(e.to_string())` (`join.rs:114`, `:139`, `:158`, `:172`),
  **losing the typed variant** — so a malformed lease and a Redis outage are
  indistinguishable to the huddle join path. Only `validate` preserves typed fence
  errors (`join.rs:180`), which is exactly what `FenceRejection::from_mesh_error`
  needs (`join.rs:996-1005`).
- `ensure_membership` returns `Result<Uuid, String>` (`handler.rs:1153-1158`) —
  stringly-typed, so the caller cannot distinguish "archived", "not linked",
  "not a member", and "db error"; all four collapse into the same WS
  `{"type":"error","message":"not a member"}` (`handler.rs:274-285`).
- `DialError` (`join.rs:1646-1653`) correctly splits a clean owner rejection from a
  transport failure, and the handler maps them to different WS errors
  (`handler.rs:474-503`).

##### 2.4 Fail-closed convention

Applied consistently at tenant and ownership boundaries:

| Decision | Fail-closed choice |
|---|---|
| Unmapped host | 404 with a generic message, never a default tenant (`handler.rs:69-88`) |
| Pre-join `get_channel` DB error | silent teardown, not admission (`handler.rs:404-410`) |
| Owner-ready loop exhaustion | transient error, never an ownerless owner peer (`join.rs:443-447`) |
| Ambiguous channel UUID on the media path | drop, never cross-tenant delivery (`room.rs:526-541`) |
| Missing row-community lookup | `ImplBug`, never substitute the resolved label (`conformance/mod.rs:245-254`) |
| Mesh bind / registry publish failure with mesh on | fatal boot (`mesh_boot.rs:423-463`) |

##### 2.5 Logging conventions

Structured `tracing` with `%`/`?` sigils and `channel_id`/`peer_id`/`session_id`
fields throughout. Production-code counts (excluding test modules):

| File | error | warn | info | debug | trace |
|---|---|---|---|---|---|
| `audio/handler.rs` | 1 | 22 | 7 | 6 | 1 |
| `audio/join.rs` | 0 | 4 | 0 | 4 | 0 |
| `audio/room.rs` | 0 | 1 | 0 | 0 | 0 |
| `audio/mesh.rs` | 0 | 1 | 0 | 3 | 0 |
| `tunnel/reliable.rs` | 0 | 4 | 0 | 0 | 0 |
| `tunnel/directory.rs` | 0 | 3 | 0 | 0 | 0 |
| `mesh_boot.rs` | 0 | 13 | 10 | 0 | 0 |
| **Total** | **1** | **48** | **17** | **13** | **1** |

`error!` is used exactly once, for a genuine invariant violation
(`handler.rs:590-600`), which matches the repo's severity discipline. The one
`trace!` is the per-frame v2 header dump (`handler.rs:996-1003`) — correctly at
`trace` so it is off by default.

Pubkeys are logged in full hex (`handler.rs:255`, `:283`, `:419`, …), consistent with
the rest of the relay and with the rationale in `conformance/mod.rs:66-68`.

---

#### 3. Backpressure conventions

The group has **two deliberately opposite** policies, and they are documented as
such:

| Lane | Policy | Sites |
|---|---|---|
| Realtime audio | **never queue, drop on full** | `try_send` at `room.rs:409`, `:427`, `handler.rs:1115` (`data_tx`), `handler.rs:1043` (Pong) |
| State-bearing control | **never drop; size generously and warn if it happens** | `mpsc::channel(32)` (`room.rs:45`), warning at `room.rs:441-446` |

Detail:

- Per-peer audio queue is 8 slots ≈ 160 ms (`room.rs:38-40`). A slow WS peer loses
  audio but never stalls the room.
- The WS-side data channel is 16 slots (`handler.rs:659`), the WS control channel 8
  (`handler.rs:660`).
- `audio_forward_loop` (`handler.rs:1093-1125`) bridges room→WS using `try_send` on
  **both** channels — so a full 8-slot WS control channel silently drops a
  `joined`/`left` message that the room-level 32-slot channel deliberately protected.
  The room's warning (`room.rs:441-446`) does not fire for this second drop point.
  This is the one inconsistency in an otherwise coherent policy.
- Roster deltas use `tokio::sync::broadcast` (cap 64, `room.rs:179`) with the
  `Lagged` → full-snapshot recovery pattern (`join.rs:1174-1182`), which is the
  correct shape for a lossy-but-recoverable ordered stream.
- Mesh media send is fire-and-forget with `debug!` on error
  (`join.rs:1762-1765`, `mesh.rs:277-282`) — no queue, no retry.
- Reliable-stream sends are `await`ed, so QUIC's own flow control provides
  backpressure (`reliable.rs:283-291`).
- The huddle-control accept task and reliable accept task are `tokio::spawn`ed per
  inbound stream (`mesh_boot.rs:270-274`, `:290-298`) with **no concurrency bound** —
  the dispatcher doc says handlers "must hand off promptly (spawn) rather than
  block" (`mesh_boot.rs:36-37`), and they do, but nothing caps how many.
- The datagram handler runs **inline on the accept task** (`mesh_boot.rs:236-245`),
  justified because `on_media_datagram` is synchronous and non-blocking. Verified:
  it is a `DashMap` lookup plus `try_send`s (`mesh.rs:204-250`).

---

#### 4. Test conventions

##### 4.1 Counts

| File | tests | `#[ignore]`d |
|---|---|---|
| `audio/join.rs` | 34 | 0 |
| `audio/room.rs` | 9 | 0 |
| `tunnel/directory.rs` | 7 | 0 |
| `audio/mesh.rs` | 6 | 0 |
| `tunnel/reliable.rs` | 6 | 0 |
| `mesh_boot.rs` | 5 | 0 |
| `conformance/mod.rs` | 9 | 0 |
| `audio/wire.rs` | 4 | 0 |
| `audio/handler.rs` | 2 | 0 |
| `audio/mod.rs`, `tunnel/mod.rs`, `conformance/tracers.rs` | 0 | 0 |
| **Total** | **82** | **0** |

All 82 are inline `#[cfg(test)] mod tests`. No `tests/` directory exists for this
group; no `#[ignore]` anywhere.

##### 4.2 Silent-skip on missing Redis — the dominant convention

`redis_directory_if_available()` pings Redis and returns `Option<SessionDirectory>`;
every integration test opens with `let Some(directory) = … else { return; }`:

- `tunnel/directory.rs:592-604`, used by 5 tests (`:650`, `:712`, `:779`, `:882`)
- `tunnel/reliable.rs:707-719`, used by 4 tests
- `api/mesh_demo.rs:169-179`, used by 2 tests

These tests **pass vacuously** when Redis is absent. `just test-unit` therefore
reports green while never executing the Lua scripts, generation monotonicity,
or the fence taxonomy. Only `just test` (Postgres + Redis) exercises them. An
`#[ignore]` + explicit opt-in, or a hard failure when `REDIS_URL` is set,
would make the gap visible.

##### 4.3 Test doubles

| Double | Where | Covers |
|---|---|---|
| `FakeDir` | `join.rs:1821-1900` | scripted `HuddleDirectory`: queued `owner_of`/`acquire`/`validate` results, a `VecDeque` of renew outcomes, and call counters. This is the reason 34 tests run without Redis |
| `ChanSend`/`ChanRecv` + `stream_pair()` | `join.rs:2088-2110` | an in-memory `MeshStream` pair over `mpsc::unbounded_channel`, built through the **public** `MeshStream::new` seam — drives `accept_inbound` end-to-end without iroh |
| `NullTransport` | `join.rs:2064-2079` | no-op `send_datagram`, erroring `open_session_stream` |
| `NoopTransport` / `DirectTransport` | `reliable.rs:734-757`, `:824-848`; `api/mesh_demo.rs:181-224` | `unreachable!()` on unexpected calls; `DirectTransport` wraps a real `MeshPeer` |
| `StubSend`/`StubRecv` | `mesh_boot.rs:552-568` | minimal stream for dispatcher routing tests |
| `VecTracer` | `conformance/mod.rs:441-450` | collects `TraceStep`s so `EmitGuard` Drop behaviour is observable |
| `install_for_test` | `join.rs:782-795` | `#[cfg(test)]`-gated registry entry with a caller-supplied `lost` token and **no renewer**, isolating fan-out from renewer timing |

`await_release_calls` (`join.rs:2440-2456`) polls a counter with a 2 s timeout
because the registry owns the renewer task and exposes no `JoinHandle` — a
sound workaround, and the comment says so.

##### 4.4 Naming and documentation convention

Test names are full sentences describing the invariant:
`admit_full_wins_over_version_mismatch`, `registry_release_is_generation_fenced`,
`owner_ready_waits_for_winner_install_then_reuses`,
`parse_clamps_out_of_range_level_keeps_frame`,
`wired_datagram_consumer_shares_the_handle_fence`. Most carry a doc comment
explaining *why* the invariant matters, several citing the reviewer who asked for
it (`room.rs:759-766` "Per Sami/Perci's review", `room.rs:663-665` "Per Max's
review checklist", `mesh_boot.rs:543-545` "Blocker fix (Wren review of 8b077fdb)").

`mesh_boot.rs:546-556` shows a good convention: rather than asserting a lie when the
environment is forced, the test skips —
`if std::env::var("BUZZ_MESH").is_ok() { return; }` with the comment
"externally forced — skip rather than assert a lie".

##### 4.5 Testing gaps (structural, not stylistic)

- `audio/handler.rs` has 1,337 production lines and **2 tests**, both covering
  peripheral concerns (semaphore budget `:1341-1358`, parser size limit
  `:1417-1427`). The 719-line `handle_active_audio_connection` — every WS error
  code, the whole join sequence, teardown ordering — is untested at unit level.
- No test asserts the emitted **JSON shape** of any WS frame. The `code`/`message`
  strings the desktop client branches on (`desktop/src-tauri/src/huddle/relay_api.rs:130-150`)
  are unpinned on the relay side.
- No test covers `emit_participant_event`'s four-step pipeline, including the
  duplicate-skip and insert-error-but-fan-out-anyway branches
  (`handler.rs:1285-1307`).
- `conformance/tracers.rs` has **zero tests** — `JsonlTracer`'s truncate-on-create,
  serialization, and poison recovery are unverified (and it has no callers either).
- No test exercises `wire_mesh_consumers`' huddle-control or reliable-stream arms;
  only the datagram arm is covered (`mesh_boot.rs:665-737`).

---

#### 5. Documentation conventions

Module-level docs are unusually rich and carry ASCII flow diagrams
(`audio/mod.rs:6-9`, `handler.rs:3-12`, `room.rs:3-6`) plus explicit
"why not the other way" reasoning: `mesh.rs:26-35` (the payload invariant),
`join.rs:22-38` (Redis is the arbiter, mesh is a hint), `room.rs:216-227`
(error precedence as an information-leak defence), `wire.rs:12-20` (threat-model
invariant). `mesh_boot.rs:404-410` even argues with itself in prose about whether a
misconfigured mesh should be fatal.

Two conventions worth naming:

1. **Invariants are pinned by a named test, and the doc comment cites it.**
   `room.rs:124-127` → "See `version_pin_persists_across_peer_churn` for the test
   that pins this behavior"; `mesh_boot.rs:667-673` → the load-bearing shared-fence
   test.
2. **Deliberate non-features are documented as such**, so a reader does not
   mistake absence for oversight: `join.rs:1281-1284` (why `UnregisterPeer` needs no
   fence), `conformance/mod.rs:128-146` (why `claimed_community` is `None` on the
   read path), `mesh_boot.rs:206-215` (why no renewer is wired).

Stale docs found: `conformance/mod.rs:46-48` says the `req.rs`/`event.rs` wire
points are "held back as additive patch" when `req.rs:144`/`:355`/`:671` have
landed; `conformance/mod.rs:51-53` claims a blake3 actor label that the code does
not compute; `room.rs:33` lists a `speakers` control message that is never emitted;
`mesh.rs:1-10` opens with "Today a huddle's audio only fans out within a single
pod … This module removes that wall", present tense on both sides of a change that
has already shipped.


## Module: buzz-relay-mesh (`crates/buzz-relay-mesh`)
### Aspect: Conventions

---

#### 1. Module layout and ownership annotation

Eight flat modules, no submodule nesting (`lib.rs:21-28`). The crate carries an
explicit **lane-ownership map in code** (`lib.rs:30-36`):

```
endpoint.rs, peer.rs                              — Mari (transport core)
registry.rs, gossip.rs, membership.rs, status.rs  — Max (membership + /_mesh)
```
with a note that the session directory and tunnel routing live relay-side (Perci)
and huddle fan-out in `buzz-relay`'s audio module (Dawn). This is an unusual
convention for this repo — no other crate embeds per-file human owners in source.
There is **no corresponding `.github/CODEOWNERS` entry** (verified: no mesh path in
CODEOWNERS), so the annotation is advisory only.

Each module opens with a `//!` header stating its contract and its *limits*, and the
limit is always the same one: "this cannot elect owners." See `membership.rs:1-6`,
`gossip.rs:1-5`, `registry.rs:1-7`, `runtime.rs:21-23`. That repetition is the
crate's dominant documentation convention.

#### 2. Error handling

- **One error enum, `thiserror`-derived.** `MeshError` (`lib.rs:65-127`), 16 variants,
  `#[derive(Debug, thiserror::Error)]`. No `anyhow` inside the crate — `anyhow`
  appears only at the consumer boundary (`mesh_boot.rs:412`).
- **Typed variants for every fence-visible reject.** Explicit policy at
  `lib.rs:102-109`: "every fence-visible reject is a typed variant, never a generic
  `Transport`, so live kill-9 / partition / replay evidence is unambiguous."
  Honoured for the four fence variants; *not* honoured for the rest — see below.
- **`MeshError::Transport(String)` is the catch-all and it is overused.** 22
  construction sites collapse ~8 distinct causes into one untyped string:
  iroh bind/connect/stream/datagram errors (`endpoint.rs:38,39,79,91,101`;
  `peer.rs:82,95,113,123,151,155,163,174,187`), attestation parse and verify
  failures (`registry.rs:56-83`), key/pubkey mismatch (`registry.rs:141`), JSON
  encode (`registry.rs:186`), Redis pool acquisition (`registry.rs:271`), missing
  datagram support (`peer.rs:109`), and unknown gossip payload version
  (`gossip.rs:72`). The last one is notable: the *outer* frame version gets a typed
  `UnknownWireVersion` (`wire.rs:185`) while the *inner* gossip version gets a
  string, so the two version channels are not observably symmetric.
- **`err.to_string()` discards iroh's structured errors** at all 12 transport sites,
  so callers cannot distinguish timeout from refused from TLS failure.
- **`#[from]` used exactly once**, for `redis::RedisError` (`lib.rs:126`); everything
  else is explicit `map_err`.
- **Errors are propagated, never swallowed silently** — with two deliberate
  exceptions, both commented: `try_send` on the gossip queue
  (`runtime.rs:556`, "dropping a frame under backpressure is strictly better than
  blocking a recv loop") and `encode_message(&delta)` failures being skipped
  (`runtime.rs:540`, `if let Ok(payload)`).

##### `unwrap` / `expect` policy

AGENTS.md forbids new `unwrap()`/`expect()` in production paths. **22 `expect()`
outside `#[cfg(test)]`, zero `unwrap()`:**

| File | Count | All of the form |
|---|---|---|
| `runtime.rs` | 13 (`:142,156,159,168,183,197,202,222,270,349,444,553,573`) | `.expect("peer lock poisoned")` / `"loop lock poisoned"` / `"handler lock poisoned"` |
| `membership.rs` | 9 (`:74,126,159,173,190,199,322,332,363`) | `.expect("membership lock poisoned")` / `"local record lock poisoned"` |

Every one is a `std::sync` lock-poison unwrap — the conventional accepted case (a
poisoned mutex means another thread already panicked while holding it). Still a
literal deviation from the stated rule, and it means a panic anywhere inside a
membership or peer critical section escalates to a panic in **every** subsequent
mesh operation, including the `/_mesh` handler. `parking_lot` or explicit
`unwrap_or_else(|e| e.into_inner())` recovery would remove the escalation.

`0` `unsafe` (verified: grep for `unsafe` in `src/` returns nothing) — compliant.
`0` `TODO`/`FIXME`/`XXX`/`HACK`/`todo!`/`unimplemented!` markers (verified). All
known gaps are expressed as prose in doc comments instead, which makes them
invisible to grep-based debt tooling.

#### 3. Concurrency and task patterns

**Nine `tokio::spawn` sites in production code**, all in `runtime.rs`:

| # | Task | Site | Lifetime |
|---|---|---|---|
| 1 | `accept_loop` | `:119` | tracked in `MeshRuntime.loops` |
| 2 | `reconcile_loop` | `:120` | tracked in `loops` |
| 3 | `gossip_tick_loop` | `:121` | tracked in `loops` |
| 4 | `datagram_recv_loop` (per peer) | `:233` | tracked in `PeerEntry.tasks` |
| 5 | `stream_accept_loop` (per peer) | `:234` | tracked in `PeerEntry.tasks` |
| 6 | `open_control_stream` (per dialed peer) | `:240` | tracked in `PeerEntry.tasks` |
| 7 | `control_stream_exchange` (accept side) | `:449` | **untracked — detached** |
| 8 | control-stream send pump | `:509` | held locally, aborted on recv-loop exit (`:549`) |
| 9 | registry heartbeat | `:599` | `JoinHandle` returned and **dropped** by the caller (`mesh_boot.rs:467`) |

Two more spawns exist in tests (`endpoint.rs:162`, `:198`).

Patterns and their gaps:

- **`JoinHandle` bookkeeping + explicit abort** is the intended discipline
  (`PeerEntry::abort`, `runtime.rs:56-62`; `MeshRuntime::shutdown`, `:155-164`).
  Broken in two places: task #7 is spawned without being pushed onto
  `PeerEntry.tasks`, so a peer removal aborts the recv loops but leaves the accept
  side's control exchange running until its stream errors; task #9's handle is
  discarded at the call site so the heartbeat can never be stopped.
- **`shutdown()` is never called in production** (`-api-surface.md` §7) — every loop
  above runs until process exit.
- **Infinite `loop { … sleep }` with no jitter and no backoff**: `reconcile_loop`
  (`:285-293`), `gossip_tick_loop` (`:563-587`), heartbeat (`:600-606`), and the
  relay-side drain watcher (`mesh_boot.rs:484-496`). All pods started together will
  gossip and rescan in lockstep.
- **Lock hygiene is careful about await points.** `send_datagram` explicitly
  `drop(peers)` before touching membership (`runtime.rs:172-173`);
  `open_session_stream` clones the `MeshPeer` out of a scoped read guard before
  awaiting (`:182-189`); `install_peer` drops the write guard before
  `mark_connection_state` (`:249-252`). No `std::sync` guard is ever held across an
  `.await` — verified by reading all guard scopes.
- **Two lock kinds by intent**: `RwLock` for read-mostly tables
  (`Inner.peers` `:69`, membership's two `:31-32`), `Mutex` for the write-once
  handler slot (`:70`) and the loop vector (`:79`). `Arc<AtomicU64>`/`AtomicBool` for
  counters and the draining flag (`membership.rs:33-35`), all `Ordering::Relaxed`
  (`membership.rs:181,286,309,310`; `peer.rs:29-34`) — appropriate for pure counters.
- **Bounded channels only.** `mpsc::channel(CONTROL_QUEUE_DEPTH = 64)`
  (`runtime.rs:46`, `:237`, `:442`) with `try_send` drop-on-full (`:556`). No
  unbounded channel anywhere.
- **Handles are cheap-clone `Arc` facades**: `MeshRuntime` (`:77-80`),
  `MeshMembership` (`:29-43`), `MeshPeer` (`peer.rs:38-43`), `ReadyRegistry`
  (`registry.rs:160-163`) all `#[derive(Clone)]` over shared inner state. Documented
  at `runtime.rs:75-76`.

#### 4. Wire-compatibility discipline

Codified in `wire.rs:1-32`:

- `wire.rs` is declared a **FROZEN surface**; "changes here require a post in the
  mesh thread **before** the edit" (`wire.rs:10-13`), with the stated failure mode
  being two lanes compiling against different frame layouts. Unenforced by tooling.
- **ALPN carries the version** — `buzz/mesh/1` (`wire.rs:37`) — so a version bump
  gets a new ALPN and "old and new pods never half-speak to each other during a
  rolling deploy" (`wire.rs:34-36`). This is the primary compatibility mechanism.
- **Version byte first, reject unknown loudly** (`wire.rs:38-41`, enforced
  `wire.rs:183-186`). "Receivers MUST reject unknown versions loudly (count it, log
  it) rather than guessing" — the *log* happens at the call site
  (`runtime.rs:406`, `:549`); the *count* does not exist.
- **Nested opacity for evolution**: gossip rides as `Vec<u8>` inside
  `MeshStreamFrame::Gossip` with its own in-struct version so the gossip lane can
  evolve without a wire bump (`wire.rs:139-141`, `gossip.rs:13`). Same trick for
  huddle control on the consumer side (`audio/join.rs:797-801`).
- **Documented invariants are stated as MUSTs in prose**: "first frame MUST be
  `Hello`" (`wire.rs:29-31`), "senders MUST check the encoded size … never truncate"
  (`wire.rs:21-22`), "receivers MUST reject stale generations at every hop"
  (`wire.rs:22-24`).
- **No `#[non_exhaustive]` on any public enum** (verified: 0 occurrences). The
  convention is "bump the ALPN," not "tolerate unknown variants" — internally
  consistent but leaves zero forward compatibility within a version.
- **Header-size budget pinned by test**, not by a const:
  `datagram_header_overhead_within_budget` asserts ≤64 B so it "can't silently grow
  past the budget" (`wire.rs:266-284`). Nice pattern; note it is one of the 32 tests
  CI never runs.

#### 5. Naming and API-shape conventions

- **`Mesh*` prefix** for crate-owned types (`MeshEndpoint`, `MeshPeer`, `MeshRuntime`,
  `MeshStream`, `MeshDatagram`, `MeshStreamFrame`, `MeshStatus`, `MeshError`,
  `MeshConfig`, `MeshMembership`). **`Relay*` prefix** for the two consumer-facing
  seam traits (`RelayMeshMembership`, `RelayPeerTransport`, `lib.rs:144`, `:158`) —
  the prefix marks "this is the relay's view," which is a genuinely useful signal.
- **Builder-by-`with_`** on `MeshMembership` (`with_expected_relay_pubkey`,
  `with_phi_suspect_threshold`, `membership.rs:61`, `:66`), consuming `mut self`.
- **`record_*` for counter mutation** (6 methods, `membership.rs:249-283`) all funnel
  through one private `update_peer_counters` (`:315-326`) using `saturating_add`.
- **`*_now` for "do the periodic thing immediately"** — `reconcile_now`
  (`runtime.rs:150`), used as the boot fast-path.
- **`*_with_*` for the test/explicit variant of a production default**:
  `bind_with_secret_key` (`endpoint.rs:26`), `start_with_intervals`
  (`runtime.rs:102`), each documented as "production should use the plain one"
  (`endpoint.rs:23-25`, `runtime.rs:84-87`).
- **Time as `u64` millis on the wire, `SystemTime`/`Duration` in memory**, with
  explicit converters `now_millis` (`gossip.rs:223-229`) and
  `system_time_from_millis` (`gossip.rs:231-233`). `now_millis` clamps to
  `u64::MAX` and uses `unwrap_or_default()` on a pre-epoch clock (`gossip.rs:225-228`)
  — no panic path.
- **Addresses as `String` at the model boundary** (`GossipRecord.endpoint_addrs`,
  `ReadyRecord.endpoint_addrs`) explicitly "so this layer does not depend on
  transport internals" (`registry.rs:105-106`), parsed lazily at dial time
  (`runtime.rs:328-336`). Trades type safety for layer independence, and moves
  parse failures from boot to runtime.
- **Boxed futures instead of `async_trait`** (`BoxFuture`, `lib.rs:141`), public
  precisely so out-of-crate implementors can name it (`lib.rs:138-140`).
- **Concrete type over trait where lanes must share an implementation**:
  `MeshStream` is a struct, not a trait, "so lanes share one framing
  implementation" (`lib.rs:183`).

#### 6. Logging conventions

28 `tracing` sites. Consistent shape: structured fields first, then a
`"mesh: <event>"` message.

- Prefix is `mesh:` in `runtime.rs` (`:246`, `:265`, `:272`, `:279`, `:290`, `:317`,
  `:331`, `:342`, `:360`, `:380`, `:404`, `:409`, `:548`, `:602`) and
  `"mesh membership …"` / `"mesh ready registry …"` in the membership/registry lanes
  (`membership.rs:96`, `:105`; `registry.rs:236`, `:241`, `:246`).
- `peer = %runtime_id` (full 64-hex via `Display`) is the standard correlation field.
- Level discipline: `info!` for lifecycle (peer connected/disconnected `:255`, `:280`;
  endpoint closed `:272`), `warn!` for rejected/malformed input and dial failures,
  `debug!` for expected loop termination (`:360`, `:380`, `:548`).
- Rejections log enough to diagnose without leaking secrets — e.g. the
  foreign-relay warn logs `record_relay_pubkey` and `anchored`
  (`membership.rs:96-101`), never a private key.
- Gap: no rate limiting on `warn!`. `"rejected inbound connection from unattested
  runtime id"` (`runtime.rs:265-269`) and the dial-failure warn
  (`runtime.rs:342-346`, every 5 s per dead peer) are both attacker- or
  drift-triggerable log floods.

#### 7. Test conventions

32 tests, `#[cfg(test)] mod tests` at the bottom of 6 of 9 files; **no `tests/`
directory**, no integration-test target. All 32 pass; **0 `#[ignore]`d** —
notable, since this repo uses `#[ignore]` heavily for infra-dependent tests
(`justfile:277-285` explains the buzz-db pattern). Here the Redis-dependent paths
simply have no tests at all rather than ignored ones.

- **Deterministic identities in tests** via `SecretKey::from_bytes(&[n; 32])`
  (`endpoint.rs:157-162`, `runtime.rs:627-631`) and `RuntimeId([byte; 32])` helpers
  named `rid(byte)` — the same helper name is repeated in `membership.rs:394`,
  `registry.rs:319`, `gossip.rs:240`, `runtime.rs:...`, and even in the consumer
  (`mesh_boot.rs:582`). Convention by copy-paste, not by a shared test-utils module
  (this crate exposes no `test-utils` feature, unlike `buzz-core`).
- **Real transport in unit tests.** Five `endpoint.rs` tests and five `runtime.rs`
  tests stand up genuine loopback iroh endpoints and connect them
  (`endpoint_pair`/`connected_pair`, `endpoint.rs:155-176`; `runtime`/`seed`/
  `connected_pair`, `runtime.rs:626-670`). No mocking of QUIC. Whole suite still runs
  in 0.25 s.
- **Poll-with-timeout instead of sleep-and-hope**: every async assertion is
  `timeout(Duration::from_secs(5), async { loop { … sleep(20ms) } })`
  (`runtime.rs:655-668`, `:724-740`, `:756-768`, `:788-800`). Consistent and correct.
- **Explicit teardown** — every multi-runtime test calls `a.shutdown(); b.shutdown();`
  (`runtime.rs:742-743` etc.), which is the only place `shutdown()` is exercised.
- **Negative tests are first-class**: unknown wire version (`wire.rs:246`),
  oversize datagram (`endpoint.rs:239`), tampered attestation (`registry.rs:348`,
  `:358`; `membership.rs:437`), foreign relay key (`membership.rs:451`), unanchored
  fail-closed (`membership.rs:465`), stale gossip (`membership.rs:474`,
  `gossip.rs:253`), unconnected peer (`runtime.rs:823`).
- **Tests as executable specs for physical budgets**: the Opus-sized loss/order gate
  (`endpoint.rs:256-291`) and the 64-byte header budget (`wire.rs:266-284`).
- `tokio = { features = ["test-util"] }` is declared (`Cargo.toml:29`) but **no test
  uses paused time** (`tokio::time::pause` appears nowhere) — the phi tests hand
  `SystemTime` values in directly instead (`gossip.rs:268-278`), which is why they are
  fast and deterministic.
- Test-only doc comments explain *why* a setup is shaped as it is, e.g.
  `runtime.rs:647-650` explaining that with no registry the acceptor's admission gate
  requires pre-seeded membership "(production gets this from the attested ready
  registry)".

#### 8. Documentation conventions

- Every public item has a doc comment (AGENTS.md rule) — spot-checked across all 9
  files; the only bare items are the `IrohSendHalf`/`IrohRecvHalf` private structs
  (`peer.rs:132-133`).
- Doc comments record **rationale and rejected alternatives**, often naming the
  reviewer: "Wren's contract-review blocker" (`wire.rs:57`), "Wren's chaos-gate
  ruling" (`lib.rs:103`), "Dawn huddle peer_index" (`endpoint.rs:259`). Valuable
  archaeology; also means the source is the only record — none of it is in
  `ARCHITECTURE.md`, which does not mention this crate at all (verified: zero
  `mesh`/`iroh`/`quic` hits in all 827 lines).
- **Six doc comments are now stale or wrong** — see `-business-rules.md` §K for the
  full list (notably `lib.rs:55-56` claiming `BUZZ_MESH` defaults on, `lib.rs:186-188`
  calling the real iroh stream halves "placeholder", and `lib.rs:102-109`
  specifying a metric that does not exist).

#### 9. Deviations from repo-wide conventions

| AGENTS.md / repo convention | This crate |
|---|---|
| No `unsafe` | ✅ zero |
| No new `unwrap()`/`expect()` in production paths | ⚠️ 22 `expect()` (all lock-poison) |
| New public API must have doc comments | ✅ |
| Prefer Nostr events over new HTTP endpoints | n/a (crate has no HTTP surface; the consumer adds `GET /_mesh` and `POST /_mesh/demo/echo`) |
| Event kinds in `buzz-core/src/kind.rs` | n/a — this crate has no `buzz-core` dep and defines its own postcard wire, entirely outside the Nostr event model |
| Channel scoping via `h` tags | n/a — no tenant identifier on the mesh wire at all (`-integrations.md` §4) |
| Crate listed in AGENTS.md repo structure | ❌ absent |
| Crate documented in ARCHITECTURE.md §6 Crate Reference | ❌ absent |
| Unit tests run by `just test-unit` | ❌ absent from the list (`justfile:275-295`) |
| `#[ignore]` for infra-dependent tests | ❌ Redis paths untested rather than ignored |


## Module: buzz-conformance (`crates/buzz-conformance`)
### Aspect: Conventions

### Naming

**Spec-mirroring is the dominant convention.** Every `TraceAction` variant is named after its
TLA+ action, character-for-character:

| Rust variant | TLA+ action | Spec line |
|---|---|---|
| `WriteInsert` (`src/lib.rs:181`) | `WriteInsert(w)` | `docs/spec/MultiTenantRelay.tla:514` |
| `WriteInsertGlobal` (`:195`) | `WriteInsertGlobal(w)` | `:559` |
| `WriteDuplicate` (`:204`) | `WriteDuplicate(w)` | `:606` |
| `SanitizedError` (`:214`) | `SanitizedError(w)` | `:778` |
| `AuthCheck` (`:221`) | `AuthCheck(w)` | `:794` |
| `ReadMessageRows` (`:231`) | `ReadMessageRows(w)` | `:643` |
| `ReadByIdRows` (`:241`) | `ReadByIdRows(w)` | `:681` |
| `ReadHostFeedRows` (`:250`) | `ReadHostFeedRows(w)` | `:703` |

`ImplBug` (`:256`) is the sole variant with no spec counterpart, and the doc comment says so
explicitly (`:193-194`).

**`Inv_*` names appear only in prose, never as Rust identifiers.** The crate references
`Inv_NonInterference` (`src/transitions.rs:53`, `:296`, `src/lib.rs:238`),
`Inv_ReadConfinement` (`src/transitions.rs:54`), `Inv_SanitizedErrors` (`src/lib.rs:125`),
`Inv_AdmissionFence` (`src/transitions.rs:223`) — all inside `///` or `//` comments. No
function or type is named after an invariant; the mapping from invariant to enforcing predicate
lives in comments, not in the type system.

**Mutation IDs (`M1`…`M8`) are a second comment-only vocabulary.** Used at `src/lib.rs:127`
("M6 mutation"), `:190` ("M2/M8 target"), `:238` ("M1/M4/M7"), `src/transitions.rs:218-221`,
`:239-240`, `crates/buzz-relay/src/conformance/mod.rs:18-19`,
`crates/buzz-relay/src/handlers/ingest.rs:1779`. No file in the repo defines what M1–M8 are —
grep for `M1 ` / `M4` / `M7` outside these comment sites finds no legend. The identifiers are
inherited from an external mutation-testing plan ("skill-runtime-formal-compliance", cited at
`Cargo.toml:7`, `src/lib.rs:6`, `src/checker.rs:9`, `tests/proptest_checker.rs:5-6`) that is
not in the repo.

**Label newtypes use a `*Label` suffix** — `CommunityLabel`, `HostLabel`, `ChannelLabel`,
`ActorLabel` (`src/lib.rs:66`, `:100`, `:106`, `:112`) — except `OpaqueId` (`:93`), which
breaks the pattern despite being the same kind of thing.

**Tracer impls use a `*Tracer` suffix** and are named for their sink: `NoopTracer`
(`src/lib.rs:323` and again `crates/buzz-relay/src/conformance/tracers.rs:16`), `JsonlTracer`
(`tracers.rs:30`), `CountingTracer` (`conformance/mod.rs:356`, private), `VecTracer` — declared
twice, independently, as a test-local sink (`conformance/mod.rs:447-456` and
`handlers/ingest.rs:2519-2528`) rather than shared through a test-support module.

**Emitter helpers are `record_*` or `emit*`.** `record_req_authcheck` (`mod.rs:148`),
`record_read_message_rows` (`:265`), `record_read_by_id_rows` (`:300`) all end in the action
they emit; `emit` (`:127`) and `emit_product_feedback_success`
(`handlers/ingest.rs:133`) are the generic and one-off forms. `step` (`mod.rs:121`) is the
odd one out — a bare noun with no caller.

**Trailing-underscore placeholder.** `Verdict_` (`src/transitions.rs:53-56`) uses a trailing
underscore to avoid colliding with the schema's `Verdict`. It has zero uses; the underscore
suffix also keeps it out of `non_camel_case_types` lint range.

---

### Error handling

- **No panics in library code.** `#![deny(unsafe_code)]` (`src/lib.rs:38`) and no `unwrap()` /
  `expect()` anywhere in `src/` — verified by reading all three source files. The one
  `expect` in the relay-side helper is a documented invariant restatement:
  `row.expect("project_row_community returns None only for Some(ch)")`
  (`conformance/mod.rs:248`).
- **`thiserror` for the single error type.** `TransitionError`
  (`src/transitions.rs:60-102`) derives `thiserror::Error` with `#[error(...)]` format strings
  carrying the step index (`:63`, `:75`, `:85`, `:95`).
- **Human-readable detail, machine-readable variant.** Every variant carries
  `detail: String` built with `format!` (`:146-151`, `:155-160`, `:164-169`, `:236-241`,
  `:278`, `:304-309`). The convention documented at `:58-59` — "the string payload is
  human-readable; mechanical consumers should match on the variant" — means offending values
  are only recoverable by parsing prose.
- **Fail-fast, first-error-wins.** `check_step` returns on the first violation and
  `check_trace` propagates with `?` (`src/checker.rs:109`). The test suites are written around
  this constraint; the discipline is spelled out at `tests/proptest_checker.rs:27-33`.
- **Fail-closed defaults.** Empty trace → `CoverageBreach` (`src/checker.rs:75-82`);
  missing projection lookup → `ImplBug`, never a substituted label
  (`conformance/mod.rs:246-253`, rationale `:203-208`).
- **Observability code never breaks the request.** `JsonlTracer::record` recovers from a
  poisoned mutex via `into_inner()` (`tracers.rs:59-63`) and discards write errors
  (`:66-71`); `req.rs` logs a `warn!` and continues with an empty lookup map on DB failure
  (`:347-353`, `:663-669`). The `Tracer` trait returns `()` (`src/lib.rs:317`), so there is no
  error channel to propagate even if a caller wanted one.

---

### Comment style

Unusually heavy — roughly half the crate is doc prose. Three recurring shapes:

1. **"Why not the obvious thing"** blocks. `Cargo.toml:7-24` (independence rule),
   `src/lib.rs:47-63` (why not `buzz_core::CommunityId`), `src/lib.rs:236-239` (why `Vec` not
   `Set`), `conformance/mod.rs:135-145` (why `claimed_community: None` on REQ).
2. **Spec-line citations** inline with each match arm (`src/transitions.rs:172-186`,
   `:188-191`, `:193-198`, `:200-205`, `:211-227`, `:251-257`, `:266-268`, `:272-276`).
3. **Named-reviewer / thread references.** `conformance/mod.rs:37-38` ("held back as additive
   patch for Eva to apply onto Max's req.rs writes — see thread `c882c9b1…`"),
   `tests/replay_fixtures.rs:19-20` ("Eva's review (thread `06aaf3f7…`)"),
   `tests/replay_fixtures.rs:145-152`, `conformance/mod.rs:170-172` ("the (B)
   projection-strategy guard-rail Eva specified"). These leave the code coupled to
   conversations that are not in the repo, and several are now stale (the req.rs patch has
   landed — `handlers/req.rs:334-361`, `:649-677` — while the comment still says "held back",
   as does `TRACE_SCHEMA.md:137`).

---

### Test organization

| Lane | Location | Convention |
|---|---|---|
| Unit | `src/checker.rs:134-337` (`#[cfg(test)] mod tests`) | one passing case + one bite case per failure mode; tiny helpers `cid`/`ch`/`state`/`step` (`:144-162`) |
| Property | `tests/proptest_checker.rs` | one `proptest!` block (`:191-431`), all 7 cases inside; generators prefixed `arb_*` (`:73-93`, `:115-170`, `:184-189`) |
| Fixture | `tests/replay_fixtures.rs` | typed builder → serialize → byte-compare → re-parse → replay (`assert_fixture_matches`, `:206-235`) |
| Emitter | `crates/buzz-relay/src/conformance/mod.rs:431-726` | in-crate `#[cfg(test)] mod tests` with a local `VecTracer` sink (`:447-456`) |

**Test names encode the assertion, not the target.** `*_bites_*` for expected failures
(`cross_community_row_bites_non_interference` `src/checker.rs:210`,
`auth_allow_with_foreign_claim_bites_m2` `:228`, `impl_bug_action_bites_coverage_breach`
`:290`, `state_flip_bites_state_mismatch` `tests/proptest_checker.rs:354`); `*_is_fine` /
`*_is_ok` / `*_passes` / `*_is_accepted` for expected successes (`:247`, `:172`,
`tests/proptest_checker.rs:199`, `:304`).

**Property tests carry `P<n>` doc-comment IDs** — P1 (`tests/proptest_checker.rs:207`),
P2 (`:195`), P3a (`:269`), P3b (`:299`), P4 (`:325`), P5 (`:351`), P6 (`:401`) — matching the
"invariant properties, NOT a parallel oracle" design note at `:9-25`.

**Fixture regeneration is env-gated, not flag-gated:** `BUZZ_CONFORMANCE_UPDATE=1`
(`tests/replay_fixtures.rs:210`), so a schema change forces a deliberate re-commit rather than
silently rewriting the golden files.

**Deterministic fixture constants.** `community_a`/`community_b`/`channel_in_a`/`channel_in_b`
are hand-picked `Uuid::from_u128` values with mnemonic hex prefixes
(`tests/replay_fixtures.rs:48-62`: `0xAAAA…`, `0xBBBB…`, `0xCAFE…`, `0xDEAD…`); the property
lane uses prefix-tagged pools instead (`0x0c00…` for communities, `0x0ca0…` for channels,
`tests/proptest_checker.rs:53-63`). The rationale — reproducible serialized JSONL — is at
`tests/replay_fixtures.rs:42-46`.

---

### Serde conventions

- Every newtype is `#[serde(transparent)]` (`src/lib.rs:65`, `:92`, `:99`, `:105`, `:111`), so
  the wire form is a bare scalar.
- Both enums use `#[serde(rename_all = "snake_case")]` (`:116`, `:131`).
- `TraceAction` uses internal tagging: `#[serde(tag = "type", rename_all = "snake_case")]`
  (`:178`), so each action object carries a `"type"` discriminant matching
  `TraceAction::kind()` (`:266-277`).
- Field names are snake_case Rust identifiers with no renames, so `schema_version` /
  `state_after` appear verbatim on the wire — which is where `TRACE_SCHEMA.md:37-46` diverges
  from reality (it documents `schema` / `state`).
- JSONL, one `TraceStep` per line, no envelope: writer `tracers.rs:66-71`, test-side
  serializer `tests/replay_fixtures.rs:179-187`, parser `:191-198` (skips blank lines, panics
  with a 1-based line number).

---

### Dead-code tolerance

Four public items have zero callers anywhere in `crates/` and produce no warning because they
are `pub` in a library:

| Item | Line |
|---|---|
| `Verdict_` | `src/transitions.rs:53-56` |
| `action_channel` | `src/transitions.rs:318-330` |
| `TraceAction::is_critical` | `src/lib.rs:283-285` |
| `Scenario::require` | `src/checker.rs:54-57` |

Two more on the relay side: `conformance::step` (`mod.rs:121-123`) and the crate's own
`NoopTracer` (`src/lib.rs:323-327`), shadowed by the relay's copy (`tracers.rs:16-20`). The
convention here is evidently "keep the reserved surface"; none of the six carries a
`#[allow(dead_code)]` or a TODO.

