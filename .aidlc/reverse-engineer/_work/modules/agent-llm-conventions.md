## Module: buzz-agent — LLM providers & configuration (`crates/buzz-agent/src`)
### Aspect: Conventions
Both files follow a consistent, deliberate house style: no `unsafe`, no lint suppressions, dense explanatory comments with dated doc-source citations, and error handling split cleanly between `String` (config) and a typed enum (runtime).

#### `unsafe` and lint attributes
Zero `unsafe` in either file — `grep -c 'unsafe' llm.rs config.rs` returned `llm.rs:0` and `config.rs:0`. This satisfies the `AGENTS.md` "No `unsafe` code" rule.

Zero lint suppressions — `grep -n '#\[allow' llm.rs config.rs` returned no matches. So there is no `#[allow(dead_code)]` standing in for a `TODO`, which is a genuine positive: the several `pub` items with no cross-file callers (see API Surface) are left honestly `pub` rather than papered over.

The only attributes present are `#[derive(...)]` on the six types, `#[cfg(test)]` on the two test modules (`llm.rs:1217`, `config.rs:1106`), `#[test]`/`#[tokio::test]` on test functions, and `#[async_trait::async_trait]` on one test impl (`llm.rs:2618`).

#### `unwrap` / `expect` / panic discipline
`AGENTS.md` states: "Do not introduce new `unwrap()` or `expect()` in production paths — use `?` and proper error types." Scanning only the pre-test regions (`llm.rs` lines 1-1216, `config.rs` lines 1-1105):

| Site | Construct | Assessment |
|---|---|---|
| `config.rs:510` | `.expect("supported is non-empty")` in `resolve_openai_effort` | **violates the rule.** Safe in practice because every table at `config.rs:334-362` is a non-empty `const`, but it is a genuine `expect()` on a production path |
| `llm.rs:1162` | `unreachable!("loop always returns on its final iteration…")` in `post` | production-path panic; reachable only if `MAX_RETRIES` (`llm.rs:1000`) were set to 0 |
| `llm.rs:518`, `llm.rs:617` | `serde_json::to_string(...).unwrap_or_else(\|_\| "{}".into())` | infallible-fallback, no panic |
| `llm.rs:742`, `llm.rs:866`, `llm.rs:894`, `llm.rs:941` | `.unwrap_or(...)` on `Option` | no panic |
| `config.rs:779`, `config.rs:959`, `config.rs:1015`, `config.rs:1037`, `config.rs:1047` | `.unwrap_or*` | no panic |

So the rule is broken exactly once (`config.rs:510`) and there is one additional deliberate panic (`llm.rs:1162`). Test code uses `unwrap`/`expect` freely, which is conventional.

#### Error handling style
Two distinct styles, split by phase:

| Phase | Error type | Convention | Site |
|---|---|---|---|
| Configuration | `Result<_, String>` | every message prefixed `config: ` and names the offending env var verbatim | `config.rs:877`, `config.rs:881`, `config.rs:887`, `config.rs:892`, `config.rs:898`, `config.rs:905`, `config.rs:909`, `config.rs:912`, `config.rs:915`, `config.rs:918`, `config.rs:921`, `config.rs:924`, `config.rs:928`, `config.rs:944`, `config.rs:963`, `config.rs:1045` |
| Runtime | `AgentError` (`types.rs:213-220`) | three LLM-specific variants: `Llm`, `LlmAuth`, `LlmModelNotFound` | constructed at `llm.rs:58`, `llm.rs:745`, `llm.rs:915`, `llm.rs:964`, `llm.rs:1046`, `llm.rs:1087`, `llm.rs:1109`, `llm.rs:1115`, `llm.rs:1124`, `llm.rs:1140`, `llm.rs:1161` |

The `config: ` prefix convention is enforced by nothing but is applied consistently at all 16 sites, and four tests assert on it by substring (`config.rs:1213`, `config.rs:1230`, `config.rs:1240`, `config.rs:1343`).

One notable convention in `AgentError` handling: `Llm::complete` funnels every provider arm through a single `map_err` that prepends `(model-name) ` to the inner string (`llm.rs:148-155`), so log lines read `llm: (gpt-5.5) 404 Not Found: …`. The rationale — "This is the single place all provider paths converge, so the mapping is centralized" — is spelled out at `llm.rs:142-147`. `Llm::summarize` does **not** do this (`llm.rs:161-254` has no `map_err`), so summarizer failures lose the model name. Nothing flags the asymmetry.

Status-code → error-variant mapping is centralized in `post` (`llm.rs:1086-1119`) and each branch carries a comment explaining whether it is a "stall path" (`llm.rs:1080-1085`, `llm.rs:1106-1107`).

#### Logging targets and levels
Both files log exclusively at `WARN`. There are no `error!`, `info!`, `debug!`, or `trace!` calls — grep for `tracing::error`, `tracing::info`, `tracing::debug`, `tracing::trace` in both files returned zero matches.

| Site | Event | Structured fields |
|---|---|---|
| `llm.rs:381-386` | sticky Responses upgrade, once per process | `provider_message` |
| `llm.rs:1040-1044` | cumulative stall past 300 s | `cumulative_stall`, `attempts` |
| `llm.rs:1071-1076` | transport error, retrying | `attempt`, `max_attempts`, `error` |
| `llm.rs:1100-1105` | retryable status, retrying | `attempt`, `max_attempts`, `status` |
| `config.rs:149-154` | thinking budget omitted, `max_output_tokens` too small | `max_output_tokens`, `level_budget`, `headroom` |
| `config.rs:219-224` | adaptive effort clamped | `model`, `requested`, `clamped` |
| `config.rs:512-518` | OpenAI effort substituted | `model`, `requested`, `resolved` |
| `config.rs:542-546` | `max` clamped to `xhigh` on unknown model | `requested`, `resolved` |
| `config.rs:566-573` | `none`/`minimal` not expressible for Anthropic | `requested` |

All nine use structured key-value fields rather than string interpolation alone, which is consistent. Four of the five `config.rs` warnings name the env var `BUZZ_AGENT_THINKING_EFFORT` in the message text (`config.rs:153`, `config.rs:223`, `config.rs:516`, `config.rs:546`, `config.rs:571`) — a good operator-facing convention, tying a runtime warning back to the knob that caused it.

No explicit `target:` is set on any event, so all nine inherit the module path target (`buzz_agent::llm`, `buzz_agent::config`). The subscriber is configured once in `lib.rs:155-158` to write to stderr with ANSI disabled — consistent with the "all logs go to stderr" claim in `crates/buzz-agent/README.md:220`.

#### Naming
Consistent patterns:
- Body builders: `<dialect>_body` — `anthropic_body` (`llm.rs:391`), `openai_body` (`llm.rs:476`), `responses_body` (`llm.rs:589`).
- Parsers: `parse_<dialect>` — `parse_anthropic` (`llm.rs:869`), `parse_openai` (`llm.rs:912`), `parse_responses` (`llm.rs:706`).
- Family predicates: `is_<property>_model` — `is_manual_budget_model` (`config.rs:586`), `is_adaptive_thinking_model` (`config.rs:603`), plus `anthropic_model_supports_xhigh` (`config.rs:184`) which breaks the pattern.
- Env parsers are pure and take `Option<&str>` so they are testable without env mutation, and each says so in its doc comment: `parse_thinking_effort` (`config.rs:621`), `parse_openai_api` (`config.rs:1012-1013`), `parse_hook_servers` (`config.rs:1080-1081`). This is a strong convention — the impure wrappers are separated (`parse_hook_servers_env`, `config.rs:1077`).

Two naming defects:
- `type OpenAiParse` (`llm.rs:28`) is used for the Anthropic parser on the DBv2 route (`llm.rs:126`), so the name lies at one call site.
- `openai_body` builds a Chat Completions body while `responses_body` builds a Responses body — the asymmetry means "openai" means "chat" in one identifier and "the whole family" in `openai_request` (`llm.rs:269`) and `post_openai` (`llm.rs:326`), which handles Databricks too.

#### Doc-comment discipline
`config.rs` is unusually well documented for its function bodies: every effort-related function carries a multi-paragraph doc comment with a dated citation to the vendor documentation it encodes — `config.rs:100-123` (Anthropic extended-thinking table), `config.rs:191-204` (Anthropic effort page), `config.rs:313-332` (OpenAI model pages, with an embedded markdown table), `config.rs:578-585` and `config.rs:591-602` (both citing `platform.claude.com` URLs), `config.rs:256-266` (the `gpt5_base_matches` acceptance rules enumerated case by case). Several include worked examples (`config.rs:82-88`).

The gap is on the *types*: `Config::from_env` (`config.rs:736`) has no doc comment at all, and 19 of `Config`'s 26 fields are undocumented (see API Surface). `Llm`'s three public methods — `new` (`llm.rs:53`), `complete` (`llm.rs:67`), `summarize` (`llm.rs:161`) — likewise carry none, while its *private* helpers `openai_request` (`llm.rs:265-268`), `post_openai` (`llm.rs:320-325`), `try_upgrade` (`llm.rs:370-371`), and `build_token_source` (`llm.rs:1165-1174`) are all documented. The discipline is inverted relative to the `AGENTS.md` rule "New public API must have doc comments".

`llm.rs` also uses inline block comments to record hard-won provider behaviour rather than doc comments — e.g. the OpenAI-Chat image batching rationale (`llm.rs:478-486`), the Responses replay invariant (`llm.rs:582-587`), the 401/403 refresh rationale (`llm.rs:343-351`), the 499/5xx stall-path annotations (`llm.rs:1080-1085`, `llm.rs:1106-1107`). Two doc comments reference GitHub issue numbers (`llm.rs:1402` cites "#559/#560"), which is the only cross-reference style used and appears only in test comments.

#### File-size discipline
| File | Lines | Test-module share |
|---|---|---|
| `llm.rs` | 2,894 | tests start at `llm.rs:1217` → ~58% test code |
| `config.rs` | 2,701 | tests start at `config.rs:1106` → ~59% test code |

`AGENTS.md` states a "Hard ceiling: **1000 lines/file**". Read in context, that ceiling is scoped to the mobile app ("Keep widgets small and composable… enforced by `mobile/scripts/check-file-sizes.mjs` via `just mobile-check` (runs in `just check` + pre-push, mirroring desktop/web)"). The enforcement is JS/TS/Dart only: `justfile:123` runs `pnpm check:file-sizes` for desktop, `justfile:585` for web, `justfile:617` runs `node ./scripts/check-file-sizes.mjs` for mobile. There is **no** Rust equivalent — grep for `check-file-sizes` in `justfile` returned only those three lines, and no `crates/**` script exists. So both files are ~2.7-2.9× the stated ceiling but are not in violation of any enforced gate, and they are not the worst offenders in the repo (`crates/buzz-acp/src/lib.rs` is 6,570 lines).

Function-level size is where the real pressure is. `Llm::complete` spans `llm.rs:67-160` (94 lines) and `Llm::summarize` spans `llm.rs:161-254` (94 lines), each a single expression built from a three-arm `match` with nested closures. `Config::from_env` spans `config.rs:736-836` (101 lines) as one struct literal with 26 initializers. `openai_efforts_for_model` spans `config.rs:333-401` (69 lines), most of it a 26-term `else if` chain (`config.rs:367-392`). The test-only `valid_effort_values_for_provider_model` is 92 lines (`config.rs:2559-2650`).

#### Test organization
Both files use a single in-file `#[cfg(test)] mod tests` with `use super::*` (`llm.rs:1217-1219`, `config.rs:1106-1108`). Conventions observed:
- Section dividers as `// ---- <topic> ----` comments: `llm.rs:1662`, `llm.rs:2009`, `llm.rs:2502`, `llm.rs:2826`; `config.rs:1398`, `config.rs:1648`, `config.rs:1747`, `config.rs:1832`, `config.rs:1986`, `config.rs:2016`, `config.rs:2053`, `config.rs:2191`, `config.rs:2276`, `config.rs:2537`.
- Shared fixture builders at the top of each module: `cfg(provider)` (`llm.rs:1225`), `cfg_responses()` (`llm.rs:1300`), `image_history()` (`llm.rs:1257`), `tool_call_history()` (`llm.rs:1306`); `make_config_for_validation` (`config.rs:1836`).
- Table-driven tests where the input space is enumerable: `is_responses_required_error_matrix` (`llm.rs:1503`), `databricks_v2_routes_by_model_family` (`llm.rs:1519`), `parse_openai_api_values` (`config.rs:1198`), `is_openai_host_matrix` (`config.rs:1267`), `clamp_adaptive_effort_low_medium_high_never_clamped` (`config.rs:1723`).
- Every assertion carries a message naming the invariant, frequently with the offending input interpolated: `llm.rs:1349`, `llm.rs:1613`, `config.rs:1741`, `config.rs:2219`.
- Test names encode the expected behaviour, not the function under test alone: `anthropic_body_omits_thinking_when_max_output_too_small` (`llm.rs:1701`), `openai_efforts_for_model_boundary_gpt5_4o_is_base_not_5_4` (`config.rs:2296`).
- Two tests are explicitly written to be mutation-sensitive and say so in their doc comments: `terminal_llm_error_below_threshold_emits_no_stall_warning` (`llm.rs:2473-2480`) and `terminal_llm_error_at_threshold_emits_one_stall_warning` (`llm.rs:2486-2493`).

Integration-style tests that need a real socket use `tokio::net::TcpListener` bound to `127.0.0.1:0` with a hand-rolled HTTP responder rather than a mock-HTTP crate — five instances (`llm.rs:2177`, `llm.rs:2242`, `llm.rs:2306`, `llm.rs:2634`, reused by `llm.rs:2704`/`2740`/`2768`/`2796`). The responder logic is duplicated across the first three (each re-writes the "read to `\r\n\r\n`, then write a 200 with a tiny JSON body" loop) while the auth tests share one factory, `spawn_auth_stub` (`llm.rs:2634-2688`).

Two conventions are violated in the test module:
- `llm_with` (`llm.rs:2689-2698`) constructs `Llm` by struct literal, bypassing `Llm::new` and using a *different* client configuration (`.timeout(5s)` instead of `connect_timeout` + `read_timeout`). The production timeout wiring at `llm.rs:54-58` is therefore never exercised.
- `valid_effort_values_for_provider_model` (`config.rs:2559-2650`) is a test-local re-implementation of production routing rather than a call into production code. See Debt.

#### Comment-quality note
Comments in this group are unusually forthcoming about *why* rather than *what*, and several record negative knowledge that would otherwise be lost — e.g. why an empty Anthropic assistant turn is skipped instead of placeholder-filled (`llm.rs:418-422`), why images are batched behind the tool-result run (`llm.rs:478-486`), why `sum_usage` distinguishes "no usage reported" from "usage was zero" (`llm.rs:823-828`), why a mixed `*,foo` hook list is not treated as a wildcard (`config.rs:1098-1102`). Two carry dated provenance ("doc-verified, July 2025") which makes staleness auditable: `config.rs:11`, `config.rs:101`, `config.rs:194`, `config.rs:315`, `config.rs:578`, `config.rs:591`, `config.rs:616`, `config.rs:619`.
