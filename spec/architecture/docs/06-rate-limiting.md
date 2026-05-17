# 6. Rate Limiting Architecture

> Maps multi-layer rate limiting to concrete deployables per §6.3 of the Architecture Checkpoint. Each layer defends against a distinct threat class, and together they form a defense-in-depth chain from edge to service.

---

## 6.1 Overview

Rate limiting is enforced at four layers, each progressively closer to business logic:

```
┌──────────────────────────────────────────────────────────────────┐
│                        PUBLIC INTERNET                           │
└──────────────────────────┬───────────────────────────────────────┘
                           │
                    ┌──────▼──────┐
                    │ LAYER 1     │  Per-IP token bucket (Redis)
                    │ api-gateway │  Before authentication
                    │ (edge)      │  Defense: DDoS, brute-force
                    └──────┬──────┘
                           │ JWT validated
                    ┌──────▼──────┐
                    │ LAYER 2     │  Per-User sliding window (Redis)
                    │ api-gateway │  After JWT validation
                    │ (post-auth) │  Defense: authenticated abuse
                    └──────┬──────┘
                           │ mTLS / private network
            ┌──────────────▼──────────────┐
            │ LAYER 3                      │  Per-Room/Tournament action
            │ game-engine, room-service,   │  In-process middleware
            │ tournament-service           │  Defense: game-level abuse
            └──────────────┬──────────────┘
                           │
            ┌──────────────▼──────────────┐
            │ LAYER 4                      │  Adaptive throttling
            │ api-gateway + services       │  Latency-based shedding
            │                              │  Defense: cascade failure
            └─────────────────────────────┘
```

**Principle:** Each layer operates independently. A request must pass all layers to reach business logic. Earlier layers are cheaper (no DB, no business context) and shed the most traffic.

---

## 6.2 Layer 1: Per-IP (Edge / API Gateway)

| Attribute | Value |
|-----------|-------|
| **Deployable** | `api-gateway` — Nginx/Envoy rate-limiting module |
| **Identity** | Source IP address (from TCP connection or `X-Forwarded-For` with trusted proxy chain) |
| **Scope** | Global per IP, applied **before** authentication |
| **Algorithm** | Token bucket in Redis |
| **Store** | Redis — key: `rl:ip:{sourceIP}`, TTL: 60 s |

### Limits

| Endpoint Class | Limit | Rationale |
|---------------|-------|-----------|
| REST API requests | 100 req/s per IP | General abuse prevention |
| WebSocket messages | 50 msg/s per IP | Prevent command flooding |
| SSE connections | 20 concurrent per IP | Prevent connection exhaustion |
| Login attempts (`POST /sessions`) | 10/min per IP | Brute-force credential protection |
| Registration (`POST /players`) | 5/min per IP | Bot registration prevention |

### Exceeded Behavior

- **REST:** HTTP 429 `Too Many Requests` with `Retry-After` header.
- **WebSocket:** Close frame with code 4029 and reason `rate_limit_exceeded`.
- **SSE:** HTTP 429 on new connection attempts (existing connections unaffected).

### Why at This Layer

Per-IP is the only identity available before authentication. Catches volumetric DDoS, credential stuffing, and bot floods before they reach application logic. Zero business-logic cost.

---

## 6.3 Layer 2: Per-User (API Gateway, Post-Auth)

| Attribute | Value |
|-----------|-------|
| **Deployable** | `api-gateway` — custom middleware, executed **after** JWT validation |
| **Identity** | `playerId` extracted from validated JWT claims |
| **Scope** | Per authenticated user, across all endpoints |
| **Algorithm** | Sliding window counter in Redis |
| **Store** | Redis — key: `rl:user:{playerId}`, TTL: sliding window duration |

### How Identity Is Obtained

1. Gateway validates JWT on WebSocket upgrade or REST request.
2. Extracts `playerId` and `sessionId` from token claims.
3. Checks session validity against Redis cache (hot path) or `identity-service` `/sessions/validate` (cache miss, mTLS).
4. After validation, `playerId` is the rate-limit key for Layer 2.

### Limits

| Action Class | Limit | Rationale |
|-------------|-------|-----------|
| Gameplay commands (via WebSocket) | 30 cmd/s | Normal play is ~1-2 cmd/s; 30/s allows bursts but blocks automation |
| REST API requests | 10 req/s | Room creation, queries, etc. |
| Room join attempts | 5/min | Prevent room-hopping abuse |
| Tournament registration | 3/min | Prevent registration spam |

### Exceeded Behavior

- **REST:** HTTP 429 with `Retry-After`.
- **WebSocket:** Error frame `{ type: "command_rejected", reason: "rate_limited", retryAfterMs: 1000 }`.

### Why at This Layer

Catches abuse from authenticated users: automated play bots, rapid command spam, account sharing. Per-IP alone doesn't stop a distributed attacker using many IPs with stolen credentials.

---

## 6.4 Layer 3: Per-Room/Tournament Action (Service-Level)

| Attribute | Value |
|-----------|-------|
| **Deployable** | In-process middleware within `game-engine`, `room-service`, `tournament-service` |
| **Identity** | `playerId` from gateway-injected claims + `roomId`/`gameId`/`tournamentId` from request path/payload |
| **Scope** | Per `(playerId, gameId)` for gameplay; per `(playerId, roomId)` for room actions; per `(playerId, tournamentId)` for tournament |
| **Algorithm** | Sliding window counter in local memory |
| **Store** | In-process (no Redis) |

### Why Local Memory Is Acceptable

Requests are partitioned by aggregate ID (`gameId`, `roomId`). With consistent routing or Kafka partition affinity, commands for a given game land on the same `game-engine` instance. Local counters are therefore accurate per game. Cross-instance precision is not needed — the aggregate itself enforces invariants (sequence numbers, turn validation).

### How Scope Is Obtained

1. Gateway injects `playerId` into request context (trusted, post-auth).
2. `gameId` / `roomId` comes from the WebSocket message payload or REST URL path.
3. Rate limiter keys on `(playerId, gameId)` or `(playerId, roomId)`.

### Limits

| Action | Limit | Rationale |
|--------|-------|-----------|
| Gameplay commands per player per game | 10 cmd/s | Normal play: ~1-2/s. Allows fast legitimate play. Blocks bots. |
| Challenges (Uno/WDF) per player per game | 3 per 5 s window | Prevent challenge spam during challenge windows |
| Room join per room | 2 attempts/min | Prevent join/leave cycling |
| Tournament actions per player | 5/min | Registration, status queries |

### Exceeded Behavior

`CommandRejected { reason: "rate_limited", commandId, retryAfterMs }` — sent over WebSocket or as HTTP 429.

---

## 6.5 Layer 4: Adaptive Throttling

Adaptive throttling responds to system-wide load signals, not individual user behavior. It protects against cascade failures and resource exhaustion during peak load (tournament surges).

### 4a. Gateway Latency-Based Shedding

| Attribute | Value |
|-----------|-------|
| **Deployable** | `api-gateway` |
| **Signal** | Backend P99 response latency (measured per upstream service) |
| **Threshold** | P99 > 500 ms for gameplay services; P99 > 2 s for other services |
| **Action** | Shed traffic by priority tier (lowest priority first) |

**Priority Tiers (highest to lowest):**

| Tier | Traffic Type | Shed Under Load? |
|------|-------------|------------------|
| 1 (Critical) | Gameplay commands (PlayCard, DrawCard, CallUno) | Never shed |
| 2 (High) | Room management (Create, Join, Leave, Forfeit) | Last resort only |
| 3 (Medium) | Tournament registration, status queries | Shed when P99 > 1 s |
| 4 (Low) | Spectator SSE new connections | Shed when P99 > 500 ms |
| 5 (Lowest) | Leaderboard queries, player stats | Shed first |

### 4b. Service-Level Queue Depth

| Attribute | Value |
|-----------|-------|
| **Deployable** | `game-engine` |
| **Signal** | Internal command queue depth per instance |
| **Threshold** | Queue > 1,000 pending commands |
| **Action** | Reject new commands with `CommandRejected { reason: "server_busy" }` (HTTP 503 equivalent) |

### 4c. Tournament Kickoff Backpressure

| Attribute | Value |
|-----------|-------|
| **Deployable** | `round-kickoff-worker` |
| **Signal** | `room-service` consumer lag on `tournament.rooms` Kafka topic |
| **Threshold** | Consumer lag > 5,000 messages |
| **Action** | Reduce publish rate from 1,000 rooms/sec/shard to 500, then 250. Resume normal rate when lag < 1,000. |

### 4d. Circuit Breaker

| Attribute | Value |
|-----------|-------|
| **Deployable** | `api-gateway` and inter-service RPC callers |
| **Signal** | Error rate > 50% over 30 s window |
| **Action** | Open circuit → return 503 for 30 s → half-open (allow 10% traffic) → close if success rate > 90% |

---

## 6.6 Shared Infrastructure

### Redis Configuration

- **Cluster:** Shared Redis cluster for rate-limit counters (same cluster as session cache and leaderboard).
- **Key Schema:** `rl:{layer}:{scope}:{identifier}`
  - `rl:ip:203.0.113.42` — Layer 1, per-IP
  - `rl:user:player-uuid-1234` — Layer 2, per-user
- **TTL:** Keys expire after the window duration (1 min for most, 5 min for login). Automatic cleanup.
- **Consistency:** Best-effort. Redis is not strongly consistent — a counter may under-count by 1-2 in split-brain scenarios. Acceptable for rate limiting (false negatives are harmless; false positives are rare and self-correcting after TTL).

### Why Layer 3 Uses Local Memory (Not Redis)

- Requests for a given game are routed to the same `game-engine` instance (partition affinity via `gameId`).
- Local counters are accurate for the partition.
- Avoids Redis round-trip on the hot gameplay path (~0.5 ms saved per command).
- On instance restart, counters reset — acceptable (brief window of no limiting, quickly corrected).

---

## 6.7 Request Flow Example

A `play_card` command traverses all four layers:

```mermaid
sequenceDiagram
    participant P as Player
    participant GW as api-gateway
    participant R as Redis
    participant GE as game-engine

    P->>GW: play_card { cardId, seq=42 } (WebSocket)

    Note over GW: Layer 1: Per-IP
    GW->>R: INCR rl:ip:203.0.113.42
    R-->>GW: count=15 (< 50/s limit ✓)

    Note over GW: JWT validation
    GW->>GW: Validate JWT, extract playerId

    Note over GW: Layer 2: Per-User
    GW->>R: INCR rl:user:player-uuid
    R-->>GW: count=3 (< 30/s limit ✓)

    Note over GW: Layer 4a: Adaptive check
    GW->>GW: Backend P99 = 80ms (< 500ms ✓)

    GW->>GE: Relay command (mTLS)

    Note over GE: Layer 3: Per-Room
    GE->>GE: Local counter (player-uuid, game-123) = 2 (< 10/s ✓)

    Note over GE: Layer 4b: Queue depth
    GE->>GE: Queue depth = 42 (< 1000 ✓)

    GE->>GE: Process command (business logic)
    GE-->>GW: ACK
    GW-->>P: game_state_update
```

---

## 6.8 Summary Table

| Layer | Deployable | Identity Source | Scope | Algorithm | Store | Key Limits |
|-------|-----------|----------------|-------|-----------|-------|------------|
| **1. Per-IP** | `api-gateway` (Nginx/Envoy) | Source IP | Global per IP | Token bucket | Redis | 100 req/s REST, 50 msg/s WS, 20 SSE, 10 login/min |
| **2. Per-User** | `api-gateway` (post-auth middleware) | `playerId` from JWT | Per authenticated user | Sliding window | Redis | 30 cmd/s gameplay, 10 req/s REST, 5 joins/min |
| **3. Per-Room** | `game-engine`, `room-service`, `tournament-service` | `playerId` + `gameId`/`roomId` from request | Per (player, game/room) | Sliding window | Local memory | 10 cmd/s per game, 3 challenges/5s |
| **4. Adaptive** | `api-gateway` + all services | System-wide signals | Global | Latency/queue thresholds | N/A | Priority-based shedding, backpressure, circuit breaker |
