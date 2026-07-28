## Module: buzz-workflow (`crates/buzz-workflow`)

### Aspect: Configuration

---

### 1. Environment variables

All three are read inside `add_reaction_impl` and only exist under `#[cfg(feature = "reqwest")]`. There is no config struct wiring, no `.env` loading, and no other `std::env::var` call in the crate (verified by grep).

| Variable | Read at | Default | Purpose |
|---|---|---|---|
| `BUZZ_RELAY_BASE_URL` | `executor.rs:886-887` | `"http://localhost:3000"` | Base URL for the reaction POST |
| `BUZZ_API_TOKEN` | `executor.rs:898` | none | If present, sent as `Authorization: Bearer {token}` |
| `BUZZ_RELAY_PUBKEY` | `executor.rs:900` | none | Fallback when `BUZZ_API_TOKEN` is absent, sent as `X-Pubkey` |

If neither auth variable is set, the request goes out unauthenticated (`executor.rs:898-902`).

---

### 2. Cargo features

| Feature | Definition | Effect | Enabled by |
|---|---|---|---|
| `reqwest` | `reqwest = ["dep:reqwest"]` (`Cargo.toml:28-29`), dependency declared `optional = true` (`Cargo.toml:27`) | Compiles the real HTTP paths: `check_ssrf` (`executor.rs:744-776`), `WEBHOOK_MAX_RESPONSE_BYTES` (`executor.rs:777-778`), `call_webhook_impl` (`executor.rs:780-866`), `shared_http_client` (`executor.rs:870-882`), `add_reaction_impl` (`executor.rs:884-930`). Disabled ⇒ `call_webhook` returns `{status:0, body:null, skipped:true}` (`executor.rs:636-647`) and `add_reaction` returns `{added:false, skipped:true}` (`executor.rs:606-616`) | `buzz-relay` (`crates/buzz-relay/Cargo.toml:63`: `features = ["reqwest"]`). `buzz-admin` depends without it (`crates/buzz-admin/Cargo.toml:21`) |

No default features are declared, so a bare `cargo build -p buzz-workflow` compiles the skip-stub variants.

---

### 3. Runtime configuration struct — `WorkflowConfig`

`lib.rs:57-71`. Every construction site in the repo uses `WorkflowConfig::default()`; no environment variable or CLI flag overrides these values.

| Field | Type | Default | Consumed at |
|---|---|---|---|
| `max_concurrent` | `usize` | `100` (`lib.rs:68`) | semaphore permits, `max(1)`-clamped (`lib.rs:110-111`) |
| `default_timeout_secs` | `u64` | `300` (`lib.rs:69`) | per-step timeout fallback when `step.timeout_secs` is absent (`executor.rs:1133-1135`) |

Construction sites: `crates/buzz-relay/src/main.rs:389-390` (production), `crates/buzz-relay/src/state.rs:1274-1276`, plus ~8 relay test fixtures — all `::default()`.

---

### 4. Compile-time constants (complete list, with values)

| Constant | Value | Where | Meaning |
|---|---|---|---|
| `EVAL_TIMEOUT` | `Duration::from_millis(100)` | `executor.rs:342` | Wall-clock cap on one evalexpr evaluation (does not cancel the blocking thread — `executor.rs:358-360`) |
| `MAX_EXPR_LEN` | `4096` (bytes) | `executor.rs:362` (fn-local const in `evaluate_condition`) | Rejects long condition expressions before evaluation |
| `MAX_DELAY_SECS` | `270` (seconds) | `executor.rs:676` (fn-local const in `dispatch_action`) | Upper bound on the `delay` action; chosen to stay under the 300 s default step timeout |
| `WEBHOOK_MAX_RESPONSE_BYTES` | `1024 * 1024` = 1 048 576 (1 MiB) | `executor.rs:778` (`#[cfg(feature = "reqwest")]`) | Outbound webhook response body cap |

---

### 5. Hard-coded values that behave like configuration (not named constants)

| Value | Where | Meaning |
|---|---|---|
| Semaphore permits = `config.max_concurrent.max(1)` → 100 by default | `lib.rs:110-111` | Concurrency admission; `try_acquire()` so overflow is rejected, not queued (`executor.rs:975`, `executor.rs:1025`) |
| Workflow cache `max_capacity(10_000)` | `lib.rs:119` | Max cached `(community_id, channel_id)` entries |
| Workflow cache `time_to_live(10s)` | `lib.rs:120` | TTL, documented as matching the relay's other moka caches and bounding cross-pod staleness (`lib.rs:92-103`) |
| Scheduler tick `sleep(Duration::from_secs(60))` | `lib.rs:432` | Cron/interval loop period; sleep happens before the first scan |
| Cron match window `60` (passed as `window_secs`) | `lib.rs:484` → `lib.rs:688-706` | Drift-tolerant lookback window for cron matching |
| Minimum allowed interval = `60` seconds | `schema.rs:220-224` | Definition-time rejection of sub-minute intervals, justified by the 60 s tick |
| HTTP client timeout `Duration::from_secs(10)` | `executor.rs:807` (per-request webhook client), `executor.rs:876` (shared reaction client) | Total request timeout |
| Redirect policy `Policy::none()` | `executor.rs:809` | Webhook redirects disabled |
| Default HTTP method `"POST"` | `executor.rs:621` | `call_webhook` when `method` is omitted |
| Default port fallback `80` | `executor.rs:798` | Used when `port_or_known_default()` yields `None` |
| Approval timeout default string `"24h"` | `executor.rs:653` | **Log-only.** The string is never parsed into a duration and no expiry is persisted, because no approval record is created (`executor.rs:663`). The documented "Defaults to 24h" in the schema (`schema.rs:138-140`) has no runtime effect today |
| Step-ID limits: non-empty, ≤ `64` chars, `[A-Za-z0-9_]` | `schema.rs:168-172` | Definition-time validation protecting evalexpr variable names |
| Cron field normalization: 5→7, 6→7 fields | `schema.rs:250-257` | `0 {expr} *` / `{expr} *` |
| Reaction `e`-tag length check `64` hex chars | `lib.rs:924-928` | Distinguishes hex event IDs from UUID channel refs when resolving `trigger.message_id` |

---

### 6. Not configurable

- The four evalexpr helper functions are registered unconditionally with no allow/deny list (`executor.rs:236-283`).
- The SSRF blocklist is compiled into `buzz-core` (`crates/buzz-core/src/network.rs:25-54`); there is no allowlist/bypass hook for internal webhook targets.
- Scheduler tick period, cache TTL/capacity, delay cap, expression cap, evaluation timeout, HTTP timeouts, and the response cap are all literals — none are surfaced through `WorkflowConfig` or environment variables.
