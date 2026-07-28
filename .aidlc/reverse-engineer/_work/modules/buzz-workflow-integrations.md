## Module: buzz-workflow (`crates/buzz-workflow`)

### Aspect: Integrations

---

### 1. Dependency inventory (`Cargo.toml:10-29`)

| Dependency | Version / source | Used for | Evidence |
|---|---|---|---|
| `buzz-core` | workspace | `CommunityId`, `StoredEvent`, `kind::{event_kind_u32, is_workflow_execution_kind, KIND_REACTION, KIND_STREAM_MESSAGE, KIND_STREAM_MESSAGE_DIFF}`, `network::is_private_ip` | `lib.rs:47-48`, `lib.rs:956`, `executor.rs:768` |
| `buzz-db` | workspace | `Db` handle, `workflow::RunStatus`, `WorkflowRecord`, `DbError` conversion | `lib.rs:49-50`, `error.rs:62-66` |
| `hex` | workspace | encode `owner_pubkey` for attribution | `executor.rs:558` |
| `serde` / `serde_json` / `serde_yaml` | workspace | schema derive, canonical JSON, YAML parsing | `schema.rs:7`, `schema.rs:263-266` |
| `dashmap` | workspace | `last_fired` interval anchors | `lib.rs:53`, `lib.rs:87` |
| `moka` | workspace | `moka::sync::Cache` for the per-channel workflow lookup | `lib.rs:104`, `lib.rs:118-121` |
| `evalexpr` | `"11"` (pinned major, direct dep — not workspace) | `if:`/filter condition evaluation, custom function registration | `executor.rs:17`, `executor.rs:224-316` |
| `cron` | `"0.16"` (direct dep) | cron expression parse + `after()` iteration | `schema.rs:239`, `lib.rs:691-694` |
| `nostr` | workspace | `PublicKey::from_hex`, `ToBech32` for the `npub` filter; test event builders | `executor.rs:18`, `executor.rs:194-198` |
| `uuid` | workspace | run/workflow ids, approval token generation, channel override parsing | `executor.rs:698-700`, `executor.rs:481` |
| `chrono` | workspace | `DateTime<Utc>` scheduling arithmetic | `lib.rs:52`, `lib.rs:692` |
| `tokio` | workspace | `Semaphore`, `spawn`, `spawn_blocking`, `time::{sleep, timeout}` | `lib.rs:54`, `executor.rs:370-373`, `executor.rs:1137` |
| `tracing` | workspace | structured logs throughout | `executor.rs:20` |
| `thiserror` | workspace | `WorkflowError`, `ActionSinkError` | `error.rs:3`, `action_sink.rs:12` |
| `reqwest` | workspace, **optional** (`features.reqwest = ["dep:reqwest"]`) | `call_webhook`, `add_reaction` | `Cargo.toml:27-29`, `executor.rs:781`, `executor.rs:885` |

No `sqlx` dependency — all Postgres access is via `buzz_db::Db` methods.

---

### 2. Outbound HTTP — `call_webhook`

| Concern | Implementation | Line |
|---|---|---|
| Client crate | `reqwest::Client`, built **per request** (required because `.resolve()` pins DNS per host) | `executor.rs:800-812` |
| Total timeout | `Duration::from_secs(10)` | `executor.rs:807` |
| Redirect policy | `reqwest::redirect::Policy::none()` — explicitly disabled so a redirect cannot bypass the SSRF check | `executor.rs:809` |
| DNS pinning | `.resolve(host, SocketAddr::new(safe_ip, port))` with the IP validated by `check_ssrf`, closing the DNS-rebinding TOCTOU window | `executor.rs:810` |
| Method | `reqwest::Method::from_bytes(method)`; default `"POST"` when `method` is absent | `executor.rs:621`, `executor.rs:814-815` |
| Headers | caller-supplied map applied verbatim (names untemplated, values templated) | `executor.rs:819-823` |
| Body | raw string body, no content-type set automatically | `executor.rs:825-827` |
| Response cap | `WEBHOOK_MAX_RESPONSE_BYTES = 1024 * 1024` (1 MiB), enforced by chunked reads (`resp.chunk()`) with early abort | `executor.rs:778`, `executor.rs:838-860` |
| Response decode | `String::from_utf8_lossy` → `{status, body}` JSON | `executor.rs:862-865` |
| Port default | `port_or_known_default()`, fallback `80` — no scheme restriction, so plain `http://` targets are allowed despite the schema doc saying "must be a public HTTPS endpoint" (`schema.rs:120`) | `executor.rs:796-798` |
| Error mapping | every failure → `WorkflowError::WebhookError(String)` | `executor.rs:786`, `executor.rs:830-833`, `executor.rs:843-845` |

SSRF guard (`check_ssrf`, `executor.rs:745-776`): resolves `host:port` through the OS resolver on `spawn_blocking`; rejects on resolver error, on zero addresses, and if **any** returned address satisfies `buzz_core::network::is_private_ip`; returns `addrs[0]` for pinning. `is_private_ip` (`crates/buzz-core/src/network.rs:25-54`) covers IPv4 loopback/private/link-local/`0.0.0.0/8`/broadcast, CGNAT `100.64.0.0/10`, benchmarking `198.18.0.0/15`, and for IPv6 loopback/unspecified/`fc00::/7`/`fe80::/10`/`ff00::/8`/`2001:db8::/32`, with IPv4-mapped addresses recursed through the IPv4 rules.

---

### 3. Outbound HTTP — `add_reaction`

| Concern | Implementation | Line |
|---|---|---|
| Client | shared `LazyLock<reqwest::Client>` with a 10 s timeout, connection pooling retained | `executor.rs:871-882` |
| Target | `POST {BUZZ_RELAY_BASE_URL}/api/messages/{message_id}/reactions`, default base `http://localhost:3000` | `executor.rs:885-891` |
| Auth | `Authorization: Bearer {BUZZ_API_TOKEN}` if set, else `X-Pubkey: {BUZZ_RELAY_PUBKEY}` if set, else unauthenticated | `executor.rs:898-902` |
| SSRF guard | **none** on this path (no `check_ssrf`, no redirect policy, no response cap) — the URL comes from an env var rather than workflow YAML | `executor.rs:885-930` |
| Failure handling | non-2xx ⇒ `WebhookError` including the response body; body parse failure falls back to `{"raw": <text>}` | `executor.rs:911-919` |
| Reachability | the relay registers no `/api/messages/*` route (`crates/buzz-relay/src/router.rs:39-125`), so this call cannot succeed against the current relay |

---

### 4. Postgres via `buzz-db`

Access is exclusively through `buzz_db::Db` methods; no SQL is written in this crate.

| Db method | Purpose | Call site |
|---|---|---|
| `list_enabled_channel_workflows` | per-event workflow lookup (behind moka cache) | `lib.rs:301-306` |
| `list_all_enabled_workflows` | cron scan | `lib.rs:436` |
| `create_workflow_run` | run row for event + cron paths | `lib.rs:346-355`, `lib.rs:592-600` |
| `update_workflow_run` | `Running` / `Completed` / `Failed` transitions and trace writes | `executor.rs:982-991`, `executor.rs:1044-1053`, `lib.rs:201-215`, `lib.rs:220-238`, `lib.rs:244-261` |
| `get_workflow_run` | resolve `workflow_id` for `send_message`; read existing trace on resume | `executor.rs:537-543`, `executor.rs:1034` |
| `get_workflow` | `channel_id` + `owner_pubkey` for `send_message` | `executor.rs:546-552` |
| `claim_scheduled_workflow_fire` | cross-pod at-most-once cron claim | `lib.rs:547-568` |
| `latest_scheduled_workflow_fire` | restart anchor for interval schedules | `lib.rs:500-517` |
| `attach_scheduled_workflow_run` | best-effort claim→run link for audit | `lib.rs:617-628` |

DB errors convert via `From<buzz_db::error::DbError> for WorkflowError` → `Database(String)` (`error.rs:62-66`). In `finalize_run` DB failures are logged only (`lib.rs:207-213`, `:231-236`, `:256-260`); in `execute_run`/`execute_from_step` the initial `Running` write failure aborts the run (`executor.rs:992-999`, `executor.rs:1054-1061`).

---

### 5. Relay integration (inbound)

| Integration point | Relay side | Line |
|---|---|---|
| Engine construction | `WorkflowEngine::new(db, WorkflowConfig::default())` | `crates/buzz-relay/src/main.rs:389-390`, `crates/buzz-relay/src/state.rs:1274-1276` |
| Side-effect sink | `RelayActionSink` registered via `set_action_sink` | `crates/buzz-relay/src/main.rs:594-595`, `crates/buzz-relay/src/workflow_sink.rs:13,159` |
| Scheduler | `tokio::spawn(async move { wf_cron.run().await })`, started only after the sink is wired | `crates/buzz-relay/src/main.rs:597-599` |
| Event hook | `workflow_engine.on_event(community, &stored_event)` spawned from the post-store fan-out | `crates/buzz-relay/src/handlers/event.rs:496-534` |
| Definition ingest | `WorkflowEngine::parse_yaml(&event.content)` in the workflow upsert command | `crates/buzz-relay/src/handlers/command_executor.rs:684` |
| Cache invalidation | `invalidate_channel_workflows` on upsert and on NIP-09 deletion | `crates/buzz-relay/src/handlers/command_executor.rs:787`, `crates/buzz-relay/src/handlers/side_effects.rs:2018,2039` |
| Manual trigger (kind-command) | ownership check, run creation, then `execute_from_step(..., 0, None)` + `finalize_run` | `crates/buzz-relay/src/handlers/command_executor.rs:826-936` |
| Inbound webhook | `POST /hooks/{id}` → host-bound tenant, `TriggerDef::Webhook` check, secret verification, body fields → `webhook_fields`, then `execute_from_step(..., 0, None)` | `crates/buzz-relay/src/router.rs:120`, `crates/buzz-relay/src/api/bridge.rs:1759-1893` |
| Approval grant/deny | relay handlers look up approvals by stored hash and call `resume_workflow` → `execute_from_step` | `crates/buzz-relay/src/handlers/command_executor.rs:1009-1169`, `:1244-1320` |
| Metrics | `buzz_workflow_runs_total{trigger,community}` incremented by the relay after a successful `on_event` | `crates/buzz-relay/src/handlers/event.rs:526-533` |

---

### 6. Error handling posture

| Boundary | Behaviour |
|---|---|
| Definition parse (per workflow, in `on_event` / cron) | warn + skip that workflow, other workflows continue (`lib.rs:331-337`, `lib.rs:449-459`) |
| Trigger filter error | warn + skip (fail-closed, no run created) (`lib.rs:838-845`) |
| Trigger-context serialization error | error log; `on_event` returns `Ok(())` (`lib.rs:319-326`); cron path `continue`s (`lib.rs:579-589`) |
| Run creation error | error log + `continue` to next workflow (`lib.rs:356-361`, `lib.rs:601-611`) |
| Step failure | aborts the run, `PartialProgress` preserved so the partial trace is persisted (`executor.rs:1110-1174`, `lib.rs:240-261`) |
| Spawned task | detached `tokio::spawn`; failures only surface through DB status + logs (`lib.rs:371-381`, `lib.rs:649-661`) |
| Sink not initialized | `WorkflowError::InvalidDefinition("action_sink not initialized …")` instead of a panic (`lib.rs:148-156`) |
| `set_action_sink` misuse | explicit `panic!("action_sink already initialized")` (`lib.rs:139-143`) — the only panic path in the crate outside tests |
