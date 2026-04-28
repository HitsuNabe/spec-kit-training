# SPEC-KIT — Methodology and Tooling Reference

> This document is the *theoretical* foundation of the course. Read it in full before tackling `SPEC_KIT_WORKFLOW_GUIDE.md`. Without grasping the philosophy, all the commands will look like "extra bureaucracy".

---

## 1. What Spec-Driven Development is

**Spec-Driven Development (SDD)** — a methodology for building software with AI assistants in which the **specification** is the primary artifact, and code is the *result* of the specification.

> ❝ Code used to be king. In SDD, **code serves specifications**. ❞
> — `spec-driven.md`, github/spec-kit repository

### Principles

| Principle | What it means in practice |
|-----------|---------------------------|
| **Specifications as lingua franca** | The spec is the source of truth. Not a PRD in Confluence, not a Jira ticket — a concrete `spec.md` file in the repository. |
| **Executable specifications** | The spec must be precise enough that working code can be generated from it. No vague wording. |
| **Continuous refinement** | Validation runs *continuously*, not once at kickoff. |
| **Research-driven context** | The AI agent gathers information about dependencies, performance, security on its own → `research.md`. |
| **Bidirectional feedback** | Production metrics feed back into the specification, and the spec evolves. |
| **Branching for exploration** | You can have several implementations of the same specification in parallel. |

---

## 2. How SDD differs from other approaches

### Vibe Coding

> "Threw a prompt at GPT/Claude — got code — committed it".

| Property | Vibe Coding | SDD |
|----------|-------------|-----|
| Artifact | Code only | Spec → plan → tasks → code |
| Predictability | Low | High |
| Refactoring | Re-prompt from scratch | Change the spec → regenerate |
| Documentation | Missing | Built in by default |
| Code review | Looks at the code | Looks at the spec *and* the code |

### Classic waterfall / PRD-first

| Property | PRD-first | SDD |
|----------|-----------|-----|
| Is the PRD updated? | Once, then thrown away | A living artifact in git |
| Drift between docs and code | Always | Eliminated by having the PRD generate the code |
| Who writes the PRD | PM/BA | Engineer + AI with checklists |
| Are requirements testable? | Often `[NEEDS CLARIFICATION]` | The template forbids it |

### Which problems SDD solves

1. **Drift between specification and code** — there's simply none, because the spec generates the code.
2. **AI hallucinations** — the template requires explicit `[NEEDS CLARIFICATION]` markers instead of "plausible-looking" guesses.
3. **Over-engineering** — `Constitution Check` gates block excess complexity.
4. **Cheap pivots** — change a requirement → regenerate `plan.md` and `tasks.md` instead of rewriting code.
5. **Brownfield development** — adding a feature to an existing codebase becomes manageable, because the spec describes integration with the existing architecture.

---

## 3. Spec-kit architecture

Spec-kit consists of two parts:

### 3.1. The `specify` CLI

A Python application that:

- Creates the `.specify/` directory with scripts and templates.
- Configures integration with the chosen AI agent (≥30 supported).
- Registers slash commands.

### 3.2. A library of templates and prompts

All commands are markdown files in `.specify/templates/commands/` that the AI agent reads and executes. The templates contain:

- Spec/plan/tasks file structure (placeholder slots).
- **Guard rails** — banning stack mentions in the spec, capping the number of `[NEEDS CLARIFICATION]` markers, requiring measurable success criteria.
- Instructions for the AI agent (how to ask clarifying questions, how to fill in tables, etc.).

### 3.3. Supported agents (as of v0.8.1)

| Key | Agent |
|-----|-------|
| `claude` | Claude Code (skills mode by default) |
| `copilot` | GitHub Copilot (IDE + CLI) |
| `cursor-agent` | Cursor |
| `gemini` | Gemini CLI |
| `codex` | OpenAI Codex CLI (skills) |
| `windsurf` | Windsurf |
| `qwen` | Qwen Code |
| `opencode`, `goose`, `kiro-cli`, `tabnine`, `vibe`, `forge`, `pi`, `qodercli`, ... | 20+ others |
| `generic` | Any custom agent via `--commands-dir` |

> 💡 In Codex CLI skills mode, the command prefix is **`$speckit-*`** (with a dollar sign), not `/speckit.*`.

---

## 4. Commands (slash commands) — Reference

All commands are invoked **in the AI agent chat**, not in the shell.

### 4.1. `/speckit.constitution` — Project principles

Creates/updates `.specify/memory/constitution.md`.

**What it generates:** a document with principles (Library-First, Test-First, Observability, Simplicity, etc.), governance rules, and a semver version.

**When to invoke:**
- Once at project start.
- When team standards change (then bump the version: MAJOR/MINOR/PATCH).

**Example:**

```
/speckit.constitution Create principles: code quality (80%+ test coverage),
TDD for the core domain, UX consistency via the Figma design system,
performance <200ms p95. Governance: MUST principles are changed only
through RFC + 2 reviews.
```

**Artifact:** `.specify/memory/constitution.md` with frontmatter:

```markdown
---
version: 1.0.0
ratification_date: 2026-04-28
last_amended_date: 2026-04-28
---
```

---

### 4.2. `/speckit.specify` — WHAT and WHY

Creates `specs/<NNN-feature-slug>/spec.md`.

**Behavior:**
- Generates a kebab-case slug (`001-photo-albums`, `002-search-filter`).
- Creates a git branch (optionally via a hook).
- **Forbids** mentioning stack/frameworks/APIs in spec.md.
- **At most 3 `[NEEDS CLARIFICATION]` markers** (priority order: scope → security → UX → tech).
- Auto-generates `specs/<feature>/checklists/requirements.md` for self-validation.

**Example:**

```
/speckit.specify Add the ability to group items by tags to the application.
A user can create a tag, assign multiple tags to one item, and filter items
by tags. Tags are unique per user.
```

**The `spec.md` artifact contains:**
- User Stories with priorities P1/P2/P3
- Functional Requirements (FR-001, FR-002, ...)
- Success Criteria (measurable, technology-agnostic)
- Edge Cases
- Key Entities (description, not schema)
- Assumptions

---

### 4.3. `/speckit.clarify` — Reducing ambiguity

Scans `spec.md` across 11 categories (Functional Scope, Domain & Data Model, UX Flow, NFR, Integration, Edge Cases, Constraints, Terminology, Completion Signals, Misc).

**Behavior:**
- Asks up to **5 targeted questions**, one at a time.
- Each question is multiple choice (2–5 options) or a free-form answer ≤5 words.
- One option is marked as **Recommended** with justification.
- After every answer, immediately edits spec.md (atomic overwrite) and adds `## Clarifications / ### Session YYYY-MM-DD`.

**Example:**

```
/speckit.clarify Focus on security and performance requirements.
```

> ⚠️ **Strong recommendation**: always run `/speckit.clarify` before `/speckit.plan`. The exception is an exploratory spike.

---

### 4.4. `/speckit.plan` — Technical plan

Reads `spec.md` + `constitution.md`, generates:
- `plan.md` — Technical Context, Constitution Check, Project Structure, Complexity Tracking
- `research.md` — Decision / Rationale / Alternatives for every `[NEEDS CLARIFICATION]` and every new technology choice
- `data-model.md` — entities, fields, relationships, validation, state transitions
- `contracts/` — API specs (OpenAPI), GraphQL, event schemas, CLI grammars
- `quickstart.md` — manual validation scenarios (definition of done)

**Constitution Check gates** run twice: before the Research phase and after the Design phase. Any MUST principle violation must either be fixed or justified in **Complexity Tracking**.

**Example:**

```
/speckit.plan Backend: FastAPI + SQLModel + Alembic + PostgreSQL.
Frontend: React 18 + TypeScript + TanStack Query.
Auth — existing JWT via FastAPI-Users.
Tests — pytest + Playwright for e2e.
```

> 💡 **Tip**: after `/plan`, ask the agent: "Walk through the plan and find areas that need additional research. For each, dispatch a parallel research task and update `research.md` with concrete library versions".

---

### 4.5. `/speckit.tasks` — Decomposition

Reads plan.md, spec.md (user stories with P1/P2/P3), data-model.md, contracts/, research.md → generates `tasks.md`.

**Strict format:**

```markdown
- [ ] T001 Create project structure per implementation plan
- [ ] T002 [P] Configure linting and formatting tools
- [ ] T003 Add Tag SQLModel in backend/app/models/tag.py
- [ ] T012 [P] [US1] Implement TagService.create_tag in backend/app/services/tag.py
- [ ] T013 [US1] Add POST /api/v1/tags endpoint in backend/app/api/v1/tags.py
```

- `T001` — sequential id
- `[P]` — task can run in parallel (different files, no dependencies)
- `[US1]` — which user story it belongs to (only in US phases)
- The description **must contain the file path**

**Phase structure:**

1. **Phase 1 — Setup** (project init, deps, lint)
2. **Phase 2 — Foundational** (BLOCKS all stories — schema, auth, routing)
3. **Phase 3+ — Per User Story** (P1, then P2, then P3) — each story is an independent MVP increment
4. **Final Phase — Polish** (docs, perf, security hardening)

Each phase ends with a **Checkpoint** for independent validation.

> 💡 Tests are **OPTIONAL** by default. If you want TDD, ask the agent in the prompt: "Use TDD: write failing tests first".

---

### 4.6. `/speckit.analyze` — Cross-artifact audit (read-only)

Runs **between `/tasks` and `/implement`**. Does not modify files.

**Check categories:**
- **A. Duplication** — near-duplicate requirements
- **B. Ambiguity** — vague adjectives ("fast", "scalable")
- **C. Underspecification** — verbs without a measurable outcome
- **D. Constitution alignment** — conflicts with MUST principles (auto-CRITICAL)
- **E. Coverage gaps** — requirements without tasks, tasks without requirements
- **F. Inconsistency** — terminology drift, conflicting tech choices

**Output:** a markdown report with a findings table (severity: CRITICAL / HIGH / MEDIUM / LOW), Coverage Summary, Metrics, Next Actions.

> ⚠️ **CRITICAL findings = blockers**. Fix them before running `/implement`.

---

### 4.7. `/speckit.implement` — Execution

**Behavior:**
- Verifies the status of every checklist in `specs/<feature>/checklists/`. If any are incomplete, asks for confirmation.
- Generates/updates `.gitignore`, `.dockerignore`, etc. based on the detected stack.
- Executes tasks **by phase**:
  - Sequentially where there are dependencies
  - In parallel for `[P]` tasks
- After every completed task, **marks `[X]`** in tasks.md.
- Halts on a non-parallel failure.
- At the end — final validation against the spec.

---

### 4.8. `/speckit.checklist` — Unit tests for English

Generates a domain-specific quality checklist that validates **the requirement text itself** (clarity, completeness, consistency, coverage, edge cases) — not the implementation.

Asks up to 3 clarifying questions to tune the checklist for the audience (lightweight pre-commit / formal release gate).

**Example:**

```
/speckit.checklist Build a UX checklist to validate spec.md
focused on accessibility and error states.
```

---

### 4.9. `/speckit.taskstoissues` — Tasks → GitHub Issues

Via the GitHub MCP, creates one issue per task from tasks.md. Before creating, it checks that the remote URL points at a GitHub repo (it will refuse to sync to an unrelated repo).

---

## 5. Artifacts — full structure

```
.
├── .specify/
│   ├── memory/
│   │   └── constitution.md          # Principles, version, ratification date
│   ├── scripts/
│   │   ├── bash/                    # check-prerequisites.sh, common.sh,
│   │   │                            # create-new-feature.sh, setup-plan.sh,
│   │   │                            # update-agent-context.sh
│   │   └── powershell/              # *.ps1 mirrors
│   ├── templates/
│   │   ├── constitution-template.md
│   │   ├── spec-template.md
│   │   ├── plan-template.md
│   │   ├── tasks-template.md
│   │   ├── checklist-template.md
│   │   └── commands/                # 9 commands as markdown prompts
│   ├── feature.json                 # active feature (for downstream commands)
│   ├── init-options.json            # branch_numbering: sequential|timestamp
│   └── extensions.yml               # registry of extension hooks
├── specs/
│   └── 001-feature-name/
│       ├── spec.md                  # WHAT & WHY (P1/P2/P3, FR-, SC-)
│       ├── plan.md                  # Technical Context, Constitution Check
│       ├── research.md              # Decision/Rationale/Alternatives
│       ├── data-model.md            # Entities, fields, relationships
│       ├── contracts/               # API/event/CLI specs
│       ├── quickstart.md            # Manual validation scenarios
│       ├── checklists/              # requirements.md (auto) + custom
│       └── tasks.md                 # Phase 1, 2, 3+, Polish
└── (agent context)
    ├── CLAUDE.md  /  GEMINI.md  /  AGENTS.md  /  QWEN.md
    ├── .github/copilot-instructions.md
    ├── .cursor/rules/specify-rules.mdc
    └── .windsurf/rules/specify-rules.md
```

> 💡 **The active feature** is determined via `.specify/feature.json` (since v0.8.1), so the commands work even when the git branch name has drifted from the specs folder.

---

## 6. Pipeline workflow (how artifacts flow)

```
┌──────────────┐
│ constitution │  (once; bumps semver on changes)
└──────┬───────┘
       │
       ▼
┌──────────────┐     ┌──────────────┐
│   specify    │────▶│   clarify    │  (recommended before /plan)
└──────┬───────┘     └──────┬───────┘
       │                    │
       └────────┬───────────┘
                ▼
        ┌──────────────┐     ┌──────────────┐
        │     plan     │────▶│  checklist   │  (optional)
        └──────┬───────┘     └──────────────┘
               │
               ▼
        ┌──────────────┐
        │    tasks     │
        └──────┬───────┘
               │
               ▼
        ┌──────────────┐
        │   analyze    │  (read-only; CRITICAL = blocker)
        └──────┬───────┘
               │
               ▼
        ┌──────────────┐
        │  implement   │
        └──────────────┘
```

---

## 7. Installation — details

### Prerequisites

- Linux / macOS / Windows (PowerShell 7+ for offline; WSL is no longer required)
- **Python 3.11+**
- **uv** (recommended) or **pipx**
- **Git**
- An AI agent (Claude Code / Copilot / Cursor / Gemini / Codex / ...)

### Installation options

**One-shot via uvx** (no permanent install):
```bash
uvx --from git+https://github.com/github/spec-kit.git@vX.Y.Z specify init <PROJECT>
uvx --from git+https://github.com/github/spec-kit.git specify init . --integration claude
uvx --from git+https://github.com/github/spec-kit.git specify init --here --integration copilot
```

**Persistent install** (recommended for regular work):
```bash
uv tool install specify-cli --from git+https://github.com/github/spec-kit.git@vX.Y.Z
# or
pipx install git+https://github.com/github/spec-kit.git@vX.Y.Z

specify version
specify check
specify integration list
```

> ⚠️ The only official source is `github.com/github/spec-kit`. Any package on PyPI named `specify-cli` is *NOT* from GitHub.

### `specify init` flags

| Flag | Purpose |
|------|---------|
| `<name>` or `.` | New folder / current folder |
| `--here` | Initialize in the current directory |
| `--force` | Skip the confirmation for a non-empty folder |
| `--integration <key>` | Pick the agent (`claude`, `copilot`, `gemini`, ...) |
| `--integration-options "<opts>"` | E.g. `--skills` for Codex/Copilot |
| `--script sh\|ps` | Force the script type |
| `--ignore-agent-tools` | Skip checking whether the agent's CLI is installed |
| `--no-git` | Skip git init (deprecated, removed in v0.10.0) |
| `--ai <key>` | Legacy alias for `--integration` |
| `--preset <name>` | Community preset (e.g., `healthcare-compliance`) |

### Air-gapped install (for closed enterprise networks)

On a connected machine:
```bash
git clone https://github.com/github/spec-kit.git
cd spec-kit
pip install build
python -m build --wheel --outdir dist/
pip download -d dist/ dist/specify_cli-*.whl
```

Move `dist/` to the target machine, then:
```bash
pip install --no-index --find-links=./dist specify-cli
```

(The same OS and Python version are required.)

---

## 8. Best Practices

### DO

- ✅ **Specs technology-free** — the stack goes in `plan.md`, not `spec.md`.
- ✅ **Always `/clarify` before `/plan`** (exception — an exploratory spike).
- ✅ **Measurable success criteria** — "User can complete checkout in under 3 minutes", not "API responds <200ms".
- ✅ **Run `/analyze` before `/implement`** — CRITICAL findings = blocker.
- ✅ **Phased implementation** — ship P1 → validate the MVP → then P2/P3. Prevents context saturation.
- ✅ **Constitution as a real contract** — version it, update it, reference it.
- ✅ **One branch — one feature** — `specs/001-foo/` ↔ branch `001-foo` ↔ one PR.
- ✅ **Pin a tag (`@vX.Y.Z`)** instead of `main` for CI stability.
- ✅ **Push back** — ask the agent: "Why did you add this component? Which requirement does it trace to?"

### DON'T

- ❌ Don't accept the first draft of the spec as final — iterate via `/clarify`.
- ❌ Don't mix the tech stack into `spec.md`.
- ❌ Don't skip `constitution.md` — without it the `Constitution Check` is toothless.
- ❌ Don't leave more than 3 `[NEEDS CLARIFICATION]` markers — prefer defaults in the Assumptions section.
- ❌ Don't allow tasks without a `[Story]` label or without a file path.
- ❌ Don't run `/implement` with an incomplete checklist and don't auto-say "yes" — that breaks the quality gate.
- ❌ Don't let the agent research broad topics ("React in general"). Only narrow, targeted research.
- ❌ Don't ignore the Sync Impact Report after `/constitution` — dependent templates can quietly drift.

---

## 8.5. Project Memory — multi-tier architecture

Spec-kit maintains project "memory" at three tiers. Understanding this architecture is critical for using the tool correctly over the long term.

### Tier 1 — Invariants (slow-moving)

Files that describe *the project as a whole* and aren't tied to a specific feature:

- **`.specify/memory/constitution.md`** — engineering principles. The things that don't change from feature to feature (Test-First, Type Safety, API-First, etc.). Versioned by semver, with a ratification and amendment process. Details — in `CONSTITUTION_GUIDE.md`.
- **`.specify/init-options.json`** — spec-kit config for the project: `branch_numbering: sequential|timestamp`, script type (`sh`/`ps`).
- **`.specify/extensions.yml`** — registry of extension hooks (`before_specify`, `after_plan`, `before_implement`, etc.).
- **`.specify/feature.json`** — the currently active feature (for downstream commands). Since version 0.8.1, it works even when the branch name has drifted from the folder.

### Tier 2 — The agent's live context (fast-moving)

A file the AI agent **reads on every new message**. Depends on the agent:

| Agent | Context file |
|-------|--------------|
| Claude Code | `CLAUDE.md` |
| Gemini CLI | `GEMINI.md` |
| Codex / generic | `AGENTS.md` |
| Qwen Code | `QWEN.md` |
| GitHub Copilot | `.github/copilot-instructions.md` |
| Cursor | `.cursor/rules/specify-rules.mdc` |
| Windsurf | `.windsurf/rules/specify-rules.md` |

Spec-kit writes into this file between markers:

```markdown
<!-- SPECKIT START -->
Active feature: 002-search-and-filter
Stack: Python 3.11 + FastAPI 0.111 + React 18 + TypeScript 5.3
Database: PostgreSQL 16 (existing schema in app/models/)
Testing: pytest + Playwright
Recent decisions:
  - Use ILIKE for substring search (not tsvector, deferred to v2)
  - Custom useDebouncedValue hook (no use-debounce dep)
  - URL state via useSearchParams
Constitution: see .specify/memory/constitution.md (v1.0.0)
<!-- SPECKIT END -->
```

On every `/speckit.plan` this block is updated — the agent learns from the project's current state instead of starting from scratch. **Outside the markers, the file is yours**, you can write your own notes; spec-kit doesn't touch them.

### Tier 3 — Decision history (immutable)

`specs/` isn't "memory" in the classic sense — it's an **archive of engineering decisions** that accumulates over the life of the project.

```
specs/
├── 001-user-auth/
│   ├── spec.md         ← what was decided (WHAT)
│   ├── plan.md         ← Constitution Check, technical decisions
│   ├── research.md     ← why exactly that way (Decision/Rationale/Alternatives)
│   ├── data-model.md   ← entities at the time of the feature
│   └── contracts/      ← OpenAPI snapshot
├── 002-search-filter/
└── 003-tags/
```

Six months later a new person can read `research.md` from `001-user-auth` and understand *why* we chose JWT via FastAPI-Users instead of Auth0 — it's documented in the Decision/Rationale/Alternatives format. This is living archival memory the agent can read when answering questions like "how is auth done here?".

> ⚠️ **At scale (200+ specs)** the flat structure becomes a junk drawer without maintenance. See the separate document `SPECS_HYGIENE.md` on lifecycle states, archival pattern, domain grouping, and quarterly grooming ritual.

### How this works in practice

Scenario: you're away for three weeks, come back, and don't remember the context. You open Claude Code — the agent *already knows*:

1. It read `CLAUDE.md` — sees the current stack and a reference to the constitution.
2. On `/speckit.specify` for a new feature, it reads `constitution.md` and follows the principles.
3. In `/speckit.plan`, it reads previous `specs/*/data-model.md` so the new code integrates consistently with existing entities.
4. A question like "why do we do auth via JWT?" — the agent opens `specs/001-user-auth/research.md` and answers with concrete rationale.

In other words, project memory is **not a single file** but a structured filesystem. No database, no cloud state. Everything is portable, human-readable, and versioned alongside the code.

---

## 8.6. Advanced agent capabilities

Spec-kit uses host-agent capabilities (Claude Code, Codex, Cursor) more deeply than it might seem. What's actually built in:

### Skills mode (Claude / Codex / Kimi / Vibe)

For these agents, spec-kit installs by default **not** as a list of markdown prompts but as **agent skills**:

```
.claude/skills/
├── speckit-constitution/SKILL.md
├── speckit-specify/SKILL.md
├── speckit-clarify/SKILL.md
├── speckit-plan/SKILL.md
├── speckit-tasks/SKILL.md
├── speckit-analyze/SKILL.md
├── speckit-implement/SKILL.md
├── speckit-checklist/SKILL.md
└── speckit-taskstoissues/SKILL.md
```

Each skill is a separate directory with `SKILL.md` + frontmatter (name, description). What this gives you:

- **Progressive disclosure** — a skill is loaded into context only when the agent recognizes it should be used. It doesn't sit there permanently.
- **Token-efficient** — the full `/speckit.plan` prompt (300+ lines of instructions) doesn't eat context until you actually run plan.
- **Skill-by-skill isolation** — `/speckit.specify` doesn't "know" the full prompt for `/speckit.plan` — each is self-contained.

In Codex CLI skills mode the prefix is different: `$speckit-specify` instead of `/speckit.specify`.

For agents without skills (Copilot, Cursor, Gemini, Windsurf), commands install as markdown / TOML / YAML prompts — functionally the same, but without skills optimization.

### Sub-agent dispatching in `/speckit.plan`

The `templates/commands/plan.md` prompt includes a **direct pattern for the Task tool / sub-agents**:

```
### Phase 0: Outline & Research

1. Extract unknowns from Technical Context above:
   - For each NEEDS CLARIFICATION → research task
   - For each dependency → best practices task
   - For each integration → patterns task

2. Generate and dispatch research agents:

   For each unknown in Technical Context:
     Task: "Research {unknown} for {feature context}"
   For each technology choice:
     Task: "Find best practices for {tech} in {domain}"

3. Consolidate findings in research.md
```

What this means: in the Research phase your AI agent *actually* spawns several parallel sub-agents — each does its own research — and consolidates the results in `research.md`. Without this, on 5 unknowns you'd be paying 5x in tokens and time sequentially.

### Parallel `[P]` markers in `/speckit.implement`

`/speckit.tasks` marks tasks as `[P]` (parallelizable — different files, no dependencies), and `/speckit.implement` **uses** those markers:

```
[Phase 3] T009-T012 (US1)...
  T009-T011 (parallel) — pytest tests written, running...
  → 3 tests PASS in parallel ✅
```

This isn't theoretical parallelism — in the Claude Code chat you'll see the agent start 3–4 sub-agents for `[P]` tasks at once. Sequential ones (without `[P]`) run one after another.

### Hooks via `extensions.yml`

The extension mechanism (early-stage, but working):

```yaml
hooks:
  before_specify:
    - id: speckit_git_feature
      enabled: true
      extension: "speckit-core"
      command: "create_feature_branch"
      description: "Create a feature branch before /specify"

  after_implement:
    - id: my_post_test
      enabled: true
      optional: true
      extension: "my-extension"
      command: "run_smoke_tests"
      description: "Run smoke tests after /implement"
      prompt: "Run smoke tests?"
```

Extension points: `before_specify`, `after_specify`, `before_plan`, `after_plan`, `before_tasks`, `after_tasks`, `before_implement`, `after_implement`.

Hooks are **shell commands or custom extensions** that run before/after a spec-kit command. They can:

- Spawn your own sub-agent via MCP.
- Call a Python script with custom logic.
- Block the command if preconditions aren't met.

> 💡 **The most common problem** for beginners: the `before_specify` hook (creating a feature branch) **doesn't fire** — `extensions.yml` is missing or the hook is disabled. Fix — check `cat .specify/extensions.yml` and `.specify/scripts/bash/create-new-feature.sh`. If the files aren't there, run `specify integration upgrade` or create the branch manually before `/specify`.

### What's **not** built in

Honestly, about the limitations. Spec-kit remains primarily a **prompt library + skills**, not a full agent framework. What's missing:

- **Multi-agent orchestration with roles** (like CrewAI, AutoGen) — there's no concept of "Architect agent → Reviewer agent → Implementer agent". If you need that, build it via hooks + custom MCP servers.
- **Persistent agent memory across sessions** beyond markdown — there's no vector store, no semantic search across spec history. Integrate Pinecone / Weaviate via MCP yourself.
- **Background daemons / watchers** — there are no built-in triggers like "automatically run `/analyze` after a push". This is done via GitHub Actions.
- **Custom sub-agent definitions** — unlike Claude Code's native sub-agent system, spec-kit doesn't directly support this. But via hooks you can call your own agent.
- **Cross-feature analysis** — `/speckit.analyze` looks at one active feature. Analyzing "do these 3 features conflict architecturally" is a separate decision for the team (ADR process).

---

## 9. Reference: key files in the spec-kit repository

If you need to dig deeper:

- `spec-driven.md` — the methodology essay (~412 lines, MUST READ)
- `AGENTS.md` — how to add your own agent
- `CHANGELOG.md` — version history
- `templates/commands/{constitution,specify,clarify,plan,tasks,analyze,implement,checklist,taskstoissues}.md` — the command prompts themselves (can be modified locally via extensions)
- `templates/{constitution,spec,plan,tasks,checklist}-template.md` — artifact templates
- `docs/{installation,quickstart,upgrade,index}.md` — official documentation
- `src/specify_cli/__init__.py` — CLI entry point, all the flags
- `src/specify_cli/integrations/` — registry of 30+ agents

---

## 10. What to read next

### Foundation block — practice on the training project

1. **`SPEC_KIT_WORKFLOW_GUIDE.md`** — an end-to-end walkthrough of SDD-2 (Search & Filter) on the training project: from `/specify` to `/implement` with the full text of every artifact.
2. **`SPEC_KIT_USE_CASES.md`** — 9 scenarios by complexity level.
3. **`TASKS.md` / `TASKS_JIRA.md`** — a practical backend with 7 tasks for self-paced work.

### Deep-dive block — for teams and real projects

4. **`SCRUM_INTEGRATION.md`** — integrating spec-kit with Scrum ceremonies: grooming, planning, standup, demo, retro. Role matrix. Jira mapping. DoR/DoD.
5. **`CONSTITUTION_GUIDE.md`** — a deep dive into `.specify/memory/constitution.md`: how to write it, how to evolve it, a real example v1.0 → v2.x over 12 months.
6. **`SPECS_HYGIENE.md`** — maintaining `specs/` at scale (100+ features): lifecycle states, archival pattern, domain hierarchy, manifest, quarterly grooming, scripts.

> 🚀 **Ready? Move on to `SPEC_KIT_WORKFLOW_GUIDE.md`** — there we'll show what this all looks like in practice.
