## Module: buzz-core (`crates/buzz-core`)

### Aspect: Integrations

This crate has **no internal Buzz dependencies** — `crates/buzz-core/Cargo.toml:13-27` lists only third-party crates. It integrates with no network service, database, cache, or file system: there is no I/O code path anywhere in `src/`.

---

### 1. External crate dependencies

All versions come from `[workspace.dependencies]` in the repo-root `Cargo.toml`; `buzz-core` declares them as `{ workspace = true }` except `percent-encoding`, which is version-pinned locally.

| Crate | Declared at | Workspace version | Why it is used (code evidence) |
|-------|-------------|-------------------|-------------------------------|
| `nostr` | `crates/buzz-core/Cargo.toml:14` | `0.44`, features `["nip44", "nip98"]` (root `Cargo.toml:61`) | The core event/key model. Provides `Event`, `EventId`, `Filter`, `Keys`, `Kind`, `PublicKey`, `SecretKey`, `Tag`, `EventBuilder`, `Timestamp`, `SingleLetterTag`, `Alphabet` (re-exported at `src/lib.rs:42`); `verify_id`/`verify_signature`/`EventId::new` for verification (`src/verification.rs:12-25`); `secp256k1::Error` in the error enum (`src/error.rs:19`); `nips::nip44` v2 encrypt/decrypt (`src/observer.rs:73-95`, `src/engram.rs:457` and `:549`, `src/pairing/session.rs:566` and `:600`); `nips::nip44::v2::ConversationKey` for the engram K_c (`src/engram.rs:136-138`); `util::hkdf::{extract, expand}` for all NIP-AB derivations (`src/pairing/crypto.rs:24`, used in `hkdf32` at `:34-43`); `util::generate_shared_key` for ECDH (`src/pairing/session.rs:186`, `:292`); `hashes::Hash` trait import (`src/pairing/crypto.rs:23`) |
| `serde` | `Cargo.toml:15` | `1`, `derive` (root `:64`) | Derives on `PresenceStatus` (`src/presence.rs:9`), `TokenCounts`/`StopReason`/`AgentTurnMetricPayload` (`src/agent_turn_metric.rs:21`, `:51`, `:87`), `Listing` (`src/engram.rs:597`), all pairing message types (`src/pairing/types.rs:19`, `:61`, `:79`); manual `Deserialize` impl for `StopReason` (`src/agent_turn_metric.rs:66-77`); low-level `de::{DeserializeSeed, MapAccess, SeqAccess, Visitor}` for the strict-JSON parser (`src/engram.rs:284`) |
| `serde_json` | `Cargo.toml:16` | `1` (root `:69`) | Payload (de)serialization in observer/pairing (`src/observer.rs:63`, `:106`, `src/pairing/session.rs:565`, `:618`); `Value` + `Deserializer::from_slice` for duplicate-key-rejecting engram body parsing (`src/engram.rs:283-380`); JSON errors wrapped into `ObserverPayloadError::Json` (`src/observer.rs:35`) and `PairingError::Json` (`src/pairing/mod.rs:71`) |
| `thiserror` | `Cargo.toml:17` | `2` (root `:85`) | `#[derive(thiserror::Error)]` on `VerificationError` (`src/error.rs:2`), `EngramError` (`src/engram.rs:37`), `ObserverPayloadError` (`src/observer.rs:28`), `PairingError` (`src/pairing/mod.rs:34`), `NormalizeRelayUrlError` (`src/relay.rs:7`) |
| `uuid` | `Cargo.toml:18` | `1`, features `["v4", "serde"]` (root `:89`) | `StoredEvent.channel_id: Option<Uuid>` (`src/event.rs:17`), `CommunityId(Uuid)` (`src/tenant.rs:37`), `uuid::Uuid::new_v4()` in filter tests (`src/filter.rs:192`, `:213`, `:224`) |
| `chrono` | `Cargo.toml:19` | `0.4`, `serde` (root `:90`) | `DateTime<Utc>` receive timestamps and `Utc::now()` (`src/event.rs:6`, `:25`), test helper (`src/lib.rs:51`) |
| `hex` | `Cargo.toml:20` | `0.4` (root `:97`) | d-tag hex encoding (`src/engram.rs:155`), session-secret hex in the QR URI (`src/pairing/qr.rs:81`, `:163`), session id / transcript hash hex on the wire (`src/pairing/session.rs:179`, `:216`, `:318`, `:357`) |
| `hmac` | `Cargo.toml:21` | `0.13` (root `:98`) | `Hmac`, `Mac`, `digest::KeyInit` for the engram d-tag HMAC-SHA256 (`src/engram.rs:10-11`, `:147-154`) |
| `sha2` | `Cargo.toml:22` | `0.11` (root `:96`) | `Sha256` as the HMAC hash (`src/engram.rs:15`, `:147`) |
| `rand` | `Cargo.toml:23` | `0.10` (root `:101`) | `rand::fill` to generate the 32-byte pairing session secret (`src/pairing/session.rs:113-115`); `rand::random::<u64>()` for the 0–30 s `created_at` jitter (`src/pairing/session.rs:578`) |
| `subtle` | `Cargo.toml:24` | `2.6` (root `:102`) | `ConstantTimeEq` behind `ct_eq` for 32-byte secret comparisons (`src/pairing/crypto.rs:126-129`) |
| `zeroize` | `Cargo.toml:25` | `1.8` (root `:103`) | `Zeroize` on plaintext buffers (`src/observer.rs:66`, `:79`, `:101`, `:109`), `QrPayload::drop` (`src/pairing/qr.rs:52-56`), `PairingSession::drop` (`src/pairing/session.rs:731-739`), `Zeroizing<String>` payload type in the pairing API (`src/pairing/session.rs:227-247`, `:388-408`) |
| `percent-encoding` | `Cargo.toml:26` | **`"2.3"` pinned locally, not via workspace** | `utf8_percent_encode` / `percent_decode_str` with the `NON_ALPHANUMERIC` set for relay URLs inside the QR URI (`src/pairing/qr.rs:31`, `:220-236`) |
| `url` | `Cargo.toml:27` | `2` (root `:114`) | `Url` + `Host` parsing in `normalize_relay_url` (`src/relay.rs:4`, `:38-64`), `relay_url_authority` (`src/tenant.rs:156-172`), and relay-URL validation during QR decode (`src/pairing/qr.rs:185-204`) |

Transitive crypto note: no direct `secp256k1` dependency is declared; the crate reaches Schnorr/secp256k1 exclusively through `nostr` (`nostr::secp256k1::Error` at `src/error.rs:19`, `event.verify_signature()` at `src/verification.rs:27`).

---

### 2. The zero-I/O prohibition — how it is (and is not) enforced

| Mechanism | Present? | Evidence |
|-----------|----------|----------|
| Comment in the crate manifest | **yes** | `crates/buzz-core/Cargo.toml:28`: `# NO tokio, NO sqlx, NO redis, NO axum — zero I/O dependencies` |
| Absence of those crates in `[dependencies]` | **yes** | `Cargo.toml:13-27` lists 14 dependencies; none is `tokio`, `sqlx`, `redis`, or `axum` |
| Absence of `dev-dependencies` that would reintroduce a runtime | **yes** | the manifest has no `[dev-dependencies]` section at all (`Cargo.toml:1-29`) |
| Crate-level lint attributes | partial | `#![deny(unsafe_code)]`, `#![warn(missing_docs)]` (`src/lib.rs:1-2`) — these do not restrict dependencies |
| `cargo-deny` ban rules | **no** | root `deny.toml:90-92` `[bans]` sets only `multiple-versions = "warn"` and `wildcards = "allow"`; no per-crate dependency bans |
| `[workspace.lints]` / `[lints]` | **no** | neither section exists in the root `Cargo.toml` nor in `crates/buzz-core/Cargo.toml` |
| Automated CI check specific to buzz-core dependencies | **none found** | the only `buzz-core` references in tooling are test invocations: `Justfile:278` (`cargo nextest run -p buzz-core -p buzz-auth --lib`), `scripts/run-tests.sh:81-82` (`cargo test -p buzz-core --lib`) |

Conclusion as coded: the no-I/O rule is a **convention documented in the manifest comment and upheld by review**, not a mechanically enforced constraint. Nothing in the repo fails a build if `tokio` is added to `crates/buzz-core/Cargo.toml`.

The same "documented fence, not compiler fence" pattern is stated explicitly for tenant construction (`src/tenant.rs:23-30`), which names a "migration-lint harness" outside this crate as the enforcing mechanism.

---

### 3. Cargo features

| Feature | Declared at | Gates |
|---------|-------------|-------|
| `test-utils` | `crates/buzz-core/Cargo.toml:10-11` | `pub mod test_helpers` via `#[cfg(any(test, feature = "test-utils"))]` (`src/lib.rs:47`) |

No default features are declared. `test-utils` is enabled by one consumer: `crates/buzz-relay/Cargo.toml:89`, inside that crate's `[dev-dependencies]` section (`crates/buzz-relay/Cargo.toml:86`).

---

### 4. Consumers (inbound integration)

`buzz-core` is depended on by 15 workspace crates plus the Tauri desktop backend — all as `{ workspace = true }` path dependencies:

| Consumer | Manifest line |
|---|---|
| `buzz-acp` | `crates/buzz-acp/Cargo.toml:20` |
| `buzz-admin` | `crates/buzz-admin/Cargo.toml:16` |
| `buzz-audit` | `crates/buzz-audit/Cargo.toml:11` |
| `buzz-auth` | `crates/buzz-auth/Cargo.toml:15` |
| `buzz-cli` | `crates/buzz-cli/Cargo.toml:46` |
| `buzz-db` | `crates/buzz-db/Cargo.toml:11` |
| `buzz-dev-mcp` | `crates/buzz-dev-mcp/Cargo.toml:41` |
| `buzz-media` | `crates/buzz-media/Cargo.toml:11` |
| `buzz-pairing-cli` | `crates/buzz-pairing-cli/Cargo.toml:15` |
| `buzz-pubsub` | `crates/buzz-pubsub/Cargo.toml:11` |
| `buzz-relay` | `crates/buzz-relay/Cargo.toml:19` (+ `:89` under `[dev-dependencies]` with `test-utils`) |
| `buzz-sdk` | `crates/buzz-sdk/Cargo.toml:11` |
| `buzz-search` | `crates/buzz-search/Cargo.toml:11` |
| `buzz-test-client` | `crates/buzz-test-client/Cargo.toml:12` |
| `buzz-workflow` | `crates/buzz-workflow/Cargo.toml:11` |
| desktop Tauri backend | `desktop/src-tauri/Cargo.toml:88`, aliased as `buzz_core_pkg = { package = "buzz-core", path = "../../crates/buzz-core" }` |

`crates/buzz-conformance/Cargo.toml:14` mentions buzz-core in a comment about its "no Serde, no `From<Uuid>`" fence on `CommunityId` rather than as a dependency line at that position.

---

### 5. Protocol/spec integrations referenced by the code

These are specification integrations (behaviour contracts), all named in doc comments:

| Spec | Where the crate implements or references it |
|---|---|
| NIP-01 (event id/sig, filters, kind semantics) | `src/verification.rs:11-32`, `src/filter.rs:1-3`, `:62`, `src/kind.rs:3-5` |
| NIP-09, NIP-17, NIP-23, NIP-25, NIP-29, NIP-33, NIP-34, NIP-38, NIP-42, NIP-43, NIP-50, NIP-51, NIP-56, NIP-65, NIP-78, NIP-94, NIP-98, BUD-01 | kind registry doc comments, `src/kind.rs:8-487` |
| NIP-44 v2 (encryption + conversation key) | `src/observer.rs:66-95`, `src/engram.rs:457`, `src/pairing/session.rs:566` |
| NIP-AB (device pairing) | `src/pairing/**`; local spec files `src/pairing/NIP-AB.md` and a Tamarin model `src/pairing/NIP-AB.spthy` live beside the code |
| NIP-AE (agent engrams) | `src/engram.rs:1-6` (points to `docs/nips/NIP-AE.md`) |
| NIP-AM (agent turn metric) | `src/agent_turn_metric.rs:1-8` (points to `docs/nips/NIP-AM.md`) |
| NIP-AP (persona/team/managed agent), NIP-ER (event reminder), NIP-PL (push lease), NIP-DV (DM visibility), NIP-IA (identity archival), NIP-RS (read state) | kind doc comments `src/kind.rs:71-313` |
| RFC 1918 / 2544 / 3849 / 6598 (address ranges), RFC 3986 (host case), RFC 8259 (JSON strings) | `src/network.rs:12-24`, `src/tenant.rs:110-116`, `src/engram.rs:261-262` |
