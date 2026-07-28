## Module: buzz-acp — harness core & orchestration (`crates/buzz-acp/src`)
### Aspect: API Surface

#### Public Rust surface: two items

```
pub use usage::TurnUsage;      // lib.rs:15
pub fn run() -> Result<()>     // lib.rs:1233
```

That is the entire `pub` surface of the 6,570-line crate root (`grep -n '^pub ' lib.rs` → lines 15 and 1233). Everything else is private or `pub(crate)`. The one `pub(crate)` item is `is_dm_channel` at `lib.rs:273-286`, exposed only so the in-file `author_gate_tests` module can drive it.

`pub fn run()` at `lib.rs:1233` **has no doc comment**. The block above it (`lib.rs:1226-1231`) is `//`, not `///`, and is separated by a blank line — so the single exported function of the crate violates the repo's "new public API must have doc comments" rule (`AGENTS.md § Quality Gates`).

`TurnUsage` is re-exported but never referenced by `lib.rs` itself; it exists for `sprig`/downstream consumers.

#### Binary surface

`src/main.rs` is three lines and delegates entirely:

```rust
fn main() -> anyhow::Result<()> { buzz_acp::run() }   // main.rs:1-3
```

`run()` does sync legacy-env propagation then enters `#[tokio::main] async fn tokio_main()` (`lib.rs:1233-1239`).

#### CLI subcommands — hand-rolled dispatch, not clap subcommands

`is_subcommand(name)` (`lib.rs:58-60`) reads `std::env::args().nth(1)` and compares strings *before* any clap parsing. Each match then rebuilds argv with index 1 filtered out and re-parses with a dedicated `Parser` (`lib.rs:1245-1279`).

| Subcommand | Dispatch | Handler | Parser | Timeout |
|---|---|---|---|---|
| `models` | `lib.rs:1245-1254` | `run_models` `lib.rs:4005-4139` | `ModelsArgs` | `MODELS_TIMEOUT` 10 s (`lib.rs:63`) |
| `auth-methods` | `lib.rs:1256-1264` | `run_auth_methods` `lib.rs:3899-3945` | `AuthMethodsArgs` | 10 s |
| `authenticate` | `lib.rs:1266-1274` | `run_authenticate` `lib.rs:3947-4003` | `AuthenticateArgs` | `AUTHENTICATE_TIMEOUT` 600 s (`lib.rs:67`) |
| *(default)* | `lib.rs:1276+` | main harness loop | `CliArgs` via `Config::from_cli()` (`lib.rs:1290`) | — |

Documented constraint at `lib.rs:52-56`: the subcommand **must** be argv[1]. `buzz-acp --verbose models` is not supported. None of the three subcommands is documented in `crates/buzz-acp/README.md` — the README's only CLI content is flags for the default path.

All three subcommands use `std::process::exit(1)` on failure rather than returning `Err` (`lib.rs:3904`, `3910`, `3915`, `3952`, `3958`, `3964`, `3974`, `3986`, `3992`, `4020`, `4034`, `4039`), so `anyhow` context is discarded and the exit code is always 1.

`run_models --json` emits a stable envelope (`lib.rs:4084-4098`) with `agent.{name,version}`, `stable.configOptions`, `unstable.{currentModelId,availableModels}`; the comment at `lib.rs:4083` names its consumer as "Phase 3 `get_agent_models`".

#### Nostr event kinds published by `lib.rs`

| Kind | Constant | Publish site | Transport |
|---|---|---|---|
| 20001 | `KIND_PRESENCE_UPDATE` (`buzz-core/src/kind.rs:327`) | `publish_presence` `lib.rs:77-91` | WebSocket only — comment at `lib.rs:71-72` states ephemeral kinds are rejected by the HTTP bridge |
| 20002 | `KIND_TYPING_INDICATOR` (`kind.rs:331`) | built by `relay.build_typing_event` and fired at `lib.rs:2332-2340` on a 3 s tick (`lib.rs:1593-1599`) | WebSocket, `try_publish_event` (non-blocking) |
| 24200 | `KIND_AGENT_OBSERVER_FRAME` (`kind.rs:333`) | `publish_relay_observer_event` `lib.rs:790-833` via `buzz_sdk::build_agent_observer_frame` (`lib.rs:810`) | WebSocket |

Presence content is a bare status string `"online"` / `"offline"` with no tags (`lib.rs:85-87`); the initial `online` is published *after* channel subscriptions so it doubles as a readiness signal (`lib.rs:1511-1515`), and `offline` is best-effort with a 2 s timeout at shutdown (`lib.rs:2679-2692`). A 60 s presence heartbeat re-publishes `online` (`lib.rs:1583-1591`, fired `lib.rs:2301-2320`).

#### Nostr event kinds consumed

Default subscription kinds when `subscribe_mode == Mentions` (`lib.rs:1442-1450`):

| Kind | Constant | Value |
|---|---|---|
| `KIND_STREAM_MESSAGE` | `kind.rs:343` | 9 |
| `KIND_WORKFLOW_APPROVAL_REQUESTED` | `kind.rs:442` | 46010 |
| `KIND_STREAM_REMINDER` | `kind.rs:355` | 40007 |

Overridable wholesale by `config.kinds_override` (`--kinds` / `BUZZ_ACP_KINDS`). `SubscribeMode::All` uses `kinds_override.unwrap_or_default()` — i.e. an **empty** kinds list (`lib.rs:1456`), which per `AGENTS.md § Common Gotchas #2` trips the relay p-gate.

Control-plane kinds handled inline in the main loop:

| Kind | Constant | Value | Handling |
|---|---|---|---|
| member added | `KIND_MEMBER_ADDED_NOTIFICATION` | 44100 (`kind.rs:396`) | dedup → subscribe new channel (`lib.rs:1891-1907`) |
| member removed | `KIND_MEMBER_REMOVED_NOTIFICATION` | 44101 (`kind.rs:400`) | unsubscribe, drain queue, invalidate sessions, remove 👀 (`lib.rs:1908-1946`) |
| observer control | `KIND_AGENT_OBSERVER_FRAME` | 24200 | `handle_relay_observer_control_event` `lib.rs:837-893` |

Every kind integer resolves through `buzz_core::kind` constants (`lib.rs:23-26`, `lib.rs:82`). **No bare kind literal exists in production code** — `grep -n 'Kind::Custom([0-9]'` finds `Kind::Custom(9)` only inside `#[cfg(test)]` fixtures (`lib.rs:5442`, `5548`, `5670`, `5763`, `5829`, `5832`, `6157`, `6242`).

#### In-band owner control commands (text protocol over kind 9)

`is_owner_control_command` (`lib.rs:2718-2727`) requires kind == 9 **and** `content.trim() == command` **and** a `p` tag equal to this agent's pubkey.

| Command | Site | Effect |
|---|---|---|
| `!shutdown` | `lib.rs:2033-2059` | `shutdown_tx.send(())` → graceful exit |
| `!cancel` | `lib.rs:2064-2092` | `ControlSignal::Cancel` to in-flight task; no-op when idle |
| `!rotate` | `lib.rs:2102-2133` | cancel + rotate if in flight, else invalidate idle session |

All three fall through to normal prompt handling when the sender is not the owner (`lib.rs:2056-2058`, `2090-2091`, `2130-2131`) — the string is then delivered to the agent as ordinary content.

#### Encrypted observer control frames (owner → harness)

`handle_relay_observer_control_event` (`lib.rs:837-893`) accepts two `type` values after signature, sender, and freshness checks:

| `type` | Handler | Result statuses emitted |
|---|---|---|
| `cancel_turn` | `lib.rs:895-938` | `sent`, `no_active_turn` |
| `switch_model` | `lib.rs:940-1005` | `sent`, `turn_ending`, `switched`, `unsupported_model`, `no_active_turn` |

Unknown `type` values are logged at debug and dropped (`lib.rs:888-891`). Outcomes are reported back as `control_result` observer events, not as JSON-RPC replies.
