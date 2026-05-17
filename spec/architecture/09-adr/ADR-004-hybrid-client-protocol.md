# ADR-004: Hybrid Client Protocol (WebSocket + SSE)

**Status:** Accepted
**Date:** 2026-05-17

## Context

UnoArena has three client types with different communication needs: **players** need bidirectional real-time communication (send game commands, receive game state updates), **spectators** need read-only live event streams, and **admins** need standard request/response queries. The platform must also enforce spectator privacy (no hand data) and support session invalidation (superseded sessions lose their connections).

## Decision

Use a **hybrid protocol model**:
- **WebSocket** for players — bidirectional, authenticated, per-game connection through `api-gateway`.
- **SSE (Server-Sent Events)** for spectators — unidirectional server→client, public (no auth), served from `spectator-projection-service` through `api-gateway`.
- **REST** for admins and non-realtime operations.

## Rationale

- **WebSocket for players:** Bidirectional full-duplex is essential — commands and state updates flow on the same channel with sub-100 ms round-trip. Sequence-number reconciliation (client tracks `expectedSequenceNumber`) works naturally over a persistent connection.
- **SSE for spectators:** Read-only is sufficient. SSE is HTTP/2 compatible (multiple streams over one TCP connection), CDN-friendly, and has built-in `Last-Event-ID` reconnection. No auth overhead — spectator streams are public, only rate-limited per IP.
- **Privacy boundary:** Different protocols enforce structural separation. Spectators connect to SSE routes that serve data exclusively from the ACL-filtered spectator projection. No path exists from an SSE spectator connection to raw `game-engine` data.

## Alternatives Considered

1. **WebSocket for everyone:** Overkill for read-only spectators. WebSocket connections are bidirectional (wasteful) and harder to scale via CDN/proxy (no HTTP/2 multiplexing). Also requires auth for spectators (unnecessary overhead).
2. **SSE for everyone (commands via separate REST):** Players would need two channels (SSE for events + REST for commands). Higher latency for gameplay commands. Sequence-number reconciliation across two channels adds complexity.
3. **gRPC streaming:** Not browser-native. Requires client-side gRPC-web or a proxy translation layer. Adds client complexity for no benefit over WebSocket.

## Consequences

- **Positive:** Each client type uses the optimal protocol. SSE scales better for spectators (CDN, HTTP/2). Privacy enforcement is structural (different routes → different services → different data). Session invalidation composes naturally (gateway closes superseded WS connections; SSE is sessionless).
- **Negative:** Two real-time protocols to maintain in the gateway. Gateway must handle both WebSocket upgrade and SSE long-polling. Testing requires both protocol stacks.
