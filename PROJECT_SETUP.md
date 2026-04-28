# PROJECT_SETUP — Розгортання тренувального проекту

> Цей файл написано як **агентський промпт**. Ви можете або виконувати кроки вручну, або повністю передати цей файл AI-агенту в окремому чаті, попросивши його виконати все автономно.

---

## Інструкції для агента

Ти — автономний setup-агент. Твоя ціль — склонувати, налаштувати і запустити проект `fastapi/full-stack-fastapi-template` через Docker Compose. Виконуй кожен крок по порядку. Не пропускай кроків. Після кожного кроку — переконайся, що він пройшов успішно, перш ніж рухатись далі.

Якщо щось пішло не так — **не імпровізуй**. Зупинись, виведи повний текст помилки і запитай, що робити.

Після завершення — повідом результат у форматі checklist (всі ✅ або вкажи, що зламалось).

---

## Передумови (перевір перед початком)

```bash
# Перевір, що всі ці команди доступні:
git --version          # >= 2.30
docker --version       # >= 24.0
docker compose version # >= v2.20
node --version         # >= 18 (опційно — для локального dev frontend)
python3 --version      # >= 3.10 (опційно — для локального dev backend)
```

> ⚠️ Якщо чогось не вистачає — STOP. Запитай, чи встановлювати самому, чи це повинен зробити користувач.

---

## Крок 1 — Клонування проекту

```bash
cd ~/projects   # або інша папка для проектів
git clone https://github.com/fastapi/full-stack-fastapi-template.git
cd full-stack-fastapi-template
```

**Перевірка:**
- Команда `ls` показує `backend/`, `frontend/`, `docker-compose.yml`, `.env.example`.

---

## Крок 2 — Налаштування environment

```bash
cp .env.example .env
```

Відкрий `.env` і **обовʼязково** заміни ці значення (інакше Docker не запуститься з production-захистами):

```env
# Безпечні дефолти для локального dev:
SECRET_KEY=dev-secret-change-me-in-production
FIRST_SUPERUSER_PASSWORD=changethis
POSTGRES_PASSWORD=changethis
SMTP_PASSWORD=
PROJECT_NAME=Spec-Kit Training Project
```

> 💡 **Практичне правило**: ніколи не комітити `.env`. Він вже в `.gitignore`.

**Перевірка:**
```bash
grep -E "^SECRET_KEY|^FIRST_SUPERUSER" .env
```
Має показати щось крім `changethis-please-replace-with-something-random`.

---

## Крок 3 — Перший запуск через Docker Compose

```bash
docker compose up -d --build
```

Перший білд — 5–15 хвилин (тягне Python, Node, Postgres-образи + збирає frontend).

**Перевірка:**
```bash
docker compose ps
```

Усі сервіси мають бути в статусі `running` (або `healthy`):
- `db` — PostgreSQL
- `backend` — FastAPI
- `frontend` — React + Nginx
- `proxy` — Traefik
- `mailcatcher` — для тестових email
- `adminer` — UI для бази (опційно)

Якщо якийсь падає — `docker compose logs <service>` → STOP, запитай.

---

## Крок 4 — Прогон міграцій і створення superuser

Міграції зазвичай прогоняються автоматично у `backend` контейнері при старті. Перевір:

```bash
docker compose logs backend | grep -i "migration\|alembic"
```

Якщо міграції не пройшли — запусти вручну:

```bash
docker compose exec backend alembic upgrade head
```

Superuser створюється автоматично з `.env` (поля `FIRST_SUPERUSER` і `FIRST_SUPERUSER_PASSWORD`).

**Перевірка через cURL:**

```bash
curl -X POST http://localhost:8000/api/v1/login/access-token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=admin@example.com&password=changethis"
```

Має повернути JSON з `access_token`. Якщо повертає 401 — superuser не створився, перевір логи.

---

## Крок 5 — Перевірка UI

Відкрий по черзі:

- ✅ http://localhost:5173 — фронтенд (логін формою)
- ✅ http://localhost:8000/docs — Swagger UI
- ✅ http://localhost:8000/redoc — ReDoc
- ✅ http://localhost:8080 — Adminer (якщо у compose)
- ✅ http://localhost:1080 — Mailcatcher

Логін на фронтенді: `admin@example.com` / `changethis`.

**Перевірка повного циклу:**
1. Залогінитись.
2. Створити item з title `Hello SDD`.
3. Побачити його в списку.
4. Видалити.

---

## Крок 6 — Встановити spec-kit у проект

> 💡 **Це робить spec-kit активним для AI-агента (Claude Code / Copilot / Cursor) у цьому репозиторії.**

```bash
# Якщо uv ще не встановлено:
pip install uv  # або brew install uv

# Ініціалізація spec-kit у поточній папці:
cd full-stack-fastapi-template
uvx --from git+https://github.com/github/spec-kit.git specify init . --integration claude
```

Заміни `claude` на свого агента: `copilot`, `cursor-agent`, `gemini`, `codex` тощо. Список — `specify integration list`.

**Перевірка:**

```bash
ls -la .specify/
ls -la .specify/templates/
ls -la .specify/memory/
cat CLAUDE.md  # або GEMINI.md, AGENTS.md залежно від агента
```

Має зʼявитись:
- `.specify/memory/constitution.md` (порожній шаблон)
- `.specify/templates/spec-template.md`, `plan-template.md`, `tasks-template.md`
- `.specify/scripts/bash/` і `powershell/`
- Файл-контекст агента (`CLAUDE.md`, `AGENTS.md`, `.github/copilot-instructions.md`...)

> ⚠️ Spec-kit НЕ конфліктує з існуючим `git`. Ініціалізація просто додає файли — не чіпає існуючий код.

---

## Крок 7 — Перевірка slash-команд у редакторі

Відкрий проект у твоєму AI-редакторі (Claude Code / Copilot / Cursor). У чаті почни друкувати `/speckit.` — має зʼявитись автокомпліт зі списку:

- `/speckit.constitution`
- `/speckit.specify`
- `/speckit.clarify`
- `/speckit.plan`
- `/speckit.tasks`
- `/speckit.analyze`
- `/speckit.implement`
- `/speckit.checklist`
- `/speckit.taskstoissues`

> 💡 У **Codex CLI skills mode** префікс інший: `$speckit-*` замість `/speckit.*`.

**Якщо команд немає** — STOP. Запитай. Найімовірніша причина: вибрано не той агент при `specify init`.

---

## Звіт про готовність

Заповни цей checklist:

- [ ] Git клон зроблено
- [ ] `.env` налаштовано
- [ ] `docker compose up -d` без помилок
- [ ] Усі контейнери в `running`/`healthy`
- [ ] Міграції пройшли
- [ ] Логін через cURL повертає `access_token`
- [ ] Фронтенд відкривається на http://localhost:5173
- [ ] Swagger відкривається на http://localhost:8000/docs
- [ ] Створено тестовий item через UI
- [ ] `specify init . --integration <agent>` пройшло
- [ ] `.specify/` директорія створена
- [ ] У редакторі автокомпліт показує `/speckit.*` команди

Якщо все ✅ — напиши коротке: **«Готово, проект і spec-kit працюють. Можна переходити до Стадії 2.»**

---

## Cleanup (коли курс завершено)

```bash
docker compose down -v  # -v видалить volumes (база)
cd ..
rm -rf full-stack-fastapi-template
```
