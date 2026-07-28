## Module: buzz-agent — MCP registry, OAuth, hints & catalog (`crates/buzz-agent/src`)
### Aspect: Conventions

#### Error handling

One error type for everything: `crate::types::AgentError` (`types.rs:203-211`). All five files return `Result<_, AgentError>` and never define a local error enum — `grep -n 'thiserror\|impl std::error::Error' mcp.rs auth.rs hints.rs builtin.rs catalog.rs` returns zero matches.

Variant selection is by *module* rather than by *cause*: `mcp.rs` uses `Mcp` for all 23 failure sites, `auth.rs` and `catalog.rs` use `Llm` for all 32 of theirs, and only three OAuth sites reach for the semantically meaningful `LlmAuth` (`auth.rs:341`, `355`, `416`). The practical effect is that a missing OAuth token is distinguishable on the wire (`-32001`) but a broken discovery endpoint is not (`-32000`, same as a provider outage) — mapping at `types.rs:249-256`.

Error message style is `"<stage> <subject>: <cause>"`, built with `format!` and interpolated source errors: `spawn {name}: {e}` (`mcp.rs:738`), `init {name}: {e}` (`mcp.rs:760`), `oauth discovery: {e}` (`auth.rs:166`), `Databricks model discovery HTTP {status}: {body}` (`catalog.rs:112-114`). Messages are written for the model as much as for the operator — several include a remediation hint: "Try again later or use a different tool" (`mcp.rs:143`), "run `buzz-agent auth databricks` first" (`auth.rs:417`), "Available: {available:?}" (`builtin.rs:62`, `builtin.rs:130`, `builtin.rs:168-170`).

Two deliberate no-error conventions:
- `hints.rs` never returns `Result`. Every filesystem failure is swallowed with `let Ok(..) = .. else { continue/return }` (`hints.rs:65-67`, `hints.rs:111-113`, `hints.rs:130-132`, `hints.rs:171-173`, `hints.rs:178-180`, plus the `match … Err(_) => continue` at `hints.rs:183-186`). Discovery is best-effort by design, but the consequence is that an unreadable `AGENTS.md` or a permission error on a skills directory is indistinguishable from absence, with no log line.
- `builtin.rs` never returns `Err`; failures become `ToolResult { is_error: true }` via a single helper (`error_result`, `builtin.rs:232-238`), because the value is destined for the model, not the transport.

`AgentError` is not `Clone`, so `mcp.rs` re-formats rather than propagates in the hook path (`mcp.rs:367-385` drops errors entirely).

#### Panic discipline

No `unwrap()`, `expect()`, or `panic!` in any production path of the five files. Verified per file by scanning only the region above `#[cfg(test)]`: the single match is `hook_timeouts.lock().unwrap_or_else(|e| e.into_inner())` (`mcp.rs:387`), which is poison recovery, not a panic. This satisfies the `AGENTS.md` rule "Do not introduce new `unwrap()` or `expect()` in production paths". Test code uses `unwrap()` freely, which is idiomatic here.

Poison handling is inconsistent within one function: `call_hooks` recovers from a poisoned mutex at `mcp.rs:387` but silently skips the update at `mcp.rs:375-377` and `mcp.rs:399-401` (`if let Ok(mut counts) = …`). Same lock, three sites, two policies.

#### `unsafe` and lint attributes

`lib.rs:1` is `#![forbid(unsafe_code)]`, and none of the five files contains `unsafe` or a `SAFETY:` comment (`grep -n 'unsafe\|SAFETY' mcp.rs auth.rs hints.rs builtin.rs catalog.rs` → zero matches). Platform-specific behaviour that would normally require `unsafe` is expressed through safe wrappers instead: `Command::process_group(0)` (`mcp.rs:733`) rather than a `pre_exec` closure, `nix::sys::signal::killpg` (`mcp.rs:846-848`) rather than raw `libc::killpg`, and `CommandExt::creation_flags` (`mcp.rs:998-999`) on Windows. The crate README's claim of "`setpgid(0,0)` in `pre_exec`" describes a mechanism the crate's own lint would reject.

The only `#[allow]` in the group is `#[allow(clippy::too_many_arguments)]` on `do_call` (`mcp.rs:555`), which takes 8 parameters. There is no `#[allow(dead_code)]` anywhere in the group (`grep -n '#\[allow' mcp.rs auth.rs hints.rs builtin.rs catalog.rs` → one match, the clippy one), so dead code is not being suppressed in place of a `TODO`.

Platform gating convention: `#[cfg(windows)]` / `#[cfg(unix)]` / `#[cfg(not(unix))]` on items rather than `cfg!` inside bodies — `mcp.rs:70-81`, `mcp.rs:732-733`, `mcp.rs:844-857`, `mcp.rs:994-1002`. The Windows env-passthrough contract is deliberately shared with Doctor through a public constant in `lib.rs:19-30` rather than duplicated (`mcp.rs:73-75`).

#### Logging

| Target / level | Sites | Content |
|---|---|---|
| `tracing::error!` | `mcp.rs:446-449`, `mcp.rs:476-479`, `mcp.rs:698-701` | server killed / marked dead, restart failure with attempt counters |
| `tracing::warn!` | `mcp.rs:393-397`, `mcp.rs:403`, `mcp.rs:838-840`; `auth.rs:278`, `auth.rs:401` | hook timeouts, oversize schema replacement, OAuth refresh failure |
| `tracing::info!` | `mcp.rs:663-667`, `mcp.rs:676-680`, `mcp.rs:849-852`, `mcp.rs:856` | restart start/finish with elapsed ms, every `killpg` with its result |
| `tracing::debug!` | `mcp.rs:795` | cancel-notification failure |
| `eprintln!` | `auth.rs:598` | the authorize URL, printed so a headless operator can copy it |

No structured target strings (`tracing::warn!(target: …)` is unused) and mostly positional formatting; only three call sites use structured fields (`error = %e` at `auth.rs:278`, `auth.rs:401`; the schema warning is positional). Logging goes to stderr because the subscriber is configured that way in `async_main` (`lib.rs:156-159`) — stdout is reserved for the ACP wire.

Observability gap: `hints.rs`, `builtin.rs`, and `catalog.rs` contain **no** logging at all (`grep -n 'tracing::' hints.rs builtin.rs catalog.rs` → zero matches). So a skipped skill, a truncated hint chain, a 20-page catalog cut-off, and a silent 32 KiB `load_skill` truncation are all invisible in the log.

Naming convention for the `killpg` "stage" argument is a short kebab/snake label: `"drop"` (`mcp.rs:119`), `"spawn_dropped"` (`mcp.rs:748`), `"kill_server"` (`mcp.rs:446`), `"call_failed"` (`mcp.rs:465`) — and the reasons passed to `kill_server` are human sentences: `"tool timeout"` (`agent.rs:534`), `"hook timeout (consecutive)"` (`mcp.rs:398`).

#### Naming

- `*_impl` suffix marks the dependency-injected inner function that the public wrapper calls with real environment values: `load_hint_files_impl`, `discover_skills_impl`, `build_hints_section_impl`, `collect_supporting_files_impl` (`hints.rs:40`, `204`, `223`, `166`). This is how `hints.rs` reaches ~90% unit-test coverage without touching `$HOME`.
- `MAX_*` for byte/count ceilings (`mcp.rs:20-27`, `hints.rs:6-7`), `*_BYTES` when the unit is bytes, `*_TIMEOUT` for durations (`auth.rs:35`, `auth.rs:39`).
- `qname` vs `bare` is used consistently for qualified and unqualified tool names (`mcp.rs:154-157` onwards).
- `try_*` marks the fallible non-interactive variant (`try_bearer_no_browser`, `auth.rs:367`).
- Test helper names read as assertions of intent (`make_skill_with_files`, `builtin.rs:262`; `seed_cache`, `tests/databricks_oauth.rs:99`).

#### Doc-comment discipline

Prose-heavy where a decision needed justifying — `ResultBudget` (`mcp.rs:28-32`), `truncate_middle` (`mcp.rs:880-885`), `call_hooks` (`mcp.rs:307-314`), `refresh_now` (`auth.rs:56-73`, `auth.rs:303-316`), `try_bearer_no_browser` (`auth.rs:361-366`), `discovery_failure_fallback` (`catalog.rs:35-47`), `parse_v1_endpoints` (`catalog.rs:126-130`). Several comments explain *why not*, which is the useful kind: "a degraded guard is worse than no guard" (`builtin.rs:176-177`), "prefer including over silently dropping" (`catalog.rs:129-130`), "Hooks are intentionally non-cancellable" (`mcp.rs:342-344`).

Gaps against the `AGENTS.md` rule "New public API must have doc comments": `StaticTokenSource::new` (`auth.rs:79`), `PkceOAuthTokenSource::new` (`auth.rs:144`), `McpRegistry` and four of its methods (`mcp.rs:159`, `172`, `266`, `272`, `286`, `485`), `build_hints_section` (`hints.rs:219`), `SkillEntry` (`hints.rs:14`), `MAX_SKILL_BODY_BYTES` (`hints.rs:7`), `LOAD_SKILL_TOOL` (`builtin.rs:13`). Module-level `//!` docs exist for `auth.rs:1-18`, `builtin.rs:1-5`, and `catalog.rs:1-11`, but not for `mcp.rs` or `hints.rs` — the two files that most need an orientation paragraph.

#### File-size discipline

| File | Lines | Test share |
|---|---|---|
| `mcp.rs` | 1,139 | tests start `mcp.rs:1005` (≈12%) |
| `auth.rs` | 845 | tests start `auth.rs:632` (≈25%) |
| `hints.rs` | 726 | tests start `hints.rs:265` (≈64%) |
| `builtin.rs` | 575 | tests start `builtin.rs:240` (≈58%) |
| `catalog.rs` | 402 | tests start `catalog.rs:302` (≈25%) |

`AGENTS.md` documents a 1,000-line ceiling only for `mobile/` (enforced by `mobile/scripts/check-file-sizes.mjs`), and `desktop/web` have their own guards. No equivalent guard exists for Rust: `grep -rn 'check-file-sizes' justfile` matches exactly one line, inside the mobile recipe (`justfile:617`). `mcp.rs` at 1,139 lines (production part ~1,000) would trip the mobile threshold, and `config.rs` in the same crate is 2,701 lines — so the convention is real but unenforced on this side of the repo.

Function-size outliers within the group: `spawn_all` (`mcp.rs:172-264`, 93 lines with 8 nested validation branches), `call_hooks` (`mcp.rs:315-419`, 105 lines), `tool_result_content` (`mcp.rs:913-990`, 78 lines with three nested closures), `browser_pkce_flow` (`auth.rs:527-630`, 104 lines including an inline axum router), `load_supporting_file` (`builtin.rs:118-230`, 113 lines with a doubly-nested `spawn_blocking` match).

#### Test organisation

In-file `#[cfg(test)] mod tests` in four files; `mcp.rs` names its module `content_tests` (`mcp.rs:1006`) even though it now also holds env-allowlist and Windows-flag tests (`mcp.rs:1010`, `mcp.rs:1017`, `mcp.rs:1118`) — the name no longer describes the contents. Integration tests live in `crates/buzz-agent/tests/` and drive the real binary over stdio with a fake LLM and a real fake-MCP subprocess (`tests/regressions.rs`, `tests/hints_integration.rs`, `tests/databricks_oauth.rs`), matching the "real subprocess, no mocks" strategy the README states.

Naming convention for tests is behaviour-as-sentence (`oversized_text_is_middle_elided`, `mcp.rs:1061`; `cached_token_within_leeway_is_expired`, `auth.rs:671`; `discover_skills_project_wins_over_global`, `hints.rs:576`), except in `auth.rs` where four tests keep a redundant `test_` prefix (`auth.rs:722`, `763`, `812`).

Three test-quality conventions worth flagging:
- `tests/databricks_oauth.rs:84-97` re-implements `cache_path_for` locally instead of calling the production function, so a change to the hash inputs would leave the test green while breaking cache lookup.
- `mcp_init_timeout_kills_child` (`tests/regressions.rs:307`) asserts only the error text and elapsed time; the doc comment claims "the child process must be killed (not lingering)" (`tests/regressions.rs:303-304`) but no assertion checks the process.
- `configure_no_window_is_a_noop_on_non_windows` (`mcp.rs:1118-1125`) documents itself as asserting only "didn't crash", and its Windows sibling (`mcp.rs:1127-1140`) contains no assertion at all — its own comment concedes the real protection is the production path.

#### Test coverage — Conventions

There is no lint or test that enforces any convention in this list. `just ci` runs `fmt` + `clippy` + tests (`AGENTS.md`), which covers formatting and clippy's own rules, but nothing checks doc-comment presence on public items, `AgentError` variant choice, log-target consistency, or Rust file size. The one convention with a guard is the desktop text-size rule (`desktop/scripts/check-px-text.mjs`), which does not apply here.
