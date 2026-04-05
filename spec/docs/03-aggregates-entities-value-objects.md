# 3. Aggregates, Entities, and Value Objects

> **Document purpose:** Define the complete tactical domain model for UnoArena, identifying every Aggregate Root, Entity, and Value Object within each bounded context. This document uses the ubiquitous language established in the [Domain Glossary](01-domain-glossary.md) and the bounded-context boundaries defined in the [Context Map](02-bounded-contexts-and-context-map.md). All assumption references (A1–A38) trace back to [Assumptions and Open Questions](08-assumptions-and-open-questions.md).

This document is the authoritative specification for the object-level domain model. Each aggregate defines a consistency boundary: all invariants listed for an aggregate are enforced atomically within a single command execution against that aggregate. Cross-aggregate invariants are enforced through domain events and eventual consistency (see Section 3.7). The terms Aggregate Root, Entity, and Value Object are used with their precise Domain-Driven Design meanings: an Aggregate Root is the entry point for all commands against its cluster and guarantees internal consistency; an Entity has a stable identity that persists across mutations; a Value Object is immutable, equality is based solely on its attribute values, and it carries no identity of its own.

---

## 3.1 Room Gameplay Context (RG)

The Room Gameplay context is the core domain. It owns the lifecycle of Rooms, Matches, and Games, and is the single source of truth for all gameplay state. All writes to gameplay state pass through one of two Aggregate Roots: **Room** (for room-level and match-level decisions) or **Game** (for turn-by-turn gameplay operations). The separation of Game into its own Aggregate Root is a deliberate design decision justified by the high write frequency of gameplay commands; see Section 3.8 for the full rationale.

---

### 3.1.1 Room Aggregate (Aggregate Root)

**Identity:** `roomId` — a globally unique, opaque identifier (UUID v4) assigned at creation by the Room Gameplay context.

**Description:** The Room is the top-level container for a Match and all its constituent Games. It governs room configuration, player slot management, and the overall room lifecycle. All players who wish to participate in gameplay must join a Room, and all Match-level results are recorded here. The Room is authoritative for room configuration (room type, player capacity, match format) and for the current Match's outcomes.

#### State Machine

The Room transitions through the following states in strict order:

```
Waiting --> Ready --> InProgress --> Completed
                                   ^
                                   | (if all players forfeit before
                   Abandoned <------  match conclusion)
```

| State | Meaning | Entry Condition | Exit Condition |
|-------|---------|-----------------|----------------|
| `Waiting` | Room is open; players may join or leave. No active Match or Game. | Room creation. | Minimum player count is met. |
| `Ready` | Minimum player count is satisfied; host may now issue `StartMatch`. | Minimum player count is met. | `StartMatch` command accepted; or player count drops below minimum (→ back to `Waiting`). |
| `InProgress` | At least one Game in the Match is underway or has been played. Room is locked to new entrants. | First `GameStarted` event emitted for Game 1. | Match conclusion (Match winner determined or Abandoned condition met). |
| `Completed` | Terminal state. The Match has concluded with a valid outcome. Results have been emitted. | Match completed with at least one valid Game. | None (terminal). |
| `Abandoned` | Terminal state. All remaining players forfeited before a Match winner was determined. No Elo updates are issued. | All active PlayerSlots reach forfeited status simultaneously. | None (terminal). |

Note: `Waiting` and `Ready` are sub-states of the broader "pre-game" phase. Transitions are irreversible. A Room that has reached `Completed` or `Abandoned` is immutable; no further commands are accepted.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `roomId` | `RoomId` (UUID) | Aggregate identity. |
| `roomType` | `RoomType` enum | `casual` or `tournament`. |
| `maxPlayers` | integer [2, 10] | Maximum number of Player Slots. |
| `matchFormat` | `MatchFormat` | Always `best_of_3` in the current model. |
| `status` | `RoomStatus` enum | One of: `Waiting`, `Ready`, `InProgress`, `Completed`, `Abandoned`. |
| `hostPlayerId` | `PlayerId` | The player who created the room (casual rooms only; tournament rooms set this to a system actor). |
| `tournamentId` | `TournamentId` (nullable) | Present only for tournament rooms; references the owning Tournament aggregate in the TO context. |
| `roundId` | `RoundId` (nullable) | Present only for tournament rooms; identifies the Round within the Tournament. |
| `playerSlots` | List of `PlayerSlot` | Ordered list of Player Slot entities (order determines initial turn order). Maximum length = `maxPlayers`. |
| `currentMatch` | `Match` (nullable) | The single Match entity owned by this Room. Null until `StartMatch` is accepted. |
| `roomSeed` | opaque token | Per-room HMAC secret generated at creation, used for event signatures (A31). Never exposed externally. |
| `createdAt` | timestamp | Server-assigned creation time. |
| `completedAt` | timestamp (nullable) | Server-assigned completion time. Set when status transitions to `Completed` or `Abandoned`. |

#### Invariants

- **INV-R-01:** `maxPlayers` must be in the range [2, 10] inclusive.
- **INV-R-02:** The number of occupied Player Slots must never exceed `maxPlayers`.
- **INV-R-03:** A `StartMatch` command is accepted only when: (a) `status` is `Waiting` or `Ready`, (b) the commanding player is the `hostPlayerId`, and (c) there are at least 2 occupied, non-forfeited Player Slots.
- **INV-R-04:** Once `status` is `InProgress`, no new players may join. Player Slot additions are only permitted in `Waiting` state.
- **INV-R-05:** A Player may occupy at most one Player Slot within this Room.
- **INV-R-06:** State transitions are strictly ordered and irreversible: `Waiting` → `Ready` → `InProgress` → (`Completed` | `Abandoned`). No backward transitions exist.
- **INV-R-07:** The Room may contain at most one `Match` at any time. A second Match cannot be initiated within the same Room.
- **INV-R-08:** The Room transitions to `Completed` only after the Match entity signals match completion (a player wins 2 out of 3 Games, or the last-player-standing condition is met per A17).
- **INV-R-09:** The Room transitions to `Abandoned` only when all Player Slots have status `forfeited` simultaneously, leaving no active player who could win the Match.
- **INV-R-10:** `tournamentId` and `roundId` must both be present or both be absent. A Room is either fully affiliated with a Tournament Round or entirely casual.
- **INV-R-11:** `roomType` is set at creation and is immutable. A Room's type cannot be changed after creation.
- **INV-R-12:** All commands issued to this aggregate must carry a valid `SequenceNumber` that matches the Room's expected next value for the issuing player. Commands with stale sequence numbers are rejected with HTTP 409 Conflict (see SequenceNumber Value Object, Section 3.1.5).
- **INV-R-13:** A player joining the Room must hold a valid, active session (cross-context precondition verified synchronously against the IS context before the command is processed by this aggregate).
- **INV-R-14:** A player may participate in at most one active Room at any time across the platform (A37). This invariant is enforced by the IS context and checked as a precondition before the `JoinRoom` command is applied.

#### Children

- **Match** (Entity, Section 3.1.2) — exactly one, created lazily when `StartMatch` is accepted.
- **PlayerSlot** (Entity, Section 3.1.4) — one per participating player, created on `JoinRoom`.

---

### 3.1.2 Match Entity (owned by Room)

**Identity:** `matchId` — a unique identifier scoped to the Room. Assigned by the Room when the Match is created. Not a global aggregate root; the Match is an Entity owned by the Room Aggregate.

**Description:** The Match represents the best-of-three series of Games played by all players within a single Room. The Match is the record-keeping layer above individual Games: it accumulates game results, tracks the match winner, and provides the complete data set needed for tournament advancement decisions (match wins, cumulative card points, and the completion timestamp of the final Game). The Match does not own the game state directly; each Game is a separate Aggregate Root (Section 3.1.3) referenced by its `gameId`.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `matchId` | `MatchId` (scoped to Room) | Entity identity. |
| `roomId` | `RoomId` | Reference to the owning Room aggregate. |
| `status` | `MatchStatus` enum | `in_progress` or `completed`. |
| `gameIds` | List of `GameId` | Ordered references to the Game aggregates (up to 3). |
| `gameResults` | List of `GameResult` | Ordered list of completed game outcomes (placement order, card point totals per player, completion timestamp). |
| `matchWins` | Map of `PlayerId` → integer | Number of Games won by each player. Range [0, 3]. |
| `cumulativeCardPoints` | Map of `PlayerId` → integer | Sum of card points held by each player at the end of each Game they have participated in. Lower is better (used for tiebreaking per V4). |
| `matchWinner` | `PlayerId` (nullable) | Set when the Match concludes. Null until completion. |
| `completionTime` | timestamp (nullable) | Server-assigned time of the final Game's completion. Used as tiebreaker per V4 (A16). |
| `currentGameNumber` | integer [0, 3] | 0 = no Game played yet; 1–3 = Game currently in progress or most recently completed. |

#### Invariants

- **INV-M-01:** A Match contains at most 3 Games (`gameIds.length ≤ 3`).
- **INV-M-02:** A new Game may only be initiated within the Match when the previous Game has reached `Completed` status.
- **INV-M-03:** `matchWins` for any single player must not exceed 3 (equal to the maximum number of Games).
- **INV-M-04:** In a 2-player room, the Match concludes as soon as one player reaches 2 game wins. In a multi-player room, all 3 Games are played unless the advancement outcome is mathematically determined for all 3 top-3 slots before Game 3 (working assumption per A15, A16).
- **INV-M-05:** `cumulativeCardPoints` is updated after every Game completion using the card points remaining in each non-winning player's hand at the moment of that Game's conclusion.
- **INV-M-06:** The Match may not transition to `completed` unless all Games assigned to it have first reached `Completed` status in the Game aggregate.
- **INV-M-07:** Once `status` is `completed`, all attributes of the Match are immutable. The Match result is final.
- **INV-M-08:** The `matchWinner` field is set only for rooms where "match winner" is a meaningful concept (i.e., 2-player rooms where one player reaches 2 wins, or any room where only 1 player survives through to match end). In multi-player rooms, the Match result is expressed as a ranked `PlacementOrder` across all players, not as a single winner.
- **INV-M-09:** Each `gameId` in `gameIds` must reference a distinct Game aggregate. The same Game cannot appear twice in a Match.

---

### 3.1.3 Game Aggregate (Aggregate Root)

**Identity:** `gameId` — a globally unique, opaque identifier (UUID v4) assigned at Game initialization by the Room Gameplay context.

**Description:** The Game is the central, high-frequency aggregate for all Uno gameplay. It is separated from the Room Aggregate Root because gameplay commands (card plays, draws, Uno calls, challenges) arrive at very high frequency — potentially several per second per room — and require a single-writer consistency boundary with strict sequence number enforcement. Elevating Game to a separate Aggregate Root allows it to be owned by a dedicated process (a single-writer game controller per active game) without serializing through the Room's broader state machine.

The Game aggregate owns all mutable gameplay state: the Draw Pile, the Discard Pile, all player Hands, turn order, Turn Direction, the current player pointer, the game phase, the Challenge Window state, the Turn Timer, per-player connection and forfeit status, and the monotonically increasing sequence number for the game.

All deck operations are server-authoritative (V11, A10, A11). No client ever influences shuffling, card order, or draw resolution. The full deck state (seed and order) is recorded in the Game Log before any card is dealt (V12, A11).

#### Turn State Machine

The Game's turn-level state machine operates within the broader `GamePhase`. Every transition produces a domain event before it takes effect.

```
[Game Phase: Initializing]
    |
    | DeckInitialized, InitialHandsDealt, FirstCardFlipped
    v
[Game Phase: InProgress]
    |
    |  Every turn cycles through:
    |
    |    AwaitingAction
    |        |
    |        |-- Player plays penultimate card --> ChallengeWindowOpen
    |        |       |
    |        |       |-- Challenge received (valid) --> back to AwaitingAction
    |        |       |      (challenger or challenged player penalized; turn advances)
    |        |       |
    |        |       |-- 5s elapsed with no challenge --> back to AwaitingAction
    |        |       |      (window closes; next player's turn begins)
    |        |       |
    |        |       |-- Next player begins their action --> window closes immediately
    |        |              (A8; late challenges rejected) --> back to AwaitingAction
    |        |
    |        |-- Player plays last card (empties hand) --> [Game Phase: Completed]
    |        |
    |        |-- Turn timer expires (A38) --> auto-draw if not drawn, auto-pass
    |        |      --> AwaitingAction (next player)
    |        |
    |        |-- Player disconnects --> ChallengeWindowOpen is unaffected if
    |               already open; disconnected player's subsequent turns are
    |               auto-skipped (AwaitingAction proceeds to next player)
    |               until ReconnectionWindow expires --> PlayerForfeited
    |
    |  Between Games (within Match):
    v
[Game Phase: BetweenGames]  <-- intermediate, not a turn-level state
    |  (Match entity decides whether to start Game 2 or 3)
    v
[Game Phase: InProgress]    <-- next Game begins (new Game aggregate)
    OR
[Game Phase: Completed]     <-- terminal for this Game aggregate
```

**Turn-level states in detail:**

| Turn State | Description | Entry | Exit |
|------------|-------------|-------|------|
| `AwaitingAction` | The current player must play a card, draw a card, or pass (if they already drew and cannot play). The Turn Timer is running (A38). | After `TurnAdvanced` event is emitted; after `ChallengeWindowOpen` closes. | Player plays a card, draws a card, passes, or Turn Timer expires. |
| `ChallengeWindowOpen` | A player just played their penultimate card. The 5-second challenge window is open (V10). Any opponent may issue a challenge. The next player's turn has not yet begun. | Player plays their second-to-last card (with or without a bundled Uno call per A5). | (a) 5 seconds elapse; (b) a valid challenge is received and processed; (c) the next player begins their turn (A8). |

**Game-level phases:**

| Game Phase | Description |
|------------|-------------|
| `Initializing` | Deck is being generated, shuffled, and dealt. Not yet ready for player commands. |
| `InProgress` | Active gameplay. Turn states cycle as described above. |
| `Completed` | Terminal. One player has emptied their Hand (or all remaining players forfeited per A17). Placement Order is finalized. |

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `gameId` | `GameId` (UUID) | Aggregate identity. |
| `roomId` | `RoomId` | Reference to the owning Room. |
| `matchId` | `MatchId` | Reference to the owning Match. |
| `gameNumber` | integer [1, 3] | Position of this Game in the Match sequence (1st, 2nd, or 3rd Game). |
| `gamePhase` | `GamePhase` enum | `Initializing`, `InProgress`, `Completed`. |
| `turnState` | `TurnState` enum | `AwaitingAction`, `ChallengeWindowOpen`. Meaningful only when `gamePhase` is `InProgress`. |
| `sequenceNumber` | `SequenceNumber` | Monotonically increasing counter; applied to every state-changing command. Stale commands rejected with HTTP 409. |
| `shuffleSeed` | opaque token | Server-generated cryptographic seed used to produce the deck order. Recorded in the Game Log before the deal. |
| `drawPile` | ordered List of `Card` | Face-down cards remaining to be drawn. Server-side only; never exposed to clients or spectators. |
| `discardPile` | ordered List of `Card` | Face-up played cards. The top element is the Top Card. The full list is available to the Game Log and Audit context; only the top card is public. |
| `playerHands` | Map of `PlayerId` → List of `Card` | Private. Each player's current hand. Never broadcast in full; only the card count per player is public (V14, A33). |
| `turnOrder` | ordered List of `PlayerId` | Canonical turn order. Initialized at deal time. Modified only by forfeit (forfeited player removed). |
| `turnDirection` | `TurnDirection` | `Clockwise` or `CounterClockwise`. Initialized to `Clockwise`; toggled by each Reverse card played. |
| `currentPlayerId` | `PlayerId` | The player whose turn is currently `AwaitingAction` or who triggered `ChallengeWindowOpen`. |
| `turnNumber` | integer | Monotonically increasing counter of completed turns within this Game. |
| `turnTimer` | `TurnTimer` | Deadline and duration for the current player's action window (A38: 30s casual, 60s tournament). |
| `challengeWindow` | `ChallengeWindow` (nullable) | Present only when `turnState` is `ChallengeWindowOpen`. Contains start time, deadline (start + 5s per V10), and status. |
| `unoCallStatus` | Map of `PlayerId` → boolean | Tracks whether each player with 1 card in hand has made their Uno call. Used to adjudicate challenges. |
| `connectionStatus` | Map of `PlayerId` → `ConnectionStatus` | `connected`, `disconnected_pending_reconnection`, `forfeited`. |
| `reconnectionWindows` | Map of `PlayerId` → `ReconnectionWindow` | Active reconnection windows for disconnected players. At most one per player. |
| `placementOrder` | `PlacementOrder` (nullable) | Final ranking of all players. Set only when `gamePhase` is `Completed`. |
| `cardPointTotals` | Map of `PlayerId` → `CardPoints` (nullable) | Card points remaining in each non-winner's hand at game end. Set only when `gamePhase` is `Completed`. |
| `completedAt` | timestamp (nullable) | Server-assigned time of Game completion. Used for Match tiebreaking (V4, A16). |
| `lastReshuffleSeed` | opaque token (nullable) | Seed used for the most recent Draw Pile reshuffle (A11). Null if no reshuffle has occurred. |
| `roomType` | `RoomType` | Denormalized from Room at Game creation (`casual` or `tournament`). Required so downstream consumers (RK context) can determine Elo eligibility without querying the Room. |

#### Invariants

- **INV-G-01 (Turn Enforcement):** Only the player identified by `currentPlayerId` may issue a `PlayCard`, `DrawCard`, or `Pass` command. Any command from a different player is rejected as an illegal out-of-turn action.
- **INV-G-02 (Legal Play):** A `PlayCard` command is accepted only if the card to be played satisfies at least one of: (a) the card's Color matches the Top Card's effective Color, (b) the card's Face matches the Top Card's Face, or (c) the card is a Wild or Wild Draw Four. All other plays are rejected (V11, term 31).
- **INV-G-03 (Pass Precondition):** A `Pass` command is accepted only if the current player has already drawn a card during the current turn (A12). A player may not pass without drawing first.
- **INV-G-04 (Draw-then-play restriction):** After drawing a card, the current player may play only the card just drawn (if it is legal) or must pass. Playing a different card from the original hand after drawing is rejected (A12).
- **INV-G-05 (Sequence Number Monotonicity):** Every command that mutates Game state must carry the correct next `SequenceNumber` for that player within the context of this Game. Commands with a sequence number lower than expected are rejected with HTTP 409 Conflict (V13). Commands with a higher sequence number (gap detected) are also rejected pending reconciliation.
- **INV-G-06 (Challenge Window Timing — Server Authoritative):** The Challenge Window opens at the moment the server processes the penultimate card play. The window closes upon the earliest of: (a) 5 seconds elapsed since server processing (V10, A7), (b) a valid Challenge command received and processed, or (c) the next player in turn order submitting a valid action (A8). Late challenges (received after any of these conditions) are rejected.
- **INV-G-07 (Challenge Adjudication):** A challenge is valid if and only if the challenged player played their penultimate card without having issued a Uno call (neither bundled with the card play nor as a separate command within the window per A5, A6). If the call was made, the challenger receives a Penalty Draw of 2 cards. If the call was not made, the challenged player receives a Penalty Draw of 2 cards (term 22).
- **INV-G-08 (Wild Card Color Declaration):** Every `PlayCard` command involving a Wild or Wild Draw Four card must include a `ColorDeclaration`. A Wild play without a Color Declaration is rejected. The Color Declaration becomes the effective color of the Top Card immediately upon acceptance (term 29).
- **INV-G-09 (Wild Draw Four — Bluff-Permitted Acceptance):** The server accepts all Wild Draw Four plays without validating whether the player holds cards matching the active color (A9). This permits strategic bluffing. Enforcement of the color-match rule occurs exclusively through the WDF Challenge mechanism: the affected next player may challenge within a 5-second window (term 22b). If no challenge is issued, the play stands and the affected player draws 4 cards.
- **INV-G-10 (Draw Stacking):** Draw Two and Wild Draw Four effects are not stackable. The affected next player must draw the prescribed number of cards and forfeit their turn; they may not play another Draw Two or Wild Draw Four to redirect the effect (design constraint per requirements).
- **INV-G-11 (Reconnection Window):** A player detected as disconnected (A4) receives exactly one Reconnection Window of 60 seconds (V9). During this window, their turns are auto-skipped (term 60). No extensions or pauses to the Reconnection Window are permitted. The window is a fixed domain value.
- **INV-G-12 (Reconnection Window Expiry — Turn Forfeit):** If the Reconnection Window expires while it is the disconnected player's turn (i.e., `currentPlayerId` equals the disconnected player when the window expires), an automatic Forfeit is issued (V18). The player's hand is removed from the game entirely (A36).
- **INV-G-13 (Reconnection Window Expiry — Non-Turn Forfeit):** If the Reconnection Window expires while it is another player's turn, the disconnected player is forfeited without affecting the current player's action window. Their Hand is removed (A36).
- **INV-G-14 (Minimum Active Players):** At any point during an `InProgress` Game, if the number of players with `connectionStatus` not equal to `forfeited` drops below 2, the Game concludes immediately. The sole remaining active player is declared the winner (A17). Placement order for forfeited players is assigned by remaining card point totals at forfeit time, with ties broken by turn proximity to the winner (A14).
- **INV-G-15 (Forfeited Hand Removal):** When a player forfeits, their hand is discarded from the game entirely. Forfeited cards are not reshuffled into the Draw Pile (A36). The total number of cards in active circulation decreases accordingly.
- **INV-G-16 (Turn Timer):** A Turn Timer is active for every connected player's turn. The timer deadline is set when `TurnAdvanced` is emitted. For casual rooms, the duration is 30 seconds; for tournament rooms, 60 seconds (A38). If the timer expires before the player acts: if the player has not yet drawn, the server auto-draws a card; if the drawn card is not legal or the player has already drawn, the server auto-passes. The timer is reset when the turn advances to the next player.
- **INV-G-17 (Deck Reshuffle):** When the Draw Pile is exhausted and a card must be drawn, the Discard Pile (minus the current Top Card) is shuffled using a new server-generated seed (A11). The new seed is appended to the Game Log before the reshuffle takes effect. The Top Card remains on the Discard Pile.
- **INV-G-18 (Turn Direction Reversal — Two-Player Special Case):** In a 2-player Game, a Reverse card acts identically to a Skip: the player who played the Reverse takes an immediate second turn (term 25). The `turnDirection` field is still toggled for consistency, but the observable effect is a skip of the other player.
- **INV-G-19 (First Card Special Handling):** The first card flipped from the Draw Pile at Game start is handled as follows: (a) If it is a Wild Draw Four, it is returned to the Draw Pile, the pile is reshuffled, and a new first card is drawn — this repeats until a non-Wild-Draw-Four card is revealed. (b) If it is a Wild (non-WDF4), it is placed on the Discard Pile; the host (or system for tournament rooms) must issue a `DeclareColor` command, which emits `ColorChosen` before the first Turn begins — this declaration occurs during the `Initializing` phase. Action cards (Skip, Reverse, Draw Two) as first cards take effect immediately on the first player's turn per standard Uno rules. The initial reshuffle (case a) does not require a new seed recording; it occurs during `Initializing` before any player-visible game state exists.
- **INV-G-20 (Placement Order Finalization):** When the Game reaches `Completed`, the `placementOrder` is finalized as follows: 1st place = the player who emptied their Hand; remaining positions assigned by ascending card point total; ties within the remaining positions broken by turn proximity to the winner (A14). This value is immutable once set.
- **INV-G-21 (Idempotency — Drawn Card):** For each turn, a player may draw at most one card voluntarily or via automatic draw (A12). A second `DrawCard` command within the same turn is rejected unless it is a replay of an already-processed command (detected via Idempotency Key).
- **INV-G-22 (Server-Authoritative RNG):** All card dealing, draw pile ordering, and reshuffle operations are exclusively server-generated. No client-provided seed or card-order preference is accepted at any time (V11).
- **INV-G-23 (Wild Draw Four Challenge Adjudication):** When a `ChallengeWildDrawFour` command is received within an open WDF Challenge Window, the server compares the challenged player's hand snapshot (captured at the moment the Wild Draw Four was played) against the active color at that moment. If the snapshot contains at least one card matching the active color, the outcome is `bluff_confirmed`: the challenged player (who played the WDF) draws 4 penalty cards, and the affected player draws none. If the snapshot contains no card matching the active color, the outcome is `legitimate_play`: the affected player (challenger) draws 6 cards (4 original WDF effect + 2 penalty). The hand snapshot is never revealed to other players or spectators; only the boolean outcome (`challengedPlayerHadMatchingColor`) and the penalty assignment are published.
- **INV-G-24 (WDF Challenge Window Timing — Server Authoritative):** The WDF Challenge Window opens at the moment the server processes a Wild Draw Four card play. The window closes upon the earliest of: (a) 5 seconds elapsed since server processing (A7 applies symmetrically), (b) a valid `ChallengeWildDrawFour` command received and processed, or (c) the affected player explicitly accepting the draw (issuing `DrawCard` or the window expiring). Only the affected next player (the one who would draw 4 cards) may issue a challenge within this window. Late challenges are rejected.
- **INV-G-25 (WDF Challenge Window and Uno Challenge Window Coexistence):** When a Wild Draw Four is played as a penultimate card (leaving the playing player with exactly 1 card), both a WDF Challenge Window and an Uno Challenge Window open simultaneously. They are independent: the WDF window is exclusive to the affected next player, while the Uno window is open to all opponents. Each window follows its own closure rules and resolution logic. Resolution of one does not close or affect the other.

#### Children

- `playerHands` — map of PlayerId to ordered lists of Card Value Objects (private, server-side only).
- `drawPile` — ordered list of Card Value Objects (private, server-side only).
- `discardPile` — ordered list of Card Value Objects (top card is public).
- `challengeWindow` — `ChallengeWindow` Value Object (nullable).
- `reconnectionWindows` — map of PlayerId to `ReconnectionWindow` Value Objects.
- `turnTimer` — `TurnTimer` Value Object.
- `placementOrder` — `PlacementOrder` Value Object (nullable until Completed).

---

### 3.1.4 Player Slot Entity (owned by Room)

**Identity:** `(roomId, playerId)` — the combination of the Room's identity and the Player's identity. A Player Slot is not identified independently of its Room.

**Description:** The Player Slot is the Room's record of a single participant. It persists across all Games in the Match: the same Player Slot exists from when the player joins the Room until the Room is `Completed` or `Abandoned`. While a new Hand is dealt to each player at the start of every Game (owned by the Game aggregate), the Player Slot accumulates match-level statistics (game wins, cumulative card points) that are needed for the Room's match outcome and for tournament advancement decisions.

The Player Slot is not used for in-game turn enforcement (that is the Game aggregate's responsibility). It is the Room's view of each participant.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `roomId` | `RoomId` | Reference to the owning Room aggregate. |
| `playerId` | `PlayerId` | Reference to the player (owned by the IS context; referenced here as an opaque identifier). |
| `displayName` | string | Denormalized player display name at time of joining. Used for Room-level event payloads. |
| `slotStatus` | `SlotStatus` enum | `active`, `disconnected_pending_reconnection`, `forfeited`. |
| `joinedAt` | timestamp | Server-assigned time when the player joined the Room. |
| `forfeitedAt` | timestamp (nullable) | Server-assigned time when the player was forfeited (if applicable). |
| `matchWins` | integer | Number of Games won by this player in the current Match. Incremented by the Room when a `GameCompleted` event reports this player as 1st place. |
| `cumulativeCardPoints` | integer | Sum of card points this player held in hand at the end of each completed Game (lower is better for tiebreaking per V4). |
| `lastGameCompletionTime` | timestamp (nullable) | Timestamp of the most recent Game in which this player participated. Used as the final tiebreaker (V4). |
| `turnOrderPosition` | integer | Zero-based index of this player's position in the initial turn order for the Match. Determines seating for all Games in the Match. |

#### Invariants

- **INV-PS-01:** A Player Slot may only be added to a Room when `roomStatus` is `Waiting`.
- **INV-PS-02:** A Player Slot's `playerId` must be unique within the Room (a player cannot have two slots in the same Room).
- **INV-PS-03:** `slotStatus` transitions are: `active` → `disconnected_pending_reconnection` → `active` (on reconnection) or → `forfeited` (on window expiry or explicit forfeit). Once `forfeited`, the status is terminal; the player cannot rejoin the Room.
- **INV-PS-04:** `matchWins` is incremented by the Room aggregate only upon receiving a `GameCompleted` event from the Game aggregate, where this player's position in `placementOrder` is 1st. It is never set directly by a client command.
- **INV-PS-05:** `cumulativeCardPoints` is updated by the Room aggregate upon receiving each `GameCompleted` event, by adding the card points attributed to this player in that Game's result. It is never decremented.
- **INV-PS-06:** The Player Slot record, including all accumulated statistics, is preserved even after the player forfeits. The historical data is required for match result calculation and tournament advancement evaluation.

---

### 3.1.5 Value Objects in Room Gameplay Context

Value Objects are immutable. Once created, they are never modified; a new instance is created instead. Equality between two Value Object instances is determined solely by their attribute values, not by identity.

---

#### Card

Represents a single Uno card. Immutable.

| Attribute | Type | Constraints |
|-----------|------|-------------|
| `color` | `Color` | `Red`, `Yellow`, `Green`, `Blue`, or `None` (for Wild and Wild Draw Four). |
| `face` | `Face` | See Face enum below. |

- Equality: two Cards are equal if and only if both `color` and `face` are equal.
- A card with `color = None` is a Wild-type card. Its effective color after play is determined by the `ColorDeclaration`, not stored in the Card itself (the Color Declaration is part of the event payload; the Card Value Object remains immutable).
- The standard deck (A10) contains exactly: per color — one `(color, 0)`, two each of `(color, 1)` through `(color, 9)`, two `(color, Skip)`, two `(color, Reverse)`, two `(color, DrawTwo)`; plus four `(None, Wild)` and four `(None, WildDrawFour)`.

---

#### Color

Enum. The set of valid card colors in UnoArena.

| Value | Description |
|-------|-------------|
| `Red` | Red suit. |
| `Yellow` | Yellow suit. |
| `Green` | Green suit. |
| `Blue` | Blue suit. |
| `None` | Absence of an inherent color; applies exclusively to Wild and Wild Draw Four cards. |

Note: `None` is not a declarable color. A `ColorDeclaration` must specify one of `Red`, `Yellow`, `Green`, or `Blue`.

---

#### Face

Enum. The complete set of valid card faces in UnoArena.

| Value | Category | Card Point Value |
|-------|----------|-----------------|
| `Zero` | Number | 0 |
| `One` through `Nine` | Number | 1 through 9 |
| `Skip` | Action | 20 |
| `Reverse` | Action | 20 |
| `DrawTwo` | Action | 20 |
| `Wild` | Wild | 50 |
| `WildDrawFour` | Wild | 50 |

Card Point values are used for tiebreaking (V4, term 18) and for Elo delta computation (V17).

---

#### SequenceNumber

A monotonically increasing integer representing the expected position of a command within the ordered command stream for a specific (Room, Player) or (Game, Player) pair.

| Attribute | Type | Constraints |
|-----------|------|-------------|
| `value` | non-negative integer | Starts at 0; increments by exactly 1 with each accepted command. |

- Two SequenceNumbers are equal if and only if their `value` fields are equal.
- A command is accepted only if its SequenceNumber equals the current expected value for that player-aggregate pair.
- A command with a lower SequenceNumber than expected is a Stale Command (term 67), rejected with HTTP 409 Conflict (V13).
- A command with a higher SequenceNumber indicates a gap and is also rejected pending resolution.
- SequenceNumbers are scoped per (Aggregate, Player) pair. The same player has independent SequenceNumber sequences for Room commands and for Game commands.

---

#### IdempotencyKey

An opaque, client-assigned identifier attached to each command to enable safe retries without double-processing.

| Attribute | Type | Constraints |
|-----------|------|-------------|
| `value` | string (UUID v4) | Must be unique within the scope of a single aggregate for a single player. |

- Two IdempotencyKeys are equal if and only if their `value` strings are equal (case-insensitive UUID comparison).
- If the server has already processed a command with a given IdempotencyKey, it returns the cached result of that command without re-executing the command's side effects (A2).
- IdempotencyKeys are distinct from SequenceNumbers: IdempotencyKeys prevent duplicate execution on retry; SequenceNumbers enforce command ordering (term 72).
- The server retains IdempotencyKey records for a minimum of the duration of the active game plus a grace period adequate to cover client retry windows.

---

#### TurnDirection

Enum. Represents the current rotational direction of turn progression.

| Value | Description |
|-------|-------------|
| `Clockwise` | Turns proceed in the clockwise direction through the turn order list. Initial value at Game start. |
| `CounterClockwise` | Turns proceed in the counter-clockwise direction. Activated by the first Reverse card play; toggled by each subsequent Reverse. |

- TurnDirection is a property of the Game aggregate, not of any individual player.
- Each Reverse card played flips the current TurnDirection to the opposite value.
- In a 2-player Game, Reverse acts as a Skip (term 25); the TurnDirection field is still toggled but the observable effect is the current player acting again.

---

#### PlacementOrder

An ordered, immutable list of `PlayerId` values representing the final ranking of all players within a single completed Game.

| Attribute | Type | Constraints |
|-----------|------|-------------|
| `rankings` | List of `(PlayerId, position: integer, cardPoints: integer)` | Ordered by position ascending. Position 1 = best (winner). |

- The first entry is always the player who emptied their Hand.
- Remaining positions are assigned in ascending order of card points remaining in hand (A14).
- Ties in card points are broken by turn proximity to the winner: the player whose turn would come soonest after the winner (following the final TurnDirection at game end) is ranked higher (A14).
- A PlacementOrder is created exactly once per Game, at the moment the Game transitions to `Completed`. It is immutable thereafter.
- This Value Object is consumed by the Match entity (to update `matchWins` and `cumulativeCardPoints`) and by the RK context (for Elo delta calculation, V17).

---

#### CardPoints

A computed, non-negative integer representing the total card point value of a set of cards held by a player at the end of a Game.

| Attribute | Type | Constraints |
|-----------|------|-------------|
| `value` | non-negative integer | Sum of individual card point values (see Face enum). |

- CardPoints is calculated server-side at the moment a Game ends. Clients do not provide or influence this value.
- The winner of a Game always has CardPoints = 0 (their hand is empty).
- Two CardPoints instances are equal if and only if their `value` fields are equal.
- CardPoints is used in two distinct contexts: (a) as input to the Elo delta computation for the RK context; (b) as a tournament tiebreaker (cumulative across all Games in a Match, per V4).

---

#### ChallengeWindow

Represents the server-enforced 5-second Uno challenge window (V10, A7).

| Attribute | Type | Constraints |
|-----------|------|-------------|
| `openedAt` | timestamp | Server time at which the window opened (moment of penultimate card play processing). |
| `deadline` | timestamp | Exactly `openedAt + 5 seconds`. |
| `status` | `ChallengeWindowStatus` enum | `Open`, `ClosedByTimeout`, `ClosedByChallenge`, `ClosedByNextPlayerAction`. |
| `challengedPlayerId` | `PlayerId` | The player who played the penultimate card and triggered the window. |
| `unoCallMade` | boolean | Whether the challenged player made a valid Uno call (bundled or separate per A5). |

- The window is a Value Object: once a ChallengeWindow is created, it is never modified. A new instance with an updated `status` replaces the old one in the Game aggregate's state.
- `deadline` is always exactly `openedAt + 5 seconds`. No extensions are permitted (V10, A7).
- The server evaluates window closure on every command received while the window is `Open`. The three closure conditions are mutually exclusive first-occurrence: whichever condition is detected first wins.
- A ChallengeWindow with `status = Open` and a `deadline` in the past is treated as `ClosedByTimeout` in any subsequent command evaluation.

---

#### WildDrawFourChallengeWindow

Represents the server-enforced 5-second Wild Draw Four challenge window (A9, term 22c).

| Attribute | Type | Constraints |
|-----------|------|-------------|
| `openedAt` | timestamp | Server time at which the window opened (moment of Wild Draw Four card play processing). |
| `deadline` | timestamp | Exactly `openedAt + 5 seconds`. |
| `status` | `WdfChallengeWindowStatus` enum | `Open`, `ClosedByTimeout`, `ClosedByChallenge`, `ClosedByDrawAccepted`. |
| `challengedPlayerId` | `PlayerId` | The player who played the Wild Draw Four and triggered the window. |
| `affectedPlayerId` | `PlayerId` | The next player in turn order who would draw 4 cards. This is the only player permitted to issue a challenge. |
| `activeColorAtTimeOfPlay` | `Color` enum | The active color on the discard pile immediately before the Wild Draw Four was played. Used for adjudication. |
| `challengedPlayerHandSnapshot` | `[CardIdentity]` | A server-side snapshot of the challenged player's hand at the exact moment the Wild Draw Four was played (before the card was removed). Used exclusively for adjudication; never transmitted to clients. |

- The window is a Value Object: once created, it is never modified. A new instance with an updated `status` replaces the old one in the Game aggregate's state.
- `deadline` is always exactly `openedAt + 5 seconds`. No extensions are permitted (A7 applies symmetrically).
- Only the `affectedPlayerId` may issue a `ChallengeWildDrawFour` command while the window is `Open`.
- The `challengedPlayerHandSnapshot` is private server-side data. It is never included in any event payload sent to clients or spectators. Only the adjudication result (`challengedPlayerHadMatchingColor: boolean`) is published.
- A WildDrawFourChallengeWindow with `status = Open` and a `deadline` in the past is treated as `ClosedByTimeout` in any subsequent command evaluation.

---

#### ReconnectionWindow

Represents the 60-second window during which a disconnected player may reconnect (V9, term 57).

| Attribute | Type | Constraints |
|-----------|------|-------------|
| `openedAt` | timestamp | Server time at which disconnection was detected and the window started. |
| `deadline` | timestamp | Exactly `openedAt + 60 seconds`. |
| `status` | `ReconnectionWindowStatus` enum | `Open`, `ExpiredByTimeout`, `ClosedByReconnection`. |
| `disconnectedPlayerId` | `PlayerId` | The player whose connection was lost. |

- The window is exactly 60 seconds. No extensions or pauses are permitted (V9).
- A `ClosedByReconnection` status indicates the player successfully reconnected. Their hand is restored intact (V19).
- An `ExpiredByTimeout` status triggers automatic forfeit processing in the Game aggregate (INV-G-12, INV-G-13, V18).
- At most one ReconnectionWindow exists per player per Game. Once a window is closed (by any cause), a second disconnection of the same player within the same Game results in immediate forfeit without a new window.

---

#### TurnTimer

Represents the per-turn action deadline for a connected player (A38).

| Attribute | Type | Constraints |
|-----------|------|-------------|
| `startedAt` | timestamp | Server time at which the turn began (when `TurnAdvanced` was emitted). |
| `deadline` | timestamp | `startedAt + duration`. |
| `duration` | duration | 30 seconds for casual rooms; 60 seconds for tournament rooms (A38). |
| `status` | `TurnTimerStatus` enum | `Running`, `ExpiredByAutoAction`, `ClosedByPlayerAction`. |

- The TurnTimer is replaced (new instance created) with each `TurnAdvanced` event.
- On expiry (`status = ExpiredByAutoAction`): the server automatically executes: (1) `AutoDraw` if the player has not yet drawn a card this turn; (2) `AutoPass` regardless of whether the drawn card is playable. This prevents indefinite turn stalling (A38).
- The TurnTimer does not apply to disconnected players; those players' turns are governed by the ReconnectionWindow and auto-skip mechanics (INV-G-11).
- Two TurnTimers are equal if `startedAt`, `deadline`, and `duration` are equal.

---

## 3.2 Tournament Orchestration Context (TO)

The Tournament Orchestration context manages the full lifecycle of Tournaments: registration, round creation, player-to-room assignment (Seeding), advancement evaluation, and final room creation. This context does not contain Uno rules; it treats Room results as opaque match outcomes and applies advancement logic on top of them.

---

### 3.2.1 Tournament Aggregate (Aggregate Root)

**Identity:** `tournamentId` — a globally unique, opaque identifier (UUID v4) assigned at creation by the Tournament Orchestration context.

**Description:** The Tournament is the top-level aggregate governing a complete multi-round elimination competition. It tracks all registered players, the sequence of Tournament Rounds, the current round in progress, and the overall tournament state. The Tournament Aggregate is the single authority on whether a player has advanced, been eliminated, or won the tournament.

#### State Machine

```
RegistrationOpen --> InProgress --> Completed
        |
        v
    Cancelled  (administrative terminal state; available in any non-Completed state)
```

| State | Description | Entry | Exit |
|-------|-------------|-------|------|
| `RegistrationOpen` | Players may register. No Rounds have begun. | Tournament creation. | `StartTournament` command (admin or system-triggered). |
| `InProgress` | At least one Round is underway. Registration is closed. | `StartTournament` accepted (minimum player count met, A19). | Final Room Match completes. |
| `Completed` | Terminal. Final Room has concluded. Tournament Placements are finalized. | Final Room `MatchCompleted` event processed. | None (terminal). |
| `Cancelled` | Terminal. Administrative cancellation. No further round or room creation occurs. | Administrative command. | None (terminal). |

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `tournamentId` | `TournamentId` (UUID) | Aggregate identity. |
| `name` | string | Human-readable tournament name. |
| `status` | `TournamentStatus` enum | `RegistrationOpen`, `InProgress`, `Completed`, `Cancelled`. |
| `registeredPlayerIds` | Set of `PlayerId` | All players registered for this tournament. |
| `currentRoundNumber` | integer | 0 when no rounds have started; increments with each new round. |
| `rounds` | ordered List of `TournamentRound` | All rounds created so far, ordered chronologically. |
| `tournamentBracket` | `TournamentBracket` | Read-optimized projection of player-to-room assignments across all rounds. |
| `eliminatedPlayerIds` | Set of `PlayerId` | Players who have been eliminated from the tournament. |
| `advancedPlayerIds` | Set of `PlayerId` | Players who have advanced to the current/next round (maintained incrementally). |
| `finalRoomId` | `RoomId` (nullable) | Set when the Final Room is created (≤10 players remaining). |
| `createdAt` | timestamp | Server-assigned creation time. |
| `startedAt` | timestamp (nullable) | Server-assigned time when `InProgress` state was entered. |
| `completedAt` | timestamp (nullable) | Server-assigned time when `Completed` state was entered. |

#### Invariants

- **INV-T-01:** A Tournament requires at least 4 registered players before it can transition to `InProgress` (A19).
- **INV-T-02:** A new Tournament Round may only be created after all Tournament Rooms in the previous Round have reached `Completed` status. The round advancement gate is enforced by the Tournament aggregate, not by individual rooms (V16).
- **INV-T-03:** The `currentRoundNumber` increments by exactly 1 with each new Round. It never decreases.
- **INV-T-04:** A player in `eliminatedPlayerIds` cannot appear in any subsequently created Tournament Round's room assignments.
- **INV-T-05:** When the number of `advancedPlayerIds` (players advancing to the next round) drops to 10 or fewer, the next action must be Final Room creation, not a standard round (V16).
- **INV-T-06:** Exactly one Final Room may exist per Tournament. Once `finalRoomId` is set, no further standard rounds are created.
- **INV-T-07:** The Tournament transitions to `Completed` only upon receiving confirmation that the Final Room's Match has concluded (a `MatchCompleted` event referencing `finalRoomId`).
- **INV-T-08:** A `Cancelled` tournament emits no further room assignments and does not process any subsequent `MatchCompleted` or `PlayerForfeited` events for advancement purposes.
- **INV-T-09:** A player may be registered in at most one active (non-Completed, non-Cancelled) tournament at a time. This invariant is enforced at registration time.
- **INV-T-10:** Registration is only accepted when `status` is `RegistrationOpen`. `JoinTournament` commands issued after the tournament has started are rejected.
- **INV-T-11:** The set of players in `registeredPlayerIds` is fixed at the moment `StartTournament` is accepted. Late registrations after this point are not permitted.

#### Children

- `TournamentRound` entities — one per round (Section 3.2.2).
- `TournamentBracket` entity — one per tournament (Section 3.2.5).

---

### 3.2.2 Tournament Round Entity (owned by Tournament)

**Identity:** `(tournamentId, roundNumber)` — the combination of the Tournament's identity and the round's sequential number. Not an Aggregate Root; owned by the Tournament aggregate.

**Description:** A Tournament Round represents one elimination tier. It owns the room assignments for that tier, tracks which rooms have completed, and provides the data needed for the Tournament aggregate to evaluate advancement and trigger the next round or the Final Room.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `tournamentId` | `TournamentId` | Reference to the owning Tournament aggregate. |
| `roundNumber` | integer | Sequential round number within the tournament (1-based). |
| `status` | `RoundStatus` enum | `in_progress`, `completed`. |
| `roomAssignments` | List of `RoomAssignment` | One per room in this round. Immutable once the round begins. |
| `completedRoomIds` | Set of `RoomId` | Rooms for which a `RoomCompleted` event has been received and processed. |
| `advancementResults` | List of `AdvancementResult` | One per completed room. Populated as rooms complete. |
| `startedAt` | timestamp | Server time when the round began (first room was assigned). |
| `completedAt` | timestamp (nullable) | Server time when the last room in the round completed. |

#### Completion Detection Logic

A Tournament Round transitions to `completed` status when and only when `completedRoomIds.size() == roomAssignments.size()`. At that moment, the Tournament aggregate evaluates all `advancementResults` to determine which players advance to the next round and which are eliminated.

#### Invariants

- **INV-TR-01:** Advancement evaluation is triggered only when ALL rooms in the round have completed. Partial round results are never used to advance players (term 52, INV-T-02).
- **INV-TR-02:** A room may not be added to `roomAssignments` after the round has started. Room assignments are immutable once the round begins.
- **INV-TR-03:** Each `RoomId` in `roomAssignments` must be unique within the round. The same Room cannot appear twice in the same round.
- **INV-TR-04:** `completedRoomIds` is a subset of `{roomId | roomId in roomAssignments}`. A Room not part of this round cannot be marked as completed here.
- **INV-TR-05:** The Round transitions to `completed` at most once. Once `status` is `completed`, it is immutable.
- **INV-TR-06:** `advancementResults` must contain exactly one entry per entry in `completedRoomIds`. Results are appended as rooms complete, not in bulk at round end.

---

### 3.2.3 Room Assignment Value Object

Represents the pairing of a set of players to a Room within a specific Tournament Round. Immutable once created.

| Attribute | Type | Description |
|-----------|------|-------------|
| `tournamentId` | `TournamentId` | Identity of the Tournament this assignment belongs to. |
| `roundNumber` | integer | The round in which this assignment applies. |
| `roomId` | `RoomId` | The Room created for this assignment. |
| `assignedPlayerIds` | List of `PlayerId` | Ordered list of players assigned to this room (order determines initial turn order). |
| `createdAt` | timestamp | Server time at which the room assignment was created. |

- Equality is based on all attribute values.
- `assignedPlayerIds` contains between 2 and 10 entries (V1).
- Room sizes within a round are balanced as evenly as possible (A20, A20a): sizes differ by at most 1 player.
- This Value Object is published as part of the `TournamentRoomAssigned` domain event, which the Room Gameplay context consumes to create the actual Room aggregate.

---

### 3.2.4 Advancement Result Value Object

Represents the outcome of a single Room's Match within a tournament round, specifically the players who advance and those who are eliminated. Immutable.

| Attribute | Type | Description |
|-----------|------|-------------|
| `roomId` | `RoomId` | The room whose Match produced this result. |
| `tournamentId` | `TournamentId` | The tournament this result applies to. |
| `roundNumber` | integer | The round in which the Match was played. |
| `advancingPlayerIds` | List of `PlayerId` | Ordered list of up to 3 players who advance (top 3 by match wins; V3). |
| `eliminatedPlayerIds` | List of `PlayerId` | Players who do not advance. |
| `tiebreakUsed` | `TiebreakLevel` enum | `None`, `CardPoints`, `CompletionTime` — indicates which tiebreak level was required. |
| `matchWinsAtAdvancement` | Map of `PlayerId` → integer | Match win counts for all players at the time of advancement evaluation. |
| `cardPointTotalsAtAdvancement` | Map of `PlayerId` → integer | Cumulative card point totals used for tiebreaking (if applicable). |
| `completionTimesAtAdvancement` | Map of `PlayerId` → timestamp | Final Game completion timestamps used for tiebreaking (if applicable). |

- Equality is based on all attribute values.
- `advancingPlayerIds` contains at most 3 entries (V3). It may contain fewer than 3 if the room had fewer than 3 active players at Match conclusion.
- Tiebreak evaluation proceeds in strict order: (1) `matchWins` descending, (2) `cumulativeCardPoints` ascending (lower is better, V4), (3) earliest `completionTime` (V4).
- This Value Object is produced by the Tournament aggregate's advancement logic upon receiving all `RoomCompleted` results for a round.

---

### 3.2.5 Tournament Bracket Entity (read-optimized, owned by Tournament)

**Identity:** `tournamentId` — same identity as the owning Tournament aggregate.

**Description:** The Tournament Bracket is a read-optimized Entity owned by the Tournament aggregate. It maps players to their room assignments across all rounds and tracks who advanced from where. The Bracket is not used for write decisions (advancement logic is computed directly from `TournamentRound.advancementResults`); it exists solely to support read model projections — specifically, the bracket visualization consumed by clients and the analytics/audit contexts.

The Bracket is updated as a side effect of every Tournament Round advancement event. It is never the authoritative source for advancement decisions; it is a derived, denormalized structure.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `tournamentId` | `TournamentId` | Entity identity. |
| `playerTrajectories` | Map of `PlayerId` → List of `(roundNumber, roomId, advancementStatus)` | Per-player path through the tournament. |
| `roundSummaries` | Map of `roundNumber` → `RoundSummary` | Per-round: room assignments, completing players, advancing players. |
| `lastUpdatedAt` | timestamp | Server time of the most recent update to the bracket. |

#### Invariants

- **INV-TB-01:** The Bracket is never used to enforce advancement decisions. It is a projection only. Any discrepancy between the Bracket and the `TournamentRound.advancementResults` is resolved in favor of `advancementResults`.
- **INV-TB-02:** The Bracket is append-only from a logical perspective: player trajectories are extended (new rounds appended) but never revised retroactively.
- **INV-TB-03:** The Bracket is updated atomically with each round completion event to ensure it is internally consistent (no partial-round states in the Bracket at any observable point).

---

## 3.3 Identity & Session Context (IS)

The Identity & Session context is the authoritative source for player identity and session validity. It is a pure upstream context: all other contexts depend on it for player authentication, but it takes no dependencies on other contexts.

---

### 3.3.1 Player Identity Aggregate (Aggregate Root)

**Identity:** `playerId` — a globally unique, opaque identifier (UUID v4) assigned at registration. This identifier is the cross-context reference for "a player" in all other bounded contexts.

**Description:** The Player Identity aggregate owns all information about a registered user: credentials, display name, account state, and registration metadata. It is the gate through which all players enter the UnoArena platform. No player can participate in a Room or Tournament without first being represented by an active Player Identity aggregate.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `playerId` | `PlayerId` (UUID) | Aggregate identity. |
| `displayName` | string | Unique, human-readable player name displayed across the platform. |
| `credentialHash` | opaque token | Cryptographic hash of the player's authentication credential. Never stored in plaintext. |
| `accountStatus` | `AccountStatus` enum | `active`, `suspended`, `banned`. |
| `registeredAt` | timestamp | Server-assigned time of registration. |
| `lastLoginAt` | timestamp (nullable) | Server-assigned time of the player's most recent successful login. |

#### Invariants

- **INV-PI-01 (Unique Display Name):** `displayName` must be unique across all Player Identity aggregates in the platform. A `RegisterPlayer` or `ChangeDisplayName` command is rejected if the requested name is already held by an existing, non-banned player.
- **INV-PI-02 (Active Account for Session):** A new Session may only be established for a Player Identity whose `accountStatus` is `active`. Attempts to log in with a `suspended` or `banned` account are rejected before a Session is created.
- **INV-PI-03 (Credential Integrity):** Credential verification is performed by comparing the provided credential against `credentialHash` using a secure, non-reversible comparison. The plaintext credential is never persisted or logged.
- **INV-PI-04 (Immutable PlayerId):** `playerId` is assigned at registration and is never changed, regardless of display name changes or account state transitions.
- **INV-PI-05 (Account Status Transitions):** Transitions are: `active` → `suspended` (administrative), `active` → `banned` (administrative), `suspended` → `active` (administrative reinstatement), `banned` → (no reinstatement). A `banned` account is a terminal state.

---

### 3.3.2 Session Aggregate (Aggregate Root)

**Identity:** `sessionId` — a globally unique, opaque, cryptographically random identifier assigned at session creation. This identifier is the bearer token used by clients to authenticate commands.

**Description:** The Session aggregate represents a single authenticated interaction window between a player and the UnoArena platform. The single-active-session invariant is the most critical invariant in this context: at most one Session may be in `active` state for any given `playerId` at any point in time. This invariant is enforced atomically during session creation.

#### State Machine

```
Active --> Invalidated
  |
  v
Expired  (if wall-clock time exceeds expiresAt without explicit invalidation)
```

| State | Description | Entry | Exit |
|-------|-------------|-------|------|
| `Active` | The session is valid. Commands bearing this `sessionId` are authenticated. | Session creation. | A new session is created for the same `playerId` (atomically invalidated), or the player logs out, or the session expires, or administrative action. |
| `Invalidated` | The session has been explicitly terminated. Commands bearing this `sessionId` are rejected. | New login for same `playerId`, explicit logout, or administrative action. | None (terminal). |
| `Expired` | The session's `expiresAt` timestamp has passed. The session is treated as `Invalidated`. | `expiresAt` exceeded without prior invalidation. | None (terminal). |

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `sessionId` | `SessionId` (opaque random token) | Aggregate identity. |
| `playerId` | `PlayerId` | The player this session belongs to. |
| `status` | `SessionStatus` enum | `Active`, `Invalidated`, `Expired`. |
| `createdAt` | timestamp | Server-assigned time of session creation. |
| `expiresAt` | timestamp | `createdAt` + configurable session duration (assumed 24 hours per A29). |
| `invalidatedAt` | timestamp (nullable) | Server-assigned time of invalidation. Null if the session expired naturally. |
| `invalidationReason` | `InvalidationReason` enum (nullable) | `NewLogin`, `ExplicitLogout`, `AdminAction`. Null if expired naturally. |
| `deviceFingerprint` | opaque string (nullable) | Optional device identifier for session attribution. Not used for security enforcement; informational only. |

#### Invariants

- **INV-S-01 (Single Active Session — Core Invariant):** For any given `playerId`, at most one Session may be in `Active` state at any time (term 55, V8). This invariant is enforced atomically: when a new Session is created for `playerId` X, any prior `Active` Session for X is atomically invalidated (set to `Invalidated` with `invalidationReason = NewLogin`) before the new Session is confirmed as `Active`. These two operations (invalidate old, create new) are executed within a single atomic transaction in the IS context. There is no window during which both sessions could be simultaneously `Active`.
- **INV-S-02 (Session Validity Gate):** All contexts that accept commands from clients must validate the session token before processing any command. A command bearing an `Invalidated` or `Expired` session token is rejected unconditionally, regardless of the command's content.
- **INV-S-03 (Account Status Gate):** A Session is created only for a Player Identity with `accountStatus = active` (INV-PI-02). If a player's account is suspended or banned after session creation, the existing session must be immediately invalidated by the IS context (emitting `SessionInvalidated`).
- **INV-S-04 (Immutable Session Attributes):** `playerId`, `createdAt`, `expiresAt`, and `sessionId` are set at creation and never modified. The only mutable fields are `status`, `invalidatedAt`, and `invalidationReason`.
- **INV-S-05 (No Reactivation):** A Session that has reached `Invalidated` or `Expired` status cannot be reactivated. A new Session must be created.
- **INV-S-06 (Expiry Enforcement):** A Session with `status = Active` but with `expiresAt` in the past is treated as `Expired` in all downstream evaluations, even before an explicit expiry check event is processed by the IS context.

---

## 3.4 Ranking & Statistics Context (RK)

The Ranking & Statistics context owns the Elo rating system for casual play and all persistent player statistics. It is a downstream conformist to the Room Gameplay context: it consumes `GameCompleted` events and applies Elo and statistics updates. It never initiates commands toward other contexts.

---

### 3.4.1 Player Rating Aggregate (Aggregate Root)

**Identity:** `playerId` — the same `PlayerId` assigned by the IS context. The RK context uses this identifier to build its own aggregate, keyed on the same player identity. No shared kernel exists; the `playerId` is a Published Language identifier.

**Description:** The Player Rating aggregate is the authoritative record of a single player's Elo rating history in UnoArena casual play. It enforces idempotent processing of `GameCompleted` events (the same game never affects Elo twice) and maintains a full history of rating deltas for auditability.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `playerId` | `PlayerId` | Aggregate identity. |
| `currentElo` | integer | Current Elo rating. Starts at 1200 (A26). Minimum 100 (A27). |
| `gamesRated` | integer | Count of casual, non-abandoned, completed Games that have contributed to this rating. |
| `ratingHistory` | ordered List of `RatingDelta` | Chronological list of all Elo changes. Each entry: `gameId`, `delta` (signed integer), `eloBeforeUpdate`, `eloAfterUpdate`, `appliedAt`. |
| `processedGameIds` | Set of `GameId` | Idempotency guard: set of all `gameId` values whose `GameCompleted` events have already been processed. |
| `tournamentPlacementRating` | integer | Separate TPR metric (V6, A28). Distinct from Elo; updated per tournament completion, not per game. Placeholder value; formula TBD (A28). |

#### Invariants

- **INV-PR-01 (Casual Games Only):** Elo updates are applied only for Games in `casual` rooms. `GameCompleted` events from `tournament` rooms are consumed for statistics tracking but do not affect `currentElo` (V5, term 61).
- **INV-PR-02 (Non-Abandoned Games Only):** `GameCompleted` events where the game is marked `abandoned` (no natural winner; all players forfeited) do not affect `currentElo` (V7, term 40).
- **INV-PR-03 (Idempotency — Core):** If a `gameId` already exists in `processedGameIds`, any subsequent `GameCompleted` event for that same `gameId` is silently discarded. No second Elo update is applied. This prevents double-processing under at-least-once delivery (A1, term 72).
- **INV-PR-04 (Elo Rating Floor):** `currentElo` must never drop below 100. If an Elo delta would bring the rating below 100, the rating is set to exactly 100 and the delta is truncated accordingly (A27).
- **INV-PR-05 (Elo K-Factor):** The K-factor used in Elo calculations is 32 for all players (A24). Delta computation uses the standard multi-player Elo extension: each player's delta is the average of N-1 pairwise expected-vs-actual comparisons against every other player in the room (A25).
- **INV-PR-06 (Starting Elo):** Newly registered players begin with `currentElo = 1200` (A26). This is set when the Player Rating aggregate is first created (triggered by `PlayerRegistered` event from the IS context).
- **INV-PR-07 (Rating History Append-Only):** `ratingHistory` is append-only. No entry may be removed or modified after it has been appended. This ensures the full audit trail of rating changes is preserved.
- **INV-PR-08 (TPR Isolation):** `tournamentPlacementRating` is never modified by any game-level event. It is only updated in response to `TournamentCompleted` events from the TO context. Elo and TPR are completely independent (V6).

---

### 3.4.2 Player Statistics Entity (owned by Player Rating)

**Identity:** `playerId` — same as the owning Player Rating aggregate.

**Description:** The Player Statistics entity accumulates all game-play performance metrics for a single player, across both casual and tournament rooms. Unlike the Player Rating aggregate (which tracks Elo and is restricted to casual games), Player Statistics records data from all game types. This entity is owned by the Player Rating aggregate for consistency (both are updated in response to the same `GameCompleted` events), though it could be separated in a future model if write contention becomes an issue.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `playerId` | `PlayerId` | Entity identity (same as owning aggregate). |
| `totalGamesPlayed` | integer | Count of all Games in which this player participated (casual and tournament). |
| `gamesWon` | integer | Count of Games in which this player finished 1st (emptied hand). |
| `gamesLost` | integer | Count of Games completed with a non-1st placement. |
| `gamesForfeited` | integer | Count of Games in which this player forfeited (explicitly or via disconnection timeout). |
| `gamesAbandoned` | integer | Count of Games this player participated in that were concluded as Abandoned (all players forfeited). |
| `averagePlacement` | float | Rolling average of `position` values from all non-forfeited, non-abandoned PlacementOrders. |
| `totalCardPointsAccumulated` | integer | Sum of all CardPoints held at the end of every non-winning game. Lower = better average performance. |
| `tournamentGamesPlayed` | integer | Subset of `totalGamesPlayed` for tournament rooms only. |
| `casualGamesPlayed` | integer | Subset of `totalGamesPlayed` for casual rooms only. |
| `currentWinStreak` | integer | Consecutive Games won (1st place) without interruption. Resets on any non-1st outcome. |
| `longestWinStreak` | integer | Historical maximum of `currentWinStreak`. |
| `lastUpdatedAt` | timestamp | Server time of the most recent statistics update. |

#### Invariants

- **INV-PST-01:** `totalGamesPlayed = gamesWon + gamesLost + gamesForfeited + gamesAbandoned`. This accounting identity must hold at all times.
- **INV-PST-02:** Statistics are updated only in response to `GameCompleted` events (and `PlayerForfeited` events for forfeit counts). They are never set directly by client commands.
- **INV-PST-03:** Statistics updates are idempotent, using the same `processedGameIds` guard as the Player Rating aggregate (INV-PR-03). The same `gameId` is not counted twice.
- **INV-PST-04:** `averagePlacement` is recomputed on each update as `(sum of all placement positions) / (totalGamesPlayed - gamesForfeited - gamesAbandoned)`. Games where the player forfeited (no clean placement) and abandoned games are excluded from the placement average.

---

## 3.5 Spectator View Context (SV)

The Spectator View context maintains a read-optimized projection of active Games for consumption by Spectators. It receives domain events from the Room Gameplay context through an Anti-Corruption Layer (ACL) that enforces strict privacy filtering. This context never writes back to any other context and never influences gameplay state.

---

### 3.5.1 Spectator Game Projection Aggregate

**Identity:** `gameId` — same `GameId` as the source Game aggregate in the RG context. Within the SV context, this is the projection's identity, not an aggregate relationship to the RG aggregate.

**Description:** The Spectator Game Projection is a read-only, eventually consistent projection derived from Room Gameplay events after ACL transformation. It contains only the Public Game State (term 77): information safe for any observer to see. All private data (hand contents, draw pile contents, deck seed) is explicitly excluded by the ACL before events reach this context.

The Spectator Game Projection is an Aggregate Root within the SV context because it is the consistency unit for spectator updates: each update to the projection is applied atomically, ensuring Spectators never observe a partially-updated state.

#### Attributes (Public Data Only)

| Attribute | Type | Description |
|-----------|------|-------------|
| `gameId` | `GameId` | Projection identity. |
| `roomId` | `RoomId` | Reference to the source Room. |
| `gamePhase` | `GamePhase` enum | `InProgress` or `Completed`. `Initializing` phase is not projected (no spectator-relevant state yet). |
| `playerProfiles` | List of `(PlayerId, displayName, cardCount: integer, connectionStatus)` | Public player info. `cardCount` is the number of cards in hand (integer only — no card identities). |
| `topCard` | `Card` | The current Top Card of the Discard Pile (publicly visible, term 44). |
| `discardPileSize` | integer | Number of cards in the Discard Pile (for display purposes). Card identities within the pile are not projected. |
| `currentTurnPlayerId` | `PlayerId` | The player whose turn is currently active. |
| `turnDirection` | `TurnDirection` | `Clockwise` or `CounterClockwise`. |
| `matchScore` | Map of `PlayerId` → integer | Number of Games won by each player within the current Match (public, A33). |
| `gameNumber` | integer | Which Game in the Match this is (1, 2, or 3). |
| `unoCallStatus` | Map of `PlayerId` → boolean | Whether each player with 1 card has made their Uno call. Visible per A33. |
| `lastEventApplied` | `EventId` | Identity of the most recent RG event applied to this projection. Used for ordering and gap detection. |
| `projectionUpdatedAt` | timestamp | Server time of the most recent update to this projection. |

#### Explicitly Excluded Data

The following data is NEVER present in the Spectator Game Projection and is actively stripped by the ACL:

| Excluded Data | Reason |
|---------------|--------|
| Player hand contents (specific card identities in any player's hand) | Core privacy invariant (term 78). Spectators may never see private hands. |
| Draw Pile contents and ordering | Would expose future game state; equivalent to knowing all players' upcoming draws. |
| Deck shuffle seed and RNG state | Would allow deterministic reconstruction of the full deck, revealing all hands (V11, V14). |
| Card identity drawn on `CardDrawn` events | Stripped by the ACL; only the resulting card count change is projected (doc 02, Section 2.1.5.2). |
| Sequence numbers (internal concurrency control) | Implementation detail of RG context; irrelevant to spectators. |
| Event signatures and HMAC tokens | Security artifacts belonging to the Audit context. |
| Idempotency keys | Client retry infrastructure; irrelevant to spectators. |
| Reconnection window deadlines (exact timestamps) | Only connection status (`connected`/`disconnected`) is exposed, not timing details. |

#### Invariants

- **INV-SGP-01 (Privacy Hard Boundary):** No player hand contents may appear in this projection at any time, under any circumstances. This is a hard security invariant, not a UI preference (term 78, V14).
- **INV-SGP-02 (Causal Ordering):** Events applied to this projection must be applied in the same causal order as they occurred in the source Game aggregate. Out-of-order application is detected via `lastEventApplied` and corrected through event replay before the projection is updated.
- **INV-SGP-03 (Read-Only):** This aggregate accepts no commands from clients. It is updated exclusively by consuming ACL-transformed events from the RG context. No write-back to any other context occurs.
- **INV-SGP-04 (Derived, Not Authoritative):** This projection is derived from the authoritative RG aggregate. Any inconsistency between the projection and the RG aggregate is resolved by replaying RG events; the projection is rebuilt if necessary.
- **INV-SGP-05 (Eventual Consistency):** The projection is eventually consistent with the authoritative RG Game aggregate. A slight built-in delay is acceptable and serves as a privacy buffer (A35).

---

## 3.6 Audit & Game Log Context (AL)

The Audit & Game Log context is a universal downstream conformist. It consumes all domain events from all other contexts and appends them to immutable, append-only logs. It never initiates commands and never modifies existing records.

---

### 3.6.1 Game Log Entity

**Identity:** `gameId` — references the Game aggregate in the RG context. Within the AL context, this is the log's identity.

**Description:** The Game Log is an append-only, immutable, ordered sequence of all domain events pertaining to a single Game. It contains the complete, unfiltered game state history — including private data (card identities in hands, deck seed, draw pile state) — for the purposes of full game replay and dispute resolution. The Game Log is the source of truth for "what happened in a game" at the event level (term 70).

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `gameId` | `GameId` | Log identity. |
| `entries` | ordered List of `GameLogEntry` | Append-only list of events, in causal order. |
| `entryCount` | integer | Current number of entries. Monotonically increasing. |
| `firstEntryAt` | timestamp | Server time of the first entry (Game initialization). |
| `lastEntryAt` | timestamp | Server time of the most recent entry. |
| `sealed` | boolean | Set to `true` when the Game reaches `Completed` or `Abandoned` and no further entries are expected. |

**Game Log Entry structure:**

Each entry in the `entries` list contains:

| Field | Type | Description |
|-------|------|-------------|
| `entryId` | UUID | Unique identifier for this log entry. |
| `sequencePosition` | integer | Monotonically increasing position within this Game Log (1-based). |
| `timestamp` | timestamp | Server time of the event. |
| `eventId` | `EventId` | Identity of the source domain event. |
| `eventType` | string | Canonical name of the domain event (e.g., `CardPlayed`, `PlayerForfeited`). |
| `causationId` | `EventId` | Identity of the event or command that caused this event. |
| `correlationId` | UUID | Business operation correlation identifier (links events that are part of the same logical operation). |
| `sourceContext` | string | Always `RoomGameplay` for Game Log entries. |
| `payload` | structured data | Full, unfiltered event payload. For `CardDrawn`, this includes the card identity drawn (private data not exposed to spectators). For `DeckInitialized`, this includes the shuffle seed. |
| `signature` | opaque token | HMAC signature over (`entryId`, `sequencePosition`, `timestamp`, `payload`) using the per-room HMAC secret (A31). Used for tamper detection. |

#### Invariants

- **INV-GL-01 (Append-Only):** No entry in `entries` may be modified or deleted after being appended. This is enforced at the storage layer and verified by the event signature chain.
- **INV-GL-02 (Causal Completeness):** The Game Log must contain every state mutation that occurred in the Game, in causal order, sufficient to deterministically replay the entire Game from the initial `shuffleSeed` (V12). No event may be omitted.
- **INV-GL-03 (Pre-Broadcast Recording):** Events are appended to the Game Log before being broadcast to any downstream consumer (V12). An event that is not yet in the Game Log is not authoritative.
- **INV-GL-04 (Signature Chain):** Each entry's `signature` is verified upon receipt by the Audit context. A signature failure indicates tampering and must raise an alert; the entry is still appended with a tamper flag, but operational alerts are triggered.
- **INV-GL-05 (Full Private Data):** Unlike the Spectator View, the Game Log records the complete, unfiltered payload for every event, including card identities drawn, hand contents at game end, and deck seed. Access to the Game Log is restricted to authorized audit and dispute resolution processes.
- **INV-GL-06 (Sequential Position Integrity):** `sequencePosition` is a gapless, monotonically increasing integer starting at 1. A gap in `sequencePosition` values indicates a missing entry and must trigger an alert.
- **INV-GL-07 (Sealed After Completion):** Once `sealed = true`, no further entries may be appended. The `sealed` flag is set when the Game reaches `Completed` or `Abandoned` and a final `GameCompleted` or `GameAbandoned` entry has been recorded.

---

### 3.6.2 Audit Trail Entity

**Identity:** `(context, aggregateId)` — a composite identity representing all events from a specific aggregate in a specific bounded context.

**Description:** The Audit Trail is a cross-context, append-only log covering all domain events from all bounded contexts. Unlike the Game Log (which is scoped to a single Game and contains full private data), the Audit Trail is indexed for cross-context query: by context, by aggregate, by timestamp, and by correlation ID. It is the primary tool for forensic analysis, compliance reporting, and tracing the full causal chain of a business operation across context boundaries.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `context` | string | Source bounded context abbreviation (`RG`, `TO`, `RK`, `IS`, `SV`, `AL`). |
| `aggregateId` | UUID | Identity of the aggregate that produced the event. |
| `entries` | ordered List of `AuditEntry` | Append-only list, ordered by `timestamp` within this (context, aggregateId) scope. |

**Audit Entry structure:**

Each entry contains:

| Field | Type | Description |
|-------|------|-------------|
| `auditEntryId` | UUID | Unique identifier for this audit entry. |
| `eventId` | `EventId` | Identity of the source domain event (same as in the originating context's event envelope). |
| `eventType` | string | Canonical event name. |
| `sourceContext` | string | Bounded context abbreviation. |
| `aggregateId` | UUID | Source aggregate identity. |
| `timestamp` | timestamp | Server time of the event. |
| `correlationId` | UUID | Cross-context correlation identifier. |
| `causationId` | `EventId` | Identity of the event or command that caused this event. |
| `payload` | structured data | Full event payload as received. May include private data for RG events (access-controlled). |
| `signature` | opaque token (nullable) | Cryptographic signature for signed events (game state changes, Elo updates, advancement decisions per A31). |

#### Indexes

The Audit Trail is indexed for efficient retrieval by:

- `(context, aggregateId, timestamp)` — all events for a specific aggregate in a specific context, chronologically ordered.
- `(context, timestamp)` — all events from a specific context across all aggregates, time-ordered.
- `(correlationId)` — all events across all contexts that share the same correlation ID (traces a single business operation end-to-end).
- `(aggregateId)` — cross-context events for the same logical aggregate identifier (e.g., all events referencing a specific `gameId`, regardless of which context produced them).

#### Invariants

- **INV-AT-01 (Append-Only):** No Audit Trail entry may be modified or deleted after being appended. This is an absolute constraint, enforced at the storage and application layers.
- **INV-AT-02 (Universal Coverage):** Every domain event emitted by every bounded context must be appended to the Audit Trail before the event is considered successfully published. An event not in the Audit Trail is not considered durably delivered.
- **INV-AT-03 (Standard Envelope):** Every entry must include `eventId`, `timestamp`, `sourceContext`, `correlationId`, `causationId`, and `payload`. Entries missing any of these mandatory fields are flagged as malformed and trigger an operational alert.
- **INV-AT-04 (Signature Verification):** For signed events (game state changes, Elo updates, tournament advancement decisions per A31), the Audit context verifies the signature upon receipt. Verification failure triggers an alert. The entry is recorded regardless (with a tamper flag) to preserve evidence.
- **INV-AT-05 (Retention Policy):** Tournament game logs and audit entries are retained indefinitely (A32). Casual game logs are retained for a minimum of 90 days (A32). Retention is enforced at the operational level; the domain model treats all entries as immutable within their retention period.

---

## 3.7 Consistency Boundaries Summary

This section summarizes the consistency guarantees that apply within and across aggregate boundaries, and the rationale for each choice.

| Aggregate or Relationship | Consistency Model | Reason |
|---------------------------|-------------------|--------|
| Within Game aggregate | **Strong (linearizable)** | Turn enforcement, legal play validation, and sequence number monotonicity require that all commands are serialized through a single writer with no concurrent mutations. A single-process game controller owns each active Game aggregate. Sequence numbers provide the client-side guarantee that no stale command is accepted. |
| Within Room aggregate | **Strong (linearizable)** | Player Slot management (joining, forfeiting), Match state transitions, and room lifecycle state changes must be consistent. Multiple concurrent `JoinRoom` commands for the same Room must be serialized to avoid exceeding `maxPlayers`. |
| Within Session aggregate (IS) | **Strong (serializable)** | The single-active-session invariant (INV-S-01) requires that old-session invalidation and new-session creation are executed atomically within a single serializable transaction. Any weaker isolation could allow two simultaneously `Active` sessions for the same player. |
| Within Player Identity aggregate (IS) | **Strong** | Display name uniqueness (INV-PI-01) requires a serializable check-and-set on registration and name-change commands. |
| Within Player Rating aggregate (RK) | **Strong (per aggregate)** | Idempotency guard (`processedGameIds`) and rating floor enforcement require that each `GameCompleted` event is applied atomically. Multiple events for the same player can be processed sequentially; cross-player Elo updates are independent. |
| Within Tournament aggregate (TO) | **Strong** | Round advancement gate (INV-TR-01, INV-T-02) requires a consistent view of all room completion statuses before triggering advancement. Concurrent `RoomCompleted` events must be serialized through the Tournament aggregate. |
| Game (RG) → Player Rating (RK) | **Eventual** | Elo updates are applied asynchronously after `GameCompleted` events are consumed by the RK context. There is a window (typically milliseconds to seconds under normal load) during which the Game is complete but the Elo has not yet been updated. This is acceptable: players expect near-real-time but not instantaneous rating updates. Idempotency (INV-PR-03) ensures correctness despite at-least-once delivery (A1). |
| Game (RG) → Tournament (TO) | **Eventual** | Tournament advancement is triggered by `MatchCompleted` and `RoomCompleted` events consumed asynchronously by the TO context. The advancement saga is a long-running process that may span multiple event deliveries. Compensating actions are defined for failure scenarios (term 94). |
| Game (RG) → Spectator View (SV) | **Eventual** | Spectator projections are updated asynchronously via the ACL-transformed event stream. A slight delay is acceptable and is a stated design property (A35). |
| Game (RG) → Audit & Game Log (AL) | **Near-Synchronous (before broadcast)** | Events are recorded in the Game Log before being broadcast to other consumers (V12, INV-GL-03). This is stronger than eventual: the ordering guarantee is that the Game Log write precedes event publication. However, the AL context itself is a conformist consumer and does not enforce strong consistency with the RG aggregate's in-memory state. |
| Room (RG) → Tournament (TO) | **Eventual** | `RoomCompleted` triggers round-completion evaluation asynchronously. The Tournament aggregate's state lags slightly behind the Room's terminal state. |
| Session (IS) → Room (RG) | **Synchronous precondition** | Room Gameplay validates session tokens synchronously before accepting any command (cross-context query, not an event flow). This is a synchronous remote call, not an event-driven consistency relationship. |
| Player Identity (IS) → Player Rating (RK) | **Eventual** | The `PlayerRegistered` event from IS triggers Player Rating aggregate creation in RK asynchronously. |

---

## 3.8 Key Design Decisions

### 3.8.1 Game is a Separate Aggregate Root from Room

The Game aggregate is elevated to a separate Aggregate Root (rather than being an entity owned by the Room aggregate) for the following reasons:

1. **Write frequency mismatch.** A Game aggregate receives 5–20 commands per second during active play (card plays, draws, Uno calls, challenges, turn advances). The Room aggregate handles lifecycle events at a much lower frequency (joining, starting, completing). Serializing all game commands through the Room aggregate would create an unnecessary bottleneck and inflate the Room's write lock contention by orders of magnitude.

2. **Single-writer ownership.** Each active Game requires a single-process controller for sequence number enforcement and turn state machine consistency. This maps cleanly to an Aggregate Root boundary. An Entity owned by Room would still require the same single-writer guarantee, but it would conflate the Room's write lock with the Game's write lock, preventing any Room-level operations (e.g., reading match state, updating player slot status) from proceeding concurrently with gameplay.

3. **Independent lifecycle.** A Game has its own lifecycle (`Initializing` → `InProgress` → `Completed`) that is independent of the Room's lifecycle. The Room may be `InProgress` while one Game has `Completed` and the next has not yet `Initialized`. Treating Game as a separate aggregate accurately models this independence.

4. **Event granularity.** Game events (e.g., `CardPlayed`, `TurnAdvanced`) are produced at very high volume and consumed by downstream contexts (Spectator View, Audit) that have no interest in Room-level concerns. A clean aggregate boundary ensures that Game events carry only game-relevant context and are not entangled with Room lifecycle state.

### 3.8.2 Session is a Separate Aggregate Root from Player Identity

The Session aggregate is separated from Player Identity for the following reasons:

1. **Different invariants and lifecycles.** Player Identity is relatively stable (display name, credentials, account status change rarely). Sessions are ephemeral and frequently created and invalidated. Coupling them into a single aggregate would force the Player Identity to absorb the high write frequency of session operations.

2. **Atomic single-active-session enforcement.** The critical invariant (INV-S-01) requires that the old session is invalidated and the new session is created atomically. This is cleanest when Session is its own aggregate: the IS context can execute a single transaction that invalidates all existing `Active` sessions for a `playerId` and creates the new one, all within the Session aggregate's consistency boundary. Player Identity does not need to be involved in this transaction.

3. **Independent scaling.** At 1,000,000 concurrent players, session validation is the highest-frequency IS operation (every command from every player validates its session token). Session aggregates must be readable at extreme throughput. Separating Sessions from Player Identities allows each to be optimized and scaled independently.

### 3.8.3 Match is an Entity (not an Aggregate Root)

The Match is deliberately modeled as an Entity owned by the Room aggregate rather than a separate Aggregate Root because:

1. **Low write frequency.** Match state changes only when a Game completes (at most 3 times per Match). This is a very low write rate that does not justify the overhead of a separate aggregate root with its own consistency boundary.

2. **Strong consistency required with Room.** The Match and Room must be consistent with each other: when the Match concludes, the Room must transition to `Completed` in the same atomic operation. Keeping Match as a child of Room ensures this atomicity without a distributed transaction.

3. **No independent commands.** No command targets the Match directly; all Match state changes are reactions to Game completion events. There is no `UpdateMatch` command from any actor. This makes a separate aggregate root unnecessary.

### 3.8.4 Tournament Bracket is Read-Optimized

The Tournament Bracket entity is explicitly designated as read-optimized and non-authoritative. Advancement decisions are made from `TournamentRound.advancementResults`, not from the Bracket. This separation ensures that:

1. **Write path correctness is not contingent on Bracket consistency.** The Bracket can be rebuilt from events at any time if it becomes inconsistent.
2. **The Bracket can be optimized for read patterns** (bracket visualization queries) without imposing those optimizations on the write-critical advancement logic.
3. **The Bracket update can be eventually consistent** with advancement events, while the advancement logic itself remains strongly consistent within the Tournament aggregate.

### 3.8.5 Spectator Game Projection uses an ACL at the Context Boundary

The Anti-Corruption Layer is owned by the Spectator View context (not by Room Gameplay) deliberately. Room Gameplay is the core domain and must not be burdened with knowledge of what spectators can or cannot see. Placing the ACL at the consumption boundary (SV context) enforces the privacy guarantee without polluting the core domain model. This is consistent with the general principle that downstream contexts are responsible for translating upstream contracts into their own models (doc 02, Section 2.2.2).

---

*This document is part of the UnoArena domain specification series. All terms used herein conform to the ubiquitous language defined in [01-domain-glossary.md](01-domain-glossary.md). Bounded context abbreviations (RG, TO, RK, IS, SV, AL) follow [02-bounded-contexts-and-context-map.md](02-bounded-contexts-and-context-map.md). All assumption references (A1–A38, V1–V20) trace to [08-assumptions-and-open-questions.md](08-assumptions-and-open-questions.md).*
