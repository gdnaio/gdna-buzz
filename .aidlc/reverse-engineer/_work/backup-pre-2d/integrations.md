<!-- Analyzed: 2026-07-25T01:12:08Z | Scope: full project -->
# Integrations

> Status: initialized in Phase 1. External services, protocols, clients, retry/error
> handling, and failure modes are populated per-module during Phase 2 and consolidated in
> Phase 3.

## Summary

Scan-time inventory of external systems: PostgreSQL, Redis, S3/MinIO, AI agent
subprocesses (ACP over stdio), OTLP collector, Prometheus, Web Push services, and optional
iroh mesh peers.

Batch 2a (foundation crates) integrates with **no network service at all**:

| Module | Internal deps | External crates | Network | Filesystem |
|---|---|---|---|---|
| `buzz-core` | **0** | 14 | none | none |
| `buzz-sdk` | `buzz-core` | 6 | **zero endpoints called** | none |
| `buzz-persona` | **0** (leaf) | 4 + 1 dev | none | read-only |
| `buzz-ws-client` | **0** | 8 | WebSocket to one relay | none |

Two findings worth carrying forward:

**The zero-I/O rule on `buzz-core` is convention, not enforcement.** The prohibition is a
comment in the manifest (`crates/buzz-core/Cargo.toml:28`: "NO tokio, NO sqlx, NO redis, NO
axum"). There is no `cargo-deny` ban (`deny.toml:90-92` sets only `multiple-versions` and
`wildcards`), no `[workspace.lints]`, and no CI check. Nothing fails the build if `tokio` is
added to that manifest. The same "documented fence, not compiler fence" pattern is stated
explicitly for the tenant type (`crates/buzz-core/src/tenant.rs:23-30`).

**Two declared dependencies are unused at the code level:**
- `buzz-acp` declares `buzz-persona` (`crates/buzz-acp/Cargo.toml:22`) but contains **zero**
  `buzz_persona` references — so the persona crate's primary stated consumer is not wired.
  The receiving field exists (`crates/buzz-acp/src/config.rs:533-535`
  `persona_env_vars`), but nothing calls `resolve_pack`/`resolve_persona_by_name`.
- `buzz-sdk` declares `serde` (`crates/buzz-sdk/Cargo.toml:14`) with no `use serde` or
  derive in `src/` — needed only transitively via `serde_json`.

Batch 2b maps the actual service integrations. Each service crate touches exactly one
external system, and none calls another service crate — matching the architecture
principle at `ARCHITECTURE.md:97` that subsystems are isolated and only the relay
coordinates them.

| Module | External system | Client | Internal deps |
|---|---|---|---|
| `buzz-db` | PostgreSQL 17 | `sqlx` runtime queries (no offline cache ⇒ no compile-time SQL validation) | `buzz-core` |
| `buzz-search` | PostgreSQL FTS | `sqlx`, fully parameterized (`push_bind` everywhere) | `buzz-core` |
| `buzz-audit` | PostgreSQL | `sqlx` + per-community advisory lock | `buzz-core` |
| `buzz-media` | S3 / MinIO (Blossom) | `rust-s3` | `buzz-core` |
| `buzz-pubsub` | **Redis only** | `deadpool-redis` pool + 3 dedicated `redis::aio::PubSub` sockets | `buzz-core`, `buzz-auth` |
| `buzz-workflow` | Outbound HTTP (`call_webhook`) | HTTP client | `buzz-core`, `buzz-db` |
| `buzz-auth` | none | — | `buzz-core` |

Redis integration detail (`buzz-pubsub`): 8 key families, all community-prefixed except
the operator-global IP rate limit. Four connections minimum per pod — one injected
`deadpool` pool for all commands plus three dedicated stateful pub/sub sockets, because a
pub/sub connection cannot be pooled (`crates/buzz-pubsub/src/lib.rs:19-20`). Reconnect is
an exponential backoff of 1 s → 30 s, implemented three times over
(`crates/buzz-pubsub/src/subscriber.rs:16-19` and two mirrors).

Findings worth carrying forward from 2b:

**Redis is a hard availability dependency for authenticated reads, not just fan-out.** The
rate limiter fails closed — a Redis error becomes `AdmissionError::Unavailable`
(`crates/buzz-relay/src/admission.rs:29-33`) and the relay rejects the request
(`crates/buzz-relay/src/connection.rs:612-621`). Correct for a limiter, but there is no
circuit breaker, no in-process fallback, and no documented operator override, so a Redis
outage degrades to "no reads" rather than "no rate limiting".

**Tenant isolation in Redis is key-prefix naming inside one shared instance** — no logical
db separation, no ACLs. Two subscribers deliberately consume *all* tenants' control
traffic via `buzz:*` patterns (`crates/buzz-pubsub/src/cache_invalidation.rs:27`,
`src/conn_control.rs:30`) and demultiplex by parsing the channel name.

**Cross-pod control messages are unversioned and unauthenticated.** Neither
`CacheInvalidation` nor `ConnControl` carries a version, timestamp, origin-pod id, or
signature, and neither is `#[non_exhaustive]`. A rolling deploy that adds a variant makes
older pods `warn`-and-skip it — for `DisconnectPubkey` that means a ban is not live-enforced
on those pods until restart (`crates/buzz-pubsub/src/conn_control.rs:152-156`).

**One more unused declared dependency**, extending the 2a pattern: `buzz-pubsub` declares
`chrono` (`crates/buzz-pubsub/Cargo.toml:18`) with no `chrono::` path in any source file.
Also, `buzz-admin` and `buzz-conformance` both declare `buzz-pubsub` with no verified call
site.

**An outbound-HTTP gap:** workflow `call_webhook` accepts plain `http://` despite
documentation stating HTTPS, and its condition-evaluation timeout does not cancel the
`spawn_blocking` task it guards.

Agent subprocess, push, and mesh integrations arrive in batches 2c–2d.

### Batch 2c integrations (mesh transport, TLA+ specs, MeshLLM)

- **`iroh` version drift** — the manifest declares `1.0.0-rc.0` while `Cargo.lock:3902-3905`
  resolves 1.0.2.
- **Mesh gossip is unauthenticated** — peer records are accepted without signature or origin
  proof (`crates/buzz-relay-mesh/src/membership.rs:120-153`), and there is no peer eviction.
- **MeshLLM is a distinct, undocumented integration.** The 5 `mesh_*.rs` examples under
  `crates/buzz-relay/examples/` talk to `mesh_llm_sdk` via git dev-dependencies
  (`crates/buzz-relay/Cargo.toml:84-85`) — they are unrelated to `buzz-relay-mesh` despite the
  shared prefix.
- **The TLA+ relationship is documentary, not mechanical.** No build step, test, or CI job
  reads `docs/spec/MultiTenantRelay.tla`; the coupling is doc comments carrying spec line
  numbers, several of which have drifted (`crates/buzz-conformance/src/lib.rs:193` cites
  "line ~720" for `ReadHostFeedRows`, actual `tla:703`; `TRACE_SCHEMA.md:57`, `:69` cite 562
  and 612 against actual 559 and 606).
- **`docs/spec/GitOnObjectStore.tla` is a second, separate spec**, consumed by
  `crates/buzz-relay/src/api/git/cas_publish.rs` — unrelated to `buzz-conformance`.
- **Correction to batch-2b context:** `buzz-admin` does **not** depend on `buzz-conformance`.
  Only the workspace root (`Cargo.toml:125`) and `crates/buzz-relay/Cargo.toml:20` declare it.
- **Neither `ARCHITECTURE.md` nor `AGENTS.md` mentions the formal-methods lane** — grep for
  `buzz-conformance`, `conformance`, `MultiTenantRelay.tla`, `TLA`, and `formal` across
  `AGENTS.md`, `ARCHITECTURE.md`, and `CONTRIBUTING.md` returns no hits.

## External Services

| Service | Direction | Protocol | Client | Purpose | Source |
|---|---|---|---|---|---|
| _pending_ | | | | | |

## Outbound HTTP

_Pending Phase 2 (workflow `call_webhook` with SSRF guard, push gateway delivery, media
storage)._

## Inbound Integrations

_Pending Phase 2 (workflow webhooks at `/hooks/{id}`, git smart HTTP, internal git policy
hook)._

## Error Handling & Resilience

_Pending Phase 2 (Redis reconnect backoff, agent crash recovery, slow-client handling,
bounded queues)._

---

# Phase 2 — Module Findings

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
| NIP-09, NIP-17, NIP-23, NIP-25, NIP-29, NIP-33, NIP-34, NIP-38, NIP-42, NIP-43, NIP-50, NIP-51, NIP-56, NIP-65, NIP-78, NIP-94, NIP-98, BUD-01 | kind registry doc comments, `src/kind.rs:8-563` |
| NIP-44 v2 (encryption + conversation key) | `src/observer.rs:66-95`, `src/engram.rs:457`, `src/pairing/session.rs:566` |
| NIP-AB (device pairing) | `src/pairing/**`; local spec files `src/pairing/NIP-AB.md` and a Tamarin model `src/pairing/NIP-AB.spthy` live beside the code |
| NIP-AE (agent engrams) | `src/engram.rs:1-6` (points to `docs/nips/NIP-AE.md`) |
| NIP-AM (agent turn metric) | `src/agent_turn_metric.rs:1-8` (points to `docs/nips/NIP-AM.md`) |
| NIP-AP (persona/team/managed agent), NIP-ER (event reminder), NIP-PL (push lease), NIP-DV (DM visibility), NIP-IA (identity archival), NIP-RS (read state) | kind doc comments `src/kind.rs:71-389` |
| RFC 1918 / 2544 / 3849 / 6598 (address ranges) — plus RFC 6052 / 8215 / 4380 / 3056 added by `c26bf594`, RFC 3986 (host case), RFC 8259 (JSON strings) | `src/network.rs:26-45` (blocked-range doc list; RFC 6052 cited at `:6`), `src/tenant.rs:110-116`, `src/engram.rs:261-262` |


## Module: buzz-sdk (`crates/buzz-sdk`)

### Aspect: Integrations

---

### 1. Declared dependencies

`crates/buzz-sdk/Cargo.toml:10-16` (all workspace-pinned):

| Crate | Version (workspace) | Where declared | Usage in this crate |
|---|---|---|---|
| `buzz-core` | path `crates/buzz-core` (`Cargo.toml:124`) | `Cargo.toml:11` | kind constants, channel enums, `canonical_channel_name`, observer constants + `content_looks_like_nip44` |
| `nostr` | `0.44`, features `["nip44","nip98"]` (`Cargo.toml:61`) | `Cargo.toml:12` | `EventBuilder`, `Kind`, `Tag`, `EventId`, `Keys`, `PublicKey`, `SecretKey`, `SECP256K1`, `hashes::sha256`, `secp256k1::{Message, schnorr::Signature}`, `FromBech32` |
| `uuid` | `1`, features `["v4","serde"]` (`Cargo.toml:89`) | `Cargo.toml:13` | channel/workflow/action identifiers; `Uuid::parse_str` for validation |
| `serde` | `1`, feature `derive` (`Cargo.toml:64`) | `Cargo.toml:14` | **declared but no `use serde` / `#[derive(Serialize)]` appears in `src/`** — transitively needed only via `serde_json` |
| `serde_json` | `1` (`Cargo.toml:69`) | `Cargo.toml:15` | kind-0 content assembly, profile JSON parsing, auth-tag JSON encode/decode |
| `thiserror` | `2` (`Cargo.toml:85`) | `Cargo.toml:16` | `SdkError` derive (`lib.rs:87`) |

No `reqwest`, `tokio`, `axum`, `tungstenite`, or any async runtime. No dev-dependencies.

---

### 2. How each external crate is used

**`nostr` (0.44)**

| Import | Site | Purpose |
|---|---|---|
| `EventBuilder`, `Kind`, `Tag` | `builders.rs:23` | every builder's return value; `Kind::Custom(u16)` for all kinds; `Tag::parse(iter)` for tag construction |
| `Tag::parse` error mapping | `builders.rs:30-32`, `205-207` | mapped into `SdkError::InvalidTag` |
| `nostr::EventId` | `lib.rs:29-31`, `builders.rs:379`, `447`, `464`, `495`, `740` | typed event references; `.to_hex()` used for tag values |
| `nostr::Event` | `builders.rs:816` | `extract_channel_id` reads `event.tags` |
| `.allow_self_tagging()` | `builders.rs:1800`, `1821` | opt out of nostr 0.44's default same-pubkey `p`-tag stripping |
| `FromBech32`, `PublicKey` | `mentions.rs:32` | NIP-19 `npub` decoding in `extract_nostr_uris` |
| `hashes::sha256::Hash`, `hashes::Hash` | `nip_oa.rs:22-23` | SHA-256 of the NIP-OA preimage |
| `secp256k1::Message`, `secp256k1::schnorr::Signature`, `SECP256K1` | `nip_oa.rs:24-26` | BIP-340 Schnorr signing (`Keys::sign_schnorr`, `nip_oa.rs:170`) and verification (`SECP256K1.verify_schnorr`, `nip_oa.rs:241-243`) |
| `Keys`, `PublicKey::xonly()` | `nip_oa.rs:26`, `237-240` | owner key handling and x-only conversion for verification |
| `nostr::SecretKey`, `Keys::new` | `builders.rs:3756-3762` (test) | deterministic signing in the NIP-IA self-path test |

The `nip44` cargo feature of `nostr` is not used directly here — NIP-44
encryption lives in `buzz-core` (`crates/buzz-core/src/observer.rs:58`); the SDK
only length-checks ciphertext.

**`serde_json`**

| Site | Use |
|---|---|
| `builders.rs:541-561` | builds the kind-0 content object via `serde_json::Map` + `Value::String`, then `.to_string()` |
| `mentions.rs:183-190` | `serde_json::from_str::<Value>` on kind-0 `content` for name matching; parse failures are swallowed (`let Ok(..) else { continue }`) |
| `nip_oa.rs:124-133` | `parse_json_array` — `from_str` into `Value`, requires `Value::Array` |
| `nip_oa.rs:174` | `serde_json::json!([...])` to emit the auth tag |
| `builders.rs:2011-2016` (tests) | asserts on parsed kind-0 JSON |

**`uuid`**

| Site | Use |
|---|---|
| all channel-scoped builders | `Uuid` parameters rendered with `.to_string()` into `h`/`d`/`action_id` tags |
| `builders.rs:822` | `Uuid::parse_str` in `extract_channel_id` (invalid ⇒ `None`) |
| `builders.rs:1371-1373` | `Uuid::parse_str` validating and canonicalizing `GitPullRequestMeta.channel_id` |

**`buzz-core`**

| Import | Site | Use |
|---|---|---|
| 26 `KIND_*` constants | `builders.rs:6-19` | kind integers, cast `as u16` into `Kind::Custom` |
| `observer::{OBSERVER_AGENT_TAG, OBSERVER_FRAME_TAG, OBSERVER_FRAME_TELEMETRY, OBSERVER_FRAME_CONTROL, content_looks_like_nip44}` | `builders.rs:20-23` | observer-frame tag names, allowed frame values, ciphertext length gate |
| `channel::canonical_channel_name` | `builders.rs:623`, `636`, `675`; re-export `lib.rs:78` | channel-name normalization |
| `channel::{ChannelType, ChannelVisibility, MemberRole}` | re-exported `lib.rs:80-84`; used `builders.rs:566-578`, `674-696` | tag value vocabularies |
| `kind` module | re-exported `lib.rs:22` | so consumers avoid a direct `buzz-core` dependency |
| `observer::encrypt_observer_payload` | `builders.rs:1887-1892` (test only) | produces real NIP-44 ciphertext for the observer-frame happy path test |

---

### 3. HTTP / REST / WebSocket calls made

**None.** There are no HTTP methods, paths, URLs, sockets, or async functions in
this crate. The module doc states it explicitly
(`crates/buzz-sdk/src/lib.rs:13`), and the dependency set contains no HTTP or
WebSocket client (`crates/buzz-sdk/Cargo.toml:10-16`).

URLs appear only as **validated string values written into tags**, never as
request targets:

| Value | Validation | File:line |
|---|---|---|
| `DiffMeta.repo_url` | must start `http://`/`https://` | `builders.rs:317-321` |
| custom-emoji `url` | `http://`/`https://`, ≤2048 bytes | `builders.rs:152-170` |
| repo `clone_urls` | non-empty, ≤512 chars, scheme unchecked (so `ssh://`/`git@` forms are accepted — see test `builders.rs:2897-2925`) | `builders.rs:868-882` |
| repo `web_url` | `http://`/`https://`, ≤512 chars | `builders.rs:884-898` |
| repo `relays` | must start `ws://`/`wss://`, ≤256 chars | `builders.rs:900-919` |
| NIP-02 contact `relay_url` | ≤2048 bytes, scheme unchecked | `builders.rs:785-792` |
| `q`-tag relay hint | passed through unvalidated | `builders.rs:1266-1272` |
| PR/PR-update `clone_urls` | only non-emptiness of the list is checked; individual URLs unvalidated | `builders.rs:1344-1350`, `1425-1431` |

### 4. Error handling at integration boundaries

- All third-party errors are converted into `SdkError` variants rather than
  propagated: `Tag::parse` → `InvalidTag` (`builders.rs:30-32`), `Uuid::parse_str`
  → `InvalidInput` (`builders.rs:1371-1373`), `serde_json::from_str` →
  `InvalidInput` (`nip_oa.rs:125-127`), `PublicKey::from_hex` → `InvalidInput`
  (`nip_oa.rs:208-209`), `Signature::from_str` → `InvalidInput`
  (`nip_oa.rs:219-220`), `verify_schnorr` → `InvalidInput` (`nip_oa.rs:241-243`),
  `PublicKey::xonly` → `InvalidInput` (`nip_oa.rs:237-240`).
- Two integration points **swallow** errors deliberately: profile JSON that
  fails to parse is skipped (`mentions.rs:183-185`) and bech32 that fails to
  decode is skipped (`mentions.rs:381-386`). Both are documented as intentional.
- Signing errors are never handled here — `sign_with_keys` is called by the
  consumer, outside this crate.

---

### 5. Consumers (who integrates with this crate)

Declared `buzz-sdk` dependents (`grep` over workspace `Cargo.toml` files):
`crates/buzz-acp`, `crates/buzz-cli`, `crates/buzz-test-client`,
`crates/buzz-relay`, `desktop/src-tauri`.

Observed usage by symbol count in `src/`:

| Consumer | Distinct `buzz_sdk::*` paths referenced | Notes |
|---|---|---|
| `crates/buzz-cli` | 71 | the primary consumer |
| `crates/buzz-acp` | 7 | `build_message`, `build_reaction`, `build_remove_reaction`, `ThreadRef`, `nip_oa::{compute_auth_tag, parse_auth_tag, verify_auth_tag}` |
| `crates/buzz-relay` | 3 | `build_agent_observer_frame`, `ThreadRef`-adjacent paths, `nip_oa::verify_auth_tag` |
| `crates/buzz-test-client` | 0 `buzz_sdk::` paths in `src/` | declares the dependency but references it elsewhere (tests) or not at all |
| `desktop/src-tauri` | 0 `buzz_sdk::` paths in `src/` | builds its own events — `desktop/src-tauri/src/events.rs` contains 36 `EventBuilder::new` call sites, including its own `identity_archive_tags` (line 658) and `build_archive_identity_request` (line 716) |


## Module: buzz-persona (`crates/buzz-persona`)

### Aspect: Integrations

### External crate dependencies

`crates/buzz-persona/Cargo.toml:9-16`. Note: unlike most crates in this workspace, these
are **direct version pins, not `workspace = true`** entries.

| Crate | Version | Features | Why (evidence) |
|---|---|---|---|
| `serde` | `1` | `derive` | `Serialize`/`Deserialize` derives on all public config types — `crates/buzz-persona/src/persona.rs:51`, `:68`, `:82`, `:99`; `crates/buzz-persona/src/manifest.rs:35`, `:47`, `:77` |
| `serde_json` | `1` | default | Parses `plugin.json` (`crates/buzz-persona/src/manifest.rs:153`) and `.mcp.json` (`crates/buzz-persona/src/pack.rs:192-197`); also used as the **internal merge medium** — `PersonaConfig` is round-tripped to `serde_json::Value` so the JSON-based merge can run (`crates/buzz-persona/src/pack.rs:406-409`, `crates/buzz-persona/src/merge.rs:47-83`) |
| `serde_yaml` | `0.9` | default | Parses persona YAML frontmatter (`crates/buzz-persona/src/persona.rs:220`) and `SKILL.md` frontmatter during advisory validation (`crates/buzz-persona/src/validate.rs:410-418`) |
| `thiserror` | `2` | default | All three error enums: `PersonaError` (`crates/buzz-persona/src/persona.rs:26`), `ManifestError` (`crates/buzz-persona/src/manifest.rs:22`), `PackError` (`crates/buzz-persona/src/pack.rs:25`) |

Dev-dependencies — `crates/buzz-persona/Cargo.toml:17-18`:

| Crate | Version | Why |
|---|---|---|
| `tempfile` | `3` | Temp pack directories in tests — `crates/buzz-persona/src/pack.rs:450`, `crates/buzz-persona/src/resolve.rs:739`, `crates/buzz-persona/src/validate.rs:462`, `crates/buzz-persona/tests/integration.rs:117`, `crates/buzz-persona/tests/e2e_env_flow.rs:37` |

No internal Buzz crates are depended on — `crates/buzz-persona/Cargo.toml` has no
`buzz-*` entries. The crate is a leaf.

`serde_yaml` 0.9 is unmaintained upstream (archived by its author); this is a supply-chain
observation recorded in the debt doc, not a code finding.

---

### Filesystem access

All I/O is synchronous `std::fs`. There is no async runtime dependency.

| Operation | Call site | Path source |
|---|---|---|
| `canonicalize` pack root | `crates/buzz-persona/src/pack.rs:126` | caller-supplied `pack_dir` |
| `exists()` manifest probe | `crates/buzz-persona/src/pack.rs:133` | `<root>/.plugin/plugin.json` |
| `read_to_string` | `crates/buzz-persona/src/pack.rs:367` (via `read_file`) | manifest, personas, instructions, `.mcp.json` |
| `metadata` size guard | `crates/buzz-persona/src/pack.rs:375`, `crates/buzz-persona/src/persona.rs:264` | any file about to be read |
| `canonicalize` declared paths | `crates/buzz-persona/src/pack.rs:357` | manifest-declared relative paths |
| `is_dir()` skills probe | `crates/buzz-persona/src/pack.rs:224`, `:277` | `<root>/skills` |
| `read_dir` skills enumeration | `crates/buzz-persona/src/pack.rs:278`, `crates/buzz-persona/src/validate.rs:377` | `<root>/skills` |
| `read_to_string` manifest (2nd read) | `crates/buzz-persona/src/validate.rs:213`, `:305` | advisory checks re-read `plugin.json` |
| `read_to_string` SKILL.md | `crates/buzz-persona/src/validate.rs:393` | `<root>/skills/<name>/SKILL.md` |

Writes: **none**. There is no `fs::write`, `create_dir`, `remove_*`, or `File::create`
in `crates/buzz-persona/src/` (all such calls are inside `#[cfg(test)]` blocks and the
`tests/` directory).

Process execution: **none**. No `std::process::Command`, no `exec`. Hooks are data only
(`crates/buzz-persona/src/resolve.rs:339-357`).

---

### Network access

**None.** No HTTP client, no socket, no URL fetching. `crates/buzz-persona/Cargo.toml`
declares no `reqwest`/`hyper`/`ureq`/`tokio`. `homepage` and `repository` manifest fields
(`crates/buzz-persona/src/manifest.rs:94`, and `"repository"` in
`KNOWN_MANIFEST_KEYS` at `crates/buzz-persona/src/validate.rs:108`) are stored as opaque
strings and never dereferenced.

`resolve.rs` states the design contract explicitly: "**Pure**: no env access, no network,
no side effects" (`crates/buzz-persona/src/resolve.rs:11`). That holds for `resolve.rs`
itself; `resolve_pack` does perform filesystem reads transitively via `pack::load_pack`
(`crates/buzz-persona/src/resolve.rs:109`).

---

### Declared consumers in the workspace

| Consumer | Dependency declaration | Actual code usage |
|---|---|---|
| `buzz-cli` | `crates/buzz-cli/Cargo.toml:70` — `buzz-persona = { path = "../buzz-persona" }` | **Yes** — `crates/buzz-cli/src/commands/pack.rs:24`, `:28`, `:31`, `:62` |
| `buzz-acp` | `crates/buzz-acp/Cargo.toml:22` — `buzz-persona = { path = "../buzz-persona" }` | **No** — a grep for `buzz_persona` across all of `crates/buzz-acp` returns zero matches. The dependency is declared but unused at the code level. |
| desktop Tauri backend | `desktop/src-tauri/Cargo.toml:89` — `buzz_persona_pkg = { package = "buzz-persona", path = "../../crates/buzz-persona" }` | **Yes, one call** — `desktop/src-tauri/src/migration.rs:1123` uses `buzz_persona_pkg::persona::split_frontmatter` |
| workspace membership | `Cargo.toml:23` — `"crates/buzz-persona"` | — |

#### `buzz-cli` consumption pattern

Two subcommands, dispatched at `crates/buzz-cli/src/lib.rs:1739-1740`:

- `buzz pack validate <path>` → `commands::pack::cmd_validate`
  (`crates/buzz-cli/src/commands/pack.rs:15-46`): checks the path exists and is a
  directory, calls `buzz_persona::validate::validate_pack`
  (`crates/buzz-cli/src/commands/pack.rs:24`), prints each `ValidationDiagnostic` to
  stderr by matching on the enum variants (`:26-35`), then maps `has_errors()` to
  `CliError::Usage` (`:37-38`). It does **not** use
  `ValidationReport::exit_code()` or the `Display` impl.
- `buzz pack inspect <path>` → `commands::pack::cmd_inspect`
  (`crates/buzz-cli/src/commands/pack.rs:52-152`): calls
  `buzz_persona::resolve::resolve_pack` (`:62`) and pretty-prints the fully resolved
  effective config per persona — recombining `llm_provider` + `model` for display
  (`:78-87`), triggers (`:101-114`), MCP server count (`:120-122`), skills (`:124-126`),
  a truncated system-prompt preview (`:132-145`), and `runtime_env_vars` as `K=V` pairs
  (`:147-155`).

#### desktop consumption pattern

`desktop/src-tauri/src/migration.rs:1121-1132` (`rewrite_legacy_persona_md_runtime`) uses
only the frontmatter splitter, then re-parses the YAML itself with `serde_yaml` to rewrite
`runtime: "sprout-agent"` → `"buzz-agent"` and re-emits `---\n{frontmatter}---\n{body}`.
It deliberately bypasses `parse_persona_md` — it must round-trip unknown/legacy keys that
`deny_unknown_fields` would reject.

#### How buzz-acp is *expected* to consume it

The dependency exists but no call sites do. The intended contract is documented rather
than exercised:

- `crates/buzz-persona/src/resolve.rs:1-14` — "`ResolvedPersona` maps 1:1 to ACP's needs";
  field-level comments name the ACP targets: `system_prompt` → `Config.system_prompt`
  (`:31`), `model` → `Config.model` (`:36`), `subscribe` → `Config.subscribe_mode +
  channels_override` (`:47`), `triggers` → "mapped to ACP filter rules at startup" (`:49`).
- `runtime_env_vars` (`crates/buzz-persona/src/resolve.rs:64`) is the projection buzz-acp
  is expected to inject at spawn. The matching consumer field exists on the ACP side:
  `crates/buzz-acp/src/config.rs:533-535` — `pub persona_env_vars: Vec<(String, String)>`
  with the comment "Populated from persona pack resolution. Empty when no pack is
  configured." It is passed to the spawn path at `crates/buzz-acp/src/lib.rs:3733`
  (`extra_env: config.persona_env_vars.clone()`).
- Operator-precedence filtering (level 1) is explicitly the consumer's job, not this
  crate's: `crates/buzz-persona/src/resolve.rs:359-364` — "ACP is responsible for
  filtering based on operator precedence (level 1)".
- `desktop/src-tauri/src/managed_agents/types.rs:52` references
  "ACP's `resolve_persona_by_name()`", indicating the desktop/ACP contract expects that
  entry point.

So: the *data contract* between this crate and buzz-acp is designed and the receiving
field exists on the ACP `Config`, but the wire-up that would call
`resolve_pack`/`resolve_persona_by_name` from buzz-acp is not present in the current tree.

#### Import-filter convention (consumer-side, mirrored in tests)

`crates/buzz-persona/tests/e2e_env_flow.rs:15-32` defines
`DERIVED_PROVIDER_MODEL_ENV_KEYS = ["GOOSE_MODEL", "GOOSE_PROVIDER", "BUZZ_AGENT_MODEL",
"BUZZ_AGENT_PROVIDER"]` and a `filter_derived` helper described as mirroring "desktop
import_persona_pack logic" (`crates/buzz-persona/tests/e2e_env_flow.rs:200`). The
test asserts that on import the derived provider/model keys are stripped while
`GOOSE_TEMPERATURE` survives (`:206-228`). This is a duplicated copy of consumer logic
living in this crate's test suite — the real implementation is outside this crate.

---

### Standards / external specs referenced

| Spec | Where referenced | Enforced in code? |
|---|---|---|
| Open Plugin Spec (`open-plugin-spec.org`) | `PERSONA_PACK_SPEC.md:6-7`, §2; `"$schema"` accepted in `KNOWN_MANIFEST_KEYS` (`crates/buzz-persona/src/validate.rs:101`) | Partially — OPS field names are accepted and unknown fields tolerated (`crates/buzz-persona/src/manifest.rs:123-130`), but no schema fetch or validation |
| Semver (`engines.buzz`, `version`) | `crates/buzz-persona/src/manifest.rs:33-40` doc; `version: String` (`:82`) | No — plain strings, no semver crate |
| ACP (Agent Client Protocol) | `crates/buzz-persona/src/resolve.rs:1-14` and field comments | No protocol code here; shape-only alignment |
| MCP (Model Context Protocol) | `McpServerConfig` (`crates/buzz-persona/src/persona.rs:70`), `.mcp.json` `mcpServers` key (`crates/buzz-persona/src/resolve.rs:285`) | Config shape only; no MCP client |


## Module: buzz-ws-client (`crates/buzz-ws-client`)

### Aspect: Integrations

No internal workspace crates are depended on — the `[dependencies]` block
(`crates/buzz-ws-client/Cargo.toml:9`–`17`) lists only third-party crates, all
inherited via `workspace = true`. The single external system it talks to is a Nostr
relay over WebSocket.

---

### 1. External crates and how each is used

| Crate | Workspace version/features | Used for | file:line |
|---|---|---|---|
| `nostr` | `0.44`, features `nip44`, `nip98` (root `Cargo.toml:61`) | `Event`, `Keys`, `Tag` (`connection.rs:5`); `EventBuilder`, `RelayUrl` (`message.rs:1`); AUTH event construction + signing (`message.rs:181`, `:188`); `event.id.to_hex()` correlation keys (`connection.rs:80`, `:97`); `nostr::event::builder::Error` conversion (`error.rs:47`) | manifest `crates/buzz-ws-client/Cargo.toml:10` |
| `tokio` | `1`, features `rt-multi-thread, macros, net, time, sync, io-util, signal, process` (root `Cargo.toml:43`) | `tokio::time::timeout` (`connection.rs:7`, `:134`, `:187`, `:244`, `:284`); `tokio::time::Instant` deadlines (`connection.rs:176`, `:222`); `tokio::net::TcpStream` in the stream type (`connection.rs:14`) | `Cargo.toml:11` |
| `tokio-tungstenite` | `0.29`, features `rustls-tls-webpki-roots` (root `Cargo.toml:113`) | `connect_async` (`connection.rs:53`), `Message` frames (`connection.rs:124`, `:141`, `:148`, `:151`), `MaybeTlsStream`/`WebSocketStream` (`connection.rs:14`), `close(None)` (`connection.rs:116`), error type in `WsClientError::WebSocket` (`error.rs:8`) | `Cargo.toml:12` |
| `futures-util` | `0.3` (root `Cargo.toml:110`) | `SinkExt` for `send`, `StreamExt` for `next` on the WebSocket (`connection.rs:4`, `:124`, `:134`) | `Cargo.toml:13` |
| `serde_json` | `1` (root `Cargo.toml:69`) | `json!` frame construction (`connection.rs:6`, `:82`, `:98`), `to_string` (`connection.rs:122`), `from_str`/`from_value` parsing (`message.rs:63`, `:77`), `Value` as the `send_raw` parameter type (`connection.rs:121`) | `Cargo.toml:14` |
| `thiserror` | `2` (root `Cargo.toml:85`) | `#[derive(Error)]` and `#[error("…")]` messages on `WsClientError` (`error.rs:1`, `:4`–`:44`) | `Cargo.toml:15` |
| `url` | `2` (root `Cargo.toml:114`) | `url.parse::<url::Url>()` validation/normalization before dialing (`connection.rs:49`–`53`) | `Cargo.toml:16` |
| `tracing` | `0.1` (root `Cargo.toml:74`) | three `debug!` calls (`connection.rs:9`, `:57`, `:91`, `:123`) | `Cargo.toml:17` |

Not used: no `serde` derive dependency (all (de)serialization goes through
`serde_json::Value` plus the `nostr` crate's own impls), no `reqwest`/HTTP client,
no `rand`, no `anyhow`.

---

### 2. Transport and TLS configuration

- The stream type is `WebSocketStream<MaybeTlsStream<tokio::net::TcpStream>>`
  (`connection.rs:14`), i.e. TLS is optional per-connection and decided by
  `tokio-tungstenite` from the URL scheme.
- The crate calls the plain `connect_async` (`connection.rs:53`). It does **not**
  call `connect_async_tls_with_config`, does not construct a `Connector`, and passes
  no TLS config — verified: a search for `connect_async_tls_with_config`, `Connector`,
  `accept_invalid`, `rustls`, and `native_tls` inside `crates/buzz-ws-client/`
  returns no matches. So TLS behaviour is entirely the dependency default.
- Root-store selection therefore comes from the workspace feature choice
  `tokio-tungstenite = { version = "0.29", features = ["rustls-tls-webpki-roots"] }`
  (root `Cargo.toml:113`) — rustls with bundled webpki roots (not the OS trust store,
  not native-tls). This crate's own manifest inherits it with `workspace = true`
  (`crates/buzz-ws-client/Cargo.toml:12`) and adds no feature overrides.
- No WebSocket tuning is applied: no `WebSocketConfig`, no max-frame/max-message
  limits, no subprotocol or custom headers, no proxy support (verified — `connect_async`
  is invoked with a single URL argument only, `connection.rs:53`).

---

### 3. URL / scheme handling (ws vs wss)

| Behaviour | Evidence |
|---|---|
| Input is a `&str`; the only validation is `url::Url` parsing, which accepts any valid absolute URL scheme (including `http`/`https`) | `connection.rs:48`–`51` |
| **No scheme allow-list, no `ws`/`wss` check, no upgrade of `ws` → `wss`** anywhere in the crate | `connection.rs:48`–`65` (whole `connect`); no `starts_with("ws")` / scheme comparison exists |
| The dialed URL is the parsed/normalized form (`parsed.as_str()`), so `url` crate normalization (default port stripping, path defaulting to `/`) applies to the connection | `connection.rs:53` |
| The **original** string is what ends up in the AUTH event's relay tag, via `RelayUrl::parse(relay_url)` — so the AUTH-event relay value and the dialed URL can differ in normalization | store `connection.rs:63`; use `connection.rs:79` → `message.rs:180` |
| Rejection of non-WebSocket schemes is left to `tokio-tungstenite`, surfacing as `WsClientError::WebSocket` | `connection.rs:53`–`55` (behaviour of the dependency, not verified in this crate) |
| `nostr::RelayUrl::parse` may impose its own scheme rules; that logic lives in the `nostr` crate and is **not verifiable from these files** | `message.rs:180` |

---

### 4. Protocol integrations (wire level)

| Protocol surface | Direction | Evidence |
|---|---|---|
| NIP-01 `EVENT` (client→relay) | out | `connection.rs:98` |
| NIP-01 `OK` | in | `message.rs:87`–`104`; matched at `connection.rs:254` |
| NIP-01 `EVENT` (relay→client) | in | `message.rs:71`–`86` |
| NIP-01 `EOSE`, `CLOSED`, `NOTICE` | in | `message.rs:105`–`138` |
| NIP-42 `AUTH` challenge | in | `message.rs:139`–`146` |
| NIP-42 `AUTH` response (signed event) | out | `connection.rs:82` |
| "NIP-OA" authorization tag carried inside the AUTH event | out | `message.rs:169`–`186` (doc comment names it; example given as `["auth", "<token>"]`) |
| WebSocket Ping/Pong/Close control frames | both | `connection.rs:148`–`151` (and duplicates at `:208`–`:211`, `:262`–`:265`) |
| Anything else (`REQ`, `CLOSE`, `COUNT`) | out, untyped | via `send_raw` (`connection.rs:121`); e.g. `crates/buzz-test-client/src/lib.rs:154`, `:160` |

---

### 5. Consumers and parallel implementations in the repo

| Crate | Relationship | Evidence |
|---|---|---|
| `buzz-cli` | Depends on this crate; calls `publish_event` with a 75 s budget | `crates/buzz-cli/Cargo.toml:77`; `crates/buzz-cli/src/client.rs:1071`, `:1077`, `:1080` |
| `buzz-test-client` | Depends on this crate; wraps `NostrWsConnection` and re-exports its message/error types | `crates/buzz-test-client/Cargo.toml:13`; `crates/buzz-test-client/src/lib.rs:13`–`14`, `:85`, `:98` |
| `buzz-acp` | Does **not** depend on this crate; carries its own `connect_async` + NIP-42 + `RelayMessage` + parser implementation. No longer a byte-for-byte parallel: since the post-analysis sync the shared crate models a 7th inbound type (`Count`) that the ACP copy does not (`crates/buzz-acp/src/relay.rs:471`–`494` has 6 variants, no `"COUNT"` arm) | `crates/buzz-acp/src/relay.rs:3435`–`3461` (auth response), `:3610`–`3616` (AUTH parse), `:3843`–`3845` (handshake), `:2344`–`2350` (mid-session re-auth), `:471`–`494` (enum) |
| Other independent WebSocket clients in the repo (context for duplication, not dependencies) | separate implementations | `crates/buzz-relay/src/router.rs`, `crates/buzz-relay/src/audio/handler.rs`, `crates/buzz-pairing-cli/src/main.rs`, `crates/buzz-pair-relay/tests/integration.rs`, `desktop/src-tauri/src/native_websocket.rs` (all contain `connect_async`) |


## Module: buzz-db (`crates/buzz-db`)

### Integrations

#### 1. Dependencies

Declared at `crates/buzz-db/Cargo.toml:11-25`; all versions come from the
workspace table in `Cargo.toml`.

| Crate | Workspace version / features | Used for |
|-------|------------------------------|----------|
| `buzz-core` | path `crates/buzz-core` (`Cargo.toml:124`) | `CommunityId`, `StoredEvent`, `kind::*` predicates and constants, `channel::{ChannelType, ChannelVisibility, MemberRole, canonical_channel_name}` |
| `sqlx` | `0.9`, features `runtime-tokio`, `tls-rustls`, `postgres`, `uuid`, `chrono`, `json` (`Cargo.toml:52-54`) | the only database driver |
| `tokio` | `1` (`Cargo.toml:43`) | `tokio::spawn` for the fence probe, `tokio::time::{interval, timeout}` |
| `serde` / `serde_json` | `1` (`Cargo.toml:64`, `:69`) | JSONB round-trips, `Serialize` on admin/feedback records, event reconstruction |
| `uuid` | `1`, `v4`+`serde` (`Cargo.toml:89`) | all UUID columns and `Uuid::new_v4()` id minting |
| `chrono` | `0.4`, `serde` (`Cargo.toml:90`) | `DateTime<Utc>` ↔ `TIMESTAMPTZ`, month arithmetic in the partition manager |
| `hex` | `0.4` (`Cargo.toml:97`) | pubkey/event-id hex encode for `event_mentions.pubkey_hex` and admin projections |
| `sha2` | `0.11` (`Cargo.toml:96`) | approval-token hashing (`crates/buzz-db/src/workflow.rs:33`), DM participant hash (`crates/buzz-db/src/dm.rs:48`), push advisory-lock key derivation (`crates/buzz-db/src/push.rs:221-230`) |
| `tracing` | `0.1` (`Cargo.toml:74`) | `info!` in the partition manager, `warn!`/`debug!` on best-effort paths |
| `thiserror` | `2` (`Cargo.toml:85`) | `DbError`, `ProbeError` |
| `nostr` | `0.44`, `nip44`+`nip98` (`Cargo.toml:61`) | `nostr::Event` in insert signatures; `EventBuilder`/`Keys`/`Tag`/`Kind` when the crate itself signs the NIP-43 snapshot |

Dev-dependencies: only `tokio` (`crates/buzz-db/Cargo.toml:23-24`). There is no
`[features]` section — the crate has **no cargo features**.

#### 2. Postgres / sqlx specifics

**Query construction.** Runtime `sqlx::query()` / `sqlx::query_as::<_, T>()` /
`sqlx::query_scalar::<_, T>()` only — there is **no** use of `sqlx::query!`,
`query_as!`, or `query_scalar!` anywhere in the crate, so no `.sqlx/` offline
cache is required (design note at `crates/buzz-db/src/lib.rs:10`). Dynamic
SQL uses either `sqlx::QueryBuilder` (`crates/buzz-db/src/event.rs:360`,
`:591`, `crates/buzz-db/src/feed.rs:91`, `crates/buzz-db/src/lib.rs:146`,
`crates/buzz-db/src/channel.rs:1337`, `crates/buzz-db/src/event.rs:877`, `:957`)
or `format!` + `sqlx::AssertSqlSafe` with all values still bound (15 sites; see
`buzz-db-security.md`).

**Pool configuration** (`crates/buzz-db/src/lib.rs:387-407`, defaults at `:236-249`):

| Knob | Default | Notes |
|------|---------|-------|
| `max_connections` | 20 | sized so 4 relay pods × (20 main + 5 audit) fit PG `max_connections=100` |
| `min_connections` | 2 | |
| `acquire_timeout_secs` | 3 | |
| `max_lifetime_secs` | 1800 | |
| `idle_timeout_secs` | 600 | |

The **writer** pool installs an `after_connect` hook that runs
`SELECT set_config('buzz.created_at_floor', $1, false)` with
`replica_fence::CREATED_AT_FLOOR_SECS` on every connection
(`crates/buzz-db/src/lib.rs:394-405`); the replica pool does not
(`arm_floor_guard = false` at `:363`).

**Read-replica handling.** `DbConfig::read_database_url` optionally connects a
second pool with identical sizing (`crates/buzz-db/src/lib.rs:222-234`, `:361-364`).
`Db::read()` returns the replica when configured, else the writer
(`:470-472`). The documented routing contract restricts replica use to
lag-tolerant reads; exactly two call sites route conditionally:
`get_thread_replies` (`:2004-2043`) and `get_channel_window` (`:2063-2077`).
A background probe (`replica_fence::run_probe`, spawned by
`Db::spawn_fence_probe` at `:449-467`) performs a writer→replica LSN handshake
every 5 s.

**Postgres features relied upon**

| Feature | Where |
|---------|-------|
| Declarative range partitioning + partition pruning | `migrations/0001_initial_schema.sql:235`, `:341` |
| Generated `STORED` columns | `events.search_tsv`, `migrations/0001_initial_schema.sql:222` |
| GIN indexes (`tsvector`, `jsonb_path_ops`) | `migrations/0001_initial_schema.sql:278`, `migrations/0004_events_tags_gin.sql:21` |
| Partial and expression indexes | e.g. `migrations/0001_initial_schema.sql:61`, `:102`, `:178`, `:269` |
| Enum types + `::text` / `::enum` casts | `migrations/0001_initial_schema.sql:28-37`; casts e.g. `crates/buzz-db/src/channel.rs:118` |
| `plpgsql` trigger functions, `CREATE CONSTRAINT TRIGGER … DEFERRABLE INITIALLY DEFERRED` | `migrations/0021_created_at_fence_floor.sql:70`, `migrations/0022_event_ttl_refresh.sql:37` |
| Session/transaction GUCs (`current_setting`, `set_config`) | `buzz.created_at_floor`, `buzz.nip_rs_hard_delete` |
| Advisory locks — transaction-scoped (`pg_advisory_xact_lock`, `_shared`) and session-scoped (`pg_try_advisory_lock`) | `crates/buzz-db/src/lib.rs:517-535`, `:3329`, `:3506`, `:3661`; `crates/buzz-db/src/push.rs:27`, `:232`, `:236`; `crates/buzz-db/src/relay_members.rs:446` |
| `hashtextextended(text, 0)` for lock keys derived in SQL | `migrations/0023_push_match_gate.sql:34-35`, `migrations/0024_…:31-32`, `crates/buzz-db/src/channel.rs:1132-1139` |
| `xmax = 0` upsert-winner detection | `crates/buzz-db/src/lib.rs:836` |
| `FOR UPDATE`, `FOR KEY SHARE`, `SKIP LOCKED` | `crates/buzz-db/src/relay_members.rs:460`, `crates/buzz-db/src/push.rs:259`, `:656`, `:864`, `:1044`; `migrations/0009_…:124` |
| `UNNEST(array…) AS t(...)` set-wise DML | `crates/buzz-db/src/push.rs:636-638`, `:673-681`, `:709-712` |
| `DISTINCT ON`, `FILTER (WHERE …)`, `ROW_NUMBER() OVER (PARTITION BY …)`, `json_agg`/`jsonb_build_object`, `jsonb_array_elements`, `array_position` | `crates/buzz-db/src/event.rs:1346`; `crates/buzz-db/src/usage.rs:47-48`; `crates/buzz-db/src/thread.rs:690-696`; `crates/buzz-db/src/channel.rs:900`; `crates/buzz-db/src/lib.rs:2813-2817`; `crates/buzz-db/src/channel.rs:840` |
| Catalog introspection (`pg_class`, `pg_namespace`, `pg_inherits`, `pg_trigger`, `pg_attrdef`, `information_schema.tables`, `to_regclass`) | `crates/buzz-db/src/partition.rs:108-121`; `crates/buzz-db/src/replica_fence.rs:147-172`; `migrations/0014_push_lease_fts.sql:15-21`; `crates/buzz-db/src/relay_members.rs:544-548`; `crates/buzz-db/src/migration.rs:36-38` |
| Replication views (`pg_stat_activity`, `pg_prepared_xacts`, `pg_current_wal_lsn`, `pg_last_wal_replay_lsn`, `pg_is_in_recovery`, `pg_lsn` casts) | `crates/buzz-db/src/replica_fence.rs:404-463` |
| `pgcrypto` extension for `gen_random_uuid()` | `migrations/0001_initial_schema.sql:24` |
| `LOCK TABLE … IN SHARE ROW EXCLUSIVE MODE` in migrations | `migrations/0007_nip_rs_retention.sql:12`, `migrations/0008_…:9` |
| `SET CONSTRAINTS ALL IMMEDIATE` (verification only) | `crates/buzz-db/src/replica_fence.rs:232` |

**TLS.** Supplied by sqlx's `tls-rustls` feature (`Cargo.toml:53`). The crate
itself never sets TLS options — mode is whatever the connection URL specifies.

**Migration runner.** `sqlx::migrate!("../../migrations")` embeds the SQL at
compile time (`crates/buzz-db/src/migration.rs:11`); `MIGRATOR.run(pool)` at
`:19`. Checksums are therefore frozen — every schema change must be a new
additive file, a constraint asserted throughout
`crates/buzz-db/src/migration.rs:559-830`.

#### 3. Non-Postgres I/O

None. The crate opens no sockets or files of its own, spawns no processes, and
makes no HTTP calls: the only network egress is the sqlx Postgres connection(s).
The only spawned task is `tokio::spawn(replica_fence::run_probe(...))`
(`crates/buzz-db/src/lib.rs:463-467`), which itself only talks to the two pools.
Environment variables are read **only** inside `#[cfg(test)]` modules
(see `buzz-db-configuration.md`).

#### 4. Upstream / downstream coupling

- **Upstream (consumed):** `buzz-core` only — no other Buzz crate is a
  dependency (`crates/buzz-db/Cargo.toml:11-25`).
- **Downstream (consumers of this crate):** `buzz-db` is declared in the
  workspace dependency table (`Cargo.toml:126`) and is imported by the relay and
  other service crates; per `ARCHITECTURE.md:97` the relay is the only
  orchestrator and sibling service crates do not call each other.
- **Cross-crate constant duplication:** the FTS exclusion/allowlist kind lists
  and the push-eligible kind allowlist are inlined in frozen SQL and must be
  kept in sync with `buzz_core::kind` by hand — stated at
  `migrations/0001_initial_schema.sql:214-221`,
  `migrations/0005_agent_turn_metric_fts.sql:20-24`, and
  `migrations/0018_push_match_queue.sql:22-24`. The moderation action vocabulary
  is duplicated in `crates/buzz-db/src/moderation.rs:104-118` and asserted
  against the SQL CHECK by a test at `crates/buzz-db/src/migration.rs:640-645`.
- **Sibling crate referenced from comments only:** `buzz-search`
  (`crates/buzz-search/tests/fts_integration.rs` is named as the place to add
  FTS regression tests — `migrations/0001_initial_schema.sql:220-221`).


## Module: buzz-auth (`crates/buzz-auth`)

### Integrations

### Infrastructure reach — explicit answer

| Resource | Does this crate touch it? | Evidence |
|----------|---------------------------|----------|
| **Redis** | **No.** No `redis`, `deadpool-redis`, or any Redis client in the manifest (`crates/buzz-auth/Cargo.toml:14-26`). The crate only *formats Redis key strings* (`crates/buzz-auth/src/rate_limit.rs:201-215`, `crates/buzz-auth/src/nip98_replay.rs:114-121`) and *documents* Redis semantics that implementors must satisfy (`crates/buzz-auth/src/nip98_replay.rs:60-62`, `crates/buzz-auth/src/rate_limit.rs:3-4`). No command is ever issued from here. |
| **Postgres** | **No.** No `sqlx` and no `buzz-db` dependency (`crates/buzz-auth/Cargo.toml:14-26`). The `ChannelAccessChecker` trait exists precisely so the crate can enforce access "without depending on `buzz-db` directly" (`crates/buzz-auth/src/access.rs:3-4`, `:18-20`). |
| **Network (HTTP/WS/DNS)** | **No.** No `reqwest`, `axum`, `hyper`, `tokio-tungstenite`, or socket usage. NIP-42 verification is documented as "Pure cryptographic verification — no network calls, no JWT, no tokens" (`crates/buzz-auth/src/lib.rs:117`). `url` is used only for string parsing/normalisation (`crates/buzz-auth/src/nip42.rs:10`, `crates/buzz-auth/src/nip98.rs:28`) — it performs no resolution or I/O. `std::net::IpAddr` is used only as a key-formatting input (`crates/buzz-auth/src/rate_limit.rs:9`, `:213-215`). |
| **Filesystem** | **No.** No `std::fs`, no path handling, no config file reading. `AuthConfig` is a plain serde struct; the caller loads it (`crates/buzz-auth/src/lib.rs:89-95`, loaded at `crates/buzz-relay/src/config.rs:585`). |
| **Clock** | Yes — `nostr::Timestamp::now()` for skew checks (`crates/buzz-auth/src/nip42.rs:78`, `crates/buzz-auth/src/nip98.rs:78`). |
| **OS CSPRNG** | Yes — `rand::random::<[u8; 32]>()` for challenges (`crates/buzz-auth/src/nip42.rs:39`). |
| **Async runtime** | Yes — `tokio::task::spawn_blocking` for the CPU-bound Schnorr verify (`crates/buzz-auth/src/lib.rs:128-132`). |

Net: `buzz-auth` is a pure-compute library. Its only side effects are reading the
wall clock and drawing OS randomness.

---

### Internal dependencies

| Crate | Declared | What is used | Where |
|-------|----------|--------------|-------|
| `buzz-core` | `crates/buzz-auth/Cargo.toml:15` | `verify_event(&Event) -> Result<(), VerificationError>` — verifies event id hash then Schnorr signature (`crates/buzz-core/src/verification.rs:11-32`) | `crates/buzz-auth/src/nip42.rs:56`, `crates/buzz-auth/src/nip98.rs:74` |
| `buzz-core` | same | `TenantContext` (and `.community()`) for community scoping | `crates/buzz-auth/src/access.rs:9`, `crates/buzz-auth/src/rate_limit.rs:11`, `crates/buzz-auth/src/nip98_replay.rs:36` |
| `buzz-core` | same | `CommunityId` — test fixtures only | `crates/buzz-auth/src/rate_limit.rs:247`, `crates/buzz-auth/src/access.rs:156`, `crates/buzz-auth/src/nip98_replay.rs:144` |

`buzz-core` is the crate's only internal dependency. It does **not** depend on
`buzz-db`, `buzz-pubsub`, `buzz-relay`, or any other workspace crate.

---

### External crates and why

| Crate | Declared | Used for | Where |
|-------|----------|----------|-------|
| `nostr` | `crates/buzz-auth/Cargo.toml:16` | `Event`, `PublicKey`, `EventId`, `Kind` (`Authentication`, `HttpAuth`), `TagKind` (`Challenge`, `Relay`, `Method`, `Payload`), `SingleLetterTag`/`Alphabet` for the `u` tag, `Timestamp`, `SecretKey`, `Keys` | `crates/buzz-auth/src/nip42.rs:9`, `crates/buzz-auth/src/nip98.rs:26`, `crates/buzz-auth/src/lib.rs:66`, `crates/buzz-auth/src/nip98_replay.rs:37` |
| `serde` | `crates/buzz-auth/Cargo.toml:17` | `Serialize`/`Deserialize` on `AuthConfig`, `RateLimitConfig`, `LimitType`; `#[serde(default = ...)]` field fallbacks and `rename_all = "snake_case"` | `crates/buzz-auth/src/lib.rs:90`, `crates/buzz-auth/src/rate_limit.rs:13`, `:56-57`, `:85` |
| `serde_json` | `crates/buzz-auth/Cargo.toml:18` | parse the NIP-98 `Authorization` event JSON (`from_str`) | `crates/buzz-auth/src/nip98.rs:62`; also test serialisation `:185` |
| `tokio` | `crates/buzz-auth/Cargo.toml:19` | `task::spawn_blocking` to keep Schnorr verification off async worker threads; `#[tokio::test]` in tests | `crates/buzz-auth/src/lib.rs:128`, tests `:199`, `crates/buzz-auth/src/access.rs:163` |
| `sha2` | `crates/buzz-auth/Cargo.toml:22` | `Sha256` for the NIP-98 `payload` body hash and for the dev key derivation | `crates/buzz-auth/src/nip98.rs:27`, `:120`; `crates/buzz-auth/src/lib.rs:161-163` |
| `hex` | `crates/buzz-auth/Cargo.toml:23` | encode the 32-byte challenge and the computed body hash for comparison | `crates/buzz-auth/src/nip42.rs:40`, `crates/buzz-auth/src/nip98.rs:121` |
| `rand` | `crates/buzz-auth/Cargo.toml:24` | `rand::random::<[u8; 32]>()` challenge entropy | `crates/buzz-auth/src/nip42.rs:39` |
| `url` | `crates/buzz-auth/Cargo.toml:26` | `Url::parse` + path/host manipulation for both URL normalisers | `crates/buzz-auth/src/nip42.rs:10`, `:20-32`; `crates/buzz-auth/src/nip98.rs:28`, `:146-152` |
| `uuid` | `crates/buzz-auth/Cargo.toml:25` | `Uuid` channel ids in `AuthContext` and the access trait | `crates/buzz-auth/src/lib.rs:72`, `crates/buzz-auth/src/access.rs:11` |
| `thiserror` | `crates/buzz-auth/Cargo.toml:21` | derive `Error` + `Display` on `AuthError` | `crates/buzz-auth/src/error.rs:8` |
| `tracing` | `crates/buzz-auth/Cargo.toml:20` | **declared but unused** — no `tracing::` call, no `use tracing`, and no `#[instrument]` anywhere in `crates/buzz-auth/src/` | manifest `crates/buzz-auth/Cargo.toml:20` |

All dependencies use `{ workspace = true }` — no crate-local version pins
(`crates/buzz-auth/Cargo.toml:15-26`).

---

### Consumers of this crate

| Consumer | Declared | What it uses |
|----------|----------|--------------|
| `buzz-relay` | `crates/buzz-relay/Cargo.toml:22` (plus `features = ["dev"]` in `[dev-dependencies]`, `:90`) | `AuthService`/`AuthConfig` (`crates/buzz-relay/src/state.rs:19`, `:500`; `crates/buzz-relay/src/main.rs:368`), `AuthContext` (`crates/buzz-relay/src/connection.rs:17`, `:44`), `generate_challenge` + `verify_auth_event` (`crates/buzz-relay/src/handlers/auth.rs:87-89`, `crates/buzz-relay/src/audio/handler.rs:222`), `verify_nip98_event` (`crates/buzz-relay/src/api/bridge.rs:111`), `LimitType`/`RateLimiter` (`crates/buzz-relay/src/admission.rs:1`), `Nip98ReplayGuard` + `DEFAULT_REPLAY_TTL_SECS` (`crates/buzz-relay/src/api/bridge.rs:16`, `crates/buzz-relay/src/state.rs:582`), `Scope::all_known()` (`crates/buzz-relay/src/api/bridge.rs:829`), `RateLimitConfig` (`crates/buzz-relay/src/config.rs:284-316`) |
| `buzz-pubsub` | `crates/buzz-pubsub/Cargo.toml:12` | implements `RateLimiter` as `RedisRateLimiter` (`crates/buzz-pubsub/src/rate_limiter.rs:99-121`) and `Nip98ReplayGuard` as `RedisNip98ReplayGuard` (`crates/buzz-pubsub/src/nip98_replay.rs:34`); calls `rate_limit_key` / `ip_rate_limit_key` (`crates/buzz-pubsub/src/rate_limiter.rs:108`, `:118`) |
| `buzz-admin` | `crates/buzz-admin/Cargo.toml:17` | **nothing** — a grep of `crates/buzz-admin/` for `buzz_auth` returns zero matches; the dependency appears unused |

`crates/buzz-conformance/Cargo.toml:18` carries an explicit prohibition comment
naming `buzz-auth` among crates it must never depend on.

---

### Protocol / spec conformance surface

| Spec | Kind | Where implemented |
|------|------|-------------------|
| NIP-42 (client authentication of clients to relays) | `Kind::Authentication` = 22242 | `crates/buzz-auth/src/nip42.rs:52`; module doc describes the 3-step handshake at `:1-7` |
| NIP-98 (HTTP Auth) | `Kind::HttpAuth` = 27235 | `crates/buzz-auth/src/nip98.rs:66`; `Authorization: Nostr <base64(event)>` header shape documented at `:9-11` (base64 decoding is the caller's job, `:38`) |
| NIP-OA (owner attestation) | — | not implemented here; only the `AuthContext.agent_owner_pubkey` field is reserved for the relay to fill (`crates/buzz-auth/src/lib.rs:75-79`). The tag extraction/verification lives in `crates/buzz-relay/src/handlers/auth.rs:26-36`. |
| NIP-29 (relay-based groups) | — | referenced only as the justification for granting full scopes (`crates/buzz-auth/src/lib.rs:135`) |

The `nostr` workspace dependency enables the `nip98` feature explicitly
(`Cargo.toml:61`), which is what provides `TagKind::Method` / `TagKind::Payload`.


## Module: buzz-pubsub (`crates/buzz-pubsub`)
### Aspect: Integrations

---

### 1. External systems

| System | Protocol | Client | Usage |
|---|---|---|---|
| Redis | RESP over TCP | `deadpool-redis` pool (`Cargo.toml:14`) | All imperative commands: `PUBLISH`, `SET`, `GET`, `MGET`, `DEL`, `TTL`, `EXPIRE`, `EVALSHA` |
| Redis | RESP pub/sub mode | `redis::Client` → `get_async_pubsub()` (`Cargo.toml:13`) | Three independent stateful connections: events (`subscriber.rs:85-87`), cache-invalidate (`cache_invalidation.rs:135-139`), conn-control (`conn_control.rs:126-130`) |

Redis is the **only** external system. No Postgres, no HTTP client, no object
store, no message broker. The crate declares no `buzz-db`, `reqwest`, `axum`, or
`sqlx` dependency (`Cargo.toml:11-24`).

Redis command inventory (exhaustive, by call site):

| Command | Site |
|---|---|
| `PUBLISH` | `publisher.rs:32-36`; `lib.rs:276-280` (cache), `:296-300` (conn-control) |
| `SET … EX` | `presence.rs:37-43` |
| `SET … NX EX` | `nip98_replay.rs:68-78` |
| `DEL` | `presence.rs:53-56` |
| `GET` | `presence.rs:69` |
| `MGET` | `presence.rs:88` |
| `EXPIRE` (repair) | `rate_limiter.rs:60-65` |
| `INCR` + `EXPIRE` + `TTL` (in Lua) | `rate_limiter.rs:24-31`, invoked `:38-42` |
| `SUBSCRIBE` / `UNSUBSCRIBE` | `subscriber.rs:98`, `:119`, `:127` |
| `PSUBSCRIBE` | `cache_invalidation.rs:139`, `conn_control.rs:130` |
| `TTL` (assertions only) | `lib.rs:491`, `presence.rs:196` — test code |

Four connections per relay pod minimum: one pool (shared, size set by the caller)
plus three dedicated pub/sub sockets. The pool is **injected**, never created here —
every constructor takes `deadpool_redis::Pool` as a parameter (`lib.rs:117`, `:122`,
`rate_limiter.rs:94`, `nip98_replay.rs:29`), so pool sizing and TLS configuration
are the relay's concern. Only the pub/sub connections are opened from the crate's
own `redis_url` string (`lib.rs:105`).

### 2. Internal crate dependencies

| Dependency | Declared | What is actually used |
|---|---|---|
| `buzz-core` | `Cargo.toml:11` | `TenantContext` (`topic.rs:6`, `presence.rs:7`, `lib.rs:50`), `CommunityId` (`topic.rs:6`, `cache_invalidation.rs:17`, `conn_control.rs:20`) |
| `buzz-auth` | `Cargo.toml:12` | `RateLimiter` / `LimitType` / `RateLimitResult` / `rate_limit_key` / `ip_rate_limit_key` (`rate_limiter.rs:12-15`, `:110`, `:118`); `Nip98ReplayGuard` / `nip98_replay_key_for_scope` / `DEFAULT_REPLAY_TTL_SECS` / `MAX_REPLAY_TTL_SECS` (`nip98_replay.rs:8-14`, `:81`); `AuthError` (`rate_limiter.rs:13`, `nip98_replay.rs:9`) |
| `nostr` | `Cargo.toml:22` | `Event` + `JsonUtil` for wire (de)serialization (`publisher.rs:4`, `subscriber.rs:8`), `PublicKey` (`presence.rs:9`), `EventId` (`nip98_replay.rs:16`) |
| `uuid` | `Cargo.toml:17` | Channel and community ids (`topic.rs:7`) |
| `futures-util` | `Cargo.toml:23` | `StreamExt` for pub/sub stream consumption + sink/stream split (`subscriber.rs:7`, `:86`) |
| `serde` / `serde_json` | `Cargo.toml:15-16` | Control-message payloads (`cache_invalidation.rs:18`, `conn_control.rs:21`) |
| `tokio` | `Cargo.toml:14` | `broadcast`, `mpsc`, `Mutex`, `select!`, `sleep`, `spawn` |
| `tracing` | `Cargo.toml:20` | Structured logging throughout |
| `thiserror` | `Cargo.toml:21` | `PubSubError` derive (`error.rs:1`) |
| `chrono` | `Cargo.toml:18` | **Declared but no `chrono::` path appears in any source file** — unused |

`buzz-auth` is a genuine inbound coupling: this crate exists partly to provide the
Redis-backed implementations of two `buzz-auth` traits, which inverts the usual
layering (a lower-level transport crate depending on the auth crate). Noted as a
structural observation, not a defect — the alternative would be a `buzz-auth`
dependency on Redis.

### 3. Consumers

| Consumer | Manifest | Verified code usage |
|---|---|---|
| `buzz-relay` | `crates/buzz-relay/Cargo.toml` | Extensive — see table below |
| `buzz-admin` | `crates/buzz-admin/Cargo.toml` | Declared; no `buzz_pubsub::` path found in the relay-side grep sweep of `crates/**/*.rs` outside `buzz-relay` and `buzz-test-client` comments |
| `buzz-conformance` | `crates/buzz-conformance/Cargo.toml` | Declared; same — no verified call site |

Relay integration points (all verified by grep against `crates/**/*.rs`):

| Seam | Relay site |
|---|---|
| Manager construction | `buzz-relay/src/state.rs` (imports `state.rs:27`) |
| `run_conn_control_subscriber` spawn | `main.rs:366` |
| `subscribe_local` | `main.rs:822`, `handlers/event.rs:1644` |
| `subscribe_conn_control` | `main.rs:903`; dispatch `main.rs:908`, `:913` |
| `retain_topic` | `handlers/req.rs:256`, `handlers/event.rs:1683`, `:1687` |
| `release_topic` | `connection.rs:268`, `handlers/close.rs:21`, `handlers/req.rs:251`, `handlers/side_effects.rs:81` |
| `publish_cache_invalidation` | `state.rs:970` |
| `publish_conn_control` | `state.rs:1044`, `:1066` |
| `set_presence` / `clear_presence` | `handlers/event.rs:798` / `connection.rs:280`, `handlers/event.rs:793` |
| `get_presence_bulk` | `api/bridge.rs:1972` |
| `RedisRateLimiter` | import `state.rs:26`, field `state.rs:584`, construction `state.rs:712`, enforcement `connection.rs:593-648` via `admission.rs:14-34` |
| `RedisNip98ReplayGuard` | import `state.rs:27`, construction `state.rs:711`, two-pod tests `api/bridge.rs:2293-2294`, `:2304` |

Cross-node behaviour is additionally pinned from outside the crate by
`crates/buzz-test-client/tests/conformance_multitenant.rs:2371` and `:2484`, which
document that presence answers come from Redis via `get_presence_bulk` with **no DB
fallback**, and that community A's query must return only A's data.

### 4. Contract boundaries and coupling risks

- **Redis is a hard availability dependency for authenticated traffic.** A Redis
  outage makes `check_and_increment` return `AuthError::Internal`
  (`rate_limiter.rs:44-46`), which the relay maps to `AdmissionError::Unavailable`
  (`admission.rs:29-33`) and treats as denial — so every authenticated WS
  `EVENT`/`REQ`/`COUNT` is rejected (`connection.rs:612-621`). Fail-closed is the
  correct security posture, but it means Redis is on the critical path for reads,
  not just for fan-out.
- **Shared Redis across all tenants.** Isolation is by key prefix only
  (`topic.rs:13`), and two subscribers use cross-tenant `buzz:*` patterns
  (`cache_invalidation.rs:27`, `conn_control.rs:30`). Any process with access to the
  Redis instance sees every community's traffic.
- **No schema versioning on control payloads.** `CacheInvalidation`
  (`cache_invalidation.rs:57-80`) and `ConnControl` (`conn_control.rs:55-73`) are
  internally tagged with no version field and are not `#[non_exhaustive]`, so a
  rolling deploy that adds a variant produces `warn`-level deserialization failures
  on old pods (`cache_invalidation.rs:161-165`, `conn_control.rs:152-156`) —
  silently skipped, which for `ConnControl` means a ban may not be enforced on
  older pods until they restart. Mitigated by the DB backstop
  (`conn_control.rs:18-21`) but not by the transport.
- **Event payloads are whole Nostr events**, so Redis bandwidth scales with event
  size, and the pub/sub message size limit becomes an implicit cap on event size
  (`publisher.rs:31`).


## Module: buzz-search (`crates/buzz-search`)

### Integrations

#### Dependencies — `crates/buzz-search/Cargo.toml`

| Dependency | Declaration | Line | Workspace version/features |
|---|---|---|---|
| `buzz-core` | `{ workspace = true }` | `Cargo.toml:11` | path `crates/buzz-core` (root `Cargo.toml:124`) |
| `sqlx` | `{ workspace = true }` | `Cargo.toml:12` | `0.9`, features `runtime-tokio, tls-rustls, postgres, uuid, chrono, json` (root `Cargo.toml:52-54`) |
| `uuid` | `{ workspace = true }` | `Cargo.toml:13` | `1`, features `v4, serde` (root `Cargo.toml:89`) |
| `thiserror` | `{ workspace = true }` | `Cargo.toml:14` | `2` (root `Cargo.toml:85`) |
| `tokio` (dev only) | `{ workspace = true }` | `Cargo.toml:17` | `1`, multi-thread/macros/etc. (root `Cargo.toml:43`) |

Four runtime dependencies total. No HTTP client, no serde, no tracing, no Redis, no
S3. Package description: "Postgres full-text search for Buzz, scoped by community"
(`Cargo.toml:8`).

#### buzz-core usage

| Item | Import | Use |
|---|---|---|
| `buzz_core::CommunityId` | `query.rs:14`, re-exported at `lib.rs:29` | `SearchQuery.community` field type (`query.rs:76`); `as_uuid()` for the SQL bind (`query.rs:241`, definition at `crates/buzz-core/src/tenant.rs:47-49`) |

That is the entire coupling to `buzz-core` in `src/`. Tests additionally import
`buzz_core::kind::{AUTHOR_ONLY_KINDS, KIND_AGENT_TURN_METRIC, KIND_MEMBER_ADDED_NOTIFICATION, KIND_MEMBER_REMOVED_NOTIFICATION, P_GATED_KINDS}`
and `buzz_core::kind::is_ephemeral` (`tests/fts_integration.rs:9-16`, `1382`) as
drift tripwires against the schema's hard-coded exclusion list.

#### sqlx / Postgres usage

| Aspect | Detail | Line |
|---|---|---|
| Imports | `sqlx::{PgPool, QueryBuilder, Row}` in query, `sqlx::PgPool` in lib | `query.rs:15`, `lib.rs:33` |
| Query style | Runtime `QueryBuilder<sqlx::Postgres>` — **not** the compile-time `sqlx::query!` macros; no `.sqlx/` offline cache is used by this crate | `query.rs:233` |
| Binding | `push_bind` for every dynamic value: community UUID (`241`), prefix/fulltext term (`144`, `168`), channel id vec (`257`, `262`), kinds vec (`270`), authors vec (`278`), since (`285`), until (`291`), limit (`296`), offset (`298`) | as listed |
| Execution | `qb.build().fetch_all(pool).await?` — single round trip, no transaction, no prepared-statement caching directives | `query.rs:300` |
| Row decoding | `Row::try_get` by column name for all six columns | `query.rs:304-318` |
| Postgres-specific SQL | `to_timestamp`, `EXTRACT(EPOCH FROM ...)::bigint`, `websearch_to_tsquery`, `to_tsvector`, `tsvector_to_array`, `ts_rank_cd`, `@@`, `= ANY(...)`, `CROSS JOIN LATERAL`, `regexp_split_to_table ... WITH ORDINALITY`, `string_agg`, `quote_literal`, `::tsquery` | `query.rs:143`, `154-176`, `234-298` |
| Types crossing the boundary | `Uuid` ↔ `uuid`, `Vec<u8>` ↔ `BYTEA`, `i32` ↔ `INT`, `i64` ↔ `BIGINT`, `f32` ↔ `real` (`ts_rank_cd` result), `Option<Uuid>` ↔ nullable `uuid` | `query.rs:304-318` |

#### Pool handling

`SearchService` stores an owned `PgPool` clone (`lib.rs:41`, `lib.rs:46-48`). The
crate never creates, configures, closes, or resizes a pool — no `PgPoolOptions` in
`src/`. `PgPool` is internally `Arc`-based, so `#[derive(Clone)]` on
`SearchService` (`lib.rs:39`) shares the same pool. The relay wraps it in
`Arc<SearchService>` in `AppState` and constructs it from the relay's pool
(`crates/buzz-relay/src/state.rs:1273`).

#### Non-Postgres I/O

None. No filesystem access, no network client, no environment reads, no process
spawning, no clock or RNG use in `src/`. The only `std::env::var` calls in the
crate are in the test harness (`tests/fts_integration.rs:33`, `92`).

#### Typesense status — explicit check

| Question | Answer | Evidence |
|---|---|---|
| Typesense dependency in `Cargo.toml`? | No | `Cargo.toml:10-18` lists only buzz-core, sqlx, uuid, thiserror, tokio |
| Typesense client/HTTP code anywhere in the crate? | No | grep for `typesense` (case-insensitive) across the crate returns 2 hits, both prose in doc comments |
| Remaining mentions | Two, historical | `query.rs:20` ("the legacy … matrix from the Typesense relay"), `query.rs:46` ("what the legacy Typesense `channel_id:=__global__` sentinel meant") |

Related historical prose outside the crate (not code): the schema comment
"Full-text search vector (Typesense → Postgres FTS)"
(`migrations/0001_initial_schema.sql:200`, `schema/schema.sql:199`).

#### Test-harness integrations

The integration test file couples directly to migration SQL at compile time via
`include_str!` (`tests/fts_integration.rs:22-32`): migrations `0001`–`0008` plus
`0014`, applied in order into a per-test schema (`tests/fts_integration.rs:57-84`).
It creates and drops a uniquely-named schema through a separate one-connection
admin pool (`tests/fts_integration.rs:36-46`, `87-103`) and passes the schema via
the connection URL option `options=-c search_path=<schema>`
(`tests/fts_integration.rs:48`). Adding a future FTS-affecting migration requires
editing this list — noted in-file at `tests/fts_integration.rs:55-56`.


## Module: buzz-audit (`crates/buzz-audit`)

### Aspect: Integrations

### Declared dependencies (`crates/buzz-audit/Cargo.toml:11-22`)

| Crate | Declared line | Workspace version (`Cargo.toml`) | Actually used in `src`? | Where |
|---|---|---|---|---|
| `buzz-core` | `Cargo.toml:11` | path `crates/buzz-core` (`Cargo.toml:130`) | Yes — `CommunityId` | `entry.rs:1`, `service.rs:7` |
| `sqlx` | `:12` | `0.9`, features `runtime-tokio, tls-rustls, postgres, uuid, chrono, json` (`Cargo.toml:52-54`) | Yes | `service.rs:3` (`Acquire, PgPool, Row`), `:84`, `:238` |
| `tokio` | `:13` | `1` (`Cargo.toml:43`) | **Tests only** | `service.rs:265` (`tokio::sync::Mutex`), `#[tokio::test]` at `:318,338,376,437,475,512` |
| `serde` | `:14` | `1` + derive (`Cargo.toml:64`) | Yes — derives | `action.rs:1,6-7`, `entry.rs:3,13` |
| `serde_json` | `:15` | `1` (`Cargo.toml:69`) | Yes | `entry.rs:34`, `hash.rs:80-116`, `error.rs:40` |
| `uuid` | `:16` | `1` + v4, serde (`Cargo.toml:89`) | Yes | `entry.rs:4,16`, `service.rs:5,246`, `hash.rs:45` (`Uuid::as_bytes`) |
| `chrono` | `:17` | `0.4` + serde (`Cargo.toml:90`) | Yes | `entry.rs:2,36`, `hash.rs:1,22-24`, `service.rs:1,21` |
| `tracing` | `:18` | `0.1` (`Cargo.toml:74`) | Yes | `service.rs:4` (`debug, instrument, warn`), `:52`, `:128`, `:159`, `:211`, `:241` |
| `thiserror` | `:19` | `2` (`Cargo.toml:85`) | Yes | `error.rs:1,11` |
| `sha2` | `:20` | `0.11` (`Cargo.toml:96`) | Yes | `hash.rs:2` (`Digest, Sha256`), `:43`, `:72` |
| `hex` | `:21` | `0.4` (`Cargo.toml:97`) | **No** — grep for `hex::` in `crates/buzz-audit/src` returns nothing | unused declaration |
| `futures-util` | `:22` | `0.3` (`Cargo.toml:110`) | Yes — `FutureExt::catch_unwind` | `service.rs:2`, `:68` |

No `[features]`, `[dev-dependencies]`, or `[build-dependencies]` sections exist in
`crates/buzz-audit/Cargo.toml`.

### Postgres / sqlx usage

- Connection handle is a `PgPool` held by value in `AuditService` (`service.rs:37-45`).
  The crate never creates the pool; the relay passes one in
  (`crates/buzz-relay/src/main.rs:321-330`, a dedicated pool with
  `max_connections(5)`, `min_connections(1)`).
- Write path pins a single pooled connection: `self.pool.acquire()` (`service.rs:54`),
  used for the lock (`:59-62`), the transaction (`:87`), and the unlock (`:71-74`) —
  necessary because advisory locks are session-scoped (`service.rs:49-51`).
- Transaction via `Acquire::begin` (`service.rs:87`) and `commit` (`:149`). No explicit
  rollback call; a dropped `Transaction` rolls back implicitly on the error paths
  (`service.rs:101`, `:126`, `:147`).
- Reads run directly on `&self.pool` with `fetch_all` (`service.rs:178`, `:231`).
- All statements are untyped `sqlx::query` with bind parameters (no `query!`/`query_as!`
  macros), so there is **no compile-time schema verification** and column access is
  runtime `Row::get` (`service.rs:105-106`, `:246-254`).
- Postgres-specific SQL used: `pg_advisory_lock`, `pg_advisory_unlock`,
  `hashtextextended($1, 0)` (`service.rs:59`, `:71`). `hashtextextended` is a Postgres
  internal hash function — this crate is not portable to other engines.
- Types crossing the boundary: `UUID`↔`Uuid`, `BIGINT`↔`i64`, `BYTEA`↔`Vec<u8>`,
  `VARCHAR`↔`String`/`&str`, `JSONB`↔`serde_json::Value`, `TIMESTAMPTZ`↔`DateTime<Utc>`
  (binds at `service.rs:137-145`; decodes at `:246-254`).

### Cryptography

`sha2::Sha256` only, used incrementally (`Sha256::new`, `update`, `finalize`) in
`compute_hash` (`hash.rs:43-72`). No HMAC, no signatures, no randomness, no key
material anywhere in the crate.

### Non-Postgres I/O

None. The crate performs no filesystem, network, HTTP, S3, Redis, or process I/O.
The only environment read is `DATABASE_URL` inside a test helper
(`service.rs:275-279`). Logging goes through `tracing` macros only.

### How the relay integrates it (fire-and-forget semantics)

| Step | Location | Behaviour |
|---|---|---|
| Construction gate | `crates/buzz-relay/src/main.rs:321-334` | `AuditService` built only when `config.audit_enabled`; otherwise `None` and an info log |
| Enabled gauge | `crates/buzz-relay/src/main.rs:139` | `buzz_audit_enabled` set to 1.0/0.0 |
| Queue | `crates/buzz-relay/src/state.rs:654` | bounded `mpsc::channel::<NewAuditEntry>(1000)`; `audit_tx: Option<Sender<...>>` (`state.rs:555`) |
| Producer (events) | `crates/buzz-relay/src/handlers/event.rs:540-577` | uses `send().await` (backpressure, not drop); on closed channel logs `error!` and increments `buzz_audit_send_errors_total` (`:575-577`) |
| Producer (media) | `crates/buzz-relay/src/api/media.rs:422-442` | same pattern; upload still returns `Ok` even if the audit send fails (`media.rs:443`) |
| Worker | `crates/buzz-relay/src/state.rs:657-690` | single task; select over `recv()` and a `CancellationToken`; on cancel closes the receiver and drains buffered entries |
| Failure handling | `crates/buzz-relay/src/state.rs:1199-1207` | `audit.log(entry)` error → `buzz_audit_log_errors_total` + `tracing::error!`; **no retry, no dead-letter**; success → `buzz_audit_log_seconds` histogram |
| Shutdown | `crates/buzz-relay/src/state.rs:632-636`, `:680-689` | `AuditShutdownHandle::drain()` flushes queued entries; a timeout path logs "Audit worker did not drain in time — exiting anyway" (`state.rs:1190-1191`) |

Consequences visible in code: a DB outage causes queued entries to be lost after one
failed attempt (`state.rs:1201-1203`), and because `log` is the only path that assigns
`seq`, a lost entry leaves **no gap** in the chain — the next successful append simply
takes the next `seq`. The chain therefore stays verifiable while being incomplete; the
crate offers no way to detect that an entry was dropped.

### Other repo touch points

- `crates/buzz-admin/Cargo.toml:20` declares the dependency, but grep for `audit` in
  `crates/buzz-admin/src` returns nothing — no operator CLI surface consumes
  `verify_chain`/`get_entries` today.
- `migrations/0023_push_match_gate.sql:21` references the `'buzz_audit:'` advisory-lock
  namespace in a comment about lock families.
- `crates/buzz-test-client/tests/conformance_multitenant.rs:2665-2710` documents that
  audit is deliberately *not* black-box testable over the wire and defers to this
  crate's own tests.
- `crates/buzz-conformance/Cargo.toml:19` explicitly excludes `buzz-audit` from the
  conformance checker's dependency set.


## Module: buzz-media (`crates/buzz-media`)

### Aspect: Integrations

### 1. Dependency inventory (`crates/buzz-media/Cargo.toml:10-35`)

| Crate | Version / spec | Purpose in this crate |
|---|---|---|
| `buzz-core` | workspace | `tenant::{CommunityId, TenantContext, normalize_host}` (`crates/buzz-media/src/storage.rs:6`, `crates/buzz-media/src/auth.rs:170`) |
| `nostr` | workspace | `Event`, `PublicKey`, `Timestamp`, `ToBech32` for Blossom auth + records (`crates/buzz-media/src/auth.rs:33`, `crates/buzz-media/src/upload_record.rs:145`) |
| `s3` = `rust-s3` | **0.37**, `default-features = false`, features `tokio-rustls-tls`, `fail-on-err`, `tags` (`crates/buzz-media/Cargo.toml:24`); resolved **0.37.2** (`Cargo.lock:7432-7434`) | all object storage |
| `infer` | **0.19** | magic-byte MIME sniffing (`crates/buzz-media/src/validation.rs:239`, `:176`) |
| `image` | **0.25**, `default-features = false`, features `jpeg`, `png`, `gif`, `webp` | full decode + thumbnail JPEG encode (`crates/buzz-media/src/thumbnail.rs:26-32`) |
| `imagesize` | **0.14** | header-only dimension parse for the pixel-bomb guard (`crates/buzz-media/src/validation.rs:270`) |
| `blurhash` | **0.2** | 4×3 blurhash from the thumbnail (`crates/buzz-media/src/thumbnail.rs:36-37`) |
| `mp4` | **0.14** | MP4 header/track parsing (`crates/buzz-media/src/validation.rs:307-386`) |
| `sha2` + `hex` | workspace | SHA-256 content addressing (`crates/buzz-media/src/upload.rs:4`, `:84`, `:397`) |
| `tempfile` | **3** | `NamedTempFile` staging for streamed video (`crates/buzz-media/src/upload.rs:307`) |
| `tokio` | workspace | async fs/IO, `spawn_blocking` (`crates/buzz-media/src/upload.rs:79`, `:410`, `:418`) |
| `tokio-util` | **0.7**, feature `io` | `StreamReader` to adapt the axum body stream (`crates/buzz-media/src/upload.rs:325`) |
| `futures-util` / `futures-core` | **0.3** | `StreamExt::map`, stream trait bounds (`crates/buzz-media/src/storage.rs:141`, `crates/buzz-media/src/upload.rs:298`) |
| `bytes` | **1** | `Bytes` bodies / chunks (`crates/buzz-media/src/upload.rs:3`) |
| `axum` | workspace | only `StatusCode`/`IntoResponse` (`crates/buzz-media/src/error.rs:3-4`) and `http::HeaderName` validation (`crates/buzz-media/src/config.rs:118`) |
| `ulid` | **1** | upload-record ids (`crates/buzz-media/src/upload_record.rs:150`) |
| `uuid` | workspace | community UUID parsing in key classification (`crates/buzz-media/src/bucket_index.rs:22`, `:112-127`) |
| `chrono` | workspace | `Utc::now().timestamp()` for `uploaded_at` (`crates/buzz-media/src/upload.rs:113`, `:132`) |
| `serde` / `serde_json` | workspace | sidecar + record JSON (`crates/buzz-media/src/storage.rs:199`, `:218`) |
| `thiserror` | workspace | `MediaError`, `SweepError` (`crates/buzz-media/src/error.rs:7`, `crates/buzz-media/src/bucket_index.rs:340`) |
| `tracing` | workspace | 3 log sites (`crates/buzz-media/src/upload.rs:135`, `crates/buzz-media/src/error.rs:135`, `:155`) |

Dev-dependencies: `tokio` with `test-util` only (`crates/buzz-media/Cargo.toml:33-34`). No `mockall`, no `wiremock`, no `testcontainers`.

---

### 2. S3 client configuration

| Aspect | Behaviour | file:line |
|---|---|---|
| Region/endpoint | `Region::Custom { region: config.s3_region, endpoint: config.s3_endpoint }` — always a custom region, even for real AWS | `crates/buzz-media/src/storage.rs:35-38` |
| Path style | **Always** `.with_path_style()` — unconditional, not gated by a flag | `crates/buzz-media/src/storage.rs:66-68` |
| Bucket | `Bucket::new(&config.s3_bucket, region, creds)` | `crates/buzz-media/src/storage.rs:66` |
| TLS | `tokio-rustls-tls` feature (no native-tls) | `crates/buzz-media/Cargo.toml:24` |
| HTTP error strictness | `fail-on-err` feature — non-2xx responses surface as `S3Error` rather than being silently returned | `crates/buzz-media/Cargo.toml:24` |
| Signing region correctness | Documented requirement: `s3_region` must match the endpoint's region for real AWS, else SigV4 credential scope is wrong; default `us-east-1` preserves MinIO behaviour | `crates/buzz-media/src/config.rs:29-37`, `crates/buzz-media/src/config.rs:11-13` |

**MinIO compatibility** is achieved by (a) unconditional path-style addressing, (b) a custom `Region` with an explicit endpoint, and (c) tolerating a non-meaningful region value ("Defaults to 'us-east-1' to preserve MinIO/local behavior, where the value is not meaningfully checked" — `crates/buzz-media/src/config.rs:33-37`). The live round-trip test targets `http://localhost:9000` with creds `buzz_dev`/`buzz_dev_secret`, bucket `buzz-media` (`crates/buzz-media/tests/static_creds_minio.rs:22-34`).

---

### 3. Credential sources

Two mutually exclusive modes, chosen by whether both static keys are non-empty (`crates/buzz-media/src/storage.rs:39-65`):

| Case | Behaviour | file:line |
|---|---|---|
| both keys non-empty | `Credentials::new(Some(access), Some(secret), None, None, None)` — static keys, no environment/metadata access | `crates/buzz-media/src/storage.rs:44-50` |
| both keys empty | `Credentials::default()` — AWS default chain: environment, shared profile, **web-identity token (IRSA on EKS, `AssumeRoleWithWebIdentity`)**, container, instance metadata | `crates/buzz-media/src/storage.rs:51-56`, documented `crates/buzz-media/src/storage.rs:25-33` |
| exactly one key set | hard error `StorageError("s3_access_key and s3_secret_key must be configured together, or both empty to use the AWS credential chain")` — never silently falls back | `crates/buzz-media/src/storage.rs:57-62`; test `crates/buzz-media/src/storage.rs:312-331` |

**Patched `aws-creds` fork** (workspace-level, applies transitively to this crate): `Cargo.toml:170-171` pins `aws-creds` to `git+https://github.com/tlongwell-block/rust-s3` rev `c9fce3620dd434c1f810101d672cf384268dbb0f` (`Cargo.lock:422-424`). The reason recorded at `Cargo.toml:163-169`: aws-creds 0.39.1 cannot read **EKS Pod Identity** credentials (`AWS_CONTAINER_CREDENTIALS_FULL_URI` + `AWS_CONTAINER_AUTHORIZATION_TOKEN_FILE`), which the relay pod requires for S3 media + git storage; the fork adopts the aws-creds portion of `durch/rust-s3#449` (FULL_URI + token-file + Authorization header, refresh-safe, with a loopback allowlist for the auth token). Marked temporary — revert when #449 lands upstream.

No credential values are logged anywhere in the crate; the only `tracing` calls are the orphan-blob warning (`crates/buzz-media/src/upload.rs:135`) and the two error-mapping logs (`crates/buzz-media/src/error.rs:135`, `:155`).

---

### 4. Media-parsing integrations (in-process)

| Library | Where it runs | Input bound before it runs |
|---|---|---|
| `infer::get` | image path (`crates/buzz-media/src/validation.rs:239`), generic path (`crates/buzz-media/src/validation.rs:176`) | full buffered body |
| `imagesize::blob_size` | image path, before full decode (`crates/buzz-media/src/validation.rs:270`) | size cap already applied (`crates/buzz-media/src/validation.rs:259-268`) |
| `image::load_from_memory` (full pixel decode) | `generate_image_metadata_sync`, inside `spawn_blocking` (`crates/buzz-media/src/thumbnail.rs:26`, `crates/buzz-media/src/upload.rs:518-522`) | byte cap + 25 MP pixel cap |
| `image` thumbnail + JPEG encode | `crates/buzz-media/src/thumbnail.rs:30-32` | as above |
| `blurhash::encode` | `crates/buzz-media/src/thumbnail.rs:36-37` — failure swallowed via `unwrap_or_default()` | operates on the ≤320px thumbnail |
| `mp4::Mp4Reader::read_header` | `validate_video_file`, inside `spawn_blocking` (`crates/buzz-media/src/validation.rs:307`, `crates/buzz-media/src/upload.rs:416-419`) | byte cap, ISO-BMFF check, moov-order scan, box allowlist walk |

Custom hand-rolled parsers (no external crate): JPEG marker walk, PNG chunk walk, WebP RIFF walk, GIF block walk, MP4 box walk, top-level atom scan — `crates/buzz-media/src/validation.rs:502-928`.

---

### 5. Retry / timeout behaviour

| Aspect | Finding | file:line |
|---|---|---|
| Retries on S3 failure | **None** in this crate — every storage call maps the first error straight to `MediaError` | `crates/buzz-media/src/storage.rs:73-265` |
| Timeouts on S3 calls | **None set** in this crate; whatever `rust-s3`/`reqwest` defaults apply | `crates/buzz-media/src/storage.rs:34-70` (no timeout config on the client) |
| Sweep timeout | Only a *variant* to represent it — `SweepError::Timeout(Duration)`, constructed by the relay's sweep task which wraps the fold in `tokio::time::timeout` | `crates/buzz-media/src/bucket_index.rs:349-357` |
| Sweep object cap | Enforced by the caller-supplied `cap`, checked before folding each page | `crates/buzz-media/src/bucket_index.rs:394-398` |
| Listing page size | Caller-supplied `max_keys`, bounds one HTTP response only | `crates/buzz-media/src/storage.rs:236-241` |

---

### 6. Error handling on storage failures

| Situation | Result | file:line |
|---|---|---|
| Any `S3Error` | `MediaError::StorageError(e.to_string())` via `From` | `crates/buzz-media/src/error.rs:94-98` |
| 404 on `get`/`get_range` | `MediaError::NotFound` (special-cased on `HttpFailWithBody(404, _)`) | `crates/buzz-media/src/storage.rs:106-110`, `:119-123` |
| 404 on `head`/`head_with_metadata` | `Ok(false)` / `Ok(None)` — not an error | `crates/buzz-media/src/storage.rs:150-154`, `:168-174` |
| 404 on `get_stream` | checked via `response.status_code == 404` (different mechanism than the others) | `crates/buzz-media/src/storage.rs:136-139` |
| Sidecar read failure vs absence | Deliberately collapsed to `None` in `read_sidecar_mime` so a cross-community request cannot distinguish a foreign blob from a missing one | `crates/buzz-media/src/storage.rs:222-233` |
| Sidecar JSON parse failure | `MediaError::StorageError` (via `From<serde_json::Error>`) | `crates/buzz-media/src/error.rs:100-104` |
| `StorageError`/`Io`/`Internal` → HTTP | logged at `error` level, response body flattened to `"internal error"` | `crates/buzz-media/src/error.rs:154-158` |
| Blob PUT succeeded but metadata failed | Error propagates; blob intentionally left orphaned (no compensating delete) | `crates/buzz-media/src/upload.rs:131-141` |
| Upload-record PUT failed | Error propagates and the upload fails **before** the sidecar publish, so media is never served unscanned | `crates/buzz-media/src/upload.rs:154-172`, `crates/buzz-media/src/upload_record.rs:132-138` |
| `spawn_blocking` join failure | `MediaError::Internal` | `crates/buzz-media/src/upload.rs:87-88`, `:414`, `:418-419`, `:523-524` |
| Axum body-limit error inside the video stream | Detected by matching three Display substrings (`length limit`, `body limit`, `LengthLimitError`), converted to `ErrorKind::WriteZero` → `FileTooLarge` (413 instead of 500) | `crates/buzz-media/src/upload.rs:328-341`, `crates/buzz-media/src/upload.rs:358-366` |

---

### 7. Consumers / integration boundary

| Consumer | How it integrates | file:line |
|---|---|---|
| `buzz-relay` HTTP handlers | Owns routes `PUT /upload`, `PUT /media/upload`, `GET|HEAD /media/{sha256_ext}` and the `RequestBodyLimitLayer` | `crates/buzz-relay/src/router.rs:33-45` |
| `buzz-relay` upload dispatch | Sniffs 4096 bytes, uses `buzz_media::looks_like_iso_bmff` to route to the video pipeline, else buffers and picks image vs generic-file path | `crates/buzz-relay/src/api/media.rs:47-51`, `:317-399` |
| `buzz-relay` read path | `read_sidecar_mime`, `get_stream`, `get_range`, and `buzz_media::serve_inline` for `Content-Disposition` | `crates/buzz-relay/src/api/media.rs:633-740` |
| `buzz-relay` config | Constructs `MediaConfig` from env; also depends on `rust-s3` 0.37 directly | `crates/buzz-relay/src/config.rs:614-657`, `crates/buzz-relay/Cargo.toml:65` |
| `buzz-relay` storage sweep | Supplies the `fetch_page` closure over `MediaStorage::list_page` | `crates/buzz-media/src/bucket_index.rs:11-14` (documented), `crates/buzz-media/src/storage.rs:235-241` |
| `buzz-moderation` (external consumer) | Triggers on S3 `ObjectCreated` under `_uploads/` and parses `UploadRecord` instead of HEADing blobs | `crates/buzz-media/src/upload_record.rs:29-48` |
| `buzz-test-client` | Also depends on `rust-s3` 0.37 for E2E media tests | `crates/buzz-test-client/Cargo.toml:38` |


## Module: buzz-workflow (`crates/buzz-workflow`)

### Aspect: Integrations

---

### 1. Dependency inventory (`Cargo.toml:10-29`)

| Dependency | Version / source | Used for | Evidence |
|---|---|---|---|
| `buzz-core` | workspace | `CommunityId`, `StoredEvent`, `kind::{event_kind_u32, is_workflow_execution_kind, KIND_REACTION, KIND_STREAM_MESSAGE, KIND_STREAM_MESSAGE_DIFF}`, `network::is_private_ip` | `lib.rs:47-48`, `lib.rs:956`, `executor.rs:768` |
| `buzz-db` | workspace | `Db` handle, `workflow::RunStatus`, `WorkflowRecord`, `DbError` conversion | `lib.rs:49-50`, `error.rs:62-66` |
| `hex` | workspace | encode `owner_pubkey` for attribution | `executor.rs:558` |
| `serde` / `serde_json` / `serde_yaml` | workspace | schema derive, canonical JSON, YAML parsing | `schema.rs:7`, `schema.rs:263-266` |
| `dashmap` | workspace | `last_fired` interval anchors | `lib.rs:53`, `lib.rs:87` |
| `moka` | workspace | `moka::sync::Cache` for the per-channel workflow lookup | `lib.rs:104`, `lib.rs:118-121` |
| `evalexpr` | `"11"` (pinned major, direct dep — not workspace) | `if:`/filter condition evaluation, custom function registration | `executor.rs:17`, `executor.rs:224-316` |
| `cron` | `"0.16"` (direct dep) | cron expression parse + `after()` iteration | `schema.rs:239`, `lib.rs:691-694` |
| `nostr` | workspace | `PublicKey::from_hex`, `ToBech32` for the `npub` filter; test event builders | `executor.rs:18`, `executor.rs:194-198` |
| `uuid` | workspace | run/workflow ids, approval token generation, channel override parsing | `executor.rs:698-700`, `executor.rs:481` |
| `chrono` | workspace | `DateTime<Utc>` scheduling arithmetic | `lib.rs:52`, `lib.rs:692` |
| `tokio` | workspace | `Semaphore`, `spawn`, `spawn_blocking`, `time::{sleep, timeout}` | `lib.rs:54`, `executor.rs:370-373`, `executor.rs:1140` |
| `tracing` | workspace | structured logs throughout | `executor.rs:20` |
| `thiserror` | workspace | `WorkflowError`, `ActionSinkError` | `error.rs:3`, `action_sink.rs:12` |
| `reqwest` | workspace, **optional** (`features.reqwest = ["dep:reqwest"]`) | `call_webhook`, `add_reaction` | `Cargo.toml:27-29`, `executor.rs:781`, `executor.rs:888` |

No `sqlx` dependency — all Postgres access is via `buzz_db::Db` methods.

---

### 2. Outbound HTTP — `call_webhook`

| Concern | Implementation | Line |
|---|---|---|
| Client crate | `reqwest::Client`, built **per request** (required because `.resolve()` pins DNS per host) | `executor.rs:800-815` |
| Total timeout | `Duration::from_secs(10)` | `executor.rs:807` |
| Redirect policy | `reqwest::redirect::Policy::none()` — explicitly disabled so a redirect cannot bypass the SSRF check | `executor.rs:812` |
| Proxy bypass | `.no_proxy()` — **added after the original analysis**; without it a configured system proxy would do its own hostname resolution and bypass the pinned `safe_ip` | `executor.rs:810` |
| DNS pinning | `.resolve(host, SocketAddr::new(safe_ip, port))` with the IP validated by `check_ssrf`, closing the DNS-rebinding TOCTOU window | `executor.rs:813` |
| Method | `reqwest::Method::from_bytes(method)`; default `"POST"` when `method` is absent | `executor.rs:621`, `executor.rs:817-818` |
| Headers | caller-supplied map applied verbatim (names untemplated, values templated) | `executor.rs:822-826` |
| Body | raw string body, no content-type set automatically | `executor.rs:828-830` |
| Response cap | `WEBHOOK_MAX_RESPONSE_BYTES = 1024 * 1024` (1 MiB), enforced by chunked reads (`resp.chunk()`) with early abort | `executor.rs:778`, `executor.rs:841-863` |
| Response decode | `String::from_utf8_lossy` → `{status, body}` JSON | `executor.rs:865-868` |
| Port default | `port_or_known_default()`, fallback `80` — no scheme restriction, so plain `http://` targets are allowed despite the schema doc saying "must be a public HTTPS endpoint" (`schema.rs:120`) | `executor.rs:796-798` |
| Error mapping | every failure → `WorkflowError::WebhookError(String)` | `executor.rs:786`, `executor.rs:833-836`, `executor.rs:846-848` |

SSRF guard (`check_ssrf`, `executor.rs:745-776`): resolves `host:port` through the OS resolver on `spawn_blocking`; rejects on resolver error, on zero addresses, and if **any** returned address satisfies `buzz_core::network::is_private_ip`; returns `addrs[0]` for pinning. `is_private_ip` (`crates/buzz-core/src/network.rs:46-95`) covers IPv4 loopback/private/link-local/`0.0.0.0/8`/broadcast, CGNAT `100.64.0.0/10`, benchmarking `198.18.0.0/15`, and for IPv6 loopback/unspecified/`fc00::/7`/`fe80::/10`/`ff00::/8`/`2001:db8::/32` plus NAT64 local-use `64:ff9b:1::/48`, Teredo `2001::/32` and 6to4 `2002::/16` (added in `c26bf594`), with IPv4-mapped **and** IPv4-compatible addresses recursed through the IPv4 rules via `to_ipv4()` (`:62-65`), and NAT64 well-known `64:ff9b::/96` (`:69-73`) plus IPv4-translated `::ffff:0:0:0/96` (`:75-79`) recursed on their embedded IPv4.

---

### 3. Outbound HTTP — `add_reaction`

| Concern | Implementation | Line |
|---|---|---|
| Client | shared `LazyLock<reqwest::Client>` with a 10 s timeout, connection pooling retained | `executor.rs:874-885` |
| Target | `POST {BUZZ_RELAY_BASE_URL}/api/messages/{message_id}/reactions`, default base `http://localhost:3000` | `executor.rs:888-894` |
| Auth | `Authorization: Bearer {BUZZ_API_TOKEN}` if set, else `X-Pubkey: {BUZZ_RELAY_PUBKEY}` if set, else unauthenticated | `executor.rs:901-905` |
| SSRF guard | **none** on this path (no `check_ssrf`, no redirect policy, no response cap) — the URL comes from an env var rather than workflow YAML | `executor.rs:888-933` |
| Failure handling | non-2xx ⇒ `WebhookError` including the response body; body parse failure falls back to `{"raw": <text>}` | `executor.rs:914-922` |
| Reachability | the relay registers no `/api/messages/*` route (`crates/buzz-relay/src/router.rs:39-125`), so this call cannot succeed against the current relay |

---

### 4. Postgres via `buzz-db`

Access is exclusively through `buzz_db::Db` methods; no SQL is written in this crate.

| Db method | Purpose | Call site |
|---|---|---|
| `list_enabled_channel_workflows` | per-event workflow lookup (behind moka cache) | `lib.rs:301-306` |
| `list_all_enabled_workflows` | cron scan | `lib.rs:436` |
| `create_workflow_run` | run row for event + cron paths | `lib.rs:346-355`, `lib.rs:592-600` |
| `update_workflow_run` | `Running` / `Completed` / `Failed` transitions and trace writes | `executor.rs:985-994`, `executor.rs:1047-1056`, `lib.rs:201-215`, `lib.rs:220-238`, `lib.rs:244-261` |
| `get_workflow_run` | resolve `workflow_id` for `send_message`; read existing trace on resume | `executor.rs:537-543`, `executor.rs:1037` |
| `get_workflow` | `channel_id` + `owner_pubkey` for `send_message` | `executor.rs:546-552` |
| `claim_scheduled_workflow_fire` | cross-pod at-most-once cron claim | `lib.rs:547-568` |
| `latest_scheduled_workflow_fire` | restart anchor for interval schedules | `lib.rs:500-517` |
| `attach_scheduled_workflow_run` | best-effort claim→run link for audit | `lib.rs:617-628` |

DB errors convert via `From<buzz_db::error::DbError> for WorkflowError` → `Database(String)` (`error.rs:62-66`). In `finalize_run` DB failures are logged only (`lib.rs:207-213`, `:231-236`, `:256-260`); in `execute_run`/`execute_from_step` the initial `Running` write failure aborts the run (`executor.rs:995-1002`, `executor.rs:1057-1064`).

---

### 5. Relay integration (inbound)

| Integration point | Relay side | Line |
|---|---|---|
| Engine construction | `WorkflowEngine::new(db, WorkflowConfig::default())` | `crates/buzz-relay/src/main.rs:389-390`, `crates/buzz-relay/src/state.rs:1274-1276` |
| Side-effect sink | `RelayActionSink` registered via `set_action_sink` | `crates/buzz-relay/src/main.rs:594-595`, `crates/buzz-relay/src/workflow_sink.rs:13,159` |
| Scheduler | `tokio::spawn(async move { wf_cron.run().await })`, started only after the sink is wired | `crates/buzz-relay/src/main.rs:597-599` |
| Event hook | `workflow_engine.on_event(community, &stored_event)` spawned from the post-store fan-out | `crates/buzz-relay/src/handlers/event.rs:496-534` |
| Definition ingest | `WorkflowEngine::parse_yaml(&event.content)` in the workflow upsert command | `crates/buzz-relay/src/handlers/command_executor.rs:684` |
| Cache invalidation | `invalidate_channel_workflows` on upsert and on NIP-09 deletion | `crates/buzz-relay/src/handlers/command_executor.rs:787`, `crates/buzz-relay/src/handlers/side_effects.rs:2018,2039` |
| Manual trigger (kind-command) | ownership check, run creation, then `execute_from_step(..., 0, None)` + `finalize_run` | `crates/buzz-relay/src/handlers/command_executor.rs:826-936` |
| Inbound webhook | `POST /hooks/{id}` → host-bound tenant, `TriggerDef::Webhook` check, secret verification, body fields → `webhook_fields`, then `execute_from_step(..., 0, None)` | `crates/buzz-relay/src/router.rs:120`, `crates/buzz-relay/src/api/bridge.rs:1777-1911` |
| Approval grant/deny | relay handlers look up approvals by stored hash and call `resume_workflow` → `execute_from_step` | `crates/buzz-relay/src/handlers/command_executor.rs:1009-1169`, `:1244-1320` |
| Metrics | `buzz_workflow_runs_total{trigger,community}` incremented by the relay after a successful `on_event` | `crates/buzz-relay/src/handlers/event.rs:549-556` |

---

### 6. Error handling posture

| Boundary | Behaviour |
|---|---|
| Definition parse (per workflow, in `on_event` / cron) | warn + skip that workflow, other workflows continue (`lib.rs:331-337`, `lib.rs:449-459`) |
| Trigger filter error | warn + skip (fail-closed, no run created) (`lib.rs:838-845`) |
| Trigger-context serialization error | error log; `on_event` returns `Ok(())` (`lib.rs:319-326`); cron path `continue`s (`lib.rs:579-589`) |
| Run creation error | error log + `continue` to next workflow (`lib.rs:356-361`, `lib.rs:601-611`) |
| Step failure | aborts the run, `PartialProgress` preserved so the partial trace is persisted (`executor.rs:1113-1177`, `lib.rs:240-261`) |
| Spawned task | detached `tokio::spawn`; failures only surface through DB status + logs (`lib.rs:371-381`, `lib.rs:649-661`) |
| Sink not initialized | `WorkflowError::InvalidDefinition("action_sink not initialized …")` instead of a panic (`lib.rs:148-156`) |
| `set_action_sink` misuse | explicit `panic!("action_sink already initialized")` (`lib.rs:139-143`) — the only panic path in the crate outside tests |


## Module: buzz-relay — core & bootstrap (`crates/buzz-relay/src`)
### Aspect: Integrations

---

#### 1. External systems wired at startup, in order

The order below is the literal execution order of `main()` (`main.rs:83-1060`).

| Step | System | Call site | Fatal on failure? | Notes |
|------|--------|-----------|-------------------|-------|
| 1 | rustls `CryptoProvider` (ring) | `main.rs:88-90` | **yes** (`expect` panic) | Required before any TLS: `rediss://` to ElastiCache, `wss://`, S3 over TLS. Both `aws-lc-rs` and `ring` are in the tree transitively so rustls cannot auto-select (`main.rs:84-87`). |
| 2 | OTLP trace exporter (gRPC/tonic) | `telemetry.rs:85-88` via `main.rs:100` | **no** | Skipped entirely when `OTEL_EXPORTER_OTLP_ENDPOINT` is unset (`telemetry.rs:80-83`). A build failure returns `ExporterBuildFailed` and is logged as `warn` after the subscriber is up (`main.rs:118-120`). |
| 3 | Prometheus exporter HTTP listener | `metrics.rs:73`, spawned `metrics.rs:146` | **yes** (`expect` at `metrics.rs:143/145`) | Binds `0.0.0.0:metrics_port` (default 9102). Panics if the port is taken or a recorder already exists. |
| 4 | Postgres — main pool (+ optional read replica) | `main.rs:151-155` | **yes** | `DbConfig { database_url, read_database_url, ..default }` (`main.rs:145-149`) ⇒ max 20 / min 2 / acquire 3 s / lifetime 1800 s / idle 600 s (`crates/buzz-db/src/lib.rs:247-257`). Replica pool gets the *same* sizing (`crates/buzz-db/src/lib.rs:381-386`). |
| 5 | Postgres — migrations | `main.rs:164-168` | **yes**, but only runs when `BUZZ_AUTO_MIGRATE` is truthy (`main.rs:161-172`) | |
| 6 | Postgres — partition pre-creation (3 ahead) | `main.rs:173-176` | **no** — `error!` only | |
| 7 | Postgres — replica freshness fence probe | `main.rs:186-196` | **no** — `error!`, fence stays closed, all cursor reads go to the writer | Deliberately after step 5 so the commit-time floor guard is checked against the live schema (`main.rs:177-183`). |
| 8 | Postgres — community/owner/allowlist/d_tag bootstrap | `main.rs:220-320` | conditional: fatal iff `require_relay_membership` (steps for community, backfill, owner); `backfill_d_tags` always non-fatal | |
| 9 | Postgres — **audit pool** (separate) | `main.rs:325-329` | **yes** | `PgPoolOptions::new().max_connections(5).min_connections(1)`. No acquire timeout, lifetime, or idle timeout set. Only created when `audit_enabled`. |
| 10 | Redis — deadpool pool | `main.rs:336-341` | **yes** | `deadpool_redis::Config::from_url(redis_url)`, `PoolConfig::new(config.redis_pool_size)` (default 16). Cloned once for the readiness handler (`main.rs:333`, comment: cheap Arc clone). |
| 11 | Redis — `PubSubManager` (dedicated connection) | `main.rs:343-347` | **yes** | |
| 12 | Redis — 3 subscriber tasks | `main.rs:350-367` | n/a (spawned) | events, cache invalidation, conn-control. |
| 13 | Postgres — **search pool** (third pool) | `main.rs:378-382` | **yes** | `PgPoolOptions::new().connect(search_db_url)` — **no sizing knobs set at all**, sqlx defaults apply. Prefers `read_database_url` when set (`main.rs:373-377`). |
| 14 | Workflow engine (in-process, DB-backed) | `main.rs:389-390` | n/a | `WorkflowConfig::default()` — no env-driven workflow config is read here. |
| 15 | Relay signing keypair | `main.rs:392-414` | **yes** (`panic!` at `main.rs:409` when `require_auth_token` and no key) | Dev fallback uses hard-coded `0x…01` with a `warn` (`main.rs:396-408`). |
| 16 | S3 / MinIO — media storage | `main.rs:415-421` | **yes** | `config.media.validate()` then `MediaStorage::new`. |
| 17 | S3 / MinIO — git object store | inside `AppState::new`, `state.rs:694-701` | **yes** (`expect`, `state.rs:701`) | Reuses the same `media.s3_*` credentials/bucket/region. The `expect` message asserts media storage already validated the same config — true only because step 16 precedes `AppState::new`. |
| 18 | Local filesystem — git pack cache | `state.rs:702-709` | **yes** (`expect`, `state.rs:708`) | Directory itself was already `create_dir_all`'d during config load (`config.rs:390-397`). |
| 19 | Redis — NIP-98 replay guard | `state.rs:710-711` | n/a (lazy, uses the pool) | Doc is explicit: must stay Redis-backed and community-keyed; process-local caching would break cross-pod replay freshness (`state.rs:576-581`). |
| 20 | Redis — admission rate limiter | `state.rs:712` | n/a (lazy) | `RedisRateLimiter::new(redis_pool.clone())` — the real implementation lives at `crates/buzz-pubsub/src/rate_limiter.rs:88-99`. |
| 21 | Inter-relay QUIC mesh (UDP bind + Redis registry) | `main.rs:442-451` | **yes when enabled** (`?` at `main.rs:451`) | Off by default: `boot_mesh` returns `None` ⇒ nothing bound/published/spawned. Consumers wired at `main.rs:455-459` **before** the handle is published to `AppState.mesh`. |
| 22 | S3 / MinIO — A3 conformance probe | `main.rs:466-503` | **yes** | Runs by default (opt-out `BUZZ_GIT_CONFORMANCE_PROBE=false`). Races `BUZZ_GIT_PROBE_WRITERS` (default 32) writers over `BUZZ_GIT_PROBE_ROUNDS` (default 3) rounds against the pointer CAS. A backend that cannot do linearizable conditional writes aborts startup. |
| 23 | APNs push gateway (HTTPS, outbound) | matcher/worker spawned `main.rs:686-691`; timeout applied `push_runtime.rs:314` | **no** — the workers simply are not spawned when `push_gateway_delivery_url` is `None` | URL must be exactly `https://…/v1/deliveries/apns` (`config.rs:342-360`). Default when unset is the hard-coded `https://push.buzz.xyz/v1/deliveries/apns` (`config.rs:332`, `config.rs:752-757`) — **outbound push integration is on by default**. |
| 24 | TCP listeners: health then app | `main.rs:1116` then `main.rs:1157` | **yes** for both | Health binds first so probes answer as early as possible. |
| 25 | Unix domain socket (optional) | `main.rs:1178-1187` | **yes when configured** | Pre-existing non-socket file at the path is fatal (`main.rs:1168-1172`). |

#### 2. Postgres connection accounting

| Pool | Created | max | min | Instrumented? |
|------|---------|-----|-----|---------------|
| Main writer | `main.rs:151` → `crates/buzz-db/src/lib.rs:361` | 20 | 2 | yes (`main.rs:956-959`) |
| Read replica (if `READ_DATABASE_URL`) | `crates/buzz-db/src/lib.rs:363` | 20 | 2 | yes (`main.rs:962-966`) |
| Audit | `main.rs:325-329` | 5 | 1 | **no** |
| Search | `main.rs:378-382` | sqlx default (unset) | sqlx default | **no** |

**Verified doc drift:** `DbConfig::default()`'s doc says "At 20 main + 5 audit = 25/pod, four relay pods fit within the PG limit" (`crates/buzz-db/src/lib.rs:244-246`). `main.rs` opens a **third** pool for search (`main.rs:378-382`) whose size is never set, and a replica pool at the same 20. The actual per-pod ceiling is 20 + 5 + search-default, plus 20 more with a replica — not 25. The "four pods within PG max_connections=100" arithmetic therefore does not hold.

Note also: the search pool connects to the **replica** when one is configured (`main.rs:373-377`), deliberately bypassing the freshness fence because search is lag-tolerant (`main.rs:369-372`).

#### 3. Failure behaviour if each dependency is unavailable

| Dependency | At startup | At runtime |
|-----------|-----------|-----------|
| Postgres (main) | abort with `DB connection failed` (`main.rs:151-155`) | `/_readiness` → 503 with `{"postgres": false}` (`router.rs:352-373`); host→community binding fails closed ⇒ every new WS gets a generic 404 (`router.rs:288-299`); NIP-11 still served, `icon` omitted (`nip11.rs:277-286`) |
| Postgres (audit) | abort (`main.rs:329`) | per-entry `error!` + `buzz_audit_log_errors_total`; worker survives (`state.rs:1200-1206`); events are still accepted (audit is off the critical path) |
| Postgres (search) | abort (`main.rs:382`) | search REQ/`POST /query` fail; not surfaced in readiness |
| Redis | abort on pool creation or `PubSubManager::new` (`main.rs:336-347`) | `/_readiness` → 503 with `{"redis": false}` (`router.rs:353-373`); **rate-limit admission fails closed and denies** (`admission.rs:29-33`); NIP-98 replay guard unavailable ⇒ NIP-98 routes fail; cross-pod fan-out / cache invalidation / ban propagation silently stop |
| S3 / MinIO | abort — media storage init (`main.rs:419-421`), git store (`state.rs:701`), and A3 probe (`main.rs:488-495`) are all fatal | media + git request failures; the storage sweep is the only path with an explicit kill switch for missing `s3:ListBucket` (`main.rs:1436-1441`) |
| OTLP collector | never fatal (`telemetry.rs:80-90`) | batch exporter drops spans; shutdown error is `warn` (`main.rs:1054-1057`) |
| Mesh peers / Redis registry | fatal **only when `BUZZ_MESH=on`** (`main.rs:451`) | `/_mesh` reports peer state; fence-rejection counters exposed |
| APNs push gateway | not contacted at startup | per-request timeout `push_gateway_timeout` (default 2000 ms, `config.rs:759-773`) applied at `push_runtime.rs:314` |

#### 4. Retry / backoff inventory

There is **no exponential backoff anywhere in this file group**. Every retry is a fixed-interval loop or a single attempt.

| Path | Strategy | Cite |
|------|----------|------|
| Postgres connect (all 3 pools) | none — single attempt, abort | `main.rs:151`, `main.rs:329`, `main.rs:382` |
| Redis pool | deadpool acquires lazily per use; no relay-side retry | `main.rs:336-341` |
| Channel-event reconciliation | fixed 5 s × 24 attempts (≈2 min), then gives up silently | `main.rs:570-590` |
| NIP-43 snapshot reconciliation | fixed interval, default 60 s, indefinite | `main.rs:517-545` |
| Ephemeral reaper | fixed interval, default 60 s; failed tick `continue`s | `main.rs:609-620` |
| Reminder scheduler | fixed interval, default 10 s; failed publish releases the claim so the next tick retries — **unless the release itself fails, in which case the reminder is never retried** | `main.rs:701-798` |
| Community revalidator | fixed interval, default 30 s clamped to `1..=300`; a failed per-community lookup simply waits for the next tick | `main.rs:882-890`, `state.rs:1076-1087` |
| Pool metrics | fixed interval, default 10 s, `.max(1)` | `main.rs:945-949` |
| Usage metrics | fixed interval, default 300 s, `.max(5)`, first tick jittered by `rand % interval`; `MissedTickBehavior::Skip` | `main.rs:1009-1022` |
| Cross-pod publishes (cache invalidation, ban) | fire-and-forget, **no retry**; backstopped by ≤10 s cache TTL and the durable DB ban row respectively | `state.rs:964-978`, `state.rs:1039-1053` |
| Community-archive publish | **awaited** (not fire-and-forget) so the archive API can offer a retryable response; plus a periodic durable revalidation backstop | `state.rs:1055-1071`, `main.rs:869-890` |
| Redis broadcast consumers | `Lagged` is counted and tolerated; `Closed` **breaks the loop permanently** | `main.rs:834-843`, `:864-873`, `:925-934` |
| Storage sweep | single-flight with `BUZZ_STORAGE_SWEEP_TIMEOUT_SECS`; last cached snapshot re-emitted | `main.rs:1442-1477`, `storage_sweep.rs:56-72` |
| A3 conformance probe | `race_width` × `race_rounds` races, single overall attempt, fatal | `main.rs:472-502` |

#### 5. Crate-level dependency integration (`Cargo.toml`)

Internal crates depended on: `buzz-core`, `buzz-conformance`, `buzz-db`, `buzz-auth`, `buzz-pubsub`, `buzz-audit`, `buzz-search`, `buzz-relay-mesh`, `buzz-sdk`, `buzz-workflow` (with `reqwest` feature), `buzz-media` (`Cargo.toml:19-27`, `:66-68`).

Notable third-party pins that are **not** workspace-managed (each pinned locally):
- `rustls 0.23` with `default-features = false, features = ["ring","std"]` (`Cargo.toml:57`) — deliberate, paired with `main.rs:88-90`.
- `s3 = { version = "0.37", package = "rust-s3", features = ["tokio-rustls-tls","fail-on-err","tags"] }` (`Cargo.toml:61`).
- `async-trait 0.1` (`Cargo.toml:28`) — but `tenant.rs:29-30` explicitly says it uses native `async fn` in trait "no `async-trait` dependency", and `HostResolver` (`tenant.rs:31`) does. So the `async-trait` dep is used elsewhere in the crate, not by the tenancy seam.
- `base64 0.22`, `tempfile 3`, `bytes 1`, `infer 0.19`, `pulldown-cmark 0.13.4`, `async-compression 0.4.42` (`Cargo.toml:60`, `:62-64`, `:78-79`).
- `tokio-util` needs the extra `io` feature for git stdout streaming (`Cargo.toml:31-34`).

Dev-dependencies pull **two git-sourced crates from an external GitHub org**: `mesh-llm-sdk` and `mesh-llm-host-runtime`, both `git = "https://github.com/Mesh-LLM/mesh-llm.git", tag = "v0.73.1"` (`Cargo.toml:84-85`). These are test-only but make `cargo test -p buzz-relay` require network access to a third-party repo.

`buzz-relay` version is deliberately independent of the workspace: `0.2.0` (`Cargo.toml:7`, rationale `Cargo.toml:4-6`) and is what `NIP-11.version` reports (`nip11.rs:161`).

#### 6. Nothing depends on `buzz-relay` as a library

Verified: no crate in the workspace and nothing under `desktop/src-tauri/**` declares `buzz-relay` as a dependency (`buzz-admin`, `buzz-conformance`, `buzz-relay-mesh`, `git-sign-nostr` mention the name only in comments). The single external textual reference is a doc mention of `buzz_relay::handlers::event::tests::`. Consequence: the crate's entire `pub` surface exists solely for `main.rs` and in-crate consumers, and `lib.rs:53-55`'s three re-exports have no consumer at all.


## Module: buzz-relay — WebSocket protocol & subscriptions (`crates/buzz-relay/src`)
### Aspect: Integrations

---

#### 1. `buzz-db` — every call from this group

| Call | Caller | Purpose | Failure handling |
|---|---|---|---|
| `is_community_active(community)` | `connection.rs:133` (closure passed to `run_registered_community_connection`) | durable community revalidation after socket registration | anything other than `Ok(true)` cancels the socket (`state.rs:149-152`) — fail closed |
| `moderation_restriction_state(community, pubkey)` | `auth.rs:119-130` | ban seam on the authenticated pubkey | `Err` → `BanOutcome::DbError` → deny with `error: internal …` |
| `moderation_restriction_state(community, owner)` | `auth.rs:136-156` | NIP-OA owner ban cascade | same |
| `is_pubkey_allowed(community, pubkey)` | `auth.rs:189-212` | allowlist gate (only when enabled **and** `Nip42`) | `Err` → `false` → deny (fail closed, `auth.rs:195-201`) |
| `is_agent_owner(community, agent, owner)` | `event.rs:1021-1025` | observer-frame publish authorisation | `Err` → `OK false "error: internal server error"` |
| `is_member(community, channel, pubkey)` | `req.rs:145-149` (REQ up-front confirm), `count.rs:126-141` (per-filter confirm) | uncached membership confirmation to repair a stale cache-negative | `Err` → `CLOSED "error: database error"` |
| `query_events(&EventQuery)` | `req.rs:313` (one per filter, concurrency 4), `count.rs:187`, `:256` (fallback) | historical/candidate rows | REQ → `EOSE` + return (**not** `CLOSED`, `req.rs:323-329`); COUNT → `CLOSED "error: {e}"` |
| `count_events(&EventQuery)` | `count.rs:176`, `:246` | exact fast-path count | `CLOSED "error: {e}"` |
| `get_events_by_ids(community, ids)` | `req.rs:641-645` | batch fetch of FTS hit ids | `warn!` + `break` out of the page loop (partial results, then EOSE) |
| `communities_of_channels(&[Uuid])` | `req.rs:355`, `:648` | conformance row-community projection **only** | `Err` → `warn!` + empty map; the emit's own guard-rail handles it |
| `db.clone()` | `req.rs:303` | cloned into the `buffered` stream | — |

Indirect (through `AppState` helpers, `state.rs`):

| Helper | Underlying | Caller | Cache |
|---|---|---|---|
| `get_accessible_channel_ids_cached` | `db.get_accessible_channel_ids` | `req.rs:94-98`, `count.rs:79-83` | moka, 10 s TTL, cap 10 000 (`state.rs:747-753`) |
| `is_member_cached` | `db.is_member` | `event.rs:209-212` (fan-out gate) | moka, 10 s TTL, cap 10 000 (`state.rs:740-746`) |
| `channel_visibility_cached` | `db.get_channel` | `event.rs:197-200` | moka, 10 s TTL — **caches only `"private"`** (`state.rs:1124-1150`) |

The read path is **not** replica-routed at this layer; `AppState.db` is a single `Db` and the optional `READ_DATABASE_URL` (`config.rs:57-59`) is handled inside `buzz-db`.

---

#### 2. `buzz-auth`

| Item | Used at |
|---|---|
| `generate_challenge()` | `connection.rs:157` |
| `AuthService::verify_auth_event(event, challenge, relay_url)` | `auth.rs:86-90` — the whole of NIP-42 verification; pure crypto |
| `AuthService::config().rate_limits` | `connection.rs:612` |
| `AuthContext` (`pubkey`, `scopes`, `channel_ids`, `auth_method`, `agent_owner_pubkey`) | stored in `AuthState::Authenticated` (`connection.rs:45`), read at `event.rs:634-654`, `req.rs:50-87`, `count.rs:37-51`, `connection.rs:604-609` |
| `AuthMethod::Nip42` | `auth.rs:191` (allowlist scoping) |
| `Scope::{MessagesRead, MessagesWrite}` | `req.rs:54`; `event.rs:681`, `:676` |
| `LimitType::{WsEvents, Messages}` | `connection.rs:619`, `:642` |
| `RateLimiter` trait (via `crate::admission::check_principal`) | `admission.rs:18-42` |
| `Nip98ReplayGuard` | not used by this group (HTTP only) |

`AuthService` is **not** consulted for authorization — only for verification and for reading rate-limit config. Every authz decision on the WS path is made in `handlers/*` against `buzz-db` and `buzz-core::kind` sets.

---

#### 3. `buzz-pubsub`

##### 3.1 Publish

| Topic | Caller | Preceded by `mark_local_event` |
|---|---|---|
| `EventTopic::Channel(ch)` | `event.rs:399` (persistent, when `stored_event.channel_id` is `Some`), `event.rs:874` (channel-scoped ephemeral) | yes (`:394`, `:824`) |
| `EventTopic::Global` | `event.rs:399` (persistent, channel-less), `event.rs:879` (channel-less ephemeral), `event.rs:1073` (observer frame) | yes (`:394`, `:852`, `:1046`) |

Every publish failure invalidates the local-echo mark before logging (`event.rs:400-405`, `:830-836`, `:858-864`, `:1052-1058`) — otherwise a failed publish would suppress a later legitimate delivery of the same id.

##### 3.2 `retain_topic` / `release_topic` lifecycle

Refcounting lives in `buzz-pubsub`: `desired_topics: HashMap<EventTopicKey, usize>`; `retain_topic` PSUBSCRIBEs only on the 0→1 transition (`buzz-pubsub/src/lib.rs:192-208`), `release_topic` schedules a debounced unsubscribe on the 1→0 transition and warns on an unmatched release (`:215-232`).

| Operation | Site | Topic |
|---|---|---|
| **retain** — after successful non-search REQ registration | `req.rs:254-257` | `topic_for_subscription(channel_id)` (`req.rs:1270-1275`) |
| **release** — the subscription that this REQ replaced | `req.rs:248-253` | `topic_for_subscription(replaced.channel_id)` |
| **release** — explicit CLOSE | `close.rs:20-25` | `topic_for_subscription(removed.channel_id)` (`close.rs:30-35`) |
| **release** — one per subscription at disconnect | `connection.rs:265-270` | `topic_for_subscription(removed.channel_id)` (`connection.rs:681-686`) |
| retain (test-only) | `event.rs:1706`, `:1687` | `EventTopic::Global`, inside `#[cfg(test)]` |

Balance audit:
- Every `register_scoped` is followed by exactly one `retain_topic`, and its `Option<RemovedSubscription>` return drives exactly one compensating `release_topic` — so a same-`sub_id` replacement is net-neutral when the scope is unchanged, and a correct swap when it changes (`req.rs:241-257`).
- `remove_connection` returns **one `RemovedSubscription` per subscription** (`subscription.rs:181-196`), and `connection.rs:265-270` releases once per element — so N subscriptions on the same topic produce N retains and N releases. Correct.
- **Three identical private copies** of `topic_for_subscription` exist (`req.rs:1270-1275`, `close.rs:30-35`, `connection.rs:681-686`). See the debt aspect.
- Search REQs never retain (they return at `req.rs:232`), so the counts stay balanced.

**Unbalanced-release risk found:** `close.rs:16` removes the entry from `conn.subscriptions` **before** `sub_registry.remove_subscription` at `:20`. Two concurrent CLOSE frames for the same `sub_id` cannot double-release, because `remove_subscription` is the one that returns `Some` and it is guarded by the DashMap entry removal (`subscription.rs:164-172`) — the second call returns `None`. Verified safe. The `conn.subscriptions` removal is not the guard.

##### 3.3 Subscribe (fan-in)

`main.rs:818-845` holds the only `subscribe_local()` consumer in production; it calls `fan_out_pubsub_event` (`event.rs:282`). Lag → `buzz_multinode_fanout_lag_total` and a warning (`main.rs:833-836`); a closed broadcast channel logs an error and ends the loop (`:840-842`) — **the loop is not restarted**, so a closed channel silently ends cross-node delivery for the process lifetime.

##### 3.4 Other pub/sub channels this group depends on

| Channel | Consumer | Effect on this group |
|---|---|---|
| cache invalidation | `main.rs:846-877` → `state.apply_cache_invalidation` | drops the membership / accessible-channels / visibility moka entries this group's gates read |
| conn control (`DisconnectPubkey`, `DisconnectCommunity`) | dispatched in `main.rs` → `conn_manager.disconnect_pubkey` / `community_connections.disconnect_community` | closes sockets owned by this group |
| presence (`set_presence` / `clear_presence`) | `event.rs:814-822`, `connection.rs:274-284` | Redis-side presence state |

---

#### 4. `buzz-search`

| Item | Used at |
|---|---|
| `SearchService::search(&SearchQuery)` | `req.rs:602-610` |
| `SearchQuery { community, q, channel_scope, kinds, authors, since, until, page, per_page, mode }` | built `req.rs:596-608` |
| `SearchMode::FullText` | `req.rs:607` — the only mode used |
| `ChannelScope::{Channels, ChannelsOrChannelLess, ChannelLessOnly}` | `req.rs:483-501`, `:580` |

Contract: search returns **event ids only** (`req.rs:637`); the full events are then fetched from Postgres (`req.rs:641-645`) and re-post-filtered (`req.rs:685-712`). This is why the sensitive-kind gates must run *before* the search branch (`req.rs:175-205`) — search hits skip the per-filter historical post-check chain by construction. A search error is non-fatal: `warn!` + `break` out of the page loop, then EOSE (`req.rs:611-616`).

---

#### 5. `buzz-conformance` (via `crate::conformance`)

| Emit | Site | Guard |
|---|---|---|
| `state_for_request(tenant, pubkey)` — builds the `AbstractState` once per REQ | `req.rs:110-116` | `None` only on malformed pubkey bytes |
| `record_req_authcheck(tracer, state, ch_id, member)` | `req.rs:141-148` | only on the DB-confirmation branch |
| `record_read_message_rows(tracer, state, per_filter_channel, &row_channels, &channel_communities)` | `req.rs:337-372` | non-search lane, per filter |
| `record_read_by_id_rows(tracer, state, None, &row_channels, &channel_communities)` | `req.rs:626-668` | search lane, per page |

Production binds `NoopTracer` (`state.rs:798`), so these are zero-cost — **except** the two `db.communities_of_channels` round-trips at `req.rs:355` and `:648`, which are issued **unconditionally** whenever `trace_state.is_some()` (i.e. always, in practice) regardless of whether the tracer is a no-op. That is a per-filter and per-search-page extra query on the hot read path with no production benefit. See the debt aspect.

No conformance emit exists on the write/fan-out side of this group; the comment at `event.rs:397-403` explains that acceptance is recorded at the ingest seam and fan-out surfaces as `ReadMessageRows` on the subscriber side.

---

#### 6. `buzz-core`

| Item | Used at |
|---|---|
| `filter::filters_match` | `subscription.rs:377`, `req.rs:372`, `:695`, `count.rs:198`, `:267` |
| `filter::reader_authorized_for_event` | **no longer called directly from these files** since `ab3af828` — the four call sites were folded into `handlers/req.rs::event_visible_to_reader` (`req.rs:1230`), reached from `req.rs:388`, `:705`, `count.rs:202`, `:271` |
| `verification::verify_event` | `event.rs:772` (ephemeral), `:927` (observer) — both on `spawn_blocking` |
| `event::StoredEvent::{new, with_received_at}` | `event.rs:292`, `:841`, `:869`, `:1060` |
| `kind::{event_kind_u32, is_ephemeral, is_workflow_execution_kind, is_command_kind, is_parameterized_replaceable}` | `event.rs:611`, `:675`, `:509-510`; `req.rs:777-782`, `:944-950` |
| `kind::{AUTHOR_ONLY_KINDS, P_GATED_KINDS, RESULT_GATED_KINDS}` | `event.rs:137`; `req.rs:1042`, `:1156`, `:1139`, `:1188`, `:1208` |
| `kind::{KIND_GIFT_WRAP, KIND_AUTH, KIND_AGENT_OBSERVER_FRAME, KIND_PRESENCE_UPDATE, KIND_DM_VISIBILITY, KIND_AGENT_TURN_METRIC, KIND_AGENT_ENGRAM, KIND_NIP43_MEMBERSHIP_LIST}` | `event.rs:659`, `:647`, `:657`, `:772`, `:438-439`; `req.rs:832`, `:1065`, `:1114` |
| `observer::{content_looks_like_nip44, OBSERVER_AGENT_TAG, OBSERVER_FRAME_TAG, OBSERVER_FRAME_TELEMETRY, OBSERVER_FRAME_CONTROL}` | `event.rs:1095-1113` |
| `tenant::{TenantContext, CommunityId}` | throughout |

---

#### 7. `buzz-audit`

Only via the bounded channel: `state.audit_tx` (capacity 1000, `state.rs:654`), written at `event.rs:574` with `send().await`. The worker (`state.rs:658-696`) does the DB write. Channel-closed → `error!` + `buzz_audit_send_errors_total` (`event.rs:574-577`); the event is **not** rejected. When `audit_enabled == false`, `audit_tx` is `None` and the enqueue short-circuits (`event.rs:571-573`).

---

#### 8. `buzz-workflow`

`workflow_engine.on_event(community, &stored_event)` is spawned from `dispatch_persistent_event_inner` (`event.rs:512-533`). The community is passed **explicitly** because `StoredEvent` does not carry it and the same channel UUID can exist in another tenant (rationale `event.rs:528-532`). Skipped for workflow-execution kinds, command kinds, relay-signed `buzz:workflow`-tagged events, and kind 1059 (`event.rs:526-534`).

---

#### 9. Slow-client / backpressure handling

Two sender surfaces, one shared counter:

| Surface | Method | Site |
|---|---|---|
| direct (handler → its own socket) | `ConnectionState::send(String)` → `send_tx.try_send(Text)` | `connection.rs:88-113` |
| fan-out (any task → any socket) | `ConnectionManager::send_to(String)` / `send_to_text_bytes(Arc<Bytes>)` → `try_send_ws_message` | `state.rs:436-438`, `:443-447`, `:449-474` |

Shared state: `backpressure_count: Arc<AtomicU8>` created at `connection.rs:164`, handed to both `ConnectionState` (`:178`) and `ConnEntry` (`:210`). Semantics (identical in both):

1. `Ok` → `store(0)` — a single successful send fully forgives accumulated strikes (`connection.rs:92`, `state.rs:456`).
2. `Full` → `fetch_add(1)+1`; `>= grace_limit` (15) → `warn!` + `buzz_ws_backpressure_disconnects_total` + `cancel()`; otherwise a graded warning (`connection.rs:95-107`, `state.rs:458-472`).
3. `Closed` → `debug!`, return `false`, **no** strike (`connection.rs:108-111`, `state.rs:473-476`).

Callers that react to a `false` return: `req.rs:398-400` and `:720-722` abandon delivery (and skip EOSE); `event.rs:81-91` counts drops into `drop_count` and logs an aggregate warning (`:249-255`, `:299-305`, `:474-479`).

Control-plane backpressure is treated as **terminal**, not graded: a full 8-slot `ctrl_tx` closes the connection in the heartbeat loop (`connection.rs:396-400`) and on a client Ping (`connection.rs:464-472`). Best-effort ctrl sends that do *not* close: ban reason (`auth.rs:177-179`), `disconnect_pubkey` reason (`state.rs:325-328`), restart close (`state.rs:359`, `:229`).

Read-side backpressure: there is none. `recv_loop` awaits `ws_recv.next()` and, for EVENT/REQ/COUNT, spawns a task under `handler_semaphore` and immediately loops (`connection.rs:530-536`, `:552-558`, `:573-579`). `AUTH` and `CLOSE` are awaited inline (`:506-508`, `:582`), which is the only inbound self-throttle.

---

#### 10. Integration failure-mode summary

| Dependency | On failure | Posture |
|---|---|---|
| Postgres — community active check | socket cancelled | fail closed |
| Postgres — ban state | deny with `error: internal …` | fail closed, cause-preserving |
| Postgres — allowlist | deny | fail closed |
| Postgres — accessible channels | `CLOSED "error: database error"` | fail closed |
| Postgres — membership confirm | `CLOSED "error: database error"` | fail closed |
| Postgres — visibility (fan-out) | recipient list emptied | fail closed |
| Postgres — membership (fan-out) | that recipient dropped | fail closed |
| Postgres — historical query | `EOSE`, subscription stays live | **fail open (silent)** |
| Postgres — COUNT | `CLOSED "error: {raw}"` | fail closed, **leaks error text** |
| Postgres — `get_events_by_ids` (search) | partial page, then EOSE | **fail open (silent)** |
| Postgres — `communities_of_channels` | empty map, trace-only | fail soft |
| Redis — admission limiter | frame rejected | fail closed |
| Redis — `publish_event` | echo mark invalidated, `warn!`, local fan-out still happens | fail open (single-node delivery only) |
| Redis — presence | ignored (`let _ =`) | fail open |
| Redis — cache-invalidation publish | spawned, warn-only | fail open, ≤10 s TTL backstop |
| Redis — broadcast lag | counted, events lost | fail open |
| Redis — broadcast closed | error logged, **loop exits permanently** | fail open |
| Search (FTS) | `warn!` + break, then EOSE | fail open (silent) |
| Audit channel closed | `error!` + counter, event still accepted | fail open |
| Workflow engine | `error!` in spawned task | fail open |


## Module: buzz-relay — event ingest & side effects (`crates/buzz-relay/src/handlers`)
### Aspect: Integrations

---

### 1. Crate dependency fan-out

| Crate | Reached from this group | How |
|---|---|---|
| `buzz-core` | all four files | kind constants + classification predicates (`ingest.rs:11-45`), `verify_event` (`:1491`), `TenantContext`, `CommunityId`, `StoredEvent`, `canonical_channel_name` (`ingest.rs:2069`, `side_effects.rs:466`) |
| `buzz-auth` | `ingest.rs:8` | `Scope` only — used by `required_scope_for_kind` (`:198-306`) and the (unreachable) gate at `:1551` |
| `buzz-db` | all except `imeta.rs` | 60+ distinct methods via `state.db` — see §2 |
| `buzz-media` | `imeta.rs` | `MediaStorage::get_sidecar` (`imeta.rs:246`, `:308`), `MediaStorage::head` (`:252`, `:290`, `:328`), `validation::mime_to_ext` (`imeta.rs:5`, used `:185`) |
| `buzz-pubsub` | `side_effects.rs` | `EventTopic` (`:26`); `publish_event` (`:701`, `:794`, `:873`); `release_topic` (`:81`) |
| `buzz-workflow` | `command_executor.rs` | `WorkflowEngine::parse_yaml` (`:719`), `TriggerDef::Webhook` match (`:738`), `WorkflowDef` deserialize (`:903`, `:1272`), `executor::execute_from_step` (`:925`, `:1317`), `engine.finalize_run` (`:934`, `:1325`), `engine.invalidate_channel_workflows` (`:783`); `side_effects.rs:2017`, `:2038` for the a-tag delete path |
| `buzz-conformance` | `ingest.rs` | `Tracer` trait (`:1414`), `TraceAction`, `EmitGuard`, `Verdict` via `crate::conformance` (`:47-50`) |
| `buzz-audit` | **not directly** | reached transitively through `dispatch_persistent_event` → `enqueue_event_created_audit` (`handlers/event.rs:359`, `:534-577`) |
| `buzz-search` | **never** | Postgres FTS is a generated `tsvector` column populated by the `insert_event` write itself; the old Typesense worker and `search_index_tx` are gone (`handlers/event.rs:501-507`). This module makes **zero** `buzz_search` calls. |
| `sqlx` | `command_executor.rs` | raw SQL — `pg_advisory_xact_lock` (`:170`), coordinate SELECT (`:176`), coordinate soft-delete (`:196`), `events` INSERT (`:196-232`). The only place in this group that bypasses the `buzz-db` API. |
| `metrics` | `ingest.rs`, `side_effects.rs`, `command_executor.rs` | 8 series — see configuration.md §4 |
| `nostr` | all | `Event`, `EventBuilder`, `Kind`, `Tag`, `Keys` |

`buzz-relay` internal reach-out: `crate::api::media::media_base_url_for_tenant`
(`ingest.rs:2238`) and `is_safe_ext` (`imeta.rs:379`); `crate::api::nip05::canonicalize_nip05`
(`side_effects.rs:1157`); `crate::api::git::{manifest, store, manifest_event}`
(`side_effects.rs:2616-2618`, `:2705`, `:2738`); `crate::protocol::RelayMessage`
(`side_effects.rs:23`); `crate::webhook_secret` (`command_executor.rs:27`);
`crate::handlers::{event, relay_admin, identity_archive, report, product_feedback,
moderation_commands, push_lease}`.

---

### 2. `buzz-db` call inventory (by concern)

| Concern | Methods called | Sites |
|---|---|---|
| Event read | `get_event_by_id`, `get_event_by_id_including_deleted`, `query_events` | `ingest.rs:361`, `:634`, `:665`, `:690`, `:790`, `:869`, `:2245`; `side_effects.rs:203`, `:552`, `:1600`, `:2114`, `:2201`, `:931`, `:2969`, `:3080` |
| Event write | `insert_event`, `insert_event_with_thread_metadata`, `insert_reaction_event_with_thread_metadata`, `replace_addressable_event`, `replace_parameterized_event` | `ingest.rs:2324`, `:2371`, `:2385`, `:2394`; `side_effects.rs:692`, `:868`, `:940`, `:2752`, `:2911`, `:3036`, `:3123`, `:3186` |
| Event delete | `soft_delete_event_and_update_thread`, `soft_delete_by_coordinate`, `soft_delete_discovery_events` | `side_effects.rs:1624`, `:2147`, `:2069`, `:1798` |
| Thread | `get_thread_metadata_by_event`, `get_thread_summary` | `ingest.rs:608`; `side_effects.rs:1616`, `:2131`, `:735` |
| Reaction | `remove_reaction_by_source_event_id`, `remove_reaction` | `side_effects.rs:2175`, `:2216` |
| Channel | `get_channel`, `create_channel`, `create_channel_with_id`, `update_channel`, `set_topic`, `set_purpose`, `archive_channel`, `unarchive_channel`, `soft_delete_channel`, `list_channels`, `open_dm`, `hide_dm`, `list_hidden_dms` | `ingest.rs:1765`, `:812`, `:2103`, `:2408`; `side_effects.rs:271`, `:545`, `:1345`, `:1372`, `:1387`, `:1416`, `:1466`, `:1485`, `:1499`, `:1789`, `:1846`, `:2952`, `:3062`; `command_executor.rs:398`, `:497`, `:534`, `:611`, `:633`, `:766` |
| Membership | `get_members`, `add_member`, `remove_member`, `is_agent_owner`, `get_agent_channel_policy` | `side_effects.rs:100`, `:298`, `:376`, `:391`, `:520`(via members), `:627`, `:647`, `:1216`, `:1275`, `:1293`, `:1526`, `:1932`; `ingest.rs:829`, `:1339`, `:2002`; `command_executor.rs:518` |
| User | `ensure_user`, `update_user_profile`, `set_channel_add_policy` | `side_effects.rs:1096`, `:1140`, `:1163`, `:1182`, `:1105`; `command_executor.rs:49` |
| Relay membership | `remove_relay_member`, `publish_nip43_membership_locked`, `nip43_membership_snapshot_needs_reconciliation`, `usage_community_hosts` | `ingest.rs:1884` (`remove_relay_member`), `:1909` (NIP-43 publish fan-out); `side_effects.rs:2866`, `:2789`, `:2777` |
| Archived identities | `list_archived` | `side_effects.rs:3009` |
| Moderation | `moderation_restriction_state` | `ingest.rs:1642` |
| Workflow | `get_workflow`, `upsert_workflow`, `create_workflow_run`, `get_workflow_run`, `update_workflow_run`, `delete_workflow_for_owner`, `find_workflow_by_owner_and_name`, `get_approval_by_stored_hash`, `update_approval_by_stored_hash` | `command_executor.rs:724`, `:775`, `:918`, `:1250`, `:1177`, `:1276`, `:1290`, `:1305`, `:1041`, `:1153`; `side_effects.rs:2000`, `:2011`, `:2026` |
| Git | `repo_name_owner`, `count_repos_for_owner`, `reserve_repo_name`, `release_repo_name` | `side_effects.rs:2470`, `:2491`, `:2500`, `:2543` |
| Transaction | `begin_transaction` | `command_executor.rs:106` |

---

### 3. Transaction boundaries and partial-failure semantics

**The core answer: ingest is *not* atomic beyond the single storage call.**

#### 3a. What *is* atomic

| Unit | Contents | Where |
|---|---|---|
| Generic event insert | `events` row + `thread_metadata` row + parent/root stubs + `reply_count`/`last_reply_at`/`descendant_count` updates | `buzz-db/src/event.rs:1004-1191`; the rationale ("could leave reply counters inconsistent if one succeeded and the other failed") is at `:1173-1177` |
| Reaction insert | `reactions` upsert + `events` row + `thread_metadata`, in a load-bearing order so an active duplicate returns before a duplicate kind:7 is stored | `buzz-db/src/event.rs:1201-1275`; ordering note `ingest.rs:2320-2323` |
| Soft delete | `events.deleted_at` + `reply_count`/`descendant_count` decrements | `soft_delete_event_and_update_thread`, called `side_effects.rs:1624`, `:2147` |
| Replaceable / NIP-33 replace | old-row supersede + new insert | inside `replace_addressable_event` / `replace_parameterized_event` |
| Relay-member removal | NotFound / IsOwner classification + delete | `remove_relay_member` (`ingest.rs:1883`) — the comment at `:1855` states it "handles both the NotFound and IsOwner cases atomically" |
| Git name reservation | `INSERT … ON CONFLICT DO NOTHING` as the TOCTOU race guard | `side_effects.rs:2500`, rationale `:2437-2447` |
| NIP-43 snapshot publish | read members + build event + replace, all inside one per-community advisory lock so a stale snapshot cannot win by arrival order | `publish_nip43_membership_locked` (`side_effects.rs:2866`), rationale `:2814-2824` |
| Command coordinate LWW | `pg_advisory_xact_lock` + head read + supersede + insert | `command_executor.rs:170-232` |

#### 3b. What is **not** atomic — the failure matrix

| Failure point | Event state | Domain state | Client sees | Evidence |
|---|---|---|---|---|
| `handle_side_effects` returns `Err` | **committed and fanned out** | not applied | `accepted: true, message: ""` | `ingest.rs:2460-2467` — `warn!` only, then `dispatch_persistent_event` at `:2513` runs unconditionally |
| 9000 `add_member` rejected by the DB's elevated-role guard (`buzz-db/src/channel.rs:385`, `:400`) | kind:9000 committed + fanned out | no membership change | success | same |
| 9002 `update_channel` fails mid-loop | kind:9002 committed; **earlier tags in the same event already applied** | partial metadata update | success | `side_effects.rs:1339-1552` — the per-tag loop uses `?`, so tag *n* failing leaves tags 1..n−1 applied |
| 9007 event insert fails after `create_channel_with_id` | not stored | channel soft-deleted by compensation | `Internal` | `ingest.rs:2430-2440` — manual compensation, itself `warn!`-only on failure |
| 30617 pointer seed fails | kind:30617 committed | name reservation released **only if this attempt created it** | success | `side_effects.rs:2528-2555` |
| 30617 kind:30618 emission fails | committed | pointer exists, subscribers miss the "repo exists" signal | success | `side_effects.rs:2588-2601` — explicitly "Non-fatal" |
| `emit_live_thread_summary` fails | committed, counters correct | live badge counts stale until the next page fetch | success | `side_effects.rs:724-815`, spawned task, `warn!` on every failure branch |
| `emit_system_message` insert fails | committed | no kind:40099 tombstone / notice | success (side effect returns `Ok`) | `side_effects.rs:690-697` — insert failure is `warn!`-ed and the function still returns `Ok(())` |
| `emit_membership_notification` fails | committed | agent never learns to resubscribe | success | `side_effects.rs:1248-1256`, `:1319-1327`, `:1766-1774` |
| `emit_group_discovery_events` fails | committed | 39000/39001/39002 stale | success | `side_effects.rs:1244`, `:1315`, `:1553`, `:1762`, `:1875` |
| Redis `publish_event` fails | committed | `local_event_ids` echo-dedupe entry **rolled back** so a later Redis retry is not swallowed | success | `side_effects.rs:790-800`, `:869-879`; same pattern in `handlers/event.rs:390` |
| Audit enqueue channel closed | committed | audit entry lost | success | `handlers/event.rs:597-600` — `error!` + `buzz_audit_send_errors_total` |
| Command mutation succeeds, `tx.commit()` fails | **not** stored | mutation persisted | `Internal("error: commit transaction: …")` | `command_executor.rs:92-98` documents this exact window; safety rests on `open_dm` / `hide_dm` / `update_approval` / `upsert_workflow` being idempotent so a retry converges |
| 28936 NIP-43 publishes fail | member already removed | 8001/13534 not published | `accepted: true, "info: you have left this relay"` | `ingest.rs:1909-1922` |

#### 3c. Fail-closed vs fail-open

Fail-**closed** (error propagates, write refused):
- restriction-state lookup (`ingest.rs:1633-1641`, comment: "a DB error must not let a
  banned/timed-out actor write");
- 9021 membership check (`side_effects.rs:1861-1866`, "Fail closed on DB errors rather
  than falling through to add_member");
- 9005 target lookup (`side_effects.rs:552-558`, "Fail closed: missing target → reject");
- 9002 `ttl` parse in the *side effect* (`side_effects.rs:1454-1464`, "a parse failure must
  reject, never silently clear the TTL to permanent");
- 30617 pointer establishment (`side_effects.rs:2528-2559`).

Fail-**open** (error swallowed):
- every side effect after storage (BR-IN-69);
- `insert_mentions` (`buzz-db/src/lib.rs:1394-1399`);
- `ensure_user` in the command executor (`command_executor.rs:60-63` — `warn!` and
  continue, even though the comment at `:44-46` says it exists to satisfy a foreign-key
  requirement; a subsequent FK violation would then surface as `Internal`);
- kind:0 NIP-05 UNIQUE collision, which retries the profile write **without** the handle
  rather than failing (`side_effects.rs:1174-1195`);
- kind:0 off-domain NIP-05, silently cleared because "the event is already persisted and
  can't be rejected" (`side_effects.rs:1150-1153`).

---

### 4. Fan-out path (via `handlers/event.rs`)

`dispatch_persistent_event` (`handlers/event.rs:326-367`) is called from
`ingest.rs:2375` (reaction), `:2513` (generic), and from `side_effects.rs:940`, `:2872`,
`:3045`, `:3132`, `:3195`. It:
1. awaits `enqueue_event_created_audit` (bounded mpsc, capacity 1000 — deliberate
   backpressure, `handlers/event.rs:540-548`);
2. spawns a task that marks the event local, publishes to Redis, then fans out to local
   subscribers, then spawns workflow evaluation.

Consequence: **NIP-01 `OK` is returned before Redis publish, local fan-out, or workflow
triggering complete** — stated explicitly at `handlers/event.rs:320-325`.

Workflow triggering is skipped for `is_workflow_execution_kind` (46001–46012),
`is_command_kind` (7 kinds), relay-signed events tagged `buzz:workflow`, and kind 1059
(`handlers/event.rs:497-502`). The workflow lookup is community-scoped, with the rationale
that a colliding channel UUID in another community must not trigger this community's
workflows (`handlers/event.rs:511-517`).

Three emitters deliberately bypass `dispatch_persistent_event` and call
`fan_out_event_to_local_subscribers` directly, skipping audit and workflow evaluation:
`emit_membership_notification` (`side_effects.rs:882-885`), `publish_nip43_delta`
(`:2905-2907`), `emit_initial_ref_state` (`:2755-2762`). The first documents this as
"Fan-out only — skip search indexing and workflow evaluation" (`side_effects.rs:855`).
`emit_live_thread_summary` does the same for a never-stored event (`:801-808`).

---

### 5. Subscription lifecycle coupling

`side_effects.rs` reaches directly into the connection and subscription registries — the
only non-transport module that does:

| Function | Effect | Trigger |
|---|---|---|
| `evict_live_channel_subscriptions` `:39` | closes a specific pubkey's channel subs across all their connections | 9001 (`:1295`), 9022 (`:1934`) |
| `evict_conn_channel_subscriptions` `:56` | removes from `sub_registry`, removes from the conn's local map, `release_topic`, sends `CLOSED restricted: channel access revoked` | the three above |
| `evict_non_member_channel_subscriptions` `:95` | closes subs for connections whose pubkey is not a current member | 9002 open→private (`:1437`) |
| `evict_all_channel_subscriptions` `:128` | closes every sub on a channel | ephemeral-channel reaper (`main.rs:672`) |

The reason string `channel access revoked` is chosen because it is in the client's
drop-set, so a connected agent drops one channel without reconnecting
(`side_effects.rs:118-125`).

---

### 6. Media integration (`imeta.rs`)

Read-only against `buzz_media::MediaStorage`: 3 `get_sidecar` reads and 3 `head` calls per
imeta tag worst case (`imeta.rs:246`, `:252`, `:290`, `:308`, `:328`). No writes, no
deletes, no retention interaction. Uploads happen out-of-band via Blossom; ingest only
proves the blob already exists and that the claimed metadata matches the sidecar.

Trust boundary: the sidecar is authoritative for `ext`, `mime_type`, `size`, and
`duration_secs`; the event's claims are checked *against* it, never the reverse
(`imeta.rs:259-278`). The upload validator's deny-list is therefore the real content gate,
and `validate_imeta_tags` only needs a structural MIME check — reasoned out at
`imeta.rs:71-76`.

Per-tenant media base URL: `media_base_url_for_tenant(&state.config.relay_url,
tenant.host())` (`ingest.rs:2211-2212`), so a tenant-host URL is accepted only when it
matches that tenant's base (tested at `imeta.rs:438-449`).

---

### 7. Git object store integration

`side_effects.rs` is the only handler that touches `state.git_store`:
`put_manifest` (`:2642`), `put_pointer(Precond::IfNoneMatchStar)` (`:2652`),
`get_pointer` (`:2664`, `:2714`). CAS semantics: `CasOutcome::Won` → success;
`CasOutcome::LostRace` → success **only if** the existing pointer names the same empty
manifest digest, otherwise a hard error rather than silently accepting a stale pointer
(`:2670-2691`). `ensure_manifest_pointer` (`:2704-2731`) is the tolerant re-announce
variant: any existing pointer is left untouched, an absent one is repaired.

The invariant maintained is "repo announced ⟺ pointer exists", so the read path can treat
pointer-absent as never-announced and keep `info_refs`'s fail-closed 404 unambiguous
(`side_effects.rs:2557-2571`).

---

### 8. Conformance-trace integration

`ingest_event` (`ingest.rs:1393`) wraps the pipeline in `EmitGuard::arm`
(`:1408-1412`), passes the counting tracer down, then maps terminal errors to a single
`SanitizedError` action (`:1436-1443`). Emitted actions: `AuthCheck` (`:1817-1825`),
`WriteInsert` / `WriteDuplicate` (`:2353-2376`, `:2493-2511`), `WriteInsertGlobal`
(`:2215-2222`, `:2506-2509`, plus `emit_product_feedback_success` `:133-154`).
Production tracer is `NoopTracer` (`state.rs:798`), so this is inert outside conformance
tests. Spec reference: `docs/spec/MultiTenantRelay.tla`, cited at `ingest.rs:1392`.


## Module: buzz-relay — HTTP API surface (`crates/buzz-relay/src/api`)
### Aspect: Integrations

---

#### 1. Dependency summary

| Collaborator | Reached via | Surface used |
|---|---|---|
| `buzz-db` | `state.db` | 27 distinct methods (§2) |
| `buzz-media` | `state.media_storage`, free fns | 5 storage methods + 6 pipeline/helper fns (§3) |
| `buzz-search` | `state.search` | `SearchService::search` (§4) |
| `buzz-pubsub` | `state.pubsub`, `state.nip98_replay`, `state.admission_rate_limiter` | presence bulk read, Redis replay seen-set, Redis rate limiter (§5) |
| `buzz-auth` | `state.auth`, free fns/traits | `verify_nip98_event`, `Scope::all_known`, `Nip98ReplayGuard`, `LimitType`, `DEFAULT_REPLAY_TTL_SECS`, `AuthConfig::rate_limits` (§6) |
| `buzz-core` | types | `TenantContext`, `CommunityId`, `StoredEvent`, `kind::*`, `filter::{filters_match, reader_authorized_for_event}`, `tenant::{normalize_host, relay_url_authority}` |
| `buzz-audit` | `state.audit_tx` | one `NewAuditEntry` for `MediaUploaded` |
| `buzz-workflow` | `state.workflow_engine` | `WorkflowDef`, `TriggerDef`, `TriggerContext`, `executor::execute_from_step`, `finalize_run` |
| `buzz-sdk` | free fn | `nip_oa::verify_auth_tag` |
| `buzz-relay-mesh` | `state.mesh()` | `RelayPeerTransport`, `ReliableStreamRouter`, `SessionDirectory` |
| `nostr` crate | ubiquitous | `Event`, `Filter`, `EventBuilder`, `Tag`, `PublicKey`, `EventId`, `SingleLetterTag` |
| `pulldown_cmark` | `invites.rs:132` | Markdown → HTML for policy pages |
| `url` crate | `admin/mod.rs:262-264` | parsing `imeta` attachment URLs |
| `infer` crate | `media.rs:50`, `:369-372`, `:380-382` | content sniffing |

**No outbound HTTP is issued from any file in this module group.** Every `reqwest` use in the
relay lives elsewhere (`buzz-workflow/src/executor.rs`). Grep for `reqwest` across the 13 assigned
files returns zero non-test hits.

---

#### 2. `buzz-db` call patterns

| Method | Called from | `file:line` | Notes |
|---|---|---|---|
| `is_relay_member(community, pubkey_hex)` | membership gate (agent, then owner) | `api/mod.rs:73-76`, `:92-96` | two round trips on the NIP-OA delegation path |
| `ensure_user(community, pubkey)` | NIP-OA backfill | `api/mod.rs:184-188` | called twice (agent + owner) before the FK-bearing write |
| `set_agent_owner` / `is_agent_owner` | NIP-OA backfill | `api/mod.rs:200-212` | first-write-wins, then confirm-same-owner |
| `query_events(&EventQuery)` | catch-all `/query`, aux closure | `bridge.rs:492-496`, `:1252-1255`, and count fallbacks `:1483`, `:1547` | `/query` catch-all runs **bounded-concurrent** with `futures_util::buffered(FILTER_QUERY_CONCURRENCY = 4)` and order-preserving post-processing |
| `count_events(&EventQuery)` | `/count` pushdown | `bridge.rs:1479`, `:1546` | only on the fully-pushable path; skipped entirely when the filter can match kind 30175 (`bridge.rs:1477`, `:1543`) |
| `get_channel_window(community, ch, limit, cursor, kinds)` | channel-window filter | `bridge.rs:466-476` | single call returning rows + `has_more` + `next_cursor` + per-row `thread_summary` |
| `get_thread_replies(community, root, depth, limit, cursor)` | thread filter | `bridge.rs:1153-1162` | keyset cursor encoded BE-i64 ‖ id bytes |
| `get_events_by_ids(community, &[&[u8]])` | FTS hydration | `bridge.rs:1701-1705` | batch fetch, then a `HashMap` to restore FTS relevance order |
| `query_feed_mentions` / `query_feed_needs_action` / `query_feed_activity` | feed filter | `bridge.rs:1066-1090` | one call per requested feed type, budget shared |
| `get_workflow(community, id)` | webhook | `bridge.rs:1809-1812` | any error ⇒ generic 404 |
| `create_workflow_run` / `update_workflow_run` | webhook | `bridge.rs:1879-1883`, `:1877-1886` | the update runs inside the detached task |
| `list_moderation_reports` / `list_moderation_actions` / `list_community_restrictions` | moderation reads | `bridge.rs:2102-2109`, `:2107-2111`, `:2124-2128` | limits clamped to ≤500 before the call |
| `get_user(community, pubkey)` | upload attribution | `media.rs:262-269` | best-effort; `.ok().flatten()` degrades to no display name |
| `get_user_by_nip05(community, name, domain)` | NIP-05 | `nip05.rs:50-54` | miss and error both fold into the empty response (`_ =>` arm at `:64`) |
| `get_relay_member(community, hex)` | invite mint authz | `invites.rs:234-239` | absent member ⇒ `role = ""` ⇒ 403 |
| `claim_relay_membership(community, hex, role, policy_version)` | invite claim | `invites.rs:325-338` | returns `was_inserted` for idempotency |
| `archive_community_owned_by(host, owner, deployment_host)` | operator | `operator.rs:234-239` | `None` ⇒ 404 |
| `unarchive_community_owned_by(host, owner)` | operator | `operator.rs:288-292` | `None` ⇒ 404 |
| `list_communities_owned_by(owner_hex)` | operator | `operator.rs:325-328` | **not** community-scoped (control plane) |
| `lookup_community_by_host_for_management(host)` | operator availability | `operator.rs:480-484` | separate from the admission lookup so archived rows are visible |
| `lookup_community_host(community)` | post-transfer publish | `operator.rs:437-441` | |
| `transfer_ownership(community, new, expected)` | operator | `operator.rs:392-396` | returns the `TransferResult` enum mapped to 200/404/409 |
| `admin_list_reports(...8 args)` / `admin_get_report(id)` / `admin_list_feedback(100)` / `admin_get_feedback(id)` | admin | `admin/mod.rs:101-111`, `:132-134`, `:155`, `:184-186` | **deployment-wide**, no `CommunityId` parameter on 3 of the 4 |
| `ensure_configured_community(host)` | tests only | `bridge.rs:3433`, `invites.rs:565`, `media.rs:1015` | |
| `ping()` | readiness (outside group) | `router.rs:249` | |

Cache reads/writes owned by `AppState` but driven from here:
`get_accessible_channel_ids_cached` (`bridge.rs:1002`, `:1425`),
`author_type_cache.insert` and `observer_owner_cache.insert` after a NIP-OA materialization
(`api/mod.rs:215-224`).

#### 3. `buzz-media` call patterns

| Call | From | `file:line` |
|---|---|---|
| `auth::verify_blossom_auth_event(event, Some(tenant.host()), 3600)` | upload extractor | `media.rs:177` |
| `auth::verify_blossom_get_auth(event, sha256, Some(tenant.host()), 3600)` | read gate | `media.rs:502` |
| `process_video_upload(storage, cfg, tenant, auth_event, stream, content_length, attribution)` | streaming video | `media.rs:344-353` |
| `process_upload(...)` (buffered image) | image path | `media.rs:375-383` |
| `process_file_upload(...)` (buffered generic) | generic path | `media.rs:390-398` |
| `MediaStorage::read_sidecar_mime(tenant, hash)` | serve + head | `media.rs:637-641`, `:648-652`, `:812-816`, `:822-826` |
| `MediaStorage::get_sidecar(tenant, hash)` | ext agreement + key resolution | `media.rs:653-657`, `:829-833`, `:868-871` |
| `MediaStorage::head_with_metadata(key)` | size for 200/206/HEAD | `media.rs:673-677`, `:705-710`, `:839` |
| `MediaStorage::get_stream(key)` | 200 body | `media.rs:678` |
| `MediaStorage::get_range(key, start, end)` | 206 body | `media.rs:730` |
| `looks_like_iso_bmff`, `serve_inline`, `parse_public_ip`, `parse_port` | helpers | `media.rs:51`, `:663`, `:279-280` |
| `MediaError` as the handler error type (its own `IntoResponse`) | all media handlers | `buzz-media/src/error.rs:107-162` |

**Key direction-of-dependency fact:** `verify_blossom_get_auth` is defined in `buzz-media`
(`auth.rs:207`) but its **only** call site in the whole repo is `media.rs:502` in this module —
i.e. `buzz-media` never gates reads itself; the gate lives here behind `require_media_get_auth`.

##### S3 access

All object access is indirect through `MediaStorage`; this module never constructs an S3 client and
never handles credentials. Object keys are content-addressed (`{sha256}.{ext}`) and derived from
either the request path (after `validate_media_path`) or the sidecar's `ext` (re-validated by
`is_safe_ext`, `media.rs:875-879`) — client input never reaches the key builder unvalidated.
Sidecar/`_uploads/` keys are structurally unreachable through the serve path
(`media.rs:547-583`; test `:1250-1264`).

#### 4. `buzz-search`

One integration point: `handle_bridge_search` (`bridge.rs:1616-1749`).

| Element | Detail | `file:line` |
|---|---|---|
| Entry | `state.search.search(&SearchQuery)` | `bridge.rs:1694-1698` |
| `SearchQuery` fields | `community`, `q`, `channel_scope`, `kinds`, `authors`, `since`, `until`, `page`, `per_page`, `mode` | `bridge.rs:1666-1704` |
| `ChannelScope` | `Channels(valid_uuids)` when `#h` present and intersects accessible, else `build_search_channel_scope_filter(accessible, include_global = true)`; `None` ⇒ early `Ok([])` | `bridge.rs:1618-1650` |
| `SearchMode` | `Prefix` on `search_mode`/`searchMode == "prefix"`, else `FullText` | `bridge.rs:368-378` |
| Post-filter | FTS returns only ids; full events are fetched from `buzz-db` and re-checked by `search_hit_accepted` (all non-pushed constraints + channel scope + reader auth) | `bridge.rs:1590-1607`, `:1717-1719` |
| Error mapping | `internal_error("search error: …")` and `internal_error("search fetch error: …")` → generic 500 | `bridge.rs:1697`, `:1694` |

#### 5. `buzz-pubsub` / Redis

| Integration | Detail | `file:line` |
|---|---|---|
| Presence bulk read | `state.pubsub.get_presence_bulk(tenant, &pubkeys)`; failure ⇒ `unwrap_or_default()` (empty) | `bridge.rs:1969-1975` |
| NIP-98 replay seen-set | `RedisNip98ReplayGuard` behind `Arc<dyn Nip98ReplayGuard>` (`state.rs:710-711`); community-scoped `try_mark` for bridge/invites, deployment-scoped `try_mark_in_scope("operator-management", …)` for operator | `bridge.rs:141`, `:158-161`; `operator.rs:108-122` |
| Rate limiter | `RedisRateLimiter` (`state.rs:712`) via `admission::check_principal` with `LimitType::ApiCalls`, 60 s window | `bridge.rs:31-38` |
| Cluster disconnect fan-out | `state.disconnect_community_clusterwide(&tenant)` after archive; failure ⇒ 503 retryable | `operator.rs:243-264` |
| NIP-43 side effects | `handlers::side_effects::publish_nip43_member_added` / `publish_nip43_membership_list` after invite claim and after ownership transfer — both best-effort | `invites.rs:344-355`; `operator.rs:445-455` |
| Mesh session directory | `SessionDirectory` (Redis fenced leases) via `ReliableStreamRouter::join` | `mesh_demo.rs:71-77`, `:100` |

Two limiters in this module are **not** Redis-backed and therefore not cluster-consistent:
`media_upload_rate_limiter` + `media_uploads_in_flight` (DashMap, `state.rs:592`, `:600`) and
`invite_claim_rate_limiter` (moka, `state.rs:597-598`).

#### 6. `buzz-auth`

| Item | Use | `file:line` |
|---|---|---|
| `verify_nip98_event(json, url, method, body)` | the actual signature/URL/method/payload check | `bridge.rs:110-111` |
| `Nip98ReplayGuard` trait | injected via `AppState`; four test doubles implement it (`AlwaysErrGuard`, `AlwaysFreshReplayGuard` ×3, `SeenOnceReplayGuard`) | `bridge.rs:2348-2356`, `:3268-3281`; `invites.rs:415-427`, `:1103-1128`; `operator.rs:551-563` |
| `DEFAULT_REPLAY_TTL_SECS` | TTL for both replay scopes | `bridge.rs:159`; `operator.rs:113` |
| `LimitType::ApiCalls` | rate-limit bucket | `bridge.rs:34` |
| `Scope::all_known()` | 16 scopes granted to every HTTP ingest | `bridge.rs:829` |
| `state.auth.config().rate_limits.human_api_calls_per_min` | the per-minute budget | `bridge.rs:29` |
| `AuthError` | surfaced only through guard failures, mapped to 401 | `bridge.rs:167-176` |
| `nip42::verify_nip42_event` | tests only in this module; production caller is `handlers/auth.rs:80-81` using `bridge::nip42_expected_relay_url` | `bridge.rs:2797-2804` |

#### 7. Reverse dependencies — who calls into this module

| Consumer | Symbol | `file:line` |
|---|---|---|
| `handlers/auth.rs` (WS NIP-42 door) | `relay_members::enforce_relay_membership`, `extract_nip_oa_owner`, `materialize_nip_oa_owner`, `bridge::nip42_expected_relay_url` | `handlers/auth.rs:217`, `:137`, `:246`, `:258`, `:81` |
| `audio/handler.rs` (huddle WS) | `relay_members::enforce_relay_membership`, `bridge::nip42_expected_relay_url` | `audio/handler.rs:244`, `:219` |
| `api/git/transport.rs` | `relay_members::enforce_relay_membership` | `git/transport.rs:211` |
| `handlers/ingest.rs` | `api::validate_imeta_tags`, `api::verify_imeta_blobs`, `api::media::media_base_url_for_tenant` | `ingest.rs:2239`, `:2215`, `:2212` |
| `handlers/product_feedback.rs` | same three | `product_feedback.rs:31-33` |
| `handlers/imeta.rs` | `api::media::is_safe_ext` | `imeta.rs:378`, `:406` |
| `handlers/side_effects.rs` | `api::nip05::canonicalize_nip05` | `side_effects.rs:1145` |
| `api/admin/mod.rs` | `api::media::serve_blob_for_tenant`, `api::media::is_safe_ext` | `admin/mod.rs:226`, `:283` |
| `handlers/command_executor.rs` | `webhook_secret::{generate_webhook_secret, inject_secret, extract_secret}` | `command_executor.rs:713-718` |
| `router.rs` | every routed handler + `api::admin::{router, is_admin_host}` | `router.rs:39-128`, `:59`, `:145`, `:264` |

**Doc delta:** `api/mod.rs:36-38` states the membership gate is "Called by `media.rs`, `bridge.rs`,
`git/transport.rs`, and `audio/handler.rs`." It omits `handlers/auth.rs:217`, which is the WebSocket
door — arguably the most important caller.

#### 8. Metrics emitted

| Metric | Labels | `file:line` |
|---|---|---|
| `buzz_admission_rejections_total` | `transport="http"`, `reason="quota"\|"unavailable"` | `bridge.rs:40`, `:49` |
| `buzz_events_rejected_total` | via `reject_with_transport("http", "invalid"\|"auth"\|"error")` | `bridge.rs:734`, `:857`, `:862`, `:868` |
| `buzz_count_fallback_rejections_total` | none | `bridge.rs:1492`, `:1558` |
| `buzz_media_upload_rejections_total` | `reason="rate_limit"\|"concurrency"` | `media.rs:216`, `:222` |
| `buzz_media_legacy_upload_route_total` | none | `media.rs:315` |
| `buzz_media_uploads_total` | `mime` (6-value allowlist), `community` (tenant host) | `media.rs:419-424` |
| `buzz_audit_send_errors_total` | none | `media.rs:443` |
| `buzz_users_created_total` | `community` | `api/mod.rs:189-193` |

Note `buzz_media_uploads_total` carries an unbounded-cardinality `community` label (one series per
tenant host) — acceptable at current tenant counts, a concern on a large multi-tenant deployment.


## Module: buzz-relay — git hosting (`crates/buzz-relay/src/api/git`)
### Aspect: Integrations

#### 1. Integration inventory

| Dependency | Direction | Surface | Criticality |
|---|---|---|---|
| S3 / MinIO (`rust-s3` 0.37) | out | 4 distinct bucket operations | **hard** — source of truth; startup probe is fatal |
| `git` binary (subprocess) | out | 10 production invocation sites, 9 distinct subcommands | **hard** — no in-process git |
| `buzz-db` (Postgres) | out | 4 read methods + 1 write | hard for push authz; the write is best-effort |
| `buzz-core` | in-proc | `TenantContext`, `CommunityId`, `MemberRole`, `git_perms::*` | hard |
| `buzz-auth` | in-proc | `nip98::verify_nip98_event` | hard |
| Relay event fan-out | out | `fan_out_event_to_local_subscribers` (bypasses `dispatch_persistent_event`) | best-effort |
| Local HTTP loopback | out+in | pre-receive hook → `POST /internal/git/policy` | **hard** — fail-closed |
| Local filesystem | out | tempdirs, tempfiles, pack cache | hard |
| `metrics` crate | out | 8 counters, 11 histograms, 3 gauges | observability |
| Runtime image | build | `curl` + `openssl` must be installed for the hook | **hard**, asserted by a test |

#### 2. Object store (every call)

Client construction — `store.rs:187-219`:
- `Region::Custom { region, endpoint }`, `Bucket::new(...).with_path_style()` (`store.rs:206-214`). Path-style is unconditional (MinIO compatibility; AWS accepts both).
- Credential selection (`store.rs:198-210`): both keys non-empty ⇒ `Credentials::new`; **both empty** ⇒ `Credentials::default()` (AWS chain: env, profile, IRSA web-identity, container, IMDS); exactly one empty ⇒ `StoreError::Config` (pinned `store.rs:961-982`).
- **Credentials and bucket are shared with `buzz-media`**: `state.rs:694-701` passes `config.media.s3_{endpoint,access_key,secret_key,bucket,region}` and `.expect("media storage was already constructed with this S3 config")`. There is no separate git bucket at runtime, despite `BUZZ_GIT_S3_*` appearing in probe/test helpers (`cas_publish.rs:1580-1601`).

| Bucket op | Site | Purpose | Headers / notes |
|---|---|---|---|
| `put_object_with_content_type_and_headers` | `store.rs:262-268` (`put_immutable`) | write `packs/<sha256>`, `manifests/<sha256>` | `If-None-Match: *`; 2xx ⇒ key, 412 ⇒ key (idempotent), other ⇒ `Backend` |
| `put_object_with_content_type_and_headers` | `store.rs:297-306` (`put_idx`) | write `idx/<pack-digest>` | `If-None-Match: *`; same 412 handling |
| `put_object_with_content_type_and_headers` | `store.rs:489-501` (`put_pointer`) | the CAS | `If-Match: <etag>` or `If-None-Match: *`; content type `application/json`; result routed through `classify_cas` (`:516-546`) |
| `put_object_with_content_type_and_headers` | `store.rs:886-899` (`put_immutable_raw`) | probe phase 3 only — returns the raw status so 412s are countable | `If-None-Match: *` |
| `get_object` | `store.rs:348-352` (`get`) | fetch any object; 404 ⇒ `NotFound` | no range requests anywhere — full-body GET |
| `get_object` | `store.rs:456-478` (`get_pointer`) | atomic `(ETag, body)` snapshot | fails if no `etag`/`ETag` header |
| `head_object` | `store.rs:407-425` (`get_limited`) | size pre-check before download | followed by a post-read length re-check (`:429-437`) |
| `delete_object` | `store.rs:618`, `:728`, `:871` | probe scratch cleanup only | **the protocol never deletes packs/manifests/pointers** (doc §Axioms A1 no-deletion rule) |

Read/write call graph:

```
receive_pack → hydrate_for_write → get_pointer, get_verified_limited(manifest),
                                    [pack_cache] get_verified_limited(pack), get_idx
             → cas_publish       → put_pack, put_idx, put_manifest, put_pointer
                                    [on LostRace] get_pointer, get_verified
info_refs fast → load_manifest_for_read → get_pointer, get_verified_limited
info_refs slow / upload_pack → hydrate_for_read → same as hydrate_for_write minus ParentState
announce (side_effects) → put_manifest, put_pointer, get_pointer
startup → run_conformance_probe → put_pack, get_verified, put_pointer, get_pointer,
                                   put_immutable_raw, delete_object
```

Note: `store.rs:25` carries a module-wide `#![allow(dead_code)]` whose comment ("wired in by the push path in a follow-up commit") is stale.

#### 3. `git` subprocess invocations

Exactly **10** production `Command::new("git")` sites (3 in `transport.rs`, 6 in `cas_publish.rs`, 1 shared helper in `hydrate.rs`). All go through `harden_git_env` except `hydrate::run_git`, which hand-rolls a *different* env.

##### Environment

`harden_git_env` (`transport.rs:294-310`):

| Var | Value |
|---|---|
| — | `env_clear()` first |
| `PATH` | inherited (`std::env::var("PATH").unwrap_or_default()` — empty string if unset) |
| `GIT_HTTP_EXPORT_ALL` | `1` |
| `GIT_CONFIG_NOSYSTEM` | `1` |
| `GIT_CONFIG_GLOBAL` | `/dev/null` |
| `HOME` | `/dev/null` |

`hydrate::run_git` (`hydrate.rs:451-465`): `env_clear()`, `PATH` (only if set), `GIT_CONFIG_NOSYSTEM=1`, `HOME=<cwd>`. **Missing `GIT_CONFIG_GLOBAL`** — its doc comment claims it "matches transport.rs's harden_git_env semantics" (`hydrate.rs:456-457`), which is inaccurate; the `HOME=<cwd>` trick is what makes the global lookup miss.

`receive_pack` adds, on top of `harden_git_env` (`transport.rs:917-931`):

| Var | Value |
|---|---|
| `BUZZ_HOOK_URL` | `http://127.0.0.1:<config.bind_addr.port()>/internal/git/policy` |
| `BUZZ_HOOK_SECRET` | `config.git_hook_hmac_secret` |
| `BUZZ_REPO_ID` | stripped repo id |
| `BUZZ_REPO_OWNER` | owner hex from the URL |
| `BUZZ_COMMUNITY_ID` | resolved community UUID |
| `BUZZ_PUSHER_PUBKEY` | authenticated pusher hex |
| `GIT_CONFIG_COUNT` | `1` |
| `GIT_CONFIG_KEY_0` | `core.hooksPath` |
| `GIT_CONFIG_VALUE_0` | `<workspace>/hooks` |

##### Invocation table

| # | Site | argv | cwd | stdin | stdout | stderr | Timeout | kill_on_drop |
|---|---|---|---|---|---|---|---|---|
| 1 | `transport.rs:645-653` | `git {upload-pack\|receive-pack} --stateless-rpc --advertise-refs <workspace>` | inherited | inherited | `NamedTempFile` in `git_repo_path` | tempfile (64 KiB prefix logged) | 120 s (`:660`) | yes |
| 2 | `transport.rs:1019-1027` | `git receive-pack --stateless-rpc <workspace>` | inherited | **piped** — request body pumped by a spawned task (`:1037-1064`) | tempfile | tempfile | 300 s (`:1067`), body task aborted on timeout | yes |
| 3 | `transport.rs:1423-1434` | `git upload-pack --stateless-rpc <workspace>` (`extra_args` always empty) | inherited | **piped** — body pumped by a detached task (`:1442-1467`) | **piped → HTTP body** | `null` | 300 s in-band deadline (`:1471`), child killed (`:1315-1331`) | yes |
| 4 | `cas_publish.rs:284-296` | `git for-each-ref --format=%(refname) %(objectname)` | workspace | inherited | tempfile in scratch | tempfile | none | no |
| 5 | `cas_publish.rs:337-345` | `git symbolic-ref --quiet HEAD` | workspace | inherited | in-memory (`.output()`) | in-memory | none | no |
| 6 | `cas_publish.rs:409-415` | `git index-pack <tmp>/pack-<digest>.pack` | private tempdir | inherited | in-memory | in-memory | none | no |
| 7 | `cas_publish.rs:524-541` | `git pack-objects --revs --stdout -q --threads=1 --window 10 --window-memory=67108864` | workspace | piped (rev-spec lines) | tempfile in scratch | tempfile | 300 s (`:433-460`) | yes |
| 8 | `cas_publish.rs:608-620` | `git count-objects -v` | workspace | piped (empty) | tempfile | tempfile | 300 s | yes |
| 9 | `cas_publish.rs:707-729` | `git pack-objects --revs -q --threads=1 --window 10 --window-memory=67108864 --max-pack-size=<n> <tmp>/compact` | workspace | piped (deduped tips) | tempfile in compaction tempdir | tempfile | 300 s inner, 600 s outer (`:1058`) | yes |
| 10 | `hydrate.rs:451-470` (`run_git`, 4 arg sets) | `git init --bare --quiet` (`:182`); `git symbolic-ref HEAD refs/heads/main` (`:183`); `git verify-pack <idx>` (`:393`); `git index-pack <pack>` (`:420`) | caller-supplied | inherited | in-memory | in-memory | **none** | yes |

Observations:

- Sites 4, 5, 6, 10 have **no timeout**. Site 10 covers `index-pack` on an attacker-influenced pack (through the pack cache), so a pathological pack can occupy a semaphore permit for an unbounded time.
- Sites 1, 4, 5, 6, 10 do not set `stdin`, so the child inherits the relay's stdin.
- All repo paths are passed as a **single argv element** (`Command::arg`, no shell), so `--stateless-rpc <path>` cannot be word-split. Paths are OS-generated tempdir paths, never user strings.
- `--threads=1` and `--window-memory` bound one `pack-objects`; total CPU is bounded only by `git_semaphore` (20) and the compaction semaphore (1).
- No `git gc`, `repack`, `fsck`, `prune`, or `update-ref` is ever invoked. Refs are written as loose files directly (`hydrate.rs:355-371`).
- Tests additionally shell out to `bash` twice (`policy.rs:682`, `:757`) to cross-verify the hook's HMAC against the Rust implementation.

#### 4. `buzz-db` (Postgres)

| Call | Site | Purpose |
|---|---|---|
| `query_events(EventQuery{ kinds:[30617], pubkey:owner, d_tag:repo_id, global_only:true, limit:1, ..for_community(community) })` | `policy.rs:254-284` | resolve the repo announcement for protection rules + channel binding. Direct DB query, so it bypasses the relay's `kinds`-required p-gate. |
| `get_channel(community, channel_id)` | `policy.rs:308-330` | archived-channel read-only check |
| `is_agent_owner(community, owner_bytes, pusher_bytes)` | `policy.rs:330-368` | NIP-OA managed-agent owner authority |
| `get_member_role(community, channel_id, pusher_bytes)` | `policy.rs:353-395` | channel role for non-owners |
| `insert_event(community, kind:30618, None)` | `transport.rs:1693-1700` | persist the derived ref-state event; `(_, false)` means DB dedup |
| `crate::tenant::bind_community(&state.db, raw_host)` | `transport.rs:128-130` | host → community (uses `db` as `HostResolver`) |

Every failure in the first four ⇒ 403 (fail-closed, `policy.rs:277-282`, `:322-326`, `:355-362`, `:388-392`). Failure of `insert_event` ⇒ warning only (`transport.rs:1727-1735`).

Not used by this module but part of the same feature: `git_repo_names` reservation and quota (`repo_name_owner`, `count_repos_for_owner`, `reserve_repo_name`, `release_repo_name`) live in `handlers/side_effects.rs:2463-2560`.

#### 5. `buzz-pubsub`

**Not used directly.** kind:30618 is fanned out through `crate::handlers::event::fan_out_event_to_local_subscribers` (`transport.rs:1701-1710`, and `handlers/side_effects.rs:2755-2761` for the announce path) — the *local* subscriber path only. It bypasses `dispatch_persistent_event`, so whatever cross-pod Redis publication that function performs does **not** happen for git ref-state events: a subscriber connected to a different pod will not see the 30618 in real time and must re-query. The code comments justify the bypass only in terms of the access gate no-op for `channel_id = None`.

#### 6. Relay event emission

| Event | Trigger | Signer | Path |
|---|---|---|---|
| kind:30618 (post-push) | successful CAS with `parent_digest != committed_digest` | relay keypair (`state.relay_keypair`) | `transport.rs:1662-1746` |
| kind:30618 (announce seed) | fresh kind:30617 reservation | relay keypair | `handlers/side_effects.rs:2728-2765` |

Tags: `["d", repo_id]`, one `[<refname>, <oid>]` per emittable ref, `["HEAD", "ref: <head>"]`, `["p", <actor-hex>]` (`manifest_event.rs:74-108`). Content is empty. Only `refs/heads/*` and `refs/tags/*` are emitted (`manifest_event.rs:117-127`), and refs with non-40/64-hex oids or malformed names are skipped silently (`:82-93`).

#### 7. The HMAC hook callback loop

```
receive_pack (transport.rs:858)
  └─ install_hook → <workspace>/hooks/pre-receive, 0o755        hook.rs:152-178
  └─ git receive-pack --stateless-rpc <workspace>               transport.rs:1019
        └─ pre-receive (bash)                                    hook.rs:32-150
             ├─ read "old new ref" lines from stdin              hook.rs:56
             ├─ git merge-base --is-ancestor old new             hook.rs:59-70  (quarantine env inherited)
             ├─ HMAC-SHA256 via `openssl dgst -sha256 -hmac`     hook.rs:118
             └─ curl --silent --max-time 10 -X POST $BUZZ_HOOK_URL
                                                                  hook.rs:129-139
                   └─ POST /internal/git/policy                   mod.rs:62 → policy.rs:173
                         ├─ require_localhost middleware          mod.rs:41-52
                         ├─ structural validation                  policy.rs:176-234
                         ├─ verify_hmac (constant-time)            policy.rs:159-171, :236-241
                         ├─ 30s TTL / 5s future skew               policy.rs:243-256
                         ├─ 4 buzz-db lookups                      policy.rs:254-395
                         └─ buzz_core::git_perms::evaluate_push    policy.rs:404-416
             └─ non-200 ⇒ echo body to stderr, exit 1 (deny)     hook.rs:141-148
```

Cross-boundary contract details:

- The bash side sets `LC_ALL=C` so `sort` order and `${#var}` byte lengths match Rust's byte comparison and `String::len()` (`hook.rs:36-38`).
- Refs are sorted by `ref_name` on both sides — bash `sort` on a `ref_name`-first line, Rust `sort_by_key(|r| r.ref_name.clone())` (`hook.rs:113-121`, `policy.rs:143-146`). Order-independence is pinned by `policy.rs:559-576`.
- The pre-image format agreement is verified by **two** tests that actually run `bash` + `openssl` and compare against `generate_hook_hmac`: `policy.rs:592-703` (two refs, unsorted input) and `:704-773` (single ref). These are the only tests of the module's most security-critical contract.
- `set -eo pipefail` plus `: "${VAR:?…}"` guards make a missing env var abort the hook (`hook.rs:35`, `:41-47`).
- Both `ref_name` and `repo_id` are JSON-escaped with `sed 's/\\/\\\\/g; s/"/\\"/g'` before interpolation (`hook.rs:74-76`, `hook.rs:126`).
- The relay's runtime container must ship `curl` and `openssl`; `hook.rs:184-206` parses the repo `Dockerfile`'s runtime stage and fails the unit test if either is missing.
- `is_ancestor` is *reported by the hook*, not recomputed by the relay, and it is HMAC-covered (pinned `policy.rs:512-520`). The relay trusts the hook's ancestry claim.

#### 8. Startup and shared state wiring

| Item | Site |
|---|---|
| `AppState.git_store` built with `.expect(...)` from the **media** S3 config | `state.rs:694-701` |
| `AppState.git_pack_cache` built with `.expect("git pack cache path must be available")` | `state.rs:702-709` |
| `AppState.git_semaphore = Semaphore::new(git_max_concurrent_ops)`; doc explicitly says it is **not** writer serialization | `state.rs:517-521`, `:729` |
| `git_router` + `git_policy_router` merged into the main router | `router.rs:48-50`, `:137-138` |
| Fatal A3 conformance probe before the listener opens | `main.rs:466-503` |
| `BUZZ_SERVE_GIT_WEB_GUI` gates SPA fallback for `/`, `/repos`, `/repos/*` | `router.rs:206-213` |
| DB-derived gauges `buzz_total_git_repos` / `buzz_community_git_repos` | `main.rs:1499`, `:1706-1725` |
| UDS listener uses `.into_make_service()` (no `ConnectInfo`) | `main.rs:1182` |
| `tokio-util` `io` feature enabled specifically to stream git stdout into the HTTP body | `crates/buzz-relay/Cargo.toml:31-34` |

#### 9. Metrics emitted

| Name | Type | Labels | Site |
|---|---|---|---|
| `buzz_git_semaphore_rejections_total` | counter | `operation` | `transport.rs:323-327` |
| `buzz_git_upload_pack_timeouts_total` | counter | — | `transport.rs:1364` |
| `buzz_git_upload_pack_stream_seconds` / `_bytes` | histogram | — | `transport.rs:1386-1389` |
| `buzz_git_hydrations_total` | counter | `outcome` ∈ {success, missing, invalid_pointer, manifest_error, store_error, hydrate_error, resource_limit} | `hydrate.rs:146` |
| `buzz_git_hydrate_seconds` | histogram | `outcome` | `hydrate.rs:147` |
| `buzz_git_hydrate_bytes` / `_packs` | histogram | — | `hydrate.rs:135-136` |
| `buzz_git_pack_cache_lookups_total` | counter | `result` ∈ {hit, miss, coalesced} | `pack_cache.rs:173`, `:197-200` |
| `buzz_git_pack_cache_populate_seconds` | histogram | `outcome` ∈ {success, bypass, error} | `pack_cache.rs:223-226` |
| `buzz_git_pack_cache_population_wait_seconds` | histogram | — | `pack_cache.rs:266` |
| `buzz_git_pack_cache_populations_active` | gauge | — | `pack_cache.rs:93`, `:268` |
| `buzz_git_pack_cache_bytes` / `_entries` | gauge | — | `pack_cache.rs:477-480` |
| `buzz_git_pack_cache_bypasses_total` | counter | — | `pack_cache.rs:302` |
| `buzz_git_pack_cache_copy_fallbacks_total` | counter | — | `pack_cache.rs:453` |
| `buzz_git_pack_cache_evictions_total` | counter | — | `pack_cache.rs:383` |
| `buzz_git_pack_compactions_total` | counter | `outcome` ∈ {success, fallback, cas_conflict, validation_error, publish_error} | `cas_publish.rs:968` |
| `buzz_git_pack_compaction_seconds` | histogram | `outcome` | `cas_publish.rs:969` |
| `buzz_git_pack_compaction_packs_before` / `_after` / `_bytes` | histogram | — | `cas_publish.rs:971-977` |
| `buzz_git_pack_compaction_required_failures_total` | counter | — | `cas_publish.rs:1129` |

**Gap:** no counter for push outcome (2xx / 409 conflict / 400 invalid-manifest / 413), and none for policy-endpoint allow/deny. CAS contention and authorization denials are therefore invisible in metrics — which matters because the design explicitly says "if contention ever shows up in metrics the fix is a short best-effort *local* lock" (doc §Scope). That signal does not currently exist.


## Module: buzz-relay — moderation, admin & background workers (`crates/buzz-relay/src`)
### Aspect: Integrations

---

#### 1. Dependency map

| Integration | Reached via | Files that touch it |
|---|---|---|
| `buzz-db` (Postgres) | `state.db` | all 12 except `moderation_authz` helpers' pure half |
| `buzz-pubsub` (Redis) | `state.pubsub`, indirectly via `state.disconnect_pubkey_clusterwide` / `dispatch_persistent_event` | `moderation_commands`, `moderation_notices`, `workflow_sink` |
| `buzz-audit` (hash chain) | `state.audit_tx` — **only transitively**, via `dispatch_persistent_event` | `moderation_notices`, `workflow_sink` (and `identity_archive` via ingest fall-through) |
| `buzz-media` / S3 | `state.media_storage` | `report` (sidecar lookup), `product_feedback` (blob verify), `storage_sweep` (bucket listing) |
| `buzz-search` | via `dispatch_persistent_event` | `moderation_notices`, `workflow_sink` |
| `buzz-workflow` | trait impl of `ActionSink` | `workflow_sink` |
| `buzz-core` | kinds, tenant, filter matching | all |
| `buzz-sdk` | `nip_oa::verify_auth_tag` | `identity_archive` |
| APNs push gateway | outbound HTTPS | `push_runtime` |
| `metrics` recorder (Prometheus) | `metrics::gauge!` / `counter!` | `storage_sweep`, `moderation_notices` |

---

#### 2. `buzz-db` (Postgres)

##### 2.1 Call inventory

| File | DB calls | Notable |
|---|---|---|
| `moderation_commands.rs` | `moderation_restriction_state` (`:105`), `ban_community_member` (`:169`), `unban_community_member` (`:248`), `timeout_community_member` (`:287`), `untimeout_community_member` (`:351`), `get_moderation_report_by_event` (`:414`), `resolve_moderation_report` (`:461`), `insert_moderation_action` (`:529`) | 8 distinct calls; **no transaction wraps them** |
| `moderation_authz.rs` | `get_relay_member` (`:98`, `:110`), `get_member_role` (`:126`) | 1–2 queries per authorization; `get_member_role` is unreachable |
| `moderation_notices.rs` | `open_dm` (`:102`), `unhide_dm` (`:127`), `query_events` (`:230`), `insert_event` (`:174`), `replace_addressable_event` (`:206`) | 5 calls per notice, minimum |
| `relay_admin.rs` | `get_relay_member` (`:135`, `:317`), `set_community_icon` (`:157`), `add_relay_member` (`:199`), `remove_relay_member_if_role` (`:243`), `remove_relay_member` (`:250`), `update_relay_member_role` (`:309`) | admin remove is a single conditional delete (TOCTOU-safe, `:235-241`) |
| `community_provisioning.rs` | `create_community_with_owner` (`:286`), `ensure_configured_community` (`:323`), `bootstrap_owner` (`:330`) | create-only path is one atomic call; convergence path is two |
| `report.rs` | `get_event_by_id` (`:56`), `insert_moderation_report` (`:79`) | |
| `product_feedback.rs` | `insert_product_feedback` (`:57`) | |
| `identity_archive.rs` | `get_relay_member` (`:241`), `query_events` (`:273`), `archive` (`:75`), `unarchive` (`:85`) | |
| `push_lease.rs` | `accept_push_lease_event` (`:563`) | single atomic transaction owning ordering + quota + endpoint namespace (`buzz-db/src/push.rs:206-212`) |
| `push_runtime.rs` | 13 distinct: `reap_exhausted_push_matches` (`:62`), `claim_due_push_match_batch` (`:72`), `active_push_match_leases` (`:102`), `membership_pairs` (`:115`), `enqueue_push_wakes` (`:184`), `complete_push_match_batch` (`:201`), `retry_push_match_batch` (`:207`/`:137`), `usage_community_hosts` (`:320`), `claim_due_push_wakes` (`:325`), `revalidate_push_wake` (`:356`/`:405`), `is_member` (`:375`), `disable_push_endpoint` (`:457`), `complete_push_wake` (`:438`/`:494`), `fail_push_wake` (`:363`/`:382`/`:412`/`:444`/`:471`/`:501`/`:535`), `retry_push_wake` (`:390`/`:541`) | heaviest DB consumer in the group |
| `workflow_sink.rs` | `lookup_community_host` (`:201`), `get_channel` (`:223`), `is_member_cached` (`:244`), `get_members` (`:275`), `get_users_bulk` (`:281`), `insert_event_with_thread_metadata` (`:339`) | 6 queries per workflow message |
| `storage_sweep.rs` | none directly — the host map is supplied by `main.rs` | |

##### 2.2 Failure modes

| Path | On DB error | Fail-open or fail-closed? |
|---|---|---|
| Restriction read (`moderation_commands.rs:107`) | `error: database error checking restriction state: {e}`, command rejected | **closed** |
| Ingest restriction gate (`ingest.rs:1645-1650`) | `IngestError::Internal` | **closed** |
| Auth-seam restriction read (`handlers/auth.rs:118-131`, `:145-152`) | `BanOutcome::DbError` ⇒ deny with `error: internal …`, distinguished from a real ban | **closed** |
| `authorize_moderation_action` (`moderation_authz.rs:99`, `:111`, `:127`) | `anyhow` error `?`-propagated, wrapped as `restricted: {e}` by `authz_denial` (`moderation_commands.rs:549`) | **closed, but the error string leaks a DB message under a `restricted:` prefix** |
| Ban/timeout write (`moderation_commands.rs:174`, `:292`) | `error: database error: {e}` | closed |
| Audit-row write (`moderation_commands.rs:544`) | `error: failed to write audit row: {e}` — **command fails after the ban already committed** | see §2.3 |
| Notice DM (any of 5 calls) | `anyhow` bubbles to `send_moderation_notice`'s caller, which logs and continues | **open by design** (`moderation_commands.rs:214-219`) |
| `relay_admin` any call | `database error: {e}`, wrapped by ingest as `invalid: database error: …` | closed |
| Report target resolution (`report.rs:58`) | `error: database error resolving report target: {e}` | closed |
| Push lease persistence (`push_lease.rs:572`) | `AcceptError::Internal("lease persistence failed")` — **the underlying `DbError` is discarded with `map_err(|_| …)`** | closed, but undiagnosable |
| Matcher context load (`push_runtime.rs:132-149`) | whole batch retried in 2 s | closed, retrying |
| Matcher per-job error (`:170-176`) | retry until `MAX_MATCH_ATTEMPTS = 8`, then discard as poison | bounded |
| Wake enqueue failure (`:186-198`) | contributing jobs retried; outbox dedup key absorbs partial commits | idempotent |
| `revalidate_push_wake` error (`:366-369`, `:414-417`) | `return` without touching the row — the claim lease simply expires | safe |
| `is_member` error during delivery (`:385-401`) | `retry_push_wake` in 2 s | closed |
| `usage_community_hosts` error (`:337`) | `error!` then the outer loop backs off | degraded, keeps looping |
| `workflow_sink` any call | `ActionSinkError::Database` → `WorkflowError::WebhookError` (`buzz-workflow/src/action_sink.rs:31-34`) → run fails | closed |
| `lookup_community_host` returns `None` (`workflow_sink.rs:205-210`) | `Database("workflow run community {id} is not mapped to a host")` | **closed by design** |

##### 2.3 Non-atomicity across the moderation write set

`handle_ban` performs four independent operations with no enclosing transaction: restriction read (`:105`), ban write (`:169`), audit write (`:180`), disconnect (`:195`), notice (`:204`). If the audit write fails, the ban is already durable but the command returns `error: failed to write audit row: …` (`:544`) — the client sees a failure for a ban that took effect. Same shape for timeout (`:287` then `:298`).

`handle_resolve` inverts the order deliberately — audit row **first** (`:453`), resolve **second** (`:461`) — with an in-code note that a lost-race resolve can leave an orphan audit row and that the residual window is tolerated (`:419-425`, `:469-474`).

---

#### 3. `buzz-pubsub` (Redis)

##### 3.1 Ban disconnect fan-out

`moderation_commands.rs:195-200` → `state.disconnect_pubkey_clusterwide` (`state.rs:1018-1050`):
1. Synchronous local socket close, fenced to the community (`state.rs:1025-1027`).
2. `tokio::spawn`ed `pubsub.publish_conn_control(&tenant, ConnControl::DisconnectPubkey { pubkey, event_id, reason })` (`state.rs:1043-1047`).

**Failure mode: fire-and-forget.** A Redis publish failure only emits `warn!("Failed to publish conn-control disconnect: {e}")` (`state.rs:1045`). The ban command still returns success. The in-code justification is that the durable ban row rejects the member again at auth (`state.rs:1039-1042`) and that the ingest write-path gate is the backstop for a surviving socket (`ingest.rs:1615-1622`).

**Consequence:** on another pod, a banned user's already-open socket keeps receiving events until (a) the Redis command arrives, (b) the socket reconnects and hits the auth seam, or (c) the user attempts a write and hits the ingest gate. Reads are not gated by the ingest path, so a missed publish means continued read access for the life of the socket.

The banning pod re-receives its own publish and no-ops; origin suppression was deliberately not added (`state.rs:1029-1031`).

##### 3.2 Notice and workflow fan-out

Both `moderation_notices.rs:178`/`:210` and `workflow_sink.rs:351` reach Redis only indirectly through `dispatch_persistent_event`. The workflow path discards the result entirely (`let _ =`, `workflow_sink.rs:351`), so a fan-out failure is invisible to the workflow run — the message is persisted and reported as sent.

Redis is also constructed directly inside two test helpers: `identity_archive.rs:445-448` and `workflow_sink.rs:580-588`. The latter deliberately points at `redis://127.0.0.1:1` (`workflow_sink.rs:578`) to prove the path works without a live Redis.

---

#### 4. `buzz-audit` (hash chain)

##### 4.1 Verified: no handler in this module writes an audit entry directly

`buzz-audit` declares **11** actions (`buzz-audit/src/action.rs:5-29`): `EventCreated`, `EventDeleted`, `ChannelCreated`, `ChannelUpdated`, `ChannelDeleted`, `MemberAdded`, `MemberRemoved`, `AuthSuccess`, `AuthFailure`, `RateLimitExceeded`, `MediaUploaded`.

Production emits exactly **2**: `EventCreated` (`handlers/event.rs:583`) and `MediaUploaded` (`api/media.rs:428`). **9 declared actions have zero producers.**

> **Documentation delta:** ARCHITECTURE.md:497 states "**10 audit actions**" and enumerates them without `MediaUploaded`. The enum has 11 (`action.rs:29`). The doc is stale by one variant, and its list is the set that is *declared*, not the set that is *emitted*.

Grep across the 12 assigned files: zero references to `buzz_audit`, `AuditAction`, or `state.audit_tx`. Confirmed.

##### 4.2 Which privileged mutations reach the hash chain, and how

| Mutation | Event stored? | Hash-chain entry? | Actor recorded |
|---|---|---|---|
| 9040 ban | no (`ingest.rs:1582-1586`) | **NO** | — |
| 9041 unban | no | **NO** | — |
| 9042 timeout | no | **NO** | — |
| 9043 untimeout | no | **NO** | — |
| 9044 resolve report | no | **NO** | — |
| 9030 add member | no (`ingest.rs:1811-1816`) | **NO** | — |
| 9031 remove member | no | **NO** | — |
| 9032 change role | no | **NO** | — |
| 9033 set workspace icon | no | **NO** | — |
| 1984 report | no (`ingest.rs:1563-1569`) | **NO** | — |
| 42000 product feedback | no (`ingest.rs:1567-1577`) | **NO** | — |
| 30350 push lease | yes (atomic with lease, `push_lease.rs:563`) | **NO** — ingest returns at `:2199` before the storage path that dispatches | — |
| **9035/9036 identity archive** | **yes** (falls through, `ingest.rs:1909-1912`) | **YES** — `EventCreated` | authenticated request signer |
| Moderation notice DM (kind:9) | yes (`moderation_notices.rs:174`) | **YES** — `EventCreated` | **relay pubkey** (`moderation_notices.rs:178`) |
| `"{host} Moderation"` kind:0 | yes, on first insert (`moderation_notices.rs:206`) | **YES** — `EventCreated`, only when `was_inserted` (`:207-211`) | relay pubkey |
| Workflow `send_message` (kind:9) | yes (`workflow_sink.rs:339`) | **YES** — `EventCreated` | **workflow owner**, not the relay key (`workflow_sink.rs:355`; rationale `handlers/event.rs:584-590`) |
| Operator community provisioning | n/a (HTTP) | **NO** | — |
| NIP-43 / NIP-IA announcement events emitted by `side_effects` | yes | yes, as `EventCreated` | per `side_effects` call |

**Net:** of the 14 privileged inbound kinds this module owns, **12 produce no hash-chain entry at all**. Bans, unbans, timeouts, role changes, member removals, ownership-affecting provisioning, and report resolutions are unauditable through `buzz-audit`. The only durable trail for moderation is the separate `moderation_actions` table (written by `moderation_commands.rs:529` only) — which is **not** hash-chained and therefore not tamper-evident. Relay-admin mutations (9030–9033) have **no** durable audit trail of any kind: no `moderation_actions` row, no event row, no hash-chain entry — only `tracing::info!` lines (`relay_admin.rs:164`, `:203-209`, `:268-272`, `:327-332`).

##### 4.3 Audit transport failure mode

`dispatch_persistent_event` sends into a bounded channel of capacity 1000 using `.send().await`, so backpressure propagates rather than silently dropping (`handlers/event.rs:548-576`). DB write failures inside the audit worker are logged but **not retried** (`:556-558`). A closed channel logs `Audit channel closed — entry lost` (`:575-577`). `state.audit_tx` being `None` skips auditing entirely (`:548-550`).

---

#### 5. `buzz-media` / S3

Three distinct integration points, all through `state.media_storage`:

| Consumer | Call | Purpose |
|---|---|---|
| `report.rs:66-71` | `get_sidecar(tenant, sha_hex)` | tenant-scoped blob existence check for `x`-tag report targets |
| `product_feedback.rs:35` | `verify_imeta_blobs(tenant, &imeta_tags, &state.media_storage)` | attachment verification before persisting feedback |
| `storage_sweep.rs` via `main.rs:1462-1470` | `list_page(token, 1000)` folded by `buzz_media::fold_bucket_listing` | whole-bucket inventory |

##### 5.1 Failure modes

**`report.rs`** — `map_err(|_| "invalid: report target blob not found")` (`:70`) collapses *every* failure class into "not found". Documented as a known Phase-1 limitation because the sidecar API exposes no typed not-found-vs-transient distinction (`:66-69`). A MinIO/S3 outage therefore tells reporters their blob does not exist.

**`product_feedback.rs`** — imeta errors propagate as `String` from `crate::api::validate_imeta_tags` / `verify_imeta_blobs` (`:34-35`) and reject the whole feedback submission.

**`storage_sweep`** — five distinct failure classes, all meaning "keep the old snapshot, never a partial one" (`buzz-media/src/bucket_index.rs:337-338`):

| `SweepError` | Cause | Site |
|---|---|---|
| `CapExceeded { seen, cap }` | cumulative object count exceeded `BUZZ_STORAGE_SWEEP_MAX_OBJECTS` | `bucket_index.rs:342`, raised `:394` |
| `Storage(MediaError)` | the S3/MinIO LIST call failed | `bucket_index.rs:345` |
| `Timeout(Duration)` | whole attempt exceeded `BUZZ_STORAGE_SWEEP_TIMEOUT_SECS`; **constructed by the relay**, not by the fold (`storage_sweep.rs:251`) | `bucket_index.rs:353` |
| `MalformedPage` | `is_truncated=true` with no continuation token — unresumable | `bucket_index.rs:357`, raised `:406` |
| task panic | `JoinError` | `storage_sweep.rs:194-202` |

All five increment `failures_total` and set `last_attempt.ok = false`; only `CapExceeded`/`Storage`/`Timeout`/`MalformedPage` log the operator-actionable hint "verify s3:ListBucket (or MinIO list) permission is granted on the bucket" (`storage_sweep.rs:176-181`).

##### 5.2 Credential coupling

The sweep uses the **same** `MediaStorage` instance as upload/download, therefore the same credentials: `BUZZ_S3_ACCESS_KEY` / `BUZZ_S3_SECRET_KEY` (`config.rs:622-625`). `list_page` is called with an empty prefix (`buzz-media/src/storage.rs:250`), i.e. whole-bucket listing across every tenant. There is no separate read-only or list-only credential and no per-tenant prefix restriction. Adding storage metrics therefore requires granting `s3:ListBucket` to the relay's read-write media key.

---

#### 6. APNs push gateway (outbound HTTPS)

##### 6.1 Client construction and destination

| Property | Value | Site |
|---|---|---|
| HTTP client | one `reqwest::Client` per worker, built once | `push_runtime.rs:313-316` |
| Timeout | `state.config.push_gateway_timeout` — applied as a whole-request `reqwest` timeout | `push_runtime.rs:314` |
| Timeout value | `BUZZ_PUSH_GATEWAY_TIMEOUT_MS`, default **2000 ms**, range-validated `100..=10000` | `config.rs:759-772` |
| Destination | `config.push_gateway_delivery_url` (`Option<url::Url>`) | `push_runtime.rs:422-424` |
| Default destination | `https://push.buzz.xyz/v1/deliveries/apns` | `config.rs:339`, `:755-758` |
| URL validation | scheme must be `https`, host required, no userinfo, path must be exactly `/v1/deliveries/apns`, no query, no fragment | `config.rs:341-361` |
| Disable | set `BUZZ_PUSH_GATEWAY_DELIVERY_URL` to an **empty** string | `config.rs:753` |
| Auth | NIP-98 kind-27235 event, base64'd into `Authorization: Nostr …`, with `u`/`method`/`payload`(sha256 of body)/`nonce` tags | `push_runtime.rs:551-565` |

**Failure mode: the client build `.expect("push HTTP client")` panics** (`push_runtime.rs:316`). This runs inside a `tokio::spawn`ed task, so a panic here silently kills the delivery worker while the matcher keeps enqueuing wakes — an unbounded outbox with no consumer, and no restart.

##### 6.2 Retry and invalidation semantics

| Condition | Action | Site |
|---|---|---|
| 2xx + `{"status":"accepted"}` | `complete_push_wake` | `:434-441` |
| 2xx + other/unparseable body | `fail_push_wake` (terminal) | `:442-447` |
| 410 + `invalid_endpoint{generation}` matching the wake | `disable_push_endpoint`, then `fail` | `:452-473` |
| 410 with mismatched generation | log only, then `fail` — **stale 410 cannot kill a rotated lease** | `:456-465` |
| 410 with unparseable body | `warn!("invalid closed-protocol 410 response")`, then `fail` | `:467` |
| 503 + `retry{retry_after_seconds>0}` | `retry_or_fail(that value)` | `:474-484` |
| 503 otherwise | `retry_or_fail(2)` | `:478-483` |
| 429 | `retry_or_fail(2)` — `Retry-After` header ignored | `:485-487` |
| 404 with `attempt > 1` | `complete_push_wake` — replay of a burned request id treated as delivered | `:488-497` |
| `is_timeout()` or `is_connect()` | `retry_or_fail(2)` | `:498` |
| anything else (incl. 4xx, TLS errors) | `fail_push_wake` (terminal) | `:499-503` |

Backoff: `delay * 2^(attempt-1)` clamped at `2^6 = 64×`, terminal at `MAX_ATTEMPTS = 8` (`push_runtime.rs:531-550`, const `:17`). Worst case with `delay=2`: 2, 4, 8, 16, 32, 64, 128, then fail — roughly 4 minutes.

**Every DB call in the response-handling path discards its result with `let _ =`** (`push_runtime.rs:436`, `:443`, `:455`, `:469`, `:492`, `:500`, `:534`, `:540`). A failed `complete_push_wake` therefore leaves the row claimed; recovery depends on the 30 s claim lease expiring and the wake being re-claimed — which is safe (idempotent via the request id) but invisible.

**SSRF assessment: not exposed.** The destination is operator config validated to a fixed path; the client-controlled `endpoint_grant` travels in the JSON body (`push_runtime.rs:507-515`), never in the URL. `reqwest` default redirect behaviour is not overridden, so a redirect from the configured gateway would be followed — a residual risk bounded by trusting the configured host.

##### 6.3 Counterpart crate

`crates/buzz-push-gateway/` implements the other side (its own `AppProfile` enum at `model.rs:14`, APNs client at `apns.rs`, grant model at `grant.rs`). It is not part of this module group; the wire contract between them is the `DeliveryRequest`/`DeliveryResponse` pair in `push_runtime.rs:31-51` and is not shared via a common crate — the two definitions are independent.

---

#### 7. `buzz-workflow`

`RelayActionSink` is the relay's implementation of `buzz_workflow::action_sink::ActionSink` (`workflow_sink.rs:172`). Wiring order matters and is documented: constructed after `AppState` (which owns `sub_registry` and `conn_manager`) and **before** the cron loop starts (`main.rs:591-597`).

Cycle avoidance: `AppState → WorkflowEngine → ActionSink → AppState` would be an `Arc` cycle, so the sink holds `Weak<AppState>` (`workflow_sink.rs:159-161`, rationale `:150-155`). Upgrade failure is mapped to `ActionSinkError::Database("relay is shutting down")` (`:186-188`).

Error surface: `ActionSinkError` has 6 variants (`buzz-workflow/src/action_sink.rs:11-29`) and **all of them collapse into a single `WorkflowError::WebhookError`** via `From` (`:31-34`). A channel-not-found, an archived channel, an access denial, and a genuine DB outage are therefore indistinguishable in workflow run output.

Failure modes reaching the workflow engine from this module:

| Cause | `ActionSinkError` | Site |
|---|---|---|
| shutting down | `Database` | `workflow_sink.rs:186-188` |
| community not mapped to a host | `Database` | `:205-210` |
| blank text | `EmptyContent` | `:212-214` |
| malformed channel UUID | `InvalidInput` | `:217-218` |
| channel missing | `ChannelNotFound` | `:225-230` |
| channel archived | `ChannelArchived` | `:232-236` |
| bad author pubkey hex | `InvalidInput` | `:238-240` |
| owner not a member of a non-open channel | `InvalidInput` | `:247-251` |
| any DB failure (5 calls) | `Database` | `:203`, `:229`, `:246`, `:277`, `:283`, `:342` |
| tag parse / signing failure | `EventBuild` | `:260-267`, `:292-294`, `:305` |

`buzz-workflow` also reaches the relay **outside** this sink for `add_reaction`, via its own HTTP client to `{BUZZ_RELAY_BASE_URL}/api/messages/{id}/reactions` (`buzz-workflow/src/executor.rs:885-919`). That route is not registered in `router.rs`, so this integration is permanently broken — see the features aspect.

---

#### 8. `buzz-search`

Reached only transitively through `dispatch_persistent_event` for the two relay-signed kind:9 paths (`moderation_notices.rs:178`, `workflow_sink.rs:351`). Indexing failures are absorbed inside `dispatch_persistent_event` and never surface here; `workflow_sink` additionally discards the whole result.

Consequence: a moderation notice DM is indexed into full-text search like any other message, so a moderator's `public_reason` becomes searchable by the recipient.

---

#### 9. `buzz-sdk` (NIP-OA)

Single integration point: `buzz_sdk::nip_oa::verify_auth_tag(auth_tag_json, &target_pubkey)` (`identity_archive.rs:320-326`), called twice per owner-consent request — once for the request's own `auth` tag (`:261`) and once for the target's live kind:0 `auth` tag (`:291`).

Failure mode: any verification error becomes a client-visible string via `e.to_string()` (`:324`), prefixed as `invalid request auth tag: {e}` (`:262`) or `invalid live kind:0 auth tag: {e}` (`:292`) — SDK-internal error text is exposed to the client.

Test helpers additionally use `nip_oa::compute_auth_tag` and `parse_auth_tag` (`identity_archive.rs:479-481`).

---

#### 10. Prometheus / `metrics`

| Metric | Type | Emitter | Labels |
|---|---|---|---|
| `buzz_channels_created_total` | counter | `moderation_notices.rs:113-118` | `community` (host), `type="dm"` |
| `buzz_storage_sweep_ok` | gauge | `storage_sweep.rs:293` | — |
| `buzz_storage_sweep_failures` | gauge (deliberately not `_total` — process-local, resets on failover, `:294-297`) | `storage_sweep.rs:298` | — |
| `buzz_storage_sweep_duration_seconds` | gauge | `:300` | — |
| `buzz_storage_sweep_age_seconds` | gauge | `:311-312` | — |
| `buzz_total_storage_bytes` / `_objects` | gauge | `:315-322` | `kind` ∈ {`physical`,`logical`} |
| `buzz_storage_orphan_blob_bytes`, `_orphan_blobs`, `_orphan_sidecars`, `_multi_variant_shas`, `_multi_variant_bytes`, `_unknown_key_bytes`, `_unknown_key_objects` | gauge | `:324-330` | — |
| `buzz_community_storage_bytes` / `_objects` | gauge | `:339-342`, zeroed at `:125-134` | `community` (host label) |
| `buzz_storage_unmapped_community_bytes` | gauge | `:347` | — |

`storage_sweep` emits **no** counters and no histograms — even sweep duration is a gauge, so percentile analysis across sweeps is not possible.

`push_runtime` emits **zero** metrics: no wake-enqueued counter, no delivery-outcome counter, no gateway-latency histogram. Delivery health is observable only through `warn!`/`error!` log lines.

---

#### 11. Integration risk summary

| Risk | Integration | Evidence |
|---|---|---|
| Delivery worker dies permanently on HTTP client build failure | reqwest | `push_runtime.rs:316` `.expect` inside a `tokio::spawn` |
| Every push DB result discarded (`let _ =`) — silent state divergence | `buzz-db` | 8 sites, `push_runtime.rs:436`…`:540` |
| Push lease DB error is undiagnosable | `buzz-db` | `push_lease.rs:572` `map_err(\|_\| …)` |
| 12 of 14 privileged kinds produce no hash-chain entry | `buzz-audit` | §4.2 |
| Relay-admin mutations have no durable audit trail at all | `buzz-audit` + `buzz-db` | `relay_admin.rs` writes only `tracing::info!` |
| Ban disconnect fan-out is fire-and-forget; reads stay open on other pods | `buzz-pubsub` | `state.rs:1043-1047` |
| S3 outage is reported to reporters as "blob not found" | `buzz-media` | `report.rs:66-70` |
| Storage sweep needs `s3:ListBucket` on the read-write media credential, whole-bucket | `buzz-media` | `config.rs:622-625`, `buzz-media/src/storage.rs:250` |
| All 6 `ActionSinkError` variants collapse to one `WorkflowError` | `buzz-workflow` | `action_sink.rs:31-34` |
| Workflow message fan-out failure invisible to the run | `buzz-pubsub`/`buzz-search` | `workflow_sink.rs:351` `let _ =` |
| Push wire contract duplicated, not shared | `buzz-push-gateway` | `push_runtime.rs:31-51` vs `buzz-push-gateway/src/model.rs` |
| Zero observability on push delivery | `metrics` | no `metrics::` calls in `push_runtime.rs` |


## Module: buzz-relay — huddle audio, tunnel & conformance seam (`crates/buzz-relay/src`)
### Aspect: Integrations

---

#### 1. Dependency map

| Integration | Reached via | Call sites in this group | Failure mode |
|---|---|---|---|
| `buzz-relay-mesh` (iroh/QUIC) | `MeshHandle.transport: Arc<dyn RelayPeerTransport>`, `MeshRuntime`, `MeshEndpoint` | `mesh_boot.rs:423`, `:475`, `:502`, `:505`; `join.rs:1675`, `:1762`; `mesh.rs:277`; `reliable.rs:120` | §2 |
| `buzz-pubsub` / Redis pub-sub | `state.pubsub.publish_event` | `handler.rs:1322-1331` | §3.1 |
| Redis (deadpool) — session directory | `SessionDirectory` over `deadpool_redis::Pool` | `directory.rs:214`, `:249`, `:283`, `:313`, `:331`, `:359` | §3.2 |
| Redis — mesh ready registry | `ReadyRegistry` | `mesh_boot.rs:445`, `:461`, `:468` | §3.3 |
| `buzz-db` (Postgres) | `state.db.*` | `handler.rs:158`, `:389`, `:838`, `:1164`, `:1179`, `:1197`, `:1212`, `:1219`, `:1281` | §4 |
| `buzz-auth` | `generate_challenge`, `state.auth.verify_auth_event` | `handler.rs:176`, `:222` | §5 |
| `buzz-core` | `CommunityId`, `TenantContext`, `StoredEvent` | throughout | compile-time only |
| `buzz-conformance` | `Tracer`, `TraceStep`, `TraceAction`, `AbstractState`, … | `conformance/mod.rs:38-44`; `tracers.rs:12` | §6 |
| `nostr` crate | `EventBuilder`, `Kind`, `Tag`, `Keys` | `handler.rs:1240-1272`; `mesh_boot.rs:415`, `:452` | signing failure → `warn!` + skip (`handler.rs:1270-1273`) |
| `postcard` | `HuddleControlMsg` codec | `join.rs:1007`, `:1012` | `MeshError::Encode`/`Decode` |
| `dashmap` | rooms, peers, generation floor, owner registry | `room.rs:490`, `:163`; `mesh.rs:90`; `join.rs:601` | none (in-process) |
| `metrics` | one counter | `directory.rs:483` | none |
| `mesh-llm-sdk` / `mesh-llm-host-runtime` | **dev-dependencies only** | `Cargo.toml:87-88`; consumed by `examples/mesh_agent_e2e.rs:25,48`, `examples/mesh_serve_client_smoke.rs:29,44-45,53`, `examples/mesh_serve_smoke.rs:13`, `examples/mesh_stack_smoke.rs:53` | §7 |

**No audio codec or media library is linked.** `crates/buzz-relay/Cargo.toml` has no
`opus`, `webrtc`, `cpal`, `dasp`, `symphonia`, `stun`, or `turn` entry — verified by
grep. The relay treats every audio byte as opaque.

---

#### 2. `buzz-relay-mesh` — the deepest coupling

##### 2.1 Surfaces consumed

| Type / fn | Where used |
|---|---|
| `RelayPeerTransport::{send_datagram, open_session_stream, set_inbound}` | `join.rs:1675`, `:1762`; `mesh.rs:277`; `reliable.rs:120`; `mesh_boot.rs:505` |
| `InboundHandler::{on_datagram, on_session_stream}` | implemented by `MeshInboundDispatcher`, `mesh_boot.rs:91-130` |
| `MeshStream::{send_frame, recv_frame, finish}` | `join.rs:1010`, `:1179`, `:1783`; `reliable.rs:283`, `:317`, `:334` |
| `MeshStreamFrame` (4 variants) | `join.rs`, `reliable.rs`, `mesh_boot.rs` |
| `FencedHeader`, `MeshDatagram`, `Profile`, `GoodbyeReason`, `RuntimeId`, `StreamHello`, `StreamRole`, `MeshError` | throughout |
| `MeshEndpoint::bind`, `endpoint.ip_addrs()`, `endpoint.runtime_id()` | `mesh_boot.rs:423`, `:392`, `:432` |
| `MeshRuntime::{start, reconcile_now, membership, clone}` | `mesh_boot.rs:475`, `:478`, `:173`, `:501-502` |
| `MeshMembership::{new, with_expected_relay_pubkey}`, `RelayMeshMembership`, `MeshStatus` | `mesh_boot.rs:441-443`, `:501`, `:172-174` |
| `ReadyRegistry`, `ReadyRecord`, `GossipRecord`, `spawn_registry_heartbeat` | `mesh_boot.rs:445-472` |
| `WIRE_VERSION`, `wire::MAX_STREAM_FRAME` | `mesh_boot.rs:367`; `reliable.rs:945` |

`MeshHandle` is the sole gateway: `AppState::mesh()` returns `Option<&MeshHandle>`
(`state.rs:812-814`), and every consumer branches on that `Option`
(`handler.rs:306`, `:449`, `:577`, `:875`).

##### 2.2 Trust boundary between the two crates

- Inbound mesh connections are gated on `is_known_peer`, which requires a
  Redis ready-registry record (`buzz-relay-mesh/src/runtime.rs:275-283`, `:309-320`).
  Records are attested against the relay signing key
  (`MeshMembership::with_expected_relay_pubkey`, `mesh_boot.rs:442-443`).
- The `from: RuntimeId` handed to every handler is the **authenticated QUIC peer
  identity** (`runtime.rs:392-399`, `:412`), which is what lets
  `accept_inbound` assert `hello.sender == from` (`join.rs:1060-1065`,
  `reliable.rs:143-148`).
- The mesh layer itself does **no** Redis fencing. Every fence check in this group
  is performed by the *consumer*: `join.rs:1231-1245` (control frames),
  `reliable.rs:381-385` (reliable frames), `mesh.rs:212-220` (media, local floor
  only — see security).

##### 2.3 Failure modes

| Failure | Behaviour |
|---|---|
| `MeshEndpoint::bind` fails with mesh on | **Fatal at boot** — `anyhow` error propagated from `mesh_boot.rs:423-431` to `main.rs:442` |
| Peer unreachable when dialing an owner | `DialError::Mesh` → WS `huddle_owner_unreachable`; the joining client gets a clean error, and `cleanup_if_empty` runs (`handler.rs:487-503`) |
| `send_datagram` fails (disconnected peer, oversize) | `debug!` and continue — audio drop-on-error (`join.rs:1762-1765`, `mesh.rs:277-282`) |
| `send_frame` on the control stream fails | breaks the serve loop with `Err`, then unconditional peer teardown (`join.rs:1245-1254`, `:1345-1367`) |
| Owner pod dies mid-call | ingress reader sees a bare close → `StreamClosed` → cancel + `fence.forget` (`join.rs:1604-1610`, `handler.rs:707-714`) |
| Owner drains (SIGTERM) | `Goodbye(Draining)` → ingress rejoins; local owner clients closed by the drain watcher (`join.rs:1157-1161`, `handler.rs:735-748`) |
| Traffic arrives before a profile handler is registered | logged and dropped; the peer's fenced retry is safe (`mesh_boot.rs:52-55`, `:92-100`, `:122-129`) |
| `RealtimeMedia` arrives as a stream | rejected without routing (`mesh_boot.rs:113-121`) |
| MTU overflow on a media datagram | the sink drops it with a `debug!`; the comment explicitly says MTU prevention "is the ship-gate's job" (`mesh.rs:278-281`) — i.e. **no runtime MTU check exists** |

---

#### 3. Redis — three independent uses

##### 3.1 `buzz-pubsub` (lifecycle-event fan-out)

Single call: `state.pubsub.publish_event(tenant, EventTopic::Channel(parent), &event)`
(`handler.rs:1322-1325`). Topic is the **lifecycle parent channel**, not the
ephemeral huddle channel.

Failure: the event is already persisted and already fanned out locally, so a publish
error only means other pods miss the live delivery. `local_event_ids` is invalidated
so a later echo is not suppressed (`handler.rs:1326-1330`), then `warn!`.

##### 3.2 Session directory (ownership arbiter)

`deadpool_redis::Pool`, shared with the rest of the relay
(`state.redis_pool.clone()` → `mesh_boot.rs:442`, `:512`). Four Lua scripts
(`directory.rs:20-79`) + two plain `GET`s (`directory.rs:313`, `:331`).

| Failure | Behaviour |
|---|---|
| pool checkout fails | `DirectoryError::Pool` → `MeshError::Transport` at every `HuddleDirectory` boundary (`join.rs:114`, `:139`, `:158`, `:172`) |
| Redis unreachable during `resolve_join` | join fails → WS `join_rejected` (`handler.rs:342-355`). **Huddles become unjoinable when Redis is down and mesh is on** |
| Redis unreachable during renewal | `Err` → treated as **owner loss**, `lost` fires, every local owner client is closed for rejoin (`join.rs:521-529`, `handler.rs:756-765`). A Redis blip therefore drops every cross-pod huddle on the pod |
| Redis unreachable during owner-side `validate` | non-fence error → the whole `HuddleControl` stream is torn down (`join.rs:1240-1244`) |
| Malformed lease value in Redis | `MalformedLease` → `Transport` error → join failure; never a silent default (`directory.rs:495-531`) |
| Lease key expires while the pod still serves peers | Redis stops naming the pod; the next renew returns `Lost`. Between expiry and the next renew tick (up to 10 s) the pod keeps fanning out with a dead lease — local WS peers have **no per-frame fence** |

##### 3.3 Mesh ready registry

`ReadyRegistry::new(redis_pool, config.mesh.registry_refresh)` (`mesh_boot.rs:445`),
first `publish_ready` at `:461`, then `spawn_registry_heartbeat` gated on
`!shutting_down` (`mesh_boot.rs:466-472`).

Failure: the **first** publish failing is fatal to boot (`mesh_boot.rs:459-463`) —
"if Redis can't take the attested record, peers can never find us". Later heartbeat
failures are internal to `buzz-relay-mesh`.

---

#### 4. `buzz-db` (Postgres)

| Call | Line | Purpose | Failure mode |
|---|---|---|---|
| `is_community_active(community)` | `handler.rs:158` | community lifecycle gate | closure result drives `run_registered_community_connection`; a DB error there rejects the connection |
| `get_channel(community, channel)` | `handler.rs:1164` (in `ensure_membership`) | load channel, reject archived | `Err` → `"db error: {e}"` → WS `not a member` |
| `get_channel(community, channel)` | `handler.rs:389` | post-`get_or_create` archived re-check | `Err` → **fail closed**, silent teardown (`handler.rs:404-410`) |
| `huddle_started_link_exists(community, parent, channel, created_by)` | `handler.rs:1179-1186` | verify a creator-signed kind-48100 link before trusting a client-supplied parent | `Err` → `"db error"`; `false` → `"ephemeral channel is not linked to claimed parent"` |
| `is_member_cached(community, channel, pubkey)` ×2 | `handler.rs:1197`, `:1212` | membership fast path + parent check | `Err` → `"db error"` |
| `add_member(community, channel, pubkey, Member, Some(created_by))` | `handler.rs:1219-1227` | ephemeral auto-add | `Err` → `"auto-add failed: {e}"` → join refused |
| `invalidate_membership(tenant, channel, pubkey)` | `handler.rs:1228` | cache coherence after auto-add | infallible |
| `archive_channel(community, channel)` | `handler.rs:838` | auto-end | `Err` → `clear_ended()`, huddle stays alive, no 48103 (`handler.rs:840-845`) |
| `insert_event(community, &event, Some(parent))` | `handler.rs:1281-1284` | persist lifecycle event | duplicate → skip fan-out; `Err` → fan out from memory anyway (`handler.rs:1285-1307`) |

Note the **double `get_channel`** on every join (`:1164` and `:389`) — two round
trips for the same row, deliberate to close a race but uncached.

---

#### 5. `buzz-auth`

- `generate_challenge()` (`handler.rs:33`, `:176`) — the challenge nonce.
- `state.auth.verify_auth_event(event, &challenge, &relay_url)` (`handler.rs:220-238`)
  — full NIP-42 verification, identical to the main relay door. The returned
  `auth_ctx.pubkey` is the only identity used downstream (`handler.rs:240-242`).
- `crate::handlers::auth::extract_auth_tag_json(&event)` (`handler.rs:217`) —
  NIP-OA tag pulled out *before* the event is consumed by the verifier.
- `crate::api::bridge::nip42_expected_relay_url(&state.config.relay_url, &tenant)`
  (`handler.rs:219`) — per-tenant expected relay URL.
- `crate::api::relay_members::enforce_relay_membership` (`handler.rs:244-262`) —
  no-op unless `require_relay_membership` is on (`api/mod.rs:67`, `:130-131`).

Failure: any verifier rejection → WS `{"type":"error","message":"auth failed"}` and
close. No retry, no second challenge.

---

#### 6. `buzz-conformance`

Consumed as a pure schema + trait crate. `conformance/mod.rs:38-44` re-exports 11
items; `tracers.rs:12` imports `TraceStep` and `Tracer`.

| Direction | Detail |
|---|---|
| relay → crate | `AppState.tracer: Arc<dyn buzz_conformance::Tracer>` (`state.rs:620`); every `record` call |
| crate → relay | nothing — the checker (`buzz-conformance/src/checker.rs`) consumes JSONL offline |

Failure modes: **none can reach a request path.** `Tracer::record` returns `()`
(`buzz-conformance/src/lib.rs:317`), so an emit cannot fail, cannot block, and
cannot apply backpressure. `JsonlTracer` swallows every I/O error
(`tracers.rs:63-71`) and recovers a poisoned mutex via `into_inner`
(`tracers.rs:57-60`). Because production binds `NoopTracer` (`state.rs:798`), the
integration is inert at runtime — see features §4.1.

One duplication: `buzz_conformance::NoopTracer` (`buzz-conformance/src/lib.rs:323`)
exists and has **zero users**; the relay defines and uses its own
(`tracers.rs:20-24`).

---

#### 7. `mesh-llm` — a name collision, not a mesh integration

`crates/buzz-relay/Cargo.toml:87-88` pins two **dev-dependencies** to
`git = "https://github.com/Mesh-LLM/mesh-llm.git", tag = "v0.73.1"`:
`mesh-llm-sdk` (features `client`, `serving`) and `mesh-llm-host-runtime`
(feature `dynamic-native-runtime`).

Consumers are exactly the five files in `crates/buzz-relay/examples/`:

| Example | Uses |
|---|---|
| `mesh_agent_e2e.rs` | `mesh_llm_sdk::{serve, MeshDiscoveryMode}` (`:25`), `mesh_llm_host_runtime::initialize_host_runtime` (`:48`) |
| `mesh_serve_client_smoke.rs` | `mesh_llm_sdk::{client, serve, MeshDiscoveryMode}` (`:29`), `native_runtime_cache` / `CURRENT_MESH_VERSION` (`:44-45`), `initialize_host_runtime` (`:53`) |
| `mesh_serve_smoke.rs` | `mesh_llm_sdk::{serve, MeshDiscoveryMode}` (`:13`) |
| `mesh_stack_smoke.rs` | `mesh_llm_host_runtime::models::download_model_ref_with_progress_details` (`:53`) |
| `mesh_admission_smoke.rs` | process-global mesh-llm state note (`:16`); no direct import |

This is **local LLM inference / model serving**, entirely unrelated to
`buzz-relay-mesh` (the inter-relay QUIC mesh in this group). Two different things
called "mesh" inside one crate, both spelled `mesh_*` in file names. Risk profile:

- A `git`-pinned dependency by **tag** (not commit SHA) — tags are mutable, so the
  build is not reproducible against a retagged upstream.
- Present in `[dev-dependencies]`, so it does not ship in the relay binary, but it
  **does** enter the dev/CI dependency graph and lockfile for anyone running
  `cargo test -p buzz-relay`.
- `mesh_stack_smoke.rs:31` requires manual sync with
  `buzz_lib::mesh_llm::MESH_WORKER_STACK_SIZE` in the desktop crate — a
  cross-crate constant duplicated by comment, not by code.

---

#### 8. Inbound/outbound integration matrix for one cross-pod huddle frame

```
Client A (pod 1, non-owner)                        Client B (pod 2, OWNER)
   │ WS binary [v2 hdr][Opus]                          
   ├──────────────────────────────► handler.rs:1019 forward_media
   │                                 join.rs:1758 media_datagram
   │                                 [ownerIdx][v2 hdr][Opus] + FencedHeader
   │                                 transport.send_datagram ──► iroh/QUIC
   │                                                              │
   │                        mesh_boot.rs:242 dispatcher.on_datagram
   │                        mesh.rs:212 GenerationFloor::check   ◄─┘
   │                        mesh.rs:221 get_unambiguous_by_channel
   │                        mesh.rs:247 room.deliver_prefixed
   │                                     └─► B's audio_tx ─► WS binary
   │
   │  B speaks: room.broadcast_frame (room.rs:398) puts the prefixed frame
   │  into A's *remote* AudioPeer.audio_tx, drained by
   │  spawn_remote_peer_sink (mesh.rs:262) ─► datagram ─► pod 1
   │  ─► mesh.rs:247 deliver_prefixed ─► A's WS
```

Redis is consulted **only** at join (`resolve_join_owner_ready` → `owner_of` /
`acquire` / `validate`), at owner-side `RegisterPeer` (`validate`), and every 10 s
by the renewer. It is **never** consulted per media frame.


## Module: buzz-relay-mesh (`crates/buzz-relay-mesh`)
### Aspect: Integrations

Five external couplings: **iroh/QUIC**, **Redis**, **nostr/secp256k1**,
**postcard**, and **`buzz-relay`** (the only consumer). Notably **`buzz-core` is
NOT a dependency** — see §4.

Full dependency list (`Cargo.toml:11-26`): `tokio`, `serde`, `serde_json`,
`postcard`, `iroh`, `redis`, `deadpool-redis`, `thiserror`, `tracing`, `uuid`,
`hmac`, `sha2`, `hex`, `nostr`, `bytes`, `futures-util`.
Dev-deps (`:28-30`): `tokio` (test-util), `proptest`.

Verified usage census (`use`-site count per crate):

| Dep | `use` sites | Status |
|---|---|---|
| `tokio` | 36 | used |
| `tracing` | 28 | used |
| `iroh` | 12 | used |
| `nostr` | 11 | used |
| `uuid` | 7 | used |
| `postcard` | 6 | used |
| `redis` / `deadpool-redis` | 5 / 5 | used |
| `serde` / `serde_json` | 4 / 4 | used (`serde_json` only in `registry.rs`) |
| `sha2`, `hex`, `bytes`, `thiserror` | 1 each | used |
| **`hmac`** | **0** | **unused dependency** (`Cargo.toml:20`) |
| **`futures-util`** | **0** | **unused dependency** (`Cargo.toml:25`) |
| **`proptest`** | **0** | **unused dev-dependency** (`Cargo.toml:29`) |

---

#### 1. iroh / QUIC

##### Version — delta against the brief

The workspace **requirement** is `iroh = { version = "1.0.0-rc.0",
default-features = false, features = ["tls-ring"] }` (`Cargo.toml:68`, comment
`:67` "Inter-relay mesh transport (buzz-relay-mesh)"). The crate takes it via
`workspace = true` (`crates/buzz-relay-mesh/Cargo.toml:15`).

**`Cargo.lock` resolves iroh to `1.0.2` from crates.io**
(`Cargo.lock:3902-3905`, checksum `5fca9b4b462c…`), not to the rc. Because
`^1.0.0-rc.0` admits `1.0.2`, the pre-release string in the manifest is a
*floor*, not a pin: the built artifact uses a stable 1.0.x release. The manifest
string is nonetheless misleading and should be `"1.0"` — see `-debt.md` D-05.

`iroh` is the **only** crate in the workspace using it (grep: no other
`Cargo.toml` references it).

##### Surface consumed

| iroh item | Used at |
|---|---|
| `Endpoint`, `Endpoint::builder`, `.bind()` | `endpoint.rs:3`, `:33-41` |
| `iroh::endpoint::presets::Minimal` | `endpoint.rs:33` |
| `SecretKey::generate()` / `SecretKey::from_bytes` | `endpoint.rs:20`; tests `endpoint.rs:158` |
| `PublicKey::as_bytes` / `from_bytes` | `endpoint.rs:97`, `:101` |
| `RelayMode::Disabled` | `endpoint.rs:36` |
| `.alpns(vec![ALPN.to_vec()])` | `endpoint.rs:35` |
| `.bind_addr(SocketAddr)` | `endpoint.rs:37` |
| `EndpointAddr`, `EndpointAddr::from_parts`, `TransportAddr::Ip` | `endpoint.rs:3`, `:65-68`, `:105-108` |
| `endpoint.accept()` → `Incoming` → `Connection` | `endpoint.rs:74-81` |
| `endpoint.connect(addr, ALPN)` | `endpoint.rs:88` |
| `Connection::alpn / remote_id / max_datagram_size / open_bi / accept_bi / send_datagram / read_datagram` | `peer.rs:50`, `:58`, `:69`, `:80`, `:93`, `:112`, `:120` |
| `endpoint::SendStream::write_all / finish`, `RecvStream::read_exact` | `peer.rs:148-165`, `:171-188` |
| `endpoint::ReadExactError::FinishedEarly` | `peer.rs:172` |

##### Configuration choices with consequences

- **`RelayMode::Disabled`** (`endpoint.rs:36`): no iroh relay servers, no hole
  punching via relays, no DERP fallback. Peers must be **directly IP-reachable** —
  which is why the deployment story is pod-to-pod inside one cluster
  (`advertise_addrs` prefers `POD_IP`, `mesh_boot.rs:398-403`).
- **`presets::Minimal`** (`endpoint.rs:33`): the leanest iroh endpoint preset — no
  discovery services.
- **`tls-ring`** (`Cargo.toml:68`) with `default-features = false`: ring rather than
  aws-lc-rs; consistent with the rest of the workspace's `tls-ring` choices
  (`Cargo.toml:57` redis, `:79` otlp).
- **Identity = iroh node key.** `RuntimeId` *is* `PublicKey::as_bytes()`
  (`endpoint.rs:96-98`), so iroh's TLS peer authentication and the mesh's identity
  model are the same thing. `MeshPeer::from_connection` derives the remote's
  RuntimeId from `conn.remote_id()` (`peer.rs:58`) — an attacker cannot claim a
  RuntimeId they do not hold the key for.

##### Failure modes

| Failure | Behaviour | Site |
|---|---|---|
| bind fails (port in use, bad addr) | `MeshError::Transport` → **fatal relay boot** | `endpoint.rs:38-41`; `mesh_boot.rs:383-390`; `main.rs:442` `?` |
| inbound handshake fails | warn `"mesh: inbound connection failed"`, accept loop continues | `runtime.rs:277-279` |
| `endpoint.accept()` returns `None` (endpoint closed) | accept loop logs and **returns** — no restart, mesh is permanently deaf | `runtime.rs:271-274` |
| dial fails for an addr | warn, try next addr; all exhausted → mark Disconnected, retry in 5 s **with no backoff** | `runtime.rs:340-354` |
| ALPN mismatch on an established conn | `MeshError::Transport("unexpected mesh ALPN …")`, peer not installed | `peer.rs:50-55` |
| peer lacks datagram support | `Transport("peer does not support QUIC datagrams")` on every send | `peer.rs:109` |
| datagram or stream read error | `remove_peer` → tasks aborted, `ConnectionState::Disconnected`; membership entry **kept** | `runtime.rs:359-363`, `:379-383`, `:267-281` |
| oversize frame/datagram | typed `FrameTooLarge` / `DatagramTooLarge`, never truncated | `peer.rs:142-147`, `:178-183`, `lib.rs:218-223` |
| iroh error detail | **flattened to `String`** via `err.to_string()` at 12 sites — the structured iroh error type is discarded, so callers cannot match on cause | `endpoint.rs:38,39,79,91,101`; `peer.rs:82,95,113,123,151,155,163,174,187` |

`iroh` being a pre-1.0-string dependency with a 1.0.x lock resolution means an
`iroh` 1.1 release will be picked up silently by `cargo update`; the API surface
consumed here (`presets::Minimal`, `EndpointAddr`, `TransportAddr`,
`remote_id()`) is broad enough that this is a real upgrade-risk area.

---

#### 2. Redis (ready registry)

Clients: `redis` 1.0 (`Cargo.toml:57`, features `tokio-comp`,
`connection-manager`, `tokio-rustls-comp`) + `deadpool-redis` 0.23 (`:58`).
The crate never creates a pool — it receives one
(`ReadyRegistry::new(pool, refresh)`, `registry.rs:166-168`), and the relay passes
`state.redis_pool.clone()` (`mesh_boot.rs:447`, from `main.rs:443`). Same pool the
rest of the relay uses (`buzz-pubsub`, session directory), so mesh registry traffic
competes for the same connections.

| Operation | Command | Site | Cadence |
|---|---|---|---|
| publish ready | `SET mesh:ready:{id} <json> EX <ttl>` | `registry.rs:188-194` | once at boot (`mesh_boot.rs:459`) then every 15 s (`runtime.rs:602`) |
| clear ready | `DEL mesh:ready:{id}` | `registry.rs:201-204` | on ready→not-ready edge only (`registry.rs:299-302`) |
| scan | `SCAN <cur> MATCH mesh:ready:* COUNT 100` + `GET` per key | `registry.rs:217-228` | every 5 s (`runtime.rs:311`) **plus once per unknown inbound connection** (`runtime.rs:318`) |

Value codec is **JSON** (`serde_json::to_string`, `registry.rs:185` /
`from_str`, `registry.rs:232`) — the only non-postcard payload in the crate,
chosen for operator legibility.

Key namespace `mesh:ready:` (`registry.rs:19`) is **global**: not community-scoped,
not deployment-prefixed. Multi-deployment isolation rests entirely on the
`relay_pubkey` anchor check in `apply_ready_records` (`membership.rs:90-102`),
which counts rejects into `foreign_relay_rejections` (`status.rs:47`).

##### Failure modes

| Failure | Behaviour | Site |
|---|---|---|
| pool `get()` fails | `MeshError::Transport("redis pool: …")` | `registry.rs:270-272` |
| **first publish fails** | **fatal relay boot** — `anyhow!("mesh ready-registry publish failed: …")` | `mesh_boot.rs:456-463` |
| heartbeat publish fails | warn `"mesh: registry heartbeat tick failed"`, loop continues; the record then TTL-expires after 45 s and peers stop dialing us | `runtime.rs:601-603` |
| reconcile scan fails | warn `"mesh: registry scan failed"`, then dial from the (stale, never-evicted) membership table | `runtime.rs:288-292`, `:307-312` |
| inbound-admission scan fails | warn `"mesh: registry rescan on inbound failed"`, fall through to `has_peer` re-check → connection rejected | `runtime.rs:315-320` |
| malformed / key-mismatched / unattested entry | warn + skip, scan continues | `registry.rs:233-247` |
| Redis transport error mid-scan | whole scan aborts with `MeshError::Redis` (`#[from]`) | `registry.rs:225`, `:228` |
| clear fails on shutdown | error propagates from `tick`, logged as a heartbeat warn; record lingers up to TTL | `registry.rs:300`, `runtime.rs:601` |

Redis outage semantics: existing warm connections keep working (transport is
independent of Redis), but (a) no new peers can be discovered, (b) our record
TTL-expires so *new* pods can't find us, and (c) every unknown inbound connection
costs a failed scan. Nothing in this crate degrades gracefully to a static peer
list.

##### CPU/IO amplification

`scan_ready` performs one `GET` **per key** in a serial loop (`registry.rs:230-231`)
— no `MGET`, no pipelining — and one **secp256k1 schnorr verify per record**
(`registry.rs:233-238` → `registry.rs:70-80`). At 5 s cadence with N pods that is
N verifies per pod per 5 s, plus a full extra scan+verify pass for **every inbound
connection from an unrecognised runtime id** (`runtime.rs:309-320`). See
`-security.md` S-04.

---

#### 3. nostr / secp256k1 (attestation)

`nostr` 0.44 (`Cargo.toml:61`, features `nip44`, `nip98`) is used only in
`registry.rs`:

| Item | Site |
|---|---|
| `nostr::Keys::sign_schnorr` | `registry.rs:41` |
| `nostr::PublicKey::from_hex` / `.to_hex()` / `.xonly()` | `registry.rs:57`, `:39`, `:63` |
| `nostr::secp256k1::{Message, XOnlyPublicKey}`, `schnorr::Signature::from_str` | `registry.rs:11-12`, `:68` |
| `nostr::secp256k1::SECP256K1.verify_schnorr` | `registry.rs:81-82` |
| `sha2::Sha256::digest` over the textual preimage | `registry.rs:94` |

The signing key is the relay's Nostr identity, injected from the consumer:
`boot_mesh(..., relay_keypair: &nostr::Keys, ...)` (`mesh_boot.rs:415`) sourced from
`state.relay_keypair` (`main.rs:445`). The same key anchors acceptance
(`mesh_boot.rs:445` → `membership.rs:61-64`).

Failure modes: every parse/convert/verify error becomes a
`MeshError::Transport(format!(...))` string (`registry.rs:56-83`) — five distinct
failure causes collapse into one untyped variant, so a caller cannot distinguish
"bad hex" from "signature forged." Rejections are logged with `runtime_id`
(`membership.rs:96-101`, `:105-108`; `registry.rs:236-246`) and counted only for the
anchor-mismatch case (`foreign_relay_rejections`), not for signature failures.

---

#### 4. `buzz-core` — **not a dependency**

Verified: `crates/buzz-relay-mesh/Cargo.toml` has no `buzz-*` dependency at all, and
no `buzz_core` import exists in `src/`. The mesh crate is deliberately
Buzz-domain-free: no `CommunityId`, no event kinds, no NIP types, no tenant concept.

The tenant boundary is applied **outside** this crate — consumers thread
`buzz_core::CommunityId` through their own layers (`tunnel/reliable.rs:13`,
`audio/join.rs:41`, `api/mesh_demo.rs:29`) and the fenced session directory scopes
by community. Consequence: **the mesh wire format carries no tenant identifier.**
`FencedHeader` is `{session_id, generation, owner_runtime_id}` (`wire.rs:85-93`) —
community scoping is entirely inside the opaque `payload`, recovered by
`recv_validated`/`community_id()` on the relay side
(`mesh_boot.rs:334-341`, `tunnel/reliable.rs`). Cross-tenant isolation on the mesh
therefore depends on consumer discipline, not on the transport contract.

---

#### 5. postcard

`postcard` 1 with `default-features = false, features = ["use-std"]`
(`Cargo.toml:65`). Used at 6 sites: `wire.rs:178` (`to_extend`), `wire.rs:184`
(`from_bytes`), `gossip.rs:63`, `gossip.rs:67`, and the two error `#[source]`
bindings (`lib.rs:68`, `:70`).

Integration risk: postcard's enum encoding is a varint discriminant with no
unknown-variant escape hatch, and **no wire enum is `#[non_exhaustive]`**
(verified: zero occurrences in `src/`). Combined with the ALPN-per-version rule
(`wire.rs:34-36`) the design is "never mix versions," which is sound but leaves the
`WIRE_VERSION` byte and `GOSSIP_PAYLOAD_VERSION` field as belt-and-braces only.
The gossip payload is doubly-encoded (postcard `GossipMessage` inside a postcard
`MeshStreamFrame::Gossip.payload: Vec<u8>`), which costs a length prefix + a copy
per gossip frame but buys independent evolution (`wire.rs:139-141`).

---

#### 6. `buzz-relay` — how the consumer wires it

Declared at `crates/buzz-relay/Cargo.toml:26` (`buzz-relay-mesh = { workspace = true }`),
path-mapped at root `Cargo.toml:135`, member at `Cargo.toml:27`.

Boot sequence (`crates/buzz-relay/src/mesh_boot.rs:412-521`):

1. `config.mesh.enabled` false → `Ok(None)` (`:417-419`).
2. `MeshEndpoint::bind(config.mesh.bind_addr)` (`:383`) — fatal on error.
3. `advertise_addrs` (`:382-403`): `BUZZ_MESH_ADVERTISE_ADDR` → `POD_IP` + actual
   bound port → all endpoint IP addrs.
4. `GossipRecord::new(runtime_id, addrs, PROTO_VERSION)` + static capabilities
   (`:439-441`).
5. `MeshMembership::new(record).with_expected_relay_pubkey(relay_pubkey_hex)`
   (`:444-445`).
6. `ReadyRegistry::new(pool, registry_refresh)`; `ReadyRecord::new(...)`
   (`:447-453`).
7. `registry.publish_ready(&record)` — **fatal on error** (`:456-463`).
8. `spawn_registry_heartbeat(registry.clone(), record, || !shutting_down)`
   (`:467-471`).
9. `MeshRuntime::start(endpoint, membership, Some(registry))` (`:473`) — spawns 3
   loops.
10. `runtime.reconcile_now().await` (`:478`) — dial seeds immediately.
11. Drain watcher task: polls `shutting_down` every 500 ms, then `begin_drain()` +
    `owners.drain_all()` and returns (`:481-497`).
12. `Arc<dyn RelayMeshMembership>` (`:501`, never read) and
    `Arc<dyn RelayPeerTransport>` (`:502`) erased from the runtime.
13. `transport.set_inbound(Box::new(dispatcher.clone()))` (`:509`).
14. Return `MeshHandle` (`:511-520`).

Then `main.rs:455-459` calls `handle.wire_consumers(...)` **before**
`state.mesh.set(handle)` (`main.rs:457`), so the three profile consumers are
registered before the handle is observable; `main.rs:460` is
`unreachable!("mesh handle is set exactly once, right here")`.

Read path: `AppState::mesh()` (`state.rs:812`) — one caller, `router.rs:381`.

Consumer-side integration points:

| Consumer file | Uses |
|---|---|
| `mesh_boot.rs` | `MeshEndpoint`, `GossipRecord`, `ReadyRecord`/`ReadyRegistry`, `MeshMembership`, `MeshRuntime`, `spawn_registry_heartbeat`, `MeshStatus`, both seams, `InboundHandler`, `Profile`, `StreamRole`, `GoodbyeReason`, `WIRE_VERSION` |
| `audio/mesh.rs` | `FencedHeader`, `MeshDatagram`, `RelayPeerTransport`, `RuntimeId` (`audio/mesh.rs:37,56,260`) |
| `audio/join.rs` | full wire set + `MeshStream`, both half traits, `MeshError` fence variants; defines `HuddleControlMsg` as the opaque `Data` payload (`join.rs:797-801`) |
| `tunnel/reliable.rs` | wire set + `MeshStream`; chunks at 1 MiB against the 16 MiB cap (`reliable.rs:26-31`, assert `:945`) |
| `tunnel/directory.rs` | `MeshError` fence variants only (`:378,395,413,430,824,842,870,914`) |
| `api/mesh_demo.rs` | `RelayPeerTransport` + `MeshEndpoint`/`MeshPeer` in tests |
| `config.rs` | `MeshConfig` (`:136`, `:508`) |
| `audio/handler.rs` | reaches the handle via `AppState` (`:308`, `:446`) |

##### Consumer-side failure modes

- Handle absent (`mesh` off) → every consumer path is a no-op by contract
  (`state.rs:624-626`, `mesh_boot.rs:19-20`).
- Traffic arriving before a dispatcher slot is registered → logged and dropped
  (`mesh_boot.rs:99-104`, `:128-134`); documented as a bounded boot-window race made
  safe by the peer's fenced retry (`mesh_boot.rs:53-55`).
- Second registration on a slot → warn + ignored, first handler wins
  (`mesh_boot.rs:72-74`, `:79-81`, `:85-87`).
- `Profile::RealtimeMedia` arriving as a *stream* → rejected as a protocol violation
  (`mesh_boot.rs:118-126`).
- Mesh runtime loops are never shut down (`MeshRuntime::shutdown` uncalled), so on
  SIGTERM the accept/reconcile/gossip loops and the heartbeat task run until process
  exit.

---

#### 7. Not integrated (absences worth recording)

- **No metrics exporter.** The crate has no `metrics` dependency; the documented
  `mesh_fence_rejections_total` (`lib.rs:102-109`) does not exist. All mesh
  observability is the ad-hoc counters in `MeshStatus` plus `tracing` (28 sites).
- **No tracing spans / OTel.** `tracing` is used only for `info!/warn!/debug!`
  events; no `#[instrument]`, no span propagation across the mesh, so a cross-pod
  session cannot be traced end to end.
- **No `buzz-audit`.** Peer admission and rejection decisions are not audited, only
  logged.
- **No k8s API client.** Peer discovery is Redis-only; `POD_IP` comes from the
  Downward API via env, explicitly to need "zero RBAC" (`mesh_boot.rs:380-381`).
- **No `buzz-db` / Postgres.** The mesh is Redis-only.


## Module: buzz-conformance (`crates/buzz-conformance`)
### Aspect: Integrations

Two integration edges only: **inward** from `buzz-relay` (the sole dependent crate) and
**outward** to a TLA+ spec file that is read by humans, never by code. No network client, no
external service, no message broker.

---

### Dependency graph

`buzz-conformance` is declared in exactly two manifests:

| Manifest | Line | Form |
|---|---|---|
| workspace root | `Cargo.toml:5` | workspace member |
| workspace root | `Cargo.toml:125` | `buzz-conformance = { path = "crates/buzz-conformance" }` |
| `crates/buzz-relay/Cargo.toml` | `:20` | `buzz-conformance = { workspace = true }` |

**`buzz-admin` does not declare this dependency.** Grep for `conformance` in
`crates/buzz-admin/Cargo.toml` returns nothing, and grep for `buzz_conformance` /
`conformance` across `crates/buzz-admin/src/` returns nothing. Repo-wide, the only
`Cargo.toml` files mentioning the crate are the workspace root and `buzz-relay`.

Outbound, the crate depends on `serde`, `serde_json`, `thiserror`, `uuid`
(`crates/buzz-conformance/Cargo.toml:26-29`) and `proptest` as a dev-dep (`:34`). The
"independence rule" comment (`:7-24`) enumerates the crates it must never depend on
(`buzz-db`, `buzz-relay`, `buzz-pubsub`, `buzz-auth`, `buzz-search`, `buzz-audit`) and the
manifest honors it — including the deliberate refusal to reuse `buzz_core::CommunityId`, so
that type's "no Serde, no `From<Uuid>`" fence stays intact (`:9-14`, restated
`src/lib.rs:47-63`).

---

### Relay call-in path

**1. Tracer binding.** `AppState` holds `pub tracer: Arc<dyn buzz_conformance::Tracer>`
(`crates/buzz-relay/src/state.rs:620`, doc `:615-619`). The constructor binds
`Arc::new(crate::conformance::NoopTracer)` (`crates/buzz-relay/src/state.rs:798`, comment
`:794-797`). Nothing else ever writes the field — grep for `tracer:` and `.tracer =` across
`crates/buzz-relay/src/` finds only this one assignment plus reads at
`handlers/ingest.rs:1383`, `handlers/req.rs:145`, `:356`, `:672`. The constructor comment
promises "Conformance tests overwrite this with a JsonlTracer … (see test helpers in
`crates/buzz-test-client` once those land)" (`:795-797`); those helpers do not exist —
`crates/buzz-test-client/tests/conformance_multitenant.rs` never references
`buzz_conformance`.

**2. Two tracer impls, relay-side.** `crates/buzz-relay/src/conformance/tracers.rs` declares
`NoopTracer` (`:16-20`, empty `record`) and `JsonlTracer` (`:30-45`) which serializes one JSON
object per line into a truncating-open file (`:37-43`) behind a `Mutex<BufWriter<File>>`
(`:31`). `JsonlTracer::record` (`:55-72`) recovers from lock poisoning by
`e.into_inner()` (`:60-63`) and swallows serialization/IO failures (`:68-70`). `JsonlTracer`
is never constructed anywhere in the repo.

The relay's `NoopTracer` shadows the identically-named one in the crate
(`crates/buzz-conformance/src/lib.rs:323-327`) and is what `mod.rs:46` re-exports, so the
crate's own no-op impl is dead.

**3. Ingest seam.** `handlers/ingest.rs:47-50` imports the helper set
(`self as conf, channel_label, claimed_community_from_event, emit, msg_id_label,
state_for_request, EmitGuard, TraceAction, Verdict`). `ingest_event`:

| Step | Line |
|---|---|
| build `AbstractState` | `:1407` |
| arm guard, receive counting tracer | `:1408-1412` |
| delegate to `ingest_event_inner(state, &tracer, …)` | `:1414` |
| map terminal `IngestError` → `SanitizedError` | `:1436-1443` |
| guard drops (implicit) | comment `:1445-1449` |

`ingest_event_inner` takes `tracer: &Arc<dyn buzz_conformance::Tracer>` (`:1455`) and emits at
`:1573` (via `emit_product_feedback_success`, `:133-147`), `:1820-1828`, `:2215-2222`,
`:2374`, `:2511`.

**4. REQ seam.** `handlers/req.rs:116-118` builds `trace_state: Option<AbstractState>` from
`PublicKey::from_slice(&pubkey_bytes).ok().map(…)`. It is `Option` **only** because the pubkey
bytes may be malformed (comment `:112-115`) — it has no relationship to which tracer is bound,
so `trace_state` is `Some` on essentially every authenticated REQ even under `NoopTracer`.

Plumbed by value into the search lane: `handle_search_req(..., trace_state.as_ref())`
(`:230`), parameter `trace_state: Option<&crate::conformance::AbstractState>` (`:514`).

Three gated blocks:

| Block | Gate | Extra DB query | Emit |
|---|---|---|---|
| membership confirmation | `:143` | none (reuses the `db.is_member` result from `:137-141`) | `record_req_authcheck` `:144-150` |
| non-search read | `:338` | `db.communities_of_channels` `:349` | `record_read_message_rows` `:359-365` |
| search read | `:649` | `db.communities_of_channels` `:661` | `record_read_by_id_rows` `:671-677` |

The two `communities_of_channels` calls are inside the `trace_state.is_some()` blocks, not
inside a tracer-type check — so both run and are awaited even though the resulting
`TraceStep` is immediately discarded by `NoopTracer::record`. On DB error the code substitutes
an empty `HashMap` (`:356`, `:668`), which turns every channel-scoped row into a
missing-lookup and yields a single `ImplBug` step for the whole page.

**5. Guard arm/disarm.** There is no disarm API. `EmitGuard::arm`
(`conformance/mod.rs:383-400`) hands back a `CountingTracer` (`:356-373`) and the caller
substitutes it for the duration; `Drop` (`:403-415`) decides based on the counter. The ingest
site names the seam `"ingest_event_exited_without_trace"` (`ingest.rs:1411`) and the
guard-drop test asserts that string flows through
(`conformance/mod.rs:516-521`). One guard exists in the whole relay — the REQ path arms none.

**6. Error-alphabet coupling.** `sanitized_reason_for` (`conformance/mod.rs:422-430`) is the
only place `buzz-relay`'s error type touches the schema, and it is an exhaustive match over
`crate::handlers::ingest::IngestError` — a new variant is a compile error, which is the
mechanism `crates/buzz-conformance/TRACE_SCHEMA.md:120-124` calls "closed".

---

### TLA+ spec relationship

The relationship is documentary: no build step, test, or CI job reads
`docs/spec/MultiTenantRelay.tla`. The coupling is doc comments carrying line numbers.

| Rust site | Cites | Actual spec line | Match |
|---|---|---|---|
| `src/lib.rs:186` | `WriteInsert` 514–550 | `:514` | yes |
| `src/lib.rs:187` | `WriteInsertGlobal` 559–595 | `:559` | yes |
| `src/lib.rs:188` | `WriteDuplicate` 606–637 | `:606` | yes |
| `src/lib.rs:189` | `SanitizedError` 778 | `:778` | yes |
| `src/lib.rs:190` | `AuthCheck` 794 | `:794` | yes |
| `src/lib.rs:191` | `ReadMessageRows` 643 | `:643` | yes |
| `src/lib.rs:192` | `ReadByIdRows` 681 | `:681` | yes |
| `src/lib.rs:193` | `ReadHostFeedRows` "line ~720" | `:703` | off by 17 |
| `src/transitions.rs:53` / `:296` | `Inv_NonInterference` "line ~983" | `:985` | ~approximate |
| `src/transitions.rs:54` | `Inv_ReadConfinement` "line ~1003" | `:999` | ~approximate |
| `src/lib.rs:115` | `AuthCheck` verdict alphabet, "spec line 794" | `:800` for `verdict ==` | close |

The relay repeats the citations: `ingest.rs:1803` ("Spec AuthCheck (line 794)"),
`:2355-2357` ("WriteInsert (line 514) / WriteDuplicate (line 606)"), `:2484-2492`
("WriteInsert (line 514) / WriteInsertGlobal (line 559) / WriteDuplicate (line 606) …
lines 559-595"), `conformance/mod.rs:422-423` ("spec line 778"). All resolve correctly.

`TRACE_SCHEMA.md` drifts from both: it cites `WriteInsertGlobal` at "line 562"
(`:57`) and `WriteDuplicate` at "line 612" (`:69`) — actual `:559` and `:606`.

**Spec surface not integrated.** `Next` has 23 disjuncts
(`docs/spec/MultiTenantRelay.tla:933-956`); the trace vocabulary covers 8. `Safety` conjoins
13 invariants (`:1129-1142`); the Rust checker enforces `Inv_NonInterference` for reads and a
fragment of `AuthCheck`. `docs/spec/MultiTenantRelay.cfg:26` declares a 9-element
`SanitizedErrors` set against the Rust enum's 3. `docs/spec/GitOnObjectStore.tla` is a
separate spec consumed by `crates/buzz-relay/src/api/git/cas_publish.rs` — unrelated to this
crate.

---

### Build / CI integration

| Surface | Where | Runs the crate? |
|---|---|---|
| `just test-unit` | `justfile:275-296`, conformance step at `:286-290` (`cargo nextest run -p buzz-conformance`) | **Yes** — all targets, lib + both integration tests |
| `just ci` | `justfile:266` → `test-unit` | Yes, transitively |
| `scripts/run-tests.sh unit` | `:78-103`, conformance at `:96-99` (`cargo test -p buzz-conformance`) | Yes (nextest-absent fallback) |
| relay-side emitter tests | `crates/buzz-relay/src/conformance/mod.rs:431-726`, `handlers/ingest.rs:2530-2565` | **No** — `buzz-relay` appears in neither unit list |
| GitHub Actions | grep `conformance` in `.github/workflows/` | No hits |

Contrast with the `buzz-relay-mesh` crate (`Cargo.toml:27`), which appears in no
`test-unit`/`run-tests.sh` step either — the same omission pattern, so `buzz-conformance`
being present in the unit gate is the exception rather than the rule for non-core crates.

**Documentation integration is absent.** Grep for `buzz-conformance`, `MultiTenantRelay.tla`,
`conformance`, `TLA`, and `formal` across `AGENTS.md`, `ARCHITECTURE.md`, and
`CONTRIBUTING.md` returns nothing. The crate is missing from `AGENTS.md`'s repo-structure
table (which lists `buzz-audit` at `AGENTS.md:46` and its neighbours but no
`buzz-conformance`), and `ARCHITECTURE.md` has no mention of the formal-methods lane at all.
The crate's own `LIMITS.md` (125 lines) and `TRACE_SCHEMA.md` (163 lines) are the only prose,
and neither is linked from any top-level doc.

