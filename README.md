# Spec-Kit Training — Навчальний план для девелоперів

> Практичний курс з Spec-Driven Development з використанням **GitHub Spec-Kit** на реальному full-stack проекті.

Курс розроблено за зразком навчальних програм для AI-driven development. Мета — провести інженера від нульового знайомства зі spec-kit до впевненого впровадження методології в робочих проектах команди.

---

## Зміст курсу

### Базовий блок (мінімум для одинокого вивчення)

| Документ | Призначення |
|----------|-------------|
| `README.md` *(цей файл)* | Точка входу. 4 стадії + 6 сценаріїв + рекомендації |
| `PROJECT_SETUP.md` | Стадія 1: розгортання тренувального проекту (FastAPI + React) |
| `SPEC-KIT-docs.md` | Довідник: філософія SDD, команди, артефакти, project memory, advanced features |
| `SPEC_KIT_WORKFLOW_GUIDE.md` | End-to-end проходження однієї задачі (SDD-2) |
| `SPEC_KIT_USE_CASES.md` | 9 сценаріїв від Beginner до Advanced |
| `TASKS.md` | Бекенд із 7 практичних задач (стислий формат) |
| `TASKS_JIRA.md` | Ті самі задачі у форматі Jira-сторі (детально) |

### Поглиблений блок (для команд і реальних проектів)

| Документ | Призначення |
|----------|-------------|
| `SCRUM_INTEGRATION.md` | Інтеграція spec-kit зі Scrum-церемоніями: грумінг, планінг, стендап, демо, ретро. Матриця ролей. Jira mapping. DoR/DoD |
| `CONSTITUTION_GUIDE.md` | Глибокий гайд по `.specify/memory/constitution.md`: як писати, як еволюціонувати, реальний приклад v1.0 → v2.x за 12 місяців |
| `SPECS_HYGIENE.md` | Підтримка `specs/` на масштабі (100+ фіч): lifecycle states, archival pattern, domain hierarchy, **5 паттернів currency maintenance** (A–E), **3 рівні автоматизації sync**, manifest, quarterly grooming, скрипти |

---

## Тренувальний проект

Курс побудовано навколо публічного шаблону **`fastapi/full-stack-fastapi-template`**:

- **Backend**: FastAPI · SQLModel · Alembic · PostgreSQL
- **Frontend**: React · TypeScript · TanStack Query · Vite
- **Інфраструктура**: Docker Compose, Traefik, GitHub Actions
- **Дефолтний логін**: `admin@example.com` / `changethis`

Вибір цього шаблону невипадковий: він достатньо реалістичний (ORM, міграції, auth, тести), але водночас компактний — повний цикл розробки фічі займає 1–3 години, що ідеально для тренінгу.

> 💡 **Ключовий принцип**: усі команди spec-kit, які ви будете виконувати в цьому курсі, працюють так само на будь-якому реальному проекті у вашій компанії. Тренувальний шаблон — лише пісочниця.

---

## Стадія 1 — Розгорнути проект

Виконайте `PROJECT_SETUP.md` повністю. Файл написано як агентський промпт — ви можете або робити кроки вручну, або передати його AI-агенту в окремому чаті.

```bash
git clone https://github.com/fastapi/full-stack-fastapi-template.git
cd full-stack-fastapi-template
cp .env.example .env
docker compose up -d
```

**Перевірка успіху:**

- ✅ http://localhost:5173 відкриває фронтенд
- ✅ http://localhost:8000/docs показує Swagger UI
- ✅ Логін `admin@example.com` / `changethis` працює

**Очікуваний час**: 20–40 хвилин.

---

## Стадія 2 — Що таке Spec-Driven Development

Прочитайте `SPEC-KIT-docs.md` повністю. Зверніть увагу на ці три розділи:

1. **Філософія SDD** — чому код *слугує* специфікації, а не навпаки.
2. **Команди spec-kit** — `/speckit.constitution`, `/speckit.specify`, `/speckit.clarify`, `/speckit.plan`, `/speckit.tasks`, `/speckit.analyze`, `/speckit.implement`.
3. **Артефакти** — що саме і де генерується (`.specify/`, `specs/<feature>/`).

> 💡 **Практична рекомендація**: не пропускайте розділ про філософію. Без розуміння *чому* SDD виглядає як «зайва бюрократія» — і команда відмовиться його використовувати.

**Очікуваний час**: 45–60 хвилин читання.

---

## Стадія 3 — Встановити spec-kit

Spec-kit ставиться через `uvx` (рекомендовано) або `pipx`. Працює з Claude Code, Copilot, Cursor, Gemini, Codex, Cursor, Windsurf і ще понад 25 AI-агентами.

### Передумови

- Python 3.11+
- `uv` (`pip install uv` або `brew install uv`)
- `git`
- AI-агент (Claude Code, Copilot, Cursor, або інший)

### Швидке встановлення в існуючий проект

```bash
cd full-stack-fastapi-template
uvx --from git+https://github.com/github/spec-kit.git specify init . --integration claude
```

### Перевірка

```bash
specify check
specify integration list
```

У редакторі (Claude Code / Copilot / Cursor) перевірте, що автокомпліт показує `/speckit.constitution`, `/speckit.specify`, `/speckit.plan`, `/speckit.tasks`, `/speckit.implement`.

> ⚠️ **Важливо**: офіційне джерело — лише `github.com/github/spec-kit`. Будь-який пакет на PyPI з назвою `specify-cli` *НЕ* пов'язаний з GitHub.

**Очікуваний час**: 10–20 хвилин.

---

## Стадія 4 — Практичні сценарії

Шість сценаріїв, від найпростішого до найскладнішого. Деталі — в `SPEC_KIT_USE_CASES.md`.

### Сценарій 1 — Quick Spec Flow (для маленьких змін)

| | |
|---|---|
| **Коли застосовувати** | Маленька ізольована фіча або баг-фікс |
| **Час** | 30–60 хв |
| **Команди** | `/speckit.specify` → `/speckit.plan` → `/speckit.tasks` → `/speckit.implement` |

**Приклад:**

```
/speckit.specify Додати endpoint GET /api/v1/items/random,
який повертає випадковий item з бази для авторизованого користувача.
```

Пропускаємо `/clarify` і `/analyze` — для маленької зміни ці кроки створюють зайвий шум.

> ⚠️ **Важливо**: `/speckit.tasks` пропускати **не можна** — `/speckit.implement` читає `tasks.md` як вхідні дані і блокує виконання без нього. Деталі — в `SPEC_KIT_USE_CASES.md`, Сценарій 1.

---

### Сценарій 2 — Brownfield Feature (повний цикл на існуючому проекті)

| | |
|---|---|
| **Коли застосовувати** | Нова фіча в існуючому коді (типовий випадок у компанії) |
| **Час** | 2–4 години |
| **Команди** | `/speckit.constitution` *(один раз)* → `/speckit.specify` → `/speckit.clarify` → `/speckit.plan` → `/speckit.tasks` → `/speckit.analyze` → `/speckit.implement` |

Це **основний** воркфлоу для робочих проектів. Повне проходження дивіться в `SPEC_KIT_WORKFLOW_GUIDE.md` (на прикладі задачі **SDD-2: Search & Filter**).

```
/speckit.specify Додати пошук по полю title у списку items
з debounce 300ms і лічильником "Found N items".
```

---

### Сценарій 3 — Constitution-First (нова команда / новий проект)

| | |
|---|---|
| **Коли застосовувати** | Старт нового проекту або введення SDD у команду |
| **Час** | 1 день |
| **Команди** | `/speckit.constitution` → дискусія → доопрацювання |

**Приклад промпту:**

```
/speckit.constitution Створи принципи для нашої команди: code quality
(80%+ test coverage, mandatory code review), testing standards (TDD для
core domain), UX consistency (Figma design system), performance
(<200ms p95). Додай governance: усі MUST-принципи можна змінювати
тільки через RFC + 2 review.
```

Результат: `.specify/memory/constitution.md`, який стає реальним контрактом для всіх наступних `/plan` через **Constitution Check** gates.

---

### Сценарій 4 — Reset / Re-spec (виявили, що йдемо не туди)

| | |
|---|---|
| **Коли застосовувати** | Після `/plan` зрозуміли, що бракує вимог, або змінився скоуп |
| **Час** | 30–90 хв |
| **Команди** | `/speckit.clarify` повторно → `/speckit.specify` (edit) → `/speckit.plan` (re-run) |

> 💡 **Ключове правило**: краще передумати на фазі specs, ніж після `/implement`. Один день на спеку економить тиждень рефакторингу.

---

### Сценарій 5 — Team Collaboration *(дискусія)*

Як впровадити spec-kit у команду 5–10 осіб:

- Хто пише `constitution.md` (зазвичай tech lead + staff engineer)
- Як ревʼювити specs (PR на сам файл `spec.md` ще *до* `/plan`)
- Як ділити задачі (один `specs/<feature>/` = один PR = один тікет)
- Як інтегрувати з Jira/Linear (`/speckit.taskstoissues`)
- Як вирішувати конфлікти між constitution і реальністю (RFC)

Не виконуємо команд — обговорюємо в групі.

---

### Сценарій 6 — Greenfield *(дискусія)*

Як стартувати **новий** проект через spec-kit:

```bash
specify init my-new-project --integration claude
```

Послідовність:

```
/speckit.constitution → /speckit.specify (high-level vision)
                     → /speckit.specify (per-epic features)
                     → /speckit.plan → /speckit.tasks → /speckit.implement
```

На greenfield-проекті spec-kit економить найбільше часу, бо не треба описувати «як новий код інтегрується з існуючим».

---

## Бекенд практичних задач

`TASKS.md` містить **7 задач** на тренувальному проекті, від Easy до Hard. Виберіть мінімум **3 задачі** для самостійного виконання:

| ID | Задача | Складність | SP |
|----|--------|------------|----|
| SDD-1 | Tags для items (M2M) | Easy | 5 |
| SDD-2 | Search & Filter з debounce | Easy | 3 |
| SDD-3 | Comments з permissions | Medium | 8 |
| SDD-4 | Dashboard з recharts | Medium | 8 |
| SDD-5 | Favorites з optimistic UI | Easy | 5 |
| SDD-6 | CSV Export через StreamingResponse | Medium | 5 |
| SDD-7 | Activity Log (audit middleware) | Hard | 13 |

Детальний опис у форматі Jira (Acceptance Criteria, Subtasks, Technical Notes) — у `TASKS_JIRA.md`.

---

## Рекомендований шлях для одинака (40 годин)

```
День 1 (8 год)  — Стадії 1–3 + читання SPEC-KIT-docs (включно з 8.5 і 8.6)
День 2 (8 год)  — Сценарій 1 (Quick Flow) + SDD-1 (Tags)
День 3 (8 год)  — Сценарій 2 (Brownfield) + SDD-2 (Search) — повний цикл
День 4 (8 год)  — SDD-3 або SDD-5 + SDD-6 + читання CONSTITUTION_GUIDE
День 5 (8 год)  — SDD-7 (Hard) + ретроспектива + написання внутрішнього constitution.md
```

## Рекомендований шлях для команди (workshop, 2 дні)

```
День 1 ранок    — Стадії 1–3 (всі разом)
День 1 вечір    — Сценарій 2 наживо: ведучий проходить SDD-2 від /specify до /implement
                  (учасники повторюють у себе синхронно)
День 2 ранок    — Парами: одна задача на пару (SDD-3, SDD-4, SDD-5)
День 2 вечір    — Читання SCRUM_INTEGRATION + CONSTITUTION_GUIDE +
                  написання draft constitution.md команди
```

## Рекомендований шлях для tech lead, що впроваджує SDD у команду (2 тижні)

```
Тиждень 1
  - День 1-2: Champion-pilot — самостійно проходите Сценарій 2 на 2 фічах
  - День 3:   Читання SCRUM_INTEGRATION і CONSTITUTION_GUIDE
  - День 4-5: Drafting workshop (constitution.md v1.0 з ratification)

Тиждень 2
  - День 1-3: Команда (3-4 інженери) робить пілот на 2-3 фічах із spec-kit
  - День 4:   Ретроспектива пілоту → корекції constitution + processes
  - День 5:   Прийняття рішення про повне впровадження або відмову
```

---

## Часті помилки початківців

1. **Згадують стек у `/specify`.** Спека має описувати *що* і *чому*, а не *як*. Стек — у `/plan`.
2. **Пропускають `/clarify`.** Економлять 5 хвилин — втрачають 2 години на рефакторинг.
3. **Не запускають `/analyze` перед `/implement`.** Пропускають CRITICAL-знахідки → агент пише код під неузгоджені вимоги.
4. **Залишають `[NEEDS CLARIFICATION]` маркери.** Якщо в `spec.md` є невирішені питання — `/plan` згенерує сміття.
5. **Не оновлюють `constitution.md`.** Документ застаріває за пів року, а Constitution Check продовжує посилатись на старі принципи.
6. **Запускають `/implement` з incomplete checklist.** Spec-kit запитає підтвердження — *не* кажіть «yes» автоматично.

---

## Ключові інсайти, які вийшли з курсу

Якщо ви вивчите усі сім (десять разом із поглибленим блоком) документів — наступне має увійти у вашу робочу свідомість:

**1. Spec-kit — інженерний робочий простір, не заміна Jira.** Jira залишається планувальником і трекером. Зв'язок: Jira ID у назві папки + `Resolves PROJ-XXXX` у PR.

**2. Constitution — виконавчий контракт, не лозунги.** Кожен принцип має проходити тест: «чи може агент порушити це у `/plan`, і чи Constitution Check це зловить?». Якщо ні — це не принцип. Деталі — в `CONSTITUTION_GUIDE.md`.

**3. Project memory має 3 рівні**: інваріанти (constitution, init-options), живий контекст (CLAUDE.md/AGENTS.md), історія рішень (specs/). Деталі — в `SPEC-KIT-docs.md`, секція 8.5.

**4. SP оцінюються на рівні Functional Requirements, не tasks.** FR → Jira stories; tasks.md → executable checklist у репо. Деталі — в `SCRUM_INTEGRATION.md`.

**5. На масштабі без curation `specs/` стає смітником.** Lifecycle states + quarterly grooming + domain hierarchy. **5 паттернів** підтримки актуальності specs між спрінтами (Atomic+Frozen, Supersession Chain, Living Domain Specs, Unified Spec, Hybrid) + **3 рівні автоматизації sync** (template mod, hooks, custom command). Деталі — в `SPECS_HYGIENE.md`.

**6. `/speckit.tasks` обов'язковий навіть у Quick Flow** — `/implement` без нього не працює. Скорочений флоу: пропускаєте `/clarify` і `/analyze`, але не `/tasks`.

**7. Spec-kit використовує advanced agent features**: skills mode (Claude/Codex), sub-agent dispatch у research-фазі, parallel markers `[P]` в `/implement`. Деталі — в `SPEC-KIT-docs.md`, секція 8.6.

**8. Інтеграція з командою — поетапна, не «з понеділка всі переходимо».** Champion pilot → showcase → constitution drafting → командний пілот → стандарт. План — в `SCRUM_INTEGRATION.md`.

---

## Що далі — для одинака

Після завершення курсу:

- Адаптуйте `constitution.md` під свою команду — використайте `CONSTITUTION_GUIDE.md` як шаблон процесу.
- Створіть **внутрішній шаблон** spec-kit-проекту з вашою конституцією і прикладом spec.md — нові репозиторії стартують за 5 хвилин.
- Інтегруйте `/speckit.taskstoissues` з Jira/Linear через MCP або власний скрипт.
- Проведіть workshop за цим планом для решти команди.

## Що далі — для tech lead команди

- Champion pilot → showcase → drafting workshop → командний пілот (план в `SCRUM_INTEGRATION.md` секція «Як впровадити SDD у команду»).
- Налаштуйте grooming/planning/retro ритуали згідно `SCRUM_INTEGRATION.md`.
- Через 1 квартал — перший `SPECS_HYGIENE.md` ритуал (archival + index).

---

> **Питання та фідбек**: тримайте репозиторій курсу як живий — додавайте власні сценарії, ускладнюйте задачі, фіксуйте свої best practices.
