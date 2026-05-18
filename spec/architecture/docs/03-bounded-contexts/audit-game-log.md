# Audit & Game Log — Bounded Context Architecture

> **Design reference:** [Spec §2.1.6](../../../domain/docs/02-bounded-contexts-and-context-map.md), [Spec §3 — Game Log Entity, Audit Trail Entity](../../../domain/docs/03-aggregates-entities-value-objects.md)

---

## Purpose and Scope

**Owns:** Immutable, append-only event log of all domain-significant events across all contexts. Game replay capability. Cryptographic integrity verification (HMAC signatures). Dispute resolution support. Compliance and forensic query API.

**Does not own:** Event production (passive consumer of all upstream events), game rules, tournament logic, player identity management.

---

## Services (Containers)

| Service | Responsibility | Scaling |
|---------|---------------|---------|
| `audit-service` | **Ingestion worker:** Subscribes to ALL Kafka topics (universal downstream conformist). Validates event envelopes, verifies HMAC signatures for integrity-critical events, appends to immutable store. **Query service:** Role-based API for game log replay, audit trail search, compliance exports. | Horizontal: consumer group partitions for ingestion. Read replicas for query load. |

---

## Public Interfaces

### Synchronous (REST — internal and admin)

| Endpoint | Method | Auth | Description |
|----------|--------|------|-------------|
| `/api/v1/audit/games/{gameId}/log` | GET | `role: operator` or `role: admin` JWT | Full game event log. Ordered by sequenceNumber. Supports pagination. |
| `/api/v1/audit/games/{gameId}/replay` | GET | `role: operator` JWT | Game replay data: initial deck state + all events in order. Sufficient for deterministic replay. |
| `/api/v1/audit/trail` | GET | `role: admin` JWT | Cross-context audit trail search. Filter by: context, aggregateId, correlationId, timestamp range, eventType. Paginated. |
| `/api/v1/audit/trail/export` | POST | `role: compliance` JWT | Compliance export: **asynchronous signed-job pattern**. The POST creates an export job (returns `{ jobId, estimatedCompletionAt }`) and immediately returns 202 Accepted. The compliance token is validated only at job creation time — the token does not need to remain valid for the duration of the export. `audit-service` executes the export server-side (no client hold required), produces a signed bundle, and stores it in a short-lived pre-signed object store URL (e.g., S3 pre-signed URL, valid 1 hour). The compliance officer polls `GET /audit/trail/export/{jobId}` (requires the same compliance JWT or a refresh; single re-attestation) to retrieve the download URL. This decouples the 15-minute token TTL from the export duration: even a multi-hour large export completes server-side without forcing the officer to hold an active session. Every job creation and download is meta-audited. |
| `/api/v1/audit/trail/export/{jobId}` | GET | `role: compliance` JWT (re-attest if expired) | Poll export job status. Returns `{ status: pending | complete | failed, downloadUrl?, expiresAt? }`. Single re-attestation sufficient; download URL valid for 1 hour. Meta-audited on download. |
| `/api/v1/audit/integrity/{gameId}` | GET | `role: admin` JWT | Verify HMAC chain integrity for a game log. Returns pass/fail with details of any broken signatures. |

**Access control:**
- All endpoints are internal (routed through `api-gateway` with admin/operator/compliance role requirements).
- No public access. Spectators and regular players cannot query audit logs.
- Every query to the audit API is itself recorded in the audit trail (meta-audit).

### Asynchronous (Kafka — consumed only)

| Topic | Events Consumed | Source Contexts |
|-------|----------------|----------------|
| `identity.sessions` | `PlayerRegistered`, `SessionEstablished`, `SessionInvalidated`, `PlayerSuspended`, `PlayerBanned` | Identity & Session |
| `gameplay.events` | All gameplay state-change events | Room Gameplay (game-engine) |
| `gameplay.games` | `GameCompleted` | Room Gameplay (game-engine) |
| `gameplay.audit` | `GameCompleted` (full payload: `finalHands[]`, `shuffleSeed`, `deckOrderingAtGameStart`, signed envelope) — audit-privileged content only. ACL-restricted: `audit-service` is the sole subscriber. This channel is the enforcement mechanism for INV-SGP-01: hand and seed data never appear on the public `gameplay.games` topic. | Room Gameplay (game-engine, via outbox relay) |
| `gameplay.rooms` | `RoomCreated`, `PlayerJoinedRoom`, `PlayerLeftRoom`, `RoomFilled`, `MatchGameStarted`, `MatchGameCompleted`, `MatchCompleted`, `RoomCompleted` | Room Gameplay (room-service) |

> **Note:** `PlayerForfeited` is produced by `game-engine` on `gameplay.events` (not `gameplay.rooms`). It is consumed by `audit-service` as part of the universal `gameplay.events` subscription above. The canonical topic for `PlayerForfeited` is `gameplay.events`; this is also reflected in `02-container-view.md` §2.2 and `04-integration-table.md` A11.
| `tournament.lifecycle` | All tournament lifecycle events | Tournament Orchestration |
| `tournament.rooms` | `TournamentRoomAssigned` | Tournament Orchestration (round-kickoff-worker) |
| `ranking.updates` | `EloUpdated`, `LeaderboardUpdated`, `PlayerStatisticsUpdated` | Ranking & Statistics |

**Produced:**

| Topic | Event | Description |
|-------|-------|-------------|
| (none by default) | `AuditEntryRecorded` | Optional: confirmation that an event was durably recorded. Used for operational monitoring. Can be published to a dedicated `audit.confirmations` topic if needed. Low priority. |

---

## Internal-Only Interfaces

| Interface | Description |
|-----------|-------------|
| HMAC verification pipeline | On ingestion of integrity-critical events (game state mutations, Elo updates, advancement decisions), verify the `signature` field using the per-room HMAC key. Signature covers `(eventId ‖ aggregateId ‖ sequenceNumber ‖ payload ‖ prevSignature)`, where `prevSignature` is the HMAC signature of the immediately preceding event (same `gameId`, `sequenceNumber - 1`), or a fixed genesis value `0x000...0` for the first event. This forms a **per-game hash chain**: any insertion or deletion of an event at position N breaks all subsequent signatures, providing tamper-evidence for the full sequence. The chain is validated by `GET /audit/integrity/{gameId}`. Log verification result. Flag chain-break failures for investigation. |
| Signature key management | HMAC keys are per-room, generated by `game-engine` at room creation, stored in a secure key store accessible to both `game-engine` and `audit-service`. Keys are rotated per room (not shared across rooms). See "HMAC Key Lifecycle" below for operational detail. |

### HMAC Key Lifecycle

| Step | Owner | Detail |
|------|-------|--------|
| Generation | `game-engine` | On `RoomCreated` (before the first game event), `game-engine` calls Vault Transit `POST /v1/transit/keys/room-{roomId}` to mint a 256-bit HMAC-SHA256 key. The Vault path encodes `roomId`; access policies bind read to the `audit-service` role and sign/verify to `game-engine`. Key never leaves Vault in cleartext when Transit is used — signing/verification is performed via `POST /v1/transit/hmac/...` and `POST /v1/transit/verify/...`. If a non-Transit deployment is used, the key is wrapped with the platform KEK and stored in the room-keys table; only the wrapped form persists outside Vault. **First-round kickoff surge:** During tournament first-round kickoff (~10k rooms/s for ~10 s = 100k `RoomCreated` events), 100k Vault Transit key-mint calls arrive in a ~10 s window. Vault Transit is designed for high-throughput key operations (benchmarked at 10k–50k ops/s per cluster node on standard hardware). To ensure availability: (1) the Vault cluster is pre-scaled to ≥3 nodes before tournament start (operational runbook step); (2) `game-engine` uses a Vault client with connection pooling and retries with exponential backoff (max 3 retries over 5 s); (3) if Vault returns 429 (rate-limit) or 503 after retries, `game-engine` falls back to a locally-generated key wrapped with the platform KEK and stored in the room-keys table, then schedules an async reconciliation job to migrate to Vault Transit when the surge subsides. Unsigned events during the fallback window are marked `signatureValid = deferred_fallback` and re-verified when Vault recovers. **Fallback key security properties:** The locally-generated fallback key uses the same algorithm (256-bit HMAC-SHA256) and entropy source (OS CSPRNG) as Vault Transit would produce. The only difference is lifecycle management: locally-generated keys are wrapped with the platform Key Encryption Key (KEK) and stored in the game-engine's local encrypted keystore. The async reconciliation job (runs within 60s) imports these keys into Vault Transit for centralized rotation policy. Security properties (entropy, algorithm) are identical; only the key-management lifecycle differs during the reconciliation window. |
| Key-creation event | `game-engine` → `audit-service` | A `RoomKeyEstablished { roomId, keyVersion, vaultPath, createdAt }` envelope is emitted on `gameplay.rooms` (audit ACL) so the audit trail records the start of the signed chain. |
| Use | `game-engine` (sign), `audit-service` (verify) | Every integrity-critical event payload is HMAC'd at write time; `audit-service` re-verifies on ingest. Verification failures emit a high-severity alert and the event is quarantined in `audit_quarantine` for human review (not silently appended). |

#### Quarantine Specification

**Schema:**
```sql
audit_quarantine (
  quarantineId    UUID            PRIMARY KEY,
  eventId         UUID            NOT NULL,
  sourceContext   TEXT            NOT NULL,
  gameId          UUID,
  reason          TEXT            NOT NULL,
  rawPayload      JSONB           NOT NULL,
  quarantinedAt   TIMESTAMPTZ     NOT NULL DEFAULT now(),
  resolvedAt      TIMESTAMPTZ     NULL,
  resolvedBy      TEXT            NULL,
  resolution      TEXT            NULL
)
```

**Access:**
- `GET /api/v1/audit/quarantine` — role: `admin` only. Returns a paginated list of quarantined events (filterable by `sourceContext`, `gameId`, `reason`, `resolved` status). Each entry includes the `rawPayload` for inspection.

**Resolution process:**
- An admin reviews the quarantined event and marks it with one of two resolutions:
  - `false_positive` — the event is re-verified (e.g., key was temporarily unavailable), confirmed valid, and re-appended to the audit trail with `signatureValid = verified_after_quarantine`.
  - `confirmed_tamper` — the event is escalated to the security team, the associated game is flagged for investigation, and the raw payload is preserved as forensic evidence. The event is **not** appended to the audit trail.
- Resolution writes `resolvedAt`, `resolvedBy` (admin identity), and `resolution` to the quarantine row. The resolution action is itself meta-audited.
| Rotation | Operations | Per-room keys are immutable for the life of the room. Cross-room rotation is implicit (each new room mints a new key). Vault Transit min-decryption-version policies prevent reuse. Master Vault keys backing Transit rotate every 90 d. |
| Unavailability during verification | `audit-service` | If Vault is unreachable: `audit-service` continues to ingest events (does not block the producer) but marks `signatureValid = unknown` until verification succeeds on retry. A background reconciliation job re-verifies pending rows when Vault returns. Operator/admin queries surface unknown-state explicitly. |
| Key destruction | Compliance | At end of retention (default 1 y active + cold archive), the Vault Transit key is marked for deletion only after archive HMAC chains are exported and counter-signed by compliance. Until then, keys are preserved to support replay. |
| Audit | Vault audit device | Every key-create, sign, verify, and policy change is logged to Vault's audit device, mirrored to the `audit_trail` table (sourceContext = `identity-session` / `infrastructure`). Tampering with Vault audit is itself meta-audited (see Meta-Audit subsection). |

This wires the T-3 / T-4 mitigations to a concrete key-store contract, makes failure modes explicit, and ensures key escrow for retention-period replay.

---

## Dependencies on Other Contexts

| Upstream Context | Dependency Type | Contract |
|-----------------|----------------|----------|
| ALL contexts | Event consumption (universal downstream) | Conformist — `audit-service` conforms to every upstream context's event schema. It never imposes its own language. |

**No downstream dependencies.** Audit is a terminal sink.

---

## Ingestion Pipeline

```
    ALL Kafka topics
         │
         ▼
┌─────────────────────────┐
│    audit-service         │
│    consumer worker       │
│                          │
│  1. Deserialize event    │
│     envelope             │
│                          │
│  2. Validate envelope:   │
│     - eventId present    │
│     - timestamp valid    │
│     - sourceContext      │
│       recognized         │
│     - correlationId      │
│       present            │
│                          │
│  3. Deduplication:       │
│     Check eventId in     │
│     processed_events     │
│     index. If seen →     │
│     ACK and skip.        │
│                          │
│  4. If signature present:│
│     Verify HMAC-SHA256   │
│     over (eventId +      │
│     aggregateId +        │
│     sequenceNumber +     │
│     payload +            │
│     prevSignature) using │
│     per-room key.        │
│     prevSignature =      │
│     sig of (seqNum-1)    │
│     event, or 0x000 for  │
│     seqNum == 0.         │
│     Log result.          │
│     Flag chain breaks.   │
│                          │
│  5. Append to audit      │
│     trail table          │
│     (immutable INSERT).  │
│                          │
│  6. If game-related      │
│     event (sourceContext │
│     == RG):              │
│     Also append to       │
│     game_log table       │
│     (partitioned by      │
│     gameId).             │
│                          │
│  7. ACK to Kafka.        │
└─────────────────────────┘
```

### Ordering

- Within a game: events are ordered by `sequenceNumber` (from the Game aggregate). The `game_log` table enforces a unique constraint on `(gameId, sequenceNumber)`.
- Cross-context: events are ordered by `timestamp` (server-authoritative). Causal ordering is tracked via `correlationId` and `causationId` for reconstruction.

**Out-of-order event handling:** Within a single game, events arrive in-order because `gameplay.events` is partitioned by `gameId` (single partition = total order). Out-of-order delivery is NOT possible for intra-game events under normal Kafka operation. The only scenario for out-of-order is consumer replay after a partition reassignment, which is handled by the deduplication guard (eventId). For HMAC chain verification, events are always validated in sequenceNumber order by reading from the `game_log` table (indexed by `(gameId, sequenceNumber)`), not from the Kafka consumer position. This means verification is always correct regardless of consumer delivery order.

---

## Persistence

**Authoritative source of record:** The `game_events` table in Room Gameplay's PostgreSQL (RG) is the single authoritative source of truth. `audit-service`'s `game_log` is an **ingested copy** (derived read model), maintained for query performance isolation and role-based access control. Audit APIs serve data from `game_log`; if it diverges from `game_events` (at-most-once delivery edge case), `game_events` is ground truth. The AL copy is the authority for field-level redaction (role-based RBAC stripping) and HMAC-chain verification — not for the underlying game state.

| Store | Technology | Data | Consistency |
|-------|-----------|------|-------------|
| Game Log (derived copy) | PostgreSQL (hash-partitioned by `gameId`) | `game_log` (gameId, sequenceNumber, eventType, payload, signature, signatureValid, timestamp, correlationId, causationId). Append-only. Ingested from `gameplay.audit` (full payload) and `gameplay.events` (game-state events). | Eventual (ingested from Kafka; authoritative source is `game_events` in RG) |
| Audit Trail | PostgreSQL (monthly range-partitioned by `timestamp`) | `audit_trail` (entryId, sourceContext, aggregateType, aggregateId, eventType, payload, timestamp, correlationId, causationId, signatureStatus). Append-only. | Eventual (at-least-once ingestion) |
| Processed Events | PostgreSQL | `processed_events` (eventId). Deduplication index. | Strong (checked before insert) |
| Signature Keys | Vault / secure key store | Per-room HMAC keys. Read-only for audit-service. | N/A (infrastructure) |

### Indexes

| Index | Purpose |
|-------|---------|
| `game_log(gameId, sequenceNumber)` | Game replay in order |
| `audit_trail(sourceContext, aggregateId)` | Per-aggregate history |
| `audit_trail(correlationId)` | Causal chain reconstruction |
| `audit_trail(timestamp)` | Time-range queries |
| `audit_trail(eventType)` | Event-type filtering |

### Immutability Enforcement

- `game_log` and `audit_trail` tables have no UPDATE or DELETE permissions for the application role. Only INSERT.
- Database-level triggers reject any UPDATE/DELETE attempt.
- For PostgreSQL: table is append-only by application convention + row-level security policy.

> **Technology commitment:** The operational store for both `audit_trail` and `game_log` is **PostgreSQL** — consistent with the rest of the system's PostgreSQL ecosystem. `audit_trail` uses monthly range-partitioned tables (by `timestamp`); `game_log` uses hash-partitioning by `gameId`. ClickHouse may be considered for future analytical workloads (e.g., cross-tournament pattern analysis, long-range compliance reporting) but the operational store is PostgreSQL.

### Retention

- **Game logs:** Retained for the configurable retention period (default: 1 year active, then archived to cold storage — S3/GCS with Parquet format). Archived logs remain queryable via the compliance export API.
- **Audit trail:** Retained for 2 years minimum (compliance requirement). Older entries archived.
- **Processed events:** Pruned after 7 days (deduplication window). **Kafka topic retention constraint:** all Kafka topics consumed by `audit-service` (`gameplay.events`, `gameplay.games`, `gameplay.rooms`, `gameplay.audit`, `identity.sessions`, `tournament.lifecycle`, `tournament.rooms`, `ranking.updates`) **must be configured with a retention period of ≤ 7 days** (time-based or size-based cap that achieves equivalent results). If a topic is replayed or the retention exceeds 7 days (e.g., a compacted topic or manual offset reset), `audit-service` will re-encounter events whose `processed_events` deduplication records have been pruned, producing duplicate `audit_trail` rows. Operators must enforce Kafka retention policy to maintain this invariant; any controlled replay beyond 7 days requires the corresponding `processed_events` records to be restored (e.g., from backup) before replay begins.

---

## Read Path for Dispute Resolution and Audit

### Use Cases

| Actor | Use Case | Endpoint | Authorization |
|-------|----------|----------|---------------|
| Support agent | Player disputes game outcome. Agent replays the full game log to verify. | `GET /audit/games/{gameId}/log` | `role: operator` |
| Anti-cheat system | Automated analysis of game logs for collusion patterns (coordinated forfeits, suspicious play patterns). | `GET /audit/games/{gameId}/replay` | Internal mTLS (service-to-service) |
| Compliance officer | Legal request for all games played by a specific player in a date range. | `POST /audit/trail/export` with `{ playerId, dateRange }` | `role: compliance` (break-glass, logged separately) |
| Platform operator | Investigate HMAC signature failure alert. | `GET /audit/integrity/{gameId}` | `role: admin` |

### Access Authorization

```
                           ┌─────────────┐
                           │ api-gateway  │
                           │ JWT + role   │
                           │ validation   │
                           └──────┬──────┘
                                  │
                    ┌─────────────┼─────────────┐
                    │             │              │
              role:operator  role:admin   role:compliance
                    │             │              │
                    ▼             ▼              ▼
              Game log       Audit trail    Compliance
              read           full search    export
              (single game)  (cross-game)   (signed bundle)
                                            + meta-audit
                                              logged
```

- **Operator:** Can read individual game logs (for support tickets), but the response is **field-level redacted**: `finalHands[]`, `shuffleSeed`, `deckOrderingAtGameStart`, and per-player private hand snapshots embedded in mid-game events are stripped or hashed before delivery. Operators see card plays, turn order, challenge resolutions, and outcomes — sufficient for typical dispute support. Cannot search across games or export. Cannot raise scope at will: privileged hand/seed access requires `role: compliance` and is meta-audited.
- **Admin:** Full audit trail access (cross-context entries, signatures, integrity checks). Hand/seed fields remain redacted for `admin` by default; admins can view game *metadata* and HMAC chain status. Cannot export (requires compliance role). Elevated hand/seed access still routes through the compliance / break-glass path.
- **Compliance:** Can export signed bundles for legal — the **only** role that may obtain unredacted `finalHands[]` / `shuffleSeed`. All compliance queries are meta-audited (logged in a separate audit stream with the compliance officer's identity, timestamp, and query parameters). See the Meta-Audit Stream subsection for storage details.
- **Automated replay jobs:** Internal service with mTLS certificate. No human JWT. Scoped to specific game IDs passed as parameters. Anti-cheat replays receive the full payload because they run server-side without exposing it to a human; output is restricted to derived signals.

### Field-Level RBAC for Game Log Read Endpoint

`GET /audit/games/{gameId}/log` applies a per-role projection before returning:

| Field | `operator` | `admin` | `compliance` | Internal mTLS replay |
|-------|-----------|---------|--------------|----------------------|
| Public outcome (placements, points, turns, plays) | ✅ | ✅ | ✅ | ✅ |
| Per-player private hand snapshots (mid-game / final) | ❌ (redacted) | ❌ (redacted) | ✅ | ✅ |
| `shuffleSeed`, `deckOrderingAtGameStart` | ❌ (redacted) | ❌ (redacted) | ✅ | ✅ |
| HMAC signature / `signatureValid` | ✅ | ✅ | ✅ | ✅ |
| Player PII beyond `playerId` | ❌ | ❌ | ✅ (scoped) | ❌ |

Redaction is applied at the query layer before serialization — the storage row remains immutable. Tightens T-6 in `12-threat-model.md` (seed disclosure) to also cover hand-content disclosure to the operator tier.

### Meta-Audit Stream

Every query to the audit API — especially break-glass compliance exports — is itself logged in a **separate meta-audit table**, isolated from `audit_trail` to prevent compliance officers from reading or redacting their own query history.

| Attribute | Detail |
|-----------|--------|
| **Storage table** | `compliance_meta_audit` — a separate append-only PostgreSQL table in a dedicated schema, owned by the `audit_meta_writer` DB role. The `audit-service` application role has INSERT only; no SELECT, UPDATE, or DELETE. |
| **Who can read** | Only `role: admin` (via a dedicated `GET /audit/meta-audit` endpoint, not accessible to `compliance`). This ensures a compliance officer cannot suppress evidence of their own break-glass access. |
| **Written by** | `audit-service` writes one row synchronously before returning the compliance query response. If the meta-audit INSERT fails, the compliance query is rejected (fail-closed). |
| **Schema** | `entryId UUID, officerId UUID, role TEXT, endpoint TEXT, queryParams JSONB, resultRowCount INT, requestTimestamp TIMESTAMPTZ, responseTimestamp TIMESTAMPTZ, ipAddress INET, correlationId UUID` |
| **Retention** | Minimum 5 years (exceeds the 2-year `audit_trail` retention) to satisfy regulatory requirements for access audit logs. |
| **Immutability** | Same enforcement as `audit_trail`: INSERT-only DB permissions + database-level trigger rejecting UPDATE/DELETE. |

This resolves the R-2 threat-model mitigation gap: compliance cannot export or redact the record of their own queries because the meta-audit table is in a separate schema inaccessible to the `compliance` role. The `admin` role can review compliance access activity independently.
