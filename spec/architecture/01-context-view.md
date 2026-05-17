# 1. Context View — Bounded Contexts to Deployable Services

> Maps each bounded context from the [Design Checkpoint](../spec/docs/02-bounded-contexts-and-context-map.md) to concrete deployable components. Every service listed here traces to aggregates, commands, and events defined in the design package.

---

## 1.1 Context-to-Service Mapping

| Bounded Context | Deployable Service(s) | Primary Responsibility | Scaling Model |
|-----------------|----------------------|------------------------|---------------|
| **Identity & Session (IS)** | `identity-service` | Player registration, authentication (JWT issuance), session lifecycle, single-active-session enforcement, push-invalidation broadcast | Horizontal (stateless, DB-backed sessions) |
| **Room Gameplay (RG)** | `room-service` | Room lifecycle (waiting→ready→in_progress→completed/abandoned), match coordination (best-of-3 scoreline, next-game start, early termination), player slot management | Horizontal, partitioned by `roomId` |
| | `game-engine` | Server-authoritative game state machine: deck/RNG, hands, turns, card plays, Uno/WDF challenges, sequence-number enforcement, log-before-broadcast (event-sourced with transactional outbox) | Horizontal, partitioned by `gameId` |
| | `timer-service` | Durable domain timers: 5-second Uno! challenge window, 5-second WDF challenge window, 60-second reconnection window, 30/60-second turn timer. Persisted deadlines survive process crashes. Idempotent expiry commands. | Horizontal, sharded by deadline bucket |
| **Tournament Orchestration (TO)** | `tournament-service` | Tournament lifecycle, registration, round management, advancement logic (top-3, tiebreak), final-room creation, completion counters | Horizontal (low-frequency, event-driven) |
| | `round-kickoff-worker` | First-round surge: fans out `TournamentRoomAssigned` for ~100k rooms. Sharded queue consumers with rate-limited enqueue to avoid thundering herd. Idempotent room creation. | Horizontal, sharded by round partition |
| **Ranking & Statistics (RK)** | `ranking-service` | Elo computation pipeline (K-factor, multi-player pairwise), leaderboard maintenance, player statistics aggregation. Consumes `GameCompleted` with idempotency guard. | Horizontal (consumer group per partition) |
| **Spectator View (SV)** | `spectator-projection-service` | ACL-filtered read model materialization from RG events. Privacy enforcement: strips hands, draw pile, deck seed, sequence numbers, signatures. Serves spectator queries and feeds. Also materializes lobby (available rooms) and tournament bracket projections. | Horizontal (consumer group, read-replica scaling) |
| **Audit & Game Log (AL)** | `audit-service` | Append-only ingestion of all domain events. Full-payload game log for replay/dispute. Signature verification (HMAC). Compliance query API. | Horizontal (partitioned ingestion), append-only store |

### Cross-Cutting Components (not owned by any single context)

| Component | Responsibility | Owner |
|-----------|---------------|-------|
| `api-gateway` | WebSocket/SSE termination, JWT validation, edge rate limiting (per-IP), request routing, session-to-connection mapping, push-invalidation listener (closes superseded sessions' connections) | Platform / Infrastructure |
| **Message Broker** (Kafka) | Event backbone for all async integration. Topics per context. Partitioned by aggregate ID for ordering guarantees. | Platform / Infrastructure |
| **Redis** | Session token cache (IS), rate-limit counters (gateway + per-service), leaderboard sorted sets (RK), timer deadlines auxiliary index (RG) | Platform / Infrastructure |

---

## 1.2 Context Map — Architectural Relationships

The DDD context map from the design translates to the following runtime integration:

```
┌─────────────────────┐
│  identity-service   │ ─── OHS / Published Language ──────────────────┐
│  (IS)               │                                                │
└────────┬────────────┘                                                │
         │ SessionInvalidated                                          │
         │ (topic: identity.sessions)                                  │
         ▼                                                             │
┌─────────────────────┐    TournamentRoomAssigned     ┌───────────────────────┐
│  room-service       │◄─────────────────────────────│  tournament-service    │
│  game-engine        │                               │  round-kickoff-worker  │
│  timer-service      │  MatchCompleted,              │  (TO)                  │
│  (RG)               │──RoomCompleted,──────────────►│                        │
│                     │  PlayerForfeited              └───────────┬───────────┘
└──┬──────┬───────┬───┘                                          │
   │      │       │                                              │
   │      │       │ GameCompleted                                │ Tournament events
   │      │       │ (topic: gameplay.games)                      │ (topic: tournament.lifecycle)
   │      │       ▼                                              │
   │      │  ┌────────────────────┐                              │
   │      │  │  ranking-service   │                              │
   │      │  │  (RK) — Conformist │                              │
   │      │  └────────────────────┘                              │
   │      │                                                      │
   │      │ All gameplay events                                  │
   │      │ (topic: gameplay.events)                             │
   │      ▼                                                      ▼
   │  ┌─────────────────────────────┐
   │  │ spectator-projection-service│ ◄── ACL (privacy filter)
   │  │ (SV)                        │     Strips: hands, deck, seed, seqNums
   │  └─────────────────────────────┘
   │
   │  All events from all contexts
   │  (topics: *.events)
   ▼
┌─────────────────────┐
│  audit-service      │ ◄── Universal Downstream Conformist
│  (AL)               │     Full payload, signatures, append-only
└─────────────────────┘
```

---

## 1.3 New Architectural Components (Delta from Design)

The following components are **new** — they did not exist as explicit entities in the Design Checkpoint but emerge from architectural constraints:

| Component | Rationale | Design Artifact It Traces To |
|-----------|-----------|------------------------------|
| `timer-service` | Domain timers (5s Uno!, 60s reconnection) must survive process crashes. A dedicated timer service with persisted deadlines and idempotent expiry ensures durability. The Design Checkpoint described these as aggregate-internal timers; the architecture externalizes the scheduling while the Game aggregate retains the deadline state. | Spec §7.3 — Disconnection→Reconnection→Forfeit saga; Spec §3 — ChallengeWindow, ReconnectionWindow value objects |
| `round-kickoff-worker` | The 100k simultaneous room fan-out at tournament start requires sharded, rate-limited workers to avoid overwhelming the broker and `room-service`. The Design Checkpoint modeled this as a single `CreateRound` command; the architecture splits execution into a sharded worker pool. | Spec §5 — Flow 2 Phase 2 (round execution); Spec §7.3 — Tournament Round Progression Saga |
| `api-gateway` | Design assumed direct client-to-context communication. Architecture introduces a gateway for WebSocket/SSE termination, edge rate limiting, auth validation, and push-invalidation of superseded sessions. | Spec §7.6 — Rate Limiting; Spec §2.1.1 — Single-Active-Session Invariant |

These deltas are tracked in [CHANGELOG-design.md](10-CHANGELOG-design.md).

---

## 1.4 Trust Boundaries

```
┌──────────────────────────────────────────────────────────────────┐
│                     PUBLIC INTERNET                              │
│   Clients (players, spectators, admins)                         │
└──────────────────────────┬───────────────────────────────────────┘
                           │ TLS
                    ┌──────▼──────┐
                    │ api-gateway │  ← Edge rate limiting (per-IP)
                    │             │  ← JWT validation
                    │             │  ← WebSocket / SSE termination
                    └──────┬──────┘
                           │ mTLS / private network
┌──────────────────────────▼───────────────────────────────────────┐
│                    INTERNAL SERVICE MESH                          │
│                                                                  │
│  identity-service  room-service  game-engine  timer-service      │
│  tournament-service  round-kickoff-worker  ranking-service       │
│  spectator-projection-service  audit-service                     │
│                                                                  │
│  ┌─────────┐  ┌─────────┐  ┌──────────────────────────┐         │
│  │  Redis   │  │  Kafka  │  │ PostgreSQL (per-context) │         │
│  └─────────┘  └─────────┘  └──────────────────────────┘         │
└──────────────────────────────────────────────────────────────────┘
```

- **Public → Gateway:** TLS only. JWT in header. Gateway validates token, extracts claims (playerId, sessionId), attaches to request context.
- **Gateway → Services:** mTLS or private network. Services trust gateway-injected claims. No token re-validation needed within mesh.
- **Services → Broker/DB:** Private network only. No public exposure.
