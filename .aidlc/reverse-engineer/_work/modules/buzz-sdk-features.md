## Module: buzz-sdk (`crates/buzz-sdk`)

### Aspect: Features

The crate's stated scope: build validated, unsigned Nostr events for Buzz
operations; the caller signs and transports them
(`crates/buzz-sdk/src/lib.rs:5-13`). Every capability below is therefore
"produce the correct wire form", never "perform the operation".

---

### 1. Capabilities driveable through the SDK

| Capability area | Builders / functions | Kinds | Completeness | Evidence |
|---|---|---|---|---|
| Stream chat messaging | `build_message` | 9 | Full — threading, mentions, broadcast flag, imeta media | `builders.rs:219-241` |
| Forum discussions | `build_forum_post`, `build_forum_comment`, `build_vote` | 45001, 45003, 45002 | Full for post/comment/vote | `builders.rs:278-305`, `446-460` |
| Message editing | `build_edit` | 40003 | Full | `builders.rs:378-389` |
| Message deletion | `build_delete_message`, `build_delete_message_with_options`, `build_delete_compat` | 9005, 5 | Full, incl. moderation tombstone metadata | `builders.rs:403-443` |
| Reactions | `build_reaction`, `build_custom_emoji_reaction`, `build_remove_reaction` | 7, 5 | Full (NIP-25 + NIP-30) | `builders.rs:463-498` |
| Custom emoji palette | `build_custom_emoji_set`, `normalize_custom_emoji_shortcode`, `CUSTOM_EMOJI_SET_D_TAG` | 30030 | Partial — write-side only; the doc states add/remove is "read-own-set → mutate → rebuild", and the read/union step is left to callers | `builders.rs:503-527` |
| Code-diff sharing | `build_diff_message` + `DiffMeta` | 40008 | Full metadata surface (repo, commit, file, branch pair, PR number, language, truncation, alt text) | `builders.rs:308-375` |
| Canvas documents | `build_set_canvas` | 40100 | Partial — no content-length validation, unlike every other content builder | `builders.rs:529-532` |
| Profiles (NIP-01) | `build_profile` | 0 | Partial — only 5 fields (`display_name`, `name`, `picture`, `about`, `nip05`); other kind-0 fields cannot be set | `builders.rs:537-562` |
| Channel lifecycle | `build_create_channel`, `build_update_channel`, `build_set_topic`, `build_set_purpose`, `build_archive`, `build_unarchive`, `build_delete_channel` | 9007, 9002, 9008 | Full for name/about/visibility/type/ttl/topic/purpose/archive | `builders.rs:604-730` |
| Membership | `build_add_member`, `build_remove_member`, `build_join`, `build_leave` | 9000, 9001, 9021, 9022 | Full | `builders.rs:565-598`, `703-706` |
| Direct messages | `build_dm_open`, `build_dm_add_member` | 41010, 41011 | Partial — conversation setup only; no DM message-body or gift-wrap builder in this crate | `builders.rs:1544-1566` |
| Presence | `build_presence_update` | 20001 | Full for the three-state vocabulary | `builders.rs:1570-1585` |
| Global social notes | `build_note` | 1 | Partial by design — flat reply model; "Full NIP-10 threading (root + reply + p-tags) is deferred" | `builders.rs:732-748` |
| Contact lists (NIP-02) | `build_contact_list` | 3 | Full replacement semantics; deltas require caller read-before-write | `builders.rs:753-813` |
| Git repo announcement (NIP-34) | `build_repo_announcement`, `build_repo_announcement_with_tags` | 30617 | Full, incl. read-modify-write path preserving unknown tags | `builders.rs:828-963` |
| Git patches | `build_git_patch` + `GitPatchMeta` | 1617 | Full tag surface (euc, series markers, commit/parent, PGP sig, committer identity) | `builders.rs:1007-1069` |
| Git issues | `build_git_issue` + `GitIssueMeta` | 1621 | Full (subject, labels, recipients) | `builders.rs:1081-1111` |
| Git status transitions | `build_git_status`, `GitStatus`, `GitStatusMeta`, `GitAppliedPatchRef::parse` | 1630–1633 | Full, incl. merge-only fields and `q`-tag relay/pubkey hints | `builders.rs:1114-1299` |
| Git pull requests | `build_git_pull_request`, `build_git_pr_update` | 1618, 1619 | Full event shape; explicitly does **not** verify commit reachability or perform network work | `builders.rs:1301-1460` |
| Workflows | `build_workflow_def`, `build_workflow_update`, `build_workflow_delete`, `build_workflow_trigger`, `build_workflow_approval` | 30620, 5, 46020, 46030/46031 | Full CRUD + trigger + approve/deny | `builders.rs:1462-1541` |
| Community moderation | `build_moderation_ban`/`unban`/`timeout`/`untimeout`/`resolve_report` | 9040–9044 | Full command surface; status↔action pairing deliberately delegated to the relay | `builders.rs:1587-1690` |
| Identity archival (NIP-IA) | `build_archive_identity_request`, `build_unarchive_identity_request` | 9035, 9036 | Full request shape; consent-path selection is relay-side | `builders.rs:1692-1823` |
| Agent observer streaming | `build_agent_observer_frame` | 24200 | Partial — validates that content *looks like* NIP-44 ciphertext but does not encrypt; encryption lives in `buzz_core::observer` | `builders.rs:243-274`; `crates/buzz-core/src/observer.rs:53-55` |
| Mention resolution | `extract_at_names`, `extract_at_mentions_with_known`, `match_names_to_profiles`, `merge_mentions`, `normalize_mention_pubkeys`, `strip_code_regions`, `extract_nostr_uris` | n/a | Full pure-function pipeline; membership/profile querying is the caller's job | `mentions.rs:1-30`, `64-387` |
| NIP-OA owner attestation | `compute_auth_tag`, `verify_auth_tag`, `parse_auth_tag` | n/a (tag) | Full sign/verify/parse with spec test vector pinned | `nip_oa.rs:146-299`; vector `nip_oa.rs:313-333`, `nip_oa.rs:487-494` |
| Event reading | `extract_channel_id` | n/a | Minimal — the crate is write-oriented; this is the only inbound-event helper | `builders.rs:816-826` |

Kind coverage: **35 distinct event kinds** across 51 public builder functions
(kinds 0, 1, 3, 5, 7, 9, 1617, 1618, 1619, 1621, 1630, 1631, 1632, 1633, 9000,
9001, 9002, 9005, 9007, 9008, 9021, 9022, 9035, 9036, 9040, 9041, 9042, 9043,
9044, 20001, 24200, 30030, 30617, 30620, 40003, 40008, 40100, 41010, 41011,
45001, 45002, 45003, 46020, 46030, 46031 — 45 kind integers, 35 unique
kind *families* if the four Git status kinds and the two approval kinds are each
counted once).

---

### 2. Explicitly stubbed / deferred behavior

| Item | Statement in code | File:line |
|---|---|---|
| Full NIP-10 threading for kind 1 | "This is intentionally simpler than the full `ThreadRef` mechanism used for channel messages — social notes use a flat reply model for now. Full NIP-10 threading (root + reply + p-tags) is deferred." | `builders.rs:732-737` |
| PR tip reachability | "this builder does no network work and does not verify reachability" | `builders.rs:1311-1315` |
| NIP-OA verification depth in builders | "Structural check only — the relay performs full NIP-OA verification." | `builders.rs:1722-1723` |
| NIP-IA consent path | "The relay verifies; this builder's job is to produce a well-formed, signed request — the relay selects the consent path (self / admin / owner)." | `builders.rs:1697-1699` |
| Moderation status/action pairing | "(`dismiss` pairs with `dismissed`, everything else with `resolved` — the relay enforces the pairing)" | `builders.rs:1648-1651` |
| Emoji palette union | "The workspace palette shown in clients is the union of every member's set, deduped by `(shortcode, url)` on read." (read side not implemented here) | `builders.rs:508-510` |
| `parse_auth_tag` skips crypto | "This is the fast path used at MCP startup — no crypto is performed." | `nip_oa.rs:249-250` |
| `GitStatusMeta.recipients` defaulting | "GitStatusMeta.recipients is the caller's responsibility (the CLI defaults it)" | `builders.rs:3157-3159` (test comment) |

---

### 3. TODO / FIXME / HACK / XXX comments

A recursive search across `crates/buzz-sdk/` (all files, including tests and the
example) for `TODO`, `FIXME`, `HACK`, and `XXX` returned **zero matches**. There
are no such markers in this crate.

The deferred-work statements listed in section 2 are ordinary doc comments, not
tagged markers.

---

### 4. Feature-flag-gated capabilities

None. There is no `[features]` table in `crates/buzz-sdk/Cargo.toml` and no
`#[cfg(feature = ...)]` anywhere in the crate (verified by search across
`src/` and `examples/`). The only conditional compilation is `#[cfg(test)]` on
the three test modules (`builders.rs:1825`, `mentions.rs:389`,
`nip_oa.rs:301`).
