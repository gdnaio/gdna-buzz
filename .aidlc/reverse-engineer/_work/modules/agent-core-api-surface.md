## Module: buzz-agent — core loop, ACP wire types & handoff (`crates/buzz-agent/src`)
### Aspect: API Surface
The real API of this group is a JSON-RPC 2.0 line protocol over stdin/stdout. The Rust surface is deliberately thin: one `pub fn run()` plus re-exports.

#### Rust public items originating in this group
| Item | Kind | Line | Doc comment? |
|---|---|---|---|
| `run()` | `pub fn` | `lib.rs:110` | No |
| `WINDOWS_SHELL_RESOLUTION_ENV` | `pub const &[&str]`, `#[cfg(windows)]` | `lib.rs:22-30` | Yes (`lib.rs:18-20`) |
| `pub mod auth`, `pub mod catalog`, `pub mod config`, `pub mod types` | module re-export | `lib.rs:4-12` | No |
| `pub use catalog::{discover_databricks_models, ModelEntry, DATABRICKS_V2_KNOWN_MODELS}` | re-export | `lib.rs:15` | n/a |
| `pub use config::Provider` | re-export | `lib.rs:16` | n/a |
| `pub use types::AgentError` | re-export | `lib.rs:17` | n/a |

`mod agent`, `mod handoff`, `mod wire`, `mod builtin`, `mod hints`, `mod llm`, `mod mcp` are private (`lib.rs:2-13`). Everything marked `pub` inside `agent.rs` and `wire.rs` is therefore **pub-in-private** — visible crate-wide but not part of the crate's external API: `RunCtx` and its 17 `pub` fields (`agent.rs:24-64`), `RunCtx::run` (`agent.rs:66`), the four JSON-RPC error constants (`wire.rs:8-11`), `WireMsg`/`WireSender`/`Inbound` (`wire.rs:13-37`), all six params structs, and `classify`/`ok`/`err`/`session_update`/`goose_session_update`/`session_update_with_goose_meta`/`send`/`read_bounded_line`/`writer_task` (`wire.rs:93-237`).

Because `pub mod types` is public (`lib.rs:11`), the entire contents of `types.rs` **are** external API: `ToolResultContent`, `HistoryItem`, `ToolCall`, `ToolResult`, `LlmResponse`, `ProviderStop`, `ToolDef`, `StopReason`, `McpServerStdio`, `EnvVar`, `ContentBlock`, `AgentError`, and the free function `clamp` (`types.rs:259`). No external consumer uses them: the only cross-crate consumers of `buzz-agent` are `sprig` (calls `buzz_agent::run()`, `crates/sprig/src/main.rs:18`) and the desktop Tauri backend (uses `WINDOWS_SHELL_RESOLUTION_ENV`, `desktop/src-tauri/src/managed_agents/git_bash.rs:136`).

#### Undocumented public items (AGENTS.md violation)
AGENTS.md:114 states "New public API must have doc comments". Undocumented public items in this group, by file:
- `types.rs`: `ToolResultContent` (`:17`), `as_text_lossy` (`:49`), `HistoryItem` (`:60`), `estimated_bytes` (`:70`), `ToolCall` (`:107`), `ToolResult` (`:114`), `text` (`:121`), `LlmResponse` (struct itself, `:131` — its fields are documented), `ProviderStop` (`:158`), `ToolDef` (`:167`), `StopReason` (`:174`), `as_wire` (`:183`), `McpServerStdio` (`:195`), `EnvVar` (`:205`), `ContentBlock` (`:212`), `AgentError` (`:224`), `json_rpc_code` (`:249`), `clamp` (`:259`). That is 18 of 20 public items in the crate's one public data module.
- `lib.rs`: `run()` (`:110`).
- `wire.rs` (crate-internal but `pub`): `classify` (`:93`), `ok` (`:123`), `err` (`:127`), `session_update` (`:131`), `send` (`:170`), `read_bounded_line` (`:174`), `writer_task` (`:220`), plus the four error constants.

#### Inbound JSON-RPC requests handled
Dispatch is a flat `match` on the method string (`lib.rs:224-265`). There is no state machine: `initialize` is **not** required before `session/new` or `session/prompt` — nothing records that it happened (`negotiated_version` is computed and dropped, `lib.rs:284`).

| Method | Handler | Concurrency | Success result | Error codes |
|---|---|---|---|---|
| `initialize` | `initialize` (`lib.rs:273`) | inline | `{protocolVersion, agentCapabilities{loadSession:false, promptCapabilities{image:false,audio:false,embeddedContext:false}, mcpCapabilities{http:false,sse:false}}, agentInfo{name:"buzz-agent",version:CARGO_PKG_VERSION}}` (`lib.rs:288-297`) | `-32602` on bad params |
| `session/new` | `session_new` (`lib.rs:329`), `tokio::spawn`ed (`lib.rs:227-231`) | concurrent | `{sessionId, models:{currentModelId, availableModels:[{modelId,name}]}}` (`lib.rs:464-474`) | `-32602` (bad params / relative `cwd` / max sessions / oversize system prompt), MCP spawn error code via `e.json_rpc_code()` (`lib.rs:392`), `-32000` on RNG failure (`lib.rs:396`) |
| `session/prompt` | `run_prompt` (`lib.rs:627`), spawned (`lib.rs:623-625`) | one per session | `{stopReason}` (`lib.rs:753-758`) | `-32602` (bad params, unknown session, "prompt already in flight"), else `AgentError::json_rpc_code()` |
| `session/set_model` | `set_model_session` (`lib.rs:503`) | inline | `{sessionId, modelId}` (`lib.rs:535-540`) | `-32602` (bad params, empty `modelId`, unknown session) |
| `session/cancel` | `cancel_session` (`lib.rs:487`) | inline | `null` (`lib.rs:239`) | none — unknown session is silently accepted (`lib.rs:487-494`) |
| `_goose/unstable/session/steer` | `steer_session` (`lib.rs:554`) | inline | `{runId, messageId}` (`lib.rs:607-610`) | `-32602` (empty prompt, empty `expectedRunId`, unknown session, no active run, run-id mismatch) |
| anything else | — | — | — | `-32601` `jsonrpc: method not found: {method}` (`lib.rs:248-258`) |

#### Inbound notifications
Only `session/cancel` is acted on (`lib.rs:267-271`); every other notification is silently discarded. Bare responses (`id` without `method`) map to `Inbound::Ignored` and are dropped without a log line (`wire.rs:110-111`, `lib.rs:215`). Malformed frames: non-object or wrong `jsonrpc` version → `-32600` (`wire.rs:94-101`); missing both method and id → `-32600` (`wire.rs:113-117`); unparseable JSON → `-32700` with id `null` (`lib.rs:196-203`).

#### Outbound notifications emitted
| Method | `update.sessionUpdate` | Emitted at | Payload |
|---|---|---|---|
| `session/update` | `session_info_update` + `_meta.goose.activeRunId` | `lib.rs:661-670` | run id, at prompt start only |
| `session/update` | `session_info_update` + `_meta.goose.queuedSteer` | `lib.rs:612-620` | `{messageId, runId}` after an accepted steer |
| `session/update` | `keepalive` | `agent.rs:134-144` | empty; every 30 s while awaiting the provider |
| `session/update` | `agent_thought_chunk` | `agent.rs:179-192` | `content.text` = provider reasoning, only when non-empty |
| `session/update` | `agent_message_chunk` | `agent.rs:194-206` | `content.text` = assistant text, only when non-empty |
| `session/update` | `tool_call` (`status:"pending"`) | `agent.rs:552-568` | `toolCallId`, `title`=tool name, `kind:"other"`, `rawInput`=arguments |
| `session/update` | `tool_call_update` (`in_progress`) | `agent.rs:570-583` | `toolCallId`, status |
| `session/update` | `tool_call_update` (`completed`) | `agent.rs:585-600` | `content[]` text + `rawOutput.isError` |
| `session/update` | `tool_call_update` (`failed`) | `agent.rs:602-616` | `rawOutput.error` |
| `_goose/unstable/session/update` | `usage_update` | `lib.rs:730-750` | `used`, `contextLimit:0`, `accumulatedInputTokens`, `accumulatedOutputTokens`, `model` |

The `_meta` envelope is nested **inside** `update`, not beside it (`wire.rs:157-169`) — deliberately matching goose's `SessionInfoUpdate` layout. `usage_update` uses a separate top-level method (`wire.rs:143-149`), not `session/update`.

Ordering guarantee: the `usage_update` notification is always sent before the `session/prompt` response (`lib.rs:714-752` precedes `lib.rs:753-759`), because buzz-acp's `UsageTracker` must see it while the turn is still open. Locked down by `usage_notification_emitted_before_prompt_response` (`tests/fake_llm.rs:801`), `no_usage_turn_emits_no_usage_notification` (`tests/fake_llm.rs:888`), and `cancelled_turn_with_usage_emits_notification_before_response` (`tests/fake_llm.rs:926`).

#### Stop reasons returned
`StopReason::as_wire` (`types.rs:183-191`) emits `end_turn`, `cancelled`, `max_tokens`, `max_turn_requests`, `refusal`. `map_stop` (`agent.rs:740-746`) collapses `ProviderStop::{EndTurn,ToolUse,Other}` → `end_turn`. All five strings are accepted by the client side (`crates/buzz-acp/src/acp.rs:66-71`, test `stop_reason_parses_all_known_values` at `acp.rs:2012`).

#### Error-code mapping
| Code | Meaning | Source |
|---|---|---|
| `-32700` | parse error | `wire.rs:8`, used `lib.rs:200` |
| `-32600` | invalid request | `wire.rs:9`, used `wire.rs:98`, `wire.rs:115` |
| `-32601` | method not found | `wire.rs:10`, used `lib.rs:252` |
| `-32602` | invalid params | `wire.rs:11`; `AgentError::InvalidParams` (`types.rs:251`) |
| `-32001` | LLM auth failure | `AgentError::LlmAuth` (`types.rs:252`) |
| `-32002` | model not found | `AgentError::LlmModelNotFound` (`types.rs:253`) |
| `-32000` | everything else (`Llm`, `Mcp`, `Cancelled`) | `types.rs:254`; also literal at `lib.rs:396` |

#### CLI entry points
`main.rs` is 6 lines: call `buzz_agent::run()`, print `Error: {e}` to stderr and `exit(1)` on failure (`main.rs:1-6`). `run()` (`lib.rs:110-121`) inspects `argv[1]`:
- `auth` → `auth_subcommand(&args[2..])` on a fresh multi-thread runtime (`lib.rs:111-116`). Accepts `databricks`, `databricks_v2`, `databricks-v2` (`lib.rs:132`); any other value errors `auth: unknown provider {other:?}` (`lib.rs:150`); missing provider errors with a usage hint (`lib.rs:151`).
- anything else (including `--help`, `--version`, or a typo) → falls through to `async_main()` and the agent begins reading stdio (`lib.rs:117-120`). There is no `--help`/`--version` and unknown arguments are silently ignored.

Exit codes: `0` normal EOF **and** after a fatal reader error (which is only logged, `lib.rs:170-176`, then `Ok(())` at `lib.rs:121`); `1` from `main.rs:4`; `2` from `die()` on config/LLM-construction failure (`lib.rs:105-108`). When invoked through the `sprig` multicall binary, dispatch is by `argv[0]` (`crates/sprig/src/main.rs:10-19`) so `argv[1]` semantics are preserved.

#### Advertised vs actual capability
`initialize` advertises `promptCapabilities.image:false` (`lib.rs:293`) and the code enforces it — only `text` and `resource_link` blocks are accepted (`agent.rs:618-631`), covered by `test_unsupported_content_block` (`tests/golden_transcripts.rs:384`). `loadSession:false` is honored: no `session/load` handler exists. Version negotiation is `min(client, PROTOCOL_VERSION=2)` (`lib.rs:284`, `config.rs:3`), tested by `test_initialize_version_check` (`tests/golden_transcripts.rs:288`, asserts 99→2 and 1→1).
