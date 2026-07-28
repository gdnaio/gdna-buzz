## Module: buzz-acp — relay client & observer (`crates/buzz-acp/src`)
### Aspect: Security

#### Private-key handling

`nostr::Keys` is cloned into four long-lived places: `HarnessRelay.keys`
(`relay.rs:547`), the background task's `bg_keys` (`relay.rs:615`, held for the
task's lifetime at `relay.rs:1540`), `RestClient.keys` (`relay.rs:237`, cloned
per `rest_client()` call at `relay.rs:723`), and `engram_fetch`'s
`agent_keys: &Keys` borrow (`engram_fetch.rs:41`). Each clone holds the secret
key material; there is no `Zeroize`, no wrapper type, and no `Debug` redaction —
`RestClient` derives `Debug` (`relay.rs:231`) with `keys` as a plain field, so
any `{:?}` of a `RestClient` depends entirely on `nostr::Keys`'s own `Debug`
impl for secrecy. No `{:?}` of a `RestClient` occurs in this module.

The secret key is never logged. The only place it crosses a process boundary is
`lib.rs`'s MCP env injection (`BUZZ_PRIVATE_KEY` bech32), outside these files.

`engram_fetch.rs:76` derives the NIP-AE conversation key from
`agent_keys.secret_key()` and the owner pubkey; the derived key is used
immediately and dropped, never stored or logged.

#### AUTH frame and token logging

`send_auth_response` logs only a fixed string:
`debug!("sent AUTH response for challenge")` (`relay.rs:3461`). The signed AUTH
event — which carries the NIP-OA `auth_tag` when present
(`relay.rs:3444-3456`) — never reaches a log sink. This is a genuine improvement
over the shared crate, where `send_raw` unconditionally logs the full frame at
debug (`buzz-ws-client/src/connection.rs:123`), leaking the bearer token on every
authenticated connect.

The other two WS write helpers also avoid logging bodies: `send_publish_event_frame`
logs only on error, without the event (`relay.rs:2620`); the three REQ builders
log a description, not the frame (`relay.rs:3204-3211`, `:3255`, `:3292`).

`x-auth-tag` on the HTTP path is set from `auth_tag_json` (`relay.rs:389-391`)
and is never logged. `Authorization: Nostr <base64>` likewise
(`relay.rs:383`).

Counter-example: `relay.rs:2059` logs the entire unparsed relay text frame at
warn on any parse failure —
`warn!("failed to parse relay message: {e} — raw: {text}")`. A malformed
inbound frame containing attacker-chosen content lands verbatim in the log with
no length cap.

#### TLS and scheme enforcement

None. `do_connect` parses the URL with `url::Url` and hands
`parsed.as_str()` straight to `connect_async` (`relay.rs:3830-3837`); there is no
assertion that the scheme is `wss` (or even `ws`). A `BUZZ_RELAY_URL` of
`ws://…` connects in cleartext, carrying the NIP-42 AUTH event and every
`h`-scoped message in plaintext. `relay_ws_to_http` (`relay.rs:3470-3475`)
mirrors this by mapping `ws://` → `http://`, so an unencrypted WS URL also
downgrades every NIP-98-authenticated bridge request to plain HTTP.

The `#[allow(dead_code)]`-free `do_connect_wrong_scheme_is_terminal` test
(`relay.rs:5549-5557`) only asserts that tungstenite's own `Error::Url` is
classified terminal — it does not add a check, and `ws://` is a scheme
tungstenite accepts.

TLS *failures* are classified with care: `is_terminal_rustls_io_error`
(`relay.rs:3724-3766`) downcasts through the `io::Error` source chain to a
`rustls::Error` and treats only five deterministic variants as terminal —
`InvalidCertificate`, `InvalidCertRevocationList`, `NoCertificatesPresented`,
`UnsupportedNameType`, `PeerIncompatible` (`relay.rs:3757-3764`). Ambiguous
protocol/decrypt/alert shapes stay transient and are retried under the bounded
budget. `WsError::Tls(_)` is terminal (`relay.rs:3713`). Nothing here weakens
certificate validation — the rustls default verifier is used
(`rustls::crypto::ring::default_provider()` installed in `lib.rs`).

#### Challenge validation and replay

The NIP-42 challenge is accepted with **no validation at all** on any of three
intake paths:

| Path | Location | Check |
|---|---|---|
| Handshake wait | `relay.rs:3901` — `RelayMessage::Auth { challenge } => return Ok(challenge)` | none |
| Parser | `relay.rs:3610-3616` — extracts `arr[1]` as string | none |
| Mid-session | `relay.rs:2344-2348` — passes straight to `send_auth_response` | none |

No length cap, no character-set check, no rate limit on how often the relay may
re-challenge. `buzz-ws-client` at least caps the handshake challenge at 1024
bytes (`buzz-ws-client/src/connection.rs:178-201`); this copy dropped that.
Consequence: a compromised or hostile relay endpoint can emit an arbitrarily
large challenge string, which is embedded verbatim in a `["challenge", …]` tag
(`relay.rs:3448`) or passed to `EventBuilder::auth` (`relay.rs:3457`), signed
with the agent key, and written back over the socket — an unbounded
sign-and-echo primitive.

Because mid-session AUTH triggers immediate re-authentication
(`relay.rs:2344-2353`) with no back-off or per-connection counter, a relay can
drive an unbounded stream of signed AUTH events from the agent. The shared
crate's record-only behaviour has no such property.

NIP-98 replay is defended: a fresh `nonce` tag (UUIDv4) is added per attempt
(`relay.rs:281-283`) and the event is re-signed on every retry so the ±60 s
window is respected (`relay.rs:311-312`, `:377-381`). A body hash is included
when a body exists (`relay.rs:284-289`).

#### Trust in relay-supplied content

The module treats the relay as authoritative for almost everything it forwards:

| Path | Verification |
|---|---|
| Channel events → `event_tx` | none — `record_event` only hashes the id (`relay.rs:1091-1107`); the `Event` is forwarded verbatim (`relay.rs:2158-2185`) |
| Membership notifications | none beyond an `h`-tag parse (`relay.rs:2078-2085`) — a forged 44100 would trigger a subscribe in `lib.rs:1441-1448` |
| Observer-control frames (kind 24200) | none in this module — `relay.rs:2069-2076` forwards the raw event; signature, owner-pubkey, and ±300 s freshness checks are all in `lib.rs::handle_relay_observer_control_event` |
| Channel discovery metadata (kind 39000) | none — `merge_discovered_channels` (`relay.rs:171-232`) reads `d`/`name`/`archived`/`t`/`hidden`/`private` tags from unverified JSON; the derived `channel_type` feeds the DM author gate in `lib.rs` |
| `CLOSED` reason strings | trusted for control flow — exact-match drop (`relay.rs:3515-3517`) vs. reconnect (`relay.rs:2250-2264`) |
| `NOTICE` rate-limit hint | trusted, but floored at 5 s and only ever able to *extend* the gate (`relay.rs:1157-1164`) |
| Engram candidates (kind 30174) | **verified** — `event.verify()` before decrypt (`engram_fetch.rs:126-128`), then `validate_and_decrypt` binds the (agent, owner) pair (`engram_fetch.rs:131-137`); an undecryptable candidate is a hard `Err`, never treated as absence (`engram_fetch.rs:139-148`) |

The discovery path is the load-bearing one: `channel_type` derived from
unverified relay JSON becomes `ChannelInfo.channel_type`, which
`is_dm_channel` reads to decide whether the DM author gate applies. The
fail-closed default is correct — missing metadata yields `"unknown"`
(`relay.rs:219-221`) and `lib.rs` treats unresolved as DM — but a relay that
*supplies* `["t","stream"]` for a real DM downgrades the gate.

`archived=true` handling is a defense-in-depth control by design
(`relay.rs:163-170`): it stops a reaped channel from being re-offered on
reconnect even when the agent missed the eviction CLOSED.

#### Unbounded buffers and resource limits

Bounded:

| Buffer | Cap | Location |
|---|---|---|
| `event_tx` / `observer_control_tx` | 256 (env-tunable, floored at 1) | `relay.rs:35-42`, `:600-602` |
| `cmd_tx` | 64 | `relay.rs:31`, `:603` |
| `seen_ids` | 6,000–12,000 ids | `relay.rs:45`, `:955-958` |
| `gated_observer_pending` | 256, drop-oldest + counter | `relay.rs:116`, `:1187-1200` |
| `observer_in_flight` | 256, drop-oldest + counter | `relay.rs:1212-1222` |
| Observer replay buffer | 1,000, drop-front | `observer.rs:19`, `:126-129` |
| Observer broadcast channel | 1,000 | `observer.rs:46` |
| Engram query | `limit(16)` | `engram_fetch.rs:88` |

Unbounded or unchecked:

| Item | Location |
|---|---|
| Handshake `VecDeque<RelayMessage>` — grows with every non-AUTH/non-OK frame the relay sends during the ≤40 s handshake window | `relay.rs:3833`, filled at `:3902` and `:3960` |
| Inbound text frame size — `parse_relay_message` deserialises the whole frame with no length cap | `relay.rs:3534` |
| NIP-42 challenge length | `relay.rs:3901`, `:3617`, `:2344` |
| `deferred_commands: VecDeque<RelayCommand>` during replay — grows for as long as pacing sleeps last | `relay.rs:2504`, pushed at `:3386` |
| `active_subscriptions` / `active_filters` / `last_seen` / `subscribe_since` / `channel_dropped_since` — bounded only by the number of channels the relay says the agent is in | `relay.rs:976-1059` |
| Mid-session AUTH response rate | `relay.rs:2344-2353` |
| `relay.rs:2059` log line — full unparsed frame | `relay.rs:2059` |

The channel-count maps do get cleaned up on unsubscribe
(`clear_channel_state`, `relay.rs:1133-1140`) and on an exact access-denial
CLOSED (`relay.rs:3524`), so growth requires the relay to keep adding channels.

#### Rate-limit cooperation

The gate is defensive rather than security-critical, but it is the mechanism
that keeps the agent from being throttled off the relay: the hint is floored to
5 s, jittered, and can only extend (`relay.rs:1156-1165`); a fresh socket
deliberately does **not** clear it because the relay quota is keyed by
community+pubkey (`relay.rs:2496-2503`); drain is capped at one frame per
125 ms tick (`relay.rs:107`, `:113`).

#### HTTP bridge exposure

This module uses only the NIP-98-authenticated bridge paths — every
`POST /query` and `POST /events` signs a kind:27235 event per attempt
(`relay.rs:378-381`). The unsigned `X-Pubkey` header path, which impersonates
any pubkey when `BUZZ_REQUIRE_AUTH_TOKEN` defaults to false, is never used and
appears nowhere in these files. The relay's inert per-kind scope gate (every
NIP-42 token carries `Scope::all_known()`) is therefore not something this
module can compensate for, and it does not attempt to: it relies on the relay's
`h`-scoped membership checks and reacts to `CLOSED restricted` after the fact
(`relay.rs:3498-3529`).

#### Error-message leakage

`RelayError::AuthFailed(ok.message)` propagates the relay's raw denial string
(`relay.rs:3851`) and it is logged at warn by every retry site
(`relay.rs:3810`, `:3816`, `:2951`, `:3119`). `is_terminal_auth_failure`
(`relay.rs:3773-3775`) treats an unrecognised prefix as terminal — failing fast
on an unknown denial rather than retrying, which is the safer default and is
documented as such (`relay.rs:3768-3771`).
