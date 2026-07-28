## Module: buzz-cli — channel, message & social commands (`crates/buzz-cli/src/commands`)
### Aspect: Data Model

This module owns no persistent storage. Its data model is (a) the Nostr event
kinds it reads and writes, (b) the tag vocabularies it emits and parses, (c) the
filter documents it POSTs to `/query`, (d) the JSON shapes it prints on stdout,
and (e) one local file format (`channel-templates.json`). All ten files in scope
are `serde_json::Value`-first: only `notes.rs` deserializes into typed
`nostr::Event` (`notes.rs:156`, `parse_events`).

#### Event kinds written

| Kind | `buzz-core/src/kind.rs` constant | Command | Builder call site |
|------|----------------------------------|---------|-------------------|
| 9 | `KIND_STREAM_MESSAGE` (`kind.rs:419`) | `messages send` (default) | `messages.rs:558` `build_message` |
| 45001 | `KIND_FORUM_POST` (`kind.rs:490`) | `messages send --kind 45001` | `messages.rs:542` `build_forum_post` |
| 45003 | `KIND_FORUM_COMMENT` (`kind.rs:494`) | `messages send --kind 45003` | `messages.rs:549` `build_forum_comment` |
| 40008 | `KIND_STREAM_MESSAGE_DIFF` (`kind.rs:433`) | `messages send-diff` | `messages.rs:659` `build_diff_message` |
| 40003 | `KIND_STREAM_MESSAGE_EDIT` (`kind.rs:423`) | `messages edit` | `messages.rs:714` `build_edit` |
| 9005 | `KIND_NIP29_DELETE_EVENT` (`kind.rs:281`) | `messages delete` | `messages.rs:682` `build_delete_message_with_options` |
| 45002 | `KIND_FORUM_VOTE` (`kind.rs:492`) | `messages vote` | `messages.rs:745` `build_vote` |
| 9007 | `KIND_NIP29_CREATE_GROUP` (`kind.rs:283`) | `channels create` (both paths) | `channels.rs:308`, `channels.rs:711` |
| 9002 | `KIND_NIP29_EDIT_METADATA` (`kind.rs:279`) | `channels update` / `topic` / `purpose` / `archive` / `unarchive` | `channels.rs:856`, `:871`, `:887`, `:924`, `:936` |
| 9000 | `KIND_NIP29_PUT_USER` (`kind.rs:275`) | `channels add-member`, template roster add | `channels.rs:978`, `channels.rs:747` |
| 9001 | `KIND_NIP29_REMOVE_USER` (`kind.rs:277`) | `channels remove-member` | `channels.rs:996` |
| 9021 | `KIND_NIP29_JOIN_REQUEST` (`kind.rs:289`) | `channels join` | `channels.rs:900` |
| 9022 | `KIND_NIP29_LEAVE_REQUEST` (`kind.rs:291`) | `channels leave` | `channels.rs:912` |
| 9008 | `KIND_NIP29_DELETE_GROUP` (`kind.rs:285`) | `channels delete` | `channels.rs:948` |
| 10100 | `KIND_AGENT_PROFILE` (`kind.rs:87`) | `channels set-add-policy` | `channels.rs:1038-1042` (hand-built `EventBuilder`) |
| 40100 | `KIND_CANVAS` (`kind.rs:435`) | `canvas set`, template canvas | `channels.rs:1058`, `channels.rs:729` |
| 7 | `KIND_REACTION` (`kind.rs:58`) | `reactions add` | `reactions.rs:20` / `:24` |
| 5 | `KIND_DELETION` (`kind.rs:56`) | `reactions remove`, `notes rm` | `reactions.rs:71`, `notes.rs:713` |
| 30030 | `KIND_EMOJI_SET` (`kind.rs:52`) | `emoji set` / `rm` / `import` | `emoji.rs:119` `build_custom_emoji_set` |
| 41010 | `KIND_DM_OPEN` (`kind.rs:447`) | `dms open` | `dms.rs:69` (hand-built, **not** `build_dm_open`) |
| 41011 | `KIND_DM_ADD_MEMBER` (`kind.rs:449`) | `dms add-member` | `dms.rs:120` |
| 41012 | `KIND_DM_HIDE` (`kind.rs:451`) | `dms hide` | `dms.rs:103` (via `buzz_sdk::kind::KIND_DM_HIDE`) |
| 1 | `KIND_TEXT_NOTE` (`kind.rs:11`) | `social publish` | `social.rs:34` `build_note` |
| 3 | `KIND_CONTACT_LIST` (`kind.rs:13`) | `social set-contacts` | `social.rs:63` `build_contact_list` |
| 10000/10001/10002/10003/30000/30003 | `KIND_MUTE_LIST` … `KIND_BOOKMARK_SET` (`kind.rs:17,22,27,32,39,43`) | `social set-list` | `social.rs:176` (hand-built `EventBuilder`) |
| 30023 | `KIND_LONG_FORM` (`kind.rs:66`) | `notes set` | `notes.rs:466-469` `build_set_event` |

#### Event kinds read (query filters only)

| Kind | Read by | Site |
|------|---------|------|
| 0 (`KIND_PROFILE`, `kind.rs:9`) | mention resolution, `--author` name resolution | `messages.rs:150`, `messages.rs:407`, `notes.rs:214` |
| 1 | `social notes` | `social.rs:97` |
| 3 | `social contacts` | `social.rs:118` |
| 7 | `reactions get` / `remove` | `reactions.rs:83`, `reactions.rs:45` |
| 9 / 40002 / 40003 / 40008 / 45001 / 45003 | `messages get` / `thread` / `search` | `messages.rs:276`, `:320`, `:361` |
| 30023 | all `notes` reads | `notes.rs:171`, `:189`, `:354`, `:681` |
| 30030 | `emoji list` / `export` | `emoji.rs:79`, `:96`, `:219` |
| 30176 (`KIND_TEAM`, `kind.rs:250`) | template team expansion | `channels.rs:405` |
| 30177 (`KIND_MANAGED_AGENT`, `kind.rs:259`) | template roster scan | `channels.rs:446` |
| 39000 (`KIND_NIP29_GROUP_METADATA`, `kind.rs:362`) | `channels list` / `get` / `search` | `channels.rs:54`, `:62`, `:131`, `:227` |
| 39002 (`KIND_NIP29_GROUP_MEMBERS`, `kind.rs:366`) | `channels list --member`, `channels members`, mention resolution | `channels.rs:37`, `:250`, `messages.rs:139` |
| 40100 | `canvas get` | `channels.rs:265` |
| 41001 (`KIND_DM_CREATED`, `kind.rs:453`) | `dms list` | `dms.rs:12` |
| (kindless) | `social event`, thread/channel resolution, `feed get` | `social.rs:75`, `messages.rs:63`/`:91`, `feed.rs:19` |

AGENTS.md gotcha 1 ("kind 39000 for channel metadata, not 41") holds: no file in
scope references 41 — `grep -n '\b41\b' channels.rs` finds only `41001`/`4101x`
in `dms.rs`, and `KIND_CHANNEL_METADATA = 41` (`kind.rs:54`) is documented there
as "Not used by Buzz today".

#### Kind constants vs. inline literals

AGENTS.md states all kind integers live in `buzz-core/src/kind.rs`. In practice
this module mixes three styles, sometimes inside one file:

| Style | Examples |
|-------|----------|
| Imported `buzz_core::kind::*` | `channels.rs:3` (`KIND_MANAGED_AGENT`, `KIND_TEAM`) |
| Imported `buzz_sdk::kind::*` | `dms.rs:103` (`KIND_DM_HIDE`), `channels.rs:1040` (`KIND_AGENT_PROFILE`), `emoji.rs:79` (`KIND_EMOJI_SET`), `social.rs:1-4` (list kinds) |
| Bare integer literal | `channels.rs:37,54,62,131,227,250,265`; `messages.rs:139,150,276,320,361,407`; `dms.rs:12,69,120`; `reactions.rs:45,83`; `social.rs:97,118` |
| **Locally redeclared constant** | `notes.rs:38` `pub const KIND_LONG_FORM: u16 = 30023` — a second definition of `kind.rs:66`, not an import |

`notes.rs:38` is the clearest deviation: the crate already depends on
`buzz-core`, and `KIND_LONG_FORM` exists at `kind.rs:66`, yet notes declares its
own `u16` copy and uses it everywhere including in printed coordinates
(`notes.rs:573`, `:747`).

#### Tag vocabulary emitted

All channel-scoped writes carry `h` (never `e`) as the channel key, matching
AGENTS.md. The `h` value is always a UUID string parsed by `parse_uuid`
(`validate.rs:19`) before the builder is called.

| Tag | Meaning | Emitted by |
|-----|---------|-----------|
| `h` | NIP-29 channel UUID | every channel write (`builders.rs:220`, `:284`, `:299`, `:340`, `:388`, `:424`, `:457`, `:530`, `:566`, `:588`, `:596`, `:628`, `:654`, `:663`, `:687`, `:704`, `:711`, `:720`, `:728`) plus `dms hide` (`dms.rs:99`) |
| `e` (`root` / `reply` marker) | NIP-10 threading | `builders.rs:171-181` from the `ThreadRef` built at `messages.rs:69-79` |
| `e` (bare) | edit / delete / vote / reaction target | `builders.rs:389`, `:425`, `:458`, `:470`, `:489`, `:496` |
| `p` | mention, member, DM participant, contact | `builders.rs:194` (mentions), `:567` (member), `dms.rs:62` (DM), `builders.rs:806` (contacts) |
| `d` | NIP-33 identifier | `dms.rs:63` (DM uuid), `emoji.rs` via `builders.rs:513`, `notes.rs:462` (slug) |
| `a` | NIP-33 coordinate deletion target | `notes.rs:713` `build_rm_event` |
| `t` | NIP-23 topic tag | `notes.rs:465` |
| `title`, `summary`, `published_at` | NIP-23 metadata | `notes.rs:463`, `:464`, `:466` |
| `emoji` (`[emoji, shortcode, url]`) | NIP-30 custom emoji | `builders.rs:490`, `:521` |
| `name`, `about`, `visibility`, `channel_type`, `ttl` | channel metadata | `builders.rs:687-698` (create), `:632-647` (update) |
| `topic`, `purpose`, `archived` | channel metadata | `builders.rs:655`, `:664`, `:711`, `:720` |
| `role` | member role | `builders.rs:575` |
| `broadcast` | also publish to public Nostr | `builders.rs:233` when `--broadcast` |
| `imeta` | NIP-92 media | `builders.rs:206-210` from `build_imeta_tag` (`client.rs:40`) |
| `repo`, `commit`, `file`, `parent-commit`, `branch`, `pr`, `l`, `description`, `truncated`, `alt` | diff metadata (`alt` is NIP-31) | `builders.rs:341-373`, populated from `DiffMeta` at `messages.rs:643-655` |
| `action_id`, `reason_code`, `public_reason` | moderation tombstone metadata | `builders.rs:418-427` from `messages.rs:685-689` |
| `auth` | NIP-OA owner attestation, injected for **every** event | `client.rs:588-596` |

#### Tag vocabulary parsed

| Tag | Parser | Site |
|-----|--------|------|
| `d` | `extract_d_tag` | `client.rs:1325`; used at `channels.rs:17`, `:44`, `:455`, `dms.rs:26` |
| `name`/`about` | `extract_tag_value` | `client.rs:1346`; `channels.rs:19-20` |
| `d`,`name`,`t`,`private`,`public`,`about`,`topic`,`purpose`,`archived` | `ChannelSummary::from_event` | `channels.rs:168-213` |
| `p` (+ role at index 3) | `extract_p_tags` | `client.rs:1366-1389`; `channels.rs:256` |
| `p` (canonicalized via `PublicKey::from_hex`) | `parse_member_pubkeys` | `messages.rs:226-241` |
| `e` with NIP-10 marker at index 3 | `find_root_from_tags` | `messages.rs:25-56` |
| `h` | inline scan | `messages.rs:100-110` (`resolve_channel_id`) |
| `emoji` | `emoji_tags_of` | `emoji.rs:19-47` |
| `d`,`title`,`summary`,`t`,`published_at` | `NoteSnapshot::from_event` | `notes.rs:100-152` |

Visibility is modelled asymmetrically: writes emit `["visibility","open"|"private"]`
(`builders.rs:690`), while reads look for NIP-29's **single-element** `["public"]`
/ `["private"]` tags (`channels.rs:70-90`, `channels.rs:186-189`). `cmd_list_channels`
additionally requires `a.len() == 1` (`channels.rs:82`), whereas
`ChannelSummary::from_event` accepts the tag at any arity — two different
readers of the same field in one file.

#### NIP-33 addressable identifiers

| Coordinate | Identifier | Site |
|------------|-----------|------|
| `30023:<pubkey>:<slug>` | note slug (`[a-z0-9._-]{1,80}`) | `notes.rs:271-274` `coord_for`; printed at `notes.rs:573`; bech32 `naddr` at `notes.rs:317-320` |
| `30030:<pubkey>:buzz:custom-emoji` | fixed d-tag constant | `emoji.rs:9` mirroring `builders.rs:503` |
| `39000:<relay>:<channel-uuid>` / `39002:…` | channel UUID as `d` | queried at `channels.rs:228`, `:251`, `messages.rs:140` |
| `30176:<owner>:<team_id>`, `30177:<owner>:<agent-pubkey>` | template roster resolution | `channels.rs:407`, `channels.rs:455` (the `d` tag of a 30177 **is** the agent pubkey) |

`notes.rs` accepts three input encodings for a coordinate — bech32 `naddr1…`,
`kind:pubkey:identifier`, and `nostr:naddr1…` — all via
`Coordinate::from_str` (`notes.rs:255`), then enforces `kind == 30023` and a
non-empty identifier (`notes.rs:257-267`).

#### JSON output shapes

| Command | Shape | Site |
|---------|-------|------|
| `messages get`/`thread`/`search`, `feed get` | array of `{id,pubkey,kind,content,created_at,tags}` (sig-stripped) | `normalize_events`, `client.rs:1307-1321` |
| the same, `--format compact` | array of `{id,content,created_at}` | `messages.rs:250-256`, `feed.rs:52-58` |
| `channels list` | array of `{channel_id,name,description,created_at}` | `channels.rs:16-23` |
| `channels list --format compact` | array of `{channel_id,name}` | `channels.rs:100-105` |
| `channels get` | object above **plus** `pubkey`, or the literal `null` | `channels.rs:235-239` |
| `channels search` | array of `{channel_id,name,channel_type,visibility,archived,about,topic,purpose}` | `channels.rs:154-164` |
| `channels members` | array of `{pubkey,role}` (role defaults to `"member"`) | `client.rs:1374-1385` |
| `canvas get` | raw markdown body, or `null` | `channels.rs:275`, `:277` |
| `channels create --template` | `{status,channel_id,template,canvas_applied,members_added,skipped,archived_excluded,member_failures}` + optional `archive_state_warning` | `channels.rs:804-815` |
| `reactions get` | `{"reactions":[{emoji,count,pubkeys}]}` sorted by emoji | `reactions.rs:107-124` |
| `emoji list`/`export` | `{"emojis":[{shortcode,url}]}` sorted by shortcode | `emoji.rs:86`, `:228`, `:300` |
| `dms list` | array of `{dm_id,participants,created_at}` | `dms.rs:38-42` |
| `dms open` | raw relay write response with `dm_id` (and defaulted `accepted`) grafted on | `dms.rs:84-91` |
| `notes get` | pretty-printed `NoteOutput` `{id,pubkey,naddr,coordinate,slug,title,summary,tags,published_at,updated_at,content}` | `notes.rs:301-336`, printed at `notes.rs:365-373` |
| `notes ls` | pretty-printed array of `NoteOutput`, newest-first | `notes.rs:375-386`, `:388` |
| `social event`/`notes`/`contacts`/`list`/`set-list` | **raw relay response, verbatim** | `social.rs:78`, `:110`, `:123`, `:207`, `:180` |
| all other writes | `{event_id,accepted,message}` | `normalize_write_response`, `client.rs:1420-1434` |
| `channels create` (non-template) | write response + `channel_id` | `print_create_response`, `client.rs:1391-1404` |

Two shapes break the "reads are sig-stripped JSON arrays / writes are
`{event_id,accepted,message}`" contract in AGENTS.md:

- `social.rs` prints the relay body unchanged (`social.rs:78,110,123,207`). The
  relay serializes full `nostr::Event` values (`bridge.rs:1125`
  `serde_json::to_value(&se.event)`), which include `sig`. So `social event`,
  `social notes`, `social contacts` and `social list` emit signatures while every
  other read strips them.
- `notes set` / `notes rm` print aligned plain-text key/value lines, not JSON
  (`notes.rs:571-580`, `notes.rs:747-748`); `notes get --content-only` prints raw
  markdown (`notes.rs:661-664`); `canvas get` prints raw markdown
  (`channels.rs:275`).

`emoji rm` on a shortcode that isn't present prints
`{"accepted":true,"message":"not present"}` with **no** `event_id` key
(`emoji.rs:148-153`) — a fourth write shape.

#### Local file format: `channel-templates.json`

`channel_templates.rs` deliberately duplicates the desktop app's wire shape
rather than sharing a crate (`channel_templates.rs:1-8`, module doc). Parsed
record (`channel_templates.rs:22-56`):

| Field | Type | Default | Site |
|-------|------|---------|------|
| `name` | `String` (required) | — | `channel_templates.rs:24` |
| `description` | `Option<String>` | `None` | `:26` |
| `channel_type` | `String` | `"stream"` (`fn default_channel_type`) | `:28`, `:51-53` |
| `visibility` | `String` | `"open"` (`fn default_visibility`) | `:30`, `:55-57` |
| `canvas_template` | `Option<String>` | `None` | `:32` |
| `agents.personas[].personaId` | `Vec<TemplateAgentEntry>` | `[]` | `:38-40`, `:44-47` |
| `agents.teams[].teamId` | `Vec<TemplateTeamEntry>` | `[]` | `:41-42`, `:49-52` |

The store is a bare JSON **array** of these records (`channel_templates.rs:87-89`).
The nested roster uses `rename_all = "camelCase"` (`:36`, `:43`, `:49`) while the
top-level record does not (`:22`) — so `channel_type` is snake_case on the wire
but `personaId` is camelCase. The test at `channel_templates.rs:180-197` pins
exactly that mixed casing.

Canvas templates support two placeholders only, substituted by plain string
replace: `{channel.name}` and `{template.name}` (`channels.rs:725-727`).

#### Test coverage — data model

Tested pure parsers: `ChannelSummary::from_event` (5 tests,
`channels.rs:1195-1259`, incl. malformed-tag tolerance at `:1252`),
`find_root_from_tags` (8 tests, `messages.rs:781-861`),
`parse_member_pubkeys` (3 tests, `messages.rs:1000-1010`, `:1063-1096`),
`NoteSnapshot::from_event` (4 tests, `notes.rs:901-965`),
`union_custom_emoji` (2 tests, `emoji.rs:330-388`),
`build_rm_event` a-tag/no-e-tag shape (`notes.rs:1264-1296`),
`build_set_event` tag matrix (10 tests, `notes.rs:1057-1229`),
template record parsing (`channel_templates.rs:149-197`).

Untested shapes, verified by grepping the `#[cfg(test)]` blocks of all ten files
for the identifier (zero matches each): `extract_channel_metadata`
(`channels.rs:16`), `emoji_tags_of` in isolation, the `reactions get` grouping
shape (`reactions.rs:96-124` — `reactions.rs` has no `#[cfg(test)]` module at
all), the `dms list` projection (`dms.rs:20-45` — no test module), the `feed get`
compact projection (`feed.rs:47-60` — no test module), `NoteOutput`'s
naddr/coordinate encoding (`notes.rs:314-336`).
