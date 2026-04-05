# Concurrent Action Resolution

Shows how the Game aggregate resolves simultaneous conflicting commands using sequence numbers and atomic processing.

## Scenario 1: Two Players Simultaneously Submit PlayCard

```mermaid
sequenceDiagram
    participant PA as Player A (active turn)
    participant PB as Player B (waiting)
    participant GA as Game Aggregate
    participant AL as Audit & Game Log

    Note over GA: state: activePlayerId=A, expectedSeq=42

    PA->>GA: PlayCard {playerId=A, cardId=C1, seqNum=42}
    PB->>GA: PlayCard {playerId=B, cardId=C2, seqNum=39}

    Note over GA: Commands arrive; processed ONE AT A TIME (serialized)

    GA->>GA: Process A's command first (arrived first)
    GA->>GA: Check: playerId==activePlayerId? ✅ A==A
    GA->>GA: Check: seqNum==expectedSeq? ✅ 42==42
    GA->>GA: Check: cardId in A's hand? ✅
    GA->>GA: Check: legal play? ✅
    GA->>GA: Apply: remove card from A's hand, update discard, increment seq to 43
    GA-->>AL: CardPlayed {playerId=A, card=C1, seqNum=42}

    GA->>GA: Process B's command
    GA->>GA: Check: playerId==activePlayerId? ❌ B≠A
    GA-->>PB: CommandRejected {reason=NotYourTurn, seqNum=42}

    Note over PB: PB receives CardPlayed via SSE, updates local state
```

## Scenario 2: Concurrent Uno Call and Challenge

```mermaid
sequenceDiagram
    participant PT as Target Player
    participant PC as Challenger
    participant GA as Game Aggregate
    participant AL as Audit & Game Log

    Note over GA: ChallengeWindowOpen, handSize(PT)=1\n windowDeadline = T+5s

    PT->>GA: CallUno {playerId=PT} [arrives at T+1.2s]
    PC->>GA: ChallengeUnoCall {challengerId=PC, targetId=PT} [arrives at T+1.3s]

    GA->>GA: Process CallUno (arrived first)
    GA->>GA: Set: unoCalled[PT] = true
    GA-->>AL: UnoCallMade {playerId=PT}
    GA-->>AL: ChallengeWindowClosed {reason=uno_called}

    GA->>GA: Process ChallengeUnoCall
    GA->>GA: Check: challengeWindowOpen? ❌ (already closed by UnoCallMade)
    GA-->>PC: CommandRejected {reason=ChallengeWindowClosed}

    Note over PC: Race condition — no penalty assessed; PC challenged in good faith within their perceived window
```

## Scenario 3: Two Simultaneous Challenges

```mermaid
sequenceDiagram
    participant PC1 as Challenger 1
    participant PC2 as Challenger 2
    participant GA as Game Aggregate
    participant AL as Audit & Game Log

    Note over GA: ChallengeWindowOpen, unoCalled=false

    PC1->>GA: ChallengeUnoCall {challengerId=PC1, targetId=PT}
    PC2->>GA: ChallengeUnoCall {challengerId=PC2, targetId=PT}

    GA->>GA: Process PC1's challenge (arrived first)
    GA->>GA: Check: windowOpen? ✅
    GA->>GA: Check: unoCalled? ❌ (not called → challenge succeeds)
    GA-->>AL: ChallengeResolved {outcome=target_penalized, challengerId=PC1}
    GA->>GA: Apply: PT draws 2 penalty cards
    GA-->>AL: PenaltyCardsDrawn {playerId=PT, cardCount=2}
    GA->>GA: Close challenge window
    GA-->>AL: ChallengeWindowClosed

    GA->>GA: Process PC2's challenge
    GA->>GA: Check: windowOpen? ❌ (already closed)
    GA-->>PC2: CommandRejected {reason=ChallengeWindowClosed}
```

## Scenario 4: Stale Sequence Number Under Concurrency

```mermaid
sequenceDiagram
    participant P1 as Player 1 (active)
    participant P2 as Player 2
    participant GA as Game Aggregate
    participant AL as Audit & Game Log

    Note over GA: expectedSeq=100

    Note over P1,P2: Both clients last saw seq=99.<br/>P1 is active player. P2 is not.

    P1->>GA: PlayCard {seqNum=100, ...} ✅
    GA-->>AL: CardPlayed {seqNum=100}
    Note over GA: expectedSeq now = 101

    Note over P2: P2 missed the CardPlayed event (SSE lag)
    Note over P2: P2's turn just arrived (TurnAdvanced)

    P2->>GA: PlayCard {seqNum=100, ...}
    GA->>GA: Check seqNum: 100 ≠ 101 (expected)
    GA-->>P2: CommandRejected {reason=StaleSequenceNumber, HTTP 409}
    Note over P2: P2 receives CardPlayed via SSE,\nrecalculates correct seqNum=101,\nretries with seqNum=101
```

## Invariants Maintained Across All Scenarios

| Invariant | Enforcement Mechanism |
|---|---|
| Exactly one command processed at a time per Game | Serialized command processing (aggregate is single-writer) |
| Only the active player may play/draw | `activePlayerId` check before any state mutation |
| Sequence number advances monotonically | Reject any command with seqNum ≠ expectedSeq |
| At most one challenge resolution per window | `challengeWindowStatus` state checked before processing |
| CardPlayed appended to log before broadcast | Write-ahead to Game Log is the commit; broadcast is secondary |
| No command is applied twice | Idempotency key (`commandId`) checked before processing |
