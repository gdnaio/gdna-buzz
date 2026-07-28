## Module: buzz-relay — git hosting (`crates/buzz-relay/src/api/git`)
### Aspect: Configuration

#### 1. `BUZZ_GIT_*` environment variables — complete inventory

Ten `BUZZ_GIT_*` names are read anywhere in the shipped binary. Three of them (`_S3_PROBE`, plus the `_S3_*` family) are **test-only**.

| Var | Type | Default | Read at | Stored as | Consumed at |
|---|---|---|---|---|---|
| `BUZZ_GIT_REPO_PATH` | `PathBuf`, `create_dir_all`'d | `./repos` | `config.rs:704-706` | `Config::git_repo_path` (`config.rs:218`) | scratch root for every tempdir/tempfile: `transport.rs:604`, `:622`, `:626`, `:797`, `:884`, `:940`; via `repo_path.parent()` in `cas_publish.rs:1028-1030`; `hydrate.rs:294`, `:229` |
| `BUZZ_GIT_PACK_CACHE_PATH` | `PathBuf`, `create_dir_all`'d | `<git_repo_path>/.pack-cache` | `config.rs:707-712` | `Config::git_pack_cache_path` (`:220`) | `state.rs:704` → `GitPackCache::new` (`pack_cache.rs:107-160`) |
| `BUZZ_GIT_MAX_PACK_BYTES` | `u64` | `500 * 1024 * 1024` = 524 288 000 | `config.rs:713-716` | `Config::git_max_pack_bytes` (`:222`) | router body limit `transport.rs:1757`; receive-pack decoded cap `:861`; `HydrationOptions.max_pack_bytes` `:607`, `:801`, `:887`; `PublishLimits.max_pack_bytes` `:1590`; per-pack + per-idx caps `hydrate.rs:305-317`, `:431-446`; `--max-pack-size` `cas_publish.rs:703` |
| `BUZZ_GIT_MAX_REPO_BYTES` | `u64` | **derived**: `git_max_pack_bytes * 2` (saturating) = 1 048 576 000 | `config.rs:717-720` | `Config::git_max_repo_bytes` (`:227`) | `HydrationOptions.max_repo_bytes` `transport.rs:608`, `:802`, `:888`; `hydrate.rs:318-329`; `PublishLimits.max_repo_bytes` `:1591` → `cas_publish.rs:1136-1146`, `:774-782` |
| `BUZZ_GIT_PACK_CACHE_MAX_BYTES` | `u64` | **derived**: `git_max_repo_bytes * 5` (saturating) = 5 242 880 000 | `config.rs:721-724` | `Config::git_pack_cache_max_bytes` (`:230`) | `state.rs:705` → `GitPackCache.max_bytes` (`pack_cache.rs:71`), used by `prune` (`:365-390`) and the bypass check (`:299-313`) |
| `BUZZ_GIT_PACK_CACHE_MAX_CONCURRENT_POPULATIONS` | `usize`, filtered `> 0` | `2` | `config.rs:725-730` | `Config::git_pack_cache_max_concurrent_populations` (`:232`) | `state.rs:706` → `population_semaphore` (`pack_cache.rs:110-113`, acquired `:253-269`) |
| `BUZZ_GIT_MAX_REPOS_PER_PUBKEY` | `u32` | `100` | `config.rs:731-734` | `Config::git_max_repos_per_pubkey` (`:234`) | `handlers/side_effects.rs:2486-2492` (announce-time quota) — **not** consumed inside this module |
| `BUZZ_GIT_MAX_CONCURRENT_OPS` | `usize` | `20` | `config.rs:735-738` | `Config::git_max_concurrent_ops` (`:236`) | `state.rs:693`, `:729` → `git_semaphore` (`state.rs:517-521`), acquired `transport.rs:322` |
| `BUZZ_GIT_HOOK_HMAC_SECRET` | `String` | **random 32 bytes hex per process** | `config.rs:739-744`; length validated `:863-871` | `Config::git_hook_hmac_secret` (`:239`) | `transport.rs:919-921` (`BUZZ_HOOK_SECRET` env for the hook); `policy.rs:236-241` (verification key) |
| `BUZZ_GIT_S3_PROBE` | `"1"` gate | unset | `store.rs:997`, `hydrate.rs:580`, `cas_publish.rs:1582` | — | **test-only**: enables the live-MinIO test bodies |
| `BUZZ_GIT_S3_{ENDPOINT,ACCESS_KEY,SECRET_KEY,BUCKET,REGION}` | `String` | falls back to `BUZZ_S3_*`, then hardcoded MinIO dev values | `cas_publish.rs:1585-1601` (test helper); `crates/buzz-test-client/tests/e2e_git.rs:118-131` | — | **test-only.** `config.rs` never reads them. |

##### Two `BUZZ_GIT_*` names read outside `config.rs`

| Var | Type | Default | Read at | Effect |
|---|---|---|---|---|
| `BUZZ_GIT_CONFORMANCE_PROBE` | `!= "false"` | enabled | `main.rs:470-472` | when `false`, skips the fatal A3 admission gate entirely |
| `BUZZ_GIT_PROBE_WRITERS` | `usize` | `32` | `main.rs:474-477` | `ProbeConfig.race_width`; must be ≥ 2 (`store.rs:572-586`) |
| `BUZZ_GIT_PROBE_ROUNDS` | `usize` | `3` | `main.rs:478-481` | `ProbeConfig.race_rounds`; must be ≥ 1 |

`ProbeConfig::default()` is `{ race_width: 32, race_rounds: 3 }` (`store.rs:114-121`) — matching the `main.rs` defaults, though `main.rs` builds the struct literally rather than using `Default`.

#### 2. Non-`BUZZ_GIT_*` configuration this module depends on

| Config | Source | Used for |
|---|---|---|
| `config.media.s3_{endpoint,access_key,secret_key,bucket,region}` | `BUZZ_S3_*` | **the git object store.** `state.rs:694-701` builds `GitStore` from the *media* config with `.expect("media storage was already constructed with this S3 config")`. There is no dedicated git bucket at runtime. |
| `config.bind_addr` | `BUZZ_BIND_ADDR`, default `0.0.0.0:3000` (`config.rs:365-368`) | only its **port** — the hook callback URL is `http://127.0.0.1:<port>/internal/git/policy` (`transport.rs:911-915`) |
| `config.relay_url` | `RELAY_URL`, default `ws://localhost:3000` | **scheme only** for the NIP-98 `u` derivation (`transport.rs:236-241`); the host always comes from the resolved tenant |
| `config.require_relay_membership` | `BUZZ_REQUIRE_RELAY_MEMBERSHIP`, **default false** (`config.rs:483-485`) | when false, the membership gate in `GitAuth` is a no-op (`api/mod.rs:130-131`) |
| `config.serve_git_web_gui` | `BUZZ_SERVE_GIT_WEB_GUI`, default false (`config.rs:848-850`) | serves the web bundle at `/`, `/repos`, `/repos/*` (`router.rs:206-213`) |
| `state.relay_keypair` | `BUZZ_RELAY_PRIVATE_KEY` | signs kind:30618 (`transport.rs:1690`) |
| `config.uds_path` | `BUZZ_UDS_PATH` | when set, a second listener without `ConnectInfo` — makes `/internal/git/policy` return 403 over UDS (`main.rs:1179-1188`, `mod.rs:41-49`) |
| `PATH` | process env | inherited into every git subprocess (`transport.rs:304`, `hydrate.rs:458-460`) |

#### 3. Derived-default chain

```
BUZZ_GIT_MAX_PACK_BYTES                 = 500 MiB                     (config.rs:713-716)
  └─ BUZZ_GIT_MAX_REPO_BYTES            = pack  × 2  =   1 GiB        (config.rs:717-720)
       └─ BUZZ_GIT_PACK_CACHE_MAX_BYTES = repo  × 5  =   5 GiB        (config.rs:721-724)

BUZZ_GIT_REPO_PATH                      = ./repos                     (config.rs:704-706)
  └─ BUZZ_GIT_PACK_CACHE_PATH           = <repo_path>/.pack-cache     (config.rs:707-712)
```

Both multiplications use `saturating_mul`, so an extreme `max_pack_bytes` saturates instead of wrapping. Consequences of the chain that are easy to miss:

- Raising `BUZZ_GIT_MAX_PACK_BYTES` alone **silently multiplies the pack-cache disk budget by 10×** unless the two downstream vars are also pinned. Setting `pack=2 GiB` yields a 20 GiB cache budget.
- The pack cache lives **inside** the scratch root by default, so a single volume holds ephemeral workspaces, subprocess tempfiles, compaction tempdirs, and up to 5 GiB of cache — and `git_max_repo_bytes` (1 GiB) is the per-request workspace bound on the *same* volume.
- The Helm chart breaks the co-location by pinning `packCachePath` to its own `emptyDir` (`deploy/charts/buzz/templates/deployment.yaml:148`, `:226-241`).

#### 4. Constants that are **not** configurable

These behave like configuration but are compile-time literals.

| Constant | Value | Site |
|---|---|---|
| `INFO_REFS_TIMEOUT` | 120 s | `transport.rs:42` |
| `PACK_OPS_TIMEOUT` | 300 s | `transport.rs:44` |
| `RECEIVE_PACK_MAX_OUTPUT_BYTES` | 1 MiB | `transport.rs:50` |
| `INFO_REFS_MAX_OUTPUT_BYTES` | 4 MiB | `transport.rs:52` |
| `UPLOAD_PACK_MAX_DECODED_BYTES` | 64 MiB | `transport.rs:59` |
| `MAX_FAST_PATH_REFNAME_LEN` | 4096 | `transport.rs:378` |
| `PKT_LINE_MAX_PAYLOAD` | `0xffff - 4` | `transport.rs:422` |
| `PACK_CAPTURE_TIMEOUT` | 300 s | `cas_publish.rs:81` |
| `PACK_COMPACTION_OPERATION_TIMEOUT` | 600 s | `cas_publish.rs:82` |
| `PACK_OBJECTS_WINDOW_MEMORY_BYTES` | 64 MiB | `cas_publish.rs:83` |
| `PACK_OBJECTS_WINDOW` | `"10"` | `cas_publish.rs:84` |
| `MAX_COMPACTION_OBJECTS` | 1 000 000 | `cas_publish.rs:85` |
| `PACK_COMPACTION_SEMAPHORE` | 1 permit | `cas_publish.rs:86` |
| `MAX_REF_SNAPSHOT_BYTES` | 4 MiB | `cas_publish.rs:270` |
| `MAX_COUNT_OBJECTS_OUTPUT_BYTES` | 64 KiB | `cas_publish.rs:595` |
| `MANIFEST_VERSION` | 1 | `manifest.rs:34` |
| `MAX_MANIFEST_PACKS` | 128 | `manifest.rs:39` |
| `PACK_COMPACTION_THRESHOLD` | 96 (`= 128 * 3 / 4`) | `manifest.rs:45` |
| `MAX_MANIFEST_REFS` | 10 000 | `manifest.rs:47` |
| `MAX_MANIFEST_BYTES` | 4 MiB | `hydrate.rs:45` |
| `HEARTBEAT_INTERVAL` | 60 s | `pack_cache.rs:20` |
| `STALE_SESSION_AGE` | 10 min | `pack_cache.rs:21` |
| `MAX_CALLBACK_AGE_SECS` | 30 s | `policy.rs:51` |
| future-skew tolerance | 5 s | `policy.rs:253` |
| policy body limit | 1 MiB | `mod.rs:63` |
| policy ref-update count | 1..=500 | `policy.rs:213` |
| policy `ref_name` max | 256 | `policy.rs:227` |
| hook `curl --max-time` | 10 s | `hook.rs:129` |
| stderr capture prefix | 64 KiB | `transport.rs:678`, `:1122`; `cas_publish.rs:288`, `:551`, `:761` |
| `MAX_PROTECTION_RULES` / `MAX_PATTERN_LENGTH` / `MAX_WILDCARDS_PER_PATTERN` | 50 / 256 / 3 | `buzz-core/src/git_perms.rs:19-23` |
| `DEFAULT_HEAD` | `refs/heads/main` | `handlers/side_effects.rs:2611` |

The hook's 10 s `curl` timeout sits well inside the 300 s `receive-pack` timeout, so a slow policy endpoint fails the push rather than hanging it.

#### 5. Validation and failure behaviour at load

| Check | Behaviour |
|---|---|
| `BUZZ_GIT_REPO_PATH` / `BUZZ_GIT_PACK_CACHE_PATH` | `create_dir_all`; failure is a hard `ConfigError::InvalidValue` naming the setting (`config.rs:388-400`); pinned by `config.rs:1304-1341` |
| Numeric vars | `.ok().and_then(parse)` — **an unparseable value silently falls back to the default**, no warning. Applies to `_MAX_PACK_BYTES`, `_MAX_REPO_BYTES`, `_PACK_CACHE_MAX_BYTES`, `_MAX_REPOS_PER_PUBKEY`, `_MAX_CONCURRENT_OPS`, and both probe vars (`config.rs:713-738`, `main.rs:474-481`) |
| `_PACK_CACHE_MAX_CONCURRENT_POPULATIONS` | additionally `.filter(|v| *v > 0)`, so `0` falls back to 2 (`config.rs:729`); `GitPackCache::new` rejects 0 defensively (`pack_cache.rs:110-113`) |
| `BUZZ_GIT_HOOK_HMAC_SECRET` | if **set**, must be ≥ 32 chars, else hard error (`config.rs:863-871`); if unset, a 64-hex random value is generated with no warning |
| Pack-cache parent is a symlink | hard error at `GitPackCache::new` → `.expect()` in `state.rs:702-709` ⇒ process abort |
| S3 config half-set (one key empty) | `StoreError::Config` → `.expect()` in `state.rs:694-701` ⇒ process abort |
| A3 conformance probe | any phase failure ⇒ `anyhow` error ⇒ startup aborts (`main.rs:490-495`); `race_width < 2` or `race_rounds == 0` fails the `config` phase (`store.rs:572-586`) |
| `BUZZ_GIT_MAX_PACK_BYTES` on 32-bit | `as usize` cast for the body limit (`transport.rs:1757`) would truncate; irrelevant on 64-bit targets |

Zero of the `BUZZ_GIT_*` numeric vars log a warning on a parse failure, and none is range-checked. A typo like `BUZZ_GIT_MAX_CONCURRENT_OPS=20x` silently yields 20.

#### 6. Deployment surfaces

##### `.env.example` (`.env.example:71-79`)

Documents 5 of the 8 real config vars: `_REPO_PATH`, `_MAX_PACK_BYTES`, `_MAX_REPO_BYTES`, `_PACK_CACHE_PATH`, `_PACK_CACHE_MAX_BYTES`, `_PACK_CACHE_MAX_CONCURRENT_POPULATIONS`. **Missing:** `BUZZ_GIT_MAX_REPOS_PER_PUBKEY`, `BUZZ_GIT_MAX_CONCURRENT_OPS`, `BUZZ_GIT_HOOK_HMAC_SECRET`, and all three probe vars.

##### Helm chart (`deploy/charts/buzz/templates/deployment.yaml`)

| Env | Source |
|---|---|
| `BUZZ_GIT_REPO_PATH` | `.Values.persistence.git.mountPath` (`:146`) |
| `BUZZ_GIT_MAX_PACK_BYTES` | `.Values.git.maxPackBytes` (`:147`) |
| `BUZZ_GIT_PACK_CACHE_PATH` | `.Values.git.packCachePath` (`:148`) |
| `BUZZ_GIT_PACK_CACHE_MAX_BYTES` | `.Values.git.packCacheMaxBytes` (`:149`) |
| `BUZZ_GIT_PACK_CACHE_MAX_CONCURRENT_POPULATIONS` | `.Values.git.packCacheMaxConcurrentPopulations` (`:150`) |
| `BUZZ_GIT_MAX_REPOS_PER_PUBKEY` | `.Values.git.maxReposPerPubkey` (`:151`) |
| `BUZZ_GIT_MAX_CONCURRENT_OPS` | `.Values.git.maxConcurrentOps` (`:152`) |
| `BUZZ_GIT_HOOK_HMAC_SECRET` | Kubernetes Secret key (`:168-172`) |
| `BUZZ_S3_*` | ConfigMap + Secret (`:157-159`, `:191-201`) |

Volumes: `git-repos` (PVC or `emptyDir` with `sizeLimit`) at the repo path, and `git-pack-cache` as a dedicated `emptyDir` with `packCacheVolumeSize` (`:226-241`). **`BUZZ_GIT_MAX_REPO_BYTES` is not set by the chart**, so it always takes the `pack × 2` derived value — and if an operator raises `maxPackBytes`, both the repo bound and the cache bound move with it while `packCacheVolumeSize` does not. A `packCacheMaxBytes` larger than `packCacheVolumeSize` results in ENOSPC rather than eviction.

#### 7. Deltas found

| Claim | Source | Reality |
|---|---|---|
| `BUZZ_GIT_S3_BUCKET/_ENDPOINT/_REGION/_ACCESS_KEY/_SECRET_KEY` listed as relay configuration | `.aidlc/reverse-engineer/configuration.md:159` | **Not configuration.** No `config.rs` read exists; the names appear only in `#[cfg(test)]` helpers (`cas_publish.rs:1585-1601`) and the e2e test (`crates/buzz-test-client/tests/e2e_git.rs:118-131`). At runtime the git store reuses `config.media.s3_*` (`state.rs:694-701`). |
| `BUZZ_GIT_HOOK_HMAC_SECRET` "required (git enabled)" | `.aidlc/reverse-engineer/configuration.md:160` | Optional — a random per-process secret is generated when unset (`config.rs:739-744`), and that is functionally safe because the hook always dials `127.0.0.1` into the same process (`transport.rs:911-915`). |
| `BUZZ_GIT_MAX_REPO_BYTES` "consumed at `api/git/hydrate.rs`" | `.aidlc/reverse-engineer/_work/modules/relay-core-configuration.md:98` | Also consumed on the publish side via `PublishLimits` (`transport.rs:1591` → `cas_publish.rs:1136-1146`, `:774-782`). |
| `BUZZ_GIT_MAX_PACK_BYTES` "consumed at `transport.rs:1757` (body limit)" | `relay-core-configuration.md:97` | Six additional consumers: decoded-gzip cap (`transport.rs:861`), three `HydrationOptions` sites, `PublishLimits`, per-pack/per-idx caps (`hydrate.rs:305-317`, `:431-446`), and `--max-pack-size` (`cas_publish.rs:703`). |
| `BUZZ_GIT_PACK_CACHE_MAX_BYTES` default "5 GB" | `.env.example:78` shows `5368709120` | Consistent: `1 048 576 000 × 5 = 5 242 880 000`. The `.env.example` literal `5368709120` (= 5 × 2³⁰) is **not** the derived default — the derived value is 5 242 880 000. Setting the documented value changes behaviour by ~125 MB. |
| Doc: "Each process may retain a byte-bounded, process-lifetime cache … Deployment mounts that cache on a per-pod ephemeral volume" | `docs/git-on-object-storage.md` §v1 | Matches the chart (`deployment.yaml:238-241`) but **not** the code default, which nests the cache inside the scratch root (`config.rs:707-712`). |
