# CONSTITUTION_GUIDE — Як писати і еволюціонувати конституцію проекту

> Глибокий гайд по `.specify/memory/constitution.md`. Що це насправді, як писати, як підтримувати в робочому стані. Реальний приклад еволюції документа за 12 місяців проекту.

---

## Що таке Constitution насправді

Не «маркетинг команди», а **виконавчий контракт**. Кожен принцип має проходити простий тест:

> Якщо я порушу його у `/speckit.plan`, чи можу побачити це у Constitution Check?

Якщо ні — це не принцип, це лозунг, видаляйте.

Constitution описує **технічні інженерні інваріанти** — те, що не змінюється від фічі до фічі. Конкретно — process-агностичні до засобів реалізації:

- **Test-First** — це процес (тести *перед* кодом). Агностично до того, чи це pytest, чи jest, чи RSpec.
- **API-First** — процес (контракт *перед* імплементацією). Агностично до OpenAPI / GraphQL / gRPC.
- **Observability** — практика (кожен endpoint емітить структуровані дані). Агностично до structlog / zap / winston.

Тобто це **«як ми приймаємо інженерні рішення»**, не «який стек ми використовуємо».

### Що Constitution **НЕ** описує

| Не описує | Куди це йде |
|-----------|-------------|
| Конкретний стек (FastAPI, React) | `plan.md` |
| Конкретні бібліотеки (pytest, structlog) | `plan.md` |
| Архітектурні рішення (CQRS, event sourcing) | ADR + `plan.md` |
| Code organization (folder structure) | Шаблони + linters |
| Scrum-процес, ритм спрінтів | Team handbook |
| Code review правила | CODEOWNERS + branch protection |

Constitution — про **інженерну дисципліну**, яку ваш AI-агент і ваші інженери дотримуються на кожній фічі, незалежно від того, *яку* фічу і *якими* інструментами робите.

---

## Анатомія робочого принципу

Кожен принцип складається з трьох частин: **rule**, **rationale**, **how to apply**.

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

### Чому всі три частини обов'язкові

**Без rationale** — через місяць принцип виглядає як абсурдна забаганка автора, через рік новачки його ігнорують. Rationale має бути конкретним фактом — інцидент, число, реальна проблема.

**Без "how to apply"** — принцип не verifiable. Constitution Check не знає, як його перевірити, агент не знає, як йому слідувати.

**Без rule** — це просто опис. Принцип має бути declarative і testable.

### Antipattern — принцип, який не працює

```markdown
## Principle 7: Quality

**Rule**: We value high-quality code.
```

Ніяк не перевіриш, ніяк не порушиш — викидаємо.

---

## MUST vs SHOULD vs MAY — не зловживайте MUST

Найпоширеніша помилка: команда пише вісім принципів, усі MUST. Через спрінт-два вони порушують пів-із-них, і MUST стає пустим звуком.

**Правило**: 4–6 справжніх MUST, решта — SHOULD або MAY. Якщо порушення принципу не зупинить merge — це не MUST.

| Рівень | Семантика | Дія Constitution Check |
|--------|-----------|------------------------|
| **NON-NEGOTIABLE** | Абсолютний закон | Блокує merge без обговорення |
| **MUST** | Обов'язково; виключення обґрунтовуються | Або виправляється, або йде в Complexity Tracking |
| **SHOULD** | Рекомендується | Помічається ⚠️, дозволено deferred |
| **MAY** | Оптимально, не обов'язково | Лише як guideline |

---

## Стандартний каталог принципів

З публічних прикладів spec-kit (`spec-driven.md` від GitHub) — стартовий набір категорій:

- **Library-First** — нова функціональність створюється як standalone module з public interface
- **CLI Interface Mandate** — кожен модуль має CLI-обгортку (text-in/text-out), щоб бути testable окремо
- **Test-First** — тести написані до коду; якщо код пройшов тест із першого разу — тест слабкий
- **Integration-First Testing** — інтеграційні тести важливіші за unit тести; mocks обмежено
- **Simplicity** — додавання залежностей вимагає обґрунтування
- **Anti-Abstraction** — забороняємо premature abstraction (DRY до third occurrence)
- **Observability** — кожен сервіс пише structured logs з trace_id; metrics on critical paths
- **Versioning** — semver на API; breaking changes тільки з major bump

Це не «бери все», а stock-каталог. Реальна конституція ідеально містить 5–8 принципів, із яких 3–4 ваші специфічні.

---

## Як написати її з нуля — процес

Найбільша помилка — *«посадили tech lead, він написав принципи за 2 години»*. Через тиждень команда мовчки нехтує половиною. Кращий шлях.

### Крок 1 — Пілот (1–2 фічі без конституції)

Champion + 1–2 інженери проходять повний spec-kit цикл на реальних задачах *без* `/speckit.constitution`. Що болить — записують. Що зайшло — теж. На виході — список із 10–15 «pain points» і «good vibes».

### Крок 2 — Drafting workshop (2 години)

Tech lead + staff + 1 product person сідають з цим списком і питають: *«який принцип, якби діяв, запобіг би цьому pain point?»*. Це дає 6–10 кандидатів.

Викидаєте дублі, дивитесь на стандартний каталог — чи спільне щось формулюється краще.

### Крок 3 — Запуск `/speckit.constitution` (15 хвилин)

У чаті агента — детальний промпт. **Не** один рядок «створи принципи», а повний контекст:

```
/speckit.constitution

Контекст: B2B SaaS на Python 3.11 + FastAPI на бекенді,
React 18 + TypeScript на фронтенді, Postgres 16, monorepo.
Команда — 6 інженерів (3 fullstack, 2 backend, 1 frontend).

Створи Project Constitution v1.0.0 із наступними принципами:

1. Test-First (NON-NEGOTIABLE)
   Rule: всі нові backend-модулі мають pytest-тести; coverage на нових
   або модифікованих файлах ≥ 75%.
   Rationale: на попередньому проекті команди було 4 регресії за квартал
   у core domain, які unit-тести зловили б; одна дійшла до production
   і призвела до 02:00 UTC rollback.
   How to apply: /speckit.tasks ставить test-задачі ПЕРЕД impl-задачами;
   CI fail при < 75% coverage diff.

2. Type Safety (MUST)
   Rule: backend MUST type-check під mypy --strict; frontend — tsc --strict.
   Any/type:ignore/as any потребують inline-коментаря з обґрунтуванням.
   Rationale: 2 з 4 incidents у попередньому проекті — runtime type errors,
   які strict typing виявив би на PR review.
   How to apply: pre-commit hook + CI gates.

3. API-First (MUST)
   Rule: кожен новий endpoint має OpenAPI fragment у specs/<feature>/contracts/
   ДО handler-коду. TS-клієнти генеруються з OpenAPI через openapi-typescript.
   Rationale: 1 з 4 incidents — schema drift backend↔frontend.
   How to apply: /speckit.plan Phase 1 MUST продукує contracts/*.yaml.

4. Observability (MUST)
   Rule: кожен endpoint і background task емітить structured log entry
   (structlog) з request_id, actor_id, duration_ms, outcome.
   Rationale: попередній incident #4 — повільний endpoint, який без
   duration_ms-метрик debug-али 3 години.
   How to apply: middleware зареєстровано глобально; /speckit.tasks Phase 6
   має задачу "verify structured logs".

5. Performance Budget (SHOULD)
   Rule: list endpoints < 300ms p95 на 10K rows; detail endpoints < 150ms p95;
   frontend FCP < 1.5s.
   Rationale: без явного бюджету регресії можуть тижнями лишатись непоміченими.
   How to apply: perf-test задача у Phase 6 для list/aggregation endpoints.

6. Simplicity (SHOULD)
   Rule: нова runtime-залежність потребує RFC у docs/rfc/.
   Dev-залежності (linter, test runner) — без RFC.
   Rationale: package.json і pyproject.toml швидко розростаються; платимо
   audit/CVE/upgrade-податок за невикористані deps.
   How to apply: /speckit.plan research.md MUST показує "Alternatives
   considered" включно зі stdlib option для нових deps.

Governance:
- Зміни до MUST/NON-NEGOTIABLE — RFC у docs/rfc/<num>-<slug>.md + 2 staff approve
- SHOULD/MAY — 1 staff approve
- Wording fix (typo, example) — 1 approve, PATCH bump
- Versioning: PATCH/wording, MINOR/новий принцип, MAJOR/видалення MUST
- Quarterly review на ретроспективі

Дата ratification: today.
Дата last_amended: today.
Версія: 1.0.0.
```

Так — це довгий промпт. Це нормально. Constitution — найважливіший документ проекту, тому 15 хвилин на промпт виправдані.

### Крок 4 — Перевірка згенерованого

Перегляньте файл уважно:

1. **Жодних `[ALL_CAPS]` placeholder-ів** не залишилось (агент мав їх замінити; іноді пропускає).
2. **Версія `1.0.0`**, дати `ratification_date` і `last_amended_date` сьогоднішні в ISO-форматі.
3. **Sync Impact Report** угорі є і має сенс.

Якщо щось не так — запустіть команду ще раз із поправкою:

```
/speckit.constitution Принцип 5 (Performance Budget) сформулюй чіткіше:
поясни, чим вимірюється p95 (testing fixture-и в CI з seed-даними).
```

Команда модифікує файл (а не пише з нуля) і bump-не PATCH версію (1.0.0 → 1.0.1).

### Крок 5 — Circulation у команді (3–5 робочих днів)

Файл відкритий для коментарів усієї команди в PR. Цикл «коментар → правка → знову `/speckit.constitution` із оновленими формулюваннями → bump».

### Крок 6 — Підкоригувати залежні шаблони

Sync Impact Report сказав, що шаблони треба оновити. Це означає, що `.specify/templates/plan-template.md` (і `spec-template.md`, `tasks-template.md`) тепер мають включити нову Constitution Check таблицю з вашими принципами.

Шаблони редагуєте руками або просите агента:

```
Онови `.specify/templates/plan-template.md` — додай у секцію Constitution
Check рядки таблиці для всіх 6 принципів.
```

Це one-time робота. Після — кожен `/speckit.plan` буде використовувати ваші принципи у Constitution Check.

### Крок 7 — Ratification commit

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

PR з ratification — нехай уся команда approve-ить. Це не бюрократія: підпис команди = команда дійсно прочитала і погодилась дотримуватись.

---

## Як тримати її живою

Тепер найскладніше — після ratification конституція починає **дрейфувати**. Команда змінює стек, додаються нові ролі, з'ясовується, що принцип занадто строгий. Без активного підтримання за пів року це «mythologized document», який ніхто не читає.

### Інструменти, які пропонує spec-kit

**Sync Impact Report.** При кожному запуску `/speckit.constitution` агент додає вгорі файлу HTML-коментар:

```html
<!-- Sync Impact Report
Modified: Principle 3 (API-First) — added gRPC requirement
Added principles: none
Removed principles: none
Templates needing update: plan-template.md (Constitution Check section)
Version bump: 1.2.0 → 1.3.0 (MINOR — broader scope)
-->
```

Це нагадування: коли в конституції щось змінилось, варто перевірити шаблони і оновити їх, якщо потрібно.

**Constitution Check у `/speckit.plan`.** Якщо ви заклали свою конституцію в шаблон, вона *активно* перевіряється на кожній фічі. Це той самий механізм, який не дає їй стати документом-привидом.

> ⚠️ Якщо у вас Constitution Check ніколи не fail-ить — або принципи занадто м'які, або шаблон їх не застосовує. Перевіряйте раз на квартал, що `plan.md`-и реально містять Constitution Check таблицю з відмітками.

### Amendment workflow

```
docs/rfc/0007-loosen-test-coverage-rule.md
├── Context: чому пропонуємо змінити
├── Current rule: цитата з constitution.md
├── Proposed change: новий текст
├── Impact: на які активні specs це впливає
├── Migration: чи треба переписувати щось у відкритих specs
└── Approval: 2 staff sign-off
```

Після merge RFC — запускаєте `/speckit.constitution` із промптом «застосуй RFC-0007» — агент зробить правку і Sync Impact Report.

**Bump versionов за semver:**

| Зміна | Bump | Приклад |
|-------|------|---------|
| Wording, fix typo, додано example | PATCH | 1.2.0 → 1.2.1 |
| Додано новий принцип, розширено scope | MINOR | 1.2.1 → 1.3.0 |
| Видалено принцип, перевизначено MUST, змінено governance | MAJOR | 1.3.0 → 2.0.0 |

### Quarterly review

На retro раз на квартал — 30 хвилин відкриваєте конституцію разом і питаєте по кожному принципу два питання:

1. *Чи реально ми його дотримуємось?* Якщо ні — або принцип нерелевантний (видаляйте), або є тертя у виконанні (виявляйте і фіксіть).
2. *Чи Constitution Check у /plan його реально перевіряв за останні 3 місяці?* Якщо ні — або у плані немає чого перевіряти, або шаблон зламаний.

Це той ритуал, який і відрізняє «живу» конституцію від «mythologized document».

---

## Сценарії змін

### Сценарій A — Змінюється стек

Наприклад, мігруємо з Express на FastAPI. Конституція *не повинна* згадувати конкретні бібліотеки чи фреймворки — це робота шаблонів і `plan.md`. Якщо ваша конституція каже «MUST use FastAPI» — це warning sign, перепишіть на «MUST use a typed Python web framework» (або взагалі видаліть, бо це деталь, яка живе у `plan.md`).

### Сценарій B — Змінюється практика

Наприклад, ввели pre-commit hook, якого раніше не було. Це доповнення «How to apply» секції релевантного принципу, без зміни самого правила:

```diff
  ## Principle 1: Test-First
  ...
  **How to apply**:
+ - Pre-commit hook runs `pytest -m 'fast'` automatically.
  - CI runs full pytest suite on PRs.
```

Це PATCH bump.

### Сценарій C — Принцип виявився надмірним

Наприклад, поставили «100% coverage» — за квартал зрозуміли, що це жере час, реальної цінності немає, треба «75% на нових модулях». Це MAJOR bump, бо змінюється поведінка MUST.

RFC з конкретним аналізом:

> За Q1 2026 ми витратили орієнтовно 18 інженеро-годин на coverage gap, які жоден реальний bug ніколи не торкнувся. Пропонуємо зменшити до 75% на нових модулях; legacy залишається на поточному рівні.

Без RFC — не міняйте. Інакше прецедент «принципи можна тихо ослабити» руйнує дію конституції на всіх рівнях.

---

## Реальний приклад еволюції за 12 місяців

Щоб ви бачили, як конституція виглядає у часі:

### v1.0.0 (start, місяць 1)

```
6 принципів: Test-First (NON-NEGOTIABLE), Type Safety (MUST),
API-First (MUST), Observability (MUST), Performance (SHOULD),
Simplicity (SHOULD).
Ratification: 2026-04-28.
```

### v1.0.1 (тиждень 3)

PATCH — fix typo, додали приклад у Test-First секції.

### v1.1.0 (місяць 2)

MINOR — додали Principle 7: Performance perf-budget reflection (SHOULD), бо за пів спрінта виявили patterns по slow endpoints. Прийшов з RFC-002.

### v1.1.1 (місяць 3)

PATCH — у Observability секції замінили generic «structured logs» на конкретний «structlog with `request_id`», після того як на ретро виявили, що різні модулі логують у різних форматах.

### v2.0.0 (місяць 6)

MAJOR — Test-First з NON-NEGOTIABLE на MUST (це послаблення), replacing страшніший NON-NEGOTIABLE blocking gate. RFC-007 з аналізом: «3/4 виключень з Test-First у Q2 були обґрунтовані, не deserved blocker». Дозволяємо exceptions через Complexity Tracking.

### v2.1.0 (місяць 9)

MINOR — додали Principle 8: Security Review (MUST) як реакцію на security incident.

### v2.2.0 (місяць 11)

MINOR — Principle 5 (Performance Budget) з SHOULD на MUST. RFC-012: за квартал було 3 регресії > 25%, які SHOULD не заблокував.

### v2.2.1 (місяць 12)

PATCH — оновили Performance budget thresholds на основі реальних production метрик (з Grafana).

**Через 12 місяців конституція має 8 принципів, версія 2.2.x, історія amendments — у git log і у `docs/rfc/`. Це жива конституція.**

---

## Найчастіші помилки

### Маркетингові формулювання

❌ «We value engineering excellence», «Quality is everyone's responsibility».

Викидайте, замініть конкретними правилами. Якщо ви не можете описати, як принцип fail'ить Constitution Check, він зайвий.

### Перебір MUST

❌ Все MUST → нічого не MUST. Реальна тарантас на ремонті: одного спрінта команда не дотримується 4 з 8 MUST, всі сказали «давайте обговоримо потім», потім — нічого не міняється.

✅ Тримайте 4–6 справжніх MUST. Решта — SHOULD.

### Constitution як waterfall

❌ Пишемо раз і не оновлюємо. Через рік документ виглядає ніби з іншого проекту.

✅ Quarterly review + Sync Impact Report при кожній зміні.

### Constitution by Tech Lead alone

❌ Без участі команди — інші не бачать процес, не відчувають authorship, не дотримуються.

✅ Drafting workshop з участю 3–5 людей, public PR на ratification.

### Constitution без rationale

❌ Принципи без обґрунтування. Через 6 місяців нікому не зрозуміло *чому*, починається cargo cult.

✅ Кожен принцип з конкретним фактом-rationale, бажано із інцидентом або числовою метрикою.

### Constitution Check toothless

❌ У `/speckit.plan` шаблон не реально перевіряє нічого.

✅ Перевірте `.specify/templates/plan-template.md` — секція Constitution Check має бути таблицею з вашими принципами, не текстом «check whatever applies».

### Принципи занадто специфічні

❌ «MUST use SQLAlchemy 2.0 with async session». Це не принцип, це деталь з `plan.md`.

✅ Принцип — «MUST use async ORM з explicit transaction boundaries».

### Constitution не синхронізована з шаблонами

❌ Sync Impact Report ігнорують, шаблони показують старі формулювання, у `plan.md` Constitution Check посилається на принцип v1.0, хоча конституція вже v1.3.

✅ При кожному `/speckit.constitution` обов'язково перевірити, чи треба оновити шаблони — Sync Impact Report підкаже які.

---

## Чек-лист перед ratification v1.0.0

- [ ] 4–6 справжніх MUST, решта SHOULD/MAY
- [ ] Кожен принцип має rule + rationale + how to apply
- [ ] Rationale містить конкретний факт (інцидент, число, реальний приклад)
- [ ] Принципи технологічно агностичні (не згадують FastAPI, React, etc.)
- [ ] Жодних `[ALL_CAPS]` placeholder-ів
- [ ] Frontmatter містить version, ratification_date, last_amended_date
- [ ] Sync Impact Report згенерований
- [ ] Шаблони (`spec-template.md`, `plan-template.md`, `tasks-template.md`) оновлені під нові принципи
- [ ] Governance секція описує amendment процес
- [ ] PR з ratification approve-нув staff engineer
- [ ] Команда прочитала і не має блокуючих заперечень

---

## Що читати далі

- **`SPEC-KIT-docs.md`** — секція 8.5 Project Memory, як constitution лягає у загальну архітектуру.
- **`SCRUM_INTEGRATION.md`** — хто пише constitution у командному контексті.
- **`SPEC_KIT_USE_CASES.md`** — Сценарій 3 (Constitution Setup).

> 🚀 **Найважливіша порада**: не намагайтеся написати «ідеальну» конституцію v1.0. Напишіть «достатньо хорошу» v1.0, ratify, і дайте їй еволюціонувати через RFC. За 6 місяців з реальним досвідом ви знатимете, *що саме* команді треба, незрівнянно краще, ніж знаєте на старті.
