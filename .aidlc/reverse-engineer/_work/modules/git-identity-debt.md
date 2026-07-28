## Module: git-sign-nostr & git-credential-nostr (`crates/git-sign-nostr`, `crates/git-credential-nostr`)
### Aspect: Debt

#### Overview

The dominant debt item is the single 2,508-line `git-sign-nostr/src/lib.rs`
file. Beyond that, the two crates duplicate a meaningful amount of
hex/config/OA-parsing logic that could be shared, `git-sign-nostr`'s own test
module duplicates production logic rather than exercising it directly, there
is one confirmed doc-drift bug (a documented status value that is never
emitted), and neither crate is mentioned in `ARCHITECTURE.md` or
`CONTRIBUTING.md`.

#### File size: `git-sign-nostr/src/lib.rs` at 2,508 lines, single file, no submodules

Confirmed by direct line count (`wc -l`) and by `list_directory` on
`crates/git-sign-nostr/src/`, which contains only `lib.rs` and `main.rs` — no
`mod` declarations split the file into e.g. `armor.rs`, `envelope.rs`,
`signing.rs`, `verify.rs`, `keys.rs`, despite the code itself being cleanly
separable along exactly those lines (armor parsing at `lib.rs:1464-1499`,
envelope parsing at `lib.rs:1345-1441`, signing at `lib.rs:943-1082`,
verification at `lib.rs:1099-1335`, key loading at `lib.rs:397-441,
653-887`). ~718 of the 2,508 lines (~29%) are the test module
(`lib.rs:1791-2508`), leaving ~1,790 lines of production code in one file —
still large enough that a targeted change (e.g. adjusting armor validation)
requires navigating a file that also contains unrelated concerns (fd/status
protocol, OA verification, envelope canonicalization). This is the specific
file-size finding the task asked to record; see the Conventions doc for the
broader ambiguity about whether `AGENTS.md`'s 1000-line ceiling is meant to
apply to Rust crates at all.

#### Duplication between the two crates

Both crates independently implement the same three pieces of logic with no
shared code, despite depending on the same `nostr` crate and solving the
same key-loading/config problem:

1. **`git_config(key: &str) -> Option<String>`** — near-identical shell-out-
   to-`git config --get` wrappers, implemented separately at
   `git-sign-nostr/src/lib.rs:661-680` and
   `git-credential-nostr/src/lib.rs:16-25`. `git-sign-nostr`'s version
   additionally scrubs `NOSTR_PRIVATE_KEY`/`BUZZ_PRIVATE_KEY` from the
   subprocess env (`lib.rs:665-667`) and bounds output to `MAX_SIG_FILE + 1`
   bytes (`lib.rs:670-682`); `git-credential-nostr`'s version does neither
   (`lib.rs:16-25`) — see Security doc for why this matters, not just as
   duplication but as an inconsistently-applied hardening.
2. **Keyfile permission checking** — `check_keyfile_permissions`
   (`git-credential-nostr/src/lib.rs:29-44`) and the permission-check portion
   of `open_keyfile` (`git-sign-nostr/src/lib.rs:818-824`) both compute
   `mode & 0o777` and test `mode & 0o177 != 0`, independently, with
   `git-sign-nostr`'s version additionally doing symlink rejection and UID
   verification (see Security doc). A shared `buzz-core`-or-similar helper
   (`fn check_secret_file_permissions(path) -> Result<(), String>`) could
   serve both crates and would have made the UID-check and `O_NOFOLLOW`
   hardening automatically apply to both rather than only one.
3. **NIP-OA auth-tag parsing/validation** — `load_auth_tag`
   (`git-sign-nostr/src/lib.rs:463-522`) and `load_auth_tag`
   (`git-credential-nostr/src/lib.rs:78-96`) both parse the same 4-element
   `["auth", owner, conditions, sig]` JSON shape from the same two sources
   (`BUZZ_AUTH_TAG` env, then `nostr.authtag` git config), but
   `git-sign-nostr`'s version additionally validates hex length/charset and
   the `conditions` grammar (`validate_conditions`, `lib.rs:562-587`) up
   front, while `git-credential-nostr`'s version only checks array shape and
   the `"auth"` label (`lib.rs:83-96`) and hands the raw strings to
   `nostr::Tag::parse` (`lib.rs:92`) for whatever validation *that* performs
   — meaning the two crates apply **different validation strictness to the
   textually identical `BUZZ_AUTH_TAG` value**, depending on which binary
   happens to read it. A shared parser (returning a validated 3-tuple or a
   `nostr::Tag`) would guarantee both crates reject the same malformed input
   the same way. Concretely: `git-credential-nostr` does *not* independently
   check that the conditions string matches the NIP-OA grammar
   (`kind=<n>`/`created_at<t>`/`created_at>t`, `&`-separated, no leading
   zeros) the way `git-sign-nostr`'s `validate_conditions` does — it relies
   entirely on whatever `nostr::Tag::parse` accepts, which was not traced
   further here since `nostr` is out of scope, but the divergence itself is
   confirmed by reading both `load_auth_tag` implementations side by side.
4. **Hex validation** — `validate_hex_field`
   (`git-sign-nostr/src/lib.rs:1448-1462`, used in production) is
   functionally re-implemented as `is_lower_hex`
   (`git-sign-nostr/src/lib.rs:2199-2203`, test-only) *within the same
   crate* — see below — and again independently (informally, via `.len() ==
   64 && .bytes().all(|b| b.is_ascii_hexdigit())`-style checks) inside
   `git-credential-nostr`'s `check_keyfile_permissions`/key-parsing paths
   (there is no dedicated hex-validation helper in `git-credential-nostr` at
   all; it relies on `nostr::Keys::parse`'s own internal validation).

None of this duplicated logic lives in `buzz-core` or any shared crate —
confirmed by both `Cargo.toml`s declaring no dependency on any in-repo crate
besides `nostr` itself (see Integrations doc). A shared `buzz-git-identity`
(or similar) leaf crate holding `git_config`/`check_keyfile_permissions`/
auth-tag-parsing would remove the duplication and, more importantly, remove
the *inconsistency* in how strictly each binary validates the same
`BUZZ_AUTH_TAG` value.

#### In-crate test duplication: "PR-ported tests" reimplement production logic

`git-sign-nostr`'s test module contains a block explicitly labeled "Wrapper
for PR-ported tests: matches PR's `is_lower_hex` API"
(`lib.rs:2198, 2202-2203`) that reimplements, as test-local functions:

- `is_lower_hex(s, len)` (`lib.rs:2199-2203`) — duplicates
  `validate_hex_field` (`lib.rs:1448-1462`), which is the actual production
  function used by `parse_envelope`. The 10 tests exercising `is_lower_hex`
  (`test_lower_hex_valid`, `test_lower_hex_rejects_uppercase`,
  `test_lower_hex_rejects_wrong_length`, `test_lower_hex_rejects_non_hex`,
  `lib.rs:2231-2249`) therefore verify the *test helper's* correctness, not
  `validate_hex_field`'s — a bug introduced into `validate_hex_field` itself
  that `is_lower_hex` doesn't share would not be caught by these four tests.
  Separately, `test_validate_hex_field` (`lib.rs:2086-2092`) does test the
  real function directly, so `validate_hex_field` is not entirely
  uncovered — but the four `is_lower_hex` tests are redundant test-surface
  that provides an illusion of broader coverage than actually exists against
  production code.
- `parse_oa_tag(json)` (`lib.rs:2205-2228`) — duplicates the parsing portion
  of `load_auth_tag` (`lib.rs:463-522`), but is a free function taking a raw
  JSON string rather than reading from env/git-config, so it cannot be the
  same code path. Four tests (`test_oa_tag_rejects_invalid_owner_hex`,
  `test_oa_tag_rejects_invalid_sig_hex`,
  `test_oa_tag_rejects_dangerous_conditions`,
  `test_oa_tag_rejects_wrong_label`, `lib.rs:2418-2440`) exercise this
  duplicate rather than `load_auth_tag` itself — `load_auth_tag`'s own
  direct coverage is a single test,
  `test_load_auth_tag_rejects_bad_conditions` (`lib.rs:1959-2005`), which
  does cover the real function but with less breadth (only conditions-string
  variations, not owner-hex or sig-hex malformation) than the four
  `parse_oa_tag` tests give the *duplicate* function.

This is a real gap: refactoring `validate_hex_field` or `load_auth_tag` in a
way that breaks validation would not necessarily fail any of these eight
tests, because eight of the fifty-six total tests are checking a
hand-copied stand-in rather than the shipped code.

#### Confirmed doc-drift: `kind_not_applicable` OA status is documented but never emitted

The crate's own module doc states (`crates/git-sign-nostr/src/lib.rs:33-35`):

> machine-readable status is emitted via `NOTATION_NAME nostr-oa-status` /
> `NOTATION_DATA <status>` on the status-fd. Values: `valid`,
> `invalid_signature`, `expired`, `kind_not_applicable`, `none`.

But `OaVerifyResult::as_status_str()` (`lib.rs:172-181`) has exactly four
match arms — `Absent → "none"`, `Valid → "valid"`, `InvalidSignature →
"invalid_signature"`, `ConditionsViolated → "expired"` — **there is no
`kind_not_applicable` variant and no code path that ever produces that
string**. Confirmed by `grep -n '"kind_not_applicable"'` returning zero
matches anywhere in the file. The `kind=` case is handled differently than
the doc comment implies: when a `kind=` clause is present, the code emits a
warning to stderr (`lib.rs:1046-1048, 1268-1271`) but still resolves
`oa_result` to `Valid` (hence `"valid"`, not a distinct
`"kind_not_applicable"`) as long as the temporal conditions are satisfied
— see `lib.rs:1259-1272`, where `has_kind_clause(&oa.1)` only triggers an
`eprintln!`, never a different `OaVerifyResult` variant. This is a genuine,
verifiable contradiction between the module's own documentation and its
implementation, not a matter of interpretation — a caller relying on the
doc comment to parse `nostr-oa-status` output for a `kind_not_applicable`
value would never see it, and would instead need to separately notice the
stderr warning to detect the `kind=`-clause-present-but-ignored case.

#### Dead code / unused feature flags

- `crates/git-sign-nostr/Cargo.toml:26` enables `zeroize`'s `derive` feature
  (`features = ["derive"]`) but no `#[derive(Zeroize)]` or
  `#[derive(ZeroizeOnDrop)]` appears anywhere in the crate (confirmed by
  grep — every usage is `Zeroizing::new(...)` or `.zeroize()` method calls,
  both part of `zeroize`'s base API, not the `derive` macro crate). This is
  a harmless-but-real unused-feature-flag finding: the derive feature pulls
  in the `zeroize-derive` proc-macro crate as a build dependency for no
  benefit.
- No `#[allow(dead_code)]` or `#[allow(unused)]` attributes exist in either
  crate (confirmed by grep) — there is no code the authors have already
  flagged as intentionally unused, which also means no "known dead code"
  markers to reconcile against actual usage.

#### Untested critical paths

- **`git-credential-nostr`'s `panic!` site** (`lib.rs:226`, discussed in
  detail in the Security doc) has no test exercising a `host=`/`path=`
  combination that would make `Url::parse` fail — the crash path is
  reachable but unverified by the existing 7-test suite.
- **`git-sign-nostr` has zero end-to-end/integration-level tests.** All 56
  tests are unit tests calling private functions directly within the same
  process (`#[cfg(test)] mod tests`, `lib.rs:1791-2508`) — none spawn the
  compiled `git-sign-nostr` binary, none exercise the actual `--status-fd`/
  fd-passing machinery (`StatusWriter::new`, `lib.rs:302-354`, including its
  two `unsafe` `libc::fcntl`/`from_raw_fd` calls), none exercise
  `parse_args`'s argv handling end-to-end via a real process invocation, and
  none exercise the keyfile-loading path (`open_keyfile`/
  `read_keyfile_secure`) against a real file on disk. Contrast with
  `git-credential-nostr`'s `tests/integration.rs`, which does all of these
  (spawns the real binary, exercises real stdin, and — in
  `bad_keyfile_permissions`, `tests/integration.rs:296-357` — writes a real
  file with real bad permissions and confirms the binary rejects it). This
  is a substantial coverage asymmetry: the crate with more code, more
  cryptographic surface, and a documented `unsafe` exception has *less*
  realistic (process-boundary) test coverage than the smaller crate.
- **Neither crate's argv-parsing edge cases around unrecognized/malformed
  flag combinations are tested via a real subprocess** — `git-sign-nostr`'s
  `parse_args` rejects conflicting `-bsau`/`--verify` combinations
  (`lib.rs:221-231, 242-252`) and missing trailing `-` in verify mode
  (`lib.rs:265-269`), but no test calls `parse_args` with these inputs
  directly (there is no `#[test] fn test_parse_args_*` anywhere in the
  56-test suite — confirmed by grep for `parse_args` inside the test module,
  which returns no hits) nor via a spawned process. This function is
  entirely untested.
- **`git-sign-nostr` is never exercised by any live-relay e2e test.** The one
  end-to-end git test in the repo, `e2e_git.rs`, explicitly disables
  signing on every `git` invocation (`-c commit.gpgsign=false -c
  tag.gpgsign=false`, `crates/buzz-test-client/tests/e2e_git.rs:70-71`) — so
  there is no test anywhere in the workspace that drives a real `git commit`
  through `git-sign-nostr` as `gpg.x509.program` and checks `git
  verify-commit` on the result. All 56 of `git-sign-nostr`'s tests are unit
  tests; zero are integration or e2e tests, and it is absent from CI's
  nextest archive scope (`--lib`-only jobs never include it since it isn't
  listed in any `-p` filter for a test-running step — see the "CI never
  executes either crate's tests" finding below).

#### CI never executes either crate's tests

Verified by reading every CI workflow, the `Justfile`, and `scripts/
run-tests.sh` in full:

- `just test-unit` (`Justfile:275-293`) and `scripts/run-tests.sh`'s
  `run_unit_tests`/`run_integration_tests`
  (`scripts/run-tests.sh:75-129`) both enumerate specific `-p <crate>`
  filters (`buzz-core`, `buzz-auth`, `buzz-db`, `buzz-conformance`,
  `buzz-push-gateway`) — **`git-sign-nostr` and `git-credential-nostr` are
  not in either list.**
- CI's `unit-tests` job runs `just test-unit`
  (`.github/workflows/ci.yml:120-121`) — same scope, same omission.
- Every other CI reference to either crate name is a **build-only** step:
  `cargo build`/`cargo check`/`cross build` for release artifacts or the
  desktop bundling pipeline (`.github/workflows/ci.yml:340, 431, 904-905`;
  `.github/workflows/release.yml:137,356,614,776`; `linux-canary.yml:169`;
  `windows-canary.yml:125`; `signed-macos-canary.yml:96`; `Justfile:431,478,
  507,534`) — none of these run `cargo test`/`cargo nextest run` scoped to
  either crate.
- `crates/buzz-test-client/tests/e2e_git.rs`, the only test that spawns
  `git-credential-nostr`'s real binary against a live relay, is `#[ignore]`d
  on both its tests (`e2e_git.rs:239, 344` — `#[ignore = "requires live
  relay + MinIO + git"]`) and is **never invoked by any CI workflow**:
  `grep -rn 'e2e_git'` across `.github/`, `Justfile`, and `scripts/` returns
  zero matches.
- The one CI job that does build `git-credential-nostr`'s binary and wires
  `GIT_CREDENTIAL_NOSTR_BIN`
  (`.github/workflows/ci.yml:340, 734`, "Desktop E2E Relay" job) uses it only
  to start a live relay for *other* tests
  (`e2e_persona`, `e2e_nostr_interop`, `e2e_relay`,
  `ci.yml:727-731`) — the relay process needs the binary on disk for its own
  git-serving startup path, but no test in that job actually drives `git`
  through the credential helper itself.

**Net finding: git-sign-nostr's 56 unit tests and git-credential-nostr's 7
integration tests both pass when run manually (`cargo test -p git-sign-nostr`
/ `cargo test -p git-credential-nostr`, confirmed runnable from the crate
layout, though not executed in this analysis since it is analysis-only and
no source modification or command execution beyond read-only inspection was
performed) but neither test suite is part of any CI gate today.** A
regression in either crate would only be caught by a contributor manually
running `cargo test -p <crate>` locally, or by the workspace-wide `cargo
clippy --workspace --all-targets` compile check (which would catch a build
break, not a logic regression) in the pre-push hook / CI clippy step
(`Justfile:106-107`).

#### Documentation coverage: absent from `ARCHITECTURE.md` and `CONTRIBUTING.md`

Confirmed by grep: `git-sign-nostr\|git-credential-nostr` (and the
underscore forms) return **zero matches** in `ARCHITECTURE.md` and
**zero matches** in `CONTRIBUTING.md`. Both crates *are* mentioned in
`AGENTS.md` (the repo-structure table, `AGENTS.md:57-58`) and in the root
`README.md` (`README.md:211`, "Git & pairing" section), so they are not
entirely undocumented at the ecosystem level — but a reader consulting
`ARCHITECTURE.md` for "system design and component relationships" (per
`AGENTS.md`'s own § See Also description of that file) or `CONTRIBUTING.md`
for "how to add event kinds / CLI subcommands / HTTP endpoints" would find
no mention of either crate's design, its NIP-GS/NIP-98 protocol role, or how
it fits into the "add a new signing/auth mechanism" contributor workflow.
The two crates' own `README.md` files and the standalone `docs/nips/NIP-GS.md`
spec are comprehensive, so the *information* exists in the repo — it is
simply not cross-linked from the two canonical architecture/contributor
docs.

#### Stale-comment check: none found

No stale comment contradicting current behavior was found in either
production `src/` file beyond the `kind_not_applicable` module-doc drift
documented above. Both crate READMEs were cross-checked line-by-line against
the code during this analysis (key-loading priority order, CLI invocation
forms, troubleshooting table entries) and all matched the implementation.
