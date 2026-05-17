# ADR-008: API Gateway for Connection Termination and Edge Security

**Status:** Accepted
**Date:** 2026-05-17

## Context

UnoArena clients connect over the public internet using WebSocket (players), SSE (spectators), and REST (admins). Several security concerns must be addressed at the edge: JWT validation, per-IP rate limiting, TLS termination, and push-invalidation of superseded sessions. Additionally, when a player logs in on a new device, the old session's live WebSocket connection must be terminated promptly — not "eventually when the token expires."

## Decision

Introduce a dedicated **`api-gateway`** (Nginx/Envoy + custom middleware) as the single public entry point. The gateway:
1. Terminates all TLS, WebSocket upgrade, and SSE connections.
2. Validates JWTs on every request/upgrade (read-through to Redis session cache).
3. Enforces per-IP rate limiting (Layer 1) before authentication.
4. Enforces per-user rate limiting (Layer 2) after authentication.
5. Maintains an in-memory `sessionId → connectionRef` map.
6. Subscribes to Kafka `identity.sessions` topic. On `SessionInvalidated`, looks up the old session's connection and sends a WebSocket close frame (code 4001: `session_superseded`).

## Rationale

- **Single trust boundary:** All public traffic passes through one component. Internal services trust gateway-injected claims and communicate over mTLS/private network. No service individually validates JWTs.
- **Push-invalidation:** The `sessionId → connectionRef` map enables prompt connection termination on session supersession (~200 ms from login). Without a gateway, there is no centralized component that knows which process holds the old session's socket.
- **Edge rate limiting:** Per-IP rate limiting is applied before any authentication or business logic. Cheapest possible defense against DDoS and brute-force.
- **Protocol unification:** The gateway translates between client-facing protocols (WebSocket, SSE) and internal service communication (mTLS REST/RPC). Services don't need WebSocket handling logic.

## Alternatives Considered

1. **Direct client-to-service communication:** Each service validates JWTs independently. No edge rate limiting (or each service implements its own). No centralized connection map for push-invalidation — session revocation relies on token expiry (insufficient). Multiple public endpoints to secure.
2. **BFF (Backend-for-Frontend) per client type:** One BFF for players (WebSocket), one for spectators (SSE), one for admins (REST). Each BFF needs the same security features (JWT validation, rate limiting, push-invalidation). Duplicates security logic across three components.

## Consequences

- **Positive:** Single security enforcement point. Push-invalidation works. Edge rate limiting. Protocol translation for internal services.
- **Negative:** Gateway is a critical component in the request path — must be highly available (horizontal scaling, health checks, rolling deploys). Gateway must subscribe to `identity.sessions` Kafka topic and understand session lifecycle events (coupling to IS context, but limited to one event type). Per-instance connection map means `SessionInvalidated` must be consumed by every gateway instance.
