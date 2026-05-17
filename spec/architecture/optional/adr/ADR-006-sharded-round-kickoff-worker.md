# ADR-006: Sharded Round-Kickoff Worker for Tournament Surge

**Status:** Accepted
**Date:** 2026-05-17

## Context

A 1,000,000-player tournament creates ~100,000 rooms in the first round. These rooms must all start within seconds — a coordinated spike, not gradual load. If `tournament-service` publishes 100k `TournamentRoomAssigned` events directly to Kafka in a tight loop, it creates a thundering herd: `room-service` consumers are overwhelmed by 100k simultaneous room creation requests, database connection pools saturate, and the broker experiences producer buffer pressure.

## Decision

Introduce a dedicated **`round-kickoff-worker`** pool, sharded by player-range partition (10–20 shards). `tournament-service` enqueues work items to an internal Kafka topic (`tournament.kickoff-work`). Each worker shard publishes `TournamentRoomAssigned` events to `tournament.rooms` at a **rate-limited** pace (1,000 rooms/sec/shard). With 10 shards: 10,000 rooms/sec total → all 100k rooms published in ~10 seconds.

## Rationale

- **Controlled fan-out:** Rate limiting prevents thundering herd. `room-service` consumer group absorbs 10k rooms/sec across its instances (not 100k in one burst).
- **Backpressure isolation:** `tournament-service` remains responsive for status queries during kickoff. Kickoff workers are separate processes that can be throttled independently.
- **Independent scaling:** Add shards for larger tournaments. Reduce shards for smaller ones. No impact on tournament-service sizing.
- **Idempotent room creation:** Each `TournamentRoomAssigned` carries a pre-generated `roomId`. Duplicate events (from at-least-once delivery or worker restart) create no duplicate rooms.

## Alternatives Considered

1. **Direct publish from `tournament-service`:** Tournament-service becomes a bottleneck. A single producer loop publishing 100k messages blocks all other tournament operations. No rate control.
2. **Bulk Kafka produce without rate limiting:** All 100k messages hit Kafka and `room-service` simultaneously. Room-service consumer lag spikes, DB connections saturate, room creation latency degrades for all rooms.
3. **Synchronous batch API to `room-service`:** REST batch endpoint to create rooms in bulk. Synchronous — `tournament-service` blocks waiting for responses. Doesn't scale. Partial failure handling is complex.

## Consequences

- **Positive:** Thundering herd eliminated. Controlled 10-second rollout. Backpressure-aware (monitors consumer lag). Independently scalable.
- **Negative:** Additional component to deploy and monitor (`round-kickoff-worker`). 10-second startup delay (acceptable — players wait in lobby during this period). Internal topic (`tournament.kickoff-work`) adds operational surface.
