## Module: buzz-persona (`crates/buzz-persona`)

### Aspect: Integrations

### External crate dependencies

`crates/buzz-persona/Cargo.toml:9-16`. Note: unlike most crates in this workspace, these
are **direct version pins, not `workspace = true`** entries.

| Crate | Version | Features | Why (evidence) |
|---|---|---|---|
| `serde` | `1` | `derive` | `Serialize`/`Deserialize` derives on all public config types — `crates/buzz-persona/src/persona.rs:51`, `:68`, `:82`, `:99`; `crates/buzz-persona/src/manifest.rs:35`, `:47`, `:77` |
| `serde_json` | `1` | default | Parses `plugin.json` (`crates/buzz-persona/src/manifest.rs:153`) and `.mcp.json` (`crates/buzz-persona/src/pack.rs:192-197`); also used as the **internal merge medium** — `PersonaConfig` is round-tripped to `serde_json::Value` so the JSON-based merge can run (`crates/buzz-persona/src/pack.rs:406-409`, `crates/buzz-persona/src/merge.rs:47-83`) |
| `serde_yaml` | `0.9` | default | Parses persona YAML frontmatter (`crates/buzz-persona/src/persona.rs:220`) and `SKILL.md` frontmatter during advisory validation (`crates/buzz-persona/src/validate.rs:410-418`) |
| `thiserror` | `2` | default | All three error enums: `PersonaError` (`crates/buzz-persona/src/persona.rs:26`), `ManifestError` (`crates/buzz-persona/src/manifest.rs:22`), `PackError` (`crates/buzz-persona/src/pack.rs:25`) |

Dev-dependencies — `crates/buzz-persona/Cargo.toml:17-18`:

| Crate | Version | Why |
|---|---|---|
| `tempfile` | `3` | Temp pack directories in tests — `crates/buzz-persona/src/pack.rs:450`, `crates/buzz-persona/src/resolve.rs:739`, `crates/buzz-persona/src/validate.rs:462`, `crates/buzz-persona/tests/integration.rs:117`, `crates/buzz-persona/tests/e2e_env_flow.rs:37` |

No internal Buzz crates are depended on — `crates/buzz-persona/Cargo.toml` has no
`buzz-*` entries. The crate is a leaf.

`serde_yaml` 0.9 is unmaintained upstream (archived by its author); this is a supply-chain
observation recorded in the debt doc, not a code finding.

---

### Filesystem access

All I/O is synchronous `std::fs`. There is no async runtime dependency.

| Operation | Call site | Path source |
|---|---|---|
| `canonicalize` pack root | `crates/buzz-persona/src/pack.rs:126` | caller-supplied `pack_dir` |
| `exists()` manifest probe | `crates/buzz-persona/src/pack.rs:133` | `<root>/.plugin/plugin.json` |
| `read_to_string` | `crates/buzz-persona/src/pack.rs:367` (via `read_file`) | manifest, personas, instructions, `.mcp.json` |
| `metadata` size guard | `crates/buzz-persona/src/pack.rs:375`, `crates/buzz-persona/src/persona.rs:264` | any file about to be read |
| `canonicalize` declared paths | `crates/buzz-persona/src/pack.rs:357` | manifest-declared relative paths |
| `is_dir()` skills probe | `crates/buzz-persona/src/pack.rs:224`, `:277` | `<root>/skills` |
| `read_dir` skills enumeration | `crates/buzz-persona/src/pack.rs:278`, `crates/buzz-persona/src/validate.rs:377` | `<root>/skills` |
| `read_to_string` manifest (2nd read) | `crates/buzz-persona/src/validate.rs:213`, `:305` | advisory checks re-read `plugin.json` |
| `read_to_string` SKILL.md | `crates/buzz-persona/src/validate.rs:393` | `<root>/skills/<name>/SKILL.md` |

Writes: **none**. There is no `fs::write`, `create_dir`, `remove_*`, or `File::create`
in `crates/buzz-persona/src/` (all such calls are inside `#[cfg(test)]` blocks and the
`tests/` directory).

Process execution: **none**. No `std::process::Command`, no `exec`. Hooks are data only
(`crates/buzz-persona/src/resolve.rs:339-357`).

---

### Network access

**None.** No HTTP client, no socket, no URL fetching. `crates/buzz-persona/Cargo.toml`
declares no `reqwest`/`hyper`/`ureq`/`tokio`. `homepage` and `repository` manifest fields
(`crates/buzz-persona/src/manifest.rs:94`, and `"repository"` in
`KNOWN_MANIFEST_KEYS` at `crates/buzz-persona/src/validate.rs:108`) are stored as opaque
strings and never dereferenced.

`resolve.rs` states the design contract explicitly: "**Pure**: no env access, no network,
no side effects" (`crates/buzz-persona/src/resolve.rs:11`). That holds for `resolve.rs`
itself; `resolve_pack` does perform filesystem reads transitively via `pack::load_pack`
(`crates/buzz-persona/src/resolve.rs:109`).

---

### Declared consumers in the workspace

| Consumer | Dependency declaration | Actual code usage |
|---|---|---|
| `buzz-cli` | `crates/buzz-cli/Cargo.toml:70` — `buzz-persona = { path = "../buzz-persona" }` | **Yes** — `crates/buzz-cli/src/commands/pack.rs:24`, `:28`, `:31`, `:62` |
| `buzz-acp` | `crates/buzz-acp/Cargo.toml:22` — `buzz-persona = { path = "../buzz-persona" }` | **No** — a grep for `buzz_persona` across all of `crates/buzz-acp` returns zero matches. The dependency is declared but unused at the code level. |
| desktop Tauri backend | `desktop/src-tauri/Cargo.toml:89` — `buzz_persona_pkg = { package = "buzz-persona", path = "../../crates/buzz-persona" }` | **Yes, one call** — `desktop/src-tauri/src/migration.rs:1123` uses `buzz_persona_pkg::persona::split_frontmatter` |
| workspace membership | `Cargo.toml:23` — `"crates/buzz-persona"` | — |

#### `buzz-cli` consumption pattern

Two subcommands, dispatched at `crates/buzz-cli/src/lib.rs:1739-1740`:

- `buzz pack validate <path>` → `commands::pack::cmd_validate`
  (`crates/buzz-cli/src/commands/pack.rs:15-46`): checks the path exists and is a
  directory, calls `buzz_persona::validate::validate_pack`
  (`crates/buzz-cli/src/commands/pack.rs:24`), prints each `ValidationDiagnostic` to
  stderr by matching on the enum variants (`:26-35`), then maps `has_errors()` to
  `CliError::Usage` (`:37-38`). It does **not** use
  `ValidationReport::exit_code()` or the `Display` impl.
- `buzz pack inspect <path>` → `commands::pack::cmd_inspect`
  (`crates/buzz-cli/src/commands/pack.rs:52-152`): calls
  `buzz_persona::resolve::resolve_pack` (`:62`) and pretty-prints the fully resolved
  effective config per persona — recombining `llm_provider` + `model` for display
  (`:78-87`), triggers (`:101-114`), MCP server count (`:120-122`), skills (`:124-126`),
  a truncated system-prompt preview (`:132-145`), and `runtime_env_vars` as `K=V` pairs
  (`:147-155`).

#### desktop consumption pattern

`desktop/src-tauri/src/migration.rs:1121-1132` (`rewrite_legacy_persona_md_runtime`) uses
only the frontmatter splitter, then re-parses the YAML itself with `serde_yaml` to rewrite
`runtime: "sprout-agent"` → `"buzz-agent"` and re-emits `---\n{frontmatter}---\n{body}`.
It deliberately bypasses `parse_persona_md` — it must round-trip unknown/legacy keys that
`deny_unknown_fields` would reject.

#### How buzz-acp is *expected* to consume it

The dependency exists but no call sites do. The intended contract is documented rather
than exercised:

- `crates/buzz-persona/src/resolve.rs:1-14` — "`ResolvedPersona` maps 1:1 to ACP's needs";
  field-level comments name the ACP targets: `system_prompt` → `Config.system_prompt`
  (`:31`), `model` → `Config.model` (`:36`), `subscribe` → `Config.subscribe_mode +
  channels_override` (`:47`), `triggers` → "mapped to ACP filter rules at startup" (`:49`).
- `runtime_env_vars` (`crates/buzz-persona/src/resolve.rs:64`) is the projection buzz-acp
  is expected to inject at spawn. The matching consumer field exists on the ACP side:
  `crates/buzz-acp/src/config.rs:533-535` — `pub persona_env_vars: Vec<(String, String)>`
  with the comment "Populated from persona pack resolution. Empty when no pack is
  configured." It is passed to the spawn path at `crates/buzz-acp/src/lib.rs:3733`
  (`extra_env: config.persona_env_vars.clone()`).
- Operator-precedence filtering (level 1) is explicitly the consumer's job, not this
  crate's: `crates/buzz-persona/src/resolve.rs:359-364` — "ACP is responsible for
  filtering based on operator precedence (level 1)".
- `desktop/src-tauri/src/managed_agents/types.rs:52` references
  "ACP's `resolve_persona_by_name()`", indicating the desktop/ACP contract expects that
  entry point.

So: the *data contract* between this crate and buzz-acp is designed and the receiving
field exists on the ACP `Config`, but the wire-up that would call
`resolve_pack`/`resolve_persona_by_name` from buzz-acp is not present in the current tree.

#### Import-filter convention (consumer-side, mirrored in tests)

`crates/buzz-persona/tests/e2e_env_flow.rs:15-32` defines
`DERIVED_PROVIDER_MODEL_ENV_KEYS = ["GOOSE_MODEL", "GOOSE_PROVIDER", "BUZZ_AGENT_MODEL",
"BUZZ_AGENT_PROVIDER"]` and a `filter_derived` helper described as mirroring "desktop
import_persona_pack logic" (`crates/buzz-persona/tests/e2e_env_flow.rs:200`). The
test asserts that on import the derived provider/model keys are stripped while
`GOOSE_TEMPERATURE` survives (`:206-228`). This is a duplicated copy of consumer logic
living in this crate's test suite — the real implementation is outside this crate.

---

### Standards / external specs referenced

| Spec | Where referenced | Enforced in code? |
|---|---|---|
| Open Plugin Spec (`open-plugin-spec.org`) | `PERSONA_PACK_SPEC.md:6-7`, §2; `"$schema"` accepted in `KNOWN_MANIFEST_KEYS` (`crates/buzz-persona/src/validate.rs:101`) | Partially — OPS field names are accepted and unknown fields tolerated (`crates/buzz-persona/src/manifest.rs:123-130`), but no schema fetch or validation |
| Semver (`engines.buzz`, `version`) | `crates/buzz-persona/src/manifest.rs:33-40` doc; `version: String` (`:82`) | No — plain strings, no semver crate |
| ACP (Agent Client Protocol) | `crates/buzz-persona/src/resolve.rs:1-14` and field comments | No protocol code here; shape-only alignment |
| MCP (Model Context Protocol) | `McpServerConfig` (`crates/buzz-persona/src/persona.rs:70`), `.mcp.json` `mcpServers` key (`crates/buzz-persona/src/resolve.rs:285`) | Config shape only; no MCP client |
