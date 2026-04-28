# TASKS — Hands-on backlog

> 7 tasks on the training project `fastapi/full-stack-fastapi-template`. Run each one through the full spec-kit workflow (Scenario 2 in `SPEC_KIT_USE_CASES.md`). Jira-format details live in `TASKS_JIRA.md`.

---

## SDD-1 — Tags for items (M2M)

**Difficulty**: Easy · **Story Points**: 5

A user can create tags (per-user) and assign multiple tags to a single item. Tags appear in the items list; the list can be filtered by tag.

**Backend:**
- `Tag` model (id UUID, name str unique, owner_id FK)
- Linked table `ItemTag` (item_id FK, tag_id FK)
- Alembic migration
- CRUD endpoints `/api/v1/tags`
- Patch endpoint `/api/v1/items/{id}` accepts `tag_ids: list[UUID]`
- Filter `?tag=<name>` on `GET /api/v1/items`

**Frontend:**
- `TagBadge` component (colored badge)
- `TagSelector` (multiselect with autocomplete)
- Integration into the Item create/edit form
- Filter the items list by clicking a tag

---

## SDD-2 — Search & Filter

**Difficulty**: Easy · **Story Points**: 3

Walked through in `SPEC_KIT_WORKFLOW_GUIDE.md` as the reference example.

**Backend:**
- `q: Optional[str]` parameter on `GET /api/v1/items`
- ILIKE substring match on `title`
- Escape special chars (`%`, `_`, `\`)

**Frontend:**
- `SearchBar` component with a clear button + Esc
- `useDebouncedValue` hook (300ms)
- URL state via `useSearchParams` (`?q=...`)
- Empty state component

---

## SDD-3 — Comments with permissions

**Difficulty**: Medium · **Story Points**: 8

Authenticated users can leave comments under items. The item owner can delete any comment; everyone else can delete only their own.

**Backend:**
- `Comment` model (id, item_id FK, author_id FK, body, created_at)
- Alembic migration
- CRUD endpoints `/api/v1/items/{id}/comments`
- Permission check on DELETE: `is_owner_of_item OR is_author_of_comment`
- Pagination + sort by `created_at DESC`

**Frontend:**
- `CommentList` (paginated, infinite scroll)
- `CommentForm` (markdown preview)
- Delete button visible only on permitted comments
- Optimistic update on add

---

## SDD-4 — Dashboard with recharts

**Difficulty**: Medium · **Story Points**: 8

A dashboard page with aggregations: items per week (line chart), items per tag (bar chart), top contributors (pie chart).

**Backend:**
- `GET /api/v1/dashboard/summary` endpoint with 3 sections
- SQL aggregations (GROUP BY week, GROUP BY tag, GROUP BY owner)
- 5-minute cache (in-memory or Redis)
- Permission: superuser only

**Frontend:**
- `/dashboard` page
- 3 chart components on `recharts`
- Loading/error states
- Date range picker (last 7/30/90 days)

---

## SDD-5 — Favorites with optimistic UI

**Difficulty**: Easy · **Story Points**: 5

Users can star items as favorites. The toggle is instant (optimistic) and syncs to the backend.

**Backend:**
- `Favorite` model (user_id, item_id, created_at) — composite PK
- Alembic migration
- Endpoints `POST /api/v1/items/{id}/favorite`, `DELETE` (idempotent)
- `is_favorite` field in the response of `GET /api/v1/items/{id}` (computed)
- Filter `?favorite=true` on the list endpoint

**Frontend:**
- StarIcon with 2 states
- Optimistic update in TanStack Query (via `onMutate`)
- Rollback on failure
- Sidebar tab "My Favorites"

---

## SDD-6 — CSV Export via StreamingResponse

**Difficulty**: Medium · **Story Points**: 5

An "Export to CSV" button on the items list. The backend streams large datasets without loading them into memory.

**Backend:**
- `GET /api/v1/items/export?format=csv` endpoint
- `StreamingResponse` with `csv.writer`
- Honor filters `?q=`, `?tag=`, `?favorite=`
- `Content-Disposition: attachment; filename=items-YYYY-MM-DD.csv` header
- Test with 50K rows (memory < 50MB)

**Frontend:**
- "Export" button in the toolbar
- Progress indicator during download
- Respects current list filters

---

## SDD-7 — Activity Log (audit middleware)

**Difficulty**: Hard · **Story Points**: 13

Every mutating operation (POST/PUT/PATCH/DELETE) is logged into an audit table with actor, resource, action, and a before/after diff.

**Backend:**
- `ActivityLog` model (id, actor_id, action, resource_type, resource_id, payload_diff JSONB, ip, ts)
- FastAPI middleware: intercepts all mutating routes
- JSON diff for before/after (via `jsondiff` or homegrown)
- Alembic migration with monthly partitioning
- `GET /api/v1/activity?actor=&resource=&from=&to=` endpoint with permissions (superuser only)
- Async write (BackgroundTask) — doesn't block the response

**Frontend:**
- `/admin/activity` page
- Filter panel (date range, actor, resource type)
- Detail modal that shows the diff
- Pagination (cursor-based)

---

## Summary table

| ID | Task | Difficulty | Backend | Frontend |
|----|------|------------|---------|----------|
| SDD-1 | Tags (M2M) | Easy | model, migration, CRUD, filter | badge, multiselect, integration |
| SDD-2 | Search & Filter | Easy | q param, ILIKE, escape | SearchBar, debounce, URL state |
| SDD-3 | Comments | Medium | model, migration, permissions | list, form, optimistic |
| SDD-4 | Dashboard | Medium | aggregations, caching | 3 charts, date range |
| SDD-5 | Favorites | Easy | M2M, computed field | optimistic, rollback |
| SDD-6 | CSV Export | Medium | StreamingResponse, filters | button, progress |
| SDD-7 | Activity Log | Hard | middleware, diff, async | admin page, filters, modal |

---

## How to pick tasks

- **First task**: SDD-2 (already walked through in `SPEC_KIT_WORKFLOW_GUIDE.md`).
- **Second task**: SDD-5 (Favorites) — a nice exercise in `/clarify` (the nuances of optimistic UI).
- **Third task**: SDD-3 (Comments) — a serious `/clarify` thanks to permissions.
- **Challenge**: SDD-7 (Activity Log) — the spec will be complex, but `/analyze` will save you.

---

## How to deliver work

For each task:

1. ✅ A separate git branch `<task-id>-<slug>` (for example, `SDD-2-search-filter`).
2. ✅ A `specs/<NNN>-<slug>/` folder with all artifacts (`spec.md`, `plan.md`, `tasks.md`, `research.md`, `data-model.md`, `contracts/`, `quickstart.md`, `checklists/`).
3. ✅ Implementation — backend + frontend + tests.
4. ✅ A PR with a description that links to `specs/<NNN>-<slug>/quickstart.md` as the "Definition of Done".
5. ✅ Every task in `tasks.md` marked `[X]`.
6. ✅ Constitution Check in `plan.md` passing (or justified in Complexity Tracking).
7. ✅ `/speckit.analyze` clean of CRITICAL/HIGH findings.

---

> 🚀 **Ready to start?** Open `TASKS_JIRA.md` for detailed acceptance criteria and the subtasks of the first task.
