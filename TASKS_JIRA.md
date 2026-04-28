# TASKS_JIRA — Детальний бекенд у форматі Jira

> Ті самі 7 задач, що й у `TASKS.md`, але у форматі Jira-сторі: Acceptance Criteria (Gherkin), Subtasks з checkboxes, Story Points, Labels, Technical Notes.

Цей формат — підкерівні документи, які можна копіювати у Jira/Linear. Acceptance Criteria — це готові тест-сценарії; subtasks мапляться на checklist-и в `tasks.md`.

---

## SDD-1 — Tags для items (M2M)

**Type**: Story · **SP**: 5 · **Priority**: Medium · **Labels**: `backend`, `frontend`, `db-migration`, `feature`

### Description

Authenticated users can tag items with multiple tags. Tags are unique per user (two users can have the same tag name independently). Items list supports filtering by a single tag via click on the badge.

### Acceptance Criteria

```gherkin
GIVEN I am logged in as user A
WHEN I create a tag named "urgent" via POST /api/v1/tags
THEN the tag is created with my owner_id
AND user B can independently create their own "urgent" without conflict

GIVEN I have an item with no tags
WHEN I PATCH /api/v1/items/{id} with tag_ids: ["uuid-1", "uuid-2"]
THEN the item now references both tags
AND GET /api/v1/items/{id} returns tags: [{id, name}, ...]

GIVEN items A and B exist; A has tag "urgent", B has tag "later"
WHEN I GET /api/v1/items?tag=urgent
THEN only A is returned
AND total: 1

GIVEN I am viewing the items list
WHEN I click on a tag badge for "urgent" on item A
THEN the URL updates to ?tag=urgent
AND the list filters to only items tagged "urgent"
```

### Subtasks

- [ ] **[BE]** Create `Tag` model (id UUID, name str max 50 chars, owner_id FK to User)
- [ ] **[BE]** Add unique constraint on `(name, owner_id)`
- [ ] **[BE]** Create `ItemTag` link table (item_id FK, tag_id FK, composite PK)
- [ ] **[BE]** Write Alembic migration `xxxx_add_tags.py`
- [ ] **[BE]** Add `tags` relationship on `Item` model (read-only)
- [ ] **[BE]** Add CRUD endpoints `/api/v1/tags`: list, create, delete
- [ ] **[BE]** Add `tag_ids: list[UUID]` to `ItemUpdate` schema
- [ ] **[BE]** Add `tag` query param to `GET /api/v1/items`
- [ ] **[BE]** pytest: M2M assignment, unique constraint, filter by tag
- [ ] **[FE]** `TagBadge` component (color hash from name, size sm/md)
- [ ] **[FE]** `TagSelector` component (multiselect, autocomplete from existing user tags)
- [ ] **[FE]** Integrate `TagSelector` in `ItemForm` (create + edit)
- [ ] **[FE]** Render `TagBadge`-s in items list and item detail
- [ ] **[FE]** Click on badge → updates `?tag=` URL param + filters list
- [ ] **[FE]** Playwright e2e: tag CRUD + filter

### Technical Notes

- Use `sqlalchemy.UniqueConstraint('name', 'owner_id', name='uq_tag_name_per_owner')`.
- Color hash: deterministic from `name` via `crc32` mod 8 → palette of 8 tailwind colors.
- Filter by tag preserves `?q=` (compose with SDD-2 if both shipped).

---

## SDD-2 — Search & Filter

**Type**: Story · **SP**: 3 · **Priority**: High · **Labels**: `backend`, `frontend`, `feature`

### Description

User can search items by partial match on `title`. Debounced 300ms. URL state via `?q=`. Counter "Found N items" above the table. Clear via X button or Esc.

### Acceptance Criteria

```gherkin
GIVEN items with titles "Test Report" and "Monthly Summary" exist
WHEN I type "test" in the search field
AND wait 300ms
THEN only "Test Report" is shown in the table
AND "Found 1 item" is displayed
AND the URL is updated to ?q=test

GIVEN I have entered a search query
WHEN I type rapidly
THEN the API request is sent only 300ms after the last keystroke

GIVEN the search field has text "test"
WHEN I press Esc
THEN the field is cleared
AND all items are shown
AND the URL no longer contains ?q=

GIVEN I navigate to /items?q=test
WHEN the page loads
THEN the search field is pre-filled with "test"
AND the items list is filtered

GIVEN the query is "xyz" with no matches
WHEN the request completes
THEN an empty state "No items match 'xyz'" is shown with a Clear button
```

### Subtasks

- [ ] **[BE]** Add `q: Optional[str] = Query(None, max_length=200)` to `GET /api/v1/items`
- [ ] **[BE]** Implement `_escape_ilike` helper for `%`, `_`, `\`
- [ ] **[BE]** Apply `WHERE title ILIKE :q` when `q` provided
- [ ] **[BE]** pytest: case-insensitive match, escape special chars, empty q returns all
- [ ] **[FE]** `useDebouncedValue<T>(value: T, delay: number): T` hook
- [ ] **[FE]** `SearchBar` component (input, X button, count display)
- [ ] **[FE]** Integrate with `useSearchParams()` for URL sync
- [ ] **[FE]** TanStack Query: `queryKey: ['items', { q, page }]`
- [ ] **[FE]** `EmptyState` component
- [ ] **[FE]** Esc key handler in SearchBar
- [ ] **[FE]** Playwright e2e: type+wait+verify, Esc, URL pre-fill, empty state

### Technical Notes

- ILIKE without index acceptable for ≤10K rows (Constitution Performance принцип задокументовано в Complexity Tracking).
- Future: `pg_trgm` GIN index in v2 (out of scope here).
- Counter must show "Found N item" (singular) when N=1.

---

## SDD-3 — Comments з permissions

**Type**: Story · **SP**: 8 · **Priority**: High · **Labels**: `backend`, `frontend`, `db-migration`, `auth`, `feature`

### Description

Authenticated users can comment on items. Comments are sorted by `created_at DESC`, paginated (cursor-based). Item owner can delete any comment; comment author can delete own. Markdown rendering with safe sanitization.

### Acceptance Criteria

```gherkin
GIVEN I am logged in as user A
WHEN I POST /api/v1/items/{id}/comments with body "Hello"
THEN a comment is created with author_id=A
AND it appears in GET /api/v1/items/{id}/comments?limit=20

GIVEN comment C with author B exists on item I owned by A
WHEN user A (item owner) DELETEs comment C
THEN it succeeds (200)
WHEN user X (other) tries to DELETE comment C
THEN 403 Forbidden

GIVEN comment C with author B exists
WHEN user B DELETEs comment C
THEN it succeeds (200)

GIVEN 50 comments exist
WHEN I GET ?cursor=&limit=20
THEN 20 comments returned with next_cursor
WHEN I GET ?cursor=<next_cursor>&limit=20
THEN next 20 returned

GIVEN I submit a new comment
WHEN the request is in flight
THEN the UI shows the comment immediately (optimistic)
WHEN the request fails
THEN the comment is removed and an error toast appears
```

### Subtasks

- [ ] **[BE]** Create `Comment` model (id UUID, item_id FK, author_id FK, body text, created_at)
- [ ] **[BE]** Alembic migration with index on `(item_id, created_at DESC)`
- [ ] **[BE]** Pydantic schemas: `CommentCreate`, `CommentRead` (with author summary)
- [ ] **[BE]** Cursor-based pagination helper
- [ ] **[BE]** Endpoints: `GET /items/{id}/comments`, `POST`, `DELETE /comments/{id}`
- [ ] **[BE]** Permission dependency: `is_item_owner_or_comment_author`
- [ ] **[BE]** pytest: 403 for non-owner non-author, 200 for both eligible roles
- [ ] **[BE]** pytest: cursor pagination with 50 fixtures
- [ ] **[FE]** `CommentList` (infinite scroll via TanStack `useInfiniteQuery`)
- [ ] **[FE]** `CommentForm` (textarea, char counter, submit button)
- [ ] **[FE]** Markdown rendering via `react-markdown` + `rehype-sanitize`
- [ ] **[FE]** Optimistic add via `onMutate` + rollback on `onError`
- [ ] **[FE]** Delete button visibility based on `canDelete: bool` from API
- [ ] **[FE]** Playwright e2e: post + delete (own + as item owner) + permission denial

### Technical Notes

- Body max length: 2000 chars (validated server + client).
- `canDelete` is computed server-side per comment for current user (avoids leaking other-user permission state).
- Markdown allowed tags: paragraph, list, code, link, bold, italic. No raw HTML.
- Cursor format: base64(`{ts}_{id}`).

---

## SDD-4 — Dashboard з recharts

**Type**: Story · **SP**: 8 · **Priority**: Medium · **Labels**: `backend`, `frontend`, `feature`, `superuser-only`

### Description

Superuser-only dashboard with 3 visualizations: items per week (line chart), items per tag (top 10 bar chart), top contributors by item count (pie chart). Date range picker (7/30/90 days).

### Acceptance Criteria

```gherkin
GIVEN I am a regular user
WHEN I navigate to /dashboard
THEN I see 403 Forbidden

GIVEN I am a superuser
WHEN I navigate to /dashboard with default range (last 30 days)
THEN I see 3 charts populated with real data
AND the response is < 1 second (cached) or < 3s (cold cache)

GIVEN data is unchanged
WHEN I refresh /dashboard within 5 minutes
THEN cached data is served (verified by header X-Cache: HIT)

GIVEN I change date range to "last 7 days"
WHEN the request fires
THEN charts update with the new period
AND URL updates to ?range=7d
```

### Subtasks

- [ ] **[BE]** SQL: items per week (DATE_TRUNC, GROUP BY week_start)
- [ ] **[BE]** SQL: items per tag (top 10, GROUP BY tag_id)
- [ ] **[BE]** SQL: top contributors (top 5, GROUP BY owner_id)
- [ ] **[BE]** Endpoint `GET /api/v1/dashboard/summary?range=7d|30d|90d`
- [ ] **[BE]** In-memory TTL cache (5 min) keyed by `(range, current_user_id)`
- [ ] **[BE]** Permission: superuser only via existing dependency
- [ ] **[BE]** pytest: data correctness, cache HIT/MISS, 403 for non-superuser
- [ ] **[FE]** `/dashboard` route with role guard
- [ ] **[FE]** Date range picker (segmented control: 7/30/90 days)
- [ ] **[FE]** `LineChartItemsByWeek` (recharts LineChart)
- [ ] **[FE]** `BarChartTopTags` (recharts BarChart, horizontal)
- [ ] **[FE]** `PieChartContributors` (recharts PieChart with legend)
- [ ] **[FE]** Loading skeletons for each chart
- [ ] **[FE]** Empty state per chart
- [ ] **[FE]** URL state for range
- [ ] **[FE]** Playwright e2e: superuser sees charts, regular user 403, range switch updates data

### Technical Notes

- Cache invalidation: TTL only (no manual invalidation in MVP).
- Date range options: hard-coded; custom ranges out of scope.
- Constitution Performance: <1s with cache, <3s cold — document as OK; deferred work to add Redis cache.

---

## SDD-5 — Favorites з optimistic UI

**Type**: Story · **SP**: 5 · **Priority**: Medium · **Labels**: `backend`, `frontend`, `db-migration`, `feature`

### Description

User can star items as favorites. Toggle is optimistic — UI updates immediately, syncs with server. Sidebar shows "My Favorites" filter.

### Acceptance Criteria

```gherkin
GIVEN item I exists; I am not favoriting it
WHEN I POST /api/v1/items/{id}/favorite
THEN a favorite record is created
AND GET /api/v1/items/{id} returns is_favorite: true
AND the response is idempotent (POST twice = single record)

GIVEN I am favoriting item I
WHEN I DELETE /api/v1/items/{id}/favorite
THEN the favorite is removed
AND GET returns is_favorite: false
AND the response is idempotent

GIVEN I have 5 favorites
WHEN I GET /api/v1/items?favorite=true
THEN only those 5 items are returned

GIVEN I click the star icon
WHEN the click is registered
THEN the icon flips to filled state immediately (optimistic)
AND the request fires in background
WHEN the request fails
THEN the icon flips back AND a toast "Could not save favorite" appears
```

### Subtasks

- [ ] **[BE]** Create `Favorite` model (user_id FK, item_id FK, created_at) — composite PK
- [ ] **[BE]** Alembic migration with index `(user_id, created_at DESC)`
- [ ] **[BE]** Endpoints `POST /items/{id}/favorite` (idempotent), `DELETE`
- [ ] **[BE]** Add `is_favorite: bool` computed field on `ItemRead`
- [ ] **[BE]** Add `?favorite=true` filter to list endpoint
- [ ] **[BE]** pytest: idempotency, computed field, filter
- [ ] **[FE]** `StarIcon` component (filled/outlined)
- [ ] **[FE]** `useFavoriteToggle(itemId)` hook with optimistic mutation
- [ ] **[FE]** TanStack `onMutate`: optimistically update cache for `['items']` and `['items', itemId]`
- [ ] **[FE]** `onError` rollback + toast
- [ ] **[FE]** Sidebar tab "My Favorites" → navigates to `/items?favorite=true`
- [ ] **[FE]** Playwright e2e: toggle, persistence, rollback simulation

### Technical Notes

- `is_favorite` computed via outer join with current_user's favorites.
- Idempotency: PostgreSQL `INSERT ... ON CONFLICT DO NOTHING`.
- Rollback simulation in tests: intercept network request and force 500.

---

## SDD-6 — CSV Export через StreamingResponse

**Type**: Story · **SP**: 5 · **Priority**: Medium · **Labels**: `backend`, `frontend`, `feature`, `performance`

### Description

User can export the current items list to CSV. Backend streams large datasets without loading into memory. Filters from the list (`q`, `tag`, `favorite`) are preserved.

### Acceptance Criteria

```gherkin
GIVEN I have 50,000 items in my list
WHEN I GET /api/v1/items/export?format=csv
THEN the response starts streaming within 500ms
AND backend memory peak is < 50MB during export
AND the file downloads with name items-2026-04-28.csv

GIVEN I have ?q=test&favorite=true active
WHEN I click Export
THEN only items matching those filters appear in the CSV

GIVEN the CSV content
WHEN I open it in Excel
THEN headers are: id, title, description, owner_email, tags, is_favorite, created_at
AND tags are comma-separated within the cell, properly escaped
```

### Subtasks

- [ ] **[BE]** Implement `csv_stream_generator(query)` async generator
- [ ] **[BE]** Endpoint `GET /api/v1/items/export?format=csv`
- [ ] **[BE]** Query params: `q`, `tag`, `favorite` — same filters as list
- [ ] **[BE]** Headers: `Content-Type: text/csv; charset=utf-8`, `Content-Disposition`
- [ ] **[BE]** Stream chunks of 1000 rows
- [ ] **[BE]** pytest: 50K rows, memory profiling via `tracemalloc`
- [ ] **[BE]** pytest: filter integration (q + tag + favorite combined)
- [ ] **[FE]** "Export" button in items toolbar
- [ ] **[FE]** Construct URL with current filters
- [ ] **[FE]** Trigger download via `<a download>` or `window.location`
- [ ] **[FE]** Loading spinner while download initiates (3s timeout)
- [ ] **[FE]** Playwright e2e: filtered export, file content sanity

### Technical Notes

- StreamingResponse чат: `media_type="text/csv"`, content via async generator.
- Use `csv.writer` with `io.StringIO()` per chunk to handle escaping.
- Filename uses UTC date.
- For datasets > 100K rows, recommend background job + email — out of scope.

---

## SDD-7 — Activity Log (audit middleware)

**Type**: Story · **SP**: 13 · **Priority**: Medium · **Labels**: `backend`, `frontend`, `db-migration`, `audit`, `compliance`, `feature`

### Description

Every mutating operation (POST/PUT/PATCH/DELETE on `/api/v1/items`, `/comments`, `/tags`, `/favorites`) is logged with actor, action, resource, payload diff. Admin page surfaces logs with filters and detail diff modal.

### Acceptance Criteria

```gherkin
GIVEN I PATCH /api/v1/items/{id} with title change
WHEN the request completes successfully
THEN an ActivityLog record is created with:
  - actor_id: my user_id
  - action: "PATCH"
  - resource_type: "Item"
  - resource_id: {id}
  - payload_diff: { title: { from: "old", to: "new" } }
  - ip_address: my IP
  - timestamp: UTC now

GIVEN the audit write fails
WHEN the main request succeeds
THEN the user request still returns 200
AND a structured error is logged

GIVEN I am a superuser
WHEN I GET /api/v1/activity?actor=<id>&from=2026-04-01&to=2026-04-30
THEN paginated activity logs are returned

GIVEN I click on an activity row
WHEN the detail modal opens
THEN the payload_diff is rendered as a colored unified-diff view

GIVEN the activity table has 1M rows partitioned by month
WHEN I query a single month range
THEN the query plan uses partition pruning (verified by EXPLAIN ANALYZE)
```

### Subtasks

- [ ] **[BE]** Create `ActivityLog` model (id, actor_id, action, resource_type, resource_id, payload_diff JSONB, ip, ts)
- [ ] **[BE]** Alembic migration with **partitioning by month** (`PARTITION BY RANGE (ts)`)
- [ ] **[BE]** Job to create next-month partition automatically (cron in CI or pg_partman)
- [ ] **[BE]** FastAPI `BaseHTTPMiddleware` that wraps mutating routes
- [ ] **[BE]** Diff helper: load resource before, serialize after, compute JSON diff
- [ ] **[BE]** Async write via `BackgroundTasks` (do not block response)
- [ ] **[BE]** Endpoint `GET /api/v1/activity?actor=&resource_type=&from=&to=&cursor=`
- [ ] **[BE]** Permission: superuser only
- [ ] **[BE]** pytest: each mutating endpoint produces correct log
- [ ] **[BE]** pytest: failed audit write does NOT fail user request
- [ ] **[BE]** pytest: query performance on 1M rows (use seed + EXPLAIN)
- [ ] **[FE]** Route `/admin/activity` with superuser guard
- [ ] **[FE]** Filter panel: actor select, resource type select, date range
- [ ] **[FE]** Cursor-based paginated table
- [ ] **[FE]** Detail modal with `react-diff-viewer-continued`
- [ ] **[FE]** Export filtered logs to CSV (reuse SDD-6 pattern)
- [ ] **[FE]** Playwright e2e: filter + open detail + verify diff content

### Technical Notes

- Diff library: implement custom (200 LoC) or use `dictdiffer`. Watch out for large blobs (truncate fields > 10KB).
- `BackgroundTasks` for async writes — but for high-throughput environments consider a queue (Redis Streams, NATS) — out of scope for MVP.
- Partitioning rationale: 1M+ rows per month possible in real systems; partition pruning saves > 90% scan time.
- Compliance angle (if relevant to your org): retention policy (e.g., 12 months) — add as separate sub-feature.

### Constitution Check

This feature touches Observability и Performance principles. Document in `plan.md`:

- ✅ Observability: structured log per audit write attempt
- ⚠️ Performance: synchronous diff computation may add 5–20ms per request — measure in load test
- ⚠️ Simplicity: middleware adds complexity — justified by compliance requirement (document in Complexity Tracking)

---

## Підсумок

Усі задачі написані з врахуванням:
- ✅ Готових Acceptance Criteria у форматі Gherkin (= тест-сценарії)
- ✅ Subtask checklist'ів, які мапляться на `tasks.md` (фази Setup/Foundational/US/Polish)
- ✅ Technical Notes з нюансами реалізації
- ✅ Constitution touchpoints — де треба обґрунтовувати compliance

**Наступний крок**: оберіть **SDD-2** і пройдіть її повністю по `SPEC_KIT_WORKFLOW_GUIDE.md`. Після цього переходьте до решти.
