## Module: buzz-cli — channel, message & social commands (`crates/buzz-cli/src/commands`)
### Aspect: Business Rules

#### Enumerated value gates

| Rule | Accepted values | Site | Rejection |
|------|----------------|------|-----------|
| channel type | `stream`, `forum` | `channels.rs:283-290` | `Usage` |
| channel visibility | `open`, `private` | `channels.rs:291-298` | `Usage` |
| template type/visibility override | same two sets, re-checked after the template supplies defaults | `channels.rs:680-697` | `Usage` |
| member role | `owner`,`admin`,`member`,`guest`,`bot` | `channels.rs:964-976` | `Usage` |
| add policy | `anyone`,`owner_only`,`nobody` | `channels.rs:1006-1013` | `Usage` |
| message kind | `9` (or omitted), `45001`, `45003` | `messages.rs:540-572` | `Usage` at `:568` |
| vote direction | `up`,`down` | `messages.rs:730-737` | `Usage` |
| feed types | `mentions`,`needs_action`,`activity`,`agent_activity` | `feed.rs:6`, checked at `:31-39` | `Usage` |
| social list kind | `10000,10001,10002,10003,30000,30003` | `social.rs:127-139` | `Usage` |
| emoji export scope | `own`,`workspace` | clap enum `lib.rs:145-150`, matched at `emoji.rs:201`/`:216` | clap |

`channels create` and `channels create --template` duplicate the type/visibility
validation and the string→enum mapping four times total
(`channels.rs:283-298` + `:300-320` vs `channels.rs:680-697` + `:699-709`), each
ending in `unreachable!()` (`:314`, `:319`, `:703`, `:708`).

#### Mutual exclusions and required combinations

| Rule | Enforced where | Notes |
|------|---------------|-------|
| `channels create` needs `--type`+`--visibility` unless `--template` | clap `required_unless_present` (`lib.rs:551`,`:554`), re-checked defensively in dispatch (`channels.rs:1136-1139`) | dispatch comment says clap guarantees it |
| `channels update` needs at least one of `--name/--description/--ttl/--no-ttl` | `channels.rs:843-847` | SDK repeats the check (`builders.rs:610-614`) |
| `--ttl` vs `--no-ttl` | clap `conflicts_with` (`lib.rs:586`) plus the tri-state match at `channels.rs:836-840` | `Some(None)` clears TTL, emitting `["ttl",""]` (`builders.rs:643-646`) |
| `messages send-diff` branch pair | `messages.rs:601-608` — both or neither | |
| `messages search` needs `--query` or `--author` | `messages.rs:345-349` | |
| `messages send --kind 45003` needs `--reply-to` | `messages.rs:546-548` | |
| `notes get`: exactly one of `--naddr`/`--name`; `--author` and `--latest` only with `--name`; `--author` xor `--latest` | `validate_get_args` `notes.rs:588-610`, called at `notes.rs:794` | not expressed in clap, so absent from `--help` |
| `notes set`: `--clear-tags` xor `--tag` | `notes.rs:775-779` (in dispatch, not `cmd_set`) | |
| `dms open`: 1–8 pubkeys | `dms.rs:52-54` | re-implements `builders.rs:1544-1548`, which the CLI bypasses |
| `emoji list --format compact` | not a rule — silently ignored (no format arg reaches `emoji::dispatch`, `emoji.rs:311`) | |

A silent-drop case: `messages send --kind 45001 --reply-to <id>` resolves the
thread ref (`messages.rs:534-538`, costing a relay round trip) and then calls
`build_forum_post`, which takes no thread ref (`messages.rs:542`,
`builders.rs:278-289`). The reply linkage is discarded with no warning. Only
kinds 9 and 45003 thread.

#### `h`-tag scoping

Every channel-scoped write carries `h` and never `e` for the channel, matching
AGENTS.md. Reads are less uniform:

| Read | Channel scoping | Site |
|------|----------------|------|
| `messages get` | `#h` | `messages.rs:277` |
| `messages thread` (reply filter) | `#h` + `#e` | `messages.rs:321-322` |
| `messages thread` (root filter) | none — `ids` only | `messages.rs:328-331` |
| `canvas get` | `#h` | `channels.rs:266` |
| `channels get`/`members`, mention resolution | `#d` (39000/39002 are NIP-33-addressed by channel UUID) | `channels.rs:228`,`:251`, `messages.rs:140` |
| `messages search` | **no channel scoping at all** | `messages.rs:360-372` |
| `reactions get`/`remove` | `#e` only | `reactions.rs:83-85`, `:45-48` |
| `feed get` | `#p` only | `feed.rs:19-22` |
| `social` reads, `notes` reads | not channel-scoped by design (global kinds) | `social.rs:74-121`, `notes.rs:170-192` |

`messages search` is community-wide: the relay intersects with accessible
channels server-side (`bridge.rs:1661-1675`), so there is no client-side
`--channel` filter available. `buzz messages search --query x` cannot be scoped
to one channel through this CLI.

#### `kinds` in filters and the p-gate

AGENTS.md says a query without `kinds` "triggers the p-gate (403)". The actual
rule (`req.rs:1038-1071`, `p_gated_filters_authorized`) is: a kindless filter is
treated as *possibly* matching a p-gated kind, and then passes if it has non-empty
`ids`, or if every `#p` value equals the authed pubkey. Checked against the four
kindless filters in scope, all pass legitimately:

| Kindless filter | Why it passes |
|-----------------|---------------|
| `social event` `{ids:[…]}` (`social.rs:74-76`) | `ids` exemption (`req.rs:1064`) |
| `resolve_thread_ref` `{ids:[…],limit:1}` (`messages.rs:63`) | `ids` exemption |
| `resolve_channel_id` `{ids:[…]}` (`messages.rs:90-92`) | `ids` exemption |
| `feed get` `{#p:[me],limit:N}` (`feed.rs:19-22`) | `#p` equals self (`req.rs:1068-1070`) |

AGENTS.md gotcha 3 ("`messages search` must include `--kinds`, pass at least
`--kinds 9,45001,45003`") is stale: `MessagesCmd::Search` has no `--kinds` flag
(`lib.rs:475-491`) and `cmd_search` hardcodes `kinds: [9,40002,45001,45003]`
(`messages.rs:361`). `crates/buzz-cli/TESTING.md:223` documents the working form
without `--kinds`, so the code and the runbook agree and AGENTS.md is the outlier.

The three message read paths use three different kind sets:

| Command | Kinds | Site |
|---------|-------|------|
| `messages get` | 9, 40002, 40008, 45001, 45003 | `messages.rs:276` |
| `messages thread` | 9, 40002, 40003, 40008, 45003 | `messages.rs:320` |
| `messages search` | 9, 40002, 45001, 45003 | `messages.rs:361` |

So `get` omits edits (40003), `thread` omits forum roots (45001) but includes
edits, and `search` omits both edits and diffs (40008). Nothing in the code
explains the divergence.

`messages get --kinds` overrides the default set, parsed with
`filter_map(|s| s.trim().parse().ok())` (`messages.rs:284`). Unparseable entries
are dropped silently, and if *every* entry is bad the `if !kind_list.is_empty()`
guard (`:285`) leaves the default kinds in place — `--kinds nine` returns the
default result set instead of erroring.

#### Threading rules

`resolve_thread_ref` (`messages.rs:58-86`) treats `--reply-to` as the **immediate
parent** and derives the root from the parent's own NIP-10 tags via a relay
fetch. `find_root_from_tags` (`messages.rs:25-56`) prefers an explicit `root`
marker, falls back to a `reply` marker, ignores unmarked `e` tags (NIP-10
deprecated positional markers), and defensively ignores marker values that aren't
64-hex (`messages.rs:39-45`) so a malformed parent can't block a send. When no
marker is found, root == parent (`messages.rs:81-84`). The SDK then emits one
`["e",root,"","reply"]` for a direct reply, or root+reply pairs for nested
(`builders.rs:171-181`).

`reply_count` / `descendant_count`: AGENTS.md requires any code inserting replies
to maintain these materialized counters. `grep -rn 'reply_count\|descendant_count'
crates/buzz-cli/src` returns **zero matches** — the CLI never reads or writes
them, delegating entirely to the relay's ingest path. That is consistent with the
CLI submitting signed events rather than rows, but it means the CLI has no
guardrail if the relay ever stops materializing them.

`messages thread` asks the relay for reply expansion via the non-standard
`depth_limit` filter field (`messages.rs:326-327`, consumed at
`bridge.rs:1140-1146`) and ORs in a root-by-id filter so the root event is
returned alongside its replies (`messages.rs:328-332`).

#### Pagination, limits and defaults

| Command | Default | Client cap | Site |
|---------|---------|-----------|------|
| `messages get` | 50 | 200 | `messages.rs:273` |
| `messages thread` | 100 | 500 | `messages.rs:314` |
| `messages search` | 20 | 100 | `messages.rs:353` |
| `feed get` | 20 | 50 | `feed.rs:15` |
| `dms list` | 50 | 200 | `dms.rs:10` |
| `notes ls` | 50 | 200 | `notes.rs:672` |
| `social notes` | 50 | 100 | `social.rs:93` |
| `social list` | fixed 10 | — | `social.rs:202` |
| `notes get --name` candidate scan | fixed 50 | — | `notes.rs:191` |
| author-name resolution | fixed 100 | — | `messages.rs:409`, `notes.rs:216` |
| `channels list` | 500 | none | `channels.rs:31` |
| `channels search` | 1000 (clap) | none | `lib.rs:539`, `channels.rs:133` |
| `emoji list`/`export --scope workspace`, `reactions get`/`remove`, `canvas get` | **no `limit` sent** | — | `emoji.rs:78-81`,`:218-221`; `reactions.rs:82-85`,`:44-48`; `channels.rs:264-267` |

Filters with no `limit` fall to the relay's `MAX_HISTORICAL_LIMIT` of 2000
(`req.rs:25`, applied at `req.rs:878-882`). So the workspace emoji palette
silently truncates at 2000 kind-30030 events and a message's reaction list at
2000 kind-7 events, with no indication to the caller.

`--limit` for `channels list` and `channels search` bounds *events fetched*, not
matches returned: filtering happens client-side after pagination
(`channels.rs:66-95` for visibility, `channels.rs:134-142` for name/archived). A
`channels search --query x --limit 10` can therefore return zero rows while
matches exist beyond the first 10 events.

Only `channels list`/`search` and the template roster scan paginate
(`query_paginated`/`query_all`, `channels.rs:42`,`:65`,`:133`,`:449`); the
roster scan is deliberately keyset-paginated because 30177 is
parameterized-replaceable and offset drift could skip a live instance
(`channels.rs:434-439`, doc comment).

`social notes` forwards `--before`→`until` and `--before_id`→`before_id`
(`social.rs:102-108`), the composite cursor the relay requires as both-or-neither
(`bridge.rs:1233-1247`); the CLI does **not** enforce that pairing, so
`--before-id` alone yields a relay 400.

#### Ordering rules

| Command | Ordering | Site |
|---------|----------|------|
| `messages get` / `thread` | ascending `created_at`, client-side | `messages.rs:298`, `:334` |
| `messages search` with `--query` | relay relevance order preserved | `messages.rs:375-381` (explicit comment) |
| `messages search` without `--query` | descending `created_at`, client-side | `messages.rs:377-381` |
| `feed get` | descending `created_at` | `feed.rs:44` |
| `channels search` | by name, then channel_id | `channels.rs:143-147` |
| `emoji list`/`export` | by shortcode (then url for `--scope own`) | `emoji.rs:71`, `:210` |
| `reactions get` | by emoji string | `reactions.rs:116-122` |
| `notes ls` / candidates | descending `updated_at` | `notes.rs:388`, `:281-283` |
| `canvas get`, `channels get`, `dms list` | **none** — takes `events.first()` / relay order as-is | `channels.rs:270`, `:233`, `dms.rs:20` |

`notes.rs` sorts defensively before taking the head of a
parameterized-replaceable coordinate ("the relay keeps only the latest… but be
defensive", `notes.rs:178-181`, `:361-363`); `channels.rs:270` and `emoji.rs:107`
rely on relay ordering instead (`emoji.rs:106` uses `events.last()` with the same
"be defensive" comment but no sort, so with multiple events it takes the *last*
returned, i.e. the oldest under `created_at DESC`).

#### Edit / delete semantics

- `messages edit` publishes kind 40003 with an `e` tag to the target and no
  ownership check (`messages.rs:701-720`); authorization is left to the relay.
- `messages delete` publishes Buzz-native kind 9005, not NIP-09 kind 5
  (`builders.rs:429`), and can attach `action_id`/`reason_code`/`public_reason`
  for moderator tombstones (`messages.rs:685-689`) with no validation of those
  free-text values.
- `notes rm` publishes NIP-09 kind 5 with an **`a` tag only**. The no-`e`-tag
  property is load-bearing: the relay routes to its coordinate soft-delete path
  only when the kind:5 has no `e` targets (`notes.rs:704-711` doc). A test pins
  both halves (`notes.rs:1264-1296`).
- `reactions remove` is a read-then-delete: it queries the caller's own kind-7
  events on the target and deletes the first whose `content` equals the `--emoji`
  string exactly (`reactions.rs:60-73`). Custom-emoji reactions store content as
  `:shortcode:` (`builders.rs:487-491`), so `reactions add --emoji party
  --emoji-url …` followed by `reactions remove --emoji party` fails with "no
  reaction with emoji 'party' found"; the user must pass `:party:`. Nothing in the
  code or `--help` (`lib.rs:717-722`) states this.
- `notes rm` refuses when the caller has no note under the slug
  (`notes.rs:721-726`), giving `NotFound` rather than emitting a pointless kind 5.

#### Reaction dedupe and counting

`reactions add` performs no duplicate check, so the same user can stack identical
reactions (`reactions.rs:9-32`). `reactions get` groups by `content`, defaults
empty content to `"+"` (`reactions.rs:99-104`), and reports
`count = pubkeys.len()` **without deduplicating by pubkey**
(`reactions.rs:107-114`) — a user who reacted twice is counted twice and appears
twice in `pubkeys`. Whether the relay filters deleted reactions out of the kind-7
query is not determinable from this module; I did not verify it.

#### Emoji scope and merge rules

- Each member publishes their own kind:30030 under the fixed d-tag
  `buzz:custom-emoji` (`emoji.rs:9`); the workspace palette is a **client-side
  union** of all members' sets (`emoji.rs:49-75`).
- Union rule: newest `created_at` wins per shortcode; equal timestamps tie-break
  to the lexicographically smallest URL, making the result independent of fetch
  order (`emoji.rs:55-63`, doc `:44-48`). Two tests pin it, including the
  reversed-input invariance check (`emoji.rs:331-372`, `:374-388`).
- `emoji set`/`rm` are read-modify-write over the caller's own set only
  (`emoji.rs:128-139`, `:141-157`); `rm` short-circuits without publishing when
  the shortcode is absent (`emoji.rs:146-154`).
- `emoji import` normalizes every shortcode, dedupes within the batch
  (first occurrence wins, `emoji.rs:276-279`), then either replaces the whole set
  (`--replace`) or merges keeping **existing** entries on conflict
  (`emoji.rs:281-296`). `--dry-run` prints the computed set to stdout and the
  marker to stderr without publishing (`emoji.rs:298-307`).
- Shortcode normalization (trim, strip colons, ≤64 bytes, `[A-Za-z0-9_-]`,
  lowercased) and URL checks (non-empty, ≤2048 bytes, http/https) live in the SDK
  (`builders.rs:127-168`), invoked at `emoji.rs:129`, `:142`, `:269`.

#### Membership, ownership and add-policy rules

- No command checks membership locally. `channels add-member`, `remove-member`,
  `join`, `leave`, `archive`, `delete`, `messages edit`/`delete` all sign and
  submit, relying on relay-side authorization.
- `channels set-add-policy` has a client-side deployment gate: if
  `BUZZ_ACP_ALLOWED_CHANNEL_ADD_POLICIES` is set and non-empty, the requested
  policy must be in the comma-separated list (`channels.rs:1021-1033`). The
  in-code comment is explicit that this covers only the CLI path and a direct
  kind:10100 submission bypasses it (`channels.rs:1015-1020`) — a rule enforced by
  convention, not by the relay.
- Template roster owner invariant (labelled F1): the effective owner is the
  NIP-OA auth-tag owner if present, else the signing pubkey, with **no**
  sole-author fallback so a same-slug 30176/30177 from another principal can never
  be selected (`channels.rs:645-651`, `client.rs:576-584`).
- Template cardinality rule (labelled F4, `channels.rs:472-503`): zero live
  instances for a persona slug is a skip, exactly one is added as
  `MemberRole::Bot` (`channels.rs:750`), more than one is a hard `Usage` error
  listing candidate pubkeys and aborting before any write.
- Archive filtering (NIP-IA) runs before cardinality and **fails open**: if the
  archived-identities snapshot can't be trusted, the filter proceeds with an empty
  archived set and threads a warning into both stderr and the report
  (`channels.rs:511-587`, `:589-603`). Skips are labelled `"all instances
  archived"` vs `"no live instances"` (`channels.rs:568-580`).
- Template writes are best-effort after creation: canvas failure sets
  `canvas_applied:false` (`channels.rs:736-737`), per-member failures land in
  `member_failures` and downgrade `status` to `"partial"`
  (`channels.rs:763-772`). Members are added sequentially because concurrent
  kind:9000 writes are LWW on the relay (`channels.rs:741-743`).

#### NIP-33 LWW conflict handling

Only `notes set` treats a `duplicate:`/`duplicate` relay message as a conflict,
returning `CliError::Conflict` → exit 5 (`notes.rs:556-566`). `notes rm` parses
`accepted` but not `duplicate:` (`notes.rs:734-745`). `emoji set`/`rm`/`import`
publish parameterized-replaceable kind 30030 and never inspect the message, so a
dominated emoji-set write reports success (`emoji.rs:118-124`). The same is true
of `social set-list` for kinds 30000/30003 (`social.rs:179-180`, which prints the
raw response).

#### DM handling

`dms` implements Buzz's channel-style DMs, not NIP-17: `open` creates kind 41010
with `p` tags plus a client-generated `d` UUID (`dms.rs:57-70`), `add-member` is
41011, `hide` is 41012, and `list` reads relay-confirmed 41001
(`dms.rs:11-15`). `dms open` builds the event by hand because
`buzz_sdk::build_dm_open` accepts no `d` tag (`dms.rs:57-59` comment), and then
prefers the relay-assigned `channel_id` from the `response:{…}` message over the
locally generated UUID (`dms.rs:74-84`). Actual DM *messages* are sent with
`messages send --channel <dm-uuid>` as plaintext kind 9 — `grep -rn
'nip44\|nip04\|encrypt\|gift_wrap' crates/buzz-cli/src` matches only
`agent_management.rs` (observer frames), so nothing in this module encrypts DM
content. Kind 1059 gift wrap (`kind.rs:60`) is never used here.

#### Note carry-forward semantics

`build_set_event` (`notes.rs:418-469`) implements a ratified omit-vs-clear matrix:
`None` carries the prior value (usage error for `title` on first publish, since
NIP-23 requires it), `Some("")` clears, `Some(v)` sets; `tags: None` carries,
`Some(&[])` clears, `Some(slice)` replaces wholesale (not merged);
`published_at` is preserved from the prior event or stamped with `now` on first
publish. Ten tests cover the matrix (`notes.rs:1057-1229`), including a malformed
prior `published_at` re-stamping to `now` (`notes.rs:1231-1240`).

Slug rules: 1–80 chars, `[a-z0-9._-]` only, lowercase enforced to avoid
`Dco-Recipe` vs `dco-recipe` ambiguity (`notes.rs:50-70`, six tests at
`notes.rs:826-857`).

`notes get --name` ambiguity: >1 author for a slug is an error listing candidates
unless `--latest` picks the newest (`notes.rs:638-652`). The candidate scan is
capped at 50 events (`notes.rs:191`), so ambiguity detection is itself bounded.

#### Author resolution — two incompatible implementations

| Behaviour | `messages.rs:394-439` | `notes.rs:204-252` |
|-----------|----------------------|--------------------|
| `"me"` accepted | no | yes (`notes.rs:205-207`) |
| 64-hex accepted | yes, lowercased (`:396-398`) | yes (`:208-211`) |
| `npub1…` accepted | yes (`:399-404`) | **no** |
| name search | kind 0 + NIP-50 `search`, limit 100 | identical filter (`:213-217`) |
| match rule | exact case-insensitive on `display_name` or `name` (`:441-467`) | same rule, re-implemented inline (`:220-235`) |
| ambiguity error | lists up to 5 candidates + "… and N more" (`:425-437`) | reports only the count (`:248-250`) |
| dedupe of duplicate kind:0 rows | yes (`:465`) | no |

So `buzz messages search --author npub1…` works while `buzz notes ls --author
npub1…` is treated as a display name and almost certainly fails. `notes ls` also
gives `--author all` a special meaning (skip the author filter,
`notes.rs:679-683`) that `messages search` has no equivalent for.

#### Mention resolution rules

`resolve_content_mentions` (`messages.rs:128-201`) only runs when the body
contains `@` (`:133-135`), resolves against the channel's kind-39002 member list
plus those members' kind-0 profiles, matches multi-word display names first then
single words (`extract_at_mentions_with_known`, `:191-193`), and returns an empty
vec on **any** I/O or parse failure so auto-tagging never blocks a send
(`:124-127` doc, `:144-147`, `:155-158`). Consequences: `@name` for a
non-member is silently unresolved, and a relay hiccup silently drops all
mentions from an otherwise successful message.

NIP-27 `nostr:npub1…` URIs are extracted separately after stripping code regions
and merged under `MENTION_CAP` (`messages.rs:526-531`). Mentions are computed
from the author-written body only, never the appended media markdown
(`messages.rs:523-525` comment). The SDK dedupes and enforces the cap
(`builders.rs:186-198`).

#### Rules enforced only by comment or convention

| Rule | Where stated | Enforcement |
|------|-------------|-------------|
| add-policy allowlist covers only this CLI path | `channels.rs:1015-1020` | none relay-side, by team decision |
| callers must not add `auth` tags manually | `client.rs:585-587` | actually enforced post-signing (`client.rs:598-610`) |
| template members added sequentially to avoid LWW races | `channels.rs:741-743` | code shape only |
| kind:5 for notes must carry no `e` tag | `notes.rs:704-711` | one unit test (`notes.rs:1288-1295`) |
| `emoji` 30030 relay keeps only latest per `(pubkey,d)` | `emoji.rs:105` | relies on relay; client takes `events.last()` unsorted |
| `--content -` is the escape hatch for shell-metacharacter-heavy text | `messages.rs:484-487` | convention |

#### Test coverage — business rules

Covered: TTL validation (3 tests, `channels.rs:1276-1290`), add-policy env gate
(4 tests, `channels.rs:1294-1382`, one through the real
`cmd_set_add_policy`), roster cardinality (6 tests, `channels.rs:1386-1466`),
archive filter + fail-open (6 tests, `channels.rs:1470-1597`), warning wiring at
both observable boundaries (3 tests, `channels.rs:1607-1683`), NIP-10 root
derivation (8 tests, `messages.rs:781-861`), author name matching (4 tests,
`messages.rs:1112-1163`), social list kind validation and d-tag requirement
(5 tests, `social.rs:239-283`), `validate_get_args` matrix (4 tests,
`notes.rs:1298-1330`), note carry-forward (10 tests, `notes.rs:1057-1229`), emoji
union LWW (2 tests, `emoji.rs:330-388`), template lookup and case-insensitivity
(5 tests, `channel_templates.rs:137-197`).

Uncovered business rules, each verified by grepping every `#[cfg(test)]` block in
the ten files for the function name with zero hits: the message-kind gate
(`messages.rs:540-572`), the 45001-drops-`--reply-to` behaviour, the branch-pair
rule (`messages.rs:601-608`), `--kinds` parse fallback (`messages.rs:283-287`),
role mapping (`channels.rs:964-976`), channel type/visibility gates
(`channels.rs:283-298`), the visibility post-filter (`channels.rs:66-95`), feed
type validation (`feed.rs:31-39`), the DM 1–8 bound (`dms.rs:52-54`), reaction
emoji matching (`reactions.rs:60-73`), reaction grouping/counting
(`reactions.rs:96-124`), emoji merge-vs-replace (`emoji.rs:281-296`), and the
LWW `duplicate:` handling in `cmd_set` (`notes.rs:556-566`).

One test re-implements a production rule rather than calling it:
`check_allowed_channel_add_policy` (`channels.rs:1296-1307`) is a verbatim copy of
the parsing/matching logic at `channels.rs:1022-1033`, so the three tests using it
(`channels.rs:1310-1343`) would stay green if the production gate changed. The
file acknowledges the gap and adds `set_add_policy_env_gate_rejects_disallowed_via_full_path`
(`channels.rs:1362-1382`) to cover the real path. A second, benign instance:
`cli_pipeline_resolves_multiword_display_names` (`messages.rs:1017-1061`)
re-implements the profile-parsing loop from `resolve_content_mentions` inline
("Simulate the single-parse pipeline", `messages.rs:1026`), so a change to the
real loop would not fail it.
