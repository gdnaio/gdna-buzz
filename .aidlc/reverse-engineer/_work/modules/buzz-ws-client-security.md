## Module: buzz-ws-client (`crates/buzz-ws-client`)

### Aspect: Security

Findings are limited to what these five files contain. Where behaviour is delegated
to `nostr` or `tokio-tungstenite`, that is stated explicitly rather than assumed.

---

### 1. Key handling (AUTH event signing)

| Observation | Evidence |
|---|---|
| Keys are always **borrowed**, never owned or stored: `&Keys` on `connect_authenticated`, `authenticate`, `publish_event`, `build_auth_event` | `connection.rs:39`, `:72`, `:280`; `message.rs:154` |
| `NostrWsConnection` holds **no key material** — its four fields are the stream, a message buffer, the last challenge, and the URL | `connection.rs:26`–`31` |
| No key is cloned, copied into a `String`, or written to a struct anywhere in the crate (verified across all 541 lines) | — |
| Signing is delegated to `nostr`: `EventBuilder::auth(...).sign_with_keys(keys)`; no manual Schnorr/secp code, no nonce generation here | `message.rs:158`, `:165` |
| Key material is never logged. The three `debug!` sites log the relay URL, a success message, and outbound frame text | `connection.rs:57`, `:91`, `:123` |
| **The outbound-frame log includes the full signed AUTH event and any `auth_tag` token**: `debug!("→ relay: {text}")` runs for every `send_raw`, including `["AUTH", <event>]` and any bearer-style tag value the caller passed | `connection.rs:82` → `:121`–`:123`; tag injected at `message.rs:159`–`163` |
| The signed AUTH event contains a public key and signature only (no secret), so the logging exposure is the authorization tag / challenge, not the private key | `message.rs:151`–`167` |

---

### 2. TLS verification

| Question | Answer from code | Evidence |
|---|---|---|
| Is certificate validation ever disabled? | **No.** There is no `Connector`, no `connect_async_tls_with_config`, no `danger_*`/`accept_invalid_*` option, and no custom verifier in the crate. A search for `connect_async_tls_with_config`, `Connector`, `accept_invalid`, `rustls`, `native_tls` inside `crates/buzz-ws-client/` returns zero matches. | `connection.rs:53` is the only dial site |
| What TLS stack is used? | Whatever `tokio-tungstenite`'s default `connect_async` selects for the enabled feature set. The workspace enables `rustls-tls-webpki-roots` | root `Cargo.toml:113`; inherited at `crates/buzz-ws-client/Cargo.toml:12` |
| Which trust anchors? | Bundled webpki roots (per that feature name), i.e. **not** the OS trust store — a relay signed by a private/corporate CA in the system store would not validate unless the CA is in webpki-roots. This follows from the feature selection, not from code in this crate. | root `Cargo.toml:113` |
| Certificate pinning / hostname override? | None. | — |
| Is plaintext `ws://` allowed? | **Yes.** `connect` performs only `url::Url` parsing with no scheme check, so `ws://` (and even `http://`, subject to `tokio-tungstenite`'s own handling) is accepted; TLS is opportunistic via `MaybeTlsStream` | `connection.rs:48`–`55`, `:14` |
| Any warning emitted for plaintext? | No — the debug log is scheme-agnostic | `connection.rs:57` |

Net: TLS is used correctly with defaults when the caller supplies `wss://`, and is
silently absent when the caller supplies `ws://`. Scheme enforcement is a caller
responsibility; nothing in this crate constrains it.

---

### 3. Challenge / replay handling

| Control | Present? | Evidence |
|---|---|---|
| Challenge length bound (1024 bytes) | Partially — enforced only on the socket-read path | enforced `connection.rs:198`–`202`; **not** enforced when the challenge comes from `pending_challenge` (`:161`–`163`) or from the buffered-message path (`:170`–`171`) |
| Challenge content validation (charset, hex/base64 shape, minimum entropy) | No | `connection.rs:197`–`203`; `message.rs:132`–`139` |
| Replay protection — rejecting a previously seen challenge | No; `pending_challenge` is a single slot that is overwritten by each new `AUTH`, with no history | `connection.rs:144`, `:256` |
| Binding of challenge to relay identity | Indirect: the AUTH event embeds the relay URL the client dialed, via `RelayUrl::parse(self.relay_url)` | `connection.rs:79` → `message.rs:157` |
| Relay URL used for the AUTH tag is the caller's raw string, not the normalized URL actually dialed | Yes — divergence is possible (e.g. trailing-slash/default-port normalization) | store `connection.rs:63`; dial `connection.rs:53` (normalized); AUTH tag `connection.rs:79` (raw) |
| Timestamp tolerance / clock-skew window | **None in this crate.** `created_at` is set inside `nostr::EventBuilder::auth`; there is no `Timestamp`, `now()`, or tolerance constant anywhere in these files. Any freshness window is enforced by the relay, not the client | `message.rs:158` |
| Are relay-provided event signatures verified on inbound `EVENT`? | Not by this crate. `serde_json::from_value::<Event>` deserializes; whether the `nostr` crate verifies the signature during deserialization is **not determinable from these files** | `message.rs:70`–`74` |
| Mid-session challenge auto-response | No — recorded and buffered only, so a relay-initiated re-auth is not answered unless the caller acts | `connection.rs:255`–`258` |

---

### 4. Input validation gaps (client-side, on relay-controlled data)

| Gap | Consequence as coded | Evidence |
|---|---|---|
| No cap on inbound frame size or on parsed field lengths (other than the 1024-byte challenge on one path) | Relay-controlled strings (`NOTICE`, `CLOSED`, `OK` reason) are copied into owned `String`s unbounded; no `WebSocketConfig` limits are set on the socket | `message.rs:87`–`91`, `:114`–`118`, `:125`–`129`; `connection.rs:53` |
| Unbounded `buffer` growth | A relay that streams non-matching messages while the client waits for an `OK` grows `VecDeque` until the deadline expires; there is no capacity limit or drop policy | `connection.rs:28`, `:205`, `:257`, `:259` |
| `OK.accepted` defaults to `false` on malformed input, `message` defaults to `""` | Fails closed for acceptance (safe direction), but a malformed `OK` is indistinguishable from an explicit rejection with no reason | `message.rs:86`, `:87`–`91` |
| `event_id` correlation is a raw string comparison of relay-supplied text against locally computed hex | Case/format mismatch would silently fail to correlate (leading to `Timeout` rather than a mismatch error); no length/hex validation of `arr[1]` | `message.rs:81`–`85`; compare `connection.rs:227`, `:254` |
| `send_raw` accepts any `serde_json::Value` and sends it verbatim | Callers can emit arbitrary frames, including a hand-rolled `AUTH`; the crate applies no schema or scheme checks on that path | `connection.rs:121`–`125` |
| No authenticated-state check before `send_event` | An event can be published on an unauthenticated connection; rejection handling is left to the relay's `OK` | `connection.rs:96`–`101` |
| Non-text frames ignored silently | A relay sending binary frames produces no error or log | `connection.rs:152`, `:212`, `:266` |
| Ping payload echoed back verbatim as Pong | Standard WebSocket behaviour, but there is no size limit applied by this crate | `connection.rs:149`, `:209`, `:263` |

---

### 5. Memory / unsafe

- `#![deny(unsafe_code)]` at the crate root (`lib.rs:1`).
- **Zero `unsafe` blocks** — verified by search across `crates/buzz-ws-client/`; the
  only match for the token `unsafe` is the deny attribute itself.
- No FFI, no raw pointers, no transmutes.
- Two panics are reachable only via internal invariants: `VecDeque::remove(...).unwrap()`
  after a successful `position` lookup (`connection.rs:170`, `:229`) and the paired
  `unreachable!()` arms (`connection.rs:172`, `:231`). Both are guarded by the
  preceding `position(...)` predicate on the same borrow, so relay input alone cannot
  trigger them; they remain panics-in-library-code rather than errors.

---

### 6. Secrets in error messages and logs

| Path | What is exposed | Evidence |
|---|---|---|
| `debug!("→ relay: {text}")` | Full outbound JSON, including the signed AUTH event and any `auth_tag` token value | `connection.rs:123` |
| `debug!("connected to relay at {url}")` | Relay URL as provided; if a caller embeds credentials in the URL userinfo, they would appear here (the crate does not strip userinfo) | `connection.rs:57` |
| `WsClientError::UnexpectedMessage(text.to_string())` | The entire raw relay frame is embedded in the error string and thus in any caller log | `message.rs:61`, `:68`, `:73`, `:84`, `:102`, `:112`, `:136` |
| `WsClientError::AuthFailed(ok.message)` | Relay-supplied rejection reason, propagated verbatim | `connection.rs:88` |

No redaction helper or `Display`-masking wrapper exists in the crate.
