## Module: buzz-test-client — core protocol, status & team E2E (`crates/buzz-test-client`)
### Aspect: Business Rules

`e2e_relay.rs` (2,477 lines, 38 tests) is the largest single source of business-rule proof in this group and covers seven distinct rule clusters. `nip42_host_binding_live.rs`, `e2e_user_status.rs`, and `e2e_team.rs` each cover one focused cluster.

#### 1. Connection lifecycle and authentication

| Test | Rule asserted |
|---|---|
| `test_connect_and_authenticate` (`:207-215`) | Baseline: `BuzzTestClient::connect` completes a full NIP-42 round trip against a live relay. |
| `test_unauthenticated_rejected` (`:756-786`) | An unauthenticated connection's write attempt must be denied, but the test explicitly tolerates **three** different denial shapes: `OK false`, `ConnectionClosed`, or `Timeout` (`:768-783`) — the test does not pin a single behavior. Cross-checked against the relay: `handle_event` (`crates/buzz-relay/src/handlers/event.rs:634-645`) unconditionally sends `OK false "auth-required: not authenticated"` for any non-`Authenticated` connection — the relay's actual behavior is deterministic (`OK false`), so the test's `ConnectionClosed`/`Timeout` tolerance is broader than necessary and could mask a future regression that silently drops the event instead of answering with `OK false` (see Debt). |
| `test_pubkey_mismatch_rejected` (`:592-609`) | An event signed by a different keypair than the one that completed NIP-42 auth on that connection must be rejected — proven by connecting as `keys_a` then calling `send_text_message(&keys_b, ...)`, which builds and signs the event with `keys_b` while sending it over `keys_a`'s authenticated socket. Confirmed against `event.rs:658-660`: `if event.pubkey != auth_pubkey && !is_gift_wrap { reject(...) }` — gift-wrap (kind 1059) is the one deliberate exception, not tested here. |
| `test_auth_event_kind_rejected` (`:568-590`) | A kind:22242 (`KIND_AUTH`) event submitted via `EVENT` (not `AUTH`) must be rejected, message containing "invalid" or "auth". Confirmed at `ingest.rs:1465-1468`: `if kind_u32 == KIND_AUTH { return Err(Rejected("invalid: AUTH events cannot be submitted")) }`. |
| `test_client_submitted_nip43_membership_snapshots_are_rejected` (`:216-244`) | A forged kind:13534 (`KIND_NIP43_MEMBERSHIP_LIST`, relay-only) event is rejected identically over both WS (`restricted: relay-only kind`) and HTTP `/events` (400, same message) — proven only after first showing the same actor can submit an *ordinary* event, so the rejection is provably kind-specific, not a broader auth failure. Confirmed at `ingest.rs:1481-1483` (`is_relay_only_kind` gate) and `kind.rs:758-768` (the `matches!` list: `KIND_NIP43_MEMBERSHIP_LIST`, `KIND_CHANNEL_SUMMARY`, `KIND_PRESENCE_SNAPSHOT`, `KIND_DM_VISIBILITY`, `KIND_THREAD_SUMMARY`, `KIND_WINDOW_BOUNDS`). |
| `test_membership_notification_kind_rejected` (`:1499-1524`) | Client-submitted kind:44100 must be rejected with a message mentioning "relay-signed only" or "relay". Confirmed at `ingest.rs:1470-1473`. |

#### 2. NIP-42 relay-URL host binding (`nip42_host_binding_live.rs`)

This file is the live, two-host proof for a specific fix: the NIP-42 AUTH event's `relay` tag must be checked against the **per-tenant host the connection actually arrived on**, not the deployment-wide `config.relay_url` constant. Four `#[ignore]`d tests (all require `--ignored --test-threads=1` against a live two-community relay, per the module doc comment `:20-22`):

| Test | Connection host | Signed `relay` tag | Expected outcome |
|---|---|---|---|
| `nip42_matching_host_accepted_a` (`:103-112`) | A | A | ACCEPT |
| `nip42_matching_host_accepted_b` (`:116-125`) | B | B | ACCEPT |
| `nip42_cross_host_rejected_a_relay_tag_on_b_connection` (`:131-145`) | B | A | REJECT, message must contain `"auth-required"` or `"verification"` |
| `nip42_cross_host_rejected_b_relay_tag_on_a_connection` (`:150-159`) | A | B | REJECT |

The two "accepted" tests are not cosmetic — the file's own comment on the second one (`:150-153`, referring back to the first pair) states they exist so the cross-host tests can't pass vacuously (e.g., if `nip42_expected_relay_url` always produced a URL nothing could match, every AUTH would fail and the cross-host tests would "pass" without proving anything about host-binding specifically). Confirmed against the relay: `handlers/auth.rs:80-82` computes `relay_url = nip42_expected_relay_url(&state.config.relay_url, &conn.tenant)` *per connection*, and `api/bridge.rs:225-...` derives that URL from `tenant.host()`, not the global config string. The relay's own unit tests pin the identical regression (`api/bridge.rs:2722-2726`: "This test bites if `nip42_expected_relay_url` is reverted to return `config.relay_url` verbatim — the exact regression the helper guards"), so this E2E file is the live-infrastructure counterpart to an existing unit-level regression guard, not a first-of-its-kind check.

The doc comment frames this as "sibling of the NIP-98 row 44 obligation" (`:1-5`) — confirmed: `bridge.rs:178-182` documents the identical per-tenant-host rule for the NIP-98 HTTP `u`-tag ("Conformance row 44 obligation: NIP-98 `u` URL host must match `req.community`"), and `bridge.rs:208-212` explicitly labels `nip42_expected_relay_url` as the "NIP-42 (WebSocket AUTH) sibling" of the NIP-98 helper. The two auth paths (WS NIP-42, HTTP NIP-98) enforce the same host-binding invariant through parallel, independently-named helper functions.

#### 3. Subscription/EVENT semantics: filters, EOSE, ephemeral events, closing

| Test | Rule asserted |
|---|---|
| `test_send_event_and_receive_via_subscription` (`:280-322`) | Baseline pub/sub round trip across two independently-authenticated clients on the same channel. |
| `test_large_event_frame_below_configured_limit_is_accepted` (`:324-364`) | An event whose serialized `["EVENT", event]` frame is >64 KiB (the *old* cap) but <512 KiB (the *new* default, `DEFAULT_MAX_FRAME_BYTES = 512 * 1024`, `config.rs:14`) is accepted, and the socket remains usable for a follow-up send afterward. This pins a specific historical limit change — the test's own assertions (`:340-348`) explicitly check the frame exceeds 65,536 bytes and stays under 512 KiB, i.e. it is testing the *gap* between the two historical values, not just "large events work." |
| `test_subscription_filters_by_kind` (`:366-419`) | A REQ filtered by `kind` receives only matching-kind events; a non-matching kind sent immediately after does not arrive within 500ms. |
| `test_close_subscription_stops_delivery` (`:421-457`) | After `CLOSE`, a subsequent matching event on the same channel produces no delivery to that subscription (still accepted by the relay — only fan-out stops). |
| `test_stored_events_returned_before_eose` (`:936-966`) | A previously-sent event is returned as part of the historical REQ batch, before `EOSE`. |
| `test_ephemeral_event_not_stored` (`:968-1000`) | A kind:20001 event (ephemeral range 20000-29999) is accepted but a subsequent REQ for that kind/channel returns nothing — ephemeral events bypass DB storage, matching `ARCHITECTURE.md`'s documented ephemeral sub-pipeline. |
| `test_eose_sent_for_empty_subscription` (`:611-635`) | A REQ with `since(now)` on a freshly created channel returns an empty historical batch before EOSE — proves EOSE fires even with zero matches. |
| `test_multiple_concurrent_clients` (`:642-675`) | Three independently-authenticated clients all subscribed to the same channel+kind each receive one broadcast event — fan-out is not first-subscriber-only. |

#### 4. Subscription limits and NIP-11 self-description

| Test | Rule asserted |
|---|---|
| `test_subscription_limit_enforced` (`:697-754`) | Opening exactly `MAX_SUBSCRIPTIONS` (1024) subscriptions succeeds; the 1025th REQ receives `CLOSED` with a message containing "too many". Confirmed at `handlers/req.rs:26,66-70`: `const MAX_SUBSCRIPTIONS: usize = 1024;` and `if !subs.contains_key(&sub_id) && subs.len() >= MAX_SUBSCRIPTIONS { ... "error: too many subscriptions" }`. The test's own comment (`:698-702`) is candid that opening all 1024 subs is slow, and that this test's real job is cross-checking the *actual enforcement constant* against the *advertised* NIP-11 value tested separately below — it is not an independent stress test of the full limit's performance characteristics. |
| `test_nip11_relay_info` (`:557-590`) | `GET /info` returns `name`, `description`, `supported_nips`, `version`, and a `limitation` object with `max_subscriptions: 1024` and `auth_required: true`. Confirmed at `nip11.rs:104,109`: `max_subscriptions: Some(1024)`, `auth_required: true`. The test's own comment (`:583-586`) states the auth_required assertion exists specifically because "the REQ, EVENT, and COUNT handlers unconditionally require an authenticated connection" — this is a documentation-accuracy check on the NIP-11 payload, not a behavioral probe of the handlers themselves (those are covered by `test_unauthenticated_rejected` and friends). |

Together, these two tests are the E2E-level cross-check that the advertised NIP-11 metadata and the actual enforcement constant haven't drifted apart — a real but narrow contract, since the 1024-subscription test only exercises a fraction of the concurrency the number implies before hitting the overflow condition.

#### 5. NIP-05/kind:0 profile sync side effect

`test_kind0_nip05_sync` (`:463-560`) is a five-step regression test:
1. Publish kind:0 with `nip05` matching the relay's own domain → accepted.
2. Query it back via `POST /query` and confirm the stored content round-trips the `nip05` field verbatim.
3. `GET /.well-known/nostr.json?name=<local>` resolves to the publishing pubkey.
4. Publish a second kind:0 with an **off-domain** `nip05` (`...@evil.com`) → still accepted (the event itself is valid; only the side effect differs).
5. The NIP-05 lookup from step 3 no longer resolves — the relay's side effect **clears** the stored `nip05_handle` when the domain doesn't match the connection's own host.

This is the only test in this group that proves a write-time side effect independent of the event's own acceptance — the kind:0 event is always accepted regardless of the `nip05` value; only the *derived* profile-lookup state changes. No relay source file for this side effect was read in this pass (out of this group's declared scope), but the test's behavior is self-consistent and unambiguous about what it proves.

#### 6. NIP-29 group-admin policy surface (`channel_add_policy`, roles, archive/unarchive)

This is the densest rule cluster in the file — nine tests exercising the interaction between a channel owner's authority, an agent's own self-service `channel_add_policy` setting, and (for private channels) member-vs-elevated-role distinctions.

| Test | Rule asserted | Relay confirmation |
|---|---|---|
| `test_nip29_put_user_default_policy_allows` (`:1160-1194`) | With no policy set, a third party (channel owner) can PUT_USER an agent into their own channel. | `side_effects.rs:335-360` third-party-add branch: unset policy (`get_agent_channel_policy` returns `None`) falls through to allow. |
| `test_nip29_put_user_nobody_blocks` (`:1279-1330`) | Setting `channel_add_policy: "nobody"` via kind:10100 blocks any third party from adding that agent; rejection message contains `"policy:nobody"`. | `side_effects.rs:352-357`. |
| `test_nip29_put_user_self_add_bypasses_policy` (`:1332-1379`) | An agent can always add **itself** even with `"nobody"` set — self-add is checked before the policy branch. | `side_effects.rs:328-332`: `if target_pubkey == actor_bytes { return Ok(()); }` runs *before* the third-party policy match. The test explicitly calls `.allow_self_tagging()` on the `EventBuilder` (`:1367`) — a client-side annotation, not itself an authorization mechanism; the actual bypass is server-side. |
| `test_nip29_put_user_owner_only_blocks` (`:1349-1403`, second occurrence at that range under a different name) | `channel_add_policy: "owner_only"` blocks anyone except the agent's registered NIP-OA owner; message contains `"policy:owner_only"`. | `side_effects.rs:341-350`. |
| `test_nip29_standard_client_flow` (`:1409-1607`) | End-to-end: channel creation triggers kind:39000/39001/39002 discovery/admins/members events; kind:9 with `h`-tag is accepted and kind:9 *without* an `h`-tag is rejected; kind:7 reaction and kind:5 deletion targeting a just-sent message are both accepted. | Multiple relay paths; the no-`h`-tag rejection specifically proves stream messages are channel-scoped by construction, not merely by convention. |
| `test_private_channel_any_member_can_invite` (`:2229-2262`) | In a **private** channel, any existing member (not just the owner) can PUT_USER a new invitee — the "Slack model" per the test's own comment. | `side_effects.rs:299-306`: the private-channel gate only checks *membership*, not role, before allowing an invite. |
| `test_private_channel_non_member_cannot_invite` (`:2264-2298`) | A non-member cannot invite into a private channel; message mentions "not authorized" or "not a channel member". | `side_effects.rs:307` (`Err(anyhow!("actor not authorized"))`). |
| `test_private_channel_member_cannot_grant_admin` (`:2300-2344`) | A regular (non-elevated) member can invite, but cannot grant an **elevated** role (e.g. `admin`) to the invitee — only owners/admins may do that. | `side_effects.rs:310-322`: `if role.is_elevated() { ... if !actor_role.is_elevated() { return Err(anyhow!("only owners/admins may grant elevated roles")); } }`. |
| `test_unarchive_emits_member_added_notification` (`:1774-1873`) | Archiving then unarchiving a channel causes the relay to re-emit a kind:44100 "member_added" notification to every current member on their always-live global membership subscription — used purely as a resubscribe trigger, since archiving evicts channel-scoped live subscriptions via `CLOSED "channel access revoked"` and unarchive otherwise gives connected agents no signal to resubscribe. | Confirmed verbatim against the relay's own comment: `side_effects.rs:1512-1520` ("We reuse the member_added notification (44100) purely as a resubscribe trigger — no membership actually changed here"). The relay's comment also documents a known limitation (four sub-second archive/unarchive toggles by the same actor could collide event IDs and skip a fan-out) that the test does not probe. |

The `channel_add_policy` cluster as a whole establishes a layered authorization model: **self-add always wins → then explicit per-agent policy (`nobody`/`owner_only`/anything else defaults to `anyone`) → then, for private channels specifically, membership for the *inviter* and elevated-role membership for role *grants*.** No single test covers all three layers at once; the layering is only visible by reading the five tests together against the one relay function (`validate_admin_event`'s kind:9000 branch, `side_effects.rs:293-361`).

#### 7. Relay-signed membership notification (kind:44100/44101) read-side gating

Six tests, all proving variants of one invariant: **a subscription that could match kind:44100/44101 must carry a `#p` filter whose *only* value is the authenticated pubkey, or the relay closes it.**

| Test | Filter shape tried | Outcome |
|---|---|---|
| `test_membership_notification_emitted_on_add` (`:1230-1298`) | `kinds:[44100,44101]` + `#p=<own pubkey>` + `since` | Accepted; live 44100 arrives with correct `p`/`h` tags when a REST-driven add happens |
| `test_membership_notification_requires_p_filter` (`:1300-1338`) | `kinds:[44100,44101]`, no `#p` | `CLOSED "restricted: ..."` |
| `test_membership_notification_wildcard_filter_rejected` (`:1340-1376`) | Empty filter (no kinds, no `#p`) — "can match 44100/44101" | `CLOSED "restricted: ..."` |
| `test_membership_notification_requires_own_p_filter` (`:1378-1418`) | `#p=<someone else's pubkey>` | `CLOSED "restricted: ..."` |
| `test_membership_notification_emitted_on_remove` (`:1420-1517`) | valid own-`#p` filter, then a REST-driven remove | Accepted; live 44101 arrives |
| `test_membership_notification_multi_p_rejected` (`:1519-1567`) | `#p` containing **both** own pubkey and a victim's pubkey | `CLOSED "restricted: ..."` — proves partial self-inclusion doesn't satisfy the gate; *every* `#p` value must equal the authenticated pubkey |
| `test_membership_notification_mixed_filter_rejected` (`:1569-1610`) | Two filters in one REQ: filter 1 is `#h`-scoped+44100 (would pass a naive per-filter-only check), filter 2 is a bare `authors` filter with no kinds (globally scoped, wildcard) | `CLOSED "restricted: ..."` — proves the gate evaluates the *subscription's effective scope* across all filters (OR semantics), not each filter independently; a second, broader filter can't be smuggled in alongside a compliant first one |

No relay-side source file implementing this specific gate (`P_GATED_KINDS`, imported at `handlers/req.rs:10`, `RESULT_GATED_KINDS`) was read to its exact enforcement lines in this pass — the tests' consistent "restricted" substring and 1:1 mapping to distinct filter shapes make the rule's existence and boundary unambiguous from the black-box side, but a full pass connecting each test to a specific `req.rs` line is left for a handlers-focused group.

#### 8. Relay membership / invite tokens (HTTP door, NIP-98-signed)

| Test | Rule asserted |
|---|---|
| `test_invite_mint_and_claim_admits_new_pubkey` (`:246-278`) | Owner mints a code (`POST /api/invites`); a new pubkey claims it (`POST /api/invites/claim`) and gets `status: "joined", role: "member"`; claiming the *same* code again returns `status: "already_member"` rather than erroring — claims are idempotent by design. Confirmed at `invites.rs:355-360`. |
| `test_invite_claim_rejects_invalid_code` (`:281-291`) | A garbage code returns 403 with `error: "invite_invalid"` — deliberately coarse (per the relay's own comment at `invites.rs:59-60`, "stays coarse so the endpoint is a poor oracle") so the endpoint can't be used to distinguish "malformed" from "expired-but-well-formed" from "wrong community" by brute-force probing (expired codes get their own more specific `invite_expired` message, `invites.rs:56-58`, which the test does not probe). |
| `test_invite_mint_requires_owner_or_admin` (`:293-303`) | A `member`-role relay member and a total outsider both get 403 attempting `POST /api/invites` — only owner/admin may mint. |
| `test_invite_code_minted_for_one_host_fails_on_another` (`:305-326`) | A code minted while bound to host A's tenant fails to claim when presented against host B — invite tokens are tenant-scoped, matching the multi-community host-binding model documented elsewhere in the codebase. |

#### 9. Live thread-summary push (kind:39005)

`test_reply_ingest_pushes_live_thread_summary` (`:2382-2477`) proves the `AGENTS.md`-documented "Thread counters" convention end-to-end over the wire: posting a reply to a thread root causes the relay to push a live kind:39005 event (not merely update a DB row) whose JSON content shows `reply_count: 1`; deleting that same reply causes a second live kind:39005 push showing `reply_count: 0`. This is the only test in this group that proves a *live push* (post-EOSE) rather than a stored-and-queryable side effect — the subscription is opened and drained to EOSE *before* either mutation, so both summary events must arrive via fan-out, not historical replay.

#### Cross-cutting observations

- Every "expect either A or B" assertion pattern in this file (`test_unauthenticated_rejected`, `test_user_status_stale_write_rejected` in the sibling status file) signals the test author was not fully certain of the relay's exact behavior at the boundary being tested. In at least one case (`test_unauthenticated_rejected`) the relay's actual behavior is more deterministic than the test insists on, meaning the test is strictly weaker than it could be against current source.
- The `channel_add_policy` and private-channel-role clusters are the only places in this group where authorization logic is layered (self > explicit policy > membership > role) rather than a flat allow/deny — and no single test exercises all three layers together, so a future regression that reorders two of the layers (e.g., making policy override self-add) would not necessarily be caught by any one existing test, only by the combination.
