# Elo Update Pipeline

Shows the cross-context flow from a completed casual game to updated player ratings.

```mermaid
sequenceDiagram
    participant RG as Room Gameplay
    participant RK as Ranking & Statistics
    participant AL as Audit & Game Log
    participant RM as Read Models

    RG-->>RK: GameCompleted { gameId, roomType, placements,\n  cardPointTotals, wasAbandoned, completedAt }
    RG-->>AL: GameCompleted (full payload, signed)

    RK->>RK: 1. Idempotency check: gameId already processed?
    alt Already processed (duplicate delivery)
        RK->>RK: DuplicateEventIgnored — skip all processing
    else Not yet processed
        RK->>RK: 2. Filter: roomType == "casual"?
        alt Tournament room
            RK->>RK: NonCasualGameIgnored — skip Elo update
            Note over RK: Statistics still updated (games played, etc.)
        else Casual room
            RK->>RK: 3. Filter: wasAbandoned == true?
            alt Abandoned game
                RK->>RK: AbandonedGameSkipped — no Elo change
            else Normal completion
                RK->>RK: 4. Fetch current Elo ratings for all players
                RK->>RK: 5. Determine K-factor per player\n  (< 30 games: K=40, 30-100: K=20, 100+: K=10)
                RK->>RK: 6. Compute pairwise expected scores\n  E(i,j) = 1 / (1 + 10^((Rj-Ri)/400))
                RK->>RK: 7. Assign actual scores from placement order\n  1st beats all: actual=1; last loses all: actual=0
                RK->>RK: 8. Compute delta per player\n  Δi = Ki × Σj(actual_ij - expected_ij)
                RK->>RK: 9. Clamp: new_rating = max(100, old_rating + delta)
                RK->>RK: 10. Atomic persist (serialized per playerId)\n    + mark gameId as processed
                RK-->>AL: EloRatingUpdated (per player: playerId, gameId, old, new, delta)
                RK-->>RM: PlayerProfileReadModel updated
                RK-->>RM: LeaderboardReadModel updated
                RK-->>RM: RatingHistoryReadModel appended
            end
        end
    end
```

## K-Factor Schedule (Assumption A24/A25)

| Games Played | K-Factor | Effect |
|---|---|---|
| < 30 (new player) | 40 | Rating moves quickly to find true skill level |
| 30–100 (intermediate) | 20 | Moderate adjustment speed |
| > 100 (established) | 10 | Stable rating; small adjustments |

## Multi-Player Elo Formula

For N players, each player i plays N-1 "virtual" pairwise matches against each opponent j:

```
delta_i = K_i × Σ_{j≠i} (actual_score(i,j) − expected_score(i,j))

where:
  actual_score(i,j) = 1   if player i placed higher than j
                    = 0.5 if tied (should not occur in Uno)
                    = 0   if player i placed lower than j

  expected_score(i,j) = 1 / (1 + 10^((R_j − R_i) / 400))
```

Rating floor: `new_rating = max(100, new_rating)`
