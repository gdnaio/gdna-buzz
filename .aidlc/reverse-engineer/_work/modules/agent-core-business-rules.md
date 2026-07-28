## Module: buzz-agent — core loop, ACP wire types & handoff (`crates/buzz-agent/src`)
### Aspect: Business Rules
#### Turn loop: fixed ordering per round
`RunCtx::run` (`agent.rs:66-257`) executes this order every round, and the order is load-bearing:

1. Round cap: `if cfg.max_rounds > 0 && round >= cfg.max_rounds → StopReason::MaxTurnRequests` (`agent.rs:88-90`). `0` means unlimited (default `0`, `config.rs:801`).
2. Cancel poll: `if *self.cancel.borrow() → Cancelled` (`agent.rs:91-93`).
3. Steer drain (`agent.rs:98`) — queued steers become `User` history items *before* the handoff decision, so a steer is included in the summarized context rather than lost to it.
4. Handoff gate (`agent.rs:99-112`). `Performed` clears `last_request_input_tokens` **and** `last_request_history_bytes` (`agent.rs:104-106`) so a stale over-threshold reading can't immediately re-fire; `Skipped` instead runs `truncate_history` (`agent.rs:111`); `Cancelled` returns.
5. Tool catalog assembly, with `load_skill` appended only when the session discovered skills (`agent.rs:115-119`).
6. `round += 1` (saturating, `agent.rs:120`), then the provider call.

Pre-loop validation: prompt blocks are flattened and joined with `\n` (`agent.rs:618-631`), rejected over `MAX_PROMPT_BYTES` = 1 MiB (`agent.rs:69-73`, `config.rs:638`), and the first prompt of a session is memoized as `original_task` (`agent.rs:78-80`) for later handoff summaries.

#### Cancellation semantics
Cancel always wins: every wait uses `tokio::select! { biased; _ = cancel.changed() => … }` — the provider call (`agent.rs:121-146`), the handoff summarizer (`handoff.rs:45-48`), and the tool `JoinSet` loop (`agent.rs:424-441`). Additional rules:
- Preflight cancel fills every empty result slot with a synthetic `cancelled` error and emits a terminal `failed` for calls that already got `pending` (`agent.rs:300-313`).
- On cancel during execution the semaphore is closed rather than tasks aborted (`agent.rs:430-435`), so each in-flight MCP call can send `notifications/cancelled` itself; late permit acquisition emits `failed` and returns (`agent.rs:378-384`).
- The post-cancel drain is bounded to 5 s, then `set.abort_all()` (`agent.rs:455-478`).
- Any still-unfilled runnable slot becomes a synthetic `cancelled` result plus a `failed` update, so no client is left with a permanently `pending` tool (`agent.rs:483-489`).
- Results are appended to history even on the cancel path (`agent.rs:311`, `agent.rs:341`) — that is what keeps history valid for the next prompt (test `cancel_leaves_history_valid_for_next_prompt`, `tests/regressions.rs:456`).
- A cancelled turn still emits `usage_update` if any tokens were seen (`lib.rs:714`; test `cancelled_turn_with_usage_emits_notification_before_response`, `tests/fake_llm.rs:926`).

#### Timeouts
| Rule | Value | Site |
|---|---|---|
| Per-tool call timeout | `cfg.tool_timeout` (default 660 s, `config.rs:804`) | `agent.rs:509-517` |
| On tool timeout, kill the owning MCP server's process group | — | `agent.rs:531-537` |
| Except when the session was cancelled — then do not kill a healthy server | — | `agent.rs:526-530` |
| Post-cancel drain bound | hardcoded 5 s | `agent.rs:455` |
| Keepalive tick while awaiting the provider | hardcoded 30 s, first tick skipped | `agent.rs:129-131` |
| `_Stop` / `_PostCompact` hook budget | `cfg.hook_timeout` (default 2500 ms, `config.rs:822`) | `agent.rs:228`, `handoff.rs:75` |

The timeout message is deliberately instructive to the model: `"tool: timeout after {n}s. The command took too long. Try a faster approach."` (`agent.rs:538-541`). Similarly, every failed tool result gets `ERROR_REFLECTION_SUFFIX` appended — "[Reflect] Before retrying, identify the cause and change your approach." (`agent.rs:21-22`, applied `agent.rs:361-366`). Both are prompt-engineering rules encoded in control flow, with no test asserting the text.

#### Concurrency and admission rules
- One prompt per session: `acquire_session` rejects with `"prompt already in flight"` when `busy` (`lib.rs:786-788`); surfaced as `-32602` `session/prompt: prompt already in flight` (`lib.rs:653-661`). Tests: `rejects_concurrent_prompts` (`tests/fake_llm.rs:320`), `test_concurrent_prompt_rejected` (`tests/golden_transcripts.rs:409`).
- Session cap checked twice: once before the (slow) MCP spawn without holding the lock (`lib.rs:346-355`) and again after, under the lock (`lib.rs:399-409`). No test covers either rejection — `grep -n 'max sessions reached' tests/` returns zero matches.
- Tool fan-out is bounded by `Semaphore(max(cfg.max_parallel_tools, 1))` (`agent.rs:371-372`); `1` degenerates to sequential execution.
- Per-turn tool-call cap: over `MAX_TOOL_CALLS_PER_TURN` = 64 the list is truncated with a `warn!` and the *truncated* list is what enters history (`agent.rs:242-252`, `config.rs:650`). Test `per_turn_tool_call_cap_enforced` (`tests/regressions.rs:606`).

#### Validation rules
| Rule | Enforcement | Test |
|---|---|---|
| `cwd` non-empty and absolute | `lib.rs:334-343` | `test_session_new_rejects_relative_cwd` (`tests/golden_transcripts.rs:314`) |
| Combined system prompt ≤ 512 KiB | `lib.rs:375-387` | `session_new_rejects_oversized_system_prompt` (`tests/fake_llm.rs:415`) |
| Prompt ≤ 1 MiB | `agent.rs:69-73` | none found (`grep 'MAX_PROMPT_BYTES' tests/` → 0 matches) |
| Inbound frame ≤ `max_line_bytes` | `wire.rs:194-198` | `rejects_oversized_line` (`tests/fake_llm.rs:387`), `test_oversized_line_kills_agent` (`tests/golden_transcripts.rs:471`) |
| Frame must be valid UTF-8 | `wire.rs:207-214` | none found |
| `modelId` non-blank | `lib.rs:509-516` | `session_set_model_empty_model_id_returns_error` (`tests/databricks_oauth.rs:906`) |
| Steer `prompt` non-empty | `lib.rs:559-566` | `steer_rejected_on_empty_prompt` (`tests/fake_llm.rs:1092`) |
| Steer `expectedRunId` non-empty | `lib.rs:567-575` | none found |
| Provider claims `stop=tool_use` with zero calls → hard error | `agent.rs:208-212` | none found |

The `cwd` check is *shape only* — existence, readability and canonicalization are not verified before the path is handed to MCP child spawn (`lib.rs:390`).

#### Precedence rules
| Decision | Winner | Site |
|---|---|---|
| Effective model | session `effective_model` (from `session/set_model`) over `cfg.model` | `lib.rs:671-673` |
| System prompt | client `systemPrompt` when non-blank, else `cfg.system_prompt` | `lib.rs:361-370` |
| Hints | appended after the chosen base with a blank line, when `cfg.hints_enabled` | `lib.rs:356-374` |
| Models catalog | cached successful discovery > fresh discovery > provider-aware fallback (never cached) | `lib.rs:312-327` |

`session/new` reports `currentModelId: cfg.model` (`lib.rs:470`) — it does not consult any session override, which is correct only because the session is brand new.

#### Steering rules (`_goose/unstable/session/steer`)
Accepted only when all hold (`lib.rs:554-605`): non-empty prompt, non-empty `expectedRunId`, known session, `active_run_id.is_some()`, `expectedRunId == active_run_id`, and `steer_tx.send()` succeeds. Any failure is `-32602`; the mismatch message names both ids (`lib.rs:590-596`). Delivery is *queue-only* — the message is folded into history at the **next round boundary**, never mid-round, and the turn is not restarted (`agent.rs:94-98`, `agent.rs:265-280`). A steer whose blocks fail to render is dropped with `warn!` rather than aborting the turn (`agent.rs:274-278`); an empty-after-render steer is dropped with `debug!` (`agent.rs:271-273`). Tests: `steer_folds_into_active_turn_without_cancelling` (`tests/fake_llm.rs:596`), `steer_rejected_when_no_active_run` (`tests/fake_llm.rs:683`), `steer_rejected_on_run_id_mismatch` (`tests/fake_llm.rs:705`).

Consequence worth noting: a steer arriving after the final round (while the provider is producing the last response) is queued but never drained, because the loop returns before the next boundary (`agent.rs:207-240`). Nothing reports the dropped steer to the client.

#### `_Stop` hook rules (agent sovereignty)
- Consulted **only** when the mapped stop reason is exactly `EndTurn` (`agent.rs:217-219`) — `max_tokens` and `refusal` are never overridden.
- Budget is per prompt, not per session: `stop_rejections` is a loop-local counter (`agent.rs:86`), checked before calling hooks (`agent.rs:220-222`). `cfg.stop_max_rejections = 0` disables the hook entirely (default 3, `config.rs:823`).
- Objections are injected as synthetic assistant-tool-call + tool-result pairs, never as user or system messages (`agent.rs:660-695`), then the loop `continue`s (`agent.rs:233-236`).
- Fail-open: `call_hooks` drops errors, timeouts and blank output (`mcp.rs:309-313`), so a broken hook can never trap the agent.
Tests: `hook_stop_blocks_premature_end` (`tests/regressions.rs:787`), `hook_stop_budget_exhausted` (`:872`), `hook_stop_consecutive_end_turn_uses_rejection_budget` (`:927`), `hook_stop_budget_resets_per_prompt` (`:979`), `hook_stop_timeout_failopen` (`:1514`), `hook_tools_hidden_from_llm` (`:1035`).

Hook tools are unreachable from the model by two independent rules: they are filtered out of `mcp.tools()` and any direct invocation is answered `unknown tool: {name}` (`agent.rs:330-337`).

#### Handoff (context compaction) rules
`maybe_handoff` (`handoff.rs:31-107`):
1. Gate — `should_handoff()` (`handoff.rs:109-129`). With provider usage known: `projected_input_tokens >= token_threshold(max_context_tokens, max_output_tokens)`. Without usage: `context_pressure_bytes > byte_fallback_threshold(...)`. **The comparison operators differ (`>=` vs `>`)** between the two branches (`handoff.rs:111-113` vs `handoff.rs:115-126`) — an off-by-one asymmetry with no stated rationale.
2. Cap — `handoff_count >= cfg.max_handoffs` (default 10, `config.rs:820`) logs and falls back to truncation (`handoff.rs:34-41`).
3. Summarize with a dedicated system prompt (`handoff.rs:25-29`, "Stay under 8192 tokens") and `HANDOFF_MAX_OUTPUT_TOKENS = 8192` (`config.rs:652`), on the session's effective model (`handoff.rs:53`).
4. Empty or failed summary → `Skipped` (truncate instead), never a hard error (`handoff.rs:54-63`).
5. History is cleared **before** calling `_PostCompact`, so the hook injects into the fresh context (`handoff.rs:66-77`).
6. Fresh context is exactly: one `User` item holding `[Context Handoff]\n{summary}` (plus optionally the labeled `_PostCompact` block), then the current user prompt re-appended verbatim (`handoff.rs:84-95`). The comment at `handoff.rs:78-83` explains the rule: tool results must not be orphaned because OpenAI Chat/Responses require them to follow an assistant tool call.

Threshold math is the best unit-tested logic in the group: `token_threshold` = `min(window*9/10, window - max_output_tokens)` (`handoff.rs:348-353`) with tests at `handoff.rs:388`, `:395`, `:403`; `byte_fallback_threshold` clamps to `max_history_bytes*9/10` (`handoff.rs:359-368`) with test at `handoff.rs:410`; `handoff_prompt_budget_bytes` tests at `handoff.rs:378`, `:383`; `estimate_tokens_from_bytes` at `handoff.rs:422`. `CONSERVATIVE_BYTES_PER_TOKEN = 1` (`handoff.rs:326`) is justified as an unconditional upper bound on tokens — the gate is intentionally biased toward handing off early.

Integration coverage: `token_usage_over_budget_triggers_handoff` (`tests/regressions.rs:1354`), `stale_usage_plus_history_growth_triggers_handoff` (`:1436`), `handoff_summary_prompt_includes_full_history_within_context_budget` (`:1224`), `handoff_summary_prompt_keeps_latest_item_when_one_item_exceeds_budget` (`:1292`), `hook_post_compact_injects_after_handoff` (`:1112`).

#### History truncation rule
`truncate_history` (`agent.rs:711-738`) drops whole leading segments up to (but not including) the next `User` item, so the window always begins at a user turn and never orphans a tool result. Two silent behaviors: if no later `User` item exists it `break`s and **returns over budget** (`agent.rs:723-725`), and it uses `estimated_bytes` (real wire size), not `context_pressure_bytes` — deliberately, per `types.rs:24-28`. Covered indirectly by `history_budget_evicts_old_turns` (`tests/regressions.rs:555`); the over-budget break path has no test.

#### Rules enforced only by comment or convention
- `lib.rs:255-259` — gating `[Base]` prompt injection on `protocol_version < 2` is described as a temporary measure, but that gate lives in the client (`crates/buzz-acp/src/pool.rs:181`), not here. Nothing in this crate enforces or checks it.
- `lib.rs:714-718` — the "usage notification must precede the response" contract is a comment plus statement ordering; no assertion or type prevents a future reorder (integration tests do catch it).
- `agent.rs:693-696` — `unique_nonce`'s stated uniqueness target ("no collision within the lifetime of one history vec") is satisfied by a process-wide `AtomicU64` with no wrap handling.
- `lib.rs:599-601` — "A live run always has a `steer_tx`" is an invariant asserted in prose; the code degrades gracefully instead of checking it.
- `docs/MCP_DRIVEN_HOOKS.md:14-17` states hook output is injected as tool-result messages and JSON-encoded; that holds for `_Stop` (`agent.rs:641-651`) but **not** for `_PostCompact`, which is plain text inside a user message (`handoff.rs:85-92`, `handoff.rs:247-253`).
