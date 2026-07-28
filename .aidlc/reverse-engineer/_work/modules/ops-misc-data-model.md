## Module: buzz-admin, sprig & countdown-bot (`crates/buzz-admin`, `crates/sprig`, `examples/countdown-bot`)
### Aspect: Data Model

This group has no persistent schema of its own. `buzz-admin` is a thin CLI shell over `buzz-db` types; `sprig` has no data model beyond an `argv[0]` string; `countdown-bot` holds a small in-memory `Config` struct and speaks NIP-01 JSON over the wire. All three are single files: `crates/buzz-admin/src/main.rs` (584 lines), `crates/sprig/src/main.rs` (53 lines), `examples/countdown-bot/src/main.rs` (437 lines).

#### buzz-admin: CLI request/response shapes

`buzz-admin` defines no structs of its own for data — it parses `clap` arguments straight into DB calls and prints the DB's return types. The `Cli`/`Command` parse tree is the only "model":

```rust
struct Cli { command: Command }                       // crates/buzz-admin/src/main.rs:36-39
enum Command {
    AddMember { pubkey: String, role: String },        // :48-58 (role default "member")
    RemoveMember { pubkey: String, role: Option<String> }, // :63-72
    ListMembers,                                        // :74
    GenerateKey,                                         // :76
    Migrate,                                             // :78
    ProductFeedback { command: ProductFeedbackCommand }, // :80-83
    ReconcileChannels { relay_key: Option<String> },     // :89-95
}
enum ProductFeedbackCommand {
    List { limit: u16 },                                 // :101-105, clamped 1..=1000, default 100
}
```
(`crates/buzz-admin/src/main.rs:34-105`)

Every subcommand ultimately reads or writes one of these `buzz-db` shapes:

| Shape | Defined at | Used by |
|---|---|---|
| `RelayMember { pubkey: String, role: String, added_by: Option<String>, created_at: DateTime<Utc>, updated_at: DateTime<Utc> }` | `crates/buzz-db/src/relay_members.rs:16-26` | `list-members` output (`main.rs:260-284`), the read side of `add-member`/`remove-member` |
| `RemoveResult` enum: `Removed \| NotFound \| IsOwner \| RoleMismatch` | `crates/buzz-db/src/relay_members.rs:180-193` | `remove-member`'s match arms, mapped to exit codes 0/2/3/4 (`main.rs:228-247`) |
| `ProductFeedbackRecord { id: Uuid, community_id: Uuid, event_id: String, submitter_pubkey: String, category: Option<String>, body: String, tags: serde_json::Value, event_created_at, received_at }` | `crates/buzz-db/src/product_feedback.rs:31-49` | `product-feedback list` output, serialized directly with `serde_json::to_string_pretty` (`main.rs:253-257`) |
| `TenantContext { community: CommunityId, host: String }` | `crates/buzz-core/src/tenant.rs:68-71` | every subcommand except `generate-key`, via `resolve_admin_tenant` (`main.rs:439-458`) |

The underlying Postgres schema `buzz-admin` reads/writes: `relay_members` — `PRIMARY KEY (community_id, pubkey)`, `role TEXT NOT NULL CHECK (role IN ('owner','admin','member'))` (`migrations/0001_initial_schema.sql:574-582`) — and `product_feedback` — `PRIMARY KEY id`, `UNIQUE(event_id)`, `category CHECK (category IN ('bug','praise','needs-work'))` (`migrations/0017_product_feedback.sql:5-16`). `buzz-admin` never inserts into `product_feedback`; only `list_product_feedback` (read-only) is called (`main.rs:253-257`; write side `product_feedback::insert` is called only from `crates/buzz-db/src/lib.rs:3037`, itself reachable only from the relay's event pipeline, not from this CLI — confirmed by grep: no `insert_product_feedback` call anywhere under `crates/buzz-admin/`).

`reconcile-channels` additionally builds three Nostr event shapes in memory before writing them via `replace_addressable_event` (`crates/buzz-admin/src/main.rs:493-576`):

| Kind | Constant | Tags built |
|---|---|---|
| 39000 | literal `39000` (`main.rs:511`) — **not** `KIND_NIP29_GROUP_METADATA` even though that constant exists (`crates/buzz-core/src/kind.rs:362`) | `d`, `name`, optional `about`, `private`/`public`, optional `hidden` (if `channel_type == "dm"`), `closed`, `t` (`main.rs:504-520`) |
| 39001 | `KIND_NIP29_GROUP_ADMINS` (`buzz_core::kind::KIND_NIP29_GROUP_ADMINS`, = 39001, `crates/buzz-core/src/kind.rs:364`) | `d`, one `p`/pubkey/role tag per owner-or-admin member (`main.rs:542-555`) |
| 39002 | literal `39002` (`main.rs:562`) — again not the named constant `KIND_NIP29_GROUP_MEMBERS` (`crates/buzz-core/src/kind.rs:366`) | `d`, one `p`/pubkey/""/role tag per member (`main.rs:558-570`) |

This mixed literal/constant usage inside a single 20-line block (`main.rs:496-576`) is inconsistent — see `ops-misc-conventions.md` and `ops-misc-debt.md`.

#### buzz-admin: the NIP-43 membership-list event

`publish_membership_list_with_bump` (`crates/buzz-admin/src/main.rs:315-385`) builds a kind:13534 (`KIND_NIP43_MEMBERSHIP_LIST`, `crates/buzz-core/src/kind.rs:338`) event with:
- an NIP-70 `["-"]` protected-event tag (`main.rs:353`)
- one `["member", pubkey, role]` tag per current member (`main.rs:354-358`)
- `custom_created_at = max(now, newest_existing_13534_created_at + 1)` (`main.rs:340-345`), computed by first reading the newest existing kind:13534 via `db.get_latest_global_replaceable` (`main.rs:333-339`, delegating to `crates/buzz-db/src/lib.rs:1128`)

This is a **hand-rolled duplicate** of the relay's own `Db::publish_nip43_membership_locked` (`crates/buzz-db/src/lib.rs:3488-3560`), which builds the identical tag shape (`["-"]` + `["member", pubkey, role]`, `lib.rs:3517-3527`) but inside a Postgres advisory-locked transaction that serializes the read-build-write cycle. `buzz-admin`'s version reads and writes as two unlocked steps, which is the direct cause of the "same-second domination" race the module doc-comment (`main.rs:14-19`) and `NOSTR.md:308-315` both describe. See `ops-misc-debt.md` for the duplication finding and `ops-misc-security.md` for the race-condition risk.

#### sprig: dispatch is a single lowercase string match, nothing more

`sprig`'s entire "data model" is one `String` derived from `argv[0]`'s file name:

```rust
let cmd = std::path::Path::new(&argv0)
    .file_name()
    .and_then(|n| n.to_str())
    .unwrap_or("")
    .to_ascii_lowercase();
```
(`crates/sprig/src/main.rs:9-14`)

No struct, no enum, no config file, no parsed flags feed the dispatch decision except this one lowercased string, matched against four literal cases (`"buzz-acp"`, `"buzz-agent"`, `"sprig"`, and a wildcard `_`) at `crates/sprig/src/main.rs:16-42`. For the `"sprig"` case only, a second argument (`std::env::args().nth(1)`) is inspected for `-V`/`--version`/`-h`/`--help` (`main.rs:19-37`) — still plain string matching, no `clap` or structured parser.

#### countdown-bot: in-memory Config and NIP-01 wire shapes

```rust
struct Config {
    relay_url: String,
    channel_id: String,
    bot_keys: Keys,
    owner_auth_tag: Option<Tag>,
}
```
(`examples/countdown-bot/src/main.rs:75-80`), populated once by `Config::from_env()` (`main.rs:83-121`) and never mutated afterward — no database, no persisted state at all. `channel_id` is stored as a raw `String` (not parsed to `Uuid` at load time); it is parsed lazily with `.parse()?` only at the one call site that needs a `Uuid` (`buzz_sdk::builders::build_message(config.channel_id.parse()?, ...)`, `main.rs:236`).

Wire-level data the bot produces/consumes (all plain NIP-01 JSON arrays, no typed wrapper beyond the `nostr` crate's `Event`/`Filter`/`Tag`):

| Direction | Shape | Built/parsed at |
|---|---|---|
| bot → relay | `["AUTH", <kind:22242 event>]` | `build_auth_event` (`main.rs:139-153`), sent at `main.rs:132-133` |
| bot → relay | `["EVENT", <kind:0 profile event>]` | `buzz_sdk::builders::build_profile(...)` (`main.rs:157-162`) |
| bot → relay | `["EVENT", <kind:9000 self-add event>]` | inline `EventBuilder` (`main.rs:174-179`) — `h`, `p`, `role=bot` tags |
| bot → relay | `["REQ", "countdown-bot", <filter>]` | `subscribe_to_channel` (`main.rs:191-197`) — `kind=9`, `#h=<channel_id>` |
| relay → bot | `["EVENT", sub_id, <event>]`, `["EOSE", ...]`, `["NOTICE"/"CLOSED", ...]` | `handle_relay_text` (`main.rs:199-217`), dispatched by `value.get(0)` |
| bot → relay | `["EVENT", <kind:9 reply event>]` | `buzz_sdk::builders::build_message(...)` (`main.rs:230-238`) |

The bot's own reply payload has no structured schema — it is a plain-text string built by `countdown_reply`/`fib_reply` (`main.rs:288-309`), e.g. `"5 4 3 2 1 🚀"` or an error string `"Please use a number from 1 to 100."` (`main.rs:311-320`), joined with `" "`.

The owner-attestation credential the bot optionally carries is a NIP-OA `auth` tag, `["auth", "<owner-pubkey-hex>", "<conditions>", "<sig-hex>"]` (four elements per `docs/nips/NIP-OA.md:34-40`), constructed via `buzz_sdk::nip_oa::compute_auth_tag` / parsed via `parse_auth_tag` / verified via `verify_auth_tag` (`crates/buzz-sdk/src/nip_oa.rs:146,179,252`; call sites `examples/countdown-bot/src/main.rs:98,104,107`).

#### Test coverage of the data model

`countdown-bot` has a `#[cfg(test)] mod tests` block (`examples/countdown-bot/src/main.rs:392-436`, verified by `grep -n "cfg(test)" examples/countdown-bot/src/main.rs` → line 392) covering exactly the three pure string-transform functions `command_reply`, `fibonacci_countdown`'s output via `command_reply("!fib ...")`, and `mention_command_reply` — i.e. the reply-text data shape, not the `Config`/wire types. `buzz-admin` and `sprig` have **zero** test modules: `grep -rn "#\[test\]|#\[cfg(test)\]|#\[tokio::test\]" crates/buzz-admin/ crates/sprig/` returns no matches (confirmed twice, once per crate). There is consequently no test that exercises the `Command`/`ProductFeedbackCommand` parse tree, the `RemoveResult` → exit-code mapping, or any of the three hand-built Nostr event shapes in `reconcile_channels`.
