# ADR-005: Database-Per-Bounded-Context

**Status:** Accepted
**Date:** 2026-05-17

## Context

UnoArena has 6 bounded contexts (Identity & Session, Room Gameplay, Tournament Orchestration, Ranking & Statistics, Spectator View, Audit & Game Log), each with distinct data access patterns, consistency requirements, and scaling needs. Room Gameplay requires high write throughput (~100k writes/sec at peak for the event store). Identity & Session is read-heavy with low write volume. Audit is append-only with very high ingestion rate.

## Decision

Each bounded context owns its own **PostgreSQL database** (or dedicated store — MongoDB for Spectator View projections, ClickHouse for Audit are acceptable alternatives). No cross-context joins. All cross-context data access flows exclusively via **Kafka events** or **REST APIs**.

## Rationale

- **Independent scaling:** Room Gameplay's PostgreSQL can be provisioned for heavy write throughput (connection pooling, SSD, tuned WAL) without affecting Identity & Session's lighter workload.
- **Independent schema evolution:** Each context migrates its schema independently. A new column in `game_events` doesn't require coordinating with the tournament schema.
- **Fault isolation:** A runaway query or table lock in the audit database does not affect gameplay latency.
- **Enforced boundaries:** Accidental coupling via shared tables (a common "shared database" failure mode) is structurally impossible. Developers cannot write a JOIN across contexts.

## Alternatives Considered

1. **Shared database with schema-level separation:** Each context gets its own schema within one PostgreSQL instance. Simpler to operate (one DB to manage) but allows accidental cross-schema queries, shared connection pools, and shared I/O. A heavy audit workload degrades gameplay.
2. **Shared database with code-level enforcement (conventions):** Relies on developer discipline to avoid cross-context access. Breaks under pressure, especially during incident response when developers write ad-hoc queries.

## Consequences

- **Positive:** True data isolation. Independent scaling. Independent migration. Fault isolation. Structurally enforced boundaries.
- **Negative:** More databases to operate (6 PostgreSQL instances + optional MongoDB/ClickHouse). Cross-context queries are impossible — must use events or APIs (acceptable: CQRS read models handle cross-context views). Database credential management is more complex (mitigated by Vault/secrets manager).
