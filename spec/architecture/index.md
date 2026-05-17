# UnoArena — Architecture Specification

Solution architecture for a global real-time Uno platform supporting ad-hoc rooms (2–10 players) and massive elimination tournaments (up to 1,000,000 players). Translates the [Domain Model Design](../domain/index.md) into deployable microservices, integration contracts, and persistence decisions.

---

## Documents

### Views

| # | Document | Description |
|---|----------|-------------|
| 1 | [Context View](01-context-view.md) | Bounded contexts → deployable services mapping |
| 2 | [Container View](02-container-view.md) | C4 container diagram — all runnable components, brokers, stores, trust boundaries |

### Bounded Context Architecture (§6.1)

| Context | Document |
|---------|----------|
| Identity & Session | [03-bounded-contexts/identity-session.md](03-bounded-contexts/identity-session.md) |
| Room Gameplay | [03-bounded-contexts/room-gameplay.md](03-bounded-contexts/room-gameplay.md) |
| Tournament Orchestration | [03-bounded-contexts/tournament-orchestration.md](03-bounded-contexts/tournament-orchestration.md) |
| Ranking & Statistics | [03-bounded-contexts/ranking-statistics.md](03-bounded-contexts/ranking-statistics.md) |
| Spectator View | [03-bounded-contexts/spectator-view.md](03-bounded-contexts/spectator-view.md) |
| Audit & Game Log | [03-bounded-contexts/audit-game-log.md](03-bounded-contexts/audit-game-log.md) |

### Integration & Communication (§6.3)

| # | Document | Description |
|---|----------|-------------|
| 4 | [Integration Table](04-integration-table.md) | From → To, pattern, rationale, failure semantics |
| 5 | [Client Connection Model](05-client-connection-model.md) | WebSocket/SSE, gateway, session invalidation, spectator channels |
| 6 | [Rate Limiting](06-rate-limiting.md) | Multi-layer limits mapped to deployables |

### Data & Scale (§6.4–§6.5)

| # | Document | Description |
|---|----------|-------------|
| 7 | [Persistence Layer](07-persistence.md) | Store per context, consistency model, read models, retention |
| 8 | [Capacity Sketch](08-capacity-sketch.md) | Order-of-magnitude reasoning for 1M-player tournaments |

### Strongly Recommended (§7)

| # | Document | Description |
|---|----------|-------------|
| 9 | [ADRs](09-adr/index.md) | Top architectural decisions — event sourcing, Kafka, timer service, client protocol, database-per-context, kickoff worker, spectator projection, API gateway |
| 11 | [NFR Matrix](11-nfr-matrix.md) | Latency budgets, throughput targets, and availability SLOs per flow and service |
| 12 | [Threat Model](12-threat-model.md) | STRIDE analysis — session takeover, event tampering, hand data disclosure, DoS, privilege escalation |
| 13 | [Observability Architecture](13-observability.md) | Per-service metrics, structured logs, distributed tracing, tournament round health dashboard |

### Mandatory Diagrams

| Diagram | Description |
|---------|-------------|
| [C4 Container](diagrams/c4-container.md) | System-wide container diagram |
| [Log-Before-Broadcast Sequence](diagrams/seq-log-before-broadcast.md) | Intra-context: PlayCard hot path in Room Gameplay |
| [Cross-Context Flow](diagrams/seq-cross-context-flow.md) | GameCompleted → Match → Tournament advancement |

### Design Alignment (§6.2)

| # | Document | Description |
|---|----------|-------------|
| 10 | [CHANGELOG-design](10-CHANGELOG-design.md) | Deltas from Design Checkpoint, with rationale and post-review corrections |

---

## Traceability

All service interfaces, event names, and API surfaces trace back to the [Commands & Events Catalog](../domain/docs/04-commands-and-domain-events.md) and [Bounded Contexts](../domain/docs/02-bounded-contexts-and-context-map.md) from the Design Checkpoint. Deltas are documented in [10-CHANGELOG-design.md](10-CHANGELOG-design.md).
