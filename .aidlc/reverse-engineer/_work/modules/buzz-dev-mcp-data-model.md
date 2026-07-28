## Module: buzz-dev-mcp (`crates/buzz-dev-mcp`)
### Aspect: Data Model

There is no database, no schema, and no persisted document store. All state is
either in-process memory or files in two process-scoped temp directories.

#### MCP tool input schemas (derived via `schemars::JsonSchema` + `serde::Deserialize`)

`ShellParams` (`crates/buzz-dev-mcp/src/shell.rs:119-128`)

| Field | Rust type | JSON | Required | Default |
|---|---|---|---|---|
| `command` | `String` | string | yes | — |
| `workdir` | `Option<String>` | string/null | no | server cwd (`shell.rs:145-149`) |
| `timeout_ms` | `Option<u64>` | integer/null | no | `120_000`, clamped to `600_000` (`shell.rs:141-144`) |

`ReadFileParams` (`crates/buzz-dev-mcp/src/read_file.rs:9-21`)

| Field | Rust type | Required | Default |
|---|---|---|---|
| `path` | `String` | yes | — |
| `offset` | `Option<usize>` | no | `0` (`read_file.rs:33`) |
| `limit` | `Option<usize>` | no | `DEFAULT_LIMIT = 2000` (`read_file.rs:6`, `read_file.rs:34`) |
| `workdir` | `Option<String>` | no | server cwd (`paths.rs:107-110`) |

`StrReplaceParams` (`crates/buzz-dev-mcp/src/str_replace.rs:12-23`)

| Field | Rust type | Required | Default |
|---|---|---|---|
| `path` | `String` | yes | — |
| `old_str` | `String` | yes | — |
| `new_str` | `String` | yes | — |
| `replace_all` | `bool` | no | `false` (`#[serde(default)]`, `str_replace.rs:19-20`) |
| `workdir` | `Option<String>` | no | server cwd |

`ViewImageParams` (`crates/buzz-dev-mcp/src/view_image.rs:63-77`)

| Field | Rust type | Required | Default / range |
|---|---|---|---|
| `source` | `String` | yes | file path, `http(s)://`, or `data:image/…;base64,…` (`view_image.rs:114-188`) |
| `max_dim` | `Option<u32>` | no | `1568`, clamped to `[64, 2048]` (`view_image.rs:38-40`, `view_image.rs:89-92`) |
| `workdir` | `Option<String>` | no | server cwd; ignored for URL sources (`view_image.rs:74-76`) |

`TodoParams` / `Item` / `HookParams` (`crates/buzz-dev-mcp/src/todo.rs:23-46`) — the only
schemas in the crate that set `#[serde(deny_unknown_fields)]` (`todo.rs:24`, `todo.rs:33`,
`todo.rs:45`).

| Type | Field | Rust type | Constraint |
|---|---|---|---|
| `Item` | `text` | `String` | `#[schemars(length(min = 1, max = 200))]` (`todo.rs:26`) |
| `Item` | `done` | `bool` | `#[serde(default)]` → `false` (`todo.rs:28-29`) |
| `TodoParams` | `todos` | `Option<Vec<Item>>` | `#[schemars(length(max = 50))]`; omitted **and** explicit `null` both mean "read" (`todo.rs:35-37`, `todo.rs:72`) |
| `HookParams` | — | empty struct | exists only because the `rmcp` macro requires `Parameters<T>` (`todo.rs:43-46`) |

Note the asymmetry: `ShellParams`, `ReadFileParams`, `StrReplaceParams`, and
`ViewImageParams` do **not** use `deny_unknown_fields`, so unrecognised keys are
silently dropped for those four tools.

#### Tool output shapes

`shell` returns a single text content block holding pretty-printed JSON
(`shell.rs:309-323`):

| Key | Type | Source |
|---|---|---|
| `exit_code` | integer | process code, else `124` on timeout, else `-1` (`shell.rs:298-301`) |
| `stdout` / `stderr` | string | possibly truncation-prefixed tail (`shell.rs:946-956`) |
| `timed_out` | bool | `shell.rs:218-296` |
| `duration_ms` | integer | `shell.rs:297` |
| `stdout_truncated` / `stderr_truncated` | bool | `shell.rs:894-900` return tuple |
| `stdout_artifact` / `stderr_artifact` | string \| null | absolute path to the artifact file (`shell.rs:914-928`) |
| `notes` | array of string | reaper/reader/artifact diagnostics accumulated during the run (`shell.rs:217`, `shell.rs:319`) |

`read_file` and `str_replace` return plain `String` (`lib.rs:56-61`,
`lib.rs:78-83`) — no structured content block. `view_image` returns two content
blocks: a text header then `Content::image(base64, mime)` (`view_image.rs:100-107`).
`todo`, `_Stop`, `_PostCompact` return a single text block via
`todo::text_result` (`todo.rs:240-243`).

#### In-process state

`SharedState` (`crates/buzz-dev-mcp/src/shell.rs:26-37`) — one instance per
process, wrapped in `Arc` and shared by every tool (`lib.rs:24-28`):

| Field | Type | Purpose |
|---|---|---|
| `cwd` | `PathBuf` | default workspace root for all path resolution (`shell.rs:27`) |
| `shim` | `Shim` | shim tempdir, PATH, and git env (`shell.rs:28`) |
| `session_dir` | `TempDir` | root for artifact files (`shell.rs:29`) |
| `bootstrap_instructions` | `String` | server `instructions` string, built once (`shell.rs:30`, `shell.rs:54`) |
| `resolved_shell` | `Result<(PathBuf, String), String>` | shell resolved exactly once so the bootstrap hint and every spawn agree (`shell.rs:31-35`, `shell.rs:47`) |
| `artifacts` | `Mutex<VecDeque<PathBuf>>` | 8-slot rotation ring (`shell.rs:36`, `shell.rs:23`) |
| `next_call_id` | `Mutex<u64>` | monotonic id used in artifact filenames (`shell.rs:37`, `shell.rs:65-72`) |

`TodoState` (`crates/buzz-dev-mcp/src/todo.rs:48-51`) — `Mutex<Vec<Item>>`. No ids;
the client sends a full replacement array. The list is per-process and dies with
the process (`todo.rs:9-14` module docs).

`Shim` (`crates/buzz-dev-mcp/src/shim.rs:18-22`) — `_dir: TempDir`,
`path_env: String`, `git_env: Vec<(String, String)>`. `KeyInfo`
(`shim.rs:78-82`) is the internal `{keyfile_path, pubkey_hex, npub}` triple used
to build `git_env`.

`CapturedStream` (`shell.rs:859-865`) — `{bytes: Vec<u8>, total_bytes: usize,
capped: bool}`; `total_bytes` keeps counting past the capture cap so the
truncation notice can report true output size (`shell.rs:867-892`).

`KillGroup` — three platform-specific shapes: `Option<i32>` pgid on Unix
(`shell.rs:694-695`), `{job: HANDLE}` on Windows (`shell.rs:738-741`), unit
struct elsewhere (`shell.rs:846-847`).

Multicall-personality internal types: `RgArgs` (`rg.rs:75-86`), `CappedSink`
(`rg.rs:137-141`), `Frame` (`tree.rs:11-16`). `view_image` internal types:
`PreparedImage` (`view_image.rs:81-86`) and `Encoded` (`view_image.rs:612-618`).

#### On-disk state

| Path | Mode | Contents | Lifetime |
|---|---|---|---|
| `$TMPDIR/buzz-dev-mcp-<rand>/` | `0700` (`shim.rs:219-224`) | multicall symlinks `rg`, `tree`, `buzz`, `git-credential-nostr`, `git-sign-nostr` (`shim.rs:32-40`) | `TempDir` drop |
| `$TMPDIR/buzz-dev-mcp-<rand>/.nostr-key` | `0600` via `OpenOptions::mode` (`shim.rs:134-144`) | raw `NOSTR_PRIVATE_KEY` string, plaintext | `TempDir` drop |
| `$TMPDIR/buzz-dev-mcp-session-<rand>/artifacts/{callid:06}.{stdout\|stderr}.txt` | default | up to 10 MiB captured stream bytes (`shell.rs:914-916`, `shell.rs:19`) | ring-evicted after 8 files (`shell.rs:971-981`) or `TempDir` drop |

The `0600` keyfile write is `create_new(true)`, so a pre-existing `.nostr-key`
causes the write to fail and git auth/signing to be disabled rather than
overwritten (`shim.rs:136-146`). On non-Unix the mode is not set — a plain
`std::fs::write` inside the (unrestricted on Windows) tempdir (`shim.rs:146-149`,
`shim.rs:226-229`).
