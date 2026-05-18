# Tournament Orchestration — Bounded Context Architecture

> **Design reference:** [Spec §2.1.3](../../../domain/docs/02-bounded-contexts-and-context-map.md), [Commands §4.2.3](../../../domain/docs/04-commands-and-domain-events.md), [Flows §5 Flow 2](../../../domain/docs/05-domain-event-flows.md), [Consistency §7.3](../../../domain/docs/07-consistency-and-recovery-strategy.md)

---

## Purpose and Scope

**Owns:** Tournament lifecycle (draft → registration_open → in_progress → completed/cancelled), player registration, round management, player-to-room distribution, advancement logic (top 3 per room, tiebreak by card points then completion time), final-room creation (≤10 remaining players), and completion counters. Produces the `TournamentCompleted` event with final placements that feeds the TPR computation downstream.

**Does not own:** Uno game rules, match gameplay, Elo rating, TPR computation or storage (owned by Ranking & Statistics — see INV-PR-08), room/game state machines.

---

## Services (Containers)

| Service | Responsibility | Scaling |
|---------|---------------|---------|
| `tournament-service` | Tournament aggregate: all lifecycle commands (`CreateTournament`, `OpenRegistration`, `RegisterForTournament`, `StartTournament`). Round management: `CreateRound`, advancement evaluation, `ForceResolveTimedOutRoom`. Completion counter: atomically increments when `RoomCompleted` arrives; triggers next round when counter == totalRooms. | Horizontal, partitioned by `tournamentId`. Low-frequency except during round transitions. |
| `round-kickoff-worker` | Sharded worker pool dedicated to the first-round surge: fans out `TournamentRoomAssigned` for ~100k rooms without overwhelming the broker or `room-service`. Reads room assignment work items from an internal queue, publishes at controlled rate. | Horizontal, sharded by player-range partition. Key scaling lever for tournament start. |

### Why `round-kickoff-worker` is Separate

The first-round surge (~100k rooms, ~1M players) is a unique scaling challenge distinct from normal tournament management. `tournament-service` handles the decision logic (which players go where); `round-kickoff-worker` handles the execution (publishing room assignments at a controlled rate). This separation allows:

1. `tournament-service` to remain responsive for status queries during kickoff.
2. Independent scaling of fan-out workers (add shards for bigger tournaments).
3. Backpressure isolation: if `room-service` slows down, kickoff workers back off without blocking tournament state transitions.

---

## Public Interfaces

### Synchronous (REST)

| Endpoint | Method | Auth | Description | Design Command |
|----------|--------|------|-------------|----------------|
| `/api/v1/tournaments` | POST | Organizer JWT | Create tournament | `CreateTournament` |
| `/api/v1/tournaments/{id}/open-registration` | POST | Organizer JWT | Open registration | `OpenRegistration` |
| `/api/v1/tournaments/{id}/register` | POST | Bearer JWT | Register player | `RegisterForTournament` |
| `/api/v1/tournaments/{id}/start` | POST | Organizer JWT | Start tournament | `StartTournament` |
| `/api/v1/tournaments/{id}` | GET | Bearer JWT | Tournament status, current round | (Query) |
| `/api/v1/tournaments/{id}/bracket` | GET | Public | Bracket progression | (Query, served by spectator-projection-service) |
| `/api/v1/tournaments/{id}/rounds/{roundId}/force-resolve` | POST | Admin JWT OR Organizer JWT | Force-resolve timed-out rooms | `ForceResolveTimedOutRoom` |
| `/api/v1/tournaments/{id}/close-registration` | POST | Organizer JWT | Close registration manually (alternative to automatic close on StartTournament) | `CloseRegistration` |
| `/api/v1/tournaments/{id}/cancel` | POST | Organizer JWT or Admin JWT | Cancel tournament | `CancelTournament` |

> **Auth for ForceResolveTimedOutRoom:** Admin JWT (any tournament) OR Organizer JWT (only tournaments where `organizerId` matches the JWT's `playerId`). This allows organizers to self-service stuck rooms in their own tournaments without admin escalation.

> **Design decision (T-03 — KickPlayer):** The `KickPlayer` command from the domain catalog is not applicable to tournament rooms (players are system-assigned by the round-kickoff process and cannot be manually removed by a host). For casual rooms, player departure is handled via `LeaveRoom` (already documented in room-gameplay.md). `ForceResolveRoom` is mapped as S16 in the integration table (`ForceResolveTimedOutRoom` endpoint above).

**CancelTournament flow:**
- **Valid states:** `registration_open`, `in_progress`. Rejected if already `completed` or `cancelled` (409).
- **Registration-open cancellation:** Immediately transitions to `cancelled`. Emits `TournamentCancelled { tournamentId, reason, cancelledAt }` on `tournament.lifecycle`. All `tournament_registrations` rows for this tournament are marked `cancelled`. No rooms to clean up.
- **In-progress cancellation:** Transitions to `cancelled`. Emits `TournamentCancelled`. **In-progress rooms are NOT force-terminated** — they complete naturally (players finish their current games). However, no further rounds are created: the round-advancement saga checks tournament status before creating the next round and stops if `cancelled`. The `TournamentCancelled` event is consumed by `spectator-projection-service` to update the bracket display.
- **Compensation:** No Elo reversal needed (tournament games don't affect Elo). No room teardown needed (rooms complete naturally). Players' `active_player_rooms` rows are cleared when their current room completes normally.

### Asynchronous (Kafka)

**Produced:**

| Topic | Event(s) | Partition Key | Description |
|-------|----------|---------------|-------------|
| `tournament.lifecycle` | `TournamentCreated`, `RegistrationOpened`, `TournamentStarted`, `TournamentRoundCreated`, `PlayerAdvanced`, `PlayerEliminated`, `FinalRoomCreated`, `AllMatchesInRoundCompleted`, `TournamentCompleted`, `TournamentCancelled` | `tournamentId` | All tournament state transitions. Consumed by `spectator-projection-service` (bracket) and `audit-service`. |
| `tournament.rooms` | `TournamentRoomAssigned` | `roomId` | Room creation commands for `room-service`. High burst at round start. Produced by `round-kickoff-worker`. |

**Consumed:**

| Topic | Event | Source | Reaction |
|-------|-------|--------|----------|
| `gameplay.rooms` | `MatchCompleted` | `room-service` | Record match result for tournament room. |
| `gameplay.events` | `PlayerForfeited` | `game-engine` | If tournament room: mark player eliminated, emit `PlayerEliminated`. Record forfeit as loss for advancement. |
| `gameplay.rooms` | `RoomCompleted` | `room-service` | Mark tournament room completed. Increment completion counter. If `completedRooms == totalRooms`: trigger advancement → next round or final. |
| `identity.sessions` | `PlayerSuspended`, `PlayerBanned` | `identity-service` | Disqualify player from active tournament (if registered/in-progress). |

---

## Internal-Only Interfaces

| Interface | Description |
|-----------|-------------|
| `tournament-service` → `round-kickoff-worker` | Internal work queue (Kafka topic `tournament.kickoff-work`). Each work item: `{ tournamentId, roundId, roomAssignment: { roomId, playerIds[], roomConfig } }`. Technology: Kafka (per ADR-006); no Redis list alternative. |
| Completion counter | Atomic counter in PostgreSQL: `UPDATE tournament_rounds SET completed_rooms = completed_rooms + 1 WHERE roundId = ? RETURNING completed_rooms`. Compared against `total_rooms` to detect round completion. |

---

## Dependencies on Other Contexts

| Upstream Context | Dependency Type | Contract |
|-----------------|----------------|----------|
| Identity & Session | Player validation on registration (sync), `PlayerSuspended`/`PlayerBanned` events (async via `identity.sessions`) | OHS / Published Language |
| Room Gameplay | `MatchCompleted`, `RoomCompleted` (via `gameplay.rooms`), `PlayerForfeited` (via `gameplay.events`) event consumption | Conformist (TO conforms to RG's event schema) |

| Downstream Context | Dependency Type | Contract |
|-------------------|----------------|----------|
| Room Gameplay | `TournamentRoomAssigned` event production | Published Language (TO defines the schema, RG conforms for room creation) |

---

## First-Round Surge Architecture (§6.1 mandatory)

The product definition specifies up to 1,000,000 players in a tournament. The first round creates ~100,000 rooms (10 players each) that must all start within seconds — a coordinated spike, not gradual load.

### Kickoff Sequence

```
tournament-service                    round-kickoff-worker (N shards)          room-service
       │                                        │                                    │
       │  1. StartTournament                    │                                    │
       │     → TournamentStarted                │                                    │
       │     → CreateRound(round=1,             │                                    │
       │       players=[1M playerIds])           │                                    │
       │                                        │                                    │
       │  2. AssignPlayersToRooms               │                                    │
       │     Shuffle 1M players                 │                                    │
       │     Partition into 100k rooms          │                                    │
       │     of 10 players each                 │                                    │
       │                                        │                                    │
       │  3. Enqueue work items to              │                                    │
       │     tournament.kickoff-work            │                                    │
       │     (partitioned by player range)      │                                    │
       │     ─────────────────────────────────► │                                    │
       │                                        │                                    │
       │                              4. Each shard processes its partition:          │
       │                                 For each room assignment:                   │
       │                                   Publish TournamentRoomAssigned            │
       │                                   to tournament.rooms                       │
       │                                   (rate-limited: 1000 rooms/sec/shard)      │
       │                                        │ ─────────────────────────────────► │
       │                                        │                                    │
       │                                        │                            5. room-service
       │                                        │                               consumes
       │                                        │                               TournamentRoomAssigned:
       │                                        │                               - Create room
       │                                        │                               - Auto-join players
       │                                        │                               - Auto-start match
       │                                        │                                    │
```

### Anti-Thundering-Herd Controls

| Control | Mechanism |
|---------|-----------|
| **Sharded workers** | `round-kickoff-worker` runs N shards (e.g., 10–20). Each shard handles a disjoint partition of rooms (by player-range hash). No coordination needed between shards. |
| **Rate-limited enqueue** | Each shard publishes at most 1,000 `TournamentRoomAssigned` events/sec to Kafka. With 10 shards: 10,000 rooms/sec total → all 100k rooms published in ~10 seconds. |
| **Kafka partitioning** | `tournament.rooms` topic partitioned by `roomId`. `room-service` consumer group distributes load across instances. No single instance handles all 100k rooms. |
| **Backpressure** | If `room-service` consumer lag exceeds threshold, kickoff workers reduce publish rate (adaptive backpressure via Kafka consumer lag monitoring). |
| **Idempotent room creation** | `TournamentRoomAssigned` carries a `roomId` (pre-generated by `tournament-service`). `room-service` uses `roomId` as idempotency key — duplicate events create no duplicate rooms. |

### Partial Failure Handling

| Failure | Detection | Recovery |
|---------|-----------|----------|
| Kickoff worker crash mid-partition | Kafka consumer group rebalance assigns orphaned partitions to surviving workers | Workers resume from last committed offset. Idempotent room creation ensures no duplicates. |
| `room-service` rejects a room (e.g., player already in another room) | `TournamentRoomAssigned` results in error event | `tournament-service` receives failure, logs it, and either retries with a substitute player or marks room as degraded. DLQ for unrecoverable failures. |
| Kafka broker overload | Producer backpressure (buffer full) | Kickoff workers pause, retry with exponential backoff. 10-second target becomes ~30 seconds. Acceptable. |
| Some rooms never start (permanent kickoff failure) | `tournament-service` monitors `RoomCreated` acknowledgments. **Room-creation timeout (5 min):** if a room has been assigned (`TournamentRoomAssigned` sent) but `room-service` has not emitted `RoomCreated` within 5 minutes, the assignment is treated as permanently failed. This is a *kickoff failure* — the room never started. Escalation path: DLQ message arrives at `tournament-service` → marks room as `creation_failed` → after the 5-minute window emits `TournamentRoomResolved { reason: "creation_timeout" }`. **This is distinct from `ForceResolveTimedOutRoom`**, which applies only to rooms that *did* start (i.e., a game is running) but haven't completed within `maxRoundDuration` (2 hours). The two timeouts target different lifecycle stages and use different resolution commands. |

---

## Round Advancement Saga

### State Machine

```
RoundCreated
  │
  ├── DistributePlayersToRooms
  │     └── TournamentRoomAssigned (per room) ──► room-service creates rooms
  │
  ├── WaitForRoomCompletions
  │     ├── RoomCompleted ──► increment completedRooms
  │     │     if completedRooms == totalRooms:
  │     │       └── AllMatchesInRoundCompleted
  │     │
  │     └── [Timeout: 75% of maxRoundDuration]
  │           └── RoundTimeoutWarning (operational alert)
  │           [Timeout: 100% of maxRoundDuration (2 hours)]
  │             └── ForceResolveTimedOutRoom (for each room that STARTED but has not COMPLETED)
  │                   — "round-completion timeout": for in-progress rooms only
  │                   — distinct from the 5-min "room-creation timeout" (§Partial Failure Handling)
  │                   └── Players ranked by current game state, advancing/eliminated
  │
  ├── EvaluateAdvancement
  │     For each completed room:
  │       Top 3 by matchWins → cardPoints → completionTime → advance
  │       Rest → eliminated
  │     Emit PlayerAdvanced / PlayerEliminated per player
  │
  └── DecideNextStep
        if remainingPlayers > 10:
          └── CreateRound(nextRoundNumber, advancingPlayerIds)
        if remainingPlayers <= 10:
          └── CreateFinalRoom(advancingPlayerIds)
              └── TournamentRoomAssigned (single room)
        if remainingPlayers == 1:
          └── TournamentCompleted (edge case: all others forfeited)
```

### Completion Counter (Critical Sync Point)

The completion counter is the synchronization mechanism for round progression:

```sql
-- Atomic increment on RoomCompleted
UPDATE tournament_rounds
SET completed_rooms = completed_rooms + 1
WHERE round_id = $1
RETURNING completed_rooms, total_rooms;

-- If completed_rooms == total_rooms → trigger advancement
```

**Idempotency:** `RoomCompleted` events are deduplicated by `(roundId, roomId)` unique constraint in `tournament_room_results`. Duplicate events do not increment the counter twice.

---

## `GameCompleted` Spike at Round End (§6.1 mandatory)

When a tournament round ends, ~100k rooms complete within a short window (minutes, not seconds — game durations vary). This creates a spike of `GameCompleted`, `MatchCompleted`, and `RoomCompleted` events.

### Ingestion Strategy

The `tournament-service` consumer group for `gameplay.rooms` topic absorbs this spike:

| Layer | Mechanism |
|-------|-----------|
| **Kafka partitioning** | `gameplay.rooms` partitioned by `roomId`. Consumer group distributes partitions across `tournament-service` instances. |
| **Consumer group scaling** | Tournament service scales horizontally during tournament rounds. Target: 1 consumer per 2–4 Kafka partitions. |
| **Async processing** | `tournament-service` processes `RoomCompleted` asynchronously — increment counter, check if round complete. No synchronous call back to `room-service`. |
| **No backpressure to RG** | `tournament-service` has its own consumer group, independent of `ranking-service` or `spectator-projection-service`. Slow tournament processing does not back-pressure Room Gameplay writers. |
| **Separate topics** | `gameplay.rooms` (room lifecycle events) is a separate topic from `gameplay.events` (high-volume per-card events). Tournament service only subscribes to `gameplay.rooms`. |

---

## Persistence

| Store | Technology | Data | Consistency |
|-------|-----------|------|-------------|
| Primary | PostgreSQL | `tournaments` (id, name, status, organizer, config, registeredCount), `tournament_registrations` (tournamentId, playerId, registeredAt), `tournament_rounds` (roundId, tournamentId, roundNumber, totalRooms, completedRooms, status), `tournament_room_results` (roundId, roomId, rankings, completedAt), `tournament_brackets` (player advancement paths) | Strong (SERIALIZABLE for completion counter) |
| Kickoff queue | Kafka topic `tournament.kickoff-work` | Work items for round-kickoff-worker | At-least-once delivery |

**Idempotency store:** `processed_events` table (eventId → timestamp). Checked before processing any consumed event.

---

## Between-Round Reconnection Window

**Owner:** `tournament-service`

### Problem

When a new tournament round begins and players are assigned to rooms, some players may be temporarily disconnected or slow to rejoin. The system must enforce a bounded deadline for players to join their assigned room — otherwise a single absent player blocks an entire room from starting.

### Mechanism

1. When `TournamentRoundCreated` is emitted and `TournamentRoomAssigned` events are published, `tournament-service` sets a **per-player deadline of 60 seconds** from room assignment.
2. If the player's `PlayerJoinedRoom` event (consumed from `gameplay.rooms`) is **not received within 60 seconds** of assignment, `tournament-service` emits `PlayerEliminated { playerId, tournamentId, roundId, reason: "failed_to_join" }`.
3. The room proceeds with fewer players (minimum 2; if below minimum, room is resolved with remaining players advancing by default).

### Persistence — `round_join_deadlines` Table

```sql
CREATE TABLE round_join_deadlines (
    tournament_id   UUID NOT NULL,
    round_id        UUID NOT NULL,
    player_id       UUID NOT NULL,
    room_id         UUID NOT NULL,
    assigned_at     TIMESTAMPTZ NOT NULL,
    joined_at       TIMESTAMPTZ,          -- NULL until PlayerJoinedRoom received
    expires_at      TIMESTAMPTZ NOT NULL,  -- assigned_at + 60s
    status          TEXT NOT NULL DEFAULT 'pending',  -- pending | joined | expired
    PRIMARY KEY (tournament_id, round_id, player_id)
);

CREATE INDEX idx_pending_deadlines ON round_join_deadlines (expires_at)
    WHERE status = 'pending';
```

### Expiry Detection

A background poll (similar to `timer-service` pattern) runs within `tournament-service`:

```
Every 1 second:
  SELECT * FROM round_join_deadlines
  WHERE status = 'pending' AND expires_at <= now()
  LIMIT 100;

  For each expired row:
    UPDATE round_join_deadlines SET status = 'expired' WHERE ... AND status = 'pending';
    IF rows_affected == 1:
      Emit PlayerEliminated { reason: "failed_to_join" }
```

### Crash Recovery

Deadlines are persisted in PostgreSQL. If `tournament-service` crashes and restarts:
- The background poll resumes and immediately picks up any deadlines that expired during downtime.
- `PlayerJoinedRoom` events that arrived during the crash window are replayed from the Kafka consumer group's last committed offset.
- If a `PlayerJoinedRoom` event is processed after the deadline was already marked `expired`, the elimination stands (see Idempotency below).

### Idempotency

- If `PlayerJoinedRoom` arrives **before** the deadline: row is updated to `status = 'joined'`, `joined_at = now()`. No elimination.
- If `PlayerJoinedRoom` arrives **after** the deadline has expired and `PlayerEliminated` has been emitted: the elimination is **irreversible**. The late join is logged but does not reverse the elimination. This prevents race conditions and ensures deterministic tournament progression.
- Duplicate `PlayerJoinedRoom` events are safe: the UPDATE is idempotent (status transition `pending → joined` only; `joined → joined` is a no-op).

---

**Single-tournament-per-player enforcement (INV-T-09):** The `tournament_registrations` table has a UNIQUE constraint on `(playerId)` scoped to tournaments in `registration_open` or `in_progress` status. Implemented as a partial unique index: `CREATE UNIQUE INDEX idx_active_tournament_player ON tournament_registrations (playerId) WHERE status IN ('registered', 'active')`. On `RegisterForTournament`, the INSERT fails if the player is already registered/active in another tournament. Rows transition to `eliminated` or `completed` status when the player is eliminated or the tournament ends, freeing the slot.
