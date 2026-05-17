# ADR-001: Event Sourcing + Transactional Outbox for Room Gameplay

**Status:** Accepted
**Date:** 2026-05-17

## Context

The `game-engine` service must satisfy the **log-before-broadcast** invariant: every authoritative state change is durably persisted in the immutable game log before any broadcast to players, spectators, or downstream consumers. This requires atomicity between the event store write (PostgreSQL) and the event publication (Kafka). A dual-write to two different systems cannot be made atomic without additional patterns.

## Decision

Use **event sourcing** for the Game aggregate: all state changes are persisted as an append-only sequence of domain events in a PostgreSQL `game_events` table. A **transactional outbox** (`outbox` table) is written in the **same PostgreSQL transaction** as the event store append. An outbox relay worker polls the outbox every 50 ms and publishes events to Kafka.

## Rationale

- **Atomicity:** Single TX guarantees both the event and the outbox row are persisted together, or neither is.
- **Crash safety:** If the process crashes after COMMIT but before Kafka publish, the outbox relay picks up the event on restart. No event is lost.
- **Rebuildable state:** The Game aggregate can be reconstructed by replaying events from the event store. No separate state table needed (though an in-memory cache is used for performance).
- **Audit trail:** The event store doubles as an immutable game log for dispute resolution.

## Alternatives Considered

1. **Direct Kafka produce (no outbox):** Cannot guarantee atomicity with DB write. A crash between DB commit and Kafka produce loses the event. Violates log-before-broadcast.
2. **CDC (Change Data Capture) via Debezium:** Atomicity is achieved by tailing the PostgreSQL WAL. However: adds significant infrastructure complexity (Debezium + connector management), harder to operate, and WAL-based CDC can lag under load.
3. **Kafka as event store (no PostgreSQL):** Eliminates the dual-write problem but loses PostgreSQL's query capabilities for game state reconstruction, idempotency checks, and the timer deadline table. Also requires careful partition management for per-game ordering.

## Consequences

- **Positive:** Atomic, crash-safe writes. Game state is fully rebuildable. Event store serves as immutable audit log. Simple operational model (PostgreSQL + polling relay).
- **Negative:** 50 ms polling interval adds latency to event publication (P99 ~50 ms). Outbox table needs periodic cleanup (delivered rows pruned after 24 h). Outbox relay is an additional component to monitor.
- **Trade-off accepted:** 50 ms publication latency is acceptable for a game where human reaction time is measured in seconds.
