## Module: buzz-admin, sprig & countdown-bot (`crates/buzz-admin`, `crates/sprig`, `examples/countdown-bot`)
### Aspect: Conventions

#### Unsafe code and lint attributes

`CONTRIBUTING.md`'s "No Unsafe Code" section states "All crates enforce `#![deny(unsafe_code)]`" and `SECURITY.md`'s Dependency Management section repeats "`#![deny(unsafe_code)]` is enforced across all crates — no unsafe Rust." This module group only partially matches that claim:

| Crate | `#![deny(unsafe_code)]` present? | Evidence |
|---|---|---|
| `buzz-admin` | Yes — `crates/buzz-admin/src/main.rs:1` | first line of the file |
| `sprig` | **No** | `crates/sprig/src/main.rs:1` is `fn main() {` — no crate-level attribute of any kind; confirmed by reading the first 5 lines and by grep for `#!\[` across the file returning no matches |
| `countdown-bot` | **No** | `examples/countdown-bot/src/main.rs:1` opens directly with a `//!` doc comment, no `#![deny(...)]`; same grep confirms no `#!\[` attributes anywhere in the file |

Neither gap is a real safety hole — a repo-wide check for the literal token `unsafe` in `crates/sprig/` and `examples/countdown-bot/` finds no `unsafe` blocks in either (only the word appears nowhere at all, since even `buzz-admin`'s deny attribute is the sole match for "unsafe" in that file). But the *documentation's* claim that the rule is enforced "across all crates" is not literally true for two of these three programs — there is no compiler-enforced guarantee for `sprig` or `countdown-bot`, only the absence of unsafe code today. `examples/` may be reasonably out-of-scope for a crate-wide policy statement, but `sprig` is a workspace member shipped as a release artifact (`.github/workflows/sprig.yml`) and CI does not appear to special-case it.

#### unwrap()/expect() in production paths

`CONTRIBUTING.md`'s Error Handling section: "Do not use `unwrap()` or `expect()` in production code paths. Use `?` or explicit error handling. `unwrap()` is acceptable in tests." Checked against all three main.rs files:

- `buzz-admin`: exactly one `.expect(...)` call, on the rustls crypto-provider install (`main.rs:116`, `.expect("failed to install rustls crypto provider")`). This mirrors the identical pattern in `buzz-relay`'s own `main()` (`crates/buzz-relay/src/main.rs:82-88`, same `.expect(...)` message) — a deliberate, repo-wide convention for this one specific "impossible unless the build is misconfigured" case, not a stray violation. No other `unwrap()`/`expect()` anywhere in the file.
- `sprig`: zero `unwrap()`/`expect()` calls (confirmed by direct grep against the 53-line file).
- `countdown-bot`: zero `unwrap()`/`expect()` calls in the production code (confirmed by direct grep against the 437-line file, outside the test module which itself contains none either) — the example instead threads `anyhow::Result` and `?` consistently, including through helper functions like `required_env` (`main.rs:388-390`) and `parse_bounded` (returns `Result<usize, String>` rather than panicking, `main.rs:311-320`).

All three conform to the letter of the rule; `buzz-admin`'s single `.expect()` is the one repo-sanctioned exception pattern.

#### Error handling style

All three use `anyhow` for top-level error propagation, matching `CONTRIBUTING.md`'s "Use `anyhow` for binary/application-level error propagation":
- `buzz-admin`: every subcommand handler returns `anyhow::Result<i32>` and converts internal errors to exit codes explicitly rather than letting them propagate as process failures with a stack trace — e.g. `cmd_add_member`/`cmd_remove_member` catch validation errors and print a formatted `error: ...` to stderr, returning `Ok(1)` rather than `Err(...)` (`main.rs:159-163,166-171`). This is a deliberate convention (CLI exit-code contract) rather than sloppy error handling, and it's applied consistently across every subcommand.
- `sprig` uses a bespoke `Result<(), String>` instead of `anyhow` (`crates/sprig/src/main.rs:8`) — reasonable given it has zero dependencies and pulling in `anyhow` just for a 53-line file would add a dependency for no benefit, but it is a deviation from the `anyhow`-for-binaries convention stated in `CONTRIBUTING.md`.
- `countdown-bot` uses `anyhow::{anyhow, bail, Context, Result}` throughout (`main.rs:16`), with `.context(...)` calls giving human-readable failure reasons at every environment-parsing step (`main.rs:87-88,101-102,105`) — this is the most idiomatic-`anyhow` file of the three.

#### Logging and tracing

`CONTRIBUTING.md`'s Logging section calls for structured `tracing` fields over string interpolation. `buzz-admin` follows the structured-field convention for its two `tracing` call sites (`tracing::info!(member_count = members.len(), ts, "...")`, `main.rs:378-382`; `warn!("Redis publish ... failed: {e}")`, `main.rs:32,368` — the latter is actually a string-interpolated `warn!`, which is the *discouraged* style per `CONTRIBUTING.md`'s own example of "Avoid: `tracing::info!("Event ingested: ...")`"). More notably, `buzz-admin` never initializes a `tracing_subscriber` anywhere (`main.rs` has no `tracing_subscriber::fmt()...init()` call, unlike `buzz-relay`'s `main()` which sets up JSON structured logging before anything else) — so in a bare `docker compose exec relay buzz-admin ...` invocation, these `tracing` calls have no configured output destination and are effectively silent unless some ambient global subscriber already exists in the process (it doesn't, since `buzz-admin` is its own binary, not embedded in the relay process). `sprig` and `countdown-bot` use plain `eprintln!`/`println!` exclusively for user-facing output and never pull in `tracing` at all — reasonable for a dispatcher and a small example, and consistent within themselves, but a third distinct logging convention within one module group.

#### File-size and module-organization discipline

`AGENTS.md`'s mobile-specific "1000 lines/file" ceiling doesn't apply to Rust crates, and there is no equivalent stated Rust-side limit — but all three files here are single-file crates/examples with no submodules: `buzz-admin` (584 lines), `sprig` (53 lines), `countdown-bot` (437 lines), none of which split logic into separate files the way most other crates in this workspace do (e.g. `buzz-db`'s one-module-per-concern convention noted in `ARCHITECTURE.md`'s crate reference). For `buzz-admin` specifically, cramming CLI parsing, DB connection setup, membership business logic, and NIP-29 event construction into one 584-line `main.rs` with no `mod` declarations is a structural outlier compared to every other operator-facing crate in the workspace's "Ops, Git, Push, Pairing, Test" layer (e.g. `buzz-pairing-cli` is also single-file but at 623 lines for a narrower single-purpose tool). See `ops-misc-debt.md` for the concrete costs of this (a 118-line `reconcile_channels` function, untestable in isolation).

#### Doc-comment discipline

`buzz-admin` has strong doc-comment discipline at the module and function level: a substantial module-level `//!` doc block explaining two non-obvious design decisions (why no kind:8000/8001 deltas, why the same-second bump doesn't serialize concurrent processes — `main.rs:3-19`), and doc comments on every `Command`/`ProductFeedbackCommand` variant (`main.rs:43-96`) that `clap` surfaces directly as `--help` text — this is a good convention: the CLI's `--help` output and the source documentation are the same text, so they cannot drift independently. `sprig` has zero doc comments (`crates/sprig/src/main.rs` contains no `///` or `//!` lines at all — confirmed by inspection of the full 53-line file, only inline `//` comments) though its `print_usage()` function (`main.rs:45-53`) serves the equivalent purpose for end users. `countdown-bot` has a substantial module doc comment (`main.rs:1-13`) plus scattered inline comments explaining non-obvious choices (e.g. why `!fib` counts down, why mention-commands require a `p` tag) but few `///`-style function docs — acceptable for a 437-line example file, less so if it were library code.

#### Naming/kind-constant consistency

Within `buzz-admin`'s own `reconcile_channels` function, kind 39001 is referenced via the named constant `KIND_NIP29_GROUP_ADMINS` (`main.rs:549`) while kinds 39000 and 39002 are hardcoded as bare integer literals (`main.rs:511,562`) even though `buzz-core::kind` defines named constants for both (`KIND_NIP29_GROUP_METADATA = 39000`, `KIND_NIP29_GROUP_MEMBERS = 39002`, `crates/buzz-core/src/kind.rs:362,366`) and `buzz-admin` already imports from `buzz_core::kind` elsewhere in the same file (`use buzz_core::kind::KIND_NIP43_MEMBERSHIP_LIST;`, `main.rs:26`; `use buzz_core::kind::KIND_NIP29_GROUP_ADMINS;` inline at `main.rs:462`). This inconsistency exists only in `buzz-admin`; `countdown-bot` cannot exhibit it since it doesn't depend on `buzz-core` at all and consistently uses `Kind::Custom(<literal>)` for every kind it emits (`main.rs:174,220,236` — kinds 9000, 9, implicitly 9 via `build_message`).

#### Test organization

None of the three follow the workspace's dominant "co-located `#[cfg(test)] mod tests` with substantial coverage" pattern uniformly:
- `buzz-admin`: no test module at all (confirmed: `grep -rn "#\[test\]|#\[cfg(test)\]|#\[tokio::test\]" crates/buzz-admin/` → no matches).
- `sprig`: no test module at all (same grep against `crates/sprig/` → no matches).
- `countdown-bot`: one `#[cfg(test)] mod tests` block at the bottom of `main.rs` (`main.rs:392-436`), following the standard in-file convention used elsewhere in the workspace, covering the three pure reply-string functions with plain `#[test]` (not `#[tokio::test]`, since these functions are synchronous) — this is the one place in the group that matches the repo-wide convention.

See `ops-misc-debt.md` for the significance of `buzz-admin` having no tests given its size and blast radius.
