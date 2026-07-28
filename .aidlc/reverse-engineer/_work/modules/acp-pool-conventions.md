## Module: buzz-acp — agent pool & lifecycle (`crates/buzz-acp/src`)
### Aspect: Conventions

#### Ownership over locking

The dominant convention is move semantics instead of shared mutability. `AcpClient` is not `Clone`, so an agent is either in its slot or inside a task — never both (`pool.rs:206-210`, `:22`). Cross-task communication is by channel, not by lock:

| Channel | Type | Purpose | Site |
|---|---|---|---|
| `result_tx`/`result_rx` | `mpsc::unbounded` | task → main loop, carries the `OwnedAgent` back | `pool.rs:214-215`, `:541-548` |
| `control_tx` | `oneshot::Sender<ControlSignal>` | main loop → task, one signal per turn | `pool.rs:67` |
| `steer_tx` | `mpsc::Sender<SteerRequest>` capacity 1 | main loop → read loop | `pool.rs:75`, install `lib.rs:2937-2938` |
| `ack_tx` | `oneshot::Sender<SteerAck>` | read loop → main loop | `pool.rs:332` |

Two locks exist, both `std::sync` rather than tokio, both held only across non-await code: `ChannelInfoResolver`'s `RwLock<HashMap<..>>` (`pool.rs:437`, guard dropped before the await at `:474`) and `LivenessState`'s `Mutex` (`pool.rs:3206-3209`). Both use the same poison-tolerant idiom — `match lock() { Ok(g) => g, Err(poisoned) => poisoned.into_inner() }` (`pool.rs:3189-3192`, `:3241-3244`, `:3252-3255`) — while the resolver instead swallows poison via `.ok()`/`if let Ok(..)` and silently skips the cache (`pool.rs:468-471`, `:475-477`).

`rx_and_join_set` (`pool.rs:668-675`) exists purely to hand out two disjoint `&mut` borrows for one `select!` — the module's chosen alternative to wrapping the pool in a lock.

#### `select!` discipline

Every `select!` in the module is `biased` and orders the prompt arm first so the completion path wins ties (`pool.rs:1828-1830`, `:1841-1843`). The control arm's `else` branch documents *why* that ordering makes a state unreachable rather than defending against it (`pool.rs:1937-1943`). The caller mirrors the convention: `biased` in the main loop with explicit `if` guards on arms that would otherwise spin on an empty `JoinSet` or a past deadline (`lib.rs:1822-1861`).

#### RAII for cross-cutting turn effects

Anything that must happen on *every* exit path — including panic — is a drop guard, not a call at each return site: `TurnCompletionGuard` for `turn_completed` (`pool.rs:3267-3302`), `LivenessGuard` for the liveness task (`pool.rs:3228-3264`), `ReactionGuard` for reaction cleanup (`pool.rs:3111-3141`). This is what makes 20 `send_prompt_result` exit points tolerable. Declaration order is used as an ordering primitive and is commented as such (`pool.rs:1305-1308`).

`ReactionGuard::drop` uses `tokio::runtime::Handle::try_current()` rather than assuming a runtime, and documents the fallback as harmless (`pool.rs:3126-3139`).

#### Error handling

`Result` + `?` on the internal seams (`create_session_and_apply_model`, `apply_model_switch`, `apply_permission_mode` all return `Result<_, AcpError>`); the outer `run_prompt_task` returns `()` and encodes every failure as a `PromptOutcome` sent through the channel (`pool.rs:405-431`). Zero `unwrap()`/`expect()` in the module's production paths except:

- `pool.rs:573` — `self.agents[i].take().unwrap()` immediately after `position(|slot| slot.is_some())`, locally provable.
- `pool.rs:3399-3400` — two `Tag::parse(..).expect("p tag")` / `expect("agent tag")` on hex-string tags.

`#![deny(unsafe_code)]` is set crate-wide (`lib.rs:1`); there is no `unsafe` in either file.

A recurring convention is the **error-class split**: transport-class `AcpError` variants (`Io`, `WriteTimeout`, `Timeout`, `Protocol`, `AgentExited`) propagate so the caller respawns, while application-class errors are logged and swallowed so the turn proceeds. The pattern appears verbatim three times — `apply_model_switch` (`pool.rs:975-987`), `apply_permission_mode` (`pool.rs:1051-1063`), and the caller-side classification in `handle_prompt_result` (`lib.rs:3347-3353`) — with the same comment wording each time.

Classification decisions that couple an error to a downstream fate are extracted into a single named seam so tests can cross the exact boundary: `classify_control_cancel_failure` returning `ControlCancelFailure { outcome, retry_batch, invalidate_all }` (`pool.rs:3007-3056`), and `requeue_cancelled_batch` mapping signal → `CancelReason` (`pool.rs:2981-3004`). The doc explicitly frames these as "the single production seam" (`pool.rs:3013-3019`).

Fail-open is the default for every optional enrichment: core memory, canvas, channel metadata, thread/DM context, and profiles all return `Option` and inject nothing on failure, each with the failure modes enumerated in the doc comment (core: `pool.rs:1364-1378`; canvas: `pool.rs:2297-2303`).

#### Naming

- States are gerund/adjective, not `*_STATE`: `Listening`, `Waking`, `Ready`, `Failed` (`pool_lifecycle.rs:14-25`).
- Transitions read as guarded imperatives: `start_wake_if_due`, `complete_wake`, `cancel_wake`, `take_ready` (`pool_lifecycle.rs:37`, `:99`, `:91`, `:60`). The `_if_due` / `take_` prefixes signal "may be a no-op" and "consumes".
- Invalidation is a verb family: `invalidate`, `invalidate_channel`, `invalidate_all` (`pool.rs:109`, `:123`, `:131`).
- Outcome types are suffixed by role: `PromptOutcome`, `PromptResult`, `PromptSource`, `PromptContext`, `TimeoutKind`, `IdleSwitchResult`, `SteerAck`/`SteerError`.
- Timeout constants are `<SCOPE>_TIMEOUT` / `<SCOPE>_GRACE` / `<SCOPE>_WINDOW` (`pool.rs:45`, `:780`, `:786`, `:793`, `:796`, `:3437`).
- Attempt counters are `attempt: u32` and always advanced with `saturating_add` (`pool_lifecycle.rs:50`).

#### Tracing

78 tracing calls in the production region; 50 carry an explicit `target:`. Targets are namespaced by concern, not by module path:

| Target | Count |
|---|---|
| `pool::prompt` | 14 |
| `canvas::fetch` | 11 |
| `pool::session` | 10 |
| `pool::model` | 5 |
| `pool::permission` | 4 |
| `pool::metrics` | 4 |
| `engram::core` | 2 |

The remaining ~28 calls use the default target, so `pool.rs` events cannot be filtered as one unit — notably the `BUG: return_agent` error (`pool.rs:584-587`), the `no batch and no prompt_text` error (`pool.rs:1788`), the reaction and `post_failure_notice` paths (`pool.rs:3466-3612`), and the profile-lookup debugs (`pool.rs:2663`, `:2667`). Level discipline is consistent: `error!` for invariant violations and process-fatal conditions, `warn!` for degraded-but-continuing, `info!` for lifecycle milestones, `debug!` for per-event noise; `log_stop_reason` (`pool.rs:3058-3082`) centralizes level choice per stop reason. Structured fields are preferred over interpolation on the fetch paths (`channel = %cid`, `timeout_ms = ..`) but interpolation is used freely elsewhere (`pool.rs:982-985`).

#### Prompt composition style

Section assembly is a chain of small total functions over `Option<String>` — `framed_system_prompt` → `with_team` → `with_core` → `with_canvas` (`pool.rs:1137`, `:1180`, `:1199`, `:1213`) — composed in one expression at `pool.rs:816-826`. Each function handles all four `(Some/None, Some/None)` cases explicitly and each has a dedicated unit test. Every appended block carries its own `[Header]` so the desktop can re-split the combined value (`pool.rs:1124-1128`).

#### Testing conventions

Unit tests live in-file under `#[cfg(test)] mod tests` (`pool.rs:3650`), 1,970 lines. Three helper functions are `#[cfg(test)]`-gated but declared in the production region: `parse_thread_response` (`pool.rs:2787`), `parse_dm_response` (`:2828`), `pct_encode` (`:3441`). Async tests that involve time use `#[tokio::test(start_paused = true)]` (`pool.rs:4686`, `pool_lifecycle.rs:138`). `SessionState` was deliberately split out of `OwnedAgent` for testability (`pool.rs:83-84`), and `has_channel_state` is a `#[cfg(test)]` accessor on it (`pool.rs:141`).
