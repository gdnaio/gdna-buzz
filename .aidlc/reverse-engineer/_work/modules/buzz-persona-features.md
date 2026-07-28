## Module: buzz-persona (`crates/buzz-persona`)

### Aspect: Features

### Capability inventory

| # | Capability | Entry point | Completeness | Evidence |
|---|---|---|---|---|
| F1 | Parse a `.persona.md` (YAML frontmatter + markdown body) from a string | `persona::parse_persona_md` (`crates/buzz-persona/src/persona.rs:208`) | Full | 29 unit tests in `crates/buzz-persona/src/persona.rs:331-644` covering minimal, full-field, missing/empty fields, malformed YAML, size limits, delimiter edge cases |
| F2 | Parse a `.persona.md` from disk with a pre-read size guard | `persona::parse_persona_file` (`crates/buzz-persona/src/persona.rs:262`) | Full | size check at `:263-267`; no dedicated test for the `TooLarge` path (see debt doc) |
| F3 | Frontmatter splitting with strict own-line closing delimiter | `persona::split_frontmatter` (`crates/buzz-persona/src/persona.rs:277`) | Full | scanner loop `:287-306`; tests `:482-501` |
| F4 | `provider:model-id` splitting on first colon | `persona::split_model` (`crates/buzz-persona/src/persona.rs:324`) | Full | tests `:545-563` |
| F5 | Parse `plugin.json` (OPS-superset manifest) | `manifest::parse_manifest` / `parse_manifest_file` (`crates/buzz-persona/src/manifest.rs:152`, `:190`) | Full | 14 tests `crates/buzz-persona/src/manifest.rs:195-364` |
| F6 | Load a whole pack directory (manifest + personas + instructions + `.mcp.json` + skills-dir detection) | `pack::load_pack` (`crates/buzz-persona/src/pack.rs:125`) | Full for the paths it reads; **partial** in coverage — pack-level `hooks/hooks.json` is deliberately never loaded (`crates/buzz-persona/src/pack.rs:111-112`) | 14 tests `crates/buzz-persona/src/pack.rs:447-734` |
| F7 | Path-traversal / symlink-escape defense for manifest-declared paths | `safe_resolve` (`crates/buzz-persona/src/pack.rs:323`) | Full for `personas`, `pack_instructions`, `mcp_config`. **Not applied** to hook paths (`crates/buzz-persona/src/resolve.rs:339-346`) or persona `skills` paths | tests `crates/buzz-persona/src/pack.rs:561-607` |
| F8 | Behavioral-config precedence merge (levels 3–5) | `merge::merge_behavioral_config`, `merge::resolve_persona_config` (`crates/buzz-persona/src/merge.rs:47`, `:85`) | Full | 22 tests `crates/buzz-persona/src/merge.rs:200-464` including null/empty-container semantics and shallow-replacement |
| F9 | Full pack resolution to ACP-ready output | `resolve::resolve_pack`, `resolve_loaded_pack` (`crates/buzz-persona/src/resolve.rs:108`, `:118`) | Full | 26 tests `crates/buzz-persona/src/resolve.rs:400-892` |
| F10 | Single-persona lookup by name | `resolve::resolve_persona_by_name` (`crates/buzz-persona/src/resolve.rs:186`) | Full — note it re-reads and re-resolves the entire pack (`:187`) | tests `:806-861` |
| F11 | MCP server merge (pack shared + per-persona, deterministic order) | `merge_mcp_servers` (`crates/buzz-persona/src/resolve.rs:277`) | Partial — merge works; `${VAR}` interpolation is explicitly out of scope | doc `:274`; test `:558-572` asserts literals survive |
| F12 | Env-var projection per agent runtime (`goose` vs `buzz-agent`) | `runtime_env_vars` (`crates/buzz-persona/src/resolve.rs:365`) | Full for the two known runtimes; every other runtime silently falls into the GOOSE_* branch (`:380`) | tests `:602-737`, `crates/buzz-persona/tests/e2e_env_flow.rs:35-356` |
| F13 | Lifecycle hooks | `resolve_hooks` (`crates/buzz-persona/src/resolve.rs:347`) | **Stubbed** — parsed and carried only. Field comment: "Hooks (parsed, not executed — reserved for future use, not yet wired)" (`crates/buzz-persona/src/resolve.rs:57`) | no process-spawn code anywhere in the crate |
| F14 | Skill scoping (claimed vs shared) | `pack::resolve_skills` (`crates/buzz-persona/src/pack.rs:249`) | **Stubbed / orphaned** — implemented and tested, but never called by `load_pack` or `resolve_pack`; `ResolvedPersona.skills` carries raw declared paths instead (`crates/buzz-persona/src/resolve.rs:249`), and the field is marked "reserved for future use, not yet wired" (`:60`) | grep for `resolve_skills` finds only the definition, its own tests, and no production caller |
| F15 | Pack validation with error/warning severity and exit codes | `validate::validate_pack` (`crates/buzz-persona/src/validate.rs:143`) | Partial — see gaps table below | 22 tests `crates/buzz-persona/src/validate.rs:436-963` |
| F16 | Human-readable validation output | `Display for ValidationReport` (`crates/buzz-persona/src/validate.rs:74-96`) | Full | rendered by `crates/buzz-cli/src/commands/pack.rs:26-35` (which iterates diagnostics itself rather than using `Display`) |
| F17 | `engines.buzz` version negotiation | field only (`crates/buzz-persona/src/manifest.rs:39-40`) | **Not implemented** — parsed, never compared; no semver dependency in `crates/buzz-persona/Cargo.toml:10-14` | — |
| F18 | Pack integrity (sha256 / `pack.lock`), zip/git install, pack packaging | — | **Absent** — no such code in the crate; `PERSONA_PACK_SPEC.md` §11/§13 describe it as buzz-acp/CLI responsibility | no `sha2`, `zip`, or HTTP dependency in `crates/buzz-persona/Cargo.toml` |
| F19 | Prompt templating / variable substitution | — | **Absent** — `system_prompt` is a verbatim copy (`crates/buzz-persona/src/resolve.rs:200`) | no template engine dependency |

---

### Validation feature gaps (F15 detail)

Measured against the checks the crate itself declares or that
`PERSONA_PACK_SPEC.md` §16 (PF-2) lists as remaining:

| Gap | Evidence |
|---|---|
| `skills:` paths are never checked for existence | `advisory_check_skill_names` only inspects directories that already exist (`crates/buzz-persona/src/validate.rs:365-370`); no "declared skill missing" diagnostic exists |
| `hooks:` paths are never checked for existence | no hook handling in `validate.rs` (grep for `hook` in that file matches only the `hooks_config` entry of `KNOWN_MANIFEST_KEYS` at `crates/buzz-persona/src/validate.rs:115`) |
| `SKILL.md` missing `name:`/`description:` is not an error | only `name` is read (`crates/buzz-persona/src/validate.rs:420-422`); `description` never inspected |
| Only the **first** structural fault is reported | early `return` at `crates/buzz-persona/src/validate.rs:148-151`; test comment acknowledges this: "load_pack fails on the first missing required field ... a single error is emitted" (`crates/buzz-persona/tests/integration.rs:296-299`) |
| `temperature` range is not validated | no range check in `merge.rs` or `validate.rs`; matches spec §10 which states type-only checking |
| Unknown persona-frontmatter keys produce a serde error message, not a targeted diagnostic | K1 path — `Error("pack failed to load: invalid file ...: failed to parse YAML frontmatter: unknown field ...")`; test asserts only that the key name appears (`crates/buzz-persona/src/validate.rs:707-733`) |

---

### TODO / FIXME / HACK / XXX comments

**Zero.** A grep for `TODO`, `FIXME`, `HACK`, and `XXX` across the entire
`crates/buzz-persona` directory (all `.rs` files, `Cargo.toml`, and
`PERSONA_PACK_SPEC.md`) returns no matches.

Deferred work is instead expressed as prose in doc comments. Verbatim, with locations:

| Location | Comment (verbatim) |
|---|---|
| `crates/buzz-persona/src/resolve.rs:57` | `// Hooks (parsed, not executed — reserved for future use, not yet wired)` |
| `crates/buzz-persona/src/resolve.rs:60` | `// Skills (bare names — reserved for future use, not yet wired)` |
| `crates/buzz-persona/src/resolve.rs:274` | `/// Env values are passed through as literals — no `${VAR}` interpolation.` |
| `crates/buzz-persona/src/resolve.rs:341-346` | `/// Security: we intentionally do NOT resolve these to absolute paths.` … `/// Hook paths come from untrusted persona frontmatter and could contain` / `/// `../` traversal. Since hooks are not executed in this PR, we store` / `/// them as-is. The PR that wires execution MUST validate through` / `/// `safe_resolve()` before use.` |
| `crates/buzz-persona/src/resolve.rs:225-231` | `// Version: LoadedPersona has no per-persona version field — persona files` / `// don't declare a version in frontmatter. The pack version is used as-is.` / `// If per-persona versioning is added in the future, LoadedPersona should` / `// gain `version: Option<String>` and this line should become:` / `//   lp.version.clone().unwrap_or_else(|| pack_version.to_owned())` |
| `crates/buzz-persona/src/resolve.rs:178-179` | `// Pack-level description not yet wired through PackManifestData.` |
| `crates/buzz-persona/src/pack.rs:111-112` | `// hooks_config is intentionally omitted: hooks are a runtime concern loaded` / `// separately by buzz-acp, not a pack-parsing concern.` |
| `crates/buzz-persona/src/persona.rs:230` | `// Fix #1: enforce non-empty required strings` |
| `crates/buzz-persona/src/persona.rs:263` | `// Fix #4: check file size before reading to avoid large allocations` |
| `crates/buzz-persona/src/merge.rs:9` | `/// Levels 1–2 (operator env vars, desktop UI) are resolved at runtime.` |

Note on `crates/buzz-persona/src/resolve.rs:178-179`: the comment says the pack
description is "not yet wired", but the code on the next line does read it
(`loaded.manifest.description.clone().unwrap_or_default()` at `:180`) and
`PackManifestData.description` exists (`crates/buzz-persona/src/pack.rs:106`) and is
populated (`:143`). The comment appears stale.

Two `// Fix #N:` markers (`persona.rs:230`, `persona.rs:263`) reference an external
review-item numbering that is not defined anywhere in the repo.

---

### Spec-vs-implementation deltas

`PERSONA_PACK_SPEC.md` ships inside this crate and describes behavior that lives partly
outside it. Items the spec marks as implemented-elsewhere or planned, cross-checked
against this crate's code:

| Spec item | Spec location | State in this crate |
|---|---|---|
| Skill copying to `$AGENT_CWD/.agents/skills/` | `PERSONA_PACK_SPEC.md:359-361` ("planned for a future release") | absent; `resolve_skills` computes the mapping but nothing copies |
| MCP `${VAR}` interpolation | `PERSONA_PACK_SPEC.md:512-514` ("planned but not yet implemented") | confirmed absent (`crates/buzz-persona/src/resolve.rs:274`) |
| Hook execution | `PERSONA_PACK_SPEC.md:551-552` ("parsed and validated at pack load time but not yet executed") | parsed only; pack-level hooks not even parsed (`crates/buzz-persona/src/pack.rs:111-112`) |
| `engines.buzz` rejection of too-new packs | `PERSONA_PACK_SPEC.md:113-114` | not implemented (F17) |
| `buzz pack validate` — "Remaining: verify `skills:` and `hooks:` paths exist; error on `SKILL.md` missing `name:` or `description:`" | `PERSONA_PACK_SPEC.md:1139` (PF-2) | confirmed still missing (gap table above) |
| Unknown keys in `defaults` are validation warnings; unknown persona frontmatter keys are hard errors | `PERSONA_PACK_SPEC.md:803-807` | matches implementation (K6, A8) |
| V6 namespaced `buzz:` block is unsupported | `PERSONA_PACK_SPEC.md:1108` | consistent — `buzz` is not a `Frontmatter` field, so such a file fails `deny_unknown_fields` |
