# Spec-Kit Training — Curriculum for Developers

> A hands-on course on Spec-Driven Development using **GitHub Spec-Kit** on a real full-stack project.

The course is modeled after training programs for AI-driven development. The goal is to take an engineer from zero familiarity with spec-kit to confidently rolling the methodology out across the team's working projects.

---

## Course contents

### Core block (minimum for self-study)

| Document | Purpose |
|----------|---------|
| `README.md` *(this file)* | Entry point. 4 stages + 6 scenarios + recommendations |
| `PROJECT_SETUP.md` | Stage 1: spinning up the training project (FastAPI + React) |
| `SPEC-KIT-docs.md` | Reference: SDD philosophy, commands, artifacts, project memory, advanced features |
| `SPEC_KIT_WORKFLOW_GUIDE.md` | End-to-end walkthrough of a single task (SDD-2) |
| `SPEC_KIT_USE_CASES.md` | 9 scenarios from Beginner to Advanced |
| `TASKS.md` | Backlog of 7 hands-on tasks (concise format) |
| `TASKS_JIRA.md` | The same tasks in Jira-story format (detailed) |

### Advanced block (for teams and real projects)

| Document | Purpose |
|----------|---------|
| `SCRUM_INTEGRATION.md` | Integrating spec-kit with Scrum ceremonies: grooming, planning, standup, demo, retro. Role matrix. Jira mapping. DoR/DoD |
| `CONSTITUTION_GUIDE.md` | Deep dive into `.specify/memory/constitution.md`: how to write it, how to evolve it, real example v1.0 → v2.x over 12 months |
| `SPECS_HYGIENE.md` | Maintaining `specs/` at scale (100+ features): lifecycle states, archival pattern, domain hierarchy, **5 currency maintenance patterns** (A–E), **3 levels of sync automation**, manifest, quarterly grooming, scripts |

---

## Training project

The course is built around the public template **`fastapi/full-stack-fastapi-template`**:

- **Backend**: FastAPI, SQLModel, Alembic, PostgreSQL
- **Frontend**: React, TypeScript, TanStack Query, Vite
- **Infrastructure**: Docker Compose, Traefik, GitHub Actions
- **Default login**: `admin@example.com` / `changethis`

This template was picked deliberately: it is realistic enough (ORM, migrations, auth, tests) yet compact enough that a full feature development cycle fits into 1–3 hours, which is ideal for training.

> 💡 **Key principle**: every spec-kit command you will run in this course works exactly the same on any real project at your company. The training template is just a sandbox.

---

## Stage 1 — Stand up the project

Work through `PROJECT_SETUP.md` end to end. The file is written as an agent prompt — you can either follow the steps manually or hand it off to an AI agent in a separate chat.

```bash
git clone https://github.com/fastapi/full-stack-fastapi-template.git
cd full-stack-fastapi-template
cp .env.example .env
docker compose up -d
```

**Success check:**

- ✅ http://localhost:5173 loads the frontend
- ✅ http://localhost:8000/docs shows the Swagger UI
- ✅ Login as `admin@example.com` / `changethis` works

**Estimated time**: 20–40 minutes.

---

## Stage 2 — What Spec-Driven Development is

Read `SPEC-KIT-docs.md` end to end. Pay particular attention to these three sections:

1. **SDD philosophy** — why code *serves* the specification, not the other way around.
2. **Spec-kit commands** — `/speckit.constitution`, `/speckit.specify`, `/speckit.clarify`, `/speckit.plan`, `/speckit.tasks`, `/speckit.analyze`, `/speckit.implement`.
3. **Artifacts** — what gets generated and where (`.specify/`, `specs/<feature>/`).

> 💡 **Practical tip**: don't skip the philosophy section. Without understanding *why*, SDD looks like "extra bureaucracy" and the team will refuse to use it.

**Estimated time**: 45–60 minutes of reading.

---

## Stage 3 — Install spec-kit

Spec-kit is installed via `uvx` (recommended) or `pipx`. It works with Claude Code, Copilot, Cursor, Gemini, Codex, Windsurf, and 25+ other AI agents.

### Prerequisites

- Python 3.11+
- `uv` (`pip install uv` or `brew install uv`)
- `git`
- An AI agent (Claude Code, Copilot, Cursor, or another)

### Quick install into an existing project

```bash
cd full-stack-fastapi-template
uvx --from git+https://github.com/github/spec-kit.git specify init . --integration claude
```

### Verification

```bash
specify check
specify integration list
```

In your editor (Claude Code / Copilot / Cursor), confirm that autocomplete shows `/speckit.constitution`, `/speckit.specify`, `/speckit.plan`, `/speckit.tasks`, `/speckit.implement`.

> ⚠️ **Important**: the only official source is `github.com/github/spec-kit`. Any PyPI package named `specify-cli` is *NOT* affiliated with GitHub.

**Estimated time**: 10–20 minutes.

---

## Stage 4 — Hands-on scenarios

Six scenarios, from the simplest to the most involved. Details live in `SPEC_KIT_USE_CASES.md`.

### Scenario 1 — Quick Spec Flow (for small changes)

| | |
|---|---|
| **When to use** | A small isolated feature or bug fix |
| **Time** | 30–60 min |
| **Commands** | `/speckit.specify` → `/speckit.plan` → `/speckit.tasks` → `/speckit.implement` |

**Example:**

```
/speckit.specify Add a GET /api/v1/items/random endpoint
that returns a random item from the database for the authenticated user.
```

We skip `/clarify` and `/analyze` — for a small change these steps just add noise.

> ⚠️ **Important**: `/speckit.tasks` **cannot** be skipped — `/speckit.implement` reads `tasks.md` as its input and refuses to run without it. See `SPEC_KIT_USE_CASES.md`, Scenario 1, for details.

---

### Scenario 2 — Brownfield Feature (full cycle on an existing project)

| | |
|---|---|
| **When to use** | A new feature in existing code (the typical case at a company) |
| **Time** | 2–4 hours |
| **Commands** | `/speckit.constitution` *(once)* → `/speckit.specify` → `/speckit.clarify` → `/speckit.plan` → `/speckit.tasks` → `/speckit.analyze` → `/speckit.implement` |

This is the **main** workflow for production projects. For a full walkthrough see `SPEC_KIT_WORKFLOW_GUIDE.md` (using task **SDD-2: Search & Filter** as the example).

```
/speckit.specify Add search by the title field on the items list
with a 300ms debounce and a "Found N items" counter.
```

---

### Scenario 3 — Constitution-First (new team / new project)

| | |
|---|---|
| **When to use** | Starting a new project or rolling SDD out to a team |
| **Time** | 1 day |
| **Commands** | `/speckit.constitution` → discussion → revisions |

**Example prompt:**

```
/speckit.constitution Create principles for our team: code quality
(80%+ test coverage, mandatory code review), testing standards (TDD for
core domain), UX consistency (Figma design system), performance
(<200ms p95). Add governance: any MUST principle can only be changed
through an RFC + 2 reviews.
```

The result: `.specify/memory/constitution.md`, which becomes a real contract for every subsequent `/plan` via the **Constitution Check** gates.

---

### Scenario 4 — Reset / Re-spec (we realized we're heading the wrong way)

| | |
|---|---|
| **When to use** | After `/plan` you realize requirements are missing or scope changed |
| **Time** | 30–90 min |
| **Commands** | `/speckit.clarify` again → `/speckit.specify` (edit) → `/speckit.plan` (re-run) |

> 💡 **Key rule**: it's better to change your mind during the specs phase than after `/implement`. One day on the spec saves a week of refactoring.

---

### Scenario 5 — Team Collaboration *(discussion)*

How to roll spec-kit out to a 5–10 person team:

- Who writes `constitution.md` (typically tech lead + staff engineer)
- How to review specs (PR on the `spec.md` file itself, *before* `/plan`)
- How to slice work (one `specs/<feature>/` = one PR = one ticket)
- How to integrate with Jira/Linear (`/speckit.taskstoissues`)
- How to resolve conflicts between the constitution and reality (RFC)

No commands here — just a group discussion.

---

### Scenario 6 — Greenfield *(discussion)*

How to start a **new** project with spec-kit:

```bash
specify init my-new-project --integration claude
```

Sequence:

```
/speckit.constitution → /speckit.specify (high-level vision)
                     → /speckit.specify (per-epic features)
                     → /speckit.plan → /speckit.tasks → /speckit.implement
```

On a greenfield project spec-kit saves the most time, because there's no need to describe "how the new code integrates with the existing one."

---

## Hands-on task backlog

`TASKS.md` contains **7 tasks** on the training project, ranging from Easy to Hard. Pick at least **3 tasks** to do on your own:

| ID | Task | Difficulty | SP |
|----|------|------------|----|
| SDD-1 | Tags for items (M2M) | Easy | 5 |
| SDD-2 | Search & Filter with debounce | Easy | 3 |
| SDD-3 | Comments with permissions | Medium | 8 |
| SDD-4 | Dashboard with recharts | Medium | 8 |
| SDD-5 | Favorites with optimistic UI | Easy | 5 |
| SDD-6 | CSV Export via StreamingResponse | Medium | 5 |
| SDD-7 | Activity Log (audit middleware) | Hard | 13 |

Detailed Jira-style descriptions (Acceptance Criteria, Subtasks, Technical Notes) live in `TASKS_JIRA.md`.

---

## Recommended path for solo learners (40 hours)

```
Day 1 (8h)  — Stages 1–3 + reading SPEC-KIT-docs (including 8.5 and 8.6)
Day 2 (8h)  — Scenario 1 (Quick Flow) + SDD-1 (Tags)
Day 3 (8h)  — Scenario 2 (Brownfield) + SDD-2 (Search) — full cycle
Day 4 (8h)  — SDD-3 or SDD-5 + SDD-6 + reading CONSTITUTION_GUIDE
Day 5 (8h)  — SDD-7 (Hard) + retrospective + writing your team's internal constitution.md
```

## Recommended path for a team (workshop, 2 days)

```
Day 1 morning  — Stages 1–3 (everyone together)
Day 1 evening  — Scenario 2 live: facilitator runs SDD-2 from /specify to /implement
                 (participants follow along on their own machines)
Day 2 morning  — In pairs: one task per pair (SDD-3, SDD-4, SDD-5)
Day 2 evening  — Reading SCRUM_INTEGRATION + CONSTITUTION_GUIDE +
                 drafting the team's constitution.md
```

## Recommended path for a tech lead rolling SDD out to a team (2 weeks)

```
Week 1
  - Days 1-2: Champion pilot — run Scenario 2 yourself on 2 features
  - Day 3:    Read SCRUM_INTEGRATION and CONSTITUTION_GUIDE
  - Days 4-5: Drafting workshop (constitution.md v1.0 with ratification)

Week 2
  - Days 1-3: Team (3-4 engineers) pilots spec-kit on 2-3 features
  - Day 4:    Pilot retrospective → corrections to constitution + processes
  - Day 5:    Decision on full rollout or rollback
```

---

## Common beginner mistakes

1. **Mentioning the stack in `/specify`.** The spec should describe *what* and *why*, not *how*. The stack belongs in `/plan`.
2. **Skipping `/clarify`.** Save 5 minutes — lose 2 hours to refactoring.
3. **Not running `/analyze` before `/implement`.** You miss CRITICAL findings → the agent ships code against inconsistent requirements.
4. **Leaving `[NEEDS CLARIFICATION]` markers in place.** If `spec.md` has unresolved questions, `/plan` will generate garbage.
5. **Not updating `constitution.md`.** The document goes stale within six months while Constitution Check keeps citing the old principles.
6. **Running `/implement` against an incomplete checklist.** Spec-kit will ask for confirmation — don't say "yes" reflexively.

---

## Keeping specifications current between sprints

This is a separate practical problem that surfaces on projects with 30+ features: `specs/<feature>/spec.md` is created as a snapshot of the moment the feature was conceived. Six months after ship it has turned into a historical document, but readers still consult it as if it were the current reality. This is a recognized issue in the community ([Discussion #152](https://github.com/github/spec-kit/discussions/152)).

Spec-kit doesn't solve this out of the box — it's a matter of team discipline and the pattern you choose. The course describes **5 patterns** agnostically, without recommendations. Overview:

| Pattern | Essence | Currency frequency |
|---------|---------|---------------------|
| **A. Atomic + Frozen** | Specs immutable after ship; current state isn't maintained | None |
| **B. Supersession Chain** | Frontmatter `supersedes` / `superseded_by` builds a version chain | Per-replacement |
| **C. Living Domain Specs** | `specs/<domain>/domain.md` is updated on every PR as part of DoD | Per-PR |
| **D. Unified Spec** | One file per domain with a `## Changelog` section | Per-commit |
| **E. Hybrid** | `docs/architecture/<domain>.md` (high-level) + flat `specs/` (atomic deltas) | Per-quarter (or per-PR) |

The choice depends on four factors:

- **Team size** — pattern C is hard to keep alive with 10+ people due to merge conflicts; pattern D breaks the spec-kit mechanics.
- **Rate of change in the domain** — pattern A is acceptable for slowly evolving domains; pattern B fits domains with clear version replacement.
- **Archival strategy** — pattern C conflicts with archival; pattern E does not.
- **Team discipline** — pattern A demands none of it; pattern C is the most demanding.

### Pattern A — Atomic + Frozen

Each spec is immutable after ship. It is never edited under any circumstances. Current state doesn't exist as a separate artifact — to understand how a feature works today, the reader has to scan all relevant specs and synthesize the answer themselves.

**Mechanics:**

```yaml
# specs/015-add-mfa/spec.md  (after ship — frozen forever)
---
status: implemented
implemented_date: 2026-01-15
---
```

**Pros:**
- Minimal effort — nothing is required after ship
- Clean immutable history: good for compliance, audit, post-mortem
- No merge conflicts in specs (because they aren't edited)
- Conceptually simple: "spec = snapshot of the moment"

**Cons:**
- "How does it work today?" has no convenient answer
- Drift from current state is inherent to the pattern **by definition**
- Onboarding a new person is expensive: they have to scan 5–10 specs in a domain to reconstruct current state
- At a scale of 100+ specs, current state becomes elusive

---

### Pattern B — Supersession Chain

Each new spec explicitly marks its predecessor via frontmatter. This builds a chain of versions where the head of the chain is the current state.

**Mechanics:**

```yaml
# specs/023-passkeys/spec.md  (new)
---
supersedes: [015-add-mfa]
status: implemented
---

# specs/015-add-mfa/spec.md  (predecessor, auto-updated)
---
status: superseded
superseded_by: 023-passkeys
---
```

A script can walk the chain to find the head — that is the current source of truth for the functionality.

**Pros:**
- Preserves history while still letting automation reach current state
- Frontmatter format is automation-friendly (easy to parse)
- Clear formal "this replaces that" relationship
- Doesn't rely on human discipline at read time — the only discipline needed is to fill the field at write time

**Cons:**
- Doesn't capture **partial** replacements: features rarely *fully* replace their predecessors
- Real relationships between features are often graphs (one-to-many, many-to-many), but a chain is strictly linear
- Discipline around filling `supersedes` is critical at write time and slips easily
- Deep chains (5+ versions) make navigation cumbersome
- Doesn't describe additive evolution, where the spec *extends* an old one rather than replacing it

---

### Pattern C — Living Domain Specs

A separate folder with a `domain.md` for each bounded context — always current state. Atomic specs update `domain.md` as part of the DoD on every PR.

**Mechanics:**

```
specs/
├── auth/
│   ├── domain.md              ← LIVING; updated on every PR
│   ├── 001-user-auth/         ← historical, frozen
│   ├── 015-add-mfa/           ← historical, frozen
│   └── 023-passkeys/          ← latest; its PR updated domain.md
└── items/
    ├── domain.md
    └── ...
```

`auth/domain.md` contains the up-to-date description of auth functionality. The atomic specs inside it are frozen delta records. The DoD rule: a PR doesn't merge without updating the affected sections of `domain.md`.

**Pros:**
- Domain locality — everything related to auth lives in one place
- Single discovery path: a new person reads `auth/domain.md` for 5 minutes and has the full picture
- Tight link between atomic changes and current state
- Atomic specs inside remain frozen — they keep all the upsides of pattern A

**Cons:**
- **Conflicts with archival**: moving `specs/auth/015-add-mfa/` into `_archive/` breaks domain locality — either you don't archive, or you break the structure
- Per-PR updates to `domain.md` create friction for the feature engineer: you want to ship the feature, but you also have to edit a huge document
- When two teams work in the same domain in parallel, you get merge conflicts in `domain.md`
- The domain folder grows fast: all specs + `domain.md` co-located; after a year `auth/` has 50–80 folders inside
- High demand on team discipline — without it, `domain.md` drifts immediately

---

### Pattern D — Unified Spec

One file per domain — `specs/auth/spec.md` — grows through commits. A `## Changelog` section at the bottom records the evolution.

**Mechanics:**

```markdown
# Auth Spec (current)

[everything that's current right now: user stories, FR, edge cases, key entities]

## Changelog

### 2026-04-28 — Added passkeys
- Added WebAuthn support
- TOTP marked as deprecated

### 2026-01-15 — Added TOTP MFA
- ...

### 2025-09-10 — Initial auth (v1)
- ...
```

Every change is a commit to the same file. Git history naturally preserves the evolution via `git log -p specs/auth/spec.md`.

**Pros:**
- Minimalist structure — one file per domain, no folders full of deltas
- Currency comes for free: the head of the file is always current state
- Git history carries the entire evolution automatically — `git log -p` gives the full sequence of changes
- Low cognitive load: one place, one read

**Cons:**
- **Breaks the mechanics of spec-kit**: `/speckit.specify` creates a new folder, it doesn't edit an existing file — you need a custom workflow or forked templates
- The file grows without bound: after 2 years a single domain hits 100+ KB and scrolling is painful
- High risk of merge conflicts under parallel work — everyone edits the same file
- The "atomic delta" concept is lost: it's hard to tell which specific changes shipped in a single release
- Tight coupling to the git workflow — without `git log` you can't reconstruct history

---

### Pattern E — Hybrid (architecture docs + flat specs)

Separate file space for current state (`docs/architecture/<domain>.md`) and a separate one for atomic deltas (`specs/<NNN-feature>/`). Architecture docs are high-level (1–2 pages); specs are detailed delta records.

**Mechanics:**

```
docs/architecture/
├── overview.md
├── auth.md            ← current state of auth domain (≤2 pages)
├── items.md
└── billing.md

specs/
├── _archive/
│   ├── 2025-Q4/
│   │   └── 015-add-mfa/
│   └── 2026-Q1/
├── 042-webauthn-fallback/    ← active
└── 048-bulk-edit/            ← active
```

Specs carry a `domain: auth` field in frontmatter so they can be queried by domain via a script. Architecture docs are updated either per-PR (through a template modification) or per-quarter (at the retro).

**Pros:**
- **Clean spec archival**: specs live in a flat structure and move into `_archive/` freely without touching architecture docs
- Architecture docs are short (1–2 pages) — easier to maintain, smaller drift surface
- Doesn't break spec-kit mechanics — standard workflow with no modifications
- Different update cadences are possible: atomic specs move fast (per-PR), architecture moves slowly (per-quarter)
- Clear separation of responsibility: the feature engineer writes the spec; the tech lead/architect maintains the architecture doc

**Cons:**
- **Two parallel systems** — risk of divergence between them
- Without a sync process, architecture docs drift away from reality
- Deciding "what goes in the spec vs. what goes in architecture" requires judgment — the boundary isn't crisp
- It's not obvious to a new person which artifact to start with
- Without quarterly reviews, architecture docs turn into historical documents

---

> 📖 **Details**: `SPECS_HYGIENE.md` contains an expanded write-up of each pattern (mechanics / what works in theory / what breaks in practice / pros and cons), a comparison table by scaling/conflict risk/spec-kit compatibility, and a checklist for scale.

---

## Key insights to take away from the course

If you go through all seven (or ten, including the advanced block) documents, the following ought to become part of your working mental model:

**1. Spec-kit is an engineering workspace, not a Jira replacement.** Jira stays the planner and tracker. The link: the Jira ID in the folder name + `Resolves PROJ-XXXX` in the PR.

**2. The constitution is an enforceable contract, not a slogan list.** Every principle has to pass the test: "could the agent violate this in `/plan`, and would Constitution Check catch it?" If not, it's not a principle. See `CONSTITUTION_GUIDE.md` for details.

**3. Project memory has 3 levels**: invariants (constitution, init options), live context (CLAUDE.md/AGENTS.md), decision history (specs/). See `SPEC-KIT-docs.md`, section 8.5.

**4. SP are estimated at the level of Functional Requirements, not tasks.** FR → Jira stories; tasks.md → executable checklist in the repo. See `SCRUM_INTEGRATION.md`.

**5. At scale, without curation, `specs/` turns into a dumping ground.** Lifecycle states + quarterly grooming + domain hierarchy. **5 patterns** for keeping specs current between sprints (Atomic+Frozen, Supersession Chain, Living Domain Specs, Unified Spec, Hybrid) + **3 levels of sync automation** (template mod, hooks, custom command). See `SPECS_HYGIENE.md`.

**6. `/speckit.tasks` is mandatory even in the Quick Flow** — `/implement` won't run without it. The shortcut flow: skip `/clarify` and `/analyze`, but never `/tasks`.

**7. Spec-kit leans on advanced agent features**: skills mode (Claude/Codex), sub-agent dispatch in the research phase, parallel `[P]` markers in `/implement`. See `SPEC-KIT-docs.md`, section 8.6.

**8. Team adoption is incremental, not "everyone switches Monday."** Champion pilot → showcase → constitution drafting → team pilot → standard. The plan is in `SCRUM_INTEGRATION.md`.

---

## What's next — for solo learners

After finishing the course:

- Adapt `constitution.md` for your own team — use `CONSTITUTION_GUIDE.md` as a process template.
- Build an **internal template** spec-kit project with your constitution and an example spec.md — new repos start in 5 minutes.
- Wire `/speckit.taskstoissues` to Jira/Linear via MCP or your own script.
- Run a workshop on this curriculum for the rest of the team.

## What's next — for the team's tech lead

- Champion pilot → showcase → drafting workshop → team pilot (the plan lives in `SCRUM_INTEGRATION.md`, section "How to roll SDD out to a team").
- Set up the grooming/planning/retro rituals per `SCRUM_INTEGRATION.md`.
- After 1 quarter — first `SPECS_HYGIENE.md` ritual (archival + index).

---

> **Questions and feedback**: keep the course repository alive — add your own scenarios, raise the bar on tasks, capture your own best practices.
