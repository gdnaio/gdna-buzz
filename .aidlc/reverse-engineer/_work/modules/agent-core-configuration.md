## Module: buzz-agent — core loop, ACP wire types & handoff (`crates/buzz-agent/src`)
### Aspect: Configuration
#### How this group gets configured
All environment parsing lives in `config.rs` (a sibling agent's scope); this group receives an immutable `Config` built once at startup (`lib.rs:160`) and stored in `App.cfg` (`lib.rs:53`). The only direct env read in the group is `DATABRICKS_HOST` inside the auth subcommand (`lib.rs:135-136`) — `grep -c 'std::env::var'` returns 1 for `lib.rs` and 0 for the other five files. There are no CLI flags: `run()` recognizes exactly one argument, `auth` (`lib.rs:111`), and ignores everything else (`lib.rs:117-120`).

#### Config fields consumed in this group
| Field | Env var | Default (site) | Read site(s) in this group |
|---|---|---|---|
| `max_line_bytes` | `BUZZ_AGENT_MAX_LINE_BYTES` | 4 MiB (`config.rs:813`) | `lib.rs:162` → `read_loop`/`read_bounded_line` (`wire.rs:189`) |
| `max_sessions` | `BUZZ_AGENT_MAX_SESSIONS` | `usize::MAX` (`config.rs:812`) | `lib.rs:349`, `lib.rs:401` |
| `hints_enabled` | `BUZZ_AGENT_NO_HINTS` (inverted: `0` → enabled) | enabled (`config.rs:825`) | `lib.rs:356` |
| `system_prompt` | `BUZZ_AGENT_SYSTEM_PROMPT` / `_FILE` | built-in default (`config.rs:658`, resolved `config.rs:786-792`) | `lib.rs:365` (fallback only) |
| `model` | `BUZZ_AGENT_MODEL` > provider-specific var | required (`config.rs:749`, `config.rs:757-785`) | `lib.rs:470`, `lib.rs:672` |
| `provider` | `BUZZ_AGENT_PROVIDER` | required, no inference (`config.rs:982`, `config.rs:1005-1008`) | `lib.rs:445` |
| `max_rounds` | `BUZZ_AGENT_MAX_ROUNDS` | `0` = unlimited (`config.rs:801`) | `agent.rs:88` |
| `max_history_bytes` | `BUZZ_AGENT_MAX_HISTORY_BYTES` | **16 MiB** (`config.rs:814`) | `agent.rs:111`, `handoff.rs:125` |
| `max_parallel_tools` | `BUZZ_AGENT_MAX_PARALLEL_TOOLS` | 8 (`config.rs:821`) | `agent.rs:371` |
| `tool_timeout` | `BUZZ_AGENT_TOOL_TIMEOUT_SECS` | 660 s (`config.rs:804`) | `agent.rs:383`, `agent.rs:509` |
| `max_tool_result_text_bytes` | `BUZZ_AGENT_MAX_TOOL_RESULT_TEXT_BYTES` | 50 KiB (`config.rs:815-818`) | `agent.rs:387` |
| `stop_max_rejections` | `BUZZ_AGENT_STOP_MAX_REJECTIONS` | 3 (`config.rs:823`) | `agent.rs:220` |
| `hook_timeout` | `BUZZ_AGENT_HOOK_TIMEOUT_MS` | 2500 ms (`config.rs:822`) | `agent.rs:228`, `handoff.rs:75` |
| `hook_servers` | `MCP_HOOK_SERVERS` | `None` = hooks off (`config.rs:824`) | `agent.rs:229`, `handoff.rs:76` |
| `max_context_tokens` | `BUZZ_AGENT_MAX_CONTEXT_TOKENS` | 200 000 (`config.rs:819`) | `handoff.rs:113`, `handoff.rs:123`, `handoff.rs:196` |
| `max_output_tokens` | `BUZZ_AGENT_MAX_OUTPUT_TOKENS` | 32 768 (`config.rs:802`) | `handoff.rs:113`, `handoff.rs:124` |
| `max_handoffs` | `BUZZ_AGENT_MAX_HANDOFFS` | 10 (`config.rs:820`) | `handoff.rs:34-38` |
| `llm_timeout`, `mcp_*`, `thinking_effort`, `api_key`, `base_url`, `openai_api`, `anthropic_api_version` | various | — | not read here; passed through `cfg` to `llm`/`mcp` |

`DATABRICKS_HOST` is read directly at `lib.rs:135-136` (auth subcommand only) and is documented in the crate README (`README.md:145`).

#### Compile-time constants that behave like configuration
| Constant | Value | Site | Tunable? |
|---|---|---|---|
| `PROTOCOL_VERSION` | 2 | `config.rs:3`, used `lib.rs:284` | no |
| `MAX_PROMPT_BYTES` | 1 MiB | `config.rs:638`, used `agent.rs:69` | no |
| `MAX_SYSTEM_PROMPT_BYTES` | 512 KiB | `config.rs:639`, used `lib.rs:375` | no |
| `MAX_TOOL_RESULT_BYTES` | 8 MiB | `config.rs:643`, used `agent.rs:386` | no |
| `MAX_TOOL_CALLS_PER_TURN` | 64 | `config.rs:650`, used `agent.rs:242` | no |
| `HANDOFF_MAX_OUTPUT_TOKENS` | 8192 | `config.rs:652`, used `handoff.rs:51`, `handoff.rs:197` | no |
| `HANDOFF_ORIGINAL_TASK_MAX_BYTES` | 16 KiB | `config.rs:654`, used `handoff.rs:175` | no |
| `HANDOFF_MAX_TOOL_NAMES` | 20 | `config.rs:656`, used `handoff.rs:182` | no |
| `IMAGE_CONTEXT_TOKEN_EQUIV` | 16 KiB | `types.rs:14` | no |
| `CONSERVATIVE_BYTES_PER_TOKEN` | 1 | `handoff.rs:326` | no |
| keepalive interval | 30 s | `agent.rs:129` (literal) | no |
| post-cancel drain bound | 5 s | `agent.rs:455` (literal) | no |
| wire channel depth | 64 | `lib.rs:164` (literal) | no |
| session/run/steer token width | 8 bytes | `lib.rs:822` (literal) | no |
| `usage_update.contextLimit` | `0` | `lib.rs:742` (literal) | no |

The keepalive interval is the most consequential hardcoded value: it exists to keep the harness idle clock alive, and the harness side *is* configurable (`.env.example:169-170` recommends raising the ACP idle timeout to 60 s), so the two can be tuned out of alignment with no way to adjust this side.

#### Documentation status of every env var this group depends on
`.env.example` documents zero of them: `grep -c 'BUZZ_AGENT' .env.example` → 0 (the file covers relay and `BUZZ_ACP_*` harness config only, `.env.example:114-170`). The authoritative reference is the crate README table (`crates/buzz-agent/README.md:130-158`).

| Env var | crate README | `docs/MCP_DRIVEN_HOOKS.md` | root `.env.example` | AGENTS.md |
|---|---|---|---|---|
| `BUZZ_AGENT_PROVIDER` | yes (`:132`) | — | no | no |
| `BUZZ_AGENT_MAX_ROUNDS` | yes (`:149`) | — | no | no |
| `BUZZ_AGENT_MAX_OUTPUT_TOKENS` | yes (`:150`) | — | no | no |
| `BUZZ_AGENT_MAX_CONTEXT_TOKENS` | yes (`:151`) | — | no | no |
| `BUZZ_AGENT_MAX_HANDOFFS` | yes (`:152`) | — | no | no |
| `BUZZ_AGENT_TOOL_TIMEOUT_SECS` | yes (`:154`) | — | no | no |
| `BUZZ_AGENT_MAX_PARALLEL_TOOLS` | yes (`:155`) | — | no | no |
| `BUZZ_AGENT_MAX_SESSIONS` | yes (`:156`) | — | no | no |
| `BUZZ_AGENT_MAX_LINE_BYTES` | yes (`:157`) | — | no | no |
| `BUZZ_AGENT_MAX_HISTORY_BYTES` | yes but **wrong default** (`:158`) | — | no | no |
| `BUZZ_AGENT_MAX_TOOL_RESULT_TEXT_BYTES` | yes (`:159`) | — | no | no |
| `BUZZ_AGENT_SYSTEM_PROMPT` / `_FILE` | yes (`:147-148`) | — | no | no |
| `MCP_HOOK_SERVERS` | no | yes (`:61`) | no | no |
| `BUZZ_AGENT_HOOK_TIMEOUT_MS` | no | yes (`:62`) | no | no |
| `BUZZ_AGENT_STOP_MAX_REJECTIONS` | no | yes (`:63`) | no | no |
| `BUZZ_AGENT_NO_HINTS` | no | no | no | no |
| `BUZZ_AGENT_MODEL` | no | no | no | no |
| `BUZZ_AGENT_THINKING_EFFORT` | no | no | no | no |
| `BUZZ_AGENT_MCP_INIT_TIMEOUT_SECS` | no | no | no | no |
| `BUZZ_AGENT_MCP_RESTART_MAX_ATTEMPTS` / `_BASE_MS` / `_MAX_MS` | no | no | no | no |

Documented default that contradicts the code: README says `BUZZ_AGENT_MAX_HISTORY_BYTES` defaults to `1048576` / "1 MiB" (`crates/buzz-agent/README.md:158`, repeated in the limits table at `:236`), but the code default is `16 * 1024 * 1024` (`config.rs:814`). Anyone sizing a deployment from the README is off by 16×. `BUZZ_AGENT_NO_HINTS` is undocumented anywhere yet is the only switch for the hints/skills injection this group performs at `lib.rs:356-359` (it *is* exercised by `hints_suppressed_with_env_var`, `tests/hints_integration.rs:223`).

#### Parsed but never read
- `InitializeParams::_client_capabilities` — deserialized with `#[serde(default)]` and discarded (`wire.rs:41-42`). The agent advertises capabilities but never adapts to the client's.
- `negotiated_version` — computed as `min(client, 2)` and echoed, never stored or consulted (`lib.rs:284-290`). The `[Base]` note at `lib.rs:255-259` implies behavior *should* depend on it; that dependency lives in buzz-acp (`crates/buzz-acp/src/pool.rs:181`) instead.
- `SessionNewParams` tolerates and drops unknown fields by design (test `session_new_params_ignores_unknown_fields`, `wire.rs:270`).

#### Config-driven behavior with no test coverage
`max_sessions` (both rejection paths, `lib.rs:346-355` and `lib.rs:399-409`) has no test — `grep -n 'max sessions reached' tests/` returns zero matches. `max_rounds` reaching the cap (`agent.rs:88-90`, returning `max_turn_requests`) likewise has no test in `crates/buzz-agent/tests/` (`grep -n 'MAX_ROUNDS\|max_turn_requests' tests/` → 0 matches). By contrast the handoff-related settings are covered by both unit tests (`handoff.rs:378-428`) and integration tests (`tests/regressions.rs:1224-1512`).
