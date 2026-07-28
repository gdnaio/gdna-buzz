## Module: buzz-agent — LLM providers & configuration (`crates/buzz-agent/src`)
### Aspect: Data Model
This group owns two shapes: the resolved configuration struct (`config.rs`) and the on-the-wire request/response representations for four provider dialects (`llm.rs`). Nothing here is a serde-derived request type — every provider body is hand-built `serde_json::Value`.

#### Resolved configuration struct
`Config` (`config.rs:687-734`) is a flat 26-field struct deriving only `Debug, Clone` (`config.rs:686`). It carries no `Serialize`/`Deserialize` — grep for `derive(Serialize` / `derive(Deserialize` in `config.rs` returned zero matches, so a `Config` cannot be accidentally serialized into a payload.

| Field | Type | Notes |
|---|---|---|
| `provider` | `Provider` | `config.rs:688` |
| `system_prompt` | `String` | `config.rs:689` |
| `max_rounds` | `u32` | `config.rs:690`; 0 = unlimited |
| `max_output_tokens` | `u32` | `config.rs:691`; feeds `max_tokens`/`max_completion_tokens`/`max_output_tokens` in every body |
| `llm_timeout` | `Duration` | `config.rs:692`; becomes `reqwest` **read** timeout only (`llm.rs:56`) |
| `tool_timeout`, `mcp_init_timeout`, `mcp_max_restart_attempts`, `mcp_restart_base_ms`, `mcp_restart_max_ms` | `Duration`/`u32`/`u64` | `config.rs:693-697`; not read by `llm.rs` |
| `max_sessions`, `max_line_bytes`, `max_history_bytes` | `usize` | `config.rs:698-700` |
| `max_tool_result_text_bytes` | `usize` | `config.rs:705` |
| `max_context_tokens` | `u64` | `config.rs:715` — deliberately `u64` so it can be compared to provider-reported `input_tokens: Option<u64>` |
| `max_handoffs`, `max_parallel_tools`, `hook_timeout`, `stop_max_rejections` | `usize`/`Duration`/`u32` | `config.rs:716-724` |
| `hook_servers` | `HookServers` | `config.rs:727` |
| `api_key` | `String` | `config.rs:723` — plaintext `String`, no wrapper type (see Security) |
| `model`, `base_url`, `anthropic_api_version` | `String` | `config.rs:724-726` |
| `openai_api` | `OpenAiApi` | `config.rs:729` |
| `hints_enabled` | `bool` | `config.rs:730` |
| `thinking_effort` | `Option<ThinkingEffort>` | `config.rs:733`; `None` means "send no thinking config at all" |

#### Config invariants and where they are (not) enforced
`Config::validate` (`config.rs:870-951`) is the only invariant gate, and it is called from exactly one place: `from_env` at `config.rs:828`.

| Invariant | Site |
|---|---|
| `max_output_tokens >= 1` | `config.rs:876-878` |
| `max_context_tokens > max_output_tokens` | `config.rs:879-884` |
| `max_history_bytes >= 4096` | `config.rs:885-889` |
| `max_history_bytes >= MAX_PROMPT_BYTES` (1 MiB, `config.rs:638`) | `config.rs:890-895` |
| `max_line_bytes >= 1024` | `config.rs:896-900` |
| `1024 <= max_tool_result_text_bytes <= MAX_TOOL_RESULT_BYTES` (8 MiB, `config.rs:643`) | `config.rs:901-907` |
| `llm_timeout`, `tool_timeout`, `mcp_init_timeout` each `>= 1s` | `config.rs:908-916` |
| `max_parallel_tools >= 1`, `mcp_max_restart_attempts >= 1`, `mcp_restart_base_ms >= 1`, `mcp_restart_max_ms >= mcp_restart_base_ms` | `config.rs:917-931` |
| `thinking_effort != none|minimal` when `provider == Anthropic` | `config.rs:939-949` |

`Config::for_discovery` (`config.rs:838-868`) constructs a `Config` that **would fail `validate()`**: it sets `max_output_tokens: 1` (`config.rs:846`), `mcp_max_restart_attempts: 0` (`config.rs:851`), `mcp_restart_base_ms: 0` (`config.rs:852`) — all rejected by `config.rs:920-925` — and never calls `validate()`. The odd `max_context_tokens: 200_001` (`config.rs:857`) exists only to satisfy the `>` comparison at `config.rs:879` if validation were ever added. There is no type-level distinction between a validated and an unvalidated `Config`, so the invariants at `config.rs:876-949` are not struct invariants, only `from_env` post-conditions.

#### Enumerations
| Enum | Variants | Derives | Site |
|---|---|---|---|
| `Provider` | `Anthropic`, `OpenAi`, `Databricks`, `DatabricksV2` | `Debug, Clone, Copy, PartialEq` (`config.rs:661`) — no `Eq`, no `Hash` | `config.rs:662-678` |
| `OpenAiApi` | `Chat`, `Responses`, `Auto` | `Debug, Clone, Copy, PartialEq` (`config.rs:679`) | `config.rs:680-684` |
| `ThinkingEffort` | `None`, `Minimal`, `Low`, `Medium`, `High`, `XHigh`, `Max` | `Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord` (`config.rs:18`) | `config.rs:19-27` |
| `HookServers` | `None`, `All`, `Only(Vec<String>)` | `Debug, Clone` (`config.rs:1054`) | `config.rs:1055-1059` |
| `DatabricksV2Route` | `OpenAiResponses`, `AnthropicMessages`, `MlflowChatCompletions` | `Debug, Clone, Copy, PartialEq, Eq` (`llm.rs:30`) — private to `llm.rs` | `llm.rs:31-35` |

`ThinkingEffort`'s **numeric discriminants** are load-bearing, not just its `Ord`: `resolve_openai_effort` computes `(e as i32 - requested as i32).unsigned_abs()` at `config.rs:487`. That distance metric depends on the variants being 0..=6 in declaration order (`config.rs:19-27`) with no explicit discriminants. `thinking_effort_ord_ordering` (`config.rs:1387-1398`) asserts the `Ord` relation but never the cast values, so a reordering that preserved `Ord` while changing gaps would not be caught.

#### `Llm` client state
`Llm` (`llm.rs:37-51`) holds exactly three fields: a `reqwest::Client`, an `AtomicBool auto_upgraded` (a one-way sticky latch for the chat→responses upgrade, `llm.rs:41-44`), and `auth: Arc<dyn TokenSource>` (`llm.rs:50`). The `auth` field is populated even for `Provider::Anthropic`, where it is never read — documented at `llm.rs:1167-1170` as a way to keep the field non-`Option`.

`type OpenAiParse = fn(Value) -> Result<LlmResponse, AgentError>` (`llm.rs:28`) is the parser half of every `(body, parser)` pair. The name is a misnomer: the DatabricksV2 Anthropic route casts `parse_anthropic as OpenAiParse` at `llm.rs:126`.

#### Request bodies per provider (all `serde_json::Value`, no typed structs)
| Builder | Top-level keys | Site |
|---|---|---|
| `anthropic_body` | `model`, `max_tokens`, `system`, `messages`, optional `thinking`, optional `output_config`, optional `tools` | `llm.rs:391-462`; base object at `llm.rs:446-447` |
| `openai_body` (Chat Completions) | `model`, `stream: false`, `max_completion_tokens`, `messages`, optional `reasoning_effort`, optional `tools` + `tool_choice: "auto"` | `llm.rs:476-554`; base object at `llm.rs:547-548` |
| `responses_body` (Responses API) | `model`, `instructions`, `max_output_tokens`, `input`, optional `reasoning: {effort}`, optional `tools` + `tool_choice: "auto"` | `llm.rs:589-677`; base object at `llm.rs:663-668` |

Streaming vs non-streaming: only the Chat Completions shape carries an explicit `"stream": false` (`llm.rs:548`, and again in the summarizer body at `llm.rs:196`). `anthropic_body` and `responses_body` emit **no** `stream` key at all (`llm.rs:446-447`, `llm.rs:663-668`), relying on each API's non-streaming default. There is no SSE/streaming parse path in this file — grep for `text/event-stream` and `data:` frame handling in `llm.rs` returned zero matches, consistent with the "non-streaming" claim in `AGENTS.md:50` and `crates/buzz-agent/README.md:255`.

#### Message representation: `HistoryItem` → wire
`HistoryItem` (`types.rs:172-179`) has three variants. Each provider maps them differently:

| `HistoryItem` | Anthropic | OpenAI Chat | Responses |
|---|---|---|---|
| `User(text)` | `{role:"user", content:[{type:"text",text}]}` (`llm.rs:404-406`) | `{role:"user", content: text}` — bare string (`llm.rs:501`) | `{role:"user", content:[{type:"input_text",text}]}` (`llm.rs:595-598`) |
| `Assistant{text, tool_calls}` | `content[]` of `text` + `tool_use` blocks; **entire turn skipped when both empty** (`llm.rs:417-423`) | one message with `content: text` plus `tool_calls[]` (`llm.rs:503-527`) | separate `{role:"assistant", content:[{type:"output_text"}]}` (emitted only when `text` non-empty, `llm.rs:601-606`) then one `function_call` item per call |
| `ToolResult(r)` | `{type:"tool_result", tool_use_id, content, is_error}` accumulated into a *single* trailing user message (`llm.rs:429-431`, flushed at `llm.rs:394-399`) | `{role:"tool", tool_call_id, content: <text>}` (`llm.rs:532-534`) | `{type:"function_call_output", call_id, output}` (`llm.rs:621-625`) |

Two ordering invariants are encoded in the builders rather than the types:
- Anthropic requires all `tool_result` blocks answering one assistant turn to arrive in one immediately-following user message; `anthropic_body` batches them via the `pending`/`flush` closure (`llm.rs:393-399`, `llm.rs:434`).
- OpenAI Chat requires the run of `role:"tool"` messages to stay contiguous, so images are deferred into a single trailing `role:"user"` message via `pending_images` (`llm.rs:487-494`, flushed at `llm.rs:537`). The rationale — an OpenAI→Anthropic translating frontend such as Databricks model serving — is documented at `llm.rs:478-486` and regression-tested by `openai_parallel_image_tool_results_stay_contiguous` (`llm.rs:1596`).
- Responses requires each `function_call` to precede its `function_call_output`; this is asserted only as a comment block (`llm.rs:583-587`) plus the test `responses_body_replay_emits_function_call_before_output` (`llm.rs:1365`), relying on `HistoryItem` insertion order rather than a type constraint.

#### Tool-call representation
`ToolCall` (`types.rs:218-222`) carries `provider_id: String` (not `Option`), `name: String`, `arguments: Value`. The three wire encodings differ in how `arguments` is carried:

| Dialect | Encoding of arguments | Site |
|---|---|---|
| Anthropic `tool_use` | JSON **object** inline as `input` | `llm.rs:425-426` |
| OpenAI Chat `tool_calls[]` | JSON **string** under `function.arguments`, serialized with `unwrap_or_else(\|_\| "{}")` | `llm.rs:513-519` |
| Responses `function_call` | JSON **string** under `arguments`, same fallback | `llm.rs:611-618` |

On the parse side, all three converge on `make_tool_call` (`llm.rs:963-982`), which is the single normalizer: it rejects an empty `id` **or** empty `name` (`llm.rs:964-966`), passes `Value::Object` through, coerces `Value::Null` to `{}` (`llm.rs:969-970`), and rejects any other JSON type (`llm.rs:971-975`).

Tool definitions (`ToolDef`, `types.rs:167-171`) are also reshaped three ways: Anthropic `{name, description, input_schema}` (`llm.rs:439-441`), Chat `{type:"function", function:{name, description, parameters}}` (`llm.rs:540-544`), Responses **flat** `{type:"function", name, description, parameters}` (`llm.rs:653-659`). The flatness of the Responses form is asserted at `llm.rs:1355-1359`.

#### Image content
`ToolResultContent::Image { data, mime_type }` (`types.rs:136`) is rendered three ways:

| Dialect | Shape | Site |
|---|---|---|
| Anthropic | `{type:"image", source:{type:"base64", media_type, data}}` | `llm.rs:463-475` |
| OpenAI Chat | text placeholder in the `role:"tool"` message plus `{type:"image_url", image_url:{url:"data:<mime>;base64,<data>"}}` on a trailing user message | `llm.rs:555-587` |
| Responses | `{type:"input_image", image_url:"data:<mime>;base64,<data>"}` on a trailing user message | `llm.rs:632-651` |

The Chat placeholder text is generated in `openai_tool_text_content` (`llm.rs:555-568`) and reports `data.len()` base64 bytes.

#### Response parse shapes
| Parser | Reads | Stop derivation | Site |
|---|---|---|---|
| `parse_anthropic` | `content[]` blocks of type `text`, `thinking`, `tool_use` | `map_stop(stop_reason)` (`llm.rs:870`) | `llm.rs:869-911` |
| `parse_openai` | `choices[0].message.{content, reasoning_content, reasoning, tool_calls}` | `map_stop(finish_reason)` (`llm.rs:917`) | `llm.rs:912-962` |
| `parse_responses` | `output[]` items of type `message`, `function_call`, `reasoning` | `status` + `incomplete_details.reason`, plus a `saw_function_call` flag | `llm.rs:706-806`; stop logic `llm.rs:781-796` |

`map_stop` (`llm.rs:807-820`) is the shared string→`ProviderStop` table, accepting both dialects' vocabularies for each variant (`end_turn|stop`, `tool_use|tool_calls`, `max_tokens|length`, `refusal|content_filter`, else `Other`). `parse_responses` does **not** use `map_stop` — it has its own status mapping (`llm.rs:781-796`), so `ProviderStop::Refusal` is unreachable on the Responses path. That asymmetry is untested: grep for `Refusal` in `llm.rs` returned only the `map_stop` definition line, no test.

Both `parse_anthropic` and `parse_openai` are *forgiving* about missing fields via `str_field` (`llm.rs:865-867`, `unwrap_or("")`), but `parse_openai` hard-errors when `choices` is absent (`llm.rs:913-917`), when `message` is absent (`llm.rs:918-920`), and when a `tool_calls[]` entry lacks `function` (`llm.rs:936-938`).

#### Reasoning / thinking content
`LlmResponse.reasoning: String` (`types.rs:151`) is filled from three different places:
- Responses: concatenated `summary[].text` on `type == "reasoning"` items, accepting `summary_text` or `text` (`llm.rs:764-780`).
- Anthropic: concatenated `thinking` from `type == "thinking"` blocks (`llm.rs:879-886`).
- OpenAI Chat: `message.reasoning_content` preferred, falling back to `message.reasoning` (`llm.rs:922-932`) — documented as DeepSeek/vLLM extensions.

There are no tests for any of the three reasoning extraction paths: grep for `reasoning` inside the `llm.rs` test module found only `reasoning_effort` body-shape assertions and one empty `"summary": []` fixture at `llm.rs:1470`.

#### Usage / token accounting
`LlmResponse.input_tokens` and `output_tokens` are `Option<u64>` (`types.rs:139`, `types.rs:145`). `sum_usage` (`llm.rs:822-836`) is the single accumulator: it returns `None` when `usage` is absent **or** when it carries none of the requested fields, and otherwise a saturating sum of whichever requested fields are present.

| Field | Wire fields summed | Site |
|---|---|---|
| Anthropic input | `input_tokens` + `cache_read_input_tokens` + `cache_creation_input_tokens` | `anthropic_input_tokens`, `llm.rs:838-848` |
| OpenAI Chat / legacy Databricks input | `prompt_tokens` + `cache_read_input_tokens` + `cache_creation_input_tokens` | `openai_chat_input_tokens`, `llm.rs:854-863` |
| Responses input | `input_tokens` only | `llm.rs:797` |
| Anthropic output | `output_tokens` | `llm.rs:906` |
| Chat output | `completion_tokens` | `llm.rs:957` |
| Responses output | `output_tokens` | `llm.rs:798` |

The Responses path deliberately does *not* sum cache fields (`llm.rs:797`), unlike the other two — no comment explains the asymmetry, and no test covers a Responses response that carries cache fields.

Token accounting is well covered: `parse_anthropic_sums_input_and_cache_tokens` (`llm.rs:2505`), `parse_anthropic_input_tokens_only` (`llm.rs:2523`), `parse_anthropic_missing_usage_is_none` (`llm.rs:2533`), `parse_openai_uses_prompt_tokens` (`llm.rs:2542`), `parse_openai_databricks_sums_cache_fields` (`llm.rs:2551`), `parse_openai_missing_usage_is_none` (`llm.rs:2568`), `parse_responses_uses_input_tokens` (`llm.rs:2576`), `parse_responses_missing_usage_is_none` (`llm.rs:2590`), `sum_usage_empty_object_is_none` (`llm.rs:2603`), plus six output-token tests at `llm.rs:2828-2895`.

#### Byte-budget constants defined in this group
| Constant | Value | Site |
|---|---|---|
| `MAX_LLM_RESPONSE_BYTES` | 16 MiB | `llm.rs:22` |
| `MAX_LLM_ERROR_BODY_BYTES` | 4 KiB | `llm.rs:23` |
| `STALL_NOTICE_THRESHOLD` | 300 s | `llm.rs:24` |
| `MAX_RETRIES` / `BASE_BACKOFF_MS` / `MAX_BACKOFF_MS` | 3 / 500 ms / 8000 ms | `llm.rs:1000-1002` |
| `MAX_PROMPT_BYTES` | 1 MiB | `config.rs:638` |
| `MAX_SYSTEM_PROMPT_BYTES` | 512 KiB | `config.rs:639` |
| `MAX_TOOL_RESULT_BYTES` | 8 MiB | `config.rs:643` |
| `DEFAULT_TOOL_RESULT_TEXT_BYTES` | 50 KiB | `config.rs:649` |
| `MAX_TOOL_CALLS_PER_TURN` | 64 | `config.rs:650` |
| `HANDOFF_MAX_OUTPUT_TOKENS` | 8192 | `config.rs:652` |
| `HANDOFF_ORIGINAL_TASK_MAX_BYTES` | 16 KiB | `config.rs:654` |
| `HANDOFF_MAX_TOOL_NAMES` | 20 | `config.rs:656` |
| `PROTOCOL_VERSION` | 2 | `config.rs:3` |

#### Test coverage for this aspect
Body-shape and parse-shape coverage is dense (~60 `#[test]` fns in `llm.rs:1218-2896`). What is **not** covered:
- No test constructs a `Config` through `from_env` (it reads process env, so it is untestable without env mutation) — every test builds a `Config` literal (`llm.rs:1225-1256`) or patches `for_discovery` output (`config.rs:1836-1857`). The 26-field literal at `llm.rs:1225-1256` must be updated by hand whenever a field is added.
- No test asserts `for_discovery`'s output would fail `validate()`, nor that it is intentionally exempt.
- No test covers `ProviderStop::Refusal` on any path, or `ProviderStop::Other`.
- No test covers `strip_model` (`llm.rs:1206-1215`) — grep for `strip_model` in `llm.rs` returned only the definition (`llm.rs:1206`) and its one call site (`llm.rs:339`).
