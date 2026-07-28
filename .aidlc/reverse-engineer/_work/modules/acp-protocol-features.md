## Module: buzz-acp — ACP protocol, config & setup mode (`crates/buzz-acp/src`)
### Aspect: Features

#### ACP client capabilities — implemented

| Feature | Evidence |
|---|---|
| Spawn an agent binary and speak NDJSON JSON-RPC 2.0 over its stdio | `acp.rs:408-497` |
| Protocol handshake with capability advertisement | `initialize` (`acp.rs:539-544`), `build_client_capabilities` (`acp.rs:347-368`) |
| Adapter-owned login flows | `authenticate` (`acp.rs:549-554`), driven by `run_authenticate` (`lib.rs:3947`) and `run_auth_methods` (`lib.rs:3899`) |
| Session creation with MCP server declarations and an optional system prompt | `session_new_full` (`acp.rs:563-588`) |
| Multi-block prompts (slash-command pass-through) | `session_prompt_blocks_with_idle_timeout` (`acp.rs:676-745`); rationale at `acp.rs:672-675` — connectors detect commands on the **first** block, so the harness sends `["/cmd args", "<buzz context>"]` |
| Dual-deadline turn supervision (idle + absolute) | `acp.rs:1198-1523` |
| Cooperative cancellation with permission cleanup and response draining | `cancel_with_cleanup` (`acp.rs:837`), `cancel_with_cleanup_until` (`acp.rs:897`) |
| Bounded Stop-button cancellation with a distinct error | `cancel_with_cleanup_grace` (`acp.rs:881-895`) |
| Automatic tool-permission approval | `handle_permission_request` (`acp.rs:1680-1755`) |
| Stable + unstable model switching with precedence | `resolve_model_switch_method` (`acp.rs:1876-1920`), `session_set_config_option` (`acp.rs:623`), `session_set_model` (`acp.rs:638`) |
| Goose-native non-cancelling mid-turn steer | read-loop steer arm (`acp.rs:1279-1358`), `build_steer_params` (`acp.rs:1791-1805`) |
| Goose token-usage ingestion | `handle_goose_usage_update` (`acp.rs:1637-1678`), `take_turn_usage` (`acp.rs:783`) |
| Raw-wire observer feed for the desktop app | `observe` (`acp.rs:524-533`), emitted at `acp.rs:963` (write), `acp.rs:1120`/`acp.rs:1414` (read), `acp.rs:1105`/`acp.rs:1399` (parse errors) |
| Process-group teardown | `kill_process_group` (`acp.rs:1979-1987`) used by `shutdown` (`acp.rs:384-388`) and `Drop` (`acp.rs:1956-1962`) |
| Windows console-window suppression | `configure_no_window` (`acp.rs:1997-2006`) |
| Codex Seatbelt network widening | `codex_network_env` (`config.rs:646-677`) + `build_codex_config_env` (`acp.rs:257-345`) |

#### Stubs, dead scaffolding, and not-implemented paths

| Item | State | Evidence |
|---|---|---|
| `drain_stale_responses` | Fully implemented, never called — "Scaffolding for future model-switch timeout cleanup; not yet wired" | `acp.rs:1022-1023` |
| `session_new` (id-only wrapper) | `#[allow(dead_code)]`, "Public API — callers outside the harness may use this" | `acp.rs:591-599` |
| `active_run_id()` accessor | `#[cfg_attr(not(test), allow(dead_code))]`; production reads the field directly inside the read loop | `acp.rs:768-771` |
| `available_commands_update` | Logged only; the doc comment says "UI surfacing is a follow-up" | `acp.rs:1573-1588` |
| `plan` updates | Logged as "plan update received"; payload discarded | `acp.rs:1563-1566` |
| `keepalive` updates | Consumed with no side effect beyond the generic idle reset | `acp.rs:1621` |
| `authenticate` | No retry, no method discovery inside `acp.rs` — the caller enumerates methods from `initialize`'s result | `acp.rs:549-554` |
| Non-Unix process-group kill | Returns `false`, caller falls back to `child.start_kill()` | `acp.rs:1990-1992` |
| `allowed_respond_to` enforcement | Validated at startup, then display-only | `config.rs:919-937`, `config.rs:1019-1025` |
| `BUZZ_API_TOKEN` | Propagated from the legacy alias, then never read | `config.rs:718`; no other read site in the crate |
| `persona_env_vars` as a persona mechanism | Declared as a general injection vector but only ever populated with the one generated `CODEX_CONFIG` entry | `config.rs:945-955` |
| `handle_setup_membership`'s `_initial_channel_ids` | Accepted, unused | `setup_mode.rs:568` |

`buzz-persona` is declared as a dependency (`crates/buzz-acp/Cargo.toml:22`, notably by `path` rather than `workspace = true` like every other internal dep) with **zero** code references anywhere under `crates/buzz-acp/src`. What `config.rs` actually does instead: `from_args` initialises `persona_env_vars` as an empty `Vec` (`config.rs:945`) and the only thing ever pushed into it is the `codex_network_env` pair (`config.rs:951-957`). The comment at `config.rs:943-944` explains the design shift — spawned desktop agents now carry a complete instance snapshot, and team instructions arrive independently so they can be layered at runtime. Persona resolution has moved out of this crate; the dependency and the field's doc comment ("Populated from persona pack resolution", `config.rs:534`) are both stale.

#### Setup mode

Entered only when `BUZZ_ACP_SETUP_PAYLOAD` is present and parses (`setup_mode.rs:83`, read at `setup_mode.rs:214`, branch at `lib.rs:1290-1295`). The module header states the contract as non-negotiable (`setup_mode.rs:16-25`): desktop is the only readiness source, buzz-acp does not re-derive readiness, and normal startup gains no second readiness path.

What it does:

- Connects to the relay, sets a startup watermark, subscribes to membership notifications and to resolved channel filters (`setup_mode.rs:329-380`).
- Runs an event loop that, for @mentions passing the author + filter + dedup gates, publishes a single "needs configuration" reply naming exactly what is missing (`setup_mode.rs:388-478`).
- Reacts to membership add/remove by subscribing/unsubscribing channels, with no queue or session teardown because there is no pool (`setup_mode.rs:563-592`).
- Reconnects when the relay stream ends, relying on the `nudged_event_ids` set to avoid double-nudging on replay (`setup_mode.rs:390-397`, `setup_mode.rs:385-386`).

What it does not do: spawn any agent subprocess, create ACP sessions, or run any prompt. The agent pool is never reached because `run_setup_listener` is a `return` from the startup path (`lib.rs:1294`).

Nudge output is dual-format: human-readable markdown plus an appended fenced `buzz:config-nudge` block carrying the payload as JSON, so the desktop renders a `ConfigNudgeCard` and other clients see a code block (`setup_mode.rs:236-242`, `setup_mode.rs:296-302`).

Five requirement surfaces produce distinct copy (`setup_mode.rs:122-193`): a missing dropdown field, a missing env-backed credential, a CLI login step (with four sub-cases keyed on `AcpAvailabilityStatus`), an unparseable external CLI config (with a stderr diagnostic and a synthesised `~/.<cli>/config.toml` path at `setup_mode.rs:184`), and missing Git for Windows.

#### The compiled-in base prompt

`base_prompt.md` (136 lines) is embedded with `include_str!("base_prompt.md")` at `lib.rs:1544`, and again at `lib.rs:3592` and `lib.rs:3602`.

Selection ladder (`lib.rs:1539-1545`): `--no-base-prompt` → `None`; else `--base-prompt-file` content if `Config::base_prompt_content` is `Some`; else the compiled-in default. `base_prompt_content` is resolved and size-checked (1 MB cap) during `Config::from_args` (`config.rs:778-791`), so a missing or oversized file fails at startup rather than at prompt time. Clap marks `--base-prompt-file` and `--no-base-prompt` mutually exclusive (`config.rs:414-418`).

Injection differs by negotiated protocol version: v2 receives the base prompt through `session/new`'s `systemPrompt`, while v1 gets it prefixed onto the prompt text — the heartbeat path documents both branches at `lib.rs:4197-4210`, and `session_new_system_prompt` gates on `agent.protocol_version` at `pool.rs:829-833`. Layering order for the combined prompt is base → `[System]` persona → team instructions → agent core memory → canvas (`pool.rs:823-830`).

Content of the prompt, by section:

| Section | What it establishes |
|---|---|
| Preamble | The agent is running inside Buzz, a Nostr-based messaging platform; the buzz-acp harness routes channel events to its session |
| Buzz CLI | `buzz` is the primary interface; names the three auth env vars (`BUZZ_RELAY_URL`, `BUZZ_PRIVATE_KEY`, `BUZZ_AUTH_TAG`), the exit-code contract (0/1/2/3/4), a 13-row command-group table, stdin usage for multiline content, and the `--channel` requirement when opening PRs |
| Conversational Agent Creation | Restricts agent-creation questions to name + purpose; gives the `buzz agents draft-create` invocation; forbids asking about runtime/provider/model/credentials; states drafts require owner review |
| Communication Patterns | Mention rules (exact full display name, no formatting, no narrative mentions), mandatory callback mentions on completed delegated work, threading rules keyed off the `[Context]` block, mandatory publishing of results, prohibition on bare acknowledgements, todo discipline, GFM formatting, polling instead of push |
| Startup Recovery | A four-step catch-up procedure using `buzz feed get`, `buzz messages get`, `AGENTS.md`, and the knowledge directories |
| Workspace Layout | Names eight working directories (`RESEARCH/`, `PLANS/`, `GUIDES/`, `WORK_LOGS/`, `OUTBOX/`, `REPOS/`, `.scratch/`) and forbids recursive searches over `$HOME` or `/` |
| Agent Memory | `core` memory is auto-injected each turn; a 65,535-byte hard limit with a ~10 KB target; eviction and cold-slug rules |
| Engineering Discipline | Work-in-the-open, candour, read-before-change, commit/verification discipline, second opinions on risky changes |
| Working in the Repo | Worktree-not-default-branch; read repo-local git identity before committing |
| Autonomy | Resolve questions independently; escalate only for product intent |

The file is plain text with no templating, placeholders, or per-agent substitution — every managed agent receives byte-identical content unless the operator overrides the whole file.
