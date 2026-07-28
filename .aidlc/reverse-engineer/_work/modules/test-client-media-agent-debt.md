## Module: buzz-test-client — media, managed-agent, mesh-LLM & persona E2E (`crates/buzz-test-client/tests`)
### Aspect: Debt

#### CI wiring: only one of six files ever runs anywhere

This is the single most consequential finding for this group. Checked exhaustively against `.github/workflows/*.yml`, `Justfile`, and `scripts/run-tests.sh`:

| File | Referenced by name in CI/Justfile/scripts? | Where |
|---|---|---|
| `e2e_media.rs` | **No** | — |
| `e2e_media_extended.rs` | **No** | — |
| `e2e_media_video.rs` | **No** | — |
| `e2e_managed_agent.rs` | **No** | — |
| `e2e_mesh_llm.rs` | **No** | — |
| `e2e_persona.rs` | **Yes** | `.github/workflows/ci.yml:729`, the `relay-e2e` job: `cargo test -p buzz-test-client --test e2e_persona --test e2e_nostr_interop -- --ignored --nocapture` |

Grep evidence for the negative claims: `grep -n 'e2e_media\|e2e_managed_agent\|e2e_mesh_llm' .github/workflows/*.yml Justfile scripts/run-tests.sh` returns zero matches in every one of those files (verified individually per pattern — a combined-alternation grep under-matched on a first pass and was re-run per-term to confirm). `scripts/run-tests.sh`'s `run_integration_tests` function runs `cargo test --test '*'` at the workspace level (`scripts/run-tests.sh:126-128`), which *would* pick up all `tests/*.rs` files including these six — but every test in this group is `#[ignore]`d, and that invocation does not pass `--ignored`, so even that catch-all wildcard executes zero tests from this group. `just test-unit` never touches `buzz-test-client` at all (`Justfile:172-192` lists `buzz-core`, `buzz-auth`, `buzz-db`, `buzz-conformance`, `buzz-push-gateway` — no `buzz-test-client`).

The one partial mitigation: `cargo clippy --workspace --all-targets` (`Justfile:107`, run in CI's `rust-lint` job) and `cargo check --workspace --all-targets` (`Justfile:659`) both compile-check and lint every file in this group as a side effect of `--all-targets`, so a syntax error, an unused import, or a clippy-flagged pattern in any of these six files *would* fail CI even though the test bodies never execute. That is compile-time coverage only — it catches nothing about runtime behavior, relay responses, or assertion correctness.

**`e2e_mesh_llm.rs` is the most severe instance of this pattern**, and it is worth calling out on its own: this file depends on the newest, least-proven subsystem in scope for this analysis pass (`buzz-agent`'s mesh "Auto" collective-routing state machine, per the prior `agent-llm` analysis) — yet it has **zero** CI execution, and even its `#[ignore]`-gated content is a further two-thirds no-ops: of its four tests, two (`trust_member_reads_mesh_status`, `trust_nonmember_read_denied`) silently skip without asserting anything unless a human manually exports `MEMBER_NSEC`/`STRANGER_NSEC` (env vars that, per the Configuration document, are undocumented anywhere outside this one file), one (`live_agent_completes_chat_over_mesh`) skips unless `MESH_OPENAI_BASE` is set and two live desktop mesh nodes are running, and the fourth (`live_split_model_completes`) is an unconditional `println!` + `return` with **no assertions in its body at all** (`e2e_mesh_llm.rs:282-291`). So this file's realistic coverage, even under a deliberate manual `cargo test --test e2e_mesh_llm -- --ignored` run with no env vars set, is **zero assertions executed** — every one of the four test bodies returns before reaching an `assert!`. This mirrors the pattern the task description says was "already confirmed for other test-client groups" — here it is present in its most extreme form: a nominally-existing E2E test file that, under the most common invocation, asserts nothing at all.

#### Duplication across the three media files

The three media files independently reimplement near-identical upload/auth infrastructure rather than sharing a helper module (there is no `tests/common/` in this crate — confirmed by directory listing):

| Helper | `e2e_media.rs` | `e2e_media_extended.rs` | `e2e_media_video.rs` | Identical? |
|---|---|---|---|---|
| `relay_http_url()` | `:25-27` | `:12-14` | `:18-20` | byte-identical body |
| `http_client()` | `:28-33`, 15s timeout | `:22-27`, 15s timeout | `:24-29`, **30s** timeout | 2 of 3 identical; video file's is a silent divergence with no comment explaining why video needs a longer default |
| `sign_blossom_auth()` | `:36-48` | `:35-44` | `:38-49` | functionally identical, trivially different formatting (`e2e_media.rs` uses `.expect(...)` messages on every `Tag::parse`; the other two use bare `.unwrap()`) |
| `blossom_auth_header()` | `:51-57` | `:47` (one-liner via a different `format!` layout) | `:52-57` | same logic, different line-wrapping |
| `tiny_jpeg()` | `:60-84` | `:63-87` | `:294-318` | byte-for-byte identical 339-byte array, three separate copies |

A single upload-auth-plus-fixture helper module (shared via `#[path]` include or a `tests/common/mod.rs`) would collapse roughly 100-150 lines of exact duplication across these three files. The `http_client()` timeout divergence (15s/15s/30s) is the one place this duplication has already produced a silent behavioral inconsistency rather than just extra lines — nothing documents why video uploads get double the client timeout, and if the intent was "video fixtures are bigger, allow more time," that reasoning is not written anywhere, and the fixture in `e2e_media_video.rs` is in fact tiny (a hand-built few-hundred-byte synthetic MP4, not a real video file).

The task's specific question — "do the three media files show signs they should have been one file, or shared a helper module, rather than three separate large files?" — the evidence says **shared helper module**, not **one file**: the three files test genuinely distinct concerns (base upload/download, auth-and-content-policy edge cases, video-specific range/poster semantics) that justify separate top-level organization, but the ~100+ lines of copy-pasted plumbing each one carries to get there is the actual debt, not the file split itself.

#### Function-name / assertion mismatch

`test_upload_hash_mismatch_returns_400` (`e2e_media.rs:274-296`) is misnamed: both its own doc comment ("must return 401", `:273`) and its actual assertion (`assert_eq!(resp.status(), 401, "hash mismatch must be 401")`, `:295`) check for **401**, not 400. The function name is the only place "400" appears in the entire test. This is a harmless-but-confusing leftover from an earlier version of the behavior or an initial assumption that never got corrected when the expected status code changed.

#### Hardcoded literal disconnected from its own fixture

`e2e_media_extended.rs`'s two WS-imeta tests hardcode `"size 347"` in the `imeta` tag (`:575`, `:637`) regardless of the actual byte length of the referenced upload. Both call sites reference the same `tiny_jpeg()` fixture (339 bytes per the Data Model doc, not 347), so the literal is already wrong for the fixture it is paired with, and would silently keep being "accepted" by a relay that (per the file's own passing tests) does not cross-validate the `imeta` `size` field against the actual stored blob size. No test in this file asserts that the relay validates `imeta size` correctness at all — the two tests that construct this tag (`test_ws_valid_imeta`, `test_ws_invalid_imeta_external_url`) never vary this field to prove the relay would reject a *mismatched* size, only a *missing* one (`test_ws_invalid_imeta_missing_fields`, `:659-712`, which omits `m`/`x`/`size` entirely rather than supplying a wrong `size`). This is both a latent-bug-in-the-fixture and a coverage gap in the same spot.

#### Doc-comment discipline gap

`e2e_media_extended.rs` — the group's second-largest file (713 lines) and, by the Business Rules analysis, the file establishing the densest and most nuanced rule set (the two-tier content-type policy, the six-way Blossom-auth-grammar rejection matrix) — has **zero** `///` doc comments on any of its 21 test functions, in contrast to every sibling file in this group (see Conventions for the full count table). A reader has to infer the *why* behind e.g. the `/upload` vs `/media/upload` content-type asymmetry from two isolated inline comments (`:385-386`, `:404-405`) rather than from a doc comment on the tests that actually pin that asymmetry (`test_legacy_media_route_rejects_non_media`, `:420-433`; `test_standard_upload_rejects_recognized_audio`, `:443-455` — neither has any comment at all explaining the policy rationale).

#### Untested critical paths within what these files claim to cover

| Gap | Why it matters | Evidence |
|---|---|---|
| No upload-size-limit rejection anywhere in `e2e_media*.rs` | `.env.example` documents `BUZZ_MEDIA_MAX_CONCURRENT_UPLOADS*`/`BUZZ_MEDIA_UPLOADS_PER_MINUTE` as real, configurable admission controls; none is exercised | `grep -rn 'too large\|MAX_.*BYTES\|413' crates/buzz-test-client/tests/e2e_media.rs crates/buzz-test-client/tests/e2e_media_extended.rs crates/buzz-test-client/tests/e2e_media_video.rs` → zero matches |
| No malformed/truncated video upload test | Only one well-formed synthetic MP4 fixture exists in `e2e_media_video.rs`; there is no negative test proving the relay's MP4 parser rejects garbage input gracefully rather than panicking or hanging | confirmed by full read of `e2e_media_video.rs` — every upload test uses `build_test_mp4()` unmodified |
| `imeta` `size` field is never cross-checked against real blob size (see above) | A relay bug that silently ignores this field would go undetected forever by this suite | `e2e_media_extended.rs:575,637` |
| `e2e_mesh_llm.rs`'s `live_split_model_completes` has an empty body | Assertion #6 of the file's own acceptance matrix is entirely unverified by any automated means in this repository — the function exists only to make the matrix "executable code, not prose" per its own doc comment (`:219-220`), but it proves nothing | `e2e_mesh_llm.rs:282-291` |
| `e2e_managed_agent.rs` never tests a **rejection** path for a malformed managed-agent event | Every test in the file publishes a well-formed event (or the deliberately-still-valid legacy-fat shape); there is no equivalent to `e2e_persona.rs`'s extensive d-tag-grammar rejection suite — e.g. no test for a managed-agent `d`-tag that is not 64-hex-chars, malformed JSON content, or a duplicate/conflicting tag | confirmed by full read; the file's 5 tests are publish-and-query, legacy-compat, round-trip-fidelity, NIP-33 replace, and tombstone-delete — no negative/validation-rejection test exists |
| No test asserts the relay rejects a managed-agent event content that **isn't valid JSON** | `agent_projection_content`/`legacy_fat_agent_projection_content` always build valid JSON via `serde_json::json!`; nothing sends a plain string or malformed JSON body | confirmed by full read of `e2e_managed_agent.rs` |

#### Doc drift between this group and the top-level repo docs

`ARCHITECTURE.md`'s `buzz-test-client` table (`:700-706`) lists only `e2e_relay.rs`, `e2e_media.rs`, `e2e_media_extended.rs`, and `e2e_nostr_interop.rs` — **four of six files in this analysis group's scope are entirely absent from that table**: `e2e_media_video.rs`, `e2e_managed_agent.rs`, `e2e_mesh_llm.rs`, and `e2e_persona.rs` are not mentioned anywhere in `ARCHITECTURE.md`, despite `e2e_persona.rs` (1,591 lines, 24 tests) being both the largest test file in the entire crate and the *only* one of these six actually wired into CI. Within this group's own files, the table's per-file counts are also stale where checkable: it lists `e2e_media.rs` at 7 tests (matches the current count exactly, `grep -c '#\[tokio::test\]' crates/buzz-test-client/tests/e2e_media.rs` → 7) but `e2e_media_extended.rs` at 18 (current count is **21**, confirmed by the same grep pattern) — so even the one in-scope file the table *does* track has drifted. The table's stated crate-wide total ("Total: **134 e2e tests**", `ARCHITECTURE.md:707`) is stale for the same underlying reason: it predates the addition of the four files this group covers that it never enumerates. `CONTRIBUTING.md`'s parallel list (`:166-170`) is smaller still and references a file, `e2e_mcp.rs`, that does not exist anywhere in the repository (`file_search` for `e2e_mcp.rs` returns zero results) — that particular drift point is outside this group's file scope, but it corroborates that the top-level e2e-test-file inventory in the prose docs has not been kept in sync with `crates/buzz-test-client/tests/`'s actual contents, and this group's four undocumented files are the largest concrete instance of that drift.

#### Doc drift within this group's own files (module-doc vs. actual behavior)

`TESTING.md` states flatly: "Neither task runs the E2E suites in `buzz-test-client` — those are marked `#[ignore]` and require a running relay" (`TESTING.md`, under "Automated Tests"). That statement is accurate for `just test`/`just test-unit`, but is silent about the fact that `e2e_persona.rs` specifically **is** run automatically, in CI, via a third path (`ci.yml`'s `relay-e2e` job) that `TESTING.md` never mentions. A contributor reading only `TESTING.md` would reasonably conclude no e2e test in this crate runs anywhere automatically, which is false for one of the six files in this scope.

#### Stale/inaccurate assertion-message text (minor)

Beyond the 400-vs-401 mismatch above, `test_persona_ingest_shared_tag_validation`'s comment for the `["shared","x"]` case (`e2e_persona.rs:1099` area) and its sibling cases are internally consistent, but the overall six-case validation test (`e2e_persona.rs:1037-1150`) mixes assertion-message specificity — some branches assert on message *substring* (`"shared" || "invalid"`, `:1057-1060`), others assert acceptance/rejection only with no message check at all (`:1116-1121`, `:1128-1133`) — inconsistent rigor within a single test function, though not a correctness bug.

#### Oversized files, restated with this group's specific numbers

| File | Lines | Note |
|---|---|---|
| `e2e_persona.rs` | 1,591 | Largest in this group and in the crate; no Rust file-size CI gate exists (confirmed: the `check-file-sizes` guard the top-level `AGENTS.md` describes is JS/TS/Dart-only, per `Justfile:585` (web), `:617`/mobile section) |
| `e2e_media_extended.rs` | 713 | Second-largest, and the one with zero doc comments (see Conventions) |
| `e2e_media_video.rs` | 645 | Contains a 170-line hand-rolled MP4 box-builder (`build_test_mp4`, `:53-222`) that is itself close to a small MP4-muxer library and could reasonably be its own tested unit rather than inline test fixture code |

None of the three are anywhere near this repo's actual largest files (`crates/buzz-acp/src/lib.rs` at 6,570 lines, per the prior `agent-llm-debt.md` cross-reference), so this is a moderate, not severe, concern — but `e2e_persona.rs`'s size combined with its zero internal `mod` structure (see Conventions) means any future contributor adding a 25th persona test has no natural seam to split the file along.
