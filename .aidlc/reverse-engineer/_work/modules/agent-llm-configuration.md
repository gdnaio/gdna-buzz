## Module: buzz-agent — LLM providers & configuration (`crates/buzz-agent/src`)
### Aspect: Configuration
`Config::from_env` (`config.rs:736-836`) is the crate's entire configuration surface: 32 environment variables, no config file, no CLI flag. Documentation lives only in `crates/buzz-agent/README.md:128-157`; **none of these variables appear in the repo-root `.env.example`, `README.md`, `AGENTS.md`, `ARCHITECTURE.md`, or `CONTRIBUTING.md`** — `grep -rn 'BUZZ_AGENT_' .env.example README.md AGENTS.md ARCHITECTURE.md CONTRIBUTING.md` returned zero matches, and `grep -n 'AGENT\|ANTHROPIC\|OPENAI\|DATABRICKS' deploy/compose/.env.example` returned zero matches for these vars.

#### CLI flags read by this group
**None.** `grep` for `std::env::args`, `clap`, `argh`, `structopt` in `llm.rs` and `config.rs` returned zero matches. The crate's only argument handling is the `auth` subcommand dispatch at `lib.rs:111-117`, outside this group.

#### Provider / credential variables
| Env var | Default | Read site | Validation | In crate README? |
|---|---|---|---|---|
| `BUZZ_AGENT_PROVIDER` | none — **required** | `config.rs:740` → `resolve_provider` `config.rs:982` | absent/blank → error (`config.rs:1007-1010`); accepts `anthropic`, `openai`, `openai-compat`, `databricks`, `databricks_v2`, `databricks-v2` case-insensitively (`config.rs:988-1002`) | yes (`README.md:132`) — but the `openai-compat` and `databricks-v2` aliases are **undocumented** |
| `ANTHROPIC_API_KEY` | none — required for `anthropic` | `config.rs:741`, `config.rs:759` | non-empty after trim (`config.rs:991`, `config.rs:978-980`) | yes (`README.md:133`) |
| `ANTHROPIC_MODEL` | none | `config.rs:762` | required unless `BUZZ_AGENT_MODEL` set (`config.rs:764`) | yes (`README.md:134`) |
| `ANTHROPIC_BASE_URL` | `https://api.anthropic.com` | `config.rs:765` | **none** — no scheme or host check | yes (`README.md:135`) |
| `ANTHROPIC_API_VERSION` | `2023-06-01` | `config.rs:799` | none | yes (`README.md:136`) |
| `OPENAI_COMPAT_API_KEY` | none — required for `openai` | `config.rs:742`, `config.rs:769` | non-empty after trim (`config.rs:995`) | yes (`README.md:137`) |
| `OPENAI_COMPAT_MODEL` | none | `config.rs:772` | required unless `BUZZ_AGENT_MODEL` set (`config.rs:774`) | yes (`README.md:138`) |
| `OPENAI_COMPAT_BASE_URL` | `https://api.openai.com/v1` | `config.rs:775` | **none** | yes (`README.md:139`) |
| `OPENAI_COMPAT_API` | `auto` | `config.rs:776` → `parse_openai_api` `config.rs:1014` | accepts `auto`, `""`, `chat`, `chat-completions`, `chat_completions`, `responses` (`config.rs:1015-1018`); anything else is an error | yes (`README.md:140`) — the `chat-completions`/`chat_completions` aliases are **undocumented** |
| `DATABRICKS_HOST` | none — required for both Databricks providers | `config.rs:737`, `config.rs:782` | presence only (`config.rs:782`); **no scheme check** | yes (`README.md:141`) |
| `DATABRICKS_MODEL` | none | `config.rs:738`, `config.rs:780` | required unless `BUZZ_AGENT_MODEL` set (`config.rs:781`) | yes (`README.md:142`) |
| `DATABRICKS_TOKEN` | `""` (empty ⇒ OAuth PKCE) | `config.rs:779` | none; emptiness is meaningful (`llm.rs:1181`) | yes (`README.md:143`) |
| `BUZZ_AGENT_MODEL` | none | `config.rs:749` | wins over the provider-specific model var (`config.rs:971-977`) | **NO** |

`BUZZ_AGENT_MODEL` is the highest-precedence model selector, is set programmatically by the persona layer (`crates/buzz-persona/src/resolve.rs:374`), is asserted in persona E2E tests (`crates/buzz-persona/tests/e2e_env_flow.rs:137-139`, `:308`), is surfaced in the desktop UI (`desktop/tests/e2e/edit-agent.spec.ts:8`), and appears in the benchmark harness docs (`benchmarks/harbor-buzz-orchestra/testbed/endpoints/README.md:19`) — yet it is missing from the one table that claims to enumerate the agent's configuration (`crates/buzz-agent/README.md:128-157`).

#### Prompt variables
| Env var | Default | Read site | Validation | In crate README? |
|---|---|---|---|---|
| `BUZZ_AGENT_SYSTEM_PROMPT` | `DEFAULT_SYSTEM_PROMPT` (`config.rs:658-659`) | `config.rs:785` | mutually exclusive with `_FILE` (`config.rs:786-788`) | yes (`README.md:144`) |
| `BUZZ_AGENT_SYSTEM_PROMPT_FILE` | none | `config.rs:785` | file read at startup; IO error → `config: read {p}: {e}` (`config.rs:790`) | yes (`README.md:145`) |

Neither is length-checked here. `MAX_SYSTEM_PROMPT_BYTES` (512 KiB, `config.rs:639`) is enforced in `lib.rs:375-383` against a per-session prompt override, **not** against the env-supplied prompt — so `BUZZ_AGENT_SYSTEM_PROMPT_FILE` pointing at a 100 MiB file is accepted at startup with no bound. `grep -n 'MAX_SYSTEM_PROMPT_BYTES' config.rs` returns only the definition at `config.rs:639`.

#### Numeric / duration variables
All go through `parse_env` (`config.rs:1041-1048`), which returns the default only when the variable is **absent**; a present-but-unparseable value is a hard startup error prefixed `config: {key}: `.

| Env var | Default (code) | Read site | Validation | README default | Match? |
|---|---|---|---|---|---|
| `BUZZ_AGENT_MAX_ROUNDS` | `0` | `config.rs:801` | none | `0` (`README.md:146`) | yes |
| `BUZZ_AGENT_MAX_OUTPUT_TOKENS` | `32_768` | `config.rs:802` | `>= 1` (`config.rs:876`); `< max_context_tokens` (`config.rs:879`) | `32768` (`README.md:147`) | yes |
| `BUZZ_AGENT_MAX_CONTEXT_TOKENS` | `200_000` | `config.rs:819` | `> max_output_tokens` (`config.rs:879`) | `200000` (`README.md:148`) | yes |
| `BUZZ_AGENT_MAX_HANDOFFS` | `10` | `config.rs:820` | none | `10` (`README.md:149`) | yes |
| `BUZZ_AGENT_LLM_TIMEOUT_SECS` | `240` | `config.rs:803` | `>= 1s` (`config.rs:908`) | `240` (`README.md:150`) | yes |
| `BUZZ_AGENT_TOOL_TIMEOUT_SECS` | `660` | `config.rs:804` | `>= 1s` (`config.rs:911`) | `660` (`README.md:151`) | yes |
| `BUZZ_AGENT_MAX_PARALLEL_TOOLS` | `8` | `config.rs:821` | `>= 1` (`config.rs:917`) | `8` (`README.md:152`) | yes |
| `BUZZ_AGENT_MAX_SESSIONS` | `usize::MAX` | `config.rs:812` | none | "unlimited" (`README.md:153`) | yes |
| `BUZZ_AGENT_MAX_LINE_BYTES` | `4 * 1024 * 1024` | `config.rs:813` | `>= 1024` (`config.rs:896`) | `4194304` (`README.md:154`) | yes |
| `BUZZ_AGENT_MAX_HISTORY_BYTES` | **`16 * 1024 * 1024`** | `config.rs:814` | `>= 4096` (`config.rs:885`) **and** `>= MAX_PROMPT_BYTES` (1 MiB, `config.rs:890`) | **`1048576` / "1 MiB"** (`README.md:155`, repeated `README.md:236`) | **NO — 16× discrepancy** |
| `BUZZ_AGENT_MAX_TOOL_RESULT_TEXT_BYTES` | `DEFAULT_TOOL_RESULT_TEXT_BYTES` = 50 KiB (`config.rs:649`) | `config.rs:816` | `1024 ..= MAX_TOOL_RESULT_BYTES` (8 MiB) (`config.rs:901-907`) | `51200` (`README.md:156`) | yes |
| `BUZZ_AGENT_MCP_INIT_TIMEOUT_SECS` | `30` | `config.rs:807` | `>= 1s` (`config.rs:914`) | — | **NO** |
| `BUZZ_AGENT_MCP_RESTART_MAX_ATTEMPTS` | `3` | `config.rs:809` | `>= 1` (`config.rs:920`) | — | **NO** |
| `BUZZ_AGENT_MCP_RESTART_BASE_MS` | `500` | `config.rs:810` | `>= 1` (`config.rs:923`) | — | **NO** |
| `BUZZ_AGENT_MCP_RESTART_MAX_MS` | `30_000` | `config.rs:811` | `>= mcp_restart_base_ms` (`config.rs:926`) | — | **NO** |
| `BUZZ_AGENT_HOOK_TIMEOUT_MS` | `2500` | `config.rs:822` | none | — | **NO** |
| `BUZZ_AGENT_STOP_MAX_REJECTIONS` | `3` | `config.rs:823` | none; `0` disables `_Stop` hooks (`config.rs:719-721`) | — | **NO** |

The `BUZZ_AGENT_MAX_HISTORY_BYTES` discrepancy is the sharpest doc contradiction in this group. Code: `16 * 1024 * 1024` at `config.rs:814`. README states `1048576` with the note "1 MiB. Old turns are evicted past this" at `README.md:155`, and repeats "History window | 1 MiB" in the Bounded Everything table at `README.md:236`. The code-side default is also confirmed by the `for_discovery` fixture (`config.rs:855`) and both test fixtures (`llm.rs:1238`, which sets `max_history_bytes: 16 * 1024 * 1024`). An operator reading the README would size their history budget 16× too small.

#### Non-numeric behavioural variables
| Env var | Default | Read site | Parse rules | In crate README? |
|---|---|---|---|---|
| `MCP_HOOK_SERVERS` | unset ⇒ `HookServers::None` (hooks off) | `config.rs:824` → `parse_hook_servers_env` `config.rs:1077` → `parse_hook_servers` `config.rs:1083` | comma-separated; entries trimmed, empties dropped (`config.rs:1088-1093`); all-empty ⇒ `None` (`config.rs:1094-1096`); lone `*` ⇒ `All` (`config.rs:1103-1105`); `*,foo` ⇒ literal `Only(["*","foo"])` (`config.rs:1098-1102`) | **NO** |
| `BUZZ_AGENT_NO_HINTS` | `0` ⇒ hints enabled | `config.rs:825` | parsed as `u8`; `hints_enabled = value == 0`, so any non-zero disables; a non-numeric value (`true`, `yes`) is a **startup error** | **NO** |
| `BUZZ_AGENT_THINKING_EFFORT` | unset ⇒ `None` (no thinking config sent) | `config.rs:826` → `parse_thinking_effort` `config.rs:622` | trimmed, lowercased; accepts `none`, `minimal`, `low`, `medium`, `high`, `xhigh`, `max`; `""` and whitespace-only ⇒ `None`; anything else is an error naming the value and the legal set (`config.rs:633-636`); Anthropic additionally rejects `none`/`minimal` at startup (`config.rs:939-949`) | **NO** |

`MCP_HOOK_SERVERS` is the odd one out on naming — it is the only variable in `from_env` without the `BUZZ_AGENT_` prefix (`config.rs:824`). It is also the gate for the entire MCP-driven-hooks feature that `README.md:253` links to (`docs/MCP_DRIVEN_HOOKS.md`), and it is used in six integration tests (`crates/buzz-agent/tests/regressions.rs:805`, `:886`, `:939`, `:993`, `:1133`, `:1519`) — a documented feature whose only enabling switch is absent from the configuration table.

`BUZZ_AGENT_THINKING_EFFORT` is the most consequential undocumented variable in the crate. It has its own 13-line type doc (`config.rs:5-17`), five dedicated warning messages that name it back to the operator (`config.rs:153`, `config.rs:223`, `config.rs:516`, `config.rs:546`, `config.rs:571`), roughly 70 unit tests across `config.rs` and `llm.rs`, a desktop UI with a shared JSON fixture (`desktop/src/features/agents/ui/effortTable.fixture.json`), a desktop-specific guidance note (`desktop/src/features/agents/AGENTS.md:35`), and is set explicitly in a relay example (`crates/buzz-relay/examples/mesh_agent_e2e.rs:281`). It appears nowhere in `crates/buzz-agent/README.md`.

`BUZZ_AGENT_NO_HINTS` is documented only implicitly, via the integration test at `crates/buzz-agent/tests/hints_integration.rs:230`.

#### Summary of documentation gaps
Nine of the 32 variables read by `from_env` are absent from the crate README's configuration table (`README.md:128-157`):

`BUZZ_AGENT_MODEL` (`config.rs:749`), `BUZZ_AGENT_MCP_INIT_TIMEOUT_SECS` (`config.rs:807`), `BUZZ_AGENT_MCP_RESTART_MAX_ATTEMPTS` (`config.rs:809`), `BUZZ_AGENT_MCP_RESTART_BASE_MS` (`config.rs:810`), `BUZZ_AGENT_MCP_RESTART_MAX_MS` (`config.rs:811`), `BUZZ_AGENT_HOOK_TIMEOUT_MS` (`config.rs:822`), `BUZZ_AGENT_STOP_MAX_REJECTIONS` (`config.rs:823`), `MCP_HOOK_SERVERS` (`config.rs:824`), `BUZZ_AGENT_NO_HINTS` (`config.rs:825`), `BUZZ_AGENT_THINKING_EFFORT` (`config.rs:826`).

Plus one wrong default (`BUZZ_AGENT_MAX_HISTORY_BYTES`) and three undocumented aliases (`openai-compat`, `databricks-v2`, `chat-completions`/`chat_completions`).

#### Parsed-but-never-read configuration
| Field | Parsed at | Read anywhere? |
|---|---|---|
| `Config.model` | `config.rs:757-783` | Yes, but **not** by `llm.rs` — every body builder takes an explicit `effective_model: &str` parameter (`llm.rs:396`, `llm.rs:481`, `llm.rs:594`) and the `Config.model` field is never read there. `grep -n 'cfg.model' llm.rs` returned zero matches. The field is consumed by the session layer (`wire.rs:85` documents `model_id` overriding `BUZZ_AGENT_MODEL`), which resolves the per-session effective model. Two tests explicitly assert that `effective_model` wins over `cfg.model` (`llm.rs:1892`, `llm.rs:1905`). |
| `Config.openai_api` for `Provider::Anthropic` | hard-coded `Auto` at `config.rs:767` | Not read on the Anthropic path (`llm.rs:77-89`) — genuinely inert, and commented as such |
| `Config.openai_api` for `Provider::Databricks` | hard-coded `Chat` at `config.rs:781`, commented "only read by OpenAI/legacy Databricks dispatch" | **Is** read, at `llm.rs:279-280` and `llm.rs:294`. Hard-coding `Chat` permanently disables the chat→responses auto-upgrade for legacy Databricks. That is a behavioural decision expressed as an inert default with no comment saying so. |
| `Config.anthropic_api_version` | `config.rs:799` | Read only on the Anthropic path (`llm.rs:260`). For `Provider::DatabricksV2`'s Anthropic Messages route (`llm.rs:701`) it is **never sent** — `post_openai` applies only `bearer_auth` (`llm.rs:356`), so the gateway's Anthropic endpoint receives no `anthropic-version` header. Whether the gateway requires one is unverified; no test covers it. |

Fields validated in `config.rs` but enforced elsewhere (not a defect, just a boundary worth recording): `max_rounds`, `max_history_bytes`, `max_line_bytes`, `max_tool_result_text_bytes`, `max_context_tokens`, `max_handoffs`, `max_parallel_tools`, `tool_timeout`, `hook_timeout`, `stop_max_rejections`, `hook_servers`, `hints_enabled`, `max_sessions`, and the four `mcp_*` fields are all read by `agent.rs`, `mcp.rs`, `handoff.rs`, `hints.rs`, or `lib.rs`. Of `Config`'s 26 fields, `llm.rs` reads exactly five: `provider`, `base_url`, `api_key`, `anthropic_api_version`, `openai_api`, `max_output_tokens`, `thinking_effort`, and `llm_timeout` (eight, verified by grepping `cfg\.` in `llm.rs` lines 1-1216).

#### Deprecated aliases
No variable is marked deprecated and there is no deprecation-warning path — `grep -ni 'deprecat' config.rs llm.rs` returned zero matches. The provider aliases (`openai-compat` at `config.rs:995`, `databricks-v2` at `config.rs:1000`) and the `OPENAI_COMPAT_API` aliases (`chat-completions`, `chat_completions` at `config.rs:1015`) are accepted silently and equally, with no indication of which form is canonical.

#### Test coverage for this aspect
Covered:
- `parse_openai_api` including the empty-string, whitespace, and mixed-case forms, plus the error path (`config.rs:1198-1214`).
- `resolve_provider` for present/missing keys, absent provider, unsupported value with casing preserved (`config.rs:1216-1264`).
- `resolve_model` all three precedence cases (`config.rs:1285`, `config.rs:1291`, `config.rs:1297`).
- `parse_thinking_effort` round-trip, empty/whitespace, case-insensitivity, and the error message content (`config.rs:1303-1349`).
- `parse_hook_servers` — ten tests covering unset, empty, whitespace-only, `*`, `*` with padding, named list, trimming, the mixed `*,foo` case, and `allows()` semantics (`config.rs:1111-1196`).
- `validate`'s thinking-effort rule for all four providers × 7 levels (`config.rs:1859-1984`).

Not covered:
- **`Config::from_env` has zero tests.** No test reads or mutates process env; every fixture is a struct literal (`llm.rs:1225-1256`) or a patched `for_discovery` (`config.rs:1836-1857`). So none of the 32 defaults, none of the required-variable error messages, the `BUZZ_AGENT_SYSTEM_PROMPT` / `_FILE` mutual exclusion (`config.rs:786-788`), the file-read error path (`config.rs:790`), or the `BUZZ_AGENT_NO_HINTS` `== 0` inversion (`config.rs:825`) is verified in a unit test. The defaults are only exercised indirectly by the integration harness in `crates/buzz-agent/tests/`.
- The 13 numeric/duration invariants at `config.rs:876-931` are never asserted — the only `validate()` tests (`config.rs:1859-1984`) construct a config that already satisfies them via `make_config_for_validation` (`config.rs:1836-1857`) and vary only `thinking_effort`. That helper even has to *undo* `for_discovery`'s invalid values (`config.rs:1849-1856`) to reach the effort check, which is direct evidence the numeric branches are unexercised.
- `HookServers::is_disabled` (`config.rs:1072`) — no test.
