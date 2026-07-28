## Module: buzz-agent — LLM providers & configuration (`crates/buzz-agent/src`)
### Aspect: Debt
The debt here is not rot — both files are actively maintained, densely tested, and free of the usual markers. `grep -rn 'TODO\|FIXME\|HACK\|XXX' llm.rs config.rs` returned **zero matches**, and `grep -n '#\[allow' llm.rs config.rs` returned zero matches, so there is no `#[allow(dead_code)]` standing in for a `TODO`. What is present instead is structural: duplicated classification logic, a test-only re-implementation that can mask drift, one wrong default in the docs, and a set of critical paths with no test.

#### Doc drift
| Claim | Doc site | Code | Severity |
|---|---|---|---|
| `BUZZ_AGENT_MAX_HISTORY_BYTES` default is `1048576` / "1 MiB" | `crates/buzz-agent/README.md:155`, repeated `README.md:236` | `16 * 1024 * 1024` at `config.rs:814` | 16× wrong; operator-visible |
| Nine env vars enumerated as the complete config surface | `README.md:128-157` | `BUZZ_AGENT_MODEL` (`config.rs:749`), `BUZZ_AGENT_MCP_INIT_TIMEOUT_SECS` (`config.rs:807`), `BUZZ_AGENT_MCP_RESTART_MAX_ATTEMPTS` (`config.rs:809`), `BUZZ_AGENT_MCP_RESTART_BASE_MS` (`config.rs:810`), `BUZZ_AGENT_MCP_RESTART_MAX_MS` (`config.rs:811`), `BUZZ_AGENT_HOOK_TIMEOUT_MS` (`config.rs:822`), `BUZZ_AGENT_STOP_MAX_REJECTIONS` (`config.rs:823`), `MCP_HOOK_SERVERS` (`config.rs:824`), `BUZZ_AGENT_NO_HINTS` (`config.rs:825`), `BUZZ_AGENT_THINKING_EFFORT` (`config.rs:826`) all missing | 10 undocumented vars, one of which (`MCP_HOOK_SERVERS`) gates a feature the README links to at `README.md:253` |
| "`Provider` is a Rust `enum` with one `match` in `Llm::complete`. There is no trait, no `Box<dyn>`, no async-trait." | `README.md:187` | a second provider `match` in `Llm::summarize` (`llm.rs:169-253`); `Arc<dyn TokenSource>` on `Llm` (`llm.rs:50`) with `#[async_trait]` methods | stale since the `TokenSource` refactor |
| "Adding a provider is a `match` arm and one `body`/`parse` pair in `llm.rs`." | `README.md:187` | requires touching `Llm::complete` (`llm.rs:76`), `Llm::summarize` (`llm.rs:169`), `build_token_source` (`llm.rs:1176`), and `from_env`'s provider arm (`config.rs:757`) | understates by three sites |
| "Everything is environment variables. No flags, no config files." | `README.md:130` | `buzz-agent auth <provider>` subcommand exists (`lib.rs:111-117`, handler `lib.rs:129-153`) | stale |
| Provider list: `anthropic`, `openai`, `databricks`, `databricks_v2` | `README.md:132` | also accepts `openai-compat` (`config.rs:995`) and `databricks-v2` (`config.rs:1000`) | undocumented aliases |
| `OPENAI_COMPAT_API` values `auto \| chat \| responses` | `README.md:140` | also accepts `chat-completions`, `chat_completions` (`config.rs:1015`) | undocumented aliases |
| MCP child env whitelist: `PATH, HOME, TERM, LANG, LC_ALL, TMPDIR` | `README.md:221` | also passes `XDG_CONFIG_HOME` (`mcp.rs:47`) | minor; sibling module |
| `ARCHITECTURE.md` describes system design | — | `grep -rni 'buzz.agent' ARCHITECTURE.md` returned zero matches — the agent surface is entirely absent from the top-level architecture doc | documentation gap, not a contradiction |

#### Stale in-code cross-references
`anthropic_efforts_for_model`'s doc comment asserts (`config.rs:407-413`):

> "This is the single production source of truth for Anthropic family routing. Both `anthropic_thinking_config` (request-time) and the effort-table UI (`valid_effort_values_for_provider_model`, via its Anthropic branch) must derive their behaviour from this helper so the two stay in sync."

Both halves of that are wrong:
1. `anthropic_thinking_config` (`config.rs:124-178`) never calls it. It classifies independently via `is_manual_budget_model` (`config.rs:136`), `is_adaptive_thinking_model` (`config.rs:161`), and `clamp_adaptive_effort` (`config.rs:166`).
2. `valid_effort_values_for_provider_model` is not "the effort-table UI" — it is a `#[cfg(test)]` function defined at `config.rs:2559`, inside `mod tests`.

The net effect: `anthropic_efforts_for_model` (`config.rs:415`) is a `pub fn` whose **only caller in the entire repo is a test** (`config.rs:2588`). `grep -rn 'anthropic_efforts_for_model' --include='*.rs' crates/` returns three hits: the doc mention (`config.rs:180`), the definition (`config.rs:415`), and the test-helper call (`config.rs:2588`).

Second stale reference: the comment at `config.rs:441-442` says "Reuse `anthropic_model_supports_xhigh` (the single source of truth shared with `clamp_adaptive_effort`) — no side-effects, no duplication." That is true for the xhigh predicate specifically, but sits three lines below a manual/adaptive classification (`config.rs:437-440`) that *is* duplicated with `anthropic_thinking_config`.

#### Duplicated classification logic
Four independent places classify a model name, with three different matching strategies:

| Site | Purpose | Strategy |
|---|---|---|
| `config.rs:136`, `config.rs:161` (inside `anthropic_thinking_config`) | pick thinking shape | prefix match after `strip_catalog_prefix` |
| `config.rs:437`, `config.rs:440` (inside `anthropic_efforts_for_model`) | pick capability set | same predicates, same order, independent call |
| `config.rs:367-392` (inside `openai_efforts_for_model`) | pick OpenAI effort table | boundary-safe `gpt5_token_matches` / `gpt5_base_matches` |
| `llm.rs:689-694` (inside `databricks_v2_route_for_model`) | pick gateway route | plain `contains("gpt-5")` / `contains("gpt5")` / `contains("claude")` |

The last one is the substantive defect. `config.rs` invested two purpose-built matchers (`config.rs:239-254`, `config.rs:267-311`) and eight boundary tests (`config.rs:2278-2399`) specifically to stop `gpt-5.1` matching `gpt-5.10`, `gpt-5-1` matching `gpt-5-1106`, and `gpt-5-4` matching `gpt-5-4o`. `llm.rs:689-690` uses naked `contains` for the routing decision in the same request path. Concretely: `databricks-gpt-5-10` routes to `/ai-gateway/openai/v1/responses` (`llm.rs:700`) while `openai_efforts_for_model` classifies it as unknown (asserted at `config.rs:2338`) and its effort passes through unverified. And `contains("claude")` is unanchored, so `my-claude-killer-llama` routes to the Anthropic Messages path. No test pairs the two classifiers — `databricks_v2_routes_by_model_family` (`llm.rs:1519-1542`) uses three unambiguous names only.

Third duplication: `strip_catalog_prefix` (`config.rs:89-97`) is re-implemented verbatim inside the test helper at `config.rs:2570-2582` rather than being called.

Fourth: `Llm::summarize` hand-builds an Anthropic body (`llm.rs:164-173`) duplicating `anthropic_body`'s top-level shape (`llm.rs:446-447`), and a Chat body (`llm.rs:192-203`) duplicating `openai_body`'s (`llm.rs:547-548`). A fix to either shape must be applied twice.

Fifth: the Databricks PKCE configuration is declared twice — canonically at `llm.rs:1185-1198` using `DATABRICKS_CLIENT_ID` (`llm.rs:19`) and `DATABRICKS_OAUTH_SCOPES` (`llm.rs:20`), and again as bare string literals in `lib.rs:135-143`. Because both constants are private to a private module, `lib.rs` structurally cannot reference them. A scope added to `DATABRICKS_OAUTH_SCOPES` would not be requested by `buzz-agent auth databricks`, producing a cached token the runtime then rejects. No test asserts the two agree.

#### Test that re-implements production rules (drift can go green)
`valid_effort_values_for_provider_model` (`config.rs:2559-2650`) is 92 lines of test-only logic mirroring the TypeScript `getProviderEffortConfig`. The fixture guard `effort_table_fixture_matches_rust_implementation` (`config.rs:2666-2700`) compares `effortTable.fixture.json` against *this function*, not against production code. The comment block at `config.rs:2537-2545` describes it as a drift guard that "fails CI here before it can silently diverge in production" — but three rules live only in the test:

1. **Prefix stripping** (`config.rs:2570-2582`) — a copy of `strip_catalog_prefix` (`config.rs:89-97`). If production's stripping changed, the fixture would still pass.
2. **Default-effort-per-family derivation** (`config.rs:2599-2607`): `GPT5_PRO → high`, `GPT5_1 → none`, else `medium`. This mapping exists nowhere in production Rust. The fixture's `defaultValue` column is therefore validated against a test-local table, not against anything the agent does.
3. **The DBv2 gpt-5 family predicate** (`config.rs:2623-2636`) — a copy of the chain at `config.rs:367-392` that **omits** the `gpt-5-6` / `gpt5-6` dash variants production checks at `config.rs:371-372`.

Item 3 is currently harmless only by accident: both the `is_gpt5` branch (`config.rs:2637-2639`) and the fall-through (`config.rs:2640-2643`) `return openai_result(&m)`, so the 14-line `is_gpt5` computation is **dead for every non-empty model name** — it only affects the blank-model case (`config.rs:2644-2646`). So the guard's most complex logic does nothing, and the omission it contains cannot be detected by any test.

There is also a hard compile-time coupling: `include_str!("../../../desktop/src/features/agents/ui/effortTable.fixture.json")` at `config.rs:2667-2668`. Moving or renaming that desktop file breaks `cargo test -p buzz-agent`, not just a test assertion.

#### Untested critical paths
Each verified by grepping the relevant test module and finding zero matches:

| Path | Site | Why it matters |
|---|---|---|
| `Config::from_env` | `config.rs:736-836` | 32 env vars, 10 error paths, the `BUZZ_AGENT_SYSTEM_PROMPT`/`_FILE` mutual exclusion (`config.rs:786-788`), the file-read error (`config.rs:790`), the `BUZZ_AGENT_NO_HINTS` `== 0` inversion (`config.rs:825`) — none exercised |
| The 13 numeric invariants in `validate` | `config.rs:876-931` | The only `validate()` tests (`config.rs:1859-1984`) vary `thinking_effort` alone. `make_config_for_validation` has to *undo* `for_discovery`'s invalid values (`config.rs:1849-1856`) to reach the effort check — direct evidence the numeric branches are unreached |
| `Llm::new` | `llm.rs:53-65` | The test helper `llm_with` (`llm.rs:2689-2698`) builds `Llm` by struct literal with `.timeout(5s)`, so the production `connect_timeout` + `read_timeout` wiring at `llm.rs:54-58` is never exercised — the tests run against a stricter client than production |
| `Llm::complete` | `llm.rs:67-160` | The provider `match` and the model-name error-stamping `map_err` at `llm.rs:148-155` are untested |
| `Llm::summarize` | `llm.rs:161-254` | Zero tests; 94 lines, three provider arms, three hand-built bodies |
| `try_upgrade` / the sticky latch | `llm.rs:372-390` | The *matcher* is well tested (`llm.rs:1503`), the *latch* is not — nothing asserts a second call skips Chat Completions or that the warn fires once |
| Both response-size caps | `llm.rs:1121-1128`, `llm.rs:1129-1141` | No test for `response too large` or `response exceeded` |
| `read_error_body` truncation | `llm.rs:983-998` | No test for the 4 KiB cap or the partial-chunk break at `llm.rs:989-991` |
| `strip_model` | `llm.rs:1206-1215` | Only caller is `llm.rs:339`; no test. The legacy-Databricks body rewrite is unverified |
| Chat-Completions malformed `arguments` | `llm.rs:941-944` | The Responses equivalent is tested (`llm.rs:1544`); this one is not |
| `make_tool_call` rejections | `llm.rs:964-966`, `llm.rs:971-975` | Empty id/name and non-object arguments — no direct test |
| Reasoning extraction, all three dialects | `llm.rs:764-780`, `llm.rs:879-886`, `llm.rs:922-932` | No test asserts a non-empty `LlmResponse.reasoning` |
| `ProviderStop::Refusal` and `::Other` | `llm.rs:810-812` | No test; `Refusal` is unreachable on the Responses path (`llm.rs:781-796` doesn't use `map_stop`) with no comment noting it |
| `HookServers::is_disabled` | `config.rs:1072` | No test |
| Body-read error non-retry | `llm.rs:1146-1152` | The request was already sent; the decision not to retry is uncommented and untested |
| `anthropic_api_version` on the DBv2 Anthropic route | `llm.rs:701` vs `llm.rs:356` | `post_openai` sends only `bearer_auth`, so no `anthropic-version` header reaches the gateway's Anthropic endpoint. Whether that is correct is unverified |

#### Production-path panics and rule violations
| Site | Issue |
|---|---|
| `config.rs:510` | `.expect("supported is non-empty")` in `resolve_openai_effort` — violates the `AGENTS.md` rule "Do not introduce new `unwrap()` or `expect()` in production paths". Safe only because every table at `config.rs:334-362` is a non-empty `const` |
| `llm.rs:1162` | `unreachable!()` at the end of `post` — reachable if `MAX_RETRIES` (`llm.rs:1000`) were ever set to 0. The invariant is asserted only in the panic message |

Both are low-probability, but both are the kind of implicit coupling that a refactor breaks silently.

#### Oversized files and functions
| Unit | Size | Ceiling context |
|---|---|---|
| `llm.rs` | 2,894 lines (~58% test) | `AGENTS.md` states a 1,000-line hard ceiling, but the enforcement (`justfile:123` desktop, `justfile:585` web, `justfile:617` mobile) is JS/TS/Dart only — there is no Rust file-size gate. So both files are ~2.7-2.9× the stated ceiling with no gate to trip. Neither is the repo's worst (`crates/buzz-acp/src/lib.rs` is 6,570 lines) |
| `config.rs` | 2,701 lines (~59% test) | same |
| `Config::from_env` | 101 lines (`config.rs:736-836`) | one struct literal with 26 initializers |
| `Llm::complete` | 94 lines (`llm.rs:67-160`) | single expression, three-arm `match`, nested closures |
| `Llm::summarize` | 94 lines (`llm.rs:161-254`) | near-mirror of `complete`, untested |
| `openai_efforts_for_model` | 69 lines (`config.rs:333-401`) | mostly a 26-term `else if` chain (`config.rs:367-392`) that would be better as a table |
| `valid_effort_values_for_provider_model` | 92 lines (`config.rs:2559-2650`) | test-only, includes dead logic |
| `post` | 112 lines (`llm.rs:1052-1163`) | retry + status mapping + streaming read in one function |

#### `pub` items with no cross-file caller
Not dead code (all are reachable), but `pub` beyond need, which enlarges the crate's committed API surface:

| Item | Site | Only caller |
|---|---|---|
| `anthropic_efforts_for_model` | `config.rs:415` | a test (`config.rs:2588`) |
| `clamp_adaptive_effort` | `config.rs:205` | `config.rs:166`, same file |
| `parse_thinking_effort` | `config.rs:622` | `config.rs:826`, same file |
| `DEFAULT_TOOL_RESULT_TEXT_BYTES` | `config.rs:649` | `config.rs:818`, same file |
| `ThinkingEffort::anthropic_budget_tokens` | `config.rs:33` | `config.rs:145`, same file |
| `ThinkingEffort::anthropic_effort_str` | `config.rs:60` | `config.rs:169`, same file |
| `is_openai_host` | `config.rs:1029` | `llm.rs:280` — crate-internal only |

`Llm`, `Llm::new`, `Llm::complete`, `Llm::summarize` are declared `pub` (`llm.rs:37`, `llm.rs:53`, `llm.rs:67`, `llm.rs:161`) inside a **private** module (`lib.rs:9`), so the `pub` keyword there is inert. `build_token_source` is the only correctly-scoped item (`pub(crate)`, `llm.rs:1175`).

#### Missing documentation on public API
`AGENTS.md` requires "New public API must have doc comments". Missing:
- `Config::from_env` (`config.rs:736`) — no doc comment at all, on the crate's primary configuration entry point.
- 19 of 26 `Config` fields (`config.rs:688-700`, `config.rs:716-726`, `config.rs:729-730`).
- `PROTOCOL_VERSION` (`config.rs:3`), `MAX_PROMPT_BYTES` (`config.rs:638`), `MAX_SYSTEM_PROMPT_BYTES` (`config.rs:639`), `MAX_TOOL_CALLS_PER_TURN` (`config.rs:650`), `HANDOFF_MAX_OUTPUT_TOKENS` (`config.rs:652`), `HANDOFF_ORIGINAL_TASK_MAX_BYTES` (`config.rs:654`), `HANDOFF_MAX_TOOL_NAMES` (`config.rs:656`).
- `Provider::Anthropic` and `Provider::OpenAi` variants (`config.rs:663-664`) — the two Databricks variants are documented (`config.rs:665-677`), the two originals are not.

The discipline is inverted: private helpers in `llm.rs` (`openai_request` `llm.rs:265-268`, `post_openai` `llm.rs:320-325`, `try_upgrade` `llm.rs:370-371`, `build_token_source` `llm.rs:1165-1174`) are all documented while the public methods are not.

#### Design fragility worth recording
1. **Implicit discriminant dependency.** `resolve_openai_effort` computes distance via `(e as i32 - requested as i32)` (`config.rs:487`), which requires `ThinkingEffort`'s variants to be contiguous 0..=6 in declaration order (`config.rs:19-27`). `thinking_effort_ord_ordering` (`config.rs:1387`) tests the `Ord` relation but not the cast values, so inserting a variant mid-enum would change every fallback distance while the test stayed green.
2. **A documented rule that is only emergent.** `config.rs:459` states "`xhigh` falls back to `high` when not supported (no model skips from `high` to `xhigh`)". Nothing implements that — it falls out of the sort at `config.rs:486-491`. A future table with `max` but not `xhigh` would resolve `xhigh` upward to `max`, contradicting the documented rule with all existing tests passing.
3. **Inert defaults that are not inert.** `openai_api` is hard-coded to `Chat` for both Databricks providers (`config.rs:781`) with the comment "only read by OpenAI/legacy Databricks dispatch". It *is* read for legacy Databricks (`llm.rs:279-280`, `llm.rs:294`) and the value permanently disables the chat→responses auto-upgrade there. Anthropic gets `Auto` instead (`config.rs:767`) for no stated reason.
4. **`for_discovery` produces a `Config` that would fail `validate`.** `max_output_tokens: 1` (`config.rs:846`), `mcp_max_restart_attempts: 0` (`config.rs:851`), `mcp_restart_base_ms: 0` (`config.rs:852`) are all rejected by `config.rs:917-925`, and `validate` is never called (`config.rs:838-868`). The odd `max_context_tokens: 200_001` (`config.rs:857`) exists only to satisfy a check that never runs. Nothing prevents this struct from being handed to code that assumes a validated config.
5. **`post_openai` discards `path` for legacy Databricks.** `llm.rs:333-341` rewrites the URL regardless of whether the caller asked for `/chat/completions` or `/responses`. Unreachable today (the forced `Chat` above blocks the upgrade), but the shape allows a Responses-body-to-Chat-URL mismatch if the latch were ever set.
6. **No wall-clock request timeout.** `Llm::new` sets only `connect_timeout` and `read_timeout` (`llm.rs:54-58`). `grep -n '\.timeout(' llm.rs` finds it only in tests (`llm.rs:2226`, `llm.rs:2290`, `llm.rs:2341`, `llm.rs:2692`). A slow-drip response is unbounded; `STALL_NOTICE_THRESHOLD` (`llm.rs:24`) only logs (`llm.rs:1039-1045`).
7. **No outbound body-size cap.** `serde_json::to_vec` at `llm.rs:1053` serializes any history, and the bytes are cloned per attempt (`llm.rs:1065`). All the relevant bounds are defined in `config.rs` but enforced in `agent.rs`/`handoff.rs`; `grep -n 'max_history_bytes' llm.rs` returns only the test fixture at `llm.rs:1237`.
8. **Unvalidated `mime_type` interpolation.** Tool-supplied `mime_type` (`types.rs:136`) goes straight into `data:{mime_type};base64,{data}` at `llm.rs:573` and `llm.rs:643` and into Anthropic's `source.media_type` at `llm.rs:468`, with no allowlist and no rejection of `;` or `,`.
9. **Test/production HTTP-client divergence.** Because `llm_with` (`llm.rs:2689-2698`) bypasses `Llm::new`, every socket-level test runs against `.timeout(5s)` while production runs with no total timeout. Any regression in `Llm::new`'s timeout configuration is invisible to the suite.
