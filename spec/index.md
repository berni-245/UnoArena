# UnoArena — Domain Model Design

> A rigorous, behavior-complete domain model for a global real-time Uno platform supporting up to 1,000,000 concurrent players, produced using Domain-Driven Design and EventStorming methodology.

---

## Deliverables

| # | Deliverable | Document |
|---|-------------|----------|
| 1 | Domain Glossary | [01-domain-glossary.md](docs/01-domain-glossary.md) |
| 2 | Bounded Contexts & Context Map | [02-bounded-contexts-and-context-map.md](docs/02-bounded-contexts-and-context-map.md) |
| 3 | Aggregates, Entities & Value Objects | [03-aggregates-entities-value-objects.md](docs/03-aggregates-entities-value-objects.md) |
| 4 | Commands & Domain Events Catalog | [04-commands-and-domain-events.md](docs/04-commands-and-domain-events.md) |
| 5 | Domain Event Flow Narratives | [05-domain-event-flows.md](docs/05-domain-event-flows.md) |
| 6 | Edge Cases & Failure Paths | [06-edge-cases-and-failure-paths.md](docs/06-edge-cases-and-failure-paths.md) |
| 7 | Consistency & Recovery Strategy | [07-consistency-and-recovery-strategy.md](docs/07-consistency-and-recovery-strategy.md) |
| 8 | Assumptions & Open Questions | [08-assumptions-and-open-questions.md](docs/08-assumptions-and-open-questions.md) |

## Diagrams

All diagrams use Mermaid syntax and are stored under [`/diagrams`](diagrams/).

| Diagram | File |
|---------|------|
| Context Map | [context-map.md](diagrams/context-map.md) |
| Room State Machine | [room-lifecycle.md](diagrams/room-lifecycle.md) |
| Game Turn Sequence | [game-turn-sequence.md](diagrams/game-turn-sequence.md) |
| Uno Call Timing | [uno-call-timing.md](diagrams/uno-call-timing.md) |
| Match Lifecycle | [match-lifecycle.md](diagrams/match-lifecycle.md) |
| Tournament Progression | [tournament-progression.md](diagrams/tournament-progression.md) |
| Tournament Round Flow | [tournament-round-flow.md](diagrams/tournament-round-flow.md) |
| Disconnection Handling | [disconnection-flow.md](diagrams/disconnection-flow.md) |
| Elo Update Pipeline | [elo-update-flow.md](diagrams/elo-update-flow.md) |
| Spectator Projection | [spectator-projection.md](diagrams/spectator-projection.md) |
| Room Creation to Completion | [room-creation-to-completion.md](diagrams/room-creation-to-completion.md) |
| Concurrent Action Resolution | [concurrent-action-resolution.md](diagrams/concurrent-action-resolution.md) |

## Methodology

This design was produced using **EventStorming** as the primary discovery technique. Each deliverable reflects the outcomes of EventStorming sessions, with:

- **Orange stickies** → Domain Events (documented in deliverable 4)
- **Blue stickies** → Commands (documented in deliverable 4)
- **Yellow stickies** → Aggregates (documented in deliverable 3)
- **Purple stickies** → Policies (documented in deliverables 5 and 7)
- **Red stickies** → Hotspots / Edge Cases (documented in deliverable 6)
- **Pink stickies** → External Systems / Integration Points (documented in deliverable 2)

The flow narratives in deliverable 5 represent the timeline-based output of the EventStorming process, showing command → event → policy → command chains.
