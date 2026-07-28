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
