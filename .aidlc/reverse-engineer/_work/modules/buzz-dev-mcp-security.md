## Module: buzz-dev-mcp (`crates/buzz-dev-mcp`)
### Aspect: Security

#### Posture statement

This crate is **not a sandbox**. It is an unrestricted local-execution surface
that runs arbitrary shell commands as the invoking user, and reads and writes any
file that user can reach. The code says so in its own documentation:
`crates/buzz-dev-mcp/src/paths.rs:3-6` — "`resolve_path` resolves and
canonicalizes a user-supplied path against a workspace root. **No containment
enforcement — the resolved path may land anywhere on the filesystem** (consistent
with the `shell` tool's posture)." Every containment test in the crate asserts the
*absence* of a boundary: `resolve_path_allows_outside_workspace`
(`paths.rs:188-209`), `read_allows_absolute_path` (`read_file.rs:143-165`),
`run_allows_path_outside_workspace` (`str_replace.rs:256-281`),
`allows_path_outside_workspace` (`view_image.rs:803-830`). The only security
boundary in the design is the trust boundary drawn by the caller — the process
itself imposes resource limits, not authority limits.

#### `shell.rs` — command execution

| Question | Answer | Evidence |
|---|---|---|
| Is a shell interpreter invoked? | Yes. `Command::new(shell).arg(flag).arg(command)` — the entire command string is handed to `bash -c` (or `/C`, or `-Command`) for parsing | `shell.rs:165-167`, `shell.rs:336-348` |
| Is argv ever passed directly? | No. There is no argv-vector code path | `shell.rs:130-323` |
| Command allow-list? | None | no allow/deny list exists in the crate |
| Command deny-list? | None | — |
| Working-directory restriction? | Only `is_dir()`. Any absolute directory on the host is accepted as `workdir` | `shell.rs:145-159` |
| Can it escape a "sandbox root"? | There is no root to escape. `state.cwd` is only a *default*, overridable per call | `shell.rs:145-149` |
| Timeout? | Yes — 120 s default, hard-clamped to 600 s | `shell.rs:16-17`, `shell.rs:141-144` |
| Output cap? | Yes — 10 MiB capture, 50 KiB / 2000-line truncation trigger, 8 KiB tail returned | `shell.rs:19-22`, `shell.rs:903-956` |
| Concurrency / rate limit? | None. No semaphore or in-flight cap; each call spawns freely | `shell.rs:183-190` |
| stdin | `Stdio::null()` — the child cannot prompt for input, so interactive credential prompts fail closed | `shell.rs:175` |

Process containment on exit is thorough: `process_group(0)` on Unix
(`shell.rs:674-677`) or a `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE` Job Object on
Windows (`shell.rs:759-796`), `kill_on_drop(true)` (`shell.rs:178`), a
`KillGroup` guard whose `Drop` re-`SIGKILL`s unless explicitly disarmed
(`shell.rs:731-736`), and `kill_graceful()` invoked even on clean exit to reap
grandchildren still holding the pipes (`shell.rs:271-273`). This is a
resource-leak defence, not an authority defence.

#### Environment exposure and the private key

The crate never calls `env_clear()` (verified: no `env_clear` or `env_remove`
anywhere under `crates/buzz-dev-mcp/src/`). `tokio::process::Command` inherits the
parent environment by default; `shell::run` overrides only `PATH` and the
`GIT_CONFIG_*` pairs (`shell.rs:169-174`). Consequences:

1. **The agent can read `BUZZ_PRIVATE_KEY` through the shell tool.** `buzz-acp`
   injects it as a raw bech32 `nsec1…` into the MCP server's environment
   (`crates/buzz-acp/src/lib.rs:4152-4166`), and `buzz-agent`'s `PASSTHROUGH_ENV`
   explicitly preserves it across `env_clear()`
   (`crates/buzz-agent/src/mcp.rs:57-63`, `crates/buzz-agent/src/mcp.rs:708-727`).
   Nothing in `shell::run` strips it, so `shell(command: "env")`,
   `echo $BUZZ_PRIVATE_KEY`, or `cat /proc/self/environ` returns the private key
   in the tool result. The code comments this as intentional:
   `shell.rs:171` — "BUZZ_PRIVATE_KEY is intentionally inherited — the buzz CLI
   needs it."
2. `BUZZ_AUTH_TAG` (the NIP-OA owner attestation) is likewise inherited and
   readable the same way.
3. The tool result travels back through the agent into LLM history, so a key read
   this way leaves the process boundary.

`NOSTR_PRIVATE_KEY` is handled differently and better: it is read once and
**unconditionally removed** from the process environment before any child can be
spawned, regardless of whether the keyfile write then succeeds
(`shim.rs:51-56`), and the in-memory copy is zeroized (`shim.rs:65-68`). Its
value lands in a `0600` file inside a `0700` tempdir, created with
`OpenOptions::create_new(true).mode(0o600)` so there is no world-readable window
(`shim.rs:134-144`, `shim.rs:219-224`). On non-Unix both the mode and the
directory permission are no-ops (`shim.rs:146-149`, `shim.rs:226-229`), so on
Windows the keyfile inherits ambient ACLs.

That care is partly undone downstream: `git-sign-nostr` resolves keys with the
priority `NOSTR_PRIVATE_KEY` > `BUZZ_PRIVATE_KEY` > `git config nostr.keyfile`
(`crates/git-sign-nostr/src/lib.rs:394-404`). Because `BUZZ_PRIVATE_KEY` *is*
inherited, the shim's removal of `NOSTR_PRIVATE_KEY` narrows the exposure surface
but does not remove a signing key from the child environment.

Two further leaks of key *presence* rather than value: the bootstrap instructions
string appends "Buzz relay configured" only when both `BUZZ_RELAY_URL` and
`BUZZ_PRIVATE_KEY` are set (`shell.rs:77-83`), and `derive_git_email` embeds the
pubkey hex and relay host into `user.email` (`shim.rs:154-172`) — both are
non-secret but confirm identity and relay to the agent.

#### `paths.rs` — traversal defence, and the comparison to the known-good pattern

The actual checks, verbatim:

```rust
let raw = Path::new(path);
let candidate: PathBuf = if raw.is_absolute() {
    raw.to_path_buf()
} else {
    root.join(raw)
};

let resolved = std::fs::canonicalize(&candidate)
    .map_err(|e| format!("path not accessible: {} ({e})", candidate.display()))?;

Ok(resolved)
```
(`paths.rs:31-44`)

| Concern | `buzz-persona::safe_resolve` (`crates/buzz-persona/src/pack.rs:323-365`) | `buzz-dev-mcp::resolve_path` |
|---|---|---|
| Absolute paths | rejected — `/` prefix and Windows `X:` drive letters (`pack.rs:324-332`) | accepted verbatim (`paths.rs:32-33`) |
| `..` components | rejected eagerly, **before** canonicalizing (`pack.rs:333-340`) | not checked at all |
| Canonicalization | performed, resolving symlinks (`pack.rs:352-356`) | performed (`paths.rs:38-39`) |
| Containment | `canonical.starts_with(pack_root)` required, else `PathEscape` (`pack.rs:357-361`) | **no check** |
| Symlinks | followed, then containment re-verified | followed, result accepted wherever it lands (asserted by `paths.rs:194-205`) |

So the ordering question ("canonicalization before or after the containment
check") does not apply: there is no containment check. `resolve_path` performs
step 2 of the repo's known-good three-step pattern and omits steps 0, 1, and 3.

#### Per-tool path routing and resource limits

| Tool | Routes through `paths.rs`? | Resource limits |
|---|---|---|
| `read_file` | Yes — `paths::read_text_file` (`read_file.rs:24`) | 10 MiB file cap, strict UTF-8, 2000-line default window; **no byte cap on the returned string** (`read_file.rs:46-58`) |
| `str_replace` | Yes — `paths::read_text_file` (`str_replace.rs:39`) | 10 MiB read cap, 1 MiB per input string, 10 MiB projected-result cap, 64 KiB diff cap (`str_replace.rs:9`, `str_replace.rs:32-37`, `str_replace.rs:70-82`, `str_replace.rs:140-155`) |
| `view_image` | Yes — calls `resolve_path` directly, then stats/reads itself (`view_image.rs:137-171`) | 20 MiB source, 3 MiB output, 64 Mpx decoded, 256 MiB decoder alloc, 10 s fetch timeout (`view_image.rs:31-50`) |
| `shell` | No — `workdir` is used raw, unresolved and uncanonicalized (`shell.rs:145-149`) | see the table above |
| `tree` (personality) | No — its own `WalkBuilder` on an arbitrary path (`tree.rs:24-50`) | depth ≤ 50, 50 KiB / 2000 lines out, 10 MiB per-file line-count cap (`tree.rs:6-9`) |
| `rg` (personality) | No — its own recursive walk on arbitrary roots (`rg.rs:345-386`) | 50 KiB / 2000 lines out, 1 MiB per line, depth ≤ 50, context ≤ 100 (`rg.rs:5-9`) |

`str_replace` writes via `NamedTempFile::new_in(parent)` + `persist(target)`,
which is atomic within the target's directory and preserves the original mode
(`str_replace.rs:124-138`) — it will not silently widen permissions, but it will
happily rewrite any writable file on the host.

`view_image` is the crate's only outbound network client. Its credential gate is
tight and explicitly documented as the sole gate (`view_image.rs:228-235`):
scheme must be http/https, path must start with `/media/`, and host **and**
`port_or_known_default()` must both match `BUZZ_RELAY_URL`
(`view_image.rs:236-247`). `is_relay_media_url` is unit-tested against
cross-origin, wrong-port, wrong-path, and `"/mediafake/"` prefix-confusion cases
(`view_image.rs:1054-1092`). When a token is attached, redirects are disabled
outright because `x-auth-tag` is not on reqwest's sensitive-header list
(`view_image.rs:327-333`) — this closes a real redirect-leak path. The signed
token is a short-lived (600 s) server-scoped kind-24242 event with no `x` tag
(`view_image.rs:252-274`).

A residual exposure: the same key gate is fail-open by design — an invalid
`BUZZ_PRIVATE_KEY` degrades to an unsigned fetch with a `tracing::warn!` rather
than an error (`view_image.rs:286-318`). Since tracing writes to inherited stderr
(`lib.rs:174-177`, and `crates/buzz-acp/src/acp.rs:423` inherits stderr), warnings
land in the parent's log stream. The warning text itself does not include the key
value (`view_image.rs:302-305`).

#### Error-message and output leakage

| Path | What is disclosed |
|---|---|
| `paths::resolve_path` failure | the full canonical candidate path and the raw OS error (`paths.rs:38-41`) — confirms filesystem layout outside any workspace |
| `paths::read_text_file` failures | absolute target path, exact byte size, and the limit (`paths.rs:127-142`) |
| `str_replace` miss | up to 80 chars of the caller's own `old_str` plus a nearest-line excerpt from the target file (`str_replace.rs:46-58`, `str_replace.rs:196-217`) |
| `shell` | full stdout+stderr tail of any command, plus absolute artifact paths inside the session tempdir (`shell.rs:309-320`) |
| `shell` spawn/resolve errors | the resolved shell path and OS error, or the Windows resolver's full probe list (`shell.rs:183-190`, `shell.rs:447-458`) |
| `view_image` fetch errors | the full URL and HTTP status (`view_image.rs:344-361`) |
| Artifact files | 10 MiB of raw command output persisted to `$TMPDIR`, surviving in the ring until 8 newer artifacts arrive or the process exits (`shell.rs:914-928`, `shell.rs:971-981`) |

No error path echoes the process environment. The disclosure risk is not the error
strings — it is that `shell` will print the environment on request.

#### `unsafe` code

`lib.rs:1-2`: `#![cfg_attr(not(windows), forbid(unsafe_code))]` and
`#![cfg_attr(windows, deny(unsafe_code))]`. On Unix the crate is genuinely
`unsafe`-free. On Windows the `deny` is opted out of five times with
`#[allow(unsafe_code)]` (`shell.rs:474`, `shell.rs:750`, `shell.rs:753`,
`shell.rs:757`, `shell.rs:826`), covering the registry probe
(`git_bash_from_registry`, `shell.rs:475-539`), the Job Object lifecycle
(`shell.rs:756-822`, `shell.rs:825-844`), and two hand-written
`unsafe impl Send/Sync for KillGroup` (`shell.rs:749-754`). Each carries a
`SAFETY:` justification (`shell.rs:490-492`, `shell.rs:743-748`,
`shell.rs:766-771`, `shell.rs:810-813`, `shell.rs:833-836`). This is a documented
deviation from the repo's blanket "no `unsafe` code" rule (`AGENTS.md`), scoped to
Windows FFI.

#### Input-validation hardening that is present

Worth recording because it is unusually careful relative to the rest of the crate:

- `todo` rejects control characters, non-ASCII-space whitespace, zero-width
  characters, bidi overrides, and BOM specifically to prevent spoofing the
  rendered checklist (`todo.rs:126-144`, `todo.rs:160-166`), with six dedicated
  tests (`todo.rs:324-373`).
- `view_image` sniffs format from magic bytes and ignores both the extension and
  `Content-Type` (`view_image.rs:394-421`).
- The animated-GIF check is a deliberate allocation-free byte scan; the comment at
  `view_image.rs:415-422` records that using `GifDecoder::into_frames()` would let
  an attacker-controlled logical-screen size allocate multi-GB *before* the
  pixel cap fires.
- Decompression-bomb guard rejects `w*h > 64 Mpx` from the header before any
  pixel buffer is allocated (`view_image.rs:549-558`).
- `rg`'s `read_bounded_line` refuses to grow a line buffer past 1 MiB rather than
  reading an entire binary file into memory (`rg.rs:228-261`).
- `rg`'s `clean_path` strips PATH entries whose `rg` canonicalizes back to this
  binary, preventing infinite self-recursion (`rg.rs:31-47`).
- Windows shell resolution actively avoids the WSL launcher in `%SystemRoot%` and
  the `WindowsApps` alias stubs, with case-insensitive component comparison so
  `C:\WINDOWS\System32` cannot slip past a `C:\Windows` prefix test
  (`shell.rs:566-643`).
