# Обзор архитектуры — Config Viewer MVP

> Статус: Предложено (готово к декомпозиции тимлидом). Автор: architect. Дата: 2026-09-23
> (в тот же день пересмотрено: `front/` пересоздан на Angular 22 — см. §2.3 и §8).
> Исходные требования: `docs/product/initial-prd.md` (PRD: Config Viewer — MVP).
> Оригинал (английская версия): `docs/architecture/initial-architecture.md`. Данный файл — полный перевод, поддерживается в синхронизации.
> Связанные ADR: `docs/architecture/adr/initial-architecture/` (будут добавлены по запросу).

**Этот документ — единственный источник правды по API-контракту front↔back.** Изменения в него вносит только архитектор.

---

## 1. Контекст

PRD требует минимального, но полного вертикального среза: пользователь логинится, видит dropdown конфигураций,
отдаваемых бэкендом, выбирает одну и видит её данные — которые в этом MVP представляют собой ровно одно поле,
тип карт-библиотеки (`map-lib`: `openlayers` | `arcgis`). Сопутствующие требования: Swagger UI для всех
эндпоинтов и seed-эндпоинт `initdb`, создающий ровно 10 пользователей и 2 конфигурации, где каждый
пользователь связан с организацией (основа мульти-тенантности, US-1a).

Репозиторий уже существует и **не** является greenfield с точки зрения инструментария:

| Область | Что уже есть | Следствие |
|---|---|---|
| `front/` | Angular **22.1**, CLI-приложение, **standalone + zoneless** (`app.config.ts`, `app.routes.ts`, `app.ts`; `app.module.ts` нет), SCSS, Vitest + jsdom, TypeScript 6.0 | Строим на standalone-API (`provideRouter`, `provideHttpClient`, функциональные guard'ы/интерцепторы, signals); NgModules **не** возвращаем. PrimeNG должен быть из линейки v22. |
| `back/` | NestJS **10**, платформа Express, TypeScript 5.1, Jest + Supertest, только скаффолд `AppController`/`AppService` | Добавляем сверху TypeORM (PostgreSQL), Swagger, JWT/Passport; смена фреймворка не нужна. |
| Монорепозиторий | Два независимых `package.json`, нет корневого workspace, один `.git` в корне, `docs/` для артефактов агентов | Общего инструментария сборки нет. Разделение типов между front и back нужно решать без workspace-пакета (см. §5.10). |
| БД | Пока ничего | Нужно ввести PostgreSQL, включая конфигурацию подключения (ОВ №7 в PRD). PRD называет MongoDB — сознательное отклонение, см. §3. |

Таким образом, этот документ не выдумывает стек с нуля: он принимает решения по *недостающим* частям (хранилище,
аутентификация, форма API, границы модулей, наполнение данными) и ограничивает их тем, что поддерживают существующие скаффолды.

Архитектурно значимые задачи, которые нужно решить:

1. **Механизм аутентификации и хранение учётных данных** — не определены на продуктовом уровне (ОВ №4 в PRD).
2. **Модель данных, включая мульти-тенантность** — организации как таблица или как обычное поле (ОВ №9–№11 в PRD).
3. **Семантика `initdb`** — защита и идемпотентность (ОВ №2, №3 в PRD).
4. **Форма API-контракта** — достаточно расширяемая для растущей схемы конфигурации (ОВ №1, №5 в PRD).
5. **Структура фронтенда** — где живут состояние аутентификации, guard'ы и HTTP-аспекты в standalone-приложении на Angular 22 с zoneless-режимом.
6. **Выбор UI-набора при новом лицензировании PrimeNG** — начиная с v22 PrimeNG больше не MIT (§2.3, §3, A11).

---

## 2. Решение

### 2.1 Высокоуровневая форма

Классическая двухзвенная схема SPA + REST API с одной базой PostgreSQL. Нет шлюза, нет BFF, нет шины сообщений,
нет микросервисов — у MVP один ограниченный контекст и горстка эндпоинтов.

```
┌──────────────────────────┐        HTTPS/JSON              ┌──────────────────────────┐
│  front/  Angular 22 SPA  │  ───────────────────────────▶  │  back/  NestJS 10 (REST) │
│  PrimeNG 22 UI           │   Authorization: Bearer <JWT>  │  Swagger UI на /api/docs │
│  AuthService + guard     │  ◀───────────────────────────  │  Глобальный ValidationPipe│
│  HTTP-интерцепторы       │       JSON + конверт ошибки    │  Глобальный exception-фильтр│
└──────────────────────────┘                                └────────────┬─────────────┘
        dev: ng serve :4200                                               │ TypeORM 0.3
        прокси /api → :3000                                               ▼
                                                             ┌──────────────────────────┐
                                                             │ PostgreSQL 16            │
                                                             │ organizations / users /  │
                                                             │ configurations           │
                                                             └──────────────────────────┘
```

**Поток запросов (happy path):**
`Форма логина → POST /api/v1/auth/login → JWT сохраняется фронтом → GET /api/v1/configurations (Bearer)
→ dropdown наполнен → пользователь выбирает → GET /api/v1/configurations/{id} (Bearer) → детали отрисованы.`

### 2.2 Технологические решения (сводка)

| Аспект | Решение | ADR |
|---|---|---|
| Аутентификация | Stateless JWT (HS256), `Authorization: Bearer`, выдаётся `POST /auth/login`, валидируется стратегией Passport JWT + глобальным `JwtAuthGuard` с отказом через `@Public()` | [ADR-001](adr/001-jwt-bearer-authentication.ru.md) |
| Хранение токена (фронт) | In-memory `signal` в `AuthService` + зеркало в `sessionStorage` для переживания перезагрузки; refresh-токена в MVP нет | ADR-001 |
| Хранилище | PostgreSQL 16 через `@nestjs/typeorm` 10 + `typeorm` 0.3 + драйвер `pg`, сущности объявляются декораторами | [ADR-002](adr/002-postgresql-data-model-and-multitenancy.ru.md) |
| Мульти-тенантность | Отдельная таблица `organizations`; `users.organizationId` — обязательный внешний ключ (`uuid`). Фильтрации конфигураций по организации в MVP **нет** | ADR-002 |
| Тестовые данные | 2 организации, 10 пользователей в разбивке 5/5, 2 конфигурации | ADR-002 / [ADR-003](adr/003-initdb-seed-endpoint.ru.md) |
| `initdb` | Без аутентификации, но **за feature-флагом** (`SEED_ENABLED`, по умолчанию `false`); идемпотентен по схеме *сброс-затем-вставка* | ADR-003 |
| Стиль API | REST, URI-версионирование `/api/v1`, объектные конверты (`{ items, total }`), единый конверт ошибки с машиночитаемым `code` | [ADR-004](adr/004-api-conventions-and-error-format.ru.md) |
| Форма полезной нагрузки конфигурации | Колонка `settings` типа `jsonb` (`settings.mapLib`) отделяет полезную нагрузку конфигурации от метаданных, поэтому будущие поля аддитивны | ADR-002 / ADR-004 |
| Документация API | `@nestjs/swagger` 7.x на `/api/docs` с `addBearerAuth()`, чтобы защищённые эндпоинты были вызываемы из UI | ADR-004 |
| Структура фронтенда | **Standalone**-приложение на Angular 22 (без NgModules, zoneless): синглтоны в `core/` + ленивые маршруты фич через `loadComponent`, функциональные `authGuard` и HTTP-интерцепторы, signals для состояния аутентификации; без библиотеки управления состоянием | [ADR-005](adr/005-frontend-structure-and-primeng.ru.md) |
| UI-набор | PrimeNG 22.1.x, настраивается через `providePrimeNG` + пресет `@primeuix/themes` 3.x (Aura), `primeicons` 8, `@angular/cdk` 22 (peer PrimeNG). **С v22 не MIT** — решение по лицензии в §2.3 / A11 | ADR-005 |
| Инструментарий фронтенда | `@angular/build` (esbuild) для сборки/serve; `ng test` → билдер `@angular/build:unit-test` с раннером **Vitest** на jsdom; требуется Node `^22.22.3 \|\| ^24.15.0 \|\| >=26.0.0` | §2.3, A10 |
| Cross-origin в разработке | Dev-прокси Angular `/api` → `http://localhost:3000`; CORS на бэкенде также настраивается для использования без прокси | ADR-005 |
| Пакетный менеджер | **npm** для обоих проектов (в обоих уже есть `package-lock.json`) | — |

### 2.3 Проверенная совместимость версий

Проверено по реестру npm 2026-09-23 (`npm view`; диапазоны peer-зависимостей и `engines`):

**Фронтенд — после апгрейда на Angular 22**

- Установлено: `@angular/*@^22.1.0`, `@angular/cli` / `@angular/build@^22.1.8`, `typescript@~6.0.2`, `rxjs@~7.8`,
  `vitest@^4` + `jsdom@^28`, `npm@11.19.0` в поле `packageManager`. Приложение standalone и **zoneless**
  (`zone.js` больше не является зависимостью).
- **Node.js:** `@angular/core@22` объявляет `engines.node = "^22.22.3 || ^24.15.0 || >=26.0.0"`. Более старые версии —
  на машине, где проводилось это ревью, стоит **Node 18.20.8** — не позволят ни установить, ни собрать `front/`.
  `back/` прекрасно работает на тех же рантаймах (`@nestjs/cli@10` требует `>= 16.14`), поэтому одна версия Node
  обслуживает весь репозиторий (A10).
- **TypeScript 6** включает `strict` по умолчанию — именно поэтому сгенерированный `tsconfig.json` его больше не
  перечисляет. В `back/` остаётся собственный `typescript@~5.1`; модели из §5.10 — обычные интерфейсы и без изменений
  компилируются в обоих проектах, так что решение о дублировании моделей (§3) не затронуто.
- **Юнит-тесты:** `ng test` запускает билдер `@angular/build:unit-test`, у которого `runner` по умолчанию — `vitest`;
  поскольку `browsers` не заданы, тесты выполняются в Node на jsdom. Karma/Jasmine больше нет — `describe/it/expect`
  в `app.spec.ts` приходят из `vitest/globals`, объявленных в `tsconfig.spec.json`.
- **PrimeNG:** актуальная версия `primeng@22.1.1`, peer'ы `@angular/core ^22.1.0`, `@angular/cdk ^22.1.0`,
  `rxjs ^6.0.0 || ^7.8.1` → совпадает с установленными версиями, но **`@angular/cdk` нужно добавить явно**.
  `@angular/animations` больше не peer (PrimeNG перешёл на CSS-анимации; пакет анимаций Angular объявлен устаревшим
  с v20.2). У `primeng@21` peer `@angular/core ^21.0.7`, поэтому здесь он неустановим.
- **Темизация PrimeNG теперь только в коде:** `providePrimeNG({ theme: { preset: Aura } })` из `primeng/config`,
  пресет — из `@primeuix/themes@^3.0` (`@primeuix/themes/aura`). Старых файлов `primeng/resources/themes/*.css`
  (`lara-light-blue` и подобных) больше не существует, поэтому единственная связанная с PrimeNG запись в
  `angular.json > styles` — это `primeicons/primeicons.css` (`primeicons@8.0.x`). `p-dropdown` объявлен устаревшим
  в пользу **`p-select`**.
- **Лицензирование PrimeNG (нужно решение человека):** начиная с v22 PrimeNG поставляется не под MIT, а по двойной
  модели PrimeUI — бесплатная лицензия **Community** (частные лица, студенты, некоммерческий open source и организации
  с выручкой < $1M, < 5 разработчиков, < 10 сотрудников, < $3M венчурных инвестиций; подтверждается ежегодно) либо
  коммерческая лицензия (~$599 на разработчика). Версии ≤ 21 остаются MIT, но привязаны к Angular ≤ 21. См. A11 и §3.

**Бэкенд — этим изменением не затронут, перепроверено**

- `@nestjs/typeorm@10.0.2` имеет peer `@nestjs/core ^8 || ^9 || ^10` и `typeorm ^0.3.0` → используем `typeorm@^0.3`
  и драйвер `pg@^8` (актуальная 8.23.x). Актуальная версия `typeorm` — 1.1.1, а 0.3.x висит под dist-tag `legacy`; мы сознательно берём линию 0.3, которую
  объявляет `@nestjs/typeorm@10`. Путь обновления: `@nestjs/typeorm@11` (peer Nest ^10 || ^11, `typeorm ^0.3 || ^1.0.0-dev`)
  + `typeorm@1` (Node ^20.19 || ^22.13 || >=24.11, покрыто A10).
- `@nestjs/swagger@7.4.2` имеет peer `@nestjs/core ^9 || ^10` и `reflect-metadata ^0.1.12 || ^0.2.0`
  (бэкенд уже фиксирует `reflect-metadata ^0.2.0`); оставляем **7.4.x** как консервативную пару для Nest 10.
- `@nestjs/jwt@10.2.0`, `@nestjs/passport@10.0.3`, `@nestjs/config@3.x` → все совместимы по peer с Nest 10
  (`back/package.json` по-прежнему NestJS 10 + TypeScript 5.1 + Jest, ровно как предполагается в §1).

### 2.4 Ответы на открытые вопросы PRD

| ОВ в PRD | Решение |
|---|---|
| №1 Схема конфигурации будет расти | Полезная нагрузка изолирована в колонке `settings` (jsonb); эндпоинт списка возвращает только метаданные; детали возвращают `settings`. Новые поля аддитивны и не ломают контракт. |
| №2 Защищён ли `initdb`? | Не защищён аутентификацией, но отключён, если `SEED_ENABLED=true` не задан; возвращает `404 SEED_DISABLED`, когда выключен. В продакшене значение `false`. |
| №3 Идемпотентен ли `initdb`? | Да — в одной транзакции он очищает (`TRUNCATE`) три таблицы и заново вставляет фиксированный набор данных. Повторные вызовы сходятся к одному и тому же состоянию (10 пользователей, 2 конфигурации, 2 организации). Разрушителен по замыслу; защищён тем же флагом. |
| №4 Механизм аутентификации | JWT bearer, срок 8 часов, без refresh; `sessionStorage` на клиенте. См. ADR-001 про компромисс с XSS и путь миграции на httpOnly-cookie. |
| №5 Подпись опций dropdown | `Configuration.name` — label, `Configuration.id` — значение; `description` доступно как вторичный текст. |
| №6 Хранение паролей | `bcrypt` (cost 10). У `passwordHash` стоит `select: false`; он никогда не возвращается ни одним эндпоинтом и не сериализуется в DTO. |
| №7 Конфигурация подключения к БД | `@nestjs/config` + `.env`; `DATABASE_URL` обязателен и валидируется при старте; `infra/docker-compose.yml` поднимает локальный PostgreSQL 16. |
| №8 Тексты ошибок/пустых состояний | Деталь реализации фронтенда; контракт фиксирует только `code` ошибок, чтобы тексты можно было замапить по коду. |
| №9 Сколько организаций в MVP | **Две** — достаточно, чтобы задействовать будущую мульти-тенантную модель, не реализуя фильтрацию. |
| №10 Назначение пользователь→организация в seed | `user01..user05` → организация 1, `user06..user10` → организация 2 (детерминированно, задокументировано ниже). |
| №11 Организации как таблица? | **Да** — настоящая таблица `organizations` с внешним ключом от пользователей. Обоснование в ADR-002. |

### 2.5 Явные допущения (пробелы в PRD)

Это допущения архитектора, а не продуктовые решения. Сообщите product-manager, если что-то из этого неверно.

- **A1.** Целевая среда развёртывания для MVP — локальная/dev (машины разработчиков + демо для QA). Продакшн-хостинг,
  терминация TLS и CI/CD здесь не проектируются (PRD помечает CI/CD как вне рамок).
- **A2.** Ожидаемый масштаб тривиален (десятки пользователей, <100 конфигураций). Пагинация, слой кеширования,
  реплики для чтения и индексы сверх индексов уникальности/поиска не проектируются.
- **A3.** «Конфигурация» в MVP — это глобальная каталожная сущность: конфигурации пока **не** принадлежат
  организации, поэтому в MVP у `configurations` нет `organizationId` (будущий инкремент фильтрации добавит
  связующую таблицу `configuration_organizations`, что является аддитивным изменением).
- **A4.** Имена пользователей простые, в стиле логина (`user01`), регистронезависимые (хранятся в нижнем регистре), не e-mail.
- **A5.** Пароль тестовых пользователей — единое общее задокументированное значение (`Password123!`) — приемлемо, поскольку
  seed-эндпоинт предназначен только для dev/test. Никогда не включайте seeding в окружении с реальными пользователями.
- **A6.** Требования к logout не заявлено; мы всё же включаем клиентский logout (сброс токена), потому что
  это ничего не стоит, а QA нужно переключаться между пользователями. Серверного отзыва токенов нет (stateless JWT).
- **A7.** Язык UI — английский, одна локаль (i18n вне рамок по PRD).
- **A8.** Фронтенд отдаётся через `ng serve` в разработке; без SSR, без Angular Universal.
- **A9.** `GET /auth/me` добавлен сверх буквального списка эндпоинтов PRD, потому что критерий приёмки
  «редирект на логин, когда сессия невалидна/истекла» (US-1, US-2) реализуется гораздо чище, когда приложение
  может валидировать восстановленный токен при старте. Это эндпоинт на 15 строк.
- **A10.** Требование к инструментарию: Angular 22 требует Node `^22.22.3 || ^24.15.0 || >=26.0.0`. На машине,
  где проводилось это ревью, стоит Node 18.20.8, поэтому ничто в `front/` не установится и не соберётся, пока Node
  не обновят. Самый дешёвый способ зафиксировать это — закоммиченный `.nvmrc` в корне репозитория; та же версия
  Node обслуживает и `back/`.
- **A11.** Мы предполагаем, что команда подпадает под бесплатную лицензию **Community** у PrimeNG (§2.3). Если нет,
  варианты — коммерческая лицензия или замена UI-набора на Angular Material 22 (MIT). Такая замена дёшева в первый
  день и дорога, когда уже написаны два экрана: ни API-контракт, ни бэкенд от UI-набора не зависят.

---

## 3. Рассмотренные альтернативы

| Решение | Отклонённые альтернативы | Почему отклонены |
|---|---|---|
| Stateless JWT bearer | (a) Express session + хранилище сессий; (b) JWT в httpOnly-cookie; (c) Basic auth на каждый запрос | (a) добавляет session-хранилище и сложность с cookie/CORS при нулевой выгоде для MVP и не является stateless; (b) безопаснее против XSS, но требует защиты от CSRF и работы с `SameSite`/credentials между `:4200`↔`:3000`, что на масштабе MVP стоит больше, чем даёт — задокументировано как намеченный шаг усиления в ADR-001; (c) пересылает учётные данные при каждом запросе и неприемлем. |
| PostgreSQL + TypeORM | (a) MongoDB + Mongoose (как названо в PRD); (b) Prisma + PostgreSQL; (c) in-memory-хранилище | **Отклонение от PRD:** (a) PRD называет MongoDB явно, но данные здесь реляционные (`users → organizations` с обязательным внешним ключом, US-1a), а гибкая часть — `settings` — укладывается в колонку `jsonb`; PRD нужно обновить (product-manager). (b) добавляет отдельный файл схемы и кодогенерацию вне декораторной модели Nest, тогда как TypeORM встраивается в DI Nest напрямую; (c) не позволяет выполнить метрики успеха «реальные данные в БД». |
| Отдельная таблица `organizations` | (a) простая строка `organizationName` у пользователя; (b) полная изоляция тенантов (БД на тенанта) | (a) дешевле сегодня, но вынуждает делать миграцию данных ровно там, где PRD называет стоимость миграции причиной проектировать сейчас (US-1a); (b) тяжеловесная мульти-тенантность для продукта с 2 тестовыми организациями и пока без фильтрации. |
| Колонка `settings` (jsonb) | (a) плоское поле `mapLib` в конфигурации; (b) произвольная jsonb-нагрузка без валидации DTO; (c) версионируемый реестр схем | (a) смешивает идентичность/метаданные и полезную нагрузку, поэтому каждое будущее поле расширяет корневой DTO; (b) отбрасывает валидацию и типизацию Swagger; (c) избыточная инженерия ради одного поля. |
| `initdb` за флагом, со сбросом | (a) полностью открытый эндпоинт; (b) эндпоинт с аутентификацией; (c) только upsert; (d) CLI-скрипт вместо эндпоинта | (a) это неаутентифицированное стирание данных в любом окружении, куда он попадёт; (b) создаёт проблему курицы и яйца (нужен пользователь, чтобы создать пользователей) и блокирует QA; (c) оставляет «уехавшие»/лишние документы, поэтому метрику «ровно 10 / ровно 2» нельзя гарантировать; (d) PRD явно требует вызываемый эндпоинт `initdb`, видимый в Swagger (US-4, US-5). CLI-скрипт можно добавить позже как тонкую обёртку над тем же сервисом. |
| Конверт списка `{ items, total }` | голый JSON-массив | Массив не сможет нести пагинацию/метаданные позже без ломающего изменения; конверт стоит одной строки на клиенте. |
| Без библиотеки управления состоянием | NgRx / NgRx Component Store / Akita / SignalStore | Два экрана и одна выбранная сущность; store был бы объёмнее самой фичи. Достаточно `AuthService` с `signal` и `computed` (ADR-005), а signals в Angular 22 уже покрывают то немногое общее состояние, которое есть. |
| Ручные TS-модели на фронте | (a) общий workspace-пакет; (b) сгенерированный клиент из Swagger JSON | (a) требует превращения репозитория в npm-workspace (CLAUDE.md говорит: корневого `package.json` нет); (b) добавляет шаг кодогенерации и связанность по порядку сборки — стоит вернуться к этому, когда контракт стабилизируется, зафиксировано как будущая работа в ADR-004. |
| PrimeNG 22 по лицензии Community | (a) остаться на PrimeNG 21 (последняя MIT-линейка); (b) Angular Material 22 (MIT); (c) вообще без UI-набора, свои компоненты | (a) у `primeng@21` peer `@angular/core ^21.0.7`, то есть «остаться на MIT» фактически означает «остаться на Angular 21» — на пересозданном скаффолде он неустановим; (b) это настоящая, чистая по лицензии альтернатива и назначенный запасной вариант, если лицензия Community не подходит (A11); отклонена сегодня лишь потому, что UI уже описан в терминах PrimeNG, а сама замена не имеет архитектурных последствий; (c) для двух экранов дороже обоих вариантов. |
| Angular 22, standalone + zoneless | (a) сохранить приложение на Angular 15 с NgModules; (b) вернуть NgModules поверх Angular 22 | **Отмечаем разворот решения:** предыдущая редакция этого документа решила остаться на Angular 15 с NgModules, потому что апгрейд был рискованной работой без ценности для MVP. Это обоснование больше не действует — `front/` с тех пор пересоздан на Angular 22, то есть апгрейд уже оплачен, и (a) потерял смысл. (b) означало бы борьбу с умолчаниями фреймворка (в скаффолде нет `app.module.ts`, а `HttpClientModule` и интерцепторы-классы — легаси) без всякой выгоды. Цена разворота: более свежий Node (A10), дисциплина zoneless-обнаружения изменений, тестовый стек на Vitest и вопрос лицензии PrimeNG — всё это разобрано ниже. |

---

## 4. Последствия

**Что мы получаем**

- Бэкенд stateless → тривиально перезапускается, в будущем масштабируется горизонтально, нет session-хранилища для эксплуатации.
- Контракт front↔back ниже достаточно полон, чтобы фронтенд и бэкенд могли начать параллельно уже сегодня;
  фронтенд может работать на моках по задокументированным формам и переключиться на реальное API без изменений кода.
- Модель данных уже несёт связь пользователь→организация, поэтому будущий инкремент
  «организация определяет, какие конфигурации видны» становится *аддитивным* изменением
  (связующая таблица `configuration_organizations` + один фильтр в запросе + один guard), а не миграцией.
- Добавление полей конфигурации позже затрагивает только `settings`: один тип `settings`, один DTO, один блок UI.
- Swagger с bearer-аутентификацией позволяет QA выполнить весь сценарий без фронтенда (US-4, метрика успеха).

**Чем расплачиваемся / что принимаем**

- **JWT в `sessionStorage` читается внедрёнными скриптами (XSS).** Принято для внутреннего MVP без
  чувствительных данных; путь смягчения (httpOnly-cookie + CSRF) задокументирован в ADR-001. Должно быть пересмотрено
  до любого продакшн-развёртывания, доступного из интернета, — это крупнейший известный долг по безопасности.
- **Нет отзыва/обновления токенов.** Украденный токен валиден до истечения срока (8 часов); logout только клиентский.
- **`initdb` разрушителен.** Если `SEED_ENABLED` когда-либо окажется `true` в окружении с реальными данными, эти данные
  будут удалены. Флаг по умолчанию `false`, а валидация конфигурации при старте делает это явным.
- **Дублирование типов** между `back/src/**/dto` и `front/src/app/core/api/api.models.ts`. Риск расхождения
  смягчается нормативностью этого документа плюс проверкой контракта в e2e-тестах; это реальный долг.
- **Фронтенд теперь идёт вровень с текущим мажором Angular.** Долга по апгрейду фреймворка в первый день нет, но
  появляются новые обязательства: Node ≥ 22.22.3 на каждой машине разработчика/CI (A10), zoneless-обнаружение
  изменений (обновления UI должны идти от signals, пайпа `async` или явного `markForCheck()`) и тестовый стек
  Vitest/jsdom вместо Karma/Jasmine. Привычки из старых Angular-проектов сюда автоматически не переносятся.
- **PrimeNG 22 не MIT.** UI-набор теперь несёт лицензионное обязательство (бесплатная лицензия Community либо
  платное место), которое должен подтвердить человек, прежде чем это уйдёт в любой коммерческий продукт. Это
  юридическая зависимость, а не техническая, и запасной вариант с Angular Material не даёт ей стать
  vendor lock-in (§2.3, §3, A11).
- **Пока нет org-фильтрации**, то есть любой залогиненный пользователь видит все конфигурации. Это сознательное,
  санкционированное PRD упрощение (US-2) — его нельзя трактовать как модель авторизации.
- **Растёт эксплуатационная поверхность**: разработчикам теперь нужны запущенный PostgreSQL (docker-compose) и `.env`.

**Оценка нефункциональных требований**

| НФТ | Цель для MVP | Как достигается / риск |
|---|---|---|
| Производительность | p95 < 200 мс на эндпоинт локально | Тривиальные полезные нагрузки, индексированный поиск по `id`/`username`; bcrypt cost 10 доминирует в логине (~50–100 мс) — намеренно. |
| Масштабируемость | Один инстанс, десятки пользователей | Stateless API может быть реплицирован за балансировщиком без изменений; PostgreSQL — единственная stateful-часть. |
| Безопасность | Никаких паролей в открытом виде, все бизнес-эндпоинты аутентифицированы, seed-флаг выключен по умолчанию | Хеширование bcrypt, `select:false` на `passwordHash`, глобальный auth-guard (запрет по умолчанию с явным `@Public()`), `ValidationPipe({whitelist:true, forbidNonWhitelisted:true})`, секрет из переменных окружения с валидацией при старте. Опциональное усиление: `@nestjs/throttler` на `/auth/login` (5 запросов/мин/IP) — рекомендуется, дёшево. |
| Надёжность | Падать быстро и заметно | Валидация окружения при старте; сбой подключения к БД прерывает запуск; глобальный exception-фильтр гарантирует типизированный конверт ошибки, чтобы UI всегда мог отрисовать состояние ошибки. |
| Поддерживаемость | Границы фич с обеих сторон | Модули Nest владеют своей схемой + сервисом + контроллером; маршруты фич Angular лениво загружаются (`loadComponent`) и зависят только от `core`. |
| Стоимость владения | Один процесс + одна база данных | Никакой инфраструктуры, кроме PostgreSQL в docker-compose. |

---

## 5. API-контракт

Нормативен и для `front/`, и для `back/`. Изменения проходят через архитектора.

### 5.1 Соглашения

- **Базовый URL:** `http://localhost:3000/api/v1` (dev). Глобальный префикс `api`, URI-версионирование `v1`
  (`app.setGlobalPrefix('api')` + `app.enableVersioning({ type: VersioningType.URI, defaultVersion: '1' })`).
- **Базовый URL на фронтенде:** `/api/v1` (относительный) — dev-прокси Angular перенаправляет `/api` на `:3000`,
  поэтому в стандартной dev-конфигурации CORS не нужен. Хранится в `core/api/api.config.ts` как `API_BASE_URL`
  (в скаффолде Angular 22 нет `src/environments/`; выполняйте `ng generate environments`, только если реально
  понадобятся сборки под разные окружения).
- **Content type:** `application/json; charset=utf-8` в обе стороны.
- **Именование полей:** `camelCase` в JSON. `map-lib` из PRD представлено как **`mapLib`** (kebab-case не
  идиоматичен для JSON/TS); это решение об именовании, а не смысловое изменение.
- **Идентификаторы:** UUID (v4, генерируется PostgreSQL) сериализуется как 36-символьная **строка** в поле `id`.
  Внутренние колонки никогда не отдаются наружу.
- **Даты:** строки ISO-8601 в UTC (`2026-09-23T10:15:30.000Z`).
- **Заголовок аутентификации:** `Authorization: Bearer <accessToken>` на каждом эндпоинте, кроме помеченных *public*.
- **Swagger:** `/api/docs` (JSON на `/api/docs-json`), с `addBearerAuth()`; у защищённых эндпоинтов показан замок.

### 5.2 Указатель эндпоинтов

| № | Метод | Путь | Аутентификация | История |
|---|---|---|---|---|
| 1 | POST | `/api/v1/auth/login` | public | US-1 |
| 2 | GET | `/api/v1/auth/me` | bearer | US-1, US-2 (валидность сессии) |
| 3 | GET | `/api/v1/configurations` | bearer | US-2 |
| 4 | GET | `/api/v1/configurations/{id}` | bearer | US-3 |
| 5 | POST | `/api/v1/admin/initdb` | public, за флагом | US-5 |
| 6 | GET | `/api/v1/health` | public | удобство эксплуатации |

### 5.3 Конверт ошибки (все ответы не-2xx)

```jsonc
{
  "statusCode": 401,
  "code": "INVALID_CREDENTIALS",      // машиночитаемый, стабильный; UI маппит тексты по нему
  "message": "Invalid username or password.",
  "details": null,                     // string[] для ошибок валидации, иначе null
  "timestamp": "2026-09-23T10:15:30.000Z",
  "path": "/api/v1/auth/login"
}
```

Коды ошибок, используемые в MVP:

| `code` | HTTP | Когда |
|---|---|---|
| `VALIDATION_ERROR` | 400 | Тело/параметр не прошли `class-validator`; `details` перечисляет сообщения |
| `INVALID_CREDENTIALS` | 401 | `/auth/login` с неизвестным username или неверным паролем |
| `UNAUTHENTICATED` | 401 | Отсутствующий/невалидный/просроченный bearer-токен на защищённом эндпоинте |
| `NOT_FOUND` | 404 | Неизвестный id конфигурации или неизвестный маршрут |
| `SEED_DISABLED` | 404 | `/admin/initdb` вызван при `SEED_ENABLED=false` |
| `INTERNAL_ERROR` | 500 | Необработанное исключение (сообщение обобщённое; детали никогда не раскрывают внутренности) |

> Замечание по безопасности: `INVALID_CREDENTIALS` возвращается и для «неизвестного пользователя», и для «неверного пароля» —
> без перечисления пользователей. UI показывает одно сообщение для обоих случаев (US-1).

### 5.4 `POST /api/v1/auth/login` — public

Запрос:
```jsonc
{ "username": "user01", "password": "Password123!" }
```
Валидация: `username` — обязателен, строка, 3..64 символа, обрезается по краям, приводится к нижнему регистру на сервере;
`password` — обязателен, строка, 8..128 символов. Лишние свойства отклоняются (`400 VALIDATION_ERROR`).

`200 OK`:
```jsonc
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "tokenType": "Bearer",
  "expiresIn": 28800,                       // секунды
  "user": {
    "id": "3f1c2a7e-9b5d-4c8a-a1e2-7d0f4b6c9e21",
    "username": "user01",
    "displayName": "User 01",
    "organization": { "id": "a7d2e4c1-5b3f-4e6a-9c8d-1f0b2a3c4d5e", "name": "Northwind Geo" }
  }
}
```
Ошибки: `400 VALIDATION_ERROR`, `401 INVALID_CREDENTIALS`.

Payload JWT (HS256, `JWT_SECRET`, `expiresIn = JWT_EXPIRES_IN`, по умолчанию `8h`):
```jsonc
{ "sub": "<userId>", "username": "user01", "orgId": "<organizationId>", "iat": 1790000000, "exp": 1790028800 }
```
`orgId` переносится уже сейчас и не используется для фильтрации в MVP, поэтому будущий инкремент не потребует изменения токена.

### 5.5 `GET /api/v1/auth/me` — bearer

`200 OK` — тот же объект `user`, что и в ответе логина:
```jsonc
{
  "id": "3f1c2a7e-9b5d-4c8a-a1e2-7d0f4b6c9e21",
  "username": "user01",
  "displayName": "User 01",
  "organization": { "id": "a7d2e4c1-5b3f-4e6a-9c8d-1f0b2a3c4d5e", "name": "Northwind Geo" }
}
```
Ошибки: `401 UNAUTHENTICATED` (в том числе когда пользователь, на которого ссылается всё ещё валидный токен, больше
не существует — например, после того как `initdb` очистил БД; фронт тогда перенаправляет на логин).

### 5.6 `GET /api/v1/configurations` — bearer

Возвращает **все** конфигурации (в MVP нет org-фильтрации — см. A3). Элементы списка содержат только метаданные;
`settings` намеренно исключён, чтобы список оставался дешёвым по мере роста схемы.

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
Пустой случай (пустое состояние `US-2`): `{ "items": [], "total": 0 }` со статусом `200` — **не** 404.
Ошибки: `401 UNAUTHENTICATED`.
Порядок: по `name` по возрастанию (стабильный порядок в dropdown).

### 5.7 `GET /api/v1/configurations/{id}` — bearer

`id` — UUID; некорректные id отклоняются с `400 VALIDATION_ERROR` (через `ParseUUIDPipe`),
так что плохой id никогда не доходит до БД.

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
Ошибки: `400 VALIDATION_ERROR`, `401 UNAUTHENTICATED`, `404 NOT_FOUND`.

**Правило расширяемости (нормативное):** будущие поля конфигурации добавляются **внутрь `settings`**.
Клиенты обязаны игнорировать неизвестные ключи в `settings`. `settings.mapLib` остаётся обязательным.

### 5.8 `POST /api/v1/admin/initdb` — public, за флагом

Тела запроса нет. Поведение (ADR-003): в одной транзакции выполнить один оператор `TRUNCATE TABLE configurations, users, organizations RESTART IDENTITY CASCADE`
(именно одним оператором: PostgreSQL не позволяет очистить по отдельности таблицу, на которую ссылается внешний ключ), затем вставить фиксированный набор данных. Идемпотентен в смысле «сходится к известному состоянию».

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
    "…10 записей…"
  ],
  "configurations": [
    { "id": "…", "name": "OpenLayers Default", "mapLib": "openlayers" },
    { "id": "…", "name": "ArcGIS Default",     "mapLib": "arcgis" }
  ],
  "defaultPassword": "Password123!"   // эндпоинт только для dev; позволяет QA залогиниться, не читая код
}
```
Ошибки: `404 SEED_DISABLED`, когда `SEED_ENABLED !== true`; `500 INTERNAL_ERROR` при сбое БД
(весь seed выполняется в одной транзакции, поэтому при сбое она откатывается и предыдущее состояние остаётся нетронутым).

### 5.9 `GET /api/v1/health` — public

`200 OK` → `{ "status": "ok", "db": "up", "uptime": 123.4 }` (`db` — это `"up" | "down"` из состояния
подключения TypeORM через `SELECT 1`). Используется QA/разработчиками, чтобы убедиться, что стек собран, прежде чем отлаживать UI.

### 5.10 Общие TypeScript-модели (дублируются дословно с обеих сторон)

Бэкенд: DTO-классы с `@ApiProperty`. Фронтенд: `front/src/app/core/api/api.models.ts`.

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

export interface ConfigurationSettings { mapLib: MapLib; }          // будущие поля идут сюда
export interface ConfigurationDetail {
  id: string;
  name: string;
  description?: string;
  settings: ConfigurationSettings;
  createdAt: string;
  updatedAt: string;
}

export type ApiErrorCode =
  | 'VALIDATION_ERROR' | 'INVALID_CREDENTIALS' | 'UNAUTHENTICATED'
  | 'NOT_FOUND' | 'SEED_DISABLED' | 'INTERNAL_ERROR';

export interface ApiError {
  statusCode: number;
  code: ApiErrorCode;
  message: string;
  details: string[] | null;
  timestamp: string;
  path: string;
}
```

---

## 6. Модель данных

### 6.1 Таблицы

**`organizations`**

| Поле | Тип | Ограничения |
|---|---|---|
| `id` | uuid | первичный ключ, `gen_random_uuid()` |
| `name` | string | обязательное, уникальное, 2..120 |
| `slug` | string | обязательное, уникальное, kebab-case в нижнем регистре (`northwind-geo`) — стабильный идентификатор для будущих ссылок из конфигураций/seed |
| `createdAt` / `updatedAt` | timestamptz | `@CreateDateColumn` / `@UpdateDateColumn` |

Индексы: unique по `name`, unique по `slug`.

**`users`**

| Поле | Тип | Ограничения |
|---|---|---|
| `id` | uuid | первичный ключ, `gen_random_uuid()` |
| `username` | string | обязательное, уникальное, хранится в нижнем регистре, 3..64 |
| `passwordHash` | string | обязательное, bcrypt cost 10, **`select: false`** |
| `displayName` | string | обязательное, 1..120 |
| `organizationId` | uuid | **обязательное**, FK → `organizations.id` (`ON DELETE RESTRICT`) (US-1a: ровно одна организация) |
| `createdAt` / `updatedAt` | timestamptz | `@CreateDateColumn` / `@UpdateDateColumn` |

Индексы: unique по `username`, обычный по `organizationId` (FK; закладывается под org-ориентированные запросы).
Регистронезависимость `username` обеспечивается нормализацией, а не индексом по выражению (декораторы TypeORM такой индекс не описывают, а `citext` требует расширения): lowercase + trim в DTO логина (`@Transform`) и в `@BeforeInsert()` сущности, поверх обычного ограничения `unique`.

**`configurations`**

| Поле | Тип | Ограничения |
|---|---|---|
| `id` | uuid | первичный ключ, `gen_random_uuid()` |
| `name` | string | обязательное, уникальное, 1..120 — это label в dropdown (ОВ №5) |
| `description` | string | необязательное, ≤500 |
| `settings` | jsonb | обязательный; MVP: `{ mapLib: 'openlayers' \| 'arcgis' }` (обязательный enum) |
| `createdAt` / `updatedAt` | timestamptz | `@CreateDateColumn` / `@UpdateDateColumn` |

Индексы: unique по `name`.
Ограничение: `@Check("(settings->>'mapLib') IN ('openlayers','arcgis')")` на сущности — единственная защита от некорректного seed, так как в MVP нет эндпоинтов записи.
Будущее (не MVP): связующая таблица `configuration_organizations` (`configurationId`, `organizationId`) для видимости в рамках организации.

Управление схемой: `synchronize` TypeORM — только вне production; версионируемые миграции — до появления любого общего окружения. TypeORM нужно настроить с `uuidExtension: 'pgcrypto'` (его умолчание `uuid-ossp` требует superuser), чтобы первичные ключи генерировал `gen_random_uuid()`.

### 6.2 Связи

```
Organization 1 ──────< N User            (users.organizationId, обязательное)
Configuration                            (в MVP — глобальный каталог, связи с организацией пока нет)
```

### 6.3 Набор тестовых данных (фиксированный, детерминированный — US-5)

Организации (2):

| name | slug |
|---|---|
| Northwind Geo | `northwind-geo` |
| Acme Mapping | `acme-mapping` |

Пользователи (10), пароль `Password123!` для всех, хеширован bcrypt:

| username | displayName | организация |
|---|---|---|
| `user01` … `user05` | `User 01` … `User 05` | Northwind Geo |
| `user06` … `user10` | `User 06` … `User 10` | Acme Mapping |

Конфигурации (2):

| name | description | settings.mapLib |
|---|---|---|
| `ArcGIS Default` | Baseline configuration using the ArcGIS map library. | `arcgis` |
| `OpenLayers Default` | Baseline configuration using the OpenLayers map library. | `openlayers` |

(Эндпоинт списка сортирует по `name`, поэтому `ArcGIS Default` появляется в dropdown первым.)

---

## 7. Структура бэкенда (`back/`)

```
back/src/
  main.ts                     # префикс+версионирование, ValidationPipe, глобальный фильтр, Swagger, CORS
  app.module.ts               # ConfigModule.forRoot(global+validated), TypeOrmModule.forRootAsync, feature-модули
  common/
    filters/all-exceptions.filter.ts     # формирует конверт из §5.3 для каждой ошибки
    dto/api-error.dto.ts                 # Swagger-модель конверта ошибки
    decorators/public.decorator.ts       # @Public() → пропускает глобальный JwtAuthGuard
  config/
    configuration.ts                     # типизированная фабрика конфигурации
    env.validation.ts                    # схема Joi/class-validator; падает быстро при старте
  organizations/
    entities/organization.entity.ts
    organizations.module.ts
  users/
    entities/user.entity.ts
    users.service.ts                     # findByUsername(+hash), findById
    users.module.ts
  auth/
    auth.module.ts  auth.controller.ts  auth.service.ts
    strategies/jwt.strategy.ts
    guards/jwt-auth.guard.ts             # регистрируется как APP_GUARD (запрет по умолчанию)
    dto/{login.request.dto.ts,login.response.dto.ts,authenticated-user.dto.ts}
    decorators/current-user.decorator.ts
  configurations/
    entities/configuration.entity.ts
    configurations.module.ts  .service.ts  .controller.ts
    dto/{configuration-summary.dto.ts,configuration-list.dto.ts,configuration-detail.dto.ts}
  seed/
    seed.module.ts  seed.controller.ts   # POST /admin/initdb
    seed.service.ts                      # сброс + вставка; переиспользуем из будущего CLI-скрипта
    seed.data.ts                         # фиксированный набор данных выше
    guards/seed-enabled.guard.ts         # бросает NotFound(SEED_DISABLED), когда флаг выключен
  health/health.controller.ts
```

Окружение (`back/.env`, с закоммиченным `back/.env.example`):

| Переменная | Пример | Примечания |
|---|---|---|
| `PORT` | `3000` | |
| `DATABASE_URL` | `postgres://config_viewer:config_viewer@localhost:5432/config_viewer` | обязательна, валидируется |
| `JWT_SECRET` | *(dev-значение в `.env.example`)* | обязателен, минимум 32 символа |
| `JWT_EXPIRES_IN` | `8h` | |
| `CORS_ORIGINS` | `http://localhost:4200` | через запятую; используется, когда работа идёт не через dev-прокси |
| `SEED_ENABLED` | `true` локально, `false` по умолчанию/в проде | ограничивает `/admin/initdb` |

`infra/docker-compose.yml` (корень репозитория) поднимает `postgres:16` на `5432` с именованным томом (плюс отдельную БД `config_viewer_test` для e2e, создаваемую скриптом из `/docker-entrypoint-initdb.d`, поскольку образ создаёт только БД из `POSTGRES_DB`) — единственный элемент инфраструктуры.

---

## 8. Структура фронтенда (`front/`)

Пересозданный скаффолд — это Angular 22: standalone-компоненты, **zoneless**-обнаружение изменений (`zone.js` не
является зависимостью), `@angular/build` (esbuild) для сборки/serve и Vitest для юнит-тестов. Файла `app.module.ts`
нет, и мы его не заводим.

```
front/src/
  main.ts                     # bootstrapApplication(App, appConfig)
  styles.scss                 # только стили приложения — темизация PrimeNG задаётся в коде (см. ниже)
  app/
    app.config.ts             # провайдеры ApplicationConfig:
                              #   provideBrowserGlobalErrorListeners()
                              #   provideRouter(routes)
                              #   provideHttpClient(withInterceptors([authInterceptor, errorInterceptor]))
                              #   providePrimeNG({ theme: { preset: Aura } })
                              #   provideAppInitializer(() => inject(AuthService).restoreSession())
    app.routes.ts             # '' → /configurations ; 'login' ; 'configurations' (authGuard) ; '**' → ''
    app.ts / app.html / app.scss          # компонент-оболочка: <router-outlet>
    core/
      api/api.config.ts       # export const API_BASE_URL = '/api/v1'
      api/api.models.ts       # типы из §5.10 (зеркало контракта)
      auth/auth.service.ts    # login(), logout(), restoreSession(); состояние в signals: user(), isAuthenticated()
      auth/token.storage.ts   # чтение/запись/очистка sessionStorage — единственное место работы с хранилищем
      auth/auth.guard.ts      # authGuard: CanActivateFn → редирект на /login с returnUrl
      http/auth.interceptor.ts    # HttpInterceptorFn — добавляет Authorization: Bearer, когда токен есть
      http/error.interceptor.ts   # HttpInterceptorFn — 401 → logout + редирект; маппит ApiError для UI
      services/configurations.service.ts    # list(), getById()
    features/
      auth/login-page.ts                        # standalone-компонент, лениво загружается
      configurations/config-selection-page.ts   # standalone-компонент, лениво загружается
  proxy.conf.json             # /api → http://localhost:3000
```

- **Маршрутизация/guard:** по одному ленивому маршруту на экран —
  `{ path: 'login', loadComponent: () => import('./features/auth/login-page').then(m => m.LoginPage) }`, и так же
  для `configurations` с `canActivate: [authGuard]`. `authGuard` — функциональный `CanActivateFn`, использующий
  `inject(AuthService)` / `inject(Router)`; неаутентифицированная навигация перенаправляет на `/login?returnUrl=…`
  (US-1). `loadChildren` + собственный `routes.ts` фичи применяем, только когда фича перерастает один маршрут.
- **Восстановление сессии:** `provideAppInitializer(() => inject(AuthService).restoreSession())` читает токен из
  `sessionStorage` и валидирует его через `GET /auth/me`; ответ 401 молча его очищает. Это избавляет от мелькания
  аутентифицированной оболочки.
- **Интерцепторы:** функциональные `HttpInterceptorFn`, регистрируются один раз через
  `provideHttpClient(withInterceptors([authInterceptor, errorInterceptor]))` — порядок важен: сначала добавить
  токен, затем маппить ошибки и обрабатывать 401. Никакого `HttpClientModule` и мульти-провайдера
  `HTTP_INTERCEPTORS`.
- **Состояние и zoneless:** `AuthService` держит `user = signal<AuthenticatedUser | null>(null)` и
  `isAuthenticated = computed(() => this.user() !== null)`; компоненты страниц выставляют signals и используют пайп
  `async` для одноразовых HTTP-потоков. Любое обновление UI должно исходить от signal, пайпа `async` или явного
  `markForCheck()` — голый `subscribe(() => this.x = …)` перерисовку не гарантирует.
- **Экраны:**
  - *Логин* — PrimeNG `p-inputtext`/`p-password`/`p-button`, reactive form, валидация обязательных полей
    (отправка заблокирована + сообщения по каждому полю), серверная ошибка показывается в `p-message` (US-1).
  - *Выбор конфигурации* — **`p-select`** (преемник устаревшего `p-dropdown` в линейке v22), привязанный к
    `ConfigurationSummary[]` (`optionLabel="name"`, `optionValue="id"`), состояния: загрузка (спиннер),
    пусто («No configurations available»), ошибка + кнопка Retry, загружено. При смене выбора: запросить детали,
    отменить/игнорировать устаревшие ответы через `switchMap`, очистить предыдущие детали до отрисовки новых
    (US-2, US-3).
  - *Детали* — карточка, показывающая `Map library: OpenLayers | ArcGIS` (подпись маппится из `settings.mapLib`),
    плюс явное состояние «не найдено»/ошибки.
- **Dev-прокси:** `front/proxy.conf.json` → `{"/api": {"target": "http://localhost:3000", "secure": false}}`,
  подключённый в `angular.json` через `serve.options.proxyConfig` — механизм не изменился и в билдере
  `@angular/build:dev-server`.
- **Зависимости для добавления:** `primeng@^22.1`, `@primeuix/themes@^3`, `primeicons@^8` и `@angular/cdk@^22`
  (peer PrimeNG, который не ставится автоматически). `@angular/animations` — **не** нужен. Темизация только в коде:
  `providePrimeNG({ theme: { preset: Aura } })`, где `Aura` берётся из `@primeuix/themes/aura`; единственная запись
  стилей, которую надо добавить в `angular.json`, — `primeicons/primeicons.css`; CSS-файлы тем
  `primeng/resources/**` больше не существуют.
- **Никакого `SharedModule`:** каждый standalone-компонент импортирует ровно те компоненты PrimeNG, которые
  использует (`Button`, `InputText`, `Password`, `Select`, `Card`, `Message`, `ProgressSpinner`).

---

## 9. Сквозные аспекты

- **Валидация:** `app.useGlobalPipes(new ValidationPipe({ whitelist: true, forbidNonWhitelisted: true, transform: true }))`;
  её ошибки преобразуются в `VALIDATION_ERROR` с `details`.
- **Модель авторизации:** глобальный `JwtAuthGuard` (`APP_GUARD`) → каждый маршрут защищён, если не помечен
  `@Public()` (login, initdb, health, Swagger). Запрет по умолчанию предотвращает баги «забыли повесить guard».
- **Сериализация:** только ответы, смаппленные в DTO; никогда не возвращать сущности TypeORM напрямую (предотвращает
  утечки `passwordHash`).
- **Логирование:** встроенный логгер Nest; логировать неудачи аутентификации на уровне `warn` без пароля; тела запросов не логировать.
- **Тестирование:**
  - back — юнит-тесты для `AuthService` (сравнение хеша, payload токена) и `ConfigurationsService`;
    e2e (Supertest + отдельная БД `config_viewer_test` в PostgreSQL из docker-compose), покрывающие логин 200/401, защищённый 401, список, детали 200/404,
    счётчики initdb. E2E-набор также выступает регрессионной защитой контракта.
  - front — Vitest на jsdom (`ng test`): юнит-тесты для `AuthService`, функционального `authGuard` и
    интерцепторов, настраиваемые в `TestBed` через `provideHttpClient(withInterceptors([...]))` +
    `provideHttpClientTesting()` (`HttpClientTestingModule` — легаси); тесты компонентов для трёх состояний
    страницы выбора. В zoneless-режиме перед проверкой отрисованного вывода делайте `await fixture.whenStable()`.
- **Определение «соответствует контракту»:** ответы в точности соответствуют §5 (имена полей, конверт, `code` ошибок).

---

## 10. Задачи для команды

**backend-developer (`back/`)**
1. Добавить зависимости: `@nestjs/config`, `@nestjs/typeorm@^10.0.2`, `typeorm@^0.3`, `pg@^8`, `@nestjs/swagger@^7.4`, `@nestjs/jwt@^10.2`,
   `@nestjs/passport@^10`, `passport`, `passport-jwt`, `bcrypt`, `class-validator`, `class-transformer`
   (+ `@types/passport-jwt`, `@types/bcrypt` как dev-зависимости).
2. Bootstrap (`main.ts`): глобальный префикс `api`, URI-версионирование v1, `ValidationPipe`, глобальный exception-фильтр,
   CORS из `CORS_ORIGINS`, Swagger на `/api/docs` с `addBearerAuth()`.
3. `config/` + `.env.example` + валидация окружения при старте (падать сразу, если нет `DATABASE_URL`/`JWT_SECRET`).
4. Сущности + модули: `organizations`, `users`, `configurations` согласно §6 (включая индексы). Настроить TypeORM с `uuidExtension: 'pgcrypto'`, добавить `@Check` и нормализацию username по §6.1, а также `data-source.ts` и скрипты миграций через CLI `typeorm` (нужны до любого общего окружения).
5. Модуль аутентификации: `POST /auth/login`, `GET /auth/me`, стратегия JWT, `JwtAuthGuard` как `APP_GUARD`,
   декоратор `@Public()`, проверка bcrypt, единообразный `INVALID_CREDENTIALS`.
6. Модуль конфигураций: эндпоинты списка и деталей, маппинг DTO, `404 NOT_FOUND`, валидация id.
7. Seed-модуль: `POST /admin/initdb`, `SeedEnabledGuard`, сброс-затем-вставка набора данных из §6.3,
   ответ согласно §5.8.
8. `GET /health`; `infra/docker-compose.yml` с `postgres:16` и init-скриптом, создающим `config_viewer_test`.
9. Тесты согласно §9; проверить каждую форму ответа из §5.

**frontend-developer (`front/`)**

*Предусловие: Node `^22.22.3 || ^24.15.0 || >=26.0.0` (A10) — на более старых рантаймах `npm install` в `front/` падает.*

1. Установить `primeng@^22.1`, `@primeuix/themes@^3`, `primeicons@^8`, `@angular/cdk@^22`; настроить
   `providePrimeNG({ theme: { preset: Aura } })` в `app.config.ts` и добавить в `angular.json > styles` только
   `primeicons/primeicons.css`. **Не** добавлять `@angular/animations` и какие-либо CSS-темы `primeng/resources/**` —
   их больше нет. Добавить `proxy.conf.json` и сослаться на него из `serve.options.proxyConfig` в `angular.json`.
2. Создать `core/` (модели API из §5.10, `API_BASE_URL`, `AuthService` на signals, `TokenStorage`, функциональный
   `authGuard`, функциональные интерцепторы аутентификации и ошибок) и подключить всё в `app.config.ts` через
   `provideHttpClient(withInterceptors([...]))` + `provideAppInitializer(...)`. Без `SharedModule`.
3. Маршрутизация согласно §8, включая ленивый `loadComponent`, guard, `returnUrl` и fallback `**`.
4. Страница логина (US-1): reactive form, клиентская валидация обязательных полей, сообщение о серверной ошибке, редирект при успехе.
5. Страница выбора конфигурации (US-2, US-3): `p-select`, наполняемый из `GET /configurations`, состояния загрузки/пусто/ошибка+повтор,
   запрос деталей при выборе с защитой от устаревших ответов, карточка деталей, состояние «не найдено».
6. Обработка 401: очистить сессию и перенаправить на `/login` (общее поведение интерцептора).
7. Юнит-тесты согласно §9 (Vitest, не Karma/Jasmine). Пока бэкенд не поднят, работать на моках по формам из §5 —
   никакой самодеятельности по контракту.

**teamlead**
1. Декомпозировать §10 на тикеты; пункты 1–4 бэкенда и 1–3 фронтенда — критический путь, они могут выполняться
   полностью параллельно по этому контракту.
2. Заполнить TODO в `CLAUDE.md` по этому документу: пакетный менеджер = **npm** (в `front/package.json` закреплён
   `npm@11.19.0`), база данных = **PostgreSQL 16**, команды запуска (`front`: `npm start` / `npm test` — Vitest;
   `back`: `npm run start:dev` / `npm test` / `npm run test:e2e`). Зафиксировать требование к Node (A10);
   закоммиченный `.nvmrc` в корне репозитория — самый дешёвый способ его соблюсти.
3. Назначить владельца и получить явный ответ по вопросу лицензирования PrimeNG (A11) **до** того, как начнётся
   работа по фронтенду: запасной вариант с Angular Material дёшев в первый день и дорог, когда уже написаны два экрана.
4. Решить, планировать ли опциональные пункты усиления (rate limiting на логине, миграция на httpOnly-cookie,
   сгенерированный из OpenAPI клиент) как тикеты долга после MVP — все они зафиксированы в ADR.

**qa-engineer**
1. Тест-план по §5 (коды статусов и значения `code` проверяемы утверждениями) и по набору тестовых данных из §6.3.
2. Сценарий: `POST /admin/initdb` → логин в Swagger под `user01` → authorize в Swagger → список → детали →
   повторить в UI; плюс негативные случаи (неверный пароль, просроченный/отсутствующий токен, неизвестный id, пустой список).

**Открытые пункты для подтверждения с product-manager**
- Допущения A1–A11 (§2.5), в частности A5 (общий пароль тестовых пользователей) и A9 (добавлен `GET /auth/me`).
- A11 / лицензирование PrimeNG — решение бизнеса, а не архитектуры: кто-то должен подтвердить, что команда
  подпадает под бесплатную лицензию Community, либо утвердить запасной вариант с Angular Material.
- То, что «организация» намеренно не видна в UI в MVP (она возвращается API, но не отображается);
  показать её в шапке стоит практически ничего, если продукт этого захочет.
