## Module: buzz-sdk (`crates/buzz-sdk`)

### Aspect: Data Model

The crate defines **no persistence model**. Its "data model" is (a) the input
parameter structs consumed by builder functions and (b) the Nostr event wire
shapes those builders emit. Every builder returns `nostr::EventBuilder`
(unsigned); the caller signs (`crates/buzz-sdk/src/lib.rs:11-13`).

---

### 1. Input types declared in `lib.rs`

| Type | Kind | Fields (type) | File:line |
|---|---|---|---|
| `ThreadRef` | struct | `root_event_id: nostr::EventId`, `parent_event_id: nostr::EventId` | `crates/buzz-sdk/src/lib.rs:28-33` |
| `DiffMeta` | struct | `repo_url: String`, `commit_sha: String`, `file_path: Option<String>`, `parent_commit: Option<String>`, `branch: Option<(String, String)>`, `pr_number: Option<u32>`, `language: Option<String>`, `description: Option<String>`, `truncated: bool`, `alt_text: Option<String>` | `crates/buzz-sdk/src/lib.rs:36-57` |
| `VoteDirection` | enum (`Debug, Clone, Copy, PartialEq, Eq`) | `Up`, `Down` | `crates/buzz-sdk/src/lib.rs:60-66` |
| `CustomEmoji` | struct (`Debug, Clone, PartialEq, Eq`) | `shortcode: String`, `url: String` | `crates/buzz-sdk/src/lib.rs:69-75` |
| `SdkError` | enum (`thiserror::Error`) | see error table below | `crates/buzz-sdk/src/lib.rs:87-113` |

Re-exported from `buzz-core` (no local definition):

| Local alias | Source | Variants → tag string | File:line |
|---|---|---|---|
| `ChannelKind` | `buzz_core::channel::ChannelType` | `Stream`→`stream`, `Forum`→`forum`, `Dm`→`dm`, `Workflow`→`workflow` | `crates/buzz-sdk/src/lib.rs:80`; values `crates/buzz-core/src/channel.rs:59-79` |
| `Visibility` | `buzz_core::channel::ChannelVisibility` | `Open`→`open`, `Private`→`private` | `crates/buzz-sdk/src/lib.rs:82`; values `crates/buzz-core/src/channel.rs:22-36` |
| `MemberRole` | `buzz_core::channel::MemberRole` | `Owner`/`Admin`/`Member`/`Guest`/`Bot` → `owner`/`admin`/`member`/`guest`/`bot` | `crates/buzz-sdk/src/lib.rs:84`; values `crates/buzz-core/src/channel.rs:108-131` |
| `canonical_channel_name` | `buzz_core::channel::canonical_channel_name` | strips leading `#`/whitespace, trims trailing ws | `crates/buzz-sdk/src/lib.rs:78`; impl `crates/buzz-core/src/channel.rs:15-18` |
| `kind` module | `buzz_core::kind` | all kind integers | `crates/buzz-sdk/src/lib.rs:22` |

`SdkError` variants:

| Variant | Payload | Display text | File:line |
|---|---|---|---|
| `ContentTooLarge` | `{ max: usize, got: usize }` | `content exceeds maximum size of {max} bytes (got {got})` | `crates/buzz-sdk/src/lib.rs:89-96` |
| `InvalidTag` | `String` | `invalid tag: {0}` | `crates/buzz-sdk/src/lib.rs:97-99` |
| `EmojiTooLong` | — | `emoji exceeds maximum length of 64 characters` | `crates/buzz-sdk/src/lib.rs:100-102` |
| `TooManyMentions` | — | `too many mentions (max 50)` | `crates/buzz-sdk/src/lib.rs:103-105` |
| `InvalidDiffMeta` | `String` | `invalid diff metadata: {0}` | `crates/buzz-sdk/src/lib.rs:106-108` |
| `InvalidInput` | `String` | `invalid input: {0}` | `crates/buzz-sdk/src/lib.rs:109-112` |

---

### 2. Input types declared in `builders.rs`

| Type | Kind | Fields (type) | File:line |
|---|---|---|---|
| `DeleteMessageOptions<'a>` | struct (`Debug, Clone, Default`) | `action_id: Option<Uuid>`, `reason_code: Option<&'a str>`, `public_reason: Option<&'a str>` | `crates/buzz-sdk/src/builders.rs:392-400` |
| `GitRepoCoord` | struct | `owner: String` (64-hex pubkey), `id: String` (`d`-tag) | `crates/buzz-sdk/src/builders.rs:964-973` |
| `GitPatchMeta` | struct (`Default`) | `euc: Option<String>`, `recipients: Vec<String>`, `reply_to: Option<String>`, `root: bool`, `root_revision: bool`, `commit: Option<String>`, `parent_commit: Option<String>`, `commit_pgp_sig: Option<String>`, `committer: Option<(String, String, String, String)>` | `crates/buzz-sdk/src/builders.rs:984-1005` |
| `GitIssueMeta` | struct (`Default`) | `labels: Vec<String>`, `recipients: Vec<String>` | `crates/buzz-sdk/src/builders.rs:1072-1078` |
| `GitStatus` | enum (`Debug, Clone, Copy, PartialEq, Eq`) | `Open`, `AppliedOrResolved`, `Closed`, `Draft` | `crates/buzz-sdk/src/builders.rs:1114-1124` |
| `GitAppliedPatchRef` | struct (`Debug, Clone, PartialEq, Eq`) | `id: String`, `relay: Option<String>`, `pubkey: Option<String>` | `crates/buzz-sdk/src/builders.rs:1132-1141` |
| `GitStatusMeta` | struct (`Default`) | `root_event: String`, `accepted_revision_root: Option<String>`, `repo: Option<GitRepoCoord>`, `euc: Option<String>`, `recipients: Vec<String>`, `applied_patches: Vec<GitAppliedPatchRef>`, `merge_commit: Option<String>`, `applied_as_commits: Vec<String>` | `crates/buzz-sdk/src/builders.rs:1200-1219` |
| `GitPullRequestMeta` | struct (`Default`) | `euc: Option<String>`, `recipients: Vec<String>`, `channel_id: Option<String>`, `subject: String`, `labels: Vec<String>`, `commit: String`, `clone_urls: Vec<String>`, `branch_name: Option<String>`, `merge_base: Option<String>`, `revision_of: Option<String>` | `crates/buzz-sdk/src/builders.rs:1302-1327` |
| `GitPrUpdateMeta` | struct (`Default`) | `euc: Option<String>`, `recipients: Vec<String>`, `pr_event: String`, `pr_author: String`, `commit: String`, `clone_urls: Vec<String>`, `merge_base: Option<String>` | `crates/buzz-sdk/src/builders.rs:1396-1411` |
| `CUSTOM_EMOJI_SET_D_TAG` | `pub const &str` = `"buzz:custom-emoji"` | — | `crates/buzz-sdk/src/builders.rs:503` |
| `MAX_CONTACTS` | private `const usize` = `10_000` | — | `crates/buzz-sdk/src/builders.rs:751` |
| `MAX_REASON_BYTES` | private `const usize` = `64` | — | `crates/buzz-sdk/src/builders.rs:1704` |

`GitStatus` → kind mapping (`crates/buzz-sdk/src/builders.rs:1187-1196`):
`Open`→1630, `AppliedOrResolved`→1631, `Closed`→1632, `Draft`→1633.

`GitRepoCoord::to_a_tag_value` renders `30617:<owner>:<id>`
(`crates/buzz-sdk/src/builders.rs:975-982`).

---

### 3. Input types declared in `mentions.rs`

| Type | Kind | Fields | File:line |
|---|---|---|---|
| `MentionProfile<'a>` | struct (`Debug, Clone, Copy`) | `pubkey: &'a str` (lowercase hex), `content_json: &'a str` (raw kind-0 `content`) | `crates/buzz-sdk/src/mentions.rs:45-51` |
| `MENTION_CAP` | `pub const usize` = `50` | — | `crates/buzz-sdk/src/mentions.rs:38` |

`match_names_to_profiles` reads only `display_name`, falling back to `name`
when `display_name` is absent (`crates/buzz-sdk/src/mentions.rs:186-190`).

---

### 4. Event wire shapes produced (tags + content)

Content column is the exact `event.content` payload the builder sets.

| Kind | Builder | Tags emitted (in order) | Content shape |
|---|---|---|---|
| 9 | `build_message` | `["h",uuid]`, NIP-10 e-tags, `["p",hex]`*, `["broadcast","1"]`?, raw imeta tags | plain text, ≤64 KiB (`builders.rs:219-241`) |
| 24200 | `build_agent_observer_frame` | `["p",recipient]`, `["agent",agent_pk]`, `["frame","telemetry"\|"control"]` | NIP-44 v2 ciphertext (`builders.rs:245-274`) |
| 45001 | `build_forum_post` | `["h",uuid]`, `["p",hex]`*, imeta | text ≤64 KiB (`builders.rs:278-289`) |
| 45003 | `build_forum_comment` | `["h",uuid]`, NIP-10 e-tags, `["p",hex]`*, imeta | text ≤64 KiB (`builders.rs:292-305`) |
| 40008 | `build_diff_message` | `["h",uuid]`, `["repo",url]`, `["commit",sha]`, `["file",p]`?, `["parent-commit",sha]`?, `["branch",src,tgt]`?, `["pr",n]`?, `["l",lang]`?, `["description",d]`?, `["truncated","true"]`?, `["alt",t]`?, e-tags? | diff text ≤60 KiB (`builders.rs:308-375`) |
| 40003 | `build_edit` | `["h",uuid]`, `["e",target]` | replacement text ≤64 KiB (`builders.rs:378-389`) |
| 9005 | `build_delete_message` / `_with_options` | `["h",uuid]`, `["e",target]`, `["action_id",uuid]`?, `["reason_code",s]`?, `["public_reason",s]`? | empty string (`builders.rs:403-431`) |
| 5 | `build_delete_compat` | `["h",uuid]`, `["e",target]` | empty (`builders.rs:434-443`) |
| 5 | `build_remove_reaction` | `["e",reaction_id]` | empty (`builders.rs:495-498`) |
| 5 | `build_workflow_delete` | `["a","30620:<pubkey>:<workflow_uuid>"]` | empty (`builders.rs:1498-1508`) |
| 45002 | `build_vote` | `["h",uuid]`, `["e",target]` | `"+"` or `"-"` (`builders.rs:446-460`) |
| 7 | `build_reaction` | `["e",target]` | emoji string, ≤64 chars (`builders.rs:463-471`) |
| 7 | `build_custom_emoji_reaction` | `["e",target]`, `["emoji",shortcode,url]` | `":shortcode:"` (`builders.rs:479-492`) |
| 30030 | `build_custom_emoji_set` | `["d","buzz:custom-emoji"]`, then `["emoji",shortcode,url]`* | empty (`builders.rs:511-527`) |
| 40100 | `build_set_canvas` | `["h",uuid]` | canvas body, **no length check** (`builders.rs:529-532`) |
| 0 | `build_profile` | none | JSON object with only the `Some` keys among `display_name`, `name`, `picture`, `about`, `nip05` (`builders.rs:537-562`) |
| 9000 | `build_add_member` | `["h",uuid]`, `["p",hex]`, `["role",role]`? | empty (`builders.rs:565-579`) |
| 9001 | `build_remove_member` | `["h",uuid]`, `["p",hex]` | empty (`builders.rs:582-592`) |
| 9022 | `build_leave` | `["h",uuid]` | empty (`builders.rs:595-598`) |
| 9002 | `build_update_channel` | `["h",uuid]`, `["name",n]`?, `["about",a]`?, `["visibility",v]`?, `["ttl",secs\|""]`? | empty (`builders.rs:604-649`) |
| 9002 | `build_set_topic` | `["h",uuid]`, `["topic",t]` | empty (`builders.rs:652-658`) |
| 9002 | `build_set_purpose` | `["h",uuid]`, `["purpose",p]` | empty (`builders.rs:661-667`) |
| 9002 | `build_archive` / `build_unarchive` | `["h",uuid]`, `["archived","true"\|"false"]` | empty (`builders.rs:709-724`) |
| 9007 | `build_create_channel` | `["h",uuid]`, `["name",n]`, `["visibility",v]`?, `["channel_type",t]`?, `["about",a]`?, `["ttl",secs]`? | empty (`builders.rs:674-700`) |
| 9021 | `build_join` | `["h",uuid]` | empty (`builders.rs:703-706`) |
| 9008 | `build_delete_channel` | `["h",uuid]` | empty (`builders.rs:727-730`) |
| 1 | `build_note` | `["e",id,"","reply"]`? | text ≤64 KiB (`builders.rs:738-748`) |
| 3 | `build_contact_list` | `["p",hex,relay_or_"",petname_or_""]`* | empty (`builders.rs:764-813`) |
| 30617 | `build_repo_announcement` | `["d",repo_id]`, `["name",n]`?, `["description",d]`?, `["clone",url…]`?, `["web",url]`?, `["relays",url…]`? | empty (`builders.rs:834-949`) |
| 30617 | `build_repo_announcement_with_tags` | caller tags with all `d` tags removed, then `["d",repo_id]` inserted at index 0 | caller-supplied `content` (`builders.rs:952-965`) |
| 1617 | `build_git_patch` | `["a",coord]`, `["r",euc,"euc"]`?, `["p",owner]`, `["p",recipient]`*, `["e",prev,"","reply"]`?, `["t","root"]`?, `["t","root-revision"]`?, `["commit",c]`+`["r",c]`?, `["parent-commit",p]`?, `["commit-pgp-sig",s]`?, `["committer",name,email,ts,tz]`? | verbatim `git format-patch` output, ≤60 KiB, non-blank (`builders.rs:1013-1069`) |
| 1621 | `build_git_issue` | `["a",coord]`, `["p",owner]`, `["p",recipient]`*, `["subject",s]`, `["t",label]`* | markdown ≤64 KiB (`builders.rs:1081-1111`) |
| 1630/1631/1632/1633 | `build_git_status` | `["e",root,"","root"]`, `["e",rev,"","reply"]`?, `["p",recipient]`*, `["a",coord]`?, `["r",euc]`?, `["q",id[,relay[,pubkey]]]`*, `["merge-commit",c]`+`["r",c]`?, `["applied-as-commits",c…]`+`["r",c]`* | markdown ≤64 KiB (may be empty) (`builders.rs:1222-1299`) |
| 1618 | `build_git_pull_request` | `["a",coord]`, `["r",euc]`?, `["p",owner]`, `["p",recipient]`*, `["subject",s]`, `["t",label]`*, `["c",commit]`, `["h",uuid]`? (UUID-validated and canonically re-rendered — test `builders.rs:3491-3517`), `["clone",url…]`, `["branch-name",b]`?, `["merge-base",c]`?, `["e",patch]`? | markdown ≤64 KiB (`builders.rs:1330-1393`) |
| 1619 | `build_git_pr_update` | `["a",coord]`, `["r",euc]`?, `["p",owner]`, `["p",recipient]`*, `["E",pr_event]`, `["P",pr_author]`, `["c",commit]`, `["clone",url…]`, `["merge-base",c]`? | markdown ≤64 KiB (`builders.rs:1416-1460`) |
| 30620 | `build_workflow_def` / `build_workflow_update` | `["d",workflow_uuid]`, `["h",channel_uuid]` | workflow YAML ≤64 KiB (`builders.rs:1463-1494`) |
| 46020 | `build_workflow_trigger` | `["d",workflow_uuid]` | empty (`builders.rs:1511-1514`) |
| 46030 / 46031 | `build_workflow_approval` | `["d",token_hash]` | free-text note (no length check) (`builders.rs:1522-1541`) |
| 41010 | `build_dm_open` | `["p",hex]` ×1–8 | empty (`builders.rs:1544-1556`) |
| 41011 | `build_dm_add_member` | `["h",uuid]`, `["p",hex]` | empty (`builders.rs:1559-1566`) |
| 20001 | `build_presence_update` | `["status",s]` | duplicate of the status string (`builders.rs:1570-1585`) |
| 9040 | `build_moderation_ban` | `["p",hex]`, `["expiration",unix]`?, `["reason",r]`? | empty (`builders.rs:1597-1611`) |
| 9041 | `build_moderation_unban` | `["p",hex]` | empty (`builders.rs:1614-1620`) |
| 9042 | `build_moderation_timeout` | `["p",hex]`, `["expiration",unix]`, `["reason",r]`? | empty (`builders.rs:1623-1637`) |
| 9043 | `build_moderation_untimeout` | `["p",hex]` | empty (`builders.rs:1640-1646`) |
| 9044 | `build_moderation_resolve_report` | `["report",id]`, `["status",s]`, `["action",a]`, `["reason",r]`? | empty (`builders.rs:1654-1690`) |
| 9035 | `build_archive_identity_request` | `["-"]`, `["p",target]`, `["reason",code]`?, `["replaced-by",pk]`?, `["auth",owner,conditions,sig]`? | optional human text ≤64 KiB (`builders.rs:1788-1802`, tags `1739-1786`) |
| 9036 | `build_unarchive_identity_request` | `["-"]`, `["p",target]`, `["reason",code]`?, `["auth",…]`? (no `replaced-by`) | optional human text ≤64 KiB (`builders.rs:1810-1823`) |

`*` = repeatable, `?` = conditional.

Notable serde/JSON shapes:
- Kind 0 content is assembled as a `serde_json::Map` and stringified, so absent
  options are **omitted keys**, not `null` (`crates/buzz-sdk/src/builders.rs:542-561`).
- The NIP-OA `auth` tag is passed to builders as a `&[String; 4]` fixed array
  (`crates/buzz-sdk/src/builders.rs:1723-1737`), and produced/consumed by
  `nip_oa` as a **JSON array string** `["auth",owner_hex,conditions,sig_hex]`
  (`crates/buzz-sdk/src/nip_oa.rs:146-176`, `252-299`).

---

### 5. Builder ↔ kind relationships worth noting

- Kind 9002 is emitted by five distinct builders (`build_update_channel`,
  `build_set_topic`, `build_set_purpose`, `build_archive`, `build_unarchive`) —
  they differ only in tag vocabulary (`builders.rs:604-724`).
- Kind 5 is emitted by three builders with three different targeting schemes:
  `h`+`e` (`builders.rs:434`), `e` only (`builders.rs:495`), and `a` coordinate
  (`builders.rs:1498`).
- `build_workflow_def` and `build_workflow_update` are byte-identical in body
  (`builders.rs:1463-1494`).
- `extract_channel_id(&nostr::Event) -> Option<Uuid>` is the only reader
  (inverse) helper in the crate (`crates/buzz-sdk/src/builders.rs:816-826`).
