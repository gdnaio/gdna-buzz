## Module: buzz-dev-mcp (`crates/buzz-dev-mcp`)
### Aspect: API Surface

#### Rust public API

Exactly one item is reachable from outside the crate: `pub fn run() -> Result<(),
Box<dyn std::error::Error>>` (`crates/buzz-dev-mcp/src/lib.rs:138`). Every module
is private (`mod paths; mod read_file; … mod view_image;`, `lib.rs:13-21`), so the
`pub` items inside them (`shell::run`, `todo::text_result`, `paths::read_text_file`,
etc.) are crate-internal despite the `pub` keyword. `run()` carries **no doc
comment** (`lib.rs:137-138`).

Two `pub(crate)` helpers exist for Windows console suppression:
`configure_no_window` (`lib.rs:191`) and `configure_no_window_async`
(`lib.rs:205`).

#### Binary / CLI surface — multicall dispatch

`buzz-dev-mcp` is a multicall binary. `run()` reads `argv[0]`'s file stem,
lowercases it, and dispatches (`lib.rs:138-160`):

| `argv[0]` stem | Behaviour | Site |
|---|---|---|
| `rg` | `std::process::exit(rg::run(args[1..]))` — sync, no tokio runtime built | `lib.rs:149` |
| `tree` | `std::process::exit(tree::run(args[1..]))` — sync | `lib.rs:150` |
| `git-credential-nostr` | `std::process::exit(git_credential_nostr::run())` | `lib.rs:151` |
| `git-sign-nostr` | `std::process::exit(git_sign_nostr::run())` | `lib.rs:152` |
| `buzz` | `std::process::exit(buzz_cli::run_from_args(std::env::args()).await)` — needs the runtime | `lib.rs:168-171` |
| anything else | MCP server mode over stdio | `lib.rs:173-186` |

There are **no flags of its own** — no `--help`, no `--version`, no argument
parsing in MCP-server mode. Unknown `argv[0]` names fall through to server mode
rather than erroring. `crates/sprig/src/main.rs:39-41` relies on this: sprig
forwards any unmatched personality to `buzz_dev_mcp::run()`.

#### MCP server registration

Server info: name `"buzz-dev-mcp"`, version `env!("CARGO_PKG_VERSION")`, only the
`tools` capability enabled, plus `instructions` set from
`SharedState.bootstrap_instructions` (`lib.rs:126-136`). Transport is stdio
(`rmcp::transport::stdio`, `lib.rs:7`, `lib.rs:185`). Tracing goes to **stderr**
with ANSI disabled (`lib.rs:174-177`).

#### MCP tools — complete reference

Seven tools are registered via `#[tool_router]` / `#[tool(...)]` on
`impl DevMcp` (`lib.rs:30-125`). No tool is defined-but-unregistered; every
`#[tool]`-annotated method is inside the single `#[tool_router]` block.

| # | Tool name | Handler | Params type | Returns |
|---|---|---|---|---|
| 1 | `shell` | `lib.rs:44-50` | `ShellParams` | `CallToolResult` (JSON text block) |
| 2 | `read_file` | `lib.rs:56-61` | `ReadFileParams` | `String` |
| 3 | `view_image` | `lib.rs:67-72` | `ViewImageParams` | `CallToolResult` (text + image blocks) |
| 4 | `str_replace` | `lib.rs:78-83` | `StrReplaceParams` | `String` |
| 5 | `todo` | `lib.rs:89-98` | `TodoParams` | `CallToolResult` |
| 6 | `_Stop` | `lib.rs:105-110` | `HookParams` | `CallToolResult` |
| 7 | `_PostCompact` | `lib.rs:118-123` | `HookParams` | `CallToolResult` |

**`shell`** (`lib.rs:40-50` → `shell::run`, `shell.rs:130-323`)

- Params: `command: string` (required), `workdir: string?`, `timeout_ms: integer?`.
- Success: `CallToolResult::success` with one text block containing the JSON object
  documented in the Data Model aspect (`shell.rs:309-323`).
- `Err(ErrorData::invalid_params)`: command over 1,000,000 bytes
  (`shell.rs:135-140`); `workdir` missing or not a directory (`shell.rs:151-159`).
- `Ok(CallToolResult::error)` (not a protocol error): no shell could be resolved —
  returns the resolver's diagnostic text (`shell.rs:161-164`); spawn failed —
  `"failed to spawn shell: {e}"` (`shell.rs:183-190`); the request was cancelled —
  the literal string `"cancelled"` (`shell.rs:220-238`).
- The tool description advertises the on-PATH helpers (`rg`, `tree`, `buzz`) and
  the timeout defaults (`lib.rs:42`).

**`read_file`** (`lib.rs:52-61` → `read_file::run`, `read_file.rs:23-63`)

- Output is a header line `"{path} (lines {start}-{end} of {total})"` followed by
  `"{1-based-line-number}:{content}"` per line (`read_file.rs:48-58`), plus a
  continuation footer `"[showing lines … use offset=… to continue]"` when the
  window ends before EOF (`read_file.rs:59-63`).
- Two non-error string returns: `"{path} is empty (0 lines)"` (`read_file.rs:29-31`)
  and `"{path} (no lines in range, file has {n} lines)"` (`read_file.rs:39-44`).
- Errors come from `paths::read_text_file`: `invalid_params` for unresolvable path
  (`paths.rs:112-115`), non-regular file (`paths.rs:127-132`), file over 10 MiB
  (`paths.rs:131-142`), file grown mid-read (`paths.rs:154-161`);
  `internal_error` for stat/open/read failure and invalid UTF-8
  (`paths.rs:117-125`, `paths.rs:145-153`, `paths.rs:163-176`, `paths.rs:170-175`).

**`str_replace`** (`lib.rs:74-83` → `str_replace::run`, `str_replace.rs:25-106`)

- Success string: `"Replaced {1 occurrence|N occurrence(s)} in {abs-path}.\n\n{unified diff}"`
  (`str_replace.rs:97-106`). Diff has `context_radius(3)` and is cut at 64 KiB with
  a `"[diff truncated]"` marker (`str_replace.rs:140-155`).
- `invalid_params` errors: empty `old_str` (`str_replace.rs:26-31`); `old_str`/`new_str`
  over 1 MiB (`str_replace.rs:32-37`); zero matches — includes a truncated echo of
  `old_str` and an optional fuzzy nearest-line hint (`str_replace.rs:46-58`);
  multiple matches without `replace_all` (`str_replace.rs:60-68`); projected result
  over 10 MiB (`str_replace.rs:70-82`).
- `internal_error`: atomic write failure (`str_replace.rs:90-95`).

**`view_image`** (`lib.rs:63-72` → `view_image::run`, `view_image.rs:88-107`)

- Returns `[Content::text(header), Content::image(base64, mime)]` where header is
  `"{WxH}, {size} [(resized from WxH)] ({mime} from {source_label})"`
  (`view_image.rs:96-107`, `view_image.rs:530-611`).
- MIME is always one of `image/png` or `image/jpeg` on the resize path
  (`view_image.rs:640-654`), or the sniffed source MIME on the pass-through path
  (`view_image.rs:396-421`, `view_image.rs:560-567`).
- `invalid_params` errors: unsupported URL scheme (`view_image.rs:130-136`); data
  URL malformed / non-image / non-base64 / oversized (`view_image.rs:190-231`);
  file not a regular file or over 20 MiB (`view_image.rs:146-160`); non-2xx HTTP,
  with a distinct message naming `BUZZ_PRIVATE_KEY` on an unauthenticated
  401/403 (`view_image.rs:347-361`); `Content-Length` over cap
  (`view_image.rs:362-370`); mid-stream cap breach (`view_image.rs:378-384`);
  unsupported magic bytes (`view_image.rs:396-421`); animated GIF/WebP
  (`view_image.rs:534-536`); pixel count over 64 Mpx (`view_image.rs:549-558`);
  still oversized after two resize passes (`view_image.rs:589-595`).
- `internal_error`: HTTP client init, fetch failure, fetch read failure, stat/open/read
  failure (`view_image.rs:334-336`, `view_image.rs:344-346`, `view_image.rs:372-377`,
  `view_image.rs:141-144`).

**`todo`** (`lib.rs:85-98` → `TodoState::handle_todo`, `todo.rs:71-94`)

- Omitting `todos` (or sending `null`) reads; sending an array replaces
  (`todo.rs:72-94`). Output is the rendered checklist (`todo.rs:219-239`), or
  `"(todo list is empty)"`, with a `⚠️` warning block appended when open items
  disappeared (`todo.rs:78-92`, `todo.rs:192-218`).
- Validation failures return `Ok(CallToolResult::error)` with `"Error: {msg}"`
  (`lib.rs:93-97`, `todo.rs:245-248`), **not** a protocol error.

**`_Stop`** (`lib.rs:101-110`) returns `stop_objection()` — non-empty objection text
only when at least one item is not `done`, otherwise the empty string
(`todo.rs:99-112`). **`_PostCompact`** (`lib.rs:114-123`) returns
`post_compact()` — `"# Todo List\n{rendered}"`, or empty when the list is empty
(`todo.rs:113-124`).

#### `shim.rs`'s purpose

`shim.rs` is not an MCP surface; it constructs the execution environment the
`shell` tool hands to children (`Shim::install`, `shim.rs:25-76`):

1. Creates a `0700` tempdir (`shim.rs:26-27`).
2. Symlinks (Unix) or copies with `.exe` (Windows) the running binary under five
   names — `rg`, `tree`, `buzz`, `git-credential-nostr`, `git-sign-nostr` — which
   the multicall dispatch in `lib.rs:144-153` then recognises (`shim.rs:31-40`,
   `shim.rs:231-242`).
3. Prepends that dir to a `path_env` string built with `std::env::join_paths`
   (`shim.rs:42-49`).
4. Reads and unconditionally removes `NOSTR_PRIVATE_KEY` from the process env,
   writes it to a `0600` keyfile, and derives `GIT_CONFIG_COUNT` /
   `GIT_CONFIG_KEY_n` / `GIT_CONFIG_VALUE_n` for ten git settings
   (`shim.rs:51-68`, `shim.rs:178-216`).

`shim::artifact_dir` (`shim.rs:244-248`) creates and returns
`{session_root}/artifacts` and is the only other item `shell.rs` consumes from
this module (`shell.rs:914-915`). It has no doc comment.

#### Internal command-line surfaces (`rg`, `tree` personalities)

`rg::run` (`rg.rs:11-16`) first tries to exec the real system `rg`
(`try_system_rg`, `rg.rs:18-29`); when absent it parses a restricted flag set
itself (`rg.rs:87-135`): `--files`, `-n|--line-number`, `-i|--ignore-case`,
`-l|--files-with-matches`, `-C|--context <n>`, `-g|--glob <pat>`, `--`. Any other
`-`-prefixed token is rejected with `"unsupported flag (fallback rg): {s}"` and
exit 2 (`rg.rs:115-117`). Exit codes: `0` match found, `1` no match, `2` parse
error (`rg.rs:168-227`).

`tree::run` (`tree.rs:18-135`) accepts only `-d|--depth <n>`, a single positional
path, and `--` (`tree.rs:137-168`). Unknown flags → `"unknown flag: {s}"`, exit 2.
Output is an indented listing with per-file and per-directory line counts
(`tree.rs:100-134`); exit code is always `0` on a valid invocation.
