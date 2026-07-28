## Module: buzz-agent — MCP registry, OAuth, hints & catalog (`crates/buzz-agent/src`)
### Aspect: Features

#### Feature inventory

| Feature | Entry point | Limits / caveats |
|---|---|---|
| Host stdio MCP servers per session | `McpRegistry::spawn_all` (`mcp.rs:172`) | 16 servers, 128 tools, stdio only; spawn is serial, any single failure fails `session/new` (`mcp.rs:217`) |
| Merge all servers into one namespaced tool catalog | `mcp.rs:230-261`, `tools()` (`mcp.rs:286`) | `server__tool`, `__` banned in both halves, qname ≤64 bytes |
| Fan tool calls out to the owning child | `McpRegistry::call` (`mcp.rs:485`) | exact-match routing; tool-set drift and non-object arguments rejected locally (`mcp.rs:500-506`, `mcp.rs:566-575`) |
| Bounded, lossy-but-marked tool results | `tool_result_content` (`mcp.rs:913`) | 8 MiB total / 50 KiB text by default; text middle-elided, images kept whole or marker-replaced |
| Process-group kill of child + grandchildren | `process_group(0)` (`mcp.rs:733`) + `killpg` (`mcp.rs:845`) | unix only; on non-unix the "kill" is a log line and reliance on `Drop` (`mcp.rs:855-857`) |
| Lazy restart of a dead server with backoff | `maybe_restart` (`mcp.rs:646`) | 3 attempts, 500 ms→30 s ±20%; only *consecutive spawn failures* are budgeted |
| Lifecycle hooks (`_Stop`, `_PostCompact`) | `call_hooks` (`mcp.rs:315`) | off by default, fail-open, 2.5 s per hook, hidden from the LLM, killed after 2 consecutive timeouts |
| MCP cancellation propagation | `fire_and_forget_cancel` (`mcp.rs:788`) | best-effort `notifications/cancelled`, never blocks the agent |
| Interactive browser OAuth 2.0 PKCE login | `interactive_login` (`auth.rs:235`), CLI at `lib.rs:126-153` | Databricks only; needs a browser and a 60 s window |
| Silent token cache + refresh | `PkceOAuthTokenSource::bearer` (`auth.rs:246`) | on-disk JSON keyed by `sha256(discovery_url\|client_id\|scopes)`; cross-process re-read |
| Headless bearer resolution (no browser) | `try_bearer_no_browser` (`auth.rs:367`) | returns `LlmAuth` with a "run `buzz-agent auth databricks`" hint instead of prompting |
| 401-driven force refresh | `refresh_now` (`auth.rs:317`) | coalesces on token identity; terminal on refresh failure, never falls to the browser |
| `AGENTS.md` hint layering into the system prompt | `build_hints_section` (`hints.rs:219`) | `$HOME` + git-root→cwd chain, 128 KiB total |
| Multi-vendor skill discovery | `discover_skills_impl` (`hints.rs:204`) | `.agents/skills`, `.goose/skills`, `.claude/skills`, `~/.agents/skills`; first name wins |
| Lazy skill loading (`load_skill`) | `builtin.rs:16`, `builtin.rs:41` | in-process, no MCP round-trip; 32 KiB per call; only offered when ≥1 skill exists (`agent.rs:117-119`) |
| Supporting-file loading within a skill | `load_supporting_file` (`builtin.rs:118`) | pre-enumerated allowlist + canonicalise containment check |
| Databricks model-catalog discovery | `discover_databricks_models` (`catalog.rs:76`) | v1 filtered by READY+chat/completions, v2 paginated to 20 pages, known-model fallback |

#### stdio MCP hosting

Each `session/new` receives an `mcpServers` array (`types.rs:McpServerStdio`) and gets its own registry — sessions do not share children (`lib.rs:390-393`). The child is launched with the session's `cwd` as its working directory (`mcp.rs:729`) and inherits the agent's stderr (`mcp.rs:730`), so child diagnostics land in the agent's log stream unfiltered. Windows child processes get `CREATE_NO_WINDOW` so a GUI-launched agent does not flash console windows (`mcp.rs:992-1003`).

Limits worth stating plainly: only the child-process transport is wired (`rmcp` is pulled in with `features = ["client", "transport-child-process"]`, `Cargo.toml`), and the agent advertises `mcpCapabilities: { http: false, sse: false }` at `lib.rs:294`. There is no support for server-initiated MCP traffic because the client is built from the unit type (`mcp.rs:83`) — including no handling of `notifications/tools/list_changed` (`grep -rn 'list_changed\|sampling\|roots' crates/buzz-agent/src` → zero matches).

#### Tool fan-out and result shaping

The registry is the single tool namespace the LLM sees; `agent.rs:116-119` appends the built-in `load_skill` on top. Results are shaped for a context window rather than for fidelity: text is middle-elided so both "what ran" and "how it ended" survive (`mcp.rs:880-885`), images pass whole because providers bill them as tiles (`mcp.rs:28-32`), and everything else (audio, embedded resources) degrades to a one-line marker (`mcp.rs:974-985`). Every elision leaves an inline marker, so the model can tell it is looking at a truncated result — except for the two `load_skill` caps, which cut silently (`builtin.rs:102-105`, `builtin.rs:206-209`).

#### Process-group kill

`process_group(0)` is set on the `tokio::process::Command` before spawn (`mcp.rs:732-733`), and every teardown path calls `killpg(SIGKILL)`: `Drop for Server` (`mcp.rs:116-122`), spawn abandonment (`PgidGuard`, `mcp.rs:741-756`), explicit kill (`mcp.rs:445-447`), and transport failure (`mcp.rs:464-466`). The crate README claims this is done "via `setpgid(0,0)` in `pre_exec`" — the code uses the safe `Command::process_group` API instead, which matters because `lib.rs:1` is `#![forbid(unsafe_code)]` and a `pre_exec` implementation would not compile.

On non-unix targets the "kill" is a no-op that logs "relying on Drop to kill MCP …" (`mcp.rs:854-857`), so the grandchild guarantee is unix-only. The fake MCP server supports `FAKE_MCP_SPAWN_GRANDCHILD` (`tests/bin/fake_mcp.rs:17`, `:228`) but no test uses it — `grep -rn 'FAKE_MCP_SPAWN_GRANDCHILD' --include='*.rs' .` matches only those two lines in the fake itself.

#### Lifecycle hooks

Delivered capability: any MCP server can expose `_`-prefixed tools; the agent calls them at two lifecycle points (`_Stop` before honouring `end_turn`, `agent.rs:224-236`; `_PostCompact` after a context handoff, `handoff.rs:73-92`) and injects non-empty responses back into the conversation. The registry contributes the discovery, allowlisting, parallel dispatch, per-hook timeout, deterministic ordering, and the kill-on-second-timeout escalation (`mcp.rs:315-419`).

Limits: hooks are advisory (dropped on any error), invisible to the model, opt-in per server via `MCP_HOOK_SERVERS`, and non-cancellable by design (`mcp.rs:342-347`). Hook results are capped at 16 KiB each but the *number* of hook results is not capped (see Business Rules).

#### Browser OAuth login with token caching

`buzz-agent auth databricks` (`lib.rs:126-153`) runs the flow once and prints where the token landed (`lib.rs:147`). The runtime path is silent: `Llm` asks for a bearer per request (`llm.rs:49`), the source serves it from memory, then disk, then a refresh grant, and only then opens a browser (`auth.rs:246-297`). Discovery-only paths use the no-browser variant so a headless harness degrades instead of hanging (`auth.rs:299-301`, `auth.rs:367-423`).

Limits: single provider shape (RFC 8414 discovery + public-client PKCE), no device-code or client-credentials flow (`grep -n 'device_code\|client_credentials' auth.rs` → zero matches), no logout/revocation (`grep -n 'revoke' auth.rs` → zero matches), no way to point the cache at a different directory in production (`cache_dir_override` is documented as test-only, `auth.rs:102-106`), and a hard `$HOME` dependency — no `$HOME`, no OAuth (`auth.rs:457-458`).

#### Hints and skills injection

Two independent capabilities behind one entry point (`hints.rs:219-221`), both gated by `BUZZ_AGENT_NO_HINTS` (`config.rs:825`, applied at `lib.rs:355-360`):

- project hints: the `AGENTS.md` of `$HOME` plus every directory from the git root down to `cwd`, concatenated most-general-first so the closest file has the last word (`hints.rs:40-84`);
- skills: name + description only, with an instruction to call `load_skill` for the body (`hints.rs:239-247`).

The lazy-loading design is deliberate and asserted: bodies must not appear in the system prompt (`hints.rs:479-495`). Limits: discovery happens once per session with no invalidation; skill descriptions are unbounded and land in the system prompt verbatim; the combined prompt is validated against a 512 KiB ceiling by the caller, and exceeding it fails `session/new` outright rather than truncating (`lib.rs:375-388`, constant at `config.rs:639`).

#### `load_skill`

Two request forms, one tool (`builtin.rs:16-39`). Both read from the blocking pool so a large file cannot stall a Tokio worker (`builtin.rs:68-71`, `builtin.rs:192`, `builtin.rs:197`), and error results are model-readable: they enumerate available skill names or available relative paths (`builtin.rs:60-65`, `builtin.rs:151-172`). Supporting files are advertised inside the loaded body with a copy-pasteable call form (`builtin.rs:88-98`).

Limits: 32 KiB per call with a silent head cut; no directory listing tool (the model can only see what discovery enumerated); a skill whose name contains `/` is unreachable in the plain form (`builtin.rs:52`); and symlinked supporting files are listed but refused at load time (interaction of `builtin.rs:143-149` with `builtin.rs:196`).

#### Databricks model discovery

Used in two places: `session/new` advertises `availableModels` so a UI model picker and `session/set_model` can work (`lib.rs:440-472`), and the desktop resolves a model list without spawning the agent (`desktop/src-tauri/src/commands/agent_models.rs:709-758`). Successful results are cached process-wide in a `OnceCell`; failures are not cached, so the next session retries (`lib.rs:312-329`).

Limits: Databricks only (`catalog.rs:82-90`); no timeout on the HTTP calls (`Client::new()` at `catalog.rs:80` versus the configured client at `llm.rs:53-57`); v2 stops silently at 2 000 endpoints (`catalog.rs:199-200`); v1 can legitimately return an empty list despite the doc promising non-empty (`catalog.rs:70` vs `catalog.rs:129`); and for non-Databricks providers the caller does not call this at all — it reports only the configured model (`lib.rs:455-457`).

#### Documented-but-absent / undocumented features

| Claim | Where | Reality |
|---|---|---|
| MCP child env whitelist is "`PATH`, `HOME`, `TERM`, `LANG`, `LC_ALL`, `TMPDIR`" | crate README, Security Model table | the allowlist has 17 entries including `SSH_AUTH_SOCK`, `GIT_*`, `NOSTR_PRIVATE_KEY`, `BUZZ_PRIVATE_KEY`, `BUZZ_AUTH_TAG` (`mcp.rs:39-63`) |
| process group established "via `setpgid(0,0)` in `pre_exec`" | crate README, Security Model table | `Command::process_group(0)` (`mcp.rs:733`); no `unsafe`, none possible under `#![forbid(unsafe_code)]` (`lib.rs:1`) |
| hints / skills / `load_skill` | — | not mentioned anywhere in `crates/buzz-agent/README.md` (`grep -cn 'load_skill\|skills\|AGENTS.md' crates/buzz-agent/README.md` → 0); the only repo-level trace is `CHANGELOG.md:738` |
| OAuth / `buzz-agent auth` subcommand | — | the README mentions OAuth twice — a quick-start comment (README:53) and the `DATABRICKS_TOKEN` row (README:143, "If unset, Databricks uses browser OAuth + refresh cache") — but never documents the `auth` subcommand, the 60 s browser window, or the cache path |
| model discovery / `availableModels` | — | undocumented in the crate README; the ACP transcript there still shows a `session/new` result with only `sessionId` (README "ACP Transcript" section) |

#### Test coverage — Features

End-to-end coverage exists for hosting (`tests/regressions.rs:241 init_session`, `:746 init_session_with_fake_mcp`), caps (`:354`, `:420`, `:606`, `:681`), init timeout (`:307`), hooks (`:787`-`:1112`, `:1514`), cancellation propagation (`:1573`, `:1710`), hints/skills/`load_skill` (`tests/hints_integration.rs:193`, `223`, `254`, `300`, `343`, `385`, `469`, `517`), and the OAuth cache/refresh feature set (`tests/databricks_oauth.rs:105`-`:305`).

No test exercises: process-group kill of grandchildren, restart-after-death as a user-visible feature, `interactive_login`, the model-discovery feature end to end (only the pure fallback helper is tested, `lib.rs:882`-`:950`), or the Windows-specific behaviours — both Windows tests are `#[cfg(windows)]` (`mcp.rs:1016-1033`, `mcp.rs:1127-1140`) and cannot run on the macOS/Linux CI paths this repo builds; the non-Windows counterpart asserts only "didn't crash" (`mcp.rs:1118-1125`).
