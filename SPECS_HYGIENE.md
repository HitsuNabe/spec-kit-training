# SPECS_HYGIENE — How to keep `specs/` tidy at scale

> On a small project (10–30 features) a flat `specs/` structure works without issues. On medium and large ones (100+ features per year), without active care the directory turns into a junk drawer. This document covers lifecycle states, the archival pattern, domain hierarchy, manifest files, the quarterly grooming ritual, and scripts for automation.

---

## Why this matters

With 200+ folders in `specs/` you'll run into:

- `ls specs/` becomes useless — too many lines
- `grep -r` slows down and drowns in noise
- 60–70% of features are already deployed and nobody comes back to their specs — but they sit there and distract
- New engineers don't know where the "live" specs are versus the historical ones
- Specs that were *cancelled* (the feature was dropped but the folder stayed) get confused with active ones
- Specs that were *replaced* (a new version of the feature) coexist with the old one — and it's unclear which is the source of truth

By default, spec-kit does **nothing** about this. It's a question of **team curation discipline**, not tooling.

---

## Lifecycle statuses

Every `spec.md` carries a `status` field in its frontmatter:

```yaml
---
status: active           # work in progress
# OR
status: implemented      # shipped, but still "fresh"
# OR
status: archived         # old, moved to _archive/
# OR
status: superseded       # replaced by a new version, no longer in use
# OR
status: dropped          # cancelled
---
```

Additional fields for context:

```yaml
---
status: implemented
implemented_date: 2026-04-15
jira_id: PROJ-1234
# or:
status: superseded
superseded_by: 042-new-search-v2
# or:
status: dropped
archived_reason: "feature scope rejected by product"
---
```

This mechanically gives you an answer to "what's alive here and what's dead" via scripts or just plain `grep`.

---

## Physically moving the archive

Once a quarter (at retro or as a separate ritual), walk through specs with status `implemented` older than N months and physically move them:

```
specs/
├── _archive/
│   ├── 2025-Q4/
│   │   ├── 001-user-auth/
│   │   └── 002-billing-mvp/
│   └── 2026-Q1/
│       ├── 003-search/
│       └── 004-tags/
├── auth/
│   └── 005-mfa/                ← active
├── items/
│   ├── 006-bulk-edit/          ← active
│   └── 007-csv-export/         ← recently shipped
└── billing/
    └── 008-stripe-webhooks/    ← active
```

`_archive/` starts with an underscore so it sits at the top of `ls` and stands out clearly. You can safely `git mv` — git history preserves the trail, and links in old PRs still resolve to the commits where the folder lived at its old address.

> 💡 **Why quarterly, not monthly**: a quarter is a long enough window for "freshly implemented" to stabilize in production without post-fact tweaks. On a monthly cadence you'd move things and then come back to spec.md two weeks later for a bug fix — that creates friction.

---

## Delete without regret for truly-dead specs

A cancelled feature that nobody will ship and nobody plans to ship — just `git rm specs/017-feature-we-dropped/`. It stays in git history forever:

```bash
# Find deleted specs:
git log --diff-filter=D --summary | grep "specs/"
```

If you ever need to bring it back — `git checkout <commit>~ -- specs/017-feature/` will restore it from history.

> ⚠️ **Don't be afraid to delete.** The "but what if we need it" fear is exactly what creates the junk drawer. Git is your undo; use it.

---

## Domain hierarchy for numbering

At 200+ features, flat numbering `001`...`200` becomes meaningless (who remembers what `127` does?). Better:

```
specs/
├── auth/
│   ├── 001-login/
│   ├── 002-mfa/
│   └── 003-sso/
├── items/
│   ├── 001-crud/
│   ├── 002-search/
│   └── 003-tags/
└── billing/
    ├── 001-stripe/
    └── 002-invoicing/
```

Domain plus local numbering. That way "items/002-search" sticks in your head better than "127-search" lost among 200 other numbers.

### How to set this up in spec-kit

1. Modify `.specify/scripts/bash/create-new-feature.sh` so it asks for the domain and numbers within that domain.
2. Or in `init-options.json` set `branch_numbering: timestamp` and ignore the number entirely, grouping only by domain.
3. Or via `extensions.yml` — a custom `before_specify` hook that asks "domain?" and creates the folder in the right place.

---

## A manifest / index file as the entry point

In the project root or in `specs/README.md`, keep an auto-generated index:

```markdown
# Specs Index (auto-generated, last updated 2026-04-28)

## Active (12)
- [auth/004-passkeys](specs/auth/004-passkeys/spec.md) — P1, in-progress, sprint 24, owner: @vadym
- [items/008-bulk-edit](specs/items/008-bulk-edit/spec.md) — P2, planning, sprint 25
- [billing/008-stripe-webhooks](specs/billing/008-stripe-webhooks/spec.md) — P1, in-progress, sprint 24

## Implemented in last 90 days (28)
- [items/007-csv-export](specs/items/007-csv-export/spec.md) — shipped 2026-04-12 (PROJ-1456)
- [auth/003-sso](specs/auth/003-sso/spec.md) — shipped 2026-03-28 (PROJ-1389)

## Archived (Q4 2025 + earlier — see _archive/)

## Statistics
- Total: 168
- Active: 12 (7%)
- Implemented (recent): 28 (17%)
- Archived: 124 (74%)
- Dropped/deleted: 4
```

It's regenerated by a script that reads the frontmatter of all specs. Wire it into a pre-commit hook or CI so it doesn't go stale.

---

## Search tooling

A simple ~50-line Python script that *reads frontmatter* across all specs and supports queries:

```python
#!/usr/bin/env python3
# scripts/specs.py

import sys
import frontmatter
from pathlib import Path

def list_specs(status=None, domain=None, sprint=None):
    for spec_path in Path('specs').rglob('spec.md'):
        post = frontmatter.load(spec_path)
        if status and post.get('status') != status:
            continue
        if domain and not str(spec_path).startswith(f'specs/{domain}/'):
            continue
        if sprint and post.get('sprint') != sprint:
            continue
        relative = spec_path.parent.relative_to('specs')
        print(f"{relative}  [{post.get('status', '?')}]  "
              f"{post.get('priority', '?')}  "
              f"{post.get('jira_id', '')}")

if __name__ == '__main__':
    args = dict(arg.split('=') for arg in sys.argv[1:] if '=' in arg)
    list_specs(**args)
```

Usage:

```bash
$ scripts/specs.py status=active domain=items
items/008-bulk-edit  [active]  P2  PROJ-1567
items/009-import     [active]  P3  PROJ-1602

$ scripts/specs.py status=implemented sprint=23
auth/003-sso         [implemented]  P1  PROJ-1389
items/006-comments   [implemented]  P2  PROJ-1402
```

Thirty minutes to write, but saves hours of searching months later.

> 💡 **Extensions**: add subcommands — `scripts/specs.py find <text>` for full-text search across spec.md, `scripts/specs.py drift` to surface specs where the code changed without a spec.md update.

---

## Quarterly grooming ritual

The moment when the trash actually gets taken out — the **quarterly `specs/` grooming**. Thirty minutes at retro, once every three months.

### Checklist

1. **List all `status: active`** older than 90 days — either *actually* active (someone is working on them) or stuck (need archive/drop).
2. **List `implemented` older than 90 days** — move to `_archive/<quarter>/`.
3. **List `superseded`** — verify the new version really covers the old requirements and archive the old one.
4. **List skipped/dropped** — just delete them.
5. **If a domain has more than 30 specs** — discuss further splitting.

### A script to prep for retro

```bash
#!/bin/bash
# scripts/grooming-prep.sh — generates lists for quarterly review

echo "=== Active specs older than 90 days ==="
scripts/specs.py status=active | while read line; do
    path=$(echo $line | awk '{print $1}')
    age=$(stat -c %Y specs/$path/spec.md 2>/dev/null)
    now=$(date +%s)
    days=$(( (now - age) / 86400 ))
    if [ $days -gt 90 ]; then
        echo "$path (age: $days days)"
    fi
done

echo ""
echo "=== Implemented specs ready to archive ==="
scripts/specs.py status=implemented | while read line; do
    # ... same as above
done

echo ""
echo "=== Domains with >30 active specs ==="
scripts/specs.py status=active | awk -F'/' '{print $1}' | sort | uniq -c | awk '$1 > 30'
```

### Without this ritual

Any structure turns into a junk drawer within a year, because this is **not a structure problem** — it's a problem of **missing curation**.

---

## Don't write specs for trivia

A separate discipline that shrinks the problem *at the root*: not every task needs `specs/<feature>/`.

From `SPEC_KIT_USE_CASES.md` Scenario 1 — Quick Spec Flow for trivia (bug fixes, trivial changes). The team can adopt the rule: "features under 3 SP — no specification, just code review." That cuts the inflow significantly.

From Discussion #152, a community quote: «Even for small features like 'Add login' we have to use spec-kit? Wouldn't we save more time if just use copilot Plan agent for small features?» — and the community's answer: no, you don't need it for everything.

> 💡 If you write a spec for every trivial change, you'll end up with 500 specs in a year. If only for medium-and-up tasks — 50–80, which is perfectly manageable.

---

## A realistic picture for a 5–10 person team over 2 years

With proper discipline:

- ~80 specs per year (1.5 SP+ tasks; trivia skipped)
- After 2 years — ~160 specs, of which:
  - ~30 active or recently shipped (living in `specs/<domain>/`)
  - ~110 archived in `specs/_archive/<quarter>/`
  - ~20 deleted (dropped/skipped)
- 4 quarterly grooming sessions
- One `specs.py` script + auto-generated index

**Perfectly livable.**

**Without** discipline — the same project after 2 years has 320 specs in a flat `specs/`, 60% of which are dead weight, and a new hire spends their first two weeks asking "where's our X?" because nothing is findable.

---

## A Constitution principle for your specific project

Add to constitution.md:

```markdown
## Principle X: Specs Hygiene (SHOULD)

**Rule**: Every spec.md has frontmatter status field. Quarterly we move
implemented specs older than 90 days into _archive/<YYYY>-Q<N>/.
Dropped/skipped specs are deleted (not kept as zombies).
Domains with > 30 active specs trigger a split discussion.

**Rationale**: Without active curation, specs/ turns into a junk drawer
within a year — a new hire can't find the source of truth and loses 30%
of their onboarding time.

**How to apply**:
- Pre-commit hook validates frontmatter (`status`, `domain` are required)
- Quarterly retro reserves a 30-minute slot for specs grooming
- CI verifies that `specs/README.md` (auto-index) is fresh
```

This makes the problem an explicit point of team discipline, not a "we'll clean up someday."

---

## Antipatterns

### Sprint-based grouping

```
specs/
├── sprint-12/
├── sprint-13/
└── sprint-14/
```

❌ **Don't do this.** A sprint is a time container; features outgrow it. If `004-favorites` started in sprint 13, didn't close, and slipped into 14 — where do you put it? Duplicate? Move?

✅ **The right way**: domains plus frontmatter `sprint: 13` for queries via the script.

### Deep nesting

```
specs/
└── product-area-1/
    └── sub-feature-group/
        └── specific-feature/
            └── 001-tiny-thing/
```

❌ Hard to navigate, hard to find, complicates scripts.

✅ Maximum two levels: `specs/<domain>/<NNN-feature>/`.

### Specs-as-tickets

The `specs/` folder mirrors the entire Jira backlog 1:1.

❌ Creates bloat: trivial bug fixes show up as specs even though they don't warrant a full specification.

✅ Specs only for what really needs specification — features ≥ 3 SP, complex refactors, new integrations.

### Never deleting

❌ "What if we need it" → 200 dead specs in `specs/`.

✅ Quarterly deletion of dropped/skipped specs via `git rm`. Git history is your archive of last resort.

### One huge file

Instead of `specs/<feature>/spec.md` + plan.md + ... — a single `specs/all-features.md` with everything in it.

❌ None of the `/speckit.*` commands will work; it completely breaks the spec-kit system.

✅ Stick to the standard spec-kit structure with separate files.

---

## Keeping specifications current between sprints

A separate task from archival — keeping specs **currently accurate**. `specs/<feature>/spec.md` is created as a snapshot of the moment the feature was authored. Six months after ship, it has turned into a historical document, but readers keep using it as if it described current reality.

### The root problem

Specs in SDD are naturally atomic: one spec = one change. But the product evolves as a **system**, not as a chain of isolated changes. Hence the gap:

- `specs/001-user-auth/` describes auth as it was built in sprint 3.
- `specs/015-add-mfa/` added MFA in sprint 8.
- `specs/023-passkeys/` replaced part of MFA with WebAuthn in sprint 14.
- `specs/031-session-management/` changed timeouts in sprint 19.

The question **"how does our auth currently work?"** ends up answered by 4 specs, some of which contradict each other. None of them *lies*, but none of them shows **current reality**.

This is a problem the community has acknowledged ([Discussion #152](https://github.com/github/spec-kit/discussions/152), Jflam — one of the maintainers):

> «if you make a new folder/doc for every new feature… a very executional, short term mentality as opposed to having a system level design mentality.»

Below are 5 patterns teams actually use, with a neutral look at the pros and cons of each. None of them is "the right one" — the choice depends on team size, level of discipline, and how often the domain changes.

---

### Pattern A — Atomic + Frozen

Each spec is immutable after ship. Once closed, you don't touch it. To understand current state, either you read all relevant specs or you look at the code.

**Mechanic:**
- `spec.md` gets `status: implemented` after merge to main.
- No further edits to that file — even if the feature has evolved.
- To understand current state, the reader walks the historical specs of the same domain in sequence.

**What works (in theory):**
- Clear history: each spec is an exact snapshot of the moment a decision was made.
- No merge conflicts in specs (because they aren't edited).

**What breaks (in practice):**
- Six months into a project with 50+ specs, a new hire can't figure out how anything works.
- The question "what's the current state?" has no convenient answer.

**Pros:**
- Minimal effort
- Immutable record — good for compliance / audit
- No arguments about "who should update this"

**Cons:**
- Doesn't scale — at 50+ specs, finding current reality becomes impossible
- Drift isn't just possible, it's **built into** the pattern
- Onboarding for new engineers is expensive

---

### Pattern B — Supersession Chain

Each new spec explicitly marks its predecessor via frontmatter. Old ones automatically get `superseded_by: <next-spec>`.

**Mechanic:**

```yaml
# specs/023-passkeys/spec.md
---
supersedes: [015-add-mfa]
status: implemented
---

# specs/015-add-mfa/spec.md (auto-updated)
---
status: superseded
superseded_by: 023-passkeys
---
```

A script tells you: "to understand auth, read specs/023-passkeys/ as the head of the chain; the specific history lives in its predecessors."

**What works (in theory):**
- If a new feature *fully* replaces an old one — the chain gives a clean path to current state.
- Tooling can automatically find the head of the chain.

**What breaks (in practice):**
- Reality is that features rarely *fully* replace previous ones. 015-add-mfa partially lives alongside 023-passkeys (a fallback for legacy users).
- Chain relationships are linear; reality is a graph.
- The `supersedes` frontmatter requires discipline that often slips.

**Pros:**
- Preserves history while letting you reach current state
- Automation-friendly (clear frontmatter format)

**Cons:**
- Doesn't capture partial replacements / additive evolution
- Frontmatter discipline is critical
- Hard to navigate with deep chains (5+ versions)

---

### Pattern C — Living Domain Specs (inside `specs/`)

A separate `domain.md` per bounded context that always reflects current state. Atomic specs for each change update domain.md as part of DoD.

**Mechanic:**

```
specs/
├── auth/
│   ├── domain.md              ← current state of auth
│   ├── 001-user-auth/         ← historical (how we started)
│   ├── 015-add-mfa/           ← historical (how we added MFA)
│   └── 023-passkeys/          ← latest; its PR updated auth/domain.md
└── items/
    ├── domain.md
    └── ...
```

The split: atomic specs are frozen deltas; `domain.md` is current state, updated by every delta. DoD rule: a PR doesn't merge without an update to domain.md.

**What works (in theory):**
- One reading point for current state in a domain.
- Locality: all artifacts for a domain live in one folder.

**What breaks (in practice):**
- Archival becomes complicated: moving `specs/auth/015-add-mfa/` into `_archive/` breaks domain locality.
- Either you don't archive — and `auth/` bloats to a hundred folders.
- A per-PR update to `domain.md` creates friction — the engineer wants to ship a feature, not edit a huge document.
- When two teams work on auth in parallel — merge conflicts in `domain.md`.

**Pros:**
- Domain locality — everything in one place
- Single discovery path for a new hire

**Cons:**
- Conflicts with the archival strategy
- The domain folder grows quickly
- Per-PR DoD on updating domain.md — high discipline bar
- Risk of merge conflicts between features in the same domain

---

### Pattern D — Unified Spec (Jflam pattern)

One file per domain `specs/auth/spec.md` lives as a unified document. All changes are commits in git to the same file. A `## Changelog` section at the bottom carries the history.

**Mechanic:**

```markdown
# Auth Spec (current)

[все актуальне — user stories, FR, edge cases]

## Changelog

### 2026-04-28 — Added passkeys
- Added WebAuthn support
- TOTP marked as deprecated

### 2026-01-15 — Added TOTP MFA
- ...

### 2025-09-10 — Initial auth (v1)
```

**What works (in theory):**
- One file, one place, minimal cognitive load.
- Git history naturally preserves evolution via `git log -p`.

**What breaks (in practice):**
- It breaks spec-kit mechanics: `/speckit.specify` creates a new folder, it doesn't edit an existing file.
- You end up manually redirecting work into the existing file, which creates friction with the standard workflow.
- Parallel work — merge conflicts in a single file.
- The file grows without bound — 100+ KB for a single domain after 2 years.

**Pros:**
- Minimalist structure
- Natural currency — current state at the very top of the file
- Git history carries all the evolution

**Cons:**
- Incompatible with standard spec-kit mechanics
- Requires custom workflow or forking templates
- Doesn't scale across time (the file grows)
- High risk of merge conflicts

---

### Pattern E — Hybrid (architecture docs + flat specs)

A separate file space for current state (`docs/architecture/<domain>.md`) and a separate one for atomic deltas (`specs/<NNN-feature>/`). Architecture docs are high-level (1–2 pages); specs are detailed delta records.

**Mechanic:**

```
docs/architecture/
├── overview.md
├── auth.md            ← current state of auth domain (≤2 pages)
├── items.md
└── billing.md

specs/
├── _archive/
│   ├── 2025-Q4/
│   └── 2026-Q1/
├── 042-webauthn-fallback/    ← active
└── 048-bulk-edit/            ← active
```

Specs carry `domain: auth` in frontmatter, which lets you find them by domain via the script. Architecture docs are updated either per-PR (via template modification, see below) or per-quarter (at retro).

**What works (in theory):**
- Two spaces, two speeds: deltas are fast (atomic), architecture is slow (high-level).
- Specs can be archived completely independently of architecture docs.
- Architecture docs are short, so drift isn't catastrophic.

**What breaks (in practice):**
- Two parallel systems — risk of divergence.
- Without a sync process, architecture docs drift.
- Splitting "what goes in a spec, what goes in architecture" requires judgment.

**Pros:**
- Clean archival of specs
- Architecture docs are short — easier to maintain
- Doesn't break spec-kit mechanics
- Different update cadences are possible

**Cons:**
- Risk of drift between the two spaces
- Requires a process / ritual for sync
- Splitting responsibility between artifacts isn't obvious

---

### Pattern summary table

| Pattern | Currency frequency | Scaling | Conflict risk | Spec-kit compatibility |
|---------|-------------------|---------|---------------|------------------------|
| A. Atomic + Frozen | None (goes stale by default) | Low | None | Full |
| B. Supersession Chain | Per-replacement | Medium | Low | Full |
| C. Living Domain Specs | Per-PR | Low (domain bloats) | High (merge in domain.md) | Full |
| D. Unified Spec | Per-commit | Low (file grows) | Very high | Partial (requires custom workflow) |
| E. Hybrid (arch docs + flat specs) | Per-quarter (or per-PR) | High | Low | Full |

---

## Sync automation mechanisms

Regardless of the chosen pattern, you can wire up three levels of automation to lower the discipline burden.

### Level 1 — Command template modification

`.specify/templates/commands/plan.md` is just a markdown prompt. You can add instructions that the agent executes on every `/speckit.plan`.

**Mechanic:**

Add a section to the end of `plan.md`:

```markdown
### Phase 2: Architecture Doc Update Proposal

1. Read frontmatter of active spec to identify the domain.
2. Read current docs/architecture/<domain>.md (if exists).
3. Generate proposed updates based on the new spec/plan.
4. Write proposal to specs/<active>/architecture-update.md.
5. Add "Architecture Impact" section to plan.md.

DO NOT modify the architecture doc directly — produce a proposal for review.
```

On every `/speckit.plan`, the agent itself checks the domain, reads the architecture doc, and generates a review-ready proposal.

**What works:**
- Proactive detection of architectural changes during planning.
- No external scripts or infrastructure required.
- Works on any agent (Claude, Copilot, Cursor) — it's just markdown.

**What breaks:**
- When spec-kit is upgraded, templates can be overwritten. You need a fork strategy or a documented patch.
- AI proposals require human review — auto-merging produces errors that accumulate.
- The proposal quality depends on how well the existing architecture doc reads.

**Pros:**
- Lowest barrier to entry
- Doesn't require extension/MCP infrastructure
- Works native in the workflow (proposal is created together with the plan)

**Cons:**
- Gets clobbered on `specify integration upgrade`
- Requires a custom patch, documented separately
- Only proposals — doesn't perform the actual sync

---

### Level 2 — Hooks via `.specify/extensions.yml`

Hooks let you run shell commands before/after spec-kit commands. Extension points: `before_specify`, `after_specify`, `before_plan`, `after_plan`, `before_tasks`, `after_tasks`, `before_implement`, `after_implement`.

**Mechanic:**

```yaml
hooks:
  after_implement:
    - id: append_arch_changelog
      enabled: true
      optional: false
      extension: "local"
      command: "scripts/append_arch_changelog.sh"
      description: "Append shipped feature to architecture doc changelog"
```

The script reads the active spec's frontmatter, identifies the domain, and appends a single line to the `## Recent changes` section of the corresponding `architecture/<domain>.md`.

**What works:**
- Mechanical updates (appending a line to a changelog) are reliable.
- Triggering on `after_implement` ensures the update happens only after a successful implementation.

**What breaks:**
- Sed-based text manipulation of markdown is brittle — the smallest structural change breaks the script.
- Hooks aren't designed for AI invocations directly. To generate prose you need Level 1 or Level 3.
- `optional` / `required` parameters on hooks aren't always respected — depends on the host agent.

**Pros:**
- Automatic recording of fact-of-shipping
- Works without AI at this stage (deterministic)
- Integrates into the native lifecycle of spec-kit commands

**Cons:**
- Mechanical updates only (not prose)
- Brittleness of sed/awk manipulation
- The hooks mechanism in spec-kit is still early-stage

---

### Level 3 — Custom slash command

You can add your own command by creating a new template under `.specify/templates/commands/<name>.md`. The command is invoked explicitly, not automatically.

**Mechanic:**

`.specify/templates/commands/archupdate.md` with instructions:

```markdown
---
description: Propose updates to architecture docs based on recently shipped specs
---

1. List specs with status: implemented, implemented_date within 30 days, group by domain.
2. For each domain: read architecture doc + read shipped specs, identify discrepancies.
3. Produce proposals: docs/architecture/_proposals/<domain>-<YYYY-MM-DD>.md
4. Generate summary report.

DO NOT modify docs/architecture/<domain>.md directly.
```

Once a quarter (at retro), you run `/speckit.archupdate` — the agent itself walks all shipped specs over the period, compares them against architecture docs, and generates proposals.

**What works:**
- Batch retrospective currency check.
- Surfaces drift that wasn't caught per-PR.
- A clear ritual tied to the quarterly cycle.

**What breaks:**
- Depends on a human trigger — if you skip retro, the sync doesn't happen.
- Analysis quality depends on the agent's context size (with a large number of specs, it can lose detail).
- Proposals still require human review.

**Pros:**
- Works as a ritual — an explicit moment of sync
- Can catch what Level 1 misses (cross-feature drift)
- AI horsepower for semantic analysis

**Cons:**
- Manual trigger — risk of being skipped
- Large context — risk of low-quality proposals
- Proposals only, no auto-merge

---

### Automation mechanism summary table

| Level | Trigger | Update type | Infrastructure | AI dependency |
|-------|---------|-------------|----------------|---------------|
| 1. Template mod | Per-`/plan` | Proposal (prose) | Markdown patch | High |
| 2. Hooks | Per-`/implement` (etc.) | Mechanical (changelog) | extensions.yml + script | Low |
| 3. Custom command | Manual (e.g. quarterly) | Proposal (prose, batch) | Command template | High |

---

### Compatibility of patterns and mechanisms

Not every pattern × mechanism combination makes sense:

| | Level 1 (template mod) | Level 2 (hooks) | Level 3 (custom command) |
|---|------------------------|-----------------|--------------------------|
| **A. Atomic + Frozen** | N/A (no currency artifact) | N/A | N/A |
| **B. Supersession Chain** | Auto-update frontmatter | Possible | Possible |
| **C. Living Domain Specs** | Update domain.md proposal | Append changelog to domain.md | Quarterly review of domain.md |
| **D. Unified Spec** | Append section to unified | Append changelog | Quarterly cleanup |
| **E. Hybrid (arch docs)** | Update arch doc proposal | Append changelog to arch doc | Quarterly arch review |

Pattern A doesn't need sync mechanisms by definition. The rest benefit from Levels 1–3 in various combinations.

---

## At-scale checklist

If your project crosses the 50–80 specs threshold:

- [ ] Every spec.md has frontmatter with a `status` field
- [ ] An `_archive/` directory has been created
- [ ] Specs with status `implemented` older than 90 days have been moved
- [ ] `dropped` specs have been deleted (via git rm)
- [ ] `specs/README.md` is set up as an auto-generated index
- [ ] `scripts/specs.py` exists for search/filtering
- [ ] A quarterly grooming ritual is scheduled at retro
- [ ] The Specs Hygiene principle has been added to constitution.md
- [ ] If domains are already visible — `specs/<domain>/` structure has been created
- [ ] A pre-commit hook validates frontmatter
- [ ] The team has chosen and documented its currency pattern (A/B/C/D/E)
- [ ] If pattern A wasn't chosen — at least one sync mechanism is in place (Level 1/2/3)

---

## What to read next

- **`SCRUM_INTEGRATION.md`** — how specs hygiene integrates with the Scrum cadence (quarterly grooming = part of retro).
- **`CONSTITUTION_GUIDE.md`** — how to add the Specs Hygiene principle to the constitution.
- **`SPEC-KIT-docs.md`** — section 8.5 Project Memory.

> 🚀 **Key tip**: don't wait until `specs/` becomes a junk drawer. Start the hygiene practice at spec #50, not at #250. Moving 5 specs into `_archive/` takes 2 minutes; moving 200 takes half a day.
