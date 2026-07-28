## Module: buzz-persona (`crates/buzz-persona`)

### Aspect: Configuration

### 1. Environment variables read by this crate

**None.** There is no `std::env::var`, `env::var_os`, `envy`, or `dotenv` usage anywhere in
`crates/buzz-persona/src/` or `crates/buzz-persona/tests/` (verified by grep — the only
`std::env` occurrences in the crate are prose inside
`crates/buzz-persona/PERSONA_PACK_SPEC.md`).

This is a stated design property, not an accident:

- `crates/buzz-persona/src/resolve.rs:11` — "**Pure**: no env access, no network, no side effects."
- `crates/buzz-persona/src/resolve.rs:359-364` — "Pure function — does NOT read the current
  process env. ACP is responsible for filtering based on operator precedence (level 1): if
  the operator already set an env var, ACP skips injection so the operator's value wins."

Consequence: this crate cannot be configured through the environment. All behavior is a
function of the pack directory contents plus the arguments passed in.

---

### 2. Environment variables *emitted* by this crate

The crate produces env vars as **data** for a consumer to inject into an agent subprocess.
`runtime_env_vars` — `crates/buzz-persona/src/resolve.rs:365-397`.

| Emitted key | Source field | Condition | Line |
|---|---|---|---|
| `BUZZ_AGENT_MODEL` | model id (post colon-split) | `runtime == "buzz-agent"` | `resolve.rs:375` |
| `BUZZ_AGENT_PROVIDER` | model provider prefix | `runtime == "buzz-agent"` **and** model contains a colon | `resolve.rs:376-378` |
| `GOOSE_PROVIDER` | model provider prefix | any other runtime (incl. `None`, `"goose"`, `"claude"`) **and** model contains a colon | `resolve.rs:381-383` |
| `GOOSE_MODEL` | model id (post colon-split) | any other runtime | `resolve.rs:384` |
| `GOOSE_TEMPERATURE` | `temperature.to_string()` | `temperature` is `Some` — **regardless of runtime** | `resolve.rs:391-393` |
| `GOOSE_CONTEXT_LIMIT` | `max_context_tokens.to_string()` | `max_context_tokens` is `Some` — **regardless of runtime** | `resolve.rs:395-397` |

The runtime-independence of the last two is deliberate and commented: "temperature and
context_limit stay as GOOSE_* (only goose reads them)"
(`crates/buzz-persona/src/resolve.rs:389`).

Consumer-side handling of these keys (outside this crate): the ACP `Config` carries them as
`persona_env_vars: Vec<(String, String)>` (`crates/buzz-acp/src/config.rs:533-535`) and
passes them to spawn as `extra_env` (`crates/buzz-acp/src/lib.rs:3733`).

A related consumer convention is mirrored in this crate's tests: the desktop import path
strips the four derived provider/model keys while preserving the "knob" keys —
`DERIVED_PROVIDER_MODEL_ENV_KEYS` at `crates/buzz-persona/tests/e2e_env_flow.rs:15-20`,
asserted at `:206-228`.

---

### 3. Cargo features

**None declared.** `crates/buzz-persona/Cargo.toml:1-18` has no `[features]` section, and a
grep for `cfg(feature` across the crate returns zero matches. There is no `default`
feature, no optional dependency, and therefore nothing gated by features.

The only conditional compilation in the crate is platform and test based:

| Directive | Location | Gates |
|---|---|---|
| `#[cfg(windows)]` | `crates/buzz-persona/src/pack.rs:334` | Windows drive-letter rejection in `safe_resolve` |
| `#[cfg(unix)]` | `crates/buzz-persona/src/pack.rs:589` | the symlink-escape test |
| `#[cfg(test)]` | `manifest.rs:195`, `merge.rs:200`, `pack.rs:447`, `persona.rs:331`, `resolve.rs:400`, `validate.rs:436` | per-module test modules |

Dependency feature selection is minimal: only `serde` enables a feature (`derive`) —
`crates/buzz-persona/Cargo.toml:10`. `serde_json`, `serde_yaml`, `thiserror`, and
`tempfile` use defaults.

---

### 4. Package-level configuration

`crates/buzz-persona/Cargo.toml:1-8`:

| Key | Value | Note |
|---|---|---|
| `name` | `buzz-persona` | |
| `version` | `0.1.0` | hard-coded, **not** `version.workspace = true` — unlike e.g. `crates/buzz-acp/Cargo.toml:3` |
| `edition` | `2021` | hard-coded; no `rust-version` key |
| `description` | `Parser and loader for Buzz persona pack files (.persona.md)` | |
| `license` | `Apache-2.0` | hard-coded |
| `repository` | `https://github.com/block/sprout` | stale — the OSS repo is `block/buzz` per `AGENTS.md` |

Workspace membership: `Cargo.toml:23` (`"crates/buzz-persona"`).

---

### 5. Compile-time constants (the crate's tunables)

Because there are no env vars or features, all tunable behavior is expressed as constants.
Changing any of them requires a recompile.

| Constant | Value | Visibility | Location | Governs |
|---|---|---|---|---|
| `MAX_FRONTMATTER_BYTES` | `1_048_576` (1 MiB) | `pub` | `crates/buzz-persona/src/persona.rs:21` | YAML frontmatter cap; also a term in the pack-level file cap |
| `MAX_BODY_BYTES` | `262_144` (256 KiB) | `pub` | `crates/buzz-persona/src/persona.rs:24` | persona prompt cap |
| `DEFAULT_THREAD_REPLIES` | `true` | private | `crates/buzz-persona/src/merge.rs:38` | built-in default (precedence level 5) |
| `DEFAULT_BROADCAST_REPLIES` | `false` | private | `crates/buzz-persona/src/merge.rs:39` | built-in default (level 5) |
| `MAX_NAME_LEN` | `64` | fn-local | `crates/buzz-persona/src/validate.rs:168` | persona name length (validator path) |
| inline `64` | `64` | literal | `crates/buzz-persona/src/resolve.rs:146` | persona name length (resolve path) — duplicated, not shared with `MAX_NAME_LEN` |
| inline trigger built-ins | `mentions=true`, `keywords=[]`, `all_messages=false` | literals | `crates/buzz-persona/src/merge.rs:181`, `:191`, `:196` and again `crates/buzz-persona/src/resolve.rs:264-266` | built-in trigger defaults — declared twice |
| `+ 100` / `+ 200` slack in size guards | — | literals | `crates/buzz-persona/src/persona.rs:266` (`+ 100`), `crates/buzz-persona/src/pack.rs:154` and `:169` (`+ 200`) | delimiter/whitespace headroom above the two caps; the two paths use different slack values |
| `KNOWN_MANIFEST_KEYS` | 16 entries | private | `crates/buzz-persona/src/validate.rs:99-118` | which `plugin.json` top-level keys avoid an "unknown key" warning |
| `KNOWN_BEHAVIORAL_KEYS` | 8 entries (incl. legacy `respond_to`) | private | `crates/buzz-persona/src/validate.rs:121-130` | which `defaults` keys avoid a warning |
| `KNOWN_RESPOND_TO_KEYS` | `mentions`, `keywords`, `all_messages` | private | `crates/buzz-persona/src/validate.rs:133` | which `defaults.triggers` sub-keys avoid a warning |

`KNOWN_MANIFEST_KEYS` full list (`crates/buzz-persona/src/validate.rs:99-118`):
`$schema`, `id`, `name`, `version`, `description`, `author`, `license`, `homepage`,
`repository`, `keywords`, `engines`, `personas`, `defaults`, `pack_instructions`,
`hooks_config`, `mcp_config`.

Drift note: `KNOWN_MANIFEST_KEYS` includes `repository`
(`crates/buzz-persona/src/validate.rs:108`), but `PackManifest` has no `repository` field
(`crates/buzz-persona/src/manifest.rs:79-121`) — the key is warning-exempt yet discarded at
parse time.

---

### 6. Pack-file configuration surface (what a pack author can configure)

This is the crate's real "configuration" input. Paths are pack-relative and resolved
against the canonicalized pack root.

| Config location | Purpose | Discovery rule |
|---|---|---|
| `.plugin/plugin.json` | pack identity + persona list + pack-wide `defaults` | fixed path, required (`crates/buzz-persona/src/pack.rs:132-135`) |
| `<manifest>.personas[]` | which `.persona.md` files to load | explicit list only — no directory globbing; each must exist (`crates/buzz-persona/src/pack.rs:155-164`) |
| `<manifest>.pack_instructions` → else `instructions.md` | pack-level instruction text | explicit path preferred; implicit `instructions.md` at pack root as fallback (`crates/buzz-persona/src/pack.rs:167-188`) |
| `<manifest>.mcp_config` → else `.mcp.json` | shared MCP server map under key `mcpServers` | explicit path preferred; implicit `.mcp.json` at pack root as fallback (`crates/buzz-persona/src/pack.rs:191-220`); shape read at `crates/buzz-persona/src/resolve.rs:285` |
| `skills/<name>/SKILL.md` | skill definitions | presence of `skills/` recorded (`crates/buzz-persona/src/pack.rs:223-230`); directory entries enumerated by `resolve_skills` (`:275-293`) and by the validator (`crates/buzz-persona/src/validate.rs:375-386`) |
| `<manifest>.hooks_config` | pack-level lifecycle hooks | **parsed into `PackManifest` but never loaded** — dropped with an explicit comment at `crates/buzz-persona/src/pack.rs:111-112` |
| `<manifest>.defaults` | behavioral defaults (precedence level 4) | `crates/buzz-persona/src/manifest.rs:49-70`; re-encoded to JSON at `crates/buzz-persona/src/pack.rs:143-147` |
| `.persona.md` frontmatter | per-persona identity + behavioral config (level 3) | `crates/buzz-persona/src/persona.rs:176-196` |

Not honored / not searched:

- No user-level or global persona path. The crate never reads `~/.buzz/packs/`,
  `$XDG_CONFIG_HOME`, or any well-known location — the pack directory is always an
  explicit `&Path` argument (`load_pack`, `resolve_pack`, `resolve_persona_by_name`,
  `validate_pack`). The `~/.buzz/packs/<pack-id>/` convention appears only as prose in
  `crates/buzz-persona/PERSONA_PACK_SPEC.md:993-996` and is the consumer's responsibility.
- No config-file layering, no `.env`, no TOML/INI config for the crate itself.
- No persona directory auto-discovery: a `.persona.md` not listed in `personas[]` is
  invisible to `load_pack`.
- `engines.buzz` is parsed (`crates/buzz-persona/src/manifest.rs:39-40`) but never
  evaluated — no minimum-version gate exists in this crate.

---

### 7. The five-level precedence model, and which levels this crate implements

Documented at `crates/buzz-persona/src/merge.rs:1-9` and
`crates/buzz-persona/PERSONA_PACK_SPEC.md:625-637`.

| Level | Source | Implemented here? | Evidence |
|---|---|---|---|
| 1 | Operator env vars already set in the parent process | **No** — explicitly the consumer's job | `crates/buzz-persona/src/resolve.rs:359-364` |
| 2 | Desktop UI per-agent overrides | **No** — runtime concern | `crates/buzz-persona/src/merge.rs:9` |
| 3 | Per-persona frontmatter | **Yes** | persona arm of `merge_behavioral_config` (`crates/buzz-persona/src/merge.rs:68-73`) |
| 4 | Pack `defaults` in `plugin.json` | **Yes** | defaults iteration (`crates/buzz-persona/src/merge.rs:66-74`) |
| 5 | Built-in hardcoded defaults | **Yes** | constants `crates/buzz-persona/src/merge.rs:38-39`; trigger built-ins `:180-198`; fallback `:159-168` |

Fields with **no** level-5 default (they stay `None` and the consumer decides):
`model`, `temperature`, `max_context_tokens` (`crates/buzz-persona/src/merge.rs:95-97`),
and `subscribe` when neither side sets it (`:114-121`) — though `LoadedPersona.subscribe`
collapses `None` to an empty `Vec` at `crates/buzz-persona/src/pack.rs:432`, so the
distinction is lost past that point.
