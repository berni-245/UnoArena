# 8. Assumptions and Open Questions

> This document separates validated requirements (explicitly stated in the specification) from assumptions (design decisions we have made where the specification is silent) and open questions (genuine ambiguities that would benefit from stakeholder clarification). Every assumption is numbered (A1, A2, ...) for cross-referencing from other deliverables. Every open question is numbered (Q1, Q2, ...) and includes an impact assessment.

---

## 8.1 Validated Requirements (Confirmed from Specification)

The following requirements are explicitly stated in the specification and are treated as non-negotiable constraints throughout this domain model.

| # | Requirement | Source |
|---|-------------|--------|
| V1 | Room size: 2--10 players. | Spec Section 1: "ad-hoc game rooms (2-10 players)." |
| V2 | Match = best-of-three series (up to 3 individual games within a room). | Spec Section 1: "best-of-three series." |
| V3 | Top 3 players by match wins advance per room in tournament rounds. | Spec Section 1: "top 3 players by match wins advance." |
| V4 | Tiebreak order: (1) match wins, (2) lower cumulative card-point total, (3) earliest final-game completion time. | Spec Section 1: explicit tiebreak chain. |
| V5 | Elo applies to casual rooms only, updated once per completed game (not per match or tournament). | Spec Section 1: "global Elo-based ranking applies to casual (ad-hoc) rooms only. Elo is updated once per completed game." |
| V6 | Tournament uses a separate tournament-placement rating (TPR), distinct from Elo. | Spec Section 1: "Tournament play uses a separate tournament-placement rating." |
| V7 | Abandoned games (forfeit by all remaining players) do not affect Elo. | Spec Section 1: "Abandoned games (forfeit by all remaining players) do not affect Elo." |
| V8 | Single active session per player; new login invalidates old session. | Spec Section 1: "single-active-session per player (new login invalidates old session)." |
| V9 | 60-second reconnection window after disconnection. During the window, the disconnected player's turn is skipped (no bot substitution). | Spec Section 1: explicit disconnection rules. |
| V10 | 5-second challenge window for Uno call challenges, closing when (a) 5 seconds elapse, (b) a valid challenge is submitted, or (c) the next player begins their turn. | Spec Section 1: "5-second window after the card is played" and "closes as soon as the next player begins their turn." |
| V11 | Server-authoritative deck: all shuffling and draws generated server-side. No client-provided randomness. | Spec Section 1: "All card shuffling and draws are generated server-side." |
| V12 | Immutable game log: every state change appended before broadcast. | Spec Section 1: "appended to an immutable game log before being broadcast." |
| V13 | Sequence numbers for concurrency control; stale commands rejected with HTTP 409 Conflict. | Spec Section 1: "serialize concurrent REST requests with strict sequence numbers. Stale actions are rejected with HTTP 409 Conflict." |
| V14 | Spectators see only player names and discard stack (public information); never private hands. | Spec Section 1: "only see public information (player names and discard stack), never private hands." |
| V15 | Forfeit in casual room = player exits, game continues with remaining players. Forfeit in tournament room = match loss, player eliminated from tournament. | Spec Section 1: explicit forfeit rules for both room types. |
| V16 | Tournament progresses in elimination rounds until 10 or fewer players remain, then a final room is created. | Spec Section 1: "Tournaments proceed in elimination rounds until 10 or fewer players remain, at which point a final room is created." |
| V17 | Elo delta is calculated from final placement order within the room (1st through last). | Spec Section 1: "Elo delta is calculated from final placement order within the room." |
| V18 | If the reconnection window expires during the player's turn, an automatic forfeit is issued. | Spec Section 1: "If the window expires during the player's turn, an automatic forfeit is issued." |
| V19 | A player who reconnects within the window resumes with their original hand intact. | Spec Section 1: "A player who reconnects within the window resumes with their original hand intact." |
| V20 | Multi-layer rate limits (per IP, per user, per room/tournament action) with adaptive throttling. | Spec Section 1: "multi-layer rate limits...with adaptive throttling." |

---

## 8.2 Assumptions (Design Decisions Where Specification Is Silent)

Each assumption represents a design decision we have made to fill a gap in the specification. These are defensible choices, but they should be reviewed with stakeholders before implementation.

---

### 8.2.1 Connection Semantics (as Permitted by Scope Constraints)

Per Section 3 of the assignment, client connection protocol details are out of scope for this iteration. However, we must state explicit assumptions about delivery semantics so that the domain model can reason about idempotency, retry behavior, and event ordering.

**A1: At-least-once delivery from server to consumers.**
Event delivery from the server to downstream consumers (including the Spectator View, Ranking, Tournament Orchestration, and Audit contexts) uses at-least-once semantics. Every consumer must be idempotent -- processing the same event twice must produce no additional side effects. This is enforced via event ID deduplication at each consumer boundary.

**A2: At-most-once command delivery via sequence numbers.**
Client-to-server commands use at-most-once delivery semantics enforced by sequence numbers. If the client does not receive an acknowledgment, it retries with the same sequence number. The server detects and deduplicates replayed commands by their sequence number. A command that has already been successfully processed is acknowledged again without re-execution.

**A3: Push-based real-time updates with snapshot recovery.**
Server-Sent Events (SSE) or an equivalent push mechanism delivers real-time game state updates to connected clients. If a client misses events (due to brief disconnection or network jitter), recovery is achieved via a state snapshot endpoint combined with event replay from a known checkpoint. The specific protocol is deferred to the next iteration.

**A4: Server-side disconnection detection within 5--10 seconds.**
Connection state (connected vs. disconnected) is detectable by the server within a reasonable timeframe via heartbeat-based mechanisms. We assume the server can detect a dropped connection within 5--10 seconds. This detection starts the 60-second reconnection window defined in V9.

---

### 8.2.2 Uno Rules Clarifications

The specification references "standard Uno rules" via a video link but does not codify every rule in detail. The following assumptions resolve specific ambiguities in Uno gameplay mechanics.

**A5: Simultaneous Uno call with penultimate card play is valid.**
A player MAY call "Uno!" in the same command as playing their penultimate card (bundled call), or as a separate command issued within the challenge window. Both forms are valid. This prevents punishing players for legitimate optimization of their command flow.

**A6: Bundled Uno call still opens the challenge window.**
If a player plays their penultimate card and calls Uno in the same command, the challenge window still opens (per V10). However, any challenger will lose -- the call was valid. The challenge window exists to give opponents the opportunity to verify, not to guarantee a penalty.

**A7: Challenge window timing is server-authoritative.**
The 5-second challenge window (V10) starts from the moment the server processes the penultimate card play, based on the server's clock. Client-perceived time is irrelevant. This eliminates disputes arising from network latency.

**A8: Next player's action closes the challenge window immediately.**
If the next player in turn order submits a valid action (plays a card or draws) before the 5-second window expires, the challenge window closes immediately. A challenge command must be received and processed by the server before the next player's action is processed. Late challenges are rejected.

**A9: WildDrawFour legality is not server-enforced; bluffing is permitted.**
The server does not validate whether the player holds cards matching the active color when a Wild Draw Four is played. This is a deliberate design choice: enforcement of the color-match rule occurs exclusively through the WDF Challenge mechanism (term 22b), preserving the strategic bluff element from official Uno rules. The full WDF Challenge mechanic — windows, adjudication, and penalties — is specified in 03 (INV-G-09, INV-G-23–25), 04 (ChallengeWildDrawFour command, WDF events), 05 (WDF Challenge Sub-Flow), and 06 (edge cases 6.7.9–6.7.11). Previously resolved via Q3.

**A10: Standard 108-card Uno deck composition.**
The deck follows standard Uno composition: 108 cards total. Per color (Red, Yellow, Green, Blue): one 0 card, two each of 1--9, two Skip, two Reverse, two DrawTwo. Plus four Wild and four WildDrawFour cards. This is consistent with the standard Uno rules referenced in the specification.

**A11: Draw pile recycling via reshuffle of discard pile.**
When the draw pile is exhausted, the discard pile (minus the top card, which remains in play) is shuffled using a new server-generated random seed and becomes the new draw pile. The new seed is recorded in the game log, maintaining the replay-safe and auditable properties required by V12.

**A12: After drawing, a player may play only the drawn card or pass.**
If a player draws a card (either voluntarily or because they have no playable card), they may immediately play that drawn card if it is a legal play, or they must pass. They cannot play a different card from their original hand after drawing. This follows standard Uno rules and prevents strategic abuse of the draw action.

---

### 8.2.3 Room and Match Mechanics

**A13: All players in a room participate in every game within the match.**
In a room with N players, all N players are dealt into each game of the best-of-three match. A game ends when one player empties their hand. The remaining N-1 players are ranked by card point totals (lower is better) for tiebreak purposes.

**A14: Placement within a game uses defined ranking rules.**
1st place: the player who empties their hand. Remaining players are ranked by ascending card point total in hand (lower remaining hand value = better placement). If two players have identical card point totals, the tie is broken by proximity in turn order to the winner (the player whose turn would come sooner after the winner is ranked higher).

**A15: "Best-of-three" in multi-player rooms means "play up to 3 games."**
The specification's "best-of-three series" maps cleanly to 2-player rooms (first to 2 wins). In rooms with 3+ players, only one player can win each game, making it impossible for any player to achieve 2 wins in fewer than 2 games. We interpret "best-of-three" as "play a series of up to 3 games, then rank all players by the number of games won." **This is a critical ambiguity; see Q1.**

**A16: Match ranking in multi-player rooms uses wins, then card points, then time.**
After all games in a match are played, players are ranked by: (1) number of games won (descending), (2) cumulative card points across all games (ascending -- lower is better), (3) earliest final-game completion time. This ranking determines the top 3 for tournament advancement.

**A17: Minimum player count to start a game is 2.**
A match (and its constituent games) cannot start unless the room contains at least 2 players. If a room drops below 2 active (non-forfeited) players during a game, the remaining player wins immediately.

**A18: Casual rooms are player-created; tournament rooms are system-created.**
A casual room is created by a player (the "host") who configures room settings (player capacity within 2--10). A tournament room is created exclusively by the Tournament Orchestration context as part of round setup. Players cannot directly create tournament rooms.

---

### 8.2.4 Tournament Mechanics

**A19: Minimum tournament size is 4 players.**
A tournament requires at least 4 registered players to start. With fewer than 4, there is insufficient structure for meaningful elimination rounds.

**A20: Room sizes are maximized to minimize tournament rounds.**
When distributing players into rooms for a tournament round, the seeding algorithm prefers fewer, larger rooms (up to the 10-player maximum) over many smaller rooms. This minimizes the number of rounds required to reach the final. For example, 25 players would be distributed as rooms of 9, 8, and 8 (three rooms, top 3 advance from each = 9 advance) rather than five rooms of 5 (top 3 each = 15 advance, requiring more rounds).

**A20a: Room sizes within a round are balanced as evenly as possible.**
The algorithm distributes players so that room sizes differ by at most 1 player. For example, 23 players with a target of 3 rooms yields rooms of 8, 8, and 7 -- not 10, 10, and 3.

**A21: Brief coordination window between tournament rounds.**
Between tournament rounds, there is a non-gameplay coordination window during which the Tournament Orchestration context assigns rooms and waits for players to confirm their connection. This window is not subject to the 60-second reconnection timer (which applies only during active games). The duration of this coordination window is a configuration parameter.

**A22: Final room uses the same best-of-3 match format.**
The tournament final room (with 10 or fewer players) follows the same match rules as any other tournament room. The match ranking within the final room determines the tournament's final standings (1st, 2nd, 3rd, etc.).

**A23: Solo player auto-advances without playing.**
If a tournament round's math produces a room with fewer than 2 players (e.g., exactly 1 player remains after seeding), that player auto-advances to the next round without playing. This is an edge case that occurs only with unusual elimination patterns.

---

### 8.2.5 Scoring and Rating

**A24: Elo K-factor is 32 for all players.**
We use a standard K-factor of 32 for Elo calculations. A variable K-factor (e.g., lower K for experienced players) is a valid refinement but is not specified and adds complexity without clear domain justification at this stage.

**A25: Multi-player Elo uses average pairwise results.**
In a game with N players, each player's Elo delta is computed as the average of N-1 pairwise expected-vs-actual comparisons against every other player in the room. This is a standard extension of Elo to multiplayer settings, treating the multi-player game as a collection of pairwise matchups.

**A26: Starting Elo for new players is 1200.**
New players begin with an Elo rating of 1200, which is the conventional starting point for Elo-based systems.

**A27: Elo rating floor is 100.**
A player's Elo rating cannot drop below 100. This prevents extreme negative spirals and maintains meaningful rating values.

**A28: Tournament placement rating formula is to be determined.**
The specification states that a separate tournament-placement rating exists but does not define its formula. We acknowledge this as a placeholder: the exact TPR calculation (e.g., points based on final placement, decay over time, weighting by tournament size) will be defined when stakeholders clarify expectations. For now, the domain model supports TPR as a separate numeric value updated per tournament.

---

### 8.2.6 Security

**A29: Session tokens are opaque, high-entropy, and time-limited.**
Session tokens are cryptographically random, opaque strings with a configurable expiry (assumed 24 hours) and refresh capability. Token format and rotation mechanics are implementation details deferred to the next iteration, but the domain model assumes tokens are tamper-proof and non-guessable.

**A30: Default rate limits for key operations.**
In the absence of specification-defined thresholds, we assume the following defaults as starting points for adaptive throttling:

| Operation | Rate Limit |
|-----------|-----------|
| Game commands (play, draw, call Uno, challenge) | 10/second per player per room |
| Room join requests | 5/minute per player |
| Tournament registration | 1/second per player |
| Room creation (casual) | 2/minute per player |
| Spectator subscriptions | 10/minute per player |

These limits are configurable and subject to adaptive throttling based on system load and abuse signals.

**A31: Event integrity uses HMAC with per-room secrets.**
Critical domain events (game state changes, Elo updates, tournament advancement decisions) are signed using HMAC with a per-room secret generated at room creation. The secret is stored securely and is never exposed to clients or spectators. This provides tamper evidence for the immutable game log.

**A32: Audit log retention is differentiated by room type.**
Tournament game logs and audit entries are retained indefinitely (required for dispute resolution in competitive play). Casual game logs are retained for 90 days, after which they may be archived or purged. This is an operational assumption; the domain model treats all events as immutable within their retention period.

---

### 8.2.7 Spectator View

**A33: Spectators see a defined set of public game information.**
Beyond the minimum stated in V14 (player names and discard stack), spectators also see:
- Card count per player (number of cards, not card identities)
- Current turn indicator (which player's turn it is)
- Play direction (clockwise or counter-clockwise)
- Game score within the match (games won per player)
- Match score (current game number, e.g., "Game 2 of 3")
- Uno call events and challenge outcomes

This enriches the spectator experience without compromising any private game state.

**A34: Spectators do NOT see pending Uno call state.**
Spectators see `UnoCallMade` and `ChallengeResolved` events (explicit actions and outcomes) but do not see the absence of a call. The system does not broadcast "Player X has NOT called Uno" -- this would reveal strategic information (whether a player forgot) before the challenge window closes. **See Q5 for further discussion.**

**A35: Spectator view has a slight built-in delay.**
Spectator projections are delivered with eventual consistency, which introduces a natural slight delay. This delay also serves as a privacy buffer: even if spectator data is relayed to a player in the same room via an external channel, the information is marginally stale and therefore less exploitable.

---

### 8.2.8 Game Lifecycle Edge Cases

**A36: Forfeited player's cards are removed from the game entirely.**
When a player forfeits (explicitly or via disconnection timeout), their remaining hand is discarded -- removed from the active game permanently. The cards are not reshuffled into the draw pile. This decreases the total number of cards in the active game, which is simpler and avoids potential issues with partially-known cards being recycled into the draw pile.

**A37: A player may be in at most one active room at a time.**
Whether casual or tournament, a player can participate in only one room at any given time. This simplifies session management, prevents split attention (especially relevant in tournaments), and avoids complex concurrency scenarios where a single player has simultaneous game states across multiple rooms. **See Q10 for discussion.**

**A38: Turn timer exists for connected players.**
Although the specification defines disconnection timeouts (V9), it does not mention a per-turn timer for connected players. We assume a turn timer exists to prevent griefing by stalling: 30 seconds for casual rooms, 60 seconds for tournament rooms. If the timer expires, the server auto-draws (if the player has not yet drawn) and auto-passes. **See Q4 for discussion.**

**A39: Off-turn reconnection window expiry results in deferred forfeit.**
V18 states that an automatic forfeit is issued if the reconnection window expires *during the player's turn*. The specification is silent on what happens when the 60-second window expires while the disconnected player is not the active player. We assume: the player is marked Inactive when the window expires, and the forfeit is issued when their turn next arrives in the rotation. This is consistent with V9 ("considered inactive" after 60 seconds) and ensures no game-blocking delay while preserving the fairness of a full-turn cycle before elimination.

---

## 8.3 Open Questions (Genuine Ambiguities Requiring Stakeholder Clarification)

Each question identifies a real ambiguity in the specification that affects the domain model. The impact is assessed and our current working assumption is stated.

---

### Q1: Best-of-Three Semantics in Multi-Player Rooms (CRITICAL)

**The ambiguity:** The specification says "best-of-three series" and "top 3 players by match wins advance." In a 2-player match, best-of-three is clear: first to 2 wins takes the match. In a 10-player room, each game has exactly one winner (the player who empties their hand). After 3 games, the distribution of wins could be: one player with 3 wins, one with 2, one with 1 -- or three players each with 1 win -- or many other combinations. "Best-of-three" does not produce a single match winner; it produces a ranking.

**Why it matters:** This is fundamental to match scoring, advancement logic, and the entire tournament progression model. If the interpretation is wrong, the core domain mechanics are wrong.

**Possible interpretations:**
1. "Best-of-three" means "play 3 games and rank by wins" (our assumption A15/A16). The term "best-of-three" is used loosely to mean "a series of up to 3 games."
2. "Best-of-three" applies only to 2-player rooms. Multi-player rooms use a different scoring system (e.g., point-based).
3. "Best-of-three" means each pairwise combination of players plays a conceptual best-of-three, with aggregated results. (Impractical at scale.)

**Our working assumption (A15, A16):** Interpretation 1. Play up to 3 games. Rank by wins (descending), then cumulative card points (ascending), then earliest final-game completion time.

**Additional sub-question:** Can the match end after 2 games if the top-3 advancement is already mathematically determined? See Q2.

---

### Q2: When Does a Match End Early?

**The ambiguity:** With multiple players, the third game might be unnecessary if the advancement outcome is already locked in after 2 games. The specification says "best-of-three" (up to 3 games) but does not address early termination.

**Why it matters:** Playing unnecessary games wastes time (especially in tournament contexts where thousands of rooms must complete before the round can advance). Conversely, always playing 3 games provides more data for tiebreaking and a fairer assessment.

**Example:** In a room of 4 players after 2 games, Player A has 2 wins. Players B, C, D have 0 wins. The top 3 advancement slots are: Player A is guaranteed top-3. Players B, C, D will compete for the remaining 2 slots in Game 3 -- but with 0 wins each, they cannot surpass Player A. However, we still need Game 3 to differentiate between B, C, and D for the remaining 2 advancement slots. So Game 3 is still necessary.

**Our working assumption:** Always play all 3 games unless the top-3 advancement outcome is fully mathematically determined (i.e., the 3 advancing players are already locked in regardless of Game 3's result). In practice, early termination is rare with 3+ players, so we conservatively default to playing all 3 games.

---

### Q3: WildDrawFour Challenge Rules — RESOLVED

**The ambiguity:** Standard Uno rules allow a player who receives a WildDrawFour to challenge it, claiming the playing player had a card of the matching color and therefore played the WildDrawFour illegally. The specification did not originally include this mechanic.

**Why it matters:** The WildDrawFour challenge is a strategically significant mechanic that adds depth. The server must snapshot the challenged player's hand at the time of the play for adjudication, which has privacy implications (resolved: only a boolean "had matching color" is revealed, not card identities).

**Resolution (A9 updated):** Implement WildDrawFour challenges with bluff-permitted semantics. The server accepts all WDF plays without color-match validation. The affected next player may challenge within a 5-second window. Adjudication checks the challenged player's hand snapshot. Penalty: bluff confirmed → playing player draws 4; legitimate play → challenger draws 6 (4 + 2 penalty). See 01 (Glossary terms 22b–22c), 03 (INV-G-09, INV-G-23, WildDrawFourChallengeWindow VO), 04 (ChallengeWildDrawFour command, WDF events), 05 (WDF Challenge Sub-Flow), 06 (edge cases 6.7.3–6.7.5).

---

### Q4: Turn Timer for Connected Players

**The ambiguity:** The specification defines a 60-second reconnection window (V9) for disconnected players but does not mention any time limit for a connected player's turn. A connected player could theoretically stall indefinitely.

**Why it matters:** Without a turn timer, a malicious or inattentive player can block an entire room's progress. This is especially damaging in tournament play, where all rooms in a round must complete before the tournament can advance. A single stalling player could hold up 1,000,000 participants.

**Our working assumption (A38):** A turn timer exists. 30 seconds for casual rooms, 60 seconds for tournament rooms (tournament players may need more time for strategic decisions). On expiry: the server auto-draws (if the player has not drawn yet) and auto-passes. The turn timer resets with each new turn.

**Stakeholder input needed:** Are these durations appropriate? Should the timer be configurable per tournament?

---

### Q5: What Spectators See Regarding Uno Calls

**The ambiguity:** Spectators should see exciting gameplay moments (Uno calls, challenges). But should they see the *absence* of an Uno call? Broadcasting "Player X played their penultimate card and has NOT yet called Uno" reveals strategic information during the challenge window.

**Why it matters:** This affects both the spectator experience (suspense vs. information neutrality) and potential collusion scenarios (a spectator relaying information to a player in the room via external channels).

**Options:**
1. Spectators see everything: penultimate card play, call (or lack thereof), challenge, resolution.
2. Spectators see only explicit actions: penultimate card play, UnoCallMade (if it happens), ChallengeMade, ChallengeResolved.
3. Spectators see the outcome only after the challenge window closes.

**Our working assumption (A34):** Option 2. Spectators see UnoCallMade and ChallengeResolved events but are never informed about the absence of a call. The challenge window is invisible to spectators; they only see its effects.

---

### Q6: Fate of a Forfeited Player's Cards

**The ambiguity:** When a player forfeits mid-game, the specification says the game continues with remaining players. But what happens to the forfeited player's hand?

**Options:**
1. Cards are removed from the game entirely (our assumption A36). Total card count in the active game decreases.
2. Cards are shuffled back into the draw pile. Total card count is preserved but the draw pile now contains cards that were in a private hand.
3. Cards are placed face-down at the bottom of the draw pile (preserving total but without shuffling).

**Why it matters:** This affects the draw pile composition, game balance, and the deck integrity invariant. Option 2 is more "fair" (cards aren't lost) but reveals information (players may deduce what was in the forfeited player's hand based on what appears in the draw pile). Option 1 is simpler and avoids information leakage.

**Our working assumption (A36):** Option 1. Removed entirely. Simplicity and privacy outweigh card-count fairness.

---

### Q7: Maximum Duration for Tournament Rooms

**The ambiguity:** The specification does not define a maximum duration for a tournament room's match. If a room takes an extremely long time (e.g., slow players, long games, edge-case deck states), the entire tournament round is blocked.

**Why it matters:** In a tournament of 1,000,000 players, a single slow room in a round of 100,000+ rooms could delay the entire tournament. The round cannot advance until all rooms complete (per the Tournament Orchestration invariant).

**Our working assumption:** A configurable maximum round duration exists (default: 2 hours). If a room has not completed its match within this window, the match is force-completed using the current standings (wins, then card points from completed games, then card points from the in-progress game at the moment of force-completion).

**Stakeholder input needed:** Is 2 hours an appropriate default? Should force-completion trigger administrative review?

---

### Q8: Spectator Access to Tournament Rooms

**The ambiguity:** The specification says spectators can watch "rooms" but does not explicitly distinguish between casual and tournament rooms for spectator access.

**Why it matters:** At scale, tournament spectating could mean thousands of spectators across 100,000+ simultaneous tournament rooms. There is also a fairness question: could a spectator watching Room A relay public information (e.g., which players advanced, how long the match took) to players in Room B?

**Analysis:** Since the spectator view already strips all private information (V14, A33, A34), and tournament rooms are simultaneous (players in different rooms are playing concurrently, not sequentially), the fairness risk is minimal. Public information visible to spectators (card counts, discard pile) does not transfer meaningfully between separate rooms.

**Our working assumption:** Tournament rooms can be spectated with the same privacy rules as casual rooms. No additional restrictions are needed. Scale concerns are an infrastructure issue, not a domain concern.

---

### Q9: Draw Card Stacking

**The ambiguity:** Some popular Uno variants allow "stacking" -- responding to a DrawTwo with another DrawTwo (passing a cumulative draw penalty to the next player). The specification references "standard Uno rules" without clarifying this variant.

**Why it matters:** Stacking fundamentally changes the game's dynamics and strategy. It also introduces complex edge cases (what if the chain goes around the entire table? what if a player has no DrawTwo to stack?).

**Our working assumption:** No stacking. Standard rules apply: a DrawTwo forces the next player to draw 2 cards and lose their turn unconditionally. There is no counter-play. This is consistent with the official Uno rules (Mattel has publicly stated that stacking is not an official rule).

---

### Q10: Multiple Active Rooms Per Player

**The ambiguity:** The specification enforces single active session (V8) but does not explicitly state whether a player can join multiple casual rooms simultaneously within that single session.

**Why it matters:** Allowing multi-room participation introduces complex concurrency (a player could be mid-turn in two rooms simultaneously), complicates the disconnection model (which room does the reconnection window apply to?), and creates UI/UX challenges.

**Our working assumption (A37):** A player may be in at most one active room at a time, whether casual or tournament. Joining a new room while in an active room requires leaving the current room first (which constitutes a forfeit if a game is in progress).

---

### Q11: Tournament Registration Deadline vs. Manual Start

**The ambiguity:** The specification describes tournament creation and round progression but does not detail how registration closes and the tournament begins.

**Options:**
1. Registration closes at a fixed deadline (time-based).
2. Registration closes when a maximum player count is reached (capacity-based).
3. An organizer manually triggers the start.
4. Some combination of the above.

**Why it matters:** This affects the Tournament aggregate's state machine and the player experience. A time-based deadline means the tournament might start with very few players. A capacity cap might never fill. Manual start requires an organizer role.

**Our working assumption:** The tournament supports all three triggers: a registration deadline (time-based), a maximum player cap, and a manual start command by the tournament creator. Registration closes when any of these conditions is met first. The minimum player threshold (A19) must still be satisfied.

---

### Q12: Card Point Calculation Interpretation

**The ambiguity:** The specification says "lower cumulative card-point total in the tied games advances." Does "lower" mean the player had fewer/lower-value cards remaining in hand (i.e., they were closer to winning)?

**Why it matters:** If the interpretation is reversed (higher card points = better), the entire tiebreak logic is inverted.

**Analysis:** In Uno, the winner is the player who empties their hand first. Card points are the sum of remaining cards in a player's hand when someone else wins. A player with fewer remaining cards was closer to winning. Therefore, lower cumulative card points = better performance.

**Confirmed:** Lower cumulative card points is better. This is consistent with the specification's language ("lower...advances") and with standard Uno scoring conventions. This is listed here for completeness and to document the reasoning, but we do not consider it a genuine ambiguity.

---

## 8.4 Assumption Cross-Reference Index

For convenience, the following table maps each assumption to the deliverables and sections where it is referenced or has impact.

| Assumption | Primary Impact Area | Related Deliverables |
|------------|-------------------|---------------------|
| A1 (at-least-once delivery) | Event consumers, idempotency | 04 (Commands/Events), 07 (Consistency/Recovery) |
| A2 (at-most-once commands) | Command processing, deduplication | 04 (Commands/Events), 06 (Edge Cases), 07 (Consistency/Recovery) |
| A3 (push-based updates) | Client state synchronization | 06 (Edge Cases), 07 (Consistency/Recovery) |
| A4 (disconnection detection) | Reconnection window trigger | 05 (Event Flows), 06 (Edge Cases) |
| A5--A8 (Uno call mechanics) | Game aggregate, challenge window | 03 (Aggregates), 04 (Commands/Events), 05 (Event Flows) |
| A9 (WildDrawFour challenge with bluff semantics) | Game aggregate, challenge window, hand snapshot | 01 (Glossary), 03 (Aggregates), 04 (Commands/Events), 05 (Event Flows), 06 (Edge Cases) |
| A10 (deck composition) | Deck value object | 01 (Glossary), 03 (Aggregates) |
| A11 (draw pile recycling) | Game aggregate, game log | 03 (Aggregates), 04 (Commands/Events), 06 (Edge Cases) |
| A12 (draw-then-play-or-pass) | Turn state machine | 03 (Aggregates), 04 (Commands/Events) |
| A13--A14 (game placement) | Scoring, Elo input | 03 (Aggregates), 05 (Event Flows) |
| A15--A16 (match ranking) | Tournament advancement | 03 (Aggregates), 05 (Event Flows) |
| A17 (minimum 2 players) | Room state machine | 03 (Aggregates), 06 (Edge Cases) |
| A18 (room creation ownership) | Room aggregate, context boundaries | 02 (Bounded Contexts), 03 (Aggregates) |
| A19--A23 (tournament mechanics) | Tournament orchestration | 03 (Aggregates), 05 (Event Flows) |
| A24--A28 (scoring/rating) | Ranking context | 03 (Aggregates), 05 (Event Flows) |
| A29--A32 (security) | Identity & Session, Audit | 02 (Bounded Contexts), 06 (Edge Cases) |
| A33--A35 (spectator view) | Spectator View context | 02 (Bounded Contexts), 06 (Edge Cases) |
| A36 (forfeited player cards) | Game aggregate, deck integrity | 03 (Aggregates), 06 (Edge Cases) |
| A37 (one room per player) | Session management | 02 (Bounded Contexts), 06 (Edge Cases) |
| A38 (turn timer) | Game aggregate, turn state machine | 03 (Aggregates), 04 (Commands/Events), 06 (Edge Cases) |

---

## 8.5 Open Question Priority Summary

| Question | Priority | Blocks Implementation? |
|----------|----------|----------------------|
| Q1: Best-of-three multi-player semantics | CRITICAL | Yes -- affects core match/tournament logic |
| Q2: Early match termination | HIGH | Partially -- affects tournament round timing |
| Q3: WildDrawFour challenge | MEDIUM | Resolved -- implemented with bluff-permitted semantics (A9) |
| Q4: Turn timer | HIGH | Yes -- needed to prevent tournament blocking |
| Q5: Spectator Uno call visibility | LOW | No -- current assumption is safe |
| Q6: Forfeited player's cards | MEDIUM | No -- both options are implementable |
| Q7: Tournament room timeout | HIGH | Yes -- needed for tournament liveness guarantee |
| Q8: Tournament room spectating | LOW | No -- current assumption is conservative and safe |
| Q9: Draw card stacking | MEDIUM | No -- standard rules are the safe default |
| Q10: Multiple rooms per player | MEDIUM | No -- single room is the safe default |
| Q11: Tournament registration triggers | MEDIUM | Partially -- affects tournament state machine |
| Q12: Card point interpretation | LOW (confirmed) | No -- interpretation is unambiguous on analysis |

---

*This document is a living artifact. As stakeholder decisions are made on the open questions, the corresponding assumptions should be updated or promoted to validated requirements. All cross-references (A1, Q1, etc.) should remain stable -- retired items are marked as resolved, not renumbered.*
