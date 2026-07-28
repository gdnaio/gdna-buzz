## Module: buzz-agent — core loop, ACP wire types & handoff (`crates/buzz-agent/src`)
### Aspect: Conventions
#### Safety and lint attributes
`#![forbid(unsafe_code)]` sits at the crate root (`lib.rs:1`) and is the only occurrence of the token `unsafe` across the six files (`grep -c 'unsafe'` → `lib.rs:1`, all others 0), satisfying the AGENTS.md "No `unsafe` code" rule (`AGENTS.md:112`). There are **no** `#[allow(...)]` attributes anywhere in this group (`grep -n 'allow(' lib.rs agent.rs types.rs wire.rs handoff.rs main.rs` → 0 matches), so no lint is being suppressed in place of a fix.

#### Error handling
Two styles, split by layer:
- Domain errors use `AgentError` (`types.rs:224-231`) with `Display` (`types.rs:233-244`), `std::error::Error` (`types.rs:246`) and a `json_rpc_code()` mapping (`types.rs:249-256`). Every `Display` arm carries a lowercase namespace prefix — `invalid params:`, `llm:`, `llm auth:`, `mcp:`, `cancelled` — and call sites add a second-level prefix (`session/prompt: {reason}` at `lib.rs:659`, `prompt: exceeds …` at `agent.rs:71`).
- Startup and infrastructure paths use `Result<_, String>`: `Config::from_env` (`lib.rs:160`), `session_token` (`lib.rs:821`), `decode` (`lib.rs:479-481`). Fatal startup errors go through `die()`, which logs at `error` and `exit(2)` (`lib.rs:105-108`).

No `unwrap()`/`expect()` appears on a production path in this group. The only bare `unwrap()` is inside `#[cfg(test)] mod tests` (`lib.rs:880`, `wire.rs:250-290`); production code uses `unwrap_or`, `unwrap_or_else`, `unwrap_or_default` or `?` exclusively (`lib.rs:160-161`, `lib.rs:577`, `lib.rs:799`, `agent.rs:352`, `handoff.rs:143`, `handoff.rs:295-297`). This satisfies `AGENTS.md:113`.

Recoverable-but-ignored results are marked with the `let _ =` idiom consistently: `let _ = session.cancel_tx.send(true)` (`lib.rs:179`, `lib.rs:492`), `let _ = wire.send(...)` (`wire.rs:171`), `let _ = writer.await` (`lib.rs:181`), `let _ = stdout.flush()` (`wire.rs:235`).

#### Logging
Single global subscriber, stderr, ANSI off (`lib.rs:155-159`). No `target:` fields and no spans are used anywhere in the group. Level discipline:

| Level | Used for | Examples |
|---|---|---|
| `error!` | fatal or protocol-breaking | `die` (`lib.rs:106`), reader failure (`lib.rs:175`), unterminated frame at EOF (`wire.rs:184-187`), serialize failure (`wire.rs:228`) |
| `warn!` | degraded-but-continuing | catalog discovery fallback (`lib.rs:322-324`), tool-call cap (`agent.rs:243-246`), join errors (`agent.rs:435`, `agent.rs:463`), drain timeout (`agent.rs:469`), unrenderable steer (`agent.rs:277`), empty/failed handoff summary (`handoff.rs:56`, `handoff.rs:60`) |
| `info!` | state transitions | `session/set_model` (`lib.rs:528-532`), history truncation (`agent.rs:733-737`), handoff cap and handoff completion (`handoff.rs:35-38`, `handoff.rs:96-100`), prompt-budget drops (`handoff.rs:219-222`) |
| `debug!` | high-frequency noise | keepalive tick (`agent.rs:133`), dropped empty steer (`agent.rs:272`) |

Structured fields are used only where a machine reader benefits (`session_id`, `model_id` at `lib.rs:529-531`); everything else is interpolated into the message string.

#### Naming
- Wire emitters are `emit_<status>`: `emit_pending`, `emit_in_progress`, `emit_completed`, `emit_failed` (`agent.rs:552-616`).
- Fabricated data is prefixed `synthetic_`: `synthetic_tool_result` (`agent.rs:703`), `synthetic_hook_id` (`agent.rs:654`).
- Handler functions mirror the method name with the noun last: `session_new`, `set_model_session`, `steer_session`, `cancel_session`, `run_prompt` (`lib.rs:329`, `:503`, `:554`, `:487`, `:627`) — note `session_new` vs `set_model_session` is inconsistent word order.
- Byte/token limits are `MAX_*` consts or `max_*` config fields; the two history measures are named for their consumer (`estimated_bytes` = wire size, `context_pressure_bytes` = window pressure, `types.rs:29-47`).
- Pure policy math is extracted to free functions specifically so it is testable without a `RunCtx` — stated at `handoff.rs:344-346` and applied to `token_threshold`, `byte_fallback_threshold`, `handoff_prompt_budget_bytes`, `estimate_tokens_from_bytes`.

#### Module layout and visibility
`lib.rs` is both the crate root and the ACP server (dispatch, session lifecycle, handlers). `agent.rs` owns the turn loop, `handoff.rs` extends `RunCtx` with compaction via a second `impl RunCtx` block (`handoff.rs:30`), `wire.rs` owns framing, `types.rs` owns data. `main.rs` is a 6-line shim.

Visibility is inconsistent: `handoff.rs` uses `pub(crate)` correctly (`HandoffOutcome` `:19`, `maybe_handoff` `:31`, `clamp_bytes` `:300`) and `agent.rs` does for two items (`push_hook_outputs_as_tool_results` `:666`, `truncate_history` `:711`), but `RunCtx` and all its fields are plain `pub` inside a private module (`agent.rs:24-64`), as are all of `wire.rs`'s items. The effective visibility is identical; the declared intent is not.

#### Doc-comment discipline
Field- and rationale-level comments are unusually thorough — most `Session` fields (`lib.rs:72-102`) and `RunCtx` fields (`agent.rs:26-63`) carry multi-line explanations, and several comments encode the *why* of a past bug fix (`types.rs:5-13`, `agent.rs:147-160`, `handoff.rs:133-142`). Item-level doc comments on public API are the gap: 18 of 20 public items in `types.rs` have none, and `pub fn run()` (`lib.rs:110`) has none — see the API Surface aspect for the enumeration and the `AGENTS.md:114` rule it violates.

#### Test organization
| File | `#[cfg(test)] mod tests` | Tests |
|---|---|---|
| `lib.rs` | `:828` | 4 (`models_cache_does_not_pin_on_discovery_error` `:841`, three `databricks_discovery_failure_fallback_*` `:891`, `:919`, `:939`) |
| `types.rs` | `:276` | 4 (`:292`, `:303`, `:326`, `:346`) |
| `wire.rs` | `:240` | 4 (`:244`, `:259`, `:270`, `:283`) |
| `handoff.rs` | `:371` | 6 (`:378`, `:383`, `:388`, `:395`, `:403`, `:410`, `:422`) |
| `agent.rs` | none — `grep -n '#\[cfg(test)\]' agent.rs` → 0 matches | 0 |
| `main.rs` | none | 0 |

Unit tests carry a stated regression target in a doc comment above the `fn` (`lib.rs:833-839`, `types.rs:327-334`, `handoff.rs:411-413`) — the convention is "each test names the bug it locks down", which the crate README states explicitly for the integration suite (`README.md:299`). Integration tests live in `tests/` and use real subprocesses and real TCP listeners rather than mocks (`README.md:295-299`): `fake_llm.rs`, `regressions.rs`, `golden_transcripts.rs`, `hints_integration.rs`, `openai_auto_upgrade.rs`, `databricks_oauth.rs`, plus a `fake-mcp` binary declared in `Cargo.toml` because cargo cannot gate bins on `cfg(test)`.

#### File-size discipline
`lib.rs` 954 lines (827 before the test module), `agent.rs` 746, `handoff.rs` 430, `types.rs` 353, `wire.rs` 293, `main.rs` 6 — 2,782 total, matching the group's stated LOC. The repo's 1000-line hard ceiling is enforced only for `mobile/` (`AGENTS.md` mobile rules) and mirrored for desktop/web; no equivalent guard exists for Rust, and `config.rs` (2,701 lines) / `llm.rs` (2,894 lines) in the same crate exceed it freely.

#### Formatting and idiom notes
`rustfmt` default style throughout; `json!` macro for every outbound payload (`lib.rs:288-297`, `agent.rs:556-566`); `#[serde(rename_all = "camelCase")]` on every params struct except `InitializeParams`, which uses explicit `rename` (`wire.rs:39-44`) because one field is intentionally unused. Unused-but-deserialized fields are marked with a leading underscore rather than `#[allow(dead_code)]` (`wire.rs:42`).
