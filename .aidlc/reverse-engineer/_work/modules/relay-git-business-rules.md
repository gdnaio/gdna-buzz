## Module: buzz-relay — git hosting (`crates/buzz-relay/src/api/git`)
### Aspect: Business Rules

Rules are numbered BR-GIT-nn. Each cites the enforcing site. §7 is the CAS correctness argument.

---

#### 1. Transport-level admission (all three git routes)

| # | Rule | Enforcement | Failure |
|---|---|---|---|
| BR-GIT-01 | Every git route requires a NIP-98 `Authorization: Nostr <base64>` header; it is validated **before** the request body is read (`FromRequestParts`). | `transport.rs:76-231` | 401 + `WWW-Authenticate` |
| BR-GIT-02 | The request `Host` must resolve, through the authoritative `communities` table, to a community. Forwarded headers are never trusted. | `transport.rs:122-130` → `tenant::bind_community` (`tenant.rs:71-92`) | 404 "repository not found" (deliberately indistinguishable from a missing repo) |
| BR-GIT-03 | The NIP-98 `u` tag must equal `<scheme>://<resolved-tenant-host><repo-root-path>`. Scheme derives from `config.relay_url` (`wss://`⇒https, else http); the **host is never taken from config**. | `transport.rs:132-142`, `git_expected_url` `:235-253`; pinned `:2067-2141` | 401 URL mismatch |
| BR-GIT-04 | The HTTP method is **deliberately not bound**: the method fed to the verifier is read out of the signed event itself, making the check tautological. Reason: git's credential helper signs once with `GET` and reuses for `POST`. | `transport.rs:158-183` (explicit SECURITY comment) | n/a |
| BR-GIT-05 | The NIP-98 payload hash is **not** verified (`body = None`) because the pack stream cannot be buffered. | `transport.rs:183-190` | n/a |
| BR-GIT-06 | NIP-98 event-id replay dedup is **deliberately disabled** on git routes; the ±60 s timestamp window is the only replay bound. | `transport.rs:192-198` | n/a |
| BR-GIT-07 | The pusher must pass the relay-membership gate. A NIP-OA attestation may travel in the signed event's `auth` tag or an `x-auth-tag` header, event tag winning. | `transport.rs:204-222`; `handlers/auth.rs:26-36`; `api/mod.rs:124-145` | 403 "restricted: not a relay member" |
| BR-GIT-08 | On an **open** relay (`require_relay_membership = false`, the default) BR-GIT-07 is a no-op — any well-formed NIP-98 signature from any keypair passes. | `api/mod.rs:130-131`; default at `config.rs:483-485`, pinned `config.rs:954-955` | n/a |
| BR-GIT-09 | `owner` must be exactly 64 **lowercase** hex chars. | `transport.rs:268-275` | 400 |
| BR-GIT-10 | `repo` (after stripping one trailing `.git`) must be 1–64 chars of `[A-Za-z0-9._-]`, must not start with `.`, must not contain `..`. | `transport.rs:278-292` | 400 |
| BR-GIT-11 | `service` must be exactly `git-upload-pack` or `git-receive-pack`. | `transport.rs:541-548` | 400 |
| BR-GIT-12 | **There is no per-repo read authorization.** Neither `info_refs` nor `upload_pack` consults kind:30617, channel membership, or the repo-name registry. Any caller past BR-GIT-01/07 can clone any repo in the community. | absence of any check in `transport.rs:539-594`, `:786-827` | — |

---

#### 2. Resource admission and limits

| # | Rule | Value | Site |
|---|---|---|---|
| BR-GIT-13 | Compressed request body ≤ `git_max_pack_bytes` (default 500 MiB) on all three routes. | config | `transport.rs:1757`, `:1763` |
| BR-GIT-14 | gzip-encoded bodies are transparently inflated; **decoded** bytes are capped independently — 64 MiB for upload-pack, `git_max_pack_bytes` for receive-pack. Exceeding it errors the stream, which the stdin pump surfaces to git as early EOF. | `transport.rs:59`, `:745-784`, `:789`, `:861` | pinned `:1823-1887` |
| BR-GIT-15 | Only `gzip`/`x-gzip` is decoded; any other non-identity encoding is passed through unchanged rather than rejected. | `transport.rs:757-765` | |
| BR-GIT-16 | Every git subprocess must hold a permit from the global `git_semaphore` (`git_max_concurrent_ops`, default 20), acquired **non-blocking**. | `transport.rs:318-338`; sized `state.rs:729` | 503 + `Retry-After: 5`, metric `buzz_git_semaphore_rejections_total` |
| BR-GIT-17 | The `info/refs` **fast path acquires no permit** — it is the dominant clone case and does no subprocess work. | `transport.rs:552-577` (permit only in `info_refs_subprocess` `:598`) | |
| BR-GIT-18 | For streamed responses the permit must outlive the body, not the handler. | `GitPermitStream` `transport.rs:1293-1310`, moved in at `:1477-1480`; pinned `:1924-1941` | |
| BR-GIT-19 | `info/refs` subprocess output ≤ 4 MiB; receive-pack buffered output ≤ 1 MiB. | `transport.rs:52`, `:50`; enforced `:680-698`, `:1092-1108` | 413 |
| BR-GIT-20 | `info/refs` subprocess timeout 120 s; pack ops 300 s (both the buffered push and the streaming fetch deadline). | `transport.rs:42`, `:44`; `:660-668`, `:1067-1078`, `:1471` | 504, or in-band stream error + child kill (`:1315-1331`) |
| BR-GIT-21 | Manifest object read ≤ 4 MiB; ref snapshot stdout ≤ 4 MiB; `count-objects -v` output ≤ 64 KiB; stderr capture ≤ 64 KiB. | `hydrate.rs:45`; `cas_publish.rs:270`, `:595`, `:678`, `:1091` etc. | 413 / `ResourceLimit` |
| BR-GIT-22 | Each pack object ≤ `git_max_pack_bytes`, enforced twice: `head_object` Content-Length pre-check then post-read length re-check. | `store.rs:406-444`; `hydrate.rs:305-317` | `ObjectTooLarge` → `ResourceLimit` → 413 |
| BR-GIT-23 | Total hydrated bytes for one repo ≤ `git_max_repo_bytes` (default `pack*2` = 1 GiB), accumulated with `checked_add`. | `hydrate.rs:318-329` | 413 |
| BR-GIT-24 | Generated/fetched `.idx` sidecar ≤ `max_pack_bytes`. | `hydrate.rs:431-446` | `ResourceLimit` |
| BR-GIT-25 | Manifest cardinality: ≤ 128 packs, ≤ 10 000 refs, every pack key canonical. | `manifest.rs:205-237` | `ManifestInvalid` → 400 |
| BR-GIT-26 | Workspace snapshot refuses a **new** ref past 10 000. | `cas_publish.rs:322-326` | `ResourceLimit` → 413 |
| BR-GIT-27 | Repos per pubkey ≤ `git_max_repos_per_pubkey` (default 100), checked at announce, not at push. | `handlers/side_effects.rs:2486-2492`; config `config.rs:731-734` | announce rejected |
| BR-GIT-28 | Protection rules per repo ≤ 50; pattern ≤ 256 chars; ≤ 3 wildcards per pattern. | `buzz-core/src/git_perms.rs:19-23`, `:387` | `RuleParseError` → 403 |
| BR-GIT-29 | Policy callback ref-update count must be `1..=500`. | `policy.rs:213-215` | 403 |
| BR-GIT-30 | `pack-objects` is pinned to `--threads=1`, `--window 10`, `--window-memory=64MiB` on both the delta and compaction paths. | `cas_publish.rs:83-84`, `:524-537`, `:707-727` | |

---

#### 3. Push authorization chain (in execution order)

1. **BR-GIT-31** — BR-GIT-01…11 (auth + id validation) run first (`transport.rs:858-861`).
2. **BR-GIT-32** — semaphore permit (`transport.rs:863`).
3. **BR-GIT-33** — `hydrate_for_write` observes the pointer exactly once and returns `(HydratedRepo, ParentState)`; pointer-absent yields a fresh empty bare repo + `ParentState::fresh()` (`hydrate.rs:207-244`, `transport.rs:878-892`).
4. **BR-GIT-34** — the pre-receive hook is written into the ephemeral workspace at 0o755 (`hook.rs:152-178`, called `transport.rs:906`). Hook install failure ⇒ 500, push aborted.
5. **BR-GIT-35** — `core.hooksPath` is force-overridden via `GIT_CONFIG_COUNT/KEY_0/VALUE_0` so a repo-local setting cannot redirect the hook (`transport.rs:924-930`).
6. **BR-GIT-36** — hook env carries exactly `BUZZ_HOOK_URL`, `BUZZ_HOOK_SECRET`, `BUZZ_REPO_ID` (stripped id), `BUZZ_REPO_OWNER`, `BUZZ_COMMUNITY_ID` (server-resolved), `BUZZ_PUSHER_PUBKEY` (`transport.rs:912-923`). The hook fails closed if any is unset (`hook.rs:42-47`).
7. **BR-GIT-37** — the hook computes `is_ancestor` per ref via `git merge-base --is-ancestor`, inheriting git's quarantine env; exit 128 (error) is treated as *not* ancestor, i.e. fail-closed toward NFF (`hook.rs:59-70`).
8. **BR-GIT-38** — the hook POSTs to `BUZZ_HOOK_URL` with `curl --max-time 10`; any network error or any non-200 ⇒ `exit 1` ⇒ push rejected (`hook.rs:129-148`).
9. **BR-GIT-39** — `BUZZ_HOOK_URL` is hardcoded to `http://127.0.0.1:<bind_addr.port()>/internal/git/policy` (`transport.rs:911-915`). A relay bound to a non-loopback-inclusive address makes every push fail closed.

##### Policy endpoint rules (`policy.rs:173-417`), evaluated in this order

| # | Rule | Site |
|---|---|---|
| BR-GIT-40 | Structural validation runs **before** HMAC (explicitly, to avoid HMAC CPU on garbage): `repo_id` 1–64; `repo_owner`/`pusher_pubkey` 64 lowercase hex; `community_id` a parseable UUID; 1–500 ref updates; each oid exactly 40 hex; each `ref_name` non-empty, ≤256, `refs/`-prefixed, no `..`, no byte ≤ 0x20 or == 0x7f. | `policy.rs:176-234` |
| BR-GIT-41 | HMAC-SHA256 over the length-prefixed canonical payload must match, compared in constant time (`subtle::ConstantTimeEq`). | `policy.rs:113-171`, `:236-241` |
| BR-GIT-42 | Callback age ≤ 30 s; future skew ≤ 5 s. | `policy.rs:51`, `:243-256` |
| BR-GIT-43 | A kind:30617 must exist for `(community, pubkey=repo_owner, d=repo_id)`, `global_only`. Absence ⇒ 403 "repository not found". | `policy.rs:258-284` |
| BR-GIT-44 | Malformed `buzz-protect` tags ⇒ deny (fail-closed). Unknown rule strings are warned and skipped. | `policy.rs:286-306` |
| BR-GIT-45 | If kind:30617 carries a `buzz-channel` tag and that channel is archived, **all** pushes are denied — including the owner's. | `policy.rs:308-330` |
| BR-GIT-46 | The repo owner pubkey, **or** a cryptographically verified NIP-OA managed-agent owner of that pubkey, is treated as `MemberRole::Owner`. | `policy.rs:332-368` (`db.is_agent_owner` `:352`) |
| BR-GIT-47 | Otherwise a `buzz-channel` binding is mandatory; without one ⇒ 403 "no channel binding". | `policy.rs:369-375` |
| BR-GIT-48 | Non-owner role comes from `get_member_role(community, channel, pusher)`; non-member ⇒ 403; unparseable role ⇒ 403. | `policy.rs:376-395` |
| BR-GIT-49 | `MemberRole::Bot` is promoted to `Member` for git only; the promotion is local to this module. | `policy.rs:397-402` |
| BR-GIT-50 | Any DB error at any lookup ⇒ 403, never 5xx. | `policy.rs:277-282`, `:322-326`, `:355-362`, `:388-392` |

##### Ref-protection rules (`buzz-core/src/git_perms.rs`)

| # | Rule | Site |
|---|---|---|
| BR-GIT-51 | Update kind is classified from the zero-oid and `is_ancestor`: zero-old ⇒ Create, zero-new ⇒ Delete, ancestor ⇒ FastForward, else NonFastForward. | `git_perms.rs:207-226` |
| BR-GIT-52 | Built-in defaults when no pattern matches: branch/tag **create** ⇒ Member; branch **FF** ⇒ Member; tag FF (i.e. tag move) ⇒ Admin; any non-`heads`/`tags` ref ⇒ Admin; **NFF ⇒ Admin; Delete ⇒ Admin**. | `git_perms.rs:403-428` |
| BR-GIT-53 | All matching `buzz-protect` patterns are **unioned**: strictest `push:<role>` wins; `no-force-push`/`no-delete`/`require-patch` are sticky-true. | `git_perms.rs:445-486` |
| BR-GIT-54 | An explicit `push:<role>` can **never weaken** the built-in default — the higher permission level of (explicit, default) is used. | `git_perms.rs:534-550` |
| BR-GIT-55 | `require-patch` denies *all* update kinds on the matched ref, not just fast-forwards. | `git_perms.rs:522-529`, doc `:255-262` |
| BR-GIT-56 | A push is **atomic**: any single denied ref rejects the whole push. | `git_perms.rs:584-600`; hook exits 1 on any 403 (`hook.rs:141-148`) |

---

#### 4. Publish gating (`finalize_push`, `transport.rs:1548-1754`)

| # | Rule | Site |
|---|---|---|
| BR-GIT-57 | `finalize_push` is the **unique** constructor of a push response; both `build_git_response` call sites are inside it. | `transport.rs:1574`, `:1748` |
| BR-GIT-58 | A push publishes nothing unless `PackOutput.ok`, where `ok = subprocess exit 0 && no `ng ` line in report-status`. A pre-receive decline exits **0**, so the report-status scan — not the exit code — is the primary signal. | `transport.rs:1113-1139`, `:1567-1580`; `receive_pack_report_rejected` `:1181-1208` |
| BR-GIT-59 | Report-status parsing must de-frame **two** pkt-line levels: side-band-64k nests the status stream inside a band-1 outer frame. Band 2/3 payloads are never treated as status. | `transport.rs:1181-1208`; pinned `:1989-2064` |
| BR-GIT-60 | On `!ok` the buffered in-band report-status is still returned (200) so the client prints the rejection — only the side effects are suppressed. | `transport.rs:1567-1579` |
| BR-GIT-61 | kind:30618 is emitted **after** a successful CAS, never as the commit, and only when `parent_digest != committed_digest`. | `transport.rs:1662-1683` |
| BR-GIT-62 | kind:30618 build failure, DB insert failure, or DB dedup are all **non-fatal** — the push stays durable and still returns 200. | `transport.rs:1712-1735` |
| BR-GIT-63 | kind:30618 is relay-signed and bypasses `dispatch_persistent_event`, going straight to `db.insert_event` + `fan_out_event_to_local_subscribers`. | `transport.rs:1693-1710` |
| BR-GIT-64 | The workspace tempdir is dropped only **after** the response bytes are built. | `transport.rs:1748-1751` |
| BR-GIT-65 | `CasError` → HTTP mapping is total and typed so `?`-bubbling cannot turn a 412 into a 500: `Conflict`→409, `ManifestInvalid`→400, `ResourceLimit`→413, `{ManifestReadFailed, Backend, PackCapture}`→500. | `transport.rs:1601-1657` |

---

#### 5. Read-path (hydration) rules

| # | Rule | Site |
|---|---|---|
| BR-GIT-66 | Pointer absent ⇒ `Ok(None)` ⇒ 404. Every **below-pointer** failure (non-64-hex pointer body, manifest 404, digest mismatch, unsupported version, oversized object, `index-pack` failure) ⇒ error ⇒ 5xx/413. Never a synthesized empty repo. | `hydrate.rs:98-121`, `:246-270`; `transport.rs:341-355` |
| BR-GIT-67 | Pointer body must be exactly 64 ASCII-hex after trim. | `hydrate.rs:255-260` |
| BR-GIT-68 | Manifests are read digest-verified and then `validate()`d on the read side too, duplicating the write-side predicates. | `hydrate.rs:262-268` |
| BR-GIT-69 | **Phase ordering is load-bearing**: all packs fetched, written, and indexed before any ref file or HEAD is written. A failed hydrate leaves a repo with no advertised refs, never refs pointing at missing objects. | `hydrate.rs:288-292`, `:300-329`, `:351-375` |
| BR-GIT-70 | Refnames and oids are re-validated at ref-write time as defense in depth, and `manifest.head` is re-validated before writing `HEAD`. | `hydrate.rs:337-350`, `:366-372` |
| BR-GIT-71 | A pack key must literally begin `packs/`; the remainder is the expected digest. | `hydrate.rs:303-305` |
| BR-GIT-72 | A first-push (pointer-absent) workspace is `git init --bare` + `symbolic-ref HEAD refs/heads/main` — HEAD defaults to `main`, never to git's environment default. | `hydrate.rs:181-184`; pinned `:508-521` |
| BR-GIT-73 | `index-pack` runs **without** `--strict`: connectivity checks are deliberately skipped because `Inv_Closed` is a write-path invariant. Structural integrity (CRC, type tags) is still validated. | `hydrate.rs:410-416` |
| BR-GIT-74 | An `.idx` sidecar from the store is trusted only after `git verify-pack` passes; on failure it is deleted and regenerated. A missing sidecar or a read error is a cache miss, not a failure. | `hydrate.rs:381-408`, `store.rs:317-334` |
| BR-GIT-75 | The `info/refs` fast path is eligible only when: no `refs/tags/*` at all (annotated tags need a `^{}` peel line the manifest cannot reproduce), `head ∈ refs`, and every emitted refname/oid re-passes `is_safe_refname`/`is_hex_oid` with refname ≤ 4096. Any failure degrades to the subprocess path. | `transport.rs:380-419`; pinned `:2164-2211` |

---

#### 6. Pack-cache rules

| # | Rule | Site |
|---|---|---|
| BR-GIT-76 | Cache keys must be 64 ASCII-hex — the path-traversal guard for the shard/entry directory names. | `pack_cache.rs:458-470`; pinned `:586-593` |
| BR-GIT-77 | The cache parent must not be a symlink. | `pack_cache.rs:118-126`; pinned `:575-585` |
| BR-GIT-78 | Concurrent misses for one digest share a single population *flight* (per-digest async mutex + refcount); a cancelled waiter must not deregister a live flight. | `pack_cache.rs:76-104`, `:186-238`, `:392-399`; pinned `:594-661` |
| BR-GIT-79 | Population concurrency ≤ `git_pack_cache_max_concurrent_populations` (default 2, must be > 0). | `pack_cache.rs:110-113`, `:253-269`; config `config.rs:725-730` |
| BR-GIT-80 | An entry is published only by **atomic directory rename** after both pack and idx exist; a lost rename race is resolved by adopting the winner's directory. | `pack_cache.rs:311-333` |
| BR-GIT-81 | `max_bytes == 0`, or a single entry larger than `max_bytes`, **bypasses** the cache: the entry is served from a `TempDir` owned by the returned `CachedPack` and never indexed. | `pack_cache.rs:303-313`; metric `buzz_git_pack_cache_bypasses_total` |
| BR-GIT-82 | Eviction is least-recently-used by logical tick, and **only entries with `Arc::strong_count == 1`** are evictable; if every entry is in flight the loop breaks and the cache temporarily exceeds `max_bytes`. | `pack_cache.rs:365-390` |
| BR-GIT-83 | A cached pair is installed into the workspace by hard link, falling back to copy (counted). Both destination files are removed before the copy attempt. | `pack_cache.rs:428-456` |
| BR-GIT-84 | At construction, sibling `session-*` directories whose `.heartbeat` mtime is older than 10 minutes are recursively deleted; the live session touches its heartbeat every 60 s. | `pack_cache.rs:20-21`, `:127-146`, `:482-509` |
| BR-GIT-85 | A cache hit whose files have vanished is treated as a miss and (on install failure) invalidated. | `pack_cache.rs:241-251`, `:172-183` |

---

#### 7. The CAS protocol — rules and correctness argument

##### 7.1 Rules

| # | Rule | Site |
|---|---|---|
| BR-GIT-86 | The pointer is the **sole** commit point. `manifests/`, `packs/`, `idx/` writes are create-only and idempotent; a 412 on a content-addressed key is *success* because the key is the digest of the bytes being written. | `store.rs:243-280`, `:294-321` |
| BR-GIT-87 | The caller does **not** choose the content-addressed key — `put_immutable` derives it internally. This is what makes the 412-is-success argument constructive rather than a trust assumption. | `store.rs:243-260` |
| BR-GIT-88 | `get_pointer` returns ETag and body from **one** GET so the CAS predicate and the observed state cannot straddle a concurrent writer. A missing ETag header is a hard error. | `store.rs:446-479` |
| BR-GIT-89 | A 2xx CAS PUT with **no** ETag header is treated as a non-conforming backend and fails the operation, rather than returning `ETag("")`. | `store.rs:511-537` |
| BR-GIT-90 | 412 is a *value* (`CasOutcome::LostRace`), not an error; every other non-2xx bubbles as `StoreError::Backend`. | `store.rs:58-64`, `:538-544`; pinned `:930-943` |
| BR-GIT-91 | The CAS predicate is `parent_state.if_match` — the ETag observed at hydrate time — never a re-read. First push uses `If-None-Match: *`. | `cas_publish.rs:1208-1226` |
| BR-GIT-92 | **No retry on `LostRace`.** The loser re-reads the pointer only to report the winner, then returns `Conflict` ⇒ 409. Retrying in-handler would reuse receive-pack output derived against a superseded parent. | `cas_publish.rs:1260-1291`, `read_winner_after_conflict` `:1294-1329` |
| BR-GIT-93 | **No advisory lock.** Writer serialization is exclusively the CAS. Concurrent same-repo pushes each hydrate and run receive-pack; the loser's work is discarded. | `transport.rs:865-877`; `state.rs:517-521` |
| BR-GIT-94 | `m_after.parent` is the parent's **bare** digest (not `manifests/<digest>`), so `parent == pointer.digest` reads literally. | `manifest.rs:238-242`, `cas_publish.rs:907-929`; pinned `manifest.rs:424-447` |
| BR-GIT-95 | `m_after.packs ⊇ parent.packs` on the normal path, sorted and deduped for byte-stability. | `cas_publish.rs:907-929`; pinned `:1440-1487` |
| BR-GIT-96 | `Manifest::validate()` runs between compose and `put_manifest`, so an un-clone-able manifest becomes a push-time 400 instead of a permanent read-time 5xx. | `cas_publish.rs:1177-1190`; call-site pinned `:1834-1878` |
| BR-GIT-97 | Winner resolution after a lost race fails closed at every step: vanished pointer, non-64-hex body, digest mismatch, or missing manifest all become `ManifestReadFailed` ⇒ 500. | `cas_publish.rs:1294-1329` |
| BR-GIT-98 | The published HEAD is resolved, not copied: observed symref if it names a published branch, else the parent's head if it does, else `refs/heads/main`, else `refs/heads/master`, else the first `refs/heads/*`, else observed, else parent. | `cas_publish.rs:355-381`; pinned `:1379-1396` |
| BR-GIT-99 | Objects the push introduced are captured by `git pack-objects --revs --stdout` with `refs_after` tips positive and **`parent_state.parent.refs`** tips negative. | `cas_publish.rs:486-570` |
| BR-GIT-100 | An empty `pack-objects` output, or an empty `refs_after`, is legitimate (refs-only or delete-all push) and still publishes a new manifest reusing the parent's packs. | `cas_publish.rs:507-511`, `:560-562`, `:1153-1157` |
| BR-GIT-101 | `parent_hydrated_bytes + new_pack_bytes ≤ max_repo_bytes`, with `checked_add` overflow handling. | `cas_publish.rs:1132-1146` |
| BR-GIT-102 | `idx` sidecar write failure is **non-fatal** — a warning; hydrate regenerates. | `cas_publish.rs:1148-1152`, `:828-844` |

##### 7.2 Pack compaction rules

| # | Rule | Site |
|---|---|---|
| BR-GIT-103 | Compaction is attempted when `parent.packs.len() >= PACK_COMPACTION_THRESHOLD` (96) — a quarter of manifest capacity kept in reserve. | `cas_publish.rs:572-574`, `:1046`; pinned `:1499-1513` |
| BR-GIT-104 | Compaction is serialized process-wide by a `Semaphore::const_new(1)` with a 300 s acquire timeout and a 600 s operation timeout. | `cas_publish.rs:86`, `:576-586`, `:82`, `:1058` |
| BR-GIT-105 | Compaction refuses repos with more than 1 000 000 objects (`count-objects -v`, `count:` + `in-pack:`, with `checked_add`). | `cas_publish.rs:85`, `:593-664` |
| BR-GIT-106 | Compaction feeds **deduplicated** `refs_after` tips with **no** negative revisions and `--max-pack-size=max_pack_bytes`, producing a self-contained closure. | `cas_publish.rs:694-727` |
| BR-GIT-107 | The compacted pack set is usable only if it reduces the pack count, or (at the hard cap) matches it while staying within `MAX_MANIFEST_PACKS`. | `cas_publish.rs:588-591`, `:853-875`; pinned `:1515-1533` |
| BR-GIT-108 | A compacted manifest **replaces** the parent's pack list rather than extending it; old objects are never deleted, so readers holding an earlier manifest stay valid. | `cas_publish.rs:931-948`; pinned `:1535-1560` |
| BR-GIT-109 | Compaction failure (any cause) degrades to the normal delta path with a `fallback` metric; but if `packs_before >= 128` **and** a new pack is needed, the push is rejected with the compaction error. | `cas_publish.rs:1096-1103`, `:1120-1131`; metric `buzz_git_pack_compaction_required_failures_total` |
| BR-GIT-110 | Compacted pack bytes are re-checked against the on-disk size after read; a size change between validation and upload is an error. | `cas_publish.rs:812-827` |

##### 7.3 Why the CAS is correct here

**Claim.** For every accepted push, the manifest installed by the winning CAS is derived from exactly the pointer state the workspace was hydrated from; concurrent pushes never lose an update; and no unauthorized ref value can be published.

**(a) Parent is observed once, and the observation is the predicate.** `hydrate_for_write` performs a single `get_pointer` and packages `(ETag, digest, Manifest)` into `ParentState` (`hydrate.rs:207-244` → `cas_publish.rs:245-255`). That value travels on `PushContext.parent_state` (`transport.rs:944-955`) into `cas_publish`, which uses `parent_state.if_match` verbatim as the precondition (`cas_publish.rs:1208-1226`) and `parent_state.parent_digest` verbatim as `m_after.parent` (`:1159-1165`, `:907-929`). There is no code path between hydrate and CAS that re-reads the pointer. So "built on `d_old`, published against `d_new`" is not expressible.

**(b) The delta pack is computed against the same parent.** `capture_pack`'s negative set is `parent_state.parent.refs` (`cas_publish.rs:1112-1119`), i.e. the manifest the workspace was materialized from — and `materialize_manifest` wrote exactly those refs to disk (`hydrate.rs:355-371`). So `refs_before` (manifest) and the workspace's pre-push refs are equal by construction, and `m_after.packs = parent.packs ∪ {new}` covers the reachable closure of `refs_after`.

**(c) At most one CAS predicated on a given ETag succeeds.** Delegated to the backend (axiom A3) and *admitted*, not assumed: `run_conformance_probe` races `race_width` (default 32) writers over `race_rounds` (default 3) rounds for both `If-Match` and `If-None-Match: *`, requiring exactly one winner among *classified* observers, ≥ 2 classified observers, and post-race byte equality (`store.rs:571-877`). It is a **fatal** startup gate (`main.rs:466-503`), opt-out via `BUZZ_GIT_CONFORMANCE_PROBE=false`.

**(d) Transport-unknown outcomes are dropped, not counted.** `S3Error::{Reqwest, Http, Io}` racers never received a classified response, so they are excluded from the observer set and surfaced as `ProbeReport.transport_drops` (`store.rs:126-149`, `:673-700`, `:781-800`). This is what keeps the probe a conformance test rather than a network-stack test, and prevents a false pass at `classified < 2`.

**(e) The loser cannot publish.** `LostRace` is returned as `CasError::Conflict` — a variant distinct from `Backend` — so `?`-bubbling cannot silently convert it (`cas_publish.rs:105-124`, mapped `transport.rs:1605-1618`). No retry loop exists in the handler; `git` re-drives the whole push, which re-hydrates against the advanced pointer.

**(f) The re-read after a loss is allowed to race.** `read_winner_after_conflict` may observe a *third* winner; that is explicitly accepted because the value is diagnostic only and the client re-pushes regardless (`cas_publish.rs:1272-1279`, `:1294-1305`).

**(g) The response fence.** The success 2xx is constructed only after `cas_publish` returns `Ok` (`transport.rs:1582-1660` then `:1748`). `run_git_at` deliberately returns an owned `PackOutput` instead of a `Response` so no earlier code can build a push response (`transport.rs:974-991`). `finalize_push` is a single sequential `async fn` with no detached `tokio::spawn`.

**(h) A false-positive `ok` cannot publish unauthorized refs.** This is the deeper reason the fence is safe, and it is *not* stated in the doc. If report-status parsing ever missed an `ng` line — e.g. a client that never negotiated `report-status`, so receive-pack emits no status at all — `ok` would be `true` and the CAS would run. But a pre-receive decline means git rejected **all** commands (pre-receive is all-or-nothing) and never migrated the quarantined objects, so the workspace refs still equal `parent.refs`. `snapshot_workspace_state` therefore yields `refs_after == parent.refs`, `capture_pack` yields empty, `compose_after` reproduces the parent's `(head, refs, packs)`, and `canonical_bytes` is deterministic ⇒ the same digest ⇒ the CAS installs the identical value and `manifest_changed` is false, so no kind:30618 is emitted. **Ref integrity comes from the workspace, not from the status parse**; `PackOutput.ok` is a cost/noise optimization on top of it.

**(i) The residual hole is pointer *creation*, not ref content.** Nothing in `receive_pack` requires the repo to be announced. A receive-pack request with **zero** ref commands causes git to skip `execute_commands`, so the pre-receive hook never runs, `ok` is true, and `cas_publish` proceeds: for an existing repo it re-installs the same digest under a fresh ETag (a 409-inducing contention vector for concurrent legitimate pushers); for an **absent** pointer it takes `IfNoneMatchStar` and creates a pointer to the canonical empty manifest for an arbitrary `(owner, repo)` — bypassing `git_repo_names` reservation (`handlers/side_effects.rs:2441-2513`) and the per-pubkey quota. This contradicts the doc's "pointer-absence means never announced" (doc §Implementation Correspondence). It cannot publish ref *content* (that requires ≥1 command ⇒ the hook ⇒ kind:30617), and the squatted manifest is byte-identical to the announce seed so a later legitimate announce still succeeds via `seed_manifest_pointer`'s digest-equality check (`handlers/side_effects.rs:2656-2688`). *Inferred from code structure plus git's `receive-pack` semantics; not confirmed by a live behavioral test.*

**(j) One correctness gap in the snapshot.** BR-GIT-111: `snapshot_workspace_state` silently `warn!`s-and-skips any ref whose oid is not exactly 40 hex (`cas_publish.rs:325-328`), while `is_hex_oid` (`manifest.rs:156`), `manifest_event::is_valid_oid` (`:129`), and the advertisement's `object-format` derivation (`transport.rs:473-479`) all accept 64. In a SHA-256 repository every ref would be dropped and the CAS would publish `refs: {}` — a silent full ref deletion that `validate()` accepts. Unreachable today because `init_bare_repo` always creates SHA-1 repos (`hydrate.rs:181-184`), but the failure mode is silent-drop rather than fail-closed.

##### 7.4 Rules the implementation adds beyond the spec

| # | Rule | Site | Spec status |
|---|---|---|---|
| BR-GIT-112 | HEAD is a published manifest field, resolved via a 6-step fallback (BR-GIT-98). | `cas_publish.rs:355-381` | spec's `m` has only `packs` + `refs` |
| BR-GIT-113 | Manifest carries `version` and is rejected on mismatch. | `manifest.rs:263-272` | not in spec |
| BR-GIT-114 | Pack compaction (BR-GIT-103…110). | `cas_publish.rs` | doc §Theorem 2 Remark covers it |
| BR-GIT-115 | `idx/` sidecar tier. | `store.rs:227-334`, `hydrate.rs:381-429` | not in spec |
| BR-GIT-116 | The `info/refs` manifest fast path bypasses hydrate entirely. | `transport.rs:380-537` | not in spec (spec §Read always hydrates) |
| BR-GIT-117 | Byte/count/time limits (BR-GIT-13…30). | throughout | spec is silent on resource bounds |
| BR-GIT-118 | Spec §Push step 4 ("validate Δ against `m_before.refs`") has **no** counterpart inside `cas_publish` — the fast-forward/protection check lives entirely in the pre-receive hook against the *workspace*, which was materialized from `m_before`. Equivalent in effect, different location. | `hook.rs:32-150` + `policy.rs:173-417`; absence in `cas_publish.rs:1021-1292` | delta |
| BR-GIT-119 | `capture_pack`'s comment claims it deduplicates the `X ^X` case; it does not (`cas_publish.rs:494-517` pushes every value unconditionally). `capture_compacted_packs` **does** dedup (`:696-702`). | `cas_publish.rs:490-517` | comment/behavior delta |
