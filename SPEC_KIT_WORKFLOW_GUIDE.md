# SPEC-KIT WORKFLOW GUIDE — End-to-End Walkthrough of SDD-2

> This document shows the **full spec-kit cycle** on a single task: **SDD-2: Search & Filter** (search items by `title` field with debounce and a counter).

We will go through, in order: `/speckit.constitution` → `/speckit.specify` → `/speckit.clarify` → `/speckit.plan` → `/speckit.tasks` → `/speckit.analyze` → `/speckit.implement`.

After each command — an explanation of *what happened*, *which artifact appeared*, *what to pay attention to*.

---

## 0. Prerequisites

- ✅ You have completed `PROJECT_SETUP.md` — the training project is up and running.
- ✅ Spec-kit is initialized (`specify init . --integration <agent>`).
- ✅ Slash commands work in your editor.

We are inside the `full-stack-fastapi-template/` directory on the `main` branch.

---

## Step 1 — `/speckit.constitution` (once per project)

Any real project starts with a constitution. If one already exists — skip this step.

### Prompt

```
/speckit.constitution Create principles for our training project:
1. Test-First for backend (pytest, minimum 70% coverage on new modules)
2. Type safety: Python 3.11+ with pydantic, TypeScript strict in frontend
3. API-First: every new endpoint must have an OpenAPI schema before code
4. Observability: structured logging via structlog, trace IDs in requests
5. Performance: backend p95 < 300ms, frontend FCP < 1.5s
6. Simplicity: do not add dependencies that can be replaced by stdlib

Governance:
- MUST-level principles change only via RFC + 1 review
- Constitution version: semver (MAJOR breaking, MINOR new principle, PATCH wording)
- Compliance is checked in /speckit.plan via the Constitution Check gate
```

### What the agent generates

The file `.specify/memory/constitution.md`:

```markdown
---
version: 1.0.0
ratification_date: 2026-04-28
last_amended_date: 2026-04-28
---

<!-- Sync Impact Report
Modified: NEW (initial ratification)
Added principles: 6 (Test-First, Type Safety, API-First, Observability,
                    Performance, Simplicity)
Templates needing update: spec-template.md ✓, plan-template.md ✓,
                          tasks-template.md ✓
-->

# Project Constitution

## Principle 1: Test-First (NON-NEGOTIABLE)
All new backend modules MUST ship with pytest tests. Coverage threshold:
70% for new code, measured by `pytest-cov`. Tests live in `backend/tests/`
mirroring source structure.

**Rationale**: Tests are executable specifications. Without them spec-driven
generation cannot validate correctness.

## Principle 2: Type Safety (MUST)
...

## Governance

Amendments to MUST principles require an RFC document and one approval.
Constitution Check gate runs in /speckit.plan before Phase 0 and after Phase 1.
Violations must be either resolved or justified in plan.md → Complexity Tracking.
```

### What to pay attention to

- The `Sync Impact Report` in the HTML comment at the top — it hints which templates the update may have affected.
- `version` — bumped on every `/speckit.constitution`.
- Each principle has a **rationale** and a **rule** (not generic slogans).

> 💡 **Practical rule**: the constitution is a constraints document, not marketing. If a principle cannot be checked in `Constitution Check`, it is unnecessary.

---

## Step 2 — `/speckit.specify` (Describing the Feature)

### Prompt

```
/speckit.specify Implement search and filtering for items.

In the items list the user sees a search field. When they type —
the list is filtered by the title field (partial match, case-insensitive).

The search has a 300ms debounce — the API request is sent only after
typing stops. Above the table the text "Found N items" appears
where N is the number of results.

The user can clear the search via an "X" button or by pressing Esc.

When the field is empty — all items are shown (as today). When there
are no results — an empty state with a message is shown.
```

### What the agent generates

It creates:
- branch `002-search-and-filter` (via a hook)
- the directory `specs/002-search-and-filter/`
- the file `specs/002-search-and-filter/spec.md`
- the file `specs/002-search-and-filter/checklists/requirements.md`
- updates `.specify/feature.json`

### Example of the generated `spec.md`

```markdown
# Feature Specification: Search and Filter Items

## Overview
Allow authenticated users to search through their items list by partial
match on the `title` field. Results update with a 300ms debounce; total
count is shown above the table.

## User Stories

### Story 1: Real-time search by title (Priority: P1)
**As an** authenticated user
**I want to** type into a search field and see filtered items
**So that** I can find specific items quickly without scrolling.

**Acceptance Criteria:**
- AC-1.1: When the user types in the search field and stops typing
  for 300ms, the items list filters by title.
- AC-1.2: Matching is case-insensitive and substring-based.
- AC-1.3: A counter "Found N items" is displayed above the list.
- AC-1.4: Empty input shows all items (no filter).

### Story 2: Clear search (Priority: P2)
**As an** authenticated user
**I want to** clear the search field with one click or Esc key
**So that** I can quickly reset the view to all items.

**Acceptance Criteria:**
- AC-2.1: A clear button "X" appears in the field while text is entered.
- AC-2.2: Clicking "X" empties the field and shows all items.
- AC-2.3: Pressing Esc has the same effect.

### Story 3: Empty state (Priority: P3)
**As an** authenticated user
**I want to** see a meaningful empty state when no items match
**So that** I understand the search is working but nothing matched.

**Acceptance Criteria:**
- AC-3.1: When N=0, the table is replaced by an empty-state component.
- AC-3.2: The empty state shows: "No items match '<query>'" + a Clear button.

## Functional Requirements

- **FR-001**: System MUST support full-text substring matching on `title`.
- **FR-002**: System MUST debounce search-triggered API calls by 300ms.
- **FR-003**: System MUST display the total count of matching items.
- **FR-004**: System MUST preserve filter state across pagination.
  [NEEDS CLARIFICATION: should filter persist across page reload?]

## Success Criteria

- **SC-001**: User can find a known item in a list of 100 in < 3 seconds.
- **SC-002**: Search response time < 500ms p95 for lists up to 10,000 items.
- **SC-003**: No more than 1 API call per typing burst (verified by network log).

## Edge Cases
- Special characters in query (`%`, `_`, SQL wildcards) MUST be escaped.
- Whitespace-only query treated as empty.
- Very long queries (>200 chars) MUST be truncated client-side.

## Key Entities

- **Item** (existing entity): we use only the existing `title` field.

## Assumptions

- Existing pagination on `/api/v1/items` is preserved.
- No need for fuzzy matching (typo tolerance) in this iteration.
- Search results respect existing per-user data isolation.

## Out of Scope

- Search by other fields (description, tags) — separate feature.
- Saved searches / search history.
- Server-side full-text indexing (PostgreSQL `tsvector`).
```

### `checklists/requirements.md` (auto-generated)

```markdown
# Requirements Quality Checklist

- [ ] No implementation details in spec
- [ ] Requirements are testable and unambiguous
- [ ] Success criteria are measurable and technology-agnostic
- [ ] Edge cases identified
- [ ] User stories have explicit priorities
- [ ] No more than 3 [NEEDS CLARIFICATION] markers
- [ ] All entities reference existing or define new ones
- [ ] Out-of-scope items listed
```

### What to pay attention to

- 🚫 The spec has **no** mention of FastAPI, React, or SQL — only `title`, `items list`, and `endpoint` in general terms.
- ✅ There is one `[NEEDS CLARIFICATION: should filter persist across page reload?]` marker — `/clarify` below will handle it.
- ✅ Success criteria are measurable (`<3 seconds`, `<500ms p95`).

---

## Step 3 — `/speckit.clarify` (Reducing Ambiguity)

### Prompt

```
/speckit.clarify
```

### What the agent does

It scans the spec across 11 ambiguity categories. It finds that:

1. It is not defined whether the filter persists on reload (from `[NEEDS CLARIFICATION]`).
2. It is not defined how to behave when focus is lost during the debounce.
3. It is not defined whether this affects Story 3 (empty state) for a whitespace-only query.

It asks **one question at a time**:

```
**Q1**: Should the search query persist across page reload?

| Option | Description |
|--------|-------------|
| A *(Recommended)* | URL query param ?q= — bookmarkable, sharable |
| B | localStorage — persists per browser, not shareable |
| C | sessionStorage — only for current tab |
| D | Don't persist — clear on reload |

Reason for A: bookmarkability is a common UX expectation for searchable
lists; URL approach is also testable.

Reply: Q1: A
```

The user replies `Q1: A`. The agent immediately updates `spec.md` — adding a `## Clarifications` section and propagating the answer into Functional Requirements:

```markdown
## Clarifications

### Session 2026-04-28
- Q: Should the search query persist across page reload?
  → A: URL query param `?q=` (bookmarkable)
- Q: What should happen if focus is lost during the 300ms debounce?
  → A: Pending request fires immediately on blur
- Q: Whitespace-only query — treat as empty or as literal?
  → A: Treat as empty (trim before debounce)
```

And in the Functional Requirements section:

```diff
- **FR-004**: System MUST preserve filter state across pagination.
-   [NEEDS CLARIFICATION: should filter persist across page reload?]
+ **FR-004**: System MUST preserve filter state across pagination
+   AND across page reload via URL query param `?q=<query>`.
+ **FR-005**: System MUST flush pending debounce on blur (immediate fire).
+ **FR-006**: System MUST trim whitespace before processing query.
```

### What to pay attention to

- Each answer goes **immediately** into spec.md (atomic overwrite).
- The `[NEEDS CLARIFICATION]` marker is gone.
- Rationales are propagated into FRs — nothing is left only in `## Clarifications`.

> 💡 **Practical rule**: never give a `Custom` answer longer than 5 words. If a long answer is needed, the question is poorly worded and the spec needs a manual rework.

---

## Step 4 — `/speckit.plan` (Technical Plan)

### Prompt

```
/speckit.plan
Backend: extend the existing FastAPI endpoint GET /api/v1/items
with a `q: Optional[str]` parameter. Implementation via SQLModel:
WHERE LOWER(title) LIKE LOWER(:q). PostgreSQL ILIKE for case-insensitivity.

Frontend: React component SearchBar with useDebouncedValue (custom hook,
300ms). Integration with TanStack Query via queryKey ['items', q, page].
URL sync via react-router useSearchParams.

Tests:
- pytest: test ILIKE query and escape special chars
- Playwright: e2e scenario typing and verifying the counter

No additional libraries. Do not use PostgreSQL tsvector in this MVP.
```

### What the agent generates

It creates:
- `specs/002-search-and-filter/plan.md`
- `specs/002-search-and-filter/research.md`
- `specs/002-search-and-filter/data-model.md`
- `specs/002-search-and-filter/contracts/items-api.yaml`
- `specs/002-search-and-filter/quickstart.md`
- Updates `CLAUDE.md` (between `<!-- SPECKIT START -->` and `<!-- SPECKIT END -->`)

### Example `plan.md`

```markdown
# Implementation Plan: Search and Filter Items

## Technical Context
- **Language/Version**: Python 3.11, TypeScript 5.3
- **Primary Dependencies**: FastAPI 0.111, SQLModel 0.0.14, PostgreSQL 16,
  React 18, TanStack Query 5, react-router 6
- **Storage**: existing PostgreSQL (no new tables)
- **Testing**: pytest 8 + pytest-asyncio + Playwright 1.42
- **Target Platform**: Docker Compose (backend, frontend, db, traefik)
- **Project Type**: Full-stack web app
- **Performance Goals**: <500ms p95 for /items?q=...
- **Constraints**: no new dependencies, no tsvector indexing in MVP
- **Scale/Scope**: lists up to 10K items per user

## Constitution Check (Phase 0 — Pre-Research)
| Principle | Compliance | Notes |
|-----------|-----------|-------|
| Test-First | ✅ | pytest covers ILIKE query, Playwright for e2e |
| Type Safety | ✅ | pydantic models for query, TS interfaces |
| API-First | ✅ | OpenAPI updated in contracts/items-api.yaml |
| Observability | ✅ | structured log entry per query |
| Performance | ⚠️ | 10K rows × ILIKE without index = ~80ms locally; |
|             |    | acceptable for MVP, document follow-up |
| Simplicity | ✅ | reuses existing endpoint |

## Project Structure
```
backend/app/api/v1/items.py            # add q param to list_items
backend/app/services/item_service.py   # ILIKE filter logic
backend/tests/api/test_items_search.py # new
frontend/src/components/SearchBar.tsx  # new
frontend/src/hooks/useDebouncedValue.ts # new
frontend/src/routes/items/index.tsx    # integrate SearchBar, useSearchParams
frontend/tests/e2e/search.spec.ts      # new
```

## Constitution Check (Phase 1 — Post-Design)
| Principle | Compliance | Notes |
|-----------|-----------|-------|
| Test-First | ✅ | tests defined before implementation |
| ... | ... | ... |

## Complexity Tracking

| Item | Complexity | Justification |
|------|-----------|---------------|
| ILIKE without index on `title` | LOW | acceptable for 10K rows; |
|                                  | follow-up: add `pg_trgm` GIN index in v2 |
```

### Example `research.md` (excerpt)

```markdown
# Research

## Decision: ILIKE vs full-text search (tsvector)
- **Decision**: ILIKE for MVP
- **Rationale**: tsvector requires migration + GIN index; for ≤10K rows,
  ILIKE is <100ms locally. Per Constitution Principle 6 (Simplicity),
  we avoid premature optimization.
- **Alternatives considered**:
  - PostgreSQL `tsvector` + GIN index — better for 100K+ rows, deferred to v2
  - External Elasticsearch — overkill, adds infra cost
  - In-memory client filter — fails at >1K items, breaks pagination

## Decision: Debounce hook (custom vs library)
- **Decision**: custom `useDebouncedValue`
- **Rationale**: 8 lines of code, avoids `use-debounce` dependency.
  Per Constitution: no new deps if stdlib/custom suffices.
- **Alternatives**: `use-debounce`, `lodash.debounce` — rejected.

## Decision: URL state — react-router useSearchParams vs nuqs
- **Decision**: useSearchParams (built into react-router 6)
- **Rationale**: zero new deps; nuqs is more ergonomic but unnecessary here.
- **Alternatives**: nuqs (deferred to wider state-management RFC).
```

### Example `data-model.md`

```markdown
# Data Model — Search & Filter

## Entities Used (Existing)

### Item
- `id: UUID`
- `title: str`  ← search target
- `description: str | None`
- `owner_id: UUID` (FK to User)

No schema changes required.

## Validation Rules
- Search query `q`: max 200 chars, trimmed, special chars escaped server-side.
```

### Example `contracts/items-api.yaml`

```yaml
paths:
  /api/v1/items:
    get:
      operationId: list_items
      parameters:
        - name: q
          in: query
          schema:
            type: string
            maxLength: 200
          description: Substring match on title (case-insensitive)
        - name: page
          in: query
          schema: { type: integer, default: 1 }
        - name: page_size
          in: query
          schema: { type: integer, default: 20 }
      responses:
        '200':
          description: Paginated items list
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ItemsPage'
components:
  schemas:
    ItemsPage:
      type: object
      properties:
        data: { type: array, items: { $ref: '#/components/schemas/Item' } }
        total: { type: integer }
        page: { type: integer }
        page_size: { type: integer }
```

### Example `quickstart.md`

```markdown
# Quickstart — Manual Validation

1. Login as admin@example.com.
2. Create 5 items with titles: "Test Report", "Monthly Summary",
   "Test Coverage", "Quarterly Update", "Test Plan".
3. Navigate to /items.
4. Type "test" in search field, wait 300ms.
   ✅ Expect 3 items shown: "Test Report", "Test Coverage", "Test Plan".
   ✅ Expect counter "Found 3 items".
   ✅ Expect URL contains `?q=test`.
5. Press Esc.
   ✅ Expect all 5 items, counter hidden, URL `?q=` removed.
6. Type "xyz".
   ✅ Expect empty state "No items match 'xyz'" + Clear button.
7. Click Clear button.
   ✅ Expect all 5 items.
8. Reload page with URL `/items?q=test`.
   ✅ Expect search field pre-filled with "test", filtered list, counter shown.
```

### What to pay attention to

- **Constitution Check** runs twice (Phase 0 and Phase 1). It flagged `Performance ⚠️` for ILIKE without an index — documented in Complexity Tracking.
- **research.md** — every decision has Decision / Rationale / Alternatives. This blocks impulsive choices.
- **contracts/** — the OpenAPI is updated *before* any code is written.
- **quickstart.md** — this is the "definition of done" for a human. If QA / PM wants to verify by hand — they start here.

> 💡 **Tip**: after `/plan`, ask the agent: "Look at the plan and find areas that need additional research on current library versions. Run parallel research tasks and update `research.md`".

---

## Step 5 — `/speckit.tasks` (Decomposition)

### Prompt

```
/speckit.tasks
```

(No arguments — the command reads plan.md, spec.md, contracts/, data-model.md.)

### What it generates

The file `specs/002-search-and-filter/tasks.md`:

```markdown
# Tasks — Search and Filter

## Phase 1 — Setup
- [ ] T001 [P] Add Playwright dev dependency to frontend/package.json
- [ ] T002 [P] Create frontend/playwright.config.ts with baseURL=http://localhost:5173

## Phase 2 — Foundational (BLOCKS all stories)
- [ ] T003 Update OpenAPI in backend/app/api/v1/items.py:
       add `q: Optional[str] = Query(None, max_length=200)` to list_items
- [ ] T004 Add `_escape_ilike` helper in backend/app/services/item_service.py:
       escape `%`, `_`, `\` characters in user input
- [ ] T005 Update `ItemService.list_items` in backend/app/services/item_service.py:
       apply WHERE title ILIKE :q when q provided

## Phase 3 — User Story 1 (P1): Real-time search by title
- [ ] T006 [P] [US1] Add pytest test:
       backend/tests/api/test_items_search.py::test_list_items_filters_by_title
- [ ] T007 [P] [US1] Add pytest test:
       backend/tests/api/test_items_search.py::test_search_is_case_insensitive
- [ ] T008 [P] [US1] Add pytest test:
       backend/tests/api/test_items_search.py::test_special_chars_are_escaped
- [ ] T009 [US1] Implement frontend useDebouncedValue hook:
       frontend/src/hooks/useDebouncedValue.ts (delay param, returns debounced value)
- [ ] T010 [US1] Implement SearchBar component:
       frontend/src/components/SearchBar.tsx (input + count display + clear button)
- [ ] T011 [US1] Integrate SearchBar in ItemsList page:
       frontend/src/routes/items/index.tsx (wire up useSearchParams, TanStack Query key)
- [ ] T012 [P] [US1] Playwright e2e test:
       frontend/tests/e2e/search.spec.ts::"finds items by partial title match"

## Phase 4 — User Story 2 (P2): Clear search
- [ ] T013 [US2] Add clear button (X icon) in SearchBar.tsx
- [ ] T014 [US2] Add Esc key handler in SearchBar.tsx
- [ ] T015 [P] [US2] Playwright e2e test:
       frontend/tests/e2e/search.spec.ts::"clears via X button and Esc"

## Phase 5 — User Story 3 (P3): Empty state
- [ ] T016 [US3] Add EmptyState component:
       frontend/src/components/EmptyState.tsx (with Clear button slot)
- [ ] T017 [US3] Wire EmptyState in ItemsList when total=0 and q!=""
- [ ] T018 [P] [US3] Playwright e2e test:
       frontend/tests/e2e/search.spec.ts::"shows empty state for no matches"

## Phase 6 — Polish & Cross-Cutting
- [ ] T019 [P] Add structured log entry for search queries:
       backend/app/services/item_service.py (log query length + result count)
- [ ] T020 [P] Update README.md frontend section with search demo screenshot
- [ ] T021 Run full test suite, verify 70%+ coverage on changed files
- [ ] T022 Update OpenAPI snapshot in CHANGELOG.md
```

### Structural details

- **Phase 2 (Foundational)** blocks all stories. Without it, the frontend cannot call the API with `q`.
- **Story 1 (P1)** = MVP. Once Phase 3 is finished, you already have a working search feature. Stories 2 and 3 are UX improvements.
- `[P]` markers — tasks that can run in parallel (different files).
- Every task has the **exact file path**.

### What to pay attention to

> ⚠️ **Do not edit the format by hand**. Spec-kit strictly parses `[ ] T001`, `[P]`, `[US1]`. If you break it — `/implement` will skip the task.

---

## Step 6 — `/speckit.analyze` (Cross-Artifact Audit)

### Prompt

```
/speckit.analyze
```

### What it returns

```markdown
# Cross-Artifact Analysis Report

## Findings

| ID | Severity | Category | Description |
|----|----------|----------|-------------|
| F-001 | LOW | Coverage | FR-005 (flush debounce on blur) — no explicit |
|       |     |          | task; covered implicitly in T009 |
| F-002 | MEDIUM | Coverage | SC-002 (<500ms p95) — no perf test in tasks.md |
| F-003 | LOW | Ambiguity | "Special chars escaped" — escape implementation |
|       |     |           | not specified; resolved in research.md but not |
|       |     |           | linked from spec.md |

## Coverage Summary

| Requirement | Covered by Task(s) |
|-------------|--------------------|
| FR-001 (substring match, case-insens) | T005 |
| FR-002 (debounce 300ms) | T009 |
| FR-003 (counter) | T010 |
| FR-004 (URL state) | T011 |
| FR-005 (flush on blur) | (implicit T009) ⚠️ |
| FR-006 (trim whitespace) | (none) ❌ |
| SC-001 (find in <3s) | T012 (timed test) ✅ |
| SC-002 (<500ms p95) | (none) ❌ |
| SC-003 (1 API call per burst) | T012 |

## Constitution Alignment Issues
None.

## Unmapped Tasks
None.

## Metrics
- Total requirements: 6 FR + 3 SC = 9
- Total tasks: 22
- Coverage: 7/9 = 78%
- Ambiguity count: 1
- Duplication count: 0
- Critical issues: 0

## Next Actions
1. Add task to cover FR-006 (whitespace trim) — backend or frontend.
2. Add perf test task for SC-002 (e.g., locust load test).
3. Make F-001 explicit by adding test "fires on blur".
```

### What to do with this

- **CRITICAL** = blocker, fix before `/implement`.
- **MEDIUM/LOW** — better fixed, but you can keep going.

In our case we add the following to `tasks.md`:

```markdown
- [ ] T023 [US1] Add backend trim of `q` parameter (strip whitespace) in
       backend/app/api/v1/items.py
- [ ] T024 [P] Add perf test backend/tests/perf/test_items_search_p95.py
       (10K items, p95 < 500ms)
- [ ] T025 [P] [US1] Playwright test "search fires on blur during debounce"
```

Run `/speckit.analyze` again — Coverage is 100%, no Findings.

> 💡 **Practical rule**: `/analyze` is a *free* linter for the specification. Run it every time after significant changes.

---

## Step 7 — `/speckit.implement`

### Prompt

```
/speckit.implement
```

### What the agent does

1. **Checks the checklists/** — `requirements.md` is all ✅, we can proceed.
2. **Updates ignore files** — `.gitignore` needs no changes (already valid for Python+Node).
3. **Executes tasks phase by phase:**

```
[Phase 1] T001, T002 (parallel) — installing playwright...
  → frontend/package.json updated ✅
  → frontend/playwright.config.ts created ✅
  → npm install ✅
  ✓ T001 [X], T002 [X] in tasks.md

[Phase 2] T003, T004, T005 (sequential, share files) — backend changes...
  → backend/app/api/v1/items.py: added q param ✅
  → backend/app/services/item_service.py: added _escape_ilike, ILIKE logic ✅
  → uvicorn reload triggered ✅
  ✓ T003 [X], T004 [X], T005 [X]

[Phase 3] T006-T012 (US1)...
  T006-T008 (parallel) — pytest tests written, running...
  → 3 tests PASS ✅
  T009 — useDebouncedValue.ts written ✅
  T010 — SearchBar.tsx written ✅
  T011 — ItemsList integration ✅
  T012 — playwright test PASSES ✅
  ✓ T006-T012 [X]
  🎉 MVP increment for US1 ready — feature works!

[Phase 4] T013-T015 (US2)...
[Phase 5] T016-T018 (US3)...
[Phase 6] T019-T022 (Polish)...
[Extra] T023-T025 (added by /analyze)...

Final validation against spec.md:
  ✅ FR-001 substring match — covered by T005, T006
  ✅ FR-002 debounce 300ms — covered by T009, verified by T025
  ...
  ✅ SC-002 <500ms p95 — measured in T024: actual 87ms locally

Implementation complete.
```

4. **Marks `[X]` in tasks.md** after each task.

### What to pay attention to

- The agent **stops** if a non-parallel task fails (for example, a test fails after implementation).
- Parallel `[P]` tasks keep running even if one fails — it reports at the end.
- All changes live in the git working tree, **not committed automatically**. The follow-up workflow:

```bash
git status
git add specs/002-search-and-filter backend/ frontend/
git commit -m "feat(items): search and filter

Implements specs/002-search-and-filter/spec.md.

Closes #SDD-2"
git push -u origin 002-search-and-filter
gh pr create --title "feat(items): search and filter" \
  --body "$(cat specs/002-search-and-filter/quickstart.md)"
```

> 💡 **Practical rule**: the PR must contain a link to `specs/<feature>/` as the primary source of truth. Code review starts from spec.md, not from the code diff.

---

## Workflow Summary

| Step | Time | What you got |
|------|------|--------------|
| `/constitution` | 15 min | `.specify/memory/constitution.md` v1.0.0 |
| `/specify` | 10 min | `spec.md` with 3 user stories, 4 FRs, 3 SCs |
| `/clarify` | 10 min | 3 questions → spec extended with `## Clarifications` |
| `/plan` | 20 min | `plan.md`, `research.md`, `data-model.md`, `contracts/`, `quickstart.md` |
| `/tasks` | 5 min | 22 tasks across phases |
| `/analyze` | 5 min | Findings → +3 tasks (T023–T025) |
| `/implement` | 60–120 min | Working code + tests + e2e |
| **Total** | **~3 hr** | A finished feature with full documentation |

> 💡 **Compared to vibe coding**: the same feature could be written in 1 hour "off the top of your head". But:
> - There would be no tests (or fake ones).
> - There would be no URL state (it would be forgotten).
> - There would be no perf test.
> - In a month you would have to figure out "what is going on here" — without spec.md that takes a long time.
> - Code review would be ad-hoc — without quickstart.md and data-model.md the reviewer loses 30 minutes just to gain context.

---

## What's next

- **Practice**: repeat the same cycle for **SDD-1 (Tags)** — slightly higher complexity because of a new table and an M2M.
- **Going deeper**: try **SDD-7 (Activity Log)** — Hard. Requires middleware, an audit trail, and aggregations.
- **Team work**: walk through scenarios 5 (Team Collaboration) and 6 (Greenfield) in `SPEC_KIT_USE_CASES.md`.
