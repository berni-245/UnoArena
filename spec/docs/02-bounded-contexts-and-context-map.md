# 2. Bounded Contexts and Context Map

> This document defines the strategic boundaries of the UnoArena domain, the responsibilities of each bounded context, and the relationships that govern information flow between them. All definitions use the ubiquitous language established in the [Domain Glossary](01-domain-glossary.md).

---

## 2.1 Bounded Context Inventory

UnoArena is decomposed into six bounded contexts. Each context owns a distinct area of domain knowledge, enforces its own invariants, and communicates with other contexts exclusively through domain events and published contracts.

---

### 2.1.1 Identity & Session Context

**Responsibility:** Owns player identity, credential management, authentication, and the enforcement of the single-active-session invariant. This context is the authority on "who is this player?" and "is this session valid?"

**Key Aggregates:**

| Aggregate | Description |
|-----------|-------------|
| **Player Identity** | Registration data, credentials, display name, account state (active, suspended, banned). |
| **Session** | Represents a single authenticated session: token, creation timestamp, expiration, device fingerprint, active/invalidated state. |

**Invariants:**

- A player may have at most one active session at any point in time. Establishing a new session invalidates any previously active session for that player.
- A session token must be valid (not expired, not invalidated) for any command to be accepted by downstream contexts.
- Player display names must be unique within the platform.
- Account state must be `active` to establish a session.

**Events Produced (Outbound):**

| Event | Description |
|-------|-------------|
| `PlayerRegistered` | A new player identity has been created. |
| `SessionEstablished` | A new session has been created for a player. |
| `SessionInvalidated` | A previously active session has been terminated (due to new login, explicit logout, expiration, or administrative action). |
| `PlayerSuspended` | A player account has been suspended (administrative action). |
| `PlayerBanned` | A player account has been permanently banned. |

**Events Consumed (Inbound):** None. Identity & Session is a pure upstream context. It does not react to events from other contexts.

---

### 2.1.2 Room Gameplay Context

**Responsibility:** The core context. Owns the complete lifecycle of rooms, matches, and individual games. This includes the game state machine, deck management, card plays, turn progression, Uno call mechanics, challenge resolution, disconnection handling within a game, and the authoritative random number generation for shuffling. This context is the single source of truth for "what happened in a game."

**Key Aggregates:**

| Aggregate | Description |
|-----------|-------------|
| **Room** | The top-level lifecycle container. Transitions through `waiting` -> `in_progress` -> `completed`. Holds room configuration (casual vs. tournament, player capacity, best-of-3 match structure). |
| **Match** | A best-of-three series between the players in a room. Tracks game wins per player, cumulative card-point totals for tiebreaking, and match outcome. |
| **Game** | A single Uno game. The primary aggregate for gameplay. Owns the deck, discard pile, player hands, turn order, direction, current player, game phase, sequence number for concurrency control, and the Uno call/challenge window state. |
| **Player Hand** | The set of cards held by a player within a game. Owned by the Game aggregate. |
| **Deck** | The draw pile for a game. Server-authoritative; seed and contents are never exposed to clients or spectators. |
| **Disconnection Timer** | Tracks the 60-second reconnection window for a disconnected player within a game. |

**Invariants:**

- Only the current player may play a card or draw from the deck (turn enforcement).
- A played card must be a legal play according to Uno rules (matching color, number, or wild).
- Every state change is assigned a monotonically increasing sequence number; commands bearing a stale sequence number are rejected (HTTP 409 Conflict).
- A player who plays their second-to-last card must call "Uno!" before the next player takes their turn. The challenge window is 5 seconds or until the next player begins their turn, whichever comes first.
- A disconnected player has exactly 60 seconds to reconnect before automatic forfeit.
- During the reconnection window, the disconnected player's turn is skipped (treated as a pass).
- A room in `waiting` state may not start a match until minimum player count is reached.
- Each game's deck state is generated server-side and appended to the immutable game log before any card is dealt or drawn.
- A match consists of at most 3 games; the first player to win 2 games wins the match.
- All state mutations are appended to the game log before being broadcast.

**Events Produced (Outbound):**

| Event | Description |
|-------|-------------|
| `RoomCreated` | A new room has been created (casual or tournament). |
| `PlayerJoinedRoom` | A player has joined a room in `waiting` state. |
| `PlayerLeftRoom` | A player has left a room in `waiting` state. |
| `RoomFilled` | Room has reached its configured player capacity. |
| `GameStarted` | A new game within a match has begun. Deck shuffled, hands dealt. |
| `CardPlayed` | A player played a card. Includes card identity, player, and resulting game state changes. |
| `CardDrawn` | A player drew a card from the deck. Includes card identity (private to consuming contexts that need it). |
| `TurnAdvanced` | The turn has moved to the next player. |
| `DirectionReversed` | Play direction changed (due to Reverse card). |
| `PlayerSkipped` | A player's turn was skipped (due to Skip card or disconnection pass). |
| `ColorChosen` | A player chose a color after playing a Wild card. |
| `UnoCallMade` | A player called "Uno!" after playing their second-to-last card. |
| `UnoChallengeWindowOpened` | The 5-second challenge window has begun. |
| `ChallengeMade` | An opponent challenged a player's Uno call (or lack thereof). |
| `ChallengeResolved` | A challenge has been adjudicated. Includes outcome: challenger penalized or challenged player penalized. |
| `UnoChallengeWindowClosed` | The challenge window has expired without a challenge. |
| `PlayerDisconnected` | A player's connection has been lost; reconnection timer started. |
| `PlayerReconnected` | A player has reconnected within the 60-second window. |
| `PlayerForfeited` | A player has forfeited (disconnection timeout or explicit forfeit). |
| `GameCompleted` | A single game has ended. Includes final placement order and card-point totals. |
| `MatchCompleted` | A best-of-three match has concluded. Includes match winner and per-game results. |
| `RoomCompleted` | All matches in the room are finished. Includes final room results. |

**Events Consumed (Inbound):**

| Event | Source Context | Reaction |
|-------|---------------|----------|
| `SessionInvalidated` | Identity & Session | Force-disconnect the affected player from any active game. Starts the 60-second reconnection window (but since session is invalid, reconnection will fail, leading to forfeit). |
| `TournamentRoomAssigned` | Tournament Orchestration | Create a tournament room with the specified players and configuration. |

---

### 2.1.3 Tournament Orchestration Context

**Responsibility:** Owns the lifecycle of tournaments: registration, round management, player-to-room assignment, advancement logic, final-room creation, and tournament-placement rating. This context does not know the rules of Uno; it only knows that rooms produce match results and that players advance or are eliminated based on those results.

**Key Aggregates:**

| Aggregate | Description |
|-----------|-------------|
| **Tournament** | The top-level aggregate. Tracks tournament state (registration_open -> in_progress -> completed), total registered players, current round number, and tournament-placement ratings. |
| **Tournament Round** | Represents one elimination tier. Owns the set of rooms for that round, tracks room completion, and enforces advancement rules. |
| **Tournament Bracket** | The logical structure mapping players to rooms within a round and tracking advancement paths across rounds. |

**Invariants:**

- A tournament round may not advance until all rooms in that round have completed their matches.
- Within each room, the top 3 players by match wins advance. Ties are broken by lowest cumulative card-point total, then by earliest final-game completion time.
- When 10 or fewer players remain, a single final room is created instead of a new round.
- A forfeited player in a tournament room is immediately eliminated from the tournament.
- Tournament-placement rating is distinct from the global Elo rating; tournament play does not affect Elo.
- A player may not be registered in two concurrent tournaments (or this is an explicit business decision to allow; see [Assumptions](08-assumptions-and-open-questions.md)).

**Events Produced (Outbound):**

| Event | Description |
|-------|-------------|
| `TournamentCreated` | A new tournament has been created and registration is open. |
| `PlayerRegisteredForTournament` | A player has registered for a tournament. |
| `TournamentStarted` | Registration closed; first round is being created. |
| `TournamentRoundCreated` | A new round has been initialized with room assignments. |
| `TournamentRoomAssigned` | A specific room has been assigned a set of players for a tournament round. |
| `PlayerAdvanced` | A player has advanced to the next round. |
| `PlayerEliminated` | A player has been eliminated from the tournament. |
| `FinalRoomCreated` | The final room (10 or fewer players) has been created. |
| `AllMatchesInRoundCompleted` | All rooms in a round have reported their match results. |
| `TournamentCompleted` | The tournament has concluded. Includes final placements and tournament-placement ratings. |

**Events Consumed (Inbound):**

| Event | Source Context | Reaction |
|-------|---------------|----------|
| `MatchCompleted` | Room Gameplay | Record match result for the corresponding tournament room. Check if all matches in the round are complete. |
| `PlayerForfeited` | Room Gameplay | If the room is a tournament room, mark the player as eliminated. Record the forfeit as a loss for match-advancement purposes. |
| `RoomCompleted` | Room Gameplay | Mark the tournament room as completed. Trigger advancement evaluation for the round. |

---

### 2.1.4 Ranking & Statistics Context

**Responsibility:** Owns the global Elo rating system, player statistics, leaderboards, and historical game result aggregation. This context computes Elo deltas and maintains read-optimized views of player performance.

**Key Aggregates:**

| Aggregate | Description |
|-----------|-------------|
| **Player Rating** | The Elo rating for a single player. Tracks current rating, rating history, and the number of rated games played. |
| **Leaderboard** | A ranked projection of all player ratings, optimized for read access. |
| **Player Statistics** | Aggregate statistics: games played, wins, losses, forfeits, average placement, favorite card color, etc. |

**Invariants:**

- Elo is updated only for completed casual games; tournament games do not affect Elo.
- Elo is updated once per completed game (not per match, not per tournament).
- Elo delta is calculated from the final placement order within the room (1st through last).
- Abandoned games (forfeit by all remaining players) do not affect Elo.
- Elo updates are idempotent: processing the same `GameCompleted` event twice must not produce a double adjustment.

**Events Produced (Outbound):**

| Event | Description |
|-------|-------------|
| `EloUpdated` | A player's Elo rating has been recalculated after a casual game. Includes old rating, new rating, and delta. |
| `LeaderboardUpdated` | The leaderboard projection has been refreshed. |
| `PlayerStatisticsUpdated` | A player's aggregate statistics have been recalculated. |

**Events Consumed (Inbound):**

| Event | Source Context | Reaction |
|-------|---------------|----------|
| `GameCompleted` | Room Gameplay | If the game was in a casual room, compute Elo deltas for all participating players based on final placement order. Update player statistics regardless of room type. |
| `PlayerForfeited` | Room Gameplay | Update player statistics (increment forfeit count). No Elo impact unless the game subsequently completes as a casual game with remaining players. |

---

### 2.1.5 Spectator View Context

**Responsibility:** Maintains a read-optimized projection of active games, purpose-built for spectator consumption. This context receives raw gameplay events and transforms them through an Anti-Corruption Layer that enforces strict privacy filtering, ensuring that no private game state (player hands, deck contents) ever crosses the boundary into spectator-visible data.

**Key Aggregates:**

| Aggregate | Description |
|-----------|-------------|
| **Spectator Game Projection** | A read model representing the spectator-visible state of an active game: public player information, card counts, discard pile, turn state, direction, and game phase. |
| **Spectator Feed** | The ordered stream of spectator-safe events for a given game, consumed by spectator clients. |

**Invariants:**

- No player hand contents may appear in the spectator projection. Only card counts per player are exposed.
- Draw pile contents and deck seed/state are never exposed.
- The spectator projection must be eventually consistent with the authoritative game state in Room Gameplay, but it is a read model with no write-back capability.
- Spectator events must preserve causal ordering (events appear in the same order as they occurred in the game).

#### 2.1.5.1 Spectator Boundary -- What Crosses and What Is Withheld

**Information that crosses into the Spectator View:**

| Data Element | Source |
|--------------|--------|
| Player display names | `GameStarted` |
| Card count per player (integer only) | Derived from `CardPlayed`, `CardDrawn`, `ChallengeResolved`, `GameStarted` |
| Discard pile top card (color, value) | `CardPlayed`, `ColorChosen` |
| Current turn indicator (which player's turn) | `TurnAdvanced` |
| Play direction (clockwise/counter-clockwise) | `DirectionReversed`, `GameStarted` |
| Game phase (in_progress, completed) | `GameStarted`, `GameCompleted` |
| Uno call status | `UnoCallMade` |
| Challenge occurrence and outcome | `ChallengeMade`, `ChallengeResolved` |
| Player connection status | `PlayerDisconnected`, `PlayerReconnected` |
| Match score (games won per player) | `GameCompleted` |

**Information explicitly withheld (never crosses the boundary):**

| Data Element | Reason |
|--------------|--------|
| Player hand contents (specific cards) | Core privacy rule: spectators must not see private hands. |
| Draw pile contents | Would reveal future game state. |
| Deck seed and RNG state | Would allow reconstruction of the full deck, exposing all hands. |
| Sequence numbers (internal concurrency control) | Implementation detail of Room Gameplay; irrelevant to spectators. |
| Event signatures (audit integrity tokens) | Belong to the Audit context; not spectator-relevant. |

#### 2.1.5.2 Domain Events Driving Spectator Updates

The following events from Room Gameplay drive updates to the Spectator View. Each is transformed by the Anti-Corruption Layer before being projected.

| Source Event (Room Gameplay) | Spectator Event (After ACL) | Transformation Applied |
|------------------------------|----------------------------|----------------------|
| `GameStarted` | `SpectatorGameStarted` | Hand contents stripped. Player names, initial card counts (7 each), initial discard top card, starting player, and direction are retained. |
| `CardPlayed` | `SpectatorCardPlayed` | Retained as-is. The played card is public (it is on the discard pile). Player's remaining card count is included. |
| `CardDrawn` | `SpectatorCardDrawn` | **Card identity stripped.** Only the player ID and new card count are included. The spectator knows a card was drawn but not which card. |
| `TurnAdvanced` | `SpectatorTurnAdvanced` | Retained as-is. Current player identifier is included. |
| `DirectionReversed` | `SpectatorDirectionReversed` | Retained as-is. |
| `PlayerSkipped` | `SpectatorPlayerSkipped` | Retained as-is. Includes reason (Skip card, disconnection pass). |
| `ColorChosen` | `SpectatorColorChosen` | Retained as-is. Chosen color is public. |
| `UnoCallMade` | `SpectatorUnoCallMade` | Retained as-is. |
| `ChallengeMade` | `SpectatorChallengeMade` | Retained as-is. Challenger and challenged player identifiers included. |
| `ChallengeResolved` | `SpectatorChallengeResolved` | Outcome retained (who was penalized). Penalty card identities stripped; only the resulting card count change is included. |
| `GameCompleted` | `SpectatorGameCompleted` | Final placement order retained. Hand contents at end of game are not revealed. |
| `PlayerDisconnected` | `SpectatorPlayerDisconnected` | Retained as-is. Spectators see that a player has disconnected. |
| `PlayerReconnected` | `SpectatorPlayerReconnected` | Retained as-is. |

**Events Produced (Outbound):**

| Event | Description |
|-------|-------------|
| `SpectatorProjectionUpdated` | The read model for a game has been updated with new spectator-safe state. |

**Events Consumed (Inbound):**

| Event | Source Context | Reaction |
|-------|---------------|----------|
| All gameplay state-change events listed above | Room Gameplay | Transform through ACL, update Spectator Game Projection, emit spectator-safe event to Spectator Feed. |

---

### 2.1.6 Audit & Game Log Context

**Responsibility:** Maintains an immutable, append-only event log of all domain-significant events across all contexts. Supports replay capability, dispute resolution, and forensic analysis. Every event is recorded with its full payload, causal metadata, and (where applicable) an integrity signature.

**Key Aggregates:**

| Aggregate | Description |
|-----------|-------------|
| **Game Log** | An immutable, ordered sequence of all events for a single game. Supports full game replay. |
| **Audit Trail** | An immutable, append-only record of all domain events across all contexts, indexed by context, aggregate, timestamp, and correlation ID. |
| **Event Signature Record** | Cryptographic signature metadata attached to events that require integrity verification (game state changes, Elo updates, tournament advancement decisions). |

**Invariants:**

- The game log is append-only. No event may be modified or deleted after recording.
- Every event must be recorded with a timestamp, source context identifier, correlation ID, and causation ID.
- Signed events must include a verifiable integrity signature that covers the event payload and its position in the log.
- The game log for a game must contain every state mutation in causal order, sufficient to deterministically replay the entire game from the initial deck state.

**Events Produced (Outbound):**

| Event | Description |
|-------|-------------|
| `AuditEntryRecorded` | Confirmation that an event has been durably recorded in the audit trail. Used primarily for operational monitoring. |

**Events Consumed (Inbound):**

| Event | Source Context | Reaction |
|-------|---------------|----------|
| All events from Room Gameplay | Room Gameplay | Append to the game log and audit trail. Verify and store event signatures for game state changes. |
| All events from Tournament Orchestration | Tournament Orchestration | Append to the audit trail. Sign and store advancement decisions. |
| All events from Ranking & Statistics | Ranking & Statistics | Append to the audit trail. Sign and store Elo update calculations. |
| All events from Identity & Session | Identity & Session | Append to the audit trail. Record session lifecycle events for security forensics. |
| All events from Spectator View | Spectator View | Append to the audit trail. Record projection updates for debugging and consistency verification. |

---

## 2.2 Context Map

The context map defines the relationships between all bounded contexts using standard DDD relationship patterns. The visual representation is available at [View Context Map](../diagrams/context-map.md).

### 2.2.1 Relationship Summary

```
+-------------------------+       +----------------------------+
|  Identity & Session     |       |  Room Gameplay             |
|  (Upstream to all)      |------>|  (Core Domain)             |
|  OHS / Published Lang.  |       |  Upstream to most contexts |
+-------------------------+       +----------------------------+
         |                             |    |    |    |
         | SessionInvalidated          |    |    |    |
         +---------------------------->+    |    |    |
                                            |    |    |
         +----------------------------------+    |    |
         | GameCompleted, MatchCompleted,        |    |
         | PlayerForfeited, RoomCompleted        |    |
         v                                       |    |
+----------------------------+                   |    |
| Tournament Orchestration   |                   |    |
| (Downstream of Gameplay)   |                   |    |
| Conformist                 |                   |    |
+----------------------------+                   |    |
         |                                       |    |
         | TournamentRoomAssigned                 |    |
         +-------------------------------------->+    |
                                                      |
         +--------------------------------------------+
         | All gameplay state-change events
         v
+----------------------------+     +----------------------------+
| Spectator View             |     | Ranking & Statistics       |
| (Downstream of Gameplay)   |     | (Downstream of Gameplay)   |
| ACL (privacy filtering)    |     | Conformist                 |
+----------------------------+     +----------------------------+
                                            |
         +----------------------------------+
         | EloUpdated (informational)
         v
+----------------------------+
| Audit & Game Log           |
| (Downstream of ALL)        |
| Conformist                 |
+----------------------------+
```

### 2.2.2 Detailed Relationship Definitions

#### Identity & Session --> Room Gameplay

| Aspect | Detail |
|--------|--------|
| **Pattern** | Open Host Service (OHS) with Published Language |
| **Direction** | Identity & Session is **Upstream (U)**; Room Gameplay is **Downstream (D)** |
| **Contract** | Identity publishes a well-defined authentication contract (session token validation) and session lifecycle events. Room Gameplay conforms to this contract for authenticating commands and reacting to session invalidation. |
| **Key Event Flow** | `SessionInvalidated` --> Room Gameplay force-disconnects the player. |
| **Integration Style** | Room Gameplay validates session tokens synchronously (query) before accepting commands. Session lifecycle events are consumed asynchronously. |

#### Identity & Session --> Tournament Orchestration

| Aspect | Detail |
|--------|--------|
| **Pattern** | Open Host Service (OHS) with Published Language |
| **Direction** | Identity & Session is **Upstream (U)**; Tournament Orchestration is **Downstream (D)** |
| **Contract** | Tournament Orchestration validates player identity when processing tournament registration commands. |
| **Key Event Flow** | `PlayerSuspended`, `PlayerBanned` --> Tournament Orchestration may disqualify the player from active tournaments. |

#### Identity & Session --> Ranking & Statistics

| Aspect | Detail |
|--------|--------|
| **Pattern** | Open Host Service (OHS) with Published Language |
| **Direction** | Identity & Session is **Upstream (U)**; Ranking & Statistics is **Downstream (D)** |
| **Contract** | Ranking uses player identity for rating records. Player display name changes are propagated via `PlayerRegistered` / profile update events. |

#### Room Gameplay --> Tournament Orchestration

| Aspect | Detail |
|--------|--------|
| **Pattern** | Conformist (Tournament conforms to Gameplay's event schema) |
| **Direction** | Room Gameplay is **Upstream (U)**; Tournament Orchestration is **Downstream (D)** |
| **Contract** | Tournament Orchestration accepts Room Gameplay's event schema as-is. It does not impose its own language on gameplay events. |
| **Key Event Flows** | `MatchCompleted` --> triggers advancement check. `PlayerForfeited` --> triggers elimination (tournament rooms only). `RoomCompleted` --> triggers round-completion check. |
| **Note** | This is a bidirectional dependency in terms of event flow: Tournament Orchestration also publishes `TournamentRoomAssigned`, which Room Gameplay consumes to create tournament rooms. However, the domain authority flows from Gameplay (upstream) to Tournament (downstream). The reverse flow (`TournamentRoomAssigned`) is a command-like event where Tournament requests room creation. |

#### Tournament Orchestration --> Room Gameplay

| Aspect | Detail |
|--------|--------|
| **Pattern** | Published Language (Tournament publishes room assignment commands in a shared schema) |
| **Direction** | Tournament Orchestration is **Upstream (U)** for room assignment commands; Room Gameplay is **Downstream (D)** for this specific flow. |
| **Contract** | `TournamentRoomAssigned` carries player IDs, room configuration, and tournament context. Room Gameplay creates a room conforming to this specification. |
| **Note** | This creates a **Partnership** between Room Gameplay and Tournament Orchestration. Neither fully dominates the other. Room Gameplay is the authority on game rules; Tournament Orchestration is the authority on tournament structure. They cooperate through published events. |

#### Room Gameplay --> Ranking & Statistics

| Aspect | Detail |
|--------|--------|
| **Pattern** | Conformist (Ranking conforms to Gameplay's event schema) |
| **Direction** | Room Gameplay is **Upstream (U)**; Ranking & Statistics is **Downstream (D)** |
| **Contract** | Ranking & Statistics consumes `GameCompleted` events and extracts placement order, room type (casual vs. tournament), and player identifiers. |
| **Key Event Flow** | `GameCompleted` (casual rooms only) --> Elo recalculation for all participants. |
| **Filtering** | Ranking & Statistics is responsible for filtering: it ignores `GameCompleted` events from tournament rooms and abandoned games. |

#### Room Gameplay --> Spectator View

| Aspect | Detail |
|--------|--------|
| **Pattern** | Anti-Corruption Layer (ACL) on the Spectator View side |
| **Direction** | Room Gameplay is **Upstream (U)**; Spectator View is **Downstream (D)** |
| **Contract** | Room Gameplay publishes its full event stream. Spectator View's ACL intercepts these events and transforms them into spectator-safe projections by stripping private data (hand contents, card identities on draw, deck state). |
| **ACL Responsibility** | The ACL is owned by the Spectator View context. Room Gameplay is unaware of spectator concerns; it publishes its events without modification. The burden of privacy enforcement lies entirely on the Spectator View's ACL. |
| **Key Transformations** | See Section 2.1.5.2 for the full event transformation table. The most critical transformation is `CardDrawn` --> `SpectatorCardDrawn` (card identity stripped). |

#### Room Gameplay --> Audit & Game Log

| Aspect | Detail |
|--------|--------|
| **Pattern** | Conformist (Audit conforms to all upstream event schemas) |
| **Direction** | Room Gameplay is **Upstream (U)**; Audit & Game Log is **Downstream (D)** |
| **Contract** | Audit consumes the full, unfiltered event stream from Room Gameplay, including all private data (card identities, deck state). This is necessary for replay capability and dispute resolution. |
| **Key Requirement** | Events must be recorded with integrity signatures. The Audit context verifies signatures upon receipt. |

#### Tournament Orchestration --> Audit & Game Log

| Aspect | Detail |
|--------|--------|
| **Pattern** | Conformist |
| **Direction** | Tournament Orchestration is **Upstream (U)**; Audit & Game Log is **Downstream (D)** |
| **Contract** | Audit consumes all tournament lifecycle events for forensic traceability of advancement decisions, round creation, and tournament outcomes. |

#### Ranking & Statistics --> Audit & Game Log

| Aspect | Detail |
|--------|--------|
| **Pattern** | Conformist |
| **Direction** | Ranking & Statistics is **Upstream (U)**; Audit & Game Log is **Downstream (D)** |
| **Contract** | Audit records all Elo update events, providing an auditable history of rating changes. |

#### Identity & Session --> Audit & Game Log

| Aspect | Detail |
|--------|--------|
| **Pattern** | Conformist |
| **Direction** | Identity & Session is **Upstream (U)**; Audit & Game Log is **Downstream (D)** |
| **Contract** | Audit records session lifecycle events for security forensics (login history, session invalidations, suspicious activity). |

#### Spectator View --> Audit & Game Log

| Aspect | Detail |
|--------|--------|
| **Pattern** | Conformist |
| **Direction** | Spectator View is **Upstream (U)**; Audit & Game Log is **Downstream (D)** |
| **Contract** | Audit records spectator projection updates for operational debugging and consistency verification. |

---

### 2.2.3 Shared Kernel

There is **no shared kernel** between any contexts. Each context maintains its own internal models. Where contexts need to reference the same conceptual entity (e.g., "player"), they do so through published identifiers (Player ID) rather than shared domain objects. This ensures:

- Independent deployability of each context.
- Freedom to evolve internal models without cross-context coupling.
- Clear ownership of invariants (each context enforces only its own).

The only shared artifacts are:

- **Player ID** (a value object / opaque identifier) -- referenced by all contexts but owned by Identity & Session.
- **Room ID**, **Game ID**, **Match ID** -- generated by Room Gameplay, referenced by Tournament Orchestration, Ranking, Spectator View, and Audit.
- **Tournament ID**, **Round ID** -- generated by Tournament Orchestration, referenced by Room Gameplay (for tournament room creation) and Audit.

These are **Published Language identifiers**, not shared kernels. Each context may wrap these identifiers in its own value objects with context-specific semantics.

---

## 2.3 Cross-Context Event Propagation Flows

The following subsections detail the key cross-context event flows that drive the UnoArena domain.

### 2.3.1 GameCompleted --> Elo Update (Casual Games)

```
Room Gameplay                      Ranking & Statistics
     |                                    |
     |  GameCompleted                      |
     |  { gameId, roomId, roomType:        |
     |    "casual", placements:            |
     |    [{playerId, position,            |
     |    cardPoints}], abandoned: false }  |
     |----------------------------------->|
     |                                    | 1. Verify roomType == "casual"
     |                                    | 2. Verify abandoned == false
     |                                    | 3. Compute Elo delta for each
     |                                    |    player based on placement
     |                                    | 4. Apply Elo updates (idempotent)
     |                                    | 5. Update player statistics
     |                                    |
     |                                    |  EloUpdated (per player)
     |                                    |---> Audit & Game Log
     |                                    |
     |                                    |  PlayerStatisticsUpdated
     |                                    |---> Audit & Game Log
```

### 2.3.2 MatchCompleted --> Tournament Advancement

```
Room Gameplay                 Tournament Orchestration
     |                                    |
     |  MatchCompleted                     |
     |  { matchId, roomId, tournamentId,   |
     |    roundId, winner, matchWins: {},   |
     |    cardPointTotals: {},             |
     |    finalGameCompletionTime }        |
     |----------------------------------->|
     |                                    | 1. Record match result for room
     |                                    | 2. Check: all matches in room done?
     |                                    |
     |  RoomCompleted                      |
     |  { roomId, tournamentId, roundId,   |
     |    results: [{playerId, matchWins,  |
     |    cardPoints, completionTime}] }   |
     |----------------------------------->|
     |                                    | 3. Rank players: top 3 by matchWins,
     |                                    |    tiebreak by cardPoints (lower wins),
     |                                    |    then by completionTime (earlier wins)
     |                                    | 4. Emit PlayerAdvanced / PlayerEliminated
     |                                    | 5. Check: all rooms in round done?
     |                                    |
     |                                    |  AllMatchesInRoundCompleted
     |                                    |  (internal or published)
     |                                    |
     |                                    | 6. If remaining players > 10:
     |                                    |    Create next round, assign rooms
     |                                    |    Emit TournamentRoundCreated
     |                                    |    Emit TournamentRoomAssigned (per room)
     |                                    |
     |                                    | 7. If remaining players <= 10:
     |                                    |    Create final room
     |                                    |    Emit FinalRoomCreated
     |                                    |    Emit TournamentRoomAssigned
     |                                    |
     |  TournamentRoomAssigned             |
     |<-----------------------------------|
     |                                    |
     | Create tournament room              |
     | with assigned players               |
```

### 2.3.3 PlayerForfeited --> Tournament Elimination

```
Room Gameplay                 Tournament Orchestration
     |                                    |
     |  PlayerForfeited                    |
     |  { playerId, gameId, roomId,        |
     |    tournamentId (if applicable),    |
     |    reason: "disconnection_timeout"  |
     |    | "explicit_forfeit" }           |
     |----------------------------------->|
     |                                    | 1. Verify room is a tournament room
     |                                    | 2. Mark player as eliminated
     |                                    | 3. Record forfeit as match loss
     |                                    |
     |                                    |  PlayerEliminated
     |                                    |  { playerId, tournamentId,
     |                                    |    roundId, reason: "forfeit" }
     |                                    |---> Audit & Game Log
```

### 2.3.4 SessionInvalidated --> Forced Disconnect

```
Identity & Session              Room Gameplay
     |                                    |
     |  SessionInvalidated                 |
     |  { sessionId, playerId,             |
     |    reason: "new_login" |            |
     |    "explicit_logout" |              |
     |    "admin_action" }                 |
     |----------------------------------->|
     |                                    | 1. Find active game for playerId
     |                                    | 2. If player is in a game:
     |                                    |    a. Emit PlayerDisconnected
     |                                    |    b. Start 60-second timer
     |                                    |    c. Player cannot reconnect
     |                                    |       (session is invalid), so
     |                                    |       timer will expire
     |                                    |    d. On expiry: emit PlayerForfeited
     |                                    |
     |                                    |  PlayerDisconnected
     |                                    |---> Spectator View, Audit
     |                                    |
     |                                    |  (after 60s)
     |                                    |  PlayerForfeited
     |                                    |---> Tournament, Ranking, Audit
```

### 2.3.5 All State Changes --> Audit Log Append

```
Every Context                    Audit & Game Log
     |                                    |
     |  <Any Domain Event>                 |
     |  { standard envelope:               |
     |    eventId, timestamp,              |
     |    sourceContext, correlationId,     |
     |    causationId, payload,            |
     |    signature (if applicable) }      |
     |----------------------------------->|
     |                                    | 1. Validate event envelope
     |                                    | 2. Verify signature (if present)
     |                                    | 3. Append to audit trail (immutable)
     |                                    | 4. If game-related: append to
     |                                    |    game-specific log
     |                                    | 5. Emit AuditEntryRecorded
```

---

## 2.4 Context Ownership Summary

| Context | Owns | Depends On |
|---------|------|------------|
| Identity & Session | Player identity, session lifecycle, authentication | None (pure upstream) |
| Room Gameplay | Room lifecycle, match lifecycle, game state, deck, hands, turns, Uno mechanics | Identity (authentication), Tournament (room assignment) |
| Tournament Orchestration | Tournament lifecycle, rounds, brackets, advancement, placement rating | Identity (player validation), Room Gameplay (match/room results) |
| Ranking & Statistics | Elo ratings, leaderboards, player statistics | Identity (player reference), Room Gameplay (game results) |
| Spectator View | Spectator-safe read projections of active games | Room Gameplay (all gameplay events, filtered through ACL) |
| Audit & Game Log | Immutable event log, replay capability, event signatures | All contexts (consumes all events) |

---

## 2.5 Key Design Decisions

1. **Room Gameplay as the core upstream context.** All other contexts derive their state from events published by Room Gameplay. This ensures a single authoritative source for game state, preventing split-brain scenarios.

2. **Spectator View ACL owned by the Spectator context, not by Room Gameplay.** Room Gameplay should not bear the responsibility of knowing what spectators may or may not see. This keeps the core domain clean and places privacy filtering where it belongs -- at the consumption boundary.

3. **No shared kernel.** Coupling between contexts is limited to published event schemas and opaque identifiers. This allows each context to evolve its internal model independently.

4. **Audit as a universal downstream conformist.** The Audit & Game Log context conforms to every other context's event schema. It never imposes its own language on upstream contexts. This makes it a passive, reliable observer.

5. **Partnership between Room Gameplay and Tournament Orchestration.** Neither context fully dominates the other. Room Gameplay is the authority on game rules; Tournament Orchestration is the authority on tournament structure. They cooperate through a bidirectional event flow: Room Gameplay publishes game results, Tournament Orchestration publishes room assignments.

6. **Elo isolation.** Tournament play and casual play are strictly separated for Elo purposes. The Ranking & Statistics context filters `GameCompleted` events by room type, ensuring tournament games never contaminate the global Elo ladder.

---

*[View Context Map Diagram](../diagrams/context-map.md)*
