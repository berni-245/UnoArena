# TP Final E2 Enunciado

# **Architecture Checkpoint Assignment Instructions**

## **1) Context (must be included in your analysis) UnoArena: Global Real-Time Uno Platform & Massive Tournaments**

**Summary (for orientation):** Build the backend for a highly competitive Uno platform that supports ad-hoc rooms (2–10 players) and massive elimination tournaments (**up to 1,000,000 players**). A **core engineering challenge** in presentation/high-level-definition.md is the **first-round surge of over 100,000 simultaneous matches**—a coordinated **round kickoff** where on the order of **100k rooms can transition to in-progress within seconds**, not merely “large tournament” load spread over time or slow bracket eventual consistency. Design explicitly for that spike (see §6.5). The same document covers server-authoritative RNG and game logs, strict concurrency on room actions, real-time updates with spectator privacy, disconnection and forfeit policies, tournament round progression, security and rate limiting, analytics/read models, and ranking semantics consistent with your prior design work.

Your architecture must be **traceable** to that definition and to the **bounded contexts, commands, events, and consistency decisions** you produced in the Design Checkpoint (concrete expectations in §2).

## **2) Assignment objective**

Translate your **domain design** into a **concrete microservices-oriented architecture** for UnoArena: for each bounded context, specify deployable services, their responsibilities, **public interfaces** (APIs and/or messaging contracts), **inter-service communication patterns**, and **persistence** choices.

This checkpoint is about **solution architecture** (services, boundaries, integration, data ownership), not full implementation. You are expected to justify how the architecture preserves domain invariants, scales under the stated load assumptions, and handles failure modes you already analyzed at the domain level.

**Traceability** to the Design Checkpoint is **not** a vibe check: your **public async contracts** (topic/queue names, event types, payload ownership) must **match** your documented **domain events**—or **state the delta** (rename, split, merge) and record it in CHANGELOG-design.md (§6.2). Likewise, **synchronous APIs** (resources, RPCs, main operations) must **map** to your **command** (and, where relevant, **query**) catalog, or document the same kind of delta. A reviewer should be able to trace each important integration row (§6.3) to a named command or event in the design package.

**Invariants that must have an explicit architectural home**

A valid microservices diagram is not enough on its own. You must state **which deployable component or layer** (service, gateway, worker, projection pipeline) is responsible for each guarantee below, including how failures and restarts affect it—not only that the domain model mentions it.

- **Sequence-number enforcement** — where stale or replayed commands/events are rejected (service and layer).
- **Log-before-broadcast atomicity** — authoritative writes persisted before clients see updates (e.g., outbox, transactional boundary).
- **5-second Uno! challenge window** — timer ownership, expiry handling, and failure behavior.
- **60-second reconnection window** — how the window is tracked, persisted, and honored across process or node failure.
- **Single-active-session** — persisting a revoked token is not enough: how **live** SSE/WebSocket (or equivalent) connections for the old session are **terminated or forced to re-auth** via a **push-invalidation path** (e.g., Identity/Session → gateway/BFF/control channel), not only a database flag the client never reads.
- **Spectator projection** — spectators **never** receive hand data; where projection, APIs, or transport enforce the filter.
- **Match series coordination** — which component **persists tournament match state across individual games** (best-of-three style: scoreline after each game, starting the next game in the series, early termination at two wins, series winner); how **room/game completion** events or commands flow **from Room Gameplay** into **match** outcome tracking and what **starts** the subsequent game (so matches are not modeled as isolated one-off games with no cross-game state machine).
- **Abandoned-game vs. completed-game outcomes (Elo and tournaments)** — which component **detects** abandonment / forfeit-by-all-remaining-players **versus** normal completion; how **tournament forfeits** are recorded as **losses** for advancement; how **abandoned casual games** **exclude** Elo updates; and how that **distinction** is carried in **events or APIs** into the **Ranking / Elo** context so downstream consumers do not “accidentally” rate abandoned games.

**Also align with the Design Checkpoint non-negotiable rubric** for **Elo scope** (no tournament or abandoned casual games), **tournament advancement** (top three, series, tie-break), and consistent **match vs. game** terminology in interfaces and events—each must have a clear owning context at runtime.

## **3) Relationship to the Design Checkpoint**

- Your submission must include the **latest version of your design** (glossary, contexts, aggregates, commands/events, flows, edge cases, consistency strategy, open questions) **as it exists after any updates you made to align with this architecture**.
- If you **change** bounded context boundaries, aggregates, or events to fit the architecture, you must **update the design documents** accordingly and briefly **explain the delta** (what changed and why).
- If the design and architecture diverge without documented rationale, the submission is incomplete.

## **4) Scope and constraints**

- **In scope:** Service decomposition per context, interface definitions, sync/async integration, data stores per context, cross-cutting concerns at an architectural level (auth boundaries, idempotency, observability hooks), and diagrams that explain runtime behavior.
- **Out of scope (unless your course instructor says otherwise):** Exact **cloud SKUs**, instance-family shopping lists, full **CI/CD** pipelines, **line-by-line** framework configuration, or production **runbooks**. **In scope:** naming **representative** technologies (e.g. PostgreSQL, Kafka, Redis) whenever they **clarify** persistence, messaging, or caching—you are **not** required to commit to a vendor, but avoiding all concrete technology classes usually weakens the architecture narrative.
- **Client protocol:** **High level** means no frame-level specs, SDK pseudocode, or reconnection backoff tables—but you **must** still **name and justify** your **client connection model** (e.g. REST + SSE, WebSocket, hybrid) and how it lands on a **gateway or BFF** (see §6.3). Hand-wavy “clients use HTTP” with no realtime story is insufficient.

## **5) Mandatory methodology**

Use **clear architectural views** so reviewers can navigate the system:

- **Context view:** Context map aligned to services (which logical context maps to which deployable components).
- **Container view:** Major runnable components (services, gateways, workers, brokers) and trust boundaries.
- **Integration view:** For each pair of components that communicate, state the pattern (see deliverables).

EventStorming or domain narratives from the design checkpoint should be **referenced** when explaining critical flows (room completion → tournament advance, Elo updates, spectator projection, etc.).

## **6) Required deliverables**

Your submission **must** include all of the following.

### **6.1 Architecture of every bounded context**

For **each** bounded context from your design (rename or split contexts only with explicit rationale):

1. **Purpose and scope** — What business capability this context owns; what it does *not* own.
2. **Services (containers)** — One or more services/processes; name each and state its primary responsibility.
3. **Public interfaces**
    - **Synchronous:** REST/GraphQL/gRPC (or equivalent) — list main resources or RPCs, auth expectations, and versioning approach if any.
    - **Asynchronous:** topics/queues, event names, payload ownership (who is the producer/consumer), and idempotency keys or correlation identifiers where relevant.
    - **Internal-only** interfaces (if any) — clearly marked as not exposed outside the context.
4. **Dependencies on other contexts** — Upstream/downstream relationships, anti-corruption layers, and who owns which contract (e.g., published language vs. conformist).

For the **Room Gameplay** bounded context (and any **server-authoritative deck / RNG** context your design separates out), you must spell out how **presentation/high-level-definition.md’s log-before-broadcast rule** is satisfied: *every* authoritative state change is **durably appended to the immutable game log before** any broadcast to players, spectators, or downstream consumers. That ordering is a **hard integration constraint**, not a persistence detail—your narrative should name the mechanism (e.g., **transactional outbox**, **event-sourced command handling**, or another pattern you justify). Illustrate **log-before-broadcast** on a **mandatory intra-context sequence diagram** (see below).

You must also architect **domain timers** that survive process crashes and leader failover: at minimum the **5-second Uno! challenge window** and the **60-second reconnection window**. For each, document **which component schedules and owns the timeout** (game aggregate + persisted deadline, separate scheduler worker, broker delayed delivery, saga/process manager, etc.), **what happens if that node dies mid-window** (how the deadline is recovered or recomputed), and **how timeout side effects are idempotent** (e.g., deduplicating ChallengeWindowExpired / ReconnectionTimerExpired so a retried message does not apply penalties twice). Tie this to the owning bounded context(s) (Room Gameplay, Identity/Session, or a dedicated process you introduce) and reflect it in your integration table (§6.3).

For **single-active-session**, document what happens on a **new login** beyond revoking the old refresh/access token in storage: **how the system reaches the gateway, BFF, or realtime edge** that holds the previous session’s long-lived connections so those streams are **closed, errored, or unsubscribed promptly** (internal event, pub/sub fan-out, session stickiness + kill signal, etc.). A design that only invalidates rows in the session store, with no path to the process that owns the open socket, is **incomplete**.

For the bounded context that **owns tournament round lifecycle and match/room provisioning** (often “Tournament Orchestration” or equivalent in your design—use your name), you must architect **round kickoff** for the **first-round surge** (presentation/high-level-definition.md): **what component** transitions the round and **fans out** room creation or match assignment work; **how** on the order of **100k** rooms get **players or seeds attached** without a single choke point; **partial failure** handling (retry, compensate, dead-letter, idempotent room creation); and **thundering-herd** controls (sharded workers, rate-limited enqueue, staged rollout, backpressure) so the kickoff does not overwhelm brokers, gameplay services, or databases. This is **not** satisfied by §6.5 alone—you need a **mechanism**, not only magnitude estimates.

For the bounded context that **owns tournament analytics, player statistics, and bracket or leaderboard read models** (often a separate “Analytics” or “Tournament read model” context), you must address the **game.completed spike** at **round end** described in presentation/high-level-definition.md: how **fan-out** of completion events is **ingested** (partitioning, consumer groups, dedicated projection workers); how the **projection pipeline** absorbs the burst **without pushing backpressure** into **Room Gameplay** writers (async buffers, separate topics, read-side scaling); and how **bracket / standings views** remain **coherent** (ordering, idempotent updates, versioning, or acceptable staleness with explicit bounds). Reflect the main producer→consumer path in §6.3.

For the **Spectator View** bounded context (or the **deployable that owns the spectator projection** if your design merged it), you must give the same level of treatment as in the Design Checkpoint: **what information crosses the boundary** into the spectator read model and API/stream, **what is deliberately withheld** (hands, hidden draw pile, anything else private), and **which domain events** drive materialization or incremental updates (names should **trace** to your event catalog). State the **projection model** (CQRS read model, event-carried state transfer, snapshot + delta, etc.) and show that **privacy is enforced in the projection / query path**—a design that only tweaks **token claims** or bolts a “filter” onto the **player** realtime channel, without a **first-class spectator projection**, is **insufficient**.

**Diagrams:** C4-style container (or equivalent) diagrams are **strongly recommended** for the system as a whole. **Sequence diagrams are mandatory** at this minimum:

1. **Intra-context** — one diagram for a **Room Gameplay** hot path (e.g. play card, draw, or shuffle) that shows **log-before-broadcast** end-to-end within that context (and RNG/deck if separated).
2. **Cross-context** — one diagram that spans **at least two bounded contexts** and their deployables—e.g. **game completion → match / series outcome → tournament or round advancement** (and next-round kickoff if in scope), or **casual game completion → Ranking / Elo update**—so **inter-service** coordination is visible, not only intra-room flows.

### **6.2 Latest design package (aligned with the architecture)**

Include the **current** design artifacts in the repository (same structure as the Design Checkpoint or an improved one), such that a reader can verify:

- Ubiquitous language and context map still match the architecture.
- Commands/events still match the integration contracts you propose.
- Edge cases and failure paths are still addressed at the domain level, with architecture-level mitigations called out where new.

Provide a **CHANGELOG-design.md** (or section in README.md) summarizing updates made for this checkpoint. It must meet a **minimum bar** (not a single vague line):

1. **Enumerate by name** every design artifact you changed (file path or doc section). **Where the change maps to the Design Checkpoint**, also cite the **deliverable number and title** from design-checkpoint/assigment.md §5 (1 — Domain glossary through 8 — Open questions and assumptions)—e.g. *Deliverable 4 — Commands and domain events catalog:* added StartMatch because…. Entries like “updated context map” **without** *Deliverable 2 — Bounded contexts and context map* (when that is what moved) are too vague for grading.
2. For each change, state **why** it was made—specifically the **architecture or integration constraint** that required it (new service boundary, messaging contract, persistence split, etc.).
3. For each change, **confirm explicitly** that no **Design Checkpoint non-negotiable** domain guarantee was weakened or dropped (or, if you believe an invariant moved shape, document how it is still enforced and cite where in the updated design).

If you made **no** design edits after the Design Checkpoint, say so in one short paragraph and still **affirm (3)** for the package as a whole.

### **6.3 Communication patterns**

**Client connection model (mandatory).** In a short dedicated subsection (or clearly under the **gateway / BFF** in §6.1), **declare** which pattern clients use for realtime play and spectating—e.g. **REST + SSE**, a single **WebSocket** per session, **mixed** surfaces—and **why** (bidirectional needs, firewall/mobile constraints, team familiarity). State **which deployable** terminates long-lived connections, how **per-room ordering** is preserved on the stream, and how this model composes with **session invalidation** (how a superseded session loses its streams; tie to the push-invalidation path) and **spectator privacy** (separate channel, subscription scope, token claims, or projection path so spectators never subscribe to player-hand payloads).

**Rate limiting (mandatory).** presentation/high-level-definition.md expects **multi-layer** limits (**per IP**, **per user**, **per room/tournament** action) and **adaptive** throttling. You must map **each layer** to **concrete deployables**—e.g. **edge/API gateway**, **service mesh / sidecar**, **BFF**, **dedicated quota service**, **in-process** middleware in Room Gameplay or Tournament—not a single vague “we rate limit.” For **per-user** and **per-room/tournament** enforcement, explain how the limiter gets **principal identity** and **scope** (signed claims, introspection, trust boundary with Identity/Session, room id on the command path, etc.) relative to **authentication**. Optional rows in the integration table may reference shared stores (Redis, etc.) used for token buckets.

Document **explicitly**, for each significant integration:

| **From → To** | **Pattern** | **Rationale** | **Failure semantics (timeout, retry, DLQ, saga step, etc.)** |
| --- | --- | --- | --- |

Patterns to consider (use what fits; justify choices):

- **Synchronous request/response** (REST/GraphQL/gRPC/others) for command/query paths needing immediate validation.
- **Publish/subscribe or log-based messaging** for domain event propagation and fan-out.
- **Outbox / transactional messaging** (or an **event-sourced** write path with the same atomicity story) — for most teams this is **effectively required on the Room Gameplay command path**, because the product definition mandates **durable game-log append before** realtime fan-out; treating outbox only as a generic “pattern to consider” without covering that path is insufficient.
- **CQRS / read models** for brackets, leaderboards, spectator views.
- **Saga / process manager** (orchestrated or choreographed) for multi-context workflows such as round advancement or compensation after partial failure — **include at least one row** in the integration table for **time-bounded domain windows** (Uno! challenge and/or reconnection timer): who emits the “start window” fact, who arms the durable timeout, who consumes the expiry, and how duplicates are suppressed.
- **SSE / WebSocket gateway** patterns for real-time delivery (who projects what, and how ordering is preserved per room). **Include at least one integration-table row** (or an equivalent subsection) for **session invalidation → live connection**: producer (Identity/Session or auth service), consumer (gateway/BFF/realtime layer), and mechanism so superseded sessions cannot keep receiving gameplay or room streams.

### **6.4 Persistence layer per context**

For **each** context (or service, if you split storage per service), specify:

- **Primary store** (e.g., relational, document, key-value) and **what data** it owns (aggregates, projections, sessions).
- **Consistency model** — strong vs. eventual; transactional boundaries; multi-region considerations if you claim them.
- **Read models** — materialized views, caches, search indexes; how they are built and how stale they may be.
- **Retention and audit** — game log immutability, tournament audit needs, PII boundaries if applicable. For Room Gameplay, also show how the **primary store and transaction boundaries** implement **log-before-broadcast** (same commit as log append + outbox row, or single event-store append before relay, etc.), so a crash cannot leave clients having seen an update that never reached the log. You must also describe the **read path** for the **immutable game log** where it supports **dispute resolution** and audit (presentation/high-level-definition.md): **who** may query or export it (operators, automated replay jobs, compliance), **for what purpose**, and **how access is authorized** (roles, mTLS, break-glass, scoped APIs)—not only the write path.

Avoid a single shared database for all contexts unless you justify it as an intentional exception and show how boundaries are still enforced in code and schema.

### **6.5 Capacity sketch (mandatory)**

Provide **order-of-magnitude** reasoning tied to the product scale above—at minimum: **peak concurrent matches** in the first tournament round (**100,000+** simultaneous matches), **approximate concurrent rooms / players / spectators** you assume at that moment, **event or command rates** that matter for brokers and gameplay services, and **which components scale horizontally** vs. which are intentionally singleton or partitioned. **Account for spectators as a multiplier on realtime load:** long-lived **SSE/WebSocket** (or equivalent) connections often scale with **spectators**, who can **far outnumber** active players in popular rooms (order-of-magnitude **10:1** or higher is plausible); if you cap fan-out or use regional edges, say so. You do not need precise benchmarks or cloud pricing; you **do** need numbers or ranges that show your decomposition and integration choices are not hand-waving. Reference presentation/high-level-definition.md if you narrow or interpret the official figures.

## **7) Suggested additional deliverables (to demonstrate understanding)**

Items below are **not** required unless your instructor says otherwise. **Capacity** is already mandatory in §6.5. Use the tiers to prioritize: **strongly recommended** artifacts match the difficulty of UnoArena (scale, security, operations); **optional enrichment** is valuable but secondary.

### **Strongly recommended**

1. **Non-functional requirements (NFR) matrix** — Latency budgets, throughput targets, availability, and how each major flow meets them.
2. **Threat model (lightweight)** — STRIDE or similar for public APIs, session takeover, event tampering, and rate-limit bypass.
3. **Observability architecture** — Logging, metrics, tracing: what is emitted per service, correlation IDs across async flows, dashboards for tournament round health.
4. **Decision log (ADRs)** — Short Architecture Decision Records for the top 5–10 choices (broker vs. none, outbox, BFF vs. direct client-to-service, client connection model, etc.).

### **Optional enrichment**

1. **Local and test topology** — docker-compose or equivalent sketch: which dependencies developers need to run integration tests.
2. **API / AsyncAPI / event catalog** — Machine-readable stubs or OpenAPI fragments for critical endpoints; AsyncAPI or schema registry outline for events.
3. **Data migration and versioning** — How you evolve event schemas and API versions without breaking consumers.

## **8) Evaluation criteria**

Submissions will be evaluated on:

- **Coherence** — Services and interfaces match bounded contexts and domain events; no hidden “big ball of mud” without justification.
- **Interface quality** — Clear ownership of commands, queries, and events; contracts are usable by another team.
- **Integration appropriateness** — Communication patterns fit the problem (consistency vs. latency vs. scale).
- **Security enforcement placement** — **Multi-layer rate limiting** is mapped to **deployables** and trust boundaries (§6.3), not only mentioned in passing; **authentication** (where tokens are validated, what each tier trusts) is **architecturally grounded** in the container/integration views, not a floating “we use JWT.”
- **Data architecture** — Per-context persistence aligns with consistency needs; read models and writes are separated where needed.
- **Alignment** — Design package and architecture tell one story; changelog explains intentional changes.
- **Traceability** — Event names and command/API surfaces line up with the design package’s catalogs, or deltas are explicit (see §2); integration table entries map to named domain commands/events.
- **Operational realism** — Failure handling, idempotency, and observability are credible for real-time, high-concurrency gameplay. *Example of what fails this bar:* stating “we use a message broker” without describing what happens to an **in-flight PlayCard command** when the **room service restarts** during a **tournament final** (retry, idempotency, client redelivery, durable outbox, partial saga state, etc.).
- **Timer durability** — **5-second Uno!** and **60-second reconnection** timers have an **explicit architectural owner**, **survive** process crashes or leader failover (or document how deadlines are **recovered**), and **expiry side effects** are **idempotent** (§6.1); hand-wavy “the app sets a timer” without durability or deduplication fails this bar.
- **Scale credibility** — The mandatory capacity sketch (§6.5) supports claims about handling the **1,000,000-player** tournament and **100,000+** first-round simultaneous matches; unsubstantiated scale claims lower the score.

## **9) Submission format and deadline**

- Deliver as Markdown (and optional diagrams) in the course repository, with a root **README.md** or **index.md** that links all documents.
- Use diagrams (Mermaid, C4, or PNG) where they clarify containers and sequences.
- Submit the repository link according to course instructions.