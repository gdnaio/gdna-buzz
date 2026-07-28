## Module: buzz-persona (`crates/buzz-persona`)

### Aspect: Data Model

Scope note: this crate ships **no persona data assets**. There are no `.persona.md`,
`.toml`, `.yaml`, or `.json` files in the crate, and no `include_str!` /
`include_dir!` / `include_bytes!` macros anywhere (verified by repo-wide grep over
`crates/buzz-persona`). The only non-Rust file is `crates/buzz-persona/PERSONA_PACK_SPEC.md`
(prose spec, not compiled in). The crate is a **parser/loader/resolver library over
on-disk pack directories** — personas are data supplied by the caller, not baked in.

---

### 1. Pack-on-disk schema (what the loader reads)

`load_pack()` expects this layout, documented at `crates/buzz-persona/src/pack.rs:1-19`
and enforced by the code paths cited below:

| Path (pack-relative) | Required | Read at | Notes |
|---|---|---|---|
| `.plugin/plugin.json` | yes | `crates/buzz-persona/src/pack.rs:132-137` | Missing → `PackError::ManifestNotFound` |
| paths listed in manifest `personas[]` | yes (each must exist) | `crates/buzz-persona/src/pack.rs:155-164` | Missing → `PackError::PersonaNotFound` |
| `pack_instructions` path, else `instructions.md` | no | `crates/buzz-persona/src/pack.rs:167-188` | Explicit path missing → hard error; implicit fallback missing → `None` |
| `mcp_config` path, else `.mcp.json` | no | `crates/buzz-persona/src/pack.rs:191-220` | Explicit path missing → hard error; parsed as free-form `serde_json::Value` |
| `skills/` directory | no | `crates/buzz-persona/src/pack.rs:223-230` | Only presence is recorded (`skills_dir`); contents are not parsed at load |
| `hooks_config` path | parsed in manifest only | `crates/buzz-persona/src/manifest.rs:116` | Deliberately dropped by the loader — see `crates/buzz-persona/src/pack.rs:111-112` |

---

### 2. `PersonaConfig` — typed `.persona.md` (frontmatter + body)

`crates/buzz-persona/src/persona.rs:99-170`. Derives `Debug, Clone, Serialize, Deserialize`
with `#[serde(rename_all = "snake_case")]` (`crates/buzz-persona/src/persona.rs:99-100`).

| Field | Type | Required | serde attr | Line |
|---|---|---|---|---|
| `name` | `String` | yes | — | `persona.rs:103` |
| `display_name` | `String` | yes | — | `persona.rs:106` |
| `avatar` | `Option<String>` | no | `skip_serializing_if = "Option::is_none"` | `persona.rs:110` |
| `description` | `String` | yes | — | `persona.rs:113` |
| `version` | `Option<String>` | no | skip-if-none | `persona.rs:116` |
| `author` | `Option<String>` | no | skip-if-none | `persona.rs:119` |
| `skills` | `Vec<String>` | no | `#[serde(default)]` | `persona.rs:123` |
| `mcp_servers` | `Vec<McpServerConfig>` | no | `#[serde(default)]` | `persona.rs:127` |
| `subscribe` | `Option<Vec<String>>` | no | `default`, skip-if-none | `persona.rs:135` |
| `triggers` | `Option<RespondTo>` | no | skip-if-none | `persona.rs:139` |
| `model` | `Option<String>` | no | skip-if-none; `"provider:model-id"` | `persona.rs:143` |
| `runtime` | `Option<String>` | no | skip-if-none; e.g. `'goose'`, `'claude'` | `persona.rs:148` |
| `temperature` | `Option<f64>` | no | skip-if-none | `persona.rs:151` |
| `max_context_tokens` | `Option<u64>` | no | skip-if-none | `persona.rs:154` |
| `thread_replies` | `Option<bool>` | no | skip-if-none | `persona.rs:158` |
| `broadcast_replies` | `Option<bool>` | no | skip-if-none | `persona.rs:162` |
| `hooks` | `Option<Hooks>` | no | skip-if-none | `persona.rs:165` |
| `prompt` | `String` | — (populated from body, never from YAML) | `#[serde(default)]` | `persona.rs:169` |

Tri-state semantics for `subscribe` are documented inline at
`crates/buzz-persona/src/persona.rs:129-134`: `None` = omitted/null → fall through to
pack default; `Some(vec![])` = intentional "subscribe to nothing"; `Some([...])` = explicit.

#### Private deserialization shadow: `Frontmatter`

`crates/buzz-persona/src/persona.rs:174-196` — a private struct with
`#[serde(rename_all = "snake_case", deny_unknown_fields)]`. All identity fields are
`Option<...>` so the parser can emit `PersonaError::MissingField` instead of a serde
path error. `triggers` carries `#[serde(alias = "respond_to")]`
(`crates/buzz-persona/src/persona.rs:186`) — the legacy key alias. `prompt` is **absent**
from this struct, so a persona file cannot inject a prompt via frontmatter; the body is
the only source (`crates/buzz-persona/src/persona.rs:257`).

#### Nested persona types

`RespondTo` — `crates/buzz-persona/src/persona.rs:53-65`:

| Field | Type | serde | Line |
|---|---|---|---|
| `mentions` | `Option<bool>` | skip-if-none | `persona.rs:56` |
| `keywords` | `Vec<String>` | `#[serde(default)]` | `persona.rs:60` |
| `all_messages` | `Option<bool>` | skip-if-none | `persona.rs:64` |

`McpServerConfig` — `crates/buzz-persona/src/persona.rs:70-79`:

| Field | Type | serde | Line |
|---|---|---|---|
| `name` | `String` | required | `persona.rs:71` |
| `command` | `String` | required | `persona.rs:72` |
| `args` | `Vec<String>` | `default` | `persona.rs:75` |
| `env` | `HashMap<String, String>` | `default` | `persona.rs:78` |

`Hooks` — `crates/buzz-persona/src/persona.rs:84-93`, with
`deny_unknown_fields` (`crates/buzz-persona/src/persona.rs:83`):
`on_start: Option<String>` (`:86`), `on_stop: Option<String>` (`:89`),
`on_message: Option<String>` (`:92`). Doc comment states paths are pack-relative
(`crates/buzz-persona/src/persona.rs:81`).

#### Size constants

`MAX_FRONTMATTER_BYTES = 1_048_576` (1 MiB) — `crates/buzz-persona/src/persona.rs:21`.
`MAX_BODY_BYTES = 262_144` (256 KiB) — `crates/buzz-persona/src/persona.rs:24`.

---

### 3. `PackManifest` — typed `plugin.json`

`crates/buzz-persona/src/manifest.rs:79-121`, `rename_all = "snake_case"`
(`crates/buzz-persona/src/manifest.rs:78`).

| Field | Type | Required | Line |
|---|---|---|---|
| `id` | `String` | yes | `manifest.rs:80` |
| `name` | `String` | yes | `manifest.rs:81` |
| `version` | `String` | yes | `manifest.rs:82` |
| `description` | `Option<String>` | no | `manifest.rs:85` |
| `author` | `Option<String>` | no | `manifest.rs:88` |
| `license` | `Option<String>` | no | `manifest.rs:91` |
| `homepage` | `Option<String>` | no | `manifest.rs:94` |
| `keywords` | `Vec<String>` | no (`default`) | `manifest.rs:97` |
| `engines` | `Option<Engines>` | no | `manifest.rs:100` |
| `personas` | `Vec<String>` | no (`default` → empty) | `manifest.rs:104` |
| `pack_instructions` | `Option<String>` | no | `manifest.rs:108` |
| `mcp_config` | `Option<String>` | no | `manifest.rs:112` |
| `hooks_config` | `Option<String>` | no | `manifest.rs:116` |
| `defaults` | `Option<BehavioralDefaults>` | no | `manifest.rs:120` |

`Engines` — `crates/buzz-persona/src/manifest.rs:37-41`: single field
`buzz: Option<String>` with `alias = "buzz"` (`crates/buzz-persona/src/manifest.rs:39`).
No semver comparison logic exists in this crate.

`BehavioralDefaults` — `crates/buzz-persona/src/manifest.rs:49-70`, described as
"same shape as the persona behavioral config fields"
(`crates/buzz-persona/src/manifest.rs:44-46`):
`model` (`:51`), `temperature` (`:54`), `max_context_tokens` (`:57`),
`subscribe` (`:60`), `triggers: Option<RespondTo>` with
`alias = "respond_to"` (`:62-63`), `thread_replies` (`:66`),
`broadcast_replies` (`:69`).

`RawManifest` — private permissive mirror at `crates/buzz-persona/src/manifest.rs:132-149`.
Explicitly **no** `deny_unknown_fields`; the rationale (OPS superset may carry foreign
fields such as `ops_category`, `marketplace_tags`) is documented at
`crates/buzz-persona/src/manifest.rs:123-130`.

---

### 4. Intermediate model: `LoadedPack` / `LoadedPersona`

`LoadedPack` — `crates/buzz-persona/src/pack.rs:64-73`:

| Field | Type | Line |
|---|---|---|
| `manifest` | `PackManifestData` | `pack.rs:65` |
| `personas` | `Vec<LoadedPersona>` | `pack.rs:66` |
| `pack_instructions` | `Option<String>` (file contents) | `pack.rs:68` |
| `shared_mcp_config` | `Option<serde_json::Value>` (raw `.mcp.json`) | `pack.rs:70` |
| `skills_dir` | `Option<PathBuf>` | `pack.rs:72` |

`LoadedPersona` — `crates/buzz-persona/src/pack.rs:77-98`. This is the post-merge shape:
behavioral fields have already had pack defaults applied, so `subscribe`,
`thread_replies`, `broadcast_replies` are no longer `Option`.

| Field | Type | Line | Origin |
|---|---|---|---|
| `source_path` | `PathBuf` | `pack.rs:78` | absolute path of the `.persona.md` |
| `name` / `display_name` / `description` | `String` | `pack.rs:79-81` | frontmatter, verbatim |
| `avatar` | `Option<String>` | `pack.rs:82` | frontmatter |
| `model` | `Option<String>` | `pack.rs:83` | merged (still `provider:id` form) |
| `runtime` | `Option<String>` | `pack.rs:85` | frontmatter only, **not merged** (`pack.rs:428`) |
| `temperature` | `Option<f64>` | `pack.rs:86` | merged |
| `max_context_tokens` | `Option<u64>` | `pack.rs:87` | merged |
| `subscribe` | `Vec<String>` | `pack.rs:88` | merged, `unwrap_or_default()` at `pack.rs:432` |
| `triggers` | `Option<TriggersData>` | `pack.rs:89` | merged |
| `thread_replies` | `bool` | `pack.rs:90` | merged (default true) |
| `broadcast_replies` | `bool` | `pack.rs:91` | merged (default false) |
| `skills` | `Vec<String>` | `pack.rs:92` | frontmatter, raw paths |
| `mcp_servers` | `Vec<serde_json::Value>` | `pack.rs:94` | typed configs re-serialized to JSON (`pack.rs:415-419`) |
| `hooks` | `Option<HooksData>` | `pack.rs:95` | frontmatter |
| `prompt` | `String` | `pack.rs:97` | markdown body |

`PackManifestData` — `crates/buzz-persona/src/pack.rs:102-115`. A reduced manifest
projection: `id`, `name`, `version`, `description`, `personas`, `pack_instructions`,
`mcp_config`, and `defaults` re-encoded as `Option<serde_json::Value>` (`pack.rs:114`)
so the JSON-based merge layer can consume it. `hooks_config` is intentionally omitted
with an inline comment at `crates/buzz-persona/src/pack.rs:111-112`.

---

### 5. Merge-layer model (`merge.rs`)

Plain (non-serde) data structs used to carry merged values:

| Type | Fields | Line |
|---|---|---|
| `TriggersData` | `mentions: bool`, `keywords: Vec<String>`, `all_messages: bool` | `merge.rs:11-15` |
| `HooksData` | `on_start`/`on_stop`/`on_message`: `Option<String>` | `merge.rs:18-22` |
| `ResolvedConfig` | `model: Option<String>`, `temperature: Option<f64>`, `max_context_tokens: Option<u64>`, `subscribe: Option<Vec<String>>`, `triggers: Option<TriggersData>`, `thread_replies: bool`, `broadcast_replies: bool` | `merge.rs:25-36` |

`ResolvedConfig::subscribe` tri-state is documented at `crates/buzz-persona/src/merge.rs:29-31`.
Built-in defaults as constants: `DEFAULT_THREAD_REPLIES = true`
(`crates/buzz-persona/src/merge.rs:38`), `DEFAULT_BROADCAST_REPLIES = false`
(`crates/buzz-persona/src/merge.rs:39`). `TriggersData` sub-field built-ins are inline
literals in `parse_triggers`: `mentions` → `true` (`merge.rs:181`), `keywords` → empty
(`merge.rs:191`), `all_messages` → `false` (`merge.rs:196`).

---

### 6. Output model: `ResolvedPack` / `ResolvedPersona` (the ACP-facing contract)

Module doc states the shape is "designed backward from ACP's `Config`"
(`crates/buzz-persona/src/resolve.rs:1-14`).

`ResolvedPersona` — `crates/buzz-persona/src/resolve.rs:23-65`:

| Field | Type | Line | Derivation |
|---|---|---|---|
| `name`, `display_name`, `description` | `String` | `resolve.rs:25-27` | copied from `LoadedPersona` |
| `avatar` | `Option<String>` | `resolve.rs:28` | copied |
| `version` | `String` | `resolve.rs:29` | always the **pack** version (`resolve.rs:225-231`) |
| `system_prompt` | `String` | `resolve.rs:32` | persona markdown body only (`resolve.rs:200`) |
| `pack_instructions` | `Option<String>` | `resolve.rs:34` | trimmed; empty → `None` (`resolve.rs:201-204`) |
| `model` | `Option<String>` | `resolve.rs:37` | model id **after** colon split |
| `llm_provider` | `Option<String>` | `resolve.rs:40` | colon prefix of `model` |
| `runtime` | `Option<String>` | `resolve.rs:43` | persona `runtime` passthrough |
| `temperature` | `Option<f64>` | `resolve.rs:44` | merged value |
| `max_context_tokens` | `Option<u64>` | `resolve.rs:45` | merged value |
| `subscribe` | `Vec<String>` | `resolve.rs:48` | merged list |
| `triggers` | `ResolvedTriggers` (non-optional) | `resolve.rs:50` | `None` collapses to built-ins (`resolve.rs:255-268`) |
| `thread_replies`, `broadcast_replies` | `bool` | `resolve.rs:51-52` | merged |
| `mcp_servers` | `Vec<ResolvedMcpServer>` | `resolve.rs:55` | pack `.mcp.json` + persona list, name-keyed |
| `hooks` | `Option<ResolvedHooks>` | `resolve.rs:58` | "parsed, not executed — reserved for future use, not yet wired" (`resolve.rs:57`) |
| `skills` | `Vec<String>` | `resolve.rs:61` | "reserved for future use, not yet wired" (`resolve.rs:60`); value is `lp.skills.clone()` (`resolve.rs:249`), i.e. raw declared paths, not bare names |
| `runtime_env_vars` | `Vec<(String, String)>` | `resolve.rs:64` | projection from model/temperature/context |

`ResolvedMcpServer` — `crates/buzz-persona/src/resolve.rs:69-74`: `name`, `command`,
`args: Vec<String>`, `env: Vec<(String, String)>` (env as ordered pairs, not a map).
Doc comment records that env values are literals with no `${VAR}` interpolation
(`crates/buzz-persona/src/resolve.rs:68`).

`ResolvedHooks` — `crates/buzz-persona/src/resolve.rs:78-82`: same three optional paths.

`ResolvedTriggers` — `crates/buzz-persona/src/resolve.rs:86-90`: `mentions: bool`,
`keywords: Vec<String>`, `all_messages: bool`.

`ResolvedPack` — `crates/buzz-persona/src/resolve.rs:94-100`: `id`, `name`, `version`,
`description: String` (empty string when the manifest omits it — `resolve.rs:180`),
`personas: Vec<ResolvedPersona>`.

---

### 7. Validation report model

`ValidationDiagnostic` — `crates/buzz-persona/src/validate.rs:19-22`: enum with
`Error(String)` and `Warning(String)`; `Display` renders `ERROR: ` / `WARN:  `
prefixes (`crates/buzz-persona/src/validate.rs:24-33`).
`ValidationReport` — `crates/buzz-persona/src/validate.rs:35-37`: single field
`diagnostics: Vec<ValidationDiagnostic>`, `#[derive(Debug, Default)]`.

---

### 8. Data-flow relationship: persona → agent

The crate performs a four-stage transformation; each stage is a distinct type:

```
plugin.json  ──parse_manifest──▶ PackManifest ──▶ PackManifestData
.persona.md  ──parse_persona_md─▶ PersonaConfig
                                      │  (serialized to serde_json::Value, pack.rs:406-409)
                                      ▼
                     resolve_persona_config ──▶ ResolvedConfig ──▶ LoadedPersona
                                                                       │
                                                       resolve_one_persona (resolve.rs:194)
                                                                       ▼
                                                                ResolvedPersona
                                                          (system_prompt + env vars + MCP)
```

Notable mechanic: the merge layer is JSON-based, so `PersonaConfig` is round-tripped
through `serde_json::to_value` before merging (`crates/buzz-persona/src/pack.rs:406-409`).
This is why `PersonaConfig` carries `skip_serializing_if = "Option::is_none"` on every
optional behavioral field — omitted fields must not appear as `null` keys in the merge
input.

Relationship to agents: a persona does **not** reference an agent identity, keypair, or
Nostr pubkey anywhere in this crate. The only agent-runtime coupling is
(a) `runtime: Option<String>` selecting an env-var naming scheme
(`crates/buzz-persona/src/resolve.rs:365-397`), and (b) `runtime_env_vars`, the
key/value pairs a harness is expected to inject into the agent subprocess.
