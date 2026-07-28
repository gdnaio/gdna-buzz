## Module: git-sign-nostr & git-credential-nostr (`crates/git-sign-nostr`, `crates/git-credential-nostr`)
### Aspect: Conventions

#### Overview

The two crates are stylistically consistent with each other (same
error-message tone, same key-loading pattern) but diverge in size, test
organization, and rigor — `git-sign-nostr` is far larger and more heavily
tested than `git-credential-nostr`. Both are plain-function style with no
`clap`, no `thiserror`/`anyhow` (both used elsewhere in the workspace per
`Cargo.toml:82-83`), and no `tracing` — errors are `String`-based and printed
directly to stderr.

#### File-size discipline

`crates/git-sign-nostr/src/lib.rs` is **2,508 lines** in a single file with
no submodules (confirmed: `crates/git-sign-nostr/src/` contains only `lib.rs`
and `main.rs`, no `mod` declarations anywhere in `lib.rs`). This exceeds the
"hard ceiling: 1000 lines/file" rule `AGENTS.md` states for the mobile app
(`AGENTS.md` § Mobile App § Rules) — a rule whose applicability to Rust
crates is itself ambiguous (already flagged workspace-wide at
`.aidlc/reverse-engineer/conventions.md:138`: "`AGENTS.md` states the ceiling
inside the Mobile App § Rules section while also saying 'mirroring
desktop/web', so whether it was ever meant to bind Rust is genuinely
ambiguous"). Whatever the ceiling's intended scope, `git-sign-nostr/src/
lib.rs` at 2,508 lines is one of the larger single files in the workspace
(the existing workspace-wide survey at `.aidlc/reverse-engineer/
conventions.md:138` lists files from 1,045 to 6,570 lines; this file would
rank in the middle of that list, just above `notes.rs` at 1,330). Of those
2,508 lines, **718 lines (1791-2508, ~29%) are the `#[cfg(test)] mod tests`
block** (`crates/git-sign-nostr/src/lib.rs:1791`) — a similar test-share to
`buzz-agent/src/llm.rs`'s ~58% and `config.rs`'s ~59% noted in the
workspace-wide survey, though a smaller fraction. The production code alone
is still ~1,790 lines in one file with no internal module split (no `mod`
boundaries for e.g. armor-parsing vs. signing vs. verification vs.
key-loading, all of which are logically separable).

`crates/git-credential-nostr/src/lib.rs` is 266 lines — well within any
reasonable ceiling, with its 356-line test suite living in a separate
`tests/integration.rs` file (proper separation, unlike `git-sign-nostr`'s
in-file test module).

#### Error handling style

Both crates use bespoke, non-`thiserror` error types:

- `git-sign-nostr` defines a two-variant `enum Error { Fatal(String),
  VerifyFailed { pk: Option<String>, msg: String } }`
  (`crates/git-sign-nostr/src/lib.rs:142-148`) with a manual `Display` impl
  (`lib.rs:150-158`). Every fallible function returns `Result<T, Error>`.
  This is a deliberate two-track model: `Fatal` for anything that should
  simply abort with a stderr message, `VerifyFailed` for verification-
  specific failures that also carry the offending pubkey (when known) for
  `ERRSIG`/`BADSIG` status-line reporting. The top-level `run()`
  (`lib.rs:1726-1791`) pattern-matches both variants identically at the call
  site (`lib.rs:1745-1747, 1756-1758, 1786-1790` — all three arms print
  `eprintln!("error: {msg}")` and return `1`), so the two-variant split
  exists for the *pk* payload, not for differentiated top-level handling.
- `git-credential-nostr` uses plain `Result<T, String>` throughout
  (`load_key() -> Result<String, String>` at `lib.rs:50`,
  `load_auth_tag() -> Result<Option<Tag>, String>` at `lib.rs:78`,
  `check_keyfile_permissions(path: &str) -> Result<(), String>` at
  `lib.rs:29`) — no custom error enum at all. Every error site does
  `eprintln!("error: {e}")` then `return 1` inline (e.g. `lib.rs:171-173,
  211-213, 220-222, 231-233, 243-245, 251-253`), via a small local
  `require!` macro (`lib.rs:166-175`) for the three "must be present"
  string fields.

Both crates share the convention "errors go to stderr, never stdout" (stated
explicitly as a doc-comment rule in `git-credential-nostr/src/lib.rs:150-151`:
"Errors go to stderr only") — enforced by inspection: neither crate has any
`println!`/`print!` call outside the designated success-output blocks
(`git-sign-nostr/src/lib.rs:1063-1066` for the armored signature;
`git-credential-nostr/src/lib.rs:161-164, 258-263` for the credential
response and the blank-line decline).

#### `unwrap()`/`expect()`/`panic!` in production paths

`AGENTS.md` § Quality Gates states: "Do not introduce new `unwrap()` or
`expect()` in production paths — use `?` and proper error types." Checked
both crates' non-test code (production code is all of
`git-credential-nostr/src/lib.rs`, 266 lines, no test module in that file;
and `git-sign-nostr/src/lib.rs:1-1790`, i.e. everything before `mod tests` at
`:1791-1792`):

- `git-credential-nostr/src/lib.rs:226`:
  ```rust
  let parsed_url = Url::parse(&url).unwrap_or_else(|e| panic!("invalid URL {url:?}: {e}"));
  ```
  This is a `panic!` in a genuinely reachable production path — `url` is
  built from `format!("{protocol}://{host}/{repo_path}")` (`lib.rs:206`)
  where `host` and `path` come directly from git's credential-helper stdin
  (ultimately server-influenced, since `host`/`path` mirror the request
  git itself is making). A malformed `host` value (e.g. containing
  whitespace or control characters passed through from a crafted remote
  URL) could make `Url::parse` fail and crash the process instead of
  exiting cleanly with code `1`. No test in `tests/integration.rs` exercises
  a malformed `host=`/`path=` combination that would trigger this specific
  panic — the closest test, `missing_path` (`tests/integration.rs:269-292`),
  covers *absent* `path=`, not a syntactically-present-but-URL-invalid one.
- `git-sign-nostr/src/lib.rs:1-1790`: **zero** `unwrap()`/`expect()`/`panic!`
  occurrences (confirmed by `grep -n 'unwrap()\|expect(\|panic!'` returning no
  matches in that range) — every fallible operation in the production code
  goes through `Result`/`?` or explicit `match`. This crate fully complies
  with the AGENTS.md rule; `git-credential-nostr` has the one exception
  above.

#### `unsafe` code

`AGENTS.md` states "No `unsafe` code" as a blanket workspace rule. Both
crates carry an explicit, self-documented exception:

- `git-sign-nostr` has **6** `unsafe` blocks, all Unix-only fd operations,
  each preceded by a `SAFETY EXCEPTION:` comment explaining the invariant:
  `libc::fcntl(fd, F_GETFD)` (`lib.rs:320`), `fs::File::from_raw_fd(fd)`
  (`lib.rs:334`), the fd-flag-clearing block in `open_keyfile`
  (`lib.rs:813-816`), `libc::getuid()` (`lib.rs:830`), the stack-zeroing
  `ptr::write_bytes` in `do_sign` (`lib.rs:969-972`), and the fd-flag-clearing
  block in `read_bounded_file` (`lib.rs:1600-1603`). The crate's own module
  doc explicitly names this as "an accepted exception to the project's
  no-unsafe rule for this standalone binary" (`lib.rs:49-52`).
- `git-credential-nostr` has **zero** `unsafe` blocks (confirmed by grep) —
  it fully complies with the blanket rule without needing an exception.

Neither crate carries a `#![deny(unsafe_code)]` or similar lint attribute
(confirmed: no `#[allow(...)]` and no `#![deny(...)]`/`#![warn(...)]` in
either crate's `src/`), and the workspace `Cargo.toml` declares no
`[workspace.lints]` table (confirmed: `Cargo.toml` has no `[lints]`/
`[workspace.lints]` section) — so the "no unsafe" rule is enforced by human
review and the doc-comment convention only, not by any compiler gate. `just
clippy` runs `cargo clippy --workspace --all-targets -- -D warnings`
(`Justfile:106-107`) with no `unsafe_code` lint elevation, so a *new* unsafe
block introduced anywhere in either crate would not be caught by CI clippy
either.

#### Doc-comment discipline

Crate-root module docs (`//!`) are present and substantial in both:
`git-sign-nostr/src/lib.rs:1-70` (70 lines covering invocation, GnuPG
protocol, known limitations, ecosystem constraints) and
`git-credential-nostr/src/lib.rs:1-6` (6 lines, much terser). Both comply
with having *a* crate doc.

For the sole public item in each crate (`pub fn run()`), doc-comment
presence differs subtly:
- `git-credential-nostr/src/lib.rs:151`: a single `///` line — "Run the
  credential helper. Returns exit code." plus a second `///` line
  (`:150`? — actually two lines at `:150-151`: "Reads from stdin, writes to
  stdout. Errors go to stderr only." then "Run the credential helper.
  Returns exit code.") immediately above `pub fn run()` at `:152` — this
  **is** a proper doc comment.
- `git-sign-nostr/src/lib.rs:1724-1725`: two `///` lines — "Entry point —
  returns an exit code. This ensures all locals are" / "dropped (and
  zeroized) before `process::exit` is called." — immediately above `pub fn
  run()` at `:1726` — this **is** also a proper doc comment.

Both crates' single public function is properly documented — unlike the
pattern flagged elsewhere in the workspace survey (e.g. `buzz-dev-mcp`'s
`pub fn run()` at `lib.rs:138` being called out as undocumented in
`.aidlc/reverse-engineer/api-surface.md:96`). This module found no doc-comment
violation on the public surface of either crate. Private helper functions are
inconsistently documented in both (many have `///` doc comments describing
behavior — e.g. `load_key`, `load_auth_tag`, `verify_oa` in `git-sign-nostr`
— but this is not required by `AGENTS.md`, which only mandates docs on new
*public* API).

#### Test organization

The two crates use different conventions for the same purpose:

- `git-sign-nostr`: **56** `#[test]` functions (confirmed by `grep -c
  '#\[test\]'`) inside a single in-file `#[cfg(test)] mod tests { ... }`
  block spanning `lib.rs:1791-2508`. Tests are unit-level: they call private
  functions directly (`compute_signing_hash`, `parse_envelope`,
  `build_envelope`, `parse_armor`, `validate_conditions`,
  `enforce_conditions`, etc.) with no subprocess spawning. Two local helper
  functions (`sign_payload` at `lib.rs:2007-2016`, `verify_sig` at
  `lib.rs:2019-2043`) wrap the real signing/verification pipeline for
  round-trip tests. Notably, the last third of the test module
  (`lib.rs:2213` onward, `is_lower_hex`/`parse_oa_tag`/`valid_pk`/
  `valid_sig`/`valid_envelope_json` helpers) is explicitly labeled "Wrapper
  for PR-ported tests" (`lib.rs:2202-2203, 2229`) — local re-implementations
  of validation logic that already exists in production code
  (`is_lower_hex` duplicates the hex-validation logic in
  `validate_hex_field`, `lib.rs:1448-1462`; `parse_oa_tag` duplicates
  `load_auth_tag`'s parsing logic, `lib.rs:463-522`) rather than testing the
  production functions directly — see Debt doc for the duplication this
  creates.
- `git-credential-nostr`: **7** `#[test]` functions in a dedicated
  `tests/integration.rs` (356 lines), each spawning the *actual compiled
  binary* as a subprocess (`env!("CARGO_BIN_EXE_git-credential-nostr")`,
  `tests/integration.rs:17`) and feeding it real stdin, asserting on real
  stdout/stderr/exit-code — this is a genuine black-box integration test,
  not a lighter unit-test-in-disguise. It isolates itself from the host
  machine's git config (`GIT_CONFIG_GLOBAL=/dev/null`,
  `GIT_CONFIG_NOSYSTEM=1`, `HOME=<tempdir>`, `tests/integration.rs:23-25`)
  and always clears `NOSTR_PRIVATE_KEY`/`BUZZ_AUTH_TAG` first
  (`tests/integration.rs:20-22`) to prevent cross-test/host pollution. No
  test in this file exercises the library's private functions directly —
  everything goes through the real binary boundary.

Neither crate has a `benches/` directory or any `#[ignore]`d test —
`git-credential-nostr`'s tests are all fast, hermetic, and always run;
`git-sign-nostr`'s are all pure-function unit tests with no I/O beyond one
`read_bounded_file` negative-path test against `/nonexistent/path`
(`lib.rs:1959-1962`).

#### Logging

Neither crate uses the workspace's `tracing`/`tracing-subscriber` stack
(both listed as workspace dependencies, `Cargo.toml:75-76`, unused by these
two crates). All diagnostic output is `eprintln!`. This is appropriate for
short-lived CLI helpers invoked by git (a logging framework's initialization
cost and structured-output format would be wasted on a process that runs for
milliseconds and exits), and is consistent with `buzz-cli`'s own documented
absence of a logging framework
(`.aidlc/reverse-engineer/api-surface.md:98`: "the crate has no logging
framework at all").

#### Naming and structural symmetry

Both crates independently define a function named `git_config`/
`check_keyfile_permissions` with near-identical signatures and purpose
(`git-sign-nostr/src/lib.rs:661` `fn git_config(key: &str) -> Option<String>`
vs. `git-credential-nostr/src/lib.rs:16` `fn git_config(key: &str) ->
Option<String>`; `git-sign-nostr` additionally has a stricter
`git_config_strict` at `lib.rs:715` with no counterpart in
`git-credential-nostr`) — same name, same shape, independently implemented,
zero shared code. See Debt doc for the consolidation opportunity this
represents.
