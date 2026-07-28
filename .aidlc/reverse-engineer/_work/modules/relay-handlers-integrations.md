## Module: buzz-relay — moderation, admin & background workers (`crates/buzz-relay/src`)
### Aspect: Integrations

---

#### 1. Dependency map

| Integration | Reached via | Files that touch it |
|---|---|---|
| `buzz-db` (Postgres) | `state.db` | all 12 except `moderation_authz` helpers' pure half |
| `buzz-pubsub` (Redis) | `state.pubsub`, indirectly via `state.disconnect_pubkey_clusterwide` / `dispatch_persistent_event` | `moderation_commands`, `moderation_notices`, `workflow_sink` |
| `buzz-audit` (hash chain) | `state.audit_tx` — **only transitively**, via `dispatch_persistent_event` | `moderation_notices`, `workflow_sink` (and `identity_archive` via ingest fall-through) |
| `buzz-media` / S3 | `state.media_storage` | `report` (sidecar lookup), `product_feedback` (blob verify), `storage_sweep` (bucket listing) |
| `buzz-search` | via `dispatch_persistent_event` | `moderation_notices`, `workflow_sink` |
| `buzz-workflow` | trait impl of `ActionSink` | `workflow_sink` |
| `buzz-core` | kinds, tenant, filter matching | all |
| `buzz-sdk` | `nip_oa::verify_auth_tag` | `identity_archive` |
| APNs push gateway | outbound HTTPS | `push_runtime` |
| `metrics` recorder (Prometheus) | `metrics::gauge!` / `counter!` | `storage_sweep`, `moderation_notices` |

---

#### 2. `buzz-db` (Postgres)

##### 2.1 Call inventory

| File | DB calls | Notable |
|---|---|---|
| `moderation_commands.rs` | `moderation_restriction_state` (`:105`), `ban_community_member` (`:169`), `unban_community_member` (`:248`), `timeout_community_member` (`:287`), `untimeout_community_member` (`:351`), `get_moderation_report_by_event` (`:414`), `resolve_moderation_report` (`:461`), `insert_moderation_action` (`:529`) | 8 distinct calls; **no transaction wraps them** |
| `moderation_authz.rs` | `get_relay_member` (`:98`, `:110`), `get_member_role` (`:126`) | 1–2 queries per authorization; `get_member_role` is unreachable |
| `moderation_notices.rs` | `open_dm` (`:102`), `unhide_dm` (`:127`), `query_events` (`:230`), `insert_event` (`:174`), `replace_addressable_event` (`:206`) | 5 calls per notice, minimum |
| `relay_admin.rs` | `get_relay_member` (`:135`, `:317`), `set_community_icon` (`:157`), `add_relay_member` (`:199`), `remove_relay_member_if_role` (`:243`), `remove_relay_member` (`:250`), `update_relay_member_role` (`:309`) | admin remove is a single conditional delete (TOCTOU-safe, `:235-241`) |
| `community_provisioning.rs` | `create_community_with_owner` (`:286`), `ensure_configured_community` (`:323`), `bootstrap_owner` (`:330`) | create-only path is one atomic call; convergence path is two |
| `report.rs` | `get_event_by_id` (`:56`), `insert_moderation_report` (`:79`) | |
| `product_feedback.rs` | `insert_product_feedback` (`:57`) | |
| `identity_archive.rs` | `get_relay_member` (`:241`), `query_events` (`:273`), `archive` (`:75`), `unarchive` (`:85`) | |
| `push_lease.rs` | `accept_push_lease_event` (`:563`) | single atomic transaction owning ordering + quota + endpoint namespace (`buzz-db/src/push.rs:206-212`) |
| `push_runtime.rs` | 13 distinct: `reap_exhausted_push_matches` (`:62`), `claim_due_push_match_batch` (`:72`), `active_push_match_leases` (`:102`), `membership_pairs` (`:115`), `enqueue_push_wakes` (`:184`), `complete_push_match_batch` (`:201`), `retry_push_match_batch` (`:207`/`:137`), `usage_community_hosts` (`:320`), `claim_due_push_wakes` (`:325`), `revalidate_push_wake` (`:356`/`:405`), `is_member` (`:375`), `disable_push_endpoint` (`:457`), `complete_push_wake` (`:438`/`:494`), `fail_push_wake` (`:363`/`:382`/`:412`/`:444`/`:471`/`:501`/`:535`), `retry_push_wake` (`:390`/`:541`) | heaviest DB consumer in the group |
| `workflow_sink.rs` | `lookup_community_host` (`:201`), `get_channel` (`:223`), `is_member_cached` (`:244`), `get_members` (`:275`), `get_users_bulk` (`:281`), `insert_event_with_thread_metadata` (`:339`) | 6 queries per workflow message |
| `storage_sweep.rs` | none directly — the host map is supplied by `main.rs` | |

##### 2.2 Failure modes

| Path | On DB error | Fail-open or fail-closed? |
|---|---|---|
| Restriction read (`moderation_commands.rs:107`) | `error: database error checking restriction state: {e}`, command rejected | **closed** |
| Ingest restriction gate (`ingest.rs:1645-1650`) | `IngestError::Internal` | **closed** |
| Auth-seam restriction read (`handlers/auth.rs:118-131`, `:145-152`) | `BanOutcome::DbError` ⇒ deny with `error: internal …`, distinguished from a real ban | **closed** |
| `authorize_moderation_action` (`moderation_authz.rs:99`, `:111`, `:127`) | `anyhow` error `?`-propagated, wrapped as `restricted: {e}` by `authz_denial` (`moderation_commands.rs:549`) | **closed, but the error string leaks a DB message under a `restricted:` prefix** |
| Ban/timeout write (`moderation_commands.rs:174`, `:292`) | `error: database error: {e}` | closed |
| Audit-row write (`moderation_commands.rs:544`) | `error: failed to write audit row: {e}` — **command fails after the ban already committed** | see §2.3 |
| Notice DM (any of 5 calls) | `anyhow` bubbles to `send_moderation_notice`'s caller, which logs and continues | **open by design** (`moderation_commands.rs:214-219`) |
| `relay_admin` any call | `database error: {e}`, wrapped by ingest as `invalid: database error: …` | closed |
| Report target resolution (`report.rs:58`) | `error: database error resolving report target: {e}` | closed |
| Push lease persistence (`push_lease.rs:572`) | `AcceptError::Internal("lease persistence failed")` — **the underlying `DbError` is discarded with `map_err(|_| …)`** | closed, but undiagnosable |
| Matcher context load (`push_runtime.rs:132-149`) | whole batch retried in 2 s | closed, retrying |
| Matcher per-job error (`:170-176`) | retry until `MAX_MATCH_ATTEMPTS = 8`, then discard as poison | bounded |
| Wake enqueue failure (`:186-198`) | contributing jobs retried; outbox dedup key absorbs partial commits | idempotent |
| `revalidate_push_wake` error (`:366-369`, `:414-417`) | `return` without touching the row — the claim lease simply expires | safe |
| `is_member` error during delivery (`:385-401`) | `retry_push_wake` in 2 s | closed |
| `usage_community_hosts` error (`:337`) | `error!` then the outer loop backs off | degraded, keeps looping |
| `workflow_sink` any call | `ActionSinkError::Database` → `WorkflowError::WebhookError` (`buzz-workflow/src/action_sink.rs:31-34`) → run fails | closed |
| `lookup_community_host` returns `None` (`workflow_sink.rs:205-210`) | `Database("workflow run community {id} is not mapped to a host")` | **closed by design** |

##### 2.3 Non-atomicity across the moderation write set

`handle_ban` performs four independent operations with no enclosing transaction: restriction read (`:105`), ban write (`:169`), audit write (`:180`), disconnect (`:195`), notice (`:204`). If the audit write fails, the ban is already durable but the command returns `error: failed to write audit row: …` (`:544`) — the client sees a failure for a ban that took effect. Same shape for timeout (`:287` then `:298`).

`handle_resolve` inverts the order deliberately — audit row **first** (`:453`), resolve **second** (`:461`) — with an in-code note that a lost-race resolve can leave an orphan audit row and that the residual window is tolerated (`:419-425`, `:469-474`).

---

#### 3. `buzz-pubsub` (Redis)

##### 3.1 Ban disconnect fan-out

`moderation_commands.rs:195-200` → `state.disconnect_pubkey_clusterwide` (`state.rs:1018-1050`):
1. Synchronous local socket close, fenced to the community (`state.rs:1025-1027`).
2. `tokio::spawn`ed `pubsub.publish_conn_control(&tenant, ConnControl::DisconnectPubkey { pubkey, event_id, reason })` (`state.rs:1043-1047`).

**Failure mode: fire-and-forget.** A Redis publish failure only emits `warn!("Failed to publish conn-control disconnect: {e}")` (`state.rs:1045`). The ban command still returns success. The in-code justification is that the durable ban row rejects the member again at auth (`state.rs:1039-1042`) and that the ingest write-path gate is the backstop for a surviving socket (`ingest.rs:1589-1596`).

**Consequence:** on another pod, a banned user's already-open socket keeps receiving events until (a) the Redis command arrives, (b) the socket reconnects and hits the auth seam, or (c) the user attempts a write and hits the ingest gate. Reads are not gated by the ingest path, so a missed publish means continued read access for the life of the socket.

The banning pod re-receives its own publish and no-ops; origin suppression was deliberately not added (`state.rs:1029-1031`).

##### 3.2 Notice and workflow fan-out

Both `moderation_notices.rs:178`/`:210` and `workflow_sink.rs:351` reach Redis only indirectly through `dispatch_persistent_event`. The workflow path discards the result entirely (`let _ =`, `workflow_sink.rs:351`), so a fan-out failure is invisible to the workflow run — the message is persisted and reported as sent.

Redis is also constructed directly inside two test helpers: `identity_archive.rs:445-448` and `workflow_sink.rs:580-588`. The latter deliberately points at `redis://127.0.0.1:1` (`workflow_sink.rs:578`) to prove the path works without a live Redis.

---

#### 4. `buzz-audit` (hash chain)

##### 4.1 Verified: no handler in this module writes an audit entry directly

`buzz-audit` declares **11** actions (`buzz-audit/src/action.rs:5-29`): `EventCreated`, `EventDeleted`, `ChannelCreated`, `ChannelUpdated`, `ChannelDeleted`, `MemberAdded`, `MemberRemoved`, `AuthSuccess`, `AuthFailure`, `RateLimitExceeded`, `MediaUploaded`.

Production emits exactly **2**: `EventCreated` (`handlers/event.rs:560`) and `MediaUploaded` (`api/media.rs:428`). **9 declared actions have zero producers.**

> **Documentation delta:** ARCHITECTURE.md:497 states "**10 audit actions**" and enumerates them without `MediaUploaded`. The enum has 11 (`action.rs:29`). The doc is stale by one variant, and its list is the set that is *declared*, not the set that is *emitted*.

Grep across the 12 assigned files: zero references to `buzz_audit`, `AuditAction`, or `state.audit_tx`. Confirmed.

##### 4.2 Which privileged mutations reach the hash chain, and how

| Mutation | Event stored? | Hash-chain entry? | Actor recorded |
|---|---|---|---|
| 9040 ban | no (`ingest.rs:1582-1586`) | **NO** | — |
| 9041 unban | no | **NO** | — |
| 9042 timeout | no | **NO** | — |
| 9043 untimeout | no | **NO** | — |
| 9044 resolve report | no | **NO** | — |
| 9030 add member | no (`ingest.rs:1811-1816`) | **NO** | — |
| 9031 remove member | no | **NO** | — |
| 9032 change role | no | **NO** | — |
| 9033 set workspace icon | no | **NO** | — |
| 1984 report | no (`ingest.rs:1563-1569`) | **NO** | — |
| 42000 product feedback | no (`ingest.rs:1541-1551`) | **NO** | — |
| 30350 push lease | yes (atomic with lease, `push_lease.rs:563`) | **NO** — ingest returns at `:2199` before the storage path that dispatches | — |
| **9035/9036 identity archive** | **yes** (falls through, `ingest.rs:1909-1912`) | **YES** — `EventCreated` | authenticated request signer |
| Moderation notice DM (kind:9) | yes (`moderation_notices.rs:174`) | **YES** — `EventCreated` | **relay pubkey** (`moderation_notices.rs:178`) |
| `"{host} Moderation"` kind:0 | yes, on first insert (`moderation_notices.rs:206`) | **YES** — `EventCreated`, only when `was_inserted` (`:207-211`) | relay pubkey |
| Workflow `send_message` (kind:9) | yes (`workflow_sink.rs:339`) | **YES** — `EventCreated` | **workflow owner**, not the relay key (`workflow_sink.rs:355`; rationale `handlers/event.rs:561-567`) |
| Operator community provisioning | n/a (HTTP) | **NO** | — |
| NIP-43 / NIP-IA announcement events emitted by `side_effects` | yes | yes, as `EventCreated` | per `side_effects` call |

**Net:** of the 14 privileged inbound kinds this module owns, **12 produce no hash-chain entry at all**. Bans, unbans, timeouts, role changes, member removals, ownership-affecting provisioning, and report resolutions are unauditable through `buzz-audit`. The only durable trail for moderation is the separate `moderation_actions` table (written by `moderation_commands.rs:529` only) — which is **not** hash-chained and therefore not tamper-evident. Relay-admin mutations (9030–9033) have **no** durable audit trail of any kind: no `moderation_actions` row, no event row, no hash-chain entry — only `tracing::info!` lines (`relay_admin.rs:164`, `:203-209`, `:268-272`, `:327-332`).

##### 4.3 Audit transport failure mode

`dispatch_persistent_event` sends into a bounded channel of capacity 1000 using `.send().await`, so backpressure propagates rather than silently dropping (`handlers/event.rs:548-576`). DB write failures inside the audit worker are logged but **not retried** (`:556-558`). A closed channel logs `Audit channel closed — entry lost` (`:575-577`). `state.audit_tx` being `None` skips auditing entirely (`:548-550`).

---

#### 5. `buzz-media` / S3

Three distinct integration points, all through `state.media_storage`:

| Consumer | Call | Purpose |
|---|---|---|
| `report.rs:66-71` | `get_sidecar(tenant, sha_hex)` | tenant-scoped blob existence check for `x`-tag report targets |
| `product_feedback.rs:35` | `verify_imeta_blobs(tenant, &imeta_tags, &state.media_storage)` | attachment verification before persisting feedback |
| `storage_sweep.rs` via `main.rs:1462-1470` | `list_page(token, 1000)` folded by `buzz_media::fold_bucket_listing` | whole-bucket inventory |

##### 5.1 Failure modes

**`report.rs`** — `map_err(|_| "invalid: report target blob not found")` (`:70`) collapses *every* failure class into "not found". Documented as a known Phase-1 limitation because the sidecar API exposes no typed not-found-vs-transient distinction (`:66-69`). A MinIO/S3 outage therefore tells reporters their blob does not exist.

**`product_feedback.rs`** — imeta errors propagate as `String` from `crate::api::validate_imeta_tags` / `verify_imeta_blobs` (`:34-35`) and reject the whole feedback submission.

**`storage_sweep`** — five distinct failure classes, all meaning "keep the old snapshot, never a partial one" (`buzz-media/src/bucket_index.rs:337-338`):

| `SweepError` | Cause | Site |
|---|---|---|
| `CapExceeded { seen, cap }` | cumulative object count exceeded `BUZZ_STORAGE_SWEEP_MAX_OBJECTS` | `bucket_index.rs:342`, raised `:394` |
| `Storage(MediaError)` | the S3/MinIO LIST call failed | `bucket_index.rs:345` |
| `Timeout(Duration)` | whole attempt exceeded `BUZZ_STORAGE_SWEEP_TIMEOUT_SECS`; **constructed by the relay**, not by the fold (`storage_sweep.rs:251`) | `bucket_index.rs:353` |
| `MalformedPage` | `is_truncated=true` with no continuation token — unresumable | `bucket_index.rs:357`, raised `:406` |
| task panic | `JoinError` | `storage_sweep.rs:194-202` |

All five increment `failures_total` and set `last_attempt.ok = false`; only `CapExceeded`/`Storage`/`Timeout`/`MalformedPage` log the operator-actionable hint "verify s3:ListBucket (or MinIO list) permission is granted on the bucket" (`storage_sweep.rs:176-181`).

##### 5.2 Credential coupling

The sweep uses the **same** `MediaStorage` instance as upload/download, therefore the same credentials: `BUZZ_S3_ACCESS_KEY` / `BUZZ_S3_SECRET_KEY` (`config.rs:622-625`). `list_page` is called with an empty prefix (`buzz-media/src/storage.rs:250`), i.e. whole-bucket listing across every tenant. There is no separate read-only or list-only credential and no per-tenant prefix restriction. Adding storage metrics therefore requires granting `s3:ListBucket` to the relay's read-write media key.

---

#### 6. APNs push gateway (outbound HTTPS)

##### 6.1 Client construction and destination

| Property | Value | Site |
|---|---|---|
| HTTP client | one `reqwest::Client` per worker, built once | `push_runtime.rs:313-316` |
| Timeout | `state.config.push_gateway_timeout` — applied as a whole-request `reqwest` timeout | `push_runtime.rs:314` |
| Timeout value | `BUZZ_PUSH_GATEWAY_TIMEOUT_MS`, default **2000 ms**, range-validated `100..=10000` | `config.rs:759-772` |
| Destination | `config.push_gateway_delivery_url` (`Option<url::Url>`) | `push_runtime.rs:422-424` |
| Default destination | `https://push.buzz.xyz/v1/deliveries/apns` | `config.rs:339`, `:755-758` |
| URL validation | scheme must be `https`, host required, no userinfo, path must be exactly `/v1/deliveries/apns`, no query, no fragment | `config.rs:341-361` |
| Disable | set `BUZZ_PUSH_GATEWAY_DELIVERY_URL` to an **empty** string | `config.rs:753` |
| Auth | NIP-98 kind-27235 event, base64'd into `Authorization: Nostr …`, with `u`/`method`/`payload`(sha256 of body)/`nonce` tags | `push_runtime.rs:551-565` |

**Failure mode: the client build `.expect("push HTTP client")` panics** (`push_runtime.rs:316`). This runs inside a `tokio::spawn`ed task, so a panic here silently kills the delivery worker while the matcher keeps enqueuing wakes — an unbounded outbox with no consumer, and no restart.

##### 6.2 Retry and invalidation semantics

| Condition | Action | Site |
|---|---|---|
| 2xx + `{"status":"accepted"}` | `complete_push_wake` | `:434-441` |
| 2xx + other/unparseable body | `fail_push_wake` (terminal) | `:442-447` |
| 410 + `invalid_endpoint{generation}` matching the wake | `disable_push_endpoint`, then `fail` | `:452-473` |
| 410 with mismatched generation | log only, then `fail` — **stale 410 cannot kill a rotated lease** | `:456-465` |
| 410 with unparseable body | `warn!("invalid closed-protocol 410 response")`, then `fail` | `:467` |
| 503 + `retry{retry_after_seconds>0}` | `retry_or_fail(that value)` | `:474-484` |
| 503 otherwise | `retry_or_fail(2)` | `:478-483` |
| 429 | `retry_or_fail(2)` — `Retry-After` header ignored | `:485-487` |
| 404 with `attempt > 1` | `complete_push_wake` — replay of a burned request id treated as delivered | `:488-497` |
| `is_timeout()` or `is_connect()` | `retry_or_fail(2)` | `:498` |
| anything else (incl. 4xx, TLS errors) | `fail_push_wake` (terminal) | `:499-503` |

Backoff: `delay * 2^(attempt-1)` clamped at `2^6 = 64×`, terminal at `MAX_ATTEMPTS = 8` (`push_runtime.rs:531-550`, const `:17`). Worst case with `delay=2`: 2, 4, 8, 16, 32, 64, 128, then fail — roughly 4 minutes.

**Every DB call in the response-handling path discards its result with `let _ =`** (`push_runtime.rs:436`, `:443`, `:455`, `:469`, `:492`, `:500`, `:534`, `:540`). A failed `complete_push_wake` therefore leaves the row claimed; recovery depends on the 30 s claim lease expiring and the wake being re-claimed — which is safe (idempotent via the request id) but invisible.

**SSRF assessment: not exposed.** The destination is operator config validated to a fixed path; the client-controlled `endpoint_grant` travels in the JSON body (`push_runtime.rs:507-515`), never in the URL. `reqwest` default redirect behaviour is not overridden, so a redirect from the configured gateway would be followed — a residual risk bounded by trusting the configured host.

##### 6.3 Counterpart crate

`crates/buzz-push-gateway/` implements the other side (its own `AppProfile` enum at `model.rs:14`, APNs client at `apns.rs`, grant model at `grant.rs`). It is not part of this module group; the wire contract between them is the `DeliveryRequest`/`DeliveryResponse` pair in `push_runtime.rs:31-51` and is not shared via a common crate — the two definitions are independent.

---

#### 7. `buzz-workflow`

`RelayActionSink` is the relay's implementation of `buzz_workflow::action_sink::ActionSink` (`workflow_sink.rs:172`). Wiring order matters and is documented: constructed after `AppState` (which owns `sub_registry` and `conn_manager`) and **before** the cron loop starts (`main.rs:591-597`).

Cycle avoidance: `AppState → WorkflowEngine → ActionSink → AppState` would be an `Arc` cycle, so the sink holds `Weak<AppState>` (`workflow_sink.rs:159-161`, rationale `:150-155`). Upgrade failure is mapped to `ActionSinkError::Database("relay is shutting down")` (`:186-188`).

Error surface: `ActionSinkError` has 6 variants (`buzz-workflow/src/action_sink.rs:11-29`) and **all of them collapse into a single `WorkflowError::WebhookError`** via `From` (`:31-34`). A channel-not-found, an archived channel, an access denial, and a genuine DB outage are therefore indistinguishable in workflow run output.

Failure modes reaching the workflow engine from this module:

| Cause | `ActionSinkError` | Site |
|---|---|---|
| shutting down | `Database` | `workflow_sink.rs:186-188` |
| community not mapped to a host | `Database` | `:205-210` |
| blank text | `EmptyContent` | `:212-214` |
| malformed channel UUID | `InvalidInput` | `:217-218` |
| channel missing | `ChannelNotFound` | `:225-230` |
| channel archived | `ChannelArchived` | `:232-236` |
| bad author pubkey hex | `InvalidInput` | `:238-240` |
| owner not a member of a non-open channel | `InvalidInput` | `:247-251` |
| any DB failure (5 calls) | `Database` | `:203`, `:229`, `:246`, `:277`, `:283`, `:342` |
| tag parse / signing failure | `EventBuild` | `:260-267`, `:292-294`, `:305` |

`buzz-workflow` also reaches the relay **outside** this sink for `add_reaction`, via its own HTTP client to `{BUZZ_RELAY_BASE_URL}/api/messages/{id}/reactions` (`buzz-workflow/src/executor.rs:885-919`). That route is not registered in `router.rs`, so this integration is permanently broken — see the features aspect.

---

#### 8. `buzz-search`

Reached only transitively through `dispatch_persistent_event` for the two relay-signed kind:9 paths (`moderation_notices.rs:178`, `workflow_sink.rs:351`). Indexing failures are absorbed inside `dispatch_persistent_event` and never surface here; `workflow_sink` additionally discards the whole result.

Consequence: a moderation notice DM is indexed into full-text search like any other message, so a moderator's `public_reason` becomes searchable by the recipient.

---

#### 9. `buzz-sdk` (NIP-OA)

Single integration point: `buzz_sdk::nip_oa::verify_auth_tag(auth_tag_json, &target_pubkey)` (`identity_archive.rs:320-326`), called twice per owner-consent request — once for the request's own `auth` tag (`:261`) and once for the target's live kind:0 `auth` tag (`:291`).

Failure mode: any verification error becomes a client-visible string via `e.to_string()` (`:324`), prefixed as `invalid request auth tag: {e}` (`:262`) or `invalid live kind:0 auth tag: {e}` (`:292`) — SDK-internal error text is exposed to the client.

Test helpers additionally use `nip_oa::compute_auth_tag` and `parse_auth_tag` (`identity_archive.rs:479-481`).

---

#### 10. Prometheus / `metrics`

| Metric | Type | Emitter | Labels |
|---|---|---|---|
| `buzz_channels_created_total` | counter | `moderation_notices.rs:113-118` | `community` (host), `type="dm"` |
| `buzz_storage_sweep_ok` | gauge | `storage_sweep.rs:293` | — |
| `buzz_storage_sweep_failures` | gauge (deliberately not `_total` — process-local, resets on failover, `:294-297`) | `storage_sweep.rs:298` | — |
| `buzz_storage_sweep_duration_seconds` | gauge | `:300` | — |
| `buzz_storage_sweep_age_seconds` | gauge | `:311-312` | — |
| `buzz_total_storage_bytes` / `_objects` | gauge | `:315-322` | `kind` ∈ {`physical`,`logical`} |
| `buzz_storage_orphan_blob_bytes`, `_orphan_blobs`, `_orphan_sidecars`, `_multi_variant_shas`, `_multi_variant_bytes`, `_unknown_key_bytes`, `_unknown_key_objects` | gauge | `:324-330` | — |
| `buzz_community_storage_bytes` / `_objects` | gauge | `:339-342`, zeroed at `:125-134` | `community` (host label) |
| `buzz_storage_unmapped_community_bytes` | gauge | `:347` | — |

`storage_sweep` emits **no** counters and no histograms — even sweep duration is a gauge, so percentile analysis across sweeps is not possible.

`push_runtime` emits **zero** metrics: no wake-enqueued counter, no delivery-outcome counter, no gateway-latency histogram. Delivery health is observable only through `warn!`/`error!` log lines.

---

#### 11. Integration risk summary

| Risk | Integration | Evidence |
|---|---|---|
| Delivery worker dies permanently on HTTP client build failure | reqwest | `push_runtime.rs:316` `.expect` inside a `tokio::spawn` |
| Every push DB result discarded (`let _ =`) — silent state divergence | `buzz-db` | 8 sites, `push_runtime.rs:436`…`:540` |
| Push lease DB error is undiagnosable | `buzz-db` | `push_lease.rs:572` `map_err(\|_\| …)` |
| 12 of 14 privileged kinds produce no hash-chain entry | `buzz-audit` | §4.2 |
| Relay-admin mutations have no durable audit trail at all | `buzz-audit` + `buzz-db` | `relay_admin.rs` writes only `tracing::info!` |
| Ban disconnect fan-out is fire-and-forget; reads stay open on other pods | `buzz-pubsub` | `state.rs:1043-1047` |
| S3 outage is reported to reporters as "blob not found" | `buzz-media` | `report.rs:66-70` |
| Storage sweep needs `s3:ListBucket` on the read-write media credential, whole-bucket | `buzz-media` | `config.rs:622-625`, `buzz-media/src/storage.rs:250` |
| All 6 `ActionSinkError` variants collapse to one `WorkflowError` | `buzz-workflow` | `action_sink.rs:31-34` |
| Workflow message fan-out failure invisible to the run | `buzz-pubsub`/`buzz-search` | `workflow_sink.rs:351` `let _ =` |
| Push wire contract duplicated, not shared | `buzz-push-gateway` | `push_runtime.rs:31-51` vs `buzz-push-gateway/src/model.rs` |
| Zero observability on push delivery | `metrics` | no `metrics::` calls in `push_runtime.rs` |
