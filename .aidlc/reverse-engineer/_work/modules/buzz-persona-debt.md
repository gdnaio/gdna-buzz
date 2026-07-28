## Module: buzz-persona (`crates/buzz-persona`)

### Aspect: Technical Debt

### 1. Size and complexity profile

Total: 9 `.rs` files, 5,197 LOC (7 in `src/`, 2 in `tests/`).

| File | LOC | Production LOC (before `#[cfg(test)]`) | Test LOC | Test share |
|---|---|---|---|---|
| `crates/buzz-persona/src/validate.rs` | 1,070 | 435 | 635 | 59% |
| `crates/buzz-persona/src/resolve.rs` | 892 | 399 | 493 | 55% |
| `crates/buzz-persona/src/pack.rs` | 734 | 446 | 288 | 39% |
| `crates/buzz-persona/src/persona.rs` | 645 | 330 | 315 | 49% |
| `crates/buzz-persona/src/merge.rs` | 464 | 199 | 265 | 57% |
| `crates/buzz-persona/src/manifest.rs` | 365 | 194 | 171 | 47% |
| `crates/buzz-persona/src/lib.rs` | 6 | 6 | 0 | — |
| `crates/buzz-persona/tests/integration.rs` | 650 | — | 650 | — |
| `crates/buzz-persona/tests/e2e_env_flow.rs` | 371 | — | 371 | — |

Production code is ~2,009 LOC; tests are ~3,188 LOC (1.59:1 test-to-code). No file exceeds
the repo's 1,000-line desktop/mobile guard, though `validate.rs` at 1,070 would trip it if
that guard applied to Rust crates (it does not — `mobile/scripts/check-file-sizes.mjs` per
`AGENTS.md` covers mobile, desktop, and web only).

#### Largest functions (production only)

| Function | Approx. body length | Location |
|---|---|---|
| `load_pack` | ~123 lines | `crates/buzz-persona/src/pack.rs:125` |
| `resolve_persona_config` | ~87 lines | `crates/buzz-persona/src/merge.rs:85` |
| `advisory_check_skill_names` | ~81 lines | `crates/buzz-persona/src/validate.rs:354` |
| `resolve_skills` | ~73 lines | `crates/buzz-persona/src/pack.rs:249` |
| `resolve_loaded_pack` | ~67 lines | `crates/buzz-persona/src/resolve.rs:118` |
| `resolve_one_persona` | ~60 lines | `crates/buzz-persona/src/resolve.rs:194` |
| `parse_persona_file` (private) | ~54 lines | `crates/buzz-persona/src/pack.rs:392` |
| `parse_persona_md` | ~53 lines | `crates/buzz-persona/src/persona.rs:208` |
| `check_respond_to_value` | ~52 lines | `crates/buzz-persona/src/validate.rs:236` |
| `advisory_check_manifest_keys` | ~51 lines | `crates/buzz-persona/src/validate.rs:302` |

Hotspot notes:

- **`load_pack`** (`crates/buzz-persona/src/pack.rs:125-247`) does six distinct jobs in one
  body (manifest, personas, instructions, MCP, skills-dir, plus size-limit computation).
  Two nearly identical explicit-path-else-implicit-fallback blocks appear back to back
  (`:167-188` for instructions, `:191-220` for MCP) — the same four-step shape duplicated.
- **`resolve_persona_config`** (`crates/buzz-persona/src/merge.rs:85-171`) contains two
  hand-written `match (persona_value, default_value)` tuples with 4 and 6 arms
  (`:104-121` for `subscribe`, `:131-152` for `triggers`) that encode the same
  null-vs-empty-container rule twice with different arm structures. The `triggers` match
  ends with an unreachable-in-practice `_ => None` catch-all (`:151`).
- **`advisory_check_skill_names`** (`crates/buzz-persona/src/validate.rs:354-434`) has five
  chained early-`continue` guards and four nested `if let` levels (`:420-431`), so a
  `SKILL.md` that fails any step yields no diagnostic at all.

---

### 2. Dead / orphaned code

| # | Item | Evidence | Status |
|---|---|---|---|
| D1 | **`pack::resolve_skills` has no production caller.** It is fully implemented (73 lines) and covered by 4 tests, but neither `load_pack` nor `resolve_pack` nor `validate_pack` invokes it, and no consumer in the workspace calls it. | grep for `resolve_skills` across the repo yields only `crates/buzz-persona/src/pack.rs:249` (definition) and its own tests (`:611-682`) | orphan — 73 production lines + 4 tests exercising nothing in the live path |
| D2 | **`ResolvedPersona.skills` is documented as "bare names" but carries raw declared paths.** The field comment says "Skills (bare names — reserved for future use, not yet wired)" (`crates/buzz-persona/src/resolve.rs:60-61`) while the assignment is `skills: lp.skills.clone()` (`:249`), i.e. the un-normalized frontmatter strings such as `"./skills/code-review/"`. | `crates/buzz-persona/src/resolve.rs:60-61` vs `:249`; `crates/buzz-persona/tests/integration.rs:73-74` shows the raw form a persona declares | comment/code mismatch; `buzz-cli` prints these raw paths as-is (`crates/buzz-cli/src/commands/pack.rs:124-126`) |
| D3 | **`manifest::parse_manifest_file` is never called and never tested.** | only occurrence is the definition at `crates/buzz-persona/src/manifest.rs:190`; `load_pack` reads + parses manually instead (`crates/buzz-persona/src/pack.rs:136-137`) | untested public API |
| D4 | **`persona::parse_persona_file` is never called inside the crate and never tested.** `load_pack` uses its own private `parse_persona_file` with a different signature (`crates/buzz-persona/src/pack.rs:392`). No workspace consumer calls the public one either. | grep results: definition `crates/buzz-persona/src/persona.rs:262`, private homonym `crates/buzz-persona/src/pack.rs:392`, call site `:165` resolves to the private one | untested public API; the name collision is itself a readability hazard |
| D5 | **`PackManifest.hooks_config` is parsed then discarded.** Pack-level hooks can never take effect through this crate. | field `crates/buzz-persona/src/manifest.rs:116`; drop comment `crates/buzz-persona/src/pack.rs:111-112` | intentional, documented — but leaves a manifest field that silently does nothing |
| D6 | **`PersonaConfig.version` and `PersonaConfig.author` are parsed and then dropped.** Neither reaches `LoadedPersona` (field list `crates/buzz-persona/src/pack.rs:77-98`) nor `ResolvedPersona`; `ResolvedPersona.version` is always the pack version. | `crates/buzz-persona/src/persona.rs:116`, `:119`; `crates/buzz-persona/src/resolve.rs:225-231` | frontmatter `version:`/`author:` are accepted but inert |
| D7 | **`Engines`/`engines.buzz` is parsed and never evaluated.** | `crates/buzz-persona/src/manifest.rs:37-41`; no semver dependency in `crates/buzz-persona/Cargo.toml:9-15` | version negotiation described in `crates/buzz-persona/PERSONA_PACK_SPEC.md:113-114` is unimplemented |
| D8 | **`resolve::resolve_loaded_pack` has no direct test** — it is exercised only transitively through `resolve_pack` (`crates/buzz-persona/src/resolve.rs:110`). | grep: only `:110` and `:118` | untested-as-entry-point public API |
| D9 | **`"repository"` is warning-exempt in the validator but absent from `PackManifest`.** | `crates/buzz-persona/src/validate.rs:108` vs field list `crates/buzz-persona/src/manifest.rs:79-121` | schema drift between the known-keys list and the type |
| D10 | **`buzz-acp` declares `buzz-persona` as a dependency but never uses it.** | `crates/buzz-acp/Cargo.toml:22`; grep for `buzz_persona` across all of `crates/buzz-acp` returns zero matches | unused dependency — adds compile time; also means the crate's primary stated consumer is not actually wired |
| D11 | **Stale comment: "Pack-level description not yet wired through PackManifestData."** The very next line reads it successfully, and the field does exist and is populated. | comment `crates/buzz-persona/src/resolve.rs:178-179`; read `:180`; field `crates/buzz-persona/src/pack.rs:106`; populated `:142` | stale |

---

### 3. Duplication

| # | Duplicated logic | Sites | Risk |
|---|---|---|---|
| U1 | **Persona-name validation implemented twice, independently.** Charset + 64-char rules exist in `resolve_loaded_pack` and again in `validate_persona_name`, with different error plumbing and different constant handling (`MAX_NAME_LEN` const vs. an inline `64`). | `crates/buzz-persona/src/resolve.rs:134-155` vs `crates/buzz-persona/src/validate.rs:167-185` | the two can drift; a change to the rule must be made in both places |
| U2 | **Zero-persona and duplicate-name checks implemented twice.** | `crates/buzz-persona/src/resolve.rs:119-133` vs `crates/buzz-persona/src/validate.rs:189-203` | same as U1; the error strings are already near-identical but not shared ("pack contains zero personas" appears at both `resolve.rs:121` and `validate.rs:190`) |
| U3 | **Built-in trigger defaults declared twice.** `mentions=true / keywords=[] / all_messages=false` appear as literals in `parse_triggers` and again in `resolve_triggers`. | `crates/buzz-persona/src/merge.rs:180-198` vs `crates/buzz-persona/src/resolve.rs:255-268` | a default change in one place silently diverges |
| U4 | **Skill-path normalization implemented twice.** Identical `trim_end_matches('/')` → `file_name()` → fallback chain. | `crates/buzz-persona/src/pack.rs:253-260` (nested fn) vs `crates/buzz-persona/src/validate.rs:365-369` (inline) | |
| U5 | **`plugin.json` read + JSON-parsed three times per validation run.** Once in `load_pack` (`crates/buzz-persona/src/pack.rs:136-137`), then again in `advisory_check_respond_to_types` (`crates/buzz-persona/src/validate.rs:212-219`) and `advisory_check_manifest_keys` (`:304-312`). | 3 reads | wasted I/O plus a TOCTOU window where the validator could report on different bytes than were loaded |
| U6 | **Size-limit arithmetic repeated with inconsistent slack.** `MAX_FRONTMATTER + MAX_BODY + 100` in `persona.rs`, `+ 200` in `pack.rs` — and computed twice within `load_pack` under two different variable names for the same value. | `crates/buzz-persona/src/persona.rs:266`; `crates/buzz-persona/src/pack.rs:153-154` (`persona_size_limit`) and `:168-169` (`text_size_limit`) | the two `pack.rs` expressions are byte-identical yet duplicated; the 100-vs-200 divergence is unexplained |
| U7 | **`crate::persona::` fully-qualified path used where `persona::` is already imported.** `:168-169` writes `crate::persona::MAX_FRONTMATTER_BYTES` while `:153-154` writes `persona::MAX_FRONTMATTER_BYTES` for the same constant. | `crates/buzz-persona/src/pack.rs:153-154` vs `:168-169` | cosmetic inconsistency |
| U8 | **Consumer import-filter logic reimplemented in this crate's tests.** `DERIVED_PROVIDER_MODEL_ENV_KEYS` + `filter_derived` are described as mirroring "desktop import_persona_pack logic". | `crates/buzz-persona/tests/e2e_env_flow.rs:15-32`, comment at `:200` | a test that asserts a *copy* of behavior owned by another crate — will not catch drift in the real implementation |

---

### 4. Test coverage gaps

145 test functions total (per-module: `persona.rs` 29, `resolve.rs` 26, `merge.rs` 22,
`validate.rs` 22, `manifest.rs` 14, `pack.rs` 14; `tests/integration.rs` 13,
`tests/e2e_env_flow.rs` 5). Coverage is dense; the gaps are specific.

| # | Gap | Evidence |
|---|---|---|
| T1 | `persona::parse_persona_file` — no test at all; its `PersonaError::TooLarge` branch is therefore never exercised (only `FrontmatterTooLarge` and `BodyTooLarge` are asserted) | `crates/buzz-persona/src/persona.rs:262-270`; assertions exist only for `:551` and `:559` |
| T2 | `manifest::parse_manifest_file` — no test | `crates/buzz-persona/src/manifest.rs:190-193` |
| T3 | `resolve::resolve_loaded_pack` — no direct test | `crates/buzz-persona/src/resolve.rs:118` |
| T4 | Windows absolute-path rejection (`X:` drive prefix) is `#[cfg(windows)]` and has no test on any platform | `crates/buzz-persona/src/pack.rs:333-337` |
| T5 | `PackError::McpConfigParse` — the malformed-`.mcp.json` path has no test (the only `.mcp.json` test writes valid JSON at `crates/buzz-persona/src/pack.rs:513`) | error defined `crates/buzz-persona/src/pack.rs:52`, raised `:193-196` |
| T6 | Explicit-but-missing `pack_instructions` / `mcp_config` paths (the hard-error branches) are untested; only the implicit fallbacks are covered | branches at `crates/buzz-persona/src/pack.rs:173-178` and `:201-206` |
| T7 | `read_bounded_file`'s oversize rejection is untested (no test writes a >1.25 MiB pack file) | `crates/buzz-persona/src/pack.rs:378-384` |
| T8 | `merge_behavioral_config` non-object short-circuits (scalar `persona_config` or scalar `pack_defaults`) untested | `crates/buzz-persona/src/merge.rs:53-60` |
| T9 | MCP entries with a missing/non-string `command`, or a persona entry with no `name`, are silently dropped — untested | `crates/buzz-persona/src/resolve.rs:294`, `:312` |
| T10 | `.mcp.json` with a shape other than `{"mcpServers": {...}}` (silently contributes nothing) untested | `crates/buzz-persona/src/resolve.rs:285` |
| T11 | `check_respond_to_value` is effectively unreachable in practice — the typed `BehavioralDefaults` parse fails first (test at `crates/buzz-persona/src/validate.rs:900-936` acknowledges this: "Serde catches the type mismatch during manifest parsing"), so its 52 lines of type-error messages are never surfaced. No test asserts the function's own error strings. | `crates/buzz-persona/src/validate.rs:236-287` |
| T12 | Runtime values other than `"buzz-agent"` and `"goose"` (e.g. `"claude"`, `"codex"`, garbage) falling into the GOOSE_* branch — untested | wildcard arm `crates/buzz-persona/src/resolve.rs:380` |
| T13 | `resolve_skills` with a persona claiming a nonexistent skill name — untested | `crates/buzz-persona/src/pack.rs:305` emits it unconditionally |
| T14 | Duplicate entries in `manifest.personas[]` on the bare `load_pack` path (no dedup, no error) — untested | `crates/buzz-persona/src/pack.rs:155-164` |
| T15 | No test covers a persona `name` validated on the `parse_persona_md` path — name charset/length is only checked in `resolve`/`validate`, so a direct-parse consumer (like the desktop importer) is unguarded and no test documents that asymmetry | `crates/buzz-persona/src/persona.rs:208-260` vs `crates/buzz-persona/src/resolve.rs:134-155` |
| T16 | No fuzz/property tests on `split_frontmatter`, which is the most intricate parser in the crate (a hand-rolled delimiter scanner) | `crates/buzz-persona/src/persona.rs:277-319` |
| T17 | No test for CRLF (`\r\n`) input end to end — the code has three separate `\r` handling branches (`crates/buzz-persona/src/persona.rs:283`, `:299`, `:314`) but every fixture uses `\n` | fixtures throughout `crates/buzz-persona/src/persona.rs:334-644` |

---

### 5. Deprecated / legacy API usage

| # | Item | Evidence |
|---|---|---|
L1 | **`serde_yaml` 0.9** is the YAML backend. The upstream crate was archived/deprecated by its author; no maintained successor is pinned here. | `crates/buzz-persona/Cargo.toml:12` |
| L2 | **`respond_to` legacy alias is carried in three places** with no deprecation timeline: persona frontmatter (`crates/buzz-persona/src/persona.rs:186`), manifest defaults (`crates/buzz-persona/src/manifest.rs:62`), and the validator's known-keys list with the comment `// legacy alias — still accepted` (`crates/buzz-persona/src/validate.rs:124`). | as cited |
| L3 | **`Engines.buzz` carries `alias = "buzz"`, which aliases the field to its own name** — a no-op serde attribute. | `crates/buzz-persona/src/manifest.rs:39` |
| L4 | **Doc comment references "V7 spec" and "(V3 contract)" version labels** that do not correspond to anything in the repo (`PERSONA_PACK_SPEC.md` carries no version number; it documents migration *from* V6). | `crates/buzz-persona/src/persona.rs:95` ("V7 spec"), `crates/buzz-persona/src/resolve.rs:208` and `crates/buzz-persona/tests/integration.rs:389` ("V3 contract") |
| L5 | **PR-relative comments baked into source**: "no interpolation in this PR" (`crates/buzz-persona/src/resolve.rs:68`), "Since hooks are not executed in this PR" (`crates/buzz-persona/src/resolve.rs:344`), "// Fix #1:" / "// Fix #4:" (`crates/buzz-persona/src/persona.rs:230`, `:263`), and a test module titled "env var flow introduced in PRs #783 and #794" (`crates/buzz-persona/tests/e2e_env_flow.rs:1-2`). These lose meaning once the PR is merged. | as cited |
| L6 | **`Cargo.toml` metadata is stale and inconsistent with the workspace.** `repository = "https://github.com/block/sprout"` (pre-rename) and hard-coded `version`/`edition`/`license` instead of `workspace = true` inheritance used by sibling crates (compare `crates/buzz-acp/Cargo.toml:3-7`). | `crates/buzz-persona/Cargo.toml:1-8` |
| L7 | **`buzz-agent`/`sprout-agent` rename residue**: this crate hard-codes `"buzz-agent"` as the runtime discriminant (`crates/buzz-persona/src/resolve.rs:374`) while the desktop layer still migrates legacy `"sprout-agent"` values (`desktop/src-tauri/src/migration.rs:1102`, `:1128`) using this crate's `split_frontmatter`. The legacy value is not recognized here, so a legacy persona silently gets GOOSE_* env vars. | as cited |

---

### 6. Design-level debt

| # | Item | Evidence |
|---|---|---|
| A1 | **The merge layer is untyped.** `PersonaConfig` is serialized to `serde_json::Value` and merged as loose JSON (`crates/buzz-persona/src/pack.rs:406-409`, `crates/buzz-persona/src/merge.rs:47-83`). Consequences: (a) the merge silently tolerates wrong types via `and_then(as_f64/as_bool/as_u64)` (`crates/buzz-persona/src/merge.rs:96-97`, `:159-168`); (b) the merge depends on serde field *names* matching, which has already caused a real bug — the regression test at `crates/buzz-persona/src/merge.rs:400-403` documents "This was broken when BehavioralDefaults serialized the field as \"respond_to\" but merge looked for \"triggers\"". | as cited |
| A2 | **Serde-name coupling is load-bearing and unenforced.** Nothing prevents a future rename of a `PersonaConfig` or `BehavioralDefaults` field from silently breaking merge lookups again — the string keys in `merge.rs` (`"model"`, `"temperature"`, `"subscribe"`, `"triggers"`, `"thread_replies"`, `"broadcast_replies"`) are literals with no compile-time link to the structs. | `crates/buzz-persona/src/merge.rs:95-168` |
| A3 | **`resolve_persona_by_name` re-reads and fully re-resolves the entire pack to return one persona.** | `crates/buzz-persona/src/resolve.rs:186-192` |
| A4 | **Tri-state `subscribe` semantics are carefully built then thrown away.** `ResolvedConfig.subscribe: Option<Vec<String>>` distinguishes absent from "subscribe to nothing" (`crates/buzz-persona/src/merge.rs:29-31`), but `LoadedPersona.subscribe` collapses it with `unwrap_or_default()` (`crates/buzz-persona/src/pack.rs:432`), so downstream consumers cannot tell the two apart. | as cited |
| A5 | **Validation reports only the first structural error.** `validate_pack` returns immediately when `load_pack` fails (`crates/buzz-persona/src/validate.rs:148-151`), so a pack with five broken personas requires five validate-fix cycles. The test suite acknowledges this (`crates/buzz-persona/tests/integration.rs:296-299`). | as cited |
| A6 | **`ValidationReport::exit_code()` is never used by the only consumer.** `buzz-cli` re-derives its own exit behavior from `has_errors()`/`has_warnings()` (`crates/buzz-cli/src/commands/pack.rs:37-44`), and its mapping differs from the documented contract — the crate documents `2 = warnings only` (`crates/buzz-persona/src/validate.rs:62`) but the CLI returns success for warnings. | as cited |
| A7 | **`Display for ValidationReport` is never used either** — `buzz-cli` iterates `report.diagnostics` and formats its own output (`crates/buzz-cli/src/commands/pack.rs:26-35`), duplicating the `ERROR:`/`WARN:` prefixes already implemented at `crates/buzz-persona/src/validate.rs:26-31`. | as cited |
| A8 | **The crate's own 1,190-line spec (`PERSONA_PACK_SPEC.md`) documents behavior that mostly lives in other crates**, and marks six features as planned (PF-1…PF-6 at `crates/buzz-persona/PERSONA_PACK_SPEC.md:1136-1145`). It is not linked from any repo-level doc, so it will drift silently. | as cited |
| A9 | **Error-source chains are flattened at module boundaries**, so a caller sees a string, not a typed cause: `PackError::ManifestParse(e.to_string())` (`crates/buzz-persona/src/pack.rs:58`) and `FileParse { reason: e.to_string() }` (`crates/buzz-persona/src/pack.rs:395`). | as cited |

---

### 7. Missing documentation

| # | Gap | Evidence |
|---|---|---|
| M1 | **`ARCHITECTURE.md` does not mention this crate at all.** A grep for both `buzz-persona` and `persona` (case-insensitive) across `ARCHITECTURE.md` returns **zero** matches. The crate is absent from the dependency diagram at `ARCHITECTURE.md:78-95`, absent from the "Agent surface" grouping alongside `buzz-acp`/`buzz-cli`, and has no per-crate section (unlike `buzz-core` at `ARCHITECTURE.md:332`, `buzz-acp` at `:644`, `buzz-admin` at `:678`, etc.). | verified by grep |
| M2 | **`CONTRIBUTING.md` does not mention it either** (zero matches for `persona`). So there is no documented guidance on adding a persona field, adding a validation rule, or the pack-schema change process — despite `CONTRIBUTING.md` covering the equivalent for event kinds, CLI subcommands, and HTTP endpoints per `AGENTS.md`. | verified by grep |
| M3 | Existing coverage is one line each: `AGENTS.md:52` (`buzz-persona        # Agent persona packs`) and `README.md:207` (`· \`buzz-persona\` (agent persona packs)`). Neither describes the pack schema, the precedence model, or the resolve pipeline. | as cited |
| M4 | **`crates/buzz-persona/src/lib.rs` has no crate-level doc comment** — no `//!` header, so `cargo doc` renders a bare module list with no orientation. | `crates/buzz-persona/src/lib.rs:1-6` |
| M5 | **`pack.rs` and `merge.rs` use `///` where `//!` was intended.** The file-opening comments (`crates/buzz-persona/src/pack.rs:1-19`, `crates/buzz-persona/src/merge.rs:1-9`) are outer doc comments that attach to the following item instead of documenting the module — so `pack`'s directory-layout diagram documents the `use` statement, and `merge`'s precedence explanation documents `struct TriggersData`. | as cited |
| M6 | **No `TESTING.md` coverage.** `TESTING.md` has zero `persona` matches, so the pack-fixture testing approach (`tempfile` + hand-built pack dirs) is undocumented for contributors. | verified by grep |
| M7 | **The `PERSONA_PACK_SPEC.md` ↔ code relationship is undocumented.** The spec describes buzz-acp and desktop behavior extensively but never states which sections this crate actually implements; readers must diff it against the code (as done in the features doc for this module). | `crates/buzz-persona/PERSONA_PACK_SPEC.md` |
| M8 | **No CHANGELOG entries reference the crate.** `CHANGELOG.md` has 32 `persona` matches, all desktop/UI or test-stability entries (e.g. `CHANGELOG.md:228`, `:313`, `:361`) — none describe pack-schema or `buzz-persona` library changes, so schema evolution is untracked. | verified by grep |

---

### 8. Debt signal summary

| Category | Count | Highest-signal item |
|---|---|---|
| Complexity hotspots | 3 functions >70 lines | `load_pack` ~123 lines with 6 responsibilities (`crates/buzz-persona/src/pack.rs:125`) |
| Dead / orphaned code | 11 (D1–D11) | `resolve_skills` fully implemented and tested but never called (`crates/buzz-persona/src/pack.rs:249`) |
| Duplication | 8 (U1–U8) | persona-name validation implemented twice with a const vs. an inline literal (`crates/buzz-persona/src/resolve.rs:146` vs `crates/buzz-persona/src/validate.rs:168`) |
| Test coverage gaps | 17 (T1–T17) | three public functions with zero direct tests (`parse_persona_file`, `parse_manifest_file`, `resolve_loaded_pack`) |
| Deprecated / legacy usage | 7 (L1–L7) | `serde_yaml` 0.9 (unmaintained upstream) at `crates/buzz-persona/Cargo.toml:12` |
| Design-level debt | 9 (A1–A9) | untyped JSON merge layer that has already produced one field-name regression (`crates/buzz-persona/src/merge.rs:400-403`) |
| Missing documentation | 8 (M1–M8) | crate entirely absent from `ARCHITECTURE.md` and `CONTRIBUTING.md` |
| TODO/FIXME/HACK/XXX markers | **0** | deferred work is expressed as prose comments instead (see features doc) |
| `unsafe` blocks | **0** | — |
| `unwrap()`/`expect()` in production paths | **0** | — |
