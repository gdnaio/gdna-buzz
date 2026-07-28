## Module: buzz-sdk (`crates/buzz-sdk`)

### Aspect: Business Rules

All rules execute **before** the caller signs — every builder returns
`Result<EventBuilder, SdkError>` and short-circuits on the first violation
(`crates/buzz-sdk/src/lib.rs:11-13`).

---

### 1. Shared validation primitives

| Rule | What it enforces | File:line | Trigger |
|---|---|---|---|
| `check_content` | `content.len()` (bytes) ≤ `max`, else `ContentTooLarge{max,got}` | `builders.rs:35-41` | every content-bearing builder |
| `check_hex_len` | ≥ `min_len` chars **and** all ASCII hex, else `InvalidDiffMeta` | `builders.rs:44-52` | `build_diff_message` commit/parent, `build_add_member`/`build_remove_member` pubkeys |
| `check_commit_hex` | length is exactly 40 (SHA-1) **or** 64 (SHA-256) and all hex; abbreviated refs rejected | `builders.rs:59-66` | patch/status/PR commit, parent-commit, euc, merge-commit, merge-base, applied-as-commits |
| `check_pubkey_hex` | exactly 64 hex chars; returns lowercased | `builders.rs:69-77` | observer frame, git owner/recipients, workflow delete, DM, moderation, identity archival, auth tag |
| `check_hex_exact` | exactly `len` hex chars; returns lowercased | `builders.rs:79-89` | patch `reply_to`, status `root_event`/`accepted_revision_root`/`applied_patch`, PR `revision_of`, `pr_event`, `report_event_id` |
| `check_repo_id` | non-empty, ≤64 chars, charset `[A-Za-z0-9._-]`, no leading `.`, no `..` | `builders.rs:92-121` | `build_repo_announcement`, `build_repo_announcement_with_tags`, and every `GitRepoCoord::to_a_tag_value` call |
| `normalize_custom_emoji_shortcode` | trims whitespace and `:`; non-empty; ≤64 bytes; charset `[A-Za-z0-9_-]`; lowercases (prevents `party_parrot` vs `Party_Parrot` collision) | `builders.rs:127-150` | `build_custom_emoji_reaction`, `build_custom_emoji_set` |
| `check_custom_emoji_url` | non-empty, ≤2048 bytes, must start `http://` or `https://` | `builders.rs:152-170` | same two builders |
| `check_reason` | ≤64 **UTF-8 bytes** (`MAX_REASON_BYTES`), no control characters | `builders.rs:1704-1721` | identity archive/unarchive `reason` |
| `check_auth_tag_shape` | element 0 == `"auth"`, element 1 is 64-hex pubkey, element 3 is 128-hex signature (structure only; relay does full verification) | `builders.rs:1723-1737` | identity archive/unarchive `auth` |

---

### 2. Tag-construction rules

| Rule | Enforcement | File:line | Trigger |
|---|---|---|---|
| NIP-29 channel scoping via `h` tag | every channel-scoped builder emits `["h", channel_uuid]` as the **first** tag; no `e`-tag substitution | `builders.rs:224`, `283`, `297`, `339`, `382`, `415`, `438`, `456`, `530`, `569`, `586`, `596`, `630`, `653`, `662`, `689`, `704`, `711`, `720`, `728`, `1476`, `1494`, `1562` | any channel operation |
| NIP-10 threading markers | `root == parent` ⇒ single `["e",root,"","reply"]`; `root ≠ parent` ⇒ `["e",root,"","root"]` + `["e",parent,"","reply"]` | `builders.rs:173-186` (documented `lib.rs:24-27`) | `build_message`, `build_forum_comment`, `build_diff_message` |
| Flat reply model for kind 1 | `build_note` emits only `["e",id,"","reply"]` — full NIP-10 root/reply/p-tags explicitly deferred | `builders.rs:732-748` | global text notes |
| Mention dedup + cap | > `MENTION_CAP` (50) raw entries ⇒ `TooManyMentions`; cap is checked **before** dedup, then remaining entries lowercased and deduped into `p` tags | `builders.rs:188-201`; cap `mentions.rs:38` | any builder taking `mentions` |
| Community-command events carry **no** `h` tag | moderation kinds 9040–9044 intentionally omit `h`; tenant is bound by connection host, a stray `h` would be rejected relay-side | `builders.rs:1587-1595` (comment), builders `1597-1690` | moderation commands |
| NIP-70 protection marker | identity archival requests always emit `["-"]` as the first tag | `builders.rs:1748` | kinds 9035/9036 |
| `allow_self_tagging()` | required so nostr 0.44 does not strip the `["p", target]` tag when actor == target (self archive/unarchive path) | `builders.rs:1798-1801`, `1819-1822` | kinds 9035/9036 |
| Non-standard `h` on NIP-09 kind 5 | `build_delete_compat` adds `h` so channel-scoped subscriptions observe the delete (documented as intentionally non-standard) | `builders.rs:433-443` | compat deletes |
| `a`-tag coordinate format | repo coordinate rendered as `30617:<owner>:<id>` with owner and id revalidated at render time | `builders.rs:975-982` | all NIP-34 builders |
| Workflow deletion coordinate | `["a", "<KIND_WORKFLOW_DEF>:<pubkey>:<workflow_id>"]` (30620) | `builders.rs:1498-1508` | workflow delete |
| Multi-value tags | `clone`, `relays`, `applied-as-commits` are single tags with N values, not N tags | `builders.rs:925-940`, `1289-1297`, `1367-1369`, `1449-1451` | repo announcement, PR, PR update, merged status |
| Duplicated `r` tags for commits | `commit`, `merge-commit`, and each `applied-as-commit` also emit a bare `["r", <commit>]` | `builders.rs:1050-1052`, `1284-1287`, `1296-1298` | patch, merged status |
| NIP-22 uppercase root tags | PR update references its PR with `["E", pr_event]` + `["P", pr_author]` | `builders.rs:1443-1444` | kind 1619 |
| Emoji set d-tag pinned | `["d","buzz:custom-emoji"]` always first, so the set is parameterized-replaceable per member | `builders.rs:503`, `513` | kind 30030 |

---

### 3. Per-builder business rules

| Rule | Enforces | File:line | Trigger |
|---|---|---|---|
| Message content bound | ≤ 64 KiB | `builders.rs:227` | `build_message` |
| Diff content bound | ≤ 60 KiB (tighter than messages) | `builders.rs:314` | `build_diff_message` |
| Diff `repo_url` scheme | must start `http://` or `https://` | `builders.rs:317-321` | `build_diff_message` |
| Diff commit SHA | ≥7 hex chars (abbreviated refs allowed here, unlike NIP-34 builders) | `builders.rs:322` | `build_diff_message` |
| Diff parent commit | ≥7 hex chars when present | `builders.rs:323-325` | `build_diff_message` |
| Diff branch pair | both source and target must be non-empty | `builders.rs:326-332` | `build_diff_message` |
| Diff PR number | must be > 0 | `builders.rs:333-339` | `build_diff_message` |
| Observer frame direction | `frame` must equal `"telemetry"` or `"control"` | `builders.rs:251-255` | `build_agent_observer_frame` |
| Observer frame encryption | `content_looks_like_nip44` must hold (length in 132..=87 472, `crates/buzz-core/src/observer.rs:53-55`) — plaintext refused | `builders.rs:256-260` | `build_agent_observer_frame` |
| Reaction emoji length | > 64 **chars** ⇒ `EmojiTooLong` (char count, not bytes) | `builders.rs:467-469` | `build_reaction` |
| Emoji set uniqueness | duplicate normalized shortcode ⇒ `InvalidInput` (hard error, not silent dedup) | `builders.rs:517-521` | `build_custom_emoji_set` |
| Profile field omission | only `Some` fields land in the kind-0 JSON object | `builders.rs:542-561` | `build_profile` |
| Channel update non-empty | at least one of name/about/visibility/ttl required | `builders.rs:610-614` | `build_update_channel` |
| Channel visibility vocabulary | free-form `&str` restricted to `"open"`/`"private"` | `builders.rs:615-621` | `build_update_channel` |
| Channel name canonicalization | leading `#`/whitespace stripped via `canonical_channel_name`; result must not be blank | `builders.rs:622-627`, `634-639` (update); `675-679` (create) | create + update channel |
| TTL tri-state | `None` = unchanged; `Some(Some(secs))` = set; `Some(None)` = clear via `["ttl",""]` | `builders.rs:641-646` | `build_update_channel` |
| Contact list cap | > 10 000 contacts ⇒ `InvalidInput`; checked before dedup | `builders.rs:751`, `770-776` | `build_contact_list` |
| Contact pubkey format | exactly 64 hex chars, any case, normalized lowercase | `builders.rs:779-784` | `build_contact_list` |
| Contact relay URL / petname bounds | ≤2048 bytes / ≤256 bytes | `builders.rs:785-799` | `build_contact_list` |
| Contact dedup | duplicate pubkeys silently skipped, first occurrence kept | `builders.rs:800-804` | `build_contact_list` |
| Repo announcement bounds | `name` ≤128, `description` ≤1024, ≤5 `clone_urls` each non-empty and ≤512, `web_url` http(s) and ≤512, ≤10 `relays` each `ws://`/`wss://` and ≤256 | `builders.rs:840-919` | `build_repo_announcement` |
| Read-modify-write d-tag canonicalization | all caller `d` tags removed, exactly one validated `d` inserted at index 0; every other tag preserved | `builders.rs:958-963` | `build_repo_announcement_with_tags` |
| Patch content must be appliable | blank/whitespace-only content rejected ("refusing to publish an unappliable patch") | `builders.rs:1018-1022` | `build_git_patch` |
| Patch size bound | ≤60 KiB per NIP-34's patch-vs-PR guidance; never silently truncated | `builders.rs:1007-1012`, `1023` | `build_git_patch` |
| Patch root exclusivity | `root` and `root_revision` are mutually exclusive | `builders.rs:1038-1042` | `build_git_patch` |
| Issue subject | non-empty, ≤256 chars | `builders.rs:1086-1094` | `build_git_issue` |
| Merge-only status fields | `applied_patches`/`merge_commit`/`applied_as_commits` allowed only on `AppliedOrResolved` (1631) | `builders.rs:1250-1259` | `build_git_status` |
| `q`-tag hint ordering | pubkey hint without a relay hint ⇒ `InvalidInput` (NIP-34 positional shape) | `builders.rs:1260-1275` | `build_git_status` |
| PR subject | non-empty, ≤256 chars | `builders.rs:1335-1343` | `build_git_pull_request` |
| PR reachability inputs | full-length `commit` plus ≥1 `clone_urls` entry required | `builders.rs:1344-1350` | `build_git_pull_request` |
| PR channel id | `channel_id` must parse as a UUID; re-rendered canonically (lowercased, untrimmed input rejected) | `builders.rs:1370-1374`; tests `builders.rs:3491-3505` and `3506-3517` | `build_git_pull_request` |
| PR update required refs | `pr_event` 64-hex, `pr_author` 64-hex pubkey, full `commit`, ≥1 clone URL | `builders.rs:1421-1431` | `build_git_pr_update` |
| Workflow YAML bound | ≤64 KiB | `builders.rs:1468`, `1486` | workflow def/update |
| Approval token hash | exactly 64 hex chars (SHA-256 digest) | `builders.rs:1528-1532` | `build_workflow_approval` |
| Approval kind selection | `approved == true` ⇒ 46030, else 46031 | `builders.rs:1533-1537` | `build_workflow_approval` |
| DM participant count | 1–8 pubkeys | `builders.rs:1545-1549` | `build_dm_open` |
| Presence vocabulary | `"online"`/`"away"`/`"offline"` only; value duplicated into content and `status` tag | `builders.rs:1571-1584` | `build_presence_update` |
| Ban expiry semantics | `None` ⇒ permanent (no `expiration` tag); `Some(unix)` ⇒ temporary | `builders.rs:1593-1606` | `build_moderation_ban` |
| Timeout expiry required | `expires_at: u64` is a non-optional parameter | `builders.rs:1623-1633` | `build_moderation_timeout` |
| Report resolution vocabulary | `status` ∈ {resolved, dismissed}; `action` ∈ {delete, kick, ban, timeout, dismiss, escalate}; the status/action **pairing** is left to the relay | `builders.rs:1660-1680` | `build_moderation_resolve_report` |
| Rotation pointer distinctness | `replaced_by` must differ from `target_pubkey` | `builders.rs:1765-1774` | `build_archive_identity_request` |
| Unarchive has no rotation pointer | `replaced_by` is not a parameter at all; `identity_archive_tags` called with `None` | `builders.rs:1810-1823` | kind 9036 |

---

### 4. Mention-resolution rules (`mentions.rs`)

| Rule | Enforces | File:line | Trigger |
|---|---|---|---|
| `@` boundary rule | `@` must be at index 0 or preceded by ASCII whitespace — excludes `user@host` email forms | `mentions.rs:78-80`, `128-131` | both extractors |
| Name charset | `[A-Za-z0-9._-]` terminates the single-word token | `mentions.rs:85-90`, `144-147` | both extractors |
| Case folding + dedup | all names lowercased, first-seen order preserved | `mentions.rs:93-97`, `149-152` | both extractors |
| Longest-known-name-first | known names sorted by descending length; a match also requires a trailing word boundary | `mentions.rs:117-121`, `134-141`, `154-159` | `extract_at_mentions_with_known` |
| UTF-8 boundary safety | `rest.get(..k.len())` returns `None` on non-char-boundary rather than panicking | `mentions.rs:135-138`; test `mentions.rs:520-531` | multi-byte content |
| Profile name precedence | `display_name` first; `name` used **only when `display_name` is absent** (an empty `display_name` blocks the fallback) | `mentions.rs:186-193`; test `mentions.rs:560-569` | `match_names_to_profiles` |
| Silent skip on bad profiles | unparseable JSON, missing or non-string name ⇒ profile skipped | `mentions.rs:183-195` | `match_names_to_profiles` |
| Ambiguity allowed | duplicate display names yield multiple pubkeys by design | `mentions.rs:174-177`, `199-203` | `match_names_to_profiles` |
| Output order = profile order | matched pubkeys follow the profile slice, not text position | `mentions.rs:165-170`, `181-205`; test `mentions.rs:571-583` | `match_names_to_profiles` |
| Explicit-mentions priority | merge appends auto-resolved entries only within `cap - explicit.len()` budget | `mentions.rs:208-220` | `merge_mentions` |
| No self-mention | sender pubkey removed case-insensitively | `mentions.rs:228-241` | `normalize_mention_pubkeys` |
| Code regions excluded from scanning | fenced blocks (``` at line start) and same-line inline spans replaced by a single space; original content stored verbatim | `mentions.rs:244-341` | pre-pass to `extract_nostr_uris` |
| NIP-27 URI shape | requires literal `nostr:npub1` + exactly 58 bech32 chars; window must land on a char boundary; uppercase normalized before decode; invalid bech32 silently skipped | `mentions.rs:353-387` | `extract_nostr_uris` |

---

### 5. NIP-OA rules (`nip_oa.rs`)

| Rule | Enforces | File:line | Trigger |
|---|---|---|---|
| Signing preimage | `"nostr:agent-auth:" ‖ agent_pubkey_hex ‖ ":" ‖ conditions`, hashed with SHA-256, signed BIP-340 Schnorr | `nip_oa.rs:109-116`, `146-176` | compute + verify |
| Self-attestation rejected | `owner_pubkey == agent_pubkey` fails at **both** compute and verify | `nip_oa.rs:154-159`, `212-217` | both paths |
| Conditions grammar | empty allowed; otherwise `&`-joined clauses, each `kind=<0..65535>`, `created_at<<0..4294967295>`, or `created_at><0..4294967295>` | `nip_oa.rs:36-73` | compute, verify, parse |
| No whitespace in conditions | any ASCII whitespace anywhere ⇒ error | `nip_oa.rs:42-47` | all three |
| No empty clauses | leading/trailing/double `&` ⇒ error | `nip_oa.rs:50-57` | all three |
| Canonical decimals | no leading zeros (except `"0"`), digits only, range-checked | `nip_oa.rs:75-107` | all three |
| Case-sensitive clause labels | `Kind=`/`CREATED_AT<` rejected; `created_at=` (wrong operator) rejected | `nip_oa.rs:61-73`; tests `nip_oa.rs:541-546` | all three |
| Tag arity | JSON must be an array of exactly 4 elements, all strings, element 0 == `"auth"` | `nip_oa.rs:124-133`, `181-205`, `254-272` | verify + parse |
| Lowercase-hex requirement (parse path) | owner pubkey 64 lowercase hex, signature 128 lowercase hex — uppercase rejected | `nip_oa.rs:120-122`, `274-296`; test `nip_oa.rs:458-486` | `parse_auth_tag` |
| Crypto-free fast path | `parse_auth_tag` performs no signature verification (documented as the MCP-startup path) | `nip_oa.rs:239-251` | agent startup |

---

### 6. Defaulting and timestamp handling

- **No timestamp is ever set.** No builder calls `custom_created_at`; `created_at`
  is whatever `nostr::EventBuilder` assigns at signing time. The only
  timestamp-shaped values the SDK writes are caller-supplied: `["expiration",
  <unix>]` for ban/timeout (`builders.rs:1602`, `1632`), `["ttl", <secs>]` for
  channels (`builders.rs:642`, `697`), and the committer tuple's `ts`/`tz`
  strings (`builders.rs:1063-1065`).
- Defaulting is limited to: `DeleteMessageOptions::default()` for the
  no-metadata delete path (`builders.rs:403-409`), `#[derive(Default)]` on the
  five Git meta structs, and `unwrap_or("")` for NIP-02 relay/petname slots
  (`builders.rs:806-811`).
- Normalization defaults: pubkeys and event ids are lowercased on the way out
  (`builders.rs:69-89`), channel UUIDs are re-rendered from `Uuid`
  (canonical lowercase hyphenated form) rather than echoed from input strings.
