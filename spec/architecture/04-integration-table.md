# 4. Integration Table — Communication Patterns

> Documents every significant inter-component integration in UnoArena. Each row traces to named commands/events from the [Design Checkpoint](../domain/docs/04-commands-and-domain-events.md). Corresponds to §6.3 of the Architecture Checkpoint.

---

## 4.1 Synchronous Integrations (REST / RPC)

| # | From → To | Pattern | Rationale | Failure Semantics |
|---|-----------|---------|-----------|-------------------|
| S1 | `Player` → `api-gateway` → `identity-service` | REST `POST /sessions` | Login requires immediate JWT response. No async alternative. | 3 s timeout. 401/403 returned to client. Rate-limited per IP (Layer 1) to prevent brute-force. |
| S2 | `Player` → `api-gateway` → `identity-service` | REST `POST /players` | Registration needs synchronous uniqueness check on display name / email. | 3 s timeout. 409 on conflict. Idempotent via `commandId`. |
| S3 | `Player` → `api-gateway` → `room-service` | REST `POST /rooms`, `POST /rooms/{id}/join` | Room creation/join needs immediate slot confirmation. | 2 s timeout. 409 if room full. Idempotent via `commandId`. |
| S4 | `Player` → `api-gateway` → `game-engine` | WebSocket relay: `play_card`, `draw_card`, `call_uno`, etc. | Bidirectional gameplay commands need sub-100 ms round-trip. WebSocket avoids per-message HTTP overhead. | Command rejected with reason code over WS. Sequence-number mismatch → `CommandRejected { reason: "stale_sequence_number" }`. Client reconciles and retries. |
| S5 | `Player` → `api-gateway` → `game-engine` | REST `GET /games/{gameId}/state` | Reconnection: player needs full game state snapshot. | 2 s timeout. 404 if game not found. Auth: participant-only (JWT). |
| S6 | `api-gateway` → `identity-service` (Redis cache) | Read-through token validation | Every request: gateway checks JWT validity against Redis session cache. Fast path avoids DB hit. | Cache miss → gateway calls `POST /sessions/validate` (mTLS, internal). 200 ms timeout. On total failure → reject request (fail-closed). |
| S7 | `Organizer` → `api-gateway` → `tournament-service` | REST `POST /tournaments`, `POST /tournaments/{id}/start` | Tournament lifecycle commands need synchronous validation (authorization, state checks). | 3 s timeout. 403 if not organizer. 409 if wrong state. |
| S8 | `Spectator` → `api-gateway` → `spectator-projection-service` | REST `GET /spectator/games/{id}`, `GET /lobby/rooms`, `GET /tournaments/{id}/bracket` | Read queries against pre-materialized projections. Low latency. | 1 s timeout. Stale data acceptable (≤ 1 s). Cache-friendly (HTTP cache headers). |
| S9 | `Admin` → `api-gateway` → `audit-service` | REST `GET /audit/games/{id}/log`, `POST /audit/trail/export` | Audit queries need synchronous response. Role-gated. | 5 s timeout (large result sets). 403 if insufficient role. Paginated. |
| S10 | `room-service` → `game-engine` | Internal RPC: `InitializeGame { gameId, roomId, players[], gameNumber }` | Match coordination: room-service tells game-engine to start a new game in a series. Synchronous ensures game initialization before match state advances. | 2 s timeout. Retry with exponential backoff (3 attempts). Idempotent via `gameId`. On permanent failure → room marked degraded, error event emitted. |
| S11 | `Spectator` → `api-gateway` → `spectator-projection-service` | SSE `GET /spectator/games/{id}/stream` | Live spectator feed. Long-lived connection. Read-only. Public. | Connection drop → client auto-reconnects with `Last-Event-ID`. Server rebuilds from projection. Per-IP connection limit (20 SSE). |
| S12 | `Player` → `api-gateway` → `ranking-service` | REST `GET /leaderboard`, `GET /players/{id}/rating` | Leaderboard and rating queries. Served from Redis sorted set (leaderboard) or PostgreSQL (rating history). | 1 s timeout. Redis fallback to DB on cache miss. |

---

## 4.2 Asynchronous Event Propagation (Kafka Pub/Sub)

| # | From → To | Pattern | Rationale | Failure Semantics |
|---|-----------|---------|-----------|-------------------|
| A1 | `identity-service` → `api-gateway` | Pub/Sub: `SessionInvalidated` on `identity.sessions` (key: `playerId`) | Push-invalidation: gateway must close superseded session's WebSocket/SSE promptly, not wait for token expiry. | At-least-once. Gateway deduplicates by `eventId`. If gateway misses event, heartbeat (30 s) catches stale token. Consumer lag target: < 200 ms. |
| A2 | `identity-service` → `game-engine` | Pub/Sub: `SessionInvalidated` on `identity.sessions` | If player in active game → emit `PlayerDisconnected`, start 60 s reconnection timer. Since session is invalid, reconnect with old token fails → eventual forfeit. | At-least-once. Idempotent: if player already disconnected, event is no-op. |
| A3 | `identity-service` → `ranking-service` | Pub/Sub: `PlayerRegistered` on `identity.sessions` | Initialize `PlayerRating` (Elo = 1200) and `PlayerStatistics` (zeroed). Per design INV-PR-06. | At-least-once. Idempotent via `(playerId)` unique constraint on `player_ratings`. Duplicate insert → constraint violation → skip. |
| A4 | `identity-service` → `tournament-service` | Pub/Sub: `PlayerSuspended`, `PlayerBanned` on `identity.sessions` | Disqualify suspended/banned players from active tournaments. | At-least-once. Idempotent: if player already eliminated, no-op. |
| A5 | `game-engine` → `room-service` | Pub/Sub: `GameCompleted` on `gameplay.games` (key: `gameId`) | Room-service evaluates match scoreline. On match end → `MatchCompleted`, `RoomCompleted`. | At-least-once. Idempotent via `(matchId, gameId)` in `match_game_results`. Duplicate → skip. |
| A6 | `game-engine` → `ranking-service` | Pub/Sub: `GameCompleted` on `gameplay.games` | Elo pipeline: filter casual + non-abandoned → compute pairwise Elo → persist. | At-least-once. Idempotent via `(playerId, gameId)` in `processed_games`. Per-player atomic TX. Partial failure → redelivery completes remaining players. |
| A7 | `game-engine` → `spectator-projection-service` | Pub/Sub: all events on `gameplay.events` (key: `gameId`) | ACL transformation → spectator read model materialization → SSE push. | At-least-once. Idempotent via `eventId` dedup + projection version check. Consumer group scales with partition count. |
| A8 | `game-engine` → `audit-service` | Pub/Sub: all events on `gameplay.events` + `gameplay.games` | Universal audit trail. HMAC verification. Append-only ingestion. | At-least-once. Idempotent via `eventId` in `processed_events`. Append-only — duplicate insert rejected by constraint. |
| A9 | `room-service` → `tournament-service` | Pub/Sub: `RoomCompleted` on `gameplay.rooms` (key: `roomId`) | Increment completion counter. If `completedRooms == totalRooms` → trigger advancement + next round. | At-least-once. Idempotent via `(roundId, roomId)` unique constraint in `tournament_room_results`. Duplicate does not increment counter. |
| A10 | `room-service` → `tournament-service` | Pub/Sub: `MatchCompleted` on `gameplay.rooms` | Record match result for tournament room (rankings, placements). | At-least-once. Idempotent via `(roundId, matchId)`. |
| A11 | `game-engine` → `tournament-service` | Pub/Sub: `PlayerForfeited` on `gameplay.events` | If tournament room: emit `PlayerEliminated`. Forfeit = loss for advancement. | At-least-once. Idempotent: if player already eliminated, no-op. |
| A12 | `round-kickoff-worker` → `room-service` | Pub/Sub: `TournamentRoomAssigned` on `tournament.rooms` (key: `roomId`) | Room creation for tournament round. High burst at round start (~10k rooms/sec). | At-least-once. Idempotent via `roomId` (pre-generated). Duplicate creates no duplicate room. DLQ for permanently failed room creation. |
| A13 | `tournament-service` → `spectator-projection-service` | Pub/Sub: `TournamentRoundCreated`, `PlayerAdvanced`, `PlayerEliminated`, `TournamentCompleted` on `tournament.lifecycle` | Bracket projection materialization. | At-least-once. Idempotent via `eventId`. Projection versioned. |
| A14 | `ranking-service` → `audit-service` | Pub/Sub: `EloUpdated`, `LeaderboardUpdated` on `ranking.updates` | Audit trail for rating changes. | At-least-once. Idempotent via `eventId`. |
| A15 | `room-service` → `spectator-projection-service` | Pub/Sub: `RoomCreated`, `PlayerJoinedRoom`, `RoomCompleted` on `gameplay.rooms` | Lobby read model (available casual rooms). Tournament rooms excluded from lobby. | At-least-once. Idempotent. Staleness ≤ 1 s. |
| A16 | `ranking-service` → `audit-service` | Pub/Sub: `PlayerStatisticsUpdated` on `ranking.updates` | Statistics change audit trail. | At-least-once. Idempotent via `eventId`. |

---

## 4.3 Transactional Outbox / Log-Before-Broadcast

| # | From → To | Pattern | Rationale | Failure Semantics |
|---|-----------|---------|-----------|-------------------|
| O1 | `game-engine` → Kafka (`gameplay.events`, `gameplay.games`) | **Transactional Outbox** — event store append + outbox row in same PostgreSQL TX. Outbox relay worker polls every 50 ms, publishes to Kafka, marks delivered. | **Log-before-broadcast invariant:** every authoritative state change is durably persisted before any broadcast. Direct Kafka produce cannot guarantee atomicity with DB write. | Crash before COMMIT → TX rolls back, nothing persisted, client retries with same `commandId`. Crash after COMMIT, before Kafka publish → outbox relay picks up on restart, publishes. At-least-once delivery to Kafka. Consumers deduplicate by `eventId`. |
| O2 | `identity-service` → Kafka (`identity.sessions`) | **Transactional Outbox** — session CAS (invalidate old + create new) + outbox row in same SERIALIZABLE TX. | Guarantees `SessionInvalidated` is published even if Kafka is temporarily unavailable at login time. Critical for push-invalidation path. | Same as O1. Outbox relay retries until Kafka available. Gateway heartbeat (30 s) is a secondary safety net. |

---

## 4.4 CQRS / Read Model Projections

| # | From → To | Pattern | Rationale | Failure Semantics |
|---|-----------|---------|-----------|-------------------|
| C1 | `gameplay.events` → `spectator-projection-service` → `SpectatorGameProjection` | **CQRS read model** — event-carried state transfer. ACL transformation strips hands/deck/seed. Materialized as denormalized document per game. | Spectators need a privacy-safe, low-latency view of active games. Cannot serve raw events (privacy). Cannot query game-engine directly (coupling + load). | Staleness ≤ 500 ms. Projection rebuilt from Kafka replay on consumer restart (earliest offset for active games). Missing events → projection gap → client sees stale state until next event. |
| C2 | `gameplay.rooms` → `spectator-projection-service` → `AvailableRooms` | **CQRS read model** — lobby listing of casual rooms with `status = waiting`. | Pre-materialized for fast lobby page load. Filters out tournament rooms. | Staleness ≤ 1 s. Room disappears from lobby on `RoomFilled` or `RoomCompleted`. |
| C3 | `tournament.lifecycle` → `spectator-projection-service` → `TournamentBracket` | **CQRS read model** — bracket projection with player advancement/elimination. | Denormalized for fast bracket rendering. No privacy concerns (all public). | Staleness ≤ 1 s. Updated on `PlayerAdvanced`, `PlayerEliminated`, `TournamentRoundCreated`. |
| C4 | `gameplay.games` → `ranking-service` → Redis Sorted Set (`leaderboard:global`) | **CQRS cache** — `ZADD leaderboard:global playerId newElo` after each Elo update. | Sub-millisecond leaderboard queries. Redis `ZRANGEBYSCORE` + `ZRANK`. | Staleness ≤ 1 s. If Redis unavailable, fall back to PostgreSQL query (slower). Cache rebuilt from DB on Redis restart. |

---

## 4.5 Sagas / Process Managers

| # | From → To | Pattern | Rationale | Failure Semantics |
|---|-----------|---------|-----------|-------------------|
| P1 | `tournament-service` → `round-kickoff-worker` → `room-service` | **Choreographed saga: Round Kickoff** — `tournament-service` enqueues work items to `tournament.kickoff-work`. Workers publish `TournamentRoomAssigned` to `tournament.rooms`. `room-service` creates rooms, emits `RoomCreated`. | 100k rooms in ~10 s. Sharded workers provide rate-limited fan-out. No single choke point. | Worker crash → Kafka consumer group rebalances, surviving workers resume from last offset. Room creation failure → DLQ + `tournament-service` monitors missing `RoomCreated` (5 min timeout) → force-resolve or substitute. All steps idempotent. |
| P2 | `room-service` → `tournament-service` → (next round) | **Choreographed saga: Round Advancement** — `RoomCompleted` → atomic counter increment → if complete → evaluate advancement → `PlayerAdvanced`/`PlayerEliminated` → `CreateRound` or `CreateFinalRoom` → kickoff. | Multi-step cross-context workflow. Each step is event-driven, independently retriable. | Counter idempotent via `(roundId, roomId)` unique constraint. Advancement evaluation is deterministic (same inputs → same outputs). If advancement fails, `tournament-service` retries on redelivery. |
| P3 | `room-service` internal: **Match Series Coordinator** | **Orchestrated process manager** — `GameCompleted` → evaluate scoreline → if `gameNumber < 3` and no early termination → `InitializeGame` (internal RPC to `game-engine`) → `MatchGameCompleted`. If series over → `MatchCompleted` → `RoomCompleted`. | Cross-game state within a match (best-of-3). Game aggregate doesn't know about match context. Room-service orchestrates. | `InitializeGame` RPC: 3 retries, 2 s timeout. Idempotent via `gameId`. If permanent failure → match marked abandoned, `MatchCompleted { wasAbandoned: true }`. Match state persisted in PostgreSQL (survives restart). |

---

## 4.6 Session Invalidation → Live Connection

| # | From → To | Pattern | Rationale | Failure Semantics |
|---|-----------|---------|-----------|-------------------|
| SI1 | `identity-service` → Kafka → `api-gateway` | **Pub/Sub push-invalidation** — `SessionInvalidated { sessionId, playerId, reason: "new_login" }` on `identity.sessions`. Gateway subscribes, looks up `sessionId` in in-memory `connectionMap`, sends WebSocket close frame (code 4001: `session_superseded`) or terminates SSE stream. | Revoking a DB token is insufficient — live connections must be terminated promptly. Push-invalidation via Kafka closes connection within ~200 ms of new login. | Gateway misses event (consumer lag) → heartbeat (30 s) catches stale token. Gateway crash → client TCP dies anyway; on reconnect, old token is invalid in DB. No entry in connectionMap (player not connected) → harmless no-op. |
| SI2 | `identity-service` → Kafka → `game-engine` | **Pub/Sub** — `SessionInvalidated` consumed by `game-engine`. If player in active game → emit `PlayerDisconnected`, start 60 s reconnection timer. Since session is invalid, reconnect attempt with old token fails at gateway → timer expires → `PlayerForfeited`. | Ensures in-game players are properly disconnected when session is superseded, triggering the reconnection window and eventual forfeit. | Idempotent: if player already disconnected (e.g., gateway closed connection first), `PlayerDisconnected` is a no-op (already in disconnected state). |

---

## 4.7 Timer / Domain-Window Management

| # | From → To | Pattern | Rationale | Failure Semantics |
|---|-----------|---------|-----------|-------------------|
| T1 | `game-engine` → `timer_deadlines` table | **Durable timer scheduling** — in same TX as event store append: INSERT `{ deadlineId: windowId, gameId, type: "uno_challenge", expiresAt: now()+5s, version: 1, fired: false }`. | Timer must survive process crashes. In-memory `setTimeout` is lost on crash. Persisted deadline is recovered on restart. | Same TX as game event → if TX fails, neither event nor timer persists (atomic). |
| T2 | `game-engine` → `timer_deadlines` table | **Durable timer scheduling: reconnection** — INSERT `{ type: "reconnection", expiresAt: disconnectedAt+60s }` in same TX as `PlayerDisconnected` event. | 60 s reconnection window. Must survive game-engine or timer-service crashes. | Same atomicity as T1. Deadline row is source of truth. |
| T3 | `game-engine` → `timer_deadlines` table | **Durable timer scheduling: turn timer** — INSERT `{ type: "turn_timer", expiresAt: now()+30s }` (casual) or `+60s` (tournament) in same TX as `TurnAdvanced`. | Turn timeout forces auto-draw/auto-pass. Must be durable. | Same atomicity as T1. |
| T4 | `timer-service` → `game-engine` | **Poll + fire: expiry command** — `timer-service` polls `timer_deadlines WHERE expiresAt <= now() AND fired = false` (sharded by time bucket). Fires `ChallengeWindowExpired { gameId, windowId, version }` or `ReconnectionTimerExpired { gameId, playerId, reconnWindowId, version }` or `TurnTimerExpired { gameId, version }`. | Decoupled timer execution. `timer-service` only owns scheduling; Game aggregate owns validation. | **Idempotent:** `game-engine` checks `version` match and window state. If window already closed (challenge resolved, player reconnected, or version advanced) → discard. Duplicate fire is harmless. **Crash recovery:** `timer-service` rescans on restart. Late fire (by restart time) is acceptable — Game aggregate enforces timestamp-based window validity. |
| T5 | `game-engine` (on `Reconnect` command) → `timer_deadlines` | **Timer cancellation** — `PlayerReconnected` event + UPDATE `timer_deadlines SET fired = true` (or version increment). | Cancels 60 s reconnection timer when player reconnects within window. Subsequent `ReconnectionTimerExpired` from `timer-service` is discarded (version mismatch). | Atomic with `PlayerReconnected` event (same TX). If crash before commit → reconnection not recorded, timer fires normally → forfeit (safe, player can reconnect again). |

---

## 4.8 Summary Statistics

| Category | Row Count | Key Patterns |
|----------|-----------|--------------|
| Synchronous (REST/WS) | 12 | Request/response, WebSocket relay |
| Async Event Propagation | 16 | Kafka pub/sub with consumer groups |
| Transactional Outbox | 2 | Log-before-broadcast, session CAS |
| CQRS / Read Models | 4 | Event-carried state transfer, Redis cache |
| Sagas / Process Managers | 3 | Choreographed (round kickoff/advancement), orchestrated (match series) |
| Session Invalidation | 2 | Push-invalidation via Kafka |
| Timer / Window Management | 5 | Durable deadlines, poll-and-fire, idempotent expiry |
| **Total** | **44** | |

---

## 4.9 Traceability Note

Every integration row above traces to named commands and events in the [Commands & Events Catalog](../domain/docs/04-commands-and-domain-events.md):

- **Synchronous endpoints** map to commands: `RegisterPlayer`, `Login`, `CreateRoom`, `JoinRoom`, `PlayCard`, `DrawCard`, `CallUno`, `CreateTournament`, `StartTournament`.
- **Async events** match the published event catalog: `SessionInvalidated`, `GameCompleted`, `MatchCompleted`, `RoomCompleted`, `PlayerForfeited`, `TournamentRoomAssigned`, `PlayerAdvanced`, `EloUpdated`, etc.
- **Timer events** map to design value objects: `UnoChallengeWindow`, `WildDrawFourChallengeWindow`, `ReconnectionWindow` (doc 03 §3.1.5).
- Deltas from the Design Checkpoint are documented in [CHANGELOG-design.md](04-CHANGELOG-design.md).
