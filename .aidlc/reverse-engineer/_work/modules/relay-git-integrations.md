## Module: buzz-relay — git hosting (`crates/buzz-relay/src/api/git`)
### Aspect: Integrations

#### 1. Integration inventory

| Dependency | Direction | Surface | Criticality |
|---|---|---|---|
| S3 / MinIO (`rust-s3` 0.37) | out | 4 distinct bucket operations | **hard** — source of truth; startup probe is fatal |
| `git` binary (subprocess) | out | 10 production invocation sites, 9 distinct subcommands | **hard** — no in-process git |
| `buzz-db` (Postgres) | out | 4 read methods + 1 write | hard for push authz; the write is best-effort |
| `buzz-core` | in-proc | `TenantContext`, `CommunityId`, `MemberRole`, `git_perms::*` | hard |
| `buzz-auth` | in-proc | `nip98::verify_nip98_event` | hard |
| Relay event fan-out | out | `fan_out_event_to_local_subscribers` (bypasses `dispatch_persistent_event`) | best-effort |
| Local HTTP loopback | out+in | pre-receive hook → `POST /internal/git/policy` | **hard** — fail-closed |
| Local filesystem | out | tempdirs, tempfiles, pack cache | hard |
| `metrics` crate | out | 8 counters, 11 histograms, 3 gauges | observability |
| Runtime image | build | `curl` + `openssl` must be installed for the hook | **hard**, asserted by a test |

#### 2. Object store (every call)

Client construction — `store.rs:187-219`:
- `Region::Custom { region, endpoint }`, `Bucket::new(...).with_path_style()` (`store.rs:206-214`). Path-style is unconditional (MinIO compatibility; AWS accepts both).
- Credential selection (`store.rs:198-210`): both keys non-empty ⇒ `Credentials::new`; **both empty** ⇒ `Credentials::default()` (AWS chain: env, profile, IRSA web-identity, container, IMDS); exactly one empty ⇒ `StoreError::Config` (pinned `store.rs:961-982`).
- **Credentials and bucket are shared with `buzz-media`**: `state.rs:694-701` passes `config.media.s3_{endpoint,access_key,secret_key,bucket,region}` and `.expect("media storage was already constructed with this S3 config")`. There is no separate git bucket at runtime, despite `BUZZ_GIT_S3_*` appearing in probe/test helpers (`cas_publish.rs:1580-1601`).

| Bucket op | Site | Purpose | Headers / notes |
|---|---|---|---|
| `put_object_with_content_type_and_headers` | `store.rs:262-268` (`put_immutable`) | write `packs/<sha256>`, `manifests/<sha256>` | `If-None-Match: *`; 2xx ⇒ key, 412 ⇒ key (idempotent), other ⇒ `Backend` |
| `put_object_with_content_type_and_headers` | `store.rs:297-306` (`put_idx`) | write `idx/<pack-digest>` | `If-None-Match: *`; same 412 handling |
| `put_object_with_content_type_and_headers` | `store.rs:489-501` (`put_pointer`) | the CAS | `If-Match: <etag>` or `If-None-Match: *`; content type `application/json`; result routed through `classify_cas` (`:516-546`) |
| `put_object_with_content_type_and_headers` | `store.rs:886-899` (`put_immutable_raw`) | probe phase 3 only — returns the raw status so 412s are countable | `If-None-Match: *` |
| `get_object` | `store.rs:348-352` (`get`) | fetch any object; 404 ⇒ `NotFound` | no range requests anywhere — full-body GET |
| `get_object` | `store.rs:456-478` (`get_pointer`) | atomic `(ETag, body)` snapshot | fails if no `etag`/`ETag` header |
| `head_object` | `store.rs:407-425` (`get_limited`) | size pre-check before download | followed by a post-read length re-check (`:429-437`) |
| `delete_object` | `store.rs:618`, `:728`, `:871` | probe scratch cleanup only | **the protocol never deletes packs/manifests/pointers** (doc §Axioms A1 no-deletion rule) |

Read/write call graph:

```
receive_pack → hydrate_for_write → get_pointer, get_verified_limited(manifest),
                                    [pack_cache] get_verified_limited(pack), get_idx
             → cas_publish       → put_pack, put_idx, put_manifest, put_pointer
                                    [on LostRace] get_pointer, get_verified
info_refs fast → load_manifest_for_read → get_pointer, get_verified_limited
info_refs slow / upload_pack → hydrate_for_read → same as hydrate_for_write minus ParentState
announce (side_effects) → put_manifest, put_pointer, get_pointer
startup → run_conformance_probe → put_pack, get_verified, put_pointer, get_pointer,
                                   put_immutable_raw, delete_object
```

Note: `store.rs:25` carries a module-wide `#![allow(dead_code)]` whose comment ("wired in by the push path in a follow-up commit") is stale.

#### 3. `git` subprocess invocations

Exactly **10** production `Command::new("git")` sites (3 in `transport.rs`, 6 in `cas_publish.rs`, 1 shared helper in `hydrate.rs`). All go through `harden_git_env` except `hydrate::run_git`, which hand-rolls a *different* env.

##### Environment

`harden_git_env` (`transport.rs:294-310`):

| Var | Value |
|---|---|
| — | `env_clear()` first |
| `PATH` | inherited (`std::env::var("PATH").unwrap_or_default()` — empty string if unset) |
| `GIT_HTTP_EXPORT_ALL` | `1` |
| `GIT_CONFIG_NOSYSTEM` | `1` |
| `GIT_CONFIG_GLOBAL` | `/dev/null` |
| `HOME` | `/dev/null` |

`hydrate::run_git` (`hydrate.rs:451-465`): `env_clear()`, `PATH` (only if set), `GIT_CONFIG_NOSYSTEM=1`, `HOME=<cwd>`. **Missing `GIT_CONFIG_GLOBAL`** — its doc comment claims it "matches transport.rs's harden_git_env semantics" (`hydrate.rs:456-457`), which is inaccurate; the `HOME=<cwd>` trick is what makes the global lookup miss.

`receive_pack` adds, on top of `harden_git_env` (`transport.rs:917-931`):

| Var | Value |
|---|---|
| `BUZZ_HOOK_URL` | `http://127.0.0.1:<config.bind_addr.port()>/internal/git/policy` |
| `BUZZ_HOOK_SECRET` | `config.git_hook_hmac_secret` |
| `BUZZ_REPO_ID` | stripped repo id |
| `BUZZ_REPO_OWNER` | owner hex from the URL |
| `BUZZ_COMMUNITY_ID` | resolved community UUID |
| `BUZZ_PUSHER_PUBKEY` | authenticated pusher hex |
| `GIT_CONFIG_COUNT` | `1` |
| `GIT_CONFIG_KEY_0` | `core.hooksPath` |
| `GIT_CONFIG_VALUE_0` | `<workspace>/hooks` |

##### Invocation table

| # | Site | argv | cwd | stdin | stdout | stderr | Timeout | kill_on_drop |
|---|---|---|---|---|---|---|---|---|
| 1 | `transport.rs:645-653` | `git {upload-pack\|receive-pack} --stateless-rpc --advertise-refs <workspace>` | inherited | inherited | `NamedTempFile` in `git_repo_path` | tempfile (64 KiB prefix logged) | 120 s (`:660`) | yes |
| 2 | `transport.rs:1019-1027` | `git receive-pack --stateless-rpc <workspace>` | inherited | **piped** — request body pumped by a spawned task (`:1037-1064`) | tempfile | tempfile | 300 s (`:1067`), body task aborted on timeout | yes |
| 3 | `transport.rs:1423-1434` | `git upload-pack --stateless-rpc <workspace>` (`extra_args` always empty) | inherited | **piped** — body pumped by a detached task (`:1442-1467`) | **piped → HTTP body** | `null` | 300 s in-band deadline (`:1471`), child killed (`:1315-1331`) | yes |
| 4 | `cas_publish.rs:284-296` | `git for-each-ref --format=%(refname) %(objectname)` | workspace | inherited | tempfile in scratch | tempfile | none | no |
| 5 | `cas_publish.rs:337-345` | `git symbolic-ref --quiet HEAD` | workspace | inherited | in-memory (`.output()`) | in-memory | none | no |
| 6 | `cas_publish.rs:409-415` | `git index-pack <tmp>/pack-<digest>.pack` | private tempdir | inherited | in-memory | in-memory | none | no |
| 7 | `cas_publish.rs:524-541` | `git pack-objects --revs --stdout -q --threads=1 --window 10 --window-memory=67108864` | workspace | piped (rev-spec lines) | tempfile in scratch | tempfile | 300 s (`:433-460`) | yes |
| 8 | `cas_publish.rs:608-620` | `git count-objects -v` | workspace | piped (empty) | tempfile | tempfile | 300 s | yes |
| 9 | `cas_publish.rs:707-729` | `git pack-objects --revs -q --threads=1 --window 10 --window-memory=67108864 --max-pack-size=<n> <tmp>/compact` | workspace | piped (deduped tips) | tempfile in compaction tempdir | tempfile | 300 s inner, 600 s outer (`:1058`) | yes |
| 10 | `hydrate.rs:451-470` (`run_git`, 4 arg sets) | `git init --bare --quiet` (`:182`); `git symbolic-ref HEAD refs/heads/main` (`:183`); `git verify-pack <idx>` (`:393`); `git index-pack <pack>` (`:420`) | caller-supplied | inherited | in-memory | in-memory | **none** | yes |

Observations:

- Sites 4, 5, 6, 10 have **no timeout**. Site 10 covers `index-pack` on an attacker-influenced pack (through the pack cache), so a pathological pack can occupy a semaphore permit for an unbounded time.
- Sites 1, 4, 5, 6, 10 do not set `stdin`, so the child inherits the relay's stdin.
- All repo paths are passed as a **single argv element** (`Command::arg`, no shell), so `--stateless-rpc <path>` cannot be word-split. Paths are OS-generated tempdir paths, never user strings.
- `--threads=1` and `--window-memory` bound one `pack-objects`; total CPU is bounded only by `git_semaphore` (20) and the compaction semaphore (1).
- No `git gc`, `repack`, `fsck`, `prune`, or `update-ref` is ever invoked. Refs are written as loose files directly (`hydrate.rs:355-371`).
- Tests additionally shell out to `bash` twice (`policy.rs:682`, `:757`) to cross-verify the hook's HMAC against the Rust implementation.

#### 4. `buzz-db` (Postgres)

| Call | Site | Purpose |
|---|---|---|
| `query_events(EventQuery{ kinds:[30617], pubkey:owner, d_tag:repo_id, global_only:true, limit:1, ..for_community(community) })` | `policy.rs:254-284` | resolve the repo announcement for protection rules + channel binding. Direct DB query, so it bypasses the relay's `kinds`-required p-gate. |
| `get_channel(community, channel_id)` | `policy.rs:308-330` | archived-channel read-only check |
| `is_agent_owner(community, owner_bytes, pusher_bytes)` | `policy.rs:330-368` | NIP-OA managed-agent owner authority |
| `get_member_role(community, channel_id, pusher_bytes)` | `policy.rs:353-395` | channel role for non-owners |
| `insert_event(community, kind:30618, None)` | `transport.rs:1693-1700` | persist the derived ref-state event; `(_, false)` means DB dedup |
| `crate::tenant::bind_community(&state.db, raw_host)` | `transport.rs:128-130` | host → community (uses `db` as `HostResolver`) |

Every failure in the first four ⇒ 403 (fail-closed, `policy.rs:277-282`, `:322-326`, `:355-362`, `:388-392`). Failure of `insert_event` ⇒ warning only (`transport.rs:1727-1735`).

Not used by this module but part of the same feature: `git_repo_names` reservation and quota (`repo_name_owner`, `count_repos_for_owner`, `reserve_repo_name`, `release_repo_name`) live in `handlers/side_effects.rs:2463-2560`.

#### 5. `buzz-pubsub`

**Not used directly.** kind:30618 is fanned out through `crate::handlers::event::fan_out_event_to_local_subscribers` (`transport.rs:1701-1710`, and `handlers/side_effects.rs:2755-2761` for the announce path) — the *local* subscriber path only. It bypasses `dispatch_persistent_event`, so whatever cross-pod Redis publication that function performs does **not** happen for git ref-state events: a subscriber connected to a different pod will not see the 30618 in real time and must re-query. The code comments justify the bypass only in terms of the access gate no-op for `channel_id = None`.

#### 6. Relay event emission

| Event | Trigger | Signer | Path |
|---|---|---|---|
| kind:30618 (post-push) | successful CAS with `parent_digest != committed_digest` | relay keypair (`state.relay_keypair`) | `transport.rs:1662-1746` |
| kind:30618 (announce seed) | fresh kind:30617 reservation | relay keypair | `handlers/side_effects.rs:2728-2765` |

Tags: `["d", repo_id]`, one `[<refname>, <oid>]` per emittable ref, `["HEAD", "ref: <head>"]`, `["p", <actor-hex>]` (`manifest_event.rs:74-108`). Content is empty. Only `refs/heads/*` and `refs/tags/*` are emitted (`manifest_event.rs:117-127`), and refs with non-40/64-hex oids or malformed names are skipped silently (`:82-93`).

#### 7. The HMAC hook callback loop

```
receive_pack (transport.rs:858)
  └─ install_hook → <workspace>/hooks/pre-receive, 0o755        hook.rs:152-178
  └─ git receive-pack --stateless-rpc <workspace>               transport.rs:1019
        └─ pre-receive (bash)                                    hook.rs:32-150
             ├─ read "old new ref" lines from stdin              hook.rs:56
             ├─ git merge-base --is-ancestor old new             hook.rs:59-70  (quarantine env inherited)
             ├─ HMAC-SHA256 via `openssl dgst -sha256 -hmac`     hook.rs:118
             └─ curl --silent --max-time 10 -X POST $BUZZ_HOOK_URL
                                                                  hook.rs:129-139
                   └─ POST /internal/git/policy                   mod.rs:62 → policy.rs:173
                         ├─ require_localhost middleware          mod.rs:41-52
                         ├─ structural validation                  policy.rs:176-234
                         ├─ verify_hmac (constant-time)            policy.rs:159-171, :236-241
                         ├─ 30s TTL / 5s future skew               policy.rs:243-256
                         ├─ 4 buzz-db lookups                      policy.rs:254-395
                         └─ buzz_core::git_perms::evaluate_push    policy.rs:404-416
             └─ non-200 ⇒ echo body to stderr, exit 1 (deny)     hook.rs:141-148
```

Cross-boundary contract details:

- The bash side sets `LC_ALL=C` so `sort` order and `${#var}` byte lengths match Rust's byte comparison and `String::len()` (`hook.rs:36-38`).
- Refs are sorted by `ref_name` on both sides — bash `sort` on a `ref_name`-first line, Rust `sort_by_key(|r| r.ref_name.clone())` (`hook.rs:113-121`, `policy.rs:143-146`). Order-independence is pinned by `policy.rs:559-576`.
- The pre-image format agreement is verified by **two** tests that actually run `bash` + `openssl` and compare against `generate_hook_hmac`: `policy.rs:592-703` (two refs, unsorted input) and `:704-773` (single ref). These are the only tests of the module's most security-critical contract.
- `set -eo pipefail` plus `: "${VAR:?…}"` guards make a missing env var abort the hook (`hook.rs:35`, `:41-47`).
- Both `ref_name` and `repo_id` are JSON-escaped with `sed 's/\\/\\\\/g; s/"/\\"/g'` before interpolation (`hook.rs:74-76`, `hook.rs:126`).
- The relay's runtime container must ship `curl` and `openssl`; `hook.rs:184-206` parses the repo `Dockerfile`'s runtime stage and fails the unit test if either is missing.
- `is_ancestor` is *reported by the hook*, not recomputed by the relay, and it is HMAC-covered (pinned `policy.rs:512-520`). The relay trusts the hook's ancestry claim.

#### 8. Startup and shared state wiring

| Item | Site |
|---|---|
| `AppState.git_store` built with `.expect(...)` from the **media** S3 config | `state.rs:694-701` |
| `AppState.git_pack_cache` built with `.expect("git pack cache path must be available")` | `state.rs:702-709` |
| `AppState.git_semaphore = Semaphore::new(git_max_concurrent_ops)`; doc explicitly says it is **not** writer serialization | `state.rs:517-521`, `:729` |
| `git_router` + `git_policy_router` merged into the main router | `router.rs:48-50`, `:137-138` |
| Fatal A3 conformance probe before the listener opens | `main.rs:466-503` |
| `BUZZ_SERVE_GIT_WEB_GUI` gates SPA fallback for `/`, `/repos`, `/repos/*` | `router.rs:206-213` |
| DB-derived gauges `buzz_total_git_repos` / `buzz_community_git_repos` | `main.rs:1499`, `:1706-1725` |
| UDS listener uses `.into_make_service()` (no `ConnectInfo`) | `main.rs:1182` |
| `tokio-util` `io` feature enabled specifically to stream git stdout into the HTTP body | `crates/buzz-relay/Cargo.toml:31-34` |

#### 9. Metrics emitted

| Name | Type | Labels | Site |
|---|---|---|---|
| `buzz_git_semaphore_rejections_total` | counter | `operation` | `transport.rs:323-327` |
| `buzz_git_upload_pack_timeouts_total` | counter | — | `transport.rs:1364` |
| `buzz_git_upload_pack_stream_seconds` / `_bytes` | histogram | — | `transport.rs:1386-1389` |
| `buzz_git_hydrations_total` | counter | `outcome` ∈ {success, missing, invalid_pointer, manifest_error, store_error, hydrate_error, resource_limit} | `hydrate.rs:146` |
| `buzz_git_hydrate_seconds` | histogram | `outcome` | `hydrate.rs:147` |
| `buzz_git_hydrate_bytes` / `_packs` | histogram | — | `hydrate.rs:135-136` |
| `buzz_git_pack_cache_lookups_total` | counter | `result` ∈ {hit, miss, coalesced} | `pack_cache.rs:173`, `:197-200` |
| `buzz_git_pack_cache_populate_seconds` | histogram | `outcome` ∈ {success, bypass, error} | `pack_cache.rs:223-226` |
| `buzz_git_pack_cache_population_wait_seconds` | histogram | — | `pack_cache.rs:266` |
| `buzz_git_pack_cache_populations_active` | gauge | — | `pack_cache.rs:93`, `:268` |
| `buzz_git_pack_cache_bytes` / `_entries` | gauge | — | `pack_cache.rs:477-480` |
| `buzz_git_pack_cache_bypasses_total` | counter | — | `pack_cache.rs:302` |
| `buzz_git_pack_cache_copy_fallbacks_total` | counter | — | `pack_cache.rs:453` |
| `buzz_git_pack_cache_evictions_total` | counter | — | `pack_cache.rs:383` |
| `buzz_git_pack_compactions_total` | counter | `outcome` ∈ {success, fallback, cas_conflict, validation_error, publish_error} | `cas_publish.rs:968` |
| `buzz_git_pack_compaction_seconds` | histogram | `outcome` | `cas_publish.rs:969` |
| `buzz_git_pack_compaction_packs_before` / `_after` / `_bytes` | histogram | — | `cas_publish.rs:971-977` |
| `buzz_git_pack_compaction_required_failures_total` | counter | — | `cas_publish.rs:1129` |

**Gap:** no counter for push outcome (2xx / 409 conflict / 400 invalid-manifest / 413), and none for policy-endpoint allow/deny. CAS contention and authorization denials are therefore invisible in metrics — which matters because the design explicitly says "if contention ever shows up in metrics the fix is a short best-effort *local* lock" (doc §Scope). That signal does not currently exist.
