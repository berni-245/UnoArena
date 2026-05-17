# Tournament Round Flow

This diagram shows the event-driven flow within a single tournament round, from creation to advancement.

```mermaid
sequenceDiagram
    participant Org as Organizer / System
    participant TO as Tournament Orchestration
    participant RG as Room Gameplay
    participant SV as Spectator View
    participant AL as Audit & Game Log

    Org->>TO: StartTournament
    TO-->>AL: TournamentStarted
    TO->>TO: CalculateRoundStructure
    TO-->>Org: TournamentStarted (N players, ~K rounds estimated)

    TO->>TO: CreateRound (roundNumber=1, playerIds=[...])
    TO-->>AL: TournamentRoundCreated
    TO->>TO: DistributePlayersIntoRooms (seeding algorithm)

    loop For each room (up to 100,000 simultaneous)
        TO-->>RG: TournamentRoomAssigned (roomId, playerIds, tournamentId, roundId)
        RG->>RG: CreateRoom (tournament type)
        RG-->>AL: RoomCreated
        RG->>RG: AutoJoinTournamentRoom for each player
        RG-->>AL: PlayerJoinedRoom (×N)
        RG->>RG: AutoStartTournamentRoom
        Note over RG: Plays best-of-3 match<br/>(see game-turn-sequence.md)
        RG-->>SV: SpectatorGameStarted
    end

    TO-->>AL: RoundRoomsCreated (roomCount, roomIds)

    Note over RG,TO: Rooms complete at different times (async)

    loop As each room finishes its match
        RG-->>TO: MatchCompleted (roomId, tournamentId, roundId, matchWins, cardPoints, completionTime)
        RG-->>TO: RoomCompleted
        TO->>TO: RecordMatchResult
        TO->>TO: RankPlayers (wins → cardPoints → completionTime)
        TO-->>AL: PlayerAdvanced (top 3 per room)
        TO-->>AL: PlayerEliminated (remaining players)
        TO->>TO: CheckRoundCompletion (completedRooms / totalRooms)
    end

    alt Timeout: room stuck beyond limit
        TO-->>RG: ForceResolveTimedOutRoom
        RG-->>TO: MatchCompleted (reason=timeout_forced)
    end

    TO->>TO: AllMatchesInRoundCompleted
    TO-->>AL: RoundCompleted (advancingCount, eliminatedCount)

    alt More than 10 players advancing
        TO->>TO: CreateRound (roundNumber=N+1, advancingPlayerIds)
        Note over TO,RG: Loop repeats for next round
    else 10 or fewer players advancing
        TO->>RG: TournamentRoomAssigned (Final Room, ≤10 players)
        Note over TO,RG: Final room plays; TournamentCompleted on MatchCompleted
        TO-->>AL: FinalRoomCreated
        TO-->>AL: TournamentCompleted (finalStandings)
    end
```
