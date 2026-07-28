## Module: buzz-dev-mcp (`crates/buzz-dev-mcp`)
### Aspect: Features

#### Tool inventory — registered vs defined

Seven tools are annotated `#[tool(...)]` inside the single `#[tool_router] impl
DevMcp` block (`crates/buzz-dev-mcp/src/lib.rs:30-125`), and `Self::tool_router()`
is installed into `DevMcp` at construction (`lib.rs:32-38`). There are **no
defined-but-unregistered tools** — every `#[tool]` method sits inside the router
block, and there is no second router or conditional registration anywhere in the
crate.

| Tool | Registered | Implementation | Purpose |
|---|---|---|---|
| `shell` | yes (`lib.rs:40-50`) | `shell.rs:130-323` | run a command string through an OS shell, ephemeral process per call |
| `read_file` | yes (`lib.rs:52-61`) | `read_file.rs:23-63` | line-numbered windowed text read |
| `view_image` | yes (`lib.rs:63-72`) | `view_image.rs:88-107` | load an image from path/URL/data-URL as an MCP image block |
| `str_replace` | yes (`lib.rs:74-83`) | `str_replace.rs:25-106` | atomic find-and-replace returning a unified diff |
| `todo` | yes (`lib.rs:85-98`) | `todo.rs:71-94` | read or replace the in-process session task list |
| `_Stop` | yes (`lib.rs:101-110`) | `todo.rs:99-112` | lifecycle hook returning an objection when items are open |
| `_PostCompact` | yes (`lib.rs:114-123`) | `todo.rs:113-124` | lifecycle hook re-emitting todo state after history compaction |

Tool count is bounded downstream, not here: `buzz-agent` caps a session at
`MAX_TOOLS_PER_SESSION = 128` and `MAX_MCP_SERVERS = 16`
(`crates/buzz-agent/src/mcp.rs:23`, `crates/buzz-agent/src/mcp.rs:26`).

#### Multicall personalities

The same binary serves six roles selected by `argv[0]` (`lib.rs:138-171`). Five
are made reachable to shell children through symlinks in the shim tempdir
(`shim.rs:31-40`).

| Personality | Provided by | Nature |
|---|---|---|
| `buzz-dev-mcp` (or any unmatched name) | `lib.rs:173-186` | MCP server over stdio |
| `rg` | `rg.rs:11-16` | delegate to system ripgrep, else a built-in substring-search fallback |
| `tree` | `tree.rs:18-135` | gitignore-aware directory listing with line counts |
| `buzz` | `buzz_cli::run_from_args` (`lib.rs:168-171`) | the full Buzz relay CLI, bundled in-process |
| `git-credential-nostr` | `git_credential_nostr::run` (`lib.rs:151`) | git credential helper for Nostr-authed push/fetch |
| `git-sign-nostr` | `git_sign_nostr::run` (`lib.rs:152`) | git object signing with a Nostr key |

The dispatch is ordered so that `rg`, `tree`, and the two git helpers exit before
any tokio runtime, tracing subscriber, or allocation beyond argv parsing is set up
(`lib.rs:146-153`).

#### Session bootstrap

`SharedState::new` builds an `instructions` string once and serves it as the MCP
server's `instructions` field (`shell.rs:40-63`, `lib.rs:128-135`). It contains
the working directory, a detected stack fingerprint, the resolved shell name with
the `BUZZ_SHELL` override hint, and an instruction to pass `workdir` per call
instead of using `cd` (`shell.rs:75-92`).

Stack detection is a flat existence check for nine marker files in the cwd —
`Cargo.toml`, `package.json`, `go.mod`, `pyproject.toml`, `requirements.txt`,
`Gemfile`, `pom.xml`, `build.gradle`, `build.gradle.kts` — sorted and
comma-joined, or `"unknown"` (`shell.rs:94-117`). It does not recurse.

A conditional extra line, `"Buzz relay configured. Run `buzz --help` …"`, appears
only when **both** `BUZZ_RELAY_URL` and `BUZZ_PRIVATE_KEY` are present in the
environment (`shell.rs:77-83`) — making the instructions string an observable
signal of whether a private key was injected.

#### Ephemeral git identity for shell children

When `NOSTR_PRIVATE_KEY` is present at startup, `Shim::install` provisions a
complete git identity for every subsequent `shell` call without writing to any
user config file (`shim.rs:51-68`, `shim.rs:178-216`). Ten settings are passed as
`GIT_CONFIG_KEY_n`/`GIT_CONFIG_VALUE_n` pairs:

| Setting | Value |
|---|---|
| `user.name` | the derived `npub` (`shim.rs:181`) |
| `user.email` | `<pubkey_hex>@<relay_host>`, falling back to `<pubkey_hex>@buzz` for localhost/127.* or an unset relay (`shim.rs:154-172`) |
| `credential.helper` | `nostr` (`shim.rs:187`) |
| `credential.useHttpPath` | `true` — required for NIP-98 against the full repo-root URL (`shim.rs:188-191`) |
| `nostr.keyfile` | the `0600` keyfile path (`shim.rs:191`) |
| `gpg.format` | `x509` (`shim.rs:192`) |
| `gpg.x509.program` | `git-sign-nostr` (`shim.rs:193`) |
| `commit.gpgSign` / `tag.gpgSign` | `true` (`shim.rs:194-195`) |
| `user.signingkey` | pubkey hex (`shim.rs:196`) |

The index base composes with any pre-existing `GIT_CONFIG_COUNT` rather than
clobbering it (`shim.rs:199-215`).

#### Output-artifact escape hatch

Truncated `shell` output is not lost: the captured bytes (up to 10 MiB) are
written to `{session}/artifacts/{callid:06}.{stdout|stderr}.txt` and the absolute
path is returned in the JSON body, so the agent can page through the full log with
`read_file` (`shell.rs:914-928`, `shell.rs:309-320`). The last 8 files are kept
(`shell.rs:971-981`).

#### Cancellation support

`shell` receives the `rmcp` `RequestContext` and threads `context.ct` into
`shell::run` (`lib.rs:44-50`), which races the child against the cancellation
token in a `biased` `tokio::select!` (`shell.rs:218-239`). This is exercised
end-to-end from the agent side by
`cancel_kills_inflight_tool_via_mcp_notification`
(`crates/buzz-agent/tests/regressions.rs:1572-1600`), which asserts a `sleep 60`
tool call completes in under 5 s and the process is actually dead. That test
skips itself when the `buzz-dev-mcp` binary has not been built
(`crates/buzz-agent/tests/regressions.rs:1592-1600`).
