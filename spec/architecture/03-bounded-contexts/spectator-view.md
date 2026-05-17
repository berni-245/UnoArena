# Spectator View — Bounded Context Architecture

> **Design reference:** [Spec §2.1.5](../../spec/docs/02-bounded-contexts-and-context-map.md), [Diagrams: spectator-projection](../../spec/diagrams/spectator-projection.md)

---

## Purpose and Scope

**Owns:** Read-optimized projections of active games for spectator consumption, lobby listing (available casual rooms), and tournament bracket visualization. Enforces strict privacy filtering via an Anti-Corruption Layer: spectators never receive hand contents, draw pile state, deck seed, sequence numbers, or event signatures.

**Does not own:** Game state (read-only projection of Room Gameplay), game rules, tournament advancement logic, player identity.

---

## Services (Containers)

| Service | Responsibility | Scaling |
|---------|---------------|---------|
| `spectator-projection-service` | **Consumer worker:** Subscribes to `gameplay.events`, `gameplay.rooms`, and `tournament.lifecycle` topics. Applies ACL transformations. Materializes read models. **Query service:** Serves spectator REST and SSE endpoints. | Horizontal: consumer group for ingestion (partition-based), read replicas for query load, SSE fan-out via edge/regional proxy. |

### Why a Single Service?

The ingestion (event consumption + ACL + materialization) and query (REST + SSE) roles share the same read model store. Splitting them would add a deployment boundary with no meaningful benefit — the SSE fan-out is the scaling bottleneck, handled by the `api-gateway` edge layer, not by this service.

---

## Public Interfaces

### Synchronous (REST)

| Endpoint | Method | Auth | Description |
|----------|--------|------|-------------|
| `/api/v1/spectator/games/{gameId}` | GET | Public | Current spectator-safe game state snapshot. |
| `/api/v1/spectator/games/{gameId}/stream` | GET (SSE) | Public | Live event stream (spectator-filtered). |
| `/api/v1/lobby/rooms` | GET | Public | Available casual rooms for joining. Paginated, filterable. |
| `/api/v1/tournaments/{id}/bracket` | GET | Public | Tournament bracket progression. |
| `/api/v1/tournaments/{id}/bracket/stream` | GET (SSE) | Public | Live tournament bracket updates. |

**Auth:** Spectator endpoints are public (no JWT required). Rate-limited per IP at the gateway.

### Asynchronous (Kafka — consumed only)

| Topic | Events Consumed | ACL Transformation |
|-------|----------------|-------------------|
| `gameplay.events` | `GameStarted`, `CardPlayed`, `CardDrawn`, `TurnAdvanced`, `TurnPassed`, `DirectionReversed`, `PlayerSkipped`, `ColorChosen`, `UnoCallMade`, `UnoChallengeWindowOpened`, `ChallengeMade`, `ChallengeResolved`, `PenaltyCardsDrawn`, `UnoChallengeWindowClosed`, `WildDrawFourChallengeWindowOpened`, `WildDrawFourChallengeMade`, `WildDrawFourChallengeResolved`, `WildDrawFourChallengeWindowClosed`, `PlayerDisconnected`, `PlayerReconnected`, `PlayerForfeited` | **Privacy filter applied** (see ACL table below) |
| `gameplay.games` | `GameCompleted` | Update game projection to completed state. Strip final hand contents. |
| `gameplay.rooms` | `RoomCreated`, `PlayerJoinedRoom`, `PlayerLeftRoom`, `RoomFilled`, `RoomCompleted` | Lobby read model update. Tournament rooms excluded. |
| `tournament.lifecycle` | `TournamentCreated`, `RegistrationOpened`, `TournamentRoundCreated`, `PlayerAdvanced`, `PlayerEliminated`, `FinalRoomCreated`, `TournamentCompleted` | Bracket projection update. No privacy filtering needed. |

**Produced:** `SpectatorProjectionUpdated` (informational, to `audit-service` only). Low value; can be omitted in initial implementation.

---

## Anti-Corruption Layer (ACL) — Privacy Enforcement

The ACL is the critical architectural component that ensures spectator privacy. It is implemented as a transformation layer within `spectator-projection-service`, applied to every event before materialization or streaming.

### Transformation Table

| Source Event (RG) | Spectator Event | Transformation |
|-------------------|----------------|----------------|
| `GameStarted` | `SpectatorGameStarted` | **Strip:** all hand contents, deck seed, deck composition. **Retain:** player names, initial card counts (7 each), initial discard top card, starting player, direction. |
| `CardPlayed` | `SpectatorCardPlayed` | **Retain as-is.** The played card is public (on the discard pile). Include player's remaining card count. |
| `CardDrawn` | `SpectatorCardDrawn` | **Strip: card identity.** Only `playerId` and new card count. Spectators know a draw happened, not which card. |
| `TurnAdvanced` | `SpectatorTurnAdvanced` | Retain as-is. |
| `DirectionReversed` | `SpectatorDirectionReversed` | Retain as-is. |
| `PlayerSkipped` | `SpectatorPlayerSkipped` | Retain as-is. Include skip reason. |
| `ColorChosen` | `SpectatorColorChosen` | Retain as-is. Chosen color is public. |
| `UnoCallMade` | `SpectatorUnoCallMade` | Retain as-is. |
| `ChallengeMade` | `SpectatorChallengeMade` | Retain as-is. |
| `ChallengeResolved` | `SpectatorChallengeResolved` | **Retain outcome.** **Strip:** penalty card identities. Only card count change. |
| `PenaltyCardsDrawn` | `SpectatorPenaltyCardsDrawn` | **Strip: card identities.** Only playerId, count, reason. |
| `WildDrawFourChallengeWindowOpened` | `SpectatorWDFChallengeWindowOpened` | **Retain:** challenger (affected player), challenged player, window duration. **Strip:** hand snapshot used for adjudication. |
| `WildDrawFourChallengeMade` | `SpectatorWDFChallengeMade` | Retain as-is. Challenger and challenged player identifiers. |
| `WildDrawFourChallengeResolved` | `SpectatorWDFChallengeResolved` | **Retain:** outcome (bluff_confirmed / legitimate_play), who was penalized. **Strip:** hand contents used in adjudication, penalty card identities. |
| `WildDrawFourChallengeWindowClosed` | `SpectatorWDFChallengeWindowClosed` | Retain as-is. |
| `TurnPassed` | `SpectatorTurnPassed` | Retain as-is. |
| `GameCompleted` | `SpectatorGameCompleted` | **Retain:** placement order, match score. **Strip:** final hand contents. |
| `PlayerDisconnected` | `SpectatorPlayerDisconnected` | Retain as-is. |
| `PlayerReconnected` | `SpectatorPlayerReconnected` | Retain as-is. |
| `PlayerForfeited` | `SpectatorPlayerForfeited` | **Retain:** playerId, reason. Remove player from active participants in projection. |

### Where Privacy Is Enforced

```
                                    TRUST BOUNDARY
                                         │
  Room Gameplay (full events)            │     Spectator View (filtered events)
                                         │
  gameplay.events topic ────────────────►│──► spectator-projection-service
    CardDrawn { cardId: "R7",            │      CardDrawn → SpectatorCardDrawn
      playerId: "P1",                    │        { playerId: "P1",
      card: { color: "red", face: "7" }  │          newCardCount: 8 }
    }                                    │        (card identity GONE)
                                         │
                                         │    Materialized read model (DB):
                                         │      Only spectator-safe fields stored.
                                         │      No raw RG events stored.
                                         │
                                         │    SSE stream to spectators:
                                         │      Only SpectatorXxx events.
                                         │      Separate channel from player WS.
```

**Defense in depth:**
1. **Event transformation** — ACL strips private data before materialization. Raw events are never written to the spectator read model.
2. **Separate storage** — Spectator DB contains only spectator-safe projections. Even a full DB compromise reveals no hand data.
3. **Separate streaming channel** — Spectators connect via SSE (read-only, public). Players connect via WebSocket (authenticated, bidirectional). Different `api-gateway` routes. A spectator cannot "upgrade" to a player stream.
4. **No subscription crossover** — Spectators subscribe to `spectator/games/{gameId}` SSE. Players subscribe to `games/{gameId}` WebSocket. The gateway enforces: spectator routes only serve spectator-projection data; player routes only serve full game-engine data (with auth).

---

## Projection Model

**CQRS Read Model** with event-carried state transfer.

### Materialization

Each `SpectatorGameProjection` is a denormalized document representing the current spectator-visible state of a game:

```
SpectatorGameProjection {
  gameId          : UUID
  roomId          : UUID
  gamePhase       : "in_progress" | "completed"
  players         : [
    { playerId, displayName, cardCount, connectionStatus, isCurrentPlayer }
  ]
  discardPile     : { topCard: { color, face } }
  turnDirection   : "clockwise" | "counterclockwise"
  currentPlayer   : playerId
  matchScore      : { playerId → gamesWon }
  lastEvent       : { type, timestamp, summary }  // For live feed rendering
  updatedAt       : ISO-8601
}
```

On each incoming event:
1. Load projection from DB (or in-memory cache).
2. Apply spectator-transformed event (update cardCount, currentPlayer, etc.).
3. Persist updated projection.
4. Push delta to SSE subscribers for this `gameId`.

### Read Models

| Read Model | Source Events | Storage | Staleness |
|------------|-------------|---------|-----------|
| Spectator Game Projection | `gameplay.events` + `gameplay.games` (ACL-filtered) | PostgreSQL or MongoDB (document) | ≤ 500ms (Kafka consumer lag + materialization) |
| Available Rooms (Lobby) | `gameplay.rooms` (casual only) | PostgreSQL | ≤ 1s |
| Tournament Bracket | `tournament.lifecycle` | PostgreSQL | ≤ 1s |

---

## Dependencies on Other Contexts

| Upstream Context | Dependency Type | Contract |
|-----------------|----------------|----------|
| Room Gameplay | All gameplay events (ACL-filtered consumption) | ACL on SV side. RG is unaware of spectator concerns. |
| Tournament Orchestration | Tournament lifecycle events (bracket projection) | Conformist (SV conforms to TO's event schema) |

---

## Persistence

| Store | Technology | Data | Consistency |
|-------|-----------|------|-------------|
| Primary | PostgreSQL or MongoDB | `spectator_game_projections` (denormalized per-game documents), `available_rooms` (lobby listing), `tournament_brackets` (bracket projection) | Eventual (derived from upstream events) |
| SSE State | In-memory (per instance) | Active SSE subscribers per gameId. Subscriber set. | Ephemeral (rebuilt on instance restart) |

**No raw gameplay events stored.** Only the transformed, spectator-safe projections. This is a deliberate privacy safeguard.

### Retention

- Game projections are retained while the game is active. After `GameCompleted`, projection is kept for 24 hours (replay browsing) then archived or deleted.
- Lobby listings are real-time only. Completed rooms are removed immediately.
- Tournament brackets are retained until the tournament completes + 30 days.
