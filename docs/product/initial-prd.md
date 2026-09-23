# PRD: Config Viewer (Login + Configuration Selection) — MVP

> Bilingual document. Full English version first, full Russian version below (not a shortened translation).
> Status: Draft for stakeholder confirmation. Author: product-manager. Date: 2026-09-23. Updated: 2026-09-23 (multi-tenancy vision added; MVP configuration scope clarified to a single field, map-lib).

---

# ENGLISH VERSION

## 1. Problem

Operators/users currently have no controlled, authenticated way to browse and inspect the map configurations stored for the system. There is no single place where a user can log in, see which configurations are available, and view the data of a chosen configuration — which, in this MVP, consists of exactly one field: the map library it uses (map-lib: openlayers or arcgis). Without this, configurations can only be inspected by someone with direct database access, which is not scalable, not auditable, and not usable by non-technical stakeholders or QA.

We need a minimal end-to-end application (Angular frontend + NestJS backend + PostgreSQL) that lets an authenticated user select a configuration from a list and view its data, and that gives the engineering team a working, demoable vertical slice (login → list → detail) plus the developer tooling (Swagger, seed data) needed to build and test it efficiently.

**Product vision (multi-tenancy):** This application is designed from the outset as a multi-tenant product. Users are grouped into organizations, and — as the product vision evolves — it will be the organization that determines which configurations are enabled/visible to its users in the configuration dropdown (see US-2). This MVP does not yet enforce that organization-based filtering (see US-1a, US-2 MVP assumption, Out of Scope, and Open Questions), but the underlying data model should account for the user–organization relationship from the very first user created, because retrofitting multi-tenancy into the data model later is significantly costlier than designing for it now. Organization-based configuration filtering itself is explicitly a future increment, not part of this MVP's functional scope.

## 2. Goal & Success Metrics

**Goal:** Ship a minimal, working end-to-end flow — login, configuration list, configuration detail view — backed by real data in PostgreSQL, with API documentation and seed tooling, so the team has a demoable MVP and a foundation to extend the configuration schema later, on top of a data model that already reflects the product's multi-tenant vision (users belong to organizations).

**Success metrics (MVP):**
- A user can log in with valid credentials and reach the configuration selection screen. Invalid credentials are rejected with a clear error.
- After login, the dropdown is populated with configuration options returned by the backend (not hardcoded on the frontend).
- Selecting a configuration option successfully retrieves and displays that configuration's data from PostgreSQL — in this MVP, the configuration object consists of exactly one field, its map library type (map-lib: openlayers | arcgis).
- All backend endpoints in scope are visible and callable from Swagger UI without needing the frontend.
- Calling `initdb` results in exactly 10 users and 2 configurations (one per map library type: OpenLayers, ArcGIS) available in the database, verifiable via the list/detail endpoints; each created user has an organization association populated.
- QA can execute the full flow (login → list → select → view detail) manually using only Swagger + the UI, without direct DB access.

This is a functional/qualitative MVP milestone rather than a metric tracked over time (e.g. no analytics/usage targets are in scope for this iteration).

## 3. User Stories & Acceptance Criteria

### US-1: Login
**As a** user, **I want to** log in with a username and password, **so that** I can access the configuration selection screen and the system knows who I am.

Acceptance criteria:
- Given valid credentials (username + password) matching a seeded user, when the user submits the login form, then the user is authenticated and navigated to the configuration selection screen.
- Given invalid credentials (wrong password or unknown username), when the user submits the login form, then the user sees a clear error message and remains on the login page.
- Given empty username or password field, when the user attempts to submit, then the form prevents submission and indicates which field(s) are required (client-side validation).
- Given the user is not authenticated, when they try to access the configuration selection screen directly (e.g. by URL), then they are redirected to the login screen.
- Given a user account exists in the system, it is associated with exactly one organization (see US-1a); this association is part of the user's identity from creation, even though it does not yet affect login behavior in this MVP.
- The login form is built with PrimeNG components (given/constraint, not a design decision made here).
- Authentication is validated against the NestJS backend (given/constraint); the exact auth mechanism (session vs token, storage, expiry) is not defined in this document — it is delegated to architect.

Out of scope for this story: password reset, "remember me", registration/sign-up UI, multi-factor auth, role-based permissions.

### US-1a: User belongs to an organization (multi-tenancy foundation)
**As a** product/system, **I need** every user account to belong to exactly one organization, **so that** the platform has the foundational data needed to later let organizations control which configurations are visible to their users.

Acceptance criteria:
- Given any user is created in the system (for MVP, this happens via `initdb`; other user-creation flows are out of scope here), when the user record is stored, then it includes an organization association — the user belongs to exactly one organization.
- Given the MVP does not yet implement organization-based filtering of configurations, when a logged-in user views the configuration dropdown (US-2), then the organization field exists on the user but does not restrict which configurations are shown (see US-2's MVP assumption and Out of Scope).
- This story establishes only the product-level expectation that "a user belongs to an organization" is a foundational, non-optional property of the data model. The exact schema representation (e.g., whether organization is a separate table/entity or a simple field) is an architecture decision — see Handoff.

Out of scope for this story: UI for creating/editing/managing organizations, switching organizations, a user belonging to more than one organization, and organization-based configuration filtering (tracked separately — see Out of Scope and US-2).

### US-2: View list of available configurations after login
**As a** logged-in user, **I want to** see a dropdown of available configurations, **so that** I can choose which configuration I want to inspect.

Acceptance criteria:
- Given the user has successfully logged in, when the configuration selection screen loads, then the frontend requests the list of configurations from the backend and populates the dropdown with the returned options.
- Given the backend returns zero configurations, when the screen loads, then the dropdown is shown empty with a clear "no configurations available" state (not a broken/blank UI).
- Given the backend request for the list fails (network/server error), when the screen loads, then the user sees an error state with the option to retry.
- Given the user's session/authentication is invalid or expired, when the list request is made, then the user is redirected to the login screen (consistent with US-1).
- The dropdown component is a PrimeNG dropdown (given/constraint).
- Each option in the dropdown identifies a configuration in a human-readable way (exact label/fields to be defined by architect based on the data schema — e.g. name, id — not decided here).

**MVP assumption/decision (multi-tenancy simplification):** For this iteration, the configuration list returned by the backend is **not** filtered by the logged-in user's organization. For example, all seeded configurations may be visible to all users, or all seeded users/configurations may simply belong to a single organization — the exact simplification is an implementation decision for architect/backend-developer (see Open Questions #9–#11), not decided here. This is an explicit, documented simplification, not a silent omission: the long-term product vision (see Problem/Vision) is that a user's organization will determine which configurations are enabled/visible for them; that filtering is deferred to a future iteration (see Out of Scope).

### US-3: Select a configuration and view its data
**As a** logged-in user, **I want to** select a configuration from the dropdown and see its data, **so that** I can see which map library it uses.

Acceptance criteria:
- Given the user selects an option from the configuration dropdown, when the selection is made, then the frontend issues a GET request to the backend for that configuration's data.
- Given the backend successfully returns the configuration, when the response arrives, then the UI displays the configuration's data — in this MVP, that data is exactly one field, the map library type (map-lib), with value "openlayers" or "arcgis".
- Given the requested configuration is not found or the request fails, when this happens, then the user sees a clear error/empty state rather than a silent failure or stale data.
- Given the user changes the dropdown selection again, when a new option is chosen, then the previously displayed configuration data is replaced with the newly fetched one (no mixing/stale data).
- In this MVP, the configuration object contains exactly one field — map-lib (values: "openlayers" or "arcgis") — and that is what the UI displays; this is the current full scope of the configuration data, not an abbreviated example. See Open Questions regarding future schema growth (more fields are expected later).

### US-4: API documentation via Swagger
**As a** backend developer or QA engineer, **I want to** browse and call all backend endpoints via Swagger UI, **so that** I can test and validate the API without needing the frontend to be built or running.

Acceptance criteria:
- Given the backend is running, when a developer/QA navigates to the Swagger UI route, then all endpoints in scope for this feature (auth/login, configuration list, configuration detail, initdb) are listed with their request/response shapes.
- Given an endpoint requires authentication, when viewed in Swagger, then this is indicated (exact auth flow in Swagger — e.g. bearer token input — is defined by architect/backend-developer, not here).
- Given a developer executes a request directly from Swagger UI, when the request is valid, then they receive the same response shape/behavior as the real frontend would receive.

### US-5: Seed the database via `initdb`
**As a** developer or QA engineer, **I want to** populate the database with a known, predictable set of test data via a single call, **so that** I can reliably develop and test the login → list → detail flow without manually creating data.

Acceptance criteria:
- Given the `initdb` endpoint is called, when it completes successfully, then the database contains exactly 10 users that can be used to log in via US-1.
- Given the `initdb` endpoint is called, when the 10 users are created, then each created user has an organization association populated (per US-1a); whether all 10 users share a single organization or are distributed across multiple organizations for MVP is an open question (see Open Questions #9–#10), to be resolved by architect/backend-developer and not decided in this document.
- Given the `initdb` endpoint is called, when it completes successfully, then the database contains exactly 2 configurations, one using the OpenLayers map library type and one using the ArcGIS map library type (each configuration's data is the MVP single field, map-lib), each retrievable via US-2/US-3.
- Given the `initdb` endpoint is called, when it completes, then the response confirms what was created (e.g. counts) so a caller can verify success without querying the DB directly.
- The exact behavior on repeated calls (idempotency, duplication, reset behavior) and whether this endpoint requires protection are **not decided in this document** — see Open Questions/Risks. Product requirement here is only: after using `initdb` as documented, the team must be able to reach a known-good state of 10 users + 2 configurations for testing.

## 4. Priority & Rationale

Priority approach: MoSCoW, ordered as the build sequence for this MVP vertical slice.

| Story | Priority | Rationale |
|---|---|---|
| US-5 Seed via `initdb` | Must have | Nothing else can be developed, demoed, or tested without deterministic seed data (users + configurations). Build/enable first. |
| US-4 Swagger docs | Must have | Backend and QA need to validate endpoints independently of frontend progress; unblocks parallel front/back work per the team's process. |
| US-1a User belongs to an organization | Must have | Multi-tenancy is a fundamental, explicitly stated product property, not an optional detail. Retrofitting the user–organization relationship into the data model later is materially costlier than including it from the first user record (created via `initdb`). Must be part of the data model from day one, even though organization-based filtering itself is deferred (see Out of Scope). |
| US-1 Login | Must have | Entry point of the flow; gates access to everything else; required to demonstrate authenticated access. |
| US-2 Configuration list | Must have | Core MVP value — without it, there is nothing to select from backend-driven data. |
| US-3 Select & view configuration | Must have | This is the actual product value (viewing configuration data — in this MVP, the map library type, map-lib) — the reason the feature exists. |

All stories are Must-have for this MVP; there is no meaningful partial slice that delivers value without all of them (login gates the list, the list gates selection, Swagger/seed data are enabling/testing infrastructure required to build and verify the rest safely, and user–organization association is a foundational data-model property that must not be an afterthought). No Should/Could/Won't items are introduced in this iteration — anything beyond these stories is explicitly Out of Scope below.

## 5. Out of Scope

- Password reset / forgot password flow.
- User self-registration or sign-up UI.
- Role-based access control or permissions (e.g. who can see which configurations).
- Editing, creating, or deleting configurations from the UI (this MVP is read-only: view/select only).
- Editing or managing users from the UI.
- Any UI for editing configuration fields beyond displaying the MVP configuration data (currently a single field, map-lib) — no config editor.
- Rendering an actual map (OpenLayers/ArcGIS) based on the selected configuration — this MVP only displays the configuration data, it does not instantiate a map.
- Multi-factor authentication, SSO, or third-party auth providers.
- Pagination, search, or filtering of the configuration list (assumed small list for MVP).
- Auditing/logging of who viewed which configuration.
- Any analytics or usage-tracking dashboards.
- Protecting/securing the `initdb` endpoint (decision explicitly deferred — see Open Questions).
- Internationalization/localization of the UI.
- Automated CI/CD pipeline setup (separate concern from this feature's product scope).
- **Organization-scoped configuration visibility/filtering** (i.e., restricting which configurations a given organization's users can see, and any UI/logic to manage those restrictions). This is the stated long-term product vision (see Problem/Vision) but is explicitly deferred to a future iteration. For this MVP, configurations may be visible to all users regardless of organization (the exact simplification — e.g. all users in one organization, or no filtering applied at all — is an implementation decision for architect/backend-developer; see Open Questions #9–#11).
- Organization management UI (creating, editing, or listing organizations) and any concept of a user belonging to more than one organization.
- Adding any configuration fields beyond the single MVP field, `map-lib` (e.g. future fields covering other configuration aspects) — explicitly future work, not part of this MVP (see Open Questions #1).

## 6. Open Questions & Risks

1. **Configuration schema will grow.** The MVP configuration schema is exactly one field: `map-lib` (values: `openlayers` | `arcgis`). This is the current full scope of the config object, not a minimal example. Future iterations will add more fields to the configuration; the exact future shape is not yet defined. Risk: frontend detail view and backend response shape will need rework as fields are added. Recommendation: architect should design the API contract/schema in a way that is reasonably extensible, but the actual future fields are out of scope now.
2. **Should `initdb` be protected?** Should this endpoint require authentication, a special role, an environment flag (e.g. only enabled in dev/test), or be entirely open? This is a security/operational question for architect to decide and document — not decided here. Risk if left fully open in production: uncontrolled reseeding/data exposure.
3. **Is `initdb` idempotent?** If called twice, does it duplicate the 10 users / 2 configurations, error out, reset/wipe existing data first, or upsert? This materially affects QA's ability to reuse the endpoint reliably during testing. Needs a defined behavior from architect/backend-developer.
4. **Auth mechanism is undefined at product level.** This document intentionally does not decide session vs token-based auth, where/how the credential is stored on the frontend, or expiry/refresh behavior. This is delegated to architect.
5. **Dropdown option labeling.** What identifies a configuration to the user in the dropdown (name, id, description)? Depends on the data schema, to be defined by architect based on backend data model.
6. **Password storage/hashing for seeded users.** Not decided here — security concern for architect/backend-developer (e.g. seeded users must not use plaintext passwords in the database).
7. **Environment/config for the PostgreSQL connection** (connection string, credentials, local vs hosted instance) is a technical setup concern for architect/backend-developer, not decided here.
8. **Error/empty states content and copy** (exact wording for "no configurations available", error messages) — to be refined during frontend implementation; only the behavior is specified here.
9. **How many organizations exist in MVP?** Will the MVP represent a single organization overall, or multiple organizations from the start (even if filtering isn't enforced yet)? This affects how the "MVP simplification" in US-2 should be interpreted and how seed data should look. Decision needed from architect/backend-developer.
10. **How is user→organization assignment determined for `initdb`?** Should all 10 seeded users belong to one single organization, or should they be distributed across several organizations to better exercise the future multi-tenant model? Not decided in this document — needs a decision from architect/backend-developer, informed by Open Question #9.
11. **Should the database already model `organization` as a separate entity/table at MVP stage?** Even though organization-based configuration filtering is deferred to a future increment, should the schema introduce a dedicated `organizations` table now (with users referencing it by id), or is a simpler field on the user record sufficient for MVP? This is explicitly an architecture decision, raised here so it is not lost when the schema is designed — see Handoff.

## 7. Handoff — What's Needed Next

**To architect:**
- Design the API contract between front and back for: authentication/login endpoint(s), the configuration list endpoint, the configuration detail-by-id/name endpoint, and the `initdb` endpoint (request/response shapes, status codes, error formats).
- Design the PostgreSQL data schema for the `users` and `configurations` tables. For MVP, the configuration object has exactly one field, `map-lib` (values: `openlayers` | `arcgis`) — per US-3/US-5 — including how that field is represented; the schema should reasonably anticipate future fields being added to the configuration (see Open Questions #1), without designing those future fields now.
- Incorporate `organization` as part of the user data model (and consider whether a dedicated `organizations` table/entity is warranted) from the start, even though organization-based configuration filtering is not implemented in this MVP — see Problem/Vision, US-1a, and Open Questions #9–#11. The goal is to avoid costly retrofitting later, per the product's stated multi-tenancy vision.
- Design the authentication/session/token storage mechanism (how the frontend proves it's logged in on subsequent requests) — currently unspecified at product level.
- Decide and document: is `initdb` protected, and is it idempotent (see Open Questions #2, #3)? How many organizations does seed data represent, and how are the 10 users distributed across them (see Open Questions #9, #10)?
- Decide how Swagger reflects authenticated endpoints (US-4).
- Produce these artifacts in `docs/architecture/` (API contract, data schema, relevant ADRs) before frontend-developer/backend-developer start implementation.

**To teamlead (after architecture is delivered):**
- Decompose this PRD + the architecture artifacts into concrete tasks for frontend-developer (`front/`, Angular + PrimeNG: login page, config-selection page with dropdown, config detail view) and backend-developer (`back/`, NestJS: auth endpoint(s), PostgreSQL connection, configuration list/detail endpoints, Swagger setup, `initdb` endpoint including organization assignment for seeded users), to be run in parallel against the shared API contract.
- Include qa-engineer's test-plan needs (manual flow: login → list → select → detail, using Swagger + UI) in planning.

---

# РУССКАЯ ВЕРСИЯ

## 1. Проблема

У операторов/пользователей сейчас нет контролируемого, аутентифицированного способа просматривать и изучать конфигурации карт, хранящиеся в системе. Нет единого места, где пользователь мог бы залогиниться, увидеть, какие конфигурации доступны, и посмотреть данные выбранной конфигурации — которые в этом MVP состоят ровно из одного поля: используемая карт-библиотека (map-lib: openlayers или arcgis). Без этого конфигурации можно посмотреть только имея прямой доступ к базе данных, что не масштабируется, не поддаётся аудиту и недоступно нетехническим стейкхолдерам или QA.

Нужно минимальное сквозное (end-to-end) приложение (Angular-фронтенд + NestJS-бэкенд + PostgreSQL), которое позволяет аутентифицированному пользователю выбрать конфигурацию из списка и посмотреть её данные, а также даёт команде разработки рабочий, демонстрируемый вертикальный срез (логин → список → детали) плюс инструментарий для разработчиков (Swagger, тестовые данные), необходимый для эффективной разработки и тестирования.

**Продуктовое видение (мульти-тенантность):** приложение изначально проектируется как мульти-тенантный продукт. Пользователи объединены в организации (organizations), и по мере развития продукта именно организация будет определять, какие конфигурации разрешены/видны её пользователям в dropdown выбора конфигурации (см. US-2). Этот MVP пока не реализует такую фильтрацию по организации (см. US-1a, MVP-допущение в US-2, "Вне рамок" и "Открытые вопросы"), но базовая модель данных должна учитывать связь "пользователь–организация" начиная с самого первого созданного пользователя, поскольку встраивать мульти-тенантность в модель данных позже значительно дороже, чем заложить её с самого начала. Сама фильтрация конфигураций по организации — это явно будущий инкремент, а не часть функционального объёма этого MVP.

## 2. Цель и метрика успеха

**Цель:** Реализовать минимальный, рабочий сквозной сценарий — логин, список конфигураций, просмотр деталей конфигурации — на реальных данных в PostgreSQL, с документацией API и инструментом наполнения тестовыми данными, чтобы у команды был демонстрируемый MVP и основа для дальнейшего расширения схемы конфигурации, поверх модели данных, которая уже отражает мульти-тенантное видение продукта (пользователи принадлежат организациям).

**Метрики успеха (MVP):**
- Пользователь может залогиниться с валидными учётными данными и попасть на экран выбора конфигурации. Невалидные учётные данные отклоняются с понятной ошибкой.
- После логина dropdown заполняется опциями конфигураций, полученными от бэкенда (а не захардкоженными на фронте).
- Выбор опции конфигурации успешно получает и отображает данные этой конфигурации из PostgreSQL — в этом MVP объект конфигурации состоит ровно из одного поля, тип карт-библиотеки (map-lib: openlayers | arcgis).
- Все эндпоинты бэкенда, входящие в объём задачи, видны и вызываемы из Swagger UI без необходимости фронтенда.
- Вызов `initdb` приводит ровно к 10 пользователям и 2 конфигурациям (по одной на каждый тип карт-библиотеки: OpenLayers, ArcGIS) в базе данных, что можно проверить через эндпоинты списка/деталей; у каждого созданного пользователя заполнена принадлежность к организации.
- QA может выполнить весь сценарий (логин → список → выбор → просмотр деталей) вручную, используя только Swagger + UI, без прямого доступа к БД.

Это функциональная/качественная веха MVP, а не метрика, отслеживаемая во времени (например, аналитика/цели по использованию не входят в объём этой итерации).

## 3. User Stories и критерии приёмки

### US-1: Логин
**Как** пользователь, **я хочу** залогиниться с помощью username и password, **чтобы** получить доступ к экрану выбора конфигурации, и чтобы система знала, кто я.

Критерии приёмки:
- Если введены валидные учётные данные (username + password), соответствующие тестовому (seeded) пользователю, и пользователь отправляет форму логина, то пользователь аутентифицируется и переходит на экран выбора конфигурации.
- Если введены невалидные учётные данные (неверный пароль или неизвестный username), и пользователь отправляет форму логина, то пользователь видит понятное сообщение об ошибке и остаётся на странице логина.
- Если поле username или password пустое, и пользователь пытается отправить форму, то отправка блокируется и указывается, какое поле(я) обязательно (клиентская валидация).
- Если пользователь не аутентифицирован и пытается напрямую попасть на экран выбора конфигурации (например, по URL), то он перенаправляется на экран логина.
- Если учётная запись пользователя существует в системе, она связана ровно с одной организацией (см. US-1a); эта связь — часть идентичности пользователя с момента создания, даже если она пока не влияет на поведение логина в этом MVP.
- Форма логина строится на компонентах PrimeNG (дано/ограничение, не решение, принятое в этом документе).
- Аутентификация проверяется против бэкенда на NestJS (дано/ограничение); точный механизм аутентификации (сессия vs токен, хранение, срок действия) в этом документе не определяется — передаётся architect.

Вне рамок этой истории: восстановление пароля, "запомнить меня", UI регистрации/создания аккаунта, многофакторная аутентификация, ролевые права доступа.

### US-1a: Пользователь принадлежит организации (основа мульти-тенантности)
**Как** продукт/система, **я нуждаюсь в том**, чтобы каждая учётная запись пользователя принадлежала ровно одной организации, **чтобы** платформа имела базовые данные, необходимые в будущем для того, чтобы организации могли управлять тем, какие конфигурации видны их пользователям.

Критерии приёмки:
- Если в системе создаётся любой пользователь (для MVP это происходит через `initdb`; другие сценарии создания пользователей вне рамок этого документа), то при сохранении записи пользователя она включает связь с организацией — пользователь принадлежит ровно одной организации.
- Если MVP пока не реализует фильтрацию конфигураций по организации, то при просмотре залогиненным пользователем dropdown конфигураций (US-2) поле организации у пользователя существует, но не ограничивает, какие конфигурации показываются (см. MVP-допущение в US-2 и "Вне рамок").
- Эта история фиксирует только продуктовое ожидание: "пользователь принадлежит организации" — это фундаментальное, не опциональное свойство модели данных. Точное представление в схеме (например, отдельная таблица/сущность organization или простое поле) — это решение архитектора, см. "Передача дальше".

Вне рамок этой истории: UI для создания/редактирования/управления организациями, переключение между организациями, принадлежность пользователя более чем одной организации, а также фильтрация конфигураций по организации (отслеживается отдельно — см. "Вне рамок" и US-2).

### US-2: Просмотр списка доступных конфигураций после логина
**Как** залогиненный пользователь, **я хочу** видеть dropdown доступных конфигураций, **чтобы** выбрать, какую конфигурацию я хочу изучить.

Критерии приёмки:
- Если пользователь успешно залогинился и загружается экран выбора конфигурации, то фронтенд запрашивает список конфигураций у бэкенда и заполняет dropdown полученными опциями.
- Если бэкенд возвращает ноль конфигураций, при загрузке экрана dropdown отображается пустым с понятным состоянием "нет доступных конфигураций" (а не сломанным/пустым UI без объяснения).
- Если запрос к бэкенду за списком завершается ошибкой (сетевая/серверная ошибка), при загрузке экрана пользователь видит состояние ошибки с возможностью повторить попытку.
- Если сессия/аутентификация пользователя невалидна или истекла на момент запроса списка, то пользователь перенаправляется на экран логина (согласовано с US-1).
- Компонент dropdown — это PrimeNG dropdown (дано/ограничение).
- Каждая опция в dropdown идентифицирует конфигурацию понятным человеку образом (точный label/поля определяет architect на основе схемы данных — например name, id — не решается в этом документе).

**MVP-допущение/решение (упрощение мульти-тенантности):** В этой итерации список конфигураций, возвращаемый бэкендом, **не** фильтруется по организации залогиненного пользователя. Например, все тестовые конфигурации могут быть видны всем пользователям, либо все тестовые пользователи/конфигурации могут просто принадлежать одной организации — точное упрощение является имплементационным решением для architect/backend-developer (см. открытые вопросы №9–№11), не решается здесь. Это явное, задокументированное упрощение, а не молчаливое упущение: долгосрочное видение продукта (см. "Проблема/Видение") состоит в том, что организация пользователя будет определять, какие конфигурации ему разрешены/видны; эта фильтрация отложена на будущую итерацию (см. "Вне рамок").

### US-3: Выбор конфигурации и просмотр её данных
**Как** залогиненный пользователь, **я хочу** выбрать конфигурацию из dropdown и увидеть её данные, **чтобы** увидеть, какую карт-библиотеку она использует.

Критерии приёмки:
- Если пользователь выбирает опцию в dropdown конфигураций, то фронтенд отправляет GET-запрос к бэкенду за данными этой конфигурации.
- Если бэкенд успешно возвращает конфигурацию, то UI отображает данные конфигурации — в этом MVP это ровно одно поле, тип карт-библиотеки (map-lib) со значением "openlayers" или "arcgis".
- Если запрошенная конфигурация не найдена или запрос завершается ошибкой, то пользователь видит понятное состояние ошибки/пустое состояние, а не тихий сбой или устаревшие данные.
- Если пользователь снова меняет выбор в dropdown, то ранее отображённые данные конфигурации заменяются на вновь полученные (без смешивания/устаревших данных).
- В этом MVP объект конфигурации содержит ровно одно поле — map-lib (значения: "openlayers" или "arcgis"), — и именно его отображает UI; это текущий полный объём данных конфигурации, а не сокращённый пример. См. раздел "Открытые вопросы" по поводу будущего расширения схемы (в будущем полей станет больше).

### US-4: Документация API через Swagger
**Как** backend-разработчик или QA-инженер, **я хочу** просматривать и вызывать все эндпоинты бэкенда через Swagger UI, **чтобы** тестировать и проверять API без необходимости собранного или запущенного фронтенда.

Критерии приёмки:
- Если бэкенд запущен, и разработчик/QA переходит на маршрут Swagger UI, то все эндпоинты, входящие в объём этой фичи (логин/аутентификация, список конфигураций, детали конфигурации, initdb), перечислены с описанием форм запроса/ответа.
- Если эндпоинт требует аутентификации, при просмотре в Swagger это указано (точный поток аутентификации в Swagger — например, ввод bearer-токена — определяет architect/backend-developer, не этот документ).
- Если разработчик выполняет запрос напрямую из Swagger UI, и запрос валиден, то он получает такую же форму/поведение ответа, как получил бы реальный фронтенд.

### US-5: Наполнение БД тестовыми данными через `initdb`
**Как** разработчик или QA-инженер, **я хочу** наполнить базу данных известным, предсказуемым набором тестовых данных одним вызовом, **чтобы** надёжно разрабатывать и тестировать сценарий логин → список → детали без ручного создания данных.

Критерии приёмки:
- Если эндпоинт `initdb` вызван и успешно завершился, то в базе данных находится ровно 10 пользователей, которых можно использовать для входа через US-1.
- Если эндпоинт `initdb` вызван и создаёт 10 пользователей, то у каждого созданного пользователя заполнена связь с организацией (согласно US-1a); принадлежат ли все 10 пользователей одной организации или распределены между несколькими организациями для MVP — это открытый вопрос (см. открытые вопросы №9–№10), который должен решить architect/backend-developer и который не решается в этом документе.
- Если эндпоинт `initdb` вызван и успешно завершился, то в базе данных находятся ровно 2 конфигурации: одна с типом карт-библиотеки OpenLayers и одна с типом ArcGIS (данные каждой конфигурации — это MVP-поле map-lib), каждая из которых доступна через US-2/US-3.
- Если эндпоинт `initdb` вызван и завершился, то ответ подтверждает, что было создано (например, счётчики), чтобы вызывающий мог проверить успех без прямого запроса к БД.
- Точное поведение при повторных вызовах (идемпотентность, дублирование, поведение сброса) и необходимость защиты этого эндпоинта **не решаются в этом документе** — см. "Открытые вопросы/риски". Продуктовое требование здесь только одно: после использования `initdb` по документации команда должна получить заведомо рабочее состояние — 10 пользователей + 2 конфигурации — для тестирования.

## 4. Приоритет и обоснование

Подход к приоритизации: MoSCoW, с порядком, отражающим последовательность сборки этого MVP-среза.

| История | Приоритет | Обоснование |
|---|---|---|
| US-5 Наполнение через `initdb` | Must have | Без детерминированных тестовых данных (пользователи + конфигурации) ничего нельзя разработать, продемонстрировать или протестировать. Реализовать первым. |
| US-4 Документация Swagger | Must have | Backend и QA должны иметь возможность проверять эндпоинты независимо от готовности фронтенда; разблокирует параллельную работу front/back согласно процессу команды. |
| US-1a Пользователь принадлежит организации | Must have | Мульти-тенантность — фундаментальное, явно заявленное свойство продукта, а не опциональная деталь. Встраивать связь "пользователь–организация" в модель данных позже существенно дороже, чем заложить её с первой же записи пользователя (создаваемой через `initdb`). Должна быть частью модели данных с первого дня, даже если сама фильтрация по организации отложена (см. "Вне рамок"). |
| US-1 Логин | Must have | Точка входа в сценарий; закрывает доступ ко всему остальному; необходима для демонстрации аутентифицированного доступа. |
| US-2 Список конфигураций | Must have | Ключевая ценность MVP — без него нечего выбирать из данных, отдаваемых бэкендом. |
| US-3 Выбор и просмотр конфигурации | Must have | Это и есть реальная продуктовая ценность (просмотр данных конфигурации — в этом MVP тип карт-библиотеки, map-lib) — причина, по которой фича существует. |

Все истории имеют приоритет Must-have для этого MVP; нет осмысленного частичного среза, который приносил бы ценность без всех них сразу (логин закрывает доступ к списку, список — к выбору, Swagger/тестовые данные — это инфраструктура, необходимая для безопасной разработки и проверки остального, а связь "пользователь–организация" — фундаментальное свойство модели данных, которое не должно быть добавлено задним числом). В этой итерации не вводятся элементы Should/Could/Won't; всё, что выходит за рамки этих историй, явно указано ниже в разделе "Вне рамок".

## 5. Вне рамок (Out of scope)

- Восстановление/сброс пароля.
- Самостоятельная регистрация пользователей / UI создания аккаунта.
- Ролевой доступ или права (например, кто какие конфигурации может видеть).
- Редактирование, создание или удаление конфигураций через UI (этот MVP — только для чтения: просмотр/выбор).
- Редактирование или управление пользователями через UI.
- Любой UI для редактирования полей конфигурации сверх отображения MVP-данных конфигурации (сейчас — единственное поле map-lib) — без редактора конфигурации.
- Рендеринг реальной карты (OpenLayers/ArcGIS) на основе выбранной конфигурации — этот MVP только отображает данные конфигурации, не инициализирует карту.
- Многофакторная аутентификация, SSO или сторонние провайдеры авторизации.
- Пагинация, поиск или фильтрация списка конфигураций (для MVP предполагается небольшой список).
- Аудит/логирование того, кто какую конфигурацию просматривал.
- Любая аналитика или дашборды отслеживания использования.
- Защита/безопасность эндпоинта `initdb` (решение явно откладывается — см. "Открытые вопросы").
- Интернационализация/локализация UI.
- Настройка автоматизированного CI/CD пайплайна (отдельная тема, не входящая в продуктовый объём этой фичи).
- **Фильтрация/привязка конфигураций к конкретной организации** (organization-scoped config visibility) — то есть ограничение того, какие конфигурации видны пользователям определённой организации, и любой UI/логика для управления такими ограничениями. Это заявленное долгосрочное видение продукта (см. "Проблема/Видение"), но оно явно отложено на будущую итерацию. Для этого MVP конфигурации могут быть видны всем пользователям независимо от организации (точное упрощение — например, все пользователи в одной организации, или фильтрация вовсе не применяется — это имплементационное решение для architect/backend-developer; см. открытые вопросы №9–№11).
- UI управления организациями (создание, редактирование, просмотр списка организаций) и любая концепция принадлежности пользователя более чем одной организации.
- Добавление любых полей конфигурации сверх единственного MVP-поля `map-lib` (например, будущие поля для других аспектов конфигурации) — явно будущая работа, не часть этого MVP (см. открытый вопрос №1).

## 6. Открытые вопросы и риски

1. **Схема конфигурации будет расти.** MVP-схема конфигурации — это ровно одно поле: `map-lib` (значения: `openlayers` | `arcgis`). Это текущий полный объём объекта конфигурации, а не минимальный пример. В будущих итерациях в конфигурацию будут добавлены новые поля; точная будущая форма пока не определена. Риск: экран деталей на фронте и форма ответа бэкенда потребуют переработки по мере добавления полей. Рекомендация: architect должен спроектировать API-контракт/схему разумно расширяемой, но конкретные будущие поля вне рамок сейчас.
2. **Нужно ли защищать `initdb`?** Должен ли этот эндпоинт требовать аутентификации, специальной роли, флага окружения (например, включён только в dev/test), или быть полностью открытым? Это вопрос безопасности/эксплуатации, который должен решить и задокументировать architect — не решается здесь. Риск, если оставить полностью открытым в продакшене: неконтролируемое повторное наполнение/утечка данных.
3. **Идемпотентен ли `initdb`?** При повторном вызове — дублирует ли он 10 пользователей / 2 конфигурации, завершается ли ошибкой, сначала сбрасывает/очищает существующие данные, или делает upsert? Это существенно влияет на возможность QA надёжно повторно использовать эндпоинт при тестировании. Требуется определённое поведение от architect/backend-developer.
4. **Механизм аутентификации не определён на продуктовом уровне.** Этот документ намеренно не решает: сессия или токен, где и как хранится учётные данные на фронте, поведение истечения срока/обновления. Передаётся architect.
5. **Подпись опций в dropdown.** Что идентифицирует конфигурацию для пользователя в dropdown (name, id, описание)? Зависит от схемы данных, определяется architect на основе модели данных бэкенда.
6. **Хранение/хеширование паролей для тестовых (seeded) пользователей.** Не решается здесь — вопрос безопасности для architect/backend-developer (например, тестовые пользователи не должны иметь пароли в открытом виде в БД).
7. **Окружение/конфигурация подключения к PostgreSQL** (строка подключения, учётные данные, локальный или хостируемый инстанс) — техническая задача настройки для architect/backend-developer, не решается здесь.
8. **Содержание и формулировки состояний ошибок/пустых состояний** (точный текст "нет доступных конфигураций", сообщения об ошибках) — уточняется в ходе реализации фронтенда; здесь специфицировано только поведение.
9. **Сколько организаций существует в MVP?** Будет ли MVP представлять одну единственную организацию в целом, или несколько организаций с самого начала (даже если фильтрация по ним пока не применяется)? Это влияет на то, как трактовать "MVP-упрощение" в US-2, и как должны выглядеть тестовые данные. Решение нужно от architect/backend-developer.
10. **Как определяется принадлежность пользователя к организации при `initdb`?** Должны ли все 10 тестовых пользователей принадлежать одной организации, или их стоит распределить между несколькими организациями, чтобы лучше отработать будущую мульти-тенантную модель? Не решается в этом документе — нужно решение от architect/backend-developer, с учётом открытого вопроса №9.
11. **Нужно ли уже на этапе MVP закладывать `organization` в БД как отдельную сущность/таблицу?** Даже если фильтрация конфигураций по организации откладывается на будущий инкремент, стоит ли уже сейчас вводить в схему отдельную таблицу `organizations` (на которую пользователи ссылаются по id), или для MVP достаточно простого поля в записи пользователя? Это явно решение архитектора, вопрос ставится здесь явно, чтобы не потерять его при проектировании схемы — см. "Передача дальше".

## 7. Передача дальше

**Для architect:**
- Спроектировать API-контракт между фронтом и бэком для: эндпоинта(ов) аутентификации/логина, эндпоинта списка конфигураций, эндпоинта получения конфигурации по id/имени, и эндпоинта `initdb` (формы запроса/ответа, коды статусов, форматы ошибок).
- Спроектировать схему данных PostgreSQL для таблиц `users` и `configurations`. Для MVP объект конфигурации содержит ровно одно поле — `map-lib` (значения: `openlayers` | `arcgis`) — согласно US-3/US-5, включая то, как это поле представлено; схема должна разумно закладывать возможность добавления новых полей конфигурации в будущем (см. открытый вопрос №1), не проектируя эти будущие поля сейчас.
- Заложить `organization` как часть модели данных пользователя (и рассмотреть, оправдана ли отдельная таблица/сущность `organizations`) с самого начала, даже если фильтрация конфигураций по организации не реализуется в этом MVP — см. "Проблема/Видение", US-1a и открытые вопросы №9–№11. Цель — избежать дорогостоящей переделки позже, в соответствии с заявленным видением мульти-тенантности продукта.
- Спроектировать механизм хранения аутентификации/сессии/токена (как фронтенд подтверждает, что пользователь залогинен, в последующих запросах) — сейчас не определён на продуктовом уровне.
- Принять решение и задокументировать: защищён ли `initdb`, и идемпотентен ли он (см. открытые вопросы №2, №3)? Сколько организаций представляют тестовые данные, и как 10 пользователей распределены между ними (см. открытые вопросы №9, №10)?
- Решить, как Swagger отражает эндпоинты, требующие аутентификации (US-4).
- Подготовить эти артефакты в `docs/architecture/` (API-контракт, схема данных, релевантные ADR) до начала работы frontend-developer/backend-developer.

**Для teamlead (после получения артефактов от architect):**
- Декомпозировать этот PRD + артефакты архитектуры на конкретные задачи для frontend-developer (`front/`, Angular + PrimeNG: страница логина, страница выбора конфигурации с dropdown, экран деталей конфигурации) и backend-developer (`back/`, NestJS: эндпоинт(ы) аутентификации, подключение PostgreSQL, эндпоинты списка/деталей конфигурации, настройка Swagger, эндпоинт `initdb`, включая назначение организации тестовым пользователям), для параллельного выполнения по общему API-контракту.
- Учесть в планировании потребности qa-engineer в тест-плане (ручной сценарий: логин → список → выбор → детали, с использованием Swagger + UI).
