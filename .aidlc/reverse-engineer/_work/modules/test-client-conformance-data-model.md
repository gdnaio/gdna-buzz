## Module: buzz-test-client — Nostr interop & multi-tenant conformance E2E (`crates/buzz-test-client/tests`)
### Aspect: Data Model

Both files are black-box wire-protocol test clients. Neither owns a persistent data model; the
"data model" here is the shape of the Nostr events, tags, and JSON filter/response bodies each
file constructs and parses against a live relay over HTTP/WS. This doc catalogs those shapes and,
per the task brief, establishes precisely how `conformance_multitenant.rs` relates to the
`buzz-conformance` crate's `Tracer`/`check_trace` types.

---

#### 1. `e2e_nostr_interop.rs` — event/tag shapes per NIP

The file builds events with `nostr::EventBuilder`/`Tag::parse` and inspects relay responses as raw
`serde_json::Value` (REST) or typed `nostr::Event` (WS, via `buzz_test_client::BuzzTestClient`).

##### NIP-50 search filter shape

| Filter field | Construction | Line |
|---|---|---|
| `search` | `Filter::new().search(&unique_token)` | `crates/buzz-test-client/tests/e2e_nostr_interop.rs:288-291` |
| `#h` scoping | `.custom_tags(SingleLetterTag::lowercase(Alphabet::H), [channel.as_str()])` | `e2e_nostr_interop.rs:291` |
| `kind` | `.kind(Kind::Custom(9))` | `e2e_nostr_interop.rs:289` |
| mixed search+non-search (rejection case) | two `Filter`s in one `subscribe` call, one with `.search(...)`, one without | `e2e_nostr_interop.rs:355-364` |
| empty-results search | `Filter::new().search(...).kind(Kind::Custom(9))`, no `#h` | `e2e_nostr_interop.rs:411-414` |

##### NIP-10 thread `e`-tag shapes

| Tag form | Meaning | Construction site |
|---|---|---|
| `["e", <root_id>, "", "reply"]` | reply pointing directly at its parent, no distinct root | `e2e_nostr_interop.rs:459` (`test_nip10_thread_reply_creates_metadata`), `:562` (`test_nip10_unknown_parent_rejected` — fake parent) |
| `["e", <wrong_root_id>, "", "root"]` + `["e", <real_parent_id>, "", "reply"]` | two-marker form: explicit root marker disagrees with the reply's real parent | `e2e_nostr_interop.rs:571-572` (`test_nip10_root_mismatch_rejected`) |
| `["broadcast", "1"]` | non-NIP-10 custom tag that flips a depth-1 reply into the top-level view | `e2e_nostr_interop.rs:922` (`test_nip10_thread_reply_not_in_top_level`) |

The root/reply markers are the 4-element form `["e", <id>, <relay-hint>, <marker>]`; every
construction in this file leaves the relay-hint slot as `""` and only ever populates `"reply"` or
`"root"` in the marker slot (`e2e_nostr_interop.rs:459`, `:571-572`, `:906`, `:915`).

##### NIP-17 gift-wrap shape

| Element | Shape | Line |
|---|---|---|
| Gift-wrap event | `Kind::Custom(1059)`, content = opaque string (`"encrypted-content"` or a UUID-tagged probe), signed by an **ephemeral** `Keys::generate()` distinct from the connection's auth key | `e2e_nostr_interop.rs:597-604` (`test_nip17_gift_wrap_accepted`), `:706-712` (`test_nip17_gift_wrap_recipient_receives`) |
| Addressing tag | single `["p", <recipient_pubkey_hex>]` | `e2e_nostr_interop.rs:601`, `:709` |
| Subscription requiring `#p` | `Filter::new().kind(Kind::Custom(1059))` with no `#p` (expected `CLOSED`) vs. `.custom_tag(SingleLetterTag::lowercase(Alphabet::P), b_pubkey_hex)` (expected delivery) | `e2e_nostr_interop.rs:632` vs. `:697-700` |

Note: this file never constructs the NIP-17 **seal** or **rumor** inner layers (kind:13/kind:14) —
the gift-wrap `content` is an opaque placeholder string, not a real NIP-44-encrypted seal. The
tests exercise the relay's *transport* handling of kind:1059 (acceptance despite pubkey mismatch,
`#p`-gating, recipient delivery, non-searchability) — see business-rules and security docs — not
NIP-17's cryptographic unwrap chain. There is no client-side decrypt/unwrap step anywhere in this
file; "gift-wrap unwrapping" as such is out of scope for this test file's data model.

##### DM discovery (kind:39000 / kind:44100) shapes

| Kind | Role | Key tags asserted | Line |
|---|---|---|---|
| `41010` | DM-open command (custom kind, not core Nostr) | `["p", <other_pubkey_hex>]`, backdated `created_at` (now − 10s) to avoid an idempotent-dedupe collision on same-second re-open | `e2e_nostr_interop.rs:118-124` |
| `39000` | DM discovery/metadata event returned by the relay | asserted to carry `["hidden", ...]` and `["private", ...]` tags | `e2e_nostr_interop.rs:854-863` |
| `44100` | Membership notification | asserted to carry `["p", <a_pubkey_hex>]` and `["h", <channel_id>]` | `e2e_nostr_interop.rs:807-819` |

##### NIP-DV (relay-signed visibility snapshot, kind:30622) shapes

Not one of the three NIPs named in the module inventory, but the largest single behavior group in
the file (9 of 25 tests, `e2e_nostr_interop.rs:1254-1699`). Shape:

| Element | Construction | Line |
|---|---|---|
| Snapshot event | `Kind::Custom(30622)`, queried by `#p = viewer_hex` (the snapshot is `#p`-gated to its owner) | `e2e_nostr_interop.rs:1213-1215` (`read_snapshot_event`) |
| Hidden-set extraction | scans `event.tags` for `["h", <channel_id>]` entries | `e2e_nostr_interop.rs:1240-1249` (`read_hidden_dms`) |
| Hide command | `Kind::Custom(41012)`, tag `["h", <channel_id>]` | `e2e_nostr_interop.rs:1272-1277` |
| Re-open command | `Kind::Custom(41010)`, tag `["p", <other_pubkey_hex>]` | `e2e_nostr_interop.rs:1288-1293` |
| Third-party `ids`-escape-hatch probe | REST `POST /query` with `{"ids": [<snapshot_id>], "limit": 1}` — no `kinds` field, deliberately, to test the "kindless filters are gate-exempt" path | `e2e_nostr_interop.rs:1573-1575` |

##### Channel-window (kind:39005/39006) shapes

| Kind | Role | Shape detail | Line |
|---|---|---|---|
| `9` (query, `top_level: true` extension) | row events only | request built by `query_channel_window`, extension fields `top_level`, `include_summaries`, `include_aux`, `until`, `before_id` | `e2e_nostr_interop.rs:1703-1727` |
| `39005` | thread summary overlay | content is JSON with `reply_count`; asserted `pubkey != requester` (relay-signed) | `e2e_nostr_interop.rs:1852-1860` |
| `39006` | pagination bounds overlay | content is JSON with `has_more`/`next_cursor`; `d`-tag suffix encodes `"{channel}:head"` or `"{channel}:{cursor_ts}:{cursor_id}"`; exactly one per response, enforced by `partition_window`'s `assert_eq!(bounds.len(), 1, ...)` | `e2e_nostr_interop.rs:1741-1773` (`partition_window`), `:1866-1869`, `:1900-1906` |

`partition_window` (`e2e_nostr_interop.rs:1741-1773`) is the file's only non-trivial parsing
helper: it buckets a flat window response array into `(rows: Vec<9>, summaries: Vec<39005>,
bounds: 39006, aux: Vec<other>)` by matching on each element's `"kind"` field.

---

#### 2. `conformance_multitenant.rs` — tenant/community fixture shapes

##### Two-relay-one-process fixture

The file's entire fixture model is **two `Host`-distinguished base URLs against what the module
doc-comment asserts is a single relay process, single Postgres, single Redis**
(`crates/buzz-test-client/tests/conformance_multitenant.rs:29-31`):

```
fn url_a()       -> "http://a.localhost:3000"        (env RELAY_URL_A)
fn url_b()       -> "http://b.localhost:3000"        (env RELAY_URL_B)
fn url_unknown() -> "http://unknown.localhost:3000"  (env RELAY_URL_UNKNOWN)
```
(`conformance_multitenant.rs:44-61`)

There is no fixture or helper anywhere in the file that starts, configures, or seeds a relay
process, a `communities` table row, or a host→community mapping. The file's model is purely
"three URLs exist and someone else made A and B resolve to different communities on one relay
before this test ran" — see Integrations doc for what that "someone else" would have to be and
Configuration doc for the env vars.

##### Channel/community coexistence fixture

The recurring fixture across 5 of the 8 implemented test modules is a **shared channel UUID**
created independently in both communities, exploiting the `(community_id, id)` primary key the
module doc-comments describe (never verified against `buzz-db` source by this agent — this file's
own comments are the source of that claim, e.g. `conformance_multitenant.rs:1352-1357`):

| Module | Helper | Line |
|---|---|---|
| `channels_membership` | `create_channel(http_base, keys, channel_uuid)` — kind:9007, `visibility=open` | `conformance_multitenant.rs:1383-1408` |
| `workflows` | `create_open_channel(http_base, keys, channel_uuid)` — same kind:9007 shape, one keypair becomes owner-member on **both** sides | `conformance_multitenant.rs:1725-1740` |
| `search_fts` | `create_channel(http_base, keys, channel_uuid)` | `conformance_multitenant.rs:1995-2020` |
| `pubsub_presence_typing` | `create_open_channel(http_base, keys, channel_uuid)` | `conformance_multitenant.rs:2328-2347` |
| `row_zero_host_binding` (negative variant) | `create_open_channel(base_url, keys)` — channel exists **only** in B, deliberately not mirrored to A | `conformance_multitenant.rs:346-372` |

The kind:9007 tag shape is identical across all four positive-coexistence helpers: `["h",
<channel_uuid>]`, `["name", "<prefix>-<channel_uuid>"]`, `["channel_type", "stream"]`,
`["visibility", "open"]` — e.g. `conformance_multitenant.rs:319-324`, `:1387-1392`.

##### Workflow fixture (server-generated id, not client `d`-tag)

`define_workflow` (`conformance_multitenant.rs:1750-1770`) posts kind:30620 with a YAML `content`
body (`workflow_yaml`, `:1656-1668`) and a `["name", <name>]` tag — deliberately **not** a `d` tag,
because the module doc-comment (`:1758-1761`) states the server generates the workflow id and
returns it in the OK message's `response:{"workflow_id":"…"}` envelope. `trigger_workflow`
(`:1776-1801`) fires it back with kind:46020, tag `["d", <server_generated_id>]`.

##### NIP-98 replay fixture

`api_tokens_nip98_replay::nip98_replay_seenset_is_shared_and_community_scoped`
(`conformance_multitenant.rs:729-893`) constructs a raw `Authorization: Nostr <base64>` header
itself (`build_nip98_header`, `:870-880`) rather than reusing any relay or `buzz-auth` helper — the
event is `Kind::HttpAuth` with `["u", url]`, `["method", method]`, `["payload", <sha256-hex>]` tags.

##### `AbstractState`/`TraceStep`/`Tracer` — **absent from this file's data model**

The task brief asks this to be established precisely: **`conformance_multitenant.rs` does not
import, construct, or reference any `buzz_conformance` type.** Verified two ways:

- `grep -n 'buzz_conformance\|buzz-conformance\|Tracer\|check_trace' crates/buzz-test-client/tests/conformance_multitenant.rs` returns zero matches.
- The file's own `use` statements are scoped per-module and are exclusively `nostr::*`,
  `buzz_test_client::{BuzzTestClient, RelayMessage}`, `reqwest`, `base64`, `sha2`, `uuid` — e.g.
  `conformance_multitenant.rs:562-564`, `:919-920`, `:1345-1346`. No module imports
  `buzz_conformance`.
- `Cargo.toml` confirms `buzz-conformance` is not even a declared dependency of `buzz-test-client`
  (dev or otherwise) — see `crates/buzz-test-client/Cargo.toml:1-32`, cross-checked against the
  `buzz-conformance`-focused analysis batch's integrations doc, which independently states the
  crate's only two dependents are the workspace root and `buzz-relay`.

So this file's "data model," despite its name, carries **zero** `CommunityLabel`/`HostLabel`/
`ActorLabel`/`TraceAction`/`AbstractState`/`Verdict` values. Every tenant-isolation assertion in
this file operates on wire-level JSON/event shapes (`serde_json::Value`, `nostr::Event`,
`reqwest::StatusCode`) constructed and inspected independently of the `buzz-conformance` crate's
types. This is the single most important data-model fact for this module: **the test-time
verification of the multi-tenant boundary that this file performs is wire-black-box, not a replay
of `buzz-conformance`'s abstract trace model.** See Business Rules, Integrations, and Security docs
for the consequences.

##### `pending_lane` sentinel — a data shape, not test data

`pending_lane(lane: &str, obligation: &str) -> !` (`conformance_multitenant.rs:64-69`) is a
`#[track_caller]` function that always panics via `todo!("conformance pending [{lane}]: {obligation}")`
(`:68`). It is called from 8 of the file's 27 test-shaped items (see full inventory in
Business Rules doc) in place of a real fixture — e.g. `archive_in_a_does_not_affect_b`
(`:906-911`), `same_event_id_and_dtag_coexist_across_communities` (`:1291-1298`). These are
`#[ignore]`d `#[tokio::test]` functions whose entire "fixture" is the panic string; there is no
event, tag, or HTTP shape to document for them because no wire traffic is ever generated. The
distinct top-level `#[allow(clippy::todo, unused)]` (`conformance_multitenant.rs:40`) is required
specifically to keep this pattern lint-clean.

---

#### 3. Cross-file data-model note: `RelayMessage`/`OkResponse` reuse

Both files consume the same wire-message enum from the shared harness (context-only per task
scope, not analyzed here as primary subject): `buzz_ws_client::RelayMessage` (`Event`, `Eose`,
`Closed`, `Notice`, `Auth`, `Ok`), re-exported through `buzz_test_client::{parse_relay_message,
OkResponse, RelayMessage, WsClientError}` (`crates/buzz-test-client/src/lib.rs:11`). Both files
match on `RelayMessage::Closed { subscription_id, message }` to assert rejection reasons
(`e2e_nostr_interop.rs:377-390`; `conformance_multitenant.rs` does not use WS `REQ`/`CLOSED`
directly except via `search_fts`'s `search_for` helper, `:2035-2052`, which calls
`BuzzTestClient::collect_until_eose`). Neither file extends or wraps this enum further.
