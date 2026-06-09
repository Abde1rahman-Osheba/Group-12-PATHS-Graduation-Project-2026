# PATHS — Project walkthrough

End-to-end picture of **Graduation_PATHS**: backend (FastAPI), frontend (Next.js), data stores, and how major features connect.

---

## 1. What this repository is

- **Backend** (`backend/`): REST API for hiring workflows — auth, candidates, jobs, CV ingestion, scoring, org matching, **interview intelligence**, **decision support (DSS)**, health/admin.
- **Frontend** (`frontend/`): Next.js App Router UI that calls the API with JWT where required.
- **Blueprint** (see `PATHS_Deep_Architecture_Blueprint.md` and `PATHS_Implementation_Checklist.md`): longer-term full product spec; this repo implements a **graduation-scale subset** plus recent **Interview + DSS** modules.

---

## 2. Tech stack

| Layer | Technology |
|--------|------------|
| API | FastAPI, Pydantic, SQLAlchemy, Alembic |
| Auth | JWT (Bearer), passlib/bcrypt |
| DB | PostgreSQL (+ Apache AGE extension for graph, where enabled) |
| Vectors | Qdrant |
| LLM / agents | LangGraph, LangChain, Ollama or OpenRouter (per feature flags) |
| Frontend | Next.js (App Router), TypeScript, Tailwind, Framer Motion |
| Infra (optional) | Docker Compose (`backend/docker-compose.yml`) |

---

## 3. Run locally (typical)

1. **PostgreSQL (+ optional Qdrant, Ollama)** via Docker or local install.  
2. **Env**: copy `backend/.env.example` → `backend/.env`, set `DATABASE_URL`, secrets.  
3. **Migrations**: `cd backend && alembic upgrade head`.  
4. **API**: `python -m uvicorn app.main:app --host 127.0.0.1 --port 8000` (or **8001** if port 8000 is blocked on Windows).  
5. **Frontend**: `cd frontend`, copy `.env.local.example` → `.env.local`, set `NEXT_PUBLIC_API_BASE_URL` to match the API (e.g. `http://127.0.0.1:8001`).  
6. **Next**: `npm install && npm run dev` → open the URL shown (often `http://localhost:3000`).

**CORS**: backend `CORS_ORIGINS` must include your UI origin (e.g. `http://localhost:3000`, `http://localhost:3001`).  

**If the UI shows “Failed to fetch”**: API not running, wrong base URL, or DB unreachable.

---

## 4. Backend API map (high level)

Routers are mounted under **`/api/v1/`** (see `backend/app/main.py`).

| Prefix | Purpose |
|--------|---------|
| `/auth` | Register candidate/org, `login`, `me` |
| `/organizations` | Members; **`GET /{org_id}/jobs`** (list jobs for org) |
| `/cv-ingestion` | CV upload, ingestion job status |
| `/candidates` | Candidate profile JSON |
| `/scoring` | Candidate–job scores (LLM + vector) |
| `/organization-matching` | DB/CSV search, shortlist, outreach-related |
| **`/interviews`** | Availability, schedule, questions, transcript, analyze, summary, decision packet, HR human decision, candidate meeting link |
| **`/decision-support`** | Generate packet, latest by application, HR decision, dev plan, emails, compliance |
| `/job-ingestion`, `/admin`, `/job-import`, `/health`, `/system` | Ingestion, sync, health, bootstrap |

Interactive docs: **`http://<host>:<port>/docs`**.

Feature flags (env / `Settings`): e.g. `interview_intelligence_enabled`, `decision_support_enabled`, `scoring_service_enabled`, `org_matching_enabled`.

---

## 5. Frontend routes (console)

| Route | Who | What |
|-------|-----|------|
| `/` | All | Landing, links to main flows |
| `/login`, `/register/candidate`, `/register/org` | Public | Auth |
| `/candidate` | Candidate | CV upload, scoring (uses `candidate_profile.id` from `/me`) |
| `/org` | Org user | Org name, jobs list, links to tools |
| `/org/matching` | Org | Database search → shortlist run |
| `/org/runs/[runId]` | Org | Run + anonymised shortlist JSON |
| **`/org/interviews`** | Org | Sample calls: availability slots, fetch interview summary by ID |
| **`/org/decision-support`** | Org | Generate DSS packet (UUIDs), fetch latest packet for an application |

All `apiFetch` calls use `NEXT_PUBLIC_API_BASE_URL`. Organisation pages require **JWT** and **`organization_member`** with a valid org in `/auth/me`.

---

## 6. Data & auth flow (simplified)

1. **Register** → user row (+ candidate profile or org + membership).  
2. **Login** → JWT with `sub` (email), `account_type`, optional `organization_id` / `role_code`.  
3. **Protected routes** → `Authorization: Bearer <token>`.  
4. **Org-scoped APIs** (interviews, DSS, org jobs) → server checks membership and often **role** (`require_org_hr`). Role codes align with **`org_admin`**, **`recruiter`**, **`hr`**, **`hiring_manager`**, etc.

---

## 7. Recent repo updates (interviews + DSS)

- **Commit message** on `main`: *“Add Interview Intelligence & Decision Support System features”*.  
- **Backend**: new modules under `app/services/interview/`, `app/services/decision_support/`, models/schemas, routers `interviews.py`, `decision_support.py`.  
- **Frontend** (this walkthrough’s update): pages **`/org/interviews`** and **`/org/decision-support`**, nav + home tiles, org dashboard links.  
- **Compatibility fix**: `require_org_hr` default allowed roles now include **`org_admin`** and other org role codes used at registration, so first admin users are not rejected from interview/DSS endpoints.

---

## 8. Known constraints

- Many flows need **real UUIDs** (applications, interviews) from your DB — the new org pages use **manual UUID fields** for power users; a full product would add pickers/lists from additional list APIs.  
- **README in `backend/`** may list outdated “no auth” notes; trust **`/docs`** and `auth` router for current behaviour.  
- **Windows**: port **8000** is sometimes blocked; use **8001** and update `NEXT_PUBLIC_API_BASE_URL`.

---

## 9. Manual requirements — what to set for a correct run

Copy templates, then adjust for **how** you run services (Docker vs all on localhost).

### Must do (otherwise auth/DB/API breaks)

| Action | Why |
|--------|-----|
| **`backend/.env`** from **`backend/.env.example`** | All API settings read from here (or real env vars). |
| **`DATABASE_URL`** | Must point at a **running PostgreSQL** with a DB the user can use. If the API runs **on your PC** (not inside Docker), use **`@localhost:5432`**, not `@postgres:5432` (the hostname `postgres` only works from other Docker containers on the same compose network). |
| **`alembic upgrade head`** (from `backend/`) | Creates/updates tables; without it, login/register fails. |
| **`pip install -r requirements.txt`** (Python venv recommended) | Backend dependencies. |
| **`SECRET_KEY`** | Change from the placeholder for anything beyond local throwaway; JWT signing depends on it. |
| **Frontend: `frontend/.env.local`** from **`.env.local.example`** | Set **`NEXT_PUBLIC_API_BASE_URL`** to the **same host:port** as Uvicorn (e.g. `http://127.0.0.1:8001` if you use port 8001). Restart **`npm run dev`** after changes. |
| **`npm install`** in `frontend/` | Frontend dependencies. |

### Must run (processes / containers)

| Service | Required for |
|---------|----------------|
| **PostgreSQL** | Auth, all relational data. |
| **Qdrant** (default `http://localhost:6333`) | CV chunks, job/candidate vectors, scoring similarity — many flows degrade or error if down. |
| **Ollama** (default `http://localhost:11434`) with pulled models | Local LLM/embedding for CV ingestion and some pipelines (see `backend/README` model names). |

Start the stack: **`cd backend && docker compose up -d postgres qdrant ollama`** (Docker Desktop must be running). For **AGE** (graph), follow **`backend/README.md`** / `scripts/init_age.sql` once per database.

### Optional but you enter them when you need the feature

| Variable / setup | If empty |
|------------------|----------|
| **`OPENROUTER_API_KEY`** | Scoring/DSS can use **`SCORING_ALLOW_OFFLINE_FALLBACK=true`** (default) for deterministic scoring; full LLM scoring and best DSS need a key. |
| **`CORS_ORIGINS`** | Add your exact UI origin if not using defaults (e.g. new port or `127.0.0.1` vs `localhost`). |
| **SMTP** (`SMTP_*`, `OUTREACH_FROM_EMAIL`) | Real email send for outreach; otherwise those paths may no-op or error. |
| **Google calendar fields** / **`GOOGLE_*`** | Optional calendar/Meet for interview scheduling. |
| **`JOB_SCRAPER_ENABLED=true`**| Needs Playwright + extra deps per `backend/README` — off by default. |

### No manual file needed for

- **Feature toggles** like `INTERVIEW_INTELLIGENCE_ENABLED`, `DECISION_SUPPORT_ENABLED` — defaults are in config; only change if you want to disable modules.

---

## 10. Where to read more

- `PATHS_Implementation_Checklist.md` — gap vs deep architecture blueprint.  
- `backend/README.md` — ingestion, scoring, job scraper (longer).  
- `frontend/README.md` — Next.js run/build.  

---

*Generated to match the repository layout as of the Interview + DSS integration and frontend update.*
