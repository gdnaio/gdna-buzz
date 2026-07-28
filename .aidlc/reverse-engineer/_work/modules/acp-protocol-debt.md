## Module: buzz-acp — ACP protocol, config & setup mode (`crates/buzz-acp/src`)
### Aspect: Technical Debt

#### File sizes

| File | Lines | Test lines | Non-test |
|---|---|---|---|
| `acp.rs` | 3,717 | from `acp.rs:2008` | ~2,007 |
| `config.rs` | 2,709 | from `config.rs:1328` | ~1,327 |
| `setup_mode.rs` | 1,135 | from `setup_mode.rs:651` | ~650 |
| `base_prompt.md` | 136 | — | — |

For context within the crate: `lib.rs` 6,570, `relay.rs` 6,233, `pool.rs` 5,620, `queue.rs` 4,759. The repo enforces a 1,000-line ceiling on mobile (`mobile/scripts/check-file-sizes.mjs`, per `AGENTS.md`) and the desktop/web equivalents; **no such guard exists for Rust crates**, and seven of thirteen files in this crate exceed 1,000 lines.

#### Zero in-code debt markers

`grep -rn 'TODO\|FIXME\|HACK\|XXX' crates/buzz-acp/src/` returns **0** matches across the entire crate. Known-incomplete work is instead marked with `#[allow(dead_code)]` plus a prose comment, which no lint or grep surfaces:

| Item | Marker | Line |
|---|---|---|
| `drain_stale_responses` | `#[allow(dead_code)] // Scaffolding for future model-switch timeout cleanup; not yet wired.` | `acp.rs:1022` |
| `session_new` (id-only wrapper) | `#[allow(dead_code)] // Public API — callers outside the harness may use this.` | `acp.rs:591` |
| `active_run_id()` | `#[cfg_attr(not(test), allow(dead_code))]` — production reads the field directly | `acp.rs:768` |
| `available_commands_update` | "Logged for observability; UI surfacing is a follow-up." | `acp.rs:1574-1575` |
| `handle_setup_membership`'s 5th param | `_initial_channel_ids` — accepted, unused | `setup_mode.rs:568` |

#### Dead and inert code

| Item | Status |
|---|---|
| `buzz-persona` dependency | Declared at `Cargo.toml:22` — and uniquely by `path = "../buzz-persona"` rather than `workspace = true` like every other internal dep — with **zero** references anywhere under `crates/buzz-acp/src`. Persona resolution moved to the desktop instance snapshot (`config.rs:943-944`); the dep was never removed |
| `Config::persona_env_vars` doc comment | Says "Populated from persona pack resolution" (`config.rs:534`) but the only writer is `codex_network_env` (`config.rs:951-957`) |
| `Config::allowed_respond_to` | Validated at startup (`config.rs:919-937`), then read only by `summary()` (`config.rs:1019-1025`). Operators reading the flag's doc comment ("the harness rejects startup if `--respond-to` is not in this list", `config.rs:456-458`) get exactly that and nothing more — there is no runtime re-check |
| `BUZZ_API_TOKEN` | Written by `propagate_legacy_env_vars` (`config.rs:718`), never read. The README still lists it as "required if relay enforces token auth" (`crates/buzz-acp/README.md:107`) |
| `codex_network_env`'s parsed host | Extracted and logged (`config.rs:654-672`) but not used in the emitted value, which is a hardcoded constant (`config.rs:674-677`). The URL parse survives purely as a fail-closed guard |

#### Duplicated logic

| Duplication | Sites | Risk |
|---|---|---|
| The two read loops | `read_until_response` (`acp.rs:1074-1166`) vs `read_until_response_with_idle_timeout` (`acp.rs:1198-1523`) | ~90 lines of shared frame-parse / observe / id-match / method-dispatch. A fix to one (e.g. the `-32601` reply, `acp.rs:1147-1156` vs `acp.rs:1497-1506`) must be mirrored by hand |
| Channel-filter resolution | `resolve_channel_filters` Config branch (`config.rs:1186-1231`) vs `resolve_dynamic_channel_filter` Config branch (`config.rs:1275-1313`) | The second carries the comment "Same merge logic as `resolve_channel_filters()` Config branch" (`config.rs:1276`) — the invariant is a comment, not a shared function |
| Validation re-implemented in tests | `validate_heartbeat_interval` (`config.rs:1959`), `validate_turn_liveness` (`config.rs:2001`), `resolve_idle_timeout` (`config.rs:2214`), `parse_allowed_respond_to` / `check_allowed_respond_to` (`config.rs:2476`, `:2490`) | These are **copies** of logic inlined in `from_args`, not calls into it. Changing `from_args` leaves the tests green. Only the three `allowed_respond_to_full_path_*` tests (`config.rs:2588`, `:2618`, `:2639`) exercise the real path via `CliArgs::try_parse_from` |
| `AcpAvailabilityStatus` | `setup_mode.rs:58-71` mirrors the desktop enum and `api/types.ts` by hand, justified at `setup_mode.rs:52-57`. Divergence is caught only by the four round-trip tests (`setup_mode.rs:1081-1136` region), which assert the four literals the desktop currently sends |
| Default kind lists | `[STREAM_MESSAGE, WORKFLOW_APPROVAL_REQUESTED, STREAM_REMINDER]` in `config.rs:1161-1165` and `config.rs:1262-1267`, but only the **first two** in `setup_mode.rs:526`. Setup mode silently ignores reminder events |

#### Untested surface

`acp.rs` has 76 tests (27 async), but coverage is concentrated on pure helpers — `StopReason::from_str`, permission-option lookup by kind, `agent_error_from_json` (`acp.rs:3458`, `:3475`), and 13 `build_codex_config_env` cases (`acp.rs:3498-3712`). Not covered:

| Surface | Why it is hard to reach |
|---|---|
| `AcpClient::spawn`'s env-injection precedence | Requires mutating the process environment; `build_codex_config_env` is tested in isolation but the `if std::env::var(key).is_err()` loop at `acp.rs:455-457` is not |
| `stderr` inheritance, `process_group(0)`, `configure_no_window` | Platform-gated, no test asserts them |
| `shutdown()`'s 5 s expiry path | Would need a SIGKILL-resistant child; the "abandoning" branch (`acp.rs:396`) is untested |
| `Drop`'s `try_wait` reap | Untested |
| `MAX_LINE_SIZE` enforcement | No test drives a >10 MB line through the codec |
| Non-matching-id skip behaviour | The silent-skip rule (`acp.rs:1123-1130`) is documented (`acp.rs:1010-1015`) but has no test |
| `-32601` reply to unknown agent requests | No test asserts the harness answers rather than hangs |
| Steer ack drain on every early return | Seven drain sites (`acp.rs:1263`, `:1362`, `:1369`, `:1374`, `:1382`, `:1449`, `:1454`); the invariant "callers are never left hanging" (`acp.rs:1213-1215`) is carried only by the comment |
| `run_setup_listener` | 22 setup-mode tests all target `nudge_body`, `from_raw_env_value`, or `should_nudge_for_event`. The 170-line event loop (`setup_mode.rs:309-480`), `handle_setup_membership`, `publish_setup_nudge`, and `build_setup_subscription_rules` have **no tests** — including the reply-threading choice at `setup_mode.rs:608-624` |
| Wildcard-`kinds` REQ | Nothing asserts that `--subscribe all` without `--kinds` yields `kinds: None`, or warns about it. `test_all_mode_wildcard` (`config.rs:1630`) asserts the wildcard is *produced*, treating it as correct rather than as the p-gate hazard `AGENTS.md § Common Gotchas #2` describes |

#### Invariants carried only in comments

| Invariant | Recorded at | Enforcement |
|---|---|---|
| `initialize` pins `protocolVersion: 2` as a deliberate squat "ahead of the upstream ACP RFD. Revisit when that RFD merges" | `acp.rs:536-537` | None — no tracking issue reference, no version gate |
| Permission write-then-flag ordering prevents an unbounded deadlock | `acp.rs:1735-1748` | Ordering only; a future refactor could reintroduce it |
| The pre-select deadline check is what keeps `biased` from defeating the hard cap | `acp.rs:1189-1194`, `acp.rs:1257-1262` | Comment only; deleting the check compiles and passes tests |
| `expectedRunId` must be sampled at write time, not at dispatch | `acp.rs:1300-1310`, `acp.rs:1786-1789` | Comment + code placement |
| Callers must `await shutdown()`; `kill_on_drop` is best-effort | `acp.rs:373-375`, `acp.rs:422-424`, `acp.rs:1955-1956` | Comment only |
| `take_turn_usage` must be called at most once per turn | `acp.rs:779-781` | Comment; the second call silently returns `None` |
| `install_steer_rx`'s one-receiver rule | `acp.rs:794-799` | The only comment-invariant with a runtime guard — an `assert!` that **panics** in production (`acp.rs:801-805`) |
| Config mode ignores `--channels` "per CLI contract" | `config.rs:1229-1232`, `config.rs:1248-1251` | Warning at startup, then divergent behaviour between two functions |

#### Error-handling gaps

- `--system-prompt-file`, `--heartbeat-prompt`, and `--heartbeat-prompt-file` have **no size cap**, while `--base-prompt-file` is capped at 1 MB (`config.rs:780-790`). All three go into every prompt.
- `--agent-owner` is trimmed and lowercased but not validated as 64-char hex (`config.rs:1003`), unlike `--respond-to-allowlist` entries which are strictly validated (`config.rs:558-572`). An owner typo silently produces an owner that never matches, and with the default `respond_to = owner-only` the agent answers nobody.
- `--max-turns-per-session` uses `value_parser!(u32)` with no range (`config.rs:372-373`), unlike its neighbours which both carry `.range(...)`.
- Invalid `--channels` entries warn and are dropped (`config.rs:816-824`) rather than erroring, so a single typo silently narrows the agent's scope.
- `parse_stop_reason` treats an unknown `stopReason` as a hard `Protocol` error (`acp.rs:1762-1763`), so any future ACP stop reason breaks the turn rather than degrading.

#### Documentation drift

`ARCHITECTURE.md:658-667`'s LOC table is wrong for every row that touches this crate:

| Row | Claimed | Actual |
|---|---|---|
| `relay.rs` (`:660`) | 3,143 | 6,233 |
| `queue.rs` (`:661`) | 2,565 | 4,759 |
| `main.rs` (`:662`) "Event loop, pool orchestration, heartbeat" | 2,457 | **3** — `main.rs` is now just `fn main() { buzz_acp::run() }`; the event loop moved to `lib.rs` |
| `pool.rs` (`:663`) | 2,253 | 5,620 |
| `config.rs` (`:664`) | 1,903 | 2,709 |
| `acp.rs` (`:665`) | 1,785 | 3,717 |
| `filter.rs` (`:666`) | 814 | 787 |

Further drift beyond the table:

- The table omits `lib.rs` (6,570 — the largest file and the actual event loop), `setup_mode.rs`, `usage.rs`, `observer.rs`, `engram_fetch.rs`, and `pool_lifecycle.rs`. Setup mode, the observer feed, and usage metering are absent from the architecture description entirely.
- `ARCHITECTURE.md:656` describes the harness as queueing `@mention` events and says "At most one prompt is in-flight per channel", with no mention of the steer/interrupt cancel modes that are now the **default** (`config.rs:356`).
- `crates/buzz-acp/README.md:105` gives the idle-timeout default as `620`; the code says `900` (`config.rs:27`). The constant's own doc comment (`config.rs:17-26`) explains the 900 choice in detail, so the README is the stale side.
- `crates/buzz-acp/README.md:107` documents `BUZZ_API_TOKEN` as a live setting; it is never read.
- `.env.example:152` steers operators to the hidden, deprecated `BUZZ_ACP_TURN_TIMEOUT=320` and never mentions `BUZZ_ACP_IDLE_TIMEOUT`. `.env.example` covers ~20 of 43 env vars, omitting the whole author-gate group (`BUZZ_ACP_RESPOND_TO`, `..._ALLOWLIST`, `ALLOWED_RESPOND_TO`), `BUZZ_ACP_PERMISSION_MODE`, the base-prompt controls, and `BUZZ_ACP_SETUP_PAYLOAD`.
- `.env.example:221` documents `BUZZ_ACP_EVENT_BUFFER=256`, which is real but read in `relay.rs:36` — the only documented env var with no `config.rs` flag behind it.

#### Structural observations

- `send_request` wraps write and read in **two sequential 60 s timeouts** rather than one budget, so a slow-but-progressing agent can consume ~90 s per non-prompt RPC (`acp.rs:974-976`, `:996-1003`). With `initialize` + `session/new` + optional system-prompt + model set, session establishment has no single bound.
- `SessionNewResponse.raw` (`acp.rs:1831`) and both model extractors (`acp.rs:1851`, `:1866`) return untyped `serde_json::Value`, pushing schema knowledge into string literals spread across `acp.rs` and `pool.rs` (`"configOptions"`, `"category"`, `"options"`, `"value"`, `"models"`, `"availableModels"`, `"modelId"`). A schema change fails silently as a `None` rather than a deserialization error.
- `protocolVersion` is read as `.as_u64().unwrap_or(1)` at `lib.rs:3776` and `lib.rs:3864`, so a missing or non-numeric field silently selects legacy v1 prompt composition. `acp.rs` contributes nothing here — it neither validates the field nor stores the negotiated version on `AcpClient`, so the two call sites are the only guard and they duplicate the same lenient parse.
- `Config` has 40 fields and is constructed literally in three places: `from_args` (`config.rs:961-1007`), `config.rs:1334-1376`, and `lib.rs:4979`/`lib.rs:5145`. Every new field requires four edits, and the test fixture uses non-default values for `respond_to` (`Anyone`) and `multiple_event_handling` (`Queue`), so fixture-based tests do not exercise the shipped posture.
