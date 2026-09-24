# ADR-005 — Frontend structure: standalone zoneless Angular 22, signals, and PrimeNG 22

- **Status:** Accepted (with one open business question — the PrimeNG license, A11)
- **Date:** 2026-09-24
- **Author:** architect
- **Source of truth:** `docs/architecture/initial-architecture.md` — §1, §2.2, §2.3, §2.5 (A7, A8, A10, A11), §3, §4, §8, §9
- **Scope:** `front/`

## Context

`front/` is **not** greenfield: it was regenerated as an Angular **22.1** CLI app — standalone and **zoneless**
(`app.config.ts`, `app.routes.ts`, `app.ts`, no `app.module.ts`, no `zone.js`), SCSS, `@angular/build` (esbuild),
Vitest on jsdom, TypeScript 6.0. The decisions left open are where auth state, guards and HTTP concerns live,
how much state machinery two screens deserve, and which UI kit to use now that PrimeNG v22 is no longer MIT.

## Decision

1. **Stay standalone and zoneless; do not re-introduce NgModules.** Build on `provideRouter`,
   `provideHttpClient`, functional guards/interceptors and signals.
2. **`core/` + lazy feature routes.** `core/` holds the singletons — `api/api.config.ts` (`API_BASE_URL = '/api/v1'`),
   `api/api.models.ts` (the §5.10 contract mirror), `auth/auth.service.ts`, `auth/token.storage.ts` (the only place
   that touches `sessionStorage`), `auth/auth.guard.ts`, `http/auth.interceptor.ts`, `http/error.interceptor.ts`,
   `session/session.service.ts`, `services/configurations.service.ts`. Features (`login-page`,
   `config-selection-page`) are standalone components loaded per route with `loadComponent` and depend on `core`
   only. `loadChildren` + a feature-level `routes.ts` only once a feature outgrows a single route.
   **No `SharedModule`** — each component imports exactly the PrimeNG components it uses.
3. **Functional guard and interceptors.** `authGuard: CanActivateFn` redirects to `/login?returnUrl=…`;
   `provideHttpClient(withInterceptors([authInterceptor, errorInterceptor]))` — order matters: attach the token
   first, map errors and handle 401 second. No `HttpClientModule`, no `HTTP_INTERCEPTORS` multi-provider.
   Session restore runs in `provideAppInitializer(() => inject(AuthService).restoreSession())`, which validates a
   stored token with `GET /auth/me` and avoids a flash of an authenticated shell.
4. **Signals instead of a state-management library.** `AuthService` holds `user = signal<AuthenticatedUser | null>(null)`
   with `isAuthenticated = computed(...)`. **Zoneless discipline is normative:** every UI update must originate from a
   signal, an `async` pipe or an explicit `markForCheck()` — a bare `subscribe(() => this.x = …)` is not guaranteed
   to repaint. Tests use `await fixture.whenStable()` before asserting rendered output.
5. **PrimeNG 22.1.x** with `providePrimeNG({ theme: { preset: Aura } })` (`@primeuix/themes@^3`), `primeicons@^8`
   and `@angular/cdk@^22` (a PrimeNG peer that is not installed automatically). **Not** `@angular/animations`
   (no longer a peer) and **no** `primeng/resources/**` theme CSS (the files no longer exist) — the only style
   entry in `angular.json` is `primeicons/primeicons.css`. `p-dropdown` is deprecated; the selection screen uses
   **`p-select`**.
6. **Dev cross-origin:** `front/proxy.conf.json` maps `/api` → `http://localhost:3000`, referenced from
   `angular.json` as `serve.options.proxyConfig`; the backend also accepts `CORS_ORIGINS` for non-proxied use.
7. **Toolchain prerequisite (A10):** Node `^22.22.3 || ^24.15.0 || >=26.0.0`. A committed `.nvmrc` at the repo root
   is the cheapest enforcement, and the same Node runs `back/`. Unit tests are Vitest on jsdom via `ng test`
   (`@angular/build:unit-test`), not Karma/Jasmine.

## Alternatives considered

| Alternative | Why rejected (§3) |
|---|---|
| Keep the Angular 15 NgModule app | **Reversal noted:** an earlier revision chose to stay on Angular 15 because the upgrade was risky work with no MVP value. `front/` has since been regenerated on Angular 22, so the upgrade is already paid for and the option is moot. |
| Re-introduce NgModules on top of Angular 22 | Fights the framework's defaults (no `app.module.ts` in the scaffold; `HttpClientModule` and class interceptors are legacy) for zero benefit. |
| NgRx / Component Store / Akita / SignalStore | Two screens and one selected entity: the store would be more code than the feature. A service with a `signal` plus `computed` is sufficient. |
| Stay on PrimeNG 21 (last MIT line) | `primeng@21` peers `@angular/core ^21.0.7`, so "stay on MIT" really means "stay on Angular 21" — not installable on the regenerated scaffold. |
| Angular Material 22 (MIT) | A genuine, license-clean alternative and **the designated fallback** if the Community license does not apply (A11); rejected today only because the UI is already specified in PrimeNG terms and the swap has no architectural consequences. |
| No UI kit, hand-rolled components | Costs more than either option, even for two screens. |

## Consequences

**Gains**

- The frontend tracks the current Angular major: no framework-upgrade debt on day one, and the standalone APIs keep
  `core`/feature boundaries explicit without module ceremony.
- Lazy `loadComponent` per screen keeps the initial bundle to the shell plus `core`.
- Nothing in the API contract or the backend depends on the UI kit, so the Material fallback stays cheap.

**Costs accepted**

- **New team obligations:** Node ≥ 22.22.3 on every dev/CI machine (A10 — `npm install` in `front/` fails on older
  runtimes), zoneless change-detection discipline, and a Vitest/jsdom test setup. Habits from older Angular apps do
  not transfer automatically.
- **PrimeNG 22 is not MIT.** From v22 it ships under the PrimeUI dual model: a free **Community** license (individuals,
  students, non-commercial OSS, and organisations under the revenue/size thresholds, re-confirmed annually) or a
  commercial seat (~$599/developer). **A11 assumes the team qualifies — a human must confirm this before the frontend
  work lands**; it is a legal dependency, not a technical one. The swap to Angular Material is cheap on day one and
  expensive after two screens exist (teamlead item, §10).
- Duplicated contract types in `core/api/api.models.ts` — see ADR-004.
- Single locale, English UI, no SSR (A7, A8).
- The refresh timer owned by `SessionService` is the most likely source of a leak bug (a timer surviving logout);
  it is `providedIn: 'root'`, clears its timeout and listeners on logout/destroy, and that is worth an explicit
  unit test (ADR-001, §8).
