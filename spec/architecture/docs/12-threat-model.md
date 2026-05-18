# 12. Threat Model (Lightweight)

> STRIDE-based threat analysis for UnoArena's public APIs, session lifecycle, event pipeline, and rate-limit surface. Each threat is mapped to the specific service or layer responsible for mitigation and to the relevant domain invariants where applicable.

---

## 12.1 Scope

This model covers:
- Public API surface (`api-gateway`, `identity-service`, player WebSocket/SSE endpoints)
- Session lifecycle (token issuance, validation, invalidation)
- Real-time command path (WebSocket → `game-engine`)
- Event pipeline (Kafka topics, outbox, consumer trust boundaries)
- Spectator privacy boundary (ACL in `spectator-projection-service`)
- Tournament integrity (room assignment, advancement)
- Audit log integrity (HMAC chain)

Out of scope: cloud infrastructure misconfigurations, physical security, supply-chain attacks on dependencies.

---

## 12.2 Assets and Trust Boundaries

| Asset | Owner | Sensitivity |
|-------|-------|-------------|
| Player credentials (hashed) | `identity-service` / PostgreSQL | High — credential database |
| Session tokens (JWTs) | `api-gateway` / Redis | High — bearer tokens; compromise = impersonation |
| Player hand contents | `game-engine` / PostgreSQL (event store) | High — private game state; never exposed to spectators |
| Deck shuffle seed | `game-engine` / PostgreSQL (game log) | High — reveals future draws |
| Elo ratings | `ranking-service` / PostgreSQL | Medium — manipulation distorts competitive ranking |
| Tournament bracket state | `tournament-service` / PostgreSQL | Medium — manipulation affects advancement |
| Game log (immutable) | `audit-service` / append-only store | High — evidence for dispute resolution |
| Kafka event stream | Internal only (no public exposure) | Medium — tampering could corrupt downstream read models |

**Trust boundaries:**

```
[Internet] ──── HTTPS/WSS ────► [api-gateway]  ← public trust boundary
                                      │
                        JWT-validated internal calls
                                      │
                 ┌────────────────────┼──────────────────────┐
                 ▼                    ▼                        ▼
          [identity-service]   [game-engine]        [spectator-projection]
          [room-service]        [timer-service]      [audit-service]
          [tournament-service]  [ranking-service]
                 │
           mTLS / internal network
                 │
            [Kafka] [PostgreSQL] [Redis]
```

Services within the internal network trust each other's identity via mTLS. No service is reachable from the internet except through `api-gateway`.

---

## 12.3 STRIDE Threat Table

### S — Spoofing

| ID | Threat | Attack Vector | Mitigation | Owner |
|----|--------|--------------|------------|-------|
| S-1 | **Session token theft** — Attacker steals a valid JWT and issues game commands as the victim | XSS (client-side), network interception, token logged in plaintext | Transport: all connections over TLS (WSS/HTTPS). Tokens not stored in localStorage (httpOnly cookie or in-memory only). Tokens are short-lived (24h, A29). `SessionInvalidated` immediately invalidates stolen token on new login. | `identity-service`, `api-gateway` |
| S-2 | **Session token replay after invalidation** — Attacker replays a captured token after the victim logs out or logs in elsewhere | Captured token used within TTL | `api-gateway` validates every token against Redis cache. `SessionInvalidated` event triggers cache eviction within ~200 ms. Tokens after invalidation → 401 on next validation | `api-gateway`, `identity-service` |
| S-3 | **Impersonation via forged JWT** | Attacker crafts a JWT with a fabricated `playerId` | JWTs signed with **RS256** (RSA-SHA256) — the sole algorithm. `identity-service` is the only signer, holding the private key; all downstream services verify using the public key. Key rotation every 90 days (see `identity-session.md` §JWT Role Catalog). HMAC-SHA256 is explicitly excluded: it would require sharing the signing secret with every downstream service, making any one compromised service a universal token forger. Gateway verifies RS256 signature before accepting any claim. Unsigned or tampered tokens are rejected at the gateway before reaching any backend. | `api-gateway` |
| S-4 | **Internal service impersonation** | Attacker on internal network sends messages as a trusted service | mTLS between all internal services; certificate pinning. Kafka ACLs restrict which service principals can produce/consume which topics. | Network / service mesh |

---

### T — Tampering

| ID | Threat | Attack Vector | Mitigation | Owner |
|----|--------|--------------|------------|-------|
| T-1 | **Game command injection** — Attacker replays a previously captured `PlayCard` command to play a card twice | Replay of a valid WebSocket message | Idempotency key (`commandId`) checked before processing; cached result returned on duplicate. Sequence number enforcement (INV-G-05): replayed commands carry a stale sequence number and are rejected with HTTP 409. | `game-engine` |
| T-2 | **Sequence number manipulation** — Attacker submits commands with artificially high sequence numbers to desynchronize the game | Modified client or MITM (requires HTTPS bypass) | Sequence number is validated server-side; the server's expected value is authoritative. Client-supplied sequence numbers outside the expected window are rejected (INV-G-05, INV-R-12). TLS prevents MITM. | `game-engine` |
| T-3 | **Game log tampering** — Attacker attempts to modify committed game log entries (e.g., to dispute an outcome) | Direct DB access, insider threat | Game log is append-only at the DB layer (no UPDATE/DELETE permissions for application role; row-level security policy). Each entry is HMAC-signed (A31) using a per-room secret stored in a secure key store. `audit-service` verifies signature on receipt; tamper flag set if verification fails. | `audit-service`, PostgreSQL RLS |
| T-4 | **Outbox/event stream poisoning** — Attacker injects fabricated events into Kafka | Requires Kafka producer access | Kafka ACLs: only authorized service principals (identified by mTLS certificate) may produce to each topic. `audit-service` verifies HMAC signatures on game state events; unsigned or mismatched events are flagged. | Kafka ACLs, `audit-service` |
| T-5 | **WDF hand snapshot manipulation** — Attacker attempts to alter the hand snapshot used for WDF challenge adjudication | Requires `game-engine` code execution | Hand snapshot is captured server-side at the moment of WDF play and never transmitted to any client. Only the boolean adjudication result is published (`WildDrawFourChallengeResolved`). Snapshot is stored only in the signed game log entry (T-3 mitigations apply). | `game-engine`, `audit-service` |
| T-6 | **Shuffle seed extraction** — Attacker reads the shuffle seed to predict future draws | DB read access or game log exfiltration | Shuffle seed is stored only in the audit/game log, which requires `role: operator` or `role: compliance` to access. No API exposes the seed to players or spectators. ACL in `spectator-projection-service` strips seed from all projected events (INV-SGP-01). | `audit-service` (access control), `spectator-projection-service` (ACL) |

---

### R — Repudiation

| ID | Threat | Attack Vector | Mitigation | Owner |
|----|--------|--------------|------------|-------|
| R-1 | **Player disputes game outcome** — Player claims a card play was not theirs or was processed incorrectly | No technical attack; social/policy dispute | Every command creates a game log entry signed with the per-room HMAC (A31). The full sequence of events is deterministically replayable from the initial shuffle seed. Audit replay via `GET /audit/games/{gameId}/replay` provides irrefutable evidence. | `audit-service`, `game-engine` |
| R-2 | **Operator disputes audit query** — Compliance officer claims they did not perform a sensitive audit export | Insider accountability | All `role: compliance` queries are meta-audited in a separate audit stream (who queried, when, which game/player, query parameters). This record is itself immutable. | `audit-service` |
| R-3 | **Tournament advancement dispute** — Player claims they were incorrectly eliminated | Policy dispute | All tournament advancement decisions (`PlayerAdvanced`, `PlayerEliminated`) are published to `tournament.lifecycle` and recorded in the audit trail with full payload and HMAC signature. The `AdvancementResult` value object (doc 03 §3.2.4) captures match wins, card points, and completion times used for tiebreaking. | `audit-service`, `tournament-service` |

---

### I — Information Disclosure

| ID | Threat | Attack Vector | Mitigation | Owner |
|----|--------|--------------|------------|-------|
| I-1 | **Hand data leak to spectators** — Spectator learns a player's hand contents | Unfiltered event delivery, ACL bypass, or subscription channel crossover | ACL in `spectator-projection-service` strips card identities on `CardDrawn`, `ChallengeResolved`, and `WildDrawFourChallengeResolved` before materialization. Separate storage: spectator DB contains only spectator-safe fields — no raw RG events stored (INV-SGP-01). Separate streaming channel: spectators use SSE (public route); players use WebSocket (authenticated route). Gateway enforces route separation. | `spectator-projection-service` (ACL), `api-gateway` (route separation) |
| I-2 | **Active player subscribes to own room's spectator feed** — Player attempts to gain additional information via the spectator channel | Second browser tab or anonymous session | ACL is unconditional: identity of subscriber is irrelevant. Spectator projection is a strict subset of what the active player already observes through the gameplay channel. No information gain (doc 06 §6.6.4). | `spectator-projection-service` (ACL) |
| I-3 | **Deck seed exposure via audit API** | Attacker queries the audit API as a player | Audit API requires `role: operator` minimum (JWT role claim). No public endpoint exposes game log private data. Player-facing APIs never include seed or hand contents. | `api-gateway` (role enforcement), `audit-service` |
| I-4 | **Session token exposure via logs or error responses** | Token accidentally logged or returned in error body | Session tokens never appear in structured logs (log sanitization). Error responses use opaque reason codes, not token values. Redis stores token hash, not plaintext (optional defense-in-depth). | `api-gateway`, `identity-service` |
| I-5 | **Event payload eavesdropping on Kafka** | Attacker on internal network reads Kafka topic | Kafka topic encryption at rest. mTLS for broker connections. Consumer ACLs ensure only authorized services can subscribe to each topic (e.g., `spectator-projection-service` can consume `gameplay.events` but not `tournament.kickoff-work`). | Kafka configuration |
| I-6 | **PII in immutable game log — data-subject erasure right (GDPR/LATAM)** — A player exercises the right to erasure; `playerId` (a pseudonymous but potentially linkable identifier) is stored in append-only `game_log` and `audit_trail` tables that cannot be deleted | Regulatory / compliance obligation; not an external attack | **Pseudonymisation on erasure request:** upon a verified right-to-erasure request, `audit-service` executes a targeted UPDATE of all rows referencing that `playerId` across `game_log`, `audit_trail`, and `compliance_meta_audit`, replacing the identifier with a one-way derived token (`erased_player_{sha256(playerId + erasure_salt)}`). The salt is discarded after the operation, making re-identification computationally infeasible. The mapping `playerId → token` is deleted from the `player_identities` table. Game log HMAC chains remain structurally valid (the token replaces the identifier in place; the signature fields are not recomputed, and the chain's continuity is preserved — the signature mismatch is logged as `signatureStatus = pseudonymised`). Audit trail rows required for ongoing legal proceedings are subject to GDPR Article 17(3)(b) (legal obligation carve-out) and are retained pseudonymised for the minimum compliance period (2 years). Erasure requests are processed within 30 days per GDPR Article 12(3). The erasure action is itself recorded in `compliance_meta_audit`. | `audit-service`, `identity-service` |

---

### D — Denial of Service

| ID | Threat | Attack Vector | Mitigation | Owner |
|----|--------|--------------|------------|-------|
| D-1 | **Command flooding** — Attacker sends commands at extremely high rate to overwhelm `game-engine` | High-frequency WebSocket messages | Per-player rate limit: 10 commands/s per player per room (A30, V20). Enforced at `api-gateway` before commands reach `game-engine`. Adaptive throttling for repeat offenders via `identity-service` directives (doc 07 §7.6). | `api-gateway`, `identity-service` |
| D-2 | **Connection exhaustion** — Attacker opens millions of WebSocket/SSE connections | Bot network | Per-IP connection limit enforced at `api-gateway` (ingress rate limiting). Unauthenticated SSE connections (spectators) are rate-limited per IP. WebSocket connections require a valid JWT after handshake (auth within N seconds or close). | `api-gateway`, edge/CDN |
| D-3 | **Sequence number DoS** — Attacker repeatedly sends commands with wrong sequence numbers to trigger 409 storms, wasting game-engine processing | Malicious client | Sequence-number mismatch is a fast-path rejection (in-memory check before DB write). Rate limit at `api-gateway` throttles the sustained 409 rate per player. Domain signal: `CommandRejected(StaleSequenceNumber)` spike triggers alert (doc 07 §7.7). | `game-engine`, `api-gateway` |
| D-4 | **Registration/login flooding** — Attacker bulk-creates accounts or hammers the login endpoint | Automation | Per-IP rate limit on `/api/v1/players` and `/api/v1/sessions`. CAPTCHA or email verification for registration (operational policy; not domain-modeled). `identity-service` returns 429 after threshold. | `api-gateway` (per-IP), `identity-service` (per-operation) |
| D-5 | **Kafka consumer lag attack** — Attacker publishes extremely large event payloads to slow Kafka consumers | Requires Kafka producer access (T-4 mitigations apply) | Kafka message size limit enforced at broker level (e.g., 1 MB max). Schema validation on ingestion rejects malformed events before processing. `audit-service` flags `MalformedEventDiscarded`. | Kafka configuration, `audit-service` |
| D-6 | **Tournament round blocking** — Attacker or malfunctioning client stalls a game indefinitely to block tournament progression | Player refuses to act | Turn timer (A38): 30 s casual, 60 s tournament. Server issues `AutoDraw`/`AutoPass` on expiry (`TurnTimedOut`). Round timeout escalation (doc 07 §7.5.3): `ForceResolveTimedOutRoom` after 2 hours. No single room can block the round indefinitely. | `timer-service`, `tournament-service` |

---

### E — Elevation of Privilege

| ID | Threat | Attack Vector | Mitigation | Owner |
|----|--------|--------------|------------|-------|
| E-1 | **Non-host starts match** — Player other than the room host issues `StartMatch` | Modified client or replayed command | `StartMatch` precondition: `command.playerId == room.hostPlayerId` (INV-R-03). Enforced by `room-service` aggregate before any state change. JWT `playerId` claim is server-assigned and tamper-proof (S-3). | `room-service` |
| E-2 | **Out-of-turn card play** — Player issues `PlayCard` when it is not their turn | Modified client | Turn enforcement: `command.playerId == game.currentPlayerId` (INV-G-01). Enforced by `game-engine` aggregate before any state change. Rejected with `CommandRejected(NotCurrentPlayer)`. | `game-engine` |
| E-3 | **Player self-joins tournament room** — Player issues `JoinRoom` targeting a tournament room | Modified client | Tournament rooms are created exclusively by `room-service` in response to `TournamentRoomAssigned`. `JoinRoom` is rejected for rooms with `roomType == tournament` (doc 07 §7.4.1). Player slots are assigned at creation; no self-join path exists. | `room-service` |
| E-4 | **Spectator subscribing to player command channel** — Spectator attempts to issue game commands via the WebSocket endpoint | Authentication bypass attempt | WebSocket game command routes require a valid JWT with an active session. Anonymous (spectator) connections are SSE-only; no WebSocket upgrade is permitted on public routes. `api-gateway` enforces route separation before any backend is reached. | `api-gateway` |
| E-5 | **Rate-limit bypass via IP rotation** | Attacker uses a pool of IP addresses to evade per-IP limits | Per-user rate limits (applied after JWT validation, keyed by `playerId`) cannot be bypassed by IP rotation. Per-room limits further constrain burst regardless of source IP. Adaptive throttling (V20) escalates per-player limits on sustained abuse. | `api-gateway` (per-user layer), `identity-service` (adaptive throttle state) |
| E-6 | **Privilege escalation via role claim forgery** — Attacker forges a JWT with `role: admin` | Forged JWT | JWTs are signed with RS256 (see S-3). `api-gateway` verifies the RS256 signature before trusting any claim. Admin/operator/compliance roles are set at issuance time by `identity-service` and cannot be modified client-side. | `api-gateway`, `identity-service` |

---

## 12.4 Risk Summary

| ID | Severity | Likelihood | Residual Risk | Notes |
|----|----------|-----------|---------------|-------|
| S-1 (session theft) | High | Low | **Low** | TLS + short TTL + push-invalidation |
| S-2 (token replay) | High | Low | **Low** | Redis invalidation within 200 ms |
| T-1 (command replay) | Medium | Medium | **Low** | Idempotency key + sequence number |
| T-3 (game log tamper) | High | Low | **Low** | Append-only + HMAC chain |
| I-1 (hand data leak) | High | Low | **Low** | ACL unconditional; separate storage and channel |
| D-1 (command flood) | Medium | Medium | **Low (degraded to Medium during Redis outage)** | Per-player rate limit at gateway. Adaptive throttle (banned/suspended player block) fails open on Redis unavailability — during a Redis outage, Layer 1/2 limits remain but `block`-tier directives for banned players are skipped. Treat Redis cluster unavailability as a security event. |
| D-6 (tournament block) | Medium | Low | **Low** | Turn timer + forced resolution |
| E-2 (out-of-turn play) | Medium | Medium | **Low** | Server-enforced invariant |
| E-5 (rate-limit bypass) | Medium | Medium | **Low** | Per-user layer independent of IP |
| T-6 (seed extraction) | High | Low | **Medium** | Requires operator-level DB or API access; insider threat risk remains |

**Highest residual risk:** T-6 (shuffle seed extraction by insider with operator/compliance credentials). Mitigated by meta-auditing of all compliance queries and break-glass access controls, but insider threat is not fully eliminable.
