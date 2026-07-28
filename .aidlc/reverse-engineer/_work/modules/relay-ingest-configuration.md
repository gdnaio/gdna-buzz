## Module: buzz-relay — event ingest & side effects (`crates/buzz-relay/src/handlers`)
### Aspect: Configuration

---

### 1. Config fields consumed (via `state.config`)

Exactly **four** fields of `crate::config::Config` are read across all 8 911 lines.

| Field | Env var | Default | Read at | Effect in this module |
|---|---|---|---|---|
| `require_relay_membership: bool` | `BUZZ_REQUIRE_RELAY_MEMBERSHIP` (`"true"` or `"1"` → true) | **`false`** (`config.rs:482-484`; asserted `config.rs:954-956`) | `ingest.rs:1821` | Gates kind 28936 (NIP-43 leave request). When `false` — the default — every 28936 is rejected with `invalid: relay membership is not enabled`. So **self-service relay leave is off out of the box.** |
| `ephemeral_ttl_override: Option<i32>` | `BUZZ_EPHEMERAL_TTL_OVERRIDE` (parsed `i32`, filtered `> 0`) | `None` (`config.rs:691-695`) | `ingest.rs:2099`, `side_effects.rs:1681` | Passed to `resolve_ttl` (`handlers/mod.rs:42-62`). When set, it **clobbers** any client-supplied `ttl` tag on kind 9007 rather than clamping it. Config emits a startup `warn!` when set (`config.rs:696-700`). Note: the 9002 `ttl` update path (`side_effects.rs:1449-1481`) does **not** apply the override — a client can change a channel's TTL post-creation to escape it. |
| `relay_url: String` | `RELAY_URL` (no `BUZZ_` prefix) | `"ws://localhost:3000"` (`config.rs:427-428`) | `ingest.rs:2212` | Input to `media_base_url_for_tenant(&state.config.relay_url, tenant.host())`, which yields the per-tenant `media_base_url` that `validate_imeta_tags` accepts absolute `imeta` URLs against (`imeta.rs:375-386`). A misconfigured `RELAY_URL` therefore rejects every absolute-form `imeta` URL while relative `/media/...` still works. |
| `git_max_repos_per_pubkey: u32` | `BUZZ_GIT_MAX_REPOS_PER_PUBKEY` | **`100`** (`config.rs:731-734`) | `side_effects.rs:2487` | Per-pubkey repo quota checked **before** claiming a name on kind 30617. Exceeded → `repo limit exceeded: {owned} >= {limit}`. Checked only on the fresh-claim path; a same-owner re-announce never grows the count (`side_effects.rs:2481-2490`). |

---

### 2. Env vars read directly (bypassing `Config`)

| Env var | Default | Read at | Effect |
|---|---|---|---|
| `SPROUT_MAX_NOT_BEFORE_DELTA` | **`31_536_000`** (1 year) | `ingest.rs:1299-1302` | Upper bound on how far in the future a kind:30300 `not_before` may be. Exceeded → `not_before too far in future`. |

⚠ Three problems with this one:
1. It is the **only** env var in the group read via `std::env::var` at request time instead
   of through `Config`. It is re-parsed on **every** kind:30300 ingest — no caching.
2. It uses the legacy `SPROUT_` prefix while every other var in this group uses `BUZZ_`
   (or bare `RELAY_URL`).
3. It is read in two places that must agree: here and `nip11.rs:97`, which advertises the
   value in the NIP-11 document. Both default to `31_536_000` independently; there is no
   shared constant. A change to one silently desynchronises the advertised limit from the
   enforced limit. The code comment at `ingest.rs:1298` asserts the coupling ("the same
   SPROUT_MAX_NOT_BEFORE_DELTA env var is advertised in NIP-11") without enforcing it.

---

### 3. Compile-time constants (effectively configuration, not tunable)

| Constant | Value | Declared | Governs |
|---|---|---|---|
| `MAX_TIMESTAMP_DRIFT_SECS` | `900` (±15 min) | `ingest.rs:1480` | Global created_at fence, all kinds |
| `MAX_EVENT_CONTENT_BYTES` | `256 * 1024` | `ingest.rs:1489` | Global content size cap |
| `MAX_REACTION_EMOJI_CHARS` | `64` | `ingest.rs:2283` | kind:7 emoji length (Unicode chars, not bytes) |
| `MAX_NOT_BEFORE` | `9_007_199_254_740_991` (`Number.MAX_SAFE_INTEGER`) | `ingest.rs:1224` | kind:30300 `not_before` ceiling |
| diff content cap | `61_440` (60 KiB) | `ingest.rs:898` | kind:40008 |
| thread depth cap | `100` | `ingest.rs:647` | NIP-10 nesting |
| 28936 freshness window | `120` s | `ingest.rs:1834` | NIP-43 leave request |
| DM participant cap | `8` others / `9` total | `command_executor.rs:322`, `:544` | kinds 41010, 41011 |
| NIP-44 v2 minimum decoded length | `99` bytes | `ingest.rs:1130` | kinds 30174, 44200 |
| `D_TAG_MAX_LEN` | `1024` | `buzz-db/src/event.rs:124`, used `ingest.rs:2378`, `command_executor.rs:141` | all NIP-33 kinds |
| persona slug max | `64` chars | `ingest.rs:1055` | kind 30175 |
| repo id max | `64` chars | `side_effects.rs:2393` | kind 30617 |
| imeta `filename` max | `255` chars | `imeta.rs:143` | all imeta |
| imeta MIME max | `255` chars | `imeta.rs:345` | all imeta |
| imeta duration tolerance | `0.1` s | `imeta.rs:272` | video imeta |
| `DEFAULT_HEAD` | `"refs/heads/main"` | `side_effects.rs:2606` | seeded git manifest + initial kind:30618 |
| audit channel capacity | `1000` | `state.rs:654`, referenced `handlers/event.rs:540` | audit backpressure on the ingest path |

None of these 17 are configurable. The 256 KiB content cap and the ±15 min drift window are
the two most likely to need per-deployment tuning and are hard-coded.

---

### 4. Metrics emitted (observability configuration surface)

| Metric | Type | Labels | Emitted at |
|---|---|---|---|
| `buzz_events_stored_total` | counter | `kind` (bounded), `author_type` (`agent`/`human`) | `ingest.rs:1396-1401` — at the shared seam so WS and HTTP count identically |
| `buzz_events_rejected_total` | counter | `transport` (`ws`/`http`), `reason` (`auth`/`invalid`/`scope`/`error`) | `reject_with_transport` `ingest.rs:157-163`; called from the transports, not from ingest |
| `buzz_channels_created_total` | counter | `community` (host), `type` | `ingest.rs:2124-2129`; `side_effects.rs:1704`, `:1731`; `command_executor.rs:373`, `:527` |
| `buzz_users_created_total` | counter | `community` (host) | `side_effects.rs:1097-1101`, `:1141-1145`; `command_executor.rs:51-56` |
| `buzz_nip43_membership_publications_total` | counter | `result` (`attempted`/`succeeded`/`failed`) | `side_effects.rs:2828`, `:2834-2838` |
| `buzz_nip43_membership_publication_seconds` | histogram | — | `side_effects.rs:2832` |
| `buzz_nip43_membership_reconciliations_total` | counter | — | `side_effects.rs:2818` |
| `buzz_nip43_membership_reconciliation_failures_total` | counter | — | `side_effects.rs:2811` |

Reached transitively via `dispatch_persistent_event`:
`buzz_post_commit_dispatch_scheduled_total` (`handlers/event.rs:348`),
`buzz_audit_send_errors_total` (`handlers/event.rs:576`),
`buzz_workflow_runs_total` with `trigger` + `community` (`handlers/event.rs:524-529`).

⚠ Gaps: there is **no** counter for side-effect failures, despite 32 `warn!` sites in
`side_effects.rs` that each represent a silently-degraded write (BR-IN-69). There is also no
histogram for ingest latency in this module, and `buzz_channels_created_total` is emitted
from five sites whose non-double-counting relies on a prose argument
(`side_effects.rs:1670-1679`) rather than a single emit point.

`author_type_label` (`ingest.rs:1328-1361`) performs a DB lookup
(`get_agent_channel_policy`) on cache miss purely to compute a metric label, cached in
`state.author_type_cache` (`state.rs:613`, `:788`). It is explicitly fail-open — unknown
pubkeys and lookup errors count as `"human"` so "the label must not add a failure path to
ingest" (`ingest.rs:1322-1324`). That cache is never invalidated, so an agent registered
after its first event is mislabelled `human` for the cache's lifetime.

---

### 5. Runtime-state dependencies (not env config, but deployment-shaping)

| `AppState` field | Used at | Note |
|---|---|---|
| `db` | everywhere | Postgres, required |
| `pubsub` | `side_effects.rs:81`, `:701`, `:794`, `:873` | Redis; publish failures are `warn!`-only |
| `media_storage` | `ingest.rs:2215` | S3/Blossom; required for any event with `imeta` |
| `git_store` | `side_effects.rs:2642-2726` | object store; required for kind 30617 |
| `workflow_engine` | `command_executor.rs:783`, `:900`, `:1076`; `side_effects.rs:2017`, `:2038` | in-process |
| `relay_keypair` | 20+ sites in `side_effects.rs` | signs every relay-minted event and is the sentinel in `effective_message_author` (`ingest.rs:731`) |
| `conn_manager`, `sub_registry` | `side_effects.rs:44-140` | subscription eviction |
| `tracer` | `ingest.rs:1383` | `NoopTracer` in production (`state.rs:798`) |
| `author_type_cache` | `ingest.rs:1337-1353` | metric labelling only |
| `audit_tx` | via `handlers/event.rs:539` | `None` disables auditing entirely (`main.rs:330` decides) |

---

### 6. `.env.example` coverage check

| Var this module reads | In `.env.example`? | In `Config`? |
|---|---|---|
| `RELAY_URL` | ✅ `.env.example:49` | ✅ `config.rs:68`, `:427-428` |
| `BUZZ_EPHEMERAL_TTL_OVERRIDE` | ✅ commented out, `.env.example:100` | ✅ `config.rs:209`, `:691-695` |
| `BUZZ_REQUIRE_RELAY_MEMBERSHIP` | ❌ **absent** | ✅ `config.rs:112`, `:482-484` |
| `BUZZ_GIT_MAX_REPOS_PER_PUBKEY` | ❌ **absent** | ✅ `config.rs:234`, `:731-734` |
| `SPROUT_MAX_NOT_BEFORE_DELTA` | ❌ **absent** | ❌ **absent** — only two independent `std::env::var` reads |

Two actionable rows:
- `BUZZ_REQUIRE_RELAY_MEMBERSHIP` gates a whole feature (self-service relay leave, kind
  28936) and defaults off, yet is not discoverable from `.env.example`.
- `SPROUT_MAX_NOT_BEFORE_DELTA` changes protocol-visible behaviour (the NIP-11-advertised
  reminder horizon) and exists only as two independent `std::env::var` reads
  (`ingest.rs:1299`, `nip11.rs:97`). It never appears in the config struct, so it is absent
  from any config dump, validation, or startup log — and the enforced value can silently
  diverge from the advertised one.
