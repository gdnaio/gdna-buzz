## Module: buzz-test-client — content & git E2E (`crates/buzz-test-client/tests`)
### Aspect: Data Model

This group's four test files construct and expect specific Nostr event/tag shapes for four unrelated features: NIP-23 long-form articles, agent/owner authorization events layered on stream messages and channel-admin commands, NIP-ER encrypted reminders, and git ref/manifest state over S3. None of these files define new data types — they build `nostr::Event`s with `EventBuilder` and assert on wire-level tag shapes returned by the relay.

#### Long-form articles (`e2e_long_form.rs`) — kind 30023

The suite hardcodes `const KIND_LONG_FORM: u16 = 30023` (`e2e_long_form.rs:26`) rather than importing a constant from `buzz-core`. This matches the relay's registry: `KIND_LONG_FORM: u32 = 30023` is documented as "NIP-23: Long-form content ... Parameterized replaceable (NIP-33, 30000–39999 range) ... Stored globally (channel_id = NULL)" (`crates/buzz-core/src/kind.rs:60-63`).

Every article is built by `build_long_form_event(keys, d_tag, title, content, extra_tags)` (`e2e_long_form.rs:37-48`), which always emits exactly two base tags — `["d", d_tag]` and `["title", title]` — then splices in caller-supplied `extra_tags`. This is the minimal NIP-23 shape the tests treat as canonical; there is no `summary`, `image`, or `t` (topic) tag anywhere in this file, so those NIP-23 fields are entirely untested here.

Tag shapes exercised, by test:
| Test | Extra tags beyond `d`/`title` | Purpose |
|---|---|---|
| `test_long_form_accepted` (`:56-79`) | none | baseline accept |
| `test_long_form_retrievable` (`:82-126`) | none | REQ round-trip by kind+author |
| `test_long_form_stray_h_tag_ignored` (`:128-180`) | `["h", <random-uuid>]` (`:143`) | proves a channel `h` tag on a global kind does not scope storage |
| `test_long_form_nip33_replacement` (`:182-240`) | none (same `d`, two publishes) | NIP-33 replace-by-`(pubkey,kind,d)` |
| `test_long_form_stale_write_rejected` (`:242-317`) | none, but `custom_created_at` set explicitly ±100s (`:255`, `:267`) | stale-write ordering |
| `test_long_form_a_tag_deletion` (`:319-396`) | none (target), plus a separate kind:5 event with `["a", "30023:<pubkey>:<d>"]` (`:367`) | NIP-09 addressable deletion |
| `test_long_form_malformed_e_plus_a_does_not_delete` (`:398-457`) | separate kind:5 with `["e", "not-a-valid-event-id"]` + `["a", <coord>]` (`:427-430`) | malformed-`e`-tag routing guard |
| `test_long_form_set_twice_preserves_published_at` (`:459-537`) | `["published_at", "<unix-secs>"]` carried across two publishes (`:475`, `:511`) | `published_at` vs `created_at` distinction |

The addressable coordinate string built for deletion is `"{kind}:{pubkey_hex}:{d_tag}"`, i.e. `format!("{}:{}:{}", KIND_LONG_FORM, keys.public_key().to_hex(), d_tag)` (`e2e_long_form.rs:359-364`, reused `:420-425`) — this is the standard NIP-01/NIP-33 `a`-tag coordinate format, not a Buzz-specific shape.

`published_at` is modeled as a plain public tag on the article event itself (`["published_at", "<value>"]`), not a separate data structure — the test asserts the *relay's stored copy* still carries the original value after a second publish with a later `created_at` (`e2e_long_form.rs:508-535`). This mirrors the carry-forward rule implemented client-side in `buzz-cli`'s `build_set_event` (`crates/buzz-cli/src/commands/notes.rs:418-460`, specifically the `published_at` line at `:459`: "preserve prior if present, otherwise set to `now`"), but the e2e test reimplements the carry inline in the raw event builder rather than calling that function — see Debt.

#### Owner/agent authorization events (`e2e_human_edit_agent_content.rs`) — kinds 40003, 5, 9000, 9001, 9002, 9005, 9007, 9008

This file does not define a new "agent content" event shape. An agent-authored message is an ordinary kind:9 stream message (`send_text_message(&agent_keys, &channel_id, &content, 9)`, e.g. `e2e_human_edit_agent_content.rs:108-111`) signed by the agent's own keypair — nothing in the event distinguishes "agent-authored" from "human-authored" at the wire level. What makes an author an "agent" is purely relational: a NIP-OA `auth` tag establishing ownership, checked against `buzz-db`'s `is_agent_owner` (`crates/buzz-db/src/user.rs:354` / `crates/buzz-db/src/lib.rs:1867`), not any field on the message event itself.

The NIP-OA `auth` tag is `["auth", "<owner-pubkey-hex>", "<conditions>", "<sig-hex>"]` per the doc comment in `crates/buzz-sdk/src/nip_oa.rs:8-16`, and the test builds it via `nip_oa::compute_auth_tag(owner_keys, &agent_keys.public_key(), "kind=9")` then `nip_oa::parse_auth_tag(&tag_json)` (`e2e_human_edit_agent_content.rs:75-78`). The signature covers `preimage = "nostr:agent-auth:" || agent_pubkey_hex || ":" || conditions` (`nip_oa.rs:14-16`, `build_preimage` `:113-115`), so `is_agent_owner` is a fact recorded when the agent authenticates (`connect_agent_with_owner`, `e2e_human_edit_agent_content.rs:82-91`, which calls `client.authenticate_with_nip_oa(agent_keys, &auth_tag)` — `lib.rs:118-124`), not encoded on every subsequent message.

Event shapes constructed:
| Kind | Shape built in this file | Line |
|---|---|---|
| 9007 (CREATE_GROUP) | `["h", <uuid>], ["name", ...], ["channel_type","stream"], ["visibility","open"\|"private"]` | `:41-46` (open), `:601-609` (private) |
| 9 (stream message) | via harness `send_text_message` — `["h", channel_id]` | `lib.rs:135-141` |
| 40003 (STREAM_MESSAGE_EDIT) | `["e", msg_event_id], ["h", channel_id]`, new content in `.content` | e.g. `:118-123` |
| 5 (standard NIP-09 deletion) | `["e", msg_event_id], ["h", channel_id]` | e.g. `:339-344` |
| 9005 (NIP-29 DELETE_EVENT) | `["e", msg_event_id], ["h", channel_id]` | e.g. `:250-255` |
| 9002 (NIP-29 EDIT_METADATA) | `["h", channel_id], ["name", ...]` or `["archived","true"]` | `:428-433`, `:465-469` |
| 9008 (NIP-29 DELETE_GROUP) | `["h", channel_id]` only | `:540-543` |
| 9000 (PUT_USER) | `["h", channel_id], ["p", victim_pubkey_hex]` | `:857-862` |
| 9001 (REMOVE_USER) | `["h", channel_id], ["p", victim_pubkey_hex]` | `:882-887` |

All nine kind constants used (40003, 5, 9000, 9001, 9002, 9005, 9007, 9008) match `buzz-core/src/kind.rs`: `KIND_STREAM_MESSAGE_EDIT = 40003` (`:445`), `KIND_DELETION = 5` (`:56`), `KIND_NIP29_PUT_USER..DELETE_GROUP = 9000..9008` (`:262-273`). No new kind is introduced by this test file.

The "private-channel" variant is the same 9007 shape with `["visibility", "private"]` instead of `"open"` (`create_private_agent_owned_channel`, `:591-624`) — visibility is a tag value, not a distinct event kind.

#### Event reminders (`e2e_event_reminder.rs`) — kind 30300 (NIP-ER)

`const KIND_EVENT_REMINDER: u16 = 30300` (`e2e_event_reminder.rs:26`) matches `KIND_EVENT_REMINDER: u32 = 30300` in `buzz-core/src/kind.rs:98-103`, documented there as "author-only" and included in `AUTHOR_ONLY_KINDS` (`kind.rs:113`).

Every reminder is built by `build_reminder(keys, d_tag, extra_tags)` (`:52-64`) or the timestamp-controlled `build_reminder_at(keys, d_tag, created_at, extra_tags)` (`:70-86`), both of which always emit `["d", d_tag]` and `["alt", "Encrypted reminder"]`, then splice `extra_tags`. `.content` is always the literal placeholder string `"nip44-ciphertext-placeholder"` (`:58-60`, `:80-82`) — this file never constructs or decrypts real NIP-44 ciphertext; the reminder's actual `{target, status, note}` JSON payload defined in `docs/nips/NIP-ER.md` ("Content" section, lines ~85-95) is out of scope for every test here, which only exercises the plaintext tag envelope (`d`, `not_before`, `expiration`).

Tag combinations exercised (all on top of the `d` + `alt` base):
| Tag(s) added | Tests | Spec cross-check |
|---|---|---|
| `not_before` valid decimal | `test_reminder_accepted_with_valid_not_before` (`:161-176`) | NIP-ER.md:69 |
| no `not_before` | `test_reminder_accepted_missing_not_before` (`:177-192`), `test_reminder_accepted_expiration_without_not_before` (`:437-456`) | NIP-ER.md:58-64 (bookmark/terminal) |
| two `not_before` tags | `test_reminder_rejected_duplicate_not_before` (`:193-215`) | — |
| `not_before="007"` (leading zero) | `test_reminder_rejected_malformed_not_before_leading_zero` (`:216-235`) | NIP-ER.md:69 |
| `not_before` ∈ {"abc","-1","1.0","1e3"," 1",""} | `test_reminder_rejected_malformed_not_before_non_digits` (`:236-260`) | NIP-ER.md:69 |
| `not_before="9007199254740992"` (>MAX_SAFE_INTEGER) | `test_reminder_rejected_not_before_above_max_safe_integer` (`:261-280`) | NIP-ER.md:69 |
| `not_before`+`expiration` (expiration < / == / > not_before) | `:281-348` (three tests) | NIP-ER.md:139 |
| `expiration` malformed, no `not_before` ordering check | `test_reminder_accepted_with_malformed_expiration` (`:349-371`) | — |
| missing / empty / duplicate `d` | `:372-436` (three tests) | NIP-ER.md `d` MUST be present, non-empty, single |
| `not_before="0"` | `test_reminder_not_before_zero_accepted` (`:765-781`) | NIP-ER "0 is valid per spec" |
| `not_before="9007199254740991"` (exactly MAX_SAFE_INTEGER) | `test_reminder_not_before_max_safe_integer_rejected_too_far_in_future` (`:782-807`) | NIP-ER.md:60 vs `:130` (structurally valid but exceeds horizon) |
| `not_before` = now+2y | `test_reminder_rejected_not_before_too_far_in_future` (`:1059-1097`) | horizon check |

Replacement semantics use the standard NIP-33 key `(pubkey, kind, d_tag)` — `test_reminder_replacement_semantics` (`:808-879`) publishes two versions with explicit 2-second-separated `created_at` (`v1_ts`/`v2_ts`, `:817-819`) specifically to avoid a same-second tiebreak, and asserts only the later `not_before` value survives.

The scheduler-delivery test (`test_scheduler_delivers_due_reminder_to_author_subscription`, `:1141-1177`) treats the *same* kind:30300 event as being delivered twice through two different channels — once historically (pre-EOSE, proving storage) and once live post-EOSE (proving the scheduler pushed it) — via the helper `await_scheduler_push` (`:1098-1140`) which matches on `has_d_tag` (`:1087-1093`), i.e. the data model distinguishes "stored" vs "delivered-live" only by *when* the identical event arrives on the wire, not by any different shape.

#### Git refs and manifest state (`e2e_git.rs`) — kind 30617, plus non-Nostr git/S3 objects

The only Nostr event this file constructs is a kind:30617 repo announcement: `EventBuilder::new(Kind::from(30617), "").tags([Tag::parse(["d", &repo]), Tag::parse(["name", "e2e git repo"])])` (`e2e_git.rs:216-222`, repeated for the concurrent test at `:363-368`). `30617` matches `KIND_GIT_REPO_ANNOUNCEMENT: u32 = 30617` in `buzz-core/src/kind.rs:437` ("NIP-34: Repository announcement (parameterized replaceable, d-tag = repo-id)"). The test never touches the sibling `KIND_GIT_REPO_STATE = 30618` event directly — ref-state observation happens entirely through git plumbing and a direct S3 read, not through a Nostr query.

The bulk of this file's data model is *not* Nostr — it is the git-on-object-storage manifest-pointer protocol described in `docs/git-on-object-storage.md`. The test defines its own client-side mirror of the pointer shape:
```rust
struct PointerSnapshot { etag: String, digest: String }  // e2e_git.rs:112-115
```
`GitS3Probe::pointer()` (`:152-169`) GETs the object at a computed key and asserts the body is exactly a 64-char lowercase-hex digest (`:163-165`), matching the spec's manifest-digest pointer body (`docs/git-on-object-storage.md`, "System Model": `pointer(R) = (e, d)` — ETag `e`, manifest digest `d`). `require_pointer()` (`:172-185`) polls up to 40×250ms and additionally calls `assert_manifest_exists(digest)` (`:186-193`), which GETs `manifests/{digest}` — directly probing the object-store layout the spec's Push protocol step 6 writes to (`docs/git-on-object-storage.md`, "Push" step 6: `d_after := PUT manifest-object(m_after)`).

The S3 key layout the test assumes is `repos/{owner}/{repo}/pointer` or, if `BUZZ_E2E_GIT_COMMUNITY_ID` is set, `repos/{community}/{owner}/{repo}/pointer` (`pointer_key`, `:141-148`) — this is the test's own inferred layout; it is not quoted from `git-on-object-storage.md`, which describes the pointer abstractly as "a single mutable object" without naming the literal S3 key. This is a plausible drift point if the relay's actual key scheme changes (not verified against relay source in this pass, since `buzz-relay/src/api/git/store.rs` is outside this group's scope — see Debt).

Everything else the test manipulates is native git state (commits, branches, tags, refs) driven through subprocess `git` commands and asserted via `git rev-parse`, `git log`, `git tag` output — there is no Buzz-specific data model for these; they are exactly git's own object model, and the test's job is to prove the relay round-trips them faithfully through the CAS-manifest layer.

#### Cross-cutting observation

None of the four files import or construct anything from `buzz_core::kind` — every kind number is a locally-hardcoded `u16`/`Kind::Custom`/`Kind::from` literal (`e2e_long_form.rs:26`, `e2e_event_reminder.rs:26`, inline literals in `e2e_human_edit_agent_content.rs` and `e2e_git.rs`). This means a future renumbering in `buzz-core/src/kind.rs` would not be caught by a compile error in this crate — cross-checked independently against each constant's current value in `kind.rs` above, and all four matched at the time of this analysis.
