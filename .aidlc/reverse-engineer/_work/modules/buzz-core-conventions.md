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
| Event kind constants | `KIND_<DOMAIN>[_<DETAIL>]: u32`, grouped by protocol with section comments | `KIND_STREAM_MESSAGE_EDIT` (`kind.rs:347`), `KIND_NIP29_PUT_USER` (`kind.rs:199`) |
| Exception to that pattern | relay-admin commands use a `RELAY_ADMIN_` prefix instead of `KIND_` | `RELAY_ADMIN_ADD_MEMBER` (`kind.rs:253`) … `RELAY_ADMIN_SET_WORKSPACE_PROFILE` (`kind.rs:259`) |
| Kind sets | plural `*_KINDS: &[u32]` | `ALL_KINDS` (`kind.rs:490`), `P_GATED_KINDS` (`kind.rs:146`), `AUTHOR_ONLY_KINDS` (`kind.rs:120`), `RESULT_GATED_KINDS` (`kind.rs:129`) |
| Range bounds | `<NAME>_KIND_MIN` / `_MAX` | `EPHEMERAL_KIND_MIN/MAX` (`kind.rs:321-323`) |
| Predicates | `is_*(kind: u32) -> bool`, `const fn` wherever possible | `is_ephemeral`, `is_replaceable`, `is_relay_only_kind` (`kind.rs:621-693`) |
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
- Spec citations are inline and section-level: `NIP-AB §Duplicate Event Handling` (`session.rs:530-533`), `NIP-AM §Numeric validity` (`agent_turn_metric.rs:138`), `NIP-AE *Head selection* rule (3)` (`engram.rs:222-224`), `RFC 6598` (`network.rs:18`).
- Cross-references use rustdoc links (`[`StoredEvent`]`, `[`crate::engram`]`) — e.g. `lib.rs:5-6`, `kind.rs:97-98`.
- Two doctests exist (both in pairing): `format_sas` (`pairing/crypto.rs:110-114`) and `encode_qr` (`pairing/qr.rs:66-77`).

---

### 5. Testing patterns

All tests are inline `#[cfg(test)] mod tests` blocks — there is **no `tests/` directory** and no proptest/quickcheck usage anywhere in the crate (searched: no `proptest`, no `quickcheck` in `src/` or `Cargo.toml`).

Static count of `#[test]` functions per file (grep of `#[test]` attributes):

| File | `#[test]` fns | Test module starts at |
|---|---|---|
| `src/engram.rs` | 34 | `:607` |
| `src/git_perms.rs` | 34 | `:601` |
| `src/network.rs` | 29 | `:56` |
| `src/pairing/qr.rs` | 27 | `:245` |
| `src/pairing/session.rs` | 18 | `:756` (a separate `#[cfg(test)] impl` block sits at `:530`) |
| `src/pairing/crypto.rs` | 14 | `:131` |
| `src/agent_turn_metric.rs` | 14 | `:194` |
| `src/pairing/types.rs` | 10 | `:98` |
| `src/tenant.rs` | 10 | `:175` |
| `src/filter.rs` | 6 | `:106` |
| `src/kind.rs` | 4 | `:747` |
| `src/presence.rs` | 4 | `:37` |
| `src/relay.rs` | 3 | `:80` |
| `src/observer.rs` | 2 | `:113` |
| `src/verification.rs` | 2 | `:34` |
| `src/channel.rs` | 1 | `:181` |
| `src/event.rs` | 1 | `:53` |
| `src/error.rs`, `src/lib.rs`, `src/pairing/mod.rs` | 0 | — |
| **Total** | **213** | |

(Counts are static — obtained by reading the source. The crate was not compiled during this analysis: `cargo` is not on the PATH in this environment without the repo's Hermit activation, so no test-run count is claimed.)

Recurring test conventions:

| Pattern | Example |
|---|---|
| Shared fixture fns at the top of the test module | `make_event()` (`event.rs:57-63`), `stored_with_tag()` (`filter.rs:113-121`), `sample_payload()` (`agent_turn_metric.rs:199-228`), `make_payload_with_turn_cost()` (`:351`), `make_payload_with_cumulative_cost()` (`:374`), `make_payload()` (`qr.rs:250-258`), `keys_from_hex()` (`engram.rs:611-613`) |
| Pinned spec test vectors as consts | `engram.rs:617-627` (SECKEY/PUBKEY/K_C/D_* vectors), `pairing/crypto.rs:136-155` (session secret + ephemeral key bytes) |
| Single test asserting a whole vector suite | `all_test_vectors` (`pairing/crypto.rs:272-323`), `d_tags_match_spec` (`engram.rs:646-654`) |
| Exhaustive-range invariant tests | `replaceable_and_parameterized_are_disjoint` loops `0..=65535` (`kind.rs:776-784`, message at `:780`) |
| Boundary-pair tests (just-inside / just-outside) | CGNAT and benchmarking range tests (`network.rs:150-196`), `parameterized_replaceable_range` (`kind.rs:766-774`) |
| Round-trip serde tests per wire type | `pairing/types.rs:102-241`, `presence.rs:41-53`, `agent_turn_metric.rs:230-251` |
| Regression tests labelled as such with the scenario in a comment | `filter.rs:216-236` (explicit h-tag authority), `git_perms.rs:926-941` (guest bypass), `agent_turn_metric.rs:476-507` (validation bypass via lower-level encrypt) |
| Negative/anti-property tests | `reject_*` naming across `qr.rs` (12 `reject_*` tests) and `session.rs` (`reject_out_of_order_operations`, `reject_invalid_session_id`, `reject_event_from_wrong_pubkey`, …) |
| Full-protocol happy-path test driving both peers in one process | `happy_path_full_protocol` (`session.rs:762`) |
| Test-only inherent methods behind `#[cfg(test)]` instead of widening the API | `session.rs:530-544` (`has_processed`, `set_timeout`) |
| Compile-time tests | `const _: () = assert!(...)` block, `kind.rs:707-744` (25 assertions) |
| `assert!` messages explaining the invariant, not just the failure | `filter.rs:234`, `kind.rs:780`, `session.rs:782` |

---

### 6. Other code conventions

- `const fn` is used wherever the computation allows it (`kind.rs:240`, `:621-693`; `tenant.rs:45`, `:50`, `:87`; `engram.rs:114`).
- `#[must_use]` on the two pure string transformers (`tenant.rs:120` for `normalize_host`, `:155` for `relay_url_authority`).
- All kind integers are `u32` with a stated rationale (NIP-01 unsigned integer, u32 covers the range) at `kind.rs:3-5`; conversion to `u16`/`i32` is centralized in `event_kind_u32`/`event_kind_i32` (`kind.rs:696-704`) rather than scattered casts.
- Secret material is handled with a consistent trio: `Zeroizing<String>` in signatures, explicit `.zeroize()` after use, and `Drop` impls (`pairing/session.rs:227`, `:573`, `:731-739`; `pairing/qr.rs:56-60`; `observer.rs:66-109`).
- Hand-rolled serialization is used only where byte-exactness is a spec requirement, with the reason stated (`engram.rs:194-197`, `:261-262`).
- Newtype + private field + accessor is the standard way this crate expresses "provenance matters" (`CommunityId`, `TenantContext`, `RefPattern`, `StoredEvent.verified`).
