## Module: buzz-relay — git hosting (`crates/buzz-relay/src/api/git`)
### Aspect: Features

#### 1. What this module is

A git server with **no persistent per-repo filesystem**. Every request materializes an ephemeral bare repo from an object-store manifest, shells out to stock `git`, and drops the tree (`hydrate.rs:1-27`, doc §v1 deployment architecture). Refs commit by compare-and-swap on a single manifest pointer (`cas_publish.rs:1208-1226`).

| Capability | Status | Evidence |
|---|---|---|
| Smart HTTP clone / fetch | supported | `transport.rs:786-827` |
| Smart HTTP push | supported | `transport.rs:858-962` |
| Ref advertisement | supported, two implementations | fast path `transport.rs:471-537`; subprocess `:596-724` |
| Dumb HTTP (`objects/info/packs`, loose-object URLs) | **absent** — no routes registered | `transport.rs:1758-1763` |
| `git://` daemon, SSH transport | **absent** | no listener anywhere in the crate |
| Protocol v2 (`Git-Protocol: version=2`) | **not implemented server-side**; the request header is never read, and the fast-path advertisement is a v0 `# service=` stream. The subprocess path inherits whatever the installed `git upload-pack --advertise-refs` does without the env var set, i.e. v0. | `transport.rs:471-537`, `:645-653` |

#### 2. Git operations

| Operation | Supported | Mechanism / limit |
|---|---|---|
| Full clone | yes | subprocess `upload-pack`, streamed (`transport.rs:1414-1498`) |
| Fetch / incremental | yes | same path |
| **Shallow clone** (`--depth`) | yes, via subprocess. The fast-path advertisement offers `shallow deepen-since deepen-not deepen-relative` as a fixed string (`transport.rs:484-491`); the real negotiation happens against `git upload-pack` in the follow-up POST. Exercised in production by the web repo browser (`web/src/features/repos/git-client.ts:86-113`, `depth: 1`). |
| `--single-branch`, `--no-tags` | yes (client-side negotiation, server-transparent) | `web/src/features/repos/git-client.ts:100-112` |
| **Partial clone** (`filter=blob:none`, promisor) | **not offered**. `filter` is absent from the fast-path capability string (`transport.rs:484-494`); the subprocess path may advertise it if the installed git does, but no promisor remote/fetch-object protocol exists on the relay. | |
| **Thin pack** | offered (`thin-pack` in caps, `transport.rs:485`) | |
| `side-band` / `side-band-64k` | offered; report-status parsing explicitly handles the nested band-1 framing | `transport.rs:485`, `:1181-1208` |
| Branch create / fast-forward | yes | `git_perms.rs:403-428` defaults Member |
| **Force push** (non-fast-forward) | yes, Admin by default, deniable per-pattern via `no-force-push` | `git_perms.rs:419-427`, `:545-551`; e2e `crates/buzz-test-client/tests/e2e_git.rs:197` |
| **Branch deletion** | yes, Admin by default, deniable via `no-delete`. `capture_pack` handles delete-all (no positive tips ⇒ `None`) and refs-only deletes. | `git_perms.rs:428`, `:553-559`; `cas_publish.rs:507-511` |
| **Tags** | stored and served, but any `refs/tags/*` **disables the advertisement fast path** because the manifest cannot reproduce an annotated tag's `^{}` peel line. Tag *creation* defaults to Member; tag *move* to Admin. | `transport.rs:388-403`; `git_perms.rs:410-424` |
| Atomic push (`atomic` capability) | not offered by the fast path (receive-pack advertisement always goes through the subprocess, so whatever git offers applies). Policy evaluation is nonetheless all-or-nothing. | `transport.rs:549-551`; `git_perms.rs:584-600` |
| **Git LFS** | **absent** — no `objects/batch` endpoint, no `lfs` string anywhere in the module. | |
| **Submodules** | no special handling; gitlinks are ordinary objects inside packs. No recursive fetch support, no `.gitmodules` awareness. | |
| **Signed pushes** (`push-cert`) | not offered; `receive-pack` runs without `--signed`. Object-level signing is a separate concern handled by the `git-sign-nostr` crate. | `transport.rs:1019-1027` |
| **Server-side hooks other than pre-receive** | absent. Only `hooks/pre-receive` is installed; `update`, `post-receive`, `post-update` are never written. | `hook.rs:152-178` |
| **Repository deletion / rename** | absent. No protocol path deletes a pointer or a repo; the doc's "no deletion under the protocol" rule (doc §Axioms A1) is upheld by omission. | |
| Garbage collection / repack of stored objects | only proactive **pack compaction** during an accepted push; old packs are never deleted. Physical pruning is an out-of-band retention concern. | `cas_publish.rs:666-875`, doc §Theorem 2 Remark |
| SHA-256 repositories | advertised as derivable (`object-format=sha256` from oid width, `transport.rs:473-479`) and accepted by `is_hex_oid`, but **not usable**: `init_bare_repo` always creates SHA-1 (`hydrate.rs:181-184`) and `snapshot_workspace_state` drops non-40-hex oids (`cas_publish.rs:325-328`). |

#### 3. Performance / operability features

| Feature | Description | Site |
|---|---|---|
| Manifest advertisement fast path | Serves `info/refs` for `git-upload-pack` straight from the verified manifest — no hydrate, no subprocess, **no semaphore permit**. Gated by `fast_path_eligible`, which doubles as a safety re-check. Byte-compatible with `git upload-pack --advertise-refs` for the branches-only case. | `transport.rs:380-537`, `:552-577` |
| Streamed fetch | `upload-pack` stdout is streamed into the HTTP body instead of buffered; the child, the tempdir, the stdin pump, and the semaphore permit are all kept alive until EOF or client disconnect. | `transport.rs:1262-1332`, `:1414-1498` |
| Streaming deadline + metrics | 300 s in-band deadline that kills the subprocess; `buzz_git_upload_pack_{timeouts_total, stream_seconds, stream_bytes}`. | `transport.rs:1282-1391` |
| Bounded pack/index cache | Process-local, digest-keyed, byte-bounded, LRU-evicted, single-flight populated, hard-linked into workspaces. | `pack_cache.rs:66-456` |
| Cross-process cache adoption | A lost publication rename is resolved by adopting the winner's directory — designed for several relay processes sharing a scratch volume. | `pack_cache.rs:314-330` |
| Stale-session GC | Abandoned `session-*` cache dirs older than 10 min are removed at startup. | `pack_cache.rs:482-509` |
| Proactive pack compaction | At ≥ 96 parent packs an accepted push captures the full post-push closure into ≤ 128 bounded packs and replaces the pack list. | `cas_publish.rs:1046-1103`, `:666-875` |
| gzip request-body inflation | Git gzips large negotiation bodies; the relay inflates transparently with an independent decoded-byte cap. Without it the subprocess dies with `bad line length character`. | `transport.rs:726-784` |
| Startup conformance gate | Fatal A1/A3 probe against the configured bucket before the listener opens. | `store.rs:571-877`, `main.rs:466-503` |
| Observability | 8 counters, 11 histograms, 3 gauges under `buzz_git_*`. | see integrations aspect |

#### 4. Authorization features

| Feature | Description | Site |
|---|---|---|
| NIP-98 transport auth | Repo-root-scoped, tenant-host-bound, ±60 s window. Method and body-hash binding intentionally waived. | `transport.rs:76-231` |
| NIP-OA managed-agent delegation | An agent's attestation may travel in the signed NIP-98 event's `auth` tag (git cannot carry `x-auth-tag` through the credential helper). | `transport.rs:204-210` |
| Managed-agent owner authority | A verified NIP-OA *owner* of the repo-owner key gets `MemberRole::Owner` on that repo. | `policy.rs:343-368` |
| Channel-bound roles | Non-owners derive their role from the channel named by kind:30617's `buzz-channel` tag. | `policy.rs:308-395` |
| Bot promotion | `MemberRole::Bot` → `Member` for git pushes only. | `policy.rs:397-402` |
| Ref protection via `buzz-protect` tags | Glob patterns (≤ 3 wildcards, ≤ 256 chars, ≤ 50 rules) with `push:<role>`, `no-force-push`, `no-delete`, `require-patch`; unioned strictest-wins; explicit rules can never weaken defaults. | `git_perms.rs:244-262`, `:445-486`, `:534-550` |
| Archived-channel read-only | An archived bound channel denies **all** pushes, owner included. | `policy.rs:308-330` |
| Per-pubkey repo quota | Default 100, enforced at announce. | `handlers/side_effects.rs:2486-2492` |
| Repo-name reservation | `(community_id, repo_id)` unique in Postgres via `INSERT … ON CONFLICT DO NOTHING`. | `handlers/side_effects.rs:2441-2513` |
| **Read authorization** | **absent.** No per-repo, per-channel, or visibility check on `info/refs`/`upload-pack`. `transport.rs:60-66` claims "No public repos for v1", but in practice every repo is readable by any caller that clears the transport gate. | absence in `transport.rs:539-827` |

#### 5. Repo browser surface

The relay does not render repository content itself. Two cooperating surfaces:

| Surface | Mechanism | Site |
|---|---|---|
| Repo/branch discovery | `POST /query` for kinds 30617 (repos) and 30618 (ref state) — plain Nostr, not this module. The web client explicitly notes it must not trust arbitrary authors' 30618. | `web/src/features/repos/use-repos.ts:66`, `use-repo-refs.ts:54-57` |
| File/tree/blob browsing | **client-side**: `isomorphic-git` over LightningFS/IndexedDB does a `depth:1, singleBranch, noTags` clone of `/git/{owner}/{repo}.git` with a NIP-98 header, then reads trees/blobs locally. | `web/src/features/repos/git-client.ts:17-140` |
| SPA route gating | The relay serves `/`, `/repos`, `/repos/*` from the web bundle only when `BUZZ_SERVE_GIT_WEB_GUI` is truthy. | `router.rs:206-213`; config `config.rs:848-850` |

Consequences: no server-side tree/blob/commit API exists, browsing costs a real clone per repo per browser, and the browser's IndexedDB holds full repo content. There is no server-side diff, blame, or search over repository contents.

#### 6. Not implemented (explicit non-features)

- No repo delete/rename/transfer protocol.
- No `receive-pack` `--signed` / push certificates.
- No `update`/`post-receive` hooks, so no server-side CI trigger from git itself (workflows are driven by kind:30618 events instead).
- No mirror/replication, no cross-relay fetch.
- No pack-content policy: received packs are never inspected for object types, sizes, or path names beyond what `git index-pack` validates.
- No per-repo read ACL, no anonymous/public read mode (both directions of that missing knob: no anonymous access, and no way to *restrict* read below community membership).
- No liveness/latency guarantee — explicitly out of scope in the design (doc §Scope and Non-Goals).
