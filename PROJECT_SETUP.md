# PROJECT_SETUP — Standing up the training project

> This file is written as an **agent prompt**. You can either follow the steps manually or hand the whole file off to an AI agent in a separate chat and have it run everything autonomously.

---

## Instructions for the agent

You are an autonomous setup agent. Your goal is to clone, configure, and start the `fastapi/full-stack-fastapi-template` project via Docker Compose. Run each step in order. Don't skip steps. After each step, confirm it succeeded before moving on.

If something goes wrong, **don't improvise**. Stop, print the full error, and ask what to do next.

When finished, report the result as a checklist (all ✅, or note what broke).

---

## Prerequisites (verify before starting)

```bash
# Verify all of these commands are available:
git --version          # >= 2.30
docker --version       # >= 24.0
docker compose version # >= v2.20
node --version         # >= 18 (optional — for local dev frontend)
python3 --version      # >= 3.10 (optional — for local dev backend)
```

> ⚠️ If anything is missing — STOP. Ask whether to install it yourself or have the user do it.

---

## Step 1 — Clone the project

```bash
cd ~/projects   # or another folder for projects
git clone https://github.com/fastapi/full-stack-fastapi-template.git
cd full-stack-fastapi-template
```

**Check:**
- `ls` shows `backend/`, `frontend/`, `docker-compose.yml`, `.env.example`.

---

## Step 2 — Configure the environment

```bash
cp .env.example .env
```

Open `.env` and **make sure** to replace these values (otherwise Docker won't start due to production safeguards):

```env
# Safe defaults for local dev:
SECRET_KEY=dev-secret-change-me-in-production
FIRST_SUPERUSER_PASSWORD=changethis
POSTGRES_PASSWORD=changethis
SMTP_PASSWORD=
PROJECT_NAME=Spec-Kit Training Project
```

> 💡 **Practical rule**: never commit `.env`. It's already in `.gitignore`.

**Check:**
```bash
grep -E "^SECRET_KEY|^FIRST_SUPERUSER" .env
```
Should show something other than `changethis-please-replace-with-something-random`.

---

## Step 3 — First run via Docker Compose

```bash
docker compose up -d --build
```

The first build takes 5–15 minutes (pulling Python, Node, Postgres images plus building the frontend).

**Check:**
```bash
docker compose ps
```

All services should be in `running` (or `healthy`) state:
- `db` — PostgreSQL
- `backend` — FastAPI
- `frontend` — React + Nginx
- `proxy` — Traefik
- `mailcatcher` — for test email
- `adminer` — DB UI (optional)

If anything is failing — `docker compose logs <service>` → STOP, ask.

---

## Step 4 — Run migrations and create the superuser

Migrations usually run automatically in the `backend` container at startup. Verify:

```bash
docker compose logs backend | grep -i "migration\|alembic"
```

If migrations didn't apply, run them manually:

```bash
docker compose exec backend alembic upgrade head
```

The superuser is created automatically from `.env` (the `FIRST_SUPERUSER` and `FIRST_SUPERUSER_PASSWORD` fields).

**Check via cURL:**

```bash
curl -X POST http://localhost:8000/api/v1/login/access-token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=admin@example.com&password=changethis"
```

It should return JSON with an `access_token`. If you get a 401, the superuser wasn't created — check the logs.

---

## Step 5 — Verify the UI

Open in turn:

- ✅ http://localhost:5173 — frontend (login form)
- ✅ http://localhost:8000/docs — Swagger UI
- ✅ http://localhost:8000/redoc — ReDoc
- ✅ http://localhost:8080 — Adminer (if it's in compose)
- ✅ http://localhost:1080 — Mailcatcher

Frontend login: `admin@example.com` / `changethis`.

**Full-cycle check:**
1. Log in.
2. Create an item with the title `Hello SDD`.
3. See it in the list.
4. Delete it.

---

## Step 6 — Install spec-kit into the project

> 💡 **This activates spec-kit for the AI agent (Claude Code / Copilot / Cursor) in this repository.**

```bash
# If uv isn't installed yet:
pip install uv  # or brew install uv

# Initialize spec-kit in the current folder:
cd full-stack-fastapi-template
uvx --from git+https://github.com/github/spec-kit.git specify init . --integration claude
```

Replace `claude` with your agent: `copilot`, `cursor-agent`, `gemini`, `codex`, etc. Run `specify integration list` for the list.

**Check:**

```bash
ls -la .specify/
ls -la .specify/templates/
ls -la .specify/memory/
cat CLAUDE.md  # or GEMINI.md, AGENTS.md depending on the agent
```

You should see:
- `.specify/memory/constitution.md` (empty template)
- `.specify/templates/spec-template.md`, `plan-template.md`, `tasks-template.md`
- `.specify/scripts/bash/` and `powershell/`
- An agent context file (`CLAUDE.md`, `AGENTS.md`, `.github/copilot-instructions.md`, etc.)

> ⚠️ Spec-kit does NOT conflict with existing `git`. Initialization just adds files — it doesn't touch existing code.

---

## Step 7 — Verify slash commands in the editor

Open the project in your AI editor (Claude Code / Copilot / Cursor). In the chat, start typing `/speckit.` — the autocomplete should show:

- `/speckit.constitution`
- `/speckit.specify`
- `/speckit.clarify`
- `/speckit.plan`
- `/speckit.tasks`
- `/speckit.analyze`
- `/speckit.implement`
- `/speckit.checklist`
- `/speckit.taskstoissues`

> 💡 In **Codex CLI skills mode** the prefix is different: `$speckit-*` instead of `/speckit.*`.

**If the commands aren't there** — STOP. Ask. The most likely cause: the wrong agent was selected during `specify init`.

---

## Readiness report

Fill in this checklist:

- [ ] Git clone done
- [ ] `.env` configured
- [ ] `docker compose up -d` ran without errors
- [ ] All containers `running`/`healthy`
- [ ] Migrations applied
- [ ] cURL login returns an `access_token`
- [ ] Frontend loads at http://localhost:5173
- [ ] Swagger loads at http://localhost:8000/docs
- [ ] Created a test item via the UI
- [ ] `specify init . --integration <agent>` succeeded
- [ ] `.specify/` directory created
- [ ] Editor autocomplete shows `/speckit.*` commands

If everything is ✅ — write a short message: **"Done, the project and spec-kit are working. Ready to move to Stage 2."**

---

## Cleanup (when the course is finished)

```bash
docker compose down -v  # -v removes volumes (the database)
cd ..
rm -rf full-stack-fastapi-template
```
