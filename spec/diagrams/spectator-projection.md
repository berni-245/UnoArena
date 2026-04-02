# Spectator Projection (Anti-Corruption Layer)

Shows how Room Gameplay events are transformed through the ACL before reaching the Spectator View projection.

```mermaid
flowchart LR
    subgraph RG ["Room Gameplay (Upstream)"]
        E1["GameStarted\n{players, hands, deck, seed,\n turnOrder, firstPlayer}"]
        E2["CardPlayed\n{playerId, card, newDiscardTop,\n handSize}"]
        E3["CardDrawn\n{playerId, card ← PRIVATE,\n newHandSize}"]
        E4["TurnAdvanced\n{activePlayerId, direction}"]
        E5["UnoCallMade\n{playerId}"]
        E6["ChallengeMade\n{challengerId, targetId}"]
        E7["ChallengeResolved\n{outcome, penalizedPlayerId,\n penaltyCards ← PRIVATE}"]
        E8["PlayerDisconnected\n{playerId}"]
        E9["GameCompleted\n{placements, hands ← PRIVATE}"]
    end

    subgraph ACL ["Spectator View ACL (Privacy Filter)"]
        T1["Strip: hand contents,\n deck seed, draw pile\n Retain: card counts (×7 each),\n discard top, turnOrder, direction"]
        T2["Pass through as-is\n (played card is public)"]
        T3["Strip: card identity\n Retain: playerId, newHandSize only"]
        T4["Pass through as-is"]
        T5["Pass through as-is"]
        T6["Pass through as-is"]
        T7["Strip: penaltyCards identity\n Retain: outcome, who penalized,\n new card count delta"]
        T8["Pass through as-is"]
        T9["Strip: all hand contents\n Retain: final placement order only"]
    end

    subgraph SV ["Spectator View (Downstream)"]
        P1["SpectatorGameStarted\n{playerNames, cardCounts={p:7},\n discardTop, firstPlayer, direction}"]
        P2["SpectatorCardPlayed\n{playerId, card, newDiscardTop,\n playerCardCount}"]
        P3["SpectatorCardDrawn\n{playerId, newCardCount}\n ← card identity NEVER revealed"]
        P4["SpectatorTurnAdvanced\n{activePlayerId, direction}"]
        P5["SpectatorUnoCallMade\n{playerId}"]
        P6["SpectatorChallengeMade\n{challengerId, targetId}"]
        P7["SpectatorChallengeResolved\n{outcome, penalizedPlayerId,\n newCardCount}"]
        P8["SpectatorPlayerDisconnected\n{playerId}"]
        P9["SpectatorGameCompleted\n{placements}"]
    end

    E1 --> T1 --> P1
    E2 --> T2 --> P2
    E3 --> T3 --> P3
    E4 --> T4 --> P4
    E5 --> T5 --> P5
    E6 --> T6 --> P6
    E7 --> T7 --> P7
    E8 --> T8 --> P8
    E9 --> T9 --> P9
```

## Data Classification: Public vs. Private

| Data Element | Classification | Rationale |
|---|---|---|
| Player display names | Public | Needed for spectator context |
| Card count per player (integer) | Public | Shows game progress without revealing strategy |
| Discard pile top card | Public | Defines legal play; spectators can see it |
| Turn indicator (whose turn) | Public | Core gameplay visibility |
| Play direction (CW / CCW) | Public | Observable game state |
| Uno call events | Public | Observable action |
| Challenge events and outcomes | Public | Observable action and result |
| Match score (games won) | Public | Tournament and match context |
| Player connection status | Public | Observable game state |
| **Hand contents (card identities)** | **PRIVATE** | Core rule: no spectator sees another player's hand |
| **Drawn card identity** | **PRIVATE** | Revealing drawn card = revealing part of hand |
| **Draw pile contents/order** | **PRIVATE** | Would allow prediction of future draws |
| **Deck seed / RNG state** | **PRIVATE** | Would allow full hand reconstruction |
| **Penalty card identities** | **PRIVATE** | Added to hand → private |
| **Sequence numbers** | **Private** (implementation) | Internal concurrency control; not spectator-relevant |

## ACL Ownership

The ACL is owned by the **Spectator View context**, not by Room Gameplay. Room Gameplay publishes its full, unfiltered event stream. The Spectator View's ACL is responsible for all privacy enforcement. Room Gameplay has no knowledge of or responsibility for spectator concerns.
