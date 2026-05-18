# ADR-009: GDPR Pseudonymisation for Immutable Event Logs

**Status:** Accepted
**Date:** 2026-05-17

## Context

UnoArena stores immutable, append-only event logs (game events, audit trail) that contain `playerId` references. Under GDPR Article 17 (Right to Erasure) and equivalent LATAM regulations, players may request deletion of personal data. However, the game logs are cryptographically signed (HMAC chain per room) and immutable by design (INV-GL-01, INV-AT-01). True deletion would break the HMAC signature chain and compromise audit integrity for all other players in the same games.

## Decision

Use **pseudonymisation with salt-and-discard** rather than physical deletion:

1. On erasure request: replace all occurrences of `playerId` in game_log and audit_trail with `erased_player_{sha256(playerId + erasure_salt)}`.
2. The `erasure_salt` is a per-erasure random value, discarded after the pseudonymisation batch completes. This makes reversal computationally infeasible.
3. Delete the `player_identities` row (credentials, display name, email) from the Identity & Session context.
4. HMAC signatures are NOT recomputed. Instead, `signatureStatus` is set to `pseudonymised` on affected rows. The HMAC chain remains verifiable for structural integrity (no insertions/deletions) but individual row signatures will not match re-computation (expected).
5. Processing deadline: ≤30 days per GDPR Article 12(3).
6. Legal proceedings carve-out per Article 17(3)(b): if a game is subject to an active dispute or legal hold, pseudonymisation is deferred until the hold is released.
7. The erasure action itself is recorded in `compliance_meta_audit` (who requested, when, which playerId was pseudonymised, batch job ID).

## Alternatives Rejected

1. **Physical deletion of affected rows** — Breaks HMAC chain for all other players in the same games. Violates INV-GL-01 (append-only). Creates gaps in sequenceNumber that trigger integrity alerts.
2. **Encryption with key deletion** — Requires per-player encryption keys, complicates all read paths, adds key management overhead for every query. More complex than pseudonymisation for the same legal outcome.
3. **Tombstone records replacing original rows** — Still breaks HMAC chain. Provides less audit value than pseudonymised records which preserve event structure.

## Consequences

**Positive:**
- Complies with GDPR/LATAM right to erasure while preserving structural log integrity.
- HMAC chain remains verifiable for insertion/deletion detection (structural integrity preserved).
- Other players' game history remains intact and queryable.
- Reversibility is computationally infeasible (salt discarded).

**Negative:**
- HMAC signature verification for pseudonymised rows will report `signatureStatus: pseudonymised` (known-false) rather than `valid`. Audit tooling must handle this status.
- Compliance export bundles must document which rows are pseudonymised.
- 30-day processing window means a player's data persists briefly after request.
- Legal hold interactions add operational complexity.
