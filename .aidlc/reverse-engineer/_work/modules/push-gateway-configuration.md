## Module: buzz-push-gateway (`crates/buzz-push-gateway`)
### Aspect: Configuration

#### Overview

All configuration is environment-variable driven, parsed once at startup by `Config::from_env()` (config.rs:78-80, delegating to `Config::from_map(&std::env::vars().collect())`, config.rs:81). There are no CLI flags for configuration (the only recognized argv value is the literal `--migrate-only` mode switch, main.rs:26, which is unrelated to `Config`). Every field is validated eagerly and fails startup with a typed `ConfigError::Missing`/`ConfigError::Invalid` on any problem — there is no silent fallback-to-default for a present-but-malformed value anywhere in this file; unset-vs-empty-string is also distinguished for optional bounded integers (see `bounded_positive` below).

#### Environment variables read by `Config::from_map`

| Variable | Type / format | Default | Read site | Documented? |
|---|---|---|---|---|
| `BUZZ_PUSH_GRANT_KEYS` | `id:base64(32 bytes)[,id:base64...]`, current key first | none — required | config.rs:91, via `parse_keyring` (config.rs:48-76) | Yes — `docs/push-gateway-deployment.md:26`, Helm `values.yaml` secret ref |
| `BUZZ_PUSH_TOKEN_KEYS` | same format; must not overlap `BUZZ_PUSH_GRANT_KEYS` by id or key bytes | none — required | config.rs:92-102 | Yes — `docs/push-gateway-deployment.md:27` |
| `BUZZ_PUSH_PUBLIC_DELIVERY_URL` | exact URL, must equal scheme `https`, host `push.buzz.xyz`, no port, path exactly `/v1/deliveries/apns`, no query/fragment/userinfo/password | none — required | config.rs:103-116 | Yes — `docs/push-gateway-deployment.md:16` |
| `BUZZ_PUSH_MAX_GRANT_LIFETIME_SECONDS` | integer, `1..=31_536_000` | none — required | config.rs:117-121 | Yes — `docs/push-gateway-deployment.md:17` |
| `BUZZ_PUSH_MAX_INSTALLATION_LIFETIME_SECONDS` | integer, `1..=31_536_000` | `7776000` (90 days) | config.rs:122-131 | Yes — `docs/push-gateway-deployment.md:18` ("default 90 days, max one year") — matches exactly |
| `BUZZ_PUSH_ENDPOINT_QUOTA_WINDOW_SECONDS` | integer, `1..=86_400` | `10` | config.rs:132-142, via `bounded_positive` | Yes — `docs/push-gateway-deployment.md:29` |
| `BUZZ_PUSH_ENDPOINT_QUOTA_MAX_DELIVERIES` | integer, `1..=10_000` | `10` | config.rs:143-144 | Yes — `docs/push-gateway-deployment.md:29` |
| `BUZZ_PUSH_ENABLED_PROFILES` | comma-separated, each ∈ `{buzz-ios-production, buzz-ios-sandbox}`, non-empty result | none — required | config.rs:145-154 | Yes — `docs/push-gateway-deployment.md:19` |
| `BUZZ_PUSH_BIND_ADDR` | `SocketAddr` | `0.0.0.0:8080` | config.rs:156-161 | Yes — `docs/push-gateway-deployment.md:7`, Helm `deployment.yaml:30` |
| `BUZZ_PUSH_HEALTH_ADDR` | `SocketAddr` | `0.0.0.0:8081` | config.rs:162-167 | Yes — `docs/push-gateway-deployment.md:8`, Helm `deployment.yaml:31` |
| `DATABASE_URL` | Postgres connection string | none — required | config.rs:174 | Yes — `docs/push-gateway-deployment.md:15` (also the generic repo-wide var; see below) |
| `BUZZ_PUSH_APP_ATTEST_APP_ID` | non-empty string, `TEAMID.bundle-id` | none — required | config.rs:175 | Yes — `docs/push-gateway-deployment.md:20` |
| `BUZZ_PUSH_APP_ATTEST_ROOT_CERT_PATH` | filesystem path | none — required | config.rs:176 | Yes — `docs/push-gateway-deployment.md:21` |
| `BUZZ_PUSH_APNS_KEY_PATH` | filesystem path | none — required | config.rs:180 | Yes — `docs/push-gateway-deployment.md:22` |
| `BUZZ_PUSH_APNS_KEY_ID` | non-empty string | none — required | config.rs:181 | Yes — `docs/push-gateway-deployment.md:23` |
| `BUZZ_PUSH_APNS_TEAM_ID` | non-empty string | none — required | config.rs:182 | Yes — `docs/push-gateway-deployment.md:24` |
| `BUZZ_PUSH_APNS_TOPIC` | non-empty string | none — required | config.rs:183 | Yes — `docs/push-gateway-deployment.md:25` |
| `BUZZ_PUSH_RUNTIME_DATABASE_ROLE` | PostgreSQL identifier | none — required (`--migrate-only` path only) | main.rs:32, validated postgres.rs:24-30 | Partially — named in the Helm chart (`migration-job.yaml:29-30`, `values.yaml`'s `migration.runtimeDatabaseRole`) and in `docs/push-gateway-deployment.md`'s prose ("`migration.runtimeDatabaseRole` names an existing LOGIN role... the default is `buzz_push_gateway_runtime`"), but the literal env-var name `BUZZ_PUSH_RUNTIME_DATABASE_ROLE` never appears in `docs/push-gateway-deployment.md`'s own required-configuration table — only the Helm value name is documented there |

All required-var lookups go through the shared closure `req(e, k)` (config.rs:83-89), which treats an **empty string** the same as **unset** (`.filter(|v| !v.is_empty())`) — an operator who sets `BUZZ_PUSH_APNS_TOPIC=""` gets the same `ConfigError::Missing` as leaving it unset entirely, not a validation error distinguishing the two cases.

#### `bounded_positive` unset-vs-empty distinction

The two quota variables use a stricter helper (config.rs:132-140) than plain `req`: if the key is **absent from the map**, the default is used; but if the key is **present with an empty or non-numeric string**, parsing fails and the default is *not* silently substituted (`.or((!e.contains_key(key)).then_some(default))` only fires when the key is truly absent, config.rs:136). So `BUZZ_PUSH_ENDPOINT_QUOTA_WINDOW_SECONDS` unset → `10`; `BUZZ_PUSH_ENDPOINT_QUOTA_WINDOW_SECONDS=""` or `="abc"` → hard startup error. This is a meaningfully different (safer) behavior than the plain `req`-based required fields' unset/empty conflation described above, and it is not documented as a distinction anywhere.

#### `BUZZ_PUSH_PUBLIC_DELIVERY_URL` exactness

Validated field-by-field, not just "must be a valid URL" (config.rs:107-116): scheme must be exactly `https`, host must be exactly `push.buzz.xyz` (hard-coded, not configurable — a staging/alternate deployment cannot use a different hostname without a code change), no explicit port, path exactly `/v1/deliveries/apns`, and no query string, fragment, username, or password. This means the value is effectively a fixed constant in practice — the config format allows specifying it, but only one string value will ever pass validation. `docs/push-gateway-deployment.md:16` documents it as "Exact externally signed URL, normally `https://push.buzz.xyz/v1/deliveries/apns`," which undersells how rigid the check actually is (the doc's "normally" wording could be read as implying other values are viable, when the code makes `push.buzz.xyz` the only viable host).

#### `DATABASE_URL` documentation ambiguity

`DATABASE_URL` is the same variable name used by the main relay and documented at the top level in `.env.example:20` for local dev (`postgres://buzz:buzz_dev@localhost:5432/buzz`, shared Postgres for the whole docker-compose stack). This crate's own `docs/push-gateway-deployment.md:15` explicitly warns the value used here "MUST name a dedicated gateway database, not the relay database" — but because both services read the identically-named `DATABASE_URL` variable, an operator copying `.env.example`'s value verbatim into this service's environment would silently point it at the relay's database (see `push-gateway-data-model.md`'s duplicate-migration finding for why that is unsafe, not just discouraged).

#### App Attest root cert pin — configuration vs. hard-coded constant

`BUZZ_PUSH_APP_ATTEST_ROOT_CERT_PATH` is configurable (the file path), but its *content* is not: `AppAttestVerifier::new` (app_attest.rs:37-45) rejects any file whose SHA-256 does not equal the hard-coded `APPLE_APP_ATTEST_ROOT_PEM_SHA256` constant (app_attest.rs:11-14). This is documented explicitly in `docs/push-gateway-deployment.md`'s "Secret and key rotation rules" section ("startup will reject any byte mismatch... Treat an Apple root rotation as a reviewed code/config rollout, not an unpinned mount replacement") — code and docs agree here, no drift found.

#### Parsed-but-conditionally-unused config

`Config::apns_topic`, `apns_key_id`, `apns_team_id`, `apns_key_path` are all read unconditionally at startup (they are required fields with no `Option<>` wrapper) even though `main.rs:38-43` immediately consumes them to build the one `ApnsTransport` — there is no code path where a fully-parsed `Config` exists without these being used, so there is no "parsed but never read" config field in this crate. I checked every field of the `Config` struct (config.rs:16-35) against its usage in `main.rs` and found each one consumed exactly once at startup (either directly or via `AppState` construction, main.rs:87-102) — no dead configuration field.

#### CLI-adjacent configuration: `--migrate-only`

Not a flag with a value — it is a positional-argument mode switch checked via exact string equality (`std::env::args().nth(1).as_deref() == Some("--migrate-only")`, main.rs:26). When active, it reads only `DATABASE_URL` and `BUZZ_PUSH_RUNTIME_DATABASE_ROLE` (main.rs:27-32) — none of the other `Config` fields (APNs credentials, App Attest, keyrings, quotas) are read or validated in this mode, so a misconfigured `BUZZ_PUSH_GRANT_KEYS` (for example) would not be caught by a `--migrate-only` invocation even though the chart runs that invocation as a pre-flight Job before the main deployment rolls out.

#### Helm chart configuration surface (cross-check)

The chart (`deploy/charts/buzz-push-gateway/values.yaml`) sets `BUZZ_PUSH_BIND_ADDR`/`BUZZ_PUSH_HEALTH_ADDR` to the same literal defaults the code already falls back to (`deployment.yaml:30-31` vs. config.rs:159, 165) — redundant but not contradictory. It wires `publicDeliveryUrl`, `maxGrantLifetimeSeconds`, `enabledProfiles`, `appAttestAppId` as top-level Helm values (`values.yaml:19-23`) but leaves `BUZZ_PUSH_MAX_INSTALLATION_LIFETIME_SECONDS` and both `BUZZ_PUSH_ENDPOINT_QUOTA_*` variables entirely unset in the chart (grep of `deployment.yaml`'s `env:` list confirms only six literal + six secret-sourced variables are set, matching exactly the required-var table in `docs/push-gateway-deployment.md`, not the optional-quota variables) — so a chart-based deployment silently relies on this crate's in-code defaults (`7776000`, `10`, `10`) for those three unless a value override is added; the chart's own `values.yaml` has no placeholder or comment calling out that these three are configurable at all.

#### Test coverage

Four `#[test]`s in `config.rs` directly exercise the validation rules: `keyrings_preserve_current_then_predecessor_order_and_are_independent` (config.rs:238), `malformed_security_configuration_fails_startup` (config.rs:248, table-driven over `BUZZ_PUSH_PUBLIC_DELIVERY_URL` http/wrong-host cases, empty `BUZZ_PUSH_APP_ATTEST_APP_ID`, unknown profile, and two `BUZZ_PUSH_MAX_GRANT_LIFETIME_SECONDS`/`BUZZ_PUSH_MAX_INSTALLATION_LIFETIME_SECONDS` boundary values), `cross_keyring_id_or_material_reuse_fails_startup` (config.rs:271), `malformed_or_empty_keyrings_fail_startup` (config.rs:283). No test specifically exercises the `bounded_positive` unset-vs-empty-string distinction described above for the quota variables, and no test exercises `BUZZ_PUSH_BIND_ADDR`/`BUZZ_PUSH_HEALTH_ADDR` default or override behavior, or the `--migrate-only` argv-parsing branch in `main.rs` (which has no test coverage at all — `main.rs` contains no `#[cfg(test)]` module).
