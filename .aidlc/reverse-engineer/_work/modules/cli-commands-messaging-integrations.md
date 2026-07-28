## Module: buzz-cli — channel, message & social commands (`crates/buzz-cli/src/commands`)
### Aspect: Integrations

#### Relay HTTP surface used

These files never touch a WebSocket and never construct a URL. All I/O goes
through `BuzzClient`, which owns the three endpoints below.

| Endpoint | Client method | Used by | Sites |
|----------|--------------|---------|-------|
| `POST /query` (single filter) | `query` (`client.rs:767`) | every read except the four below | e.g. `channels.rs:231`, `messages.rs:296`, `notes.rs:176`, `emoji.rs:82`, `dms.rs:16`, `feed.rs:41`, `reactions.rs:86`, `social.rs:77` |
| `POST /query` (ORed filters) | `query_multi` (`client.rs:773`) | `messages thread` only | `messages.rs:332` |
| `POST /query` (keyset-paginated) | `query_paginated` (`client.rs:715`) | `channels list`, `channels search` | `channels.rs:42`, `:65`, `:133` |
| `POST /query` (paginate to exhaustion) | `query_all` (`client.rs:724`) | template roster scan | `channels.rs:449` |
| `POST /events` | `submit_event` (`client.rs:863`) | every write | e.g. `messages.rs:576`, `channels.rs:326`, `notes.rs:545`, `emoji.rs:122`, `dms.rs:72`, `reactions.rs:29`, `social.rs:38` |
| Blossom media upload | `upload_file` (`client.rs:1100`) | `messages send --file` | `messages.rs:510` |
| `GET /` (NIP-11 info) | `get_public` (`client.rs:753`), reached via `agents::fetch_archived_snapshot` | `channels create --template` archive filter | `channels.rs:632`, implementation `agents.rs:272-282` |

`POST /count` (`client.rs:803`) and `get_authed` (`client.rs:836`) are never
called from these files. Every request carries a NIP-98 `Authorization` event and,
when configured, the `x-auth-tag` header (`client.rs:120-127`).

Relay-side extension fields the CLI injects into otherwise-standard Nostr filters
(all three are Buzz extensions, parsed out of the raw JSON before
`nostr::Filter` drops them — `bridge.rs:968-971`):

| Field | Set by | Consumed at |
|-------|--------|-------------|
| `before_id` | `social notes --before-id` (`social.rs:106-108`), and by `query_paginated`'s cursor (`client.rs:500-520`) | `bridge.rs:263-281`, `:1233-1247` |
| `depth_limit` | `messages thread --depth-limit` (`messages.rs:326-327`) | `bridge.rs:283-291`, `:1140-1146` |
| `feed_types` | `feed get --types` (`feed.rs:38`) | `bridge.rs:324-330`, `:1054-1111` |

The `feed_types` coupling is load-bearing and undocumented on the CLI side: if
`--types` is omitted the field is absent, `extract_feed_types` returns `None`, and
the relay `continue`s past its feed branch (`bridge.rs:1054-1057`), so the query
degrades to a plain "events p-tagging me" scan. `feed.rs:6`'s
`VALID_FEED_TYPES = ["mentions","needs_action","activity","agent_activity"]`
duplicates the relay's accepted set (`bridge.rs:1069-1111`, where
`agent_activity` canonicalizes to `activity`); the two lists agree today but
nothing keeps them in sync.

#### NIPs implemented or consumed

| NIP | Where | Sites |
|-----|-------|-------|
| NIP-01 (kinds 0/1/3, filters) | `social` reads/writes, profile lookups | `social.rs:34`, `:97`, `:118`; `messages.rs:407`; `notes.rs:214` |
| NIP-02 contact list | `social set-contacts` | `social.rs:63`, `builders.rs:764-810` |
| NIP-09 deletion | `reactions remove` (kind 5 on the reaction), `notes rm` (kind 5 with `a` tag) | `reactions.rs:71`; `notes.rs:712-715` |
| NIP-10 threading | root/reply marker parsing and emission | `messages.rs:25-56`, `builders.rs:171-181` |
| NIP-11 relay info | archive snapshot trust check | `agents.rs:272-282` via `channels.rs:632` |
| NIP-19 bech32 `naddr` / `npub` | note coordinates, `--author npub…` | `notes.rs:317-320`, `:255`; `messages.rs:400-403` |
| NIP-21 `nostr:` URI | accepted by `parse_naddr` | `notes.rs:249` (doc), `:255` |
| NIP-23 long-form | whole `notes` group | `notes.rs:1-31`, `:418-469` |
| NIP-25 reactions | `reactions add` | `reactions.rs:20-26`, `builders.rs:463-492` |
| NIP-27 inline mentions | `extract_nostr_uris` on message bodies | `messages.rs:527-529` |
| NIP-29 groups | whole `channels` group (`h` tags, kinds 9000-9008/9021/9022, 39000/39002) | `channels.rs` throughout; `builders.rs:563-730` |
| NIP-30 custom emoji | `emoji` group, custom emoji reactions | `emoji.rs:9`, `builders.rs:127-168`, `:479-492` |
| NIP-31 `alt` tag | `messages send-diff` | `messages.rs:620-626`, `builders.rs:371-373` |
| NIP-33 parameterized replaceable | 30023 / 30030 / 39000 / 39002 / 30176 / 30177 addressing, and the LWW conflict check | `notes.rs:271-274`, `emoji.rs:9`, `channels.rs:434-439`, `notes.rs:556-566` |
| NIP-50 search | `messages search --query`, both author-name resolvers | `messages.rs:365`, `:408`; `notes.rs:215` |
| NIP-51 lists/sets | `social set-list` / `social list` | `social.rs:127-139` |
| NIP-65 relay list metadata | kind 10002 accepted by the same pair | `social.rs:130` |
| NIP-92 `imeta` | media attachment tags | `client.rs:39-40` → `messages.rs:511` |
| NIP-98 HTTP auth | every request (transport) | `client.rs:84`, invoked in `query_multi`/`submit_event` |
| NIP-OA owner attestation | auth-tag injection + template owner derivation | `client.rs:588-596`; `channels.rs:645-651` |
| NIP-IA identity archive | template roster archive filter (kind 13535 snapshot, states 1/2/3) | `channels.rs:511-587`, `agents.rs:270-305` |
| NIP-CW cursor grammar | `until`+`before_id` composite cursor | `social.rs:102-108` (partially — see below), `client.rs:500-520` |
| NIP-AE engrams, NIP-AB pairing, NIP-17 gift wrap, NIP-34 git | **not** used here | `mem.rs` / `buzz-pairing-cli` / unused / `repos.rs`,`patches.rs`,`pr.rs`,`issues.rs` |

`social notes` exposes `--before` and `--before-id` independently
(`lib.rs:1000-1006`) but the relay requires both or neither
(`bridge.rs:1240-1246`); the CLI does not enforce the pairing, so
`--before-id` alone is a relay 400 rather than a local `Usage` error.

#### Crate dependencies used from these files

| Crate / module | What is used | Sites |
|----------------|-------------|-------|
| `buzz_sdk` (builders) | 24 `build_*` functions | `channels.rs:308,856,871,887,900,912,924,936,948,978,996,1058,747,729`; `messages.rs:542,549,558,659,682,714,745`; `reactions.rs:20,24,71`; `emoji.rs:119`; `dms.rs:118`; `social.rs:33,62`; `notes.rs` uses its own builder instead |
| `buzz_sdk::kind` | `KIND_DM_HIDE`, `KIND_AGENT_PROFILE`, `KIND_EMOJI_SET`, six social list kinds | `dms.rs:103`; `channels.rs:1040`; `emoji.rs:79`; `social.rs:1-4` |
| `buzz_sdk::mentions` | `extract_at_mentions_with_known`, `extract_nostr_uris`, `merge_mentions`, `strip_code_regions`, `MENTION_CAP` | `messages.rs:11-14`, used `:192`, `:527-531` |
| `buzz_sdk` types | `ThreadRef`, `DiffMeta`, `VoteDirection`, `DeleteMessageOptions`, `CustomEmoji`, `Visibility`, `ChannelKind`, `MemberRole` | `messages.rs:1`, `emoji.rs:6`, `channels.rs:301-320`, `:750` |
| `buzz_sdk::CUSTOM_EMOJI_SET_D_TAG` | the `buzz:custom-emoji` d-tag | `emoji.rs:9` |
| `buzz_core::kind` | `KIND_MANAGED_AGENT`, `KIND_TEAM` | `channels.rs:3` |
| `nostr` | `EventBuilder`, `Kind`, `Tag`, `EventId`, `PublicKey`, `Event`, `Timestamp`, `ToBech32`, `Coordinate` | `dms.rs:57`, `social.rs:5`, `channels.rs:1036`, `reactions.rs:3`, `notes.rs:36`, `messages.rs:2` |
| `uuid` | channel/DM UUID generation and parsing | `channels.rs:5`,`:294`,`:699`; `dms.rs:1`,`:60`; `messages.rs:3` |
| `serde` / `serde_json` | every filter, every output | throughout |
| `dirs` | platform app-data dir for the template store | `channel_templates.rs:73` |
| `tempfile` (dev) | template store fixtures | `channel_templates.rs:132` |
| `tokio` (dev) | the single async test | `channels.rs:1362` |

#### Intra-crate coupling

| Import | From | Purpose |
|--------|------|---------|
| `client::{normalize_events, normalize_write_response, print_create_response, extract_d_tag, extract_p_tags, extract_tag_value, build_imeta_tag, BuzzClient}` | `client.rs` | transport + output normalization |
| `validate::{parse_uuid, validate_uuid, validate_hex64, validate_content_size, parse_event_id, read_or_stdin, truncate_diff, infer_language, sdk_err, MAX_DIFF_BYTES}` | `validate.rs` | input validation |
| `error::CliError` | `error.rs` | error taxonomy |
| `crate::{OutputFormat, ChannelsCmd, MessagesCmd, …, EmojiScope}` | `lib.rs` | clap enums |
| `commands::agents::fetch_archived_snapshot` | `agents.rs:270` | **cross-command dependency**: `channels create --template` reaches into the agents module for the NIP-IA snapshot (`channels.rs:11`, called `channels.rs:632`) |
| `commands::channel_templates` | in-scope sibling | template store loading (`channels.rs:12`) |

`channels.rs` → `agents.rs` is the only cross-command-module call in scope. It
couples channel creation to the agent-management module's trust-check semantics
(states 1/2/3), which `channels.rs` then re-interprets as fail-open
(`channels.rs:511-524`).

#### Local filesystem integration

| Path | Direction | Site |
|------|-----------|------|
| `<platform-data-dir>/xyz.block.buzz.app/templates/channel-templates.json` | read | `channel_templates.rs:18`, resolved `:71-84`, read `:95-96` |
| any path from `--templates-file` | read | `channel_templates.rs:73-75` |
| any path from `emoji import --file` | read | `emoji.rs:166-167` |
| any path from `emoji export --file` | **write (overwrite)** | `emoji.rs:187-188` |
| stdin | read | `messages send --content -` and `send-diff --diff -` (`validate.rs:186-197`), `canvas set --content -` (`channels.rs:1055`), `notes set --content -` (`notes.rs:490-508`), `emoji import` default (`emoji.rs:169-183`) |
| stdout | write | every command |
| stderr | write | `emoji import --dry-run` marker (`emoji.rs:303`), template archive warning (`channels.rs:597`) |

The template path is derived from `dirs::data_dir()` joined with a hardcoded Tauri
bundle identifier (`channel_templates.rs:15-18`), i.e. the CLI reverse-engineers
the desktop app's storage location. The comment states this matches
`app.path().app_data_dir()` exactly; that equivalence is asserted only by a
suffix check in a test (`channel_templates.rs:143-147`) and cannot be verified
from this crate.

#### Logic duplicated across these files rather than shared

| Duplicated logic | Copies |
|------------------|--------|
| Compact-format event projection `{id,content,created_at}` | `messages.rs:249-257` and `feed.rs:50-59` (byte-for-byte equivalent); `channels.rs:98-106` is a third, channel-specific copy |
| Author-name resolution against kind:0 with NIP-50 | `messages.rs:394-467` and `notes.rs:204-252` — different accepted inputs and different ambiguity reporting (see Business Rules) |
| Newest-first sort by `created_at` | `messages.rs:377-381`, `feed.rs:44`, `notes.rs:388`, `notes.rs:178-181`, `notes.rs:361-363` |
| Read-modify-write on a NIP-33 own-set | `emoji.rs:128-157` (30030) and `notes.rs:528-534` (30023) |
| `accepted` / `message` extraction from a write response | `notes.rs:546-555` and `notes.rs:734-744` (twice in one file); `dms.rs:74-84` extracts `message` differently again; `client.rs:1407-1418` already provides `extract_relay_response_field` and is used by neither |
| DM pubkey count + hex validation | `dms.rs:52-56` re-implements `builders.rs:1544-1553` because the SDK builder lacks a `d`-tag parameter |
| Channel type/visibility string→enum mapping | `channels.rs:300-320` and `channels.rs:699-709` |
| Env-gate parsing for add policy | production `channels.rs:1022-1033`, test copy `channels.rs:1296-1307` |
| Profile-JSON parsing loop (`display_name` else `name`) | `messages.rs:167-190`, `messages.rs:451-459`, `notes.rs:222-234` — three implementations of the same rule |

#### Test coverage — integrations

No test in scope exercises a relay call: every `#[test]` is a pure-function test,
and the one `#[tokio::test]` (`channels.rs:1362`) constructs a `BuzzClient`
pointed at `ws://localhost:3000` (`channels.rs:1352-1358`) precisely because the
gate returns before any network I/O (`channels.rs:1345-1350` comment). Relay
contract coverage for these kinds lives outside this module in
`crates/buzz-test-client/tests/` (e.g. `e2e_long_form.rs` for kind 30023); there
is no e2e test that drives the `buzz` binary itself, and no
`crates/buzz-cli/tests/` directory exists.

The filesystem integration is the best-covered: `channel_templates.rs` has six
tests including override precedence, prod-path suffix, case-insensitive lookup,
missing-store `NotFound`, and full roster parsing
(`channel_templates.rs:137-197`). `emoji.rs`'s `read_source` / `write_output`
have no tests (`grep -n 'read_source\|write_output' emoji.rs` shows only
definitions at `:164`/`:185` and call sites at `:241`/`:230`).
