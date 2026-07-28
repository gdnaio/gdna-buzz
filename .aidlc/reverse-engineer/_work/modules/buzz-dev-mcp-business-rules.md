## Module: buzz-dev-mcp (`crates/buzz-dev-mcp`)
### Aspect: Business Rules

#### Cross-cutting: path resolution

One function resolves paths for every file tool: `paths::resolve_path`
(`crates/buzz-dev-mcp/src/paths.rs:20-45`).

| Rule | Evidence |
|---|---|
| Absolute inputs are used verbatim | `if raw.is_absolute() { raw.to_path_buf() }` (`paths.rs:32-33`) |
| Relative inputs are joined onto the workspace root | `root.join(raw)` (`paths.rs:34-36`) |
| The joined path is then `std::fs::canonicalize`d | `paths.rs:38-39` |
| Canonicalization failure is the **only** rejection | error string `"path not accessible: {} ({e})"` (`paths.rs:38-41`) |
| No `..` rejection, no containment check, no symlink restriction | module doc states it outright: "No containment enforcement — the resolved path may land anywhere on the filesystem" (`paths.rs:3-6`) |
| The workspace root is whatever the caller passed as `workdir`, unvalidated | `PathBuf::from(w)` with no checks (`paths.rs:107-110`) |

`resolve_path` requires the target to already exist (canonicalize fails
otherwise), so all four file tools are read/modify-only — none can create a new
file at a fresh path.

On Windows only, MSYS-form absolute paths are rewritten before resolution
(`msys_to_windows`, `paths.rs:62-99`): `/c/Users/x` → `C:\Users\x`,
`/c` → `C:\`, `//server/share/x` → `\\server\share\x`. Two-letter leading
segments (`/cc/x`) and root-anchored MSYS paths (`/tmp`, `/usr/bin/git`) are
deliberately left untranslated so they fail with a clear error instead of being
mis-mapped (`paths.rs:51-59`, `paths.rs:96-98`).

`paths::read_text_file` (`paths.rs:102-180`) layers the shared read pipeline on
top:

| Rule | Limit / behaviour | Evidence |
|---|---|---|
| Must be a regular file | `invalid_params` `"not a regular file: …"` | `paths.rs:127-132` |
| Size cap | `MAX_FILE_BYTES = 10 MiB`; `invalid_params` naming both actual and limit | `paths.rs:15`, `paths.rs:131-142` |
| TOCTOU re-check | read is `take(MAX_FILE_BYTES + 1)`; over-cap read yields `"file grew past … bytes during read"` | `paths.rs:154-161` |
| Encoding | strict UTF-8 via `String::from_utf8`; invalid bytes are an `internal_error`, never lossily decoded | `paths.rs:163-176` |

#### `shell`

| Rule | Behaviour | Evidence |
|---|---|---|
| Command size | over `MAX_COMMAND_BYTES = 1_000_000` → `invalid_params` | `shell.rs:18`, `shell.rs:135-140` |
| Timeout default | `120_000 ms` when omitted | `shell.rs:16`, `shell.rs:141-144` |
| Timeout cap | silently `min`-clamped to `MAX_TIMEOUT_MS = 600_000`; a larger request is *not* an error | `shell.rs:17`, `shell.rs:141-144` |
| Working directory | `workdir` if given, else `state.cwd`; must satisfy `is_dir()` or `invalid_params` — no containment check of any kind | `shell.rs:145-159` |
| Interpreter | a shell **is** invoked: `Command::new(shell).arg(flag).arg(command)`; the command string is parsed by the shell | `shell.rs:166-167` |
| Shell flag dispatch | `cmd` → `/C`, `powershell`/`pwsh` → `-Command`, everything else → `-c` | `shell.rs:336-348` |
| Command allow-list / deny-list | none exists anywhere in the crate | `shell.rs:130-323` |
| stdin | always `Stdio::null()` — commands cannot read input | `shell.rs:175` |
| Process group | `process_group(0)` on Unix; Windows Job Object with `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE` | `shell.rs:674-677`, `shell.rs:759-796` |
| Capture cap | reader stops storing at `CAPTURE_CAP = 10 MiB` but keeps counting `total_bytes` | `shell.rs:19`, `shell.rs:867-892` |
| Truncation trigger | any of: capture capped, over `MAX_BYTES = 50 KiB`, or over `MAX_LINES = 2000` newlines | `shell.rs:20-21`, `shell.rs:906-908` |
| Truncation shape | full captured bytes written to an artifact file, then the **last** `TAIL_BYTES = 8 KiB` returned behind a `[truncated: …]` notice | `shell.rs:22`, `shell.rs:914-956` |
| UTF-8 boundary | tail start is advanced past continuation bytes (`b & 0xC0 == 0x80`) before slicing | `shell.rs:930-931`, `shell.rs:959-965` |
| Encoding of output | `lossy()` — valid UTF-8 passes through, otherwise `from_utf8_lossy` | `shell.rs:967-969` |
| Artifact rotation | ring of `ARTIFACT_RING_SIZE = 8`; the evicted file is deleted | `shell.rs:23`, `shell.rs:971-981` |
| Artifact write failure | non-fatal; appends a `notes` entry and returns `artifact: null` | `shell.rs:917-928` |
| Timeout kill sequence | `SIGTERM` to the process group → 200 ms sleep → `SIGKILL`, then a 2-second `try_wait` poll loop, then `start_kill` + `wait` | `shell.rs:713-724`, `shell.rs:242-269` |
| Timeout exit code | `124` when the process yielded no code and timed out; `-1` otherwise | `shell.rs:298-301` |
| Post-success kill | `kill_graceful()` is called even on a clean exit, to reap orphaned grandchildren holding the pipes | `shell.rs:271-273` |
| Cancellation | `notifications/cancelled` → immediate `SIGKILL` of the group, 1-second bounded reap, reader tasks aborted, tool returns the text `"cancelled"` | `shell.rs:218-239` |
| Reader stall | each reader gets 5 s to finish after process exit; on timeout it is aborted, an empty stream is substituted, and a `notes` entry is added | `shell.rs:275-295` |
| Drop safety | `KillGroup::drop` re-`SIGKILL`s the group unless `disarm()` ran; `disarm()` happens only on the success path and the clean cancel-reap | `shell.rs:731-736`, `shell.rs:226`, `shell.rs:322` |

#### `read_file`

| Rule | Behaviour | Evidence |
|---|---|---|
| Empty file | short-circuits with `"{path} is empty (0 lines)"` before any windowing | `read_file.rs:27-31` |
| Offset | 0-based; clamped with `offset.min(total)` so an out-of-range offset yields an empty slice, not a panic | `read_file.rs:36` |
| Limit | default 2000; `limit: 0` produces the "no lines in range" message | `read_file.rs:34`, `read_file.rs:37-44` |
| Line numbering | output uses 1-based numbers (`offset + i + 1`) while `offset` itself is 0-based | `read_file.rs:46-58` |
| Line splitting | `content.lines()` — `\r\n` line endings lose the `\r`, and a missing trailing newline still counts as a line | `read_file.rs:24`, test `read_file_without_trailing_newline` (`read_file.rs:218-233`) |
| Continuation hint | emitted only when `end_line < total`, advising `offset={end_line}` | `read_file.rs:59-63` |
| Output size | **no byte cap on the returned string** — 2000 arbitrarily long lines are returned in full | `read_file.rs:48-58` |

#### `str_replace`

| Rule | Behaviour | Evidence |
|---|---|---|
| `old_str` non-empty | required; empty → `invalid_params` | `str_replace.rs:26-31` |
| Input size | `old_str`/`new_str` each capped at `MAX_INPUT_BYTES = 1 MiB` | `str_replace.rs:9`, `str_replace.rs:32-37` |
| Match semantics | plain byte-substring search (`str::find` / `str::matches`) — **not** regex, whitespace- and case-sensitive, no normalisation | `str_replace.rs:41-45`, `str_replace.rs:108-122` |
| Uniqueness | without `replace_all`, exactly one match is required; ≥2 → `invalid_params` advising more context | `str_replace.rs:60-68` |
| Counting shortcut | the non-`replace_all` path uses `count_occurrences_capped`, which stops at 2 | `str_replace.rs:108-122` |
| `replace_all` | replaces every occurrence; zero matches is still an error | `str_replace.rs:41-43`, `str_replace.rs:46-58`, `str_replace.rs:84-88` |
| Miss diagnostics | echoes `old_str` truncated to 80 chars plus a fuzzy nearest-line hint | `str_replace.rs:46-58`, `str_replace.rs:157-164` |
| Hint scope | only the first `HINT_SCAN_LINE_LIMIT = 200` lines are scanned, similarity threshold `> 0.6`, comparison strings truncated to 512 bytes | `str_replace.rs:10`, `str_replace.rs:177-217` |
| Growth preflight | projected size computed before allocation; negative or over 10 MiB → `invalid_params` | `str_replace.rs:70-82` |
| Write atomicity | `NamedTempFile::new_in(parent)` → write → flush → `persist(target)` rename | `str_replace.rs:124-138` |
| Permission preservation | original `Permissions` re-applied after the rename; failure is ignored (`let _ =`) | `str_replace.rs:127`, `str_replace.rs:134-137` |
| Diff budget | hunks appended until 64 KiB, then `"[diff truncated]"` | `str_replace.rs:140-155` |
| Encoding | inherits `read_text_file`'s strict UTF-8 rule — binary files cannot be edited | `paths.rs:163-176` |

#### `view_image`

| Rule | Behaviour | Evidence |
|---|---|---|
| `max_dim` | default 1568, `clamp(64, 2048)` — out-of-range values are silently clamped | `view_image.rs:38-40`, `view_image.rs:89-92` |
| Source dispatch order | `data:` → `http(s)://` → any other `scheme://` rejected → filesystem path | `view_image.rs:117-188` |
| Non-http scheme guard | `src.contains("://")` rejects e.g. `ftp://`, preventing it becoming a relative path | `view_image.rs:130-136` |
| Data URL rules | must be `data:image/*`, must declare `;base64`; percent-encoded forms rejected; encoded length prechecked at `ceil(20MiB/3)*4 + 4` chars; decoded length re-checked | `view_image.rs:190-231`, `view_image.rs:118-129` |
| File source | must be a regular file; ≤ `MAX_SOURCE_BYTES = 20 MiB`; read is `take(cap + 1)` for TOCTOU | `view_image.rs:31`, `view_image.rs:137-171` |
| URL source | 10 s connect **and** read timeout; up-front `Content-Length` rejection; per-chunk cumulative cap enforcement | `view_image.rs:50`, `view_image.rs:325-390` |
| Format detection | magic bytes only — extension and `Content-Type` are ignored. Accepts PNG, JPEG, GIF87a/89a, WebP (RIFF…WEBP) | `view_image.rs:394-421` |
| Animation rejection | animated GIF (≥2 image descriptors, byte-level scan) and animated WebP (VP8X flags byte offset 20, bit 1) are rejected outright | `view_image.rs:424-527`, `view_image.rs:534-536` |
| Decompression-bomb guard | dimensions read header-only, then `w*h > MAX_PIXELS = 64 Mpx` → reject *before* decoding | `view_image.rs:45`, `view_image.rs:549-558` |
| Decoder allocation cap | `Limits::max_alloc = 256 MiB` applied on the resize path | `view_image.rs:48`, `view_image.rs:55-61`, `view_image.rs:562-566` |
| Pass-through | if longest edge ≤ `max_dim` **and** bytes ≤ `MAX_FINAL_RAW_BYTES = 3 MiB`, original bytes are returned verbatim with the sniffed MIME | `view_image.rs:35`, `view_image.rs:560-567` |
| Resize | Lanczos3 `resize_exact`, aspect preserved by longest edge, output dims floored at 1 px | `view_image.rs:626-634` |
| Output format choice | driven by the **decoded** colour type, not the input MIME: alpha → PNG, else JPEG q85 | `view_image.rs:640-654` |
| Second pass | if still over 3 MiB, one retry at 75 % of target (floored at 64 px); still over → error | `view_image.rs:586-596` |
| Relay-media auth gate | header attached only when scheme is http/https **and** path starts with `/media/` **and** host and effective port both equal `BUZZ_RELAY_URL`'s | `view_image.rs:236-247` |
| Token shape | Blossom BUD-01 kind-24242 event with `t=get`, `expiration = now + 600 s`, `server=<authority>`; no `x` tag | `view_image.rs:53`, `view_image.rs:252-274` |
| Redirect policy | when a token is attached, redirects are disabled entirely because reqwest does not treat `x-auth-tag` as sensitive | `view_image.rs:327-333` |
| Fail-open auth | missing/invalid `BUZZ_PRIVATE_KEY` or a signing error degrades to an unsigned fetch with a `tracing::warn!` | `view_image.rs:286-318` |
| 401/403 messaging | unauthenticated 401/403 produces a distinct error naming `BUZZ_PRIVATE_KEY` and `BUZZ_RELAY_URL` | `view_image.rs:347-361` |

#### `todo`, `_Stop`, `_PostCompact`

| Rule | Behaviour | Evidence |
|---|---|---|
| Read vs write | `todos` omitted or explicitly `null` → read; array → full replacement | `todo.rs:71-94`, test `explicit_null_is_read` (`todo.rs:400-412`) |
| Item cap | `MAX_ITEMS = 50` | `todo.rs:20`, `todo.rs:147-149` |
| Text length | `MAX_TEXT_CHARS = 200`, measured **after trim**, in `chars()` not bytes | `todo.rs:21`, `todo.rs:154-159` |
| Empty text | rejected after trim | `todo.rs:151-153` |
| Character allow-list | rejects all control chars, all whitespace except ASCII space, U+200B–200F, U+202A–202E, U+2060–206F, U+FEFF — spoofing defence for the rendered list | `todo.rs:126-144`, `todo.rs:160-166` |
| Duplicates | duplicate text after trim is rejected, because silent-removal diffing has no ids to disambiguate | `todo.rs:167-181` |
| Normalisation | text is trimmed on store so diffing and rendering share one canonical form | `todo.rs:75-81` |
| Silent-removal warning | any `!done` item in the old list whose text is absent from the new list triggers a `⚠️` block; removing a `done` item does not | `todo.rs:183-218` |
| Atomicity | validation, mutation, warning computation, and rendering all occur under one lock hold | `todo.rs:58-64`, `todo.rs:78-92` |
| Lock poisoning | `PoisonError::into_inner()` — a poisoned mutex is used anyway rather than panicking | `todo.rs:59-62` (same pattern at `shell.rs:66-69`, `shell.rs:972-975`) |
| Render format | `[x]`/`[ ] {1-based index}. {text}`, with `"  ← next"` appended to the first open item | `todo.rs:219-239` |
| `_Stop` semantics | empty string when the list is empty or fully done; otherwise `"You have open todo items. Keep working."` plus the rendering | `todo.rs:99-112` |
| `_PostCompact` semantics | empty string when the list is empty; otherwise `"# Todo List\n"` plus the rendering | `todo.rs:113-124` |

#### `rg` personality

| Rule | Behaviour | Evidence |
|---|---|---|
| Delegation first | if a real `rg` is found on a self-filtered PATH, args are passed through **unmodified** and its exit code is propagated | `rg.rs:11-29` |
| Self-recursion guard | `clean_path` drops every PATH entry whose `rg` canonicalizes to this binary | `rg.rs:31-47` |
| PATH parsing | hardcoded `split(':')` in both `clean_path` (`rg.rs:34`) and `which_rg` (`rg.rs:50`), and the probe name is bare `"rg"` with no `.exe` (`rg.rs:39`, `rg.rs:54`) — so on Windows the system-`rg` path effectively never resolves and the fallback always runs |
| Executable check | Unix requires mode `& 0o111`; non-Unix returns `true` unconditionally | `rg.rs:62-73` |
| Fallback flag allow-list | only `--files`, `-n/--line-number`, `-i/--ignore-case`, `-l/--files-with-matches`, `-C/--context`, `-g/--glob`, `--`; every other `-`-prefixed token errors with exit 2 | `rg.rs:87-135` |
| Fallback match semantics | literal substring (`line.contains(needle)`), lowercased on both sides for `-i` — **not** regex, diverging from real `rg` | `rg.rs:198-204`, `rg.rs:289-296` |
| Context cap | `-C` value clamped to `MAX_CONTEXT = 100` | `rg.rs:8`, `rg.rs:99-103` |
| Glob engine | hand-rolled `*`/`?` matcher tried against both the file name and the full path; no `**`, no character classes, no brace expansion | `rg.rs:388-434` |
| Output caps | `MAX_OUTPUT_BYTES = 50 KiB` / `MAX_OUTPUT_LINES = 2000`; on breach the sink latches `capped`, logs a `tracing::warn!`, and silently stops printing — **no truncation marker reaches stdout** | `rg.rs:6-7`, `rg.rs:137-166` |
| Long-line skip | a line exceeding `MAX_LINE_BYTES = 1 MiB` aborts the whole file scan | `rg.rs:5`, `rg.rs:228-261`, `rg.rs:283-287` |
| Non-UTF-8 files | a decode failure aborts the file scan and returns whatever was already found | `rg.rs:252-259`, `rg.rs:284-286` |
| Fallback walk exclusions | skips dot-prefixed names, all symlinks, and the fixed set `target`, `node_modules`, `dist`, `build`; depth capped at `MAX_WALK_DEPTH = 50`. Does **not** read `.gitignore` | `rg.rs:345-386` |
| Exit codes | `0` found, `1` not found, `2` bad args | `rg.rs:168-227` |

#### `tree` personality

| Rule | Behaviour | Evidence |
|---|---|---|
| Depth | default and hard cap both `MAX_WALK_DEPTH = 50`; requested depth is `min`-clamped, never rejected | `tree.rs:8`, `tree.rs:138`, `tree.rs:167` |
| Argument rules | one positional path only; a second positional or a second path after `--` errors "multiple paths not supported"; unknown flags error; exit 2 | `tree.rs:137-168` |
| Root validation | non-directory root → `"tree: not a directory"` + exit 2 | `tree.rs:26-29` |
| Ignore semantics | uses `ignore::WalkBuilder` with `git_ignore`, `git_exclude`, `git_global`, `ignore` all on, `require_git(false)`, and `hidden(true)` — unlike the `rg` fallback, gitignore rules apply | `tree.rs:41-50` |
| Ordering | `sort_by_file_name` for deterministic output | `tree.rs:49` |
| Annotation | every file gets `[line-count]`; every non-leaf directory gets a subtree total; the root line carries the grand total | `tree.rs:53-117` |
| Depth-boundary directories | directories at exactly `max_depth` are marked `leaf` and get **no** count annotation | `tree.rs:80-86`, `tree.rs:69-74` |
| Line counting | files over `MAX_FILE_BYTES = 10 MiB`, or unreadable, count as `0` — silently, with no marker | `tree.rs:9`, `tree.rs:170-184` |
| Output caps | line budget `MAX_OUTPUT_LINES - 1 = 1999`, byte budget `50 KiB`; both emit an explicit `[truncated]` line | `tree.rs:38`, `tree.rs:76-79`, `tree.rs:120-134` |
| Write failure | a failed `writeln!` returns exit `0`, so a broken pipe looks like success | `tree.rs:106-108`, `tree.rs:126-128` |
