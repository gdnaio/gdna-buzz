## Module: buzz-agent — core loop, ACP wire types & handoff (`crates/buzz-agent/src`)
### Aspect: Debt
#### Documentation drift (crate README vs code)
The crate README is the only real reference for this group, and seven of its statements no longer match the code:

| README claim | Line | Reality |
|---|---|---|
| "The full server is hand-rolled in `main.rs`" | `README.md:124` | `main.rs` is 6 lines and only calls `buzz_agent::run()` (`main.rs:1-6`); the server is `lib.rs:187-825` |
| "Three request methods (`initialize`, `session/new`, `session/prompt`), one inbound notification (`session/cancel`), and three outbound update variants" | `README.md:124` | Six request methods are handled — plus `session/set_model` (`lib.rs:234`), `session/cancel` as a *request* (`lib.rs:237`), and `_goose/unstable/session/steer` (`lib.rs:245`) — and ten outbound update variants exist (`agent.rs:552-616`, `agent.rs:134-206`, `lib.rs:612-620`, `lib.rs:661-670`, `lib.rs:730-750`) |
| Handshake transcript shows `protocolVersion: 1` in both directions | `README.md:68`, `README.md:70` | `PROTOCOL_VERSION` is 2 (`config.rs:3`); a client asking for 2+ gets 2 (`lib.rs:284`), asserted by `test_initialize_version_check` (`tests/golden_transcripts.rs:288`) |
| `session/new` result is `{"sessionId":"ses_…"}` | `README.md:84` | Result also carries `models.currentModelId` and `models.availableModels` (`lib.rs:464-474`) |
| "Everything is environment variables. No flags, no config files." | `README.md:128` | There is a CLI subcommand: `buzz-agent auth <provider>` (`lib.rs:111-116`, `lib.rs:129-152`) |
| `BUZZ_AGENT_MAX_HISTORY_BYTES` default `1048576` / "History window 1 MiB" | `README.md:158`, `README.md:236` | Code default is 16 MiB (`config.rs:814`) — a 16× discrepancy |
| "One reader, one writer, up to 8 concurrent prompt tasks (one per session)" | `README.md:286` | Sessions default to unlimited (`config.rs:812`) and each gets its own prompt task (`lib.rs:623-625`); `8` is the default `max_parallel_tools` (`config.rs:821`), a different limit |

The transcript section also predates `agent_thought_chunk`, `keepalive`, `usage_update`, `session_info_update`, and `session/set_model`, so a reader implementing a client from `README.md:60-122` will not know to handle them.

#### Documentation drift (repo-level)
- `ARCHITECTURE.md` never mentions this crate: `grep -rn 'buzz-agent' ARCHITECTURE.md` returns zero matches, and the crate inventory at `ARCHITECTURE.md:88-93` omits it while listing `buzz-acp`, `buzz-sdk`, `buzz-cli`, `buzz-admin`, `buzz-test-client`. The root `README.md:209` and `AGENTS.md:50` both do list it, so the architecture doc is the outlier.
- `CONTRIBUTING.md` has zero mentions (`grep -rn 'buzz-agent' CONTRIBUTING.md` → 0 matches), so no contributor guidance exists for the crate that owns the agent loop.
- `.env.example` documents none of the ~20 `BUZZ_AGENT_*` variables (`grep -c 'BUZZ_AGENT' .env.example` → 0) while documenting the `BUZZ_ACP_*` harness set in detail (`.env.example:114-170`).

#### Cross-crate contract drift
`crates/buzz-acp/src/acp.rs:185-186` documents that "goose/buzz-agent emit `activeRunId: null` at end of turn", and the client relies on that to clear its cached run id. buzz-agent emits `activeRunId` exactly once per turn, at prompt start (`lib.rs:661-670`; `grep -n 'activeRunId' src/*.rs` shows one emission site), and clears the value only internally (`lib.rs:707`). The client's cache therefore stays stale until the next turn starts; the only thing preventing a misdirected steer is buzz-agent's own mismatch rejection (`lib.rs:588-598`). Either the comment or the emission is wrong — worth resolving in whichever direction the steer protocol intends.

`docs/MCP_DRIVEN_HOOKS.md:14-17` states hook responses are injected as tool-result messages and are JSON-encoded for prompt-injection safety. True for `_Stop` (`agent.rs:641-651`, `agent.rs:666-695`); false for `_PostCompact`, which is concatenated as plain `[{server}]\n{text}` (`handoff.rs:247-253`) into a synthetic **user** message (`handoff.rs:85-92`). The code has a documented reason (`handoff.rs:78-83`) but the spec doc does not record the exception.

#### Stale / misplaced in-code comments
- `handoff.rs:152-162` — the comment on the `None` arm of `projected_handoff_input_tokens` describes byte-to-token mapping, "capped conservatively", and "Never raise the cap above the configured byte budget". That arm is a one-liner returning `current_tokens` (`handoff.rs:163`); the capping logic it describes lives in `should_handoff` (`handoff.rs:115-126`) and `byte_fallback_threshold` (`handoff.rs:359-368`). Eleven lines of explanation attached to the wrong site.
- `lib.rs:255-259` — a deferred-work note ("Revisit when that RFD merges") about `[Base]` gating that this crate does not implement; the gate is in `crates/buzz-acp/src/pool.rs:181`.
- `agent.rs:284-293` — `execute_calls`' doc describes a "(degenerate, max_parallel=1) path" as if two code paths existed; there is only one (`agent.rs:339`).

There are **no `TODO`/`FIXME`/`XXX`/`HACK` markers anywhere in this group** (`grep -n 'TODO\|FIXME\|XXX\|HACK' lib.rs agent.rs types.rs wire.rs handoff.rs main.rs` → 0 matches). Deferred work is instead recorded in prose comments like the one above, which means no tooling or grep-based audit will ever surface it.

#### Structural debt
- **13-element tuple as an interface.** `acquire_session` returns a 13-field anonymous tuple (`lib.rs:764-783`) destructured into 13 named locals at the call site (`lib.rs:632-652`). Adding one piece of session state means touching the signature, the tuple literal (`lib.rs:806-819`), the destructuring pattern, and the write-back block (`lib.rs:704-712`) — four coupled edits with no compiler help on ordering, since several fields share the type `Option<String>` / `Option<u64>`.
- **`Session::id` duplicates the map key.** Both are set from the same `session_id` (`lib.rs:411-412`) and the field is read once (`lib.rs:809`); they can never legitimately diverge but nothing enforces that.
- **No session teardown.** `grep -n 'sessions.remove' lib.rs` → 0 matches. Sessions and their MCP child processes accumulate for the process lifetime, with `max_sessions` defaulting to `usize::MAX` (`config.rs:812`).
- **Panic in a prompt task wedges the session permanently.** `busy` is cleared only on the normal path (`lib.rs:704-705`), history has already been moved out (`lib.rs:807`), and there is no `catch_unwind` (`grep -c 'catch_unwind'` → 0 in all six files). A panicked turn leaves the session unusable and its context lost, with no diagnostic beyond the tokio task-panic message.
- **Mutex held across an await.** The second session-cap re-check takes the `sessions` lock at `lib.rs:399` and awaits `reject(...)` at `lib.rs:401-408` while still holding it; if the 64-slot wire channel (`lib.rs:164`) is full, that blocks every other session operation including `session/cancel` (`lib.rs:489`).
- **Unbounded steer channel.** `mpsc::unbounded_channel()` (`lib.rs:802`) with no per-steer size check in `drain_steers` (`agent.rs:266-270`), while the initial prompt *is* capped at 1 MiB (`agent.rs:69-73`). The asymmetry is unexplained.
- **Three truncation helpers.** `types::clamp` (`types.rs:259-274`, marker `\n[truncated]`), `handoff::clamp_bytes` (`handoff.rs:300-315`, marker `…`), `mcp::truncate_middle`/`truncate_at_boundary` (`mcp.rs:866`, `mcp.rs:886`). `clamp` additionally lives in `types.rs` but has zero callers there — both users are in `mcp.rs` (`:254`, `:630`), so it is a misplaced helper exported as public API.
- **Inconsistent exit codes.** `die()` → 2 (`lib.rs:105-108`); `main.rs` → 1 (`main.rs:2-5`); a fatal reader error (oversize frame, invalid UTF-8) is only logged and the process exits **0** (`lib.rs:170-176` then `Ok(())` at `lib.rs:121`). A supervisor cannot distinguish clean shutdown from protocol abort.
- **Inconsistent RNG failure policy.** `session_new` fails closed on RNG error (`lib.rs:394-397`) but run ids and steer message ids fail open to the literals `run_x` / `steer_x` (`lib.rs:799`, `lib.rs:577`), which are the values steer authorization compares against (`lib.rs:588-598`).
- **Threshold operator asymmetry.** `should_handoff` uses `>=` on the token path and `>` on the byte path (`handoff.rs:111-113` vs `handoff.rs:115-126`), with no comment explaining the difference.
- **`Session` field write-back is manual and partial.** `run_prompt` restores seven fields (`lib.rs:704-712`) but not `accumulated_*`, which are updated separately under a second lock acquisition (`lib.rs:717-727`) — two lock round-trips per turn and two places that must agree about what "end of turn" means.

#### Hardcoded values that should probably be config
| Value | Site | Why it matters |
|---|---|---|
| keepalive interval 30 s | `agent.rs:129` | Exists to satisfy the harness idle clock, which *is* configurable (`.env.example:169-170`); the two can drift out of alignment |
| post-cancel drain 5 s | `agent.rs:455` | After this, tasks are aborted mid-flight (`agent.rs:466-471`) |
| wire channel depth 64 | `lib.rs:164` | Backpressure point for every notification, including the lock-holding path above |
| `usage_update.contextLimit = 0` | `lib.rs:742` | Reported to the harness as "no limit" even though `max_context_tokens` is known (`handoff.rs:113`) |
| token width 8 bytes | `lib.rs:822` | 64-bit session/run id entropy |

#### Test-coverage gaps
`agent.rs` — 746 lines, the busiest file in the group — has **no unit tests at all** (`grep -n '#\[cfg(test)\]' agent.rs` → 0 matches), while `lib.rs`, `types.rs`, `wire.rs` and `handoff.rs` each carry a test module. Directly untested logic with non-obvious behavior:

| Untested path | Site | Risk |
|---|---|---|
| `truncate_history` `break` when no later `User` item exists | `agent.rs:723-725` | Returns silently over budget; only the happy path is exercised, and only via integration (`tests/regressions.rs:555`) |
| `prompt_to_text` joining and `resource_link` rendering | `agent.rs:618-631` | Only the error branch is covered (`tests/golden_transcripts.rs:384`) |
| `map_stop` collapse of `ToolUse`/`Other` → `end_turn` | `agent.rs:740-746` | Silent semantic mapping |
| `format_hook_output_body` escaping (the prompt-injection defense) | `agent.rs:641-651` | The security property has no direct assertion |
| `synthetic_hook_id` / `unique_nonce` uniqueness | `agent.rs:654-657`, `agent.rs:697-701` | Duplicate ids would corrupt the provider wire format |
| Preflight-cancel slot filling and `j < idx` `failed` emission | `agent.rs:300-313` | Ordering-sensitive; covered only indirectly |
| `max_rounds` cap | `agent.rs:88-90` | `grep -n 'MAX_ROUNDS\|max_turn_requests' tests/` → 0 matches |
| Both `max_sessions` rejections | `lib.rs:346-355`, `lib.rs:399-409` | `grep -n 'max sessions reached' tests/` → 0 matches |
| Non-UTF-8 frame rejection | `wire.rs:207-214` | No test |
| Steer `expectedRunId` empty-string rejection | `lib.rs:567-575` | No test (the other three steer rejections are covered) |
| `session/cancel` for an unknown session returning success | `lib.rs:487-494` | Behavior is indistinguishable from a real cancel |

#### Things the LOC table gets right
The stated sizes are exact: `wc -l` gives `lib.rs` 954, `main.rs` 6, `agent.rs` 746, `types.rs` 353, `wire.rs` 293, `handoff.rs` 430 — 2,782 total. Note, though, that 127 of `lib.rs`'s lines are its test module (`lib.rs:827-954`), so the production body is ~2,655 lines, and the four test modules total ~180 lines against ~2,600 lines of logic.
