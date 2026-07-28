## Module: buzz-dev-mcp (`crates/buzz-dev-mcp`)
### Aspect: Technical Debt

#### Repo-rule compliance

| Rule (`AGENTS.md`) | Status |
|---|---|
| No `unsafe` code | Violated on Windows only: 5 × `#[allow(unsafe_code)]` against the crate's own `deny` (`shell.rs:474`, `:750`, `:753`, `:757`, `:826`), covering the registry probe, Job Object lifecycle, and two hand-written `unsafe impl Send/Sync for KillGroup`. Non-Windows builds are `forbid(unsafe_code)` (`lib.rs:1-2`). All 13 `unsafe` occurrences carry `SAFETY:` comments |
| No new `unwrap()`/`expect()` in production paths | Clean. Every `expect(` and bare `unwrap()` in the crate is inside `#[cfg(test)]`. Production code uses only infallible `unwrap_or*` forms (`lib.rs:143`, `shell.rs:143`, `shell.rs:301`, `rg.rs:28`, `rg.rs:67`, `rg.rs:396`, `shim.rs:164`, `shim.rs:167`, `shim.rs:203`, `tree.rs:111`, `tree.rs:183`, `view_image.rs:91`, `view_image.rs:199`) |
| New public API must have doc comments | Violated. The crate's single externally-reachable item, `pub fn run()` (`lib.rs:138`), has no doc comment. `pub fn artifact_dir` (`shim.rs:244`) and `pub fn Shim::install` (`shim.rs:25`) also lack their own doc comments, though the `Shim` struct doc (`shim.rs:7-17`) covers `install`'s behaviour |
| TODO/FIXME/HACK/XXX count | **0.** The single `grep` hit across the whole crate is the string literal `"GIF89aXXX"` in a test fixture (`view_image.rs:1048`) |

#### File sizes against the 1,000-line ceiling

`AGENTS.md` enforces a hard 1,000-line-per-file ceiling for mobile
(`mobile/scripts/check-file-sizes.mjs` via `just mobile-check`) and states it
mirrors desktop and web. No equivalent guard exists for Rust crates, and two files
here exceed that number:

| File | Lines | Over ceiling |
|---|---|---|
| `shell.rs` | 1,503 | +503 |
| `view_image.rs` | 1,136 | +136 |
| `todo.rs` | 558 | — |
| `rg.rs` | 491 | — |
| `str_replace.rs` | 356 | — |
| `paths.rs` | 279 | — |
| `shim.rs` | 248 | — |
| `read_file.rs` | 234 | — |
| `lib.rs` | 213 | — |
| `tree.rs` | 184 | — |
| `main.rs` | 3 | — |

`shell.rs` mixes five concerns in one file: `SharedState`/bootstrap
(`shell.rs:26-117`), the tool handler (`shell.rs:130-323`), a ~150-line
cross-platform shell resolver (`shell.rs:326-672`), three `KillGroup`
implementations (`shell.rs:674-857`), and output capture/truncation
(`shell.rs:859-981`) — plus 520 lines of tests. `view_image.rs` similarly carries
image handling, HTTP fetching, and a Nostr/Blossom auth signer
(`view_image.rs:236-318`) in one module.

#### Untested surface

Three source files have zero tests: `lib.rs` (213 lines), `shim.rs` (248 lines),
`tree.rs` (184 lines). 23 of the crate's 97 tests are `#[cfg(windows)]`-gated
(`shell.rs:1139-1503`, `paths.rs:215-278`), so 74 run on a Unix CI host.

Specific untested behaviour:

| Untested item | Site |
|---|---|
| `Shim::install` end-to-end — 0700 dir mode, five symlinks, `path_env` prepending | `shim.rs:25-76` |
| `NOSTR_PRIVATE_KEY` removal from the process env — the crate's main secret-handling invariant | `shim.rs:51-56` |
| `write_keyfile` / `write_keyfile_atomic` 0600 mode and `create_new` collision behaviour | `shim.rs:87-132`, `shim.rs:134-149` |
| `build_git_env` — the ten `GIT_CONFIG_*` pairs and `GIT_CONFIG_COUNT` composition | `shim.rs:178-216` |
| `derive_git_email` — localhost/`127.*`/empty-host fallback to `@buzz` | `shim.rs:154-172` |
| `tree::run` and `tree::parse` — every rule, including depth clamping, `[truncated]` emission, leaf-directory annotation suppression, and the 10 MiB line-count skip | `tree.rs:18-184` |
| Multicall dispatch in `run()` — no test exercises `argv[0]`-based personality selection | `lib.rs:138-160` |
| `detect_stack` / `build_bootstrap` — the marker table and the conditional `BUZZ_RELAY_URL`+`BUZZ_PRIVATE_KEY` line | `shell.rs:75-117` |
| `finalize_stream` truncation and artifact writing; `rotate_artifacts` 8-file eviction; `align_to_char_boundary` | `shell.rs:894-981` |
| `read_capped` `CAPTURE_CAP` behaviour and `total_bytes` accounting | `shell.rs:867-892` |
| `shell` cancellation path — covered only from the agent side, in a test that skips when the binary is unbuilt | `shell.rs:218-239`; `crates/buzz-agent/tests/regressions.rs:1572-1600` |
| `rg::try_system_rg`, `clean_path`, `which_rg`, `walk`, `emit_line`, `read_bounded_line`, `CappedSink` line-limit branch | `rg.rs:18-73`, `rg.rs:228-261`, `rg.rs:336-386` |
| `str_replace`'s 10 MiB projected-size preflight and the 64 KiB diff cap | `str_replace.rs:70-82`, `str_replace.rs:140-155` |
| `read_file`'s lack of an output byte cap | `read_file.rs:46-58` |
| `view_image::fetch_url` — no HTTP test server; `Content-Length` rejection, mid-stream cap, redirect-disable, and 401/403 messaging are all unexercised | `view_image.rs:321-390` |
| `relay_media_get_auth` composition — only `is_relay_media_url` and `sign_media_get_auth` are unit-tested in isolation | `view_image.rs:286-318` vs `view_image.rs:1054-1136` |

#### Cross-platform defects

`rg.rs` hardcodes the Unix PATH separator and the Unix executable name:
`.split(':')` at `rg.rs:34` and `rg.rs:50`, and `dir.join("rg")` at `rg.rs:39` and
`rg.rs:54` — no `.exe` suffix, no `PATHEXT`, no `std::env::split_paths`. On Windows
`try_system_rg` therefore effectively never resolves a real ripgrep, so the
substring-only fallback silently becomes the permanent behaviour on that platform.
This contrasts with the rest of the crate, which is scrupulous about
`std::env::split_paths` (`shim.rs:44-48`, `shell.rs:378`, `shell.rs:627`,
`shell.rs:652`) and about `.exe` suffixing (`shell.rs:645-670`,
`shim.rs:236-242`).

`tree.rs` returns exit code `0` when `writeln!` fails (`tree.rs:106-108`,
`tree.rs:126-128`), so a broken pipe or full disk is reported as success.

#### Behavioural divergence from the advertised contract

The `shell` tool description tells the model that `rg` is on PATH with flags
`-n -i -l -g <glob> -C <n> --files` (`lib.rs:42`). When the fallback is in use the
match is a literal substring, not a regex (`rg.rs:289-296`), and the glob engine
is a hand-rolled `*`/`?` matcher with no `**`, no character classes, and no brace
expansion (`rg.rs:401-434`). A model that writes a regex pattern gets silently
wrong results with exit code `1` rather than an error. Nothing in the description
or the output distinguishes the delegated path from the fallback.

The `rg` fallback also caps output silently: `CappedSink` latches, logs a
`tracing::warn!` to stderr, and stops printing with **no marker on stdout**
(`rg.rs:152-166`) — unlike `shell`, `tree`, `read_file`, and `str_replace`, which
all emit an in-band truncation marker.

#### Inconsistency in error channels

`todo` returns validation failures as `Ok(CallToolResult::error)` in-band text
(`lib.rs:93-97`, `todo.rs:245-248`), while `str_replace`, `read_file`, and
`view_image` return the same class of user error as protocol-level
`Err(ErrorData::invalid_params)`. `shell` uses both: argument validation is a
protocol error (`shell.rs:136`, `shell.rs:152`) but shell-resolution failure,
spawn failure, and cancellation are in-band (`shell.rs:163`, `shell.rs:186`,
`shell.rs:237`).

Similarly, only `todo.rs` sets `#[serde(deny_unknown_fields)]` (`todo.rs:24`,
`:33`, `:45`). `ShellParams`, `ReadFileParams`, `StrReplaceParams`, and
`ViewImageParams` silently accept and discard unknown keys, so a model
misspelling `timeout_ms` gets the default with no signal.

#### Documentation drift

| Document | Status |
|---|---|
| `ARCHITECTURE.md` | `buzz-dev-mcp` is **absent** — zero matches for `dev-mcp` or `dev_mcp`. This crate is one of the ~11 not covered by that document |
| `AGENTS.md` | Present in the crate table (`AGENTS.md:51`) and referenced as "separate" from `buzz-cli` (`AGENTS.md:147`). Neither entry mentions `BUZZ_SHELL`, the absence of path containment, or the `buzz`/`rg`/`tree` multicall behaviour |
| `CONTRIBUTING.md` | **No mention at all** — zero matches for `dev-mcp`, despite `AGENTS.md` pointing there for "how to add event kinds / CLI subcommands / HTTP endpoints" |
| `README.md` | One clause: "`buzz-dev-mcp` (shell + file-edit tools)" (`README.md:207`) |
| `VISION_AGENT.md` | The most substantive prose (`VISION_AGENT.md:1`, `:15`, `:45`). Its claim that "File edits resolve against the working directory" (`:15`) is misleading — relative paths resolve against `workdir`, but absolute paths and `..` are accepted without restriction (`paths.rs:3-6`, `paths.rs:32-33`) |
| Crate-level docs | There is **no `crates/buzz-dev-mcp/README.md`**, and `lib.rs` has no `//!` module doc. Four modules do have `//!` headers (`paths.rs:1-9`, `shim.rs:7-17` as a struct doc, `todo.rs:1-18`, `view_image.rs:1-12`); `shell.rs`, `rg.rs`, `tree.rs`, `read_file.rs`, `str_replace.rs`, and `main.rs` have none |

#### Duplication with `buzz-cli` and elsewhere

There is no tool-level duplication of `buzz-cli`: the router exposes no relay
operation (`lib.rs:40-125`), and `buzz-cli` is instead linked in and re-exposed
verbatim as the `buzz` multicall personality (`lib.rs:168-171`). The
`AGENTS.md:147` boundary holds.

Real duplication exists elsewhere:

- The Blossom `t=get` signer (`view_image.rs:252-274`) reimplements the desktop
  client's media-read auth, with the constant `MEDIA_GET_AUTH_EXPIRY_SECS = 600`
  copied by reference rather than shared (`view_image.rs:51-53`). Divergence
  between the two is possible and unguarded.
- `resolve_path` (`paths.rs:20-45`) is a fourth path-resolution implementation
  alongside `buzz-persona::safe_resolve` (`crates/buzz-persona/src/pack.rs:323-361`),
  with different (weaker) semantics and no shared helper.
- An identical `fn make_state(cwd)` test fixture is duplicated in four modules
  (`shell.rs:991-994`, `read_file.rs:74-77`, `str_replace.rs:231-234`,
  `view_image.rs:684-687`).
- Numeric caps are re-declared per module rather than shared: `50 KiB` output cap
  appears as `MAX_BYTES` (`shell.rs:20`), `MAX_OUTPUT_BYTES` (`rg.rs:6`), and
  `MAX_OUTPUT_BYTES` (`tree.rs:6`); `2000` lines as `MAX_LINES` (`shell.rs:21`),
  `MAX_OUTPUT_LINES` (`rg.rs:7`), `MAX_OUTPUT_LINES` (`tree.rs:7`), and
  `DEFAULT_LIMIT` (`read_file.rs:6`); `10 MiB` as `MAX_FILE_BYTES` in both
  `paths.rs:15` and `tree.rs:9`, and as `CAPTURE_CAP` in `shell.rs:19`.

#### Dead or unreachable code

- `GIT_CONFIG_COUNT` composition (`shim.rs:199-215`) is documented as reachable
  only when dev-mcp is run standalone, because `buzz-agent` calls `env_clear()`
  before spawning (`shim.rs:174-177`, `crates/buzz-agent/src/mcp.rs:714`).
- The `#[cfg(not(any(unix, windows)))] KillGroup` stub (`shell.rs:846-857`) exists
  only to keep the crate compiling; no such target is built.
- `Encoded.summary` is always constructed as `String::new()` by `encode_resized`
  and filled by the caller instead (`view_image.rs:617`, `view_image.rs:657-663`,
  `view_image.rs:602-610`) — a field that is never meaningfully written where it
  is defined.
- `truncate` (`str_replace.rs:157-164`) and `truncate_str`
  (`str_replace.rs:166-175`) are two near-identical truncation helpers differing
  only in char-vs-byte accounting.
