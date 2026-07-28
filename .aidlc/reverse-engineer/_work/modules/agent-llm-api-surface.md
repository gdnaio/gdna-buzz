## Module: buzz-agent — LLM providers & configuration (`crates/buzz-agent/src`)
### Aspect: API Surface
Two very different visibility stories sit in this group. `config` is a `pub mod` (`lib.rs:6`) so everything marked `pub` in `config.rs` is part of the crate's external API. `llm` is a **private** module (`lib.rs:9`), so every `pub` item in `llm.rs` is effectively crate-internal despite the keyword.

#### Module visibility
| Declaration | Site | Effect |
|---|---|---|
| `pub mod config;` | `lib.rs:6` | all `pub` items in `config.rs` are externally reachable as `buzz_agent::config::*` |
| `mod llm;` | `lib.rs:9` | `pub struct Llm`, `pub fn new`, `pub async fn complete`, `pub async fn summarize` are **not** externally reachable |
| `pub use config::Provider;` | `lib.rs:15` | the only `config` item re-exported at crate root |

`lib.rs:14` and `lib.rs:16` re-export from `catalog` and `types`; nothing else from `config` or `llm` is re-exported. So `buzz_agent::Provider` is a supported name, while `buzz_agent::config::Config` requires the longer path.

#### Public API from `config.rs`
| Item | Signature / shape | Site | Doc comment? |
|---|---|---|---|
| `PROTOCOL_VERSION` | `pub const u32 = 2` | `config.rs:3` | no |
| `ThinkingEffort` | `pub enum`, 7 variants | `config.rs:19-27` | yes (`config.rs:5-17`) |
| `ThinkingEffort::anthropic_budget_tokens` | `pub fn(self) -> u32` | `config.rs:33` | yes |
| `ThinkingEffort::openai_effort_str` | `pub fn(self) -> &'static str` | `config.rs:45` | yes |
| `ThinkingEffort::anthropic_effort_str` | `pub fn(self) -> &'static str` | `config.rs:60` | yes |
| `anthropic_thinking_config` | `pub fn(&str, ThinkingEffort, u32) -> (Option<Value>, Option<Value>)` | `config.rs:124-128` | yes (`config.rs:100-123`) |
| `clamp_adaptive_effort` | `pub fn(&str, ThinkingEffort) -> ThinkingEffort` | `config.rs:205` | yes (`config.rs:191-204`) |
| `anthropic_efforts_for_model` | `pub fn(&str) -> (&'static [ThinkingEffort], Option<ThinkingEffort>)` | `config.rs:415-417` | yes (`config.rs:403-414`) |
| `normalize_effort_for_openai_route` | `pub fn(ThinkingEffort, &str) -> ThinkingEffort` | `config.rs:538` | yes (`config.rs:522-537`) |
| `normalize_effort_for_anthropic_route` | `pub fn(ThinkingEffort) -> Option<ThinkingEffort>` | `config.rs:563` | yes (`config.rs:552-562`) |
| `parse_thinking_effort` | `pub fn(Option<&str>) -> Result<Option<ThinkingEffort>, String>` | `config.rs:622` | one-liner (`config.rs:621`) |
| `MAX_PROMPT_BYTES` | `pub const usize` = 1 MiB | `config.rs:638` | no |
| `MAX_SYSTEM_PROMPT_BYTES` | `pub const usize` = 512 KiB | `config.rs:639` | no |
| `MAX_TOOL_RESULT_BYTES` | `pub const usize` = 8 MiB | `config.rs:643` | yes (`config.rs:640-642`) |
| `DEFAULT_TOOL_RESULT_TEXT_BYTES` | `pub const usize` = 50 KiB | `config.rs:649` | yes (`config.rs:644-648`) |
| `MAX_TOOL_CALLS_PER_TURN` | `pub const usize` = 64 | `config.rs:650` | no |
| `HANDOFF_MAX_OUTPUT_TOKENS` | `pub const u32` = 8192 | `config.rs:652` | no |
| `HANDOFF_ORIGINAL_TASK_MAX_BYTES` | `pub const usize` = 16 KiB | `config.rs:654` | no |
| `HANDOFF_MAX_TOOL_NAMES` | `pub const usize` = 20 | `config.rs:656` | no |
| `Provider` | `pub enum`, 4 variants | `config.rs:662-678` | variants 3-4 documented (`config.rs:665-677`), `Anthropic`/`OpenAi` not |
| `OpenAiApi` | `pub enum`, 3 variants | `config.rs:680-684` | yes (`config.rs:673-678`) |
| `Config` | `pub struct`, 26 `pub` fields | `config.rs:687-734` | 7 of 26 fields documented (`config.rs:701-733`); `provider`, `system_prompt`, `max_rounds`, `max_output_tokens`, `llm_timeout`, `tool_timeout`, `mcp_*`, `max_sessions`, `max_line_bytes`, `max_history_bytes`, `max_handoffs`, `max_parallel_tools`, `hook_timeout`, `api_key`, `model`, `base_url`, `anthropic_api_version` carry none |
| `Config::from_env` | `pub fn() -> Result<Self, String>` | `config.rs:736` | **no doc comment** |
| `Config::for_discovery` | `pub fn(Provider, String, String) -> Self` | `config.rs:838` | yes (`config.rs:830-837`) |
| `is_openai_host` | `pub fn(&str) -> bool` | `config.rs:1029` | yes (`config.rs:1025-1028`) |
| `HookServers` | `pub enum`, 3 variants | `config.rs:1055-1059` | yes (`config.rs:1049-1053`) |
| `HookServers::allows` | `pub fn(&self, &str) -> bool` | `config.rs:1063` | yes |
| `HookServers::is_disabled` | `pub fn(&self) -> bool` | `config.rs:1072` | yes |

`Config::validate` is **private** (`config.rs:870`), so an external caller who builds a `Config` literal (all fields are `pub`) or uses `for_discovery` has no way to run the invariant checks. That makes `Config` a public struct with un-runnable invariants from outside the crate.

#### Undocumented public items
- `Config::from_env` (`config.rs:736`) — the crate's primary entry point for configuration, with no doc comment at all. Contradicts the `AGENTS.md` rule "New public API must have doc comments".
- `PROTOCOL_VERSION` (`config.rs:3`), `MAX_PROMPT_BYTES` (`config.rs:638`), `MAX_SYSTEM_PROMPT_BYTES` (`config.rs:639`), `MAX_TOOL_CALLS_PER_TURN` (`config.rs:650`), and the three `HANDOFF_*` constants (`config.rs:652`, `config.rs:654`, `config.rs:656`) — all `pub`, none documented.
- 19 of the 26 `Config` fields carry no doc comment (`config.rs:688-700`, `config.rs:716-726`, `config.rs:729-730`).

#### Public items with no caller outside their own file
| Item | Only production caller | Note |
|---|---|---|
| `clamp_adaptive_effort` (`config.rs:205`) | `config.rs:166`, same file | `pub` with no cross-file use; grep across `crates/**/*.rs` excluding `config.rs` returned zero matches |
| `anthropic_efforts_for_model` (`config.rs:415`) | **none in production** — sole caller is the test helper at `config.rs:2588` | see below |
| `parse_thinking_effort` (`config.rs:622`) | `config.rs:826`, same file | `pub` with no cross-file use |
| `is_openai_host` (`config.rs:1029`) | `llm.rs:280` | crate-internal only, but `pub` |
| `DEFAULT_TOOL_RESULT_TEXT_BYTES` (`config.rs:649`) | `config.rs:818`, same file | `pub` with no cross-file use |
| `ThinkingEffort::anthropic_budget_tokens` (`config.rs:33`) | `config.rs:145`, same file | `pub` |
| `ThinkingEffort::anthropic_effort_str` (`config.rs:60`) | `config.rs:169`, same file | `pub` |

`anthropic_efforts_for_model` is the notable one. Its doc comment asserts (`config.rs:409-413`): "Both `anthropic_thinking_config` (request-time) and the effort-table UI ... must derive their behaviour from this helper so the two stay in sync." But `anthropic_thinking_config` (`config.rs:124-178`) does **not** call it — it calls `is_manual_budget_model` (`config.rs:136`), `is_adaptive_thinking_model` (`config.rs:161`), and `clamp_adaptive_effort` (`config.rs:166`) directly. The two functions duplicate the same three-way family classification independently. The doc is aspirational, not descriptive.

#### `llm.rs` items (nominally `pub`, actually crate-internal)
| Item | Signature | Site |
|---|---|---|
| `Llm` | `pub struct` with 3 private fields | `llm.rs:37-51` |
| `Llm::new` | `pub fn(&Config) -> Result<Self, AgentError>` | `llm.rs:53` |
| `Llm::complete` | `pub async fn(&self, &Config, &str, &[HistoryItem], &[ToolDef], &str) -> Result<LlmResponse, AgentError>` | `llm.rs:67-74` |
| `Llm::summarize` | `pub async fn(&self, &Config, &str, &str, u32, &str) -> Result<String, AgentError>` | `llm.rs:161-168` |
| `build_token_source` | `pub(crate) fn(&Config) -> Result<Arc<dyn TokenSource>, AgentError>` | `llm.rs:1175` |

`build_token_source` is the only correctly-scoped one (`pub(crate)`) and the only one with a cross-file caller: `catalog.rs:77` (imported at `catalog.rs:17`). `Llm::new` is called at `lib.rs:161`; `Llm::complete` at `agent.rs:124`.

`Llm::new` and `Llm::complete` have **no doc comments** (`llm.rs:53`, `llm.rs:67`); `Llm::summarize` likewise (`llm.rs:161`). Only the private helpers `openai_request` (`llm.rs:265-268`), `post_openai` (`llm.rs:320-325`), and `try_upgrade` (`llm.rs:370-371`) are documented.

#### Provider dispatch: enum + match, no trait
There is no provider trait. `Llm::complete` (`llm.rs:67-160`) dispatches on `cfg.provider` in a single `match` (`llm.rs:76-141`) with three arms — `Anthropic`, `OpenAi | Databricks` (shared), and `DatabricksV2`. `Llm::summarize` (`llm.rs:161-254`) repeats the identical three-arm structure. This matches the design claim in `crates/buzz-agent/README.md:187`: "`Provider` is a Rust `enum` with one `match` in `Llm::complete`. There is no trait, no `Box<dyn>`, no async-trait."

That claim is now half-true: `Llm::summarize` (`llm.rs:161`) contains a *second* full provider `match`, and there *is* a `Box<dyn>`-equivalent — `Arc<dyn TokenSource>` (`llm.rs:50`) with `#[async_trait]` methods, and adding a provider now means touching `build_token_source` (`llm.rs:1175-1204`) as well. The README's "Adding a provider is a `match` arm and one `body`/`parse` pair in `llm.rs`" (`README.md:187`) understates it by three sites.

Route selection helpers, all private to `llm.rs`:
| Helper | Purpose | Site |
|---|---|---|
| `databricks_v2_route_for_model` | model name → `DatabricksV2Route` | `llm.rs:685-697` |
| `databricks_v2_path` | route → static URL path | `llm.rs:698-704` |
| `is_responses_required_error` | provider error text → should auto-upgrade | `llm.rs:678-683` |
| `strip_model` | remove top-level `model` for legacy Databricks | `llm.rs:1206-1215` |
| `map_stop`, `sum_usage`, `str_field`, `make_tool_call` | shared parse helpers | `llm.rs:807`, `llm.rs:822`, `llm.rs:865`, `llm.rs:963` |
| `read_error_body`, `backoff_with_jitter`, `is_retryable_transport_error`, `terminal_llm_error`, `post` | HTTP layer | `llm.rs:983`, `llm.rs:1004`, `llm.rs:1026`, `llm.rs:1038`, `llm.rs:1052` |

#### CLI flags
**Neither file parses any CLI argument.** grep for `std::env::args`, `clap`, `argh`, `structopt` in `llm.rs` and `config.rs` returned zero matches. The only argument handling in the crate is `lib.rs:111-117`, which recognises a single `auth` subcommand. `Config::from_env` (`config.rs:736`) reads process environment exclusively.

`crates/buzz-agent/README.md:130` states "Everything is environment variables. No flags, no config files." That is accurate for this group but stale for the crate: `buzz-agent auth databricks` exists (`lib.rs:111-113`, handler `lib.rs:129-153`) and is documented nowhere in the README's Configuration section.

#### Test coverage for this aspect
Every public `config.rs` function has direct unit tests: `parse_thinking_effort` (`config.rs:1303`, `config.rs:1322`, `config.rs:1329`, `config.rs:1341`), `anthropic_thinking_config` (24 tests, `config.rs:1400-2189`), `clamp_adaptive_effort` (14 tests, `config.rs:1651-2140`), `normalize_effort_for_openai_route` (`config.rs:1988`, `config.rs:2413-2534`), `normalize_effort_for_anthropic_route` (`config.rs:2018`, `config.rs:2027`, `config.rs:2036`), `is_openai_host` (`config.rs:1267`), `HookServers::allows` (`config.rs:1168`, `config.rs:1176`, `config.rs:1181`), the three `ThinkingEffort` string/budget mappers (`config.rs:1351`, `config.rs:1364`, `config.rs:1375`).

Not covered:
- `Config::from_env` (`config.rs:736`) — zero tests; grep for `from_env()` inside the `config.rs` test module returned only two comment mentions (`config.rs:1248`, `config.rs:1834`).
- `Config::for_discovery` (`config.rs:838`) is exercised only as a fixture builder (`config.rs:1840`), never asserted on.
- `HookServers::is_disabled` (`config.rs:1072`) — grep for `is_disabled` in `config.rs` returned only the definition; no test.
- `Llm::new` (`llm.rs:53`) — no test constructs it; the test helper `llm_with` (`llm.rs:2689-2698`) bypasses it entirely and builds the struct literal with a *different* `reqwest::Client` configuration (`.timeout(5s)` instead of `.connect_timeout(10s).read_timeout(...)`), so `Llm::new`'s timeout wiring at `llm.rs:54-58` is never exercised.
- `Llm::complete` (`llm.rs:67`) and `Llm::summarize` (`llm.rs:161`) — no unit test calls either; tests exercise the body builders and `post_openai` separately. The provider `match` arms and the error-stamping `map_err` at `llm.rs:148-155` are untested at this level.
