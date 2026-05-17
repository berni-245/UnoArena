# ADR-003: Dedicated Timer Service for Domain Timers

**Status:** Accepted
**Date:** 2026-05-17

## Context

UnoArena has four types of domain timers that are game-critical: the 5-second Uno! challenge window, the 5-second Wild Draw Four challenge window, the 60-second reconnection window, and the 30/60-second turn timer. These timers must **survive process crashes** — if a `game-engine` instance dies while a challenge window is open, the timer must still fire. Timer expiry side effects must be **idempotent** to handle at-least-once delivery.

## Decision

Introduce a dedicated `timer-service` that owns the scheduling and firing of domain timers. Deadlines are persisted in a PostgreSQL `timer_deadlines` table (written in the same TX as the game event that opens the window). `timer-service` polls the table for expired deadlines, sharded by time bucket, and fires idempotent expiry commands to `game-engine`.

The **Game aggregate retains the deadline state** (records `challengeWindowExpiresAt`, etc.). `timer-service` only owns the scheduling mechanism — it does not validate game rules.

## Rationale

- **Crash durability:** Deadlines are persisted in PostgreSQL. A `timer-service` crash → rescan on restart. A `game-engine` crash → aggregate rebuilt from event store; the deadline row fires normally.
- **Idempotent expiry:** Each deadline has a `version` field. If the window was closed early (e.g., a challenge was made), the version increments. A late-arriving `ChallengeWindowExpired` with a stale version is discarded by `game-engine`.
- **Separation of concerns:** `game-engine` focuses on game logic. `timer-service` focuses on reliable scheduling. Neither needs to understand the other's internals.

## Alternatives Considered

1. **In-process timers (setTimeout):** Simple but lost on crash. Unacceptable for 5-second windows in tournament games where fairness is critical.
2. **Kafka delayed delivery:** Kafka does not natively support message delivery delays with sub-second granularity. Workarounds (delay topics, time-based partitions) add complexity.
3. **Distributed scheduler (Temporal, Quartz):** Heavyweight for simple deadline polling. Temporal is excellent for complex workflows but overkill for "fire at time T" semantics. Adds infrastructure dependency.

## Consequences

- **Positive:** Crash-durable. Idempotent. Simple polling model. Scales by sharding time buckets.
- **Negative:** Polling adds ~100 ms latency on expiry (acceptable — human reaction time is larger). Separate service to deploy and monitor. Timer table needs periodic cleanup (fired rows pruned after 24 h).
