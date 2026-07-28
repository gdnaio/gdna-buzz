## Module: buzz-dev-mcp (`crates/buzz-dev-mcp`)
### Aspect: Configuration

There is no config file, no CLI flag, and no `clap` parser in MCP-server mode. All
configuration is environment variables read at startup or per call, plus the
process's `argv[0]` and current working directory.

#### Environment variables read by this crate

| Variable | Default | Parse site | Effect |
|---|---|---|---|
| `BUZZ_SHELL` | unset → `bash` | `shell.rs:366` (Unix), `shell.rs:411` (Windows) | selects the shell binary. A value with more than one path component (or a root) must exist as a file, else it is ignored and the resolver falls through; a bare name is searched on PATH. On Windows this branch deliberately skips the `%SystemRoot%` exclusion so `cmd`/`powershell` resolve (`shell.rs:411-423`). The resolved stem also drives the flag: `cmd`→`/C`, `powershell`/`pwsh`→`-Command`, else `-c` (`shell.rs:336-348`) |
| `GIT_BASH` | unset | `shell.rs:424` | Windows-only legacy override; must be an existing file |
| `PATH` | inherited | `shim.rs:42-49` (rebuild), `shell.rs:377` (Unix `BUZZ_SHELL` bare-name scan), `rg.rs:32` (`clean_path`) | the shim dir is prepended with `std::env::join_paths` and the result is set as the child's `PATH` (`shell.rs:169`). `rg.rs` re-reads the raw `PATH` and splits it on hardcoded `':'` (`rg.rs:34`, `rg.rs:50`) |
| `NOSTR_PRIVATE_KEY` | unset | `shim.rs:54` | if set and parseable, written to a `0600` keyfile and used to build ten `GIT_CONFIG_*` settings; then **removed from the process env unconditionally** (`shim.rs:51-68`). An unparseable value logs a warning to stderr and disables git auth/signing (`shim.rs:90-99`) |
| `BUZZ_PRIVATE_KEY` | unset | `shell.rs:78` (presence only), `view_image.rs:299` (value) | presence gates the "Buzz relay configured" bootstrap line; value is parsed as `nostr::Keys` to sign Blossom `t=get` media-read tokens. Also inherited verbatim by every `shell` child (`shell.rs:171`) |
| `BUZZ_RELAY_URL` | unset | `shell.rs:78` (presence), `shim.rs:155` (host extraction), `view_image.rs:294` (URL parse) | presence gates the bootstrap hint; host becomes the `user.email` domain unless it is empty/localhost/`127.*` (`shim.rs:154-172`); parsed URL is the authority the relay-media auth gate compares against (`view_image.rs:236-247`) |
| `BUZZ_AUTH_TAG` | unset | `view_image.rs:341` | when a relay-media token is attached and the value is non-blank after trim, sent as the `x-auth-tag` header (`view_image.rs:341-345`) |
| `GIT_CONFIG_COUNT` | `0` | `shim.rs:200-204` | existing count is used as the index base so the shim's ten entries append rather than clobber |
| `SystemRoot` | unset | `shell.rs:431` | Windows-only: PATH entries under it are excluded from the implicit `bash.exe` scan (WSL avoidance) |
| `ProgramFiles`, `ProgramFiles(x86)`, `LocalAppData` | unset | `shell.rs:544-546` | Windows-only: standard Git-for-Windows install-location probes |

Per-call inputs that behave as configuration: `workdir` (`shell.rs:145-149`,
`read_file.rs:20-21`, `str_replace.rs:22-23`, `view_image.rs:74-76`) and
`timeout_ms` (`shell.rs:141-144`).

Implicit configuration from process state: `std::env::current_dir()` becomes
`SharedState.cwd`, the default workspace root for every tool (`lib.rs:179-181`);
`argv[0]`'s file stem selects the multicall personality (`lib.rs:139-153`);
`std::env::current_exe()` is the symlink target for the shim
(`shim.rs:29`, `rg.rs:19-21`).

#### Sandbox-root configuration

**There is none.** `SharedState.cwd` (`shell.rs:27`) is only the default value
used when `workdir` is omitted; it is not a boundary. No env var, flag, or
parameter restricts which directories the tools may touch — see the Security
aspect and the explicit statement at `paths.rs:3-6`.

#### Compile-time configuration

| Setting | Value | Site |
|---|---|---|
| `unsafe` policy | `forbid` on non-Windows, `deny` on Windows | `lib.rs:1-2` |
| tokio runtime | `new_current_thread().enable_all()` | `lib.rs:159-162` |
| tracing | stderr writer, ANSI disabled | `lib.rs:174-177` |
| TLS provider | `rustls::crypto::ring` installed as default (repeat installs ignored) | `lib.rs:164-166`, `Cargo.toml:34-36` |
| `image` codecs | `jpeg`, `png`, `gif`, `webp` only — `default-features = false` | `Cargo.toml:40` |
| `reqwest` | rustls, no default features | `Cargo.toml:93` (workspace) |
| Windows `windows-sys` features | `Win32_Foundation`, `Win32_Security`, `Win32_System_JobObjects`, `Win32_System_Registry`, `Win32_System_Threading` | `Cargo.toml:47-52` |
| `[lib]` / `[[bin]]` | lib `buzz_dev_mcp` at `src/lib.rs`, bin `buzz-dev-mcp` at `src/main.rs` | `Cargo.toml:8-15` |

Everything else that could plausibly be configurable is a hardcoded `const`. The
following have **no** environment or flag override:

| Constant | Value | Site |
|---|---|---|
| `DEFAULT_TIMEOUT_MS` / `MAX_TIMEOUT_MS` | 120 s / 600 s | `shell.rs:16-17` |
| `MAX_COMMAND_BYTES` | 1 MB | `shell.rs:18` |
| `CAPTURE_CAP` / `MAX_BYTES` / `MAX_LINES` / `TAIL_BYTES` | 10 MiB / 50 KiB / 2000 / 8 KiB | `shell.rs:19-22` |
| `ARTIFACT_RING_SIZE` / `READ_CHUNK` | 8 / 16 KiB | `shell.rs:23-24` |
| `MAX_FILE_BYTES` | 10 MiB | `paths.rs:15` |
| `DEFAULT_LIMIT` (read_file) | 2000 lines | `read_file.rs:6` |
| `MAX_INPUT_BYTES` / `HINT_SCAN_LINE_LIMIT` / `MAX_DIFF_BYTES` | 1 MiB / 200 / 64 KiB | `str_replace.rs:9-10`, `str_replace.rs:140` |
| `MAX_ITEMS` / `MAX_TEXT_CHARS` | 50 / 200 | `todo.rs:20-21` |
| `MAX_SOURCE_BYTES` / `MAX_FINAL_RAW_BYTES` / `DEFAULT_MAX_DIM` / `MIN_MAX_DIM` / `MAX_MAX_DIM` / `MAX_PIXELS` / `MAX_DECODER_ALLOC` / `FETCH_TIMEOUT` / `MEDIA_GET_AUTH_EXPIRY_SECS` | 20 MiB / 3 MiB / 1568 / 64 / 2048 / 64 Mpx / 256 MiB / 10 s / 600 s | `view_image.rs:31-53` |
| `rg` caps | 1 MiB line / 50 KiB / 2000 lines / context 100 / depth 50 | `rg.rs:5-9` |
| `tree` caps | 50 KiB / 2000 lines / depth 50 / 10 MiB per-file | `tree.rs:6-9` |
| `rg` fallback ignore list | `target`, `node_modules`, `dist`, `build` | `rg.rs:375` |
| `detect_stack` marker list | 9 filenames | `shell.rs:95-105` |
| shim symlink names | `rg`, `tree`, `buzz`, `git-credential-nostr`, `git-sign-nostr` | `shim.rs:33-39` |

#### Parsed-but-unused / partially-used variables

- `BUZZ_AUTH_TAG` is read at exactly one site (`view_image.rs:341`) and applied
  only to relay-hosted `/media/` fetches. It is not used by any other tool in this
  crate even though `buzz-acp` injects it (`crates/buzz-acp/src/lib.rs:4170-4180`)
  and `buzz-agent` passes it through (`crates/buzz-agent/src/mcp.rs:63`). It reaches
  the bundled `buzz` CLI and `git-credential-nostr` only by env inheritance.
- `BUZZ_PRIVATE_KEY` and `BUZZ_RELAY_URL` are read for presence at `shell.rs:78`
  purely to decide whether to append one sentence to the bootstrap string; the
  values are discarded there.
- `GIT_CONFIG_COUNT` composition (`shim.rs:199-204`) is documented as reachable
  only when dev-mcp is run directly, because `buzz-agent` clears the env
  (`shim.rs:174-177`) — dead in the primary launch path.

#### `.env.example` coverage

`.env.example` documents `BUZZ_PRIVATE_KEY` (`:125`), `BUZZ_RELAY_URL` (`:130`),
and the `BUZZ_ACP_PRIVATE_KEY` → `BUZZ_PRIVATE_KEY` rename (`:226`).

Missing from `.env.example` entirely:

| Variable | Why it matters |
|---|---|
| `BUZZ_SHELL` | the only supported way to change the interpreter, and the documented Windows escape hatch in the resolver's own error text (`shell.rs:455-457`); also part of `buzz-agent`'s `WINDOWS_SHELL_RESOLUTION_ENV` public contract (`crates/buzz-agent/src/lib.rs:22-30`) |
| `GIT_BASH` | Windows fallback override (`shell.rs:424`) |
| `NOSTR_PRIVATE_KEY` | drives the entire ephemeral git-identity feature (`shim.rs:54`) and is on `buzz-agent`'s passthrough list (`crates/buzz-agent/src/mcp.rs:60`) |
| `BUZZ_AUTH_TAG` | referenced only indirectly at `.env.example:226`; never given its own entry despite being injected by `buzz-acp` |
