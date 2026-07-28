<!-- Analyzed: 2026-07-25T01:12:08Z | Scope: full project -->
# Security Analysis

> Status: initialized in Phase 1. Auth coverage, input validation, secret handling, and
> data-exposure findings are populated per-module during Phase 2 and consolidated in
> Phase 3.
>
> These are static-analysis signals, not a penetration test. Professional security review
> is still recommended.

## Summary

Identity is secp256k1 keypairs; auth is NIP-42 (WebSocket challenge/response) and NIP-98
(HTTP); authorization is channel membership plus a scope enum; the audit log is a SHA-256
hash chain; SSRF protection is centralized in `buzz-core::is_private_ip`. Dependency policy
is gated by `cargo-deny` in CI.

Batch 2a posture: **zero `unsafe` blocks across all four crates**, no panics in
`buzz-core`/`buzz-sdk`/`buzz-persona` production paths, and disciplined secret hygiene in
the pairing and observer paths (`Zeroizing<String>` at API boundaries, explicit `.zeroize()`
after use, `Drop` impls, constant-time compares via `subtle`).

Strongest controls found:
- **Path-traversal defense in depth** — `buzz-persona::safe_resolve` rejects absolute paths,
  rejects any `..` component *before* canonicalization, canonicalizes (resolving symlinks),
  then requires the result to stay under the canonicalized pack root
  (`crates/buzz-persona/src/pack.rs:323-365`), with a Unix symlink-escape test.
- **Repo-id traversal defense** — `check_repo_id` blocks `..`, leading `.`, and any char
  outside `[A-Za-z0-9._-]`, enforced both in the builder *and* inside
  `GitRepoCoord::to_a_tag_value`, so a coordinate cannot smuggle a bad `d` value
  (`crates/buzz-sdk/src/builders.rs:92-121`, `:975-982`).
- **Self-attestation rejected both directions** in NIP-OA, so a stolen agent key cannot
  self-authorize (`crates/buzz-sdk/src/nip_oa.rs:154-159`, `:212-217`).
- **Encryption-required gate** — kind 24200 refuses content that does not look like NIP-44
  ciphertext, so plaintext telemetry cannot be published
  (`crates/buzz-sdk/src/builders.rs:256-260`).

Findings that need attention (detail in the per-module sections below):

| ID | Finding | Location |
|---|---|---|
| SEC-1 | **Persona hook paths stored unvalidated and traversal-capable.** The code documents this as accepted-and-deferred: "The PR that wires execution MUST validate through `safe_resolve()` before use." No type-level marker prevents a consumer from using the raw string. | `crates/buzz-persona/src/resolve.rs:339-346` |
| SEC-2 | **Persona `skills` and `avatar` paths unvalidated** in the emitted `ResolvedPersona`. | `crates/buzz-persona/src/resolve.rs:249`, `:243` |
| SEC-3 | **MCP `env` values (the credential channel) stored verbatim, `Debug`-printable, unredacted.** A literal secret committed in a pack is neither detected nor warned about; `{:?}` on a resolved persona prints it in the clear. | `crates/buzz-persona/src/resolve.rs:325-333`, derives at `:22`, `:67` |
| SEC-4 | **No capability/permission model for agents.** Capability is granted transitively: a persona's `mcp_servers` entry is an arbitrary subprocess spec (`command` + `args` + `env`) with no allow-list. A persona-level entry can wholly **replace** a pack-level server's `command`. | `crates/buzz-persona/src/resolve.rs:293-300` |
| SEC-5 | **Prompt injection surface is unmitigated by design.** Persona body, pack instructions, and `SKILL.md` content flow into agent context verbatim, size-capped only. The crate's own spec names skills as an injection vector. | `crates/buzz-persona/src/resolve.rs:200`; `crates/buzz-persona/PERSONA_PACK_SPEC.md:1041-1043` |
| SEC-6 | **`ws://` accepted with no scheme check or warning** — TLS is opportunistic via `MaybeTlsStream`; scheme enforcement is entirely the caller's responsibility. | `crates/buzz-ws-client/src/connection.rs:48-55` |
| SEC-7 | **Outbound AUTH frame logged in full at debug level**, including any `auth_tag` bearer-style token. | `crates/buzz-ws-client/src/connection.rs:123` |
| SEC-8 | **Challenge 1024-byte cap enforced on only 1 of 3 intake paths**; no replay/nonce tracking, no challenge content validation. | cap `crates/buzz-ws-client/src/connection.rs:198`; bypasses `:161`, `:170` |
| SEC-9 | **SSRF ranges incomplete** — `is_private_ip` omits RFC 5737 documentation ranges, IPv4 multicast/reserved, NAT64/6to4 prefixes, and IPv4-*compatible* IPv6 (`::a.b.c.d`, since `to_ipv4_mapped()` matches only `::ffff:*`). **Status: partially resolved** in `c26bf594` — the IPv6-side gaps are closed: `to_ipv4_mapped()` was replaced by `to_ipv4()` so IPv4-*compatible* `::a.b.c.d` now recurses into the IPv4 rules (`network.rs:62-65`), and NAT64 well-known `64:ff9b::/96` (`:71-73`), NAT64 local-use `64:ff9b:1::/48` (`:86-88`), Teredo `2001::/32` (`:89`), 6to4 `2002::/16` (`:90`) and legacy SIIT IPv4-translated `::ffff:0:0:0/96` (`:77-79`) are all blocked, the embedded-IPv4 forms recursively via `embedded_ipv4` (`:15-20`). **Residual (still open):** the IPv4-side omissions — RFC 5737 documentation `192.0.2.0/24` / `198.51.100.0/24` / `203.0.113.0/24`, IPv4 multicast `224.0.0.0/4`, IPv4 reserved `240.0.0.0/4`, and 6to4 relay anycast `192.88.99.0/24` — none appear anywhere in the file (zero matches for `192.0.2`, `198.51.100`, `203.0.113`, `224.`, `240.`, `192.88.99`), so the IPv4 arm is unchanged from the original finding. | `crates/buzz-core/src/network.rs:46-95`; IPv4 arm `:48-60` |
| SEC-10 | **Security-relevant set duplicated instead of shared** — `reader_authorized_for_event` hard-codes its two gated kinds rather than reading `kind::RESULT_GATED_KINDS`, so adding a kind to the constant does **not** extend the runtime gate. | `crates/buzz-core/src/filter.rs:25` vs `crates/buzz-core/src/kind.rs:129` |
| SEC-11 | **`check_hex_len` is a minimum-length check used where exact was meant** — a 65-hex pubkey passes on kinds 9000/9001 while being rejected elsewhere; the relay's `extract_p_tag_bytes` requires exactly 64 and would silently drop it. | `crates/buzz-sdk/src/builders.rs:44-52`, used `:569`, `:586` |
| SEC-12 | **`test-utils` helper sets `verified = true` without verifying.** Compile-gated (dev-deps only), so hygiene rather than live exposure. | `crates/buzz-core/src/lib.rs:47-74` (flag at `:73`) |

### Batch 2b posture (service crates)

Zero `unsafe` across all seven service crates. The dominant theme is **stale
documentation describing security controls incorrectly** — in three cases the docs
describe a control as absent when it exists, or as existing when it does not.

**The rate-limiting contradiction is resolved: the documentation is wrong, the control is real.**
`ARCHITECTURE.md` §9 #2 (`ARCHITECTURE.md:823`), and the supporting claims at `:390` and
`:460`, state that rate limiting is not enforced and only a test stub exists. In fact
`RedisRateLimiter` (`crates/buzz-pubsub/src/rate_limiter.rs:99`, struct `:88-90`) is
constructed unconditionally at `crates/buzz-relay/src/state.rs:712` (field `:584`) and
enforced *before* work on every WebSocket `EVENT`/`REQ`/`COUNT`
(`crates/buzz-relay/src/connection.rs:498-500` → `:594-653`) and on the HTTP bridge
endpoints `/events`, `/query`, `/count` (`crates/buzz-relay/src/api/bridge.rs:760`,
`:955`, `:1386` → `:24-56`). It is a fixed-window Lua `INCR`+`EXPIRE`
(`rate_limiter.rs:24-31`) and fails **closed** on Redis error
(`crates/buzz-relay/src/admission.rs:29-33`).

| ID | Finding | Location |
|---|---|---|
| SEC-13 | **Rate limiting is enforced, contrary to the docs — but 3 of 7 configured limits are dead.** `AGENT_STANDARD_API_CALLS`, `AGENT_ELEVATED_MESSAGES`, and `AGENT_PLATFORM_MESSAGES` are parsed and then read nowhere, so operators can set them with no effect. | parsed `crates/buzz-relay/src/config.rs:303-314`; no reader |
| SEC-14 | **Pre-authentication traffic is unmetered.** The limiter is pubkey-keyed, so `enforce_ws_admission` returns early for any unauthenticated connection. The one control that would cover this, `check_ip_connection`/`LimitType::IpConnections`, is implemented and has **no production caller**. | `crates/buzz-relay/src/connection.rs:602-609`; dead impl `crates/buzz-pubsub/src/rate_limiter.rs:112-120` |
| SEC-15 | **Rate-limiter counting logic is untested anywhere in the repo.** `rate_limiter.rs` has zero tests; relay tests substitute a stub that never touches Redis, so the Lua script, the `count <= limit` boundary, and the TTL-repair branch are unverified. | `crates/buzz-pubsub/src/rate_limiter.rs` (no tests); stub `crates/buzz-relay/src/admission.rs:65-90` |
| SEC-16 | **NIP-42 grants 16 scopes, not the 14 the docs describe.** `Scope::all_known()` is what the auth path calls; `all_non_admin()` (14 scopes) is never called anywhere. | `crates/buzz-auth` scope module; caller in the NIP-42 path |
| SEC-17 | **`ChannelAccessChecker` has zero implementors repo-wide.** The trait doc claims `buzz-db` implements it; it does not. `check_read_access`, `check_write_access`, and `require_scope` have no production callers — authorization runs through other paths, so this looks like a live control but is not one. | `crates/buzz-auth` access-checker module |
| SEC-18 | **Audit hash pre-image differs from its specification on five points**, including that it contains no `event_id`, `event_kind`, or `channel_id` at all. Anyone re-implementing verification from `ARCHITECTURE.md` would compute a different chain. | `crates/buzz-audit/src/hash.rs`; doc `ARCHITECTURE.md` audit section |
| SEC-19 | **Audit chain is unsigned and unanchored: a DB-write-capable attacker can rewrite the entire chain undetected.** Tail truncation and a missing genesis entry are also undetectable, and `verify_chain`/`get_entries` have no production caller — nothing verifies the chain in normal operation. | `crates/buzz-audit/src/hash.rs`, `src/service.rs` |
| SEC-20 | **Media size limits are not what the docs state.** "50 MB limit" applies to images only and is a relay default, not a crate invariant; real caps are video 500 MB and file 100 MB. Metadata policy is **reject**, not strip. | `crates/buzz-media/src/validation.rs`; relay defaults |
| SEC-21 | **Blossom GET authorization is implemented but off by default.** `verify_blossom_get_auth` is never called from `buzz-media`; the gate lives in the relay behind `BUZZ_REQUIRE_MEDIA_GET_AUTH`, which defaults to false — so media reads are unauthenticated out of the box, and content hashes are not verified on read. | `crates/buzz-media` (uncalled); relay gate + default |
| SEC-22 | **Unchecked arithmetic in the MP4 atom parser** — `offset += atom_size` at one site while a sibling site correctly uses `checked_add`. | `crates/buzz-media/src/validation.rs:481` vs `:891-893` |
| SEC-23 | **Workflow `call_webhook` accepts plain `http://`** despite documentation stating HTTPS, and the condition-evaluation timeout does not actually cancel the `spawn_blocking` task it guards. **Both halves still open at `07d0265c`** — re-verified: `call_webhook_impl` parses the URL and takes the host/port with no scheme check at all (`executor.rs:790-798`, and `:797-798` explicitly defaults port 80 for `http`), and the eval path still documents its own non-cancellation (`executor.rs:359-360`, timeout at `:370-372`). The sync did harden this function in one respect the report did not cover: `.no_proxy()` was added to the per-request client (`executor.rs:810`) so a system proxy cannot re-resolve the hostname past the pinned address. | `crates/buzz-workflow/src/executor.rs:790-798`, `:359-372` |
| SEC-24 | **Workflow approval gates are not a functioning control.** A token is generated but never persisted (TODO WF-08), runs are marked `Failed`, and **nothing in the repo ever writes the `WaitingApproval` state**, so the relay's resume path is unreachable and `Db::create_approval` has no caller. | `crates/buzz-workflow/src/executor.rs:663`; relay resume path |
| SEC-25 | **Search performs zero authorization by design** — the caller must re-authorize every hit. `community_id` is always the first predicate, so tenant scoping holds, but a caller that forgets to re-check leaks cross-channel content. | `crates/buzz-search/src/query.rs` |
| SEC-26 | **Cross-pod control messages are unauthenticated and unversioned.** Redis write access allows disconnecting arbitrary members with an attacker-chosen `reason` string echoed to the victim; a rolling deploy that adds a variant makes old pods silently skip a ban. | `crates/buzz-pubsub/src/conn_control.rs:55-73`, skip path `:152-156` |
| SEC-27 | **One tenant-table write lacks a community predicate** — `Db::backfill_d_tags` is the single exception among all tenant-scoped writes. | `crates/buzz-db/src/lib.rs:2810` |
| SEC-28 | **Fresh and brownfield installs run permanently different search policies.** Migration 0008 grants only *empty* databases a positive FTS allowlist, so two deployments of the same version index different content. | `migrations/0008*`; `crates/buzz-db/schema/schema.sql:212` |

Confirmed non-issues from 2b: the `dev` feature (with its `SHA-256("buzz-test-key:…")`
key derivation) is **not** enabled in production builds — verified via `resolver = "2"`
feature unification and `Dockerfile:67`. Media content hashes **are** verified
server-side on upload. Search is fully parameterized with no SQL-injection surface.

Auth coverage and scope enforcement at the relay seam are completed in batch 2c.

### Batch 2c posture (relay, mesh, conformance)

This is the batch where the enforcement seams actually live, and it inverts several
assumptions carried from 2a/2b. Two unauthenticated administrative surfaces and one
header-based impersonation path are the headline items; the formal-verification gate that
would detect tenant-boundary regressions is present but bound to a no-op in every deployed
configuration.

| ID | Finding | Location |
|---|---|---|
| SEC-29 | **`BUZZ_REQUIRE_AUTH_TOKEN` defaults to false, so an unsigned `X-Pubkey: <hex>` header impersonates any pubkey.** This covers `POST /events`, `/query`, `/count` and all three `/moderation/*` reads — including `private_reason`. The replay guard is skipped via a zero-id sentinel. The variable is **absent from the root `.env.example`**, so an operator has no signal it exists; the separate compose bundle does set it (`deploy/compose/.env.example:16` — `BUZZ_REQUIRE_AUTH_TOKEN=true`), which is the only place a deployment default is visible. | default `crates/buzz-relay/src/config.rs:475-477`; header path `api/bridge.rs:117-125`, `:2026`; `private_reason` `:2183`; sentinel `:122-123` → `:149-152` |
| SEC-30 | **The admin API has no credential at all** — authorization is `Host` equality plus an *optional* `Origin` check. Additionally 3 of its 4 DB calls take no `CommunityId`, so they are not community-scoped. | `crates/buzz-relay/src/admin/auth.rs:16-33`; unscoped calls `admin/mod.rs:132`, `:155`, `:184` |
| SEC-31 | **The WS p-gate is skipped for channel-scoped REQ while historical delivery runs unconditionally.** Kind 1059 is also absent from `is_global_only_kind` and not covered by `reader_authorized_for_event`. HTTP is stricter than WebSocket here. Still holds after `ab3af828`: that commit added a per-event result gate (`event_visible_to_reader`) but did not move the filter-level p-gate out of the `channel_id.is_none()` branch. | skip `handlers/req.rs:182`; delivery `:260-409`; `ingest.rs:379-436`; `crates/buzz-core/src/filter.rs:25-27`; HTTP `api/bridge.rs:981` |
| SEC-32 | **The DM-visibility owner fence (kinds 30622 / 44200) is missing on the cross-node path.** It exists only in `dispatch_persistent_event_inner`, not in `fan_out_pubsub_event`. Contrast the kind-30175 persona gate added by `ab3af828`, which was placed in the shared `filter_fanout_by_access` chokepoint (`handlers/event.rs:154-175`) and therefore *does* cover the Redis cross-node path (`:307`) — the same placement would close SEC-32. | `handlers/event.rs:457-491` (gate `:461`, recipient filter `:477-491`) vs `fan_out_pubsub_event` `:282`, access filter `:307` |
| SEC-33 | **The per-kind scope gate is inert.** Both transports pass `Scope::all_known()`, so the per-kind denial can never fire; the 81-kind scope map is documentation. All channel-scoped-token logic is dead (`channel_ids` hard-coded `None`). | `crates/buzz-auth/src/lib.rs:137`, `:138`; `api/bridge.rs:829`; unreachable gate `handlers/ingest.rs:1551` |
| SEC-34 | **Git has no read authorization at all**, and pointer creation is ungated — a zero-ref-command push creates pointers, bypassing `git_repo_names` and quota. `hydrate::run_git` has no timeout and runs `index-pack` without `--strict`. | `api/git/transport.rs:539-594`, `:786-827`; `api/git/hydrate.rs:451-470` |
| SEC-35 | **`GET /_mesh` is unauthenticated on the hard-coded `0.0.0.0` health listener** and leaks peer `endpoint_addrs`. Mesh gossip records are themselves unauthenticated, there is no peer eviction, and `MeshRuntime::shutdown()` is never called. | `crates/buzz-relay/src/main.rs:1116`, `:1119`; `crates/buzz-relay-mesh/src/status.rs:20`; `membership.rs:120-153` |
| SEC-36 | **`POST /api/invites/accept-policy` is an unauthenticated HMAC oracle**, and `/_mesh/demo/echo` takes `community_id` from the request body rather than the host-derived `TenantContext`. | relay router + mesh demo handler |
| SEC-37 | **Audit coverage is far narrower than the action set suggests** — the hash chain covers 2 of 11 actions, 12 of 14 privileged kinds are unaudited, and kinds 9030–9033 plus operator provisioning have **no durable trail** (`tracing::info!` only). `relay_admin.rs` performs **zero** ban re-check. | `crates/buzz-relay/src/handlers/relay_admin.rs` (unchanged in 2c sync); ingest-side routing `handlers/ingest.rs:1834-1842` (9030–9033, never stored) vs the deliberately-stored 9035/9036 fall-through `:1940-1945`; audit call sites |
| SEC-38 | **Side-effect failure is reported to the client as success** — warn-only, then dispatch proceeds, with no metric. | `handlers/ingest.rs:2460-2467`, `:2513` |
| SEC-39 | **The tenant-isolation conformance gate protects nothing at runtime.** Production binds `NoopTracer` (the sole assignment to the field), `check_trace` has zero production callers, and `JsonlTracer` is never instantiated — so every trace step, every `EmitGuard` `ImplBug`, and every projection breach is constructed and discarded with no log, metric, or counter. | `crates/buzz-relay/src/state.rs:798`; `crates/buzz-conformance/src/checker.rs:74`; `conformance/tracers.rs:18-20`, `:30-72` |
| SEC-40 | **Seven `Ok`-returning ingest paths emit no trace step**, including the command lane and the moderation lane — both of which perform community-scoped authorization and durable mutation. A tenant-boundary bug in either is structurally invisible to the gate. The REQ path arms no guard at all. | `handlers/ingest.rs:1560-1562`, `:1587-1596`, `:1605-1614`, `:1834-1842`, `:1846-1928`, `:2341-2347`, `:2452-2458`; no REQ guard `handlers/req.rs:116-118` |
| SEC-41 | **`bound_host` is written to the trace as a plaintext hostname**, making a JSONL trace file a cross-tenant correlation artifact; `JsonlTracer` writes to a caller-supplied path with default permissions and no redaction hook. | `crates/buzz-relay/src/conformance/mod.rs:58`; `conformance/tracers.rs:37-43` |

Positive controls confirmed in 2c: the `CommunityId` / `TenantContext` type fence is intact and
`buzz-conformance` deliberately declines to weaken it
(`crates/buzz-conformance/Cargo.toml:9-14`, `src/lib.rs:47-58`); host-derived tenancy is applied
uniformly across WebSocket, HTTP bridge, git, and Blossom paths; the sanitized-error alphabet is
closed by exhaustive match, so a new `IngestError` variant is a compile error
(`crates/buzz-relay/src/conformance/mod.rs:422-430`); row-community projection labels rows from
the row's own `channel_id` rather than the query predicate, and fails closed to `ImplBug` rather
than substituting a label (`conformance/mod.rs:186-260`).

## Authentication Coverage

| Surface | Mechanism | Enforcement Point | Source |
|---|---|---|---|
| _pending_ | | | |

## Authorization Model

_Pending Phase 2._

## Input Validation

_Pending Phase 2._

## Secrets & Credential Handling

Scan-level observations (to verify and expand in Phase 2):

| Observation | Location | Note |
|---|---|---|
| Desktop identity keys stored in OS keychain | `desktop/src-tauri/src/secret_store.rs`, `app_state_keyring.rs` | Preferred over env vars |
| Hardcoded public test private key + `0x…01` relay key in dev recipe | `Justfile` (`mesh-dev-fresh`) | Labelled local-only; must never target staging/prod |
| Dev-only key derivation `SHA-256("buzz-test-key:{username}")` | `buzz-auth` (`#[cfg(any(test, feature = "dev"))]`) | `dev` feature must stay disabled in production |
| S3 credentials via env or EKS Pod Identity | `BUZZ_S3_*`, `BUZZ_GIT_S3_*`, patched `aws-creds` | Fork pin is temporary |
| Git hook HMAC secret | `BUZZ_GIT_HOOK_HMAC_SECRET` | Required when git hosting is enabled |
| No vault/secrets-manager integration in-repo | — | Delegated to deployment platform |

## Known Limitations (from `ARCHITECTURE.md` §9 — verified in Phase 2)

| # | Limitation | Verification status |
|---|---|---|
| 2 | Rate limiting not enforced (only `AlwaysAllowRateLimiter` test stub) | ❌ **FALSE — doc is stale.** `RedisRateLimiter` is real, ungated, and enforced before work on WS `EVENT`/`REQ`/`COUNT` and the HTTP bridge. See SEC-13. Residual gaps: 3 dead limit vars (SEC-13), unmetered pre-auth traffic (SEC-14), no tests (SEC-15) |
| 5 | Workflow approval gates not wired end-to-end (runs marked Failed) | ✅ **CONFIRMED, and worse than stated** — nothing ever writes `WaitingApproval`, so the resume path is unreachable and the approval token is never persisted. See SEC-24 |
| 6 | `send_dm` / `set_channel_topic` workflow actions return `NotImplemented` | ✅ **CONFIRMED, plus one the doc misses** — `add_reaction` is also non-functional: it POSTs to a route the relay never registers (`crates/buzz-relay/src/router.rs:39-125`) |

## Findings

| Severity | Finding | Location | Recommendation |
|---|---|---|---|
| _pending_ | | | |

---

# Phase 2 — Module Findings

## Module: buzz-core (`crates/buzz-core`)

### Aspect: Security

---

### 1. SSRF helper — `is_private_ip` exact coverage

Function: `crates/buzz-core/src/network.rs:46-95`. Purpose per doc: "webhook targets must not resolve to these addresses" (`network.rs:22-24`).

#### IPv4 branch (`network.rs:48-60`)

Unchanged in substance by `c26bf594` — every IPv4 check below is byte-identical to the reported version, only shifted.

| Check as coded | CIDR / value covered | Line |
|---|---|---|
| `v4.is_loopback()` | `127.0.0.0/8` | `:50` |
| `v4.is_private()` | `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` (std semantics) | `:51` |
| `v4.is_link_local()` | `169.254.0.0/16` | `:52` |
| `octets[0] == 0` | `0.0.0.0/8` (whole first octet, not just `0.0.0.0`) | `:53` |
| `v4.is_broadcast()` | `255.255.255.255` exactly | `:54` |
| `octets[0] == 100 && (octets[1] & 0xC0) == 64` | `100.64.0.0/10` — CGNAT, RFC 6598; comment cites cloud metadata routing risk | `:57` |
| `octets[0] == 198 && (octets[1] & 0xFE) == 18` | `198.18.0.0/15` — benchmarking, RFC 2544 | `:59` |

Boundary tests pin the CGNAT and benchmarking edges: `100.64.0.1` and `100.127.255.254` blocked, `100.63.255.255` and `100.128.0.0` allowed (`network.rs:299-319`); `198.18.0.1` and `198.19.255.254` blocked, `198.17.255.255` and `198.20.0.0` allowed (`network.rs:321-339`).

#### IPv6 branch (`network.rs:61-93`)

**Rewritten by `c26bf594`.** The table below is the current code; the reported version had only the first six rows and used `to_ipv4_mapped()`.

| Check as coded | Range covered | Line |
|---|---|---|
| `v6.to_ipv4()` → recurse into the IPv4 rules | `::ffff:0:0/96` (mapped) **and** `::a.b.c.d` (IPv4-compatible) — was `to_ipv4_mapped()`, which matched only `::ffff:*` | `:62-65` |
| `embedded_ipv4(v6, &NAT64_WELL_KNOWN_PREFIX)` → recurse | `64:ff9b::/96` — RFC 6052 NAT64; public embedded IPv4 stays reachable, private/reserved embedded IPv4 is blocked | `:69-73` (prefix const `:7`, helper `:15-20`) |
| `embedded_ipv4(v6, &IPV4_TRANSLATED_PREFIX)` → recurse | `::ffff:0:0:0/96` — legacy SIIT IPv4-translated, not recognised by `to_ipv4()` | `:75-79` (prefix const `:10`) |
| `v6.is_loopback()` | `::1` | `:81` |
| `v6.is_unspecified()` | `::` | `:82` |
| `segments[0] & 0xfe00 == 0xfc00` | `fc00::/7` — ULA | `:83` |
| `segments[0] & 0xffc0 == 0xfe80` | `fe80::/10` — link-local | `:84` |
| `segments[0] & 0xff00 == 0xff00` | `ff00::/8` — multicast | `:85` |
| `segments[0] == 0x0064 && segments[1] == 0xff9b && segments[2] == 1` | `64:ff9b:1::/48` — RFC 8215 local-use NAT64 | `:86-88` |
| `segments[0] == 0x2001 && segments[1] == 0` | `2001::/32` — RFC 4380 Teredo | `:89` |
| `segments[0] == 0x2002` | `2002::/16` — RFC 3056 6to4 | `:90` |
| `segments[0] == 0x2001 && segments[1] == 0x0db8` | `2001:db8::/32` — RFC 3849 documentation | `:91-92` |

The IPv6 additions are covered by new boundary tests: IPv4-compatible (`:178-185`), NAT64 well-known (`:187-221`), IPv4-translated (`:223-252`), NAT64 local-use (`:254-267`), Teredo (`:269-282`), 6to4 (`:284-297`).

#### Ranges **not** covered (verified absent from the code)

| Range | Status in code |
|---|---|
| IPv4 `192.0.2.0/24`, `198.51.100.0/24`, `203.0.113.0/24` (RFC 5737 documentation) | not checked |
| IPv4 `192.88.99.0/24` (6to4 relay anycast) | not checked |
| IPv4 `224.0.0.0/4` multicast, `240.0.0.0/4` reserved | still not checked at `07d0265c` (IPv6 multicast *is* checked at `network.rs:85`; IPv4 multicast is not — zero matches for `224.`/`240.` in the file) |
| IPv6 `64:ff9b::/96` (NAT64 well-known prefix), `2002::/16` (6to4) | **now checked** — `c26bf594` added NAT64 well-known with recursive embedded-IPv4 evaluation (`network.rs:69-73`) and 6to4 (`:90`), plus NAT64 local-use `64:ff9b:1::/48` (`:86-88`) and Teredo `2001::/32` (`:89`) |
| IPv6 `100::/64` (discard-only) | still not checked (zero matches for `100::`/`0x0100`) |
| IPv6 IPv4-**compatible** addresses (`::a.b.c.d`, deprecated) | **now checked** — `to_ipv4_mapped()` was replaced by `to_ipv4()` (`network.rs:63`), so `::10.0.0.1` recurses into the IPv4 rules; asserted by `test_ipv4_compatible_v6_private` (`:178-185`) |
| DNS-rebinding / TOCTOU between resolution and connect | out of scope for this pure predicate — the caller must resolve then check then connect |

The function is a pure classifier over an already-resolved `IpAddr`; it performs no DNS resolution, so protection depends entirely on the caller checking every resolved address and pinning the connection.

---

### 2. Signature / ID verification

| Control | Implementation | file:line |
|---|---|---|
| Event ID must equal the canonical hash | `event.verify_id()`; on failure the expected ID is recomputed with `EventId::new(pubkey, created_at, kind, tags, content)` and both values are surfaced in the error | `verification.rs:12-25` |
| Schnorr signature check | `event.verify_signature()`, evaluated only after the ID check | `verification.rs:27-29` |
| CPU-bound warning | module doc mandates `tokio::task::spawn_blocking` in async contexts (this crate cannot enforce it — it has no runtime) | `verification.rs:1-2`, `:9` |
| Underlying crypto | delegated to `nostr` (which wraps `secp256k1`); the only direct secp256k1 surface here is the error type | `error.rs:19` |
| Verification inside the pairing state machine | `event.verify()` (id **and** signature) before any other check on inbound pairing events | `pairing/session.rs:632-635` |
| Verification for engrams | explicitly the **caller's** responsibility before `validate_and_decrypt` — documented, not enforced | `engram.rs:478-482` |
| Tamper tests | ID tamper (`verification.rs:44-58`), signature tamper (`:60-68`), and an independent check that a `sig`-only tamper leaves `verify_id()` true (`event.rs:57-66`) | |

`StoredEvent.verified` is a private bool with only a read accessor (`event.rs:17`, `:33-35`); `new()` forces `false` (`event.rs:28`) so a verified flag can only be asserted deliberately through `with_received_at` (`event.rs:38-48`).

---

### 3. `unsafe` code

| Finding | Evidence |
|---|---|
| Crate-level deny | `#![deny(unsafe_code)]` at `src/lib.rs:1` |
| Actual `unsafe` blocks/fns | **none**. The only textual matches for "unsafe" in `src/` are the lint attribute (`lib.rs:1`) and two prose uses of the word in `pairing/qr.rs:23` and `:226` ("characters unsafe in a query-parameter value") |
| Misleading comment | `engram.rs:410-412` contains a `// SAFETY:` comment justifying a slice index (`&body[start..j]` at `:413`), but the code is a **safe** slice operation, not an `unsafe` block. The comment style is borrowed from unsafe-code convention; no unsafe code is present |

---

### 4. Constant-time comparison

| Where | Implementation | file:line |
|---|---|---|
| Primitive | `ct_eq(&[u8;32], &[u8;32]) -> bool` via `subtle::ConstantTimeEq`, with a doc note on preventing compiler short-circuiting | `pairing/crypto.rs:120-129` |
| Pairing session-ID comparison | offer's `session_id` compared with `ct_eq` | `pairing/session.rs:173-181` |
| Transcript-hash comparison | received hash decoded then compared with `ct_eq` | `pairing/session.rs:357-363` |
| **Not** constant-time (by contrast) | engram `d`-tag comparison uses plain `!=` on `String` (`engram.rs:551`); `p`-tag owner comparison uses `eq_ignore_ascii_case` (`engram.rs:533`); the filter `#p` reader gate compares with `==` on `&str` (`filter.rs:32`). These compare values that are derivable by the legitimate reader or already public, but the asymmetry with the pairing path is a factual difference worth noting |

---

### 5. Secret handling

| Control | Implementation | file:line |
|---|---|---|
| Session secret generated from `rand::fill` (32 bytes) | `pairing/session.rs:113-115` |
| All-zero session secret rejected on QR decode | `pairing/qr.rs:169-174` |
| `QrPayload` zeroizes `session_secret` on drop | `pairing/qr.rs:52-56` |
| `PairingSession` zeroizes `session_secret`, `session_id`, `sas_input` on drop | `pairing/session.rs:731-739` |
| ECDH shared secret zeroized immediately after SAS derivation (both roles) | `pairing/session.rs:189` (source), `:295` (target) |
| Serialized plaintext zeroized after encryption | `pairing/session.rs:573`; `observer.rs:79` |
| Decrypted plaintext zeroized on success **and** on failure paths | `pairing/session.rs:610`, `:619`; `observer.rs:66`, `:101`, `:109` |
| Payload secrets travel as `Zeroizing<String>` across the API boundary | `pairing/session.rs:227-230`, `:388-391`, `:405` |
| Transient payload clone zeroized even on error (deferred `?`) | `pairing/session.rs:241-247` |
| Documented residual exposure | `serde_json::to_string` intermediate buffers and `nip44::encrypt`'s internal copy cannot be zeroized — stated at `pairing/session.rs:556-560` |
| Ephemeral private keys | zeroing delegated to `nostr::SecretKey`'s own `Drop`, which uses `write_volatile` (noted at `pairing/session.rs:729-730`) |
| Metadata privacy | outbound pairing events subtract random 0–30 s jitter from `created_at` | `pairing/session.rs:577-579` |

---

### 6. Authorization / access-control logic in this crate

| Control | Rule | file:line |
|---|---|---|
| Result-level private-event gate | reader must appear in the event's `p` tag for kinds 30622 and 44200; every other kind passes | `filter.rs:23-33` |
| Owner-only read for turn metrics | even the authoring agent is denied (test asserts it) | gate `filter.rs:28-32`, assertion `filter.rs:294-298` |
| Declared read-gate kind sets consumed by the relay | `AUTHOR_ONLY_KINDS` (`kind.rs:120`), `P_GATED_KINDS` (`kind.rs:146-156`), `RESULT_GATED_KINDS` (`kind.rs:129`); contracts documented at `kind.rs:112-144` |
| Relay-only kinds (client submission must be rejected) | `is_relay_only_kind` — 13534, 40901, 40902, 30622, 39005, 39006 | `kind.rs:758-769` |
| Cross-channel leak prevention in filters | an explicit `h` tag is authoritative; the `channel_id` fallback applies only when the event has no `h` tags | `filter.rs:78-100` |
| Role ladder used for git push authorization | Owner 4 / Admin 3 / Member 2 / Guest 1 / Bot 0; Bot satisfies nothing at this layer | `channel.rs:142-157` |
| Explicit rules can never weaken destructive defaults | effective minimum is `max(explicit push:role, built-in default)` | `git_perms.rs:535-552` |
| Protection constraints bind everyone including Owner | `no-force-push` / `no-delete` / `require-patch` are checked after the role check and have no role exemption | `git_perms.rs:527-570`; doc `git_perms.rs:5-7` |
| Atomic push semantics | any denied ref fails the whole push | `git_perms.rs:584-597` |
| Pairing peer binding | source locks to the offering pubkey; all later events must match it | `pairing/session.rs:187-188`, `:681-696` |
| Anonymous abort DoS prevention | `handle_abort` refuses before a peer is known | `pairing/session.rs:456-462` |
| Terminal-state immutability | `abort`/`handle_abort` refuse from `Completed`/`Aborted` | `pairing/session.rs:431-437`, `:449-454` |
| Replay/duplicate protection | per-session processed-id set; ids recorded only after full acceptance so probes cannot poison it | `pairing/session.rs:638-643`, `:676-678` |
| Session expiry | 120 s default, checked at the top of every handler | `pairing/session.rs:43`, `:698-703` |
| Tenant fence | `CommunityId`/`TenantContext` cannot be deserialized from client input; single `resolved()` constructor | `tenant.rs:17-30`, `:37-91` |

---

### 7. Feature-gated / test-only code and what it exposes

| Gate | What it exposes | file:line |
|---|---|---|
| `#[cfg(any(test, feature = "test-utils"))]` | `pub mod test_helpers`: `make_event` (`lib.rs:55`), `make_event_with_keys` (`:64`), `make_stored_event` (`:72`). These generate random `Keys` and sign test events; `make_stored_event` constructs a `StoredEvent` with **`verified = true`** without running `verify_event` | `lib.rs:47-74` (verified flag passed at `:73`) |
| `#[cfg(test)] impl PairingSession` | `has_processed()` (reads the dedup set) and `set_timeout()` (shrinks the session lifetime) — deliberately kept out of the public API per the doc comment | `pairing/session.rs:530-544` |

Notes:
- There is **no** `dev` feature in this crate (the `dev` feature seen in `crates/buzz-relay/Cargo.toml:84` belongs to `buzz-auth`, not `buzz-core`). `buzz-core` declares exactly one feature: `test-utils` (`Cargo.toml:10-11`).
- `test-utils` is enabled only by `crates/buzz-relay/Cargo.toml:89` inside `[dev-dependencies]`, so the helper cannot leak into a release relay binary through that edge.
- Because `test_helpers::make_stored_event` sets `verified = true`, any production code path that accepted a `StoredEvent` from this helper would bypass verification. It is compile-gated, so this is a hygiene observation rather than a live exposure.

---

### 8. Input validation inventory and gaps

Validated (fail-closed) inputs:

| Input | Validation | file:line |
|---|---|---|
| Nostr event (id, sig) | full verification | `verification.rs:11-32`; pairing path `session.rs:632-635` |
| NIP-44 ciphertext length | 132–87,472 chars before decryption | `observer.rs:53-55`, `:85-90`; `session.rs:595-601` |
| Decrypted plaintext size | ≤ 65,535 bytes, zeroized on rejection | `observer.rs:96-104`; `session.rs:609-615` |
| Engram slug | grammar + 255-byte cap + per-segment 64-byte cap | `engram.rs:67-112` |
| Engram body JSON | strict parse, duplicate member names rejected at any depth, trailing data rejected, field types checked | `engram.rs:216-259`, `:283-380` |
| Engram envelope | kind, author, exactly one `d`, 64 lowercase hex `d`, exactly one `p`, owner match, slug↔d-tag re-derivation | `engram.rs:489-557` |
| Engram body size | ≤ 65,535 bytes at build time | `engram.rs:436-440` |
| QR URI | ≤ 2048 chars, scheme, 64-char lowercase hex pubkey, 64-char lowercase hex secret, non-zero secret, ≥ 1 relay, each relay a parseable `ws`/`wss` URL with a host, version must be 1 | `qr.rs:104-206` |
| Relay URL for runtime identity | scheme, no credentials, no fragment, host required | `relay.rs:37-64` |
| Host header | normalization with fail-closed empty result | `tenant.rs:121-139` |
| Git ref pattern | length, `refs/` prefix, segment charset, wildcard count, `**` position | `git_perms.rs:83-146` |
| `buzz-protect` tag | min value count, role validity, `push:bot`/`push:guest` rejection, per-repo rule cap | `git_perms.rs:303-362`, `:379-399` |
| NIP-AM payload numbers | `cost_usd` finite and ≥ 0 on both encrypt and decrypt | `agent_turn_metric.rs:147-190` |
| Presence status | strict serde enum | `presence.rs:10-18` |
| Pairing protocol version | must be 1 (QR and offer) | `qr.rs:145-151`, `session.rs:169-175` |

Gaps and asymmetries found in the code (stated factually, no severity judgement):

| # | Gap | Evidence |
|---|---|---|
| G-1 | NIP-AM structural requirements are documented but unenforced: `session_id`/`turn_seq` required when `cumulative` is present, `turn_seq` strictly increasing, `timestamp` must be RFC 3339. `validate()` checks only `cost_usd`; `timestamp` is an unparsed `String` | doc `agent_turn_metric.rs:97-111`; `validate()` `:147-169`; field type `:111` |
| G-2 | `harness`, `model`, `session_id`, `turn_id`, `channel_id` have no length or format bounds; `channel_id` is a free `String` rather than a UUID type | `agent_turn_metric.rs:89-100` |
| G-3 | `reader_authorized_for_event` hard-codes its two kinds instead of reading `RESULT_GATED_KINDS`, so adding a kind to the constant does not extend the gate | `filter.rs:25` vs `kind.rs:129` |
| G-4 | Filter matching implements no `limit` and no NIP-50 `search` handling; a caller relying on `filters_match` alone gets no result-count bound | `filter.rs:35-104` |
| G-5 | `filters_match` treats a present-but-empty `kinds`/`authors`/`ids` collection as "match nothing" purely as a side effect of `contains`; the NIP-01 empty-vs-absent distinction is not addressed explicitly | `filter.rs:36-66` |
| G-6 | Event ID prefix matching uses `starts_with` on hex (`filter.rs:60-66`); a 1-character filter id would match a large set. No minimum prefix length is enforced |
| G-7 | `normalize_host` strips only exact `:443`/`:80` suffixes, so an IPv6 literal with a non-default port keeps its port while `[::1]:443` collapses — correct per the doc, but it means host strings must already be authority-shaped; no validation rejects a malformed host string | `tenant.rs:121-139` |
| G-8 | `engram::extract_refs` silently drops malformed candidates (documented) — no signal to the caller that a reference was present but invalid | `engram.rs:373-381`, `:421-425` |
| G-9 | `git_perms::parse_protection_tags` checks the rule count **before** parsing each tag (`rules.len() >= MAX_PROTECTION_RULES` at `:387-389` precedes the parse at `:391`), so a 51st malformed tag reports `TooManyRules` rather than the parse error | `git_perms.rs:383-394` |
| G-10 | `UpdateKind::classify` trusts the caller-supplied `is_ancestor` boolean and does not validate OID shape (any non-zero string is treated as a real OID) | `git_perms.rs:206-221` |
| G-11 | `url_decode` in the QR path uses `decode_utf8_lossy`, so invalid UTF-8 percent-escapes become replacement characters rather than an error (the subsequent `Url::parse` is the effective guard) | `qr.rs:236-238`, validation at `:183-204` |
| G-12 | Pairing `PayloadType::Custom` content is passed through unvalidated by design ("interpretation is out-of-band") | `pairing/types.rs:75-76` |
| G-13 | `PairingSession::processed_ids` grows unbounded within a session; the bound is argued from the 120 s lifetime and ~6 expected events rather than enforced by a cap | doc `pairing/session.rs:626-628`, field `:105-106` |
| G-14 | `engram::select_head` performs no validation of its inputs (kind, tags, signature) — documented as the caller's job | `engram.rs:560-563` |
| G-15 | `is_private_ip` does not cover the RFC 5737 documentation ranges, IPv4 multicast/reserved space, NAT64/6to4 prefixes, or IPv4-compatible IPv6 addresses (see §1). **Status: partially resolved** in `c26bf594` — NAT64/6to4/Teredo and IPv4-compatible IPv6 are now covered (`network.rs:62-65`, `:71-73`, `:77-79`, `:86-90`); the RFC 5737 documentation ranges, IPv4 multicast `224.0.0.0/4`, IPv4 reserved `240.0.0.0/4` and `192.88.99.0/24` remain uncovered | `network.rs:48-60` (IPv4 arm, unchanged) |


## Module: buzz-sdk (`crates/buzz-sdk`)

### Aspect: Security

---

### 1. Signing and key handling

| Finding | Detail | File:line |
|---|---|---|
| Builders never sign | All 51 builders return an unsigned `nostr::EventBuilder`; the caller signs with its own `Keys`. Documented as "No keys are held here." | `crates/buzz-sdk/src/lib.rs:11-13` |
| No key material in builder signatures | No builder takes `Keys`, `SecretKey`, or any secret parameter — pubkeys are accepted as hex `&str` only | `builders.rs:219-1823` (all signatures) |
| One signing path in the crate | `nip_oa::compute_auth_tag` takes `&Keys` (owner) and calls `owner_keys.sign_schnorr(&message)`. The `Keys` value is borrowed, used once, never stored or logged | `nip_oa.rs:146-176`, sign at `nip_oa.rs:170` |
| Verification path | `verify_auth_tag` uses `SECP256K1.verify_schnorr` against the x-only owner pubkey extracted from the tag | `nip_oa.rs:237-244` |
| Cryptography used | SHA-256 digest (`nostr::hashes::sha256`) + BIP-340 Schnorr over secp256k1 — current, not deprecated | `nip_oa.rs:22-26`, `109-116` |
| Self-attestation rejected on both sides | `owner == agent` fails at compute (`nip_oa.rs:154-159`) and at verify (`nip_oa.rs:212-217`), so a stolen agent key cannot self-authorize | `nip_oa.rs:154-159`, `212-217` |
| Structure-only fast path is explicit | `parse_auth_tag` performs **no** signature verification and says so; used at MCP startup | `nip_oa.rs:239-251` |
| `verify_auth_tag` returns the owner key | Callers get the attesting owner pubkey back for authorization decisions rather than a bare bool | `nip_oa.rs:179-248` |

**Secret material in memory:** the `zeroize` crate is not a dependency and is not
used anywhere in this crate (`crates/buzz-sdk/Cargo.toml:10-16`). The only place
a secret exists is the borrowed `&Keys` inside `compute_auth_tag`
(`nip_oa.rs:146`) — the SDK creates no secret copies, so there is nothing local
to zeroize; lifetime of the key material is the caller's responsibility. By
contrast, `buzz-core`'s observer encryption path does zeroize its plaintext
(`crates/buzz-core/src/observer.rs:63-68`), so the pattern exists in the
workspace but is not needed here.

The example binary accepts an owner secret key **as a command-line argument**
(`examples/compute_auth_tag.rs:11-21`), which exposes it to the process table
and shell history. It is an example, not a shipped binary, but it is the one
place in the crate where secret handling is user-facing.

---

### 2. Input validation posture

Validation is the crate's primary security function — it is the last gate before
a signature is applied to a wire form the relay will trust.

| Control | Coverage | File:line |
|---|---|---|
| Pubkey format | exactly 64 ASCII hex, lowercased on output — used in 20+ builders. Tests explicitly pin rejection of **over-long** (65-char) pubkeys because the relay's `extract_p_tag_bytes` requires exactly 64 and would silently drop a longer value | `builders.rs:69-77`; test `builders.rs:3613-3618` |
| Event-id format | exactly 64 hex, lowercased | `builders.rs:79-89` |
| Commit id format | exactly 40 or 64 hex — abbreviated refs refused for NIP-34 canonical tags | `builders.rs:59-66` |
| Path-traversal defense on repo ids | `check_repo_id` blocks `..`, leading `.`, and any character outside `[A-Za-z0-9._-]`; enforced both in `build_repo_announcement` and inside `GitRepoCoord::to_a_tag_value`, so a coordinate built directly through the SDK cannot smuggle an invalid `d` value into an `a` tag. Regression test uses `../etc/passwd` | `builders.rs:92-121`, `975-982`; test `builders.rs:3113-3121` |
| Control-character filtering | `reason` codes for identity archival reject any `char::is_control()` | `builders.rs:1706-1721`; test `builders.rs:3790-3795` |
| Size bounds | content caps 60/64 KiB, emoji 64 chars, shortcode 64 bytes, URL 2048 bytes, repo name 128, description 1024, clone URL 512, relay 256, petname 256, reason 64 bytes, contacts 10 000, mentions 50, DM participants 8, clone URLs 5, relays 10 | `builders.rs:35-41`, `152-170`, `751`, `840-919`, `1545-1549`, `1704`; `mentions.rs:38` |
| Encryption-required gate | `build_agent_observer_frame` refuses content that does not pass `content_looks_like_nip44` (length 132–87 472), so plaintext telemetry cannot be published on kind 24200. Dedicated negative test exists | `builders.rs:256-260`; `crates/buzz-core/src/observer.rs:53-55`; test `builders.rs:1914-1924` |
| NIP-70 protection | identity archival requests are always marked `["-"]` (protected) | `builders.rs:1748` |
| Enum-constrained vocabularies | presence status, moderation status/action, channel visibility, observer frame direction are all whitelist-matched | `builders.rs:1571-1584`, `1660-1680`, `615-621`, `251-255` |
| NIP-OA conditions grammar | strict canonical-decimal parser: no whitespace, no empty clauses, no leading zeros, range-bounded, case-sensitive labels — reduces the chance that relay and client disagree on an attestation's scope | `nip_oa.rs:36-107` |
| Lowercase-hex enforcement on auth tags | `parse_auth_tag` rejects uppercase/mixed-case owner pubkey and signature, preventing two encodings of the same tag | `nip_oa.rs:120-122`, `274-296`; test `nip_oa.rs:458-486` |
| UTF-8 boundary safety | `extract_nostr_uris` checks `is_char_boundary` before slicing a fixed-width window; `extract_at_mentions_with_known` uses `get(..)` rather than direct indexing. Both have explicit no-panic tests | `mentions.rs:135-138`, `374-377`; tests `mentions.rs:520-531`, `787-800` |

---

### 3. Validation gaps observed

| Gap | Consequence | File:line |
|---|---|---|
| `build_set_canvas` has no content-length check | a canvas document of unbounded size can be signed; every other content builder caps at 60–64 KiB | `builders.rs:529-532` |
| `build_workflow_approval` note is unbounded | free-text `note` becomes event content with no size check | `builders.rs:1522-1541` |
| PR / PR-update `clone_urls` entries are unvalidated | only list non-emptiness is checked; individual URLs get no scheme or length check (unlike `build_repo_announcement`, which validates each) | `builders.rs:1344-1350`, `1425-1431` vs `868-882` |
| `q`-tag relay hints pass through raw | relay-url hint on applied patches is written verbatim with no scheme/length validation | `builders.rs:1266-1272` |
| NIP-02 `relay_url` scheme unchecked | length-capped at 2048 bytes but any scheme accepted | `builders.rs:785-792` |
| `build_diff_message` accepts 7-char commit SHAs | intentionally looser than `check_commit_hex`; a truncated SHA is ambiguous but is written into the `commit` tag | `builders.rs:322-325` |
| Free-text tag values are not sanitized | `about`, `topic`, `purpose`, `description`, `reason`, `public_reason`, `alt`, `subject`, `branch-name`, labels, and petnames accept any UTF-8 including control characters and newlines (only the NIP-IA `reason` is control-char filtered). Consumers render these | `builders.rs:632-638`, `655`, `664`, `1093`, `1361`, `1607-1609`, `1685-1687` |
| `build_repo_announcement_with_tags` trusts caller tags | every non-`d` tag is preserved verbatim in a read-modify-write update; only the `d` tag is canonicalized | `builders.rs:952-963` |
| `build_profile` does not validate `picture`/`nip05` | any string is accepted into the kind-0 JSON | `builders.rs:537-562` |
| `check_auth_tag_shape` does not validate the conditions element | the builder-side auth-tag check verifies label/pubkey/signature shape but not `auth[2]`, while `nip_oa::parse_auth_tag` does validate conditions — the two entry points differ | `builders.rs:1723-1737` vs `nip_oa.rs:283-286` |

These are documented as observations; several are explicitly delegated to the
relay in the source comments (e.g. `builders.rs:1697-1699` for NIP-IA consent,
`builders.rs:1648-1651` for moderation status/action pairing).

---

### 4. Hardcoded values

| Value | Nature | File:line |
|---|---|---|
| `CUSTOM_EMOJI_SET_D_TAG = "buzz:custom-emoji"` | protocol constant (public) | `builders.rs:503` |
| `MAX_CONTACTS = 10_000`, `MAX_REASON_BYTES = 64`, `MENTION_CAP = 50` | limit constants | `builders.rs:751`, `1704`; `mentions.rs:38` |
| Inline size limits (`64 * 1024`, `60 * 1024`, `2048`, `1024`, `512`, `256`, `128`, `64`, `8`, `5`, `10`) | magic numbers used directly at call sites rather than named constants | `builders.rs:227`, `314`, `157`, `856`, `875`, `907`, `845`, `1092`, `1546`, `868`, `900` |
| NIP-OA preimage prefix `"nostr:agent-auth:"` | protocol constant | `nip_oa.rs:110` |
| Bech32 window `"nostr:npub1"` + 58 chars | protocol constant | `mentions.rs:354-355` |
| Test key material: spec owner/agent pubkeys, spec signature, `0x…02` secret key | **test-only** constants, no production use | `nip_oa.rs:303-310`; `builders.rs:3701-3705`, `3756-3759` |

No credentials, tokens, URLs, or hostnames are hardcoded outside tests.

---

### 5. Unsafe code

`#![deny(unsafe_code)]` at the crate root (`crates/buzz-sdk/src/lib.rs:1`).
A search for `unsafe` across `src/` and `examples/` returns only that lint
attribute — **zero unsafe blocks**, confirmed.

### 6. Panic surface

No `unwrap()` or `expect()` exists outside `#[cfg(test)]` code (verified by scan
of all three source files). `GitAppliedPatchRef::parse` uses
`unwrap_or_default()` (`builders.rs:1145`), which cannot panic. Slicing in
`GitAppliedPatchRef::parse` (`builders.rs:1157`, `1161-1162`) and
`strip_code_regions` operates on byte offsets derived from `find`/`rfind` and
`char_indices`, so indices are char-boundaries by construction; the one
fixed-width window in the crate is explicitly boundary-guarded
(`mentions.rs:374-377`). The example binary panics by design on malformed CLI
arguments (`examples/compute_auth_tag.rs:21-27`).


## Module: buzz-persona (`crates/buzz-persona`)

### Aspect: Security

### 1. `unsafe` usage

**None.** A grep for `unsafe` across the whole of `crates/buzz-persona` (all `.rs`,
`Cargo.toml`, and `PERSONA_PACK_SPEC.md`) returns zero matches. There is no
`#![allow(unsafe_code)]`, no FFI, no raw pointer manipulation. The crate is 100% safe Rust.

Also absent from non-test code: `unwrap()`, `expect()`, `panic!`, `unreachable!`,
`todo!` (all matches are inside `#[cfg(test)]` modules or `tests/`), so a malformed pack
cannot panic the caller through this crate's parsing paths.

---

### 2. Prompt-injection surface

This is the crate's largest security-relevant characteristic: **persona content flows into
agent prompts entirely unvalidated and unsanitized.**

| Finding | Evidence |
|---|---|
| The markdown body of a `.persona.md` becomes the agent system prompt with **no** content inspection — no length-bounded sanitization beyond the raw byte cap, no escaping, no scan for instruction-override strings | `let system_prompt = lp.prompt.clone();` — `crates/buzz-persona/src/resolve.rs:200`; body captured verbatim at `crates/buzz-persona/src/persona.rs:257` |
| The only guard on prompt content is a size cap: 256 KiB body, 1 MiB frontmatter | `crates/buzz-persona/src/persona.rs:21`, `:24`, enforced `:213-218`; second cap at the file level `crates/buzz-persona/src/pack.rs:153-154` |
| `pack_instructions` (contents of `instructions.md`) is also passed through verbatim, trimmed only | `crates/buzz-persona/src/resolve.rs:201-204` |
| `display_name` and `description` are free-form strings with only a non-empty check — no charset restriction, no length cap. They reach the agent/UI as-is. | `crates/buzz-persona/src/persona.rs:233-239`; test fixtures include emoji values such as `"Pip 🐝"` (`crates/buzz-persona/tests/integration.rs:64`) |
| `triggers.keywords` are free-form strings, unbounded in count and length | `crates/buzz-persona/src/persona.rs:60`; merged without limit at `crates/buzz-persona/src/merge.rs:184-193` |
| Skills are described in the crate's own spec as prompt-injection vectors, and this crate performs no skill content validation at all — `SKILL.md` bodies are never read except to compare the `name:` field | spec statement at `crates/buzz-persona/PERSONA_PACK_SPEC.md:1041-1043` ("Skills are markdown injected into agent context; malicious content can attempt prompt injection"); code reads only frontmatter `name` at `crates/buzz-persona/src/validate.rs:410-431` |

Mitigating structural choices that *are* present:

- `prompt` cannot be injected through YAML — the private `Frontmatter` struct has no
  `prompt` field (`crates/buzz-persona/src/persona.rs:176-196`), so a persona file cannot
  smuggle prompt content through a frontmatter key while presenting an innocuous body.
- The persona body and pack instructions are kept as **separate fields**, not concatenated
  (`crates/buzz-persona/src/resolve.rs:32-34`; asserted in
  `crates/buzz-persona/tests/integration.rs:394-407`), leaving the consumer able to frame
  or label each layer. `crates/buzz-acp/src/pool.rs:1122-1127` shows the consumer does
  frame them (`[Base]` / `[System]` markers) so the boundary stays recoverable downstream.
- Unknown frontmatter keys are hard-rejected (`deny_unknown_fields` at
  `crates/buzz-persona/src/persona.rs:175`), so a pack cannot introduce new
  behavior-bearing keys that a future version might honor.

Assessment: prompt content is treated as trusted-by-installation. The trust boundary is
"who may place a pack directory on disk", and this crate does not attempt to reduce it.
`crates/buzz-persona/PERSONA_PACK_SPEC.md:1035-1043` states this explicitly as an
operational requirement ("Only install packs from trusted sources").

---

### 3. Path traversal — implemented defenses

`safe_resolve` (`crates/buzz-persona/src/pack.rs:323-365`) is a three-layer check, and its
own doc calls it "Defense-in-depth" (`:317-322`):

| Layer | Check | Line |
|---|---|---|
| 0 | Reject absolute paths — Unix `/` prefix, and on Windows a drive-letter `X:` prefix | `crates/buzz-persona/src/pack.rs:330-337` |
| 1 | Reject any `Component::ParentDir` (`..`) **before** canonicalization | `crates/buzz-persona/src/pack.rs:339-345` |
| 2 | `canonicalize()` the joined path (resolves symlinks) | `crates/buzz-persona/src/pack.rs:357-360` |
| 3 | Require `canonical.starts_with(pack_root)` | `crates/buzz-persona/src/pack.rs:361-364` |

Applied to: `personas[]` entries (`crates/buzz-persona/src/pack.rs:156`),
`pack_instructions` (`:171`), `mcp_config` (`:199`).

Test coverage: `../../etc/passwd` (`crates/buzz-persona/src/pack.rs:561-568`),
`..` in the middle of a path (`:570-575`), symlink escaping the pack root
(`:589-607`, `#[cfg(unix)]`), and the traversal case through the validator
(`crates/buzz-persona/src/validate.rs:496-517`).

Non-existent paths short-circuit before canonicalization and return the lexical join
(`crates/buzz-persona/src/pack.rs:349-355`). This is safe because layers 0 and 1 already
ran; the comment says so ("The `..` check above already guards against traversal").

`pack_root` itself is canonicalized once up front (`crates/buzz-persona/src/pack.rs:126`),
so the prefix comparison is symlink-stable.

---

### 4. Path traversal — gaps

| # | Gap | Evidence | Consequence |
|---|---|---|---|
| S1 | **Hook paths are stored raw and never validated.** The code documents this as an accepted, deferred risk. | `resolve_hooks` at `crates/buzz-persona/src/resolve.rs:339-357`; verbatim doc: "Security: we intentionally do NOT resolve these to absolute paths. Hook paths come from untrusted persona frontmatter and could contain `../` traversal. Since hooks are not executed in this PR, we store them as-is. The PR that wires execution MUST validate through `safe_resolve()` before use." | A persona declaring `on_start: "../../../bin/anything"` parses cleanly and the string reaches `ResolvedPersona.hooks`. Currently inert because nothing executes it — the risk transfers to whichever consumer wires execution. There is no compile-time or type-level marker preventing that consumer from using the string directly. Test `crates/buzz-persona/src/resolve.rs:577-590` locks in the raw-string behavior. |
| S2 | **Persona `skills` paths are never validated.** `LoadedPersona.skills` is `pc.skills` verbatim (`crates/buzz-persona/src/pack.rs:434`) and `ResolvedPersona.skills` is `lp.skills.clone()` (`crates/buzz-persona/src/resolve.rs:249`) — neither goes through `safe_resolve`. | `crates/buzz-persona/src/persona.rs:123` (raw `Vec<String>`) | `skills: ["../../secrets"]` survives resolution. The only normalization is `resolve_skills`/`advisory_check_skill_names` taking the final path component (`crates/buzz-persona/src/pack.rs:253-260`, `crates/buzz-persona/src/validate.rs:365-369`), which happens to defuse traversal *for those two functions only* — `ResolvedPersona.skills` still carries the raw string. |
| S3 | **`avatar` path is unvalidated.** Documented as "Pack-relative path to avatar image" (`crates/buzz-persona/src/persona.rs:109`) but never resolved or checked. | `crates/buzz-persona/src/persona.rs:110`; passthrough at `crates/buzz-persona/src/pack.rs:425` and `crates/buzz-persona/src/resolve.rs:243` | Any consumer that loads the avatar from this string must do its own containment check. |
| S4 | The `skills/` enumeration in `resolve_skills` and `advisory_check_skill_names` uses `pack_dir` as passed by the caller, **not** the canonicalized root, and does not re-verify containment. | `crates/buzz-persona/src/pack.rs:276` (`pack_dir.join("skills")`), `crates/buzz-persona/src/validate.rs:375` | Low impact — the joined literal `"skills"` cannot escape — but it means the two functions are not protected by the canonicalization performed in `load_pack`. |

---

### 5. Credential / secret handling

| Finding | Evidence |
|---|---|
| MCP `env` maps are the credential channel, and this crate stores their values **verbatim** — including `${VAR}` placeholders, which are *not* expanded | `McpServerConfig.env: HashMap<String,String>` (`crates/buzz-persona/src/persona.rs:78`); passthrough at `crates/buzz-persona/src/resolve.rs:325-333`; explicit doc "Env values are passed through as literals — no `${VAR}` interpolation" (`:274`); test asserts `env["SECRET"] == "${MY_SECRET}"` (`crates/buzz-persona/src/resolve.rs:558-572`) |
| Consequence: a pack **can** hard-code a literal secret in `env` and this crate will carry it into `ResolvedMcpServer.env` unchanged, with no detection, warning, or redaction | same call sites; no secret-pattern scanning anywhere in the crate |
| No `Debug` redaction — `ResolvedMcpServer` derives plain `Debug` (`crates/buzz-persona/src/resolve.rs:67`) and `ResolvedPersona` likewise (`:22`), so `{:?}` on a resolved persona prints all MCP env values in the clear | derives at `crates/buzz-persona/src/resolve.rs:22`, `:67`; `McpServerConfig` also derives `Debug` (`crates/buzz-persona/src/persona.rs:68`) |
| The crate never reads the process environment, so it cannot leak host secrets into pack data | no `std::env` usage in `crates/buzz-persona/src/` (verified by grep — the only `std::env` mentions are prose in `PERSONA_PACK_SPEC.md`); design intent stated at `crates/buzz-persona/src/resolve.rs:11` and `:360-361` |
| The crate never writes files or opens sockets, so it cannot exfiltrate | no `fs::write`/`create`/network APIs outside `#[cfg(test)]` |
| `runtime_env_vars` emits only model/provider/temperature/context-limit keys — no credential keys are synthesized | `crates/buzz-persona/src/resolve.rs:365-397` |

Note on the interpolation gap direction: because `${VAR}` is **not** expanded here, this
crate does not read secrets out of the environment. The residual risk is the inverse —
literal secrets committed into a pack file are passed through silently.

---

### 6. Capability / tool-permission model

| Finding | Evidence |
|---|---|
| There is **no** permission, capability, allow-list, or sandbox model in this crate. No `permissions`, `allowed_tools`, `tools`, `sandbox`, or `permission_mode` field exists on any type. | complete field inventories: `crates/buzz-persona/src/persona.rs:101-170`, `crates/buzz-persona/src/manifest.rs:79-121`, `crates/buzz-persona/src/resolve.rs:23-65` |
| Agent capability is granted **transitively** through MCP server definitions: a persona supplies `command` + `args` + `env`, which is a specification of an arbitrary subprocess for the runtime to launch. There is no allow-list of permitted commands, no path restriction on `command`, and no argument filtering. | `McpServerConfig` (`crates/buzz-persona/src/persona.rs:70-79`); merge preserves `command`/`args` unchanged (`crates/buzz-persona/src/resolve.rs:311-340`) |
| Collision policy amplifies this: a per-persona MCP entry **wholly replaces** a same-named pack-level entry, including its `command`. A persona can therefore repoint a pack's trusted server name at a different binary. | `crates/buzz-persona/src/resolve.rs:293-300`; test `mcp_merge_persona_wins_on_collision` (`:513-535`) confirms `command` is replaced (`old-cmd` → `new-cmd`) |
| Operator-owned limits are deliberately **out of pack authors' reach**: `idle_timeout`, `max_turn_duration`, `agents`, `heartbeat_interval`, `permission_mode` are all rejected by `deny_unknown_fields`. The test names the intent: "This documents the security boundary: pack authors define behavior, operators define limits." | `crates/buzz-persona/tests/integration.rs:632-650` |
| Operator env-var precedence (level 1) is **not** enforced here — the crate produces the desired env vars and documents that the consumer must skip injection when the operator already set a key. | `crates/buzz-persona/src/resolve.rs:359-364`; consumer field at `crates/buzz-acp/src/config.rs:533-535` |
| Persona `runtime` selects which env-var namespace is emitted, and any unrecognized value silently falls into the GOOSE_* branch rather than being rejected. | wildcard arm at `crates/buzz-persona/src/resolve.rs:380-386` |

---

### 7. Input-validation gaps (non-path)

| # | Gap | Evidence | Notes |
|---|---|---|---|
| V1 | `temperature` has no range validation — any `f64` including negatives, `inf` (via YAML `.inf`), or absurd magnitudes is accepted and projected into `GOOSE_TEMPERATURE` via `to_string()` | `crates/buzz-persona/src/persona.rs:151`; `crates/buzz-persona/src/merge.rs:96`; `crates/buzz-persona/src/resolve.rs:391-393` | consistent with the spec, which states buzz-acp "passes through without range validation" (`crates/buzz-persona/PERSONA_PACK_SPEC.md:815`) |
| V2 | `max_context_tokens` has no upper bound (any `u64`) | `crates/buzz-persona/src/persona.rs:154`; projected at `crates/buzz-persona/src/resolve.rs:395-397` | |
| V3 | `model` string is unvalidated apart from the colon split. An empty provider (`":model"`) is filtered to `None` (`crates/buzz-persona/src/resolve.rs:213`) but env-var **values** are otherwise arbitrary strings — newlines, `=`, shell metacharacters are all passed through into `runtime_env_vars` | `crates/buzz-persona/src/resolve.rs:369-387` | The consumer injects these via `Command::env()` per `crates/buzz-persona/PERSONA_PACK_SPEC.md:1145`, which is not shell-interpreted, so injection risk depends on the consumer not routing them through a shell. |
| V4 | `subscribe` channel names are unvalidated — no length, charset, or count limit; the `#` prefix is not stripped in this crate | `crates/buzz-persona/src/merge.rs:106-121`; spec says the consumer strips `#` (`crates/buzz-persona/PERSONA_PACK_SPEC.md:824-828`) | |
| V5 | Manifest `id`, `name`, `version` are only checked for presence and non-emptiness — `id` has no reverse-DNS/charset validation despite being used as a directory name by consumers (`~/.buzz/packs/<pack-id>/` per `crates/buzz-persona/PERSONA_PACK_SPEC.md:995`) | `crates/buzz-persona/src/manifest.rs:155-170` | A pack `id` containing `/` or `..` would be a traversal vector **in the consumer**, not here |
| V6 | `.mcp.json` is accepted as any valid JSON; a wrong shape contributes nothing and produces no diagnostic | `crates/buzz-persona/src/pack.rs:192-197`; `crates/buzz-persona/src/resolve.rs:283-291` | silent-empty, not an error |
| V7 | Persona `name` is validated for charset/length only on the **resolve** and **validate** paths, not in `parse_persona_md`. A direct `parse_persona_md` caller (e.g. the desktop import path) receives an unvalidated `name`. | validation at `crates/buzz-persona/src/resolve.rs:134-155` and `crates/buzz-persona/src/validate.rs:167-185`; absent from `crates/buzz-persona/src/persona.rs:208-260` | `desktop/src-tauri/src/migration.rs:1123` uses only `split_frontmatter`, bypassing even that |
| V8 | `name.len() > 64` is a **byte** comparison, not char count | `crates/buzz-persona/src/resolve.rs:146`, `crates/buzz-persona/src/validate.rs:169` | practically moot since the charset check already rejects non-ASCII |
| V9 | No total-pack limits: unbounded persona count, unbounded MCP servers per persona, unbounded keyword lists. Combined with the 256 KiB/persona body cap, a manifest listing thousands of personas is a memory-amplification path. | `crates/buzz-persona/src/pack.rs:155-164` loops over `manifest.personas` with no cap | Each individual file is capped; the aggregate is not. |
| V10 | Duplicate keys in the manifest `personas[]` array are not deduplicated before loading — the same file would be loaded twice, then caught by the duplicate-name check at `crates/buzz-persona/src/resolve.rs:125-133` (but only on the resolve/validate path, not on a bare `load_pack`) | `crates/buzz-persona/src/pack.rs:155-164` | `load_pack` alone will happily return two identical personas |

---

### 8. Denial-of-service considerations

| Finding | Evidence |
|---|---|
| Size caps exist at two layers: `read_bounded_file` stats before reading (`crates/buzz-persona/src/pack.rs:374-389`) and `parse_persona_file` stats before reading (`crates/buzz-persona/src/persona.rs:263-267`) — both avoid unbounded allocation on a huge file | limits `crates/buzz-persona/src/persona.rs:21`, `:24`; pack-level limit `crates/buzz-persona/src/pack.rs:153-154` |
| `instructions.md` and `.mcp.json` share the same ~1.25 MiB cap | `crates/buzz-persona/src/pack.rs:168-169`, used at `:180`, `:186`, `:205`, `:213` |
| The frontmatter close-delimiter scan is a forward-only loop with a monotonically advancing cursor — no backtracking, no quadratic blowup | `crates/buzz-persona/src/pack.rs` n/a; `crates/buzz-persona/src/persona.rs:289-306` (`search_from = after_dashes`) |
| YAML parsing is delegated to `serde_yaml` 0.9 within the 1 MiB frontmatter cap. No explicit alias/anchor-expansion guard exists in this crate; billion-laughs resistance depends entirely on `serde_yaml` behavior — **unverified** here. | `crates/buzz-persona/src/persona.rs:220` |
| `read_dir` on `skills/` is unbounded in entry count | `crates/buzz-persona/src/pack.rs:278-293`, `crates/buzz-persona/src/validate.rs:377-386` |
| No recursion in any parse path — no risk of stack exhaustion from nested structures other than what `serde_yaml`/`serde_json` impose internally | manual review of all parse functions |

---

### 9. Symlink / TOCTOU notes

- Symlink escape is caught by canonicalization + prefix check
  (`crates/buzz-persona/src/pack.rs:357-364`), with a Unix test at `:589-607`.
- There is a small TOCTOU window: `exists()` is checked (`crates/buzz-persona/src/pack.rs:157`,
  `:172`, `:200`), then `metadata()` (`:375`), then `read_to_string()` (`:367`) — three
  separate syscalls on the same path. A path replaced between the checks (e.g. file → symlink)
  could bypass the size check on the read. Exploiting it requires write access to the pack
  directory, which already implies control of the persona content.
- The advisory validation pass re-reads `plugin.json` from disk twice more
  (`crates/buzz-persona/src/validate.rs:213`, `:305`) after `load_pack` already read it,
  so the validator can in principle report on different content than was loaded.

---

### 10. Summary of security findings

| ID | Finding | Severity signal (factual, not graded) |
|---|---|---|
| S1 | Hook paths stored unvalidated, traversal-capable, execution deferred to consumer | documented-and-accepted in code (`crates/buzz-persona/src/resolve.rs:339-346`) |
| S2 | Persona `skills` paths unvalidated in `ResolvedPersona.skills` | no containment check on the emitted value |
| S3 | `avatar` path unvalidated | consumer-dependent |
| PI1 | Persona prompt/instructions/description flow into agent context verbatim | by design; size-capped only |
| PI2 | `SKILL.md` content never inspected despite being named a prompt-injection vector in the crate's own spec | `crates/buzz-persona/PERSONA_PACK_SPEC.md:1041-1043` |
| C1 | MCP `env` values (potential secrets) stored verbatim, `Debug`-printable, no redaction | `crates/buzz-persona/src/resolve.rs:325-333`, derives at `:67` |
| C2 | Literal secrets committed in a pack are neither detected nor warned about | no scanning logic |
| P1 | Persona-level MCP entry can replace a pack-level server's `command` entirely | `crates/buzz-persona/src/resolve.rs:293-300` |
| P2 | No tool-permission or capability model; capability = arbitrary MCP subprocess spec | complete field inventory |
| V1–V10 | Input-validation gaps enumerated in §7 | — |
| — | No `unsafe`, no panics in production paths, no env reads, no writes, no network | verified by grep |


## Module: buzz-ws-client (`crates/buzz-ws-client`)

### Aspect: Security

Findings are limited to what these five files contain. Where behaviour is delegated
to `nostr` or `tokio-tungstenite`, that is stated explicitly rather than assumed.

---

### 1. Key handling (AUTH event signing)

| Observation | Evidence |
|---|---|
| Keys are always **borrowed**, never owned or stored: `&Keys` on `connect_authenticated`, `authenticate`, `publish_event`, `build_auth_event` | `connection.rs:39`, `:72`, `:280`; `message.rs:177` |
| `NostrWsConnection` holds **no key material** — its four fields are the stream, a message buffer, the last challenge, and the URL | `connection.rs:26`–`31` |
| No key is cloned, copied into a `String`, or written to a struct anywhere in the crate (verified across all 541 lines) | — |
| Signing is delegated to `nostr`: `EventBuilder::auth(...).sign_with_keys(keys)`; no manual Schnorr/secp code, no nonce generation here | `message.rs:181`, `:188` |
| Key material is never logged. The three `debug!` sites log the relay URL, a success message, and outbound frame text | `connection.rs:57`, `:91`, `:123` |
| **The outbound-frame log includes the full signed AUTH event and any `auth_tag` token**: `debug!("→ relay: {text}")` runs for every `send_raw`, including `["AUTH", <event>]` and any bearer-style tag value the caller passed | `connection.rs:82` → `:121`–`:123`; tag injected at `message.rs:182`–`186` |
| The signed AUTH event contains a public key and signature only (no secret), so the logging exposure is the authorization tag / challenge, not the private key | `message.rs:174`–`190` |

---

### 2. TLS verification

| Question | Answer from code | Evidence |
|---|---|---|
| Is certificate validation ever disabled? | **No.** There is no `Connector`, no `connect_async_tls_with_config`, no `danger_*`/`accept_invalid_*` option, and no custom verifier in the crate. A search for `connect_async_tls_with_config`, `Connector`, `accept_invalid`, `rustls`, `native_tls` inside `crates/buzz-ws-client/` returns zero matches. | `connection.rs:53` is the only dial site |
| What TLS stack is used? | Whatever `tokio-tungstenite`'s default `connect_async` selects for the enabled feature set. The workspace enables `rustls-tls-webpki-roots` | root `Cargo.toml:113`; inherited at `crates/buzz-ws-client/Cargo.toml:12` |
| Which trust anchors? | Bundled webpki roots (per that feature name), i.e. **not** the OS trust store — a relay signed by a private/corporate CA in the system store would not validate unless the CA is in webpki-roots. This follows from the feature selection, not from code in this crate. | root `Cargo.toml:113` |
| Certificate pinning / hostname override? | None. | — |
| Is plaintext `ws://` allowed? | **Yes.** `connect` performs only `url::Url` parsing with no scheme check, so `ws://` (and even `http://`, subject to `tokio-tungstenite`'s own handling) is accepted; TLS is opportunistic via `MaybeTlsStream` | `connection.rs:48`–`55`, `:14` |
| Any warning emitted for plaintext? | No — the debug log is scheme-agnostic | `connection.rs:57` |

Net: TLS is used correctly with defaults when the caller supplies `wss://`, and is
silently absent when the caller supplies `ws://`. Scheme enforcement is a caller
responsibility; nothing in this crate constrains it.

---

### 3. Challenge / replay handling

| Control | Present? | Evidence |
|---|---|---|
| Challenge length bound (1024 bytes) | Partially — enforced only on the socket-read path | enforced `connection.rs:198`–`202`; **not** enforced when the challenge comes from `pending_challenge` (`:161`–`163`) or from the buffered-message path (`:170`–`171`) |
| Challenge content validation (charset, hex/base64 shape, minimum entropy) | No | `connection.rs:197`–`203`; `message.rs:139`–`146` |
| Replay protection — rejecting a previously seen challenge | No; `pending_challenge` is a single slot that is overwritten by each new `AUTH`, with no history | `connection.rs:144`, `:256` |
| Binding of challenge to relay identity | Indirect: the AUTH event embeds the relay URL the client dialed, via `RelayUrl::parse(self.relay_url)` | `connection.rs:79` → `message.rs:180` |
| Relay URL used for the AUTH tag is the caller's raw string, not the normalized URL actually dialed | Yes — divergence is possible (e.g. trailing-slash/default-port normalization) | store `connection.rs:63`; dial `connection.rs:53` (normalized); AUTH tag `connection.rs:79` (raw) |
| Timestamp tolerance / clock-skew window | **None in this crate.** `created_at` is set inside `nostr::EventBuilder::auth`; there is no `Timestamp`, `now()`, or tolerance constant anywhere in these files. Any freshness window is enforced by the relay, not the client | `message.rs:181` |
| Are relay-provided event signatures verified on inbound `EVENT`? | Not by this crate. `serde_json::from_value::<Event>` deserializes; whether the `nostr` crate verifies the signature during deserialization is **not determinable from these files** | `message.rs:77`–`81` |
| Mid-session challenge auto-response | No — recorded and buffered only, so a relay-initiated re-auth is not answered unless the caller acts | `connection.rs:255`–`258` |

---

### 4. Input validation gaps (client-side, on relay-controlled data)

| Gap | Consequence as coded | Evidence |
|---|---|---|
| No cap on inbound frame size or on parsed field lengths (other than the 1024-byte challenge on one path) | Relay-controlled strings (`NOTICE`, `CLOSED`, `OK` reason) are copied into owned `String`s unbounded; no `WebSocketConfig` limits are set on the socket | `message.rs:94`–`98`, `:121`–`118`, `:132`–`129`; `connection.rs:53` |
| Unbounded `buffer` growth | A relay that streams non-matching messages while the client waits for an `OK` grows `VecDeque` until the deadline expires; there is no capacity limit or drop policy | `connection.rs:28`, `:205`, `:257`, `:259` |
| `OK.accepted` defaults to `false` on malformed input, `message` defaults to `""` | Fails closed for acceptance (safe direction), but a malformed `OK` is indistinguishable from an explicit rejection with no reason | `message.rs:93`, `:94`–`98` |
| `event_id` correlation is a raw string comparison of relay-supplied text against locally computed hex | Case/format mismatch would silently fail to correlate (leading to `Timeout` rather than a mismatch error); no length/hex validation of `arr[1]` | `message.rs:88`–`92`; compare `connection.rs:227`, `:254` |
| `send_raw` accepts any `serde_json::Value` and sends it verbatim | Callers can emit arbitrary frames, including a hand-rolled `AUTH`; the crate applies no schema or scheme checks on that path | `connection.rs:121`–`125` |
| No authenticated-state check before `send_event` | An event can be published on an unauthenticated connection; rejection handling is left to the relay's `OK` | `connection.rs:96`–`101` |
| Non-text frames ignored silently | A relay sending binary frames produces no error or log | `connection.rs:152`, `:212`, `:266` |
| Ping payload echoed back verbatim as Pong | Standard WebSocket behaviour, but there is no size limit applied by this crate | `connection.rs:149`, `:209`, `:263` |

---

### 5. Memory / unsafe

- `#![deny(unsafe_code)]` at the crate root (`lib.rs:1`).
- **Zero `unsafe` blocks** — verified by search across `crates/buzz-ws-client/`; the
  only match for the token `unsafe` is the deny attribute itself.
- No FFI, no raw pointers, no transmutes.
- Two panics are reachable only via internal invariants: `VecDeque::remove(...).unwrap()`
  after a successful `position` lookup (`connection.rs:170`, `:229`) and the paired
  `unreachable!()` arms (`connection.rs:172`, `:231`). Both are guarded by the
  preceding `position(...)` predicate on the same borrow, so relay input alone cannot
  trigger them; they remain panics-in-library-code rather than errors.

---

### 6. Secrets in error messages and logs

| Path | What is exposed | Evidence |
|---|---|---|
| `debug!("→ relay: {text}")` | Full outbound JSON, including the signed AUTH event and any `auth_tag` token value | `connection.rs:123` |
| `debug!("connected to relay at {url}")` | Relay URL as provided; if a caller embeds credentials in the URL userinfo, they would appear here (the crate does not strip userinfo) | `connection.rs:57` |
| `WsClientError::UnexpectedMessage(text.to_string())` | The entire raw relay frame is embedded in the error string and thus in any caller log | `message.rs:68`, `:75`, `:80`, `:91`, `:109`, `:119`, `:143` |
| `WsClientError::AuthFailed(ok.message)` | Relay-supplied rejection reason, propagated verbatim | `connection.rs:88` |

No redaction helper or `Display`-masking wrapper exists in the crate.


## Module: buzz-db (`crates/buzz-db`)

### Security

#### 1. Memory safety

`#![deny(unsafe_code)]` at `crates/buzz-db/src/lib.rs:1`. A search for `unsafe`
across `crates/buzz-db/src/**` returns three hits, all non-code: the lint
attribute itself, a doc comment at `crates/buzz-db/src/thread.rs:342`, and a code
comment at `crates/buzz-db/src/workflow.rs:1674`. **No `unsafe` blocks exist.**

#### 2. SQL injection surface

No user value is ever concatenated into SQL. All 15 `sqlx::AssertSqlSafe` sites
interpolate only: (a) compile-time literals, (b) generated `$n` placeholder
indices, (c) values already validated by an allowlist/validator, or (d) values
whose Rust type cannot carry SQL. Ten sites are production code, five are
test-only.

| Site | Interpolated content | Why it is safe | Values still bound? |
|------|----------------------|----------------|---------------------|
| `crates/buzz-db/src/channel.rs:870` (`get_accessible_channels`) | `membership_clause` chosen from two string literals (`:822-826`); the base template; an appended `AND c.visibility::text = $3` | no runtime data reaches the string | yes — `$1` community, `$2` pubkey, `$3` visibility (`:871-878`) |
| `crates/buzz-db/src/channel.rs:957` (`get_users_bulk`) | `"$2, $3, …"` list generated from `2..(len+2)` (`:942-945`) | digits only, derived from a length | yes — every pubkey bound in a loop (`:958-960`) |
| `crates/buzz-db/src/channel.rs:1107` (`update_channel`) | SET fragments from fixed literals + generated indices (`:1080-1105`) | column names are literals; only indices vary | yes — `:1110-1122` |
| `crates/buzz-db/src/thread.rs:430` (`get_thread_replies`) | `${param_idx}` indices only (`:387-400`) | digits only | yes — `:431-448` |
| `crates/buzz-db/src/thread.rs:631` (`get_channel_window`) | `${param_idx}` indices **and** `kind IN ({list})` where `list` is `&[u32]` joined via `to_string()` (`:617-627`) | `u32` cannot encode SQL; this is the only caller-supplied data interpolated in production | partially — community/channel/cursor/limit bound (`:632-643`); the kind list is inlined as decimal integers |
| `crates/buzz-db/src/user.rs:148` (`update_user_profile`) | SET fragments + generated indices (`:112-142`) | as above | yes — `:149-160` |
| `crates/buzz-db/src/usage.rs:281` (`active_user_counts`) | `INTERVAL '{interval_sql}'` where `interval_sql: &'static str` (`:252`) | `&'static str` signature keeps it a compile-time literal; documented "must be a trusted literal" at `:249-251` and re-stated on the `Db` wrapper at `crates/buzz-db/src/lib.rs:621-623` | no other value; this query takes no binds |
| `crates/buzz-db/src/usage.rs:323` (`active_channel_counts`) | same `&'static str` interval | same | same |
| `crates/buzz-db/src/partition.rs:130` (`ensure_partition`) | `partition_name`, `table_name`, two date strings — DDL, which cannot be parameterized | allowlist + three validators re-checked immediately before, `:84-105` | n/a (DDL) |
| `crates/buzz-db/src/lib.rs:5235`, `:5256`, `:6009`, `:6014` | scratch database names from `Uuid::new_v4().simple()` | test-only (`#[cfg(test)]`), UUID hex | n/a |
| `crates/buzz-db/src/replica_fence.rs:613`, `:638` | `CREATE ROLE`/`DROP ROLE` with a UUID-derived role name | test-only | n/a |

Additional non-interpolating dynamic-SQL surfaces, all fully parameterized:
`QueryBuilder` with `push_bind`/`separated`/`push_values`
(`crates/buzz-db/src/event.rs:360`, `:591`, `:836`, `:957`;
`crates/buzz-db/src/feed.rs:91`, `:150`, `:207`; `crates/buzz-db/src/lib.rs:146`;
`crates/buzz-db/src/channel.rs:1337`), and the e-tag JSONB containment predicate
which binds a `serde_json::Value` rather than string-building JSON
(`crates/buzz-db/src/event.rs:426-435`).

**DDL construction** exists in exactly one production place — the partition
manager. Its three-layer defence:

1. Table allowlist `PARTITIONED_TABLES = ["events", "delivery_log"]`
   (`crates/buzz-db/src/partition.rs:12`), re-checked inside `ensure_partition`
   even though the only caller iterates the same constant (`:84-88`).
2. `validate_partition_suffix` — non-empty, ASCII digits and `_` only
   (`:58-60`), enforced at `:89-93`.
3. `validate_date_str` — exactly 10 bytes, `-` at indices 4 and 7, digits
   elsewhere (`:63-73`), enforced at `:94-105`.

Negative unit tests pin all three, including the explicit injection payloads
`"2026_03; DROP TABLE events--"` and `"2026-03-01; DROP TABLE events--"`
(`crates/buzz-db/src/partition.rs:156-174`) and that `api_tokens`/`users` are
**not** in the allowlist (`:176-181`).

LIKE-metacharacter escaping: `escape_like` escapes `\`, `%`, `_` and the query
uses `ESCAPE '\'`, so a search for `"%"` cannot become a full-table wildcard
(`crates/buzz-db/src/user.rs:207-222`, `:246-256`).

#### 3. Secret handling

| Secret | Storage form | Where |
|--------|--------------|-------|
| Workflow approval tokens | SHA-256 digest only; `create_approval` takes the **raw** token and hashes it internally, so plaintext never reaches the DB layer | `crates/buzz-db/src/workflow.rs:33-36`, `:915-946`; hash-only lookup/update paths `:973-994`, `:1059-1092` |
| API tokens | SHA-256 digest supplied by the caller; DB CHECK enforces 32 bytes | `crates/buzz-db/src/api_token.rs:8-9`; `migrations/0001_initial_schema.sql:488` |
| DM participant sets | SHA-256 over sorted, deduped pubkeys | `crates/buzz-db/src/dm.rs:48-60` |
| Push endpoint identity | `endpoint_hash` (SHA-256, 32 bytes) plus an opaque `endpoint_grant`; both are **copied from the stored lease** on wake enqueue so a caller cannot redirect a wake | `migrations/0012_push_leases.sql:11-12`; `crates/buzz-db/src/push.rs:502-511`, `:650-701` |
| APNs device tokens | never in a buzz-db-managed tenant table: `push_gateway_installations.token_ciphertext` + `token_fingerprint` are operator-global and have no Rust module here | `migrations/0015_push_gateway_authority.sql:12-26` |
| AUTH events (kind 22242) | **never stored** — they carry bearer tokens | `crates/buzz-db/src/error.rs:19-22`; `crates/buzz-db/src/event.rs:264-266` |
| Community signing key | `communities.signing_key BYTEA` — no read/write helper in this crate | `migrations/0001_initial_schema.sql:56` |
| Default DB URL literal | `postgres://buzz:buzz_dev@localhost:5432/buzz` in `DbConfig::default()` and in test constants, each annotated `// sadscan:disable np.postgres.1` where scanned | `crates/buzz-db/src/lib.rs:240`; `crates/buzz-db/src/replica_fence.rs:512` |

`ApiTokenRecord` intentionally carries `token_hash`, and the doc comment tells
callers to strip it before returning data to clients
(`crates/buzz-db/src/api_token.rs:201-206`).

Privacy-by-storage: NIP-44-encrypted / p-gated kinds are excluded from the FTS
vector so they are unsearchable at the storage layer, not just filtered at query
time (`migrations/0001_initial_schema.sql:207-226`,
`migrations/0005_agent_turn_metric_fts.sql:1-33`,
`migrations/0014_push_lease_fts.sql:1-9`). Reporter identity, private moderation
reasons, and `matched_principal` are documented as mod/audit-only and are never
placed on a public surface by this crate (`migrations/0006_moderation.sql:9-12`,
`:88-113`).

#### 4. TOCTOU / race safety

| Hazard | Mitigation | Evidence |
|--------|------------|----------|
| Inviter loses their role between the role check and the member insert | whole sequence in one transaction | `crates/buzz-db/src/channel.rs:362-456` |
| Actor's role changes between check and removal | one transaction; last-owner count also inside it | `crates/buzz-db/src/channel.rs:461-529` |
| Two concurrent reaction adds both see "absent" | single `INSERT … ON CONFLICT DO UPDATE … WHERE removed_at IS NOT NULL` | `crates/buzz-db/src/reaction.rs:66-112` |
| Two concurrent token mints both pass a count check | count is a subquery inside the INSERT | `crates/buzz-db/src/api_token.rs:91-126` |
| Two concurrent agent-owner mints | conditional `UPDATE … WHERE agent_owner_pubkey IS NULL` (first-mint-wins) | `crates/buzz-db/src/user.rs:283-303` |
| Concurrent replaceable-event writers | per-`(community, kind, pubkey, coordinate)` transaction advisory lock; ordering checked inside the same tx | `crates/buzz-db/src/lib.rs:3320-3336`, `:3654-3664` |
| Stale NIP-43 snapshot overwriting a newer one | advisory lock acquired **before** reading membership, so the whole read-build-write cycle is serialized | `crates/buzz-db/src/lib.rs:3498-3524` |
| Owner removal race (read role, then delete) | conditional `DELETE … AND role <> 'owner'` / `AND role = $3`; the follow-up read is only for error classification | `crates/buzz-db/src/relay_members.rs:196-285` |
| Stale-owner overwrite during ownership transfer | owner rows locked `FOR UPDATE` + `expected_owner_pubkey` verified in-tx → `OwnerConflict` | `crates/buzz-db/src/relay_members.rs:453-474` |
| Two transfers/creates to the same recipient both passing the 3-community cap | per-recipient FNV-1a advisory lock shared by both paths | `crates/buzz-db/src/relay_members.rs:410-417`, `:444-450`; `crates/buzz-db/src/lib.rs:869-874` |
| Two pods firing the same cron instant | unique claim insert `(community, workflow, scheduled_for)` | `crates/buzz-db/src/workflow.rs:496-534` |
| Two pods delivering the same reminder | `UPDATE … WHERE delivered_at IS NULL`; release is compare-and-clear on the pod's own stamp | `crates/buzz-db/src/event.rs:1411-1472` |
| Two decisions on one approval | `AND status = 'pending'` in the UPDATE | `crates/buzz-db/src/workflow.rs:1043-1051` |
| Repo-name claim race | `INSERT … ON CONFLICT DO NOTHING RETURNING`; a vanished conflicting row is treated as taken, never granted | `crates/buzz-db/src/git_repo.rs:96-139` |
| Lost push wake when a lease activates concurrently with an event insert | per-community advisory lock: events take it **shared**, activations **exclusive**; plus a 120 s `received_at` backfill inside the activation tx | `migrations/0023_push_match_gate.sql:1-42`; `crates/buzz-db/src/push.rs:15-66`, `:239-243`, `:464-468` |
| Stale TTL prefetch (permanent→ephemeral transition mid-ingest) | deferred trigger reads `ttl_seconds` under a shared per-channel advisory key; `update_channel` takes the same key exclusive | `migrations/0024_event_ttl_refresh_shared_lock.sql:25-57`; `crates/buzz-db/src/channel.rs:1131-1147` |
| Orphan mention after a concurrent NIP-RS hard delete | mention insert takes `FOR KEY SHARE` on the live event, skips if gone; the Rust delete path deletes event-then-mentions in a fixed order | `migrations/0009_nip_rs_database_guards.sql:111-137`; `crates/buzz-db/src/lib.rs:3759-3781` |
| NIP-RS resurrection window between an existence check and uniqueness enforcement | the watermark trigger returns NULL for an exact coordinate replay instead of probing the payload | `migrations/0010_nip_rs_exact_replay_guard.sql:34-47` |
| Old relay binary hard-deleting a NIP-RS coordinate | BEFORE DELETE trigger refuses unless the transaction opted in via a transaction-local GUC | `migrations/0011_nip_rs_exact_tag_cardinality.sql:45-60`; opt-in `crates/buzz-db/src/lib.rs:3742-3747` (verified transaction-local by test `crates/buzz-db/src/lib.rs:4194-4218`) |
| Below-fence row committed by a long-held transaction | deferred constraint trigger re-evaluates `clock_timestamp()` **inside commit**, so holding the tx open cannot outrun the floor | `migrations/0021_created_at_fence_floor.sql:20-42`, `:44-74` |
| An armed GUC with no enforcing trigger opening the fence | probe gated on catalog **and** behavioural verification; migration also fails closed | `crates/buzz-db/src/lib.rs:449-462`; `crates/buzz-db/src/replica_fence.rs:145-330`; `crates/buzz-db/src/migration.rs:25` |
| A session advisory lock leaking back into the pool | the lock-owning connection is `detach()`ed and closed on guard drop | `crates/buzz-db/src/lib.rs:511-535` |

Lock-ordering discipline is explicit: push takes address → author → gate
(`crates/buzz-db/src/push.rs:239-243`); the TTL key is acquired at commit, after
any push-gate shared lock, and no path takes both domains exclusively
(`migrations/0024_event_ttl_refresh_shared_lock.sql:20-24`); the NIP-RS delete
path fixes event-before-mentions ordering
(`crates/buzz-db/src/lib.rs:3759-3762`).

#### 5. Tenant isolation

Storage-level guarantees (schema): every non-allowlisted table has
`community_id NOT NULL` and every PK/UNIQUE/FK/unique index on it leads with
`community_id`, asserted over the concatenation of all 24 migrations by
`crates/buzz-db/src/migration.rs:635-670`. `channels.community_id` is immutable
(trigger `migrations/0001_initial_schema.sql:115-128`) and migrations may not
re-tenant it (`crates/buzz-db/src/migration.rs:527-556`, `:672-688`).

Query-level guarantees: `CommunityId` is a required parameter of essentially
every read/write. Notable belt-and-braces cases:

- `event_mentions` joins carry the **community tuple**
  (`e.community_id = m.community_id AND e.id = m.event_id`) — a bare
  `e.id = m.event_id` would leak cross-community mentions
  (`crates/buzz-db/src/event.rs:364-377`, `:576-586`;
  `crates/buzz-db/src/feed.rs:92-100`; schema note
  `migrations/0001_initial_schema.sql:281-284`).
- API-token lookup filters `community_id` in the WHERE clause **in addition to**
  the unique index, so the property holds even if storage uniqueness is relaxed
  (`crates/buzz-db/src/api_token.rs:129-158`; `crates/buzz-db/src/lib.rs:2337-2340`).
- Reminder claim/release bind `community_id` because the same event id may exist
  in two communities (`crates/buzz-db/src/event.rs:1400-1409`, `:1394-1400`).
- `communities_of_channels` deliberately **omits** unknown channels from its map
  so the relay's `MissingLookup → ImplBug → CoverageBreach` chain stays
  non-vacuous (`crates/buzz-db/src/lib.rs:1041-1049`, test `:5053-5085`).

Deliberate, documented cross-community reads (audited exceptions):

| Path | Scope | Justification in code |
|------|-------|----------------------|
| `admin_moderation::{list_reports, get_report, list_feedback, get_feedback}` | deployment-global | "the only moderation repository allowed to omit a `CommunityId`" — `crates/buzz-db/src/admin_moderation.rs:1-8` |
| `product_feedback::list` | deployment-global | operator inbox; `community_id` is provenance only — `crates/buzz-db/src/product_feedback.rs:3-6`, `migrations/0017_product_feedback.sql:23-24` |
| all of `usage.rs` (11 fns) | deployment-global aggregates | Prometheus rollups — `crates/buzz-db/src/usage.rs:1-14` |
| `Db::list_communities_owned_by` | filtered by pubkey only | "operator-plane helper… callers must gate it on deployment-level operator auth" — `crates/buzz-db/src/lib.rs:710-716` |
| owner-count checks (`SELECT count(*) FROM relay_members WHERE pubkey=$1 AND role='owner'`) | intentionally cross-community | the 3-community cap is a per-pubkey property — `crates/buzz-db/src/lib.rs:894-899`; `crates/buzz-db/src/relay_members.rs:485-493` |
| `Db::{community_of_channel, communities_of_channels}` | resolve tenancy from a row | inputs are internal channel ids; the result **is** the community label — `crates/buzz-db/src/lib.rs:1006-1077` |
| `Db::{archive,unarchive}_community_owned_by`, `lookup_community_by_host*`, `get/set_community_icon`, `ensure_configured_community`, `create_community_with_owner` | act on `communities` itself | operator-global registry table |
| `workflow::list_all_enabled_workflows` | scheduler scan | returns each row's `community_id` and filters archived communities — `crates/buzz-db/src/workflow.rs:449-478` |
| `workflow::prune_scheduled_workflow_fires_before` | global DELETE by `claimed_at` | janitor concern — `migrations/0001_initial_schema.sql:464-466` |
| `channel::reap_expired_ephemeral_channels` | global UPDATE | joins `communities`, skips archived, returns `(community_id, host, channel_id)` — `crates/buzz-db/src/channel.rs:1387-1417` |
| `event::query_due_reminders` | global scan | joins `communities`, skips archived, returns per-row `community_id` + `host` — `crates/buzz-db/src/event.rs:1334-1380` |
| `push::{claim_due_match_batch, reap_exhausted_matches}` | global | the claim CTE selects ONE community then scopes everything else — `crates/buzz-db/src/push.rs:840-882`, `:925-941` |
| `relay_members::backfill_from_allowlist` | community-scoped, but reads `information_schema` | startup migration helper — `crates/buzz-db/src/relay_members.rs:542-574` |
| `Db::backfill_d_tags` | **global `UPDATE events`, no community filter** | idempotent maintenance over legacy NIP-33 rows — `crates/buzz-db/src/lib.rs:2806-2824`. This is the one tenant-table **write** with no community predicate; it only fills a NULL derived column. |

Cross-tenant isolation is regression-tested with identical shapes in two
communities: events (`crates/buzz-db/src/event.rs:1601`), channels
(`crates/buzz-db/src/channel.rs:1553`), users
(`crates/buzz-db/src/channel.rs:1499`), reactions
(`crates/buzz-db/src/lib.rs:4890`), allowlist (`crates/buzz-db/src/lib.rs:4948`),
feeds (`crates/buzz-db/src/feed.rs:311`, `:365`, `:415`), relay membership
(`crates/buzz-db/src/relay_members.rs:652`), archived identities
(`crates/buzz-db/src/archived_identities.rs:150`), git repo names
(`crates/buzz-db/src/git_repo.rs:334`), reminders
(`crates/buzz-db/src/event.rs:2224`), and product feedback
(`crates/buzz-db/src/product_feedback.rs:125`).

#### 6. Least privilege

- The crate assumes the relay's role can create partitions
  (`CREATE TABLE … PARTITION OF`), set GUCs, and take advisory locks. It does
  **not** issue `GRANT`/`REVOKE`, create roles, or manage extensions beyond
  `CREATE EXTENSION IF NOT EXISTS pgcrypto` in migration 0001.
- The fence probe needs `pg_monitor`-level visibility into `pg_stat_activity`.
  An unprivileged role sees NULL `state`/`xact_start` (and NULL `backend_type`),
  and the probe **fails closed** with
  `MaskedActivity { masked }` rather than silently `MIN()`-ing hidden rows away
  — the classification is explicitly ordered so masked rows are detected
  *before* any `backend_type` filter (`crates/buzz-db/src/replica_fence.rs:369-374`,
  `:414-441`). A dedicated test creates an unprivileged login role and asserts
  the failure (`crates/buzz-db/src/replica_fence.rs:595-644`).
- A "replica" URL pointed at a primary is rejected: the replay-LSN check is
  gated on `pg_is_in_recovery()` and NULL is an error, not "fresh"
  (`crates/buzz-db/src/replica_fence.rs:441-463`, test `:648-660`).
- Operational bypasses of the floor guard are named and bounded: sessions without
  the `buzz.created_at_floor` GUC (pg_restore, manual backfills) and
  `session_replication_role = replica` restores are outside the proof by design
  and require holding the fence closed for their duration
  (`migrations/0021_created_at_fence_floor.sql:26-42`;
  `crates/buzz-db/src/replica_fence.rs:38-42`).

#### 7. Input validation

| Validated | Where |
|-----------|-------|
| pubkey length = 32 bytes (channels, members, DMs) | `crates/buzz-db/src/channel.rs:96-101`, `:184-189`, `:355-360`; `crates/buzz-db/src/dm.rs:118-124`; DB CHECK `migrations/0001_initial_schema.sql:171` |
| `p`-tag pubkeys must be 64 lowercase-able hex chars; malformed ones are dropped with a debug log rather than poisoning the batch | `crates/buzz-db/src/lib.rs:127-142` |
| Channel name canonicalised + non-empty | `crates/buzz-db/src/channel.rs:103-106`, `:1063-1068` |
| Nil channel UUID rejected | `crates/buzz-db/src/channel.rs:191-195` |
| DM participant count 2–9 | `crates/buzz-db/src/dm.rs:107-117`, `:366-370` |
| `channel_add_policy` vocabulary | `crates/buzz-db/src/user.rs:374-380` |
| Mutually exclusive / dependent query params (`before_id` needs `until`; `global_only` vs `channel_id`) | `crates/buzz-db/src/event.rs:320-333` |
| Stored `kind` must fit `u16` on read; otherwise `InvalidData` | `crates/buzz-db/src/event.rs:484-486` |
| Timestamps convertible from Unix seconds; otherwise `InvalidTimestamp` | `crates/buzz-db/src/event.rs:281-283`; `crates/buzz-db/src/lib.rs:3316-3318` |
| Enum/status strings parsed strictly | `crates/buzz-db/src/workflow.rs:61-71`, `:103-116`, `:148-160`; `crates/buzz-db/src/moderation.rs:580-590` |
| Huddle-link candidate content capped at 512 bytes and 32 rows before JSON parsing | `crates/buzz-db/src/event.rs:146-152`, `:210-217` |
| NIP-RS coordinate shape: lowercase-hex 32 chars, exactly one `d`, exactly one `["t","read-state"]` | `crates/buzz-db/src/lib.rs:3672-3687`; DB mirror `migrations/0011_nip_rs_exact_tag_cardinality.sql:66-88` |
| Byte-length and range CHECKs on push/moderation/feedback columns | `migrations/0012_push_leases.sql:5-21`, `migrations/0015_push_gateway_authority.sql:14-21`, `migrations/0006_moderation.sql:20-26`, `migrations/0017_product_feedback.sql:7-13` |

Known validation gaps (not defects per se, but the boundary is elsewhere):

1. **`d_tag` length is not enforced on write.** `D_TAG_MAX_LEN = 1024` is defined
   (`crates/buzz-db/src/event.rs:138-140`) but never compared against anything in
   this crate; `extract_d_tag` explicitly preserves the full value and defers
   length enforcement to the ingest layer
   (`crates/buzz-db/src/event.rs:2051-2060`).
2. **`max_active_leases` is caller-supplied**, so the push quota ceiling is a
   relay-side policy value, not a DB invariant
   (`crates/buzz-db/src/push.rs:213-221`).
3. **Git repo-name quota is enforced by the caller**, not by the module
   (`crates/buzz-db/src/git_repo.rs:88-92`).
4. **`usage_active_*_counts` trust their `&'static str` interval.** The type
   prevents runtime user input, but nothing rejects a malformed literal at
   compile time (`crates/buzz-db/src/usage.rs:249-253`).
5. **`api_tokens.scopes`/`channel_ids` are unconstrained JSONB** — schema has no
   CHECK; malformed JSON surfaces as `InvalidData` on read
   (`crates/buzz-db/src/lib.rs:3876-3899`).
6. **`get_channel_window`'s kind filter is inlined, not bound** — safe because
   the parameter is `&[u32]`, but it is the only production interpolation of
   caller data (`crates/buzz-db/src/thread.rs:617-627`).
7. **`Db::backfill_d_tags` has no community predicate** — a global maintenance
   write (`crates/buzz-db/src/lib.rs:2810-2824`).
8. **`api_token::list_tokens_by_owner` has no `LIMIT`** — unbounded result set
   for a single owner (`crates/buzz-db/src/api_token.rs:208-266`).


## Module: buzz-auth (`crates/buzz-auth`)

### Security

### Signature verification

Both auth paths delegate to one primitive: `buzz_core::verify_event`
(`crates/buzz-core/src/verification.rs:11-32`), which does two things in order:

1. `event.verify_id()` — recomputes the event id from
   `(pubkey, created_at, kind, tags, content)` and compares
   (`crates/buzz-core/src/verification.rs:13-25`). This is what binds the tags
   (challenge, relay, `u`, method, payload) to the signature.
2. `event.verify_signature()` — Schnorr verification
   (`crates/buzz-core/src/verification.rs:27-29`).

Call sites: `crates/buzz-auth/src/nip42.rs:56` and
`crates/buzz-auth/src/nip98.rs:74`.

| Property | State | file:line |
|----------|-------|-----------|
| Kind checked before signature | yes, NIP-42 (`kind != Authentication` → reject) | `crates/buzz-auth/src/nip42.rs:52-54` |
| Kind checked before signature | yes, NIP-98 (`kind != HttpAuth` → reject) | `crates/buzz-auth/src/nip98.rs:66-71` |
| Signature checked before tag comparisons | yes, both paths | `crates/buzz-auth/src/nip42.rs:56` vs `:58-83`; `crates/buzz-auth/src/nip98.rs:74` vs `:89-127` |
| CPU-bound verify offloaded | NIP-42 yes (`spawn_blocking` in `AuthService::verify_auth_event`) | `crates/buzz-auth/src/lib.rs:128-132` |
| CPU-bound verify offloaded | NIP-98 **no** — `verify_nip98_event` is a sync fn and this crate never wraps it | `crates/buzz-auth/src/nip98.rs:55`; the `spawn_blocking` requirement is documented upstream at `crates/buzz-core/src/verification.rs:1-2` |
| Error detail on failure | NIP-42 collapses everything to `InvalidSignature` (no oracle) | `crates/buzz-auth/src/nip42.rs:53`, `:56` |
| Error detail on failure | NIP-98 returns descriptive strings including the offending URL/method values | `crates/buzz-auth/src/nip98.rs:98-100`, `:111-113` — doc warns not to forward verbatim to clients (`:53-54`) |

---

### Replay protection

| Path | Replay protection in this crate | file:line |
|------|--------------------------------|-----------|
| NIP-42 | Single-use is achieved indirectly: the challenge is compared against a per-connection value the relay holds, and the relay refuses AUTH once the connection is `Authenticated` or `Failed`. This crate only performs the equality check | `crates/buzz-auth/src/nip42.rs:64-66`; state machine at `crates/buzz-relay/src/handlers/auth.rs:45-68` |
| NIP-98 | **Not** performed by the verifier. `verify_nip98_event` has no seen-set; the module doc states this explicitly: "It does **not** check whether the same event id has already been used" | `crates/buzz-auth/src/nip98_replay.rs:4-8` |

The replay contract is expressed as the `Nip98ReplayGuard` trait
(`crates/buzz-auth/src/nip98_replay.rs:64-104`) with these security-relevant
obligations:

- **Verify-then-mark ordering.** Marking before verifying would let an attacker
  who can predict a victim's future event id burn the slot and DoS the legitimate
  request (`crates/buzz-auth/src/nip98_replay.rs:19-27`).
- **Fail closed on error.** "On `Err` (Redis unreachable, etc.) callers MUST fail
  closed" (`crates/buzz-auth/src/nip98_replay.rs:83-86`).
- **Atomic set-if-absent.** "a read-then-write sequence loses to concurrent
  inserts and forfeits the freshness proof"
  (`crates/buzz-auth/src/nip98_replay.rs:88-90`).
- **TTL bounds.** floor 120s = 2× the ±60s tolerance
  (`crates/buzz-auth/src/nip98_replay.rs:46`), ceiling 3600s to stay inside Redis
  `EX`'s i64 range (`crates/buzz-auth/src/nip98_replay.rs:59`).
- **No process-local caching** — with multiple relay pods an in-process cache does
  not carry freshness across pods (`crates/buzz-auth/src/nip98_replay.rs:6-10`).

The production implementation is `RedisNip98ReplayGuard`
(`crates/buzz-pubsub/src/nip98_replay.rs:34`), held by the relay as
`Arc<dyn Nip98ReplayGuard>` (`crates/buzz-relay/src/state.rs:582`) and invoked on
the HTTP bridge paths (`crates/buzz-relay/src/api/bridge.rs:766`, `:956`,
`:1387`). This crate itself ships only `AlwaysFreshReplayGuard`
(`crates/buzz-auth/src/nip98_replay.rs:126-139`), cfg-gated to tests.

---

### Timestamp tolerance

| Path | Constant | Value | Comparison | file:line |
|------|----------|-------|-----------|-----------|
| NIP-42 | `TIMESTAMP_TOLERANCE_SECS` | 60 | `now.abs_diff(created_at) > 60` → reject | const `crates/buzz-auth/src/nip42.rs:35`, check `:78-83` |
| NIP-98 | `TIMESTAMP_TOLERANCE_SECS` | 60 | same | const `crates/buzz-auth/src/nip98.rs:32`, check `:78-85` |

`abs_diff` makes both windows symmetric, so future-dated events are rejected too.
The two constants are separate private definitions with the same name and value —
drifting one would silently desynchronise the pair, and the NIP-98 replay TTL
floor (`DEFAULT_REPLAY_TTL_SECS = 120`) is derived from the NIP-98 value by hand
(`crates/buzz-auth/src/nip98_replay.rs:42-45`). The 120-vs-tolerance relationship
is asserted only against the literal `120`, not against
`nip98::TIMESTAMP_TOLERANCE_SECS` (`crates/buzz-auth/src/nip98_replay.rs:210-218`).

---

### Constant-time comparisons

None. Every security-relevant comparison in the crate is a short-circuiting
equality:

| Comparison | Operator | file:line |
|-----------|----------|-----------|
| Challenge vs expected challenge | `challenge != expected_challenge` (`&str` `PartialEq`) | `crates/buzz-auth/src/nip42.rs:64` |
| Relay URL | `normalize_relay_url(a) != normalize_relay_url(b)` (`String` `PartialEq`) | `crates/buzz-auth/src/nip42.rs:74` |
| NIP-98 URL | `normalize_url(a) != normalize_url(b)` | `crates/buzz-auth/src/nip98.rs:97` |
| NIP-98 method | `eq_ignore_ascii_case` | `crates/buzz-auth/src/nip98.rs:110` |
| NIP-98 payload hash | `computed_hex != payload_hex` (hex `String` compare) | `crates/buzz-auth/src/nip98.rs:122` |

The `subtle` crate is in the workspace (`Cargo.toml:72` of `buzz-relay` uses it)
but is **not** a dependency of `buzz-auth`
(`crates/buzz-auth/Cargo.toml:14-26`). Practical exposure is limited: the
challenge is a 256-bit CSPRNG value the attacker must match exactly, and the
payload hash is verified against an already-signature-bound tag, so a timing
oracle on either yields little. It is nonetheless a deviation from
constant-time-compare hygiene and is not documented as a considered tradeoff
anywhere in the crate.

---

### Dev-only key derivation and exactly what gates it

```rust
#[cfg(any(test, feature = "dev"))]                       // crates/buzz-auth/src/lib.rs:159
pub fn derive_pubkey_from_username(username: &str) -> Result<nostr::PublicKey, AuthError> {
    let seed = format!("buzz-test-key:{username}");      // crates/buzz-auth/src/lib.rs:162
    let hash: [u8; 32] = Sha256::digest(seed.as_bytes()).into();   // :163
    let secret_key = nostr::SecretKey::from_slice(&hash) ... ;      // :164
    Ok(nostr::Keys::new(secret_key).public_key())                   // :166
}
```

The derivation is `SHA-256("buzz-test-key:{username}")` used directly as secret
key material. The doc states the risk plainly: "The derived keys are
deterministic and predictable from the username alone. Any attacker who knows a
username can compute the corresponding private key"
(`crates/buzz-auth/src/lib.rs:157-158`).

Gate chain, verified end to end:

| Link | State | file:line |
|------|-------|-----------|
| The `dev` cargo feature exists in `buzz-auth` and is **not** in any default set (there is no `[features] default = ...`) | confirmed | `crates/buzz-auth/Cargo.toml:10-12` |
| `buzz-relay` declares a passthrough feature `dev = ["buzz-auth/dev"]`, also not default | confirmed | `crates/buzz-relay/Cargo.toml:83-84` |
| `buzz-relay`'s **normal** dependency on `buzz-auth` requests no features | confirmed | `crates/buzz-relay/Cargo.toml:22` |
| `buzz-relay`'s **`[dev-dependencies]`** entry requests `features = ["dev"]` | confirmed | `crates/buzz-relay/Cargo.toml:90` |
| Workspace uses `resolver = "2"`, under which dev-dependency features are not unified into a non-test build | confirmed | `Cargo.toml:32` |
| Production container build passes no feature flags | confirmed | `Dockerfile:67-69` (`cargo build --release --locked -p buzz-relay --bin buzz-relay ...`) |
| Any `--features dev` / `--features buzz-auth/dev` invocation in `justfile`, CI, or Dockerfiles | none found (repo-wide grep) | — |

Conclusion: `derive_pubkey_from_username` is **not** compiled into the production
relay binary. It is compiled when building `buzz-relay`'s tests (via the
dev-dependency feature at `crates/buzz-relay/Cargo.toml:90`) and when building
`buzz-auth`'s own tests. Residual risk: the `dev` feature is one CLI flag away
from a release build, and the dev-dependency declaration means the code path is
compiled and linked in every `cargo test` run. Also note the function has **no
caller anywhere in the repo** — a repo-wide grep for
`derive_pubkey_from_username` matches only its definition
(`crates/buzz-auth/src/lib.rs:160`), so the risk is currently theoretical.

---

### Scope escalation surface

| Surface | Assessment | file:line |
|---------|-----------|-----------|
| NIP-42 grants all 16 known scopes to any pubkey that completes the handshake, including `AdminChannels` and `AdminUsers` | This is the widest grant in the crate. There is no tier, allowlist, or role input to the decision — only a valid signature over the right challenge | `crates/buzz-auth/src/lib.rs:136-142`; list `crates/buzz-auth/src/scope.rs:69-86` |
| Compensating control | Stated to be NIP-29 membership checks in the relay, not scopes: "Per-channel access is enforced by the relay's membership checks (NIP-29)" | `crates/buzz-auth/src/lib.rs:134-135` |
| `Scope::all_non_admin()` (14 scopes, admin excluded) exists and is documented as the dev-mode grant | never called anywhere in the repo — the admin-excluding path is dead | `crates/buzz-auth/src/scope.rs:94-111` |
| `Scope::Unknown(s)` | cannot satisfy a known-variant requirement (exact `PartialEq`), so an attacker-supplied scope string grants nothing | `crates/buzz-auth/src/scope.rs:60`, `:164`; check `crates/buzz-auth/src/access.rs:61` |
| Scope hierarchy | none — no wildcard, no implication. `admin:channels` does not imply `channels:write` | enumerated grant lists only, `crates/buzz-auth/src/scope.rs:69-111` |
| `AuthContext` mutability | all fields are `pub` with no invariant enforcement, so any consumer holding an owned/`mut` context can widen `scopes` or set `agent_owner_pubkey` | `crates/buzz-auth/src/lib.rs:64-80`; the relay does exactly this (`let Ok(mut auth_ctx)`, `crates/buzz-relay/src/handlers/auth.rs:91`) |
| `channel_ids` restriction field | declared but never enforced by any code in this crate; always `None` | `crates/buzz-auth/src/lib.rs:69-72`, `:139` |

---

### Input-validation gaps

| Gap | Detail | file:line |
|-----|--------|-----------|
| No length bound on `event_json` | `serde_json::from_str` runs on whatever the caller passes; memory bound is the caller's HTTP body limit | `crates/buzz-auth/src/nip98.rs:62` |
| No length bound on `username` in the dev derivation | unbounded `format!` then hash; low impact | `crates/buzz-auth/src/lib.rs:162` |
| `payload` tag not required | a request whose body is not covered by a `payload` tag still authenticates, so the signed claim binds URL+method only. Presence enforcement is delegated to the caller | `crates/buzz-auth/src/nip98.rs:117-127`; caller-side `require_payload` at `crates/buzz-relay/src/api/bridge.rs:99-112` |
| `payload` hex not format-validated | compared as a raw string; a non-hex or wrong-length tag simply fails the equality | `crates/buzz-auth/src/nip98.rs:122` |
| Unparseable URL falls back to string compare | NIP-42 compares raw strings verbatim (`crates/buzz-auth/src/nip42.rs:22`); NIP-98 lowercases first (`crates/buzz-auth/src/nip98.rs:148`). Both are fail-closed (mismatch → reject) but the two fallbacks differ |
| Scheme not normalised in NIP-42 | `ws://` and `wss://` are distinct strings; a downgrade cannot pass as an upgrade, so this is fail-closed, but it also means a relay that changes scheme rejects otherwise-valid AUTH | `crates/buzz-auth/src/nip42.rs:19-32` |
| Loopback aliasing asymmetry | NIP-42 collapses `localhost`/`::1` → `127.0.0.1` (`crates/buzz-auth/src/nip42.rs:25-29`); NIP-98 deliberately does **not**, because the `u`-tag host is the community binding (`crates/buzz-auth/src/nip98.rs:138-144`). The same asymmetry means the NIP-42 relay-tag check is a weaker host binding than NIP-98's under multi-tenant. Not flagged in the NIP-42 docs |
| `expected_url` trust | the verifier trusts whatever URL the caller reconstructs; the doc pushes reverse-proxy header handling (`X-Forwarded-Proto`/`X-Forwarded-Host`) onto the caller | `crates/buzz-auth/src/nip98.rs:40-41` |
| No rejection of duplicate tags | `tags.find(...)` takes the first match; a second `challenge`/`u`/`method` tag is ignored rather than rejected | `crates/buzz-auth/src/nip42.rs:58-60`, `:68-70`; `crates/buzz-auth/src/nip98.rs:89-94`, `:104-107` |

---

### Unsafe code

None. `#![deny(unsafe_code)]` is declared at `crates/buzz-auth/src/lib.rs:1`, and a
grep of the entire `crates/buzz-auth/` directory for the token `unsafe` returns
exactly that one line — no `unsafe` blocks, fns, traits, or impls, and no
`#[allow(unsafe_code)]` escape hatch.

---

### Randomness

`generate_challenge` draws 32 bytes from `rand::random::<[u8; 32]>()`
(`crates/buzz-auth/src/nip42.rs:39`) — 256 bits, hex-encoded to 64 chars, pinned
by a uniqueness + charset test (`crates/buzz-auth/src/nip42.rs:102-109`). No
seeded or deterministic RNG path exists in the crate.

---

### Rate limiting — verified state

**Verdict: `ARCHITECTURE.md` §9 item 2 is stale and materially wrong.** A
Redis-backed `RateLimiter` implementation exists, is constructed
unconditionally at relay startup, and is called before work is admitted on both
the WebSocket and HTTP surfaces in ordinary (non-feature-gated) production
builds. The `.env.example` heading "Shared Redis-backed admission limits"
(`.env.example:58`) is the accurate description.

#### Every `RateLimiter` implementor in the repo

Search: `impl .*RateLimiter` across all `*.rs` (including
`desktop/src-tauri/`), excluding `target/`. Three results, no more:

| Implementor | Location | Gate | Backing store |
|-------------|----------|------|---------------|
| `RedisRateLimiter` | `crates/buzz-pubsub/src/rate_limiter.rs:99-121` (struct `:88-90`) | **none — always compiled** | Redis via `deadpool_redis::Pool` (`crates/buzz-pubsub/src/rate_limiter.rs:89`) |
| `AlwaysAllowRateLimiter` | `crates/buzz-auth/src/rate_limit.rs:221-242` | `#[cfg(any(test, feature = "test-utils"))]` (`:218`, `:221`) | none (always allows) |
| `StubLimiter` | `crates/buzz-relay/src/admission.rs:69-96` | inside `#[cfg(test)] mod tests` (`crates/buzz-relay/src/admission.rs:47-48`) | none (test double) |

`desktop/src-tauri/` contains no `RateLimiter` implementor.

#### The Redis-backed limiter is real

`RedisRateLimiter::check_and_increment` builds the community-scoped key via
`buzz_auth::rate_limit::rate_limit_key`
(`crates/buzz-pubsub/src/rate_limiter.rs:108`) and runs an atomic Lua script that
`INCR`s, sets `EXPIRE` on first increment, and reads back the TTL
(`crates/buzz-pubsub/src/rate_limiter.rs:24-31`, invoked at `:50-56`).
Allow/deny boundary is `count <= limit`
(`crates/buzz-pubsub/src/rate_limiter.rs:74-78`). A missing-TTL key (crash
between INCR and EXPIRE in an older non-atomic version) is repaired with a fresh
`EXPIRE` and a warning (`crates/buzz-pubsub/src/rate_limiter.rs:58-72`).
`check_ip_connection` uses `ip_rate_limit_key`
(`crates/buzz-pubsub/src/rate_limiter.rs:118`).

#### It is wired into the relay

| Wiring step | file:line |
|-------------|-----------|
| `AppState` field `admission_rate_limiter: Arc<RedisRateLimiter>`, doc "Shared Redis-backed admission limits for ordinary HTTP and WebSocket work" | `crates/buzz-relay/src/state.rs:583-584` |
| Constructed at state build: `Arc::new(RedisRateLimiter::new(redis_pool.clone()))` — no cfg, no feature flag, no `Option` | `crates/buzz-relay/src/state.rs:712`, stored `:772` |
| Shared enforcement helper `check_principal<L: RateLimiter>` — `Ok` + `allowed` → admit; `Ok` + denied → `AdmissionError::Exceeded { reset_in_secs }`; `Err` → `AdmissionError::Unavailable` (fail closed) | `crates/buzz-relay/src/admission.rs:17-38` |
| WS burst budget helper: per-second limit × 5s window | `crates/buzz-relay/src/admission.rs:9`, `:40-45` |

#### Enforcement points — a check *is* called before work is admitted

**WebSocket.** `handle_text_message` parses the frame and immediately calls
`enforce_ws_admission`; on `false` it returns without dispatching
(`crates/buzz-relay/src/connection.rs:498-500`).
`enforce_ws_admission` (`crates/buzz-relay/src/connection.rs:594-653`):

- applies to `EVENT`, `REQ`, and `COUNT` frames; all other frame types
  (including `AUTH` and `CLOSE`) bypass
  (`crates/buzz-relay/src/connection.rs:599-602`);
- bypasses unauthenticated connections (`_ => return true`,
  `crates/buzz-relay/src/connection.rs:604-610`);
- first check: `LimitType::WsEvents` with a 5s window and
  `human_ws_events_per_sec × 5` budget
  (`crates/buzz-relay/src/connection.rs:612-623`);
- second check, EVENT frames only: `LimitType::Messages`, 60s window, limit =
  `agent_standard_messages_per_min` if `agent_owner_pubkey.is_some()` else
  `human_messages_per_min` (`crates/buzz-relay/src/connection.rs:632-650`);
- rejection emits `CLOSED` for REQ or `NOTICE` otherwise plus a
  `buzz_admission_rejections_total` metric
  (`crates/buzz-relay/src/connection.rs:655-679`).

**HTTP.** `enforce_http_admission` uses `LimitType::ApiCalls`, 60s window, limit
`human_api_calls_per_min`; denial → HTTP 429, limiter error → HTTP 503
(`crates/buzz-relay/src/api/bridge.rs:24-56`). Called on all three Nostr bridge
endpoints, before body parse:

| Endpoint | Call site |
|----------|-----------|
| `POST /events` (`submit_event_authed`) | `crates/buzz-relay/src/api/bridge.rs:760` |
| `POST /query` (`query_events_authed`) | `crates/buzz-relay/src/api/bridge.rs:955` |
| `POST /count` (`count_events_authed`) | `crates/buzz-relay/src/api/bridge.rs:1386` |

**Media upload.** Not covered by the Redis limiter. It uses a separate
in-process `DashMap` fixed-window counter, `upload_rate_limited`
(`crates/buzz-relay/src/api/media.rs:88-111`), over a 60s window
(`crates/buzz-relay/src/api/media.rs:66`) against `config.media_uploads_per_minute`
(`crates/buzz-relay/src/api/media.rs:95`), plus a concurrency permit
(`crates/buzz-relay/src/api/media.rs:113-137`). State field at
`crates/buzz-relay/src/state.rs:590-592`. Per-pod, not shared.

**Not enforced:** `LimitType::IpConnections` / `check_ip_connection`. The Redis
implementation exists (`crates/buzz-pubsub/src/rate_limiter.rs:112-120`) but the
only call sites repo-wide are the trait definition, the test stubs, and doc
references — no production path invokes it, so there is no per-IP connection cap.

#### Env var → consumer trace (all seven)

Every var is read by `rate_limit_config_from_env`
(`crates/buzz-relay/src/config.rs:284-316`) via `positive_u64_from_env`
(`crates/buzz-relay/src/config.rs:270-282`), which rejects zero and non-integers
with `ConfigError::InvalidValue` and falls back to the `RateLimitConfig::default()`
value when unset. The result lands in `config.auth.rate_limits`
(`crates/buzz-relay/src/config.rs:585`).

| Env var | Read at | Field | Consumed at | Enforced? |
|---------|---------|-------|-------------|-----------|
| `BUZZ_RATE_LIMIT_HUMAN_MESSAGES_PER_MIN` | `config.rs:287-290` | `human_messages_per_min` | `crates/buzz-relay/src/connection.rs:636` | yes — WS EVENT |
| `BUZZ_RATE_LIMIT_HUMAN_API_CALLS_PER_MIN` | `config.rs:291-294` | `human_api_calls_per_min` | `crates/buzz-relay/src/api/bridge.rs:29` | yes — HTTP `/events`, `/query`, `/count` |
| `BUZZ_RATE_LIMIT_HUMAN_WS_EVENTS_PER_SEC` | `config.rs:295-298` | `human_ws_events_per_sec` | `crates/buzz-relay/src/connection.rs:614` | yes — WS EVENT/REQ/COUNT |
| `BUZZ_RATE_LIMIT_AGENT_STANDARD_MESSAGES_PER_MIN` | `config.rs:299-302` | `agent_standard_messages_per_min` | `crates/buzz-relay/src/connection.rs:634` | yes — WS EVENT from NIP-OA agents |
| `BUZZ_RATE_LIMIT_AGENT_STANDARD_API_CALLS_PER_MIN` | `config.rs:303-306` | `agent_standard_api_calls_per_min` | **nowhere** | no |
| `BUZZ_RATE_LIMIT_AGENT_ELEVATED_MESSAGES_PER_MIN` | `config.rs:307-310` | `agent_elevated_messages_per_min` | **nowhere** | no |
| `BUZZ_RATE_LIMIT_AGENT_PLATFORM_MESSAGES_PER_MIN` | `config.rs:311-314` | `agent_platform_messages_per_min` | **nowhere** | no |

Repo-wide greps for the last three field names return only the `buzz-auth`
declaration/default and the `buzz-relay` config parser — no read site. CI sets
three of them to large values to keep integration tests from tripping the limits
(`.github/workflows/ci.yml:492-494`), which is itself evidence that enforcement
is live: a no-op limiter would not need raising.

#### Point-by-point scoring of the `ARCHITECTURE.md` claim

Claim text is at `ARCHITECTURE.md:823` (plus the same assertion at
`ARCHITECTURE.md:390`).

| Claim element | Verdict | Evidence |
|---------------|---------|----------|
| "`RateLimiter` trait exists in `buzz-auth`" | **TRUE** | `crates/buzz-auth/src/rate_limit.rs:168-194` |
| "Only implementation is `AlwaysAllowRateLimiter` (test stub…)" | **FALSE** | `RedisRateLimiter` at `crates/buzz-pubsub/src/rate_limiter.rs:99`, ungated |
| "No Redis-backed rate limiter exists anywhere in the codebase" (`ARCHITECTURE.md:390`) | **FALSE** | `crates/buzz-pubsub/src/rate_limiter.rs:24-121` |
| "`RateLimitConfig` defines 4 tiers" | **TRUE** | `crates/buzz-auth/src/rate_limit.rs:86-108` |
| "…but none are enforced" | **FALSE for 2 of 4 tiers** — human and agent-standard are enforced (`crates/buzz-relay/src/connection.rs:614`, `:634`, `:636`; `crates/buzz-relay/src/api/bridge.rs:29`); agent-elevated and agent-platform are genuinely unenforced | see the table above |
| "rate limiting is not currently enforced" (`ARCHITECTURE.md:390`) | **FALSE** | `crates/buzz-relay/src/connection.rs:498-500` and `crates/buzz-relay/src/api/bridge.rs:760`, `:955`, `:1386` |
| "`buzz-pubsub` … Does NOT implement the rate limiter" (`ARCHITECTURE.md:460`) | **FALSE** | `crates/buzz-pubsub/src/rate_limiter.rs:99` |

Residual truth worth preserving from the doc: the two agent tiers above
`agent-standard` are still design-only, per-IP connection limits are still
unenforced, and the algorithm is a fixed window with 2× boundary burst
(`crates/buzz-auth/src/rate_limit.rs:165-167`,
`crates/buzz-pubsub/src/rate_limiter.rs:7-8`,
`crates/buzz-relay/src/admission.rs:6-8`).

---

### Secondary verification results

| # | Claim | Verdict | Evidence |
|---|-------|---------|----------|
| a | "NIP-42 grants `Scope::all_known()` (all 14 scopes)" | **PARTIAL** — the function is right, the count is wrong. `verify_auth_event` sets `scopes: Scope::all_known()` (`crates/buzz-auth/src/lib.rs:138`), and `all_known()` returns **16** variants (`crates/buzz-auth/src/scope.rs:68-87`, pinned by `assert_eq!(all.len(), 16)` at `:237`). 14 is the cardinality of `all_non_admin()` (`crates/buzz-auth/src/scope.rs:94-111`, `assert_eq!(scopes.len(), 14)` at `:208`), which no auth path calls |
| b | "NIP-42 timestamp tolerance: ±60 seconds" | **TRUE** — `TIMESTAMP_TOLERANCE_SECS = 60` (`crates/buzz-auth/src/nip42.rs:35`), symmetric via `abs_diff` (`:80-83`); test rejects a 120s-old event (`:143-157`) |
| c | "NIP-98 auth: Schnorr-signed `kind:27235` events with URL + method tags" | **TRUE** — kind gate `Kind::HttpAuth` (`crates/buzz-auth/src/nip98.rs:66`), Schnorr via `buzz_core::verify_event` (`:74`), single-letter `u` tag (`:89-95`), `method` tag (`:104-108`). Optional `payload` body-hash tag adds a third check (`:117-127`) |
| d | "Dev-only key derivation: `SHA-256(\"buzz-test-key:{username}\")` gated behind `#[cfg(any(test, feature = \"dev\"))]`" | **TRUE**, and `buzz-relay` does **not** enable the feature in non-dev builds. Derivation at `crates/buzz-auth/src/lib.rs:162-166`, gate at `:159`. `buzz-relay`'s normal dep requests no features (`crates/buzz-relay/Cargo.toml:22`); `dev = ["buzz-auth/dev"]` is an opt-in feature (`:84`); the only `features = ["dev"]` request is in `[dev-dependencies]` (`:90`), which under `resolver = "2"` (`Cargo.toml:32`) does not affect a non-test build; `Dockerfile:67` passes no feature flags |
| e | "`buzz-auth` implements `ChannelAccessChecker` itself or only defines the trait" | **Defines the trait; the only impl in the crate is test-gated.** Trait at `crates/buzz-auth/src/access.rs:31-57`; sole implementor `MockAccessChecker` under `#[cfg(any(test, feature = "test-utils"))]` (`crates/buzz-auth/src/access.rs:135-151`). No implementor exists anywhere else in the repo — the trait doc's claim that `buzz-db` implements it (`crates/buzz-auth/src/access.rs:18-19`) is unsupported by code |


## Module: buzz-pubsub (`crates/buzz-pubsub`)
### Aspect: Security

The crate performs **no authorization**. It states this explicitly and consistently:
pub/sub topics are "a routing/performance boundary, not an authorization boundary"
(`topic.rs:3-6`), and the relay re-checks access before local fan-out. Assessment
below covers what the crate *does* guarantee, and where the guarantees stop.

---

### 1. Tenant isolation — mechanism and limits

| Control | Implementation | Strength |
|---|---|---|
| Community id sourced only from server-resolved context | `EventTopicKey::from_context` reads `ctx.community()` (`topic.rs:35-40`); every key builder takes `&TenantContext` (`presence.rs:19`, `cache_invalidation.rs:30`, `conn_control.rs:33`, `topic.rs:103`, `:108`) | Strong — no public API accepts a caller-supplied community id |
| Delivered community comes from the parsed channel name, not the payload | `subscriber.rs:159-163`; control paths `cache_invalidation.rs:144-148`, `conn_control.rs:135-139` | Strong — a spoofed community field inside an event body cannot redirect routing |
| Strict channel-name grammar | `parse_redis_channel` `topic.rs:53-99`; rejects wrong prefix, non-UUID community, unknown scope word, and any extra segment | Strong — 13 negative cases asserted (`topic.rs:179-195`), including a `presence:` key (`:187`) |
| Refcount keyed by `(community, topic)` not by channel id | `subscriber.rs:21`, `lib.rs:193`, `:216` | Strong — regression test proves releasing community A does not silence community B (`lib.rs:510-590`) |
| Replay seen-sets isolated per community | `nip98_replay.rs:81` | Strong — asserted `nip98_replay.rs:144-162` |

**Limit:** isolation is key-prefix naming inside one shared Redis instance. There is
no per-tenant Redis logical db, no ACL, no separate credentials. Two subscribers
deliberately consume **all** tenants' control traffic via wildcard patterns —
`buzz:*:cache-invalidate` (`cache_invalidation.rs:27`, subscribed `:139`) and
`buzz:*:conn-control` (`conn_control.rs:30`, subscribed `:130`). Any process able to
reach Redis can read every community's event stream and inject forged control
messages. The trust model is stated: "Redis only ever carries events between nodes
inside the relay trust domain" (`lib.rs:317-318`).

### 2. Control-message integrity

Neither control payload is authenticated, versioned, timestamped, or
origin-attributed (`CacheInvalidation` `cache_invalidation.rs:57-80`; `ConnControl`
`conn_control.rs:55-73`). Consequences for an attacker with Redis write access:

- Publishing `ConnControl::DisconnectPubkey` to `buzz:{community}:conn-control`
  disconnects arbitrary members with an attacker-chosen `reason` string, which is
  echoed to the victim in the closing `OK` frame (`conn_control.rs:62-65`, dispatch
  `buzz-relay/src/main.rs:913`) — a denial-of-service and a message-injection
  surface into a security-relevant UI string. `reason` has no length or content
  validation on either side.
- Publishing `ConnControl::DisconnectCommunity` disconnects every socket for a
  community (`conn_control.rs:57-58`, dispatch `buzz-relay/src/main.rs:908`).
- Publishing a forged Nostr event to a topic cannot forge *content* — the relay
  applies `filter_fanout_by_access` on the Redis path
  (`buzz-relay/src/handlers/event.rs:135`, `main.rs:696`) and the event carries a
  Nostr signature — but it can cause unsolicited delivery attempts.
- `CacheInvalidation` forgery is comparatively benign by design: the payload is a
  pure idempotent key drop and the next read re-fetches authoritative state from
  the DB (`cache_invalidation.rs:11-14`). Worst case is added DB load.

`pubkey` in both `CacheInvalidation::Membership` and `ConnControl::DisconnectPubkey`
is typed `Vec<u8>` while documented as "32 raw bytes" (`conn_control.rs:60`); no
length check exists on publish or receive, so a malformed length propagates to the
relay's matching logic.

Mitigating factor for `ConnControl`: the DB ban row is the durable backstop, so a
*dropped* message still denies the next auth attempt (`conn_control.rs:18-21`). That
protects availability of the control, not its integrity against injection.

### 3. Rate limiting (`rate_limiter.rs`)

| Property | Finding |
|---|---|
| Atomicity | `INCR` + conditional `EXPIRE` + `TTL` in a single Lua script (`rate_limiter.rs:24-31`), eliminating the crash window that could leave a TTL-less key (`:19-22`) |
| Fixed-window weakness | Self-documented: up to **2× burst** at window boundaries; "upgrade to sliding window or token bucket for strict limiting" (`rate_limiter.rs:9-10`). Echoed at `buzz-relay/src/admission.rs:4-7` |
| Broken-state repair | `ttl < 0` triggers a fresh `EXPIRE` plus a `warn` (`rate_limiter.rs:57-70`). Note the repair resets the window to full duration, so a key in that state effectively gets a fresh allowance |
| Allow boundary | Inclusive — `count <= limit` allows (`rate_limiter.rs:74-78`), so the Nth request within a limit of N succeeds |
| Failure mode | Redis error → `AuthError::Internal` (`rate_limiter.rs:44-46`, `:52`, `:66`) → relay maps to `AdmissionError::Unavailable` (`buzz-relay/src/admission.rs:29-33`) → request **denied**. Fail-closed, which is correct, but makes Redis a hard dependency for all authenticated reads and writes (`buzz-relay/src/connection.rs:612-621`) |
| Coverage gap | **Unauthenticated frames are never rate limited.** `enforce_ws_admission` returns `true` for any connection not in `AuthState::Authenticated` (`buzz-relay/src/connection.rs:602-609`), because the limiter is pubkey-keyed. Pre-AUTH traffic is unmetered by this control |
| Scope gap | Only `EVENT`, `REQ`, and `COUNT` are metered; every other frame type short-circuits (`buzz-relay/src/connection.rs:598-601`) |
| Dead control | `check_ip_connection` (`rate_limiter.rs:112-120`) — the one limiter that could cover pre-auth traffic — has **no production caller**. Repo-wide, `LimitType::IpConnections` and `ip_rate_limit_key` appear only in the `buzz-auth` definitions (`buzz-auth/src/rate_limit.rs:66`, `:76`, `:213`), this impl, and a `#[cfg(test)]` `StubLimiter` (`buzz-relay/src/admission.rs:85`) |
| Test coverage | **Zero tests in this file.** The Lua script, the `count <= limit` boundary, and the TTL-repair branch are unverified anywhere; `buzz-relay`'s tests stub the limiter out entirely (`admission.rs:65-90`) |

### 4. NIP-98 replay protection (`nip98_replay.rs`)

| Property | Finding |
|---|---|
| Atomicity | `SET key 1 NX EX ttl` — single round-trip set-if-absent (`nip98_replay.rs:68-78`); freshness proven by Redis returning `OK` only on first claim (`:20-23`) |
| Replay outcome | `nil` → `Ok(false)` so the caller rejects (`:87`); tested `:127-142` |
| TTL clamping | `ttl_secs.clamp(DEFAULT_REPLAY_TTL_SECS, MAX_REPLAY_TTL_SECS)` (`:47`). Both directions matter and both are tested: sub-floor is lifted (`:164-177`), and `u64::MAX` is pushed down so Redis cannot reject the `EX` arg and turn every request into an error (`:179-200`, rationale `:184-190`) |
| Unexpected reply | Logged at `error` and returned as `Err`, never treated as a claim (`:88-98`) — closes the "any non-nil means success" foot-gun |
| Failure mode | `Err` on pool or command failure with explicit "caller MUST fail closed" contract, restated in the log payloads (`:53-57`, `:74-79`) |
| Residual | Correctness depends on the caller honouring fail-closed; the crate cannot enforce it. Redis eviction under `maxmemory` pressure would silently shorten the seen-set window — no `noeviction` requirement is asserted or documented |

### 5. Secret and PII handling in logs

| Site | Content | Assessment |
|---|---|---|
| `rate_limiter.rs:58` | `warn!(key = %key, ...)` — the key embeds the full `pubkey_hex` | Pubkeys are public identifiers, not secrets; still a per-user identifier in logs |
| `nip98_replay.rs:53-56`, `:74-78`, `:91-96` | `scope` (contains community id) and Redis error strings | Redis errors can include connection strings; the crate does not scrub them before logging |
| `subscriber.rs:143`, `:154` | Channel name and deserialization error | Channel name contains community + channel UUIDs |
| Event payloads | Never logged | Good — no `warn!("{payload}")` anywhere; failures log only the error (`subscriber.rs:154`) |
| `redis_url` | Stored in `PubSubManager` (`lib.rs:105`) and passed to each reconnect | **Never logged.** No `Debug` derive on `PubSubManager` (`lib.rs:100`), so a credential-bearing URL cannot leak through a struct dump. `PubSubConfig` *does* derive `Debug` (`lib.rs:72`) and holds `redis_url`, so debug-printing a config would expose credentials — no such site exists today |

### 6. Denial-of-service considerations

| Vector | Status |
|---|---|
| Broadcast lag | `broadcast::channel(4096)` (`lib.rs:126-128`); a slow WS receiver gets `RecvError::Lagged` mapped to `PubSubError::BroadcastLagged` (`error.rs:22`, `:33-34`) rather than blocking producers. Capacity is hardcoded, not configurable |
| Unbounded topic growth | `desired_topics` (`subscriber.rs:21`) grows with distinct subscribed topics; entries are removed at refcount zero (`lib.rs:227`), so growth is bounded by concurrent live subscriptions. No explicit cap on topics per connection is enforced here |
| Subscription churn | Debounced unsubscribe (default 500 ms, `lib.rs:82`) with refcount re-check (`subscriber.rs:123-130`) damps thrash |
| Per-release task spawn | `release_topic` spawns a fresh `tokio::spawn` for every last-release (`lib.rs:236-243`); a client rapidly toggling subscriptions creates one short-lived task per toggle |
| Reconnect hot loop | A Redis that accepts connections then immediately closes the stream cleanly resets backoff to 1 s each cycle (`subscriber.rs:57-63`, `cache_invalidation.rs:105-111`, `conn_control.rs:95-101`), producing a sustained 1 s reconnect loop with no escalation |
| Event size | Whole Nostr events are relayed as JSON (`publisher.rs:31`); the crate imposes no size cap of its own |

### 7. Positive findings

- Zero `unsafe` (`#![deny(unsafe_code)]` `lib.rs:1`).
- Zero `unwrap`/`expect` outside `#[cfg(test)]`.
- No SQL, no shell, no filesystem, no HTTP — the attack surface is Redis-only.
- No path where a caller can supply a community id directly.
- Strict deny-by-default channel-name parsing with explicit negative tests.
- Both `buzz-auth` seams are fail-closed by contract, and the replay guard restates
  that obligation in its telemetry.
- Author-private reminder routing is explicitly reasoned about rather than assumed
  safe, with the real enforcement point named (`lib.rs:305-320`).


## Module: buzz-search (`crates/buzz-search`)

### Security

#### SQL injection surface

| Question | Finding | Evidence |
|---|---|---|
| Is the FTS term parameterized? | Yes, in both modes. | `FullText`: `qb.push("websearch_to_tsquery('simple', "); qb.push_bind(search_text); qb.push(")")` (`query.rs:143-145`). `Prefix`: the raw text is a single `push_bind` (`query.rs:168`) consumed by in-SQL `regexp_split_to_table(...)`. |
| Is any user input interpolated into SQL text? | No. Every dynamic value is `push_bind`; `push` receives only `&'static`-style literal fragments. | binds at `query.rs:144`, `168`, `241`, `257`, `262`, `270`, `278`, `285`, `291`, `296`, `298`; no `format!`/`write!`/`+`-built SQL exists in `src/` |
| Does the shape of the statement depend on user data? | Predicate *presence* depends on `Option`/enum discriminants (`query.rs:248-293`), never on user-supplied strings. Each branch pushes a fixed fragment plus a bind. | `query.rs:248-293` |
| Test-only exception | `tests/fts_integration.rs` interpolates a locally generated schema name into DDL, explicitly opting in via `sqlx::AssertSqlSafe`. Name is `fts_test_<Uuid::new_v4().simple()>` — not caller-controlled. | `tests/fts_integration.rs:37`, `43-46`, `99-102` |

Conclusion: no first-order SQL injection surface in this crate.

#### tsquery injection via search operators

| Mode | Exposure | Mechanism |
|---|---|---|
| `FullText` | The term is interpreted by Postgres' `websearch_to_tsquery`, which is the sanitizing parser: it never raises syntax errors and treats stray operators as text. User operators can influence *matching* (quoted phrases, `or`, leading `-`) but cannot construct arbitrary tsquery structure outside websearch grammar, and cannot alter SQL. | `query.rs:143-145` |
| `Prefix` | Lexemes are produced by `to_tsvector('simple', token)` and each is wrapped in `quote_literal(...)` before concatenation into a tsquery, then cast `::tsquery`. The stated intent is exactly this: "`quote_literal` prevents tsquery syntax injection from punctuation/operators in the raw topbar input". Only the trailing token receives `:*`; the `&` conjunction is fixed by the code, so a user cannot inject `|`, `!`, `<->`, or parentheses into the query tree. | `query.rs:150-153`, `155-176` |
| Regression coverage | A test feeds `"operators ' : & | ( ) ! alpha be"` through `Prefix` and asserts one clean hit rather than an error or a widened match. | `tests/fts_integration.rs:441-481` |

Residual DoS-ish considerations (not injection): `Prefix` builds a tsquery whose
term count grows with the number of tokens in the input; the only bound is the
4096-char cap (`query.rs:134`, `185`), so a 4096-char whitespace-dense input can
produce a many-term conjunction. `FullText` inherits Postgres' own websearch
parser limits. `tsquery` conjunctions are AND-only in `Prefix`, so the result set
narrows rather than widens as terms grow.

#### Tenant isolation

| Question | Finding | Evidence |
|---|---|---|
| Does every query filter `community_id`? | Yes. The crate contains exactly one executed statement, and `WHERE community_id = <bind>` is written into the builder before any conditional branch. | `query.rs:240-241`, `300` |
| Can a caller omit it? | No. `SearchQuery.community: CommunityId` is not `Option`, has no default, and `CommunityId` cannot be parsed from client input (`crates/buzz-core/src/tenant.rs:37-45`). | `query.rs:76` |
| Exceptions | None — there is no second query method, no count method, no admin/bypass path. | whole `src/` |
| Regression coverage | Event inserted under community A, queried as B → asserted zero hits. | `tests/fts_integration.rs:201-259` |

Index-level note (storage, not this crate): tenant filtering relies on
community-leading btree indexes BitmapAnd-ed with the single-column GIN
(`migrations/0001_initial_schema.sql:270-278`). Correctness comes from the
predicate, not the index.

#### Authorization boundary (caller responsibility)

The crate performs **no** authorization. The visibility-affecting predicates it
applies are only: `community_id` (`query.rs:241`), `deleted_at IS NULL`
(`query.rs:242`), the caller-supplied `ChannelScope` (`query.rs:248-264`), and the
caller's `kinds`/`authors`/time filters (`query.rs:267-293`). Documented explicitly
at `lib.rs:15-18` and `query.rs:3-9` ("Search is never the access boundary — it
cannot widen visibility").

Where the boundary actually is:

| Step | Site |
|---|---|
| Caller maps its accessible-channel set into `ChannelScope`, short-circuiting when the reader has nothing accessible | `crates/buzz-relay/src/handlers/req.rs:484-501`, `517-524`; bridge equivalent `crates/buzz-relay/src/api/bridge.rs:1642-1651` |
| Hits are re-fetched by `(community, ids)` through buzz-db | `crates/buzz-relay/src/handlers/req.rs:601-621`, `crates/buzz-relay/src/api/bridge.rs:1727-1731` |
| Per-hit re-authorization: full NIP-01 filter re-match, channel-accessibility, reader authorization | `search_hit_accepted` at `crates/buzz-relay/src/api/bridge.rs:1594-1625`, called at `bridge.rs:1743`; WS equivalent `crates/buzz-relay/src/handlers/req.rs:685-701` |
| Author-only kinds dropped (now inside `event_visible_to_reader`) | `crates/buzz-relay/src/api/bridge.rs:1753`; helper `handlers/req.rs:1222-1234` |

If a caller forgets: with `ChannelScope::Any`, the crate returns hits from any
channel in the community regardless of membership; with any scope, constraints the
crate cannot push down (`#p`, `#e`, `#d`, `ids`) are simply unevaluated. The relay
documents this exact leak scenario for `/query` — an authorized-looking
`{"kinds":[30174],"#p":[self]}` search would otherwise return text-matching
envelopes belonging to another owner (`crates/buzz-relay/src/api/bridge.rs:1582-1592`).

Backstops that survive a caller mistake (defense in depth, both outside this crate):

| Backstop | Effect | Evidence |
|---|---|---|
| Storage-level NULL `search_tsv` for privacy-sensitive kinds | `@@` can never match NULL, so those rows are not candidates at all | `migrations/0001_initial_schema.sql:222-226`, `migrations/0005_agent_turn_metric_fts.sql:26-31`, `migrations/0014_push_lease_fts.sql:28-32`, `schema/schema.sql:211-215` |
| Fresh-install positive allowlist (empty DBs only) | only kinds `0, 9, 40002, 45001, 45003` are indexed | `migrations/0008_fresh_install_search_allowlist.sql:13-23` |
| Tripwire tests over `AUTHOR_ONLY_KINDS` and persistent `P_GATED_KINDS` | fail if a Rust privacy constant gains a kind the schema does not exclude | `tests/fts_integration.rs:1256-1361`, `1338-1447` |

#### Information leakage

| Vector | Assessment | Evidence |
|---|---|---|
| Result counts | No total/`found` is returned — only the page's hits. A caller can still infer existence by page length, but only within its own community and channel scope. | `query.rs:122-127` |
| Ranking | `SearchHit.rank` (`ts_rank_cd`) is returned for every hit and is derived from the matched row's tsvector. If a caller forwarded a hit's rank without re-authorizing, that leaks a relevance signal about content the reader may not read. In-tree callers discard everything but `event_id` (`crates/buzz-relay/src/handlers/req.rs:625-626`, `crates/buzz-relay/src/api/bridge.rs:1725`). | `query.rs:117`, `236` |
| Metadata in hits | `kind`, `pubkey`, `channel_id`, `created_at` are returned pre-authorization — same caller-discipline caveat as `rank`. | `query.rs:104-118` |
| Error messages | `SearchError::Db` forwards the underlying sqlx message; decode errors embed only a byte length, no row data (`query.rs:307`, `310`). Whether the relay surfaces it is the caller's choice (bridge wraps it into an internal error string, `crates/buzz-relay/src/api/bridge.rs:1719-1722`). | `error.rs:7` |
| Timing | No constant-time requirements apply; search timing is inherently data-dependent. | — |

#### Denial-of-service controls present

| Control | Value | Site |
|---|---|---|
| Search text length cap | 4096 chars | `query.rs:134`, `185` |
| Empty query rejected before any DB roundtrip | — | `query.rs:217-222` |
| `per_page` cap | 500 (`0 → 100`) | `query.rs:129-130`, `224-229` |
| `page` cap (bounds OFFSET) | 1000 | `query.rs:138`, `230` |
| NUL scrub (prevents Postgres error-path churn from adversarial text) | — | `query.rs:186`, test `tests/fts_integration.rs:972-1012` |

Not present: per-caller rate limiting, statement timeout, query cancellation, or
concurrency limit — all left to the pool/caller.

#### `unsafe` code

`#![deny(unsafe_code)]` at `crates/buzz-search/src/lib.rs:1`; no `unsafe` block
appears in any of the four `.rs` files. Verified by reading all of them.

#### Input-validation gaps

| Gap | Consequence | Site |
|---|---|---|
| `authors` entries are `Vec<u8>` of arbitrary length — no 32-byte check | A malformed pubkey simply matches nothing (`pubkey = ANY`), so it is a correctness/telemetry gap rather than a security hole | `query.rs:88`, `275-281` |
| `kinds` values are unbounded `i32`; negative or out-of-range kinds are accepted | matches nothing | `query.rs:86`, `267-273` |
| No `since <= until` validation | inverted window yields zero rows silently | `query.rs:283-293` |
| No cap on the *number* of channel ids / kinds / authors bound | a caller could bind a very large array; only caller-side discipline bounds it | `query.rs:255-281` |
| Char cap counts `chars`, not tokens or bytes | a 4096-char multi-token input in `Prefix` mode still builds a large conjunction (see tsquery section) | `query.rs:185` |
| Crate cannot verify the caller re-authorizes | the entire authorization model is external; nothing in-crate can detect a forgetful caller | `lib.rs:15-18` |


## Module: buzz-audit (`crates/buzz-audit`)

### Aspect: Security

### 1. Tamper-evidence: what the chain actually proves

| Property | Mechanism (file:line) | Holds? |
|---|---|---|
| Mutating any hashed field of a stored row is detectable | `verify_chain` recomputes `compute_hash` per row and compares to stored `hash` (`service.rs:197-200`) | Yes — test `verify_detects_tampering_within_a_community` (`service.rs:437-473`) |
| Deleting an interior row is detectable | the next row's `prev_hash` no longer matches its predecessor's `hash` → `ChainViolation` (`service.rs:190-195`) | Yes, **provided** the verified range spans the deletion |
| Deleting/truncating the chain **tail** is detectable | nothing in the crate persists an external head pointer; `log_inner` derives the head from `MAX(seq)` in the table itself (`service.rs:94-101`) | **No** — see §4 |
| Re-ordering rows | `seq` is inside the pre-image (`hash.rs:46`) and rows are read `ORDER BY seq ASC` (`service.rs:172`) | Yes for content, but `seq` gaps are not asserted (§4) |
| Moving a row into another community's chain | `community_id` leads the pre-image (`hash.rs:44-45`) | Yes — test `cross_community_row_does_not_verify` (`service.rs:475-510`) |

### 2. Can an attacker with DB write access rewrite the whole chain?

**Yes.** The chain is keyed only by SHA-256 over data the attacker can also see and
write; there is no secret key, no signature, and no external anchor anywhere in
`crates/buzz-audit/src` (the crate's only crypto import is `sha2` at `hash.rs:2`; no
`hmac`, no signing crate is declared in `crates/buzz-audit/Cargo.toml:11-22`).

Concretely, an attacker who can `UPDATE`/`INSERT` on `audit_log` can:
1. modify or drop entries,
2. recompute each subsequent `hash` with the same public algorithm (`hash.rs:42-73`) —
   all inputs are stored in the row itself,
3. rewrite the chain so `verify_chain` returns `Ok(true)`.

Nothing detects this: there is no signed head, no periodic checkpoint export, no
transparency-log submission, no WORM storage, and no DB-level protection —
`migrations/0001_initial_schema.sql:606-619` creates a plain table with a PK and a
unique index, no triggers, no revoked `DELETE`/`UPDATE` grants, no row-level security.
The crate itself never issues `UPDATE`/`DELETE` (only the `INSERT` at
`service.rs:130-147`), so append-only is a property of *this code path*, not of the
datastore.

The relay uses the **same** database credentials for the audit pool as for everything
else (`crates/buzz-relay/src/main.rs:322-328` connects with `config.database_url`), so
a compromised relay process holds write access to the audit table.

Accurate statement of the guarantee: the chain gives **tamper evidence against actors
who can modify rows but cannot recompute and rewrite the suffix** (e.g. partial
corruption, a single UPDATE, restore of a stale row, accidental edit) and against
**cross-tenant row replay**. It does not give tamper-*proofing* against a full-DB
adversary.

### 3. Hash construction weaknesses

- **Unframed concatenation.** Fields are appended with no length prefixes and no
  domain separator (`hash.rs:45-71`). Only the two optional fields get a 1-byte
  presence tag (`hash.rs:55`, `:58`, `:62`, `:65`). Because `action` (field 4) is a
  variable-length string immediately followed by the `actor_pubkey` presence tag, and
  `object_id` is variable-length UTF-8 followed by the canonical-JSON `detail` string,
  the boundaries between adjacent variable-length fields are not cryptographically
  marked. Practical exploitability is limited by (a) `action` being drawn from a fixed
  11-element set (`action.rs:37-47`) — an attacker cannot choose it freely, and
  (b) `detail` always starting with a JSON-legal character. Still, "distinct field
  tuples ⇒ distinct pre-images" is not guaranteed by construction, only by the
  restricted value domains. A length-prefixed or tagged encoding would close it.
- **Length-extension.** SHA-256 is Merkle–Damgård and therefore length-extendable, but
  the digest is not used as a MAC and no secret is prefixed, so length extension buys
  an attacker nothing here: any forged successor must still be a row whose *stored*
  fields recompute to the digest (`service.rs:197-200`). No `hash.rs` code accepts an
  externally supplied digest as a pre-image continuation.
- **`prev_hash` length is unchecked.** `compute_hash` feeds whatever bytes are in the
  `Option<Vec<u8>>` (`hash.rs:69`); the column is untyped `BYTEA`
  (`migrations/0001_initial_schema.sql:610`). A row with a 5-byte `prev_hash` hashes
  fine; only the link check would catch it, and only if the predecessor is in range
  (`service.rs:192`).
- **No depth limit on `canonical_json` recursion** (`hash.rs:80-116`). `detail` is
  caller-supplied JSON from the relay (`crates/buzz-relay/src/handlers/event.rs:566-569`,
  `crates/buzz-relay/src/api/media.rs:431-435`); a deeply nested value would recurse
  proportionally to nesting depth. Both current producers build shallow literals, so
  this is a latent rather than reachable issue.

### 4. Deletion / truncation detectability

| Scenario | Detected? | Why (file:line) |
|---|---|---|
| Interior row deleted, range covers the hole | Yes — `ChainViolation` | `service.rs:190-195` |
| Interior row deleted, verification range starts *after* the hole | No | the first row in a range is compared against nothing; `expected_prev` starts as `None` (`service.rs:185-190`) |
| Tail rows deleted (chain truncated) | **No** | `verify_chain(from, to)` returns `Ok(true)` for whatever exists; an empty range returns `Ok(false)` not an error (`service.rs:181-183`); there is no stored head/length to compare against |
| Whole community's chain deleted | **No** | subsequent `log` sees no head and starts again at `seq = 1` with `prev_hash = NULL` (`service.rs:103-110`) — indistinguishable from a fresh community |
| Genesis integrity (first row must be `seq = 1` with `prev_hash = NULL`) | Not checked | no such assertion exists in `verify_chain` (`service.rs:160-206`) |
| `seq` contiguity within a range | Not checked | loop iterates rows as returned, never compares `entry.seq` to `previous.seq + 1` (`service.rs:187-203`) |

The `UNIQUE (community_id, hash)` index (`migrations/0001_initial_schema.sql:619`)
prevents storing two identical digests in one community, which blocks a naive
duplicate-row replay.

### 5. Canonical-serialization determinism

`canonical_json` sorts object keys via `BTreeMap<&str, &Value>` (`hash.rs:86`),
preserves array order (`hash.rs:101-113`), and delegates scalars to `serde_json`
(`hash.rs:114`). Determinism therefore depends on `serde_json`'s scalar rendering
(notably float formatting) being stable across versions — the doc claims stability
"across machines and Rust versions" (`hash.rs:39-41`) but nothing in the crate pins or
tests float formatting. Sorting is by `&str` byte order, which is well-defined.
Timestamp determinism is handled separately by `to_storage_precision`
(`hash.rs:22-24`), and the RFC3339 variable-width trap is documented and tested
(`hash.rs:167-183`).

Serialization failure is a hard error rather than a silent empty value
(`hash.rs:67`, `error.rs:39-40`) — important because an "empty on failure" fallback
would let two different `detail` payloads share a digest.

### 6. Lock-release safety on panic

`log` wraps the append in `AssertUnwindSafe(...).catch_unwind()` (`service.rs:67-69`),
runs the unlock unconditionally (`service.rs:71-74`), then re-raises any captured panic
with `resume_unwind` (`service.rs:76-79`). Residual risks visible in code:

- The unlock's own error is discarded (`let _ =`, `service.rs:71`). If the unlock fails
  (connection already broken), the connection returns to the pool still holding the
  advisory lock for that community, and subsequent `log` calls for that community would
  block until the session ends.
- **Future cancellation** (the `log` future dropped mid-await, e.g. by a `select!` or
  task abort) bypasses the unlock entirely — `catch_unwind` covers unwinding, not
  cancellation. In the relay this path is single-task and not raced
  (`crates/buzz-relay/src/state.rs:657-672`), so it is latent.
- `AssertUnwindSafe` is an assertion, not a proof: state mutated before a panic
  (`audit_entry` is local; the transaction is dropped and rolls back) is confined to the
  call, so the assertion looks sound for the current body (`service.rs:82-152`).
- No test induces a panic inside `log_inner`, so the branch is untested.

### 7. Tenant isolation of chains

- `community_id` is the first hashed field (`hash.rs:44-45`) and leads the PK
  (`migrations/0001_initial_schema.sql:616`).
- Every statement filters on `community_id`: head read (`service.rs:96`), verify
  (`service.rs:171`), list (`service.rs:223`). No query in the crate reads across
  communities and there is no "all communities" API.
- Input type is `CommunityId` (`entry.rs:57`), a `buzz-core` newtype with no
  `Deserialize` (`crates/buzz-core/src/tenant.rs:37`), and `NewAuditEntry` is
  intentionally non-deserializable (`entry.rs:46-51`). `buzz-core` states plainly that
  this is a lint-and-review fence, not a compiler fence, because
  `CommunityId::from_uuid` is public (`crates/buzz-core/src/tenant.rs:19-25`).
- Error text is sanitized so a failure while verifying one tenant cannot name another:
  variants hold only `seq` (`error.rs:22-32`), asserted by
  `audit_error_text_carries_no_community_id_or_constraint` (`error.rs:58-107`). Note the
  `Database(sqlx::Error)` variant is *not* covered by that test (it iterates only the
  three domain variants, `error.rs:79-83`) — a Postgres error string can carry the
  constraint name `audit_log_pkey` and would render through `database error: {0}`
  (`error.rs:14-15`). Mitigated by the fact that audit errors are logged
  operator-side, never returned to clients (`error.rs:5-7`;
  `crates/buzz-relay/src/state.rs:1201-1203`).
- The advisory lock is per-community precisely to avoid a cross-tenant timing oracle
  (`service.rs:25-28`). Residual: `hashtextextended` can collide two community keys onto
  one lock key; the code has no collision handling, so colliding tenants would serialize
  (a timing side channel, not a data leak).

### 8. `unsafe` code

None. `#![deny(unsafe_code)]` at `crates/buzz-audit/src/lib.rs:1`, and a grep for
`unsafe` across `crates/buzz-audit/src` matches only that lint line.

### 9. Input validation gaps

| Input | Validation present | Gap |
|---|---|---|
| `actor_pubkey` | none | any length accepted (`entry.rs:61`, bound at `service.rs:142`); a 32-byte pubkey is a caller convention only. Relay passes `hex::decode(...).ok()` so a malformed hex actor silently becomes `None` (`crates/buzz-relay/src/handlers/event.rs:589`) |
| `object_id` | none | unbounded `TEXT` (`entry.rs:63`, `migrations/0001_initial_schema.sql:613`) |
| `detail` | none | unbounded JSONB, arbitrary nesting; documented-but-unenforced "no secrets" rule (`entry.rs:64-71`) |
| `action` on read | `FromStr` over the 11 known strings, else `UnknownAction` (`service.rs:240-243`) | one bad row fails an entire read/verify call (`service.rs:234`) |
| `from_seq`/`to_seq`/`limit` | none | negative or inverted ranges are passed straight to SQL (`service.rs:176-177`, `:229-230`); an inverted range simply yields no rows → `Ok(false)`, and an unbounded `limit` is the caller's responsibility |
| `community_id` | type-level only | see the fence caveat in §7 |

Positive: all values are bound as parameters, so no SQL injection surface — the only
string interpolation is the advisory-lock key, itself passed as `$1`
(`service.rs:58-60`).

### 10. Secrets handling

The crate reads no credentials and logs no payloads: `log`'s span skips the entry
(`service.rs:52`), the `debug!` records only `seq` (`service.rs:128`), and the
unknown-action `warn!` omits the offending string (`service.rs:241`). The only
environment read is `DATABASE_URL` inside a test helper (`service.rs:275-279`).


## Module: buzz-media (`crates/buzz-media`)

### Aspect: Security

### 0. Summary table

| # | Concern | Status in this crate | Key file:line |
|---|---|---|---|
| S-1 | Content-type spoofing | Mitigated — MIME always sniffed, header never read | `crates/buzz-media/src/validation.rs:239-247` |
| S-2 | Pixel bomb / decompression bomb | Mitigated — 25 MP pre-decode cap, fail-closed | `crates/buzz-media/src/validation.rs:264-273` |
| S-3 | Memory exhaustion (images/files) | Partially mitigated — caps bound RAM but the whole body is buffered | `crates/buzz-media/src/validation.rs:167-172`, `crates/buzz-relay/src/api/media.rs:359-372` |
| S-4 | Memory exhaustion (video) | Mitigated — streamed to disk, 64 KiB chunks | `crates/buzz-media/src/upload.rs:325-395` |
| S-5 | Disk exhaustion (video temp files) | Partially mitigated — per-request bound only | `crates/buzz-media/src/upload.rs:307-320` |
| S-6 | Path traversal in object keys | Mitigated in this crate by construction | `crates/buzz-media/src/upload.rs:94`, `crates/buzz-media/src/storage.rs:183-185` |
| S-7 | SSRF | Not applicable — no remote fetch exists | whole crate |
| S-8 | Hash verification of uploaded bytes | Mitigated — server computes the hash, client hash must match | `crates/buzz-media/src/upload.rs:84-86`, `crates/buzz-media/src/auth.rs:190-196` |
| S-9 | Integrity on read | **Gap** — bytes are never re-verified on read | `crates/buzz-media/src/storage.rs:105-146` |
| S-10 | EXIF/metadata privacy | Mitigated by rejection, not stripping | `crates/buzz-media/src/validation.rs:487-928` |
| S-11 | Read authentication | **Not in this crate** — verifier provided, gate lives in relay | `crates/buzz-media/src/auth.rs:207-236`, `crates/buzz-relay/src/api/media.rs:489-515` |
| S-12 | Signed URLs | Not implemented anywhere in the crate | `crates/buzz-media/src/storage.rs:73-265` |
| S-13 | Tenant isolation of blobs | Sidecar-scoped; raw CAS bytes are shared | `crates/buzz-media/src/storage.rs:177-185` |
| S-14 | `unsafe` | **None** — zero occurrences | all 12 files |
| S-15 | Panic surface in parsers | Bounded; 9 infallible `try_into().unwrap()` + unchecked `offset += atom_size` | `crates/buzz-media/src/validation.rs:481`, `:604-886` |
| S-16 | Durable-bytes quota | **Gap** — no per-pubkey/per-community storage quota | `crates/buzz-relay/src/api/media.rs:302-304` (TODO) |

---

### 1. Content-type spoofing

The declared HTTP `Content-Type` is never consulted in this crate. Both validators start from `infer::get` on the actual bytes, with the comment "Magic bytes — never trust Content-Type header" (`crates/buzz-media/src/validation.rs:239-242`).

Cross-path spoofing is closed in both directions:
- MP4 submitted to the image path → `infer` reports `video/mp4`, which is absent from `ALLOWED_MIME_TYPES` → `DisallowedContentType` (`crates/buzz-media/src/validation.rs:9-13`, `:245-247`; test `crates/buzz-media/src/validation.rs:1626-1648`).
- Media submitted to the generic-file path → sniffed `image/*`, `video/*`, `audio/*` all rejected (`crates/buzz-media/src/validation.rs:186-196`; tests `crates/buzz-media/src/validation.rs:1507-1555`).
- MP4 with a proprietary major brand that `infer` cannot classify → still caught by the structural `ftyp` check, so it cannot fall through the generic path as an opaque attachment (`crates/buzz-media/src/validation.rs:174-181`; test asserts `infer::get(...)` is `None` yet the file is rejected, `crates/buzz-media/src/validation.rs:1518-1525`).

Stored-XSS defence for generic files is layered: a deny-list here (`text/html`, `application/xhtml+xml`, `image/svg+xml`, `application/javascript`, `text/javascript`, plus 9 executable/installer types — `crates/buzz-media/src/validation.rs:75-95`) and `serve_inline` returning false for everything except `image/*`/`video/*` (`crates/buzz-media/src/validation.rs:216-218`). The comment records that the response headers (`Content-Disposition: attachment`, `nosniff`, `CSP: default-src 'none'`) already neutralise these and the list is defence in depth against a future header regression (`crates/buzz-media/src/validation.rs:66-74`). Those headers are set in the relay (`crates/buzz-relay/src/api/media.rs:660-700`), not here.

Residual: unsniffable bytes on the generic path are accepted as `application/octet-stream` (`crates/buzz-media/src/validation.rs:207`). An HTML/JS payload with no magic signature is therefore stored — safe only because of the attachment/nosniff headers plus the `serve_inline` policy, i.e. this is a defence-in-depth dependency on the relay's header behaviour.

---

### 2. Decompression / pixel bombs and malformed media

| Guard | Value | file:line |
|---|---|---|
| Byte cap before any parse (image) | `max_image_bytes` / `max_gif_bytes` | `crates/buzz-media/src/validation.rs:259-268` |
| Pixel cap before full decode | `25_000_000` px; comment "25 megapixels — 100MB max RGBA decode" | `crates/buzz-media/src/validation.rs:269-273` |
| Fail-closed on unparseable geometry | `imagesize::blob_size(...).map_err(\|_\| InvalidImage)?` — unknown-geometry images never reach the decoder | `crates/buzz-media/src/validation.rs:264-270` |
| Structural pre-checks before decode | full container walk (`validate_image_metadata_free`) runs before the pixel check and before decode | `crates/buzz-media/src/validation.rs:263`, `:492-500` |
| MP4 top-level atom scan cap | `MAX_ATOMS = 1024`, header-only reads, fail-closed on exceeding | `crates/buzz-media/src/validation.rs:411-433` |
| MP4 box-walk caps | `MAX_BOXES = 100_000`, `MAX_BOX_DEPTH = 32` | `crates/buzz-media/src/validation.rs:832-833`, `:871-878` |
| MP4 division-by-zero guard | `timescale() == 0` rejected before `duration()` ("Fail fast before it panics") | `crates/buzz-media/src/validation.rs:346-351` |
| GIF/PNG/WebP/JPEG walks | all use `checked_add` + explicit `<= len` filters, no unchecked indexing into attacker-controlled offsets | `crates/buzz-media/src/validation.rs:533-538`, `:596-600`, `:672-682`, `:739-748` |

Order matters and is correct: the pixel guard runs **before** `image::load_from_memory`, and the decode happens in a separate function called from `spawn_blocking` (`crates/buzz-media/src/validation.rs:264-273` → `crates/buzz-media/src/thumbnail.rs:26` via `crates/buzz-media/src/upload.rs:518-524`).

Residuals:
1. The 25 MP cap bounds a single RGBA decode to ~100 MB, but the *concurrent* count is not bounded here — concurrency limits live in the relay (`crates/buzz-relay/src/config.rs:659-676`).
2. Animated GIF/WebP are decoded via `image::load_from_memory`, which yields a single frame; the only bound on animation complexity is `max_gif_bytes` (`crates/buzz-media/src/validation.rs:259-262`).
3. `check_moov_before_mdat` advances with `offset += atom_size` where `atom_size` can be an attacker-supplied 64-bit value (`crates/buzz-media/src/validation.rs:463-481`). No `checked_add` — this is an overflow-wrap in release builds (workspace release profile has `panic = "abort"`, `Cargo.toml:159-162`) and a debug-build panic. Impact is bounded by `MAX_ATOMS = 1024` iterations and by the fact that a wrapped offset yields a failed `read_exact` or a `MoovNotAtFront`; the sibling walker in `validate_mp4_metadata_free` does use `checked_add` (`crates/buzz-media/src/validation.rs:891-893`), so this is an internal inconsistency worth closing.

---

### 3. Memory and disk exhaustion

| Path | Peak RAM | file:line |
|---|---|---|
| Image | full body (≤ `max_image_bytes`) + decoded RGBA (≤ ~100 MB) + thumbnail | `crates/buzz-media/src/upload.rs:207-236`, `crates/buzz-media/src/thumbnail.rs:26-32` |
| Generic file | full body (≤ `max_file_bytes`); the body is cloned into the blocking closure but `Bytes` clones are refcount bumps, documented | `crates/buzz-media/src/upload.rs:76-78`, `:193-201` |
| Video | 64 KiB read buffer + 4 KiB sniff buffer; body never fully in RAM | `crates/buzz-media/src/upload.rs:349-353` |
| Video S3 upload | 8 MiB `BufReader`, streamed from disk | `crates/buzz-media/src/storage.rs:91-101` |
| Blob read | `get`/`get_range` materialise into `Vec<u8>`; `get_stream` does not | `crates/buzz-media/src/storage.rs:105-146` |
| Sweep | only per-sha/per-binding aggregates, never the full listing (documented) | `crates/buzz-media/src/bucket_index.rs:230-246` |

Video disk usage: a `NamedTempFile` is created per request and dropped immediately after the S3 upload (`crates/buzz-media/src/upload.rs:307-309`, `:435`). Per-request size is capped three times (Content-Length pre-check, running total, on-disk size — `crates/buzz-media/src/upload.rs:311-320`, `:372-378`, `crates/buzz-media/src/validation.rs:298-304`), but **total concurrent temp-file bytes are not bounded in this crate**; that depends on the relay's upload concurrency semaphore (`crates/buzz-relay/src/config.rs:659-668`). With the defaults (8 concurrent × 500 MB) the worst case is ~4 GB of temp space.

A body-limit error raised by axum's layer mid-stream is translated to `FileTooLarge`/413 rather than a 500, via substring matching on three known Display forms with a test that guards against all three breaking at once (`crates/buzz-media/src/upload.rs:328-341`, `:358-366`, test `crates/buzz-media/src/upload.rs:685-712`). This is fragile by construction (acknowledged in the comment) but fails toward a correct-status rejection, not toward acceptance.

---

### 4. Path traversal in object keys

Every key written by this crate is built from server-controlled components:

| Key | Components | file:line |
|---|---|---|
| blob | server-computed sha256 hex + ext from a fixed MIME→ext map | `crates/buzz-media/src/upload.rs:94`, `crates/buzz-media/src/validation.rs:930-939` |
| thumb | server sha256 + literal `.thumb.jpg` | `crates/buzz-media/src/upload.rs:531` |
| sidecar | `_meta/` + `TenantContext::community()` UUID + server sha256 | `crates/buzz-media/src/storage.rs:183-185` |
| record | `_uploads/` + community UUID + server sha256 + generated ULID | `crates/buzz-media/src/upload_record.rs:181-183` |

No client string ever reaches a key. The one path with a partly-external component is the generic-file extension: when `file_mime_to_ext` has no mapping it falls back to `infer`'s `kind.extension()` (`crates/buzz-media/src/validation.rs:203-206`) — a static string from the `infer` crate's own table, not user input, though it is not re-validated against a charset here. The sweep's classifier independently enforces a 1–8 alphanumeric ext when reading keys back (`crates/buzz-media/src/bucket_index.rs:84-87`), and the relay's `validate_media_path` constrains the read path (`crates/buzz-relay/src/api/media.rs:547`, tests `crates/buzz-relay/src/api/media.rs:1206-1215`).

`MediaStorage::get`/`get_stream`/`get_range`/`delete` take an arbitrary `&str` key with **no validation inside this crate** (`crates/buzz-media/src/storage.rs:105-164`) — key sanitation for read paths is entirely the caller's responsibility.

---

### 5. SSRF

No outbound fetch of user-supplied URLs exists anywhere in the crate. The only network egress is to the configured S3 endpoint via `rust-s3` (`crates/buzz-media/src/storage.rs:34-70`); there is no mirror/BUD-04 implementation, no URL field consumed as an input, and no HTTP client dependency. `public_base_url` is used only for string formatting of response URLs (`crates/buzz-media/src/upload.rs:548`, `crates/buzz-media/src/thumbnail.rs:44`).

Adjacent note: the `aws-creds` fork the workspace pins reads container-credential URIs from the environment and, per the patch rationale, includes "a loopback allowlist for the auth token" (`Cargo.toml:167-169`) — the relevant SSRF-adjacent surface is in that dependency, not in this crate's code.

---

### 6. Hash verification of uploaded bytes — verified

The hash in the key/URL is always the server's own digest of the bytes actually received:

1. Buffered: `hex::encode(Sha256::digest(&bytes))` over the exact `Bytes` that are later PUT (`crates/buzz-media/src/upload.rs:84`, PUT at `:129`).
2. Streamed: `hasher.update(&buf[..n])` on each chunk immediately before `file.write_all(&buf[..n])`, so hash and file content cannot diverge (`crates/buzz-media/src/upload.rs:379-382`); the same temp file is then uploaded (`crates/buzz-media/src/upload.rs:434`).
3. The client's declared hash is checked *against* the computed one — `verify_blossom_upload_auth` requires at least one `x` tag equal to `sha256` else `HashMismatch` (`crates/buzz-media/src/auth.rs:189-196`).
4. The key uses the computed hash (`crates/buzz-media/src/upload.rs:94`, `:421`), so even a matching-but-irrelevant `x` tag cannot rename a blob.

Gaps: no post-PUT read-back verification, and no verification on the read path (`crates/buzz-media/src/storage.rs:105-146`) — a bucket-level tamper (direct S3 write with the relay's credentials, or a compromised store) would be served without detection. The relay does not re-hash either (`crates/buzz-relay/src/api/media.rs:619-760`).

---

### 7. EXIF / metadata privacy

The crate's stance is **reject, not strip** (`crates/buzz-media/src/validation.rs:487-491`): "This is deliberately a structural allowlist rather than an EXIF-tag denylist: location can also live in XMP, comments, PNG text, ICC descriptions, or private chunks. Client encoders remove these before upload."

Coverage: JPEG `APP1..APP13/APP15/COM` and non-canonical `APP0`/`APP14` (`crates/buzz-media/src/validation.rs:539-563`); PNG `eXIf`/`zTXt`/`iTXt`/`iCCP` + all unknown ancillary chunks + `pHYs` (excluded as "an identity channel") (`crates/buzz-media/src/validation.rs:614-651`); WebP `EXIF`/`XMP `/`ICCP`/unknown chunks + `VP8X` metadata presence flags even when the chunk is absent (`crates/buzz-media/src/validation.rs:690-732`); GIF comment/plain-text/non-loop application extensions (`crates/buzz-media/src/validation.rs:783-826`); MP4 `meta ilst keys data uuid xml  bxml loci ©xyz name chap`, non-allowlisted boxes, alternate tracks, timed-metadata tracks (`crates/buzz-media/src/validation.rs:834-846`, `:895-908`, `:325-386`). Trailing-bytes-after-terminator is forbidden in all four image containers, closing the "motion photo"/appended-payload channel (`crates/buzz-media/src/validation.rs:528-532`, `:653-656`, `:698-701`, `:819-823`).

Tests prove the policy against real metadata rather than synthetic markers: a hand-built EXIF GPS IFD verified by an independent parser (`crates/buzz-media/src/validation.rs:1011-1075`), real XMP GPS strings (`crates/buzz-media/src/validation.rs:1112-1162`), QuickTime ISO-6709 `©xyz` coordinates (`crates/buzz-media/src/validation.rs:2411-2432`), and unsanitized iOS/Android encoder output (`crates/buzz-media/src/validation.rs:1203-1280`).

Two deliberate exceptions to note in a privacy review:
- Exactly one PNG `tEXt` chunk keyed `buzz_agent_snapshot` or `buzz_team_snapshot` is permitted, i.e. a sanctioned in-image data channel for agent/team sharing (`crates/buzz-media/src/validation.rs:575-590`, `:601-611`). Duplicate or near-miss keywords are rejected (`crates/buzz-media/src/validation.rs:1328-1366`).
- One exact 53-byte ffmpeg empty-`udta` byte sequence is permitted verbatim in MP4 (`crates/buzz-media/src/validation.rs:834-840`, `:895-903`).

Consequence: privacy depends on **clients** sanitizing. A user uploading an untouched camera JPEG gets a 422, not a stripped image — a usability/robustness tradeoff, not a leak.

---

### 8. Read access, authentication, and signed URLs

- This crate provides the *verifier* `verify_blossom_get_auth` but never calls it (`crates/buzz-media/src/auth.rs:207-236`). Nothing in `MediaStorage`, `upload.rs`, or `validation.rs` gates reads.
- The relay owns the gate: `authenticate_media_read` returns early when `require_media_get_auth` is false, otherwise calls `buzz_media::auth::verify_blossom_get_auth(..., 3600)` (`crates/buzz-relay/src/api/media.rs:489-515`), fed by `BUZZ_REQUIRE_MEDIA_GET_AUTH`, default **false** (`crates/buzz-relay/src/config.rs:677-685`).
- Documented consequence of BUD-01 server-scoped get tokens: they "intentionally grant reads for all blobs on the host until expiration; callers must still apply relay membership after this verifier returns" (`crates/buzz-media/src/auth.rs:201-205`). The `RelayMembershipRequired` variant exists (`crates/buzz-media/src/error.rs:69-70`) but is never constructed here — that check must exist in the relay.
- **No presigned URLs** anywhere: no `presign_get`/`presign_put` call in `crates/buzz-media/src/storage.rs:73-265`. Blobs are served by the relay proxying S3 bytes, so bucket credentials never reach clients.
- Auth replay window: `created_at` freshness is bounded (600 s buffered, 3600 s video — `crates/buzz-media/src/upload.rs:85`, `:412`) *in addition to* the `expiration` tag, explicitly to bound replay (`crates/buzz-media/src/auth.rs:112-122`). Within that window a captured token is replayable; no nonce/jti store exists (the `TokenRevoked` variant is declared but unused here).
- Auth error responses are uniform 401 `"authentication failed"` to avoid an enumeration oracle (`crates/buzz-media/src/error.rs:120-146`).

---

### 9. Tenant isolation of blobs

| Property | Finding |
|---|---|
| Raw blob bytes | **Not** community-scoped — key is `{sha}.{ext}` globally (`crates/buzz-media/src/upload.rs:94`) |
| Metadata sidecar | Community-scoped and described as "the tenant read gate": "A blob in another community must never be observable through a global `_meta/{sha}.json` lookup" (`crates/buzz-media/src/storage.rs:177-182`) |
| Community source | Always `TenantContext` (server-resolved); comment: "Callers must never derive the community from client-supplied blob metadata, URLs, or event tags" (`crates/buzz-media/src/storage.rs:204-209`) |
| Existence-oracle handling | `read_sidecar_mime` collapses read failure and absence to `None` so an A-bound request cannot distinguish a B-only blob from a missing blob (`crates/buzz-media/src/storage.rs:222-233`) |
| Blossom `server` tag | Validated against the *per-request* bound tenant host, not a process-global domain, with `normalize_host` equivalence; unknown host → reject (`crates/buzz-media/src/auth.rs:124-143`) |
| Regression tests | `sidecar_keys_are_community_scoped`, `same_sha_sidecars_do_not_bleed_between_communities` (`crates/buzz-media/src/storage.rs:333-377`), `test_server_tag_normalized_against_bound_host` (`crates/buzz-media/src/auth.rs:470-530`) |

Structural consequence: isolation is *metadata-level*, not *byte-level*. Anyone who can guess or learn a sha256 and reach a relay path that bypasses the sidecar lookup would hit shared bytes. The read path in the relay does perform the sidecar lookup before serving (`crates/buzz-relay/src/api/media.rs:625-660`), so the gate holds for the shipped handler — but the property depends on every future reader honouring it. A second consequence: cross-community dedup is observable as a timing/behaviour difference on upload (the idempotent short-circuit needs the *community's* sidecar, so it does not leak across tenants — `crates/buzz-media/src/upload.rs:97-103`).

---

### 10. `unsafe`

**Verified none.** Zero occurrences of the `unsafe` keyword across `crates/buzz-media/src/{lib,types,thumbnail,config,error,storage,upload_record,auth,upload,bucket_index,validation}.rs` and `crates/buzz-media/tests/static_creds_minio.rs`. All binary parsing uses safe slicing with explicit bounds checks and `try_into()` on fixed-size ranges.

---

### 11. Input-validation gaps and other residual findings

| # | Finding | file:line |
|---|---|---|
| G-1 | No integrity check on read — stored bytes are never re-hashed against their key | `crates/buzz-media/src/storage.rs:105-146` |
| G-2 | `MediaStorage` key-taking methods accept arbitrary keys without validation; callers must sanitize | `crates/buzz-media/src/storage.rs:105-164` |
| G-3 | Unchecked `offset += atom_size` in the moov scanner (attacker-controlled 64-bit size), inconsistent with the `checked_add` used in the sibling box walker | `crates/buzz-media/src/validation.rs:481` vs `:891-893` |
| G-4 | Unsniffable generic uploads accepted as `application/octet-stream`; safety rests on relay response headers | `crates/buzz-media/src/validation.rs:207`, `crates/buzz-media/src/validation.rs:66-74` |
| G-5 | No durable storage quota per pubkey/community; admission limits bound only in-flight work (relay TODO acknowledges it) | `crates/buzz-relay/src/api/media.rs:302-304` |
| G-6 | Orphan blobs are deliberately never cleaned up and no GC exists, so failed/partial uploads accumulate bytes an attacker can grow (bounded per-request, unbounded in aggregate) | `crates/buzz-media/src/upload.rs:122-127` |
| G-7 | `max_image_bytes`/`max_gif_bytes` have no serde default; a config source that omits them fails to deserialize rather than defaulting safely (fail-closed, but easy to misread as "50 MB is the crate default") | `crates/buzz-media/src/config.rs:38-44` |
| G-8 | Blurhash failure is silently swallowed (`unwrap_or_default()`), producing an empty blurhash rather than an error | `crates/buzz-media/src/thumbnail.rs:36-37` |
| G-9 | `head_with_metadata` coerces a missing `content_length` to `0`, which a caller could misread as a zero-byte object | `crates/buzz-media/src/storage.rs:170-172` |
| G-10 | Aggregation counters use unchecked `+=` on `u64` (`physical_bytes += size`); overflow-wrap only, no correctness guard | `crates/buzz-media/src/bucket_index.rs:251-252` |
| G-11 | IP collection is opt-in, fail-empty, and scoped to the `_uploads/` record only — never blob metadata, response, or audit log (privacy-positive, listed for completeness) | `crates/buzz-media/src/upload_record.rs:30-40`, `:191-256` |
| G-12 | `_uploads/` records are stated to be unreachable via the media serve path "by construction" because `validate_media_path` requires a bare 64-hex first segment — a cross-crate invariant, enforced in the relay, not here | `crates/buzz-media/src/upload_record.rs:24-27`, `crates/buzz-relay/src/api/media.rs:547` |


## Module: buzz-workflow (`crates/buzz-workflow`)

### Aspect: Security

---

### 1. Unsafe code

`#![deny(unsafe_code)]` at `crates/buzz-workflow/src/lib.rs:1`. Grep over the crate finds no `unsafe` blocks. **Verified: none.**

---

### 2. Expression-evaluation sandboxing (`evalexpr`)

| Question | Answer | Evidence |
|---|---|---|
| Can an expression read engine/DB state? | No. The context is a fresh `HashMapContext` populated only with the 6 trigger fields, prefixed webhook fields, and step outputs of the current run (`build_eval_context`). No handle to `Db`, the sink, or the filesystem is exposed. | `executor.rs:224-316` |
| Can an expression write state? | Not into the engine. `eval_boolean_with_context` takes `&ctx` (immutable), so assignment operators cannot mutate anything the engine later reads; the context is dropped after evaluation. | `executor.rs:373` |
| Can it call arbitrary functions? | Only the four registered helpers plus `evalexpr`'s own builtins: `str_contains`, `str_starts_with`, `str_ends_with`, `str_len`. No I/O, process, or reflection functions are registered. | `executor.rs:236-283` |
| Can it DoS the engine? | Partially mitigated. Two controls: a 4096-byte expression length cap (`MAX_EXPR_LEN`) and a 100 ms `tokio::time::timeout` around a `spawn_blocking` evaluation. **The code itself documents the gap**: "The spawn_blocking thread cannot be cancelled by tokio::time::timeout — it will run to completion even after timeout" (`executor.rs:358-360`). A pathological ≤ 4 KiB expression therefore keeps occupying a blocking-pool thread after the caller has already errored out. | `executor.rs:342`, `executor.rs:362-368`, `executor.rs:370-380` |
| Who can supply expressions? | Workflow authors only — `filter` and `if:` come from the stored definition, which the relay gates on workflow ownership before upsert (`crates/buzz-relay/src/handlers/command_executor.rs:684`, ownership check at `:838-842`). Not attacker-controlled from the message path. |

Secondary hardening: webhook-derived variables are registered **before** the standard trigger fields so a webhook body cannot shadow `trigger_text`/`trigger_author`, and any webhook key starting with `trigger_` or `steps_` is dropped entirely (`executor.rs:285-296`).

---

### 3. Step-ID validation (evalexpr variable injection)

`WorkflowDef::validate` requires each step id to be non-empty, ≤ 64 chars, and `[A-Za-z0-9_]` only (`schema.rs:168-186`). Rationale is stated in-code: ids become `steps_{id}_output_{field}` variables (`schema.rs:165-167`). A dash id like `my-step` would otherwise be parsed by evalexpr as subtraction — locked down by the test at `schema.rs:756-772`; special characters (`step;drop table`) are rejected at `schema.rs:786-798`. Uniqueness is enforced separately (`schema.rs:187-192`), preventing later steps from silently overwriting an earlier step's output variables.

Residual note: the field portion of `steps_{id}_output_{field}` comes from action **output** JSON keys, all of which are engine-generated literals (`sent`, `event_id`, `status`, `body`, `added`, `response`, `slept_secs`, `skipped`) plus, for `call_webhook`, nothing attacker-controlled — the remote body lands in a single `body` string, not as object keys (`executor.rs:865-868`).

---

### 4. Template-injection surface

- Substitution is single-pass and never re-scans substituted output, so a value containing `{{trigger.text}}` cannot cause a second expansion (`executor.rs:82-121`).
- Unknown variables are emitted literally rather than erroring or resolving to empty, so a typo cannot silently blank a field (`executor.rs:110-117`).
- Templates are **not** evaluated as expressions — `resolve_variable` only performs prefix matching on `trigger.` / `steps.` (`executor.rs:126-151`); there is no code path from a template into `evalexpr`.
- Injection risk is downstream, not in the resolver:
  - `call_webhook.url` **is** templated (`executor.rs:427`), so trigger text can influence the request target. The SSRF check runs *after* resolution (`executor.rs:790-794`), so private targets remain blocked, but an attacker who controls message text of a workflow that templates the URL can steer requests to arbitrary **public** hosts, and header values are likewise templated (`executor.rs:417-424`) — a credential-bearing header could be sent to an attacker-chosen host.
  - `send_message.text` is templated and posted as a relay-signed message (see §7).
  - `send_message.channel` is templated but then must parse as a UUID and, for channel-bound workflows, must equal the bound channel (`executor.rs:477-493`).
- Header **names** and `call_webhook.method` are not templated (`executor.rs:419-426`), so header/method smuggling via trigger data is not possible.
- No shell, SQL, or filesystem sink exists in this crate; all DB access is through parameterized `buzz-db` methods.

---

### 5. SSRF protection on outbound webhooks

| Control | Exact implementation | Line |
|---|---|---|
| Guard | `check_ssrf(host, port)`: resolve via OS resolver on `spawn_blocking`; reject resolver error, reject empty address list, reject if **any** resolved address matches `buzz_core::network::is_private_ip`; return `addrs[0]` | `executor.rs:745-776` |
| Blocked ranges | IPv4 loopback, RFC1918 private, link-local, `0.0.0.0/8`, broadcast, CGNAT `100.64.0.0/10` (called out for cloud metadata reachability), benchmarking `198.18.0.0/15`; IPv6 loopback, unspecified, `fc00::/7`, `fe80::/10`, `ff00::/8`, `2001:db8::/32`, plus (added in `c26bf594`) NAT64 local-use `64:ff9b:1::/48`, Teredo `2001::/32`, 6to4 `2002::/16`, and three recursive embedded-IPv4 paths — IPv4-mapped *and* IPv4-compatible via `to_ipv4()`, NAT64 well-known `64:ff9b::/96`, and IPv4-translated `::ffff:0:0:0/96` | `crates/buzz-core/src/network.rs:46-95` |
| Rebinding TOCTOU | `.resolve(host, SocketAddr::new(safe_ip, port))` pins the validated IP; the client is rebuilt per request specifically for this, at the documented cost of connection pooling | `executor.rs:800-815` |
| Redirect policy | `reqwest::redirect::Policy::none()` — redirects fully disabled with the stated reason that a redirect to an internal host would bypass the check | `executor.rs:812` |
| Proxy bypass (**new** — added by the post-analysis sync, not in the original report) | `.no_proxy()` on the per-request client, with the stated reason that a system proxy would resolve the original hostname itself and so defeat the pinned address | `executor.rs:810` (comment `:808-809`) |
| Response cap | 1 MiB (`WEBHOOK_MAX_RESPONSE_BYTES = 1024 * 1024`), enforced by incremental `resp.chunk()` reads that abort early rather than `resp.bytes()` | `executor.rs:778`, `executor.rs:841-863` |
| Timeout | 10 s total | `executor.rs:807` |

Gaps:
- **No scheme allowlist.** The schema doc says "must be a public HTTPS endpoint" (`schema.rs:120`) but the code only requires `reqwest::Url::parse` to succeed and a host to be present (`executor.rs:786-794`); `http://` is accepted and the port default falls back to 80 (`executor.rs:796-798`).
- **No port restriction** — any port on a public IP is reachable.
- **`add_reaction` has no SSRF guard, no redirect policy, and no response cap** (`executor.rs:888-933`); its base URL comes from `BUZZ_RELAY_BASE_URL`, i.e. operator-controlled rather than workflow-controlled, which is why the guard is absent, but a misconfigured env var is unchecked.
- Multi-address hosts: all resolved addresses are validated, but only `addrs[0]` is used, so there is no fallback attempt — a correct fail-safe rather than a gap.

---

### 6. Inbound webhook secret comparison

Not implemented in this crate. The `webhook` trigger's only inbound path is the relay's `POST /hooks/{id}` handler (`crates/buzz-relay/src/router.rs:120`, `crates/buzz-relay/src/api/bridge.rs:1759-1893`). Findings, for boundary completeness:

| Aspect | Finding | Line |
|---|---|---|
| What is compared | The **secret itself**, not an HMAC of the request body. Provided secret comes from the `X-Webhook-Secret` header, falling back to the `?secret=` query parameter; the stored value is read out of the definition JSON key `_webhook_secret`. There is no request-body signature, so bodies are unauthenticated beyond bearer-secret possession, and no replay protection. | `crates/buzz-relay/src/api/bridge.rs:1815-1835`, `crates/buzz-relay/src/webhook_secret.rs:41-47` |
| Constant-time? | Yes for content: `verify_secret` XOR-folds all bytes without short-circuiting. Length mismatch returns early — documented as acceptable because generated secrets are always 36-byte UUIDs. | `crates/buzz-relay/src/webhook_secret.rs:68-87` |
| Secret entropy | `Uuid::new_v4()` string, documented as 122 bits. | `crates/buzz-relay/src/webhook_secret.rs:22-27` |
| Missing secret | Fails closed with 401 ("webhook secret required but not configured"). | `crates/buzz-relay/src/api/bridge.rs:1810-1815` |
| Storage | Plaintext inside the workflow definition JSON (covered by `definition_hash`), stripped before API responses via `strip_secret`. Because `WorkflowDef` does not set `deny_unknown_fields` (`schema.rs:13-14`), the extra `_webhook_secret` key deserializes cleanly in this crate and is silently dropped on re-serialization. | `crates/buzz-relay/src/webhook_secret.rs:29-66` |
| Tenant binding | The community is bound from the HTTP `Host` header before any lookup; unmapped host / missing workflow both return the same generic 404. | `crates/buzz-relay/src/api/bridge.rs:1772-1788` |

---

### 7. Privilege of relay-signed actions

`send_message` executes with **relay authority**: the run's community and the workflow's `owner_pubkey` are looked up server-side (`executor.rs:535-556`), the owner pubkey is passed only as an attribution value, and the relay keypair signs the event (`action_sink.rs:62-68` documents "the relay keypair signs the event"). Consequences and containment:

- The destination is constrained to the workflow's bound channel when one exists; cross-channel overrides are rejected (`executor.rs:477-493`). Unbound workflows may target any valid channel UUID (`executor.rs:495-503`), so the sink's own channel-existence/archived checks are the remaining fence (`action_sink.rs:22-27`).
- Message **content** is fully attacker-influenceable when a workflow templates trigger text, and it is posted under the relay identity. Impersonation of a *user* is not possible (the signing key is the relay's), but relay-signed content can be produced by any channel member whose message satisfies the trigger filter.
- Manual triggers are restricted to the workflow owner by the relay ("Manual triggers execute with the workflow owner's authority, so only the owner may start them", `crates/buzz-relay/src/handlers/command_executor.rs:836-843`) — channel membership alone is insufficient.
- Approval authorization (`approver_spec`) is enforced relay-side and fails closed for role-style specs (`crates/buzz-relay/src/handlers/command_executor.rs:942-976`), not in this crate.

---

### 8. Approval-token generation / storage / single-use

| Property | Finding | Line |
|---|---|---|
| Generation | `Uuid::new_v4().to_string()`; the doc states it draws from the OS CSPRNG via `getrandom` and deliberately does **not** mix in `run_id`/`step_id` to avoid time-based predictability | `executor.rs:693-700` |
| Storage | **Never stored by this crate.** `TODO (WF-08): create approval record in DB, emit kind:46010` (`executor.rs:663`). Repo-wide grep confirms `Db::create_approval` has no caller outside `buzz-db` itself | `executor.rs:650-668` |
| Transport | Returned in-memory as `StepResult::Suspended { approval_token }` → `ExecutionResult.approval_token`; logged only as `token: <redacted>` (`executor.rs:1189-1193`) | `executor.rs:665-667` |
| Single-use enforcement | Not exercised: the primitive exists in `buzz-db` (tokens stored as SHA-256, `crates/buzz-db/src/workflow.rs:29-35`; status-guarded `UPDATE` documented as TOCTOU-safe so grant and deny cannot both win, `crates/buzz-db/src/workflow.rs:1020-1031`) but no token is ever written, so the guard is unreachable | — |
| Net effect | A run reaching an approval gate is finalized as `Failed` (`lib.rs:192-215`); the token is discarded. There is no persisted secret to leak, and equally no working approval gate |

---

### 9. Workflow-loop prevention

Two layers; **most of it lives in `buzz-relay`, not in this crate**:

| Guard | Where |
|---|---|
| Kinds 46001–46012 excluded | Both: relay pre-check `!is_workflow_execution_kind(kind_u32)` (`crates/buzz-relay/src/handlers/event.rs:528`) and again inside `on_event` (`crates/buzz-workflow/src/lib.rs:293-295`). `is_workflow_execution_kind` is defined in `crates/buzz-core/src/kind.rs:717-719` |
| Relay-signed `buzz:workflow`-tagged messages excluded | **Relay only** — requires `state.relay_keypair` and the tag scan, unavailable to this crate (`crates/buzz-relay/src/handlers/event.rs:521-526`, used at `:507`) |
| `KIND_GIFT_WRAP` (1059) excluded | **Relay only** (`crates/buzz-relay/src/handlers/event.rs:531`) |
| Command kinds excluded | **Relay only** — `is_command_kind(kind_u32)` (`crates/buzz-relay/src/handlers/event.rs:529`) |
| Workflow-authored messages carry the loop-breaking tag | Relay sink emits the `buzz:workflow` tag (`crates/buzz-relay/src/handlers/event.rs:1782-1793`) |

Consequence: a caller invoking `WorkflowEngine::on_event` directly (bypassing the relay handler) gets only the 46001–46012 exclusion. There is also **no depth/fan-out counter and no per-run loop budget** anywhere — a workflow whose `send_message` output satisfies another workflow's trigger in the same channel would be stopped only by the `buzz:workflow` relay-side tag check.

---

### 10. Input-validation gaps

| Gap | Detail | Line |
|---|---|---|
| No `deny_unknown_fields` on any schema type | Unknown YAML/JSON keys silently accepted (relied on for `_webhook_secret`, but also means typo'd action fields pass validation) | `schema.rs:13`, `:36`, `:71`, `:90` |
| `call_webhook.url` unvalidated at parse time | No scheme/host check in `validate()`; only checked at execution | `schema.rs:152-229` vs `executor.rs:786-798` |
| `send_dm.to` unvalidated | No pubkey/npub format check anywhere (the action is stubbed) | `schema.rs:102-107`, `executor.rs:580-584` |
| `request_approval.timeout` never parsed | The string is only interpolated into a log line; `"24h"` default is a literal, not a duration | `executor.rs:653-658` |
| `request_approval.from` never parsed | Passed through templating only; the approver spec check lives relay-side | `executor.rs:446-450` |
| `delay.duration` validated only at run time | Over-270 s delays fail the run instead of failing the save | `executor.rs:671-684` |
| `add_reaction.emoji` unvalidated | Sent verbatim in a JSON body | `executor.rs:592-617` |
| `step.timeout_secs` unbounded | Any `u64` accepted; a large value effectively disables the step timeout | `schema.rs:82-83`, `executor.rs:1136-1151` |
| Definition size unbounded | No cap on step count, template length, or header count in `validate()` | `schema.rs:152-229` |
| Webhook header values templated | Trigger-derived data can be injected into outbound header values (see §4) | `executor.rs:417-424` |
| Trace growth unbounded | Every step appends a trace entry containing the full action output, including up to 1 MiB of webhook response body, and the whole array is written to `execution_trace` | `executor.rs:1179-1183`, `executor.rs:865-868` |
| Emoji trigger matching is byte-exact | `reaction_added { emoji }` compares raw strings, so `"👍"` vs `"thumbsup"` vs `"+"` are distinct — a filter can be bypassed by using an equivalent representation | `lib.rs:807-822` |


## Module: buzz-relay — core & bootstrap (`crates/buzz-relay/src`)
### Aspect: Security

---

#### 1. Auth enforcement architecture

**There is no authentication middleware anywhere in the router.** The only `middleware::from_fn` layers on the app router are `track_metrics` (`router.rs:188`) and, route-scoped, `require_localhost` on the git policy router (`api/git/mod.rs:64`) and `security_headers` on the admin router (`api/admin/mod.rs:38`). Every auth decision lives inside a handler body.

Consequences:
- Adding a route registration without adding an in-handler auth call yields a silently unauthenticated endpoint. Nothing structural prevents it.
- The relay-core files in this group contain **zero** auth checks of their own; they only bind tenancy (`router.rs:288`), refuse during shutdown (`router.rs:311`), cap frame size (`router.rs:324`), and cap concurrency (`state.rs:514-516`).

#### 2. Unauthenticated surface reachable from the routers

| Endpoint | Listener | Line | What it exposes |
|----------|----------|------|-----------------|
| `GET /` (NIP-11 branch) | app | `router.rs:63`, `:275-277`, `:333` | full NIP-11 doc incl. `self` pubkey, community `icon`, push descriptor with the relay's executor pubkey (`nip11.rs:207`) |
| `GET /info` | app | `router.rs:64` | identical document |
| `GET /.well-known/nostr.json` | app | `router.rs:65` | NIP-05 mapping |
| `GET /health` | app | `router.rs:67` | `"ok"` |
| `GET /_liveness` | app **and** health | `router.rs:68`, `:227` | `"ok"` |
| `GET /_readiness` | app **and** health | `router.rs:69`, `:228` | `{"status","postgres","redis"}` — **discloses which backing store is down** (`router.rs:369-372`) |
| `GET /_status` | health | `router.rs:229`, `:387-394` | service name, **exact build version** (`CARGO_PKG_VERSION` = `0.2.0`), uptime |
| `GET /_mesh` | health | `router.rs:230`, `:380-386` | full mesh peer table: `runtime_id`, **`endpoint_addrs`** (internal socket addresses), `proto_version`, `phi`, `load`, `record_version`, `last_heartbeat_millis`, per-peer counters (`crates/buzz-relay-mesh/src/status.rs:8-29`) |
| `GET /metrics` | metrics | `metrics.rs:73` | every gauge/counter/histogram, **including per-community series labelled by host** (`main.rs:1341-1359`, `:1546-1552`) — i.e. an enumeration of tenant hostnames |
| `GET /api/join-policy`, `/terms`, `/privacy` | app | `router.rs:96`, `:99-106` | operator ToS/privacy Markdown |
| `GET /media/{sha256_ext}`, `HEAD` same | app | `router.rs:41-43` | **media blobs — auth is off by default** (`require_media_get_auth`, `config.rs:196-197`, default `false` at `config.rs:682-689`) |
| SPA fallback `/invite/{code}`, `/assets/*` | app | `router.rs:145-183` | static bundle |
| `/` , `/repos`, `/repos/*` SPA | app | `router.rs:210-212` | only when `serve_git_web_gui` |

##### The three hard-coded `0.0.0.0` binds
- Health listener: `("0.0.0.0", config.health_port)` (`main.rs:1116`) — ignores `BUZZ_BIND_ADDR`'s IP.
- Metrics listener: `([0,0,0,0], port)` (`metrics.rs:74`) — same.
- Mesh default bind: `"0.0.0.0:3478"` (`config.rs:507`).

So `/_status`, `/_mesh`, and `/metrics` are exposed on **all** interfaces regardless of `BUZZ_BIND_ADDR`, with no auth, no CORS restriction, and no rate limit. `/_mesh` leaking `endpoint_addrs` is internal-topology disclosure; `/metrics` leaking per-community host labels is tenant enumeration. Both are normally shielded by K8s network policy, but neither is shielded by the application.

#### 3. Tenancy isolation enforcement

Strong points, all verified:

| Control | Cite |
|---------|------|
| Community is bound from the request `Host` **before** the WebSocket upgrade, so no frame is read on an unbound connection | `router.rs:279-300` |
| Empty/whitespace host fails closed **before** the resolver, guarding against a misconfigured `host = ''` row that the schema permits | `tenant.rs:81-88` |
| Unmapped host and lookup error produce a **single generic** `404`, never echoing the host — no probe oracle for which communities exist | `router.rs:292-297`, `tenant.rs:56-59` |
| No default/fallback community path exists anywhere in the seam | `tenant.rs:89-95` |
| Server-internal paths (git, hook, workflow sink, startup) resolve `relay_url` through the same fail-closed helper, not a bypass | `tenant.rs:120-133` |
| Background sweeps build a per-row `TenantContext` from the DB `RETURNING`, never a default | `main.rs:637-644`, `main.rs:741-746` |
| Every cache key is `(CommunityId, …)`-prefixed; predicate invalidation is community-scoped, with over-invalidation as the failure mode rather than serving stale access state | `state.rs:541-553`, `state.rs:881-895`, `state.rs:928-975` |
| Local-echo dedup is community-keyed so a publish in A cannot suppress a same-id event in B | `state.rs:530-540` |
| Ban-driven disconnects are fenced to the banning community | `state.rs:296-308`, `state.rs:328-334` |
| Fan-out compares the receiver-side community label recorded at handshake against the event label | `state.rs:52-54` |
| NIP-11 `build` is pinned by a compile-time const to a static/scalar-only signature so it can never grow an unscoped DB/search/audit input | `nip11.rs:307-335` |
| NIP-98 replay prevention is Redis-backed and community-keyed; the doc forbids replacing it with process-local caching | `state.rs:576-581`, `state.rs:710-711` |
| Admin host is short-circuited **first** in both the `/` handler and the SPA fallback, so it can never serve the public bundle, NIP-11, or the WebSocket endpoint | `router.rs:252-273`, `router.rs:157-168` |

##### Gaps in tenancy isolation
- `/metrics` publishes per-community gauges labelled with the tenant **host** (`main.rs:1341-1359`), which is exactly the information the `404` at `router.rs:292` refuses to disclose. `BUZZ_USAGE_METRICS_PER_COMMUNITY=off` removes it, but the default is `all` (`main.rs:65`).
- `GET /` NIP-11 is served fail-open before binding (deliberate, `router.rs:279-286`) but it also performs 1–3 Postgres queries per request (`nip11.rs:237`, `:246`, `:278`, `:283`), so the unauthenticated path is DB-coupled.

#### 4. Secret handling

##### `Config` is `Debug` with no redaction
`#[derive(Debug, Clone)]` at `config.rs:50` covers fields that hold secrets in cleartext:

| Secret field | Line |
|--------------|------|
| `database_url` (contains the PG password) | `config.rs:55` |
| `read_database_url` | `config.rs:58` |
| `redis_url` (may contain a password) | `config.rs:60` |
| `relay_private_key` | `config.rs:92` |
| `git_hook_hmac_secret` | `config.rs:239` |
| `media.s3_secret_key` / `s3_access_key` (via `buzz_media::MediaConfig`) | `config.rs:187`, populated `config.rs:622-625` |

Any `{:?}` / `tracing::debug!(?config)` would print all of them. Verified that the code does **not** currently do this: the startup log at `main.rs:128-136` selects 6 non-secret fields, and `AppState`'s manual `Debug` prints only `relay_url` and `max_connections` (`state.rs:1209-1215`). So the derive is a loaded gun, not a live leak. There is no `SecretString`/`Zeroize` wrapper anywhere in `config.rs`.

##### Secret-adjacent logging that does happen
- `RELAY_OWNER_PUBKEY` is echoed on invalid input: `warn!("RELAY_OWNER_PUBKEY is not a valid 64-char hex pubkey — ignoring. Got: {s:?}")` (`config.rs:538-542`). A pubkey is not secret, but a mis-set env var (e.g. an nsec pasted into the wrong variable) would be logged verbatim.
- `RELAY_OPERATOR_PUBKEYS` echoes the offending entry in the config error (`config.rs:568-570`).
- The dev relay pubkey is logged at `warn` (`main.rs:402-407`).

##### Weak-secret defaults
| Item | Behaviour | Cite |
|------|-----------|------|
| `BUZZ_GIT_HOOK_HMAC_SECRET` | unset ⇒ a fresh random 32-byte hex secret **per process**. In a multi-pod deployment each pod has a different hook secret. Explicitly-set values must be ≥32 chars | `config.rs:739-744`, `config.rs:862-871` |
| `BUZZ_RELAY_PRIVATE_KEY` | unset **and** `require_auth_token=false` ⇒ hard-coded `0000…0001` with a `warn` | `main.rs:396-408` |
| `BUZZ_S3_ACCESS_KEY` / `BUZZ_S3_SECRET_KEY` | default to `"buzz_dev"` / `"buzz_dev_secret"` | `config.rs:622-625` |
| `DATABASE_URL` | defaults to `postgres://buzz:buzz_dev@localhost:5432/buzz` (annotated `// sadscan:disable np.postgres.1`) | `config.rs:410-411` |

Every one of these is a *dev* default that silently activates in production if the env var is missing. `require_auth_token=false` is itself the default (`config.rs:475-477`), so the hard-coded relay key path is the default path for a bare deployment.

#### 5. CORS and response headers

| Condition | Result | Cite |
|-----------|--------|------|
| `BUZZ_CORS_ORIGINS` unset/empty (**the default**) | `CorsLayer::permissive()` over the entire app router — media, git, invites, operator, moderation, admin | `router.rs:410-412`, `config.rs:595-600` |
| Origins set, at least one parses | allow-list origins, `allow_methods(Any)`, `allow_headers(Any)`; **no `allow_credentials`** | `router.rs:428-431` |
| Origins set, none parse | bare `CorsLayer::new()` after an `error!` — refuses to degrade to permissive | `router.rs:419-426` |

`CorsLayer::permissive()` in `tower-http` sends `Access-Control-Allow-Origin: *` with `Any` methods/headers. Combined with the NIP-98-signed-request model this is not a session-hijack vector (no cookies, no ambient credentials), but it does let any web origin read `/_readiness`, `/info`, `/api/join-policy`, and — when GET auth is off — `/media/{sha256}` blobs cross-origin.

Security headers are applied on **exactly one** router: `api/admin/mod.rs:43-56` sets `Cache-Control: no-store`, `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, and a referrer policy. The public app router, the SPA fallback (`router.rs:214-219`, `router.rs:145-183`), the health router (`router.rs:225-232`), and `/metrics` have **no** security headers — no CSP, no HSTS, no `nosniff`, no frame protection. The SPA fallback serves `text/html` from `web_dir` (`router.rs:216`) with no CSP.

#### 6. Panic / DoS surface

| Surface | Assessment | Cite |
|---------|-----------|------|
| Frame size | capped at the parser (`max_message_size` + `max_frame_size`) before assembly, default 512 KiB | `router.rs:324-332`, `config.rs:15` |
| HTTP body size | 1 MB on the api router, `max(image,video)` on media (default 500 MB), `git_max_pack_bytes` on git (default 500 MB), 1 MB on git policy, 1 KB on admin | `router.rs:129`, `:44`, `api/git/transport.rs:1763`, `api/git/mod.rs:63`, `api/admin/mod.rs:39` |
| Concurrent connections | `conn_semaphore`, default 10,000, `try_acquire_owned` | `state.rs:729`, `config.rs:449-452` |
| Concurrent handlers | `handler_semaphore`, default 1,024 | `state.rs:730` |
| Slow-client backpressure | cancels after `slow_client_grace_limit` (default 15) consecutive full buffers | `state.rs:451-478` |
| Subscription count | 1,024 per connection | `handlers/req.rs:26/66` |
| Filters per REQ / sub-id length | 10 / 256 bytes | `protocol.rs:14`, `:11` |
| **Per-IP connection limiting** | **implemented but never called.** `RateLimiter::check_ip_connection` exists (`crates/buzz-auth/src/rate_limit.rs:188`) and `RedisRateLimiter` implements it (`crates/buzz-pubsub/src/rate_limiter.rs:112`), but the only other reference in the workspace is the test stub at `admission.rs:85`. There is no IP-level flood control | verified by grep |
| Client IP | derived in the router (`router.rs:236-240`) and stored on `ConnectionState` (`connection.rs:61`, `:170`) but **never read in production** — only test fixtures read it (`state.rs:1358`, `handlers/event.rs:1388`). Not used for limiting, logging, or audit | verified by grep |
| UDS clients have no IP at all | `main.rs:1179` uses `.into_make_service()` (no `ConnectInfo`), so `router.rs:239` falls back to `0.0.0.0:0` for every UDS connection | |
| Unbounded per-pubkey `DashMap`s | `observer_rate_limiter` (`state.rs:589`) and `media_upload_rate_limiter` (`state.rs:592`) have **no capacity cap, no TTL, no eviction** in this file. Keys are `(community, caller-chosen pubkey bytes)`. `invite_claim_rate_limiter` explicitly documents this exact risk and defends with a moka capacity cap (`state.rs:593-596`) — the same defence is missing on the other two | memory-growth DoS |
| Unauthenticated DB amplification | `GET /` and `GET /info` each issue 1–3 Postgres queries with no cache and no rate limit; push is enabled by default so the 3-query path is the default | `nip11.rs:237/246/278/283`, `config.rs:752-757` |
| Boot-time panics | 17 `expect` in `metrics.rs:79-145` (compile-time literals), `expect` on the rustls provider (`main.rs:90`), on SIGTERM install (`main.rs:1230`), on the git store (`state.rs:701`) and pack cache (`state.rs:708`); `panic!` at `main.rs:409` | all boot-time, not request-reachable |
| Request-path panic candidates | `state.rs:446` `WsUtf8Bytes::try_from(...).expect("relay fan-out frames are serialized UTF-8 JSON")` — panics inside the fan-out path if a caller ever passes non-UTF-8; `main.rs:1253` `expect("valid event JSON from DB row")` in the reminder scheduler — panics the scheduler task on a malformed DB row; `metrics.rs:180` `path.unwrap()` (guarded by the `None` arm above, safe) | |
| `debug_assert` in release | `nip11.rs:143` — the NIP-43-without-`self` inconsistency check is compiled out of release builds | |
| Hard exit | `std::process::exit(1)` at `main.rs:1153` bypasses the audit drain and OTEL flush | |

`#[deny(unsafe_code)]` at `lib.rs:1`, **0 unsafe blocks**, and **0 `TODO`/`FIXME`/`HACK`/`XXX` markers** across all 12 files.

#### 7. Auth-relevant configuration defaults (all permissive)

| Setting | Default | Effect at default | Cite |
|---------|---------|-------------------|------|
| `BUZZ_REQUIRE_AUTH_TOKEN` | `false` | REST bypasses token auth; a startup `warn` is emitted. Also unlocks the hard-coded dev relay key | `config.rs:475-477`, `:590-594`, `main.rs:396-408` |
| `BUZZ_PUBKEY_ALLOWLIST` | `false` | any NIP-42 pubkey may connect | `config.rs:479-481` |
| `BUZZ_REQUIRE_RELAY_MEMBERSHIP` | `false` | relay-level membership check is a no-op for all authenticated callers | `config.rs:483-485` |
| `BUZZ_REQUIRE_MEDIA_GET_AUTH` | `false` | media GET/HEAD served without Blossom auth or membership | `config.rs:682-689` |
| `BUZZ_CORS_ORIGINS` | unset | permissive CORS | `router.rs:410-412` |
| `BUZZ_ALLOW_NIP_OA_AUTH` | `false` | fail-closed (good default) | `config.rs:520-523` |
| `RELAY_OPERATOR_PUBKEYS` | empty | community provisioning **disabled** — fail-closed (good default) | `config.rs:164-168`, `:555-576` |
| `BUZZ_MESH` | off | nothing bound — fail-closed (good default) | `config.rs:492-509` |
| `BUZZ_AUTO_MIGRATE` | off | no schema mutation — fail-closed (good default) | `main.rs:25-33` |
| `BUZZ_AUDIT_ENABLED` | **`true`** | audit on by default (good default) | `config.rs:793` |
| `BUZZ_GIT_CONFORMANCE_PROBE` | on (`!= "false"`) | S3 CAS is proven before serving git — fail-closed (good default) | `main.rs:469-471` |

The five permissive defaults at the top of the table compose into a fully open relay from a bare `docker run` with only `DATABASE_URL`/`REDIS_URL` set: no REST token, no allowlist, no membership, unauthenticated media reads, `*` CORS, and a deterministic well-known signing key. Each individually has a rationale (dev ergonomics) and one emits a warning; the composition does not.

#### 8. Deliberate fail-closed behaviours worth preserving

| Behaviour | Cite |
|-----------|------|
| Rate-limit admission denies when Redis is unreachable | `admission.rs:29-33` |
| Host binding rejects on lookup error, not just unmapped | `tenant.rs:93-94` |
| Empty host rejected before the resolver | `tenant.rs:85-87` |
| Channel-visibility cache stores only `private`, never a permissive value, so the worst stale entry is over-restrictive | `state.rs:1106-1122` |
| Cache-invalidation-closure failure over-invalidates rather than serving stale access | `state.rs:886-894`, `:932-974` |
| Community-lifecycle revalidation retains sockets on lookup failure but keeps sweeping | `state.rs:1082-1085` |
| Push lease acceptance is disabled without an exact gateway URL, so no undeliverable work accumulates | `main.rs:684-691`, `config.rs:342-360` |
| Git policy endpoint 403s when `ConnectInfo` is absent (`unwrap_or(false)`) — so the UDS listener, which has no `ConnectInfo`, cannot reach it | `api/git/mod.rs:41-48`, `main.rs:1179` |
| Mesh demo echo requires **two** explicit opt-ins | `router.rs:121-123`, `config.rs:514-518` |
| Membership enforcement makes six otherwise-non-fatal startup steps fatal | `main.rs:201`, `:214`, `:242`, `:250`, `:276`, `:296` |


## Module: buzz-relay — WebSocket protocol & subscriptions (`crates/buzz-relay/src`)
### Aspect: Security

---

#### 1. What an UNAUTHENTICATED socket can do

Reached: TCP connect + WS upgrade to a host that resolves to a community (`router.rs:286-315`). Budget: one `conn_semaphore` permit, held up to **5 s** (`connection.rs:27`, `:228-252`), and up to `max_frame_bytes` (512 KiB default) per frame with **no** frame-rate limit.

| Action | Allowed? | Site |
|---|---|---|
| Receive the AUTH challenge | yes | `connection.rs:182-186` |
| Send `AUTH` (any number of times until one succeeds or one fails) | yes | `connection.rs:503-509` |
| Send `EVENT` | rejected `OK false "auth-required: not authenticated"` | `event.rs:667-674` |
| Send `REQ` | rejected `NOTICE` + `CLOSED` | `req.rs:76-85` |
| Send `COUNT` | rejected `CLOSED` | `count.rs:42-48` |
| Send `CLOSE` | **fully executed** — `conn.subscriptions.remove`, `sub_registry.remove_subscription`, conditional `release_topic`, `CLOSED ""` reply | `connection.rs:581-583`; `close.rs:12-30` |
| Send arbitrary malformed JSON | parsed, rejected with a `NOTICE`, socket stays open, **unmetered** | `connection.rs:490-496` |
| Send oversized frames | one `NOTICE` then disconnect | `connection.rs:421-434`, `:440-452` |
| Send `Ping` | Pong'd through the priority control channel | `connection.rs:464-472` |
| Consume `handler_semaphore` permits | **no** — permits are taken after admission, and all three permit-taking branches require nothing, but the handlers themselves reject on auth *after* taking the permit | `connection.rs:513-521` then `event.rs:621` |
| Consume admission quota | **no** — admission short-circuits `true` for non-`Authenticated` | `connection.rs:604-609` |
| Read any event | **no** — every delivery path requires a registered subscription, which requires REQ, which requires auth | `req.rs:76-85` |
| Receive a fan-out event | **no** — `pubkey_for_conn` returns `None`, and both the author-only fence (`event.rs:150-153`) and the private-channel fence (`event.rs:207-209`) drop such recipients. Open/channel-less events would pass, but the connection holds no subscription to match | `event.rs:135-195` |

**Net exposure of an unauthenticated socket:** it can (a) hold a connection slot for 5 s, (b) burn CPU on `serde_json` parsing of up to 512 KiB frames at line rate, and (c) obtain `handler_semaphore` permits transiently. It cannot read data, write data, or affect other tenants' state.

Note that the `handler_semaphore` permit for an unauthenticated EVENT/REQ/COUNT **is** acquired before the auth check runs (`connection.rs:513` → `event.rs:611`), so an unauthenticated client can transiently occupy up to `max_concurrent_handlers` (1024) permits within its 5 s window. The window bounds it.

---

#### 2. Authorization decision points — write path

In order, all in `handle_event` unless noted:

| # | Decision | Site | Fails |
|---|---|---|---|
| W1 | `AuthState::Authenticated`? | `event.rs:634-654` | closed |
| W2 | `event.pubkey == auth.pubkey` (kind 1059 exempt) | `event.rs:636-645` | closed |
| W3 | kind ≠ 22242 | `event.rs:647-655` | closed |
| W4 | observer frames: `MessagesWrite` scope | `event.rs:681-689` | **permissive when `scopes == []`** |
| W5 | observer frames: NIP-44 content, tag arity, direction derivation, ±300 s freshness | `event.rs:949-958`, `:1071-1109` | closed |
| W6 | observer frames: caller is the agent's proven owner (session fast-path, else 5-min cache, else DB) | `event.rs:986-1030` | closed; DB error denies |
| W7 | observer frames: 100/s per `(community, agent)` telemetry cap | `event.rs:1055-1067` | closed |
| W8 | ephemeral kinds: `MessagesWrite` scope | `event.rs:675-684` | **permissive when `scopes == []`** |
| W9 | ephemeral: Schnorr signature + id hash on `spawn_blocking` | `event.rs:771-791` | closed |
| W10 | ephemeral with `#h`: channel membership | `event.rs:808-820` | closed |
| W11 | persistent: everything inside `ingest_event` (per-kind scope allowlist, verification, ±900 s drift, 256 KiB content cap, membership, relay-only kinds, ban) | `event.rs:728`; `ingest.rs:1393+` | closed |

Pre-dispatch fences that also gate writes: WS admission (`connection.rs:498-500`), `handler_semaphore` (`:513-521`), frame size (`:421`), and the connection-level ban seam at auth (`auth.rs:105-183`).

---

#### 3. Authorization decision points — read path

##### REQ (`req.rs:44-418`)

| # | Decision | Site |
|---|---|---|
| R1 | `AuthState::Authenticated` | `req.rs:50-87` |
| R2 | `MessagesRead` scope (permissive when `scopes == []`) | `req.rs:54-61` |
| R3 | ≤1024 subscriptions for a new sub_id | `req.rs:65-72` |
| R4 | accessible-channel set resolved (10 s cache) | `req.rs:89-104` |
| R5 | narrowed by scoped-token `channel_ids` | `req.rs:105-107` |
| R6 | subscription-level `#h` channel access, with a DB-confirmed repair bounded by R5 | `req.rs:130-174` |
| R7 | **p-gate** — only when `channel_id.is_none()` | `req.rs:182-190` |
| R8 | engram gate (kind 30174) — only when `channel_id.is_none()` | `req.rs:191-197` |
| R9 | author-only filter gate — only when `channel_id.is_none()` | `req.rs:198-204` |
| R10 | per-filter SQL scope: single `#h` → `channel_id`; global → `channel_ids = accessible` | `req.rs:265-283`, `:993-1002` |
| R11 | per-row: `filters_match` for *that* filter | `req.rs:372-374` |
| R12 | per-row: `stored.channel_id ∈ accessible_channels` | `req.rs:376-380` |
| R13 | per-row: **one combined result gate** `event_visible_to_reader` — author-only kinds, the kind-30175 persona shared-gate, and `reader_authorized_for_event` (kinds 30622 / 44200 owner-only) | `req.rs:388`; helper `req.rs:1222-1234` |
| R14 | pre-filter SQL pushdown for kind 30175: `params.persona_reader` is set whenever the filter can match 30175, so the visibility predicate is applied **before** `ORDER … LIMIT` | `req.rs:296-297`; clause `crates/buzz-db/src/event.rs:504-527` |

**Changed by `ab3af828` (post-analysis sync).** R13 and R14 were two separate
per-row predicates (`reader_authorized_for_event` then `is_author_only_event`);
they are now a single `event_visible_to_reader` call that also enforces the new
kind-30175 gate, and a SQL-level pre-filter was added so the gate cannot starve
shared rows off a page.

Search REQ adds the same R11–R13 chain per hit (`req.rs:705`) plus `#h`∩accessible intersection (`req.rs:568-583`) and `ChannelScope` construction (`req.rs:483-501`).

##### COUNT (`count.rs:30-286`)

R1 (`:37-51`), **no R2**, R4/R5 (`:78-97`), p-gate/engram/author-only **unconditionally** (`:53-76`), per-filter `#h` access with the same repair (`:120-151`), then either the exact SQL path or the bounded fallback with per-row R11/R13 (`:202`, `:271`). Since `ab3af828` the fast SQL path is additionally bypassed whenever the filter can match kind 30175 (`count.rs:110`, guards `:174`, `:243`), because `count_events()` has no per-event gate and would otherwise over-count foreign unshared personas.

##### Fan-out (`event.rs:115-195`, `:457-491`)

| # | Fence | Site |
|---|---|---|
| F1 | receiver-side tenant label: `community_for_conn(conn) == Some(event_community)` | `event.rs:126-133` |
| F2 | author-only kinds → only the author's connection (unauthenticated fails closed) | `event.rs:139-152` |
| F2b | kind 30175 → author's own connections always; foreign connections only when the event carries `["shared","true"]`; unknown pubkey fails closed | `event.rs:154-175` |
| F3 | channel-less → pass through | `event.rs:177-179` |
| F4 | private channel → per-recipient `is_member_cached`; visibility error empties the list | `event.rs:180-221` |
| F5 | kinds 30622 / 44200 → recipient pubkey must equal the event's `#p` — **only in `dispatch_persistent_event_inner`** | `event.rs:457-491` |

F2b is inside `filter_fanout_by_access`, so unlike F5 it also covers the Redis
cross-node `subscribe_local` path (`event.rs:307`).

---

#### 4. The p-gate, precisely

`p_gated_filters_authorized` (`req.rs:1038-1070`) returns `true` only if **every** filter satisfies:

```
can_match = filter.kinds.is_none_or(|ks| ks.iter().any(|k| P_GATED_KINDS.contains(k)))
if !can_match                                            → OK
if !explicitly_names(30622|44200) && ids non-empty       → OK          (ids exemption)
else require: #p non-empty AND every #p value == authed_pubkey_hex
```

`P_GATED_KINDS` = `{24200, member-added, member-removed, 1059, 30622, 44200}` (`buzz-core/src/kind.rs:146-156`).

Because `kinds: None` makes `can_match` true, **a filter that omits `kinds` is gated** — this is the mechanism behind AGENTS.md's "relay queries must specify `kinds`, or you get a p-gate 403". Two corrections to that framing:

1. The **403 is HTTP-only**. On HTTP `/query` and `/count` the failure is `StatusCode::FORBIDDEN` with `"restricted: p-gated kinds require #p tag matching your pubkey"` (`api/bridge.rs:981-985`, `:1406-1410`). On WS REQ it is `CLOSED "restricted: p-gated events require #p matching your pubkey"` (`req.rs:184-189`); on WS COUNT it is `CLOSED` with the HTTP wording (`count.rs:56-61`).
2. Omitting `kinds` is not itself fatal — supplying a non-empty `ids` also clears the gate (`req.rs:1062-1064`), and any filter whose named kinds contain no p-gated kind clears it outright.

Comparison of the `#p` value uses `==` on hex strings (`req.rs:1070-1073`) while the engram and author-only gates use `eq_ignore_ascii_case` (`req.rs:1115-1118`, `:1216-1219`). Since `PublicKey::to_hex()` is lowercase and `hex::encode` is lowercase, an uppercase client-supplied `#p` fails the p-gate but would pass an equivalent engram check. Inconsistent, and it fails **closed** for the p-gate, so it is a usability wart rather than a hole.

##### Finding S1 (HIGH) — the p-gate does not run for channel-scoped REQ

`req.rs:182` wraps all three sensitive-kind gates in `if channel_id.is_none()`. The in-code justification (`req.rs:179-181`) is: *"channel-scoped subs can never receive globally-stored events because of the `fan_out()` invariant in subscription.rs."*

That justification covers **live fan-out** correctly (`subscription.rs:265-330`). It does **not** cover **historical delivery**, which runs unconditionally at `req.rs:261-406`, nor does it hold for p-gated kinds that are genuinely channel-scoped. Verified:

- `KIND_GIFT_WRAP` (1059) is **not** in `is_global_only_kind` (`ingest.rs:379-436`), so a gift wrap carrying an `h` tag is stored with `channel_id = Some(ch)`.
- `reader_authorized_for_event` gates only 30622 and 44200 (`buzz-core/src/filter.rs:25-33`) — **not** 1059, not 24200, not member notifications. So R13 does not backstop it.
- `is_author_only_event` covers only reminders and push leases (`buzz-core/src/kind.rs:120`). Not a backstop either.

Result: an authenticated member of channel `X` can send `["REQ","s",{"#h":["X"],"kinds":[1059]}]` and the p-gate is skipped; the historical query is `channel_id = X` with no `#p` predicate, and R11–R14 do not exclude another member's gift wraps. Every other member's channel-scoped gift wraps in `X` become readable ciphertext plus full metadata envelope (sender, recipient `#p`, timestamps).

The HTTP bridge does **not** have this hole: `api/bridge.rs:981` applies the p-gate to all filters unconditionally. The WS path is the weaker of the two.

*What I verified:* the gate condition, the kind-set memberships, the absence of a result-level backstop for 1059, and that `KIND_GIFT_WRAP` is absent from `is_global_only_kind`. *What I did not verify:* whether the shipped clients actually attach `h` tags to gift wraps (that determines whether exploitable rows exist today, not whether the code path is open).

##### Finding S2 (MEDIUM) — `#p` gating relies on `Filter::generic_tags`, not on tag semantics

Both `p_gated_filters_authorized` (`req.rs:1039`) and `result_gated_count_safe_for_pushdown` (`req.rs:1188-1191`) read `filter.generic_tags[SingleLetterTag::lowercase(P)]`. `filter_match_one` matches the same tag key by `tag.kind().to_string() == "p"` (`buzz-core/src/filter.rs:70-78`). These agree today. But the *fan-out* index key is built from `event_p_tag_values`, which also compares `tag.kind().to_string() != "p"` (`subscription.rs:526`) with **no normalisation of the value**, so an event tagging `#p` with uppercase hex lands under a key that no lowercase-hex subscription will ever match — a silent non-delivery, not a leak.

---

#### 5. Result-gated kinds: three hardcoded copies of one list

`RESULT_GATED_KINDS = [KIND_DM_VISIBILITY, KIND_AGENT_TURN_METRIC]` (`buzz-core/src/kind.rs:129`).

| Site | Uses the constant? |
|---|---|
| `req.rs:1165-1170` `filter_can_match_result_gated_kinds` | **yes** — `RESULT_GATED_KINDS.contains(...)` |
| `buzz-core/src/filter.rs:25-27` `reader_authorized_for_event` | **no** — `if kind != KIND_DM_VISIBILITY && kind != KIND_AGENT_TURN_METRIC` |
| `event.rs:461-462` `owner_only_kind` in `dispatch_persistent_event_inner` | **no** — `kind_u32 == KIND_DM_VISIBILITY \|\| kind_u32 == KIND_AGENT_TURN_METRIC` |
| `req.rs:1058-1063` ids-exemption carve-out | **no** — same two-way `==` |

`ab3af828` added a fourth per-event read class (kind 30175) and, unlike the
three copies above, routed every read surface through one shared helper —
`event_visible_to_reader` (`req.rs:1222-1234`) — with the kind test itself
centralised in `buzz_core::kind::is_persona_shared_kind`
(`crates/buzz-core/src/kind.rs:192-194`). That is the shape this drift needs;
the pre-existing `RESULT_GATED_KINDS` copies were not consolidated into it.

So the relay **compounds** the `buzz-core/src/filter.rs:25` drift with two more copies. Adding a third result-gated kind to `RESULT_GATED_KINDS` would silently gate COUNT's pushdown decision while leaving the actual result-level check (`filter.rs`), the live fan-out owner fence (`event.rs`), and the ids-exemption carve-out (`req.rs`) wide open — a fail-**open** divergence. Severity: MEDIUM now, HIGH the moment a kind is added.

##### Finding S3 (MEDIUM) — the owner fence is missing on the cross-node path

F5 (`event.rs:457-491`) is implemented **only** inside `dispatch_persistent_event_inner` (`event.rs:396`). `fan_out_pubsub_event` (`event.rs:282`) and `fan_out_event_to_local_subscribers` (`event.rs:241`) apply F1–F4 (and, since `ab3af828`, F2b) only. Since neither 30622 nor 44200 is in `AUTHOR_ONLY_KINDS`, F2 does not substitute. A kind 30622 / 44200 event that reaches a *second* pod over Redis is fanned out to any subscription whose filters match — including a kindless `ids:[…]` subscription, exactly the case F5's own comment says it exists to close (`event.rs:457-460`). The doc comment at `event.rs:235-240` asserts the two exception paths are "equivalent to this helper plus their own extra step"; for `fan_out_pubsub_event` that is **not** true — it has one fewer step.

Mitigating: on the origin pod the event is echo-suppressed on arrival (`event.rs:300-304`), so single-pod deployments are unaffected. This bites only multi-pod.

**Still open after `ab3af828`.** That commit added an analogous per-event
delivery gate for kind 30175 and placed it inside `filter_fanout_by_access`
(`event.rs:154-175`) — the shared chokepoint — so the persona gate does *not*
have this cross-node hole. Moving the F5 owner fence to the same place is the
already-demonstrated fix.

---

#### 6. Rate-limit enforcement placement

| Element | Site |
|---|---|
| Invocation, before dispatch, for every frame | `connection.rs:498-500` |
| Implementation | `connection.rs:594-653` |
| Metered message set — `EVENT`, `REQ`, `COUNT` only | `connection.rs:599-602` |
| **Pre-auth bypass** — non-`Authenticated` returns `true` immediately | `connection.rs:604-609` |
| Limiter | `state.admission_rate_limiter: Arc<RedisRateLimiter>` (`state.rs:584`, constructed `:712`) |
| Fail-closed on limiter error | `admission.rs:34-40` → `connection.rs:670-677` |
| Community-scoped Redis key | `buzz-auth/src/rate_limit.rs:153-156` |

Correction to a prior batch's note: the pre-auth bypass is `connection.rs:604-609` (the `match &*auth { … _ => return true }`), and the metered-set filter is `:599-602`.

The rate limiter is **pubkey-keyed by design**, which is exactly why it cannot cover pre-auth traffic. The intended companion control for that window — `LimitType::IpConnections` / `RateLimiter::check_ip_connection` — is implemented (`buzz-pubsub/src/rate_limiter.rs:112`, trait `buzz-auth/src/rate_limit.rs:188`) and **never called from anywhere in the relay**. Confirmed by a workspace-wide grep: the only occurrences are the trait declaration, the two impls, and a test stub (`admission.rs:85`).

**Delta vs `ARCHITECTURE.md`.** §9 item 2 (`ARCHITECTURE.md:823`) claims "No rate limiting implementation … `RateLimitConfig` defines 4 tiers … but none are enforced." Two of the tiers (`human_messages_per_min`, `agent_standard_messages_per_min`) and `human_ws_events_per_sec` **are** enforced on this path (`connection.rs:612-650`). The elevated and platform tiers are not referenced by this group. The doc row should be rewritten as "IP-connection limiting not wired; elevated/platform agent tiers unused".

---

#### 7. DoS surfaces

| # | Surface | Bound | Gap |
|---|---|---|---|
| D1 | Connection count | `conn_semaphore` = `max_connections`, default **10 000** (`config.rs:449-452`; `connection.rs:149`) | **no per-IP cap** — one host can take all 10 000 slots; each costs a 1000-slot mpsc, an 8-slot ctrl channel, a `HashMap`, 3 tasks. At 5 s per unauthenticated slot the sustained cost is 2 000 slot-acquisitions/s to hold the relay full |
| D2 | Pre-auth frame flood | none | admission bypassed (`connection.rs:604-609`); each frame costs a `serde_json::from_str` over up to 512 KiB and, for EVENT/REQ/COUNT, a `handler_semaphore` permit + task spawn |
| D3 | `CLOSE` flood (pre- or post-auth) | none | never metered (`connection.rs:599-602`), always answered (`close.rs:27`), awaited inline — self-throttling per socket but free work at scale |
| D4 | Malformed-JSON flood | none | `NOTICE` per frame, socket stays open (`connection.rs:490-496`) |
| D5 | Subscription memory | 1024 subs × 10 filters × unbounded per-filter arrays | `MAX_SUBSCRIPTIONS` (`req.rs:26`) and `MAX_FILTERS_PER_REQ` (`protocol.rs:14`) bound the counts, but **nothing bounds a filter's internal cardinality** — no cap on `ids`, `authors`, `kinds`, or generic-tag value counts. A REQ with 10 filters × 100 000 `authors` each is accepted, stored (`subscription.rs:81`, and again at `req.rs:238`), and re-evaluated linearly on **every** matching event (`filter_match_one`, `buzz-core/src/filter.rs:41-46`) |
| D6 | Index fan-out amplification | one index entry per (kind × filter) | `register_scoped` inserts one entry per kind in the union (`subscription.rs:104-112`). 10 filters × 1000 kinds = 10 000 index entries **per subscription**, ×1024 subs = ~10 M entries on one connection. Each carries a cloned `sub_id` up to 256 B → ~2.6 GB of `String` per connection, from a single client, entirely within the advertised caps |
| D7 | Removal cost | `Vec::retain` per index bucket | O(bucket) per kind on removal (`subscription.rs:407`, `:429`); D6's shape makes disconnect cleanup O(10 M) |
| D8 | Historical read cost | 2000 rows × 10 filters, concurrency 4 | `req.rs:885`, `:314`. No global per-connection or per-pubkey read budget beyond the 5 s WsEvents window |
| D9 | Search cost | 10 pages × 100 hits × 10 filters, plus one `get_events_by_ids` per page | `req.rs:421`, `:589`, `:594` — up to 100 FTS queries and 100 batch fetches per REQ |
| D10 | COUNT cost | 5001-row candidate scan per non-pushable filter × 10 filters | `req.rs:753`; `count.rs:173`, `:243` — up to ~50 000 rows scanned per COUNT, and **no `MessagesRead` scope required** (S4) |
| D11 | Conformance side-queries | 1 extra `communities_of_channels` per filter + per search page | `req.rs:355`, `:648` — issued even with the production `NoopTracer` (`state.rs:798`) |
| D12 | `observer_rate_limiter` growth | none | plain `DashMap` with no capacity and no eviction (`state.rs:589`, ctor `:773`; only `entry().or_insert()` at `event.rs:920-924`). One permanent entry per distinct `(community, agent pubkey)` observed. Any authenticated agent-owner pair can seed entries at 100/s |
| D13 | Spawned-task fan-out | `handler_semaphore` (1024) for message handlers | but `dispatch_persistent_event` (`event.rs:375`) and the workflow trigger (`event.rs:512`) spawn **unbounded** detached tasks per accepted event, outside any semaphore |
| D14 | Slow-client memory | 1000-message send buffer × 15 strikes | `config.rs:459-462`, `:470-473`. At 512 KiB per frame the theoretical per-connection buffer is ~500 MB; in practice bounded by event size (256 KiB ingest cap, `ingest.rs:1510`) |

D5+D6 together are the sharpest: they are reachable by a single authenticated member with no elevated permissions, entirely inside every advertised limit, and there is no test covering filter-cardinality limits.

---

#### 8. Additional findings

##### S4 (MEDIUM) — COUNT skips the read-scope check

`handle_req` refuses a session without `MessagesRead` (`req.rs:54-61`). `handle_count` has no equivalent branch (`count.rs:37-51`). A scoped session holding, say, only `ChannelsWrite` can therefore issue `COUNT` and learn exact event counts for every channel it is a member of — the count is an existence oracle that the read-scope gate exists to deny. The `#h`/token narrowing (`count.rs:95-97`) limits *which* channels, not *whether* counting is permitted at all.

##### S5 (MEDIUM) — COUNT leaks raw database error text

`count.rs:179`, `:209`, `:249`, `:278` all send `CLOSED "error: {e}"` with the `buzz_db` error rendered verbatim. `handle_event` explicitly sanitises the equivalent case with a comment ("don't leak DB/system details over WS", `event.rs:749-750`). A crafted filter that provokes a Postgres error returns schema/constraint/type detail to any authenticated member.

##### S6 (LOW) — `Failed` is permanently sticky, including for transient causes

`auth.rs:172`, `:206`, `:230`, `:287` all set `AuthState::Failed`, which is terminal (`auth.rs:58-66`). The ban seam goes to visible lengths to avoid *mislabelling* a transient DB error as a ban (`auth.rs:98-117`) — but it still writes `Failed` for the DB-error case (`auth.rs:161-165`), so a Postgres blip permanently kills that socket's ability to authenticate. The client must reconnect. Correct as a fail-closed choice; worth noting because the surrounding comment implies the distinction preserves more than it does.

##### S7 (LOW) — `remote_addr` is captured but never used for security

`ConnectionState.remote_addr` (`connection.rs:61`) is read only by two `info!` calls (`connection.rs:183`, `:289`). No per-IP accounting, no abuse correlation, no ban-by-IP. Combined with D1, the relay has the data for a per-IP cap and does not use it. Also note the address comes from `ConnectInfo` with a `0.0.0.0:0` fallback (`router.rs:238-243`) and **no** `X-Forwarded-For` handling, so behind a proxy every connection would appear to share one address — a per-IP cap would need proxy-header trust configured first.

##### S8 (LOW) — binary frames with invalid UTF-8 are silently dropped

`connection.rs:457-459` decodes binary frames and, on failure, falls through with no `else` — no NOTICE, no counter, no disconnect. A client sending malformed binary gets total silence, which is indistinguishable from a hung relay.

##### S9 (INFO) — the production `expect()` on the fan-out hot path

`event.rs:88` `.expect("fan-out frame cache covers every recipient subscription id")`. The invariant does hold at all three call sites, because the cache is built from the same iterator that drives the send (`event.rs:260-263`, `:320-323`, `:492`). But it is the only `expect()` outside `#[cfg(test)]` in these 8 files, it sits on the delivery path, and a panic there would poison a spawned dispatch task. `AGENTS.md` prohibits new `unwrap()`/`expect()` in production paths.

##### SEC-42 (NEW, added by `ab3af828`) — kind 30175 author-only-unless-shared read gate

A fourth per-event read class was introduced for `KIND_PERSONA` (30175), whose
plaintext payload carries `system_prompt` and `respond_to_allowlist`.

**The rule.** A 30175 event is visible to a foreign reader only if it carries a
tag of exactly two elements `["shared","true"]`. The author always sees their
own. All three conditions (persona kind, non-author reader, tag absent) must
hold for the event to be withheld — `crates/buzz-core/src/kind.rs:205-216`
(`is_unshared_persona_event`), kind test at `:192-194`, tag test
`persona_event_is_shared` at `:226`. The marker is a **tag**, not a content
field, so toggling share state does not change the content bytes that also
serve as the drift basis.

**Ingest side.** `validate_persona_envelope` (`handlers/ingest.rs:1034-1093`)
now rejects any `shared` tag that is not exactly two elements with the second
equal to `"true"` (`:1047-1052`), and rejects more than one `shared` tag
(`:1056-1060`). This is what lets every read path treat a stored event as
unambiguously shared or not, and is also what makes the SQL containment check
sound — a three-element `["shared","true","extra"]` tag would satisfy
`tags @> '[["shared","true"]]"'` and cannot be stored.

**Where it is enforced — both transports, unconditionally:**

| Surface | Site |
|---|---|
| WS REQ historical delivery | `handlers/req.rs:388` (via `event_visible_to_reader`, `:1222-1234`) |
| WS NIP-50 search REQ | `handlers/req.rs:705` |
| WS live fan-out (incl. Redis cross-node) | `handlers/event.rs:154-175`, reached from `:307` and `:241` |
| WS COUNT (fallback rows) | `handlers/count.rs:202`, `:271`; fast-path bypass `:110`, guards `:174`, `:243` |
| HTTP NIP-98 `POST /query` | `api/bridge.rs:1295` |
| HTTP NIP-98 `POST /count` | `api/bridge.rs:1512`, `:1569`; fast-path bypass `:1447-1448`, guards `:1477`, `:1543` |
| HTTP FTS `/search` bridge | `api/bridge.rs:1753` (defense-in-depth only — 30175 is absent from the FTS allowlist, `migrations/0008_fresh_install_search_allowlist.sql:16`) |
| SQL pre-filter before `ORDER … LIMIT` | `crates/buzz-db/src/event.rs:504-527`, set at `req.rs:296-297`, `count.rs:162-163`, `:230-231`, `bridge.rs:1229-1230`, `:1465-1466`, `:1530-1531` |

**No WS/HTTP asymmetry — unlike SEC-31.** The persona gate is a *result-level*
per-event check, so it is not conditioned on `channel_id.is_none()` the way the
filter-level p-gate / engram / author-only pre-gates are (`req.rs:182-204`).
Both transports call the identical helper, and both set the same SQL pushdown.
The SEC-31 shape cannot recur here.

**Residual gaps.**

1. There is **no filter-level pre-gate** for 30175 — no analogue of
   `author_only_filters_authorized` (`req.rs:1236+`). A foreign
   `{kinds:[30175]}` REQ is accepted and silently returns only the shared
   subset rather than being `CLOSED`. This is deliberate (the catalog query must
   work for foreign readers) but it means the gate is enforced *only* at the
   row level; a future read path added without calling
   `event_visible_to_reader` reopens the leak with no filter-level backstop.
2. Side-band existence oracles are an explicit **non-goal**: reaction, report,
   and deletion target resolution can still reveal that an unshared 30175 event
   exists (`docs/nips/NIP-AP.md:256`). Content stays withheld; existence does not.
   The stated mitigation is that an attacker must already hold the 64-hex event
   id, which no gated read path emits.
3. The SQL clause depends on `idx_events_tags_gin` (`migrations/0004_events_tags_gin.sql`)
   and on the ingest exact-shape rule holding for **already-stored** rows.
   Personas written before `ab3af828` were not re-validated, so a pre-existing
   malformed `shared` tag is still trusted by the containment check
   (`crates/buzz-db/src/event.rs:504-527`).

---

#### 9. Multi-tenant isolation posture

Isolation is layered and each layer is independently verified in code:

| Layer | Mechanism | Site |
|---|---|---|
| Bind before read | community resolved from `Host` pre-upgrade; unmapped → generic 404 | `router.rs:286-300` |
| Subscription indexes | every index key is prefixed with `CommunityId` | `subscription.rs:49-58` |
| Fan-out receiver fence | `community_for_conn(conn) == Some(event_community)` | `event.rs:126-133` |
| Redis topic keys | `EventTopicKey::from_context(ctx, topic)` | `buzz-pubsub/src/lib.rs:193`, `:216` |
| Cross-node community label | taken from the **parsed Redis channel**, not the event body | `event.rs:268` |
| Local-echo dedup key | `(community_id, event_id)` | `event.rs:298`; `state.rs:530-540` |
| Every cache key | `(CommunityId, …)` | `state.rs:740-787` |
| Cache invalidation | predicate-scoped to one community, with a documented over-invalidate fallback | `state.rs:882-899`, `:753-773` |
| Rate-limit keys | `buzz:{community}:ratelimit:{pubkey}:{suffix}` | `buzz-auth/src/rate_limit.rs:153-156` |
| Observer limiter + owner cache | `(CommunityId, …)` keys | `state.rs:589`, `:607` |
| Ban / disconnect | tenant-fenced: a ban in A never closes B's socket | `state.rs:261-282`, `:296-325`; test `:1662-1704` |
| Presence clear | only when no other connection for that pubkey **in that community** | `connection.rs:274-287` |
| Workflow trigger | community passed explicitly because `StoredEvent` doesn't carry it | `event.rs:528-537` |
| Audit chain | `community_id` from the `TenantContext`, not the event | `event.rs:582`; test `:1714-1858` |

The one deliberate non-fence: `ConnectionManager::drain_all` cancels every connection regardless of tenant (`state.rs:352-366`), correct for process shutdown and asserted as such at `state.rs:1829`.

Cross-tenant isolation is the best-covered property in this group — the `redteam` module (`event.rs:2346-2483`) plus `subscription.rs:1379-1441` and `event.rs:1576-1617` pin it from three directions.


## Module: buzz-relay — event ingest & side effects (`crates/buzz-relay/src/handlers`)
### Aspect: Security

This is the relay's only write door. Everything below is verified against source, not
against the doc comments.

---

### 1. Headline findings

| # | Finding | Severity | Evidence |
|---|---|---|---|
| **S-01** | **The per-kind scope gate is inert in production.** Both transports construct `IngestAuth` with `Scope::all_known()` — all 16 scopes. So `!auth.scopes().contains(&required)` at `ingest.rs:1551` can never be true, and the entire 81-row scope mapping in `required_scope_for_kind` (`ingest.rs:198-306`) is documentation only. `AdminUsers` on 9030–9033 and `AdminChannels` on 9000/9001/9008 provide **zero** enforcement; real authorization for those kinds lives in `relay_admin.rs` and `validate_admin_event`. | **High** (defence-in-depth absent, not directly exploitable) | `buzz-auth/src/lib.rs:133-141`; `api/bridge.rs:826-829`; `buzz-auth/src/scope.rs:68-87` |
| **S-02** | **All channel-scoped-token logic is dead.** `AuthContext.channel_ids` is hard-coded `None` on the WS path and absent from `IngestAuth::Http`, so `auth.channel_ids()` never returns `Some`. Four gates never fire: `ingest.rs:1538` (relay-admin vs channel token), `:1546` (28936 vs channel token), `:1747-1754` (channel token publishing a global event), and `check_token_channel_access` (`:525-532`). The code's own doc admits it (`ingest.rs:113-115`). | **Medium** (dead code masquerading as a control) | `buzz-auth/src/lib.rs:138`; `ingest.rs:117-126` |
| **S-03** | **Privileged writes are not audited.** `buzz-audit` declares 11 actions; production emits exactly 2: `EventCreated` (`handlers/event.rs:583`) and `MediaUploaded`. Every privileged mutation this group performs — role grants, member removal, channel deletion, visibility flips, relay-membership changes, identity archival — produces **only** a generic `EventCreated` row, and the 4 kinds that are *never stored* (9030–9033) produce **no audit row at all**. `EventDeleted`, `ChannelCreated`, `ChannelUpdated`, `ChannelDeleted`, `MemberAdded`, `MemberRemoved`, `AuthSuccess`, `AuthFailure`, `RateLimitExceeded` have zero producers workspace-wide. | **High** | `buzz-audit/src/action.rs:8-31`; grep for `AuditAction::*` outside `crates/buzz-audit/` |
| **S-04** | **Relay-admin commands (9030–9033) leave no durable trace whatsoever.** They are handled directly and never stored (`ingest.rs:1834-1844`), so no `events` row exists → `dispatch_persistent_event` is never called → no `EventCreated` audit entry. Membership and role changes at the *relay* level are the highest-privilege operation in the system and are the only privileged path with neither an event row nor an audit row. Compare 9035/9036, which deliberately *do* fall through to storage precisely so the audit reference resolves (`ingest.rs:1966-1971`). | **High** | `ingest.rs:1834-1844` vs `:1941-1945` |
| **S-05** | **Community moderation (9040–9044) has no `buzz-audit` entry either.** Also never stored (`ingest.rs:1605-1614`). The module header's "audit row" (`moderation_commands.rs:11-14`) refers to a `moderation_actions` domain table, not the hash-chained `buzz-audit` log — so ban/timeout/unban are outside the tamper-evident chain. | **Medium** | `ingest.rs:1605-1614`; `moderation_commands.rs:11-14`; no `buzz_audit` reference in that file |
| **S-06** | **Side-effect failure is invisible to the client and to the audit log.** A rejected `add_member`, a failed `update_channel`, a failed `remove_member` — all leave the command event committed, fanned out, and reported as `accepted: true`. Clients render the change; the DB never made it. | **High** | `ingest.rs:2486-2493` then `:2513` |
| **S-07** | **9000 on an *open* channel gets no relay-layer authorization.** `validate_admin_event`'s elevated-role check sits inside `if channel.visibility == "private"` (`side_effects.rs:296-338`). On an open channel, any authenticated principal may submit a 9000 with `role=owner`. The escalation is blocked only by `buzz_db::channel::add_member`'s granter check (`buzz-db/src/channel.rs:391-410`). Not currently exploitable, but the relay-layer gate reads as if it were the control and is not, and a DB refusal manifests as S-06 (event stored, no change, success reported). | **Medium** | `side_effects.rs:296-338` vs `buzz-db/src/channel.rs:391-410` |
| **S-08** | **`is_global_only_kind` does not neutralize a stray `h` tag on the read side.** `channel_id` is nulled at write (`ingest.rs:1735-1737`), but the signed tag remains and `filter.rs` treats explicit `h` tags as authoritative, so a global-only event can still match `#h` queries for a channel the author cannot read. Self-documented as a known limitation at `ingest.rs:369-377`. Affects all **44** global-only kinds. | **Medium** | `ingest.rs:369-377` |
| **S-09** | **4 `expect()` calls in the ingest hot path.** `ingest.rs:2024`, `:2026` (kind:44200 `p` tag / hex decode), `:2364`, `:2370` (`channel_id.expect("reaction path has channel")`). Each is justified by an earlier check in the same function, but all four are ~340–570 lines away from the invariant they rely on, and a panic in `ingest_event_inner` aborts the request path. AGENTS.md forbids new `unwrap()`/`expect()` in production paths. | **Medium** | `ingest.rs:2024`, `:2026`, `:2364`, `:2370` |
| **S-10** | **2 `unwrap()` calls in `validate_admin_event`.** `role_str.parse().unwrap()` (`side_effects.rs:311`) and `actor_member.unwrap()` (`side_effects.rs:314`) — both inside the 9000 private-channel branch. The parse is pre-validated at `:288-291` and the `Option` is checked at `:299-303`, so both are currently sound, but they sit in an authorization path. | **Medium** | `side_effects.rs:311`, `:314` |

---

### 2. Tenancy enforcement — full write-path audit

Every DB call in the group takes `tenant.community()`, resolved server-side from the
request host. **No write path derives the community from event content.** Verified
call-by-call:

| Write path | Community argument | Site |
|---|---|---|
| generic insert | `tenant.community()` | `ingest.rs:2420` |
| replaceable / NIP-33 replace | `tenant.community()` | `ingest.rs:2397`, `:2385` |
| reaction insert | `tenant.community()` | `ingest.rs:2324` |
| channel pre-create (9007) | `tenant.community()` | `ingest.rs:2129` |
| compensating soft-delete | `tenant.community()` | `ingest.rs:2434` |
| restriction lookup | `tenant.community()` | `ingest.rs:1642` |
| membership check | `tenant.community()` | `ingest.rs:501` (via `is_member_cached`) |
| relay-member removal | `tenant.community()` | `ingest.rs:1883` |
| command-event raw INSERT | `tenant.community().as_uuid()` bound as `$1` | `command_executor.rs:207` |
| command coordinate lock + SELECT + supersede | `tenant.community().as_uuid()` | `command_executor.rs:157-195` |
| all side-effect writes | `tenant.community()` | `side_effects.rs` — 60+ sites, no exceptions found |

Two places explicitly refuse to re-derive the community from client input, with the
attack spelled out in the comment:
- 30620: "`community_of_channel(channel_id)` is ambiguous when the same channel UUID exists
  in two communities and could mint the workflow under the wrong tenant"
  (`command_executor.rs:759-771`), followed by a community-scoped `get_channel` to fail
  closed;
- 46020: "a bare-id lookup could load B's workflow and then satisfy the membership check
  below against B's colliding channel, letting B trigger A's workflow"
  (`command_executor.rs:826-831`).

The conformance trace records `claimed_community_from_event(&event)` alongside every write
and auth verdict, but as a **claim only** — the verdict basis is always
`tenant.community()`, stated at `ingest.rs:1777-1782`.

`ThreadedChannelVisibility` (`ingest.rs:1776-1786`) bundles the resolved visibility with
the `(community_id, channel_id)` it was resolved under, so fan-out cannot apply it to the
wrong pair. `None` means "fan-out does its own fail-closed lookup", never "assume open" —
documented as fence 1 at `ingest.rs:1742-1749`.

---

### 3. Authorization matrix — who may write what

| Kind(s) | Required identity | Enforced where | Bypass paths |
|---|---|---|---|
| 0, 1, 3, 10000–10003, 10030, 10100, 30000, 30003, 30023, 30030, 30078, 30174–30177, 30300, 30315, 30350 | self (pubkey-match, `ingest.rs:1499`) | envelope | — |
| 1059 | **any authenticated principal**, pubkey-match deliberately waived | `ingest.rs:1498-1503` | by design (NIP-59 ephemeral keys) |
| 9, 40002, 40004–40008, 40100, 45001, 45003, 48100–48106 | channel member, or any principal if the channel is `open` | `check_channel_membership` `ingest.rs:493-523` | — |
| 40003 | target's effective author, or the NIP-OA owner of the agent that authored it; author path re-gated on membership | `validate_edit_ownership` `ingest.rs:763-842` | **skips** the generic membership gate (`ingest.rs:1798`) |
| 45002 | channel member + target is a 45001/45003 in the same channel | `ingest.rs:844-894` | — |
| 5 | target's effective author, or the NIP-OA owner of that author | `validate_standard_deletion_event` `side_effects.rs:179-238` | — |
| 7 | channel member of the *target's* channel (channel derived from target, `ingest.rs:1645`) | generic gate | — |
| 9000 | private channel: existing member; elevated role needs owner/admin. **Open channel: nothing** | `side_effects.rs:284-372` | S-07 |
| 9001 | self (not last owner), or owner/admin, or member + NIP-OA owner of target | `side_effects.rs:373-409` | non-members refused even for their own agent (`:365-367`) |
| 9002 | privileged tags: owner/admin, or owner of any active owner-role agent. topic/purpose: any member | `side_effects.rs:410-624` | **skips** the generic gate (`ingest.rs:1773`) |
| 9005 | target author (unless moderation metadata present), or owner/admin, or NIP-OA owner of the agent author | `side_effects.rs:508-624` | **skips** the generic gate (`ingest.rs:1774`) |
| 9007 | any authenticated principal — creator becomes owner | `ingest.rs:2029-2132` | **skips** the generic gate (no channel yet) |
| 9008 | owner, or owner of any active owner-role agent | `side_effects.rs:625-644` | **skips** the generic gate (`ingest.rs:1775`) |
| 9021 | any authenticated principal, `open` channels only | `ingest.rs:2160-2180` + `side_effects.rs:1846-1854` | **skips** the generic gate by design |
| 9022 | active member, not last owner | `side_effects.rs:645-663` | — |
| 9030–9033 | relay owner/admin — decided entirely inside `relay_admin.rs` | out of module | scope gate is inert (S-01) |
| 9035/9036 | self, admin role, or owner-via-live-kind:0 — decided in `identity_archive.rs`. Scope deliberately `UsersWrite` not `AdminUsers`, with the rationale at `ingest.rs:265-273` | out of module | — |
| 9040–9044 | moderator capability, decided in `moderation_commands.rs`; ban re-checked there (`:103-108`) | out of module | **exempt** from the ingest ban/timeout gate (`ingest.rs:1639`) |
| 28936 | self; relay owner refused | `ingest.rs:1846-1928` | — |
| 30617 | self; name must be unowned or self-owned; per-pubkey quota | `side_effects.rs:2464-2510` | — |
| 30618 | any principal with `ReposWrite` (i.e. everyone, S-01) — no repo-ownership check | `ingest.rs:294` + generic store | can publish a competing 30618 under its own `d` coordinate |
| 1617–1621, 1630–1633 | any authenticated principal; no repo relationship checked | generic store | — |
| 41010 | self | `command_executor.rs:310` | — |
| 41011/41012 | member of the DM; channel must be `channel_type == "dm"` | `command_executor.rs:518`, `:611` | — |
| 30620 | channel member; on update, must match existing owner **and** channel | `command_executor.rs:713-735` | — |
| 46020 | workflow **owner** only — "channel membership alone is insufficient: a member could otherwise invoke another user's webhook or message actions" (`command_executor.rs:838-840`) | `command_executor.rs:842-847` | — |
| 46030/46031 | `approver_spec` = `""`/`"any"` (anyone) or an exact pubkey; **fails closed** on role-based or unrecognised specs | `check_approver_spec` `command_executor.rs:961-984` | ⚠ `""`/`"any"` means *any authenticated principal in the community* may approve a workflow gate — including one they have no relationship to. That is the documented design, but it is a broad default. |
| 44200 | `event.pubkey` must be a registered agent and `p` must be its DB-registered owner | `ingest.rs:1981-2016` | — |
| 1984, 42000 | any authenticated principal | `ingest.rs:1540`, `:1561` | 1984 is deliberately allowed while timed out and in the post-ban window (`ingest.rs:1551-1559`) |

---

### 4. Tag injection / spoofing resistance

| Vector | Defence | Site |
|---|---|---|
| Forged author via `event.pubkey` | pubkey must equal the authenticated principal (except 1059) | `ingest.rs:1525` |
| Forged author via relay-signed attribution | `effective_message_author` only trusts `actor`/`p` tags **when `event.pubkey == relay_pubkey`** — a user-signed event can never claim a different author | `ingest.rs:729-761`; duplicate impl `side_effects.rs:2271-2298` |
| Stray `h` tag on a global kind | `channel_id` forced `None` (write side only — see S-08) | `ingest.rs:1735-1737` |
| Missing `h` on a channel kind | hard reject | `ingest.rs:1713-1717` |
| Cross-channel edit (h=A, e=event-in-B) | `validate_edit_ownership` compares the edit's channel to the target's | `ingest.rs:790-807` |
| Cross-channel delete (h=A, e=event-in-B) | checked twice — pre-storage `validate_admin_event` (`side_effects.rs:560-573`) and again in the side effect (`:1583-1598`), with the attack named in the comment | `side_effects.rs:1580-1583` |
| Cross-channel vote | `validate_forum_vote_target` compares channels | `ingest.rs:880-892` |
| Cross-channel reply | parent's `channel_id` must equal the reply's; a parent with no channel is refused | `ingest.rs:620-628` |
| Multi-target deletion ambiguity | `e_count + a_count` must be exactly 1; malformed `e` tags **count** | `ingest.rs:1972-1984`, `count_e_tags` `:719-726`, regression test `:3033-3043` |
| Malformed `e` + valid `a` routing to the coordinate soft-delete | routing keys on `has_e_tag` (presence, not validity) | `side_effects.rs:2296-2302` |
| Case-folding attack on `#p` visibility | 30174 and 44200 both require **lowercase** hex in `d`/`p`/`agent`, because an uppercase head wins NIP-33 replacement then becomes invisible to lowercase `#p` readers | `ingest.rs:1004-1019`, `:1178-1214`; regression test `:2612-2620` |
| Empty-`d` slot collapse | 30175 requires a non-empty slug (`ingest.rs:1027-1082`); 30300 requires exactly one non-empty `d` (`:1287-1296`). ⚠ 30176/30177 have no such guard | `ingest.rs:1027`, `:1287` |
| Garbage ciphertext superseding a valid head | NIP-44 v2 shape check (base64 alphabet, length ≡ 0 mod 4, decoded ≥99 B, `0x02` prefix) on 30174 and 44200 | `ingest.rs:1084-1149` |
| Channel identity smuggled in a 44200 envelope | an `h` tag is a hard reject | `ingest.rs:1170-1174` |
| `imeta` path traversal via `filename` | rejects `/`, `\`, control chars; explicitly "display-only and must never influence storage keys, which are content-addressed" | `imeta.rs:138-155` |
| `imeta` pointing at an external host | `is_local_media_url` requires `/media/` or the tenant's own base URL; rejects `?`, `#`, `%` | `imeta.rs:373-418`; test `:456-462` |
| `imeta` claiming a blob it does not own | sidecar must exist and MIME/size/duration must match | `imeta.rs:246-278` |
| `imeta` MIME deny-list bypass | `m` must equal the stored sidecar MIME, and a sidecar exists only for content that passed the upload validator — reasoned at `imeta.rs:71-76` | `imeta.rs:259-266` |
| `imeta` thumbnail masquerading as a primary | `url` and `image` reject `.thumb.`; `thumb` must end `.thumb.jpg` | `imeta.rs:62-66`, `:97-105`, `:123-128` |
| Back-dated event injection | ±900 s symmetric fence on all kinds; ±120 s on 28936 and 9040–9044 | `ingest.rs:1506-1513`, `:1830-1841` |
| Same-second replacement race on relay-signed addressables | `created_at` forced strictly past any existing event of the same coordinate, so the random-event-id NIP-16 tiebreaker cannot strand a stale snapshot | `side_effects.rs:905-928`, `:3092-3115` |
| Thread-depth exhaustion | depth ≤ 100 | `ingest.rs:645-649` |
| Content exhaustion | 256 KiB global, 60 KiB for 40008, 64 chars for kind:7 emoji | `ingest.rs:1489`, `:898`, `:2283` |
| SQL injection in the one raw-SQL module | all values bound as parameters; the advisory-lock key is a computed `i64` | `command_executor.rs:157-232` |
| Log injection / reflected event content | `Rejected` messages can embed event-controlled values (raw `visibility`, `channel_type`, tag pubkeys); the HTTP bridge truncates before logging but returns the full string to the client | `api/bridge.rs:842-851` |
| Revoked access to a live subscription | eviction on 9001/9022/9002-flip/archive, plus per-event fan-out membership re-check as the cluster-wide backstop | `side_effects.rs:39-140`, `:1428-1438` |
| Banned socket that missed the disconnect fan-out | durable re-check on every write — the write-path gate is explicitly "the durable backstop the fan-out's best-effort delivery relies on" | `ingest.rs:1589-1601` |

---

### 5. Privilege-escalation surfaces

| Surface | Assessment |
|---|---|
| **Role grant via 9000** | Private channels: gated (member + elevated-granter). Open channels: relay layer does nothing; only `buzz-db` blocks it (S-07). Self-add always bypasses the `channel_add_policy` check (`side_effects.rs:334-338`) — correct for joining, but it means the policy cannot stop an agent from adding *itself*. |
| **Role change via 9000 upsert** | `add_member`'s `ON CONFLICT … DO UPDATE SET role = EXCLUDED.role` (`buzz-db/src/channel.rs:412-421`) means 9000 doubles as a role-*change* command with no separate authorization. On an open channel, `effective_role` gating applies; on a private channel, `validate_admin_event`'s elevated check applies. No path grants owner without an existing owner/admin granter — but the two checks are in different layers and neither is named "change role". |
| **Last-owner orphaning** | Guarded in four places: `validate_admin_event` 9001 (`side_effects.rs:379-389`) and 9022 (`:647-661`), plus a second guard inside `handle_remove_user` (`:1273-1287`) and `handle_leave_request` (`:1919-1930`). Redundant but consistent. |
| **NIP-OA owner acting on private channels without membership** | Deliberate: 40003, 9002, 9005, 9008 skip the membership gate so an owning human can act on their agent's private channels ("OQ1 decision", `ingest.rs:1789-1796`). `actor_owns_any_owner_agent` (`side_effects.rs:240-257`) checks **all** active owners, not just the first, which is correct under co-ownership. Note the asymmetry: 9001 explicitly refuses this for non-members (`:365-367`) while 9002/9008 explicitly allow it (`:598-608`, `:632-640`) — three different answers to "can a non-member owner act?" across four kinds. |
| **Timeout does not cascade owner→agent** | Documented Phase-1 asymmetry (`ingest.rs:1602-1612`): a timed-out owner's agents can still write. Ban cascades structurally at the auth seam; timeout has no auth-seam presence. |
| **Moderation commands exempt from the ban/timeout gate** | Intentional so a timed-out admin can lift a timeout (`ingest.rs:1598-1604`). The handler independently re-checks the durable ban (`moderation_commands.rs:103-108`), so a banned actor cannot use the exemption. |
| **Relay-admin commands exempt from the ban/timeout gate** | `ingest.rs:1639` also excludes `is_relay_admin_kind`, with only "Relay-admin commands retain their separate authorization policy" as justification (`ingest.rs:1626-1627`). Unlike moderation commands, there is no visible statement that `relay_admin.rs` re-checks the community ban. This exemption is the weaker of the two. |
| **Approval `approver_spec = ""` / `"any"`** | Any authenticated principal may approve any workflow gate with an open spec (`command_executor.rs:965-968`). Role-based specs fail closed (`:979-983`), so the only two states are "one exact pubkey" or "anyone". |
| **kind:1059 pubkey-match waiver** | Any authenticated principal may publish a gift wrap signed by an arbitrary ephemeral key. Required by NIP-59; the accounting/metrics path deliberately classifies by the authenticated principal rather than the envelope signer (`ingest.rs:1370-1372`, test `:2934-2949`). |

---

### 6. Kinds that mutate state with no audit entry

| Kind(s) | State mutated | `events` row? | `EventCreated` audit? |
|---|---|---|---|
| 9030–9033 | `relay_members` (add / remove / role change), workspace profile | **no** | **no** |
| 9040–9044 | `community_bans`, `muted_until`, report resolution | **no** | **no** |
| 1984 | `moderation_reports` | **no** | **no** |
| 42000 | product-feedback table | **no** | **no** |
| 30350 | `push_leases` | **no** | **no** |
| 28936 | `relay_members` (self-removal) | **no** | **no** |
| 41010–41012, 30620, 46020, 46030, 46031 | DMs, workflows, runs, approvals | yes (raw INSERT, `command_executor.rs:196`) | ⚠ **no** — `handle_command` returns without ever calling `dispatch_persistent_event`, so command events are stored but never audited and never fanned out |
| 9000–9008, 9021, 9022 | membership, channel metadata, deletions | yes | yes, but only as generic `EventCreated` |
| 9035/9036 | `archived_identities` | yes (deliberately) | generic `EventCreated` |

**Six kind groups mutate durable state with no audit row of any kind**, and the seven
command kinds are stored-but-unaudited. Given the hash-chain design of `buzz-audit`
(`buzz-audit/src/hash.rs`), the chain's coverage of privileged actions is materially
narrower than its API suggests.

---

### 7. Controls that work well

- **Verification before everything.** Signature and event-id verification precede all
  routing except the four kind-classification rejects (`ingest.rs:1460-1478`), and the
  command-routing comment pins the invariant: routed "AFTER signature verification,
  timestamp check, pubkey/auth match, and scope validation — never before"
  (`ingest.rs:1558-1559`, echoed `command_executor.rs:7-12`).
- **Fail-closed restriction lookup.** A DB error refuses the write rather than allowing it
  (`ingest.rs:1633-1641`).
- **Durable ban re-check on every write** as a backstop for missed live disconnects
  (`ingest.rs:1589-1601`).
- **Community-scoped everything**, with two explicit refusals to re-derive tenancy from
  client input (`command_executor.rs:759-771`, `:826-831`).
- **Cross-channel target checks** on all four target-bearing kinds (5, 9005, 40003, 45002).
- **Sidecar-authoritative media validation** — the event's claims are checked against
  storage, never trusted (`imeta.rs:259-278`).
- **Lowercase-hex enforcement** on `#p`-queried tags, closing a real invisibility attack
  with a regression test for it (`ingest.rs:3139-3152`).
- **NIP-70 `["-"]` tag required on 28936** (`ingest.rs:1869-1878`) and set on every
  relay-signed NIP-43/NIP-IA event.
- **`no unsafe`** — zero occurrences across all 8 911 lines.

---

### 8. Counted metrics

| Metric | ingest.rs | side_effects.rs | command_executor.rs | imeta.rs |
|---|---|---|---|---|
| `unsafe` blocks | 0 | 0 | 0 | 0 |
| `unwrap()` outside `#[cfg(test)]` | 0 | **2** (`:311`, `:314`) | 0 | 0 |
| `expect()` outside `#[cfg(test)]` | **4** (`:1998`, `:2000`, `:2338`, `:2344`) | 0 | 0 | 0 |
| `TODO`/`FIXME`/`XXX`/`HACK` | 0 | 0 | 0 | 0 |
| `#[test]` | 95 | 5 | **0** | 11 |
| `#[tokio::test]` | 0 | 0 | 0 | 0 |
| `#[ignore]` | 0 | 0 | 0 | 0 |

Total: 111 unit tests, **0 ignored**, 0 async tests. `command_executor.rs` (1 327 lines,
7 command handlers, the only raw-SQL module, the only LWW implementation outside
`buzz-db`) has **no `#[cfg(test)]` module at all**.


## Module: buzz-relay — HTTP API surface (`crates/buzz-relay/src/api`)
### Aspect: Security

---

#### 1. Auth requirement of every endpoint in this module group

| # | Endpoint | Auth actually required | Rate limited? | Tenant bound from `Host`? | `file:line` |
|---|---|---|---|---|---|
| 1 | `POST /events` | NIP-98 **or** unsigned `X-Pubkey` (default) + relay membership | ✅ Redis | ✅ | `bridge.rs:636-642`, `:798`, `:760` |
| 2 | `POST /query` | same | ✅ Redis | ✅ | `bridge.rs:904-910`, `:960`, `:955` |
| 3 | `POST /count` | same | ✅ Redis | ✅ | `bridge.rs:1339-1345`, `:1391`, `:1386` |
| 4 | `GET /moderation/reports` | same + `ViewQueue` (owner/admin) | ❌ **none** | ✅ | `bridge.rs:2051`, `:2036-2052` |
| 5 | `GET /moderation/audit` | same | ❌ **none** | ✅ | `bridge.rs:2118-2124` |
| 6 | `GET /moderation/restricted` | same | ❌ **none** | ✅ | `bridge.rs:2136-2141` |
| 7 | `POST /hooks/{id}` | **webhook secret only** — no user identity | ❌ | ✅ | `bridge.rs:1823-1844` |
| 8 | `PUT /upload` | Blossom kind:24242 + `X-SHA-256` binding + membership | ✅ per-pod | ✅ | `media.rs:142-234` |
| 9 | `PUT /media/upload` | same | ✅ per-pod | ✅ | same |
| 10 | `GET /media/{sha256_ext}` | **none by default**; Blossom `get` + membership only when `require_media_get_auth` | ❌ | ✅ | `media.rs:489-514`; `config.rs:682-689` |
| 11 | `HEAD /media/{sha256_ext}` | same as GET | ❌ | ✅ | `media.rs:798-806` |
| 12 | `POST /api/invites` | NIP-98 **strict** + payload tag + role owner/admin | ❌ | ✅ | `invites.rs:210-218`, `:233-247` |
| 13 | `POST /api/invites/claim` | NIP-98 **strict** + payload tag; membership gate intentionally skipped | ✅ per-pod moka | ✅ | `invites.rs:210-218`, `:293-298` |
| 14 | `POST /api/invites/accept-policy` | **NONE** | ❌ | ❌ (config-only) | `invites.rs:162-190` |
| 15 | `GET /api/join-policy` | **NONE** | ❌ | ❌ | `invites.rs:75-90` |
| 16 | `GET /api/join-policy/terms` | **NONE** | ❌ | ❌ | `invites.rs:95-102` |
| 17 | `GET /api/join-policy/privacy` | **NONE** | ❌ | ❌ | `invites.rs:104-110` |
| 18 | `POST /operator/communities` | NIP-98 strict + payload tag + operator allowlist | ❌ | ❌ (origin-based) | `operator.rs:66-98`, `:154-160` |
| 19 | `GET /operator/communities` | NIP-98 strict + operator allowlist | ❌ | ❌ | `operator.rs:307-315` |
| 20 | `POST /operator/communities/archive` | NIP-98 strict + payload + allowlist | ❌ | ❌ | `operator.rs:208-209` |
| 21 | `POST /operator/communities/unarchive` | same | ❌ | ❌ | `operator.rs:270-271` |
| 22 | `GET /operator/communities/availability` | NIP-98 strict + allowlist | ❌ | ❌ | `operator.rs:473-480` |
| 23 | `POST /operator/communities/transfer` | NIP-98 strict + payload + allowlist | ❌ | ❌ | `operator.rs:355-364` |
| 24 | `GET /api/admin/v1/reports` | **`Host` header equality only** (+ `Origin` iff present) | ❌ | ❌ (deployment-wide) | `admin/auth.rs:16-33`; `admin/mod.rs:100` |
| 25 | `GET /api/admin/v1/reports/{id}` | same | ❌ | ❌ | `admin/mod.rs:130` |
| 26 | `GET /api/admin/v1/feedback` | same | ❌ | ❌ | `admin/mod.rs:155` |
| 27 | `GET /api/admin/v1/feedback/{id}` | same | ❌ | ❌ | `admin/mod.rs:182` |
| 28 | `GET /api/admin/v1/feedback/{id}/attachments/{sha256}` | same + `imeta` provenance binding | ❌ | derived from the **row**, then cross-checked | `admin/mod.rs:196-240` |
| 29 | `GET /.well-known/nostr.json` | **NONE** | ❌ | ✅ (fails open to empty) | `nip05.rs:26-70` |
| 30 | `POST /_mesh/demo/echo` | **NONE** (double flag gate only) | ❌ | ❌ — **community from the body** | `mesh_demo.rs:58-77` |

##### Fully unauthenticated endpoints (explicit list)

1. `POST /api/invites/accept-policy` — `invites.rs:162`
2. `GET /api/join-policy` — `invites.rs:75`
3. `GET /api/join-policy/terms` — `invites.rs:95`
4. `GET /api/join-policy/privacy` — `invites.rs:104`
5. `GET /.well-known/nostr.json` — `nip05.rs:26`
6. `GET /media/{sha256_ext}` (default config) — `media.rs:604`
7. `HEAD /media/{sha256_ext}` (default config) — `media.rs:798`
8. `POST /_mesh/demo/echo` (when both flags on) — `mesh_demo.rs:58`
9. `POST /hooks/{id}` — no *user* auth; a shared bearer secret only — `bridge.rs:1780`
10. All 5 `GET /api/admin/v1/*` — no credential of any kind — `admin/auth.rs:16`

---

#### 2. Highest-severity findings

##### SEC-01 (CRITICAL, config-dependent) — `X-Pubkey` header grants unsigned impersonation by default

`verify_bridge_auth_with_options` accepts a bare `X-Pubkey: <hex>` header, with **no signature**,
whenever `require_auth_token == false` (`bridge.rs:118-127`). `BUZZ_REQUIRE_AUTH_TOKEN` defaults to
**false** (`config.rs:475-477`, asserted by `config.rs:954-956`'s sibling default tests). Four
routes pass that flag straight through: `/events` (`:640`), `/query` (`:908`), `/count` (`:1341`),
and all three `/moderation/*` reads (`:2033`).

Impact on a default-configured relay: any network-reachable caller can publish events as an
arbitrary pubkey, read any accessible channel as that pubkey, and — by naming a community
owner/admin's pubkey — read the entire moderation queue, audit log (including `private_reason`,
`bridge.rs:2165`), and ban list. Replay protection is additionally **skipped** for this path because
the zero event-id sentinel short-circuits the guard (`bridge.rs:122-125` → `:150-153`).

The startup warning understates it: "REST API requests bypass token auth" (`config.rs:588-593`) —
the actual effect is full pubkey impersonation. `.env.example` does not mention the variable at all.

##### SEC-02 (HIGH) — the admin API has no application-layer credential

`authorize` (`admin/auth.rs:16-33`) checks only:
(a) `config.admin.is_some()`, (b) `Host` header string-equals `config.admin.host`,
(c) **if** an `Origin` header is present, that it matches. A request with **no** `Origin` — i.e. any
non-browser client such as `curl -H 'Host: admin.example.com'` — passes (b) and skips (c) entirely.

The routes behind it are deployment-wide: `admin_list_reports`, `admin_get_report`,
`admin_list_feedback`, `admin_get_feedback` take no `CommunityId` (`admin/mod.rs:101-111`, `:132`,
`:155`, `:184`), and `feedback_attachment` streams raw blob bytes (`admin/mod.rs:226`).

This is a documented deliberate posture — "The human trust boundary remains the private admin
ingress" (`docs/admin/README.md:64-70`) — but it means a single ingress/`NetworkPolicy`
misconfiguration, or any in-cluster workload that can reach the relay port, yields full cross-tenant
read of moderation reports and user-submitted feedback plus attachments. There is no defence in
depth, no audit of who read what beyond a `tracing::info!` with no principal
(`admin/mod.rs:228-233`), and `docs/admin/README.md:66-70` explicitly disclaims per-operator
attribution.

##### SEC-03 (HIGH) — `POST /_mesh/demo/echo` takes the tenant from the request body

`DemoEchoRequest.community_id` is client-supplied (`mesh_demo.rs:50-51`) and is converted straight
into a `CommunityId` used to acquire a Redis fenced session lease (`mesh_demo.rs:99-101`). This is
the only route in the module that does not derive the tenant from `Host`, so it bypasses the row-zero
boundary every other door enforces (BR-API-01). With no authentication either, an unauthenticated
caller on a demo-enabled deployment can take or contend session leases in **any** community and
inject arbitrary payloads onto the mesh reliable stream.

Mitigation is the double flag gate (`BUZZ_MESH=on` **and** `BUZZ_MESH_DEMO_ECHO=on`, both default
off, `config.rs:509-518`). The file itself says "Not a product flow… the route stays demo-gated
until it is deleted" (`mesh_demo.rs:21-23`).

Secondary: the 404-indistinguishability property claimed at `mesh_demo.rs:71-73` does not hold —
`Json<DemoEchoRequest>` is a `FromRequest` extractor evaluated **before** the handler body, so a
malformed body returns 400/422 on a mesh-off relay while a genuinely absent route would return
404/405. The route's existence is therefore probeable.

##### SEC-04 (MEDIUM-HIGH) — `POST /api/invites/accept-policy` is an unauthenticated HMAC-minting oracle

`accept_policy` (`invites.rs:162-190`) has no NIP-98, no pubkey binding, and no rate limit. It
accepts an arbitrary `code` string, hashes it, and returns a relay-HMAC'd receipt
(`invite_token.rs:346-359`).

Consequences:
- The consent gate proves "someone POSTed the right version string", not "this pubkey accepted the
  policy". Any party holding an invite link can obtain a receipt on behalf of the eventual joiner,
  so the acceptance recorded at `invites.rs:324-338` is not attributable.
- It is an unbounded, unauthenticated MAC oracle over attacker-chosen input signed with
  `derive_invite_key(relay_keypair)` — the **same** key that signs invite codes
  (`invite_token.rs:112-117`, used at `invites.rs:186`, `:260`, `:301`).
- No domain-separation label distinguishes the two payload types inside the signed bytes.
  Cross-type confusion is prevented only by serde's missing-field strictness (`InvitePayload` needs
  `r` + `n`, `invite_token.rs:64-74`; `PolicyAcceptancePayload` needs `v`, `:335-343`). Adding an
  optional field to either struct would open it.

##### SEC-05 (MEDIUM) — `/moderation/*` reads have no rate limit

`authorize_moderation_read` (`bridge.rs:2008-2054`) performs bind → NIP-98 → replay → capability
check but never calls `enforce_http_admission`. These are the only NIP-98 bridge routes without a
limiter, and they return up to 500 rows each including `private_reason`. Combined with SEC-01 on a
default-config relay, this is an unmetered bulk export of the moderation record.

##### SEC-06 (MEDIUM) — unbounded per-pod media upload limiter map

`media_upload_rate_limiter` is a plain `DashMap<(CommunityId,[u8;32]), (u32, Instant)>`
(`state.rs:38-39`, `:592`, `:774`) with **no capacity bound, no TTL, and no sweep task** — grep for
`media_upload_rate_limiter` returns exactly three hits: the field, its `DashMap::new()`, and
`media.rs:97`.

`invites.rs:36-43` argues the exact reason this is wrong for the sibling limiter: "a pre-membership
caller can cheaply create fresh Nostr keypairs; retaining one immortal entry per key would make the
limiter itself an unbounded-memory denial-of-service vector." On an open relay
(`require_relay_membership = false`, the default) media upload authorization is a valid Blossom
signature from *any* key (`media.rs:196-206`), so the same argument applies verbatim — but the
media limiter got a bare DashMap while the invite limiter got moka with capacity 10 000 + 60 s TTL
(`state.rs:775-780`).

##### SEC-07 (MEDIUM) — media reads are unauthenticated by default and `verify_blossom_get_auth` has one conditional caller

`require_media_get_auth` defaults to **false** (`config.rs:682-689`, asserted by
`config.rs:973-976`). `authenticate_media_read` returns early with only the tenant when the flag is
off (`media.rs:491-494`), so `GET`/`HEAD /media/{sha256_ext}` serve blob bytes to anyone who knows a
SHA-256. `verify_blossom_get_auth` is defined in `buzz-media` (`auth.rs:207`) and its **only** call
site repo-wide is `media.rs:502` — behind that flag.

Mitigations that are real: unguessable 256-bit keys, sidecar-per-tenant lookup so a blob is only
readable under a community that has its own sidecar (`media.rs:632-660`), `attachment` disposition
for non-inline types plus `nosniff` and `CSP: default-src 'none'` (`media.rs:662-687`), and
`Cache-Control: private` once auth is on (`media.rs:517-523`).

Related: the admin attachment route calls `serve_blob_for_tenant` **directly**
(`admin/mod.rs:226`), bypassing `authenticate_media_read`, so `BUZZ_REQUIRE_MEDIA_GET_AUTH` never
applies there.

##### SEC-08 (LOW-MEDIUM) — `/hooks/{id}` is a workflow-state oracle

An unauthenticated caller can distinguish four states by status/message
(`bridge.rs:1787-1826`): unknown workflow in this community → 404 `workflow not found`; exists but
not webhook-triggered → 400 `workflow does not have a webhook trigger`; exists, webhook, no secret
configured → 401 with the descriptive `"webhook secret required but not configured — re-save the
workflow to generate one"`; exists, webhook, wrong secret → 401 `authentication failed`. UUID
entropy makes enumeration impractical, but a leaked-then-rotated id still leaks its trigger type and
secret-configuration state.

Also: the `?secret=` query fallback (`bridge.rs:1770-1775`, `:1809`) puts a bearer credential in
access logs and `Referer` headers; the doc acknowledges the header is preferred but the fallback is
unconditional and never warns.

##### SEC-09 (LOW) — reflected request content in 4xx bodies

`bridge.rs:735-742` explicitly reasons that `serde_json`'s `Display` "embeds the offending input
verbatim… would otherwise reflect attacker-controlled text into a log line at full size" and fixes
the **log** path — but the same string is still returned to the client at `bridge.rs:745`
(`"invalid event JSON: {e}"`), `:970` (`"invalid filters: {e}"`), `:1381`, and in
`invites.rs:172-176`, `:252-257`, `:302`, `operator.rs:163-167` etc. Similarly, `submit_event`
truncates `IngestError::Rejected` to 256 bytes for logs (`bridge.rs:850`) but returns the
**untruncated** message in the body (`:855`). Self-reflection only (the caller sees its own input),
so impact is bounded — but the asymmetry means the log-size defence has no response-side twin.

`internal_error` (`api/mod.rs:23-26`) is the correct pattern and is used consistently for DB and
serialization failures.

##### SEC-10 (LOW) — operator actions are not attributed

`transfer_community` binds the authenticated operator to `let _pubkey` and discards it
(`operator.rs:355`); `archive_community` (`:209`) and `unarchive_community` (`:271`) call
`authorize_operator_request(...).await?;` without binding at all. The resulting `tracing::info!`
lines carry community + host but no actor (`operator.rs:281-282`, `:299`, `:429`), and no audit-log
entry is written. `provision_community` is the only one that threads the operator pubkey through
(`operator.rs:157`, `:179`). Community archival and ownership transfer are the two highest-impact
operations on the surface.

---

#### 3. NIP-98 verification and replay protection

| Property | Status | `file:line` |
|---|---|---|
| Signature/URL/method/payload verification | delegated to `buzz_auth::verify_nip98_event` | `bridge.rs:110-111` |
| `u`-tag host bound to the **resolved tenant**, not `config.relay_url` | ✅ | `bridge.rs:195-206`; tests `:2417-2449`, `:2477-2504`, `:2636-2654` |
| Query string included verbatim in the expected URL for GETs | ✅ | `bridge.rs:2027-2031`; `operator.rs:73-77`; tests `:2529-2630` |
| `payload` tag required on body-bearing requests | ✅ for invites (`invites.rs:217`) and operator (`operator.rs:84`); ❌ **not** for `/events`, `/query`, `/count`, which call `verify_bridge_auth` (`require_payload = false`, `bridge.rs:69`) | `bridge.rs:97-108`, `:62-70` |
| Replay guard consulted on every NIP-98 request | ✅ all 8 NIP-98 routes: `bridge.rs:766` (`/events`), `:956` (`/query`), `:1387` (`/count`), `:2034` (moderation ×3), `invites.rs:219` (mint + claim), `operator.rs:85` (all 6 operator) | — |
| Guard error fails **CLOSED** | ✅ 401 `"NIP-98: replay check unavailable"` | `bridge.rs:167-176`; operator sibling `operator.rs:126-137`; infra-free test `bridge.rs:2340-2377` |
| Cross-pod correctness | ✅ shared Redis `SET NX EX`, community-scoped key | `state.rs:710-711`; tests `bridge.rs:2262-2292` (two guard instances, one pool) |
| Same-pod replay still rejected after the moka→Redis migration | ✅ | test `bridge.rs:2294-2317` |
| Community scoping of the seen-set | ✅ same event id in a different community succeeds | test `bridge.rs:2290-2292` |
| Operator scope isolation | ✅ dedicated `"operator-management"` scope, not a community | `operator.rs:55`, `:108-122` |
| **Gap** | replay is skipped for the zero event-id sentinel produced by the `X-Pubkey` path | `bridge.rs:122-125`, `:150-153` |

**Note on the payload-tag asymmetry:** `/events` does not require a `payload` tag, but
`verify_nip98_event` receives `Some(&body)` (`bridge.rs:639`), so if a payload tag *is* present it is
verified against the body. A NIP-98 event without a payload tag is accepted and can therefore be
replayed against a *different* body — except the replay guard blocks the second use of the same
event id, and the seen-set is community-scoped. Net effect: single-use, so not exploitable, but the
safety depends entirely on the replay guard rather than on payload binding. The invite and operator
routes chose the stronger posture (`require_payload = true`).

---

#### 4. Rate limiting posture

| Surface | Backend | Scope | Fail mode | `file:line` |
|---|---|---|---|---|
| `/events`, `/query`, `/count` | **Redis** `RedisRateLimiter` | `(tenant, pubkey)`, 60 s window, `human_api_calls_per_min` (default 300) | **closed** (503) | `bridge.rs:24-56`; `admission.rs:24-33`; `state.rs:712` |
| Media upload | in-process DashMap | `(community, pubkey)`, 60 s, `media_uploads_per_minute` (30) | n/a — pure local | `media.rs:88-111` |
| Media upload concurrency | in-process `Semaphore` + DashMap counter | global 8, per-pubkey 2 | reject 429 | `media.rs:113-136` |
| Invite claim | in-process moka (cap 10 000, TTL 60 s) | `(community, pubkey)`, 10/window | n/a | `invites.rs:374-390`; `state.rs:775-780` |
| Everything else in the module | **none** | — | — | see §1 |

**Documentation delta (confirmed):** `ARCHITECTURE.md:823` (§9 limitation #2) states "No rate
limiting implementation… none are enforced"; `ARCHITECTURE.md:390` states "No Redis-backed rate
limiter exists anywhere in the codebase — rate limiting is not currently enforced"; and
`ARCHITECTURE.md:459` says `buzz-pubsub` "Does NOT implement the rate limiter". All three are
**false**: `RedisRateLimiter` lives in `buzz_pubsub::rate_limiter` (`state.rs:26`), is constructed at
`state.rs:712`, and is consulted on all three bridge POSTs (`bridge.rs:760`, `:955`, `:1386`).

---

#### 5. Admin token handling / constant-time comparison

There is **no admin token** — see SEC-02. The only constant-time comparison in the module group is
the webhook secret:

```rust
if provided.len() != stored.len() { return false; }      // webhook_secret.rs:82-84
let mut result = 0u8;
for (a, b) in provided.bytes().zip(stored.bytes()) { result |= a ^ b; }  // :86-88
result == 0
```

Length is revealed non-constant-time, justified at `webhook_secret.rs:74-79` on the grounds that the
generator always emits a 36-char UUID. Sound.

The operator allowlist uses plain `==` string comparison (`operator.rs:91-95`) — acceptable, since
the compared value is a public pubkey, not a secret. The admin host comparison is likewise plain
equality on a non-secret (`admin/auth.rs:11-14`).

---

#### 6. Invite-token forgery resistance

| Property | Assessment | `file:line` |
|---|---|---|
| MAC | HMAC-SHA256 over the exact serialized payload bytes | `invite_token.rs:119-123` |
| Key | `sha256(relay_secret_key ‖ b"buzz-invite-v1")` — domain-separated from other uses of the keypair | `invite_token.rs:112-117`, `:58` |
| Verification order | MAC **before** trusting any payload claim | `invite_token.rs:171-176`; doc `:151-155` |
| Comparison | `mac.verify_slice` — constant-time by the `hmac` crate | `invite_token.rs:174` |
| Tamper resistance | re-encoding the payload with `r:"owner"` and the original MAC ⇒ `BadSignature` | test `invite_token.rs:238-262` |
| Wrong key | ⇒ `BadSignature` | test `invite_token.rs:264-272` |
| Role ceiling re-checked post-MAC | a correctly-signed `r:"admin"` payload is still rejected | `invite_token.rs:189-191`; test `:311-331` |
| Community binding | code minted for A rejected on B | `invite_token.rs:186-188`; endpoint test `invites.rs:1004-1046` |
| Input bounds before parsing | `MAX_CODE_LEN = 1024` (receipts 2048) | `invite_token.rs:57`, `:157-159`, `:369-371` |
| Oracle hardening at the HTTP layer | only `Expired` is distinguishable; all else → 403 `invite_invalid` | `invites.rs:303-312` |
| Nonce | 16 random bytes so identically-parameterised invites differ | `invite_token.rs:133` |
| **Non-properties (documented)** | multi-use until expiry (no used-bit); revocation only by rotating the relay keypair | `invite_token.rs:32-34`, `:43-46` |
| **Residual risk** | shared HMAC key with the policy-receipt payload and no in-payload purpose label (see SEC-04) | `invite_token.rs:346-359` |

Forgery resistance is sound. The weak links are operational, not cryptographic: a leaked link is
valid for up to 30 days for unlimited redemptions, and revocation is deployment-wide.

---

#### 7. Tenancy isolation per endpoint

| Endpoint group | Isolation mechanism | Verdict |
|---|---|---|
| Bridge (`/events`, `/query`, `/count`) | `bind_community(Host)` → every DB call takes `tenant.community()`; NIP-98 `u`-host must equal `tenant.host()`; accessible-channel set resolved per tenant | ✅ strong |
| Moderation reads | same bind; queries community-scoped; `ViewQueue` evaluated against `relay_members` in that community | ✅ |
| Webhook | tenant from `Host`, **not** from the workflow row — same UUID in another community is a 404 | ✅ documented at `bridge.rs:1773-1782` |
| Media upload | tenant bound before Blossom verification so the `server` tag validates against the bound host | ✅ `media.rs:145-156` |
| Media read | sidecar lookup is tenant-scoped; blobs are shared content-addressed objects, so a blob is only readable under a community holding its own sidecar | ✅ `media.rs:632-660` |
| Invites | community from `Host`; code carries the community UUID and is re-checked | ✅ double-bound |
| Join policy | **deployment-global config**, not per-community — all tenants share one policy and one version | ⚠️ by design (`config.rs:794-811`) |
| Operator | deliberately **not** tenant-bound; authority is `relay_operator_api_origin` + allowlist; `list_communities_owned_by` and `lookup_community_by_host_for_management` are cross-community by design | ⚠️ intended control plane (`operator.rs:57-60`) |
| Admin | deliberately deployment-wide; `communityId` is an optional filter, and 3 of 4 DB calls take no community at all | ⚠️ intended, but the boundary is the ingress (SEC-02) |
| Admin attachment | tenant derived from the row's own `community_host`, then asserted to still resolve to `feedback.community_id`, plus `imeta` `x`+`url` binding | ✅ the strongest provenance chain in the module (`admin/mod.rs:206-240`) |
| NIP-05 | bound from `Host`; handle domain must equal the tenant host | ✅ `nip05.rs:36-58`, `:79-102` |
| Mesh demo | **community from the request body** | ❌ SEC-03 |

Per-tenant limiter isolation is explicitly tested: media rate + concurrency
(`media.rs:1120-1161`), invite claim (`invites.rs:481-503`), replay seen-set
(`bridge.rs:2290-2292`).

---

#### 8. SSRF, path traversal, IDOR, enumeration

##### SSRF
No outbound HTTP is issued from any of the 13 files. The only URL parsing is
`attachment_url_matches` (`admin/mod.rs:260-286`), which **compares** a stored `imeta` URL and never
fetches it — and even then it requires scheme ∈ {http, https}, normalized-host equality with the
feedback row's community host, exact hash match, a safe extension, and **no query and no fragment**
(`admin/mod.rs:281-286`; test `:483-504` covers `?token=leak`, `.thumb.jpg` substitution, an extra
path segment, and `evil.example`).

##### Path traversal
`validate_media_path` (`media.rs:547-583`) is the gate: 1–3 dot segments, first segment exactly 64
lowercase hex, `ext` matching `is_safe_ext` (1–8 chars `[a-z0-9]`, `media.rs:533-535`), 3-segment
form allowed only as `.thumb.jpg`. Tests cover `../etc/passwd`, `../{hash}.jpg`, uppercase hashes,
`_uploads/…`, `_meta/…`, `{hash}.png.metadata`, `{hash}.tar.gz`, and overlong extensions
(`media.rs:1163-1272`). Sidecar-derived extensions are re-validated before key construction
(`media.rs:875-879`) — "never trust storage as authoritative for path construction". Axum's single
path segment cannot contain `/` anyway, and the validator is asserted to hold independently of
routing (`media.rs:1250-1253`).

##### IDOR
| Vector | Control | `file:line` |
|---|---|---|
| Read another user's private snapshot via kindless `ids` | `reader_authorized_for_event` at 4 result-level checkpoints | `bridge.rs:1080`, `:1170`, `:1283`, `:1601`; test `:3125-3160` |
| Read another owner's engram via NIP-50 text match | `search_hit_accepted` re-applies `#p`/authors/channel | `bridge.rs:1590-1607`; tests `:2842-2896` |
| Read a channel you are not in | accessible-channel intersection at query build + result time | `bridge.rs:1000-1005`, `:583-588` |
| Count-based existence leak on result-gated kinds | forced per-event fallback unless `#p=[self]` is pushed down | `bridge.rs:1436-1443` |
| Claim an invite as a different pubkey | NIP-98 proves control of the claiming key; role fixed to `member` | `invites.rs:210-218`; `invite_token.rs:189-191` |
| Archive/unarchive someone else's community | `*_owned_by(host, owner, …)` requires the asserted owner to actually own it; wrong owner and unknown host both 404 | `operator.rs:234-240`, `:288-293` |
| Transfer from a stale owner | CAS on `expected_owner_pubkey` ⇒ 409 | `operator.rs:392-431` |
| Read an attachment not belonging to the named feedback | `imeta` `x` + `url` binding + community cross-check | `admin/mod.rs:216-240` |
| **Weak spot** | `list_owned_communities` returns any pubkey's communities to any allowlisted operator | `operator.rs:305-331` |

##### Enumeration surfaces
| Surface | Status | `file:line` |
|---|---|---|
| Which communities exist on a deployment | closed — unmapped host always yields the same generic 404, host never echoed | `bridge.rs:626-633`; `router.rs:288-296` |
| NIP-05 handle existence | **open by design** — but a miss, an unmapped host, and a missing `name` are indistinguishable (all `{names:{},relays:{}}` with 200), so it leaks handle existence only, and that is the point of NIP-05 | `nip05.rs:41-62` |
| Blob existence via `HEAD /media/{hash}` | open by default, but requires a 256-bit hash | `media.rs:798` |
| Workflow existence / trigger type / secret-configured state | **open** — four distinguishable outcomes on `/hooks/{id}` (SEC-08) | `bridge.rs:1787-1826` |
| Community host availability + community UUID | operator-only | `operator.rs:480-491` |
| Mesh demo route existence | leaks via the `Json` extractor rejection (SEC-03) | `mesh_demo.rs:60-70` |
| Admin route existence | 404 when `admin` unconfigured vs 403 on a non-admin host — distinguishable, but both non-informative | `admin/auth.rs:17-24` |
| Media auth failures | all 15 variants collapse to one 401 `"authentication failed"` explicitly to avoid an oracle | `buzz-media/src/error.rs:118-127` |
| Invite validity | only `Expired` distinguishable; everything else `invite_invalid` | `invites.rs:303-312` |

---

#### 9. Security headers and CORS

| Control | Coverage | `file:line` |
|---|---|---|
| `security_headers` middleware (`no-store`, `nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: no-referrer`, `CSP: default-src 'none'; frame-ancestors 'none'`) | **only** the `/api/admin/v1` sub-router, and only when `BUZZ_ADMIN_HOST` is set | `admin/mod.rs:38`, `:43-61`; `router.rs:56-59` |
| `CSP: default-src 'none'` + `nosniff` on blob responses | media GET (200 and 206) only — **not** on HEAD | `media.rs:684-686`, `:741-743` vs `:841-851` |
| `Content-Disposition: attachment` for non-inline types | media GET | `media.rs:662-668` |
| `Access-Control-Allow-Origin: *` | unconditionally on NIP-05 | `nip05.rs:65-69` |
| CORS | `CorsLayer::permissive()` when `BUZZ_CORS_ORIGINS` is empty — the **default**; an explicit list that fails to parse falls back to `CorsLayer::new()` (deny) rather than permissive, with an error log | `router.rs:373-397` |
| Body limits | 1 MiB bridge/api, `max(image, video)` = 500 MB media, 1 KiB admin, 1 MiB git policy | `router.rs:130`, `:33-46`; `admin/mod.rs:39` |

**Gap:** every non-admin HTML-bearing response — notably the operator-authored policy pages at
`invites.rs:95-124`, which return `text/html` on an unauthenticated route — receives **no**
`X-Frame-Options`, `Referrer-Policy`, or CSP. The pages do escape raw HTML in the operator Markdown
(`invites.rs:126-155`, test `:1254-1271`), so stored XSS via Markdown is closed, but there is no
header-level defence in depth and the pages are framable.

---

#### 10. Positive security properties worth preserving

1. **Fail-closed replay guard** with an infrastructure-free regression test (`bridge.rs:2321-2377`).
2. **Fail-closed rate limiter** — `AdmissionError::Unavailable` ⇒ 503, not a bypass
   (`bridge.rs:47-55`).
3. **Fail-closed tenant binding** with a uniform, non-echoing 404 on every door.
4. **Auth before body buffering** on uploads via `FromRequestParts` (`media.rs:29-38`).
5. **Sidecar-before-I/O** ordering and storage-is-not-authoritative discipline
   (`media.rs:629-660`).
6. **Read gates applied before the search branch** so NIP-50 cannot be used to bypass `#p`/author
   rules (`bridge.rs:981-998` then `:1006`).
7. **Result-level read auth** as defence in depth on all four `/query` result paths, with an
   explicit "defense-in-depth" comment at each (`bridge.rs:1076-1079`, `:1166-1169`).
8. **COUNT never silently truncates** — an over-large fallback candidate set is a 400, not a wrong
   number (`bridge.rs:1489-1497`).
9. **Attacker-controlled text truncated at a UTF-8 boundary before logging**, with a 1 MiB-input
   regression test (`bridge.rs:595-611`, `:3208-3241`).
10. **Admin errors are `&'static str` by construction**, so no dynamic text can leak through that
    surface (`admin/error.rs:12-17`).
11. **Cross-host NIP-98/NIP-42 binding** with positive *and* negative controls so neither test can
    pass vacuously (`bridge.rs:2435-2522`, `:2709-2747`).
12. **Zero `unsafe`, zero `unwrap()` outside tests**; the 5 production `expect()`s are all
    infallible-by-construction HMAC/serialize calls in `invite_token.rs`.


## Module: buzz-relay — git hosting (`crates/buzz-relay/src/api/git`)
### Aspect: Security

#### 1. Authentication and authorization per path

| Path | Authentication | Authorization | Rate limit |
|---|---|---|---|
| `GET /git/{o}/{r}/info/refs` | NIP-98 (`transport.rs:76-231`) | **none beyond relay membership** | **none** |
| `POST …/git-upload-pack` | NIP-98 | **none beyond relay membership** | **none** |
| `POST …/git-receive-pack` | NIP-98 | pre-receive hook → policy endpoint → `git_perms::evaluate_push` | **none** |
| `POST /internal/git/policy` | HMAC-SHA256 over the body + `require_localhost` | n/a (it *is* the authorizer) | none |

No git route calls `enforce_http_admission` or touches `state.admission_rate_limiter` (verified by absence in `transport.rs`; contrast `api/bridge.rs:24-56`). Only the shared `track_metrics`/`TraceLayer`/CORS layers apply (`router.rs:187-190`).

##### The NIP-98 gate, and what it deliberately does not check

`GitAuth::from_request_parts` (`transport.rs:79-231`) in order: header present → `Nostr ` prefix → base64 (standard, then URL-safe-no-pad) → UTF-8 → **`bind_community(db, Host)`** → derive expected `u` → extract `method` **from the event itself** → `verify_nip98_event(json, expected_url, event_method, None)` → parse event → relay-membership gate.

| Weakening | Rationale in code | Residual risk |
|---|---|---|
| Method binding is tautological (`transport.rs:158-183`) — the method fed to the verifier is read from the signed event | git's credential helper signs once with `GET` and reuses for `POST` | one token authorizes **clone and push** on the repo for its ±60 s window. A token leaked from a read operation is a write token. |
| Body hash not verified (`body = None`, `transport.rs:183-190`) | pack stream cannot be buffered | a captured token can be replayed with a *different* pack body |
| Replay dedup explicitly disabled (`transport.rs:192-198`) | credential protocol reuses one token across the session | unlimited replay within ±60 s from any network position |
| Token is repo-scoped, not service-scoped (`transport.rs:143-157`) | git does not pass query strings to credential helpers | same as row 1 |

What it does get right: the expected `u` host comes from the **server-resolved tenant**, never `config.relay_url`'s host, so a token signed for community A cannot authenticate against community B (`transport.rs:227-234`, `git_expected_url` `:235-253`). Pinned by two mutation-bite tests (`transport.rs:2101-2141`). Note the token is `.git`-sensitive: `/git/o/r` and `/git/o/r.git` produce different expected URLs even though both resolve to the same pointer.

##### The membership gate is off by default

`enforce_relay_membership` returns `Ok(None)` immediately when `require_relay_membership == false` (`api/mod.rs:130-131`), and that flag **defaults to false** (`config.rs:483-485`, pinned `config.rs:954-955`). On a default-configured relay, the entire authorization content of BR-GIT-07 is: *possess any secp256k1 keypair and sign a NIP-98 event for the right host*. Combined with the absence of read authorization (below), a freshly generated keypair can clone every repo in the community.

##### There is no read authorization at all

Neither `info_refs` (`transport.rs:539-594`) nor `upload_pack` (`:786-827`) consults kind:30617, the `buzz-channel` binding, channel membership, channel visibility, or the `git_repo_names` registry. The module doc claims "No public repos for v1" (`transport.rs:60-66`); operationally the model is "all repos are readable by anyone who clears the transport gate." A private channel's repository is not private.

#### 2. The internal policy endpoint

##### `require_localhost` (`mod.rs:38-52`)

```rust
let is_loopback = req.extensions().get::<ConnectInfo<SocketAddr>>()
    .map(|ci| ci.0.ip().is_loopback()).unwrap_or(false);
if !is_loopback { return 403 }
```

- **Fail-closed on missing `ConnectInfo`** — `unwrap_or(false)` (`mod.rs:47`).
- Layer order: `.layer(RequestBodyLimitLayer).layer(from_fn(require_localhost))` (`mod.rs:63-64`) — the last layer added is outermost, so the loopback check runs **before** the 1 MiB body limit and before JSON deserialization. Correct ordering.
- Uses the real peer address, never `X-Forwarded-For`. Good.
- **Confirmed: unreachable over the UDS listener.** `main.rs:1182` serves the same router with `.into_make_service()` (no `ConnectInfo`), while the TCP listeners use `.into_make_service_with_connect_info::<SocketAddr>()` (`main.rs:1193`, `:1214`). Any request arriving over `BUZZ_UDS_PATH` gets 403 from `require_localhost`. Since the hook always dials `http://127.0.0.1:<port>` (`transport.rs:911-915`) this is harmless today, but it is an undocumented coupling: adding a UDS-only deployment mode silently breaks every push.
- **Weakness:** any process that can open a loopback socket to the relay port clears this gate — including a same-host reverse proxy. If the relay sits behind a proxy on the same host or in the same pod network namespace, an *external* request forwarded by that proxy arrives with a loopback peer and passes. The design comment calls this "defense-in-depth" (`mod.rs:36-37`), which is the right framing: HMAC is the real control.

##### HMAC

- Key: `config.git_hook_hmac_secret`. When `BUZZ_GIT_HOOK_HMAC_SECRET` is set it must be ≥ 32 chars (`config.rs:863-871`); when unset a fresh 32-byte random hex secret is generated **per process** (`config.rs:739-744`).
- **The "pods disagree in a multi-pod deployment" concern does not apply.** The hook's callback URL is hardcoded to `127.0.0.1:<bind_addr.port()>` (`transport.rs:911-915`), so the callback never leaves the process that spawned `receive-pack`. A per-process secret is functionally correct. Separately, the Helm chart *does* inject a shared secret from a Kubernetes Secret (`deploy/charts/buzz/templates/deployment.yaml:168-172`), so production runs with an operator-managed value.
- Pre-image is length-prefixed and `|`-separated to eliminate field-confusion (`policy.rs:113-157`), covers `repo_id`, `repo_owner`, `community_id`, `pusher_pubkey`, every `(old_oid, new_oid, ref_name, is_ancestor)` sorted by ref name, and the timestamp. Tamper coverage is pinned individually for each field, including `is_ancestor` (`policy.rs:512-520`) and `community_id` (`:547-557`).
- Comparison is constant-time (`subtle::ConstantTimeEq`, `policy.rs:167-171`).
- Cross-language agreement between the bash hook and Rust is verified by two tests that actually run `bash` + `openssl` (`policy.rs:592-773`).
- **No replay cache.** Within the 30 s TTL (`policy.rs:51`, `:243-256`) an identical signed payload can be replayed arbitrarily. The endpoint is read-only, so the impact is limited to repeated DB lookups and decision disclosure to a local attacker who observed one callback.
- Secret exposure surface: the value lives in the relay process env and in the `receive-pack` child's env, so it is readable via `/proc/<pid>/environ` by anything running as the same UID. It is never logged (`policy.rs:238-240` logs only `repo`).

##### Structural validation before HMAC

`policy.rs:176-234` validates shape first, explicitly to avoid spending HMAC CPU on garbage. Two gaps:

1. `repo_id` is checked only for `1..=64` length (`policy.rs:176-178`) — **no alphabet restriction**, unlike `validate_repo_id` (`transport.rs:278-292`) and `side_effects::validate_repo_id` (`handlers/side_effects.rs:2390-2398`). The hook only ever sends a validated value, so this is a defense-in-depth gap rather than a live hole; a `repo_id` containing `/` or `..` would reach the `d_tag` DB query (parameterized) and the log line.
2. OIDs are pinned to exactly 40 hex (`policy.rs:224-230`), so a SHA-256 push would be rejected here even before the deeper snapshot bug (§7).

#### 3. Command injection through argv

**No injection surface found.** Every `Command` is built with `Command::arg`/`args` (no shell), and every argv element is either a compile-time literal, a formatted integer (`--window-memory=`, `--max-pack-size=`), or an OS-generated tempdir path:

| Argv element | Origin |
|---|---|
| `<workspace>` | `TempDir::new_in(scratch_dir).path()` (`hydrate.rs:294-299`) |
| `pack-<digest>.pack` / `.idx` | digest re-validated as 64 hex before use (`pack_cache.rs:458-470`) |
| `<tmp>/compact` | tempdir path, UTF-8 checked with a typed error (`cas_publish.rs:702-705`) |
| `--format=%(refname) %(objectname)` | literal |

`owner` and `repo` **never reach argv**. They reach only (a) the object-store key via `pointer_key` after `validate_repo_id`, and (b) the hook env as `BUZZ_REPO_OWNER`/`BUZZ_REPO_ID`. Ref names never reach argv either — they are written as file paths (§4) and as JSON body fields.

Within the bash hook, `repo_id` and `ref_name` are interpolated into JSON after `sed 's/\\/\\\\/g; s/"/\\"/g'` (`hook.rs:74-76`, `:126`), and every variable expansion is double-quoted. `set -eo pipefail` plus `: "${VAR:?…}"` guards make a missing variable abort (`hook.rs:35`, `:41-47`). `LC_ALL=C` is forced for byte-accurate lengths and sort order (`hook.rs:36-38`).

#### 4. Path traversal

| Vector | Guard | Verdict |
|---|---|---|
| `owner` in the pointer key | exactly 64 lowercase hex (`transport.rs:268-275`) | sound |
| `repo` in the pointer key | `[A-Za-z0-9._-]{1,64}`, no leading `.`, no `..`, one `.git` strip (`transport.rs:278-292`) | sound |
| Ref name → file path `path.join(refname)` (`hydrate.rs:352`) | `is_safe_refname` requires a `refs/` prefix and rejects `..`, `//`, leading/trailing `/`, and any char outside `[A-Za-z0-9/_.-]` — applied on **both** write (`manifest.rs:210-232`) and read (`hydrate.rs:337-350`) | sound; no escape from the workspace, and `config`/`HEAD`/`hooks/` are unreachable because every name must start with `refs/` |
| `manifest.head` → `HEAD` file content | re-validated before write (`hydrate.rs:366-372`) | sound |
| Pack digest → cache shard/entry directory names | `validate_digest` requires 64 ASCII hex; pinned against `../pack`, `/absolute`, short, non-hex (`pack_cache.rs:458-470`, `:586-593`) | sound |
| `pack_digest` → `idx/<digest>` key | re-validated in `idx_key_for_pack_digest` (`store.rs:227-241`, pinned `:923-929`) | sound |
| Pack key → digest in hydrate | must literally start `packs/` (`hydrate.rs:303-305`) | sound |
| Cache parent directory | rejected if a symlink (`pack_cache.rs:118-126`) | sound; note `git_repo_path` itself is **not** symlink-checked (`config.rs:388-400`) |
| Policy `ref_name` | `refs/` prefix, ≤ 256, no `..`, no byte ≤ 0x20 or 0x7f (`policy.rs:225-233`) | sound |

Residual: `is_safe_refname` permits a manifest holding both `refs/heads/a` and `refs/heads/a/b`. Hydration writes `a` as a file first (BTreeMap order) and then `create_dir_all(.../a)` fails, making the repo permanently un-hydratable (500 on every read). Unreachable via push (`git receive-pack` rejects D/F conflicts), but a manifest written out of band would brick the repo.

#### 5. Resource exhaustion

| Surface | Bound | Gap |
|---|---|---|
| Compressed request body | `git_max_pack_bytes` (default 500 MiB) | applies to `GET info/refs` too |
| gzip expansion ("gzip bomb") | independent decoded cap — 64 MiB for upload-pack, `git_max_pack_bytes` for receive-pack (`transport.rs:745-784`) | pinned by three tests (`:1823-1887`) |
| Concurrent git subprocesses | `git_semaphore`, default 20, non-blocking acquire (`transport.rs:318-338`) | **global, not per-tenant and not per-pubkey** — one authenticated caller with 20 slow pushes starves git for every community on the pod |
| `info/refs` fast path | acquires **no** permit (`transport.rs:552-577`) | unbounded concurrent pointer+manifest GETs; each is 2 S3 round-trips, so an authenticated caller can amplify request rate into object-store load with no local bound |
| Pack compaction | process-global semaphore of 1, 300 s acquire + 600 s operation timeout, ≤ 1 000 000 objects (`cas_publish.rs:86`, `:576-586`, `:593-664`, `:1058`) | a repo just under the object cap can hold the single compaction slot for 600 s |
| Pack-object CPU/memory | `--threads=1`, `--window 10`, `--window-memory=64 MiB` (`cas_publish.rs:83-84`) | |
| Subprocess wall time | 120 s advertise, 300 s pack ops, 300 s pack-objects/count-objects | **`hydrate::run_git` has no timeout** (`hydrate.rs:451-470`), and it runs `git index-pack` on attacker-influenced pack bytes (`:420`). A crafted pack can hold a semaphore permit indefinitely. |
| Pack bomb / delta bomb | `index-pack` runs **without `--strict`** (deliberately, `hydrate.rs:410-416`), so connectivity is not re-verified; structural validation (CRC, type tags, internal refs) still runs | combined with the missing timeout above, this is the weakest resource surface in the module |
| Repo bytes | `git_max_repo_bytes` accumulated with `checked_add` (`hydrate.rs:318-329`); publish-side `parent_hydrated_bytes + new_pack` (`cas_publish.rs:1132-1146`) | |
| Manifest cardinality | ≤ 128 packs, ≤ 10 000 refs (`manifest.rs:205-215`) | |
| Buffered outputs | 4 MiB advertisement, 1 MiB receive-pack status, 4 MiB manifest, 4 MiB ref snapshot, 64 KiB `count-objects`, 64 KiB stderr | |
| pkt-line frame | over-long payload degrades to `0004` + `error!` rather than emitting a 5-hex length (`transport.rs:424-454`) | prevents stream corruption, not resource use |
| Pack cache disk | `git_pack_cache_max_bytes` (default 5 GiB) with LRU eviction — but **only entries with `Arc::strong_count == 1` are evictable**; if every entry is in flight the prune loop breaks and the cache exceeds its bound (`pack_cache.rs:365-390`) | plus bypassed over-capacity entries are never counted (`:303-313`) |
| Unauthenticated | any base64 blob in `Authorization` triggers one `bind_community` DB lookup before NIP-98 verification (`transport.rs:110-130`) — with no rate limiting | low-cost unauthenticated DB touch |

#### 6. Pack-content validation and injection risks

**Pack contents are never inspected by the relay.** The only validation is whatever `git index-pack` (no `--strict`) and `git receive-pack` perform. Specifically not checked: object types, tree entry modes, path names inside trees, blob sizes, presence of `.gitmodules`/`.gitattributes`/`.git*` entries, symlink entries.

| Risk | Assessment |
|---|---|
| **Symlink entries in trees** | Inert server-side: every workspace is a **bare** repo (`git init --bare`, `hydrate.rs:182`) and the relay never checks out a working tree. The risk transfers to consumers — notably the web repo browser, which does check out into LightningFS/IndexedDB in the browser (`web/src/features/repos/git-client.ts:86-113`). |
| **`.git/config` / `core.hooksPath` injection via pack contents** | Not reachable. The bare repo's `config` is written only by `git init --bare`; pack objects never become working-tree files. Additionally `harden_git_env` sets `GIT_CONFIG_NOSYSTEM=1` and `GIT_CONFIG_GLOBAL=/dev/null` (`transport.rs:302-309`), and `receive_pack` force-overrides `core.hooksPath` via `GIT_CONFIG_COUNT/KEY_0/VALUE_0` (`transport.rs:924-930`) so a repo-local setting cannot redirect the hook. |
| `hydrate::run_git` env is weaker | It omits `GIT_CONFIG_GLOBAL` and relies on `HOME=<cwd>` to make the global lookup miss (`hydrate.rs:451-465`), while its own doc claims parity with `harden_git_env`. Exploiting it would require writing `<cwd>/.gitconfig` — the cwd is a fresh tempdir or a pack-cache staging dir, neither of which receives pack contents. |
| `PATH` is inherited | `harden_git_env` copies the parent `PATH` (`transport.rs:304`) and `hydrate::run_git` likewise. Anyone who can influence the relay's `PATH` controls which `git`, `curl`, `openssl` binaries run. Same trust boundary as the process itself. |
| **The `idx/` object is the one non-content-addressed object** | `idx/<pack-digest>` is keyed by the *pack's* digest, not by the idx bytes (`store.rs:227-241`), so `get_idx` cannot digest-verify it. The only defense is `git verify-pack` (`hydrate.rs:388-407`), with regeneration on failure. An attacker with bucket write access who wins the `If-None-Match: *` create race for a pack that has no sidecar yet controls bytes that git will parse. Every other object in the design is digest-verified. |

#### 7. Integrity findings

##### F-SEC-1 (High) — pointer creation is not gated by announcement or quota

`receive_pack` performs no kind:30617 or `git_repo_names` lookup (`transport.rs:858-962`). `hydrate_for_write` creates a fresh empty workspace when the pointer is absent (`hydrate.rs:227-244`), and `cas_publish` will `put_pointer(IfNoneMatchStar)` (`cas_publish.rs:1208-1237`). The only ref-level control is the pre-receive hook, and **git does not run pre-receive when the client sends zero ref commands** (`execute_commands` is skipped), so `PackOutput.ok` is true and `finalize_push` proceeds.

Consequences: (a) an authenticated caller can create a pointer to the canonical empty manifest for an arbitrary `(owner, repo)` in the community, bypassing name reservation (`handlers/side_effects.rs:2441-2513`) and the `git_max_repos_per_pubkey` quota (`:2486-2492`); (b) on an existing repo, a zero-command push re-installs the same digest under a **fresh ETag**, invalidating concurrent legitimate pushers' `If-Match` and turning their pushes into 409s — a cheap, authenticated CAS-contention DoS.

Bounds on the impact: no ref *content* can be published (that requires ≥ 1 command ⇒ the hook ⇒ kind:30617 lookup), and the squatted manifest is byte-identical to the announce seed, so a later legitimate announce still succeeds via `seed_manifest_pointer`'s digest-equality path (`handlers/side_effects.rs:2656-2688`). It does, however, falsify the design's stated invariant that "pointer-absence means never announced" (doc §Implementation Correspondence), on which `info_refs`'s unambiguous 404 depends.

*Basis: code structure plus `git receive-pack` command-dispatch semantics. Not confirmed by a live behavioral test — that test does not exist.*

##### F-SEC-2 (High) — no read authorization

See §1. Any caller past the transport gate reads any repo in the community. On a default (open) relay that is any keypair holder.

##### F-SEC-3 (Medium) — silent ref-wipe path for non-40-hex OIDs

`snapshot_workspace_state` `warn!`s and **skips** any ref whose oid is not exactly 40 hex (`cas_publish.rs:325-328`). Every neighbouring layer accepts 64 (`manifest.rs:156`, `manifest_event.rs:129`, `transport.rs:473-479`). In a SHA-256 repository the snapshot would be empty, `resolve_published_head` would still produce a valid head (`cas_publish.rs:355-381`), `validate()` would pass, and the CAS would publish `refs: {}` — a silent deletion of every ref. Unreachable today (`init_bare_repo` always creates SHA-1, `hydrate.rs:182`), but the failure mode is drop-and-continue rather than fail-closed.

##### F-SEC-4 (Medium) — the safety of the publish fence rests on the workspace, not the status parse

`PackOutput.ok = status.success() && !receive_pack_report_rejected(stdout)` (`transport.rs:1136-1139`). A client that does not negotiate `report-status` produces no status stream at all, so a hook decline yields `ok == true` and the CAS runs. It is **safe anyway**, because a pre-receive decline rejects all commands and never migrates quarantined objects, so `refs_after == parent.refs` and the CAS installs an identical digest. This reasoning is nowhere stated in the code or the design doc, which present the report-status scan as the fence ("the primary fence for a denied push", `transport.rs:1128-1135`). The real invariant is weaker and stronger at once, and should be written down.

##### F-SEC-5 (Medium) — one token authorizes both read and write

Because method binding is tautological and the token is repo-root-scoped (§1), a NIP-98 token captured from a `git clone` is a valid `git-receive-pack` token for ±60 s. On non-TLS deployments this is a direct privilege escalation; the code names HTTPS as the mitigation (`transport.rs:143-152`).

##### F-SEC-6 (Low) — conformance gate is disableable

`BUZZ_GIT_CONFORMANCE_PROBE=false` skips the fatal A3 admission probe (`main.rs:470-472`), allowing the relay to serve git against a backend that has not been admitted against the one load-bearing axiom.

##### F-SEC-7 (Low) — probe pollutes the production media bucket

`git_store` is constructed from the **media** S3 credentials and bucket (`state.rs:694-701`), and the probe writes immutable objects that it deliberately never deletes ("immutable probe writes accumulate by design", `store.rs:864-866`): at default settings 3 `packs/<digest>` + 3 `probe/inm-race/<digest>` objects per boot, in the same bucket as user media. Only `probe/pointer-<uuid>` is cleaned up (`store.rs:618`, `:871`).

##### F-SEC-8 (Low) — cross-tenant cache and semaphore coupling

`git_pack_cache` is keyed on the pack digest alone, with no community component (`pack_cache.rs:271`, `:311`), and `git_semaphore` is process-global (`state.rs:729`). Content sharing is safe (content addressing), but occupancy and eviction are cross-tenant, and pack-digest existence is inferable by timing.

##### F-SEC-9 (Low) — denial reasons are echoed to the pushing client

The hook does `cat "$RESP_FILE" >&2` on any non-200 (`hook.rs:145`), so the policy endpoint's body reaches the pusher's terminal. Today those bodies are static strings plus `DenialResponse` JSON containing role names and rule text (`policy.rs:404-416`) — intentional UX. It does mean any future error detail added to that endpoint leaks to unprivileged pushers.

##### F-SEC-10 (Low) — `repo_id` alphabet not re-validated at the policy endpoint

`policy.rs:176-178` checks length only, weaker than the two other `validate_repo_id` implementations. Defense-in-depth gap for a localhost-only, HMAC-authenticated endpoint.

#### 8. What an unauthenticated caller can reach

| Attempt | Result |
|---|---|
| Any git route without `Authorization` | 401 before the body is read (`transport.rs:86-97`); the body-limit layer prevents a large unauthenticated upload |
| Any git route with a garbage `Authorization: Nostr <b64>` | one `bind_community` DB lookup, then 401 (`transport.rs:110-190`) — unrated |
| `POST /internal/git/policy` from off-host | 403 by `require_localhost` (`mod.rs:49`) before deserialization |
| `POST /internal/git/policy` from loopback without a valid MAC | 403; structural checks run first, HMAC second (`policy.rs:176-241`) |
| Repo-existence probing | not possible unauthenticated. Once authenticated, a nonexistent repo and an unmapped Host both return the same 404 body (`transport.rs:130`, `:579`), which is a deliberate non-disclosure choice. |

#### 9. TOCTOU analysis

| Race | Guard | Verdict |
|---|---|---|
| Two announces for the same repo name | `INSERT … ON CONFLICT DO NOTHING` in `git_repo_names`; the preceding `repo_name_owner` peek is only for classification (`handlers/side_effects.rs:2470-2513`) | sound |
| Announce seed vs. concurrent push | `seed_manifest_pointer` uses `IfNoneMatchStar` and accepts `LostRace` **only** when the existing pointer names the same empty digest (`handlers/side_effects.rs:2656-2688`); `ensure_manifest_pointer` accepts any existing pointer for a same-owner re-announce (`:2696-2726`) | sound |
| Reservation exists but pointer missing | repaired on re-announce by `ensure_manifest_pointer`; a fresh `Reserved` claim that fails to seed is rolled back, an `AlreadyOwned` one is not (`handlers/side_effects.rs:2545-2570`) | sound, and explicitly reasoned in comments |
| Two concurrent pushes to one repo | the pointer CAS; loser gets 409, no retry (`cas_publish.rs:1260-1291`) | sound (§Business Rules 7.3) |
| Pointer read vs. CAS predicate | one GET yields both ETag and body (`store.rs:446-479`); `ParentState` carries the ETag through to the CAS with no re-read (`cas_publish.rs:1208-1226`) | sound |
| Pack-cache publication race across processes | atomic rename; adopt the winner's directory on failure (`pack_cache.rs:314-330`) | sound |
| Stale-session GC vs. a live but stalled process | a process paused > 10 min can have its live session directory deleted by another process's startup sweep (`pack_cache.rs:482-509`) | cache-only impact |
| Compacted pack size validation vs. upload | re-checked after read; a size change is an error (`cas_publish.rs:812-827`) | sound |
| **Pointer namespace vs. name reservation** | **none** — see F-SEC-1 | gap |

#### 10. Cryptography inventory

| Use | Primitive | Site |
|---|---|---|
| Content addressing | SHA-256 (`sha2`) | `store.rs:218-225`, `:878-885` |
| Hook callback authentication | HMAC-SHA256 (`hmac` + `sha2`), constant-time compare (`subtle`) | `policy.rs:113-171` |
| Transport auth | NIP-98 (schnorr over secp256k1, delegated to `buzz_auth`) | `transport.rs:183-190` |
| kind:30618 signing | relay's nostr keypair | `manifest_event.rs:110-114` |
| Dev HMAC secret generation | `rand::random::<[u8; 32]>()` | `config.rs:739-744` |
| Probe nonces | `uuid::Uuid::new_v4()` | `store.rs:583` |

`unsafe`: **0**. TODO/FIXME/XXX/HACK: **0**.


## Module: buzz-relay — moderation, admin & background workers (`crates/buzz-relay/src`)
### Aspect: Security

---

#### 1. Authorization on every privileged path

| # | Path | Transport auth | Handler authorization | Tenant fence | Ban re-check | Freshness |
|---|---|---|---|---|---|---|
| 1 | 9040 ban | NIP-42 / NIP-98 + `Scope::MessagesWrite` (`ingest.rs:216`) | `authorize_moderation_action(Ban)` (`moderation_commands.rs:156-166`) | `tenant.community()` on both role reads (`moderation_authz.rs:96-116`) | **yes** (`:103-108`) | ±120 s (`:113-121`) |
| 2 | 9041 unban | same | `Unban` (`:235-244`) | yes | yes | yes |
| 3 | 9042 timeout | same | `Timeout` (`:274-283`) | yes | yes | yes |
| 4 | 9043 untimeout | same | `Untimeout` (`:338-347`) | yes | yes | yes |
| 5 | 9044 resolve | same | `ResolveReport` (`:399-408`) + report must exist in-tenant (`:414-417`) | yes | yes | yes |
| 6 | `GET /moderation/reports` | NIP-98 + replay guard (`bridge.rs:2067-2069`) | `ViewQueue` (`bridge.rs:2054-2067`) | host-derived tenant | **no** | n/a |
| 7 | `GET /moderation/audit` | same | same | same | **no** | n/a |
| 8 | 9030 add member | NIP-42/98 + `Scope::AdminUsers` (`ingest.rs:251-256`) + channel-scoped-token rejection (`ingest.rs:1538-1542`) | inline `sender_role != "admin" && != "owner"` (`relay_admin.rs:177`) | `tenant.community()` (`:135`) | **NO** | ±120 s (`:125-130`) |
| 9 | 9031 remove member | same | inline (`:227`) + self-check (`:231`) + atomic role-conditional delete (`:243`) | yes | **NO** | yes |
| 10 | 9032 change role | same | inline `sender_role != "owner"` (`:286`) + self-check (`:290`) | yes | **NO** | yes |
| 11 | 9033 set workspace icon | same | inline (`:148`) | yes | **NO** | yes |
| 12 | 9035 archive identity | NIP-42/98 + `Scope::UsersWrite` (`ingest.rs:266`) | 3 consent paths (`identity_archive.rs:228-251`), NIP-70 `["-"]` required (`:156-167`) | `tenant.community()` on role + profile query (`:241`, `:279`) | **NO** (subject to ingest's own gate — see §3) | ±120 s (`:141-153`) |
| 13 | 9036 unarchive | same | same | yes | (as above) | yes |
| 14 | 30350 push lease | NIP-42/98, author-only kind (`kind.rs:120`) | none beyond authorship — the lease is self-owned | tenant-derived origin binding (`push_lease.rs:494`, `:585-596`) | subject to ingest's gate | expiration-only skew (`:138-143`) |
| 15 | 1984 report | NIP-42/98 + `Scope::MessagesWrite` (`ingest.rs:212`) | **none** — any authenticated principal may report | target must resolve in-tenant (`report.rs:56`, `:69`) | **deliberately bypassed** (`ingest.rs:1551-1562`) | none |
| 16 | 42000 feedback | same | **none** | tenant-scoped insert (`product_feedback.rs:57`) | **deliberately bypassed** (`ingest.rs:1564-1567`) | none |
| 17 | `POST /operator/communities` | NIP-98 against `RELAY_OPERATOR_API_ORIGIN` + replay guard (`api/operator.rs:104-135`, `:155-163`) | `RELAY_OPERATOR_PUBKEYS` allowlist, fail-closed on empty (`community_provisioning.rs:255-266`) | **above the tenant fence by design** (`:3-14`) | n/a | NIP-98 window |
| 18 | workflow `send_message` | none — internal sink | workflow owner must be a channel member unless the channel is open (`workflow_sink.rs:243-251`) | community comes from the run, never `config.relay_url` (`:190-210`) | **no** | n/a |
| 19 | push delivery | outbound NIP-98 signed by the relay key (`push_runtime.rs:551-565`) | 3-layer read authorization per wake (`:222-233`, `:250`) + membership revalidation at send (`:372-402`) | per-community claim (`:322-325`) | **no** | claim/generation fences |
| 20 | storage sweep | none — leader-gated background task | Postgres advisory lock (`main.rs:1414-1421`) | reads the whole bucket across all tenants | n/a | n/a |

##### 1.1 Findings

**S-1 (HIGH) — Relay-admin commands (9030–9033) have no ban re-check.** `ingest.rs:1639` exempts `is_relay_admin_kind` from the durable ban/timeout write gate, on the stated grounds that "relay-admin commands retain their separate authorization policy" (`ingest.rs:1626-1627`). That separate policy is `relay_admin.rs:133-142`, which reads only `relay_members.role` — grep for `restriction`, `banned`, or `moderation_restriction_state` in `relay_admin.rs` returns **zero hits**. The moderation handler pointedly does re-check, with an in-code rationale that HTTP NIP-98 requests and already-authenticated sockets can reach the handler without a fresh NIP-42 challenge (`moderation_commands.rs:97-101`). That exact reasoning applies verbatim to 9030–9033 and is not implemented. A banned owner or admin with a surviving socket, or issuing over NIP-98, can add members, remove members, change roles, and rewrite the workspace icon.

**S-2 (MEDIUM) — The "reject channel-scoped API tokens" contract for 9040–9044 is not enforced.** `moderation_commands.rs:50` pins it as part of the routing contract. Ingest rejects channel-scoped tokens only for relay-admin kinds (`ingest.rs:1512-1516`) and leave requests (`:1520-1523`). The generic "channel-scoped tokens cannot publish global events" gate sits at `ingest.rs:1747-1750`, **after** the moderation dispatch's early `return` at `:1582-1586`, so it never runs for 9040–9044. Combined with `required_scope_for_kind` granting them only `Scope::MessagesWrite` (`ingest.rs:216`), a legacy channel-scoped WS API token held by a community owner/admin can issue a community-wide ban. Privilege is not escalated beyond the token holder's community role, but the token's channel confinement is silently void.

**S-3 (MEDIUM) — Community moderation has no self-action prevention.** `moderation_commands.rs` never compares actor to target. An owner issuing 9040 against their own pubkey passes `decide_authority` (`moderation_authz.rs:149-150`), writes the ban row (`:169`), and then has their own sockets closed cluster-wide (`:195-200`). Recovery is blocked by the handler's own ban re-check (`:103-108`), so it requires a second owner/admin or direct DB access. Contrast `relay_admin.rs:231-233` (`cannot remove yourself`) and `:290-292` (`cannot change your own role`), which do implement the check. An admin is protected only incidentally, by the peer guard tripping on their own `admin` role (`moderation_authz.rs:165-167`).

**S-4 (MEDIUM) — Legacy provisioning mode can rotate an existing community's owner.** Without `create_only`, an operator-signed request runs `bootstrap_owner` on an existing community, demoting the previous owner to admin (`community_provisioning.rs:321-334`). This is documented and deliberate (`:236-247`) but it makes the operator allowlist a community-takeover capability, not just a creation capability. The mitigation ("clients provisioning on behalf of end users must use create_only", `:317-320`) is prose, not code.

**S-5 (LOW) — 1984 and 42000 accept writes from banned actors in the missed-disconnect window.** Both are dispatched before the restriction gate (`ingest.rs:1541`, `:1562` vs `:1613`). For reports this is explicitly intended and reasoned (`ingest.rs:1551-1559`: reports are non-actioning signals, mod-only visibility). For 42000 product feedback there is **no** stated rationale — a banned user can keep writing 32 KiB bodies plus 64 KiB of tags and `imeta`-verified media references into the operator feedback table indefinitely.

**S-6 (LOW) — No authorization at all on report submission.** Any authenticated principal in the tenant may report any in-tenant event, blob, or pubkey. Idempotency is per signed event id (`buzz-db/src/moderation.rs:39`), so distinct signed reports from one pubkey all land. See §7.

---

#### 2. Audit-trail coverage of privileged mutations

`buzz-audit` declares 11 actions (`buzz-audit/src/action.rs:5-29`). Production emits exactly two: `EventCreated` (`handlers/event.rs:583`) and `MediaUploaded` (`api/media.rs:428`). Confirmed by grep: **none of the 12 assigned files references `buzz_audit`, `AuditAction`, or `state.audit_tx`.**

> ARCHITECTURE.md:497 says "**10 audit actions**" and omits `MediaUploaded` — stale by one variant, and it enumerates declared actions rather than emitted ones.

##### 2.1 Unaudited privileged mutations (the finding)

| Mutation | `events` row | Hash chain | `moderation_actions` | Durable trail? |
|---|---|---|---|---|
| 9040 ban | no | **no** | yes (`moderation_commands.rs:180`) | `moderation_actions` only |
| 9041 unban | no | **no** | yes (`:256`) | `moderation_actions` only |
| 9042 timeout | no | **no** | yes (`:298`) | `moderation_actions` only |
| 9043 untimeout | no | **no** | yes (`:358`) | `moderation_actions` only |
| 9044 resolve | no | **no** | yes (`:453`) | `moderation_actions` only |
| **9030 add member** | no | **no** | **no** | **`tracing::info!` only** (`relay_admin.rs:203-209`) |
| **9031 remove member** | no | **no** | **no** | **`tracing::info!` only** (`:268-272`) |
| **9032 change role** | no | **no** | **no** | **`tracing::info!` only** (`:327-332`) |
| **9033 set workspace icon** | no | **no** | **no** | **`tracing::info!` only** (`:164`) |
| **operator provisioning / owner rotation** | n/a | **no** | **no** | **`tracing::info!` only** (`community_provisioning.rs:302-308`, `:336-343`) |
| 1984 report | no | **no** | no (own table) | `moderation_reports` |
| 42000 feedback | no | **no** | no (own table) | `product_feedback` |
| 30350 push lease | yes | **no** — ingest returns at `:2199` before the dispatch path | no | `push_leases` + event row |
| 9035/9036 identity archive | **yes** (fall-through, `ingest.rs:1909-1912`) | **YES** `EventCreated` | no | full |

**S-7 (HIGH) — 12 of the 14 privileged kinds this module owns produce no hash-chain entry.** Bans, unbans, timeouts, report resolutions, member additions/removals, role changes, and ownership-affecting provisioning are all invisible to `verify_chain()`. The only tamper-evident audit surface in the relay does not cover the relay's own privilege model.

**S-8 (HIGH) — Relay-admin and operator-provisioning mutations have no durable audit trail of any kind.** No event row, no hash-chain entry, no `moderation_actions` row. The record of "who made X an admin" or "who rotated community Y's owner" exists only in ephemeral process logs. `moderation_actions` at least gives moderation a queryable trail — membership and ownership changes get nothing.

**S-9 (MEDIUM) — `moderation_actions` is not hash-chained.** It is a plain table (`migrations/0006_moderation.sql`), so the moderation trail is not tamper-evident. `moderation_actions.matched_principal` — the field designed to record which NIP-OA principal an enforcement check matched (`buzz-db/src/moderation.rs:139`) and surfaced verbatim by the audit API (`api/bridge.rs:2184`) — is hard-coded `None` at its single writer (`moderation_commands.rs:540`) and is therefore **always NULL in production**. `reason_code`, `private_reason`, and `channel_id` are likewise always NULL (`:536-539`). Five of nine audit columns carry no data.

**S-10 (LOW) — Audit attribution for relay-signed events.** Moderation notice DMs are audited as `EventCreated` with the **relay pubkey** as actor (`moderation_notices.rs:178`), so the moderator behind the notice is not recoverable from the chain. Workflow messages correctly attribute the workflow owner rather than the relay key (`workflow_sink.rs:355`, rationale `handlers/event.rs:584-590`) — the moderation path does not follow that pattern.

---

#### 3. Self-ban and privilege-escalation prevention

| Vector | Prevented? | Mechanism |
|---|---|---|
| Admin bans the owner | **yes** | peer guard (`moderation_authz.rs:163-171`) |
| Admin bans a fellow admin | **yes** | same |
| Admin times out the owner/an admin | **yes** | same |
| Admin bans a non-member / plain member | allowed by design | guard trips on role, not on a missing row (`:165-167`, test `:246-266`) |
| Admin **unbans/untimeouts** the owner or an admin | **NOT prevented** | deliberately excluded from the guard (`:165`), documented as benign and owner-reversible (`:157-162`) |
| Admin promotes self to owner | **yes** | `cannot change your own role` (`relay_admin.rs:290-292`) and `cannot set role to owner` (`:300-302`) |
| Admin promotes another to admin | **yes** | owner-only (`relay_admin.rs:187-189`) |
| Anyone reaches owner via events | **yes** | 9030 refuses `role=owner` (`:184-186`), 9032 refuses `owner` (`:300-302`); the only paths are `RELAY_OWNER_PUBKEY` config or the operator endpoint |
| Admin removes the owner | **yes** | atomic `RemoveResult::IsOwner` (`:257-259`) |
| Admin removes another admin | **yes** | admin path is `remove_relay_member_if_role(…, "member")` (`:243`) → `RoleMismatch` (`:263-265`) |
| Promote-between-read-and-delete TOCTOU | **yes** | single conditional delete, explicitly to close that race (`:235-241`) |
| Channel owner/admin escalates to community actions | **yes** | `ChannelRole` authorizes only `DeleteMessage`/`Kick` (`moderation_authz.rs:174-178`, test `:294-310`) — and is unreachable anyway |
| **Owner self-bans** | **NO** | see S-3 |
| Actor bans by raw `p`-tag pubkey with no in-tenant existence check | allowed | `moderation_commands.rs:147` accepts any 64-hex; only 9044's report target is tenant-resolved (`:414`) |

**S-11 (LOW) — Denial reasons leak internal detail.** `authz_denial` wraps *any* `anyhow` error from the authorization seam, including `sqlx` errors from `get_relay_member`, as `restricted: {e}` (`moderation_commands.rs:548-550` + `moderation_authz.rs:99`, `:111`, `:127`). A DB failure therefore surfaces to an unauthorized client as a `restricted:` message containing database text. The HTTP moderation read path does this correctly, normalizing every denial to a fixed 403 string (`api/bridge.rs:2062-2067`).

---

#### 4. Ban cascade to NIP-OA agents; timeout non-cascade

##### 4.1 The cascade (auth seam only)

`handlers/auth.rs:107-186`, evaluated after signature verification and **before** the allowlist and relay-membership gates:

1. Read restriction state for the authenticated pubkey (`:120-131`).
2. Only if that is `Clear`, extract the cryptographically-proven NIP-OA owner from the self-proving auth tag with **no DB round-trip** (`:135-140`) and read the owner's restriction state (`:141-155`).
3. `Banned` ⇒ `blocked: you are banned from this community`, `AuthState::Failed`, reason frame on the **control** channel (not `send`, to avoid racing the cancel), then immediate socket close (`:158-186`).
4. `DbError` ⇒ deny with `error: internal error checking restriction state` — deliberately distinguished so a transient blip never tells an innocent user they are banned or pins `Failed` on a false premise (`:112-118`).

Semantics: **owner ban ⇒ agents banned; agent ban is agent-only** (`handlers/auth.rs:104-106`).

##### 4.2 The non-cascade (ingest write path)

`ingest.rs:1613-1650` checks `auth.pubkey()` **only**, with no owner resolution. Documented as deliberate (`ingest.rs:1624-1637`): for bans the cascade is *structural* — an agent whose owner is banned can never authenticate, so its socket never exists to reach ingest. Timeout has no auth-seam presence at all (it is write-block-only), so:

**S-12 (MEDIUM, accepted) — An owner timeout does not silence that owner's agents.** Stated as a "deliberate Phase-1 asymmetry" (`ingest.rs:1633-1634`), with the reason given as `IngestAuth` not carrying the self-proving auth tag and the follow-up named as a restriction-state cache (`:1608-1611`). Operationally: timing out a human leaves every agent they own free to post.

**S-13 (MEDIUM) — The cascade is absent on the moderation-command path.** `moderation_commands.rs:103-108` reads restriction state for `event.pubkey` only. An agent whose owner is banned but whose own pubkey is clear can, over HTTP NIP-98 (which never crosses the auth seam), reach `handle_moderation_command`. It will then be denied by `authorize_moderation_action` unless the agent pubkey itself holds `owner`/`admin` in `relay_members` — an unusual but not impossible configuration. The same gap applies to `relay_admin.rs` with no ban check at all (S-1).

##### 4.3 Live-enforcement split

Ban closes sockets cluster-wide (`moderation_commands.rs:195-200` → `state.rs:1018-1050`); timeout does not. The Redis publish is `tokio::spawn`ed fire-and-forget whose failure only warns (`state.rs:1043-1047`), justified because the durable row re-rejects at auth (`:1039-1042`) and the ingest gate backstops surviving sockets (`ingest.rs:1589-1596`).

**S-14 (MEDIUM) — A missed disconnect publish leaves read access open indefinitely.** The ingest write gate covers writes; nothing gates an established subscription's *reads*. On a pod that missed the Redis command, a banned user's open socket keeps receiving fan-out until it reconnects. `disconnect_pubkey_clusterwide` returns the local close count (`state.rs:1049`) and the caller discards it (`moderation_commands.rs:195`), so there is no signal that zero sockets were closed.

---

#### 5. Push-lease forgery and endpoint-hijack resistance

| Threat | Control | Site |
|---|---|---|
| Forged lease for another pubkey | 30350 is an author-only kind (`kind.rs:120`); ingest enforces `event.pubkey == auth.pubkey()` (`ingest.rs:1534-1538`); the event is signature-verified before routing | strong |
| Cross-tenant lease injection via the decrypted `origin` | `origin` must equal `canonical_origin(config.relay_url, tenant.host())` — scheme from config, authority from the **server-resolved** tenant (`push_lease.rs:494`, `:585-596`); module doc pins that tenant selection never comes from the plaintext (`:3-5`) | strong |
| Metadata leakage via public tags | only `d`/`expiration`/`exec`/`alt` permitted; anything else is a hard reject; duplicates rejected; every tag must have exactly 2 elements (`push_lease.rs:106-126`) | strong |
| Subscribing to another user's traffic | every positive filter must be narrowed (`:292-295`) and every `#p` value must equal the lease author (`:298-303`) | strong |
| Gift-wrap (1059) recipient-activity leak through wake timing | match-time counterpart of REQ's `#p` gate: the filter must be self-`#p` **and** the event must carry that author's `p` tag (`push_runtime.rs:292-310`, rationale `:287-290`); tested `:580-620` | strong |
| Reading events the lease author cannot see | `reader_authorized_for_event` (`push_runtime.rs:222-225`) + channel membership from the pre-loaded pair set (`:226-233`) + membership revalidation at send (`:372-402`) | strong |
| **Endpoint hijack** (claiming another device's APNs token) | DB-enforced: one active address per endpoint tuple ⇒ `EndpointAlreadyLeased` → `invalid: endpoint already leased` (`ingest.rs:2196-2200`); `endpoint_hash` is SHA-256 of the endpoint (`push_lease.rs:535`) | **first-claim-wins, no proof of possession** — see S-15 |
| Lease rollback / replay | NIP-01 addressable ordering (`StaleEvent`) + strictly-increasing generation watermark (`StaleGeneration`) (`ingest.rs:2189-2193`); generation must be a positive JSON-safe integer (`push_lease.rs:198-200`) | strong |
| Source-event reuse across addresses | `SourceEventCollision` (`ingest.rs:2205-2209`) | strong |
| Quota exhaustion | 16 active leases per author (`push_lease.rs:479`), 12 further hard-coded quotas (`:513-521`), 64 KiB ciphertext, 32 KiB plaintext (`:477-478`) | strong |
| JSON parser confusion (duplicate keys) | custom `DeserializeSeed` rejecting duplicates at every depth (`push_lease.rs:383-455`), tested at top level and nested (`:637-649`) | strong |
| Executor-key confusion | `exec` must equal `config.push_executor_key_id` (`:484-486`) | strong |
| Non-finite JSON numbers | rejected (`push_lease.rs:419`) | strong |
| Stale 410 killing a rotated lease | endpoint invalidation is generation-fenced (`push_runtime.rs:456-465`) | strong |
| Double-delivery via a fresh request id after timeout | request id is the durable wake row id, stable across retries, pinned by an HTTP test (`push_runtime.rs:486-490`, `:626-655`) | strong |

**S-15 (MEDIUM) — Endpoint ownership is first-claim-wins with no proof of possession.** The relay never validates that the submitting pubkey controls the APNs token; it accepts any 1..=4096-byte `endpoint` string, hashes it, and lets the DB unique constraint arbitrate (`push_lease.rs:533-535`, `:563-570`). A user who learns another device's token can register it first and thereafter receive that device's wakes — or, once registered, block the legitimate device with `endpoint already leased`. The wake payload itself is content-free (`DeliveryRequest` carries only `v`/`endpoint_grant`/`request_id`/`expires_at`, `push_runtime.rs:31-37`), so the leak is wake *timing* and *existence*, not message content — but timing plus the 3-layer read authorization means the attacker learns when the *victim's* pubkey has readable traffic, since the lease's own author gates matching. Net: the practical impact is denial of push to the real device plus a notification-timing oracle scoped to the attacker's own readable events.

**S-16 (LOW) — `endpoint_grant` is a misnomer that invites a false sense of opacity.** The DB field is documented as "Opaque endpoint grant issued by the stateless gateway" (`buzz-db/src/push.rs:94`), but the relay stores the client-supplied endpoint verbatim (`push_lease.rs:544`, `:555`) and forwards it unchanged (`push_runtime.rs:507-515`). The stored value is a raw APNs device token — a credential-adjacent secret — persisted in plaintext in `push_leases` and transmitted in every delivery body. It is never logged (verified: `push_runtime.rs` logs only `wake=%id`), which is the one saving grace.

**S-17 (LOW) — `urgent` class dead code creates a latent policy hole.** `URGENT_KINDS = &[]` (`push_lease.rs:16`) and `"urgent"` absent from `supported_classes` (`:509`) means the urgent-kind confinement check at `:281-283` never runs. If a future change adds `"urgent"` to `supported_classes` without also populating `URGENT_KINDS`, the confinement check would then reject *all* urgent leases — fail-closed, so the hole is benign today. `class_rank` in both copies still ranks `urgent` highest (`push_lease.rs:582`, `push_runtime.rs:574`).

---

#### 6. SSRF in push delivery

**Not exposed.** The destination is `state.config.push_gateway_delivery_url`, an operator-configured `url::Url` validated at boot to be:
- scheme exactly `https` (`config.rs:349`)
- host present (`:350`)
- no username, no password (`:351-352`)
- path exactly `/v1/deliveries/apns` (`:353`)
- no query, no fragment (`:354-355`)

Failure to satisfy any of these is a boot-time `ConfigError` (`config.rs:356-360`), i.e. fail-closed. The client-controlled `endpoint_grant` travels only in the JSON body (`push_runtime.rs:507-515`); no user-supplied value ever reaches the URL, headers, or method. The NIP-98 `u` tag is derived from the same validated URL (`push_runtime.rs:552-556`).

Residual risks:

**S-18 (LOW) — Redirects are not disabled.** `reqwest::Client::builder()` at `push_runtime.rs:313-316` sets only `.timeout(...)`; `redirect::Policy` is left at the default (follow up to 10). A compromised or misconfigured gateway host could redirect the delivery POST — including the NIP-98 `Authorization` header and the endpoint token — to an arbitrary host. `buzz-workflow`'s `call_webhook` explicitly disables redirects (per ARCHITECTURE.md:539); this path does not.

**S-19 (LOW) — The default destination is a hard-coded third-party host.** Unset `BUZZ_PUSH_GATEWAY_DELIVERY_URL` falls back to `https://push.buzz.xyz/v1/deliveries/apns` (`config.rs:339`, `:755-758`), which also **enables** the matcher and delivery worker (`main.rs:686-692`) and lease acceptance (`push_lease.rs:480-482`). A self-hosted relay that never configures push will still accept leases, run both loops, and attempt outbound HTTPS to `push.buzz.xyz` — leaking the existence of wakes (and the device token in the body) to a host the operator never chose. Disabling requires explicitly setting the variable to an *empty* string (`config.rs:753`), which is non-obvious.

---

#### 7. Report abuse surface

| Property | Value | Site |
|---|---|---|
| Who may report | any authenticated principal with `Scope::MessagesWrite` | `ingest.rs:212` |
| Rate limiting | **none** in this path. ARCHITECTURE.md:821 confirms the only `RateLimiter` implementation repo-wide is a test stub | — |
| Idempotency | per signed event id (`report_event_id`) | `buzz-db/src/moderation.rs:39` |
| Reportable while timed out | **yes**, by design | `ingest.rs:1551-1562` |
| Reportable while banned (missed-disconnect window) | **yes**, tolerated | `ingest.rs:1554-1559` |
| Target must exist in-tenant | yes for `e` (`report.rs:56-59`) and `x` (`:66-71`); **no** for `p`-only (`:73`) | |
| Note size cap | **none** — `event.content` is stored unbounded (`report.rs:222-228`); only the global event-size limit applies | |
| Self-reporting | not prevented | |
| Auto-action on report | **never**, by design (NIP-56) | `report.rs:3-6` |
| Reporter identity exposure | mod-queue only, never revealed to the author | `buzz-db/src/moderation.rs:42`; enforced by the notice-privacy invariant (`moderation_notices.rs:268-272`) |

**S-20 (MEDIUM) — Unbounded report flooding.** A single principal can generate arbitrarily many distinct signed 1984 events (each a fresh event id, so idempotency does not help) against any in-tenant target, each writing a `moderation_reports` row with an uncapped note. There is no per-pubkey rate limit, no per-target cap, and no dedup on `(reporter, target)`. The moderation read API caps at 500 rows per request (`api/bridge.rs:2053`), so a flood degrades the queue's usability for moderators. Contrast `product_feedback.rs:12-13`, which caps body at 32 KiB and tags at 64 KiB — the report path applies neither.

**S-21 (LOW) — Report note is relayed to the reporter but never sanitized on the resolution path.** 9044's `reason` tag is passed verbatim into the reporter's DM (`moderation_commands.rs:477-481` → `moderation_notices.rs:283-286`). The doc requires it be "safe for the reporter's eyes" (`moderation_commands.rs:44-47`); nothing enforces that. The notice is also indexed into full-text search via `dispatch_persistent_event` (`moderation_notices.rs:178`), so a moderator's `public_reason` becomes searchable content.

---

#### 8. Storage-sweep credential scope

| Property | Value | Site |
|---|---|---|
| Credentials | the **same** `BUZZ_S3_ACCESS_KEY` / `BUZZ_S3_SECRET_KEY` used for media upload and download | `config.rs:622-625`; shared `MediaStorage` handle at `main.rs:1459` |
| Additional IAM requirement | `s3:ListBucket` (or MinIO list) — surfaced in the failure log as an operator hint | `storage_sweep.rs:176-181` |
| Prefix scope | **empty prefix — whole bucket, every tenant** | `buzz-media/src/storage.rs:249-256` |
| Page size | 1000 objects per LIST | `main.rs:1466` |
| Cumulative cap | `BUZZ_STORAGE_SWEEP_MAX_OBJECTS`, default 1,000,000, checked before folding each page | `storage_sweep.rs:65-68`; `buzz-media/src/bucket_index.rs:392-395` |
| Data retained in memory | bounded per-sha/per-binding aggregates only, never the full listing | `bucket_index.rs:228-246` |
| Kill switch | `BUZZ_STORAGE_METRICS=off` — suppresses the sweep and every storage gauge including health ones | `main.rs:1454-1456` |

**S-22 (LOW) — No least-privilege separation for the sweep.** Enabling storage metrics requires adding `s3:ListBucket` to the relay's **read-write** media credential, over the whole bucket. There is no separate list-only credential and no per-tenant prefix restriction. A relay compromise therefore also yields a full cross-tenant object inventory. The kill switch is the only mitigation offered, and it is documented for that purpose (`storage_sweep.rs:42-45`, `deploy/charts/buzz/values.yaml:308-313`).

**S-23 (INFO) — Sweep output is cross-tenant by construction.** `per_community` maps community UUIDs to byte/object totals, and unmapped UUIDs roll into a global `buzz_storage_unmapped_community_bytes` gauge (`storage_sweep.rs:318-330`). Per-community series are gated by the same `EmissionScope` predicate as DB-derived gauges (`main.rs:1474-1477`), so scope exclusion does apply — but the aggregate totals (`buzz_total_storage_bytes`, orphan/anomaly counters) are unscoped and expose whole-deployment volume to any Prometheus scraper.

---

#### 9. Secret handling in logs

Reviewed all `tracing` calls in the 12 files. **No secret material is logged.**

| Candidate | Handling |
|---|---|
| APNs endpoint token (`endpoint_grant`) | never logged; `push_runtime.rs` logs only `wake=%uuid`, `?invalid_at`, and error text |
| Lease ciphertext / decrypted plaintext | `push_lease.rs` contains **zero** `tracing` calls |
| Relay secret key | used at `push_lease.rs:488` and for signing at `moderation_notices.rs:165`, `:198`, `workflow_sink.rs:304`, `push_runtime.rs:562`; never formatted into a log |
| Workspace icon (may be a base64 data URL up to 96 KB) | only its length is logged: `icon_len = icon.len()` (`relay_admin.rs:164`) |
| Moderation notice body / `public_reason` | never logged |
| Report note | never logged |
| Product-feedback body | never logged |
| NIP-98 `Authorization` header | constructed at `push_runtime.rs:551-565`, never logged |
| Pubkeys | logged as hex (`moderation_commands.rs:223`, `relay_admin.rs:203-209`, `identity_archive.rs:90-97`) — public data, appropriate |
| `DATABASE_URL` with an inline dev password | present as a **test-only** fallback literal `postgres://buzz:buzz_dev@localhost:5432/buzz` (`identity_archive.rs:439-440`); inside `#[cfg(test)]`, never logged |

**S-24 (LOW) — Database error text reaches clients.** Not a log issue but the same class of exposure: `error: database error: {e}` (`moderation_commands.rs:174` and 5 more), `database error: {e}` (`relay_admin.rs:137` and 6 more), and `restricted: {e}` wrapping a possible `sqlx` error (`moderation_commands.rs:549`). `push_lease.rs:572` is the only path that deliberately opaques the DB error — at the cost of losing the diagnostic entirely.

---

#### 10. Positive security controls worth preserving

| Control | Site |
|---|---|
| Single authorization seam for all moderation, with a pure exhaustively-tested policy core | `moderation_authz.rs:83-181`, tests `:184-330` |
| Ban re-checked at three independent layers (auth seam, ingest write gate, command handler) with fail-closed DB-error handling at each | `handlers/auth.rs:112-176`, `ingest.rs:1613-1650`, `moderation_commands.rs:103-108` |
| NIP-OA owner→agent ban cascade with no DB round-trip for owner extraction | `handlers/auth.rs:134-155` |
| DB-error denial distinguished from a real ban, so an innocent user is never told they are banned | `handlers/auth.rs:112-118` |
| Fail-closed operator allowlist, empty by default, plus a boot-time requirement that it be paired with an explicit API origin | `community_provisioning.rs:258-266`; `config.rs:577-580` |
| Canonical-authority host validation with a byte-identical round-trip requirement | `community_provisioning.rs:120-146` |
| NIP-98 replay guard on the operator endpoint, fail-closed when the guard itself errors | `api/operator.rs:104-135` |
| Closed-world push-lease validation: 4 permitted tags, 5 permitted filter members, duplicate-key rejection at every JSON depth | `push_lease.rs:106-126`, `:264`, `:383-455` |
| Push-eligible kinds pinned to the SQL trigger by an `include_str!` test | `push_lease.rs:696-710` |
| Gift-wrap wake-timing leak closed at match time | `push_runtime.rs:292-310` |
| Generation-fenced endpoint invalidation and claim-fenced completion | `push_runtime.rs:456-465`; `ClaimedWake.claim_id` (`buzz-db/src/push.rs:141`) |
| Exact-HTTPS-path gateway URL validation, fail-closed at boot | `config.rs:341-361` |
| NIP-IA live-revocation check: replacing your kind:0 auth tag invalidates outstanding owner-signed archive requests | `identity_archive.rs:270-296`, integration test `:515-578` |
| NIP-70 protected-event tag mandatory on archive requests | `identity_archive.rs:156-167` |
| Notice-DM privacy invariant: bodies built only from sanitized fields, never reporter identities or raw notes | `moderation_notices.rs:20-23`, `:268-272` |
| Workspace icon rejects `javascript:` and non-image data URLs, control chars, and whitespace | `relay_admin.rs:69-94`, tests `:441-450` |
| Workflow tenancy derived from the run's community, failing closed on an unmapped community — never re-derived from `config.relay_url` | `workflow_sink.rs:190-210` |
| Zero `unsafe`, zero production `unwrap()`, zero production `panic!` across 6,720 LOC | measured |


## Module: buzz-relay — huddle audio, tunnel & conformance seam (`crates/buzz-relay/src`)
### Aspect: Security

---

#### 1. Authentication on the audio WebSocket

##### 1.1 The five gates, in order

| # | Gate | Mechanism | Line | Effect if it fails |
|---|---|---|---|---|
| 1 | Tenant binding | `tenant::bind_community(&state.db, Host)` — **before** the WS upgrade | `handler.rs:74-88` | 404 with a generic body; deliberately non-enumerable |
| 2 | Connection budget | shared `conn_semaphore` | `handler.rs:90-99` | 503 |
| 3 | Community liveness | `db.is_community_active` | `handler.rs:156-164` | silent close |
| 4 | NIP-42 | `state.auth.verify_auth_event(event, challenge, relay_url)` | `handler.rs:220-238` | `{"type":"error","message":"auth failed"}` + close |
| 5 | Relay membership | `api::relay_members::enforce_relay_membership` | `handler.rs:244-262` | `restricted: not a relay member` |
| 6 | Channel membership | `ensure_membership` | `handler.rs:265-286` | `not a member` |

##### 1.2 Huddle auth does reuse NIP-42, faithfully

The audio route runs the **same verifier object** as the main relay door
(`state.auth`, `handler.rs:222`), with:

- a fresh per-connection challenge from `buzz_auth::generate_challenge()`
  (`handler.rs:176`) — not reusable, not derived from anything client-supplied;
- a **per-tenant** expected relay URL via
  `api::bridge::nip42_expected_relay_url(&state.config.relay_url, &tenant)`
  (`handler.rs:219`), so an auth event minted for community A cannot be replayed
  against community B on a multi-tenant deployment;
- NIP-OA tag extraction *before* the event is consumed (`handler.rs:217`), so the
  delegated-owner path works identically to HTTP;
- a hard 5 s window (`handler.rs:61`, `:190`) after which the socket closes.

The identity used for everything downstream is `auth_ctx.pubkey`
(`handler.rs:240-242`) — never `auth_msg.event.pubkey` directly, and never a
client-supplied identifier.

##### 1.3 What the pre-auth surface exposes

Before authenticating, an anonymous caller reaches:

- The tenant lookup (a DB query per attempt) and the resulting 404/upgrade
  distinction. Because the 404 body is identical for "no such community" and
  "DB error" (`handler.rs:80-88`), community enumeration by host is not possible.
- A WebSocket upgrade + one `challenge` frame, holding a `conn_semaphore` permit and
  a spawned connection task for up to 5 s. **This is the primary unauthenticated
  resource cost** — see §6.1.
- Nothing about whether `{channel_id}` exists: the channel is not looked up until
  after auth (`handler.rs:1164`), so an unauthenticated caller cannot probe channel
  existence.

---

#### 2. Authorization / channel-membership enforcement

##### 2.1 Enforcement chain (`ensure_membership`, `handler.rs:1153-1235`)

1. `db.get_channel(tenant.community(), channel_id)` — community-scoped, so a
   cross-tenant channel UUID simply does not resolve (`handler.rs:1162-1167`).
2. Archived channels are rejected **first**, before any membership logic
   (`handler.rs:1168-1170`) — an auto-ended huddle cannot be rejoined by a member.
3. For a TTL channel, the lifecycle parent is verified against a **creator-signed
   kind-48100 link in the DB**, not the client's `parent_channel_id`
   (`handler.rs:1172-1191`). This is the notable hardening: the comment at
   `handler.rs:1172-1175` says explicitly "instead of trusting the UUID supplied by
   the client during audio auth". Without it, a member of any channel could inject
   48101/48102/48103 events into an arbitrary parent channel by claiming it.
4. Member fast path (`is_member_cached`, `handler.rs:1196-1203`).
5. `visibility == "open"` bypass (`handler.rs:1205-1207`).
6. Auto-add, restricted to TTL channels whose caller is a member of the **resolved**
   parent (`handler.rs:1208-1231`).
7. Otherwise `"not a member"`.

##### 2.2 What a non-member can reach

| Actor | Reach |
|---|---|
| Unauthenticated | challenge frame only; §1.3 |
| Authenticated, non-relay-member, `require_relay_membership=false` (**default**) | passes gate 5 unconditionally — `api/mod.rs:67` returns `OpenRelay`, `api/mod.rs:130-131` maps it to `Ok`. So on a default deployment gate 5 is decorative |
| Authenticated, non-channel-member, channel `visibility != "open"`, non-TTL | refused at gate 6 |
| Authenticated, channel `visibility == "open"` | **admitted with no membership row at all** (`handler.rs:1205-1207`). Any authenticated pubkey can join and both hear and speak in any open channel's huddle. This mirrors the relay's read model for open channels, but audio is a live bidirectional capability, not a read |
| Authenticated member of a parent channel, huddle is TTL+private | **auto-added as a member** (`handler.rs:1219-1227`) — a write to `channel_members` triggered by a WebSocket connect, attributed to `channel.created_by` rather than to the joiner |

##### 2.3 Residual authorization gaps

- **No per-channel huddle permission.** Channel membership is the only
  authorization; there is no "may join voice" role, no admin-only huddle, and no
  moderator capability. Any admitted peer can transmit to every other peer.
- **No revocation mid-call.** Membership is checked once at join. Removing a member
  from the channel does not disconnect them; there is no membership-change watcher
  on the audio path (contrast `fan_out_event_to_local_subscribers`, which re-gates
  per event — `handler.rs:1318`).
- **The auto-add write is a side effect of an unauthenticated-at-HTTP-level
  request.** It happens after NIP-42, so the actor is authenticated, but the
  resulting row records `added_by = channel.created_by` (`handler.rs:1224`) —
  attributing the membership to someone who did not perform the action. Audit
  attribution is therefore wrong for auto-added huddle members.

---

#### 3. Mesh trust model

##### 3.1 Who can become a mesh peer

Inbound QUIC connections are accepted only from a runtime already present in
membership, and membership is populated from the Redis **ready registry**
(`buzz-relay-mesh/src/runtime.rs:275-283`, `:309-320`). Records are attested
against the deployment's relay signing key
(`MeshMembership::with_expected_relay_pubkey(relay_keypair.public_key().to_hex())`,
`mesh_boot.rs:441-443`) — the comment cites the review rationale: "all pods share the
relay signing key, so a seed attested by any other key is foreign and rejected
(possession is not authorization)".

**Trust boundary:** anyone who can (a) write to the shared Redis and (b) sign with
the relay keypair can join the mesh as a peer. Both are pod-level secrets, so the
mesh trust domain is exactly "processes holding the relay identity". There is no
per-pod identity, no mTLS beyond iroh's own key exchange, and no authorization
distinction between peers — every peer is fully trusted once admitted.

##### 3.2 Who can claim ownership of a room

Ownership is **not** claimable over the mesh. It is granted only by the Redis
fenced CAS:

- `ACQUIRE_SCRIPT` (`directory.rs:20-34`) refuses when a live lease exists, and
  INCRs the non-expiring generation counter only when it creates one.
- `resolve_join` never treats a mesh hint as authority (`join.rs:317-379`); the
  owner side re-validates every registration's fence against Redis
  (`join.rs:1231-1245`).
- `accept_inbound` rejects a stream whose `fenced.owner_runtime_id` is not this pod
  (`join.rs:1079-1086`), so a peer cannot ask pod X to act as owner for a lease
  pod Y holds.
- `HuddleControlAcceptor` never calls `acquire` — it only ever `validate`s
  (verified: the acceptor's only directory call is `join.rs:1231`).

So a mesh peer can only ever *route* to the true owner. Ownership forgery requires
Redis write access, which is the same trust domain as §3.1.

##### 3.3 Can a peer forge a forwarded frame? Yes, within the mesh trust domain

This is the sharpest finding in this aspect.

**Control frames are Redis-fenced. Media datagrams are not.**

`MeshAudioRouter::on_media_datagram` (`mesh.rs:204-250`) performs exactly two checks
before delivering audio to local WS clients:

1. `GenerationFloor::check(session_id, generation)` — a **purely local, in-memory
   monotone counter** (`mesh.rs:102-128`). It accepts any generation `>= floor`, and
   a *higher* generation simply advances the floor (`mesh.rs:113-119`). It never
   consults Redis.
2. `rooms.get_unambiguous_by_channel(session_id)` — a **community-free** lookup
   (`room.rs:526-541`).

Consequences for a mesh peer (or anything that can reach the pod's QUIC endpoint and
pass §3.1 admission):

- It can inject audio into **any** huddle room on that pod by sending a datagram
  whose `fenced.session_id` is the target channel UUID and whose `generation` is
  large. `fenced.owner_runtime_id` is **never checked** on the datagram path — it is
  documented as "advisory for routing/diagnostics"
  (`buzz-relay-mesh/src/wire.rs:90-92`), and `mesh.rs` honours that literally.
- The injected `payload[0]` is an arbitrary `peer_index`, so the frame is attributed
  to whichever participant occupies that index — a **spoofing primitive**: the
  injected audio reaches every local peer except the impersonated one
  (`room.rs:422-429`), so the impersonated speaker cannot hear their own forgery.
- Because the check is monotone-advancing, a single high-generation forged datagram
  **poisons the floor** and causes the legitimate owner's subsequent frames to be
  rejected as stale — a targeted per-session audio DoS
  (`mesh.rs:106-112` returns `RejectStale` for anything below the poisoned floor).
- The room lookup is community-free, so the **tenant boundary does not hold on the
  media path**. It fails closed only when two *active* communities happen to share
  the channel UUID (`room.rs:534-539`). With a single active community per UUID —
  the normal case — a peer addresses that community's room without naming it.
  `mesh.rs:36-42` and `room.rs:519-525` both acknowledge that "the current mesh media
  envelope does not carry a community identifier".

The design intent is stated at `mesh.rs:44-51`: "Every datagram carries a
`FencedHeader`. Both ends reject frames whose generation is stale". The code
implements the *stale* half only; there is no owner check and no Redis check. Within
the stated trust model (all mesh peers are relay-key holders) this is defensible;
it is not defensible against a single compromised pod, which today can eavesdrop on
nothing but can inject into and silence any huddle on any peer pod.

##### 3.4 Cross-pod control-path forgery — well defended

By contrast the `HuddleControl` path is tight:

| Attack | Defence |
|---|---|
| Claim to be another pod | `hello.sender != from` rejected (`join.rs:1060-1065`) |
| Route a stream to a non-owner | `fenced.owner_runtime_id != local` rejected (`join.rs:1079-1086`) |
| Register into another tenant's room | fence keyed `(community, session)`; a wrong community keys a lease the owner never wrote → `no_active_lease` **before** `add_peer` (`join.rs:1231-1245`); pinned `join.rs:2332-2392` |
| Switch community mid-stream | latched and rejected (`join.rs:1200-1209`) |
| Change fence mid-stream | `f != fenced` → `OwnerMismatch` (`join.rs:1191-1198`) |
| Mint a `CommunityId` from wire input | impossible by type: `CommunityId` is non-`Deserialize`, so the wire field is a raw `Uuid` re-minted explicitly at `join.rs:1211` (rationale `join.rs:851-865`) |
| Send owner→non-owner replies to the owner | protocol violation, stream torn down (`join.rs:1302-1310`) |
| Leak peers by abandoning a stream | teardown always drops every registered peer (`join.rs:1345-1367`); pinned `join.rs:2253-2330` |

---

#### 4. Session-directory lease forgery

| Attack | Feasibility |
|---|---|
| Present a fabricated `FencedHeader` on a control frame | Fails — `validate_fenced_header` compares against the live Redis lease on generation **and** owner (`directory.rs:408-437`) |
| Replay an expired lease's fence | Fails — the generation counter never expires, so `known > 0` and `frame.generation < known` yields `StaleGeneration`; after expiry-before-takeover it yields `NoActiveLease` (`directory.rs:382-406`); pinned `directory.rs:882-919` |
| Claim a future generation to look "newer" | Fails on the control path — `FutureGeneration` when it does not match the live lease exactly (`directory.rs:408-422`). **Succeeds on the media datagram path** (§3.3), where a higher generation is *accepted and pinned* |
| Forge the lease value in Redis | Requires Redis write access — the same trust domain as the mesh. Nothing signs or MACs the lease value; it is a plain `owner_hex\|gen\|profile` string (`directory.rs:31`) |
| Corrupt the lease to a malformed value | Detected: wrong part count, `generation == 0`, non-hex owner, or unknown profile all error (`directory.rs:495-531`) rather than defaulting |
| Cross-tenant lease collision | Prevented by the key shape `buzz:{community}:tunnel:{session}` (`directory.rs:453-461`); pinned `directory.rs:625-637` |
| Cross-**profile** session hijack | Partially prevented: `reliable.rs:99-105` refuses a lease whose profile is not `ReliableStream`. The **huddle path does not check the profile** — `HuddleDirectory::owner_of` (`join.rs:107-127`) discards `lease.profile` entirely, so a `reliable-stream` lease on the same session id would be honoured as a huddle owner lease. Since huddle session ids are channel UUIDs and reliable session ids are caller-chosen, this becomes reachable through the demo route (§5) |

##### 4.1 Lease/fence blind window

Between lease expiry (30 s TTL) and the owner's next renew tick (10 s), the pod
still believes it owns the room. Local WS peers have **no per-frame fence** — the
comment at `handler.rs:568-575` says exactly this: "local WS peers have no per-frame
fence". The `resolve_join_owner_ready` loop exists to prevent an *ownerless* owner
peer at admission; it does not shorten the post-admission blind window. A network
partition longer than 30 s can therefore produce two pods each fanning out locally
for up to ~10 s, with cross-pod media fenced but same-pod media not.

---

#### 5. `POST /_mesh/demo/echo` — the tenant-boundary bypass

`api/mesh_demo.rs:59-79`:

- **No authentication of any kind.** No NIP-42, no NIP-98, no secret, no
  `enforce_relay_membership`. The only gates are `state.mesh().is_some()` and
  `config.mesh_demo_echo` — both 404 when off (`api/mesh_demo.rs:66-72`).
- **`community_id` is taken from the request body** (`api/mesh_demo.rs:50-52`,
  minted at `:92` via `CommunityId::from_uuid(req.community_id)`), not from the
  `Host` header. Every other route in the relay derives the tenant from the host;
  this one is the exception.
- `session_id` is also caller-chosen (`api/mesh_demo.rs:53-54`).

From the tunnel side, what that grants when both flags are on:

1. **Arbitrary lease creation in any community's key space.** `router.join` →
   `directory.acquire(community_from_body, session_from_body, …)` writes
   `buzz:{arbitrary}:tunnel:{arbitrary}:lease` and INCRs the matching non-expiring
   generation counter (`reliable.rs:87-95` → `directory.rs:200-225`).
2. **Denial of huddle ownership.** A huddle's `session_id` **is** its `channel_id`
   (`handler.rs:324`). An unauthenticated POST with
   `{community_id: <victim>, session_id: <channel uuid>}` takes the lease for that
   channel under `profile = reliable-stream`. A subsequent huddle join resolves
   `RemoteOwner` pointing at the attacker's chosen pod (or `LocalOwner` on a stale
   snapshot) and — because the huddle path never checks `lease.profile`
   (§4, last row) — proceeds against a lease minted by a non-huddle caller. The
   lease is deliberately never renewed (`api/mesh_demo.rs:11-15`), so it self-heals
   after 30 s, but it can be re-posted.
3. **Unbounded monotone generation inflation.** Repeated post/expire cycles INCR the
   never-expiring counter without limit, and `validate_fenced_header` computes
   `known = max(lease.generation, counter)` (`directory.rs:375-380`) — so a legitimate
   owner at a lower generation is rejected as `StaleGeneration`.
4. **Redis key-space growth** in arbitrary community namespaces, with no TTL on the
   generation keys.

Mitigating facts: both `BUZZ_MESH` and `BUZZ_MESH_DEMO_ECHO` default **off**
(`config.rs:498-500`, `:516-518`) and require an explicit `on`/`true`/`1`; the route
404s otherwise, indistinguishably from a build without it
(`api/mesh_demo.rs:67-72`). The module doc labels it "testbed-only" and "Not a
product flow" and says "the route stays demo-gated until it is deleted"
(`api/mesh_demo.rs:1-22`). It is nonetheless registered on the **public** router
(`router.rs:123`), not behind the admin host.

Also unauthenticated: `GET /_mesh` (`router.rs:230`, handler `:399-406`) on the
health router, which `router.rs:222-224` documents as having "no auth". It returns
the serialized `MeshStatus` — peer runtime ids, addresses, and drain state — to
anyone who can reach the health port.

---

#### 6. DoS surfaces

##### 6.1 Unauthenticated

| Surface | Bound | Gap |
|---|---|---|
| WS upgrade + 5 s auth window | one `conn_semaphore` permit + one spawned task + 1 DB query (`bind_community`) per attempt | The permit is held for the **whole** 5 s window (`handler.rs:90-99`, `:190`). Because the semaphore is **shared with ordinary relay WebSockets** (`handler.rs:105-107`, verified `state.rs:727`), an attacker can exhaust the relay's entire WebSocket budget through the audio route alone, at ~`max_connections/5s` request rate. There is no per-IP rate limit and no separate audio budget |
| Pre-auth text frames | 8192 bytes/message at the parser; unlimited *count* inside the 5 s window | The auth loop `continue`s on oversize and on non-`auth` JSON (`handler.rs:194-204`), so a client may send unlimited 8 KB frames for 5 s. Each one is `serde_json::from_str`-parsed into a full `nostr::Event` attempt |
| `POST /_mesh/demo/echo` (when enabled) | 1 MB body limit (`router.rs:130`) | §5 — unbounded Redis writes, no auth, no rate limit |
| `GET /_mesh` | none | information disclosure, and an unbounded-rate endpoint on the health port |

##### 6.2 Authenticated

| Surface | Bound | Gap |
|---|---|---|
| Peers per room | 25 soft (`room.rs:50`), 255 hard | Enforced inside the admission lock (`room.rs:233-238`) — sound |
| Rooms per pod | **none** | `AudioRoomManager.rooms` is an unbounded `DashMap` (`room.rs:490`). Any authenticated member can create a room per channel; `get_or_create` has no cap (`room.rs:503-509`). Bounded only indirectly by `conn_semaphore` and by channel count |
| Frames per second per peer | **none** | Only a 4096-byte size cap (`handler.rs:961-964`). No token bucket, no bitrate cap. A single peer can flood at line rate; each frame costs one allocation plus N−1 refcount clones (`room.rs:398-410`) |
| Fan-out amplification | 1 → N−1 within a pod | With 25 peers, one 4 KB frame produces 24 outbound 4 KB frames = **24×** amplification, capped only by the 8-slot drop-on-full queues. Cross-pod, the owner additionally re-emits to every remote peer's mesh sink (`mesh.rs:262-283`), so a non-owner pod's single inbound frame becomes 24 datagrams from the owner — the same 24× ratio, now on the network rather than in-process |
| Tasks per connection | 4–6 (`handler.rs:663`, `:667`, `:670`, `:704`, `:733`) | Multiplied by the shared connection budget |
| `GenerationFloor.seen` | **unbounded** `DashMap` (`mesh.rs:90`) | One entry per session ever observed. Removed only by explicit `forget` (`handler.rs:755`, `:763`, `:909`); no TTL sweep. A peer can create entries at will by sending datagrams for fresh random session ids (§3.3) — each one a permanent 24-byte-plus-overhead map entry. **Unauthenticated-within-the-mesh unbounded memory growth** |
| `HuddleOwnerRegistry.entries` | one per owned room | Removed by `release`/`drain` (`join.rs:734-761`). `release` only fires when the room empties (`handler.rs:868-877`); a room whose archive fails keeps its entry and its renewer |
| Inbound mesh streams | **unbounded** `tokio::spawn` per stream (`mesh_boot.rs:270-274`, `:290-298`) | No concurrency limit, no per-peer cap |
| Reliable-stream payload | 1 MiB per frame (`reliable.rs:31`), 16 MiB wire cap | No total-bytes, rate, or session-count bound; each `Data` frame allocates a fresh `Vec<u8>` (`reliable.rs:471-474`) |
| Redis calls per join | 1–3 (`owner_of`, possibly `acquire`, possibly `validate`) plus up to **25 retries** in the owner-ready loop (`join.rs:423-441`) | A contended room can drive 25 `owner_of` round trips in 500 ms per joining connection |

##### 6.3 Not a DoS surface (verified)

- Audio queues never grow: every audio send is `try_send` with drop-on-full
  (`room.rs:409`, `:427`, `handler.rs:1115`).
- `broadcast` roster channel is bounded at 64 with `Lagged` recovery, so a slow
  control-stream consumer cannot grow memory (`room.rs:179`, `join.rs:1174-1182`).
- The WS parser rejects oversize messages before the handler allocates
  (`handler.rs:116-119`), pinned by test (`handler.rs:1417-1427`).

---

#### 7. Data handling / confidentiality

| Concern | Finding |
|---|---|
| Audio content | Never decoded, never logged, never persisted. `broadcast_frame` copies bytes (`room.rs:398-410`); `forward_media` passes them through (`join.rs:1758-1766`). The relay has no plaintext of the Opus body beyond the transport |
| End-to-end encryption | **Not present for audio.** `buzz-relay-mesh/src/wire.rs:118-121` claims "the client frame's encrypted content is NIP-44 between client endpoints, so server-side plaintext of the media itself never exists". Nothing in this group or in `desktop/src-tauri/src/huddle/` encrypts the Opus payload — `relay_api.rs:303` builds a plain v2 frame. The relay does hold the audio in the clear (TLS-terminated), and so does any mesh peer it forwards to. The comment overstates the guarantee |
| Client telemetry | `level_dbov`, `seq`, `ts_48k` are logged at `trace` only (`handler.rs:996-1003`), and `wire.rs:16-20` forbids them from feeding trust decisions. Verified: no other consumer |
| Pubkeys in logs | Full hex, at `warn`/`info` (`handler.rs:255`, `:283`, `:419`, `:602`, `:880`). Consistent with the rest of the relay |
| Lifecycle events | Relay-signed (`handler.rs:1268`) with `p` = participant pubkey and content `{"ephemeral_channel_id": …}` (`handler.rs:1238-1256`). So huddle attendance is a **permanent, publicly queryable record** in the parent channel for anyone who can read it — the intended design, worth noting as a privacy property |
| Conformance trace | Projects community UUID, **full host string**, 16-hex-char pubkey prefix, 16-hex-char event-id prefix, raw channel UUID (`conformance/mod.rs:55-91`). No payloads, no signatures, no timestamps. In production nothing is written (`NoopTracer`, `state.rs:798`); `JsonlTracer` would write plaintext JSONL to a local file with no redaction and no permissions handling (`tracers.rs:37-46` uses default `OpenOptions`) |
| Redis lease values | Contain a mesh runtime id and a profile string — no secrets (`directory.rs:31`) |
| Information leak defence | Deliberate and documented: `Ended > Full > VersionMismatch` precedence exists so a rejected joiner cannot learn the room's pinned protocol version (`room.rs:216-227`, pinned `room.rs:759-782`) |

---

#### 8. Resource limits — consolidated table

| Limit | Value | Configurable? | Line |
|---|---|---|---|
| WS message / frame size | 8192 B | no | `handler.rs:52` |
| Binary audio frame | 4096 B | no | `handler.rs:44` |
| Text control frame | 8192 B | no | `handler.rs:47` |
| Auth timeout | 5 s | no | `handler.rs:61` |
| Heartbeat interval | 30 s | no | `handler.rs:55` |
| Missed pongs before close | 3 | no | `handler.rs:58` |
| Peers per room (soft) | 25 | no | `room.rs:50` |
| Peer index space | 0..=254 | no | `room.rs:146-152` |
| Per-peer audio queue | 8 | no | `room.rs:40` |
| Per-peer control queue | 32 | no | `room.rs:45` |
| WS data queue | 16 | no | `handler.rs:659` |
| WS control queue | 8 | no | `handler.rs:660` |
| Roster broadcast | 64 | no | `room.rs:179` |
| Protocol version ceiling | 2 | no | `handler.rs:124` |
| Owner-ready retries | 25 × 20 ms | no | `join.rs:387-388` |
| Huddle renew interval | 10 s | no | `join.rs:452` |
| Reliable renew interval | 10 s | no | `reliable.rs:34` |
| Lease TTL | 30 s | not via env (`with_lease_ttl` exists but production uses `new`) | `directory.rs:17`, `mesh_boot.rs:512` |
| Reliable payload chunk | 1 MiB | no | `reliable.rs:31` |
| Demo echo timeout | 10 s | no | `api/mesh_demo.rs:45` |
| Connection budget | `max_connections` | **yes** — shared with all relay WS | `state.rs:727` |
| Rooms per pod | unbounded | — | `room.rs:490` |
| `GenerationFloor` entries | unbounded | — | `mesh.rs:90` |
| Inbound mesh stream concurrency | unbounded | — | `mesh_boot.rs:270`, `:290` |
| Frames/sec per peer | unbounded | — | — |

Nineteen of the twenty-three bounded limits are compile-time constants. See
configuration for the tunability assessment.

---

#### 9. Ranked security findings

| # | Severity | Finding | Line |
|---|---|---|---|
| 1 | **High** | Media datagrams are not Redis-fenced and the `owner_runtime_id` is never checked; a mesh peer can inject audio into any room on any peer pod, spoof any `peer_index`, and poison the generation floor to silence the legitimate owner | `mesh.rs:204-250` |
| 2 | **High** | The media datagram envelope carries no community, and the room lookup is community-free, so the host-derived tenant boundary does not hold on the media path (fails closed only on an active UUID collision) | `room.rs:519-541`, `mesh.rs:221-227` |
| 3 | **High** (when both flags on) | `POST /_mesh/demo/echo` is unauthenticated **and** takes `community_id` from the body, allowing arbitrary lease creation and generation inflation in any community's Redis key space — including on a channel UUID that is a live huddle's `session_id` | `api/mesh_demo.rs:50-95`; `router.rs:123` |
| 4 | Medium | The huddle path ignores `lease.profile` (`HuddleDirectory::owner_of` discards it), so a `reliable-stream` lease is honoured as huddle ownership — the mechanism that makes finding 3 reach huddles | `join.rs:107-127` vs. `reliable.rs:99-105` |
| 5 | Medium | Audio connections share the global `conn_semaphore` and hold a permit for the whole 5 s pre-auth window, so an unauthenticated client can exhaust the relay's entire WebSocket budget via `/huddle/…/audio` | `handler.rs:90-99`, `:190` |
| 6 | Medium | `GenerationFloor.seen` is an unbounded `DashMap` with no TTL, growable at will from within the mesh | `mesh.rs:90`, `:131-133` |
| 7 | Medium | No frame-rate or bitrate limit; 24× in-pod and 24× cross-pod fan-out amplification from one authenticated peer | `room.rs:398-411`, `mesh.rs:262-283` |
| 8 | Medium | `require_relay_membership` defaults **false**, making the relay-membership gate on the audio route a no-op on default deployments | `config.rs:481-483`; `api/mod.rs:67`, `:130-131` |
| 9 | Medium | Membership is checked once at join with no revocation path; removing a member does not disconnect their audio | `handler.rs:1196-1234` |
| 10 | Low-Medium | `visibility == "open"` grants full bidirectional audio to any authenticated pubkey with no membership row | `handler.rs:1205-1207` |
| 11 | Low-Medium | Post-admission lease blind window: local WS peers have no per-frame fence, so a >30 s partition can produce two pods fanning out locally for up to ~10 s | `handler.rs:568-575`; `join.rs:452` |
| 12 | Low | Ephemeral auto-add writes a membership row attributed to `channel.created_by`, not to the joiner — wrong audit attribution | `handler.rs:1219-1227` |
| 13 | Low | `GET /_mesh` is unauthenticated and returns peer runtime ids, addresses, and drain state | `router.rs:230`, `:399-406` |
| 14 | Low | `buzz-relay-mesh/src/wire.rs:118-121` claims NIP-44 client-to-client encryption of media payloads; no such encryption exists — the relay and every mesh peer see plaintext Opus | verified across `audio/` and `desktop/src-tauri/src/huddle/` |
| 15 | Low | `mesh-llm` dev-dependencies are pinned by mutable git **tag**, not commit SHA, in the CI dependency graph | `Cargo.toml:87-88` |
| 16 | Informational | The conformance coverage guard is armed in production but writes into `NoopTracer`, so an `ImplBug` breach is silently discarded; `JsonlTracer` has zero callers | `state.rs:798`; `tracers.rs:20-24` |


## Module: buzz-relay-mesh (`crates/buzz-relay-mesh`)
### Aspect: Security

**Default posture: everything below is OFF by default.** `BUZZ_MESH` must be an
explicit `on`/`true`/`1` (`config.rs:498-500`); absent/`off`/typo → `boot_mesh`
returns `Ok(None)` before binding a socket, writing Redis, or spawning a task
(`mesh_boot.rs:417-419`). No config in this repo enables it (grep of `.env.example`,
`deploy/`, helm: zero `BUZZ_MESH`). So the findings below are latent, not live —
except **F-01 (`GET /_mesh`)**, which is registered unconditionally.

---

#### 1. Trust model between relay pods

Three layers, in decreasing strength:

| Layer | Mechanism | Strength |
|---|---|---|
| **Transport identity** | iroh/QUIC mutual TLS; peer identity = ed25519 node key; `RuntimeId` derived from `conn.remote_id()` (`peer.rs:58`) | strong — a peer cannot claim a RuntimeId it lacks the key for |
| **Deployment authorization** | relay-key schnorr attestation on Redis ready records, anchored to *our* relay pubkey (`membership.rs:90-103`) | strong for the registry path |
| **Membership propagation** | scuttlebutt gossip records | **none — unauthenticated** (F-02) |

The intended chain (`wire.rs:52-60`): all pods share one secp256k1
`BUZZ_RELAY_PRIVATE_KEY`, which would collapse the ownership plane if used as the
mesh identity, so each pod generates a fresh ed25519 mesh keypair at boot and the
shared relay key *signs* a binding published to Redis. Peers accept mesh connections
only from endpoint ids present in attested records.

##### Does a peer prove identity? Yes — twice, unevenly.

1. **Cryptographically, at the transport.** QUIC/TLS with ALPN `buzz/mesh/1`
   (`endpoint.rs:35`, `:88`); ALPN re-checked defensively (`peer.rs:50-55`); the
   RuntimeId is *read out of the authenticated connection*, never taken from a frame
   (`peer.rs:58`). `StreamHello.sender` (`wire.rs:161`) is attacker-controlled and
   **is never compared against the authenticated `remote_id`** — verified: the accept
   loop passes `hello` straight to the handler (`runtime.rs:461-464`) without
   cross-checking `hello.sender == remote`. Downstream consumers use the
   `from: RuntimeId` argument (authenticated), so this is currently latent, but the
   contract at `wire.rs:160-163` implies `sender` is meaningful and nothing enforces
   it.
2. **Authorization, at admission.** `is_known_peer` (`runtime.rs:305-323`) requires
   presence in the membership table. Entries reach that table via two paths with
   *asymmetric* verification (F-02).

---

#### 2. Findings

##### F-01 — `GET /_mesh` is unauthenticated and leaks the mesh topology · **HIGH**

Registered on the **health** router (`crates/buzz-relay/src/router.rs:230`), handler
`router.rs:396-406`, reading `AppState::mesh()` (`state.rs:812`) and serialising
`MeshStatus` directly (`router.rs:381`).

The health router is served by a listener bound to a **hard-coded `0.0.0.0`** on
`config.health_port` (default 8080) — `main.rs:1119-1121` — which **ignores
`BUZZ_BIND_ADDR`**. So an operator who binds the app listener to a private
interface still exposes the health listener on all interfaces. The health router
(`router.rs:225-232`) has **no auth layer, no NIP-42, no NIP-98, no operator-pubkey
check** — just four `get` routes.

Disclosed by `MeshStatus` (`status.rs:7-61`):

| Field | Site | Value to an attacker |
|---|---|---|
| `peers[].endpoint_addrs` | `status.rs:20` | **the exact IP:port of every mesh UDP socket in the fleet** — the dial targets for F-03/F-06 |
| `local_runtime_id`, `peers[].runtime_id` | `status.rs:10`, `:19` | full 64-hex ed25519 node keys; the allowlist contents |
| `peer_count`, `draining` | `status.rs:12`, `:11` | replica count and live rollout state (useful for timing an attack at a drain window) |
| `phi`, `connection_state` | `status.rs:24`, `:23` | which pods are currently suspect/partitioned |
| `proto_version`, `record_version`, `last_heartbeat_millis` | `status.rs:21`, `:26`, `:27` | build/version fingerprinting and liveness timing |
| per-peer counters (7 fields) | `status.rs:51-61` | traffic volumetrics per pod pair |

`MeshStatus` derives only `Serialize` (`status.rs:7`), and nothing redacts. Note
`mesh_boot.rs` deliberately keeps `runtime: MeshRuntime` private and exposes only
`status()` (`mesh_boot.rs:169-174`) — the encapsulation is fine; the exposure is the
route's lack of auth.

**Mitigation:** move `/_mesh` off the health listener onto the operator-authenticated
surface (the relay already has `RELAY_OPERATOR_PUBKEYS` + NIP-98 machinery,
`config.rs:~530`), or gate it behind `config.mesh.enabled` **and** operator auth, and
drop `endpoint_addrs` from the default projection.

##### F-02 — Gossip records are unauthenticated, so any admitted peer can inject arbitrary membership · **HIGH**

The two entry paths into `MeshMembership.peers` verify differently:

| Path | Verification |
|---|---|
| `apply_ready_records` (`membership.rs:85-118`) | relay-pubkey anchor match (`:90-92`) **then** schnorr verify (`:103`) |
| `apply_gossip_record` (`membership.rs:120-153`) | **none** — version comparison only |

`MeshStreamFrame::Gossip { payload }` (`wire.rs:144-146`) carries no `FencedHeader`
and no signature. `control_stream_exchange` decodes a `Delta` and applies every
record verbatim (`runtime.rs:534-538`). A record's `runtime_id`, `endpoint_addrs`,
`version`, `draining`, `capabilities`, and `heartbeat_millis` are all
attacker-chosen.

Consequences for a single compromised (or malicious-insider) pod that has been
admitted once:

1. **Admission-list poisoning.** Injecting `GossipRecord::new(attacker_runtime_id,
   …)` makes `has_peer(attacker_runtime_id)` true on every other pod
   (`membership.rs:187-192`), so `is_known_peer` (`runtime.rs:305-307`) admits inbound
   connections from a runtime that **never had a relay-key attestation**. The whole
   point of the attestation (`wire.rs:56-60`, `membership.rs:36-40`) is bypassed
   transitively. Redis is the "only way *into* the mesh" (`registry.rs:3`) only if
   gossip is trusted, and it is not verified.
2. **Outbound dial redirection / SSRF-by-UDP.** `reconcile_once` dials every
   non-draining record (`runtime.rs:296-326`) and `dial_peer` parses
   `endpoint_addrs` and connects (`runtime.rs:328-348`). An attacker sets
   `endpoint_addrs` to arbitrary internal `host:port` values; every pod then sends
   QUIC Initial packets there every 5 s, forever (no eviction, no backoff). The
   handshake will fail against a non-iroh target, but the *packets* reach arbitrary
   internal endpoints, repeatedly, from every pod.
3. **Address hijack of a legitimate peer.** Because the merge is
   version-greater-wins with no signature, a forged record for an existing
   `runtime_id` with `version = u64::MAX - 1` permanently pins that peer's
   `endpoint_addrs` (BR-MESH-24: the version-1 registry seed can never correct it
   again) and can set `draining = true` to make the fleet stop dialing a healthy pod.
4. **Unbounded table growth.** No eviction path exists (verified: no
   `remove`/`retain`/`clear` in `membership.rs`), so injected records are permanent
   for the process lifetime and each carries a `Vec<String>` of arbitrary length.

**Mitigation:** carry the `{relay_pubkey, relay_sig}` attestation *inside*
`GossipRecord` and run the same anchor+verify check in `apply_gossip_record` that
`apply_ready_records` already runs. The attestation preimage
(`registry.rs:85-91`) already binds exactly the right tuple.

##### F-03 — Admission is monotonically permissive; there is no revocation · **MEDIUM-HIGH**

Once a `runtime_id` enters the table it stays forever: `has_peer`
(`membership.rs:187-192`) has no TTL, no attestation re-check, and no eviction.
Registry records expire after 45 s (`registry.rs:187`, BR-MESH-08) but that has **no
effect on the in-memory allowlist**. Concretely:

- A retired/compromised pod's runtime key remains admissible on every surviving pod
  until those pods restart.
- Suspicion (`phi >= 8.0`) does not revoke: `peers()` filters it
  (`membership.rs:366-369`) but `has_peer` does not, and `peers()` has **zero
  production readers** (`-api-surface.md` §6), so suspicion has no security effect at
  all today.
- Draining does not revoke either — it only stops *outbound* dials
  (`runtime.rs:302`).

There is no key-revocation mechanism anywhere in the crate (no CRL, no epoch, no
"attestation valid until"). The attestation preimage (`registry.rs:85-91`) has **no
expiry, no nonce, and no timestamp**, so a captured `ReadyRecord` is valid forever
(see F-05).

##### F-04 — Unauthenticated inbound connections trigger a full Redis scan + N signature verifies · **MEDIUM**

`is_known_peer` (`runtime.rs:305-323`): unknown RuntimeId → `registry.scan_ready()`.
That is a `SCAN`+`GET`-per-key loop (serial, no `MGET`, `registry.rs:217-231`) with
one **secp256k1 schnorr verification per record** (`registry.rs:233-238` →
`registry.rs:70-83`).

Cost to the attacker: generate a fresh ed25519 keypair (microseconds), complete a
QUIC/TLS handshake to `0.0.0.0:3478`. Cost to the pod: one full Redis registry scan
plus N schnorr verifies, then a rejection. Repeat at line rate. Additionally each
rejection emits an unrate-limited `warn!` (`runtime.rs:265-269`) → log-volume
amplification.

Aggravating factors: the accept loop is **single-tasked and serial**
(`runtime.rs:259-283`) — one slow `scan_ready` head-of-line-blocks all legitimate
inbound connections; and the scan shares the relay's main `deadpool_redis` pool
(`mesh_boot.rs:447`), so the amplification lands on the same Redis connections the
event pipeline uses.

**Mitigation:** rate-limit or coalesce the rescan (one in-flight scan, shared
result, with a minimum interval), and cache negative admission decisions.

##### F-05 — Registry records are replayable; the attestation has no freshness binding · **MEDIUM**

Preimage: `buzz-relay-mesh-ready-v1\nruntime_pubkey={hex}\nrelay_pubkey={hex}`
(`registry.rs:85-91`). It covers **only** the two keys — **not** `endpoint_addrs`,
`proto_version`, or `capabilities`, and carries no timestamp/nonce/expiry.

Therefore anyone who can write to Redis (or MITM a non-TLS Redis link) can:

- **Rewrite `endpoint_addrs` on a validly-attested record.** `verify_attestation`
  (`registry.rs:139-146`) checks `runtime_pubkey == runtime_id.to_hex()` and the
  signature over the two keys; the address list is unsigned. This is a full
  redirection primitive against the *registry* path, matching F-02's gossip path.
- **Replay an old record** for a runtime id that has since restarted (its key is
  gone, so no handshake succeeds — the impact is dial churn, not admission) or
  resurrect a drained pod's record.

The record→key binding (BR-MESH-09, `registry.rs:233`) prevents key substitution but
not field tampering. Redis access control is the only real defence, and the crate
receives an externally-configured pool with no TLS assertion of its own.

**Mitigation:** sign the whole record (canonical serialization or add the fields to
the textual preimage) and add an issued-at/expires-at that verifiers check against
the TTL.

##### F-06 — Frame-size and stream-count amplification from an admitted peer · **MEDIUM**

- `MAX_STREAM_FRAME = 16 MiB` (`wire.rs:46`). The receive path reads a 4-byte
  attacker-controlled length and immediately allocates `vec![0u8; len]`
  (`peer.rs:184`) **before** any body has arrived. A peer can force a 16 MiB
  allocation per frame with 4 bytes sent, then stall. The relay's own reliable-stream
  consumer self-limits to 1 MiB chunks (`tunnel/reliable.rs:26-31`) but that is a
  sender-side convention; the receiver honours the full 16 MiB.
- `stream_accept_loop` (`runtime.rs:386-472`) accepts bi-streams in an unbounded
  loop; each session stream is handed to a consumer that spawns a task
  (`mesh_boot.rs:267-277`, `:285-298`). No per-peer stream cap, no concurrency
  semaphore.
- `datagram_recv_loop` (`runtime.rs:358-372`) decodes and dispatches inline with **no
  rate limit and no per-peer quota**.
- Gossip is the one bounded channel (depth 64, drop-on-full, `runtime.rs:46`, `:556`).

Blast radius is limited to already-admitted peers — but per F-02 that set is
attacker-extensible.

##### F-07 — `POST /_mesh/demo/echo` bypasses the host-derived tenant boundary · **MEDIUM** (double-gated)

`crates/buzz-relay/src/api/mesh_demo.rs`, route `router.rs:123`. Requires **both**
`BUZZ_MESH=on` and `BUZZ_MESH_DEMO_ECHO=on`, else 404 (`mesh_demo.rs:10-12`,
`config.rs:516-518`). It takes `community_id: Uuid` **from the request body**
(`mesh_demo.rs:47-48`) — the only relay route that does not derive the tenant from
the HTTP `Host`. It also acquires a real fenced lease and deliberately spawns no
renewer, so the lease lives out its Redis TTL (~30 s) (`mesh_demo.rs:12-16`).

Correctly gated and honestly documented as "not a product flow … the route stays
demo-gated until it is deleted" (`mesh_demo.rs:21-22`). The residual risk is
operator error: `BUZZ_MESH_DEMO_ECHO=on` left set in an environment that also has
mesh on yields an unauthenticated tenant-selecting endpoint. (Verified the gate is
strict-opt-in with the same parser as `BUZZ_MESH`, `config.rs:516-518`.)

##### F-08 — Lock poisoning escalates to fleet-wide mesh failure, including `/_mesh` · **LOW-MEDIUM**

22 `.expect("… lock poisoned")` outside `#[cfg(test)]` (`membership.rs:74,126,159,
173,190,199,322,332,363`; `runtime.rs:142,156,159,168,183,197,202,222,270,349,444,
553,573`). A single panic inside any membership or peer critical section poisons the
`RwLock`, after which **every** mesh operation panics — including
`MeshHandle::status()` (`mesh_boot.rs:173` → `membership.rs:296-313`), which is
called from the axum health handler. An attacker-triggerable panic in a critical
section would therefore turn `/_mesh` into a panicking route on the health listener.
No panic path was found inside those sections (all arithmetic is `saturating_*`,
no indexing, no `unwrap` on user data), so this is a robustness rather than an
exploitable finding.

##### F-09 — Transport error detail is discarded · **LOW**

All 12 iroh error sites flatten to `MeshError::Transport(err.to_string())`
(`endpoint.rs:38,39,79,91,101`; `peer.rs:82,95,113,123,151,155,163,174,187`), and the
five attestation failure causes collapse into the same variant
(`registry.rs:56-83`). Security consequence: a verifier cannot programmatically
distinguish "malformed hex" from "signature forgery attempt," so the two cannot be
alerted on differently. Only the anchor-mismatch case gets a counter
(`foreign_relay_rejections`, `membership.rs:93-94`); signature failures are logged
and forgotten (`membership.rs:104-109`).

---

#### 3. Positive security properties (verified, worth preserving)

- **Off by default, hard no-op.** Two tests pin it: `mesh_off_boots_nothing` uses an
  unroutable Redis URL so any accidental touch fails (`mesh_boot.rs:526-541`), and
  `mesh_defaults_off_when_env_absent` (`mesh_boot.rs:544-555`).
- **Fail-closed when unanchored.** `expected_relay_pubkey: None` rejects every ready
  record (`membership.rs:93`, documented `:36-40`, test `:465-471`) — the unanchored
  state is not accept-any.
- **Authorization checked before signature validity**, deliberately, so "signed by
  some relay key" is never mistaken for authorization (`membership.rs:80-84`, `:90-92`;
  test `ready_records_from_foreign_relay_identity_are_rejected`, `:451-462`).
- **Boot-unique identity** means a restarted pod cannot inherit a dead pod's
  ownership claims (`endpoint.rs:19-21`, `wire.rs:47-50`).
- **The mesh identity is deliberately not the shared deployment key** — reviewer-blocked
  design decision recorded in-source (`wire.rs:52-60`).
- **Transport encryption is mandatory**: QUIC/TLS 1.3 via `tls-ring`
  (`Cargo.toml:68`), mutual node-key authentication, ALPN-pinned. No plaintext mode
  exists; `RelayMode::Disabled` (`endpoint.rs:36`) means no third-party relay ever
  sees the traffic.
- **Media plaintext never exists server-side.** The datagram payload's client frame is
  NIP-44 between client endpoints; only the routing `peer_index` byte is
  relay-visible (`wire.rs:117-121`).
- **No ownership grant anywhere in the crate.** No Redis `SET NX`, `INCR`, `EVAL`, or
  CAS — verified; the only commands are `SET…EX`/`DEL`/`SCAN`/`GET` on
  `mesh:ready:*` (`registry.rs:188-228`). Forged frames cannot cause takeover, only
  rejection, because the fence lives in Redis (`tunnel/directory.rs`).
- **Self-record immutability by peers.** `apply_gossip_record` and
  `apply_ready_records` both drop records about self (`membership.rs:121-123`, `:87-89`),
  so no peer can rewrite our advertised addresses or set our draining flag.
- **Key/record binding in Redis** blocks key substitution (`registry.rs:233`) and
  `runtime_pubkey`/`runtime_id` cross-check blocks record splicing
  (`registry.rs:140-145`).
- **Loud, non-truncating size rejection** for frames and datagrams
  (`peer.rs:142-147`, `:178-183`, `lib.rs:218-223`).
- **One malformed registry entry cannot deny bootstrap** (`registry.rs:233-247`).
- **Zero `unsafe`**, zero `unwrap()` in production paths (only lock-poison
  `expect`s).
- **No secrets logged.** The foreign-relay warn logs the public key and an `anchored`
  boolean only (`membership.rs:96-101`); the private mesh key never leaves
  `iroh::SecretKey`.

---

#### 4. Forged-input resistance summary

| Attack | Outcome | Why |
|---|---|---|
| Forge a `RuntimeId` in a `StreamHello` | ineffective for routing — consumers use the authenticated `from` | `peer.rs:58`, `runtime.rs:461`; but `hello.sender` is never validated (§1) |
| Forge a `FencedHeader` generation | rejected by the Redis lease at the consumer | `tunnel/directory.rs:378,395,413,430` |
| Forge a `ReadyRecord` without the relay key | rejected — anchor + schnorr | `membership.rs:90-103`, `registry.rs:233-238` |
| Tamper `endpoint_addrs` on a valid `ReadyRecord` | **succeeds** — field not covered by the signature | F-05 |
| Forge a `GossipRecord` (any field, any runtime id) | **succeeds** — no verification at all | F-02 |
| Connect with a random keypair | rejected at admission, but costs a full Redis scan + N verifies | F-04 |
| Replay an old `ReadyRecord` | accepted (no freshness) — impact is dial churn / stale allowlist | F-05, F-03 |
| Replay a captured mesh frame | prevented by QUIC/TLS (per-connection AEAD + packet numbers); `seq` is not a replay guard (`wire.rs:114-116`) | — |
| Poison Redis with millions of `mesh:ready:*` keys | amplifies every scan; requires Redis write access | F-04, F-05 |
| Read `GET /_mesh` | **succeeds unauthenticated** — full topology + dial targets | F-01 |


## Module: buzz-conformance (`crates/buzz-conformance`)
### Aspect: Security

The crate is a **detection** mechanism, not an enforcement one. It never influences a request
outcome: `Tracer::record` returns `()` (`src/lib.rs:317`), no relay code branches on a
`TraceStep`, and `check_trace` has no production caller. `crates/buzz-conformance/LIMITS.md:82-86`
states the design intent directly — the gate "is observation only — it does not feed back into
the decision."

---

### What the invariants are meant to protect

| Property | Threat it addresses | Rust predicate |
|---|---|---|
| Tenant non-interference | a request bound to community A observing a row labelled B | `check_row_labels` (`src/transitions.rs:294-312`) |
| Confused-deputy on the host axis | an A-host connection driving a B-channel authorization | `AuthCheck` Allow + foreign claim → `IllegalTransition` (`src/transitions.rs:228-250`) |
| Tenant-context stability | a handler swapping tenant mid-request or emitting from a non-`TenantContext` scope | three `StateMismatch` guards (`src/transitions.rs:143-169`) |
| Error-channel sanitization | tenant-derived information leaking through error strings | 3-variant closed enum (`src/lib.rs:132-140`) + exhaustive relay mapping (`crates/buzz-relay/src/conformance/mod.rs:422-430`) |
| Instrumentation completeness | a seam silently losing its trace (regression hiding a leak) | `ImplBug` → `CoverageBreach` (`src/transitions.rs:277-279`) |

The TLA+ statement of the first is
`Inv_NonInterference == \A o \in observations : o.labels \subseteq {o.community}`
(`docs/spec/MultiTenantRelay.tla:985-986`) — described in the spec comment as "the single-run
label/taint encoding of the two-execution non-interference theorem" (`:982-984`).

---

### Secret handling in the trace payload

The schema is designed around not carrying secrets, and the implementation mostly holds:

| Field | Value on the wire | Producer |
|---|---|---|
| `resolved_community` | community UUID | `conformance/mod.rs:57` — `tenant.community().as_uuid()` |
| `bound_host` | **raw hostname string** | `conformance/mod.rs:58` — `tenant.host().to_string()` |
| `actor` | first 16 hex chars of the authed pubkey | `conformance/mod.rs:70-75` |
| `msg_id` | first 8 bytes of the event id, hex | `conformance/mod.rs:78-85` |
| `channel` | channel UUID, direct | `conformance/mod.rs:89-91` |

No payload content, signature, private key, NIP-98 token, or wall-clock timestamp is projected
— confirmed by reading `AbstractState` (`src/lib.rs:150-175`) and all nine `TraceAction`
variants (`src/lib.rs:179-261`).

Three doc/implementation mismatches are relevant to a security reader (all catalogued in the
data-model doc): `ActorLabel`'s doc claims `blake3(pubkey)` (`src/lib.rs:110`) where the code
takes a raw pubkey prefix; `HostLabel`'s doc claims a registry-mediated opaque label
(`src/lib.rs:96-98`) where the code stores the raw host; `OpaqueId`'s doc claims a hash
(`src/lib.rs:88-91`) where the code takes an event-id prefix. The relay-side comments are
honest about the pubkey case and give the reasoning — the pubkey is already a public
Schnorr X-only key and full hexes appear in relay logs elsewhere
(`conformance/mod.rs:64-70`, `:76-77`).

The one field that is *not* covered by that reasoning is `bound_host`: it is a plaintext
hostname, and it is the field that reveals which tenant a trace belongs to. In a multi-tenant
deployment a JSONL trace file is therefore a cross-tenant correlation artifact. `JsonlTracer`
writes it to a caller-supplied path with default permissions (`tracers.rs:37-43`) and no
redaction hook.

**Production fence preserved.** The crate deliberately does not add `Serialize`/`From<Uuid>` to
`buzz_core::CommunityId`, so that type's "cannot be conjured from client input" property is
untouched; the reasoning is at `Cargo.toml:9-14` and `src/lib.rs:47-58`. Instead the crate owns
a parallel `CommunityLabel` with a public `from_uuid` (`src/lib.rs:74`) — safe because
`CommunityLabel` never feeds a decision. `LIMITS.md:70-73` correctly notes the gate cannot
detect someone *removing* that fence.

---

### Consequence of `NoopTracer` in production

`AppState` binds `Arc::new(crate::conformance::NoopTracer)` at
`crates/buzz-relay/src/state.rs:798` and nothing overwrites it — the field is assigned exactly
once repo-wide. `NoopTracer::record` has an empty body (`tracers.rs:18-20`). So in every
deployed configuration:

- Every `TraceStep` is constructed, then discarded.
- Every `EmitGuard` `ImplBug` on drop is constructed, then discarded
  (`conformance/mod.rs:404-414`).
- Every `RowCommunityProjection::MissingLookup` becomes an `ImplBug` that is discarded
  (`conformance/mod.rs:284-296`, `:319-331`).
- No file, log line, metric, or counter records any of it — `Tracer` is the only sink, and
  there is no `tracing::warn!` fallback on the `ImplBug` paths.

**What still runs anyway** (real work performed for a discarded result):

| Work | Site | Per |
|---|---|---|
| `state_for_request` — 2 `to_string()`/hex allocations + an `AbstractState` clone | `conformance/mod.rs:55-61`; called at `ingest.rs:1407`, `:1801`, `:2348`, `:2485`, `:2194`, `:145` | every ingest, plus once per write/auth emit |
| `EmitGuard::arm` — 2 `Arc` allocations + an `AtomicUsize` | `conformance/mod.rs:383-400`, armed at `ingest.rs:1408` | every ingest |
| `CountingTracer::record` indirection on every emit | `conformance/mod.rs:367-373` | every emit |
| `claimed_community_from_event` — tag scan + UUID parse | `conformance/mod.rs:101-119`; `ingest.rs:143`, `:1788`, `:2193`, `:2334`, `:2468` | every channel-bearing write |
| `msg_id_label` — 8-byte hex format | `conformance/mod.rs:78-85` | every write emit |
| **`db.communities_of_channels` — an extra Postgres round-trip** | `handlers/req.rs:345` and `:661` | **every REQ filter and every search page** |
| `BTreeSet` build over distinct row channel ids | `handlers/req.rs:337-344`, `:648-660` | same |
| per-row label `Vec` allocation | `conformance/mod.rs:238-260` | same |

The DB round-trip is the material one, and it is **not** gated on the tracer type. The gate is
`if let Some(state_snap) = trace_state` (`handlers/req.rs:334`, `:649`), and `trace_state` is
`Some` whenever the authenticated pubkey bytes parse (`handlers/req.rs:116-118`, comment
`:112-115` — "on the hot read path this is always `Some`"). So an authenticated REQ pays one
extra query per filter to build a trace step that `NoopTracer` throws away. On the failure
branch the code substitutes an empty map and logs `warn!` (`:347-353`, `:663-669`), so a
`communities_of_channels` outage produces log noise on a purely observational path.

**Security posture summary:** the invariants above protect nothing at runtime today. They are
assertions that hold over test-constructed traces. Any tenant-isolation regression in the
relay would be caught by this crate only if someone (a) bound `JsonlTracer`, (b) exercised the
regressing path, and (c) fed the resulting file to `check_trace` — and step (c) has no
implementation in the repo (grep: `check_trace` appears in no non-test file).

---

### Uninstrumented paths (blind spots even with a real tracer bound)

Seven `Ok`-returning paths through `ingest_event_inner` emit nothing, so the guard converts
each into an `ImplBug`/`CoverageBreach` rather than a modeled write action:

| Path | Line | Kinds affected |
|---|---|---|
| command routing → `handle_command` | `ingest.rs:1560-1562` | 30620, 41010, 41011, 41012, 46020, 46030, 46031 (`buzz-core/src/kind.rs:743-754`) |
| NIP-56 report | `ingest.rs:1587-1596` | 1984 (`buzz-core/src/kind.rs:267`) |
| moderation commands | `ingest.rs:1605-1614` | 5 kinds (`buzz-core/src/kind.rs:316-325`) |
| relay-admin | `ingest.rs:1834-1842` | 4 kinds (`buzz-core/src/kind.rs:723-731`) |
| NIP-43 leave | `ingest.rs:1846-1928` | 28936 (`buzz-core/src/kind.rs:344`) |
| reaction duplicate | `ingest.rs:2341-2347` | 7 |
| stored-event duplicate | `ingest.rs:2452-2458` | all stored kinds |

Security relevance: the command lane (`handle_command`) and the moderation lane both perform
community-scoped authorization and durable mutation, and neither surfaces an `AuthCheck` or a
`Write*` observation. A tenant-boundary bug inside `command_executor::handle_command` or
`moderation_commands::handle_moderation_command` is structurally invisible to this gate — it
would present as a `CoverageBreach`, identical to a legitimate command.

Additional structural blind spots:

- **REQ arms no guard.** `handle_req` builds `trace_state` (`req.rs:116-118`) but never calls
  `EmitGuard::arm`. A REQ that closes early — the three p-gate rejections at `req.rs:184-189`,
  `:191-196`, `:198-203`, or the `query_events` failure at `:321-325` — emits nothing and
  produces no `ImplBug`, so read-path emit loss is undetectable. The read path also emits no
  `SanitizedError` for any of those rejections: the only `SanitizedError` producer in the relay
  is the ingest wrapper (`ingest.rs:1436-1444`).
- **`AuthCheck` is skipped for six kinds** by `skip_membership`
  (`ingest.rs:1796-1802`: `KIND_NIP29_JOIN_REQUEST`, `KIND_NIP29_CREATE_GROUP`,
  `KIND_STREAM_MESSAGE_EDIT`, `KIND_NIP29_EDIT_METADATA`, `KIND_NIP29_DELETE_EVENT`,
  `KIND_NIP29_DELETE_GROUP`) — no membership check runs, so no verdict is traced.
- **`ReadHostFeedRows` has no producer.** Grep across `crates/buzz-relay/` finds none, so the
  kinds-only host-feed read surface (`docs/spec/MultiTenantRelay.tla:703`) is untraced.
- **The M2/M8 rule is inert on the read path** because `record_req_authcheck` hard-wires
  `claimed_community: None` (`conformance/mod.rs:152`); the checker's guard requires
  `Some(c)` (`src/transitions.rs:233`).
- **Row-label confinement is the only leak detector, and it needs a same-request read.** A
  write that persisted under the wrong community produces an in-spec trace: all three write
  arms return `Ok(())` unconditionally (`src/transitions.rs:187`, `:191`, `:198`).
- **`Inv_NoTenantContextFailsClosed`** (`docs/spec/MultiTenantRelay.tla:1116-1118`) — the
  "missing TenantContext serves no rows" backstop — has no Rust predicate; an empty
  `row_communities` passes `check_row_labels` vacuously.

`LIMITS.md:47-79` documents several of these classes honestly (DB-layer leaks the projection
doesn't read, cross-pod leaks, timing, fan-out, type-fence removal, spec bugs), but it also
says "the read-seam half of the gate is **not yet armed**" (`:56-57`) — stale, since
`handlers/req.rs:334-361` and `:649-677` now emit.

