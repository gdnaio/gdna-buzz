## Module: buzz-relay — HTTP API surface (`crates/buzz-relay/src/api`)
### Aspect: Conventions

---

#### 1. Handler signature conventions

Three distinct return-type dialects coexist in one module tree:

| Dialect | Return type | Used by | `file:line` |
|---|---|---|---|
| **Bridge/invites/operator** | `Result<Json<Value>, (StatusCode, Json<Value>)>` | 12 handlers | `bridge.rs:616`, `:883`, `:1317`, `:2077`, `:2103`, `:2121`; `invites.rs:233`, `:294`, `:165`; `operator.rs:152`, `:206`, `:268`, `:305`, `:357`, `:471` |
| **Media** | `Result<Json<BlobDescriptor>, MediaError>` / `Result<Response, MediaError>` — a domain error type with its own `IntoResponse` | 3 handlers | `media.rs:305-310`, `:604-608`, `:798-802` |
| **Admin** | `Result<Json<T>, ApiError>` with a typed error struct | 5 handlers | `admin/mod.rs:93-97`, `:125-129`, `:151-154`, `:177-181`, `:191-195` |

Two more one-offs: `workflow_webhook` returns `Result<(StatusCode, Json<Value>), (StatusCode, Json<Value>)>`
so it can emit 202 (`bridge.rs:1766`); `demo_echo` and `nostr_nip05` return bare `Response`
(`mesh_demo.rs:61`, `nip05.rs:29`), i.e. they are infallible by construction.

##### Extractor ordering

Consistent and load-bearing: `State` first, then `Path`, then `RawQuery`, then `Query`, then
`HeaderMap`, then the body (`Bytes` / `Body` / `Json`).

- `bridge.rs:2078-2081` — `State`, `HeaderMap`, `RawQuery`, `Query`: `RawQuery` **and** `Query` are
  both taken because NIP-98 signs the verbatim query string while the handler wants typed params.
  Same pattern at `operator.rs:305-309`, `:471-475`.
- `media.rs:306-309` — `State`, `AuthenticatedUpload` (a `FromRequestParts` extractor), `HeaderMap`,
  `Body`. The auth extractor runs before the body extractor by axum's rules, which is the whole
  point (`media.rs:29-38`).
- `mesh_demo.rs:59-62` — `State`, `Json<DemoEchoRequest>`. This is the one place where extractor
  ordering **defeats** the handler's intent: the `Json` rejection fires before the feature-flag 404.

##### Body handling

Handlers that need the raw bytes for NIP-98 payload verification take `axum::body::Bytes` and
`serde_json::from_slice` manually (`bridge.rs:618`, `:885`, `:1319`; `invites.rs:167`, `:236`;
`operator.rs:155`). Only `mesh_demo.rs:61` uses the `Json<T>` extractor — and it is the only handler
with no NIP-98 requirement, so nothing is lost there except the 404-indistinguishability property.

##### Tenant-binding preamble

Every tenant-scoped handler repeats the same 8-line block verbatim: read `Host`, `to_str().ok()`,
`unwrap_or("")`, `bind_community`, `map_err` to a fixed 404. Copied at `bridge.rs:621-633`,
`:888-901`, `:1321-1334`, `:1777-1786`, `:2013-2025`; `invites.rs:198-207`; `media.rs:154-166`,
`:477-487`; `nip05.rs:31-40`. Only `media.rs` factors it (`bind_media_read_tenant`, `media.rs:478`).

##### Post-auth helper split

`/events`, `/query`, `/count` each split into a thin routed wrapper plus an `_authed` helper, so the
wrapper can own exactly one terminal attribution log for every outcome:

| Wrapper | Helper | `file:line` |
|---|---|---|
| `submit_event` | `submit_event_authed` | `bridge.rs:613` / `:750` |
| `query_events` | `query_events_authed` | `bridge.rs:880` / `:947` |
| `count_events` | `count_events_authed` | `bridge.rs:1314` / `:1378` |

`submit_event` additionally routes outcomes through a `SubmitOutcome` enum carrying both log fields
and the HTTP response (`bridge.rs:706-747`), with `into_response()` collapsing it (`:722-729`).

#### 2. Error-to-status mapping

##### Envelope helpers (`api/mod.rs`)

```rust
api_error(status, msg)  -> (status, Json({"error": msg}))     // mod.rs:19-21
internal_error(msg)     -> tracing::error!; api_error(500, "internal server error")  // mod.rs:23-26
not_found(msg)          -> api_error(404, msg)                // mod.rs:28-31
```

`internal_error` is the only helper that deliberately withholds detail from the client while keeping
it in logs. It is used 30+ times across the module for every DB/serialize failure.

##### Status conventions actually observed

| Condition | Status | Body |
|---|---|---|
| Unmapped `Host` | 404 | fixed `"relay: no community is configured for this host"` (never echoes the host) |
| Missing/invalid NIP-98, replay detected, replay guard down | 401 | `"missing Nostr auth"` / `"NIP-98: {e}"` / `"NIP-98: replay detected"` / `"NIP-98: replay check unavailable"` |
| Read-gate violation (p-gate / engram / author-only) | **403** | `"restricted: …"` — note the WS sibling sends a `CLOSED` frame with an analogous string (`handlers/req.rs:186-204`) |
| Not a relay member | 403 | `{"error":"relay_membership_required","message":…}` — the **only** two-key error body in the non-admin surface (`api/mod.rs:134-139`) |
| Malformed filter/cursor/JSON | 400 | reflects the `serde_json` message |
| Rate limited | 429 | `"rate-limited: quota exceeded; retry in {n}s"` |
| Rate limiter unavailable | 503 | `"rate-limited: shared admission unavailable"` |
| Any DB/serialize error | 500 | fixed `"internal server error"` |

##### Media (`buzz-media/src/error.rs:107-162`)

All 15 authentication-ish variants (`MissingAuth`, `InvalidBase64`, `HashMismatch`, `ServerMismatch`,
`MissingTag`, `TokenExpired`, …) collapse to one 401 `"authentication failed"` explicitly "to prevent
oracle enumeration" (`error.rs:118-121`). `InsufficientScope` and `RelayMembershipRequired` are 403;
size 413; content-type 415; validation 422; rate/concurrency 429; IO/storage 500 with a generic body.

##### Admin (`admin/error.rs`)

Four constructors only — `bad_request(code, message)`, `forbidden()`, `not_found()`, `internal()` —
with `&'static str` code/message, so **no dynamic text can leak** through this surface by
construction. `From<DbError>` maps unconditionally to `internal()` (`admin/error.rs:79-83`).

##### Operator error mapping by string prefix

`operator.rs:180-199` matches on the `String` error returned by `community_provisioning`:
`"actor not authorized"`→403, `"community already exists"` / `"limit_reached:"`→409,
`"failed to create community:"` / `"community provisioned but owner bootstrap failed:"`→500 (generic
body + `tracing::error!`), fallthrough→400 with the message passed through. This is stringly-typed
control flow — a wording change in the provisioning module silently reclassifies the HTTP status.

#### 3. JSON shape conventions

| Convention | Where | Deviations |
|---|---|---|
| snake_case keys | bridge, invites, operator, media, mesh demo | admin uses `camelCase` (`admin/mod.rs:64`, `:140`, `:22`) |
| Reads return a bare JSON **array** | `/query`, `/moderation/*` | `/count` returns `{count}`; NIP-05 returns an object |
| Writes return `{event_id, accepted, message}` | `POST /events` (`bridge.rs:834-838`) | invites return `{code,…}` / `{status,…}`; operator returns `{community_id,…}` |
| Hex-encode all byte fields | `report_json`/`action_json`/`ban_json` (`bridge.rs:2132-2184`) | — |
| `#[serde(skip_serializing_if = "Option::is_none")]` for optional response fields | `operator.rs:51`, `handlers/community_provisioning.rs:67` | most ad-hoc `json!` bodies emit `null` instead |
| Ad-hoc `serde_json::json!` for response bodies | dominant style — 20+ sites | only 4 typed `Serialize` response structs exist (`TransferCommunityResponse`, `ProvisionCommunityResponse`, `FeedbackSummary`, `ErrorEnvelope`) |
| Empty-result shape is `[]` / `{}` , never 404 | `/query`, `/count`, NIP-05, `/api/join-policy` | — |
| Request DTOs are permissive | 12 of 13 DTOs ignore unknown fields | `ReportQuery` alone uses `deny_unknown_fields` (`admin/mod.rs:64`) |

#### 4. Tenancy conventions

- **Row zero**: bind the community from `Host` before anything else. The phrase "Row zero" appears
  as a literal comment marker at `bridge.rs:621`, `:888`, `:1321`, `:1773`; `media.rs:145`;
  `nip05.rs:32`; `router.rs:280`. Grepping for it is the fastest way to audit door coverage.
- **Never derive identity from `config.relay_url`'s host** — only its scheme. Three helper pairs
  encode this: `nip98_expected_url` (`bridge.rs:195-206`), `nip42_expected_relay_url`
  (`bridge.rs:225-231`), `media_base_url_for_tenant` (`media.rs:447-455`),
  `relay_url_for_tenant_host` (`nip05.rs:105-111`). Each has a paired test asserting the config host
  does **not** influence the output (`bridge.rs:2636-2654`, `:2749-2771`; `media.rs:1272-1280`;
  `nip05.rs:143-152`).
- Two deliberate exceptions, both documented in place: operator routes authenticate against
  `relay_operator_api_origin` and never bind a tenant (`operator.rs:57-60`); the admin surface is
  deployment-wide by design (`docs/admin/README.md:1-9`).

#### 5. Comment / documentation conventions

- Doc comments on handlers state method + path + auth mechanism as the first line
  (`bridge.rs:612`, `:877`, `:1310`; `media.rs:589-601`; `invites.rs:225-229`).
- Security-relevant decisions are argued inline at length, often citing the attack they close and
  the PR/review that found it: `bridge.rs:184-193` (NIP-98 host binding), `:208-224` (NIP-42
  sibling), `:1582-1594` (search post-filter), `:595-611` (log truncation), `media.rs:145-156`
  (bind-before-verify ordering), `invites.rs:36-43` (limiter capacity rationale),
  `invite_token.rs:24-46` (security properties **and non-properties**).
- Tests carry "bites if …" statements naming the regression they detect
  (`bridge.rs:2326-2328`, `:2414-2416`, `:2524-2528`, `:2706-2707`, `:3396-3400`).
- Module headers enumerate routes (`media.rs:3-8`, `invites.rs:3-13`, `mesh_demo.rs:1-23`).
- `// sadscan:disable np.postgres.1` suppresses the hardcoded-credential scanner on test DB URLs
  (`bridge.rs:3283`, `invites.rs:429`, `operator.rs:589`).

#### 6. Test conventions

**Counts (all 13 assigned files):** 159 test functions, **28** `#[ignore]`d, 0 `unsafe` blocks
(one occurrence of the word in a doc comment at `bridge.rs:303`), **1** TODO marker
(`media.rs:303`), **0** `unwrap()` outside `#[cfg(test)]`, **5** `expect()` outside `#[cfg(test)]`
(all in `invite_token.rs`: `:119`, `:139`, `:172`, `:349`, `:374` — every one an
infallible-by-construction HMAC/serialize call).

| File | tests | `#[ignore]` |
|---|---|---|
| `bridge.rs` | 64 | 8 |
| `media.rs` | 33 | 0 |
| `invites.rs` | 14 | 9 |
| `operator.rs` | 11 | 11 |
| `admin/mod.rs` | 10 | 0 |
| `webhook_secret.rs` | 10 | 0 |
| `invite_token.rs` | 9 | 0 |
| `api/mod.rs` | 3 | 0 |
| `nip05.rs` | 2 | 0 |
| `mesh_demo.rs` | 2 | 0 |
| `admin/auth.rs` | 1 | 0 |
| `admin/error.rs`, `events.rs` | 0 | 0 |

Established patterns:

1. **`#[ignore = "requires Postgres"` / `"requires Redis"`** is the gate for anything touching real
   infrastructure — 28 tests. Every `operator.rs` test is ignored, so the operator surface has
   **zero** coverage in `just test-unit`.
2. **Router-level `oneshot`** via `tower::ServiceExt` drives real HTTP through `build_router`, so
   route registration + extractor order + middleware are all in scope:
   `bridge.rs:3372-3390`, `invites.rs:598-620`, `operator.rs:635-660`, `admin/mod.rs:375-391`,
   `media.rs:1000-1010`.
3. **Injected `Nip98ReplayGuard` doubles** instead of Redis — four distinct doubles:
   `AlwaysErrGuard` (`bridge.rs:2330`), `AlwaysFreshReplayGuard` (`bridge.rs:3268`,
   `invites.rs:415`, `operator.rs:551`), `SeenOnceReplayGuard` (`invites.rs:1103`). The fail-closed
   test needs **no infrastructure** because the double supplies the error (`bridge.rs:2321-2377`).
4. **Positive controls beside every negative test** so a cross-host rejection test cannot pass
   vacuously (`bridge.rs:2477-2504`, `:2731-2747`).
5. **`current_thread` runtime + `metrics::with_local_recorder`** for metric assertions, because the
   recorder is a thread-local that a multi-thread scheduler would lose — reasoning spelled out at
   `bridge.rs:3255-3262`.
6. **Custom `tracing` capture writer** to assert the exactly-one-attribution-line invariant
   (`bridge.rs:3512-3584`).
7. **Test state builders** named `*_test_state` that mutate `Config::from_env()` then override
   `state.nip98_replay`: `bridge_handler_test_state` (`bridge.rs:3286`),
   `invite_test_state` (`invites.rs:441`), `operator_test_state` (`operator.rs:591`),
   `test_state_with_media_get_auth` (`media.rs:951`), `test_state` (`admin/mod.rs:335`).
8. **Silent skip vs panic is inconsistent**: `invites.rs:664-666` and `operator.rs:686-688` do
   `let Some(state) = … else { return; }` (test passes vacuously when Postgres is absent), while
   `bridge.rs:3405-3407` and `invites.rs:1074-1076` `panic!`/`expect` with an actionable message.
9. **Community isolation asserted explicitly** for every per-pubkey limiter
   (`media.rs:1120-1161`, `invites.rs:481-503`) and for the replay seen-set (`bridge.rs:2290-2292`).
10. Helper `nip98_auth_header(keys, url, method, body)` is **duplicated** in `invites.rs:505-519`
    and `operator.rs:596-616` with near-identical bodies, plus a third variant
    `build_nip98_event_json` + `nip98_auth_headers` in `bridge.rs:2380-2404`.

#### 7. Naming conventions

| Pattern | Examples |
|---|---|
| `verify_*` — cryptographic check returning `Result` | `verify_bridge_auth`, `verify_secret`, `verify_invite`, `verify_policy_acceptance` |
| `enforce_*` — check that returns an HTTP error tuple | `enforce_http_admission`, `enforce_relay_membership` |
| `authorize_*` — auth prelude returning the authenticated principal or tenant | `authorize_moderation_read`, `authorize_operator_request`, `authorize` (admin) |
| `extract_*` — pull an optional value out of raw JSON / headers | `extract_before_id`, `extract_depth_limit`, `extract_feed_types`, `extract_thread_cursor`, `extract_search_mode`, `extract_search_page`, `extract_page_offset`, `extract_channel_from_filter`, `extract_blossom_auth`, `extract_secret`, `extract_nip_oa_owner`, `extract_domain` |
| `*_json` — hand-rolled row→`Value` projection | `report_json`, `action_json`, `ban_json` |
| `handle_*_filter` / `handle_bridge_*` — one `/query` dispatch branch | `handle_channel_window_filter`, `handle_bridge_search` |
| `*_authed` — post-authentication continuation | `submit_event_authed`, `query_events_authed`, `count_events_authed` |
| `is_*` — pure boolean predicate | `is_safe_ext`, `is_sha256`, `is_admin_host`, `is_member_tag` |
| SCREAMING_SNAKE consts co-located with the code that reads them | `BRIDGE_FEED_MAX_LIMIT`, `MODERATION_READ_LIMIT`, `MAX_RANGE_CHUNK`, `CLAIM_RATE_LIMIT`, `MAX_INVITE_TTL_SECS`, `ECHO_TIMEOUT` |

#### 8. Convention violations / inconsistencies

| Issue | `file:line` |
|---|---|
| Three incompatible handler error dialects in one module tree (tuple / `MediaError` / `ApiError`) | see §1 |
| Two incompatible error-envelope JSON shapes | `api/mod.rs:19-21` vs `admin/error.rs:16-28` |
| camelCase only in the admin sub-tree | `admin/mod.rs:64`, `:140` |
| Stringly-typed status classification in the operator provisioning path | `operator.rs:180-199` |
| Tenant-binding preamble copy-pasted 9 times; factored only in `media.rs` | `media.rs:478-488` vs the rest |
| Stale `#[allow(dead_code)]` on a live function | `api/mod.rs:28` |
| `#[allow(private_interfaces)]` placed **between** two doc-comment blocks, splitting the doc | `media.rs:299-304` |
| `let _pubkey = …` discards the acting operator on transfer; archive/unarchive discard it entirely | `operator.rs:355`, `:209`, `:271` |
| `nostr_nip05` folds DB errors and misses into the same 200 via a catch-all `_ =>` arm | `nip05.rs:64` |
| `deny_unknown_fields` used on exactly one DTO | `admin/mod.rs:64` |
| `serve_blob_for_tenant` re-runs `validate_media_path` that its two callers already ran | `media.rs:604`/`:619`, `:630` |
