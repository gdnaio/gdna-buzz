## Module: buzz-acp — agent pool & lifecycle (`crates/buzz-acp/src`)
### Aspect: Technical Debt

#### File and function size

`pool.rs` is 5,620 lines: 3,649 lines of production code and 1,970 lines of in-file tests (`#[cfg(test)] mod tests` opens at `pool.rs:3650`). Within it, `run_prompt_task` is a single 948-line function (`pool.rs:1265-2212`) with 20 `send_prompt_result` exit sites and roughly ten nesting levels at the deepest point (the `Err(error)` arm of the control-cancel branch nested inside a `select!` inside a `match` inside `run_prompt_task`, `pool.rs:1894-1930`). The invariant "every path must send exactly one `PromptResult` or the main loop deadlocks" is stated in a comment (`pool.rs:1945`) and enforced by review, not by types — a `#[must_use]` result token or a consuming builder would make it structural.

The module's responsibilities are not separable as written. Beyond pooling, `pool.rs` owns: prompt-section composition (`pool.rs:1090-1231`), five relay REST fetchers (`pool.rs:2237-2784`), Nostr event parsing and signature verification for canvas (`pool.rs:2366-2477`), kind-0 profile parsing (`pool.rs:2594-2630`), NIP-AM metric encryption and publishing (`pool.rs:3322-3430`), NIP-25/NIP-09 reaction management (`pool.rs:3462-3648`), and a kind-9 dead-letter notice publisher (`pool.rs:3495-3535`). Nothing named `pool` is required for most of that.

#### The 4-line test file

`crates/buzz-acp/tests/pool_lifecycle_state.rs` contains no assertions. It is a re-inclusion shim:

```
#[allow(dead_code)]
#[path = "../src/pool_lifecycle.rs"]
mod pool_lifecycle;
```

(`tests/pool_lifecycle_state.rs:1-4`). Because integration targets compile with `cfg(test)`, this makes the nine tests already inside `pool_lifecycle.rs` (`:139`, `:150`, `:174`, `:203`, `:226`, `:244`, `:269`, `:283`, `:293`) compile and run a **second** time in a second binary. It adds zero coverage over `cargo test -p buzz-acp --lib`, doubles their runtime, and requires a blanket `#[allow(dead_code)]` because most of the state machine is unused from an integration target's perspective. It is also the crate's **only** file under `tests/` — `crates/buzz-acp/tests/` contains nothing else.

#### Untested surface

The pool's actual pooling logic has **no tests in `pool.rs`**. Searching the test region (`pool.rs:3650-5620`) for each `AgentPool` method returns zero matches for: `try_claim`, `return_agent`, `live_count`, `slot_alive`, `any_idle`, `has_session_for`, `send_steer`, `switch_idle_agent_model`, `invalidate_channel_sessions`, `from_slots`, and `IdleSwitchResult`. Specifically untested behaviours:

| Untested | Site |
|---|---|
| Affinity pass wins over first-idle | `pool.rs:560-568` |
| `try_claim` returns `None` when all checked out | `pool.rs:571-574` |
| Double-return `BUG:` branch and slot overwrite | `pool.rs:579-589` |
| `live_count` = idle + checked-out | `pool.rs:610-614` |
| `slot_alive` true for a checked-out agent | `pool.rs:686-693` |
| `send_steer` → `PromptCompleted` when no task in flight | `pool.rs:650-652` |
| `send_steer` → `Transport` on `Full`/`Closed` | `pool.rs:657-659` |
| All three `IdleSwitchResult` outcomes | `pool.rs:732-762` |
| `invalidate_channel_sessions` return count | `pool.rs:707-720` |
| `from_slots` index-preservation invariant | `pool.rs:541-556` |

`from_slots` is used 14 times in `lib.rs` tests, so it is exercised incidentally as a fixture; the invariant it exists to protect is never asserted. `run_prompt_task` itself has no test — every test in `pool.rs` targets a pure helper, a guard, or `SessionState`.

Lifecycle-state coverage is better but incomplete. Covered transitions: `Listening → Waking`, `Waking → Ready`, `Waking → Failed`, `Failed → Waking` (due and not-due), `Ready → Listening` via `take_ready`, stale-attempt rejection, non-`Waking` `complete_wake` rejection, `cancel_wake`. Not covered:

- `cancel_wake` with a **mismatched** attempt returning `false` (`pool_lifecycle.rs:91-93`) — only the matching case is tested (`:271`).
- `Ready` → `start_wake_if_due` returning `None` is asserted only after a retry cycle (`:218-221`), never directly from a first-attempt success.
- `take_ready` on `Waking` or `Failed` returning `None` without mutating state (`pool_lifecycle.rs:63-66`) — only the `Ready`-then-empty sequence is tested (`:288-289`).
- `waking_attempt()` / `retry_at()` / `failed_error()` returning `None` in non-matching states (`pool_lifecycle.rs:70-89`).
- The `Failed` → `start_wake_if_due(true, now)` path when `now` is exactly `retry_at - 1ns`.
- Interaction with the caller's `pool_ready` flag: nothing tests that a `Listening` state after `take_ready` cannot start a second wake, because that suppression lives in `lib.rs:1714`, not in the machine.

#### Dead code and stale annotations

Three `#[allow(dead_code)]` attributes in `pool.rs`:

| Site | Target | Status |
|---|---|---|
| `pool.rs:73` | `AgentModelCapabilities` | **Stale-ish and mis-documented.** Both fields *are* read in-process at `pool.rs:750-751`. The comment says "Scaffolding for desktop integration — fields read via serde", and the type doc (`pool.rs:70-72`) says they are "read by the desktop's `get_agent_models` Tauri command (Phase 3)". That command is an out-of-process Tauri handler (`desktop/src-tauri/src/commands/agent_models.rs:29`) that cannot read this struct, and neither field derives `Serialize`. |
| `pool.rs:335` | `SteerError` | Explained: variants are `Debug`-rendered through `?ack` in `lib.rs` and the lint cannot follow that (`pool.rs:329-334`). |
| `pool.rs:405` | `PromptOutcome` | Unexplained. All six variants are constructed and matched; the payload of `Ok(_)` is never read at either `lib.rs` match arm (`lib.rs:3183`, `:3231`), which is the likely reason. |

Three `#[cfg(test)]` helpers sit in the production region rather than in the test module: `parse_thread_response` (`pool.rs:2787`), `parse_dm_response` (`pool.rs:2828`), `pct_encode` (`pool.rs:3441`). The first two parse a **legacy REST shape** that production no longer requests — production uses `parse_nostr_thread_response`/`parse_nostr_dm_response` (`pool.rs:2892`, `:2938`) — yet nine tests still assert against the retired format (`pool.rs:3879`, `:3916`, `:3951`, `:3961`, `:3968`, `:4007`, `:4034`, `:4063`, `:4072`). `pct_encode` has six tests (`pool.rs:4213`–`:4241`) and zero production callers; the URL-path encoding it existed for is gone now that reactions go through signed events.

`PromptContext::heartbeat_prompt` (`pool.rs:501`) is carried on the shared context but never read in `pool.rs`. `PoolLifecycle::failed_error()` (`pool_lifecycle.rs:84`) has one caller, a `debug_assert_eq!` (`lib.rs:2565`), so it is dead in release builds.

#### TODO / FIXME / unsafe / unwrap

- `TODO`, `FIXME`, `XXX`, `HACK`: **0** occurrences across both files.
- `unsafe`: **0**; `#![deny(unsafe_code)]` is crate-wide (`lib.rs:1`).
- `unwrap()`/`expect()` in production paths: **3** total. `pool.rs:573` (`take().unwrap()` guarded by a preceding `position(..is_some())`) and `pool.rs:3399-3400` (two `Tag::parse(..).expect(..)` on hex strings). The remaining 93 of the file's 96 occurrences are inside `#[cfg(test)]`.

#### Doc-comment corruption

Three confirmed doc-block splices, each of which mis-attributes documentation and leaves a public item undocumented:

1. `PromptContext`'s doc block is glued to the front of `ChannelInfoResolver`'s: `pool.rs:426-429` reads "Immutable config subset shared (via `Arc`) by all spawned prompt tasks… Avoids cloning the full config into every task." immediately followed at `pool.rs:430` by "Shared channel-metadata resolver…", and the actual `pub struct PromptContext` at `pool.rs:482` has **no doc comment at all**.
2. `apply_permission_mode`'s doc is split across two items: the sentence begins at `pool.rs:1008-1011` ("…falls back to its default permission mode (`"default"`), which still works via") and is interrupted by `agent_supports_mode`'s doc + body (`pool.rs:1012-1026`); the continuation "per-tool auto-approval in `handle_permission_request`." opens the function's own doc block mid-sentence at `pool.rs:1028`.
3. `parse_dm_response` carries two contradictory summary lines: "Parse the DM messages REST response into a `ConversationContext::Dm`." followed by "Parse the legacy REST DM response (used in tests only)." (`pool.rs:2824-2826`).

Other public-item doc gaps under the repo's "new public API must have doc comments" rule: `ChannelInfoResolver::new` (`pool.rs:442`), `ChannelInfoResolver::resolve` (`pool.rs:464`), `AgentPool::task_map` (`pool.rs:616`), `task_map_mut` (`pool.rs:620`), `result_tx` (`pool.rs:664`), `agents_mut` (`pool.rs:695`).

#### Doc drift vs `ARCHITECTURE.md` and `README.md`

`ARCHITECTURE.md:658-667` lists a per-module LOC table for `buzz-acp` that no longer matches any file:

| `ARCHITECTURE.md` claim | Actual | Delta |
|---|---|---|
| `pool.rs` — 2,253 | 5,620 | +150 % |
| `main.rs` — 2,457, "Event loop, pool orchestration, heartbeat" | 3 lines; the event loop is in `lib.rs` (6,570) | wrong file entirely |
| `relay.rs` — 3,143 | 6,233 | +98 % |
| `queue.rs` — 2,565 | 4,759 | +85 % |
| `config.rs` — 1,903 | 2,709 | +42 % |
| `acp.rs` — 1,785 | 3,717 | +108 % |
| `filter.rs` — 814 | 787 | −3 % |

`pool_lifecycle.rs`, `setup_mode.rs`, `usage.rs`, `engram_fetch.rs`, and `observer.rs` are absent from that table. `ARCHITECTURE.md:668-672` also omits the deferred/lazy pool entirely — "Pool of 1–32 agent subprocesses with claim/return lifecycle" and "Crash recovery: agent subprocess crashes are detected and the agent is respawned" describe only the eager path, with no mention of `--lazy-pool`, the wake state machine, the per-slot circuit breaker, or the `CancelDrainTimeout` class.

`crates/buzz-acp/README.md` is accurate on the pool: `--agents` range and default (`README.md:117`), `--lazy-pool` including the retry-with-backoff behaviour (`README.md:118`), single shared bot identity for all N agents (`README.md:197`), heartbeat dropped when all agents are busy (`README.md:204`), one prompt in flight per channel (`README.md:251`). It does not document the steer/interrupt/rotate/switch-model control signals or the circuit breaker.

`.env.example` drift is covered in the Configuration aspect: eight pool-relevant `BUZZ_ACP_*` vars are missing and the deprecated `BUZZ_ACP_TURN_TIMEOUT` is documented in place of `BUZZ_ACP_IDLE_TIMEOUT` (`.env.example:152`).

#### Structural risks carried in comments rather than code

- `return_agent` knowingly overwrites an occupied slot and leaks the resident `AcpClient` rather than leaking the slot (`pool.rs:579-589`). Both outcomes are bad; the choice is documented but not measured — nothing counts how often the `BUG:` branch fires.
- `task_map_mut` and `agents_mut` expose the invariant-bearing structures for unrestricted mutation (`pool.rs:620`, `:695`), so the index invariant is `lib.rs`'s responsibility.
- `result_tx` is unbounded (`pool.rs:542`); safe only because sends are gated by pool width. A future non-agent sender would remove that bound silently.
- `agent_name == "goose"` is a string comparison used as a capability switch in three places (`pool.rs:176`, `:826`, and via `has_system_prompt_support`), rather than a negotiated capability.
- `ctx.cwd` defaults to `/` when `current_dir()` fails (`lib.rs:1547-1550`); the only mitigation is that `workspace_section` declines to name `/` in the prompt (`pool.rs:1165-1178`). The `session/new` `cwd` parameter is still sent as `/`.
- `buzz-persona` is declared as a dependency (`Cargo.toml:22`) with zero references in `crates/buzz-acp/src` — a dependency edge with no code behind it. If persona-driven spawning is intended to land here, the seam does not exist yet: `PoolStartup` carries one command/args/env triple for the whole pool (`lib.rs:3717-3725`) and `ctx.mcp_servers` is a single shared list built only from `config.mcp_command` (`lib.rs:4141-4184`).
