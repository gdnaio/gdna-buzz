## Module: buzz-admin, sprig & countdown-bot (`crates/buzz-admin`, `crates/sprig`, `examples/countdown-bot`)
### Aspect: API Surface

None of the three programs expose an HTTP or WebSocket server. `buzz-admin` is a CLI (subcommands), `sprig` is a multicall dispatcher (no subcommands of its own), and `countdown-bot` is a WebSocket *client* with no listening surface. This directly contradicts nothing in `AGENTS.md`, but readers should not expect any of the three to open a port.

#### buzz-admin — full CLI surface

Binary name: `buzz-admin` (`crates/buzz-admin/Cargo.toml:10-12`, `crates/buzz-admin/src/main.rs:34-35`, `#[command(name = "buzz-admin", ...)]`). Every subcommand and flag, verified directly against the `Command`/`ProductFeedbackCommand` enums:

| Subcommand | Flags | Default | Defined at | Handler |
|---|---|---|---|---|
| `add-member` | `--pubkey <STRING>` (required), `--role <STRING>` | role default `"member"` | `main.rs:44-58` | `cmd_add_member` (`:158-192`) |
| `remove-member` | `--pubkey <STRING>` (required), `--role <Option<STRING>>` | role omitted = remove regardless of role | `main.rs:59-72` | `cmd_remove_member` (`:194-251`) |
| `list-members` | none | — | `main.rs:74` | `cmd_list_members` (`:260-286`) |
| `generate-key` | none | — | `main.rs:76` | inline in `run()` (`:132-138`) |
| `migrate` | none | — | `main.rs:78` | inline in `run()` (`:139-143`) |
| `product-feedback list` | `--limit <u16>`, range `1..=1000` (`clap::value_parser!(u16).range(1..=1000)`) | `100` | `main.rs:80-105` | `cmd_list_product_feedback` (`:253-258`) |
| `reconcile-channels` | `--relay-key <Option<STRING>>` | falls back to `BUZZ_RELAY_PRIVATE_KEY` env, then an ephemeral generated key | `main.rs:89-95` | `reconcile_channels` (`:461-579`) |

Both `add-member` and `remove-member` accept `--pubkey` as **either** a bech32 `npub` or 64-char hex string — parsed by `parse_pubkey_hex` via `nostr::PublicKey::parse` (`main.rs:303-308`), matching the doc comment on `AddMember` (`:45-47`).

Global entry point: `#[tokio::main] async fn main()` (`main.rs:109-128`) installs the rustls ring crypto provider, calls `Cli::parse()`, dispatches to `run()`, and maps the returned `i32` to `std::process::exit`. `run()` (`main.rs:130-156`) is a single `match cli.command` over the seven variants above — there is no subcommand not listed in this table (verified by reading the full `Command` enum, `main.rs:41-96`).

**Exit codes** (only place they're defined is inline `Ok(N)`/`Err` returns; also documented in `NOSTR.md:243-251`):

| Code | Meaning | Source |
|---|---|---|
| 0 | success | every happy path |
| 1 | validation error (bad role, bad pubkey) | `cmd_add_member`/`cmd_remove_member` early returns, `main.rs:159-163,166-172,195-200,203-208` |
| 2 | remove: member not found | `RemoveResult::NotFound`, `main.rs:229-232` |
| 3 | remove: target is relay owner | `RemoveResult::IsOwner`, `main.rs:233-239` |
| 4 | remove: `--role` filter didn't match current role | `RemoveResult::RoleMismatch`, `main.rs:240-243` |
| 5 | DB/Redis/internal error | `RemoveResult`/`Err` fallthrough (`:244-247`) and the top-level `main()` catch-all (`:120-125`) |

**Contradiction vs the module inventory:** `.aidlc/reverse-engineer/modules.md` describes `buzz-admin` as having subcommands "membership, key-gen, migrate, reconcile" (singular "reconcile", no mention of `product-feedback`). The actual surface has **seven** leaf commands, not four categories, and includes an entire `product-feedback list` subcommand that the inventory line omits entirely. `ARCHITECTURE.md`'s own `buzz-admin` table (search for "### buzz-admin — Operator CLI") lists five subcommands (`add-member`, `remove-member`, `list-members`, `generate-key`, `reconcile-channels`) and likewise omits `migrate` and `product-feedback list` even though both exist in code (`main.rs:78,80-83`). Both docs undercount the real surface; see `ops-misc-debt.md`.

#### sprig — public interface is argv[0]-based, not subcommands

`sprig` is a single `[[bin]]` (`crates/sprig/Cargo.toml:12-14`) with **no library target** — `cargo doc -p sprig` would produce nothing beyond the binary's own `main`/`dispatch`/`print_usage` functions, none of which are `pub` (there is no `lib.rs` in `crates/sprig/src/`, confirmed by directory listing showing only `main.rs`). Its "API" is entirely behavioral: which of three downstream crates' `run()` function gets called, chosen by the lowercased `file_name()` of `argv[0]`:

| Invoked as (`argv[0]` basename, lowercased) | Calls | Defined at |
|---|---|---|
| `buzz-acp` | `buzz_acp::run() -> anyhow::Result<()>` (`crates/buzz-acp/src/lib.rs:1233`) | `sprig/src/main.rs:17` |
| `buzz-agent` | `buzz_agent::run() -> Result<(), Box<dyn std::error::Error>>` (`crates/buzz-agent/src/lib.rs:110`) | `sprig/src/main.rs:18` |
| `sprig` (no symlink, invoked directly) | handles `-V`/`--version`, `-h`/`--help`/no-args (prints usage, then errors with "invoke Sprig via a personality symlink" if truly no args), or errors on any other first argument | `sprig/src/main.rs:19-38` |
| anything else (including `rg`, `tree`, `buzz`, `git-credential-nostr`, `git-sign-nostr`, or an unrecognized name) | falls through to `buzz_dev_mcp::run() -> Result<(), Box<dyn std::error::Error>>` (`crates/buzz-dev-mcp/src/lib.rs:138`) | `sprig/src/main.rs:41` |

Confirmed exactly by reading `crates/sprig/src/main.rs:8-43` in full: the `match cmd.as_str()` has exactly four arms (`"buzz-acp"`, `"buzz-agent"`, `"sprig"`, and `_`) — there is no fifth arm for `"buzz-dev-mcp"` itself; invoking the binary as `buzz-dev-mcp` falls into the wildcard `_` arm and reaches `buzz_dev_mcp::run()` the same as any unrecognized name, since `buzz_dev_mcp::run()` performs its **own** second-level `argv[0]` dispatch (`crates/buzz-dev-mcp/src/lib.rs:138-146`) for `rg`/`tree`/`git-credential-nostr`/`git-sign-nostr`/`buzz`, and otherwise starts the MCP server over stdio (`lib.rs:158-172`). This two-level dispatch (sprig → buzz-dev-mcp → its own sub-dispatch) is accurately summarized by the code comment at `sprig/src/main.rs:39-40` and by `crates/sprig/Cargo.toml:10`'s description, and matches the task's framing exactly: sprig is confirmed to be the argv[0] multicall dispatcher its dependency list implies.

`Cargo.toml` confirms the three dispatch targets are real path dependencies, nothing else: `buzz-acp`, `buzz-agent`, `buzz-dev-mcp` (`crates/sprig/Cargo.toml:17-19`) — no other crate is depended on, so no fourth personality is possible without a code change.

#### countdown-bot — no server surface; it's purely a WebSocket client

`examples/countdown-bot` has one binary entry point, `#[tokio::main] async fn main() -> Result<()>` (`main.rs:35-73`), and exposes no HTTP/WS listener, no CLI flags, and no subcommands — confirmed by the absence of any `clap`/`argv` parsing anywhere in the file (only `std::env::var` calls, enumerated in `ops-misc-configuration.md`). Its "API" is the sequence of NIP-01 client operations it performs against the relay, in order:

| Step | Function | Purpose |
|---|---|---|
| 1 | `Config::from_env()` | read env vars, build bot keys and optional owner-attestation tag (`main.rs:83-121`) |
| 2 | `connect_and_authenticate` | open WS, wait for `["AUTH", challenge]`, sign and send `["AUTH", event]`, wait for `OK` (`main.rs:126-136`) |
| 3 | `publish_profile` | send kind:0 profile event, wait for `OK` (`main.rs:155-170`) |
| 4 | `announce_channel_membership` | best-effort self-add kind:9000 with `role=bot` (`main.rs:172-190`) |
| 5 | `subscribe_to_channel` | send `["REQ", "countdown-bot", <filter kind=9, #h=channel>]` (`main.rs:191-197`) |
| 6 | main loop | `tokio::select!` between `ctrl_c` and `ws.next()`, dispatching `Text`/`Ping`/`Close` frames (`main.rs:56-72`) |

The only "commands" it exposes to end users are the two chat-message-triggered ones described in `ops-misc-business-rules.md` (`!countdown N`, `!fib N`, and `@Countdown Bot countdown|fib N`) — these are **content-string** commands parsed out of incoming kind:9 events (`command_reply`/`mention_command_reply`, `main.rs:258-277`), not a CLI or RPC surface.

#### Test coverage of this aspect

`buzz-admin`'s CLI surface (all seven subcommands, all flag parsing, all exit codes) has zero automated test coverage — confirmed by `grep -rn "#\[test\]|#\[cfg(test)\]|#\[tokio::test\]" crates/buzz-admin/` returning no matches. `sprig`'s dispatch table likewise has no test coverage (same grep against `crates/sprig/` returns no matches). `countdown-bot`'s only tests exercise the plain-text reply functions (`main.rs:392-436`), not the WebSocket protocol steps above.
