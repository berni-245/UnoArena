# ADR-002: Kafka as Event Backbone

**Status:** Accepted
**Date:** 2026-05-17

## Context

UnoArena has 6 bounded contexts that communicate primarily via domain events. Peak event rate is ~25,000 events/sec on the `gameplay.events` topic during the first tournament round (100k concurrent games). Events must be ordered per aggregate (per `gameId`), and multiple independent consumer groups must process the same events (spectator projection, ranking, audit, tournament).

## Decision

Use **Apache Kafka** as the central event backbone. One topic per concern (e.g., `gameplay.events`, `gameplay.games`, `tournament.lifecycle`), partitioned by aggregate ID (`gameId`, `roomId`, `tournamentId`, `playerId`) to guarantee per-aggregate ordering.

## Rationale

- **Partition ordering:** Kafka guarantees total ordering within a partition. Partitioning by aggregate ID ensures all events for a game arrive in sequence.
- **Consumer group independence:** Multiple consumer groups read the same topic independently. `spectator-projection-service`, `ranking-service`, and `audit-service` scale independently without affecting each other.
- **Log retention:** Kafka retains events for a configurable period, enabling consumer replay (e.g., rebuilding a spectator projection after a crash).
- **Throughput:** Kafka handles 25 MB/s write + 100 MB/s read (4 consumer groups) on a standard 3–5 broker cluster.

## Alternatives Considered

1. **RabbitMQ:** Simpler to set up but lacks native partition ordering at scale. Fan-out requires exchange configuration per consumer. No built-in log retention for replay.
2. **Redis Streams:** Good for moderate scale but consumer group rebalancing is less mature than Kafka's. Lacks the operational tooling ecosystem.
3. **NATS JetStream:** Strong contender with good ordering and persistence. Less ecosystem tooling, smaller community for operational support at this scale.

## Consequences

- **Positive:** Proven at scale. Log retention enables replay. Partition ordering by aggregate. Multiple consumer groups.
- **Negative:** Operational complexity (ZooKeeper/KRaft, broker management). Higher latency (~5–20 ms) than in-memory brokers. Requires careful partition count planning.
