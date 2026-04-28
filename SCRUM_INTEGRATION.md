# SCRUM_INTEGRATION — Spec-Kit in Scrum/Agile Processes

> How to roll spec-kit out to a team of 5–10 engineers running a standard Scrum cycle. Who writes which artifacts, when, and how it fits into grooming / planning / standup / demo / retro. Integration with Jira/Linear.

---

## Honest disclaimer about state-of-the-art

Spec-kit is roughly a year old at the time of writing (Sept 2025+). Real public production case studies of "team X has been running it in Scrum for a year" are rare. From community discussions ([#152](https://github.com/github/spec-kit/discussions/152), [#299](https://github.com/github/spec-kit/issues/299), [#889](https://github.com/github/spec-kit/issues/889)) we can see:

- **Pain points** from real teams (single-spec-shot trap, work item sprawl, PR-too-large)
- **A handful of patterns** that have emerged as consensus (Jira ID in the folder name, FRs as subtasks)
- **Lots of blank spots** — exact timing, Sprint demo, retrospective practices

This document is a **synthesis of the available practices plus reasoned recommendations** built on top of standard Scrum and the mechanics of spec-kit. Adapt it to your team, capture in retros what works and what doesn't.

---

## The big picture

Three things to accept up front to avoid setting wrong expectations:

**1. Spec-kit doesn't replace Jira/Linear — it lives alongside them.** Jira is the planner and tracker; spec-kit is the engineering workspace. The link between them is thin: a Jira ID in the folder name plus `Resolves PROJ-XXXX` in the PR.

**2. Out of the box, spec-kit produces one large feature branch and one large PR per spec.** The community openly acknowledges this as a problem (Discussion #152). The realistic Scrum shape of "1 user story → 1 small branch → 1 short PR" has to be adapted explicitly — for example, by splitting into per-phase MRs.

**3. SPs are estimated at the Functional Requirements level, not at the task level.** This is the strongest pattern from the discussions cited (Issue #889, plainly):

> Functional requirements (FR-001, FR-002…) in spec.md are what go into Jira as user stories with story points. tasks.md stays as an executable checklist in the repository.

Why: FRs are a stable container; tasks can get rewritten after `/speckit.analyze`; PMs/QA look at FRs (readable), not tasks (technical detail).

---

## The sprint cycle with spec-kit

### Before the sprint — Backlog Refinement (Grooming)

This is the critical moment where SDD either works or turns into "SpecFall" (a markdown wrapper around waterfall).

**Preparation (async, before the meeting)**

The PM/PO arrives with 3–7 candidate features for the next sprint. The tech lead or owner of each feature runs `/speckit.specify` for each one **ahead of the meeting**. 5–15 minutes per feature buys you a draft `spec.md` with User Stories, FR-001..N, and Success Criteria.

> 💡 **Key argument for batching the kickoff** (from Issue #299): "Each user story creates an isolated tree of development. This causes problems: design tunnel-vision, it becomes hard to motivate enabling work… Encourage running specify multiple times up-front so multiple user stories are collected." In team mode, it's better to collect several drafts before grooming rather than going one-shot per ticket.

**During the meeting (1 hour)**

1. The owner of each feature spends 5 minutes walking through the draft `spec.md`.
2. The team reads the User Stories and FRs — asks, pushes back, clarifies.
3. For features with ambiguity, run `/speckit.clarify` **live**: the agent asks 5 multiple-choice questions, the team decides together, answers land directly in `## Clarifications`.
4. For the highest-priority items, run `/speckit.checklist` for quality validation from the PM/QA side.

**Artifacts coming out of grooming:**

```
specs/
├── PROJ-1234-user-tags/
│   ├── spec.md              ← refined, clarified
│   └── checklists/requirements.md
├── PROJ-1235-search-filter/
│   ├── spec.md
│   └── checklists/requirements.md
└── PROJ-1236-comments/
    └── spec.md              ← not yet clarified, rolls into next grooming
```

### Start of the sprint — Sprint Planning

**Before the meeting (async)** — the owner of each feature runs `/speckit.plan` and `/speckit.tasks`.

> ⚠️ **Why async**: a full `/plan → /tasks → /analyze` cycle takes 30–60 minutes per feature. Running it live in the planning meeting will stretch it to 3 hours. Scott Logic's review complains about being "10x slower if executed live" — that confirms the suspicion.

This produces, per feature:

- `plan.md` — Constitution Check, Technical Context, Project Structure
- `research.md` — Decision/Rationale/Alternatives
- `data-model.md`, `contracts/`, `quickstart.md`
- `tasks.md` — phases Setup → Foundational → US1 (P1) → US2 (P2) → Polish

**At the meeting itself (1 hour):**

1. The owner of each feature spends 5 minutes covering: what the feature requires, which Constitution Check it bumps into, how it decomposes into user stories.
2. The team estimates story points **at the FR level** (FR-001 = 2 SP, FR-002 = 5 SP).
3. Tickets are created in Jira/Linear **labelled per user story** (US1, US2). Subtasks under each are *not a mirror of tasks.md* — they're coarse-grained chunks of one to two days of work. Avoid "work item sprawl".
4. `/speckit.taskstoissues` (if GitHub) or a custom integration (if Jira/Linear) syncs the high-level tasks.

**Distributing features across engineers:**

One `specs/<feature>/` = one owner = one branch. If a feature is large (>10 SP), split it into MRs by phase:

- Phase 2 Foundational + US1 = a separate PR
- US2 = a second PR
- US3 + Polish = a third PR

This partly mitigates the "PR-too-large" problem the community calls out.

### During the sprint — Daily Standup

Real reports *from other teams' retrospectives* are scarce. Issue #181 surfaces a related pain: *there's no automatic `[X]` marking in `tasks.md`*, so "reporting against checkboxes" is currently awkward.

Recommended practice:

- Each engineer reports **at the user story / FR level**, not the task ID level. So not "finished T012, T013, starting T014", but "closed US1 for search-filter, starting US2 — needs half a day; blocker: a question for PM about the edge case with whitespace queries."
- tasks.md is open in the repo — the tech lead and the PR reviewer see real-time progress through GitHub.
- If tasks.md drifts away from spec.md (the engineer is implementing something other than what's in the FRs), that's a "STOP, go back to `/clarify` or `/analyze`" signal.

> ⚠️ **Mandatory ritual**: when an engineer opens a PR, they must confirm in the description that `spec.md` has been updated (added `## Implementation Notes` or amended an FR if reality diverged from the plan).

Direct community quotes about **drift** (Discussion #152, Jflam): "these things naturally diverge as it's pretty rare for the agent to one-shot the code… those changes ultimately become material. So unfortunately right now it's on you to fold those things back into the spec."

### Mid-sprint check / Refinement of next sprint

Mid-sprint — grooming of the next sprint runs in parallel. Features that didn't make it into the current one (`PROJ-1236-comments` in my example) are walked through `/clarify` and `/plan` and prepped for the next sprint's planning. Standard Scrum practice — spec-kit doesn't change anything here.

### End of the sprint — Sprint Review / Demo

What you show stakeholders:

- **A working feature** — that's primary.
- **`specs/<feature>/quickstart.md`** as a "scripted demo flow". By spec-kit's documentation, quickstart.md is for manual validation scenarios, but those map perfectly to demo scripts: "User creates a tag → assigns it to an item → filters — sees the result."
- **`spec.md`** for PM/business — *exactly what* was built, which user stories were closed, which FRs are covered.

> 💡 I have no direct reports of "teams showing stakeholders quickstart.md" in my research — this is a reasoned recommendation given that quickstart.md is already structured precisely that way.

### Retrospective

The place where teams give honest feedback on SDD.

**Questions worth asking in the retro:**

1. *Did the extra 30–60 minutes of specify+clarify+plan per feature pay off?* If not for small ones, switch to a Quick Spec Flow for < 4 SP.
2. *Has the Constitution Check in `/plan` actually failed in the last 2 sprints?* If not, either the principles are toothless or the template doesn't enforce them.
3. *Are the specs actually being read before PR review?* If not, this is SpecFall (markdown that nobody uses).
4. *Which features got stuck — why? Is it a decomposition problem (US1 should have been smaller), a clarify problem (missing information), or implementation?*

**Direct community quotes about retros:**

What worked (Discussion #1549): "we've been using speck-kit since a few months now, love it."

What didn't:
- Martin Fowler (fragment): "verbose and tedious-to-review markdown files… overkill for the feature size".
- Scott Logic (fragment): "10x slower… a sea of markdown documents".
- Marmelab (fragment): "risks burying agility under layers of Markdown."
- InfoQ (fragment): "SpecFall — markdown monster that generates layers of outdated documentation on arrival."
- EPAM (Discussion #152): "that model is effective for a startup with a monorepo, but it doesn't map well to enterprise environments."

---

## Roles matrix — who creates what

| Artifact | Who drafts | Who reviews / approves | When |
|----------|------------|------------------------|------|
| `constitution.md` | Tech lead + 1 staff | Whole team in discussion; formally 2 staff approvals | Once; bumps on RFC |
| `spec.md` (per feature) | Feature owner (engineer) or PM | PM + 1 engineer; mandatory *before* `/plan` | Before grooming (draft), at grooming (refined) |
| `## Clarifications` section | Team, live | Implicitly — by consensus | At grooming |
| `plan.md`, `research.md`, `data-model.md`, `contracts/` | Owner runs `/plan`; AI generates | Tech lead — focused on Constitution Check, Complexity Tracking | Between grooming and planning, async |
| `tasks.md` | AI via `/speckit.tasks` | Owner eyeballs it; tech lead at planning | Before planning |
| `/speckit.analyze` report | AI; not persisted to a file | Owner resolves CRITICAL/HIGH | Before `/implement` |
| Code + `[X]` in tasks.md | AI via `/speckit.implement` + engineer | Standard PR review | Sprint execution |
| Updated `spec.md` after implement | Owner | Reviewer before merge | At PR stage |
| Quickstart demo | Owner | PM at review | Sprint demo |

> 💡 **Authoring constitution** (Discussion-style fragments): "authoring constitution demands senior engineering judgment, time, and iteration… junior developers cannot produce this in a day, and most teams will revise theirs repeatedly before it holds up in practice."

---

## Integration with Jira/Linear

### Base mapping

| Jira | Spec-Kit | Where it lives |
|------|----------|----------------|
| Epic | (opt.) `vision-spec.md` or architecture ADR | `specs/EPIC-XX-vision/` |
| Story | `spec.md` for the entire feature | `specs/PROJ-XXXX-feature/spec.md` |
| Subtask | Functional Requirement (FR-001...) | A section in `spec.md` |
| Sprint | (frontmatter tag) | `sprint: 24` in YAML |
| Story Points | (frontmatter tag + on FR) | `story_points: 5` |
| Acceptance Criteria | User Story → AC in spec.md (Gherkin) | Inside `spec.md` under the User Story |
| Implementation tasks | `tasks.md` (T001, T002...) | In git, *not* in Jira |

### Workflow from Jira ticket to PR

```bash
# Ticket: PROJ-1234 "Add search to items list"

git checkout -b PROJ-1234-search-filter
```

In the agent chat:

```
/speckit.specify

PROJ-1234. Implement search over items by title. Debounce 300ms,
counter "Found N items", clear via X or Esc.
```

This creates `specs/PROJ-1234-search-filter/spec.md`. Then the standard cycle: `/clarify` → `/plan` → `/tasks` → `/analyze` → `/implement`.

PR description:

```markdown
## Resolves PROJ-1234

Spec: [specs/PROJ-1234-search-filter/spec.md](./specs/PROJ-1234-search-filter/spec.md)
Plan: [specs/PROJ-1234-search-filter/plan.md](./specs/PROJ-1234-search-filter/plan.md)
Quickstart (manual validation): [quickstart.md](./specs/PROJ-1234-search-filter/quickstart.md)

### Functional requirements covered
- ✅ FR-001 — Substring matching (PROJ-1235)
- ✅ FR-002 — Debounce 300ms (PROJ-1236)
- ✅ FR-003 — Counter (PROJ-1237)
- ✅ FR-004 — URL state (PROJ-1238)
```

If Jira has GitHub integration enabled, `Resolves PROJ-1234` automatically closes the ticket on merge.

### Frontmatter for a bidirectional link

In `spec.md` add:

```yaml
---
status: active
jira_id: PROJ-1234
jira_url: https://yourcompany.atlassian.net/browse/PROJ-1234
sprint: 24
priority: P1
story_points: 5
created: 2026-04-28
---
```

In the Jira ticket, in turn — link to the specification in git ("Linked issues" field or in the description).

### `/speckit.taskstoissues` for Jira

The command exists, but only works with **GitHub Issues**. For Jira:

- Create FR-subtasks **manually** during sprint planning (5 minutes).
- Or via Atlassian MCP — Jira Cloud has an MCP server through which an AI agent can create tickets. This is an extension on top of spec-kit, not out of the box.
- Or your own script via the Jira REST API.

Discussion #1549 shows that community teams are already building their own integrations — "we explored a way to integrate it with our stack (e.g. Jira, tools for requirements quality analysis, code quality, etc)."

---

## Definition of Ready / Done

Lock these in as a team standard.

### DoR (Definition of Ready) for a ticket — before pulling into the sprint

- ✅ `spec.md` exists, with no `[NEEDS CLARIFICATION]` markers
- ✅ `## Clarifications` section is filled in or marked as N/A
- ✅ Constitution Check in `plan.md` passes
- ✅ `/speckit.analyze` has no CRITICAL findings
- ✅ SPs estimated (at the FR level, not tasks)
- ✅ Owner assigned, branch created

### DoD (Definition of Done) for a ticket — before merge

- ✅ All tasks in `tasks.md` marked `[X]`
- ✅ `quickstart.md` scenarios executed manually (by at least one engineer who isn't the author)
- ✅ `spec.md` updated if implementation diverged from the original FRs
- ✅ PR description links to `specs/<feature>/quickstart.md`
- ✅ CI green, code coverage ≥ Constitution rule
- ✅ Code review approval from at least 1 staff (or 2 for critical changes)

---

## Common pitfalls in a Scrum context

From direct quotes and real discussions:

1. **"SpecFall"** (InfoQ) — you set up the tools and ceremonies without the cultural shift → outdated markdown. Antidote: in retros, regularly ask "Are specs actually being read? Does review rely on them?" If not, sound the alarm.
2. **Single-spec-shot trap** (Issue #299) — every user story in isolation, no cross-story architectural decisions emerge. Antidote: batched `/specify` before grooming + a single shared `vision-spec` for the project.
3. **PR-too-large** (Discussion #152) — one spec → one giant PR. Antidote: split into per-phase MRs (Foundational | US1 | US2 + Polish).
4. **Work-item sprawl** (Issue #889) — 1:1 mapping of `tasks.md` → Jira subtasks creates hundreds of tiny tickets. Antidote: map FRs → Jira stories, keep tasks.md as an executable checklist in the repo.
5. **Spec drift** (Jflam, Discussion #152) — implementation diverges from the spec. Antidote: a mandatory `spec.md` update as part of DoD; the PR description must link to the updated spec.md.
6. **Slow on small features** (Scott Logic, Discussion #152) — 10x slower on "Add login". Antidote: an explicit Quick Spec Flow for < 4 SP — skip `/clarify` and `/analyze` (but `/tasks` is still mandatory).
7. **Multi-repo / cross-team doesn't fit** (EPAM, Discussion #152) — spec-kit is optimized for monorepo + a single team. Antidote: for cross-team work — a coordination spec at org level (not in the repo), with per-repo specs rolled up.
8. **No automatic completion tracking** (Issue #181) — `[X]` has to be marked by hand. Antidote: wait for future versions or write your own git pre-commit hook.

---

## How to roll SDD out to the team — phased plan

Don't push SDD onto the whole team at once. The realistic path visible in the discussions:

### Weeks 1–2 — Champion pilot

**1 engineer (champion)** runs a pilot on 2–3 of their tasks. Walks the full cycle, records hours spent. Runs Scenario 2 (Brownfield Feature) from `SPEC_KIT_USE_CASES.md`.

### Week 3 — Showcase at retro

The champion presents an example at the retro: spec, plans, artifacts, final PR. Honestly: where it was faster, where it was slower, where quality went up.

### Week 4 — Constitution drafting

Tech lead + staff + 1 product person sit down with the accumulated pain-point list from the champion and write **constitution v1.0.0**. Details in `CONSTITUTION_GUIDE.md`.

### Weeks 5–6 — Hands-on team work on 2 features

Tech lead + 1 engineer take 2 features through the full spec-kit. Everyone else works as usual but can experiment on small tasks.

### Week 7 — Rest of the team, optional

In grooming you propose `/speckit.specify` as the first stage. Whoever wants tries it; whoever doesn't goes the traditional route. Within 2 sprints most will see the upside and opt in.

### Week 9+ — Standard

SDD becomes the standard for new features; traditional flow is used only for special cases (emergency hotfix, small changes < 2 SP). Constitution v1.x with an RFC process.

---

## Further reading

- **`CONSTITUTION_GUIDE.md`** — how to write and evolve constitution.md.
- **`SPECS_HYGIENE.md`** — at scale (200+ specs), how to keep `specs/` tidy.
- **`SPEC_KIT_USE_CASES.md`** — Scenario 5 (Team Collaboration) and Scenario 9 (Compliance).

---

> 🚀 **Simplest piece of advice**: try it for one sprint. Don't try to swallow it all at once — the payoff comes after 2–3 sprints, once the constitution has matured, the team is comfortable with the slash-commands, and `specs/` starts accumulating useful decision history.

**Sources:**
- [GitHub Discussion #152 — Evolving specs](https://github.com/github/spec-kit/discussions/152)
- [GitHub Issue #889 — Support for Jira/Azure DevOps](https://github.com/github/spec-kit/issues/889)
- [GitHub Issue #299 — Phases too coupled](https://github.com/github/spec-kit/issues/299)
- [Scott Logic — Putting Spec Kit Through Its Paces](https://blog.scottlogic.com/2025/11/26/putting-spec-kit-through-its-paces-radical-idea-or-reinvented-waterfall.html)
- [Marmelab — SDD: The Waterfall Strikes Back](https://marmelab.com/blog/2025/11/12/spec-driven-development-waterfall-strikes-back.html)
- [Martin Fowler — SDD Tools Comparison](https://martinfowler.com/articles/exploring-gen-ai/sdd-3-tools.html)
- [InfoQ — SDD adoption at Enterprise Scale](https://www.infoq.com/articles/enterprise-spec-driven-development/)
