# CONSTITUTION_GUIDE — How to write and evolve a project constitution

> A deep dive into `.specify/memory/constitution.md`. What it really is, how to write one, and how to keep it in working shape. A real-world example of how the document evolved over 12 months on a project.

---

## What a Constitution actually is

Not "team marketing", but an **executable contract**. Every principle has to pass a simple test:

> If I violate it in `/speckit.plan`, will it show up in the Constitution Check?

If not — it's not a principle, it's a slogan. Delete it.

A constitution describes **technical engineering invariants** — the things that don't change from feature to feature. Specifically — process-agnostic to the means of implementation:

- **Test-First** — a process (tests *before* code). Agnostic to whether it's pytest, jest, or RSpec.
- **API-First** — a process (contract *before* implementation). Agnostic to OpenAPI / GraphQL / gRPC.
- **Observability** — a practice (every endpoint emits structured data). Agnostic to structlog / zap / winston.

In other words, it's about **"how we make engineering decisions"**, not "what stack we use".

### What a Constitution does **NOT** describe

| Doesn't describe | Where it goes |
|------------------|---------------|
| A specific stack (FastAPI, React) | `plan.md` |
| Specific libraries (pytest, structlog) | `plan.md` |
| Architectural decisions (CQRS, event sourcing) | ADR + `plan.md` |
| Code organization (folder structure) | Templates + linters |
| Scrum process, sprint cadence | Team handbook |
| Code review rules | CODEOWNERS + branch protection |

A constitution is about the **engineering discipline** your AI agent and your engineers follow on every feature, regardless of *which* feature you're building or *which* tools you're using.

---

## Anatomy of a working principle

Every principle has three parts: **rule**, **rationale**, **how to apply**.

```markdown
## Principle 3: API-First (NON-NEGOTIABLE)

**Rule**: Every new HTTP/gRPC endpoint MUST have its OpenAPI/proto schema
written and reviewed before any handler code is committed. The schema is
the source of truth — clients are generated from it, not the other way around.

**Rationale**: Three out of last five integration bugs in 2025 came from
client/server schema drift discovered only at runtime. Forcing schema-first
catches these in PR review, not in production.

**How to apply**:
- /speckit.plan MUST place the OpenAPI fragment in `specs/<feature>/contracts/`
  before listing handler tasks in /speckit.tasks.
- PR review checklist: "Is contracts/*.yaml updated?" must be ✅ before merge.
- Constitution Check fails if `Phase 1 — Design & Contracts` doesn't list
  a contract artifact for endpoint changes.
```

### Why all three parts are mandatory

**Without rationale** — in a month the principle looks like the author's arbitrary whim, in a year newcomers will ignore it. Rationale should be a concrete fact — an incident, a number, a real problem.

**Without "how to apply"** — the principle is not verifiable. The Constitution Check has no way to validate it, and the agent has no way to follow it.

**Without a rule** — it's just a description. A principle has to be declarative and testable.

### Antipattern — a principle that doesn't work

```markdown
## Principle 7: Quality

**Rule**: We value high-quality code.
```

Can't verify it, can't violate it — throw it out.

---

## MUST vs SHOULD vs MAY — don't overuse MUST

The most common mistake: a team writes eight principles, all MUST. Within a sprint or two they violate half of them, and MUST becomes meaningless.

**Rule of thumb**: 4–6 real MUSTs, the rest are SHOULD or MAY. If violating a principle won't block a merge — it's not a MUST.

| Level | Semantics | Constitution Check action |
|-------|-----------|---------------------------|
| **NON-NEGOTIABLE** | Absolute law | Blocks merge with no discussion |
| **MUST** | Required; exceptions must be justified | Either fixed, or moved to Complexity Tracking |
| **SHOULD** | Recommended | Flagged with ⚠️, deferred is allowed |
| **MAY** | Optimal, not required | Guideline only |

---

## Standard catalog of principles

From the public spec-kit examples (`spec-driven.md` from GitHub) — a starting set of categories:

- **Library-First** — new functionality is built as a standalone module with a public interface
- **CLI Interface Mandate** — every module has a CLI wrapper (text-in/text-out) so it's testable in isolation
- **Test-First** — tests are written before code; if the code passes the test on the first try, the test was weak
- **Integration-First Testing** — integration tests are more important than unit tests; mocks are limited
- **Simplicity** — adding dependencies requires justification
- **Anti-Abstraction** — premature abstraction is forbidden (DRY only after the third occurrence)
- **Observability** — every service writes structured logs with trace_id; metrics on critical paths
- **Versioning** — semver on the API; breaking changes only with a major bump

This is not "take everything", it's a stock catalog. A real constitution ideally contains 5–8 principles, of which 3–4 are specific to your project.

---

## How to write one from scratch — the process

The biggest mistake — *"we sat the tech lead down and they wrote the principles in 2 hours"*. A week later the team silently ignores half of them. There's a better way.

### Step 1 — Pilot (1–2 features without a constitution)

A champion + 1–2 engineers go through the full spec-kit cycle on real tasks *without* `/speckit.constitution`. They write down what hurts. They also write down what worked well. The output is a list of 10–15 "pain points" and "good vibes".

### Step 2 — Drafting workshop (2 hours)

Tech lead + staff + 1 product person sit down with this list and ask: *"what principle, if it had been in force, would have prevented this pain point?"* That gives you 6–10 candidates.

Drop duplicates, look at the standard catalog — maybe something common is phrased better there.

### Step 3 — Run `/speckit.constitution` (15 minutes)

In the agent chat — a detailed prompt. **Not** a one-liner like "create some principles", but full context:

```
/speckit.constitution

Context: B2B SaaS on Python 3.11 + FastAPI on the backend,
React 18 + TypeScript on the frontend, Postgres 16, monorepo.
Team — 6 engineers (3 fullstack, 2 backend, 1 frontend).

Create Project Constitution v1.0.0 with the following principles:

1. Test-First (NON-NEGOTIABLE)
   Rule: all new backend modules must have pytest tests; coverage on new
   or modified files ≥ 75%.
   Rationale: on the previous project the team had 4 regressions per quarter
   in core domain that unit tests would have caught; one made it to production
   and led to a 02:00 UTC rollback.
   How to apply: /speckit.tasks places test tasks BEFORE impl tasks;
   CI fails when coverage diff < 75%.

2. Type Safety (MUST)
   Rule: backend MUST type-check under mypy --strict; frontend — tsc --strict.
   Any/type:ignore/as any require an inline comment with justification.
   Rationale: 2 of 4 incidents in the previous project were runtime type errors
   that strict typing would have caught at PR review.
   How to apply: pre-commit hook + CI gates.

3. API-First (MUST)
   Rule: every new endpoint has an OpenAPI fragment in specs/<feature>/contracts/
   BEFORE handler code. TS clients are generated from OpenAPI via openapi-typescript.
   Rationale: 1 of 4 incidents — schema drift between backend and frontend.
   How to apply: /speckit.plan Phase 1 MUST produce contracts/*.yaml.

4. Observability (MUST)
   Rule: every endpoint and background task emits a structured log entry
   (structlog) with request_id, actor_id, duration_ms, outcome.
   Rationale: previous incident #4 — a slow endpoint that took 3 hours to
   debug because we had no duration_ms metrics.
   How to apply: middleware registered globally; /speckit.tasks Phase 6
   has a "verify structured logs" task.

5. Performance Budget (SHOULD)
   Rule: list endpoints < 300ms p95 on 10K rows; detail endpoints < 150ms p95;
   frontend FCP < 1.5s.
   Rationale: without an explicit budget, regressions can sit unnoticed for weeks.
   How to apply: perf-test task in Phase 6 for list/aggregation endpoints.

6. Simplicity (SHOULD)
   Rule: a new runtime dependency requires an RFC in docs/rfc/.
   Dev dependencies (linter, test runner) — no RFC.
   Rationale: package.json and pyproject.toml grow fast; we pay an
   audit/CVE/upgrade tax on deps we don't use.
   How to apply: /speckit.plan research.md MUST show "Alternatives
   considered" including the stdlib option for new deps.

Governance:
- Changes to MUST/NON-NEGOTIABLE — RFC in docs/rfc/<num>-<slug>.md + 2 staff approvals
- SHOULD/MAY — 1 staff approval
- Wording fix (typo, example) — 1 approval, PATCH bump
- Versioning: PATCH/wording, MINOR/new principle, MAJOR/removal of MUST
- Quarterly review at retro

Ratification date: today.
Last amended date: today.
Version: 1.0.0.
```

Yes — this is a long prompt. That's fine. The constitution is the most important document in the project, so 15 minutes spent on the prompt is justified.

### Step 4 — Review what was generated

Look the file over carefully:

1. **No `[ALL_CAPS]` placeholders** left (the agent should have replaced them; sometimes it skips one).
2. **Version is `1.0.0`**, the `ratification_date` and `last_amended_date` are today's date in ISO format.
3. **Sync Impact Report** is at the top and makes sense.

If something is off — run the command again with a correction:

```
/speckit.constitution Reword Principle 5 (Performance Budget) more precisely:
explain how p95 is measured (testing fixtures in CI with seed data).
```

The command modifies the file (instead of writing it from scratch) and bumps the PATCH version (1.0.0 → 1.0.1).

### Step 5 — Circulation in the team (3–5 working days)

The file is open for comments from the whole team in the PR. Cycle: "comment → edit → run `/speckit.constitution` again with the updated wording → bump".

### Step 6 — Adjust dependent templates

The Sync Impact Report said templates need to be updated. That means `.specify/templates/plan-template.md` (and `spec-template.md`, `tasks-template.md`) now need to include a new Constitution Check table with your principles.

You either edit the templates by hand or ask the agent:

```
Update `.specify/templates/plan-template.md` — add table rows for all 6
principles to the Constitution Check section.
```

This is a one-time job. After that, every `/speckit.plan` will use your principles in the Constitution Check.

### Step 7 — Ratification commit

```bash
git add .specify/
git commit -m "chore: ratify Project Constitution v1.0.0

Initial constitution with 6 principles:
- Test-First (NON-NEGOTIABLE)
- Type Safety (MUST)
- API-First (MUST)
- Observability (MUST)
- Performance Budget (SHOULD)
- Simplicity (SHOULD)

Constitution Check enforced via /speckit.plan."
```

PR for ratification — get the whole team to approve. This isn't bureaucracy: the team's signature = the team has actually read it and agreed to follow it.

---

## How to keep it alive

Now the hardest part — after ratification the constitution starts to **drift**. The team changes the stack, new roles are added, you find out a principle is too strict. Without active maintenance, in six months it becomes a "mythologized document" that nobody reads.

### Tools spec-kit provides

**Sync Impact Report.** On every run of `/speckit.constitution` the agent adds an HTML comment at the top of the file:

```html
<!-- Sync Impact Report
Modified: Principle 3 (API-First) — added gRPC requirement
Added principles: none
Removed principles: none
Templates needing update: plan-template.md (Constitution Check section)
Version bump: 1.2.0 → 1.3.0 (MINOR — broader scope)
-->
```

This is a reminder: when something changed in the constitution, you should check the templates and update them if needed.

**Constitution Check in `/speckit.plan`.** If you've embedded your constitution in the template, it's *actively* checked on every feature. That's the same mechanism that prevents it from becoming a ghost document.

> ⚠️ If your Constitution Check never fails — either the principles are too soft, or the template doesn't enforce them. Verify once per quarter that `plan.md` files actually contain a Constitution Check table with marks.

### Amendment workflow

```
docs/rfc/0007-loosen-test-coverage-rule.md
├── Context: why we're proposing a change
├── Current rule: quote from constitution.md
├── Proposed change: new text
├── Impact: which active specs this affects
├── Migration: do we need to rewrite anything in open specs
└── Approval: 2 staff sign-off
```

After the RFC is merged, you run `/speckit.constitution` with the prompt "apply RFC-0007" — the agent makes the edit and produces the Sync Impact Report.

**Version bump per semver:**

| Change | Bump | Example |
|--------|------|---------|
| Wording, fix typo, added example | PATCH | 1.2.0 → 1.2.1 |
| New principle added, scope expanded | MINOR | 1.2.1 → 1.3.0 |
| Principle removed, MUST redefined, governance changed | MAJOR | 1.3.0 → 2.0.0 |

### Quarterly review

At retro, once per quarter — spend 30 minutes opening the constitution together and asking two questions about each principle:

1. *Are we actually following it?* If not — either the principle is irrelevant (delete it), or there's friction in execution (find it and fix it).
2. *Has Constitution Check in /plan actually enforced it in the last 3 months?* If not — either there's nothing to check in the plan, or the template is broken.

This is the ritual that distinguishes a "living" constitution from a "mythologized document".

---

## Change scenarios

### Scenario A — The stack changes

For example, you migrate from Express to FastAPI. The constitution *should not* mention specific libraries or frameworks — that's the job of templates and `plan.md`. If your constitution says "MUST use FastAPI" — that's a warning sign, rewrite it as "MUST use a typed Python web framework" (or just delete it, because it's a detail that lives in `plan.md`).

### Scenario B — A practice changes

For example, you introduce a pre-commit hook that wasn't there before. That's an addition to the "How to apply" section of the relevant principle, without changing the rule itself:

```diff
  ## Principle 1: Test-First
  ...
  **How to apply**:
+ - Pre-commit hook runs `pytest -m 'fast'` automatically.
  - CI runs full pytest suite on PRs.
```

That's a PATCH bump.

### Scenario C — A principle turned out to be excessive

For example, you set "100% coverage" — and after a quarter you realize it's eating time, has no real value, and "75% on new modules" is enough. That's a MAJOR bump because it changes MUST behavior.

RFC with concrete analysis:

> In Q1 2026 we spent roughly 18 engineer-hours on coverage gaps that no real bug ever touched. We propose reducing to 75% on new modules; legacy stays at the current level.

Without an RFC — don't change it. Otherwise the precedent "principles can be quietly weakened" undermines the constitution at every level.

---

## A real example of evolution over 12 months

So you can see what a constitution looks like over time:

### v1.0.0 (start, month 1)

```
6 principles: Test-First (NON-NEGOTIABLE), Type Safety (MUST),
API-First (MUST), Observability (MUST), Performance (SHOULD),
Simplicity (SHOULD).
Ratification: 2026-04-28.
```

### v1.0.1 (week 3)

PATCH — fix typo, added an example to the Test-First section.

### v1.1.0 (month 2)

MINOR — added Principle 7: Performance perf-budget reflection (SHOULD), because half a sprint in we noticed patterns around slow endpoints. Came in via RFC-002.

### v1.1.1 (month 3)

PATCH — in the Observability section, replaced generic "structured logs" with a concrete "structlog with `request_id`", after retro revealed that different modules were logging in different formats.

### v2.0.0 (month 6)

MAJOR — Test-First downgraded from NON-NEGOTIABLE to MUST (this is a relaxation), replacing a stricter NON-NEGOTIABLE blocking gate. RFC-007 with the analysis: "3/4 exceptions to Test-First in Q2 were justified, didn't deserve a blocker". We allow exceptions through Complexity Tracking.

### v2.1.0 (month 9)

MINOR — added Principle 8: Security Review (MUST) as a reaction to a security incident.

### v2.2.0 (month 11)

MINOR — Principle 5 (Performance Budget) promoted from SHOULD to MUST. RFC-012: in one quarter there were 3 regressions > 25% that SHOULD didn't block.

### v2.2.1 (month 12)

PATCH — updated Performance budget thresholds based on real production metrics (from Grafana).

**After 12 months the constitution has 8 principles, version 2.2.x, with the amendment history in git log and `docs/rfc/`. That's a living constitution.**

---

## Most common mistakes

### Marketing-style wording

❌ "We value engineering excellence", "Quality is everyone's responsibility".

Throw it out, replace it with concrete rules. If you can't describe how the principle fails the Constitution Check, it's dead weight.

### Overusing MUST

❌ Everything MUST → nothing is MUST. Real-world clunker on the repair lift: in one sprint the team violates 4 of 8 MUSTs, everyone says "let's discuss it later", and then nothing changes.

✅ Keep 4–6 real MUSTs. The rest are SHOULDs.

### Constitution as waterfall

❌ Write it once and never update it. A year later the document looks like it belongs to a different project.

✅ Quarterly review + Sync Impact Report on every change.

### Constitution by Tech Lead alone

❌ Without team participation — others don't see the process, don't feel ownership, don't follow it.

✅ A drafting workshop with 3–5 people involved, public PR for ratification.

### Constitution without rationale

❌ Principles without justification. In 6 months nobody understands *why*, and a cargo cult sets in.

✅ Every principle has a concrete fact-rationale, ideally with an incident or numerical metric.

### Toothless Constitution Check

❌ The `/speckit.plan` template doesn't actually check anything.

✅ Verify `.specify/templates/plan-template.md` — the Constitution Check section should be a table with your principles, not the text "check whatever applies".

### Principles that are too specific

❌ "MUST use SQLAlchemy 2.0 with async session". That's not a principle, it's a `plan.md` detail.

✅ The principle is "MUST use an async ORM with explicit transaction boundaries".

### Constitution out of sync with templates

❌ Sync Impact Report is ignored, templates show old wording, and `plan.md` Constitution Check references principle v1.0 even though the constitution is already v1.3.

✅ On every `/speckit.constitution` run, always check whether templates need updating — the Sync Impact Report tells you which.

---

## Pre-ratification checklist for v1.0.0

- [ ] 4–6 real MUSTs, the rest SHOULD/MAY
- [ ] Each principle has rule + rationale + how to apply
- [ ] Rationale contains a concrete fact (incident, number, real example)
- [ ] Principles are technology-agnostic (no mention of FastAPI, React, etc.)
- [ ] No `[ALL_CAPS]` placeholders
- [ ] Frontmatter contains version, ratification_date, last_amended_date
- [ ] Sync Impact Report generated
- [ ] Templates (`spec-template.md`, `plan-template.md`, `tasks-template.md`) updated for the new principles
- [ ] Governance section describes the amendment process
- [ ] Ratification PR approved by a staff engineer
- [ ] Team has read it and has no blocking objections

---

## What to read next

- **`SPEC-KIT-docs.md`** — section 8.5 Project Memory, how the constitution fits into the overall architecture.
- **`SCRUM_INTEGRATION.md`** — who writes the constitution in a team setting.
- **`SPEC_KIT_USE_CASES.md`** — Scenario 3 (Constitution Setup).

> 🚀 **Most important advice**: don't try to write the "perfect" v1.0 constitution. Write a "good enough" v1.0, ratify it, and let it evolve through RFCs. After 6 months of real experience you'll know *exactly* what the team needs, far better than you do at the start.
