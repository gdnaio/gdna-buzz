## Module: buzz-test-client — Nostr interop & multi-tenant conformance E2E (`crates/buzz-test-client/tests`)
### Aspect: Debt

Ordered by how much each item subtracts from the confidence claims a reader might otherwise place
in this test suite. The CI-execution question is the most important finding in this entire batch
and is addressed first.

---

#### D-TCC-01 — `conformance_multitenant.rs` is compile-checked in CI but never executed; `e2e_nostr_interop.rs` is executed

This is the central finding the task brief asked this agent to establish. Direct evidence:

**`e2e_nostr_interop.rs` runs in CI.** `.github/workflows/ci.yml`, job `relay-e2e` (needs
`desktop-e2e-relay`, gated on `rust` path changes or `push`):

```yaml
- name: Relay E2E tests
  run: |
    cargo test -p buzz-test-client --test e2e_persona --test e2e_nostr_interop -- --ignored --nocapture
    cargo test -p buzz-test-client --test e2e_relay invite -- --ignored --nocapture
    cargo test -p buzz-test-client --test e2e_relay nip43_membership_snapshots_are_rejected -- --ignored --nocapture
  env:
    RELAY_URL: ws://localhost:3000
```
(`.github/workflows/ci.yml:729-735`, job starts `:704`)

This runs against a real relay started by `scripts/start-relay-for-tests.sh --no-build`
(`ci.yml:709-711`) using the artifact built by the `desktop-e2e-relay` job. All 25 of
`e2e_nostr_interop.rs`'s `#[ignore]`d tests are included via `--ignored` with no test-name filter,
so every test in this file this agent analyzed **does run on every push and on every PR that
touches `crates/**`** (subject to the `changes.outputs.rust` path filter, `ci.yml:33-34`,
`:47-50`).

**`conformance_multitenant.rs` does not run in CI, anywhere, ever, as far as this agent could
establish.** Evidence, each independently conclusive:

- `grep -rn 'conformance_multitenant' .github/workflows/` — zero matches across all 12 workflow
  files.
- `grep -n 'conformance_multitenant' Justfile` — zero matches.
- `grep -n 'conformance_multitenant' scripts/run-tests.sh` — zero matches; that script's
  integration-test catch-all is `cargo test --test '*' -- --nocapture` (`scripts/run-tests.sh:118-120`)
  run **without** `-- --ignored`, so even if this catch-all were wired into CI (it is not — see
  below), it would compile the binary but skip every `#[ignore]`d test inside it, i.e. all but one
  (`token_minted_in_a_does_not_authorize_in_b`, the empty-body `#[test]`).
- `scripts/run-tests.sh` itself is not invoked from any `.github/workflows/*.yml` file
  (`grep -rn 'run-tests.sh' .github/workflows/` returns zero matches) — it exists as a local
  developer convenience script (`just test` / `just test-integration` call it, per
  `Justfile:271-282`), not a CI entry point. CI's own unit/integration jobs (`unit-tests`,
  `backend-integration`, `relay-e2e`) each construct their own explicit `cargo test`/`cargo nextest`
  invocations rather than delegating to this script.
- `just ci` (`Justfile:266`) expands to `check test-unit desktop-test desktop-build
  desktop-tauri-check desktop-tauri-test web-build mobile-test` — `test-unit` is `--lib`-scoped for
  every crate it touches (`buzz-core`, `buzz-auth`, `buzz-db --lib`, `buzz-conformance`,
  `buzz-push-gateway`; `Justfile:106-124`) and never mentions `buzz-test-client` at all.

**What CI *does* do to this file: compile + lint, silently.** `just clippy` — part of the
`rust-lint` CI job (`ci.yml:99-105`) and of `just check` (`Justfile:95`), hence of `just ci`
(`Justfile:266`) — runs `cargo clippy --workspace --all-targets -- -D warnings`
(`Justfile:106-107`). `--all-targets` includes test binaries, so `conformance_multitenant.rs` is
type-checked and lint-checked (modulo its own `#![allow(clippy::todo, unused)]` suppression, `:40`)
on every PR. `cargo fmt --all -- --check` (`Justfile:102-103`) likewise formats-checks it. So the
file is **not entirely invisible to CI** — a syntax error, a type error, or a non-suppressed lint
violation in it would fail a PR. But **zero of its assertions have ever executed** in any
automated gate this agent could find: `cargo test`/`cargo nextest` is never invoked against this
binary with `--ignored` (or without — the one non-ignored test is a no-op) anywhere in
`.github/workflows/`, `Justfile`, or `scripts/run-tests.sh`.

**Consequence for the "does either file's tests get wired into CI" question, stated precisely**:
`e2e_nostr_interop.rs` — yes, fully, unconditionally (modulo the `rust`-path-change gate common to
all Rust CI jobs). `conformance_multitenant.rs` — compiled and linted only; its 18 live
`#[ignore]`d assertions and 1 empty `#[test]` have no confirmed execution history in this
repository's automation.

---

#### D-TCC-02 — Even if CI were extended to run it, `conformance_multitenant.rs`'s required fixture is unprovisioned

Independent of D-TCC-01: even a hypothetical future CI job that added
`cargo test -p buzz-test-client --test conformance_multitenant -- --ignored` would fail
immediately, because (per the Integrations doc's exhaustive search) nothing in this repository
provisions a relay bound to two communities (`a.localhost:3000` / `b.localhost:3000`). The
`desktop-e2e-relay`/`backend-integration`/`relay-e2e` CI jobs that *do* start a relay all seed
exactly one community row at `localhost:3000` (`ci.yml:487-493`, `:611-619`;
`scripts/start-relay-for-tests.sh:104-127`). This is a second, independent blocker beyond simply
"nobody added the test command" — the supporting infrastructure genuinely does not exist yet.

---

#### D-TCC-03 — What this means for the multi-tenant-isolation confidence claim elsewhere in the repo

Per the task brief's explicit question: **the overall multi-tenant-isolation confidence claim in
`docs/multi-tenant-conformance.md` is, as far as any of this batch's evidence shows, backed by
zero executed end-to-end tests.** That document frames itself as a pre-implementation checklist
("Before multi-tenant mode is admitted, the implementation must have automated gates for these
classes of mistakes," final section, "Migration gates") — consistent with multi-tenancy being a
forward-looking rewrite rather than a shipped capability (the codebase's only confirmed-seeded
community, in every CI job and in `scripts/start-relay-for-tests.sh`, is the single default
`localhost:3000` row). So it is not that a shipped multi-tenant feature is unguarded — it's that
the *conformance suite meant to gate its eventual admission* is itself unexecuted, which means
when multi-tenancy work does land, there is currently no CI evidence that the gate this specific
file represents would actually catch a regression before this agent's review, since it has
apparently never been run against the fixture it requires. Combined with the earlier
`buzz-conformance`-batch finding that the *other* candidate gate (the `Tracer`/`check_trace`
runtime mechanism) is also unexercised in production (bound to `NoopTracer`), this repository
currently has, to the best of this agent's ability to verify, **no executed automated coverage of
its own stated multi-tenant isolation requirements**, despite two substantial, well-designed test
artifacts (`buzz-conformance`'s replay checker and this file) existing specifically to provide it.

---

#### D-TCC-04 — File size: `conformance_multitenant.rs` is the largest test file in the repository

Confirmed by direct measurement: `find crates -name '*.rs' -path '*/tests/*' | xargs wc -l | sort
-rn` ranks `conformance_multitenant.rs` at 2,739 lines first, ahead of the sibling group's
`e2e_relay.rs` at 2,477 lines (2nd) and this agent's own `e2e_nostr_interop.rs` at 1,987 lines
(3rd). Per the Conventions doc, roughly 31% of `conformance_multitenant.rs`'s bulk is doc-comment
prose rather than test logic — the file's size is driven as much by its executable-specification
ambitions (naming a mutate-bite and cross-referencing sibling tests for nearly every assertion) as
by the number of distinct behaviors it tests. This is a defensible design choice for a
specification-adjacent artifact, but it also means the file is difficult to review or modify
incrementally — a single-PR change touching one obligation row requires navigating a 2,739-line
file with no sub-file structure (all 19 `mod` blocks live in one physical file; none use `#[path]`
to split into separate files).

---

#### D-TCC-05 — 11 of 19 obligation rows are unimplemented; the stub pattern hides this from a shallow read

`conformance_multitenant.rs` declares 19 `mod` blocks mirroring `docs/multi-tenant-conformance.md`'s
conformance table. Of these:

- 8 modules contain a `#[tokio::test] #[ignore]` function whose entire body is a single
  `pending_lane(...)` call, which always panics via `todo!(...)` if actually run:
  `membership_allowlist::archive_in_a_does_not_affect_b` (`:906-911`),
  `channelless_global_events_dms::same_event_id_and_dtag_coexist_across_communities` (`:1291-1298`),
  `channelless_global_events_dms::dm_does_not_cross_deliver_between_communities` (`:1301-1307`),
  `feed_read_side_isolation::feed_mentions_do_not_cross_communities_over_the_wire` (`:1326-1336`),
  `workflows::approval_token_is_community_confined` (`:1941-1948`),
  `media_blossom::media_metadata_boundary_holds_while_blob_bytes_shared` (`:2622-2627`),
  `git_hosting::same_owner_repo_isolated_push_does_not_cross` (`:2640-2645`),
  `mesh_agents_cli::one_key_two_communities_no_bleed` (`:2658-2663`).
- 2 modules (`audit_log`, `n1_parity`) contain **no test function whatsoever** — only doc comments
  (`:2669-2723`, `:2728-2739`).
- 1 module (`api_tokens_nip98_replay::token_minted_in_a_does_not_authorize_in_b`, `:636-679`)
  contains a compiling, non-`#[ignore]`d `#[test]` with an **empty body** — it will show as
  "passed" in any test run (since it runs unconditionally, unlike the `#[ignore]`d items) while
  asserting literally nothing.

Given the CI status established in D-TCC-01, a developer reading a green CI run today gets no
signal about any of this — the file isn't run at all. But even a developer who *does* run `cargo
test -p buzz-test-client --test conformance_multitenant -- --ignored` locally against a live
single-host relay would see 8 immediate panics (from the stubs), 8 passes or failures depending on
fixture availability (the live tests, several of which — e.g. `client_supplied_community_cannot_
override_host` — would likely fail outright against a single-community relay, since their
premise is that `url_a()`/`url_b()` resolve to *different* communities), and 1 silent, contentless
pass. Interpreting a partial run of this file requires reading the source; the pass/fail signal
alone is not informative given the stub density. The empty-body `#[test]` in particular is a
distinct hazard: unlike a `pending_lane` stub (visibly marked `#[ignore]`, panics loudly if forced),
it is a normal, always-green test that could be mistaken for a real assertion.

---

#### D-TCC-06 — Heavy duplication of connection-normalization and channel-creation helpers within `conformance_multitenant.rs`

`to_ws`/`to_http` URL-scheme-normalization helpers are reimplemented nearly verbatim in 4 separate
modules (`users_profiles_nip05:924-948`, `channels_membership:1348-1372`, `search_fts:1964-1988`,
`pubsub_presence_typing:2300-2324`) rather than hoisted to file scope. `create_channel`/
`create_open_channel`/`post_kind9` recur with minor variance across 5 modules
(`row_zero_host_binding`, `channels_membership`, `workflows`, `search_fts`,
`pubsub_presence_typing`). This is pure duplication debt — none of the variance between copies
appears semantically necessary (all normalize the same four URL-scheme prefixes; all build the
same kind:9007 tag shape) — and it means a future change to, say, the channel-creation tag shape
(e.g. adding a required new tag) would need to be applied at 5 separate call sites within this one
file, in addition to the already-duplicated equivalent in `e2e_nostr_interop.rs`
(`create_test_channel`, `:44-72`) and presumably in sibling test files this agent does not own.

---

#### D-TCC-07 — Stale/uncross-referenced documentation: `RELAY_URL_A`/`RELAY_URL_B` absent from every doc a contributor would actually consult

Per the Configuration doc: `TESTING.md`'s "Automated Tests" section documents exactly one
`#[ignore]`d-suite invocation pattern and one env var (`RELAY_URL`); it never mentions
`conformance_multitenant.rs`, `RELAY_URL_A`, `RELAY_URL_B`, or the two-host fixture requirement.
A contributor following `TESTING.md`'s documented pattern (`cargo test -p buzz-test-client --
--ignored`) would run `conformance_multitenant.rs`'s tests incidentally (they're in the same
crate) and hit 8 immediate `todo!()` panics plus a batch of `url_a()`/`url_b()`-driven failures
against whatever single-community relay `TESTING.md`'s own setup instructions produce — with
**no warning anywhere in that document** that this specific file has a fundamentally different
infrastructure precondition than every other suite it documents.

---

#### D-TCC-08 — `e2e_nostr_interop.rs`'s own doc comment is accurate; no drift found in this file

For contrast: this agent found no doc/code drift inside `e2e_nostr_interop.rs` itself. Its
top-of-file comment's claims (`#[ignore]` by default, `RELAY_URL` override, the exact
`cargo test --test e2e_nostr_interop -- --ignored` invocation) all match the source and match the
CI job that actually runs it (`ci.yml:729`, modulo CI combining it with `--test e2e_persona` in one
invocation, which is a superset of what the doc comment describes, not a contradiction). This file
does not carry this batch's doc-drift debt; `conformance_multitenant.rs` does, per D-TCC-02/D-TCC-07.

---

#### D-TCC-09 — Untested critical path within what this file claims to cover: approval-token isolation is blocked on an external gap

Beyond the general stub count (D-TCC-05), one stub is worth calling out specifically because it
names its own blocker rather than simply being unimplemented: `workflows::approval_token_is_
community_confined` (`:1941-1948`) is blocked on WF-08 — the workflow executor's approval gate is
an explicit TODO elsewhere in the codebase, per this file's own doc comment at `:1933-1936`
("the executor's approval gate is an explicit TODO ... see WF-08"), independently cross-referenced
by the `buzz-workflow` batch's debt doc at `conformance_multitenant.rs:1863-1946`. So this
particular gap is not just missing test coverage — it is missing test coverage for a feature
(approval-token minting over the wire) that does not yet exist to be tested, correctly identified
as such by the test file's own author. This is a better-than-average stub in the file's set of 8:
it documents precisely why it cannot be filled in yet, rather than leaving the reader to guess.
The remaining 7 stubs (`membership_allowlist`, `channelless_global_events_dms` ×2,
`feed_read_side_isolation`, `media_blossom`, `git_hosting`, `mesh_agents_cli`) each name a lane
owner (e.g. `"buzz-auth"`, `"buzz-media"`, `"relay-wiring"`) but, unlike `approval_token_is_
community_confined`, do not name an external blocker — nothing in the repository indicates why
these specific rows remain unimplemented while others in the same file are live.
