## Module: buzz-relay — git hosting (`crates/buzz-relay/src/api/git`)
### Aspect: Data Model

#### 1. Storage tiers

Git state lives in **three** places. None of them is a persistent per-repo filesystem.

| Tier | Physical form | Mutability | Authority | Code |
|---|---|---|---|---|
| Object store (S3/MinIO) | `pointer`, `manifests/<sha256>`, `packs/<sha256>`, `idx/<pack-sha256>` | pointer mutable (CAS); rest create-only | **source of truth** | `store.rs:170-908` |
| Postgres | `git_repo_names` (name registry + quota), events (kind 30617/30618) | mutable | name allocation only | `handlers/side_effects.rs:2441-2480` |
| Local disk | per-request bare-repo tempdir; process-lifetime pack cache | ephemeral | pure cache/scratch | `hydrate.rs:293-379`, `pack_cache.rs:66-426` |

`store.rs:22-24` states the read-side rule explicitly: bytes are verified against the key digest, so deviation is *detectable*, not silent.

#### 2. Object-store key scheme

| Key | Producer | Body | Content type | Precondition |
|---|---|---|---|---|
| `repos/<community-uuid>/<owner-hex>/<repo>/pointer` | `manifest.rs:181-184` | 64-hex manifest digest, bare (no prefix, no JSON) | `application/json` (mislabeled — body is bare hex) | `If-Match: <etag>` or `If-None-Match: *` (`store.rs:481-514`) |
| `manifests/<sha256(bytes)>` | `store.rs:337-340` via `put_immutable` (`:254-280`) | canonical manifest JSON | `application/json` | `If-None-Match: *` |
| `packs/<sha256(bytes)>` | `store.rs:282-285` | raw git packfile | `application/x-git-pack` | `If-None-Match: *` |
| `idx/<pack-sha256>` | `store.rs:294-321` | `.idx` sidecar | `application/x-git-index` | `If-None-Match: *` |
| `probe/pointer-<uuid>`, `probe/inm-race/<sha256>`, `probes/pack…` | `store.rs:589-877` | probe payloads | mixed | mixed |

Three properties worth naming:

- **The pointer namespace is community-scoped; the CAS namespace is not.** `pointer_key` interpolates `CommunityId` (`manifest.rs:181-184`, pinned by `manifest.rs:491-523`), but `manifests/`, `packs/`, and `idx/` are global. Two communities pushing byte-identical packs share one object. That is intentional (content addressing) and safe, but it means **cross-community existence of a pack digest is observable to anyone with bucket read access**.
- **The idx key is *not* content-addressed on its own bytes** — it is keyed by the *pack* digest (`store.rs:227-241`). The doc comment says this keeps the manifest schema unchanged so readers can derive `idx/<pack_digest>`. Consequence: idx bytes are unverified on read (`hydrate.rs:388-408` writes them, then validates via `git verify-pack`, regenerating on failure). This is the only object read where digest verification is replaced by a subprocess check.
- **`.git` suffix is stripped at the key boundary** (`manifest.rs:182`), so `/git/o/r` and `/git/o/r.git` address the same pointer.

#### 3. Manifest format (`manifest.rs:49-73`)

```json
{"version":1,"head":"refs/heads/main","refs":{"refs/heads/main":"aaaa…"},"packs":["packs/<64hex>"],"parent":null}
```

| Field | Type | Meaning | Constraint | Enforced |
|---|---|---|---|---|
| `version` | `u32` | schema version, currently `1` | must `== MANIFEST_VERSION` on read | `manifest.rs:263-272`, const `:34` |
| `head` | `String` | symbolic HEAD target, **unprefixed** (no `ref: `) | non-empty + `is_safe_refname` | `manifest.rs:216-221` |
| `refs` | `BTreeMap<String,String>` | refname → hex oid | ≤ `MAX_MANIFEST_REFS` = 10 000 (`:47`); each key `is_safe_refname`, each value `is_hex_oid` | `manifest.rs:210-232` |
| `packs` | `Vec<String>` | full store keys, sorted+deduped | ≤ `MAX_MANIFEST_PACKS` = 128 (`:39`); each `is_pack_key` | `manifest.rs:205-209`, `:233-237` |
| `parent` | `Option<String>` | **bare** 64-hex digest of superseded manifest | `is_manifest_digest` (lowercase hex only) | `manifest.rs:238-242` |

Canonicalization (`manifest.rs:255-261`): clone → `packs.sort()` → `packs.dedup()` → `serde_json::to_vec`. Determinism rests on (a) `BTreeMap` iteration order, (b) declaration field order, (c) serde's no-whitespace output. Pinned byte-for-byte by `manifest.rs:550-568` and `:367-385`.

Notable asymmetries:

- `canonical_bytes` deliberately does **not** call `validate()` (`manifest.rs:251-254`) so the validation seam is visible at the write site (`cas_publish.rs:1177-1190`).
- `is_hex_oid` accepts 40 **and** 64 hex (`manifest.rs:156-158`) but `is_manifest_digest` (`:165-167`) accepts only lowercase 64. So a ref oid may be uppercase hex while a parent digest may not.
- `head` is stored in `refs`-key form and *not* required to be present in `refs`. `manifest.rs:367-385` pins that the announce seed is `head="refs/heads/main"` with `refs:{}` — a legal manifest whose HEAD dangles.

##### Derived constants

| Const | Value | Derivation | Site |
|---|---|---|---|
| `MANIFEST_VERSION` | 1 | literal | `manifest.rs:34` |
| `MAX_MANIFEST_PACKS` | 128 | literal | `manifest.rs:39` |
| `PACK_COMPACTION_THRESHOLD` | 96 | `MAX_MANIFEST_PACKS * 3 / 4` | `manifest.rs:45` |
| `MAX_MANIFEST_REFS` | 10 000 | literal | `manifest.rs:47` |
| `MAX_MANIFEST_BYTES` (read cap) | 4 MiB | literal | `hydrate.rs:45` |
| `MAX_REF_SNAPSHOT_BYTES` | 4 MiB | literal | `cas_publish.rs:270` |

#### 4. Pointer / CAS model

```
pointer(repo) = (ETag e, digest d)      store.rs:455-479 — one GET returns both
manifest(d)   = get_verified("manifests/"+d, d)   hydrate.rs:262-266
packs         = manifest.packs                    hydrate.rs:302-329
```

Types:

| Type | Shape | Site |
|---|---|---|
| `ETag(pub String)` | opaque, quoting preserved verbatim | `store.rs:37` |
| `Precond` | `IfNoneMatchStar \| IfMatch(ETag)` | `store.rs:41-46` |
| `CasOutcome` | `Won(ETag) \| LostRace` — 412 is a *value*, not an error | `store.rs:58-64` |
| `ParentState` | `{ if_match: Option<ETag>, parent_digest: Option<String>, parent: Manifest }` | `cas_publish.rs:215-228` |
| `CasSuccess` | `{ manifest: Manifest, manifest_key: String }` | `cas_publish.rs:192-197` |

`ParentState::fresh()` (`cas_publish.rs:232-243`) models the first push as `if_match=None`, `parent_digest=None`, and an **empty manifest with `head = ""`** — the only place a manifest with empty `head` exists in memory; `validate()` rejects it (`manifest.rs:216-218`), which is exactly the guard `cas_publish.rs:1885-1904` pins.

**One GET, not HEAD-then-GET.** `get_pointer` extracts ETag and body from the *same* response (`store.rs:455-479`) and errors if the ETag header is absent (`:462-471`). `classify_cas` similarly fails closed when a 2xx PUT omits the ETag (`store.rs:516-537`) rather than returning `ETag("")`.

#### 5. Ref-state representation across layers

The same ref state has four distinct on-the-wire shapes. Divergence between them is the module's main data-model hazard.

| Layer | HEAD form | Ref form | Site |
|---|---|---|---|
| Manifest (storage) | bare `refs/heads/main` | `BTreeMap<refname, oid>` | `manifest.rs:53-73` |
| Hydrated bare repo | `HEAD` file = `ref: refs/heads/main\n` | loose file per ref, body `oid\n` | `hydrate.rs:355-371` |
| pkt-line advertisement | `<oid> HEAD\0…symref=HEAD:<ref>` | `<oid> <refname>\n`, BTreeMap order | `transport.rs:471-537` |
| kind:30618 event | tag `["HEAD","ref: refs/heads/main"]` | tag `[<refname>, <oid>]` per ref | `manifest_event.rs:88-104` |

Filters differ per layer, and they are **not** the same predicate:

| Predicate | Accepts | Site |
|---|---|---|
| `is_safe_refname` | any `refs/…`, alphabet `[A-Za-z0-9/_.-]`, no `..`/`//`/leading-or-trailing `/` | `manifest.rs:142-154` |
| `is_emittable_ref` (30618) | only `refs/heads/*` and `refs/tags/*` | `manifest_event.rs:117-127` |
| `fast_path_eligible` | no `refs/tags/*` at all; `head ∈ refs`; refname ≤ 4096 | `transport.rs:401-419` |
| policy-endpoint ref check | `refs/…` prefix, ≤ 256 bytes, no `..`, no byte ≤ 0x20 or 0x7f | `policy.rs:225-233` |

So `refs/notes/*`, `refs/stash`, `refs/pull/*` can be stored in a manifest and hydrated, but are **silently dropped** from kind:30618 (`manifest_event.rs:82-93`, pinned `:255-289`). Subscribers reading only 30618 see an incomplete ref set. Same for oids that are not 40/64 hex (`manifest_event.rs:129-131`, `:310-335`) — skipped, not failed.

##### Snapshot → manifest conversion is 40-hex only

`snapshot_workspace_state` (`cas_publish.rs:268-352`) parses `git for-each-ref --format=%(refname) %(objectname)` and **skips any ref whose oid is not exactly 40 hex** with a `warn!` (`cas_publish.rs:325-328`). Every other layer accepts 64-hex (`manifest.rs:156`, `manifest_event.rs:129`, `transport.rs:475-479` even derives `object-format=sha256` from a 64-char oid). A SHA-256 repository would therefore snapshot to `refs = {}` and publish a manifest that **silently deletes every ref**. Unreachable today because `init_bare_repo` (`hydrate.rs:181-184`) always creates a SHA-1 bare repo, but the failure mode is silent-drop rather than fail-closed.

#### 6. Pack-cache on-disk layout (`pack_cache.rs`)

```
<BUZZ_GIT_PACK_CACHE_PATH>/                     # rejected if a symlink — pack_cache.rs:118-126
  session-<rand>/                               # one per process — pack_cache.rs:127-132
    .heartbeat                                  # mtime touched every 60s — pack_cache.rs:20, :133-146
    <d0><d1>/                                   # 2-hex shard of the pack digest — pack_cache.rs:271
      .staging-<rand>/                           # pack + idx built here — pack_cache.rs:274-279
      <64-hex-digest>/                           # published by atomic rename — pack_cache.rs:311-331
        pack-<digest>.pack
        pack-<digest>.idx
```

In-memory index (`pack_cache.rs:53-64`): `HashMap<digest, CacheRecord{ entry: Arc<CachedPack>, last_used: u64 }>` plus `total_bytes` and a monotonic `tick` used as a logical LRU clock (`:241-251`). `CachedPack` (`:24-51`) records `pack_bytes`, `total_bytes = pack + idx`, and an optional `_temporary: TempDir` that keeps an *over-capacity* (bypassed) entry alive without ever entering the index (`:303-313`).

Installation into a request workspace is **hard-link first, copy fallback** (`pack_cache.rs:428-456`), so cache and workspace share inodes; a copy fallback increments `buzz_git_pack_cache_copy_fallbacks_total`.

Cross-process publication races are resolved by re-reading `CachedPack::from_dir` after a failed rename (`pack_cache.rs:314-330`) — the design assumes multiple relay processes may share the scratch volume.

Startup GC deletes any sibling `session-*` directory whose `.heartbeat` (falling back to the dir) mtime is older than `STALE_SESSION_AGE` = 10 min (`pack_cache.rs:21`, `:482-509`).

#### 7. Hydrated workspace layout (`hydrate.rs:293-379`)

```
<BUZZ_GIT_REPO_PATH>/.tmpXXXXXX/     # TempDir, dropped with HydratedRepo — hydrate.rs:294-299
  HEAD                               # "ref: <manifest.head>\n" — written LAST, hydrate.rs:373-375
  objects/pack/pack-<digest>.{pack,idx}   # phase 1 — hydrate.rs:302-329
  refs/…                             # loose, one file per manifest ref — phase 2, hydrate.rs:355-371
  hooks/pre-receive                  # write path only, 0o755 — hook.rs:152-178
```

`HydratedRepo` (`hydrate.rs:51-77`) carries `hydrated_bytes` and `hydrated_packs`, which flow into `PublishLimits.parent_hydrated_bytes` (`cas_publish.rs:148-155`) and the `buzz_git_hydrate_bytes`/`_packs` histograms (`hydrate.rs:135-136`).

**The phase boundary is the data-model invariant**: packs are all present and indexed before any ref file exists, so a failed hydrate yields "no refs" rather than "refs pointing at missing objects" (`hydrate.rs:288-292`, `:300-304`, `:351-353`).

#### 8. Relation of kind 30617 / 30618 to stored objects

| Event | Author | `d` tag | Role w.r.t. object store |
|---|---|---|---|
| kind **30617** (NIP-34 repo announcement) | repo **owner** | `repo_id` | authorization input only. Read by the policy endpoint to resolve protection rules + channel binding (`policy.rs:283-302`). Its arrival triggers name reservation + pointer seeding (`handlers/side_effects.rs:2405-2578`). |
| kind **30618** (NIP-34 repo state) | **relay** keypair | `repo_id` (must equal 30617's `d`) | *derived notification*, never the commit (`transport.rs:1662-1746`; doc §System Model). Built from `CasSuccess.manifest` — the bytes that landed. |

Mapping details:

- The relay signs 30618, not the pusher; the pusher/owner rides in a `p` tag as a Buzz extension (`manifest_event.rs:106-107`, doc-commented `:17-22`).
- `d` must be the **stripped** repo id. `PushContext.repo_id` is set from `validate_repo_id`'s return (`transport.rs:860`, `:948`), while `PushContext.repo` keeps the raw URL segment for `pointer_key` (`:947`, `:1521-1526`). A mismatch would make 30618 un-joinable to 30617 — pinned by `manifest_event.rs:382-393`.
- Announce seeds the pointer with the canonical empty manifest **before** publishing 30618 (`handlers/side_effects.rs:2606-2694`, then `:2733`). All empty manifests across all repos and communities are byte-identical, hence one shared `manifests/<digest>` object (`handlers/side_effects.rs:2624-2627`; pinned `manifest.rs:373-384`).
- `DEFAULT_HEAD = "refs/heads/main"` is pinned once (`handlers/side_effects.rs:2611`) and shared by the seed manifest and the initial 30618 so they cannot drift.
- Emission is skipped when `parent_digest == committed_digest` (`transport.rs:1677-1683`), and the DB dedups a repeat (`transport.rs:1712-1719`).

#### 9. HTTP-layer data shapes

`HookCallbackRequest` (`policy.rs:53-86`) / `HookRefUpdate` (`:73-86`) are the only JSON bodies the module accepts:

```json
{"repo_id":"…","repo_owner":"<64hex>","community_id":"<uuid>","pusher_pubkey":"<64hex>",
 "ref_updates":[{"old_oid":"<40hex>","new_oid":"<40hex>","ref_name":"refs/…","is_ancestor":true}],
 "timestamp":<u64>,"signature":"<hex hmac-sha256>"}
```

Response: `HookCallbackResponse { allowed: bool, denials: Vec<DenialResponse{ref_name, reason}> }` (`policy.rs:87-104`), `denials` omitted when empty (`:91-92`).

HMAC pre-image (`policy.rs:113-157`) is length-prefixed and `|`-separated:
`len(repo_id):repo_id | repo_owner | community_id | pusher_pubkey | (old_oid ‖ new_oid ‖ len(ref):ref ‖ "1"/"0")* sorted by ref_name | timestamp`.

#### 10. Deltas found against documentation

| Claim | Source | Reality |
|---|---|---|
| "manifest … containing `m.packs` … `m.refs`" | doc §System Model | Manifest also carries `version`, `head`, `parent` — `head` and `parent` are load-bearing (`manifest.rs:53-73`) |
| `ParentState` at `cas_publish.rs:154`; `cas_publish` at `:410`; `CasError::Conflict` at `:92` | doc §Implementation Correspondence table | Actual: `:215`, `:997`, `:105`. The doc warns line numbers are pinned at landing time. |
| "`build_git_response` (sole `Body::from(stdout)` site)" … "has two call sites" shared with read paths | doc §Implementation Correspondence | **Stale.** Both call sites are inside `finalize_push` (`transport.rs:1574`, `:1748`). Read paths now use `stream_git_read` (`:1414`) and an inline builder in `info_refs_subprocess` (`:715-721`). Push-side uniqueness is now trivially true. |
| "pointer-absence means never announced" | doc §Implementation Correspondence | Not enforced on the write path: `hydrate_for_write` creates a fresh workspace for an absent pointer (`hydrate.rs:227-244`) and `cas_publish` will `IfNoneMatchStar` a pointer into existence (`cas_publish.rs:1230-1237`) without consulting kind:30617 or `git_repo_names`. See security aspect. |
| `store.rs` module comment "wired in by the push path in a follow-up commit" beside `#![allow(dead_code)]` | `store.rs:25` | Stale — the push path is wired in (`cas_publish.rs:826`, `:1194`, `:1235`); the blanket allow now masks real dead code. |
| `hydrate.rs:24-30` "We narrow `#[allow(dead_code)]` to those specific items" | `hydrate.rs:24-30` | Stale — there is no `#[allow(dead_code)]` anywhere in `hydrate.rs`. |
