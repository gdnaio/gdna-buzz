## Module: buzz-pair-relay & buzz-pairing-cli (`crates/buzz-pair-relay`, `crates/buzz-pairing-cli`)
### Aspect: Data Model

#### Two distinct data models share this scope

This module group spans two data models that share a wire shape but no code:

1. **The sidecar relay's data model** (`crates/buzz-pair-relay/src/lib.rs`) — a generic, protocol-agnostic pub/sub router. It has no concept of "pairing state": it only tracks WebSocket subscriptions keyed by a 32-byte `#p` value and a global event-ID dedup set.
2. **The actual NIP-AB pairing state machine** (`crates/buzz-core/src/pairing/{session,crypto,qr,types}.rs`) — not owned by this scope, but consumed directly by `buzz-pairing-cli`, which links against it (`crates/buzz-pairing-cli/Cargo.toml:15`, `buzz-core = { workspace = true }`). This is where session states, cryptographic derivations, and message shapes actually live.

`buzz-pair-relay` never parses NIP-44 plaintext, so it cannot know whether an EVENT is an `offer`, `sas-confirm`, `payload`, `complete`, or `abort` — its own module doc says as much: "No persistence. No auth. No history." (`crates/buzz-pair-relay/src/lib.rs:1-4`).

#### Pairing session state machine (`buzz-core::pairing::session::SessionState`)

Defined at `crates/buzz-core/src/pairing/session.rs:59-76`:

| State | Meaning | Entered by |
|---|---|---|
| `Waiting` | Session created; QR displayed (source) or briefly held before the offer is sent (target) | `PairingSession::new_source` (`session.rs:112`), `new_target` (`session.rs:286`, transitions out again by the end of the same call) |
| `Confirming` | SAS code displayed, awaiting user confirmation (source) or awaiting `sas-confirm` from source (target) | `handle_offer` sets it at `session.rs:194`; `new_target` also lands here by `session.rs:317` |
| `AwaitingConfirmation` | Target verified the transcript hash but must wait for explicit user approval before touching any payload | `handle_sas_confirm`, set at `session.rs:365` |
| `Transferring` | SAS confirmed on both sides; payload can be sent/received | `confirm_sas` (source, set at `session.rs:223`); `confirm_target_sas` (target, set at `session.rs:379`) |
| `PayloadExchanged` | Payload sent (source) or received (target); awaiting `complete` | `send_payload`, set at `session.rs:250`; `handle_payload`, set at `session.rs:397` |
| `Completed` | Protocol finished successfully | `handle_complete` (source, set at `session.rs:258`); `send_complete` (target, set at `session.rs:416`) |
| `Aborted` | Either side aborted | `abort` (`session.rs:430-445`), `handle_abort` (`session.rs:448-465`), or `handle_complete` on `success:false` (`session.rs:265`, without recording the event — see Business Rules) |

Every public state-changing method is guarded by `expect_state`/`expect_role` (`session.rs:706-724`, e.g. invoked at `session.rs:150-152`). `Completed` and `Aborted` are terminal: `abort()` explicitly refuses calls from either terminal state (`session.rs:431-435`), and `handle_abort()` does the same (`session.rs:449-453`) — verified by the tests `local_abort_after_completed_is_rejected` (`session.rs:871-897`) and `reject_abort_after_completed` (`session.rs:933-983`).

**Two roles** (`session.rs:48-55`, `pub enum Role { Source, Target }`): `Source` holds the secret and initiates; `Target` scans the QR and receives. A `PairingSession` is pinned to one role for its lifetime — there is no dual-role or observer mode.

**Timeout**: `DEFAULT_TIMEOUT = Duration::from_secs(120)` (`session.rs:43`), checked at the top of nearly every state-changing method via `check_expired()` (`session.rs:698-703`, e.g. called at `session.rs:150`), which compares `Instant::now() - created_at` against `timeout` (`is_expired`, `session.rs:476-478`). There is no background timer or task that expires a session proactively — expiry is only detected the *next* time a method is called on it (test: `expired_session_rejects_operations`, `session.rs:1129-1145`). A session abandoned mid-flow simply sits inert in memory until its owning process drops it or calls into it again.

#### Cryptographic material generated/exchanged

All derivation functions live in `crypto.rs` and are pure (no I/O):

| Value | Derivation | Source |
|---|---|---|
| `session_secret` | 32 random bytes via `rand::fill` | `session.rs:114` (source only; target receives it from the QR payload) |
| Ephemeral keypair | `nostr::Keys::generate()` — fresh secp256k1 keypair per session, per role, discarded after | `session.rs:113` (source), `session.rs:287` (target) |
| `session_id` | `HKDF-SHA256(IKM=session_secret, salt="", info="nostr-pair-session-id", L=32)` | `crypto.rs:54-56`, `derive_session_id`; constant `INFO_SESSION_ID` at `crypto.rs:22` |
| `ecdh_shared` | `nostr::util::generate_shared_key(own_privkey, peer_pubkey)` — raw, unhashed 32-byte x-coordinate | called at `session.rs:184` (source) and `session.rs:290` (target) |
| `sas_input`, `sas_code` | `HKDF-SHA256(IKM=ecdh_shared, salt=session_secret, info="nostr-pair-sas-v1", L=32)`; `sas_code = be_u32(sas_input[0..4]) % 1_000_000` | `crypto.rs:70-75`, `derive_sas`; constant `INFO_SAS` at `crypto.rs:23` |
| `transcript_hash` | `HKDF-SHA256(IKM = session_id‖source_pk‖target_pk‖sas_input (128 bytes), salt=session_secret, info="nostr-pair-transcript-v1", L=32)` | `crypto.rs:89-105`, `derive_transcript_hash`; constant `INFO_TRANSCRIPT` at `crypto.rs:24` |
| NIP-44 conversation key | Derived internally by `nostr::nips::nip44::encrypt`/`decrypt` from `(own_ephemeral_privkey, peer_ephemeral_pubkey)` | encrypt call at `session.rs:568-573` (`build_event`); decrypt call at `session.rs:601-605` (`decrypt_message`) |

All of these derivations are exercised end-to-end by `buzz-pairing-cli test-vectors` (`crates/buzz-pairing-cli/src/main.rs:335-401`), which reproduces the fixed test vectors pinned in `crates/buzz-core/src/pairing/NIP-AB.md`'s "Test Vectors" section and independently in `crypto.rs`'s `all_test_vectors` unit test (`crypto.rs:239-273`). SAS is always exactly 6 zero-padded decimal digits (`format_sas`, `crypto.rs:116-118`).

Secret hygiene: `session_secret`, `session_id`, and `sas_input` are `zeroize`d on `PairingSession::drop` (`session.rs:731-738`); the ECDH shared secret is explicitly zeroized immediately after use in both `handle_offer` (`session.rs:187`) and `new_target` (`session.rs:291`), rather than waiting for `Drop`; decrypted plaintext is zeroized after parsing in `decrypt_message` (`session.rs:614-617`); the CLI's `resolve_payload` wraps the resolved nsec/secret in `Zeroizing<String>` (`crates/buzz-pairing-cli/src/main.rs:581-599`).

#### QR-code payload shape (`buzz-core::pairing::qr::QrPayload`)

Struct at `crates/buzz-core/src/pairing/qr.rs:34-48`:

| Field | Type | Notes |
|---|---|---|
| `source_pubkey` | `nostr::PublicKey` | Source's ephemeral pubkey |
| `session_secret` | `[u8; 32]` | Zeroized on `Drop` (`qr.rs:52-56`) |
| `relays` | `Vec<String>` | One or more `ws://`/`wss://` URLs |
| `version` | `u32` | Always `1` on encode; on decode, defaults to `1` if the `v=` parameter is absent, and any other value is rejected (`qr.rs:181-186`) |

The wire format is a `nostrpair://` URI string, not a separate binary QR payload — the "QR code" is a rendering of this URI, not an independent structure:
```
nostrpair://<source_pubkey_hex>?secret=<session_secret_hex>&relay=<url-encoded-relay>[&relay=...]&v=1
```
(module doc, `qr.rs:12-20`; encoder `encode_qr`, `qr.rs:79-93`; decoder `decode_qr`, `qr.rs:104-238`). Validation on decode: the pubkey and secret must each be exactly 64 lowercase-hex characters (`qr.rs:129-135`, `qr.rs:196-202`); an all-zeros secret is explicitly rejected (`qr.rs:210-214`); at least one `relay=` parameter must be present (`qr.rs:217-221`) and each must parse as a full URL with a `ws`/`wss` scheme and a host (`qr.rs:224-238`); the total URI must not exceed 2048 characters (`qr.rs:105-110`).

#### NIP-44-encrypted message shapes (`buzz-core::pairing::types::PairingMessage`)

Defined at `crates/buzz-core/src/pairing/types.rs:21-58`, tagged by a `"type"` discriminant in kebab-case (`#[serde(tag = "type", rename_all = "kebab-case")]`, `types.rs:20`):

| Variant | Fields | JSON `type` |
|---|---|---|
| `Offer` | `session_id: String` (hex), `version: u32` (defaults to `1` via `#[serde(default = "default_version")]`, `types.rs:24-26`, helper at `types.rs:9-11`) | `"offer"` |
| `SasConfirm` | `transcript_hash: String` (hex) | `"sas-confirm"` |
| `Payload` | `payload_type: PayloadType`, `payload: String` | `"payload"` |
| `Complete` | `success: bool` | `"complete"` |
| `Abort` | `reason: AbortReason` | `"abort"` |

`PayloadType` (`types.rs:63-71`, snake_case on the wire): `Nsec`, `Bunker`, `Connect`, `Custom`. `AbortReason` (`types.rs:81-96`): `SasMismatch`, `UserDenied`, `Timeout`, `ProtocolError`, plus a catch-all `Unknown` produced only by deserializing an unrecognized string (`#[serde(other)]`, `types.rs:94-95`) — its own doc comment says callers must not construct `Unknown` for outbound aborts (`types.rs:91-93`).

These `PairingMessage` values are never sent as-is: each is JSON-serialized, then NIP-44-v2-encrypted, then wrapped in a signed Nostr event by `build_event` (`session.rs:561-585`). The plaintext type tag is therefore opaque to any on-the-wire observer, including `buzz-pair-relay` itself.

#### The kind:24134 event envelope — what actually crosses the wire

Both sides of this module group independently agree on this shape: the relay validates it in `validate_event` (`crates/buzz-pair-relay/src/lib.rs:418-497`), and `buzz-core::pairing::session` builds it in `build_event` (`session.rs:561-585`):

```jsonc
{
  "id": "<64-char lowercase hex>",
  "pubkey": "<64-char lowercase hex, ephemeral>",
  "kind": 24134,
  "created_at": <unix ts, jittered 0-30s into the past — session.rs:578-580>,
  "tags": [["p", "<64-char lowercase hex, recipient ephemeral pubkey>"]],
  "content": "<base64 NIP-44 v2 ciphertext>",
  "sig": "<128-char lowercase hex Schnorr signature>"
}
```

The relay enforces this is *exactly* these 7 keys, no more (`validate_event`'s `ALLOWED_KEYS`, `lib.rs:424-436`), *exactly* one tag shaped `["p", <64-hex>]` (`lib.rs:480-497`), and a `content` that passes a hand-rolled structural NIP-44-v2 sanity check — base64 alphabet, decoded length ≥ 99 bytes, first decoded byte `0x02` (`validate_nip44_content`, `lib.rs:343-411`).

#### In-memory session/subscription storage in `buzz-pair-relay` — "ephemeral" is literal

The relay's storage is entirely process-local, in-memory, and never persisted:

```rust
pub struct Relay {
    subs: Mutex<Vec<Sub>>,                                              // lib.rs:104, 108
    conn_count: AtomicU32,                                              // lib.rs:105
    next_conn_id: AtomicU64,                                            // lib.rs:106
    seen_ids: Mutex<Vec<([u8; 32], tokio::time::Instant)>>,             // lib.rs:107, dedup
    delivered: Mutex<HashMap<[u8; 32], (u32, tokio::time::Instant)>>,   // lib.rs:109, per-#p budget
}
```
(struct at `lib.rs:104-110`; `Sub { conn_id, sub_id, p_value, writer_tx }` at `lib.rs:97-102`).

`subs` is a flat `Vec<Sub>`, not a `HashMap` — lookups are `O(n)` linear scans (e.g. `lib.rs:163`, `lib.rs:783`), acceptable at the documented cap of 128 connections (`MAX_CONNS`, `lib.rs:61`) but not the hash-indexed structure a reader might assume from "session storage." `seen_ids` (event-ID dedup) is likewise a `Vec`, not a `HashSet`, so duplicate checking is also `O(n)` per event (`reserve_id`, `lib.rs:135-148`), bounded by `DEDUP_CAP = 1024` (`lib.rs:79`). `delivered` (per-`#p` delivery counter) genuinely is a `HashMap<[u8;32], (u32, Instant)>`, bounded by `DELIVERED_MAP_CAP = 4096` (`lib.rs:82`).

Both `seen_ids` and `delivered` entries expire via `ENTRY_TTL = Duration::from_secs(300)` (`lib.rs:86`), evicted lazily on the next access (`retain` calls at `lib.rs:137` and `lib.rs:176`), not by a background sweep task. There is no database, file, or Redis dependency anywhere in the crate's manifest (`crates/buzz-pair-relay/Cargo.toml:18-30` lists only `tokio`, `tokio-tungstenite`, `serde_json`, `futures-util`, `tokio-util`, `parking_lot`, `hyper`, `hyper-util`, `http-body-util`, `secp256k1`, `sha2` — no `sqlx`/`redis`/storage crate of any kind). The module doc's claim "No persistence" (`lib.rs:5`) holds up structurally: killing the process discards every subscription, dedup entry, and delivery counter instantly.

#### Uncertainty

Whether `nostr::util::generate_shared_key` truly returns the raw, unhashed x-coordinate — as `NIP-AB.md`'s "Implementation warning" under §Cryptographic Primitives demands, since "many secp256k1 libraries... hash the ECDH output with SHA-256 by default" — cannot be verified from this crate's code alone; that guarantee lives inside the external `nostr` crate (`nostr = "0.44"`, root `Cargo.toml:61`), which is out of scope for this module group. The pinned test vectors in `crypto.rs:239-273` matching the spec's own published vectors is the strongest evidence available from within this scope that the derivation is wired correctly, but it does not constitute independent verification of the `nostr` crate's internal ECDH implementation.
