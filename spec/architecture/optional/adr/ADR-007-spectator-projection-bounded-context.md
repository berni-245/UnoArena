# ADR-007: Spectator Projection as First-Class Bounded Context

**Status:** Accepted
**Date:** 2026-05-17

## Context

Spectators must **never** see hand contents, draw pile state, deck seed, sequence numbers, or event signatures. The product definition requires that "spectators receive a filtered game view." A solution that merely adds a filter to the player API (e.g., checking token claims to strip fields) is insufficient: it places privacy enforcement in the transport layer, is brittle (a misconfigured claim could leak data), and couples the spectator read path to the gameplay write path.

## Decision

Implement spectator support as a **first-class bounded context** with its own deployable (`spectator-projection-service`), its own read model database, and an **Anti-Corruption Layer (ACL)** that transforms raw gameplay events into spectator-safe events before materialization. The spectator API and SSE feed serve data exclusively from this projection. No raw gameplay events are ever stored or forwarded.

## Rationale

- **Defense in depth:** Privacy is enforced at three levels: (1) ACL transformation strips private fields before materialization, (2) spectator database contains only safe projections, (3) separate SSE route serves only projection data.
- **Structural isolation:** A full database compromise of the spectator store reveals no hand data — it was never written there.
- **Independent scalability:** Spectator read load (5M+ SSE connections at peak) scales independently from gameplay write load. Read replicas and regional edge proxies handle fan-out.
- **Clean bounded context boundary:** The spectator context conforms to Room Gameplay's event schema (conformist relationship) and applies ACL on its side. `game-engine` is completely unaware of spectator concerns.

## Alternatives Considered

1. **Filter at API gateway/BFF:** Gateway would need to understand game domain (which fields are private). Violates separation of concerns. A configuration error leaks data.
2. **Token-claim-based filtering on the player API:** Spectators would access the same endpoints as players with a "spectator" claim that triggers field stripping. Risk: a crafted or leaked JWT could bypass the filter. Also couples spectator read load to the game-engine's request handling.
3. **Separate Kafka topic for spectator events (produced by `game-engine`):** `game-engine` would produce two versions of each event (full + spectator-safe). Couples game-engine to spectator concerns. Doubles event production. The game aggregate shouldn't know about spectators.

## Consequences

- **Positive:** Privacy enforced structurally. Independent scalability. Clean bounded context. Defense in depth.
- **Negative:** Additional service and database to operate. ~500 ms event delivery latency (Kafka consumer lag + materialization) — acceptable for spectators who are already viewing with inherent delay.
