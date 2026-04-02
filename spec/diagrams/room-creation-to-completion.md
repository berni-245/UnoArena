# Room Creation to Completion

End-to-end sequence for a casual room through the full best-of-3 match lifecycle.

```mermaid
sequenceDiagram
    participant Host as Host Player
    participant P2 as Other Players
    participant RG as Room Gameplay
    participant RK as Ranking & Statistics
    participant SV as Spectator View
    participant AL as Audit & Game Log

    %% === ROOM SETUP ===
    Host->>RG: CreateRoom {roomType=casual, maxPlayers, matchFormat=best_of_3}
    RG-->>AL: RoomCreated
    RG-->>Host: RoomCreated {roomId, status=waiting}

    P2->>RG: JoinRoom {roomId} (repeated per player)
    RG-->>AL: PlayerJoinedRoom (×N)
    RG-->>SV: [room visible in lobby]

    Host->>RG: StartGame {roomId}
    RG->>RG: Validate: ≥2 players, status=waiting, issuer=host

    %% === GAME 1 ===
    RG->>RG: InitializeDeck (server-side RNG, seed recorded)
    RG-->>AL: DeckInitialized {shuffleSeed}
    RG->>RG: DealInitialHands (7 cards each)
    RG-->>AL: InitialHandsDealt (private payload)
    RG->>RG: FlipFirstDiscardCard [handle special first-card rules]
    RG-->>AL: GameStarted {gameId=G1, turnOrder, firstPlayer}
    RG-->>SV: SpectatorGameStarted {playerNames, cardCounts={p:7}, discardTop}

    loop Active Gameplay (repeated per turn)
        RG-->>RG: TurnAdvanced {activePlayerId}
        Note over RG: Player acts: PlayCard / DrawCard / PassTurn
        RG-->>AL: CardPlayed / CardDrawn / TurnPassed
        RG-->>SV: SpectatorCardPlayed / SpectatorCardDrawn (card identity stripped)
        opt Player plays penultimate card
            RG-->>AL: ChallengeWindowOpened (5s timer)
            alt UnoCallMade in time
                RG-->>AL: UnoCalledSuccessfully
            else ChallengeUnoCall received
                RG-->>AL: ChallengeResolved {penalizedPlayerId}
            else Window expires
                RG-->>AL: ChallengeWindowClosed (reason=expired)
            end
        end
    end

    RG-->>AL: GameCompleted {G1, placements, cardPoints, wasAbandoned=false}
    RG-->>SV: SpectatorGameCompleted

    RG->>RG: CheckMatchProgress: wins so far?

    %% === GAME 2 (if needed) ===
    alt Match not yet decided (no player has 2 wins)
        RG-->>AL: MatchGameCompleted {matchNumber, gameNumber=1, nextGame=2}
        RG->>RG: InitializeDeck (new seed), DealInitialHands
        RG-->>AL: GameStarted {gameId=G2}
        Note over RG: Game 2 follows same loop as Game 1
        RG-->>AL: GameCompleted {G2, ...}
    end

    %% === GAME 3 (if needed) ===
    alt Match still not decided after Game 2
        RG-->>AL: MatchGameCompleted {gameNumber=2, nextGame=3}
        Note over RG: Game 3 follows same loop
        RG-->>AL: GameCompleted {G3, ...}
    end

    %% === MATCH & ROOM COMPLETION ===
    RG->>RG: MatchCompleted: rank by wins → cardPoints → completionTime
    RG-->>AL: MatchCompleted {matchWinnerId, matchPlacements, matchWins, cumulativeCardPoints}
    RG-->>AL: RoomCompleted {roomType=casual, finalStandings}

    %% === DOWNSTREAM ===
    RG-->>RK: GameCompleted (×1 to 3, one per game)
    Note over RK: Compute Elo delta per game (not per match)
    RK-->>AL: EloRatingUpdated (per player, per game)

    RG-->>SV: [Room removed from active listings]
```

## State Machine Summary

```
Room: Waiting → In-Progress → Completed
Match: [G1 started] → [G1 done] → [G2 started?] → [G2 done] → [G3 started?] → [G3 done] → MatchCompleted
Game:  AwaitingAction → [CardPlayed/Drawn] → [ChallengeWindowOpen?] → AwaitingAction → ... → GameCompleted
```
