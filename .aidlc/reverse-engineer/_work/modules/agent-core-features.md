## Module: buzz-agent — core loop, ACP wire types & handoff (`crates/buzz-agent/src`)
### Aspect: Features
#### Operator-visible capabilities
| Capability | Where | Limits |
|---|---|---|
| ACP JSON-RPC 2.0 agent over stdio | `read_loop` (`lib.rs:187-206`), `writer_task` (`wire.rs:220-237`) | Newline-delimited frames only; one frame ≤ `max_line_bytes` (default 4 MiB, `config.rs:813`); oversize frame terminates the read loop (`wire.rs:194-198`) |
| Agentic tool loop (LLM → tool calls → results → repeat) | `RunCtx::run` (`agent.rs:66-257`) | Non-streaming: one request per round (`agent.rs:121-146`); rounds capped by `max_rounds` (0 = unlimited, `agent.rs:88`) |
| Parallel tool execution | `execute_parallel` (`agent.rs:370-490`) | ≤ `max_parallel_tools` concurrent (default 8, `config.rs:821`); ≤ 64 calls per round (`agent.rs:242-252`) |
| Interruptible turns | biased `select!` at every await (`agent.rs:121-125`, `agent.rs:424-441`, `handoff.rs:45-48`) | Cancel is per session, not per tool; drain bounded to 5 s then tasks are aborted (`agent.rs:455-478`) |
| Live model switching | `session/set_model` (`lib.rs:503-542`) | Takes effect on the *next* prompt (`lib.rs:671-673`); no validation against the advertised catalog; no notification to the client that a switch happened |
| Model catalog advertisement | `session/new` response (`lib.rs:443-474`) | Real discovery only for Databricks/DatabricksV2; other providers get a one-entry list echoing `cfg.model` (`lib.rs:459`) |
| Mid-turn steering without cancel | `steer_session` (`lib.rs:554-621`) + `drain_steers` (`agent.rs:265-280`) | Applied at round boundaries only; requires a matching `expectedRunId`; a steer queued after the last round is silently never delivered |
| Long-provider-call keepalive | `agent.rs:126-145` | Fixed 30 s interval, not configurable; consumed by the harness idle clock (`crates/buzz-acp/src/acp.rs:1623`) |
| Automatic context handoff (self-summarizing compaction) | `maybe_handoff` (`handoff.rs:31-107`) | ≤ `max_handoffs` per session (default 10); summary capped at 8192 output tokens (`config.rs:652`); after the cap, plain truncation takes over (`handoff.rs:34-41`) |
| Graceful history truncation | `truncate_history` (`agent.rs:711-738`) | Drops oldest whole turns; can return over budget when no later user turn exists (`agent.rs:723-725`) |
| MCP lifecycle hooks (`_Stop`, `_PostCompact`) | `agent.rs:224-236`, `handoff.rs:70-77` | Off unless `MCP_HOOK_SERVERS` is set (`config.rs:824`); advisory, fail-open, per-prompt objection budget (default 3) |
| Built-in `load_skill` tool | injected `agent.rs:115-119`, executed inline `agent.rs:317-327` | Only when the session discovered skills; no MCP round trip |
| Reasoning/thinking passthrough | `agent.rs:179-192` | Emitted as `agent_thought_chunk` before the message chunk; empty for providers that don't expose it (`types.rs:141-155`) |
| Per-turn + cumulative token reporting | `lib.rs:714-752` | Emitted only when the provider reported at least one count; `contextLimit` is hardcoded `0` (`lib.rs:742`) |
| Interactive provider auth | `buzz-agent auth databricks` (`lib.rs:129-152`) | Databricks OAuth PKCE only; needs a browser; writes under `~/.config/buzz-agent/oauth/databricks/` (`lib.rs:143`, path built at `auth.rs:454-461`) |

#### Deliberate non-features (verified in code, not just claimed)
- **No session load/resume**: `loadSession:false` advertised (`lib.rs:292`); no `session/load` arm exists (`lib.rs:224-265`).
- **No image/audio/embedded-context prompts**: advertised false (`lib.rs:293`) and enforced — `ContentBlock` accepts only `text` and `resource_link` (`types.rs:210-221`), anything else errors (`agent.rs:623-628`).
- **No HTTP/SSE MCP transport**: advertised false (`lib.rs:294`); `session/new` only accepts stdio server specs (`types.rs:195-202`).
- **No persistence**: zero filesystem writes in this group (`grep -c 'fs::write'` and `grep -c 'File::create'` → 0 in all six files).
- **No streaming**: a single `llm.complete` per round (`agent.rs:124`); text arrives as one `agent_message_chunk` (`agent.rs:194-206`).
- **No session teardown**: `grep -n 'sessions.remove' lib.rs` → 0 matches. Sessions and their MCP children persist until the process exits; shutdown only broadcasts cancel (`lib.rs:178-180`).
- **No CLI surface beyond `auth`**: unknown arguments are ignored and the agent starts reading stdio (`lib.rs:110-121`). No `--help`, no `--version`.

#### Feature limits a user will actually hit
- A prompt over 1 MiB is rejected outright (`agent.rs:69-73`) — but a *steer* has no equivalent per-message cap, only the frame cap.
- Concurrent prompts on one session are refused rather than queued (`lib.rs:786-788`); the client must serialize.
- `session/cancel` for an unknown session succeeds silently (`lib.rs:487-494`), so a client cannot distinguish "cancelled" from "no such session".
- After a handoff, the summary replaces all detail: only `[Context Handoff]\n{summary}` and the current prompt survive (`handoff.rs:84-95`).
- Tool results are middle-elided at `max_tool_result_text_bytes` (default 50 KiB) before entering history — enforced in the MCP layer via the `ResultBudget` this group constructs (`agent.rs:385-388`, `config.rs:649`, `config.rs:815-818`).
