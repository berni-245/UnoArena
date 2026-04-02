# 6. Edge Cases and Failure-Path Analysis

> For each case, we define expected domain behavior, emitted events, and invariants that must hold throughout. Where a scenario is partially covered by doc 05 event flows, this document references that flow (e.g., "see Flow 1, Exception E1.1") and contributes deeper domain analysis rather than repeating the narrative.

---

## 6.1 Concurrent Conflicting Actions

Concurrency conflicts arise whenever two or more commands target the same aggregate state before any of them has been serialized by the aggregate's command handler. The Game aggregate is the primary contention point; all cases below apply unless noted otherwise.

---

### 6.1.1 Two Players Simultaneously Playing a Card

**Description.** Player A and Player B both issue a `PlayCard` command for the same game at nearly the same instant. On each player's client, the local state (built from SSE event replay) shows it is their turn — possible when, for example, a `TurnAdvanced` event propagates to client A before reaching client B, or when both clients suffered a momentary lag spike at the same time and their local states diverged.

**Root cause.** Network asymmetry causes two clients to hold locally-valid-but-mutually-exclusive beliefs about whose turn it is. Neither client acted incorrectly given what it knew.

**Detection mechanism.** The Game aggregate maintains two invariants checked atomically on every `PlayCard` command:
1. `command.sequenceNumber == aggregate.expectedSequenceNumber` — staleness guard.
2. `command.playerId == aggregate.activePlayerId` — turn-ownership guard.

Both conditions must be true for the command to be accepted. The aggregate processes exactly one command at a time (single writer, per-game serialized command handler). Whichever `PlayCard` arrives first in the serialized queue satisfies both conditions and is accepted. The second arrives after the first has advanced `activePlayerId` and `expectedSequenceNumber`; it therefore fails both checks simultaneously.

**Domain behavior.** Exactly one `PlayCard` succeeds. The succeeding command is processed in full: card is removed from the player's hand, applied to the discard pile, and all downstream policies fire. The failing command is rejected before any state mutation occurs.

**Events emitted.**
- Winning command: `CardPlayed { gameId, playerId, card, newHandSize, sequenceNumber }`
- Losing command: `CommandRejected { gameId, playerId, commandId, reason: NotYourTurn, sequenceNumber }`

**Client reconciliation.** The losing client receives the `CardPlayed` event (and the subsequent `TurnAdvanced`) via its open SSE stream. It updates its local state, learns the true `activePlayerId` and expected `sequenceNumber`, and may issue a new command if applicable on the next turn.

**Invariants that must hold.**
- INV-G1: Only one command is processed atomically per Game aggregate per turn.
- INV-G2: The aggregate's `sequenceNumber` advances by exactly one per accepted command.
- INV-G3: `activePlayerId` changes only as a result of a successfully processed turn action.

---

### 6.1.2 Player Plays a Card While the Uno Challenge Window Is Open

**Description.** After a player plays their penultimate card (reducing hand to 1), the Uno challenge window opens. Two sub-cases arise:

**(a) The next player in turn order immediately submits a `PlayCard` command** before issuing a `ChallengeUnoCall` or waiting for the window to expire.

**(b) A player who is NOT the next player in turn order submits a `PlayCard` command** while the challenge window is open.

**Root cause (a).** The next player either did not notice the challenge window was open (UI lag) or deliberately chose to play rather than challenge, thereby implicitly waiving the challenge right.

**Root cause (b).** Client-side state divergence; client believes it is their turn.

**Detection mechanism.** The Game aggregate tracks `challengeWindowOpen: boolean` and `challengeWindowTarget: playerId`. On every incoming `PlayCard`, the aggregate checks:
1. Is `challengeWindowOpen == true`?
   - If yes: is `command.playerId == nextPlayerId`?
     - If yes (sub-case a): the challenge window is atomically closed first, then the `PlayCard` is processed normally.
     - If no (sub-case b): the command is rejected (`reason: NotYourTurn`) without touching the challenge window.
2. If `challengeWindowOpen == false`: normal turn-ownership and sequence-number checks apply.

**Domain behavior.**

*Sub-case (a):* The challenge window closes because the next player acted (closing condition (c) per domain rules). The next player's `PlayCard` is then processed as a normal card play. The aggregate emits `ChallengeWindowClosed` before `CardPlayed` — both emitted in the same atomic aggregate transaction. No penalty is assessed to either player for the Uno call.

*Sub-case (b):* The command is rejected outright. The challenge window remains open. Its expiry timer continues.

**Events emitted (sub-case a).**
- `ChallengeWindowClosed { gameId, reason: next_player_acted, targetPlayerId, challengeWindowId }`
- `CardPlayed { ... }`
- `TurnAdvanced { ... }`

**Events emitted (sub-case b).**
- `CommandRejected { reason: NotYourTurn }`

**Invariants that must hold.**
- INV-CW1: The challenge window is atomically closed before any subsequent action is processed. No event from a subsequent turn action may precede `ChallengeWindowClosed` in the game log.
- INV-CW2: A challenge window closed by reason `next_player_acted` results in no penalty for the Uno-holding player (they are safe; the challenge was waived).
- INV-G1 (from 6.1.1) still applies.

---

### 6.1.3 Concurrent Uno Call and Challenge

**Description.** The target player submits `CallUno` (declaring their own Uno) at the same instant an opponent submits `ChallengeUnoCall`. Both commands arrive within the 5-second challenge window.

**Root cause.** The challenge window is open on all clients simultaneously; the target may call Uno reactively while an opponent submits a challenge in the same network tick.

**Detection mechanism.** The Game aggregate serializes all commands. Whichever of `CallUno` or `ChallengeUnoCall` arrives first in the serialized command queue is processed first. The second command arrives after the first has already closed or resolved the challenge window.

**Domain behavior.**

*If `CallUno` arrives first:*
The target successfully calls Uno. The challenge window closes with reason `uno_called`. The subsequent `ChallengeUnoCall` arrives after the window is closed; it is rejected with `reason: ChallengeWindowClosed`. The challenger is penalized (two draw cards) for submitting a late challenge — no, wait: the challenger submitted within the window but lost the race. The domain rule (A7) is server-authoritative: since `CallUno` was processed first, the challenge window is already closed when `ChallengeUnoCall` is evaluated. No penalty is assessed to either player beyond the rejection of the challenge command. The challenger issued the command in good faith within what their client believed was the window; this is an expected race condition, not abuse.

*If `ChallengeUnoCall` arrives first:*
The challenge is processed. The aggregate checks: did the target call Uno before this challenge? No — `CallUno` has not been processed. Challenge succeeds. Target receives penalty cards (`UnoChallengeAccepted`, `PenaltyCardsDrawn`). Challenge window closes. The subsequent `CallUno` arrives after the window is closed; rejected with `reason: ChallengeWindowClosed`. No Uno call credit is given.

**Events emitted (`CallUno` first).**
- `UnoCalledSuccessfully { gameId, playerId, challengeWindowId }`
- `ChallengeWindowClosed { reason: uno_called }`
- `CommandRejected { commandId: challengeCommand.commandId, reason: ChallengeWindowClosed }`

**Events emitted (`ChallengeUnoCall` first).**
- `UnoChallengeAccepted { gameId, challengerId, targetPlayerId, challengeWindowId }`
- `PenaltyCardsDrawn { playerId: targetPlayerId, count: 2, reason: failed_uno_call }`
- `ChallengeWindowClosed { reason: challenge_resolved }`
- `CommandRejected { commandId: callUnoCommand.commandId, reason: ChallengeWindowClosed }`

**Invariants that must hold.**
- INV-CW3: Exactly one resolution is emitted per challenge window instance. The `challengeWindowId` is unique and matched to exactly one `ChallengeWindowClosed` event.
- INV-CW4: The resolution outcome (penalty or safe) is determined solely by the server's processing order — client-perceived timing is irrelevant.

---

### 6.1.4 Multiple Simultaneous Challenges

**Description.** Two or more opponents submit `ChallengeUnoCall` within the challenge window, believing they are each the first to challenge.

**Root cause.** All eligible challengers see an open window on their clients and act simultaneously.

**Detection mechanism.** The aggregate holds `challengeWindowOpen: boolean`. Upon processing the first `ChallengeUnoCall`, the aggregate sets `challengeWindowOpen = false` and emits `ChallengeWindowClosed`. Any subsequent `ChallengeUnoCall` commands are evaluated against `challengeWindowOpen == false` and rejected.

**Domain behavior.** Only the first challenge received (by arrival order in the serialized command queue) is processed and resolved. All subsequent challenges are rejected with `reason: ChallengeWindowClosed`. No additional penalties are assessed for the redundant challenges.

**Events emitted.**
- `UnoChallengeAccepted` or `UnoChallengeRejected` (for the first challenger, depending on whether the target had already called Uno)
- `PenaltyCardsDrawn` (for whichever party loses the first challenge)
- `ChallengeWindowClosed { reason: challenge_resolved }`
- `CommandRejected { reason: ChallengeWindowClosed }` (for each subsequent challenger)

**Invariants that must hold.**
- INV-CW3 (from 6.1.3): at most one challenge resolution per challenge window.
- INV-CW5: A player who submits a challenge that is rejected due to the window already being closed is not penalized.

---

### 6.1.5 Player Submits a Command During Reconnection Window (Before Completing Handshake)

**Description.** A player's WebSocket connection drops. Their reconnection window (60 seconds) begins. Before completing a `Reconnect` handshake, the player's client retains buffered state and attempts to send a `PlayCard` or other game command.

**Root cause.** Client-side retry logic or buffered outgoing messages may attempt to deliver commands after a connection drop, before the reconnection protocol is completed.

**Detection mechanism.** The Game aggregate tracks each player's `connectionStatus`. A player in status `disconnected` (or `reconnecting`) is not eligible to issue game commands. The command handler checks `player.connectionStatus == connected` before processing any action command.

**Domain behavior.** The command is rejected without state mutation. The reconnection window continues unaffected. Auto-skip logic continues to govern turn handling during the window.

**Events emitted.**
- `CommandRejected { gameId, playerId, commandId, reason: PlayerDisconnected }`

**Invariants that must hold.**
- INV-D1: A player in `disconnected` status cannot affect game state until a successful `Reconnect` handshake changes their status to `connected`.
- INV-G1: No game state mutation occurs from a rejected command.

---

### 6.1.6 Race Between Reconnection Command and Auto-Forfeit Timer

**Description.** A player's 60-second reconnection window is about to expire. At the boundary instant, two things happen nearly simultaneously: the auto-forfeit timer fires (triggering an internal `ExpireReconnectionWindow` command) and the player sends a `Reconnect` command.

**Root cause.** Distributed timer imprecision. The timer fires at `t=60.000s`; the `Reconnect` command arrives at `t=59.998s` but is processed in the queue after the timer event due to ordering in the command queue, or vice versa.

**Detection mechanism.** The aggregate serializes all commands, including timer-sourced internal commands. The canonical ordering is determined by which command enters the aggregate's command queue first. The aggregate's `reconnectionWindowExpiry` timestamp is the authoritative deadline; the aggregate checks `now() < reconnectionWindowExpiry` atomically when processing `Reconnect`.

**Domain behavior.**

*If `Reconnect` is processed first (arrives before `ExpireReconnectionWindow` in the queue):*
The aggregate evaluates `now() < reconnectionWindowExpiry`. If true: `PlayerReconnected` is emitted. The window is closed normally. The subsequent `ExpireReconnectionWindow` command (if it still arrives) is evaluated against `reconnectionWindowExpiry`; since the player is now `connected`, the expiry command is a no-op.

*If `ExpireReconnectionWindow` is processed first:*
`ReconnectionTimerExpired` is emitted. If it is currently the disconnected player's turn, `PlayerForfeited` is emitted immediately. The player's hand is removed. Any subsequent `Reconnect` command is rejected with `reason: ReconnectionWindowExpired`.

**Events emitted (`Reconnect` first).**
- `PlayerReconnected { gameId, playerId, handRestored: true }`
- (Subsequent `ExpireReconnectionWindow` is silently discarded — no-op)

**Events emitted (`ExpireReconnectionWindow` first).**
- `ReconnectionTimerExpired { gameId, playerId }`
- `PlayerForfeited { gameId, playerId, reason: reconnection_window_expired }` (only if currently their turn)
- `HandRemoved { gameId, playerId }` (cards removed, not reshuffled)
- `CommandRejected { reason: ReconnectionWindowExpired }` (for subsequent `Reconnect`)

**Invariants that must hold.**
- INV-D2: Once `PlayerForfeited` is emitted, no `Reconnect` command is accepted for that player in that game. The forfeit is irreversible.
- INV-D3: `PlayerReconnected` and `PlayerForfeited` are mutually exclusive for the same player in the same game. The aggregate must not emit both.
- INV-D4: The aggregate's internal clock (`reconnectionWindowExpiry`) is the sole authority for window expiry determination — no client-provided timestamp is consulted.

---

### 6.1.7 Stale Sequence Number Under High Concurrency (Multi-Player Room)

**Description.** In a room with up to 10 players, multiple players' clients have slightly out-of-sync local state (e.g., during a burst of rapid play). Several players issue commands within the same processing window, each using a `sequenceNumber` that was valid when they composed the command.

**Root cause.** In a 10-player room with fast play, a client that last received state at sequence N may compose a command with `sequenceNumber = N+1`. If three players do this simultaneously, only one can be right; the aggregate has already moved to N+2 or N+3 by the time the others' commands arrive.

**Detection mechanism.** The aggregate's `expectedSequenceNumber` is compared against each incoming command's `sequenceNumber`. Commands with a non-matching sequence number are rejected with HTTP 409 Conflict.

**Domain behavior.** Only the command matching the current `expectedSequenceNumber` succeeds. All others are rejected with `reason: StaleSequenceNumber` or `reason: UnexpectedSequenceNumber` (as applicable per 6.3.1 and 6.3.2). Clients receive the accepted event via SSE, update their local sequence number, and may retry if their action is still valid on the new state.

**Events emitted.**
- One `[ActionEvent]` (e.g., `CardPlayed`) for the single accepted command.
- One `CommandRejected { reason: StaleSequenceNumber }` per rejected command.

**Invariants that must hold.**
- INV-G2: The aggregate's sequence number advances by exactly one per accepted command, never more.
- INV-G4: Rejected commands leave the aggregate's sequence number unchanged.

---

## 6.2 Disconnections and Late Rejoin Attempts

The reconnection window opens the moment the server detects that a player's heartbeat has been lost. See Flow 1, Exception E1.1 for the base narrative. The analysis below adds domain-level precision on timing, ordering, and the forfeit condition.

---

### 6.2.1 Disconnection During Active Turn

**Description.** A player's connection drops at the exact moment it is their turn to act. The heartbeat is lost; the server detects the disconnection.

**Detection mechanism.** The session manager (IS context) detects heartbeat failure and emits `PlayerDisconnected`. The Game aggregate receives this (via a policy) and transitions the player's `connectionStatus` to `disconnected`, starting the reconnection timer.

**Domain behavior.** The player is now disconnected on their own turn. Auto-skip logic fires: after a brief period (the turn timer elapses without action), the aggregate emits `TurnSkippedDueToDisconnection` and advances to the next player. This cycle repeats every time the disconnected player's turn comes around during the reconnection window. If the reconnection window expires while it is still (or again) the disconnected player's turn: `PlayerForfeited` is emitted.

The precise forfeit trigger (per V18): `PlayerForfeited` is issued if and only if the reconnection window expires during the player's turn. "During their turn" means the aggregate's `activePlayerId == disconnectedPlayerId` at the moment the `ExpireReconnectionWindow` command is processed.

**Events emitted (full sequence).**
- `PlayerDisconnected { gameId, playerId, timestamp }`
- `ReconnectionTimerStarted { gameId, playerId, expiresAt }`
- `TurnSkippedDueToDisconnection { gameId, playerId }` (repeated on each turn cycle during window)
- `ReconnectionTimerExpired { gameId, playerId }` (if player does not reconnect)
- `PlayerForfeited { gameId, playerId, reason: reconnection_window_expired }` (if window expires on their turn)
- `HandRemoved { gameId, playerId }`

**Invariants that must hold.**
- INV-D5: Auto-forfeit is triggered only when `reconnectionWindowExpiry` is reached AND `activePlayerId == disconnectedPlayerId`. If the window expires between turns, see 6.2.2.
- INV-D6: Cards of a forfeited player are removed entirely from the game — they are not reshuffled into the draw pile.
- INV-D7: No bot substitution occurs. The disconnected player's slot is held throughout the reconnection window.

---

### 6.2.2 Disconnection NOT During Active Turn

**Description.** A player disconnects while it is another player's turn. The game continues; the disconnected player's turns are auto-skipped as they come around. The reconnection window expires, but it expires in a moment between turns — specifically when `activePlayerId != disconnectedPlayerId`.

**Root cause.** The reconnection timer is a wall-clock timer independent of turn order. It fires at a fixed point in time, which may not coincide with the disconnected player's turn.

**Domain behavior nuance.** V18 states: "If the window expires during the player's turn, an automatic forfeit is issued." This is the literal condition. If the window expires between turns, the player is transitioned to `inactive` status. On the very next time the `activePlayerId` would advance to this player's position, the aggregate detects `connectionStatus == inactive` (window already expired) and immediately emits `PlayerForfeited` at that moment — the forfeit is deferred to their next turn.

This creates a two-phase sequence: first `ReconnectionTimerExpired` (between turns), then `PlayerForfeited` (on their next turn).

**Events emitted (full sequence).**
- `PlayerDisconnected { gameId, playerId }`
- `ReconnectionTimerStarted { gameId, playerId, expiresAt }`
- `TurnSkippedDueToDisconnection { gameId, playerId }` (×N, during window)
- `ReconnectionTimerExpired { gameId, playerId }` (fires between turns — no forfeit yet)
- `TurnSkippedDueToDisconnection { gameId, playerId }` (possibly one more before forfeit triggers, depending on implementation choice; see note)
- `PlayerForfeited { gameId, playerId, reason: reconnection_window_expired }` (emitted when their turn arrives post-expiry)
- `HandRemoved { gameId, playerId }`

> **Implementation note.** The aggregate may choose to emit `PlayerForfeited` immediately on `ReconnectionTimerExpired` regardless of turn position, for simplicity. The distinction (deferred-to-next-turn vs. immediate) is a domain decision that must be recorded as an assumption. The invariant is: `PlayerForfeited` is emitted at most once per player per game.

**Invariants that must hold.**
- INV-D5 (from 6.2.1): The condition for forfeit is `reconnectionWindowExpiry` reached, regardless of whether it happens mid-turn or between turns. The exact emission moment is determined by the chosen implementation policy (immediate vs. deferred), but must be consistent.
- INV-D8: `PlayerForfeited` is emitted exactly once per player per game, regardless of how many turns are skipped.

---

### 6.2.3 Reconnection with Invalid Session (New Login on Another Device)

**Description.** Player P is mid-game and their connection drops (reconnection window opens). While disconnected, they (or an attacker) logs in on a new device. The new login triggers `SessionInvalidated` for the old session (single-active-session invariant, A29). Now the reconnection window is still open, but the session token that would be used for `Reconnect` is invalid.

**Root cause.** The single-active-session invariant means any new successful authentication immediately revokes all prior sessions for that identity.

**Detection mechanism.** IS context emits `SessionInvalidated { playerId, oldSessionId, reason: new_login }`. The Game aggregate (or a policy in IS) receives this and marks the player's `sessionStatus` as `invalid` for that game slot, even while the reconnection window remains open. Any subsequent `Reconnect` command carrying the invalidated session token is rejected at the authentication layer before reaching the aggregate.

**Domain behavior.** The reconnection window continues to count down, but reconnection is now impossible because no valid session token exists for that player's old game slot. The window will expire. At expiry (or on next turn, per 6.2.2's nuance), `PlayerForfeited` is emitted.

If the player logs in on the new device and tries to reconnect using the new session: the new session is valid, but it has no association to the in-progress game room (the room membership was tied to the old session). Reconnect via new session is rejected with `reason: SessionNotAssociatedWithRoom` — the game aggregate does not allow session substitution mid-game.

**Events emitted.**
- `SessionInvalidated { playerId, oldSessionId, reason: new_login }` (IS context)
- `PlayerDisconnected { gameId, playerId }` (if not already emitted)
- `ReconnectionTimerExpired { gameId, playerId }` (when window elapses)
- `PlayerForfeited { gameId, playerId, reason: reconnection_window_expired }`
- `HandRemoved { gameId, playerId }`
- `CommandRejected { reason: InvalidSession }` (for any `Reconnect` attempt with old token)
- `CommandRejected { reason: SessionNotAssociatedWithRoom }` (for any `Reconnect` attempt with new token)

**Invariants that must hold.**
- INV-S1: An invalidated session token is rejected on every command, including `Reconnect`, regardless of whether the reconnection window is still open.
- INV-S2: Session substitution mid-game is not permitted. A player's game slot is bound to the session that joined the room.
- INV-D2 (from 6.1.6): Once `PlayerForfeited` is emitted, no reconnection is accepted.

---

### 6.2.4 Reconnect Command Arrives After Window Expiry

**Description.** A player sends a `Reconnect` command after their 60-second reconnection window has expired.

**Detection mechanism.** The aggregate checks: `player.reconnectionWindowExpiry != null AND now() > player.reconnectionWindowExpiry`. This check fails before the `Reconnect` is processed.

**Domain behavior.** The `Reconnect` is rejected. `PlayerForfeited` was already emitted when the window expired (or on the player's next turn, per 6.2.2). The game state is unaffected. See Flow 1, Exception E1.1 for base coverage.

**Events emitted.**
- `CommandRejected { gameId, playerId, commandId, reason: ReconnectionWindowExpired }`

**Invariants that must hold.**
- INV-D2: Forfeit is irreversible. No reconnection is accepted post-expiry.

---

### 6.2.5 Player Reconnects but the Game Has Already Ended

**Description.** While a player is in their reconnection window (window still open), the game ends — for example, because all other players forfeited (last-player-standing win condition, see Flow 1, Exceptions E1.4/E1.5) or because another player went out normally.

**Detection mechanism.** The Game aggregate transitions to `status: completed`. Any incoming commands targeting a completed game are evaluated against this status.

**Domain behavior.** The player's `Reconnect` command is received while the game is in `completed` status. The command is rejected or acknowledged as informational: "Game already ended — no reconnection required." No game state is restored. The disconnected player's result (win by last-standing, loss, etc.) was already recorded in the `GameCompleted` event.

If the disconnected player won by last-standing: their result is preserved regardless of whether they reconnect. Their final hand state at disconnect time is the record.

**Events emitted.**
- `CommandRejected { gameId, playerId, commandId, reason: GameAlreadyCompleted }` (or a soft acknowledgement with game result payload — domain decision)

**Invariants that must hold.**
- INV-G5: A `completed` game aggregate accepts no further state-mutating commands.
- INV-R1: Game result (winner, final hand sizes) is determined at the moment `GameCompleted` is emitted, not at reconnection time.

---

### 6.2.6 Disconnection of the Last Two Players Simultaneously

**Description.** Only two players remain in a game. Both disconnect at nearly the same instant (e.g., a shared network segment fails).

**Root cause.** Shared network infrastructure failure or coordinated disconnect.

**Detection mechanism.** IS context detects heartbeat failures for both players, emitting `PlayerDisconnected` for each. The Game aggregate receives both events. After processing both, the aggregate checks: `connectedPlayerCount == 0`.

**Domain behavior.** Both reconnection windows start. The game enters a fully-paused state (both players' turns auto-skipped, with no actions possible). Three possible outcomes:

1. **Both reconnect within their respective windows:** Game resumes normally from its paused state. `PlayerReconnected` for each. Turn order restores to the active player.

2. **One reconnects, the other does not:** The reconnected player continues. When the non-reconnected player's window expires, `PlayerForfeited` is triggered on their next turn. The reconnected player wins by last-player-standing. `GameCompleted { outcome: last_player_standing }`. `MatchCompleted` follows. If this was a casual game: Elo is updated for the winner (per Elo rules — the game was not abandoned).

3. **Neither reconnects:** Both windows expire. Both players forfeit. `GameAbandoned { reason: all_players_forfeited }`. No Elo updates. `MatchAbandoned`. `RoomCompleted { outcome: abandoned }`.

**Events emitted (scenario 3).**
- `PlayerDisconnected` × 2
- `ReconnectionTimerStarted` × 2
- `ReconnectionTimerExpired` × 2
- `PlayerForfeited` × 2
- `HandRemoved` × 2
- `GameAbandoned { gameId, reason: all_players_forfeited }`
- `MatchAbandoned { roomId, matchId }`
- `RoomCompleted { roomId, outcome: abandoned }`

**Invariants that must hold.**
- INV-A1: A game where all remaining players forfeit is classified as `abandoned`. No Elo changes are applied.
- INV-A2: `GameAbandoned` and `GameCompleted` are mutually exclusive for a given `gameId`.
- INV-D6: Forfeited players' cards are removed, not reshuffled, even in an abandoned game.

---

### 6.2.7 Between-Rounds Disconnection in a Tournament

**Description.** A player has advanced from Round N and is in the waiting state between rounds (while the system prepares Round N+1 rooms). The player's session drops during this waiting period.

**Root cause.** Network instability during the inter-round lobby phase.

**Detection mechanism.** IS context detects heartbeat failure. Since the player is not yet in an active game room (they are between rounds), the standard in-game reconnection window does not apply. Instead, a between-round reconnection policy applies.

**Domain behavior.** When TO emits `TournamentRoomAssigned` for Round N+1, the RG context attempts `AutoJoinTournamentRoom` for the player. If the player is disconnected at this point, a between-round reconnection window is started (60 seconds from room-ready time). During this window, the player's slot is reserved in the new room. If the player reconnects and completes the join before the window expires: they enter the room normally. If the window expires: `PlayerForfeited { gameId: newGameId, reason: failed_to_join_tournament_room }` is emitted for the new game. `PlayerEliminated { tournamentId, playerId, round: N+1, reason: failed_to_join }` is emitted by TO. The room proceeds with one fewer player (if the room still meets the minimum player count) or is restructured by TO.

This window is distinct from the in-game reconnection window: it starts at room-ready time (not at heartbeat loss time) and applies to the joining phase rather than an active game.

**Events emitted.**
- `PlayerDisconnected { playerId }` (IS context — between rounds)
- `TournamentRoomAssigned { tournamentId, roundNumber, roomId, playerId }` (TO context)
- `AutoJoinTournamentRoom` (command, issued by policy)
- `BetweenRoundReconnectionWindowStarted { roomId, playerId, expiresAt }` (if player is disconnected at room-ready time)
- `BetweenRoundReconnectionWindowExpired { roomId, playerId }` (if no reconnect)
- `PlayerForfeited { gameId, playerId, reason: failed_to_join_tournament_room }`
- `PlayerEliminated { tournamentId, playerId, round: N+1 }`

**Invariants that must hold.**
- INV-T1: A disconnected tournament player's slot is reserved during the between-round window. The room is not filled with another player's slot during this period.
- INV-T2: `PlayerEliminated` from failure to join is treated identically to elimination by match loss for ranking and bracket purposes.

---

## 6.3 Stale Commands and Replayed Commands

---

### 6.3.1 Stale Sequence Number (Client Behind)

**Description.** A client issues a command with a `sequenceNumber` lower than the aggregate's current `expectedSequenceNumber`. The client's local state is behind (it has not yet received and applied one or more SSE events).

**Detection mechanism.** `command.sequenceNumber < aggregate.expectedSequenceNumber`. The aggregate detects this before any state mutation.

**Domain behavior.** The command is rejected with HTTP 409 Conflict. No state mutation occurs. The client, upon receiving the rejection, re-processes any missed SSE events (its stream delivers all events in order). Once it has the latest state, it may recompose and reissue the command (with updated `sequenceNumber`) if the action is still valid. See Flow 1, Exception E1.2 for base coverage.

**Events emitted.**
- `CommandRejected { gameId, playerId, commandId, reason: StaleSequenceNumber, expectedSequenceNumber: N, receivedSequenceNumber: M }`

**Invariants that must hold.**
- INV-G2: The aggregate's sequence number does not advance on rejection.
- INV-G4: No state mutation from a rejected command.

---

### 6.3.2 Future Sequence Number (Client Ahead)

**Description.** A client sends a command with a `sequenceNumber` higher than the aggregate's `expectedSequenceNumber`. This indicates a client implementation bug: the client believes the aggregate is further along than it actually is.

**Root cause.** Client-side state management bug; the client may have applied an event locally that was never actually accepted by the aggregate, or it incremented the sequence counter without server confirmation.

**Detection mechanism.** `command.sequenceNumber > aggregate.expectedSequenceNumber`.

**Domain behavior.** The command is rejected. Unlike a stale command (where the client just needs to catch up), a future sequence number signals a client-side desync that requires a full state re-sync. The client should re-fetch the authoritative game state snapshot (or replay all SSE events from the beginning) and reset its local sequence counter. The command should not be automatically retried without re-sync.

**Events emitted.**
- `CommandRejected { gameId, playerId, commandId, reason: UnexpectedSequenceNumber, expectedSequenceNumber: N, receivedSequenceNumber: M }`

**Invariants that must hold.**
- INV-G2, INV-G4 (same as 6.3.1).
- INV-G6: Future sequence numbers are never silently accepted or buffered. They are rejected immediately.

---

### 6.3.3 Replayed Command (Same `commandId`, Same `sequenceNumber`)

**Description.** A client retries a command using the identical `commandId` (idempotency key) and identical `sequenceNumber` that it used in a prior attempt. The client believes the first attempt was lost in transit (no acknowledgement received), but the server actually processed it successfully.

**Root cause.** Network packet loss or timeout between server processing and client ACK delivery. The client's retry logic fires correctly — this is the intended use of idempotency keys.

**Detection mechanism.** Before evaluating sequence numbers, the aggregate (or a preceding command handler layer) checks an idempotency store: `idempotencyStore.contains(command.commandId)`. If found, the stored result is returned immediately.

**Domain behavior.** The idempotency check fires before sequence-number validation. The stored result (the events emitted on first processing) is returned to the client as if the command were just processed. No re-execution occurs. No new events are emitted. The response is identical to the original.

**Why idempotency check precedes sequence-number check:** If sequence-number validation fired first on a retry, the command would be rejected with `StaleSequenceNumber` (since the aggregate has already advanced past the sequence number). This would be incorrect: the client legitimately retried a command that succeeded. Idempotency check must be first.

**Events emitted.**
- None (the original events are replayed from cache to the client; no new events enter the event log).

**Invariants that must hold.**
- INV-I1: An idempotency key is stored permanently (for the duration of the game) upon first processing and never re-executed.
- INV-I2: The idempotency check is evaluated before any other validation (sequence number, authorization, game state).
- INV-I3: The event log contains exactly one copy of the events from the original command execution, regardless of how many retries occur.

---

### 6.3.4 Replayed Command with Different `sequenceNumber`

**Description.** A client retries a command with the same `commandId` but a different `sequenceNumber` (e.g., the client updated its local sequence counter between the first attempt and the retry, or due to a bug).

**Root cause.** Client implementation error: the idempotency key (`commandId`) should be tied to a specific command instance and never reused with different parameters.

**Detection mechanism.** The idempotency store is keyed by `commandId`. It finds an existing entry for `commandId` regardless of the `sequenceNumber` in the new request.

**Domain behavior.** The idempotency store returns the cached result from the original processing. The new (different) `sequenceNumber` is ignored entirely. The response is the same as in 6.3.3. No reprocessing occurs.

**Events emitted.**
- None (cached result returned).

**Invariants that must hold.**
- INV-I1, INV-I2, INV-I3 (from 6.3.3).
- INV-I4: The `commandId` is the sole key for idempotency lookup. Changing other fields (including `sequenceNumber`) on a retry does not produce a new execution.

---

### 6.3.5 Delayed Challenge After Window Close

**Description.** A player submits `ChallengeUnoCall` within 5 seconds of the challenge window opening according to their client's clock. However, due to network latency, the command arrives at the server after the challenge window has already closed (either by 5-second expiry, by the next player acting, or by another challenge resolving it).

**Root cause.** Client-perceived timing diverges from server-authoritative timing. Network RTT causes the command to arrive late. This is an expected and unavoidable occurrence at scale.

**Detection mechanism.** The aggregate checks `challengeWindowOpen == false` when processing the `ChallengeUnoCall`. See also: A7 — challenge window deadline is server-authoritative.

**Domain behavior.** The challenge is rejected with `reason: ChallengeWindowClosed`. No penalty is assessed to the challenger for submitting a late challenge that was in good faith within the client-visible window. (Contrast with 6.5.6, where timing manipulation is intentional abuse.) The rejection is logged but not treated as abusive.

**Events emitted.**
- `CommandRejected { gameId, playerId, commandId, reason: ChallengeWindowClosed }`

**Invariants that must hold.**
- INV-CW6: Challenge window closure is final and cannot be reopened by a late `ChallengeUnoCall`.
- INV-CW5 (from 6.1.4): No penalty for a rejected challenge.

---

### 6.3.6 Duplicate `GameCompleted` Event at Ranking Context

**Description.** The message bus delivers the `GameCompleted` event to the Ranking & Statistics (RK) context more than once, due to at-least-once delivery semantics. See Flow 3, Exception E3.6 for base coverage.

**Detection mechanism.** RK maintains an idempotency store keyed by `gameId`. On receiving `GameCompleted`, RK checks: `eloUpdateStore.contains(event.gameId)`. If found: already processed; acknowledge and discard.

**Domain behavior.** The second (and any subsequent) delivery is acknowledged to the message bus (so it is not re-delivered again) but triggers no state change. No Elo calculation is performed. No `EloUpdated` event is emitted again.

**Events emitted.**
- None (on duplicate delivery).

**Invariants that must hold.**
- INV-E1: `EloUpdated` is emitted exactly once per `gameId` in the RK context.
- INV-I3 (applied to event consumers): At-least-once delivery combined with consumer-side idempotency yields exactly-once business-logic execution.

---

## 6.4 Partial Failures Between Contexts

Context-level partial failures occur when one bounded context is temporarily unavailable while another continues operating. Because contexts communicate asynchronously via domain events, these failures manifest as message queue buildup, delayed state propagation, and eventual-consistency windows.

---

### 6.4.1 Ranking Context Down When `GameCompleted` Is Emitted

**Description.** A casual game completes (RG emits `GameCompleted`). At that moment, the RK context is temporarily unavailable (crashed, restarting, or experiencing a network partition from the message bus).

**Domain behavior.** The `GameCompleted` event remains in the message queue, unacknowledged by RK. The queue retains the event with exponential-backoff retry. RG continues operating normally — game completion is not blocked by RK availability. Players' rankings are stale (their Elo scores are not updated) until RK recovers.

**Recovery.** When RK recovers, it processes the queued events in order. Idempotency (per 6.3.6) ensures no double-update. Elo scores are updated for all pending games in recovery order. The Elo update may be minutes or hours late; this is acceptable under eventual consistency.

**Risk.** A prolonged RK outage may result in thousands of `GameCompleted` events queued. Burst processing on recovery may cause transient Elo recalculation storms. Per-player serialization (6.4.6) prevents rating corruption during the burst.

**Invariants that must hold.**
- INV-E2: Elo updates are eventually consistent. There is no hard real-time guarantee for Elo update latency.
- INV-E1 (from 6.3.6): Exactly one Elo update per `gameId` upon RK recovery.
- INV-G5 (from 6.2.5): Game completion in RG is not gated on RK acknowledgement.

---

### 6.4.2 Tournament Orchestration Down When `MatchCompleted` Is Emitted

**Description.** A tournament room's match completes (RG emits `MatchCompleted`). The Tournament Orchestration (TO) context is temporarily unavailable.

**Domain behavior.** `MatchCompleted` sits in the message queue. TO cannot advance the round. All other completed rooms in the same round are also queued. The round advancement is blocked until TO recovers. This may delay the entire tournament (hundreds of thousands of players waiting) if the outage is extended.

**Recovery.** TO processes queued `MatchCompleted` events in order on recovery. Idempotency (per `matchId`) prevents double-advancement. TO re-evaluates the round completion condition (all rooms in the round completed) after each event. If all rooms' `MatchCompleted` events have queued up, TO processes them all on recovery and advances the round at once.

**Risk.** In a 1,000,000-player tournament, a single round may have ~125,000 rooms (8 players per room). A full-round outage could queue 125,000 `MatchCompleted` events. This is a flagged domain-level operational risk. See doc 08 for the open question on TO resiliency requirements.

**Invariants that must hold.**
- INV-T3: A round does not advance until ALL rooms in that round have emitted `MatchCompleted` and TO has processed them all.
- INV-T4: Round advancement is idempotent — processing a `MatchCompleted` for an already-processed room is a no-op.

---

### 6.4.3 Room Gameplay Context Fails to Create a Tournament Room

**Description.** TO emits `TournamentRoomAssigned { roomId, playerIds, roundNumber }`. The RG context attempts to create the assigned room but fails (e.g., RG is temporarily degraded for new room creation).

**Domain behavior.** `TournamentRoomAssigned` is redelivered by the message bus (at-least-once). On each redelivery, RG retries room creation. If RG successfully creates the room on a retry: the idempotency check (`roomId`) prevents duplicate room creation. Players are notified and the game begins.

If room creation continues to fail after the retry budget is exhausted: the room is in a stuck state. Players assigned to that room cannot play. The round cannot complete because this room never completes.

**Recovery (escalation path).** An admin issues `ForceResolveRoom { roomId, reason: creation_failure, outcome: walkover }` or re-emits the `TournamentRoomAssigned` event. The domain must support an administrative resolution path for permanently-stuck rooms. This is a domain-level risk flagged in doc 08.

**Events emitted (failure path).**
- `TournamentRoomAssigned` (TO — redelivered)
- `RoomCreationFailed { roomId, reason, attemptNumber }` (RG — per failed attempt)
- `RoomCreationEscalated { roomId }` (RG — after retry budget exhausted)

**Invariants that must hold.**
- INV-R2: A `roomId` that has already been created cannot be created again. The idempotency check on `roomId` is authoritative.
- INV-T3 (from 6.4.2): The round cannot advance while a room is unresolved.

---

### 6.4.4 Event Bus Partition (Room Gameplay Cannot Communicate with Spectator View)

**Description.** The message bus between the RG context and the SV context is partitioned. Events emitted by RG do not reach SV. Spectators stop receiving game state updates.

**Domain behavior.** The gameplay in RG continues unaffected — RG's game logic does not depend on SV being current. SV becomes stale: spectators see the last-known state before the partition. Spectators may see an indefinitely frozen view.

**Recovery.** When the partition heals, SV rebuilds its projection by replaying missed events from the event log (or from a checkpoint maintained per `gameId`). Spectators receive a burst of catch-up events and then resume real-time updates.

**Events emitted (partition healing).**
- SV internally re-processes missed events from the log (no new domain events emitted).
- Spectators' SSE streams receive buffered and replayed events in order.

**Invariants that must hold.**
- INV-SV1: SV staleness never affects the authoritative game state in RG.
- INV-SV2: SV's projection, once rebuilt after partition healing, is consistent with the RG event log (no invented state).
- INV-SV3: Spectator experience degradation (stale view) is acceptable; game integrity is not compromised.

---

### 6.4.5 Audit Log Write Failure

**Description.** The Audit context cannot append an event to the immutable game log (e.g., the audit store is at capacity, experiencing a write failure, or partitioned from RG).

**Domain behavior.** The domain rule states: "every state change is appended to an immutable game log before being broadcast" (A12). This makes the audit log write a synchronous precondition of event broadcast, not a fire-and-forget side effect.

**Consequence.** If the audit log write fails, the state change event is not broadcast. The game aggregate's in-memory state has changed (the command was accepted), but the durable record and player notification are withheld. The game appears frozen to clients until the log write succeeds.

**Resolution approaches (domain tradeoff).**
- **Strong guarantee (chosen):** Audit write is synchronous and blocking. A failed write rolls back the in-memory state change (or holds it in a pending state) and returns an error to the command issuer. The game effectively pauses until audit availability is restored. This preserves full auditability at the cost of availability.
- **Relaxed approach (rejected per A12):** Broadcast without audit log. Risks unauditable state mutations — unacceptable per the domain's compliance requirements.

**Risk.** This creates a hard availability dependency on the audit store. The audit store must be high-availability. This tension between availability (1M concurrent players) and auditability is a flagged domain-level risk (see doc 08).

**Events emitted.**
- No events are broadcast until the audit write succeeds.
- `AuditWriteFailed { gameId, commandId, reason }` — internal operational event (not a domain event in the game stream).

**Invariants that must hold.**
- INV-AU1: No domain event is broadcast to clients before it is successfully appended to the immutable audit log.
- INV-AU2: The audit log is append-only and immutable once written.

---

### 6.4.6 Concurrent Elo Updates for the Same Player

**Description.** Player P finishes two casual games at nearly the same instant (possible in a multi-room scenario). The RK context receives two `GameCompleted` events referencing player P at nearly the same time. If processed concurrently, both reads of P's current Elo rating would see the same "before" value, resulting in one update being silently discarded or both using stale input.

**Root cause.** Fan-in of multiple events for the same player in a parallel event processor.

**Detection mechanism.** RK must serialize Elo updates per `playerId`. This can be achieved via per-player partitioned message queue (messages for the same `playerId` are always routed to the same consumer partition) or via an optimistic-locking versioned rating record.

**Domain behavior.** The first `GameCompleted` is processed: P's Elo is read (value R1), delta computed, Elo written as R2. The second `GameCompleted` is then processed: P's Elo is read (now R2), delta computed, Elo written as R3. The updates are sequential. No rating value is lost.

**Events emitted.**
- `EloUpdated { playerId, gameId: game1Id, oldRating: R1, newRating: R2, delta: Δ1 }`
- `EloUpdated { playerId, gameId: game2Id, oldRating: R2, newRating: R3, delta: Δ2 }`

**Invariants that must hold.**
- INV-E3: Player rating updates are serialized per `playerId`. Concurrent Elo updates for the same player are prevented.
- INV-E4: Each `EloUpdated` event records the `oldRating` and `newRating` for full auditability of the rating history.

---

## 6.5 Security and Abuse Scenarios

---

### 6.5.1 Session Takeover Attempt

**Description.** An attacker intercepts a player's session token (e.g., via network sniffing or token theft) and attempts to issue game commands on behalf of the legitimate player.

**Prevention.** Session tokens are high-entropy, time-limited, and validated on every request against the IS context (A29). HTTPS/WSS ensures transport security.

**Detection.** The legitimate player logs in on their device, triggering a new session (single-active-session invariant). IS emits `SessionInvalidated { oldSessionId, reason: new_login }`. The attacker's stolen session is revoked at this moment.

**Domain behavior.** Commands issued by the attacker using the invalidated token are rejected at the authentication layer. If the attacker was mid-game (having issued commands before the legitimate player's re-login): the game aggregate recorded those commands as legitimate (they passed validation at the time). Upon session invalidation, the attacker's connection is force-disconnected. The game aggregate receives `PlayerDisconnected` → reconnection window starts → window expires → `PlayerForfeited` (if the legitimate player cannot reconnect in time with a new valid session — though per 6.2.3, the new session cannot reconnect to the existing game slot). The legitimate player's in-game position is forfeited.

**Audit escalation.** Rapid session creation (multiple `SessionInvalidated` events for the same `playerId` in a short window) triggers an adaptive throttle flag and a security alert in the audit log. This is the primary signal for session abuse detection.

**Events emitted.**
- `SessionInvalidated { playerId, oldSessionId, reason: new_login }`
- `PlayerDisconnected { gameId, playerId }`
- `ReconnectionTimerStarted`, `ReconnectionTimerExpired`, `PlayerForfeited` (if not reconnected)
- `SecurityAlertRaised { playerId, reason: rapid_session_cycling }` (audit context — if threshold exceeded)

**Invariants that must hold.**
- INV-S3: A single player identity may have at most one active session at any time.
- INV-S1 (from 6.2.3): Invalidated session tokens are rejected on all commands immediately upon invalidation.

---

### 6.5.2 Command Flooding / Denial of Service via Game Commands

**Description.** An attacker sends thousands of game commands per second (e.g., `PlayCard`, `DrawCard`) to overwhelm the game aggregate for a specific room, or to exhaust server resources globally.

**Prevention.** Multi-layer rate limiting: per-IP, per-authenticated-user, and per-room-action. The domain-specified limit is 10 game commands per second per player per room (A30). Commands exceeding this limit are rejected without being forwarded to the aggregate.

**Domain behavior.** Rate-limited commands are rejected at the edge/gateway layer, before reaching the aggregate. The aggregate's state is unaffected. Repeat offenders receive progressively stricter limits (adaptive throttling). Persistent offenders trigger a `PlayerThrottled` or `PlayerSuspended` event in the IS context.

**Events emitted.**
- `CommandRejected { reason: RateLimitExceeded }` (per rejected command — or batched)
- `PlayerThrottled { playerId, reason: excessive_command_rate }` (IS context — if threshold exceeded)

**Invariants that must hold.**
- INV-RL1: Rate limiting is applied before aggregate command processing. The aggregate is shielded from flooding.
- INV-G1: The aggregate processes at most one command at a time per game; flooding cannot bypass serialization.

---

### 6.5.3 Replay Attack Using Old Command Packets

**Description.** An attacker captures a previously valid `PlayCard` command packet (with its `commandId` and `sequenceNumber`) and replays it, hoping to re-execute the action (e.g., to play a card they already played).

**Prevention (layer 1 — idempotency).** The idempotency store (INV-I1) finds the `commandId` already processed and returns the cached result. No re-execution occurs.

**Prevention (layer 2 — sequence number).** Even if the attacker crafts a new `commandId` but reuses the same `sequenceNumber`: the aggregate's `expectedSequenceNumber` has advanced past the replayed value. The command is rejected with `StaleSequenceNumber` (HTTP 409).

**Prevention (layer 3 — session validity).** The session token in the replayed packet must still be valid. Time-limited tokens expire, making old packet replays invalid after token expiry.

**Domain behavior.** The replay is harmless at every layer. No state change occurs.

**Events emitted.**
- `CommandRejected { reason: StaleSequenceNumber }` or cached idempotency result (no new events in the game log).

**Invariants that must hold.**
- INV-I1, INV-I2, INV-I3 (from 6.3.3).
- INV-G2: Sequence number is monotonically increasing; replayed old sequence numbers are always stale.

---

### 6.5.4 Spectator Attempting to Read Private Hand Contents

**Description.** A spectator (or a player misusing the spectator channel) attempts to query the SV context for a specific player's card hand contents.

**Prevention.** The SV Anti-Corruption Layer (ACL) strips all hand content from every `CardDrawn`, `HandDealt`, and `CardPlayed` event before writing to the spectator-side read model. The SV read model never contains card identity for any player's hand — only card counts. There is no API endpoint in the SV context that returns hand contents. The data was never written to the SV store; a query for it returns a 404 (resource not found) or an empty array.

**Domain behavior.** The query returns no data. No authorization bypass is possible because the data does not exist in the SV context — it is an absence of data, not an access-controlled presence.

**Events emitted.**
- None (read query, no state mutation).

**Invariants that must hold.**
- INV-SV4: The SV read model contains no player hand contents for any game, past or present.
- INV-SV5: The SV ACL is the sole data transformation point between RG events and the SV projection. No path exists for raw RG events (with hand contents) to reach the SV read model.

---

### 6.5.5 Player Attempting to Read an Opponent's Hand via the Game Endpoint

**Description.** A player calls the RG context's player-state endpoint with another player's `playerId`, attempting to retrieve their hand contents.

**Prevention.** Player-state endpoints are scoped to the authenticated requester. The authorization check: `session.authenticatedPlayerId == request.playerId`. If the authenticated player is requesting state for a different `playerId`, the hand contents are stripped from the response. Only public game state (card count, turn position) is returned for other players.

**Domain behavior.** The request returns public information for the queried player (card count, turn order) but omits hand contents. This is not a 403 rejection but a filtered response — the endpoint is legitimately usable to view another player's public state.

**Events emitted.**
- None (read query, filtered response).

**Invariants that must hold.**
- INV-H1: Hand contents are returned only to the authenticated player whose hand it is.
- INV-H2: No endpoint in the RG context returns hand contents for a player other than the authenticated requester.

---

### 6.5.6 Manipulating Uno Call Timing

**Description.** A player deliberately submits `CallUno` just after the 5-second deadline, exploiting network conditions or server clock imprecision to have the late call accepted. Alternatively: a player submits `ChallengeUnoCall` just after the window closes, hoping for a race.

**Prevention.** The challenge window's expiry is managed by a server-side timer (A7). The window closes when the server's timer fires and `ChallengeWindowClosed` is emitted — not when the client's timer reaches zero. Any `CallUno` or `ChallengeUnoCall` received after `ChallengeWindowClosed` is emitted is rejected, regardless of the client's perceived timing.

**Key asymmetry from 6.3.5.** In 6.3.5, a late challenge was submitted in good faith; the client's clock showed it was within the window. In this scenario, the abuse involves deliberately exploiting the timing boundary. The domain's defense is the same in both cases (server clock wins), but the audit log records the submission timestamp for pattern analysis. Repeated near-boundary submissions from the same player trigger a `SuspiciousTimingPattern` audit event.

**Events emitted.**
- `CommandRejected { reason: ChallengeWindowClosed }` (same as 6.3.5)
- `SuspiciousTimingPattern { playerId, reason: repeated_boundary_submissions }` (audit context — if pattern detected)

**Invariants that must hold.**
- INV-CW6 (from 6.3.5): Challenge window closure is final and server-authoritative.
- A7: Server clock, not client clock, determines the window's open/closed state.

---

### 6.5.7 Tournament Bracket Manipulation

**Description.** An attacker (or a malicious operator) attempts to directly advance a player who should have been eliminated — e.g., by crafting a `PlayerAdvanced` event or calling an internal API directly.

**Prevention.** `PlayerAdvanced` is only emitted by the TO context as a deterministic consequence of processing a verified `MatchCompleted` event. No external command (`AdvancePlayer`) exists in the public API. TO's advancement logic is purely policy-driven: `MatchCompleted` → evaluate tiebreaks → emit `PlayerAdvanced` for qualifying players.

The audit log records every `PlayerAdvanced` event with full provenance: the `matchId`, `matchCompleted` event ID, and the tiebreak scores that produced the decision (A31). Any `PlayerAdvanced` event without a traceable `MatchCompleted` parent would be detectable as anomalous.

**Domain behavior.** If an internal API is somehow called directly (bypassing the event-driven policy): the audit log will lack the requisite `MatchCompleted` parent event. Audit validation (a separate reconciliation process) would detect and flag this as a bracket integrity violation.

**Events emitted (detection path).**
- `BracketIntegrityViolationDetected { tournamentId, playerId, reason: unverifiable_advancement }` (audit context)

**Invariants that must hold.**
- INV-T5: Every `PlayerAdvanced` event has exactly one causal `MatchCompleted` event in the audit log.
- INV-T6: No external command can directly emit `PlayerAdvanced` without processing through the TO policy chain.

---

### 6.5.8 Collusion via Spectator Information Leakage

**Description.** Player A and Player B are in separate tournament rooms of the same round. Player A spectates Player B's room (or vice versa), hoping to gain strategic information about their future opponent's play style or current hand.

**Analysis.** The SV projection for Player B's room shows: card counts per player, the discard pile's top card, and turn order. It does not show hand contents (INV-SV4). Strategic inferences from card count alone (e.g., "Player B has 2 cards left") are low-fidelity and provide minimal competitive advantage in Uno, which is heavily chance-based.

**Residual risk.** Card count information reveals when a player is approaching a win. In a tournament context, knowing a rival is about to win could theoretically influence bet timing (if betting were implemented — it is not in this domain). The residual risk is therefore low.

**Spectator delay (A35).** Spectator feeds are intentionally delayed (A35), further reducing the utility of real-time collusion. By the time Player A sees Player B's card count drop to 1, the hand may have already resolved.

**Domain decision.** Tournament rules may restrict spectating of active tournament rooms where the spectator is also a remaining participant. This is a domain policy decision (not a technical enforcement problem). If restricted: the SV context checks `spectator.isActiveTournamentParticipant AND room.tournamentId == spectator.tournamentId AND room.roundNumber == spectator.currentRound` → deny spectating request.

**Events emitted (if restriction is enforced).**
- `SpectateRequestRejected { roomId, playerId, reason: active_participant_restriction }`

**Invariants that must hold.**
- INV-SV4 (from 6.5.4): Hand contents are never in the spectator feed regardless of collusion.

---

### 6.5.9 Joining the Same Room Twice

**Description.** A player rapidly sends two `JoinRoom` commands for the same `roomId` (e.g., double-click on the join button, or a client-side retry).

**Detection mechanism.** Two guards apply independently:
1. Idempotency key on the command (`commandId`): if the same `commandId` is used, the second is a safe retry (6.3.3) and returns the cached `PlayerJoinedRoom` result.
2. Room aggregate state check: the aggregate checks `room.members.contains(playerId)` before processing any `JoinRoom`, regardless of `commandId`.

**Domain behavior.** If the second `JoinRoom` uses the same `commandId`: idempotency returns the cached result. If it uses a new `commandId`: the aggregate's membership check fires and rejects with `reason: PlayerAlreadyInRoom`.

**Events emitted.**
- First `JoinRoom`: `PlayerJoinedRoom { roomId, playerId }`
- Second `JoinRoom` (new `commandId`): `CommandRejected { reason: PlayerAlreadyInRoom }`

**Invariants that must hold.**
- INV-R3: A player appears at most once in a room's member list.

---

### 6.5.10 Joining a Room While Already in Another Active Room

**Description.** Player P is in Room A (game in progress or waiting) and submits a `JoinRoom` command for Room B.

**Prevention.** The IS context maintains a `PlayerRoomMembershipReadModel` tracking each player's active room membership. The Room B aggregate (or a cross-context authorization check) queries this model before accepting the `JoinRoom`. Alternatively, the IS context enforces the one-active-room invariant by embedding a `activeRoomId` claim in the session, checked on every room-mutating command.

**Domain behavior.** `JoinRoom` for Room B is rejected. Player P remains a member of Room A only.

**Events emitted.**
- `CommandRejected { roomId: roomBId, playerId, reason: PlayerAlreadyInActiveRoom }`

**Invariants that must hold.**
- INV-R4: A player is a member of at most one active room at any time (A37).

---

## 6.6 Spectator Privacy Violations

---

### 6.6.1 ACL Failure: Private Hand Leaks into Spectator View

**Description.** A bug in the SV Anti-Corruption Layer causes a `CardDrawn` event to be forwarded to the SV read model with the drawn card's identity intact (rather than being stripped to only increment the card count).

**Domain behavior.** The authoritative game state in RG is unaffected — this is an SV read model corruption only. Spectators who have materialized the corrupted projection have seen hand contents they should not have.

**Detection.** Two mechanisms:
1. Schema validation on SV read model writes: SV projections have a defined schema where player hand contents must be absent. A schema validator rejects any write containing card identity for a player's hand.
2. Output projection auditing: periodic reconciliation compares SV projections against expected schema; any presence of hand data triggers an alert.

**Recovery.**
1. The corrupted SV projection for the affected game is invalidated (deleted or flagged as stale).
2. Affected spectators' SSE connections are terminated (they receive a `ProjectionInvalidated` message).
3. Spectators reconnect and receive a fresh SV projection built from the raw event log, with the ACL re-applied correctly.
4. The audit log records: `PrivacyViolation { gameId, affectedSpectators: [...], dataType: hand_contents, discoveredAt, remediedAt }`.

**Events emitted.**
- `SpectatorProjectionInvalidated { gameId, reason: acl_failure }` (internal SV event)
- `PrivacyViolation { ... }` (audit context)

**Invariants violated (temporarily).** INV-SV4: "No player hand contents may appear in the spectator projection." This is the violated invariant. Recovery restores it.

**Invariants that must hold (post-recovery).**
- INV-SV4: Restored after projection rebuild.
- INV-SV5: The ACL is the sole transformation point; the bug must be fixed before the ACL processes further events.

---

### 6.6.2 Spectator Subscribes with Elevated Permissions

**Description.** A requester with an admin-level identity claim subscribes to a game's spectator feed, hoping that elevated permissions grant access to the full (unfiltered) event stream rather than the spectator projection.

**Prevention.** The SV context exposes only one data model for game observation: the spectator-safe projection. There is no "full game state" API or "admin game view" in the SV context — the distinction does not exist in the SV data model. The ACL filters unconditionally, regardless of the requester's role. Role-based access control governs *whether* a requester can subscribe to a spectator feed; it does not govern *what data* is in the feed (the feed is always spectator-safe).

The full unfiltered event stream is accessible only by the internal Audit context, via internal system channels not exposed externally.

**Domain behavior.** The admin-role requester receives the same spectator-safe projection as any other spectator. No additional data is returned.

**Events emitted.**
- None (read operation; no state mutation).

**Invariants that must hold.**
- INV-SV6: The SV spectator projection is unconditionally filtered, regardless of the requester's identity or role.

---

### 6.6.3 Player Masquerading as Spectator After Elimination

**Description.** A tournament player is eliminated and immediately subscribes to the spectator feed for another active tournament room where their future opponents are playing.

**Analysis.** Per domain rules, eliminated players may spectate (the prohibition in A37 is on participating in active rooms, not on spectating). The spectator feed exposes only public information: card counts, discard pile top card, turn order. No hand contents.

**Strategic advantage assessment.** An eliminated player cannot exploit spectator information to influence their own game (they have no game). The information may be shared externally to collude with an active player (see 6.5.8). This risk is accepted as a domain policy decision.

**Domain behavior.** The eliminated player's spectate request is accepted normally. They receive the standard spectator projection with spectator delay (A35).

**Events emitted.**
- `SpectatorJoined { roomId, spectatorPlayerId }` (SV context)

**Invariants that must hold.**
- INV-SV4: Even for eliminated players spectating, hand contents are never exposed.

---

### 6.6.4 Spectator Replay Attack (Historical Game Replay)

**Description.** An attacker requests a replay of a completed historical game's SSE stream or event log, hoping that historical events — recorded before the ACL was properly applied — contain unfiltered card hand data.

**Prevention.** The SV context stores only the filtered `SpectatorFeed` projection. Historical replays are served from the `SpectatorFeed` (the SV-filtered event log), not from the raw RG event log. The raw RG event log contains private hand data and is accessible only internally (Audit context). Private data was never written to the SV projection; historical replays contain only what was in the SV projection at the time.

**Mitigation.** Replay requests are rate-limited (to prevent bulk data extraction attempts) and authenticated (to prevent anonymous harvesting of historical game data, even spectator-safe data).

**Domain behavior.** The requester receives the historical spectator-safe projection. No hand data is present.

**Events emitted.**
- None (read operation).

**Invariants that must hold.**
- INV-SV7: Historical spectator replays are served from the SV-filtered event log, never from the raw RG event log.
- INV-SV4: Historical projections were filtered at write time; no re-filtering at read time is necessary (though it may be applied as defense-in-depth).

---

## 6.7 Game Mechanic Edge Cases

---

### 6.7.1 First Card Flipped Is Wild Draw Four

**Description.** After dealing initial hands, the top card of the deck is flipped to start the discard pile. This card is a Wild Draw Four.

**Domain rule.** Wild Draw Four is illegal as the starting discard card. The card must be returned to the deck, the deck reshuffled with a new server-generated seed, and a new top card flipped. This repeats until the top card is not a Wild Draw Four.

**Domain behavior.**
1. `DeckInitialized` (first shuffle, seed S1).
2. `InitialCardFlipped` → card is Wild Draw Four.
3. `InitialCardRejected { reason: wild_draw_four }`.
4. `DeckReshuffled { newSeed: S2, reason: initial_card_was_wdf }`.
5. `InitialCardFlipped` → check again. If still Wild Draw Four: repeat from step 3.
6. Eventually: `InitialCardFlipped` with a valid card → `GameStarted { initialDiscardCard: [valid card] }`.

All reshuffle seeds are recorded in the game log for replay and audit (A11).

**Events emitted.**
- `InitialCardRejected { gameId, card: WildDrawFour, reason: illegal_start_card }` (per rejection)
- `DeckReshuffled { gameId, newSeed, reason: initial_card_was_wdf }` (per reshuffle)
- `GameStarted { gameId, initialDiscardCard }` (when valid)

**Invariants that must hold.**
- INV-GM1: The first discard pile card is never a Wild Draw Four.
- INV-GM2: Every reshuffle seed is recorded in the audit log before being used (A11).
- INV-GM3: The loop terminates because Wild Draw Four cards constitute 4 of 108 cards (~3.7%); the probability of all shuffles producing a Wild Draw Four is negligibly small.

---

### 6.7.2 Draw Pile Exhausted Mid-Turn

**Description.** A player is required to draw one or more cards (due to their turn, a Draw Two, or a Draw Four), but the draw pile is empty.

**Domain behavior (standard case).** The discard pile — minus its current top card (which stays face-up as the reference) — is collected, shuffled with a new server-generated seed (A11), and becomes the new draw pile. The draw then proceeds.

**Edge within the edge: Discard pile has 0 or 1 cards remaining (only the top card).** This can happen in an extreme game state where nearly all cards are in players' hands (large room, many cards drawn, few played). If the discard pile has only the top card and the draw pile is empty, there are no cards available to form a new draw pile.

**Domain behavior (degenerate case).** The draw is skipped. The turn passes without the player drawing. A `DrawSkipped { reason: no_cards_available }` event is emitted. Game continues. This is an extremely rare state (requires approximately 100+ cards in hands simultaneously).

**Events emitted (standard reshuffle).**
- `DrawPileExhausted { gameId }`
- `DiscardPileReshuffled { gameId, newSeed, cardCount }` (A11 — new seed recorded before use)
- `CardDrawn { gameId, playerId, count }` (after reshuffle)

**Events emitted (degenerate — no cards available).**
- `DrawPileExhausted { gameId }`
- `DrawSkipped { gameId, playerId, reason: no_cards_available }`
- `TurnAdvanced { gameId, nextPlayerId }`

**Invariants that must hold.**
- INV-GM2: New shuffle seed is recorded before use (A11).
- INV-GM4: The current top card of the discard pile is never included in the reshuffle — it remains face-up as the discard reference.
- INV-G5 (game continues): Draw pile exhaustion alone does not end the game.

---

### 6.7.3 Player Plays Their Last Card (Win) Without Having Called Uno

**Description.** A player plays their second-to-last card (hand size goes from 2 to 1). The challenge window opens. Before anyone challenges (or before the player calls Uno), the player immediately plays their final card on their next turn, winning the game.

**Domain rule clarification.** The Uno call obligation and the associated challenge window apply when a player's hand reaches size 1 (playing the penultimate card). Playing the final card (hand goes from 1 to 0) is the win condition — it is processed immediately as `GameWon`. The Uno call window, if still open when the final card is played, is closed by the win action (closing condition (c): next player acted — or in this case, the Uno-holding player themselves acted on their next turn).

**Wait — clarification on whose turn it is.** After playing the penultimate card: challenge window opens. The turn advances to the next player. If the next player passes or is skipped, the turn eventually comes back to the near-winner. On that turn, they play their final card. The challenge window, if still open, is atomically closed first (as per 6.1.2, sub-case a extended logic: the Uno-holding player's own subsequent play closes the window). Then the win is processed.

**However,** if the penultimate card and final card are played on consecutive turns without intervention: the challenge window closes on the next player's action (or turn start). A player cannot play two cards in one turn.

**Key invariant.** Uno call obligation (and potential penalty) only applies if the player enters a new turn with hand size 1 without having called Uno and without the window having been closed by another means. Playing the final card is not a Uno-penalty event — it is a win.

**Events emitted.**
- `CardPlayed { handSize: 1 }` → `ChallengeWindowOpened { targetPlayerId }`
- (turns advance, window expires or closes)
- `CardPlayed { handSize: 0 }` → `GameWon { gameId, winnerId }`
- `ChallengeWindowClosed { reason: game_ended }` (if window was still open — edge-within-edge)

**Invariants that must hold.**
- INV-GM5: Uno challenge window opens only when hand size reaches exactly 1 (playing the penultimate card). It does not open when hand size reaches 0 (win).
- INV-GM6: `GameWon` is processed immediately upon `CardPlayed` with resulting `handSize == 0`. No additional turn advancement occurs after a win.

---

### 6.7.4 All Remaining Players Have Empty Hands Simultaneously

**Description.** Is it possible for multiple players to simultaneously empty their hands?

**Analysis.** In standard Uno, exactly one player acts per turn. A player empties their hand by playing their last card on their turn, and the game ends immediately at that point. No other player can simultaneously play their last card because it is not their turn. `GameWon` is emitted immediately upon detecting `handSize == 0`, before any other player's turn begins.

**Conclusion.** Simultaneous multi-player wins are impossible under these rules. The win condition check is immediate and blocking (no other commands are processed after `GameWon`).

**Invariants that must hold.**
- INV-GM7: Win condition is evaluated immediately after each `CardPlayed`. The first player to reach `handSize == 0` wins. No other player can reach `handSize == 0` on the same turn.
- INV-G5 (from 6.2.5): A completed game accepts no further state-mutating commands.

---

### 6.7.5 Turn Direction Reversal at the Last Turn (Interplay with Challenge Window)

**Description.** A player plays a Reverse card. This changes the turn direction. The reversal affects who the "next player" is — specifically for the Uno challenge window (which specifies "challenge closes when the next player acts").

**Domain behavior.** `DirectionReversed` is emitted as part of processing the `CardPlayed` event for the Reverse card. The aggregate immediately recalculates `nextPlayerId` using the new direction. Any open challenge window's "next player" reference is updated atomically with the direction reversal.

**Events emitted.**
- `CardPlayed { card: Reverse, playerId }`
- `DirectionReversed { gameId, newDirection }`
- `TurnAdvanced { gameId, nextPlayerId: [recalculated under new direction] }`

**Invariants that must hold.**
- INV-GM8: `nextPlayerId` after a Reverse card is always computed using the new direction, not the old direction.
- INV-CW7: The challenge window's "next player closes the window" condition uses the `nextPlayerId` as recalculated after any direction change on the same turn.

---

### 6.7.6 Skip Card When Only Two Players Remain

**Description.** One of the last two players plays a Skip card.

**Domain behavior.** A Skip card causes the next player to lose their turn. With two players: the "next player" is the only other player, so they are skipped. The turn returns to the player who played the Skip. This is equivalent to the Skip player taking two consecutive turns.

**Events emitted.**
- `CardPlayed { card: Skip }`
- `PlayerSkipped { gameId, skippedPlayerId }`
- `TurnAdvanced { gameId, nextPlayerId: [same player who played Skip] }`

**Invariants that must hold.**
- INV-GM9: Skip card always causes the next player (in turn order) to lose their turn, regardless of total player count.

---

### 6.7.7 Reverse Card in a Two-Player Game

**Description.** One of the two remaining players plays a Reverse card.

**Domain rule.** Per standard Uno rules, in a two-player game, Reverse acts as Skip: the playing player takes another turn.

**Domain behavior.** `DirectionReversed` is emitted (the direction formally reverses, even in a two-player game, for consistency with the game model). Since there are only two players, the "next player" in the new direction is still the same player who played the Reverse. `TurnAdvanced` points back to the Reverse-playing player.

**Events emitted.**
- `CardPlayed { card: Reverse }`
- `DirectionReversed { gameId, newDirection }`
- `TurnAdvanced { gameId, nextPlayerId: [same player who played Reverse] }`

**Invariants that must hold.**
- INV-GM10: In a two-player game, a Reverse card results in the playing player retaining the turn. `DirectionReversed` is still emitted for model consistency.
- INV-GM8 (from 6.7.5): `nextPlayerId` is computed under the new direction (which, with two players, loops back to the same player).

---

### 6.7.8 Match Outcome Not Decidable After Full Tiebreak Chain

**Description.** In a room running a best-of-3 match format with 4+ players, after all games are complete, multiple players are tied on all three tiebreak criteria: (1) match wins, (2) cumulative card points (lower is better), (3) earliest game completion time.

**Root cause.** Theoretically possible (e.g., two players each win one game, one game is abandoned — both have 1 win, identical card points, and completed at the same millisecond). Practically near-impossible, but the domain must specify behavior.

**Domain behavior.** The tiebreak chain (A16) produces no unique ordering among the tied players. The domain's declared assumption: tied players are considered equally ranked. If more than 3 players from one room need to advance (because the tie creates, e.g., 4 players all qualifying for top-3), the tournament must absorb the extra player in the next round's room assignment.

**Handling mechanism.** TO emits `PlayerAdvanced` for all tied-qualifying players (even if the count exceeds 3). The next round's room assignment accommodates the extra players by creating rooms of size N+1 for that round. If the room size would exceed `maxPlayers` (10), additional rooms are created as needed.

**Events emitted.**
- `TiebreakerExhausted { roomId, tiedPlayerIds: [...], tiebreakChain: [wins, cardPoints, completionTime] }` (TO context)
- `PlayerAdvanced { tournamentId, playerId }` (for each tied player — even if >3)
- `RoomOversizeAdjustment { tournamentId, roundNumber, reason: tiebreaker_overflow }` (TO context — if room sizes need adjustment)

**Invariants that must hold.**
- INV-T7: The tiebreak chain is evaluated in strict order: wins → card points → completion time. No criterion is skipped or reordered.
- INV-T8: If the tiebreak chain is exhausted without a unique ranking, all tied players are advanced (rather than eliminated). No player is arbitrarily excluded.
- INV-T9: The next round's room assignment algorithm must handle variable room sizes produced by tiebreaker overflow.

---

## 6.8 Summary Table

| # | Scenario | Category | Detection | Domain Response | Key Events |
|---|----------|----------|-----------|-----------------|------------|
| 6.1.1 | Two players simultaneously play a card | Concurrent conflict | Sequence number + active player check on Game aggregate | First arrival wins; second rejected | `CardPlayed`, `CommandRejected(NotYourTurn)` |
| 6.1.2a | Next player plays while challenge window open | Concurrent conflict | `challengeWindowOpen` flag + turn ownership check | Window atomically closed, then play processed | `ChallengeWindowClosed(next_player_acted)`, `CardPlayed` |
| 6.1.2b | Non-next player plays while challenge window open | Concurrent conflict | Turn ownership check | Rejected; window unaffected | `CommandRejected(NotYourTurn)` |
| 6.1.3 | Concurrent Uno call and challenge | Concurrent conflict | Aggregate serialization; first arrival wins | Winner determined by processing order; loser rejected | `UnoCalledSuccessfully` or `UnoChallengeAccepted`, `ChallengeWindowClosed`, `CommandRejected` |
| 6.1.4 | Multiple simultaneous challenges | Concurrent conflict | `challengeWindowOpen` flag | Only first challenge resolved; others rejected | `ChallengeResolved`, `ChallengeWindowClosed`, `CommandRejected(ChallengeWindowClosed)` |
| 6.1.5 | Command during reconnection window (before handshake) | Concurrent conflict | Player `connectionStatus` check | Rejected; reconnection window continues | `CommandRejected(PlayerDisconnected)` |
| 6.1.6 | Race: reconnect vs. auto-forfeit timer | Concurrent conflict | Aggregate serialization; `reconnectionWindowExpiry` check | First-processed determines outcome; irreversible | `PlayerReconnected` or `PlayerForfeited`, `CommandRejected(ReconnectionWindowExpired)` |
| 6.1.7 | Stale sequence under high concurrency | Concurrent conflict | Sequence number comparison | Only matching sequence accepted; all others rejected | One action event, multiple `CommandRejected(StaleSequenceNumber)` |
| 6.2.1 | Disconnection during active turn | Disconnection | Heartbeat failure (IS context) | Auto-skip; forfeit if window expires on their turn | `PlayerDisconnected`, `TurnSkippedDueToDisconnection`, `ReconnectionTimerExpired`, `PlayerForfeited` |
| 6.2.2 | Disconnection not during active turn | Disconnection | Heartbeat failure (IS context) | Auto-skip; forfeit triggered on next turn after window expiry | `PlayerDisconnected`, `TurnSkippedDueToDisconnection`, `ReconnectionTimerExpired`, `PlayerForfeited` (deferred) |
| 6.2.3 | Reconnect with invalid session (new login) | Disconnection + Security | `SessionInvalidated` from IS context | Reconnect rejected; window expires; forfeit follows | `SessionInvalidated`, `PlayerForfeited`, `CommandRejected(InvalidSession)` |
| 6.2.4 | Reconnect after window expiry | Disconnection | `reconnectionWindowExpiry` timestamp check | Rejected; forfeit already issued | `CommandRejected(ReconnectionWindowExpired)` |
| 6.2.5 | Reconnect after game ended | Disconnection | Game `status: completed` check | Rejected or informational; result already recorded | `CommandRejected(GameAlreadyCompleted)` |
| 6.2.6 | Both last two players disconnect | Disconnection | `connectedPlayerCount == 0` | Both windows run; winner determined by reconnection; abandoned if both forfeit | `PlayerDisconnected`×2, `GameAbandoned` or `GameCompleted(last_player_standing)` |
| 6.2.7 | Between-rounds tournament disconnection | Disconnection | Heartbeat failure; room-ready time window | Slot reserved; between-round window; forfeit + elimination if expired | `BetweenRoundReconnectionWindowExpired`, `PlayerForfeited`, `PlayerEliminated` |
| 6.3.1 | Stale sequence number (client behind) | Stale/Replayed command | `sequenceNumber < expectedSequenceNumber` | Rejected HTTP 409; client re-syncs via SSE | `CommandRejected(StaleSequenceNumber)` |
| 6.3.2 | Future sequence number (client ahead) | Stale/Replayed command | `sequenceNumber > expectedSequenceNumber` | Rejected HTTP 409; client must full-resync | `CommandRejected(UnexpectedSequenceNumber)` |
| 6.3.3 | Replayed command (same commandId + seqNum) | Stale/Replayed command | Idempotency store lookup (checked first) | Cached result returned; no re-execution | None (cached events replayed) |
| 6.3.4 | Replayed command (same commandId, different seqNum) | Stale/Replayed command | Idempotency store lookup on commandId | Cached result returned; new seqNum ignored | None (cached events replayed) |
| 6.3.5 | Delayed challenge after window close | Stale/Replayed command | `challengeWindowOpen == false` | Rejected; no penalty | `CommandRejected(ChallengeWindowClosed)` |
| 6.3.6 | Duplicate `GameCompleted` at Ranking context | Stale/Replayed event | `gameId` in Elo idempotency store | Acknowledged; no state change | None (on duplicate) |
| 6.4.1 | Ranking context down at game completion | Partial failure | Queue buildup; no ACK from RK | Event queued; Elo delayed until RK recovers | Delayed `EloUpdated` |
| 6.4.2 | Tournament Orchestration down at match completion | Partial failure | Queue buildup; round advancement blocked | Events queued; round paused until TO recovers | Delayed `RoundAdvanced` |
| 6.4.3 | RG fails to create tournament room | Partial failure | `RoomCreationFailed` after retry; idempotency on roomId | Retried; admin escalation if persistent | `RoomCreationFailed`, `RoomCreationEscalated` |
| 6.4.4 | Event bus partition (RG ↔ SV) | Partial failure | SV stops receiving events | Game continues; SV stale; rebuilt on partition heal | SV internal replay (no new domain events) |
| 6.4.5 | Audit log write failure | Partial failure | Audit write error response | Event broadcast withheld until audit write succeeds | `AuditWriteFailed` (operational) |
| 6.4.6 | Concurrent Elo updates for same player | Partial failure | Per-player queue partition / optimistic lock | Updates serialized per playerId | `EloUpdated`×2 (sequential) |
| 6.5.1 | Session takeover attempt | Security | `SessionInvalidated` on new login; stolen token rejected | Attacker disconnected; eventual forfeit | `SessionInvalidated`, `PlayerForfeited` |
| 6.5.2 | Command flooding / DoS | Security | Per-player-per-room rate limit (10/s) | Excess commands rejected at gateway | `CommandRejected(RateLimitExceeded)`, `PlayerThrottled` |
| 6.5.3 | Replay attack with old command packet | Security | Idempotency store; stale sequence number | Harmless; no re-execution | `CommandRejected(StaleSequenceNumber)` or cached result |
| 6.5.4 | Spectator reads private hand via SV | Security | SV ACL strips hand data; no hand endpoint exists | 404 or empty response (data never written to SV) | None |
| 6.5.5 | Player reads opponent hand via RG endpoint | Security | Session playerId vs. query playerId authorization check | 403 Forbidden or filtered response (hand omitted) | None |
| 6.5.6 | Manipulating Uno call timing | Security | Server clock vs. challenge window expiry | Challenge/call rejected if window closed; repeated offenders flagged | `CommandRejected(ChallengeWindowClosed)`, `SuspiciousTimingPattern` |
| 6.5.7 | Tournament bracket manipulation | Security | No external `AdvancePlayer` command; audit provenance check | No direct advancement path; audit flags unverifiable advancements | `BracketIntegrityViolationDetected` |
| 6.5.8 | Collusion via spectator information | Security | Domain policy analysis | Card counts visible but low-value; delay (A35) applied; tournament restriction optional | `SpectateRequestRejected` (if restriction enforced) |
| 6.5.9 | Joining same room twice | Security | Idempotency key; `room.members.contains(playerId)` | First join accepted; second rejected | `PlayerJoinedRoom`, `CommandRejected(PlayerAlreadyInRoom)` |
| 6.5.10 | Joining room while in another room | Security | `PlayerRoomMembershipReadModel` / session `activeRoomId` claim | Rejected | `CommandRejected(PlayerAlreadyInActiveRoom)` |
| 6.6.1 | ACL failure: hand leaks into spectator view | Spectator privacy | Schema validation on SV write; projection reconciliation | Projection invalidated; spectators reconnected; audit logged | `SpectatorProjectionInvalidated`, `PrivacyViolation` |
| 6.6.2 | Spectator subscribes with elevated permissions | Spectator privacy | SV ACL filters unconditionally regardless of role | Standard spectator-safe projection returned | None |
| 6.6.3 | Eliminated player spectates active rooms | Spectator privacy | Domain policy analysis | Permitted; only public data exposed | `SpectatorJoined` |
| 6.6.4 | Historical game replay attack | Spectator privacy | SV replay served from filtered `SpectatorFeed` log | Spectator-safe historical projection returned; no hand data | None |
| 6.7.1 | First card flipped is Wild Draw Four | Game mechanic | `card.type == WildDrawFour` check on initial flip | Reshuffle with new seed; repeat until valid | `InitialCardRejected`, `DeckReshuffled`, `GameStarted` |
| 6.7.2 | Draw pile exhausted mid-turn | Game mechanic | Empty draw pile detected at draw time | Discard pile reshuffled (new seed); draw proceeds; if still empty: skip draw | `DrawPileExhausted`, `DiscardPileReshuffled`, `CardDrawn` or `DrawSkipped` |
| 6.7.3 | Last card played with open Uno window | Game mechanic | `handSize == 0` check on `CardPlayed` | Win processed immediately; challenge window closed | `CardPlayed`, `ChallengeWindowClosed(game_ended)`, `GameWon` |
| 6.7.4 | All players empty hands simultaneously | Game mechanic | Structural impossibility in turn-based model | Cannot occur; win check is immediate after each `CardPlayed` | N/A |
| 6.7.5 | Direction reversal affects challenge window next-player | Game mechanic | `DirectionReversed` recalculates `nextPlayerId` atomically | Challenge window's next-player reference updated with new direction | `DirectionReversed`, `TurnAdvanced` |
| 6.7.6 | Skip card with two players | Game mechanic | `playerCount == 2` + Skip card played | Skip player retains turn (equivalent to two turns) | `PlayerSkipped`, `TurnAdvanced` (back to Skip player) |
| 6.7.7 | Reverse card in two-player game | Game mechanic | `playerCount == 2` + Reverse card played | Acts as Skip; playing player retains turn | `DirectionReversed`, `TurnAdvanced` (back to Reverse player) |
| 6.7.8 | Tiebreak chain exhausted with no unique winner | Game mechanic | All three tiebreak criteria equal among tied players | All tied-qualifying players advanced; next round accommodates extra players | `TiebreakerExhausted`, `PlayerAdvanced`×N, `RoomOversizeAdjustment` |
