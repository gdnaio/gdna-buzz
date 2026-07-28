## Module: buzz-pubsub (`crates/buzz-pubsub`)
### Aspect: Configuration

`buzz-pubsub` reads **no environment variables in any production path**. All
configuration is injected by the caller as constructor arguments. A repo-wide grep
for `env::var` inside `crates/buzz-pubsub/src/` returns exactly one hit, and it is
inside a `#[cfg(test)]` module.

---

### 1. Injected configuration

| Input | Type | Entry point | Supplied by |
|---|---|---|---|
| Redis URL (for pub/sub connections) | `String` | `PubSubManager::new` (`lib.rs:117`) → stored `lib.rs:105` | `buzz-relay/src/main.rs:344` passes `&config.redis_url` |
| Redis connection pool (for all commands) | `deadpool_redis::Pool` | `PubSubManager::new` (`lib.rs:117`), `RedisRateLimiter::new` (`rate_limiter.rs:94`), `RedisNip98ReplayGuard::new` (`nip98_replay.rs:29`) | `buzz-relay/src/state.rs:711-712`, `:717`; pool field `state.rs:494` |
| Unsubscribe debounce | `Duration` | `PubSubConfig::with_unsubscribe_debounce` (`lib.rs:93`) | **nobody** — see §3 |
| Rate-limit window + limit | `u64, u64` per call | `check_and_increment` args (`rate_limiter.rs:100-107`) | `buzz-relay/src/connection.rs:610-611`, `:628-646` from `state.auth.config().rate_limits` (`connection.rs:610`) |
| Replay TTL | `u64` per call, then clamped | `try_mark_in_scope` (`nip98_replay.rs:38`), clamp `:47` | `buzz-auth` constants `DEFAULT_REPLAY_TTL_SECS` / `MAX_REPLAY_TTL_SECS` (`nip98_replay.rs:11-12`) |

The Redis URL reaches the crate from `REDIS_URL`, resolved in the relay with a
default of `redis://localhost:6379` (`buzz-relay/src/config.rs:418-419`, field
declared `config.rs:60`). The crate never reads that variable itself.

Note the pool is **injected, not constructed** — so pool size, timeouts, and any TLS
settings are entirely the relay's concern; `buzz-pubsub` cannot influence them. Only
the three dedicated pub/sub sockets are opened from the crate's own stored URL
(`subscriber.rs:85-86`, `cache_invalidation.rs:135-136`, `conn_control.rs:126-127`),
each via `redis::Client::open` on every reconnect attempt.

### 2. Compile-time constants (the crate's real tunables)

| Constant | Value | Site | Configurable at runtime? |
|---|---|---|---|
| `PubSubConfig::DEFAULT_UNSUBSCRIBE_DEBOUNCE` | 500 ms | `lib.rs:82` | Only via the builder — unused in production |
| Event broadcast capacity | 4096 | `lib.rs:126` | No |
| Cache-invalidation broadcast capacity | 4096 | `lib.rs:127` | No |
| Conn-control broadcast capacity | 4096 | `lib.rs:128` | No |
| Subscription-command mpsc capacity | 4096 | `lib.rs:129` | No |
| `PRESENCE_TTL_SECS` | 90 | `presence.rs:16` | No |
| `BACKOFF_INITIAL_SECS` | 1 | `subscriber.rs:16`, `cache_invalidation.rs:91`, `conn_control.rs:81` | No — three separate copies |
| `BACKOFF_MAX_SECS` | 30 | `subscriber.rs:18`, `cache_invalidation.rs:93`, `conn_control.rs:83` | No — three separate copies |
| `BUZZ_PREFIX` | `"buzz"` | `topic.rs:13` | No |
| `CACHE_INVALIDATION_SUFFIX` / `_PATTERN` | `"cache-invalidate"` / `"buzz:*:cache-invalidate"` | `cache_invalidation.rs:23`, `:27` | No |
| `CONN_CONTROL_SUFFIX` / `_PATTERN` | `"conn-control"` / `"buzz:*:conn-control"` | `conn_control.rs:26`, `:30` | No |
| `RATE_LIMIT_SCRIPT` | Lua source | `rate_limiter.rs:24-31` | No |

Operationally significant: **presence TTL, all four channel capacities, and the
reconnect backoff envelope cannot be tuned without a recompile.** For a horizontally
scaled deployment these are exactly the knobs an operator would want — e.g. raising
broadcast capacity for pods with many concurrent sockets, or lengthening presence TTL
for clients on flaky networks.

### 3. The unsubscribe-debounce knob is unreachable

`PubSubConfig` exists (`lib.rs:73-97`) and `with_unsubscribe_debounce`
(`lib.rs:93`) works, but every production construction goes through
`PubSubManager::new` (`lib.rs:117-119`), which hardcodes
`PubSubConfig::new(redis_url)` and therefore always takes the 500 ms default.

All 11 `PubSubManager::new` call sites in the relay confirm this — the production one
is `buzz-relay/src/main.rs:344`; the other ten are test fixtures
(`state.rs:1267`, `api/operator.rs:591`, `api/bridge.rs:3309`, `api/invites.rs:536`,
`api/admin/mod.rs:348`, `api/media.rs:970`, `workflow_sink.rs:585`,
`handlers/identity_archive.rs:451`, `handlers/event.rs:1990`, `:2043`). Zero call
sites for `with_config` or `with_unsubscribe_debounce` exist outside this crate's own
tests (`lib.rs:514`, `:596`, `:621`).

### 4. Rate-limit configuration reaching this crate

The crate holds no limits of its own; window and limit arrive per call. The relay
derives them from `state.auth.config().rate_limits` (`buzz-relay/src/connection.rs:610`):

| Limit | Derivation | Site |
|---|---|---|
| WS admission | `human_ws_events_per_sec` converted to a 5 s window via `ws_admission_budget` → `(5, limit × 5)` | `buzz-relay/src/admission.rs:8`, `:35-40`; used `connection.rs:611-621` |
| Per-message (human) | `human_messages_per_min`, fixed 60 s window | `connection.rs:629-633`, `:641` |
| Per-message (agent) | `agent_standard_messages_per_min`, fixed 60 s window | `connection.rs:629-631`, `:641` |

The IP-connection limit path (`rate_limiter.rs:112-120`) receives no configuration
because it has no production caller.

### 5. Test-only configuration

| Item | Site | Behaviour |
|---|---|---|
| `REDIS_URL` | `nip98_replay.rs:110` | `std::env::var("REDIS_URL")` with fallback `redis://127.0.0.1:6379` — the only env read in the crate, and it is inside `#[cfg(test)] mod tests` |
| `test_util::make_test_pool` | `lib.rs:371-376` | Hardcodes `redis://127.0.0.1:6379`, ignoring `REDIS_URL` |
| `#[ignore = "requires Redis"]` | 11 tests (`lib.rs:400`, `:438`, `:476`, `:511`; `presence.rs:137`, `:160`, `:187`; `nip98_replay.rs:128`, `:145`, `:165`, `:180`) | Integration tests opt out of the default `cargo test` run |

Inconsistency: `nip98_replay.rs` honours `REDIS_URL` while `test_util::make_test_pool`
(`lib.rs:371-376`) and every test using it hardcode `127.0.0.1:6379`. Pointing CI at
a non-default Redis host would run some of this crate's tests against the wrong
endpoint. Also note the relay's default (`localhost`, `config.rs:419`) and the
crate's test default (`127.0.0.1`) differ in spelling, which matters on hosts where
`localhost` resolves to IPv6 first.

### 6. Not configured anywhere

No configuration exists for: Redis TLS or auth (must be embedded in the URL string),
Redis logical db index, `maxmemory`/eviction expectations for the replay seen-set,
pub/sub message size limits, or a cap on topics retained per connection.
