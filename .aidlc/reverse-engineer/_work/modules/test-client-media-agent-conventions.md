## Module: buzz-test-client — media, managed-agent, mesh-LLM & persona E2E (`crates/buzz-test-client/tests`)
### Aspect: Conventions

#### File-size discipline

| File | Lines | vs. `AGENTS.md`'s no-Rust-file-size-gate reality |
|---|---|---|
| `e2e_persona.rs` | 1,591 | Largest file in this group by 2.2×; no Rust file-size ceiling exists in this repo (only desktop/web/mobile have `check-file-sizes` gates per `AGENTS.md`'s Mobile section and the earlier `agent-llm-debt.md` finding that `justfile:123/585/617` are JS/TS/Dart-only) |
| `e2e_media_extended.rs` | 713 | |
| `e2e_media_video.rs` | 645 | |
| `e2e_media.rs` | 376 | |
| `e2e_managed_agent.rs` | 368 | |
| `e2e_mesh_llm.rs` | 291 | |

There is no mechanical guard preventing `e2e_persona.rs` from growing further — its 24 tests are one flat `mod`-less file with three comment-delimited sections (`// ─── NIP-98 HTTP bridge persona gate tests ───`, `e2e_persona.rs:719`, and a similar rule at `:625`), not actual Rust submodules. Splitting along those same section boundaries (core CRUD, shared-gate, HTTP-bridge-gate, limit-regression) would be mechanical since each section already only depends on module-level `fn`s (`persona_event`, `persona_event_with_shared*`, `submit_event_http`, etc.) rather than section-local state.

#### Test organization and naming

All 68 `#[tokio::test]` functions in this group follow `test_<subject>_<expectation>` (media/persona/managed-agent files) except `e2e_mesh_llm.rs`, whose four tests use bare descriptive names with no `test_` prefix (`trust_member_reads_mesh_status`, `trust_nonmember_read_denied`, `live_agent_completes_chat_over_mesh`, `live_split_model_completes` — `e2e_mesh_llm.rs:93,180,223,284`). This is a deliberate, self-consistent convention within that one file (grouping by `trust_*` vs `live_*` prefix, matched by the module doc's own filter examples: "`cargo test --test e2e_mesh_llm trust`", `e2e_mesh_llm.rs:17-18`), not an oversight — but it is the only file in the group that departs from `test_*`.

Every file is single-`mod` — no `mod tests { ... }` nesting, no helper submodule. Shared per-file setup (`http_client()`, `relay_url()`/`relay_http_url()`) is a bare top-of-file `fn`, not a `OnceCell`/`lazy_static`/fixture struct.

#### Doc-comment discipline: one clear outlier

Five of six files carry a `///` doc comment on every (or nearly every) test function explaining what business rule it pins, often with a "why" and a citation to a spec or regression commit:

| File | `///` lines | Representative pattern |
|---|---|---|
| `e2e_persona.rs` | 74 | Nearly every test has a multi-line doc citing the specific AC (`/// AC-1: ...`, `:653-657`) or a regression commit (`/// Verifies at \`312014d5e\`...`, `e2e_persona.rs:1401-1409`) |
| `e2e_mesh_llm.rs` | 29 | Doc comments include a full acceptance matrix table in the module doc (`:29-38`) cross-referencing which assertion lives in which file |
| `e2e_managed_agent.rs` | 30 | Per-test docs explicitly state what the test does *not* prove (e.g. `:186-193` disclaiming the secret-exclusion guard) |
| `e2e_media_video.rs` | 18 | Doc comments explain *why* a behavior is correct, not just what is asserted (e.g. `:222-229` explaining the Content-Type-ignored rule) |
| `e2e_media.rs` | 12 | Module doc gives exact run commands (`:6-16`) |
| **`e2e_media_extended.rs`** | **0** | Not one of its 21 test functions has a `///` doc comment — confirmed by `grep -c '^///' crates/buzz-test-client/tests/e2e_media_extended.rs` → 0, and `grep '^//[^!/]'` (plain non-doc, non-module comments) also → 0. The only prose in the file is the 4-line module doc (`:1-4`) and a handful of inline `//` comments inside three function bodies (`:385-386`, `:404-405`, `:460-461`, `:479-480`) |

`e2e_media_extended.rs` is also the second-largest file in the group (713 lines) and the one with the densest, most varied rule set (auth grammar, format matrix, content-policy asymmetry — see Business Rules) — meaning the file that most needs per-test explanation is the one file with none. Every other file's convention of "doc comment states the rule, test proves it" is absent here; test names carry the entire explanatory burden instead (e.g. `test_legacy_media_route_rejects_non_media` is self-explanatory, but the *why* — the two-tier content-type policy rationale — is undocumented at the test level and only appears in inline comments on two of the twenty-one tests).

#### `println!` / success-marker convention

Three files use a `println!("✅ ...")` convention to mark successful assertions for `--nocapture` runs: `e2e_media_extended.rs` (20 occurrences), `e2e_media.rs` (1, only in `test_upload_real_image`), `e2e_media_video.rs` (1, only in `test_video_content_type_header_ignored`). `e2e_managed_agent.rs` and `e2e_persona.rs` use zero `println!`/`✅` markers — they rely entirely on `assert!`/`assert_eq!` failure messages. `e2e_mesh_llm.rs` uses plain `eprintln!` (2 occurrences, both `SKIP: ...` diagnostics for the env-var-gated skip paths, `:64-68`, `:227-229`) rather than `println!`, and one plain `println!` for the unconditional `live_split_model_completes` skip message (`:288`). No file uses the `tracing` crate's macros (`grep -c 'tracing::'` → 0 in all six files), even though `tracing`/`tracing-subscriber` are declared dependencies of this crate (`Cargo.toml:19`, dev-dependency `:24`) and are used by `lib.rs` itself (imported `lib.rs:11`, `debug!` called at `lib.rs:99`). This group's tests never initialize a `tracing_subscriber`, so any `debug!`/`tracing::` output from the harness they call is silently dropped unless the caller sets up their own subscriber externally.

#### Error handling

Every file is `.unwrap()`/`.expect()`-heavy, which is idiomatic for `#[ignore]`d integration tests reporting failure via panic rather than `Result`: `e2e_persona.rs` alone has 176 occurrences, `e2e_media_extended.rs` 94, `e2e_media_video.rs` 59, `e2e_managed_agent.rs` 33, `e2e_media.rs` 31, `e2e_mesh_llm.rs` 14. `.expect(...)` messages are consistently used over bare `.unwrap()` at the top level of each test (naming what failed), but nested/chained calls (e.g. inside a `.json::<Value>()` parse) more often fall back to bare `.unwrap()`. None of this violates `AGENTS.md`'s "do not introduce new `unwrap()`/`expect()` in production paths" rule — these are `tests/*.rs` integration-test binaries, not library/production code, so the rule does not apply, and no file blurs that line (no shared production-facing helper is defined in any of the six files).

#### Lint attributes and `unsafe`

`grep -rn '#\[allow\|unsafe'` across all six files returns zero matches — no lint suppression, no `unsafe` blocks anywhere in this group. `lib.rs` itself carries `#![deny(unsafe_code)]` and `#![warn(missing_docs)]` (`lib.rs:1-2`), but those crate-level attributes apply to the library target (`src/lib.rs`), not to `tests/*.rs` integration-test binaries, which are separate compilation units and are not covered by them.

#### Fixture-data conventions

The 339-byte `tiny_jpeg()` byte literal is reproduced identically in three files (`e2e_media.rs:60-84`, `e2e_media_extended.rs:63-87`, `e2e_media_video.rs:294-318`) — same 27-line hex array, same comment ("A valid 1×1 red JPEG"/"Minimal valid JPEG (1x1 pixel)"). `e2e_media_extended.rs` additionally defines `tiny_png()`, `tiny_gif()`, `tiny_webp()` (`:89-137`) with provenance comments ("generated by ffmpeg") that the other two files' single-format fixtures lack. `e2e_media_video.rs`'s `build_test_mp4()` (`:53-222`) is the one hand-rolled binary-format builder in the group — a 170-line MP4-box constructor with per-box comments, contrasted with the other files' flat byte-literal approach. See Debt for the cost of the fixture duplication.
