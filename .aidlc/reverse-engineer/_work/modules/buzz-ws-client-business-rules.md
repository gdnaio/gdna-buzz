## Module: buzz-ws-client (`crates/buzz-ws-client`)

### Aspect: Business Rules

All rules below are read directly from `src/connection.rs` and `src/message.rs`.
The crate encodes protocol rules only (NIP-42 handshake, NIP-01 framing, timeouts);
it contains no domain/authorization logic.

---

### 1. NIP-42 authentication handshake — step by step

| # | Step | Code | file:line |
|---|---|---|---|
| 1 | Parse the URL string with `url::Url`; failure → `WsClientError::Url` | `url.parse::<url::Url>()` | `connection.rs:49`–`51` |
| 2 | Open the WebSocket with the **normalized** URL (`parsed.as_str()`, not the raw input) | `connect_async(parsed.as_str())` | `connection.rs:53`–`55` |
| 3 | Store the **original, un-normalized** `url` string as `relay_url` for later AUTH-event construction | `relay_url: url.to_string()` | `connection.rs:63` |
| 4 | Wait for the relay's `["AUTH", <challenge>]`, bounded by `AUTH_CHALLENGE_TIMEOUT_SECS` (20 s) | `wait_for_auth_challenge(Duration::from_secs(AUTH_CHALLENGE_TIMEOUT_SECS))` | `connection.rs:75`–`77`, const at `:17` |
| 5 | Challenge resolution order: (a) `pending_challenge` if already captured, (b) first buffered `RelayMessage::Auth`, (c) read from the socket | `pending_challenge.take()` / `buffer` scan / read loop | `connection.rs:161`–`163`, `:165`–`174`, `:178`–`214` |
| 6 | Reject a socket-read challenge longer than 1024 bytes → `AuthFailed("challenge exceeds 1024 bytes")` | `if challenge.len() > 1024` | `connection.rs:198`–`202` |
| 7 | Non-`AUTH` messages seen while waiting are pushed to `buffer` (never dropped) | `other => self.buffer.push_back(other)` | `connection.rs:205` |
| 8 | Build the AUTH event: parse `relay_url` into `nostr::RelayUrl`, call `EventBuilder::auth(challenge, url)`, append `auth_tag` when supplied, sign with `keys` | `build_auth_event` | `message.rs:151`–`167` (parse `:157`, builder `:158`, tag `:159`–`163`, sign `:165`) |
| 9 | Capture the AUTH event's hex id as the correlation key | `auth_event.id.to_hex()` | `connection.rs:80` |
| 10 | Send `["AUTH", <event>]` as a text frame | `send_raw(&json!(["AUTH", auth_event]))` | `connection.rs:82`, `:121`–`124` |
| 11 | Wait for `OK` whose `event_id` equals that hex id, bounded by `AUTH_OK_TIMEOUT_SECS` (20 s) | `wait_for_ok(&event_id, …)` | `connection.rs:84`–`86`, const at `:20` |
| 12 | If `ok.accepted == false` → `WsClientError::AuthFailed(ok.message)` (relay's reason string is propagated verbatim) | `if !ok.accepted` | `connection.rs:87`–`89` |
| 13 | On success, log at debug and return `Ok(())` — no state flag is recorded | `debug!("NIP-42 authentication successful")` | `connection.rs:91`–`92` |

`connect_authenticated` is exactly steps 1–13 composed
(`connection.rs:42`–`44`). `publish_event` runs steps 1–13, then `EVENT` + `OK`,
then close (`connection.rs:285`–`288`).

---

### 2. Rule catalogue

| Rule | What it enforces | Trigger | file:line |
|---|---|---|---|
| Challenge size cap | Challenge string must be ≤ 1024 bytes (byte length via `str::len`) | An `AUTH` message read from the socket inside `wait_for_auth_challenge` | `connection.rs:198`–`202` |
| Cap not applied on alternate paths | The same cap is **not** checked when the challenge comes from `pending_challenge` or from `buffer` | Challenge captured earlier by `recv_one`/`wait_for_ok` | `connection.rs:161`–`163`, `:170`–`171` (vs `:198`) |
| AUTH acceptance gate | Authentication only succeeds when the relay's `OK` for the AUTH event has `accepted == true` | `authenticate` after sending AUTH | `connection.rs:87`–`89` |
| OK correlation | An `OK` is only accepted when `ok.event_id` string-equals the locally computed hex event id; non-matching `OK`s are buffered | Every `OK` observed in `wait_for_ok` | `connection.rs:227`, `:254`, `:259` |
| Message preservation | Any relay message that is not the one being awaited is queued in `buffer` rather than discarded, so a later `next_event` still sees it | All wait loops | `connection.rs:205`, `:257`, `:259`; drain at `:108`, `:129` |
| Mid-flight AUTH capture | An `AUTH` challenge arriving while awaiting an `OK` is recorded in `pending_challenge` **and** buffered (so it is observable twice: as state and as a message) | `wait_for_ok` sees `RelayMessage::Auth` | `connection.rs:255`–`258` |
| Opportunistic AUTH capture | An `AUTH` returned to the caller via `next_event`/`recv_one` also updates `pending_challenge` | `recv_one` parses an `AUTH` | `connection.rs:143`–`145` |
| Keepalive | Inbound `Ping` must be answered with `Pong` carrying the same payload; the loop then continues waiting | Any `Ping` frame in any of the three loops | `connection.rs:148`–`150`, `:208`–`:210`, `:262`–`:264` |
| Close handling | An inbound `Close` frame, or a stream end (`None`), terminates the operation with `ConnectionClosed` | Any read loop | `connection.rs:137`, `:151`; `:190`, `:211`; `:247`, `:265` |
| Unknown frames ignored | Binary/Pong/raw-Frame variants are skipped without error | Any read loop | `connection.rs:152`, `:212`, `:266` |
| Strict message typing | The first array element must be a string and one of the six known types; otherwise `UnexpectedMessage` | `parse_relay_message` | `message.rs:58`–`61`, `:140`–`142` |
| Required positional fields | `EVENT` requires `arr[1]` (sub id, string) and `arr[2]` (event object); `OK`/`EOSE`/`CLOSED`/`AUTH` require `arr[1]` as string | `parse_relay_message` | `message.rs:65`–`74`, `:81`–`85`, `:99`–`103`, `:109`–`113`, `:133`–`137` |
| Lenient optional fields | `OK.accepted` defaults `false` if missing/not a bool; `OK.message`, `CLOSED.message`, `NOTICE.message` default to `""` | `parse_relay_message` | `message.rs:86`, `:87`–`91`, `:114`–`118`, `:125`–`129` |
| Relay-URL validity for AUTH | `relay_url` must parse as `nostr::RelayUrl`, else `WsClientError::Url` | `build_auth_event` | `message.rs:157` |
| Optional NIP-OA tag | When `auth_tag` is `Some`, exactly that one tag is cloned onto the AUTH event; when `None`, the builder is left untouched | `build_auth_event` | `message.rs:159`–`163` |

---

### 3. Timeout values and how they are applied

| Constant / parameter | Value | Applied to | file:line |
|---|---|---|---|
| `AUTH_CHALLENGE_TIMEOUT_SECS` | 20 s | Waiting for the `AUTH` challenge; expiry → `NoAuthChallenge` | const `connection.rs:17`; use `:76`; expiry `:184`, `:189` |
| `AUTH_OK_TIMEOUT_SECS` | 20 s | Waiting for the `OK` to the AUTH event; expiry → `Timeout` | const `connection.rs:20`; use `:85`; expiry `:241`, `:246` |
| `PUBLISH_OK_TIMEOUT_SECS` | 30 s | Waiting for the `OK` to a published `EVENT` | const `connection.rs:23`; use `:99` |
| `timeout_dur` (caller-supplied) | caller's choice | `next_event`/`recv_one` single-message read; expiry → `Timeout` | `connection.rs:104`, `:134`, `:136` |
| `timeout_secs` (caller-supplied) | caller's choice (`buzz-cli` passes `75`) | Wraps the whole connect+auth+publish+close sequence in `publish_event`; expiry → `Timeout` | `connection.rs:282`, `:284`, `:292`; caller `crates/buzz-cli/src/client.rs:1080` |

Deadline discipline: both multi-iteration waiters compute an absolute deadline once
(`connection.rs:176`, `:222`) and re-derive `remaining` each iteration with
`checked_duration_since(...).unwrap_or(Duration::ZERO)` (`:179`–`181`, `:236`–`238`),
returning the timeout error when `remaining.is_zero()` (`:183`–`185`, `:240`–`242`).
So the budget is for the whole wait, not per frame. `recv_one` by contrast applies
`timeout_dur` **per socket read** inside its loop (`connection.rs:133`–`134`), so a
stream of `Ping`s can extend its total wall time.

---

### 4. Retry / reconnect / backoff

**None.** Verified across all 314 lines of `connection.rs`: there is no retry loop,
no attempt counter, no backoff constant, no reconnect helper, and no
`connect_authenticated` re-invocation on failure. `publish_event` establishes one
fresh connection per call (`connection.rs:285`) and returns the first error it hits;
its `disconnect` result is explicitly discarded (`let _ = conn.disconnect().await;`
— `connection.rs:288`), so a failing close never masks the `OK` result. Recovery
after `ConnectionClosed` is entirely the caller's responsibility.

Re-authentication is also not automated: a mid-session `AUTH` challenge is only
recorded/buffered (`connection.rs:255`–`258`); the caller must call `authenticate`
again to consume it via `pending_challenge` (`connection.rs:161`).

---

### 5. Error classification

| Condition | Error produced | file:line |
|---|---|---|
| Bad URL string | `Url(String)` | `connection.rs:51`; `message.rs:157` |
| Upgrade/transport failure, read error, send error | `WebSocket(tungstenite::Error)` | `connection.rs:55`, `:138`, `:191`, `:248`; `#[from]` at `error.rs:8` |
| JSON encode of an outbound frame, or decode of an inbound frame | `Json(serde_json::Error)` | `connection.rs:122`; `message.rs:56`, `:70`–`74`; `#[from]` at `error.rs:12` |
| Malformed/unknown relay message | `UnexpectedMessage(String)` (carries the raw frame, or `"unknown message type: {other}"`) | `message.rs:61`, `:68`, `:73`, `:84`, `:102`, `:112`, `:136`, `:140`–`142` |
| No challenge within budget | `NoAuthChallenge` | `connection.rs:184`, `:189` |
| Oversized challenge, or relay `OK` with `accepted == false` for the AUTH event | `AuthFailed(String)` | `connection.rs:199`–`201`, `:88` |
| Awaited `OK` did not arrive in time; or whole-operation timeout | `Timeout` | `connection.rs:136`, `:241`, `:246`, `:292` |
| Stream ended or `Close` frame received | `ConnectionClosed` | `connection.rs:137`, `:151`, `:190`, `:211`, `:247`, `:265` |
| Event signing / builder failure | `EventBuilder(String)` | `message.rs:166`; `From` impl at `error.rs:47`–`51` |
| Relay rejected a published event | **Not raised by this crate.** `send_event` returns `Ok(OkResponse { accepted: false, … })` and leaves interpretation to the caller; `EventRejected` (`error.rs:40`) is never constructed here | `connection.rs:96`–`101` vs `error.rs:40` |

---

### 6. Challenge validation / replay considerations (as coded)

- The only validation performed on a relay challenge is the 1024-byte length cap
  (`connection.rs:198`). There is no charset check, no entropy check, no
  nonce-uniqueness tracking, and no rejection of a repeated challenge value.
- Freshness is delegated: the AUTH event's `created_at` is set by
  `nostr::EventBuilder::auth` (`message.rs:158`); this crate performs no timestamp
  computation or tolerance check of its own (verified — no `Timestamp`, `now()`, or
  clock-skew constant anywhere in the crate).
- `pending_challenge` holds only the most recent challenge — a second `AUTH` frame
  overwrites the first (`connection.rs:144`, `:256`), and the value is cleared only
  by `take()` in `wait_for_auth_challenge` (`connection.rs:161`).
