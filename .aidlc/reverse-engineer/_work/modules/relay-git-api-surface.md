## Module: buzz-relay — git hosting (`crates/buzz-relay/src/api/git`)
### Aspect: API Surface

#### 1. HTTP endpoints — exact count: 4

Three git smart-HTTP routes (`transport.rs:1756-1765`) plus one internal policy route (`mod.rs:60-66`). Both routers are merged into the single relay router at `router.rs:48-50`, `:137-138`.

| # | Method + path | Handler | Auth | Body limit | Response CT |
|---|---|---|---|---|---|
| 1 | `GET /git/{owner}/{repo}/info/refs?service=…` | `info_refs` (`transport.rs:539`) | `GitAuth` extractor | `git_max_pack_bytes` | `application/x-git-{service}-advertisement` (`:568-571`, `:717`) |
| 2 | `POST /git/{owner}/{repo}/git-upload-pack` | `upload_pack` (`transport.rs:786`) | `GitAuth` | `git_max_pack_bytes` | `application/x-git-upload-pack-result` (`:825`) |
| 3 | `POST /git/{owner}/{repo}/git-receive-pack` | `receive_pack` (`transport.rs:858`) | `GitAuth` | `git_max_pack_bytes` | `application/x-git-receive-pack-result` (`:1500-1503`) |
| 4 | `POST /internal/git/policy` | `policy::hook_policy_check` (`policy.rs:173`) | HMAC-SHA256 in body + `require_localhost` middleware | **1 MiB** (`mod.rs:63`) | `application/json` |

Both body limits come from `tower_http::limit::RequestBodyLimitLayer`. The git limit is read once at router-construction time and cast: `state.config.git_max_pack_bytes as usize` (`transport.rs:1757`). No layer other than the body limit is applied inside `git_router`; the shared `track_metrics` / `TraceLayer` / CORS layers are applied over the merged router (`router.rs:187-190`).

##### Route-shape notes

- The git routes use axum 0.8 `{param}` capture syntax, so `{repo}` cannot contain `/`. `owner`/`repo` are re-validated by `validate_repo_id` in every handler (`transport.rs:266-297`, called at `:549`, `:788`, `:860`).
- `info/refs` is the only route with a required query parameter; `InfoRefsQuery { service: String }` (`transport.rs:358-361`) is a mandatory field, so a missing `?service=` is a 400 from axum's `Query` rejection before the handler body runs.
- `git_router` does **not** register `HEAD`, `OPTIONS`, or the dumb-HTTP paths (`objects/info/packs`, `objects/<2>/<38>`). Dumb HTTP is entirely absent.

#### 2. Request/response framing

##### `info/refs`

Two code paths produce byte-different-but-protocol-equivalent output.

| Path | Trigger | Body layout | Permit | Subprocess |
|---|---|---|---|---|
| Fast path | `service == git-upload-pack` **and** `fast_path_eligible(manifest)` (`transport.rs:552-577`) | `pkt(# service=git-upload-pack\n)` `0000` `pkt(<oid> HEAD\0<caps>\n)` `pkt(<oid> <ref>\n)`× `0000` (`transport.rs:497-537`) | **none acquired** | none |
| Subprocess | receive-pack advertisement, or upload-pack on a repo with any `refs/tags/*` | `pkt(# service=<svc>\n)` `0000` ‖ raw `git <svc> --stateless-rpc --advertise-refs` stdout (`transport.rs:712-721`) | yes (`:598`) | yes (`:645-653`) |

Fast-path capability offer is a fixed string (`transport.rs:484-494`): `multi_ack thin-pack side-band side-band-64k ofs-delta shallow deepen-since deepen-not deepen-relative no-progress include-tag multi_ack_detailed no-done symref=HEAD:<head> object-format=<sha1|sha256> agent=buzz-git`. `object-format` is derived from the HEAD oid width (`transport.rs:473-479`), which is the only place SHA-256 is inferred rather than rejected.

pkt-line encoding is centralized in `pkt_line` (`transport.rs:437-454`). It refuses payloads > `0xffff - 4` by emitting an empty `0004` frame and logging at `error` — a deliberately non-panicking degradation (pinned `transport.rs:2231-2237`).

Both paths set `Cache-Control: no-cache` (`transport.rs:566`, `:719`).

##### `git-upload-pack`

- Request body: pkt-line want/have negotiation, optionally `Content-Encoding: gzip`. Transparently inflated by `decode_git_request_body` (`transport.rs:745-784`) with a **decoded** cap of `UPLOAD_PACK_MAX_DECODED_BYTES` = 64 MiB (`transport.rs:59`, applied `:789`).
- Response: **streamed**. `stream_git_read` (`transport.rs:1414-1498`) pipes child stdout into `Body::from_stream` through three wrappers: `TimedByteStream` (300 s deadline, byte/duration histograms — `:1282-1390`), `StreamingGit` (owns `Child` + `HydratedRepo` + stdin pump; kills the child on timeout — `:1262-1332`), `GitPermitStream` (holds the semaphore permit to EOF — `:1293-1310`, pinned `:1924-1941`).
- Status is committed to 200 before any pack byte exists, so post-head errors surface in-band per git's smart-HTTP contract (`transport.rs:1405-1412`).
- `extra_args` is `&[]` at the only call site (`transport.rs:824`) — the parameter is unused in production.

##### `git-receive-pack`

- Request body decoded with a cap of `git_max_pack_bytes` (`transport.rs:861`) — i.e. the *same* number bounds compressed (router layer) and decoded (decode seam) bytes.
- Response: **buffered**, capped at `RECEIVE_PACK_MAX_OUTPUT_BYTES` = 1 MiB (`transport.rs:50`, enforced `:1092-1108`). Buffering is the structural fence: `run_git_at` returns an owned `PackOutput` (`transport.rs:971-991`), never a `Response`, so `finalize_push` can sequence the CAS before a 2xx exists (`transport.rs:1540-1547`).

##### `/internal/git/policy`

Request JSON is `HookCallbackRequest` (`policy.rs:53-86`); response is `HookCallbackResponse` (`policy.rs:87-97`) with 200 on allow and 403 on every denial or error. Deny bodies are sometimes JSON (`policy.rs:407-413`) and sometimes bare text (`:176-233`, `:274-281`, …) — the hook only reads the status code and echoes the body to stderr (`hook.rs:141-146`).

#### 3. Status-code matrix

| Status | Condition | Site |
|---|---|---|
| 200 | all success paths | `transport.rs:566`, `:1487`, `:1501` |
| 400 | invalid `service` value | `transport.rs:548` |
| 400 | `owner` not 64 lowercase hex | `transport.rs:273` |
| 400 | repo name empty / >64 / leading `.` / contains `..` / bad alphabet | `transport.rs:288` |
| 400 | path matches no git endpoint shape (NIP-98 `u` derivation) | `transport.rs:140` |
| 400 | `CasError::ManifestInvalid` | `transport.rs:1626` |
| 401 | missing `Authorization` header (+ `WWW-Authenticate: Nostr realm="buzz", method=…`) | `transport.rs:88-97` |
| 401 | header lacks `Nostr ` prefix | `transport.rs:99-108` |
| 401 | bad base64 / bad utf-8 / unparseable event / NIP-98 verify failure | `transport.rs:110-115`, `:185-190`, `:200-202` |
| 403 | relay membership denied | `transport.rs:221` |
| 403 | every policy-endpoint rejection (structural, HMAC, TTL, missing 30617, archived channel, non-member, unknown role, DB error, protection denial) | `policy.rs:176-413` |
| 403 | non-loopback (or `ConnectInfo`-absent) request to `/internal/git/policy` | `mod.rs:49` |
| 404 | Host does not resolve to a community (`bind_community` error) — deliberately reused as "repository not found" | `transport.rs:130` |
| 404 | pointer absent | `transport.rs:579`, `:619`, `:812` |
| 409 | `CasError::Conflict` (CAS 412) | `transport.rs:1609` |
| 413 | `HydrateError::ResourceLimit` | `transport.rs:344-349` |
| 413 | advertisement stdout > 4 MiB | `transport.rs:691-698` |
| 413 | subprocess stdout > `max_output_bytes` | `transport.rs:1098-1106` |
| 413 | `CasError::ResourceLimit` | `transport.rs:1639` |
| 500 | tempfile/spawn/wait/read failures, non-zero `--advertise-refs`, hook install failure, other `CasError` | `transport.rs:630-700`, `:908`, `:1656` |
| 503 | git semaphore exhausted (+ `Retry-After: 5`) | `transport.rs:326-337` |
| 504 | `INFO_REFS_TIMEOUT` 120 s or `PACK_OPS_TIMEOUT` 300 s exceeded | `transport.rs:667`, `:1076` |

Body-limit overflow (compressed bytes past `git_max_pack_bytes`) is produced by `RequestBodyLimitLayer` itself, not by this module.

#### 4. Public Rust items and their callers

"External" = outside `crates/buzz-relay/src/api/git/`. Verified by workspace grep across `crates/**`.

##### `transport.rs`

| Item | Vis | External callers | Notes |
|---|---|---|---|
| `git_router` | `pub` (re-exported `mod.rs:35`) | `router.rs:48` | 1 |
| `info_refs`, `upload_pack`, `receive_pack` | `pub` | 0 direct — reached via routes `transport.rs:1760-1762` | axum handlers |
| `GitAuth { pubkey, tenant }` | `pub` | 0 | extractor, used by the three handlers |
| `InfoRefsQuery`, `GitRepoParams` | `pub` | 0 | fields are private, so unusable externally |
| `harden_git_env` | `pub(crate)` | 0 outside module; 4 in-module (`cas_publish.rs:290`, `:344`, `:412`, `:534`, `:614`, `:717`, and test `:1534`) | crate-visible but effectively module-private |
| `PackOutput`, `PushContext` | `pub(crate)` | 0 | fence types |

##### `store.rs` — entire module carries `#![allow(dead_code)]` (`store.rs:25`)

| Item | Vis | Callers |
|---|---|---|
| `GitStore::new` | `pub` | `state.rs:694-701` (1 prod), tests |
| `GitStore::content_key` | `pub` | in-module only (`store.rs:260`, probe `:726`) — **zero external callers** |
| `GitStore::idx_key_for_pack_digest` | `pub` | in-module only (`:295`, `:328`) |
| `put_pack` | `pub` | `cas_publish.rs:826`, `:1150`; probe `store.rs:594` |
| `put_manifest` | `pub` | `cas_publish.rs:1194`, `handlers/side_effects.rs:2643` |
| `put_idx` | `pub` | `cas_publish.rs:429`, `:830` |
| `get_idx` | `pub` | `hydrate.rs:389` |
| `get` | `pub` | in-module (`:391`, `:428`) |
| `get_verified` | `pub` | `cas_publish.rs:1315`; probe |
| `get_verified_limited` | `pub` | `hydrate.rs:277` |
| `get_limited` | `pub` | in-module (`:329`, `:391`) |
| `get_pointer` | `pub` | `hydrate.rs:251`, `cas_publish.rs:1297`, `handlers/side_effects.rs:2665`, `:2715` |
| `put_pointer` | `pub` | `cas_publish.rs:1235`, `handlers/side_effects.rs:2653`, probe |
| `run_conformance_probe` | `pub` | `main.rs:492` (fatal gate) |
| `ETag`, `Precond`, `CasOutcome`, `StoreError`, `ProbeConfig` | `pub` | `Precond`/`CasOutcome` used by `side_effects.rs:2622`; `ProbeConfig` by `main.rs:481` |
| `ProbeReport`, `ProbeFailure` | `pub` | in-module only; `ProbeReport` fields read at `main.rs:496-499` |

##### `manifest.rs`

| Item | Vis | Callers |
|---|---|---|
| `Manifest` + `validate` / `canonical_bytes` / `from_bytes` | `pub` | `cas_publish.rs`, `hydrate.rs`, `transport.rs`, `handlers/side_effects.rs:2621-2645` |
| `pointer_key` | `pub` | `cas_publish.rs:1023`, `hydrate.rs:247`, `handlers/side_effects.rs:2650`, `:2712` |
| `MANIFEST_VERSION` | `pub` | `handlers/side_effects.rs:2629` |
| `MAX_MANIFEST_PACKS`, `PACK_COMPACTION_THRESHOLD`, `MAX_MANIFEST_REFS` | `pub` | in-module only (`cas_publish.rs`, `manifest.rs`) — **zero external callers** |
| `is_safe_refname`, `is_hex_oid` | `pub` | `hydrate.rs`, `transport.rs:402-419`, `manifest.rs:210-232` — **zero external callers** |
| `is_pack_key` | `pub` | `manifest.rs:234` only |
| `ManifestError` | `pub` | `cas_publish.rs:139`, `hydrate.rs:101` |

##### `hydrate.rs`

| Item | Vis | Callers |
|---|---|---|
| `hydrate_for_read` | `pub` | `transport.rs:600`, `:792`; test `cas_publish.rs:1786` |
| `hydrate_for_write` | `pub` | `transport.rs:878`; test `cas_publish.rs:1712` |
| `load_manifest_for_read` | `pub` | `transport.rs:553` |
| `HydratedRepo` + `path` / `hydrated_bytes` / `hydrated_packs` | `pub` | `transport.rs`, `cas_publish.rs:1587-1591`; `hydrated_packs` used only by the metric at `hydrate.rs:136` |
| `HydrationOptions`, `HydrateError` | `pub` | `transport.rs`, `pack_cache.rs` |
| `get_verified_limited`, `install_or_generate_idx` | `pub(super)` | `pack_cache.rs:288`, `:290` |

##### `cas_publish.rs`

| Item | Vis | Callers |
|---|---|---|
| `cas_publish` | `pub` | `transport.rs:1582` only |
| `CasError` (7 variants) | `pub` | `transport.rs:1601-1656` |
| `PublishLimits` | `pub` | `transport.rs:1588-1592` |
| `CasSuccess` | `pub` | `transport.rs:1582`, `:1685-1690` |
| `ParentState` + `fresh` / `from_loaded` | `pub` | `hydrate.rs:212`, `:241`; `transport.rs:878` |

##### `manifest_event.rs`

| Item | Vis | Callers |
|---|---|---|
| `build_ref_state_event` | `pub` | `transport.rs:1690`, `handlers/side_effects.rs:2749` |
| `RefStateInputs` | `pub` | `transport.rs:1685`, `handlers/side_effects.rs:2743` |
| `BuildError` | `pub` | returned only; matched in tests (`:369-380`) |

##### `hook.rs` / `policy.rs` / `mod.rs`

| Item | Vis | Callers |
|---|---|---|
| `install_hook` | `pub` | `transport.rs:906` only |
| `hook_policy_check` | `pub` | route `mod.rs:62` |
| `HookCallbackRequest`, `HookRefUpdate`, `HookCallbackResponse`, `DenialResponse` | `pub` | deserialized/serialized by axum; `HookRefUpdate` also used by `generate_hook_hmac` |
| `generate_hook_hmac` | `pub` | **ZERO production callers.** Only `policy.rs:581`, `:626`, `:720` — all `#[cfg(test)]`. Test-only public API. |
| `git_policy_router` | `pub` | `router.rs:50` |
| `require_localhost` | private | `mod.rs:64` |

##### Dead-code summary (zero production callers anywhere)

1. `policy::generate_hook_hmac` (`policy.rs:419-437`) — the *only* item with zero non-test callers.
2. `stream_git_read`'s `extra_args` parameter (`transport.rs:1418`) — always `&[]` (`:824`).
3. `GitStore::content_key` / `idx_key_for_pack_digest` / `get` / `get_limited` are `pub` but reachable only in-module; the module-wide `#![allow(dead_code)]` (`store.rs:25`) suppresses any future warning if the last caller disappears.
4. `MAX_MANIFEST_PACKS`, `PACK_COMPACTION_THRESHOLD`, `MAX_MANIFEST_REFS`, `is_safe_refname`, `is_hex_oid`, `is_pack_key`, `HydratedRepo::hydrated_packs`, `ProbeReport`, `ProbeFailure` — `pub` with no external caller (over-exported surface, not dead).

#### 5. Consumers outside the relay

| Consumer | Surface used | Site |
|---|---|---|
`web/` repo browser | isomorphic-git `clone`/`fetch` with `depth: 1, singleBranch, noTags` against `/git/{owner}/{repo}.git`; NIP-98 header from `makeNip98AuthHeader(<repo-root>.git, "GET")` | `web/src/features/repos/git-client.ts:41-113` |
| `web/` repo list / refs | `POST /query` for kinds 30617 and 30618 — not this module's HTTP surface | `web/src/features/repos/use-repos.ts:66`, `use-repo-refs.ts:54-57` |
| `git-credential-nostr` | supplies the NIP-98 `Authorization` header to the git CLI | `crates/buzz-test-client/tests/e2e_git.rs:63-88` |
| `e2e_git.rs` | drives all three routes through real `git`, and reads the S3 pointer directly | `crates/buzz-test-client/tests/e2e_git.rs:195-475` |

#### 6. Delta: the NIP-98 `u` tag is repo-root, `.git`-sensitive

`git_expected_url` (`transport.rs:235-253`) derives the expected `u` by stripping the endpoint suffix, keeping whatever the client sent — **including `.git`**. So a token signed for `…/git/o/r` will not authenticate a request to `…/git/o/r.git` and vice versa, even though both address the same pointer (`manifest.rs:182`). The web client deliberately signs the `.git` form (`web/src/features/repos/git-client.ts:41-50`). The scheme comes from `config.relay_url` (`wss://` ⇒ `https`, else `http`) but the **host always comes from the resolved tenant**, pinned by `transport.rs:2067-2141`.

The `/info/refs` branch uses `split_once("/info/refs")` (`transport.rs:243-244`), so `…/info/refs/anything` also derives a valid prefix; harmless because the route itself would not match.
