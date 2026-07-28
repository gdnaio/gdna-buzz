## Module: buzz-dev-mcp (`crates/buzz-dev-mcp`)
### Aspect: Conventions

#### Tool-definition pattern

Every tool follows the same three-part shape:

1. A params struct in the tool's own module, deriving
   `#[derive(Debug, Deserialize, JsonSchema)]` — e.g. `ShellParams`
   (`crates/buzz-dev-mcp/src/shell.rs:119-128`), `ReadFileParams`
   (`read_file.rs:9-21`), `StrReplaceParams` (`str_replace.rs:12-23`),
   `ViewImageParams` (`view_image.rs:63-77`), `TodoParams` (`todo.rs:32-38`).
   Optional fields are `Option<T>` with `#[serde(default)]`.
2. A free `run` function in that module taking `&SharedState` plus the params —
   `shell::run` (`shell.rs:130`), `read_file::run` (`read_file.rs:23`),
   `str_replace::run` (`str_replace.rs:25`), `view_image::run`
   (`view_image.rs:88`). `todo` deviates: it is a method on `TodoState`
   (`todo.rs:71`).
3. A thin `#[tool]`-annotated method on `DevMcp` in `lib.rs` that unwraps
   `Parameters(p)` and delegates (`lib.rs:40-125`). No business logic lives in
   `lib.rs`.

Tool descriptions are long, prose, and operationally specific — they name
defaults, caps, and the helper binaries on `PATH` (`lib.rs:42`, `lib.rs:54`,
`lib.rs:65`, `lib.rs:76`, `lib.rs:87`). Hook tools are prefixed with an
underscore to mark them as agent-lifecycle rather than user-facing: `_Stop`,
`_PostCompact` (`lib.rs:102`, `lib.rs:115`).

Shared plumbing is centralised rather than duplicated: `paths::read_text_file` is
the single resolve→stat→size-check→read→UTF-8 pipeline used by both `read_file`
and `str_replace` (`paths.rs:102-180`), and `paths::resolve_path` is the single
path resolver (`paths.rs:20-45`).

#### Error handling

Two distinct failure channels, used inconsistently across tools:

| Channel | Meaning | Used by |
|---|---|---|
| `Err(ErrorData::invalid_params(msg, None))` | caller's fault | all four file/shell tools' argument validation (`shell.rs:136`, `read_file` via `paths.rs:113`, `str_replace.rs:27`, `view_image.rs:109-111`) |
| `Err(ErrorData::internal_error(msg, None))` | server/IO fault | `paths.rs:118`, `paths.rs:147`, `paths.rs:166`, `paths.rs:171`, `str_replace.rs:91`, `view_image.rs:141` |
| `Ok(CallToolResult::error(vec![Content::text(...)]))` | in-band tool failure the model should read | `shell` no-shell / spawn-fail / cancelled (`shell.rs:163`, `shell.rs:186`, `shell.rs:237`); all `todo` validation failures (`lib.rs:93-97`, `todo.rs:245-248`) |

So `todo`'s validation errors are in-band text while `str_replace`'s are protocol
errors — the same class of user mistake surfaces differently depending on the
tool.

No `panic!`, `unwrap()`, or `expect()` exists in any production path. Production
code uses only infallible fallbacks: `unwrap_or("")` (`lib.rs:143`),
`unwrap_or(DEFAULT_TIMEOUT_MS)` (`shell.rs:143`), `unwrap_or(2)` (`rg.rs:28`),
`unwrap_or(0)` (`shim.rs:203`, `tree.rs:183`), `unwrap_or(DEFAULT_MAX_DIM)`
(`view_image.rs:91`). Every `expect(` and bare `unwrap()` in the crate is inside
`#[cfg(test)]`.

Mutex poisoning is never propagated as a panic: the crate uses
`Err(p) => p.into_inner()` uniformly (`shell.rs:66-69`, `shell.rs:972-975`,
`todo.rs:59-62`).

Failures that would be cosmetic are deliberately swallowed with `let _ =`:
`killpg` results (`shell.rs:709`, `shell.rs:720-722`), artifact deletion
(`shell.rs:977`), permission restore after atomic write (`str_replace.rs:135`),
artifact dir creation (`shim.rs:246`), `rustls` provider install (`lib.rs:166`),
and the `tree` truncation writes (`tree.rs:123`, `tree.rs:132`).

Diagnostics that must reach the operator but not the model go to stderr:
`eprintln!` for shim key problems (`shim.rs:94-98`, `shim.rs:109-111`,
`shim.rs:116-118`) and `tracing::{error,warn,debug}` elsewhere (`rg.rs:157-159`,
`tree.rs:21`, `tree.rs:27`, `shell.rs:229`, `shell.rs:233`,
`view_image.rs:302`, `view_image.rs:314`). The subscriber is explicitly pinned to
stderr with ANSI off (`lib.rs:174-177`) so stdout stays a clean MCP framing
channel.

#### Output formatting

- Structured results are JSON via `serde_json::json!` + `to_string_pretty`, wrapped
  in one text content block (`shell.rs:309-321`).
- Human-readable results are plain strings with an explicit machine-parseable
  header line: `"{path} (lines {a}-{b} of {n})"` (`read_file.rs:50-53`),
  `"Replaced N occurrence(s) in {path}."` (`str_replace.rs:97-106`).
- Truncation is always announced in-band with a bracketed marker:
  `"[truncated: showing last … bytes; … lines / … bytes total …]"`
  (`shell.rs:946-952`), `"[diff truncated]"` (`str_replace.rs:149`),
  `"[showing lines … use offset=… to continue]"` (`read_file.rs:59-63`),
  `"[truncated]"` (`tree.rs:123`, `tree.rs:132`). The `rg` fallback is the
  exception — it caps silently and only logs (`rg.rs:152-166`).
- Byte sizes are humanised via a local `human_bytes` (`view_image.rs:665-675`).
- Limits are always named in error text alongside the offending value
  (`paths.rs:133-141`, `str_replace.rs:33-36`, `view_image.rs:154-159`).

Every numeric limit is a named `const` at the top of its module rather than an
inline literal: `shell.rs:16-24` (9 consts), `view_image.rs:31-53` (8),
`rg.rs:5-9` (5), `tree.rs:6-9` (4), `str_replace.rs:9-10` + `:140` (3),
`todo.rs:20-21` (2), `paths.rs:15` (1), `read_file.rs:6` (1).

#### Platform-conditional code style

Platform divergence is handled with paired `#[cfg]` functions of identical
signature rather than inline branching: `resolve_bash`
(`shell.rs:363-388` / `shell.rs:409-459`), `set_process_group`
(`shell.rs:674-680`), `KillGroup` in three variants
(`shell.rs:694-736` / `shell.rs:738-844` / `shell.rs:846-857`),
`write_keyfile_atomic` (`shim.rs:134-149`), `set_owner_only`
(`shim.rs:218-229`), `symlink` (`shim.rs:231-242`), `is_executable`
(`rg.rs:62-73`), `configure_no_window` (`lib.rs:189-212`). The
non-`unix`/non-`windows` `KillGroup` stub exists purely to keep the crate
compiling everywhere (`shell.rs:846-857`).

`is_windows_apps_alias` is gated `#[cfg(any(windows, test))]` (`shell.rs:574`) so
its six predicate tests run on every host (`shell.rs:1083-1137`) — a deliberate
pattern for making Windows-only logic testable on CI runners.

Comments are unusually dense and explain *why*, often citing the bug they fixed:
the MSYS translation rationale (`paths.rs:21-27`), the three MSYS forms and why
form 3 is not guessed (`paths.rs:47-59`), the `0x8007072c` WSL spawn failure that
motivated case-insensitive `is_under_dir` (`shell.rs:595-601`), why the
`sleep 5` timeout test is not `sleep 999` (`shell.rs:1030-1035`), and why
`GifDecoder::into_frames()` must not be used (`view_image.rs:415-422`).

#### Test organisation and counts

All tests are inline `#[cfg(test)] mod tests` at the bottom of each source file.
There is no `tests/` directory in the crate and no integration-test target.

| File | Test count | Notes |
|---|---|---|
| `todo.rs` | 25 | `todo.rs:251-558`; the densest coverage in the crate |
| `shell.rs` | 24 | 3 behavioural + 6 cross-host predicate tests in `mod tests` (`shell.rs:984-1137`), plus 15 in `#[cfg(all(test, windows))] mod windows_resolver_tests` (`shell.rs:1139-1503`) that never run on Linux/macOS CI |
| `view_image.rs` | 18 | `view_image.rs:678-1136` |
| `paths.rs` | 9 | 1 cross-platform, 8 in `#[cfg(windows)] mod windows_msys` (`paths.rs:215-278`) |
| `read_file.rs` | 8 | `read_file.rs:66-234` |
| `str_replace.rs` | 7 | `str_replace.rs:220-356` |
| `rg.rs` | 6 | `rg.rs:432-491` — parser and glob only |
| `lib.rs` | 0 | — |
| `shim.rs` | 0 | — |
| `tree.rs` | 0 | — |
| **Total** | **97** | of which 23 are `#[cfg(windows)]`-gated, so 74 run on a Unix CI host |

Test naming is descriptive-assertive (`rejects_duplicate_text_after_trim`,
`windows_apps_alias_first_real_bash_second_returns_real`,
`pixel_count_cap_rejects_decompression_bomb`,
`relay_media_url_matches_relay_host_only`). Shared fixtures are duplicated per
module rather than extracted: an identical `fn make_state(cwd)` helper appears
four times (`shell.rs:991-994`, `read_file.rs:74-77`, `str_replace.rs:231-234`,
`view_image.rs:684-687`).

Windows env-mutating tests serialise on a module-local
`static ENV_MUTEX: Mutex<()>` held for the whole test body
(`shell.rs:1144-1149`), with explicit save/restore of `SystemRoot`, `BUZZ_SHELL`,
and `GIT_BASH` (`shell.rs:1209-1219`, `shell.rs:1341-1356`).

Several tests synthesise fixtures byte-by-byte rather than allocating real data:
`synth_oversized_png` hand-rolls IHDR/IDAT/IEND with a local CRC32 so a 9000×9000
declaration can be tested without an 81 Mpx buffer (`view_image.rs:1001-1040`,
used by `pixel_count_cap_rejects_decompression_bomb` at `view_image.rs:987-992`),
and `webp_scan_detects_animated_via_vp8x_flag` hand-builds a VP8X header
(`view_image.rs:966-985`).
