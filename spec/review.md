# Spec Review — Findings and Suggestions

> Cross-document review of `index.md`, `docs/*.md`, and `diagrams/*.md` against `requirements.md`. Findings are organized from most to least critical.

---

## Critical Inconsistencies

### 1. `AN` vs `AL` — Bounded Context Abbreviation Clash

**The most pervasive inconsistency in the entire spec.**

- `docs/01-domain-glossary.md` abbreviation table lists `AN = Analytics`
- `docs/02-bounded-contexts-and-context-map.md` defines a sixth context called **Audit & Game Log**, but never assigns it an abbreviation in that document
- `docs/04-commands-and-domain-events.md` event envelope uses `sourceContext: IS | RG | TO | RK | SV | **AL**`
- All diagrams use `AL` for Audit & Game Log
- The glossary uses `AN` for terms like "Audit Log" (term 84: `IS, AN`), "Bracket" (`TO, AN`), "Projection / Read Model" (`AN, SV`)

**Fix:** Update the glossary abbreviation table to replace `AN → Analytics` with `AL → Audit & Game Log`. If there is an Analytics sub-concern, it belongs inside AL or RK, not as a separate context. Audit-related glossary terms (84, 95, 96) should reference `AL`, not `AN`.

---

### 2. `StartMatch` vs `StartGame` — Command Name Mismatch

Two different names for the same command appear in different documents:

| Document | Command Used |
|----------|-------------|
| `docs/03` INV-R-03 | `StartMatch` |
| `docs/05` Flow 1, step 8 | `StartGame` |
| `diagrams/room-creation-to-completion.md` line 23 | `StartGame` |

**Fix:** Standardize to one name. `StartMatch` is more semantically precise (it starts a Match that contains Games), so prefer that. Update all references in doc 05 and the diagram.

---

### 3. Uno Challenge Race Condition: Penalty Contradiction

`docs/06` section 6.1.3 and `diagrams/concurrent-action-resolution.md` Scenario 2 directly contradict each other:

- **Doc 06** (text): "No penalty is assessed to either player beyond the rejection of the challenge command. The challenger issued the command in good faith within what their client believed was the window."
- **Diagram** Scenario 2: Shows `GA->>GA: Apply: PC draws 2 penalty cards` after the challenger's command is rejected

This is a domain rule that must be decided and stated once. The doc 06 reasoning (no penalty for a good-faith race-condition challenge) is more principled and consistent with the design intent. The diagram should be corrected to show no penalty applied when `CallUno` wins the race.

---

### 4. Room State Inventory — Three Different Versions

| Document | States Listed |
|----------|--------------|
| `docs/01` glossary terms 35–37 | `Waiting`, `In-Progress`, `Completed` (3 states) |
| `docs/02` Room aggregate description | "`waiting` → `in_progress` → `completed`" (3 states) |
| `docs/03` Room state machine | `Waiting`, `Ready`, `InProgress`, `Completed`, `Abandoned` (5 states) |
| `diagrams/room-lifecycle.md` | Adds `Created` as initial pseudo-state + `Ready` + `Abandoned` + `ForceCompleted` transitions |

`Ready` and `Abandoned` exist in doc 03 and the diagram but are not mentioned anywhere in doc 01 or doc 02. **Fix:**
- Add `Ready` and `Abandoned` to the glossary (terms 35–37 section).
- Update doc 02 Room aggregate description to reflect all 5 states.
- Remove `Created` from the lifecycle diagram (it's not a true state; it's the precondition for `Waiting`).

---

### 5. `GameStartRequested` — Phantom Event

`docs/05` step 9 emits `GameStartRequested` as an event from the Room aggregate when `StartGame` is accepted. This event does **not appear** in:
- Doc 02 events produced table for Room Gameplay
- Doc 04 events catalog

Either add it formally as an event (it serves as the internal trigger for deck initialization), or remove it from the flow narrative and replace it with `GameStarted` being emitted after the internal initialization completes.

---

## Significant Inconsistencies

### 6. `UnoCallMade` vs `UnoCalledSuccessfully` — Event Name Mismatch

| Document | Event Name |
|----------|-----------|
| `docs/02` Room Gameplay events | `UnoCallMade` |
| `docs/06` section 6.1.3 | `UnoCalledSuccessfully` |
| `diagrams/concurrent-action-resolution.md` Scenario 2 | `UnoCalledSuccessfully` |
| `diagrams/room-creation-to-completion.md` | `UnoCalledSuccessfully` |

**Fix:** `UnoCallMade` is the canonical name per doc 02 (and doc 04 implicitly). Update all references in doc 06 and the diagrams.

---

### 7. `ChallengeResolved` vs `UnoChallengeAccepted`/`UnoChallengeRejected`

| Document | Event Name |
|----------|-----------|
| `docs/02` | `ChallengeResolved` (single event with outcome field) |
| `docs/06` 6.1.3 | `UnoChallengeAccepted`, `UnoChallengeRejected` (two separate events) |
| `diagrams/concurrent-action-resolution.md` | `UnoChallengeRejected` |

Decide whether challenge outcomes are one event with an `outcome` field or two distinct events. The single-event approach (`ChallengeResolved`) is more consistent with DDD event naming and what the spectator projection uses (`SpectatorChallengeResolved`). Update doc 06 and the diagrams accordingly.

---

### 8. Context Map Diagram Missing Relationships

`diagrams/context-map.md` is missing edges that are explicitly defined in `docs/02`:

| Relationship | Defined in doc 02 | In diagram |
|---|---|---|
| Identity & Session → Ranking & Statistics | Section 2.2.2 | Missing |
| Spectator View → Audit & Game Log | Section 2.1.5 events consumed | Missing |
| Identity & Session → Audit & Game Log | Section 2.2.2 | Missing |
| `PlayerForfeited` RG → TO flow | Section 2.3.3 | Missing |

The diagram only shows 7 edges; the full context map has 10+ relationships.

---

### 9. `game-turn-sequence.md` Uses `RoomAggregate` for Gameplay Commands

The diagram participant is named `RoomAggregate` and it handles `PlayCard`, `DrawCard`, appends to `GameLog`. But per doc 03, `Game` is a **separate Aggregate Root** from `Room`, explicitly justified by high write frequency. The diagram should use `GameAggregate` (or just `Game`) as the participant for all turn-level operations.

---

### 10. Sequence Number Scope: "per-Room" vs "per-Game"

| Document | Stated Scope |
|----------|-------------|
| `docs/01` term 66 | "Sequence Numbers are **per-Room, per-Player**" |
| `docs/04` section 4.1.3 | "Per-**Game** Sequence Number" and "Per-Player Sequence Number" |
| `docs/07` section 7.1.1.1 | Commands rejected if `commandSequenceNumber != currentSequenceNumber + 1` (per-game implied) |

The authoritative scope is per-Game (doc 04 is more detailed and correct). Update doc 01 term 66 from "per-Room, per-Player" to "per-Game (for gameplay commands)".

---

## Minor Issues & Suggestions

### 11. Spectator Projection Diagram Missing Events

`diagrams/spectator-projection.md` shows 9 event transformations, but `docs/02` section 2.1.5.2 lists more events crossing the ACL, including:
- `DirectionReversed` → `SpectatorDirectionReversed`
- `PlayerSkipped` → `SpectatorPlayerSkipped`
- `ColorChosen` → `SpectatorColorChosen`
- `PlayerReconnected` → `SpectatorPlayerReconnected`
- `MatchCompleted` (for match score updates)

The diagram should be expanded to include these, or a note added that it's illustrative, not exhaustive.

---

### 12. `uno-call-timing.md` Scenario 1: Timer Sequence is Misleading

In Scenario 1, after the challenge is resolved (`ChallengeResolved` outcome), the diagram still shows `Timer-->>Server: ChallengeWindowClosed`. Per domain rules, a successfully processed challenge closes the window immediately — the timer should be cancelled, not naturally expire. The note could read `Server->>Timer: Cancel timer (challenge resolved)` followed by `ChallengeWindowClosed`.

---

### 13. `tournament-progression.md` State Loop Label

The diagram shows `EvaluatingAdvancement --> Round1 : playersRemaining > 10 (next round)`. The label points back to `Round1`, implying it's always round 1. Should be labeled something like "Create Next Round (RoundN+1)" to avoid confusion.

---

### 14. Match Lifecycle Diagram: "forfeiter scored as last"

The `diagrams/match-lifecycle.md` note says "Player forfeits (game continues, forfeiter scored as last)". Doc 03 INV-G-20 is more precise: remaining players are ranked by ascending card point totals, with ties broken by turn proximity. If multiple players forfeit, calling them all "last" is incorrect — they have a defined relative ranking. Suggest changing the note to "game continues, forfeiter gets lowest available placement by card points".

---

### 15. `MatchGameCompleted` in `room-creation-to-completion.md` — Unlisted Event

Lines 59 and 68 of that diagram emit `MatchGameCompleted` — this event does not appear anywhere in doc 02 or doc 04's event catalog. Either add it to the events catalog (it's a useful inter-game coordination signal) or replace it with what actually fires (the Room internally records the game result and decides whether to start the next game; the external event is just another `GameCompleted`).

---

### 16. `PenaltyCardsDrawn` Not in Event Catalog

Doc 05, doc 06, and several diagrams emit `PenaltyCardsDrawn`. This event is **not listed** in doc 02's Room Gameplay events produced table. It appears as an observable fact (2 penalty cards drawn by a player), which is domain-meaningful for spectators and audit. It should be added to the doc 02 event table.

---

### 17. Glossary: `Abandoned Game` (term 40) vs Room `Abandoned` State

Term 40 defines "Abandoned Game" (all players forfeit). But doc 03 defines an `Abandoned` **Room** state. The glossary doesn't define `Abandoned Room` (or `Abandoned Match`). Add a term for it or clarify that the `Abandoned` state in the Room maps to the "all players forfeited" condition, which is distinct from an Abandoned Game within a still-active Room.

---

### 18. Wild First-Card Handling Missing from INV-G-19

Doc 05 step 13 says if the first card is a Wild (non-WDF4), the host declares a color (`DeclareColor` command, `ColorDeclared` event). Doc 03 INV-G-19 only explicitly handles the Wild Draw Four case during initialization. The handling of a starting Wild card should be stated as an invariant in INV-G-19, and `ColorChosen` should be noted as possible during the `Initializing` phase.

---

## Summary Priority Table

| Priority | Issue | Files Affected |
|---|---|---|
| Critical | AN vs AL abbreviation | doc 01, doc 04, all diagrams |
| Critical | StartMatch vs StartGame | doc 03, doc 05, room-creation-to-completion |
| Critical | Challenge race condition penalty contradiction | doc 06, concurrent-action-resolution |
| Critical | Room state count mismatch (3 vs 5 states) | doc 01, doc 02, doc 03, room-lifecycle |
| Critical | `GameStartRequested` phantom event | doc 02, doc 05 |
| High | `UnoCallMade` vs `UnoCalledSuccessfully` | doc 02, doc 06, diagrams |
| High | `ChallengeResolved` vs split events | doc 02, doc 06, diagrams |
| High | Context map missing 3+ relationships | context-map.md |
| High | `RoomAggregate` vs `GameAggregate` in turn diagram | game-turn-sequence.md |
| High | Sequence number scope (per-Room vs per-Game) | doc 01, doc 04 |
| Medium | Spectator projection diagram incomplete | spectator-projection.md |
| Medium | `MatchGameCompleted` unlisted event | doc 02, room-creation-to-completion |
| Medium | `PenaltyCardsDrawn` missing from event catalog | doc 02 |
| Low | Timer cancel vs expire in uno-call-timing | uno-call-timing.md |
| Low | Tournament state loop label | tournament-progression.md |
| Low | "forfeiter scored as last" imprecise | match-lifecycle.md |
| Low | Wild first-card not in INV-G-19 | doc 03 |
