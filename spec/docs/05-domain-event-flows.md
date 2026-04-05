# 5. Domain Event Flow Narratives

> Timeline-based output of EventStorming sessions. Each flow captures the full command-event-policy chain for a major business scenario, including both happy paths and exception paths.

---

## Notation Convention

| Sticky Color | Meaning | Notation |
|---|---|---|
| Blue | Command (issued by actor or policy) | `CommandName` |
| Orange | Domain Event (fact that happened) | `EventName` |
| Purple | Policy (reactive rule that triggers a command) | **PolicyName** -- description |
| Red | Hotspot / Decision Point | description |
| Green | Read Model (query/projection updated) | `ReadModelName` |

---

## Flow 1: Room Creation to Game Completion (Casual Room)

This flow covers the complete lifecycle of a casual ad-hoc room, from its creation through player assembly, a full best-of-three match containing up to three individual games, and the final room closure.

[View Room Creation to Completion Diagram](../diagrams/room-creation-to-completion.md)
[View Game Turn Sequence](../diagrams/game-turn-sequence.md)
[View Uno Call Timing](../diagrams/uno-call-timing.md)

---

### Phase 1: Room Setup and Player Assembly

**1.** A logged-in player (the host) issues the command:
- `CreateRoom` with parameters: `roomType = casual`, `maxPlayers` (2--10), `matchFormat = best_of_3`

**2.** The Room aggregate validates:
- The requesting player has a valid, active session (enforced via Identity/Session context).
- The requesting player is not already participating in another active room.
- `maxPlayers` is within the allowed range [2, 10].
- `EventName`: `RoomCreated` with `roomId`, `hostPlayerId`, `roomType = casual`, `maxPlayers`, `matchFormat`, `status = waiting`.

**3.** `RoomCreated` -- Read model updated:
- `AvailableRoomsReadModel` gains a new entry for lobby browsing.
- `PlayerRoomMembershipReadModel` records the host as a member.

**4.** Other players discover the room (via lobby listing or invite link) and issue:
- `JoinRoom` with `roomId`, `playerId`.

**5.** For each `JoinRoom`, the Room aggregate validates:
- Room status is `waiting` (not yet started, not full, not completed).
- The player has a valid, active session.
- The player is not already in another active room.
- The room has not reached `maxPlayers`.
- If all validations pass: `PlayerJoinedRoom` with `roomId`, `playerId`, `currentPlayerCount`.
- If any validation fails: the command is rejected with the appropriate error (e.g., `RoomFull`, `PlayerAlreadyInRoom`, `InvalidSession`).

**6.** `PlayerJoinedRoom` -- Read model updates:
- `AvailableRoomsReadModel` updates player count.
- `PlayerRoomMembershipReadModel` records the new member.
- `RoomParticipantsReadModel` refreshed for all current members (real-time push via SSE).

**7.** Repeat step 4--6 until the host decides to start or the room fills.

**8.** The host issues:
- `StartMatch` (requires at least 2 players in the room).

**9.** The Room aggregate validates the start conditions:
- Issuer is the host.
- At least 2 players are present.
- Room status is `waiting` or `ready`.
- If validation fails: command rejected.
- If validation passes: `GameStartRequested` with `roomId`, `matchNumber = 1`, `gameNumber = 1`.

**10.** `GameStartRequested` triggers:
- `InitializeDeck` (internal command within the Game aggregate).
- The server generates a cryptographically seeded shuffle. The `shuffleSeed` is recorded in the immutable game log for auditability and replay.
- A standard 108-card Uno deck is shuffled server-side (authoritative).
- `DeckInitialized` with `gameId`, `shuffleSeed`, `deckSize = 108`.

**11.** `DeckInitialized` triggers:
- `DealInitialHands` -- deal 7 cards to each player in turn order.
- Cards are removed from the top of the shuffled deck and assigned to each player.
- `InitialHandsDealt` with `gameId`, per-player card assignments (private, not broadcast publicly), `remainingDeckSize`.

**12.** `InitialHandsDealt` triggers:
- `FlipFirstDiscardCard` -- the top card of the remaining deck is placed face-up on the discard pile.
- `FirstCardFlipped` with `gameId`, `card`, `discardPileTopCard`.

**13.** First card special handling:
- If the first flipped card is a **number card**: no special effect; play proceeds normally.
- If the first flipped card is a **Skip**: the first player is skipped; `PlayerSkipped` emitted; turn advances to the second player.
- If the first flipped card is a **Reverse**: play direction is reversed (clockwise becomes counter-clockwise); `DirectionReversed` emitted; in a 2-player game, this acts as a Skip.
- If the first flipped card is a **Draw Two**: the first player must draw 2 cards before their turn; `PenaltyCardsDrawn` emitted with `playerId`, `cardCount = 2`; their turn is skipped.
- If the first flipped card is a **Wild**: the host (or first player) is prompted to declare a color; `DeclareColor` command expected; `ColorDeclared` emitted.
- If the first flipped card is a **Wild Draw Four**: the card is returned to the deck, the deck is reshuffled, and a new first card is flipped. Repeat until a non-Wild-Draw-Four card appears.

**14.** After first-card resolution:
- `GameStarted` with `gameId`, `roomId`, `matchNumber`, `gameNumber`, `turnOrder` (list of playerIds), `initialDirection = clockwise` (unless reversed), `firstActivePlayerId`.
- `TurnAdvanced` with `gameId`, `activePlayerId`, `turnNumber = 1`, `turnTimeoutDeadline`.

**15.** `GameStarted` -- Read model updates:
- `AvailableRoomsReadModel` marks the room as `in_progress`.
- `SpectatorGameStateReadModel` initialized with discard pile top card, player card counts (no hand details).
- `PlayerGameStateReadModel` initialized for each player with their private hand.

---

### Phase 2: Active Gameplay Loop

This phase repeats for every turn until a player empties their hand or the game is otherwise terminated.

**16.** The active player receives notification via SSE that it is their turn (driven by `TurnAdvanced` event). A turn timer begins (configurable, e.g., 30 seconds per turn).

**17.** **Path A -- Player plays a card:**
- The active player issues `PlayCard` with `gameId`, `playerId`, `cardId`, `sequenceNumber`.
- Optional: if the card is Wild or Wild Draw Four, the command also includes `declaredColor`.

**18.** The Game aggregate validates the `PlayCard` command:
- a) The `playerId` matches the current active player.
- b) The `sequenceNumber` matches the expected next sequence number (optimistic concurrency).
- c) The `cardId` exists in the player's hand.
- d) The card is a legal play given the current discard pile top card and declared color:
    - Same color, same number, same action symbol, or Wild/Wild Draw Four.
    - Wild Draw Four has an additional legality constraint: the player must not hold any card matching the current color (honor system in casual, but server-enforced in UnoArena).
- e) If the card is Wild or Wild Draw Four: a `declaredColor` must be provided (red, blue, green, or yellow).
- If any validation fails: `CommandRejected` with reason code (e.g., `NotYourTurn`, `InvalidCard`, `StaleSequenceNumber`). The client reconciles state via SSE.

**19.** If `PlayCard` is valid:
- `CardPlayed` with `gameId`, `playerId`, `card`, `sequenceNumber`, `newDiscardTop`, `remainingHandSize`.
- The card is removed from the player's hand and placed on the discard pile.
- The event is appended to the immutable game log.

**20.** Post-play effects based on card type:
- **Number card**: No special effect. Proceed to step 26.
- **Skip**: `PlayerSkipped` emitted with the next player's ID. The skipped player's turn does not occur. Turn advances to the player after the skipped one.
- **Reverse**: `DirectionReversed` emitted. The turn order direction toggles. In a 2-player game, Reverse acts as Skip.
- **Draw Two**: `PenaltyCardsDrawn` emitted. The next player draws 2 cards from the deck (server-side, authoritative). Their turn is skipped. `PlayerSkipped` emitted.
- **Wild**: `ColorDeclared` emitted with the chosen color. The declared color becomes the active color for matching.
- **Wild Draw Four**: `ColorDeclared` emitted. The next player draws 4 cards (`PenaltyCardsDrawn` with `cardCount = 4`). Their turn is skipped. `PlayerSkipped` emitted.

**21.** After `CardPlayed` -- check for Uno condition:
- If the player now has exactly **1 card** remaining:
    - `ChallengeWindowOpened` with `gameId`, `targetPlayerId`, `windowDuration = 5 seconds`, `windowDeadline`.
    - The challenge window allows the target player to call Uno and opponents to challenge.
    - Proceed to Uno Call sub-flow (steps 28--35).

**22.** After `CardPlayed` -- check for win condition:
- If the player now has exactly **0 cards** remaining:
    - The player has won this game. Proceed to Phase 3 (step 36).

**23.** **Path B -- Player draws a card:**
- The active player issues `DrawCard` with `gameId`, `playerId`, `sequenceNumber`.
- Validated: correct player, correct sequence number, it is their turn.
- The server draws the top card from the deck (authoritative).
- `CardDrawn` with `gameId`, `playerId`, `drawnCard` (private to the player), `remainingDeckSize`.

**24.** After drawing:
- If the drawn card is playable (matches current discard top color, number, or is Wild):
    - The player may issue `PlayCard` for the drawn card immediately (optional play).
    - Or the player may issue `PassTurn` to end their turn without playing.
- If the drawn card is not playable:
    - The turn automatically passes. `TurnPassed` emitted with `gameId`, `playerId`.

**25.** **Path C -- Turn timer expires:**
- If the active player does not act within the turn time limit:
    - `TurnTimedOut` with `gameId`, `playerId`.
    - **Policy: AutoDrawOnTimeout** -- the system automatically draws a card for the player.
    - If the drawn card is playable, it is not auto-played; the turn passes.
    - `CardDrawn` emitted, then `TurnPassed` emitted.

**26.** After all post-play effects are resolved:
- `TurnAdvanced` with `gameId`, `nextActivePlayerId`, `turnNumber`, `direction`, `turnTimeoutDeadline`.
- The next player is notified. The loop returns to step 16.

**27.** Read model updates after each turn action:
- `SpectatorGameStateReadModel` updated: new discard top, player card counts, turn indicator, direction.
- `PlayerGameStateReadModel` updated for the acting player and any affected players (e.g., player who drew penalty cards).
- `GameLogReadModel` appended with the event for audit trail.

---

### Uno Call Sub-Flow (steps 28--35)

**28.** `ChallengeWindowOpened` is emitted (from step 21). The 5-second window begins.

**29.** **Path A -- Player calls Uno in time:**
- The target player issues `CallUno` with `gameId`, `playerId`.
- This command may be issued concurrently with `PlayCard` (i.e., the player may include the Uno call at the same time as playing their penultimate card, or issue it as a separate command within the window).
- `UnoCallMade` with `gameId`, `playerId`, `timestamp`.

**30.** **Path B -- Opponent challenges the Uno call:**
- Any opponent issues `ChallengeUnoCall` with `gameId`, `challengerId`, `targetPlayerId`.
- Validated: the challenge window is still open, the challenger is a participant in the game, the target is the player who just played.

**31.** Challenge resolution:
- **If the target player DID call Uno (step 29 occurred before the challenge):**
    - The challenge fails. The challenger is penalized.
    - `ChallengeResolved` with `outcome = challenger_penalized`.
    - **Policy: PenalizeFailedChallenger** -- the challenger draws 2 penalty cards.
    - `PenaltyCardsDrawn` with `playerId = challengerId`, `cardCount = 2`, `reason = failed_uno_challenge`.

- **If the target player did NOT call Uno (step 29 has not occurred):**
    - The challenge succeeds. The target is penalized.
    - `ChallengeResolved` with `outcome = target_penalized`.
    - **Policy: PenalizeUncalledUno** -- the target player draws 2 penalty cards.
    - `PenaltyCardsDrawn` with `playerId = targetPlayerId`, `cardCount = 2`, `reason = missed_uno_call`.
    - The target player now has 3 cards (1 remaining + 2 penalty).

**32.** **Path C -- Window expires with no challenge and no Uno call:**
- `ChallengeWindowClosed` with `gameId`, `targetPlayerId`, `reason = expired`.
- No penalty is applied. The player "got away with" not calling Uno.

**33.** **Path D -- Window closes because the next player acts:**
- If the next player issues a command (e.g., `PlayCard` or `DrawCard`) before the window expires:
    - `ChallengeWindowClosed` with `reason = next_player_acted`.
    - Any pending challenge is no longer valid.

**34.** **Path E -- Player calls Uno concurrently with PlayCard:**
- The `CallUno` command can be submitted alongside or immediately after `PlayCard`.
- The system processes `PlayCard` first (validates, emits `CardPlayed`), then processes `CallUno`.
- If both arrive within the same processing cycle, Uno is considered called.

**35.** After the challenge window closes (by any path), normal gameplay resumes from step 26.

---

### Phase 3: Game Completion and Match Progression

**36.** A player plays their last card (hand size reaches 0):
- `CardPlayed` emitted (as in step 19).
- `GameCompleted` with `gameId`, `roomId`, `matchNumber`, `gameNumber`, `winnerId`, `placements` (ordered list of players by elimination/remaining cards), `cardPointTotals` (per player, based on cards remaining in their hands).
- Card point calculation: Number cards = face value, Skip/Reverse/DrawTwo = 20 points, Wild/WildDrawFour = 50 points.
- All remaining players' hands are scored.

**37.** `GameCompleted` -- Read model updates:
- `MatchStandingsReadModel` updated with the game result.
- `GameLogReadModel` finalized with outcome.
- `SpectatorGameStateReadModel` updated to show final state.

**38.** **Policy: CheckMatchProgress** evaluates the match state:
- Count each player's game wins within this match.
- **Decision point**: Has any player won 2 games (best-of-3 threshold)?

**39.** **Path A -- Match not yet decided (fewer than 2 wins by any player):**
- `MatchGameCompleted` with `matchNumber`, `gameNumber`, `nextGameNumber`.
- **Policy: StartNextGameInMatch** triggers `StartGame` for the next game within the same match.
- Return to step 10 (deck initialization) for the new game. Turn order rotates: the starting player shifts by one position.

**40.** **Path B -- Match decided (a player has 2 wins):**
- `MatchCompleted` with `roomId`, `matchNumber`, `matchWinnerId`, `matchPlacements` (ordered by: match wins descending, then cumulative card points ascending, then earliest final-game completion time).
- All players receive the match result.

**41.** `MatchCompleted` -- calculate final room standings:
- Since this is a casual room with a single match, the match placements become the room placements.
- `RoomCompleted` with `roomId`, `finalStandings` (ordered placement list), `roomType = casual`.

**42.** `RoomCompleted` -- Read model updates:
- `AvailableRoomsReadModel` removes the room from active listings.
- `PlayerRoomMembershipReadModel` clears all memberships.
- `PlayerMatchHistoryReadModel` updated for each participant.

**43.** **Policy: TriggerCasualEloUpdate** -- since the room is casual:
- For each completed game within the match, emit `RequestEloUpdate` toward the Ranking bounded context.
- This is an asynchronous cross-context event. See Flow 3 for details.

---

### Exception Paths within Flow 1

#### E1.1: Player Disconnects Mid-Game

**44.** The system detects that a player's connection has dropped (heartbeat missed / SSE connection closed).
- `PlayerDisconnected` with `gameId`, `playerId`, `disconnectedAt`.

**45.** **Policy: StartReconnectionTimer** -- a 60-second reconnection window begins.
- `ReconnectionTimerStarted` with `playerId`, `gameId`, `expiresAt`.

**46.** During the reconnection window:
- If it becomes the disconnected player's turn: their turn is automatically skipped (as if they passed).
- `TurnSkippedDueToDisconnection` with `gameId`, `playerId`.
- `TurnAdvanced` to the next player.
- This repeats each time the turn rotation reaches the disconnected player.

**47.** **Path A -- Player reconnects within 60 seconds:**
- Player issues `Reconnect` with `gameId`, `playerId`, `sessionToken`.
- Session validated (same player, valid token, reconnection window still open).
- `PlayerReconnected` with `gameId`, `playerId`, `reconnectedAt`.
- `ReconnectionTimerCancelled`.
- Player resumes with their original hand intact. On their next turn, they play normally.

**48.** **Path B -- Reconnection window expires:**
- `ReconnectionTimerExpired` with `gameId`, `playerId`.
- **Policy: AutoForfeitOnTimeout** -- the player is automatically forfeited.
- `PlayerForfeited` with `gameId`, `playerId`, `reason = reconnection_timeout`.
- The player is removed from the active turn order.
- The game continues with remaining players.

#### E1.2: Stale Command (Sequence Number Mismatch)

**49.** A player issues `PlayCard` with a `sequenceNumber` that does not match the server's expected sequence number.
- This typically occurs when the client's local state is behind the server (e.g., another event was emitted between the client's last sync and this command).
- `CommandRejected` with `gameId`, `playerId`, `reason = stale_sequence_number`, HTTP status 409 Conflict.
- The client receives updated state via SSE and may retry with the correct sequence number.

#### E1.3: Draw Pile Exhausted

**50.** A `DrawCard` command is issued but the draw pile is empty.
- **Policy: ReshuffleDiscardPile** -- take all cards from the discard pile except the current top card, shuffle them (new server-side seed recorded), and form a new draw pile.
- `DrawPileReshuffled` with `gameId`, `newShuffleSeed`, `newDeckSize`.
- The draw then proceeds from the new pile.
- If the discard pile also has no cards to reshuffle (extremely rare edge case with many players): `DrawCard` is skipped, and the player's turn passes.

#### E1.4: All Players But One Forfeit

**51.** After a `PlayerForfeited` event, the Game aggregate checks remaining active players.
- If only 1 player remains:
    - `GameCompleted` with `winnerId = last remaining player`, `reason = all_opponents_forfeited`.
    - `MatchCompleted` (the last player wins the match by default since opponents cannot contest further games).
    - `RoomCompleted`.

#### E1.5: All Players Forfeit (Game Abandoned)

**52.** If all remaining players disconnect and their reconnection windows expire:
- `GameAbandoned` with `gameId`, `roomId`.
- `MatchAbandoned` with `roomId`, `matchNumber`.
- `RoomCompleted` with `finalStandings = none`, `outcome = abandoned`.
- **Policy: SuppressEloUpdate** -- no Elo changes are applied for abandoned games.

#### E1.6: Duplicate / Replayed Command

**53.** A player sends the same command twice (same `sequenceNumber` and `commandId`):
- If the command has already been processed (idempotency check via `commandId`): the system returns the previously emitted response without re-processing.
- If the `sequenceNumber` has advanced past the duplicated one: `CommandRejected` with `reason = stale_sequence_number`.

---

## Flow 2: Tournament Round Advancement

This flow covers the lifecycle of a tournament from creation through multi-round elimination to final completion, supporting up to 1,000,000 concurrent participants.

[View Tournament Progression](../diagrams/tournament-progression.md)
[View Tournament Round Flow](../diagrams/tournament-round-flow.md)

---

### Phase 1: Tournament Setup and Registration

**1.** A tournament organizer issues:
- `CreateTournament` with `name`, `scheduledStartTime`, `minPlayers`, `maxPlayers` (up to 1,000,000), `registrationDeadline`.

**2.** The Tournament aggregate validates:
- The organizer has the required permissions.
- `scheduledStartTime` is in the future.
- `maxPlayers` is within system limits.
- `TournamentCreated` with `tournamentId`, `organizerId`, `name`, `scheduledStartTime`, `status = draft`.

**3.** `TournamentCreated` -- Read model updates:
- `TournamentListReadModel` gains a new entry.
- `TournamentDetailReadModel` initialized.

**4.** The organizer issues:
- `OpenRegistration` with `tournamentId`.
- `RegistrationOpened` with `tournamentId`, `registrationDeadline`.
- `TournamentListReadModel` updated to show registration is open.

**5.** Players register by issuing:
- `RegisterForTournament` with `tournamentId`, `playerId`.

**6.** For each registration, the Tournament aggregate validates:
- The player has a valid, active session.
- The player is not already registered for this tournament.
- Registration is open (status, deadline not passed).
- The tournament has not reached `maxPlayers`.
- `PlayerRegistered` with `tournamentId`, `playerId`, `registrationNumber`, `currentRegisteredCount`.
- If validation fails: command rejected with reason (e.g., `AlreadyRegistered`, `RegistrationClosed`, `TournamentFull`).

**7.** `PlayerRegistered` -- Read model updates:
- `TournamentParticipantsReadModel` updated.
- `TournamentDetailReadModel` updated with current count.

**8.** Registration continues until the organizer starts the tournament or the scheduled start time arrives.

**9.** The organizer (or a scheduled trigger) issues:
- `StartTournament` with `tournamentId`.

**10.** The Tournament aggregate validates:
- Status is `registration_open`.
- At least `minPlayers` have registered.
- Issuer is the organizer (or it is the scheduled auto-start).
- `RegistrationClosed` with `tournamentId`, `finalPlayerCount`.

**11.** Round structure calculation:
- Calculate the number of elimination rounds needed.
- With rooms of up to 10 players and top 3 advancing, each round reduces the player pool by approximately 70%.
- Formula: keep dividing `playerCount` by the room size and taking the top 3 until 10 or fewer remain.
    - Example with 1,000,000 players:
        - Round 1: ~100,000 rooms of 10 players each. Top 3 from each room = 300,000 advance.
        - Round 2: ~30,000 rooms. Top 3 = 90,000 advance.
        - Round 3: ~9,000 rooms. Top 3 = 27,000 advance.
        - Round 4: ~2,700 rooms. Top 3 = 8,100 advance.
        - Round 5 (Final): 1 final room is not possible with 8,100. So ~810 rooms. Top 3 = 2,430.
        - Round 6: ~243 rooms. Top 3 = 729.
        - Round 7: ~73 rooms. Top 3 = 219.
        - Round 8: ~22 rooms. Top 3 = 66.
        - Round 9: ~7 rooms. Top 3 = 21.
        - Round 10: ~3 rooms. Top 3 = 9.
        - Round 11 (Final): 1 final room with 9 players.
    - The exact number of rounds is determined by `ceil(log(playerCount) / log(10/3))` approximately.
- `TournamentStarted` with `tournamentId`, `playerCount`, `estimatedRounds`, `status = in_progress`.

**12.** `TournamentStarted` -- Read model updates:
- `TournamentBracketReadModel` initialized with round structure.
- `TournamentDetailReadModel` updated.
- All registered players notified via event propagation.

---

### Phase 2: Round Execution

**13.** The Tournament Orchestration context issues:
- `CreateRound` with `tournamentId`, `roundNumber = 1`, `playerIds` (all registered players for round 1, or advancing players for subsequent rounds).

**14.** `RoundCreated` with `tournamentId`, `roundNumber`, `totalPlayersInRound`, `status = pending`.

**15.** Player distribution into rooms:
- **Policy: DistributePlayersIntoRooms** -- divide the player pool into rooms of up to 10 players.
    - If `playerCount % 10 == 0`: all rooms have exactly 10 players.
    - If `playerCount` does not divide evenly: some rooms have fewer players (minimum 2 per room).
    - Distribution algorithm ensures fairness (e.g., random assignment or seeded by Elo to avoid clustering top players).
- For each room: `CreateRoom` is issued with `roomType = tournament`, `tournamentId`, `roundNumber`, `assignedPlayerIds`, `matchFormat = best_of_3`.
- Each room creation emits `RoomCreated` with `roomType = tournament`.

**16.** `RoundRoomsCreated` with `tournamentId`, `roundNumber`, `roomCount`, `roomIds`.

**17.** For each room, players are auto-joined:
- `AutoJoinTournamentRoom` for each assigned player.
- `PlayerJoinedRoom` emitted per player per room.
- Players who are disconnected at this point have their 60-second reconnection window to join before being forfeited.

**18.** Each room's game starts automatically (no host trigger needed in tournament rooms):
- **Policy: AutoStartTournamentRoom** -- once all assigned players have joined (or their reconnection window has expired), the game begins.
- The room follows the same gameplay flow as Flow 1, Phase 2 (steps 10--35), with the room type set to `tournament`.

**19.** All rooms in a round execute their best-of-3 matches simultaneously.
- At 1,000,000 players in round 1, this means approximately 100,000 rooms playing concurrently.
- Each room is independent and processes its own game state.

**20.** As each room completes its match:
- `MatchCompleted` emitted from the Room Gameplay context with `roomId`, `tournamentId`, `roundNumber`, `matchPlacements`.

**21.** `MatchCompleted` -- cross-context event received by Tournament Orchestration:
- **Policy: RecordMatchResult** -- store the match result and identify the top 3 players who advance.
- `MatchResultRecorded` with `tournamentId`, `roundNumber`, `roomId`, `advancingPlayerIds` (top 3 by match wins, then card points, then completion time), `eliminatedPlayerIds`.

**22.** For each eliminated player:
- `PlayerEliminated` with `tournamentId`, `playerId`, `eliminatedInRound`, `finalPlacement`.
- Player is notified of their elimination.
- `TournamentParticipantsReadModel` updated.

**23.** **Policy: CheckRoundCompletion** -- after each `MatchResultRecorded`:
- Count completed rooms vs. total rooms in the round.
- If all rooms have completed: proceed to step 24.
- If not all rooms are complete: wait. This is the **critical synchronization point** at scale.
    - Must handle stragglers (rooms that take significantly longer).
    - Must handle stuck rooms (no activity for extended period).
    - **Policy: RoundTimeoutEscalation** -- if a room in the round has not completed within a configurable timeout (e.g., 2 hours), escalate for review or auto-resolve.

**24.** All rooms in the round are complete:
- `RoundCompleted` with `tournamentId`, `roundNumber`, `advancingPlayerCount`, `eliminatedPlayerCount`.

**25.** `RoundCompleted` -- Read model updates:
- `TournamentBracketReadModel` updated with round results and advancement lines.
- `TournamentDetailReadModel` updated with current round status.
- `TournamentStandingsReadModel` updated.

---

### Phase 3: Advancement and Next Round

**26.** After `RoundCompleted`, the Tournament aggregate evaluates:
- Collect all advancing player IDs from the completed round.
- **Decision point**: How many players are advancing?

**27.** **Path A -- More than 10 players advancing:**
- `AdvancePlayers` with `tournamentId`, `nextRoundNumber`, `advancingPlayerIds`.
- `PlayersAdvanced` with `tournamentId`, `roundNumber`, `advancingPlayerCount`.
- Return to step 13 (`CreateRound`) for the next round with the advancing player pool.

**28.** **Path B -- 10 or fewer players advancing:**
- This is the final. Instead of creating a new round with multiple rooms:
- `CreateFinalRoom` with `tournamentId`, `finalPlayerIds`.
- `FinalRoomCreated` with `tournamentId`, `roomId`, `playerCount`.
- Proceed to Phase 4.

---

### Phase 4: Finals and Tournament Completion

**29.** The final room plays its best-of-3 match following the standard gameplay flow (Flow 1, steps 10--35).
- This room is flagged as the tournament final; spectator interest is expected to be high.
- `SpectatorGameStateReadModel` receives enhanced updates for the final room.

**30.** The final room's match completes:
- `MatchCompleted` from the final room with full placement data.

**31.** `MatchCompleted` from the final room triggers:
- **Policy: CompleteTournament** -- finalize the tournament.
- `CompleteTournament` command issued with `tournamentId`, `finalRoomPlacements`.

**32.** Final standings calculated:
- The final room placements determine the top positions (1st through Nth based on final room size).
- Players eliminated in earlier rounds receive placements based on the round they were eliminated in and their match performance.
- `TournamentCompleted` with `tournamentId`, `finalStandings` (complete ordered list of all participants), `tournamentDuration`.

**33.** `TournamentCompleted` -- Read model updates:
- `TournamentBracketReadModel` finalized.
- `TournamentDetailReadModel` marked as completed.
- `TournamentStandingsReadModel` finalized with all placements.
- `TournamentHistoryReadModel` archived.

**34.** **Policy: UpdateTournamentPlacementRatings** -- tournament placement ratings are updated:
- This uses the tournament-specific rating system (separate from casual Elo).
- `TournamentPlacementRatingUpdated` for each participant based on their final standing.

---

### Exception Paths within Flow 2

#### E2.1: Player Forfeits in Tournament Room

**35.** A player forfeits (voluntarily or due to disconnection timeout) during a tournament room's match:
- `PlayerForfeited` with `gameId`, `playerId`, `reason`.
- In the context of the tournament: the player is eliminated from the tournament.
- `PlayerEliminated` with `tournamentId`, `playerId`, `eliminatedInRound`, `reason = forfeit`.
- The room's game continues with remaining players.

#### E2.2: Room With Only One Player Remaining

**36.** All players in a tournament room forfeit except one:
- `GameCompleted` with `winnerId = last remaining player`, `reason = all_opponents_forfeited`.
- `MatchCompleted` for the room -- the sole remaining player is ranked 1st.
- That player auto-advances. The other 2 advancement slots for the room are left empty (fewer advancers from this room).
- **Policy: RecordMatchResult** handles rooms with fewer than 3 advancers gracefully.

#### E2.3: Tournament Room Times Out

**37.** A room in a tournament round has been playing for an extended period without completing:
- `RoomTimeoutWarning` emitted at a configurable threshold (e.g., 90 minutes).
- If the room still has not completed after the hard timeout (e.g., 2 hours):
    - `RoomTimeoutForced` with `roomId`, `tournamentId`, `roundNumber`.
    - **Policy: ForceResolveTimedOutRoom** -- the current match standing is used to determine placements. The game in progress is declared complete based on current card counts.
    - `MatchCompleted` emitted with `reason = timeout_forced`.
    - `TournamentBracketReadModel` flags this room as timeout-resolved.

#### E2.4: Player Disconnects Between Rounds

**38.** A player who advanced is disconnected when the next round's room is created:
- The player's slot is reserved in the new room.
- The 60-second reconnection window applies from the moment the room is ready and `AutoJoinTournamentRoom` is attempted.
- If the player reconnects within 60 seconds: they join the room normally.
- If the reconnection window expires: `PlayerForfeited` for the new room. `PlayerEliminated` from the tournament.
- The room proceeds with one fewer player (as long as at least 2 remain).

#### E2.5: Tournament Cancelled Mid-Way

**39.** The organizer issues `CancelTournament` with `tournamentId`:
- Validated: tournament is in progress, issuer is the organizer.
- `TournamentCancelled` with `tournamentId`, `reason`, `cancelledInRound`.
- All active rooms in the current round receive `RoomCancelled`.
- All remaining players are notified.
- No placement changes or rating updates are applied.
- `TournamentDetailReadModel` marked as cancelled.

---

## Flow 3: Elo/Ranking Update After Game Completion

This flow describes the cross-context event propagation from the Room Gameplay context to the Ranking context when a casual game is completed.

[View Elo Update Pipeline](../diagrams/elo-update-flow.md)

---

### Trigger

**1.** The Room Gameplay context emits `GameCompleted` with:
- `gameId`, `roomId`, `roomType`, `placements` (ordered list of players from 1st to last), `cardPointTotals`, `winnerId`, `wasAbandoned` (boolean), `completedAt`.

---

### Phase 1: Event Reception and Validation

**2.** The Ranking bounded context receives the `GameCompleted` event via asynchronous event propagation (assumed at-least-once delivery semantics).

**3.** **Idempotency check**:
- The Ranking context maintains a processed-events store keyed by `gameId`.
- If `gameId` has already been processed: acknowledge the event and skip processing. No duplicate Elo update.
- `DuplicateEventIgnored` (internal log event, not a domain event).

**4.** **Room type validation**:
- Check `roomType` from the event payload.
- If `roomType == tournament`: this game does not affect casual Elo. Acknowledge and skip.
- `NonCasualGameIgnored` (internal log event).
- Only `roomType == casual` games proceed.

**5.** **Abandonment check**:
- If `wasAbandoned == true` (all players forfeited):
    - No Elo changes are applied. This is a firm domain rule.
    - `AbandonedGameSkipped` with `gameId`, `reason = no_elo_change_for_abandoned_games`.
    - Acknowledge the event. End of flow.

**6.** **Minimum player validation**:
- Verify that the game had at least 2 non-forfeiting players who completed the game.
- If fewer than 2 non-forfeiting players: skip Elo update (this would be an edge case where all but one player forfeited, and the game technically "completed" with a default winner).
- `InsufficientPlayersForElo` with `gameId`. Acknowledge and skip.

---

### Phase 2: Elo Calculation

**7.** Extract the placement order from the `GameCompleted` event:
- 1st place (winner), 2nd place, 3rd place, ..., last place.
- Players who forfeited are placed last (ordered by forfeit time, earliest forfeit = worst placement).

**8.** Retrieve current Elo ratings for all players in the game:
- `FetchPlayerRatings` (internal query) for each `playerId` in the placements list.
- Returns current Elo rating and game count (used for K-factor determination).

**9.** **K-factor determination** for each player:
- **Policy: DetermineKFactor** -- the K-factor may vary based on player experience:
    - New players (fewer than 30 rated games): K = 40 (ratings change faster for new players).
    - Intermediate players (30--100 rated games): K = 20.
    - Established players (100+ rated games): K = 10.
    - This is a documented assumption; the exact thresholds are configurable.

**10.** For each pair of players (i, j) in the game, calculate the expected score:
- `expected_score_i = 1 / (1 + 10^((rating_j - rating_i) / 400))`
- This is the standard Elo expected score formula.

**11.** Determine actual scores based on placements:
- For each pair (i, j):
    - If player i placed higher than player j: `actual_score_i = 1`, `actual_score_j = 0`.
    - If players tied (same placement, which should not normally occur in Uno): `actual_score = 0.5` each.

**12.** Calculate the Elo delta for each player:
- For each player i, sum the deltas across all opponents j:
    - `delta_i = K_i * SUM_over_j(actual_score_ij - expected_score_ij)`
- This produces a positive delta for players who outperformed expectations and a negative delta for those who underperformed.

**13.** Apply the new ratings:
- `new_rating_i = old_rating_i + delta_i`
- Enforce a rating floor (e.g., minimum rating of 100 -- a documented assumption).
- No rating ceiling is enforced.

---

### Phase 3: Persistence and Event Emission

**14.** For each player whose rating changed:
- `UpdatePlayerRating` (internal command within the Ranking context).
- Persist the new rating, the game that caused the change, and the delta.
- `EloRatingUpdated` with `playerId`, `gameId`, `oldRating`, `newRating`, `delta`, `newGameCount`.

**15.** `EloRatingUpdated` events are emitted for downstream consumers:
- Mark the `gameId` as processed in the idempotency store.

**16.** `EloRatingUpdated` -- Read model updates (asynchronous):
- `PlayerProfileReadModel` updated with new rating and game count.
- `LeaderboardReadModel` updated -- the player's position may change.
- `PlayerMatchHistoryReadModel` updated with the Elo change for this game.
- `RatingHistoryReadModel` appended with the data point for trend visualization.

---

### Cross-Context Event Flow Summary

```
Room Gameplay Context                    Ranking Context                         Read Models
=====================                    ===============                         ===========

GameCompleted ----[async event]----->    Receive GameCompleted
                                         |
                                         +-- Idempotency check (gameId)
                                         +-- Room type filter (casual only)
                                         +-- Abandonment filter
                                         +-- Minimum players filter
                                         |
                                         v
                                         Calculate Elo deltas
                                         |
                                         v
                                         Persist new ratings
                                         |
                                         v
                                         EloRatingUpdated ----[async]----->     PlayerProfileReadModel
                                                                                LeaderboardReadModel
                                                                                RatingHistoryReadModel
```

---

### Exception Paths within Flow 3

#### E3.1: Duplicate GameCompleted Event

**17.** The same `GameCompleted` event arrives twice (due to at-least-once delivery):
- The idempotency check at step 3 catches the duplicate.
- The event is acknowledged without any state mutation.
- No double Elo update occurs.

#### E3.2: Ranking Service Temporarily Unavailable

**18.** The `GameCompleted` event cannot be processed because the Ranking context is temporarily down:
- The event remains in the message queue / event bus (not acknowledged).
- **Policy: RetryWithBackoff** -- the delivery infrastructure retries delivery with exponential backoff.
- Once the Ranking context recovers, it processes the event normally.
- Idempotency protection ensures that if the event was partially processed before the failure, it will not be double-applied.

#### E3.3: Out-of-Order Game Events

**19.** Game 3 of a match completes and its `GameCompleted` event arrives at the Ranking context before game 2's `GameCompleted` event:
- Each game's Elo update is independent. The Elo formula is applied per-game, not per-match.
- Order of processing does not affect the final rating (Elo updates are commutative at the individual game level since each game's calculation uses the player's rating at the time of processing).
- **Caveat**: if game 2 and game 3 are processed nearly simultaneously, there is a minor race condition on the rating read. The system should serialize Elo updates per player (e.g., via a per-player lock or sequential processing) to ensure consistent intermediate states.
- `EloRatingUpdated` events will reflect the correct cumulative result regardless of processing order.

#### E3.4: Abandoned Game Event

**20.** A `GameCompleted` event with `wasAbandoned == true` arrives:
- Caught at step 5.
- No rating mutation occurs.
- Event is acknowledged and marked as processed (to prevent reprocessing on retry).
- `AbandonedGameSkipped` logged for audit purposes.

#### E3.5: Player Rating Not Found

**21.** A player in the `GameCompleted` placements does not have an existing rating record (e.g., their very first game):
- **Policy: InitializeDefaultRating** -- create a new rating record with the default starting Elo (e.g., 1200) and game count of 0.
- `PlayerRatingInitialized` with `playerId`, `initialRating = 1200`.
- Proceed with normal Elo calculation using the initialized rating.

#### E3.6: Rating Calculation Produces Anomalous Result

**22.** The calculated delta would drop a player below the rating floor (e.g., 100):
- The new rating is clamped to the floor value.
- `EloRatingUpdated` reflects the clamped rating.
- `RatingFloorApplied` logged for monitoring.

---

## Cross-Flow Integration Points

The three flows above do not exist in isolation. The following integration points connect them:

| Source Flow | Event | Target Flow / Context | Action |
|---|---|---|---|
| Flow 1 (Room Completion) | `GameCompleted` | Flow 3 (Elo Update) | Triggers Elo recalculation for casual rooms |
| Flow 1 (Room Completion) | `MatchCompleted` | Flow 2 (Tournament) | Triggers advancement logic if room is a tournament room |
| Flow 2 (Tournament) | `CreateRoom` | Flow 1 (Room Lifecycle) | Creates tournament rooms that follow the same gameplay flow |
| Flow 2 (Tournament) | `TournamentCompleted` | Ranking Context | Triggers tournament placement rating updates (separate from Elo) |
| Flow 3 (Elo Update) | `EloRatingUpdated` | Spectator/Profile Context | Updates player profiles and leaderboards |

---

## Event Ordering and Consistency Guarantees

| Guarantee | Scope | Mechanism |
|---|---|---|
| Total ordering of game events | Within a single Game aggregate | Sequence numbers enforced on all commands; events appended to immutable log in order |
| Causal ordering across games in a match | Within a single Room aggregate | Match state machine ensures games are sequential |
| Eventual consistency for Elo updates | Cross-context (Room to Ranking) | At-least-once delivery with idempotency; no strict ordering required |
| Round synchronization in tournaments | Within Tournament Orchestration | `CheckRoundCompletion` policy counts completed rooms; round does not advance until all rooms report |

---

*This document represents the timeline output of EventStorming sessions for UnoArena's three major business flows. Each step corresponds to a sticky note on the EventStorming board, with commands (blue), events (orange), policies (purple), hotspots (red), and read models (green) clearly identified in the narrative.*
