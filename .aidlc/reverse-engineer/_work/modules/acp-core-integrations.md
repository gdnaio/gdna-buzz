## Module: buzz-acp — harness core & orchestration (`crates/buzz-acp/src`)
### Aspect: Integrations

#### Internal crate dependencies (`Cargo.toml:19-22`)

| Crate | Declared | Actually used by `lib.rs` |
|---|---|---|
| `buzz-core` | `Cargo.toml:20` | yes — `kind::*` (`lib.rs:23-26`, `lib.rs:82`), `observer::{decrypt_observer_payload, encrypt_observer_payload, OBSERVER_FRAME_TELEMETRY, OBSERVER_MAX_PLAINTEXT_LEN}` (`lib.rs:27-30`), `verify_event` (`lib.rs:845`) |
| `buzz-sdk` | `Cargo.toml:21` | yes — `nip_oa::verify_auth_tag` (`lib.rs:128`, `lib.rs:350`), `nip_oa::parse_auth_tag` (`lib.rs:1341`), `build_agent_observer_frame` (`lib.rs:810`) |
| `buzz-persona` | `Cargo.toml:22` | **no** — zero references. `grep -rn 'buzz_persona\|buzz-persona' crates/buzz-acp/` returns only `Cargo.toml:22`. The only `persona`-named thing in the crate is `Config::persona_env_vars` (`config.rs:533-535`), a plain `Vec<(String,String)>` populated in `config.rs:945-999` and threaded to `AcpClient::spawn` (`lib.rs:1762`, `3488`, `3666`, `3733`) without touching the crate. |

`buzz-persona` is also the only path-declared internal dep (`{ path = "../buzz-persona" }`) rather than `workspace = true`.

#### Relay: WebSocket primary, HTTP bridge secondary — both used

`buzz-acp` does **not** depend on `buzz-ws-client`. It reimplements the client stack in `relay.rs`: its own `connect_async` (`relay.rs:3825`), `RelayMessage` enum (`relay.rs:471`), AUTH challenge wait (`relay.rs:3865`), `send_auth_response` (`relay.rs:3433`), background task (`relay.rs:1534`), and reconnect state machine (`relay.rs:2893`, `3022`). `Cargo.toml` pulls `tokio-tungstenite` directly (`Cargo.toml:31`) instead of relying on the shared crate. It also diverges functionally: `relay.rs` re-authenticates on a mid-session AUTH challenge, where `buzz-ws-client` only records it.

`lib.rs`'s own contributions to that duplication:

| Concern | `lib.rs` site |
|---|---|
| Presence must go over WS, not HTTP | `publish_presence` `lib.rs:77-91`; the doc comment at `lib.rs:71-72` states ephemeral kinds 20000–29999 are rejected by the HTTP bridge |
| Typing indicators over WS with non-blocking `try_publish_event` | `lib.rs:2332-2340`; the comment at `lib.rs:2329-2331` cites issue #35 — they must not block the loop during reconnect |
| Observer frames over WS via `RelayEventPublisher` | `lib.rs:790-833` |
| Startup watermark handed to the background task so the first REQ uses `watermark − 5s` instead of `since=now` | `lib.rs:1348-1354` |
| Graceful WS close on exit (5 s wait, not abort) | `relay.shutdown()` `lib.rs:2690`; comment cites issue #40 |

HTTP bridge usage reached from `lib.rs`:

| Call | Site | Path |
|---|---|---|
| `rest_client.query(&[filter])` for a kind:0 profile during sibling verification | `lib.rs:310-315` | `POST /query` |
| `pool::reaction_add` / `reaction_remove` (👀) | `lib.rs:2204-2213`, `lib.rs:2000-2010` | via `RestClient` |
| `pool::post_failure_notice` | `lib.rs:3014-3031` | via `RestClient` |
| `ChannelInfoResolver` lazy kind:39000 fetch backing `is_dm_channel` | `lib.rs:273-286` → `pool::ChannelInfoResolver::resolve` | `POST /query` |
| channel discovery at startup | `relay.discover_channels()` `lib.rs:1437-1443` | REST |

`RestClient` signs every request with NIP-98 (`relay.rs:264-307`, header applied `relay.rs:380-385`) rather than relying on an `X-Pubkey` header, so the harness is not exposed to the unsigned-header impersonation path that `BUZZ_REQUIRE_AUTH_TOKEN=false` leaves open on the relay bridge. That said, the DM classification that the author gate fails closed on (`lib.rs:273-286`) depends on this HTTP path succeeding — a bridge outage degrades every non-startup channel to "treated as DM", collapsing `allowlist`/`anyone` to owner-only.

#### ACP subprocess protocol

`lib.rs` drives `acp::AcpClient` over stdio JSON-RPC. Sites:

| Operation | `lib.rs` site | `acp.rs` |
|---|---|---|
| spawn | `initialize_agent_pool` `lib.rs:3755-3760`, `spawn_and_init` `lib.rs:3856-3858`, `spawn_auth_client` `lib.rs:3885-3888` | `acp.rs:408-503` |
| `initialize` (60 s timeout in pool startup) | `lib.rs:3765`, `lib.rs:3861` | `acp.rs:539` |
| `session/new` (models probe only, in `lib.rs`) | `lib.rs:4048` | `acp.rs:563` |
| `authenticate` | `lib.rs:3985` | `acp.rs:549` |
| `session/prompt` | delegated to `pool::run_prompt_task` (`lib.rs:2971-2979`, `lib.rs:3563-3572`) | `acp.rs:654`, `676` |
| steer channel install | `agent.acp.install_steer_rx(rx)` `lib.rs:2934` | `acp.rs:800` |
| `shutdown` | `lib.rs:2610`, `2637`, `2645`, `3672`, `3698`, `3707`, `3711`, `3866` | `acp.rs:376` |

Protocol version is read as `init_result["protocolVersion"].as_u64().unwrap_or(1)` (`lib.rs:3785`, `lib.rs:3864`) — a missing or non-numeric field silently becomes legacy v1, which changes prompt composition (`pool::prepend_base_for_legacy`, pinned by tests at `lib.rs:4193-4216`).

Agent identity is normalized from `agentInfo.name` with `serverInfo.name` as fallback (`normalized_agent_name` `lib.rs:3686-3695`), lowercased and trimmed. `run_models` checks the same two keys in the **opposite** order (`serverInfo` first, `lib.rs:4062-4064`) — an agent that sets both would report different names depending on the code path.

`AcpClient::spawn` inherits the parent environment by default and only injects `extra_env` keys **not already set** in the parent (`acp.rs:456-461`, operator-wins). Consequence: the harness's own `BUZZ_PRIVATE_KEY` / `BUZZ_RELAY_URL` / `BUZZ_AUTH_TAG` are inherited by every agent subprocess implicitly, in addition to the explicit `mcpServers` env injection described below.

#### MCP server integration

`build_mcp_servers` (`lib.rs:4141-4184`) produces at most **one** `McpServer`, gated on `config.mcp_command` being non-empty (`lib.rs:4142-4144`). Default is `""` (`config.rs:261`), so an out-of-the-box run hands the agent zero MCP servers.

Server name is `Path::file_stem()` of the command, falling back to `"mcp"` (`lib.rs:4146-4150`). `args` is always empty (`lib.rs:4152`). Injected env:

| Var | Line | Value |
|---|---|---|
| `BUZZ_RELAY_URL` | `lib.rs:4155-4158` | `config.relay_url` |
| `BUZZ_PRIVATE_KEY` | `lib.rs:4159-4170` | `config.keys.secret_key().to_bech32()` — the raw `nsec1…` |
| `BUZZ_AUTH_TAG` | `lib.rs:4171-4180` | forwarded from the harness env, skipped when empty |

This `McpServer` list flows to `PromptContext.mcp_servers` (`lib.rs:1531`) → `pool.rs:832` → `acp.rs:566-571` where it becomes the `mcpServers` field of the `session/new` JSON-RPC request. **The private key travels over the agent's stdin pipe as part of that request body**, not via argv or the process environment of the ACP child itself. See the Security aspect for the observer-feed consequence.

#### Nostr / crypto libraries

`nostr` crate for `Event`, `EventBuilder`, `Keys`, `PublicKey`, `Filter`, `Tag`, `ToBech32` (`lib.rs:38`). `rustls` with the `ring` provider installed once at startup for `wss://` (`Cargo.toml:33`, `lib.rs:1241-1243`). NIP-44 encrypt/decrypt of observer payloads is delegated to `buzz-core` (`lib.rs:797`, `lib.rs:871`), so `lib.rs` holds no crypto primitives of its own.

#### Desktop / launcher integration

Two env vars read directly in `lib.rs` come from the desktop launcher, not the CLI:

| Var | Line | Role |
|---|---|---|
| `BUZZ_AUTH_TAG` | `lib.rs:125`, `lib.rs:1338-1341`, `lib.rs:4173` | NIP-OA owner attestation: owner resolution, relay membership delegation, MCP forwarding |
| `BUZZ_MANAGED_AGENT_START_NONCE` | `lib.rs:1503` | correlation ID stamped into every `managed_agent_runtime_lifecycle` observer event (`lib.rs:93-121`) |

`BUZZ_ACP_SETUP_PAYLOAD` (`setup_mode.rs:83`, read `setup_mode.rs:214`) is the desktop's signal that the agent is not ready; it diverts to the setup listener at `lib.rs:1298-1303`.

Lifecycle states emitted for the launcher: `listening` (`lib.rs:1517-1526`), `waking` (`lib.rs:1714-1721`), `ready` (`lib.rs:2514-2521`), `failed` (`lib.rs:2482-2489` and `lib.rs:2534-2541`).
