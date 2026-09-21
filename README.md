# IT-команда из агентов Claude Code

Готовый набор из 6 сабагентов (subagents) для Claude Code: **product-manager, architect, teamlead, frontend-developer, backend-developer, qa-engineer**.

Файлы лежат в `.claude/agents/` в корне этого репозитория (`map-end-to-end`) — то есть на уровень выше ваших будущих папок `front` и `back`. Это сделано специально: конфигурация в корне репозитория действует на весь проект, включая обе подпапки, так что не нужно дублировать её отдельно для frontend и backend.

## Проверка

В Claude Code CLI (открытом в корне `map-end-to-end`) выполните:

```
/agents
```

Должны появиться все 6 ролей.

## Как вызывать

- Автоматически — Claude Code сам подбирает роль по полю `description` в каждом файле.
- Явно — например: "используй агента architect, чтобы спроектировать API между front и back".

## Типовой процесс

1. **product-manager** — PRD и user stories → `docs/product/`.
2. **architect** — архитектура, API-контракт между front и back → `docs/architecture/`.
3. **teamlead** — декомпозиция задач для frontend/backend/QA → `docs/planning/`.
4. **frontend-developer** (работает в `front/`) и **backend-developer** (работает в `back/`) — реализация параллельно, по общему контракту из шага 2.
5. **qa-engineer** — тест-план, тестирование, баг-репорты.
6. **teamlead** — финальное код-ревью и вердикт.

## Модели и права

- `architect`, `teamlead` — модель `opus` (более сложные решения и ревью).
- `product-manager`, `frontend-developer`, `backend-developer`, `qa-engineer` — модель `sonnet`.
- У `product-manager` нет доступа к `Edit`/`Write` кода — только к чтению и документам.

Подробности и обоснование каждой роли — в самих файлах `.claude/agents/*.md`.
