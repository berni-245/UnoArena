# Room Gameplay — Bounded Context Architecture

> **Design reference:** [Spec §2.1.2](../../../domain/docs/02-bounded-contexts-and-context-map.md), [Commands §4.2.2](../../../domain/docs/04-commands-and-domain-events.md), [Flows §5](../../../domain/docs/05-domain-event-flows.md), [Consistency §7](../../../domain/docs/07-consistency-and-recovery-strategy.md)

---

## Purpose and Scope

**Owns:** Room lifecycle, match coordination (best-of-3 scoreline, next-game start, early termination at 2 wins in 2-player rooms), individual game state machine (deck, hands, turns, card plays, Uno/WDF challenges, sequence-number enforcement), server-authoritative RNG, and the immutable game log (write path).

**Does not own:** Tournament structure or advancement, Elo computation, spectator projections, audit trail cross-context.

---

## Services (Containers)

| Service | Responsibility | Scaling |
|---------|---------------|---------|
| `room-service` | Room aggregate: lifecycle state machine (waiting→ready→in_progress→completed/abandoned), player slot management, match coordination. Receives `StartMatch`, `CreateRoom`, `JoinRoom`, `LeaveRoom`, `ForfeitGame`. Coordinates match series: on `GameCompleted`, evaluates scoreline, starts next game or emits `MatchCompleted`. | Horizontal, partitioned by `roomId` |
| `game-engine` | Game aggregate: event-sourced state machine. Processes `PlayCard`, `DrawCard`, `PassTurn`, `CallUno`, `ChallengeUnoCall`, `ChallengeWildDrawFour`, `Reconnect`. Internal commands: `InitializeDeck`, `DealInitialHands`, `FlipFirstDiscardCard`, `AutoSkipTurn`, `AutoForfeit`, `AutoDrawOnTimeout`, `ReshuffleDiscardPile`. Enforces all gameplay invariants. Writes to event store + outbox in same TX. | Horizontal, partitioned by `gameId` |
| `timer-service` | Durable domain timer management. Schedules and fires: 5s Uno challenge window, 5s WDF challenge window, 60s reconnection window, 30/60s turn timer. Persisted deadlines. Idempotent expiry. | Horizontal, sharded by time bucket |

### Why Three Services?

1. **`room-service` vs `game-engine`:** The design already separates Room and Game as distinct aggregates (Spec §3). Game is the high-frequency hot path (~10 commands/player/minute during active play). Room is low-frequency (lifecycle transitions). Separating them allows independent scaling: 100k concurrent games need many `game-engine` instances, while `room-service` handles far fewer transitions.

2. **`timer-service` separated:** Domain timers must survive process crashes. Embedding timers in `game-engine` means a crash mid-challenge-window loses the deadline. A dedicated timer service with persisted deadlines and leader-election-free polling ensures durability. The Game aggregate still owns the deadline state (records `challengeWindowOpensAt`, `reconnectionWindowExpiresAt`); `timer-service` only owns the scheduling and firing.

---

## Public Interfaces

### Synchronous (REST + WebSocket)

**REST (via api-gateway):**

| Endpoint | Method | Auth | Description | Design Command |
|----------|--------|------|-------------|----------------|
| `/api/v1/rooms` | POST | Bearer JWT | Create casual room | `CreateRoom` |
| `/api/v1/rooms/{roomId}/join` | POST | Bearer JWT | Join room | `JoinRoom` |
| `/api/v1/rooms/{roomId}/leave` | POST | Bearer JWT | Leave room (waiting state only) | `LeaveRoom` |
| `/api/v1/rooms/{roomId}/start` | POST | Bearer JWT (host) | Start match | `StartMatch` |
| `/api/v1/rooms/{roomId}/forfeit` | POST | Bearer JWT | Forfeit game | `ForfeitGame` |
| `/api/v1/games/{gameId}/state` | GET | Bearer JWT (participant) | Full game state snapshot (for reconnection) | (Query) |

**WebSocket (via api-gateway, bidirectional):**

Once a game is in progress, the player's WebSocket connection carries both commands and updates:

| Direction | Message Type | Description | Design Command/Event |
|-----------|-------------|-------------|---------------------|
| Client → Server | `play_card` | Play a card | `PlayCard` |
| Client → Server | `draw_card` | Draw from deck | `DrawCard` |
| Client → Server | `pass_turn` | Pass after drawing | `PassTurn` |
| Client → Server | `call_uno` | Call Uno! | `CallUno` |
| Client → Server | `challenge_uno` | Challenge missed Uno call | `ChallengeUnoCall` |
| Client → Server | `challenge_wdf` | Challenge Wild Draw Four | `ChallengeWildDrawFour` |
| Client → Server | `reconnect` | Reconnect to game | `Reconnect` |
| Server → Client | `game_state_update` | Batched events since last ack | All gameplay events |
| Server → Client | `command_rejected` | Rejection with reason code | `CommandRejected` |

**Sequence numbers:** Every gameplay command from the client must include `sequenceNumber` (except `CallUno`, `ChallengeUnoCall`, `ChallengeWildDrawFour` — time-sensitive, no seq required). The `game-engine` rejects stale sequence numbers with `CommandRejected { reason: "stale_sequence_number" }` (HTTP 409 equivalent over WebSocket).

### Asynchronous (Kafka)

**Produced:**

| Topic | Event(s) | Partition Key | Idempotency Key |
|-------|----------|---------------|-----------------|
| `gameplay.events` | `CardPlayed`, `CardDrawn`, `TurnAdvanced`, `TurnPassed`, `DirectionReversed`, `PlayerSkipped`, `ColorChosen`, `UnoCallMade`, `UnoChallengeWindowOpened`, `ChallengeMade`, `ChallengeResolved`, `PenaltyCardsDrawn`, `UnoChallengeWindowClosed`, `WildDrawFourChallengeWindowOpened`, `WildDrawFourChallengeMade`, `WildDrawFourChallengeResolved`, `WildDrawFourChallengeWindowClosed`, `PlayerDisconnected`, `PlayerReconnected`, `PlayerForfeited`, `TurnSkippedDueToDisconnection`, `TurnTimedOut`, `DrawPileReshuffled`, `DeckInitialized`, `InitialHandsDealt`, `FirstCardFlipped`, `GameStarted` | `gameId` | `eventId` |
| `gameplay.games` | `GameCompleted` | `gameId` | `eventId` |
| `gameplay.rooms` | `RoomCreated`, `PlayerJoinedRoom`, `PlayerLeftRoom`, `RoomFilled`, `MatchGameCompleted`, `MatchCompleted`, `RoomCompleted` | `roomId` | `eventId` |

**Consumed:**

| Topic | Event | Source | Reaction |
|-------|-------|--------|----------|
| `identity.sessions` | `SessionInvalidated` | IS | `game-engine`: if player in active game → emit `PlayerDisconnected`, start 60s reconnection timer (but reconnect will fail since session is invalid → eventual forfeit) |
| `tournament.rooms` | `TournamentRoomAssigned` | TO | `room-service`: create tournament room with assigned players, auto-join all, auto-start match |
| `gameplay.games` | `GameCompleted` | self (game-engine) | `room-service`: evaluate match scoreline → start next game or emit `MatchCompleted` |
| `gameplay.events` | `PlayerForfeited` | self (game-engine) | `room-service`: update room/match state, may trigger `GameCompleted` if only 1 player remains |

---

## Internal-Only Interfaces

| Interface | Description |
|-----------|-------------|
| `game-engine` ↔ `timer-service` | `game-engine` writes deadline rows to `timer_deadlines` table (shared DB or dedicated). `timer-service` polls and fires expiry commands back to `game-engine` via internal queue or direct RPC. |
| Outbox relay | Background worker within `game-engine` deployment reads `outbox` table, publishes to Kafka, marks as delivered. Polling interval: 50ms. |
| `room-service` → `game-engine` | Internal RPC: `InitializeGame { gameId, roomId, players[], gameNumber }`. Triggers the deck initialization pipeline. |

---

## Dependencies on Other Contexts

| Upstream Context | Dependency Type | Contract |
|-----------------|----------------|----------|
| Identity & Session | Token validation (sync query to Redis cache or `/sessions/validate`), `SessionInvalidated` event consumption | OHS / Published Language |
| Tournament Orchestration | `TournamentRoomAssigned` event consumption | Published Language (TO is upstream for room assignment) |

---

## Invariant: Log-Before-Broadcast (§6.1 mandatory)

Every authoritative state change is durably appended to the immutable game log **before** any broadcast to players, spectators, or downstream consumers.

### Mechanism: Event Sourcing + Transactional Outbox

```
┌────────────┐     PlayCard(gameId, cardId, seq=42)     ┌───────────────────┐
│   Player   │ ────────────────────────────────────────► │   game-engine     │
│  (via WS)  │                                          │                   │
└────────────┘                                          │ 1. Load Game      │
                                                        │    aggregate from │
                                                        │    event store    │
                                                        │    (replay events │
                                                        │    or use cache)  │
                                                        │                   │
                                                        │ 2. Validate:      │
                                                        │    - seq == 42 ✓  │
                                                        │    - is your turn ✓│
                                                        │    - legal play ✓ │
                                                        │                   │
                                                        │ 3. BEGIN TX       │
                                                        │    INSERT INTO    │
                                                        │    game_events (  │
                                                        │      gameId,      │
                                                        │      seq=42,      │
                                                        │      type=        │
                                                        │      'CardPlayed',│
                                                        │      payload,     │
                                                        │      signature    │
                                                        │    )              │
                                                        │    INSERT INTO    │
                                                        │    outbox (       │
                                                        │      eventId,     │
                                                        │      topic,       │
                                                        │      payload      │
                                                        │    )              │
                                                        │ 4. COMMIT TX      │
                                                        │    ───────────────│──── Log is now durable.
                                                        │                   │     Crash-safe.
                                                        │ 5. ACK to player  │
                                                        │    via WebSocket  │
                                                        └───────┬───────────┘
                                                                │
                                                    ┌───────────▼────────────┐
                                                    │   Outbox Relay Worker  │
                                                    │   (polls every 50ms)  │
                                                    │                       │
                                                    │ 6. Read undelivered   │
                                                    │    outbox rows        │
                                                    │ 7. Publish to Kafka   │
                                                    │    gameplay.events    │
                                                    │ 8. Mark as delivered  │
                                                    └───────────┬───────────┘
                                                                │
                                                    ┌───────────▼───────────┐
                                                    │       Kafka           │
                                                    │  topic: gameplay.*    │
                                                    └───────┬───────┬──────┘
                                                            │       │
                                                            ▼       ▼
                                                     Spectator   Ranking
                                                     Audit       Tournament
```

**Why this satisfies the invariant:**
- Steps 3–4 are a single PostgreSQL transaction. If the process crashes after step 3 but before step 4, the TX rolls back — no event is persisted, no broadcast happens.
- If the process crashes after step 4 but before step 7, the event is durable in `game_events` and in `outbox`. The outbox relay will pick it up on restart and publish to Kafka.
- Clients never see an update that isn't in the log. The WebSocket ACK (step 5) happens after COMMIT (step 4).

**Sequence diagram:** See [diagrams/seq-log-before-broadcast.md](../diagrams/seq-log-before-broadcast.md).

---

## Invariant: Sequence-Number Enforcement

**Owner:** `game-engine` — command handler layer.

**Mechanism:**
1. Client sends command with `sequenceNumber = N`.
2. `game-engine` loads Game aggregate. Checks `game.expectedSequenceNumber == N`.
3. If mismatch: reject with `CommandRejected { reason: "stale_sequence_number", expected: game.expectedSequenceNumber }`. Client reconciles state from the event stream and retries.
4. If match: process command, increment `expectedSequenceNumber` to `N+1`, persist events.

**Replay protection:** Idempotency check on `commandId` runs *before* sequence-number check. If `commandId` is already in the idempotency ledger, return cached response — even if sequence number is now stale.

**Exception:** `CallUno`, `ChallengeUnoCall`, `ChallengeWildDrawFour` do not require sequence numbers (time-sensitive, accepted outside normal turn sequence). They are guarded by challenge window state instead.

---

## Invariant: 5-Second Uno! Challenge Window (Timer Durability)

### Scheduling

1. When a player plays their penultimate card, `game-engine` emits `UnoChallengeWindowOpened { gameId, targetPlayerId, windowId, expiresAt }`.
2. In the same TX, inserts a row into `timer_deadlines`:
   ```
   { deadlineId: windowId, gameId, type: "uno_challenge", expiresAt: now()+5s, version: 1, fired: false }
   ```
3. `timer-service` polls `timer_deadlines` for rows where `expiresAt <= now() AND fired = false`.

### Expiry

4. `timer-service` reads the deadline row, sends `ChallengeWindowExpired { gameId, windowId, version: 1 }` to `game-engine`.
5. `game-engine` checks: is the challenge window still open? Is the `version` current? (The window might have been closed early by a challenge.)
6. If valid: emit `UnoChallengeWindowClosed`, apply no penalty (no challenge was made).
7. If already closed (challenge resolved it, or version mismatch): discard — idempotent no-op.

### Crash Recovery

- If `timer-service` crashes mid-window: on restart, it re-scans `timer_deadlines` for unfired rows with `expiresAt <= now()`. The deadline is recovered. Worst case: the expiry fires slightly late (by restart time), but the game-engine enforces the 5-second window server-side — a late-arriving challenge is rejected by timestamp comparison in the Game aggregate.
- If `game-engine` crashes mid-window: on restart, the Game aggregate is rebuilt from event store. If `UnoChallengeWindowOpened` is in the log but no `UnoChallengeWindowClosed`, the window is still logically open. The deadline row in `timer_deadlines` will fire normally.

### Idempotency

- `windowId` + `version` form the idempotency key. If `ChallengeWindowExpired` is delivered twice (at-least-once), the second delivery sees `version` has advanced (or window is closed) and is discarded.

**Same pattern applies to:** WDF challenge window (5s), turn timer (30/60s). Only deadline type and expiry command differ.

---

## Invariant: 60-Second Reconnection Window (Timer Durability)

### Scheduling

1. On `PlayerDisconnected` (detected by gateway heartbeat miss or `SessionInvalidated` consumption), `game-engine` emits `PlayerDisconnected { gameId, playerId, disconnectedAt }`.
2. In same TX, inserts `timer_deadlines`:
   ```
   { deadlineId: reconnWindowId, gameId, type: "reconnection", playerId, expiresAt: disconnectedAt+60s, version: 1, fired: false }
   ```

### During the Window

- If it becomes the disconnected player's turn, `game-engine` executes `AutoSkipTurn` (no draw, just skip and advance).
- The player's turn timer does NOT run while disconnected.

### Expiry

3. `timer-service` fires `ReconnectionTimerExpired { gameId, playerId, reconnWindowId, version }`.
4. `game-engine` validates window is still open, then executes `AutoForfeit`:
   - Emits `PlayerForfeited { gameId, playerId, reason: "reconnection_timeout" }`.
   - If only 1 player remains: `GameCompleted` → `MatchCompleted` → `RoomCompleted`.

### Cancellation (Reconnect)

5. If player reconnects within 60s via `Reconnect` command:
   - `game-engine` emits `PlayerReconnected`, updates deadline row: `SET fired = true` (or version increment).
   - Subsequent `ReconnectionTimerExpired` from `timer-service` is discarded (version mismatch).

### Crash Recovery

- Same as Uno challenge window: `timer-service` rescans on restart. Deadline is persisted, not in-memory.
- If `game-engine` crashes: aggregate rebuilt from event store. If `PlayerDisconnected` is in log without `PlayerReconnected` or `PlayerForfeited`, the reconnection window is logically open. Timer fires normally.

---

## Invariant: Match Series Coordination

**Owner:** `room-service` — Match entity within Room aggregate.

### State Machine

```
MatchNotStarted
  │
  ├─ Game 1 Started ──► Game 1 InProgress ──► Game 1 Completed
  │                                                    │
  │   [2-player room: if player has 2 wins → MatchCompleted (early termination)]
  │   [multi-player room: always continue to next game unless game 3 done]
  │                                                    │
  ├─ Game 2 Started ──► Game 2 InProgress ──► Game 2 Completed
  │                                                    │
  │   [2-player room: if player has 2 wins → MatchCompleted]
  │                                                    │
  ├─ Game 3 Started ──► Game 3 InProgress ──► Game 3 Completed
  │                                                    │
  └─ MatchCompleted (always, after game 3)
```

### Mechanism

1. `room-service` subscribes to `gameplay.games` topic (partition key: `gameId`).
2. On `GameCompleted { gameId, placements }`:
   - Look up which room/match this game belongs to.
   - Update Match entity: increment wins for game winner, add card points.
   - If `gameNumber < 3`:
     - **2-player room:** Check if any player has 2 wins. If yes → early termination, emit `MatchCompleted`.
     - **Multi-player room:** Always proceed to next game.
     - If continuing: emit `MatchGameCompleted { matchId, gameNumber, nextGameNumber }`, issue internal `InitializeGame` to `game-engine`.
   - If `gameNumber == 3`: compute final match ranking (matchWins → cardPoints → completionTime), emit `MatchCompleted { matchId, roomId, rankings }`, then `RoomCompleted`.

3. `MatchCompleted` includes full ranking:
   ```
   rankings: [
     { playerId, matchWins, cumulativeCardPoints, finalGameCompletionTime, placement: 1 },
     { playerId, matchWins, cumulativeCardPoints, finalGameCompletionTime, placement: 2 },
     ...
   ]
   ```

### Cross-Game State

The Match entity persists:
- `gameResults[]` — per-game placements and card points.
- `perPlayerWins{}` — running tally.
- `currentGameNumber` — which game is active.
- `matchStatus` — NotStarted / InProgress / Completed.

This state survives `room-service` restarts (persisted in PostgreSQL).

---

## Invariant: Abandoned-Game vs. Completed-Game Outcomes

### Detection

**Owner:** `game-engine` — Game aggregate.

A game is marked as **abandoned** when all remaining active players forfeit (no player wins by gameplay). The `GameCompleted` event carries:
```
{
  wasAbandoned: boolean,        // true if all remaining players forfeited
  roomType: "casual" | "tournament",
  placements: [...],
  ...
}
```

- **`wasAbandoned: false`:** At least one player won by emptying their hand, or one player remains after others forfeit.
- **`wasAbandoned: true`:** All remaining players forfeited (directly or via reconnection timeout). No winner.

### Downstream Impact

| Consumer | `!wasAbandoned` + `casual` | `!wasAbandoned` + `tournament` | `wasAbandoned` + `casual` | `wasAbandoned` + `tournament` |
|----------|----------------------|--------------------------|----------------------|--------------------------|
| `ranking-service` (Elo) | ✅ Compute Elo deltas | ❌ Skip (no Elo for tournaments) | ❌ Skip (no Elo for abandoned) | ❌ Skip |
| `tournament-service` | N/A (casual) | ✅ Record result, advance top 3 | N/A (casual) | ✅ Record all as eliminated (forfeit losses) |
| `ranking-service` (stats) | ✅ Update statistics | ✅ Update statistics | ✅ Update stats (increment abandoned count) | ✅ Update stats |

The `wasAbandoned` and `roomType` fields in `GameCompleted` are the canonical discriminators. Downstream consumers filter on these — they do not infer outcome from other signals.

For tournaments, a forfeit during a match is recorded as a loss for advancement purposes (the player is eliminated). The `PlayerForfeited` event carries `tournamentId` when applicable, allowing `tournament-service` to immediately emit `PlayerEliminated`.

---

## Persistence

| Store | Technology | Data | Consistency |
|-------|-----------|------|-------------|
| Event Store | PostgreSQL | `game_events` (gameId, sequenceNumber, eventType, payload, signature, timestamp). Append-only. Partitioned by gameId. | Strong (sequential writes per game) |
| Outbox | PostgreSQL (same DB) | `outbox` (eventId, topic, partitionKey, payload, delivered). Shared TX with event store. | Strong (same TX as event append) |
| Room State | PostgreSQL | `rooms` (roomId, status, hostPlayerId, maxPlayers, roomType), `player_slots` (roomId, playerId, joinedAt), `matches` (matchId, roomId, currentGameNumber, perPlayerWins, status) | Strong (per-room) |
| Timer Deadlines | PostgreSQL (same or dedicated) | `timer_deadlines` (deadlineId, gameId, type, expiresAt, version, fired) | Strong (polled by timer-service) |
| Idempotency | PostgreSQL | `command_idempotency` (commandId, gameId, response, createdAt). TTL: 24h. | Strong |
| Game State Cache | In-memory (per game-engine instance) | Materialized Game aggregate from event replay. Invalidated on instance loss; rebuilt from event store. | Derived (rebuildable) |

### Read Path for Immutable Game Log (Dispute Resolution / Audit)

The `game_events` table is the authoritative source for game replay and dispute resolution.

**Who may query:**
- **Operators / support agents:** Via `audit-service` API (`GET /api/v1/audit/games/{gameId}/log`). Requires `role: operator` or `role: admin` in JWT.
- **Automated replay jobs:** Internal service with mTLS. Used for anti-cheat analysis, statistical validation.
- **Compliance / legal:** Break-glass access via `audit-service` with elevated role (`role: compliance`). Logged separately in audit trail.

**Access control:**
- All queries go through `audit-service`, which has a read replica of game logs (or queries `game_events` directly via read replica).
- No direct DB access for operators. All access is via scoped APIs with role-based authorization.
- Every query to the game log API is itself logged in the audit trail (who queried, when, which game).

**Retention:** Game logs are retained for the configurable retention period (default: 1 year). After retention, logs may be archived to cold storage but are not deleted.
