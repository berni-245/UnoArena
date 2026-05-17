# 7. Consistency and Recovery Strategy

> This document defines domain-level consistency boundaries, saga patterns, deduplication strategies, and compensation/recovery mechanisms for UnoArena. Infrastructure-level details (specific technologies, storage engines, message brokers, network protocols) are explicitly out of scope. All reasoning is expressed in terms of the domain model, invariants, and event flows established in the preceding deliverables.

---

## 7.1 Consistency Boundaries

A foundational design decision for UnoArena is that **not all state needs the same consistency guarantee**. The cost of providing strong consistency is availability: a system that refuses to proceed until every participant agrees on state is one that can be brought to a halt by any single slow or failing participant. Conversely, accepting eventual consistency for a piece of state that must be correct at the moment of decision leads to incorrect outcomes. The design therefore applies the minimum consistency level that each invariant actually requires, and no stronger.

The following sections define those levels precisely.

---

### 7.1.1 Strong Consistency: Within Aggregates

Strong consistency within an aggregate means that every state mutation is applied atomically, that conflicting mutations are serialized and one is definitively rejected, and that the committed state is immediately visible to the next operation on the same aggregate. No observer of that aggregate can see a partially applied mutation.

This guarantee is achieved through two mechanisms:
1. **Single-writer rule:** At any point in time, only one handler processes commands for a given aggregate instance. Concurrent commands queue and are processed in order.
2. **Sequence number enforcement:** Every command bears a `sequenceNumber` representing the version of aggregate state the client believes it is operating against. If the aggregate's actual current version differs from the expected value, the command is rejected (HTTP 409 Conflict, per V13). The aggregate never branches; it is a linear chain of committed mutations.

The following aggregates enforce strong consistency.

---

#### 7.1.1.1 Game Aggregate (Room Gameplay Context)

The Game aggregate is the most consistency-critical object in the system. Every Uno rule is an invariant enforced here. An incorrect game state -- even momentarily visible -- could allow an illegal play, an undeserved Uno challenge win, or a fabricated forfeit.

**What is strongly consistent:**
- The identity of the active player (only they may play or draw).
- The contents of each player's hand (card membership is checked before any play).
- The discard pile top card and declared color (used to validate legality of the next play).
- The sequence number (ordering primitive for all commands).
- The Uno call/challenge window state (open, pending challenge resolution, or closed).
- The turn timer deadline (set by the server at `TurnAdvanced` time; expiry triggers auto-draw/auto-pass per A38).
- The reconnection timer state for disconnected players (started at `PlayerDisconnected`, cancelled at `PlayerReconnected`, expired at forfeit).
- The game phase (dealing, in_progress, completed).

**Why strong consistency is required here:** Uno rules are turn-based and mutually exclusive. The correctness of "Player A played a Blue 7" depends on it being Blue A's turn, Blue A having a Blue 7 in hand, and the discard pile currently showing a Blue card or a 7. If any of these reads is stale, an illegal play succeeds. The game log is the foundation for dispute resolution (V12); a game log that permitted illegal state transitions has no evidentiary value.

**Enforcement mechanism:**
- Every command is rejected if `commandSequenceNumber != currentSequenceNumber + 1`.
- Every command is also checked against an idempotency store (see Section 7.2.1) before the sequence number check is evaluated.
- Every state mutation is appended to the Game Log (owned by Room Gameplay) before the mutation is considered committed, and before any broadcast to clients or downstream contexts.
- No two commands are processed simultaneously for the same Game instance.

**Atomicity in domain terms:** "Atomic" here means that the sequence number increment, the hand mutation, the discard pile update, and the game log append all succeed together or none of them take effect. If any part fails after the game log write (e.g., broadcast fails), the system recovers by re-broadcasting from the log -- the state is still correctly committed.

---

#### 7.1.1.2 Room Aggregate (Room Gameplay Context)

The Room aggregate manages the container lifecycle that wraps Matches and Games.

**What is strongly consistent:**
- Room status transitions: `waiting` → `in_progress` → `completed`. A room cannot be observed in two states simultaneously.
- Player slot management: join, leave, and forfeit are atomic. The current player count is always exact. No room can exceed `maxPlayers` (V1).
- Match start validation: the minimum player count (A17) is checked atomically when `StartMatch` is processed. If two concurrent `StartMatch` commands arrive (e.g., two rapid retries from the host), only one succeeds; the second is deduplicated or rejected.
- Assignment of a `tournamentId` and `roundId` when created from a `TournamentRoomAssigned` event: this binding is immutable once set.

**Why strong consistency is required here:** Room membership is the basis for dealing hands and determining turn order. A race condition on join/leave could result in a player being dealt into a game they were not present for, or a player being excluded from a game while still counted. Either scenario violates the basic fairness invariant.

---

#### 7.1.1.3 Match (Within Room Gameplay Context)

The Match is a coordination structure owned by the Room aggregate. It tracks wins per player across up to three Games.

**What is strongly consistent:**
- The count of games completed within the match (cannot exceed 3, per A15).
- The per-player win count within the match.
- The match status (in_progress, completed).
- The determination of `MatchCompleted`: exactly one `MatchCompleted` event is emitted per Match, and only after the final game's result is recorded.

**Why strong consistency is required:** A race condition on match completion (e.g., two `GameCompleted` events processed concurrently) could emit two `MatchCompleted` events, triggering double tournament advancement. Since `MatchCompleted` crosses context boundaries and drives the Tournament Round Progression Saga (Section 7.3.1), a duplicate emission would corrupt tournament state.

---

#### 7.1.1.4 Session Aggregate (Identity & Session Context)

**What is strongly consistent:**
- The single-active-session invariant (V8): at most one session is active per player at any instant.
- Session creation atomically invalidates any prior active session for the same player. This is a compare-and-swap operation in domain terms: "if no current active session exists (or the existing session has `sessionId` X), create new session; simultaneously mark session X as invalidated."
- Session token validation: any context accepting commands may validate a session token. A token that has been invalidated must not be accepted, regardless of when the invalidation event propagates to downstream contexts (see Section 7.1.2 for the propagation lag discussion).

**Why strong consistency is required:** The single-active-session invariant is a security property (A29). A race condition where two concurrent logins both succeed would allow two independent actors to issue game commands on behalf of the same player simultaneously, violating the single-writer rule on the Game aggregate and enabling potential cheating.

**Atomicity in domain terms:** The IS context processes new-session commands with a guard: "read current active session for playerId; if one exists, emit `SessionInvalidated` for it; emit `SessionEstablished` for the new session." These two events are causally linked. No observer should see `SessionEstablished` without also eventually seeing `SessionInvalidated` for the prior session.

---

#### 7.1.1.5 Tournament Aggregate and Tournament Round (Tournament Orchestration Context)

**What is strongly consistent:**
- Tournament status transitions: `registration_open` → `in_progress` → `completed`.
- The count of completed rooms within a round (used to determine if `AllMatchesInRoundCompleted` can be emitted).
- The set of advancing players per round (determined atomically once all room results are recorded).
- The creation of the next round or final room: exactly one `TournamentRoundCreated` or `FinalRoomCreated` event is emitted per round completion.

**Why strong consistency is required for the room counter:** The hard synchronization requirement for tournament round progression (V16 and the Tournament Round Progression Saga in Section 7.3.1) means that `AllMatchesInRoundCompleted` must be emitted exactly once, when and only when all rooms have reported. A missed increment (lost event) would block advancement forever; a double increment (duplicate event) would trigger premature advancement. The room completion counter is therefore updated atomically, with idempotency protection against duplicate `RoomCompleted` events (Section 7.2.2).

---

### 7.1.2 Eventual Consistency: Cross-Context

Across context boundaries, UnoArena uses eventual consistency via at-least-once event delivery (A1). This means:
- An event emitted by one context will be delivered to all subscribing contexts at least once.
- Delivery is not instantaneous; there is a lag (typically milliseconds to seconds, but unbounded in degraded conditions).
- Each consumer must be idempotent (A1): receiving the same event twice must produce the same outcome as receiving it once.

The trade-off accepted here is that read models and derived state in downstream contexts may be momentarily stale. For UnoArena, this is acceptable for the flows listed below, because staleness does not violate the fairness invariant of the game in progress: the authoritative state lives in the upstream context, and downstream contexts are derivative.

The following table enumerates all cross-context eventual consistency flows, the lag risk, and the justification for accepting eventual (rather than strong) consistency.

| Event Flow | From → To | Consistency Level | Lag Risk | Why Eventual Is Acceptable |
|---|---|---|---|---|
| `GameCompleted` → Elo update | RG → RK | Eventual | Elo displayed as stale for a short period after game end | Elo is a statistical summary; being off by one game's delta for a few seconds does not affect the game in progress or any fairness invariant. |
| `GameCompleted` → Tournament advancement check | RG → TO | Eventual | Round cannot advance until TO processes; the delay is expected and tolerated | Advancement is a saga with a defined waiting state. The tournament is designed to wait. Correctness is maintained; only latency is affected. |
| `MatchCompleted` → Bracket update (spectator read model) | RG/TO → SV | Eventual | Bracket shows stale round results temporarily | The spectator bracket is a read model; it has no write-back capability. Staleness is cosmetic. |
| `PlayerForfeited` → Player elimination in tournament | RG → TO | Eventual | Tournament elimination slightly delayed | The player is already removed from their game; the elimination record in TO is a derived fact. No additional turns can be taken. |
| `GameCompleted` → Spectator view update | RG → SV | Eventual (with privacy ACL) | Spectators see slightly delayed state (A35) | The spectator view is a read projection. The slight delay is by design (A35) and serves as a privacy buffer. |
| `SessionInvalidated` → Force disconnect in active game | IS → RG | Eventual | Player's in-game commands may succeed for a brief window after the new login on the other device | See Section 7.1.3 for detailed analysis of this risk. |
| All events → Audit log | All contexts → AL | Eventual (within-context write-ahead first) | See Section 7.1.4 | See Section 7.1.4. |
| `EloUpdated` → Leaderboard and player profile | RK → SV/read models | Eventual | Leaderboard may show prior rating for minutes | The leaderboard is not authoritative for any gameplay decision. Staleness is cosmetic. |
| `TournamentRoundCreated` → Spectator bracket | TO → SV | Eventual | Bracket shows prior round temporarily | Same as above; cosmetic. |
| `PlayerSuspended` / `PlayerBanned` → Active game | IS → RG | Eventual | A suspended player may complete their current turn | Administrative actions are not expected to be instantaneous mid-game. The suspension is effective from the next command. |

---

### 7.1.3 Special Analysis: SessionInvalidated → Forced Disconnect Lag

When a player logs in on a new device (triggering `SessionInvalidated` for the prior session), Room Gameplay receives this event asynchronously. During the delivery lag, the old session's player may still submit game commands, and if those commands carry a valid (not-yet-expired) session token and the correct sequence number, they may be accepted.

**Why this risk is accepted:**
- The window is bounded by the event delivery lag (typically sub-second; at worst a few seconds).
- The commands are valid game commands; they are not cheating in a domain sense. The player issued them while connected.
- Once `SessionInvalidated` is processed by RG, the next command from the old session is rejected (the session token is now known to be invalid, per IS's authority).
- The 60-second reconnection timer begins from `PlayerDisconnected`, which is emitted once RG processes `SessionInvalidated`. Since the session is invalid, no reconnection is possible, and the player forfeits after 60 seconds (Section 7.3.4).

**Mitigation (domain level):** RG validates session tokens against IS synchronously for the first command after a reconnect attempt. This catches the stale-session case without blocking normal gameplay. The brief window of accepted stale-session commands does not grant any persistent advantage, because the player will be disconnected within seconds.

---

### 7.1.4 Special Case: Audit Log "Write Before Broadcast" Requirement

The specification mandates: "every state change is appended to an immutable game log before being broadcast" (V12). This creates a hard ordering constraint within the Room Gameplay context. The design resolves this as follows.

**Option A (Synchronous cross-context write):** RG writes to AL synchronously before broadcasting. This satisfies the letter of the requirement but makes gameplay availability dependent on AL's availability. If AL is slow or unavailable, no cards can be played.

**Option B (Write-ahead within RG, async propagation to AL):** RG maintains its own internal Game Log (owned and written by RG, not by AL). Every state mutation is appended to this Game Log synchronously before the mutation is considered committed and before any broadcast. The AL context asynchronously receives and archives these events. The Game Log in RG IS the authoritative record; AL is an archive consumer.

**Chosen approach: Option B.** The Game Log (owned by RG, internal to the Room Gameplay aggregate) serves as the write-ahead log. Broadcasting to clients and to downstream contexts (SV, TO, RK) occurs only after the Game Log write is confirmed within RG. The AL context is an eventual consumer of this log: it receives game log events via at-least-once delivery (A1) and deduplicates by `eventId`. This design:
- Satisfies V12 (write before broadcast) without depending on an external context.
- Maintains AL's role as the durable archive for dispute resolution and replay.
- Eliminates a cross-context availability dependency on the critical path of every game turn.

**Trade-off accepted:** There is a window (delivery lag) during which a state change has been applied and broadcast, but has not yet been archived in AL. If AL is unavailable for an extended period, events may pile up. Correctness is not compromised (the Game Log in RG is the source of truth), but the AL archive will be incomplete until catch-up occurs. Since AL is append-only and idempotent (Section 7.2.2), catch-up after AL recovery is safe.

---

## 7.2 Idempotency and Deduplication Strategy

Given at-least-once event delivery (A1) and at-most-once command delivery enforced by sequence numbers (A2), every processing boundary must be designed to handle duplicates without producing incorrect outcomes.

---

### 7.2.1 Command Idempotency (Client → Server)

Every command issued by a client carries two identifiers:
- **`commandId`**: A UUID generated by the client for this specific operation. This is the idempotency key. If the client retries (because it did not receive an acknowledgment), it uses the same `commandId`.
- **`sequenceNumber`**: The per-game ordering primitive. It represents the version of Game aggregate state the client expects to be operating against.

**Why both are needed:** The `sequenceNumber` alone cannot serve as an idempotency key, because its purpose is to detect stale commands, not to deduplicate. A command with sequence number 42 that has already been processed must return the original result, not be re-executed. But the `sequenceNumber` for the next command is 43, so checking "has sequence 42 been processed?" requires storing the result keyed by something other than the advancing sequence counter. The `commandId` serves this purpose.

**Deduplication rules (applied in this order):**
1. Check the idempotency store for the `commandId`.
2. If found: return the previously recorded response without re-executing any logic. The sequence number is not re-checked.
3. If not found: proceed to sequence number validation.
4. If the sequence number is not `currentSequenceNumber + 1`: reject with HTTP 409.
5. If the sequence number is correct: execute the command, append to Game Log, increment sequence number, record the result in the idempotency store keyed by `commandId`, return the result.

**Why idempotency check precedes sequence number check:** A retry with the same `commandId` arrives after the command has already succeeded and the sequence number has advanced. If the sequence number check ran first, the retry would produce a 409 Conflict (stale), which is incorrect -- the command did succeed. The idempotency check must run first to short-circuit and return the original success response.

**Edge case -- same commandId, different sequenceNumber:** This indicates a client bug (the client reused a `commandId` for a different logical command). The idempotency store wins: the stored result for the original `commandId` is returned. The sequence number in the retry is ignored. This is a defensive behavior; clients must generate fresh UUIDs for each distinct command.

**Idempotency store scope and lifetime:**
- Scoped per Game aggregate (not per player, not global). Each Game maintains its own store.
- Entries are retained for the duration of the game. Once `GameCompleted` is processed and the aggregate is sealed, the idempotency store may be discarded. There is no operational need to retain command idempotency records beyond the game lifecycle, since replays of past games are handled via the Game Log (event replay), not by re-executing commands.

---

### 7.2.2 Event Idempotency (Consumer Side)

Every domain event carries an `eventId` (a UUID assigned at emission time by the source context). Each consumer maintains a deduplication store keyed by `eventId`. Before processing any event, the consumer checks this store. If the `eventId` is present, the event is discarded. If absent, the event is processed and the `eventId` is recorded.

This is a direct consequence of A1 (at-least-once delivery): any consumer that lacks idempotency protection will produce incorrect outcomes when the same event is delivered more than once.

The following table identifies the most critical idempotency requirements, ordered by severity of failure if deduplication is absent.

| Consumer Context | Event | Idempotency Key | Consequence of Processing a Duplicate |
|---|---|---|---|
| Ranking & Statistics (RK) | `GameCompleted` | `gameId` (per player) | Double Elo update: player rating is shifted twice. Serious -- corrupts the rating system. |
| Tournament Orchestration (TO) | `MatchCompleted` | `matchId` | Double match recording: advancement evaluated on inflated win counts. Serious -- could advance wrong players or trigger premature round close. |
| Tournament Orchestration (TO) | `RoomCompleted` | `roomId` | Room counted twice against the total; could trigger `AllMatchesInRoundCompleted` prematurely. Serious. |
| Audit & Game Log (AL) | Any event | `eventId` | Duplicate log entry: audit trail contains the same event twice. Serious for replay integrity (replay would apply the same mutation twice). |
| Room Gameplay (RG) | `TournamentRoomAssigned` | `roomId` | Duplicate room creation attempt. If not idempotent, could create two rooms for the same assignment. Serious. |
| Tournament Orchestration (TO) | `PlayerForfeited` | `(playerId, gameId)` | Double elimination: could fail gracefully if the elimination check is "mark if not already eliminated," but the downstream `PlayerEliminated` event could be emitted twice. Minor if downstream consumers of `PlayerEliminated` are also idempotent. |
| Room Gameplay (RG) | `SessionInvalidated` | `(sessionId, playerId)` | Double force-disconnect: `PlayerDisconnected` emitted twice. If RG checks player connection state before emitting, the second emission is a no-op. Minor. |
| Spectator View (SV) | Any gameplay event | `eventId` | Duplicate UI update: spectator sees a card played twice or a turn advanced twice. Minor -- can be corrected by the client if it tracks `eventId`. |

**Detailed deduplication for the Ranking context (highest risk):**

The Elo update pipeline requires a two-level idempotency guard:

1. **Game-level guard:** Before processing any `GameCompleted` event, RK checks a `processed_games` set for the combination `(playerId, gameId)`. If present, skip entirely.
2. **Atomic persist:** After computing Elo deltas, the new rating and the `(playerId, gameId)` entry in `processed_games` are persisted together as a single atomic domain operation. "Atomic" here means: either both the new rating and the processed-games record are saved, or neither is. If the process crashes between these two writes, the game is not in `processed_games`, so the `GameCompleted` event is reprocessed on the next delivery, recomputing the delta from scratch.

**Why the per-player scope (not per-game) matters:** A single `GameCompleted` event triggers Elo updates for all N players in the game. If the Elo computation crashes after updating players 1 through 3 but before updating players 4 through N, the event will be redelivered. Players 1-3 already have their gameId in `processed_games`, so their updates are skipped. Players 4-N receive their updates. This is correct behavior: partial failure is recoverable without double-updating already-processed players.

**Deduplication for Room Gameplay (TournamentRoomAssigned):**

Room creation from `TournamentRoomAssigned` must be idempotent by `roomId`. If the event is delivered multiple times (at-least-once, A1), the Room aggregate checks whether a Room with the given `roomId` already exists. If yes, the duplicate is acknowledged and discarded -- the existing room is not modified. This means `roomId` in tournament rooms must be deterministic and assigned by TO at the time the event is emitted (not generated by RG on receipt), so that retries produce the same `roomId`.

**Deduplication for the Audit & Game Log:**

The AL context is the most sensitive to duplicate event entries, because the game log supports deterministic replay. The deduplication key is `eventId`, which is globally unique per event emission. However, AL must also verify **causal ordering**: events for a single game must be archived in their emission order (by sequence number within the game). Out-of-order delivery is accepted by AL but the archive index must be ordered by sequence number, not by arrival order. The deduplication check prevents the same event from appearing at two different positions in the log.

---

### 7.2.3 Draw Pile Reshuffle Idempotency

When the draw pile is exhausted mid-game (A11), a reshuffle is triggered: the discard pile (minus the current top card) becomes the new draw pile, shuffled with a new server-generated seed.

**Idempotency requirement:** The reshuffle event carries a `reshuffleSeed` that is generated once and appended to the Game Log at the moment the reshuffle is triggered. If the command that triggered the draw (and thus the reshuffle) is retried via the same `commandId`, the idempotency store returns the original result -- no second reshuffle occurs. If the command processing crashes after the Game Log write (reshuffle seed recorded) but before the response is sent, the recovery path reads the Game Log and uses the already-committed `reshuffleSeed`. The same seed is used. This ensures:
- The deck state is deterministic and auditable (V12, V11).
- Retry does not generate a different deck order.
- The game log contains the full information needed to replay the game from any checkpoint.

---

## 7.3 Saga Patterns and Long-Running Processes

A saga is a sequence of domain operations across multiple aggregates or contexts, where each step produces an event that triggers the next step, and where the saga as a whole may need to handle partial failure and compensation. UnoArena contains four primary sagas and one pipeline.

---

### 7.3.1 Tournament Round Progression Saga

This is the primary long-running saga in UnoArena. It is owned by the Tournament Orchestration context and coordinates the lifecycle of an entire tournament round.

**Saga state machine:**

```
[TournamentStarted | RoundCompleted]
        |
        v
  [CreateRound]
  TournamentRoundCreated emitted
        |
        v
  [DistributePlayersIntoRooms]
  (seeding algorithm per A20, A20a)
        |
        v
  [For each room: emit TournamentRoomAssigned]
  ← RG creates room, idempotent by roomId
        |
        v
  [Saga waits -- tracking completion counter]
  ← receives MatchCompleted, PlayerForfeited, RoomCompleted
        |
        v (when completedRooms == totalRooms)
  [AllMatchesInRoundCompleted] emitted
        |
        v
  [Evaluate advancement: top 3 per room, apply tiebreaks V3/V4]
  PlayerAdvanced, PlayerEliminated emitted per player
        |
        v
  [If remainingPlayers > 10] ──────→ loop to [CreateRound]
        |
  [If remainingPlayers <= 10]
        |
        v
  [CreateFinalRoom]
  FinalRoomCreated emitted
  TournamentRoomAssigned emitted (single room)
        |
        v
  [Wait for RoomCompleted (final room)]
        |
        v
  [TournamentCompleted emitted, final placements recorded]
```

**Critical synchronization point:** The transition from "waiting for rooms" to "AllMatchesInRoundCompleted" is a hard synchronization barrier (V16). The saga maintains a counter `{completedRooms: N, totalRooms: M}` within the Tournament Round aggregate. This counter is incremented atomically each time a valid, non-duplicate `RoomCompleted` event is processed. The counter is the single source of truth for whether the round is done.

**Handling out-of-order and duplicate RoomCompleted events:**
- The counter increment is protected by event-level idempotency (Section 7.2.2 -- key: `roomId`).
- A `RoomCompleted` event that has already been recorded is a no-op on the counter.
- There is no ordering requirement on which room completes first. The saga accepts room completions in any order.

**Compensation: stuck rooms (per Q7 / working assumption):**
A room may fail to complete within the configured round timeout (default: 2 hours). This blocks the entire round, which in a large tournament means thousands of players are waiting. The compensation mechanism is:

1. At `roundTimeout * 0.75`: emit `RoundTimeoutWarning` (monitoring signal, no state change).
2. At `roundTimeout`: the Tournament Orchestration saga emits a `ForceResolveTimedOutRoom` command to the affected room's Game aggregate in RG.
3. RG processes `ForceResolveTimedOutRoom`: the current in-progress game is ended using existing standings (wins from completed games; card point totals from the in-progress game as of the force-resolve moment). A `GameCompleted` event is emitted with `reason: timeout_forced`, followed by `MatchCompleted` with `reason: timeout_forced`, followed by `RoomCompleted`.
4. The saga processes this `RoomCompleted` normally. The advancement evaluation uses the forced results.
5. `PlayerEliminated` and `PlayerAdvanced` events are emitted normally.

**Consequence of forced resolution:** Players in the timed-out room may receive a different outcome than they would have if the game had played to completion. This is a deliberate trade-off: the alternative (blocking the entire tournament indefinitely) is unacceptable for 1,000,000 concurrent players. The `reason: timeout_forced` field on the emitted events allows the Audit context and monitoring systems to flag these outcomes for review.

**Compensation: failed room creation (TournamentRoomAssigned not processed):**
If RG fails to create a room after TO emits `TournamentRoomAssigned`:
- The event will be redelivered (at-least-once, A1).
- Room creation is idempotent by `roomId` (Section 7.2.2).
- Repeated retries eventually result in the room being created.
- The saga's completion counter for that room remains at zero until `RoomCompleted` is received. The saga simply continues to wait.
- If retries exhaust a reasonable bound without success (an operational concern), the room enters a `RoundTimeoutWarning` state and eventually receives a forced resolution.

**No saga rollback:** Once a round has started and some rooms have completed, there is no rollback. Completed rooms' results are irrevocable. A partially completed round is paused (not cancelled) while waiting for remaining rooms. The only mechanism that terminates a tournament before `TournamentCompleted` is the administrative `CancelTournament` command, which is an explicit domain action with its own event (`TournamentCancelled`) that notifies all participants and contexts.

---

### 7.3.2 Match Lifecycle Saga (Within Room Gameplay Context)

Within a single Room, the Match lifecycle is a mini-saga that is entirely within the RG context. Because it is intra-context, it benefits from the strong consistency of the Room aggregate and does not require cross-context coordination or external compensation.

**Saga steps:**
```
[StartMatch command received, game count = 1]
        |
        v
  [Game 1: GameStarted → gameplay → GameCompleted]
  wins-per-player updated in Match
        |
        v
  [CheckMatchProgress]
  → if top 3 advancement not locked in (per Q2 working assumption: always play all 3):
        |
        v
  [Game 2: GameStarted → gameplay → GameCompleted]
  wins-per-player updated
        |
        v
  [CheckMatchProgress]
  → if not all 3 games played:
        |
        v
  [Game 3: GameStarted → gameplay → GameCompleted]
        |
        v
  [MatchCompleted emitted]
  [RoomCompleted emitted if match is the only/final match]
```

**Recovery from crash mid-game:** The Room aggregate re-reads its owned Game aggregate's state from the Game Log. The Game aggregate is rebuilt by replaying its log entries from the beginning (event sourcing pattern). No lost state: every committed mutation is in the log (V12). The game resumes from the last logged event.

**No external compensation needed:** All state is within RG. If an inconsistency is detected (e.g., a sequence number gap in the Game Log), the Game Log is the source of truth and the aggregate state is rebuilt from it. Any commands received after a crash are re-evaluated against the rebuilt state.

---

### 7.3.3 Disconnection → Reconnection → Forfeit Saga

This saga governs what happens when a player loses their connection mid-game. It is owned by the Room Gameplay context.

**Saga steps:**
```
[Server detects connection loss (A4: within 5-10 seconds)]
        |
        v
  [PlayerDisconnected emitted]
  [ReconnectionTimer started: 60 seconds (V9)]
        |
        ├── [During window: player's turn comes up]
        │         → TurnSkippedDueToDisconnection emitted
        │         → Turn advances to next player (game continues)
        │
        ├── Path A: [Reconnect command received within 60s]
        │         → Session token validated (A29)
        │         → If token valid: PlayerReconnected emitted
        │           ReconnectionTimer cancelled
        │           Player resumes with original hand intact (V19)
        │         → If token invalid: reconnection rejected; timer continues
        │
        └── Path B: [60-second timer expires (V9, V18)]
                  → If currently the disconnected player's turn:
                    PlayerForfeited immediately
                  → If not their turn:
                    PlayerForfeited on next time it would be their turn
                  → [If tournament room] → PlayerEliminated emitted in TO
                  → [If casual room] → Game continues with remaining players
                    (if only 1 player remains, GameCompleted with that player as winner)
```

**Handling the intersection with turn timer (A38):** If the disconnected player's turn arrives during the reconnection window, the server does not run the turn timer for that player. Instead, the turn is immediately skipped (`TurnSkippedDueToDisconnection`). The turn timer (A38) only applies to connected players.

**Compensation:** Forfeit is a terminal and irreversible domain event. There is no mechanism within the domain to undo a forfeit. If a forfeit was issued in error (e.g., a bug caused premature timer expiry), the correction path is administrative: a `ForfeitRevoked` administrative event is appended to the Game Log with a manual override flag and an audit reference. This is an exceptional operation outside normal domain flow and requires human authorization.

**Deduplication:** `PlayerDisconnected` carries the `(playerId, gameId)` as a natural key. If the event is redelivered to TO (in a tournament room), the second delivery is a no-op because the player is already marked as eliminated (or the elimination is pending). The `PlayerForfeited` event is similarly idempotent by `(playerId, gameId)`.

---

### 7.3.4 Session Invalidation → Forced Disconnect Chain

This saga spans two contexts (Identity & Session and Room Gameplay). It is triggered by a player logging in on a new device, which invalidates their prior session.

**Full chain:**
```
[Player logs in on new device]
        |
        v
  [IS: SessionEstablished (new session)]
  [IS: SessionInvalidated (prior session) emitted]
        |
        ↓ (at-least-once delivery, A1)
  [RG: SessionInvalidated received]
        |
        v
  [RG: Look up active game for playerId]
        |
        ├── [Player not in any active game]
        │         → No further action in RG
        │
        └── [Player is in an active game]
                  → PlayerDisconnected emitted (reason: session_invalidated)
                  → ReconnectionTimer started (60 seconds)
                  → Reconnection timer tick runs...
                  → Player attempts reconnect with new session token → rejected:
                    the new session is for the new device (a different game room
                    cannot be joined; per A37, the old game is still active)
                  → No valid reconnect possible
                  → Timer expires → PlayerForfeited
                  → [If tournament] → PlayerEliminated in TO
```

**Key subtlety:** The 60-second reconnection window begins from the moment RG processes `SessionInvalidated`. During this window, the player's turns are skipped. The player on the new device cannot reconnect to the old game (the game requires the player to be "the player who was already in the room"; a new-session reconnect to an in-progress game requires the same `playerId` and a valid session, but by A37 the old game is the active room, so the new device's attempt to "join" would be rejected as "player already in room"). The net effect: the forfeit is guaranteed 60 seconds after RG processes the `SessionInvalidated` event, regardless of what the player does on their new device.

**Why eventual consistency is acceptable here:** The brief window between `SessionInvalidated` emission and RG processing it may allow a few more game commands from the old device's session. These commands are legitimate actions (the player was in the game); they do not represent cheating. The window is bounded (sub-second to a few seconds). The eventual forfeit is guaranteed. This is an accepted trade-off between availability and strict session termination.

---

### 7.3.5 Elo Update Pipeline

The Elo update pipeline is not strictly a saga (it does not require compensation or rollback), but it is a multi-step, eventually-consistent pipeline with critical idempotency requirements.

**Pipeline steps:**
```
[GameCompleted event emitted by RG]
        |
        ↓ (at-least-once delivery, A1)
  [RK: Event received]
        |
        v
  [Filter check]
  → roomType == "casual"? If not: discard (V5).
  → abandoned == false? If abandoned: discard (V7).
        |
        v
  [Per-player idempotency check]
  → For each player in placements:
      Is (playerId, gameId) in processed_games? If yes: skip this player.
        |
        v
  [Fetch current ratings for all players not yet processed]
        |
        v
  [Compute Elo deltas]
  Using multi-player Elo (A25): average pairwise expected-vs-actual comparisons.
  K-factor = 32 (A24).
  Starting rating for new players = 1200 (A26).
        |
        v
  [Apply rating floor]
  newRating = max(100, computedRating) (A27, V27)
  If floor applied: RatingFloorApplied event logged for monitoring.
        |
        v
  [Atomic persist per player]
  For each player: atomically write (newRating, processed_games += (playerId, gameId))
  These two writes are atomic in domain terms (either both succeed or neither does).
        |
        v
  [EloUpdated emitted per player]
  { playerId, gameId, oldRating, newRating, delta, timestamp }
        |
        ↓
  [Leaderboard and PlayerStatisticsUpdated read models refreshed]
  [AL: EloUpdated recorded in audit trail]
```

**Recovery from partial failure mid-pipeline:** If the pipeline crashes after atomically persisting some players but before persisting others, the `GameCompleted` event will be redelivered. For already-persisted players, the idempotency check (first step per player) short-circuits -- they are in `processed_games`. For not-yet-persisted players, the pipeline continues normally. This ensures exactly-once Elo updates per player per game, even under failure.

**Bounded inconsistency on retry:** If a player played another game between the crash and the retry (their rating changed), the Elo delta computed on retry uses the new rating as the baseline. This is a known limitation: the delta will be slightly different from what it would have been at the time of the original game. The error is bounded by the K-factor (at most 32 rating points off), is transient, and self-corrects on the next game. This is accepted as an inherent limitation of eventual consistency in rating pipelines, not a domain-level correctness failure.

**Tournament games are filtered:** The pipeline's first check (`roomType == "casual"`) ensures that tournament games never flow into the Elo update path, regardless of how the `GameCompleted` event is structured. This is the RK context's responsibility (per the Conformist relationship defined in Section 2.2); it does not depend on RG to exclude tournament events.

---

## 7.4 Invariant Enforcement Strategy

---

### 7.4.1 Intra-Aggregate Invariants (Enforced Synchronously)

These invariants are enforced within a single aggregate on every command, before any state mutation or event emission. Violation results in immediate command rejection. No event is emitted for a rejected command (only a `CommandRejected` response is returned to the caller).

| Invariant | Aggregate | Enforcement Mechanism |
|---|---|---|
| Only the current player may play a card or draw | Game | `command.playerId == game.activePlayerId` check before processing `PlayCard` or `DrawCard`. |
| The played card must be a legal play | Game | Card validation: `playedCard.color == discard.effectiveColor OR playedCard.value == discard.topValue OR playedCard.isWild`. Special card rules (Skip, Reverse, DrawTwo, Wild, WildDrawFour) also validated. |
| The command sequence number must be the next expected value | Game | `command.sequenceNumber == game.currentSequenceNumber + 1`. Violations yield HTTP 409. |
| The player's hand must contain the played `cardId` | Game | Hand membership check: `game.handsById[playerId].contains(command.cardId)`. |
| The challenge window may only be in one state at a time | Game | State machine: `CLOSED → OPEN → (CHALLENGED → RESOLVED | EXPIRED)`. Only valid transitions allowed. A challenge can only be issued in `OPEN` state. |
| A challenge can only be made during the open challenge window, before the next player acts | Game | Challenge command rejected if `challengeWindowStatus != OPEN`. Per A8: if the next player's turn is already being processed, the challenge window is closed first; the challenge arrives too late. |
| The Uno call challenge window timing is server-authoritative | Game | Server clock starts the 5-second timer at `UnoChallengeWindowOpened` event time (A7). Client-reported time is not accepted. |
| A room in `waiting` or `ready` state may not start a match until minimum player count (2) is reached | Room | `room.playerCount >= 2` checked on `StartMatch` command. |
| A room may not accept new players if not in `waiting` state | Room | `room.status == waiting` checked on `JoinRoom`. |
| A match has at most 3 games | Match (within Room) | `match.gameCount < 3` checked before starting a new game. `MatchCompleted` is emitted when the 3rd game completes (or when the outcome is determined, per Q2 working assumption). |
| Single active session per player | Session (IS) | Atomic compare-and-swap on session creation: read current active session; if exists, invalidate it atomically before creating the new one. |
| Elo rating floor of 100 | PlayerRating (RK) | `newRating = max(100, computedRating)` applied on every Elo update. |
| Game Log is append-only | GameLog (RG) | No update or delete operations are permitted on the Game Log. Every write is an append. Any attempt to modify or delete a log entry is rejected at the domain boundary. |
| The draw pile reshuffle seed is generated once and immutable | Game | The seed is written to the Game Log on first reshuffle trigger; subsequent retries use the logged seed, not a newly generated one. |
| A player can be in at most one active room | Room + Session | On `JoinRoom`: cross-check against `PlayerRoomMembershipReadModel`. If player is already in an active room, the command is rejected with `PlayerAlreadyInActiveRoom`. |
| Tournament room players are assigned by TO; players cannot self-join tournament rooms | Room | Tournament rooms are created with a fixed player list from `TournamentRoomAssigned`. `JoinRoom` commands are rejected for rooms with `roomType == tournament`. |
| The auto-draw and auto-pass on turn timer expiry count as a game action | Game | When the turn timer (A38) expires, the server issues an internal `AutoDraw` and/or `AutoPass` command. These are appended to the Game Log and treated identically to player-issued commands. |

---

### 7.4.2 Cross-Aggregate Invariants (Enforced Eventually)

Some invariants span multiple aggregates or contexts and cannot be enforced atomically. These are detected reactively and resolved via saga compensation or idempotency.

| Invariant | How Detected | Recovery / Resolution |
|---|---|---|
| All rooms in a round complete before round advancement | Round completion counter in Tournament Round aggregate (TO) | Saga waits (Section 7.3.1). `RoundTimeoutEscalation` for rooms that do not complete in time. |
| Elo is updated at most once per player per game | `processed_games` idempotency set in RK per player | Idempotency check at the start of the Elo update pipeline (Section 7.2.2). |
| A player is in at most one active room at any time | `PlayerRoomMembershipReadModel` (eventually consistent read model) | Checked synchronously on `JoinRoom`. Eventual consistency lag is tolerated: the read model is updated immediately on `PlayerJoinedRoom` or `RoomCompleted`, and the join check reads from it synchronously. |
| Each `TournamentRoomAssigned` event results in exactly one room being created in RG | Idempotent room creation by `roomId` in RG | At-least-once delivery retries until room exists; duplicate attempts are no-ops (Section 7.2.2). |
| `MatchCompleted` from each room in a tournament round reaches TO exactly once | Idempotency in TO keyed by `matchId` | Section 7.2.2. |
| A player eliminated from a tournament cannot be advanced | Tournament Bracket state in TO | `PlayerEliminated` is a terminal state for that `(playerId, tournamentId)` pair. Any subsequent `PlayerAdvanced` for the same pair is rejected as an invariant violation and logged to the Audit trail. |
| Elo is never updated for tournament games | RK filters `GameCompleted` by `roomType` | The filter is the enforcement mechanism. If a `GameCompleted` event lacks `roomType` (a malformed event), it is rejected and logged to the Audit trail as a `MalformedEventDiscarded` signal. |

---

### 7.4.3 Detecting and Surfacing Invariant Violations

**Runtime enforcement (command rejection):** When a command violates an intra-aggregate invariant (Section 7.4.1), the aggregate emits a `CommandRejected` event with a structured reason code (e.g., `StaleSequenceNumber`, `NotCurrentPlayer`, `IllegalCardPlay`, `RoomNotWaiting`). This event is logged to the Audit trail and is available for real-time monitoring. It is NOT broadcast to other players (it is a private rejection response to the command issuer).

**Post-hoc verification (audit replay):** The Audit & Game Log context supports full deterministic replay of any game. An audit verification process can replay a game's log entries from the initial state, applying each mutation, and verify that the final reconstructed state matches the authoritative state at each recorded checkpoint. Discrepancies constitute evidence of invariant violations. This is the basis for dispute resolution and forensic analysis.

**Monitoring signal:** A spike in `CommandRejected` events for reason `StaleSequenceNumber` indicates that clients are submitting commands based on stale state -- possibly due to SSE delivery lag (A3) or a bug in client state management. This is a domain-level consistency health signal (Section 7.7).

---

## 7.5 Recovery Strategies by Failure Type

---

### 7.5.1 Game Aggregate State Loss (Mid-Game Process Crash)

**Scenario:** The server process handling a Game aggregate crashes after committing several events to the Game Log but before the aggregate's in-memory state is persisted elsewhere.

**Recovery:** The Game aggregate is rebuilt from its Game Log using event replay (event sourcing pattern). Events are replayed from the beginning of the game (or from the most recent snapshot, if snapshots are maintained as a performance optimization) to reconstruct the current state. This is safe because:
- Every committed mutation is in the Game Log (V12, Option B from Section 7.1.4).
- The Game Log is append-only and immutable; it is the single source of truth.
- The replay produces a deterministic result: the same events in the same order produce the same state.

**Events broadcast after crash but before recovery:** Per the design (Section 7.1.4), events are broadcast only after being written to the Game Log. If the crash occurred after a Game Log write but before the broadcast (or before the client acknowledgment), the event will be re-delivered from the Game Log on recovery (at-least-once delivery). Clients must be idempotent with respect to event replay (they track the last `eventId` received and discard duplicates, per A3).

**Turn timer recovery:** The turn timer deadline is stored in the Game Log as part of the `TurnAdvanced` event (or equivalent). On recovery, the aggregate reads the deadline from the log and determines whether the timer has already expired. If it has, the auto-draw/auto-pass is applied immediately as part of recovery. If not, the timer is re-armed for the remaining duration.

**Reconnection timer recovery:** Similarly, the `PlayerDisconnected` event in the Game Log carries the timer start time. On recovery, if the 60-second window has already elapsed, the forfeit is processed immediately.

---

### 7.5.2 Client State Divergence (Missed SSE Events)

**Scenario:** A client misses one or more events from the SSE stream (e.g., due to a brief network interruption or reconnect, per A3). The client's local state is now out of sync with the server's authoritative state.

**Detection:** The client detects divergence when it receives an event with a `sequenceNumber` or `eventId` that is higher than expected (indicating a gap). Alternatively, a command rejection with `StaleSequenceNumber` implies the client's view of the sequence counter is behind the server's.

**Recovery path A (snapshot):** The client requests the current game state snapshot from a designated snapshot endpoint. The snapshot represents the full authoritative state as of the server's latest committed event. The client replaces its local state with the snapshot and resumes normal event processing from that point forward.

**Recovery path B (event replay):** If the client knows the `eventId` of the last event it successfully processed, it can request event replay from that checkpoint. The SSE stream supports event replay from a specified event ID (A3). The client receives only the missed events and applies them sequentially to its local state.

Path A is simpler and more robust; Path B is more efficient for small gaps. The client chooses based on the size of the gap (if it received the last event more than N seconds ago, use a snapshot; otherwise, replay).

**Invariant maintained:** The server's authoritative state is never replaced by the client's stale state. The recovery is always client-side; the server's Game aggregate state is not affected.

---

### 7.5.3 Tournament Round Stuck (Room Never Completes)

**Scenario:** One or more rooms in a tournament round fail to complete their matches, blocking round advancement. This is the most availability-threatening failure mode for the tournament saga.

**Detection and escalation:**
1. The Tournament Round aggregate tracks the time elapsed since `TournamentRoundCreated`.
2. At `roundTimeout * 0.75` (e.g., 90 minutes): `RoundTimeoutWarning` emitted. This is a monitoring signal. No state change.
3. At `roundTimeout` (e.g., 120 minutes): the Tournament Orchestration saga identifies the non-completing rooms and initiates forced resolution.

**Forced resolution steps:**
1. TO emits `ForceResolveTimedOutRoom` for each non-completing room. This is delivered to RG as a command-like event.
2. RG processes the force-resolve: it terminates the in-progress game immediately. Current hand values are used as card point totals for tiebreaking. Prior completed games in the match contribute their normal results. A `GameCompleted` event is emitted with `reason: timeout_forced`, followed by `MatchCompleted` with `reason: timeout_forced`, followed by `RoomCompleted` with `reason: timeout_forced`.
3. TO processes `RoomCompleted` normally. The round completion counter is incremented. If this was the last outstanding room, `AllMatchesInRoundCompleted` is emitted and the saga proceeds.
4. Downstream contexts (RK, SV, AL) receive the forced-result events and process them normally.

**Downstream context behavior on timeout_forced:**
- RK: does not apply Elo updates for games marked `reason: timeout_forced` (tournament games are already excluded, per V5).
- AL: records the forced completion with the reason field for forensic audit.
- TO: records the advancement results. Players eliminated by force-resolution are marked with `eliminationReason: timeout_forced`.
- SV: displays the forced result to spectators.

**Player experience:** Players in a force-resolved room receive their placements based on the state at the moment of forced resolution. This may feel unfair to a player who was about to win. The system accepts this trade-off to preserve tournament liveness.

---

### 7.5.4 Message Queue Backlog (At Scale -- 100,000+ Simultaneous Rooms)

**Scenario:** In a large tournament round (e.g., 1,000,000 players producing ~100,000 rooms), all rooms in a round may complete within a short window. TO receives ~100,000 `MatchCompleted` and `RoomCompleted` events in quick succession.

**Domain-level consistency concern:** The round completion counter in the Tournament Round aggregate must reach exactly `totalRooms` before `AllMatchesInRoundCompleted` is emitted. Under high event volume, events may be delayed in delivery. This is not a correctness problem -- idempotency ensures each `RoomCompleted` increments the counter at most once -- but it is a liveness concern.

**Recovery:** There is no domain-level recovery needed here. The saga correctly waits for all events to be processed. The counter will eventually reach `totalRooms`, and the saga will advance. The only risk is latency between rounds, which is an availability concern (a slower tournament experience), not a correctness concern. The `RoundTimeoutEscalation` mechanism (Section 7.5.3) handles the degenerate case where some events never arrive.

**Idempotency ensures correctness under replay:** If a batch of events is redelivered (e.g., after a TO process crash), the idempotency guards on each `RoomCompleted` ensure the counter is not double-incremented. The counter is monotonically increasing and capped at `totalRooms`.

---

### 7.5.5 Elo Update Pipeline Failure During Write

**Scenario:** The Ranking & Statistics process crashes after computing Elo deltas for all players in a game, but before persisting the results.

**Recovery:** On restart, the `GameCompleted` event is redelivered (A1). For each player, the idempotency check (`processed_games` lookup) is performed. Since the crash occurred before the atomic persist, no player has their `gameId` in `processed_games`. The pipeline recomputes all deltas from scratch and re-persists.

**Scenario variant:** The crash occurs after atomically persisting the results for players 1-3 but before processing players 4-N (e.g., processing is done one player at a time). On recovery:
- Players 1-3 have their `(playerId, gameId)` in `processed_games`: their updates are skipped.
- Players 4-N do not: their updates are computed and applied.
- Result: all players' Elo is updated exactly once.

**Bounded drift on retry:** If a player played another game between the crash and the retry, their baseline rating has changed. The Elo delta computed on retry will differ slightly from the delta that would have been computed immediately after the original game. This is a bounded, self-correcting drift (bounded by the K-factor of 32). It is accepted as an inherent limitation of eventual consistency in rating systems. It is not a domain correctness failure.

---

### 7.5.6 Session Context Unavailability (IS Down)

**Scenario:** The Identity & Session context is temporarily unavailable. New logins cannot be processed. Existing sessions cannot be validated.

**Impact on in-progress games:** Commands from players with already-validated sessions (where RG has a cached session validity indicator) may continue to be accepted during a short grace period. The domain principle is that IS is the authority on session validity, but RG does not re-validate every single command against IS synchronously -- it validates on connection establishment and on reconnect. Commands within an established connection are assumed to be from a valid session.

**Impact on new connections and reconnects:** Reconnection attempts cannot be validated (session token validation requires IS). Reconnection attempts will be rejected, and the reconnection timer continues. If IS remains unavailable for the full 60 seconds, the player forfeits.

**Impact on new logins:** No new sessions can be established. Players who are not already logged in cannot join rooms or participate in games.

**No domain-level compensation is possible:** IS unavailability is an infrastructure failure. The domain model cannot compensate for the inability to authenticate. The 60-second reconnection window provides a natural grace period that absorbs short IS outages without affecting in-progress games.

---

## 7.6 Rate Limiting as a Consistency Mechanism

Rate limiting at multiple layers is not only a security measure but also a consistency protection mechanism. Unbounded command rates can overwhelm the sequence-number processing pipeline for a Game aggregate, cause the idempotency store to grow unboundedly, and starve other rooms of processing capacity.

The following rate limits are defined at the domain level (V20, A30). They are enforced at every context boundary before commands reach aggregate handlers.

| Layer | Scope | Rate Limit (default, A30) | Domain Consistency Purpose |
|---|---|---|---|
| Per-IP | All requests | (configurable per A30) | Prevents basic denial-of-service against command processing. |
| Per-Player | Game commands (PlayCard, DrawCard, CallUno, Challenge, Forfeit) | 10/second per player per room (A30) | Prevents command flooding that would overwhelm the sequence-number deduplication pipeline for a Game aggregate. |
| Per-Room | All commands to a specific Game aggregate | Derived from per-player * maxPlayers | Protects the Game aggregate from aggregate-level command flooding (e.g., all 10 players issuing at maximum rate simultaneously). |
| Per-Player | Room join requests | 5/minute (A30) | Prevents rapid join-leave cycles that could corrupt `PlayerRoomMembershipReadModel` consistency. |
| Per-Tournament | Registration commands | 1/second per player (A30) | Prevents registration abuse that could create invalid tournament states. |
| Per-Player | Casual room creation | 2/minute (A30) | Prevents creation of rooms that are immediately abandoned, polluting the `AvailableRoomsReadModel`. |
| Per-Player | Spectator subscriptions | 10/minute (A30) | Prevents spectator feed abuse that could generate excessive events in the SV context. |

**Adaptive throttling (V20):** Repeat offenders (players whose commands are consistently rejected for rate limit violations) receive progressively stricter limits. The Identity & Session context maintains the throttling state per player, and all contexts enforce IS-provided throttling directives. This is an IS responsibility delegated to all context boundaries.

**Turn timer interaction with rate limits:** The turn timer (A38) may cause a player to be auto-drawn and auto-passed. These auto-actions are server-initiated and are not subject to the per-player rate limit (they originate from the server, not the client). They are appended to the Game Log and broadcast normally.

---

## 7.7 Domain-Level Monitoring and Alerting

The following domain events serve as consistency health signals. These are observability hooks defined at the domain level; the specific monitoring infrastructure is out of scope. Each signal indicates a potential deviation from expected domain behavior.

| Signal Event | Emitted By | Meaning | Suggested Action |
|---|---|---|---|
| `CommandRejected(reason=StaleSequenceNumber)` spike | Game aggregate (RG) | Clients are submitting commands based on stale game state. Possible causes: SSE delivery lag (A3), client bug, or high network jitter. | Investigate SSE stream delivery latency. Check client reconnect logic. |
| `CommandRejected(reason=NotCurrentPlayer)` spike | Game aggregate (RG) | Clients are submitting commands out of turn. Possible causes: turn advancement events are not being received by clients (SSE lag), or a client-side bug in turn tracking. | Same as above. Correlate with `StaleSequenceNumber` rejections. |
| `ReconnectionTimerExpired` spike | Game aggregate (RG) | Elevated disconnection-to-forfeit rate. Possible causes: widespread network instability, DDoS against player connections, or a server-side connection detection bug (A4). | Activate rate limiting (Section 7.6). Investigate connection detection latency. Correlate with IP-level patterns for DDoS response. |
| `RoundTimeoutWarning` | Tournament Round (TO) | One or more rooms in a tournament round are at 75% of the timeout threshold. Possible causes: very slow players (turn timer should prevent this per A38), a bug in the auto-draw/auto-pass mechanism, or a Room aggregate that is stuck. | Human review of the flagged room(s). Verify turn timer is functioning. Escalate to `ForceResolveTimedOutRoom` if approaching hard timeout. |
| `ForceResolveTimedOutRoom` emitted | Tournament saga (TO) | A room was forced to resolve due to timeout. | Flag for post-tournament review. Investigate why the room stalled. If systematic, indicates a bug in the turn timer (A38) or the auto-forfeit mechanism. |
| `AbandonedGameSkipped` spike | Ranking pipeline (RK) | Unusual rate of games being abandoned (all remaining players forfeiting). Possible causes: collusion (players forfeiting to deny Elo changes), organized griefing, or a bug in the forfeit logic. | Flag for abuse review. Correlate `PlayerForfeited` events to identify repeated offender patterns. Trigger adaptive throttling (V20). |
| `DuplicateEventIgnored` spike | Any consumer (RK, TO, SV, AL) | The event delivery system is retrying excessively. The idempotency layer is absorbing duplicates, but the volume indicates an upstream delivery issue. | Check event bus health. Investigate the source of retries. If limited to a specific event type, investigate the producer. |
| `RatingFloorApplied` | Ranking pipeline (RK) | Players are hitting the Elo floor of 100 (A27). A low-to-moderate rate is expected for consistently losing players. A spike may indicate matchmaking degradation (e.g., new players being systematically matched against much stronger players). | Monitor matchmaking quality. If floor applications are heavily concentrated in a small cohort, investigate matchmaking parameters. |
| `MalformedEventDiscarded` | Any consumer | An event arrived without required fields or with invalid values (e.g., missing `roomType`). | Investigate the emitting context for a bug. Treat as a schema contract violation between contexts. Correlate with recent deployments. |
| `CommandRejected(reason=PlayerAlreadyInActiveRoom)` spike | Room aggregate (RG) | Players are attempting to join new rooms while already in active rooms. Could indicate a client-side bug in room membership tracking or an attempt to exploit a race condition in the membership check. | Verify `PlayerRoomMembershipReadModel` consistency. Investigate whether the read model is being updated correctly on room completion and player forfeit events. |
| `SessionInvalidated(reason=new_login)` followed immediately by `PlayerForfeited` spike | IS → RG chain | A pattern of players logging in on new devices mid-game, resulting in forfeit of the old game. Could indicate account sharing, or a client bug causing inadvertent re-logins. | Review session lifecycle patterns for affected players. May indicate UX issue (e.g., players accidentally triggering a new login). |

---

## 7.8 Summary of Consistency Guarantees

The following table provides a consolidated reference of the consistency model applied to each major domain operation.

| Domain Operation | Consistency Level | Enforcement Mechanism | Reference |
|---|---|---|---|
| Card play, draw, Uno call, challenge | Strong (within Game aggregate) | Sequence numbers, single-writer, synchronous invariant check | 7.1.1.1 |
| Turn timer expiry → auto-draw/auto-pass | Strong (within Game aggregate) | Server-initiated internal command, appended to Game Log | 7.4.1 |
| Room state transitions | Strong (within Room aggregate) | Single-writer, synchronous lifecycle check | 7.1.1.2 |
| Match game count tracking | Strong (within Room aggregate) | Synchronous count check before `StartMatch` | 7.1.1.3 |
| Session creation / invalidation | Strong (within Session aggregate) | Atomic compare-and-swap | 7.1.1.4 |
| Tournament round completion counter | Strong (within Tournament Round aggregate) | Atomic increment, idempotent by roomId | 7.1.1.5 |
| Elo update after casual game | Eventual | At-least-once delivery + idempotency store (gameId per player) | 7.2.2, 7.3.5 |
| Tournament advancement on MatchCompleted | Eventual | At-least-once delivery + idempotency store (matchId) | 7.2.2, 7.3.1 |
| Spectator view update | Eventual | At-least-once delivery + ACL privacy filter | 7.1.2 |
| Audit log archival | Eventual (after write-ahead in RG) | At-least-once delivery + idempotency store (eventId) | 7.1.4, 7.2.2 |
| Player elimination in tournament | Eventual | At-least-once delivery + idempotency on PlayerForfeited | 7.2.2 |
| Client state synchronization | Client-side recovery | Snapshot or event replay on divergence detection | 7.5.2 |
| Reconnection timer / forfeit | Eventual (timer owned by RG) | Timer state in Game Log; recovered from log on crash | 7.5.1, 7.3.3 |
| Draw pile reshuffle seed | Strong (within Game aggregate) | Seed written to Game Log before use; replays use logged seed | 7.2.3 |
| Game Log append-before-broadcast | Strong (within RG, Option B) | Write-ahead log; broadcast only after successful append | 7.1.4 |

---

*This document should be read in conjunction with [2. Bounded Contexts and Context Map](02-bounded-contexts-and-context-map.md), [5. Domain Event Flow Narratives](05-domain-event-flows.md), and [8. Assumptions and Open Questions](08-assumptions-and-open-questions.md). All assumption references (A1–A38) and open question references (Q1–Q12) correspond to the numbered items in Document 8.*
