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
| `nostr` | `0.44`, features `nip44`, `nip98` (root `Cargo.toml:61`) | `Event`, `Keys`, `Tag` (`connection.rs:5`); `EventBuilder`, `RelayUrl` (`message.rs:1`); AUTH event construction + signing (`message.rs:158`, `:165`); `event.id.to_hex()` correlation keys (`connection.rs:80`, `:97`); `nostr::event::builder::Error` conversion (`error.rs:47`) | manifest `crates/buzz-ws-client/Cargo.toml:10` |
| `tokio` | `1`, features `rt-multi-thread, macros, net, time, sync, io-util, signal, process` (root `Cargo.toml:43`) | `tokio::time::timeout` (`connection.rs:7`, `:134`, `:187`, `:244`, `:284`); `tokio::time::Instant` deadlines (`connection.rs:176`, `:222`); `tokio::net::TcpStream` in the stream type (`connection.rs:14`) | `Cargo.toml:11` |
| `tokio-tungstenite` | `0.29`, features `rustls-tls-webpki-roots` (root `Cargo.toml:113`) | `connect_async` (`connection.rs:53`), `Message` frames (`connection.rs:124`, `:141`, `:148`, `:151`), `MaybeTlsStream`/`WebSocketStream` (`connection.rs:14`), `close(None)` (`connection.rs:116`), error type in `WsClientError::WebSocket` (`error.rs:8`) | `Cargo.toml:12` |
| `futures-util` | `0.3` (root `Cargo.toml:110`) | `SinkExt` for `send`, `StreamExt` for `next` on the WebSocket (`connection.rs:4`, `:124`, `:134`) | `Cargo.toml:13` |
| `serde_json` | `1` (root `Cargo.toml:69`) | `json!` frame construction (`connection.rs:6`, `:82`, `:98`), `to_string` (`connection.rs:122`), `from_str`/`from_value` parsing (`message.rs:56`, `:70`), `Value` as the `send_raw` parameter type (`connection.rs:121`) | `Cargo.toml:14` |
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
| The **original** string is what ends up in the AUTH event's relay tag, via `RelayUrl::parse(relay_url)` — so the AUTH-event relay value and the dialed URL can differ in normalization | store `connection.rs:63`; use `connection.rs:79` → `message.rs:157` |
| Rejection of non-WebSocket schemes is left to `tokio-tungstenite`, surfacing as `WsClientError::WebSocket` | `connection.rs:53`–`55` (behaviour of the dependency, not verified in this crate) |
| `nostr::RelayUrl::parse` may impose its own scheme rules; that logic lives in the `nostr` crate and is **not verifiable from these files** | `message.rs:157` |

---

### 4. Protocol integrations (wire level)

| Protocol surface | Direction | Evidence |
|---|---|---|
| NIP-01 `EVENT` (client→relay) | out | `connection.rs:98` |
| NIP-01 `OK` | in | `message.rs:80`–`97`; matched at `connection.rs:254` |
| NIP-01 `EVENT` (relay→client) | in | `message.rs:64`–`79` |
| NIP-01 `EOSE`, `CLOSED`, `NOTICE` | in | `message.rs:98`–`131` |
| NIP-42 `AUTH` challenge | in | `message.rs:132`–`139` |
| NIP-42 `AUTH` response (signed event) | out | `connection.rs:82` |
| "NIP-OA" authorization tag carried inside the AUTH event | out | `message.rs:146`–`163` (doc comment names it; example given as `["auth", "<token>"]`) |
| WebSocket Ping/Pong/Close control frames | both | `connection.rs:148`–`151` (and duplicates at `:208`–`:211`, `:262`–`:265`) |
| Anything else (`REQ`, `CLOSE`, `COUNT`) | out, untyped | via `send_raw` (`connection.rs:121`); e.g. `crates/buzz-test-client/src/lib.rs:154`, `:160` |

---

### 5. Consumers and parallel implementations in the repo

| Crate | Relationship | Evidence |
|---|---|---|
| `buzz-cli` | Depends on this crate; calls `publish_event` with a 75 s budget | `crates/buzz-cli/Cargo.toml:77`; `crates/buzz-cli/src/client.rs:1071`, `:1077`, `:1080` |
| `buzz-test-client` | Depends on this crate; wraps `NostrWsConnection` and re-exports its message/error types | `crates/buzz-test-client/Cargo.toml:13`; `crates/buzz-test-client/src/lib.rs:13`–`14`, `:85`, `:98` |
| `buzz-acp` | Does **not** depend on this crate; carries its own `connect_async` + NIP-42 + `RelayMessage` + parser implementation | `crates/buzz-acp/src/relay.rs:3435`–`3461` (auth response), `:3610`–`3616` (AUTH parse), `:3843`–`3845` (handshake), `:2344`–`2350` (mid-session re-auth) |
| Other independent WebSocket clients in the repo (context for duplication, not dependencies) | separate implementations | `crates/buzz-relay/src/router.rs`, `crates/buzz-relay/src/audio/handler.rs`, `crates/buzz-pairing-cli/src/main.rs`, `crates/buzz-pair-relay/tests/integration.rs`, `desktop/src-tauri/src/native_websocket.rs` (all contain `connect_async`) |
