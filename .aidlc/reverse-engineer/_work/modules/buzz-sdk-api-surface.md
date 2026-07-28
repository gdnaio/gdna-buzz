## Module: buzz-sdk (`crates/buzz-sdk`)

### Aspect: API Surface

Crate root declares three public modules and glob-re-exports the builders
(`crates/buzz-sdk/src/lib.rs:15-19`):

```rust
pub mod builders;   // 51 public builder/helper fns
pub mod mentions;   // 7 public fns + 1 struct + 1 const
pub mod nip_oa;     // 3 public fns
pub use builders::*;
pub use buzz_core::kind;               // lib.rs:22
```

Crate-level lints: `#![deny(unsafe_code)]`, `#![warn(missing_docs)]`
(`crates/buzz-sdk/src/lib.rs:1-2`).

**No REST/WS helpers exist.** The crate has no HTTP or WebSocket client, no
async, and no network dependency; the module doc states "No keys are held here.
No network calls are made." (`crates/buzz-sdk/src/lib.rs:13`). Dependency list
confirms it (`crates/buzz-sdk/Cargo.toml:10-16`).

---

### 1. Builder → kind → tags → content

Signature abbreviations: all builders return `Result<EventBuilder, SdkError>`.
`R` = required, `O` = optional.

| # | Builder (signature) | Kind | Required tags | Optional tags | Content |
|---|---|---|---|---|---|
| 1 | `build_message(channel_id: Uuid, content: &str, thread_ref: Option<&ThreadRef>, mentions: &[&str], broadcast: bool, media_tags: &[Vec<String>])` `builders.rs:219` | 9 | `h` | `e`(NIP-10), `p`*, `broadcast`, imeta | text ≤64 KiB |
| 2 | `build_agent_observer_frame(recipient_pubkey: &str, agent_pubkey: &str, frame: &str, encrypted_content: &str)` `builders.rs:245` | 24200 | `p`, `agent`, `frame` | — | NIP-44 v2 ciphertext |
| 3 | `build_forum_post(channel_id, content, mentions, media_tags)` `builders.rs:278` | 45001 | `h` | `p`*, imeta | text ≤64 KiB |
| 4 | `build_forum_comment(channel_id, content, thread_ref: &ThreadRef, mentions, media_tags)` `builders.rs:292` | 45003 | `h`, `e` | `p`*, imeta | text ≤64 KiB |
| 5 | `build_diff_message(channel_id, content, diff_meta: &DiffMeta, thread_ref: Option<&ThreadRef>)` `builders.rs:308` | 40008 | `h`, `repo`, `commit` | `file`, `parent-commit`, `branch`, `pr`, `l`, `description`, `truncated`, `alt`, `e` | diff ≤60 KiB |
| 6 | `build_edit(channel_id, target_event_id: EventId, new_content)` `builders.rs:378` | 40003 | `h`, `e` | — | text ≤64 KiB |
| 7 | `build_delete_message(channel_id, target_event_id)` `builders.rs:403` | 9005 | `h`, `e` | — | `""` |
| 8 | `build_delete_message_with_options(channel_id, target_event_id, options: DeleteMessageOptions)` `builders.rs:411` | 9005 | `h`, `e` | `action_id`, `reason_code`, `public_reason` | `""` |
| 9 | `build_delete_compat(channel_id, target_event_id)` `builders.rs:434` | 5 | `h`, `e` | — | `""` |
| 10 | `build_vote(channel_id, target_event_id, direction: VoteDirection)` `builders.rs:446` | 45002 | `h`, `e` | — | `"+"`/`"-"` |
| 11 | `build_reaction(target_event_id, emoji: &str)` `builders.rs:463` | 7 | `e` | — | emoji ≤64 chars |
| 12 | `build_custom_emoji_reaction(target_event_id, shortcode, url)` `builders.rs:479` | 7 | `e`, `emoji` | — | `":shortcode:"` |
| 13 | `build_remove_reaction(reaction_event_id)` `builders.rs:495` | 5 | `e` | — | `""` |
| 14 | `build_custom_emoji_set(emojis: &[CustomEmoji])` `builders.rs:511` | 30030 | `d`=`buzz:custom-emoji` | `emoji`* | `""` |
| 15 | `build_set_canvas(channel_id, content)` `builders.rs:529` | 40100 | `h` | — | canvas text (unbounded) |
| 16 | `build_profile(display_name, name, picture, about, nip05: Option<&str>)` `builders.rs:537` | 0 | — | — | JSON object of present fields |
| 17 | `build_add_member(channel_id, target_pubkey, role: Option<MemberRole>)` `builders.rs:565` | 9000 | `h`, `p` | `role` | `""` |
| 18 | `build_remove_member(channel_id, target_pubkey)` `builders.rs:582` | 9001 | `h`, `p` | — | `""` |
| 19 | `build_leave(channel_id)` `builders.rs:595` | 9022 | `h` | — | `""` |
| 20 | `build_update_channel(channel_id, name, about, visibility: Option<&str>, ttl: Option<Option<i32>>)` `builders.rs:604` | 9002 | `h` + ≥1 of the four | `name`, `about`, `visibility`, `ttl` | `""` |
| 21 | `build_set_topic(channel_id, topic)` `builders.rs:652` | 9002 | `h`, `topic` | — | `""` |
| 22 | `build_set_purpose(channel_id, purpose)` `builders.rs:661` | 9002 | `h`, `purpose` | — | `""` |
| 23 | `build_create_channel(channel_id, name, visibility: Option<Visibility>, channel_type: Option<ChannelKind>, about, ttl: Option<i32>)` `builders.rs:674` | 9007 | `h`, `name` | `visibility`, `channel_type`, `about`, `ttl` | `""` |
| 24 | `build_join(channel_id)` `builders.rs:703` | 9021 | `h` | — | `""` |
| 25 | `build_archive(channel_id)` `builders.rs:709` | 9002 | `h`, `archived=true` | — | `""` |
| 26 | `build_unarchive(channel_id)` `builders.rs:718` | 9002 | `h`, `archived=false` | — | `""` |
| 27 | `build_delete_channel(channel_id)` `builders.rs:727` | 9008 | `h` | — | `""` |
| 28 | `build_note(content, reply_to_event_id: Option<EventId>)` `builders.rs:738` | 1 | — | `e` (`reply` marker) | text ≤64 KiB |
| 29 | `build_contact_list(contacts: &[(&str, Option<&str>, Option<&str>)])` `builders.rs:764` | 3 | — | `p`* (≤10 000) | `""` |
| 30 | `build_repo_announcement(repo_id, name, description, clone_urls: &[&str], web_url, relays: &[&str])` `builders.rs:834` | 30617 | `d` | `name`, `description`, `clone`, `web`, `relays` | `""` |
| 31 | `build_repo_announcement_with_tags(repo_id, content, tags: Vec<Tag>)` `builders.rs:952` | 30617 | `d` | all caller tags preserved except `d` | caller content |
| 32 | `build_git_patch(repo: &GitRepoCoord, content, meta: &GitPatchMeta)` `builders.rs:1013` | 1617 | `a`, `p`(owner) | `r`(euc), `p`*, `e`, `t`, `commit`, `r`, `parent-commit`, `commit-pgp-sig`, `committer` | patch ≤60 KiB, non-blank |
| 33 | `build_git_issue(repo, subject, content, meta: &GitIssueMeta)` `builders.rs:1081` | 1621 | `a`, `p`(owner), `subject` | `p`*, `t`* | markdown ≤64 KiB |
| 34 | `build_git_status(status: GitStatus, content, meta: &GitStatusMeta)` `builders.rs:1222` | 1630/1631/1632/1633 | `e`(root) | `e`(reply), `p`*, `a`, `r`, `q`*, `merge-commit`, `applied-as-commits` | markdown ≤64 KiB |
| 35 | `build_git_pull_request(repo, content, meta: &GitPullRequestMeta)` `builders.rs:1330` | 1618 | `a`, `p`(owner), `subject`, `c`, `clone` | `r`, `p`*, `t`*, `h`, `branch-name`, `merge-base`, `e` | markdown ≤64 KiB |
| 36 | `build_git_pr_update(repo, content, meta: &GitPrUpdateMeta)` `builders.rs:1416` | 1619 | `a`, `p`(owner), `E`, `P`, `c`, `clone` | `r`, `p`*, `merge-base` | markdown ≤64 KiB |
| 37 | `build_workflow_def(channel_id, workflow_id, yaml)` `builders.rs:1463` | 30620 | `d`, `h` | — | YAML ≤64 KiB |
| 38 | `build_workflow_update(channel_id, workflow_id, yaml)` `builders.rs:1481` | 30620 | `d`, `h` | — | YAML ≤64 KiB |
| 39 | `build_workflow_delete(author_pubkey, workflow_id)` `builders.rs:1498` | 5 | `a` (`30620:pk:uuid`) | — | `""` |
| 40 | `build_workflow_trigger(workflow_id)` `builders.rs:1511` | 46020 | `d` | — | `""` |
| 41 | `build_workflow_approval(token_hash, approved: bool, note)` `builders.rs:1522` | 46030 / 46031 | `d` | — | note (unbounded) |
| 42 | `build_dm_open(pubkeys: &[&str])` `builders.rs:1544` | 41010 | `p` ×1–8 | — | `""` |
| 43 | `build_dm_add_member(channel_id, pubkey)` `builders.rs:1559` | 41011 | `h`, `p` | — | `""` |
| 44 | `build_presence_update(status: &str)` `builders.rs:1570` | 20001 | `status` | — | status string |
| 45 | `build_moderation_ban(target_pubkey, expires_at: Option<u64>, reason: Option<&str>)` `builders.rs:1597` | 9040 | `p` | `expiration`, `reason` | `""` |
| 46 | `build_moderation_unban(target_pubkey)` `builders.rs:1614` | 9041 | `p` | — | `""` |
| 47 | `build_moderation_timeout(target_pubkey, expires_at: u64, reason)` `builders.rs:1623` | 9042 | `p`, `expiration` | `reason` | `""` |
| 48 | `build_moderation_untimeout(target_pubkey)` `builders.rs:1640` | 9043 | `p` | — | `""` |
| 49 | `build_moderation_resolve_report(report_event_id, status, action, reason)` `builders.rs:1654` | 9044 | `report`, `status`, `action` | `reason` | `""` |
| 50 | `build_archive_identity_request(target_pubkey, content, reason, replaced_by, auth: Option<&[String;4]>)` `builders.rs:1788` | 9035 | `-`, `p` | `reason`, `replaced-by`, `auth` | text ≤64 KiB |
| 51 | `build_unarchive_identity_request(target_pubkey, content, reason, auth)` `builders.rs:1810` | 9036 | `-`, `p` | `reason`, `auth` | text ≤64 KiB |

Kind integers are sourced from `buzz_core::kind` for 26 of the builders
(`crates/buzz-sdk/src/builders.rs:6-19`); the remainder pass literals directly
to `Kind::Custom` (9, 45001, 45003, 40008, 40003, 9005, 5, 45002, 7, 40100, 0,
9000, 9001, 9022, 9002, 9007, 9021, 9008, 1, 3).

---

### 2. Non-builder public API in `builders.rs`

| Item | Signature | Purpose | File:line |
|---|---|---|---|
| `normalize_custom_emoji_shortcode` | `fn(&str) -> Result<String, SdkError>` | trims `:`/whitespace, validates charset+length, lowercases | `builders.rs:127-150` |
| `extract_channel_id` | `fn(&nostr::Event) -> Option<Uuid>` | reads first `h` tag as UUID | `builders.rs:816-826` |
| `GitAppliedPatchRef::parse` | `fn(&str) -> Result<Self, SdkError>` | parses `<id>[:<relay>[:<pubkey>]]` | `builders.rs:1143-1185` |
| `CUSTOM_EMOJI_SET_D_TAG` | `pub const &str` | `"buzz:custom-emoji"` | `builders.rs:503` |
| `DeleteMessageOptions`, `GitRepoCoord`, `GitPatchMeta`, `GitIssueMeta`, `GitStatus`, `GitAppliedPatchRef`, `GitStatusMeta`, `GitPullRequestMeta`, `GitPrUpdateMeta` | public structs/enums | builder inputs | see data-model doc |

`GitRepoCoord::to_a_tag_value` and `GitStatus::kind` are **private** inherent
methods (`builders.rs:976`, `builders.rs:1188`).

---

### 3. `mentions` module public API

| Item | Signature | Behavior | File:line |
|---|---|---|---|
| `MENTION_CAP` | `pub const usize = 50` | hard cap on mention p-tags | `mentions.rs:38` |
| `MentionProfile<'a>` | struct | `{ pubkey, content_json }` | `mentions.rs:46-51` |
| `extract_at_names` | `fn(&str) -> Vec<String>` | single-word `@name` scan; `@` must be at start or after ASCII whitespace; charset `[A-Za-z0-9._-]`; lowercased, deduped, first-seen order | `mentions.rs:64-104` |
| `extract_at_mentions_with_known` | `fn(&str, known_names: &[&str]) -> Vec<String>` | longest-known-name-first matching with word-boundary check, falls back to single-word tokenizer | `mentions.rs:107-152` |
| `match_names_to_profiles` | `fn(&[String], &[MentionProfile]) -> Vec<String>` | matches `display_name` (fallback `name`) case-insensitively; returns pubkeys in **profile order** | `mentions.rs:179-206` |
| `merge_mentions` | `fn(&mut Vec<String>, &[String], cap: usize)` | appends non-duplicate auto-resolved pubkeys up to `cap` | `mentions.rs:208-220` |
| `normalize_mention_pubkeys` | `fn(&[String], sender_pubkey: Option<&str>) -> Vec<String>` | lowercases, dedupes, drops self-mention | `mentions.rs:228-241` |
| `strip_code_regions` | `fn(&str) -> String` | replaces fenced blocks and inline spans with a single space | `mentions.rs:244-341` |
| `extract_nostr_uris` | `fn(&str) -> Vec<String>` | finds `nostr:npub1` + 58 bech32 chars, decodes to hex, dedupes | `mentions.rs:353-387` |

---

### 4. `nip_oa` module public API

| Item | Signature | Behavior | File:line |
|---|---|---|---|
| `compute_auth_tag` | `fn(&Keys, &PublicKey, conditions: &str) -> Result<String, SdkError>` | rejects self-attestation, validates conditions, BIP-340 Schnorr-signs `SHA256("nostr:agent-auth:<agent_hex>:<conditions>")`, returns JSON array string | `nip_oa.rs:146-176` |
| `verify_auth_tag` | `fn(auth_tag_json: &str, &PublicKey) -> Result<PublicKey, SdkError>` | full structural + cryptographic verification; returns owner pubkey | `nip_oa.rs:179-249` |
| `parse_auth_tag` | `fn(json_str: &str) -> Result<Tag, SdkError>` | structure-only fast path (no crypto); requires 64-hex lowercase owner, 128-hex lowercase sig | `nip_oa.rs:252-299` |

Private helpers: `validate_conditions` (`nip_oa.rs:36-59`), `validate_clause`
(`61-73`), `validate_canonical_decimal` (`75-107`), `build_preimage` (`109-111`),
`hash_preimage` (`113-116`), `is_lowercase_hex` (`120-122`), `parse_json_array`
(`124-133`).

---

### 5. Example binary

`examples/compute_auth_tag.rs` — CLI wrapper: takes `<owner_secret_hex>
<agent_pubkey_hex> [conditions]` from `std::env::args`, prints the auth-tag JSON
to stdout (`crates/buzz-sdk/examples/compute_auth_tag.rs:11-28`). There are no
`src/bin/*.rs` entry points.
