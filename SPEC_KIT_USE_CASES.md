# SPEC-KIT USE CASES — Сценарії за рівнями складності

> Каталог із 9 сценаріїв застосування spec-kit. Від найпростіших (Beginner) до enterprise-патернів. Кожен сценарій — це шаблон для реальної ситуації, з якою ви зіткнетесь у роботі.

Як читати:

- **Level**: складність *для застосування*, не складність задачі.
- **When**: конкретні тригери — коли саме обирати цей сценарій.
- **Time**: реалістичний часовий бюджет.
- **Commands**: які slash-команди потрібні.
- **Walkthrough**: коротке проходження.
- **Pitfalls**: що зазвичай ламається.

---

## Сценарій 1 — Quick Spec Flow

**Level**: Beginner
**When**: Маленька ізольована фіча, баг-фікс, або refactor одного модуля. Зміна локальна, скоуп ясний.
**Time**: 30–60 хв
**Commands**: `/speckit.specify` → `/speckit.plan` → `/speckit.implement`

### Walkthrough

```
/speckit.specify Виправити баг: при видаленні item старший за 30 днів,
soft-delete не записує deleted_at у БД. Має проставлятись timestamp UTC.

/speckit.plan Backend-only зміна. Оновити ItemService.delete_item.
Без міграцій, поле deleted_at вже є.

/speckit.implement
```

Пропускаємо `/clarify`, `/tasks`, `/analyze` — для маленької зміни ці кроки створюють зайвий шум, а спека достатньо однозначна.

### Коли НЕ застосовувати

- Зміна торкається 3+ файлів і кількох модулів — використайте Сценарій 2.
- Невідомий стек/архітектура — використайте Сценарій 7 (Brownfield Discovery).

### Pitfalls

- Спокуса не описувати в спеці edge case → відсутній тест → баг повертається.
- «Маленька фіча» виявилась великою на середині `/implement` — STOP, зробіть `/clarify` і `/tasks` ретроактивно.

---

## Сценарій 2 — Brownfield Feature (повний цикл)

**Level**: Basic
**When**: Нова фіча в існуючому проекті. Це **типовий** робочий випадок у компанії.
**Time**: 2–4 години (не враховуючи підготовку constitution.md)
**Commands**: `/speckit.specify` → `/speckit.clarify` → `/speckit.plan` → `/speckit.tasks` → `/speckit.analyze` → `/speckit.implement`

### Walkthrough

Дивіться **`SPEC_KIT_WORKFLOW_GUIDE.md`** — там повне проходження SDD-2 (Search & Filter) з прикладами всіх артефактів.

### Особливості

- Constitution вже існує — `/speckit.constitution` НЕ запускаємо.
- В `/specify` коротко описуємо інтеграцію з існуючими сутностями (`Item`, `User`).
- В `/plan` обовʼязково згадуємо існуючий стек — інакше агент може запропонувати щось чуже (Express замість FastAPI).

### Pitfalls

- Забути запустити `/clarify` → отримати plan з помилковими припущеннями.
- Скопіпастити `/specify` промпт зі старого тікета у Jira, де згадано стек → стек просочується в спеку.
- Не запустити `/analyze` → пропустити CRITICAL Coverage gap.

---

## Сценарій 3 — Constitution Setup (нова команда)

**Level**: Basic
**When**: Старт нового проекту АБО введення SDD у команду, яка ще не використовувала spec-kit.
**Time**: 1 робочий день (з обговореннями в команді)
**Commands**: `/speckit.constitution` → дискусія → доопрацювання

### Walkthrough

```
/speckit.constitution Створи принципи для команди з 6 інженерів,
що працює над B2B SaaS на Python (FastAPI) + React.

Test-First (NON-NEGOTIABLE): pytest, 75% coverage на нові модулі,
mutation testing раз на спринт.

Type Safety (MUST): Python 3.11+ pydantic, TypeScript strict.

API contracts (MUST): OpenAPI generator before code, breaking changes
тільки з версіонуванням API.

Observability (MUST): structlog, OTEL traces, кожен endpoint логується
з trace_id.

Performance (SHOULD): backend p95 < 200ms на ендпоінти списків,
< 500ms на агрегації.

Simplicity (SHOULD): no new deps without RFC. Замість абстракцій —
explicit code.

Governance: принципи MUST змінюються через RFC на github discussions
+ 2 review від staff+. Constitution version: semver.
```

Результат: `.specify/memory/constitution.md` v1.0.0.

### Доопрацювання в команді

1. Прочитати разом на зустрічі.
2. Кожен інженер пропонує свою правку → обговорення.
3. Після консенсусу — повторно запускаєте `/speckit.constitution` з оновленими формулюваннями.
4. Bump версії: PATCH для wording, MINOR для нового принципу, MAJOR для зміни/видалення.
5. Кожна зміна — окремий PR з обґрунтуванням.

### Pitfalls

- **Маркетингові принципи**: «We value craftsmanship» — не testable, спалюємо.
- **Принципи без rationale**: «100% coverage» без пояснення *чому* → команда саботує.
- **Перебір MUST**: усе MUST → нічого не MUST. Залиште 4–6 справжніх MUST + рештa SHOULD.
- **Constitution як waterfall** — пишемо раз і не оновлюємо. Версіонуйте і ревʼюйте раз на квартал.

---

## Сценарій 4 — Re-spec / Reset (зрозуміли, що йдемо не туди)

**Level**: Basic
**When**: Після `/plan` або під час `/implement` зрозуміли, що бракує вимог, або змінився скоуп.
**Time**: 30–90 хв
**Commands**: `/speckit.clarify` повторно → ручна правка `spec.md` → `/speckit.plan` (re-run) → `/speckit.tasks` (re-run) → `/speckit.analyze`

### Walkthrough

Уявіть, ви на половині `/implement` SDD-2 (Search). Раптом продакт каже: «Ще треба пошук по description, а не тільки по title».

```bash
# 1. Зупиняємо implement
# 2. Відкатуємо некомічений код:
git stash

# 3. Re-spec:
```

```
/speckit.specify (edit) Розширити скоуп: пошук тепер по двох полях —
title AND description (substring match, OR-логіка). Counter "Found N items"
все ще показує загальну кількість матчів.

Решта — без змін.
```

Агент модифікує існуючий `spec.md`. Запускаємо:

```
/speckit.clarify     # перевірка нових двозначностей
/speckit.plan        # перегенерація плану
/speckit.tasks       # нові задачі
/speckit.analyze     # аудит
```

`tasks.md` автоматично адаптується. Після цього:

```
/speckit.implement
```

Агент бачить, що деякі `[X]` уже зроблені (попередні задачі) — виконує тільки нові/змінені.

### Pitfalls

- **Не тримати `git stash`**: код накладається на новий tasks.md → конфлікти.
- **Забути bump constitution**: якщо changescope порушує принцип Simplicity, треба або обґрунтувати в Complexity Tracking, або переглянути конституцію.
- **Не оновити PR description**: ревʼюєр здивується, чому код не відповідає старій спеці.

> 💡 **Ключове правило**: краще передумати на фазі specs (1 година), ніж після `/implement` (тиждень рефакторингу).

---

## Сценарій 5 — Team Collaboration

**Level**: Intermediate
**When**: Впровадження spec-kit у команду 5–10 людей.
**Time**: 1–2 спринти на адаптацію
**Commands**: усі + GitHub MCP `/speckit.taskstoissues`

### Walkthrough

#### Хто пише constitution.md

Tech lead + staff engineer. Мінімум 2 людини, максимум 4 — щоб не загрузнути в bikeshedding.

#### Як ревʼюити specs

1. PR на `specs/<feature>/spec.md` створюється **до** `/plan`.
2. Ревʼєвери (PM + 1 інженер) дивляться **тільки spec.md** — стек ще не вибраний.
3. Після approve — той самий PR доповнюється `plan.md`, `tasks.md`.
4. Окремий ревʼю на `plan.md` (фокус на Constitution Check, Complexity Tracking).
5. Final PR з кодом — посилається на specs/<feature>/.

#### Як ділити роботу

| Артефакт | Хто | Коли |
|----------|-----|------|
| `spec.md` | PM + інженер парою | Перший день спринту |
| `plan.md` | Tech lead інженер | Другий день |
| `tasks.md` | Згенеровано → ревʼю | Третій день |
| Імплементація | Будь-який інженер | Решта спринту |

Один `specs/<feature>/` = один PR = один тікет у Jira/Linear.

#### Інтеграція з Jira/Linear

```
/speckit.taskstoissues
```

Створює GitHub issues. Через GitHub→Jira інтеграцію (наприклад, Atlassian Marketplace app `Issues for Jira`) issues автоматично копіюються в Jira.

#### Конфлікт між constitution і реальністю

- Інженер хоче додати lib, що порушує Simplicity.
- Створює RFC у `docs/rfc/<num>-<title>.md`.
- 2 staff approve → конституція оновлюється (`/speckit.constitution`) → MINOR bump.

### Pitfalls

- **«Constitution by Tech Lead»** без команди → всі ігнорують.
- **Ревʼю `spec.md` форматально** → виявляють проблему на code review → переробка.
- **PM пише `spec.md` сам** → інженерські вимоги (rate limit, observability) випадають.
- **Не оновлюють constitution** → стає «mythologized document» → нові інженери не вірять.

---

## Сценарій 6 — Greenfield Project

**Level**: Intermediate
**When**: Створюється новий проект з нуля.
**Time**: 1–2 дні до першого PR
**Commands**: `specify init <new-project>` → `/speckit.constitution` → `/speckit.specify` (vision) → `/speckit.specify` (per epic) → ... → `/speckit.implement`

### Walkthrough

```bash
specify init taskify --integration claude
cd taskify
git init
```

```
/speckit.constitution Створи принципи для нового pet-проекту:
Library-First, CLI Interface Mandate (кожен модуль має CLI),
Test-First, Observability, Simplicity (KISS).
```

```
/speckit.specify Taskify — це SaaS для управління задачами в малих
командах (5–15 осіб). Вища ціль: дати lightweight Trello-альтернативу
з фокусом на kanban і простотою.

Основні epics:
1. Auth & Workspaces (multi-tenant)
2. Boards & Lists & Cards (drag-and-drop)
3. Comments & @mentions
4. Notifications (in-app + email)
5. API for integrations

Цей spec — high-level vision. Деталі будуть у per-epic spec-ах.
```

Створено `specs/001-vision/spec.md`.

```
/speckit.specify Реалізуй Epic 1: Auth & Workspaces.
Multi-tenant з ізоляцією даних на рівні workspace_id.
Email + password auth через FastAPI-Users.
Workspace ↔ User M2M через WorkspaceMember (з ролью: owner/admin/member).
```

Створено `specs/002-auth-workspaces/spec.md`.

Далі — повний цикл: `/clarify` → `/plan` → `/tasks` → `/analyze` → `/implement`.

#### Послідовність епіків

```
001-vision (тільки spec, без plan/implement) ← живе як reference
002-auth-workspaces ← повний цикл
003-boards ← повний цикл (depends on 002)
004-comments-mentions ← (depends on 003)
005-notifications ← (parallel with 004)
006-public-api ← після всього
```

### Pitfalls

- **Описати все в одній спеці** → 50-сторінковий монстр, який ніхто не прочитає.
- **Vision без Definition of Done** → проект ніколи не «закінчується».
- **Стрибати в `/plan` без vision** → через 3 епіки виявляється архітектурна несумісність.

---

## Сценарій 7 — Brownfield Discovery (адаптація spec-kit до незнайомої кодбази)

**Level**: Intermediate
**When**: Заходите в репозиторій, який ви бачите вперше — і потрібно додати фічу.
**Time**: 2–6 годин (включаючи discovery)
**Commands**: discovery (вручну + AI) → `/speckit.specify` → решта стандартного циклу

### Walkthrough

#### Phase 0 — Discovery (без spec-kit)

```
[chat with AI agent, без slash-команди]

Перед тим як створювати спеку — допоможи розібратися з кодбазою.

1. Покажи структуру: backend/, frontend/, infra/.
2. Які основні моделі в backend/app/models/?
3. Які API-роути зараз існують? Покажи routerи з /api/v1/.
4. Як організована авторизація?
5. Чи є існуючі тести? Як вони запускаються?

Узагальни в 200 словах.
```

#### Phase 1 — Architecture brief

```
Запиши узагальнення в docs/architecture-brief.md.
Включи:
- ER-діаграма ASCII
- Список endpoints
- Auth flow
- Тестова стратегія
- Точки розширення для нових фіч
```

Тепер у вас є контекст для нормального `/specify`.

#### Phase 2 — Standard cycle

```
/speckit.constitution    # якщо ще немає
/speckit.specify ...     # тепер з knowledge of existing code
/speckit.clarify
/speckit.plan
/speckit.tasks
/speckit.analyze
/speckit.implement
```

#### Альтернатива

Існує спільнотний extension **`spec-kit-brownfield`**, який автоматизує discovery. Інсталюється через `extensions.yml`. Не рекомендується для production без вивчення коду самостійно — automated discovery може пропустити нюанси.

### Pitfalls

- **Стрибнути одразу в `/specify`** → агент імпровізує, бо не знає вашу архітектуру.
- **Discovery без артефакту** → втрачено в історії чату, наступний інженер повторює.
- **Спека ігнорує існуючі патерни** → код стилістично контрастує з рештою.

---

## Сценарій 8 — Refactor під SDD (legacy → spec-kit)

**Level**: Advanced
**When**: Існуючий складний модуль, який треба переписати, але документація відсутня.
**Time**: 1–2 тижні
**Commands**: discovery → `/speckit.specify` (reverse-engineering) → `/speckit.plan` → `/speckit.implement`

### Walkthrough

#### Reverse-engineering existing behavior

```
/speckit.specify Створи специфікацію для існуючого модуля
backend/app/services/notification_service.py.

Прочитай файл і всі його тести (якщо є). Опиши actual behavior:
- Які типи нотифікацій підтримуються?
- Які тригери?
- Які канали (email, in-app, push)?
- Які retry-механізми?
- Які edge cases (помилки, дублі, rate limits)?

Це reverse-engineered specification. Помітити її як `## Reverse-Engineered`
у заголовку. Зазначити в Out of Scope: "feature parity з existing code".
```

Створено `specs/010-notifications-refactor/spec.md` із описом as-is поведінки.

#### Аналіз дельти

```
/speckit.specify (edit) Тепер додай to-be вимоги:
1. Підтримку push-нотифікацій (зараз тільки email).
2. Rate limiting per user (10/min).
3. Centralized retry queue (зараз inline).
4. Observability metrics для delivery rate.

Збережи розділ "Existing Behavior" як reference.
```

#### План переходу

```
/speckit.plan Стратегія міграції: Strangler Fig pattern.
Phase 1: створити нові модулі поряд (notification_v2_service.py).
Phase 2: feature flag для перемикання.
Phase 3: deprecate v1, видалити після місяця.

Rollback: feature flag.
```

`/tasks` згенерує задачі, що включають подвійну реалізацію + інтеграційні тести на feature parity.

### Pitfalls

- **Refactor без spec.md** → за місяць нікому не зрозуміло, *що* саме переписали.
- **Big-bang refactor** → 5K LoC PR, ніхто не ревʼюїть.
- **Без rollback-плану** → у production щось ламається, panic.

---

## Сценарій 9 — Compliance / Regulated Domain

**Level**: Advanced
**When**: Healthcare, finance, government — є compliance-вимоги (HIPAA, SOC2, GDPR).
**Time**: +30–50% до базового workflow
**Commands**: усі + `/speckit.checklist` для compliance

### Walkthrough

#### Constitution з compliance-блоком

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
Compliance officer reviewer на всіх PR з тегом `compliance`.
```

#### Compliance checklist на кожну фічу

```
/speckit.checklist Generate HIPAA compliance checklist for spec.md.
Focus on:
- PHI identification (which fields are PHI)
- Encryption coverage
- Audit logging coverage
- Access control verification
- Data retention rules
```

Згенеровано `specs/<feature>/checklists/hipaa.md`. **`/speckit.implement` зупиниться**, якщо цей checklist не повністю ✅.

#### Preset для compliance

```bash
specify init my-project --preset healthcare-compliance --integration claude
```

(Якщо preset існує — перевірте `specify integration list` і community presets.)

#### Поглиблений audit перед relase

```
/speckit.analyze
```

Виводить додаткові категорії: `Compliance Coverage`, `PHI Tracking`, `Audit Trail Gaps`. Будь-який CRITICAL — release blocker.

### Pitfalls

- **Compliance як «галочка» в Jira** → реально не виконано → fail при аудиті.
- **Constitution без compliance principles** → агент не блокує небезпечні фічі.
- **Custom checklist без revision history** → compliance officer не довіряє.
- **Імплементація без `/analyze`** → CRITICAL знахідки прилітають у production.

---

## Підсумкова таблиця

| Сценарій | Level | Час | Ключові команди |
|----------|-------|-----|----------------|
| 1. Quick Spec Flow | Beginner | 30–60 хв | specify → plan → implement |
| 2. Brownfield Feature | Basic | 2–4 год | повний цикл |
| 3. Constitution Setup | Basic | 1 день | constitution + дискусія |
| 4. Re-spec / Reset | Basic | 30–90 хв | clarify → spec edit → plan/tasks re-run |
| 5. Team Collaboration | Intermediate | 1–2 спринти | усі + taskstoissues |
| 6. Greenfield | Intermediate | 1–2 дні | init → vision → per-epic |
| 7. Brownfield Discovery | Intermediate | 2–6 год | discovery → стандартний цикл |
| 8. Refactor під SDD | Advanced | 1–2 тижні | reverse-engineer → strangler fig |
| 9. Compliance | Advanced | +30–50% | + checklist + analyze з compliance-mode |

---

## Рекомендації — з чого почати?

### Якщо ви новачок у spec-kit
→ **Сценарій 1** (Quick Spec Flow) на одному маленькому багу. Потім **Сценарій 2** на середній фічі.

### Якщо вводите spec-kit у команду
→ **Сценарій 3** (Constitution) як перший крок. Потім **Сценарій 5** (Team Collaboration) як шаблон процесу.

### Якщо у вас legacy-проект
→ **Сценарій 7** (Brownfield Discovery) перед першим `/specify`. Не пропускайте discovery.

### Якщо стартуєте новий проект
→ **Сценарій 6** (Greenfield) із vision-spec → per-epic spec-и.

### Якщо у вас compliance-вимоги
→ **Сценарій 9** ще на стадії конституції — інакше переробляти втричі важче.

### Якщо щось пішло не так
→ **Сценарій 4** (Re-spec / Reset). Не намагайтесь героїчно довести `/implement` до кінця, якщо спека хибна.

---

> 🚀 **Готові до практики?** Перейдіть до `TASKS.md` — там 7 практичних задач, на яких можна відточити кожен сценарій.
