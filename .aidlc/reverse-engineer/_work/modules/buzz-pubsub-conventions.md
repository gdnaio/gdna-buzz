## Module: buzz-pubsub (`crates/buzz-pubsub`)
### Aspect: Conventions

---

### 1. Lint posture and repo-rule compliance

| Rule | Status | Evidence |
|---|---|---|
| No `unsafe` | Enforced at crate level | `#![deny(unsafe_code)]` `lib.rs:1`; zero `unsafe` blocks |
| Public API documented | Enforced at crate level | `#![warn(missing_docs)]` `lib.rs:2` — a `warn`, not `deny`, so a missing doc does not fail the build |
| No `unwrap()`/`expect()` in production paths | **Compliant** | Every `unwrap`/`expect` occurrence is inside `#[cfg(test)]` modules (`lib.rs:369-627`, `presence.rs:105-208`, `nip98_replay.rs:103-201`, `topic.rs:117-196`, `cache_invalidation.rs:183-250`, `conn_control.rs:169-228`, `subscriber.rs:190-204`). The single production `expect` is in `#[cfg(test)] mod test_util` (`lib.rs:374`) |
| Workspace-inherited manifest fields | Compliant | `version`/`edition`/`rust-version`/`license`/`repository` all `.workspace = true` (`Cargo.toml:3-7`) |
| Workspace dependency pinning | Compliant | All 13 deps use `{ workspace = true }` (`Cargo.toml:11-23`); no local version literals |
| No TODO/FIXME/HACK markers | Compliant | none present in any of the 10 files |

### 2. Error-handling conventions

- One `thiserror` enum per crate, `PubSubError` (`error.rs:5-28`), with `#[from]`
  conversions for the three foreign error types (`error.rs:8`, `:12`, `:16`) and a
  hand-written `From<broadcast::error::RecvError>` that maps lag and closure onto
  domain variants (`error.rs:31-38`).
- **Trait-imposed exception:** the two `buzz-auth` trait impls return
  `AuthError` instead, because the trait signatures demand it (`rate_limiter.rs:35`,
  `nip98_replay.rs:37`). Foreign errors are flattened into
  `AuthError::Internal(format!(...))` at `rate_limiter.rs:45`, `:52`, `:66` and
  `nip98_replay.rs:57`, `:79`, `:95` — a deliberate choice to keep the user-facing
  category string bounded, called out in a comment at `nip98_replay.rs:50-51`.
- **Resilience convention: never let one bad message kill a loop.** Every subscriber
  handles a malformed input with `tracing::warn!` + `continue`
  (`subscriber.rs:132-157`, `cache_invalidation.rs:144-165`,
  `conn_control.rs:135-156`).
- **Fire-and-forget convention:** where the DB is the durable backstop, send results
  are discarded with `let _ =` and the rationale is documented at the call site
  (`lib.rs:203-207`, `:236-243`; doc at `lib.rs:266-271`, `:285-291`).

### 3. Naming and structural conventions

- All Redis keys and channels are built through a helper, never inline-formatted at
  the call site; the prefix is a single constant `BUZZ_PREFIX` (`topic.rs:13`) reused
  by every module (`presence.rs:14`, `cache_invalidation.rs:20`, `conn_control.rs:23`).
- Key layout is uniformly `buzz:{community}:{kind}[:{id}]`. The one deliberate
  exception, IP rate limits, is documented as such (`rate_limiter.rs:84-85`).
- Channel-suffix constants are exported so producer and subscriber cannot drift:
  `CACHE_INVALIDATION_SUFFIX`/`_PATTERN` (`cache_invalidation.rs:23`, `:27`),
  `CONN_CONTROL_SUFFIX`/`_PATTERN` (`conn_control.rs:26`, `:30`).
- Every construct-a-key path takes `&TenantContext` rather than a raw
  `CommunityId`, so a caller cannot fabricate a tenant
  (`topic.rs:35`, `:103`, `:108`, `presence.rs:19`, `cache_invalidation.rs:30`,
  `conn_control.rs:33`).
- Paired parse/format functions live beside each other and are round-trip tested
  (`topic.rs:43`/`:53` tested `topic.rs:150-177`;
  `cache_invalidation.rs:30`/`:38` tested `:201-207`;
  `conn_control.rs:33`/`:38` tested `:183-188`).
- Builder pattern with a `DEFAULT_*` associated const for tunables
  (`lib.rs:82`, `:93-96`).
- Backoff constants are duplicated per module rather than shared:
  `BACKOFF_INITIAL_SECS`/`BACKOFF_MAX_SECS` appear three times
  (`subscriber.rs:16-19`, `cache_invalidation.rs:91-94`, `conn_control.rs:81-84`)
  with identical values.

### 4. Concurrency conventions

- `Arc<Self>` receivers for infinite background loops (`lib.rs:148`, `:165`, `:175`);
  `PubSubManager` is intentionally not `Clone` (`lib.rs:100`).
- Take-once initialisation guard: the `mpsc::Receiver` is stored as
  `Mutex<Option<..>>` and `take()`n so a second `run_subscriber` is a logged no-op
  rather than a panic or a silent double-consume (`lib.rs:107`, `:149-152`).
- Locks are held for the shortest possible span: the refcount mutations compute a
  boolean inside a block, then the lock drops before any `await` on the channel
  (`lib.rs:194-207`, `:217-231`) — avoids holding a `tokio::Mutex` across `await`.
- `tokio::select!` for multiplexing commands against the message stream, with an
  `else` arm for total-shutdown detection (`subscriber.rs:110-171`).

### 5. Documentation conventions

- Crate-level ASCII architecture diagram (`lib.rs:8-16`) plus explicit "why" notes
  for non-obvious choices: why the pub/sub connection is not pooled (`lib.rs:19-20`),
  why cache-invalidation and conn-control are separate channels
  (`conn_control.rs:12-18`), why topics are not an isolation boundary
  (`topic.rs:3-6`, `lib.rs:305-320`), why the TTL is 3× the heartbeat
  (`presence.rs:4-6`).
- Known-limitation callouts are inline rather than in a separate doc: the fixed-window
  2× burst warning (`rate_limiter.rs:9-10`), the "upgrade to sliding window" note
  (same lines), and the `⚠️` marker convention.
- Contract obligations are written into log messages, not just docs — e.g. "caller
  MUST fail closed" appears in the `warn!` payloads at `nip98_replay.rs:55`, `:76`.

### 6. Testing conventions

34 test functions across 8 of 10 files. 11 require live Redis and are gated
`#[ignore = "requires Redis"]` — a consistent, uniform gate string
(`lib.rs:400`, `:438`, `:476`, `:511`; `presence.rs:137`, `:160`, `:187`;
`nip98_replay.rs:128`, `:145`, `:165`, `:180`).

| File | Tests | Redis-gated |
|---|---|---|
| `lib.rs` | 6 | 4 |
| `topic.rs` | 6 | 0 |
| `conn_control.rs` | 6 | 0 |
| `cache_invalidation.rs` | 5 | 0 |
| `presence.rs` | 5 | 3 |
| `nip98_replay.rs` | 4 | 4 |
| `subscriber.rs` | 2 | 0 |
| `error.rs`, `publisher.rs`, `rate_limiter.rs` | 0 | — |

Conventions observed:
- A shared `#[cfg(test)] mod test_util` provides `make_test_pool()` (`lib.rs:369-377`)
  reused by `presence.rs:107`; `nip98_replay.rs` instead builds its own pool honouring
  a `REDIS_URL` env override (`nip98_replay.rs:110-116`) — an inconsistency in an
  otherwise uniform pattern.
- Every module defines an identical local `fn ctx(id, host)` helper
  (`lib.rs:388`, `topic.rs:119`, `presence.rs:119`, `cache_invalidation.rs:185`,
  `conn_control.rs:171`, and inline at `subscriber.rs:191`) — six copies of the same
  three-line fixture.
- Negative-path tables: malformed inputs are asserted as a `for` loop over a literal
  array (`topic.rs:179-195`, `cache_invalidation.rs:222-233`).
- Serde round-trip tests for every wire enum variant (`cache_invalidation.rs:235-249`,
  `conn_control.rs:202-207`, `:219-227`) plus a forward-compat test asserting an
  unknown `op` is rejected without poisoning subsequent messages
  (`conn_control.rs:209-217`).
- Intent-documenting test names and comments: `lib.rs:556-557` explains that the test
  exists to catch a channel-id-only keying bug; `nip98_replay.rs:184-190` explains why
  clamping beats propagating an error.
- `test_presence_set_and_get` is **duplicated** — defined at both `lib.rs:477` and
  `presence.rs:138` with overlapping assertions.

`rate_limiter.rs` has **zero tests** despite implementing a security control; its
behaviour is covered only indirectly through `buzz-relay/src/admission.rs`'s
`StubLimiter` (`admission.rs:65-90`), which stubs out the Redis logic entirely — so
the Lua script, the `count <= limit` boundary (`rate_limiter.rs:74`), and the TTL
repair path (`rate_limiter.rs:57-70`) are untested anywhere in the repo.
