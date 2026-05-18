# 11. Non-Functional Requirements (NFR) Matrix

> Latency budgets, throughput targets, and availability requirements for UnoArena's major flows and services. Numbers are grounded in the capacity sketch (§8) and the domain's timing invariants (V9, V10, A30, A38). All p-values are server-side unless noted.

> **Measurement scope — read this first:** All latency budgets in §11.1 are measured **server-side**: from the moment a WebSocket frame or HTTP request is received by the gateway to the moment the response leaves the gateway back toward the client. They do **not** include internet round-trip time (RTT). A player's perceived latency is always `server-side budget + RTT`, where RTT varies from ~10 ms (local) to ~300 ms (intercontinental). For a globally distributed platform, regional deployments are the primary lever for reducing perceived latency; the server-side budgets defined here remain constant regardless of where the player is located.

---

## 11.1 Latency Budgets per Flow

### 11.1.1 PlayCard (Game Command Hot Path)

The PlayCard flow is the single most latency-sensitive path: the player issues a command, the server validates, writes to the event store + outbox atomically, ACKs the WebSocket, and the outbox relay publishes to Kafka. Downstream consumers (spectator projection, audit) are asynchronous and do not block the ACK.

| Stage | Budget | Owner | How Met |
|-------|--------|-------|---------|
| WebSocket frame → `game-engine` handler | < 5 ms | `api-gateway` → `game-engine` | Co-located or same-AZ; no synchronous cross-service hop beyond the internal network |
| Idempotency + sequence-number check | < 2 ms | `game-engine` (in-memory aggregate cache) | Game aggregate is held in memory; DB read only on cache miss or restart |
| Invariant validation (turn, legality, hand) | < 1 ms | `game-engine` | Pure in-memory computation |
| `BEGIN TX` → event store + outbox `COMMIT` | **< 30 ms p95** | PostgreSQL (per-instance) | Cloud-managed PostgreSQL on fast block storage (e.g., AWS EBS gp3); 2 inserts per command; PgBouncer connection pool. p50 is typically 2–5 ms; p95 reaches 20–30 ms on cloud storage under sustained write load. |
| WebSocket ACK to player | **< 50 ms p95** | | Aggregate of above stages; dominated by the PostgreSQL commit |
| **PlayCard p99 budget** | **< 150 ms** | | Covers cloud storage tail latency, GC pauses, minor network jitter; p99 PostgreSQL commits can reach 50–80 ms under peak load |
| Outbox relay → Kafka publish | + 50–100 ms | Outbox relay worker | Polling interval 50 ms; downstream consumers see update within ~100–200 ms of commit |

**Why the DB commit dominates:** With 500 cmd/s per `game-engine` instance, each instance issues ~1,000 PostgreSQL writes/s (event row + outbox row). WAL I/O on cloud block storage at that rate typically produces p95 commits in the 20–30 ms range. This is the irreducible cost of the log-before-broadcast guarantee (V12): durability requires a synchronous fsync before the ACK is sent.

**Player-perceived latency examples (server-side + RTT):**

| Player location | Typical RTT to server | Perceived PlayCard p95 |
|----------------|----------------------|----------------------|
| Same city | ~10–20 ms | **60–70 ms** — imperceptible |
| Cross-country | ~60–100 ms | **110–150 ms** — acceptable for a card game |
| Intercontinental | ~150–300 ms | **200–350 ms** — noticeable; regional deployment recommended |

**Domain constraint:** Sequence-number enforcement (V13) and the log-before-broadcast rule (V12) impose no additional latency — both are satisfied within the single PostgreSQL transaction.

---

### 11.1.2 Session Validation (Every Command)

Every authenticated command passes through the `api-gateway`, which validates the JWT and checks session liveness before forwarding to a backend service.

| Path | Budget | How Met |
|------|--------|---------|
| JWT signature verification (local) | < 1 ms | Symmetric HMAC or RSA verify in-process; no network hop |
| Session liveness check (Redis cache) | < 5 ms p99 | Server-side network call to same-AZ Redis. Same-AZ Redis p50 is 0.5–1 ms; p99 is 3–5 ms with connection pooling. This is not visible to the client — it is part of the server-side budget. Cache miss (rare, on invalidation) triggers a synchronous `identity-service` call and may add 10–20 ms. |
| **Total validation overhead per command** | **< 10 ms p99** | Added to the server-side processing time on every command path; already included within the PlayCard 50 ms p95 budget above |

**Design reference:** `identity-session.md` §Persistence — Redis cache for hot session tokens, TTL = session expiry, invalidated on `SessionInvalidated` event.

---

### 11.1.3 Session Invalidation → Live Connection Close

When a player logs in on a new device, the old session's live WebSocket/SSE connection must be closed promptly (§6.1 non-negotiable).

| Stage | Budget | How Met |
|-------|--------|---------|
| `identity-service` DB commit → Kafka publish | < 50 ms | Transactional outbox; relay polling 50 ms |
| Kafka `identity.sessions` → `api-gateway` consumer | < 100–200 ms | Single Kafka consumer hop |
| Gateway lookup + WebSocket close frame | < 5 ms | In-memory `sessionId → connectionRef` map |
| **Old connection closed after new login** | **< 300 ms p95** | End-to-end; aligns with §7.1.3 analysis (sub-second window) |

---

### 11.1.4 5-Second Uno! Challenge Window

The challenge window is a hard domain invariant (V10, INV-G-06). Timing is server-authoritative (A7).

| Requirement | Budget | How Met |
|-------------|--------|---------|
| Window open event → clients | < 150 ms | Outbox relay (50 ms) + Kafka + client delivery |
| Challenge command accepted | Window must be `Open` at server processing time | `timer-service` deadline check; no client-clock dependency |
| Window expiry fired by `timer-service` | `expiresAt ± 100 ms` | Polling interval 100 ms on `timer_deadlines` table; acceptable given 5s window |
| Late challenge (arrives after close) | Rejected with `ChallengeWindowClosed` | `game-engine` checks window state on every `ChallengeUnoCall` |

**Note:** A ±100 ms expiry jitter on a 5-second window represents 2% error — acceptable for gameplay and consistent with A7 (server clock is authoritative, client latency is not counted).

---

### 11.1.5 60-Second Reconnection Window

Domain invariant: exactly 60 seconds from disconnection detection (V9, A4).

| Requirement | Budget | How Met |
|-------------|--------|---------|
| Connection drop → `PlayerDisconnected` emitted | < 10 s | Heartbeat detection (A4: 5–10 s); `game-engine` emits event + inserts timer deadline |
| Timer fires within window ± 200 ms | `expiresAt ± 200 ms` | `timer-service` polling; 60-second window makes 200 ms jitter negligible |
| `AutoForfeit` idempotent on double-fire | No double-forfeit | `timer-service` `version` field; `game-engine` checks window state |

---

### 11.1.6 Tournament First-Round Surge (Room Kickoff)

Domain requirement: ~100,000 rooms must be created and started within a bounded window after `TournamentStarted`.

| Metric | Target | How Met |
|--------|--------|---------|
| Time to publish all 100k `TournamentRoomAssigned` events | **≤ 60 seconds** | 10–20 `round-kickoff-worker` shards × 1,000 rooms/s/shard = 10,000 rooms/s → 10 s for 100k |
| Time from `TournamentRoomAssigned` → `GameStarted` | **≤ 120 seconds** | `room-service` consumer lag + room creation + deck init pipeline |
| `tournament-service` responsiveness during kickoff | Status queries < 500 ms | Kickoff work offloaded to `round-kickoff-worker`; `tournament-service` state machine unblocked |

---

### 11.1.7 Elo Update Pipeline (Eventual)

Elo is eventual: the pipeline runs asynchronously after `GameCompleted` is published.

| Metric | Target | How Met |
|--------|--------|---------|
| `GameCompleted` publish → Elo update persisted | **< 10 s p95** under normal load | Kafka consumer lag (~1 s) + per-player atomic write |
| Elo update during round-end burst (100k events) | **< 5 minutes** for full cohort | Horizontal `ranking-service` instances; Kafka consumer group distributes partitions |
| Leaderboard visible after Elo update | **< 1 s** | Redis ZADD after atomic DB write; `leaderboard:global` sorted set |
| Leaderboard read latency | **< 50 ms p99** | Direct Redis ZRANGEBYSCORE; no DB hit |

---

### 11.1.8 Spectator Projection Update

The spectator view is eventually consistent with a deliberate delay (A35).

| Metric | Target | How Met |
|--------|--------|---------|
| `CardPlayed` → `SpectatorCardPlayed` visible | **≤ 500 ms p95** | Outbox (50 ms) + Kafka (50–100 ms) + ACL transform + DB write + SSE push |
| Spectator SSE delivery ordering per game | Causal order guaranteed | Kafka partition key = `gameId`; single consumer per partition maintains order |

---

## 11.2 Throughput Targets per Service

| Service | Peak Throughput | Scaling | Notes |
|---------|----------------|---------|-------|
| `api-gateway` | **6M concurrent connections** (1M WS + 5M SSE) | Horizontal; ~10k connections/instance → 600 instances | SSE fan-out via edge/regional proxies reduces gateway load |
| `game-engine` | **~100,000 cmd/s** aggregate | Horizontal; partitioned by `gameId`; ~200 instances × 500 cmd/s/instance | Each game is single-writer; no cross-instance coordination |
| `room-service` | ~5,000 room-lifecycle events/s at round start | Horizontal; partitioned by `roomId` | Low-frequency relative to `game-engine` |
| `tournament-service` | ~100,000 `RoomCompleted` events within a round window | Horizontal; partitioned by `tournamentId` | Atomic counter with idempotency guard; burst is bounded |
| `round-kickoff-worker` | **10,000 `TournamentRoomAssigned`/s** (10 shards × 1k/s) | Horizontal; sharded by player-range hash | Rate-controlled; backpressure from Kafka consumer lag |
| `ranking-service` | ~100,000 `GameCompleted` events per round end | Horizontal; Kafka consumer group | Per-player atomic write; independent across players |
| `spectator-projection-service` | ~25,000 events/s ingested; fan-out to ~5M SSE | Horizontal; consumer group + edge SSE proxies | SSE fan-out is the bottleneck, not ingestion |
| `audit-service` | ~30,000 events/s (all topics combined) | Horizontal; consumer group | Append-only writes; scales with ClickHouse/PostgreSQL write throughput |
| `identity-service` | ~50,000 token validations/s (mostly Redis, not `identity-service`) | Horizontal; Redis absorbs read load | New login and registration much lower frequency |
| Kafka `gameplay.events` topic | **~25,000 events/s** sustained; **~60,000 events/s** burst at round-start (deck init + deal for 100k games, ~5 s window per §8.3); round-end is a traffic decrease, not a burst, as games stop generating card-play events | Partitioned by `gameId`; target ≥ 128 partitions | Round-start burst is the true peak; Kafka buffers absorb the ~5 s spike. Consumers sized for 25k/s sustained (4× fan-out → ~100k events/s read rate) comfortably handle 60k/s producer burst. |

---

## 11.3 Availability Targets per Service

| Service | Target SLO | Rationale | Failure Impact |
|---------|-----------|-----------|----------------|
| `api-gateway` | **99.95%** (~4.4 h/year) | All client traffic flows through gateway; outage means no gameplay | Active-active; health-checked; graceful drain on rolling deploys |
| `game-engine` | **99.9%** (~8.7 h/year) | In-progress games lose connectivity on crash; 60 s reconnection window covers short outages | Event-sourced state; Game aggregate rebuilt from event store on restart |
| `room-service` | **99.9%** | Room lifecycle decisions; tournament rooms auto-started | Persistent state in PostgreSQL; crash-safe |
| `identity-service` | **99.9%** | New logins blocked during outage; in-progress sessions continue (cached in Redis) | Redis cache provides read-through during brief IS outage |
| `tournament-service` | **99.9%** (during active tournaments) | Round advancement blocked if unavailable | Saga state persisted; resumes from Kafka offset after restart |
| `round-kickoff-worker` | **99.5%** | Delayed kickoff acceptable; Kafka consumer group rebalances | Work queue is durable in Kafka; resumes from committed offset |
| `timer-service` | **99.9%** | Missed timer fires delay challenge window expiry or reconnection forfeit | Deadlines persisted in PostgreSQL; recovered on restart via re-scan |
| `ranking-service` | **99.5%** | Elo updates delayed; no gameplay impact | Kafka re-delivers `GameCompleted`; idempotency prevents double-update |
| `spectator-projection-service` | **99.5%** | Spectators lose feed; players unaffected | Projection rebuilt from Kafka replay on restart |
| `audit-service` | **99.5%** | Audit lag; no gameplay impact; events buffered in Kafka | Idempotent append; catches up after recovery |

**Note:** SLOs are per-instance class, not per individual instance. All services run multiple instances behind load balancers or Kafka consumer groups. Individual instance failure does not constitute an SLO breach.

---

## 11.4 How the Architecture Meets These Targets

### 11.4.1 Low Latency for Gameplay Commands

- **Event sourcing + in-memory aggregate:** `game-engine` maintains a materialized view of the Game aggregate in memory. Most validation is pure computation — no synchronous DB reads on the hot path.
- **Transactional outbox in same DB:** The event store append and outbox row are a single PostgreSQL commit. No two-phase commit, no distributed transaction.
- **Partitioning by `gameId`:** Each game-engine instance owns a disjoint set of games. No inter-instance coordination on the command path.

### 11.4.2 High Availability for In-Progress Games

- **60-second reconnection window (V9):** Covers brief `game-engine` restarts; players reconnect after the instance recovers.
- **Event-sourced recovery (doc 07 §7.5.1):** Game aggregate is deterministically rebuilt from `game_events` log after crash. No in-memory state is lost permanently.
- **Durable timer deadlines:** `timer_deadlines` table persists all open windows. `timer-service` rescans on restart and fires any overdue deadlines.

### 11.4.3 Scale for Tournament Surge

- **Sharded kickoff workers:** Fan-out is controlled, rate-limited, and backpressure-aware. The `tournament-service` is never the bottleneck for room creation.
- **Kafka consumer group elasticity:** `game-engine`, `ranking-service`, and `spectator-projection-service` all scale horizontally by adding Kafka consumer group members.
- **Independent write paths:** Each `game-engine` instance writes to its own PostgreSQL shard. No shared write path exists across the 100k simultaneous games.

### 11.4.4 Eventual Consistency Bounds

The following cross-context flows are eventually consistent with explicit staleness bounds:

| Flow | Acceptable Staleness | Why Acceptable |
|------|---------------------|----------------|
| `GameCompleted` → Elo visible in leaderboard | < 5 minutes | Elo is a statistical summary; one delayed update does not affect fairness of an in-progress game |
| `GameCompleted` → spectator bracket update | < 1 second | Cosmetic; spectators expect slight delay (A35) |
| `SessionInvalidated` → gameplay command rejected | < 300 ms | Commands issued in this window are legitimate actions by the player; bounded window is accepted (doc 07 §7.1.3) |
| `RoomCompleted` → tournament round counter | < 2 seconds | Counter update is idempotent; eventually reaches `totalRooms` |
