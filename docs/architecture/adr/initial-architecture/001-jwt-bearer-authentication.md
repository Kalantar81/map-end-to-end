# ADR-001 — JWT bearer authentication with timed token rotation

- **Status:** Accepted
- **Date:** 2026-09-24
- **Author:** architect
- **Source of truth:** `docs/architecture/initial-architecture.md` — §2.2, §2.4 (OQ #4, #6), §2.5 (A6, A12, A13), §3, §4, §5.4, §5.11, §6.4, §9
- **Scope:** `back/` auth module, `front/` `core/auth` + `core/session`

## Context

The PRD leaves the auth mechanism undefined (PRD OQ #4) and later adds a "refresh every 30 minutes"
requirement without saying what a session is. The MVP is a two-tier SPA + REST API (§2.1) with one
bounded context, no production hosting (A1) and trivial scale (A2). We need: a credential the SPA can
attach to every call, a way to keep a logged-in user working for a workday, and a defensible answer to
"what happens when the token is stolen" — without building a session store the MVP would not otherwise need.

## Decision

1. **Stateless JWT (HS256)** signed with `JWT_SECRET` (min 32 chars, validated at boot), presented as
   `Authorization: Bearer <accessToken>`. Issued by `POST /api/v1/auth/login`, validated by a Passport JWT
   strategy behind a **global `JwtAuthGuard` (`APP_GUARD`, deny-by-default)** with an explicit `@Public()`
   opt-out for login, `initdb`, health and Swagger (§9).
2. **Access-token TTL is 35 min** (`JWT_ACCESS_TTL`, values below `10m` are rejected at boot). The payload
   carries `sub`, `username`, `orgId`, plus two claims added for sessions: `sid` (session id, stable across
   every rotation) and `sae` (session absolute expiry, epoch seconds) — §5.4.
3. **Session keep-alive is timed HTTP rotation.** The client calls the body-less `POST /api/v1/auth/refresh`
   every **30 min**, scheduled from `expiresIn - 300` s (floor 60 s), i.e. 5 min of head-room. The server
   re-signs the *same* `sub`/`username`/`orgId`/`sid`/`sae` with a fresh `iat`/`exp`, clamping `exp` to `sae`.
   `SessionTokenService.rotate()` is the only code path that signs a non-login token (§9).
4. **The 8 h session is absolute.** `sae = iat + JWT_SESSION_MAX_AGE` (default `8h`) is set once at login and
   copied verbatim into every rotated token; rotation can never extend it. Past `sae` → `401 SESSION_EXPIRED`
   (log in again, do not retry). An already-expired access token is **not** renewable → `401 UNAUTHENTICATED`.
   Rate limit per `sid`: `AUTH_REFRESH_RATE_LIMIT` (default 10/min) via `@nestjs/throttler` → `429 RATE_LIMITED`,
   which must **not** log the user out.
5. **No refresh-token entity and no server-side session store** (§6.4). The currently-valid access token is
   itself the proof of session continuity, so no second credential is stored, hashed or transmitted.
6. **Token storage on the front:** an in-memory `signal` in `AuthService` plus a mirror in `sessionStorage`
   for reload survival; `AuthService.setToken()` is the single writer of both. Sessions are therefore per
   browser tab (A13), each tab running its own timer. `setTimeout` is not trusted alone: on
   `visibilitychange` → visible and on `window.focus` the client re-checks the stored token's `exp` and
   refreshes when under 5 min remain, all triggers sharing one single-flight request (§5.11, §8).

## Alternatives considered

| Alternative | Why rejected (§3) |
|---|---|
| Express session + server-side cookie store | Adds a session store and cookie/CORS complexity for zero MVP benefit, and is not stateless. |
| JWT in an httpOnly cookie | More secure against XSS, but needs CSRF protection and `SameSite`/credentials handling across `:4200`↔`:3000` — costs more than it buys at MVP scale. **Recorded as the intended hardening step**, not a rejection on merit. |
| Basic auth per request | Re-sends credentials on every request; unacceptable. |
| Server-pushed rotation over a WebSocket | Identical user-visible behaviour for a much larger bill (gateway + adapter, `Upgrade`-capable path through every proxy, handshake auth outside HTTP headers, reconnect/backoff, and an HTTP fallback that has to exist anyway). Revisit when a *second* real-time need appears — the channel is then additive. |
| Silent polling of `/auth/me` | Burns a request per interval for a payload that is not the point, and still needs a rotation endpoint. |
| Refresh token in an httpOnly cookie + `/auth/refresh` | Genuinely more secure (the refresh credential becomes unreadable by injected scripts) and remains the recommended hardening step; deferred for the same CSRF/`SameSite` cost as above. |
| Keep an 8 h token with no refresh at all | The previous state; superseded by the 30-minute requirement (A12). |
| Sliding-window session instead of a hard 8 h cap | With no revocation, a stolen token could be refreshed indefinitely. A sliding window only makes sense together with server-side sessions — `sid` is the hook for that later (A12). |

## Consequences

**Gains**

- Backend stays stateless: trivially restartable, horizontally scalable, no session store to operate; a backend
  restart loses nothing and the next refresh succeeds as usual.
- **The blast radius of a stolen token shrinks from 8 h to 35 min** — the most valuable effect of the change.
- Keep-alive adds no transport, no infrastructure and one optional dependency (`@nestjs/throttler`): ~20 lines of
  backend and ~60 of frontend, zero new frontend dependencies.
- Rotation is invisible in the UI — the `user` signal does not change, so no re-render and no navigation.

**Costs accepted**

- **JWT in `sessionStorage` is readable by injected scripts (XSS).** Accepted for an internal MVP with no sensitive
  data; **the single largest known security debt** and it must be revisited before any internet-facing deployment.
  Migration path: httpOnly cookie + CSRF protection (see the alternatives above).
- **No revocation.** Rotation shortens exposure but does not revoke: a stolen token stays valid until its 35 min
  `exp`, logout is client-side only (A6), and there is no session audit trail. The additive fix is designed in §6.4
  (a `sessions` table keyed by `sid`); it converts auth to stateful — one indexed lookup per request — which is why
  it is not paid for today.
- **The client owns the schedule.** Rotation happens only if the frontend timer runs; a server-initiated
  "terminate this session now" would need a frontend release or the push channel deferred above.
- **Five minutes is all the slack there is.** A suspended laptop or a frozen background tab past 35 min logs the
  user out — the `visibilitychange`/`focus` check removes the common cases but not the class of failure.
  "Session expired" redirects therefore become more frequent than with an 8 h token (behaviour unchanged,
  frequency not). Confirm this accepted failure mode with product (§10).
- **Rotation is neither revocation nor an extension:** the old token stays valid until its own `exp` (up to 5 min of
  overlap — which is exactly what makes rotation race-free for in-flight requests), and after 8 h everyone logs in again.

## Related

- Passwords: `bcrypt` cost 10, `passwordHash` is `select: false` and never serialized (OQ #6, ADR-002).
- Uniform `INVALID_CREDENTIALS` for unknown user and wrong password — no user enumeration (§5.3).
- `GET /auth/me` exists so a restored token can be validated on bootstrap (A9).
- Never log the `Authorization` header or an issued token; `sid` is logged for correlation (ADR-006).
- Optional, cheap hardening recorded for post-MVP (§4, §10): `@nestjs/throttler` on `POST /auth/login`
  (5 req/min/IP), and the httpOnly-cookie migration above.
