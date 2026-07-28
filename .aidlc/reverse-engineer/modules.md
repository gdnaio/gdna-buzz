<!-- Analyzed: 2026-07-25T01:12:08Z | Scope: full project -->
# Module Inventory

## Summary

30 top-level modules identified: 26 Rust workspace crates (+ 1 example crate) plus
3 client applications (desktop, web, mobile). Modularity is **high and deliberate** —
the backend is a layered crate graph rooted at a zero-I/O `buzz-core`, with services
(db, auth, pubsub, search, audit, media, workflow) isolated from one another and
composed only by `buzz-relay`. Coupling is low between services and concentrated at
the relay, which is the single integrator (and the largest crate at ~60K LOC). The
desktop frontend (~282K LOC) is the largest single module and is internally organized
into 28 feature packages under `desktop/src/features/`. No circular crate dependencies
are possible (Rust rejects cyclic crate graphs) and none were detected within modules.

## Modules

### Backend — Foundation & Shared (layer 0)

| Module | Purpose | Files | LOC | Tests | Dependencies |
|---|---|---|---|---|---|
| buzz-core | Zero-I/O types, event verification, NIP-01 filter matching, kind registry (~81 kinds) | 20 | 7,548 | Unit | — |
| buzz-sdk | Typed Nostr event builders | 5 | 5,380 | Unit | buzz-core |
| buzz-persona | Agent persona packs | 9 | 5,197 | — | — |
| buzz-ws-client | Shared NIP-42 WebSocket client (connect, auth, publish) | 4 | 541 | — | — |

### Backend — Services (layer 1, composed by the relay)

| Module | Purpose | Files | LOC | Tests | Dependencies |
|---|---|---|---|---|---|
| buzz-db | Postgres event store + data access (events, channels, workflows, feed, partitions) | 22 | 25,580 | Unit + integration | buzz-core |
| buzz-auth | NIP-42/98 auth, scopes, rate-limit traits | 8 | 1,877 | Unit | buzz-core |
| buzz-pubsub | Redis pub/sub fan-out, presence, typing | 10 | 2,118 | Unit | buzz-core, buzz-auth |
| buzz-search | Postgres FTS query layer | 4 | 1,863 | Unit | buzz-core |
| buzz-audit | Hash-chain tamper-evident audit log | 6 | 1,114 | Unit | buzz-core |
| buzz-media | Blossom/S3 media storage | 12 | 6,027 | Unit | buzz-core |
| buzz-workflow | YAML-as-code automation engine (evalexpr) | 5 | 4,411 | Unit | buzz-core, buzz-db |

### Backend — Relay & Mesh (layer 2)

| Module | Purpose | Files | LOC | Tests | Dependencies |
|---|---|---|---|---|---|
| buzz-relay | Axum WS+REST server; huddle audio; git HTTP; ties all subsystems together | 76 | 60,056 | Unit + integration | core, conformance, db, auth, pubsub, audit, search, relay-mesh, sdk, workflow, media |
| buzz-relay-mesh | Inter-relay mesh transport (iroh) | 9 | 3,139 | Unit | — |
| buzz-conformance | Multi-tenant trace replay checker + golden fixtures | 5 | 1,749 | Unit + fixtures | — |

### Backend — Agent Surface (layer 2/3)

| Module | Purpose | Files | LOC | Tests | Dependencies |
|---|---|---|---|---|---|
| buzz-acp | ACP harness bridging relay events ↔ AI agents | 14 | 33,155 | Unit | buzz-core, buzz-sdk, buzz-persona |
| buzz-agent | Minimal ACP-compliant agent (non-streaming) | 20 | 17,864 | Unit | — (external rmcp/acp) |
| buzz-dev-mcp | Developer MCP server — shell + file-edit tools | 11 | 5,205 | Unit | buzz-cli, git-credential-nostr, git-sign-nostr, buzz-core |
| buzz-cli | Agent-first CLI (JSON in/out) | 27 | 15,211 | Unit | buzz-sdk, buzz-core, buzz-persona, buzz-ws-client |

### Backend — Operations, Git, Push, Pairing, Test (layer 3)

| Module | Purpose | Files | LOC | Tests | Dependencies |
|---|---|---|---|---|---|
| buzz-admin | Operator CLI (membership, key-gen, migrate, reconcile) | 1 | 584 | — | db, core, auth, pubsub, search, audit, workflow, media |
| buzz-push-gateway | Web Push fan-out gateway (separate service) | 13 | 4,092 | Unit + integration | — |
| buzz-pair-relay | Ephemeral NIP-AB device-pairing sidecar relay | 3 | 2,445 | Unit | — |
| buzz-pairing-cli | NIP-AB pairing interop CLI | 1 | 623 | — | buzz-core |
| git-sign-nostr | Sign git objects with a Nostr key | 2 | 2,511 | Unit | — |
| git-credential-nostr | Git credential helper for Nostr push/fetch | 3 | 625 | Unit | — |
| buzz-test-client | Integration test client + E2E suite (~134 tests) | 20 | 14,672 | E2E harness | buzz-core, buzz-ws-client, buzz-sdk |
| sprig | All-in-one harness bundling ACP + agent + dev-mcp | 1 | 53 | — | buzz-acp, buzz-agent, buzz-dev-mcp |
| countdown-bot (example) | Example workflow bot | 1 | — | — | — |

### Clients

| Module | Purpose | Files | LOC | Tests | Dependencies |
|---|---|---|---|---|---|
| desktop (frontend) | Tauri+React 19 desktop UI, 28 feature packages | 1,249 (.ts/.tsx) | 281,922 | Vitest + Playwright | Tauri IPC → src-tauri |
| desktop (src-tauri) | Rust native shell: relay mgmt, event sync, media proxy, huddle, mesh, managed agents | 219 (.rs) | 97,897 | Cargo unit tests | buzz-* sidecars (spawned) |
| web | Browser repo-browser client (served by relay) | 51 | 4,557 | Playwright smoke | isomorphic-git, nostr-tools |
| mobile | Flutter iOS/Android client (Riverpod + hooks) | 217 (.dart) | 53,343 | flutter test | web_socket_channel, nostr |

## Dependency Graph

```
                         buzz-core  (zero-I/O foundation)
                             ▲
      ┌───────┬────────┬─────┼──────┬────────┬────────┬─────────┐
   buzz-sdk buzz-db buzz-auth│  buzz-search buzz-audit buzz-media│
      ▲        ▲       ▲      │      ▲          ▲         ▲       │
      │        │   buzz-pubsub┘      │          │         │      buzz-workflow
      │        │       ▲             │          │         │        ▲   ▲
      │        └───────┼─────────────┼──────────┼─────────┼────────┘   │
      │                │        buzz-relay-mesh  buzz-conformance       │
      │                │             ▲          ▲                       │
      └────────────────┴─────────────┴──────────┴───────────────────────┘
                                     │
                                buzz-relay  ── the sole integrator (server)

  Agent surface:                    Ops / git / test:
   buzz-persona ◀── buzz-acp         buzz-admin ──▶ (db,auth,pubsub,search,
   buzz-core   ◀── buzz-acp,buzz-cli               audit,workflow,media)
   buzz-ws-client ◀── buzz-cli,      buzz-test-client ──▶ (core,ws-client,sdk)
                      buzz-test-client
   buzz-cli, git-credential-nostr,   sprig ──▶ (acp, agent, dev-mcp)
   git-sign-nostr ◀── buzz-dev-mcp

  buzz-agent, buzz-push-gateway, buzz-pair-relay, git-*  — standalone (no internal deps)
```

Arrows point from a dependency to its dependents (▲ = "is depended on by"). The relay
sits at the sink of the service layer; agent/ops/test crates form a separate cluster
rooted at `buzz-core`/`buzz-sdk`.

## Circular Dependencies

No circular dependencies detected. (Rust's crate graph is acyclic by construction; the
workspace compiles, so no cycles exist among crates. Within-crate module cycles were not
inspected in this lightweight pass and will be checked per-module in Phase 2.)

## Module Details

Detailed per-module analysis (exports, imports, responsibilities, data models, APIs,
business rules) is produced in Phase 2 and recorded across `data-model.md`,
`api-surface.md`, `business-rules.md`, `features.md`, `integrations.md`,
`conventions.md`, `security.md`, and `debt.md`. High-level responsibilities:

### buzz-core
- **Path**: `crates/buzz-core`
- **Responsibilities**: Nostr event types (`StoredEvent`), Schnorr `verify_event`, NIP-01 `filters_match`, the kind registry (`KIND_*`, `ALL_KINDS`), SSRF `is_private_ip`. Explicitly prohibits tokio/sqlx/redis/axum.

### buzz-relay
- **Path**: `crates/buzz-relay`
- **Responsibilities**: WebSocket connection lifecycle, NIP-42 auth, EVENT pipeline, REQ subscription registry + three-tier fan-out, HTTP bridge (`/events`, `/query`, `/count`, `/hooks`, `/media`, `/git`, `/info`, health), huddle Opus audio relay, community/tenant fencing, orchestration of every subsystem. Largest and most central module.

### desktop (frontend)
- **Path**: `desktop/src`
- **Responsibilities**: 28 feature packages (channels, chat, forum, huddle, agents, workflows, search, communities, moderation, projects, reminders, settings, onboarding, etc.), a `shared/` layer, and an `app/` shell. Community switching via React key-remount with explicit singleton resets.

<!-- Full module deep-dives follow in Phase 2 (one crate/client at a time). -->
