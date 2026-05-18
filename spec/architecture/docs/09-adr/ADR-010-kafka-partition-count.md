# ADR-010: Kafka Partition Count Sizing

**Status:** Accepted

---

## Context

The `gameplay.events` topic handles a peak rate of ~300k events/s with 4 consumer groups (spectator-projection, ranking, tournament, audit). To sustain this throughput with adequate consumer parallelism, the partition count must be sized to allow enough concurrent consumers per group while keeping rebalance time manageable. Topic partition counts are set at creation time and are difficult to change without operational disruption (consumer rebalancing, key-based ordering guarantees affected).

---

## Decision

| Topic | Partition Count |
|-------|----------------|
| `gameplay.events` | 256 |
| `gameplay.games` | 64 |
| `gameplay.rooms` | 64 |
| `tournament.lifecycle` | 32 |
| `tournament.rooms` | 32 |
| `tournament.kickoff-work` | 32 |
| `identity.sessions` | 32 |
| `ranking.updates` | 32 |
| `gameplay.audit` | 32 |

---

## Rationale

- **256 partitions for `gameplay.events`:** At 300k events/s and 4 consumer groups, each group needs to sustain ~300k reads/s. With 256 partitions, each partition carries ~1,170 events/s, allowing up to 256 consumer instances per group. This provides ~2x headroom above current peak (~500k events/s sustainable).
- **Rebalance time:** At 256 partitions, Kafka consumer group rebalance completes in ~10 seconds (acceptable given the at-least-once delivery model and idempotent consumers).
- **64 for `gameplay.games` and `gameplay.rooms`:** These topics have much lower throughput (~83 events/s and ~500 events/s respectively). 64 partitions provide ample parallelism without excessive metadata overhead.
- **32 for remaining topics:** Low-throughput topics (burst or steady-state < 10k events/s). 32 partitions support scaling to 32 consumer instances per group, which exceeds the planned instance counts.

---

## Alternatives Considered

| Alternative | Reason Rejected |
|-------------|-----------------|
| 128 partitions for `gameplay.events` | Insufficient for 4x read fan-out at peak. Each consumer group would need to process ~2,340 events/s/partition, limiting max consumer instances to 128 (below the 200 game-engine instances that produce). |
| 512 partitions for `gameplay.events` | Excessive rebalance time at current scale (~20s). Increases Kafka controller metadata load and ZooKeeper/KRaft session overhead. Not justified until throughput exceeds ~600k events/s. |

---

## Consequences

- Partition count is set at topic creation and is operationally hard to change (requires consumer coordination, potential ordering disruption for keyed topics). The chosen counts are sized for 2x headroom above current peak load.
- Consumer group parallelism is bounded by partition count: no group can have more active consumers than partitions.
- If peak throughput doubles beyond ~600k events/s, `gameplay.events` must be repartitioned to 512 (planned as a multi-region migration step per capacity sketch 8.7).
