## Module: buzz-conformance (`crates/buzz-conformance`)
### Aspect: Configuration

This aspect is genuinely thin, and the thinness is the finding: **a subsystem whose entire
purpose is switchable observation has no switch.** There is no environment variable, no config
struct field, no CLI flag, and no Cargo feature that selects a tracer. The only runtime
selector is a hard-coded constructor line.

---

### Cargo features

**None.** `crates/buzz-conformance/Cargo.toml` (35 lines) has `[package]`
(`:1-8`), `[dependencies]` (`:25-29`), and `[dev-dependencies]` (`:33-34`). There is no
`[features]` table — verified by grep. No `#[cfg(feature = ...)]` appears anywhere in `src/`.

Two comments promise a feature that does not exist:

| Claim | Location |
|---|---|
| "Zero cost: the build can omit emission entirely behind a feature" | `src/lib.rs:321-322` |
| "the build can have the compiler eliminate them entirely behind a feature flag if the cost ever shows up in benches" | `crates/buzz-relay/src/conformance/tracers.rs:9-13` |

`buzz-relay`'s dependency is unconditional: `buzz-conformance = { workspace = true }`
(`crates/buzz-relay/Cargo.toml:20`) with no `optional = true` and no feature gate, so the crate
compiles into every relay build.

---

### Environment variables

| Variable | Scope | Read at | Effect |
|---|---|---|---|
| `BUZZ_CONFORMANCE_UPDATE` | test-only | `tests/replay_fixtures.rs:210` (`std::env::var(...).is_ok()`) | when set to anything, `assert_fixture_matches` writes the golden JSONL instead of comparing (`:211-214`) |

That is the complete list. Grep for `CONFORMANCE`, `TRACER`, and `BUZZ_TRACE` across
`.env.example` and `crates/buzz-relay/src/config.rs` returns nothing. `.env.example` has no
tracing/conformance section.

Note the variable is presence-checked, not value-parsed — `BUZZ_CONFORMANCE_UPDATE=0`
regenerates the fixtures just as `=1` does.

---

### How a non-`Noop` tracer would be selected

There is no selection mechanism. The single binding site is:

```
tracer: Arc::new(crate::conformance::NoopTracer),
```
`crates/buzz-relay/src/state.rs:798`, inside `AppState`'s constructor. The field is
`pub tracer: Arc<dyn buzz_conformance::Tracer>` (`:620`), so it is publicly readable and
writable — but grep across `crates/buzz-relay/src/` finds exactly one assignment (`:798`) and
four reads (`handlers/ingest.rs:1383`, `handlers/req.rs:145`, `:356`, `:672`).

Switching to `JsonlTracer` therefore requires one of:

1. Mutating `AppState.tracer` after construction — the path the constructor comment envisions:
   "Conformance tests overwrite this with a JsonlTracer after construction (see test helpers in
   `crates/buzz-test-client` once those land)" (`state.rs:794-797`). Those helpers do not exist;
   `crates/buzz-test-client/tests/conformance_multitenant.rs` never references
   `buzz_conformance` (verified by grep).
2. Editing `state.rs:798` and rebuilding.

`JsonlTracer::create` (`crates/buzz-relay/src/conformance/tracers.rs:37-45`) takes the output
path as a plain `P: AsRef<Path>` argument — no default, no env fallback, no directory
convention. It is never called anywhere in the repo.

---

### Constants that behave like configuration

| Constant | Value | Line | Notes |
|---|---|---|---|
| `SCHEMA_VERSION` | `1` | `src/lib.rs:86` | compared at `src/checker.rs:85`, `:97`; stamped by `TraceStep::new` (`:305`) |
| `ProptestConfig::with_cases` | `128` | `tests/proptest_checker.rs:193` | per-property case count; not overridable via env in this file |
| `POOL` | `3` | `tests/proptest_checker.rs:51` | community/channel pool width, chosen so foreign-vs-resolved collisions are frequent (`:44-49`) |
| `EmitGuard` seam name | `"ingest_event_exited_without_trace"` | `crates/buzz-relay/src/handlers/ingest.rs:1385` | `&'static str`, passed as `kind` |
| projection breach tag | `"row_community_lookup_missing"` | `crates/buzz-relay/src/conformance/mod.rs:250` | `&'static str` |

`PROPTEST_CASES` and the other standard proptest env overrides are honoured by the proptest
library itself, but the crate ships no `proptest-regressions/` directory and no
`proptest.toml` — verified by `ls crates/buzz-conformance` (only `Cargo.toml`, `LIMITS.md`,
`TRACE_SCHEMA.md`, `src/`, `tests/`).

---

### Test-invocation configuration

| Entry point | Command | Line |
|---|---|---|
| `just test-unit` (nextest present) | `cargo nextest run -p buzz-conformance` | `justfile:290` |
| `just test-unit` (fallback) → `scripts/run-tests.sh unit` | `cargo test -p buzz-conformance -- --nocapture` | `scripts/run-tests.sh:98-99` |
| `just ci` | `check test-unit desktop-test …` | `justfile:266` |

Both run all targets (lib + `proptest_checker` + `replay_fixtures`); the justfile comment at
`:286-289` states the intent ("Run all targets (lib + the tests/replay_fixtures.rs integration
test), not just --lib"). No `--ignored` tests exist in this crate, so nothing is gated behind a
second invocation.

`crates/buzz-conformance/LIMITS.md:88-118` documents a three-command CI contract
(`cargo test -p buzz-conformance --lib`, `--test replay_fixtures`, and
`cargo test -p buzz-relay --lib conformance::`) and totals it as "9 + 5 + 2 = 16 tests". The
actual counts are 9 lib + 6 `replay_fixtures` + 7 `proptest_checker` = 22 in this crate, plus 9
in `crates/buzz-relay/src/conformance/mod.rs` and 1 in `handlers/ingest.rs`. The proptest lane
is absent from that doc's command list entirely, and no justfile recipe or GitHub workflow runs
the `-p buzz-relay --lib conformance::` command it calls mandatory (grep `conformance` in
`.github/workflows/` returns nothing).
