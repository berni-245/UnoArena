# 4. Integration Table — Communication Patterns

> Documents every significant inter-component integration in UnoArena. Each row traces to named commands/events from the [Design Checkpoint](../../domain/docs/04-commands-and-domain-events.md). Corresponds to §6.3 of the Architecture Checkpoint.

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
| S8 | `Spectator` → `api-gateway` → `spectator-projection-service` | REST `GET /spectator/games/{id}`, `GET /lobby/rooms`, `GET /tournaments/{id}/bracket` | Read queries against pre-materialized projections. Low latency. | 1 s timeout. Stale data acceptable (≤ 1 s). Cache-friendly (HTTP cache headers). **Auth conditional on room visibility:** `public` / `tournament` / `unlisted` rooms are anonymous; `private` rooms require Bearer JWT and the JWT's `playerId` must appear on the host-managed invitee list (enforced by `spectator-projection-service`, 403 otherwise). See `spectator-view.md` §Public Interfaces. |
| S9 | `Admin` → `api-gateway` → `audit-service` | REST `GET /audit/games/{id}/log`, `POST /audit/trail/export` | Audit queries need synchronous response. Role-gated. | 5 s timeout (large result sets). 403 if insufficient role. Paginated. |
| S10 | `room-service` → `game-engine` | Internal RPC: `InitializeGame { gameId, roomId, players[], gameNumber }` | Match coordination: room-service tells game-engine to start a new game in a series. Synchronous ensures game initialization before match state advances. | 2 s timeout. Retry with exponential backoff (3 attempts). Idempotent via `gameId`. On permanent failure → room marked degraded, error event emitted. |
| S11 | `Spectator` → `api-gateway` → `spectator-projection-service` | SSE `GET /spectator/games/{id}/stream` | Live spectator feed. Long-lived connection. Read-only. Anonymous for `public`/`tournament`/`unlisted` rooms. | Connection drop → client auto-reconnects with `Last-Event-ID`. Server rebuilds from projection. Per-IP connection limit (20 SSE). **Auth for `private` rooms:** Bearer JWT required; `spectator-projection-service` validates the JWT's `playerId` against the invitee list before opening the SSE stream (403 `forbidden_not_invited` otherwise). |
| S13 | `Spectator` → `regional-edge-proxy` → `spectator-projection-service` | **SSE fan-out via regional edge proxy** — for games with >1k spectators, `spectator-projection-service` opens 1 upstream SSE connection per edge region per game. Each `regional-edge-proxy` fans out to up to 10k local spectator connections. Reduces upstream SSE connections from 100k to ~10 (one per region) for tournament finals. Only activated for `public` / `tournament` rooms (private rooms never reach edge fan-out — they always go through `spectator-projection-service` direct so the invitee ACL stays authoritative). | High-spectator rooms (tournament finals with 100k+ spectators). Reduces load on `spectator-projection-service` by 1000×. | Per-IP rate limits still enforced at `regional-edge-proxy`. Proxy auto-subscribes when spectator count exceeds threshold; auto-terminates when room completes. **Upstream SSE failover:** on connection drop, proxy reconnects with `Last-Event-ID` to any healthy `spectator-projection-service` instance (load-balanced, not affinity-pinned); new instance replays events from `sequenceNumber > last_event_id` using the materialized read model (index read, no Kafka replay required). Worst-case gap during consumer-group rebalance: ≤ 30 s. See `spectator-view.md` §Upstream SSE Failover for full failure-path analysis. |
| S12 | `Player` → `api-gateway` → `ranking-service` | REST `GET /leaderboard`, `GET /players/{id}/rating` | Leaderboard and rating queries. Served from Redis sorted set (leaderboard) or PostgreSQL (rating history). | 1 s timeout. Redis fallback to DB on cache miss. |

---

## 4.2 Asynchronous Event Propagation (Kafka Pub/Sub)

| # | From → To | Pattern | Rationale | Failure Semantics |
|---|-----------|---------|-----------|-------------------|
| A1 | `identity-service` → `api-gateway` | Pub/Sub: `SessionInvalidated` on `identity.sessions` (key: `playerId`) | Push-invalidation: gateway must close superseded session's WebSocket/SSE promptly, not wait for token expiry. | At-least-once. Gateway deduplicates by `eventId`. If gateway misses event, heartbeat (30 s) catches stale token. Consumer lag target: < 200 ms. |
| A2 | `identity-service` → `game-engine` | Pub/Sub: `SessionInvalidated` on `identity.sessions` | If player in active game → emit `PlayerDisconnected`, start 60 s reconnection timer. Since session is invalid, reconnect with old token fails → eventual forfeit. | At-least-once. Idempotent: if player already disconnected, event is no-op. |
| A3 | `identity-service` → `ranking-service` | Pub/Sub: `PlayerRegistered` on `identity.sessions` | Initialize `PlayerRating` (Elo = 1200) and `PlayerStatistics` (zeroed). Per design INV-PR-06. | At-least-once. Idempotent via `(playerId)` unique constraint on `player_ratings`. Duplicate insert → constraint violation → skip. |
| A4 | `identity-service` → `tournament-service`, `game-engine`, `room-service` | Pub/Sub: `PlayerSuspended`, `PlayerBanned` on `identity.sessions` | `tournament-service`: disqualify suspended/banned player from active tournaments. `game-engine`: if player in active game → emit `PlayerForfeited` immediately (no reconnection window). `room-service`: if player in waiting room → remove from slot. | At-least-once. Idempotent: each consumer checks current state before acting (already eliminated/removed → no-op). |
| A5 | `game-engine` → `room-service` | Pub/Sub: `GameCompleted` on `gameplay.games` (key: `gameId`) | Room-service evaluates match scoreline. On match end → `MatchCompleted`, `RoomCompleted`. | At-least-once. Idempotent via `(matchId, gameId)` in `match_game_results`. Duplicate → skip. |
| A6 | `game-engine` → `ranking-service` | Pub/Sub: `GameCompleted` on `gameplay.games` | Elo pipeline: filter casual + non-abandoned → compute pairwise Elo → persist. | At-least-once. Idempotent via `(playerId, gameId)` in `processed_games`. Per-player atomic TX. Partial failure → redelivery completes remaining players. |
| A7 | `game-engine` → `spectator-projection-service` | Pub/Sub: all events on `gameplay.events` (key: `gameId`) | ACL transformation → spectator read model materialization → SSE push. | At-least-once. Idempotent via `eventId` dedup + projection version check. Consumer group scales with partition count. |
| A8 | `game-engine` → `audit-service` | Pub/Sub: all events on `gameplay.events` + `gameplay.games` | Universal audit trail. HMAC verification. Append-only ingestion. | At-least-once. Idempotent via `eventId` in `processed_events`. Append-only — duplicate insert rejected by constraint. |
| A17 | `game-engine` → `audit-service` | Pub/Sub: `GameCompleted` (full payload: `finalHands[]`, `shuffleSeed`, `deckOrderingAtGameStart`, signed envelope) on `gameplay.audit` (key: `gameId`) | Separates audit-privileged hand/seed content from `gameplay.games`. Enforces INV-SGP-01: the public `gameplay.games` topic carries no hand or seed data; `audit-service` is the only subscriber (ACL-restricted single consumer group). Producer path: outbox relay, same TX as the `gameplay.games` event. | At-least-once (outbox relay guarantees delivery). Idempotent via `eventId` in `processed_events` (same pipeline as A8). Single consumer group — no other context receives this topic. |
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
| SI3 | `identity-service` → Redis → `api-gateway` | **Adaptive throttle directive** — on detecting repeat-offender patterns (failed logins, anomalous session creation, suspension/ban), `identity-service` writes `rl:adaptive:{targetType}:{targetId}` to the shared Redis cluster. `api-gateway` reads this key on every authenticated request (piggy-backs on Layer 2 Redis call). Wires D-1 mitigation in `12-threat-model.md`. | Protects against credential stuffing, login brute-force, and banned-player reuse. | Fail-open on Redis unavailability (throttling skipped; Layer 1/2 limits still active). Directive TTL auto-expires. See `identity-session.md` §Throttle Directive Triggers for thresholds. |

---

## 4.7 Timer / Domain-Window Management

| # | From → To | Pattern | Rationale | Failure Semantics |
|---|-----------|---------|-----------|-------------------|
| T1 | `game-engine` → `timer_deadlines` table | **Durable timer scheduling** — in same TX as event store append: INSERT `{ deadlineId: windowId, gameId, type: "uno_challenge", expiresAt: now()+5s, version: 1, fired: false }`. | Timer must survive process crashes. In-memory `setTimeout` is lost on crash. Persisted deadline is recovered on restart. | Same TX as game event → if TX fails, neither event nor timer persists (atomic). |
| T2 | `game-engine` → `timer_deadlines` table | **Durable timer scheduling: reconnection** — INSERT `{ type: "reconnection", expiresAt: disconnectedAt+60s }` in same TX as `PlayerDisconnected` event. | 60 s reconnection window. Must survive game-engine or timer-service crashes. | Same atomicity as T1. Deadline row is source of truth. |
| T3 | `game-engine` → `timer_deadlines` table | **Durable timer scheduling: turn timer** — INSERT `{ type: "turn_timer", expiresAt: now()+30s }` (casual) or `+60s` (tournament) in same TX as `TurnAdvanced`. | Turn timeout forces auto-draw/auto-pass. Must be durable. | Same atomicity as T1. |
| T4 | `timer-service` → `game-engine` | **Poll + fire: expiry command** — `timer-service` polls `timer_deadlines WHERE expiresAt <= now() AND fired = false` (sharded by time bucket) at a **100 ms interval per instance** (canonical value, consistent with `08-capacity-sketch.md` §8.2.6 and `11-nfr-matrix.md` §11.1.4). Fires `ChallengeWindowExpired { gameId, windowId, version }` or `ReconnectionTimerExpired { gameId, playerId, reconnWindowId, version }` or `TurnTimerExpired { gameId, version }`. Note: this 100 ms timer poll is **distinct** from the 50 ms outbox relay poll. | Decoupled timer execution. `timer-service` only owns scheduling; Game aggregate owns validation. | **Idempotent:** `game-engine` checks `version` match and window state. If window already closed (challenge resolved, player reconnected, or version advanced) → discard. Duplicate fire is harmless. **Crash recovery:** `timer-service` rescans on restart. Late fire (by restart time) is acceptable — Game aggregate enforces timestamp-based window validity. |
| T5 | `game-engine` (on `Reconnect` command) → `timer_deadlines` | **Timer cancellation** — `PlayerReconnected` event + UPDATE `timer_deadlines SET fired = true` (or version increment). | Cancels 60 s reconnection timer when player reconnects within window. Subsequent `ReconnectionTimerExpired` from `timer-service` is discarded (version mismatch). | Atomic with `PlayerReconnected` event (same TX). If crash before commit → reconnection not recorded, timer fires normally → forfeit (safe, player can reconnect again). |

---

## 4.8 Summary Statistics

| Category | Row Count | Key Patterns |
|----------|-----------|--------------|
| Synchronous (REST/WS) | 12 | Request/response, WebSocket relay |
| Async Event Propagation | 17 | Kafka pub/sub with consumer groups |
| Transactional Outbox | 2 | Log-before-broadcast, session CAS |
| CQRS / Read Models | 4 | Event-carried state transfer, Redis cache |
| Sagas / Process Managers | 3 | Choreographed (round kickoff/advancement), orchestrated (match series) |
| Session Invalidation | 2 | Push-invalidation via Kafka |
| Timer / Window Management | 5 | Durable deadlines, poll-and-fire, idempotent expiry |
| **Total** | **45** | |

---

## 4.9 Traceability Note

Every integration row above traces to named commands and events in the [Commands & Events Catalog](../../domain/docs/04-commands-and-domain-events.md). The table below provides direct cross-links from Kafka topic → canonical event name → domain catalog section:

### Domain-Event → Topic Mapping

| Domain Event / Command | Kafka Topic | Architecture Integration Row | Domain Catalog Reference |
|------------------------|------------|------------------------------|--------------------------|
| `SessionEstablished` | `identity.sessions` | A1 (push-invalidation) | [Commands §4.2.1 Login](../../domain/docs/04-commands-and-domain-events.md#login-createsession) |
| `SessionInvalidated` | `identity.sessions` | A1, A2, SI1, SI2 | [Events §4.3 IS](../../domain/docs/04-commands-and-domain-events.md#sessioninvalidated) |
| `PlayerRegistered` | `identity.sessions` | A3 | [Events §4.3 IS](../../domain/docs/04-commands-and-domain-events.md#playerregistered) |
| `PlayerSuspended`, `PlayerBanned` | `identity.sessions` | A4 | [Events §4.3 IS](../../domain/docs/04-commands-and-domain-events.md#playersuspended) |
| `CardPlayed`, `CardDrawn`, `TurnAdvanced`, … | `gameplay.events` | A7, A8, C1 | [Events §4.3 RG](../../domain/docs/04-commands-and-domain-events.md#cardplayed) |
| `PlayerForfeited` | `gameplay.events` | A11 | [Events §4.3 RG](../../domain/docs/04-commands-and-domain-events.md#playerforfeited) |
| `GameCompleted` | `gameplay.games` | A5, A6, C4 | [Events §4.3 RG](../../domain/docs/04-commands-and-domain-events.md#gamecompleted) |
| `GameCompleted` (full payload: finalHands, shuffleSeed) | `gameplay.audit` | A17 | [Events §4.3 RG](../../domain/docs/04-commands-and-domain-events.md#gamecompleted) — audit-privileged subset, ACL-restricted |
| `MatchGameCompleted`, `MatchCompleted` | `gameplay.rooms` | A9, A10 | [Events §4.3 RG](../../domain/docs/04-commands-and-domain-events.md#matchcompleted) |
| `RoomCreated`, `RoomCompleted` | `gameplay.rooms` | A12, A15 | [Events §4.3 RG](../../domain/docs/04-commands-and-domain-events.md#roomcompleted) |
| `TournamentRoomAssigned` | `tournament.rooms` | A12, P1 | [Events §4.3 TO](../../domain/docs/04-commands-and-domain-events.md#tournamentroomassigned) |
| `PlayerAdvanced`, `PlayerEliminated` | `tournament.lifecycle` | A13 | [Events §4.3 TO](../../domain/docs/04-commands-and-domain-events.md#playeradvanced) |
| `EloUpdated`, `PlayerStatisticsUpdated` | `ranking.updates` | A14, A16 | [Events §4.3 RK](../../domain/docs/04-commands-and-domain-events.md#eloupdated) |
| `UnoChallengeWindow` (timer) | DB `timer_deadlines` | T1, T4 | [Value Objects §3.1.5](../../domain/docs/03-aggregates-entities-value-objects.md#unochallengewindow) |
| `ReconnectionWindow` (timer) | DB `timer_deadlines` | T2, T5 | [Value Objects §3.1.5](../../domain/docs/03-aggregates-entities-value-objects.md#reconnectionwindow) |

### Synchronous Command Traceability

| REST Endpoint / Pattern | Architecture Row | Domain Command | Domain Catalog Reference |
|------------------------|-----------------|----------------|--------------------------|
| `POST /sessions` | S1 | `Login` | [Commands §4.2.1](../../domain/docs/04-commands-and-domain-events.md#login-createsession) |
| `POST /players` | S2 | `RegisterPlayer` | [Commands §4.2.1](../../domain/docs/04-commands-and-domain-events.md#registerplayer) |
| `POST /rooms`, `POST /rooms/{id}/join` | S3 | `CreateRoom`, `JoinRoom` | [Commands §4.2.2](../../domain/docs/04-commands-and-domain-events.md#createroom) |
| WebSocket relay: `play_card`, `draw_card`, `call_uno` | S4 | `PlayCard`, `DrawCard`, `CallUno` | [Commands §4.2.2](../../domain/docs/04-commands-and-domain-events.md#playcard) |
| `POST /tournaments`, `POST /tournaments/{id}/start` | S7 | `CreateTournament`, `StartTournament` | [Commands §4.2.3](../../domain/docs/04-commands-and-domain-events.md#createtournament) |
| `InitializeGame` (internal RPC) | S10, P3 | Internal (no domain command); triggers `GameStarted` domain event | [Events §4.3 RG](../../domain/docs/04-commands-and-domain-events.md#gamestarted) |

- Deltas from the Design Checkpoint are documented in [CHANGELOG-design.md](10-CHANGELOG-design.md).
