## Module: buzz-cli — channel, message & social commands (`crates/buzz-cli/src/commands`)
### Aspect: Debt
The dominant debt in these ten files is **duplication of read-path logic** —
three different kind:39000 parsers in one file, two independent `resolve_author`
implementations, two byte-identical compact-output projections — plus a small
number of tests that assert against a copy of a production rule rather than the
rule itself. There is no `TODO`/`FIXME` backlog and no `unsafe`; the problems are
structural.
#### File size against the documented 1,000-line ceiling
| File | Total | Production (pre-`#[cfg(test)]`) | Test region | vs 1,000 |
|---|---|---|---|---|
| `channels.rs` | 1,713 | 1,175 (`#[cfg(test)]` at `:1176`) | 538 (31%) | 1.7× total, **1.2× production alone** |
| `notes.rs` | 1,330 | 820 (`:821`) | 510 (38%) | 1.3× total, production under |
| `messages.rs` | 1,167 | 876 (`:877`) | 291 (25%) | 1.2× total, production under |
| `emoji.rs` | 389 | 325 (`:326`) | 64 | under |
| `social.rs` | 284 | 238 (`:239`) | 46 | under |
| `channel_templates.rs` | 195 | 125 (`:126`) | 70 | under |
| `reactions.rs` | 138 | 138 | 0 | under |
| `dms.rs` | 136 | 136 | 0 | under |
| `feed.rs` | 80 | 80 | 0 | under |
| `mod.rs` | 20 | 20 | 0 | under |
`AGENTS.md § Mobile App` states a "**Hard ceiling: 1000 lines/file**, enforced by
`mobile/scripts/check-file-sizes.mjs` via `just mobile-check` (runs in `just
check` + pre-push, mirroring desktop/web)" and instructs "**split the file —
never bump the limit**". **No Rust gate enforces it.** Proof: the only file-size
checkers in the repo are `desktop/scripts/check-file-sizes.mjs`,
`web/scripts/check-file-sizes.mjs` and `mobile/scripts/check-file-sizes.mjs`;
`grep -rln 'crates/'` across all three returns zero, and the `justfile` wires
them only at `justfile:123` (desktop), `:585` (web) and `:617` (mobile). So
`channels.rs` at 1,713 lines is 71% over a ceiling that is documented as
repo-wide policy and implemented for three of four language surfaces.
#### Oversized functions
| Function | Lines | Range | Note |
|---|---|---|---|
| `channels.rs::cmd_create_channel_from_template` | 133 | `:655-787` | 8 params behind `#[allow(clippy::too_many_arguments)]` (`:654`); does flag validation, template load, roster resolution, channel create, canvas apply, and a sequential member-add loop |
| `messages.rs::dispatch` | 123 | `:754-876` | pure clap-enum-to-struct plumbing |
| `messages.rs::cmd_send_message` | 113 | `:483-595` | stdin, upload loop, media markdown, thread resolution, two mention pipelines, four-arm builder match |
| `channels.rs::dispatch` | 102 | `:1066-1167` | |
| `notes.rs::cmd_set` | 96 | `:487-582` | stdin cap, read-before-write, submit, LWW conflict detection, five-line receipt |
| `emoji.rs::cmd_import` | 75 | `:234-308` | numbered 8-step comment block is doing the work a decomposition should |
| `messages.rs::resolve_content_mentions` | 75 | `:128-202` | 5 numbered stages, none extracted |
Inconsistent remedy for the same problem inside one module: `messages.rs`
introduces parameter structs (`SendMessageParams` at `:474-481`, `SendDiffParams`
at `:577-590`) while `channels.rs` reaches for
`#[allow(clippy::too_many_arguments)]` twice (`:654`, `:794`).
#### Duplication inside the ten files
**Three parsers for kind:39000, all in `channels.rs`.**
| Parser | Site | Fields extracted |
|---|---|---|
| `extract_channel_metadata` | `:16-24` | `channel_id`, `name`, `description` (from `about`), `created_at` |
| `ChannelSummary::from_event` | `:166-211` | `d`, `name`, `t`, `private`/`public`, `about`, `topic`, `purpose`, `archived` |
| inline `--visibility` post-filter | `:68-89` | single-element `public`/`private` tag detection only |
Consequences, not just redundancy: `channels get` (`:224`, via
`extract_channel_metadata`) returns strictly less than `channels search`
(`:119`, via `ChannelSummary`) for the *same event* — no `topic`, `purpose`,
`archived`, `channel_type` or `visibility`. And `channels list` has no archived
filter at all while `channels search` excludes archived by default
(`:136`), so the two read commands over kind:39000 disagree on defaults. The
inline filter at `:68-89` re-derives the single-element-tag rule that
`ChannelSummary::from_event:181-184` already implements. The vocabulary also
splits: `--visibility open` is mapped to the wire value `public` at `:71-73`,
but `ChannelSummary.visibility` reports `"public"` verbatim (`:183`), so input
and output speak different words for the same state.
**Two `resolve_author` implementations.**
| | `messages.rs:394-440` | `notes.rs:204-252` |
|---|---|---|
| Returns | `String` (hex) | `PublicKey` |
| Accepts `"me"` | no | yes (`notes.rs:205-207`) |
| Accepts `npub1…` | yes (`:400-404`) | no |
| Profile query | `{kinds:[0], search, limit:100}` (`:406-410`) | `{kinds:[0], search, limit:100}` (`:213-217`) — identical filter |
| Name match | `match_profiles_by_name` (`:441-476`), extracted + tested | inline closure (`:220-234`), untested |
| Ambiguity error | caps the candidate list at 5 with "… and N more" (`:425-436`) | prints only the count (`:249-251`) |
The exact-case-insensitive `display_name`-or-`name` comparison is written twice
(`messages.rs:456-461`, `notes.rs:224-233`) with the same semantics. `notes.rs`
also has a *third* author-facing formatter, `format_note_candidates`
(`:279-296`), doing the same "list the ambiguous candidates" job in a different
shape.
**Two byte-identical compact projections.** `messages.rs::format_events`
(`:242-262`) and the inline block in `feed.rs:45-64` both build
`{id, content, created_at}` from a normalized array and both fall back to
`unwrap_or_default()`. Neither is shared; `channels.rs:95-109` adds a third,
differently-shaped compact branch. All three live downstream of the same
`--format` flag.
**Two `p`-tag extractors.** `messages.rs::parse_member_pubkeys` (`:226-241`,
canonicalizing through `PublicKey::from_hex`) and the inline closure in
`dms.rs:20-38` (raw strings, no validation) — while `client.rs:1366`
already exports `extract_p_tags`, which `channels.rs:250` does use.
**Duplicated 1–8 pubkey rule.** `dms.rs:52-54` rejects `pubkeys.is_empty() ||
pubkeys.len() > 8`; `buzz_sdk::build_dm_open` enforces exactly the same bound at
`builders.rs:1544-1548`. The CLI never calls `build_dm_open` — it hand-builds
`EventBuilder::new(Kind::Custom(41010), "")` at `dms.rs:69` — so the SDK builder
is dead from this module and its validation is reimplemented.
**Duplicated relay-response parsing.** `notes.rs:544-563` and `notes.rs:723-736`
each hand-parse `accepted`/`message` out of the submit response, and
`notes.rs:557-562` reimplements the `duplicate:` LWW-conflict detection that
`client::normalize_write_response` (`client.rs:1420`) exists to centralize —
the function every other write path in scope calls.
#### Dead and vestigial code
| Item | Evidence |
|---|---|
| `notes.rs::KIND_LONG_FORM` | `notes.rs:38` redeclares `30023`, already `buzz_core::kind::KIND_LONG_FORM` at `kind.rs:66` |
| `emoji.rs::CUSTOM_EMOJI_SET_D_TAG` | `emoji.rs:9` is `const … = buzz_sdk::CUSTOM_EMOJI_SET_D_TAG;` — a pure alias |
| `notes.rs::snapshot_from_event` | `:338-340` is a one-line wrapper around `NoteSnapshot::from_event` with no added behavior |
| `notes.rs:678` `author.unwrap_or("me")` | unreachable — clap supplies `default_value = "me"` (`lib.rs:1069-1071`), so the `Option` is always `Some` |
| `channels.rs` `member: Option<bool>` | `dispatch` always passes `Some(member)` (`:1079`); `cmd_list_channels` tests `member == Some(true)` (`:34`). The `Option` layer can never be `None` |
| `buzz_sdk::build_dm_open` | never called from `buzz-cli`; `dms.rs:69` hand-builds the same event |
No `#[allow(dead_code)]` standing in for a `TODO`: `grep -rn '#\[allow'` over the
ten files returns exactly two lines, both `#[allow(clippy::too_many_arguments)]`
(`channels.rs:654`, `:794`). `grep -rnE 'TODO|FIXME|XXX|HACK'` over the ten files
returns **zero matches**.
#### Stale comments and cross-references
| Comment | Site | Why it is wrong |
|---|---|---|
| "1. Replies referencing this event via e-tag (**no kind restriction**)" | `messages.rs:316-318` | The very next statement restricts kinds to `[9, 40002, 40003, 40008, 45003]` (`:320`) |
| "`build_dm_open` doesn't accept a d-tag, so we build the event manually **using the SDK builder** and add the d-tag ourselves" | `dms.rs:66-67` | No SDK builder is used — `dms.rs:69` calls `EventBuilder::new(Kind::Custom(41010), "")` directly. The "add a d-tag" motive is real; the mechanism described is not |
| "The relay keeps only the latest per (pubkey, d_tag), **but be defensive**" | `emoji.rs:104` | `events.last()` (`:105`) is not defensive — the filter already carries `limit: 1` (`:102`), and if several events did come back, `.last()` picks by array position, not `created_at`. Compare `notes.rs:180-182` and `:361-362`, which sort `Reverse(created_at)` for exactly this case |
| "the relay's #714 coordinate soft-delete **lands in this same change**" | `notes.rs:24-25` | An in-flight-PR note left behind after merge. The relay code exists: `handle_a_tag_deletion` at `buzz-relay/src/handlers/side_effects.rs:1979`, `handle_standard_deletion_event` at `:2108` |
In-code cross-references were checked individually and all **resolve**:
`desktop/src-tauri/src/templates/types.rs` (`channel_templates.rs:5`) exists;
`useApplyTemplate.ts` (`channels.rs:736`) resolves to
`desktop/src/features/channel-templates/useApplyTemplate.ts`; `channelAgents.ts`
(`channels.rs:741`) to `desktop/src/features/agents/channelAgents.ts`; and the
`req.rs` `#d`-pushdown claim (`notes.rs:185-187`) is accurate —
`buzz-relay/src/handlers/req.rs:936-977` pushes a single-value `#d` into the
`d_tag` column, gated on `filter_is_nip33_only` (`:951`), which
`notes.rs::fetch_by_slug` satisfies. The two desktop references cite bare
filenames with no path, which will not survive a file move.
#### Tests that assert against a copy of the rule
This is the module's most consequential test debt.
1. **`BUZZ_ACP_ALLOWED_CHANNEL_ADD_POLICIES` gate.** `channels.rs:1296-1307`
   defines a test-local `check_allowed_channel_add_policy` whose body is a
   verbatim copy of the production gate at `channels.rs:1022-1033` — same split,
   same trim, same filter, same message template. Three tests exercise the copy:
   `set_add_policy_rejects_disallowed_policy` (`:1312`),
   `set_add_policy_accepts_allowed_policy` (`:1330`),
   `set_add_policy_no_restriction_allows_all` (`:1336`). Deleting the gate from
   `cmd_set_add_policy` leaves all three green. The file's own comment at
   `:1345-1350` admits this and adds
   `set_add_policy_env_gate_rejects_disallowed_via_full_path` (`:1362`) as the
   one test that actually calls production. So the honest coverage is 1 real test
   plus 3 tautologies.
2. **Mention pipeline.** `cli_pipeline_resolves_multiword_display_names`
   (`messages.rs:1013`) rebuilds the `name_to_pubkeys` / `display_names` loop
   inline — its own comment says "Simulate the single-parse pipeline from
   `resolve_content_mentions`" (`:1025`). Production's loop lives at
   `messages.rs:161-190` and is never invoked by the test.
3. **The stated regression guard isn't one.**
   `cli_pipeline_resolves_body_at_names_to_member_pubkeys` (`messages.rs:973`)
   documents itself as "the regression guard for the previous stub that always
   returned `vec![]`" (`:969-971`), but it calls `extract_at_names` +
   `match_names_to_profiles` from the SDK, whereas production calls
   `extract_at_mentions_with_known` + a local map (`messages.rs:192-201`). It
   would stay green if `resolve_content_mentions` reverted to a stub.
#### Untested critical paths, grep-proven
| Path | Proof |
|---|---|
| All of `reactions.rs`, `dms.rs`, `feed.rs` | `grep -c '#\[cfg(test)\]'` → 0 for each; `grep -cE '#\[(tokio::)?test\]'` → 0 for each. Includes `reactions remove`'s emoji-match-and-delete (`reactions.rs:42-80`), `dms open`'s relay-id extraction (`dms.rs:72-92`), and `feed get`'s type allowlist (`feed.rs:29-40`) |
| `--format compact` output shaping | `grep -n 'format_events' messages.rs` → definition `:242` and production call sites `:300`, `:336`, `:383`, all above the test boundary at `:877`. `feed.rs` and `channels.rs`' compact branches have no test module / no test importing `cmd_list_channels` (`channels.rs:1178-1184` import list) |
| `emoji import` merge / replace / dedup / dry-run | `grep -n 'cmd_import\|read_source\|write_output' emoji.rs` → only definitions and production call sites (`:164`, `:185`, `:230`, `:234`, `:241`, `:322`); the two tests in the file (`:330`, `:369`) both target `union_custom_emoji` |
| `messages.rs` relay-touching helpers | `grep -n 'resolve_thread_ref\|resolve_channel_id\|resolve_content_mentions'` → all hits are definitions or production call sites (`:58`, `:89`, `:128`, `:524`, `:531`, `:635`, `:679`, `:710`, `:741`); none inside the test module |
| `channel_templates.rs::resolve_templates_path` dev-bundle case | the two path tests (`:139`, `:145`) cover only the override and the prod default; nothing covers `xyz.block.buzz.app.dev` |
| `social.rs` cursor pairing | the five tests (`:242-283`) cover only `validate_social_list_kind`, `parse_tags_json`, `has_d_tag`, `is_parameterized_social_list_kind`. `cmd_get_user_notes` (`:83-113`) is untested |
The `notes` command group is additionally unguarded by the CLI drift tests.
`grep -n 'names(&cmd, ' crates/buzz-cli/src/lib.rs` returns 19 calls covering
`agents, messages, channels, canvas, reactions, emoji, dms, users, workflows,
feed, social, repos, pr, patches, issues, media, upload, pack, moderation` —
`notes` is **not** among them (the `"notes"` string at `lib.rs:1940` is the
`social notes` subcommand inside the `social` list). And
`subcommand_counts_are_stable` (`lib.rs:1996-2033`) enumerates 18 groups,
omitting `mem`, `moderation` and `notes`. `command_inventory_is_stable`
(`lib.rs:1807`) does list `"notes"` at `:1820`, so the *group's existence* is
guarded but none of its four subcommands is. Adding or renaming a `notes` verb
breaks no test.
#### `unwrap()` / `expect()` on production paths
`AGENTS.md § Quality Gates` says "Do not introduce new `unwrap()` or `expect()`
in production paths — use `?` and proper error types". Scanning only the region
above each file's `#[cfg(test)]`:
| Site | Call | Assessment |
|---|---|---|
| `channels.rs:148` | `serde_json::to_string(&matches).expect("serializing ChannelSummary")` | Violates the letter of the rule. Practically unreachable (`ChannelSummary` is all `String`/`Option<String>`/`bool`), but the same file uses `unwrap_or_default()` for the identical operation at `:105` and `:108`, so the module contradicts itself on the idiom |
| `notes.rs:632` | `name.expect("dispatch enforces --name xor --naddr")` | The invariant is real — `validate_get_args` runs at `notes.rs:795` before `cmd_get` — but it is enforced by a *sibling call site*, not by the type. Any second caller of `cmd_get` panics |
| `channels.rs:314`, `:319`, `:703`, `:708` | `unreachable!()` in `match visibility` / `match channel_type` | Guarded by an immediately preceding validating `match` (`:288-305`, `:673-687`). Correct today; a panic-shaped assertion that a `TryFrom` or an enum-typed parameter would make impossible |
Everything else propagates with `?`. `messages.rs`, `emoji.rs`, `social.rs`,
`channel_templates.rs`, `reactions.rs`, `dms.rs`, `feed.rs` and `mod.rs` have
**zero** `unwrap`/`expect`/`panic!`/`unreachable!` in their production regions.
#### Output-contract drift against `AGENTS.md`
`AGENTS.md § Agent CLI` states: "All reads return sig-stripped JSON arrays; all
writes return `{event_id, accepted, message}`". Both halves are violated in
scope.
| Command | Actual output | Site |
|---|---|---|
| `social event` | raw relay response, **`sig` included** — `normalize_events` is never imported by `social.rs` (`social.rs:8`) | `social.rs:78` |
| `social notes` | raw relay response, `sig` included | `social.rs:111` |
| `social contacts` | raw relay response, `sig` included | `social.rs:122` |
| `social list` | raw relay response, `sig` included | `social.rs:205` |
| `social set-list` | raw relay response, not `normalize_write_response` | `social.rs:180` |
| `emoji list` / `export` | `{"emojis": [...]}` — an object, not an array | `emoji.rs:88`, `:228` |
| `reactions get` | `{"reactions": [...]}` — an object, not an array | `reactions.rs:135` |
| `emoji rm` (no-op case) | `{"accepted": true, "message": "not present"}` — **no `event_id`** | `emoji.rs:145-149` |
| `dms open` | hand-assembled object; injects `accepted: true` when the relay omitted it (`dms.rs:90-92`) | `dms.rs:88-93` |
| `notes set` / `rm` | five and two lines of plain-text `key value` receipts | `notes.rs:566-581`, `:738-739` |
| `canvas get` | bare content string or the literal `null` | `channels.rs:270-277` |
| `channels get` | bare object or `null`, not an array | `channels.rs:234-240` |
`normalize_events` (`client.rs:1307-1322`) is the sig-stripping helper; only
`messages.rs` (`:299`, `:335`, `:382`) and `feed.rs` (`:44`) call it.
Also drifting: `social notes --before-id` is accepted without `--before`
(`social.rs:91-93` validates the hex but not the pairing) even though
`buzz-relay/src/api/bridge.rs:1240-1245` returns `400 before_id requires until
to be set`. The caller gets exit **2** (relay/network) for what the exit-code
table at `lib.rs:74` classifies as exit **1** (bad input). Contrast
`messages send-diff`, which *does* validate its own flag pairing locally
(`messages.rs:589-596`).
#### The four `AGENTS.md` leads, verified
**Filters without `kinds`.** Five exist in scope:
| Site | Filter | Trips the p-gate? |
|---|---|---|
| `messages.rs:63` | `{ids, limit}` | No — `ids` exemption, `req.rs:1063-1065` |
| `messages.rs:91-93` | `{ids}` | No — same |
| `messages.rs:328-331` | `{ids, limit}` | No — same |
| `social.rs:74-76` | `{ids}` | No — same |
| `feed.rs:19-22` | `{#p:[self], limit}` | No — `#p` equals the authed pubkey, `req.rs:1067-1069` |
So none of them 403s, but only because both documented exemptions happen to
apply. `AGENTS.md § Common Gotchas #2` states the rule flatly ("omitting `kinds`
triggers the p-gate (403). Always include explicit kind filters"), which
contradicts the actual predicate in
`req.rs::p_gated_filters_authorized:1038-1071`: a kindless filter is only
rejected when it has neither non-empty `ids` nor a self-matching `#p`. Five
in-scope call sites depend on that nuance, undocumented.
**Gotcha #3 documents a flag that does not exist.** `AGENTS.md` says
"`messages search` must include `--kinds` … Pass at least `--kinds 9,45001,45003`".
`MessagesCmd::Search` (`lib.rs:476-489`) declares `query`, `author`, `since`,
`limit` — no `kinds`. Confirmed empirically: `./target/debug/buzz messages search
--kinds 9` → `{"error":"user_error","message":"error: unexpected argument
'--kinds' found …"}`. `cmd_search` hardcodes `kinds: [9, 40002, 45001, 45003]`
at `messages.rs:361`, so the failure mode the gotcha warns about cannot occur.
The same stale text is duplicated in `CLAUDE.md:172` (both docs carry the
identical Common Gotchas block).
**`reply_count` / `descendant_count`.** `AGENTS.md § Key Patterns` requires that
"any code that inserts replies must update these counters". Not applicable here
and correctly so: `grep -rn 'reply_count\|descendant_count' crates/buzz-cli/`
returns **zero matches**. The CLI publishes signed events over
`POST /events`; materialization is the relay's job. No debt — but also no
in-code note pointing a future contributor at that division of labor.
**Kind `39000`, not `41`.** Honored. Channel metadata reads use `39000` at
`channels.rs:54`, `:62`, `:131`, `:227`; there is no bare `41` kind anywhere in
scope. The `41xxx` literals in `dms.rs` (`41001` at `:12`, `41010` at `:69`) are
the distinct DM kinds `KIND_DM_CREATED` / `KIND_DM_OPEN` (`kind.rs:453`, `:447`),
not NIP-01 kind 41. The residual debt is that they are written as literals when
the constants exist — see the Configuration aspect.
**`h`-tag scoping.** Applied inconsistently, some of it deliberately:
| Read | `#h` present? | Site |
|---|---|---|
| `messages get` | yes | `messages.rs:263-268` |
| `messages thread` — reply half | yes | `messages.rs:319-324` |
| `messages thread` — root half | **no** (`ids` only) | `messages.rs:328-331` |
| `messages search` | **no** — cross-channel by design | `messages.rs:360-363` |
| `canvas get` | yes | `channels.rs:265-268` |
| `reactions get` / `remove` | **no** — `#e` (+ `authors`) only | `reactions.rs:83-86`, `:44-48` |
| `feed get` | **no** — `#p` only | `feed.rs:19-22` |
| `dms list` | **no** — `#p` only | `dms.rs:11-15` |
| `notes` (all) | **no** — kind:30023 is not channel-scoped | `notes.rs:171`, `:189`, `:354`, `:681` |
| `channels list` / `get` / `members` | `#d`, not `#h` | `channels.rs:38`, `:55`, `:228`, `:250` |
The `messages thread` root half is the one real gap: the reply half is scoped to
`channel_id` but the root is fetched by bare id, so passing a `--channel` that
does not contain `--event` still returns the root event, silently, in a payload
the caller will read as belonging to that channel. Only relay-side access
control stops a cross-channel read; `cmd_get_thread` itself never checks that the
root's `h` tag matches the `channel_id` it validated at `:312`.
#### Other correctness-adjacent debt
- `reactions.rs:44-48` queries kind:7 with no `limit`, then takes the **first**
  array element whose content matches the emoji (`:56-64`) with no `created_at`
  ordering. If a user reacted, removed, and re-reacted, which reaction gets
  deleted depends on relay row order.
- `notes.rs:191` caps the cross-author `#d` fan-out at `limit: 50`. With more
  than 50 authors on a slug, `notes get --name <slug> --latest` picks the newest
  of a *truncated* set and reports success. The `--latest` doc
  (`lib.rs:1057-1060`) promises "the most recently updated note" unqualified.
- `channel_templates.rs` reads and parses the whole store **twice** on the
  not-found path (`:102` inside `find_template`, `:114` inside
  `available_names`), so a corrupt store produces a parse error from the
  *error-message* code path rather than the lookup.
- `channels.rs:12` imports `fetch_archived_snapshot` from
  `crate::commands::agents` — an in-scope file depending on an out-of-scope
  sibling module for the NIP-IA trust check that
  `resolve_roster_with_archive_filter` (`:526`) is built around.
#### Stated uncertainties
- I did not run `cargo test -p buzz-cli`; every coverage claim above is derived
  from reading the `#[cfg(test)]` regions and from the greps quoted inline.
- The `--format` and `messages search --kinds` empirical checks used the
  prebuilt `target/debug/buzz` (dated 2025-07-26). Both results also follow
  directly from `lib.rs:92-94` and `lib.rs:476-489` as read, so I am confident,
  but they are not from a fresh build.
- Whether the `messages thread` root-half gap is *exploitable* depends on relay
  access control for `POST /query` by bare id, which I traced only as far as the
  p-gate and the accessible-channel scoping at `bridge.rs:997-1001`. I did not
  confirm whether a bare-`ids` lookup is intersected with
  `accessible_channels`, so I am flagging the CLI-side omission, not asserting a
  cross-channel read is possible.
