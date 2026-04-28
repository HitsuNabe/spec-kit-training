# TASKS — Практичний бекенд

> 7 задач на тренувальному проекті `fastapi/full-stack-fastapi-template`. Виконуйте через повний spec-kit workflow (Сценарій 2 з `SPEC_KIT_USE_CASES.md`). Деталі у форматі Jira — у `TASKS_JIRA.md`.

---

## SDD-1 — Tags для items (M2M)

**Складність**: Easy · **Story Points**: 5

Користувач може створювати теги (per-user) і призначати кілька тегів одному item. У списку items видно теги; можна фільтрувати list по тегу.

**Backend:**
- Модель `Tag` (id UUID, name str unique, owner_id FK)
- Linked table `ItemTag` (item_id FK, tag_id FK)
- Alembic migration
- CRUD endpoints `/api/v1/tags`
- Patch endpoint `/api/v1/items/{id}` приймає `tag_ids: list[UUID]`
- Filter `?tag=<name>` у `GET /api/v1/items`

**Frontend:**
- `TagBadge` компонент (кольоровий бейдж)
- `TagSelector` (multiselect з autocomplete)
- Інтеграція в Item create/edit form
- Фільтрація списку items по кліку на тег

---

## SDD-2 — Search & Filter

**Складність**: Easy · **Story Points**: 3

Реалізовано в `SPEC_KIT_WORKFLOW_GUIDE.md` як reference.

**Backend:**
- Параметр `q: Optional[str]` у `GET /api/v1/items`
- ILIKE substring match на `title`
- Escape special chars (`%`, `_`, `\`)

**Frontend:**
- `SearchBar` компонент з clear button + Esc
- `useDebouncedValue` hook (300ms)
- URL state через `useSearchParams` (`?q=...`)
- Empty state component

---

## SDD-3 — Comments з permissions

**Складність**: Medium · **Story Points**: 8

Authenticated users можуть лишати коментарі під items. Власник item може видаляти будь-які; інші — тільки свої.

**Backend:**
- Модель `Comment` (id, item_id FK, author_id FK, body, created_at)
- Alembic migration
- CRUD endpoints `/api/v1/items/{id}/comments`
- Permission check: `is_owner_of_item OR is_author_of_comment` для DELETE
- Pagination + сортування за `created_at DESC`

**Frontend:**
- `CommentList` (paginated, infinite scroll)
- `CommentForm` (markdown preview)
- Кнопка Delete тільки для дозволених коментарів
- Optimistic update при додаванні

---

## SDD-4 — Dashboard з recharts

**Складність**: Medium · **Story Points**: 8

Dashboard-сторінка з агрегаціями: items per week (line chart), items per tag (bar chart), top contributors (pie chart).

**Backend:**
- Endpoint `GET /api/v1/dashboard/summary` з 3 секціями
- SQL агрегації (GROUP BY week, GROUP BY tag, GROUP BY owner)
- Кешування 5 хвилин (in-memory або Redis)
- Permission: тільки superuser

**Frontend:**
- Сторінка `/dashboard`
- 3 chart-компоненти на `recharts`
- Loading/error states
- Date range picker (last 7/30/90 days)

---

## SDD-5 — Favorites з optimistic UI

**Складність**: Easy · **Story Points**: 5

Користувач може зірочкою позначати items як favorite. Toggle мгновений (optimistic), синхронізується з backend.

**Backend:**
- Модель `Favorite` (user_id, item_id, created_at) — composite PK
- Alembic migration
- Endpoints `POST /api/v1/items/{id}/favorite`, `DELETE` (idempotent)
- Поле `is_favorite` у відповіді `GET /api/v1/items/{id}` (computed)
- Filter `?favorite=true` у list endpoint

**Frontend:**
- StarIcon з 2 станами
- Optimistic update в TanStack Query (через `onMutate`)
- Rollback при failure
- Sidebar tab "My Favorites"

---

## SDD-6 — CSV Export через StreamingResponse

**Складність**: Medium · **Story Points**: 5

Кнопка "Export to CSV" у списку items. Backend стримить великі датасети без завантаження в памʼять.

**Backend:**
- Endpoint `GET /api/v1/items/export?format=csv`
- `StreamingResponse` з `csv.writer`
- Підтримка фільтрів `?q=`, `?tag=`, `?favorite=`
- Header `Content-Disposition: attachment; filename=items-YYYY-MM-DD.csv`
- Тест на 50K rows (memory < 50MB)

**Frontend:**
- Кнопка "Export" у toolbar
- Прогрес-індикатор під час downloading
- Враховує поточні фільтри списку

---

## SDD-7 — Activity Log (audit middleware)

**Складність**: Hard · **Story Points**: 13

Кожна mutating операція (POST/PUT/PATCH/DELETE) логується в audit-таблицю з actor, resource, action, before/after diff.

**Backend:**
- Модель `ActivityLog` (id, actor_id, action, resource_type, resource_id, payload_diff JSONB, ip, ts)
- FastAPI middleware: перехоплює всі mutating routes
- JSON-diff для before/after (через `jsondiff` або власний)
- Alembic migration з partitioning по місяцю
- Endpoint `GET /api/v1/activity?actor=&resource=&from=&to=` з permissions (superuser only)
- Async write (BackgroundTask) — не блокує response

**Frontend:**
- Сторінка `/admin/activity`
- Filter panel (date range, actor, resource type)
- Detail modal показує diff
- Pagination (cursor-based)

---

## Підсумкова таблиця

| ID | Задача | Складність | Backend | Frontend |
|----|--------|------------|---------|----------|
| SDD-1 | Tags (M2M) | Easy | model, migration, CRUD, filter | badge, multiselect, integration |
| SDD-2 | Search & Filter | Easy | q param, ILIKE, escape | SearchBar, debounce, URL state |
| SDD-3 | Comments | Medium | model, migration, permissions | list, form, optimistic |
| SDD-4 | Dashboard | Medium | aggregations, caching | 3 charts, date range |
| SDD-5 | Favorites | Easy | M2M, computed field | optimistic, rollback |
| SDD-6 | CSV Export | Medium | StreamingResponse, filters | button, progress |
| SDD-7 | Activity Log | Hard | middleware, diff, async | admin page, filters, modal |

---

## Як обрати задачі

- **Перша задача**: SDD-2 (вже розписана у `SPEC_KIT_WORKFLOW_GUIDE.md`).
- **Друга задача**: SDD-5 (Favorites) — приємне практикування `/clarify` (нюанси optimistic UI).
- **Третя задача**: SDD-3 (Comments) — серйозний `/clarify` через permissions.
- **Челенж**: SDD-7 (Activity Log) — спека буде складна, але `/analyze` врятує.

---

## Як здавати роботу

Для кожної задачі:

1. ✅ Окрема git-branch `<task-id>-<slug>` (наприклад, `SDD-2-search-filter`).
2. ✅ Папка `specs/<NNN>-<slug>/` із усіма артефактами (`spec.md`, `plan.md`, `tasks.md`, `research.md`, `data-model.md`, `contracts/`, `quickstart.md`, `checklists/`).
3. ✅ Реалізація — backend + frontend + тести.
4. ✅ PR з description, що посилається на `specs/<NNN>-<slug>/quickstart.md` як «Definition of Done».
5. ✅ Усі задачі у `tasks.md` помічені `[X]`.
6. ✅ Constitution Check у `plan.md` пройдений (або обґрунтовано в Complexity Tracking).
7. ✅ `/speckit.analyze` без CRITICAL/HIGH findings.

---

> 🚀 **Готові починати?** Відкрийте `TASKS_JIRA.md` для детальних acceptance criteria і subtask-ів першої задачі.
