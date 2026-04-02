# Disconnection Handling Flow

This diagram shows all paths from disconnection through reconnection or forfeit.

```mermaid
stateDiagram-v2
    [*] --> Connected : PlayerJoinedRoom / GameStarted

    Connected --> Disconnected : Heartbeat lost (detected within 5-10s)

    state Disconnected {
        [*] --> ReconnectionWindowOpen : PlayerDisconnected + ReconnectionTimerStarted (60s)

        ReconnectionWindowOpen --> ReconnectionWindowOpen : TurnSkippedDueToDisconnection\n(auto-skip each time it is their turn)

        ReconnectionWindowOpen --> Reconnected_Valid : Reconnect [valid session, window open]
        ReconnectionWindowOpen --> Reconnected_Invalid : Reconnect [session invalidated]
        ReconnectionWindowOpen --> WindowExpired_OffTurn : ReconnectionTimerExpired [not player's turn]
        ReconnectionWindowOpen --> WindowExpired_OnTurn : ReconnectionTimerExpired [player's turn]
    }

    Reconnected_Valid --> Connected : PlayerReconnected\nReconnectionTimerCancelled\n(resumes with original hand)
    Reconnected_Invalid --> ReconnectionWindowOpen : CommandRejected (SessionInvalidated)\n[window still ticking]

    WindowExpired_OffTurn --> WaitingForNextTurn : Player is now Inactive\nNo longer auto-skipped proactively
    WaitingForNextTurn --> Forfeited : Next turn arrives → PlayerForfeited

    WindowExpired_OnTurn --> Forfeited : PlayerForfeited (immediately,\nwindow expired during their turn)

    Forfeited --> [*] : casual room → game continues\ntournament room → PlayerEliminated
```

## Disconnection Scenarios at a Glance

| When Disconnected | Window Expires On Turn? | Outcome |
|-------------------|------------------------|---------|
| During my turn | Yes (always) | Immediate PlayerForfeited on window expiry |
| Between turns | Yes (when rotation returns) | PlayerForfeited on next turn arrival |
| Between turns | No (reconnects before turn) | PlayerReconnected, game resumes |
| Session was invalidated | N/A | Reconnect always fails; window runs out; PlayerForfeited |

## Session Invalidation → Forced Forfeit Chain

```mermaid
sequenceDiagram
    participant IS as Identity & Session
    participant RG as Room Gameplay
    participant TO as Tournament Orchestration

    IS-->>RG: SessionInvalidated (playerId, reason=new_login)
    RG->>RG: Find active game for playerId
    RG-->>RG: PlayerDisconnected (internal)
    RG-->>RG: ReconnectionTimerStarted (60s)

    Note over RG: During 60s: any Reconnect attempt<br/>fails (session invalid → 401 Unauthorized)

    loop Each time player's turn arrives
        RG-->>RG: TurnSkippedDueToDisconnection
    end

    RG-->>RG: ReconnectionTimerExpired
    RG-->>TO: PlayerForfeited (reason=reconnection_timeout)
    TO->>TO: [if tournament room] PlayerEliminated
```
