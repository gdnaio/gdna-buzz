## Module: buzz-cli — dispatch, relay client & validation (`crates/buzz-cli/src`)
### Aspect: Integrations

#### Relay over HTTP

All HTTP goes through one `reqwest::Client` built in `BuzzClient::new`
(`client.rs:547-551`). Every request is signed per attempt; there is no session
or token.

| Method + path | Auth header(s) | Body | Site |
|---|---|---|---|
| `POST /query` | `Authorization: Nostr <b64 kind-27235>`, `Content-Type: application/json`, optional `x-auth-tag` | JSON array of filters | `client.rs:773-801` |
| `POST /count` | same | `[filter]` | `client.rs:803-834` (dead code) |
| `POST /events` | same | one signed event | `client.rs:873-1022` (moderation), `client.rs:1024-1071` (stored) |
| `PUT /upload` | `Authorization: Nostr <b64url kind-24242 t=upload>`, `Content-Type: <sniffed mime>`, `X-SHA-256`, optional `x-auth-tag` | raw file bytes | `client.rs:1152-1178` |
| `PUT /media/upload` | same | raw bytes | legacy fallback, `client.rs:1195-1226` |
| `GET /media/<sha256[.ext]>` | `Authorization: Nostr <b64url kind-24242 t=get>` + optional `x-auth-tag` | — | `client.rs:1229-1256` |
| `GET <path>` authed | NIP-98 GET over the full URL | — | `client.rs:836-856`; callers pass `/moderation/reports?…`, `/moderation/restricted`, `/moderation/audit?limit=…` (`commands/moderation.rs:114,120,127`) |
| `GET /` | none; `Accept: application/nostr+json` | — | `client.rs:753-765`; NIP-11 read from `commands/agents.rs:273` |

NIP-98 construction (`client.rs:84-110`): kind 27235, empty content, tags `u`
(full URL incl. query), `method`, `nonce` (UUIDv4), and `payload`
(hex SHA-256) when a body is present; header value is
`Nostr <base64-standard(event json)>`. No `expiration` tag is added
(`grep -n 'expiration' client.rs` shows it only on the two Blossom paths,
`client.rs:330` and `client.rs:364`).

Blossom (BUD-01 get / BUD-02 upload) uses kind 24242 with URL-safe,
unpadded base64 (`client.rs:325-348`, `client.rs:350-385`). Get auth carries
`t=get`, `expiration`, `server=<authority>` and deliberately **no `x` tag**
(asserted by `media_get_auth_header_is_server_scoped`, `client.rs:452-481`).
Upload auth carries `t=upload`, `x=<sha256>`, `expiration`, and `server` when the
authority is non-empty.

Response handling is centralized in `handle_response` (`client.rs:1258-1289`):
non-2xx bodies are parsed for an `error` or `message` field, and a 403 gets a
hint appended when `BUZZ_AUTH_TAG` is set in the environment
(`client.rs:1271-1279`).

#### Relay over WebSocket — shared client, not reimplemented

`publish_ephemeral_event` (`client.rs:1073-1098`) converts the HTTP base back to
`ws(s)://` (`to_ws_url`, `client.rs:1299-1305`) and delegates the entire
connect → NIP-42 AUTH → EVENT → OK → close sequence to
`buzz_ws_client::publish_event` (`client.rs:1084`), which is the shared crate
declared at `Cargo.toml:75-76`. The inner ceilings it relies on
(`AUTH_CHALLENGE_TIMEOUT_SECS = 20`, `AUTH_OK_TIMEOUT_SECS = 20`,
`PUBLISH_OK_TIMEOUT_SECS = 30` — `crates/buzz-ws-client/src/connection.rs:17-23`)
sum to 70 s, and the CLI passes a 75 s outer budget with a comment citing those
exact constants (`client.rs:1075-1085`) — a rare in-code cross-reference that is
still accurate.

There is **no duplicated WebSocket or NIP-42 logic** in this group:
`grep -n 'tokio_tungstenite\|WebSocket\|AUTH' client.rs` finds no connection
code, only the delegation. The one gap is error fidelity: any WS failure —
including NIP-42 auth rejection and timeout — collapses to
`CliError::Other(e.to_string())` (`client.rs:1084`), which exits **4**, whereas
the HTTP equivalents exit 2 or 3. A relay OK with `accepted:false` is mapped to
a synthetic `Relay{status:400}` (`client.rs:1090-1096`), i.e. exit 2.

#### Workspace crate dependencies

| Crate | Used for | Site |
|---|---|---|
| `buzz-ws-client` | ephemeral publish (kinds 20000-29999 are WS-only) | `client.rs:1084`, `Cargo.toml:75-76` |
| `buzz-core` | `tenant::relay_url_authority` for the Blossom `server` tag (`client.rs:113`); `observer::{encrypt_observer_payload, OBSERVER_FRAME_TELEMETRY}` (`agent_management.rs:3`) |
| `buzz-sdk` | `nip_oa::{parse_auth_tag, verify_auth_tag}` (`lib.rs:1757-1763`); `build_agent_observer_frame` (`agent_management.rs:113`); `SdkError` mapping (`validate.rs:155-160`) |
| `buzz-persona` | persona pack parse/validate — reached only via `commands/pack.rs`; declared at `Cargo.toml:67-68` |
| `nostr` | keys, event builders, tags, `Kind`, `Timestamp` | throughout `client.rs`, `lib.rs:10` |
| `rustls` (ring) | process-level CryptoProvider install | `lib.rs:39`, `Cargo.toml:78-84` |

Third-party integrations: `reqwest` (HTTP), `infer` (magic-byte MIME sniffing,
`client.rs:1109-1111`), `sha2` + `hex` (NIP-98 payload hash, Blossom hash —
`client.rs:106`, `client.rs:1133`), `base64` (both standard and URL-safe
alphabets — `client.rs:109`, `client.rs:326`), `url` (origin comparison,
`client.rs:277`, `client.rs:302`), `bytes` (request bodies and download return),
`uuid` (nonce + validation), `chrono` (RFC3339 observer timestamps,
`agent_management.rs:97`), `rand` (jitter, `client.rs:134`), `diffy`/`dirs`
(declared here, used by sibling command modules).

Every declared dependency resolves to a real use: `diffy` in `commands/mem.rs`,
`dirs` in `commands/channel_templates.rs`, `buzz-persona` in `commands/pack.rs`,
`tempfile`/`axum` in `#[cfg(test)]` code (`client.rs:2123`, `client.rs:1587-1591`).
I found no unused dependency. Two Cargo.toml *comments* are stale, though: line
19 claims clap's env support is "(BUZZ_API_TOKEN auto-wired)" — that variable is
never read by this crate (`grep -rn 'BUZZ_API_TOKEN' crates/buzz-cli/src` → no
matches; it exists only in `buzz-acp` and `buzz-workflow`) — and line 44
describes `nostr` as "used in `buzz auth`, auto-mint", but there is no `auth`
subcommand (`command_inventory_is_stable`, `lib.rs:1808-1829`, lists 21 groups
and none is `auth`).

#### Code duplicated instead of shared

- `extract_relay_message_field` (`client.rs:190-198`) is the named helper for
  pulling `error`/`message` out of a relay error body, yet the same logic is
  re-inlined twice: in `submit_moderation_event` (`client.rs:985-996`, with the
  comment "Map the body through `handle_response`'s error logic inline") and in
  `handle_response` itself (`client.rs:1261-1270`). Three copies of one rule.
- `resp_was_success` (`client.rs:217-219`) re-implements
  `reqwest::StatusCode::is_success` for `u16`, needed only because
  `submit_moderation_event` consumes the response before checking status.
- The 403 `BUZZ_AUTH_TAG` hint is duplicated verbatim in
  `submit_moderation_event` (`client.rs:991-997`) and `handle_response`
  (`client.rs:1271-1279`).

#### Subprocesses, filesystem, keychain

- **No subprocesses.** `grep -n 'Command::new\|process::Command' lib.rs client.rs validate.rs error.rs agent_management.rs main.rs`
  → zero matches; the only `spawn` calls are `tokio::spawn` inside test servers
  (`client.rs:1634`, `:1873`, `:1989`, `:2052`, `:2140`, `:2220`). Nothing in
  this group injects env vars into a child — the CLI is itself the child that
  the ACP harness configures.
- **Filesystem reads:** `upload_file` stats and slurps an arbitrary path
  (`client.rs:1102-1108`); `read_file_or_stdin` reads an arbitrary path
  (`validate.rs:189-192`); `read_or_stdin`/`read_file_or_stdin` read stdin
  (`validate.rs:164-170`, `:182-187`). No writes: `grep -n 'fs::write\|File::create' lib.rs client.rs validate.rs error.rs agent_management.rs`
  → matches only inside `#[cfg(test)]` (`validate.rs:492`).
- **No keychain or OS credential store.**
  `grep -rn 'keychain\|security_framework' crates/buzz-cli/src` → zero matches.
  Secrets arrive purely through env/flags.

#### Multicall integration with `buzz-dev-mcp`

`buzz-dev-mcp` re-exports this CLI: when its argv[0] resolves to `buzz` it calls
`buzz_cli::run_from_args` and exits with the returned code
(`crates/buzz-dev-mcp/src/lib.rs:167-170`). Both processes install the ring
provider (`crates/buzz-dev-mcp/src/lib.rs:165`, `lib.rs:39`), which is exactly
the double-install the `let _ =` swallow accommodates. This coupling is
documented in the code (`lib.rs:30-38`) and in `Cargo.toml:78-83`.
