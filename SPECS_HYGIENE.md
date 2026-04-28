# SPECS_HYGIENE — Як тримати `specs/` у порядку на масштабі

> На дрібному проекті (10–30 фічей) flat-структура `specs/` працює без проблем. На середньому й великому (100+ фічей за рік) без активного догляду директорія перетворюється на смітник. Цей документ — про lifecycle states, archival pattern, domain hierarchy, manifest файли, quarterly grooming ritual і скрипти для автоматизації.

---

## Чому це важливо

При 200+ папках у `specs/` ви зіткнетесь із:

- `ls specs/` стає некорисним — занадто багато рядків
- `grep -r` сповільнюється і тоне у шумі
- 60–70% фіч уже задеплоєно і ніхто не повертається до їхніх specs — але вони лежать і відволікають
- Нові інженери не знають, де "живі" специфікації, а де історичні
- Specs, що були *скасовані* (фічу скасували, але папка лишилась), плутаються з активними
- Specs, що були *replaced* (нова версія фічі), співіснують зі старою — і незрозуміло, яка є source of truth

Spec-kit за дефолтом проти цього **нічого не робить**. Це питання **командної curation discipline**, а не інструменту.

---

## Lifecycle статуси

Кожен `spec.md` має у frontmatter поле `status`:

```yaml
---
status: active           # активна робота
# OR
status: implemented      # задеплоєно, але ще "свіже"
# OR
status: archived         # старе, переміщено в _archive/
# OR
status: superseded       # замінено новою версією, не використовуємо
# OR
status: dropped          # скасовано
---
```

Додаткові поля для контексту:

```yaml
---
status: implemented
implemented_date: 2026-04-15
jira_id: PROJ-1234
# або:
status: superseded
superseded_by: 042-new-search-v2
# або:
status: dropped
archived_reason: "feature scope rejected by product"
---
```

Це механічно дає вам відповідь на «що тут живе, а що мертве» через скрипти або просто `grep`.

---

## Фізичне переміщення архіву

Раз на квартал (на retro або окремим ритуалом) проходитесь по specs зі статусом `implemented` старше N місяців і фізично рухаєте їх:

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

`_archive/` починається з підкреслення, тому в `ls` він лежить зверху і явно виділяється. Можете спокійно `git mv` — git history зберігає trail, посилання у старих PR ще ведуть до commit-ів, де папка існувала за старою адресою.

> 💡 **Чому quarterly, не monthly**: квартал — достатньо довгий період, щоб «свіже implemented» стабілізувалось у production без пост-фактум змін. На monthly будете рухати, а через 2 тижні повертатись до spec.md для bug-fix-ів — це створює тертя.

---

## Видалення без жалю для truly-dead

Скасована фіча, яку ніхто не реалізує і не планує — просто `git rm specs/017-feature-we-dropped/`. У git history воно лишається назавжди:

```bash
# Знайти видалені specs:
git log --diff-filter=D --summary | grep "specs/"
```

Якщо колись треба буде підняти — `git checkout <commit>~ -- specs/017-feature/` поверне з історії.

> ⚠️ **Не бійтесь видаляти.** Страх «а раптом знадобиться» якраз і створює смітник. Git — ваш undelete; використовуйте його.

---

## Domain hierarchy для нумерації

При 200+ фічах flat-нумерація `001`...`200` стає безглуздою (хто пам'ятає, що `127` робить?). Краще:

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

Домен + локальна нумерація. Так у голові залишається «items/002-search», що краще, ніж «127-search» серед 200 інших чисел.

### Як це налаштувати у spec-kit

1. Модифікувати `.specify/scripts/bash/create-new-feature.sh` — щоб запитувати домен і нумерувати в його межах.
2. Або в `init-options.json` поставити `branch_numbering: timestamp` і номер взагалі ігнорувати, групуючи лише за доменами.
3. Або через `extensions.yml` — кастомний `before_specify` hook, що питає: «домен?» і створює папку у правильному місці.

---

## Manifest / index файл як точка входу

У корені проекту або у `specs/README.md` тримаєте автоматично згенерований індекс:

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

Регенерується скриптом, який читає frontmatter усіх specs. У pre-commit hook або CI — щоб не ставав stale.

---

## Search tooling

Простий Python-скрипт ~50 рядків, який *читає frontmatter* всіх specs і підтримує запити:

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

Використання:

```bash
$ scripts/specs.py status=active domain=items
items/008-bulk-edit  [active]  P2  PROJ-1567
items/009-import     [active]  P3  PROJ-1602

$ scripts/specs.py status=implemented sprint=23
auth/003-sso         [implemented]  P1  PROJ-1389
items/006-comments   [implemented]  P2  PROJ-1402
```

Це 30 хвилин написати, але економить години на пошуку через місяці.

> 💡 **Розширення**: додайте підкоманди — `scripts/specs.py find <text>` для full-text search по spec.md, `scripts/specs.py drift` для виявлення specs, де код був змінений без оновлення spec.md.

---

## Quarterly grooming ritual

Той момент, коли реально розкидається сміття — **квартальний `specs/` grooming**. 30 хвилин на retro раз на 3 місяці.

### Чек-лист

1. **Список усіх `status: active`**, що старші 90 днів — або *реально* активні (хтось робить), або застрягли (треба archive/drop).
2. **Список `implemented` старше 90 днів** — рухаємо у `_archive/<quarter>/`.
3. **Список `superseded`** — перевіряємо, що нова версія дійсно покриває старі вимоги, і архівуємо стару.
4. **Список skipped/dropped** — просто видаляємо.
5. **Якщо в домені понад 30 specs** — обговорюємо подальший split.

### Скрипт для підготовки до retro

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
    # ... аналогічно
done

echo ""
echo "=== Domains with >30 active specs ==="
scripts/specs.py status=active | awk -F'/' '{print $1}' | sort | uniq -c | awk '$1 > 30'
```

### Без цього ритуалу

Будь-яка структура за рік перетвориться на смітник, бо це **не проблема структури**, а проблема **відсутності curation**.

---

## Спеки на дрібниці — не пишемо

Окрема дисципліна, яка зменшує проблему *в корені*: не кожна задача потребує `specs/<feature>/`.

З `SPEC_KIT_USE_CASES.md` Сценарій 1 — Quick Spec Flow для дрібниць (баг-фікс, тривіальна зміна). Команда може вирішити правилом: «фічі менше 3 SP — без специфікації, лише code review». Це сильно зріже потік.

З Discussion #152 цитата community: «Even for small features like 'Add login' we have to use spec-kit? Wouldn't we save more time if just use copilot Plan agent for small features?» — і відповідь спільноти: ні, не треба для всього.

> 💡 Якщо ви робите спеку на кожну дрібницю — отримаєте 500 specs за рік. Якщо тільки на медіум-плюс задачі — 50–80, що цілком керовано.

---

## Реалістична картинка для команди 5–10 за 2 роки

З нормальною дисципліною:

- ~80 specs на рік (1.5 SP+ задач, дрібниці пропускаємо)
- Через 2 роки — ~160 specs, з яких:
  - ~30 active або recently shipped (живуть у `specs/<domain>/`)
  - ~110 archived у `specs/_archive/<quarter>/`
  - ~20 deleted (dropped/skipped)
- 4 квартальних gromming-сесії
- Один скрипт `specs.py` + auto-generated index

**Цілком придатне для життя.**

**Без** дисципліни — той самий проект через 2 роки матиме 320 specs у плоскому `specs/`, з яких 60% є мертвим вантажем, і нова людина перші 2 тижні буде питати «де у нас X?» бо нічого не знаходить.

---

## Принцип у Constitution для специфічного проекту

Додайте до constitution.md:

```markdown
## Principle X: Specs Hygiene (SHOULD)

**Rule**: Every spec.md has frontmatter status field. Quarterly we move
implemented specs older than 90 days into _archive/<YYYY>-Q<N>/.
Dropped/skipped specs are deleted (not kept as zombies).
Domains with > 30 active specs trigger a split discussion.

**Rationale**: Без активної curation specs/ за рік перетворюється на
смітник, де нова людина не може знайти source of truth і втрачає 30%
часу onboarding.

**How to apply**:
- Pre-commit hook валідує frontmatter (`status`, `domain` поля обов'язкові)
- Quarterly retro має 30-хвилинний slot на specs grooming
- CI перевіряє, що `specs/README.md` (auto-index) свіжий
```

Це робить проблему явним пунктом командної дисципліни, а не "ну колись поприбираємо".

---

## Антипатерни

### Спрінт-групування

```
specs/
├── sprint-12/
├── sprint-13/
└── sprint-14/
```

❌ **Не робіть так**. Спрінт — часовий контейнер, фічі переростають його. Якщо `004-favorites` стартував у спрінті 13, не закрився, переїхав у 14 — куди ставити? Дублювати? Переносити?

✅ **Правильно**: домени + frontmatter `sprint: 13` для запитів через скрипт.

### Глибока вкладеність

```
specs/
└── product-area-1/
    └── sub-feature-group/
        └── specific-feature/
            └── 001-tiny-thing/
```

❌ Складна навігація, важко знаходити, скрипти ускладнюються.

✅ Максимум 2 рівня: `specs/<domain>/<NNN-feature>/`.

### Specs-as-tickets

Папка `specs/` повторює всю структуру Jira backlog 1:1.

❌ Створює надлишок: дрібні bug-fixes у specs, які не варті повної специфікації.

✅ Specs тільки для того, що реально потребує специфікації — фічі ≥ 3 SP, складні рефакторинги, нові інтеграції.

### Не видаляти ніколи

❌ «А раптом знадобиться» → 200 dead specs у `specs/`.

✅ Quarterly видалення dropped/skipped через `git rm`. Git історія = архів на крайній випадок.

### Один великий файл

Замість `specs/<feature>/spec.md` + plan.md + ... — один файл `specs/all-features.md` з усім.

❌ Жодних `/speckit.*` команд не працюватимуть, повністю руйнує всю систему spec-kit.

✅ Дотримуйтесь стандартної структури spec-kit з окремими файлами.

---

## Підтримка актуальності специфікацій між спрінтами

Окреме від archival завдання — підтримка **поточної актуальності** specs. `specs/<feature>/spec.md` створюється як snapshot моменту створення фічі. Через 6 місяців після ship-у вона перетворюється на історичний документ, але читачі продовжують її використовувати, думаючи що це поточна реальність.

### Корінь проблеми

Specs у SDD натурально атомарні: один spec = одна зміна. Але продукт еволюціонює як **система**, а не як ланцюг ізольованих змін. Звідси розрив:

- `specs/001-user-auth/` описує auth, як його зробили в спрінті 3.
- `specs/015-add-mfa/` додав MFA в спрінті 8.
- `specs/023-passkeys/` замінив частину MFA на WebAuthn у спрінті 14.
- `specs/031-session-management/` змінив timeout-и в спрінті 19.

Питання **«як у нас зараз працює auth?»** — отримує відповідь з 4 specs, частина з яких суперечить одна одній. Ніхто з них не *бреше*, але ніхто не показує **поточну реальність**.

Це проблема, визнана у community ([Discussion #152](https://github.com/github/spec-kit/discussions/152), Jflam — один із maintainer'ів):

> «if you make a new folder/doc for every new feature… a very executional, short term mentality as opposed to having a system level design mentality.»

Нижче — 5 паттернів, які реально застосовуються командами, з нейтральним розглядом плюсів і мінусів кожного. Жоден з них не є «правильним» — вибір залежить від розміру команди, рівня дисципліни, частоти змін у домені.

---

### Паттерн A — Atomic + Frozen

Кожен spec — immutable після ship. Закрив, не торкаєшся. Для розуміння поточного стану — або читаєш всі relevant specs, або дивишся в код.

**Механіка:**
- `spec.md` отримує `status: implemented` після merge у main.
- Жодних подальших правок цього файлу — навіть якщо фіча еволюціонувала.
- Для розуміння current state читач послідовно проходить historical specs того ж домену.

**Що працює (теоретично):**
- Чітка історія: кожен spec — точний snapshot моменту прийняття рішення.
- Жодних merge-конфліктів у specs (бо їх не редагують).

**Що ламається (на практиці):**
- 6 місяців на проекті 50+ specs нова людина не може зрозуміти, як що працює.
- Питання «який поточний стан?» не має зручної відповіді.

**Плюси:**
- Мінімальні зусилля
- Immutable record — добре для compliance / audit
- Жодних суперечок про «хто має оновлювати»

**Мінуси:**
- Не масштабується — на 50+ specs пошук поточної реальності стає неможливим
- Drift не просто можливий, він **закладений** у патерн
- Onboarding нових інженерів дорогий

---

### Паттерн B — Supersession Chain

Кожен наступний spec явно помічає предка через frontmatter. Старі автоматично отримують `superseded_by: <next-spec>`.

**Механіка:**

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

Скрипт показує: «для розуміння auth читай specs/023-passkeys/ як head of chain, специфічна історія — у предків».

**Що працює (теоретично):**
- Якщо нова фіча *повністю* замінює стару — chain дає чіткий шлях до current state.
- Tooling може автоматично знаходити head of chain.

**Що ламається (на практиці):**
- Реальність — фічі рідко *повністю* замінюють попередні. 015-add-mfa частково живе разом із 023-passkeys (бекап для legacy users).
- Chain-ове відношення — лінійне, реальність — графова.
- Frontmatter `supersedes` потребує дисципліни, яка часто провисає.

**Плюси:**
- Зберігає історію та дозволяє reach current state
- Automation-friendly (чіткий формат frontmatter)

**Мінуси:**
- Не охоплює часткові заміни / additive evolution
- Дисципліна заповнення frontmatter критична
- Складно навігувати при глибоких chain-ах (5+ versions)

---

### Паттерн C — Living Domain Specs (всередині `specs/`)

Окрема папка з `domain.md` для кожного bounded context, яка завжди — поточний стан. Атомарні specs для кожної зміни оновлюють domain.md як частину DoD.

**Механіка:**

```
specs/
├── auth/
│   ├── domain.md              ← поточний стан auth
│   ├── 001-user-auth/         ← історичний (як стартували)
│   ├── 015-add-mfa/           ← історичний (як додали MFA)
│   └── 023-passkeys/          ← останній; обов'язково оновив auth/domain.md
└── items/
    ├── domain.md
    └── ...
```

Розподіл: atomic specs — заморожені delta; `domain.md` — current state, оновлюється кожним delta. DoD-правило: PR не merge-ється без оновлення domain.md.

**Що працює (теоретично):**
- Одна точка читання для current state у домені.
- Локальність: усі artifact-и домену в одній папці.

**Що ламається (на практиці):**
- Архівація стає складною: при перенесенні `specs/auth/015-add-mfa/` у `_archive/` порушується domain-локальність.
- Або не архівуєте — і `auth/` пухне до сотні папок.
- Per-PR оновлення `domain.md` створює тертя — інженер хоче ship-нути фічу, а не редагувати huge document.
- При паралельній роботі двох команд над auth — merge-конфлікти у `domain.md`.

**Плюси:**
- Domain locality — все на одному місці
- Single discovery path для нової людини

**Мінуси:**
- Конфлікт із archival стратегією
- Domain-папка швидко росте
- Per-PR DoD на оновлення domain.md — висока дисципліна
- Ризик merge-конфліктів між фічами одного домену

---

### Паттерн D — Unified Spec (Jflam pattern)

Один файл на домен `specs/auth/spec.md` живе як unified document. Усі зміни — це commits у git до того ж файлу. `## Changelog` секція внизу веде історію.

**Механіка:**

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

**Що працює (теоретично):**
- Один файл, одне місце, мінімальна когнітивна навантага.
- Git history природно зберігає еволюцію через `git log -p`.

**Що ламається (на практиці):**
- Ламає механіку spec-kit: `/speckit.specify` створює нову папку, не редагує існуючий файл.
- Доводиться вручну перенаправляти роботу у існуючий файл, що створює тертя зі стандартним workflow.
- При паралельній роботі — merge-конфлікти у єдиному файлі.
- Файл росте необмежено — через 2 роки 100+ KB на один домен.

**Плюси:**
- Мінімалістична структура
- Природна currency — current state на самому початку файлу
- Git history несе всю еволюцію

**Мінуси:**
- Несумісно зі стандартною механікою spec-kit
- Потребує custom workflow або форкання шаблонів
- Не масштабується по часу (файл росте)
- Високий ризик merge-конфліктів

---

### Паттерн E — Hybrid (architecture docs + flat specs)

Окремий файловий простір для current state (`docs/architecture/<domain>.md`) і окремий для atomic deltas (`specs/<NNN-feature>/`). Архітектурні docs — high-level (1-2 сторінки), specs — детальні delta records.

**Механіка:**

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

Specs мають у frontmatter `domain: auth`, що дозволяє знаходити їх по домену через скрипт. Architecture docs оновлюються або per-PR (через template modification, див. нижче), або per-quarter (на retro).

**Що працює (теоретично):**
- Два простори, дві швидкості: deltas — швидкі (atomic), architecture — повільні (high-level).
- Specs можуть архівуватись повністю незалежно від architecture docs.
- Architecture docs — короткі, тому drift не катастрофічний.

**Що ламається (на практиці):**
- Дві паралельні системи — ризик розходжень.
- Без процесу sync architecture docs дрейфують.
- Розподіл «що йде в spec, що в architecture» вимагає judgment-у.

**Плюси:**
- Чиста архівація specs
- Architecture docs короткі — легше підтримувати
- Не ламає механіку spec-kit
- Different update cadences можливі

**Мінуси:**
- Ризик drift між двома просторами
- Потребує процесу/ритуалу для sync
- Розподіл відповідальності між artifact-ами не очевидний

---

### Підсумкова таблиця паттернів

| Паттерн | Currency frequency | Scaling | Conflict ризик | Сумісність зі spec-kit |
|---------|-------------------|---------|----------------|------------------------|
| A. Atomic + Frozen | None (за дефолтом застаріває) | Низьке | Немає | Повна |
| B. Supersession Chain | Per-replacement | Середнє | Низький | Повна |
| C. Living Domain Specs | Per-PR | Низьке (домен пухне) | Високий (merge у domain.md) | Повна |
| D. Unified Spec | Per-commit | Низьке (файл росте) | Дуже високий | Часткова (потребує custom workflow) |
| E. Hybrid (arch docs + flat specs) | Per-quarter (або per-PR) | Високе | Низький | Повна |

---

## Механізми автоматизації sync

Незалежно від обраного паттерну, можна задіяти три рівні автоматизації для зменшення дисциплінарного навантаження.

### Рівень 1 — Модифікація шаблону команди

`.specify/templates/commands/plan.md` — це звичайний markdown-промпт. Можна додати інструкції, які агент виконує при кожному `/speckit.plan`.

**Механіка:**

В кінець `plan.md` додається секція:

```markdown
### Phase 2: Architecture Doc Update Proposal

1. Read frontmatter of active spec to identify the domain.
2. Read current docs/architecture/<domain>.md (if exists).
3. Generate proposed updates based on the new spec/plan.
4. Write proposal to specs/<active>/architecture-update.md.
5. Add "Architecture Impact" section to plan.md.

DO NOT modify the architecture doc directly — produce a proposal for review.
```

При кожному `/speckit.plan` агент сам перевіряє domain, читає architecture doc, генерує review-ready proposal.

**Що працює:**
- Проактивне виявлення архітектурних змін під час планування.
- Не потребує зовнішніх скриптів чи інфраструктури.
- Працює на будь-якому агенті (Claude, Copilot, Cursor) — це звичайний markdown.

**Що ламається:**
- При оновленні spec-kit шаблони можуть бути перезаписані. Потрібен fork стратегія або документований патч.
- AI proposals потребують human review — автоматичний merge створює помилки, які накопичуються.
- Якість proposal залежить від quality зчитування існуючого architecture doc.

**Плюси:**
- Найнижчий поріг входу
- Не вимагає extension/MCP infrastructure
- Працює native у workflow (proposal створюється разом з planом)

**Мінуси:**
- Затирається при `specify integration upgrade`
- Вимагає custom patch, документованого окремо
- Тільки proposals — не виконує реальний sync

---

### Рівень 2 — Hooks через `.specify/extensions.yml`

Hooks дозволяють виконувати shell-команди до/після команд spec-kit. Точки розширення: `before_specify`, `after_specify`, `before_plan`, `after_plan`, `before_tasks`, `after_tasks`, `before_implement`, `after_implement`.

**Механіка:**

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

Скрипт читає frontmatter активної спеки, визначає домен, додає одну строчку в `## Recent changes` секцію відповідного `architecture/<domain>.md`.

**Що працює:**
- Mechanical updates (append рядка у changelog) — надійно.
- Тригер на `after_implement` гарантує, що update відбувається тільки після успішної імплементації.

**Що ламається:**
- Sed-based текстові маніпуляції з markdown крихкі — найменша зміна структури документа ламає скрипт.
- Hooks не призначені для AI-invocations напряму. Для генерації prose потрібен Рівень 1 або 3.
- Optional / required параметри hook-ів не завжди respected — залежить від host-агента.

**Плюси:**
- Автоматична фіксація fact-of-shipping
- Працює без AI на цьому етапі (детерміновано)
- Інтегрується в native lifecycle команд spec-kit

**Мінуси:**
- Лише mechanical updates (не prose)
- Крихкість sed/awk-маніпуляцій
- Hooks механізм у spec-kit поки early-stage

---

### Рівень 3 — Кастомний slash-command

Можна додати власну команду через створення нового шаблону у `.specify/templates/commands/<name>.md`. Команда викликається явно, не автоматично.

**Механіка:**

`.specify/templates/commands/archupdate.md` із інструкціями:

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

Раз на квартал (на ретро) запускається `/speckit.archupdate` — агент сам сходить, прочитає всі shipped specs за період, порівняє з architecture docs, згенерує proposals.

**Що працює:**
- Batch-ретроспективна перевірка currency.
- Виявляє drift, який не було помічено per-PR.
- Чіткий ритуал, прив'язаний до квартального циклу.

**Що ламається:**
- Залежить від human trigger — якщо retro пропустили, sync не відбувся.
- Якість аналізу залежить від context-розміру агента (для great кількості specs може втратити деталі).
- Proposals все одно потребують human review.

**Плюси:**
- Працює як ритуал — explicit moment of sync
- Може охопити те, що не покрив Рівень 1 (cross-feature drift)
- AI-power для семантичного аналізу

**Мінуси:**
- Manual trigger — risk пропуску
- Великий context — ризик низької якості proposals
- Тільки proposals, не auto-merge

---

### Підсумкова таблиця механізмів автоматизації

| Рівень | Тригер | Тип update | Інфраструктура | AI-залежність |
|--------|--------|-----------|----------------|---------------|
| 1. Template mod | Per-`/plan` | Proposal (prose) | Markdown patch | Висока |
| 2. Hooks | Per-`/implement` (etc) | Mechanical (changelog) | extensions.yml + скрипт | Низька |
| 3. Custom command | Manual (e.g. quarterly) | Proposal (prose, batch) | Шаблон команди | Висока |

---

### Сумісність паттернів і механізмів

Не всі комбінації паттернів і механізмів мають сенс:

| | Рівень 1 (template mod) | Рівень 2 (hooks) | Рівень 3 (custom command) |
|---|------------------------|------------------|---------------------------|
| **A. Atomic + Frozen** | N/A (нема currency artifact) | N/A | N/A |
| **B. Supersession Chain** | Auto-update frontmatter | Можливо | Можливо |
| **C. Living Domain Specs** | Update domain.md proposal | Append changelog у domain.md | Quarterly review domain.md |
| **D. Unified Spec** | Append section у unified | Append changelog | Quarterly cleanup |
| **E. Hybrid (arch docs)** | Update arch doc proposal | Append changelog у arch doc | Quarterly arch review |

Паттерн A не вимагає sync механізмів за визначенням. Решта — отримують вигоду з рівнів 1-3 у різних комбінаціях.

---

## Чек-лист на масштабі

Якщо ваш проект перетинає поріг 50–80 specs:

- [ ] Усі spec.md мають frontmatter з `status` полем
- [ ] Створено `_archive/` директорію
- [ ] Перенесено specs зі статусом `implemented` старше 90 днів
- [ ] Видалено `dropped` specs (через git rm)
- [ ] Створено `specs/README.md` як auto-generated index
- [ ] Створено `scripts/specs.py` для пошуку/фільтрації
- [ ] Налаштовано quarterly grooming ritual на retro
- [ ] Додано принцип Specs Hygiene у constitution.md
- [ ] Якщо домени вже видно — створено `specs/<domain>/` структуру
- [ ] Pre-commit hook валідує frontmatter
- [ ] Команда обрала і задокументувала свій паттерн currency (A/B/C/D/E)
- [ ] Якщо паттерн A не обрано — налаштовано хоча б один механізм sync (Рівень 1/2/3)

---

## Що читати далі

- **`SCRUM_INTEGRATION.md`** — як specs hygiene інтегрується з Scrum-ритмом (quarterly grooming = частина retro).
- **`CONSTITUTION_GUIDE.md`** — як додати Specs Hygiene принцип у конституцію.
- **`SPEC-KIT-docs.md`** — секція 8.5 Project Memory.

> 🚀 **Ключова порада**: не чекайте, поки `specs/` стане смітником. Заведіть hygiene-практику з 50-ї спеки, не з 250-ї. Перенести 5 specs у `_archive/` — 2 хвилини; перенести 200 — півдня.
