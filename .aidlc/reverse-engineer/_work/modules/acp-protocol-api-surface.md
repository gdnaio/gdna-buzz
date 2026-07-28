## Module: buzz-acp — ACP protocol, config & setup mode (`crates/buzz-acp/src`)
### Aspect: API Surface

#### `AcpClient` — lifecycle

| Signature | Line |
|---|---|
| `pub async fn spawn(command: &str, args: &[String], extra_env: &[(String,String)], has_generated_codex_config: bool) -> Result<Self, AcpError>` | `acp.rs:408-413` |
| `pub async fn shutdown(&mut self)` | `acp.rs:376` |
| `impl Drop for AcpClient` | `acp.rs:1953-1967` |

#### `AcpClient` — protocol methods

| Signature | JSON-RPC method sent | Line |
|---|---|---|
| `pub async fn initialize(&mut self) -> Result<serde_json::Value, AcpError>` | `initialize` | `acp.rs:539-544` |
| `pub async fn authenticate(&mut self, method_id: &str) -> Result<Value, AcpError>` | `authenticate` (`params.methodId`) | `acp.rs:549-554` |
| `pub async fn session_new_full(&mut self, cwd: &str, mcp_servers: Vec<McpServer>, system_prompt: Option<&str>) -> Result<SessionNewResponse, AcpError>` | `session/new` | `acp.rs:563-588` |
| `pub async fn session_new(...) -> Result<String, AcpError>` | `session/new` (wrapper) | `acp.rs:592-599`, `#[allow(dead_code)]` at `acp.rs:591` |
| `pub async fn session_set_goose_system_prompt(&mut self, session_id: &str, text: &str) -> Result<Value, AcpError>` | `_goose/unstable/session/system-prompt/set` with `mode:"append"`, `key:"buzz"` | `acp.rs:605-619` |
| `pub async fn session_set_config_option(&mut self, session_id, config_id, value: &str) -> Result<Value, AcpError>` | `session/set_config_option` | `acp.rs:623-634` |
| `pub async fn session_set_model(&mut self, session_id, model_id: &str) -> Result<Value, AcpError>` | `session/set_model` (unstable) | `acp.rs:638-650` |
| `pub async fn session_prompt_with_idle_timeout(&mut self, session_id, prompt_text, idle_timeout, max_duration) -> Result<StopReason, AcpError>` | `session/prompt` (single block) | `acp.rs:654-670` |
| `pub async fn session_prompt_blocks_with_idle_timeout(&mut self, session_id, prompt_blocks: &[&str], idle_timeout, max_duration) -> Result<StopReason, AcpError>` | `session/prompt` (N blocks) | `acp.rs:676-745` |
| `pub async fn session_cancel(&mut self, session_id: &str) -> Result<(), AcpError>` | `session/cancel` **notification** | `acp.rs:747-752` |
| `pub async fn cancel_with_cleanup(&mut self, session_id, _idle_timeout) -> Result<StopReason, AcpError>` | permission-cancel + `session/cancel` + drain | `acp.rs:837-875` |
| `pub async fn cancel_with_cleanup_grace(&mut self, session_id, grace: Duration) -> Result<StopReason, AcpError>` | same, bounded window | `acp.rs:881-895` |

`_goose/unstable/session/steer` has no public wrapper — it is written only from inside the read loop's steer arm (`acp.rs:1338-1355`), because `expectedRunId` must be sampled at write time.

#### `AcpClient` — accessors and per-turn plumbing

| Signature | Line | Notes |
|---|---|---|
| `pub fn set_observer(&mut self, observer: Option<ObserverHandle>, agent_index: usize)` | `acp.rs:503` | |
| `pub fn set_observer_context(&mut self, context: ObserverContext)` | `acp.rs:509` | |
| `pub(crate) fn observer_handle(&self) -> Option<ObserverHandle>` | `acp.rs:514` | |
| `pub(crate) fn observer_agent_index(&self) -> Option<usize>` | `acp.rs:519` | |
| `pub fn observe(&self, kind: impl Into<String>, payload: serde_json::Value)` | `acp.rs:524` | |
| `pub fn has_in_flight_prompt(&self) -> bool` | `acp.rs:755` | `last_prompt_id.is_some()` |
| `pub fn active_run_id(&self) -> Option<&str>` | `acp.rs:769` | `#[cfg_attr(not(test), allow(dead_code))]` (`acp.rs:768`) — kept public only for tests |
| `pub fn take_turn_usage(&mut self) -> Option<TurnUsage>` | `acp.rs:783` | at most once per turn |
| `pub fn install_steer_rx(&mut self, rx: mpsc::Receiver<pool::SteerRequest>)` | `acp.rs:800` | **panics** via `assert!` if one is already installed (`acp.rs:801-805`) |
| `pub fn clear_steer_rx(&mut self)` | `acp.rs:815` | idempotent |
| `pub fn steer_rx_is_none(&self) -> bool` | `acp.rs:824` | `#[cfg(test)]` (`acp.rs:823`) |
| `pub async fn drain_stale_responses(&mut self, drain_timeout: Duration)` | `acp.rs:1023` | `#[allow(dead_code)]` — "not yet wired" (`acp.rs:1022`) |

Private internals: `write_ndjson` (`acp.rs:951`), `send_request` (`acp.rs:979`), `send_notification` (`acp.rs:1047`), `read_until_response` (`acp.rs:1074`), `read_until_response_with_idle_timeout` (`acp.rs:1198-1204`), `cancel_with_cleanup_until` (`acp.rs:897`), `handle_session_update` (`acp.rs:1527`), `handle_goose_usage_update` (`acp.rs:1637`), `handle_permission_request` (`acp.rs:1680`), `parse_stop_reason` (`acp.rs:1758`).

#### Module-level free functions in `acp.rs`

| Signature | Visibility | Line |
|---|---|---|
| `pub fn extract_model_config_options(result: &Value) -> Vec<Value>` | pub | `acp.rs:1851` |
| `pub fn extract_model_state(result: &Value) -> Option<Value>` | pub | `acp.rs:1866` |
| `pub fn resolve_model_switch_method(session_new_result: &Value, desired_model: &str) -> Option<ModelSwitchMethod>` | pub | `acp.rs:1876-1879` |
| `pub fn model_in_catalog(config_options: &[Value], available_models: Option<&Value>, desired_model: &str) -> bool` | pub | `acp.rs:1922-1926` |
| `pub(crate) fn build_codex_config_env(extra_env, parent_codex_config, has_generated_codex_config) -> Result<Option<String>, AcpError>` | crate | `acp.rs:257-261` |
| `fn deep_merge(base: &mut Map, overlay: Map)` | private | `acp.rs:208-211` |
| `fn agent_error_from_json(error: &Value) -> AcpError` | private | `acp.rs:115` |
| `fn kill_process_group(pid: u32) -> bool` | private, `#[cfg(unix)]` / `#[cfg(not(unix))]` | `acp.rs:1979`, `acp.rs:1990` |
| `fn configure_no_window(cmd: &mut Command)` | private | `acp.rs:1997` |

#### Notifications and agent-initiated requests handled inbound

Dispatched identically in both read loops (`acp.rs:1136-1163` and `acp.rs:1481-1510`):

| Inbound method | Handling | Line |
|---|---|---|
| `session/update` | `handle_session_update`, returns "tool call started" flag | `acp.rs:1138-1140` / `acp.rs:1483-1490` |
| `_goose/unstable/session/update` | `handle_goose_usage_update` (usage tracking) | `acp.rs:1141-1143` / `acp.rs:1491-1493` |
| `session/request_permission` | auto-answered by `handle_permission_request` | `acp.rs:1144-1146` / `acp.rs:1494-1496` |
| anything else **with** an `id` | replied `-32601 Method not found` | `acp.rs:1147-1156` / `acp.rs:1497-1506` |
| anything else **without** an `id` | debug-logged, dropped | `acp.rs:1160` / `acp.rs:1507` |

`session/update` sub-kinds recognised by `handle_session_update` (discriminator is `sessionUpdate`, `acp.rs:1529-1532`): `agent_message_chunk` (`:1535`), `tool_call` (`:1541`, returns `true`), `tool_call_update` (`:1553`), `plan` (`:1563`), `agent_thought_chunk` (`:1567`), `available_commands_update` (`:1573`), `session_info_update` (`:1590`), `keepalive` (`:1621`), anything else (`:1622-1625`).

#### Clap CLI surface — `CliArgs` (`config.rs:234-474`)

| Flag | Env | Default | Line |
|---|---|---|---|
| `--relay-url` | `BUZZ_RELAY_URL` | `ws://localhost:3000` | `config.rs:240` |
| `--private-key` | `BUZZ_PRIVATE_KEY` | *(required)* | `config.rs:243` |
| `--agent-owner` | `BUZZ_ACP_AGENT_OWNER` | — | `config.rs:247` |
| `--agent-command` | `BUZZ_ACP_AGENT_COMMAND` | `goose` | `config.rs:250` |
| `--agent-args` | `BUZZ_ACP_AGENT_ARGS` | `acp` (comma-delimited) | `config.rs:253-258` |
| `--mcp-command` | `BUZZ_ACP_MCP_COMMAND` | `""` | `config.rs:261` |
| `--idle-timeout` | `BUZZ_ACP_IDLE_TIMEOUT` | none → `900` | `config.rs:266` |
| `--max-turn-duration` | `BUZZ_ACP_MAX_TURN_DURATION` | `7200` | `config.rs:270` |
| `--turn-timeout` | `BUZZ_ACP_TURN_TIMEOUT` | — | `config.rs:274`, **`hide = true`**, deprecated |
| `--system-prompt` | `BUZZ_ACP_SYSTEM_PROMPT` | — | `config.rs:277-281`, conflicts with `--system-prompt-file` |
| `--system-prompt-file` | `BUZZ_ACP_SYSTEM_PROMPT_FILE` | — | `config.rs:284-288` |
| `--agents` | `BUZZ_ACP_AGENTS` | `1`, range `1..=32` | `config.rs:292-293` |
| `--heartbeat-interval` | `BUZZ_ACP_HEARTBEAT_INTERVAL` | `0` | `config.rs:297` |
| `--turn-liveness-secs` | `BUZZ_ACP_TURN_LIVENESS_SECS` | `10` | `config.rs:302` |
| `--heartbeat-prompt` | `BUZZ_ACP_HEARTBEAT_PROMPT` | — | `config.rs:306-310` |
| `--heartbeat-prompt-file` | `BUZZ_ACP_HEARTBEAT_PROMPT_FILE` | — | `config.rs:314-318` |
| `--initial-message` | `BUZZ_ACP_INITIAL_MESSAGE` | — | `config.rs:321` |
| `--subscribe` | `BUZZ_ACP_SUBSCRIBE` | `mentions` | `config.rs:324-329` |
| `--kinds` | `BUZZ_ACP_KINDS` | — (comma) | `config.rs:332` |
| `--channels` | `BUZZ_ACP_CHANNELS` | — (comma) | `config.rs:335` |
| `--no-mention-filter` | `BUZZ_ACP_NO_MENTION_FILTER` | `false` | `config.rs:338` |
| `--config` | `BUZZ_ACP_CONFIG` | `./buzz-acp.toml` | `config.rs:341` |
| `--dedup` | `BUZZ_ACP_DEDUP` | `queue` | `config.rs:344` |
| `--multiple-event-handling` | `BUZZ_ACP_MULTIPLE_EVENT_HANDLING` | `steer` | `config.rs:353-358` |
| `--no-ignore-self` | `BUZZ_ACP_NO_IGNORE_SELF` | `false` | `config.rs:361` |
| `--context-message-limit` | `BUZZ_ACP_CONTEXT_MESSAGE_LIMIT` | `12`, range `0..=100` | `config.rs:366-367` |
| `--max-turns-per-session` | `BUZZ_ACP_MAX_TURNS_PER_SESSION` | `0` | `config.rs:372-373` |
| `--no-presence` | `BUZZ_ACP_NO_PRESENCE` | `false` | `config.rs:377` |
| `--no-typing` | `BUZZ_ACP_NO_TYPING` | `false` | `config.rs:381` |
| `--memory` | `BUZZ_ACP_MEMORY` | `true` | `config.rs:393-398`, conflicts with `no_memory` |
| `--no-memory` | `BUZZ_ACP_NO_MEMORY` | `false` | `config.rs:404` |
| `--no-base-prompt` | `BUZZ_ACP_NO_BASE_PROMPT` | `false` | `config.rs:409` |
| `--base-prompt-file` | `BUZZ_ACP_BASE_PROMPT_FILE` | — | `config.rs:414-418`, conflicts with `no_base_prompt` |
| `--model` | `BUZZ_ACP_MODEL` | — | `config.rs:423` |
| `--permission-mode` | `BUZZ_ACP_PERMISSION_MODE` | `bypass-permissions` | `config.rs:432-437` |
| `--respond-to` | `BUZZ_ACP_RESPOND_TO` | `owner-only` | `config.rs:442-447` |
| `--respond-to-allowlist` | `BUZZ_ACP_RESPOND_TO_ALLOWLIST` | — (comma) | `config.rs:452` |
| `--allowed-respond-to` | `BUZZ_ACP_ALLOWED_RESPOND_TO` | — (comma) | `config.rs:460` |
| `--team-instructions` | `BUZZ_ACP_TEAM_INSTRUCTIONS` | — | `config.rs:464` |
| `--relay-observer` | `BUZZ_ACP_RELAY_OBSERVER` | `false` | `config.rs:468` |
| `--lazy-pool` | `BUZZ_ACP_LAZY_POOL` | `false` | `config.rs:472` |

43 `env = "…"` attributes across the file. `--turn-timeout` is the only hidden flag.

#### Auxiliary clap parsers

Three standalone `Parser` structs, each dispatched by manual argv sniffing in `lib.rs` rather than a clap subcommand:

| Struct | Fields | Line | Dispatched at |
|---|---|---|---|
| `AuthAgentArgs` | `--agent-command` (`BUZZ_ACP_AGENT_COMMAND`, default `goose`), `--agent-args` (`BUZZ_ACP_AGENT_ARGS`, default `acp`) | `config.rs:188-203` | flattened into all three below |
| `ModelsArgs` | `#[command(flatten)] agent`, `--json` | `config.rs:171-186` | `lib.rs:1252-1253` |
| `AuthMethodsArgs` | `agent`, `--json` | `config.rs:205-216` | `lib.rs:1262-1263` |
| `AuthenticateArgs` | `agent`, `--method-id` (required, no env) | `config.rs:218-232` | `lib.rs:1272-1273` |

`ModelsArgs`' doc comment (`config.rs:165-169`) states the `models` path deliberately bypasses `Config::from_cli()` — no relay, no private key.

#### `Config` and free-function surface in `config.rs`

| Signature | Line |
|---|---|
| `pub fn Config::from_cli() -> Result<Self, ConfigError>` | `config.rs:729` |
| `pub fn Config::from_args(mut args: CliArgs) -> Result<Self, ConfigError>` | `config.rs:740` |
| `pub fn Config::summary(&self) -> String` | `config.rs:1012` |
| `pub fn propagate_legacy_env_vars()` | `config.rs:715` |
| `pub fn load_rules(path: &Path) -> Result<Vec<SubscriptionRule>, ConfigError>` | `config.rs:1060` |
| `pub fn resolve_channel_filters(config, discovered_channels: &[Uuid], rules) -> HashMap<Uuid, ChannelFilter>` | `config.rs:1134-1138` |
| `pub fn resolve_dynamic_channel_filter(config, channel_id: Uuid, rules) -> Option<ChannelFilter>` | `config.rs:1236-1240` |
| `pub fn codex_network_env(agent_command: &str, relay_url: &str) -> Option<(String, String)>` | `config.rs:646` |
| `pub fn normalize_agent_args(command: &str, agent_args: Vec<String>) -> Vec<String>` | `config.rs:679` |
| `pub(crate) fn normalize_agent_command_identity(command: &str) -> String` | `config.rs:600` |
| `pub fn PermissionMode::as_wire_str(&self) -> &'static str` / `is_default(&self) -> bool` | `config.rs:144`, `config.rs:156` |
| private: `validate_allowlist` (`:558`), `validate_multiple_event_handling` (`:579`), `default_agent_args` (`:617`), `rule_applies_to_channel` (`:1316`) | |

#### `setup_mode` surface (all `pub(crate)`)

| Item | Line |
|---|---|
| `const SETUP_PAYLOAD_ENV_VAR: &str = "BUZZ_ACP_SETUP_PAYLOAD"` | `setup_mode.rs:83` |
| `SetupPayload::from_env() -> Result<Option<Self>>` | `setup_mode.rs:213` |
| `SetupPayload::from_raw_env_value(raw: Option<String>) -> Result<Option<Self>>` | `setup_mode.rs:226` |
| `async fn run_setup_listener(config: Config, payload: SetupPayload) -> Result<()>` | `setup_mode.rs:309` |
| `#[must_use] fn should_nudge_for_event(event_id, author_allowed, filter_matched, nudged_event_ids) -> bool` | `setup_mode.rs:494-500` |
| private `SetupPayload::nudge_body(&self) -> String` | `setup_mode.rs:243` |
| private `RequirementPayload::instruction(&self) -> String` | `setup_mode.rs:122` |
| private `build_setup_subscription_rules(config) -> Vec<SubscriptionRule>` | `setup_mode.rs:521` |
| private `mentions_rule(kinds: Vec<u32>) -> SubscriptionRule` | `setup_mode.rs:545` |
| private `async fn handle_setup_membership(relay, buzz_event, config, rules, _initial_channel_ids)` | `setup_mode.rs:563-569` |
| private `async fn publish_setup_nudge(publisher, keys, channel_id, triggering_event, payload) -> Result<()>` | `setup_mode.rs:595-601` |

`handle_setup_membership`'s fifth parameter `_initial_channel_ids` is accepted and never used (`setup_mode.rs:568`).
