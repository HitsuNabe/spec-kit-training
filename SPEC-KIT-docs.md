# SPEC-KIT — Довідник методології та інструменту

> Цей документ — *теоретична* база курсу. Прочитайте його повністю перед тим, як братися за `SPEC_KIT_WORKFLOW_GUIDE.md`. Без розуміння філософії всі команди виглядатимуть як «зайва бюрократія».

---

## 1. Що таке Spec-Driven Development

**Spec-Driven Development (SDD)** — методологія розробки з AI-асистентами, у якій **специфікація** є основним артефактом, а код — *результатом* специфікації.

> ❝ Code used to be king. In SDD, **code serves specifications**. ❞
> — `spec-driven.md`, репозиторій github/spec-kit

### Принципи

| Принцип | Що означає на практиці |
|---------|------------------------|
| **Specifications as lingua franca** | Спека — джерело правди. Не PRD у Confluence, не ticket в Jira, а конкретний файл `spec.md` у репозиторії. |
| **Executable specifications** | Спека має бути настільки точною, щоб з неї можна було згенерувати робочий код. Жодних розмитих формулювань. |
| **Continuous refinement** | Validation працює *постійно*, а не один раз при kickoff. |
| **Research-driven context** | AI-агент сам збирає інформацію про залежності, performance, security → `research.md`. |
| **Bidirectional feedback** | Production-метрики повертаються в специфікацію, спека еволюціонує. |
| **Branching for exploration** | Можна паралельно мати кілька реалізацій однієї специфікації. |

---

## 2. Чим SDD відрізняється від інших підходів

### Vibe Coding

> «Кинув промпт у GPT/Claude — отримав код — закомітив».

| Властивість | Vibe Coding | SDD |
|-------------|-------------|-----|
| Артефакт | Тільки код | Спека → план → задачі → код |
| Передбачуваність | Низька | Висока |
| Рефакторинг | Знову промптити з нуля | Змінити спеку → регенерувати |
| Документація | Відсутня | Є за замовчуванням |
| Code review | Дивляться код | Дивляться спеку *і* код |

### Класичний waterfall / PRD-first

| Властивість | PRD-first | SDD |
|-------------|-----------|-----|
| PRD оновлюється? | Раз і викидається | Живий артефакт у git |
| Drift між docs і code | Завжди | Усувається тим, що PRD генерує код |
| Хто пише PRD | PM/BA | Інженер + AI з checklists |
| Тестованість вимог | Часто `[NEEDS CLARIFICATION]` | Шаблон забороняє цього |

### Які проблеми SDD вирішує

1. **Drift між специфікацією і кодом** — її просто немає, бо спека генерує код.
2. **AI-галюцинації** — шаблон вимагає явних `[NEEDS CLARIFICATION]` маркерів замість «правдоподібних» здогадів.
3. **Over-engineering** — є `Constitution Check` gates, які блокують надлишкову складність.
4. **Дешеві pivot-и** — змінили вимогу → перегенерували `plan.md` і `tasks.md`, замість переписувати код.
5. **Brownfield-розробка** — додавання фічі в існуючий код стає керованим, бо спека описує інтеграцію з існуючою архітектурою.

---

## 3. Архітектура spec-kit

Spec-kit складається з двох частин:

### 3.1. CLI `specify`

Python-додаток, який:

- Створює `.specify/` директорію зі скриптами і шаблонами.
- Конфігурує інтеграцію з обраним AI-агентом (≥30 підтримуваних).
- Реєструє slash-команди.

### 3.2. Бібліотека шаблонів і промптів

Усі команди — це markdown-файли в `.specify/templates/commands/`, які AI-агент читає і виконує. Шаблони містять:

- Структуру файлів спеки/плану/задач (placeholder slots).
- **Guard rails** — заборону згадувати стек у спеці, обмеження кількості `[NEEDS CLARIFICATION]`, обовʼязковість вимірюваних success criteria.
- Інструкції AI-агенту (як ставити уточнюючі питання, як заповнювати таблиці тощо).

### 3.3. Підтримувані агенти (на момент v0.8.1)

| Ключ | Агент |
|------|-------|
| `claude` | Claude Code (skills mode за замовчуванням) |
| `copilot` | GitHub Copilot (IDE + CLI) |
| `cursor-agent` | Cursor |
| `gemini` | Gemini CLI |
| `codex` | OpenAI Codex CLI (skills) |
| `windsurf` | Windsurf |
| `qwen` | Qwen Code |
| `opencode`, `goose`, `kiro-cli`, `tabnine`, `vibe`, `forge`, `pi`, `qodercli`, ... | Інші 20+ |
| `generic` | Будь-який кастомний агент через `--commands-dir` |

> 💡 У Codex CLI skills-режимі префікс команд **`$speckit-*`** (з доларом), а не `/speckit.*`.

---

## 4. Команди (slash commands) — Reference

Усі команди викликаються **в чаті AI-агента**, не в shell.

### 4.1. `/speckit.constitution` — Принципи проекту

Створює/оновлює `.specify/memory/constitution.md`.

**Що генерує:** документ із принципами (Library-First, Test-First, Observability, Simplicity тощо), governance-правилами та semver-версією.

**Коли викликати:**
- Один раз при старті проекту.
- Коли змінюються командні стандарти (тоді bumps версію: MAJOR/MINOR/PATCH).

**Приклад:**

```
/speckit.constitution Створи принципи: code quality (80%+ test coverage),
TDD для core domain, UX consistency через Figma design system,
performance <200ms p95. Governance: MUST-принципи змінюються тільки
через RFC + 2 review.
```

**Артефакт:** `.specify/memory/constitution.md` із frontmatter:

```markdown
---
version: 1.0.0
ratification_date: 2026-04-28
last_amended_date: 2026-04-28
---
```

---

### 4.2. `/speckit.specify` — WHAT and WHY

Створює `specs/<NNN-feature-slug>/spec.md`.

**Поведінка:**
- Генерує kebab-case slug (`001-photo-albums`, `002-search-filter`).
- Створює git-branch (опційно через hook).
- **Заборонено** згадувати стек/фреймворки/API в spec.md.
- **Максимум 3 `[NEEDS CLARIFICATION]` маркери** (за пріоритетом: scope → security → UX → tech).
- Auto-генерує `specs/<feature>/checklists/requirements.md` для self-validation.

**Приклад:**

```
/speckit.specify Додай у застосунок можливість групувати items по тегах.
Користувач може створити тег, призначити кілька тегів одному item,
відфільтрувати items по тегах. Теги унікальні per-user.
```

**Артефакт `spec.md` містить:**
- User Stories з пріоритетами P1/P2/P3
- Functional Requirements (FR-001, FR-002, ...)
- Success Criteria (вимірювані, technology-agnostic)
- Edge Cases
- Key Entities (description, не схема)
- Assumptions

---

### 4.3. `/speckit.clarify` — Зменшення двозначностей

Сканує `spec.md` за 11 категоріями (Functional Scope, Domain & Data Model, UX Flow, NFR, Integration, Edge Cases, Constraints, Terminology, Completion Signals, Misc).

**Поведінка:**
- Задає до **5 цілеспрямованих питань**, по одному за раз.
- Кожне питання — multiple-choice (2–5 варіантів) або вільна відповідь ≤5 слів.
- Один варіант помічено як **Recommended** з обґрунтуванням.
- Після кожної відповіді негайно редагує spec.md (атомарний overwrite) і додає `## Clarifications / ### Session YYYY-MM-DD`.

**Приклад:**

```
/speckit.clarify Сфокусуйся на security і performance вимогах.
```

> ⚠️ **Сильна рекомендація**: завжди запускайте `/speckit.clarify` перед `/speckit.plan`. Виняток — exploratory spike.

---

### 4.4. `/speckit.plan` — Технічний план

Читає `spec.md` + `constitution.md`, генерує:
- `plan.md` — Technical Context, Constitution Check, Project Structure, Complexity Tracking
- `research.md` — Decision / Rationale / Alternatives для кожного `[NEEDS CLARIFICATION]` і кожного нового технологічного вибору
- `data-model.md` — entities, fields, relationships, validation, state transitions
- `contracts/` — API specs (OpenAPI), GraphQL, event schemas, CLI grammars
- `quickstart.md` — сценарії ручної валідації (definition of done)

**Constitution Check gates** виконуються двічі: до фази Research і після фази Design. Будь-яке порушення MUST-принципу — або виправляється, або обґрунтовується в **Complexity Tracking**.

**Приклад:**

```
/speckit.plan Backend: FastAPI + SQLModel + Alembic + PostgreSQL.
Frontend: React 18 + TypeScript + TanStack Query.
Авторизація — існуючий JWT через FastAPI-Users.
Тести — pytest + Playwright для e2e.
```

> 💡 **Лайфхак**: після `/plan` запитайте агента: «Пройди по плану і знайди області, де потрібен additional research. Для кожної — запусти параллельну research task і онови `research.md` з конкретними версіями бібліотек».

---

### 4.5. `/speckit.tasks` — Декомпозиція

Читає plan.md, spec.md (user stories з P1/P2/P3), data-model.md, contracts/, research.md → генерує `tasks.md`.

**Жорсткий формат:**

```markdown
- [ ] T001 Create project structure per implementation plan
- [ ] T002 [P] Configure linting and formatting tools
- [ ] T003 Add Tag SQLModel in backend/app/models/tag.py
- [ ] T012 [P] [US1] Implement TagService.create_tag in backend/app/services/tag.py
- [ ] T013 [US1] Add POST /api/v1/tags endpoint in backend/app/api/v1/tags.py
```

- `T001` — sequential id
- `[P]` — task можна виконувати паралельно (різні файли, немає dependencies)
- `[US1]` — до якої user story належить (тільки в US-фазах)
- Опис **обовʼязково містить шлях до файлу**

**Структура фаз:**

1. **Phase 1 — Setup** (project init, deps, lint)
2. **Phase 2 — Foundational** (БЛОКУЄ всі stories — schema, auth, routing)
3. **Phase 3+ — Per User Story** (P1, потім P2, потім P3) — кожна story = independent MVP increment
4. **Final Phase — Polish** (docs, perf, security hardening)

Кожна фаза закінчується **Checkpoint**-ом для незалежної валідації.

> 💡 Тести **OPTIONAL** за замовчуванням. Якщо хочете TDD — попросіть агента у промпті: «Use TDD: write failing tests first».

---

### 4.6. `/speckit.analyze` — Cross-artifact аудит (read-only)

Виконується **між `/tasks` і `/implement`**. Не змінює файли.

**Категорії перевірок:**
- **A. Duplication** — близькі дублі вимог
- **B. Ambiguity** — розмиті прикметники («швидкий», «масштабований»)
- **C. Underspecification** — дієслова без вимірюваного результату
- **D. Constitution alignment** — конфлікти з MUST-принципами (auto-CRITICAL)
- **E. Coverage gaps** — вимоги без задач, задачі без вимог
- **F. Inconsistency** — terminology drift, конфліктуючі tech choices

**Output:** markdown-звіт з findings table (severity: CRITICAL / HIGH / MEDIUM / LOW), Coverage Summary, Metrics, Next Actions.

> ⚠️ **CRITICAL знахідки = блокери**. Виправіть, перш ніж запускати `/implement`.

---

### 4.7. `/speckit.implement` — Виконання

**Поведінка:**
- Перевіряє статус усіх checklist у `specs/<feature>/checklists/`. Якщо є incomplete — запитує підтвердження.
- Генерує/оновлює `.gitignore`, `.dockerignore` тощо за detected stack.
- Виконує задачі **по фазах**:
  - Sequential там, де є dependencies
  - Parallel для `[P]`-задач
- Після кожної завершеної задачі — **ставить `[X]`** у tasks.md.
- Halts при non-parallel failure.
- В кінці — final validation проти спеки.

---

### 4.8. `/speckit.checklist` — Юніт-тести для англійської

Генерує domain-specific quality checklist, який валідує **сам текст вимог** (clarity, completeness, consistency, coverage, edge cases) — а не імплементацію.

Задає до 3 уточнюючих питань, щоб налаштувати checklist під аудиторію (lightweight pre-commit / formal release gate).

**Приклад:**

```
/speckit.checklist Сформуй UX checklist для перевірки spec.md
з фокусом на accessibility і error states.
```

---

### 4.9. `/speckit.taskstoissues` — Tasks → GitHub Issues

Через GitHub MCP створює один issue на кожну задачу з tasks.md. Перед створенням перевіряє, що remote-URL відповідає GitHub-репо (відмовиться синхронізувати в чужий repo).

---

## 5. Артефакти — повна структура

```
.
├── .specify/
│   ├── memory/
│   │   └── constitution.md          # Принципи, версія, дата ратифікації
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
│   │   └── commands/                # 9 команд як markdown-промпти
│   ├── feature.json                 # активна фіча (для downstream команд)
│   ├── init-options.json            # branch_numbering: sequential|timestamp
│   └── extensions.yml               # реєстр extension hooks
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
└── (контекст агента)
    ├── CLAUDE.md  /  GEMINI.md  /  AGENTS.md  /  QWEN.md
    ├── .github/copilot-instructions.md
    ├── .cursor/rules/specify-rules.mdc
    └── .windsurf/rules/specify-rules.md
```

> 💡 **Активна фіча** визначається через `.specify/feature.json` (з v0.8.1), тому команди працюють навіть якщо назва git-branch розійшлася зі specs-папкою.

---

## 6. Pipeline workflow (як артефакти течуть)

```
┌──────────────┐
│ constitution │  (один раз; bumps semver при змінах)
└──────┬───────┘
       │
       ▼
┌──────────────┐     ┌──────────────┐
│   specify    │────▶│   clarify    │  (рекомендовано перед /plan)
└──────┬───────┘     └──────┬───────┘
       │                    │
       └────────┬───────────┘
                ▼
        ┌──────────────┐     ┌──────────────┐
        │     plan     │────▶│  checklist   │  (опційно)
        └──────┬───────┘     └──────────────┘
               │
               ▼
        ┌──────────────┐
        │    tasks     │
        └──────┬───────┘
               │
               ▼
        ┌──────────────┐
        │   analyze    │  (read-only; CRITICAL = блокер)
        └──────┬───────┘
               │
               ▼
        ┌──────────────┐
        │  implement   │
        └──────────────┘
```

---

## 7. Встановлення — деталі

### Передумови

- Linux / macOS / Windows (PowerShell 7+ для офлайн; WSL більше не обовʼязковий)
- **Python 3.11+**
- **uv** (рекомендовано) або **pipx**
- **Git**
- AI-агент (Claude Code / Copilot / Cursor / Gemini / Codex / ...)

### Варіанти встановлення

**One-shot через uvx** (без постійного встановлення):
```bash
uvx --from git+https://github.com/github/spec-kit.git@vX.Y.Z specify init <PROJECT>
uvx --from git+https://github.com/github/spec-kit.git specify init . --integration claude
uvx --from git+https://github.com/github/spec-kit.git specify init --here --integration copilot
```

**Persistent install** (рекомендовано для регулярної роботи):
```bash
uv tool install specify-cli --from git+https://github.com/github/spec-kit.git@vX.Y.Z
# або
pipx install git+https://github.com/github/spec-kit.git@vX.Y.Z

specify version
specify check
specify integration list
```

> ⚠️ Єдине офіційне джерело — `github.com/github/spec-kit`. Будь-який пакет на PyPI з назвою `specify-cli` *НЕ* від GitHub.

### Прапори `specify init`

| Прапор | Призначення |
|--------|-------------|
| `<name>` або `.` | Нова папка / поточна папка |
| `--here` | Ініціалізація у поточній директорії |
| `--force` | Не питати підтвердження для непорожньої папки |
| `--integration <key>` | Вибрати агента (`claude`, `copilot`, `gemini`, ...) |
| `--integration-options "<opts>"` | Напр. `--skills` для Codex/Copilot |
| `--script sh\|ps` | Примусово вибрати тип скриптів |
| `--ignore-agent-tools` | Не перевіряти, чи встановлено CLI агента |
| `--no-git` | Пропустити git init (deprecated, видалено в v0.10.0) |
| `--ai <key>` | Legacy alias for `--integration` |
| `--preset <name>` | Спільнотний preset (наприклад, `healthcare-compliance`) |

### Air-gapped install (для closed enterprise мереж)

На підключеній машині:
```bash
git clone https://github.com/github/spec-kit.git
cd spec-kit
pip install build
python -m build --wheel --outdir dist/
pip download -d dist/ dist/specify_cli-*.whl
```

Перенесіть `dist/` на цільову машину, потім:
```bash
pip install --no-index --find-links=./dist specify-cli
```

(Той самий OS і Python-версія обовʼязкові.)

---

## 8. Best Practices

### DO

- ✅ **Specs technology-free** — стек йде в `plan.md`, не в `spec.md`.
- ✅ **Завжди `/clarify` перед `/plan`** (виняток — exploratory spike).
- ✅ **Measurable success criteria** — «User can complete checkout in under 3 minutes», а не «API responds <200ms».
- ✅ **Запуск `/analyze` перед `/implement`** — CRITICAL знахідки = блокер.
- ✅ **Phased implementation** — реалізуйте P1 → валідуйте MVP → потім P2/P3. Запобігає context saturation.
- ✅ **Constitution як реальний контракт** — версіонуйте, оновлюйте, посилайтесь.
- ✅ **Один branch — одна фіча** — `specs/001-foo/` ↔ branch `001-foo` ↔ один PR.
- ✅ **Pin tag (`@vX.Y.Z`)** замість `main` для стабільності CI.
- ✅ **Брат push back** — питайте агента: «Чому ти додав цей компонент? До якої вимоги він трасується?»

### DON'T

- ❌ Не приймайте перший draft спеки як фінальний — ітеруйте через `/clarify`.
- ❌ Не змішуйте tech stack у `spec.md`.
- ❌ Не пропускайте `constitution.md` — без неї `Constitution Check` беззубий.
- ❌ Не залишайте більше 3 `[NEEDS CLARIFICATION]` маркерів — краще defaults у Assumptions section.
- ❌ Не дозволяйте задачам без `[Story]`-лейбла або без шляху до файлу.
- ❌ Не запускайте `/implement` із incomplete checklist і не кажіть «yes» автоматично — це руйнує quality gate.
- ❌ Не давайте агенту research broad topics («React in general»). Тільки narrow targeted.
- ❌ Не ігноруйте Sync Impact Report після `/constitution` — залежні шаблони можуть тихо drift-нути.

---

## 8.5. Project Memory — багаторівнева архітектура

Spec-kit веде «пам'ять» проекту на трьох рівнях. Розуміння цієї архітектури критично важливе для правильного використання інструменту в довгостроковій перспективі.

### Рівень 1 — Інваріанти (повільні зміни)

Файли, які описують *проект як ціле* і не прив'язані до конкретної фічі:

- **`.specify/memory/constitution.md`** — інженерні принципи. Те, що не змінюється від фічі до фічі (Test-First, Type Safety, API-First тощо). Versioned semver, з ratification і amendment процесом. Деталі — у `CONSTITUTION_GUIDE.md`.
- **`.specify/init-options.json`** — конфіг spec-kit для проекту: `branch_numbering: sequential|timestamp`, тип скриптів (`sh`/`ps`).
- **`.specify/extensions.yml`** — реєстр extension hooks (`before_specify`, `after_plan`, `before_implement`, etc.).
- **`.specify/feature.json`** — поточна активна фіча (для downstream команд). З версії 0.8.1 працює навіть якщо назва branch розійшлась з папкою.

### Рівень 2 — Живий контекст агента (швидкі зміни)

Файл, який AI-агент **читає на кожне нове повідомлення**. Залежить від агента:

| Агент | Файл контексту |
|-------|---------------|
| Claude Code | `CLAUDE.md` |
| Gemini CLI | `GEMINI.md` |
| Codex / generic | `AGENTS.md` |
| Qwen Code | `QWEN.md` |
| GitHub Copilot | `.github/copilot-instructions.md` |
| Cursor | `.cursor/rules/specify-rules.mdc` |
| Windsurf | `.windsurf/rules/specify-rules.md` |

Spec-kit пише в цей файл між маркерами:

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

При кожному `/speckit.plan` цей блок оновлюється — агент учиться з актуального стану проекту, а не починає з нуля. **Поза маркерами файл — ваш**, можете писати свої нотатки, spec-kit їх не зачіпає.

### Рівень 3 — Історія рішень (immutable)

`specs/` — це не «пам'ять» у класичному сенсі, а **архів інженерних рішень**, який накопичується протягом життя проекту.

```
specs/
├── 001-user-auth/
│   ├── spec.md         ← що було вирішено (WHAT)
│   ├── plan.md         ← Constitution Check, технічні рішення
│   ├── research.md     ← чому саме так (Decision/Rationale/Alternatives)
│   ├── data-model.md   ← entities на момент фічі
│   └── contracts/      ← OpenAPI snapshot
├── 002-search-filter/
└── 003-tags/
```

Через 6 місяців нова людина може прочитати `research.md` із `001-user-auth` і зрозуміти, *чому* ми обрали JWT через FastAPI-Users, а не Auth0 — це задокументовано у форматі Decision/Rationale/Alternatives. Це жива архівна пам'ять, яку агент може читати при питаннях типу «як у нас зроблено auth?».

> ⚠️ **На масштабі (200+ specs)** flat-структура без догляду стає смітником. Дивіться окремий документ `SPECS_HYGIENE.md` про lifecycle states, archival pattern, domain grouping і quarterly grooming ritual.

### Як це працює на практиці

Сценарій: ви відсутні три тижні, повертаєтесь, не пам'ятаєте контексту. Відкриваєте Claude Code — агент *вже знає*:

1. Прочитав `CLAUDE.md` — бачить актуальний стек і референс на конституцію.
2. При `/speckit.specify` для нової фічі — читає `constitution.md` і дотримується принципів.
3. У `/speckit.plan` — читає попередні `specs/*/data-model.md`, щоб новий код консистентно інтегрувався з існуючими entities.
4. Питання типу «чому в нас auth через JWT?» — агент відкриває `specs/001-user-auth/research.md` і відповідає з конкретним rationale.

Тобто проектна пам'ять — це **не один файл**, а структурована файлова система. Жодної бази даних, жодного хмарного state. Усе портабельне, переглядається людиною, версіонується разом з кодом.

---

## 8.6. Просунуті агентні можливості

Spec-kit використовує можливості host-агента (Claude Code, Codex, Cursor) глибше, ніж може здатись. Що саме вбудовано:

### Skills mode (Claude / Codex / Kimi / Vibe)

Для цих агентів spec-kit за дефолтом встановлюється **не** як список markdown-промптів, а як **agent skills**:

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

Кожен skill — окрема директорія з `SKILL.md` + frontmatter (name, description). Що це дає:

- **Progressive disclosure** — skill завантажується в контекст тільки коли агент розуміє, що треба його використати. Не висить постійно.
- **Token-efficient** — повний промпт `/speckit.plan` (300+ рядків інструкцій) не з'їдає контекст, поки ви реально не запускаєте plan.
- **Skill-by-skill ізоляція** — `/speckit.specify` не «знає» повного промпту `/speckit.plan` — кожен self-contained.

У Codex CLI skills-режимі префікс інший: `$speckit-specify` замість `/speckit.specify`.

Для агентів без skills (Copilot, Cursor, Gemini, Windsurf) команди ставляться як markdown / TOML / YAML промпти — функціонально те саме, але без skills-оптимізації.

### Sub-agent dispatching у `/speckit.plan`

У промпті `templates/commands/plan.md` є **прямий патерн для Task tool / sub-agents**:

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

Це означає: на фазі Research ваш AI-агент *дійсно* spawn-ить кілька паралельних під-агентів — кожен робить свій research — і збирає результати у `research.md`. Без цього на 5 невідомих ви платили б 5x токенів і часу sequentially.

### Parallel markers `[P]` у `/speckit.implement`

`/speckit.tasks` маркує задачі як `[P]` (parallelizable — різні файли, нема залежностей), а `/speckit.implement` ці маркери **використовує**:

```
[Phase 3] T009-T012 (US1)...
  T009-T011 (parallel) — pytest tests written, running...
  → 3 tests PASS in parallel ✅
```

Це не теоретичний паралелізм — у chat'і Claude Code ви побачите, як агент стартує 3–4 sub-agents для `[P]`-задач одночасно. Послідовні (без `[P]`) виконуються одна за одною.

### Hooks через `extensions.yml`

Механізм розширення (early-stage, але працює):

```yaml
hooks:
  before_specify:
    - id: speckit_git_feature
      enabled: true
      extension: "speckit-core"
      command: "create_feature_branch"
      description: "Створити feature branch перед /specify"

  after_implement:
    - id: my_post_test
      enabled: true
      optional: true
      extension: "my-extension"
      command: "run_smoke_tests"
      description: "Запустити smoke tests після /implement"
      prompt: "Запустити smoke tests?"
```

Точки розширення: `before_specify`, `after_specify`, `before_plan`, `after_plan`, `before_tasks`, `after_tasks`, `before_implement`, `after_implement`.

Hooks — це **shell-команди або кастомні extensions**, які виконуються до/після команди spec-kit. Можуть:

- Запустити власного sub-agent через MCP.
- Покликати скрипт на Python з власною логікою.
- Заблокувати команду, якщо передумови не виконані.

> 💡 **Найчастіша проблема** у початківців: `before_specify` hook (створення feature branch) **не спрацьовує** — `extensions.yml` відсутній або hook вимкнений. Лікування — перевірте `cat .specify/extensions.yml` і `.specify/scripts/bash/create-new-feature.sh`. Якщо файлів нема — виконайте `specify integration upgrade` або вручну створюйте branch перед `/specify`.

### Що **не** вбудовано

Чесно про обмеження. Spec-kit залишається перш за все **prompt library + skills**, не повноцінний agent framework. Що відсутнє:

- **Multi-agent orchestration з ролями** (як CrewAI, AutoGen) — немає концепту «Architect agent → Reviewer agent → Implementer agent». Якщо потрібно — будуєте через hooks + кастомні MCP servers.
- **Persistent agent memory across sessions** окрім markdown — немає вектор-сховища, semantic search по історії specs. Інтегруйте Pinecone / Weaviate через MCP самі.
- **Background daemons / watchers** — немає вбудованих тригерів типу «автоматично запусти `/analyze` після push». Робиться через GitHub Actions.
- **Custom sub-agent definitions** — на відміну від Claude Code's native sub-agent system, spec-kit прямо цього не підтримує. Але через hooks ви можете викликати свого агента.
- **Cross-feature analysis** — `/speckit.analyze` дивиться на одну активну фічу. Аналіз «чи 3 фічі суперечать архітектурно» — окреме рішення команди (ADR-процес).

---

## 9. Reference: ключові файли в репозиторії spec-kit

Якщо потрібно копнути глибше:

- `spec-driven.md` — методологічне есе (~412 рядків, MUST READ)
- `AGENTS.md` — як додати свого агента
- `CHANGELOG.md` — історія версій
- `templates/commands/{constitution,specify,clarify,plan,tasks,analyze,implement,checklist,taskstoissues}.md` — самі промпти команд (можна модифікувати локально через extensions)
- `templates/{constitution,spec,plan,tasks,checklist}-template.md` — шаблони артефактів
- `docs/{installation,quickstart,upgrade,index}.md` — офіційна документація
- `src/specify_cli/__init__.py` — entry-point CLI, всі прапори
- `src/specify_cli/integrations/` — реєстр 30+ агентів

---

## 10. Що читати далі

### Базовий блок — практика на тренувальному проекті

1. **`SPEC_KIT_WORKFLOW_GUIDE.md`** — наскрізне проходження SDD-2 (Search & Filter) на тренувальному проекті: від `/specify` до `/implement` з повним текстом усіх артефактів.
2. **`SPEC_KIT_USE_CASES.md`** — 9 сценаріїв за рівнями складності.
3. **`TASKS.md` / `TASKS_JIRA.md`** — практичний бекенд із 7 задач для самостійної роботи.

### Поглиблений блок — для команд і реальних проектів

4. **`SCRUM_INTEGRATION.md`** — інтеграція spec-kit зі Scrum-церемоніями: грумінг, планінг, стендап, демо, ретро. Матриця ролей. Jira mapping. DoR/DoD.
5. **`CONSTITUTION_GUIDE.md`** — глибокий гайд по `.specify/memory/constitution.md`: як писати, як еволюціонувати, реальний приклад v1.0 → v2.x за 12 місяців.
6. **`SPECS_HYGIENE.md`** — підтримка `specs/` на масштабі (100+ фіч): lifecycle states, archival pattern, domain hierarchy, manifest, quarterly grooming, скрипти.

> 🚀 **Готові? Переходьте до `SPEC_KIT_WORKFLOW_GUIDE.md`** — там покажемо, як це все виглядає на практиці.
