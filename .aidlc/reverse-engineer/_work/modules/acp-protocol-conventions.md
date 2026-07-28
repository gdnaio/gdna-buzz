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
