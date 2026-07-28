## Module: buzz-relay — event ingest & side effects (`crates/buzz-relay/src/handlers`)
### Aspect: Data Model

This group owns no tables. It is the **write orchestrator**: it decides which rows are
written, in what order, and inside which transaction boundary. Below: what gets written,
the in-memory types that carry it, and the materialized-counter contract.

---

### 1. Tables written, by write path

| Table | Written by | Transaction boundary |
|---|---|---|
| `events` | `insert_event_with_thread_metadata` (`ingest.rs:2394`), `replace_addressable_event` (`ingest.rs:2371`), `replace_parameterized_event` (`ingest.rs:2385`), `insert_reaction_event_with_thread_metadata` (`ingest.rs:2298`), `insert_event` (`side_effects.rs:692`, `:868`, `:2752`, `:2911`, `:3186`), raw `INSERT` (`command_executor.rs:196-232`) | one tx per call, except the command path (see §6) |
| `thread_metadata` | same tx as the `events` insert, via `insert_event_with_thread_metadata_tx` (`buzz-db/src/event.rs:1004`) | atomic with the event |
| `reactions` | `insert_reaction_event_with_thread_metadata` (`ingest.rs:2298`); removal in `side_effects.rs:2175`, `:2216` | atomic with the kind:7 event |
| `channels` | `create_channel_with_id` (`ingest.rs:2103`), `create_channel` (`side_effects.rs:1692`, `:1719`), `update_channel` (`side_effects.rs:1345`, `:1360`, `:1424`, `:1466`), `archive_channel` / `unarchive_channel` (`:1485`, `:1499`), `soft_delete_channel` (`ingest.rs:2408`, `side_effects.rs:1789`), `open_dm` (`command_executor.rs:398`, `:534`) | separate tx per call |
| `channel_members` | `add_member` (`side_effects.rs:1216`, `:1868`), `remove_member` (`:1293`, `:1932`) | separate tx |
| `users` | `ensure_user` (`side_effects.rs:1096`, `:1140`, `command_executor.rs:49`), `update_user_profile` (`side_effects.rs:1163`, `:1182`), `set_channel_add_policy` (`side_effects.rs:1105`) | separate tx |
| `relay_members` | `remove_relay_member` (`ingest.rs:1857`); 9030–9033 delegate to `relay_admin` | separate tx |
| `moderation_reports` | `report::handle_report_event` (`ingest.rs:1562`) | out of module |
| product-feedback table | `product_feedback::handle` (`ingest.rs:1541`) | out of module |
| `push_leases` | `push_lease::accept` (`ingest.rs:2157`) | out of module |
| `archived_identities` | `identity_archive::handle_identity_archive_event` (`ingest.rs:1916`) | out of module |
| `workflows` | `upsert_workflow` (`command_executor.rs:775`), `delete_workflow_for_owner` (`side_effects.rs:2000`, `:2026`) | pool, **outside** the command tx |
| `workflow_runs` | `create_workflow_run` (`command_executor.rs:918`), `update_workflow_run` (`:1177`, `:1276`, `:1305`) | pool |
| `workflow_approvals` | `update_approval_by_stored_hash` (`command_executor.rs:1041`, `:1153`) | pool |
| `git_repo_names` | `reserve_repo_name` (`side_effects.rs:2500`), `release_repo_name` (`side_effects.rs:2543`) | separate tx; `ON CONFLICT DO NOTHING` is the TOCTOU guard |
| object store (git) | `put_manifest` / `put_pointer` / `get_pointer` (`side_effects.rs:2642-2726`) | CAS, not SQL |
| `audit_log` | indirectly, via `dispatch_persistent_event` → `enqueue_event_created_audit` (`handlers/event.rs:336`, `:534-577`) | bounded mpsc, capacity 1000 |
| `mentions` | indirectly, inside `Db::insert_event_with_thread_metadata` (`buzz-db/src/lib.rs:1394-1399`) — best-effort, `warn!` on failure | same call, non-transactional |

---

### 2. Ingest write order (the happy path, kind 9 with a reply)

1. `get_channel` prefetch — one SELECT reused by three later gates (`ingest.rs:1738-1741`).
2. `channel_visibility_cached` — resolves fan-out visibility once, bundled with
   `(community, channel)` so it cannot be applied to the wrong pair
   (`ingest.rs:1750-1760`, type `crate::state::ThreadedChannelVisibility`).
3. `is_member_cached` (+ open-visibility fallback) — authorization read (`ingest.rs:1785`).
4. Blob existence reads for every `imeta` tag (`ingest.rs:2215`).
5. `get_event_by_id` + `get_thread_metadata_by_event` for the reply parent, issued
   **concurrently** via `tokio::join!` (`ingest.rs:606-611`); possibly a third
   `get_event_by_id` for the root (`:634`, `:665`).
6. **Write**: `insert_event_with_thread_metadata` — one transaction containing the
   `events` row, the `thread_metadata` row, parent/root stub rows, `reply_count + 1` on
   the parent and `descendant_count + 1` on the root
   (`buzz-db/src/event.rs:1090-1191`).
7. `insert_mentions` — same call, **not** in the transaction, `warn!` on failure
   (`buzz-db/src/lib.rs:1394-1399`).
8. `handle_side_effects` — separate, non-transactional, failure only `warn!`-logged
   (`ingest.rs:2434-2441`).
9. `emit_live_thread_summary` — spawned task; re-reads counters and publishes a
   relay-signed kind:39005 (`ingest.rs:2448-2455`).
10. `enqueue_event_created_audit` — awaited (backpressure by design)
    (`handlers/event.rs:336-343`).
11. `dispatch_persistent_event` — spawns Redis publish + local fan-out + workflow trigger
    (`ingest.rs:2489-2497`).

Steps 6 and 7 are the only durable writes to `events`; steps 8–11 are all
post-commit and best-effort.

---

### 3. In-memory types

#### `IngestAuth` (`ingest.rs:63-86`)

| Variant | Fields | Populated by |
|---|---|---|
| `Nip42` | `pubkey: nostr::PublicKey`, `scopes: Vec<Scope>`, `channel_ids: Option<Vec<Uuid>>`, `conn_id: Uuid` | `handlers/event.rs:697-702` |
| `Http` | `pubkey`, `scopes: Vec<Scope>`, `auth_method: HttpAuthMethod` | `api/bridge.rs:826-829` |

Verified population reality: `scopes` is always `Scope::all_known()` (16 scopes) on both
paths (`buzz-auth/src/lib.rs:137`, `api/bridge.rs:827`); `channel_ids` is always `None`
(`buzz-auth/src/lib.rs:138`); `auth_method` is always `Nip98` (`api/bridge.rs:828`) —
`HttpAuthMethod::DevPubkey` (`ingest.rs:57`) is never constructed in production.

#### `IngestResult` / `IngestError` (`ingest.rs:166-184`)

`IngestResult { event_id: String (hex), accepted: bool, message: String }`. `message` is a
tagged union encoded as a string prefix: `""` (plain success), `"duplicate:"` /
`"duplicate: …"` (conflict or replay), `"info: …"` (28936), `"response:{json}"` (command
kinds), `"{}"` (41012). ⚠ There is no type for this — callers string-match
(`buzz-cli/src/commands/mem.rs:105`).

`IngestError` has 3 variants mapped per transport at `api/bridge.rs:842-871`:
`Rejected` → 400, `AuthFailed` → 403, `Internal` → 500.

#### `ThreadMetadataOwned` (`ingest.rs:535-561`)

Owned mirror of `buzz_db::event::ThreadMetadataParams<'_>`, needed because the borrowed
form cannot outlive the resolution step. Fields: `event_id`, `event_created_at`,
`channel_id`, `parent_event_id`, `parent_event_created_at`, `root_event_id`,
`root_event_created_at`, `depth: i32`, `broadcast: bool`. `as_params()` (`:548`) always
sets the parent/root `Option`s to `Some` — this type only exists for actual replies.

`broadcast` is derived from a `["broadcast","1"]` tag (`ingest.rs:695-698`); `depth` is
`parent_meta.depth + 1`, or the heuristic 1/2 when the parent has no `thread_metadata`
row yet (`ingest.rs:660`).

#### `ReactionChannelResult` (`ingest.rs:322-328`)

Five outcomes distinguishing "target is global" (`NoChannel` → store globally) from
"target missing" (`NotFound` → reject) — the distinction that makes a reaction to a
global kind:1 legal while a reaction to a nonexistent id is not.

#### `PersistResult` (`command_executor.rs:80-85`)

`Duplicate` | `Inserted(sqlx::Transaction<'static, Postgres>)`. The **open transaction is
the return value** — the caller must commit it after running domain mutations. Dropping it
rolls back.

---

### 4. The imeta metadata model (`imeta.rs`)

An `imeta` tag is a Nostr tag whose elements after `"imeta"` are `"key value"` strings
(single-space split, `splitn(2, ' ')` — `imeta.rs:46-49`), i.e. a flattened key/value map
inside a tag array.

| Key | Type / grammar | Mandatory | Singleton | Validated at |
|---|---|---|---|---|
| `url` | local `/media/{64-hex}.{ext}` or `{media_base_url}/{64-hex}.{ext}`; never a `.thumb.` path | ✅ | ✅ | `imeta.rs:59-68` |
| `m` | well-formed `type/subtype`, ≤255 chars, no whitespace/control | ✅ | ✅ | `imeta.rs:69-81`, `is_well_formed_mime` `:340-349` |
| `x` | 64-char lowercase hex SHA-256 | ✅ | ✅ | `imeta.rs:82-89` |
| `size` | positive `u64` (0 rejected) | ✅ | ✅ | `imeta.rs:90-96` |
| `dim` | opaque (allowed, unvalidated) | ❌ | ✅ | allowlist only |
| `blurhash` | opaque | ❌ | ✅ | allowlist only |
| `alt` | opaque | ❌ | ✅ | allowlist only |
| `thumb` | local `.thumb.jpg` path | ❌ | ✅ | `imeta.rs:97-105` |
| `fallback` | opaque — **the only allowed non-singleton key** | ❌ | ❌ | `imeta.rs:11-18` |
| `duration` | positive finite `f64`; video/mp4 only | ❌ | ✅ | `imeta.rs:106-115`, `:167-176` |
| `bitrate` | positive `u64`; video/mp4 only | ❌ | ✅ | `imeta.rs:116-118` |
| `image` | local `/media/` path with ext ∈ {jpg,png,gif,webp}, not a thumbnail; video/mp4 only | ❌ | ✅ | `imeta.rs:119-137` |
| `filename` | 1–255 chars, no `/`, `\`, or control chars — display-only, never influences the content-addressed storage key | ❌ | ✅ | `imeta.rs:138-155` |

13 allowed keys, 12 singletons (`imeta.rs:11-18`).

**Internal consistency rules** (`imeta.rs:178-200`):
- hash embedded in `url` must equal `x`;
- for the 5 "previewable" MIMEs (`image/jpeg`, `image/png`, `image/gif`, `image/webp`,
  `video/mp4` — `imeta.rs:24-30`) the url extension must equal `mime_to_ext(m)` exactly;
- for any other MIME the ext is not derivable, so equality is deferred to the sidecar
  cross-check;
- `thumb`'s embedded hash must equal `x`.

**Storage cross-check** (`verify_imeta_blobs` `imeta.rs:209-335`) — five reads per tag:
1. sidecar must exist for `x`;
2. `HEAD {x}.{sidecar.ext}` must exist;
3. claimed `m` == `sidecar.mime_type`, claimed `size` == `sidecar.size`, claimed
   `duration` within 0.1 s of `sidecar.duration_secs`;
4. if `thumb` claimed, `HEAD {x}.thumb.jpg`;
5. if `image` claimed, resolve its sidecar (MIME must be an image type, ext must match)
   and `HEAD` the poster blob.

The sidecar is the authority for `ext`; `is_local_media_url` (`imeta.rs:373-418`) only
performs a structural gate, rejecting `?`, `#`, and `%` and accepting either
`{64hex}.{safe_ext}` or `{64hex}.thumb.jpg`. `is_safe_ext` is shared with the serve path
so the predicate cannot drift (`imeta.rs:377-379`).

---

### 5. Command-executor input/output types

| Kind | Inputs read from the event | Output `message` |
|---|---|---|
| 41010 | 1–8 `p` tags → `Vec<Vec<u8>>` participants, deduped with self prepended (`command_executor.rs:325-345`) | `response:{"channel_id":"<uuid>","created":<bool>}` (`:432-440`) |
| 41011 | `h` → `Uuid`; `p` tags merged into existing member set; ≤9 total | `response:{"channel_id":"<uuid>"}` (`:571-577`) |
| 41012 | `h` → `Uuid` | `{}` (`:648`) |
| 30620 | `h` → `Uuid`; `d` → workflow `Uuid`; optional `name` tag; `content` = workflow YAML → `(WorkflowDef, definition_json)`; SHA-256 of the **post-secret-injection** JSON as `definition_hash` (`:306-308`, `:747-751`) | `response:{"workflow_id":"<uuid>"[,"webhook_secret":"…"]}` (`:797-805`) |
| 46020 | `d` or `e` → workflow `Uuid`; `content` (if a JSON object) → `TriggerContext.webhook_fields: HashMap<String,String>`, non-string values stringified (`:889-902`) | `response:{"run_id":"<uuid>"}` (`:951-957`) |
| 46030 / 46031 | `d` or `e` → hex → `token_hash: Vec<u8>`; `content` → optional approver note | `response:{"status":"granted"\|"denied","run_id":"<uuid>"}` (`:1088-1095`, `:1315-1322`) |

Tag extractors (all "first match wins", all comparing `tag.kind().to_string()`):
`extract_p_tags` `:235`, `extract_h_tag` `:250`, `extract_d_tag` `:261`,
`extract_e_tag` `:272`, `extract_tag` `:283`. `decode_pubkey` `:294` enforces exactly
32 bytes. `compute_definition_hash` `:306` is raw SHA-256 bytes.

`persist_command_event` writes 11 columns of `events` by hand
(`command_executor.rs:196-217`): `community_id, id, pubkey, created_at, kind, tags,
content, sig, received_at, channel_id, d_tag`. ⚠ It bypasses
`Db::insert_event_with_thread_metadata`, so command events get **no** `thread_metadata`
row and **no** `mentions` row. Harmless today (no command kind is threaded or
mention-bearing), but it is a schema-drift hazard: a new `events` column with a NOT NULL
default handled inside `buzz-db` would not be applied here.

---

### 6. Transaction boundaries — precise

| Path | Atomic set | Non-atomic tail |
|---|---|---|
| Generic insert | `events` + `thread_metadata` + counter updates (`buzz-db/src/event.rs:1004-1191`) | `mentions`, side effects, audit, fan-out |
| Reaction | `reactions` upsert + `events` insert + `thread_metadata` (`buzz-db/src/event.rs:1201-1275`) | `mentions`, fan-out |
| Replaceable / NIP-33 | old-row soft-delete + new insert inside `replace_*_event` | everything after |
| 9007 create-group | ⚠ **two** transactions: `create_channel_with_id` (`ingest.rs:2103`) then the event insert (`:2394`). Compensated manually — an insert failure triggers `soft_delete_channel` + `invalidate_channel_deleted` (`ingest.rs:2404-2414`) | side effects |
| Command kinds | `events` row only (`command_executor.rs:196`) | ⚠ **all domain mutations** — `open_dm`, `hide_dm`, `upsert_workflow`, `create_workflow_run`, `update_approval_by_stored_hash` all run on the pool before `tx.commit()`; the module documents this at `:92-98` |
| Deletion | soft-delete + counter decrement (`soft_delete_event_and_update_thread`) | reaction-row removal, system message, 39005 |
| 30617 announce | ⚠ **three** stores: Postgres name reservation, object-store manifest, object-store pointer. Manual compensation: only a fresh `Reserved` claim releases the name on pointer failure (`side_effects.rs:2528-2555`) | kind:30618 emission |

---

### 7. Materialized thread counters — full audit

Columns: `thread_metadata.reply_count`, `thread_metadata.descendant_count`,
`thread_metadata.last_reply_at` (`buzz-db/src/thread.rs:49-51`).

**Semantics** (`buzz-db/src/thread.rs:241-249`): `reply_count` counts *direct children*;
`descendant_count` counts *all* descendants at every depth. The root's
`descendant_count` is bumped even when `root == parent`.

#### Every insert path, and whether it touches the counters

| Insert path | Passes thread metadata? | Counters updated? | Correct? |
|---|---|---|---|
| `ingest.rs:2394` `insert_event_with_thread_metadata` (generic) | yes, when `thread_meta.is_some()` | ✅ same tx (`buzz-db/src/event.rs:1090-1188`) | ✅ the only reply-bearing path in ingest |
| `ingest.rs:2298` `insert_reaction_event_with_thread_metadata` | `thread_params` passed but always `None` in practice — kind 7 is not in `requires_h_channel_scope` (`ingest.rs:455-491`), so `thread_meta` is `None` at `:2220-2231` | n/a | ✅ reactions are not replies |
| `ingest.rs:2371` `replace_addressable_event` | **no such parameter** | never | ✅ today — no `is_replaceable` kind is in `requires_h_channel_scope` |
| `ingest.rs:2385` `replace_parameterized_event` | **no such parameter** | never | ✅ today — no 30000–39999 kind is in `requires_h_channel_scope` |
| `command_executor.rs:196` raw `INSERT` | no | never | ✅ no command kind is threaded |
| `side_effects.rs:692` `emit_system_message` → `insert_event` | no | never | ✅ kind 40099 is a channel-level notice, never a reply |
| `side_effects.rs:868`, `:2752`, `:2911`, `:3186` `insert_event` (44100/44101, 30618, 8000/8001, 8002/8003) | no | never | ✅ all global (`channel_id = None`) |
| `workflow_sink.rs:331` `insert_event_with_thread_metadata` | yes, with `depth: 0`, `parent: None`, `root: None` and an explicit comment "Workflow messages are always top-level" (`workflow_sink.rs:318-329`) | no increment (guarded on `parent_event_id.is_some()`) | ✅ |

#### Decrement paths

| Path | Function | Counters |
|---|---|---|
| kind 9005 | `side_effects.rs:1624` `soft_delete_event_and_update_thread` | ✅ same tx |
| kind 5 (per target) | `side_effects.rs:2147` same | ✅ same tx |

Both re-emit a kind:39005 afterwards so live badge counts move **down**
(`side_effects.rs:1638-1645`, `:2158-2168`).

#### Verdict

**Every reply-insert path reachable in production updates the counters, and every
delete path decrements them.** The AGENTS.md invariant holds today.

Two latent traps:
1. `replace_addressable_event` and `replace_parameterized_event` have **no thread-metadata
   parameter at all**. `thread_meta` is computed at `ingest.rs:2220-2231` and then
   *silently dropped* on those two branches (`:2367-2390`). Adding any replaceable kind to
   `requires_h_channel_scope` would lose thread ancestry with no compile error and no
   warning. There is no test asserting the two predicates are disjoint from the
   replaceable ranges — only that `is_global_only_kind` and `requires_h_channel_scope` are
   disjoint from each other (`ingest.rs:2753-2762`).
2. `buzz_db::thread::increment_reply_count` (`buzz-db/src/thread.rs:251-287`) has **zero
   callers anywhere in the workspace**, and `Db::decrement_reply_count`
   (`buzz-db/src/lib.rs:2088-2095`) likewise. `Db::insert_thread_metadata`
   (`buzz-db/src/lib.rs:1973`) is only reached from `#[cfg(test)]` code
   (`buzz-db/src/thread.rs:1315`, inside the test module that starts at `:810`). Three
   dead counter-mutation entry points remain available for a future caller to pick up and
   use *outside* a transaction, which is exactly the inconsistency the transactional
   variants exist to prevent (the rationale is spelled out at
   `buzz-db/src/thread.rs:111-114`).

#### Thread summary read shape

`get_thread_summary` (`buzz-db/src/lib.rs:2049`) yields `reply_count`,
`descendant_count`, `last_reply_at`, `participants: Vec<Vec<u8>>`, rendered as the
kind:39005 content JSON at `side_effects.rs:751-756`:

```json
{"reply_count":n,"descendant_count":n,"last_reply_at":<unix|null>,"participants":["<hex>",...]}
```

with tags `["e",root]`, `["d",root]`, `["h",channel]` (`side_effects.rs:757-761`). Same
contract as the channel-window page overlay in `api/bridge.rs` — documented as "one
contract, two delivery doors" (`side_effects.rs:748-750`), matching
`docs/nips/NIP-CW.md:129-133` and `docs/bridge-channel-window.md:99`.

---

### 8. Relay-signed events this module mints

| Kind | Emitter | Storage | Replacement key |
|---|---|---|---|
| 39000 group metadata | `emit_group_discovery_events` `side_effects.rs:962` | channel-scoped | `(kind, relay_pubkey, channel_id)` addressable |
| 39001 group admins | same | channel-scoped | same |
| 39002 group members | same | channel-scoped | same |
| 39005 thread summary | `emit_live_thread_summary` `side_effects.rs:724` | **never stored** — fan-out only | n/a |
| 40099 system message | `emit_system_message` `side_effects.rs:677` | channel-scoped, plain insert | n/a |
| 44100 / 44101 membership notification | `emit_membership_notification` `side_effects.rs:817` | **global** (`channel_id = None`) so global subscribers see it; `h` tag carries the channel as metadata | n/a |
| 8000 / 8001 NIP-43 delta | `publish_nip43_delta` `side_effects.rs:2881` | global, NIP-70 `["-"]` | n/a |
| 13534 NIP-43 membership list | `publish_nip43_membership_locked` (DB-side) `side_effects.rs:2866` | global addressable | per-community advisory lock covers read+build+write |
| 8002 / 8003 NIP-IA delta | `publish_nipia_delta` `side_effects.rs:3142` | global; tags `["-"]`, `["p",target]`, `["consent",path,actor]`, `["e",request_id]`, optional `reason` / `replaced-by` | n/a |
| 13535 NIP-IA archived list | `publish_nipia_archival_list` `side_effects.rs:3008` | global addressable | `(kind, relay_pubkey)` |
| 30622 NIP-DV DM visibility | `publish_dm_visibility_snapshot` `side_effects.rs:3058` | global, `d` = viewer hex, `p` = viewer (so the `#p`-gated read path scopes it to its owner), one `h` per hidden DM | `(kind, relay_pubkey, d=viewer)` |
| 30618 git ref state | `emit_initial_ref_state` `side_effects.rs:2733` | global | `(kind, relay_pubkey, d=repo_id)` |

Note the inversion: 39000–39002 are stored **channel-scoped** so private member lists
inherit channel access control, at the documented cost that live global subscriptions
(`{kinds:[39000]}`) never receive them (`side_effects.rs:952-960`). 44100/44101 go the
other way — stored **global** precisely so an agent's always-live global subscription can
learn about a new channel.

---

### 9. Caches invalidated (module-level consistency surface)

| Cache | Invalidator | Trigger |
|---|---|---|
| membership | `state.invalidate_membership` | 9000 `side_effects.rs:1217`, 9001 `:1294`, 9007 `:1745`, 9021 `:1878`, 9022 `:1933`, DM open `command_executor.rs:383`, DM add `:543` |
| accessible channels | `state.invalidate_all_accessible_channels` | 9002 visibility flip `side_effects.rs:1426`, 9007 open channel `:1749` |
| channel visibility | `state.invalidate_channel_visibility` | 9002 visibility flip `side_effects.rs:1427` |
| channel-deleted (both membership + accessible) | `state.invalidate_channel_deleted` | 9008 `side_effects.rs:1811`, 9007 insert-failure compensation `ingest.rs:2412` |
| workflow-by-channel | `workflow_engine.invalidate_channel_workflows` | 30620 upsert `command_executor.rs:783`, a-tag workflow delete `side_effects.rs:2004`, `:2030` |
| `author_type_cache` | never invalidated | populated at `ingest.rs:1348-1354`; metric-labelling only, explicitly "never used for authorization" (`ingest.rs:1319-1321`) |
| `local_event_ids` (echo dedupe) | `mark_local_event` then `invalidate` on publish failure | `side_effects.rs:784-798`, `:862-878` |
