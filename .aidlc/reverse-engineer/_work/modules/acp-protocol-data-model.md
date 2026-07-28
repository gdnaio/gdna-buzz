## Module: buzz-acp — ACP protocol, config & setup mode (`crates/buzz-acp/src`)
### Aspect: Data Model

#### JSON-RPC wire types (constructed, never modelled as structs)

There is no typed request/response envelope. Every outbound frame is built inline with `serde_json::json!` and every inbound frame is a bare `serde_json::Value`.

| Frame | Construction site | Shape |
|---|---|---|
| Request | `send_request` (`acp.rs:986-991`) | `{jsonrpc:"2.0", id:<u64>, method, params}` |
| Prompt request | `session_prompt_blocks_with_idle_timeout` (`acp.rs:696-701`) | same, built separately so `last_prompt_id` can be captured first (`acp.rs:688`) |
| Notification | `send_notification` (`acp.rs:1053-1057`) | `{jsonrpc:"2.0", method, params}` — **no `id`** (`acp.rs:1052`) |
| Steer request | read-loop steer arm (`acp.rs:1345-1350`) | `{jsonrpc:"2.0", id, method:"_goose/unstable/session/steer", params}` |
| Method-not-found reply | `acp.rs:1150-1154`, `acp.rs:1497-1501` | `{jsonrpc:"2.0", id:<echoed>, error:{code:-32601, message}}` |
| Permission reply | `permission_response_selected` (`acp.rs:1808-1814`), `permission_response_cancelled` (`acp.rs:1817-1823`) | `{jsonrpc:"2.0", id, result:{outcome:{outcome:"selected"|"cancelled", optionId?}}}` |

Params builders: `build_initialize_params` (`acp.rs:124-133`), `build_client_capabilities` (`acp.rs:347-368`), `build_prompt_params` (`acp.rs:1768-1778`), `build_steer_params` (`acp.rs:1791-1805`).

`build_initialize_params` hardcodes `protocolVersion: 2` (`acp.rs:126`), `clientInfo.name = "buzz-acp"`, `clientInfo.version = env!("CARGO_PKG_VERSION")` (`acp.rs:129-130`).

`build_client_capabilities` advertises three keys (`acp.rs:348-367`): `auth.terminal = true`, `_meta.goose.customNotifications = true`, `_meta["terminal-auth"] = true`.

#### `McpServer` / `EnvVar` — the credential carrier

```rust
// acp.rs:27-34
pub struct McpServer { pub name: String, pub command: String, pub args: Vec<String>, pub env: Vec<EnvVar> }
// acp.rs:36-40
pub struct EnvVar { pub name: String, pub value: String }
```

Both derive `Debug, Clone, serde::Serialize` (`acp.rs:27`, `acp.rs:36`). No `Deserialize`, no custom `Debug`, no `Drop`, no wrapper type. `EnvVar.value` is a plain `String` and is serialized verbatim into `params.mcpServers` by `session_new_full` (`acp.rs:568-571`). Populated with the raw bech32 secret key at `lib.rs:4159-4170`. Doc comment at `acp.rs:24-26` states all four `McpServer` fields are schema-required.

#### `StopReason` (`acp.rs:44-58`)

`Debug, Clone, PartialEq` only. Variants and their wire literals, parsed by `StopReason::from_str` (`acp.rs:66-76`) after `to_ascii_lowercase()`:

| Variant | Wire string | Line |
|---|---|---|
| `EndTurn` | `end_turn` | `acp.rs:48` |
| `Cancelled` | `cancelled` | `acp.rs:50` |
| `MaxTokens` | `max_tokens` | `acp.rs:52` |
| `MaxTurnRequests` | `max_turn_requests` | `acp.rs:54` |
| `Refusal` | `refusal` | `acp.rs:57` |

Unknown strings return `None` (`acp.rs:74`), which `parse_stop_reason` turns into `AcpError::Protocol` (`acp.rs:1762-1763`).

#### `AcpError` (`acp.rs:78-111`)

`thiserror::Error` with two `#[from]` conversions (`Io` ← `std::io::Error` at `acp.rs:81`, `Json` ← `serde_json::Error` at `acp.rs:84`).

| Variant | Payload | Line |
|---|---|---|
| `Io` | `std::io::Error` | `acp.rs:80-81` |
| `Json` | `serde_json::Error` | `acp.rs:83-84` |
| `AgentExited` | — | `acp.rs:86-87` |
| `IdleTimeout` | `Duration` | `acp.rs:89-90` |
| `HardTimeout` | `{ silence: Duration }` | `acp.rs:92-93` |
| `CancelDrainTimeout` | `Duration` | `acp.rs:95-96` |
| `Timeout` | `Duration` | `acp.rs:98-99` |
| `WriteTimeout` | `Duration` | `acp.rs:101-102` |
| `Protocol` | `String` | `acp.rs:104-105` |
| `AgentError` | `{ code: i64, message: String }` | `acp.rs:107-108` |

`agent_error_from_json` (`acp.rs:115-122`) builds `AgentError`, defaulting `code` to `-32000` when absent (`acp.rs:116`) and falling back to `error.to_string()` when `message` is missing or non-string (`acp.rs:117-120`).

#### `AcpClient` internal state (`acp.rs:139-203`)

| Field | Type | Purpose / line |
|---|---|---|
| `child` | `tokio::process::Child` | `acp.rs:141` |
| `stdin` | `ChildStdin` | `acp.rs:143` |
| `reader` | `FramedRead<ChildStdout, LinesCodec>` | `acp.rs:147` |
| `next_id` | `u64` | monotonic request id; harness ids are always numeric (`acp.rs:150`) |
| `pending_permission_id` | `Option<serde_json::Value>` | `Value` because agents may send string ids (`acp.rs:156`) |
| `permission_responded` | `bool` | double-response guard (`acp.rs:160`) |
| `last_prompt_id` | `Option<u64>` | `acp.rs:164` |
| `current_hard_deadline` | `Option<tokio::time::Instant>` | inherited by cancel drain (`acp.rs:168`) |
| `observer` | `Option<ObserverHandle>` | `acp.rs:170` |
| `observer_agent_index` | `Option<usize>` | `acp.rs:172` |
| `observer_context` | `ObserverContext` | `acp.rs:174` |
| `active_run_id` | `Option<String>` | latest `_meta.goose.activeRunId` (`acp.rs:189`) |
| `steer_rx` | `Option<mpsc::Receiver<pool::SteerRequest>>` | per-turn, taken by the read loop (`acp.rs:199`) |
| `goose_usage` | `UsageTracker` | `acp.rs:203` |

#### Session / model / capability types

- `SessionNewResponse` (`acp.rs:1828-1832`): `{ session_id: String, raw: serde_json::Value }` — `raw` is the whole `result`, no schema.
- `ModelSwitchMethod` (`acp.rs:1835-1846`): `#[serde(tag="type")]` enum, `ConfigOption { config_id, option_value }` (stable) and `SetModel { model_id }` (unstable).
- Model catalogs stay as raw JSON: `extract_model_config_options` returns `Vec<serde_json::Value>` filtered on `category == "model"` (`acp.rs:1851-1863`); `extract_model_state` returns `result["models"]` untyped (`acp.rs:1866-1868`).
- Usage payloads are the only strongly-typed inbound data — `GooseSessionUpdateNotification` / `GooseSessionUpdateVariant` / `TurnUsage`, all defined in `usage.rs` and deserialized at `acp.rs:1656`.

#### `Config` (`config.rs:485-556`) — 40 fields

`#[derive(Debug)]` only (`config.rs:485`). Notable field/CLI divergences:

| Field | Type | Note |
|---|---|---|
| `keys` | `nostr::Keys` | parsed at `config.rs:741`; `Debug` on `Config` covers it |
| `idle_timeout_secs` / `max_turn_duration_secs` | `u64` | resolved, not raw (`config.rs:492-493`) |
| `ignore_self` | `bool` | inverted from `--no-ignore-self` (`config.rs:977`) |
| `presence_enabled` / `typing_enabled` | `bool` | inverted from `--no-presence` / `--no-typing` (`config.rs:983-984`) |
| `memory_enabled` | `bool` | `args.memory && !args.no_memory` (`config.rs:985`) |
| `respond_to_allowlist` | `HashSet<String>` | validated + lowercased (`config.rs:558-572`) |
| `allowed_respond_to` | `Vec<String>` | raw strings, not parsed enums (`config.rs:532`) |
| `persona_env_vars` | `Vec<(String, String)>` | untyped pairs (`config.rs:535`) |
| `has_generated_codex_config` | `bool` | drives the `CODEX_CONFIG` merge in `spawn` (`config.rs:541`) |
| `base_prompt_content` | `Option<String>` | file content resolved in `from_args` (`config.rs:556`) |
| `agent_owner` | `Option<String>` | trimmed + lowercased (`config.rs:1003`) |

#### Config enums

| Enum | Variants | Line |
|---|---|---|
| `ConfigError` | `KeyParse(nostr::key::Error)`, `Io(std::io::Error)`, `ConfigFile(String)` | `config.rs:38-49` |
| `SubscribeMode` | `Mentions`, `All`, `Config` | `config.rs:50-55` |
| `DedupMode` | `Drop`, `Queue` | `config.rs:57-61` |
| `MultipleEventHandling` | `Queue`, `Steer`, `Interrupt`, `OwnerInterrupt` (renamed `owner-interrupt`, `config.rs:87`) | `config.rs:64-89` |
| `RespondTo` | `OwnerOnly` (`#[default]`), `Allowlist`, `Anyone`, `Nobody` | `config.rs:94-100` |
| `PermissionMode` | `Default`, `AcceptEdits`, `BypassPermissions`, `DontAsk`, `Plan` — each with a camelCase clap alias | `config.rs:122-139` |

`RespondTo` derives `Debug, Clone, Default, PartialEq, Eq, Hash, ValueEnum` (`config.rs:93`) plus a `Display` producing kebab-case (`config.rs:103-112`). `PermissionMode::as_wire_str` (`config.rs:144-153`) maps to camelCase wire literals and `Display` delegates to it (`config.rs:161-165`).

#### `ChannelFilter` (`config.rs:477-483`)

```rust
pub struct ChannelFilter {
    pub kinds: Option<Vec<u32>>,   // None = wildcard (all kinds)
    pub require_mention: bool,      // adds a #p filter for the agent pubkey
}
```

`Debug, Clone` (`config.rs:477`). `None` is explicitly documented as wildcard (`config.rs:479`).

#### TOML rule types

`TomlConfig` (`config.rs:1053-1057`) is a private `Deserialize` struct with a single `#[serde(default)] rules: Vec<SubscriptionRule>`. `SubscriptionRule` and `ChannelScope` are defined in `filter.rs`; `config.rs` mutates two of their fields post-deserialization: `compiled_filter: Option<Arc<evalexpr Node>>` (`config.rs:1104`) and `consecutive_timeouts: Arc<AtomicU32>` (`config.rs:1128`).

#### Setup-mode payload types (`setup_mode.rs`)

- `SetupPayload` (`setup_mode.rs:197-204`): `{ agent_name: String, agent_pubkey: String, requirements: Vec<RequirementPayload> }`, `Debug, Clone, Serialize, Deserialize`.
- `RequirementPayload` (`setup_mode.rs:91-118`): `#[serde(tag="surface", rename_all="snake_case")]` — `NormalizedField { field }` (`:95`), `EnvKey { key }` (`:97`), `CliLogin { probe_args, setup_copy, availability }` (`:99-106`), `CliConfigInvalid { probe_args, setup_copy, diagnostic }` (`:110-116`), `GitBash` (`:117`).
- `AcpAvailabilityStatus` (`setup_mode.rs:58-71`): `snake_case` — `Available`, `AdapterMissing`, `AdapterOutdated`, `CliMissing`, `NotInstalled`. Doc comment at `setup_mode.rs:52-57` states this is a hand-maintained mirror of the desktop `AcpAvailabilityStatus` enum and `api/types.ts`, deliberately duplicated because buzz-acp must not depend on desktop types.

All three are `pub(crate)` — the payload schema is not part of the crate's public API.
