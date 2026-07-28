## Module: buzz-push-gateway (`crates/buzz-push-gateway`)
### Aspect: Business Rules

#### Overview

The gateway implements a five-stage capability lifecycle: (1) App Attest challenge issuance, (2) installation enrollment (binds an App Attest key to an encrypted APNs token), (3) relay delegation (mints an opaque `endpoint_grant` capability for one relay), (4) delivery (a relay redeems the grant to request one APNs push), and (5) rotation/revocation of either the installation's endpoint or a specific relay delegation. Every mutating step re-verifies a fresh App Attest assertion against a single-use challenge except delivery, which is authenticated by NIP-98 instead (the caller there is a relay, not the installation).

#### Challenge issuance and single-use enforcement

`POST /v1/installations/challenges` generates 32 random bytes via `getrandom::fill` (http.rs:120-123), sets `expires_at = now + 300` (http.rs:116-119), and stores it via `AuthorityStore::put_challenge`. Every subsequent mutating request (`enroll`, `delegate`, `rotate_endpoint`, `revoke_delegation`, `revoke_installation`) must present a `challenge_id` + `challenge` pair that is consumed exactly once via `consume_challenge(id, value, now)` (authority.rs:127-131 trait method), which on Postgres is a single `DELETE ... WHERE id=$1 AND challenge_hash=$2 AND expires_at >= $3` and rejects unless exactly one row was deleted (`postgres.rs:122-129`). This makes replay of a challenge structurally impossible: the second use of the same `(id, value)` finds no row to delete.

#### Exact App Attest transcript construction (anti-replay binding)

Every attestation/assertion call signs a **transcript** — a domain string plus a compact, field-ordered JSON object — never the raw request bytes. Each mutating route builds its own transcript struct with a fixed `audience` string naming the exact route URL, so a signed payload for one route cannot be replayed against another:

| Route | Transcript domain | `audience` | Citation |
|---|---|---|---|
| `enroll` | `buzz.push.enroll.v1` | `https://push.buzz.xyz/v1/installations` | http.rs:186, 178-188 (`EnrollTranscript`) |
| `delegate` | `buzz.push.delegate.v1` | `https://push.buzz.xyz/v1/delegations` | http.rs:326, 282-292 (`DelegateTranscript`) |
| `rotate_endpoint` | `buzz.push.rotate-endpoint.v1` | `https://push.buzz.xyz/v1/installations/endpoint` | http.rs:411, 370-379 (`RotateTranscript`) |
| `revoke_delegation` | `buzz.push.revoke-delegation.v1` | `https://push.buzz.xyz/v1/delegations/revoke` | http.rs:471, 445-453 (`RevokeDelegationTranscript`) |
| `revoke_installation` | `buzz.push.revoke-installation.v1` | `https://push.buzz.xyz/v1/installations/revoke` | http.rs:520, 494-502 (`RevokeInstallationTranscript`) |

The shared helper `transcript(domain, value)` (http.rs:104-107) serializes the struct to JSON and prepends `"{domain}\n"`. This is the exact bytes fed to `AppAttestVerifier::verify_attestation`/`verify_assertion` as `client_data`.

#### Installation enrollment (`enroll`, http.rs:156-235)

Order of checks, all enforced before any authority-store call:
1. `endpoint_bytes(&r.endpoint)` must decode as valid lowercase hex ≤ `MAX_ENDPOINT_HEX_BYTES*2` chars (http.rs:167-170, calling `valid_endpoint` at http.rs:55-61).
2. `r.v == WIRE_VERSION`, `r.endpoint_epoch == 1` (first enrollment always starts at epoch 1), `r.expires_at` in `(now, now + max_installation_lifetime_seconds]`, and `r.app_profile` present in `s.enabled_profiles` (http.rs:171-178).
3. `challenge` must base64url-decode to exactly 32 bytes (http.rs:179-182, `decode_challenge`).
4. The enrollment transcript is built and passed to `AppAttestVerifier::verify_attestation`, which validates Apple's attestation chain against the pinned root cert, the configured `app_id`, the key id, and the transcript (http.rs:190-199, `app_attest.rs:44-71`).
5. Only after successful attestation verification is the challenge consumed (http.rs:200-206) — an attacker who does not hold a valid App Attest key can never burn a legitimate challenge (the challenge is consumed *after* verification succeeds, not before).
6. The raw endpoint (APNs token bytes) is sealed with `TokenKeyring::seal` (http.rs:207-210) and the installation row is created with `endpoint_epoch: 1`, `assertion_counter: 0` (http.rs:211-222).

`create_installation` on Postgres is `INSERT ... ON CONFLICT DO NOTHING` and rejects unless exactly one row was inserted (`postgres.rs:139-145`), which is the enforcement point for "installation id collision" and — via the table's `UNIQUE(app_profile, token_fingerprint)` constraint — "this exact APNs token is already registered under this profile." The in-memory store enforces the same token-uniqueness rule explicitly with a comment: "Token possession alone never supersedes a live installation" (authority.rs:216-218).

**Rule enforced only by comment, not by a type or runtime check:** `app_attest.rs:1-5`'s module doc comment states "App Attest authenticates the app instance and exact request transcript. It does not prove an Apple-issued binding between that key and the APNs token; accepting the directly submitted token is the protocol's explicit bootstrap assumption." Nothing in the code enforces or double-checks that the submitted `endpoint` actually belongs to the attested device — it is accepted on trust at enrollment time. This is also stated normatively in `docs/nips/NIP-PL.md` ("Apple documents no APNs-token-to-App-Attest-key binding; token provenance at enrollment is an explicit bootstrap assumption").

#### Assertion counter monotonicity (replay defense for installation-scoped operations)

Every non-enrollment mutating route (`delegate`, `rotate_endpoint`, `revoke_delegation`, `revoke_installation`) funnels through `verify_installation_assertion` (http.rs:238-269, private), which: loads the installation, verifies the assertion against `installation.app_attest_public_key` and `installation.assertion_counter` (the *previous* counter value), consumes the challenge, then calls `advance_assertion_counter(id, previous, next)`. On Postgres this is a compare-and-swap: `UPDATE ... SET assertion_counter=$3 WHERE id=$1 AND assertion_counter=$2 AND revoked_at IS NULL`, requiring `next <= previous` to be rejected before the query even runs (`postgres.rs:172-180`). This means two concurrent requests both built against the same starting counter value can only have one win — the loser's assertion (which the App Attest library already required to have a monotonically-increasing `signCount` relative to what was passed in) fails the CAS and the whole request fails. The counter itself is extracted from the assertion's `authenticatorData` field, bytes 33..37 big-endian (`app_attest.rs:120-135`, `assertion_counter`).

#### Relay delegation and endpoint-grant issuance (`delegate`, http.rs:295-353)

Field validation before any store or crypto call: `r.v == WIRE_VERSION`, `valid_relay_pubkey(&r.relay_pubkey)` (64 lowercase hex chars, http.rs:62-66), `r.endpoint_epoch >= 1`, `r.generation >= 1`, `r.not_before <= now + 300` (5-minute future-skew allowance), `r.expires_at > r.not_before`, and `r.expires_at <= now + s.max_grant_lifetime_seconds` (http.rs:307-315). After the installation-assertion check passes, `upsert_delegation` is called. On Postgres this locks the installation row `FOR UPDATE`, rejects if the installation is revoked, if its `endpoint_epoch` no longer matches `d.endpoint_epoch`, or if `d.expires_at` exceeds the installation's own `expires_at` (postgres.rs:186-193) — **a delegation can never outlive the installation it is attached to.** The `INSERT ... ON CONFLICT (installation_id, relay_pubkey) DO UPDATE ... WHERE EXCLUDED.generation > push_gateway_delegations.generation` clause (postgres.rs:195-196) enforces strictly-increasing `generation` per `(installation, relay)` pair at the database level, not just in application code.

Once the delegation is persisted, the handler re-reads the installation to get its `AppProfile` (http.rs:340-343) and constructs an `EndpointGrant`, sealed via `GrantKeyring::issue` (http.rs:345-352). The grant's plaintext never contains the raw APNs token — only `delegation_id`, `relay_pubkey`, `app_profile`, `endpoint_epoch`, `generation`, `expires_at` (model.rs:41-53). This is the capability the relay stores and replays on every delivery attempt.

#### Endpoint rotation (`rotate_endpoint`, http.rs:382-437)

`new_endpoint_epoch` must equal `endpoint_epoch.saturating_add(1)` exactly (http.rs:394-397) — epochs cannot skip or jump backward. The new raw endpoint is re-sealed under the *current* token key (http.rs:426-429) and `rotate_endpoint` on Postgres is a CAS on `(id, endpoint_epoch=$2)` (postgres.rs:209-224). Because every live delegation's `endpoint_epoch` is checked against the installation's current epoch at delivery time (see below), **a successful rotation atomically invalidates every previously issued grant for that installation** without needing to touch the `push_gateway_delegations` table at all — this is stated explicitly in `docs/nips/NIP-PL.md` ("A successful atomic rotation invalidates every grant sealed to the old epoch").

#### Revocation (`revoke_delegation` / `revoke_installation`)

`revoke_delegation` (http.rs:457-490) requires the caller-supplied `generation` to be strictly greater than the delegation's current generation; Postgres enforces this as `UPDATE ... SET generation=$3, revoked_at=now() WHERE ... generation<$3` and rejects unless exactly one row changed (postgres.rs:245-251). `revoke_installation` (http.rs:506-550) uses the same epoch-increment pattern as rotation (`new_endpoint_epoch = endpoint_epoch + 1`, http.rs:513-516) combined with setting `revoked_at`, which per the epoch-check at delivery time invalidates every delegation across every relay in one step, not just one.

#### Delivery admission (`deliver`, http.rs:553-706) — the core business rule

This is the most involved rule in the crate, implemented as a single Postgres transaction in `authorize_delivery` (postgres.rs:262-336):
1. NIP-98 header must be present and `verify_auth_header` must succeed against the exact `s.delivery_url`, method `POST`, `Timestamp::now()`, and the SHA-256 hash of the *received* body bytes (http.rs:562-579). The event id from that same header is extracted separately (http.rs:565-569, `auth_event_id`) and is the value burned in `push_gateway_delivery_auth_replays`.
2. The `endpoint_grant` is opened with `GrantKeyring::open` — this fails closed (`404 invalid_grant`) on any tampering, wrong key id, or unknown key id (http.rs:580-583).
3. The opened grant is cross-checked against the caller: `grant.relay_pubkey != relay` (the NIP-98-authenticated identity) is rejected, along with epoch/generation `< 1`, an expired grant, or a request `expires_at` outside `[now, grant.expires_at]` (http.rs:586-596). **This is the rule that makes a grant single-relay-bound**: a relay cannot present another relay's grant even if it somehow obtained the ciphertext, because the NIP-98 signer identity must equal the embedded `relay_pubkey`.
4. `authorize_delivery` then, inside one transaction: locks the installation row `FOR UPDATE` and validates it is unrevoked, unexpired, and its `endpoint_epoch` matches the grant's epoch (postgres.rs:283-294); locks the delegation row `FOR UPDATE` and validates it matches on `(delegation_id, relay, epoch, generation)`, is unrevoked, and `not_before <= now <= expires_at` (postgres.rs:296-310); re-validates the request's `expires_at` bound (postgres.rs:317-319); then performs the quota upsert, the auth-event-replay insert, and the request-id-replay insert, requiring all three to affect exactly one row each, and commits (postgres.rs:320-338). Any single failure aborts the whole transaction — nothing is charged or fenced on a rejected admission.
5. Only after a `DeliveryPermit` is obtained does the handler check `permit.authority.profile != grant.app_profile` (http.rs:625-632) — an extra defense-in-depth check that cannot actually be hit through the normal flow (the profile is derived from the same installation, so if it changed via a race, the epoch check would already have rejected it); if hit, the handler calls `finish_delivery(permit, Terminal)`, permanently burning the fence rather than leaving it retryable — a deliberate fail-safe choice for an unexpected state.
6. The endpoint token is decrypted via `TokenKeyring::open` only at this final step (http.rs:637-647) — the raw APNs token exists in process memory for the shortest possible window, right before the outbound APNs call.

#### Endpoint delivery quota — "rate limiting"

The gateway's only rate limit is a fixed-window counter per `token_fingerprint` (i.e. per physical APNs endpoint, not per relay or per installation): `BUZZ_PUSH_ENDPOINT_QUOTA_WINDOW_SECONDS` (default 10s) / `BUZZ_PUSH_ENDPOINT_QUOTA_MAX_DELIVERIES` (default 10). The Postgres upsert (postgres.rs:320-321) resets the window when `window_started_at <= now - window_seconds`, otherwise increments `admitted` only if `admitted < max_deliveries`, and the query's own `WHERE` clause re-evaluates against the row it just locked — so under a two-writer race for the last slot, the loser observes `admitted` already at the ceiling and updates zero rows (this exact race is what `concurrent_admissions_never_over_admit_past_quota_ceiling`, postgres.rs:796, is designed to prove). There is no quota keyed on relay identity, installation, or IP address — a single relay hammering the delivery endpoint for many *different* installations is not rate-limited at all by this mechanism (see `push-gateway-security.md`).

#### APNs retry/backoff policy

`classify(code, reason, timestamp)` (apns.rs:41-70) is a pure function mapping an APNs HTTP status + `reason` string + optional timestamp to one of six `DeliveryOutcome` variants:

| APNs response | Outcome | Citation |
|---|---|---|
| `200` | `Accepted` | apns.rs:43 |
| `410` + reason `Unregistered` | `InvalidEndpoint{unregistered_at: timestamp}` | apns.rs:44-46 |
| `400` + reason `BadDeviceToken`/`DeviceTokenNotForTopic` | `InvalidEndpoint{unregistered_at: None}` | apns.rs:47-51 |
| `403` + reason `ExpiredProviderToken` | `RefreshCredential` | apns.rs:52-54 |
| `403` (other reason) or `429`+`TooManyProviderTokenUpdates` | `ConfigurationFault` | apns.rs:55-57 |
| `429`/`500`/`503`, or reason ∈ {`IdleTimeout`,`InternalServerError`,`ServiceUnavailable`,`Shutdown`,`TooManyRequests`} | `Retry{retry_after_seconds: None}` | apns.rs:58-67 |
| anything else | `PermanentRequestFault` | apns.rs:69 |

The transport layer (`ApnsTransport::send`, apns.rs:213-269) performs **at most one retry per delivery attempt**, and only for the specific `RefreshCredential` case: `http.rs:667-671` calls `transport.send`, and if the outcome is `RefreshCredential`, calls `transport.refresh_credential()` (which drops the cached JWT, apns.rs:264-269) and sends exactly once more. A `Retry`-classified outcome is surfaced to the caller (the relay) as `503 {"status":"retry","retry_after_seconds":...}` — the gateway itself never retries a `Retry` outcome; retry-with-backoff is the **relay's** responsibility (`buzz-relay/src/push_runtime.rs:531-549`, `retry_or_fail`, exponential backoff `delay * 2^(attempt-1)` capped by `MAX_ATTEMPTS = 8`, outside this crate's scope). `retry_after_seconds` returned from APNs' `Retry-After` header is clamped to `1..=3600` before being passed through (apns.rs:255-259).

Disposition mapping from outcome to replay-fence handling (http.rs:672-681): `Retry`, `ConfigurationFault`, and `RefreshCredential` map to `DeliveryDisposition::Retryable` (releases the request-id fence so a fresh NIP-98 event can retry); `Accepted`, `InvalidEndpoint`, and `PermanentRequestFault` map to `Terminal` (fence stays burned forever). The one-use auth-event fence is **never** released by either disposition — only the request-id fence is conditionally released (authority.rs:170-176 trait doc comment: "Retain terminal request ids; release retryable request ids while always retaining the one-use auth event").

#### `strict_json` validation rule

`strict_json::from_slice` (strict_json.rs:76-81) parses one JSON document into an intermediate `serde_json::Value` via a custom `Visitor` that tracks seen object keys in a `HashSet` per object and returns an error on any duplicate (strict_json.rs:59-68, `visit_map`), then re-deserializes into the target type, and separately calls `deserializer.end()?` to reject trailing bytes after the document. This is used for every HTTP request body via handler call sites (e.g. http.rs:112, 157, 296) and for the grant's decrypted plaintext (grant.rs:92) — but notably **not** by `serde_json::to_string`/`to_vec` call sites that only serialize (those have no duplicate-key ambiguity to defend against). Standard `serde_json::Deserialize` silently accepts duplicate keys by keeping only the last occurrence; this module closes that ambiguity so `{"v":1,"v":2}` is a hard parse error rather than silently resolving to `v:2`. Test: `rejects_duplicate_keys_at_any_depth` (strict_json.rs:83-87).

#### Test coverage of business rules

- Config-level validation rules (keyring format, cross-keyring reuse, URL exactness, bounds on lifetime/quota fields) are covered by four `#[test]`s in `config.rs`: `keyrings_preserve_current_then_predecessor_order_and_are_independent` (config.rs:238), `malformed_security_configuration_fails_startup` (config.rs:248, table-driven over 7 cases), `cross_keyring_id_or_material_reuse_fails_startup` (config.rs:271), `malformed_or_empty_keyrings_fail_startup` (config.rs:283, table-driven over 6 cases).
- APNs response classification is covered by `response_classes_do_not_massacre_endpoints_on_provider_faults` (apns.rs:344-361), which asserts four representative classifications, not all documented status/reason combinations — e.g. no test exercises the `429 TooManyProviderTokenUpdates` → `ConfigurationFault` branch or the plain `403` (no reason) → `ConfigurationFault` branch specifically.
- The delegation/delivery admission transaction's concurrency guarantees are covered only by the six `#[ignore]`d Postgres tests in `postgres.rs` (see `push-gateway-data-model.md`), which do not run in CI.
- **No test in this crate drives the actual HTTP handlers** (`enroll`, `delegate`, `rotate_endpoint`, `revoke_delegation`, `revoke_installation`, `deliver`) end-to-end, so the ordering of validation checks documented above (e.g. "challenge consumed only after attestation succeeds") is asserted by reading the code, not by any test that would fail if the order were accidentally swapped. This is a gap: reordering step 5 before step 4 in `enroll` (consuming the challenge before verifying attestation) would let an attacker burn a victim's challenge without holding a valid key, and no test would catch that regression.
