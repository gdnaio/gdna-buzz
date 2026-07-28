## Module: buzz-acp — ACP protocol, config & setup mode (`crates/buzz-acp/src`)
### Aspect: Integrations

#### The agent subprocess contract

`AcpClient::spawn` (`acp.rs:408-497`) is the whole contract:

| Aspect | Value | Line |
|---|---|---|
| Launcher | `tokio::process::Command::new(command)` — no shell, argv passed via `.args(args)` | `acp.rs:416-417` |
| stdin | `Stdio::piped()` | `acp.rs:418` |
| stdout | `Stdio::piped()` | `acp.rs:419` |
| stderr | **`Stdio::inherit()`** — "so agent logs are visible in the harness terminal" | `acp.rs:420-421` |
| Drop behaviour | `.kill_on_drop(true)`, with a comment that callers must still `await shutdown()` | `acp.rs:422-424` |
| Process group | `cmd.process_group(0)` under `#[cfg(unix)]` — PID == PGID | `acp.rs:463-467` |
| Windows | `configure_no_window(&mut cmd)` sets `CREATE_NO_WINDOW` (0x0800_0000) | `acp.rs:469-471`, impl `acp.rs:1997-2006` |
| Working directory | **not set** — the child inherits the harness's cwd; the ACP `cwd` is only a `session/new` parameter | `acp.rs:416-471`, `acp.rs:566` |
| Missing pipes | `AcpError::Protocol("failed to open agent stdin"/"…stdout")` | `acp.rs:475-483` |

The command is resolved by the OS through the inherited `PATH`. The shipped default is the bare name `goose` — `config.rs:250` for the harness and `config.rs:191` for the auxiliary subcommands.

Five spawn call sites, differing only in whether persona env and the Codex signal are passed:

| Site | Args | Purpose |
|---|---|---|
| `lib.rs:3749` | full `extra_env`, real signal | pool agent spawn |
| `lib.rs:3856` | `command, args, extra_env, has_generated_codex_config` | shared spawn+initialize helper |
| `lib.rs:3887` | `&[], false` | `spawn_auth_client` for auth subcommands |
| `lib.rs:4017` | `&[], false` | `run_models` |
| `pool.rs:4983`, `pool.rs:5041` | test spawns | steer-invariant tests |
| `lib.rs:5178` | `AcpClient::spawn("cat", …)` | test double — `cat` echoes stdin back as stdout |

#### MCP server declaration — the credential channel

`build_mcp_servers` (`lib.rs:4141-4180`) is the only producer of `Vec<McpServer>`:

- Returns an **empty vec** when `config.mcp_command` is empty (`lib.rs:4142-4144`), which is the default (`config.rs:261`) — a stock run gives the agent zero MCP servers.
- Server `name` is derived from the command's file stem, defaulting to `"mcp"` (`lib.rs:4146-4150`); `args` is always empty (`lib.rs:4152`).
- `env` always carries `BUZZ_RELAY_URL` and `BUZZ_PRIVATE_KEY`, the latter as the **raw bech32 `nsec1…`** from `config.keys.secret_key().to_bech32()` with an `.expect(...)` (`lib.rs:4155-4170`).
- `BUZZ_AUTH_TAG` is forwarded from the harness's own environment when non-empty (`lib.rs:4171-4179`).

Flow to the wire: `pool.rs:832` clones the vec into each `session_new_full` call → `acp.rs:568-571` serializes it as `params.mcpServers` → `write_ndjson` (`acp.rs:951`) puts it on the child's stdin. `McpServer.mcp_servers` is stored on the per-agent context at `pool.rs:483`.

#### `lib.rs` / `pool.rs` call sites into this module

| Consumer | Calls | Line |
|---|---|---|
| Startup | `config::propagate_legacy_env_vars()` | `lib.rs:1234` |
| Startup | `setup_mode::SetupPayload::from_env()` then `run_setup_listener` | `lib.rs:1290-1295` |
| Startup | `config.summary()` for the boot log | `lib.rs:1296` |
| Startup | base-prompt selection from `config.no_base_prompt` / `base_prompt_content` / `include_str!` | `lib.rs:1529-1545` |
| Subcommands | `ModelsArgs::parse_from` → `run_models` | `lib.rs:1252-1253` |
| Subcommands | `AuthMethodsArgs::parse_from` → `run_auth_methods` | `lib.rs:1262-1263` |
| Subcommands | `AuthenticateArgs::parse_from` → `run_authenticate`, which calls `client.authenticate(&args.method_id)` under an `AUTHENTICATE_TIMEOUT` | `lib.rs:1272-1273`, `lib.rs:3983` |
| Spawn | `config.persona_env_vars.clone()` as `extra_env` | `lib.rs:1762`, `lib.rs:3488`, `lib.rs:3666`, `lib.rs:3733` |
| Handshake | `init_result["protocolVersion"].as_u64().unwrap_or(1) as u32` | `lib.rs:3776`, `lib.rs:3864` |
| Session | `session_new_full(&ctx.cwd, ctx.mcp_servers.clone(), session_new_system_prompt(...))` | `pool.rs:830-836` |
| Session | `session_set_goose_system_prompt`, gated on `goose_system_prompt_supported != Some(false)` | `pool.rs:838-846` |
| Session | `-32601` latch: `Err(AcpError::AgentError { code: -32601, .. })` marks the goose system-prompt method unsupported | `pool.rs:849`, documented `pool.rs:341` |
| Turn | `install_steer_rx` / `clear_steer_rx` | `pool.rs:5006`, `pool.rs:5032`, `pool.rs:5089`; cleared at `pool.rs:1243` |
| Filters | `config::resolve_channel_filters`, `resolve_dynamic_channel_filter`, `load_rules` | `setup_mode.rs:375`, `setup_mode.rs:578`, `setup_mode.rs:530` |

`setup_mode` reaches back into `lib.rs` for shared gates: `crate::resolve_agent_owner` (`setup_mode.rs:346`, defined `lib.rs:123`), `crate::OwnerCache::new` (`setup_mode.rs:347`), `crate::author_allowed` (`setup_mode.rs:433`), `crate::event_mentions_agent` (`setup_mode.rs:424`), `crate::is_dm_channel` (`setup_mode.rs:432`), `crate::queue::parse_thread_tags` (`setup_mode.rs:604`), and `crate::pool::ChannelInfoResolver::new` (`setup_mode.rs:383`).

#### `CODEX_CONFIG` — the Codex adapter integration

Two-part mechanism spanning both files:

- `config.rs:646-677` generates the env pair for normalized `codex` / `codex-acp` commands only, and only when the relay URL parses with a host. The doc comment (`config.rs:628-645`) explains the reason: Codex sandboxes MCP subprocesses (including `buzz-cli`) behind a macOS Seatbelt policy that blocks all outbound network, so `buzz-cli` cannot reach the relay WebSocket without it. The value is forwarded by the `@agentclientprotocol/codex-acp` adapter (1.x) as a session-level config override (`CODEX_CONFIG` → `thread/start config`), equivalent to the TOML `sandbox_workspace_write.network_access = true`, which sets `NetworkSandboxPolicy::Enabled` and emits `(allow network-outbound)` in the Seatbelt policy — full outbound TCP/TLS at the OS level.
- `acp.rs:257-345` merges persona entries, additional generated entries, and the parent process's own `CODEX_CONFIG` (parent wins on collisions at every nesting level), then **force-sets `sandbox_workspace_write.network_access = true`** as the final step (`acp.rs:329-343`). That single key is the only thing forced; everything else in the object is whatever the merge produced.

#### Goose-specific integrations

| Integration | Wire surface | Line |
|---|---|---|
| Custom notification opt-in | `_meta.goose.customNotifications = true` in `initialize` capabilities; without it goose suppresses the notification and no usage data arrives | `acp.rs:357-360` |
| Usage notifications | `_goose/unstable/session/update`, deserialized into `GooseSessionUpdateNotification` | `acp.rs:1637-1678` |
| System-prompt append | `_goose/unstable/session/system-prompt/set` with `mode:"append"`, `key:"buzz"` | `acp.rs:605-619` |
| Active-run tracking | `session_info_update` → `params.update._meta.goose.activeRunId`; the doc comment cites goose's own `crates/goose/src/acp/server.rs:2277 send_active_run_update` | `acp.rs:176-186`, `acp.rs:1590-1620` |
| Non-cancelling steer | `_goose/unstable/session/steer` with `expectedRunId` | `acp.rs:1338-1355` |

`acp.rs:1594-1597` notes that buzz-agent also emits `session_info_update` with the same field, and that other agents leave `active_run_id` as `None` so steer callers fall back to cancel+merge. The `_meta` position is documented as being on the update object itself (`params.update._meta`), not at params level (`acp.rs:1600-1604`).

A third non-standard capability, `_meta["terminal-auth"] = true`, is described as a claude-agent-acp extension for advertising the terminal login argv, relying on other adapters ignoring unknown `_meta` keys (`acp.rs:361-366`).

#### External crate dependencies exercised by this group

| Crate | Use | Line |
|---|---|---|
| `tokio` (`process`, `time`, `io`, `sync`) | subprocess, deadlines, `select!`, oneshot/mpsc | `acp.rs:13-14`, `acp.rs:1276-1360` |
| `tokio-util` `LinesCodec` / `FramedRead` | bounded NDJSON framing | `acp.rs:15`, `acp.rs:487` |
| `futures-util` `StreamExt` | `reader.next()` | `acp.rs:12` |
| `serde_json` | all wire construction and parsing | throughout |
| `thiserror` | `AcpError` (`acp.rs:78`), `ConfigError` (`config.rs:38`) |
| `nix` (`signal` feature, `default-features = false`) | `killpg` — **Unix-only target dependency** | `Cargo.toml:76-77`, used `acp.rs:1979-1986` |
| `clap` v4 (`derive`, `env`) | the whole CLI surface | `Cargo.toml:66` |
| `toml` 1.0 | `load_rules` | `Cargo.toml:69`, `config.rs:1065` |
| `evalexpr` | rule filter compilation at load time | `Cargo.toml:72`, `config.rs:1110` |
| `url` | relay-URL validation in `codex_network_env` | `config.rs:13`, `config.rs:654` |
| `uuid` | channel ids in filters | `config.rs:15` |
| `nostr` | `Keys`, `EventId`, `Tag`, event building | `config.rs:11`, `setup_mode.rs:40` |
| `anyhow` | setup-mode error type | `setup_mode.rs:36` |

`Cargo.toml:76-77` carries a comment pointing at the `#[cfg(not(unix))]` fallback in `acp.rs`, so the Unix-only scoping is deliberate. The `nix` dependency is pinned to `0.31` and the `signal` feature is chosen specifically so `killpg` can be called without `unsafe` (`acp.rs:1975-1977`).

#### Relay-side integration in setup mode

`run_setup_listener` uses `HarnessRelay` and `RelayEventPublisher` from `relay.rs` (`setup_mode.rs:73-77`):

| Step | Call | Line |
|---|---|---|
| Connect | `HarnessRelay::connect(&config.relay_url, &config.keys, &pubkey_hex, relay_auth_tag)` | `setup_mode.rs:330-333` |
| Watermark | `relay.set_startup_watermark(startup_watermark)` — failure warns only | `setup_mode.rs:334-336` |
| Membership | `relay.subscribe_membership_notifications()` — failure is fatal | `setup_mode.rs:338-341` |
| Discovery | `relay.discover_channels()` (REST) — failure is fatal | `setup_mode.rs:351-354` |
| Channel subs | `relay.subscribe_channel(*channel_id, filter.clone())` — per-channel failure warns only | `setup_mode.rs:379-382` |
| Publisher | `relay.event_publisher()`, `relay.rest_client()` | `setup_mode.rs:383-384` |
| Reconnect | `relay.reconnect()` when the event stream ends | `setup_mode.rs:394` |
| Publish | `buzz_sdk::build_message(...)` → `sign_with_keys(keys)` → `publisher.publish_event(signed)` | `setup_mode.rs:626-644` |

`BUZZ_AUTH_TAG` is parsed via `buzz_sdk::nip_oa::parse_auth_tag` and silently dropped when empty or unparseable (`setup_mode.rs:319-322`).
