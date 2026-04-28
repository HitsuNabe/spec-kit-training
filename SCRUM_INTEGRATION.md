# SCRUM_INTEGRATION — Spec-Kit у Scrum/Agile процесах

> Як впровадити spec-kit у команду 5–10 інженерів, що працює зі стандартним Scrum-циклом. Хто пише які артефакти, коли, як це лягає на грумінг / планування / стендап / демо / ретро. Інтеграція з Jira/Linear.

---

## Чесне попередження про state-of-the-art

Spec-kit — інструмент, якому близько року на момент написання (Sept 2025+). Реальних публічних production-кейсів «команда X у Scrum уже рік» дуже мало. У community-обговореннях [#152](https://github.com/github/spec-kit/discussions/152), [#299](https://github.com/github/spec-kit/issues/299), [#889](https://github.com/github/spec-kit/issues/889) бачимо:

- **Pain points** реальних команд (single-spec-shot trap, work item sprawl, PR-too-large)
- **Кілька pattern-ів**, що з'явились як консенсус (Jira ID у назві папки, FR як subtasks)
- **Багато порожніх ніш** — точний таймінг, Sprint demo, retrospective практики

Цей документ — **синтез доступних практик + обґрунтовані рекомендації** на основі загального Scrum + механіки spec-kit. Адаптуйте під свою команду, фіксуйте на ретро що працює, що ні.

---

## Загальна картина

Перші три речі, які треба прийняти як дані, щоб уникнути помилкових очікувань:

**1. Spec-kit не замінює Jira/Linear, а живе паралельно.** Jira — це планувальник і трекер; spec-kit — інженерний робочий простір. Зв'язок між ними тонкий: Jira ID у назві папки + `Resolves PROJ-XXXX` у PR.

**2. За дефолтом spec-kit генерує один великий feature-branch + один великий PR на спеку.** Community відкрито визнає це проблемою (Discussion #152). Реалістичну Scrum-схему «1 user story → 1 невелика гілка → 1 короткий PR» треба явно адаптувати — наприклад, розбивати на per-phase MR.

**3. SP оцінюються на рівні Functional Requirements, не tasks.** Це найвдаліший pattern із цитованих обговорень (Issue #889, прямо):

> Functional requirements (FR-001, FR-002…) у spec.md — це те, що йде у Jira як user stories із story points. Tasks.md залишається як executable checklist у репозиторії.

Чому: FR — стабільний контейнер; tasks можуть переписатись після `/speckit.analyze`; PM/QA дивляться FR (читабельні), не tasks (технічні деталі).

---

## Цикл спрінта з spec-kit

### Перед спрінтом — Backlog Refinement (Grooming)

Це критичний момент, де SDD або працює, або перетворюється на «SpecFall» (markdown-обгортка над waterfall).

**Підготовка (асинхронно, перед зустріччю)**

PM/PO приходить з 3–7 потенційними фічами на наступний спринт. Tech lead або owner кожної фічі заздалегідь, **до зустрічі**, прогоняє `/speckit.specify` для кожної. 5–15 хвилин на фічу — отримуєте чорновий `spec.md` з User Stories, FR-001..N, Success Criteria.

> 💡 **Ключовий аргумент для пакетного запуску** (з Issue #299): «Each user story creates an isolated tree of development. This causes problems: design tunnel-vision, it becomes hard to motivate enabling work… Encourage running specify multiple times up-front so multiple user stories are collected». У командному режимі краще зібрати кілька drafts перед грумінгом, а не one-shot per ticket.

**На зустрічі (1 година)**

1. Owner кожної фічі за 5 хвилин показує draft `spec.md`.
2. Команда дивиться User Stories і FR — питає, заперечує, уточнює.
3. Для фіч із ambiguity — запускаємо `/speckit.clarify` **наживо**: агент задає 5 multiple-choice питань, команда вирішує разом, відповіді одразу записуються у `## Clarifications`.
4. Для найпріоритетніших — `/speckit.checklist` на quality validation з боку PM/QA.

**Артефакти на виході грумінгу:**

```
specs/
├── PROJ-1234-user-tags/
│   ├── spec.md              ← refined, clarified
│   └── checklists/requirements.md
├── PROJ-1235-search-filter/
│   ├── spec.md
│   └── checklists/requirements.md
└── PROJ-1236-comments/
    └── spec.md              ← ще не clarified, переноситься на наступний грумінг
```

### Початок спрінта — Sprint Planning

**До зустрічі (асинхронно)** — owner кожної фічі прогоняє `/speckit.plan` і `/speckit.tasks`.

> ⚠️ **Чому асинхронно**: повний цикл `/plan → /tasks → /analyze` займає 30–60 хвилин на фічу. Якщо запускати наживо на planning meeting, він розтягнеться на 3 години. Scott Logic у своєму огляді скаржиться на «10x slower if executed live» — це підтверджує підозри.

Це дає для кожної фічі:

- `plan.md` — Constitution Check, Technical Context, Project Structure
- `research.md` — Decision/Rationale/Alternatives
- `data-model.md`, `contracts/`, `quickstart.md`
- `tasks.md` — фази Setup → Foundational → US1 (P1) → US2 (P2) → Polish

**На самій зустрічі (1 година):**

1. Owner кожної фічі за 5 хвилин показує: чого вимагає фіча, у який Constitution Check впирається, як декомпонована на user stories.
2. Команда оцінює story points **на рівні FR** (FR-001 = 2 SP, FR-002 = 5 SP).
3. У Jira/Linear створюються тікети **з лейблом per-user-story** (US1, US2). Subtask-и під кожним — *не дзеркало tasks.md*, а укрупнені блоки на день-два роботи. Уникайте «work item sprawl».
4. `/speckit.taskstoissues` (якщо GitHub) або custom integration (якщо Jira/Linear) синхронізує high-level задачі.

**Розподіл фічей між інженерами:**

Один `specs/<feature>/` = один owner = одна гілка. Якщо фіча велика (>10 SP), розбивайте на МР по фазах:

- Phase 2 Foundational + US1 = окремий PR
- US2 = другий PR
- US3 + Polish = третій PR

Це частково знімає проблему «PR-too-large», на яку скаржиться community.

### Під час спрінта — Daily Standup

Реальних кейсів *у ретроспективах* інших команд не знайдено. Issue #181 показує пов'язану біль: *немає автоматичного маркування `[X]` у `tasks.md`*, тому «звітувати по галочках» поки незручно.

Рекомендована практика:

- Кожен інженер каже **на рівні user story / FR**, не на рівні task ID. Тобто не «зробив T012, T013, починаю T014», а «закрив US1 для search-filter, починаю US2 — треба піврядня; блокер — питання до PM по edge case з whitespace queries».
- Tasks.md відкритий у репозиторії — tech lead і ревʼюєр PR бачать прогрес у real-time через GitHub.
- Якщо tasks.md розходиться зі spec.md (інженер реалізує не зовсім те, що в FR) — це сигнал «STOP, повернись до `/clarify` або `/analyze`».

> ⚠️ **Обов'язковий ритуал**: коли інженер відкриває PR, він повинен підтвердити у description, що `spec.md` оновлено (додав `## Implementation Notes` або відкоректував FR, якщо реальність розійшлась з планом).

Прямі цитати з community про **drift** (Discussion #152, Jflam): «these things naturally diverge as it's pretty rare for the agent to one-shot the code… those changes ultimately become material. So unfortunately right now it's on you to fold those things back into the spec.»

### Mid-sprint check / Refinement of next sprint

На середині спрінта — паралельний грумінг наступного. Ті фічі, що не пройшли в поточний (`PROJ-1236-comments` у моєму прикладі), доводяться через `/clarify`, `/plan`, готуються до планування наступного спрінта. Це класична Scrum-практика, spec-kit нічого не міняє.

### Кінець спрінта — Sprint Review / Demo

Що показуємо стейкхолдерам:

- **Працюючу фічу** — це первинне.
- **`specs/<feature>/quickstart.md`** як «scripted demo flow». quickstart.md за документацією spec-kit — це manual validation scenarios, але вони ідеально підходять як демо-сценарії: «Користувач створює tag → присвоює item → фільтрує — бачить результат».
- **`spec.md`** для PM/business — *що саме* реалізували, які user stories закрили, які FR покриті.

> 💡 Прямих звітів «команди показують stakeholder'ам quickstart.md» у моєму research немає — це обґрунтована рекомендація на основі того, що quickstart.md уже структурований саме так.

### Retrospective

Місце, де команди дають справжню зворотну зв'язку про SDD.

**Питання, які варто ставити на ретро:**

1. *Чи виправдалось додаткове 30–60 хвилин на specify+clarify+plan для кожної фічі?* Якщо для дрібних — ні, переходимо до Quick Spec Flow для < 4 SP.
2. *Чи Constitution Check у `/plan` реально fail-ив за останні 2 спрінти?* Якщо ні — або принципи м'які, або шаблон їх не перевіряє.
3. *Чи specs реально читають перед PR review?* Якщо ні — це SpecFall (markdown без вживання).
4. *Які фічі застрягли — чому? Це проблема decomposition (треба було меншу US1), чи clarify (бракує інформації), чи implementation?*

**Прямі цитати community про ретро:**

Що зайшло (Discussion #1549): «we've been using speck-kit since a few months now, love it.»

Що не спрацювало:
- Martin Fowler (fragment): «verbose and tedious-to-review markdown files… overkill for the feature size».
- Scott Logic (fragment): «10x slower… a sea of markdown documents».
- Marmelab (fragment): «risks burying agility under layers of Markdown.»
- InfoQ (fragment): «SpecFall — markdown monster that generates layers of outdated documentation on arrival.»
- EPAM (Discussion #152): «that model is effective for a startup with a monorepo, but it doesn't map well to enterprise environments.»

---

## Матриця ролей — хто що створює

| Артефакт | Хто пише draft | Хто рев'ює / approve | Коли |
|----------|----------------|---------------------|------|
| `constitution.md` | Tech lead + 1 staff | Уся команда дискусійно; формально 2 staff approve | Один раз; bumps при RFC |
| `spec.md` (per feature) | Owner фічі (інженер) або PM | PM + 1 інженер; обов'язково *до* `/plan` | Перед грумінгом (draft), на грумінгу (refined) |
| `## Clarifications` секція | Команда наживо | Автоматично — як консенсус | На грумінгу |
| `plan.md`, `research.md`, `data-model.md`, `contracts/` | Owner запускає `/plan`; AI генерує | Tech lead — фокус на Constitution Check, Complexity Tracking | Між грумінгом і планінгом, асинхронно |
| `tasks.md` | AI з `/speckit.tasks` | Owner проходить очима; tech lead на планінгу | Перед планінгом |
| `/speckit.analyze` report | AI; не персистує файл | Owner усуває CRITICAL/HIGH | Перед `/implement` |
| Код + `[X]` у tasks.md | AI з `/speckit.implement` + інженер | Standard PR review | Sprint execution |
| Updated `spec.md` після implement | Owner | Reviewer перед merge | На етапі PR |
| Quickstart-демо | Owner | PM на review | Sprint demo |

> 💡 **Authoring constitution** (Discussion-style fragments): «authoring constitution demands senior engineering judgment, time, and iteration… junior developers cannot produce this in a day, and most teams will revise theirs repeatedly before it holds up in practice.»

---

## Інтеграція з Jira/Linear

### Базова мапа

| Jira | Spec-Kit | Де живе |
|------|----------|---------|
| Epic | (опц.) `vision-spec.md` або architecture ADR | `specs/EPIC-XX-vision/` |
| Story | `spec.md` цілої фічі | `specs/PROJ-XXXX-feature/spec.md` |
| Subtask | Functional Requirement (FR-001...) | у `spec.md` як секція |
| Sprint | (frontmatter тег) | `sprint: 24` у YAML |
| Story Points | (frontmatter тег + на FR) | `story_points: 5` |
| Acceptance Criteria | User Story → AC у spec.md (Gherkin) | у `spec.md` під User Story |
| Implementation tasks | `tasks.md` (T001, T002...) | у git, *не* в Jira |

### Робочий процес від Jira-тікета до PR

```bash
# Тікет: PROJ-1234 "Add search to items list"

git checkout -b PROJ-1234-search-filter
```

У чаті агента:

```
/speckit.specify

PROJ-1234. Реалізуй пошук items по title. Debounce 300ms,
counter "Found N items", clear через X або Esc.
```

Створюється `specs/PROJ-1234-search-filter/spec.md`. Далі — стандартний цикл: `/clarify` → `/plan` → `/tasks` → `/analyze` → `/implement`.

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

Якщо у Jira налаштовано GitHub integration — `Resolves PROJ-1234` автоматично закриває тікет при merge.

### Frontmatter для bidirectional link

У `spec.md` додаєте:

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

У Jira-тікеті, відповідно — посилання на специфікацію в git (поле "Linked issues" або у description).

### `/speckit.taskstoissues` для Jira

Команда існує, але працює тільки з **GitHub Issues**. Для Jira:

- Створюйте FR-subtasks **вручну** на спрінт-плануванні (5 хвилин).
- Або через Atlassian MCP — Jira Cloud має MCP server, через який AI-агент може створювати тікети. Це extension над spec-kit, не з коробки.
- Або власний скрипт через Jira REST API.

Discussion #1549 показує, що community команди вже роблять свої інтеграції — «we explored a way to integrate it with our stack (e.g. Jira, tools for requirements quality analysis, code quality, etc).»

---

## Definition of Ready / Done

Закріпіть як командний стандарт.

### DoR (Definition of Ready) для тікета — перед взяттям у спрінт

- ✅ `spec.md` існує, без `[NEEDS CLARIFICATION]` маркерів
- ✅ `## Clarifications` секція заповнена або marked as N/A
- ✅ Constitution Check у `plan.md` пройдено
- ✅ `/speckit.analyze` без CRITICAL findings
- ✅ SP оцінено (на рівні FR, не tasks)
- ✅ Owner призначений, гілка створена

### DoD (Definition of Done) для тікета — перед merge

- ✅ Усі задачі в `tasks.md` помічені `[X]`
- ✅ `quickstart.md` сценарії відпрацьовано вручну (хоча б одним інженером, не автором)
- ✅ `spec.md` оновлено, якщо реалізація розійшлась із початковими FR
- ✅ PR description посилається на `specs/<feature>/quickstart.md`
- ✅ CI зелений, code coverage ≥ Constitution-rule
- ✅ Code review approve від мінімум 1 staff (або 2 для critical)

---

## Common pitfalls у Scrum-контексті

З прямих цитат і реальних обговорень:

1. **«SpecFall»** (InfoQ) — встановили tools і ceremonies без культурного зсуву → outdated markdown. Антидот: на ретро регулярно питайте «Чи specs реально читають? Чи рев'ю спирається на них?». Якщо ні — тривога.
2. **Single-spec-shot trap** (Issue #299) — кожен user story в isolation, cross-story архітектурних рішень не виходить. Антидот: пакетний `/specify` перед грумінгом + один загальний `vision-spec` для проекту.
3. **PR-too-large** (Discussion #152) — один spec → один великий PR. Антидот: розбивайте на per-phase MR (Foundational | US1 | US2 + Polish).
4. **Work-item sprawl** (Issue #889) — 1:1 mapping `tasks.md` → Jira subtasks створює сотні дрібних тікетів. Антидот: мапайте FR → Jira stories, tasks.md → executable checklist у репо.
5. **Spec drift** (Jflam, Discussion #152) — реалізація розходиться зі спекою. Антидот: обов'язковий update `spec.md` як частина DoD; PR description має посилання на оновлений spec.md.
6. **Slow для дрібних фіч** (Scott Logic, Discussion #152) — 10x slower на «Add login». Антидот: явний Quick Spec Flow для < 4 SP — без `/clarify`, `/analyze` (але `/tasks` обов'язково).
7. **Multi-repo / cross-team не вписується** (EPAM, Discussion #152) — spec-kit оптимізовано на monorepo + одну команду. Антидот: для cross-team — coordination spec на org-level (не в репо), per-repo specs зведені.
8. **No automatic completion tracking** (Issue #181) — `[X]` маркувати треба руками. Антидот: дочекайтесь майбутніх версій або власний git pre-commit hook.

---

## Як впровадити SDD у команду — поетапний план

Не нав'язуйте SDD на всю команду одразу. Реальний шлях, який видно у обговореннях:

### Тиждень 1–2 — Champion pilot

**1 інженер (champion)** робить пілот на 2–3 своїх задачах. Проходить повний цикл, фіксує hours spent. Запускає Сценарій 2 (Brownfield Feature) із `SPEC_KIT_USE_CASES.md`.

### Тиждень 3 — Showcase на ретро

Champion на ретро показує приклад: спека, плани, артефакти, фінальний PR. Чесно: де було швидше, де повільніше, де якість виросла.

### Тиждень 4 — Constitution drafting

Tech lead + staff + 1 product person сідають з накопиченим список pain-points від champion і пишуть **constitution v1.0.0**. Деталі — у `CONSTITUTION_GUIDE.md`.

### Тиждень 5–6 — Команда рук на 2 фічах

Tech lead + 1 інженер беруть 2 фічі через повний spec-kit. Інші — як зазвичай, але можуть пробувати на дрібних задачах.

### Тиждень 7 — Решта команди опціонально

На грумінгу пропонуєте `/speckit.specify` як перший етап. Хто хоче — пробує, хто ні — традиційно. Через 2 спрінти більшість зрозуміє переваги і опту-ін.

### Тиждень 9+ — Стандарт

SDD стає стандартом для нових фіч, traditional використовується тільки для спеціальних випадків (емergency hotfix, дрібні зміни < 2 SP). Constitution v1.x з RFC-процесом.

---

## Що читати далі

- **`CONSTITUTION_GUIDE.md`** — як написати і еволюціонувати constitution.md.
- **`SPECS_HYGIENE.md`** — на масштабі (200+ specs) як підтримувати порядок у `specs/`.
- **`SPEC_KIT_USE_CASES.md`** — Сценарій 5 (Team Collaboration) і Сценарій 9 (Compliance).

---

> 🚀 **Найпростіша порада**: спробуйте на одному спрінті. Не намагайтесь з'їсти все одразу — ефект приходить через 2–3 спрінти, коли constitution дозріла, команда звикла до slash-команд, і `specs/` починає накопичувати корисну історію рішень.

**Sources:**
- [GitHub Discussion #152 — Evolving specs](https://github.com/github/spec-kit/discussions/152)
- [GitHub Issue #889 — Support for Jira/Azure DevOps](https://github.com/github/spec-kit/issues/889)
- [GitHub Issue #299 — Phases too coupled](https://github.com/github/spec-kit/issues/299)
- [Scott Logic — Putting Spec Kit Through Its Paces](https://blog.scottlogic.com/2025/11/26/putting-spec-kit-through-its-paces-radical-idea-or-reinvented-waterfall.html)
- [Marmelab — SDD: The Waterfall Strikes Back](https://marmelab.com/blog/2025/11/12/spec-driven-development-waterfall-strikes-back.html)
- [Martin Fowler — SDD Tools Comparison](https://martinfowler.com/articles/exploring-gen-ai/sdd-3-tools.html)
- [InfoQ — SDD adoption at Enterprise Scale](https://www.infoq.com/articles/enterprise-spec-driven-development/)
