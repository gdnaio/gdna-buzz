## Module: buzz-acp — harness core & orchestration (`crates/buzz-acp/src`)
### Aspect: Conventions

#### Module layout

`lib.rs:1-15` declares the whole crate: `#![deny(unsafe_code)]` then eleven private modules and one re-export.

```
acp  config  engram_fetch  filter  observer
pool  pool_lifecycle  queue  relay  setup_mode  usage
```

No module is `pub`. Everything crosses seams via `pub(crate)` / `pub` items visible only inside the crate. File sizes (`wc -l crates/buzz-acp/src/*.rs`):

| File | Lines |
|---|---|
| `lib.rs` | 6,570 |
| `relay.rs` | 6,233 |
| `pool.rs` | 5,620 |
| `queue.rs` | 4,759 |
| `acp.rs` | 3,717 |
| `config.rs` | 2,709 |
| `setup_mode.rs` | 1,135 |
| `usage.rs` | 892 |
| `filter.rs` | 787 |
| `pool_lifecycle.rs` | 312 |
| `engram_fetch.rs` | 248 |
| `observer.rs` | 166 |
| `main.rs` | 3 |

The desktop and mobile trees enforce a 1,000-line-per-file ceiling (`AGENTS.md § Mobile App`, `desktop`/`web` equivalents). No such guard exists for Rust crates, and the five largest files here are 3.7×–6.6× that ceiling.

`main.rs` is a pure three-line delegate (`main.rs:1-3`) — the whole binary lives in the library, so everything is unit-testable in-crate.

#### Error handling

Three layers, applied consistently:

| Layer | Type | Sites |
|---|---|---|
| Module-internal | `thiserror` enums — `acp::AcpError` (`acp.rs:79`), `relay::RelayError` (`relay.rs:437`), `pool::SteerError` (`pool.rs:337`) | |
| Orchestration | `anyhow::Result` with `map_err(\|e\| anyhow!("<context>: {e}"))` | `lib.rs:1290`, `1345`, `1360`, `1428`, `1440`, `3857`, `3878` |
| Subcommands | `eprintln!` + `std::process::exit(1)` | `lib.rs:3904-3916`, `3952-3966`, `3974-3979`, `3986-3999`, `4020-4023`, `4034-4041` |

The subcommand layer discards `anyhow` context and collapses every failure to exit code 1, unlike `buzz-cli`'s documented 0/1/2/3/4/5 scheme (`AGENTS.md § Agent CLI`).

`unsafe`: none. `#![deny(unsafe_code)]` at `lib.rs:1`. `largest_shrinkable_leaf` explicitly does two passes to satisfy the borrow checker "without unsafe" (`lib.rs:696-701`).

`unwrap()` / `expect()` in production paths — 4 total, all `expect` with justifying comments:

| Site | Call |
|---|---|
| `lib.rs:1243` | `.expect("failed to install rustls crypto provider")` |
| `lib.rs:1645` | `.expect("SIGTERM handler")` |
| `lib.rs:2548` | `.expect("successful wake stores a ready pool")` |
| `lib.rs:4167` | `.expect("secret key bech32 encoding should never fail")` |

The remaining 46 of the 50 `unwrap`/`expect` occurrences are inside `#[cfg(test)]` modules. `lib.rs:4161-4164` documents why the bech32 panic is preferred over silent fallback. `lib.rs:2548` is reachable if `PoolLifecycle::complete_wake` and `take_ready` ever disagree — an invariant panic in a non-test path.

There are three `debug_assert` invariants: `lib.rs:3043`, `lib.rs:2531`, and the `try_send` contract comment at `lib.rs:1183-1187`.

#### Async patterns

- Single `#[tokio::main]` runtime (`lib.rs:1238-1239`); `run()` stays sync only so `config::propagate_legacy_env_vars()` can call `std::env::set_var` before worker threads exist (documented `lib.rs:1226-1231`, a Rust 2024 soundness requirement).
- One giant `tokio::select!` `biased;` (`lib.rs:1823`) inside the main `loop` (`lib.rs:1707`). Ordering is exploited: pool results first, then panics, steer acks, wake results, retry sleep, observer control, relay events, heartbeats, presence, typing, shutdown.
- Empty-collection spin guards on every `join_next()` arm: `if !join_set.is_empty()` (`lib.rs:1836`), `if !wake_tasks.is_empty()` (`lib.rs:1862`). The comment at `lib.rs:1833-1835` names the tight-spin failure mode.
- `std::future::pending()` is the idiom for a disabled arm: `heartbeat` (`lib.rs:2282`), `presence_heartbeat` (`lib.rs:2305`), `typing_refresh` (`lib.rs:2325`), observer control (`lib.rs:1885`), retry deadline (`lib.rs:1859`).
- Borrow splitting: `pool.rx_and_join_set()` (`lib.rs:1710`) yields two disjoint `&mut`s; arms that need `pool` whole end the split with an explicit `let _ = result_rx;` (`lib.rs:1888`, `1906`, `2285`, `2308`, `2328`).
- Maintenance runs at the **top** of the loop on an `Instant` check, not as a `select!` arm, specifically so `biased` cannot starve it (comment `lib.rs:1605-1607`, code `lib.rs:1744`).
- Fire-and-forget `tokio::spawn` for cosmetic side effects: 👀 add (`lib.rs:2206-2211`), 👀 cleanup (`lib.rs:1938-1943`), failure notices (`lib.rs:3025-3029`), presence heartbeat (`lib.rs:2314-2318`), steer ack watcher (`lib.rs:2853-2860`). Presence is the only one whose handle is retained and aborted (`lib.rs:2311-2313`, `2676-2678`).
- RAII is used where a dropped task would corrupt state: `RespawnGuard` (`lib.rs:1172-1231`).

#### Channel sizing rationale is documented at every declaration

| Channel | Line | Capacity | Stated reason |
|---|---|---|---|
| `respawn_tx` | `lib.rs:1613` | `config.agents` | at most one respawn per slot in flight |
| `wake_tx` | `lib.rs:1616` | 1 | one wake attempt at a time |
| `steer_ack_tx` | `lib.rs:1617-1629` | unbounded | losing an ack would leak a withheld event until `IN_FLIGHT_DEADLINE_SECS` |
| `shutdown_tx` | `lib.rs:1632` | `watch` | multi-consumer broadcast |
| control signal | `lib.rs:2961` | `oneshot` | one signal per turn |
| steer request | `lib.rs:2933` | `mpsc(1)` | one in-flight steer per turn |

#### Logging / tracing discipline

`tracing` macros with structured fields, initialized once at `lib.rs:1276-1281` (`EnvFilter`, `.compact()`, default `buzz_acp=info`).

Level usage is consistent:

| Level | Use | Examples |
|---|---|---|
| `error!` | invariant violations and terminal states | `lib.rs:1190`, `1213`, `2299`, `2372`, `3276`, `3423`, `3466` |
| `warn!` | recoverable degradation | `lib.rs:139`, `282`, `483`, `801`, `846`, `853`, `863`, `1379`, `1477`, `1486` |
| `info!` | lifecycle | `lib.rs:131`, `1284`, `1356`, `1362`, `1372`, `1438`, `1488`, `2775` |
| `debug!` | per-event decisions | `lib.rs:355`, `1871`, `1885`, `2028`, `2161`, `2178`, `2271`, `2905`, `2911` |

Machine-readable snake_case event names appear as bare messages for grep-ability: `heartbeat_skipped_events`, `heartbeat_skipped_busy`, `heartbeat_skipped_pool_not_ready` (`lib.rs:2271`, `2280`, `2270`), `pool_exhausted` (`lib.rs:2905`), `agent_claimed` (`lib.rs:2911`), `dispatch_pending` (`lib.rs:2994`), `agent_returned` (`lib.rs:3237`), `agent_pool_ready` (`lib.rs:3844`), `heartbeat_fired` (`lib.rs:3585`).

Author pubkeys are logged at debug on gate rejection (`lib.rs:2161-2168`) and warn on non-owner control frames (`lib.rs:851-858`). No secret material is logged — `config.summary()` (`config.rs:1012-1040`) prints `relay`, `pubkey`, commands, and timeouts, never `keys.secret_key()`.

Diagnostic context deliberately attached to fatal-outcome logs: `configured_model` and `pid` (`lib.rs:3230-3236`, `3288-3296`, `3348-3354`, `3374-3381`) — the comment at `lib.rs:3188-3195` explains that `configured_model` is spawn-time config and may legitimately differ from a `session/set_model` override, and is kept only to identify stale orphans.

#### Comment style

The file leans heavily on long prose block comments that carry design decisions and rejected alternatives, often 20+ lines:

- steer ack semantics with per-variant rationale and attribution: `lib.rs:2417-2478` (61 lines)
- `fit_observer_event_to_budget` doc including a termination proof and an explicit "double-serialize accepted" tradeoff note: `lib.rs:634-658`
- accepted membership race with the cost of the correct fix spelled out: `lib.rs:1664-1680`
- `is_auth_error` classification rationale with false-positive analysis: `lib.rs:2989-3002`
- two-layer membership dedup and why `<` not `<=`: `lib.rs:1863-1876`
- `try_native_steer` caller invariants as a contract: `lib.rs:2779-2802`

Three sites carry explicit cross-file "edit one, review the other" coupling notes: `lib.rs:539-543` (↔ `buzz-core/src/observer.rs:25`), `lib.rs:2812-2819` (↔ `queue::native_steer_framing`), `lib.rs:3048-3051` (requeue-before-mark_complete ordering).

#### Test organization

Tests are **in-file** `#[cfg(test)]` modules, not `tests/`. Eleven modules in `lib.rs`:

| Module | Line |
|---|---|
| `agent_draft_prompt_tests` | 3589 |
| `heartbeat_base_prompt_tests` | 4187 |
| `owner_control_command_tests` | 4219 |
| `owner_cache_tests` | 4347 |
| `author_gate_tests` | 4370 |
| `observer_snapshot_race_tests` | 4742 |
| `observer_publish_pacer_tests` | 4807 |
| `observer_chunk_coalescer_tests` | 4841 |
| `build_mcp_servers_tests` | 4939 |
| `error_outcome_emission_tests` | 5085 |
| `observer_payload_trim_tests` | 6324 |

Test code occupies roughly 2,406 of 6,570 lines (~37 %): lines 3588–3608 and 4186–6570, with production code at 1–3587 and 3609–4185.

The whole `crates/buzz-acp/tests/` directory holds one file, `pool_lifecycle_state.rs`, 159 bytes — so there is essentially no integration-level test surface for the crate.

Conventions inside tests:

- `#[tokio::test(start_paused = true)]` for time-dependent logic (`lib.rs:4809`, `4823`, `4752`).
- Real subprocesses as inert fixtures: `AcpClient::spawn("cat", …)` (`lib.rs:5183`) and `agent_command: "true"` (`lib.rs:5104`), with comments explaining why (`lib.rs:5100-5103`, `5179-5182`).
- A real `tokio::net::TcpListener` HTTP stub for lazy channel resolution (`lib.rs:4626-4685`) rather than a mocking framework.
- `static ENV_LOCK: Mutex<()>` to serialize env-var tests, since env is process-global (`lib.rs:4941-4942`).
- Test-only accessors on production types: `queue.set_retry_count_for_test` (`queue.rs:610`, used `lib.rs:5721`) and `RelayEventPublisher::test_pair` (`lib.rs:4756`).
- Two full `Config` literals are hand-maintained in tests (`lib.rs:4946-4990`, `lib.rs:5097-5151`) — every new `Config` field requires editing both.
- `#[allow(clippy::too_many_arguments)]` on three orchestration functions (`lib.rs:3033`, `3401`, `3500`); `handle_prompt_result` takes 11 parameters.

Notable: `error_outcome_emission_tests` documents a *structural* invariant — `handle_prompt_result` takes no relay handle, so it physically cannot post a channel message; re-adding one would break compilation of these tests (`lib.rs:5086-5096`).
