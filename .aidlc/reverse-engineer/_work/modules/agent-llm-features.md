## Module: buzz-agent — LLM providers & configuration (`crates/buzz-agent/src`)
### Aspect: Features
This group is the provider abstraction layer: four providers, three wire dialects, five HTTP endpoint shapes, and a per-model reasoning-effort capability system.

#### Providers supported
| `BUZZ_AGENT_PROVIDER` | `Provider` variant | Wire dialect(s) | Auth | Site |
|---|---|---|---|---|
| `anthropic` | `Anthropic` (`config.rs:663`) | Anthropic Messages | `x-api-key` header | `llm.rs:77-89`, `llm.rs:256-264` |
| `openai`, `openai-compat` | `OpenAi` (`config.rs:664`) | Chat Completions or Responses | `Authorization: Bearer` | `llm.rs:90-107`, `llm.rs:269-303` |
| `databricks` | `Databricks` (`config.rs:668`) | OpenAI-Chat-compatible, model in URL path | Bearer, static token or OAuth PKCE | `llm.rs:333-341`, `llm.rs:1179-1199` |
| `databricks_v2`, `databricks-v2` | `DatabricksV2` (`config.rs:677`) | routed per model family across three gateway dialects | Bearer, static token or OAuth PKCE | `llm.rs:108-139`, `llm.rs:304-318` |

Provider aliases are accepted at `config.rs:995` (`openai-compat`) and `config.rs:1000` (`databricks-v2`); neither alias appears in `crates/buzz-agent/README.md:132`, which lists only `anthropic`, `openai`, `databricks`, `databricks_v2`.

#### Endpoints reached
| Provider | Method + URL | Site |
|---|---|---|
| Anthropic | `POST {base_url}/v1/messages` | `llm.rs:257` |
| OpenAI (Responses) | `POST {base_url}/responses` | `llm.rs:285`, `llm.rs:297` |
| OpenAI (Chat) | `POST {base_url}/chat/completions` | `llm.rs:290` |
| Databricks (legacy) | `POST {base_url}/serving-endpoints/{effective_model}/invocations` | `llm.rs:335-339` |
| DatabricksV2 (GPT-5 family) | `POST {base_url}/ai-gateway/openai/v1/responses` | `llm.rs:700` |
| DatabricksV2 (Claude family) | `POST {base_url}/ai-gateway/anthropic/v1/messages` | `llm.rs:701` |
| DatabricksV2 (everything else) | `POST {base_url}/ai-gateway/mlflow/v1/chat/completions` | `llm.rs:702` |

Trailing slashes on `base_url` are stripped at every construction site (`llm.rs:257`, `llm.rs:337`, `llm.rs:343`, `llm.rs:1190`).

#### Capabilities exposed per dialect
| Capability | Anthropic | OpenAI Chat | Responses |
|---|---|---|---|
| Tool calling | yes (`llm.rs:436-444`) | yes + `tool_choice:"auto"` (`llm.rs:538-553`) | yes + `tool_choice:"auto"` (`llm.rs:653-676`) |
| Parallel tool calls | yes, batched tool_results (`llm.rs:429-434`) | yes, contiguous `role:"tool"` run (`llm.rs:487-537`) | yes, one `function_call_output` each (`llm.rs:621-625`) |
| Image tool results | native `image` block (`llm.rs:466-470`) | data-URI `image_url` on a trailing user message (`llm.rs:569-587`) | `input_image` data URI (`llm.rs:635-651`) |
| Reasoning/thinking request | `thinking` + `output_config` (`llm.rs:449-458`) | `reasoning_effort` string (`llm.rs:550`) | `reasoning: {effort}` (`llm.rs:670`) |
| Reasoning content in response | `thinking` blocks (`llm.rs:879-886`) | `reasoning_content` / `reasoning` on the message (`llm.rs:922-932`) | `summary[].text` on `reasoning` items (`llm.rs:764-780`) |
| Cache-token accounting | yes, summed (`llm.rs:838-848`) | yes, summed (`llm.rs:854-863`) | **no** — `input_tokens` only (`llm.rs:797`) |
| Streaming | no | no (`"stream": false`, `llm.rs:548`) | no (no `stream` key emitted) |

#### Reasoning-effort levels
Seven levels, parsed from `BUZZ_AGENT_THINKING_EFFORT` (`config.rs:622-637`), case-insensitive and trimmed:

| Level | OpenAI string | Anthropic effort string | Anthropic manual budget |
|---|---|---|---|
| `none` | `none` | `low` (defensive only) | 0 |
| `minimal` | `minimal` | `low` (defensive only) | 0 |
| `low` | `low` | `low` | 1 024 |
| `medium` | `medium` | `medium` | 8 192 |
| `high` | `high` | `high` | 32 768 |
| `xhigh` | `xhigh` | `xhigh` | 32 768 (clamped) |
| `max` | `max` | `max` | 32 768 (clamped) |

Sites: string mapper `config.rs:45-57`, Anthropic mapper `config.rs:60-71`, budget mapper `config.rs:33-43`. All three are exhaustively tested (`config.rs:1351`, `config.rs:1364`, `config.rs:1375`).

#### OpenAI model families and their effort limits
`openai_efforts_for_model` (`config.rs:333-401`) is the doc-verified capability table:

| Family | Supported efforts | Table const | Match tokens |
|---|---|---|---|
| `gpt-5-pro` / `gpt5-pro` | `high` only | `GPT5_PRO` (`config.rs:335`) | `config.rs:367` |
| `gpt-5.6` / `gpt5.6` / `gpt-5-6` / `gpt5-6` | `none, low, medium, high, xhigh, max` | `GPT5_6` (`config.rs:336-343`) | `config.rs:369-373` |
| `gpt-5.5`, `gpt-5.4` (+ `gpt5.5`, `gpt-5-5`, `gpt5-5`, `gpt5.4`, `gpt-5-4`, `gpt5-4`) | `none, low, medium, high, xhigh` | `GPT5_5_AND_5_4` (`config.rs:344-350`) | `config.rs:375-384` |
| `gpt-5.1` / `gpt5.1` / `gpt-5-1` / `gpt5-1` | `none, low, medium, high` | `GPT5_1` (`config.rs:351-356`) | `config.rs:386-390` |
| `gpt-5` base / `gpt5` base | `minimal, low, medium, high` | `GPT5_BASE` (`config.rs:357-362`) | `config.rs:392` |
| anything else | not doc-verified → `None`; `max` clamps to `xhigh` | — | `config.rs:394-397`, clamp `config.rs:541-548` |

The `none` vs `minimal` split is the notable asymmetry: base `gpt-5` supports `minimal` but not `none`; the versioned families support `none` but not `minimal` (`config.rs:322-324`). Both directions are tested (`config.rs:2255`, `config.rs:2440`, `config.rs:2470`).

Match ordering matters and is documented: `-pro` is checked before the versioned strings so `gpt-5-pro` cannot fall into the base bucket (`config.rs:326-327`, `config.rs:365-367`), tested by `openai_efforts_for_model_pro_before_base_gpt5` (`config.rs:2401`).

Boundary safety is a first-class feature here, with two distinct matchers:
- `gpt5_token_matches` (`config.rs:239-254`) requires end-of-string or `-` after the token, so `gpt-5.1` does not match `gpt-5.10` and `gpt-5-4` does not match `gpt-5-4o`.
- `gpt5_base_matches` (`config.rs:267-311`) additionally rejects short 1-3-digit numeric suffixes (`-10`, `-10-preview`) as version-like while accepting 4+-digit date/build segments (`-1106`, `-0514`) and non-digit capability suffixes (`-pro`).

Eight boundary tests cover this (`config.rs:2278`, `config.rs:2296`, `config.rs:2315`, `config.rs:2355`, `config.rs:2379`, `config.rs:2390`, `config.rs:2401`, plus `config.rs:2508`).

#### Anthropic model families and their thinking shapes
| Family | Shape | Predicate site |
|---|---|---|
| `claude-3*` (all 3.x) | manual `budget_tokens` | `config.rs:587` |
| `claude-opus-4-5` (exact) | manual `budget_tokens` | `config.rs:587` |
| `claude-opus-4-6`, `-4-7`, `-4-8` | adaptive + `output_config.effort` | `config.rs:606-608` |
| `claude-sonnet-5*` | adaptive | `config.rs:610` |
| `claude-sonnet-4-6*` | adaptive | `config.rs:612` |
| `claude-fable-5*` | adaptive (always-on) | `config.rs:614` |
| `claude-mythos-5*` | adaptive (always-on) | `config.rs:615` |
| `claude-mythos-preview*` | adaptive (default-on) | `config.rs:618` |
| any other `claude-*` or non-Anthropic name | both fields omitted | `config.rs:171-176` |

`xhigh` support is a narrower set than adaptive membership (`anthropic_model_supports_xhigh`, `config.rs:184-190`): Opus 4.7, Opus 4.8, Sonnet 5, Fable 5, Mythos 5 — explicitly **not** Opus 4.6, Sonnet 4.6, or Mythos Preview (`config.rs:196-198`). `max` is supported by all adaptive families (`config.rs:428-433`).

Every family is individually tested: Claude 3 (`config.rs:1400`), Opus 4.5 (`config.rs:1489`), Opus 4.6 (`config.rs:1782`, `config.rs:1796`), Opus 4.7 (`config.rs:1456`), Opus 4.8 (`config.rs:1445`), Sonnet 5 (`config.rs:1467`), Sonnet 4.6 (`config.rs:1478`), Fable 5 (`config.rs:2055`), Mythos 5 (`config.rs:2066`), Mythos Preview (`config.rs:2077`), unknown Claude (`config.rs:1534`), non-Claude (`config.rs:1559`).

#### Catalog-prefix tolerance
`strip_catalog_prefix` (`config.rs:89-97`) makes the Anthropic classifiers work regardless of how a gateway names its endpoints: it finds the first occurrence of `claude-` or `gpt-` and drops everything before it, rather than maintaining an allowlist of prefixes. So `databricks-claude-opus-4-7`, `goose-claude-fable-5`, and `team-x-claude-opus-4-7` all resolve correctly. Tested for `databricks-` (`config.rs:1574`, `config.rs:1584`, `config.rs:1597`), `goose-` (`config.rs:1610`, `config.rs:1623`), and an arbitrary `team-x-` prefix (`config.rs:1635`).

Note: `openai_efforts_for_model` does **not** call `strip_catalog_prefix` — it relies on `gpt5_token_matches`/`gpt5_base_matches` tolerating a prefix because `-` is a legal boundary character (`config.rs:250-252`, `config.rs:265`). Both approaches work but the crate has two different prefix-tolerance mechanisms.

#### Auto-upgrade from Chat Completions to Responses
When `OPENAI_COMPAT_API=auto` and a Chat Completions call comes back with a "use `/v1/responses`" provider error, the process latches into Responses mode for its remaining lifetime (`llm.rs:41-44`, `llm.rs:294-299`, latch at `llm.rs:384-390`). Matcher covers the literal path and two prose phrasings (`llm.rs:678-683`).

#### Summarization
`Llm::summarize` (`llm.rs:161-254`) is a separate, tool-free single-turn completion used for context handoff. It takes an explicit `max_output_tokens` argument (callers pass `HANDOFF_MAX_OUTPUT_TOKENS` = 8192, `config.rs:652`, from `handoff.rs:51` and `handoff.rs:197`) rather than `cfg.max_output_tokens`, and it never sends tools, never sends thinking/reasoning config, and discards `LlmResponse.reasoning` — it returns only `.text` (`llm.rs:174`, `llm.rs:206`, `llm.rs:251`).

Its Anthropic body (`llm.rs:164-173`) is hand-built rather than routed through `anthropic_body`, and its Responses body passes `input` as a bare string (`llm.rs:187`) rather than the item array `responses_body` builds (`llm.rs:667`). Both forms are accepted by the respective APIs, but the duplication means a future body-shape fix must be applied in two places. There are no tests for `summarize` at any level — grep for `summarize` in the `llm.rs` test module returned zero matches.

#### Hook-server allowlist
`HookServers` (`config.rs:1055-1105`) supports three modes from a comma-separated `MCP_HOOK_SERVERS`: unset/empty/whitespace-only → `None` (hooks off, the default), a lone `*` → `All`, otherwise `Only([names])` with entries trimmed and empties dropped (`config.rs:1088-1104`). A mixed `*,foo` deliberately degrades to a literal `Only(["*","foo"])` to avoid silently widening scope on a typo (`config.rs:1098-1102`). Ten tests cover the parser and `allows()` (`config.rs:1111-1196`).

#### Limits and known non-features
| Limit | Value | Site |
|---|---|---|
| LLM response body | 16 MiB, checked twice (Content-Length + streaming) | `llm.rs:22`, `llm.rs:1121-1145` |
| LLM error body captured | 4 KiB | `llm.rs:23`, `llm.rs:983-998` |
| Attempts per LLM POST | 3 | `llm.rs:1000` |
| Backoff ceiling | 8 s (never reached at 3 attempts) | `llm.rs:1002` |
| Auth refresh attempts per call | 1 | `llm.rs:353-362` |
| Connect timeout | 10 s | `llm.rs:55` |
| Read (inactivity) timeout | `BUZZ_AGENT_LLM_TIMEOUT_SECS`, default 240 s | `llm.rs:56`, `config.rs:803` |
| Total wall-clock request timeout | **absent** | no `.timeout(` in `llm.rs` production code |

Not features of this group: streaming, token counting before send, prompt caching control, provider failover (a failed provider is not retried on a different provider — `Llm::complete`'s `match` has one arm per provider and no fallback), and multi-key rotation.
