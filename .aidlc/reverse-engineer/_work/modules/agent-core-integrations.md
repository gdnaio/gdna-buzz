## Module: buzz-agent — core loop, ACP wire types & handoff (`crates/buzz-agent/src`)
### Aspect: Integrations
#### The ACP client (stdio, both directions)
Inbound frames are read from `tokio::io::stdin()` wrapped in `BufReader` (`lib.rs:171`) by `read_bounded_line` (`wire.rs:174-218`); outbound frames go through a single `mpsc::Sender<WireMsg>` with capacity 64 (`lib.rs:164`) into `writer_task`, the only writer of `tokio::io::stdout()` (`wire.rs:220-237`). All logging goes to stderr with ANSI disabled (`lib.rs:156-159`), keeping the stdout channel pure.

Failure behavior at the seam: if stdout write fails, `writer_task` returns (`wire.rs:232-234`) and every later `wire::send` silently no-ops because the result is discarded (`wire.rs:170-172`). The agent keeps looping without any output.

#### buzz-acp (the primary client)
This group is written against buzz-acp's expectations, and several contracts are cross-crate:

| Contract | buzz-agent side | buzz-acp side |
|---|---|---|
| `usage_update` on `_goose/unstable/session/update`, before the prompt response | `lib.rs:730-750` | `UsageTracker` (`crates/buzz-acp/src/usage.rs:164`), payload shape documented at `usage.rs:47` |
| `keepalive` resets the idle clock | `agent.rs:134-144` | classified as non-content (`crates/buzz-acp/src/acp.rs:1623`), test `keepalive_resets_idle_past_deadline` (`acp.rs:2870`) |
| `_meta.goose.activeRunId` seeds the steer target | `lib.rs:661-670` | cached in `AcpClient::active_run_id` (`acp.rs:189`, accessor `acp.rs:769`), used as `expectedRunId` (`acp.rs:1293-1313`) |
| `stopReason` strings | `types.rs:183-191` | parsed at `acp.rs:61-72` |
| `agent_thought_chunk` / `agent_message_chunk` | `agent.rs:185`, `agent.rs:199` | handled at `acp.rs:1566` and `acp.rs:1535` |
| `systemPrompt` on `session/new` (protocol ≥ 2) | consumed `lib.rs:361-370` | `[Base]` composition gated on version (`crates/buzz-acp/src/pool.rs:181`, `crates/buzz-acp/src/lib.rs:4191-4210`) |

Contract drift found: `crates/buzz-acp/src/acp.rs:185-186` documents that "goose/buzz-agent emit `activeRunId: null` at end of turn". buzz-agent does not — `grep -n 'activeRunId' src/*.rs` shows exactly one emission site (`lib.rs:665`), at prompt start; end-of-turn only clears the field internally (`lib.rs:707`). The client's cached run id therefore goes stale, and staleness is caught only by buzz-agent's own mismatch rejection (`lib.rs:588-598`).

Observability side effect: buzz-acp forwards every frame it reads from the agent verbatim to its observer as `acp_read` (`crates/buzz-acp/src/acp.rs:1120`, `acp.rs:1414`). Everything this group writes — including `rawInput` (the full tool arguments, `agent.rs:412`) and completed tool result text (`agent.rs:592`) — is therefore republished; the desktop transcript reads `update.rawInput` directly (`desktop/src/features/agents/ui/agentSessionTranscriptHelpers.ts:361`).

#### Sibling modules inside the crate
| Dependency | Used for | Call sites |
|---|---|---|
| `config` | `Config::from_env` and every tunable; `PROTOCOL_VERSION`, `MAX_*` constants | `lib.rs:41`, `lib.rs:160`, `agent.rs:8` |
| `llm` | `Llm::new`, `complete`, `summarize` | `lib.rs:161`, `agent.rs:124`, `handoff.rs:49-54` |
| `mcp` | `McpRegistry::spawn_all`, `tools`, `has`, `is_hook`, `call`, `call_hooks`, `server_of`, `kill_server`, `ResultBudget` | `lib.rs:390`, `agent.rs:115`, `agent.rs:224-231`, `agent.rs:330-337`, `agent.rs:509-537`, `handoff.rs:70-77`, `handoff.rs:177` |
| `builtin` | `LOAD_SKILL_TOOL`, `load_skill_def`, `call_load_skill` | `agent.rs:118`, `agent.rs:316-323` |
| `hints` | `build_hints_section`, `SkillEntry` | `lib.rs:356-359` |
| `catalog` | `discover_databricks_models`, `discovery_failure_fallback`, `ModelEntry` | `lib.rs:448-452`, `lib.rs:325` |
| `auth` | `PkceOAuthConfig`, `PkceOAuthTokenSource::interactive_login` | `lib.rs:136-142` |

#### External crates touched directly by this group
| Crate | Use | Site |
|---|---|---|
| `tokio` | runtime, stdio, `Mutex`, `watch`, `mpsc`, `Semaphore`, `JoinSet`, `OnceCell`, `interval`, `timeout_at` | `lib.rs:110-120`, `lib.rs:164`, `agent.rs:11-12`, `agent.rs:455` |
| `serde` / `serde_json` | all params deserialization and every outbound frame | `wire.rs:1-5`, `types.rs:1-2` |
| `getrandom` | 8-byte session/run/steer tokens | `lib.rs:821-825` |
| `tracing` / `tracing_subscriber` | stderr logging | `lib.rs:156-159` and throughout |

#### Cross-crate consumers of this group
- `sprig` multicall binary dispatches `argv[0] == "buzz-agent"` straight to `buzz_agent::run()` (`crates/sprig/src/main.rs:18`), so the argv contract in `run()` (`lib.rs:110-121`) is shared.
- The desktop Tauri backend links the crate as `buzz_agent_pkg` (`desktop/src-tauri/Cargo.toml:91`) purely to read `WINDOWS_SHELL_RESOLUTION_ENV` (`desktop/src-tauri/src/managed_agents/git_bash.rs:136`, `:438`) — a shared-constant integration, deliberately sourced from this crate rather than copied.

#### Subprocess and filesystem surface
This group spawns no processes itself; MCP children are created by `McpRegistry::spawn_all` with the client-supplied `cwd` and env (`lib.rs:390`). Its only filesystem interaction is indirect: hints/skills discovery under the session `cwd` (`lib.rs:356-359`) and the OAuth token cache written by `auth` (`lib.rs:140-144`). No direct file reads or writes exist here (`grep -c 'fs::write'`, `grep -c 'File::create'`, `grep -c 'std::fs'` → 0 across all six files).

#### Copied-rather-than-shared behavior
- The goose wire extensions (`_goose/unstable/session/steer`, `_goose/unstable/session/update`, `_meta.goose.*`) are reimplemented from goose's contract, documented in comments rather than by a shared type (`lib.rs:544-553`, `wire.rs:69-74`, `wire.rs:143-149`). The upstream reference is cited only from the client side (`crates/buzz-acp/src/acp.rs:178-180`).
- Three independent string-truncation helpers coexist: `types::clamp` (`types.rs:259-274`, used only by `mcp.rs:254` and `mcp.rs:630`), `handoff::clamp_bytes` (`handoff.rs:300-315`), and `mcp::truncate_at_boundary` / `mcp::truncate_middle` (`mcp.rs:866`, `mcp.rs:886`). `clamp` appends `\n[truncated]`, `clamp_bytes` appends `…` — same job, different markers, no shared implementation.
