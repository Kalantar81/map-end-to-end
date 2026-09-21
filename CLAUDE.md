# map-end-to-end

## О проекте

<!-- TODO: 2–3 предложения — что делает продукт, для кого, ключевая идея. -->

## Структура репозитория

Монорепозиторий с двумя независимыми проектами:

- `front/` — клиентская часть, **Angular**.
- `back/` — серверная часть, **NestJS**.
- `docs/` — артефакты команды агентов:
  - `docs/product/` — PRD, user stories (product-manager)
  - `docs/architecture/` — архитектура, API-контракты, ADR (architect)
  - `docs/planning/` — декомпозиция задач, планы (teamlead)
- `.claude/agents/` — команда сабагентов Claude Code.

У `front/` и `back/` свои `package.json`; общего корневого нет. Внутри `front/` и `back/` не должно быть собственных `.git` — репозиторий один, в корне.

## Стек и соглашения

- Frontend: Angular, TypeScript.
- Backend: NestJS, TypeScript.
- Пакетный менеджер: <!-- TODO: npm / pnpm / yarn — один для обеих частей -->
- База данных: <!-- TODO -->
- Команды запуска и тестов: <!-- TODO: front — ng serve / ng test; back — npm run start:dev / npm run test -->

## Команда агентов

Агенты лежат в `.claude/agents/`: product-manager, architect, teamlead, frontend-developer, backend-developer, qa-engineer.

Типовой порядок для новой фичи:

1. **product-manager** → PRD в `docs/product/`
2. **architect** → архитектура и API-контракт front↔back в `docs/architecture/`
3. **teamlead** → задачи для frontend/backend/QA в `docs/planning/`
4. **frontend-developer** (работает в `front/`) и **backend-developer** (работает в `back/`) — параллельно, по общему API-контракту
5. **qa-engineer** → тест-план, тестирование, баг-репорты
6. **teamlead** → финальное ревью и вердикт

Правила:

- Между шагами передавайте артефакты файлами в `docs/`, а не «из памяти»: контекст у агентов изолирован.
- После каждого шага показывайте промежуточный результат и ждите подтверждения.
- Оркестрирует только основная сессия. Агенты других агентов не запускают (в их `tools` нет `Agent` — не добавлять).
- Для мелких правок не гоняйте фичу через всю цепочку — вызывайте одного нужного агента.
- API-контракт — единственный источник правды для front и back; изменения контракта делает только architect.
