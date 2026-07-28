## Module: buzz-persona (`crates/buzz-persona`)

### Aspect: Security

### 1. `unsafe` usage

**None.** A grep for `unsafe` across the whole of `crates/buzz-persona` (all `.rs`,
`Cargo.toml`, and `PERSONA_PACK_SPEC.md`) returns zero matches. There is no
`#![allow(unsafe_code)]`, no FFI, no raw pointer manipulation. The crate is 100% safe Rust.

Also absent from non-test code: `unwrap()`, `expect()`, `panic!`, `unreachable!`,
`todo!` (all matches are inside `#[cfg(test)]` modules or `tests/`), so a malformed pack
cannot panic the caller through this crate's parsing paths.

---

### 2. Prompt-injection surface

This is the crate's largest security-relevant characteristic: **persona content flows into
agent prompts entirely unvalidated and unsanitized.**

| Finding | Evidence |
|---|---|
| The markdown body of a `.persona.md` becomes the agent system prompt with **no** content inspection — no length-bounded sanitization beyond the raw byte cap, no escaping, no scan for instruction-override strings | `let system_prompt = lp.prompt.clone();` — `crates/buzz-persona/src/resolve.rs:200`; body captured verbatim at `crates/buzz-persona/src/persona.rs:257` |
| The only guard on prompt content is a size cap: 256 KiB body, 1 MiB frontmatter | `crates/buzz-persona/src/persona.rs:21`, `:24`, enforced `:213-218`; second cap at the file level `crates/buzz-persona/src/pack.rs:153-154` |
| `pack_instructions` (contents of `instructions.md`) is also passed through verbatim, trimmed only | `crates/buzz-persona/src/resolve.rs:201-204` |
| `display_name` and `description` are free-form strings with only a non-empty check — no charset restriction, no length cap. They reach the agent/UI as-is. | `crates/buzz-persona/src/persona.rs:233-239`; test fixtures include emoji values such as `"Pip 🐝"` (`crates/buzz-persona/tests/integration.rs:64`) |
| `triggers.keywords` are free-form strings, unbounded in count and length | `crates/buzz-persona/src/persona.rs:60`; merged without limit at `crates/buzz-persona/src/merge.rs:184-193` |
| Skills are described in the crate's own spec as prompt-injection vectors, and this crate performs no skill content validation at all — `SKILL.md` bodies are never read except to compare the `name:` field | spec statement at `crates/buzz-persona/PERSONA_PACK_SPEC.md:1041-1043` ("Skills are markdown injected into agent context; malicious content can attempt prompt injection"); code reads only frontmatter `name` at `crates/buzz-persona/src/validate.rs:410-431` |

Mitigating structural choices that *are* present:

- `prompt` cannot be injected through YAML — the private `Frontmatter` struct has no
  `prompt` field (`crates/buzz-persona/src/persona.rs:176-196`), so a persona file cannot
  smuggle prompt content through a frontmatter key while presenting an innocuous body.
- The persona body and pack instructions are kept as **separate fields**, not concatenated
  (`crates/buzz-persona/src/resolve.rs:32-34`; asserted in
  `crates/buzz-persona/tests/integration.rs:394-407`), leaving the consumer able to frame
  or label each layer. `crates/buzz-acp/src/pool.rs:1122-1127` shows the consumer does
  frame them (`[Base]` / `[System]` markers) so the boundary stays recoverable downstream.
- Unknown frontmatter keys are hard-rejected (`deny_unknown_fields` at
  `crates/buzz-persona/src/persona.rs:175`), so a pack cannot introduce new
  behavior-bearing keys that a future version might honor.

Assessment: prompt content is treated as trusted-by-installation. The trust boundary is
"who may place a pack directory on disk", and this crate does not attempt to reduce it.
`crates/buzz-persona/PERSONA_PACK_SPEC.md:1035-1043` states this explicitly as an
operational requirement ("Only install packs from trusted sources").

---

### 3. Path traversal — implemented defenses

`safe_resolve` (`crates/buzz-persona/src/pack.rs:323-365`) is a three-layer check, and its
own doc calls it "Defense-in-depth" (`:317-322`):

| Layer | Check | Line |
|---|---|---|
| 0 | Reject absolute paths — Unix `/` prefix, and on Windows a drive-letter `X:` prefix | `crates/buzz-persona/src/pack.rs:330-337` |
| 1 | Reject any `Component::ParentDir` (`..`) **before** canonicalization | `crates/buzz-persona/src/pack.rs:339-345` |
| 2 | `canonicalize()` the joined path (resolves symlinks) | `crates/buzz-persona/src/pack.rs:357-360` |
| 3 | Require `canonical.starts_with(pack_root)` | `crates/buzz-persona/src/pack.rs:361-364` |

Applied to: `personas[]` entries (`crates/buzz-persona/src/pack.rs:156`),
`pack_instructions` (`:171`), `mcp_config` (`:199`).

Test coverage: `../../etc/passwd` (`crates/buzz-persona/src/pack.rs:561-568`),
`..` in the middle of a path (`:570-575`), symlink escaping the pack root
(`:589-607`, `#[cfg(unix)]`), and the traversal case through the validator
(`crates/buzz-persona/src/validate.rs:496-517`).

Non-existent paths short-circuit before canonicalization and return the lexical join
(`crates/buzz-persona/src/pack.rs:349-355`). This is safe because layers 0 and 1 already
ran; the comment says so ("The `..` check above already guards against traversal").

`pack_root` itself is canonicalized once up front (`crates/buzz-persona/src/pack.rs:126`),
so the prefix comparison is symlink-stable.

---

### 4. Path traversal — gaps

| # | Gap | Evidence | Consequence |
|---|---|---|---|
| S1 | **Hook paths are stored raw and never validated.** The code documents this as an accepted, deferred risk. | `resolve_hooks` at `crates/buzz-persona/src/resolve.rs:339-357`; verbatim doc: "Security: we intentionally do NOT resolve these to absolute paths. Hook paths come from untrusted persona frontmatter and could contain `../` traversal. Since hooks are not executed in this PR, we store them as-is. The PR that wires execution MUST validate through `safe_resolve()` before use." | A persona declaring `on_start: "../../../bin/anything"` parses cleanly and the string reaches `ResolvedPersona.hooks`. Currently inert because nothing executes it — the risk transfers to whichever consumer wires execution. There is no compile-time or type-level marker preventing that consumer from using the string directly. Test `crates/buzz-persona/src/resolve.rs:577-590` locks in the raw-string behavior. |
| S2 | **Persona `skills` paths are never validated.** `LoadedPersona.skills` is `pc.skills` verbatim (`crates/buzz-persona/src/pack.rs:434`) and `ResolvedPersona.skills` is `lp.skills.clone()` (`crates/buzz-persona/src/resolve.rs:249`) — neither goes through `safe_resolve`. | `crates/buzz-persona/src/persona.rs:123` (raw `Vec<String>`) | `skills: ["../../secrets"]` survives resolution. The only normalization is `resolve_skills`/`advisory_check_skill_names` taking the final path component (`crates/buzz-persona/src/pack.rs:253-260`, `crates/buzz-persona/src/validate.rs:365-369`), which happens to defuse traversal *for those two functions only* — `ResolvedPersona.skills` still carries the raw string. |
| S3 | **`avatar` path is unvalidated.** Documented as "Pack-relative path to avatar image" (`crates/buzz-persona/src/persona.rs:109`) but never resolved or checked. | `crates/buzz-persona/src/persona.rs:110`; passthrough at `crates/buzz-persona/src/pack.rs:425` and `crates/buzz-persona/src/resolve.rs:243` | Any consumer that loads the avatar from this string must do its own containment check. |
| S4 | The `skills/` enumeration in `resolve_skills` and `advisory_check_skill_names` uses `pack_dir` as passed by the caller, **not** the canonicalized root, and does not re-verify containment. | `crates/buzz-persona/src/pack.rs:276` (`pack_dir.join("skills")`), `crates/buzz-persona/src/validate.rs:375` | Low impact — the joined literal `"skills"` cannot escape — but it means the two functions are not protected by the canonicalization performed in `load_pack`. |

---

### 5. Credential / secret handling

| Finding | Evidence |
|---|---|
| MCP `env` maps are the credential channel, and this crate stores their values **verbatim** — including `${VAR}` placeholders, which are *not* expanded | `McpServerConfig.env: HashMap<String,String>` (`crates/buzz-persona/src/persona.rs:78`); passthrough at `crates/buzz-persona/src/resolve.rs:325-333`; explicit doc "Env values are passed through as literals — no `${VAR}` interpolation" (`:274`); test asserts `env["SECRET"] == "${MY_SECRET}"` (`crates/buzz-persona/src/resolve.rs:558-572`) |
| Consequence: a pack **can** hard-code a literal secret in `env` and this crate will carry it into `ResolvedMcpServer.env` unchanged, with no detection, warning, or redaction | same call sites; no secret-pattern scanning anywhere in the crate |
| No `Debug` redaction — `ResolvedMcpServer` derives plain `Debug` (`crates/buzz-persona/src/resolve.rs:67`) and `ResolvedPersona` likewise (`:22`), so `{:?}` on a resolved persona prints all MCP env values in the clear | derives at `crates/buzz-persona/src/resolve.rs:22`, `:67`; `McpServerConfig` also derives `Debug` (`crates/buzz-persona/src/persona.rs:68`) |
| The crate never reads the process environment, so it cannot leak host secrets into pack data | no `std::env` usage in `crates/buzz-persona/src/` (verified by grep — the only `std::env` mentions are prose in `PERSONA_PACK_SPEC.md`); design intent stated at `crates/buzz-persona/src/resolve.rs:11` and `:360-361` |
| The crate never writes files or opens sockets, so it cannot exfiltrate | no `fs::write`/`create`/network APIs outside `#[cfg(test)]` |
| `runtime_env_vars` emits only model/provider/temperature/context-limit keys — no credential keys are synthesized | `crates/buzz-persona/src/resolve.rs:365-397` |

Note on the interpolation gap direction: because `${VAR}` is **not** expanded here, this
crate does not read secrets out of the environment. The residual risk is the inverse —
literal secrets committed into a pack file are passed through silently.

---

### 6. Capability / tool-permission model

| Finding | Evidence |
|---|---|
| There is **no** permission, capability, allow-list, or sandbox model in this crate. No `permissions`, `allowed_tools`, `tools`, `sandbox`, or `permission_mode` field exists on any type. | complete field inventories: `crates/buzz-persona/src/persona.rs:101-170`, `crates/buzz-persona/src/manifest.rs:79-121`, `crates/buzz-persona/src/resolve.rs:23-65` |
| Agent capability is granted **transitively** through MCP server definitions: a persona supplies `command` + `args` + `env`, which is a specification of an arbitrary subprocess for the runtime to launch. There is no allow-list of permitted commands, no path restriction on `command`, and no argument filtering. | `McpServerConfig` (`crates/buzz-persona/src/persona.rs:70-79`); merge preserves `command`/`args` unchanged (`crates/buzz-persona/src/resolve.rs:311-340`) |
| Collision policy amplifies this: a per-persona MCP entry **wholly replaces** a same-named pack-level entry, including its `command`. A persona can therefore repoint a pack's trusted server name at a different binary. | `crates/buzz-persona/src/resolve.rs:293-300`; test `mcp_merge_persona_wins_on_collision` (`:513-535`) confirms `command` is replaced (`old-cmd` → `new-cmd`) |
| Operator-owned limits are deliberately **out of pack authors' reach**: `idle_timeout`, `max_turn_duration`, `agents`, `heartbeat_interval`, `permission_mode` are all rejected by `deny_unknown_fields`. The test names the intent: "This documents the security boundary: pack authors define behavior, operators define limits." | `crates/buzz-persona/tests/integration.rs:632-650` |
| Operator env-var precedence (level 1) is **not** enforced here — the crate produces the desired env vars and documents that the consumer must skip injection when the operator already set a key. | `crates/buzz-persona/src/resolve.rs:359-364`; consumer field at `crates/buzz-acp/src/config.rs:533-535` |
| Persona `runtime` selects which env-var namespace is emitted, and any unrecognized value silently falls into the GOOSE_* branch rather than being rejected. | wildcard arm at `crates/buzz-persona/src/resolve.rs:380-386` |

---

### 7. Input-validation gaps (non-path)

| # | Gap | Evidence | Notes |
|---|---|---|---|
| V1 | `temperature` has no range validation — any `f64` including negatives, `inf` (via YAML `.inf`), or absurd magnitudes is accepted and projected into `GOOSE_TEMPERATURE` via `to_string()` | `crates/buzz-persona/src/persona.rs:151`; `crates/buzz-persona/src/merge.rs:96`; `crates/buzz-persona/src/resolve.rs:391-393` | consistent with the spec, which states buzz-acp "passes through without range validation" (`crates/buzz-persona/PERSONA_PACK_SPEC.md:815`) |
| V2 | `max_context_tokens` has no upper bound (any `u64`) | `crates/buzz-persona/src/persona.rs:154`; projected at `crates/buzz-persona/src/resolve.rs:395-397` | |
| V3 | `model` string is unvalidated apart from the colon split. An empty provider (`":model"`) is filtered to `None` (`crates/buzz-persona/src/resolve.rs:213`) but env-var **values** are otherwise arbitrary strings — newlines, `=`, shell metacharacters are all passed through into `runtime_env_vars` | `crates/buzz-persona/src/resolve.rs:369-387` | The consumer injects these via `Command::env()` per `crates/buzz-persona/PERSONA_PACK_SPEC.md:1145`, which is not shell-interpreted, so injection risk depends on the consumer not routing them through a shell. |
| V4 | `subscribe` channel names are unvalidated — no length, charset, or count limit; the `#` prefix is not stripped in this crate | `crates/buzz-persona/src/merge.rs:106-121`; spec says the consumer strips `#` (`crates/buzz-persona/PERSONA_PACK_SPEC.md:824-828`) | |
| V5 | Manifest `id`, `name`, `version` are only checked for presence and non-emptiness — `id` has no reverse-DNS/charset validation despite being used as a directory name by consumers (`~/.buzz/packs/<pack-id>/` per `crates/buzz-persona/PERSONA_PACK_SPEC.md:995`) | `crates/buzz-persona/src/manifest.rs:155-170` | A pack `id` containing `/` or `..` would be a traversal vector **in the consumer**, not here |
| V6 | `.mcp.json` is accepted as any valid JSON; a wrong shape contributes nothing and produces no diagnostic | `crates/buzz-persona/src/pack.rs:192-197`; `crates/buzz-persona/src/resolve.rs:283-291` | silent-empty, not an error |
| V7 | Persona `name` is validated for charset/length only on the **resolve** and **validate** paths, not in `parse_persona_md`. A direct `parse_persona_md` caller (e.g. the desktop import path) receives an unvalidated `name`. | validation at `crates/buzz-persona/src/resolve.rs:134-155` and `crates/buzz-persona/src/validate.rs:167-185`; absent from `crates/buzz-persona/src/persona.rs:208-260` | `desktop/src-tauri/src/migration.rs:1123` uses only `split_frontmatter`, bypassing even that |
| V8 | `name.len() > 64` is a **byte** comparison, not char count | `crates/buzz-persona/src/resolve.rs:146`, `crates/buzz-persona/src/validate.rs:169` | practically moot since the charset check already rejects non-ASCII |
| V9 | No total-pack limits: unbounded persona count, unbounded MCP servers per persona, unbounded keyword lists. Combined with the 256 KiB/persona body cap, a manifest listing thousands of personas is a memory-amplification path. | `crates/buzz-persona/src/pack.rs:155-164` loops over `manifest.personas` with no cap | Each individual file is capped; the aggregate is not. |
| V10 | Duplicate keys in the manifest `personas[]` array are not deduplicated before loading — the same file would be loaded twice, then caught by the duplicate-name check at `crates/buzz-persona/src/resolve.rs:125-133` (but only on the resolve/validate path, not on a bare `load_pack`) | `crates/buzz-persona/src/pack.rs:155-164` | `load_pack` alone will happily return two identical personas |

---

### 8. Denial-of-service considerations

| Finding | Evidence |
|---|---|
| Size caps exist at two layers: `read_bounded_file` stats before reading (`crates/buzz-persona/src/pack.rs:374-389`) and `parse_persona_file` stats before reading (`crates/buzz-persona/src/persona.rs:263-267`) — both avoid unbounded allocation on a huge file | limits `crates/buzz-persona/src/persona.rs:21`, `:24`; pack-level limit `crates/buzz-persona/src/pack.rs:153-154` |
| `instructions.md` and `.mcp.json` share the same ~1.25 MiB cap | `crates/buzz-persona/src/pack.rs:168-169`, used at `:180`, `:186`, `:205`, `:213` |
| The frontmatter close-delimiter scan is a forward-only loop with a monotonically advancing cursor — no backtracking, no quadratic blowup | `crates/buzz-persona/src/pack.rs` n/a; `crates/buzz-persona/src/persona.rs:289-306` (`search_from = after_dashes`) |
| YAML parsing is delegated to `serde_yaml` 0.9 within the 1 MiB frontmatter cap. No explicit alias/anchor-expansion guard exists in this crate; billion-laughs resistance depends entirely on `serde_yaml` behavior — **unverified** here. | `crates/buzz-persona/src/persona.rs:220` |
| `read_dir` on `skills/` is unbounded in entry count | `crates/buzz-persona/src/pack.rs:278-293`, `crates/buzz-persona/src/validate.rs:377-386` |
| No recursion in any parse path — no risk of stack exhaustion from nested structures other than what `serde_yaml`/`serde_json` impose internally | manual review of all parse functions |

---

### 9. Symlink / TOCTOU notes

- Symlink escape is caught by canonicalization + prefix check
  (`crates/buzz-persona/src/pack.rs:357-364`), with a Unix test at `:589-607`.
- There is a small TOCTOU window: `exists()` is checked (`crates/buzz-persona/src/pack.rs:157`,
  `:172`, `:200`), then `metadata()` (`:375`), then `read_to_string()` (`:367`) — three
  separate syscalls on the same path. A path replaced between the checks (e.g. file → symlink)
  could bypass the size check on the read. Exploiting it requires write access to the pack
  directory, which already implies control of the persona content.
- The advisory validation pass re-reads `plugin.json` from disk twice more
  (`crates/buzz-persona/src/validate.rs:213`, `:305`) after `load_pack` already read it,
  so the validator can in principle report on different content than was loaded.

---

### 10. Summary of security findings

| ID | Finding | Severity signal (factual, not graded) |
|---|---|---|
| S1 | Hook paths stored unvalidated, traversal-capable, execution deferred to consumer | documented-and-accepted in code (`crates/buzz-persona/src/resolve.rs:339-346`) |
| S2 | Persona `skills` paths unvalidated in `ResolvedPersona.skills` | no containment check on the emitted value |
| S3 | `avatar` path unvalidated | consumer-dependent |
| PI1 | Persona prompt/instructions/description flow into agent context verbatim | by design; size-capped only |
| PI2 | `SKILL.md` content never inspected despite being named a prompt-injection vector in the crate's own spec | `crates/buzz-persona/PERSONA_PACK_SPEC.md:1041-1043` |
| C1 | MCP `env` values (potential secrets) stored verbatim, `Debug`-printable, no redaction | `crates/buzz-persona/src/resolve.rs:325-333`, derives at `:67` |
| C2 | Literal secrets committed in a pack are neither detected nor warned about | no scanning logic |
| P1 | Persona-level MCP entry can replace a pack-level server's `command` entirely | `crates/buzz-persona/src/resolve.rs:293-300` |
| P2 | No tool-permission or capability model; capability = arbitrary MCP subprocess spec | complete field inventory |
| V1–V10 | Input-validation gaps enumerated in §7 | — |
| — | No `unsafe`, no panics in production paths, no env reads, no writes, no network | verified by grep |
