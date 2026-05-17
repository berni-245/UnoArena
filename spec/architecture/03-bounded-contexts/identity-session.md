# Identity & Session — Bounded Context Architecture

> **Design reference:** [Spec §2.1.1](../../spec/docs/02-bounded-contexts-and-context-map.md), [Commands §4.2.1](../../spec/docs/04-commands-and-domain-events.md)

---

## Purpose and Scope

**Owns:** Player identity (registration, credentials, display-name uniqueness), authentication (JWT issuance and validation), session lifecycle, and the **single-active-session invariant**.

**Does not own:** Authorization policies per resource (each downstream context enforces its own), player gameplay state, Elo rating, or tournament registration.

---

## Services (Containers)

| Service | Responsibility |
|---------|---------------|
| `identity-service` | Handles all IS commands: `RegisterPlayer`, `Login`, `Logout`, `InvalidateSession`. Manages Player Identity and Session aggregates. Issues JWTs. Enforces single-active-session via atomic compare-and-swap on session creation. Publishes session lifecycle events to the broker. |

**Why a single service (not split identity + session)?** Both aggregates are low-throughput and share the transactional requirement of atomic CAS on login (invalidate old session + create new session in one TX). Splitting would add distributed TX complexity for no scaling benefit — the service is horizontally scalable as-is.

---

## Public Interfaces

### Synchronous (REST)

| Endpoint | Method | Auth | Description | Design Command |
|----------|--------|------|-------------|----------------|
| `/api/v1/players` | POST | None | Register new player | `RegisterPlayer` |
| `/api/v1/sessions` | POST | None | Login — creates session, returns JWT | `Login` |
| `/api/v1/sessions/{sessionId}` | DELETE | Bearer JWT | Logout — invalidates session | `Logout` |
| `/api/v1/sessions/{sessionId}` | DELETE | Admin JWT | Admin force-invalidate | `InvalidateSession` |
| `/api/v1/sessions/validate` | POST | Internal (mTLS) | Token introspection for gateway. Returns `{valid, playerId, sessionId, expiresAt}`. | (Query, no design command) |

**Versioning:** URL-path versioned (`/v1/`). Breaking changes get a new version; old version deprecated with sunset header.

**Auth expectations:**
- `RegisterPlayer` and `Login` are unauthenticated.
- All other endpoints require a valid JWT (Bearer token in Authorization header).
- Admin endpoints require `role: admin` claim in JWT.
- `/sessions/validate` is internal-only (mTLS, not exposed through gateway's public routes).

### Asynchronous (Kafka)

| Topic | Event | Payload Ownership | Idempotency Key | Consumers |
|-------|-------|-------------------|-----------------|-----------|
| `identity.sessions` | `SessionEstablished` | IS produces | `eventId` | `audit-service` |
| `identity.sessions` | `SessionInvalidated` | IS produces | `eventId` | `api-gateway`, `game-engine`, `audit-service` |
| `identity.sessions` | `PlayerRegistered` | IS produces | `eventId` | `ranking-service` (init Elo 1200), `audit-service` |
| `identity.sessions` | `PlayerSuspended` | IS produces | `eventId` | `tournament-service`, `audit-service` |
| `identity.sessions` | `PlayerBanned` | IS produces | `eventId` | `tournament-service`, `audit-service` |

**Partitioning:** All events on `identity.sessions` are keyed by `playerId` — ensures ordering of session events for a given player.

---

## Internal-Only Interfaces

| Interface | Description |
|-----------|-------------|
| Password hashing pipeline | Argon2id re-hash on server side. Never exposed. |
| Session CAS (compare-and-swap) | Internal DB operation: atomically set existing active session to `invalidated` + insert new session row, within a single SERIALIZABLE transaction. |

---

## Dependencies on Other Contexts

**None.** Identity & Session is a pure upstream context. It does not consume events from any other context.

---

## Invariant: Single-Active-Session with Push-Invalidation

This is the invariant the assignment requires an explicit architectural home for (§2, §6.1).

### Problem

Revoking a token in the database is insufficient. The old session may hold a live WebSocket or SSE connection through the `api-gateway`. That connection must be terminated promptly — not "eventually when the token expires."

### Mechanism

```
┌──────────────┐   1. Login(email, pass)        ┌───────────────────┐
│   Player B   │ ──────────────────────────────► │ identity-service  │
│  (new login) │                                 │                   │
└──────────────┘                                 │  2. BEGIN TX      │
                                                 │     UPDATE sessions SET status='invalidated'
                                                 │       WHERE playerId=X AND status='active'
                                                 │     INSERT session (newSessionId, playerId, 'active')
                                                 │  3. COMMIT TX     │
                                                 │                   │
                                                 │  4. Publish to Kafka:
                                                 │     SessionInvalidated {
                                                 │       sessionId: oldSessionId,
                                                 │       playerId: X,
                                                 │       reason: "new_login"
                                                 │     }
                                                 │  5. Return JWT (newSessionId)
                                                 └─────────┬─────────┘
                                                           │
                                                           ▼
                                                 ┌───────────────────┐
                                                 │   Kafka topic:    │
                                                 │ identity.sessions │
                                                 └─────────┬─────────┘
                                                           │
                                          ┌────────────────┼────────────────┐
                                          ▼                                 ▼
                                 ┌─────────────────┐              ┌──────────────┐
                                 │   api-gateway    │              │ game-engine   │
                                 │                  │              │              │
                                 │ 6. Lookup        │              │ 8. If player │
                                 │    connectionMap  │              │    in active │
                                 │    [oldSessionId] │              │    game:     │
                                 │                  │              │    emit      │
                                 │ 7. Send close    │              │    PlayerDis-│
                                 │    frame on old  │              │    connected │
                                 │    WebSocket/SSE │              │    (starts   │
                                 │    with reason:  │              │    60s timer)│
                                 │    "session_     │              │              │
                                 │    superseded"   │              └──────────────┘
                                 │                  │
                                 │    Remove entry  │
                                 │    from map.     │
                                 └─────────────────┘
```

**Key properties:**
1. **Atomic DB write** — old session invalidated + new session created in single TX (SERIALIZABLE isolation). No window where two sessions are simultaneously active.
2. **Push to gateway** — `api-gateway` subscribes to `identity.sessions` topic, maintains an in-memory `sessionId → connectionRef` map. On `SessionInvalidated`, looks up the old session's connection and sends a WebSocket close frame (code 4001: "session_superseded") or terminates the SSE stream.
3. **Push to game-engine** — `game-engine` also subscribes. If the player is mid-game, emits `PlayerDisconnected` and starts the 60-second reconnection timer. Since the session is now invalid, any reconnect attempt with the old token will fail → timer expires → `PlayerForfeited`.
4. **Latency** — typical Kafka consumer lag: 50–200ms. The old connection is closed within ~200ms of the new login TX commit.
5. **Gateway crash recovery** — if the gateway instance holding the old connection crashes before processing the `SessionInvalidated` event, the client's TCP connection dies anyway. On reconnect, the client must present a valid session token — the old token is already invalidated in DB.

### Failure Modes

| Failure | Impact | Recovery |
|---------|--------|----------|
| Gateway misses `SessionInvalidated` (consumer lag) | Old connection lives briefly longer | Gateway also validates token on heartbeat (every 30s). Stale token caught on next heartbeat. |
| Kafka unavailable during login | `SessionInvalidated` event not published | identity-service publishes via transactional outbox (writes event to outbox table in same TX as session update). Outbox relay retries until Kafka is available. |
| Gateway has no entry for `oldSessionId` (player wasn't connected) | No-op | Harmless — no connection to close. |

---

## Persistence

| Store | Technology | Data | Consistency |
|-------|-----------|------|-------------|
| Primary | PostgreSQL | `player_identities` (credentials, display names, status), `sessions` (sessionId, playerId, status, expiresAt, deviceFingerprint) | Strong (SERIALIZABLE for session CAS) |
| Cache | Redis | Hot session tokens for gateway validation. TTL = session expiry. Invalidated on `SessionInvalidated`. | Eventual (write-through from identity-service, invalidate on event) |

**Idempotency store:** `command_idempotency` table (commandId → response, TTL 24h). Checked before any command processing.
