# Architecture Overview — Config Viewer MVP

> Status: Proposed (ready for teamlead decomposition). Author: architect. Date: 2026-09-23
> (revised the same day: `front/` was regenerated on Angular 22 — see §2.3 and §8).
> Source requirements: `docs/product/initial-prd.md` (PRD: Config Viewer — MVP).
> Russian version: `docs/architecture/initial-architecture.ru.md` (full translation, kept in sync).
> Related ADRs: `docs/architecture/adr/initial-architecture/` (to be added on request).

**This document is the single source of truth for the front↔back API contract.** Only the architect changes it.

---

## 1. Context

The PRD asks for a minimal but complete vertical slice: a user logs in, sees a dropdown of configurations
served by the backend, selects one, and sees its data — which in this MVP is exactly one field,
the map library type (`map-lib`: `openlayers` | `arcgis`). Supporting requirements: Swagger UI for all
endpoints, and an `initdb` seed endpoint producing exactly 10 users and 2 configurations, where every
user is associated with an organization (multi-tenancy foundation, US-1a).

The repository already exists and is **not** greenfield in terms of tooling:

| Area | What already exists | Implication |
|---|---|---|
| `front/` | Angular **22.1** CLI app, **standalone + zoneless** (`app.config.ts`, `app.routes.ts`, `app.ts`; no `app.module.ts`), SCSS, Vitest + jsdom, TypeScript 6.0 | Build on the standalone APIs (`provideRouter`, `provideHttpClient`, functional guards/interceptors, signals); do **not** re-introduce NgModules. PrimeNG must be the v22 line. |
| `back/` | NestJS **10**, Express platform, TypeScript 5.1, Jest + Supertest, `AppController`/`AppService` scaffold only | Add TypeORM (PostgreSQL), Swagger, JWT/Passport on top; no framework change needed. |
| Monorepo | Two independent `package.json`, no root workspace, single `.git` at root, `docs/` for agent artifacts | No shared build tooling. Type sharing between front and back must be solved without a workspace package (see §5.10). |
| DB | Nothing yet | PostgreSQL must be introduced, connection configuration included (PRD OQ #7). The PRD names MongoDB — a deliberate deviation, see §3. |

So this document does not invent a stack from scratch: it decides the *missing* pieces (persistence,
auth, API shape, module boundaries, seeding) and constrains them to what the existing scaffolds support.

The architecturally significant problems to solve are:

1. **Auth mechanism and credential storage** — undefined at product level (PRD OQ #4).
2. **Data model incl. multi-tenancy** — organizations as a table vs a plain field (PRD OQ #9–#11).
3. **`initdb` semantics** — protection and idempotency (PRD OQ #2, #3).
4. **API contract shape** — extensible enough for a growing configuration schema (PRD OQ #1, #5).
5. **Frontend structure** — where auth state, guards and HTTP concerns live in a standalone, zoneless Angular 22 app.
6. **UI kit under PrimeNG's new licensing** — from v22 PrimeNG is no longer MIT (§2.3, §3, A11).

---

## 2. Decision

### 2.1 High-level shape

A classic two-tier SPA + REST API with a single PostgreSQL database. No gateway, no BFF, no message bus,
no microservices — the MVP has one bounded context and a handful of endpoints.

```
┌──────────────────────────┐        HTTPS/JSON              ┌──────────────────────────┐
│  front/  Angular 22 SPA  │  ───────────────────────────▶  │  back/  NestJS 10 (REST) │
│  PrimeNG 22 UI           │   Authorization: Bearer <JWT>  │  Swagger UI at /api/docs │
│  AuthService + guard     │  ◀───────────────────────────  │  Global ValidationPipe   │
│  HTTP interceptors       │        JSON + error envelope   │  Global exception filter │
└──────────────────────────┘                                └────────────┬─────────────┘
        dev: ng serve :4200                                               │ TypeORM 0.3
        proxy /api → :3000                                                ▼
                                                             ┌──────────────────────────┐
                                                             │ PostgreSQL 16            │
                                                             │ organizations / users /  │
                                                             │ configurations           │
                                                             └──────────────────────────┘
```

**Request flow (happy path):**
`Login form → POST /api/v1/auth/login → JWT stored by front → GET /api/v1/configurations (Bearer)
→ dropdown populated → user selects → GET /api/v1/configurations/{id} (Bearer) → detail rendered.`

### 2.2 Technology decisions (summary)

| Concern | Decision | ADR |
|---|---|---|
| Authentication | Stateless JWT (HS256), `Authorization: Bearer`, issued by `POST /auth/login`, validated by a Passport JWT strategy + global `JwtAuthGuard` with `@Public()` opt-out. **Short-lived access token (35 min)** carrying a session identity (`sid`) and an absolute session deadline (`sae`, 8 h) | [ADR-001](adr/001-jwt-bearer-authentication.md) |
| Session keep-alive | **Timed HTTP rotation**: the client calls `POST /auth/refresh` with its current bearer token every **30 min** (5 min of head-room before the 35 min token expires), up to the 8 h absolute deadline. **No refresh-token entity** — rotation is stateless re-signing of the same `sid`/`sae` (§5.11, A12–A13) | ADR-001 |
| Token storage (front) | In-memory `signal` in `AuthService` + mirror in `sessionStorage` for reload survival; the rotated token replaces both on every refresh. No separate refresh token is ever stored | ADR-001 |
| Persistence | PostgreSQL 16 via `@nestjs/typeorm` 10 + `typeorm` 0.3 + `pg` driver, entities declared with decorators | [ADR-002](adr/002-postgresql-data-model-and-multitenancy.md) |
| Multi-tenancy | Dedicated `organizations` table; `users.organizationId` is a required foreign key (`uuid`). **No** org filtering of configurations in MVP | ADR-002 |
| Seed data | 2 organizations, 10 users split 5/5, 2 configurations | ADR-002 / [ADR-003](adr/003-initdb-seed-endpoint.md) |
| `initdb` | Unauthenticated but **feature-flagged** (`SEED_ENABLED`, default `false`); idempotent via *reset-then-insert* | ADR-003 |
| API style | REST, URI-versioned `/api/v1`, object envelopes (`{ items, total }`), unified error envelope with machine-readable `code` | [ADR-004](adr/004-api-conventions-and-error-format.md) |
| Config payload shape | `settings` `jsonb` column (`settings.mapLib`) separates configuration payload from metadata, so future fields are additive | ADR-002 / ADR-004 |
| API docs | `@nestjs/swagger` 7.x at `/api/docs` with `addBearerAuth()` so protected endpoints are callable from the UI | ADR-004 |
| Frontend structure | **Standalone** Angular 22 app (no NgModules, zoneless): `core/` singletons + lazy feature routes via `loadComponent`, functional `authGuard` and HTTP interceptors, signals for auth state; no state-management library | [ADR-005](adr/005-frontend-structure-and-primeng.md) |
| UI kit | PrimeNG 22.1.x configured with `providePrimeNG` + a `@primeuix/themes` 3.x preset (Aura), `primeicons` 8, `@angular/cdk` 22 (PrimeNG peer). **Not MIT from v22** — licensing decision in §2.3 / A11 | ADR-005 |
| Frontend tooling | `@angular/build` (esbuild) for build/serve; `ng test` → `@angular/build:unit-test` with the **Vitest** runner on jsdom; Node `^22.22.3 \|\| ^24.15.0 \|\| >=26.0.0` required | §2.3, A10 |
| Dev cross-origin | Angular dev proxy `/api` → `http://localhost:3000`; backend CORS also configurable for non-proxied use | ADR-005 |
| Package manager | **npm** for both projects (both already have `package-lock.json`) | — |

### 2.3 Verified version compatibility

Checked against the npm registry on 2026-09-23 (`npm view`; peer-dependency and `engines` ranges):

**Frontend — after the Angular 22 upgrade**

- Installed: `@angular/*@^22.1.0`, `@angular/cli` / `@angular/build@^22.1.8`, `typescript@~6.0.2`, `rxjs@~7.8`,
  `vitest@^4` + `jsdom@^28`, `npm@11.19.0` as `packageManager`. The app is standalone and **zoneless**
  (`zone.js` is no longer a dependency).
- **Node.js:** `@angular/core@22` declares `engines.node = "^22.22.3 || ^24.15.0 || >=26.0.0"`. Anything older —
  the machine this review ran on has **Node 18.20.8** — cannot install or build `front/`. `back/` is happy on the
  same runtimes (`@nestjs/cli@10` requires `>= 16.14`), so a single Node version serves the whole repo (A10).
- **TypeScript 6** enables `strict` by default, which is why the generated `tsconfig.json` no longer lists it.
  `back/` keeps its own `typescript@~5.1`; the §5.10 models are plain interfaces and compile unchanged under both,
  so the duplicated-models decision (§3) is unaffected.
- **Unit tests:** `ng test` runs the `@angular/build:unit-test` builder, whose default `runner` is `vitest`; with no
  `browsers` configured, specs execute in Node on jsdom. Karma/Jasmine are gone — the `describe/it/expect` in
  `app.spec.ts` come from `vitest/globals`, declared in `tsconfig.spec.json`.
- **PrimeNG:** latest is `primeng@22.1.1`, peers `@angular/core ^22.1.0`, `@angular/cdk ^22.1.0`,
  `rxjs ^6.0.0 || ^7.8.1` → matches the installed versions, but **`@angular/cdk` must be added explicitly**.
  `@angular/animations` is no longer a peer (PrimeNG moved to CSS animations; Angular's animations package is
  deprecated since v20.2). `primeng@21` peers `@angular/core ^21.0.7` and is therefore not installable here.
- **PrimeNG theming is code-only:** `providePrimeNG({ theme: { preset: Aura } })` from `primeng/config`, with the
  preset from `@primeuix/themes@^3.0` (`@primeuix/themes/aura`). The old `primeng/resources/themes/*.css` files
  (`lara-light-blue` and friends) no longer exist, so the only PrimeNG-related entry left in `angular.json > styles`
  is `primeicons/primeicons.css` (`primeicons@8.0.x`). `p-dropdown` was deprecated in favour of **`p-select`**.
- **PrimeNG licensing (needs a human decision):** from v22 PrimeNG ships under the PrimeUI dual model instead of
  MIT — a free **Community** license (individuals, students, non-commercial OSS, and organisations with < $1M
  revenue, < 5 developers, < 10 employees, < $3M VC funding; re-confirmed annually) or a commercial license
  (~$599 per developer). Versions ≤ 21 remain MIT but are pinned to Angular ≤ 21. See A11 and §3.

**Backend — untouched by this change, re-verified**

- `@nestjs/typeorm@10.0.2` peers `@nestjs/core ^8 || ^9 || ^10` and `typeorm ^0.3.0` → use `typeorm@^0.3` and the
  `pg@^8` driver (current 8.23.x). `typeorm` latest is 1.1.1 and 0.3.x sits on the `legacy` dist-tag; we deliberately take the 0.3 line that
  `@nestjs/typeorm@10` declares. Upgrade path: `@nestjs/typeorm@11` (peers Nest ^10 || ^11, `typeorm ^0.3 || ^1.0.0-dev`)
  + `typeorm@1` (Node ^20.19 || ^22.13 || >=24.11, covered by A10).
- `@nestjs/swagger@7.4.2` peers `@nestjs/core ^9 || ^10` and `reflect-metadata ^0.1.12 || ^0.2.0`
  (the backend already pins `reflect-metadata ^0.2.0`); we keep **7.4.x** as the conservative pairing for Nest 10.
- `@nestjs/jwt@10.2.0`, `@nestjs/passport@10.0.3`, `@nestjs/config@3.x` → all peer-compatible with Nest 10
  (`back/package.json` is still NestJS 10 + TypeScript 5.1 + Jest, exactly as assumed in §1).
- **Session keep-alive (§5.11) adds no new transport dependency on either side.** On the backend
  `POST /auth/refresh` reuses the already-selected `@nestjs/jwt@10.2.0`; the only addition is
  `@nestjs/throttler@10.0.1` (peers `@nestjs/common ^10.0.0`, `@nestjs/core ^10.0.0`, `reflect-metadata ^0.1.13 ||
  ^0.2.0`) for the per-`sid` rate limit, and it is optional. On the frontend the refresh schedule is a plain
  `setTimeout` plus the existing `HttpClient` — **zero new frontend dependencies**.

### 2.4 Answers to the PRD's open questions

| PRD OQ | Decision |
|---|---|
| #1 Config schema will grow | Payload isolated in the `settings` jsonb column; list endpoint returns metadata only; detail returns `settings`. New fields are additive and non-breaking. |
| #2 Is `initdb` protected? | Not auth-protected, but disabled unless `SEED_ENABLED=true`; returns `404 SEED_DISABLED` when off. Production sets it to `false`. |
| #3 Is `initdb` idempotent? | Yes — in one transaction it truncates the three tables and re-inserts the fixed dataset. Repeated calls converge to the same state (10 users, 2 configurations, 2 organizations). Destructive by design; guarded by the same flag. |
| #4 Auth mechanism | JWT bearer; **access token lives 35 min**, rotated by the client calling `POST /auth/refresh` every **30 min**, up to an **absolute session deadline of 8 h**. The token is kept in a signal + `sessionStorage` on the client. There is **no refresh-token credential and no server-side session store** — rotation re-signs the `sid`/`sae` claims the token already carries (§5.11, A12). See ADR-001 for the XSS trade-off and the migration path to httpOnly cookies. |
| #5 Dropdown labeling | `Configuration.name` is the label, `Configuration.id` the value; `description` is available as secondary text. |
| #6 Password storage | `bcrypt` (cost 10). `passwordHash` has `select: false`; it is never returned by any endpoint or serialized into a DTO. |
| #7 DB connection config | `@nestjs/config` + `.env`; `DATABASE_URL` required and validated at boot; `infra/docker-compose.yml` provides local PostgreSQL 16. |
| #8 Copy for error/empty states | Frontend implementation detail; contract only fixes error `code`s so copy can be mapped per code. |
| #9 How many organizations in MVP | **Two** — enough to exercise the future multi-tenant model without implementing filtering. |
| #10 User→org assignment in seed | `user01..user05` → org 1, `user06..user10` → org 2 (deterministic, documented below). |
| #11 Organizations as a table? | **Yes** — a real `organizations` table with a foreign key from users. Rationale in ADR-002. |

### 2.5 Explicit assumptions (gaps in the PRD)

These are architect assumptions, not product decisions. Flag to product-manager if any is wrong.

- **A1.** Deployment target for the MVP is local/dev (developer machines + QA demo). No production hosting,
  TLS termination, or CI/CD is designed here (PRD marks CI/CD out of scope).
- **A2.** Expected scale is trivial (tens of users, <100 configurations). No pagination, caching layer,
  read replicas or indexes beyond uniqueness/lookup indexes are designed.
- **A3.** "Configuration" is a global catalogue entity in the MVP: configurations are **not** owned by an
  organization yet, so `configurations` has no `organizationId` in MVP (the future filtering increment adds
  a `configuration_organizations` join table, which is an additive change).
- **A4.** Usernames are simple, login-name style (`user01`), case-insensitive (stored lowercased), not emails.
- **A5.** Seeded password is a single shared, documented value (`Password123!`) — acceptable because the seed
  endpoint is dev/test-only. Never enable seeding in an environment with real users.
- **A6.** No logout requirement is stated; we still include a client-side logout (drop the token) because
  it costs nothing and QA needs to switch users. No server-side token revocation (stateless JWT).
- **A7.** UI language is English, single locale (i18n is out of scope per PRD).
- **A8.** The frontend is served by `ng serve` in dev; no SSR, no Angular Universal.
- **A9.** `GET /auth/me` is added beyond the PRD's literal endpoint list because the "redirect to login when
  the session is invalid/expired" acceptance criteria (US-1, US-2) is much cleaner when the app can validate
  a restored token on bootstrap. It is a 15-line endpoint.
- **A10.** Toolchain prerequisite: Angular 22 requires Node `^22.22.3 || ^24.15.0 || >=26.0.0`. The machine used
  for this review runs Node 18.20.8, so nothing in `front/` installs or builds until Node is upgraded. Cheapest
  enforcement: a committed `.nvmrc` at the repo root; the same Node version also runs `back/`.
- **A11.** We assume the team qualifies for PrimeNG's free **Community** license (§2.3). If it does not, the
  options are a commercial seat or swapping the UI kit for Angular Material 22 (MIT). That swap is cheap on day
  one and expensive after two screens exist — nothing in the API contract or the backend depends on the UI kit.
- **A12 (confirmed by the user).** *What "refresh every 30 minutes" means.* The PRD says nothing about session duration or refresh, so
  the semantics are an architect decision: the **access token TTL drops from 8 h to 35 min**, and the client
  re-issues the token by calling `POST /auth/refresh` every **30 min** (5 min of head-room before expiry). The
  **total session length stays 8 h** — an absolute deadline (`sae` claim) that rotation cannot extend, so "refresh
  forever" is explicitly *not* what is built. A hard cap is chosen over a sliding window
  because there is no token revocation: with a sliding window a stolen token could be refreshed indefinitely. A
  sliding window only makes sense together with server-side sessions (the `sid` claim is the hook for that later).
- **A13.** Sessions are per browser tab. `sessionStorage` is per-tab, so tabs already hold independent sessions;
  each tab therefore runs its own refresh timer and rotates its own token. Acceptable at PRD scale (tens of
  users); the per-`sid` rate limit in §5.11 (default 10 req/min) caps the obvious abuse.

---

## 3. Alternatives considered

| Decision | Alternatives rejected | Why rejected |
|---|---|---|
| Stateless JWT bearer | (a) Express session + cookie store; (b) JWT in httpOnly cookie; (c) Basic auth per request | (a) adds a session store and cookie/CORS complexity for zero MVP benefit and is not stateless; (b) is more secure against XSS but needs CSRF protection and `SameSite`/credentials handling across `:4200`↔`:3000`, which costs more than it buys at MVP scale — documented as the intended hardening step in ADR-001; (c) re-sends credentials on every request and is unacceptable. |
| PostgreSQL + TypeORM | (a) MongoDB + Mongoose (as named in the PRD); (b) Prisma + PostgreSQL; (c) in-memory store | **Deviation from the PRD:** (a) the PRD names MongoDB explicitly, but the data here is relational (`users → organizations` with a required foreign key, US-1a) and the flexible part, `settings`, fits a `jsonb` column; the PRD must be updated (product-manager). (b) adds a separate schema file and code generation outside Nest's decorator model, whereas TypeORM plugs into Nest DI directly; (c) cannot satisfy the "real data in the database" success metrics. |
| Separate `organizations` table | (a) plain `organizationName` string on the user; (b) full tenant isolation (DB-per-tenant) | (a) is cheaper today but forces a data migration precisely where the PRD says migration cost is the reason to design now (US-1a); (b) is heavy multi-tenancy for a product with 2 seeded orgs and no filtering yet. |
| `settings` jsonb column | (a) flat `mapLib` field on the configuration; (b) free-form jsonb payload without DTO validation; (c) versioned schema registry | (a) mixes identity/metadata and payload, so every future field widens the root DTO; (b) throws away validation and Swagger typing; (c) is over-engineering for one field. |
| Feature-flagged, reset-style `initdb` | (a) fully open endpoint; (b) auth-protected endpoint; (c) upsert-only; (d) CLI script instead of an endpoint | (a) is an unauthenticated data-wipe in any environment where it ships; (b) creates a chicken-and-egg problem (you need a user to create users) and blocks QA; (c) leaves drifted/extra documents behind so the "exactly 10 / exactly 2" metric cannot be guaranteed; (d) the PRD explicitly requires a callable `initdb` endpoint visible in Swagger (US-4, US-5). A CLI script may be added later as a thin wrapper over the same service. |
| `{ items, total }` list envelope | bare JSON array | An array cannot carry pagination/metadata later without a breaking change; the envelope costs one line on the client. |
| No state-management library | NgRx / NgRx Component Store / Akita / SignalStore | Two screens and one selected entity; a store would be more code than the feature. An `AuthService` holding a `signal` plus `computed` is sufficient (ADR-005), and Angular 22 signals already cover the little shared state there is. |
| Modular monolith (single NestJS app) | Microservices (per resource or per domain); a gateway/BFF in front | The number of endpoints is not a reason to split: a growing API in one bounded context is still one deploy, and splitting it along the live foreign keys `organizations → users → configurations` would give a distributed monolith (the same coupling over HTTP, without transactions). Microservices pay off for different scaling profiles, fault isolation, independent release cycles of separate teams, or different data/tech requirements. Extraction triggers for this product: map tile/proxy/rendering (different CPU/traffic profile — the first realistic candidate), import/validation of large configurations as a background job, a tenant needing compliance-grade data isolation, organisation management growing into billing/SSO with its own release cycle, or more than two teams blocking each other's releases. The module boundaries and dependency rules that keep this option open are in §7. |
| Manual TS models on the front | (a) shared workspace package; (b) generated client from Swagger JSON; (c) decorated runtime classes (`json2typescript` `@JsonObject`/`@JsonProperty`) implementing the §5.10 interfaces | (a) requires converting the repo into an npm workspace (CLAUDE.md says no root `package.json`); (b) adds a codegen step — worth revisiting once the contract stabilises or a third resource appears — the intended tool is `openapi-typescript` (types only, no runtime client, so no build-order coupling), recorded as future work in ADR-004; (c) works — the §5.10 shapes compile and deserialise under TypeScript 6.0.3 — and is rejected on cost and strategy, not feasibility: a third copy of every field (back DTO → front interface → `@JsonProperty`), a custom converter per union (`MapLib`, `ApiErrorCode`), an error path mapping `JsonConvert`'s plain `Error` onto the §5.3 envelope — and it catches less than it promises, since extra server-side fields are silently dropped, so drift is detected in one direction only. `json2typescript@1.6.1` is also single-maintainer, last published March 2025, built against TypeScript 4.3, with no `exports` map and no `peerDependencies`, and rests on legacy `experimentalDecorators` rather than TC39 decorators — a longevity risk, not a blocker. Decisive: it forecloses (b), and codegen removes the cause of the duplication where runtime classes only detect symptoms. If runtime validation at the HTTP boundary ever becomes a real requirement (a third-party JSON producer, or independent front/back releases), evaluate schema-first `zod` instead — `z.infer` for the type, `z.strictObject()` to reject unknown keys. |
| PrimeNG 22 under the Community license | (a) stay on PrimeNG 21 (last MIT line); (b) Angular Material 22 (MIT); (c) no UI kit, hand-rolled components | (a) `primeng@21` peers `@angular/core ^21.0.7`, so "stay on MIT" really means "stay on Angular 21" — it is not installable on the regenerated scaffold; (b) is a genuine, license-clean alternative and the designated fallback if the Community license does not apply (A11), rejected today only because the UI is already specified in PrimeNG terms and the swap has no architectural consequences; (c) costs more than either option for two screens. |
| Angular 22 standalone + zoneless | (a) keep the Angular 15 NgModule app; (b) re-introduce NgModules on top of Angular 22 | **Reversal noted:** the previous revision of this document decided to stay on Angular 15 with NgModules because an upgrade was risky work with no MVP value. That reasoning no longer applies — `front/` has since been regenerated on Angular 22, so the upgrade is already paid for and (a) is moot. (b) would fight the framework's defaults (the scaffold has no `app.module.ts`, `HttpClientModule` and class interceptors are legacy) for zero benefit. The price of the reversal: a newer Node (A10), zoneless change-detection discipline, a Vitest test setup and the PrimeNG licensing question — all addressed below. |
| Client-timed `POST /auth/refresh` | (a) token rotation pushed by the server over a WebSocket channel; (b) silent polling of `/auth/me`; (c) refresh token in an httpOnly cookie + `/auth/refresh`; (d) keep the 8 h token with no refresh at all | (a) **WebSocket delivers the identical user-visible behaviour for a much larger bill**: a gateway plus a custom adapter, an `Upgrade`-capable path through every proxy/load balancer/dev proxy, handshake auth that cannot use HTTP headers, reconnect-with-backoff and a liveness watchdog on the client, and two code paths for session continuity (the socket plus an HTTP refresh that has to exist as a fallback anyway). What it would buy — the server owning the schedule and a free presence signal — is worth little while the only server-initiated event is "here is a new token". **Revisit when a second real-time need appears** (live configuration updates, notifications, presence, forced logout): the channel is then added *additively* and rotation can move onto it, with this endpoint kept as the fallback. (b) burns a request per interval for a payload that is not the point and still needs a rotation endpoint. (c) is the genuinely more secure option (the refresh credential becomes unreadable by injected scripts) and remains the recommended hardening step, but it needs CSRF protection and `SameSite`/credentials handling across `:4200`↔`:3000` — the same cost that deferred httpOnly cookies in the row above. (d) is the current state and is what the 30-minute requirement supersedes. The price of the chosen option is in §4. |

---

## 4. Consequences

**What we gain**

- Backend is stateless → trivially restartable, horizontally scalable later, no session store to operate.
- The front↔back contract below is complete enough that frontend and backend can start in parallel today;
  the frontend can mock against the documented shapes and switch to the real API without code changes.
- The data model already carries the user→organization relationship, so the future
  "organization decides which configurations are visible" increment is an *additive* change
  (a `configuration_organizations` join table + one query filter + one guard), not a migration.
- Adding configuration fields later touches `settings` only: one `settings` type, one DTO, one UI block.
- Swagger with bearer auth lets QA execute the full flow without the frontend (US-4, success metric).
- **The blast radius of a stolen token shrinks from 8 h to 35 min** — the single most valuable side effect of the
  change, and a partial mitigation of the `sessionStorage`/XSS debt below.
- **Session keep-alive adds no transport, no dependency and no infrastructure requirement**: one body-less
  `POST`, one client timer, and the same stateless JWT everywhere. It is ~20 lines of backend and ~60 of frontend.

**What it costs / what we accept**

- **JWT in `sessionStorage` is readable by injected scripts (XSS).** Accepted for an internal MVP with no
  sensitive data; mitigation path (httpOnly cookie + CSRF) documented in ADR-001. Must be revisited before
  any production/internet-facing deployment — this is the single largest known security debt.
- **Still no token revocation.** Rotation shortens exposure but does not revoke: a stolen token stays valid until
  its 35 min `exp`, and logout remains client-side only. True revocation needs server-side session state — the
  `sid` claim is the hook that makes adding it additive (§6.4).
- **The client, not the server, owns the schedule.** Rotation happens only if the frontend timer runs, and the
  interval is derived on the client from `expiresIn` (§5.11). Changing the cadence is still a backend TTL change,
  but adding a server-initiated "terminate this session now" would need either a frontend release or the push
  channel deferred in §3.
- **Five minutes is all the slack there is.** The token is rotated at 30 min and dies at 35 min; anything that
  delays the call by more than 5 min — a suspended laptop, a frozen or throttled background tab, a long offline
  spell — lets the token expire, and **an expired token cannot be refreshed** (§5.11 rule 1). The user is then
  redirected to `/login`. The `visibilitychange`/`focus` check in §8 removes the common cases cheaply but cannot
  remove the class of failure; the honest statement is "a tab that sleeps past 35 min logs you out".
- **Rotation is neither revocation nor an extension.** The old token stays valid until its own `exp` (up to 5 min
  of overlap — which is exactly why the 35 min TTL exceeds the 30 min interval, and what makes rotation race-free
  for in-flight requests), and no number of refreshes moves the 8 h `sae` deadline: after 8 h everyone logs in
  again, by design (A12).
- **"Session expired" redirects become more frequent than they were.** Users hit the US-1/US-2 redirect-to-login
  path after 35 min of a dead tab instead of after 8 h. The behaviour is unchanged, the frequency is not.
- **`initdb` is destructive.** If `SEED_ENABLED` is ever true in an environment with real data, that data is
  deleted. The flag defaults to `false` and boot-time config validation makes it explicit.
- **Type duplication** between `back/src/**/dto` and `front/src/app/core/api/api.models.ts`. Drift risk is
  mitigated by this document being normative plus a contract check in e2e tests; it is real debt.
- **The frontend now tracks the current Angular major.** No framework-upgrade debt on day one, but new
  obligations: Node ≥ 22.22.3 on every dev/CI machine (A10), zoneless change detection (UI updates must come
  from signals, the `async` pipe or an explicit `markForCheck()`), and a Vitest/jsdom test setup instead of
  Karma/Jasmine. Team habits from older Angular apps do not transfer automatically.
- **PrimeNG 22 is not MIT.** The UI kit now carries a licensing obligation (free Community license or a paid
  seat) that must be confirmed by a human before this ships anywhere commercial. It is a legal dependency, not
  a technical one, and the Angular Material fallback keeps it from becoming a lock-in (§2.3, §3, A11).
- **No org filtering yet** means any logged-in user sees all configurations. This is a deliberate,
  PRD-sanctioned simplification (US-2) — it must not be read as an authorization model.
- **Operational surface grows**: developers now need a running PostgreSQL (docker-compose) and a `.env`.

**Non-functional assessment**

| NFR | Target for MVP | How it is met / risk |
|---|---|---|
| Performance | p95 < 200 ms per endpoint locally | Trivial payloads, indexed lookups by `id`/`username`; bcrypt cost 10 dominates login (~50–100 ms) — intentional. |
| Scalability | Single instance, tens of users | Stateless API can be replicated behind a load balancer with no changes; PostgreSQL is the only stateful part. |
| Security | No plaintext passwords, all business endpoints authenticated, seed flag off by default | bcrypt hashing, `select:false` on `passwordHash`, global auth guard (deny-by-default with explicit `@Public()`), `ValidationPipe({whitelist:true, forbidNonWhitelisted:true})`, secret from env with boot-time validation. Optional hardening: `@nestjs/throttler` on `/auth/login` (5 req/min/IP) — recommended, cheap. |
| Reliability | Fail fast and visibly | Boot-time env validation; database connection failure aborts startup; global exception filter guarantees a typed error envelope so the UI can always render an error state. |
| Maintainability | Feature boundaries on both sides | Nest modules own their schema + service + controller; Angular feature routes are lazy-loaded (`loadComponent`) and depend on `core` only. |
| Cost of ownership | One process + one database | No infra beyond docker-compose PostgreSQL. |

---

## 5. API contract

Normative for both `front/` and `back/`. Changes go through the architect.

### 5.1 Conventions

- **Base URL:** `http://localhost:3000/api/v1` (dev). Global prefix `api`, URI versioning `v1`
  (`app.setGlobalPrefix('api')` + `app.enableVersioning({ type: VersioningType.URI, defaultVersion: '1' })`).
- **Frontend base URL:** `/api/v1` (relative) — the Angular dev proxy forwards `/api` to `:3000`,
  so no CORS in the default dev setup. It lives in `core/api/api.config.ts` as `API_BASE_URL` (the Angular 22
  scaffold ships no `src/environments/`; run `ng generate environments` only if per-environment builds appear).
- **Content type:** `application/json; charset=utf-8` in both directions.
- **Field naming:** `camelCase` in JSON. The PRD's `map-lib` is represented as **`mapLib`** (kebab-case is not
  idiomatic JSON/TS); this is a naming decision, not a semantic change.
- **IDs:** UUID (v4, generated by PostgreSQL) serialized as a 36-char **string** in the `id` field. Internal
  columns are never exposed.
- **Dates:** ISO-8601 UTC strings (`2026-09-23T10:15:30.000Z`).
- **Auth header:** `Authorization: Bearer <accessToken>` on every endpoint except those marked *public*.
- **Swagger:** `/api/docs` (JSON at `/api/docs-json`), with `addBearerAuth()`; protected endpoints show a lock. Served only when `SWAGGER_ENABLED=true` (default `false` outside local dev, like `SEED_ENABLED`); the Nest CLI Swagger plugin (`"plugins": ["@nestjs/swagger"]` in `nest-cli.json`) derives most `@ApiProperty` metadata from the DTO types.

### 5.2 Endpoint index

| # | Method | Path | Auth | Story |
|---|---|---|---|---|
| 1 | POST | `/api/v1/auth/login` | public | US-1 |
| 2 | GET | `/api/v1/auth/me` | bearer | US-1, US-2 (session validity) |
| 3 | GET | `/api/v1/configurations` | bearer | US-2 |
| 4 | GET | `/api/v1/configurations/{id}` | bearer | US-3 |
| 5 | POST | `/api/v1/admin/initdb` | public, flag-gated | US-5 |
| 6 | GET | `/api/v1/health` | public | ops convenience |
| 7 | POST | `/api/v1/auth/refresh` | bearer | session keep-alive, called every 30 min (§5.11, A12) |

### 5.3 Error envelope (all non-2xx responses)

```jsonc
{
  "statusCode": 401,
  "code": "INVALID_CREDENTIALS",      // machine-readable, stable; UI maps copy off this
  "message": "Invalid username or password.",
  "details": null,                     // string[] for validation errors, otherwise null
  "timestamp": "2026-09-23T10:15:30.000Z",
  "path": "/api/v1/auth/login"
}
```

Error codes used in the MVP:

| `code` | HTTP | When |
|---|---|---|
| `VALIDATION_ERROR` | 400 | Body/param failed `class-validator`; `details` lists messages |
| `INVALID_CREDENTIALS` | 401 | `/auth/login` with unknown username or wrong password |
| `UNAUTHENTICATED` | 401 | Missing/invalid/expired bearer token on a protected endpoint |
| `SESSION_EXPIRED` | 401 | `/auth/refresh` called after the absolute session deadline (`sae`) — the client must log in again, not retry |
| `RATE_LIMITED` | 429 | Too many `/auth/refresh` calls for one `sid` (§5.11) — the client backs off; it must **not** log out |
| `NOT_FOUND` | 404 | Unknown configuration id, or unknown route |
| `SEED_DISABLED` | 404 | `/admin/initdb` called while `SEED_ENABLED=false` |
| `INTERNAL_ERROR` | 500 | Unhandled exception (message is generic; details never leak internals) |

> Security note: `INVALID_CREDENTIALS` is returned for both "unknown user" and "wrong password" —
> no user enumeration. The UI shows one message for both (US-1).

### 5.4 `POST /api/v1/auth/login` — public

Request:
```jsonc
{ "username": "user01", "password": "Password123!" }
```
Validation: `username` — required, string, 3..64 chars, trimmed, lowercased server-side;
`password` — required, string, 8..128 chars. Extra properties are rejected (`400 VALIDATION_ERROR`).

`200 OK`:
```jsonc
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "tokenType": "Bearer",
  "expiresIn": 2100,                       // seconds
  "user": {
    "id": "3f1c2a7e-9b5d-4c8a-a1e2-7d0f4b6c9e21",
    "username": "user01",
    "displayName": "User 01",
    "organization": { "id": "a7d2e4c1-5b3f-4e6a-9c8d-1f0b2a3c4d5e", "name": "Northwind Geo" }
  }
}
```
Errors: `400 VALIDATION_ERROR`, `401 INVALID_CREDENTIALS`.

JWT payload (HS256, `JWT_SECRET`, `expiresIn = JWT_ACCESS_TTL`, default `35m`):
```jsonc
{
  "sub": "<userId>",
  "username": "user01",
  "orgId": "<organizationId>",
  "sid": "9c1f0b7e-4a2d-4f6b-8e3c-5d7a1b9c0e2f",   // session id — stable across every rotation
  "sae": 1790028800,                                // session absolute expiry (epoch s) — rotation may NOT extend it
  "iat": 1790000000,
  "exp": 1790002100                                 // 35 min
}
```
`orgId` is carried now, unused for filtering in the MVP, so the future increment needs no token change.
`sid` and `sae` are added by this revision: `sid` identifies the login session across rotations (useful for log
correlation today, and the hook for server-side revocation tomorrow — §6.4); `sae` is set once at login to
`iat + JWT_SESSION_MAX_AGE` (default `8h`) and is copied verbatim into every rotated token, which is what makes
"refresh forever" impossible without any server-side storage (A12). Rotation is therefore a pure function of the
presented token — no refresh-token entity exists (§5.11, §6.4).

### 5.5 `GET /api/v1/auth/me` — bearer

`200 OK` — the same `user` object as in the login response:
```jsonc
{
  "id": "3f1c2a7e-9b5d-4c8a-a1e2-7d0f4b6c9e21",
  "username": "user01",
  "displayName": "User 01",
  "organization": { "id": "a7d2e4c1-5b3f-4e6a-9c8d-1f0b2a3c4d5e", "name": "Northwind Geo" }
}
```
Errors: `401 UNAUTHENTICATED` (also when the user referenced by a still-valid token no longer exists,
e.g. after `initdb` wiped the DB — the front then redirects to login).

### 5.6 `GET /api/v1/configurations` — bearer

Returns **all** configurations (no org filtering in MVP — see A3). List items are metadata only;
`settings` is intentionally excluded so the list stays cheap as the schema grows.

`200 OK`:
```jsonc
{
  "items": [
    { "id": "c2f6d0e4-3b5a-4c8d-9f7e-1a2b3c4d5e6f", "name": "ArcGIS Default",     "description": "Baseline configuration using the ArcGIS map library." },
    { "id": "b1e5c9d3-2a4f-4b7c-8e6d-0f1a2b3c4d5e", "name": "OpenLayers Default", "description": "Baseline configuration using the OpenLayers map library." }
  ],
  "total": 2
}
```
Empty case (`US-2` empty state): `{ "items": [], "total": 0 }` with `200` — **not** a 404.
Errors: `401 UNAUTHENTICATED`.
Ordering: by `name` ascending (stable dropdown order).

### 5.7 `GET /api/v1/configurations/{id}` — bearer

`id` — UUID; malformed ids are rejected with `400 VALIDATION_ERROR` (via `ParseUUIDPipe`),
so a bad id never reaches the database.

`200 OK`:
```jsonc
{
  "id": "b1e5c9d3-2a4f-4b7c-8e6d-0f1a2b3c4d5e",
  "name": "OpenLayers Default",
  "description": "Baseline configuration using the OpenLayers map library.",
  "settings": {
    "mapLib": "openlayers"            // enum: "openlayers" | "arcgis"
  },
  "createdAt": "2026-09-23T10:00:00.000Z",
  "updatedAt": "2026-09-23T10:00:00.000Z"
}
```
Errors: `400 VALIDATION_ERROR`, `401 UNAUTHENTICATED`, `404 NOT_FOUND`.

**Extensibility rule (normative):** future configuration fields are added **inside `settings`**.
Clients must ignore unknown `settings` keys. `settings.mapLib` stays required.

### 5.8 `POST /api/v1/admin/initdb` — public, flag-gated

No request body. Behaviour (ADR-003): in a single transaction, run one statement `TRUNCATE TABLE configurations, users, organizations RESTART IDENTITY CASCADE`
(one statement, because PostgreSQL refuses to truncate a table referenced by a foreign key on its own), then insert the fixed dataset. Idempotent in the "converges to a known state" sense.

`200 OK`:
```jsonc
{
  "status": "ok",
  "created": { "organizations": 2, "users": 10, "configurations": 2 },
  "organizations": [
    { "id": "…", "name": "Northwind Geo" },
    { "id": "…", "name": "Acme Mapping" }
  ],
  "users": [
    { "username": "user01", "organization": "Northwind Geo" },
    "…10 entries…"
  ],
  "configurations": [
    { "id": "…", "name": "OpenLayers Default", "mapLib": "openlayers" },
    { "id": "…", "name": "ArcGIS Default",     "mapLib": "arcgis" }
  ],
  "defaultPassword": "Password123!"   // dev-only endpoint; lets QA log in without reading the code
}
```
Errors: `404 SEED_DISABLED` when `SEED_ENABLED !== true`; `500 INTERNAL_ERROR` on DB failure
(the whole seed runs in one transaction, so a failure rolls back and leaves the previous state untouched).

### 5.9 `GET /api/v1/health` — public

`200 OK` → `{ "status": "ok", "db": "up", "uptime": 123.4 }` (`db` is `"up" | "down"` from a `SELECT 1` ping
through the TypeORM connection). Used by QA/devs to confirm the stack is wired before debugging the UI.

### 5.10 Shared TypeScript models (duplicated verbatim on both sides)

Backend: DTO classes with `@ApiProperty`. Frontend: `front/src/app/core/api/api.models.ts`.

```ts
export type MapLib = 'openlayers' | 'arcgis';

export interface OrganizationRef { id: string; name: string; }

export interface AuthenticatedUser {
  id: string;
  username: string;
  displayName: string;
  organization: OrganizationRef;
}

export interface LoginRequest  { username: string; password: string; }
export interface LoginResponse {
  accessToken: string;
  tokenType: 'Bearer';
  expiresIn: number;
  user: AuthenticatedUser;
}

export interface ConfigurationSummary { id: string; name: string; description?: string; }
export interface ConfigurationListResponse { items: ConfigurationSummary[]; total: number; }

export interface ConfigurationSettings { mapLib: MapLib; }          // future fields go here
export interface ConfigurationDetail {
  id: string;
  name: string;
  description?: string;
  settings: ConfigurationSettings;
  createdAt: string;
  updatedAt: string;
}

export type ApiErrorCode =
  | 'VALIDATION_ERROR' | 'INVALID_CREDENTIALS' | 'UNAUTHENTICATED' | 'SESSION_EXPIRED'
  | 'NOT_FOUND' | 'SEED_DISABLED' | 'RATE_LIMITED' | 'INTERNAL_ERROR';

export interface ApiError {
  statusCode: number;
  code: ApiErrorCode;
  message: string;
  details: string[] | null;
  timestamp: string;
  path: string;
}

/* ── session keep-alive (§5.11) ────────────────────────────────────────────── */

/** Response of the body-less POST /auth/refresh. */
export interface SessionTokenPayload {
  accessToken: string;
  tokenType: 'Bearer';
  expiresIn: number;             // seconds until this token's exp (2100 with the default TTL)
  issuedAt: string;              // ISO-8601
  sessionExpiresAt: string;      // ISO-8601 absolute deadline (`sae`) — never moves
}

export type RefreshResponse = SessionTokenPayload;
```

### 5.11 `POST /api/v1/auth/refresh` — bearer

The session keep-alive endpoint. No request body; the only credential is the **current, still-valid** access token
in the `Authorization: Bearer` header. It returns a freshly signed token carrying the same session identity, which
is what keeps a logged-in user working past the 35 min access-token TTL without a second credential and without any
server-side session state.

`200 OK` — `SessionTokenPayload`:
```jsonc
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "tokenType": "Bearer",
  "expiresIn": 2100,                               // seconds until this token's exp
  "issuedAt": "2026-09-24T10:15:30.000Z",
  "sessionExpiresAt": "2026-09-24T18:00:00.000Z"   // absolute deadline (`sae`) — never moves
}
```

**Server rules (normative; one implementation — `SessionTokenService.rotate(claims)`):**

1. The presented token must be currently valid (signature + `exp`) — it is checked by the same global
   `JwtAuthGuard` as every other protected route. An **expired token is not renewable** → `401 UNAUTHENTICATED`;
   there is no separate refresh credential that outlives the access token, which is the deliberate consequence of
   having no refresh-token entity (§6.4). The client must log in again.
2. `now >= sae` → `401 SESSION_EXPIRED`, even when the access token itself is still valid. This is the 8 h
   ceiling. The client logs out and must **not** retry — it is the one case where the two 401 codes must be
   distinguished on the client.
3. Otherwise a new token is signed with the **same** `sub`/`username`/`orgId`/`sid`/`sae` and a fresh `iat`/`exp`
   (`JWT_ACCESS_TTL`). If `sae - now < JWT_ACCESS_TTL`, `exp` is clamped to `sae`, so a token never outlives its
   session and `expiresIn` simply shrinks to the remaining seconds as the deadline approaches.
4. The old token stays valid until its own `exp` (up to 5 min of overlap). This is intentional: it makes rotation
   race-free for requests that are already in flight. It also means rotation is **not** revocation.
5. Rate limit: `@nestjs/throttler`, `AUTH_REFRESH_RATE_LIMIT` (default 10) requests per minute per `sid` →
   `429 RATE_LIMITED`. A normal client calls this twice an hour.

Errors: `401 UNAUTHENTICATED`, `401 SESSION_EXPIRED`, `429 RATE_LIMITED`.

**Client rules (normative — the schedule lives on the client):**

1. After a successful login, and after a successful `restoreSession()`, schedule the next call at
   `expiresIn - 300` seconds (floor 60 s). With the default 35 min TTL that is exactly the required **30 min**.
   Deriving the interval from `expiresIn` rather than hard-coding 1 800 000 means a server-side TTL change needs no
   frontend release, and the clamp in server rule 3 is handled for free near the 8 h deadline.
2. Reschedule from every successful response; cancel the timer on logout.
3. **Do not trust the timer alone.** `setTimeout` is throttled in hidden tabs and does not run at all in a frozen
   or sleeping one, so on `visibilitychange` → visible and on `window.focus` the client compares the stored
   token's `exp` with `Date.now()` and refreshes immediately when less than 5 min remain (§8).
4. All refresh triggers share a single in-flight request, so a waking tab issues one `POST /auth/refresh`, not
   three.
5. Error handling: `401 SESSION_EXPIRED` from any endpoint → log out and redirect to `/login?returnUrl=<current URL>` with a
   "session expired" message, never retry (after login the user returns to the page they were on);
   `401 UNAUTHENTICATED` → one single-flight refresh attempt and, if it succeeds, replay the original request,
   otherwise log out; `429 RATE_LIMITED` → keep the current token and back off, do **not** log out.

**Lifecycle summary**

| Event | Behaviour |
|---|---|
| Login succeeds | `AuthService` stores the token and starts the refresh timer |
| Page reload | `restoreSession()` validates via `GET /auth/me`, then starts the timer |
| Token rotated | `AuthService.setToken()` updates the signal + `sessionStorage`; subsequent HTTP requests pick it up from the interceptor. No re-render, no navigation |
| Tab returns from background/sleep | `exp` is re-checked on `visibilitychange`/`focus`: refresh now if under 5 min remain, log out if it already expired |
| Offline or backend down at refresh time | retried on the next tick and on the next request; if the token expires first, the user is redirected to `/login` |
| Absolute deadline reached | the next refresh returns `401 SESSION_EXPIRED` → client logs out |
| User logs out | timer cancelled, signal and `sessionStorage` cleared. The token itself is **not** revoked (it dies with its 35 min `exp`) |
| Backend restarts | nothing is lost — no server-side session state exists, and the next refresh succeeds as usual |

---

## 6. Data model

### 6.1 Tables

**`organizations`**

| Field | Type | Constraints |
|---|---|---|
| `id` | uuid | pk, `gen_random_uuid()` |
| `name` | string | required, unique, 2..120 |
| `slug` | string | required, unique, lowercase kebab (`northwind-geo`) — stable handle for future config/seed references |
| `createdAt` / `updatedAt` | timestamptz | `@CreateDateColumn` / `@UpdateDateColumn` |

Indexes: unique on `name`, unique on `slug`.

**`users`**

| Field | Type | Constraints |
|---|---|---|
| `id` | uuid | pk, `gen_random_uuid()` |
| `username` | string | required, unique, stored lowercase, 3..64 |
| `passwordHash` | string | required, bcrypt cost 10, **`select: false`** |
| `displayName` | string | required, 1..120 |
| `organizationId` | uuid | **required**, FK → `organizations.id` (`ON DELETE RESTRICT`) (US-1a: exactly one organization) |
| `createdAt` / `updatedAt` | timestamptz | `@CreateDateColumn` / `@UpdateDateColumn` |

Indexes: unique on `username`, non-unique on `organizationId` (FK; anticipates org-scoped queries).
Case-insensitivity of `username` is enforced by normalisation, not by an expression index (TypeORM decorators cannot declare one, and `citext` needs an extension): lowercase + trim in the login DTO (`@Transform`) and in the entity's `@BeforeInsert()`, on top of a plain `unique` constraint.

**`configurations`**

| Field | Type | Constraints |
|---|---|---|
| `id` | uuid | pk, `gen_random_uuid()` |
| `name` | string | required, unique, 1..120 — the dropdown label (OQ #5) |
| `description` | string | optional, ≤500 |
| `settings` | jsonb | required; MVP: `{ mapLib: 'openlayers' \| 'arcgis' }` (required enum) |
| `createdAt` / `updatedAt` | timestamptz | `@CreateDateColumn` / `@UpdateDateColumn` |

Indexes: unique on `name`.
Constraint: `@Check("(settings->>'mapLib') IN ('openlayers','arcgis')")` on the entity — the only guard against a malformed seed, since the MVP has no write endpoints.
Future (not MVP): a `configuration_organizations` join table (`configurationId`, `organizationId`) for org-scoped visibility.

Schema management: TypeORM `synchronize` only outside production; versioned migrations before any shared environment. TypeORM must be configured with `uuidExtension: 'pgcrypto'` (its default `uuid-ossp` needs a superuser) so that `gen_random_uuid()` generates the primary keys.

### 6.2 Relationships

```
Organization 1 ──────< N User            (users.organizationId, required)
Configuration                            (global catalogue in MVP — no org link yet)
```

### 6.3 Seed dataset (fixed, deterministic — US-5)

Organizations (2):

| name | slug |
|---|---|
| Northwind Geo | `northwind-geo` |
| Acme Mapping | `acme-mapping` |

Users (10), password `Password123!` for all, bcrypt-hashed:

| username | displayName | organization |
|---|---|---|
| `user01` … `user05` | `User 01` … `User 05` | Northwind Geo |
| `user06` … `user10` | `User 06` … `User 10` | Acme Mapping |

Configurations (2):

| name | description | settings.mapLib |
|---|---|---|
| `ArcGIS Default` | Baseline configuration using the ArcGIS map library. | `arcgis` |
| `OpenLayers Default` | Baseline configuration using the OpenLayers map library. | `openlayers` |

(The list endpoint sorts by `name`, so `ArcGIS Default` appears first in the dropdown.)

### 6.4 No refresh-token table (decision)

**No new table is introduced for session keep-alive**, and that is a deliberate decision rather than an omission.

A refresh-token entity (`refresh_tokens`: `id`, `userId`, `tokenHash`, `expiresAt`, `rotatedFrom`, `revokedAt`,
`userAgent`) exists to solve two problems: (1) let a *long-lived* credential outlive a short-lived access token,
and (2) allow revocation and rotation-reuse detection. Neither applies here:

- Rotation is driven by a request that is **already authenticated by a currently-valid access token**. That
  token *is* the proof of session continuity, so no second credential has to exist, be stored, be hashed, or be
  transmitted. This is the main reason the design can stay storage-free.
- The 8 h ceiling is enforced by the `sae` claim, which rotation copies but never extends — no row is needed to
  remember when the session started.
- Revocation is *already* out of scope (A6, §4): the MVP has no logout-everywhere, no admin kill-switch and no
  compliance requirement. Adding a table that nothing reads would be pure ceremony.

Cost of this choice, stated plainly: **a stolen token cannot be revoked** before its 35 min `exp`, and there is no
audit trail of sessions. If either becomes a requirement, the additive change is small and already designed for:
create `sessions (sid PK, userId FK, issuedAt, absoluteExpiry, revokedAt, lastSeenAt)`, have login insert a row,
have `JwtAuthGuard` and `rotate()` reject tokens whose `sid` is revoked or missing, and have logout set
`revokedAt`. The `sid` claim added in §5.4 exists precisely so this can be done without reissuing tokens or
changing the API contract. It converts auth to stateful — one indexed lookup per request — which is why it is not
being paid for today.

---

## 7. Backend structure (`back/`)

```
back/src/
  main.ts                     # prefix+versioning, ValidationPipe, global filter, Swagger, CORS
  app.module.ts               # ConfigModule.forRoot(global+validated), TypeOrmModule.forRootAsync, feature modules
  common/
    filters/all-exceptions.filter.ts     # produces the §5.3 envelope for every error
    dto/api-error.dto.ts                 # Swagger model for the error envelope
    decorators/public.decorator.ts       # @Public() → skips the global JwtAuthGuard
  config/
    configuration.ts                     # typed config factory
    env.validation.ts                    # Joi/class-validator schema; fails fast at boot
  organizations/
    entities/organization.entity.ts
    organizations.module.ts
  users/
    entities/user.entity.ts
    users.service.ts                     # findByUsername(+hash), findById
    users.module.ts
  auth/
    auth.module.ts  auth.controller.ts  auth.service.ts   # POST /auth/login, GET /auth/me, POST /auth/refresh
    session-token.service.ts             # rotate(claims) — the ONE place that re-signs a non-login token
    strategies/jwt.strategy.ts
    guards/jwt-auth.guard.ts             # registered as APP_GUARD (deny by default)
    dto/{login.request.dto.ts,login.response.dto.ts,authenticated-user.dto.ts,session-token.dto.ts}
    decorators/current-user.decorator.ts
  configurations/
    entities/configuration.entity.ts
    configurations.module.ts  .service.ts  .controller.ts
    dto/{configuration-summary.dto.ts,configuration-list.dto.ts,configuration-detail.dto.ts}
  seed/
    seed.module.ts  seed.controller.ts   # POST /admin/initdb
    seed.service.ts                      # reset + insert; reusable from a future CLI script
    seed.data.ts                         # the fixed dataset above
    guards/seed-enabled.guard.ts         # throws NotFound(SEED_DISABLED) when flag is off
  health/health.controller.ts
```

**Session keep-alive adds no module.** `POST /auth/refresh` is a third route on `AuthController`, and
`SessionTokenService` is the single place that re-signs a token. A separate `session/` module would have to depend
on `auth` for claims and signing while `auth` depends on it for the controller — exactly the cycle that produces
`forwardRef`, and it buys nothing at this size.

Environment (`back/.env`, with a committed `back/.env.example`):

| Var | Example | Notes |
|---|---|---|
| `PORT` | `3000` | |
| `DATABASE_URL` | `postgres://config_viewer:config_viewer@localhost:5432/config_viewer` | required, validated |
| `JWT_SECRET` | *(dev value in `.env.example`)* | required, min 32 chars |
| `JWT_ACCESS_TTL` | `35m` | access-token lifetime and the source of `expiresIn`; the client refreshes 5 min before it expires, so values under `10m` are rejected at boot (replaces `JWT_EXPIRES_IN`) |
| `JWT_SESSION_MAX_AGE` | `8h` | absolute session deadline written to the `sae` claim; rotation cannot extend it |
| `AUTH_REFRESH_RATE_LIMIT` | `10` | `POST /auth/refresh` calls per minute per `sid` (§5.11); exceeded → `429 RATE_LIMITED` |
| `CORS_ORIGINS` | `http://localhost:4200` | comma-separated; used when not going through the dev proxy |
| `SEED_ENABLED` | `true` locally, `false` by default/prod | gates `/admin/initdb` |
| `SWAGGER_ENABLED` | `true` locally, `false` by default/prod | gates `/api/docs` and `/api/docs-json` |

`infra/docker-compose.yml` (repo root) runs `postgres:16` on `5432` with a named volume (plus a separate `config_viewer_test` database for e2e, created by a script mounted into `/docker-entrypoint-initdb.d`, since the image creates only the `POSTGRES_DB` database) — the only infra piece.

**Module boundaries (keep the monolith modular and extractable):**
- Each feature is a Nest module with explicit `exports`: only its service and DTOs leave the module — never entities or repositories.
- Dependencies point one way: `auth → users → organizations`, `configurations → organizations`; `seed` may use all data modules and nothing depends on `seed`; `common/` knows nothing about features. No `forwardRef` — it signals a wrong boundary.
- Cross-module reads go through the owning module's service, never via `@InjectRepository` of another module's entity, and never as a join across the boundary (reference by id only).
- Enforce the rules with ESLint (`no-restricted-imports` on feature folders); otherwise they erode within months.
- Prepare for extraction: versioned migrations instead of `synchronize` (§6.1), idempotent writes.

---

## 8. Frontend structure (`front/`)

The regenerated scaffold is Angular 22: standalone components, **zoneless** change detection (`zone.js` is not a
dependency), `@angular/build` (esbuild) for build/serve and Vitest for unit tests. There is no `app.module.ts`
and none is introduced.

```
front/src/
  main.ts                     # bootstrapApplication(App, appConfig)
  styles.scss                 # app styles only — PrimeNG theming is code-side (see below)
  app/
    app.config.ts             # ApplicationConfig providers:
                              #   provideBrowserGlobalErrorListeners()
                              #   provideRouter(routes)
                              #   provideHttpClient(withInterceptors([authInterceptor, errorInterceptor]))
                              #   providePrimeNG({ theme: { preset: Aura } })
                              #   provideAppInitializer(() => inject(AuthService).restoreSession())
    app.routes.ts             # '' → /configurations ; 'login' ; 'configurations' (authGuard) ; '**' → ''
    app.ts / app.html / app.scss          # shell component: <router-outlet>
    core/
      api/api.config.ts       # export const API_BASE_URL = '/api/v1'
      api/api.models.ts       # §5.10 types (contract mirror)
      auth/auth.service.ts    # login(), logout(), restoreSession(); state in signals: user(), isAuthenticated()
      auth/token.storage.ts   # sessionStorage read/write/clear — the only place that touches storage
      auth/auth.guard.ts      # authGuard: CanActivateFn → redirect to /login with returnUrl
      http/auth.interceptor.ts    # HttpInterceptorFn — adds Authorization: Bearer when a token exists
      http/error.interceptor.ts   # HttpInterceptorFn — 401 → logout + redirect; maps ApiError for the UI
      session/session.service.ts            # start()/stop(); refresh timer + visibilitychange check
      services/configurations.service.ts    # list(), getById()
    features/
      auth/login-page.ts                        # standalone component, lazy-loaded
      configurations/config-selection-page.ts   # standalone component, lazy-loaded
  proxy.conf.json             # /api → http://localhost:3000
```

- **Routing/guard:** one lazy route per screen —
  `{ path: 'login', loadComponent: () => import('./features/auth/login-page').then(m => m.LoginPage) }`, and the
  same for `configurations` with `canActivate: [authGuard]`. `authGuard` is a functional `CanActivateFn` using
  `inject(AuthService)` / `inject(Router)`; unauthenticated navigation redirects to `/login?returnUrl=…` (US-1).
  Use `loadChildren` + a feature-level `routes.ts` only once a feature outgrows a single route.
- **Session restore:** `provideAppInitializer(() => inject(AuthService).restoreSession())` reads the token from
  `sessionStorage` and validates it with `GET /auth/me`; a 401 clears it silently. This avoids a flash of an
  authenticated shell.
- **Interceptors:** functional `HttpInterceptorFn`s, registered once via
  `provideHttpClient(withInterceptors([authInterceptor, errorInterceptor]))` — order matters: attach the token
  first, map errors/handle 401 second. No `HttpClientModule`, no `HTTP_INTERCEPTORS` multi-provider.
- **State under zoneless CD:** `AuthService` holds `user = signal<AuthenticatedUser | null>(null)` with
  `isAuthenticated = computed(() => this.user() !== null)`; page components expose signals and use the `async`
  pipe for one-shot HTTP streams. Every UI update must originate from a signal, an `async` pipe or an explicit
  `markForCheck()` — a bare `subscribe(() => this.x = …)` is not guaranteed to repaint.
- **Screens:**
  - *Login* — PrimeNG `p-inputtext`/`p-password`/`p-button`, reactive form, required-field validation
    (submit blocked + per-field messages), server error shown in a `p-message` (US-1).
  - *Configuration selection* — **`p-select`** (the v22 successor of the deprecated `p-dropdown`) bound to
    `ConfigurationSummary[]` (`optionLabel="name"`, `optionValue="id"`), states: loading (spinner),
    empty ("No configurations available"), error + Retry button, loaded. On selection change: fetch detail,
    cancel/ignore stale responses with `switchMap`, clear the previous detail before rendering the new one
    (US-2, US-3).
  - *Detail* — a card showing `Map library: OpenLayers | ArcGIS` (label mapped from `settings.mapLib`),
    plus an explicit not-found/error state.
- **Dev proxy:** `front/proxy.conf.json` → `{"/api": {"target": "http://localhost:3000", "secure": false}}`,
  referenced from `angular.json` as `serve.options.proxyConfig` — unchanged mechanism under the
  `@angular/build:dev-server` builder.
- **Dependencies to add:** `primeng@^22.1`, `@primeuix/themes@^3`, `primeicons@^8` and `@angular/cdk@^22`
  (a PrimeNG peer that is not installed automatically). **Not** `@angular/animations`. Theming is code-only:
  `providePrimeNG({ theme: { preset: Aura } })` with `Aura` from `@primeuix/themes/aura`; the only style entry to
  add in `angular.json` is `primeicons/primeicons.css` — the `primeng/resources/**` theme CSS files are gone.
- **Session keep-alive (§5.11):** `SessionService` is started by `AuthService` after a successful login or a
  successful `restoreSession()`, and stopped on logout — page components know nothing about it. It owns one
  `setTimeout` scheduled at `expiresIn - 300` s (floor 60 s, i.e. 30 min with the default TTL), calls
  `POST /auth/refresh`, and hands the response to `AuthService.setToken(payload)`, which updates the token
  `signal` and `sessionStorage` in one place — the existing `authInterceptor` then picks the new token up with no
  change. **The `user` signal does not change, so rotation triggers no re-render and no navigation.**
- **Background tabs — the one real pitfall.** Browsers clamp `setTimeout` in hidden tabs (≥ 1 min, and a frozen
  or sleeping tab does not run timers at all), so a 5 min margin cannot rest on the timer alone. The fix is
  deliberately small: `SessionService` also subscribes to `document.visibilitychange` (→ visible) and
  `window.focus`, decodes the stored token's `exp` once, and refreshes immediately when less than 5 min remain.
  If `exp` has already passed there is nothing to refresh — it clears the session and redirects to `/login`
  (§4, accepted consequence).
- **Single-flight:** timer, visibility check and interceptor all go through one shared in-flight request
  (`shareReplay(1)` over the refresh call), so a waking tab issues one `POST /auth/refresh`, not three.
- **Interceptors:** `authInterceptor` attaches the current token; `errorInterceptor` maps `ApiError` and, on
  `401 UNAUTHENTICATED` (excluding `/auth/login` and `/auth/refresh` themselves), makes **one** single-flight
  refresh attempt and replays the request — if that fails it calls `AuthService.logout()` and navigates to
  `/login`. `401 SESSION_EXPIRED` logs out immediately with no retry; `429 RATE_LIMITED` surfaces as an error but
  must never log the user out. Under zoneless CD every one of these paths must end in a signal write — never a
  direct field assignment.
- **Never leak a timer:** `SessionService` is `providedIn: 'root'`, clears its timeout and removes its event
  listeners in `ngOnDestroy` / `DestroyRef` and on logout. A refresh timer that survives logout is the most likely
  bug here and is worth an explicit unit test.
- **No `SharedModule`:** each standalone component imports exactly the PrimeNG components it uses
  (`Button`, `InputText`, `Password`, `Select`, `Card`, `Message`, `ProgressSpinner`).

---

## 9. Cross-cutting concerns

- **Validation:** `app.useGlobalPipes(new ValidationPipe({ whitelist: true, forbidNonWhitelisted: true, transform: true }))`;
  its errors are converted to `VALIDATION_ERROR` with `details`.
- **Authorization model:** global `JwtAuthGuard` (`APP_GUARD`) → every route is protected unless annotated
  `@Public()` (login, initdb, health, Swagger). Deny-by-default prevents "forgot the guard" bugs.
- **Swagger exposure:** `/api/docs` and `/api/docs-json` are registered only when `SWAGGER_ENABLED=true`; otherwise they do not exist (404), the same env-gated logic as `SEED_ENABLED`. Without it the full API surface is public in any non-local deployment.
- **Serialization:** DTO-mapped responses only; never return TypeORM entities directly (prevents
  `passwordHash` leaks).
- **Logging:** Nest's built-in logger; log auth failures at `warn` without the password; no request-body logging.
- **Testing:**
  - back — unit tests for `AuthService` (hash compare, token payload) and `ConfigurationsService`;
    e2e (Supertest + a dedicated `config_viewer_test` PostgreSQL database from docker-compose) covering login 200/401, protected 401, list, detail 200/404,
    initdb counts. The e2e suite is also the contract regression guard.
  - front — Vitest on jsdom (`ng test`): unit tests for `AuthService`, the functional `authGuard` and the
    interceptors, configured in `TestBed` with `provideHttpClient(withInterceptors([...]))` +
    `provideHttpClientTesting()` (`HttpClientTestingModule` is legacy); component tests for the three states of
    the selection page. Zoneless: `await fixture.whenStable()` before asserting rendered output.
- **Refresh endpoint security (§5.11):** `POST /auth/refresh` is protected by the same global `JwtAuthGuard` as
  every other business route — there is no second credential, no cookie and no special-case guard, so the attack
  surface added by keep-alive is one throttled `POST`. Keep the `sae` check inside `rotate()` rather than in the
  controller, throttle per `sid`, and never log the `Authorization` header or the issued token.
- **Token rotation is one code path:** `SessionTokenService.rotate()` is the only place that signs a non-login
  token. Two implementations of the `sae` clamp would guarantee a security bug.
- **Testing session keep-alive** (in addition to the testing bullet above):
  - back — unit tests for `rotate()` (claims preserved, `sae` never extended, `exp` clamped to `sae`, expired
    token rejected with `UNAUTHENTICATED`, past-`sae` token rejected with `SESSION_EXPIRED`); an e2e test that
    logs in, calls `POST /auth/refresh`, and verifies the new token is accepted by a protected endpoint while the
    old one still works until its `exp`. Use a short `JWT_ACCESS_TTL`/`JWT_SESSION_MAX_AGE` in tests rather than
    waiting 30 minutes.
  - front — unit tests for `SessionService` with fake timers: the call fires at `expiresIn - 300` s, a successful
    response updates the signal and `sessionStorage`, `SESSION_EXPIRED` logs out, logout cancels the timer and
    detaches listeners, the `visibilitychange` path refreshes a nearly-expired token, and concurrent triggers
    produce exactly one request.
- **Definition of "contract-compliant":** responses match §5 exactly (field names, envelope, error `code`s).

---

## 10. Tasks for the team

**backend-developer (`back/`)**
1. Add deps: `@nestjs/config`, `@nestjs/typeorm@^10.0.2`, `typeorm@^0.3`, `pg@^8`, `@nestjs/swagger@^7.4`, `@nestjs/jwt@^10.2`,
   `@nestjs/passport@^10`, `passport`, `passport-jwt`, `bcrypt`, `class-validator`, `class-transformer`
   (+ `@types/passport-jwt`, `@types/bcrypt` as dev).
2. Bootstrap (`main.ts`): global prefix `api`, URI versioning v1, `ValidationPipe`, global exception filter,
   CORS from `CORS_ORIGINS`, Swagger at `/api/docs` with `addBearerAuth()` (registered only when `SWAGGER_ENABLED=true`; enable the `@nestjs/swagger` Nest CLI plugin in `nest-cli.json`).
3. `config/` + `.env.example` + boot-time env validation (fail fast if `DATABASE_URL`/`JWT_SECRET` missing).
4. Entities + modules: `organizations`, `users`, `configurations` per §6 (indexes included). Configure TypeORM with `uuidExtension: 'pgcrypto'`, add `@Check` and username normalisation per §6.1, plus a `data-source.ts` and `typeorm` CLI migration scripts (needed before any shared environment).
5. Auth module: `POST /auth/login`, `GET /auth/me`, JWT strategy, `JwtAuthGuard` as `APP_GUARD`,
   `@Public()` decorator, bcrypt verification, uniform `INVALID_CREDENTIALS`.
6. Configurations module: list + detail endpoints, DTO mapping, `404 NOT_FOUND`, id validation.
7. Seed module: `POST /admin/initdb`, `SeedEnabledGuard`, reset-then-insert of the §6.3 dataset,
   response per §5.8.
8. `GET /health`; `infra/docker-compose.yml` with `postgres:16` and an init script creating `config_viewer_test`.
9. Tests per §9; verify every §5 response shape. Commit a snapshot of `/api/docs-json` and assert it in e2e, so an undeclared contract change fails the build.

10. **Session keep-alive (§5.11):** add `@nestjs/throttler@^10` — no other new dependency. Add `sid`/`sae` to the
    login payload, implement `SessionTokenService.rotate()` in `auth/` (the only re-signing path; it owns the
    `sae` check and the `exp` clamp) and expose it as `POST /auth/refresh` on `AuthController`, documented in
    Swagger like any other operation. Switch `JWT_EXPIRES_IN` → `JWT_ACCESS_TTL=35m` + `JWT_SESSION_MAX_AGE=8h`,
    add `AUTH_REFRESH_RATE_LIMIT`, and register all three in the boot-time env validation (fail fast if
    `JWT_ACCESS_TTL` is below 10 min, which would leave no head-room for the client's 5 min margin).

**frontend-developer (`front/`)**

*Prerequisite: Node `^22.22.3 || ^24.15.0 || >=26.0.0` (A10) — `npm install` in `front/` fails on older runtimes.*

1. Install `primeng@^22.1`, `@primeuix/themes@^3`, `primeicons@^8`, `@angular/cdk@^22`; configure
   `providePrimeNG({ theme: { preset: Aura } })` in `app.config.ts` and add only `primeicons/primeicons.css` to
   `angular.json > styles`. Do **not** add `@angular/animations` or any `primeng/resources/**` theme CSS — they
   no longer exist. Add `proxy.conf.json` and reference it from `serve.options.proxyConfig` in `angular.json`.
2. Create `core/` (API models from §5.10, `API_BASE_URL`, signal-based `AuthService`, `TokenStorage`, functional
   `authGuard`, functional auth + error interceptors) and wire it in `app.config.ts` via
   `provideHttpClient(withInterceptors([...]))` + `provideAppInitializer(...)`. No `SharedModule`.
3. Routing per §8 incl. lazy `loadComponent`, guard, `returnUrl`, and `**` fallback.
4. Login page (US-1): reactive form, client-side required validation, server error message, redirect on success.
5. Configuration selection page (US-2, US-3): `p-select` fed by `GET /configurations`, loading/empty/error+retry
   states, detail fetch on selection with stale-response protection, detail card, not-found state.
6. 401 handling: clear session and redirect to `/login` (shared interceptor behaviour).
7. Unit tests per §9 (Vitest, not Karma/Jasmine). Until the backend is up, mock against §5 shapes — no contract
   improvisation.

8. **Session keep-alive (§5.11):** add `core/session/session.service.ts` — one `setTimeout` scheduled at
   `expiresIn - 300` s, a `visibilitychange`/`focus` check of the stored token's `exp`, and a single-flight
   `POST /auth/refresh`. **No new dependency.** Wire it into `AuthService` (start after login and after
   `restoreSession()`, stop and clear on logout), add `setToken()` as the single place that writes the token
   signal + `sessionStorage`, and implement the client error rules of §5.11 (one refresh-and-replay on
   `401 UNAUTHENTICATED`, immediate logout on `401 SESSION_EXPIRED`, back off on `429 RATE_LIMITED`). Rotation
   must be invisible in the UI: no re-render, no navigation, no toast.

**teamlead**
1. Decompose §10 into tickets; backend items 1–4 and frontend items 1–3 are the critical path and can run
   fully in parallel against this contract.
2. Fill the `CLAUDE.md` TODOs from this document: package manager = **npm** (`npm@11.19.0` is pinned in
   `front/package.json`), database = **PostgreSQL 16**, run commands (`front`: `npm start` / `npm test` — Vitest;
   `back`: `npm run start:dev` / `npm test` / `npm run test:e2e`). Record the Node prerequisite (A10); a
   committed `.nvmrc` at the repo root is the cheapest enforcement.
3. Get an explicit owner and answer for the PrimeNG licensing question (A11) **before** the frontend work lands —
   the Angular Material fallback is cheap on day one and expensive after two screens exist.
4. Decide whether to schedule the optional hardening items (login rate limiting, httpOnly-cookie migration,
   OpenAPI-generated client) as post-MVP debt tickets — all are recorded in the ADRs.

5. **Get product's answer on A12** (§2.5) before backend item 10 starts: is the session a hard 8 h cap, or a
   sliding window that activity extends? The code differs by one line; the product behaviour differs a lot.
   Also get the accepted consequence in §4 acknowledged in writing: a tab that sleeps past the 35 min mark lands
   on the login page.

**qa-engineer**
1. Test plan against §5 (status codes and `code` values are assertable) and the seed dataset in §6.3.
2. Flow: `POST /admin/initdb` → Swagger login as `user01` → authorize in Swagger → list → detail →
   repeat in the UI; plus negative cases (bad password, expired/absent token, unknown id, empty list).

3. **Session keep-alive (§5.11)**, which needs a longer-running test pass than the rest of the MVP:
   log in, keep the tab open and focused past the 30 min mark and assert the session survives with no visible
   change and no re-login (run once with the real TTL, then with a shortened `JWT_ACCESS_TTL` for the rest);
   assert exactly one `POST /auth/refresh` per interval in the network tab; leave the tab in the background past
   the token's `exp` (or suspend the machine) and assert the documented behaviour — a clean redirect to `/login`,
   with no refresh loop; bring a backgrounded tab back *before* `exp` and assert the `visibilitychange` refresh
   fires; stop the backend across one refresh window and assert recovery on the next attempt; reach the 8 h
   absolute deadline (use a small `JWT_SESSION_MAX_AGE`) and assert a clean redirect to `/login`; log out and
   assert no further `/auth/refresh` calls appear.

**Open items to confirm with product-manager**
- Assumptions A1–A13 (§2.5), in particular A5 (shared seeded password), A9 (`GET /auth/me` added) and the
  session assumptions A12 (what "every 30 minutes" means, and the hard 8 h cap) and A13 (per-tab sessions).
- **The accepted failure mode of a client-driven refresh.** Rotation only happens while the tab is alive: a
  sleeping laptop, or a tab frozen in the background for more than 35 min, ends on the login page (§4). If product
  considers that unacceptable, the levers are a longer `JWT_ACCESS_TTL` (a wider stolen-token window) or the
  server-push channel deferred in §3 — a decision to take before implementation, not after.
- **A second real-time need, if one exists.** The PRD's endpoint list has none, so no push channel is designed.
  If live configuration updates, notifications or presence are actually expected, say so now: it changes the §3
  verdict on a server-push channel, and it is cheaper to add such a channel once than to add it twice.
- A11 / PrimeNG licensing is a business decision, not an architectural one: someone has to confirm the team is
  eligible for the free Community license, or approve the Angular Material fallback.
- That "organization" is intentionally invisible in the MVP UI (it is returned by the API but not displayed);
  showing it in a header costs ~nothing if product wants it.
