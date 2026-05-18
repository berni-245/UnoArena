# 8. Capacity Sketch

> Order-of-magnitude reasoning for UnoArena under peak tournament load (1M players, 100k+ simultaneous matches). Corresponds to §6.5 of the Architecture Checkpoint.

---

## 8.1 Baseline Assumptions

| Parameter | Value | Source |
|-----------|-------|--------|
| Max tournament players | 1,000,000 | presentation/high-level-definition.md |
| Players per room | 10 | Design Checkpoint §2.1.3 |
| First-round rooms | ~100,000 | 1M / 10 |
| Spectator-to-player ratio (average) | 5:1 | Conservative estimate; popular rooms higher |
| Spectator-to-player ratio (finals) | up to 100:1 | High-profile final matches; served via `regional-edge-proxy` fan-out (declared deployable, see §2.3 and §8.5) |
| Hard spectator cap per room | None enforced at the domain level; practical cap = 10 regions × 10k spectators per edge per game = **~100k spectators per room** | Regional edge proxy architecture is the enforcement mechanism |
| Average game duration | ~10 minutes | Typical Uno game length |
| Games per match (best-of-3) | 2.5 average | Early termination at 2 wins (2-player) or always 3 (multi-player) |
| Match duration | ~25 minutes | ~10 min × 2.5 games |
| Commands per player per game | ~60 | PlayCard, DrawCard, PassTurn, CallUno, challenges |
| Events per game (all types) | ~150 | State-change events including turns, draws, plays, challenges |
| Event payload size | ~1 KB average | JSON with game state delta |

---

## 8.2 Peak Load Calculations

### 8.2.1 Concurrent Connections

| Metric | Value | Derivation |
|--------|-------|------------|
| Player WebSocket connections | 1,000,000 | All tournament players connected |
| Spectator SSE connections | 5,000,000 | 5:1 average across 100k rooms |
| Total long-lived connections | **~6,000,000** | Players + spectators |
| Gateway instances needed | **~600** | 10,000 connections per instance |

**Note:** Not all 1M players are in active games simultaneously. Round 1 has 100k rooms, each with 10 players. Between rounds, some players disconnect. Peak WebSocket connections occur during the first round.

### 8.2.2 Command Rate (Player → game-engine)

| Metric | Value | Derivation |
|--------|-------|------------|
| Commands per game | ~600 | 10 players × 60 commands/player |
| Commands per second per game | ~1.0 | 600 commands / 600 s |
| **Aggregate command rate** | **~100,000 cmd/s** | 100k games × 1.0 cmd/s |
| game-engine instances | 200 | Horizontal scaling |
| Commands per instance | **~500 cmd/s** | 100k / 200 |

Each `game-engine` instance handles ~500 commands/sec. PostgreSQL write per command: 2 inserts (event store + outbox) = ~1,000 writes/sec per instance. Aggregate across all 200 instances: ~100k writes/s flowing to the single RG PostgreSQL primary. This is at the high end of what cloud PostgreSQL can sustain; the single-primary design is a known bottleneck at this scale. See §8.6 for the concrete mitigation path (horizontal write sharding by `gameId` range).

### 8.2.3 Event Production Rate (game-engine → Kafka)

| Metric | Value | Derivation |
|--------|-------|------------|
| Events per game per second | ~0.25 | 150 events / 600 s |
| **gameplay.events throughput** | **~25,000 events/s** | 100k games × 0.25 events/s |
| Payload throughput | **~25 MB/s** | 25k events × 1 KB |
| Outbox relay latency | 50 ms (polling interval) | Adds ≤ 50 ms to event delivery |

### 8.2.4 Kafka Consumer Read Multiplier

Each consumer group reads the full topic independently:

| Topic | Producers | Consumer Groups | Write Rate | Total Read Rate |
|-------|-----------|----------------|------------|-----------------|
| `gameplay.events` | game-engine | spectator-projection, ranking, tournament, audit | 25k events/s | 100k events/s (4× fan-out) |
| `gameplay.games` | game-engine | room-service, ranking, spectator-projection, audit | ~83 events/s (steady) | ~332 events/s |
| `gameplay.rooms` | room-service | tournament, spectator-projection, audit | ~500 events/s | ~1.5k events/s |
| `tournament.lifecycle` | tournament-service | spectator-projection, ranking, audit | ~100 events/s (burst) | ~300 events/s |
| `tournament.rooms` | round-kickoff-worker | room-service | 10k events/s (burst, 10 s) | 10k events/s |

**Peak Kafka throughput:** ~25 MB/s write + ~100 MB/s read. Standard Kafka cluster (3–5 brokers) handles this comfortably.

### 8.2.5 Database Load Per Context

| Context | Operation | Rate | Notes |
|---------|-----------|------|-------|
| **RG — Event store** | INSERT game_events + outbox | ~100k writes/s (50k events + 50k outbox) | Distributed across 200 game-engine instances. ~500 writes/s per instance per DB connection. Partitioned by gameId. |
| **RG — Room state** | UPDATE rooms, matches | ~500/s | Low frequency. State transitions are infrequent per room. |
| **RG — Timer deadlines** | INSERT + UPDATE (poll+fire) | ~10k/s | Turn timers: 100k active. ~1 fire/s per game average. |
| **TO — Completion counter** | Atomic UPDATE + INSERT | ~83/s (steady), burst ~5k/s (round end) | SERIALIZABLE per round. Idempotent via unique constraint. |
| **RK — Elo updates** | UPDATE + INSERT (per player) | Burst: ~8k/s at round end (100k games × 10 players / ~120 s) | Per-player atomic TX. Consumer group scales with partitions. |
| **SV — Projection updates** | UPSERT spectator projections | ~25k/s (matches gameplay.events rate) | Document updates. Can batch. |
| **AL — Audit ingestion** | INSERT (append-only) | ~30k/s (all events) | Append-only. Batch inserts. ClickHouse native batch support. |
| **IS — Sessions** | Mostly reads (token validation cache in Redis) | ~1M reads/s (Redis), ~100 writes/s (logins) | Redis handles read volume. PostgreSQL only on cache miss. |

### 8.2.6 Timer Service Load

| Metric | Value | Derivation |
|--------|-------|------------|
| Active turn timers | ~100,000 | One per active game |
| Active reconnection timers | ~1,000 (worst case) | ~1% of players disconnected at any time |
| Active challenge windows | ~500 | ~0.5% of games in challenge phase |
| **Total active timers** | **~101,500** | Sum |
| Timer-service instances | 10 | Sharded by deadline time bucket |
| Timers per instance | ~10,000 | Evenly distributed |
| Poll frequency | Every 100 ms per instance | |
| Timer fire rate (steady state) | ~1,700/s | Turn timers: ~1/game/10s = ~10k/10 = ~1k. Challenge windows: 500 fires/5s = 100/s. Reconnection: rare. |

---

## 8.3 First-Round Surge (T=0 Spike)

The first-round kickoff is the peak transient load — 100k rooms must be created in seconds.

### Kickoff Timeline

| T (seconds) | Event | Rate |
|-------------|-------|------|
| T+0 | `StartTournament` command | 1 event |
| T+0.1 | `tournament-service` shuffles 1M players, partitions into 100k rooms | CPU-bound, ~100 ms |
| T+0.2 | Enqueue 100k work items to `tournament.kickoff-work` | ~100k messages |
| T+0.2 → T+10 | `round-kickoff-worker` (10 shards × 1,000 rooms/s) publishes `TournamentRoomAssigned` | 10,000 rooms/s |
| T+0.2 → T+12 | `room-service` creates rooms (with 2 s buffer) | ~10,000 creates/s |
| T+10 → T+15 | All rooms initialized, games start dealing cards | ~300k events in ~5 s |

### Room Creation DB Load

Each room creation involves:
- 1 INSERT into `rooms`
- 10 INSERTs into `player_slots`
- 1 `InitializeGame` RPC → game-engine
- game-engine: 3 events (DeckInitialized, InitialHandsDealt, FirstCardFlipped) + 3 outbox rows

**Per room:** ~17 DB operations.
**Aggregate:** 10,000 rooms/s × 17 ops = **~170,000 DB ops/s** for ~10 seconds.
**Per instance:** With 50 `room-service` instances: ~3,400 ops/s/instance. With 200 `game-engine` instances: ~850 ops/s/instance for initialization events.

PostgreSQL handles this with connection pooling and prepared statements. The burst is short-lived (~10 s).

### Kafka Burst

- `tournament.rooms`: 10,000 events/s for 10 s (100k total). 64 partitions → ~156 events/s/partition.
- `gameplay.events`: ~60k events/s for ~5 s (deck init + deal for 100k games). Spike above steady-state 25k/s. Kafka buffers absorb short bursts.

---

## 8.4 Round-End Spike (GameCompleted Burst)

Games within a round don't finish simultaneously — game durations vary (~5–20 min). But the majority finish within a ~10-minute window, creating a burst of completion events.

### Burst Profile

| Metric | Value |
|--------|-------|
| Total GameCompleted events | 100,000 (one per room, final game) |
| Burst window | ~10 minutes (~600 s) |
| Average rate | ~167 GameCompleted/s |
| Peak burst (80th percentile of games finishing) | ~500 GameCompleted/s |

### Consumer Impact

| Consumer | Load | Mitigation |
|----------|------|------------|
| `room-service` | ~500 GameCompleted/s → evaluate match → emit MatchCompleted | Consumer group scales with partitions. |
| `tournament-service` | ~500 RoomCompleted/s → counter increment | SERIALIZABLE per-round TX. Idempotent. Not a bottleneck (single UPDATE). |
| `ranking-service` | ~500 GameCompleted/s × 10 players = ~5,000 Elo updates/s (but tournament = skip Elo, stats only) | For tournament games, only stats updated (lighter load). |
| `spectator-projection-service` | ~500 projection updates/s (game → completed) | Light: just update `gamePhase` field. |
| `audit-service` | ~500 events/s (additional to steady-state) | Append-only, batch inserts. |

**No backpressure to Room Gameplay:** Each consumer has its own consumer group. Slow consumers do not block game-engine writes.

---

## 8.5 Spectator Fan-Out

### Average Case

| Metric | Value |
|--------|-------|
| Concurrent spectated games (assuming 20% of 100k rooms have spectators) | ~20,000 games |
| Average spectators per watched game | ~250 (5M total / 20k games) |
| Events per game per second | ~0.25 |
| Spectator events per second (total) | ~5,000 |
| SSE pushes per second (total) | ~1,250,000 (5k events × 250 spectators) |
| Per gateway instance (600 instances) | ~2,000 SSE pushes/s |

### Popular Rooms (Tournament Finals)

| Metric | Value |
|--------|-------|
| Spectators on final room | 100,000+ |
| Events per second | ~0.25 |
| SSE pushes per second | ~25,000 |

**Mitigation for popular rooms:**
- **Regional edge proxies** (declared deployable in `02-container-view.md` §2.3 and integration-table S13) subscribe to 1 upstream SSE and fan out to local spectators.
- 10 regional edges × 10,000 spectators each = 100k spectators served.
- Upstream: 10 SSE connections (one per edge). Downstream: 100k SSE connections (distributed).
- This is the enforcement plan for the 100:1 finals ratio. The practical hard cap is ~100k spectators per room with 10 regional edges (expandable to 20 edges for 200k). Domain does not set a lower bound — edge proxy capacity is the operative constraint.

---

## 8.6 Horizontal Scaling Summary

| Component | Peak Instances | Bottleneck | Scaling Lever | Singleton Risk |
|-----------|---------------|------------|---------------|----------------|
| `api-gateway` | 600 | Connection count (6M total) | Add instances (stateless) | None |
| `identity-service` | 10 | DB writes on login surge | Redis cache absorbs reads | None |
| `room-service` | 50 | Room creation burst at T=0 | Kafka consumer group partitions | None |
| `game-engine` | 200 | Event store writes (100k cmd/s) | Partition by `gameId` | None |
| `timer-service` | 10 | Poll + fire (101k active timers) | Shard by deadline bucket | None |
| `tournament-service` | 5 | Completion counter (per-round) | Partition by `tournamentId` | Low (few concurrent tournaments) |
| `round-kickoff-worker` | 10–20 | Kafka produce rate at T=0 | Add shards | None |
| `ranking-service` | 20 | Elo updates at round end (~8k/s) | Consumer group partitions | None |
| `spectator-projection-service` | 30 | Event consumption + projection writes (25k/s) | Consumer group partitions + read replicas | None |
| `audit-service` | 10 | Append-only ingestion (~30k/s) | Consumer group partitions, batch inserts | None |
| **Kafka** | 5 brokers | 25 MB/s write, 100 MB/s read | Add brokers, increase partitions | None (clustered) |
| **PostgreSQL (RG)** | 3–N (primary shard(s) + 2 replicas each) | 100k writes/s | Horizontal write sharding by `gameId` range (each shard owns a contiguous `gameId` range; `game-engine` instances route writes to their shard's primary). At 1M-player scale, 4 shards × ~25k writes/s each is well within cloud PostgreSQL limits (provisioned IOPS io2, 32k IOPS per shard). Single-shard primary is sufficient for ≤ 50k games (i.e., roughly the first tournament round); sharding is activated before scaling beyond that. | Write sharding is the concrete path to scale; partition affinity at the service layer (each `game-engine` instance is pinned to a `gameId` range) makes routing deterministic without distributed transactions. |
| **Redis** | 3–6 (cluster) | 1M+ reads/s (session cache) | Cluster sharding | None (clustered) |

---

## 8.7 Capacity Headroom

### 2M Total-Player Derivation

The "~2M total players" estimate is derived from the combination of simultaneous casual-play and tournament capacity:

| Constraint | Capacity at current design | Reasoning |
|------------|---------------------------|-----------|
| Gateway connections | 6M (600 instances × 10k each) | 1M tournament players + 5M spectators = 6M peak. Remaining headroom supports 1M additional casual players (1M WS + ≈0 spectators for casual) within the same 6M connection budget. |
| Redis session cache | 1M reads/s (cluster) | Each active session generates ~1 read/request. At 2M simultaneous active sessions (each issuing ~0.5 requests/s average), that is ~1M reads/s — at the cluster limit. |
| RG PostgreSQL | 100k writes/s with single shard (2 shards → 200k writes/s) | 1M tournament games + an additional 100k casual games = ~110k total games, each at ~1 cmd/s = ~110k writes/s. Within 2-shard capacity. |
| Kafka (gameplay.events) | 25k events/s sustained per producer group | 1.1M total games × 0.25 events/s = ~275k events/s requires ~11 parallel producer groups / 11× partition count. This is the binding constraint: expanding to 2M concurrent players requires scaling Kafka partitions from 128 to 256+. |

**Conclusion:** The "~2M total players" estimate reflects a realistic upper bound where 1M play in a tournament and 1M play in casual rooms simultaneously. The binding constraints are the Redis session cache (1M reads/s) and Kafka partition count. Scaling to 5M requires multi-region deployment.

| Scaling Dimension | Current Design Supports | Hard Limit | Path to Scale Further |
|-------------------|------------------------|------------|----------------------|
| Total players | ~2M (1M tournament + 1M casual, per derivation above) | ~5M before multi-region needed | Geo-partitioned tournaments |
| Concurrent games | ~200k | ~500k before Kafka/DB bottleneck | Increase Kafka partitions (128+), more game-engine instances |
| Spectators | ~10M | ~20M before edge capacity limit | More regional edges, CDN-backed SSE |
| Event throughput | ~50k events/s | ~100k events/s per Kafka cluster | Multi-cluster Kafka, tiered topics |
| DB write throughput (RG) | ~100k writes/s (single shard, provisioned IOPS) | ~400k writes/s with 4-shard horizontal sharding | Add `gameId`-range shards; `game-engine` instances pin to their shard. Sharding is the concrete next step once a single primary approaches its WAL throughput ceiling (~50k writes/s sustained with fsync). |

---

## 8.8 Key Numbers at a Glance

| Metric | Value |
|--------|-------|
| Peak concurrent long-lived connections | ~6M (1M WS + 5M SSE) |
| Peak aggregate command rate | ~100,000 cmd/s |
| Peak event production (gameplay.events) | ~25,000 events/s |
| Peak Kafka throughput | ~25 MB/s write, ~100 MB/s read |
| First-round surge room creation | ~10,000 rooms/s for ~10 s |
| Peak SSE fan-out | ~1,250,000 pushes/s |
| Total gateway instances | ~600 |
| Total game-engine instances | ~200 |
| Total service instances (all) | ~950 |
