# Identity & Session — Bounded Context Architecture

> **Design reference:** [Spec §2.1.1](../../../domain/docs/02-bounded-contexts-and-context-map.md), [Commands §4.2.1](../../../domain/docs/04-commands-and-domain-events.md)

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
| `identity.sessions` | `SessionEstablished` | IS produces | `eventId` | `audit-service` *(gateway does not consume this event for cache warming — the session cache is warmed by `identity-service` itself via write-through at commit time, before the JWT is returned; the gateway never needs to wait for a Kafka delivery to have a valid Redis entry)* |
| `identity.sessions` | `SessionInvalidated` | IS produces | `eventId` | `api-gateway`, `game-engine`, `audit-service` |
| `identity.sessions` | `PlayerRegistered` | IS produces | `eventId` | `ranking-service` (init Elo 1200), `audit-service` |
| `identity.sessions` | `PlayerSuspended` | IS produces | `eventId` | `tournament-service`, `game-engine`, `room-service`, `audit-service` |
| `identity.sessions` | `PlayerBanned` | IS produces | `eventId` | `tournament-service`, `game-engine`, `room-service`, `audit-service` |

**Partitioning:** All events on `identity.sessions` are keyed by `playerId` — ensures ordering of session events for a given player.

---

## Internal-Only Interfaces

| Interface | Description |
|-----------|-------------|
| Password hashing pipeline | Argon2id re-hash on server side. Never exposed. |
| Session CAS (compare-and-swap) | Internal DB operation: atomically set existing active session to `invalidated` + insert new session row, within a single SERIALIZABLE transaction. |
| Adaptive throttle directives | When `identity-service` detects repeat-offender patterns (e.g., N failed logins from one IP within a window, or a session exhibiting bot-like command cadence), it writes a short-lived throttle directive to Redis: key `rl:adaptive:{targetType}:{targetId}` (e.g., `rl:adaptive:ip:203.0.113.42` or `rl:adaptive:player:{playerId}`), value `{ tier: "strict" | "block", expiresAt, reason }`, TTL = directive lifetime. The `api-gateway` checks this key as part of Layer 2 (post-auth) processing and applies the escalated limit tier. No new Kafka event is needed — Redis provides the sub-millisecond push path. |

### Throttle Directive Triggers

| Pattern | Target | Directive | Duration |
|---------|--------|-----------|----------|
| ≥ 5 failed logins in 60 s from same IP | `ip:{sourceIP}` | `strict` (login rate capped to 1/min) | 10 min |
| ≥ 10 failed logins in 300 s from same IP | `ip:{sourceIP}` | `block` (all requests 429) | 1 h |
| `playerId` suspended or banned | `player:{playerId}` | `block` (all authenticated requests rejected) | Until suspension lifted |
| Anomalous session creation rate (> 5 sessions/min for same `playerId`) | `player:{playerId}` | `strict` (session creation capped to 1/5min) | 30 min |

The `api-gateway` consults `rl:adaptive:*` keys on every authenticated request (Redis GET, same call as Layer 2 counter). Directive evaluation adds < 1 ms. On Redis unavailability, directives are skipped (fail-open for throttling, not fail-closed — availability takes precedence).

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

### Per-Session Command Validation

To close the 200–300 ms (worst-case 30 s on missed event) gap during which a superseded session can still send commands:

1. The JWT issued by `identity-service` carries both `playerId` **and** `sessionId` (already in the issuance design — now made authoritative for command validation).
2. `api-gateway` records `(playerId, currentSessionId)` in Redis on connection upgrade. On every gameplay command relayed to `game-engine`, the gateway forwards `sessionId` in the mTLS-injected context.
3. `game-engine`'s command middleware checks `sessionId` against an in-memory invalidated-session set (populated by the `SessionInvalidated` Kafka consumer with a 24 h TTL). Mismatch → `CommandRejected { reason: "session_superseded" }`, connection closed.
4. On Redis miss (cold instance), `game-engine` calls `identity-service` `/sessions/validate` synchronously (cache-fill, ≤ 5 ms p99 from the hot cache). Fail-closed on validate timeout.

This makes the single-active-session invariant authoritative on every in-flight command, not just at connection-upgrade time. The 30 s heartbeat fallback becomes a defense-in-depth backstop rather than the primary control.

### Failure Modes

| Failure | Impact | Recovery |
|---------|--------|----------|
| IS crashes between `BEGIN TX` and `COMMIT` (mid-CAS) | TX rolls back. **JWT is never issued** — the JWT is returned to the client only after `COMMIT` (step 5 in the diagram above). The old session remains active; no new session was created. | Client retries `Login`; a fresh CAS is attempted. No double-session window. No data loss. |
| IS crashes after `COMMIT` but before outbox relay publishes `SessionInvalidated` | JWT already returned (COMMIT = JWT issuance per diagram above). Old session is invalidated in the DB. Old connection remains open until `SessionInvalidated` is published. | Outbox relay on a surviving or restarted IS instance picks up the undelivered outbox row and publishes `SessionInvalidated`. Brief window (at most relay restart time) where the superseded connection is still alive. The 30 s gateway heartbeat is the backstop. |
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

---

## JWT Role Catalog

`identity-service` is the **single issuer** of JWTs. Every token carries exactly one of the following `role` claims, plus a `scope[]` claim for fine-grained capabilities. Authorization at downstream services is always a function of `role` and `scope`.

| Role | Issued To | Issuance Trigger | Capabilities (illustrative) | Lifetime |
|------|-----------|------------------|------------------------------|----------|
| `player` | Default for any registered, non-suspended player | `Login` succeeds for a regular account | Gameplay (`PlayCard`, `JoinRoom`, …), tournament registration (`RegisterForTournament`), leaderboard reads, spectator routes that allow auth boost | Access 1 h, refresh 14 d |
| `organizer` | Account flagged `is_organizer = true` in `player_identities` (set by admin via `POST /admin/players/{id}/grant-organizer`) | `Login` for that account | All `player` capabilities + tournament lifecycle endpoints (`CreateTournament`, `OpenRegistration`, `StartTournament`, `ForceResolveTimedOutRoom` only for tournaments they own) | Access 1 h, refresh 14 d |
| `operator` | Support staff account (separate identity store partition: `operator_identities`) | Internal SSO → `Login` against operator partition | Reads on `audit-service` operator endpoints (game log with field-level redaction), session force-invalidate for support tickets | Access 30 m, refresh 8 h (shorter lifetimes for staff) |
| `admin` | Privileged platform staff | Internal SSO + MFA challenge | All `operator` capabilities + cross-game audit trail search, integrity verification, tournament admin endpoints, organizer grants | Access 15 m, refresh 4 h, MFA on every refresh |
| `compliance` | Compliance officers (legal/regulatory) | Internal SSO + MFA + named-officer attestation | Break-glass: `POST /audit/trail/export`, unredacted hand/seed access. Every call meta-audited. | Access 15 m, no auto-refresh; re-attest each session |

**Role hierarchy:** `compliance ⊃ admin ⊃ operator`; `organizer ⊃ player`. `compliance`/`admin`/`operator` are mutually exclusive with `organizer`/`player` (staff and players cannot mix on a single JWT — separate accounts). Service-to-service mTLS bypasses JWT entirely.

**Issuance rules:**
- Player and organizer roles come from columns on the player identity row; never asserted by the client.
- Operator, admin, and compliance accounts live in a separate identity store and authenticate via internal SSO (OIDC); MFA is enforced by `identity-service` for `admin`/`compliance`.
- `identity-service` is the sole signer (RS256 with key rotation every 90 d). All downstream services validate the signature and the `role`/`scope` claims; they do not invent roles.
- The `role` claim is immutable for the lifetime of a token. Role escalation (e.g., granting `organizer`) only takes effect on the next login or refresh.

Downstream services reference this catalog: `tournament-orchestration.md` requires `organizer` for tournament lifecycle endpoints; `audit-game-log.md` requires `operator`/`admin`/`compliance` per its field-level RBAC table.
