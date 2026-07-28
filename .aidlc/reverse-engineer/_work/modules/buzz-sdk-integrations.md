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
