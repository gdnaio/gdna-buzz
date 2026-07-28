## Module: buzz-persona (`crates/buzz-persona`)

### Aspect: API Surface

### Crate root

`crates/buzz-persona/src/lib.rs:1-6` — six public modules, no re-exports, no prelude,
no crate-level doc comment:

```rust
pub mod manifest;
pub mod merge;
pub mod pack;
pub mod persona;
pub mod resolve;
pub mod validate;
```

Every item below is reached by its full module path (e.g. `buzz_persona::resolve::resolve_pack`).

---

### Built-in personas shipped in the crate

**None.** There are zero built-in / bundled personas.

Verified by: full file listing of `crates/buzz-persona` (only `Cargo.toml`,
`PERSONA_PACK_SPEC.md`, 7 files in `src/`, 2 files in `tests/`); repo-wide grep for
`include_str!`, `include_dir!`, `include_bytes!` inside the crate returns no matches;
no `[build-dependencies]`, `build.rs`, or asset directory in
`crates/buzz-persona/Cargo.toml:1-16`.

Persona names that appear in the crate are **test fixtures only**, not shipped
artifacts:

| Fixture name | Where | Role in the test |
|---|---|---|
| `berry` | `crates/buzz-persona/src/pack.rs:481-487` (`SIMPLE_PERSONA`) | minimal valid persona for loader tests |
| `pip`, `lep` | `crates/buzz-persona/tests/integration.rs:57-93` | pack-defaults override vs inherit |
| `alpha`, `beta`, `gamma` | `crates/buzz-persona/tests/integration.rs:471-490` | 3-persona defaults matrix |
| `bot`, `test-agent`, `my-bot`, `full-bot`, `t`, `goose-bot`, `buzz-bot` | `crates/buzz-persona/src/persona.rs:334-336`, `crates/buzz-persona/src/resolve.rs:461`, `crates/buzz-persona/tests/e2e_env_flow.rs:129`, `crates/buzz-persona/tests/e2e_env_flow.rs:277-289` | parser/env-projection unit tests |

The `PERSONA_PACK_SPEC.md` examples (`pip`, `lep`, `thistle`, `berry` as a "Meadow
Security Team") are documentation only — no corresponding files exist in the crate.

---

### `persona` module — `crates/buzz-persona/src/persona.rs`

| Item | Kind | Signature / definition | Line |
|---|---|---|---|
| `MAX_FRONTMATTER_BYTES` | `pub const usize` | `1_048_576` | `:21` |
| `MAX_BODY_BYTES` | `pub const usize` | `262_144` | `:24` |
| `PersonaError` | `pub enum` (thiserror) | see variants below | `:27` |
| `RespondTo` | `pub struct` | `mentions: Option<bool>`, `keywords: Vec<String>`, `all_messages: Option<bool>` | `:53` |
| `McpServerConfig` | `pub struct` | `name: String`, `command: String`, `args: Vec<String>`, `env: HashMap<String,String>` | `:70` |
| `Hooks` | `pub struct` | `on_start`/`on_stop`/`on_message`: `Option<String>` | `:84` |
| `PersonaConfig` | `pub struct` | 18 fields (see data-model doc) | `:101` |
| `parse_persona_md` | `pub fn` | `fn(content: &str) -> Result<PersonaConfig, PersonaError>` | `:208` |
| `parse_persona_file` | `pub fn` | `fn(path: &Path) -> Result<PersonaConfig, PersonaError>` | `:262` |
| `split_frontmatter` | `pub fn` | `fn(content: &str) -> Result<(&str, &str), PersonaError>` | `:277` |
| `split_model` | `pub fn` | `fn(model: &str) -> (Option<&str>, &str)` | `:324` |

`PersonaError` variants — `crates/buzz-persona/src/persona.rs:27-48`:

| Variant | Payload | Display message | Line |
|---|---|---|---|
| `Io` | `#[from] std::io::Error` | `failed to read file: {0}` | `:28-29` |
| `NoFrontmatter` | — | `missing \`---\` frontmatter delimiters` | `:31-32` |
| `FrontmatterTooLarge` | — | `frontmatter exceeds {MAX_FRONTMATTER_BYTES} bytes` | `:34-35` |
| `BodyTooLarge` | — | `body exceeds {MAX_BODY_BYTES} bytes` | `:37-38` |
| `TooLarge` | `String` | `file too large: {0}` | `:40-41` |
| `Yaml` | `#[from] serde_yaml::Error` | `failed to parse YAML frontmatter: {0}` | `:43-44` |
| `MissingField` | `String` | `missing required field: {0}` | `:46-47` |

---

### `manifest` module — `crates/buzz-persona/src/manifest.rs`

| Item | Kind | Signature / definition | Line |
|---|---|---|---|
| `ManifestError` | `pub enum` (thiserror) | `Io(#[from] io::Error)`, `Json(#[from] serde_json::Error)`, `MissingField(String)` | `:23-31` |
| `Engines` | `pub struct` | `buzz: Option<String>` | `:37` |
| `BehavioralDefaults` | `pub struct` | 7 optional behavioral fields | `:49` |
| `PackManifest` | `pub struct` | 14 fields | `:79` |
| `parse_manifest` | `pub fn` | `fn(content: &str) -> Result<PackManifest, ManifestError>` | `:152` |
| `parse_manifest_file` | `pub fn` | `fn(path: &Path) -> Result<PackManifest, ManifestError>` | `:190` |

---

### `pack` module — `crates/buzz-persona/src/pack.rs`

| Item | Kind | Signature / definition | Line |
|---|---|---|---|
| `PackError` | `pub enum` (thiserror) | 8 variants, see below | `:26` |
| `LoadedPack` | `pub struct` | `manifest`, `personas`, `pack_instructions`, `shared_mcp_config`, `skills_dir` | `:64` |
| `LoadedPersona` | `pub struct` | 18 fields | `:77` |
| `PackManifestData` | `pub struct` | 8 fields | `:102` |
| `load_pack` | `pub fn` | `fn(pack_dir: &Path) -> Result<LoadedPack, PackError>` | `:125` |
| `resolve_skills` | `pub fn` | `fn(pack_dir: &Path, personas: &[LoadedPersona]) -> HashMap<String, Vec<String>>` | `:249` |
| `impl From<ManifestError> for PackError` | trait impl | flattens to `PackError::ManifestParse(e.to_string())` | `:56-60` |

`PackError` variants — `crates/buzz-persona/src/pack.rs:26-54`:

| Variant | Payload | Display | Line |
|---|---|---|---|
| `ManifestNotFound` | `PathBuf` | `manifest not found at {0}` | `:27-28` |
| `Io` | `{ path: PathBuf, source: io::Error }` | `failed to read {path}: {source}` | `:30-35` |
| `ManifestParse` | `String` | `failed to parse manifest: {0}` | `:37-38` |
| `PersonaNotFound` | `PathBuf` | `persona file not found: {0}` | `:40-41` |
| `FileParse` | `{ path: PathBuf, reason: String }` | `invalid file {path}: {reason}` | `:43-44` |
| `PathTraversal` | `String` | `path traversal rejected: {0}` | `:46-47` |
| `PathEscape` | `PathBuf` | `path escapes pack root: {0}` | `:49-50` |
| `McpConfigParse` | `{ path: PathBuf, reason: String }` | `failed to parse .mcp.json at {path}: {reason}` | `:52-53` |

Private helpers (not public API): `safe_resolve` (`:323`), `read_file` (`:366`),
`read_bounded_file` (`:374`), `parse_persona_file` (`:392` — note: name collides with
the public `persona::parse_persona_file` but is a distinct private fn).

---

### `merge` module — `crates/buzz-persona/src/merge.rs`

| Item | Kind | Signature / definition | Line |
|---|---|---|---|
| `TriggersData` | `pub struct` | `mentions: bool`, `keywords: Vec<String>`, `all_messages: bool` | `:11` |
| `HooksData` | `pub struct` | three `Option<String>` | `:18` |
| `ResolvedConfig` | `pub struct` | 7 fields | `:25` |
| `merge_behavioral_config` | `pub fn` | `fn(persona_config: &serde_json::Value, pack_defaults: &serde_json::Value) -> serde_json::Value` | `:47-50` |
| `resolve_persona_config` | `pub fn` | `fn(persona_frontmatter: &serde_json::Value, pack_defaults: Option<&serde_json::Value>) -> ResolvedConfig` | `:85-88` |

Private: `string_field` (`:173`), `parse_triggers` (`:177`). Constants
`DEFAULT_THREAD_REPLIES` / `DEFAULT_BROADCAST_REPLIES` are private
(`crates/buzz-persona/src/merge.rs:38-39`).

---

### `resolve` module — `crates/buzz-persona/src/resolve.rs`

| Item | Kind | Signature / definition | Line |
|---|---|---|---|
| `ResolvedPersona` | `pub struct` | 19 fields | `:23` |
| `ResolvedMcpServer` | `pub struct` | `name`, `command`, `args`, `env: Vec<(String,String)>` | `:69` |
| `ResolvedHooks` | `pub struct` | three `Option<String>` | `:78` |
| `ResolvedTriggers` | `pub struct` | `mentions: bool`, `keywords: Vec<String>`, `all_messages: bool` | `:86` |
| `ResolvedPack` | `pub struct` | `id`, `name`, `version`, `description`, `personas` | `:94` |
| `resolve_pack` | `pub fn` | `fn(pack_dir: &Path) -> Result<ResolvedPack, PackError>` | `:108` |
| `resolve_loaded_pack` | `pub fn` | `fn(loaded: &LoadedPack) -> Result<ResolvedPack, PackError>` | `:118` |
| `resolve_persona_by_name` | `pub fn` | `fn(pack_dir: &Path, name: &str) -> Result<ResolvedPersona, PackError>` | `:186` |

Private: `resolve_one_persona` (`:194`), `resolve_triggers` (`:255`),
`merge_mcp_servers` (`:277`), `parse_mcp_server_config` (`:311`), `resolve_hooks` (`:347`),
`runtime_env_vars` (`:365`).

---

### `validate` module — `crates/buzz-persona/src/validate.rs`

| Item | Kind | Signature / definition | Line |
|---|---|---|---|
| `ValidationDiagnostic` | `pub enum` | `Error(String)`, `Warning(String)` | `:19` |
| `ValidationReport` | `pub struct` | `diagnostics: Vec<ValidationDiagnostic>` | `:35` |
| `ValidationReport::error` | `pub fn` | `fn(&mut self, msg: impl Into<String>)` | `:40` |
| `ValidationReport::warn` | `pub fn` | `fn(&mut self, msg: impl Into<String>)` | `:45` |
| `ValidationReport::has_errors` | `pub fn` | `fn(&self) -> bool` | `:50` |
| `ValidationReport::has_warnings` | `pub fn` | `fn(&self) -> bool` | `:56` |
| `ValidationReport::exit_code` | `pub fn` | `fn(&self) -> i32` — 0 clean / 1 errors / 2 warnings-only | `:63` |
| `validate_pack` | `pub fn` | `fn(pack_dir: &Path) -> ValidationReport` | `:143` |
| `impl Display for ValidationDiagnostic` | trait impl | `ERROR: {msg}` / `WARN:  {msg}` | `:24-33` |
| `impl Display for ValidationReport` | trait impl | `✓ Pack is valid.` or list + `{errors} error(s), {warnings} warning(s).` | `:74-96` |

Private constants: `KNOWN_MANIFEST_KEYS` (`:99-118`), `KNOWN_BEHAVIORAL_KEYS` (`:121-130`),
`KNOWN_RESPOND_TO_KEYS` (`:133`). Private fns: `validate_persona_name` (`:167`),
`semantic_check_personas` (`:187`), `advisory_check_respond_to_types` (`:210`),
`check_respond_to_value` (`:236`), `value_type_name` (`:289`),
`advisory_check_manifest_keys` (`:302`), `advisory_check_skill_names` (`:354`).

---

### Traits

No traits are **defined** by this crate. Derived/implemented trait surface:

| Trait | On | Where |
|---|---|---|
| `Serialize` + `Deserialize` | `RespondTo`, `McpServerConfig`, `Hooks`, `PersonaConfig`, `Engines`, `BehavioralDefaults`, `PackManifest` | `persona.rs:51`, `:68`, `:82`, `:99`; `manifest.rs:35`, `:47`, `:77` |
| `Deserialize` only | private `Frontmatter`, `RawManifest` | `persona.rs:174`, `manifest.rs:130` |
| `std::error::Error` (via `thiserror::Error`) | `PersonaError`, `ManifestError`, `PackError` | `persona.rs:26`, `manifest.rs:22`, `pack.rs:25` |
| `From<ManifestError>` | `PackError` | `pack.rs:56` |
| `Display` | `ValidationDiagnostic`, `ValidationReport` | `validate.rs:24`, `:74` |
| `Debug` | all public types | derives on each struct/enum |
| `Clone` | `RespondTo`, `McpServerConfig`, `Hooks`, `PersonaConfig`, `Engines`, `BehavioralDefaults`, `PackManifest`, `TriggersData`, `HooksData`, `ResolvedConfig`, `ResolvedPersona`, `ResolvedMcpServer`, `ResolvedHooks`, `ResolvedTriggers`, `ValidationDiagnostic` | respective derives |
| `PartialEq` | `TriggersData`, `HooksData`, `ResolvedConfig`, `ResolvedMcpServer`, `ResolvedHooks`, `ResolvedTriggers` | `merge.rs:10`, `:17`, `:24`; `resolve.rs:68`, `:77`, `:85` |
| `Default` | `ValidationReport` | `validate.rs:34` |

Not implemented anywhere: `Copy`, `Hash`, `Ord`, `Send`/`Sync` (auto), custom
`Deserialize` impls, builder types.

---

### Entry-point summary (three practical call paths)

| Goal | Call | Returns |
|---|---|---|
| Validate a pack directory | `validate::validate_pack(&Path)` | `ValidationReport` (never `Err`) |
| Get ACP-ready output for all personas | `resolve::resolve_pack(&Path)` | `Result<ResolvedPack, PackError>` |
| Get one persona by name | `resolve::resolve_persona_by_name(&Path, &str)` | `Result<ResolvedPersona, PackError>` |
| Parse a single standalone `.persona.md` | `persona::parse_persona_md(&str)` / `parse_persona_file(&Path)` | `Result<PersonaConfig, PersonaError>` |
| Split frontmatter from any markdown file | `persona::split_frontmatter(&str)` | `Result<(&str, &str), PersonaError>` |
| Map skills → personas | `pack::resolve_skills(&Path, &[LoadedPersona])` | `HashMap<String, Vec<String>>` |
