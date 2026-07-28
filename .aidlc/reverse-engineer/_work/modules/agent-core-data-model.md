## Module: buzz-agent — core loop, ACP wire types & handoff (`crates/buzz-agent/src`)
### Aspect: Data Model
All state in this group is in-memory and per-process. `grep -c 'sqlite'`, `grep -c 'fs::write'` and `grep -c 'File::create'` over `lib.rs agent.rs types.rs wire.rs handoff.rs main.rs` each return 0 for every file — this group has **no persistence whatsoever**. Session state dies with the process.

#### Process-level state: `App`
`App` (`lib.rs:52-62`) is the single `Arc`-shared root, constructed once in `async_main` (`lib.rs:164-168`).

| Field | Type | Line | Notes |
|---|---|---|---|
| `cfg` | `Config` | `lib.rs:53` | Immutable after `Config::from_env()` (`lib.rs:160`); no interior mutability |
| `llm` | `Arc<Llm>` | `lib.rs:54` | Shared HTTP client, one per process (`lib.rs:161`) |
| `sessions` | `Mutex<HashMap<String, Session>>` | `lib.rs:55` | Keyed by session id; `tokio::sync::Mutex` |
| `models_cache` | `tokio::sync::OnceCell<Vec<ModelEntry>>` | `lib.rs:61` | Populated only on *successful* Databricks discovery via `get_or_try_init` (`lib.rs:319`) so an error leaves the cell empty and the next `session/new` retries |

#### Session record
`Session` (`lib.rs:64-103`) — 17 fields, inserted at `lib.rs:410-429`:

| Field | Type | Line | Invariant / lifetime |
|---|---|---|---|
| `id` | `String` | `lib.rs:65` | Duplicates the `HashMap` key (both set from `session_id` at `lib.rs:411-412`); read back at `lib.rs:809` |
| `mcp` | `Arc<McpRegistry>` | `lib.rs:66` | Spawned once per session (`lib.rs:390`); cloned into each run (`lib.rs:810`) |
| `skills` | `Vec<SkillEntry>` | `lib.rs:68` | Read-only after creation; cloned per prompt (`lib.rs:792`) |
| `history` | `Vec<HistoryItem>` | `lib.rs:69` | Moved out with `std::mem::take` while a prompt runs (`lib.rs:807`) and written back at `lib.rs:709` — the map holds an **empty** history for the duration of the turn |
| `cancel_tx` | `watch::Sender<bool>` | `lib.rs:70` | Replaced with a fresh channel on each `acquire_session` (`lib.rs:789-790`) so a stale cancel can't kill the next turn |
| `busy` | `bool` | `lib.rs:71` | Single-prompt-per-session mutex; set at `lib.rs:788`, cleared at `lib.rs:705` |
| `active_run_id` | `Option<String>` | `lib.rs:76` | `Some` only while a turn is in flight (`lib.rs:800`, cleared `lib.rs:707`); gates steer acceptance |
| `steer_tx` | `Option<mpsc::UnboundedSender<Vec<ContentBlock>>>` | `lib.rs:81` | Created per turn (`lib.rs:802-803`), cleared at `lib.rs:708`; **unbounded** |
| `original_task` | `Option<String>` | `lib.rs:82` | First prompt text of the session, set once (`agent.rs:78-80`); `take()`n into the run (`lib.rs:812`) and restored at `lib.rs:710` |
| `handoff_count` | `usize` | `lib.rs:83` | Monotonic per session; compared to `cfg.max_handoffs` (`handoff.rs:34`) |
| `last_request_input_tokens` | `Option<u64>` | `lib.rs:88` | Cache-summed provider input tokens of the most recent request; cleared on handoff (`agent.rs:105`) |
| `last_request_history_bytes` | `Option<usize>` | `lib.rs:91` | Paired baseline for the above; cleared in lockstep (`agent.rs:106`) |
| `effective_system_prompt` | `Arc<str>` | `lib.rs:92` | Immutable for the session's life; composed at `lib.rs:361-387` |
| `effective_model` | `Option<String>` | `lib.rs:96` | Per-session override from `session/set_model` (`lib.rs:527`); survives across prompts |
| `accumulated_input_tokens` | `u64` | `lib.rs:100` | Session-cumulative, `saturating_add` at `lib.rs:719-721` |
| `accumulated_output_tokens` | `u64` | `lib.rs:102` | Same, `lib.rs:722-724` |

#### Session state machine
Created → idle (`busy=false, active_run_id=None, steer_tx=None`, `lib.rs:415-417`) → running (`acquire_session`: `busy=true`, fresh `cancel_tx`, fresh `run_id`, fresh steer channel, history moved out — `lib.rs:785-819`) → idle again (`run_prompt` restores `busy/active_run_id/steer_tx/history/original_task/handoff_count/last_request_*`, `lib.rs:704-712`).

There is **no terminal state**: `grep -n 'sessions.remove'` over `lib.rs` returns zero matches, so a session (and its MCP child processes) lives until process exit. Shutdown only broadcasts cancel to every session (`lib.rs:178-180`).

#### Borrowed per-turn state: `RunCtx`
`RunCtx<'a>` (`agent.rs:24-64`) owns nothing — every field is a shared or mutable borrow of locals in `run_prompt` (`lib.rs:674-693`). Mutable borrows: `cancel`, `steer`, `history`, `original_task`, `handoff_count`, `last_request_input_tokens`, `last_request_history_bytes`, `turn_input_tokens`, `turn_output_tokens`. `effective_model: &'a str` resolves session override over `cfg.model` at `lib.rs:671-673`. `turn_input_tokens`/`turn_output_tokens` are reset to `None` at turn start (`agent.rs:86-87`).

#### Conversation model
`HistoryItem` (`types.rs:60-67`) is a three-variant enum — `User(String)`, `Assistant { text, tool_calls }`, `ToolResult(ToolResult)`. Nothing in the type system enforces role alternation or that every `Assistant.tool_calls` entry has a matching `ToolResult`; that pairing is maintained procedurally by `append_results` pushing one result per call in original order (`agent.rs:350-368`).

| Type | Fields | Line |
|---|---|---|
| `ToolCall` | `provider_id: String`, `name: String`, `arguments: Value` | `types.rs:107-111` |
| `ToolResult` | `provider_id: String`, `content: Vec<ToolResultContent>`, `is_error: bool` | `types.rs:114-118` |
| `ToolResultContent` | `Text(String)` \| `Image { data, mime_type }` | `types.rs:17-20` |
| `ToolDef` | `name`, `description`, `input_schema: Value` | `types.rs:167-171` |
| `LlmResponse` | `text`, `tool_calls`, `stop: ProviderStop`, `input_tokens: Option<u64>`, `output_tokens: Option<u64>`, `reasoning: String` | `types.rs:131-156` |

`provider_id` is a non-`Option` `String` on both `ToolCall` and `ToolResult` — the type-level encoding of `tool_use ↔ tool_result` pairing the crate README claims (`README.md:224`). Synthetic results always copy the originating id (`agent.rs:703-709`), and the built-in `load_skill` result is re-stamped with it (`agent.rs:325`).

#### Two independent size measures (a deliberate invariant)
Every history item can be measured two ways, dispatched through one `size_with` (`types.rs:83-104`):

| Measure | Text | Image | Consumer |
|---|---|---|---|
| `estimated_bytes` (`types.rs:29-36`, `types.rs:70-72`) | `s.len()` | `data.len() + mime_type.len()` | `truncate_history` request-body sizing (`agent.rs:712`) |
| `context_pressure_bytes` (`types.rs:42-47`, `types.rs:79-81`) | `s.len()` | flat `IMAGE_CONTEXT_TOKEN_EQUIV + mime_type.len()` | handoff gate (`handoff.rs:116-120`, `handoff.rs:144-149`, `handoff.rs:328-335`) and the usage baseline (`agent.rs:163-168`) |

`IMAGE_CONTEXT_TOKEN_EQUIV = 16 * 1024` (`types.rs:14`), justified in the comment at `types.rs:5-13` as a ceiling over the ~2K visual tokens providers actually bill. This split is the most heavily unit-tested invariant in the group: `image_estimated_bytes_is_real_wire_size` (`types.rs:292`), `image_context_pressure_is_token_equivalent_not_base64_len` (`types.rs:303`), `single_image_does_not_trip_default_handoff_threshold` (`types.rs:326`, asserts against the shipped 167_232 threshold), `text_content_size_is_identical_for_both_measures` (`types.rs:346`).

#### Wire-facing (deserialized) types
All in `wire.rs`, all `#[serde(rename_all = "camelCase")]` except `InitializeParams` (explicit renames):

| Type | Fields | Line |
|---|---|---|
| `InitializeParams` | `protocol_version: u32`, `_client_capabilities: Value` (`#[serde(default)]`) | `wire.rs:39-44` |
| `SessionNewParams` | `cwd: String`, `mcp_servers: Vec<McpServerStdio>` (default), `system_prompt: Option<String>` (default) | `wire.rs:48-54` |
| `SessionPromptParams` | `session_id`, `prompt: Vec<ContentBlock>` | `wire.rs:58-61` |
| `SessionCancelParams` | `session_id` | `wire.rs:65-67` |
| `SessionSteerParams` | `session_id`, `prompt: Vec<ContentBlock>` (default), `expected_run_id: String` | `wire.rs:76-80` |
| `SessionSetModelParams` | `session_id`, `model_id` | `wire.rs:88-91` |
| `McpServerStdio` | `name`, `command`, `args: Vec<String>` (default), `env: Vec<EnvVar>` (default) | `types.rs:195-202` |
| `EnvVar` | `name`, `value` | `types.rs:205-208` |
| `ContentBlock` | `Text { text }` \| `ResourceLink { uri }` \| `Unsupported` (`#[serde(other)]`) | `types.rs:210-221` |

`ContentBlock::Unsupported` is a catch-all: an unknown `type` deserializes successfully and only fails later, in `prompt_to_text` (`agent.rs:623-628`). Note the asymmetry — a bad block in the *initial* prompt aborts the turn, but the same block in a steer is dropped with a `warn!` (`agent.rs:275-278`). `SessionNewParams` field-level deserialization is the only serde behavior with dedicated unit tests: `session_new_params_deserializes_system_prompt` (`wire.rs:244`), `..._defaults_to_none` (`wire.rs:259`), `..._ignores_unknown_fields` (`wire.rs:270`), `..._empty_string_system_prompt` (`wire.rs:283`). `SessionSteerParams` and `SessionSetModelParams` have no deserialization unit tests.

#### Control / status enums
| Enum | Variants | Line |
|---|---|---|
| `ProviderStop` | `EndTurn, ToolUse, MaxTokens, Refusal, Other` | `types.rs:158-164` |
| `StopReason` | `EndTurn, Cancelled, MaxTokens, MaxTurnRequests, Refusal` (+ `as_wire`, `types.rs:183-191`) | `types.rs:174-180` |
| `AgentError` | `InvalidParams, Llm, LlmAuth, LlmModelNotFound, Mcp, Cancelled` | `types.rs:224-231` |
| `Inbound` | `Request{id,method,params}`, `Notification{method,params}`, `Ignored`, `Invalid{id,code,message}` | `wire.rs:20-37` |
| `WireMsg` | `Notify(Value)` — single variant | `wire.rs:13-15` |
| `HandoffOutcome` | `Performed, Skipped, Cancelled` | `handoff.rs:19-23` |
| `HandoffTokenCounts` | `{ before: u64, after: u64 }`, `Display` only (`handoff.rs:11-17`) — used solely for the log line at `handoff.rs:96-100` | `handoff.rs:7-9` |
| `InvokeOutcome` | `Done(ToolResult)`, `Failed(String)` | `agent.rs:492-495` |

`WireMsg` having exactly one variant means the writer's `let WireMsg::Notify(v) = msg;` (`wire.rs:223`) is infallible — the enum is a forward-compat placeholder, not a discriminant in use.

#### Derived/ephemeral data built inside a turn
- Tool result slots: `Vec<Option<ToolResult>>` sized to `calls.len()` (`agent.rs:296`) plus a `runnable: Vec<usize>` index list (`agent.rs:297`) — index correspondence with `calls` is the invariant that keeps ordering correct across the `JoinSet` (`agent.rs:373`, results written by index at `agent.rs:431`).
- Handoff prompt: a `Vec<String>` of newest-first snippets, budget-checked and then `reverse()`d back to oldest-first (`handoff.rs:200-224`).
- Synthetic hook exchange: an `Assistant{text:"", tool_calls:[…]}` + `ToolResult` pair per objection, id `buzz_hook_{hook}_{server}_{ordinal}` from a process-wide `AtomicU64` (`agent.rs:654-657`, `agent.rs:697-701`).

#### Identifier formats
| Id | Format | Line |
|---|---|---|
| Session | `ses_` + 8 random bytes hex (16 chars) | `lib.rs:394-395`, `lib.rs:821-825` |
| Run | `run_` + same token, or `run_x` if RNG fails | `lib.rs:799` |
| Steer message | `steer_` + same token, or `steer_x` if RNG fails | `lib.rs:577` |
| Synthetic hook call | `buzz_hook_{hook}_{server}_{counter}` | `agent.rs:654-657` |
