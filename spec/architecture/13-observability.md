# 13. Observability Architecture

> Logging, metrics, and tracing strategy for UnoArena. Covers what each service emits, how `correlationId`/`causationId` propagate across async boundaries, and the dashboards and alerts that keep tournament operations healthy.

---

## 13.1 Principles

1. **Correlation-first:** Every request, command, and event carries a `correlationId` that ties together all spans, log lines, and events belonging to the same business operation across all services and async hops.
2. **Domain signals as alerts:** The domain model already defines observable failure signals (doc 07 §7.7). These map directly to alert rules — no additional instrumentation needed for domain-level anomalies.
3. **Structured logs everywhere:** All log lines are JSON with a standard schema. Free-text messages are for human context only; machine-readable fields drive dashboards and alerts.
4. **Separate observability plane:** Logs, metrics, and traces are written to a dedicated observability sink (e.g., OpenTelemetry Collector → backend). Observability path does not share the Kafka event bus used for domain events.

---

## 13.2 Correlation ID Propagation

The `correlationId` (UUID) originates at the client's first request and flows through every layer.

### 13.2.1 Synchronous path (HTTP / WebSocket)

```
Client ──── HTTP/WebSocket ────► api-gateway
              X-Correlation-Id: <uuid>        (injected by gateway if absent)
                     │
              passes header downstream
                     │
              ► identity-service / room-service / game-engine / etc.
                     │
              structured log entry:
                { correlationId, spanId, serviceId, ... }
```

The `api-gateway` generates a `correlationId` UUID for every new client request that does not already carry one. For WebSocket sessions, the `correlationId` is set at connection establishment and reused for all messages on that connection.

### 13.2.2 Async path (Kafka)

When a service publishes a domain event to Kafka, it carries the `correlationId` and `causationId` in:
1. **The event envelope** (doc 04 §4.1.2) — `correlationId` and `causationId` fields.
2. **Kafka message headers** — same values as W3C `traceparent`/`tracestate` headers (for OpenTelemetry propagation across the broker boundary).

```
game-engine (produces CardPlayed to gameplay.events)
  message headers:
    X-Correlation-Id: <original-play-card-correlationId>
    X-Causation-Id:   <commandId of PlayCard>
    traceparent:      00-<traceId>-<spanId>-01

spectator-projection-service (consumes CardPlayed)
  → creates child span with parent = traceparent from header
  → log line: { correlationId, causationId, traceId, spanId, service: "spectator-projection" }

audit-service (consumes CardPlayed)
  → appends audit entry with correlationId, causationId from event envelope
  → enables: "show me all events from this correlationId across all contexts"
```

### 13.2.3 Cross-context saga correlation

For long-running sagas (Tournament Round Progression, Disconnection → Forfeit chain), the `correlationId` traces the entire chain:

```
TournamentStarted    correlationId: T-saga-<tournamentId>
  └─ TournamentRoundCreated            causationId: TournamentStarted.eventId
       └─ TournamentRoomAssigned × N   causationId: TournamentRoundCreated.eventId
            └─ GameStarted             causationId: TournamentRoomAssigned.eventId
                 └─ GameCompleted      causationId: last CardPlayed.eventId
                      └─ PlayerAdvanced / PlayerEliminated
```

A single query on `correlationId = T-saga-<tournamentId>` in the audit trail reconstructs the full saga timeline across all services.

---

## 13.3 Structured Log Schema (Standard Fields)

Every log line from every service includes:

```json
{
  "ts":            "2026-05-17T14:23:01.042Z",
  "level":         "INFO | WARN | ERROR",
  "service":       "game-engine",
  "instanceId":    "game-engine-pod-7f4b",
  "traceId":       "4bf92f3577b34da6a3ce929d0e0e4736",
  "spanId":        "00f067aa0ba902b7",
  "correlationId": "a1b2c3d4-...",
  "causationId":   "e5f6a7b8-...",
  "aggregateType": "Game",
  "aggregateId":   "<gameId>",
  "event":         "CardPlayed",
  "msg":           "Command processed successfully",
  ...service-specific fields...
}
```

Service-specific fields are documented per service below. The standard fields above are always present and indexed for query.

---

## 13.4 Per-Service Observability

### 13.4.1 `api-gateway`

**Metrics (RED pattern):**

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `gateway_connections_active` | Gauge | `type={websocket,sse}` | Current live connections |
| `gateway_connections_total` | Counter | `type, result={accepted,rejected}` | Connection attempts |
| `gateway_auth_failures_total` | Counter | `reason={invalid_token,expired,invalidated,missing}` | Auth rejection breakdown |
| `gateway_rate_limited_total` | Counter | `layer={ip,player,room}` | Rate limit hits per layer |
| `gateway_message_latency_ms` | Histogram | `direction={inbound,outbound}`, `type` | WebSocket message processing latency |
| `gateway_session_invalidations_total` | Counter | — | Push-invalidation events processed |

**Key log events:**
- Connection established: `{ event: "ws_connected", playerId, sessionId }`
- Connection closed: `{ event: "ws_closed", playerId, sessionId, reason, durationMs }`
- Auth failure: `{ event: "auth_failed", reason, ip }`
- Session push-invalidated: `{ event: "session_invalidated_push", sessionId, playerId, lagMs }`

---

### 13.4.2 `identity-service`

**Metrics:**

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `identity_logins_total` | Counter | `result={success,invalid_creds,suspended,banned}` | Login outcomes |
| `identity_session_created_total` | Counter | — | New sessions issued |
| `identity_session_invalidated_total` | Counter | `reason={new_login,logout,admin}` | Invalidations by trigger |
| `identity_token_validation_latency_ms` | Histogram | `path={redis_hit,redis_miss_db}` | Validation path distribution |
| `identity_registration_total` | Counter | `result={success,name_taken,email_exists}` | Registration outcomes |

**Key log events:**
- Login: `{ event: "login", playerId, result, ip }`
- Session invalidated: `{ event: "session_invalidated", sessionId, playerId, reason }`
- Registration: `{ event: "player_registered", playerId, displayName }`

---

### 13.4.3 `room-service`

**Metrics:**

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `room_created_total` | Counter | `type={casual,tournament}` | Rooms created |
| `room_state_transitions_total` | Counter | `from, to` | Room lifecycle transitions |
| `room_match_completed_total` | Counter | `result={normal,abandoned,timeout_forced}` | Match outcomes |
| `room_players_current` | Gauge | — | Total players in active rooms |
| `room_command_latency_ms` | Histogram | `command={JoinRoom,LeaveRoom,StartMatch}` | Room command processing time |

**Key log events:**
- Room created: `{ event: "room_created", roomId, roomType, hostPlayerId }`
- Match started: `{ event: "match_started", roomId, matchId, playerCount }`
- Room completed: `{ event: "room_completed", roomId, result, durationMs }`

---

### 13.4.4 `game-engine`

The `game-engine` is the highest-cardinality service. Metrics focus on command throughput, rejection rates, and the health signals defined in doc 07 §7.7.

**Metrics:**

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `game_commands_total` | Counter | `command, result={accepted,rejected}` | All command outcomes |
| `game_rejections_total` | Counter | `reason={stale_sequence,not_your_turn,illegal_play,window_closed,rate_limited}` | Rejection breakdown — domain health signals |
| `game_command_latency_ms` | Histogram | `command` | End-to-end command processing (receive → DB commit → ACK) |
| `game_event_store_write_latency_ms` | Histogram | — | PostgreSQL TX commit latency |
| `game_outbox_lag_ms` | Gauge | — | Age of oldest undelivered outbox row |
| `game_aggregate_cache_misses_total` | Counter | — | Game aggregate rebuilt from event store (startup/crash) |
| `game_timer_deadlines_fired_total` | Counter | `type={uno_challenge,wdf_challenge,reconnection,turn_timer}` | Timer firings by type |
| `game_active_games_current` | Gauge | — | Games currently `InProgress` on this instance |
| `game_completed_total` | Counter | `result={normal,abandoned,timeout_forced}` | Game outcomes |

**Domain alert signals (from doc 07 §7.7) → metrics mapping:**

| Domain Signal | Metric | Alert Threshold |
|---------------|--------|-----------------|
| `CommandRejected(StaleSequenceNumber)` spike | `game_rejections_total{reason="stale_sequence"}` | > 5% of commands in 5-min window |
| `CommandRejected(NotCurrentPlayer)` spike | `game_rejections_total{reason="not_your_turn"}` | > 2% of commands in 5-min window |
| `ReconnectionTimerExpired` spike | `game_timer_deadlines_fired_total{type="reconnection"}` rate > baseline × 3 | Investigate DDoS or network instability |

**Key log events:**
- Command accepted: `{ event: "command_accepted", command, gameId, playerId, sequenceNumber, latencyMs }`
- Command rejected: `{ event: "command_rejected", command, reason, gameId, playerId, sequenceNumber }`
- Game completed: `{ event: "game_completed", gameId, result, wasAbandoned, roomType, durationMs }`
- Aggregate rebuilt from event store: `{ event: "aggregate_rebuilt", gameId, eventCount, rebuildMs }` (WARN)
- Timer fired: `{ event: "timer_fired", type, gameId, playerId, delayMs }` (actual vs. scheduled)

---

### 13.4.5 `timer-service`

**Metrics:**

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `timer_deadlines_pending_current` | Gauge | `type` | Open deadline rows by type |
| `timer_deadlines_fired_total` | Counter | `type, result={valid,already_closed,idempotent_discard}` | Fire outcomes |
| `timer_poll_latency_ms` | Histogram | — | Time between scan and fire |
| `timer_overdue_deadlines_current` | Gauge | `type` | Deadlines past `expiresAt` not yet fired (lag indicator) |

**Alert:** `timer_overdue_deadlines_current{type="reconnection"} > 100` for 2+ minutes → timer-service may be lagging; investigate polling health.

---

### 13.4.6 `tournament-service`

**Metrics:**

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `tournament_rounds_active_current` | Gauge | — | Rounds currently awaiting completion |
| `tournament_rooms_completed_total` | Counter | `tournamentId` | Rooms that have reported results |
| `tournament_rooms_pending_current` | Gauge | `tournamentId` | Rooms still outstanding per round |
| `tournament_advancement_latency_ms` | Histogram | — | Time from `AllMatchesInRoundCompleted` to `TournamentRoundCreated` (next round) |
| `tournament_timeout_warnings_total` | Counter | — | `RoundTimeoutWarning` events emitted |
| `tournament_forced_resolutions_total` | Counter | — | Rooms force-resolved; always alert |

**Domain alert signals:**

| Domain Signal | Metric | Alert |
|---------------|--------|-------|
| `RoundTimeoutWarning` | `tournament_timeout_warnings_total` rate | Alert immediately on any occurrence; page on-call |
| `ForceResolveTimedOutRoom` | `tournament_forced_resolutions_total` rate | Alert; post-tournament review |

---

### 13.4.7 `round-kickoff-worker`

**Metrics:**

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `kickoff_rooms_published_total` | Counter | `shardId` | `TournamentRoomAssigned` events published |
| `kickoff_rooms_per_second` | Gauge | `shardId` | Current publish rate |
| `kickoff_consumer_lag_current` | Gauge | `topic=tournament.kickoff-work` | Unprocessed work items in queue |
| `kickoff_dlq_depth_current` | Gauge | — | Events in dead-letter queue (permanent failures) |
| `kickoff_backpressure_pauses_total` | Counter | — | Times publish rate was throttled |

**Alert:** `kickoff_dlq_depth_current > 0` → room creation failures requiring operator review.

---

### 13.4.8 `ranking-service`

**Metrics:**

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `ranking_elo_updates_total` | Counter | `result={updated,skipped_tournament,skipped_abandoned,idempotent_discard}` | Pipeline outcomes |
| `ranking_pipeline_latency_ms` | Histogram | — | `GameCompleted` receive → Elo persisted |
| `ranking_consumer_lag_ms` | Gauge | `topic=gameplay.games` | Kafka consumer lag |
| `ranking_leaderboard_update_latency_ms` | Histogram | — | DB write → Redis ZADD |
| `ranking_rating_floor_applied_total` | Counter | — | Players hitting Elo = 100 (domain signal) |
| `ranking_abandoned_games_skipped_total` | Counter | — | `AbandonedGameSkipped` domain signal |

**Alert:** `ranking_consumer_lag_ms > 30000` (30 s) during non-surge periods → pipeline falling behind; scale consumers.

---

### 13.4.9 `spectator-projection-service`

**Metrics:**

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `spectator_sse_subscribers_current` | Gauge | — | Active SSE connections |
| `spectator_projection_update_latency_ms` | Histogram | — | `CardPlayed` received → SSE push to subscribers |
| `spectator_acl_transforms_total` | Counter | `event, stripped_fields` | ACL transformation counts (audit of privacy enforcement) |
| `spectator_consumer_lag_ms` | Gauge | `topic=gameplay.events` | Kafka consumer lag |
| `spectator_projection_rebuild_total` | Counter | — | Projections rebuilt from Kafka replay (instance restart) |

**Alert:** `spectator_projection_update_latency_ms p95 > 2000` → spectator projection degraded; investigate consumer lag or SSE fan-out.

---

### 13.4.10 `audit-service`

**Metrics:**

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `audit_events_ingested_total` | Counter | `sourceContext, topic` | All ingested domain events |
| `audit_duplicate_events_discarded_total` | Counter | `sourceContext` | Idempotency deduplication (domain signal) |
| `audit_signature_failures_total` | Counter | `sourceContext` | HMAC verification failures — always alert |
| `audit_malformed_events_total` | Counter | `sourceContext` | Envelope validation failures (domain signal) |
| `audit_ingestion_lag_ms` | Gauge | `topic` | Consumer lag per topic |
| `audit_query_total` | Counter | `endpoint, role` | API query counts by role |
| `audit_compliance_exports_total` | Counter | — | Compliance export operations (meta-audit signal) |

**Alerts:**
- `audit_signature_failures_total > 0` → potential tampering; page security team immediately.
- `audit_malformed_events_total` rate spike → schema contract violation between contexts; investigate recent deployments.

---

## 13.5 Distributed Tracing

**Technology:** OpenTelemetry SDK in each service; W3C `traceparent` header for HTTP and Kafka propagation.

**Trace structure for PlayCard (hot path):**

```
Trace: <traceId>
│
├── Span: api-gateway "ws_message_received" (2 ms)
│     tags: playerId, gameId, command=PlayCard
│
├── Span: game-engine "handle_play_card" (25 ms)
│     tags: gameId, sequenceNumber, result=accepted
│     ├── Span: "idempotency_check" (< 1 ms)
│     ├── Span: "invariant_validation" (< 1 ms)
│     └── Span: "db_commit_event_outbox" (15 ms)
│           tags: eventType=CardPlayed, outboxRowId
│
└── Span: api-gateway "ws_ack_sent" (< 1 ms)

[async, separate trace linked by correlationId]
├── Span: outbox-relay "publish_to_kafka" (10 ms)
└── Span: spectator-projection-service "consume_card_played" (50 ms)
      ├── Span: "acl_transform" (< 1 ms)
      └── Span: "projection_write" (10 ms)
```

**Sampling strategy:**
- 100% sampling for `CommandRejected` traces (debugging) and all `ERROR`-level spans.
- 10% head-based sampling for normal gameplay commands during peak load (100k cmd/s → 10k sampled/s, manageable).
- 100% sampling for all tournament lifecycle events (`TournamentRoundCreated`, `PlayerAdvanced`, `PlayerEliminated`, `TournamentCompleted`).

---

## 13.6 Tournament Round Health Dashboard

The single most important operational dashboard during a live tournament. Updated every 30 seconds.

### Panel 1 — Round Progress

```
Tournament: <name>   Round: <N>   Status: IN PROGRESS
────────────────────────────────────────────────────
Rooms completed:   ████████████░░░░░░░░░  62,341 / 100,000  (62.3%)
Players advanced:  ████████░░░░░░░░░░░░░  186,123 advancing
Time in round:     47 min 12 sec
Estimated completion: ~32 min remaining  (extrapolated from completion rate)
```

**Source:** `tournament_rooms_completed_total` counter, `tournament_rooms_pending_current` gauge.

### Panel 2 — Round Timeout Risk

| Room ID | Time Since Created | Status |
|---------|--------------------|--------|
| (none) | — | — |

Alert fires at 90 min (75% of 2h timeout). Page at 100 min.

### Panel 3 — Kickoff Surge Metrics (visible during round start only)

```
Rooms published:  ████████████████████  100,000 / 100,000  (100%)
Publish rate:     8,750 rooms/sec (10 shards active)
Time elapsed:     11.4 seconds
DLQ depth:        0
```

### Panel 4 — Game Completion Rate

```
GameCompleted events/sec:  ████  ~1,200/sec  (rising as games finish)
GameAbandoned rate:        0.2%  (within normal)
Force-resolved rooms:      0
```

### Panel 5 — Consumer Lag Heatmap

Shows Kafka consumer lag across all services consuming `gameplay.rooms` and `gameplay.games`. A lag spike here predicts delayed advancement evaluation.

---

## 13.7 Alert Runbook Summary

  | Alert | Condition | Severity | Action |
  |-------|-----------|----------|--------|
  | **Signature failure** | `audit_signature_failures_total > 0` | P0 | Suspend affected room; notify security; preserve game log |
  | **Round timeout warning** | `tournament_timeout_warnings_total` increments | P1 | Identify stuck room(s); verify turn timer is functioning; prepare force-resolve |
  | **Forced resolution** | `tournament_forced_resolutions_total` increments | P2 | Post-tournament review; investigate turn timer or room-service bug |
  | **Stale sequence spike** | `game_rejections_total{reason="stale_sequence"}` > 5% in 5 min | P2 | Investigate SSE delivery lag to clients; check `game_outbox_lag_ms` |
| **Reconnection timer spike** | `game_timer_deadlines_fired_total{type="reconnection"}` > baseline × 3 | P2 | Investigate network stability; check for DDoS pattern |
| **Audit consumer lag** | `audit_ingestion_lag_ms > 120000` (2 min) | P3 | Scale `audit-service` consumer group; not gameplay-blocking |
| **Malformed event** | `audit_malformed_events_total` rate spike | P2 | Correlate with recent deployments; check schema contract between contexts |
| **DLQ depth > 0** | `kickoff_dlq_depth_current > 0` | P1 | Investigate failed room creation; may need operator intervention to assign substitute rooms |
| **Timer overdue backlog** | `timer_overdue_deadlines_current > 100` for 2+ min | P1 | `timer-service` polling degraded; reconnection windows may expire late |
