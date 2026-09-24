# ADR-004 — API conventions, error envelope, Swagger and how types are shared

- **Status:** Accepted
- **Date:** 2026-09-24
- **Author:** architect
- **Source of truth:** `docs/architecture/initial-architecture.md` — §2.2, §2.4 (OQ #1, #5, #8), §3, §4, §5.1–§5.3, §5.6, §5.10, §9
- **Scope:** every endpoint in §5; `back/src/common/`, `front/src/app/core/api/`

## Context

Frontend and backend start in parallel against a written contract (§5), and the repository has **no root
workspace** (CLAUDE.md: no root `package.json`), so the two sides cannot import a shared package. The contract
therefore has to be self-consistent enough to be mocked, and predictable enough that a growing configuration
schema (PRD OQ #1) does not force breaking changes.

## Decision

1. **REST with URI versioning:** global prefix `api` + `VersioningType.URI`, `defaultVersion: '1'` → base URL
   `/api/v1`. The frontend uses the relative `/api/v1` (`core/api/api.config.ts`) and the Angular dev proxy
   forwards `/api` to `:3000`, so the default dev setup needs no CORS.
2. **JSON conventions:** `application/json; charset=utf-8` both ways, `camelCase` fields (the PRD's `map-lib`
   becomes **`mapLib`** — a naming decision, not a semantic one), UUID ids as 36-char strings, ISO-8601 UTC
   timestamps, `Authorization: Bearer` on everything not marked *public*.
3. **Object envelopes, never bare arrays:** lists return `{ items, total }` (§5.6). The empty case is
   `{ "items": [], "total": 0 }` with `200`, **not** a 404.
4. **One error envelope for every non-2xx response** (§5.3), produced by a global exception filter:
   `{ statusCode, code, message, details, timestamp, path }`, where `code` is a **stable, machine-readable**
   string the UI maps copy off (so copy stays a frontend concern, OQ #8). MVP codes: `VALIDATION_ERROR`,
   `INVALID_CREDENTIALS`, `UNAUTHENTICATED`, `SESSION_EXPIRED`, `NOT_FOUND`, `SEED_DISABLED`, `RATE_LIMITED`,
   `INTERNAL_ERROR`. `INVALID_CREDENTIALS` covers both unknown user and wrong password (no user enumeration),
   and `INTERNAL_ERROR` never leaks internals — the detail goes to the server log (ADR-006).
   Input validation is global: `ValidationPipe({ whitelist: true, forbidNonWhitelisted: true, transform: true })`.
5. **Swagger** via `@nestjs/swagger@7.4.x` at `/api/docs` (`/api/docs-json`) with `addBearerAuth()`, so QA can
   execute the whole flow without the frontend; the Nest CLI Swagger plugin derives most `@ApiProperty` metadata
   from DTO types. Served **only** when `SWAGGER_ENABLED=true` (default `false`), the same gate style as `SEED_ENABLED`.
6. **Payload vs metadata:** list responses carry metadata only; `settings` is returned by the detail endpoint
   (ADR-002). Future fields go inside `settings`; clients ignore unknown `settings` keys.
7. **Type sharing: plain TypeScript interfaces, duplicated verbatim** on both sides — backend DTO classes with
   `@ApiProperty`, frontend `front/src/app/core/api/api.models.ts` (§5.10). They are plain interfaces, so they
   compile unchanged under the backend's TypeScript 5.1 and the frontend's 6.0.
   **Drift control:** this document is normative, plus a committed snapshot of `/api/docs-json` asserted in e2e —
   an undeclared contract change fails the build.
8. **Future work — code generation.** Once the contract stabilises or a third resource appears, generate the
   frontend types from the Swagger JSON with **`openapi-typescript`** (types only, no runtime client, therefore
   no build-order coupling between `front/` and `back/`).

## Alternatives considered

| Alternative | Why rejected (§3, row "Manual TS models on the front") |
|---|---|
| Bare JSON array for lists | Cannot carry pagination/metadata later without a breaking change; the envelope costs one line on the client. |
| Shared workspace package for the models | Requires converting the repo into an npm workspace — CLAUDE.md says there is no root `package.json`. |
| Generated client from Swagger JSON, now | Adds a codegen step before the contract has stabilised. **Deferred, not rejected** — the intended tool is `openapi-typescript` (see decision 8). |
| Decorated runtime classes (`json2typescript` `@JsonObject`/`@JsonProperty`) implementing the §5.10 interfaces | Feasible (the §5.10 shapes compile and deserialise under TypeScript 6.0.3) and rejected **on cost and strategy**: a third copy of every field (back DTO → front interface → `@JsonProperty`), a custom converter per union (`MapLib`, `ApiErrorCode`), and an extra error path mapping `JsonConvert`'s plain `Error` onto the §5.3 envelope — while catching less than it promises, since extra server-side fields are silently dropped, so drift is detected in one direction only. `json2typescript@1.6.1` is additionally single-maintainer, last published March 2025, built against TypeScript 4.3, with no `exports` map and no `peerDependencies`, and rests on legacy `experimentalDecorators` — a longevity risk, not a blocker. **Decisive:** it forecloses codegen, and codegen removes the *cause* of the duplication where runtime classes only detect symptoms. |
| Runtime schema validation at the HTTP boundary (`zod`) | Not a requirement today. **Trigger to revisit:** a third-party JSON producer, or independent front/back release cycles. If that happens, evaluate schema-first `zod` (`z.infer` for the type, `z.strictObject()` to reject unknown keys) — not decorated classes. |

## Consequences

**Gains**

- The contract in §5 is complete enough for frontend and backend to start today; the frontend mocks the documented
  shapes and switches to the real API with no code change.
- A single error envelope with stable `code`s means the UI can always render an error state, and copy changes never
  touch the backend.
- Swagger with bearer auth is a first-class test tool (US-4 success metric).
- URI versioning plus additive `settings` growth means the first breaking change is a deliberate `/api/v2`, not an accident.

**Costs accepted**

- **Type duplication** between `back/src/**/dto` and `front/src/app/core/api/api.models.ts` is real debt; it is
  mitigated (normative document + docs-json snapshot in e2e), not eliminated. Codegen is the designed exit.
- The envelope adds one unwrapping line on every list call.
- Error `code`s are now part of the contract: adding or renaming one is a contract change and goes through the architect.
- Two more env flags to get right per environment (`SWAGGER_ENABLED`, `SEED_ENABLED`); with Swagger off, the API is
  undiscoverable by design.
