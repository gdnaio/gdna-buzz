## Module: buzz-core (`crates/buzz-core`)

### Aspect: Features

`buzz-core` is a foundation library — it enables capabilities in other crates rather than exposing user-facing surfaces itself. Completeness below is judged only against what this crate's own code does; a capability marked "partial" means part of the contract described in its own doc comments is not implemented here.

---

### Capability inventory

| # | Capability | What this crate provides | Completeness | Evidence |
|---|-----------|--------------------------|--------------|----------|
| F-1 | Nostr event verification | ID-hash recomputation + Schnorr signature check, with a typed error carrying computed vs claimed ID | **full** | `src/verification.rs:11-32`, `src/error.rs:2-20` |
| F-2 | Relay event metadata wrapper | `StoredEvent` binding an event to receive time, channel scope, and a private verified flag | **full** | `src/event.rs:11-48` |
| F-3 | Event-kind registry | 130 kind constants, 4 range bounds, 4 kind-set slices, `ALL_KINDS` (127), 9 classification predicates, 2 kind extractors, 25 compile-time range assertions | **full** | `src/kind.rs:9-744` |
| F-4 | NIP-01 subscription filter matching | kinds/authors/since/until/ids(prefix)/generic-tag matching with OR across filters, AND within a filter, plus the `#h` channel fallback | **full** for the fields it implements; **partial** vs NIP-01 overall — `limit` and NIP-50 `search` are not handled here (no reference to either in `filter.rs`) | `src/filter.rs:10-104` |
| F-5 | Per-viewer result-level read authorization | `reader_authorized_for_event` gate for DM-visibility and agent-turn-metric kinds | **full** (as a predicate; enforcement at delivery sites lives in `buzz-relay`, per doc `filter.rs:19-22`) | `src/filter.rs:23-33` |
| F-6 | SSRF address classification | `is_private_ip` covering 7 IPv4 and 6 IPv6 categories plus IPv4-mapped recursion | **full** for the enumerated ranges; see security doc for ranges *not* covered | `src/network.rs:25-53` |
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
