<!-- Analyzed: 2026-07-25T01:12:08Z | Scope: full project -->
# Code Conventions

> Status: initialized in Phase 1. Patterns, naming, error handling, and testing
> conventions are populated per-module during Phase 2 and consolidated in Phase 3.

## Summary

House rules are documented in `AGENTS.md` / `CONTRIBUTING.md`: no `unsafe`, no new
`unwrap()`/`expect()` in production paths, doc comments on new public API, prefer new event
kinds over new HTTP endpoints, rem-based text sizing in desktop, no `StatefulWidget` in
Flutter, 1000-line file ceiling enforced by scripts.

Re-verified after the post-analysis `CONTRIBUTING.md` rewrite: the rules still
stand, but they now live only in prose — `#![deny(unsafe_code)]` / "do not add
unsafe blocks" at `CONTRIBUTING.md:222-223` and the `unwrap()`/`expect()` ban at
`:230-231`, with `just ci` required at `:183-187` and `:277`. The machine-readable
**PR checklist** that previously restated all of them as tick-boxes was deleted
by that rewrite, so a contributor now has to read the prose sections; the
checklist is no longer available to copy into a PR body.

Batch 2a compliance against those rules:

| Rule | `buzz-core` | `buzz-sdk` | `buzz-persona` | `buzz-ws-client` |
|---|---|---|---|---|
| No `unsafe` | ✅ `#![deny(unsafe_code)]`, 0 blocks | ✅ same | ✅ 0 blocks (no lint attr) | ✅ `#![deny(unsafe_code)]` |
| No `unwrap()`/`expect()` in production | ⚠️ 2 `expect` + 2 `unreachable!` (1 justified inline) | ✅ 0 | ✅ 0 | ⚠️ **2 `unwrap()` + 2 `unreachable!()`** (`connection.rs:170`, `:229`) |
| Doc comments on public API | ✅ `#![warn(missing_docs)]` | ✅ same | ⚠️ no lint; `lib.rs` has no crate doc | ⚠️ convention only, no lint |
| File-size discipline | ⚠️ 3 files >1,000 lines | ⚠️ `builders.rs` **3,824 lines** | ⚠️ `validate.rs` 1,070 | ✅ largest 314 |

The 1,000-line guard (`check-file-sizes.mjs`) covers only `desktop/`, `web/`, and `mobile/`
— **there is no equivalent guard for Rust crates**, which is why `builders.rs` at 3,824
lines holds 51 of 61 public functions without tripping any gate.

Shared idioms across all four crates: one `thiserror` enum per boundary (no `anyhow` in
libraries), guard-clause validation with early `return Err`, `as_str()` + `Display` +
`FromStr` triads on string-mapped enums, `normalize_*`/`canonical_*` for transformers,
inline `#[cfg(test)] mod tests` (no `tests/` dir except `buzz-persona`), and pinned spec
test vectors as module constants. Test-to-code ratio is high: 213 tests in `buzz-core`,
235 in `buzz-sdk`, 145 in `buzz-persona`, but only **3** in `buzz-ws-client` (all
compile-time constant-floor assertions). No property-based tests anywhere in batch 2a.

### Batch 2b compliance (service crates)

| Rule | Status across `buzz-db`, `buzz-auth`, `buzz-pubsub`, `buzz-search`, `buzz-audit`, `buzz-media`, `buzz-workflow` |
|---|---|
| No `unsafe` | ✅ zero blocks in all seven |
| No `unwrap()`/`expect()` in production | ✅ compliant — occurrences are confined to `#[cfg(test)]` modules |
| Doc comments on public API | ⚠️ `missing_docs` is `warn`, not `deny`, where present (e.g. `crates/buzz-pubsub/src/lib.rs:2`), so coverage can regress silently — and did: an orphaned doc comment at `crates/buzz-pubsub/src/lib.rs:43` advertises a module that does not exist |
| No TODO/FIXME/HACK markers | ✅ six of seven clean; `buzz-workflow` carries TODO WF-08 (`src/executor.rs:663`) |
| File-size discipline | ❌ **`crates/buzz-db/src/lib.rs` is 6,106 lines** — larger than `buzz-sdk/src/builders.rs` (3,824) and the biggest Rust file found so far |

The missing Rust file-size guard is now the clearest cross-cutting gap: two files exceed
3,800 lines with nothing to catch them, while `desktop/`, `web/`, and `mobile/` are held to
1,000.

Additional 2b conventions observed:

- **Redis/SQL keys and channel names are always built through helpers**, never
  string-formatted at the call site, with the prefix as a single shared constant
  (`crates/buzz-pubsub/src/topic.rs:13`). Paired parse/format functions sit adjacent and are
  round-trip tested.
- **Every key-building path takes `&TenantContext` rather than a raw `CommunityId`**, so a
  caller cannot fabricate a tenant. This is the most consistently applied convention in the
  service layer.
- **"Never let one bad message kill a loop"** — every Redis subscriber handles malformed
  input with `warn` + `continue` (`crates/buzz-pubsub/src/subscriber.rs:132-157` and
  mirrors).
- **Contract obligations written into telemetry, not just docs** — e.g. "caller MUST fail
  closed" appears in the `warn!` payloads themselves
  (`crates/buzz-pubsub/src/nip98_replay.rs:55`, `:76`).
- **Known-limitation callouts are inline** with a `⚠️` marker and a stated upgrade path
  (`crates/buzz-pubsub/src/rate_limiter.rs:9-10`), rather than tracked in an issue.

Testing-convention regressions in 2b:

- **`buzz-db`: 121 of 122 async tests are `#[ignore]`d**, so the default gate exercises
  essentially none of the data layer. Combined with runtime `sqlx::query()` (no compile-time
  validation), the layer has neither static nor default dynamic verification.
- **`buzz-pubsub`: 11 of 34 tests are `#[ignore = "requires Redis"]`** — including the two
  most valuable ones (cross-community topic isolation, replay-TTL clamping).
- **`crates/buzz-pubsub/src/rate_limiter.rs` has zero tests** despite being a live security
  control; the relay's tests substitute a stub that never touches Redis
  (`crates/buzz-relay/src/admission.rs:65-90`).
- Fixture duplication is common: `buzz-pubsub` defines the same three-line `ctx()` helper
  six times and duplicates `test_presence_set_and_get` across two files
  (`src/lib.rs:477`, `src/presence.rs:138`).
- Still **no property-based tests anywhere** in 2a or 2b.

### Batch 2c conventions (relay, mesh, conformance)

- **`AppState`-threading is the relay's dominant convention** — handlers take
  `State<Arc<AppState>>` and reach subsystems through it rather than receiving narrow
  dependencies, which is why `crates/buzz-relay/src/state.rs` is the file every feature
  touches.
- **Host-derived tenancy is applied uniformly** across WebSocket, HTTP bridge, git, and
  Blossom paths — `TenantContext` is constructed once per request from the `Host` header
  (`crates/buzz-relay/src/tenant.rs`). The single deviation is `/_mesh/demo/echo`, which reads
  `community_id` from the request body.
- **No `unsafe` anywhere in this batch**, consistent with 2a/2b.
- **Conformance mirrors TLA+ naming character-for-character** — every `TraceAction` variant is
  named after its spec action (`crates/buzz-conformance/src/lib.rs:181-250` against
  `docs/spec/MultiTenantRelay.tla:514`, `:559`, `:606`, `:643`, `:681`, `:703`, `:778`, `:794`),
  with `ImplBug` (`src/lib.rs:256`) the one variant having no counterpart.
- **`Inv_*` names exist only in comments, never as Rust identifiers** — the mapping from
  invariant to enforcing predicate lives in prose
  (`crates/buzz-conformance/src/transitions.rs:53-54`, `:296-297`, `src/lib.rs:238`).
- **`M1`…`M8` mutation IDs are a second comment-only vocabulary with no legend in the repo** —
  used at `crates/buzz-conformance/src/lib.rs:127`, `:190`, `:238`;
  `src/transitions.rs:218-221`; `crates/buzz-relay/src/conformance/mod.rs:18-19`;
  `crates/buzz-relay/src/handlers/ingest.rs:1779`. They are inherited from an external
  mutation-testing plan that is not in the repo.
- **Named-reviewer and thread-hash comments couple source to conversations not in the repo**
  (`crates/buzz-relay/src/conformance/mod.rs:37-38`,
  `crates/buzz-conformance/tests/replay_fixtures.rs:19-20`), and several are now stale.
- **Test names encode the assertion, not the target** — `*_bites_*` for expected failures,
  `*_is_fine` / `*_passes` for expected successes
  (`crates/buzz-conformance/src/checker.rs:210`, `:228`, `:247`, `:290`).
- **Observability code never breaks the request** — the `Tracer` trait returns `()`
  (`crates/buzz-conformance/src/lib.rs:317`), `JsonlTracer` recovers from a poisoned mutex and
  swallows IO errors (`crates/buzz-relay/src/conformance/tracers.rs:59-71`), and the read seam
  logs `warn!` and continues on DB failure (`handlers/req.rs:347-353`, `:663-669`).

### Batch 2d conventions (agent surface: buzz-acp, buzz-agent, buzz-dev-mcp, buzz-cli)

The agent surface is the most *convention-dense* batch and the least *tooling-enforced*. Three of
the four `AGENTS.md` quality rules hold well here by discipline alone; the fourth (the file-size
ceiling) has no Rust gate at all, and the no-`unwrap` rule is reviewed by humans rather than clippy.
The distinctive local convention is that **deferred work is recorded in prose, not markers** — which
is why several stale claims in this batch went unnoticed.

| ID | Finding | Location |
|---|---|---|
| CONV-2d-1 | **The `AGENTS.md` "no `unsafe`" rule holds across the batch.** `#![deny(unsafe_code)]` at `crates/buzz-acp/src/lib.rs:1`, `#![forbid(unsafe_code)]` at `crates/buzz-agent/src/lib.rs:1`; zero `unsafe` matches in `buzz-cli` at all. `buzz-dev-mcp` is the only crate with any: it `forbid`s on non-Windows and uses `deny` plus five `#[allow(unsafe_code)]` on Windows, **each carrying a `SAFETY:` comment** — the correct pattern for a platform-gated exception. | `crates/buzz-acp/src/lib.rs:1`; `crates/buzz-agent/src/lib.rs:1` |
| CONV-2d-2 | **The no-`unwrap()`/`expect()` rule is honoured to four sites across ~34,000 lines, and nothing enforces it.** Violations: `config.rs:510` (`.expect("supported is non-empty")`, unaffected by the sync — the 8-line `prefer_mesh_for_auto` diff lands after it), `agents.rs:299` (guarded by an early return two lines above, so it cannot panic today), `client.rs:506` and `client.rs:1379` — the last two both trigger on malformed *relay* data rather than programmer error, which is the more concerning class. Plus three deliberate panics: `unreachable!()` at `llm.rs:1516` (moved from the pre-sync `:1162` by the mesh-routing insertions earlier in the file; reachable only if `MAX_RETRIES` were 0, an invariant asserted only inside the panic message), `client.rs:679` and `:1018`. `just clippy` runs with `-D warnings` (`justfile:107`), but `clippy::unwrap_used` and `expect_used` are allow-by-default and are enabled nowhere — no `[lints]` table in the root `Cargo.toml`, no `clippy.toml`, no crate-level `#![deny]`. So the rule is convention, reviewed by humans. | `crates/buzz-agent/src/config.rs:510`; `llm.rs:1516`; `crates/buzz-cli/src/client.rs:506`, `:1379`; `justfile:107` |
| CONV-2d-3 | **`#[allow(...)]` is used sparingly and never as a `TODO` substitute in `buzz-agent`** — zero matches in the core, LLM and tools groups; the tools group's single attribute is `clippy::too_many_arguments` on `do_call` (`mcp.rs:555`). `buzz-cli` has nine: seven `too_many_arguments` on functions of 8-16 parameters (`mem.rs:537`, `pr.rs:19`/`:65`/`:151`, `patches.rs:8`/`:113`, `issues.rs:80`) and two `dead_code` on unused public client methods (`client.rs:567`, `:802`). The seven suppressions are the notable ones: `buzz-sdk` already provides exactly the right parameter-object types (`GitPatchMeta`, `GitPullRequestMeta`, `GitStatusMeta`, `GitIssueMeta`, `GitPrUpdateMeta`) and each suppressed function **immediately constructs one** — so taking the meta struct as the parameter would delete the `allow` and the positional-argument hazard in one move. No `TODO` sits next to any of the seven. | `crates/buzz-cli/src/commands/pr.rs:19`; `patches.rs:8`; `client.rs:567`, `:802` |
| CONV-2d-4 | **Zero `TODO`/`FIXME`/`HACK`/`XXX` markers across `buzz-acp`, `buzz-dev-mcp`, `buzz-agent`'s core/LLM/tools groups, and all 21 `buzz-cli` command modules.** Deferred work is recorded in prose comments instead — `drain_stale_responses` marked "not yet wired" with `#[allow(dead_code)]` (`acp.rs:1022`), the ACP-v2 squat note (`lib.rs:255-259`), `workflows runs` shipping knowingly non-functional (`workflows.rs:60-64`). No grep-based audit surfaces any of it. The two real markers that do exist are in `buzz-cli`: `workflows.rs:10` is accurate (the raw `EventBuilder` remains on the `--inputs` branch, `:172-177`), and `users.rs:5` is stale — the work is done at `users.rs:299`, and the file's only `EventBuilder` match is the TODO comment itself. | verified by grep; `crates/buzz-cli/src/commands/users.rs:5`, `workflows.rs:10` |
| CONV-2d-5 | **The 1,000-line ceiling `AGENTS.md` documents has no Rust gate, and 15 files in this batch exceed it.** The three `check-file-sizes.mjs` scripts cover desktop (`justfile:123`), web (`:585`) and mobile (`:617`) only; `just check` has no Rust size step. Worst offenders: `buzz-acp/src/lib.rs` 6,570, `relay.rs` 6,233, `pool.rs` 5,620, `queue.rs` 4,759, `acp.rs` 3,717, `buzz-agent/src/llm.rs` 2,894, `config.rs` 2,701 (both crates), `buzz-cli/src/client.rs` 2,477, `lib.rs` 2,035, `channels.rs` 1,713, `shell.rs` 1,503, `notes.rs` 1,330, `messages.rs` 1,167, `view_image.rs` 1,136, `mem.rs` 1,045. Mitigating factor for several: test modules are 25-59% of the file (`llm.rs` ~58%, `config.rs` ~59%, `client.rs` 42%). `buzz-cli/src/lib.rs` has no such excuse — it is one flat clap declaration with 22 subcommand enums plus the dispatch match and no submodule split. **`AGENTS.md` states the ceiling inside the Mobile App § Rules section while also saying "mirroring desktop/web"**, so whether it was ever meant to bind Rust is genuinely ambiguous; either the doc should scope it explicitly or a Rust gate should exist. | `AGENTS.md` Mobile App § Rules; `justfile:123`, `:585`, `:617` |
| CONV-2d-6 | **Doc-comment discipline is inverted relative to the `AGENTS.md` rule.** Private helpers are documented while public API is not. In `buzz-agent`: `Config::from_env` (`config.rs:742`) — the crate's primary configuration entry point — has no doc comment, 20 of `Config`'s 27 fields have none (the field count and the undocumented tally both moved by one when `16d4ec33` added `prefer_mesh_for_auto` at `config.rs:734`, itself documented), and `Llm`'s three public methods (`llm.rs:108`, `:123`, `:230`) have none, while the *private* `openai_request`, `post_openai`, `try_upgrade` and `build_token_source` are all documented (`llm.rs:340-343`, `:594-598`, `:655-656`, `:1519-1528`) — and the same commit extended the inversion rather than correcting it, documenting three more private items (`resolve_openai_model` `:406-409`, `openai_request_for_model` `:532-534`, `PostError` `:1337-1342`) while leaving the public methods bare. 18 of the 20 public items in `types.rs` are undocumented (file unchanged by the sync), as is `pub fn run()` (`lib.rs:110`). In `buzz-cli`, 11 of 22 `*Cmd` enums plus four value enums carry no `///`, and `agent_management.rs` has a `//!` header but no doc on any of its five public items. `buzz-dev-mcp` has no `//!` module doc and no doc on its only public item (`lib.rs:138`). Eight of eleven `buzz-cli` command modules have no module doc; the three that do (`mem.rs`, `moderation.rs`, `pack.rs`) are also the three whose private helpers are documented most thoroughly. | `crates/buzz-agent/src/config.rs:734`, `:742`; `llm.rs:108-230`, `:340-1528`; `crates/buzz-cli/src/lib.rs:101-1015`; `crates/buzz-dev-mcp/src/lib.rs:138` |
| CONV-2d-7 | **Error-handling style splits cleanly by phase in `buzz-agent` and messily by module in `buzz-cli`.** `buzz-agent`: configuration errors are `Result<_, String>` with a `config: ` prefix applied consistently at all 16 sites and the offending env var named verbatim; runtime errors are the typed `AgentError` with a lowercase namespace prefix per variant (`invalid params:`, `llm:`, `mcp:`) and a second-level prefix added at call sites. `Llm::complete` funnels every provider arm through one `map_err` that prepends the model name (`llm.rs:148-155`) with the rationale stated at `:142-147` — but `Llm::summarize` does not, so summarizer failures lose the model name, and nothing flags the asymmetry. `buzz-cli` has **three** competing styles for the same class of failure: the shared `sdk_err` mapper (`validate.rs:151-156`, 20 uses) sends everything except `InvalidInput` to exit 4, while ten sites hardcode `CliError::Usage` (exit 1) and twelve hardcode `CliError::Other` (exit 4) — so a 100 KiB PR body exits **4** while the same over-size input via `moderation resolve` exits **1**. | `crates/buzz-agent/src/llm.rs:142-155`; `crates/buzz-cli/src/validate.rs:151-156` |
| CONV-2d-8 | **`buzz-cli` has exactly two output statements in its core and a strict stdout/stderr split** — `println!` in `print_create_response` (`client.rs:1402`), `eprintln!` in `print_error` (`error.rs:135`), both single-line JSON, with human prose reaching the terminal only through clap's help renderer. The command modules then break the discipline four ways: the shared `normalize_write_response` path (12 commands), `print_create_response` (1 command), bare `println!("{resp}")` passing the relay body through unnormalized (19 commands), and hand-built `serde_json::json!` objects (`agents.rs`). Within a single file: `repos protect set/remove` normalize (`repos.rs:198`) while `repos create` does not (`:228`). `normalize_events` — the sig-stripping helper — is called by two of 21 command modules. | `crates/buzz-cli/src/client.rs:1402`; `error.rs:135`; `commands/repos.rs:198`, `:228` |
| CONV-2d-9 | **Three deliberate non-JSON stdout exceptions, each documented.** `mem get` writes the raw value with **no trailing newline** (`mem.rs:296`, `:300`) so it round-trips with `mem set <slug> -`, explained at `:295`; `mem ls` without `--json` prints TSV (`:268`) and puts the empty-case notice on **stderr** (`:265`) leaving stdout empty; `pack validate`/`inspect` print human-readable text only. `upload file` is the batch's only pretty-printer (`upload.rs:8-11`). Consequence worth naming: `mem set`/`patch`/`rm` print nothing to stdout on success, so they are the only writes in the CLI whose result cannot be piped to `jq`. | `crates/buzz-cli/src/commands/mem.rs:265-303` |
| CONV-2d-10 | **Logging: `buzz-agent` uses four levels with a clear discipline and no `target:` overrides; `buzz-cli` has no logging framework at all.** `buzz-agent` sets one global subscriber to stderr with ANSI disabled (`lib.rs:155-159`) — `error!` for fatal/protocol-breaking, `warn!` for degraded-but-continuing, `info!` for state transitions, `debug!` for high-frequency noise — and uses structured key-value fields only where a machine reader benefits. `llm.rs`/`config.rs` log **exclusively** at `warn!` (nine sites, all structured), and four of the five `config.rs` warnings name `BUZZ_AGENT_THINKING_EFFORT` back to the operator — a good operator-facing convention tying a runtime warning to the knob that caused it. `buzz-cli` has zero `tracing`/`log` matches and declares no logging crate: no secret can reach a log sink from it, but retries, endpoint fallbacks and cursor pagination are invisible, so a caller sees a 60-second hang with no way to observe three attempts and two sleeps. Three `buzz-agent` files — `hints.rs`, `builtin.rs`, `catalog.rs` — have **zero** `tracing::` calls, so a shadowed skill, a truncated hint chain, a 32 KiB `load_skill` cut and a 20-page catalog cut-off are all silent. | `crates/buzz-agent/src/lib.rs:155-159`; verified by grep across `crates/buzz-cli/src` |
| CONV-2d-11 | **Naming conventions are consistent within crates and inconsistent across them.** `buzz-agent`: body builders `<dialect>_body`, parsers `parse_<dialect>`, family predicates `is_<property>_model`, wire emitters `emit_<status>`, fabricated data prefixed `synthetic_`. Two defects: `type OpenAiParse` (`llm.rs:28`) is used for the *Anthropic* parser on the DBv2 route (`:126`), so the name lies at one call site; and "openai" means "chat" in `openai_body` but "the whole family" in `openai_request`/`post_openai`, which also handles Databricks. `buzz-cli`: command enums `<Group>Cmd`, value enums with explicit kebab-case `#[value(name)]`, predicates as questions, and a deliberate `validate_*` (returns `()`) vs `parse_*` (returns the value) split called out at `validate.rs:15-18`. Its defects: handler verb/noun order mixes both directions inside one file (`cmd_create_repo` vs `cmd_send_patch`, `cmd_open_pr` vs `cmd_pr_status`), and `validate_hex64`'s doc says "lowercase hex" while the body accepts uppercase via `is_ascii_hexdigit` (`validate.rs:28-30`) — the same crate enforces lowercase explicitly for media paths (`client.rs:260-262`). | `crates/buzz-agent/src/llm.rs:28`, `:126`; `crates/buzz-cli/src/validate.rs:15-30` |
| CONV-2d-12 | **A strong local convention: env parsers are pure and take `Option<&str>` so they are testable without mutating process state**, with the impure wrappers separated — `parse_thinking_effort` (`config.rs:621`), `parse_openai_api` (`:1012-1013`), `parse_hook_servers` vs `parse_hook_servers_env` (`:1077-1081`) — and each says so in its doc comment. The counterexample in the same file is `Config::from_env` itself, which reads process env directly and consequently has zero tests (DEBT-68). | `crates/buzz-agent/src/config.rs:621`, `:1077-1081` |
| CONV-2d-13 | **Test organization: inline `#[cfg(test)] mod tests` everywhere, with `buzz-agent` splitting real integration tests into `tests/` and `buzz-cli` having none.** `buzz-agent` uses real subprocesses and real TCP listeners rather than mocks (`tests/fake_llm.rs`, `regressions.rs`, `golden_transcripts.rs`, `hints_integration.rs`, `openai_auto_upgrade.rs`, `databricks_oauth.rs`, plus a `fake-mcp` binary declared in `Cargo.toml` because cargo cannot gate bins on `cfg(test)`). Table-driven tests where the input space is enumerable; section dividers as `// ---- topic ----`; assertion messages that name the invariant and interpolate the offending input; test names encoding expected behaviour rather than the function under test (`anthropic_body_omits_thinking_when_max_output_too_small`); and several tests carrying a stated regression target or a review-finding tag (`F5`/`F6`/`F7` in `agents.rs`, "Max's offset-search case" in `mem.rs`). Two are explicitly written to be mutation-sensitive and say so (`llm.rs:2473-2493`). **`buzz-cli` has no `tests/` directory and no async test in any command module** (zero `tokio::test` matches), so no `dispatch` arm and no relay interaction in the command layer is exercised; six of eleven dev-command files and three of ten messaging files have zero tests. | `crates/buzz-agent/tests/`; verified by grep across `crates/buzz-cli/src/commands` |
| CONV-2d-14 | **A recurring anti-convention: tests that assert against a copy of the production rule.** Six instances, each of which would stay green if the production rule were deleted or changed — `buzz-acp`'s four test-local validation copies (`config.rs:1959`, `:2001`, `:2214`, `:2490`); `buzz-agent`'s `valid_effort_values_for_provider_model` (`config.rs:2559-2650`), whose most complex branch is provably dead; `buzz-cli`'s three `BUZZ_ACP_ALLOWED_CHANNEL_ADD_POLICIES` tautologies (`channels.rs:1296-1307`, admitted in the file's own comment at `:1345-1350`); `multi_file_header_count` (`mem.rs:1037-1046`); the two `print_error` JSON-shape tests that rebuild its object inline (`error.rs:197-221`); and `tests/databricks_oauth.rs:84-97`, which re-derives the token-cache path. Two related patterns: `llm_with` (`llm.rs:2689-2698`) constructs `Llm` by struct literal with a **different** client configuration, so every socket-level test runs against a stricter timeout than production; and three `mem.rs` tests (`:885`, `:901`, `:914`) pin third-party `diffy` behaviour rather than first-party code — defensible as a dependency-drift canary and correctly labelled as such (`:880-882`), but worth knowing when reading the coverage number. | `crates/buzz-agent/src/config.rs:2559-2650`; `crates/buzz-cli/src/commands/channels.rs:1296-1307`; `error.rs:197-221` |
| CONV-2d-15 | **`#[cfg(test)]`-gated functions living in production files, with their own test suites.** `parse_retry_in_secs` (`client.rs:172-186`, 6 tests) and `percent_encode` (`validate.rs:75-99`, 5 tests) never run in production — 11 of the group's 95 tests cover code that cannot execute, and `crates/buzz-cli/README.md:216` still advertises percent-encode as live. The second one matters beyond tidiness: it is why `moderation.rs:110-113` interpolates `--status` into a signed query string unescaped (SEC-58). | `crates/buzz-cli/src/client.rs:172-186`; `validate.rs:75-99` |
| CONV-2d-16 | **Comment quality is genuinely high in `buzz-agent`, and that is what makes the stale ones conspicuous.** Comments record *why* rather than *what*, and several preserve negative knowledge that would otherwise be lost: why an empty Anthropic assistant turn is skipped rather than placeholder-filled (`llm.rs:418-422`), why images are batched behind the tool-result run (`:478-486`), why `sum_usage` distinguishes "no usage reported" from "zero" (`:823-828`), why a mixed `*,foo` hook list is not a wildcard (`config.rs:1098-1102`). Eight carry **dated provenance** ("doc-verified, July 2025" — `config.rs:11`, `:101`, `:194`, `:315`, `:578`, `:591`, `:616`, `:619`), which makes staleness auditable rather than invisible. Cross-references are by rustdoc identifier link rather than `file:line` in `buzz-agent`'s tools group (zero `\.rs:[0-9]` matches there), so rustdoc fails if the target disappears — the better pattern. `buzz-acp` does the opposite and carries five stale `file:line` pointers (DEBT-81). | `crates/buzz-agent/src/config.rs:11`, `:1098-1102`; `llm.rs:418-422`, `:478-486` |
| CONV-2d-17 | **Ignored-result and error-suppression idioms are applied consistently and, where deliberate, explained.** `let _ = …` marks recoverable-but-ignored results uniformly (`lib.rs:179`, `:492`; `wire.rs:171`; `lib.rs:181`; `wire.rs:235`). Deliberate suppression is confined to three sites in `buzz-cli`, each with a comment: skipping a corrupt event so it cannot deny-of-service a listing (`mem.rs:120-128`), skipping undecryptable engrams (`:161-170`), and treating a structurally malformed `auth` tag as "bare" rather than an error (`agents.rs:204-206`). The inconsistency is `serde_json::from_str(&resp).unwrap_or_default()` at six sites (`users.rs:47`, `:263`; `workflows.rs:20`, `:45`, `:79`; `users.rs:242-243`), which turns a malformed relay response into an empty array with exit 0 — while `mem.rs:95`, `agents.rs:36` and `repos.rs:174` surface the same failure as an error. Same operation, two opposite policies, no comment explaining either; `users.rs:242-243` is the one with a correctness consequence, since a malformed current profile silently becomes `{}` and a merge can drop fields it was meant to preserve. | `crates/buzz-cli/src/commands/users.rs:47`, `:242-243`; `mem.rs:120-170` |

## Tooling-Enforced Conventions (scan-level)

| Convention | Enforcement | Config |
|---|---|---|
| Rust formatting | `cargo fmt --all -- --check` | `just fmt-check`, CI rust-lint |
| Rust linting | `cargo clippy --workspace --all-targets -- -D warnings` | `just clippy` |
| JS/TS lint + format | Biome | `biome.json`, `just desktop-check` / `web-check` |
| File-size ceiling | `check-file-sizes.mjs` (desktop, web, mobile) | `pnpm check:file-sizes`, `just mobile-check` |
| No arbitrary px/rem text sizes | `check-px-text.mjs` | `pnpm check:px-text` |
| Pubkey truncation policy | `check-pubkey-truncation.mjs` | `pnpm check:pubkey-truncation` |
| Dart format + analyze | `dart format --set-exit-if-changed`, `flutter analyze` | `just mobile-check` |
| Dependency policy | `cargo-deny check` | `deny.toml`, CI security job |
| Dead API-token guard | grep guard in CI | `ci.yml` dead-token-guard |
| Git hooks | lefthook (pre-commit fix + re-stage, pre-push clippy/tests) | `lefthook.yml` |

## Naming Conventions

_Pending Phase 2._

## Error Handling Patterns

_Pending Phase 2 (`thiserror` per-crate error enums, `anyhow` at binaries, `?`
propagation)._

## Testing Conventions

_Pending Phase 2 (unit tests in-crate, `#[ignore]` infra-dependent E2E, cargo-nextest,
Playwright projects `smoke`/`integration`, Flutter widget tests)._

## Architectural Conventions

_Pending Phase 2 (kind-first feature addition, `h`-tag channel scoping, community fencing,
module-level singleton resets on community switch)._

---

# Phase 2 — Module Findings

## Module: buzz-core (`crates/buzz-core`)

### Aspect: Conventions

---

### 1. Module organization

Flat module tree with one nested module. All 15 top-level modules are declared and documented inline in `src/lib.rs:9-38` — each `pub mod` carries a one-line `///` doc *above the declaration* rather than relying solely on the module's own `//!` header:

```
src/
  lib.rs                 crate root: lints, module decls, re-exports, test_helpers
  error.rs               VerificationError only (20 lines — smallest module)
  event.rs               StoredEvent
  verification.rs        verify_event
  filter.rs              NIP-01 matching + reader gate
  kind.rs                kind registry (784 lines, constants + predicates)
  network.rs             is_private_ip
  channel.rs             ChannelVisibility / ChannelType / MemberRole
  presence.rs            PresenceStatus
  relay.rs               normalize_relay_url
  tenant.rs              CommunityId / TenantContext / host normalization
  observer.rs            NIP-44 payload helpers + constants
  agent_turn_metric.rs   NIP-AM payload types (built on observer.rs)
  engram.rs              NIP-AE primitives
  git_perms.rs           git ref protection model
  pairing/
    mod.rs               re-exports + PairingError
    crypto.rs            HKDF derivations
    types.rs             wire messages
    qr.rs                nostrpair:// URI codec
    session.rs           state machine
    NIP-AB.md            spec text kept beside the code
    NIP-AB.spthy         Tamarin formal model kept beside the code
```

Convention: one cohesive concern per file; the only module that composes another is `agent_turn_metric.rs`, which reuses `observer.rs` crypto and error types (`agent_turn_metric.rs:12-15`), and `git_perms.rs`, which imports `channel::MemberRole` (`git_perms.rs:15`).

`pairing/mod.rs` is a facade: submodule declarations (`:22-25`), curated re-exports (`:27-29`), and the shared error enum (`:35-71`). It also documents its own layout in a markdown table inside the module doc comment (`pairing/mod.rs:15-20`).

---

### 2. Naming patterns

| Pattern | Convention as used | Examples |
|---|---|---|
| Event kind constants | `KIND_<DOMAIN>[_<DETAIL>]: u32`, grouped by protocol with section comments | `KIND_STREAM_MESSAGE_EDIT` (`kind.rs:423`), `KIND_NIP29_PUT_USER` (`kind.rs:275`) |
| Exception to that pattern | relay-admin commands use a `RELAY_ADMIN_` prefix instead of `KIND_` | `RELAY_ADMIN_ADD_MEMBER` (`kind.rs:329`) … `RELAY_ADMIN_SET_WORKSPACE_PROFILE` (`kind.rs:335`) |
| Kind sets | plural `*_KINDS: &[u32]` | `ALL_KINDS` (`kind.rs:566`), `P_GATED_KINDS` (`kind.rs:146`), `AUTHOR_ONLY_KINDS` (`kind.rs:120`), `RESULT_GATED_KINDS` (`kind.rs:129`) |
| Range bounds | `<NAME>_KIND_MIN` / `_MAX` | `EPHEMERAL_KIND_MIN/MAX` (`kind.rs:397-399`) |
| Predicates | `is_*(kind: u32) -> bool`, `const fn` wherever possible | `is_ephemeral`, `is_replaceable`, `is_relay_only_kind` (`kind.rs:697-769`) |
| Limits | `MAX_*` / `*_MAX` / `*_MIN` constants rather than inline literals | `MAX_PROTECTION_RULES` (`git_perms.rs:19`), `NIP44_MAX_CONTENT_LEN` (`observer.rs:23`), `SLUG_MAX_LEN` (`engram.rs:31`) |
| Enum → string | `as_str(&self) -> &'static str` plus `Display` delegating to it | `channel.rs:31-56`, `:72-105`, `:123-155`; `presence.rs:22-35` |
| String → enum | `FromStr` with `type Err = String` and a `format!` message | `channel.rs:44-53`, `:88-99`, `:163-179` |
| Constructors that assert provenance | verb-named, not `new` | `TenantContext::resolved` (`tenant.rs:79`), `CommunityId::from_uuid` (`tenant.rs:45`), `StoredEvent::with_received_at` (`event.rs:38`) |
| Crypto derivations | `derive_*` returning fixed-size arrays | `derive_session_id`, `derive_sas`, `derive_transcript_hash` (`pairing/crypto.rs:54-105`) |
| Protocol handlers | `handle_*` consumes an inbound event; `send_*`/`confirm_*` produces an outbound event | `pairing/session.rs:149`, `:227`, `:254`, `:329`, `:388`, `:412` |
| HKDF domain separation strings | `INFO_*` private consts | `pairing/crypto.rs:22-25` |
| Normalizers | `normalize_*` / `canonical_*` | `normalize_host` (`tenant.rs:121`), `normalize_relay_url` (`relay.rs:37`), `normalize_slug` (`engram.rs:123`), `canonical_channel_name` (`channel.rs:15`) |

---

### 3. Error handling

Every error type is an enum; five of seven use `thiserror`, two are hand-rolled.

| Error type | Style | Variants | file:line |
|---|---|---|---|
| `VerificationError` | `thiserror` | 3 (incl. `#[from] nostr::secp256k1::Error`) | `error.rs:2-20` |
| `EngramError` | `thiserror` | 7 | `engram.rs:37-59` |
| `ObserverPayloadError` | `thiserror` | 5 (2 `#[from]`) | `observer.rs:23-46` |
| `PairingError` | `thiserror` | 10 (2 `#[from]`) | `pairing/mod.rs:34-71` |
| `NormalizeRelayUrlError` | `thiserror` + `PartialEq, Eq` (so tests can compare) | 5 | `relay.rs:7-24` |
| `PatternError` | manual `Display` + `impl std::error::Error` | 5 | `git_perms.rs:52-82` |
| `RuleParseError` | manual `Display` + `impl std::error::Error` | 5 (one wraps `PatternError`) | `git_perms.rs:264-301` |
| `Denial` | not an `Error` impl — a value type with `Display` | struct | `git_perms.rs:479-490` |
| `FromStr` errors on channel enums | plain `String` | — | `channel.rs:45`, `:89`, `:164` |

Conventions observed:
- Fallible operations return `Result<_, E>`; no panicking public API.
- No `unwrap()`/`expect()` in production paths **except two documented infallibility cases**, both with a justification comment: `ConversationKey::derive(...).expect("valid keys produce conversation key")` (`engram.rs:137`) and the HMAC key-length `expect` (`engram.rs:147-149`, justified at `:145-147`). Every other `unwrap`/`expect` in the crate is inside `#[cfg(test)]` code or the `test-utils`-gated helpers (`lib.rs:59`, `:68`).
- Error messages embed the offending value with `{x:?}` for round-trippable diagnostics (`channel.rs:52`, `engram.rs:79`, `git_perms.rs:76`).
- Guard-clause style: validation functions early-`return Err(...)` rather than nesting (e.g. `git_perms.rs:84-92`, `qr.rs:105-181`, `session.rs:150-152`).
- Fail-closed defaults are stated in doc comments where a caller must cooperate (`tenant.rs:117-119`, `tenant.rs:151`).

---

### 4. Doc-comment practice

- `#![warn(missing_docs)]` at `lib.rs:2` — every public item carries a doc comment; spot-checking the pub-item inventory found no undocumented public item.
- Module headers use `//!` with a short summary and, where a protocol is involved, an ASCII diagram: derivation flow (`pairing/crypto.rs:12-28`), protocol sequence (`pairing/session.rs:19-40`), permission pipeline (`git_perms.rs:11-16`).
- Private fields and private helpers are also documented (e.g. `PatternSegment` variants `git_perms.rs:41-49`, `PairingSession` fields `session.rs:86-112`).
- Doc comments carry the *rationale*, not just the description — several are effectively design records: the tenant "fence" (`tenant.rs:17-30`), why `normalize_relay_url` is not the AUTH comparator (`relay.rs:28-32`), why `require_patch` blocks all update kinds (`git_perms.rs:254-258`), why the `Unknown` abort reason must not be sent outbound (`pairing/types.rs:95-98`), residual un-zeroizable copies (`session.rs:551-559`).
- Spec citations are inline and section-level: `NIP-AB §Duplicate Event Handling` (`session.rs:530-533`), `NIP-AM §Numeric validity` (`agent_turn_metric.rs:138`), `NIP-AE *Head selection* rule (3)` (`engram.rs:222-224`), `RFC 6598` (`network.rs:32`).
- Cross-references use rustdoc links (`[`StoredEvent`]`, `[`crate::engram`]`) — e.g. `lib.rs:5-6`, `kind.rs:97-98`.
- Two doctests exist (both in pairing): `format_sas` (`pairing/crypto.rs:110-114`) and `encode_qr` (`pairing/qr.rs:66-77`).

---

### 5. Testing patterns

All tests are inline `#[cfg(test)] mod tests` blocks — there is **no `tests/` directory** and no proptest/quickcheck usage anywhere in the crate (searched: no `proptest`, no `quickcheck` in `src/` or `Cargo.toml`).

Static count of `#[test]` functions per file (grep of `#[test]` attributes):

| File | `#[test]` fns | Test module starts at |
|---|---|---|
| `src/network.rs` | 35 (was 29; `c26bf594` added 6 SSRF prefix tests) | `:97` (was `:56`) |
| `src/engram.rs` | 34 | `:607` |
| `src/git_perms.rs` | 34 | `:601` |
| `src/pairing/qr.rs` | 27 | `:245` |
| `src/pairing/session.rs` | 18 | `:756` (a separate `#[cfg(test)] impl` block sits at `:530`) |
| `src/kind.rs` | 14 (was 4; `07d0265c` added 10 persona-visibility tests) | `:823` (was `:747`) |
| `src/pairing/crypto.rs` | 14 | `:131` |
| `src/agent_turn_metric.rs` | 14 | `:194` |
| `src/pairing/types.rs` | 10 | `:98` |
| `src/tenant.rs` | 10 | `:175` |
| `src/filter.rs` | 6 | `:106` |
| `src/presence.rs` | 4 | `:37` |
| `src/relay.rs` | 3 | `:80` |
| `src/observer.rs` | 2 | `:113` |
| `src/verification.rs` | 2 | `:34` |
| `src/channel.rs` | 1 | `:181` |
| `src/event.rs` | 1 | `:53` |
| `src/error.rs`, `src/lib.rs`, `src/pairing/mod.rs` | 0 | — |
| **Total** | **229** (was **213** at `b8510ede`; +6 `network.rs`, +10 `kind.rs`) | |

(Counts are static — obtained by reading the source. The crate was not compiled during this analysis: `cargo` is not on the PATH in this environment without the repo's Hermit activation, so no test-run count is claimed.)

Recurring test conventions:

| Pattern | Example |
|---|---|
| Shared fixture fns at the top of the test module | `make_event()` (`event.rs:57-63`), `stored_with_tag()` (`filter.rs:113-121`), `sample_payload()` (`agent_turn_metric.rs:199-228`), `make_payload_with_turn_cost()` (`:351`), `make_payload_with_cumulative_cost()` (`:374`), `make_payload()` (`qr.rs:250-258`), `keys_from_hex()` (`engram.rs:611-613`) |
| Pinned spec test vectors as consts | `engram.rs:617-627` (SECKEY/PUBKEY/K_C/D_* vectors), `pairing/crypto.rs:136-155` (session secret + ephemeral key bytes) |
| Single test asserting a whole vector suite | `all_test_vectors` (`pairing/crypto.rs:272-323`), `d_tags_match_spec` (`engram.rs:646-654`) |
| Exhaustive-range invariant tests | `replaceable_and_parameterized_are_disjoint` loops `0..=65535` (`kind.rs:852-859`, message at `:856`) |
| Boundary-pair tests (just-inside / just-outside) | CGNAT and benchmarking range tests (`network.rs:299-339`), and — added in `c26bf594` — the NAT64 local-use / Teredo / 6to4 prefix-boundary tests (`network.rs:254-297`), `parameterized_replaceable_range` (`kind.rs:842-850`) |
| Round-trip serde tests per wire type | `pairing/types.rs:102-241`, `presence.rs:41-53`, `agent_turn_metric.rs:230-251` |
| Regression tests labelled as such with the scenario in a comment | `filter.rs:216-236` (explicit h-tag authority), `git_perms.rs:926-941` (guest bypass), `agent_turn_metric.rs:476-507` (validation bypass via lower-level encrypt) |
| Negative/anti-property tests | `reject_*` naming across `qr.rs` (12 `reject_*` tests) and `session.rs` (`reject_out_of_order_operations`, `reject_invalid_session_id`, `reject_event_from_wrong_pubkey`, …) |
| Full-protocol happy-path test driving both peers in one process | `happy_path_full_protocol` (`session.rs:762`) |
| Test-only inherent methods behind `#[cfg(test)]` instead of widening the API | `session.rs:530-544` (`has_processed`, `set_timeout`) |
| Compile-time tests | `const _: () = assert!(...)` block, `kind.rs:783-820` (25 assertions) |
| `assert!` messages explaining the invariant, not just the failure | `filter.rs:234`, `kind.rs:856`, `session.rs:782` |

---

### 6. Other code conventions

- `const fn` is used wherever the computation allows it (`kind.rs:316`, `:697-769`; `tenant.rs:45`, `:50`, `:87`; `engram.rs:114`).
- `#[must_use]` on the two pure string transformers (`tenant.rs:120` for `normalize_host`, `:155` for `relay_url_authority`).
- All kind integers are `u32` with a stated rationale (NIP-01 unsigned integer, u32 covers the range) at `kind.rs:3-5`; conversion to `u16`/`i32` is centralized in `event_kind_u32`/`event_kind_i32` (`kind.rs:772-780`) rather than scattered casts.
- Secret material is handled with a consistent trio: `Zeroizing<String>` in signatures, explicit `.zeroize()` after use, and `Drop` impls (`pairing/session.rs:227`, `:573`, `:731-739`; `pairing/qr.rs:56-60`; `observer.rs:66-109`).
- Hand-rolled serialization is used only where byte-exactness is a spec requirement, with the reason stated (`engram.rs:194-197`, `:261-262`).
- Newtype + private field + accessor is the standard way this crate expresses "provenance matters" (`CommunityId`, `TenantContext`, `RefPattern`, `StoredEvent.verified`).


## Module: buzz-sdk (`crates/buzz-sdk`)

### Aspect: Conventions

---

### 1. Module organization

| File | LOC | Role |
|---|---|---|
| `crates/buzz-sdk/src/lib.rs` | 112 | crate docs, lints, module declarations, shared input types, `SdkError`, `buzz-core` re-exports |
| `crates/buzz-sdk/src/builders.rs` | 3 824 | all 51 event builders + private validation helpers + 162 tests |
| `crates/buzz-sdk/src/mentions.rs` | 820 | pure mention-resolution helpers + 51 tests |
| `crates/buzz-sdk/src/nip_oa.rs` | 595 | NIP-OA auth-tag compute/verify/parse + 22 tests |
| `crates/buzz-sdk/examples/compute_auth_tag.rs` | 29 | example binary wrapping `nip_oa::compute_auth_tag` |

Flat two-level layout — no submodule directories. Types shared by more than one
builder live in `lib.rs`; types used by a single builder family live next to
that family in `builders.rs` (e.g. `GitPatchMeta` at `builders.rs:985`,
`DeleteMessageOptions` at `builders.rs:393`).

`pub use builders::*` (`lib.rs:19`) flattens the builder namespace so consumers
write `buzz_sdk::build_message`, while `mentions` and `nip_oa` stay
namespace-qualified (`lib.rs:15-17`).

Section banners with `// ---` rules mark thematic groups inside `builders.rs`
(`builders.rs:1587-1595` for moderation, `1692-1701` for NIP-IA,
`builders.rs:3286` and `3577` for test groups using `// ──` box-drawing rules).

---

### 2. Naming patterns

| Pattern | Convention | Examples |
|---|---|---|
| Event builders | `build_<operation>` | `build_message`, `build_forum_post`, `build_git_pr_update` (`builders.rs:219`, `278`, `1416`) |
| Variant builders | base name + `_with_<what>` suffix | `build_delete_message_with_options` (`builders.rs:411`), `build_repo_announcement_with_tags` (`builders.rs:952`) |
| Compatibility shims | `_compat` suffix | `build_delete_compat` (kind 5 vs native 9005) (`builders.rs:434`) |
| Private validators | `check_<subject>` returning `Result<(), SdkError>` or `Result<String, SdkError>` | `check_content`, `check_hex_len`, `check_commit_hex`, `check_pubkey_hex`, `check_hex_exact`, `check_repo_id`, `check_custom_emoji_url`, `check_reason`, `check_auth_tag_shape` (`builders.rs:35-170`, `1706-1737`) |
| Public normalizers | `normalize_<subject>` | `normalize_custom_emoji_shortcode` (`builders.rs:127`), `normalize_mention_pubkeys` (`mentions.rs:228`) |
| Tag emitters | `<subject>_tags(…, tags: &mut Vec<Tag>)` | `thread_tags`, `mention_tags`, `imeta_tags`, `identity_archive_tags` (`builders.rs:173`, `188`, `203`, `1739`) |
| Extractors | `extract_<subject>` | `extract_channel_id` (`builders.rs:816`), `extract_at_names`, `extract_nostr_uris` (`mentions.rs:64`, `353`) |
| Input bundles | `<Domain><Thing>Meta` / `Options` | `GitPatchMeta`, `GitIssueMeta`, `GitStatusMeta`, `GitPullRequestMeta`, `GitPrUpdateMeta`, `DeleteMessageOptions` |
| Constants | SCREAMING_SNAKE with unit in the name where relevant | `MENTION_CAP` (`mentions.rs:38`), `MAX_CONTACTS` (`builders.rs:751`), `MAX_REASON_BYTES` (`builders.rs:1704`), `CUSTOM_EMOJI_SET_D_TAG` (`builders.rs:503`) |
| `nip_oa` functions | verb-first, NIP-term nouns | `compute_auth_tag`, `verify_auth_tag`, `parse_auth_tag` |

Local shorthand inside tests: `ev`, `cid`, `eid`, `pk`, `wid`, `rid`
(`builders.rs:1958-1975`).

---

### 3. Builder-pattern style

There is **no fluent/typestate builder struct**. The style is:

1. Free function taking positional parameters, plus one `&Meta` struct when the
   optional-tag count grows large (Git family, `builders.rs:1013`, `1081`,
   `1222`, `1330`, `1416`).
2. Validate everything first, returning early on error.
3. Accumulate `Vec<Tag>` in wire order — `h`/`a`/`d` first, then optional tags.
4. Return `EventBuilder::new(Kind::Custom(n), content).tags(tags)` as the final
   expression. Signing is deliberately excluded (`lib.rs:11-13`).

Optionality is expressed through the type system rather than setters:
`Option<&str>` for optional strings, `Option<MemberRole>` for enums,
`Option<Option<i32>>` for the tri-state TTL (`builders.rs:604-609`), `bool` flags
for presence-only tags (`broadcast`, `truncated`, `root`, `root_revision`).

`#[derive(Default)]` on the five Git meta structs enables `..Default::default()`
partial initialization, which is how tests and the CLI construct them
(`builders.rs:984`, `1072`, `1200`, `1302`, `1396`).

Every kind integer reaches nostr through `Kind::Custom(... as u16)` — 26 sites
use `buzz_core::kind::KIND_*` constants (`builders.rs:6-19`), the rest use bare
literals.

---

### 4. Error handling

- Single crate-level error enum, `SdkError`, derived with `thiserror::Error`
  (`lib.rs:87-113`); six variants, three of which carry a free-form `String`
  message (`InvalidTag`, `InvalidDiffMeta`, `InvalidInput`) and one structured
  (`ContentTooLarge { max, got }`).
- Uniform return type: every fallible public function returns
  `Result<_, SdkError>`. There is no `Box<dyn Error>`, no `anyhow`, and no `From`
  impl chain — third-party errors are stringified at the boundary
  (`builders.rs:31`, `1372`; `nip_oa.rs:126`, `209`, `220`, `242`).
- **No `unwrap()`/`expect()` in non-test code** (verified by scan of `src/*.rs`
  outside `#[cfg(test)]`). The one near-miss is
  `parts.next().unwrap_or_default()` in `GitAppliedPatchRef::parse`
  (`builders.rs:1145`), which cannot panic.
- Guard-clause style: validation blocks sit at the top of each function and
  `return Err(...)` immediately; the tag-building section is unconditional
  afterwards (e.g. `builders.rs:314-340`, `840-919`).
- Error messages embed the offending value or measured size for diagnosis:
  `"{field} must be at least {min_len} hex characters (got {:?})"`
  (`builders.rs:47-49`), `"repo_id exceeds 64 characters (got {})"`
  (`builders.rs:97-100`).
- The example binary is the only place that panics on bad input, via `.expect()`
  on argument parsing (`examples/compute_auth_tag.rs:21-27`).

---

### 5. Doc-comment practice

- `#![warn(missing_docs)]` (`lib.rs:2`) — every public item is documented,
  including struct fields (e.g. `DiffMeta`'s ten fields each carry a `///`
  line, `lib.rs:37-56`) and enum variants (`lib.rs:62-65`).
- Crate doc opens with an ASCII "Mental Model" pipeline diagram
  (`lib.rs:5-13`); `mentions.rs:1-30` and `nip_oa.rs:1-19` do the same for their
  modules (pipeline diagram and tag-format/preimage blocks respectively).
- Builder docs state the kind number in prose ("Build a stream message (kind 9)",
  `builders.rs:211`) and enumerate parameters as a bullet list when there are
  more than three (`builders.rs:211-218`, `1465-1469`).
- Rationale comments cite the governing NIP and, frequently, the exact
  cross-repo file being mirrored: `desktop/src-tauri/src/events.rs:624-743`
  (`builders.rs:1700`), `desktop/src-tauri/src/events.rs:635-647`
  (`builders.rs:1702-1704`), `moderation_commands.rs` (`builders.rs:1593-1594`),
  `NIP-IA.md §Vector 1` (`builders.rs:3625`).
- Deviations from a NIP are called out inline rather than hidden — e.g. "The `h`
  tag is non-standard for NIP-09 but is required so channel-scoped subscriptions
  observe the delete." (`builders.rs:432-433`).
- `# Errors` sections appear in `nip_oa` public functions
  (`nip_oa.rs:140-144`, `173-177`, `245-250`).

---

### 6. Testing patterns

| Aspect | Convention | Evidence |
|---|---|---|
| Location | one `#[cfg(test)] mod tests` per source file; no `tests/` directory in the crate | `builders.rs:1825`, `mentions.rs:389`, `nip_oa.rs:301` |
| Volume | 235 `#[test]` functions total: 162 in `builders.rs`, 51 in `mentions.rs`, 22 in `nip_oa.rs` | counted per file |
| Framework | stock `#[test]` only — no proptest, no rstest, no async tests | no `proptest`/`quickcheck` reference anywhere in the crate |
| Test helpers | small fixture fns at the top of the module: `keys()`, `sign()`, `event_id()`, `uuid()`, `tag_values()`, `has_tag()` | `builders.rs:1830-1873` |
| Domain fixtures | per-family fixture fns: `good_diff_meta()` (`builders.rs:2054`), `pr_repo()` (`builders.rs:3396`), `full_clone_tag()` (`builders.rs:3403`), `profile()` (`mentions.rs:545`) |
| Naming | `<subject>_<scenario>`: `*_happy_path`, `*_minimal`, `*_all_fields`, `*_rejects_*`, `*_too_large` | `builders.rs:1874`, `2821`, `2145`, `2872` |
| Assertion style | sign the builder, then assert on `ev.kind.as_u16()`, `ev.content`, and tag presence via `has_tag`/`tag_values`; positional tag layout asserted where wire order matters | `builders.rs:3707-3735`, `3744-3771` |
| Error assertions | `assert!(matches!(result, Err(SdkError::Variant { .. })))` | `builders.rs:2080-2087`, `2177-2185` |
| Boundary pairs | max-allowed and max+1 tested together | `builders.rs:2009-2020` (64 KiB), `builders.rs:2260-2273` (64-char emoji), `builders.rs:3773-3788` (64-byte reason) |
| Regression comments | tests carry a comment naming the bug or cross-component contract they pin | `builders.rs:3101-3105` (whitespace-only patch), `builders.rs:3613-3615` (relay `extract_p_tag_bytes`), `builders.rs:3688-3690` (relay `extract_report_tag`) |
| Spec vectors | NIP-OA spec pubkeys/conditions/signature and NIP-IA target/owner hex are hoisted into module `const`s and asserted against | `nip_oa.rs:303-310`, `builders.rs:3701-3705` |
| Unicode/panic-safety tests | explicit non-panic tests for multi-byte input | `mentions.rs:520-531`, `mentions.rs:787-800` |
| Cross-client pinning | test comments state that the assertions mirror `desktop/src-tauri/src/events.rs`'s own tests so both clients stay on one wire form | `builders.rs:3696-3699` |


## Module: buzz-persona (`crates/buzz-persona`)

### Aspect: Conventions

### Module organization

`crates/buzz-persona/src/lib.rs:1-6` declares six flat public modules with no re-exports
and no crate-level `//!` doc. The layout mirrors a linear pipeline, one module per stage:

| Module | Role | LOC |
|---|---|---|
| `persona` | `.persona.md` parse (leaf — no intra-crate deps) | 645 |
| `manifest` | `plugin.json` parse (imports `persona::RespondTo` at `crates/buzz-persona/src/manifest.rs:20`) | 365 |
| `merge` | precedence resolution over `serde_json::Value` (leaf — no intra-crate deps) | 464 |
| `pack` | directory loader; composes `manifest` + `persona` + `merge` (`crates/buzz-persona/src/pack.rs:22-24`) | 734 |
| `resolve` | final projection; composes `merge` + `pack` + `persona` (`crates/buzz-persona/src/resolve.rs:18-21`) | 892 |
| `validate` | diagnostics layer over `pack` (`crates/buzz-persona/src/validate.rs:15`) | 1070 |

Dependency direction is strictly one-way (`persona`/`merge` → `manifest` → `pack` →
`resolve`/`validate`); no cycles.

Each stage owns a distinct type family, and types are **re-declared rather than reused**
across stages: `RespondTo` (serde-facing, `crates/buzz-persona/src/persona.rs:53`) →
`TriggersData` (merged, `crates/buzz-persona/src/merge.rs:11`) → `ResolvedTriggers`
(output, `crates/buzz-persona/src/resolve.rs:86`); likewise `Hooks` → `HooksData` →
`ResolvedHooks` and `PersonaConfig` → `LoadedPersona` → `ResolvedPersona`.

### Naming patterns

| Pattern | Examples |
|---|---|
| `parse_*` for string/file → typed | `parse_persona_md`, `parse_persona_file` (`crates/buzz-persona/src/persona.rs:208`, `:262`), `parse_manifest`, `parse_manifest_file` (`crates/buzz-persona/src/manifest.rs:152`, `:190`), `parse_triggers` (`crates/buzz-persona/src/merge.rs:177`), `parse_mcp_server_config` (`crates/buzz-persona/src/resolve.rs:311`) |
| `load_*` for directory → aggregate | `load_pack` (`crates/buzz-persona/src/pack.rs:125`) |
| `resolve_*` for merge/projection | `resolve_persona_config` (`crates/buzz-persona/src/merge.rs:85`), `resolve_pack`, `resolve_loaded_pack`, `resolve_persona_by_name`, `resolve_one_persona`, `resolve_triggers`, `resolve_hooks` (`crates/buzz-persona/src/resolve.rs:108`–`:357`), `resolve_skills` (`crates/buzz-persona/src/pack.rs:249`) |
| `merge_*` for two-sided combination | `merge_behavioral_config` (`crates/buzz-persona/src/merge.rs:47`), `merge_mcp_servers` (`crates/buzz-persona/src/resolve.rs:277`) |
| `split_*` for string decomposition | `split_frontmatter` (`crates/buzz-persona/src/persona.rs:277`), `split_model` (`:324`) |
| `validate_*` public / `*_check_*` private | `validate_pack` (`crates/buzz-persona/src/validate.rs:143`), `validate_persona_name` (`:167`); `semantic_check_personas` (`:187`), `advisory_check_manifest_keys` (`:302`), `advisory_check_respond_to_types` (`:210`), `advisory_check_skill_names` (`:354`) |
| `Loaded*` / `Resolved*` type prefixes marking pipeline stage | `LoadedPack`, `LoadedPersona`, `PackManifestData`; `ResolvedConfig`, `ResolvedPack`, `ResolvedPersona`, `ResolvedMcpServer`, `ResolvedHooks`, `ResolvedTriggers` |
| `*Data` suffix for plain (non-serde) carriers | `TriggersData`, `HooksData` (`crates/buzz-persona/src/merge.rs:11`, `:18`), `PackManifestData` (`crates/buzz-persona/src/pack.rs:102`) |
| `Raw*` / bare shadow structs for permissive deserialization | `RawManifest` (`crates/buzz-persona/src/manifest.rs:132`), `Frontmatter` (`crates/buzz-persona/src/persona.rs:176`) |
| `MAX_*` / `DEFAULT_*` / `KNOWN_*` constant prefixes | `MAX_FRONTMATTER_BYTES`, `MAX_BODY_BYTES` (`crates/buzz-persona/src/persona.rs:21`, `:24`), `MAX_NAME_LEN` (`crates/buzz-persona/src/validate.rs:168`), `DEFAULT_THREAD_REPLIES`/`DEFAULT_BROADCAST_REPLIES` (`crates/buzz-persona/src/merge.rs:38-39`), `KNOWN_MANIFEST_KEYS`/`KNOWN_BEHAVIORAL_KEYS`/`KNOWN_RESPOND_TO_KEYS` (`crates/buzz-persona/src/validate.rs:99`, `:121`, `:133`) |

Note the one naming collision: private `pack::parse_persona_file`
(`crates/buzz-persona/src/pack.rs:392`) shares a name with public
`persona::parse_persona_file` (`crates/buzz-persona/src/persona.rs:262`) while having a
different signature and return type.

### Error handling

Three `thiserror`-derived enums, one per boundary, with no shared error type and no
`anyhow`:

| Enum | Location | Variant count | `#[from]` conversions |
|---|---|---|---|
| `PersonaError` | `crates/buzz-persona/src/persona.rs:26-48` | 7 | `std::io::Error` (`:29`), `serde_yaml::Error` (`:44`) |
| `ManifestError` | `crates/buzz-persona/src/manifest.rs:22-31` | 3 | `std::io::Error` (`:26`), `serde_json::Error` (`:29`) |
| `PackError` | `crates/buzz-persona/src/pack.rs:25-54` | 8 | none automatic; manual `From<ManifestError>` at `:56-60` |

Conventions observed:

- Error messages are lowercase, no trailing period, with the offending value interpolated:
  `"path traversal rejected: {0}"` (`crates/buzz-persona/src/pack.rs:46`),
  `"missing required field: {0}"` (`crates/buzz-persona/src/persona.rs:46`).
- Constants are interpolated into `#[error]` strings:
  `#[error("frontmatter exceeds {MAX_FRONTMATTER_BYTES} bytes")]`
  (`crates/buzz-persona/src/persona.rs:34`).
- Structured variants carry a `path: PathBuf` plus a `reason: String` for file-scoped
  faults (`PackError::Io`, `FileParse`, `McpConfigParse` —
  `crates/buzz-persona/src/pack.rs:30-53`), and `#[source]` is used to preserve the IO
  cause chain (`:33`).
- Required-field checks are done manually against an `Option`-shaped shadow struct so the
  error is a clean `MissingField` instead of a serde path error — rationale stated at
  `crates/buzz-persona/src/manifest.rs:123-125` and `crates/buzz-persona/src/persona.rs:172-173`.
- Cross-boundary errors are **flattened to strings**, losing the source chain:
  `PackError::ManifestParse(e.to_string())` (`crates/buzz-persona/src/pack.rs:58`) and
  `.map_err(|e| PackError::FileParse { reason: e.to_string(), .. })`
  (`crates/buzz-persona/src/pack.rs:393-396`).
- `validate_pack` never returns `Err` — it converts all failures into
  `ValidationDiagnostic::Error` strings (`crates/buzz-persona/src/validate.rs:145-152`).
- No `unwrap()` / `expect()` / `panic!` in any non-test code path (verified by grep — all
  matches are inside `#[cfg(test)]` modules or `tests/`). No `unsafe` anywhere.
- Silent-drop is a deliberate recurring pattern for malformed sub-values:
  `filter_map(...)` on MCP entries (`crates/buzz-persona/src/pack.rs:415-419`,
  `crates/buzz-persona/src/resolve.rs:294-299`), `?` on missing `command`
  (`crates/buzz-persona/src/resolve.rs:312`), `.ok()?`/`continue` chains in the skill
  advisory check (`crates/buzz-persona/src/validate.rs:392-418`).

### Doc-comment practice

- Every module opens with a `//!` header **except** `pack.rs` and `merge.rs`, which use
  `///` on the first item instead — `crates/buzz-persona/src/pack.rs:1-19` and
  `crates/buzz-persona/src/merge.rs:1-9` are outer doc comments attached to the following
  item rather than module-level docs. `lib.rs` has no doc at all.
- Module docs carry a literal example: a `.persona.md` sample in
  `crates/buzz-persona/src/persona.rs:5-14`, a `plugin.json` sample in
  `crates/buzz-persona/src/manifest.rs:8-15`, a directory tree in
  `crates/buzz-persona/src/pack.rs:6-16`.
- Public functions document semantics, and `load_pack` documents its algorithm as a
  numbered list matching in-body step comments (`crates/buzz-persona/src/pack.rs:117-124`
  vs. the `// 1.` … `// 6.` markers at `:131`, `:151`, `:166`, `:190`, `:222`).
- Non-obvious decisions get a rationale block rather than a bare statement — e.g. the
  three-step defense-in-depth explanation on `safe_resolve`
  (`crates/buzz-persona/src/pack.rs:317-322`), the security note on `resolve_hooks`
  (`crates/buzz-persona/src/resolve.rs:339-346`), and the permissiveness rationale on
  `RawManifest` (`crates/buzz-persona/src/manifest.rs:123-130`).
- Tri-state field semantics are documented on the field itself with a bullet list —
  `PersonaConfig::subscribe` (`crates/buzz-persona/src/persona.rs:129-134`),
  `ResolvedConfig::subscribe` (`crates/buzz-persona/src/merge.rs:29-31`).
- `# Limits` is used as a doc heading on `parse_persona_md`
  (`crates/buzz-persona/src/persona.rs:205-207`).
- No `#[doc(hidden)]`, no `#![deny(missing_docs)]`, no doctests (all fenced blocks are
  ```text` or ```json`, e.g. `crates/buzz-persona/src/persona.rs:5`,
  `crates/buzz-persona/src/manifest.rs:8`).

### Testing patterns

| Convention | Evidence |
|---|---|
| Per-module `#[cfg(test)] mod tests { use super::*; }` | `crates/buzz-persona/src/manifest.rs:195-197`, `merge.rs:200-202`, `pack.rs:447-449`, `persona.rs:331-333`, `resolve.rs:400-404`, `validate.rs:436-438` |
| Test counts | `persona.rs` 29, `resolve.rs` 26, `merge.rs` 22, `validate.rs` 22, `manifest.rs` 14, `pack.rs` 14; `tests/integration.rs` 13, `tests/e2e_env_flow.rs` 5 — **145 total** |
| Descriptive snake_case test names encoding the expectation | `unknown_frontmatter_keys_error`, `closing_delimiter_with_trailing_junk_is_not_valid` (`crates/buzz-persona/src/persona.rs:403`, `:483`), `triggers_shallow_replacement` (`crates/buzz-persona/src/merge.rs:352`) |
| Fixture builders instead of inline setup | `minimal()` (`crates/buzz-persona/src/persona.rs:334`), `minimal_json()` (`crates/buzz-persona/src/manifest.rs:198`), `make_pack()` (`crates/buzz-persona/src/pack.rs:451`), `make_loaded_persona()` (`crates/buzz-persona/src/pack.rs:609`), `stub_persona()` (`crates/buzz-persona/src/resolve.rs:864`), `create_test_pack()` (`crates/buzz-persona/tests/integration.rs:14`) |
| Named fixture constant for the canonical persona | `SIMPLE_PERSONA` (`crates/buzz-persona/src/pack.rs:481-487`) |
| `tempfile::TempDir` / `tempfile::tempdir()` for on-disk pack fixtures | `crates/buzz-persona/src/pack.rs:450`, `crates/buzz-persona/src/validate.rs:462` |
| Error assertions via `matches!` on the variant, with the error echoed in the failure message | `assert!(matches!(&err, PersonaError::MissingField(f) if f == "name"), "got: {err}")` — `crates/buzz-persona/src/persona.rs:433-437` |
| Assertion messages state the *rule*, not the mechanics | `"pack keywords should be lost under shallow replacement"` (`crates/buzz-persona/src/merge.rs:361-364`); `"built-in default for mentions is true"` (`:379`) |
| Regression tests annotated with the bug they lock in | `crates/buzz-persona/src/merge.rs:400-403` — "Critical regression test … This was broken when BehavioralDefaults serialized the field as \"respond_to\" but merge looked for \"triggers\""; `crates/buzz-persona/tests/integration.rs:158-159` |
| Platform-gated tests | `#[cfg(unix)] #[test] fn symlink_escape_rejected` (`crates/buzz-persona/src/pack.rs:589-607`) |
| Integration tests exercise the documented pipeline end-to-end and are described in a module doc | `crates/buzz-persona/tests/integration.rs:1-5`, `crates/buzz-persona/tests/e2e_env_flow.rs:1-9` |
| Table-driven loop for a family of rejections | `for field in ["idle_timeout", "max_turn_duration", ...]` — `crates/buzz-persona/tests/integration.rs:637-650` |
| Local test helper for readable multiline fixtures | `fn indoc(s: &str) -> &str` (`crates/buzz-persona/src/persona.rs:642-644`) — hand-rolled rather than pulling the `indoc` crate |

No property-based testing, no snapshot testing, no mocking framework, no `#[should_panic]`.

### Asset embedding

There is none. No `include_str!` / `include_dir!` / `include_bytes!`, no `build.rs`, no
`[build-dependencies]`, no asset directory (`crates/buzz-persona/Cargo.toml:1-18` and the
full file listing confirm this). Personas are read from caller-supplied directories at
runtime via `std::fs`. The only bundled non-Rust file is
`crates/buzz-persona/PERSONA_PACK_SPEC.md`, which is documentation and not compiled in.

The convention for "packaged" data is therefore **path-relative directory conventions**
rather than embedding: `.plugin/plugin.json`, `instructions.md`, `.mcp.json`, `skills/`
are located by joining onto the canonicalized pack root
(`crates/buzz-persona/src/pack.rs:132`, `:183`, `:209`, `:224`).

### Other crate-level conventions

- Dependencies are pinned inline with explicit versions rather than
  `{ workspace = true }` (`crates/buzz-persona/Cargo.toml:10-14`), and the package
  declares its own `version`, `edition`, `license`, `repository` rather than inheriting
  from the workspace (`crates/buzz-persona/Cargo.toml:1-8`) — the opposite of the pattern
  in e.g. `crates/buzz-acp/Cargo.toml:1-8`. The declared `repository` is
  `https://github.com/block/sprout` (`crates/buzz-persona/Cargo.toml:7`), the pre-rename
  repo name.
- `edition = "2021"` (`crates/buzz-persona/Cargo.toml:4`) with no `rust-version` key.
- Import style: `use std::...` first, blank line, then external crates, then `use crate::...`
  (`crates/buzz-persona/src/pack.rs:20-24`, `crates/buzz-persona/src/resolve.rs:16-21`).
  `persona.rs` and `manifest.rs` follow the same order (`crates/buzz-persona/src/persona.rs:15-19`).
- Fully-qualified inline paths are used in places instead of top-level imports
  (`std::collections::HashSet::new()` at `crates/buzz-persona/src/pack.rs:264`,
  `std::path::Path::new` at `:254`), while the same types are imported at the top of other
  modules — the style is not uniform.
- Nested helper functions are used where a helper is single-use: `normalize_skill_name`
  declared inside `resolve_skills` (`crates/buzz-persona/src/pack.rs:253-260`), and a
  closure `parse_mcp` inside `load_pack` (`crates/buzz-persona/src/pack.rs:192-197`).
- Inline captured-identifier format strings throughout (`format!("...{e}")`,
  `write!(f, "ERROR: {msg}")` — `crates/buzz-persona/src/validate.rs:28`), consistent with
  modern clippy preferences.


## Module: buzz-ws-client (`crates/buzz-ws-client`)

### Aspect: Conventions

---

### 1. Module organization

| Module | Role | file:line |
|---|---|---|
| `lib.rs` | Crate lint gate + module declarations + flat re-exports; contains no logic | `lib.rs:1`–`9` |
| `connection.rs` | Transport, handshake, timeouts, read loops, one-shot helper, unit tests | `connection.rs:1`–`314` |
| `message.rs` | Wire types + parser + AUTH-event builder (pure functions, no I/O) | `message.rs:1`–`190` |
| `error.rs` | Single error enum + one manual `From` impl | `error.rs:1`–`51` |

Conventions observed: one concern per file; pure logic (`message.rs`) separated from
I/O (`connection.rs`); errors centralized in one enum rather than per-module error
types; re-exports flattened at the root so callers can use
`buzz_ws_client::{NostrWsConnection, RelayMessage, …}` (`lib.rs:7`–`9`) while module
paths remain available.

---

### 2. Naming

| Pattern | Examples | file:line |
|---|---|---|
| Types in `UpperCamelCase`, protocol-flavoured names | `NostrWsConnection`, `RelayMessage`, `OkResponse`, `WsClientError` | `connection.rs:26`, `message.rs:8`, `message.rs:51`, `error.rs:5` |
| Enum variants mirror wire message names (`OK`→`Ok`, `EOSE`→`Eose`) | `Event`, `Ok`, `Eose`, `Closed`, `Notice`, `Auth` | `message.rs:10`–`39` |
| Constants `SCREAMING_SNAKE_CASE` with unit suffix `_SECS` | `AUTH_CHALLENGE_TIMEOUT_SECS`, `AUTH_OK_TIMEOUT_SECS`, `PUBLISH_OK_TIMEOUT_SECS` | `connection.rs:17`, `:20`, `:23` |
| Verb-first fn names; `wait_for_*` for blocking waits, `recv_*`/`send_*` for I/O | `connect`, `connect_authenticated`, `authenticate`, `send_event`, `send_raw`, `next_event`, `disconnect`, `recv_one`, `wait_for_auth_challenge`, `wait_for_ok` | `connection.rs:37`–`269` |
| Parameter naming avoids shadowing the imported `timeout` fn | `timeout_dur` (not `timeout`) | `connection.rs:106`, `:159`, `:220` |
| Builder/parse free functions named after the artifact | `build_auth_event`, `parse_relay_message` | `message.rs:174`, `:62` |
| Test names assert an invariant | `auth_challenge_timeout_meets_floor` | `connection.rs:301` |

---

### 3. Error handling

- One crate-wide `thiserror` enum, `WsClientError`, `#[derive(Debug, Error)]`
  (`error.rs:4`–`5`), with a `#[error("…")]` display string on every variant
  (`error.rs:7`, `:11`, `:15`, `:19`, `:23`, `:27`, `:31`, `:35`, `:39`, `:43`).
- Automatic conversions where the source type is owned by a dependency:
  `#[from] tokio_tungstenite::tungstenite::Error` (`error.rs:8`) and
  `#[from] serde_json::Error` (`error.rs:12`). A hand-written
  `From<nostr::event::builder::Error>` stringifies instead of wrapping
  (`error.rs:47`–`51`), and the same stringify is duplicated inline at
  `message.rs:189`.
- Foreign errors that are not `#[from]` are stringified into `Url(String)` /
  `EventBuilder(String)` via `map_err(|e| … e.to_string())`
  (`connection.rs:51`, `message.rs:180`, `message.rs:189`) — source chains are not
  preserved for those.
- `?` propagation throughout; every fallible fn returns
  `Result<_, WsClientError>` (`connection.rs:41`, `:48`, `:74`, `:96`, `:107`, `:115`,
  `:121`, `:283`; `message.rs:62`, `:179`).
- Guard-clause style: early `return Err(...)` for rule violations
  (`connection.rs:88`, `:184`, `:199`, `:241`) and early `return Ok(...)` for cache
  hits (`connection.rs:109`, `:130`, `:162`, `:171`, `:203`, `:230`, `:254`).
- Repo rule "no new `unwrap()`/`expect()` in production paths"
  (`AGENTS.md`, Quality Gates) is **not fully honoured**: two `unwrap()` calls exist
  on `VecDeque::remove` immediately after a successful `position` lookup
  (`connection.rs:170`, `:229`), paired with `_ => unreachable!()` arms
  (`connection.rs:172`, `:231`). No `expect()` anywhere.
- `#[allow(clippy::result_large_err)]` is applied to `parse_relay_message`
  (`message.rs:61`) — the only lint suppression in the crate.

---

### 4. Async patterns

- All I/O methods are `async fn` on an owned `WebSocketStream`; there is no
  `Arc`/`Mutex`, no spawned task, no channel — the connection is single-owner and
  `&mut self`-driven (`connection.rs:70`, `:96`, `:104`, `:121`).
- Timeouts are applied with `tokio::time::timeout` around a single `.next()` await
  (`connection.rs:134`, `:187`, `:244`) or around a whole async block
  (`connection.rs:284`).
- Absolute deadlines with per-iteration recomputation for multi-frame waits
  (`connection.rs:176`–`185`, `:222`, `:236`–`242`) — avoids the timeout being reset
  by unrelated frames.
- Message buffering via `VecDeque` (`connection.rs:28`) with `push_back` for
  deferral and `pop_front` for delivery (`connection.rs:108`, `:129`, `:205`, `:257`).
- `disconnect(mut self)` consumes the connection so use-after-close is a compile
  error (`connection.rs:115`).
- Sink/stream traits are brought in explicitly rather than via a prelude:
  `use futures_util::{SinkExt, StreamExt};` (`connection.rs:4`).
- The three read loops share an identical `match raw { Text | Ping | Close | _ }`
  shape (`connection.rs:140`–`153`, `:193`–`:213`, `:250`–`:267`) — a deliberate
  copy of the same frame-dispatch skeleton in each waiter.

---

### 5. Documentation conventions

- Every public item has a `///` doc comment: constants (`connection.rs:16`, `:19`,
  `:22`), the struct (`:25`), each public method (`:34`–`36`, `:47`, `:67`–`69`,
  `:95`, `:103`, `:114`, `:120`), the free functions (`:272`–`276`; `message.rs:60`,
  `:169`–`173`), both public message types (`message.rs:6`, `:49`) and every enum
  variant *and* variant field (`message.rs:9`–`46`, now including the `Count`
  variant and its two fields at `:40`–`45`), and every error variant
  (`error.rs:3`, `:6`, `:10`, `:14`, `:18`, `:22`, `:26`, `:30`, `:34`, `:38`, `:42`).
  This satisfies the repo rule that new public API carries doc comments.
- Intra-doc links are used for cross-references: `[`RelayMessage`]`
  (`message.rs:60`), `[`crate::NostrWsConnection`]` (`error.rs:3`).
- Multi-paragraph docs explain the *why* on the two most subtle items: the
  `auth_tag` parameter (`message.rs:169`–`173`) and the bounded one-shot helper
  (`connection.rs:272`–`276`).
- No `//!` module-level docs on any file, and no `#![warn(missing_docs)]` — the
  doc-comment discipline is convention, not enforced here.

---

### 6. Lints and safety posture

- `#![deny(unsafe_code)]` at the crate root (`lib.rs:1`) — matches the repo's
  "no `unsafe` code" rule; no `unsafe` block exists in the crate (verified).
- No `#![forbid(...)]`, no `clippy.toml`, no crate-level `#![allow]`.
- Dependencies are all `{ workspace = true }` (`Cargo.toml:10`–`17`), so versions and
  features are centralized in the root manifest rather than pinned per crate;
  package metadata is likewise inherited (`Cargo.toml:3`–`7`).

---

### 7. Testing conventions

- Inline `#[cfg(test)] mod tests` with `use super::*;` at the bottom of the file
  under test (`connection.rs:296`–`298`) — no `tests/` directory, no dev-dependencies
  (`Cargo.toml` has no `[dev-dependencies]` section).
- Tests are compile-time invariant assertions rather than behavioural tests:
  `const { assert!(CONST >= floor) }` inside `#[test] fn` bodies
  (`connection.rs:302`, `:307`, `:312`) — encoding timeout *floors* so a future edit
  that lowers a timeout fails to build.
- No `#[tokio::test]`, no mock relay, no parser unit tests, no property tests.


## Module: buzz-db (`crates/buzz-db`)

### Aspect: Conventions

#### 1. Crate-level lints and module layout

- `#![deny(unsafe_code)]` and `#![warn(missing_docs)]` at
  `crates/buzz-db/src/lib.rs:1-2`; every public item carries a doc comment.
- One module per persistence concern, each declared with a doc comment in
  `lib.rs` (`crates/buzz-db/src/lib.rs:12-51`): `admin_moderation`, `api_token`,
  `archived_identities`, `channel`, `dm`, `error`, `event`, `feed`, `git_repo`,
  `migration`, `moderation`, `partition`, `product_feedback`, `push`, `reaction`,
  `relay_members`, `replica_fence`, `thread`, `usage`, `user`, `workflow`
  (21 modules + `lib.rs`).
- Two-layer API: module-level free functions take `pool: &PgPool` (or
  `tx: &mut Transaction<'_, Postgres>`) plus a `CommunityId`; `Db` methods are
  thin delegating wrappers. Inline SQL on `Db` is the exception, not the rule.
- Section banners inside larger modules: `// -- Public structs ---`,
  `// -- Write operations ---`, `// -- Read operations ---`, `// -- Row mappers ---`
  (e.g. `crates/buzz-db/src/reaction.rs:10`, `:64`, `:276`;
  `crates/buzz-db/src/thread.rs:16`, `:108`, `:339`).
- Re-export policy: enums shared with the SDK live in `buzz-core` and are
  re-exported, not redefined — `pub use buzz_core::channel::{ChannelType, ChannelVisibility, MemberRole}`
  with the rationale in the comment (`crates/buzz-db/src/channel.rs:13-17`).

#### 2. Naming

| Pattern | Convention | Examples |
|---------|-----------|----------|
| Reads returning a single row | `get_*` / `find_*` (`find_*` when absence is normal) | `get_channel`, `get_event_by_id`, `find_dm_by_participants`, `find_by_owner_and_name` |
| Reads returning many rows | `list_*` / `query_*` (`query_*` when filters are involved) | `list_channels`, `list_relay_members`, `query_events`, `query_mentions` |
| Batched variants | `*_bulk` suffix | `get_members_bulk`, `get_users_bulk`, `get_member_counts_bulk`, `get_last_message_at_bulk`, `get_reactions_bulk` |
| Predicates | `is_*` / `has_*` | `is_member`, `is_relay_member`, `is_archived`, `is_agent_owner`, `has_allowlist_entries`, `has_join_policy_acceptance`, `has_read_pool` |
| Idempotent create | `ensure_*` | `ensure_user`, `ensure_configured_community`, `ensure_future_partitions` |
| Upsert with ordering semantics | `replace_*` / `upsert_*` / `accept_*` | `replace_addressable_event`, `replace_parameterized_event`, `upsert_workflow`, `accept_lease_event` |
| Queue/lease lifecycle | `claim_* / complete_* / retry_* / fail_* / release_* / reap_* / prune_*` | `claim_due_wakes`, `complete_match_batch`, `retry_wake`, `fail_wake`, `release_due_reminder`, `reap_exhausted_matches`, `prune_wake_outbox` |
| Row → struct mappers | private `row_to_*` | `row_to_stored_event`, `row_to_channel_record`, `row_to_member_record`, `row_to_report`, `row_to_ban`, `row_to_action`, `row_to_workflow_record`, `row_to_run_record`, `row_to_approval_record`, `row_to_claimed_wake`, `row_to_archived_identity`, `row_to_feedback` |
| Transaction-aware twins | `*_tx` suffix | `get_active_role_tx`, `get_channel_tx`, `add_reaction_tx`, `insert_event_with_thread_metadata_tx` |
| Insert parameter bags | `New*` / `*Params` structs instead of long argument lists | `NewReport`, `NewAction`, `NewProductFeedback`, `NewWake`, `ThreadMetadataParams`, `CreateApprovalParams`, `ChannelUpdate` |
| Outcome enums instead of booleans when >2 states | `*Outcome` / `*Result` | `ReserveOutcome`, `AcceptLeaseOutcome`, `ReplaceLeaseOutcome`, `EnqueueWakeOutcome`, `RevalidateWakeOutcome`, `ReactionEventInsertOutcome`, `RemoveResult`, `TransferResult`, `CreateCommunityWithOwnerResult` |
| Advisory-lock key namespaces | `'<domain>:'` prefixes, documented as mutually distinct | `'buzz_push_gate:'` (`crates/buzz-db/src/push.rs:21`), `'buzz_channel_ttl:'` (`migrations/0024_…:20`), `'buzz_audit:'` (referenced at `migrations/0023_…:20`) |
| SQL identifiers | `snake_case`; indexes `idx_<table>_<cols>` in `0001`–`0007`/`0017`, bare `<table>_<purpose>` for push tables in `0012`/`0015`/`0018` | `idx_events_community_channel_created` vs `push_wake_outbox_due` |

Boolean returns are consistently "did this call change state": `rows_affected() > 0`
(e.g. `crates/buzz-db/src/event.rs:742`, `crates/buzz-db/src/reaction.rs:110`,
`crates/buzz-db/src/git_repo.rs:180`) or `== 1` where exactly one row is the
contract (`crates/buzz-db/src/push.rs:1148`, `crates/buzz-db/src/event.rs:1471`).

#### 3. Error handling

Single error enum `DbError` (`crates/buzz-db/src/error.rs:7-52`) with
`thiserror`, plus `pub type Result<T> = std::result::Result<T, DbError>` at `:51`.

| Variant | Source | Meaning |
|---------|--------|---------|
| `Sqlx(#[from] sqlx::Error)` | `:11` | driver-level failure |
| `Migrate(#[from] sqlx::migrate::MigrateError)` | `:15` | migration failure |
| `AuthEventRejected` | `:21` | kind 22242 must not be stored |
| `EphemeralEventRejected(u16)` | `:25` | kinds 20000–29999 must not be stored |
| `ChannelNotFound(uuid::Uuid)` | `:29` | |
| `MemberNotFound(uuid::Uuid)` | `:33` | |
| `NotFound(String)` | `:37` | generic |
| `AccessDenied(String)` | `:41` | permission/state refusal |
| `Serde(#[from] serde_json::Error)` | `:45` | JSON round-trip |
| `InvalidData(String)` | `:49` | malformed input or malformed stored value |
| `InvalidTimestamp(i64)` | `:53` | timestamp could not be interpreted |

Separate probe-only error type: `replica_fence::ProbeError`
(`crates/buzz-db/src/replica_fence.rs:363-380`) with `Writer`,
`MaskedActivity { masked }`, `ReplicaLsnUnavailable`; all three are treated
identically (fail closed) by the probe loop at `:492-502`.

Conventions:
- **No `unwrap()`/`expect()` on fallible DB results in production paths.** Row
  decoding uses `row.try_get(...)?`. The only non-test `expect`/`unwrap` calls are
  infallible-slice conversions in lock-key derivation
  (`crates/buzz-db/src/push.rs:224`, `:230`), a `expect("one outcome per request")`
  on a locally guaranteed vector length (`crates/buzz-db/src/push.rs:601`), and
  `expect("length checked")` after an explicit length check
  (`crates/buzz-db/src/thread.rs:328`).
- Postgres error codes are matched explicitly where behaviour depends on them:
  `42P17` overlap in the partition manager (`crates/buzz-db/src/partition.rs:139-141`),
  `23505` + constraint name and the generic `23xxx` family in push
  (`crates/buzz-db/src/push.rs:392-410`), `23514` check-violation in the fence
  verifier (`crates/buzz-db/src/replica_fence.rs:206`).
- Best-effort side indexes never fail the caller: mention inserts are logged and
  swallowed (`crates/buzz-db/src/lib.rs:1086-1089`, `:1392-1395`, `:1424-1427`,
  `:3428-3431`, `:3610-3613`, `:3812-3815`).
- Enum/status strings are parsed with `FromStr` returning
  `DbError::InvalidData(format!("unknown … : {other}"))` — never a silent default
  (`crates/buzz-db/src/workflow.rs:61-71`, `:103-116`, `:148-160`).
- `try_get(...).unwrap_or(None)` is used deliberately for columns that may be
  absent from a given projection, and is documented as such
  (`crates/buzz-db/src/channel.rs:990-999`).

#### 4. Query-construction style

| Style | When used | Examples |
|-------|-----------|----------|
| Static SQL string + positional `.bind()` | the default | most of the crate |
| `sqlx::QueryBuilder` with `push_bind` / `separated(", ")` / `push_values` | variable-length `IN (…)` lists and optional predicates | `crates/buzz-db/src/event.rs:360-465`, `:591-698`; `crates/buzz-db/src/feed.rs:91-119`; `crates/buzz-db/src/lib.rs:146-163`; `crates/buzz-db/src/channel.rs:1337-1349`; `crates/buzz-db/src/event.rs:877-889`, `:957-966` |
| `format!` + `sqlx::AssertSqlSafe`, all values still bound | dynamic SET/ORDER/placeholder shapes that `QueryBuilder` can't express, and DDL | 15 sites: `crates/buzz-db/src/channel.rs:870`, `:957`, `:1107`; `crates/buzz-db/src/thread.rs:430`, `:631`; `crates/buzz-db/src/user.rs:148`; `crates/buzz-db/src/usage.rs:281`, `:323`; `crates/buzz-db/src/partition.rs:130`; `crates/buzz-db/src/lib.rs:5235`, `:5256`, `:6009`, `:6014`; `crates/buzz-db/src/replica_fence.rs:613`, `:638` (last five are test-only) |
| `sqlx::query_as::<_, (…tuple…)>` | small fixed projections | `crates/buzz-db/src/user.rs:61`, `crates/buzz-db/src/usage.rs:43` and siblings, `crates/buzz-db/src/lib.rs:3345` |
| `sqlx::query_scalar::<_, T>` | single-value reads | `crates/buzz-db/src/lib.rs:519`, `:687`; `crates/buzz-db/src/push.rs:299`; `crates/buzz-db/src/relay_members.rs:455` |
| `ANY($n)` array binds | fixed-arity list predicates | `crates/buzz-db/src/channel.rs:565`, `:625`; `crates/buzz-db/src/push.rs:912`, `:980` |
| Nullable-filter idiom `($n::type IS NULL OR col = $n)` | optional filters without dynamic SQL | `crates/buzz-db/src/admin_moderation.rs:106-112`; `crates/buzz-db/src/moderation.rs:222` |
| Two static query variants instead of one dynamic string | when only a single optional predicate exists | `crates/buzz-db/src/channel.rs:669-708`; `crates/buzz-db/src/dm.rs:239-306` |

Ordering/pagination conventions: `ORDER BY created_at DESC, id ASC` for event
reads (`crates/buzz-db/src/event.rs:535-537`, rationale `:529-534`); composite keyset cursors rather
than OFFSET for channel windows and thread pages
(`crates/buzz-db/src/thread.rs:380-386`, `:595-602`); a `LIMIT n+1` probe rather
than a second COUNT for `has_more` (`crates/buzz-db/src/thread.rs:640-643`).
Every list query has a bound: an explicit `LIMIT` literal (1000), a clamped
parameter, or a constant (`FEED_MAX_LIMIT`, `LIST_MAX_LIMIT`, `MAX_PAGE_SIZE`).

#### 5. Transactions and locking

- `pool.begin()` … `tx.commit()` / `tx.rollback()`: **33** `begin()` sites and
  **30** `commit()` sites in `crates/buzz-db/src/**` — the difference is
  early-return paths that `rollback()` deliberately
  (e.g. `crates/buzz-db/src/lib.rs:3366`, `crates/buzz-db/src/event.rs:1263`,
  `crates/buzz-db/src/relay_members.rs:463`).
- Transactions are used wherever two writes must agree: channel create +
  owner bootstrap, event + thread metadata + counters, reaction + kind:7 event,
  membership + policy acceptance, community + owner, lease + source event.
- Advisory locks are acquired **first** inside a transaction, in one documented
  global order per subsystem (`crates/buzz-db/src/push.rs:239-243`,
  `migrations/0024_event_ttl_refresh_shared_lock.sql:20-24`).
- `Db::begin_transaction` (`crates/buzz-db/src/lib.rs:648-650`) exposes a
  `Transaction<'static, Postgres>` to callers, justified by `PgPool` being
  `Arc`-backed.
- Session-scoped advisory locks are held on a **detached** connection so a
  locked session is never returned to the pool
  (`crates/buzz-db/src/lib.rs:511-535`, guard type at `:203-219`).

#### 6. Row mapping

No `#[derive(sqlx::FromRow)]` anywhere in the crate (zero matches). Every row is
decoded manually, which keeps enum columns readable as `::text` and lets
projections vary per query. Common shapes:

```rust
rows.into_iter().map(row_to_report).collect()                 // moderation.rs:236
row.map(row_to_report).transpose()                            // moderation.rs:260
row.map(|row| { Ok(Record { … row.try_get("x")? }) }).transpose()  // lib.rs:673
```

Byte columns are `Vec<u8>` in structs and hex-encoded only at the presentation
boundary (`crates/buzz-db/src/admin_moderation.rs:172-176`,
`crates/buzz-db/src/product_feedback.rs:112-113`). `pubkey_hex` in
`event_mentions` is the one place hex is the storage form, always lowercased on
write (`crates/buzz-db/src/lib.rs:140`) and on read predicates
(`crates/buzz-db/src/event.rs:377`).

#### 7. Testing patterns

Counts (all tests live in in-file `#[cfg(test)] mod tests`; there is no
`tests/` directory in the crate):

| Metric | Count |
|--------|-------|
| `#[test]` (pure, no infrastructure) | **81** |
| `#[tokio::test]` | **122** |
| `#[ignore]`-gated | **121** |
| Non-ignored `#[tokio::test]` | **1** (`read_falls_back_to_writer_when_no_replica_configured`, `crates/buzz-db/src/lib.rs:5361`, uses `connect_lazy` so it never touches the network) |
| Files with a `mod tests` | 19 of 22 |

Per file:

| File | `#[test]` | `#[tokio::test]` | `#[ignore]` |
|------|-----------|------------------|-------------|
| `workflow.rs` | 24 | 7 | 7 |
| `feed.rs` | 22 | 3 | 3 |
| `event.rs` | 14 | 12 | 12 |
| `migration.rs` | 7 | 3 | 3 |
| `user.rs` | 5 | 8 | 8 |
| `dm.rs` | 4 | 0 | 0 |
| `partition.rs` | 3 | 0 | 0 |
| `replica_fence.rs` | 2 | 3 | 3 |
| `lib.rs` | 0 | 25 | 24 |
| `push.rs` | 0 | 14 | 14 |
| `relay_members.rs` | 0 | 10 | 10 |
| `thread.rs` | 0 | 10 | 10 |
| `usage.rs` | 0 | 8 | 8 |
| `channel.rs` | 0 | 7 | 7 |
| `git_repo.rs` | 0 | 4 | 4 |
| `moderation.rs` | 0 | 4 | 4 |
| `api_token.rs` | 0 | 2 | 2 |
| `archived_identities.rs` | 0 | 1 | 1 |
| `product_feedback.rs` | 0 | 1 | 1 |
| `admin_moderation.rs`, `error.rs`, `reaction.rs` | 0 | 0 | 0 |

Conventions:
- Infra tests are gated `#[ignore = "requires Postgres"]` (or
  `"requires migrated Postgres"` at `crates/buzz-db/src/product_feedback.rs:124`).
- Test DB URL resolution: a `const TEST_DB_URL` default plus
  `BUZZ_TEST_DATABASE_URL` → `DATABASE_URL` (most modules) or `TEST_DATABASE_URL`
  (`lib.rs`, `replica_fence.rs`) — see `buzz-db-configuration.md`.
- Every infra test mints its own community via a local
  `make_test_community` / `make_community` helper with a UUID-derived host, so
  tests are isolated on a shared database (e.g.
  `crates/buzz-db/src/event.rs:1484-1495`).
- Cross-tenant isolation tests deliberately use **identical** ids/shapes in two
  communities (`crates/buzz-db/src/event.rs:1601`,
  `crates/buzz-db/src/channel.rs:1553`, `crates/buzz-db/src/lib.rs:4890`).
- Replica-routing tests create two scratch databases with **divergent** fixtures
  so every assertion observes which pool actually served the query
  (`crates/buzz-db/src/lib.rs:5216-5262`, `:5379`, `:5464`), and include
  explicit "counterfactual" assertions that pin the hazard an over-open fence
  would cause (`:5647-5661`, `:5741-5755`).
- Concurrency tests use `tokio::spawn` + `sleep` + `is_finished()` to assert a
  statement *blocks*, then `tokio::time::timeout` to assert it completes
  (`crates/buzz-db/src/lib.rs:4351-4380`, `crates/buzz-db/src/event.rs:1544-1574`).
- The migration lint tests hand-roll a small SQL parser (statement splitting
  respecting `'` and `$$`, paren matching, top-level CSV split) and assert
  tenant-isolation properties over the concatenation of **all** migrations
  (`crates/buzz-db/src/migration.rs:120-370`, `:635-688`).
- Pure unit tests concentrate on the code that has no DB dependency: validators
  (`partition.rs:153-181`), tag extraction (`event.rs:1968-2081`), hashing/ordering
  (`dm.rs:520+`), SQL-shape assertions built from `QueryBuilder`
  (`feed.rs:766-861`), and enum round-trips (`workflow.rs:1199+`).


## Module: buzz-auth (`crates/buzz-auth`)

### Aspect: Conventions

### Module organisation

8 source files, 1,877 LOC total. One file per concern; `lib.rs` holds the
aggregate types plus re-exports.

| File | LOC | Responsibility |
|------|-----|----------------|
| `crates/buzz-auth/src/lib.rs` | 243 | crate docs, module decls, re-exports, `AuthMethod`/`AuthContext`/`AuthConfig`/`AuthService`, dev key derivation |
| `crates/buzz-auth/src/access.rs` | 251 | `ChannelAccessChecker`, `require_scope`, read/write helpers, `MockAccessChecker` |
| `crates/buzz-auth/src/error.rs` | 59 | `AuthError` only |
| `crates/buzz-auth/src/nip42.rs` | 183 | challenge gen + AUTH verification + relay-URL normalisation |
| `crates/buzz-auth/src/nip98.rs` | 317 | HTTP auth verification + URL normalisation |
| `crates/buzz-auth/src/nip98_replay.rs` | 249 | `Nip98ReplayGuard`, key formats, TTL constants |
| `crates/buzz-auth/src/rate_limit.rs` | 326 | `RateLimiter`, `RateLimitConfig`, `LimitType`, `RateLimitResult`, key formats |
| `crates/buzz-auth/src/scope.rs` | 249 | `Scope`, `parse_scopes` |

Structural conventions observed:

- Every module has a `//!` module-level doc block; several encode the algorithm
  as a numbered list before the code (`crates/buzz-auth/src/nip42.rs:1-7`,
  `crates/buzz-auth/src/nip98.rs:13-24`).
- Every module ends with `#[cfg(test)] mod tests { use super::*; ... }` except
  `error.rs`, which has no tests (`crates/buzz-auth/src/error.rs` — 59 lines, no
  test module).
- Test doubles live in the same file as the trait they implement, gated on
  `#[cfg(any(test, feature = "test-utils"))]`, and are re-exported from `lib.rs`
  under the same gate (`crates/buzz-auth/src/lib.rs:46-51`).
- Private helpers are placed immediately adjacent to their only caller
  (`normalize_relay_url` above `verify_nip42_event`,
  `crates/buzz-auth/src/nip42.rs:19-33`; `normalize_url` below
  `verify_nip98_event`, `crates/buzz-auth/src/nip98.rs:145-153`).
- Crate lints are declared at the top of `lib.rs`: `#![deny(unsafe_code)]`,
  `#![warn(missing_docs)]` (`crates/buzz-auth/src/lib.rs:1-2`). Every public item
  carries a doc comment, including individual enum variants and struct fields.

---

### Naming conventions

| Category | Convention | Examples |
|----------|-----------|----------|
| Verification fns | `verify_<spec>_event` | `verify_nip42_event` (`nip42.rs:47`), `verify_nip98_event` (`nip98.rs:55`) |
| Redis key builders | `<subject>_key` / `<subject>_key_for_scope` | `rate_limit_key` (`rate_limit.rs:201`), `ip_rate_limit_key` (`rate_limit.rs:213`), `nip98_replay_key` (`nip98_replay.rs:114`), `nip98_replay_key_for_scope` (`nip98_replay.rs:119`) |
| Check fns | `check_<subject>` (async, returns `Result<(), AuthError>`) or `require_<subject>` (sync) | `check_read_access`/`check_write_access` (`access.rs:72`, `:88`), `require_scope` (`access.rs:60`) |
| Trait methods that mutate shared state | verb-first, `try_`-prefixed when they can legitimately decline | `try_mark`, `try_mark_in_scope` (`nip98_replay.rs:97`, `:66`); `check_and_increment` (`rate_limit.rs:174`) |
| Traits | noun-agent (`-er` / `-Checker` / `-Guard`) | `RateLimiter`, `ChannelAccessChecker`, `Nip98ReplayGuard` |
| Test doubles | `Always<Behaviour><Trait>` or `Mock<Subject>` | `AlwaysAllowRateLimiter` (`rate_limit.rs:219`), `AlwaysFreshReplayGuard` (`nip98_replay.rs:127`), `MockAccessChecker` (`access.rs:108`) |
| Constants | SCREAMING_SNAKE with unit suffix | `TIMESTAMP_TOLERANCE_SECS` (`nip42.rs:35`, `nip98.rs:32`), `DEFAULT_REPLAY_TTL_SECS` / `MAX_REPLAY_TTL_SECS` (`nip98_replay.rs:46`, `:59`) |
| serde default fns | `default_<tier>_<metric>`, abbreviated | `default_human_msg`, `default_agent_std_api`, `default_agent_plat_msg` (`rate_limit.rs:110-130`) |
| Enum-to-wire mapping | `as_str()` returning `&str` (or `&'static str`) | `Scope::as_str` (`scope.rs:114`), `LimitType::key_suffix` (`rate_limit.rs:71`) |
| Scope wire strings | `resource:action` lowercase, `admin:resource` for admin | `messages:read`, `admin:channels` (`scope.rs:116-131`) |
| Redis key namespace | `buzz:` prefix, community segment second, subject last | `buzz:{community}:ratelimit:{pubkey}:{suffix}` (`rate_limit.rs:203`), `buzz:{scope}:nip98:{id}` (`nip98_replay.rs:120`) |

---

### Error handling

Single crate-wide error enum, `thiserror`-derived, no `anyhow`, no boxed errors,
no custom `Display` impls (`crates/buzz-auth/src/error.rs:8-9`).

| Variant | `#[error(...)]` message | Constructed at |
|---------|-------------------------|----------------|
| `InvalidSignature` | `invalid signature or malformed auth event` | `nip42.rs:53` (wrong kind), `nip42.rs:56` (sig/id failure) |
| `ChallengeMismatch` | `challenge mismatch` | `nip42.rs:62` (tag missing), `nip42.rs:65` (mismatch) |
| `RelayUrlMismatch` | `relay url mismatch` | `nip42.rs:72` (tag missing), `nip42.rs:75` (mismatch) |
| `EventExpired` | `auth event timestamp outside ±60s window` | `nip42.rs:82` |
| `Nip98Invalid(String)` | `NIP-98 HTTP Auth verification failed: {0}` | `nip98.rs:63`, `:67`, `:75`, `:82`, `:95`, `:98`, `:108`, `:111`, `:123` (9 sites) |
| `Nip98Replay` | `NIP-98 replay: event id already seen within window` | **never constructed in this crate** — intended for callers of `Nip98ReplayGuard` (usage example at `nip98_replay.rs:24`) |
| `PubkeyMismatch` | `pubkey mismatch: event pubkey does not match authenticated identity` | **never constructed in this crate** |
| `InsufficientScope { required: String, have: Vec<String> }` | `insufficient scope: required {required}, have {have:?}` | `access.rs:64-67` |
| `ChannelAccessDenied` | `channel access denied` | `access.rs:83`, `access.rs:99` |
| `Internal(String)` | `internal auth error: {0}` | `lib.rs:132` (spawn_blocking panic), `lib.rs:165` (dev key derivation) |

Handling patterns:

- **Guard clauses with early return**, not nested conditionals — both verifiers
  are flat sequences of `if ... { return Err(...) }`
  (`crates/buzz-auth/src/nip42.rs:52-84`, `crates/buzz-auth/src/nip98.rs:66-127`).
- **Coarsening on the NIP-42 path**: the underlying `VerificationError` is
  discarded via `.map_err(|_| AuthError::InvalidSignature)`
  (`crates/buzz-auth/src/nip42.rs:56`) and wrong-kind reuses the same variant
  (`:53`) — a caller cannot distinguish "wrong kind" from "bad signature".
- **Descriptive on the NIP-98 path**: each failure carries a formatted string,
  including the offending values for URL/method mismatches
  (`crates/buzz-auth/src/nip98.rs:98-100`, `:111-113`). The doc explicitly says
  the message is "safe for server logs but should not be forwarded verbatim to
  clients" (`crates/buzz-auth/src/nip98.rs:53-54`).
- **Error-hygiene rule stated on the enum**: "Do **not** include raw token
  values, database contents, or stack traces in error messages"
  (`crates/buzz-auth/src/error.rs:5-7`).
- **Double-`?` for nested Results**: `spawn_blocking` join error and the inner
  verification error are both propagated on one line with `??`
  (`crates/buzz-auth/src/lib.rs:132`).
- **`Infallible` instead of a fallible parse**: `FromStr for Scope` cannot fail;
  unknown input becomes `Scope::Unknown(_)`
  (`crates/buzz-auth/src/scope.rs:143-166`).
- **One production-path `expect`**: `parse_scopes` calls
  `.expect("infallible: Scope::from_str cannot fail")`
  (`crates/buzz-auth/src/scope.rs:175`) — statically unreachable given the
  `Infallible` error type, but it is the only non-test `expect`/`unwrap` in the
  crate. Every other `unwrap`/`expect` occurrence is inside a `#[cfg(test)]`
  module.
- **No `unsafe`**: `#![deny(unsafe_code)]` (`crates/buzz-auth/src/lib.rs:1`); the
  only occurrence of the token `unsafe` in the whole crate is that attribute.
- **No logging on error paths**: despite `tracing` being a declared dependency
  (`crates/buzz-auth/Cargo.toml:20`), nothing in `src/` emits a log or span.

---

### Async conventions

- Traits use **RPITIT** (`-> impl Future<Output = ...> + Send`) for
  `ChannelAccessChecker` (`crates/buzz-auth/src/access.rs:35-39`, `:46-51`) and
  `RateLimiter` (`crates/buzz-auth/src/rate_limit.rs:174-181`, `:188-193`) —
  these are not dyn-compatible.
- `Nip98ReplayGuard` deliberately uses `Pin<Box<dyn Future + Send + 'a>>`
  instead (`crates/buzz-auth/src/nip98_replay.rs:66-71`, `:97-103`) so it can be
  stored as `Arc<dyn Nip98ReplayGuard>` by the relay
  (`crates/buzz-relay/src/state.rs:582`). Two different async-trait styles
  coexist in one crate, chosen per dyn-dispatch need.
- Default trait method bodies use `async move { ... }` blocks
  (`crates/buzz-auth/src/access.rs:52-55`) or `Box::pin(async move { ... })`
  (`crates/buzz-auth/src/nip98_replay.rs:101`).
- Free async fns are plain `pub async fn` (`crates/buzz-auth/src/access.rs:72`,
  `:88`).
- CPU-bound crypto is offloaded with `spawn_blocking` and the values are cloned
  into the closure first (`crates/buzz-auth/src/lib.rs:125-132`); the sync
  verifier documents the requirement in its own doc comment
  (`crates/buzz-auth/src/nip42.rs:46`).

---

### Configuration conventions

- Config structs derive `Serialize + Deserialize + Default`, and **every**
  `RateLimitConfig` field carries `#[serde(default = "fn")]` so partial input
  deserialises (`crates/buzz-auth/src/rate_limit.rs:85-108`).
- The manual `Default` impl duplicates the same seven `default_*` fns rather
  than using `#[derive(Default)]` with literals, keeping one source of truth for
  each number (`crates/buzz-auth/src/rate_limit.rs:132-144`).
- Nested config uses `#[serde(default)]` on the field
  (`crates/buzz-auth/src/lib.rs:93`).
- Enum wire forms use `#[serde(rename_all = "snake_case")]`
  (`crates/buzz-auth/src/rate_limit.rs:57`).

---

### Testing patterns

Totals by attribute (grep of `crates/buzz-auth/src/*.rs`):

| File | `#[test]` | `#[tokio::test]` | Total |
|------|-----------|------------------|-------|
| `crates/buzz-auth/src/nip98.rs` | 11 | 0 | 11 |
| `crates/buzz-auth/src/nip42.rs` | 8 | 0 | 8 |
| `crates/buzz-auth/src/nip98_replay.rs` | 6 | 1 | 7 |
| `crates/buzz-auth/src/rate_limit.rs` | 4 | 1 | 5 |
| `crates/buzz-auth/src/scope.rs` | 5 | 0 | 5 |
| `crates/buzz-auth/src/access.rs` | 0 | 5 | 5 |
| `crates/buzz-auth/src/lib.rs` | 1 | 3 | 4 |
| `crates/buzz-auth/src/error.rs` | 0 | 0 | 0 |
| **Total** | **35** | **10** | **45** |

There is no `crates/buzz-auth/tests/` directory — all 45 tests are in-file unit
tests.

Patterns:

- **Builder helpers per file**: `make_auth_event(keys, challenge, relay_url)`
  (`crates/buzz-auth/src/nip42.rs:95-100`, duplicated at
  `crates/buzz-auth/src/lib.rs:174-179`), `make_nip98_event(...)` with optional
  payload/timestamp params (`crates/buzz-auth/src/nip98.rs:163-186`).
- **`fixture_ctx(host)` deterministic tenant fixtures**: SHA-256 of the host name
  → first 16 bytes → `Uuid` → `CommunityId`, so key-prefix assertions are stable
  (`crates/buzz-auth/src/rate_limit.rs:253-260`,
  `crates/buzz-auth/src/nip98_replay.rs:149-155`). `access.rs` uses a simpler
  random-UUID variant (`crates/buzz-auth/src/access.rs:159-161`).
- **`assert!(matches!(result, Err(AuthError::X)))`** as the standard rejection
  assertion (`crates/buzz-auth/src/nip42.rs:124-127`,
  `crates/buzz-auth/src/access.rs:188-191`, `crates/buzz-auth/src/lib.rs:227`).
- **Real keypairs, never fixtures**: every test calls `Keys::generate()` and
  signs with `EventBuilder`, so signature verification is genuinely exercised
  (e.g. `crates/buzz-auth/src/nip98.rs:190-191`).
- **Const-drift tripwires**: assertions deliberately made over constants, with
  `#[allow(clippy::assertions_on_constants)]` and a comment explaining why
  (`crates/buzz-auth/src/nip98_replay.rs:210-230`,
  `crates/buzz-auth/src/nip98_replay.rs:232-237`).
- **Property-style invariant tests over key formats**: all-lowercase character
  scan (`crates/buzz-auth/src/rate_limit.rs:290-306`,
  `crates/buzz-auth/src/nip98_replay.rs:194-208`), cross-community
  distinctness (`crates/buzz-auth/src/rate_limit.rs:275-288`,
  `crates/buzz-auth/src/nip98_replay.rs:178-192`,
  `crates/buzz-auth/src/access.rs:225-251`).
- **Set-cardinality + no-duplicate + no-`Unknown` assertions** on the scope
  constructors (`crates/buzz-auth/src/scope.rs:205-248`).
- **Security-rationale comments inside tests** explaining what the assertion
  protects (`crates/buzz-auth/src/nip98.rs:290-294`,
  `crates/buzz-auth/src/rate_limit.rs:277-278`,
  `crates/buzz-auth/src/access.rs:227-230`).
- Test-only `unwrap`/`expect` is used freely; no test-helper crate, no
  `proptest`/`quickcheck`, no `mockall` — hand-written doubles only.


## Module: buzz-pubsub (`crates/buzz-pubsub`)
### Aspect: Conventions

---

### 1. Lint posture and repo-rule compliance

| Rule | Status | Evidence |
|---|---|---|
| No `unsafe` | Enforced at crate level | `#![deny(unsafe_code)]` `lib.rs:1`; zero `unsafe` blocks |
| Public API documented | Enforced at crate level | `#![warn(missing_docs)]` `lib.rs:2` — a `warn`, not `deny`, so a missing doc does not fail the build |
| No `unwrap()`/`expect()` in production paths | **Compliant** | Every `unwrap`/`expect` occurrence is inside `#[cfg(test)]` modules (`lib.rs:369-627`, `presence.rs:105-208`, `nip98_replay.rs:103-201`, `topic.rs:117-196`, `cache_invalidation.rs:183-250`, `conn_control.rs:169-228`, `subscriber.rs:190-204`). The single production `expect` is in `#[cfg(test)] mod test_util` (`lib.rs:374`) |
| Workspace-inherited manifest fields | Compliant | `version`/`edition`/`rust-version`/`license`/`repository` all `.workspace = true` (`Cargo.toml:3-7`) |
| Workspace dependency pinning | Compliant | All 13 deps use `{ workspace = true }` (`Cargo.toml:11-23`); no local version literals |
| No TODO/FIXME/HACK markers | Compliant | none present in any of the 10 files |

### 2. Error-handling conventions

- One `thiserror` enum per crate, `PubSubError` (`error.rs:5-28`), with `#[from]`
  conversions for the three foreign error types (`error.rs:8`, `:12`, `:16`) and a
  hand-written `From<broadcast::error::RecvError>` that maps lag and closure onto
  domain variants (`error.rs:31-38`).
- **Trait-imposed exception:** the two `buzz-auth` trait impls return
  `AuthError` instead, because the trait signatures demand it (`rate_limiter.rs:35`,
  `nip98_replay.rs:37`). Foreign errors are flattened into
  `AuthError::Internal(format!(...))` at `rate_limiter.rs:45`, `:52`, `:66` and
  `nip98_replay.rs:57`, `:79`, `:95` — a deliberate choice to keep the user-facing
  category string bounded, called out in a comment at `nip98_replay.rs:50-51`.
- **Resilience convention: never let one bad message kill a loop.** Every subscriber
  handles a malformed input with `tracing::warn!` + `continue`
  (`subscriber.rs:132-157`, `cache_invalidation.rs:144-165`,
  `conn_control.rs:135-156`).
- **Fire-and-forget convention:** where the DB is the durable backstop, send results
  are discarded with `let _ =` and the rationale is documented at the call site
  (`lib.rs:203-207`, `:236-243`; doc at `lib.rs:266-271`, `:285-291`).

### 3. Naming and structural conventions

- All Redis keys and channels are built through a helper, never inline-formatted at
  the call site; the prefix is a single constant `BUZZ_PREFIX` (`topic.rs:13`) reused
  by every module (`presence.rs:14`, `cache_invalidation.rs:20`, `conn_control.rs:23`).
- Key layout is uniformly `buzz:{community}:{kind}[:{id}]`. The one deliberate
  exception, IP rate limits, is documented as such (`rate_limiter.rs:84-85`).
- Channel-suffix constants are exported so producer and subscriber cannot drift:
  `CACHE_INVALIDATION_SUFFIX`/`_PATTERN` (`cache_invalidation.rs:23`, `:27`),
  `CONN_CONTROL_SUFFIX`/`_PATTERN` (`conn_control.rs:26`, `:30`).
- Every construct-a-key path takes `&TenantContext` rather than a raw
  `CommunityId`, so a caller cannot fabricate a tenant
  (`topic.rs:35`, `:103`, `:108`, `presence.rs:19`, `cache_invalidation.rs:30`,
  `conn_control.rs:33`).
- Paired parse/format functions live beside each other and are round-trip tested
  (`topic.rs:43`/`:53` tested `topic.rs:150-177`;
  `cache_invalidation.rs:30`/`:38` tested `:201-207`;
  `conn_control.rs:33`/`:38` tested `:183-188`).
- Builder pattern with a `DEFAULT_*` associated const for tunables
  (`lib.rs:82`, `:93-96`).
- Backoff constants are duplicated per module rather than shared:
  `BACKOFF_INITIAL_SECS`/`BACKOFF_MAX_SECS` appear three times
  (`subscriber.rs:16-19`, `cache_invalidation.rs:91-94`, `conn_control.rs:81-84`)
  with identical values.

### 4. Concurrency conventions

- `Arc<Self>` receivers for infinite background loops (`lib.rs:148`, `:165`, `:175`);
  `PubSubManager` is intentionally not `Clone` (`lib.rs:100`).
- Take-once initialisation guard: the `mpsc::Receiver` is stored as
  `Mutex<Option<..>>` and `take()`n so a second `run_subscriber` is a logged no-op
  rather than a panic or a silent double-consume (`lib.rs:107`, `:149-152`).
- Locks are held for the shortest possible span: the refcount mutations compute a
  boolean inside a block, then the lock drops before any `await` on the channel
  (`lib.rs:194-207`, `:217-231`) — avoids holding a `tokio::Mutex` across `await`.
- `tokio::select!` for multiplexing commands against the message stream, with an
  `else` arm for total-shutdown detection (`subscriber.rs:110-171`).

### 5. Documentation conventions

- Crate-level ASCII architecture diagram (`lib.rs:8-16`) plus explicit "why" notes
  for non-obvious choices: why the pub/sub connection is not pooled (`lib.rs:19-20`),
  why cache-invalidation and conn-control are separate channels
  (`conn_control.rs:12-18`), why topics are not an isolation boundary
  (`topic.rs:3-6`, `lib.rs:305-320`), why the TTL is 3× the heartbeat
  (`presence.rs:4-6`).
- Known-limitation callouts are inline rather than in a separate doc: the fixed-window
  2× burst warning (`rate_limiter.rs:9-10`), the "upgrade to sliding window" note
  (same lines), and the `⚠️` marker convention.
- Contract obligations are written into log messages, not just docs — e.g. "caller
  MUST fail closed" appears in the `warn!` payloads at `nip98_replay.rs:55`, `:76`.

### 6. Testing conventions

34 test functions across 8 of 10 files. 11 require live Redis and are gated
`#[ignore = "requires Redis"]` — a consistent, uniform gate string
(`lib.rs:400`, `:438`, `:476`, `:511`; `presence.rs:137`, `:160`, `:187`;
`nip98_replay.rs:128`, `:145`, `:165`, `:180`).

| File | Tests | Redis-gated |
|---|---|---|
| `lib.rs` | 6 | 4 |
| `topic.rs` | 6 | 0 |
| `conn_control.rs` | 6 | 0 |
| `cache_invalidation.rs` | 5 | 0 |
| `presence.rs` | 5 | 3 |
| `nip98_replay.rs` | 4 | 4 |
| `subscriber.rs` | 2 | 0 |
| `error.rs`, `publisher.rs`, `rate_limiter.rs` | 0 | — |

Conventions observed:
- A shared `#[cfg(test)] mod test_util` provides `make_test_pool()` (`lib.rs:369-377`)
  reused by `presence.rs:107`; `nip98_replay.rs` instead builds its own pool honouring
  a `REDIS_URL` env override (`nip98_replay.rs:110-116`) — an inconsistency in an
  otherwise uniform pattern.
- Every module defines an identical local `fn ctx(id, host)` helper
  (`lib.rs:388`, `topic.rs:119`, `presence.rs:119`, `cache_invalidation.rs:185`,
  `conn_control.rs:171`, and inline at `subscriber.rs:191`) — six copies of the same
  three-line fixture.
- Negative-path tables: malformed inputs are asserted as a `for` loop over a literal
  array (`topic.rs:179-195`, `cache_invalidation.rs:222-233`).
- Serde round-trip tests for every wire enum variant (`cache_invalidation.rs:235-249`,
  `conn_control.rs:202-207`, `:219-227`) plus a forward-compat test asserting an
  unknown `op` is rejected without poisoning subsequent messages
  (`conn_control.rs:209-217`).
- Intent-documenting test names and comments: `lib.rs:556-557` explains that the test
  exists to catch a channel-id-only keying bug; `nip98_replay.rs:184-190` explains why
  clamping beats propagating an error.
- `test_presence_set_and_get` is **duplicated** — defined at both `lib.rs:477` and
  `presence.rs:138` with overlapping assertions.

`rate_limiter.rs` has **zero tests** despite implementing a security control; its
behaviour is covered only indirectly through `buzz-relay/src/admission.rs`'s
`StubLimiter` (`admission.rs:65-90`), which stubs out the Redis logic entirely — so
the Lua script, the `count <= limit` boundary (`rate_limiter.rs:74`), and the TTL
repair path (`rate_limiter.rs:57-70`) are untested anywhere in the repo.


## Module: buzz-search (`crates/buzz-search`)

### Aspect: Conventions

#### Module organization

| Module | File | Role |
|---|---|---|
| crate root | `src/lib.rs` (54 LOC) | lints, module decls, re-exports, `SearchService` handle |
| `error` | `src/error.rs` (9 LOC) | `SearchError` only |
| `query` | `src/query.rs` (352 LOC) | all query types + SQL construction + execution + unit tests |
| integration tests | `tests/fts_integration.rs` (1448 LOC) | Postgres-backed behavior tests |

Both modules are declared `pub mod` with doc comments on the declaration itself
(`lib.rs:24-27`), and their contents are flattened into the crate root by a single
`pub use` line each (`lib.rs:29-31`). Callers use both spellings in-tree:
`buzz_search::SearchService` (`crates/buzz-relay/src/state.rs:28`) and
`buzz_search::ChannelScope` / `SearchQuery` / `SearchMode` via the root
(`crates/buzz-relay/src/api/bridge.rs:1665`, `1687`, `1697`).

#### Lints

| Lint | Line |
|---|---|
| `#![deny(unsafe_code)]` | `lib.rs:1` |
| `#![warn(missing_docs)]` | `lib.rs:2` |

`missing_docs` is honored throughout: every public item, field, and enum variant
carries a doc comment (`query.rs:43-52`, `75-98`, `106-117`, `123-126`;
`error.rs:3-6`; `lib.rs:35-51`). One `#[allow]` appears in the crate, in tests
only: `#[allow(clippy::too_many_arguments)]` on the `insert_event` helper
(`tests/fts_integration.rs:118`).

#### Naming

| Pattern | Examples |
|---|---|
| Types: `Search*` prefix for the public surface | `SearchService`, `SearchQuery`, `SearchHit`, `SearchResult`, `SearchMode`, `SearchError` |
| Enum variants read as constraints, not flags | `Any`, `ChannelLessOnly`, `Channels`, `ChannelsOrChannelLess` (`query.rs:44-52`) |
| Private helpers: verb-first (`push_*`) or noun-phrase for pure fns | `push_tsquery` (`query.rs:140`), `normalized_search_text` (`query.rs:179`) |
| Constants: SCREAMING_SNAKE with the bound in the name | `PER_PAGE_MAX`, `PER_PAGE_DEFAULT`, `SEARCH_TEXT_MAX_CHARS`, `PAGE_MAX` (`query.rs:129-138`) |
| SQL aliases spelled out | `created_at_s`, `search_query.query`, `prefix_terms`, `raw_token`, `normalized` (`query.rs:235-237`, `168-176`) |
| Test names assert behavior, not method names | `search_does_not_return_other_community_events`, `channel_less_only_excludes_per_channel_events`, `excluded_kinds_are_storage_level_unsearchable` |

#### Error handling

Single-variant `thiserror` enum (`error.rs:4-9`):

| Variant | Attributes | Message |
|---|---|---|
| `Db(sqlx::Error)` | `#[from]`, `#[error("database error: {0}")]` | `error.rs:7-8` |

Conventions observed:
- All fallible steps use `?` — no `unwrap()`/`expect()`/`panic!` anywhere in `src/`
  (checked in all three source files), matching the repo rule against new
  `unwrap()`/`expect()` in production paths.
- Domain-shaped decode failures are expressed by re-wrapping into
  `sqlx::Error::Decode` with a message that names the column and the observed
  length, rather than adding an enum variant (`query.rs:306-311`).
- Degenerate input is handled by returning a valid empty result rather than an
  error (`query.rs:217-222`).

#### Query style

- One statement, assembled with `QueryBuilder` in strict order: projection + `FROM`
  seed (`query.rs:233-238`), tenant predicate (`240-241`), always-on predicates
  (`242`), optional predicates (`248-293`), `ORDER BY`/`LIMIT`/`OFFSET`
  (`295-298`).
- Every dynamic value goes through `push_bind`; `push` is used only for fixed SQL
  text (see integrations doc for the full bind list).
- Vectors are `.clone()`d into the bind because the builder needs owned values
  (`query.rs:257`, `262`, `270`, `278`).
- Multi-line SQL uses trailing `\` line continuations inside one string literal
  (`query.rs:234-237`, `154-176`).
- Non-obvious SQL carries an adjacent rationale comment (prefix-mode design at
  `query.rs:148-153`; channel-scope mapping at `query.rs:244-247`).
- Doc comments state the invariant in prose next to the code that enforces it
  ("`community_id = $ctx` is the first predicate and is non-negotiable",
  `query.rs:209-210`).

#### Testing patterns

| Metric | Count | Where |
|---|---|---|
| `#[test]` (sync unit tests) | 3 | `query.rs:329`, `338`, `346` inside `#[cfg(test)] mod tests` (`query.rs:325-352`) |
| `#[tokio::test]` (async integration tests) | 18 | `tests/fts_integration.rs` |
| `#[ignore = "requires Postgres"]` | 18 | every integration test; none of the 3 unit tests is ignored |

So 21 tests total, 18 infra-gated. Unit tests cover only
`normalized_search_text` (trim/reject-empty, NUL replacement, char cap).

Integration-test conventions:
- Per-test isolated schema named `fts_test_<uuid-simple>`, created and dropped
  around each test, declared parallel-safe (`tests/fts_integration.rs:6-8`,
  `35-46`, `87-103`).
- Full migration chain replayed so the test schema matches production
  (`tests/fts_integration.rs:55-84`).
- Test-only DDL string interpolation is explicitly marked with
  `sqlx::AssertSqlSafe` (`tests/fts_integration.rs:44`, `100`).
- Fixture helpers: `mk_community`, `insert_event`, `rand_bytes32`
  (`tests/fts_integration.rs:105-153`).
- Deterministic timestamps in the `1_700_000_000` family, unique content tokens
  per test to avoid cross-test coupling.
- Several tests document their own mutation-kill argument — flip the predicate and
  the assertion goes red (`tests/fts_integration.rs:877-887`, `1100-1104`).
- Two tests are explicit drift tripwires that iterate Rust constants
  (`AUTHOR_ONLY_KINDS`, persistent subset of `P_GATED_KINDS`) against the schema's
  inlined exclusion list (`tests/fts_integration.rs:1256-1265`, `1338-1361`).
- Run instruction is documented at the top of the file, including the
  `BUZZ_TEST_DATABASE_URL` override and `-- --include-ignored`
  (`tests/fts_integration.rs:3`).


## Module: buzz-audit (`crates/buzz-audit`)

### Aspect: Conventions

### Module organization

| File | Lines | Role | Declared in `lib.rs` |
|---|---|---|---|
| `src/lib.rs` | 35 | crate docs, lints, module declarations, re-exports | — |
| `src/action.rs` | 100 | `AuditAction` enum + string mapping + `FromStr`/`Display` | `lib.rs:21` |
| `src/entry.rs` | 72 | `AuditEntry` (stored) and `NewAuditEntry` (input) | `lib.rs:23` |
| `src/error.rs` | 108 | `AuditError` | `lib.rs:25` |
| `src/hash.rs` | 272 | `GENESIS_HASH`, `to_storage_precision`, `compute_hash`, `canonical_json` | `lib.rs:27` |
| `src/service.rs` | 527 | `AuditService` (append/verify/read) + row decode | `lib.rs:29` |

One concept per file; no `mod.rs` nesting, no `tests/` directory. Every module gets a
doc comment at its declaration site (`lib.rs:20-29`) in addition to its own file-level
items.

Crate-level lints: `#![deny(unsafe_code)]` and `#![warn(missing_docs)]`
(`lib.rs:1-2`). Public items are documented throughout (e.g. `action.rs:5`, `:9-30`,
`:34`; `entry.rs:8-12`, `:15-36`; `service.rs:31-36`, `:42`, `:47-51`).

Selective re-export at the root rather than a blanket `pub use`: `AuditAction`,
`AuditEntry`, `NewAuditEntry`, `AuditError`, `compute_hash`, `GENESIS_HASH`,
`AuditService` (`lib.rs:31-35`). `to_storage_precision` stays module-qualified
(`hash.rs:22`).

### Naming

- Types: `UpperCamelCase` — `AuditService`, `AuditEntry`, `NewAuditEntry`,
  `AuditAction`, `AuditError`. The `New*` prefix marks the pre-persistence input
  (`entry.rs:52`).
- Functions: `snake_case` verbs — `log`, `verify_chain`, `get_entries`, `compute_hash`,
  `canonical_json`, `to_storage_precision`, `log_timestamp`, `row_to_audit_entry`.
  `*_inner` marks the private continuation of a public wrapper (`service.rs:82`).
- Constants: `SCREAMING_SNAKE_CASE` — `GENESIS_HASH` (`hash.rs:9`),
  `AUDIT_LOCK_NAMESPACE` (`service.rs:29`), private `ALL` (`action.rs:51`).
- DB action strings are snake_case of the variant name (`action.rs:37-47`).
- Row-decode helper is a free function, not a `From` impl, because it must return
  `Result` (`service.rs:238`).
- Aliased import to avoid trait-name pollution: `use futures_util::FutureExt as _`
  (`service.rs:2`).

### Error handling

Single crate error enum `AuditError`, `thiserror`-derived (`error.rs:11-41`):

| Variant | Payload | `#[error]` message | Line | Constructed at |
|---|---|---|---|---|
| `Database` | `#[from] sqlx::Error` | `database error: {0}` | `error.rs:14-15` | implicitly by `?` on every sqlx call (`service.rs:54`, `:62`, `:101`, `:147`, `:149`, `:179`, `:232`) |
| `ChainViolation` | `{ seq: i64 }` | `hash chain integrity violation at seq {seq}: prev_hash does not match preceding entry` | `error.rs:19-25` | `service.rs:193` |
| `HashMismatch` | `{ seq: i64 }` | `hash mismatch at seq {seq}: stored hash does not match recomputed hash` | `error.rs:28-32` | `service.rs:199` |
| `UnknownAction` | none | `unknown audit action in database` | `error.rs:35-36` | `service.rs:242` |
| `Serialization` | `#[from] serde_json::Error` | `serialization error: {0}` | `error.rs:39-40` | via `?` on `canonical_json` (`hash.rs:67`) |

Conventions observed:
- `?` propagation everywhere in production paths; **no `unwrap()`/`expect()`/`panic!`/
  `unimplemented!()` outside `#[cfg(test)]`** (grep confirms all 38 hits are inside test
  modules, e.g. `service.rs:304`, `hash.rs:155`).
- `#[from]` used only for foreign error types (`error.rs:15`, `:40`); domain variants
  are constructed explicitly with named fields.
- Deliberate error-sanitization rule: no variant carries a `community_id` or constraint
  name, documented at `error.rs:3-10` and pinned by a test that asserts the rendered
  text contains neither the community UUID (both hyphenated and simple forms) nor the
  strings `community_id`, `audit_log_pkey`, `constraint`, `communities`
  (`error.rs:58-107`).
- One intentional error suppression: the advisory unlock result is discarded with
  `let _ = ...` (`service.rs:71`).
- `FromStr for AuditAction` uses `Err = String` (`action.rs:73`, message at `:80`); the
  service discards that string and substitutes `UnknownAction` after a `warn!`
  (`service.rs:240-243`).

### Tracing / observability conventions

- `#[instrument(skip(self, entry), fields(action = %entry.action))]` on `log`
  (`service.rs:52`) — the entry itself is skipped so `detail` never lands in a span.
- `#[instrument(skip(self))]` on `verify_chain` (`service.rs:159`) and `get_entries`
  (`service.rs:211`), so `community`/`seq` args *are* recorded.
- `debug!(seq, "writing audit entry")` (`service.rs:128`); `warn!("unknown action in
  audit log")` without the offending value (`service.rs:241`).
- No metrics emitted from this crate; counters/histograms live in the relay
  (`crates/buzz-relay/src/state.rs:1201-1206`).

### Documentation conventions

Doc comments carry rationale, not just description — e.g. why the pre-image order is
frozen (`hash.rs:28-30`), why timestamps are truncated (`hash.rs:11-21`), why
`NewAuditEntry` is not `Deserialize` (`entry.rs:46-51`), why the lock is per-community
(`service.rs:25-28`), why `detail` must not hold tokens (`entry.rs:64-71`). Intra-doc
links are used (`[`crate::hash::GENESIS_HASH`]` at `entry.rs:22`,
`[`AuditService::log_inner`]` at `service.rs:19`).

### Testing patterns

Counts across `crates/buzz-audit/src`:

| Metric | Count | Locations |
|---|---|---|
| `#[cfg(test)] mod tests` blocks | 4 | `action.rs:84`, `error.rs:43`, `hash.rs:118`, `service.rs:258` |
| `#[test]` (sync) | 13 | `action.rs:88,96`; `error.rs:58`; `hash.rs:152,159,167,184,201,216,226,256,266`; `service.rs:283` |
| `#[tokio::test]` (async) | 6 | `service.rs:318,338,376,437,475,512` |
| **Total tests** | **19** | — |
| `#[ignore = "requires Postgres"]` | 6 | `service.rs:319,339,377,438,476,513` — exactly the 6 async tests |
| Tests runnable without infra | 13 | all `#[test]` |

Patterns:
- Fixture builders instead of literals: `sample_entry()` (`hash.rs:125-141`),
  `new_entry()` (`service.rs:307-315`), `make_community()` which inserts an FK-satisfying
  `communities` row with a unique host (`service.rs:296-305`).
- Intent-naming helpers that document the scenario: `nanosecond_instant()`
  (`hash.rs:145-147`), `after_database_round_trip()` (`hash.rs:150-152`).
- Infra tests degrade gracefully rather than fail when Postgres is absent:
  `PgPool::connect(...).await.ok()` then `let Some(pool) = ... else { return; }`
  (`service.rs:275-280`, used at `:321-323` etc.).
- Shared-table serialization via a `static OnceLock<tokio::sync::Mutex<()>>` guard taken
  at the top of each DB test (`service.rs:263-267`, `_g = db_lock().lock().await` at
  `:320`, `:340`, `:378`, `:439`, `:477`, `:514`).
- Tampering is simulated with raw SQL `UPDATE`/`INSERT` against the table
  (`service.rs:459-465`, `:492-505`) — testing detection, not the API.
- `matches!` assertions on error variants (`service.rs:472`, `:509`).
- Test doc comments state the property under test (`service.rs:281-282`, `:369-372`,
  `:475-477`).
- One unit test exists specifically so a regression is caught by `just test-unit`
  rather than only by the ignored tests (`service.rs:281-292`).

### Naming/typing convention at the tenant boundary

Input uses the newtype (`NewAuditEntry.community_id: CommunityId`, `entry.rs:57`);
storage and read-back use the raw `Uuid` (`entry.rs:16`, `service.rs:246`); the
conversion happens once, explicitly, at the DB boundary with a comment marking it
(`service.rs:89-91`). Query methods take `CommunityId` and bind `community.as_uuid()`
(`service.rs:162`, `:175`, `:214`, `:228`).


## Module: buzz-media (`crates/buzz-media`)

### Aspect: Conventions

### 1. Module organization

Flat, one concern per file, all modules `pub` and re-exported selectively (`crates/buzz-media/src/lib.rs:5-28`):

| File | LOC | Concern |
|---|---|---|
| `src/lib.rs` | 29 | module declarations + curated re-exports |
| `src/types.rs` | 31 | wire response type (`BlobDescriptor`) |
| `src/thumbnail.rs` | 51 | sync CPU-bound derived artifacts |
| `src/config.rs` | 187 | config struct + startup validation |
| `src/error.rs` | 198 | error enum + HTTP mapping |
| `src/storage.rs` | 404 | S3 client and key builders |
| `src/upload_record.rs` | 419 | moderation side-channel records |
| `src/auth.rs` | 552 | Blossom kind-24242 verification |
| `src/upload.rs` | 732 | upload pipelines (orchestration) |
| `src/bucket_index.rs` | 755 | key taxonomy + pure accounting fold |
| `src/validation.rs` | 2594 | all content validation |
| `tests/static_creds_minio.rs` | 75 | live MinIO round-trip (`#[ignore]`) |

Layering is one-directional: `upload.rs` orchestrates and depends on `auth`, `config`, `error`, `storage`, `thumbnail`, `types`, `upload_record`, `validation` (`crates/buzz-media/src/upload.rs:1-20`); `validation.rs` and `bucket_index.rs` have no S3 dependency at all (`crates/buzz-media/src/bucket_index.rs:4-6`).

Notable structural convention: `bucket_index.rs` is explicitly written as I/O-free pure logic with an injected page-fetcher so it can be driven by synthetic listings in tests (`crates/buzz-media/src/bucket_index.rs:4-14`).

---

### 2. Naming

| Pattern | Examples |
|---|---|
| `validate_*` for fail-closed checks returning `Result` | `validate_content`, `validate_file_content`, `validate_video_file`, `validate_image_metadata_free`, `validate_jpeg_metadata_free`, `validate_mp4_metadata_free` (`crates/buzz-media/src/validation.rs:159`, `:238`, `:289`, `:492`, `:502`, `:831`) |
| `looks_like_*` for boolean structural probes | `looks_like_iso_bmff`, `looks_like_mp4_iso_bmff` (`crates/buzz-media/src/validation.rs:48`, `:52`) |
| `verify_blossom_*` for auth | `verify_blossom_auth_event_for_verb`, `verify_blossom_auth_event`, `verify_blossom_upload_auth`, `verify_blossom_get_auth` (`crates/buzz-media/src/auth.rs:31`, `:147`, `:175`, `:207`) |
| `process_*_upload` for pipelines | `process_upload`, `process_file_upload`, `process_video_upload` (`crates/buzz-media/src/upload.rs:207`, `:245`, `:292`) |
| `*_key` for object-key builders | `sidecar_key`, `ctx_sidecar_key`, `upload_record_key` (`crates/buzz-media/src/storage.rs:183`, `:188`, `crates/buzz-media/src/upload_record.rs:181`) |
| `parse_*` returning `Option` for lenient parses | `parse_public_ip`, `parse_port`, `parse_thumb_key`, `parse_blob_key`, `parse_sidecar_key`, `parse_auxiliary_key`, `parse_canonical_uuid` (`crates/buzz-media/src/upload_record.rs:191`, `:197`, `crates/buzz-media/src/bucket_index.rs:129`-`:172`, `:112`) |
| `is_*` predicates | `is_sha256`, `is_blob_ext`, `is_ulid_charset`, `is_public_ip`, `is_snapshot_text_chunk` (`crates/buzz-media/src/bucket_index.rs:75`, `:84`, `:93`, `crates/buzz-media/src/upload_record.rs:207`, `crates/buzz-media/src/validation.rs:584`) |
| `_sync` suffix marks CPU-bound functions meant for `spawn_blocking` | `generate_image_metadata_sync` (`crates/buzz-media/src/thumbnail.rs:15`) |
| SCREAMING_SNAKE consts for policy tables and bounds | `ALLOWED_MIME_TYPES`, `BLOCKED_FILE_MIME_TYPES`, `MP4_BRANDS`, `PNG_SNAPSHOT_KEYWORDS`, `MAX_PIXELS`, `MAX_ATOMS`, `MAX_BOXES`, `MAX_BOX_DEPTH`, `MIN_SNIFF_BYTES`, `BUF`, `UPLOAD_RECORD_VERSION` |

---

### 3. Error handling

Single crate error enum `MediaError` (`crates/buzz-media/src/error.rs:8-86`), `thiserror`-derived, 35 variants, plus a separate `SweepError` for the accounting fold (`crates/buzz-media/src/bucket_index.rs:341-362`).

| Variant | Message | HTTP status |
|---|---|---|
| `UnknownContentType` | `unknown content type` | 415 |
| `DisallowedContentType(String)` | `disallowed content type: {0}` | 415 |
| `FileTooLarge { size: u64, max: u64 }` | `file too large: {size} bytes (max {max})` | 413 |
| `ImageTooLarge` | `image dimensions too large` | 413 |
| `InvalidImage` | `invalid image data` | 422 |
| `MetadataForbidden` | `media contains metadata or a non-canonical metadata channel` | 422 |
| `InvalidSignature` | `invalid signature` | 401 (generic) |
| `InvalidAuthKind` | `invalid auth event kind` | 401 |
| `InvalidAuthVerb` | `invalid auth verb` | 401 |
| `MissingTag(&'static str)` | `missing required tag: {0}` | 401 |
| `HashMismatch` | `hash mismatch` | 401 |
| `ServerMismatch` | `server mismatch` | 401 |
| `TokenExpired` | `token expired` | 401 |
| `TimestampOutOfWindow` | `timestamp out of window` | 401 |
| `StorageError(String)` | `storage error: {0}` | 500 (body flattened) |
| `Internal` | `internal error` | 500 |
| `NotFound` | `not found` | 404 |
| `MissingAuth` | `missing authorization header` | 401 |
| `InvalidAuthScheme` | `invalid authorization scheme` | 401 |
| `InvalidBase64` | `invalid base64 encoding` | 401 |
| `InvalidAuthEvent` | `invalid auth event` | 401 |
| `Unauthorized` | `unauthorized` | 401 |
| `InsufficientScope` | `insufficient scope` | 403 |
| `RelayMembershipRequired` | `relay membership required` | 403 |
| `TokenRevoked` | `token revoked` | 401 |
| `PubkeyMismatch` | `pubkey mismatch` | 401 |
| `UploadRateLimitExceeded` | `upload rate limit exceeded` | 429 |
| `UploadConcurrencyLimitReached` | `upload concurrency limit reached` | 429 |
| `WrongCodec` | `unsupported media codec: only H.264 video and AAC audio are accepted` | 415 |
| `DurationTooLong` | `video too long: duration exceeds 600 seconds` | 422 |
| `ResolutionTooHigh` | `video resolution too high: maximum is 3840x2160` | 422 |
| `MoovNotAtFront` | `moov atom not at front of file (not fast-start)` | 422 |
| `UnsupportedContainer` | `unsupported container: only MP4 is accepted` | 415 |
| `InvalidVideo` | `invalid video data` | 422 |
| `Io(String)` | `io error: {0}` | 500 |

Conventions visible in the mapping (`crates/buzz-media/src/error.rs:106-160`):
- All 15 authentication-failure variants collapse to a single `401 "authentication failed"` body, explicitly "to prevent oracle enumeration"; `InsufficientScope` stays 403 because it is authorization, not authentication (`crates/buzz-media/src/error.rs:120-146`).
- 5xx bodies are flattened to `"internal error"` and logged at `error` (`crates/buzz-media/src/error.rs:154-158`).
- Errors are converted, never `unwrap`ped: three `From` impls (`image::ImageError` → `InvalidImage`, `S3Error` → `StorageError`, `serde_json::Error` → `StorageError`) at `crates/buzz-media/src/error.rs:88-104`.
- `MediaConfig::validate` returns `Result<(), String>` (plain strings, not `MediaError`) because it is a startup check (`crates/buzz-media/src/config.rs:66`).
- Some variants are declared here but not constructed in this crate (`Unauthorized`, `TokenRevoked`, `PubkeyMismatch`, `RelayMembershipRequired`, `MissingAuth`, `InvalidAuthScheme`, `InvalidBase64`, `UploadRateLimitExceeded`, `UploadConcurrencyLimitReached`) — they exist for relay handlers that share the type; see the Debt aspect.

---

### 4. Async patterns

| Pattern | Usage |
|---|---|
| CPU-bound work always inside `tokio::task::spawn_blocking` | validation+hash+auth (`crates/buzz-media/src/upload.rs:79-89`), video auth (`crates/buzz-media/src/upload.rs:410-414`), MP4 validation (`crates/buzz-media/src/upload.rs:416-419`), thumbnail (`crates/buzz-media/src/upload.rs:518-524`) |
| Join errors mapped, never unwrapped | `.map_err(\|_\| MediaError::Internal)??` (double `?` over join + inner result) at `crates/buzz-media/src/upload.rs:87-88`, `:414`, `:419`, `:524` |
| Owned inputs cloned into blocking closures; `Bytes` clones are refcount bumps (documented) | `crates/buzz-media/src/upload.rs:193-201` |
| Generic closure injection instead of trait objects for the two variable steps of the buffered pipeline | `crates/buzz-media/src/upload.rs:54-63` |
| Streaming rather than buffering for large payloads | `StreamReader` + 64 KiB chunks to temp file (`crates/buzz-media/src/upload.rs:325-395`), 8 MiB `BufReader` upload (`crates/buzz-media/src/storage.rs:91-101`), `ByteStream` download (`crates/buzz-media/src/storage.rs:131-146`) |
| Sync-blocking `std::fs` used deliberately inside blocking contexts | `crates/buzz-media/src/validation.rs:295`, `:415`, `:921` |
| Async page fetcher expressed as `FnMut(Option<String>) -> Future` | `crates/buzz-media/src/bucket_index.rs:377-383` |

---

### 5. Documentation conventions

- Every file opens with a `//!` module doc (`crates/buzz-media/src/lib.rs:1-3`, `crates/buzz-media/src/upload_record.rs:1-48`, `crates/buzz-media/src/bucket_index.rs:1-19`).
- All public items carry `///` doc comments; several document *why* a rule exists, including consumer contracts (`crates/buzz-media/src/upload_record.rs:29-48`) and threat rationale (`crates/buzz-media/src/validation.rs:66-86`).
- A markdown table inside module docs describes the key taxonomy (`crates/buzz-media/src/bucket_index.rs:12-19`).
- Spec references are inline: BUD-01 (`crates/buzz-media/src/auth.rs:201-205`, `crates/buzz-media/src/storage.rs:378`), BUD-02 (`crates/buzz-media/src/types.rs:1`), BUD-11 §5/§6 (`crates/buzz-media/src/auth.rs:47`, `:124`, `:189`).

---

### 6. Testing patterns

| Metric | Count |
|---|---|
| `#[test]` (sync) in `src/` | **98** |
| `#[tokio::test]` | **6** (5 in `crates/buzz-media/src/bucket_index.rs:538-661`, 1 in `crates/buzz-media/tests/static_creds_minio.rs:44`) |
| Total tests | **104** |
| `#[ignore]` | **1** — `crates/buzz-media/tests/static_creds_minio.rs:45` (`"requires a live MinIO (docker compose up -d minio minio-init)"`) |
| Integration test files | 1 (`tests/static_creds_minio.rs`) |
| Binary fixtures | 12 PNG/JPEG files under `crates/buzz-media/tests/fixtures/{android,ios}` |

Per-file distribution: `validation.rs` 47, `bucket_index.rs` 16 + 5 async, `auth.rs` 14, `upload_record.rs` 7, `config.rs` 4, `storage.rs` 4, `upload.rs` 4, `error.rs` 2; `lib.rs`, `types.rs`, `thumbnail.rs` have **zero**.

Patterns:
- `#[cfg(test)] mod tests` at the bottom of each file with a local `test_config()`/`valid_config()`/`storage_config()` builder (`crates/buzz-media/src/validation.rs:945-983`, `crates/buzz-media/src/config.rs:128-145`, `crates/buzz-media/src/storage.rs:281-301`, `crates/buzz-media/src/upload.rs:566-584`).
- Hand-built binary fixtures as consts (`TINY_JPEG`, `TINY_PNG`, `TINY_GIF`, `MP4_FTYP_MAGIC`, `TINY_PDF`, `TINY_ZIP`) plus ~30 MP4 box-builder helpers (`crates/buzz-media/src/validation.rs:1651-2150`).
- Real-device fixtures compiled in with `include_bytes!` to pin the mobile-sanitizer contract (`crates/buzz-media/src/validation.rs:1164-1280`).
- "Prove the fixture" pattern: an independent parser asserts the fixture really contains GPS EXIF before the validator is exercised (`crates/buzz-media/src/validation.rs:1044-1075`).
- Negative-first assertions via `matches!(result, Err(MediaError::X))` (`crates/buzz-media/src/validation.rs:1298-1305`).
- Mutation-resistance tests named for the property they defend, e.g. `same_sha_sidecars_do_not_bleed_between_communities` (`crates/buzz-media/src/storage.rs:352-377`) and `malformed_uploads_key_is_unknown_not_auxiliary` (`crates/buzz-media/src/bucket_index.rs:490-506`).
- Async tests drive the sweep fold through canned `Page`s and an `Arc<Mutex<Vec<Page>>>` script (`crates/buzz-media/src/bucket_index.rs:551-600`).
- Env-overridable integration config (`BUZZ_S3_ENDPOINT`/`ACCESS_KEY`/`SECRET_KEY`/`BUCKET`) in the ignored test only (`crates/buzz-media/tests/static_creds_minio.rs:22-34`).

---

### 7. Repo-guideline compliance

| Guideline (AGENTS.md) | Status in this crate |
|---|---|
| No `unsafe` | Satisfied — zero occurrences of `unsafe` across all 12 files |
| No new `unwrap()`/`expect()` in production paths | Mostly satisfied: 9 remaining non-test `.unwrap()` calls, all `try_into()` on fixed-size slices behind explicit length checks (`crates/buzz-media/src/validation.rs:604`, `:605`, `:673`, `:674`, `:697`, `:706`, `:707`, `:885`, `:886`); plus one `unwrap_or_default()` that intentionally swallows blurhash failure (`crates/buzz-media/src/thumbnail.rs:37`) |
| Public API must have doc comments | Satisfied for all public fns/types reviewed |


## Module: buzz-workflow (`crates/buzz-workflow`)

### Aspect: Conventions

---

### 1. Module organization

| File | LOC | Role |
|---|---|---|
| `src/lib.rs` | 1564 | crate root: `WorkflowConfig`, `WorkflowEngine`, event hook, scheduler loop, trigger matching, `build_trigger_context` |
| `src/schema.rs` | 878 | definition types + validation + YAML parsing |
| `src/executor.rs` | 1834 | templates, conditions, action dispatch, HTTP impls, step loop |
| `src/error.rs` | 66 | `WorkflowError`, `PartialProgress` |
| `src/action_sink.rs` | 69 | `ActionSink` trait + `ActionSinkError` |

Flat module layout — four sibling modules, all declared `pub` at `lib.rs:33-36`, with the commonly used types re-exported at the crate root (`lib.rs:38-41`) so callers write `buzz_workflow::WorkflowDef` rather than `buzz_workflow::schema::WorkflowDef`.

Crate-level lints: `#![deny(unsafe_code)]` and `#![warn(missing_docs)]` (`lib.rs:1-2`); every public item carries a doc comment, and `///` docs are also used on private helpers and enum fields.

---

### 2. Naming

| Convention | Examples |
|---|---|
| Types `UpperCamelCase`, suffix conveys role | `WorkflowDef`, `TriggerDef`, `ActionDef`, `TriggerContext`, `ExecutionResult`, `StepResult`, `PartialProgress`, `WorkflowError`, `ActionSinkError` |
| Serde variant naming | `rename_all = "snake_case"` on both tagged enums so Rust `SendChannelTopic`-style variants map to YAML `set_channel_topic` (`schema.rs:37`, `schema.rs:91`) |
| Keyword collisions | Rust field `if_expr` renamed to YAML `if` (`schema.rs:79-80`) |
| Verb-first functions | `parse_yaml`, `validate_cron`, `normalize_cron`, `resolve_template`, `resolve_variable`, `resolve_step_templates`, `build_eval_context`, `build_trigger_context`, `evaluate_condition`, `dispatch_action`, `execute_run`, `execute_steps`, `execute_from_step`, `finalize_run` |
| Predicate functions prefixed `is_`/`should_`/`_matches_` | `interval_should_fire`, `interval_prefilter_should_fire`, `should_fire_workflow`, `trigger_matches_event` |
| `_impl` suffix for feature-gated internals | `call_webhook_impl`, `add_reaction_impl` (`executor.rs:781`, `executor.rs:888`) |
| Constants `SCREAMING_SNAKE_CASE`, unit in the name | `EVAL_TIMEOUT`, `MAX_EXPR_LEN`, `MAX_DELAY_SECS`, `WEBHOOK_MAX_RESPONSE_BYTES` |
| Deliberately unused params prefixed `_` | `generate_approval_token(_run_id, _step_id)` (`executor.rs:698`) |
| evalexpr variable mangling | dots → underscores: `trigger_text`, `steps_{id}_output_{field}` (documented as a table in the fn doc, `executor.rs:207-217`) |

---

### 3. Error handling

Single crate error enum `WorkflowError` (`error.rs:18-60`), `thiserror`-derived, all variants documented:

| Variant | Payload | `#[error]` message | Line |
|---|---|---|---|
| `InvalidYaml` | `#[from] serde_yaml::Error` | `invalid YAML: {0}` | `error.rs:20-22` |
| `InvalidDefinition` | `String` | `invalid definition: {0}` | `error.rs:24-26` |
| `ConditionError` | `String` | `condition evaluation error: {0}` | `error.rs:28-30` |
| `TemplateError` | `String` | `template error: {0}` | `error.rs:32-34` |
| `StepTimeout` | `{ step_id: String, timeout_secs: u64 }` | `step '{step_id}' timed out after {timeout_secs}s` | `error.rs:36-43` |
| `WebhookError` | `String` | `webhook error: {0}` | `error.rs:45-47` |
| `CapacityExceeded` | — | `capacity exceeded` | `error.rs:49-51` |
| `Database` | `String` | `database error: {0}` | `error.rs:53-55` |
| `NotImplemented` | `String` | `action not implemented: {0}` | `error.rs:57-59` |

Companion `ActionSinkError` (`action_sink.rs:12-32`) has 6 variants (`InvalidInput`, `ChannelNotFound`, `ChannelArchived`, `EventBuild`, `Database`, `EmptyContent`) and collapses into `WorkflowError::WebhookError` (`action_sink.rs:34-38`).

Patterns:
- `?` with `map_err` closures everywhere; no `unwrap()`/`expect()` in production paths except `LazyLock` client construction `expect("HTTP client build must succeed")` (`executor.rs:882`) and `parts.next().unwrap_or("")` style safe fallbacks (`executor.rs:99`).
- Fallible operations that must not abort a batch use "log-and-continue": `tracing::warn!`/`error!` then `continue` (`lib.rs:333-336`, `lib.rs:466-469`, `lib.rs:604-610`).
- Partial results are first-class: `Result<ExecutionResult, (WorkflowError, PartialProgress)>` is the executor's return type so trace data survives failure (`executor.rs:975`, `executor.rs:1088`).
- One deliberate panic: double `set_action_sink` (`lib.rs:139-143`), documented with `# Panics`.
- `WorkflowError::WebhookError` is overloaded — it also carries `send_message` DB-lookup failures (`executor.rs:539-541`, `executor.rs:548-551`) and SSRF/DNS failures (`executor.rs:757-763`).

---

### 4. Async patterns

| Pattern | Usage |
|---|---|
| Detached background execution | `tokio::spawn` for each triggered run, both event and cron paths (`lib.rs:371-381`, `lib.rs:649-661`) |
| Non-blocking admission | `Semaphore::try_acquire()` (no `acquire().await`), permit held in a `_permit` binding for the run's lifetime (`executor.rs:978`, `executor.rs:1028`) |
| CPU/blocking isolation | `spawn_blocking` for evalexpr evaluation (`executor.rs:372`) and for the blocking DNS resolver (`executor.rs:747-755`) |
| Timeouts | `tokio::time::timeout` around expression evaluation (`executor.rs:370`) and around each `dispatch_action` (`executor.rs:1139-1151`) |
| Sleep-based loop | `loop { sleep(60s).await; … }` scheduler with sleep-first ordering (`lib.rs:430-432`) |
| `Arc<Self>` receivers | `on_event(self: &Arc<Self>)` and `run(self: &Arc<Self>)` so spawned tasks can clone the engine without `'static` on `&self` (`lib.rs:276-279`, `lib.rs:428`) |
| Late init without `Mutex` | `OnceLock<Arc<dyn ActionSink>>` (`lib.rs:90`) |
| dyn-compatible async trait | manual `Pin<Box<dyn Future … + Send + '_>>` return instead of `async_trait` (`action_sink.rs:60-70`) |
| Lock-free shared state | `DashMap` for interval anchors, `moka::sync::Cache` for workflow lookups (`lib.rs:87`, `lib.rs:104`) |
| Sync-in-async caution | `evaluate_condition` is `async` purely to host the timeout; `resolve_template`/`build_eval_context` stay synchronous |

---

### 5. Testing patterns

All tests are inline `#[cfg(test)] mod tests` blocks: `schema.rs:270`, `lib.rs:966`, `executor.rs:1219`. No `tests/` directory, no fixtures directory, no mocking crate.

| File | `#[test]` | `#[tokio::test]` | Total |
|---|---|---|---|
| `schema.rs` | 50 | 0 | 50 |
| `lib.rs` | 38 | 0 | 38 |
| `executor.rs` | 39 | 22 | 61 |
| `error.rs` | 0 | 0 | 0 |
| `action_sink.rs` | 0 | 0 | 0 |
| **Total** | **127** | **22** | **149** |

Conventions observed:
- YAML fixtures are inline `&str` literals built with `concat!` or `\n`-escaped strings, with a comment explaining why raw strings are avoided (`schema.rs:275-276`, `schema.rs:326-328`).
- Error assertions use `matches!(err, WorkflowError::Variant(_))` plus substring checks on the message (`schema.rs:404-407`, `schema.rs:428-436`).
- Deterministic time: fixed RFC-3339 instants parsed for cron/interval tests (`lib.rs:969-985`, `lib.rs:1140-1166`) alongside `Utc::now()`-relative tests for elapsed-interval logic (`lib.rs:1168-1235`).
- Pure-function extraction for testability: `interval_prefilter_should_fire` is a free function over `&DashMap` explicitly "so it is unit-testable without a `Db`/Postgres" (`lib.rs:777-782`).
- Shared builders instead of a framework: `make_trigger()` (`executor.rs:1223-1233`), `make_message_event()` (`lib.rs:1338-1347`), `make_reaction_event()` (`lib.rs:1350-1371`).
- Regression tests carry intent comments naming the bug they lock down (`lib.rs:1211-1216`, `executor.rs:1812-1814`, `schema.rs:756-758`).
- Nothing that requires Postgres or an `ActionSink` is unit-tested — no test constructs a `WorkflowEngine`, so `on_event`, `run`, `execute_run`, `execute_from_step`, `execute_steps`, `dispatch_action`, and `finalize_run` have zero unit coverage in this crate.

---

### 6. Documentation conventions

- Module-level `//!` headers with a Responsibilities list (`executor.rs:1-11`) and an Architecture/Usage section including a `rust,ignore` example (`lib.rs:3-31`).
- Markdown tables inside doc comments to specify mappings (`executor.rs:207-217`).
- Long rationale comments attached to consistency-critical fields — the workflow cache's cross-pod invalidation trade-off (`lib.rs:92-103`) and the interval cold-start liveness argument (`lib.rs:385-399`).
- Ticket-tagged deferrals: `WF-07`, `WF-08`, `WF-09` (`executor.rs:582`, `:588`, `:663`, `:675`; `lib.rs:192`).
- Numbered "Fix N" comments preserving review history (`schema.rs:218`, `lib.rs:465`, `lib.rs:572`, `lib.rs:638`, `lib.rs:664`).


## Module: buzz-relay — core & bootstrap (`crates/buzz-relay/src`)
### Aspect: Conventions

---

#### 1. Crate-level lint policy

| Attribute | Line | Effect |
|-----------|------|--------|
| `#![deny(unsafe_code)]` | `lib.rs:1` | zero `unsafe` blocks in the crate; verified — the only textual `unsafe` in this group is a comment (`main.rs:1013`) |
| `#![warn(missing_docs)]` | `lib.rs:2` | warn, not deny — an undocumented public item compiles |
| crate `//!` doc | `lib.rs:3` | placed after the inner attributes (valid) |

Local `#[allow]` count in this group: **6**, all narrowly scoped, no blanket allows.

| Allow | Line | Justification present? |
|-------|------|------------------------|
| `clippy::too_many_arguments` on `ConnectionManager::register` | `state.rs:204` | yes — `state.rs:202-203` explains a params struct would only relocate the fields |
| `clippy::type_complexity` on `membership_cache` | `state.rs:544` | no |
| `clippy::type_complexity` on `accessible_channels_cache` | `state.rs:548` | no |
| `clippy::type_complexity` on `observer_owner_cache` | `state.rs:606` | no |
| `clippy::too_many_arguments` on `AppState::new` (10 params) | `state.rs:636` | no |
| `clippy::type_complexity` on the NIP-11 fence const | `nip11.rs:328` | implicit (the whole const is a deliberate signature pin) |

#### 2. Module layout convention

`lib.rs` declares 21 modules. Every `pub mod` carries a one-line `///` doc **except** `storage_sweep` (`lib.rs:43`) — the only inconsistency. `storage_sweep.rs` does have `//!` inner docs (`storage_sweep.rs:1-5`) so `missing_docs` is satisfied, but the `lib.rs` listing reads as an oversight.

Exactly **one** private module: `mod admission;` (`lib.rs:5`). It is also the only file in the group with **no module-level doc comment at all** — `admission.rs:1` is a `use` statement. Every other file starts with `//!`:

| File | Module doc | Line |
|------|-----------|------|
| `state.rs` | "Shared application state — Arc-wrapped, shared across all connections." | `:1` |
| `config.rs` | "Relay configuration from environment variables." | `:1` |
| `router.rs` | "axum routers — app (WebSocket + REST), health (K8s probes), metrics (Prometheus)." | `:1` |
| `nip11.rs` | "NIP-11 relay information document." | `:1` |
| `protocol.rs` | "NIP-01 client/relay message parsing and formatting." | `:1` |
| `tenant.rs` | 15-line conformance narrative ("row zero") | `:1-15` |
| `telemetry.rs` | ASCII architecture diagram + honoured env vars | `:1-32` |
| `metrics.rs` | ASCII architecture diagram | `:1-19` |
| `error.rs` | "Error types for the relay crate." | `:1` |
| `admission.rs` | **none** | — |

**ASCII-diagram convention**: three files open with a boxed ASCII diagram of their data flow — `metrics.rs:3-12`, `telemetry.rs:3-15`, and the `serve` fn doc (`main.rs:1097-1112`). This is a distinctive house style in the relay core.

#### 3. Error-handling convention — three coexisting styles

| Layer | Style | Cite |
|-------|-------|------|
| `main()` | `anyhow::Result` with `.map_err(|e| anyhow!("…: {e}"))` and an `error!` log immediately before returning | `main.rs:83`, `main.rs:122-127`, `main.rs:151-155` |
| `config.rs` | `thiserror` enum `ConfigError` with 2 variants | `config.rs:19-27` |
| `protocol.rs` | `thiserror` enum `RelayError` + `type Result<T>` alias | `error.rs:8-49`, used only at `protocol.rs:6` |
| handlers (outside group) | `axum::response::Response` / `IntoResponse` tuples, no shared error type | `router.rs:295-301`, `router.rs:304-375` |

`RelayError` is the crate's nominal error type but is effectively single-purpose: only `InvalidMessage` is constructed, only `protocol.rs` imports it (see debt). There is no `impl IntoResponse for RelayError`, so HTTP handlers cannot use it — that is why the third style exists.

Fallibility conventions actually followed:
- Fatal-at-boot: `.map_err(anyhow!)?` (11 sites in `main.rs`).
- Non-fatal-at-boot: `match … { Err(e) => error!(…) }` with an explicit "(non-fatal…)" phrase in the message — a consistent, greppable convention (`main.rs:174`, `:190-195`, `:253`, `:286`, `:305`, `:319`).
- Conditional fatality is always written as `if config.require_relay_membership { … return Err(…) } else { error!(…) }` (`main.rs:250-262`, `:275-288`, `:295-309`).

#### 4. `unwrap`/`expect`/panic conventions

**26 production `unwrap()`/`expect()` calls** in this file group (outside `#[cfg(test)]`), plus 1 `panic!`, 1 `unreachable!`, 1 `debug_assert!`, 1 `std::process::exit`.

| File | Count | Lines |
|------|-------|-------|
| `metrics.rs` | 17 | `:79, 84, 89, 94, 99, 104, 109, 114, 119, 124, 129, 134, 136, 141, 143, 145` (`expect`) + `:180` (`unwrap`) |
| `main.rs` | 4 | `:90, 401, 1230, 1253` |
| `state.rs` | 3 | `:446, 701, 708` |
| `config.rs` | 1 | `:507` |
| `protocol.rs` | 1 | `:189` |
| `router.rs`, `nip11.rs`, `tenant.rs`, `telemetry.rs`, `admission.rs`, `lib.rs`, `error.rs` | 0 | — |

Convention: `expect` messages are written as **assertions about an invariant**, not as error text — `"hardcoded dev key is valid"` (`main.rs:401`), `"relay fan-out frames are serialized UTF-8 JSON"` (`state.rs:446`), `"media storage was already constructed with this S3 config"` (`state.rs:701`), `"metrics exporter must build exactly once"` (`metrics.rs:143`), `"SAFETY: nostr::Event serialization is infallible for well-formed events"` (`protocol.rs:189`). Two carry an inline `// safe:` comment (`metrics.rs:180`, `config.rs:507`).

This contradicts AGENTS.md's "Do not introduce new `unwrap()` or `expect()` in production paths". The 17 in `metrics.rs` are all boot-time bucket registration where the argument is a compile-time literal array, so they are provably-infallible; still, they are production `expect`s.

Other panic-shaped exits:
- `panic!` at `main.rs:409` when `BUZZ_REQUIRE_AUTH_TOKEN=true` and no relay key — a deliberate hard stop, but inconsistent with the surrounding `return Err(anyhow!(…))` style used for the two neighbouring preconditions (`main.rs:206-211`, `:216-219`).
- `unreachable!("mesh handle is set exactly once, right here")` at `main.rs:460`.
- `debug_assert!` at `nip11.rs:143` — release builds skip the NIP-43/`self` consistency check.
- `std::process::exit(1)` at `main.rs:1153` — the only non-`main`-return process exit.

#### 5. Logging / tracing conventions

| Convention | Cite |
|-----------|------|
| JSON-only structured logs, `flatten_event(true)` | `main.rs:109` |
| Filter is `RUST_LOG` **plus** a forced `buzz_relay=info` directive | `main.rs:111` |
| OTLP layer attached only when the endpoint env var is set | `main.rs:101-107` |
| Structured fields preferred over interpolation: `info!(bind_addr = %…, relay_url = %…, …, "Config loaded")` | `main.rs:128-136` |
| `%` for `Display`, `?` for `Debug` — used consistently (`error = %e`, `error = ?e`) | `main.rs:119`, `main.rs:558`, `state.rs:886` |
| Sentence-case, no trailing period, human-readable message last | `main.rs:157`, `main.rs:197`, `state.rs:1185` |
| Startup progress logged as a linear narrative ("Postgres connected", "Redis pub/sub connected", "Media storage connected", "…listener started") | `main.rs:157-159`, `:348`, `:421`, `:1117`, `:1155` |
| Two logging import styles coexist: `use tracing::{error, info, warn}` at `main.rs:5` **and** fully-qualified `tracing::warn!`/`tracing::info!`/`tracing::error!` at `main.rs:404`, `:483`, `:513`, `:837`, `:1152` and throughout `state.rs` | mixed |

`state.rs` never imports the tracing macros; it always fully-qualifies (`state.rs:463`, `:472`, `:886`, `:1185`, …). `main.rs` does both. No lint enforces either.

#### 6. Test conventions

**Test counts in this file group** (`#[test]` + `#[tokio::test]`):

| File | Tests | `#[ignore]` | Notes |
|------|-------|-------------|-------|
| `config.rs` | 22 | 0 | serialized by a module-level `ENV_MUTEX` (`config.rs:931-934`) with an explicit rationale about flakiness |
| `state.rs` | 18 | 0 | 5 need a live Postgres/Redis via `test_state()` (`state.rs:1257-1290`) |
| `nip11.rs` | 15 | 0 | deliberately test the `SUPPORTED_NIPS` **constant** rather than `Config::from_env()` to avoid the env race (`nip11.rs:358-360`) |
| `tenant.rs` | 10 | 0 | includes a `redteam_attack2` sub-module (`tenant.rs:249-332`) |
| `protocol.rs` | 7 | 0 | table-driven with `Box<dyn Fn>` case tables and `type ParseCase`/`FormatCase` aliases to dodge `clippy::type_complexity` (`protocol.rs:230-232`, `:378-380`) |
| `main.rs` | 7 | 0 | includes `#[tokio::test(start_paused = true)]` for the cancellation loop (`main.rs:1827`) |
| `telemetry.rs` | 5 | 0 | `ENV_LOCK` mutex (`telemetry.rs:119`) |
| `router.rs` | 4 | 0 | one spins a real TCP + tungstenite client (`router.rs:463-501`) |
| `admission.rs` | 4 | 0 | stub `RateLimiter` impl (`admission.rs:64-95`) |
| `metrics.rs` | 0 | 0 | **no tests at all** |
| `lib.rs`, `error.rs` | 0 | 0 | no logic |
| **Total** | **92** | **0** | |

Conventions observed:
- **Env-mutation serialization**: two independent static mutexes, `config.rs:934` (`ENV_MUTEX`) and `telemetry.rs:119` (`ENV_LOCK`), both with a comment explaining process-global env races. `nip11.rs` avoids the problem structurally instead.
- **Save/restore around env mutation**: `let previous = std::env::var_os(…); … if let Some(v) = previous { set_var } else { remove_var }` — used at `config.rs:990-1000`, `:1015-1030`, `:1043-1055`, `:1058-1070`, `:1076-1086`, `:1192-1210`. Not used in `config.rs:1092-1105` or `:1109-1113` (rate-limit tests just `remove_var` afterwards) — inconsistent.
- **Red-team gate tests**: `tenant.rs:249-332` names its module `redteam_attack2`, documents the RED-then-GREEN cycle, and includes a *negative control* (`tenant.rs:326-331`). Notable convention.
- **Named-case tables**: `protocol.rs:234-273` and `:382-448` use `&[(name, Box<dyn Fn>)]` slices.
- **Behaviour-named test fns**: e.g. `disconnect_pubkey_is_fenced_to_the_banning_community` (`state.rs:1737`), `register_after_drain_self_signals_restart_close_and_cancel` (`state.rs:1881`), `relay_websocket_parser_rejects_oversized_messages_before_handler_reads_them` (`router.rs:504`). Long, assertion-shaped names are the norm.
- **Assertion messages**: nearly every `assert!`/`assert_eq!` carries an explanatory third argument (`state.rs:1454`, `:1770`, `:1795`, `nip11.rs:432`, `tenant.rs:320`).
- **`#[should_panic(expected = …)]`** used once, to pin the `debug_assert` (`nip11.rs:513-517`).
- **Zero `#[ignore]`d tests** in the group. However `tenant.rs:225-236` and `:238-243` still instruct the reader to "Delete this `#[ignore]` when the fix lands" — the attributes were already removed and the fix landed (`tenant.rs:81-88`). Stale instructions (see debt).
- Integration-style tests that need infrastructure are inlined into `#[cfg(test)]` modules rather than `tests/`: `crates/buzz-relay/tests/` **does not exist**.

#### 7. Naming conventions

| Pattern | Examples |
|---------|----------|
| Env vars: `BUZZ_` prefix, except deliberate exceptions | `RELAY_URL`, `RELAY_OWNER_PUBKEY`, `RELAY_OPERATOR_PUBKEYS`, `RELAY_OPERATOR_API_ORIGIN` — rationale given at `config.rs:524-525` and `:548-551` ("relay-identity config that may be shared across multiple services"); plus `DATABASE_URL`/`READ_DATABASE_URL`/`REDIS_URL`/`AWS_REGION`/`RUST_LOG`/`OTEL_*` |
| Env vars: **`SPROUT_` legacy prefix survives in 3 vars** | `SPROUT_REMINDER_SCHEDULER_INTERVAL_SECS` (`main.rs:701`), `SPROUT_REMINDER_SCHEDULER_BATCH_LIMIT` (`main.rs:705`), `SPROUT_MAX_NOT_BEFORE_DELTA` (`nip11.rs:97`) — the repo has otherwise renamed to `buzz`/`BUZZ_` |
| Metrics: `buzz_` prefix, `_total` counters, `_seconds` histograms | `state.rs:466`, `:1201`, `:1205`, `main.rs:836` |
| Framework metrics keep the CAKE names without the `buzz_` prefix | `http_requests_total`, `http_request_latency_ms` (`metrics.rs:204-205`) |
| Cross-pod method pairs: public `X` publishes, `pub(crate) X_local` applies a received drop (so a received drop is never re-published) | `state.rs:850/862`, `:876/881`, `:899/906`, `:921/928` |
| Cache accessors: `*_cached` | `is_member_cached` (`state.rs:827`), `get_accessible_channel_ids_cached` (`state.rs:1089`), `channel_visibility_cached` (`state.rs:1124`) |
| Cluster-wide operations: `*_clusterwide` | `disconnect_pubkey_clusterwide` (`state.rs:1018`), `disconnect_community_clusterwide` (`state.rs:1056`) |
| Health/internal routes prefixed `_` | `/_liveness`, `/_readiness`, `/_status`, `/_mesh`, `/_mesh/demo/echo` — and `metrics.rs:170` skips `/_*` on that basis |

#### 8. Doc-comment conventions

- Long "why" narratives on invariant-bearing items are the norm, not the exception: `state.rs:296-308` (tenant fence on ban), `state.rs:336-350` (drain rationale), `state.rs:530-540` (community-keyed dedup), `state.rs:1106-1122` (fail-safe visibility caching), `config.rs:114-128` (huddle single-pod constraint), `nip11.rs:307-327` (static-input fence), `tenant.rs:1-15` (row zero), `main.rs:36-48` (metric-cardinality cost lever).
- Spec/plan cross-references are embedded in doc comments: `COMMUNITY_MODERATION_PLAN.md §0 decision 4` (`state.rs:301`), `docs/spec/MultiTenantRelay.tla` (`state.rs:616-619`, `lib.rs:14-18`), `docs/git-on-object-storage.md` (`state.rs:563`), `PLANS/S3_STORAGE_METRICS_PLAN.md` (`storage_sweep.rs:4`), `E1 §4.8` (`state.rs:1116`, `state.rs:1155`), `spec §Push step 7, Inv_NoFork` (`state.rs:519-520`), `plan §4 fork B` / `§5b` (`config.rs:118-122`).
- Invariant names are used as identifiers in prose: `Inv_RowZero` (`tenant.rs:70`, `:251`), `Inv_LabelPropagation` (`state.rs:1159`), `Inv_NoFork` (`state.rs:520`), `A3` (`main.rs:467`), `C4`/`K1` (`main.rs:1486`, `:1506`).
- **Compile-time doc enforcement**: `nip11.rs:329-335` turns a documented conformance obligation into a type-level fence. This is the only instance of the pattern in the group and is explicitly described as "the same way a deny-lint would" (`nip11.rs:322-324`).

#### 9. Config-parsing conventions (inconsistent — see business-rules BR-RC-109)

Three helper functions exist (`positive_u64_from_env` `config.rs:270`, `parse_bool` `config.rs:363`, `parse_optional_bool` `config.rs:379`) but most fields bypass them with an inline `std::env::var(...).ok().and_then(|v| v.parse().ok()).unwrap_or(default)` chain. Result: **eight distinct boolean grammars** and two distinct numeric-invalid policies (silent default vs hard error) inside one file. `parse_optional_bool` (`config.rs:379-381`) is a one-line wrapper that just calls `parse_bool(name, false)` and has a single caller (`config.rs:792`).

Numeric-invalid policy split:
- Hard error: `BUZZ_RATE_LIMIT_*` (`config.rs:270-282`), `BUZZ_PUSH_GATEWAY_TIMEOUT_MS` (`config.rs:759-773`).
- Silent default: `BUZZ_REDIS_POOL_SIZE`, `BUZZ_MAX_CONNECTIONS`, `BUZZ_MAX_CONCURRENT_HANDLERS`, `BUZZ_SEND_BUFFER`, `BUZZ_MAX_FRAME_BYTES`, `BUZZ_SLOW_CLIENT_GRACE_LIMIT`, `BUZZ_HEALTH_PORT`, `BUZZ_METRICS_PORT`, all `BUZZ_GIT_*` numerics, all `BUZZ_MAX_*_BYTES`, all `BUZZ_MEDIA_*` numerics, and every interval var read directly in `main.rs`. `redis_pool_size` has an explicit test asserting the silent-fallback behaviour (`config.rs:988-1011`).


## Module: buzz-relay — WebSocket protocol & subscriptions (`crates/buzz-relay/src`)
### Aspect: Conventions

---

#### 1. Handler signatures

Every WS message handler follows one shape — `(payload…, Arc<ConnectionState>, Arc<AppState>)`, returning `()`, replying through `conn.send`:

| Handler | Signature | Site |
|---|---|---|
| AUTH | `async fn handle_auth(event: nostr::Event, conn: Arc<ConnectionState>, state: Arc<AppState>)` | `auth.rs:44-49` |
| EVENT | `async fn handle_event(event: Event, conn: Arc<ConnectionState>, state: Arc<AppState>)` | `event.rs:608` |
| REQ | `async fn handle_req(sub_id: String, filters: Vec<Filter>, conn: Arc<ConnectionState>, state: Arc<AppState>)` | `req.rs:44-49` |
| COUNT | `async fn handle_count(sub_id: String, filters: Vec<Filter>, conn: Arc<ConnectionState>, state: Arc<AppState>)` | `count.rs:30-35` |
| CLOSE | `async fn handle_close(sub_id: String, conn: Arc<ConnectionState>, state: Arc<AppState>)` | `close.rs:12` |

Conventions that hold across all five:

1. **No `Result`.** Errors are terminal side effects (a `conn.send` + `return`), never propagated. There is not one `?` in a handler's top-level body.
2. **`Arc` by value, not `&`.** Uniform even for the two handlers that are awaited inline (`auth`, `close`) and never need `'static` — so a handler can be moved into a spawn without a signature change.
3. **Auth is extracted into locals inside a scoped block**, so the `RwLock` guard drops before any `.await` on I/O. `event.rs:634-654`, `req.rs:50-87`, `count.rs:37-51`, `auth.rs:45-74` all use the same `let (…) = { let auth = conn.auth_state.read().await; match &*auth { … } };` shape.
4. **Internal helper naming**: `handle_*` for message entry points, `handle_*_event` for sub-branches (`handle_ephemeral_event` `event.rs:762`, `handle_agent_observer_event` `event.rs:943`, `handle_search_req` `req.rs:504`), `*_authorized` for boolean filter gates, `filter_can_match_*` for capability predicates, `extract_*` for tag/field readers.
5. **Tracing**: `#[tracing::instrument(skip_all, fields(...))]` with `tracing::field::Empty` placeholders recorded once the values exist (`event.rs:607`, `:591-594`; `auth.rs:45`, `:77-80`). The dispatcher captures the span *before* `tokio::spawn` and attaches it with `.instrument(span)`, with an explicit comment on why (`connection.rs:522-523`).
6. **`#[allow(clippy::too_many_arguments)]`** is used rather than introducing param structs, with a stated rationale where non-obvious (`req.rs:504`, `state.rs:201-204`).

Free-function helper conventions:
- `pub(crate)` for cross-module helpers inside the crate (`req.rs:448`, `:483`, `:993`, `:1042`, `:1099`, `:1137`, `:1154`, `:1172`, `:1186`, `:1204`; `event.rs:35`, `:218`, `:326`).
- `pub` only where the HTTP bridge needs it (`req.rs:733` `build_event_query_from_filter`, `req.rs:773` `filter_fully_pushable`, `event.rs:115` `filter_fanout_by_access`, `event.rs:282` `fan_out_pubsub_event`).
- Private for file-local (`req.rs:829`, `:842`, `:860`, `:1013`, `:1225`; `event.rs:55`, `:59`, `:63`, `:76`, `:894`, `:1071`, `:1111`, `:1117`).

---

#### 2. Error-to-wire mapping

The rule is **message-kind determines frame kind**:

| Inbound | Rejection frame | Rationale / site |
|---|---|---|
| `EVENT` | `["OK", <event_id>, false, "<prefix>: <detail>"]` | NIP-20 acknowledgement — `event.rs:669`, `:639`, `:649`, `:660`, `:678`, `:733` |
| `REQ` | `["CLOSED", <sub_id>, "<prefix>: <detail>"]` | `req.rs:56`, `:67`, `:80`, `:101`, `:167`, `:185`, `:192`, `:199`, `:214` |
| `COUNT` | `["CLOSED", <sub_id>, …]` | `count.rs:44`, `:57`, `:64`, `:71`, `:86`, `:179` |
| parse failure (no sub_id yet) | `["NOTICE", "invalid message: …"]` | `connection.rs:493` |
| pre-dispatch throttle | `CLOSED` if a sub_id exists, else `NOTICE` | `request_rejection_message`, `connection.rs:587-592` |

Reason strings use the NIP-01 machine-readable prefix set, consistently:

| Prefix | Meaning as used here | Examples |
|---|---|---|
| `auth-required:` | not authenticated, or an auth-state error | `event.rs:649`, `req.rs:78`/`:82`, `count.rs:47`, `auth.rs:54`/`:63`/`:210`/`:291` |
| `restricted:` | authenticated but not permitted | `event.rs:686`, `:681`, `:1027`; `req.rs:170`, `:187`, `:194`, `:201`; `auth.rs:234` |
| `invalid:` | malformed / semantically rejected | `event.rs:665`, `:652`, `:757`, `:956`, `:1073`, `:1096` |
| `blocked:` | moderation ban | `auth.rs:160` |
| `rate-limited:` | throttled | `connection.rs:517`, `:546`, `:567`, `:666`, `:674`; `event.rs:1063` |
| `error:` | server-side fault or a protocol-level cap | `event.rs:788`, `:1012`; `req.rs:69`, `:103`, `:216`; `count.rs:86`, `:179` |

Two deliberate escalations beyond the frame:
- **Ban** — frame queued on `ctrl_tx`, then `cancel()` so the socket closes immediately (`auth.rs:173-182`). The "queue on ctrl, then cancel" idiom is named as a reusable convention at `connection.rs:328-336` and pinned by `connection.rs:856-882`.
- **Oversized frame** — `NOTICE` then `break` out of `recv_loop` (`connection.rs:428-433`, `:447-452`).

##### Sanitisation convention (not uniformly applied)

`handle_event` sanitises `IngestError::Internal` to `error: internal server error` with an explicit comment (`event.rs:749-754`). `handle_count` does the opposite: four sites forward the raw error with `format!("error: {e}")` (`count.rs:179`, `:209`, `:249`, `:278`). The `req.rs` historical path forwards nothing (it emits a bare `EOSE`, `:327`). Three different postures for the same class of failure.

##### Metrics-on-rejection convention

`event.rs` funnels every rejection through `reject(reason)` (`event.rs:30-32`) → `reject_with_transport("ws", reason)` (`ingest.rs:156`) with a **bounded** reason label — the four values used are `"auth"`, `"invalid"`, `"scope"`, `"error"` (`event.rs:645`, `:638`, `:648`, `:659`, `:677`, `:732`, `:969`, `:1022`). `req.rs`, `count.rs`, and `close.rs` do **not** emit a rejection counter at all, so REQ/COUNT denials are invisible in metrics.

---

#### 3. Locking conventions

| Convention | Evidence |
|---|---|
| `auth_state` is a `RwLock` because it is read-heavy after auth; `subscriptions` is a `Mutex` because it is write-heavy during REQ/CLOSE. Stated in the struct doc. | `connection.rs:50-52` |
| Guards are scoped so they drop before I/O `await`s. | `event.rs:634-654`, `req.rs:50-87`, `count.rs:37-51` |
| Only one nested acquisition exists, always in the same order: `auth_state` (read) → `subscriptions` (lock). No reverse ordering exists, so no deadlock. | `req.rs:51` → `:65` |
| DashMap guards are explicitly `drop`ped before a `remove` on the same map, to avoid self-deadlock. | `subscription.rs:408-410`, `:430-432`, `:447-449`, `:456-457`, `:465-466` |
| `authenticated_pubkey` uses `std::sync::RwLock` (not tokio) because reads are non-async and lock-poisoning is handled with `.ok()?`. | `state.rs:56`, `:246-256`, `:286-290` |

---

#### 4. Concurrency conventions

| Convention | Evidence |
|---|---|
| `try_send` / `try_acquire_owned` everywhere on the hot path — never `send().await` or `acquire().await`, so the recv loop cannot be blocked by a slow peer or a saturated handler pool. | `connection.rs:89`, `:149`, `:513`, `:541`, `:563`; `state.rs:453` |
| One documented exception: the audit enqueue uses `send().await` **on purpose**, with a written rationale. | `event.rs:574` (rationale `:551-557`) |
| Semaphore permits are `Owned` and dropped explicitly at the end of the spawned body. | `connection.rs:533`, `:555`, `:576` |
| CPU-bound signature verification always goes to `spawn_blocking`. | `event.rs:772`, `:927`; `ingest.rs:1488` |
| Long delivery loops yield cooperatively every 100 items. | `req.rs:401-404` |
| Ordered concurrency uses `buffered`, never `buffer_unordered`, when downstream ordering is semantically load-bearing — with the reason spelled out. | `req.rs:318` (doc `:299-303`, and the constant's own doc `:28-34`) |
| Concurrency bounds are compile-time asserted where a wrong value would be silently harmful. | `req.rs:37-41` (`const _: () = assert!(…)`) |
| `biased;` in `select!` where starvation would break a safety property. | `connection.rs:326-327` |

---

#### 5. Test conventions

Counts (all in-file `#[cfg(test)] mod tests`):

| File | Tests | `#[ignore]` | Module starts |
|---|---|---|---|
| `connection.rs` | 5 | 0 | `:689` |
| `subscription.rs` | 29 | 0 | `:574` |
| `handlers/event.rs` | 24 | 0 | `:1135` |
| `handlers/req.rs` | 45 | 0 | `:1233` |
| `handlers/auth.rs` | 3 | 0 | `:298` |
| `handlers/count.rs` | **0** | — | — |
| `handlers/close.rs` | **0** | — | — |
| `handlers/mod.rs` | **0** | — | — |
| **total** | **106** | **0** | |

Conventions observed:

1. **Infra-optional tests skip, not `#[ignore]`.** Instead of the `#[ignore = "requires Postgres"]` style used elsewhere in the crate (e.g. `api/operator.rs:706`), this group probes availability at runtime and returns early with an `eprintln!`: `event.rs:1676-1679` (Redis), `:1636-1639` and `:1735-1738` (Postgres+Redis via `audit_state()`), `:1642-1649` (`redis_url_if_available`). Net effect: `just test-unit` never fails on a missing dependency, but it also never reports that a test was skipped as a test outcome.
2. **Lazy-pool test state.** `fanout_access::test_state()` (`event.rs:2041-2043`) builds a full `AppState` with `PgPool::connect_lazy` and an intentionally dead Redis (`redis://127.0.0.1:1`, `event.rs:1997-2002`), so pure fan-out logic is testable with no infrastructure. Cache pre-seeding substitutes for DB reads (`event.rs:2123-2126`, `:2143-2152`).
3. **Fail-closed proved by omission.** `threaded_visibility_open_passes_through` (`event.rs:2298-2321`) relies on the lazy pool erroring a fresh lookup: pass-through therefore *proves* the threaded value was consulted. The reasoning is written into the test doc comment (`:2291-2294`).
4. **Test-only single-tenant wrappers** keep pre-multi-tenant tests readable: `register`, `remove_channel_subscriptions`, `channel_subscriber_conns`, `fan_out`, all `#[cfg(test)]` and delegating to the `*_scoped` form with `test_community()` = nil UUID (`subscription.rs:154-160`, `:228-233`, `:257-260`, `:333-336`, `:568-571`).
5. **Security regressions are named as such** and carry the invariant they pin in the doc comment: `test_global_sub_does_not_receive_channel_events` (`subscription.rs:996-1032`), `channel_less_event_must_drop_recipient_in_different_community` (`event.rs:2458-2481`), `local_echo_suppression_is_scoped_to_its_community` (`event.rs:1576-1617`).
6. **Red-team module convention.** `event.rs:2346-2483` is a `mod redteam` (declaration at `:2395`) whose ~50-line header cites the TLA+ spec (`docs/spec/MultiTenantRelay.tla`), the invariant, the mutation class, the exact code sites, the required structural fix, and the ownership routing. It also documents a self-deleting pattern ("MUST be deleted in the same change that fixes the leak") — though the companion "documents the current broken behavior" test it refers to at `:2383-2385` **is not present**, so the header is now partly stale (see the debt aspect).
7. **Byte-compatibility pinning** for anything performance-refactored: `fanout_event_frame_matches_legacy_format_byte_for_byte` (`event.rs:1177-1189`) and the `Arc`-must-not-escape-a-cycle assertion (`:1168-1188`).
8. **Truth-table tests** for boolean gates, one test per row: `resolve_request_local_access` gets all four rows (`req.rs:1299-1360`); `d_tag` pushdown gets five (`req.rs:1594-1648`); `p_gated_filters_authorized` for 44200 gets four numbered cases in one test with case comments (`req.rs:1490-1546`).
9. **Assertion messages carry the invariant**, not the mechanics: `"kinds:[] sub must NOT be in the wildcard index"` (`subscription.rs:962`), `"Inv_NonInterference: a connection bound to community A must not receive a community-B event"` (`event.rs:2475-2478`).
10. **Mock sink over a real socket** for send-loop tests: `MockSink` implements `Sink<WsMessage>` with a scripted `fail_after_flushes` so `send_loop_inner` terminates deterministically (`connection.rs:692-757`). This is why `send_loop` is a thin wrapper over the generic `send_loop_inner` (`connection.rs:296-306`).

Notable coverage gaps in convention terms: `count.rs` (281 LOC) and `close.rs` (35 LOC) have **no** in-file tests at all, and no test in this group drives `handle_req`, `handle_count`, or `handle_close` end-to-end — only their extracted helpers. `handle_agent_observer_event` is the single message handler with an end-to-end unit test (`event.rs:1318-1404`).

---

#### 6. Documentation conventions

| Convention | Evidence |
|---|---|
| Module-level `//!` stating the pipeline in one line | `connection.rs:1`, `subscription.rs:1`, `event.rs:1`, `req.rs:1`, `count.rs:1`, `auth.rs:1-13` |
| Every `pub` item has a doc comment (crate enforces `missing_docs`; the one opt-out is `#[allow(dead_code, missing_docs)] pub mod push_lease` at `handlers/mod.rs:23`) | throughout |
| Invariants are written as prose next to the code that enforces them, and cross-reference the plan/spec section | `event.rs:196-217`, `:100-114`; `req.rs:423-447`; `auth.rs:92-112` |
| Rejected alternatives are recorded inline rather than dropped | `event.rs:574-580` (why `send().await`), `req.rs:28-34` (why per-filter queries), `state.rs:1107-1116` (why only `private` is cached) |
| Numbered fences referencing an external design doc | `event.rs:184-190` ("Fence 3 (§4.8 phase-2)", "Fence 1"), `auth.rs:95-97` ("COMMUNITY_MODERATION_PLAN.md §0 decision 4", "MOD-7/M20") |
| Stale-comment risk is high because comments name line numbers and revisions | `event.rs:2368` ("this file, line 62" — `filter_fanout_by_access` is now at `:115`), `event.rs:2361` (cites `state.rs:30-44` for `ConnEntry`, which is now `:41-58`) |

---

#### 7. Metrics naming convention

`buzz_<subsystem>_<noun>_<unit>` with `_total` for counters, `_seconds` for duration histograms, bare noun for gauges:

- counters: `buzz_ws_connections_total`, `buzz_ws_auth_timeouts_total`, `buzz_ws_backpressure_disconnects_total`, `buzz_admission_rejections_total`, `buzz_auth_attempts_total`, `buzz_auth_failures_total`, `buzz_events_received_total`, `buzz_community_events_received_total`, `buzz_multinode_fanout_total`, `buzz_post_commit_dispatch_scheduled_total`, `buzz_post_commit_dispatch_errors_total`, `buzz_audit_send_errors_total`, `buzz_req_global_access_resolution_skips_total`, `buzz_count_fallback_rejections_total`
- gauges: `buzz_ws_connections_active`, `buzz_subscriptions_active`
- histograms: `buzz_event_processing_seconds`, `buzz_fanout_recipients`, `buzz_ws_send_batch_size`

Label cardinality is explicitly bounded: `bounded_kind_label` (`event.rs:35-53`) collapses unknown kinds to `"other"`, and kind × community is deliberately never crossed (rationale `event.rs:620-627`). Rejection reasons are `&'static str` by type (`event.rs:30`), so the label set cannot grow at runtime.

---

#### 8. Code-style facts (quality-gate relevant)

| Check | Result |
|---|---|
| `unsafe` blocks in the 8 files | **0** |
| `unwrap()` / `expect()` outside `#[cfg(test)]` | **1** — `event.rs:88` `.expect("fan-out frame cache covers every recipient subscription id")` |
| `TODO` / `FIXME` / `XXX` / `HACK` markers | **0** |
| `#[ignore]`d tests | **0** |
| `#[allow(dead_code)]` | 1, on the out-of-group `push_lease` module (`handlers/mod.rs:23`) |
| Files over 1000 lines | 3 of 8 — `event.rs` 2461, `req.rs` 1946, `subscription.rs` 1562 |
| Production-code share | `connection.rs` 688/893 (77%), `subscription.rs` 573/1562 (37%), `event.rs` 1134/2461 (46%), `req.rs` 1232/1946 (63%), `auth.rs` 297/350 (85%) |


## Module: buzz-relay — event ingest & side effects (`crates/buzz-relay/src/handlers`)
### Aspect: Conventions

---

### 1. Handler shape

Three distinct shapes coexist, distinguished by return type and error style.

**A. The ingest pipeline** — one linear function, no sub-handlers.

```rust
async fn ingest_event_inner(
    state: &Arc<AppState>,
    tracer: &Arc<dyn buzz_conformance::Tracer>,
    tenant: &TenantContext,
    event: Event,
    auth: IngestAuth,
) -> Result<IngestResult, IngestError>          // ingest.rs:1453
```

Argument order is stable across the group: `tenant` first (or after `state`), then
`state`, then `event`, then auth. Two orderings are actually in use —
`(state, tenant, …)` in `ingest.rs` and `(tenant, state, …)` in `side_effects.rs` and
`command_executor.rs`. No file mixes them internally.

**B. Side-effect handlers** — `anyhow::Result<()>`, private, one per kind.

```rust
async fn handle_put_user(
    tenant: &TenantContext,
    event: &Event,
    state: &Arc<AppState>,
) -> anyhow::Result<()>                          // side_effects.rs:1203
```

All 13 follow this exactly: `handle_kind0_profile` `:1113`, `handle_agent_profile` `:1078`,
`handle_put_user` `:1203`, `handle_remove_user` `:1265`, `handle_edit_metadata` `:1335`,
`handle_delete_event_side_effect` `:1560`, `handle_create_group` `:1660`,
`handle_delete_group` `:1783`, `handle_join_request` `:1835`, `handle_leave_request` `:1913`,
`handle_a_tag_deletion` `:1979`, `handle_standard_deletion_event` `:2108`,
`handle_git_repo_announcement` `:2412`.

**C. Command handlers** — `Result<IngestResult, IngestError>`, taking `&IngestAuth`.

```rust
async fn handle_dm_open(
    tenant: &TenantContext,
    state: &Arc<AppState>,
    event: &Event,
    auth: &IngestAuth,
) -> Result<IngestResult, IngestError>           // command_executor.rs:310
```

All 7 identical (`:310`, `:443`, `:580`, `:653`, `:809`, `:986`, `:1098`). Each follows the
same 6-step comment skeleton — `// 1. Extract`, `// 2. Validate`, `// Persist the command
event`, `// 4. Execute`, `// Commit`, `// 5./6. Side effects` + `// Return response` —
which makes the group readable as a template. `handle_dm_open` `:310-441` is the reference
implementation.

---

### 2. Validation-function conventions

| Convention | Detail |
|---|---|
| Naming | `validate_*` for pre-storage gates. `validate_edit_ownership` `ingest.rs:763`, `validate_forum_vote_target` `:844`, `validate_diff_event` `:896`, `validate_engram_envelope` `:965`, `validate_persona_envelope` `:1027`, `validate_engram_nip44_content` `:1084`, `validate_agent_turn_metric_envelope` `:1151`, `validate_not_before` `:1223`, `validate_event_reminder` `:1252`, `validate_admin_event` `side_effects.rs:259`, `validate_standard_deletion_event` `:179`, `validate_imeta_tags` `imeta.rs:11`, `validate_repo_id` `side_effects.rs:2391`. |
| Error type | Pure/synchronous validators return `Result<(), String>`; the two `&'static str` returners (`validate_not_before`, `validate_event_reminder`) are the only exceptions, and their strings are closed-set wire values the spec pins for client matching (`ingest.rs:1285-1287`). Async DB-touching validators in `side_effects.rs` return `anyhow::Result<()>`. |
| Error prefix | Validators return **bare** messages; the ingest call site adds the prefix: `.map_err(\|e\| IngestError::Rejected(format!("invalid: {e}")))`. 12 sites use this exact line (`ingest.rs:1934`, `:1918`, `:1924`, `:1962`, `:1968`, `:1973`, `:1978`, `:1983`, `:2020`, `:2025`, `:2214`, `:2217`). ⚠ This produces `invalid: restricted: not a channel member` for edit-ownership failures (`ingest.rs:838`), i.e. a double prefix. |
| Predicate naming | `is_*` for classification: `is_global_only_kind` `ingest.rs:379`, `requires_h_channel_scope` `:455`, `is_admin_kind` `side_effects.rs:26`, `is_side_effect_kind` `:35`, `is_local_media_url` `imeta.rs:373`, `is_well_formed_mime` `:340`, `has_e_tag` `side_effects.rs:2300`, `actor_is_channel_owner_or_admin` `:2357`, `author_delete_can_use_self_delete_path` `:2353`. |
| Position | Every validator runs **pre-storage**. The only post-storage logic is `handle_side_effects`, whose failures are non-fatal by design (`ingest.rs:2460-2467`). |

---

### 3. Error-string wire format

A three-token prefix vocabulary, matched by clients:

| Prefix | Meaning | Typical mapping |
|---|---|---|
| `invalid: ` | client-side protocol/data error | `Rejected` → HTTP 400 |
| `restricted: ` | authorization refusal | either `Rejected` or `AuthFailed`; HTTP 400 or 403 |
| `blocked: ` | community ban | `AuthFailed` → 403 |
| `error: ` | server-side failure | mostly `Internal` → 500 |
| `duplicate: ` | idempotent no-op | `Ok` |
| `info: ` | successful non-storage action | `Ok` |
| `response:` | JSON payload follows (command kinds) | `Ok` |
| `forbidden: ` | **command-executor only** | `Rejected` → 400 |

⚠ Three inconsistencies:
1. `forbidden: ` appears only in `command_executor.rs` (`:509`, `:625`, `:711`, `:845`,
   `:975`, `:982`) and always as `Rejected`, so an authorization failure on a command kind
   returns HTTP **400**, while the same class of failure on any other kind returns **403**.
2. `restricted: ` maps to `Rejected` in some places (`ingest.rs:1482`, `:1507`, `:521`) and
   `AuthFailed` in others (`:1513`, `:1521`, `:1526`, `:1726`, `:2012`).
3. `error: database error: {e}` from `check_channel_membership` (`ingest.rs:501`) is
   surfaced as `Rejected` (`:1802`), giving a 400 for a server fault.

The prefix set is not centralised anywhere. `crate::conformance::sanitized_reason_for`
(`ingest.rs:1411`) is the only place that classifies `IngestError` variants into a closed
alphabet, and it is for the trace, not the wire.

---

### 4. Tag access convention

Every tag read in this group goes through `tag.kind().to_string() == "name"` and
`tag.content()`, or through `tag.as_slice()` for positional access. Two families:

| Helper | Semantics | Copies |
|---|---|---|
| first-match extractor | returns the first matching tag's content | `extract_channel_id` `ingest.rs:308`; `extract_h_tag_channel` `side_effects.rs:2237`; `extract_p_tag` `:2251`; `extract_tag_value` `:2325`; `extract_h_tag` `command_executor.rs:250`; `extract_d_tag` `:261`; `extract_e_tag` `:272`; `extract_tag` `:283` |
| all-match collector | `extract_target_event_ids` `side_effects.rs:2304`; `extract_p_tags` `command_executor.rs:235`; `count_e_tags` `ingest.rs:719` |

⚠ **`e`-tag selection direction is inconsistent and load-bearing.** Reactions take the
**last** `e` tag (`.rev()` — `ingest.rs:334`, `:2251`, `side_effects.rs:2192`), per NIP-25.
Edits (`ingest.rs:766`), votes (`:847`), deletion channel derivation (`:1670`), and 9005
(`side_effects.rs:531`) take the **first**. Nothing names or documents the rule; it must be
read off each call site.

⚠ **Duplicated helpers.** `effective_message_author` exists twice with identical bodies —
`ingest.rs:729-761` (`pub(crate)`) and `side_effects.rs:2271-2298` (private). The
`side_effects.rs` copy uses `extract_tag_value(event, "actor")` where the `ingest.rs` copy
inlines the same loop. `side_effects.rs:2195` then reaches back for
`super::ingest::effective_message_author`, so both copies are live in the same file's call
graph. Similarly `extract_channel_id` (`ingest.rs:308`) and `extract_h_tag_channel`
(`side_effects.rs:2237`) are byte-equivalent, and `command_executor.rs:250` has a third
variant returning `Option<String>` instead of `Option<Uuid>`.

---

### 5. How a new kind is added (the actual sequence)

Derived from the code, not from docs — no doc describes this.

1. Add the constant to `crates/buzz-core/src/kind.rs` and to `ALL_KINDS`
   (`kind.rs:566-693`). Add a compile-time range assertion if the kind is
   replaceable/parameterized (`kind.rs:783-820`).
2. Add a match arm to `required_scope_for_kind` (`ingest.rs:198-306`). **Without this the
   kind is rejected with `restricted: unknown event kind`.** This is the real gate.
3. Add it to exactly one of `is_global_only_kind` (`ingest.rs:379-453`) or
   `requires_h_channel_scope` (`:455-491`), or neither if the channel is derived some other
   way. The disjointness test (`:2753-2762`) will catch getting both.
4. If the kind needs pre-storage validation, write a `validate_*` returning
   `Result<(), String>` and call it in the `ingest_event_inner` gauntlet
   (`ingest.rs:1986-2052` is where the per-kind validators cluster).
5. If it needs post-storage effects, add it to `is_side_effect_kind`
   (`side_effects.rs:35-37`) **and** add a `handle_side_effects` arm
   (`:143-176`). Both, or the kind is silently ignored.
6. If it is a transactional command, add it to `buzz_core::kind::is_command_kind`
   (`kind.rs:743-755`), then a `handle_command` arm (`command_executor.rs:66-77`) plus a
   handler following shape C.
7. If it is relay-signed only, add it to `is_relay_only_kind` (`kind.rs:758-769`) so the
   reject message is `restricted: relay-only kind` rather than
   `restricted: unknown event kind`. ⚠ Eight relay-minted kinds skip this step today
   (8000, 8001, 8002, 8003, 13535, 39000, 39001, 39002, 40099) — see features.md §3.
8. Add unit tests to the `ingest.rs` test module asserting scope, global-only, and
   `requires_h` classification. The existing suite has one test per property per kind
   family (e.g. `:2683-2718` for NIP-51 lists, `:2909-2927` for teams/managed agents), and
   `per_kind_scope_allowlist_covers_all_migrated_kinds` (`:2822-2879`) is the running
   checklist — 44 kinds are listed there, out of 81 accepted, so it is not exhaustive.

There is **no** single registry or trait. Adding a kind touches 3–6 disjoint `match`
statements across 2 crates, none of which are exhaustive over `ALL_KINDS`. Nothing fails to
compile if a step is skipped.

---

### 6. Test conventions

| Convention | Detail |
|---|---|
| Location | `#[cfg(test)] mod tests` at the bottom of the file: `ingest.rs:2532`, `side_effects.rs:3266`, `imeta.rs:419`. `command_executor.rs` has none. |
| Style | Pure-function unit tests only. **Zero** `#[tokio::test]`, so no handler with a DB dependency is tested in-file. All 111 tests are synchronous. |
| Builders | `ingest.rs` has a small builder set: `make_dummy_event()` `:3045`, `make_event_with_tags(kind, content, &[&[&str]])` `:3053`, then kind-specific wrappers `make_engram` `:3083`, `make_reminder` `:3283`, `make_persona` `:3421`, `make_agent_turn_metric` `:3541`, plus `fake_nip44_v2()` `:3090` producing a shape-valid 99-byte NIP-44 v2 payload. |
| Assertion style | Property assertions loop over kind arrays with a message naming the kind: `assert!(is_global_only_kind(kind), "kind {kind} must be global-only")` (`ingest.rs:2597`, `:2707`, `:2865`, …). Error assertions match on substrings, not equality: `assert!(err.contains("`p` tag"), "got: {err}")` (`ingest.rs:3137`). |
| Exhaustive properties | One brute-force test over the whole kind space: `global_only_and_channel_scoped_are_disjoint` iterates `0..=65535` (`ingest.rs:2779-2788`). |
| Regression documentation | Regressions carry a doc comment explaining the failure they prevent, e.g. the uppercase-`p` invisibility bug (`ingest.rs:2612-2616`) and the non-base64 replacement bug (`:3169-3174`). |
| Metrics tests | `reject_with_transport_labels_http_and_ws_as_separate_series` (`ingest.rs:3752-3793`) uses `metrics_util::debugging::DebuggingRecorder` + `with_local_recorder` to assert label cardinality. The only metrics test in the group. |
| Conformance tests | `feedback_success_action_satisfies_ingest_emit_guard` (`ingest.rs:2557-2591`) arms a real `EmitGuard` against a `VecTracer` (`:2545-2555`) and asserts exactly one `WriteInsertGlobal`. The pattern exists for one kind only. |
| `side_effects.rs` tests | All 5 cover pure helpers: `delete_tombstone_content` (a `#[cfg(test)]`-only function at `:2363-2391` that duplicates the production tombstone builder at `:1650-1656`), `author_delete_can_use_self_delete_path`, `actor_is_channel_owner_or_admin`. Nothing touches the 418-line `validate_admin_event`. |
| Integration coverage | Behaviour of this group is covered out-of-file in `crates/buzz-test-client/tests/` — e.g. `e2e_human_edit_agent_content.rs:5-6` names `validate_standard_deletion_event` and the `validate_admin_event` 9005 branch as its subjects. Per AGENTS.md these need Postgres + Redis (`just test`). |

---

### 7. Logging conventions

| Convention | Detail |
|---|---|
| Levels | `debug!` for pipeline entry (`ingest.rs:1458`); `info!` for accepted writes and completed side effects (`ingest.rs:2385`, `:2499`); `warn!` for every swallowed side-effect failure (32 sites in `side_effects.rs`); `error!` for genuine bugs (`ingest.rs:1498` spawn panic, 8 sites in `command_executor.rs` for spawned-task failures). |
| Structured fields | `event_id = %hex`, `kind = u32`, `channel = %uuid`, `target = %hex`, `pubkey = %hex`, `error = %e`. Consistent throughout. |
| Sensitive values | Pubkeys are logged as hex — acceptable (they are public). Event content is never logged. Reject reasons that may embed event-controlled data are truncated **at the transport**, not here (`api/bridge.rs:842-851`). |
| Success/failure symmetry | `ingest.rs` logs one `info!` per accepted event (`:2359` reaction, `:2499` generic) but nothing on rejection — rejection telemetry is the `buzz_events_rejected_total` counter, emitted by the transport (`reject_with_transport` `:156`), not by ingest. |

---

### 8. Comment conventions

This group is unusually heavily commented, and the comments carry design decisions rather
than restating code. Recurring patterns:

- **Decision references.** `(OQ1 decision …)` `ingest.rs:1794`; `(E1 within-request
  threading; correctness ruling §4.8)` `:485`; `(§4.8 phase-2 addendum)` `:1742`;
  `(COMMUNITY_MODERATION_PLAN.md §0 decision 4)` `:1589`; `(C5)` double-count analysis
  `side_effects.rs:1670-1679`; `(F9)` `buzz-db/src/thread.rs:114`;
  `(spec line 794)` `ingest.rs:1776`.
- **Load-bearing-ordering markers.** "Ordering is load-bearing" `ingest.rs:2320`;
  "Redis before local fan-out so subscribers on other relay pods receive it too"
  `side_effects.rs:783`; "Listed after the workflow branch so workflow's bespoke deletion
  … takes precedence" `:2051-2056`.
- **Known-limitation blocks.** `ingest.rs:369-377` (stray `h` on the read path);
  `side_effects.rs:952-960` (channel-scoped discovery vs live global subs);
  `:1516-1524` (four sub-second archive toggles);
  `command_executor.rs:92-98` (non-atomic command mutations).
- **Negative rationale** — why something is *not* done: "we intentionally do NOT check
  is_agent_owner for non-members" `side_effects.rs:365-367`; "Not reachable in practice …
  so we don't engineer around it" `:1522-1524`; "diverges from kind:9001 intentionally"
  `:598-600`, `:632-634`.
- **Unreachable-arm annotations.** `RemoveResult::RoleMismatch` is documented as
  "unreachable but exhaustiveness requires it" (`ingest.rs:1901-1902`).

---

### 9. Metrics conventions

Label cardinality is reasoned about explicitly. Fleet-wide counters carry `kind` but
**no** `community`, because `bounded_kind_label` passes through all 10 000 ephemeral kind
values and crossing kind × community "would produce up to millions of series"
(`handlers/event.rs:629-632`). Per-community counters carry `community` but no `kind`.
`author_type` is accepted as a label only because it is 2-valued and "merely doubles the
kind series" (`ingest.rs:1391-1394`). `reject_with_transport`'s `reason` is documented as
"one of a small closed set … bounded, no cardinality risk" (`ingest.rs:150-155`).

---

### 10. AGENTS.md compliance

| Rule | Status |
|---|---|
| No `unsafe` | ✅ 0 occurrences in 8 911 lines |
| No new `unwrap()`/`expect()` in production paths | ❌ 4 `expect()` (`ingest.rs:2024`, `:2000`, `:2338`, `:2344`) + 2 `unwrap()` (`side_effects.rs:311`, `:314`) |
| New public API must have doc comments | ✅ every `pub` item in all four files is documented; `IngestAuth`'s fields are individually documented (`ingest.rs:64-85`) |
| Channels use `h` tags, not `e` tags | ✅ `extract_channel_id` reads only `h` (`ingest.rs:308-319`) |
| Event kinds defined in `buzz-core/src/kind.rs` | ⚠ mostly — `KIND_PUSH_LEASE` is defined in **both** `buzz-core/src/kind.rs:109` and `handlers/push_lease.rs:19`, and `ingest.rs` imports the `push_lease` copy (`ingest.rs:216`, `:451`, `:2156`) rather than the `buzz-core` one. Two sources of truth for one integer. |
| Prefer Nostr events over new HTTP endpoints | ✅ this group adds no HTTP surface |
| Thread counters updated on every reply-insert path | ✅ verified — see data-model.md §7 |
| 1 000-line file ceiling | ❌ not applicable to Rust (guard is JS/mobile only), but `ingest.rs` (3 686) and `side_effects.rs` (3 347) are 3.3–3.7× over the ceiling the repo applies elsewhere — see debt.md D-01 |


## Module: buzz-relay — HTTP API surface (`crates/buzz-relay/src/api`)
### Aspect: Conventions

---

#### 1. Handler signature conventions

Three distinct return-type dialects coexist in one module tree:

| Dialect | Return type | Used by | `file:line` |
|---|---|---|---|
| **Bridge/invites/operator** | `Result<Json<Value>, (StatusCode, Json<Value>)>` | 12 handlers | `bridge.rs:616`, `:883`, `:1317`, `:2077`, `:2103`, `:2121`; `invites.rs:233`, `:294`, `:165`; `operator.rs:152`, `:206`, `:268`, `:305`, `:357`, `:471` |
| **Media** | `Result<Json<BlobDescriptor>, MediaError>` / `Result<Response, MediaError>` — a domain error type with its own `IntoResponse` | 3 handlers | `media.rs:305-310`, `:604-608`, `:798-802` |
| **Admin** | `Result<Json<T>, ApiError>` with a typed error struct | 5 handlers | `admin/mod.rs:93-97`, `:125-129`, `:151-154`, `:177-181`, `:191-195` |

Two more one-offs: `workflow_webhook` returns `Result<(StatusCode, Json<Value>), (StatusCode, Json<Value>)>`
so it can emit 202 (`bridge.rs:1766`); `demo_echo` and `nostr_nip05` return bare `Response`
(`mesh_demo.rs:61`, `nip05.rs:29`), i.e. they are infallible by construction.

##### Extractor ordering

Consistent and load-bearing: `State` first, then `Path`, then `RawQuery`, then `Query`, then
`HeaderMap`, then the body (`Bytes` / `Body` / `Json`).

- `bridge.rs:2096-2099` — `State`, `HeaderMap`, `RawQuery`, `Query`: `RawQuery` **and** `Query` are
  both taken because NIP-98 signs the verbatim query string while the handler wants typed params.
  Same pattern at `operator.rs:305-309`, `:471-475`.
- `media.rs:306-309` — `State`, `AuthenticatedUpload` (a `FromRequestParts` extractor), `HeaderMap`,
  `Body`. The auth extractor runs before the body extractor by axum's rules, which is the whole
  point (`media.rs:29-38`).
- `mesh_demo.rs:59-62` — `State`, `Json<DemoEchoRequest>`. This is the one place where extractor
  ordering **defeats** the handler's intent: the `Json` rejection fires before the feature-flag 404.

##### Body handling

Handlers that need the raw bytes for NIP-98 payload verification take `axum::body::Bytes` and
`serde_json::from_slice` manually (`bridge.rs:618`, `:885`, `:1319`; `invites.rs:167`, `:236`;
`operator.rs:155`). Only `mesh_demo.rs:61` uses the `Json<T>` extractor — and it is the only handler
with no NIP-98 requirement, so nothing is lost there except the 404-indistinguishability property.

##### Tenant-binding preamble

Every tenant-scoped handler repeats the same 8-line block verbatim: read `Host`, `to_str().ok()`,
`unwrap_or("")`, `bind_community`, `map_err` to a fixed 404. Copied at `bridge.rs:621-633`,
`:888-901`, `:1321-1334`, `:1777-1786`, `:2013-2025`; `invites.rs:198-207`; `media.rs:154-166`,
`:477-487`; `nip05.rs:31-40`. Only `media.rs` factors it (`bind_media_read_tenant`, `media.rs:478`).

##### Post-auth helper split

`/events`, `/query`, `/count` each split into a thin routed wrapper plus an `_authed` helper, so the
wrapper can own exactly one terminal attribution log for every outcome:

| Wrapper | Helper | `file:line` |
|---|---|---|
| `submit_event` | `submit_event_authed` | `bridge.rs:613` / `:750` |
| `query_events` | `query_events_authed` | `bridge.rs:880` / `:947` |
| `count_events` | `count_events_authed` | `bridge.rs:1316` / `:1378` |

`submit_event` additionally routes outcomes through a `SubmitOutcome` enum carrying both log fields
and the HTTP response (`bridge.rs:706-747`), with `into_response()` collapsing it (`:722-729`).

#### 2. Error-to-status mapping

##### Envelope helpers (`api/mod.rs`)

```rust
api_error(status, msg)  -> (status, Json({"error": msg}))     // mod.rs:19-21
internal_error(msg)     -> tracing::error!; api_error(500, "internal server error")  // mod.rs:23-26
not_found(msg)          -> api_error(404, msg)                // mod.rs:28-31
```

`internal_error` is the only helper that deliberately withholds detail from the client while keeping
it in logs. It is used 30+ times across the module for every DB/serialize failure.

##### Status conventions actually observed

| Condition | Status | Body |
|---|---|---|
| Unmapped `Host` | 404 | fixed `"relay: no community is configured for this host"` (never echoes the host) |
| Missing/invalid NIP-98, replay detected, replay guard down | 401 | `"missing Nostr auth"` / `"NIP-98: {e}"` / `"NIP-98: replay detected"` / `"NIP-98: replay check unavailable"` |
| Read-gate violation (p-gate / engram / author-only) | **403** | `"restricted: …"` — note the WS sibling sends a `CLOSED` frame with an analogous string (`handlers/req.rs:186-204`) |
| Not a relay member | 403 | `{"error":"relay_membership_required","message":…}` — the **only** two-key error body in the non-admin surface (`api/mod.rs:134-139`) |
| Malformed filter/cursor/JSON | 400 | reflects the `serde_json` message |
| Rate limited | 429 | `"rate-limited: quota exceeded; retry in {n}s"` |
| Rate limiter unavailable | 503 | `"rate-limited: shared admission unavailable"` |
| Any DB/serialize error | 500 | fixed `"internal server error"` |

##### Media (`buzz-media/src/error.rs:107-162`)

All 15 authentication-ish variants (`MissingAuth`, `InvalidBase64`, `HashMismatch`, `ServerMismatch`,
`MissingTag`, `TokenExpired`, …) collapse to one 401 `"authentication failed"` explicitly "to prevent
oracle enumeration" (`error.rs:118-121`). `InsufficientScope` and `RelayMembershipRequired` are 403;
size 413; content-type 415; validation 422; rate/concurrency 429; IO/storage 500 with a generic body.

##### Admin (`admin/error.rs`)

Four constructors only — `bad_request(code, message)`, `forbidden()`, `not_found()`, `internal()` —
with `&'static str` code/message, so **no dynamic text can leak** through this surface by
construction. `From<DbError>` maps unconditionally to `internal()` (`admin/error.rs:79-83`).

##### Operator error mapping by string prefix

`operator.rs:180-199` matches on the `String` error returned by `community_provisioning`:
`"actor not authorized"`→403, `"community already exists"` / `"limit_reached:"`→409,
`"failed to create community:"` / `"community provisioned but owner bootstrap failed:"`→500 (generic
body + `tracing::error!`), fallthrough→400 with the message passed through. This is stringly-typed
control flow — a wording change in the provisioning module silently reclassifies the HTTP status.

#### 3. JSON shape conventions

| Convention | Where | Deviations |
|---|---|---|
| snake_case keys | bridge, invites, operator, media, mesh demo | admin uses `camelCase` (`admin/mod.rs:64`, `:140`, `:22`) |
| Reads return a bare JSON **array** | `/query`, `/moderation/*` | `/count` returns `{count}`; NIP-05 returns an object |
| Writes return `{event_id, accepted, message}` | `POST /events` (`bridge.rs:834-838`) | invites return `{code,…}` / `{status,…}`; operator returns `{community_id,…}` |
| Hex-encode all byte fields | `report_json`/`action_json`/`ban_json` (`bridge.rs:2132-2184`) | — |
| `#[serde(skip_serializing_if = "Option::is_none")]` for optional response fields | `operator.rs:51`, `handlers/community_provisioning.rs:67` | most ad-hoc `json!` bodies emit `null` instead |
| Ad-hoc `serde_json::json!` for response bodies | dominant style — 20+ sites | only 4 typed `Serialize` response structs exist (`TransferCommunityResponse`, `ProvisionCommunityResponse`, `FeedbackSummary`, `ErrorEnvelope`) |
| Empty-result shape is `[]` / `{}` , never 404 | `/query`, `/count`, NIP-05, `/api/join-policy` | — |
| Request DTOs are permissive | 12 of 13 DTOs ignore unknown fields | `ReportQuery` alone uses `deny_unknown_fields` (`admin/mod.rs:64`) |

#### 4. Tenancy conventions

- **Row zero**: bind the community from `Host` before anything else. The phrase "Row zero" appears
  as a literal comment marker at `bridge.rs:621`, `:888`, `:1321`, `:1773`; `media.rs:145`;
  `nip05.rs:32`; `router.rs:280`. Grepping for it is the fastest way to audit door coverage.
- **Never derive identity from `config.relay_url`'s host** — only its scheme. Three helper pairs
  encode this: `nip98_expected_url` (`bridge.rs:195-206`), `nip42_expected_relay_url`
  (`bridge.rs:225-231`), `media_base_url_for_tenant` (`media.rs:447-455`),
  `relay_url_for_tenant_host` (`nip05.rs:105-111`). Each has a paired test asserting the config host
  does **not** influence the output (`bridge.rs:2636-2654`, `:2749-2771`; `media.rs:1272-1280`;
  `nip05.rs:143-152`).
- Two deliberate exceptions, both documented in place: operator routes authenticate against
  `relay_operator_api_origin` and never bind a tenant (`operator.rs:57-60`); the admin surface is
  deployment-wide by design (`docs/admin/README.md:1-9`).

#### 5. Comment / documentation conventions

- Doc comments on handlers state method + path + auth mechanism as the first line
  (`bridge.rs:612`, `:877`, `:1310`; `media.rs:589-601`; `invites.rs:225-229`).
- Security-relevant decisions are argued inline at length, often citing the attack they close and
  the PR/review that found it: `bridge.rs:184-193` (NIP-98 host binding), `:208-224` (NIP-42
  sibling), `:1582-1594` (search post-filter), `:595-611` (log truncation), `media.rs:145-156`
  (bind-before-verify ordering), `invites.rs:36-43` (limiter capacity rationale),
  `invite_token.rs:24-46` (security properties **and non-properties**).
- Tests carry "bites if …" statements naming the regression they detect
  (`bridge.rs:2326-2328`, `:2414-2416`, `:2524-2528`, `:2706-2707`, `:3396-3400`).
- Module headers enumerate routes (`media.rs:3-8`, `invites.rs:3-13`, `mesh_demo.rs:1-23`).
- `// sadscan:disable np.postgres.1` suppresses the hardcoded-credential scanner on test DB URLs
  (`bridge.rs:3283`, `invites.rs:429`, `operator.rs:589`).

#### 6. Test conventions

**Counts (all 13 assigned files):** 159 test functions, **28** `#[ignore]`d, 0 `unsafe` blocks
(one occurrence of the word in a doc comment at `bridge.rs:303`), **1** TODO marker
(`media.rs:303`), **0** `unwrap()` outside `#[cfg(test)]`, **5** `expect()` outside `#[cfg(test)]`
(all in `invite_token.rs`: `:119`, `:139`, `:172`, `:349`, `:374` — every one an
infallible-by-construction HMAC/serialize call).

| File | tests | `#[ignore]` |
|---|---|---|
| `bridge.rs` | 64 | 8 |
| `media.rs` | 33 | 0 |
| `invites.rs` | 14 | 9 |
| `operator.rs` | 11 | 11 |
| `admin/mod.rs` | 10 | 0 |
| `webhook_secret.rs` | 10 | 0 |
| `invite_token.rs` | 9 | 0 |
| `api/mod.rs` | 3 | 0 |
| `nip05.rs` | 2 | 0 |
| `mesh_demo.rs` | 2 | 0 |
| `admin/auth.rs` | 1 | 0 |
| `admin/error.rs`, `events.rs` | 0 | 0 |

Established patterns:

1. **`#[ignore = "requires Postgres"` / `"requires Redis"`** is the gate for anything touching real
   infrastructure — 28 tests. Every `operator.rs` test is ignored, so the operator surface has
   **zero** coverage in `just test-unit`.
2. **Router-level `oneshot`** via `tower::ServiceExt` drives real HTTP through `build_router`, so
   route registration + extractor order + middleware are all in scope:
   `bridge.rs:3372-3390`, `invites.rs:598-620`, `operator.rs:635-660`, `admin/mod.rs:375-391`,
   `media.rs:1000-1010`.
3. **Injected `Nip98ReplayGuard` doubles** instead of Redis — four distinct doubles:
   `AlwaysErrGuard` (`bridge.rs:2348`), `AlwaysFreshReplayGuard` (`bridge.rs:3286`,
   `invites.rs:415`, `operator.rs:551`), `SeenOnceReplayGuard` (`invites.rs:1103`). The fail-closed
   test needs **no infrastructure** because the double supplies the error (`bridge.rs:2321-2377`).
4. **Positive controls beside every negative test** so a cross-host rejection test cannot pass
   vacuously (`bridge.rs:2477-2504`, `:2731-2747`).
5. **`current_thread` runtime + `metrics::with_local_recorder`** for metric assertions, because the
   recorder is a thread-local that a multi-thread scheduler would lose — reasoning spelled out at
   `bridge.rs:3255-3262`.
6. **Custom `tracing` capture writer** to assert the exactly-one-attribution-line invariant
   (`bridge.rs:3512-3584`).
7. **Test state builders** named `*_test_state` that mutate `Config::from_env()` then override
   `state.nip98_replay`: `bridge_handler_test_state` (`bridge.rs:3304`),
   `invite_test_state` (`invites.rs:441`), `operator_test_state` (`operator.rs:591`),
   `test_state_with_media_get_auth` (`media.rs:951`), `test_state` (`admin/mod.rs:335`).
8. **Silent skip vs panic is inconsistent**: `invites.rs:664-666` and `operator.rs:686-688` do
   `let Some(state) = … else { return; }` (test passes vacuously when Postgres is absent), while
   `bridge.rs:3423-3425` and `invites.rs:1074-1076` `panic!`/`expect` with an actionable message.
9. **Community isolation asserted explicitly** for every per-pubkey limiter
   (`media.rs:1120-1161`, `invites.rs:481-503`) and for the replay seen-set (`bridge.rs:2290-2292`).
10. Helper `nip98_auth_header(keys, url, method, body)` is **duplicated** in `invites.rs:505-519`
    and `operator.rs:596-616` with near-identical bodies, plus a third variant
    `build_nip98_event_json` + `nip98_auth_headers` in `bridge.rs:2380-2404`.

#### 7. Naming conventions

| Pattern | Examples |
|---|---|
| `verify_*` — cryptographic check returning `Result` | `verify_bridge_auth`, `verify_secret`, `verify_invite`, `verify_policy_acceptance` |
| `enforce_*` — check that returns an HTTP error tuple | `enforce_http_admission`, `enforce_relay_membership` |
| `authorize_*` — auth prelude returning the authenticated principal or tenant | `authorize_moderation_read`, `authorize_operator_request`, `authorize` (admin) |
| `extract_*` — pull an optional value out of raw JSON / headers | `extract_before_id`, `extract_depth_limit`, `extract_feed_types`, `extract_thread_cursor`, `extract_search_mode`, `extract_search_page`, `extract_page_offset`, `extract_channel_from_filter`, `extract_blossom_auth`, `extract_secret`, `extract_nip_oa_owner`, `extract_domain` |
| `*_json` — hand-rolled row→`Value` projection | `report_json`, `action_json`, `ban_json` |
| `handle_*_filter` / `handle_bridge_*` — one `/query` dispatch branch | `handle_channel_window_filter`, `handle_bridge_search` |
| `*_authed` — post-authentication continuation | `submit_event_authed`, `query_events_authed`, `count_events_authed` |
| `is_*` — pure boolean predicate | `is_safe_ext`, `is_sha256`, `is_admin_host`, `is_member_tag` |
| SCREAMING_SNAKE consts co-located with the code that reads them | `BRIDGE_FEED_MAX_LIMIT`, `MODERATION_READ_LIMIT`, `MAX_RANGE_CHUNK`, `CLAIM_RATE_LIMIT`, `MAX_INVITE_TTL_SECS`, `ECHO_TIMEOUT` |

#### 8. Convention violations / inconsistencies

| Issue | `file:line` |
|---|---|
| Three incompatible handler error dialects in one module tree (tuple / `MediaError` / `ApiError`) | see §1 |
| Two incompatible error-envelope JSON shapes | `api/mod.rs:19-21` vs `admin/error.rs:16-28` |
| camelCase only in the admin sub-tree | `admin/mod.rs:64`, `:140` |
| Stringly-typed status classification in the operator provisioning path | `operator.rs:180-199` |
| Tenant-binding preamble copy-pasted 9 times; factored only in `media.rs` | `media.rs:478-488` vs the rest |
| Stale `#[allow(dead_code)]` on a live function | `api/mod.rs:28` |
| `#[allow(private_interfaces)]` placed **between** two doc-comment blocks, splitting the doc | `media.rs:299-304` |
| `let _pubkey = …` discards the acting operator on transfer; archive/unarchive discard it entirely | `operator.rs:355`, `:209`, `:271` |
| `nostr_nip05` folds DB errors and misses into the same 200 via a catch-all `_ =>` arm | `nip05.rs:64` |
| `deny_unknown_fields` used on exactly one DTO | `admin/mod.rs:64` |
| `serve_blob_for_tenant` re-runs `validate_media_path` that its two callers already ran | `media.rs:604`/`:619`, `:630` |


## Module: buzz-relay — git hosting (`crates/buzz-relay/src/api/git`)
### Aspect: Conventions

#### 1. Module layout

| File | LOC | Layer | Depends on |
|---|---|---|---|
| `mod.rs` | 66 | wiring + `require_localhost` | `policy`, `transport` |
| `transport.rs` | 2 288 | HTTP handlers, subprocess runners, fence | all others |
| `cas_publish.rs` | 1 891 | commit protocol | `manifest`, `store`, `transport::harden_git_env` |
| `store.rs` | 1 164 | object-store primitives + conformance probe | none (leaf) |
| `hydrate.rs` | 893 | read/write materialization | `manifest`, `store`, `pack_cache`, `cas_publish::ParentState` |
| `policy.rs` | 775 | HMAC callback endpoint | `buzz_core::git_perms`, `buzz_db` |
| `pack_cache.rs` | 686 | bounded local cache | `hydrate` (`pub(super)` helpers), `store` |
| `manifest.rs` | 570 | schema + predicates | leaf except `CommunityId` |
| `manifest_event.rs` | 395 | manifest → kind:30618 | `nostr` only (pure) |
| `hook.rs` | 207 | bash hook literal + installer | none |

Dependency direction is mostly clean, with two documented back-edges: `pack_cache` → `hydrate` (`hydrate.rs:272`, `:381` are `pub(super)` for exactly this) and `cas_publish`/`hydrate` → `transport::harden_git_env` (a `pub(crate)` helper living in the HTTP layer, which is where it least belongs).

#### 2. Error handling

Three distinct idioms, one per layer:

| Layer | Error type | Idiom |
|---|---|---|
| HTTP handlers | `Result<Response, Response>` | errors *are* responses, built at the failure site. `#[allow(clippy::result_large_err)]` is applied where needed (`transport.rs:265`, `:317`, `:1413`) |
| Protocol layers | `thiserror` enums | `CasError` (7 variants, `cas_publish.rs:90-145`), `HydrateError` (5, `hydrate.rs:96-121`), `StoreError` (6, `store.rs:65-102`), `ManifestError` (9, `manifest.rs:74-131`), `BuildError` (2, `manifest_event.rs:44-52`) |
| Hook installer | `anyhow::Result` | the only `anyhow` in the module (`hook.rs:152`) |

Conventions worth naming:

- **Variants encode HTTP class, not just cause.** `CasError::Conflict` is deliberately separate from `Backend` so `?`-bubbling cannot turn a 412 into a 500 (`cas_publish.rs:105-124`); `ManifestInvalid` is 4xx-class while `ManifestReadFailed` is 5xx-class (`:126-141`). `finalize_push` maps all seven (`transport.rs:1601-1657`).
- **"Not found" is a type, not a variant.** `hydrate_for_read` returns `Result<Option<HydratedRepo>>` so 404-vs-5xx is enforced by the type system (`hydrate.rs:94-97`).
- **Semantic non-errors are values.** `CasOutcome::LostRace` for HTTP 412 (`store.rs:52-64`); a 412 on a content-addressed PUT is folded into `Ok(key)` (`store.rs:271-275`).
- **Fail closed on ambiguity.** A 2xx CAS without an ETag becomes an error rather than `ETag("")` (`store.rs:522-536`); `get_pointer` errors on a missing ETag (`:462-471`); the policy endpoint returns 403 for *every* failure including DB errors (`policy.rs:277-282`).
- **Non-fatal degradations are explicit `warn!` + continue**, never swallowed: idx sidecar write (`cas_publish.rs:1148-1152`, `:828-844`), idx validation failure (`hydrate.rs:398-406`), kind:30618 build/insert (`transport.rs:1712-1735`), compaction fallback (`cas_publish.rs:1096-1103`).
- **Error strings are opaque to clients.** All git 5xx bodies are the literal `"git error"` (`transport.rs:630-700`, `:1004-1124`); detail goes to `tracing`. Policy denials leak only role/rule text.
- **`?` is used freely inside protocol layers; handlers use explicit `match` + `map_err`** so each arm can pick a status.

#### 3. Panic policy

AGENTS.md forbids new `unwrap()`/`expect()` in production paths. Current production count (outside `#[cfg(test)]`): **17**.

| File | Count | Lines | Assessment |
|---|---|---|---|
| `transport.rs` | 10 | 97, 108, 334, 572, 721, 1037, 1442, 1469, 1493, 1507 | 7 are `Response::builder().body(..).unwrap()` (infallible); 3 are `child.stdin/stdout.take()` immediately after `Stdio::piped()` (infallible by construction, one of them written as `.unwrap()` rather than `.expect()`) |
| `store.rs` | 5 | 262, 297, 490, 892 (`"*".parse().unwrap()` — const header value), 718 (`new_etag.expect("winner exists")` — guarded by `winners == 1` two lines above) | all structurally infallible |
| `cas_publish.rs` | 1 | 410 — `pack_path.to_str().unwrap()` | path is `tempdir/pack-<hex>.pack`; the sibling compaction path handles the same case with a real error (`:702-705`), so this one is an inconsistency |
| `policy.rs` | 1 | 126 — `Hmac::new_from_slice(..).expect("HMAC can take key of any size")` | infallible for HMAC |
| `hydrate.rs`, `pack_cache.rs`, `manifest.rs`, `manifest_event.rs`, `hook.rs`, `mod.rs` | 0 | — | clean |

Two additional panic-avoidance conventions:

- `pack_cache` never `unwrap()`s a poisoned mutex: `self.state.lock().unwrap_or_else(|e| e.into_inner())` at `:242`, `:335`, `:350`, `:367`.
- `pkt_line` degrades an over-long payload to an empty `0004` frame + `error!` rather than panicking or truncating a length prefix, "non-panicking in every build profile" (`transport.rs:424-454`, pinned `:2231-2237`).

`unsafe`: **0** occurrences anywhere in the module (the only matches for the string are `unsafe_refname` identifiers and error text).

`TODO`/`FIXME`/`XXX`/`HACK`: **0** occurrences.

#### 4. Streaming patterns

Two mutually exclusive strategies, chosen by whether the operation mutates published state (`transport.rs:1244-1260`, `:1405-1412`):

**Read paths — stream.** `stream_git_read` (`transport.rs:1414-1498`) composes four layers:
1. `ReaderStream` over `ChildStdout`.
2. `TimedByteStream` — hard deadline, byte/duration histograms in `Drop` (`:1282-1391`).
3. `StreamingGit` — parks the `Child` and the `HydratedRepo` to extend their lifetimes past the last byte, aborts the stdin pump on `Drop`, and kills the child when it observes a `TimedOut` item (`:1226-1332`).
4. `GitPermitStream` — holds the semaphore permit to EOF (`:1293-1310`).

**Write path — buffer.** `run_git_at` returns an owned `PackOutput` (`transport.rs:971-991`) precisely so no `Response` can exist before the CAS. The rationale is documented at the type (`:963-970`) and at the streaming helper (`:1244-1260`).

Request bodies are always pumped by a **spawned task**, never awaited inline, so stdin backpressure cannot deadlock against stdout reads (`transport.rs:1039-1064`, `:1442-1467`). Both pumps log body/decode errors at `warn` and then close stdin so git sees EOF rather than an opaque hang.

Body decoding is a stream transformer with a running counter, not a buffer-then-check (`transport.rs:766-783`).

#### 5. Temp-file and cleanup discipline

| Rule | Practice |
|---|---|
| Scratch location | Every tempdir/tempfile is created with `*_in(scratch_dir)` so it lands on the mounted volume, never `/tmp`. `cas_publish` derives its scratch as `repo_path.parent()` rather than taking another argument (`cas_publish.rs:1028-1030`). |
| Subprocess output | `NamedTempFile::new_in(...)` + `.reopen()` → `Stdio::from(file)`. The `NamedTempFile` handle stays in scope so `Drop` unlinks it; the reopened descriptor is what the child writes. Repeated verbatim at 8 sites (`transport.rs:622-644`, `:996-1018`; `cas_publish.rs:272-284`, `:511-523`, `:595-607`, `:704-712`). |
| Metadata-before-read | Every buffered read is `tokio::fs::metadata(...).len()` → size check → `tokio::fs::read(...)`, never read-then-check (`transport.rs:680-701`, `:1085-1126`; `cas_publish.rs:307-315`, `:552-568`). |
| Bounded stderr | `read_log_prefix` / `read_prefix` cap at 64 KiB using `AsyncReadExt::take`, returning `"<stderr unavailable>"` on failure — duplicated in two files (`transport.rs:1141-1155`, `cas_publish.rs:877-891`). |
| Lifetime-as-cleanup | `HydratedRepo` owns its `TempDir` (`hydrate.rs:51-56`); the explicit `drop(repo)` / `drop(ctx.repo_handle)` calls at `transport.rs:707`, `:1576`, `:1750` are load-bearing ordering, and are commented as such. |
| `kill_on_drop(true)` | On 6 of 10 subprocess sites; absent on the four `.output()`/`.status()` sites that already await completion (`cas_publish.rs:284`, `:337`, `:409`) — except `hydrate::run_git`, which sets it despite using `.output()` (`hydrate.rs:453`). |
| Cache publication | staging `TempDir` → atomic `rename` into the final digest directory; on rename failure the winner's directory is adopted (`pack_cache.rs:274-333`). |
| Process-lifetime GC | Startup sweep of stale `session-*` dirs plus a 60 s heartbeat writer aborted in `Drop` (`pack_cache.rs:127-146`, `:420-426`, `:482-509`). |
| Bash hook | `WORK_DIR=$(mktemp -d)` with `trap 'rm -rf "$WORK_DIR"' EXIT` (`hook.rs:49-53`). |

#### 6. Concurrency conventions

| Mechanism | Scope | Site |
|---|---|---|
| `git_semaphore` | global, `try_acquire_owned` (never blocks), 503 on exhaustion | `transport.rs:318-338` |
| `PACK_COMPACTION_SEMAPHORE` | process-global `const_new(1)`, `acquire` with 300 s timeout | `cas_publish.rs:86`, `:576-586` |
| `population_semaphore` | per-cache, bounds concurrent object-store pack fetches | `pack_cache.rs:110`, `:253-269` |
| Per-digest single-flight | `DashMap<String, Arc<PopulationFlight>>` with an `AtomicUsize` refcount and an RAII `FlightParticipant` that deregisters on the last drop | `pack_cache.rs:76-104`, `:186-238`, `:392-399` |
| **No per-repo lock** | deliberate; the CAS is the only writer serialization, with a 14-line justification comment | `transport.rs:865-877` |
| Sync mutex in async | `std::sync::Mutex` for cache index (never held across `await`) | `pack_cache.rs:63`, `:241-390` |

#### 7. Documentation conventions

- Every file opens with a `//!` module doc that maps code to the spec by section (`cas_publish.rs:1-49` walks §Push steps 1–8; `hydrate.rs:1-27` walks §Read).
- Invariant names from the TLA+ model (`Inv_NoFork`, `Inv_Closed`, `Inv_RefEffectApplied`, `Inv_RefDerivedFromParent`) are cited inline at the code that establishes them (`cas_publish.rs:200-213`, `:897-906`; `transport.rs:832-856`).
- Negative documentation is a first-class pattern: "What this function deliberately does *not* do" (`cas_publish.rs:34-49`), the `SECURITY:` note explaining why method binding is tautological (`transport.rs:174-183`), the "no advisory lock — by design" block (`transport.rs:865-877`).
- Named-reviewer attribution appears in production comments ("Max's blocker", "Eva's call, on record in #proj-git-on-s3", "Sami #2 / Max / Dawn") — `transport.rs:530-537`, `:875-877`; `cas_publish.rs:1171-1176`. This ages badly and encodes decisions in people's names rather than in the argument.
- All public items carry doc comments, satisfying the AGENTS.md rule.
- Two module doc comments are **stale**: `store.rs:25` ("wired in … in a follow-up commit") and `hydrate.rs:24-30` (describes an `#[allow(dead_code)]` that no longer exists).

#### 8. Test conventions

Totals: **99 unit tests** in-module (76 `#[test]`, 23 `#[tokio::test]`), **0** `#[ignore]`d. Plus 2 `#[ignore]`d live E2E tests in `crates/buzz-test-client/tests/e2e_git.rs:195-475`.

| File | `#[test]` | `#[tokio::test]` | Notes |
|---|---|---|---|
| `manifest.rs` | 23 | 0 | schema + predicates + byte-pinning |
| `transport.rs` | 20 | 7 | gzip decode, pkt-line framing, report-status de-framing, NIP-98 host binding, fast-path eligibility |
| `cas_publish.rs` | 16 | 2 | pure composition + 2 that shell out to real `git` |
| `policy.rs` | 14 | 0 | 12 HMAC unit + 2 cross-language bash/Rust |
| `manifest_event.rs` | 9 | 0 | tag shape and filtering |
| `store.rs` | 6 | 4 | 2 pure (`classify_cas`) + 2 config + 4 live-MinIO probes |
| `pack_cache.rs` | 5 | 3 | eviction, symlink guard, flight coalescing |
| `hydrate.rs` | 2 | 6 | predicates + 4 live-MinIO roundtrips |
| `hook.rs` | 1 | 0 | Dockerfile parse assertion |
| `mod.rs` | 0 | 0 | **`require_localhost` is untested** |

Conventions:

- **Environment-gated live tests instead of `#[ignore]`.** `probe_enabled()` reads `BUZZ_GIT_S3_PROBE == "1"` and returns early otherwise, so the tests always "pass" in CI. Duplicated in three files (`store.rs:996-998`, `hydrate.rs:579-581`, `cas_publish.rs:1578-1580`). Cost: a silently skipped test is indistinguishable from a passing one in CI output — only `store.rs:1020-1022` prints a skip notice.
- **Named mutation-bite tests.** Tests are written to fail under a specific regression and say so: `git_nip98_rejects_token_signed_for_wrong_community_host` (`transport.rs:2101-2122`), `same_owner_repo_pointers_do_not_bleed_between_communities` (`manifest.rs:505-524`), `validate_invoked_between_compose_and_put_manifest` (`cas_publish.rs:1833-1878`).
- **Byte-pinning.** Canonical manifest bytes (`manifest.rs:544-568`, `:367-385`) and pkt-line framing against a stated git-2.51 oracle (`transport.rs:2239-2277`) are asserted literally.
- **Cross-language verification.** The bash HMAC is re-implemented inside the test and diffed against Rust (`policy.rs:592-773`) — the only test of the module's most security-critical contract, and it depends on `bash` + `openssl` being present.
- **Real-`git` helpers.** `run_test_git` asserts success and applies `harden_git_env` (`cas_publish.rs:1530-1541`); `build_source_repo` constructs a real repo and extracts its pack (`hydrate.rs:595-640`).
- **Test-only accessors** are gated: `GitPackCache::flight` is `#[cfg(test)]` (`pack_cache.rs:401-418`).
- Test module names are mostly `tests`, except `transport.rs`'s `track_c_tests` (`:1768`) and `store.rs`'s second module `probe` (`:984`), which document *why* they exist.

#### 9. Naming and style conventions

| Convention | Example |
|---|---|
| Spec vocabulary in identifiers | `ParentState`, `CasSuccess`, `CasOutcome`, `Precond`, `PublishLimits` |
| `m_before` / `m_after` mirroring the spec | `cas_publish.rs:1105`, `:1177` |
| Predicate functions named `is_*` | `is_safe_refname`, `is_hex_oid`, `is_pack_key`, `is_manifest_digest`, `is_emittable_ref`, `is_valid_oid`, `fast_path_eligible`, `should_compact`, `compacted_pack_set_is_usable` |
| `*_inner` for the metric-wrapped body | `hydrate_for_read` → `hydrate_for_read_inner` (`hydrate.rs:124-168`); `cas_publish` → `cas_publish_inner` (`:997-1021`) |
| Limit constants `SCREAMING_SNAKE` with a doc paragraph explaining the number | `transport.rs:42-59`, `cas_publish.rs:81-86`, `manifest.rs:34-47` |
| `_`-prefixed fields that exist only for lifetime | `_tempdir`, `_repo`, `_permit`, `_temporary`, `_session_dir` |
| `#[derive(Debug, Clone, Copy)]` on small option structs | `PublishLimits`, `PublishOptions`, `HydrationOptions` |
| Options bundled into a struct rather than long argument lists | `HydrationOptions` (`hydrate.rs:79-89`), `PublishLimits` (`cas_publish.rs:147-155`) |

#### 10. Convention violations and inconsistencies

| # | Issue | Site |
|---|---|---|
| 1 | 17 production `unwrap()`/`expect()` against the AGENTS.md rule (all structurally infallible, but the rule is absolute) | §3 above |
| 2 | Two distinct git-env hardening implementations that claim to match but do not (`GIT_CONFIG_GLOBAL` missing in one) | `transport.rs:294-310` vs `hydrate.rs:451-465` |
| 3 | `read_log_prefix` and `read_prefix` are the same 14-line function in two files | `transport.rs:1141-1155`, `cas_publish.rs:877-891` |
| 4 | `probe_enabled()` triplicated | `store.rs:996`, `hydrate.rs:579`, `cas_publish.rs:1578` |
| 5 | `tenant()` test helper duplicated three ways | `hydrate.rs:536`, `cas_publish.rs:1603`, `transport.rs:1902` |
| 6 | Path-to-`&str` handled inconsistently: `.unwrap()` in one place, a typed error two functions away | `cas_publish.rs:410` vs `:702-705` |
| 7 | Module-wide `#![allow(dead_code)]` in `store.rs` will hide any future genuinely-dead item | `store.rs:25` |
| 8 | `harden_git_env` lives in the HTTP transport module but is consumed by two storage layers | `transport.rs:302` |
| 9 | `stream_git_read` carries an unused `extra_args` parameter | `transport.rs:1418`, called with `&[]` at `:824` |
| 10 | `transport.rs` at 2 288 lines mixes auth extractor, pkt-line codec, three handlers, two subprocess runners, four stream adapters, and the fence | whole file |


## Module: buzz-relay — moderation, admin & background workers (`crates/buzz-relay/src`)
### Aspect: Conventions

---

#### 1. Handler signature shapes

Three distinct shapes coexist across 6 event handlers. Argument order is **not** consistent.

| Handler | Signature | Site |
|---|---|---|
| `moderation_commands::handle_moderation_command` | `(tenant, state, event) -> Result<(), String>` | `:91-95` |
| `relay_admin::handle_relay_admin_event` | `(tenant, state, event) -> Result<(), String>` | `:108-112` |
| `identity_archive::handle_identity_archive_event` | `(tenant, state, event) -> Result<(), String>` | `:40-44` |
| `report::handle_report_event` | `(tenant, event, state) -> Result<(), String>` — **state last** | `:44-48` |
| `product_feedback::handle` | `(tenant, event, state) -> Result<(), String>` — **state last** | `:19-23` |
| `push_lease::accept` | `(tenant, state, event, now: i64) -> Result<AcceptLeaseOutcome, AcceptError>` — **typed error, typed outcome, injected clock** | `:469-474` |

`push_lease::accept` is the only handler with a typed error and the only one taking an injected `now`. The other five call `SystemTime::now()` internally, making their freshness checks untestable without wall-clock manipulation.

Naming is also inconsistent: four handlers use `handle_<domain>_event`, `moderation_commands` uses `handle_<domain>_command`, `product_feedback` uses the bare `handle`.

`state` is always `&Arc<AppState>` except in `push_runtime`, which takes `&AppState` for helper functions (`push_runtime.rs:98`, `:125`, `:349`, `:531`) and `Arc<AppState>` by value for the two spawned loops (`:57`, `:312`).

---

#### 2. Error-string conventions

##### 2.1 Three competing prefix strategies

| Strategy | Handlers | Effect at the wire |
|---|---|---|
| **Self-prefixing** with `invalid:` / `error:` / `restricted:` | `moderation_commands` (via helpers `:548-558`), `report` (inline literals), `product_feedback` (inline literals) | ingest passes through verbatim |
| **Unprefixed**, ingest wraps as `invalid: {e}` | `relay_admin` (`ingest.rs:1837`), `identity_archive` (`ingest.rs:1943`) | authorization failures surface as `invalid: actor not authorized: …` — semantically wrong |
| **Typed error enum**, ingest maps | `push_lease` → `AcceptError::{Validation, Internal}` (`ingest.rs:187-195`) | correct 400/500 split |

`moderation_commands` is the only handler with named prefix helpers and the only one with a **test that pins the prefixes** (`moderation_commands.rs:669-680`):

```rust
fn authz_denial(e: anyhow::Error) -> String { format!("restricted: {e}") }   // :548
fn invalid(message: impl Into<String>) -> String { format!("invalid: {}", …) } // :552
fn error(message: impl Into<String>) -> String { format!("error: {}", …) }     // :556
```

`report.rs` and `product_feedback.rs` inline the same prefixes as string literals at 20+ sites, with no helper and no test pinning them.

##### 2.2 The `blocked:` prefix is unique and unprefixed

`blocked: you are banned from this community` appears at `moderation_commands.rs:139` (returned bare, not via `invalid()`), `:199` (as the disconnect reason), `ingest.rs:1648`, and `handlers/auth.rs:171` — four independent literals of the same string with no shared constant.

##### 2.3 Freshness error text is duplicated verbatim three times

```
"event timestamp out of range: created_at={event_ts}, now={now}, delta={}s (max ±120s)"
```
`moderation_commands.rs:117-120` (interpolating the named const), `relay_admin.rs:126-129` (hard-coded `±120s`), `identity_archive.rs:148-151` (hard-coded `±120s`).

##### 2.4 DB error text reaches clients

`error: database error: {e}` (`moderation_commands.rs:174` and 5 more), `database error: {e}` (`relay_admin.rs:137` and 6 more), `restricted: {e}` where `e` may be a `sqlx` error (`moderation_commands.rs:549` wrapping `moderation_authz.rs:99`). Only `push_lease.rs:572` deliberately opaques it — and in doing so discards the diagnostic entirely.

---

#### 3. Tag-extraction conventions

Four near-duplicate helper families, all private, all reimplemented per file.

| Helper | Copies | Divergence |
|---|---|---|
| `extract_tag_value(event, name) -> Option<String>` | 3 — `moderation_commands.rs:608`, `relay_admin.rs:49`, `identity_archive.rs:189` | bodies are functionally identical; `identity_archive` uses `find_map`, the others use a `for` loop |
| p-tag extraction | 3 — `extract_p_tag_bytes` (`moderation_commands.rs:561`, returns `Vec<u8>`), `extract_p_tag_hex` (`relay_admin.rs:33`, returns `String`), `extract_single_p_tag_hex` (`identity_archive.rs:170`, returns `String` **and rejects a second `p` tag**) | three different contracts for the same tag |
| 64-hex validation | 4 inline copies — `moderation_commands.rs:567`/`:582`, `relay_admin.rs:41`, `identity_archive.rs:178`, plus typed variants `report.rs:211-220` (`decode_32_byte_hex`) and `push_lease.rs:365-374` (`check_exact_hex`, lowercase-only) | `push_lease` is the only one rejecting uppercase hex |
| tag-name matching | two idioms: `tag.as_slice().first().map(\|s\| s.as_str()) == Some("p")` (`moderation_commands.rs:564`, `relay_admin.rs:36`, `identity_archive.rs:173`) vs `tag.kind().to_string() == "imeta"` (`product_feedback.rs:24`, `:81`, `push_runtime.rs:263`) | the second allocates a `String` per tag per call |

Case handling also diverges: `relay_admin.rs:169` and `identity_archive.rs:58` lowercase the target pubkey with `to_ascii_lowercase()`; `moderation_commands.rs:561-574` does not (it accepts mixed-case hex and `hex::decode`s it, so the bytes normalize anyway).

---

#### 4. Logging conventions

##### 4.1 Level usage

| Level | Convention | Examples |
|---|---|---|
| `info!` | one line per successful privileged mutation | `moderation_commands.rs:223` (`"community ban applied"`), `:258`, `:325`, `:362`, `:497`; `relay_admin.rs:164`, `:203-209`, `:268-272`, `:327-332`; `identity_archive.rs:90-97`; `community_provisioning.rs:302-308`, `:336-343`; `workflow_sink.rs:316-321` |
| `warn!` | best-effort side effect failed | `relay_admin.rs:215`, `:218`, `:275`, `:278`, `:335`; `identity_archive.rs:131`, `:135`; `moderation_notices.rs:153`; `community_provisioning.rs:220-226`; `push_runtime.rs:63`, `:130`, `:187`, `:367`, `:386`, `:415`, `:427`, `:464`, `:467` |
| `error!` | loop-level failure that will retry | `push_runtime.rs:65`, `:83`, `:337`; `storage_sweep.rs:176-181`, `:195` |
| `info!` for a *failed* side effect | **inconsistent** — notice-DM failures use `info!`, not `warn!` | `moderation_commands.rs:217`, `:319`, `:493` |

The notice-DM failure at `info!` level is the outlier: a user who was banned and never told is an `info`, while a failed NIP-43 announcement is a `warn`.

##### 4.2 Structured field conventions

Consistent field names across the group: `sender`, `target`, `actor`, `operator`, `community`, `host`, `role`, `new_role`, `kind`, `error`, `wake`, `event_id`, `attempt`, `reaped`, `changed`. `%`-sigil display formatting is used for hex/UUID values (`moderation_commands.rs:223`, `push_runtime.rs:171`).

Notably **absent**: `moderation_commands` logs `target` but never the `actor` on success (`:223`, `:258`, `:325`, `:362`) — `relay_admin` logs both `sender` and `target` (`:203-209`). The moderation success lines are therefore not attributable from logs alone.

##### 4.3 Secret handling in logs

No file logs a pubkey secret, token, or ciphertext. Verified specifics:
- `relay_admin.rs:164` logs `icon_len`, not the icon value.
- `push_runtime.rs` never logs `endpoint_grant`; the closest is `wake=%claimed.id` (a UUID).
- `push_lease.rs` logs nothing at all — zero `tracing` calls in the file.
- `moderation_notices.rs` never logs the notice body.

---

#### 5. Concurrency and background-loop conventions

| Convention | Followed by | Site |
|---|---|---|
| `loop { claim → work → backoff }` with exponential idle backoff capped at 2 s | `push_runtime::run_matcher` (`:57-90`), `run_delivery_worker` (`:312-347`) | both reset the delay on finding work |
| Off-claim-path periodic sweep using `tokio::time::Instant::elapsed()` rather than a second task | `push_runtime.rs:59-68` | rationale at `:26-28` |
| Single-flight via a stored `JoinHandle` + `is_finished()` | `storage_sweep.rs:161-165` | harvest and spawn deliberately share one lock (`:143-149`) |
| Leader election via Postgres advisory lock | `storage_sweep` (through `main.rs:1414-1430`) | **not** followed by `push_runtime`, which runs on every pod |
| `Weak<AppState>` to break `Arc` cycles | `workflow_sink.rs:159-161` | rationale `:150-155` |
| Function-local `OnceLock` for localized feature config | `main.rs:1447-1453` | rationale `:1448-1451` |
| Cross-tick state in `AppState` behind `tokio::sync::Mutex` | `storage_sweep` (`state.rs:561`) | pattern documented at `storage_sweep.rs:128-130` |

Pure/impure separation is applied consistently in the two most-tested files: `decide_authority` is factored out of `authorize_moderation_action` "so it is exhaustively unit-testable" (`moderation_authz.rs:137-139`), `match_job` is documented as "Pure match evaluation: no DB access" (`push_runtime.rs:216-218`), `should_spawn` is a pure cadence predicate (`storage_sweep.rs:105-127`), and `resolve_mention_pubkeys` is a pure function (`workflow_sink.rs:45`).

---

#### 6. Test conventions

##### 6.1 Counts

| File | LOC | Tests | `#[ignore]` | Test-mod start |
|---|---|---|---|---|
| `handlers/moderation_commands.rs` | 768 | 10 | 0 | `:619` |
| `handlers/moderation_notices.rs` | 398 | 4 | 0 | `:310` |
| `handlers/moderation_authz.rs` | 335 | 7 | 0 | `:184` |
| `handlers/relay_admin.rs` | 468 | 15 | 0 | `:348` |
| `handlers/community_provisioning.rs` | 445 | 13 | 0 | `:354` |
| `handlers/push_lease.rs` | 771 | 10 | 0 | `:600` |
| `handlers/identity_archive.rs` | 580 | 6 | 0 | `:360` |
| `handlers/report.rs` | 337 | 6 | 0 | `:231` |
| `handlers/product_feedback.rs` | 161 | 4 | 0 | `:100` |
| `push_runtime.rs` | 656 | **2** | 0 | `:578` |
| `storage_sweep.rs` | 1090 | 15 | 0 | `:360` |
| `workflow_sink.rs` | 711 | 18 | **1** (`:613`) | `:368` + `integration_tests` `:560` |
| **Total** | **6,720** | **110** | **1** | |

Test density is wildly uneven: `storage_sweep` has 15 tests for 1090 LOC (~1 per 73), `push_runtime` has **2** for 656 LOC (~1 per 328) — and one of those two is an HTTP-level test of request-id stability, not of the delivery state machine. `deliver_one`'s 10-branch response handling and `retry_or_fail`'s backoff have zero coverage.

##### 6.2 Test-module structure

Standard: `#[cfg(test)] mod tests { use super::*; … }`. `workflow_sink.rs` is the only file with a second module — `#[cfg(test)] mod integration_tests` (`:560`) with a module-level doc comment naming the commits it regresses and the exact command to run it (`:561-567`).

##### 6.3 Event-builder helper convention

Every event-handling file defines a local `make_*` helper, all slightly different:

| Helper | Signature | Site |
|---|---|---|
| `make_event(kind: u16, created_at_secs: u64, tags: Vec<Vec<String>>)` | includes a timestamp | `moderation_commands.rs:646-657` |
| `make_test_event(kind: u16, tags: Vec<Vec<&'static str>>)` | no timestamp; needs `Box::leak` at call sites (`:391`, `:369`) | `relay_admin.rs:355-365`, `identity_archive.rs:363-373` |
| `report_with_tags(tags: &[&[&str]])` | `&str` slices | `report.rs:234-245` |
| `feedback(tags: Vec<Tag>)` | pre-built `Tag`s | `product_feedback.rs:107-115` |
| `event(tags: Vec<Tag>)` | fixed `created_at = 1000` | `push_lease.rs:604-611` |

The `Box::leak(hex.clone().into_boxed_str())` idiom (`relay_admin.rs:391`, `identity_archive.rs:369`) is a deliberate test-only leak to satisfy the `&'static str` parameter — a smell caused by the helper's signature, not by the test.

##### 6.4 Postgres-gated tests: three different strategies

| Strategy | File | Behaviour without Postgres |
|---|---|---|
| `#[ignore = "requires Postgres"]` | `workflow_sink.rs:613` | **skipped and reported as ignored** — correct |
| Silent `return` on connect failure | `identity_archive.rs:515-527` | **passes green** — three bailouts: `test_pool()` returns `None` (`:517-519`), a probe `SELECT` fails (`:520-526`), `test_state()` returns `None` (`:527-529`) |
| n/a | all others | pure unit tests |

`identity_archive.rs:515` is a false-green: the module's only integration test — and the only coverage of the live-kind:0 revocation rule — silently no-ops in CI without Postgres.

`storage_sweep.rs:381-397` uses a fourth variant: it returns early if any of the four `BUZZ_STORAGE_*` env vars is externally set (`:386-394`), with an honest in-code comment "externally forced — skip rather than assert a lie" (`:392`).

##### 6.5 Cross-artifact invariant tests (the strongest convention here)

Three tests assert Rust constants match non-Rust artifacts:

| Test | Asserts | Site |
|---|---|---|
| `resolve_audit_actions_are_allowed_by_db_check_vocabulary` | every 9044 action maps into `MODERATION_ACTION_CHECK_VOCAB`, with a failure message naming `migrations/0006_moderation.sql` | `moderation_commands.rs:659-667` |
| `migration_trigger_allowlist_matches_advertised_push_kinds` | `include_str!("../../../../migrations/0018_push_match_queue.sql")` contains the literal `NEW.kind IN (7, 9, 1059, 40007, 46010)` | `push_lease.rs:696-710` |
| `command_error_prefix_helpers_preserve_machine_readable_token` | the three wire prefixes | `moderation_commands.rs:669-680` |

The `include_str!` migration test is the most valuable pattern in the group — it makes Rust/SQL drift a compile-adjacent failure. It is applied once.

##### 6.6 Async-test conventions in `storage_sweep`

- `#[tokio::test(start_paused = true)]` for timeout and multi-tick behaviour (`:686`, `:743`).
- `tokio::task::yield_now()` between `maybe_spawn_sweep` calls, with an in-code explanation that harvest happens on the *next* call so two failures need three calls (`:507-513`).
- `async { panic!("must not spawn …") }` as an assertion that a future is never polled — used 5× (`:493`, `:546`, `:671`, `:760`, `:781`), relying on the documented guarantee that an unpolled async value has not started its body (`:151-153`).
- `metrics_util::debugging::DebuggingRecorder` + `metrics::with_local_recorder` + `futures::executor::block_on` to capture gauges (`:652-656`, snapshot helpers `:800-812`, `:820-838`).
- A bounded `for _ in 0..50 { … if finished { break } yield_now() }` poll loop instead of a fixed poll count, with rationale (`:697-711`).

##### 6.7 Documentation-in-tests convention

Test names encode the rule (`admin_cannot_ban_or_timeout_owner_or_fellow_admin`, `lowercase_expansion_does_not_shift_later_mentions`, `a_completed_but_unharvested_sweep_never_emits_its_snapshot`), and several tests carry multi-paragraph rationale comments naming the reviewer or counterexample that motivated them: "Wren's redteam counterexample" (`workflow_sink.rs:490-497`), "Quinn's re-review" (`:526-528`), "Wren's L5 lesson: never a UUID where an event id belongs" (`moderation_commands.rs:717`), "Rev 3 required tests" (`storage_sweep.rs:602-628`).

Two tests document *why a test was dropped* rather than deleting silently: `workflow_sink.rs:526-528` explains the two `ẞ→ss`-premised cases were vacuous because `ẞ` lowercases to one char.

---

#### 7. Documentation conventions

##### 7.1 Module-doc structure

Every file opens with `//!` docs. The moderation files follow a house style unique to this group: a summary table of kinds → operations → side effects, then an explicitly labelled pinned-contract section, then a lane-ownership footer.

| Convention | Examples |
|---|---|
| Markdown table of kinds/permissions in the module doc | `moderation_commands.rs:14-21`, `relay_admin.rs:9-16` |
| `## Routing (pinned — …)` / `## Tag vocabulary (pinned — …)` sections with a date and reviewer | `moderation_commands.rs:23-27`, `:29-55` |
| `Lane ownership: L<n> (<name>)` footer naming the owning engineer | `moderation_commands.rs:57-60`, `moderation_notices.rs:30`, `moderation_authz.rs:21`, `report.rs:23-25`, `buzz-db/src/moderation.rs:15-16` |
| Cross-lane coordination instruction | `moderation_commands.rs:58-60` ("coordinate, don't edit ingest.rs") |
| Reference to a `PLANS/` design doc | `moderation_authz.rs:4-5`, `storage_sweep.rs:5-7`, `:35`, `moderation_notices.rs:3` |
| Reference to a TLA+ spec | `report.rs:11`, `buzz-db/src/moderation.rs:12-13` |
| `## Privacy` invariant section | `moderation_notices.rs:20-23` |
| `DESIGN:` inline marker for a deliberate refusal | `relay_admin.rs:296-299` |
| Named-thread / event-id citation for a pinned decision | `moderation_commands.rs:33` ("thread event `86f46207`"), `workflow_sink.rs:561` ("`e3661764` / `7899c1a8`") |

This is unusually good provenance documentation. The cost is that several pinned claims have drifted from the code (see the api-surface and features aspects: the "reject channel-scoped API tokens" claim at `moderation_commands.rs:50`, the "recorded in the audit row" claim at `moderation_authz.rs:61`, and the "fan out through the existing 9005/9001 + 9040 paths" claim at `moderation_commands.rs:20`).

##### 7.2 Rationale-comment convention

Long inline comments explaining *why*, including accepted tradeoffs stated explicitly rather than hidden. Representative examples:
- accepted residual race, named as tolerated — `moderation_commands.rs:419-425`
- crash-safe but not concurrency-safe, with the follow-up named — `moderation_notices.rs:132-138`
- why discovery is re-emitted unconditionally — `moderation_notices.rs:141-151`
- why `limit: Some(1000)` and not `Some(1)` — `moderation_notices.rs:222-226`
- why the operator allowlist is deployment-root, not create-only, authority — `community_provisioning.rs:236-247`
- why the storage sweep respawns on every tick after a failure — `storage_sweep.rs:89-103`
- why harvest and spawn share one lock — `storage_sweep.rs:143-149`
- why the gateway request id must be stable across retries — `push_runtime.rs:486-490`
- why case folding must run in original-char coordinates — `workflow_sink.rs:78-96`
- why the deliberately non-standard `moderation_source` tag is not an `e` tag — `moderation_notices.rs:35-38`

##### 7.3 Public-API doc coverage

All `pub` items carry doc comments, satisfying the AGENTS.md rule. Verified across `push_lease.rs` (every `pub` struct field documented, `:24-81`), `storage_sweep.rs` (`:34-46`, `:105-127`, `:143-153`, `:258-282`), `moderation_authz.rs` (`:28-69`), `buzz-db/src/moderation.rs` (every field, `:37-170`).

---

#### 8. Rust-hygiene conventions (measured)

| Rule | Count in the 12 files | Detail |
|---|---|---|
| `unsafe` | **0** | none anywhere |
| `unwrap()` in production paths | **0** | all 49 occurrences are inside `#[cfg(test)]` modules |
| `.expect()` in production paths | **7** | `push_lease.rs:534`, `:539`, `:543`, `:548`, `:552` (all justified by prior validation — "validated active endpoint" etc.); `push_runtime.rs:316` (`"push HTTP client"`), `:514` (`"closed delivery body"`) |
| `panic!` in production paths | **0** | 6 occurrences, all test assertions (`storage_sweep.rs:493/546/671/760/781`, `workflow_sink.rs:631`) |
| `todo!` / `unimplemented!` | **0** | — |
| `TODO` / `FIXME` / `HACK` / `XXX` markers | **0** | the two `// TODO (WF-07)` markers live in `buzz-workflow/src/executor.rs:577`, `:582` — outside this module |
| `#[allow(…)]` | **0** in these 12 files | `#[allow(clippy::too_many_arguments)]` appears on the DB functions they call (`buzz-db/src/push.rs:210`, `archived_identities.rs:49`) |
| `unwrap_or(0)` on `SystemTime` | 3 | `moderation_commands.rs:115`, `relay_admin.rs:124`, `identity_archive.rs:146` — fails closed but produces `now=0` in the error string |

The 5 `.expect()` calls in `push_lease.rs:530-556` are a direct consequence of `LeasePlaintext` using `Option<T>` for fields that are mandatory when `active == true`. A `LeasePlaintext::Active { … } | Inactive { … }` enum would make them unrepresentable. This is the module's clearest type-modelling debt against the "no new `expect()` in production paths" rule.

---

#### 9. Deviations from repo-wide conventions (AGENTS.md)

| AGENTS.md rule | Compliance |
|---|---|
| No `unsafe` | ✅ 0 |
| No new `unwrap()`/`expect()` in production paths | ⚠️ 7 `.expect()` (5 in `push_lease.rs`, 2 in `push_runtime.rs`) |
| New public API must have doc comments | ✅ all `pub` items documented |
| Event kinds defined in `buzz-core/src/kind.rs` | ⚠️ `KIND_PUSH_LEASE` is defined **twice** — `buzz-core/src/kind.rs:109` and `push_lease.rs:19`; ingest imports the `push_lease` copy (`ingest.rs:204`, `:450`, `:2156`) while `req.rs:1734` imports the `buzz-core` one |
| Channels scoped by `h` tags, not `e` | ✅ `workflow_sink.rs:262-263`, `moderation_notices.rs:161`; `moderation_notices.rs:35-38` explicitly refuses to abuse `e` for a row UUID |
| Prefer Nostr events over new HTTP endpoints | ✅ 13 of 14 operations are event kinds; the one HTTP endpoint (`/operator/communities`) is justified in-code as necessarily above the tenant fence (`community_provisioning.rs:3-14`) |
| Thread counters (`reply_count`/`descendant_count`) updated by reply inserters | n/a — `workflow_sink` inserts only top-level (`depth: 0`, `workflow_sink.rs:333`) and `moderation_notices` inserts via `insert_event` with no thread metadata (`:174`) |


## Module: buzz-relay — huddle audio, tunnel & conformance seam (`crates/buzz-relay/src`)
### Aspect: Conventions

---

#### 1. Task / loop patterns

##### 1.1 Spawned tasks — exact inventory

| File | `tokio::spawn` sites | Which are production |
|---|---|---|
| `audio/join.rs` | 9 | **1** — the renewer at `join.rs:498` (reached via `attach_signals` at `:674`, `:688`, `:695`). The other 8 are in the test module |
| `audio/handler.rs` | 6 | **4** per connection: `send_loop` (`:663`), `heartbeat_loop` (`:667`), `audio_forward_loop` (`:670`), plus conditionally the owner-control reader (`:704`) and the owner-teardown watcher (`:733`). 1 test spawn |
| `audio/mesh.rs` | 1 | **1** — `spawn_remote_peer_sink` (`mesh.rs:272`), one per remote peer on the owner pod |
| `tunnel/reliable.rs` | 3 | **1** — the lease renewer (`reliable.rs:588`), which no production site invokes. 2 test spawns |
| `mesh_boot.rs` | 3 | **3** — the huddle-control accept task (`:271`), the reliable accept task (`:291`), and the drain watcher (`:485`) |
| `audio/room.rs`, `audio/wire.rs`, `tunnel/directory.rs`, `conformance/*` | 0 | — |

**Per audio connection: 4–6 tasks** (3 unconditional + reader + owner watcher).
At the 25-peer soft cap × N rooms this is the dominant task cost.

##### 1.2 Structured-concurrency discipline in `handler.rs`

Consistently applied: one root `CancellationToken` per connection
(`handler.rs:145`), child tokens for the loops that must die first
(`send_cancel` `:662`, `fwd_cancel` `:669`), and the parent cloned for loops that
must observe global cancellation (`hb_cancel` `:665`, `reader_cancel` `:688`,
`owner_cancel` `:732`). After `recv_loop` returns, `cancel.cancel()` then **every**
task is awaited (`handler.rs:787-800`) before cleanup runs — including the reader
task, so its clean-close reaches the owner before the peer is removed.

Deviation: the huddle-lease renewer is **detached**. `attach_signals` keeps only
`renewer.lost` (`join.rs:694`) and drops the `HuddleLeaseRenewer` struct, so its
`task: JoinHandle<()>` is never awaited. The struct exposes `.task` publicly
(`join.rs:467`) but no production code reads it. The same shape exists in the
reliable lane (`reliable.rs:200-205`).

Deviation: `spawn_remote_peer_sink` (`mesh.rs:272-283`) is fully detached — it
terminates only when its `mpsc::Receiver<Bytes>` closes, i.e. when the owner's
`Room` drops the peer. The teardown path relies on that implicit chain
(documented `join.rs:1341-1344`).

##### 1.3 `select!` conventions

`biased;` is used wherever cancellation or control must win over data:

| Site | Ordering |
|---|---|
| `handler.rs:187-193` | cancel → auth timeout |
| `handler.rs:952-956` | cancel → `ws_recv.next()` |
| `handler.rs:1071-1085` | cancel → control → data |
| `handler.rs:1099-1123` | cancel → peer control → audio |

Not biased: `heartbeat_loop` (`handler.rs:1130-1148`, tick vs. cancel — either order
is safe), the renewer loops (`join.rs:506-533`, `reliable.rs:592-621`), and
`serve_control_loop` (`join.rs:1156-1190`, where drain/lost arms are listed first
so the compiler's random polling still reaches them promptly).

A recurring idiom for an optional `select!` arm — a `pending()` future when the
token is absent, so the arm degenerates instead of the branch being duplicated:

```rust
let lost_fired = async {
    match &lost { Some(t) => t.cancelled().await, None => std::future::pending().await }
};
```

Used at `join.rs:1145-1155`, `join.rs:1167-1172` (roster), and
`handler.rs:735-748`. This is the module's signature pattern.

##### 1.4 Interval conventions

Both renewers set `MissedTickBehavior::Delay` (`join.rs:502`, `reliable.rs:591`) so
a stalled Redis call does not produce a burst of catch-up renewals.
`heartbeat_loop` (`handler.rs:1131`) does **not** set a missed-tick behaviour, so a
long stall can fire several ticks in a row and trip `MAX_MISSED_PONGS` spuriously.
The demo-echo drain poll uses a 100 ms interval (`mesh_boot.rs:315`); the mesh drain
watcher uses `sleep(500ms)` in a loop rather than an interval (`mesh_boot.rs:495`).

---

#### 2. Error handling

##### 2.1 No `unwrap` / `expect` / `panic` in most of the group

Counted over production code only (everything before the file's `#[cfg(test)]` mod):

| File | `unwrap()` | `expect(` | `panic!`/`unreachable!` | `unsafe` |
|---|---|---|---|---|
| `audio/join.rs` (1..1806) | 0 | 0 | 0 | 0 |
| `audio/room.rs` (1..556) | 0 | 0 | 0 | 0 |
| `audio/mesh.rs` (1..284) | 0 | 0 | 0 | 0 |
| `audio/wire.rs` (1..88) | 0 | 0 | 0 | 0 |
| `audio/mod.rs` | 0 | 0 | 0 | 0 |
| `tunnel/mod.rs` | 0 | 0 | 0 | 0 |
| `mesh_boot.rs` (1..521) | 0 | 0 | 0 | 0 |
| `conformance/tracers.rs` | 0 | 0 | 0 | 0 |
| **`audio/handler.rs` (1..1337)** | 0 | **6** | **1** `unreachable!` | 0 |
| **`tunnel/reliable.rs` (1..658)** | 0 | **1** | 0 | 0 |
| **`tunnel/directory.rs` (1..576)** | 0 | **2** | 0 | 0 |
| **`conformance/mod.rs` (1..430)** | 0 | **1** | 0 | 0 |
| **Total** | **0** | **10** | **1** | **0** |

The 10 `expect`s and the `unreachable!`:

| Site | Justification quality |
|---|---|
| `handler.rs:451` `pending_remote.expect("RemoteOwner matched above")` | Sound but fragile — the `if let` at `:448-450` already matched, then the value is re-taken |
| `handler.rs:457` `unreachable!("matched RemoteOwner above")` | Same pattern; a `let-else` re-destructure of a value proven to be `RemoteOwner` |
| `handler.rs:689` `remote_fence.expect("remote_fence set whenever remote_stream is")` | Invariant across three `Option`s set together at `:466-469`; not type-enforced |
| `handler.rs:692`, `:696`, `:701` `remote_session.expect(...)` ×3 | Same invariant, asserted three more times |
| `handler.rs:731` `state.mesh().expect("owner teardown watcher only exists when mesh owner state exists")` | Sound — guarded by `owner_lost.is_some() \|\| owner_draining.is_some()` at `:727` |
| `reliable.rs:471` `bytes[2..18].try_into().expect("16 byte community id slice")` | Infallible given the `len < 18` check at `:462`; could be `TryInto` on a fixed array |
| `directory.rs:261` `current.expect("renewed returns lease")` | Trusts the Lua contract — `renewed` always returns a non-empty value (`directory.rs:47`) |
| `directory.rs:291` `current.expect("released returns lease")` | Same for `released` (`directory.rs:65`) |
| `conformance/mod.rs:246` `row.expect("project_row_community returns None only for Some(ch)")` | Sound; a `match` would remove it |

Four of the six `handler.rs` `expect`s exist because `remote_session`,
`remote_stream`, and `remote_fence` are three parallel `Option`s that are always
`Some` together (`handler.rs:445-503`). A single `Option<struct{…}>` would make all
four unnecessary — this is the clearest local refactor available.

Repo rule from `AGENTS.md`: "Do not introduce new `unwrap()` or `expect()` in
production paths". The group is compliant on `unwrap()`, non-compliant on `expect()`
in four files.

##### 2.2 Lock-poisoning convention (inconsistent)

`audio/room.rs` uses three different strategies for the same mutex:

| Site | Strategy |
|---|---|
| `mark_ended` `:193` | `if let Ok(mut g)` → else return `false` |
| `clear_ended` `:202` | `if let Ok(mut g)` → else silently do nothing |
| `add_peer` `:229-231` | `.map_err(\|_\| AdmissionError::Ended)` — "poisoned ≈ shutting down" |
| `add_peer_at_index` `:282` | same |
| `remove_peer` `:338-340` | `let Ok(mut g) = … else { return }` — **the peer is never removed and its index leaks** |
| `remove_peer_and_check_ended` `:363` | `.ok()?` — returns `None`, caller treats it as "not ended" |
| `roster_snapshot` `:462` | `.unwrap_or_else(\|e\| e.into_inner())` — the only site that recovers through poisoning |

`conformance/tracers.rs:57-60` also recovers via `into_inner`, with an explicit
rationale (`:56`). Since nothing in the group can panic while holding the guard
(no user code runs inside the critical sections), poisoning is unreachable in
practice — but seven different handlings of one impossible case is noise, and two of
them (`remove_peer`, `clear_ended`) fail silently in a way that leaks state.

##### 2.3 Error type conventions

- Two `thiserror` enums: `DirectoryError` (6 variants, `directory.rs:141-176`) and
  `ReliableStreamError` (10 variants, `reliable.rs:529-568`). Both carry structured
  context (community, session, raw value) rather than strings.
- `ReliableStreamError` is annotated `#[allow(missing_docs)]` (`reliable.rs:531`) —
  the only doc-comment escape hatch in the group, against the `AGENTS.md` rule "New
  public API must have doc comments".
- Every `HuddleDirectory` method flattens `DirectoryError` into
  `MeshError::Transport(e.to_string())` (`join.rs:114`, `:139`, `:158`, `:172`),
  **losing the typed variant** — so a malformed lease and a Redis outage are
  indistinguishable to the huddle join path. Only `validate` preserves typed fence
  errors (`join.rs:180`), which is exactly what `FenceRejection::from_mesh_error`
  needs (`join.rs:996-1005`).
- `ensure_membership` returns `Result<Uuid, String>` (`handler.rs:1153-1158`) —
  stringly-typed, so the caller cannot distinguish "archived", "not linked",
  "not a member", and "db error"; all four collapse into the same WS
  `{"type":"error","message":"not a member"}` (`handler.rs:274-285`).
- `DialError` (`join.rs:1646-1653`) correctly splits a clean owner rejection from a
  transport failure, and the handler maps them to different WS errors
  (`handler.rs:474-503`).

##### 2.4 Fail-closed convention

Applied consistently at tenant and ownership boundaries:

| Decision | Fail-closed choice |
|---|---|
| Unmapped host | 404 with a generic message, never a default tenant (`handler.rs:69-88`) |
| Pre-join `get_channel` DB error | silent teardown, not admission (`handler.rs:404-410`) |
| Owner-ready loop exhaustion | transient error, never an ownerless owner peer (`join.rs:443-447`) |
| Ambiguous channel UUID on the media path | drop, never cross-tenant delivery (`room.rs:526-541`) |
| Missing row-community lookup | `ImplBug`, never substitute the resolved label (`conformance/mod.rs:245-254`) |
| Mesh bind / registry publish failure with mesh on | fatal boot (`mesh_boot.rs:423-463`) |

##### 2.5 Logging conventions

Structured `tracing` with `%`/`?` sigils and `channel_id`/`peer_id`/`session_id`
fields throughout. Production-code counts (excluding test modules):

| File | error | warn | info | debug | trace |
|---|---|---|---|---|---|
| `audio/handler.rs` | 1 | 22 | 7 | 6 | 1 |
| `audio/join.rs` | 0 | 4 | 0 | 4 | 0 |
| `audio/room.rs` | 0 | 1 | 0 | 0 | 0 |
| `audio/mesh.rs` | 0 | 1 | 0 | 3 | 0 |
| `tunnel/reliable.rs` | 0 | 4 | 0 | 0 | 0 |
| `tunnel/directory.rs` | 0 | 3 | 0 | 0 | 0 |
| `mesh_boot.rs` | 0 | 13 | 10 | 0 | 0 |
| **Total** | **1** | **48** | **17** | **13** | **1** |

`error!` is used exactly once, for a genuine invariant violation
(`handler.rs:590-600`), which matches the repo's severity discipline. The one
`trace!` is the per-frame v2 header dump (`handler.rs:996-1003`) — correctly at
`trace` so it is off by default.

Pubkeys are logged in full hex (`handler.rs:255`, `:283`, `:419`, …), consistent with
the rest of the relay and with the rationale in `conformance/mod.rs:66-68`.

---

#### 3. Backpressure conventions

The group has **two deliberately opposite** policies, and they are documented as
such:

| Lane | Policy | Sites |
|---|---|---|
| Realtime audio | **never queue, drop on full** | `try_send` at `room.rs:409`, `:427`, `handler.rs:1115` (`data_tx`), `handler.rs:1043` (Pong) |
| State-bearing control | **never drop; size generously and warn if it happens** | `mpsc::channel(32)` (`room.rs:45`), warning at `room.rs:441-446` |

Detail:

- Per-peer audio queue is 8 slots ≈ 160 ms (`room.rs:38-40`). A slow WS peer loses
  audio but never stalls the room.
- The WS-side data channel is 16 slots (`handler.rs:659`), the WS control channel 8
  (`handler.rs:660`).
- `audio_forward_loop` (`handler.rs:1093-1125`) bridges room→WS using `try_send` on
  **both** channels — so a full 8-slot WS control channel silently drops a
  `joined`/`left` message that the room-level 32-slot channel deliberately protected.
  The room's warning (`room.rs:441-446`) does not fire for this second drop point.
  This is the one inconsistency in an otherwise coherent policy.
- Roster deltas use `tokio::sync::broadcast` (cap 64, `room.rs:179`) with the
  `Lagged` → full-snapshot recovery pattern (`join.rs:1174-1182`), which is the
  correct shape for a lossy-but-recoverable ordered stream.
- Mesh media send is fire-and-forget with `debug!` on error
  (`join.rs:1762-1765`, `mesh.rs:277-282`) — no queue, no retry.
- Reliable-stream sends are `await`ed, so QUIC's own flow control provides
  backpressure (`reliable.rs:283-291`).
- The huddle-control accept task and reliable accept task are `tokio::spawn`ed per
  inbound stream (`mesh_boot.rs:270-274`, `:290-298`) with **no concurrency bound** —
  the dispatcher doc says handlers "must hand off promptly (spawn) rather than
  block" (`mesh_boot.rs:36-37`), and they do, but nothing caps how many.
- The datagram handler runs **inline on the accept task** (`mesh_boot.rs:236-245`),
  justified because `on_media_datagram` is synchronous and non-blocking. Verified:
  it is a `DashMap` lookup plus `try_send`s (`mesh.rs:204-250`).

---

#### 4. Test conventions

##### 4.1 Counts

| File | tests | `#[ignore]`d |
|---|---|---|
| `audio/join.rs` | 34 | 0 |
| `audio/room.rs` | 9 | 0 |
| `tunnel/directory.rs` | 7 | 0 |
| `audio/mesh.rs` | 6 | 0 |
| `tunnel/reliable.rs` | 6 | 0 |
| `mesh_boot.rs` | 5 | 0 |
| `conformance/mod.rs` | 9 | 0 |
| `audio/wire.rs` | 4 | 0 |
| `audio/handler.rs` | 2 | 0 |
| `audio/mod.rs`, `tunnel/mod.rs`, `conformance/tracers.rs` | 0 | 0 |
| **Total** | **82** | **0** |

All 82 are inline `#[cfg(test)] mod tests`. No `tests/` directory exists for this
group; no `#[ignore]` anywhere.

##### 4.2 Silent-skip on missing Redis — the dominant convention

`redis_directory_if_available()` pings Redis and returns `Option<SessionDirectory>`;
every integration test opens with `let Some(directory) = … else { return; }`:

- `tunnel/directory.rs:592-604`, used by 5 tests (`:650`, `:712`, `:779`, `:882`)
- `tunnel/reliable.rs:707-719`, used by 4 tests
- `api/mesh_demo.rs:169-179`, used by 2 tests

These tests **pass vacuously** when Redis is absent. `just test-unit` therefore
reports green while never executing the Lua scripts, generation monotonicity,
or the fence taxonomy. Only `just test` (Postgres + Redis) exercises them. An
`#[ignore]` + explicit opt-in, or a hard failure when `REDIS_URL` is set,
would make the gap visible.

##### 4.3 Test doubles

| Double | Where | Covers |
|---|---|---|
| `FakeDir` | `join.rs:1821-1900` | scripted `HuddleDirectory`: queued `owner_of`/`acquire`/`validate` results, a `VecDeque` of renew outcomes, and call counters. This is the reason 34 tests run without Redis |
| `ChanSend`/`ChanRecv` + `stream_pair()` | `join.rs:2088-2110` | an in-memory `MeshStream` pair over `mpsc::unbounded_channel`, built through the **public** `MeshStream::new` seam — drives `accept_inbound` end-to-end without iroh |
| `NullTransport` | `join.rs:2064-2079` | no-op `send_datagram`, erroring `open_session_stream` |
| `NoopTransport` / `DirectTransport` | `reliable.rs:734-757`, `:824-848`; `api/mesh_demo.rs:181-224` | `unreachable!()` on unexpected calls; `DirectTransport` wraps a real `MeshPeer` |
| `StubSend`/`StubRecv` | `mesh_boot.rs:552-568` | minimal stream for dispatcher routing tests |
| `VecTracer` | `conformance/mod.rs:441-450` | collects `TraceStep`s so `EmitGuard` Drop behaviour is observable |
| `install_for_test` | `join.rs:782-795` | `#[cfg(test)]`-gated registry entry with a caller-supplied `lost` token and **no renewer**, isolating fan-out from renewer timing |

`await_release_calls` (`join.rs:2440-2456`) polls a counter with a 2 s timeout
because the registry owns the renewer task and exposes no `JoinHandle` — a
sound workaround, and the comment says so.

##### 4.4 Naming and documentation convention

Test names are full sentences describing the invariant:
`admit_full_wins_over_version_mismatch`, `registry_release_is_generation_fenced`,
`owner_ready_waits_for_winner_install_then_reuses`,
`parse_clamps_out_of_range_level_keeps_frame`,
`wired_datagram_consumer_shares_the_handle_fence`. Most carry a doc comment
explaining *why* the invariant matters, several citing the reviewer who asked for
it (`room.rs:759-766` "Per Sami/Perci's review", `room.rs:663-665` "Per Max's
review checklist", `mesh_boot.rs:543-545` "Blocker fix (Wren review of 8b077fdb)").

`mesh_boot.rs:546-556` shows a good convention: rather than asserting a lie when the
environment is forced, the test skips —
`if std::env::var("BUZZ_MESH").is_ok() { return; }` with the comment
"externally forced — skip rather than assert a lie".

##### 4.5 Testing gaps (structural, not stylistic)

- `audio/handler.rs` has 1,337 production lines and **2 tests**, both covering
  peripheral concerns (semaphore budget `:1341-1358`, parser size limit
  `:1417-1427`). The 719-line `handle_active_audio_connection` — every WS error
  code, the whole join sequence, teardown ordering — is untested at unit level.
- No test asserts the emitted **JSON shape** of any WS frame. The `code`/`message`
  strings the desktop client branches on (`desktop/src-tauri/src/huddle/relay_api.rs:130-150`)
  are unpinned on the relay side.
- No test covers `emit_participant_event`'s four-step pipeline, including the
  duplicate-skip and insert-error-but-fan-out-anyway branches
  (`handler.rs:1285-1307`).
- `conformance/tracers.rs` has **zero tests** — `JsonlTracer`'s truncate-on-create,
  serialization, and poison recovery are unverified (and it has no callers either).
- No test exercises `wire_mesh_consumers`' huddle-control or reliable-stream arms;
  only the datagram arm is covered (`mesh_boot.rs:665-737`).

---

#### 5. Documentation conventions

Module-level docs are unusually rich and carry ASCII flow diagrams
(`audio/mod.rs:6-9`, `handler.rs:3-12`, `room.rs:3-6`) plus explicit
"why not the other way" reasoning: `mesh.rs:26-35` (the payload invariant),
`join.rs:22-38` (Redis is the arbiter, mesh is a hint), `room.rs:216-227`
(error precedence as an information-leak defence), `wire.rs:12-20` (threat-model
invariant). `mesh_boot.rs:404-410` even argues with itself in prose about whether a
misconfigured mesh should be fatal.

Two conventions worth naming:

1. **Invariants are pinned by a named test, and the doc comment cites it.**
   `room.rs:124-127` → "See `version_pin_persists_across_peer_churn` for the test
   that pins this behavior"; `mesh_boot.rs:667-673` → the load-bearing shared-fence
   test.
2. **Deliberate non-features are documented as such**, so a reader does not
   mistake absence for oversight: `join.rs:1281-1284` (why `UnregisterPeer` needs no
   fence), `conformance/mod.rs:128-146` (why `claimed_community` is `None` on the
   read path), `mesh_boot.rs:206-215` (why no renewer is wired).

Stale docs found: `conformance/mod.rs:46-48` says the `req.rs`/`event.rs` wire
points are "held back as additive patch" when `req.rs:144`/`:355`/`:671` have
landed; `conformance/mod.rs:51-53` claims a blake3 actor label that the code does
not compute; `room.rs:33` lists a `speakers` control message that is never emitted;
`mesh.rs:1-10` opens with "Today a huddle's audio only fans out within a single
pod … This module removes that wall", present tense on both sides of a change that
has already shipped.


## Module: buzz-relay-mesh (`crates/buzz-relay-mesh`)
### Aspect: Conventions

---

#### 1. Module layout and ownership annotation

Eight flat modules, no submodule nesting (`lib.rs:21-28`). The crate carries an
explicit **lane-ownership map in code** (`lib.rs:30-36`):

```
endpoint.rs, peer.rs                              — Mari (transport core)
registry.rs, gossip.rs, membership.rs, status.rs  — Max (membership + /_mesh)
```
with a note that the session directory and tunnel routing live relay-side (Perci)
and huddle fan-out in `buzz-relay`'s audio module (Dawn). This is an unusual
convention for this repo — no other crate embeds per-file human owners in source.
There is **no corresponding `.github/CODEOWNERS` entry** (verified: no mesh path in
CODEOWNERS), so the annotation is advisory only.

Each module opens with a `//!` header stating its contract and its *limits*, and the
limit is always the same one: "this cannot elect owners." See `membership.rs:1-6`,
`gossip.rs:1-5`, `registry.rs:1-7`, `runtime.rs:21-23`. That repetition is the
crate's dominant documentation convention.

#### 2. Error handling

- **One error enum, `thiserror`-derived.** `MeshError` (`lib.rs:65-127`), 16 variants,
  `#[derive(Debug, thiserror::Error)]`. No `anyhow` inside the crate — `anyhow`
  appears only at the consumer boundary (`mesh_boot.rs:412`).
- **Typed variants for every fence-visible reject.** Explicit policy at
  `lib.rs:102-109`: "every fence-visible reject is a typed variant, never a generic
  `Transport`, so live kill-9 / partition / replay evidence is unambiguous."
  Honoured for the four fence variants; *not* honoured for the rest — see below.
- **`MeshError::Transport(String)` is the catch-all and it is overused.** 22
  construction sites collapse ~8 distinct causes into one untyped string:
  iroh bind/connect/stream/datagram errors (`endpoint.rs:38,39,79,91,101`;
  `peer.rs:82,95,113,123,151,155,163,174,187`), attestation parse and verify
  failures (`registry.rs:56-83`), key/pubkey mismatch (`registry.rs:141`), JSON
  encode (`registry.rs:186`), Redis pool acquisition (`registry.rs:271`), missing
  datagram support (`peer.rs:109`), and unknown gossip payload version
  (`gossip.rs:72`). The last one is notable: the *outer* frame version gets a typed
  `UnknownWireVersion` (`wire.rs:185`) while the *inner* gossip version gets a
  string, so the two version channels are not observably symmetric.
- **`err.to_string()` discards iroh's structured errors** at all 12 transport sites,
  so callers cannot distinguish timeout from refused from TLS failure.
- **`#[from]` used exactly once**, for `redis::RedisError` (`lib.rs:126`); everything
  else is explicit `map_err`.
- **Errors are propagated, never swallowed silently** — with two deliberate
  exceptions, both commented: `try_send` on the gossip queue
  (`runtime.rs:556`, "dropping a frame under backpressure is strictly better than
  blocking a recv loop") and `encode_message(&delta)` failures being skipped
  (`runtime.rs:540`, `if let Ok(payload)`).

##### `unwrap` / `expect` policy

AGENTS.md forbids new `unwrap()`/`expect()` in production paths. **22 `expect()`
outside `#[cfg(test)]`, zero `unwrap()`:**

| File | Count | All of the form |
|---|---|---|
| `runtime.rs` | 13 (`:142,156,159,168,183,197,202,222,270,349,444,553,573`) | `.expect("peer lock poisoned")` / `"loop lock poisoned"` / `"handler lock poisoned"` |
| `membership.rs` | 9 (`:74,126,159,173,190,199,322,332,363`) | `.expect("membership lock poisoned")` / `"local record lock poisoned"` |

Every one is a `std::sync` lock-poison unwrap — the conventional accepted case (a
poisoned mutex means another thread already panicked while holding it). Still a
literal deviation from the stated rule, and it means a panic anywhere inside a
membership or peer critical section escalates to a panic in **every** subsequent
mesh operation, including the `/_mesh` handler. `parking_lot` or explicit
`unwrap_or_else(|e| e.into_inner())` recovery would remove the escalation.

`0` `unsafe` (verified: grep for `unsafe` in `src/` returns nothing) — compliant.
`0` `TODO`/`FIXME`/`XXX`/`HACK`/`todo!`/`unimplemented!` markers (verified). All
known gaps are expressed as prose in doc comments instead, which makes them
invisible to grep-based debt tooling.

#### 3. Concurrency and task patterns

**Nine `tokio::spawn` sites in production code**, all in `runtime.rs`:

| # | Task | Site | Lifetime |
|---|---|---|---|
| 1 | `accept_loop` | `:119` | tracked in `MeshRuntime.loops` |
| 2 | `reconcile_loop` | `:120` | tracked in `loops` |
| 3 | `gossip_tick_loop` | `:121` | tracked in `loops` |
| 4 | `datagram_recv_loop` (per peer) | `:233` | tracked in `PeerEntry.tasks` |
| 5 | `stream_accept_loop` (per peer) | `:234` | tracked in `PeerEntry.tasks` |
| 6 | `open_control_stream` (per dialed peer) | `:240` | tracked in `PeerEntry.tasks` |
| 7 | `control_stream_exchange` (accept side) | `:449` | **untracked — detached** |
| 8 | control-stream send pump | `:509` | held locally, aborted on recv-loop exit (`:549`) |
| 9 | registry heartbeat | `:599` | `JoinHandle` returned and **dropped** by the caller (`mesh_boot.rs:467`) |

Two more spawns exist in tests (`endpoint.rs:162`, `:198`).

Patterns and their gaps:

- **`JoinHandle` bookkeeping + explicit abort** is the intended discipline
  (`PeerEntry::abort`, `runtime.rs:56-62`; `MeshRuntime::shutdown`, `:155-164`).
  Broken in two places: task #7 is spawned without being pushed onto
  `PeerEntry.tasks`, so a peer removal aborts the recv loops but leaves the accept
  side's control exchange running until its stream errors; task #9's handle is
  discarded at the call site so the heartbeat can never be stopped.
- **`shutdown()` is never called in production** (`-api-surface.md` §7) — every loop
  above runs until process exit.
- **Infinite `loop { … sleep }` with no jitter and no backoff**: `reconcile_loop`
  (`:285-293`), `gossip_tick_loop` (`:563-587`), heartbeat (`:600-606`), and the
  relay-side drain watcher (`mesh_boot.rs:484-496`). All pods started together will
  gossip and rescan in lockstep.
- **Lock hygiene is careful about await points.** `send_datagram` explicitly
  `drop(peers)` before touching membership (`runtime.rs:172-173`);
  `open_session_stream` clones the `MeshPeer` out of a scoped read guard before
  awaiting (`:182-189`); `install_peer` drops the write guard before
  `mark_connection_state` (`:249-252`). No `std::sync` guard is ever held across an
  `.await` — verified by reading all guard scopes.
- **Two lock kinds by intent**: `RwLock` for read-mostly tables
  (`Inner.peers` `:69`, membership's two `:31-32`), `Mutex` for the write-once
  handler slot (`:70`) and the loop vector (`:79`). `Arc<AtomicU64>`/`AtomicBool` for
  counters and the draining flag (`membership.rs:33-35`), all `Ordering::Relaxed`
  (`membership.rs:181,286,309,310`; `peer.rs:29-34`) — appropriate for pure counters.
- **Bounded channels only.** `mpsc::channel(CONTROL_QUEUE_DEPTH = 64)`
  (`runtime.rs:46`, `:237`, `:442`) with `try_send` drop-on-full (`:556`). No
  unbounded channel anywhere.
- **Handles are cheap-clone `Arc` facades**: `MeshRuntime` (`:77-80`),
  `MeshMembership` (`:29-43`), `MeshPeer` (`peer.rs:38-43`), `ReadyRegistry`
  (`registry.rs:160-163`) all `#[derive(Clone)]` over shared inner state. Documented
  at `runtime.rs:75-76`.

#### 4. Wire-compatibility discipline

Codified in `wire.rs:1-32`:

- `wire.rs` is declared a **FROZEN surface**; "changes here require a post in the
  mesh thread **before** the edit" (`wire.rs:10-13`), with the stated failure mode
  being two lanes compiling against different frame layouts. Unenforced by tooling.
- **ALPN carries the version** — `buzz/mesh/1` (`wire.rs:37`) — so a version bump
  gets a new ALPN and "old and new pods never half-speak to each other during a
  rolling deploy" (`wire.rs:34-36`). This is the primary compatibility mechanism.
- **Version byte first, reject unknown loudly** (`wire.rs:38-41`, enforced
  `wire.rs:183-186`). "Receivers MUST reject unknown versions loudly (count it, log
  it) rather than guessing" — the *log* happens at the call site
  (`runtime.rs:406`, `:549`); the *count* does not exist.
- **Nested opacity for evolution**: gossip rides as `Vec<u8>` inside
  `MeshStreamFrame::Gossip` with its own in-struct version so the gossip lane can
  evolve without a wire bump (`wire.rs:139-141`, `gossip.rs:13`). Same trick for
  huddle control on the consumer side (`audio/join.rs:797-801`).
- **Documented invariants are stated as MUSTs in prose**: "first frame MUST be
  `Hello`" (`wire.rs:29-31`), "senders MUST check the encoded size … never truncate"
  (`wire.rs:21-22`), "receivers MUST reject stale generations at every hop"
  (`wire.rs:22-24`).
- **No `#[non_exhaustive]` on any public enum** (verified: 0 occurrences). The
  convention is "bump the ALPN," not "tolerate unknown variants" — internally
  consistent but leaves zero forward compatibility within a version.
- **Header-size budget pinned by test**, not by a const:
  `datagram_header_overhead_within_budget` asserts ≤64 B so it "can't silently grow
  past the budget" (`wire.rs:266-284`). Nice pattern; note it is one of the 32 tests
  CI never runs.

#### 5. Naming and API-shape conventions

- **`Mesh*` prefix** for crate-owned types (`MeshEndpoint`, `MeshPeer`, `MeshRuntime`,
  `MeshStream`, `MeshDatagram`, `MeshStreamFrame`, `MeshStatus`, `MeshError`,
  `MeshConfig`, `MeshMembership`). **`Relay*` prefix** for the two consumer-facing
  seam traits (`RelayMeshMembership`, `RelayPeerTransport`, `lib.rs:144`, `:158`) —
  the prefix marks "this is the relay's view," which is a genuinely useful signal.
- **Builder-by-`with_`** on `MeshMembership` (`with_expected_relay_pubkey`,
  `with_phi_suspect_threshold`, `membership.rs:61`, `:66`), consuming `mut self`.
- **`record_*` for counter mutation** (6 methods, `membership.rs:249-283`) all funnel
  through one private `update_peer_counters` (`:315-326`) using `saturating_add`.
- **`*_now` for "do the periodic thing immediately"** — `reconcile_now`
  (`runtime.rs:150`), used as the boot fast-path.
- **`*_with_*` for the test/explicit variant of a production default**:
  `bind_with_secret_key` (`endpoint.rs:26`), `start_with_intervals`
  (`runtime.rs:102`), each documented as "production should use the plain one"
  (`endpoint.rs:23-25`, `runtime.rs:84-87`).
- **Time as `u64` millis on the wire, `SystemTime`/`Duration` in memory**, with
  explicit converters `now_millis` (`gossip.rs:223-229`) and
  `system_time_from_millis` (`gossip.rs:231-233`). `now_millis` clamps to
  `u64::MAX` and uses `unwrap_or_default()` on a pre-epoch clock (`gossip.rs:225-228`)
  — no panic path.
- **Addresses as `String` at the model boundary** (`GossipRecord.endpoint_addrs`,
  `ReadyRecord.endpoint_addrs`) explicitly "so this layer does not depend on
  transport internals" (`registry.rs:105-106`), parsed lazily at dial time
  (`runtime.rs:328-336`). Trades type safety for layer independence, and moves
  parse failures from boot to runtime.
- **Boxed futures instead of `async_trait`** (`BoxFuture`, `lib.rs:141`), public
  precisely so out-of-crate implementors can name it (`lib.rs:138-140`).
- **Concrete type over trait where lanes must share an implementation**:
  `MeshStream` is a struct, not a trait, "so lanes share one framing
  implementation" (`lib.rs:183`).

#### 6. Logging conventions

28 `tracing` sites. Consistent shape: structured fields first, then a
`"mesh: <event>"` message.

- Prefix is `mesh:` in `runtime.rs` (`:246`, `:265`, `:272`, `:279`, `:290`, `:317`,
  `:331`, `:342`, `:360`, `:380`, `:404`, `:409`, `:548`, `:602`) and
  `"mesh membership …"` / `"mesh ready registry …"` in the membership/registry lanes
  (`membership.rs:96`, `:105`; `registry.rs:236`, `:241`, `:246`).
- `peer = %runtime_id` (full 64-hex via `Display`) is the standard correlation field.
- Level discipline: `info!` for lifecycle (peer connected/disconnected `:255`, `:280`;
  endpoint closed `:272`), `warn!` for rejected/malformed input and dial failures,
  `debug!` for expected loop termination (`:360`, `:380`, `:548`).
- Rejections log enough to diagnose without leaking secrets — e.g. the
  foreign-relay warn logs `record_relay_pubkey` and `anchored`
  (`membership.rs:96-101`), never a private key.
- Gap: no rate limiting on `warn!`. `"rejected inbound connection from unattested
  runtime id"` (`runtime.rs:265-269`) and the dial-failure warn
  (`runtime.rs:342-346`, every 5 s per dead peer) are both attacker- or
  drift-triggerable log floods.

#### 7. Test conventions

32 tests, `#[cfg(test)] mod tests` at the bottom of 6 of 9 files; **no `tests/`
directory**, no integration-test target. All 32 pass; **0 `#[ignore]`d** —
notable, since this repo uses `#[ignore]` heavily for infra-dependent tests
(`justfile:277-285` explains the buzz-db pattern). Here the Redis-dependent paths
simply have no tests at all rather than ignored ones.

- **Deterministic identities in tests** via `SecretKey::from_bytes(&[n; 32])`
  (`endpoint.rs:157-162`, `runtime.rs:627-631`) and `RuntimeId([byte; 32])` helpers
  named `rid(byte)` — the same helper name is repeated in `membership.rs:394`,
  `registry.rs:319`, `gossip.rs:240`, `runtime.rs:...`, and even in the consumer
  (`mesh_boot.rs:582`). Convention by copy-paste, not by a shared test-utils module
  (this crate exposes no `test-utils` feature, unlike `buzz-core`).
- **Real transport in unit tests.** Five `endpoint.rs` tests and five `runtime.rs`
  tests stand up genuine loopback iroh endpoints and connect them
  (`endpoint_pair`/`connected_pair`, `endpoint.rs:155-176`; `runtime`/`seed`/
  `connected_pair`, `runtime.rs:626-670`). No mocking of QUIC. Whole suite still runs
  in 0.25 s.
- **Poll-with-timeout instead of sleep-and-hope**: every async assertion is
  `timeout(Duration::from_secs(5), async { loop { … sleep(20ms) } })`
  (`runtime.rs:655-668`, `:724-740`, `:756-768`, `:788-800`). Consistent and correct.
- **Explicit teardown** — every multi-runtime test calls `a.shutdown(); b.shutdown();`
  (`runtime.rs:742-743` etc.), which is the only place `shutdown()` is exercised.
- **Negative tests are first-class**: unknown wire version (`wire.rs:246`),
  oversize datagram (`endpoint.rs:239`), tampered attestation (`registry.rs:348`,
  `:358`; `membership.rs:437`), foreign relay key (`membership.rs:451`), unanchored
  fail-closed (`membership.rs:465`), stale gossip (`membership.rs:474`,
  `gossip.rs:253`), unconnected peer (`runtime.rs:823`).
- **Tests as executable specs for physical budgets**: the Opus-sized loss/order gate
  (`endpoint.rs:256-291`) and the 64-byte header budget (`wire.rs:266-284`).
- `tokio = { features = ["test-util"] }` is declared (`Cargo.toml:29`) but **no test
  uses paused time** (`tokio::time::pause` appears nowhere) — the phi tests hand
  `SystemTime` values in directly instead (`gossip.rs:268-278`), which is why they are
  fast and deterministic.
- Test-only doc comments explain *why* a setup is shaped as it is, e.g.
  `runtime.rs:647-650` explaining that with no registry the acceptor's admission gate
  requires pre-seeded membership "(production gets this from the attested ready
  registry)".

#### 8. Documentation conventions

- Every public item has a doc comment (AGENTS.md rule) — spot-checked across all 9
  files; the only bare items are the `IrohSendHalf`/`IrohRecvHalf` private structs
  (`peer.rs:132-133`).
- Doc comments record **rationale and rejected alternatives**, often naming the
  reviewer: "Wren's contract-review blocker" (`wire.rs:57`), "Wren's chaos-gate
  ruling" (`lib.rs:103`), "Dawn huddle peer_index" (`endpoint.rs:259`). Valuable
  archaeology; also means the source is the only record — none of it is in
  `ARCHITECTURE.md`, which does not mention this crate at all (verified: zero
  `mesh`/`iroh`/`quic` hits in all 827 lines).
- **Six doc comments are now stale or wrong** — see `-business-rules.md` §K for the
  full list (notably `lib.rs:55-56` claiming `BUZZ_MESH` defaults on, `lib.rs:186-188`
  calling the real iroh stream halves "placeholder", and `lib.rs:102-109`
  specifying a metric that does not exist).

#### 9. Deviations from repo-wide conventions

| AGENTS.md / repo convention | This crate |
|---|---|
| No `unsafe` | ✅ zero |
| No new `unwrap()`/`expect()` in production paths | ⚠️ 22 `expect()` (all lock-poison) |
| New public API must have doc comments | ✅ |
| Prefer Nostr events over new HTTP endpoints | n/a (crate has no HTTP surface; the consumer adds `GET /_mesh` and `POST /_mesh/demo/echo`) |
| Event kinds in `buzz-core/src/kind.rs` | n/a — this crate has no `buzz-core` dep and defines its own postcard wire, entirely outside the Nostr event model |
| Channel scoping via `h` tags | n/a — no tenant identifier on the mesh wire at all (`-integrations.md` §4) |
| Crate listed in AGENTS.md repo structure | ❌ absent |
| Crate documented in ARCHITECTURE.md §6 Crate Reference | ❌ absent |
| Unit tests run by `just test-unit` | ❌ absent from the list (`justfile:275-295`) |
| `#[ignore]` for infra-dependent tests | ❌ Redis paths untested rather than ignored |


## Module: buzz-conformance (`crates/buzz-conformance`)
### Aspect: Conventions

### Naming

**Spec-mirroring is the dominant convention.** Every `TraceAction` variant is named after its
TLA+ action, character-for-character:

| Rust variant | TLA+ action | Spec line |
|---|---|---|
| `WriteInsert` (`src/lib.rs:181`) | `WriteInsert(w)` | `docs/spec/MultiTenantRelay.tla:514` |
| `WriteInsertGlobal` (`:195`) | `WriteInsertGlobal(w)` | `:559` |
| `WriteDuplicate` (`:204`) | `WriteDuplicate(w)` | `:606` |
| `SanitizedError` (`:214`) | `SanitizedError(w)` | `:778` |
| `AuthCheck` (`:221`) | `AuthCheck(w)` | `:794` |
| `ReadMessageRows` (`:231`) | `ReadMessageRows(w)` | `:643` |
| `ReadByIdRows` (`:241`) | `ReadByIdRows(w)` | `:681` |
| `ReadHostFeedRows` (`:250`) | `ReadHostFeedRows(w)` | `:703` |

`ImplBug` (`:256`) is the sole variant with no spec counterpart, and the doc comment says so
explicitly (`:193-194`).

**`Inv_*` names appear only in prose, never as Rust identifiers.** The crate references
`Inv_NonInterference` (`src/transitions.rs:53`, `:296`, `src/lib.rs:238`),
`Inv_ReadConfinement` (`src/transitions.rs:54`), `Inv_SanitizedErrors` (`src/lib.rs:125`),
`Inv_AdmissionFence` (`src/transitions.rs:223`) — all inside `///` or `//` comments. No
function or type is named after an invariant; the mapping from invariant to enforcing predicate
lives in comments, not in the type system.

**Mutation IDs (`M1`…`M8`) are a second comment-only vocabulary.** Used at `src/lib.rs:127`
("M6 mutation"), `:190` ("M2/M8 target"), `:238` ("M1/M4/M7"), `src/transitions.rs:218-221`,
`:239-240`, `crates/buzz-relay/src/conformance/mod.rs:18-19`,
`crates/buzz-relay/src/handlers/ingest.rs:1779`. No file in the repo defines what M1–M8 are —
grep for `M1 ` / `M4` / `M7` outside these comment sites finds no legend. The identifiers are
inherited from an external mutation-testing plan ("skill-runtime-formal-compliance", cited at
`Cargo.toml:7`, `src/lib.rs:6`, `src/checker.rs:9`, `tests/proptest_checker.rs:5-6`) that is
not in the repo.

**Label newtypes use a `*Label` suffix** — `CommunityLabel`, `HostLabel`, `ChannelLabel`,
`ActorLabel` (`src/lib.rs:66`, `:100`, `:106`, `:112`) — except `OpaqueId` (`:93`), which
breaks the pattern despite being the same kind of thing.

**Tracer impls use a `*Tracer` suffix** and are named for their sink: `NoopTracer`
(`src/lib.rs:323` and again `crates/buzz-relay/src/conformance/tracers.rs:16`), `JsonlTracer`
(`tracers.rs:30`), `CountingTracer` (`conformance/mod.rs:356`, private), `VecTracer` — declared
twice, independently, as a test-local sink (`conformance/mod.rs:447-456` and
`handlers/ingest.rs:2519-2528`) rather than shared through a test-support module.

**Emitter helpers are `record_*` or `emit*`.** `record_req_authcheck` (`mod.rs:148`),
`record_read_message_rows` (`:265`), `record_read_by_id_rows` (`:300`) all end in the action
they emit; `emit` (`:127`) and `emit_product_feedback_success`
(`handlers/ingest.rs:133`) are the generic and one-off forms. `step` (`mod.rs:121`) is the
odd one out — a bare noun with no caller.

**Trailing-underscore placeholder.** `Verdict_` (`src/transitions.rs:53-56`) uses a trailing
underscore to avoid colliding with the schema's `Verdict`. It has zero uses; the underscore
suffix also keeps it out of `non_camel_case_types` lint range.

---

### Error handling

- **No panics in library code.** `#![deny(unsafe_code)]` (`src/lib.rs:38`) and no `unwrap()` /
  `expect()` anywhere in `src/` — verified by reading all three source files. The one
  `expect` in the relay-side helper is a documented invariant restatement:
  `row.expect("project_row_community returns None only for Some(ch)")`
  (`conformance/mod.rs:248`).
- **`thiserror` for the single error type.** `TransitionError`
  (`src/transitions.rs:60-102`) derives `thiserror::Error` with `#[error(...)]` format strings
  carrying the step index (`:63`, `:75`, `:85`, `:95`).
- **Human-readable detail, machine-readable variant.** Every variant carries
  `detail: String` built with `format!` (`:146-151`, `:155-160`, `:164-169`, `:236-241`,
  `:278`, `:304-309`). The convention documented at `:58-59` — "the string payload is
  human-readable; mechanical consumers should match on the variant" — means offending values
  are only recoverable by parsing prose.
- **Fail-fast, first-error-wins.** `check_step` returns on the first violation and
  `check_trace` propagates with `?` (`src/checker.rs:109`). The test suites are written around
  this constraint; the discipline is spelled out at `tests/proptest_checker.rs:27-33`.
- **Fail-closed defaults.** Empty trace → `CoverageBreach` (`src/checker.rs:75-82`);
  missing projection lookup → `ImplBug`, never a substituted label
  (`conformance/mod.rs:246-253`, rationale `:203-208`).
- **Observability code never breaks the request.** `JsonlTracer::record` recovers from a
  poisoned mutex via `into_inner()` (`tracers.rs:59-63`) and discards write errors
  (`:66-71`); `req.rs` logs a `warn!` and continues with an empty lookup map on DB failure
  (`:347-353`, `:663-669`). The `Tracer` trait returns `()` (`src/lib.rs:317`), so there is no
  error channel to propagate even if a caller wanted one.

---

### Comment style

Unusually heavy — roughly half the crate is doc prose. Three recurring shapes:

1. **"Why not the obvious thing"** blocks. `Cargo.toml:7-24` (independence rule),
   `src/lib.rs:47-63` (why not `buzz_core::CommunityId`), `src/lib.rs:236-239` (why `Vec` not
   `Set`), `conformance/mod.rs:135-145` (why `claimed_community: None` on REQ).
2. **Spec-line citations** inline with each match arm (`src/transitions.rs:172-186`,
   `:188-191`, `:193-198`, `:200-205`, `:211-227`, `:251-257`, `:266-268`, `:272-276`).
3. **Named-reviewer / thread references.** `conformance/mod.rs:37-38` ("held back as additive
   patch for Eva to apply onto Max's req.rs writes — see thread `c882c9b1…`"),
   `tests/replay_fixtures.rs:19-20` ("Eva's review (thread `06aaf3f7…`)"),
   `tests/replay_fixtures.rs:145-152`, `conformance/mod.rs:170-172` ("the (B)
   projection-strategy guard-rail Eva specified"). These leave the code coupled to
   conversations that are not in the repo, and several are now stale (the req.rs patch has
   landed — `handlers/req.rs:334-361`, `:649-677` — while the comment still says "held back",
   as does `TRACE_SCHEMA.md:137`).

---

### Test organization

| Lane | Location | Convention |
|---|---|---|
| Unit | `src/checker.rs:134-337` (`#[cfg(test)] mod tests`) | one passing case + one bite case per failure mode; tiny helpers `cid`/`ch`/`state`/`step` (`:144-162`) |
| Property | `tests/proptest_checker.rs` | one `proptest!` block (`:191-431`), all 7 cases inside; generators prefixed `arb_*` (`:73-93`, `:115-170`, `:184-189`) |
| Fixture | `tests/replay_fixtures.rs` | typed builder → serialize → byte-compare → re-parse → replay (`assert_fixture_matches`, `:206-235`) |
| Emitter | `crates/buzz-relay/src/conformance/mod.rs:431-726` | in-crate `#[cfg(test)] mod tests` with a local `VecTracer` sink (`:447-456`) |

**Test names encode the assertion, not the target.** `*_bites_*` for expected failures
(`cross_community_row_bites_non_interference` `src/checker.rs:210`,
`auth_allow_with_foreign_claim_bites_m2` `:228`, `impl_bug_action_bites_coverage_breach`
`:290`, `state_flip_bites_state_mismatch` `tests/proptest_checker.rs:354`); `*_is_fine` /
`*_is_ok` / `*_passes` / `*_is_accepted` for expected successes (`:247`, `:172`,
`tests/proptest_checker.rs:199`, `:304`).

**Property tests carry `P<n>` doc-comment IDs** — P1 (`tests/proptest_checker.rs:207`),
P2 (`:195`), P3a (`:269`), P3b (`:299`), P4 (`:325`), P5 (`:351`), P6 (`:401`) — matching the
"invariant properties, NOT a parallel oracle" design note at `:9-25`.

**Fixture regeneration is env-gated, not flag-gated:** `BUZZ_CONFORMANCE_UPDATE=1`
(`tests/replay_fixtures.rs:210`), so a schema change forces a deliberate re-commit rather than
silently rewriting the golden files.

**Deterministic fixture constants.** `community_a`/`community_b`/`channel_in_a`/`channel_in_b`
are hand-picked `Uuid::from_u128` values with mnemonic hex prefixes
(`tests/replay_fixtures.rs:48-62`: `0xAAAA…`, `0xBBBB…`, `0xCAFE…`, `0xDEAD…`); the property
lane uses prefix-tagged pools instead (`0x0c00…` for communities, `0x0ca0…` for channels,
`tests/proptest_checker.rs:53-63`). The rationale — reproducible serialized JSONL — is at
`tests/replay_fixtures.rs:42-46`.

---

### Serde conventions

- Every newtype is `#[serde(transparent)]` (`src/lib.rs:65`, `:92`, `:99`, `:105`, `:111`), so
  the wire form is a bare scalar.
- Both enums use `#[serde(rename_all = "snake_case")]` (`:116`, `:131`).
- `TraceAction` uses internal tagging: `#[serde(tag = "type", rename_all = "snake_case")]`
  (`:178`), so each action object carries a `"type"` discriminant matching
  `TraceAction::kind()` (`:266-277`).
- Field names are snake_case Rust identifiers with no renames, so `schema_version` /
  `state_after` appear verbatim on the wire — which is where `TRACE_SCHEMA.md:37-46` diverges
  from reality (it documents `schema` / `state`).
- JSONL, one `TraceStep` per line, no envelope: writer `tracers.rs:66-71`, test-side
  serializer `tests/replay_fixtures.rs:179-187`, parser `:191-198` (skips blank lines, panics
  with a 1-based line number).

---

### Dead-code tolerance

Four public items have zero callers anywhere in `crates/` and produce no warning because they
are `pub` in a library:

| Item | Line |
|---|---|
| `Verdict_` | `src/transitions.rs:53-56` |
| `action_channel` | `src/transitions.rs:318-330` |
| `TraceAction::is_critical` | `src/lib.rs:283-285` |
| `Scenario::require` | `src/checker.rs:54-57` |

Two more on the relay side: `conformance::step` (`mod.rs:121-123`) and the crate's own
`NoopTracer` (`src/lib.rs:323-327`), shadowed by the relay's copy (`tracers.rs:16-20`). The
convention here is evidently "keep the reserved surface"; none of the six carries a
`#[allow(dead_code)]` or a TODO.


## Module: buzz-acp — harness core & orchestration (`crates/buzz-acp/src`)
### Aspect: Conventions

#### Module layout

`lib.rs:1-15` declares the whole crate: `#![deny(unsafe_code)]` then eleven private modules and one re-export.

```
acp  config  engram_fetch  filter  observer
pool  pool_lifecycle  queue  relay  setup_mode  usage
```

No module is `pub`. Everything crosses seams via `pub(crate)` / `pub` items visible only inside the crate. File sizes (`wc -l crates/buzz-acp/src/*.rs`):

| File | Lines |
|---|---|
| `lib.rs` | 6,570 |
| `relay.rs` | 6,233 |
| `pool.rs` | 5,620 |
| `queue.rs` | 4,759 |
| `acp.rs` | 3,717 |
| `config.rs` | 2,709 |
| `setup_mode.rs` | 1,135 |
| `usage.rs` | 892 |
| `filter.rs` | 787 |
| `pool_lifecycle.rs` | 312 |
| `engram_fetch.rs` | 248 |
| `observer.rs` | 166 |
| `main.rs` | 3 |

The desktop and mobile trees enforce a 1,000-line-per-file ceiling (`AGENTS.md § Mobile App`, `desktop`/`web` equivalents). No such guard exists for Rust crates, and the five largest files here are 3.7×–6.6× that ceiling.

`main.rs` is a pure three-line delegate (`main.rs:1-3`) — the whole binary lives in the library, so everything is unit-testable in-crate.

#### Error handling

Three layers, applied consistently:

| Layer | Type | Sites |
|---|---|---|
| Module-internal | `thiserror` enums — `acp::AcpError` (`acp.rs:79`), `relay::RelayError` (`relay.rs:437`), `pool::SteerError` (`pool.rs:337`) | |
| Orchestration | `anyhow::Result` with `map_err(\|e\| anyhow!("<context>: {e}"))` | `lib.rs:1290`, `1345`, `1360`, `1428`, `1440`, `3857`, `3878` |
| Subcommands | `eprintln!` + `std::process::exit(1)` | `lib.rs:3904-3916`, `3952-3966`, `3974-3979`, `3986-3999`, `4020-4023`, `4034-4041` |

The subcommand layer discards `anyhow` context and collapses every failure to exit code 1, unlike `buzz-cli`'s documented 0/1/2/3/4/5 scheme (`AGENTS.md § Agent CLI`).

`unsafe`: none. `#![deny(unsafe_code)]` at `lib.rs:1`. `largest_shrinkable_leaf` explicitly does two passes to satisfy the borrow checker "without unsafe" (`lib.rs:696-701`).

`unwrap()` / `expect()` in production paths — 4 total, all `expect` with justifying comments:

| Site | Call |
|---|---|
| `lib.rs:1243` | `.expect("failed to install rustls crypto provider")` |
| `lib.rs:1645` | `.expect("SIGTERM handler")` |
| `lib.rs:2548` | `.expect("successful wake stores a ready pool")` |
| `lib.rs:4167` | `.expect("secret key bech32 encoding should never fail")` |

The remaining 46 of the 50 `unwrap`/`expect` occurrences are inside `#[cfg(test)]` modules. `lib.rs:4161-4164` documents why the bech32 panic is preferred over silent fallback. `lib.rs:2548` is reachable if `PoolLifecycle::complete_wake` and `take_ready` ever disagree — an invariant panic in a non-test path.

There are three `debug_assert` invariants: `lib.rs:3043`, `lib.rs:2531`, and the `try_send` contract comment at `lib.rs:1183-1187`.

#### Async patterns

- Single `#[tokio::main]` runtime (`lib.rs:1238-1239`); `run()` stays sync only so `config::propagate_legacy_env_vars()` can call `std::env::set_var` before worker threads exist (documented `lib.rs:1226-1231`, a Rust 2024 soundness requirement).
- One giant `tokio::select!` `biased;` (`lib.rs:1823`) inside the main `loop` (`lib.rs:1707`). Ordering is exploited: pool results first, then panics, steer acks, wake results, retry sleep, observer control, relay events, heartbeats, presence, typing, shutdown.
- Empty-collection spin guards on every `join_next()` arm: `if !join_set.is_empty()` (`lib.rs:1836`), `if !wake_tasks.is_empty()` (`lib.rs:1862`). The comment at `lib.rs:1833-1835` names the tight-spin failure mode.
- `std::future::pending()` is the idiom for a disabled arm: `heartbeat` (`lib.rs:2282`), `presence_heartbeat` (`lib.rs:2305`), `typing_refresh` (`lib.rs:2325`), observer control (`lib.rs:1885`), retry deadline (`lib.rs:1859`).
- Borrow splitting: `pool.rx_and_join_set()` (`lib.rs:1710`) yields two disjoint `&mut`s; arms that need `pool` whole end the split with an explicit `let _ = result_rx;` (`lib.rs:1888`, `1906`, `2285`, `2308`, `2328`).
- Maintenance runs at the **top** of the loop on an `Instant` check, not as a `select!` arm, specifically so `biased` cannot starve it (comment `lib.rs:1605-1607`, code `lib.rs:1744`).
- Fire-and-forget `tokio::spawn` for cosmetic side effects: 👀 add (`lib.rs:2206-2211`), 👀 cleanup (`lib.rs:1938-1943`), failure notices (`lib.rs:3025-3029`), presence heartbeat (`lib.rs:2314-2318`), steer ack watcher (`lib.rs:2853-2860`). Presence is the only one whose handle is retained and aborted (`lib.rs:2311-2313`, `2676-2678`).
- RAII is used where a dropped task would corrupt state: `RespawnGuard` (`lib.rs:1172-1231`).

#### Channel sizing rationale is documented at every declaration

| Channel | Line | Capacity | Stated reason |
|---|---|---|---|
| `respawn_tx` | `lib.rs:1613` | `config.agents` | at most one respawn per slot in flight |
| `wake_tx` | `lib.rs:1616` | 1 | one wake attempt at a time |
| `steer_ack_tx` | `lib.rs:1617-1629` | unbounded | losing an ack would leak a withheld event until `IN_FLIGHT_DEADLINE_SECS` |
| `shutdown_tx` | `lib.rs:1632` | `watch` | multi-consumer broadcast |
| control signal | `lib.rs:2961` | `oneshot` | one signal per turn |
| steer request | `lib.rs:2933` | `mpsc(1)` | one in-flight steer per turn |

#### Logging / tracing discipline

`tracing` macros with structured fields, initialized once at `lib.rs:1276-1281` (`EnvFilter`, `.compact()`, default `buzz_acp=info`).

Level usage is consistent:

| Level | Use | Examples |
|---|---|---|
| `error!` | invariant violations and terminal states | `lib.rs:1190`, `1213`, `2299`, `2372`, `3276`, `3423`, `3466` |
| `warn!` | recoverable degradation | `lib.rs:139`, `282`, `483`, `801`, `846`, `853`, `863`, `1379`, `1477`, `1486` |
| `info!` | lifecycle | `lib.rs:131`, `1284`, `1356`, `1362`, `1372`, `1438`, `1488`, `2775` |
| `debug!` | per-event decisions | `lib.rs:355`, `1871`, `1885`, `2028`, `2161`, `2178`, `2271`, `2905`, `2911` |

Machine-readable snake_case event names appear as bare messages for grep-ability: `heartbeat_skipped_events`, `heartbeat_skipped_busy`, `heartbeat_skipped_pool_not_ready` (`lib.rs:2271`, `2280`, `2270`), `pool_exhausted` (`lib.rs:2905`), `agent_claimed` (`lib.rs:2911`), `dispatch_pending` (`lib.rs:2994`), `agent_returned` (`lib.rs:3237`), `agent_pool_ready` (`lib.rs:3844`), `heartbeat_fired` (`lib.rs:3585`).

Author pubkeys are logged at debug on gate rejection (`lib.rs:2161-2168`) and warn on non-owner control frames (`lib.rs:851-858`). No secret material is logged — `config.summary()` (`config.rs:1012-1040`) prints `relay`, `pubkey`, commands, and timeouts, never `keys.secret_key()`.

Diagnostic context deliberately attached to fatal-outcome logs: `configured_model` and `pid` (`lib.rs:3230-3236`, `3288-3296`, `3348-3354`, `3374-3381`) — the comment at `lib.rs:3188-3195` explains that `configured_model` is spawn-time config and may legitimately differ from a `session/set_model` override, and is kept only to identify stale orphans.

#### Comment style

The file leans heavily on long prose block comments that carry design decisions and rejected alternatives, often 20+ lines:

- steer ack semantics with per-variant rationale and attribution: `lib.rs:2417-2478` (61 lines)
- `fit_observer_event_to_budget` doc including a termination proof and an explicit "double-serialize accepted" tradeoff note: `lib.rs:634-658`
- accepted membership race with the cost of the correct fix spelled out: `lib.rs:1664-1680`
- `is_auth_error` classification rationale with false-positive analysis: `lib.rs:2989-3002`
- two-layer membership dedup and why `<` not `<=`: `lib.rs:1863-1876`
- `try_native_steer` caller invariants as a contract: `lib.rs:2779-2802`

Three sites carry explicit cross-file "edit one, review the other" coupling notes: `lib.rs:539-543` (↔ `buzz-core/src/observer.rs:25`), `lib.rs:2812-2819` (↔ `queue::native_steer_framing`), `lib.rs:3048-3051` (requeue-before-mark_complete ordering).

#### Test organization

Tests are **in-file** `#[cfg(test)]` modules, not `tests/`. Eleven modules in `lib.rs`:

| Module | Line |
|---|---|
| `agent_draft_prompt_tests` | 3589 |
| `heartbeat_base_prompt_tests` | 4187 |
| `owner_control_command_tests` | 4219 |
| `owner_cache_tests` | 4347 |
| `author_gate_tests` | 4370 |
| `observer_snapshot_race_tests` | 4742 |
| `observer_publish_pacer_tests` | 4807 |
| `observer_chunk_coalescer_tests` | 4841 |
| `build_mcp_servers_tests` | 4939 |
| `error_outcome_emission_tests` | 5085 |
| `observer_payload_trim_tests` | 6324 |

Test code occupies roughly 2,406 of 6,570 lines (~37 %): lines 3588–3608 and 4186–6570, with production code at 1–3587 and 3609–4185.

The whole `crates/buzz-acp/tests/` directory holds one file, `pool_lifecycle_state.rs`, 159 bytes — so there is essentially no integration-level test surface for the crate.

Conventions inside tests:

- `#[tokio::test(start_paused = true)]` for time-dependent logic (`lib.rs:4809`, `4823`, `4752`).
- Real subprocesses as inert fixtures: `AcpClient::spawn("cat", …)` (`lib.rs:5183`) and `agent_command: "true"` (`lib.rs:5104`), with comments explaining why (`lib.rs:5100-5103`, `5179-5182`).
- A real `tokio::net::TcpListener` HTTP stub for lazy channel resolution (`lib.rs:4626-4685`) rather than a mocking framework.
- `static ENV_LOCK: Mutex<()>` to serialize env-var tests, since env is process-global (`lib.rs:4941-4942`).
- Test-only accessors on production types: `queue.set_retry_count_for_test` (`queue.rs:610`, used `lib.rs:5721`) and `RelayEventPublisher::test_pair` (`lib.rs:4756`).
- Two full `Config` literals are hand-maintained in tests (`lib.rs:4946-4990`, `lib.rs:5097-5151`) — every new `Config` field requires editing both.
- `#[allow(clippy::too_many_arguments)]` on three orchestration functions (`lib.rs:3033`, `3401`, `3500`); `handle_prompt_result` takes 11 parameters.

Notable: `error_outcome_emission_tests` documents a *structural* invariant — `handle_prompt_result` takes no relay handle, so it physically cannot post a channel message; re-adding one would break compilation of these tests (`lib.rs:5086-5096`).


## Module: buzz-acp — relay client & observer (`crates/buzz-acp/src`)
### Aspect: Conventions

#### Actor-over-channels concurrency

One background tokio task owns the `WsStream` exclusively
(`relay.rs:615-634`); `HarnessRelay` never touches the socket. All
communication is through three `mpsc` channels:

| Channel | Direction | Capacity | Location |
|---|---|---|---|
| `event_tx`/`event_rx: Option<BuzzEvent>` | bg → harness | `event_channel_capacity()`, default 256 | `relay.rs:610` |
| `observer_control_tx`/`_rx: Event` | bg → harness | same as above | `relay.rs:611-612` |
| `cmd_tx`/`cmd_rx: RelayCommand` | harness → bg | `CMD_CHANNEL_CAPACITY` = 64 (`relay.rs:31`) | `relay.rs:613` |

`Option<BuzzEvent>` is used as an in-band sentinel: `None` = connection lost
(`relay.rs:811-813`). All recovery-path sends use `try_send` so a full event
channel cannot stall reconnection (`relay.rs:1697`, `:1770`, `:1826`, `:1885`,
`:1919`, `:2131`, `:2170`), with the reason stated inline at `relay.rs:1819-1821`.

`observer.rs` uses a different shape: `tokio::sync::broadcast` for live fan-out
plus a `std::sync::Mutex<VecDeque>` replay buffer, both sized by
`OBSERVER_BUFFER_CAP` (`observer.rs:45-53`). `AtomicU64` with
`Ordering::Relaxed` for the sequence counter (`observer.rs:107`).

#### Single-dispatch discipline

`handle_ws_message` (`relay.rs:2043-2390`) is the only frame handler.
`process_handshake_buffer` re-serialises buffered `RelayMessage`s back to JSON
text specifically to route them through it, with the tradeoff acknowledged in a
comment: "slightly wasteful but keeps the handler as the single source of truth"
(`relay.rs:2405-2407`).

Similarly `ws_send_timeout` (`relay.rs:3312-3323`) is the single send path —
"All `ws.send()` calls go through here" (`relay.rs:3307`). The three read loops
each answer `Ping` through it.

#### State-intent separation

Commands are applied to `BgState` independently of whether the wire send
succeeded. Three named helpers encode the policy:

| Helper | Location | Contract |
|---|---|---|
| `apply_command_to_state` | `relay.rs:1244-1290` | disconnected path; `Shutdown` arm is a `debug_assert!(false)` (`:1285-1288`) |
| `retain_failed_command_intent` | `relay.rs:1307-1319` | live-send-failure path; observer frames parked, other publishes discarded |
| `retain_deferred_command_intent` | `relay.rs:1324-1338` | replay-lost-socket path; drains a `VecDeque` in arrival order |

`execute_connected_command` (`relay.rs:1346-1531`) returns `bool` — `false`
means "dead socket, reconnect" — and its `Shutdown`/`Reconnect` arm is also a
`debug_assert!(false)` (`:1526-1529`). The invariants are documented at the
function level rather than encoded in the type: `RelayCommand` still carries the
control variants the function refuses to handle.

#### Typed outcome enums instead of booleans-plus-comments

`ReconnectOutcome { Ok, Failed, Shutdown }` (`relay.rs:2778-2787`) and
`ResubscribeResult { Ok, RetryConnection, Shutdown }` (`relay.rs:2457-2466`).
Callers `matches!` on `Shutdown` and return immediately; the doc comment on
`try_autonomous_reconnect` spells out why falling through would loop forever
(`relay.rs:2884-2888`). `send_subscribe`,
`send_membership_subscribe`, `send_observer_control_subscribe`, and
`send_publish_event_frame` all return bare `bool`.

#### `select!` conventions

The main loop is a four-arm `tokio::select!` (`relay.rs:1784-2021`): socket read,
command receive, ping tick, and a pacing timer. The pacing arm parks on
`std::future::pending()` when no drain is scheduled so it "never fires
spuriously and never blocks the other select! arms"
(`relay.rs:2011-2013`, implementation `:1991-1998`).

Every sleep that could swallow a `Shutdown` is a `select!` over
`sleep_until(deadline)` and `cmd_rx.recv()`. Three variants:
`pacing_sleep` defers commands for later live execution
(`relay.rs:3369-3392`); `dns_flat_sleep` applies them to state immediately
(`relay.rs:3395-3411`); the two inline backoff sleeps also apply to state
(`relay.rs:2996-3008`, `:3133-3145`). Deadlines rather than `sleep(d)` are used
so command traffic cannot reset the timer — stated at `relay.rs:2990-2992` and
`:3121-3126`.

`ping_interval` sets `MissedTickBehavior::Delay` (`relay.rs:1613`).

#### Error handling

`RelayError` is a `thiserror` enum (`relay.rs:435-459`). `WebSocket` boxes the
inner `tungstenite::Error` (`relay.rs:437`) to keep the enum small. Errors are
mapped with `map_err` closures at every boundary — `RelayError::Http` is reused
as a catch-all for URL parse (`relay.rs:3831`, `:3440`), tag parse
(`relay.rs:3446`, `:3448`), reqwest client build (`relay.rs:594`), and NIP-98
signing (`relay.rs:272`, `:294`, `:296`), which flattens genuinely different
failures into one variant.

**Zero `unwrap()` / `expect()` / `panic!()` in the production half of all three
files.** `relay.rs` lines 1–3,994 contain none; the first occurrence is
`relay.rs:4040`, inside `mod tests`. `observer.rs` has none at all.
`engram_fetch.rs` has 9, all inside `#[cfg(test)] mod tests` (lines 178-234).
`unwrap_or_default()` is used freely on `SystemTime` arithmetic
(`relay.rs:257`, `:3190`, `:3245`, `:3282`, `:3341`), and
`nip98_header(...).unwrap_or_default()` at `relay.rs:379` silently yields an
empty `Authorization` header on signing failure — the comment claims it is
"infallible in practice" (`relay.rs:377-378`).

`#![deny(unsafe_code)]` is set crate-wide (`lib.rs:1`); no `unsafe` block exists
in any of the three files. The one place that reaches for a raw pointer
alternative avoids it explicitly: `largest_shrinkable_leaf`'s two-pass borrow
dance in `lib.rs` is annotated "keep the borrow checker happy without unsafe".

#### Logging discipline

`tracing` only, imported as `use tracing::{debug, info, warn};`
(`relay.rs:126`). No `println!`/`eprintln!` in any of the three files.
No `error!` level is used anywhere in `relay.rs` — the most severe events
(reconnect exhaustion, dropped telemetry) are `warn!`.

Structured fields are used inconsistently: some sites use key-value form
(`relay.rs:1218-1221`, `:2103-2106`, `:2160-2168`, `:2656-2659`) while most use
interpolated messages (`relay.rs:1408`, `:2246-2254`, `:2688`, `:3186`).
`observer.rs` uses `target: "observer"` on its two warnings
(`observer.rs:98`, `:130`); `engram_fetch.rs` uses `target: "engram::core"`
(`engram_fetch.rs:51`). `relay.rs` sets no `target`.

Sensitive-value discipline is good on the AUTH path: `relay.rs:3461` logs a
fixed string, never the frame. It is weaker on the failure path:
`relay.rs:2059` logs the entire unparsed relay frame
(`warn!("failed to parse relay message: {e} — raw: {text}")`).

#### Documentation style

Long rationale comments are the dominant convention — most constants carry a
paragraph explaining the number (`relay.rs:47-113`), and several functions have
20-40 line doc comments covering invariants and known tradeoffs
(`relay.rs:2468-2487` on `resubscribe_after_reconnect`, `:3625-3656` on
terminal-error classification, `:925-933` on the dedup amnesia window,
`:3489-3496` on the CLOSED string matching). Cross-file coupling is called out
explicitly where it exists (`relay.rs:3494-3496` names `req.rs:153` and
`side_effects.rs:71`; `relay.rs:3762-3763` names the rustls version).

Every public item in the three files has a doc comment. `#[allow(...)]` is used
sparingly and always locally: `clippy::too_many_arguments` on the five
9-argument background functions (`relay.rs:1533`, `:2042`, `:2392`, `:2892`,
`:3021`), `dead_code` on `HarnessRelay::publish_event` (`relay.rs:820`),
`private_interfaces` on `parse_relay_message` (`relay.rs:3532`).

#### Naming

`send_*` for wire writes, `drain_*` for paced queue processing, `is_terminal_*`
/ `is_dns_error` / `is_retriable_status` for classifiers, `*_sleep` for
shutdown-aware waits, `apply_*`/`retain_*` for state mutation. Subscription ids
are built and parsed by a matched pair, `channel_sub_id` (`relay.rs:3478-3480`)
and `channel_id_from_sub_id` (`relay.rs:3484-3488`), with a round-trip test
(`relay.rs:4048-4053`).

#### Test conventions

All tests are inline `#[cfg(test)] mod tests` — `relay.rs:3995-6233` (2,239
lines, 36 % of the file), `engram_fetch.rs:167-247`. `observer.rs` has no tests
of its own. Async tests use `#[tokio::test(start_paused = true)]` (10
occurrences) so backoff and pacing are asserted deterministically. Real
WebSocket pairs are built over `127.0.0.1:0` by the `test_ws_pair` helper
(`relay.rs:4340-4355`) with `next_test_frame` for assertions
(`relay.rs:4357-4367`). `RelayEventPublisher::test_pair` (`relay.rs:575-588`) is
a `#[cfg(test)]` seam that forwards published events to a receiver instead of a
socket. Test names read as behaviour statements
(`dropped_channel_is_not_resubscribed_so_loop_cannot_re_form`,
`:5044`; `membership_dedup_does_not_touch_last_seen`, `:4872`).


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


## Module: buzz-acp — work queue, filtering & usage accounting (`crates/buzz-acp/src`)
### Aspect: Conventions

#### Naming

| Pattern | Examples |
|---|---|
| Verb-first mutators on `EventQueue` | `push`, `flush_next`, `mark_complete`, `requeue`, `drain_channel`, `compact_expired_state` (`queue.rs:230`, `:260`, `:392`, `:429`, `:625`, `:807`) |
| `_for_test` suffix on `#[cfg(test)]` accessors used by *other* modules' tests | `set_retry_count_for_test` (`queue.rs:610`), `queued_event_count` (`queue.rs:600`) |
| `MAX_*` for hard caps, `BASE_*`/`DEFAULT_*` for tunable-by-code values | `MAX_PENDING_PER_CHANNEL`, `MAX_BATCH_EVENTS`, `MAX_RETRIES`, `MAX_RETRY_DELAY_SECS`, `MAX_EXPR_LEN`, `MAX_CONSECUTIVE_TIMEOUTS`, `MAX_CONCURRENT_FILTER_EVALS`, `MAX_PROMPT_LABEL_LEN` vs `BASE_RETRY_DELAY_SECS`, `DEFAULT_IN_FLIGHT_DEADLINE_SECS` (`queue.rs:24-42`, `:1023`, `filter.rs:162-341`) |
| `format_*` for pure string builders, `resolve_*` for lookups that can fail, `parse_*`/`extract_*` for input → structure | `format_prompt`, `format_event_block`, `format_context_hints`, `format_conversation_context`, `format_prompt_actor` / `resolve_prompt_label`, `resolve_reply_anchor` / `parse_thread_tags`, `extract_slash_command` |
| Builder-style consuming setter | `with_in_flight_deadline(self) -> Self` (`queue.rs:197`) — the only builder method; everything else is `&mut self` |
| `Args` struct instead of long parameter lists | `FormatPromptArgs<'a>` with `#[derive(Default)]` so tests set only what they need (`queue.rs:1352-1375`) |

#### Error handling

- `filter.rs` is the only module with an error type: `FilterError` via `thiserror` with three variants (`filter.rs:14-24`). `evaluate_filter` returns `Result`; `match_event` swallows all errors into `Option::None` after logging (`filter.rs:426-448`).
- `queue.rs` and `usage.rs` define **no** error types. Every fallible situation is expressed as `Option`, `bool`, or a `tracing` log:
  - `push` → `bool` (`queue.rs:230`)
  - `flush_next` / `requeue` / `slash_command_for_batch` / `extract_slash_command` / `UsageTracker::take` → `Option`
  - `mark_native_steer_pending` → `bool`; `release_native_steer` / `remove_event` → silent idempotent no-ops (`queue.rs:704-711`, `:739-750`)
  - `format_prompt` on an empty batch → `tracing::error!` + empty `Vec` (`queue.rs:1411-1417`)
- Log-level convention is consistent: `ERROR` for invariant violations (`"BUG: in-flight channel expired without mark_complete"` at `queue.rs:272-278` / `:568-574`, dead-letter at `queue.rs:439-445`, disabled filter rule at `filter.rs:406-412`); `WARN` for lossy-but-expected events (cap overflow at `queue.rs:245-249`, `:490-494`, `:723-728`, `:776-781`; requeue at `queue.rs:466-472`; withheld-steer recovery at `queue.rs:783-788`); `INFO` for deadline extension (`queue.rs:216-219`); `DEBUG` for drop-mode discards (`queue.rs:235-238`).
- `no unsafe` is enforced crate-wide by `#![deny(unsafe_code)]` (`lib.rs:1`); none of the three files contains `unsafe`.
- Production `unwrap`/`expect` in these files: `q.front().unwrap()` inside the `min_by_key` on a pre-filtered non-empty queue (`queue.rs:299`) and `q.remove(pos).expect("position came from iter so remove must succeed")` (`queue.rs:681-682`). Both are guarded by a preceding check. Everything else is `unwrap_or*`: `queue.rs:271`, `:317`, `:366`, `:459`, `:567`, `:630`, `:915`, `:939`, `:1083`, `:1087`, `:1192`, `:1222`, `:1422`. `filter.rs` and `usage.rs` have **zero** `unwrap`/`expect` outside tests.
- `filter.rs` timeouts fail closed by policy, documented three times in-code (`filter.rs:357-366`, `:413`, `:434`, `:444`).

#### Documentation style

- Every public item in all three files has a doc comment; several carry the invariant *and* its rationale (e.g. `mark_complete`'s retry-preservation contract at `queue.rs:386-391`, `requeue`'s "does NOT remove from `in_flight_channels`" at `queue.rs:426-427`).
- `EventQueue`'s state machine is documented as an ASCII pseudocode block in the type-level doc comment (`queue.rs:94-136`) — the only place the full transition table exists.
- `usage.rs` documents its three delta cases as a numbered list in the module header (`usage.rs:16-30`) and repeats the three `record` branches as a numbered list on the method (`usage.rs:198-210`).
- `filter.rs` documents the evalexpr variable surface as a markdown table inside the `build_eval_context` doc comment (`filter.rs:253-259`).
- Comments name individual reviewers as the source of a requirement — "per Dawn's framing review" (`queue.rs:1592`), "Eva's drift-proof requirement" (`queue.rs:1620`, `lib.rs:2813`), "Eva+Wren and Thufir both flagged" (`usage.rs:453`), "Regression for Thufir fix 2" (`usage.rs:507-508`). These carry design intent that exists nowhere else.

#### How state machines are expressed

There is no state-machine type or enum-based state. `EventQueue` encodes channel state implicitly across five parallel maps plus one set, and the "state" of a channel is the conjunction of its memberships:

| Channel state | Encoding |
|---|---|
| idle | absent from every map |
| pending | `queues[ch]` non-empty, not in `in_flight_channels` |
| in flight | `in_flight_channels` ∋ ch, `in_flight_deadlines[ch]` set |
| throttled | `retry_after[ch] > now` |
| retrying | `retry_counts[ch] > 0` |
| cancelled-pending | `cancelled_batches[ch]` non-empty |
| steer-withheld | `withheld_native_steer[ch]` non-empty |

Consequences: transitions are enforced by call-order discipline, not types (see the requeue-before-`mark_complete` contract at `lib.rs:3061-3065`), and the same expiry block is physically duplicated between `flush_next` (`queue.rs:263-287`) and `has_flushable_work` (`queue.rs:558-581`) rather than extracted.

`CancelReason` (`queue.rs:65-74`) and `MergeFraming` (`queue.rs:1571-1610`) are the one place a state-to-behavior mapping is expressed as an exhaustive `match` returning a data struct — `MergeFraming::for_reason` folds `None` into the `Steer` arm deliberately (`queue.rs:1586-1588`).

`UsageTracker` is a three-state machine expressed as `Option<String> in_flight_session` + `Option<TurnUsage> pending`, with the branch decided by an `is_in_flight` bool computed once (`usage.rs:219`, `:259-296`).

#### Test organization

| File | Tests | Location | Runtime |
|---|---|---|---|
| `queue.rs` | 112 | inline `#[cfg(test)] mod tests` at `queue.rs:1628-4759` (3,132 lines — 66 % of the file) | all sync `#[test]` |
| `filter.rs` | 15 | inline at `filter.rs:462-787` | 11 `#[tokio::test]`, 4 `#[test]` |
| `usage.rs` | 20 | inline at `usage.rs:322-891` | all sync `#[test]` |

Conventions inside the test modules:

- Shared fixture helpers at the top of each module: `make_event` / `make_queued` / `make_queued_at` / `make_queued_created_at` / `make_event_with_tags` / `make_merged_batch` / `make_single_batch` / `pending_count` / `any_in_flight` (`queue.rs:1635-1690`, `:1897`, `:2885`, `:4160`); `make_event` / `make_event_with_p_tag` / `any_channel` / `make_rule` (`filter.rs:469-510`); `payload` / `payload_no_context` / `payload_with_model` (`usage.rs:326-349`, `:828`).
- `queue.rs` and `filter.rs` prefix every test `test_`; `usage.rs` does **not** (e.g. `first_turn_no_prior_delta_unreliable`, `usage.rs:461`) — an inconsistency between sibling files in the same crate.
- Test helpers reach **private fields directly** (`q.queues`, `q.in_flight_channels` at `queue.rs:1684`, `:1688`; `tracker.pending` at `usage.rs:360-363`) rather than through the public API, which is why some public accessors have no test caller.
- `usage.rs` groups tests with box-drawing section banners (`usage.rs:348`, `:458`, `:586`, `:662`, `:826`).
- Regression tests carry the original defect in the comment body rather than the name — `cross_session_notification_does_not_corrupt_other_sessions_delta` explains the exact undercount it prevents (`usage.rs:404-460`).
- Float comparisons use explicit epsilons (`(dc - 0.007).abs() < 1e-9`, `usage.rs:385`), never `assert_eq!` on `f64`.
- Zero `TODO` / `FIXME` / `HACK` / `XXX` markers across all three files.


## Module: buzz-acp — ACP protocol, config & setup mode (`crates/buzz-acp/src`)
### Aspect: Conventions

#### Error handling

Two `thiserror` enums, one per concern: `AcpError` for the wire/subprocess (`acp.rs:78-108`) and `ConfigError` for startup (`config.rs:38-49`). `setup_mode.rs` uses `anyhow::Result` throughout (`setup_mode.rs:36`) and wraps every relay error with `anyhow::anyhow!` plus a `setup-mode` prefix (`setup_mode.rs:332`, `:340`, `:354`).

`?` propagation is the norm. The crate is `#![deny(unsafe_code)]` (`lib.rs:1`), which is why `killpg` goes through the `nix` safe wrapper rather than a raw syscall (`acp.rs:1975-1977`).

`unwrap()` / `expect()` in these three files, all outside the request path:

| Site | Form | Line |
|---|---|---|
| Wire log formatting | `serde_json::to_string(&msg).unwrap_or_default()` (4 sites) | `acp.rs:703`, `:994`, `:1059`, `:1344` |
| Steer routing | `pending_steer.take().expect("just checked")` | `acp.rs:1462` |
| `install_steer_rx` | `assert!` with an explanatory message | `acp.rs:801-805` |
| Command basename | `.expect("rsplit always yields at least one element")` | `config.rs:605` |
| Setup nudge JSON | `.expect("SetupPayload must be serializable to JSON")` behind a `// SAFETY:` comment | `setup_mode.rs:294-296` |
| MCP key encoding | `.expect("secret key bech32 encoding should never fail")` — outside these files, at `lib.rs:4168`, with a comment arguing the panic is *correct* because a bogus secret would cause delayed downstream failures | `lib.rs:4162-4169` |

Test code uses `unwrap()` freely. The `AGENTS.md` rule is "no *new* `unwrap()`/`expect()` in production paths"; the surviving instances are all either infallible-by-construction or documented invariant assertions.

#### Async and read-loop structure

- Every I/O method is `async fn` on `&mut self` — there is exactly one owner of the pipes and no interior mutability.
- Two near-duplicate read loops: `read_until_response` (`acp.rs:1074-1166`) for non-prompt RPCs and `read_until_response_with_idle_timeout` (`acp.rs:1198-1523`) for prompts. Both share the frame-parse, observe, id-match, and method-dispatch shape; the second adds dual deadlines and the steer arm. The duplication is ~90 lines.
- `tokio::select!` uses `biased` (`acp.rs:1277`) with reader → steer → sleep ordering, plus a pre-select deadline check to compensate (`acp.rs:1256-1274`).
- The steer arm's disabled case is expressed as `async { match steer_rx.as_mut() { Some(rx) => rx.recv().await, None => None } }` with a `Some(req)` pattern, so a missing receiver disables the branch without busy-looping (`acp.rs:1283-1292`). Cancel-safety is called out explicitly in the comment (`acp.rs:1284-1286`).
- Timeouts are applied with `tokio::time::timeout` around individual awaits, sequenced rather than nested, because a borrow of `self` cannot cross two awaits inside a single `timeout` future (`acp.rs:991-1003`).
- `Drop` does the non-await-able half of teardown (`start_kill` + `try_wait`) and the doc comment directs callers to `shutdown().await` (`acp.rs:1954-1956`, `acp.rs:373-375`).

#### Tracing targets

A per-concern target namespace, all under `acp::`:

| Target | Levels used | Sites |
|---|---|---|
| `acp::wire` | `debug` | outbound frames (`acp.rs:703`, `:994`, `:1059`, `:1317`), inbound frames (`acp.rs:1101`, `:1395`), parse warnings (`acp.rs:1114`, `:1408`), unknown-method notices (`acp.rs:1160`, `:1507`), drain (`acp.rs:1036`) |
| `acp::init` | `debug` | initialize response (`acp.rs:542`) |
| `acp::session` | `info` | session created (`acp.rs:583`) |
| `acp::cancel` | `debug`, `info` | `acp.rs:917`, `:923` |
| `acp::stream` | `info` | agent message chunks (`acp.rs:1537`) |
| `acp::thought` | `debug` | thought chunks (`acp.rs:1569`) |
| `acp::tool` | `info` | tool_call / tool_call_update (`acp.rs:1549`, `:1559`) |
| `acp::plan` | `info` | `acp.rs:1564` |
| `acp::update` | `debug`, `info` | `acp.rs:1578`, `:1607`, `:1613`, `:1623` |
| `acp::usage` | `debug` | `acp.rs:1642`, `:1658`, `:1672` |
| `acp::permission` | `debug`, `warn`, `info` | `acp.rs:1687`, `:1712`, `:1718` |

Timeout and lifecycle warnings deliberately use the **default** target rather than a namespaced one (`acp.rs:1265`, `:1271`, `:1365`, `:1370`, `:395`), so they surface under the crate path.

`config.rs` and `setup_mode.rs` use no targets at all — plain `tracing::warn!` / `info!` / `debug!`. `setup_mode` prefixes every message with the literal `setup-mode:` (e.g. `setup_mode.rs:342`, `:355`, `:381`).

Structured fields are used sparingly and inconsistently: `config.rs:653-656` uses `relay_url` and `error = %e`; `config.rs:818-821` uses `channel = %ch`; `acp.rs:1660-1664` uses `session_id = %…, input, output`; but most messages interpolate inline.

#### Doc-comment style

Module headers open with a `//!` block; `acp.rs:1-9` lays out the five-step lifecycle, `config.rs:1-4` states the CLI-first principle, `setup_mode.rs:1-33` gives an ASCII branch diagram plus a "Contract (NON-NEGOTIABLE)" section.

Public items carry `///` docs, per the repo rule. Several long comments encode reasoning rather than description and are the only record of the corresponding invariant:

- Why `initialize` pins v2 — an intentional squat ahead of the upstream ACP RFD (`acp.rs:536-537`).
- Why permission responses are write-then-flag, with the deadlock analysis (`acp.rs:1735-1748`).
- Why the pre-select deadline check exists, attributed to a named reviewer (`acp.rs:1257-1262`, also `acp.rs:1189-1194`).
- Why `build_steer_params` is built inside the read loop instead of at dispatch (`acp.rs:1786-1789`, `acp.rs:1300-1310`).
- Why `propagate_legacy_env_vars` must run before the tokio runtime (`config.rs:708-714`).
- Why `DEFAULT_IDLE_TIMEOUT_SECS` is 900 — 300 s of headroom above the 600 s max shell timeout (`config.rs:17-26`).
- Why `MAX_TURN_DURATION_CEILING_SECS` is 7 days — arithmetic-overflow risk when deriving the in-flight deadline (`config.rs:33-35`).
- Why `AcpAvailabilityStatus` is duplicated rather than imported (`setup_mode.rs:52-57`).

`build_codex_config_env` carries a numbered `# Merge contract` and `# Errors` section (`acp.rs:230-256`) — the most formally specified function in the group.

#### Naming

- ACP wire identifiers stay camelCase in JSON and snake_case in Rust; there is no serde rename layer because frames are hand-built.
- Method-mirroring names: `session_new`, `session_cancel`, `session_set_model` map 1:1 to `session/new`, `session/cancel`, `session/set_model`. Goose extensions get a `goose` infix — `session_set_goose_system_prompt`, `handle_goose_usage_update`, `goose_usage`.
- Config enums render kebab-case on the CLI via `clap::ValueEnum`, with `#[value(alias = …)]` camelCase aliases on `PermissionMode` (`config.rs:124-138`) and one explicit `#[value(name = "owner-interrupt")]` (`config.rs:87`).
- Boolean CLI flags are negative (`--no-presence`, `--no-typing`, `--no-memory`, `--no-ignore-self`, `--no-mention-filter`, `--no-base-prompt`) and inverted into positive `Config` fields (`config.rs:977-985`).

#### Test organisation and counts

All tests are inline `#[cfg(test)] mod tests` at the bottom of each file — no `tests/` directory for this group.

| File | `#[test]` + `#[tokio::test]` | of which async | Module start |
|---|---|---|---|
| `acp.rs` | 76 | 27 | `acp.rs:2008` |
| `config.rs` | 102 | 0 | `config.rs:1328` |
| `setup_mode.rs` | 22 | 0 | `setup_mode.rs:651` |

Conventions visible in the test bodies:

- Tests are grouped by comment banners, e.g. `// ── config-invalid footer tests ──` (`setup_mode.rs:814`), `// ── sentinel block tests ──` (`setup_mode.rs:893`), `// ── should_nudge_for_event gate tests ──` (`setup_mode.rs:655-660`), `// ── availability round-trip tests ──` (`setup_mode.rs:702-712`).
- `config.rs` re-implements the logic under test as local helpers when the real path is embedded in `from_args`: `validate_heartbeat_interval` (`config.rs:1959`), `validate_turn_liveness` (`config.rs:2001`), `resolve_idle_timeout` (`config.rs:2214`), `parse_allowed_respond_to` / `check_allowed_respond_to` (`config.rs:2476`, `:2490`). These are copies, not calls — they can drift from `from_args` silently. Three `allowed_respond_to_full_path_*` tests (`config.rs:2588`, `:2618`, `:2639`) do exercise the real `from_args` via `CliArgs::try_parse_from`.
- `config.rs:1334-1376` provides a `test_config(mode)` builder that constructs all 40 `Config` fields literally, so every new field must be added there. Note it sets `respond_to: RespondTo::Anyone` and `multiple_event_handling: Queue`, neither of which is the production default.
- Test intent is encoded in names rather than comments: `find_allow_once_by_kind_not_by_option_id` (`acp.rs:2254` region) uses deliberately non-obvious `optionId` values ("optionId values are intentionally non-obvious to prove we don't hardcode them") to prove the lookup is by `kind`.
- Assertions carry explanatory messages with the offending value interpolated, e.g. `"codex nudge must not mention OPENAI_API_KEY; got: {body:?}"` (`setup_mode.rs:747`).
- Regression tests name the bug they guard: the availability round-trip block's comment (`setup_mode.rs:709-712`) records that `CliLogin` once lacked `availability`, so serde silently dropped it and the desktop card never rendered.
- `dev-dependencies` add `tokio` with `test-util` (for time control) and `httparse` (`Cargo.toml:79-81`).

Test-only API is gated rather than left public: `steer_rx_is_none` is `#[cfg(test)]` (`acp.rs:823`), `active_run_id` is `#[cfg_attr(not(test), allow(dead_code))]` (`acp.rs:768`).


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
| `warn!` | degraded-but-continuing | catalog discovery fallback (`lib.rs:321-324`), tool-call cap (`agent.rs:243-246`), join errors (`agent.rs:435`, `agent.rs:463`), drain timeout (`agent.rs:469`), unrenderable steer (`agent.rs:277`), empty/failed handoff summary (`handoff.rs:56`, `handoff.rs:60`) |
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
| `lib.rs` | `:828` | 4 (`models_cache_does_not_pin_on_discovery_error` `:841`, three `databricks_discovery_failure_fallback_*` `:891`, `:919`, `:945`) |
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


## Module: buzz-agent — LLM providers & configuration (`crates/buzz-agent/src`)
### Aspect: Conventions
Both files follow a consistent, deliberate house style: no `unsafe`, no lint suppressions, dense explanatory comments with dated doc-source citations, and error handling split cleanly between `String` (config) and a typed enum (runtime). The mesh subsystem added in `16d4ec33` mostly conforms; where it diverges, it does so in three specific ways noted below.

#### `unsafe` and lint attributes
Re-verified after the mesh work. Zero `unsafe` in either file — `grep -c 'unsafe' llm.rs config.rs` returned `llm.rs:0` and `config.rs:0`. This satisfies the `AGENTS.md` "No `unsafe` code" rule.

Zero lint suppressions — `grep -n '#\[allow' llm.rs config.rs` still returns no matches, so nearly 1,000 new lines of `llm.rs` arrived without one. There is no `#[allow(dead_code)]` standing in for a `TODO`, which is a genuine positive: the several `pub` items with no cross-file callers (see API Surface) are left honestly `pub` rather than papered over.

The only attributes present are `#[derive(...)]` on the nine types (`config.rs:18`, `config.rs:661`, `config.rs:679`, `config.rs:686`, `config.rs:1062`; `llm.rs:40`, `llm.rs:73`, `llm.rs:81`, `llm.rs:1343`), `#[cfg(test)]` on the two test modules (`llm.rs:1571`, `config.rs:1114`), `#[test]`/`#[tokio::test]` on test functions, and `#[async_trait::async_trait]` on one test impl (`llm.rs:3563`).

#### `unwrap` / `expect` / panic discipline
`AGENTS.md` states: "Do not introduce new `unwrap()` or `expect()` in production paths — use `?` and proper error types." Scanning only the pre-test regions (`llm.rs` lines 1-1570, `config.rs` lines 1-1113):

| Site | Construct | Assessment |
|---|---|---|
| `config.rs:510` | `.expect("supported is non-empty")` in `resolve_openai_effort` | **violates the rule.** Safe in practice because every table at `config.rs:334-362` is a non-empty `const`, but it is a genuine `expect()` on a production path |
| `llm.rs:1516` | `unreachable!("loop always returns on its final iteration…")` in `post` | production-path panic; reachable only if `MAX_RETRIES` (`llm.rs:1285`) were set to 0 |
| `llm.rs:803`, `llm.rs:901` | `serde_json::to_string(...).unwrap_or_else(\|_\| "{}".into())` | infallible-fallback, no panic |
| `llm.rs:1026`, `llm.rs:1151`, `llm.rs:1179`, `llm.rs:1226` | `.unwrap_or(...)` on `Option` | no panic |
| `config.rs:785`, `config.rs:967`, `config.rs:1023`, `config.rs:1045`, `config.rs:1055` | `.unwrap_or*` on `Option` | no panic |

So the rule is still broken exactly once (`config.rs:510`) and there is still exactly one additional deliberate panic (`llm.rs:1516`). **The mesh code added neither.** Its fallible steps all return early with a typed value instead: `?` on the `data` array (`llm.rs:48`), `match` on the bearer (`llm.rs:474-483`), `match` on the send (`llm.rs:485-499`), and a let-else on the viability test (`llm.rs:509-514`) — every one of them producing `MeshCatalogObservation::Unknown` rather than an unwrap. `saturating_add` is used for the observation counter (`llm.rs:442`) rather than a bare `+`. Test code uses `unwrap`/`expect` freely, which is conventional; the one new `panic!` (`llm.rs:1729`, on an unsupported stub status) is test-only.

#### Error handling style
Two distinct styles, split by phase — plus a third, narrow one introduced by the mesh work:

| Phase | Error type | Convention | Site |
|---|---|---|---|
| Configuration | `Result<_, String>` | every message prefixed `config: ` and names the offending env var verbatim | `config.rs:885`, `config.rs:889`, `config.rs:895`, `config.rs:900`, `config.rs:906`, `config.rs:913`, `config.rs:917`, `config.rs:920`, `config.rs:923`, `config.rs:926`, `config.rs:929`, `config.rs:932`, `config.rs:936`, `config.rs:952`, `config.rs:971`, `config.rs:1053` |
| Runtime | `AgentError` (`types.rs:213-220`) | three LLM-specific variants: `Llm`, `LlmAuth`, `LlmModelNotFound` | constructed at `llm.rs:113`, `llm.rs:1029`, `llm.rs:1202`, `llm.rs:1249`, `llm.rs:1331`, `llm.rs:1355`, `llm.rs:1440`, `llm.rs:1473`, `llm.rs:1479`, `llm.rs:1486`, `llm.rs:1497`, `llm.rs:1515` |
| HTTP-layer internal | `PostError` (`llm.rs:1344-1347`) | a two-variant private enum that exists only to carry one extra signal out of `post`; never escapes the module | doc `llm.rs:1337-1342`, collapse `llm.rs:1350-1356`, lift `llm.rs:1358-1362` |

`PostError` is the file's first internal error type, and it follows the local convention rather than inventing one: it does **not** derive `thiserror`/`Display`, does not implement `std::error::Error`, and provides an explicit `into_agent()` (`llm.rs:1350`) plus a `From<AgentError>` (`llm.rs:1358`) so that both directions are one call. Its doc comment states the scope — "It is consumed inside `openai_request`" (`llm.rs:1341`) — which is verifiable: every escape route collapses it (`llm.rs:337`, `llm.rs:372`, `llm.rs:389`, `llm.rs:393`, `llm.rs:590`).

The `config: ` prefix convention is enforced by nothing but is applied consistently at all 16 sites, and several tests assert on its payload by substring (`config.rs:1220`, `config.rs:1239`, `config.rs:1249`, `config.rs:1351`).

One notable convention in `AgentError` handling: `Llm::complete` funnels every provider arm through a single `map_err` that prepends `(model-name) ` to the inner string (`llm.rs:222-228`), so log lines read `llm: (gpt-5.5) 404 Not Found: …`. The rationale — "This is the single place all provider paths converge, so the mapping is centralized" — is spelled out at `llm.rs:216-221`. `Llm::summarize` does **not** do this (`llm.rs:230-328` has no `map_err`), so summarizer failures lose the model name. Nothing flags the asymmetry.

The mesh work introduced a subtler version of the same asymmetry: the `map_err` stamps the **configured** `effective_model` (`llm.rs:223`, `llm.rs:225`), not the model actually posted. A failure served by `mesh` is therefore reported as `llm: (auto) …`. The two `warn!` events at `llm.rs:364-370` and `llm.rs:382-387` are the only place both names appear together, via the structured fields `configured_model`, `attempted_model` and `fallback_model`.

Status-code → error-variant mapping is centralized in `post` (`llm.rs:1439-1483`) and each branch carries a comment explaining whether it is a "stall path" (`llm.rs:1433-1438`, `llm.rs:1468-1471`). The one new branch — the mesh-fallback return at `llm.rs:1450-1452` — breaks that local convention: it is the only status-mapping branch in `post` without an explanatory comment.

#### Logging targets and levels
This is the clearest style change from `16d4ec33`. **The claim that both files log exclusively at `WARN` is no longer true**: `llm.rs` now has six `debug!` events, all of them in the mesh path. `error!`, `info!`, and `trace!` remain absent from both files — grep for `tracing::error`, `tracing::info`, `tracing::trace` returned zero matches.

| Site | Level | Event | Structured fields |
|---|---|---|---|
| `llm.rs:364-370` | warn | collective request failed; retrying once with `auto` | `configured_model`, `attempted_model`, `fallback_model`, `provider_message` |
| `llm.rs:382-387` | warn | collective emitted unstructured tool markup; retrying with `auto` | `configured_model`, `attempted_model`, `fallback_model` |
| `llm.rs:464-468` | debug | resolved request model from live catalog | `configured_model`, `request_model` |
| `llm.rs:477-480` | debug | catalog auth unavailable; preserving last route | `error` |
| `llm.rs:494-497` | debug | catalog probe failed | `error` |
| `llm.rs:502-505` | debug | catalog probe not successful | `status` |
| `llm.rs:511-513` | debug | catalog response has no `data` array | — |
| `llm.rs:523-526` | debug | invalid catalog response | `error` |
| `llm.rs:666-671` | warn | sticky Responses upgrade, once per process | `provider_message` |
| `llm.rs:1325-1329` | warn | cumulative stall past 300 s | `cumulative_stall`, `attempts` |
| `llm.rs:1415-1420` | warn | transport error, retrying | `attempt`, `max_attempts`, `error` |
| `llm.rs:1454-1459` | warn | retryable status, retrying | `attempt`, `max_attempts`, `status` |
| `config.rs:149-154` | warn | thinking budget omitted, `max_output_tokens` too small | `max_output_tokens`, `level_budget`, `headroom` |
| `config.rs:219-224` | warn | adaptive effort clamped | `model`, `requested`, `clamped` |
| `config.rs:512-518` | warn | OpenAI effort substituted | `model`, `requested`, `resolved` |
| `config.rs:542-546` | warn | `max` clamped to `xhigh` on unknown model | `requested`, `resolved` |
| `config.rs:566-573` | warn | `none`/`minimal` not expressible for Anthropic | `requested` |

The level split is coherent and worth stating as the new convention: a *silent, expected* routing decision or a probe that simply did not answer is `debug`; anything that changes the model the user asked for after the request was already made is `warn`. Note the consequence — under the default subscriber (`lib.rs:155-158`) a `debug` event is invisible, so in normal operation there is **no log record at all** that a turn was served by `mesh` rather than `auto`.

All seventeen events use structured key-value fields rather than string interpolation alone, which is consistent. The mesh events also adopt a shared message prefix, `"relay-mesh auto: "` (all eight of `llm.rs:364-526`), which is a new convention in this file — no other event group here namespaces its message text. Four of the five `config.rs` warnings name the env var `BUZZ_AGENT_THINKING_EFFORT` in the message text (`config.rs:153`, `config.rs:223`, `config.rs:516`, `config.rs:545`, `config.rs:568`) — a good operator-facing convention, tying a runtime warning back to the knob that caused it. The mesh events do **not** follow it: none of the eight names `BUZZ_AGENT_PREFER_MESH_FOR_AUTO`, so an operator seeing `relay-mesh auto:` lines has no pointer back to the switch that enabled them.

No explicit `target:` is set on any event, so all seventeen inherit the module path target (`buzz_agent::llm`, `buzz_agent::config`). The subscriber is configured once in `lib.rs:155-158` to write to stderr with ANSI disabled — consistent with the "all logs go to stderr" claim in `crates/buzz-agent/README.md:220`.

#### Naming
Consistent patterns:
- Body builders: `<dialect>_body` — `anthropic_body` (`llm.rs:676`), `openai_body` (`llm.rs:761`), `responses_body` (`llm.rs:874`).
- Parsers: `parse_<dialect>` — `parse_anthropic` (`llm.rs:1154`), `parse_openai` (`llm.rs:1197`), `parse_responses` (`llm.rs:991`).
- Family predicates: `is_<property>_model` — `is_manual_budget_model` (`config.rs:586`), `is_adaptive_thinking_model` (`config.rs:603`), plus `anthropic_model_supports_xhigh` (`config.rs:184`) which breaks the pattern.
- Body classifiers: `is_<condition>_body` — `is_mesh_moa_unavailable_body` (`llm.rs:1364`), `is_mesh_moa_failure_body` (`llm.rs:1377`). A new, internally consistent pair, though it sits beside the older `is_responses_required_error` (`llm.rs:963`) which does the same job on the same input under a different naming shape.
- Heuristic predicates: `looks_like_<thing>` (`llm.rs:66`) — a new prefix in this file, and a reasonable one: it signals "best-effort classification" where `is_` signals "exact contract".
- Constant namespacing: all seven mesh constants share the `MESH_` prefix (`llm.rs:28-34`), which is the file's first grouped constant prefix; the pre-existing constants (`llm.rs:22-27`) share no scheme.
- Env parsers are pure and take `Option<&str>` so they are testable without env mutation, and each says so in its doc comment: `parse_thinking_effort` (`config.rs:621`), `parse_openai_api` (`config.rs:1020-1021`), `parse_hook_servers` (`config.rs:1089-1090`). This is a strong convention — the impure wrappers are separated (`parse_hook_servers_env`, `config.rs:1085`). **`prefer_mesh_for_auto` does not follow it**: it is parsed inline in the struct literal with `parse_env("BUZZ_AGENT_PREFER_MESH_FOR_AUTO", 0u8)? != 0` (`config.rs:807`), so there is no pure, unit-testable parser for it — the same shortcut `hints_enabled` takes (`config.rs:832`).

Three naming defects:
- `type OpenAiParse` (`llm.rs:38`) is used for the Anthropic parser on the DBv2 route (`llm.rs:199`, `llm.rs:309`), so the name lies at two call sites.
- `openai_body` builds a Chat Completions body while `responses_body` builds a Responses body — the asymmetry means "openai" means "chat" in one identifier and "the whole family" in `openai_request` (`llm.rs:344`), `openai_request_for_model` (`llm.rs:535`) and `post_openai` (`llm.rs:599`), which handles Databricks too.
- `resolve_openai_model` (`llm.rs:410`) is named after the family but implements a Buzz-specific mesh policy; nothing in the name suggests it can rewrite the caller's model. Its doc comment (`llm.rs:406-409`) carries that weight instead.

#### Doc-comment discipline
`config.rs` is unusually well documented for its function bodies: every effort-related function carries a multi-paragraph doc comment with a dated citation to the vendor documentation it encodes — `config.rs:100-123` (Anthropic extended-thinking table), `config.rs:191-204` (Anthropic effort page), `config.rs:313-332` (OpenAI model pages, with an embedded markdown table), `config.rs:578-585` and `config.rs:591-602` (both citing `platform.claude.com` URLs), `config.rs:256-266` (the `gpt5_base_matches` acceptance rules enumerated case by case). Several include worked examples (`config.rs:82-88`).

The gap is on the *types*: `Config::from_env` (`config.rs:742`) has no doc comment at all, and 20 of `Config`'s 27 fields are undocumented (see API Surface). `Llm`'s three public methods — `new` (`llm.rs:108`), `complete` (`llm.rs:123`), `summarize` (`llm.rs:230`) — likewise carry none, while its *private* helpers `openai_request` (`llm.rs:340-343`), `resolve_openai_model` (`llm.rs:406-409`), `openai_request_for_model` (`llm.rs:532-534`), `post_openai` (`llm.rs:594-598`), `try_upgrade` (`llm.rs:655-656`), and `build_token_source` (`llm.rs:1519-1528`) are all documented. The discipline is inverted relative to the `AGENTS.md` rule "New public API must have doc comments" — and `16d4ec33` extended the inversion rather than correcting it: it documented three more private items and one new private type (`PostError`, `llm.rs:1337-1342`) while leaving all three public methods bare. `prefer_mesh_for_auto` (`config.rs:729-733`) is the one place the new work moved the needle the right way — it is the eighth-ever documented `Config` field, and the only one whose doc names both its env var and its intended setter.

Six of the new items carry no doc comment at all: `MeshCatalogObservation` (`llm.rs:41`), `mesh_catalog_supports_collective` (`llm.rs:47`), `looks_like_unstructured_tool_call` (`llm.rs:66`), `MeshAutoState` (`llm.rs:74`), `cool_down_collective` (`llm.rs:397`), and `observe_mesh_virtual_model` (`llm.rs:472`) — plus the two body classifiers (`llm.rs:1364`, `llm.rs:1377`) and all seven `MESH_*` constants (`llm.rs:28-34`). Their semantics live at the use sites instead: the `Llm` field doc (`llm.rs:95-97`) describes the state machine, and the `Unknown` policy is a four-line comment inside the `match` (`llm.rs:452-456`). That is the file's existing habit, described next.

`llm.rs` uses inline block comments to record hard-won provider behaviour rather than doc comments — e.g. the OpenAI-Chat image batching rationale (`llm.rs:770-777`), the Responses replay invariant (`llm.rs:867-872`), the 401/403 refresh rationale (`llm.rs:622-629`), the 499/5xx stall-path annotations (`llm.rs:1433-1438`, `llm.rs:1468-1471`), the DBv2 substring-routing caveat (`llm.rs:971-972`), and now the mesh `Unknown`-is-not-evidence rule (`llm.rs:452-456`). Two doc comments reference GitHub issue numbers (`llm.rs:2347` cites "#559/#560"), which is the only cross-reference style used and appears only in test comments.

#### File-size discipline
| File | Lines | Test-module share |
|---|---|---|
| `llm.rs` | 3,846 | tests start at `llm.rs:1571` → ~59% test code |
| `config.rs` | 2,709 | tests start at `config.rs:1114` → ~59% test code |

`llm.rs` grew from 2,894 to 3,846 lines in `16d4ec33`/`8eb6e3eb` — a third larger — and the split held: roughly 220 production lines and 590 test lines. `AGENTS.md` states a "Hard ceiling: **1000 lines/file**". Read in context, that ceiling is scoped to the mobile app ("Keep widgets small and composable… enforced by `mobile/scripts/check-file-sizes.mjs` via `just mobile-check` (runs in `just check` + pre-push, mirroring desktop/web)"). The enforcement is JS/TS/Dart only: `justfile:123` runs `pnpm check:file-sizes` for desktop, `justfile:585` for web, `justfile:617` runs `node ./scripts/check-file-sizes.mjs` for mobile. There is **no** Rust equivalent — grep for `check-file-sizes` in `justfile` returned only those three lines, and no `crates/**` script exists. So `llm.rs` is now ~3.8× and `config.rs` ~2.7× the stated ceiling with no gate to trip, and neither is the worst offender in the repo (`crates/buzz-acp/src/lib.rs` is 6,570 lines).

Function-level size is where the real pressure is, and the mesh work is mixed on this. `Llm::complete` spans `llm.rs:123-228` (106 lines, up from 94) and `Llm::summarize` spans `llm.rs:230-328` (99 lines), each a single expression built from a three-arm `match` with nested closures. `post` grew from 112 to 128 lines (`llm.rs:1390-1517`). `Config::from_env` spans `config.rs:742-837` (96 lines) as one struct literal with 27 initializers. `openai_efforts_for_model` spans `config.rs:333-401` (69 lines), most of it a 26-term `else if` chain (`config.rs:367-392`). The test-only `valid_effort_values_for_provider_model` is 92 lines (`config.rs:2567-2658`).

On the credit side, `16d4ec33` *reduced* one hot spot by extraction rather than growing it: the old single `openai_request` was split into `openai_request` (52 lines, `llm.rs:344-395`) for policy and `openai_request_for_model` (40 lines, `llm.rs:535-574`) for dispatch, and the Anthropic POST moved out into its own `post_anthropic` (9 lines, `llm.rs:330-338`). `resolve_openai_model` (60 lines, `llm.rs:410-469`) and `observe_mesh_virtual_model` (59 lines, `llm.rs:472-530`) are the two largest new functions; both are single linear flows, not nested closures. The new test stub `spawn_sequence_stub` (84 lines, `llm.rs:1662-1745`) is the largest test helper in the file.

#### Test organization
Both files use a single in-file `#[cfg(test)] mod tests` with `use super::*` (`llm.rs:1571-1573`, `config.rs:1114-1116`). Conventions observed:
- Shared fixture builders at the top of each module: `cfg(provider)` (`llm.rs:1579`), `cfg_responses()` (`llm.rs:2245`), `image_history()` (`llm.rs:2202`), `tool_call_history()` (`llm.rs:2251`); `make_config_for_validation` (`config.rs:1844`).
- Table-driven tests where the input space is enumerable: `is_responses_required_error_matrix` (`llm.rs:2448`), `databricks_v2_routes_by_model_family` (`llm.rs:2464`), `collective_catalog_requires_two_distinct_physical_models` (`llm.rs:1891`), `parse_openai_api_values` (`config.rs:1206`), `is_openai_host_matrix` (`config.rs:1275`), `clamp_adaptive_effort_low_medium_high_never_clamped` (`config.rs:1731`).
- Every assertion carries a message naming the invariant, frequently with the offending input interpolated: `llm.rs:2293`, `llm.rs:2597`, `config.rs:1271`, `config.rs:2227`. The mesh tests follow this closely — e.g. "two spellings of one model are not collective capacity" (`llm.rs:1908`), "cooldown request must not re-probe the catalog" (`llm.rs:2003`), "mesh-specific failure must enter cooldown without re-probing" (`llm.rs:2049`), "pseudo tool markup must enter cooldown without another catalog probe" (`llm.rs:2099`).
- Test names encode the expected behaviour, not the function under test alone: `anthropic_body_omits_thinking_when_max_output_too_small` (`llm.rs:2646`), `openai_efforts_for_model_boundary_gpt5_4o_is_base_not_5_4` (`config.rs:2304`), and the whole mesh set (`mesh_auto_does_not_enable_while_second_model_flaps`, `llm.rs:1861`; `explicit_models_are_never_rewritten_or_fallback_retried`, `llm.rs:2136`).
- Two tests are explicitly written to be mutation-sensitive and say so in their doc comments: `terminal_llm_error_below_threshold_emits_no_stall_warning` (`llm.rs:3418-3425`) and `terminal_llm_error_at_threshold_emits_one_stall_warning` (`llm.rs:3431-3438`).

Integration-style tests that need a real socket use `tokio::net::TcpListener` bound to `127.0.0.1:0` with a hand-rolled HTTP responder rather than a mock-HTTP crate — now six instances (`llm.rs:1662`, `llm.rs:3122`, `llm.rs:3187`, `llm.rs:3251`, `llm.rs:3579`, reused by `llm.rs:3650`/`3686`/`3717`/`3748`). The responder logic is duplicated across three of them (each re-writes the "read to `\r\n\r\n`, then write a 200 with a tiny JSON body" loop) while the auth tests share `spawn_auth_stub` (`llm.rs:3579-3632`) and the mesh tests share `spawn_sequence_stub` (`llm.rs:1662-1745`). Two stub factories now coexist with overlapping responsibilities and no shared code — `spawn_sequence_stub` is strictly more capable (queued responses, captured request bodies, per-path assertions), so the older responders are the ones now out of step.

Two conventions remain violated in the test module, one of them narrowed by the mesh work:
- `llm_with` (`llm.rs:3634-3648`) still constructs `Llm` by struct literal, bypassing `Llm::new` and using a *different* client configuration (`.timeout(5s)` at `llm.rs:3637` instead of `connect_timeout` + `read_timeout`). It also had to be updated for the new field (`mesh_auto_state`, `llm.rs:3641`) — which is exactly the maintenance cost a struct-literal fixture incurs. The blast radius is smaller than it was: the ten mesh tests go through `Llm::new` (e.g. `llm.rs:1837`), so the production wiring at `llm.rs:109-113` is now exercised somewhere, but the auth suite still is not.
- `valid_effort_values_for_provider_model` (`config.rs:2567-2658`) is a test-local re-implementation of production routing rather than a call into production code. See Debt.

A third convention gap is specific to the new tests: they manipulate private state directly to make time pass — `expire_mesh_catalog_check` (`llm.rs:1806-1809`) and `expire_mesh_auto_cooldown` (`llm.rs:1811-1815`) reach into `llm.mesh_auto_state` and backdate `last_checked`/`cooldown_until`. The crate already enables `tokio`'s `test-util` feature (`Cargo.toml:51`), so `tokio::time::pause`/`advance` was available; hand-editing the state instead means the TTL and cooldown *durations* are never actually exercised, only the branches they guard.

#### Comment-quality note
Comments in this group are unusually forthcoming about *why* rather than *what*, and several record negative knowledge that would otherwise be lost — e.g. why an empty Anthropic assistant turn is skipped instead of placeholder-filled (`llm.rs:709-712`), why images are batched behind the tool-result run (`llm.rs:770-777`), why `sum_usage` distinguishes "no usage reported" from "usage was zero" (`llm.rs:1102-1106`), why a mixed `*,foo` hook list is not treated as a wildcard (`config.rs:1104-1110`). The mesh code sustains that standard: `llm.rs:452-456` explains why a failed catalog probe must not be read as absence and why inference stays authoritative, `llm.rs:1337-1342` explains why `PostError` exists rather than a new `AgentError` variant, and `llm.rs:1341-1342` records the asymmetry between an adaptive `auto` call and an explicit `mesh` call. Two comments carry dated provenance ("doc-verified, July 2025") which makes staleness auditable: `config.rs:11`, `config.rs:101`, `config.rs:194`, `config.rs:315`, `config.rs:578`, `config.rs:591`, `config.rs:616`, `config.rs:619`. The mesh constants carry no such provenance, which matters most for `MESH_MOA_UNAVAILABLE_MESSAGE` (`llm.rs:34`) — a verbatim copy of an upstream message with no date and no source reference.


## Module: buzz-agent — MCP registry, OAuth, hints & catalog (`crates/buzz-agent/src`)
### Aspect: Conventions

#### Error handling

One error type for everything: `crate::types::AgentError` (`types.rs:203-211`). All five files return `Result<_, AgentError>` and never define a local error enum — `grep -n 'thiserror\|impl std::error::Error' mcp.rs auth.rs hints.rs builtin.rs catalog.rs` returns zero matches.

Variant selection is by *module* rather than by *cause*: `mcp.rs` uses `Mcp` for all 23 failure sites, `auth.rs` and `catalog.rs` use `Llm` for all 32 of theirs, and only three OAuth sites reach for the semantically meaningful `LlmAuth` (`auth.rs:341`, `355`, `416`). The practical effect is that a missing OAuth token is distinguishable on the wire (`-32001`) but a broken discovery endpoint is not (`-32000`, same as a provider outage) — mapping at `types.rs:249-256`.

Error message style is `"<stage> <subject>: <cause>"`, built with `format!` and interpolated source errors: `spawn {name}: {e}` (`mcp.rs:738`), `init {name}: {e}` (`mcp.rs:760`), `oauth discovery: {e}` (`auth.rs:166`), `Databricks model discovery HTTP {status}: {body}` (`catalog.rs:152-154`). Messages are written for the model as much as for the operator — several include a remediation hint: "Try again later or use a different tool" (`mcp.rs:143`), "run `buzz-agent auth databricks` first" (`auth.rs:417`), "Available: {available:?}" (`builtin.rs:62`, `builtin.rs:130`, `builtin.rs:168-170`).

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

Observability gap: `hints.rs`, `builtin.rs`, and `catalog.rs` contain **no** logging at all (`grep -n 'tracing::' hints.rs builtin.rs catalog.rs` → zero matches, re-run after `8eb6e3eb`). So a skipped skill, a truncated hint chain, a 20-page catalog cut-off, and a silent 32 KiB `load_skill` truncation are all invisible in the log. `8eb6e3eb` added two more silent outcomes to that list rather than closing the gap: an endpoint dropped by the chat-capability name heuristic (`catalog.rs:370-373`) and a `created_timestamp` that failed to parse and therefore sorts last (`catalog.rs:320-325`).

Naming convention for the `killpg` "stage" argument is a short kebab/snake label: `"drop"` (`mcp.rs:119`), `"spawn_dropped"` (`mcp.rs:748`), `"kill_server"` (`mcp.rs:446`), `"call_failed"` (`mcp.rs:465`) — and the reasons passed to `kill_server` are human sentences: `"tool timeout"` (`agent.rs:534`), `"hook timeout (consecutive)"` (`mcp.rs:398`).

#### Naming

- `*_impl` suffix marks the dependency-injected inner function that the public wrapper calls with real environment values: `load_hint_files_impl`, `discover_skills_impl`, `build_hints_section_impl`, `collect_supporting_files_impl` (`hints.rs:40`, `204`, `223`, `166`). This is how `hints.rs` reaches ~90% unit-test coverage without touching `$HOME`.
- `MAX_*` for byte/count ceilings (`mcp.rs:20-27`, `hints.rs:6-7`), `*_BYTES` when the unit is bytes, `*_TIMEOUT` for durations (`auth.rs:35`, `auth.rs:39`).
- `qname` vs `bare` is used consistently for qualified and unqualified tool names (`mcp.rs:154-157` onwards).
- `try_*` marks the fallible non-interactive variant (`try_bearer_no_browser`, `auth.rs:367`).
- Test helper names read as assertions of intent (`make_skill_with_files`, `builtin.rs:262`; `seed_cache`, `tests/databricks_oauth.rs:99`).

#### Doc-comment discipline

Prose-heavy where a decision needed justifying — `ResultBudget` (`mcp.rs:28-32`), `truncate_middle` (`mcp.rs:880-885`), `call_hooks` (`mcp.rs:307-314`), `refresh_now` (`auth.rs:56-73`, `auth.rs:303-316`), `try_bearer_no_browser` (`auth.rs:361-366`), `discovery_failure_fallback` (`catalog.rs:35-50`), `parse_v1_endpoints` (`catalog.rs:166-170`). Several comments explain *why not*, which is the useful kind: "a degraded guard is worse than no guard" (`builtin.rs:176-177`), "prefer including over silently dropping" (`catalog.rs:169-170`), "Hooks are intentionally non-cancellable" (`mcp.rs:342-344`).

`8eb6e3eb` kept to that convention and extended it: all four new items carry rationale-first doc comments — `is_chat_capable_endpoint` explains why the signal is the name and why the heuristic is deliberately narrow (`catalog.rs:82-96`), `sort_v2_endpoints_newest_first` explains the two-phase gateway ordering it corrects and why the name tiebreak exists (`catalog.rs:327-336`), `endpoint_created_ms` names the wire shape it tolerates and why (`catalog.rs:315-319`), and `V2Endpoint::created_ms` documents the `None`-sorts-last decision at the field (`catalog.rs:310-311`). Two inline comments follow the same "why not the obvious thing" pattern: the segment-match rationale (`catalog.rs:102`) and the `resolve_model` trim note (`catalog.rs:52-54`).

Gaps against the `AGENTS.md` rule "New public API must have doc comments": `StaticTokenSource::new` (`auth.rs:79`), `PkceOAuthTokenSource::new` (`auth.rs:144`), `McpRegistry` and four of its methods (`mcp.rs:159`, `172`, `266`, `272`, `286`, `485`), `build_hints_section` (`hints.rs:219`), `SkillEntry` (`hints.rs:14`), `MAX_SKILL_BODY_BYTES` (`hints.rs:7`), `LOAD_SKILL_TOOL` (`builtin.rs:13`). Module-level `//!` docs exist for `auth.rs:1-18`, `builtin.rs:1-5`, and `catalog.rs:1-11`, but not for `mcp.rs` or `hints.rs` — the two files that most need an orientation paragraph.

#### File-size discipline

| File | Lines | Test share |
|---|---|---|
| `mcp.rs` | 1,139 | tests start `mcp.rs:1005` (≈12%) |
| `auth.rs` | 845 | tests start `auth.rs:632` (≈25%) |
| `hints.rs` | 726 | tests start `hints.rs:265` (≈64%) |
| `builtin.rs` | 575 | tests start `builtin.rs:240` (≈58%) |
| `catalog.rs` | 631 | tests start `catalog.rs:396` (≈37%) |

`catalog.rs` grew from 402 to 631 lines in `8eb6e3eb` (260 insertions, 31 deletions), and most of the growth was tests — the production region gained 94 lines while the in-file test module went from 101 lines to 236, lifting the test share from ≈25% to ≈37%. `AGENTS.md` documents a 1,000-line ceiling only for `mobile/` (enforced by `mobile/scripts/check-file-sizes.mjs`), and `desktop/web` have their own guards. No equivalent guard exists for Rust: `grep -rn 'check-file-sizes' justfile` matches exactly one line, inside the mobile recipe (`justfile:617`). `mcp.rs` at 1,139 lines (production part ~1,000) would trip the mobile threshold, and `config.rs` in the same crate is 2,701 lines — so the convention is real but unenforced on this side of the repo.

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


## Module: buzz-dev-mcp (`crates/buzz-dev-mcp`)
### Aspect: Conventions

#### Tool-definition pattern

Every tool follows the same three-part shape:

1. A params struct in the tool's own module, deriving
   `#[derive(Debug, Deserialize, JsonSchema)]` — e.g. `ShellParams`
   (`crates/buzz-dev-mcp/src/shell.rs:119-128`), `ReadFileParams`
   (`read_file.rs:9-21`), `StrReplaceParams` (`str_replace.rs:12-23`),
   `ViewImageParams` (`view_image.rs:63-77`), `TodoParams` (`todo.rs:32-38`).
   Optional fields are `Option<T>` with `#[serde(default)]`.
2. A free `run` function in that module taking `&SharedState` plus the params —
   `shell::run` (`shell.rs:130`), `read_file::run` (`read_file.rs:23`),
   `str_replace::run` (`str_replace.rs:25`), `view_image::run`
   (`view_image.rs:88`). `todo` deviates: it is a method on `TodoState`
   (`todo.rs:71`).
3. A thin `#[tool]`-annotated method on `DevMcp` in `lib.rs` that unwraps
   `Parameters(p)` and delegates (`lib.rs:40-125`). No business logic lives in
   `lib.rs`.

Tool descriptions are long, prose, and operationally specific — they name
defaults, caps, and the helper binaries on `PATH` (`lib.rs:42`, `lib.rs:54`,
`lib.rs:65`, `lib.rs:76`, `lib.rs:87`). Hook tools are prefixed with an
underscore to mark them as agent-lifecycle rather than user-facing: `_Stop`,
`_PostCompact` (`lib.rs:102`, `lib.rs:115`).

Shared plumbing is centralised rather than duplicated: `paths::read_text_file` is
the single resolve→stat→size-check→read→UTF-8 pipeline used by both `read_file`
and `str_replace` (`paths.rs:102-180`), and `paths::resolve_path` is the single
path resolver (`paths.rs:20-45`).

#### Error handling

Two distinct failure channels, used inconsistently across tools:

| Channel | Meaning | Used by |
|---|---|---|
| `Err(ErrorData::invalid_params(msg, None))` | caller's fault | all four file/shell tools' argument validation (`shell.rs:136`, `read_file` via `paths.rs:113`, `str_replace.rs:27`, `view_image.rs:109-111`) |
| `Err(ErrorData::internal_error(msg, None))` | server/IO fault | `paths.rs:118`, `paths.rs:147`, `paths.rs:166`, `paths.rs:171`, `str_replace.rs:91`, `view_image.rs:141` |
| `Ok(CallToolResult::error(vec![Content::text(...)]))` | in-band tool failure the model should read | `shell` no-shell / spawn-fail / cancelled (`shell.rs:163`, `shell.rs:186`, `shell.rs:237`); all `todo` validation failures (`lib.rs:93-97`, `todo.rs:245-248`) |

So `todo`'s validation errors are in-band text while `str_replace`'s are protocol
errors — the same class of user mistake surfaces differently depending on the
tool.

No `panic!`, `unwrap()`, or `expect()` exists in any production path. Production
code uses only infallible fallbacks: `unwrap_or("")` (`lib.rs:143`),
`unwrap_or(DEFAULT_TIMEOUT_MS)` (`shell.rs:143`), `unwrap_or(2)` (`rg.rs:28`),
`unwrap_or(0)` (`shim.rs:203`, `tree.rs:183`), `unwrap_or(DEFAULT_MAX_DIM)`
(`view_image.rs:91`). Every `expect(` and bare `unwrap()` in the crate is inside
`#[cfg(test)]`.

Mutex poisoning is never propagated as a panic: the crate uses
`Err(p) => p.into_inner()` uniformly (`shell.rs:66-69`, `shell.rs:972-975`,
`todo.rs:59-62`).

Failures that would be cosmetic are deliberately swallowed with `let _ =`:
`killpg` results (`shell.rs:709`, `shell.rs:720-722`), artifact deletion
(`shell.rs:977`), permission restore after atomic write (`str_replace.rs:135`),
artifact dir creation (`shim.rs:246`), `rustls` provider install (`lib.rs:166`),
and the `tree` truncation writes (`tree.rs:123`, `tree.rs:132`).

Diagnostics that must reach the operator but not the model go to stderr:
`eprintln!` for shim key problems (`shim.rs:94-98`, `shim.rs:109-111`,
`shim.rs:116-118`) and `tracing::{error,warn,debug}` elsewhere (`rg.rs:157-159`,
`tree.rs:21`, `tree.rs:27`, `shell.rs:229`, `shell.rs:233`,
`view_image.rs:302`, `view_image.rs:314`). The subscriber is explicitly pinned to
stderr with ANSI off (`lib.rs:174-177`) so stdout stays a clean MCP framing
channel.

#### Output formatting

- Structured results are JSON via `serde_json::json!` + `to_string_pretty`, wrapped
  in one text content block (`shell.rs:309-321`).
- Human-readable results are plain strings with an explicit machine-parseable
  header line: `"{path} (lines {a}-{b} of {n})"` (`read_file.rs:50-53`),
  `"Replaced N occurrence(s) in {path}."` (`str_replace.rs:97-106`).
- Truncation is always announced in-band with a bracketed marker:
  `"[truncated: showing last … bytes; … lines / … bytes total …]"`
  (`shell.rs:946-952`), `"[diff truncated]"` (`str_replace.rs:149`),
  `"[showing lines … use offset=… to continue]"` (`read_file.rs:59-63`),
  `"[truncated]"` (`tree.rs:123`, `tree.rs:132`). The `rg` fallback is the
  exception — it caps silently and only logs (`rg.rs:152-166`).
- Byte sizes are humanised via a local `human_bytes` (`view_image.rs:665-675`).
- Limits are always named in error text alongside the offending value
  (`paths.rs:133-141`, `str_replace.rs:33-36`, `view_image.rs:154-159`).

Every numeric limit is a named `const` at the top of its module rather than an
inline literal: `shell.rs:16-24` (9 consts), `view_image.rs:31-53` (8),
`rg.rs:5-9` (5), `tree.rs:6-9` (4), `str_replace.rs:9-10` + `:140` (3),
`todo.rs:20-21` (2), `paths.rs:15` (1), `read_file.rs:6` (1).

#### Platform-conditional code style

Platform divergence is handled with paired `#[cfg]` functions of identical
signature rather than inline branching: `resolve_bash`
(`shell.rs:363-388` / `shell.rs:409-459`), `set_process_group`
(`shell.rs:674-680`), `KillGroup` in three variants
(`shell.rs:694-736` / `shell.rs:738-844` / `shell.rs:846-857`),
`write_keyfile_atomic` (`shim.rs:134-149`), `set_owner_only`
(`shim.rs:218-229`), `symlink` (`shim.rs:231-242`), `is_executable`
(`rg.rs:62-73`), `configure_no_window` (`lib.rs:189-212`). The
non-`unix`/non-`windows` `KillGroup` stub exists purely to keep the crate
compiling everywhere (`shell.rs:846-857`).

`is_windows_apps_alias` is gated `#[cfg(any(windows, test))]` (`shell.rs:574`) so
its six predicate tests run on every host (`shell.rs:1083-1137`) — a deliberate
pattern for making Windows-only logic testable on CI runners.

Comments are unusually dense and explain *why*, often citing the bug they fixed:
the MSYS translation rationale (`paths.rs:21-27`), the three MSYS forms and why
form 3 is not guessed (`paths.rs:47-59`), the `0x8007072c` WSL spawn failure that
motivated case-insensitive `is_under_dir` (`shell.rs:595-601`), why the
`sleep 5` timeout test is not `sleep 999` (`shell.rs:1030-1035`), and why
`GifDecoder::into_frames()` must not be used (`view_image.rs:415-422`).

#### Test organisation and counts

All tests are inline `#[cfg(test)] mod tests` at the bottom of each source file.
There is no `tests/` directory in the crate and no integration-test target.

| File | Test count | Notes |
|---|---|---|
| `todo.rs` | 25 | `todo.rs:251-558`; the densest coverage in the crate |
| `shell.rs` | 24 | 3 behavioural + 6 cross-host predicate tests in `mod tests` (`shell.rs:984-1137`), plus 15 in `#[cfg(all(test, windows))] mod windows_resolver_tests` (`shell.rs:1139-1503`) that never run on Linux/macOS CI |
| `view_image.rs` | 18 | `view_image.rs:678-1136` |
| `paths.rs` | 9 | 1 cross-platform, 8 in `#[cfg(windows)] mod windows_msys` (`paths.rs:215-278`) |
| `read_file.rs` | 8 | `read_file.rs:66-234` |
| `str_replace.rs` | 7 | `str_replace.rs:220-356` |
| `rg.rs` | 6 | `rg.rs:432-491` — parser and glob only |
| `lib.rs` | 0 | — |
| `shim.rs` | 0 | — |
| `tree.rs` | 0 | — |
| **Total** | **97** | of which 23 are `#[cfg(windows)]`-gated, so 74 run on a Unix CI host |

Test naming is descriptive-assertive (`rejects_duplicate_text_after_trim`,
`windows_apps_alias_first_real_bash_second_returns_real`,
`pixel_count_cap_rejects_decompression_bomb`,
`relay_media_url_matches_relay_host_only`). Shared fixtures are duplicated per
module rather than extracted: an identical `fn make_state(cwd)` helper appears
four times (`shell.rs:991-994`, `read_file.rs:74-77`, `str_replace.rs:231-234`,
`view_image.rs:684-687`).

Windows env-mutating tests serialise on a module-local
`static ENV_MUTEX: Mutex<()>` held for the whole test body
(`shell.rs:1144-1149`), with explicit save/restore of `SystemRoot`, `BUZZ_SHELL`,
and `GIT_BASH` (`shell.rs:1209-1219`, `shell.rs:1341-1356`).

Several tests synthesise fixtures byte-by-byte rather than allocating real data:
`synth_oversized_png` hand-rolls IHDR/IDAT/IEND with a local CRC32 so a 9000×9000
declaration can be tested without an 81 Mpx buffer (`view_image.rs:1001-1040`,
used by `pixel_count_cap_rejects_decompression_bomb` at `view_image.rs:987-992`),
and `webp_scan_detects_animated_via_vp8x_flag` hand-builds a VP8X header
(`view_image.rs:966-985`).


## Module: buzz-cli — dispatch, relay client & validation (`crates/buzz-cli/src`)
### Aspect: Conventions

#### Error handling

`thiserror` enum + `?` propagation throughout; every fallible function returns
`Result<_, CliError>` (`error.rs:1-45`). Conversion from `reqwest::Error` is
`#[from]` (`error.rs:15-16`); everything else is mapped explicitly at the call
site, e.g. `Tag::parse(...).map_err(|e| CliError::Other(format!("tag error: {e}")))`
(`client.rs:93-95`). SDK errors get a single mapping helper so exit codes stay
consistent: `sdk_err` sends `InvalidInput` → `Usage` (1) and everything else →
`Other` (4) (`validate.rs:155-160`), and it is used 20 times in `commands/`
(`grep -rn 'sdk_err' commands/ | wc -l` → 20).

Error text is deliberately machine-shaped: one JSON object per failure with
`error` (category), `message`, `retryable` (`error.rs:127-136`). The
`fmt_reqwest_error` helper walks the full `source()` chain and de-duplicates
substrings so network failures carry the real cause rather than
`error sending request` (`error.rs:49-61`; test
`network_display_includes_detail_beyond_prefix`, `error.rs:225-235`).

#### Output discipline

Exactly two output statements exist in the whole group:
`println!` in `print_create_response` (`client.rs:1402`) and `eprintln!` in
`print_error` (`error.rs:135`) — verified by
`grep -n 'println!\|eprintln!\|print!(' lib.rs client.rs validate.rs error.rs agent_management.rs main.rs`.
Payload goes to stdout, diagnostics to stderr, both single-line JSON. Human
prose reaches the terminal only through clap's own help rendering
(`lib.rs:50`).

#### Logging

None. `grep -rn 'tracing\|log::' lib.rs client.rs validate.rs error.rs agent_management.rs main.rs`
returns zero matches, and no logging crate is declared in `Cargo.toml`. The
upside is that no secret can reach a log sink from this layer; the downside is
that retries, endpoint fallbacks and cursor pagination are invisible — a caller
sees a 60-second hang with no way to observe the three attempts and two sleeps.

#### `unsafe`, lint attributes, panics

- **No `unsafe`**: `grep -n 'unsafe' lib.rs client.rs validate.rs error.rs agent_management.rs main.rs`
  → zero matches, satisfying the AGENTS.md rule.
- Only two `#[allow(...)]` attributes in the group, both `dead_code` on unused
  public methods (`client.rs:567`, `client.rs:802`) — used in place of deletion
  or a tracking marker.
- `AGENTS.md` forbids new `unwrap()`/`expect()` on production paths. Remaining
  violations, all in `client.rs`:
  `advance_query_cursor`'s `.expect("a full query page always has a last event")`
  (`client.rs:504-506`) and `extract_p_tags`'s `t.as_array().unwrap()`
  (`client.rs:1379`). Both are locally justified (the caller only calls
  `advance_query_cursor` on a full page; the filter already matched `as_array`),
  but both are reachable panics on malformed relay data rather than `?`.
- Three `unreachable!` sites: loop-exhaustion guards in `with_retry_body`
  (`client.rs:679`) and `submit_moderation_event` (`client.rs:1018`), and the
  `Cmd::Pack(_) => unreachable!("handled above")` arm (`lib.rs:1791`) that pairs
  with the early return at `lib.rs:1737-1743`.
- `lib.rs:1948-1957` uses `.expect("repos command")` / `.expect("repos protect command")`
  but inside `#[cfg(test)]`, which is idiomatic.

#### Naming

Command enums are `<Group>Cmd` (`AgentsCmd`, `MessagesCmd`, …, `lib.rs:260-1731`);
value enums are noun-shaped with explicit `#[value(name = "…")]` kebab-case wire
names (`lib.rs:101-172`); relay-facing verbs are `submit_*`/`query_*`/`get_*`;
predicates read as questions (`is_moderation_kind`, `is_safe_media_ext`,
`resp_was_success`, `is_stored_event_exhaustion_ambiguous`). Validators are
`validate_*` when they return `()` and `parse_*` when they return the value — a
distinction called out explicitly in the doc comment on `parse_uuid`
(`validate.rs:15-18`). One naming defect: `validate_hex64`'s doc says
"64-character lowercase hex string" (`validate.rs:28`) but the body accepts
uppercase via `is_ascii_hexdigit` (`validate.rs:30`), while the media path
checker in the same crate enforces lowercase explicitly
(`is_lower_hex_sha256`, `client.rs:260-262`).

#### Doc-comment discipline

Mixed. `client.rs` is exemplary: near-every item carries a `///`, and the long
comments on retry/idempotency (`client.rs:1024-1039`), `sign_event_unchecked`
(`client.rs:729-742`) and the rustls install (`lib.rs:30-38`) explain *why*, not
what. `lib.rs`'s clap surface is documented for users (every flag has a `///`
that becomes help text, plus 14 `after_help` example blocks), but the public Rust
types are not: `ChannelType`, `ChannelVisibility`, `PresenceStatus`, `EmojiScope`
and 11 of the `*Cmd` enums have no doc comment (`lib.rs:101`, `:118`, `:135`,
`:145`, `:260`, `:348`, `:502`, `:679`, `:698`, `:729`, `:771`, `:802`, `:844`,
`:923`, `:939`). `agent_management.rs` has a module-level `//!` (`:1`) but no
doc comment on any of its five public items. `CliError` variants are documented
individually while the enum itself is not (`error.rs:4`).

#### File-size discipline

`client.rs` is 2,477 lines and `lib.rs` 2,035 — both far past the 1,000-line
ceiling the repo enforces for mobile Dart (`justfile:617`,
`mobile/scripts/check-file-sizes.mjs`). There is no equivalent guard for Rust:
`grep -rn 'check-file-sizes' justfile` matches only the mobile recipe. Mitigating
factor for `client.rs`: production code ends at line 1,433 and lines 1,434-2,477
(42%) are test modules. `lib.rs` has no such excuse — it is a single flat clap
declaration with no submodule split, and `enum Cmd`'s dispatch match plus 22
subcommand enums live in one file. Largest single function bodies:
`submit_moderation_event` (`client.rs:873-1022`, ~150 lines with 5 distinct
retry branches) and `upload_file` (`client.rs:1100-1227`, ~128 lines including
the legacy-endpoint duplicate block).

#### Test organization

95 tests in the group: `validate.rs` 42, `client.rs` 29 `#[test]` + 14
`#[tokio::test]`, `error.rs` 7, `lib.rs` 4, `agent_management.rs` 3, `main.rs` 0
(counts via `grep -c '#\[test\]'` / `'#\[tokio::test\]'` per file). Convention is
inline `#[cfg(test)] mod tests` at the bottom of each file, with `client.rs`
splitting into four purpose-named modules — `media_download_tests`
(`client.rs:387`), `retry_tests` (`:1434`), `retry_policy_tests` (`:1582`),
`tests` (`:2297`) — and using `// ---- name ----` banner comments to group
assertions (`client.rs:1440`, `:1479`, `:1508`, `:1528`, `:1545`, `:1554`).
Integration-style tests spin a real axum server or raw `TcpListener` and assert
observable behavior (attempt counts, elapsed time, identical request bytes)
rather than internals — see `test_server` (`client.rs:1603-1636`) and
`stored_event_body_loss_is_retried_with_same_event_bytes` (`client.rs:2038-2114`).

Two convention breaks worth flagging:

1. **Tests that re-implement the production rule.** `error.rs`'s
   `json_error_includes_retryable_field_for_network` (`error.rs:197-210`) and
   `json_error_retryable_false_for_usage` (`error.rs:213-221`) rebuild
   `print_error`'s JSON object inline with `serde_json::json!` instead of calling
   `print_error`, so the actual production serializer and its category strings
   (`error.rs:109-126`) are never executed by a test.
   `grep -c 'print_error' <(awk 'NR>=137' error.rs)` finds one hit and it is a
   banner comment.
2. **Production-file helpers that exist only for tests.**
   `parse_retry_in_secs` (`client.rs:172-186`) and `percent_encode`
   (`validate.rs:75-99`) are `#[cfg(test)]`-gated functions in production files,
   each with its own test suite (6 tests at `client.rs:1444-1477`; 5 at
   `validate.rs:277-306`) — coverage on code that never runs in production.

#### Formatting and toolchain conventions

`rustfmt` defaults appear respected (100-col wrapping, trailing commas). Test
attributes use `#[tokio::test]` without flavour arguments, relying on the
`macros` + `rt-multi-thread` features declared at `Cargo.toml:25`. Cargo.toml
groups dependencies with a one-line rationale comment above each
(`Cargo.toml:18-86`) — a good convention undermined by two comments that have
gone stale (see the Integrations aspect).


## Module: buzz-cli — channel, message & social commands (`crates/buzz-cli/src/commands`)
### Aspect: Conventions

#### The dominant handler shape

Nine of the ten files follow one template: `pub async fn cmd_<verb>(client, …) ->
Result<(), CliError>` that validates flags, calls a `buzz_sdk::build_*` builder,
signs via `client.sign_event`, submits, and prints
`normalize_write_response(&resp)`. `channels.rs:864-878` (`cmd_set_channel_topic`)
is the canonical 15-line instance; `join`, `leave`, `archive`, `unarchive`,
`delete`, `add-member`, `remove-member`, `set-canvas`, `dms hide`,
`dms add-member`, `reactions add`, `social publish`, `social set-contacts` are all
the same shape. Reads mirror it with a `serde_json::json!` filter,
`client.query`, and a print.

`notes.rs` deliberately departs: it declares its own kind constant
(`notes.rs:38`), builds events with a local pure `build_set_event`
(`notes.rs:418-469`) instead of the SDK, deserializes into typed `nostr::Event`
(`notes.rs:156-159`), and prints plain-text receipts. Its module doc
(`notes.rs:1-31`) explains the verbs and implementation state; no other file in
scope has a module doc except `channel_templates.rs:1-8`.

#### Error handling

`CliError` variants are used consistently by intent: `Usage` for anything the
caller can fix, `NotFound` for absent resources, `Other` for build/parse/internal
failures, `Conflict` for NIP-33 LWW. Builder failures are wrapped with the builder
name in the message, e.g. `"build_forum_post failed: {e}"`
(`messages.rs:543`), `"build_create_channel failed: {e}"` (`channels.rs:311`) —
consistent across ~24 call sites.

Two inconsistencies:

- `validate.rs:167-173` provides `sdk_err`, which maps
  `SdkError::InvalidInput` → `Usage` (exit 1) and everything else → `Other`
  (exit 4). Only `dms.rs:118` uses it. Every other builder call site wraps the SDK
  error in `CliError::Other` unconditionally, so a user-input SDK rejection (empty
  channel name, bad emoji URL, over-long content) exits **4** instead of 1 —
  e.g. `channels.rs:311`, `messages.rs:543`, `emoji.rs:120`, `reactions.rs:21`,
  `social.rs:35`.
- `resolve_channel_id`'s "event not found" and "no h-tag" cases are `Other`
  (`messages.rs:98`, `:112`) where `NotFound`/`Usage` would match the taxonomy;
  `notes.rs:618` uses `NotFound` for the same situation.

Failure-swallowing is used in exactly one place and is documented as intentional:
`resolve_content_mentions` returns an empty vec on any error
(`messages.rs:124-127` doc, `:144-147`, `:155-158`, `:203-211`). Everywhere else
`?` propagates.

`unwrap_or_default()` on `serde_json::from_str` is the module's idiom for
tolerating a malformed relay body (`channels.rs:233`, `:256`, `:269`;
`messages.rs:297`, `:333`, `:374`, `:412`; `dms.rs:17`; `feed.rs:42`;
`reactions.rs:87`), which turns a parse failure into an empty result set rather
than an error. `notes.rs:156-159` and `emoji.rs:83-84` take the opposite line and
surface a parse error.

#### Output discipline

| Rule | Followed by | Exceptions |
|------|------------|-----------|
| stdout is exactly one line of JSON | most commands | `notes set` prints 5 plain-text lines (`notes.rs:571-580`); `notes rm` prints 2 (`:747-748`); `notes get`/`ls` print **pretty-printed** multi-line JSON (`notes.rs:367-372`, `:380-385`); `notes get --content-only` prints raw markdown (`:661-664`); `canvas get` prints raw markdown (`channels.rs:275`) |
| reads are sig-stripped | via `normalize_events` | `social.rs:78`, `:110`, `:123`, `:207` print the relay body verbatim, including `sig` |
| writes print `{event_id,accepted,message}` | via `normalize_write_response` | `social.rs:180` prints raw; `dms.rs:91` prints a hand-assembled object; `emoji.rs:148-153` prints `{accepted,message}` with no `event_id`; `channels.rs:785` prints a bespoke report |
| stderr is for errors only | `error.rs:135` | `emoji.rs:303` writes `(dry run — not published)` as bare text (not JSON) to stderr; `channels.rs:597` writes a JSON `{"warning":…}` line to stderr |
| absent single resource prints `null` | `channels.rs:239`, `:277` | `notes get` errors with `NotFound` instead (`notes.rs:637`) |

So there are two conventions for "not found on a single-item read" and four
distinct write-response shapes.

#### Naming

- Handlers: `cmd_<verb>_<noun>` in `channels.rs`/`messages.rs`/`dms.rs`
  (`cmd_list_channels`, `cmd_send_message`, `cmd_hide_dm`), but bare `cmd_<verb>`
  in `emoji.rs` and `notes.rs` (`cmd_list`, `cmd_set`, `cmd_rm`, `cmd_ls`), which
  are also `async fn` (private) in `emoji.rs` vs `pub async fn` in `notes.rs`.
- Filter builders: inline `serde_json::json!` literals everywhere; no named filter
  constructors.
- Helper naming for relay reads is inconsistent: `fetch_own_note`
  (`notes.rs:168`), `fetch_own_emoji` (`emoji.rs:93`), `fetch_by_slug`
  (`notes.rs:187`), `fetch_by_coord` (`notes.rs:349`), `fetch_events`
  (`messages.rs:203`), `fetch_member_pubkeys` (`messages.rs:213`),
  `fetch_team_persona_slugs` (`channels.rs:399`), `scan_managed_agents_by_owner`
  (`channels.rs:440`) — `fetch_*` vs `scan_*` for the same operation shape.
- Two different local names for the same idea: `resolve_author` exists in both
  `messages.rs:394` and `notes.rs:204` with different signatures
  (`String` vs `PublicKey`) and different accepted inputs.
- The template-resolution code uses requirement IDs in comments as identifiers
  ("Owner invariant (F1)" `channels.rs:645`, "the F4 cardinality rule"
  `channels.rs:472`) with no in-repo definition of F1/F4 — an external-doc
  reference that cannot be resolved from the codebase.

#### `unsafe`, lint attributes, `unwrap`/`expect`

`grep -rn 'unsafe' crates/buzz-cli/src` returns **zero matches** — the no-`unsafe`
rule holds.

Lint attributes in scope: `#[allow(clippy::too_many_arguments)]` twice
(`channels.rs:654`, `:794`), on `cmd_create_channel_from_template` (8 params) and
`build_template_report` (7 params). No `#[allow(dead_code)]` in these ten files
(`grep -n 'allow(dead_code)'` → zero matches; the two in the crate are in
`client.rs:567` and `client.rs:802`).

Production-path `expect()`/`unwrap()` (AGENTS.md forbids new ones):

| Site | Call | Reachability |
|------|------|-------------|
| `channels.rs:148` | `serde_json::to_string(&matches).expect("serializing ChannelSummary")` | infallible in practice; inconsistent with `unwrap_or_default()` used 4 lines later at `channels.rs:104`/`:108` and at `:236`, `:258` |
| `notes.rs:632` | `name.expect("dispatch enforces --name xor --naddr")` | panics if `cmd_get` (a `pub fn`) is called without `validate_get_args` — the invariant lives in the caller, not the type |
| `channels.rs:314`, `:319`, `:703`, `:708` | `unreachable!()` | guarded by the preceding string match; correct but four copies |

`client.rs:501` also has an `expect("a full query page always has a last event")`
that these files depend on transitively via `query_paginated`.

#### Doc-comment discipline

Strong: `notes.rs` (module doc + every public item, including a "Carry-forward
semantics (ratified)" contract block at `:390-417`), `channel_templates.rs`
(module doc + every item), the template-resolution cluster in `channels.rs`
(`:329-398`, `:466-530`, `:583-654`, `:789-794` — unusually thorough, explaining
fail-open, testability seams, and why members are added sequentially),
`messages.rs`'s NIP-10 helpers (`:19-24`, `:41-56`, `:118-127`).

Weak: 30+ `pub` items with no doc comment, enumerated in the API Surface aspect —
including all nine `dispatch` functions, all `channels` lifecycle handlers, and
the `SendMessageParams`/`SendDiffParams` structs and their fields
(`messages.rs:474-482`, `:581-595`). `mod.rs` has no doc at all (`mod.rs:1-20`).
`reactions.rs`, `dms.rs` and `feed.rs` have per-function docs on some handlers
(`dms.rs:7`, `:50`, `:95`, `:110`; `feed.rs:8`) but none on `cmd_add_reaction`,
`cmd_remove_reaction`, `cmd_get_reactions`.

#### File-size discipline

| File | Total | Production (pre-`#[cfg(test)]`) | Test |
|------|-------|-------------------------------|------|
| `channels.rs` | 1713 | 1175 | 538 |
| `notes.rs` | 1330 | 820 | 510 |
| `messages.rs` | 1167 | 876 | 291 |
| `emoji.rs` | 389 | 325 | 64 |
| `social.rs` | 284 | 238 | 46 |
| `channel_templates.rs` | 195 | 125 | 70 |
| `reactions.rs` / `dms.rs` / `feed.rs` / `mod.rs` | 138 / 136 / 80 / 20 | all | none |

The repo enforces a 1000-line ceiling for mobile, desktop and web
(`justfile:123`, `:585`, `:617` invoking `check-file-sizes.mjs`), but there is no
equivalent guard for Rust — `grep -n 'check-file-sizes\|max-lines' justfile`
matches only those three JS/Dart invocations. `channels.rs` (1713) and `notes.rs`
(1330) exceed that ceiling; `channels.rs` is the largest file in the crate after
`lib.rs` and `client.rs`.

Longest functions: `cmd_create_channel_from_template` (`channels.rs:655-790`, 136
lines, 8 params, 3 sequential write stages), `messages::dispatch`
(`messages.rs:754-875`, 122 lines of struct-shuffling), `channels::dispatch`
(`channels.rs:1066-1167`, 102 lines including branching logic for
`create --template`), `cmd_send_message` (`messages.rs:483-579`, 97 lines doing
stdin, upload, threading, mentions and kind selection), `cmd_set`
(`notes.rs:487-582`, 96 lines), `cmd_import` (`emoji.rs:234-309`, 76 lines with
numbered steps 1–8 in comments).

`channels::dispatch` is the only dispatcher containing business logic rather than
pure delegation: it decides template-vs-plain creation and re-derives the
"required" flags (`channels.rs:1128-1148`).

#### Test organization

All tests are inline `#[cfg(test)] mod tests` at file bottom; there is no
`crates/buzz-cli/tests/` directory. Conventions observed:

- Named, intent-revealing test functions (`cardinality_multiple_instances_is_hard_error_listing_candidates`,
  `channels.rs:1423`; `malformed_root_does_not_shadow_valid_reply`,
  `messages.rs:848`).
- Section banner comments grouping tests (`channels.rs:1292`, `:1384`, `:1468`,
  `:1599`; `messages.rs:988`, `:1108`; `notes.rs:875`, `:1032`, `:1258`).
- Local fixture builders: `event()` (`channels.rs:1192`), `agent()`
  (`channels.rs:1388`), `profile_event()` (`messages.rs:1112`), `build_30023()`
  (`notes.rs:884`), `prior_snapshot()` (`notes.rs:1036`), `write_store()`
  (`channel_templates.rs:131`).
- Explanatory comments stating *why* a case matters, including several that
  document a previously-shipped bug (`messages.rs:995-998` "regression guard for
  the previous stub that always returned `vec![]`"; `channels.rs:1599-1606`
  "deleting the stderr emission … left all of them green").
- Constants for fixture hex values (`messages.rs:882-891`), with an honest
  comment about what `PublicKey::from_hex` actually validates
  (`messages.rs:1085-1088`).

Deviations: `reactions.rs`, `dms.rs`, `feed.rs` have no `#[cfg(test)]` module at
all; `channels.rs:1296-1307` re-implements a production rule inside the test
module (see Business Rules); `channels.rs:1363-1366` mutates process env with
`std::env::set_var`/`remove_var` without serialization, which is unsound if
another test in the same binary ever reads that variable concurrently.


## Module: buzz-cli — repo, agent, memory & moderation commands (`crates/buzz-cli/src/commands`)
### Aspect: Conventions

#### Error handling and `CliError` mapping style

`CliError` (`error.rs:4-49`) has nine variants; exit codes come from `error::exit_code`
(`error.rs:89-107`). Three distinct mapping styles coexist in this group:

| Style | Sites | Effect |
|---|---|---|
| `map_err(sdk_err)` — the shared mapper (`validate.rs:151-156`) | `pr.rs:58`, `:100`, `:209`; `patches.rs:50`, `:169`, `:183`; `issues.rs:29`, `:140`; `workflows.rs:108`, `:130`, `:145`, `:185`, `:207`; `users.rs:299` | `SdkError::InvalidInput` → `Usage` (exit 1); **everything else** → `Other` (exit 4) |
| inline `map_err(\|e\| CliError::Usage(...))` | `moderation.rs:43`, `:54`, `:72`, `:82`, `:98`; `agents.rs:104`, `:134`; `repos.rs:82`, `:105`, `:333` | forces exit 1 regardless of the underlying `SdkError` variant |
| inline `map_err(\|e\| CliError::Other(...))` | `repos.rs:57`, `:124-126`, `:137`, `:220`; `users.rs:206`; `mem.rs:95`, `:356`; `agents.rs:36`, `:78`, `:273`, `:275` | exit 4 — note `repos.rs:124-126` classifies *pre-existing malformed rules on the fetched event* as internal, which is arguably right (not the caller's input) but reads oddly next to `repos.rs:82` |

The consequence of the first style is worth stating plainly: a user handing in a 100 KiB PR
body gets `SdkError::ContentTooLarge` (`builders.rs:35-41`, cap applied at
`builders.rs:1335`), which `sdk_err` routes to `CliError::Other` → **exit 4 ("other")**, not
exit 1 ("input error"). The same over-size input via `moderation resolve` would be exit 1,
because `moderation.rs:98` hardcodes `Usage`. So the same class of user mistake produces two
different exit codes depending on which command it hits. `AGENTS.md` documents exit 1 as
"input error" and 4 as "other".

`?`-propagation is the norm; no `.ok()`-swallowing of relay errors. Deliberate error
suppression is confined to three places and each carries a comment explaining it:
`mem.rs:120-128` (a corrupt event must not deny-of-service a listing),
`mem.rs:161-170` (undecryptable engrams are skipped, not fatal), and
`agents.rs:204-206` (a structurally malformed `auth` tag means "bare", not error).

Fallible-to-default conversions are pervasive on the read path — `serde_json::from_str(&resp)
.unwrap_or_default()` at `users.rs:47`, `users.rs:259`, `workflows.rs:20`, `workflows.rs:45`,
`workflows.rs:78`. These silently turn a malformed relay response into an empty array, so
`workflows list` on a broken relay prints `[]` and exits 0. The stricter modules parse and
error: `mem.rs:115-117`, `repos.rs:12-13`, `agents.rs:280-281`.

#### Output discipline: stdout, stderr, and which helper prints

The group's stated contract (`crates/buzz-cli/README.md:3`) is one-line JSON on stdout,
diagnostics on stderr. Four distinct printing conventions are in use:

| Convention | Commands | Sites |
|---|---|---|
| `println!("{}", normalize_write_response(&resp))` — the shared normalizer (`client.rs:1420-1434`) | all five `moderation` mutations, `users set-profile`, `users set-presence`, `workflows update/delete/trigger/approve` | `moderation.rs:47`, `:57`, `:75`, `:85`, `:101`; `users.rs:212`, `:303`; `workflows.rs:134`, `:148`, `:182`, `:187`, `:210` |
| `print_create_response` (`client.rs:1401-1403`) | `workflows create` only | `workflows.rs:112` |
| bare `println!("{resp}")` — raw relay body, unnormalized | `repos create/get/list`, all four `pr` reads/writes that aren't `protect`, all four `patches`, all four `issues`, all three `moderation` reads | `repos.rs:228`, `:252`, `:280`; `pr.rs:61`, `:103`, `:114`, `:147`, `:212`; `patches.rs:53`, `:80`, `:109`, `:186`; `issues.rs:32`, `:43`, `:76`, `:143`; `moderation.rs:115`, `:121`, `:129` |
| hand-built JSON via `serde_json::json!` | `agents archive/unarchive/archived`, `agents draft-*` (relay body reparsed and augmented) | `agents.rs:108-115`, `:138-145`, `:312`; `agents.rs:35-42`, `:77-84` |

So within one command family the discipline splits: `repos protect set/remove` normalize
(`repos.rs:198` → `validate_write_response` → `normalize_write_response`, `repos.rs:192`)
while `repos create` in the same file does not (`repos.rs:228`). `normalize_events`
(`client.rs:1307-1323`) exists and is used by other command families, but **no file in this
group calls it** — `grep -n 'normalize_events' ` across the eleven returns zero matches;
`workflows.rs` and `users.rs` hand-roll equivalent projections instead
(`workflows.rs:22-31`, `workflows.rs:79-89`, `users.rs:50-62`, `users.rs:260-269`).

Non-JSON stdout — three deliberate exceptions, each documented:

- `mem get` writes the raw value with no trailing newline (`mem.rs:296`, `mem.rs:300`),
  explained at `mem.rs:295` so it round-trips with `mem set <slug> -`.
- `mem ls` without `--json` prints TSV (`mem.rs:268`) and puts the empty-case notice
  `(no memories besides core)` on **stderr** (`mem.rs:265`) leaving stdout empty.
- `pack validate` / `pack inspect` print human-readable text only (`pack.rs:40-42`,
  `pack.rs:66-147`).
- `upload file` is the group's only pretty-printer (`upload.rs:8-11`).

stderr is used for two things: `error::print_error`'s JSON error object
(`error.rs:112-137`) and `mem`'s progress/confirmation lines — `mem.rs:370`, `:671-674`,
`:696`, `:733` — plus `pack`'s diagnostics (`pack.rs:29`, `pack.rs:32`). `mem set`/`rm`
therefore print **nothing to stdout on success**, which makes them the only writes in the
group whose result cannot be piped into `jq`.

#### Naming patterns

| Pattern | Convention | Deviations |
|---|---|---|
| Public handler | `cmd_<verb>[_<noun>]`, `async`, takes `&BuzzClient` first | `agents.rs` has **no** `cmd_*` handlers — all five subcommands are inline arms of `dispatch` (`agents.rs:14-157`); `pack.rs`'s two are sync and take no client (`pack.rs:15`, `pack.rs:52`) |
| Verb/noun order | inconsistent: `cmd_create_repo`/`cmd_get_repo`/`cmd_list_repos` (`repos.rs:202`, `:232`, `:256`) put the verb first, but `cmd_send_patch`/`cmd_patch_status` (`patches.rs:9`, `:114`) and `cmd_open_pr`/`cmd_pr_status` (`pr.rs:20`, `:152`) mix both orders in one file | — |
| Dispatch entry | `pub async fn dispatch(cmd, client)` | `upload.rs` needs two (`dispatch`, `dispatch_media`, `upload.rs:4`, `:17`); `users.rs:307` and `moderation.rs:133` add a third `&OutputFormat` parameter |
| Pure helper | private, snake_case, no `cmd_` prefix — `resolve_owner` (`mem.rs:33`), `sha256_hex` (`mem.rs:377`), `extract_owner_auth_tag` (`agents.rs:191`), `protection_pattern` (`repos.rs:48`), `presence_subject` (`users.rs:279`), `resolve_expiry` (`moderation.rs:26`), `parse_committer` (`patches.rs:61`) | consistent |
| Cross-module helper | `pub(crate)` — `patches::parse_status` (`patches.rs:194`), `agents::fetch_archived_snapshot` (`agents.rs:270`) | consistent |

#### `unsafe` and lint attributes

`unsafe`: **zero occurrences.** `grep -rn 'unsafe'` across the eleven files returns no
matches, matching `AGENTS.md:114`.

Lint attributes: seven, all the same one —
`#[allow(clippy::too_many_arguments)]` at `mem.rs:537`, `pr.rs:19`, `pr.rs:65`, `pr.rs:151`,
`patches.rs:8`, `patches.rs:113`, `issues.rs:80`. No `#[deny]`, no `#[warn]`, no crate-level
`#![...]` in `lib.rs` or `main.rs` (`grep -n '^#!\[' ` on both returns zero). The suppressed
functions have 8-15 parameters (`cmd_open_pr` has 15, `pr.rs:20-36`) — the flag-per-parameter
shape is the group's accepted cost of avoiding a params struct.

#### `unwrap()` / `expect()` / `panic!` / `unreachable!` on production paths

`AGENTS.md:115`: "Do not introduce new `unwrap()` or `expect()` in production paths — use
`?` and proper error types."

Scanning only the region above each file's `#[cfg(test)]` (or the whole file where there is
no test module — `workflows.rs`, `issues.rs`, `moderation.rs`, `pack.rs`, `upload.rs`), there
is **exactly one violation**:

- `agents.rs:299` — `let raw_event = events.into_iter().next().unwrap();`

It is guarded: `agents.rs:294-296` returns early when `events.is_empty()`, so it cannot panic
today. It is also unnecessary — the surrounding function already returns
`Result<Vec<String>, CliError>` and the two lines below it (`agents.rs:300-301`) use `?`. A
`let Some(raw_event) = … else` would cost one line. Zero `expect(`, zero `panic!`, zero
`unreachable!` on any production path in the group. (For contrast, the shared client carries
two: `unreachable!("loop exhausts all RETRY_MAX_ATTEMPTS")` at `client.rs:680` and
`client.rs:1007` — outside this group's files.)

Nothing enforces the rule mechanically. `just clippy` runs
`cargo clippy --workspace --all-targets -- -D warnings` (`justfile:107`), but
`clippy::unwrap_used` and `clippy::expect_used` are allow-by-default and are not enabled
anywhere: there is no `[lints]` table in the root `Cargo.toml`, no `clippy.toml`
(`ls clippy.toml` → "No such file or directory"), and no crate-level `#![deny]`. So
`AGENTS.md:115` is convention, reviewed by humans, not by CI.

`unwrap_or*` is used freely and is not a violation — it is the group's idiom for
best-effort output projection (`users.rs:47`, `workflows.rs:20`, `mem.rs:263`) and for
clock reads (`now_secs`'s `.unwrap_or(0)`, `mem.rs:85`, which would silently date a write to
the epoch if the system clock preceded 1970; `engram::monotonic_created_at` at `mem.rs:363`,
`:689`, `:726` covers the consequence).

#### Doc-comment discipline

`AGENTS.md:116`: "New public API must have doc comments."

| File | `//!` module doc | `pub fn` count | Undocumented `pub fn` |
|---|---|---|---|
| `mem.rs` | 15 lines, `mem.rs:1-15` | 7 | 1 — `dispatch` (`mem.rs:737`) |
| `moderation.rs` | 15 lines, `moderation.rs:1-15` — the best in the group; explains *why* `/moderation/*` is REST | 1 | 1 — `dispatch` (`moderation.rs:133`) |
| `pack.rs` | 3 lines, `pack.rs:1-3` | 2 | 0 |
| `agents.rs` | none | 1 | 1 — `dispatch` (`agents.rs:12`) |
| `repos.rs` | none | 4 | 4 — `cmd_create_repo` (`:202`), `cmd_get_repo` (`:232`), `cmd_list_repos` (`:256`), `dispatch` (`:349`) |
| `users.rs` | none | 5 | 2 — `cmd_set_profile` (`:150`), `dispatch` (`:307`) |
| `pr.rs` | none | 6 | 6 — including `cmd_open_pr` (`:20`) and `cmd_update_pr` (`:66`), whose preceding line is the `#[allow]` attribute, not a doc |
| `patches.rs` | none | 5 | 5 |
| `issues.rs` | none | 5 | 5 |
| `workflows.rs` | none | 9 | 1 — `dispatch` (`:214`); the other 8 are documented |
| `upload.rs` | none | 2 | 2 |

Eight of eleven files have no module-level doc. The three that do (`mem.rs`,
`moderation.rs`, `pack.rs`) are also the three whose *private* helpers are documented most
thoroughly — `verify_hunks_at_declared_position` carries a 17-line rationale
(`mem.rs:383-399`) plus a named known limitation and a PR reference inline at
`mem.rs:422-428` ("see PR #627 review"). `agents.rs` has no module doc but its private
verification helpers are well documented (`agents.rs:163-171`, `:198-205`, `:246-249`,
`:255-269`, `:319-322`). `pr.rs`,
`patches.rs`, `issues.rs` and `upload.rs` are the thin spots: mechanical arg-shuffling code
with almost no prose, and where comments exist they restate the NIP rather than the code
(`patches.rs:152-155`, `issues.rs:118-121`).

#### File-size discipline

`AGENTS.md:533` documents a **1000-line-per-file** ceiling, but scoped to the mobile app and
enforced by `mobile/scripts/check-file-sizes.mjs`. Equivalent guards exist for desktop
(`desktop/scripts/check-file-sizes.mjs`, wired at `desktop/package.json:11` and `:15`) and
web (`web/scripts/check-file-sizes.mjs`, `web/package.json:10`). **No Rust gate exists** —
`just check` (`justfile:95`) composes `fmt-check clippy desktop-check desktop-tauri-fmt-check
desktop-tauri-clippy web-check mobile-check`; none of those runs a line count over
`crates/`. `ls` of `crates/*/scripts` finds no such script.

Current sizes: `mem.rs` 1045, `agents.rs` 718, `repos.rs` 644, `users.rs` 359, `pr.rs` 342,
`patches.rs` 323, `workflows.rs` 243, `issues.rs` 198, `moderation.rs` 165, `pack.rs` 151,
`upload.rs` 36.

So `mem.rs` at 1,045 lines is the one file in the group that would trip the ceiling the
other three platforms enforce. Its production region is 779 lines (`#[cfg(test)]` at
`mem.rs:780`) with 266 lines of tests. The same is true across the group: the largest files
are large because of their test modules — `agents.rs` is 383 production + 335 test
(`#[cfg(test)]` at `agents.rs:383`), `repos.rs` 401 + 243 (`repos.rs:401`).

#### Test organization across the eleven files

All tests are inline `#[cfg(test)] mod tests` at the bottom of the file. Six of eleven files
have no tests at all. There is no `crates/buzz-cli/tests/` directory, and no async test
anywhere — `grep -rn 'tokio::test' crates/buzz-cli/src/commands/` returns zero matches, so
**no `dispatch` arm and no relay interaction in this group is exercised**.

| File | Tests | Focus |
|---|---|---|
| `mem.rs` | 15 (`mem.rs:793`-`:1044`) | `resolve_reader` matrix, `sha256_hex` vectors, diffy behavior, strict-position checker |
| `agents.rs` | 26 (`agents.rs:401`-`:717`) | `extract_owner_auth_tag` (14 cases), `normalize_relay_self_hex` (3), `verify_archived_event` tri-state (9) |
| `repos.rs` | 9 (`repos.rs:423`-`:643`) | protection tag build/update/remove, list JSON, write-response conflict split |
| `users.rs` | 3 (`users.rs:343`, `:349`, `:355`) | `presence_subject` fallback only |
| `pr.rs` | 2 (`pr.rs:334`, `:339`) | `read_optional_body` |
| `patches.rs` | 4 (`patches.rs:284`, `:298`, `:304`, `:319`) | `parse_committer`, `parse_status` |
| `workflows.rs`, `issues.rs`, `moderation.rs`, `pack.rs`, `upload.rs` | **0** | — |

Naming is descriptive-assertion style (`protection_update_preserves_metadata_and_replaces_only_matching_pattern`,
`repos.rs:423`; `archived_state3_lone_malformed_nip70_tag_errors`, `agents.rs:650`) and
several carry provenance comments tying them to a review finding — `F5` at `agents.rs:665-669`,
`F6` at `agents.rs:500-503`, `F7` at `agents.rs:544-547`, "Max's offset-search case" at
`mem.rs:923-927`. Shared fixtures are per-file, not shared: `hex64`/`hex128`
(`agents.rs:390`, `:394`), `signed_repo`/`tag` (`repos.rs:410`, `:418`),
`test_client` (`mem.rs:788`), `build_archived_event` (`agents.rs:559`).

Tests that assert against a **copy** of a production rule rather than calling it:

- `multi_file_header_count` (`mem.rs:1037-1044`) re-implements the multi-file guard inline —
  `single.lines().filter(|l| l.starts_with("--- ")).count()` — instead of exercising the
  production check at `mem.rs:618`. If `cmd_patch`'s predicate changed to `"--- a/"` or moved
  to a helper, this test would still pass. Its comment (`mem.rs:1031-1036`) explains *why the
  rule is what it is* but not why it is duplicated rather than called.
- `diffy_apply_refuses_mismatched_context` (`mem.rs:878`),
  `diffy_apply_succeeds_on_exact_context` (`mem.rs:895`) and
  `diffy_roundtrip_preserves_content` (`mem.rs:915`) test the third-party `diffy` crate, not
  `buzz-cli` code. That is deliberate and stated (`mem.rs:874-876`: "if a future diffy
  upgrade loosens it, this test catches it") — a dependency-pinning test, correctly labelled,
  but it means three of `mem.rs`'s fifteen tests cover no first-party line.
- `protection_update_enforces_repository_rule_limit` (`repos.rs:551`) asserts
  `error.to_string().contains("exceeds max 50")` — the "50" is `buzz-core`'s rule, reached
  through `parse_protection_tags` (`repos.rs:126`). It does call production code, but its
  assertion is coupled to another crate's message text.

By contrast the strongest tests in the group do call the real function and assert on named
failure modes: `strict_position_rejects_offset_slide` (`mem.rs:928`) first demonstrates that
`diffy::apply` *would* slide the hunk, then asserts
`verify_hunks_at_declared_position` refuses — a genuinely discriminating test.

Notable coverage gaps beyond the six untested files: `resolve_expiry` (`moderation.rs:26-32`,
including the unchecked `Timestamp::now().as_secs() + secs` at `moderation.rs:28`),
`cmd_get_workflow`'s `null` branch (`workflows.rs:55`), the `--inputs` hand-rolled trigger
builder (`workflows.rs:170-181`), the `RepoPushRole` → string mapping (`repos.rs:310-314`),
and the `/moderation/*` query-string construction (`moderation.rs:110-112`).

#### Where I am uncertain

- The "undocumented `pub fn`" counts come from checking the line immediately preceding each
  `^pub (async )?fn`. A doc comment separated from its item by a blank line or an attribute
  block would be miscounted; I hand-checked `pr.rs` and `patches.rs` and found the attribute
  case, but did not hand-verify all 51 `pub fn` sites.
- I did not run `just ci`, so the claim that no gate rejects `agents.rs:299` is from reading
  `justfile:95-107` and the absence of lint configuration, not from an observed clean run.

