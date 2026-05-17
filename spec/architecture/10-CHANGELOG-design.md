# 10 — CHANGELOG-design.md — Design Package Updates for Architecture Checkpoint

> **Scope:** This changelog documents every modification made to the UnoArena domain design artifacts after crossing them with the Architecture Checkpoint (`spec/requirements/2-architecture-checkpoint.md`) and the per-context architecture documents (`spec/architecture/03-bounded-contexts/*`). It follows the minimum-bar requirements of §6.2 of the Architecture Checkpoint assignment.
>
> **Format per entry:**
> 1. File path and document section modified.
> 2. Deliverable number and title from the Design Checkpoint §5.
> 3. What changed (delta) and why (the specific architecture or integration constraint that required it).
> 4. Confirmation that no Design Checkpoint non-negotiable domain guarantee was weakened.

---

## Summary: Did Any Design Edits Occur?

**Yes.** After mapping the domain design to the concrete microservices architecture, sixteen distinct change clusters were identified across six bounded contexts. Of these, **twelve require actual edits to domain files** (new events, updated invariants, enriched payloads, new commands); **four are architecture-only decisions** that the domain already covers correctly at its level of abstraction — those entries document the delta without modifying domain files.

> **Principle applied throughout:** Domain documents express *what* must hold (invariants, events, commands, ubiquitous language). Architecture documents express *how* those guarantees are implemented (service names, table names, protocols, infrastructure patterns). Mixing infrastructure vocabulary into domain files pollutes the ubiquitous language and creates coupling to implementation choices that should remain free to evolve.

None of the changes weaken any Design Checkpoint non-negotiable guarantee; in every case the guarantee is made *stronger and more operationally precise* by naming the architectural component that enforces it. The sections below document each cluster individually.

---

## Part I — Room Gameplay Context (RG)

### Change 1 — Explicit WDF Challenge Events Added to Outbound Event Catalog

**File:** `spec/domain/docs/02-bounded-contexts-and-context-map.md` — Section 2.1.2 (Events Produced)
**File:** `spec/domain/docs/04-commands-and-domain-events.md` — Section 4.3 (Room Gameplay events)

**Deliverable:** *Deliverable 4 — Commands and domain events catalog* | *Deliverable 2 — Bounded contexts and context map*

**Delta:** The domain design described the Wild Draw Four challenge window as a Value Object (`WildDrawFourChallengeWindow`, doc 03 §3.1.5) and its adjudication logic (INV-G-23, INV-G-24, INV-G-25) but omitted the corresponding domain events from the *published outbound catalog* of Section 2.1.2. The architecture explicitly lists the following four events in the `gameplay.events` Kafka topic:

| New Event | Trigger |
|-----------|---------|
| `WildDrawFourChallengeWindowOpened` | Server processes a Wild Draw Four card play |
| `WildDrawFourChallengeMade` | Affected player issues a challenge within the window |
| `WildDrawFourChallengeResolved` | Server adjudicates the WDF challenge |
| `WildDrawFourChallengeWindowClosed` | Window expires without challenge |

**Why:** The Spectator View ACL (architecture doc `spectator-view.md`) and the `audit-service` (architecture doc `audit-game-log.md`) both consume `gameplay.events` and require named, enumerable event types to apply transformations. An undeclared event type cannot be ACL-filtered or HMAC-verified. The integration table (§6.3) also requires explicit event names for the `gameplay.events → spectator-projection-service` row.

**Non-negotiable confirmation:** INV-G-23, INV-G-24, and INV-G-25 are unchanged. The adjudication logic, window timing, and the prohibition on revealing the hand snapshot to spectators are all preserved. The spectator ACL strips `challengedPlayerHandSnapshot` from `WildDrawFourChallengeResolved` before materialization, the same privacy rule as for `CardDrawn → SpectatorCardDrawn`.

---

### Change 2 — New Explicit Gameplay Events for Initialization and Turn Lifecycle

**File:** `spec/domain/docs/02-bounded-contexts-and-context-map.md` — Section 2.1.2 (Events Produced)
**File:** `spec/domain/docs/04-commands-and-domain-events.md` — Section 4.3

**Deliverable:** *Deliverable 4 — Commands and domain events catalog*

**Delta:** The following events were implied by the domain's state machine descriptions (Game phase: `Initializing`, turn timer auto-expiry, disconnection skip) but were not enumerated as published domain events. The architecture makes them explicit members of the `gameplay.events` Kafka topic:

| New Event | Previously Described As |
|-----------|------------------------|
| `DeckInitialized` | Implicit: "deck state is generated server-side and appended to the immutable game log before any card is dealt" (doc 02 §2.1.2 invariant) |
| `InitialHandsDealt` | Implicit: part of `Initializing` phase in Game aggregate |
| `FirstCardFlipped` | Implicit: INV-G-19 described the logic but not the event name |
| `TurnPassed` | Implied: INV-G-03 describes the Pass command but no resulting event was named in the outbound catalog |
| `TurnTimedOut` | Implicit: INV-G-16 described auto-draw/auto-pass on turn timer expiry but named no event |
| `TurnSkippedDueToDisconnection` | Implicit: INV-G-11 described disconnected player turn-skip but named no event |
| `DrawPileReshuffled` | Implicit: INV-G-17 described reshuffle logic with a new seed |

**Why:** The `game-engine` service writes to the transactional outbox in the same PostgreSQL TX as the event store append (architecture §Log-Before-Broadcast). Every state change must correspond to a named, typed event so the outbox relay, Kafka consumers, and the `audit-service` HMAC verifier can process them deterministically. Unnamed state transitions cannot satisfy INV-GL-02 (causal completeness).

**Non-negotiable confirmation:** INV-G-16 (turn timer auto-draw/auto-pass) and INV-G-17 (reshuffle idempotency) are unchanged. The new events are the *publication form* of already-specified domain behaviors. The domain invariants that govern *when* these transitions occur are not modified.

---

### Change 3 — `GameCompleted` Event Payload: `wasAbandoned` and `roomType` Fields Made Explicit

**File:** `spec/domain/docs/02-bounded-contexts-and-context-map.md` — Section 2.1.2 (Events Produced) and Section 2.3.1
**File:** `spec/domain/docs/04-commands-and-domain-events.md` — Section 4.3 (`GameCompleted` payload)

**Deliverable:** *Deliverable 4 — Commands and domain events catalog* | *Deliverable 2 — Bounded contexts and context map*

**Delta:** The domain design stated that `GameCompleted` includes "final placement order and card-point totals" (doc 02 §2.1.2) and that the Ranking & Statistics context filters by room type and abandonment status. However, the `GameCompleted` payload schema did not explicitly include:

- `wasAbandoned: boolean` — distinguishes a game where all remaining players forfeited (no Elo) from a game with a valid winner.
- `roomType: "casual" | "tournament"` — denormalized from the Room aggregate at game creation (already described in doc 03 §3.1.3 as a Game aggregate attribute) but not carried as a payload field in the published event.

The architecture (`room-gameplay.md` §Abandoned-Game Detection) makes these two fields the *canonical discriminators* consumed by `ranking-service` and `tournament-service`. Neither downstream context infers outcome from secondary signals.

**Why:** The Ranking & Statistics architecture requires that downstream consumers do not need to query Room state to determine Elo eligibility (§6.1 Integration Appropriateness). Embedding both fields in the event enables stateless, idempotent Elo filtering (architecture `ranking-statistics.md` §Elo Pipeline, steps 2–3). The `tournament-service` also needs `wasAbandoned` to correctly record forfeit losses for tournament advancement (§2 non-negotiable: abandoned casual games exclude Elo, tournament forfeits are losses).

**Non-negotiable confirmation:** The Elo scope non-negotiable is preserved and strengthened: `ranking-service` now has a first-class boolean field (`wasAbandoned`) rather than inferring abandonment from the absence of a winner. The tournament forfeit-as-loss rule is unchanged; `tournament-service` records all forfeited players as eliminated regardless of `wasAbandoned`.

---

### Change 4 — Sequence Number Exception for Time-Critical Commands

**File:** `spec/domain/docs/04-commands-and-domain-events.md` — Section 4.1.3 (Sequence Numbers)
**File:** `spec/domain/docs/03-aggregates-entities-value-objects.md` — Section 3.1.3, INV-G-05

**Deliverable:** *Deliverable 4 — Commands and domain events catalog* | *Deliverable 3 — Aggregates, entities, and value objects*

**Delta:** INV-G-05 stated that *every* command mutating Game state must carry the correct next `SequenceNumber`. The architecture (`room-gameplay.md` §Sequence-Number Enforcement) introduces an explicit exception:

> `CallUno`, `ChallengeUnoCall`, `ChallengeWildDrawFour` **do not require sequence numbers** — they are time-sensitive commands accepted outside the normal turn sequence, guarded by challenge-window state instead.

INV-G-05 is updated to read: "Every command that mutates Game state must carry the correct next SequenceNumber **except** commands whose validity is governed exclusively by a time-bounded challenge window (`CallUno`, `ChallengeUnoCall`, `ChallengeWildDrawFour`)."

**Why:** These commands may arrive from *any opponent* (not just the current player), and requiring a per-player sequence number for them would create a sequencing dependency that conflicts with the 5-second window. A late-arriving challenge rejected for a stale sequence number — rather than for the genuine reason that the window closed — would produce misleading error responses and degrade gameplay. The challenge-window state machine (INV-G-06, INV-G-24) already provides the equivalent ordering guarantee by only accepting challenges when the window is `Open`.

**Non-negotiable confirmation:** The concurrency control guarantee for card plays, draws, and passes is unchanged. Only the three time-window commands are exempted, and each of those is still gated by the server-authoritative window state (INV-G-06, INV-G-24). No illegal play can result from this exception.

---

### Change 5 — Log-Before-Broadcast: Architecture Names the Mechanism (No Domain File Change)

**Domain file changed:** *None.* The domain already covers this correctly.

**Architecture file:** `spec/architecture/03-bounded-contexts/room-gameplay.md` — §Log-Before-Broadcast

**Deliverable:** *Deliverable 7 — Consistency and recovery strategy* (for reference only; no edit needed)

**Delta:** The domain design (doc 07 §7.1.4) chose "Option B: write-ahead within RG, async propagation to AL" and correctly expresses the guarantee at the domain level: every authoritative state change is durably recorded before any broadcast. The invariant (V12, INV-GL-03) is complete. No domain vocabulary needs to change.

The architecture document names the concrete implementation of Option B — a transactional outbox — but that is an infrastructure pattern, not a domain concept. Adding table names (`game_events`, `outbox`), technology identifiers (PostgreSQL, Kafka), or polling intervals to the domain doc would couple the ubiquitous language to an implementation choice that may evolve independently.

**Why the domain doc stays unchanged:** The domain's formulation ("every state mutation is appended to the Game Log before the mutation is considered committed, and before any broadcast") is a precise, implementation-neutral expression of the invariant. Any concrete mechanism that satisfies that ordering is acceptable — the domain does not need to name it.

**Non-negotiable confirmation:** INV-GL-03 (pre-broadcast recording) is satisfied by the architecture's choice. The guarantee is unchanged and is now demonstrated to be implementable without weakening any domain invariant.

---

### Change 6 — Durable Domain Timers: Architecture Names the Firing Mechanism (No Domain File Change)

**Domain file changed:** *None.* The domain already covers this correctly.

**Architecture file:** `spec/architecture/03-bounded-contexts/room-gameplay.md` — §5-Second Uno! Challenge Window, §60-Second Reconnection Window

**Deliverable:** *Deliverable 3 — Aggregates, entities, and value objects* | *Deliverable 7 — Consistency and recovery strategy* (for reference only; no edit needed)

**Delta:** The domain design models timers as Value Objects inside the Game aggregate (`ChallengeWindow`, `WildDrawFourChallengeWindow`, `ReconnectionWindow`, `TurnTimer`). Each Value Object carries a `deadline` timestamp. The domain's crash-recovery section (doc 07 §7.5.1) already states that "the turn timer deadline is stored in the Game Log as part of the `TurnAdvanced` event" and that "on recovery, the aggregate reads the deadline from the log and determines whether the timer has already expired." This correctly expresses the domain-level guarantee: deadline state is durable and recoverable.

The architecture document names *how* expiry firing survives crashes — via a dedicated scheduling component with persisted deadline rows. That is an infrastructure decision. The domain Value Object definitions (`deadline: timestamp`) and the invariants (INV-G-06: 5-second window, INV-G-11: 60-second window, INV-G-12/13: forfeit on expiry) require no change because they are expressed at the right level: they state *what* must happen, not *which process* fires the event.

**Why the domain docs stay unchanged:** Adding a service name or a table structure to the `ChallengeWindow` Value Object would couple a domain concept to an infrastructure artifact. The domain correctly delegates "how a deadline fires after a crash" to the architecture.

**Non-negotiable confirmation:**
- **5-second Uno! challenge window (INV-G-06):** Unchanged. `openedAt + 5s` is the server-authoritative deadline. The architecture's firing mechanism must validate against the window's logical state, not override it.
- **60-second reconnection window (INV-G-11, V9):** Unchanged. The 60-second duration is a fixed domain value. `AutoForfeit` is applied only if the window is still logically open at expiry time.
- **Idempotency of expiry side effects:** Required by the domain (at-least-once delivery, A1); the architecture's mechanism must satisfy it. INV-G-11, INV-G-12 are unchanged.

---

## Part II — Tournament Orchestration Context (TO)

### Change 7 — `OpenRegistration` Command and `RegistrationOpened` Event Added

**File:** `spec/domain/docs/02-bounded-contexts-and-context-map.md` — Section 2.1.3 (Events Produced)
**File:** `spec/domain/docs/04-commands-and-domain-events.md` — Section 4.2.3

**Deliverable:** *Deliverable 4 — Commands and domain events catalog* | *Deliverable 2 — Bounded contexts and context map*

**Delta:** The domain design defined tournament states as `RegistrationOpen → InProgress → Completed` (doc 03 §3.2.1) but did not include an explicit `OpenRegistration` command or `RegistrationOpened` event to transition into `RegistrationOpen`. The architecture (`tournament-orchestration.md` §Public Interfaces) adds:

- **Command:** `OpenRegistration` — organizer action that opens player registration for a created tournament. Maps to REST endpoint `POST /api/v1/tournaments/{id}/open-registration`.
- **Event:** `RegistrationOpened` — published to `tournament.lifecycle` topic when registration opens. Consumed by `spectator-projection-service` (lobby/bracket display) and `audit-service`.

**Why:** The architecture requires explicit REST endpoints for every lifecycle transition (§6.1 synchronous interfaces). Without `OpenRegistration`, a tournament moves from `Created` to `RegistrationOpen` implicitly, making the transition invisible to the audit trail and the spectator bracket projection. The `tournament.lifecycle` topic contract (§6.3 integration table) requires all tournament state transitions to be named events.

**Non-negotiable confirmation:** INV-T-10 (registration only accepted when `status == RegistrationOpen`) is unchanged. The new command is the explicit trigger for that state transition — it tightens the model rather than loosening it.

---

### Change 8 — First-Round Surge: Architecture Names the Fan-Out Mechanism (No Domain File Change)

**Domain file changed:** *None.* The domain already covers the relevant saga step correctly.

**Architecture file:** `spec/architecture/03-bounded-contexts/tournament-orchestration.md` — §First-Round Surge Architecture

**Deliverable:** *Deliverable 7 — Consistency and recovery strategy* (for reference only; no edit needed)

**Delta:** The domain design (doc 07 §7.3.1) describes the Tournament Round Progression Saga step "For each room: emit `TournamentRoomAssigned`" — this is a correct domain-level description of what happens. The saga also already documents partial-failure handling (idempotent room creation by `roomId`) and the compensation mechanism for stuck rooms.

What the domain does not and should not specify is *how* 100,000 events get emitted without overwhelming downstream services — that is an infrastructure scaling concern. The architecture introduces a dedicated sharded worker pool with controlled publish rate and adaptive backpressure. These are implementation decisions: service count, shard partitioning strategy, rate limits, Kafka topic names. None of these belong in the domain saga narrative.

**Why the domain doc stays unchanged:** The domain saga correctly states the logical step ("emit `TournamentRoomAssigned` per room") and the idempotency requirement. Naming a specific worker service or a rate limit of "1,000 rooms/sec/shard" would anchor the domain to an infrastructure sizing decision that should be tunable without touching domain artifacts.

**Non-negotiable confirmation:**
- **INV-T-02 (round may not advance until all rooms complete):** Unchanged. The architecture's fan-out mechanism only handles room *creation*; the advancement gate (completion counter) is a domain invariant and unchanged.
- **Idempotent room creation (doc 07 §7.2.2):** Already correctly specified in the domain. The architecture satisfies it.

---

### Change 9 — `TournamentRoomAssigned` Carries Pre-Generated `roomId`

**File:** `spec/domain/docs/04-commands-and-domain-events.md` — Section 4.3 (`TournamentRoomAssigned` payload)
**File:** `spec/domain/docs/03-aggregates-entities-value-objects.md` — Section 3.2.3 (`RoomAssignment` Value Object)

**Deliverable:** *Deliverable 4 — Commands and domain events catalog* | *Deliverable 3 — Aggregates, entities, and value objects*

**Delta:** The domain design (doc 07 §7.2.2) stated that `roomId` in tournament rooms "must be deterministic and assigned by TO at the time the event is emitted (not generated by RG on receipt)." However, the `RoomAssignment` Value Object (doc 03 §3.2.3) and the `TournamentRoomAssigned` event payload did not explicitly include the pre-assigned `roomId`. The architecture formalizes that `tournamentRoomAssigned.roomId` is always a TO-generated UUID, and `room-service` uses this field as its idempotency key for creation.

**Why:** Without an explicit pre-assigned `roomId` in the event payload, RG would generate a new `roomId` on each delivery of the same event, creating duplicate rooms on retry. The architecture's at-least-once delivery guarantee (A1) makes this a practical concern, not just theoretical.

**Non-negotiable confirmation:** INV-R-10 (`tournamentId` and `roundId` must both be present or absent) is unchanged. The new `roomId` field is additive; the existing coupling invariant stands.

---

## Part III — Identity & Session Context (IS)

### Change 10 — Session Invalidation Must Terminate the Live Connection (Domain-Level Clarification Only)

**File:** `spec/domain/docs/02-bounded-contexts-and-context-map.md` — Section 2.3.4 (SessionInvalidated → Forced Disconnect flow)

**Deliverable:** *Deliverable 2 — Bounded contexts and context map*

**Delta:** The domain design (doc 02 §2.3.4 and doc 07 §7.3.4) described the `SessionInvalidated → Forced Disconnect` chain correctly as a domain saga: IS emits `SessionInvalidated` → RG processes it asynchronously → `PlayerDisconnected` starts the 60-second timer. This saga is unchanged.

What was missing at the **domain level** is an explicit statement that `SessionInvalidated` must also reach *the component holding the live player connection* — not only the game state service — to prevent the superseded session from observing further gameplay events. This is a domain-level guarantee (the session is no longer authorized; its connection must be terminated promptly), not only an infrastructure concern.

The domain event flow description in doc 02 §2.3.4 is updated to add a step: **"The `SessionInvalidated` event must also be delivered to the component holding the live connection so that the superseded session's stream is closed before any further game state is observable through it."**

This update expresses the domain requirement. The architecture document (`identity-session.md` §Push-Invalidation) describes the concrete mechanism (which component, which protocol). The domain doc does not name the mechanism.

**Why:** Architecture Checkpoint §6.1 requires this as a non-negotiable: "A design that only invalidates rows in the session store, with no path to the process that owns the open socket, is *incomplete*." The domain gap was the absence of any statement that live-connection termination is a required outcome of `SessionInvalidated`, not only a game-service-level disconnect.

**Non-negotiable confirmation:** INV-S-01 (single active session — atomic CAS at creation) is unchanged. The domain clarification adds a delivery requirement for `SessionInvalidated` without changing the invariant itself or naming any infrastructure component.

---

## Part IV — Ranking & Statistics Context (RK)

### Change 11 — Variable K-Factor Replaces Fixed K=32

**File:** `spec/domain/docs/03-aggregates-entities-value-objects.md` — Section 3.4.1, INV-PR-05
**File:** `spec/domain/docs/08-assumptions-and-open-questions.md` — Assumption A24

**Deliverable:** *Deliverable 3 — Aggregates, entities, and value objects*

**Delta:** INV-PR-05 (doc 03 §3.4.1) and Assumption A24 stated a fixed K-factor of 32 for all players. The architecture (`ranking-statistics.md` §Elo Pipeline, Step 5) replaces this with a variable K-factor based on games rated:

| Games Rated | K-Factor |
|-------------|----------|
| < 30 | 40 (high volatility for new players) |
| 30 – 99 | 20 (stabilizing) |
| ≥ 100 | 10 (established players) |

INV-PR-05 is updated accordingly. A24 is revised to: "K-factor is variable per player based on games rated (see INV-PR-05)."

**Why:** A fixed K=32 applied equally to a player's first game and their 500th game overvalues early results and undervalues late results. Variable K is standard Elo practice (used by FIDE, Chess.com, etc.) and reduces rating noise at scale. With 1,000,000 players, a fixed high K amplifies Elo volatility and makes the leaderboard less stable. The architecture chose variable K to improve rating credibility under the stated scale assumptions.

**Non-negotiable confirmation:** The core Elo non-negotiables are unchanged:
- INV-PR-01 (casual rooms only): unchanged.
- INV-PR-02 (non-abandoned only): unchanged.
- INV-PR-03 (idempotency — no double update): unchanged.
- INV-PR-04 (Elo floor at 100): unchanged.
- INV-PR-08 (TPR isolation from Elo): unchanged.

The multi-player pairwise Elo formula (A25) is unchanged — only the K scalar varies per player.

---

### Change 12 — TPR Computation Formally Wired to `TournamentCompleted` Event

**File:** `spec/domain/docs/02-bounded-contexts-and-context-map.md` — Section 2.1.4 (Events Consumed by RK)
**File:** `spec/domain/docs/04-commands-and-domain-events.md` — Section 4.2.4

**Deliverable:** *Deliverable 4 — Commands and domain events catalog* | *Deliverable 2 — Bounded contexts and context map*

**Delta:** INV-PR-08 (doc 03 §3.4.1) stated that Tournament Placement Rating (TPR) is updated only in response to `TournamentCompleted` events from the TO context. However, this integration was not present in the *Events Consumed* table of Section 2.1.4 (Ranking & Statistics). The architecture formalizes it: `ranking-service` subscribes to the `tournament.lifecycle` Kafka topic and processes `TournamentCompleted` events to compute TPR for all finalists.

The Events Consumed table for RK (doc 02 §2.1.4) is updated:

| Event | Source Context | Reaction |
|-------|---------------|----------|
| `TournamentCompleted` | Tournament Orchestration | Compute Tournament Placement Rating (TPR) for all finalists based on tournament placement order. Update `PlayerRating.tournamentPlacementRating`. |

**Why:** The integration table (§6.3) requires all producer→consumer paths to be explicit. The `tournament.lifecycle → ranking-service` row was absent from the domain's context map. Without it, a reviewer cannot trace how TPR is updated at the architectural level.

**Non-negotiable confirmation:** INV-PR-08 is unchanged and now strengthened: the consuming event (`TournamentCompleted`), the producing topic (`tournament.lifecycle`), and the consuming service (`ranking-service`) are all named explicitly. The isolation of TPR from Elo (separate fields, separate update triggers) is preserved.

---

## Part V — Spectator View Context (SV)

### Change 13 — ACL Transformation Table Extended for New Events

**File:** `spec/domain/docs/02-bounded-contexts-and-context-map.md` — Section 2.1.5.2 (Domain Events Driving Spectator Updates)

**Deliverable:** *Deliverable 2 — Bounded contexts and context map*

**Delta:** The ACL transformation table (doc 02 §2.1.5.2) was designed against the original event catalog. With the addition of WDF events (Change 1) and the new turn-lifecycle events (Change 2), the following rows are added to the spectator transformation table:

| Source Event (RG) | Spectator Event | Transformation |
|-------------------|----------------|----------------|
| `WildDrawFourChallengeWindowOpened` | `SpectatorWDFChallengeWindowOpened` | **Retain:** affected player, challenged player, window duration. **Strip:** `challengedPlayerHandSnapshot`. |
| `WildDrawFourChallengeMade` | `SpectatorWDFChallengeMade` | Retain as-is (challenger/challenged IDs). |
| `WildDrawFourChallengeResolved` | `SpectatorWDFChallengeResolved` | **Retain:** outcome (`bluff_confirmed` / `legitimate_play`), who penalized. **Strip:** hand contents used in adjudication, penalty card identities. |
| `WildDrawFourChallengeWindowClosed` | `SpectatorWDFChallengeWindowClosed` | Retain as-is. |
| `TurnPassed` | `SpectatorTurnPassed` | Retain as-is. |
| `PenaltyCardsDrawn` | `SpectatorPenaltyCardsDrawn` | **Strip:** card identities. Retain: `playerId`, `count`, `reason`. |
| `TurnSkippedDueToDisconnection` | `SpectatorTurnSkippedDueToDisconnection` | Retain as-is. |
| `TurnTimedOut` | `SpectatorTurnTimedOut` | Retain as-is (player ID, action taken: auto-draw / auto-pass). |
| `DrawPileReshuffled` | `SpectatorDrawPileReshuffled` | **Strip:** `reshuffleSeed`. **Retain:** `gameId`, event occurrence (spectators may observe the reshuffle but not the seed). |
| `DeckInitialized`, `InitialHandsDealt`, `FirstCardFlipped` | — | **Not projected.** These occur during `Initializing` phase before spectators can join. Spectator projection begins at `GameStarted`. |

**Why:** The spectator ACL must enumerate every event type on `gameplay.events` to avoid defaulting to pass-through behavior for unrecognized event types. The architecture (`spectator-view.md` §ACL Transformation Table) requires an explicit disposition for each event. A silent pass-through for an unrecognized WDF event could inadvertently expose hand data (`challengedPlayerHandSnapshot`).

**Non-negotiable confirmation:** INV-SGP-01 (no player hand contents in spectator projection) is preserved and strengthened: `challengedPlayerHandSnapshot` is now explicitly listed as stripped from `WildDrawFourChallengeResolved`. The privacy boundary is complete for all enumerated event types.

---

### Change 14 — Spectator Transport and Channel Separation: Architecture Decision (No Domain File Change)

**Domain file changed:** *None.* The domain already covers this correctly.

**Architecture file:** `spec/architecture/03-bounded-contexts/spectator-view.md` — §Public Interfaces, §Where Privacy Is Enforced

**Deliverable:** *Deliverable 2 — Bounded contexts and context map* (for reference only; no edit needed)

**Delta:** The domain design correctly expresses the spectator channel invariants: spectators receive only read-optimized projections through an Anti-Corruption Layer (INV-SGP-01, INV-SGP-03); the spectator projection is separate from the authoritative game state; no write-back is possible. These are domain invariants — they say *what* must hold.

The choice of transport protocol (SSE vs WebSocket), the specific route paths (`/spectator/games/{gameId}/stream`), the JWT requirements per route, and the gateway routing rules are all architecture decisions. Adding SSE or WebSocket as domain vocabulary would introduce implementation coupling into the ubiquitous language.

**Why the domain doc stays unchanged:** INV-SGP-03 ("this aggregate accepts no commands from clients… no write-back to any other context occurs") already fully captures the read-only guarantee at the domain level. The transport mechanism that enforces this in production is the architecture's responsibility.

**Non-negotiable confirmation:** INV-SGP-01 (privacy hard boundary) and INV-SGP-03 (read-only, no write-back) are unchanged. The architecture demonstrates that the domain invariants are implementable with defense-in-depth: ACL transformation, separate storage, and separate streaming channel all reinforce INV-SGP-01 without the domain needing to name any of them.

---

## Part VI — Audit & Game Log Context (AL)

### Change 15 — Role-Based Access Tiers Formalized (Operator / Admin / Compliance)

**File:** `spec/domain/docs/03-aggregates-entities-value-objects.md` — Section 3.6.1 (Game Log Entity), INV-GL-05
**File:** `spec/domain/docs/02-bounded-contexts-and-context-map.md` — Section 2.1.6

**Deliverable:** *Deliverable 3 — Aggregates, entities, and value objects* | *Deliverable 2 — Bounded contexts and context map*

**Delta:** The domain design (doc 03 §3.6.1, INV-GL-05) stated that "access to the Game Log is restricted to authorized audit and dispute resolution processes" but did not define the access roles. The architecture (`audit-game-log.md` §Read Path for Dispute Resolution) formalizes three roles:

| Role | Access |
|------|--------|
| `operator` | Read individual game logs (per-game, for support tickets). Cannot search cross-game or export. |
| `admin` | Full audit trail search. Can verify HMAC integrity. Cannot export (compliance role required). |
| `compliance` | Export signed bundles for legal/regulatory requests. All compliance queries are meta-audited (logged separately with the officer's identity and query parameters). |
| Internal mTLS | Anti-cheat / automated replay jobs, scoped to specific game IDs. |

INV-GL-05 is updated to reference these role tiers explicitly.

**Why:** Architecture Checkpoint §6.4 requires documenting "who may query or export [the immutable game log], for what purpose, and how access is authorized (roles, mTLS, break-glass, scoped APIs)." The domain correctly recognized access restriction but did not define the authorization model needed for operational deployment.

**Non-negotiable confirmation:** INV-GL-01 (append-only) and INV-GL-05 (restricted access, full private data) are unchanged. The new role definitions are constraints on *read access*, not on write behavior. The append-only property is enforced at the database layer (no UPDATE/DELETE permissions for the application role), not by the access-control layer.

---

### Change 16 — Audit Trail Retention Period Updated (2 Years Minimum)

**File:** `spec/domain/docs/03-aggregates-entities-value-objects.md` — Section 3.6.2, INV-AT-05

**Deliverable:** *Deliverable 3 — Aggregates, entities, and value objects*

**Delta:** INV-AT-05 (doc 03 §3.6.2) stated: "Tournament game logs and audit entries are retained indefinitely (A32). Casual game logs are retained for a minimum of 90 days (A32)." The architecture (`audit-game-log.md` §Retention) sets:

- **Audit trail:** 2-year minimum retention (compliance requirement), then archived to cold storage.
- **Game logs:** 1-year active retention, then archived to cold storage (S3/GCS with Parquet format). Archived logs remain queryable via the compliance export API.
- **Processed events (deduplication index):** Pruned after 7 days.

INV-AT-05 is updated: "The audit trail is retained for a minimum of 2 years. Game logs are retained for a minimum of 1 year in active storage, then archived to cold storage. Within their retention period, all entries are immutable."

**Why:** "Indefinitely" is not an operational policy — it provides no guidance for storage capacity planning or compliance commitments. The architecture must name concrete retention windows to satisfy §6.4 (Retention and audit) and to ground the capacity sketch (§6.5). 

**Non-negotiable confirmation:** INV-AT-01 (append-only, immutable) is unchanged and applies to the full retention period plus archival. The 7-day pruning applies only to the `processed_events` deduplication index (a performance artifact), not to the audit trail or game log — which remain immutable for their full retention period.

---

## Summary Table of All Changes

| # | Domain File(s) Changed | Design Checkpoint Deliverable | Delta Type | Invariants Affected |
|---|----------------------|------------------------------|------------|---------------------|
| 1 | `02-bounded-contexts.md` §2.1.2, `04-commands.md` §4.3 | Deliverable 4, Deliverable 2 | **Added events** — WDF challenge events to outbound catalog | None weakened; new events are publication form of existing domain logic |
| 2 | `02-bounded-contexts.md` §2.1.2, `04-commands.md` §4.3 | Deliverable 4 | **Added events** — initialization + turn lifecycle events | INV-G-16, INV-G-17 unchanged; events are named forms of existing behaviors |
| 3 | `02-bounded-contexts.md` §2.1.2, §2.3.1; `04-commands.md` §4.3 | Deliverable 4, Deliverable 2 | **Enriched payload** — `wasAbandoned`, `roomType` on `GameCompleted` | Elo scope non-negotiable strengthened (explicit boolean) |
| 4 | `04-commands.md` §4.1.3; `03-aggregates.md` INV-G-05 | Deliverable 4, Deliverable 3 | **Invariant updated** — sequence number exception for `CallUno`, `ChallengeUnoCall`, `ChallengeWildDrawFour` | Challenge window invariants unchanged; turn-based invariants unchanged |
| 5 | *(no domain file changed)* | Deliverable 7 | **Architecture-only** — transactional outbox implements Option B; domain invariant (V12, INV-GL-03) already correct | INV-GL-03 satisfied by architecture |
| 6 | *(no domain file changed)* | Deliverable 3, Deliverable 7 | **Architecture-only** — dedicated scheduling component implements timer durability; domain Value Objects (`deadline` field) and invariants (INV-G-06, INV-G-11) already correct | 5s/60s timer invariants satisfied by architecture |
| 7 | `02-bounded-contexts.md` §2.1.3; `04-commands.md` §4.2.3 | Deliverable 4, Deliverable 2 | **Added command + event** — `OpenRegistration` / `RegistrationOpened` | INV-T-10 tightened |
| 8 | *(no domain file changed)* | Deliverable 7 | **Architecture-only** — sharded fan-out implements "emit TournamentRoomAssigned per room"; domain saga step already correct | INV-T-02 unchanged; idempotent room creation already specified in domain |
| 9 | `04-commands.md` §4.3; `03-aggregates.md` §3.2.3 | Deliverable 4, Deliverable 3 | **Payload addition** — pre-generated `roomId` in `TournamentRoomAssigned` | At-least-once idempotency (doc 07 §7.2.2) strengthened |
| 10 | `02-bounded-contexts.md` §2.3.4 | Deliverable 2 | **Domain clarification** — `SessionInvalidated` must reach the live-connection holder, not only the game service | INV-S-01 unchanged; domain delivery requirement made explicit |
| 11 | `03-aggregates.md` INV-PR-05; `08-assumptions.md` A24 | Deliverable 3 | **Invariant modified** — variable K-factor (40/20/10) replaces fixed K=32 | Elo scope invariants (PR-01 through PR-08) otherwise unchanged |
| 12 | `02-bounded-contexts.md` §2.1.4; `04-commands.md` §4.2.4 | Deliverable 4, Deliverable 2 | **Added consumed event** — `TournamentCompleted` → TPR update in RK context map | INV-PR-08 unchanged, integration now explicit |
| 13 | `02-bounded-contexts.md` §2.1.5.2 | Deliverable 2 | **Extended ACL table** — rows for WDF events + turn lifecycle events | INV-SGP-01 strengthened: complete enumeration prevents silent pass-through |
| 14 | *(no domain file changed)* | Deliverable 2 | **Architecture-only** — SSE/WebSocket transport implements spectator privacy separation; INV-SGP-01, INV-SGP-03 already correct at domain level | INV-SGP-01, INV-SGP-03 satisfied by architecture |
| 15 | `03-aggregates.md` §3.6.1 INV-GL-05; `02-bounded-contexts.md` §2.1.6 | Deliverable 3, Deliverable 2 | **Access roles named** — operator / admin / compliance / internal automated | INV-GL-05 unchanged, authorized-actor model made explicit |
| 16 | `03-aggregates.md` §3.6.2 INV-AT-05; `08-assumptions.md` A32 | Deliverable 3 | **Retention policy updated** — 2-year audit trail, 1-year active game log replaces "indefinitely" | INV-AT-01 unchanged; retention period operationalized |

---

## Global Affirmation of Non-Negotiable Domain Guarantees

The following Design Checkpoint non-negotiable guarantees are affirmed as intact after all sixteen changes:

| Guarantee | Status | Architectural Owner |
|-----------|--------|---------------------|
| **Elo scope:** no tournament or abandoned casual games affect Elo | ✅ Preserved and strengthened | `ranking-service`: filters on `wasAbandoned` (Change 3) and `roomType` fields in `GameCompleted` |
| **Tournament advancement:** top 3 by match wins → card points → completion time | ✅ Preserved | `tournament-service`: advancement logic unchanged; `MatchCompleted` payload carries full ranking data |
| **Match vs. game terminology:** consistent in interfaces and events | ✅ Preserved | Kafka topics: `gameplay.games` (per-game events) vs. `gameplay.rooms` (match/room events). `MatchCompleted` and `GameCompleted` remain distinct. |
| **Sequence-number enforcement:** stale/replayed commands rejected | ✅ Preserved | `game-engine`: command handler. Exception added only for time-window commands guarded by window state (Change 4). |
| **Log-before-broadcast atomicity** | ✅ Preserved | Domain: V12, INV-GL-03 (Option B, unchanged). Architecture: names the concrete mechanism (`room-gameplay.md` §Log-Before-Broadcast). |
| **5-second Uno! challenge window durability** | ✅ Preserved | Domain: INV-G-06, `ChallengeWindow` Value Object (unchanged). Architecture: names the scheduling component (`room-gameplay.md` §5-Second Uno! Challenge Window). |
| **60-second reconnection window durability** | ✅ Preserved | Domain: INV-G-11, V9, `ReconnectionWindow` Value Object (unchanged). Architecture: names the scheduling component. |
| **Single-active-session enforcement** | ✅ Preserved and clarified | Domain: INV-S-01 (atomic CAS, unchanged) + doc 02 §2.3.4 clarified that `SessionInvalidated` must reach the live-connection holder (Change 10). Architecture: names the mechanism (`identity-session.md` §Push-Invalidation). |
| **Spectator projection privacy** | ✅ Preserved and strengthened | Domain: INV-SGP-01 (unchanged) + ACL table extended for new events (Change 13). Architecture: names transport and route separation (`spectator-view.md` §Where Privacy Is Enforced). |
| **Match series coordination** | ✅ Preserved | `room-service`: Match entity persists game results, evaluates scoreline, starts next game or emits `MatchCompleted`. State survives restarts (PostgreSQL). |
| **Abandoned-game vs. completed-game distinction for Elo** | ✅ Preserved and strengthened | `wasAbandoned` field in `GameCompleted` payload (Change 3). |

---

## Part VII — Post-Review Corrections (Design Checkpoint Evaluator Feedback)

The following changes address deductions reported by the course evaluator on the Design Checkpoint submission. They are separate from the Architecture Checkpoint deltas above and are catalogued here for traceability.

---

### Correction A — Match Definition: Multi-Player Semantics Added to INV-R-08

**File:** `spec/domain/docs/03-aggregates-entities-value-objects.md` — INV-R-08

**Deliverable:** *Deliverable 3 — Aggregates, entities, and value objects*

**Issue:** INV-R-08 read "a player wins 2 out of 3 Games," which is unambiguous only for 2-player rooms. For multi-player rooms (3–10 players), only one player wins each game so the "2 out of 3" framing does not apply — all 3 games are played and a ranking is established.

**Delta:** INV-R-08 updated to explicitly distinguish the two cases: 2-player rooms end when one player reaches 2 wins; multi-player rooms conclude after all 3 Games (or last-player-standing) per INV-M-04 and A15/A16.

**Note:** The glossary entry for Match (doc 01, term 2) already contained the correct multi-player semantics. This correction aligns INV-R-08 with it.

**Non-negotiable confirmation:** INV-M-04 (the authoritative multi-player match rule) is unchanged. INV-R-08 now references it explicitly.

---

### Correction B — Sequence Number Scope Clarified: INV-R-12 vs INV-G-05

**File:** `spec/domain/docs/03-aggregates-entities-value-objects.md` — INV-R-12

**Deliverable:** *Deliverable 3 — Aggregates, entities, and value objects*

**Issue:** INV-R-12 ("All commands issued to this aggregate must carry a valid SequenceNumber") was read as potentially conflicting with INV-G-05 (the Game aggregate's independent sequence counter). The overlap was ambiguous about which commands are governed by which sequence.

**Delta:** INV-R-12 updated to explicitly state that it applies only to **Room-level commands** (`JoinRoom`, `LeaveRoom`, `StartMatch`, `ForfeitGame`) and that gameplay commands (`PlayCard`, `DrawCard`, `PassTurn`, etc.) are governed exclusively by INV-G-05. The two sequences are independent.

**Non-negotiable confirmation:** Both INV-R-12 (Room sequence) and INV-G-05 (Game sequence, including the challenge-window exception from Change 4) are maintained without weakening either.

---

### Correction C — Spectator Privacy: Active Player as Spectator Scenario Added

**File:** `spec/domain/docs/06-edge-cases-and-failure-paths.md` — Section 6.6 (new subsection 6.6.4; prior 6.6.4 renumbered to 6.6.5)

**Deliverable:** *Deliverable 6 — Edge cases and failure paths*

**Issue:** The required scenario — "an active player opens a second connection as an anonymous spectator in their own room" — was not explicitly documented as a named edge case, even though the underlying protection mechanisms (ACL, INV-SGP-01, INV-SGP-03) were already in place.

**Delta:** New subsection 6.6.4 added, titled "Active Player Opens a Second Connection as Anonymous Spectator in Their Own Room." It documents: (a) the scenario, (b) why the ACL is unconditional regardless of subscriber identity, (c) that the spectator projection is a strict subset of what a participant already observes, and (d) that there is no information gain. References INV-SGP-01, INV-SGP-03, INV-SGP-05. Summary table row added.

**Non-negotiable confirmation:** INV-SGP-01 (privacy hard boundary) is unchanged and confirmed to hold unconditionally — identity of the subscriber is irrelevant to ACL enforcement.

---

### Correction D — K-Factor Model: Internal Consistency Restored

**File:** `spec/domain/docs/07-consistency-and-recovery-strategy.md` — Section 7.3.5 (Elo Update Pipeline)
**File:** `spec/domain/docs/03-aggregates-entities-value-objects.md` — INV-PR-05 *(updated in Change 11 above)*
**File:** `spec/domain/docs/08-assumptions-and-open-questions.md` — A24 *(already consistent)*

**Deliverable:** *Deliverable 7 — Consistency and recovery strategy* | *Deliverable 3 — Aggregates, entities, and value objects* | *Deliverable 8 — Open questions and assumptions*

**Issue:** A contradiction existed between: (a) A24 and INV-PR-05, which in their original form stated K=32 for all players, and (b) the Elo pipeline description in doc 07 and the architecture document, which used a variable K-factor schedule (K=40/20/10 by games rated). The evaluator deducted 1.0 pt for this internal inconsistency.

**Resolution chosen:** Variable K-factor (Option A from evaluator's recommendation). This is the approach already implemented in the architecture document and provides better rating stability at scale.

**Delta:**
- Doc 07 §7.3.5 pipeline diagram: `K-factor = 32 (A24)` → `K-factor per player (A24): K=40 if gamesRated < 30; K=20 if 30 ≤ gamesRated < 100; K=10 if gamesRated ≥ 100`.
- Doc 07 §7.3.5 bounded-drift note: "bounded by the K-factor of 32" → "bounded by the applicable K-factor (at most 40 for a new player)."
- INV-PR-05 (doc 03): Updated in Change 11 above.
- A24 (doc 08): Already reflects variable K — no further change needed.

**Non-negotiable confirmation:** All Elo-scope invariants (INV-PR-01 through INV-PR-04, INV-PR-06 through INV-PR-08) are unchanged. Only the K scalar varies; the formula, floor, idempotency guarantee, and casual-only filter are unaffected.

---

### Post-Review Summary Table

| # | File Changed | Design Checkpoint Deliverable | Evaluator Deduction | Delta |
|---|-------------|------------------------------|---------------------|-------|
| A | `03-aggregates.md` INV-R-08 | Deliverable 3 | -0.5 pts (Match definition ambiguity) | INV-R-08 now distinguishes 2-player vs multi-player match completion |
| B | `03-aggregates.md` INV-R-12 | Deliverable 3 | -0.5 pts (Sequence number scope ambiguity) | INV-R-12 scoped to Room-level commands; INV-G-05 scoped to gameplay commands |
| C | `06-edge-cases.md` §6.6.4 (new) | Deliverable 6 | -0.5 pts (Missing spectator scenario) | Added active-player-as-spectator edge case with invariant analysis |
| D | `07-consistency.md` §7.3.5; `03-aggregates.md` INV-PR-05; `08-assumptions.md` A24 | Deliverable 7, 3, 8 | -1.0 pts (K-factor contradiction) | Variable K-factor (40/20/10) applied consistently across all three documents |

---

*Generated for Architecture Checkpoint — 2026-05-17, updated with post-review corrections 2026-05-17. All cross-references use document section numbers from the design package in `spec/domain/docs/`.*
