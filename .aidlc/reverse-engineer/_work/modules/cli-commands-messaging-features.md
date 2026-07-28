## Module: buzz-cli — channel, message & social commands (`crates/buzz-cli/src/commands`)
### Aspect: Features

This module is the agent-facing surface for everything a Buzz participant does in
a community: talk in channels, thread, react, run forums, manage channel
lifecycle and membership, keep a long-form knowledge base, curate custom emoji,
open DMs, and read a personal activity feed. 45 subcommands across nine groups.

#### Messaging

| Capability | Command | Limits |
|-----------|---------|--------|
| Send a channel message | `messages send --channel --content` (`messages.rs:483`) | body ≤ 64 KiB (`validate.rs:5`, checked `messages.rs:492`); `-` reads stdin unbounded before the check (`validate.rs:186-197`) |
| Send from stdin to dodge shell quoting | `--content -` (`messages.rs:488`, rationale `:484-487`) | no byte cap on the read itself |
| Reply / thread | `--reply-to <event-id>` (`messages.rs:534-538`) | resolves the parent's root via one relay fetch; **ignored** for `--kind 45001` |
| Auto-`@mention` | any `@name` in the body (`messages.rs:521`) | members of that channel only; capped at `MENTION_CAP`; silently empty on any lookup failure |
| NIP-27 inline references | `nostr:npub1…` in the body (`messages.rs:527-529`) | code regions stripped first |
| Broadcast to public Nostr | `--broadcast` (`lib.rs:369-371`) | adds `["broadcast","1"]` (`builders.rs:233`) |
| Attach media | `--file` (repeatable, `lib.rs:373-375`) | each upload is a separate Blossom call (`messages.rs:509-521`); markdown link appended per file, `![video]` vs `![image]` by MIME prefix (`messages.rs:513-517`) |
| Forum post / comment | `--kind 45001` / `45003` (`messages.rs:541-557`) | 45003 requires `--reply-to` |
| Post a diff with metadata | `messages send-diff` (`messages.rs:596`) | diff truncated at a hunk boundary at 60 KiB (`validate.rs:6`, `messages.rs:614`); language inferred from the file extension across 25 extension groups (`validate.rs:126-158`); NIP-31 `alt` text auto-composed (`messages.rs:621-626`) |
| Edit your message | `messages edit` (`messages.rs:701`) | channel re-derived from the event's `h` tag |
| Delete a message, optionally with public tombstone metadata | `messages delete [--action-id --reason-code --public-reason]` (`messages.rs:669`) | kind 9005; metadata free-text, unvalidated |
| Read a channel | `messages get [--limit --before --since --kinds]` (`messages.rs:263`) | default 50, cap 200, ascending time |
| Read a thread | `messages thread [--limit --depth-limit]` (`messages.rs:304`) | default 100, cap 500; root event included |
| Full-text / author search | `messages search [--query --author --since --limit]` (`messages.rs:340`) | community-wide only, no channel scoping; `--author` accepts hex, npub or a display name |
| Vote on forum content | `messages vote --direction up\|down` (`messages.rs:724`) | kind 45002, content `+`/`-` |

#### Channels

| Capability | Command | Notes |
|-----------|---------|-------|
| List channels | `channels list [--visibility --member --limit]` (`channels.rs:25`) | `--member` costs two queries; visibility filtering is client-side after fetch |
| Search by name | `channels search --query [--exact --include-archived --limit]` (`channels.rs:119`) | substring or exact, case-insensitive; archived excluded by default |
| Inspect one channel | `channels get` (`channels.rs:224`) | prints `null` when absent |
| Create | `channels create --name --type --visibility [--description --ttl]` (`channels.rs:282`) | `--ttl` seconds, positive and ≤ `i32::MAX` (`channels.rs:822-830`); relay archives on idle |
| Create from a desktop template | `channels create --name --template [--templates-file …]` (`channels.rs:655`) | applies type/visibility/description/canvas, resolves and adds the agent roster |
| Update metadata | `channels update [--name --description --ttl\|--no-ttl]` (`channels.rs:832`) | `--no-ttl` makes the channel permanent |
| Topic / purpose | `channels topic`, `channels purpose` (`channels.rs:864`, `:880`) | free text, no length cap client-side |
| Lifecycle | `join`, `leave`, `archive`, `unarchive`, `delete` (`channels.rs:896`–`:952`) | all single kind-90xx writes |
| Membership | `members`, `add-member [--role]`, `remove-member` (`channels.rs:244`, `:956`, `:987`) | roles owner/admin/member/guest/bot |
| Personal add policy | `channels set-add-policy --policy` (`channels.rs:1005`) | who may add you to channels; optionally restricted per deployment by env var |
| Channel canvas | `canvas get`, `canvas set --content [-]` (`channels.rs:262`, `:1049`) | markdown document per channel; `get` prints the raw body, not JSON |

The template flow is the richest single feature here. From one command it: loads a
JSON store the desktop app owns (`channel_templates.rs:60-84`), expands team
entries into persona slugs via kind 30176 (`channels.rs:399-431`), scans the
owner's kind-30177 managed agents (`channels.rs:440-464`), filters archived
identities (`channels.rs:526-587`), enforces one live instance per persona,
creates the channel, applies the canvas with `{channel.name}` /
`{template.name}` substitution (`channels.rs:725-727`), adds each resolved agent
as a `bot` member, and prints a structured report distinguishing
`members_added` / `skipped` (with reasons) / `archived_excluded` /
`member_failures` / `archive_state_warning` (`channels.rs:795-820`).

Limits worth knowing: the store must already exist on this machine — there is no
`channels template create`, and a missing store is `NotFound` with a hint to use
Buzz Desktop or `--templates-file` (`channel_templates.rs:88-93`). Cold-start
provisioning of a persona that has no live instance is explicitly out of scope
(`channels.rs:353-355`, `:472-476`), so those slugs are skipped. The channel is
created even if canvas and every member-add fail; only the roster stage can abort
before any write.

#### Notes (NIP-23 knowledge base)

| Capability | Command | Limits |
|-----------|---------|--------|
| Idempotent upsert keyed by `(you, slug)` | `notes set --name --content -` (`notes.rs:487`) | stdin capped at 1 MiB (`notes.rs:485`, checked `:496-500`); empty stdin refused without `--allow-empty` (`:501-507`) |
| Preserve publication date across edits | automatic (`notes.rs:453`) | `published_at` stamped once |
| Carry or clear title/summary/tags | `--title`/`--summary` omitted vs `""`; `--tag` vs `--clear-tags` (`notes.rs:420-450`) | `--tag` replaces, never merges |
| Read by coordinate | `notes get --naddr <naddr1…\|30023:pk:slug>` (`notes.rs:614-627`) | kind must be 30023 |
| Read by slug across authors | `notes get --name [--author\|--latest]` (`notes.rs:628-656`) | candidate scan capped at 50 |
| Body-only output for piping | `--content-only` (`notes.rs:659-665`) | appends a newline if missing |
| List | `notes ls [--author me\|all\|<ref> --tag --limit]` (`notes.rs:671`) | default own notes, 50, cap 200 |
| Delete your own note | `notes rm --name` (`notes.rs:717`) | a-tag-only kind 5; `NotFound` if you never published it |

`notes set` prints paste-ready durable references (`event_id`, `naddr`,
`coordinate`, `slug`, `title`) as plain text (`notes.rs:571-580`), and surfaces
NIP-33 LWW domination as exit 5 rather than a false success (`notes.rs:556-566`).

#### Reactions and custom emoji

| Capability | Command | Limits |
|-----------|---------|--------|
| React with a unicode emoji | `reactions add --event --emoji` (`reactions.rs:9`) | ≤64 chars (`builders.rs:466-468`); duplicates not prevented |
| React with a custom emoji | `reactions add --emoji <shortcode> --emoji-url <url>` (`reactions.rs:20-22`) | content becomes `:shortcode:` |
| Un-react | `reactions remove --event --emoji` (`reactions.rs:34`) | matches `content` exactly, so custom emoji need `:shortcode:` |
| Tally reactions | `reactions get --event` (`reactions.rs:80`) | counts are per-event, not per-user-deduped; no `limit` sent |
| See the workspace palette | `emoji list` (`emoji.rs:77`) | union of every member's set |
| Curate your own set | `emoji set --shortcode --url`, `emoji rm --shortcode` (`emoji.rs:128`, `:141`) | read-modify-write; shortcode `[A-Za-z0-9_-]` ≤64 bytes, url http(s) ≤2048 bytes |
| Back up / migrate | `emoji export [--file --scope own\|workspace]` (`emoji.rs:197`) | stable sort so `export \| import --replace` round-trips |
| Bulk load | `emoji import [--file --replace --dry-run]` (`emoji.rs:234`) | stdin capped at 10 MB (`emoji.rs:160`); merge keeps existing on conflict |

#### DMs, feed, and the social graph

| Capability | Command | Limits |
|-----------|---------|--------|
| Open a 1:1 or small-group DM | `dms open --pubkey … (1–8)` (`dms.rs:51`) | returns the relay-assigned `dm_id`; messages are then sent with `messages send --channel <dm_id>` in **plaintext** |
| Grow a DM group | `dms add-member` (`dms.rs:112`) | kind 41011 |
| Hide a conversation | `dms hide` (`dms.rs:96`) | kind 41012 |
| List conversations | `dms list [--limit]` (`dms.rs:8`) | default 50, cap 200; returns `dm_id` + participants only, no last-message preview |
| Personal activity feed | `feed get [--since --limit --types]` (`feed.rs:9`) | default 20, cap 50; **without `--types` this is not a feed query at all** — the relay's feed path only engages when `feed_types` is present (`bridge.rs:1054-1057`), so a bare `feed get` degrades to "any event p-tagging me" |
| Publish a public note | `social publish --content [--reply-to]` (`social.rs:22`) | kind 1, flat reply model only (`builders.rs:733-737`) |
| Replace your follow list | `social set-contacts --contacts <json>` (`social.rs:43`) | whole-list replace, ≤10 000 contacts (`builders.rs:750`) |
| Fetch any event / a user's notes / a user's contacts | `social event`, `social notes`, `social contacts` (`social.rs:72`, `:83`, `:115`) | raw relay JSON passthrough, signatures included |
| Publish and read NIP-51/NIP-65 lists | `social set-list --kind --tags`, `social list --kind [--d-tag]` (`social.rs:162`, `:184`) | six kinds only; parameterized kinds require a `d` tag |

#### Cross-cutting features

- **Agent-friendly output**: every read path except `canvas get`, `notes
  get --content-only` and the `notes set`/`rm` receipts emits one line of JSON on
  stdout; errors are JSON on stderr with a `retryable` hint
  (`error.rs:113-140`).
- **`--format compact`** reduces events to `{id,content,created_at}` for cheaper
  agent scanning, but only for `messages get`/`thread`/`search`,
  `channels list` and `feed get` (`messages.rs:242-261`, `channels.rs:96-109`,
  `feed.rs:47-60`).
- **Stdin escape hatch** (`-`) exists for `messages send --content`, `messages
  send-diff --diff`, `canvas set --content`, `notes set --content`, and
  `emoji import` (implicitly, when `--file` is omitted) — but with four different
  size policies: none, none, none, 1 MiB, 10 MB.
- **NIP-OA delegation** is transparent: the auth tag is injected into every event
  and sent as a header without any per-command handling (`client.rs:588-596`,
  `:120-127`); template resolution is the only feature that reads the owner out of
  it (`channels.rs:645-651`).

#### Capabilities notably absent

Verified by grepping the ten files for the relevant identifiers with zero
matches: no pin/bookmark of a message (kinds 40004/40005 exist at `kind.rs:425`,
`:427`), no scheduled message (40006), no typing indicator or presence from this
module (20001/20002 live in `users.rs`), no read-state write (30078), no huddle
commands (48100-48106), no channel invite (9009), no NIP-17 encrypted DM (1059),
no watch/tail/subscribe mode (nothing here opens a WebSocket), no `--channel`
scoping for `messages search`, no delete-my-own-note-by-naddr (only `--name`),
and no local template authoring.
