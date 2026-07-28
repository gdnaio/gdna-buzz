## Module: buzz-pair-relay & buzz-pairing-cli (`crates/buzz-pair-relay`, `crates/buzz-pairing-cli`)
### Aspect: API Surface

#### buzz-pair-relay: HTTP/WebSocket surface

The sidecar exposes exactly one network surface: a single HTTP listener that either upgrades to WebSocket or returns `400 Bad Request`. There is no path routing inside the binary at all — `http_service` (`crates/buzz-pair-relay/src/lib.rs:913-995`) inspects only the method and headers, never `req.uri().path()`. Any request path upgrades if the WebSocket handshake headers are present; the module doc explicitly delegates path restriction (e.g. limiting traffic to `/pair`) to a reverse proxy (`lib.rs:6-16`):

| Surface | Detail | Citation |
|---|---|---|
| Upgrade acceptance criteria | `GET`, HTTP/1.1+, `Connection: upgrade`, `Upgrade: websocket`, `Sec-WebSocket-Version: 13`, and a 24-char ASCII `Sec-WebSocket-Key` | `lib.rs:918-943` |
| Non-WS request | `400 Bad Request`, empty body | `lib.rs:946-949` |
| Over connection cap | `503 Service Unavailable` if `conn_count >= MAX_CONNS` (128) | `lib.rs:950-955` |
| Successful upgrade | `101 Switching Protocols` with `Connection: Upgrade`, `Upgrade: websocket`, `Sec-WebSocket-Accept` via `derive_accept_key` | `lib.rs:960-995` |
| Bind | `run_server(listener, relay)` takes a caller-supplied `TcpListener`; no TLS built in | `lib.rs:999-1017` |

Once upgraded, the WebSocket speaks a **subset of the NIP-01 relay wire protocol**, not a bespoke pairing protocol. Three inbound verbs are recognized inside the `Message::Text` branch (`lib.rs:685-892`):

| Verb | Shape | Behavior | Citation |
|---|---|---|---|
| `REQ` | `["REQ", "<sub_id>", {"#p": ["<64-hex>"], "kinds"?: [24134]}]` | Registers exactly one subscription for this connection, keyed by the `#p` value. Replies `EOSE` on success, `CLOSED` on any validation failure. Only one live subscriber per `#p` value is allowed relay-wide. | `lib.rs:691-768`; filter validation `validate_filter`, `lib.rs:312-334` |
| `EVENT` | `["EVENT", <kind:24134 event object>]` | Validates shape, freshness, tag structure, NIP-44 envelope shape, and Schnorr signature, then dedups and forwards to the single live subscriber for the event's `p` tag. Always replies `OK`. | `lib.rs:771-864` |
| `CLOSE` | `["CLOSE", "<sub_id>"]` | Removes the subscription if `sub_id` matches the connection's active one; unknown sub_ids are silently ignored per NIP-01. Does not close the WebSocket itself. | `lib.rs:867-884` |
| anything else | any other first array element (e.g. `AUTH`, `COUNT`) | `NOTICE "error: unsupported message"` | `lib.rs:887-890` |

Outbound message shapes are hand-built helpers, all NIP-01-shaped JSON arrays (`lib.rs:259-291`): `make_ok(id, ok, msg)` → `["OK", id, ok, msg]`; `make_closed(sub_id, msg)` → `["CLOSED", sub_id, msg]`; `make_eose(sub_id)` → `["EOSE", sub_id]`; `make_notice(msg)` → `["NOTICE", msg]`. Delivered events are wrapped as `["EVENT", sub_id, <event>]` inside `deliver_single` (`lib.rs:186-190`). Ping frames get a Pong echoing the same payload (`lib.rs:656-660`); Binary/Frame messages close the connection immediately (`lib.rs:654`); a client Close frame triggers a Close reply then teardown (`lib.rs:663-666`).

**No NIP-11 endpoint, no `/health`, no metrics endpoint** — grepping the crate for any of these turns up nothing; the only non-WebSocket HTTP responses this binary can produce are `400` and `503`. This is a materially narrower surface than the main `buzz-relay`, which serves NIP-11 and dedicated health/readiness probes over plain HTTP. The pairing-relay Helm chart compensates by using a raw `tcpSocket` readiness/liveness probe (`deploy/charts/buzz/templates/pairing-relay.yaml:42-47`) instead of an HTTP path probe, since there is no HTTP health path to hit.

#### buzz-pair-relay: public Rust API (`lib.rs`)

The crate builds as both a library (`buzz_pair_relay`) and a binary (`buzz-pair-relay`), per `crates/buzz-pair-relay/Cargo.toml:9-16`. Exports:

| Item | Signature | Consumers |
|---|---|---|
| `Relay` (struct) | `pub struct Relay { .. }` (`lib.rs:104`) — every field private | `main.rs:24`, `tests/integration.rs:14` |
| `Relay::new()` | `pub fn new() -> Self` (`lib.rs:121`) | `main.rs:24`, `tests/integration.rs:36` |
| `impl Default for Relay` | delegates to `new()` (`lib.rs:114-118`) | not called anywhere in this scope (see Debt) |
| `run_server` | `pub async fn run_server(listener: TcpListener, relay: Arc<Relay>)` (`lib.rs:999`) — accept loop, one task per connection | `main.rs:25`, `tests/integration.rs:38` (bound to `:0` for randomized test ports) |
| `CONN_TIMEOUT` | `pub(crate) const CONN_TIMEOUT: Duration` (`lib.rs:59`) | doc comment claims "for test access", but `pub(crate)` is not visible to `tests/integration.rs`, which is compiled as a separate crate — see Debt |

Nothing else is public. `Sub`, `ConnGuard`, `RateWindow`, `OutMsg`, and every validation/crypto helper (`validate_filter`, `validate_event`, `verify_event_sig`, `validate_nip44_content`, `decode_hex32`, `decode_hex64`, `deliver_single`, `reserve_id`, `unreserve_id`, `remove_sub`) are private module items (`lib.rs:94-102`, `219-234`, `236-257`, `312-334`, `343-411`, `418-497`, `521-555`, `135-217`). No other crate in the workspace depends on `buzz-pair-relay` as a library — confirmed by there being no `buzz_pair_relay` import anywhere outside this crate's own `main.rs` and `tests/`.

#### buzz-pairing-cli: CLI surface (`buzz-pair` binary)

Built with `clap` derive macros, defined at `crates/buzz-pairing-cli/src/main.rs:40-72`:

```
buzz-pair <SUBCOMMAND>
```

| Subcommand | Flags | Default | Citation |
|---|---|---|---|
| `source` | `--relay <URL>`, `--nsec <BECH32>` (optional) | `--relay` = `"wss://relay.damus.io"`; `--nsec` omitted ⇒ generates a throwaway test key | `main.rs:48-56` |
| `target` | `--relay <URL>` (optional override), `--show-secret` (flag) | `--relay` = falls back to the relay URL embedded in the QR; `--show-secret` = `false` | `main.rs:59-67` |
| `test-vectors` | none | — | `main.rs:70` |

Dispatch: `run()` matches `Cmd::Source`/`Cmd::Target`/`Cmd::TestVectors` to `cmd_source`/`cmd_target`/`cmd_test_vectors` (`main.rs:106-111`).

No `--format`, no JSON output mode, no non-interactive flag — `source` and `target` both block on stdin for the SAS y/n prompt (`read_yes_no`, `main.rs:612-616`), and `target` additionally blocks on stdin to read the pasted QR URI (`read_line`, `main.rs:601-609`). This makes the tool inherently interactive, matching its own stated purpose: "designed for interop testing and NIP submission, not production use" (`crates/buzz-pairing-cli/README.md:3`).

Exit behavior: `main()` prints `error: {e}` to stderr and calls `std::process::exit(1)` on any `Err` from `run()` (`main.rs:99-103`) — there is no distinct exit-code taxonomy comparable to `buzz-cli`'s documented 0-5 scheme; every failure here exits `1`.

#### buzz-pairing-cli: what it actually talks to

The CLI does not connect exclusively to `buzz-pair-relay`'s bespoke listener — it connects to whatever relay URL is supplied (`--relay`, or the URL embedded in the scanned QR), speaking the standard Nostr WS wire protocol (`REQ`/`EVENT`/`CLOSE`/`AUTH`/`OK`/`EOSE`) via `tokio_tungstenite::connect_async` (`main.rs:132`, `main.rs:212`). It layers its own NIP-42 AUTH handling on top (`handle_nip42_auth`, `main.rs:416-478`) so it can pair through the production `buzz-relay` as well — confirmed by the README's "Testing Against a Local Buzz Relay" section (`crates/buzz-pairing-cli/README.md:53-99`) and by `buzz-relay`'s own advertisement of a `pairing_relay_url` in its NIP-11 document (`crates/buzz-relay/src/config.rs:430-446`, `crates/buzz-relay/src/nip11.rs:52-54`). `buzz-pairing-cli` is a generic NIP-AB client; `buzz-pair-relay` is only one of the relays it can target, so the two crates are not a tightly-coupled client/server pair despite being filed together in `AGENTS.md`.

#### buzz-core::pairing public API consumed by the CLI

Not owned by this scope, but load-bearing for it — `buzz-pairing-cli` is essentially a thin I/O harness around `buzz_core::pairing`:

| Item | Used for | Call sites |
|---|---|---|
| `PairingSession::new_source` / `new_target` | Session bootstrap | `main.rs:119`, `main.rs:227` |
| `handle_offer`, `confirm_sas`, `send_payload`, `handle_complete` | Source-side flow | `main.rs:151`, `173`, `178`, `186` |
| `handle_sas_confirm`, `confirm_target_sas`, `handle_payload`, `send_complete` | Target-side flow | `main.rs:269`, `300`, `307`, `328` |
| `abort`, `handle_abort` | Both sides | `main.rs:165`, `275`, `294`; `handle_abort` via `check_for_abort`, `main.rs:403-411` |
| `encode_qr` / `decode_qr` | QR URI round-trip | `main.rs:120`, `217` (via `qr::decode_qr` at `main.rs:217`) |
| `derive_sas`, `derive_session_id`, `derive_transcript_hash`, `format_sas` | `test-vectors` subcommand only | `main.rs:356-367` |

No HTTP endpoints are exposed by `buzz-pairing-cli` itself — it is a pure outbound client with no listener of its own.
