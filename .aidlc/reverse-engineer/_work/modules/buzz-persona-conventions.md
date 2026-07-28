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
