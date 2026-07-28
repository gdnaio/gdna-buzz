## Module: buzz-acp — agent pool & lifecycle (`crates/buzz-acp/src`)
### Aspect: Integrations

#### ACP subprocess protocol (stdio, NDJSON JSON-RPC)

The pool does not own framing or the child handle — both live in `AcpClient` (`acp.rs`). `pool.rs` drives the protocol through typed methods on `agent.acp`:

| ACP method / call | Call site | Notes |
|---|---|---|
| `session/new` (via `session_new_full`) | `pool.rs:828-840` | passes `ctx.cwd`, `ctx.mcp_servers.clone()`, and the combined system prompt |
| `_goose/unstable/session/system-prompt/set` | `pool.rs:842-849` | probed once; `-32601` latches `goose_system_prompt_supported = Some(false)` (`pool.rs:852-858`) |
| `session/set_config_option` (model) | `pool.rs:953-960` via `ModelSwitchMethod::ConfigOption` | 5 s timeout |
| `session/set_model` (unstable) | `pool.rs:961-963` | 5 s timeout |
| `session/set_config_option` `configId: "mode"` | `pool.rs:1037-1041` | only when the mode is advertised in `session/new` (`pool.rs:924-928`) |
| `session/prompt` (with idle timeout) | `pool.rs:1611-1619` (initial message), `pool.rs:1832-1837` / `1843-1848` (blocks form) | idle + hard deadline both passed in |
| `session/cancel` (via `cancel_with_cleanup`) | `pool.rs:1646-1648`, `pool.rs:2075-2077` | timeout paths, grace = `ctx.idle_timeout` |
| `cancel_with_cleanup_grace` | `pool.rs:1863-1866` | control-signal path, fixed 5 s grace |
| Goose native steer | delivered as `SteerRequest` over a capacity-1 mpsc into the read loop (`pool.rs:646-662`) | read loop, not the pool, writes the JSON-RPC line |

Framing details the pool depends on but does not implement: line-delimited JSON with `LinesCodec::new_with_max_length(MAX_LINE_SIZE)` (`acp.rs:487`), request/response correlation by integer `id` with non-matching ids skipped (`acp.rs:979-1004`), and notifications distinguished by absence of `id` (`acp.rs:1038-1052`).

Protocol-version branching lives here: `has_system_prompt_support` treats `protocol_version >= 2` as system-prompt-capable except for `"goose"`, which requires its custom method to have succeeded (`pool.rs:173-183`). Legacy (`< 2`) agents get `[Base]` and `[Channel Canvas]` folded into the user message instead (`pool.rs:1090-1122`).

#### MCP servers

`PromptContext::mcp_servers: Vec<McpServer>` (`pool.rs:483`) is cloned into every `session/new` (`pool.rs:832`). The pool treats it as an opaque, already-built list: it neither validates the command, resolves it, nor spawns it. The list is built once in `lib.rs::build_mcp_servers` (`lib.rs:4141-4184`) from `config.mcp_command`, producing at most **one** server whose `name` is the command's file stem, `args` empty, and `env` carrying `BUZZ_RELAY_URL`, `BUZZ_PRIVATE_KEY` (bech32 nsec), and optionally `BUZZ_AUTH_TAG`. Empty `mcp_command` yields an empty vec (`lib.rs:4142-4144`).

Consequence: the MCP server process is spawned by the *agent*, from a spec the harness sent over stdio. Nothing in this module constrains that command — the only constraint anywhere is that `mcp_command` comes from harness config/CLI, not from a channel message or a persona pack (persona `mcp_servers` never reach this code path; see Features).

#### Relay (over `relay::RestClient`, HTTP)

Every relay interaction in this module is REST, not WebSocket, and every one is timeout-bounded and fail-open:

| Purpose | Function | Filter / endpoint | Timeout |
|---|---|---|---|
| Channel metadata (name, type) | `fetch_channel_info` (`pool.rs:2237-2304`) | `POST /query`, kind `KIND_NIP29_GROUP_METADATA`, `#d = channel uuid` | 3 s + 1 retry after 500 ms |
| Canvas revision | `fetch_canvas_section` (`pool.rs:2307-2363`) | `POST /query`, kind `KIND_CANVAS`, `#h`, `limit 1` | 3 s, no retry |
| Thread context | `fetch_thread_context` (`pool.rs:2676-2738`) | two filters: root by id, replies by `#e` + `#h`, kinds `KIND_STREAM_MESSAGE`/`_V2` | 3 s + 1 retry |
| DM context | `fetch_dm_context` (`pool.rs:2741-2784`) | `#h` + stream kinds, `limit` | 3 s + 1 retry |
| Author profiles | `fetch_prompt_profile_lookup` (`pool.rs:2632-2673`) | kind `Metadata` by authors | 3 s + 1 retry |
| Core memory (NIP-AE) | `engram_fetch::build_core_section` (`pool.rs:1387-1391`) | delegated | 3 s |
| Turn metric (NIP-AM) | `publish_agent_turn_metric` (`pool.rs:3416`) | `POST /events`, kind `44200` | 3 s |
| Reactions add/remove | `reaction_add` / `reaction_remove` (`pool.rs:3462`, `:3540`) | `POST /events` kind 7 / kind 5, plus a kind-7 lookup query | 500 ms add, 1 s query + 1 s delete |
| Dead-letter notice | `post_failure_notice` (`pool.rs:3495-3535`) | `POST /events` kind 9 | 5 s |

`fetch_with_retry` (`pool.rs:2219-2235`) is the shared one-retry wrapper: the closure is called at most twice with a 500 ms gap, so worst-case per fetch is ~6.5 s (`pool.rs:776-779`). Several of these run serially inside a single turn (channel info → context → profiles), so pre-prompt latency compounds.

Relay responses are not trusted blindly on the canvas path: `canvas_section_from_query_response` (`pool.rs:2366-2477`) deserializes a full `nostr::Event`, calls `event.verify()` (`pool.rs:2396-2405`), checks the kind is `KIND_CANVAS` (`:2408-2417`), and re-checks the `h` tag "to prevent a misbehaving relay from injecting a different channel's canvas" (`:2419-2432`). The other fetch paths (`fetch_channel_info`, thread/DM context, profiles) parse raw JSON fields with no signature verification (`pool.rs:2258-2287`, `:2892-2936`, `:2594-2630`).

#### `buzz-core`

| Item | Use | Site |
|---|---|---|
| `kind::KIND_NIP29_GROUP_METADATA` | channel-info filter | `pool.rs:2246` |
| `kind::KIND_CANVAS` | canvas filter + validation | `pool.rs:2312`, `:2400` |
| `kind::KIND_STREAM_MESSAGE`, `KIND_STREAM_MESSAGE_V2` | thread/DM filters | `pool.rs:2703-2706`, `:2751-2754` |
| `kind::KIND_AGENT_TURN_METRIC` | NIP-AM event | `pool.rs:3395` |
| `agent_turn_metric::{AgentTurnMetricPayload, TokenCounts, StopReason, encrypt_agent_turn_metric}` | metric build + encrypt | `pool.rs:3327`, `:3372-3390` |

`acp_stop_to_core` (`pool.rs:3305-3320`) maps ACP stop reasons onto the core enum, collapsing `MaxTurnRequests` and `Refusal` to `Unknown` — a lossy mapping that is asserted in `test_acp_stop_to_core_maps_all_variants` (`pool.rs:5098`).

#### `buzz-sdk` and `nostr`

`buzz_sdk::build_reaction` / `build_remove_reaction` / `build_message` / `ThreadRef` for the cosmetic and dead-letter events (`pool.rs:3470`, `:3595`, `:3514`). Signing always uses `rest.keys` for reactions/notices (`pool.rs:3477`, `:3521`, `:3602`) and `ctx.agent_keys` for the NIP-AM metric (`pool.rs:3402`). `nostr::EventBuilder`/`Tag::parse` build the metric event with `p` and `agent` tags (`pool.rs:3394-3400`).

#### Internal crate modules

`crate::acp` (client, `AcpError`, `McpServer`, `ModelSwitchMethod`, `StopReason`, catalog helpers) at `pool.rs:31-35`; `crate::config::{DedupMode, PermissionMode}` at `:36`; `crate::observer` at `:37`; `crate::queue` (batch, context, framing, `parse_thread_tags`, `format_prompt`, `base_section`, `slash_command_for_batch`) at `:38-41`; `crate::relay::{ChannelInfo, RestClient}` at `:42`; `crate::engram_fetch` at `:1387`. `pool_lifecycle.rs` imports nothing from the crate — only `std::time::Duration` and `tokio::time::Instant` (`pool_lifecycle.rs:7-8`), which is what makes it independently testable.

#### OS process APIs

None are called from `pool.rs` or `pool_lifecycle.rs`. All process control is in `acp.rs`, invoked by `lib.rs`:

- `tokio::process::Command::new(command)` with `.args(args)`, `stdin`/`stdout` piped, **`stderr` inherited**, `.kill_on_drop(true)` (`acp.rs:416-423`).
- `cmd.process_group(0)` on Unix so a group kill does not reach the harness's own group (`acp.rs:463-466`).
- `configure_no_window` for Windows console suppression (`acp.rs:469`, impl `acp.rs:1997`).
- `shutdown()`: `kill_process_group(pid)` when available, else `child.start_kill()`, then a 5 s bounded `child.wait()`; on expiry it logs and abandons the child (`acp.rs:372-397`).
- `Drop`: `start_kill()` only, no reap (`acp.rs:373`, `:1961`) — documented as best-effort.
- `nix` with the `signal` feature is a Unix-only dependency for the group kill (`Cargo.toml:56-58`).

The pool's contribution to process lifecycle is indirect: it decides *when* an agent is poisoned (via `PromptOutcome`) and hands the `OwnedAgent` back, and `lib.rs` then calls `agent.acp.shutdown()` inside the respawn task (`lib.rs:3672`) or the shutdown sequence (`lib.rs:2628`, `:2645`, `:3700-3712`).

#### Desktop / observer consumers

`session_config_captured` carries `configOptions`, `modes`, `models`, `modelOverridden`, and `relayUrl` so the desktop can key its cache by `(agent, relay)` (`pool.rs:906-922`). `AgentModelCapabilities` is documented as feeding the desktop's `get_agent_models` Tauri command (`pool.rs:70-73`); that command is a separate out-of-process Tauri handler (`desktop/src-tauri/src/commands/agent_models.rs:29`) and cannot read this struct — the in-process reader is `switch_idle_agent_model` (`pool.rs:748-756`).
