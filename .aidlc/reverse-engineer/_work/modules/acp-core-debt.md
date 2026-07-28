## Module: buzz-acp — harness core & orchestration (`crates/buzz-acp/src`)
### Aspect: Technical Debt

#### File size

`lib.rs` is 6,570 lines / 266,642 bytes in a single file. Five of the crate's twelve source files exceed 3,700 lines:

| File | Lines | vs. the 1,000-line ceiling enforced on desktop/web/mobile |
|---|---|---|
| `lib.rs` | 6,570 | 6.6× |
| `relay.rs` | 6,233 | 6.2× |
| `pool.rs` | 5,620 | 5.6× |
| `queue.rs` | 4,759 | 4.8× |
| `acp.rs` | 3,717 | 3.7× |

`AGENTS.md § Mobile App` mandates a hard 1,000-line ceiling enforced by `mobile/scripts/check-file-sizes.mjs` via `just mobile-check`, mirroring desktop and web guards, and instructs "split the file — never bump the limit". No equivalent guard exists for Rust crates, and `buzz-acp` is where that gap is most visible.

Within `lib.rs`, `tokio_main` alone spans **1,462 lines** (`lib.rs:1239-2700`) as one function containing a `select!` with twelve arms, a nested enum declaration (`PoolEvent`, `lib.rs:1700-1705`), and the entire shutdown sequence. `handle_prompt_result` is 366 lines (`lib.rs:3034-3399`) with 11 parameters behind `#[allow(clippy::too_many_arguments)]` (`lib.rs:3033`) — the same `allow` appears on `recover_panicked_agent` (`lib.rs:3401`) and `drain_ready_join_results` (`lib.rs:3500`).

#### Duplication with `buzz-ws-client`

`buzz-acp` does not depend on `buzz-ws-client` (`Cargo.toml:19-22` lists only `buzz-core`, `buzz-sdk`, `buzz-persona`). It pulls `tokio-tungstenite` directly (`Cargo.toml:31`) and reimplements the full client in `relay.rs` — 6,233 lines including its own `RelayMessage` enum (`relay.rs:471`), `do_connect` (`relay.rs:3825`), `wait_for_auth_challenge` (`relay.rs:3865`), `send_auth_response` (`relay.rs:3433`), reconnect state machine (`relay.rs:2893`, `3022`), terminal-error classification (`relay.rs:3657-3785`), and two-generation dedup (`relay.rs:935`). It also diverges functionally: `relay.rs` re-authenticates on a mid-session AUTH challenge where `buzz-ws-client` only records it, so the two clients now have different NIP-42 semantics.

`lib.rs`'s share of that divergence is the routing knowledge encoded in comments rather than types: `publish_presence`'s doc comment (`lib.rs:69-75`) has to explain that ephemeral kinds 20000–29999 are rejected by the HTTP bridge and must go over WS, and the typing arm has to use `try_publish_event` because a blocking publish would stall the main loop during reconnect (`lib.rs:2329-2331`, citing issue #35). None of this is enforced by the type system — a future contributor can call `publish_event` with an ephemeral kind and it will compile.

#### Zero-usage dependency

`buzz-persona = { path = "../buzz-persona" }` at `Cargo.toml:22`. `grep -rn 'buzz_persona\|buzz-persona' crates/buzz-acp/` returns exactly one line: the manifest. It is also the only internal dep declared by path rather than `workspace = true`.

`Config::persona_env_vars` (`config.rs:533-535`) is a plain `Vec<(String,String)>` built in `config.rs:945-999` and threaded through `lib.rs:1762`, `3488`, `3666`, `3733` into `AcpClient::spawn` — no call into the crate. The dependency adds compile time and a version-coupling edge for nothing.

#### Dead / unreachable code

| Item | Evidence |
|---|---|
| `BUZZ_API_TOKEN` | Propagated from its legacy alias at `config.rs:718` but never read anywhere in `crates/buzz-acp/src/`. Documented as a real setting in `README.md § Core`. |
| `Config::allowed_respond_to` | `config.rs:460`; referenced only by `config.summary()` (`config.rs:1019-1025`). No gate reads it. |
| `pub use usage::TurnUsage` | `lib.rs:15`; `TurnUsage` appears nowhere else in `lib.rs`. |
| `RespawnResult` doc comment | `lib.rs:1142-1145` documents a third tuple element `supports_goose_steer` that is "always `true`". The tuple is `(AcpClient, u32, String)` — the described field does not exist. Stale doc. |
| `SubscribeMode::All` with no `--kinds` | `lib.rs:1456` produces an empty kinds vector, which per `AGENTS.md § Common Gotchas #2` trips the relay p-gate and 403s. `--subscribe all` is documented in `README.md § Forum Channels` only ever paired with an explicit `--kinds`. Nothing warns. |
| Owner re-resolution | `OwnerCache.pubkey` is set once at construction (`lib.rs:158-163`) with no setter. `respond-to=owner-only` + startup resolution failure = permanently deaf agent, and `--relay-observer` silently off (`lib.rs:1421-1425`). |

#### TODO / FIXME inventory

`grep -cn -E 'TODO|FIXME|XXX|HACK' crates/buzz-acp/src/lib.rs` → **0**.

No inline debt markers at all. The debt is instead recorded as prose in long comment blocks, which is harder to inventory mechanically. Examples of accepted-limitation comments that would conventionally be TODOs:

| Accepted limitation | Site |
|---|---|
| Membership remove→re-add race can requeue a stale batch; fix needs per-channel epochs on `TaskMeta` + `PromptResult`, judged not worth it | `lib.rs:1670-1680` |
| Stale 👀 reactions after membership removal because the relay revokes access before emitting the notification; "fix belongs in the relay" | `lib.rs:2000-2006` |
| Fire-and-forget 👀 add can race `ReactionGuard` cleanup, leaving a cosmetic stale reaction | `lib.rs:2204-2208` |
| Native steer accepted but event already drained ⇒ possible duplicate delivery, logged as a warning that should never fire | `lib.rs:2840-2852` |
| `fit_observer_event_to_budget` accepts a redundant serialize on the common path because fixing it would require changing `buzz-core`'s `encrypt_observer_payload` signature | `lib.rs:651-658` |
| `desired_model` does not track `session/set_model`, so the harness's model view can be stale | `lib.rs:3188-3195` |
| Subcommands must be argv[1]; `buzz-acp --verbose models` unsupported | `lib.rs:52-56` |

#### Fragile coupling with no enforcement

| Coupling | Sites | Failure mode if broken |
|---|---|---|
| `OBSERVER_CHUNK_MAX_TEXT_BYTES` (60,000) must stay under `OBSERVER_MAX_PLAINTEXT_LEN` in `buzz-core/src/observer.rs:25` | `lib.rs:539-544` | frames silently dropped by `encrypt_observer_payload` instead of trimmed |
| Requeue must precede `mark_complete` or `retry_counts` is cleared and backoff/dead-lettering is defeated | `lib.rs:3048-3051`, mirrored `lib.rs:3427` | every retry restarts at attempt 1; infinite retry |
| Native steer framing must match cancel+merge framing | `lib.rs:2812-2819` ↔ `queue::native_steer_framing` | agent gets inconsistent orientation depending on transport |
| `crash_history.len()` must equal `config.agents`, not `pool.live_count()` | `lib.rs:1684-1695`, indexed `lib.rs:1774`, `3266`, `3465` | index-out-of-bounds panic |
| `respawn_tx` capacity must equal slot count for `RespawnGuard::send`'s `try_send` to be infallible | `lib.rs:1613`, `lib.rs:1183-1187` | slot lost forever (guard logs `error!` and falls through to `Drop`) |
| `is_auth_error` matches substrings of vendor CLI error text | `lib.rs:3003-3011` | an upstream wording change silently converts non-retryable auth failures back into 10 futile retries |
| `protocolVersion` missing ⇒ silently treated as legacy v1 | `lib.rs:3785`, `lib.rs:3864` | wrong prompt composition (`prepend_base_for_legacy`) |
| `agentInfo`/`serverInfo` precedence differs between `normalized_agent_name` (`lib.rs:3688`) and `run_models` (`lib.rs:4062`) | | agent that sets both reports two different names |

None of these couplings has a dedicated regression test. The closest is `hard_timeout_recently_active_budget_exhausted_reports_dead_lettered` (`lib.rs:5727`), which exercises dead-lettering *through* the requeue path but would still pass if the requeue/`mark_complete` order were inverted, since it pre-seeds `retry_counts` via `set_retry_count_for_test`. The constant coupling, the `crash_history` sizing invariant, and the framing-drift coupling rely on comments alone.

#### Doc drift

| Claim | Doc | Code |
|---|---|---|
| `BUZZ_ACP_IDLE_TIMEOUT` default `620` | `README.md § Core` | `DEFAULT_IDLE_TIMEOUT_SECS = 900` (`config.rs:27`) |
| `BUZZ_API_TOKEN` is a working setting | `README.md § Core` | never read (`config.rs:718` only) |
| Heartbeat calls `get_feed_actions()` / `get_feed_mentions()` | `README.md § Heartbeat Semantics` | prompt says `buzz feed get --types needs_action` / `--types mentions` (`lib.rs:3620-3628`) |
| Default kinds are "9, 46010, 40007" | `README.md § Forum Channels` | matches `lib.rs:1447-1449` ✓ |
| Typing indicators are Redis sorted sets (`ZADD buzz:typing:{channel_id}` + `ZREMRANGEBYSCORE` + `EXPIRE`) | `ARCHITECTURE.md:452-457` | typing is a kind 20002 ephemeral Nostr event published by the harness (`lib.rs:2332-2340` → `relay.rs:842-870`, `KIND_TYPING_INDICATOR` = 20002 at `buzz-core/src/kind.rs:331`). The ZADD/ZREM/EXPIRE design at `ARCHITECTURE.md:454-456` does not exist in `buzz-acp`; `ARCHITECTURE.md:824` separately and correctly describes 20002 delivery via fan-out + pub/sub, so the document contradicts itself. |
| Harness auto-injects `BUZZ_RELAY_URL` / `BUZZ_PRIVATE_KEY` / `BUZZ_AUTH_TAG` into managed agent subprocesses | `AGENTS.md § Agent CLI` | true but imprecise. The explicit injection at `lib.rs:4141-4184` targets the **MCP server** declaration inside `session/new`, delivered over the agent's stdin pipe — not the ACP child's environment. The ACP child receives them only by ordinary environment inheritance (`acp.rs:456-461`), and only if the harness itself was launched with them set. |
| `models` / `auth-methods` / `authenticate` subcommands | not documented anywhere | `lib.rs:1245-1274`, handlers `lib.rs:3899`, `3947`, `4005` |
| 19 env vars including `RESPOND_TO`, `PERMISSION_MODE`, `LAZY_POOL`, `AUTH_TAG` | absent from `.env.example` | declared in `config.rs` / read in `lib.rs` — see the Configuration aspect for the full list |
| `.env.example:152` documents `BUZZ_ACP_TURN_TIMEOUT` | | that flag is `hide = true` and deprecated (`config.rs:274`) |

#### Test coverage

`lib.rs` carries ~2,406 lines of tests across 11 in-file `#[cfg(test)]` modules (lines 3588–3608 and 4186–6570), against ~4,164 lines of production code — roughly 37 % test.

`crates/buzz-acp/tests/` holds a single 159-byte file, `pool_lifecycle_state.rs`. There is effectively **no integration test surface** for a crate that spawns subprocesses, speaks WebSocket + HTTP to a relay, and runs a twelve-arm event loop.

Covered well:

| Area | Tests |
|---|---|
| Author gate incl. DM hardening | `author_gate_tests` `lib.rs:4370-4740` — 14 cases including `test_dm_rejects_stranger_under_anyone`, `test_dm_nobody_rejects_even_owner`, `test_discovery_without_metadata_stays_fail_closed_at_author_gate` |
| Outcome → retry/dead-letter matrix | `error_outcome_emission_tests` `lib.rs:5085-6321` — one `turn_error` per error outcome, `hard_timeout_recently_active_requeue_success_reports_requeued_for_retry`, `hard_timeout_recently_active_budget_exhausted_reports_dead_lettered`, `auth_error_dead_letters_immediately_without_requeueing`, `cancel_drain_timeout_requeues_batch_and_does_not_return_agent` |
| Observer trimming | `observer_payload_trim_tests` `lib.rs:6324-6570` — 8 cases including UTF-8 boundary, stub termination, multi-block header survival |
| Observer pacing + snapshot/live race | `lib.rs:4742-4838` with `start_paused` virtual time |
| Chunk coalescing | `lib.rs:4841-4936` |
| MCP env construction | `build_mcp_servers_tests` `lib.rs:4939-5082` |
| Owner control command parsing + mode gate | `lib.rs:4219-4344` |

Untested in `lib.rs`:

| Gap | Why it matters |
|---|---|
| `SlotCircuit` state machine | No test for `record_crash`, `can_refill`, half-open pre-seeding, backoff math, or `mark_spawn_failed`. The breaker is the sole guard against respawn storms and its half-open pre-seed logic is duplicated in two methods (`lib.rs:1064-1074`, `1113-1131`). |
| `RespawnGuard::Drop` | The documented failure mode is "silently losing the slot forever" (`lib.rs:1168-1171`); nothing verifies `Drop` actually emits the fallback. |
| Steer ack decision table | The six-way `(release, drop, fallback)` matrix at `lib.rs:2474-2486` — including the `-32601` special case — is inline in the `select!` arm, not extracted into a testable function. `try_native_steer` has no test. |
| Membership dedup | Two-generation rotation, the strict-`<` watermark, and the drain/invalidate/un-react sequence (`lib.rs:1861-1949`) are all inline in the loop with no test. |
| Lazy-pool wake path | `PoolEvent::Wake`, stale-attempt discard (`lib.rs:2505-2509`), the busy-spin guard on `retry_at` (`lib.rs:1735-1742`), and the drain-not-abort shutdown (`lib.rs:2566-2603`) are untested. The single file in `tests/` is named for this area but is 159 bytes. |
| Shutdown sequence | Four-stage reaping (`lib.rs:2605-2672`) has no test; the 30 s grace and abort fallback are unverified. |
| `run_models` / `run_auth_methods` / `run_authenticate` | No tests; all three `std::process::exit(1)` on failure, which is itself untestable in-process. |
| `is_dm_channel` under lazy REST failure | Covered (`lib.rs:4720-4739`), but via a raw `TcpListener` stub — no coverage of a 5xx or a slow-loris response against the 2 s gate timeout. |
| `Config` literals | Two hand-maintained full-struct literals in tests (`lib.rs:4946-4990`, `lib.rs:5097-5151`) must be edited for every new field — a recurring merge-conflict and drift source. |

#### Other observations

- `Box::leak` on a user-supplied base prompt (`lib.rs:1545`) intentionally leaks the string for `'static` lifetime. Bounded (one per process) but a deliberate leak.
- Jitter for respawn backoff is derived from `SystemTime` subsec nanos (`lib.rs:1091-1097`), not a PRNG. Slots crashing within the same nanosecond window receive correlated jitter, weakening the thundering-herd protection the jitter exists to provide.
- `handle_prompt_result` recomputes `agent_index` twice (`lib.rs:3048` and `lib.rs:3191`) and `let mut hard_timeout_fate_suffix` threads a `&'static str` through six branches to reconstruct a message string 150 lines later (`lib.rs:3060`, consumed `lib.rs:3254`) — a sign the function wants splitting.
- Three `debug_assert`s (`lib.rs:2531`, `lib.rs:3043`, and the `try_send` contract) encode invariants that vanish in release builds.
