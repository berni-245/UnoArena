# Sequence Diagram — Log-Before-Broadcast (PlayCard Hot Path)

> **Mandatory intra-context diagram** (§6.1). Shows the Room Gameplay hot path for a `PlayCard` command end-to-end, demonstrating that every authoritative state change is durably appended to the immutable game log **before** any broadcast to players, spectators, or downstream consumers.

---

## Diagram

```mermaid
sequenceDiagram
    autonumber
    participant P as Player<br/>(WebSocket)
    participant GW as api-gateway
    participant GE as game-engine
    participant DB as PostgreSQL<br/>(event store + outbox)
    participant OR as Outbox Relay<br/>Worker
    participant K as Kafka<br/>(gameplay.events)
    participant SP as spectator-projection<br/>-service
    participant AU as audit-service

    P->>GW: play_card { cardId: "R7", seq: 42, commandId: "cmd-abc" }
    GW->>GW: Validate JWT ✓<br/>Extract playerId, sessionId
    GW->>GE: RelayCommand(playerId, play_card, seq=42, commandId="cmd-abc")

    Note over GE: Load Game aggregate
    GE->>DB: SELECT FROM game_events WHERE gameId = $1 ORDER BY seq
    DB-->>GE: Events (or use in-memory cache)

    Note over GE: Validate command
    GE->>GE: Check commandId in idempotency ledger → not seen ✓
    GE->>GE: Check seq == expectedSequenceNumber (42) ✓
    GE->>GE: Check player's turn ✓
    GE->>GE: Check card play is legal ✓

    rect rgb(220, 245, 220)
        Note over GE,DB: ═══ BEGIN TRANSACTION ═══
        GE->>DB: INSERT INTO game_events<br/>(gameId, seq=42, type='CardPlayed',<br/>payload, signature)
        GE->>DB: INSERT INTO outbox<br/>(eventId, topic='gameplay.events',<br/>partitionKey=gameId, payload, delivered=false)
        GE->>DB: INSERT INTO command_idempotency<br/>(commandId='cmd-abc', gameId, response)
        Note over GE,DB: ═══ COMMIT ═══
    end

    Note over GE,DB: ✅ LOG IS NOW DURABLE<br/>Crash-safe from this point

    GE-->>GW: ACK { seq=42, events: [CardPlayed] }
    GW-->>P: game_state_update { events: [CardPlayed], aggregateSeq: 42 }

    Note over OR: Polls outbox every 50ms
    OR->>DB: SELECT FROM outbox<br/>WHERE delivered = false<br/>ORDER BY created_at LIMIT 100
    DB-->>OR: [CardPlayed event row]
    OR->>K: Produce CardPlayed<br/>topic: gameplay.events<br/>key: gameId
    K-->>OR: ACK (partition committed)
    OR->>DB: UPDATE outbox<br/>SET delivered = true<br/>WHERE eventId = $1

    par Spectator projection
        K->>SP: CardPlayed (consumer group: spectator)
        SP->>SP: ACL transform:<br/>CardPlayed → SpectatorCardPlayed<br/>(retain: playerId, card, remainingCards)<br/>(strip: nothing for played cards)
        SP->>SP: Update SpectatorGameProjection
        SP->>SP: Push to SSE subscribers
    and Audit trail
        K->>AU: CardPlayed (consumer group: audit)
        AU->>AU: Verify HMAC-SHA256 signature
        AU->>AU: Dedup by eventId
        AU->>AU: Append to game_log + audit_trail
    end
```

---

## Crash Recovery Analysis

| Crash Point | Log State | Broadcast State | Client Sees | Recovery |
|-------------|-----------|-----------------|-------------|----------|
| **Before COMMIT** (steps 8–10) | ❌ Not persisted | ❌ Not broadcast | Nothing (no ACK) | TX rolls back. Client retries with same `commandId` → idempotent processing. |
| **After COMMIT, before ACK** (step 11) | ✅ Persisted | ❌ Not broadcast yet | No ACK → client may retry | Client retries with same `commandId` → idempotency ledger returns cached response. Outbox relay publishes event on next poll. |
| **After ACK, before Kafka publish** (step 14) | ✅ Persisted | ❌ Not yet in Kafka | Player sees ACK ✓ | Outbox relay picks up on next poll (50 ms) or on restart. Event reaches Kafka. At-least-once. |
| **After Kafka publish, before outbox mark** (step 16) | ✅ Persisted | ✅ In Kafka | Player sees ACK ✓ | Outbox relay retries → duplicate publish. Consumers deduplicate by `eventId`. |
| **Consumer crashes** (step 17+) | ✅ Persisted | ✅ In Kafka | Player sees ACK ✓ | Kafka retains event. Consumer redelivers on restart. Consumer deduplicates by `eventId`. |

### Key Guarantee

**No client ever sees an update that isn't in the log.** The WebSocket ACK (step 11) happens strictly **after** COMMIT (step 10). If the process crashes between steps 8 and 10, the TX rolls back — the event never existed.

### Why Transactional Outbox (Not Direct Kafka Produce)

Direct Kafka produce cannot be made atomic with the PostgreSQL event store write. Two failure modes:
1. **DB commit succeeds, Kafka fails:** Event is logged but never broadcast (consumers miss it).
2. **Kafka succeeds, DB commit fails:** Event is broadcast but never logged (violates log-before-broadcast).

The transactional outbox eliminates both: the outbox row and event store row are in the **same TX**. The relay worker handles Kafka publication asynchronously with at-least-once semantics.
