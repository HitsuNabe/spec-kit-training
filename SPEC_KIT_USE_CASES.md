# SPEC-KIT USE CASES — Scenarios by Complexity Levels

> A catalog of 9 spec-kit application scenarios. From the simplest (Beginner) to enterprise patterns. Each scenario is a template for a real situation you will run into at work.

How to read this:

- **Level**: complexity *of applying* the scenario, not the complexity of the task itself.
- **When**: concrete triggers — exactly when to pick this scenario.
- **Time**: a realistic time budget.
- **Commands**: which slash commands you will need.
- **Walkthrough**: a short end-to-end walkthrough.
- **Pitfalls**: what typically breaks.

---

## Scenario 1 — Quick Spec Flow

**Level**: Beginner
**When**: A small isolated feature, a bug fix, or a refactor of a single module. The change is local and the scope is clear.
**Time**: 30–60 min
**Commands**: `/speckit.specify` → `/speckit.plan` → `/speckit.tasks` → `/speckit.implement`

### Walkthrough

```
/speckit.specify Fix bug: when deleting an item older than 30 days,
soft-delete does not write deleted_at to the DB. A UTC timestamp
should be set.

/speckit.plan Backend-only change. Update ItemService.delete_item.
No migrations, the deleted_at field already exists.

/speckit.tasks

/speckit.implement
```

We skip `/clarify` and `/speckit.analyze` — for a small change these steps just add noise.

> ⚠️ **Important**: `/speckit.tasks` **cannot** be skipped, even for small changes. `/speckit.implement` reads `tasks.md` as input — without it, execution is blocked with the message "This command assumes a complete task breakdown exists in tasks.md". A minute spent on `/tasks` is cheaper than getting blocked in the middle of implementation.

### When NOT to apply this

- The change touches 3+ files and several modules — use Scenario 2.
- Unknown stack/architecture — use Scenario 7 (Brownfield Discovery).
- A truly trivial change (one line in one file) — write the code by hand without spec-kit at all.

### Pitfalls

- The temptation to skip describing edge cases in the spec → no test → the bug comes back.
- A "small feature" turned out to be large mid-`/implement` — STOP, run `/clarify` and `/analyze` retroactively.
- Working on `master` without a feature branch → `/implement` halts because the `before_specify` hook did not cut a branch. The fix is `git checkout -b 001-feature-name` and re-running `/implement`.

---

## Scenario 2 — Brownfield Feature (Full Cycle)

**Level**: Basic
**When**: A new feature in an existing project. This is the **typical** working case in a company.
**Time**: 2–4 hours (not counting constitution.md preparation)
**Commands**: `/speckit.specify` → `/speckit.clarify` → `/speckit.plan` → `/speckit.tasks` → `/speckit.analyze` → `/speckit.implement`

### Walkthrough

See **`SPEC_KIT_WORKFLOW_GUIDE.md`** — it contains the full SDD-2 (Search & Filter) walkthrough with examples of every artifact.

### Specifics

- The constitution already exists — do NOT run `/speckit.constitution`.
- In `/specify`, briefly describe integration with existing entities (`Item`, `User`).
- In `/plan`, you must mention the existing stack — otherwise the agent may suggest something foreign (Express instead of FastAPI).

### Pitfalls

- Forgetting to run `/clarify` → ending up with a plan built on flawed assumptions.
- Copy-pasting a `/specify` prompt from an old Jira ticket that mentions the stack → the stack leaks into the spec.
- Skipping `/analyze` → missing a CRITICAL Coverage gap.

---

## Scenario 3 — Constitution Setup (New Team)

**Level**: Basic
**When**: Starting a new project OR introducing SDD into a team that has not used spec-kit yet.
**Time**: 1 working day (with team discussions)
**Commands**: `/speckit.constitution` → discussion → revision

### Walkthrough

```
/speckit.constitution Create principles for a team of 6 engineers
working on a B2B SaaS in Python (FastAPI) + React.

Test-First (NON-NEGOTIABLE): pytest, 75% coverage on new modules,
mutation testing once per sprint.

Type Safety (MUST): Python 3.11+ pydantic, TypeScript strict.

API contracts (MUST): OpenAPI generator before code, breaking changes
only with API versioning.

Observability (MUST): structlog, OTEL traces, every endpoint logs
with trace_id.

Performance (SHOULD): backend p95 < 200ms on list endpoints,
< 500ms on aggregations.

Simplicity (SHOULD): no new deps without RFC. Instead of abstractions —
explicit code.

Governance: principles MUST be changed via an RFC on github discussions
+ 2 reviews from staff+. Constitution version: semver.
```

Result: `.specify/memory/constitution.md` v1.0.0.

### Team revision

1. Read it together in a meeting.
2. Each engineer proposes their edit → discussion.
3. Once there is consensus — re-run `/speckit.constitution` with the updated wording.
4. Version bump: PATCH for wording, MINOR for a new principle, MAJOR for changes/removal.
5. Each change is a separate PR with rationale.

### Pitfalls

- **Marketing-style principles**: "We value craftsmanship" — not testable, drop them.
- **Principles without rationale**: "100% coverage" without explaining *why* → the team will sabotage it.
- **Too many MUSTs**: if everything is MUST, nothing is MUST. Keep 4–6 real MUSTs + the rest as SHOULD.
- **Constitution as waterfall** — written once and never updated. Version it and review once a quarter.

---

## Scenario 4 — Re-spec / Reset (Realizing You Are Going the Wrong Way)

**Level**: Basic
**When**: After `/plan` or during `/implement` you realize requirements are missing, or the scope changed.
**Time**: 30–90 min
**Commands**: `/speckit.clarify` again → manual edit of `spec.md` → `/speckit.plan` (re-run) → `/speckit.tasks` (re-run) → `/speckit.analyze`

### Walkthrough

Imagine you are halfway through `/implement` of SDD-2 (Search). Suddenly the product owner says: "We also need search over description, not only title."

```bash
# 1. Stop implement
# 2. Roll back uncommitted code:
git stash

# 3. Re-spec:
```

```
/speckit.specify (edit) Expand scope: search now covers two fields —
title AND description (substring match, OR-logic). The "Found N items"
counter still shows the total number of matches.

The rest is unchanged.
```

The agent modifies the existing `spec.md`. Then run:

```
/speckit.clarify     # check for new ambiguities
/speckit.plan        # regenerate the plan
/speckit.tasks       # new tasks
/speckit.analyze     # audit
```

`tasks.md` adapts automatically. After that:

```
/speckit.implement
```

The agent sees that some `[X]` are already done (from previous tasks) — it only executes the new/changed ones.

### Pitfalls

- **Not keeping the `git stash`**: code lands on top of the new tasks.md → conflicts.
- **Forgetting to bump the constitution**: if the scope change violates the Simplicity principle, you must either justify it in Complexity Tracking or revise the constitution.
- **Not updating the PR description**: the reviewer will be surprised that the code does not match the old spec.

> 💡 **Key principle**: it is better to change your mind during the specs phase (1 hour) than after `/implement` (a week of refactoring).

---

## Scenario 5 — Team Collaboration

**Level**: Intermediate
**When**: Rolling out spec-kit across a team of 5–10 people.
**Time**: 1–2 sprints to adapt
**Commands**: all of them + GitHub MCP `/speckit.taskstoissues`

### Walkthrough

#### Who writes constitution.md

Tech lead + staff engineer. Minimum 2 people, maximum 4 — to avoid getting stuck in bikeshedding.

#### How to review specs

1. A PR for `specs/<feature>/spec.md` is opened **before** `/plan`.
2. Reviewers (PM + 1 engineer) look at **only spec.md** — the stack is not picked yet.
3. After approval — the same PR gets `plan.md` and `tasks.md` added.
4. A separate review on `plan.md` (focus on Constitution Check, Complexity Tracking).
5. The final code PR — references specs/<feature>/.

#### How to divide the work

| Artifact | Who | When |
|----------|-----|------|
| `spec.md` | PM + engineer pair | First day of the sprint |
| `plan.md` | Tech lead engineer | Second day |
| `tasks.md` | Generated → review | Third day |
| Implementation | Any engineer | Rest of the sprint |

One `specs/<feature>/` = one PR = one ticket in Jira/Linear.

#### Integration with Jira/Linear

```
/speckit.taskstoissues
```

Creates GitHub issues. Through a GitHub→Jira integration (for example the Atlassian Marketplace app `Issues for Jira`), the issues are automatically copied to Jira.

#### Conflict between constitution and reality

- An engineer wants to add a library that violates Simplicity.
- They open an RFC at `docs/rfc/<num>-<title>.md`.
- 2 staff approvals → the constitution is updated (`/speckit.constitution`) → MINOR bump.

### Pitfalls

- **"Constitution by Tech Lead"** without team buy-in → everyone ignores it.
- **Reviewing `spec.md` as a formality** → the problem surfaces at code review → rework.
- **PM writes `spec.md` alone** → engineering requirements (rate limit, observability) drop out.
- **Constitution never updated** → it becomes a "mythologized document" → new engineers do not trust it.

---

## Scenario 6 — Greenfield Project

**Level**: Intermediate
**When**: A brand new project being created from scratch.
**Time**: 1–2 days to the first PR
**Commands**: `specify init <new-project>` → `/speckit.constitution` → `/speckit.specify` (vision) → `/speckit.specify` (per epic) → ... → `/speckit.implement`

### Walkthrough

```bash
specify init taskify --integration claude
cd taskify
git init
```

```
/speckit.constitution Create principles for a new pet project:
Library-First, CLI Interface Mandate (every module has a CLI),
Test-First, Observability, Simplicity (KISS).
```

```
/speckit.specify Taskify is a SaaS for task management in small
teams (5–15 people). Higher goal: provide a lightweight Trello
alternative focused on kanban and simplicity.

Main epics:
1. Auth & Workspaces (multi-tenant)
2. Boards & Lists & Cards (drag-and-drop)
3. Comments & @mentions
4. Notifications (in-app + email)
5. API for integrations

This spec is the high-level vision. Details will live in per-epic specs.
```

Created `specs/001-vision/spec.md`.

```
/speckit.specify Implement Epic 1: Auth & Workspaces.
Multi-tenant with data isolation at the workspace_id level.
Email + password auth via FastAPI-Users.
Workspace ↔ User M2M via WorkspaceMember (with role: owner/admin/member).
```

Created `specs/002-auth-workspaces/spec.md`.

Then — the full cycle: `/clarify` → `/plan` → `/tasks` → `/analyze` → `/implement`.

#### Epic sequence

```
001-vision (spec only, no plan/implement) ← lives as a reference
002-auth-workspaces ← full cycle
003-boards ← full cycle (depends on 002)
004-comments-mentions ← (depends on 003)
005-notifications ← (parallel with 004)
006-public-api ← after everything
```

### Pitfalls

- **Describing everything in one spec** → a 50-page monster nobody reads.
- **Vision without a Definition of Done** → the project never "finishes".
- **Jumping into `/plan` without a vision** → after 3 epics an architectural mismatch surfaces.

---

## Scenario 7 — Brownfield Discovery (Adapting spec-kit to an Unfamiliar Codebase)

**Level**: Intermediate
**When**: You enter a repository you are seeing for the first time and need to add a feature.
**Time**: 2–6 hours (including discovery)
**Commands**: discovery (manual + AI) → `/speckit.specify` → the rest of the standard cycle

### Walkthrough

#### Phase 0 — Discovery (no spec-kit)

```
[chat with AI agent, no slash command]

Before we create a spec — help me understand the codebase.

1. Show the structure: backend/, frontend/, infra/.
2. What are the main models in backend/app/models/?
3. What API routes exist today? Show the routers under /api/v1/.
4. How is authorization organized?
5. Are there existing tests? How do they run?

Summarize in 200 words.
```

#### Phase 1 — Architecture brief

```
Save the summary to docs/architecture-brief.md.
Include:
- ASCII ER diagram
- List of endpoints
- Auth flow
- Test strategy
- Extension points for new features
```

Now you have the context for a proper `/specify`.

#### Phase 2 — Standard cycle

```
/speckit.constitution    # if it does not exist yet
/speckit.specify ...     # now with knowledge of existing code
/speckit.clarify
/speckit.plan
/speckit.tasks
/speckit.analyze
/speckit.implement
```

#### Alternative

A community extension **`spec-kit-brownfield`** exists that automates discovery. Installed via `extensions.yml`. Not recommended for production without studying the code yourself — automated discovery can miss nuances.

### Pitfalls

- **Jumping straight into `/specify`** → the agent improvises because it does not know your architecture.
- **Discovery without an artifact** → it is lost in chat history, the next engineer repeats it.
- **The spec ignores existing patterns** → the code stylistically clashes with the rest.

---

## Scenario 8 — Refactor Under SDD (legacy → spec-kit)

**Level**: Advanced
**When**: An existing complex module that needs to be rewritten, but documentation is missing.
**Time**: 1–2 weeks
**Commands**: discovery → `/speckit.specify` (reverse-engineering) → `/speckit.plan` → `/speckit.implement`

### Walkthrough

#### Reverse-engineering existing behavior

```
/speckit.specify Create a specification for the existing module
backend/app/services/notification_service.py.

Read the file and all of its tests (if any). Describe actual behavior:
- Which notification types are supported?
- What are the triggers?
- Which channels (email, in-app, push)?
- What retry mechanisms exist?
- What edge cases (errors, duplicates, rate limits)?

This is a reverse-engineered specification. Mark it as `## Reverse-Engineered`
in the title. Note in Out of Scope: "feature parity with existing code".
```

Created `specs/010-notifications-refactor/spec.md` describing the as-is behavior.

#### Delta analysis

```
/speckit.specify (edit) Now add the to-be requirements:
1. Push notification support (currently email only).
2. Rate limiting per user (10/min).
3. Centralized retry queue (currently inline).
4. Observability metrics for delivery rate.

Keep the "Existing Behavior" section as a reference.
```

#### Transition plan

```
/speckit.plan Migration strategy: Strangler Fig pattern.
Phase 1: build new modules side by side (notification_v2_service.py).
Phase 2: feature flag to switch.
Phase 3: deprecate v1, remove after one month.

Rollback: feature flag.
```

`/tasks` will generate tasks that include parallel implementation + integration tests for feature parity.

### Pitfalls

- **Refactor without a spec.md** → in a month nobody understands *what* exactly was rewritten.
- **Big-bang refactor** → a 5K LoC PR that nobody reviews.
- **No rollback plan** → something breaks in production, panic.

---

## Scenario 9 — Compliance / Regulated Domain

**Level**: Advanced
**When**: Healthcare, finance, government — there are compliance requirements (HIPAA, SOC2, GDPR).
**Time**: +30–50% on top of the base workflow
**Commands**: all of them + `/speckit.checklist` for compliance

### Walkthrough

#### Constitution with a compliance block

```
/speckit.constitution ... + Compliance Principles:

7. PHI Encryption (NON-NEGOTIABLE): all fields tagged @phi MUST be
   encrypted at rest (AES-256) and in transit (TLS 1.3).

8. Audit Logging (MUST): every read/write of @phi fields generates
   audit log entry with: actor_id, resource_id, action, timestamp,
   ip_address.

9. Data Residency (MUST): EU user data MUST stay in EU regions.

10. Right to Erasure (MUST): user can request deletion; cascades within
    30 days; compliance reports show completion.

Governance: any violation of NON-NEGOTIABLE = release blocker.
Compliance officer reviewer on all PRs tagged `compliance`.
```

#### Compliance checklist for every feature

```
/speckit.checklist Generate HIPAA compliance checklist for spec.md.
Focus on:
- PHI identification (which fields are PHI)
- Encryption coverage
- Audit logging coverage
- Access control verification
- Data retention rules
```

This generates `specs/<feature>/checklists/hipaa.md`. **`/speckit.implement` will halt** if this checklist is not fully checked off.

#### A compliance preset

```bash
specify init my-project --preset healthcare-compliance --integration claude
```

(If the preset exists — check `specify integration list` and community presets.)

#### Deeper audit before release

```
/speckit.analyze
```

It outputs additional categories: `Compliance Coverage`, `PHI Tracking`, `Audit Trail Gaps`. Any CRITICAL is a release blocker.

### Pitfalls

- **Compliance as a "checkbox" in Jira** → not actually done → fail the audit.
- **Constitution without compliance principles** → the agent does not block dangerous features.
- **Custom checklist with no revision history** → the compliance officer does not trust it.
- **Implementation without `/analyze`** → CRITICAL findings land in production.

---

## Summary table

| Scenario | Level | Time | Key commands |
|----------|-------|------|--------------|
| 1. Quick Spec Flow | Beginner | 30–60 min | specify → plan → implement |
| 2. Brownfield Feature | Basic | 2–4 hr | full cycle |
| 3. Constitution Setup | Basic | 1 day | constitution + discussion |
| 4. Re-spec / Reset | Basic | 30–90 min | clarify → spec edit → plan/tasks re-run |
| 5. Team Collaboration | Intermediate | 1–2 sprints | all + taskstoissues |
| 6. Greenfield | Intermediate | 1–2 days | init → vision → per-epic |
| 7. Brownfield Discovery | Intermediate | 2–6 hr | discovery → standard cycle |
| 8. Refactor under SDD | Advanced | 1–2 weeks | reverse-engineer → strangler fig |
| 9. Compliance | Advanced | +30–50% | + checklist + analyze with compliance mode |

---

## Recommendations — Where to start?

### If you are new to spec-kit
→ **Scenario 1** (Quick Spec Flow) on a single small bug. Then **Scenario 2** on a medium feature.

### If you are introducing spec-kit to a team
→ **Scenario 3** (Constitution) as the first step. Then **Scenario 5** (Team Collaboration) as the process template.

### If you have a legacy project
→ **Scenario 7** (Brownfield Discovery) before the first `/specify`. Do not skip discovery.

### If you are starting a new project
→ **Scenario 6** (Greenfield) with a vision-spec → per-epic specs.

### If you have compliance requirements
→ **Scenario 9** at the constitution stage already — otherwise reworking later is three times harder.

### If something went wrong
→ **Scenario 4** (Re-spec / Reset). Do not heroically try to drive `/implement` to the end if the spec is broken.

---

> 🚀 **Ready to practice?** Move on to `TASKS.md` — it has 7 practical tasks where you can sharpen each scenario.
