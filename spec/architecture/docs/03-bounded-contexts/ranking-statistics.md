# Ranking & Statistics — Bounded Context Architecture

> **Design reference:** [Spec §2.1.4](../../../domain/docs/02-bounded-contexts-and-context-map.md), [Flows §5 Flow 3](../../../domain/docs/05-domain-event-flows.md), [Consistency §7.3](../../../domain/docs/07-consistency-and-recovery-strategy.md)

---

## Purpose and Scope

**Owns:** Elo rating system (global, casual-only), player statistics (all game types), leaderboard projections, and historical rating aggregation.

**Does not own:** Tournament-placement rating (TPR — owned by Tournament Orchestration), game outcomes (owned by Room Gameplay), player identity.

---

## Services (Containers)

| Service | Responsibility | Scaling |
|---------|---------------|---------|
| `ranking-service` | Combined application service + consumer worker. Consumes `GameCompleted` events, applies Elo pipeline (filter → idempotency → compute → persist), maintains leaderboard and player statistics. Serves read queries (leaderboard, player profile stats). | Horizontal via Kafka consumer group. Scale with `GameCompleted` throughput. |

**Why not split service + worker?** The write path (Elo computation) and read path (leaderboard queries) share the same data store and are both low-latency. Splitting would add deployment complexity without meaningful scaling benefit. If read pressure grows (e.g., leaderboard API hot), scale read replicas + Redis cache independently.

---

## Public Interfaces

### Synchronous (REST)

| Endpoint | Method | Auth | Description |
|----------|--------|------|-------------|
| `/api/v1/leaderboard` | GET | Public | Top-N players by Elo. Paginated. Served from Redis sorted set. |
| `/api/v1/leaderboard/around/{playerId}` | GET | Bearer JWT | Player's rank ± N neighbors. |
| `/api/v1/players/{playerId}/rating` | GET | Bearer JWT | Current Elo, rating history, games rated. |
| `/api/v1/players/{playerId}/statistics` | GET | Bearer JWT | Aggregate stats: games played, W/L/F/A, avg placement, streaks. |

### Asynchronous (Kafka)

**Consumed:**

| Topic | Event | Source | Reaction |
|-------|-------|--------|----------|
| `identity.sessions` | `PlayerRegistered` | `identity-service` | Initialize `PlayerRating` record with `currentElo = 1200`, `gamesRated = 0`. Initialize `PlayerStatistics` with zeroed counters. (Per spec INV-PR-06.) |
| `gameplay.games` | `GameCompleted` | `game-engine` | **Elo pipeline** (see below). Also updates player statistics regardless of filters. |
| `gameplay.events` | `PlayerForfeited` | `game-engine` | Increment forfeit count in player statistics. No Elo impact unless the game subsequently completes as casual. |
| `tournament.lifecycle` | `TournamentCompleted` | `tournament-service` | Compute Tournament Placement Rating (TPR) for all finalists based on tournament placement. Update `PlayerRating.tournamentPlacementRating`. |

**Produced:**

| Topic | Event | Partition Key | Description |
|-------|-------|---------------|-------------|
| `ranking.updates` | `EloUpdated` | `playerId` | Per-player Elo change. Consumed by `audit-service`. |
| `ranking.updates` | `LeaderboardUpdated` | `"global"` | Leaderboard refresh signal. Informational. |
| `ranking.updates` | `PlayerStatisticsUpdated` | `playerId` | Stats update confirmation. |

---

## Internal-Only Interfaces

None. All interfaces are public (REST) or async (Kafka). No intra-context RPC.

---

## Dependencies on Other Contexts

| Upstream Context | Dependency Type | Contract |
|-----------------|----------------|----------|
| Identity & Session | `PlayerRegistered` event consumption (initializes rating record), Player ID reference | OHS / Published Language |
| Room Gameplay | `GameCompleted`, `PlayerForfeited` event consumption | Conformist (RK conforms to RG's event schema) |
| Tournament Orchestration | `TournamentCompleted` event consumption (TPR computation) | Conformist (RK conforms to TO's event schema) |

---

## Elo Pipeline (Core Processing Path)

### Step-by-Step

```
GameCompleted event arrives
         │
         ▼
    ┌─────────────────┐
    │ 1. Idempotency   │   Check: (playerId, gameId) in processed_games?
    │    Guard          │   If yes → ACK and discard (already processed).
    └────────┬─────────┘
             │ no
             ▼
    ┌─────────────────┐
    │ 2. Room Type     │   Check: event.roomType == "casual"?
    │    Filter        │   If tournament → skip Elo, still update stats.
    └────────┬─────────┘
             │ yes
             ▼
    ┌─────────────────┐
    │ 3. Abandonment   │   Check: event.wasAbandoned == false?
    │    Filter        │   If wasAbandoned → skip Elo, still update stats.
    └────────┬─────────┘
             │ yes
             ▼
    ┌─────────────────┐
    │ 4. Fetch Current │   Load currentElo and gamesRated for ALL
    │    Ratings       │   players in event.placements[].
    └────────┬─────────┘
             │
             ▼
    ┌─────────────────┐
    │ 5. K-Factor      │   Per player:
    │    Assignment     │     gamesRated < 30  → K = 40
    │                   │     30 ≤ gamesRated < 100 → K = 20
    │                   │     gamesRated ≥ 100 → K = 10
    └────────┬─────────┘
             │
             ▼
    ┌─────────────────┐
    │ 6. Pairwise Elo  │   Multi-player adaptation:
    │    Computation    │   For each pair (i, j):
    │                   │     E_i = 1 / (1 + 10^((R_j - R_i)/400))
    │                   │     S_i = 1 if placement(i) < placement(j), else 0
    │                   │   Sum over all opponents:
    │                   │     Δ_i = K * Σ(S_ij - E_ij) / (N-1)
    └────────┬─────────┘
             │
             ▼
    ┌─────────────────┐
    │ 7. Floor + Apply │   newElo = max(currentElo + Δ, 100)
    │                   │   Rating floor: 100 (never below).
    └────────┬─────────┘
             │
             ▼
    ┌─────────────────┐
    │ 8. Atomic Persist │   Single TX per player:
    │                   │     UPDATE player_ratings SET elo = newElo,
    │                   │       games_rated = games_rated + 1
    │                   │     INSERT INTO rating_history (playerId, gameId, oldElo, newElo, delta)
    │                   │     INSERT INTO processed_games (playerId, gameId)
    │                   │   COMMIT
    └────────┬─────────┘
             │
             ▼
    ┌─────────────────┐
    │ 9. Emit Events   │   EloUpdated { playerId, oldElo, newElo, delta }
    │    + Cache Update │   ZADD leaderboard playerId newElo (Redis)
    └──────────────────┘
```

### Idempotency Detail

The `processed_games` table has a unique constraint on `(playerId, gameId)`. The INSERT in step 8 will fail on duplicate → TX rolls back → no double-adjustment. This is checked both at step 1 (fast path: query before compute) and at step 8 (safety net: DB constraint).

### Partial Failure

If the service crashes after persisting Elo for players A and B but before C:
- On redelivery of `GameCompleted`, step 1 detects A and B are already processed (skip them).
- Only C is computed and persisted.
- Result: correct. Each player's Elo update is independently atomic.

---

## Persistence

| Store | Technology | Data | Consistency |
|-------|-----------|------|-------------|
| Primary | PostgreSQL | `player_ratings` (playerId, currentElo, gamesRated, updatedAt), `rating_history` (playerId, gameId, oldElo, newElo, delta, timestamp), `processed_games` (playerId, gameId — idempotency), `player_statistics` (playerId, gamesPlayed, won, lost, forfeited, abandoned, avgPlacement, currentStreak, longestStreak) | Strong per-player (single TX) |
| Leaderboard Cache | Redis Sorted Set | `leaderboard:global` — score = Elo, member = playerId | Eventual (updated after Elo persist, ~100ms lag) |

### Read Models

| Read Model | Source | Staleness | Mechanism |
|------------|--------|-----------|-----------|
| Global leaderboard (top-N) | Redis sorted set | ≤ 1 second | Updated on every `EloUpdated`. ZRANGEBYSCORE for queries. |
| Player rank | Redis ZRANK | ≤ 1 second | Direct Redis query. |
| Rating history | PostgreSQL | Real-time (same DB) | Direct query with playerId index. |
| Player statistics | PostgreSQL | Real-time (same DB) | Direct query. |

### Retention

- `rating_history` is append-only, retained indefinitely (audit trail for rating disputes).
- `processed_games` entries older than 30 days can be pruned (idempotency is only needed for the redelivery window).
- `player_statistics` are kept for the lifetime of the player account.
