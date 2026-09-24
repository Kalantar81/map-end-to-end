# ADR-006 — Logging: Nest's built-in logger to stdout, nothing else

- **Status:** Accepted
- **Date:** 2026-09-24
- **Author:** architect
- **Source of truth:** `docs/architecture/initial-architecture.md` — §2.2 (row "Logging"), §2.5 (A1), §4, §9
- **Scope:** `back/` (logger, global exception filter), `front/` (explicitly: nothing)

## Context

The only environments in the MVP are a developer laptop and docker-compose (A1) — there is no log collector,
no shared staging, no on-call. At the same time the session design adds events worth seeing (auth failures,
rate limiting, absolute session expiry), and the error envelope deliberately hides internals from the client
(ADR-004), so the detail has to live *somewhere*.

## Decision

1. **Nest's built-in logger, stdout only.** No logging library, no log file, and **no log table in the database** —
   the database holds domain data (the session audit trail sketched in §6.4 is a different, out-of-scope feature).
2. **The global exception filter logs every unhandled exception at `error` with its stack trace** and still returns
   only the generic `INTERNAL_ERROR` envelope to the client: the detail lives in the server log, never in the response.
3. **Levels for the known events:** auth failures at `warn`, **without the password** and with no request-body
   logging; `429 RATE_LIMITED` at `warn` (a healthy client calls `/auth/refresh` twice an hour, so hitting the limit
   means a client loop or abuse); `401 SESSION_EXPIRED` at `log`, because the 8 h deadline is the designed outcome,
   not an anomaly.
4. **Correlation by `sid`.** Session-related entries carry the `sid` claim (ADR-001): `sid` names a session but is
   not a credential. The `Authorization` header and the issued token are **never** logged.
5. **No client-side logging.** No error-reporting SDK, no `POST /client-logs` endpoint, and no `console.log` in
   committed code: failures surface as the UI error states (§8) and nothing is shipped anywhere. The token is never
   written to the console.

## Alternatives considered

| Alternative | Why rejected |
|---|---|
| A logging library (`pino`) with structured JSON | Buys nothing until something is actually collecting the output. It is the *second* step, after a `LOG_LEVEL` env var. |
| A log table in the database | Mixes operational data with domain data and needs retention/rotation that nobody would operate at this scale; the audit trail people actually ask for later is the `sessions` table in §6.4, not a log table. |
| Client error reporting (Sentry or `POST /client-logs`) | Deliberately deferred, not overlooked: with no shared environment there is no one to receive the reports, and it would add a third-party dependency and a privacy question to an internal MVP. |
| Returning exception details to the client | Leaks internals; the §5.3 envelope stays generic by design (ADR-004). |

## Consequences

**Gains**

- Zero dependencies, zero infrastructure, zero configuration: `docker compose logs` and the `ng serve`/`nest start`
  terminals are the whole observability story, which is proportionate to two environments.
- Nothing sensitive can leak through a log pipeline that does not exist.

**Costs accepted — this is known debt, stated plainly**

- Plain-text lines with **no configurable level, no JSON output, no request log** (method, path, status, duration),
  **no correlation beyond `sid`**, and **nothing at all from the browser**. Diagnosing a user-reported problem means
  reproducing it locally.
- **Trigger to revisit:** the first shared environment with a log collector. The cheap steps then, in order:
  (1) a `LOG_LEVEL` env var, (2) JSON output outside dev, (3) a small Nest interceptor for request lines.
  A logging library comes after those, not before.
