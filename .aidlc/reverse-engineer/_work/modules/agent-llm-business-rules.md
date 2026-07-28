## Module: buzz-agent — LLM providers & configuration (`crates/buzz-agent/src`)
### Aspect: Business Rules
The rules here fall into four clusters: provider/endpoint selection, retry and auth recovery, thinking-effort normalization, and configuration precedence and validation.

#### Configuration precedence
There is no config file and no CLI flag layer, so precedence is only two-deep.

| Rule | Site |
|---|---|
| `BUZZ_AGENT_PROVIDER` is mandatory — absent or whitespace-only is a startup error, with no inference from which API keys are present | `config.rs:1007-1010` |
| `BUZZ_AGENT_MODEL` beats the provider-specific model var (`ANTHROPIC_MODEL` / `OPENAI_COMPAT_MODEL` / `DATABRICKS_MODEL`) | `resolve_model`, `config.rs:971-977`; call sites `config.rs:760-763`, `config.rs:770-773`, `config.rs:780` |
| Provider name matching is case-insensitive and trimmed; aliases `openai-compat`, `databricks-v2` accepted | `config.rs:988-1002` |
| An unsupported provider value is echoed back with the user's original casing (not the lowercased form) | `config.rs:1002-1004`; tested `config.rs:1261` |
| `BUZZ_AGENT_SYSTEM_PROMPT` and `BUZZ_AGENT_SYSTEM_PROMPT_FILE` are mutually exclusive — both set is a hard error | `config.rs:786-788` |
| When neither is set, the built-in prompt is used | `config.rs:658-659`, selected at `config.rs:791` |
| Every numeric var is parsed via `parse_env`, which returns the default only when the var is *absent*; a present-but-unparseable value is a startup error | `config.rs:1041-1048` |
| `OPENAI_COMPAT_API` is only parsed on the `Provider::OpenAi` arm; Anthropic gets a hard-coded `Auto` and Databricks a hard-coded `Chat` | `config.rs:776` vs `config.rs:767` and `config.rs:781` |

The `openai_api` inert values are inconsistent: Anthropic gets `OpenAiApi::Auto` (`config.rs:767`, commented "unused for Anthropic") while both Databricks providers get `OpenAiApi::Chat` (`config.rs:781`). For legacy `Provider::Databricks` the value is *not* inert — it is read at `llm.rs:279-280` and `llm.rs:294`. Setting it to `Chat` therefore permanently disables the chat→responses auto-upgrade for legacy Databricks, because the upgrade guard at `llm.rs:294` requires `cfg.openai_api == OpenAiApi::Auto`. This is a real behavioural rule that exists only as a side effect of an "inert default", with no comment saying so.

#### Provider selection and key requirements
| Rule | Site |
|---|---|
| `anthropic` requires a non-empty, non-whitespace `ANTHROPIC_API_KEY`; otherwise error | `config.rs:991-994`, emptiness via `present_nonempty` `config.rs:978-980` |
| `openai`/`openai-compat` requires a non-empty `OPENAI_COMPAT_API_KEY` | `config.rs:995-998` |
| `databricks` and `databricks_v2` require **no** key at `resolve_provider` time — `DATABRICKS_TOKEN` is optional and defaults to `""` | `config.rs:999-1000`, `config.rs:779` |
| An empty `api_key` for either Databricks provider means "use OAuth 2.0 PKCE" | `llm.rs:1181-1199` |
| `DATABRICKS_HOST` and a model are required, but validated later in `from_env`, not in `resolve_provider` | `config.rs:780-782` |
| There is no fallback between providers — a missing key never silently degrades to Databricks | tested `config.rs:1228`, `config.rs:1245` |

#### Endpoint selection (OpenAI family)
`use_responses` is a three-term disjunction at `llm.rs:278-280`:
1. the sticky `auto_upgraded` latch, or
2. `openai_api == Responses` (pinned), or
3. `openai_api == Auto` **and** `is_openai_host(base_url)`.

`is_openai_host` (`config.rs:1029-1039`) accepts a URL only if it starts with `https://` **or** `http://` (`config.rs:1031-1032`), then takes the host up to the first `/` or `:` and matches `api.openai.com` or any `*.openai.com` suffix (`config.rs:1038`). Lookalike hosts such as `api.openai.com.evil.example` return `false` (tested `config.rs:1277`), and a non-URL string returns `false` (tested `config.rs:1278`). The rule is documented and tested by `is_openai_host_matrix` (`config.rs:1267-1283`).

Note the rule is host-only, **not** scheme-gated: `http://eu.api.openai.com/v1` returns `true` and is asserted to do so (`config.rs:1273`). A plaintext base URL for an OpenAI host is therefore treated as fully trusted for endpoint-selection purposes.

Legacy Databricks ignores `path` entirely: `post_openai` rewrites the URL to `{base}/serving-endpoints/{model}/invocations` for `Provider::Databricks` regardless of whether the caller asked for `/chat/completions` or `/responses` (`llm.rs:333-341`). Combined with the forced `OpenAiApi::Chat` above, this is currently unreachable, but the code shape allows a Responses-shaped body to be POSTed to the Chat invocation URL if the upgrade latch were ever set for a Databricks process.

#### Chat → Responses auto-upgrade (one-way, process-wide)
| Rule | Site |
|---|---|
| Triggered only from `AgentError::Llm` (never auth or transport errors) | `llm.rs:373-377` |
| Matcher is a lowercase substring test on the provider error body: `/v1/responses`, `responses api instead`, or `use the responses api` | `llm.rs:678-683` |
| Latch is a one-way `AtomicBool::swap(true)`; warns exactly once | `llm.rs:384-390` |
| Only retried when `openai_api == Auto` | `llm.rs:294` |
| Latch is per-`Llm`-instance, i.e. per-process, and never reset | `llm.rs:41-44`; no `store(false)` anywhere — grep for `auto_upgraded` in `llm.rs` returned only `llm.rs:41`, `llm.rs:62`, `llm.rs:278`, `llm.rs:385`, `llm.rs:2693` |

The matcher's false-positive risk is bounded by design and tested: `is_responses_required_error_matrix` (`llm.rs:1503-1517`) covers the real Databricks GPT-5.5 message, a forward-compat prose variant, and three negatives including `{"error":"invalid_api_key"}` and the empty string. What is **not** tested is the latch behaviour itself — no test exercises `try_upgrade` (`llm.rs:372`) or asserts that a second call skips Chat Completions.

#### DatabricksV2 route selection
`databricks_v2_route_for_model` (`llm.rs:685-697`) lowercases and does plain `contains` checks:

| Condition | Route | Path |
|---|---|---|
| contains `gpt-5` or `gpt5` | `OpenAiResponses` | `/ai-gateway/openai/v1/responses` (`llm.rs:700`) |
| else contains `claude` | `AnthropicMessages` | `/ai-gateway/anthropic/v1/messages` (`llm.rs:701`) |
| else | `MlflowChatCompletions` | `/ai-gateway/mlflow/v1/chat/completions` (`llm.rs:702`) |

This is a **second, looser** model-family classifier than the boundary-safe one in `config.rs`. `config.rs` deliberately built `gpt5_token_matches` (`config.rs:239-254`) and `gpt5_base_matches` (`config.rs:267-311`) to prevent `gpt-5.1` matching `gpt-5.10`, `gpt-5-1` matching `gpt-5-1106`, and `gpt-5-4` matching `gpt-5-4o` — with eight boundary tests (`config.rs:2278-2399`). `llm.rs:689-690` throws all of that away for the routing decision. Consequence: a model named `databricks-gpt-5-10` routes to the OpenAI Responses path (`contains("gpt-5")` is true) while `openai_efforts_for_model("databricks-gpt-5-10")` returns `None` (asserted at `config.rs:2338`) and its effort is passed through unverified. The two classifiers disagree on the same string in the same request, and no test covers that pairing — `databricks_v2_routes_by_model_family` (`llm.rs:1519-1542`) only tests three unambiguous names.

The `contains("claude")` check is also unanchored: a model named `my-claude-killer-llama` would route to the Anthropic Messages path.

#### Retry and backoff
All retry logic lives in `post` (`llm.rs:1052-1163`).

| Parameter | Value | Site |
|---|---|---|
| `MAX_RETRIES` (total attempts, not extra retries) | 3 | `llm.rs:1000` |
| `BASE_BACKOFF_MS` | 500 | `llm.rs:1001` |
| `MAX_BACKOFF_MS` | 8000 | `llm.rs:1002` |
| Backoff formula | `min(500 << attempt, 8000)`, then subtract a uniform jitter in `[0, base/2)` | `llm.rs:1005-1016` |
| Effective delays | attempt 0 → 250-500 ms, attempt 1 → 500-1000 ms | derived from `llm.rs:1006-1008` |
| Jitter source | `getrandom::fill`; on RNG failure the full `base` is used with no jitter | `llm.rs:1011-1015` |

Retryable conditions:
- Transport errors where `is_timeout() || is_connect() || is_request()` (`llm.rs:1026-1028`). The `is_request()` term is the important one — it catches a socket accepted then dropped before any response bytes, and is regression-tested by `post_retries_on_dropped_connection_before_response` (`llm.rs:2177`).
- Any 5xx, plus 429, plus the non-standard 499 (`llm.rs:1093`). 499 handling is tested twice: recovery (`post_retries_499_and_succeeds`, `llm.rs:2242`) and exhaustion (`post_exhausts_retries_on_persistent_499`, `llm.rs:2306`, which asserts exactly `MAX_RETRIES` server-side accepts).

Non-retryable: 401/403 (routed to auth recovery instead, `llm.rs:1086-1088`), 404 (`llm.rs:1108-1113`), any other non-2xx (`llm.rs:1114-1119`), and JSON decode failure (`llm.rs:1161`). A **body-read** error mid-stream is explicitly not retried (`llm.rs:1146-1152`) even though the request was already sent — the comment does not say why, and there is no test.

The retry loop ends in `unreachable!()` (`llm.rs:1162`) with a message asserting the final iteration always returns. That is a production-path panic guarded only by `MAX_RETRIES > 0`; it is not an `#[allow]`-suppressed dead branch but a real reachable panic if the constant were changed to 0.

`terminal_llm_error` (`llm.rs:1038-1051`) is the single exit-formatter for give-up paths: it appends `(cumulative <dur>, N attempt[s])` and, when cumulative elapsed `>= STALL_NOTICE_THRESHOLD` (300 s, `llm.rs:24`), also emits a `tracing::warn!`. The threshold is a strict `>=` and both sides of the boundary are mutation-tested: `terminal_llm_error_below_threshold_emits_no_stall_warning` at 299 s (`llm.rs:2479`) and `terminal_llm_error_at_threshold_emits_one_stall_warning` at exactly 300 s (`llm.rs:2492`), using a scoped `tracing` subscriber (`llm.rs:2412-2472`). Pluralization is tested too (`llm.rs:2394`).

Two paths deliberately bypass `terminal_llm_error` and are annotated: 401/403 (`llm.rs:1080-1085`) and 404 (`llm.rs:1106-1107`).

#### Timeouts
| Timeout | Value | Site |
|---|---|---|
| Connect | 10 s, hard-coded | `llm.rs:55` |
| Read (per-read inactivity) | `cfg.llm_timeout`, default 240 s | `llm.rs:56`, default `config.rs:803` |
| Total request / wall-clock | **none** | see below |

There is no `.timeout(...)` on the production client — `Llm::new` (`llm.rs:54-58`) sets only `connect_timeout` and `read_timeout`. A provider that trickles one byte every 239 s can hold a request open indefinitely; the only backstop is `STALL_NOTICE_THRESHOLD`, which merely logs (`llm.rs:1039-1045`). grep for `.timeout(` in `llm.rs` returned five matches, all inside the test module (`llm.rs:2226`, `llm.rs:2290`, `llm.rs:2341`, `llm.rs:2692`) — so the tests run against a *stricter* client than production, and the production timeout configuration is never exercised. The `README.md:150` wording "per-read inactivity, not wall-clock" is accurate and honest about this.

#### Auth recovery (401/403 refresh-once)
| Rule | Site |
|---|---|
| A bearer is fetched once per `post_openai` call, before the loop | `llm.rs:352` |
| 401 **and** 403 both map to `AgentError::LlmAuth` | `llm.rs:1086-1088` |
| On the first `LlmAuth`, force `refresh_now(&rejected_bearer)` and retry once | `llm.rs:354-362` |
| The `refreshed` guard is a local, so an earlier turn cannot suppress a later turn's retry | `llm.rs:353`, rationale `llm.rs:343-351` |
| Anthropic never uses this path — it sends `x-api-key` directly from `cfg.api_key` | `llm.rs:256-264`, rationale `llm.rs:45-49` |

Well covered: `post_openai_refreshes_once_per_call_on_401` (`llm.rs:2704`) asserts one refresh *and* that a second call gets its own; `post_openai_persistent_401_propagates_after_one_retry` (`llm.rs:2740`); `post_openai_persistent_403_propagates_after_one_retry` (`llm.rs:2768`); `post_openai_refreshes_once_on_403` (`llm.rs:2796`).

Treating 403 as refreshable is a deliberate trade documented at `llm.rs:1080-1085` and `llm.rs:343-351`: a pure authorization 403 costs one wasted refresh.

#### Thinking-effort normalization
Startup validation is asymmetric by design (`config.rs:932-949`): only `Provider::Anthropic` rejects `none`/`minimal` at startup (`config.rs:941-948`); OpenAI, Databricks, and DatabricksV2 accept all seven values and defer to request time, because `session/set_model` can change the model after startup. The rationale is a comment block at `config.rs:932-938`. All four provider × 7 effort combinations are tested (`config.rs:1859`, `config.rs:1877`, `config.rs:1885`, `config.rs:1906`, `config.rs:1926`, `config.rs:1946`, `config.rs:1956`, `config.rs:1963`, `config.rs:1970`).

Request-time normalization:

| Route | Normalizer | Site |
|---|---|---|
| Pure Anthropic (`Provider::Anthropic`) | **none** — `cfg.thinking_effort` is passed raw to `anthropic_body` | `llm.rs:75`, `llm.rs:78-88` |
| OpenAI / legacy Databricks | `normalize_effort_for_openai_route` | called `llm.rs:94`, defined `config.rs:538-550` |
| DBv2 OpenAI Responses route | `normalize_effort_for_openai_route` | `llm.rs:113-114` |
| DBv2 Anthropic Messages route | `normalize_effort_for_anthropic_route` | `llm.rs:121` |
| DBv2 MLflow Chat route | `normalize_effort_for_openai_route` | `llm.rs:130-131` |

The pure-Anthropic asymmetry is real: `Provider::Anthropic` never calls `normalize_effort_for_anthropic_route`, so `none`/`minimal` reaching `anthropic_body` would be handled only by the defensive fallbacks at `config.rs:41-43` (budget 0) and `config.rs:69-70` (string `"low"`). Startup validation makes that unreachable *today*, but the two defensive fallbacks are the only thing standing between a validation-bypass and a wrong `output_config.effort`. Both fallbacks are tested (`config.rs:1360-1362`, `config.rs:1383-1385`), and both carry comments saying they exist only because startup rejects those values.

`normalize_effort_for_openai_route` (`config.rs:538-550`):
- known model family → `resolve_openai_effort` nearest-supported substitution;
- unknown family + `Max` → clamp to `XHigh` with a warn (`config.rs:541-548`);
- unknown family + anything else → pass through unchanged (`config.rs:549`).

`resolve_openai_effort` (`config.rs:466-520`) implements two rules:
1. `none ↔ minimal` are each other's first fallback — explicit `peer` branch at `config.rs:479-483`, applied at `config.rs:494-498`.
2. Otherwise, nearest by ordinal distance with upward ties winning — `config.rs:486-491`.

The doc comment at `config.rs:459` states as a rule that "`xhigh` falls back to `high` when not supported (no model skips from `high` to `xhigh`)". **This rule is not implemented explicitly** — it is an emergent property of the distance sort at `config.rs:486-491` plus the shape of the tables at `config.rs:334-359`. If a future table ever supported `max` but not `xhigh`, `xhigh` would resolve upward to `max`, contradicting the stated rule while every existing test still passed. It is tested only for `gpt-5.1` (`config.rs:2480`).

`normalize_effort_for_anthropic_route` (`config.rs:563-577`) collapses `none`/`minimal` to `None` (omit thinking fields entirely) with a warn, and passes everything else through. Its warning text is honest that omission is not the same as disabling thinking (`config.rs:571-573`: "default-on/always-on adaptive models may still think"). Tested at `config.rs:2018`, `config.rs:2027`, `config.rs:2036`.

#### Anthropic thinking-shape rules
`anthropic_thinking_config` (`config.rs:124-178`) picks between three shapes after stripping catalog prefixes (`config.rs:134`, helper `config.rs:89-97`):

| Bucket | Predicate | Emitted fields |
|---|---|---|
| Manual budget | `is_manual_budget_model` — `claude-3*` prefix **or** exactly `claude-opus-4-5` | `thinking: {type:"enabled", budget_tokens}`, no `output_config` (`config.rs:136-160`) |
| Adaptive | `is_adaptive_thinking_model` — explicit list of nine prefixes | `thinking: {type:"adaptive"}` + `output_config: {effort}` (`config.rs:161-170`) |
| Unknown | neither | both omitted (`config.rs:171-176`) |

Manual-budget clamp rule (`config.rs:141-158`): `budget = min(level_budget, max_output_tokens - 1024)`; if the result is `< 1024`, thinking is **omitted entirely** with a `warn!` rather than sending an answer-starving budget. The 1024 reserve is a local `const MIN_ANSWER_TOKENS` (`config.rs:145`). Boundary behaviour is tested on both sides: omit at `max_output_tokens = 2047` (`config.rs:1415`) and at 1025 (`config.rs:1513`); emit at exactly 2048 with `budget_tokens == 1024` (`config.rs:1427`, mirrored at `llm.rs:1720`).

The bucket boundaries are asserted as *doc-verified* against Anthropic's published tables with dated comments (`config.rs:100-122`, `config.rs:578-620`), and the "omit rather than guess" rule for unrecognised `claude-*` names is explicitly tested for five names including a hypothetical `claude-opus-4-9` (`config.rs:1534-1557`).

`clamp_adaptive_effort` (`config.rs:205-236`) enforces one rule: `xhigh` → `high` for adaptive models that do not support `xhigh`; everything else including `max` passes through. The xhigh-support set is `anthropic_model_supports_xhigh` (`config.rs:184-190`), shared with `anthropic_efforts_for_model` (`config.rs:443`) — genuinely a single source of truth for that one predicate, unlike the manual/adaptive split which is duplicated.

Note the clamp is asymmetric with the capability tables: `anthropic_efforts_for_model` returns `ADAPTIVE_NO_XHIGH = [low, medium, high, max]` for non-xhigh adaptive models (`config.rs:428-433`, `config.rs:446`), so `max` is advertised as valid — and `clamp_adaptive_effort` agrees by passing `max` through (`config.rs:212-214`). Consistent, and tested (`config.rs:1696`, `config.rs:2133`).

#### Tool-call parsing rules
| Rule | Site |
|---|---|
| `arguments` must parse as JSON; a parse failure is a hard `AgentError::Llm`, not a silent `{}` | Responses `llm.rs:741-747`; Chat `llm.rs:941-944` |
| A missing `arguments` string defaults to the literal `"{}"` before parsing | `llm.rs:742`, `llm.rs:941` |
| A tool call with empty `id` or empty `name` is rejected | `llm.rs:964-966` |
| `arguments` must be a JSON object; `null` becomes `{}`; any other type is rejected | `llm.rs:968-976` |
| Unknown Responses `output[]` item types are ignored for forward compatibility | `llm.rs:801-802` |
| Responses `output_text` parts accept the alias `text`; `summary_text` accepts `text` | `llm.rs:730-733`, `llm.rs:770-773` |
| An empty Anthropic assistant turn (no text, no tool calls) is skipped rather than sent as an empty block | `llm.rs:417-423` |
| An empty Responses assistant text is skipped but its tool calls still emit | `llm.rs:601-619`; tested `llm.rs:1401` |

Malformed-arguments rejection is tested for Responses (`llm.rs:1544`) but **not** for Chat Completions — grep for `tool_call.arguments not valid JSON` in the `llm.rs` test module returned zero matches, so the `llm.rs:943` error path is untested. Likewise `make_tool_call`'s three rejection branches (`llm.rs:964`, `llm.rs:971`) have no direct test.

#### Truncation and context-window handling
Neither file truncates history or counts tokens. `max_history_bytes` (`config.rs:700`) and `max_context_tokens` (`config.rs:715`) are only *defined* and *validated* here; the enforcement lives in `handoff.rs` and `agent.rs` (`handoff.rs:162` references `MAX_PROMPT_BYTES`; `agent.rs:68` enforces it). `llm.rs` sends whatever history it is handed — grep for `truncat`, `elide`, and `max_history_bytes` in `llm.rs` returned zero matches outside the test `Config` literal at `llm.rs:1237`.

Response-size caps are enforced, however: a `Content-Length` precheck against `MAX_LLM_RESPONSE_BYTES` (`llm.rs:1121-1128`) plus a streaming-buffer cap (`llm.rs:1129-1145`), and a 4 KiB cap on error bodies (`read_error_body`, `llm.rs:983-998`). No test covers either cap — grep for `MAX_LLM_RESPONSE_BYTES` and `response too large` in the `llm.rs` test module returned zero matches.

#### Rules enforced only by comment or convention
1. Responses replay ordering (`function_call` before `function_call_output`) — comment at `llm.rs:583-587`, no type or runtime check; relies on `HistoryItem` insertion order.
2. `xhigh → high` fallback preference — stated at `config.rs:459`, implemented only as an emergent property of the distance sort (`config.rs:486-491`).
3. `ThinkingEffort` discriminants being contiguous 0..=6 — required by the `as i32` distance metric at `config.rs:487`, guaranteed only by declaration order (`config.rs:19-27`).
4. `MAX_RETRIES >= 1` — required to make `llm.rs:1162`'s `unreachable!()` actually unreachable; asserted only in that panic message.
5. `for_discovery` being exempt from `validate` — stated at `config.rs:832-836`, enforced by nothing.
6. `anthropic_efforts_for_model` being "the single production source of truth" for Anthropic family routing (`config.rs:407`) — contradicted by `anthropic_thinking_config` classifying independently (`config.rs:136`, `config.rs:161`).
7. The `*` wildcard in `MCP_HOOK_SERVERS` being honoured only as a sole entry, on the grounds that `*` cannot pass the MCP name validator (`config.rs:1098-1102`) — a cross-module assumption with no compile-time link. Tested behaviourally at `config.rs:1158` and `config.rs:1186`, including the deliberate `hs.allows("*") == true` at `config.rs:1195`.

#### Tests that re-implement production rules locally
`valid_effort_values_for_provider_model` (`config.rs:2559-2650`) is a test-only function that re-implements the provider→effort-table routing which the TypeScript `getProviderEffortConfig` performs. The fixture guard `effort_table_fixture_matches_rust_implementation` (`config.rs:2666-2700`) tests *this re-implementation* against `effortTable.fixture.json`, not any production function. Three rules exist only inside it:
- the catalog-prefix strip at `config.rs:2570-2582` duplicates `strip_catalog_prefix` (`config.rs:89-97`) line-for-line rather than calling it;
- the default-effort-per-family derivation at `config.rs:2599-2607` (`GPT5_PRO → high`, `GPT5_1 → none`, else `medium`) exists nowhere in production;
- the DBv2 gpt-5 family predicate at `config.rs:2623-2636` duplicates the chain from `config.rs:367-392` while omitting the `gpt-5-6`/`gpt5-6` dash variants that production checks at `config.rs:371-372`.

That last omission is harmless today only because both the `is_gpt5` branch (`config.rs:2637-2639`) and the fall-through (`config.rs:2640-2643`) return the identical `openai_result(&m)` — making the 14-line `is_gpt5` computation dead for every non-empty model name. The fixture can therefore go green while the production and test routing rules diverge.
