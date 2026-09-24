# ADR-003 — `initdb`: an unauthenticated but feature-flagged, reset-style seed endpoint

- **Status:** Accepted
- **Date:** 2026-09-24
- **Author:** architect
- **Source of truth:** `docs/architecture/initial-architecture.md` — §2.2, §2.4 (OQ #2, #3, #9, #10), §2.5 (A5), §3, §4, §5.8, §6.3, §7
- **Scope:** `back/src/seed/`

## Context

The PRD requires a callable `initdb` endpoint, visible in Swagger, that produces exactly 10 users and 2
configurations with every user attached to an organization (US-4, US-5), and leaves two questions open:
is it protected (OQ #2) and is it idempotent (OQ #3). The endpoint writes over the whole dataset, so both
answers are security decisions, not conveniences.

## Decision

1. **Not auth-protected, but feature-flagged.** `POST /api/v1/admin/initdb` is `@Public()`, and a
   `SeedEnabledGuard` throws `404 SEED_DISABLED` unless `SEED_ENABLED=true`. The flag **defaults to `false`**
   and is part of boot-time env validation, so enabling seeding is an explicit, visible act.
2. **Idempotent by reset-then-insert.** In a single transaction: one statement
   `TRUNCATE TABLE configurations, users, organizations RESTART IDENTITY CASCADE` (one statement, because
   PostgreSQL refuses to truncate a table referenced by a foreign key on its own), then insert the fixed dataset.
   Repeated calls converge to the same known state — exactly 2 organizations, 10 users, 2 configurations.
   A failure rolls the whole thing back and leaves the previous state untouched (`500 INTERNAL_ERROR`).
3. **Fixed, deterministic dataset** (§6.3): organizations `Northwind Geo` / `Acme Mapping`; `user01…user05` → org 1,
   `user06…user10` → org 2 (OQ #10); configurations `ArcGIS Default` (`arcgis`) and `OpenLayers Default`
   (`openlayers`). All users share the documented password `Password123!` (A5), bcrypt-hashed.
4. **The response is a QA affordance** (§5.8): `status`, a `created` count object, the created organizations,
   users and configurations, and `defaultPassword` — so QA can log in from Swagger without reading the code.
5. **The logic lives in `SeedService`**, with the controller and guard as a thin shell, so a future CLI script is
   a wrapper over the same service rather than a second implementation (§7).

## Alternatives considered

| Alternative | Why rejected (§3) |
|---|---|
| Fully open endpoint, no flag | An unauthenticated data-wipe in every environment it ships to. |
| Auth-protected endpoint | Chicken-and-egg: you need a user to create users. It blocks QA on an empty database. |
| Upsert-only (non-destructive) seeding | Leaves drifted or extra rows behind, so the "exactly 10 users / exactly 2 configurations" success metric cannot be guaranteed. |
| A CLI script instead of an endpoint | The PRD explicitly requires a callable endpoint visible in Swagger (US-4, US-5). A CLI wrapper over `SeedService` may be added later. |

## Consequences

**Gains**

- QA and developers can bring any environment to a known state in one click from Swagger — the seed is the
  precondition of every manual and e2e test scenario, and the counts are directly assertable.
- With no auth on the endpoint, the flow works on a completely empty database.
- One transaction means no half-seeded state to debug.

**Costs accepted**

- **`initdb` is destructive.** If `SEED_ENABLED` is ever `true` in an environment with real data, that data is
  deleted. Mitigations are the default-`false` flag and boot-time config validation that makes the setting
  explicit; the rule is: **never enable seeding where real users exist** (A5).
- A shared, documented seed password is published in the response body. Acceptable only because the endpoint is
  dev/test-only — it is a second reason the flag must stay off outside local dev.
- Truncation invalidates tokens issued before the wipe: a still-valid token whose user no longer exists yields
  `401 UNAUTHENTICATED` from `GET /auth/me` and the frontend redirects to login (§5.5). Intended behaviour, worth
  knowing during test runs.
- The same env-gating pattern applies to Swagger (`SWAGGER_ENABLED`, §9) — two flags to get right per environment.
