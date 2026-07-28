## Module: buzz-test-client — Nostr interop & multi-tenant conformance E2E (`crates/buzz-test-client/tests`)
### Aspect: Conventions

---

#### 1. File-size discipline

| File | Lines | Rank among `crates/**/tests/*.rs` |
|---|---|---|
| `conformance_multitenant.rs` | 2,739 | **1st** (largest test file in the repo, confirmed by `find crates -name '*.rs' -path '*/tests/*' \| xargs wc -l \| sort -rn`) |
| `e2e_relay.rs` (sibling group's file) | 2,477 | 2nd |
| `e2e_nostr_interop.rs` | 1,987 | 3rd |

`conformance_multitenant.rs` is 262 lines (11%) longer than `e2e_relay.rs`. Neither file
this agent owns respects any evident soft file-size ceiling — `AGENTS.md` states a hard 1,000-line
ceiling for **mobile** widget files only (enforced by `mobile/scripts/check-file-sizes.mjs`); no
equivalent guard exists for Rust test files anywhere this agent could find (`grep` for
`check-file-sizes` / `check_file_sizes` outside `mobile/` returns nothing). Both files are
substantially larger than the largest non-test `.rs` files in most other crates, and
`conformance_multitenant.rs`'s size is significantly inflated by prose: see §2.

---

#### 2. Doc-comment density — extreme, and specifically doc-comment-heavy rather than code-heavy

| File | Doc-comment lines (`///` + `//!`) | Fraction of file |
|---|---|---|
| `e2e_nostr_interop.rs` | 128 (`///`) | ~6.4% of 1,987 lines |
| `conformance_multitenant.rs` | 750 (`///`) + 101 (`//!`) = 851 | **~31%** of 2,739 lines |

Roughly a third of `conformance_multitenant.rs` is prose. This is a deliberate, unusual
convention for this file specifically: each test function is preceded by a multi-paragraph
doc comment that states the obligation, the precise wire-observable shape, an explicit
"mutate-bite" (what code change would turn the test red), and — frequently — a named
cross-reference to a sibling test elsewhere in the same file that shares scope but asserts a
different property (e.g. `client_supplied_community_cannot_override_host`'s doc comment at
`:157-186` explicitly distinguishes itself from `channels_membership::
same_channel_uuid_in_two_communities_is_isolated` and two auth-lane tests by name). Attributed
authorship appears throughout as a first-name comment convention on `mod` block headers (e.g.
`// Row zero: request community binding (Eva — relay-wiring)` at `:73`,
`// API tokens and NIP-98 replay (Sami — buzz-auth)` at `:556`) — a lane-ownership convention not
seen in `e2e_nostr_interop.rs`, which carries no attributed-author comments at all.

This density makes the file read more like an executable specification annex than a typical
integration test — consistent with its module doc-comment's framing as "the executable form of
the conformance contract" (`:2-3`) mirroring `docs/multi-tenant-conformance.md`.

---

#### 3. Error handling patterns

| Pattern | `e2e_nostr_interop.rs` count | `conformance_multitenant.rs` count |
|---|---|---|
| `.unwrap()` | 46 | 42 |
| `.expect("...")` | 160 | 72 |
| `panic!(...)` | 10 | 21 |
| `unwrap_or_else` | 5 | (not separately counted; folded into `.expect`-adjacent chains) |

Both files follow the same idiom throughout: helper functions and test bodies use
`.expect("<context string>")` almost exclusively over bare `.unwrap()`, and every `.expect()`
message is a specific, actionable phrase (e.g. `"submit create-channel event"`,
`e2e_nostr_interop.rs:64`; `"parse create-channel response"`, `conformance_multitenant.rs:1401`)
rather than a generic message — there is no use of `.unwrap()` on a `Result`/`Option` without a
preceding assertion establishing why failure is impossible, as far as this agent sampled. Neither
file uses `?`-based error propagation in test bodies (test functions return `()`, not
`Result<(), E>`), so every failure path is `panic!`/`.expect()`-based — standard for
`#[tokio::test]` integration suites in this style.

`conformance_multitenant.rs` additionally uses `unwrap_or_else(|e| panic!("...: {e}"))` as its
dominant idiom for network-call failures specifically (e.g. `:135-137`, `:200-202`), which
produces a panic message that embeds the underlying `reqwest::Error` — a slightly more
informative pattern than plain `.expect("...")` for network-boundary calls, applied consistently
across the file's ~15+ `reqwest::Client` call sites.

---

#### 4. `unsafe` / lint attributes

- Neither file contains `unsafe` code (`grep -c 'unsafe' ` on both returns 0 beyond doc-comment
  prose, none found). This is consistent with `AGENTS.md`'s repo-wide "No `unsafe` code" rule.
- `conformance_multitenant.rs` opens with a file-level `#![allow(clippy::todo, unused)]`
  (`:40`) — the only lint-suppression attribute in either file. This is necessitated by the file's
  `pending_lane` stub pattern (`todo!()`-based panics, which `clippy::todo` would otherwise flag)
  and by the fact that 8 of 19 modules import types (`Keys`, `EventBuilder`, etc. via `super::*`
  or module-local `use`) that go unused in modules containing only a `pending_lane` stub. This is
  a blanket, file-wide suppression rather than a scoped `#[allow(...)]` on the specific stub
  functions — a broader-than-necessary lint bypass, though a defensible one given the file's
  intentionally-incremental-fill-in design (per its own doc comment, `:31-34`: "A row is
  `todo!()`-stubbed until the lane it depends on lands... a green run can never be faked by an
  empty body").
- `e2e_nostr_interop.rs` carries no file-level `#![...]` attributes at all.
- Neither file's crate (`buzz-test-client`) declares `#![warn(missing_docs)]` at the test-binary
  level (that attribute exists only in `src/lib.rs:2`, which does not apply to `tests/*.rs`
  binaries) — so the extensive doc comments in `conformance_multitenant.rs` are a voluntary
  convention, not a compiler-enforced one, for these two files specifically.

---

#### 5. Test organization and naming

- **`e2e_nostr_interop.rs`** is flat: no `mod` blocks, no attributed ownership, tests declared
  directly at file scope in a single sequence, with private `async fn`/`fn` helpers interleaved
  between test groups (helpers at `:27-256`, then tests and more helpers alternating through
  `:1987`). Test names follow a consistent `test_<nip-or-feature>_<behavior>` convention:
  `test_nip50_search_returns_results_and_eose`, `test_nip10_root_mismatch_rejected`,
  `test_nipdv_ids_query_rejects_third_party` — every one of the 25 test functions matches this
  `test_` prefix pattern.
- **`conformance_multitenant.rs`** is hierarchical: 19 named `mod { ... }` blocks, each
  corresponding 1:1 to a row in `docs/multi-tenant-conformance.md`'s conformance table (confirmed
  by comparing the module list against that document's table row labels — e.g. `row_zero_host_binding`
  ↔ "Row zero: request community binding," `channels_membership` ↔ "Channels and channel
  membership"). Test names inside each module drop the `test_` prefix entirely and instead read as
  full behavioral assertions: `unmapped_host_fails_closed_generically`,
  `same_channel_uuid_in_two_communities_is_isolated`, `workflow_trigger_is_community_confined`.
  Helper functions (`create_channel`, `post_kind9`, `to_ws`, `to_http`) are **duplicated
  near-verbatim across modules** rather than factored into a shared file-level helper — e.g. `to_ws`/
  `to_http` are reimplemented identically in `users_profiles_nip05` (`:924-948`),
  `channels_membership` (`:1348-1372`), `search_fts` (`:1964-1988`), and `pubsub_presence_typing`
  (`:2300-2324`); `create_channel`/`create_open_channel`/`post_kind9` similarly recur with minor
  signature variance across `row_zero_host_binding`, `channels_membership`, `workflows`,
  `search_fts`, and `pubsub_presence_typing`. This is the file's single most visible
  code-duplication convention — see Debt doc.

---

#### 6. Comment-driven traceability convention (`conformance_multitenant.rs`-specific)

Distinct from ordinary doc comments, this file has a convention of naming an exact **mutate-bite**
for nearly every live assertion — a sentence of the form "Mutate-bite (would-it-fail-without-the-
fix): [specific code change] → [specific assertion that would go red]," each citing an exact
file:line in relay source (e.g. `:203-206` for `client_supplied_community_cannot_override_host`,
citing `ingest::check_channel_membership`; `:806-810` for the NIP-98 replay test, citing
`bridge.rs:79`). This convention is a form of executable-spec traceability not seen in
`e2e_nostr_interop.rs`, whose doc comments describe *what* is asserted but rarely name the exact
production code path whose regression the assertion is meant to catch.
