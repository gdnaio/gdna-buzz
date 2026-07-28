## Module: buzz-persona (`crates/buzz-persona`)

### Aspect: Business Rules

Rules are grouped by stage. Each row states what is enforced, where, and what triggers it.

---

### A. `.persona.md` parsing rules

| # | Rule | Enforced at | Trigger | Outcome |
|---|---|---|---|---|
| A1 | File must begin with the literal `---` | `crates/buzz-persona/src/persona.rs:279-281` | any call to `split_frontmatter` / `parse_persona_md` | `PersonaError::NoFrontmatter` |
| A2 | Opening `---` must be followed by `\n` (an optional single `\r` is tolerated first) | `crates/buzz-persona/src/persona.rs:283-285` | `---junk` on line 1, or leading whitespace/newline before `---` | `PersonaError::NoFrontmatter` |
| A3 | Closing delimiter must be `\n---` followed by `\n`, `\r`, or EOF; `---junk` is skipped and the search continues | `crates/buzz-persona/src/persona.rs:287-306` | `---junk` inside a YAML block scalar | scanner advances past it and finds the real close (test: `crates/buzz-persona/src/persona.rs:492-501`) |
| A4 | No valid closing delimiter → parse failure | `crates/buzz-persona/src/persona.rs:293` | unterminated frontmatter | `PersonaError::NoFrontmatter` |
| A5 | Frontmatter ≤ 1 MiB | `crates/buzz-persona/src/persona.rs:213-215` (limit at `:21`) | oversized YAML block | `PersonaError::FrontmatterTooLarge` |
| A6 | Body ≤ 256 KiB | `crates/buzz-persona/src/persona.rs:216-218` (limit at `:24`) | oversized markdown body | `PersonaError::BodyTooLarge` |
| A7 | File size pre-check before read: `metadata.len() > MAX_FRONTMATTER + MAX_BODY + 100` | `crates/buzz-persona/src/persona.rs:263-267` | `parse_persona_file` on a >~1.25 MiB file | `PersonaError::TooLarge` (avoids the large allocation) |
| A8 | Unknown frontmatter keys are a **hard error** | `#[serde(deny_unknown_fields)]` at `crates/buzz-persona/src/persona.rs:175` | any key not in the `Frontmatter` struct (`:176-196`) | `PersonaError::Yaml` |
| A9 | Unknown keys inside `hooks:` are a hard error | `deny_unknown_fields` at `crates/buzz-persona/src/persona.rs:83` | e.g. `hooks.on_init` | `PersonaError::Yaml` (test: `:412-418`) |
| A10 | `name`, `display_name`, `description` are required | `crates/buzz-persona/src/persona.rs:222-228` | key absent | `PersonaError::MissingField("<field>")` |
| A11 | Those three must be non-empty after `trim()` | `crates/buzz-persona/src/persona.rs:231-239` | `name: ""` or `name: "   "` | `PersonaError::MissingField("name (empty)")` etc. |
| A12 | `respond_to` is accepted as an alias for `triggers` | `#[serde(alias = "respond_to")]` at `crates/buzz-persona/src/persona.rs:186` | legacy persona file | parsed into `triggers` |
| A13 | The markdown body — everything after the closing delimiter, with one leading `\r\n` or `\n` stripped — becomes `prompt` | `crates/buzz-persona/src/persona.rs:312-318`, assigned at `:257` | every parse | `PersonaConfig.prompt` |
| A14 | `prompt` cannot be set from YAML | `prompt` is absent from `Frontmatter` (`crates/buzz-persona/src/persona.rs:176-196`) | frontmatter containing `prompt:` | rejected by A8 |
| A15 | Empty body is valid | no non-empty check on `body`; test at `crates/buzz-persona/src/persona.rs:355-359` | persona with no prose | `prompt == ""` |

---

### B. `plugin.json` manifest rules

| # | Rule | Enforced at | Trigger | Outcome |
|---|---|---|---|---|
| B1 | `id`, `name`, `version` required | `crates/buzz-persona/src/manifest.rs:155-159` | key absent | `ManifestError::MissingField` |
| B2 | Those three must be non-empty after `trim()` | `crates/buzz-persona/src/manifest.rs:161-170` | `"id": "   "` | `ManifestError::MissingField("id (empty)")` |
| B3 | Unknown top-level manifest keys are **silently accepted** at parse time (OPS superset) | no `deny_unknown_fields` on `RawManifest`, rationale at `crates/buzz-persona/src/manifest.rs:123-130` | foreign OPS fields | ignored (advisory warning later — see E2) |
| B4 | `personas` is optional and defaults to empty | `#[serde(default)]` at `crates/buzz-persona/src/manifest.rs:141-142`; test at `:245-251` | manifest with no `personas` key | `Vec::new()` (later rejected by D1) |
| B5 | `defaults.respond_to` accepted as alias for `defaults.triggers` | `alias = "respond_to"` at `crates/buzz-persona/src/manifest.rs:62-63` | legacy manifest | parsed into `BehavioralDefaults.triggers` |
| B6 | Wrong types inside `defaults` are rejected by serde at parse time | typed `BehavioralDefaults` (`crates/buzz-persona/src/manifest.rs:49-70`) | `"mentions": "yes"` | `ManifestError::Json` → surfaces as a load error (test: `crates/buzz-persona/src/validate.rs:900-936`) |
| B7 | `engines.buzz` is parsed but **never evaluated** | field at `crates/buzz-persona/src/manifest.rs:39-40`; no version comparison exists anywhere in the crate | pack declaring `engines.buzz: ">=99.0.0"` | no rejection (spec §2 says buzz-acp rejects; not implemented here) |
| B8 | `hooks_config` is parsed into `PackManifest` but dropped by the loader | dropped at `crates/buzz-persona/src/pack.rs:111-112` (comment: "hooks are a runtime concern loaded separately by buzz-acp") | manifest declaring `hooks_config` | pack-level hooks never reach `LoadedPack` |

---

### C. Pack loading rules (`load_pack`)

| # | Rule | Enforced at | Trigger | Outcome |
|---|---|---|---|---|
| C1 | Pack directory must canonicalize (must exist) | `crates/buzz-persona/src/pack.rs:126-129` | nonexistent dir | `PackError::Io` |
| C2 | `.plugin/plugin.json` must exist | `crates/buzz-persona/src/pack.rs:132-135` | missing manifest | `PackError::ManifestNotFound` |
| C3 | Every path in `personas[]` must exist | `crates/buzz-persona/src/pack.rs:157-159` | dangling reference | `PackError::PersonaNotFound` |
| C4 | Persona files are size-capped at `MAX_FRONTMATTER + MAX_BODY + 200` bytes | limit computed at `crates/buzz-persona/src/pack.rs:153-154`, applied at `:160` via `read_bounded_file` (`:374-389`) | oversized file | `PackError::FileParse` ("file too large") |
| C5 | Absolute paths in the manifest are rejected (`/` prefix; on Windows also `X:`) | `crates/buzz-persona/src/pack.rs:330-337` | `"personas": ["/etc/passwd"]` | `PackError::PathTraversal` |
| C6 | Any `..` path component is rejected before canonicalization | `crates/buzz-persona/src/pack.rs:339-345` | `"../../etc/passwd"`, `"personas/../../x"` | `PackError::PathTraversal` (tests `:561-575`) |
| C7 | After canonicalization the path must still be inside `pack_root` (catches symlink escapes) | `crates/buzz-persona/src/pack.rs:359-364` | symlink in `personas/` pointing outside the pack | `PackError::PathEscape` (test `:590-607`) |
| C8 | Non-existent joined paths skip canonicalization and are returned as-is so the caller can emit "not found" | `crates/buzz-persona/src/pack.rs:349-355` | manifest referencing a missing file | `Ok(joined)` → C3 fires |
| C9 | Explicit `pack_instructions` path must exist; **implicit** `instructions.md` is optional | `crates/buzz-persona/src/pack.rs:170-181` vs `:182-188` | declared-but-missing file | `PackError::FileParse`; implicit-missing → `None` |
| C10 | Explicit `mcp_config` path must exist; **implicit** `.mcp.json` is optional | `crates/buzz-persona/src/pack.rs:198-207` vs `:208-217` | declared-but-missing file | `PackError::FileParse`; implicit-missing → `None` |
| C11 | `.mcp.json` must be syntactically valid JSON (any shape) | `crates/buzz-persona/src/pack.rs:192-197` | malformed JSON | `PackError::McpConfigParse` |
| C12 | `skills/` presence is detected only if it is a directory | `crates/buzz-persona/src/pack.rs:223-230` | `skills` as a plain file | `skills_dir == None` |
| C13 | Persona `runtime` is **not** subject to pack-defaults merging | `crates/buzz-persona/src/pack.rs:428` copies `pc.runtime` directly; `runtime` is not a field of `BehavioralDefaults` (`crates/buzz-persona/src/manifest.rs:49-70`) | pack trying to set a default runtime | ignored (advisory warning via E2) |
| C14 | MCP server entries missing `name`/`command` are dropped silently during JSON re-encode | `filter_map(...ok())` at `crates/buzz-persona/src/pack.rs:415-419` | serialization failure | entry omitted |

---

### D. Semantic validation (applied in `resolve_loaded_pack`)

These duplicate the checks in `validate.rs` but are enforced as hard errors on the
resolve path.

| # | Rule | Enforced at | Trigger | Outcome |
|---|---|---|---|---|
| D1 | A pack must contain ≥ 1 persona | `crates/buzz-persona/src/resolve.rs:119-123` | `personas: []` | `PackError::ManifestParse("pack contains zero personas")` |
| D2 | Persona names must be unique within a pack | `crates/buzz-persona/src/resolve.rs:125-133` | two personas named `bot` | `PackError::FileParse("duplicate persona name ...")` |
| D3 | Persona name charset: `[a-zA-Z0-9_-]` only | `crates/buzz-persona/src/resolve.rs:134-145` | `name: "my bot"` | `PackError::FileParse("... invalid characters ...")` |
| D4 | Persona name ≤ 64 characters | `crates/buzz-persona/src/resolve.rs:146-155` | 65-char name | `PackError::FileParse("... exceeds 64 characters ...")` |

Note: `name.len()` is byte length, not char count — a multi-byte name shorter than 64
chars can still trip D4. Non-ASCII names are already rejected by D3, so the practical
effect is limited to ASCII input.

---

### E. Precedence / resolution rules (levels 3–5)

The module doc states this crate handles levels 3–5 only; levels 1–2 (operator env
vars, desktop UI) are resolved at runtime by the consumer
(`crates/buzz-persona/src/merge.rs:1-9`).

| # | Rule | Enforced at | Trigger | Outcome |
|---|---|---|---|---|
| E1 | Per-persona frontmatter beats pack `defaults` beats built-in defaults | `merge_behavioral_config` at `crates/buzz-persona/src/merge.rs:47-83` | any field present on both sides | persona value wins (`:68-73`) |
| E2 | Persona value of `null` is treated as **absent** → falls through to pack default | `Some(Value::Null) | None => default_val.clone()` at `crates/buzz-persona/src/merge.rs:69-71` | `temperature: null` | pack default applies (test `:225-231`) |
| E3 | Persona keys not present in `defaults` are carried through unless null | `crates/buzz-persona/src/merge.rs:76-81` | persona sets a field the pack does not | persona value used |
| E4 | Non-object inputs short-circuit the merge | `crates/buzz-persona/src/merge.rs:53-60` | scalar JSON passed in | returns the other side unchanged |
| E5 | `subscribe: []` is **present**, not absent — it overrides the pack default | dedicated match at `crates/buzz-persona/src/merge.rs:106-113` | persona sets empty array | `Some(vec![])` = "subscribe to nothing" (test `:445-451`) |
| E6 | `subscribe` absent/null falls through to the pack default; if neither, `None` | `crates/buzz-persona/src/merge.rs:114-121` | neither side sets it | `None` → later becomes `vec![]` at `crates/buzz-persona/src/pack.rs:432` |
| E7 | Non-string entries inside `subscribe` are silently dropped | `filter_map(|v| v.as_str()...)` at `crates/buzz-persona/src/merge.rs:108-112` | `subscribe: [1, "x"]` in defaults JSON | only `"x"` survives |
| E8 | `triggers` uses **shallow replacement** — a persona `triggers` object discards the pack's object entirely | `crates/buzz-persona/src/merge.rs:131-152` (persona arm at `:147-149`) | persona sets only `mentions` | pack `keywords` are lost; missing sub-fields take **built-in** defaults (test `:353-368`) |
| E9 | `triggers: {}` is present-but-empty → all sub-fields take built-in defaults | same match arm as E8 + `parse_triggers` (`crates/buzz-persona/src/merge.rs:177-199`) | `triggers: {}` | `mentions=true`, `keywords=[]`, `all_messages=false` (test `:370-388`) |
| E10 | `triggers: null` or absent → pack default object used verbatim | `crates/buzz-persona/src/merge.rs:150-151` | legacy "unset" pattern | pack triggers applied (tests `:390-397`, `:340-351`) |
| E11 | Built-in trigger sub-defaults: `mentions=true`, `keywords=[]`, `all_messages=false` | `crates/buzz-persona/src/merge.rs:180-198` | any trigger object with missing sub-keys | those literals |
| E12 | Built-in `thread_replies=true`, `broadcast_replies=false` | constants at `crates/buzz-persona/src/merge.rs:38-39`, applied `:159-168` | neither persona nor pack sets them | those literals |
| E13 | `model` / `temperature` / `max_context_tokens` have **no** built-in default — they stay `None` | `crates/buzz-persona/src/merge.rs:95-97` | nothing sets them | `None` (consumer falls back) |
| E14 | Non-`Option` reads are type-tolerant: a wrong-typed merged value degrades to the default instead of erroring | `and_then(|v| v.as_f64())`, `as_bool()`, `as_u64()` at `crates/buzz-persona/src/merge.rs:96-97,159-168` | `temperature: "hot"` reaching the merge JSON | field becomes `None`/default silently |

---

### F. Prompt composition and template rules

| # | Rule | Enforced at | Trigger | Outcome |
|---|---|---|---|---|
| F1 | `system_prompt` is the persona markdown body **verbatim** — no templating, no variable substitution, no escaping | `let system_prompt = lp.prompt.clone();` at `crates/buzz-persona/src/resolve.rs:200` | every resolve | body copied byte-for-byte |
| F2 | Pack instructions are kept **separate** from the system prompt (not concatenated) | `crates/buzz-persona/src/resolve.rs:201-204`, field at `:34`; asserted in test `crates/buzz-persona/tests/integration.rs:394-407` | pack with `instructions.md` | `ResolvedPersona.pack_instructions` |
| F3 | Pack instructions are trimmed; whitespace-only content becomes `None` | `.map(str::trim).filter(|s| !s.is_empty())` at `crates/buzz-persona/src/resolve.rs:202-203` | `instructions.md` containing only newlines | `None` |
| F4 | There is **no** `${VAR}` interpolation anywhere — MCP `env` values pass through as literals | `parse_mcp_server_config` at `crates/buzz-persona/src/resolve.rs:325-333`; doc note at `:274`; test `crates/buzz-persona/src/resolve.rs:558-572` | `env: { SECRET: "${MY_SECRET}" }` | stored as the literal string `"${MY_SECRET}"` |
| F5 | No per-persona version: `ResolvedPersona.version` is always the pack version | `crates/buzz-persona/src/resolve.rs:225-231` | persona declaring `version: 1.2.3` in frontmatter | frontmatter `version` is parsed into `PersonaConfig` (`crates/buzz-persona/src/persona.rs:116`) but **not** propagated to `LoadedPersona`/`ResolvedPersona` |

---

### G. MCP server merge rules

| # | Rule | Enforced at | Trigger | Outcome |
|---|---|---|---|---|
| G1 | Pack `.mcp.json` servers form the base set, read from the `mcpServers` object | `crates/buzz-persona/src/resolve.rs:283-291` | `.mcp.json` present | keyed by object key |
| G2 | Per-persona `mcp_servers[]` are layered on top; **name collision → persona wins entirely** (no field merge) | `crates/buzz-persona/src/resolve.rs:293-300` | same server name on both sides | persona entry replaces (test `:513-535`) |
| G3 | A server entry without a string `command` is dropped silently | `?` on `command` in `parse_mcp_server_config` at `crates/buzz-persona/src/resolve.rs:312` | `.mcp.json` entry with no `command` | entry omitted, no diagnostic |
| G4 | A persona entry without a string `name` is skipped | `crates/buzz-persona/src/resolve.rs:294` | malformed persona MCP entry | entry omitted |
| G5 | Non-string `args` / `env` values are dropped; missing → empty | `crates/buzz-persona/src/resolve.rs:313-333` | `args: [1,2]` | empty args |
| G6 | Output order is deterministic: sorted by server name | `crates/buzz-persona/src/resolve.rs:303-306` | more than one server | alphabetical (test `:537-556`) |
| G7 | Any `.mcp.json` shape other than `{ "mcpServers": {...} }` contributes nothing | `get("mcpServers").and_then(as_object)` at `crates/buzz-persona/src/resolve.rs:285` | flat map of servers | silently ignored |

---

### H. Hooks rules

| # | Rule | Enforced at | Trigger | Outcome |
|---|---|---|---|---|
| H1 | Hooks are parsed and carried, **never executed** by this crate | field comment at `crates/buzz-persona/src/resolve.rs:57`; no `Command`/`process` usage in the crate | pack with `hooks:` | stored only |
| H2 | Hook paths are stored as **raw relative strings**, deliberately not resolved | `resolve_hooks` at `crates/buzz-persona/src/resolve.rs:339-357`, security rationale at `:339-346` | `on_start: "../../evil.sh"` | stored verbatim; the crate documents that the PR wiring execution MUST run `safe_resolve()` first |
| H3 | A hooks object with all three fields `None` collapses to `None` | `crates/buzz-persona/src/resolve.rs:349-351` | `hooks: {}` in frontmatter | `ResolvedPersona.hooks == None` (test `:592-600`) |
| H4 | Pack-level `hooks/hooks.json` is never loaded | see B8 (`crates/buzz-persona/src/pack.rs:111-112`) | pack shipping `hooks_config` | ignored entirely |

---

### I. Env-var projection rules

`runtime_env_vars` — `crates/buzz-persona/src/resolve.rs:365-397`. Pure: it does **not**
read the process environment (doc at `:359-364`).

| # | Rule | Enforced at | Trigger | Outcome |
|---|---|---|---|---|
| I1 | `model` splits on the **first** colon into provider + model id | `split_model` at `crates/buzz-persona/src/persona.rs:324-329`; used at `resolve.rs:371` and `:210-218` | `"databricks:gpt-5:preview"` | provider `databricks`, id `gpt-5:preview` (test `crates/buzz-persona/src/persona.rs:559-563`) |
| I2 | `runtime == "buzz-agent"` → emit `BUZZ_AGENT_MODEL` (+ `BUZZ_AGENT_PROVIDER` when a provider exists) | `crates/buzz-persona/src/resolve.rs:374-379` | persona `runtime: "buzz-agent"` | BUZZ_AGENT_* only; no GOOSE_* model/provider (test `crates/buzz-persona/tests/e2e_env_flow.rs:139-166`) |
| I3 | Any other runtime — including `None` and `"goose"` — falls into the GOOSE_* branch | wildcard `_ =>` arm at `crates/buzz-persona/src/resolve.rs:380-386` | `runtime: "claude"`, or absent | `GOOSE_PROVIDER` + `GOOSE_MODEL` (tests `crates/buzz-persona/src/resolve.rs:688-707`, `:709-717`) |
| I4 | Provider env var is omitted when the model has no colon prefix | `if let Some(p) = provider` at `crates/buzz-persona/src/resolve.rs:376,381` | `model: "gpt-4o"` | `GOOSE_MODEL` only (test `crates/buzz-persona/tests/e2e_env_flow.rs:333-356`) |
| I5 | `temperature` → `GOOSE_TEMPERATURE`, `max_context_tokens` → `GOOSE_CONTEXT_LIMIT`, regardless of runtime | `crates/buzz-persona/src/resolve.rs:390-396` with comment at `:389` | any runtime | always GOOSE_-prefixed |
| I6 | No model → no model/provider vars at all | `if let Some(model_str)` at `crates/buzz-persona/src/resolve.rs:369` | persona with no `model` | vars vector may be empty (test `crates/buzz-persona/src/resolve.rs:664-669`) |
| I7 | `llm_provider`/`model` on `ResolvedPersona` are `None` when the model string is empty or whitespace | guard at `crates/buzz-persona/src/resolve.rs:210-218` | `model: "   "` | both `None` — note `runtime_env_vars` uses a different guard (`Some(model_str)` only), so a whitespace model string would still project an env var |

---

### J. Skill scoping rules

`resolve_skills` — `crates/buzz-persona/src/pack.rs:249-321`. Not called from anywhere
else in the crate (`load_pack`/`resolve_pack` do not invoke it).

| # | Rule | Enforced at | Trigger | Outcome |
|---|---|---|---|---|
| J1 | Declared skill paths are normalized to the final path component | `normalize_skill_name` at `crates/buzz-persona/src/pack.rs:253-260` | `"./skills/security-review/"` | `"security-review"` (test `:667-682`) |
| J2 | Only directories under `skills/` count; dotfiles/dot-dirs are skipped | `crates/buzz-persona/src/pack.rs:275-293` | `skills/.hidden`, `skills/README.md` | excluded |
| J3 | A skill claimed by ≥ 1 persona is private to the claiming personas | `claimed` set at `crates/buzz-persona/src/pack.rs:263-268` + shared filter `:296-301` | one persona lists `web-search` | other personas don't get it (test `:611-635`) |
| J4 | An unclaimed skill directory is granted to **all** personas | `crates/buzz-persona/src/pack.rs:296-312` | `skills/code-review` listed by nobody | every persona gets it (test `:637-652`) |
| J5 | Personas can claim skills that don't exist on disk — they are still returned | claimed names are emitted from `p.skills` unconditionally (`crates/buzz-persona/src/pack.rs:305`) with no existence check | `skills: ["ghost"]` | `"ghost"` appears in the map |
| J6 | No `skills/` directory → shared set is empty | `crates/buzz-persona/src/pack.rs:290-293` | pack without `skills/` | personas get only their declared names (test `:654-659`) |

---

### K. Validation rules (`validate_pack`)

Architecture: structural validation is delegated to `load_pack()`; advisory checks run
on raw files afterwards (`crates/buzz-persona/src/validate.rs:1-11`).

| # | Rule | Enforced at | Trigger | Outcome |
|---|---|---|---|---|
| K1 | If `load_pack()` fails, one error is recorded and validation stops | `crates/buzz-persona/src/validate.rs:145-152` | any structural fault (C1–C14) | single `Error("pack failed to load: ...")` — later checks skipped, so only the first fault is reported |
| K2 | Zero personas → error, and duplicate/name checks are skipped | `crates/buzz-persona/src/validate.rs:189-192` | `personas: []` | `Error("pack contains zero personas")` |
| K3 | Duplicate persona names → error (one per extra duplicate) | `crates/buzz-persona/src/validate.rs:195-203` | two `bot` personas | `Error("duplicate persona name \"bot\"")` |
| K4 | Persona name ≤ 64 chars and `[a-zA-Z0-9_-]` only | `validate_persona_name` at `crates/buzz-persona/src/validate.rs:167-185` | `"my bot"`, 65-char name | `Error` (duplicates rules D3/D4) |
| K5 | Unknown top-level `plugin.json` key → **warning** | `crates/buzz-persona/src/validate.rs:319-324` against `KNOWN_MANIFEST_KEYS` (`:99-118`) | `"totally_made_up": true` | `Warning("plugin.json unknown key ...")` |
| K6 | Unknown key in `defaults` → warning | `crates/buzz-persona/src/validate.rs:326-333` against `KNOWN_BEHAVIORAL_KEYS` (`:121-130`) | `"temprature": 0.5` | `Warning` (test `:568-604`) |
| K7 | Unknown sub-key in `defaults.triggers` / `defaults.respond_to` → warning | `crates/buzz-persona/src/validate.rs:335-348` against `KNOWN_RESPOND_TO_KEYS` (`:133`) | `triggers.mentioned` | `Warning` |
| K8 | `defaults.triggers` sub-key type errors → **error** | `check_respond_to_value` at `crates/buzz-persona/src/validate.rs:236-287`, invoked `:210-234` | `mentions: "yes"`, `keywords: [42]` | `Error(...)` — in practice `load_pack` (B6) already fails first, so K1 short-circuits this path |
| K9 | `SKILL.md` `name:` differing from its directory name → warning | `advisory_check_skill_names` at `crates/buzz-persona/src/validate.rs:354-434`, comparison `:420-431` | dir `code-review` with `name: code_review` | `Warning` (test `:735-770`) |
| K10 | Skill dirs with no `SKILL.md`, unreadable content, or malformed frontmatter are skipped silently | `crates/buzz-persona/src/validate.rs:392-418` | missing/broken `SKILL.md` | no diagnostic (spec PF-5 lists this as unimplemented) |
| K11 | `SKILL.md` missing `name:`/`description:` is **not** an error | only `name` is read (`crates/buzz-persona/src/validate.rs:420`); `description` is never inspected | skill without `description:` | no diagnostic |
| K12 | Exit code contract: 0 clean, 1 errors, 2 warnings-only | `ValidationReport::exit_code` at `crates/buzz-persona/src/validate.rs:63-71` | — | i32 (note: `buzz-cli` does not use this value; it maps errors to `CliError::Usage` at `crates/buzz-cli/src/commands/pack.rs:37-44`) |
| K13 | Advisory checks read the raw manifest again and bail silently if unreadable/unparseable | `crates/buzz-persona/src/validate.rs:212-219`, `:304-312` | manifest deleted between load and check | checks skipped |

---

### L. Capability / permission gating

| # | Finding | Evidence |
|---|---|---|
| L1 | The crate has **no** tool-permission, allow-list, or capability model. There is no `permissions`, `allowed_tools`, `tools`, or `permission_mode` field in any struct. | full field lists in `crates/buzz-persona/src/persona.rs:101-170`, `manifest.rs:79-121`, `resolve.rs:23-65` |
| L2 | Agent capability is granted **indirectly** — a persona's `mcp_servers` entries (`command` + `args` + `env`) define which external tool processes the runtime will launch. | `crates/buzz-persona/src/persona.rs:70-79`; merged at `crates/buzz-persona/src/resolve.rs:277-307` |
| L3 | Operator-owned knobs are deliberately **rejected** in persona frontmatter — `idle_timeout`, `max_turn_duration`, `agents`, `heartbeat_interval`, `permission_mode` all fail `deny_unknown_fields`. The test calls this "the security boundary: pack authors define behavior, operators define limits". | `crates/buzz-persona/tests/integration.rs:637-650` |
