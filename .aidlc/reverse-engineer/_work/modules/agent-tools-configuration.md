## Module: buzz-agent — MCP registry, OAuth, hints & catalog (`crates/buzz-agent/src`)
### Aspect: Configuration

Documentation columns below mean: **README** = `crates/buzz-agent/README.md` config table; **.env.example** = repo-root `.env.example`; **AGENTS.md** = repo-root `AGENTS.md`; **hooks doc** = `docs/MCP_DRIVEN_HOOKS.md`. Counts come from `grep -c '<VAR>' <file>` run per variable.

#### Environment read directly inside this group

| Variable | Read site | Purpose | Default / fallback | Documented |
|---|---|---|---|---|
| `HOME` | `hints.rs:11` (`home_dir`) | locate `~/AGENTS.md` and `~/.agents/skills` | `None` → global layer silently skipped (`hints.rs:55-60`, `hints.rs:212-214`) | no |
| `HOME` | `auth.rs:457-458` (`cache_path_for`) | OAuth cache root | none — hard error `oauth cache: $HOME not set` | no |
| every key in `PASSTHROUGH_ENV` | `mcp.rs:716` (`std::env::var(k)`) | forwarded into each MCP child after `env_clear()` | absent keys are simply not forwarded | partially, and inaccurately (see below) |
| `PASSTHROUGH_ENV_WINDOWS` + `WINDOWS_SHELL_RESOLUTION_ENV` | `mcp.rs:722` | Windows child temp/profile/shell resolution | as above | no |

The forwarded set (`mcp.rs:39-63`, plus `mcp.rs:70-71` and `lib.rs:19-30` on Windows):

| Key | Forwarded from | Notes |
|---|---|---|
| `PATH`, `HOME`, `TERM`, `LANG`, `LC_ALL`, `TMPDIR`, `XDG_CONFIG_HOME` | agent env | `XDG_CONFIG_HOME` is forwarded to children but **not** honoured by this crate's own cache path, which hardcodes `.config` (`auth.rs:460`) |
| `SSH_AUTH_SOCK`, `SSH_AGENT_PID` | agent env | git-over-SSH |
| `GIT_ASKPASS`, `GIT_SSH_COMMAND`, `GIT_CONFIG_GLOBAL` | agent env | operator git overrides |
| `NOSTR_PRIVATE_KEY`, `BUZZ_PRIVATE_KEY`, `BUZZ_RELAY_URL`, `BUZZ_AUTH_TAG` | agent env | `AGENTS.md:162-164` documents the last three as harness-injected; `NOSTR_PRIVATE_KEY` appears in neither `AGENTS.md` nor the crate README |
| `TMP`, `TEMP`, `USERPROFILE`, `APPDATA` (Windows) | agent env | needed because `std::env::temp_dir()` otherwise resolves to an unwritable location (`mcp.rs:66-69`) |
| `PATH`, `BUZZ_SHELL`, `GIT_BASH`, `SystemRoot`, `ProgramFiles`, `ProgramFiles(x86)`, `LOCALAPPDATA` (Windows) | agent env | contract shared with Doctor (`lib.rs:16-30`) |

Client-supplied `env` from the `session/new` payload is applied *after* the allowlist (`mcp.rs:726-728`), so a payload entry wins over an inherited value. That channel is per-server configuration, not process configuration, and is not covered by any env-var documentation.

#### Configuration consumed by this group but read in `config.rs`

| Variable | Read site (`config.rs`) | Default | Consumed here | README | .env.example | hooks doc |
|---|---|---|---|---|---|---|
| `BUZZ_AGENT_MCP_INIT_TIMEOUT_SECS` | `805-808` | 30 s | `mcp.rs:191` → `spawn_one` init + `tools/list` (`mcp.rs:757`, `:767`) | no (0) | no (0) | no |
| `BUZZ_AGENT_MCP_RESTART_MAX_ATTEMPTS` | `809` | 3 | `mcp.rs:188` (`.max(1)`), `mcp.rs:135`, `mcp.rs:303` | no (0) | no (0) | no |
| `BUZZ_AGENT_MCP_RESTART_BASE_MS` | `810` | 500 ms | `mcp.rs:189` (`.max(1)`) → `backoff` (`mcp.rs:813`) | no (0) | no (0) | no |
| `BUZZ_AGENT_MCP_RESTART_MAX_MS` | `811` | 30 000 ms | `mcp.rs:190` (`.max(1)`) → `backoff` cap | no (0) | no (0) | no |
| `BUZZ_AGENT_MAX_TOOL_RESULT_TEXT_BYTES` | `815-819` | 51 200 (`config.rs:649`) | `agent.rs:388` → `ResultBudget.text` (`mcp.rs:933-935`) | yes | no (0) | no |
| `BUZZ_AGENT_HOOK_TIMEOUT_MS` | `822` | 2 500 ms | `agent.rs:228`, `handoff.rs:78` → `mcp.rs:352` | no (0) | no (0) | yes, default matches |
| `BUZZ_AGENT_STOP_MAX_REJECTIONS` | `823` | 3 | `agent.rs:228-236` gate around `call_hooks` | no (0) | no (0) | yes, default matches |
| `MCP_HOOK_SERVERS` | `824` (parser `1083-1105`) | unset → `HookServers::None` | `mcp.rs:321-331` | no (0) | no (0) | yes, default matches |
| `BUZZ_AGENT_NO_HINTS` | `825` | `0` (hints on) | `lib.rs:355-360` → `hints::build_hints_section` | no (0) | no (0) | no |
| `BUZZ_AGENT_PROVIDER` | `739-743` | required | `catalog.rs:82-90` dispatch, `llm.rs:1176-1198` token-source choice | yes | no (0) | no |
| `DATABRICKS_HOST` | `737`, `782` | required for Databricks | `catalog.rs:81` (`cfg.base_url`), OAuth discovery URL (`llm.rs:1183-1186`) | yes | no (0) | no |
| `DATABRICKS_TOKEN` | `779` | empty → use PKCE | empty/non-empty selects `StaticTokenSource` vs `PkceOAuthTokenSource` (`llm.rs:1179-1197`) | yes | no (0) | no |
| `DATABRICKS_MODEL` | `738`, `781` | required for Databricks | `discovery_failure_fallback(provider, cfg.model)` (`catalog.rs:48-66`, `lib.rs:447-452`) | yes | no (0) | no |
| `BUZZ_AGENT_MODEL` | `748` | none | overrides `DATABRICKS_MODEL`, so it changes the fallback catalog | no (0) | no (0) | no |

`MCP_HOOK_SERVERS` semantics (`config.rs:1083-1105`): unset / empty / whitespace-only → `None` (hooks off); a lone `*` → `All`; otherwise a trimmed comma list → `Only([...])`. A mixed value like `*,foo` is deliberately *not* a wildcard — `*` cannot pass `valid_name` (`mcp.rs:859-864`) so it never matches a real server (comment `config.rs:1096-1099`, tests `config.rs:1128`, `config.rs:1176`).

#### CLI surface

| Invocation | Site | Behaviour |
|---|---|---|
| `buzz-agent` (no args) | `lib.rs:109-122` | runs the ACP stdio loop |
| `buzz-agent auth <provider>` | `lib.rs:111-116`, `lib.rs:126-153` | one-shot interactive OAuth; `databricks`, `databricks_v2`, `databricks-v2` accepted (`lib.rs:131`); requires `DATABRICKS_HOST` (`lib.rs:133-134`); prints the cache directory on success (`lib.rs:147`) |
| `buzz-agent auth` (no provider) | `lib.rs:151` | error: "provider required (try: buzz-agent auth databricks)" |
| `buzz-agent auth <other>` | `lib.rs:150` | error: `auth: unknown provider "<other>"` |
| anything else, e.g. `--help`, `--version` | `lib.rs:110-121` | **not** a flag — argv is only inspected for the literal `auth`, so any other argument falls through to the stdio loop and the process waits on stdin |

There is no flag parser and no `clap` dependency (`grep -n 'clap' crates/buzz-agent/Cargo.toml` → zero matches). This matches the README's stance ("Everything is environment variables. No flags, no config files") except that the README does not mention the `auth` subcommand at all.

The `auth` subcommand hardcodes its own OAuth parameters instead of reusing the constants the runtime uses: `client_id: "databricks-cli"` and `scopes: ["all-apis","offline_access"]` at `lib.rs:139-141` versus `DATABRICKS_CLIENT_ID` / `DATABRICKS_OAUTH_SCOPES` at `llm.rs:19-20`. Since the cache filename hashes those exact values (`auth.rs:446-454`), a divergence would send the CLI's token to a file the runtime never reads — a configuration coupling with no compile-time or test-time guard.

#### Configuration that exists in code but is not reachable in production

| Item | Site | Why |
|---|---|---|
| `PkceOAuthConfig::cache_dir_override` | `auth.rs:102-107` | documented as test-only; both production constructors pass `None` (`lib.rs:143`, `llm.rs:1194`). There is no env var to relocate the token cache |
| `PkceOAuthConfig::cache_namespace` | `auth.rs:101` | a real knob, but hardcoded to `"databricks"` at both sites (`lib.rs:142`, `llm.rs:1193`) |
| `MAX_NAME_LEN = 128` | `mcp.rs:20` | tighter `MAX_QNAME_LEN = 64` makes it unreachable for any name that produces a tool (`mcp.rs:242-248`) |
| `next_retry = now + 86_400s` on exhaustion | `mcp.rs:687-690` | never consulted; the exhausted branch errors earlier (`mcp.rs:135-140`) |

#### Hardcoded values that arguably should be configurable

| Value | Site | Impact |
|---|---|---|
| `MAX_MCP_SERVERS = 16`, `MAX_TOOLS_PER_SESSION = 128` | `mcp.rs:26`, `mcp.rs:22` | hard session-creation failure when exceeded; documented in the README's "Bounded Everything" table but not tunable |
| `MAX_HOOK_RESULT_BYTES = 16 KiB` | `mcp.rs:27` | per-hook output cap; not in the README or hooks doc |
| `MAX_HINTS_BYTES = 128 KiB` | `hints.rs:6` | hint-chain cap; undocumented anywhere |
| `MAX_SKILL_BODY_BYTES = 32 KiB` | `hints.rs:7` | `load_skill` output cap; undocumented anywhere |
| `SKILL_DIRS` | `hints.rs:8` | the three project skill directories and their precedence are not configurable and not documented |
| `TOKEN_REFRESH_LEEWAY = 60s`, `BROWSER_AUTH_TIMEOUT = 60s` | `auth.rs:35`, `auth.rs:39` | the browser window in particular is a UX-visible timeout with no override |
| 20-page / `page_size=100` catalog ceiling | `catalog.rs:199-203` | silently caps discovery at 2 000 endpoints |
| `DATABRICKS_V2_KNOWN_MODELS` | `catalog.rs:31-33` | the fallback model list is compiled in; operators cannot extend it |
| no HTTP timeouts for OAuth/catalog | `auth.rs:153`, `catalog.rs:80` | `Client::new()` means neither a default nor a configurable timeout, unlike `llm.rs:53-57` |

#### Documentation status summary

- Repo-root `.env.example` contains **no** agent configuration at all: `grep -c 'BUZZ_AGENT\|DATABRICKS\|MCP_HOOK' .env.example` → 0 (the file is scoped to the relay/backend, header at `.env.example:1-16`). `deploy/compose/.env.example` likewise → 0. So every variable in this group is undocumented in the canonical env template.
- The crate README documents provider/Databricks selection and the tool-result text cap, but none of the four `BUZZ_AGENT_MCP_*` knobs, none of the three hook knobs, and neither hints nor skills configuration.
- `docs/MCP_DRIVEN_HOOKS.md` is the only place the hook configuration is documented, and its three defaults (unset, 2500, 3) agree with `config.rs:822-824`.
- Independent evidence that the README's config table is not maintained in lockstep with the code: it lists `BUZZ_AGENT_MAX_HISTORY_BYTES` as `1048576` / "1 MiB" (`crates/buzz-agent/README.md:155`, repeated at `:236`) while `config.rs:815` defaults it to `16 * 1024 * 1024`. That variable is not consumed by this group, but it calibrates how much the "documented" column above can be trusted.

#### Test coverage — Configuration

Covered: `MCP_HOOK_SERVERS` parsing has twelve dedicated unit tests (`config.rs:1111`, `1116`, `1121`, `1129`, `1134`, `1142`, `1150`, `1158`, `1168`, `1176`, `1181`, `1186`); `BUZZ_AGENT_NO_HINTS` is covered end-to-end (`tests/hints_integration.rs:223 hints_suppressed_with_env_var`); `MCP_HOOK_SERVERS` reaching the registry is covered by six hook integration tests that set it on the agent process (`tests/regressions.rs:805`, `886`, `939`, `993`, `1133`, `1519`) plus one that deliberately leaves it unset (`:1044`); `BUZZ_AGENT_MCP_INIT_TIMEOUT_SECS` is exercised implicitly by `mcp_init_timeout_kills_child` (`tests/regressions.rs:307`).

Not covered: the three restart knobs (`grep -rn 'RESTART_MAX_ATTEMPTS\|RESTART_BASE_MS\|RESTART_MAX_MS' crates/buzz-agent/tests` → zero matches); `HOME`-absent behaviour for the OAuth cache (`auth.rs:457-458` returns an error no test triggers) versus `HOME`-absent behaviour for hints (covered, `hints.rs:523 load_hint_files_no_home_dir`); the `auth` CLI argument parsing (`grep -rn '"auth"' crates/buzz-agent/tests` → zero matches); and the claim that `cache_dir_override` is production-unused, which nothing enforces.
