# Domain Glossary -- Ubiquitous Language

> **Document purpose:** Establish a single, authoritative vocabulary for every conversation, model, and line of code in the UnoArena platform. Every team member -- developer, domain expert, QA -- must use these terms with the meanings defined here and no others.

---

## How to read this glossary

| Column | Meaning |
|---|---|
| **Term** | The canonical name used in code, events, and conversation. |
| **Definition** | Precise, unambiguous meaning within UnoArena. |
| **Bounded Context** | The context that *owns* (is authoritative for) this concept. A second context may hold a read-only or projected copy. |
| **Disambiguation** | Clarifies how this term differs from similar or overloaded terms. |

Bounded-context abbreviations used below:

| Abbreviation | Bounded Context |
|---|---|
| **RG** | Room Gameplay |
| **TO** | Tournament Orchestration |
| **RK** | Ranking |
| **IS** | Identity & Session |
| **SV** | Spectator View |
| **AL** | Audit & Game Log |

---

## 1. Core Game Concepts

| # | Term | Definition | Context | Disambiguation |
|---|------|------------|---------|----------------|
| 1 | **Game** | A single, complete play-through of Uno within a Room, starting when the first card is dealt and ending when one player empties their hand or all remaining players forfeit. A Game produces exactly one placement order. | RG | Not a Match (best-of-three series) and not a Round (tournament tier). A Game is the atomic scoring unit for Elo in casual rooms. |
| 2 | **Match** | A best-of-three series of Games played between the same set of players within a single Room. The player who wins 2 out of 3 Games wins the Match. In tournament rooms, advancement is determined by Match wins. | RG | A Match exists only as a grouping of Games within a Room. It is not the same as a Round (tournament elimination tier). In casual rooms, a Match is still best-of-three but carries no tournament consequence. |
| 3 | **Room** | A bounded play area hosting 2--10 players who compete in a single Match (best-of-three Games). Every Room has a lifecycle (Waiting, In-Progress, Completed) and a unique identifier. | RG | See Casual Room vs. Tournament Room for behavioral differences. A Room is not a Round. |
| 4 | **Turn** | The period during which exactly one player holds the right to play a card, draw a card, or pass (if unable to play). A Turn ends when the active player commits a valid action or is skipped due to a game mechanic (Skip card, disconnection). | RG | A Turn belongs to a Game, not to a Match or Room directly. |
| 5 | **Card** | An immutable value object representing a single Uno card, identified by its Color and Face (number or action symbol). Wild cards have no inherent color until played. | RG | Cards in the Draw Pile are face-down and private; cards in the Discard Pile are public. |
| 6 | **Color** | One of four values: Red, Yellow, Green, Blue. A Wild or Wild Draw Four card has no color until its player issues a Color Declaration. | RG | "Color" always refers to an Uno card color, never to a UI theme or player avatar color. |
| 7 | **Face** | The value printed on a card: a number (0--9) or an action symbol (Skip, Reverse, Draw Two, Wild, Wild Draw Four). | RG | -- |
| 8 | **Deck** | The full set of 108 Uno cards generated server-side at the start of each Game. The Deck is immediately split into the Draw Pile and initial Hands; it is never referenced directly after the deal. | RG | The Deck is an ephemeral concept; once dealt, only Draw Pile, Hands, and Discard Pile exist. |
| 9 | **Hand** | The private, ordered collection of cards held by a single player during a Game. A Hand is visible only to its owner and the server. | RG | Spectators and other players must never see a Hand's contents. A Hand is not a "round of cards." |
| 10 | **Draw Pile** | The face-down stack of remaining cards from which players draw. When exhausted, the Discard Pile (minus its top card) is reshuffled server-side and becomes the new Draw Pile. | RG | Also called the "stock" in some Uno variants; within UnoArena the canonical term is Draw Pile. |
| 11 | **Discard Pile** | The face-up stack onto which players play cards. Only the top card is relevant for determining legal plays. The full Discard Pile history is part of the Game Log. | RG | Spectators can see the Discard Pile (specifically its top card and stack history). |
| 12 | **Top Card** | The single card currently on top of the Discard Pile. It defines the legal Color and Face for the next play. | RG | After a Wild card, the Top Card's effective color is determined by the Color Declaration, not the printed card color. |
| 13 | **Deal** | The server-authoritative action of shuffling the Deck and distributing 7 cards to each player's Hand at the start of a Game, then placing the first card on the Discard Pile. | RG | Dealing is entirely server-side; no client randomness is involved. |
| 14 | **Play** (verb/noun) | The act of a player placing a card from their Hand onto the Discard Pile. A Play is legal only if the card matches the Top Card's Color, Face, or is a Wild. | RG | "Play" is always a card action; do not confuse with "Game" or "playing a Match." |
| 15 | **Draw** (verb/noun) | The act of taking the top card from the Draw Pile into the active player's Hand. A player who cannot (or chooses not to) play a card must draw one card. | RG | "Draw" is a game action. "Penalty Draw" is a separate, forced draw triggered by a rule violation or card effect. |
| 16 | **Pass** | The act of ending a Turn without playing a card after the player has already drawn one card and still cannot (or chooses not to) play. | RG | A player may not Pass without first Drawing, except when their turn is being auto-skipped due to disconnection. |
| 17 | **Placement Order** | The final ranking of players within a single Game, determined by the order in which they empty their Hands. The first player to empty their Hand is 1st place. Players who forfeit or are eliminated receive the lowest remaining positions. | RG | Placement Order within a Game is used for Elo delta calculation. It is not the same as Tournament Placement. |
| 18 | **Card Points** | The numeric value assigned to each card remaining in a player's Hand when a Game ends: number cards = face value, Skip/Reverse/Draw Two = 20 points each, Wild/Wild Draw Four = 50 points each. Used as a tiebreaker in tournament advancement. | RG, TO | Card Points are a tiebreaker metric, not a scoring system. Lower cumulative Card Points is better in tiebreak scenarios. |

---

## 2. Uno-Specific Mechanics

| # | Term | Definition | Context | Disambiguation |
|---|------|------------|---------|----------------|
| 19 | **Uno Call** | A required declaration by a player at the moment they play their second-to-last card, leaving exactly one card in their Hand. The call must be issued before the next player begins their Turn. | RG | An Uno Call is not a challenge; it is the declaration itself. Failure to call triggers a Challenge Window. |
| 20 | **Challenge** | An action by any opponent during the Challenge Window asserting that the player who just played their second-to-last card failed to call "Uno!". | RG | "Challenge" in UnoArena always refers to an Uno-call challenge, not a Wild Draw Four challenge (that variant rule is not implemented). |
| 21 | **Challenge Window** | A 5-second server-enforced timer that opens immediately after a player plays their second-to-last card. During this window, any opponent may issue a Challenge. The window closes when either (a) 5 seconds elapse, (b) the next player begins their Turn, or (c) a valid Challenge is submitted. | RG | The Challenge Window is a time-bound domain concept, not a UI element. Its expiry is tracked server-side. |
| 22 | **Penalty Draw** | A forced draw of 2 cards imposed on a player as a consequence of either (a) failing to call Uno when challenged, or (b) issuing a false Challenge against a player who did call Uno. | RG | Distinct from a normal Draw (voluntary/mandatory single card draw) and from Draw Two / Wild Draw Four effects. |
| 23 | **Wild Card** | A card with no inherent color that can be played on any Turn regardless of the Top Card's Color or Face. Upon playing a Wild Card, the player must issue a Color Declaration. | RG | "Wild Card" is the generic category; "Wild Draw Four" is a specific subtype with additional effects. |
| 24 | **Action Card** | Any non-number card: Skip, Reverse, Draw Two, Wild, and Wild Draw Four. Action Cards trigger immediate mechanical effects beyond simply changing the Top Card. | RG | All Wild Cards are Action Cards, but not all Action Cards are Wild. |
| 25 | **Reverse** | An Action Card that inverts the current Turn direction (clockwise becomes counter-clockwise and vice versa). In a two-player Game, Reverse acts identically to a Skip. | RG | Reverse changes the Turn direction for the remainder of the Game (until another Reverse is played), not just for one Turn. |
| 26 | **Skip** | An Action Card that causes the next player in Turn order to lose their Turn entirely. | RG | A skipped Turn is not the same as a Turn skipped due to disconnection, though the observable effect is similar. |
| 27 | **Draw Two** | An Action Card that forces the next player to draw 2 cards from the Draw Pile and forfeit their Turn. | RG | Draw Two is a card effect, not a Penalty Draw. The two concepts impose draws for different reasons. |
| 28 | **Wild Draw Four** | A Wild Card that forces the next player to draw 4 cards and forfeit their Turn, while allowing the playing player to declare the new active Color. | RG | In UnoArena, the "illegal play" challenge variant for Wild Draw Four is not implemented. |
| 29 | **Color Declaration** | The mandatory announcement of a Color (Red, Yellow, Green, or Blue) that accompanies every Wild Card play. The declared Color becomes the effective color of the Top Card. | RG | A Color Declaration is not a separate Turn action; it is part of the Wild Card play command. |
| 30 | **Turn Direction** | The current rotational order in which Turns proceed: clockwise or counter-clockwise. Initialized to clockwise at Game start; toggled by each Reverse card played. | RG | -- |
| 31 | **Legal Play** | A card play that satisfies the current game state: the card matches the Top Card's Color or Face, or the card is a Wild. No other plays are accepted by the server. | RG | The server is authoritative; illegal plays are rejected, not corrected. |

---

## 3. Room & Lifecycle

| # | Term | Definition | Context | Disambiguation |
|---|------|------------|---------|----------------|
| 32 | **Casual Room** | A Room created for ad-hoc play outside any tournament. Elo ratings are updated at the conclusion of each Game in a Casual Room. Players may leave freely (forfeit) without tournament consequences. | RG, RK | Casual Room Elo updates happen per Game, not per Match. |
| 33 | **Tournament Room** | A Room created by the Tournament Orchestration context as part of a specific Round. Tournament Rooms follow the same Match rules as Casual Rooms, but results feed into tournament advancement, not Elo. | RG, TO | A Tournament Room is owned by Room Gameplay for mechanics but its creation and result consumption are driven by Tournament Orchestration. |
| 34 | **Room State** | The lifecycle phase of a Room. A Room passes through five possible states: **Waiting**, **Ready**, **In-Progress**, **Completed**, or **Abandoned**. Transitions are strictly ordered and irreversible. | RG | Room State is distinct from Game state (a Room contains a Match of up to 3 Games, each with its own sub-state). |
| 35 | **Waiting** (Room State) | The initial Room State. The Room is open for players to join. Transitions to Ready when the minimum player count is met. | RG | A Waiting room has no Games in progress. Transitions back to Waiting if players leave and drop the count below minimum. |
| 36 | **Ready** (Room State) | The Room State after the minimum player count is met but before the host has started the Match. The host may issue `StartMatch` from this state. | RG | Distinct from Waiting: the room has enough players to begin. A player leaving drops it back to Waiting. |
| 37 | **In-Progress** (Room State) | The Room State during active gameplay. At least one Game within the Match is ongoing or the next Game in the series is being initialized. | RG | An In-Progress room may be between Games (e.g., Game 1 finished, Game 2 not yet started) but is still considered In-Progress. |
| 38 | **Completed** (Room State) | The terminal Room State reached when the Match concludes with a valid outcome (a player won 2 Games). Results are finalized and emitted. No further state changes are permitted. | RG | A Completed room is immutable. Events from this room are consumed downstream by Ranking and/or Tournament Orchestration. |
| 39 | **Abandoned** (Room State) | The terminal Room State reached when all remaining players forfeit before the Match produces a winner. No Elo changes are issued; the outcome is recorded as abandoned. | RG | Distinct from Completed: there is no winner. See also Abandoned Game (term 43) for the game-level concept. |
| 40 | **Player Slot** | A reserved position within a Room for a single player. A Room has between 2 and 10 Player Slots. Each slot tracks the player's identity, connection status, and match-level statistics. | RG | A Player Slot is not a Hand; the slot persists across all Games in the Match, while a new Hand is dealt each Game. |
| 41 | **Minimum Players** | The minimum number of occupied Player Slots required to start a Match. For casual rooms, the Room creator sets this (minimum 2). For tournament rooms, this is determined by the Round's player distribution algorithm. | RG, TO | If a Room drops below 2 active (non-forfeited) players during a Game, the Game ends immediately with the remaining player as winner. |
| 42 | **Abandoned Game** | A Game in which all remaining players forfeit, leaving no natural winner. An Abandoned Game produces no Elo changes and is recorded in the Game Log as abandoned. When all players forfeit across an entire Match, the Room itself reaches the Abandoned state (term 39). | RG, RK | An Abandoned Game is not the same as a Game with one winner by last-player-standing; that Game has a valid Placement Order. |

---

## 4. Tournament Concepts

| # | Term | Definition | Context | Disambiguation |
|---|------|------------|---------|----------------|
| 41 | **Tournament** | A complete, multi-round elimination competition supporting up to 1,000,000 players. A Tournament progresses through successive Rounds until 10 or fewer players remain, at which point a Final Room is created. | TO | A Tournament is not a Room or a Match. It is an orchestration-level aggregate that coordinates many Rooms across many Rounds. |
| 42 | **Round** | One elimination tier within a Tournament. In each Round, all remaining players are distributed into Tournament Rooms, play their Matches, and the top 3 per Room advance to the next Round. | TO | A Round is a tournament-level concept. It is not a Game (single Uno play-through) or a Turn. |
| 43 | **Bracket** | A read-model visualization of the Tournament's elimination structure, showing which players advanced from which Rooms in which Rounds. The Bracket is a projection, not a command-side entity. | TO, AL | The Bracket is not authoritative; it is derived from domain events. It exists for display and analytics purposes. |
| 44 | **Elimination** | The removal of a player from a Tournament. A player is eliminated when they do not finish in the top 3 of their Room's Match results, or when they forfeit during a Tournament Room Match. | TO | Elimination applies only to tournaments. In casual rooms, a player who forfeits simply leaves. |
| 45 | **Advancement** | The promotion of a player from one Round to the next. The top 3 players (by Match wins) in each Tournament Room advance. Ties are broken first by lowest cumulative Card Points, then by earliest Completion Time. | TO | Advancement is a tournament domain event, not a game mechanic. |
| 46 | **Seeding** | The algorithm that distributes players into Tournament Rooms for each Round. Seeding may consider tournament placement rating or be randomized, depending on tournament configuration. | TO | Seeding happens once per Round, before Rooms are created. It is not the same as the Deal within a Game. |
| 47 | **Final Room** | The single Tournament Room created when 10 or fewer players remain in a Tournament. The Final Room's Match determines the Tournament winner and final standings. | TO | The Final Room follows the same Match rules as any other Tournament Room but produces the terminal Tournament Placement. |
| 48 | **Tournament Placement** | A player's finishing position in a Tournament (1st, 2nd, ... Nth). Determined by Round of elimination and, for the Final Room, by Match results within that Room. | TO | Tournament Placement is not Elo. Tournaments use a separate Tournament Placement Rating. |
| 49 | **Tournament Placement Rating (TPR)** | A separate rating metric (distinct from Elo) that reflects a player's historical performance across Tournaments. Updated after each Tournament based on Tournament Placement. | TO, RK | TPR is never mixed with Elo. Elo is for casual rooms only; TPR is for tournament performance only. |
| 50 | **Completion Time** | The server-recorded timestamp at which a player's final Game in a Match concludes. Used as the last tiebreaker for Advancement when Match wins and cumulative Card Points are equal. | TO, RG | Completion Time is wall-clock time recorded by the server, not a duration. |
| 51 | **Cumulative Card Points** | The sum of Card Points across all Games in a Match where two players are tied on Match wins. The player with the lower total advances. | TO | Cumulative Card Points are computed only when needed as a tiebreaker. They are not a persistent score. |
| 52 | **Round Progression** | The process by which Tournament Orchestration detects that all Rooms in a Round have Completed, collects advancement results, and triggers Seeding for the next Round. | TO | Round Progression is an asynchronous, event-driven process, not a synchronous blocking call. |

---

## 5. Player & Session

| # | Term | Definition | Context | Disambiguation |
|---|------|------------|---------|----------------|
| 53 | **Player** | A registered user of UnoArena, uniquely identified by a Player ID. A Player may participate in at most one Room at a time and holds persistent Elo and TPR ratings. | IS, RG | "Player" always refers to a registered human user. There are no bots or AI players in UnoArena. |
| 54 | **Session** | An authenticated connection between a Player and the platform, represented by a session token. A Player may have at most one Active Session at any time. | IS | A Session is an identity/auth concept. It is not a Game or a Room. |
| 55 | **Active Session** | The single currently valid Session for a Player. Creating a new Session automatically invalidates any previous Active Session (single-active-session invariant). | IS | If a player logs in on a second device, the first device's session is invalidated immediately. |
| 56 | **Disconnection** | The server-detected loss of a Player's real-time connection during an active Game. A Disconnection triggers the start of a Reconnection Window. | IS, RG | A Disconnection is a connectivity event, not a forfeit. The player may reconnect. |
| 57 | **Reconnection Window** | A 60-second server-enforced timer that begins when a Disconnection is detected. During this window, the disconnected Player's Turns are automatically skipped. If the player reconnects within 60 seconds, they resume with their original Hand. | RG, IS | The Reconnection Window is a fixed 60-second duration. No extensions or pauses are supported. |
| 58 | **Forfeit** | The permanent withdrawal of a Player from the current Game (and, by extension, the current Match). In casual rooms, the player simply exits and the Game continues with remaining players. In tournament rooms, a forfeit counts as a Match loss and triggers Elimination. | RG, TO | Forfeit is a deliberate or system-triggered action. It is distinct from Disconnection (which allows reconnection). A forfeit is also triggered automatically when the Reconnection Window expires during the player's Turn. |
| 59 | **Inactive Player** | A Player whose Reconnection Window has expired without successful reconnection. An Inactive Player is treated as having forfeited. | RG | An Inactive Player is not the same as an offline Player who is not currently in a Room. |
| 60 | **Auto-Skip** | The server behavior of automatically passing a disconnected Player's Turn during their Reconnection Window. No bot substitution occurs; the Turn is simply skipped. | RG | Auto-Skip is mechanically similar to a Pass but is system-initiated, not player-initiated. |

---

## 6. Ranking & Scoring

| # | Term | Definition | Context | Disambiguation |
|---|------|------------|---------|----------------|
| 61 | **Elo Rating** | A numeric skill rating assigned to each Player, updated only after completed Games in Casual Rooms. Elo is never affected by tournament play or by Abandoned Games. | RK | Elo is per-player and global. It is not per-room or per-match. It updates once per Game, not once per Match. |
| 62 | **Elo Delta** | The positive or negative change applied to a Player's Elo Rating after a single Game, calculated from the Player's Placement Order relative to the other players' Elo Ratings. | RK | Elo Delta is computed per Game. In a best-of-three Match, a player receives up to 3 separate Elo Deltas (one per completed Game). |
| 63 | **Placement Order** (repeated for Ranking context) | The finishing positions (1st, 2nd, ..., last) of all players in a Game. Used as the input for Elo Delta computation. | RG, RK | Placement Order within a Game, not within a Match or Tournament. |
| 64 | **Match Wins** | The number of Games won by a Player within a single Match (0, 1, or 2). The primary criterion for tournament Advancement. | RG, TO | Match Wins is a count of Game victories, not a score. |
| 65 | **Tiebreak** | The ordered set of rules applied when two or more players in a Tournament Room have equal Match Wins: (1) lowest Cumulative Card Points, then (2) earliest Completion Time. | TO | Tiebreak rules apply only to tournament advancement decisions, not to Elo calculations. |

---

## 7. Concurrency & Ordering

| # | Term | Definition | Context | Disambiguation |
|---|------|------------|---------|----------------|
| 66 | **Sequence Number** | A monotonically increasing integer attached to every gameplay command sent by a client. For gameplay commands, the Sequence Number is scoped per-Game (resets each new Game within a Match). The server rejects any command whose Sequence Number does not match the expected next value. | RG | Sequence Numbers are per-Game (for gameplay commands). They are not global and not per-Room. |
| 67 | **Stale Command** | A command received by the server with a Sequence Number lower than the expected next value, indicating the client's view of game state is outdated. Stale Commands are rejected with HTTP 409 Conflict. | RG | A Stale Command is not an invalid command (wrong card, out of turn); it is specifically a sequencing issue. |
| 68 | **Command Rejection** | The server's refusal to process a command. Reasons include: stale Sequence Number (409), illegal play, out-of-turn action, or rate limit exceeded. The rejection event is logged and communicated to the client. | RG | Command Rejection is a server-authoritative decision. The client must reconcile its local state. |
| 69 | **Conflict** (HTTP 409) | The specific error response returned when a Stale Command is detected. The client is expected to re-fetch the current game state (via SSE catch-up or explicit query) and retry with the correct Sequence Number. | RG | "Conflict" in UnoArena always means a sequencing conflict, not a generic error. |
| 70 | **Game Log** | The append-only, immutable record of every state change within a Game: card plays, draws, Uno calls, challenges, penalties, forfeits, and system actions (auto-skip, reshuffle). Every entry includes a timestamp and causal event reference. | RG | The Game Log is the source of truth for replay and dispute resolution. It is never modified after writing. |
| 71 | **Immutable Event Log** | The platform-wide, append-only log of all domain events across all bounded contexts. Each event is recorded before being broadcast. The Game Log is a subset scoped to a single Game. | RG, TO, RK, IS | The Immutable Event Log is a system-level concept broader than the Game Log. |
| 72 | **Idempotency Key** | A unique identifier attached to a command by the client to allow safe retries. If the server has already processed a command with the same Idempotency Key, it returns the cached result without re-executing. | RG | Idempotency Keys prevent duplicate processing; they are not the same as Sequence Numbers (which enforce ordering). |
| 73 | **Server-Authoritative RNG** | The principle that all randomness (deck shuffling, draw order) is generated exclusively on the server. Clients never influence or provide random seeds. | RG | This is a design invariant, not a runtime entity. |
| 74 | **Event Ordering** | The guarantee that events within a single Game are totally ordered and causally consistent. Cross-game events are eventually consistent. | RG, TO | Within a Game, ordering is strict. Across Rooms or Contexts, ordering is best-effort and event-driven. |

---

## 8. Spectating

| # | Term | Definition | Context | Disambiguation |
|---|------|------------|---------|----------------|
| 75 | **Spectator** | A connected user observing a Game in real time without participating. Spectators cannot issue game commands and are restricted to the Public Game State view. | SV | A Spectator is not a Player in the Room. A Player in the Room who has been eliminated/forfeited does not become a Spectator; they simply disconnect or wait. |
| 76 | **Observer** | Synonym for Spectator within UnoArena. Both terms may be used interchangeably in documentation, but "Spectator" is the canonical code-level term. | SV | -- |
| 77 | **Public Game State** | The subset of Game state that is safe to expose to Spectators: player names, player card counts (number of cards, not identities), Turn direction, current Top Card, Discard Pile history, and game phase. | SV | Public Game State explicitly excludes all Hand contents. |
| 78 | **Private Hand** | A player's Hand contents, which are never included in the Spectator View or any public broadcast. Only the owning player and the server have access. | RG, SV | The boundary between Private Hand and Public Game State is a hard security invariant, not a UI preference. |
| 79 | **Spectator View** | The projected read model consumed by Spectators. It is derived from domain events but filtered to exclude Private Hand data. Updates are pushed in near real time. | SV | The Spectator View is a downstream projection, not the authoritative game state. |
| 80 | **Card Count** | The number of cards in a player's Hand, visible to all players and Spectators. It reveals Hand size but not Hand contents. | RG, SV | Card Count is public; card identities are private. |

---

## 9. Security, Rate Limiting & Audit

| # | Term | Definition | Context | Disambiguation |
|---|------|------------|---------|----------------|
| 81 | **Rate Limit** | A server-enforced cap on the number of commands or requests a client may submit within a time window. Applied at multiple layers: per IP, per User, and per Room/Tournament action type. | IS, RG | Rate limiting is a cross-cutting concern; the Identity & Session context owns the policy, but enforcement occurs at the Room Gameplay boundary. |
| 82 | **Adaptive Throttling** | A dynamic rate-limiting strategy that adjusts thresholds based on current system load, player behavior patterns, and abuse signals. Repeat offenders receive progressively stricter limits. | IS | Adaptive Throttling is a refinement of Rate Limiting, not a separate mechanism. |
| 83 | **Session Invalidation** | The act of terminating a Player's Active Session, either because a new Session was created (single-active-session enforcement) or due to administrative action (ban, security incident). | IS | Session Invalidation is distinct from Disconnection. Invalidation revokes auth; Disconnection is a connectivity loss. An invalidated session cannot reconnect. |
| 84 | **Audit Log** | A tamper-resistant record of sensitive operations: login/logout, session invalidations, administrative actions, Elo adjustments, and tournament result overrides. Separate from the Game Log. | IS, AL | The Audit Log is for security and compliance. The Game Log is for gameplay replay. They may share storage infrastructure but have different access controls and retention policies. |
| 85 | **Event Signature** | A cryptographic signature attached to critical domain events (e.g., game results, Elo updates, tournament advancement) to ensure integrity and non-repudiation. | RG, RK, TO | Event Signatures are a security measure, not a domain-logic concept. They protect against tampering of the Immutable Event Log. |
| 86 | **Input Validation** | Server-side verification that every command conforms to expected schema, value ranges, and business preconditions before processing. Invalid inputs are rejected before reaching domain logic. | RG, IS | Input Validation is the first line of defense; domain invariants are the second. Both must pass for a command to succeed. |
| 87 | **Single-Active-Session Invariant** | The system-wide rule that each Player may have at most one valid Session at any time. Enforced at login: creating a new Session immediately and atomically invalidates any prior Session. | IS | This invariant prevents multi-device abuse and session-duplication attacks. |

---

## 10. Cross-Cutting & Infrastructure-Adjacent Domain Terms

| # | Term | Definition | Context | Disambiguation |
|---|------|------------|---------|----------------|
| 88 | **Domain Event** | A record of something meaningful that happened in the domain: a card was played, a game completed, a player was eliminated. Domain Events are immutable, timestamped, and the primary integration mechanism between bounded contexts. | All | A Domain Event is not a command (which requests a change) or a query (which reads state). |
| 89 | **Command** | A request to change domain state, issued by a player or by the system (e.g., auto-forfeit). Commands are validated, sequenced, and either accepted (producing Domain Events) or rejected. | RG, TO | Commands are imperative; events are past-tense facts. |
| 90 | **Aggregate** | A cluster of domain objects treated as a single unit for consistency purposes. Each Aggregate enforces its own invariants and processes commands atomically. In UnoArena, the Room (containing Match and Games) is a primary aggregate. | All | An Aggregate is a DDD modeling concept, not a runtime service. |
| 91 | **Bounded Context** | A linguistic and model boundary within which terms have precise, consistent meanings. UnoArena defines at least six bounded contexts (see abbreviation table above). | All | The same word (e.g., "completed") may have different implications in different bounded contexts. |
| 92 | **Eventual Consistency** | The consistency model applied between bounded contexts. When Room Gameplay emits a GameCompleted event, the Ranking context will eventually process it, but not instantaneously. Within a single aggregate, consistency is immediate (strong). | All | Eventual consistency is inter-context; strong consistency is intra-aggregate. |
| 93 | **Saga / Process Manager** | A coordination pattern that manages long-running, multi-step business processes spanning multiple aggregates or contexts. Tournament Round Progression is a primary saga in UnoArena. | TO | A Saga is not a transaction; it uses compensating actions rather than rollbacks. |
| 94 | **Compensation** | A corrective domain action taken when a step in a saga fails. For example, if advancement results cannot be persisted for a Round, a compensation event may freeze the tournament and alert administrators. | TO | Compensation is not "undo"; it is a domain-meaningful corrective action. |
| 95 | **Projection / Read Model** | A denormalized, query-optimized view built by consuming Domain Events. Bracket visualizations, player statistics, and the Spectator View are all projections. | AL, SV | Projections are eventually consistent and can be rebuilt from the event log at any time. |

---

## Quick-Reference: Key Disambiguations

| Often Confused | Correct Distinction |
|---|---|
| Game vs. Match | A **Game** is a single Uno play-through. A **Match** is a best-of-three series of Games. |
| Match vs. Round | A **Match** is played within a single Room. A **Round** is a tournament-wide elimination tier containing many Rooms and Matches. |
| Round vs. Tournament | A **Round** is one tier. A **Tournament** is the full multi-round competition. |
| Forfeit vs. Disconnection | A **Forfeit** is permanent withdrawal. A **Disconnection** allows a 60-second Reconnection Window before becoming a forfeit. |
| Elo vs. TPR | **Elo** applies to casual rooms only, updated per Game. **TPR** applies to tournaments only, updated per Tournament. |
| Penalty Draw vs. Draw Two | **Penalty Draw** is a 2-card punishment for Uno call violations. **Draw Two** is a card effect forcing the next player to draw 2. |
| Sequence Number vs. Idempotency Key | **Sequence Number** enforces command ordering. **Idempotency Key** enables safe command retries. |
| Game Log vs. Audit Log | **Game Log** records gameplay actions for replay. **Audit Log** records security-sensitive operations for compliance. |
| Spectator vs. Player | A **Spectator** observes but cannot act. A **Player** participates in the Game. |
| Public Game State vs. Private Hand | **Public Game State** is visible to all including Spectators. **Private Hand** is visible only to its owner and the server. |

---

*This glossary is a living document. All additions and modifications must be reviewed to ensure consistency with the ubiquitous language. Any term used in code, events, or APIs that does not appear here must be added before use.*
