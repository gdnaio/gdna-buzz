## Module: buzz-cli — channel, message & social commands (`crates/buzz-cli/src/commands`)
### Aspect: Security

Threat model note: per AGENTS.md, `BUZZ_RELAY_URL` / `BUZZ_PRIVATE_KEY` /
`BUZZ_AUTH_TAG` are auto-injected into managed agent subprocesses by the ACP
harness. So every argument to these commands can be attacker-influenced in the
"prompt-injected agent" case, and every value printed to stdout flows back into an
agent's context. Both directions matter below.

#### Identifier validation

| Input | Validation | Site |
|-------|-----------|------|
| `--channel` (writes) | `parse_uuid` → `Uuid::parse_str` | `validate.rs:19`; `messages.rs:497`, `channels.rs:851`, `:868`, `:884`, `:898`, `:910`, `:922`, `:934`, `:946`, `:962`, `:993`, `:1056`, `dms.rs:97`, `:119` |
| `--channel` (reads) | `validate_uuid` | `messages.rs:272`, `:312`; `channels.rs:225`, `:247`, `:263` |
| `--event` / `--reply-to` / `--before-id` | `validate_hex64` (length 64 + `is_ascii_hexdigit`) | `validate.rs:29`; `messages.rs:313`, `:494`, `:598`, `:676`, `:706`, `:729`; `reactions.rs:15`, `:41`; `social.rs:26`, `:73`, `:93`; `notes.rs:209` |
| `--pubkey` | `validate_hex64` then SDK `check_hex_len`/`check_pubkey_hex` | `channels.rs:957`, `:988`; `dms.rs:55`, `:120`; `social.rs:88`, `:116`, `:186` |
| relay-supplied `p` tags | canonicalized through `PublicKey::from_hex` | `messages.rs:236-238` |
| relay-supplied `h` tag | `Uuid::parse_str`, error if not a UUID | `messages.rs:104-107` |
| `--naddr` | `Coordinate::from_str` + kind must be 30023 + non-empty identifier | `notes.rs:255-267` |
| `--name` (note slug) | `parse_slug`: 1–80 chars, `[a-z0-9._-]` only | `notes.rs:50-70` |
| `--shortcode` | SDK `normalize_custom_emoji_shortcode`: ≤64 bytes, `[A-Za-z0-9_-]`, lowercased | `builders.rs:127-149`, called `emoji.rs:129`, `:142`, `:269` |
| `--url` (emoji) | SDK `check_custom_emoji_url`: non-empty, ≤2048 bytes, `http://`/`https://` prefix | `builders.rs:152-168`, called `emoji.rs` via builders and `reactions.rs:21` |
| `--ttl` | positive, fits `i32` | `channels.rs:822-830` |
| `--kind` (messages) | allowlist {9,45001,45003} | `messages.rs:540-572` |
| `--kind` (social list) | allowlist of six | `social.rs:127-139` |
| `--types` (feed) | allowlist of four | `feed.rs:31-39` |
| `--role`, `--policy`, `--type`, `--visibility`, `--direction` | string allowlists | `channels.rs:283-298`, `:964-976`, `:1006-1013`; `messages.rs:730-737` |

`validate_hex64` accepts mixed case (`is_ascii_hexdigit`, `validate.rs:31`) while
the "must be a 64-character hex string" error message and `messages.rs:396-398`'s
lowercasing imply lowercase; the relay side is unaffected because
`EventId::parse`/`PublicKey::from_hex` normalize.

#### Unvalidated free-text reaching published events

| Input | Cap / check | Effective bound |
|-------|------------|-----------------|
| `messages send --content` | `validate_content_size` 64 KiB (`messages.rs:492`) + SDK `check_content` | bounded |
| `messages edit --content` | same (`messages.rs:707`) | bounded |
| `social publish --content` | SDK only (`builders.rs:739`) | bounded, but exits 4 not 1 on violation |
| `messages send-diff --diff` | truncated at 60 KiB hunk boundary (`messages.rs:614`) | bounded |
| `channels topic --topic` | **none** — no client check; `build_set_topic` has no `check_content` (`builders.rs:652-658`) | unbounded until the relay rejects |
| `channels purpose --purpose` | **none** (`builders.rs:661-667`) | unbounded |
| `channels update --name/--description` | name canonicalized and non-empty-checked (`builders.rs:625-631`); description unchecked | unbounded description |
| `canvas set --content` | **none** — `cmd_set_canvas` never calls `validate_content_size` (`channels.rs:1049-1063`) and `build_set_canvas` has no cap (`builders.rs:529-532`) | unbounded |
| `notes set --content <literal>` | **none** for the non-stdin path (`notes.rs:508`); `build_set_event` has no content check (`notes.rs:418-469`) | unbounded via argv |
| `messages delete --reason-code/--public-reason` | **none** (`messages.rs:685-689`, `builders.rs:418-427`) | unbounded, and these land in a room-visible tombstone |
| `social set-list --tags` | tags parsed from arbitrary JSON and passed through verbatim (`social.rs:145-155`) | arbitrary tag names/values |
| `social set-contacts --contacts` | pubkey hex + relay_url ≤2048 + petname ≤256 in the SDK (`builders.rs:764-810`) | bounded |

Nothing in this module HTML-escapes, shell-escapes or otherwise sanitizes content
before publishing — correct for a Nostr client (content is opaque), but it means
the *rendering* clients carry the full XSS/markdown-injection burden for CLI-authored
canvas, topic, purpose and tombstone-reason values, none of which the CLI bounds.

One notable positive: `social set-list --tags` cannot forge a NIP-OA `auth` tag.
`sign_event` counts `auth` tags after signing and rejects any count that differs
from the injected expectation (`client.rs:598-610`), so `--tags '[["auth", …]]'`
fails with `CliError::Other` instead of publishing a spoofed owner attestation.
Every write in this module goes through `sign_event`, including the three
hand-built events (`dms.rs:70`, `channels.rs:1043`, `social.rs:177`).

#### File paths and traversal

| Argument | Operation | Validation |
|----------|-----------|-----------|
| `emoji import --file <path>` | `std::fs::read_to_string` | none (`emoji.rs:166-167`) |
| `emoji export --file <path>` | `std::fs::write` (silent overwrite) | none (`emoji.rs:187-188`) |
| `channels create --templates-file <path>` | `std::fs::read_to_string` | none — the override short-circuits path resolution entirely (`channel_templates.rs:71-75`, `:95-96`) |
| `messages send --file <path>` | full read + magic-byte MIME allowlist + size cap | `client.rs:1101-1126` |
| default template path | `dirs::data_dir()` + hardcoded bundle id | `channel_templates.rs:71-84` |

There is no path normalization, allowlisting, or symlink check on any of these
(`grep -n 'canonicalize\|starts_with(\"/\")\|components()' emoji.rs channel_templates.rs`
returns zero matches). Consequences for an agent-driven CLI:

- `emoji export --file <any writable path>` is an arbitrary-file **write** (the
  content is attacker-shaped only insofar as shortcodes/URLs are, but the
  truncate-and-overwrite is unconditional).
- `emoji import --file <any readable path>` is an arbitrary-file **read**; only
  files that parse as `{"emojis":[{shortcode,url},…]}` reach a published event, so
  exfiltration through this path is narrow but not zero (a URL field can carry up
  to 2048 bytes and is published verbatim).
- `messages send --file` is the broader exfiltration path: any host file whose
  magic bytes match `image/jpeg|png|gif|webp` or `video/mp4`
  (`client.rs:64-70`) is uploaded to relay media and linked into a channel
  message. `MAX_IMAGE_BYTES`/`MAX_VIDEO_BYTES` (50 MB / 500 MB,
  `client.rs:73-76`) are checked **after** the whole file is read into memory
  (`client.rs:1107` then `:1117-1124`).

#### Secrets and key material

`grep -rn 'secret_key\|to_secret_hex\|nsec' crates/buzz-cli/src/commands/{channels,notes,messages,emoji,social,dms,reactions,feed,channel_templates,mod}.rs`
returns **zero matches** — no file in scope touches the private key. All identity
use is `client.keys().public_key()` (`channels.rs:35`, `:650`; `messages.rs:145`;
`emoji.rs:94`; `dms.rs:9`; `feed.rs:13`; `notes.rs:169`, `:205`, `:718`;
`reactions.rs:41`). Signing is delegated to `client.sign_event`. No command prints
or logs the relay URL, auth tag, or NIP-98 header.

The only key-adjacent leak surface is `notes.rs` printing the caller's own pubkey
in coordinates and error text (`notes.rs:573`, `:723`, `:747`) — public data.

#### DM confidentiality

`dms` provides channel-shaped DMs with **no encryption**: `open` publishes kind
41010 with plaintext `p` tags (`dms.rs:57-70`), and the conversation body is then
sent with `messages send --channel <dm-uuid>` as plaintext kind 9. `grep -rn
'nip44\|nip04\|encrypt\|gift_wrap\|GiftWrap' crates/buzz-cli/src` matches only
`agent_management.rs` (observer-frame encryption, out of scope). NIP-17 gift wrap
(kind 1059, `kind.rs:60`) is never used here. Confidentiality therefore rests
entirely on relay-side channel ACLs, and `dms list` exposes participant pubkeys
for every conversation the caller can read (`dms.rs:26-42`).

`dms hide` (kind 41012 → relay kind 30622 `KIND_DM_VISIBILITY`) is a per-user
preference the relay treats as p-gated and *not* `ids`-exempt
(`req.rs:1051-1063`), so hide choices can't be enumerated by a third party who
learns an event id. Nothing in the CLI weakens that.

#### Authorization assumptions

No command performs a local authorization check before writing. `channels
add-member`, `remove-member`, `archive`, `delete`, `messages edit`, `messages
delete` (including the moderator-tombstone metadata), `canvas set`, `channels
update` all sign and submit; the relay is the only enforcement point. The one
local gate is `channels set-add-policy`'s
`BUZZ_ACP_ALLOWED_CHANNEL_ADD_POLICIES` allowlist (`channels.rs:1021-1033`), and
its own comment states it is bypassable by submitting kind 10100 directly and
that relay-side enforcement is intentionally out of scope
(`channels.rs:1015-1020`). That is a client-side-only control by design, which is
worth flagging explicitly: an agent that can run `buzz` can also run any Nostr
client with the same key.

Template roster resolution does enforce a security-relevant invariant locally: the
effective owner is the verified auth-tag owner, else the signing pubkey, with no
sole-author fallback, so a same-slug kind-30176/30177 published by another
principal cannot be selected (`channels.rs:645-651`). The auth tag itself is
verified against the signer at startup (`lib.rs:1753-1762`), so
`auth_tag_owner_hex()` (`client.rs:576-584`) is trustworthy by the time
`channels.rs` reads it.

The NIP-IA archive filter **fails open** (`channels.rs:511-524`,
`:526-587`): an untrusted archived-identities snapshot results in no filtering
plus a warning, rather than refusing to add agents. The rationale given is that
filtering can only remove ambiguity, never create it — sound, but it means a relay
that can suppress or corrupt its own kind-13535 list can get archived agent
identities added to a fresh channel, with the only signal being a stderr/report
warning (`channels.rs:597`, `:816-818`).

#### Output injection into terminals and agent context

Most output is serialized through `serde_json`, which escapes control characters.
Three paths print relay-supplied bytes verbatim:

| Path | Site |
|------|------|
| `canvas get` prints the canvas body raw | `channels.rs:275` |
| `notes get --content-only` prints the note body raw | `notes.rs:661-664` |
| the ambiguous-slug error interpolates relay-supplied note titles into stderr text | `notes.rs:279-297`, raised at `notes.rs:645-650` |

Any member who can write a canvas or a note can therefore emit ANSI escape
sequences, carriage returns or prompt-injection text directly into another user's
terminal or an agent's context window. There is no escaping or control-character
stripping (`grep -n 'escape_default\|strip_ansi\|is_control' channels.rs notes.rs`
→ zero matches). `channels search` mitigates by construction (names go through
`serde_json::to_string`, `channels.rs:148`), and `messages get` likewise via
`normalize_events`.

A second, subtler channel: `emoji list`/`export` echo relay-supplied `url` values
into JSON (`emoji.rs:41-45`). They are constrained to `http(s)` ≤2048 bytes **on
write** (`builders.rs:152-168`) but read paths do **not** re-validate, so a URL
published by an older client or a hostile relay is echoed as-is and may be
followed by a consumer.

#### Resource exhaustion

| Vector | Bound | Site |
|--------|-------|------|
| `messages send --content -` | **unbounded** stdin read; the 64 KiB check happens after the whole read | `validate.rs:186-197` then `messages.rs:490-492` |
| `messages send-diff --diff -` | **unbounded** stdin read, truncated afterwards | `messages.rs:611-614` |
| `canvas set --content -` | **unbounded** stdin read, no cap at all | `channels.rs:1055` |
| `notes set --content -` | capped at 1 MiB + 1 byte via `Read::take`, then rejected | `notes.rs:485`, `:490-500` |
| `emoji import` (stdin) | capped at 10 MB via `Read::take` | `emoji.rs:160`, `:172-177` |
| `emoji import --file` / `--templates-file` | **unbounded** `read_to_string` | `emoji.rs:166`, `channel_templates.rs:95` |
| `messages send --file` | whole file read into RAM before the size check | `client.rs:1107` |
| `emoji list`, `emoji export --scope workspace`, `reactions get`, `reactions remove`, `canvas get` | no `limit` in the filter → relay default 2000 events | `emoji.rs:78-81`, `:218-221`; `reactions.rs:82-85`, `:44-48`; `channels.rs:264-267`; relay bound `req.rs:25`, `:878-882` |
| template roster scan | `query_all` — paginates to exhaustion over all of the owner's kind-30177 events | `channels.rs:449`, `client.rs:724-729` |
| `channels list` / `search` | paginate to `--limit` events (defaults 500 / 1000) then filter client-side | `channels.rs:31`, `:133` |

Three of the four stdin paths differ in policy for no stated reason, and the two
with no cap (`canvas set`, `messages send`) are the ones whose content is also
never length-validated before signing.

#### Test coverage — security

Covered: slug character/length rules (6 tests, `notes.rs:826-857`), hex64 and
UUID validation (`validate.rs:212-262`, sibling file), content-size boundary
(`validate.rs:266-282`), diff truncation including UTF-8 boundary handling
(`validate.rs:319-359`), the add-policy allowlist (4 tests,
`channels.rs:1310-1382`), TTL overflow rejection (`channels.rs:1287-1289`),
pubkey filtering of relay-supplied `p` tags including non-hex and off-by-one
lengths (`messages.rs:1078-1096`), archive fail-open semantics and warning
propagation (9 tests, `channels.rs:1470-1683`).

Not covered, verified by grepping every `#[cfg(test)]` block in the ten files for
the identifier with zero hits: `read_source` and `write_output` path handling
(`emoji.rs:164`, `:185`), template `--templates-file` override with a hostile path
(only the happy path at `channel_templates.rs:138-141`), the absence of a canvas
content cap, the raw-print paths (`channels.rs:275`, `notes.rs:661`), the
unbounded-stdin paths, reaction emoji matching, and the `auth`-tag forgery defense
as exercised through `cmd_set_list` (`client.rs:598-610` has no test in scope; I
did not check `client.rs`'s own test modules for one).
