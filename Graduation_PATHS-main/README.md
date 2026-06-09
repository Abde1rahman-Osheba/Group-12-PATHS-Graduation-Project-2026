# PATHS — Personalised AI Talent Hiring System

> Evidence-driven, human-in-the-loop hiring powered by LangGraph agents,
> a Next.js 16 frontend, and a FastAPI/PostgreSQL/Qdrant backend.

---

## Stack at a glance

| Layer | Technology |
|---|---|
| Frontend | Next.js 16 (App Router) · React 19 · Tailwind v4 · shadcn/ui · TanStack Query |
| Backend | FastAPI · SQLAlchemy (async) · Alembic · PostgreSQL 16 + Apache AGE (graph) |
| Vector DB | Qdrant (candidate + job embeddings) |
| Agents | LangGraph (CV ingestion, scoring, sourcing, interview, decision-support) |
| Auth | JWT (argon2id) · Stripe (billing) |
| Observability | Sentry · Prometheus · OpenTelemetry |
| CI/CD | GitHub Actions → Fly.io (backend) + Vercel (frontend) |

---

## Prerequisites

| Tool | Version |
|---|---|
| Python | ≥ 3.12 |
| Node.js | ≥ 20 |
| pnpm | ≥ 9 |
| PostgreSQL + Apache AGE | PG 16, AGE 1.5 |
| Qdrant | ≥ 1.9 |
| Docker (optional) | any recent version |

---

## 15-minute setup

### 1. Clone & enter the repo

```bash
git clone https://github.com/your-org/paths.git
cd paths
```

### 2. Install pre-commit hooks (blocks secret leaks)

```bash
pip install pre-commit
pre-commit install
```

### 3. Backend

```bash
cd backend

# Create a virtual environment
python -m venv .venv && source .venv/bin/activate   # Windows: .venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Copy and edit the environment file
cp .env.example .env
# At minimum, set:  DATABASE_URL, SECRET_KEY, QDRANT_URL

# Run database migrations
alembic upgrade head

# Start the server (port 8001)
uvicorn app.main:app --reload --port 8001
```

The API docs are at **http://localhost:8001/docs**.

### 4. Frontend

```bash
cd frontend

# Install dependencies
pnpm install

# Copy and edit the environment file
cp apps/web/.env.example apps/web/.env.local
# At minimum, set:  NEXT_PUBLIC_API_URL=http://localhost:8001

# Start the dev server (port 3000)
pnpm --filter @paths/web dev
```

Open **http://localhost:3000**.

### 5. Seed demo data (optional)

```bash
cd backend
python -m seed.demo
```

Creates 3 orgs, 9 jobs, 50 candidates, one complete hire pipeline with
bias report + growth plan — ready for a 10-minute demo.

---

## Project layout

```
paths/
├── backend/          FastAPI service
│   ├── app/
│   │   ├── api/v1/   Route handlers
│   │   ├── core/     Config, logging, security, telemetry
│   │   ├── models/   SQLAlchemy ORM models
│   │   ├── services/ Business logic + LangGraph agents
│   │   └── main.py   Entry point
│   ├── alembic/      DB migrations
│   ├── tests/        pytest suites
│   └── seed/         Demo data generator
├── frontend/
│   └── apps/web/     Next.js application
│       ├── src/app/  Route groups + pages
│       ├── src/components/
│       └── src/lib/  API client + React Query hooks
├── docs/
│   ├── plan/         Phase-by-phase implementation plan
│   ├── architecture.md
│   └── runbooks/     Operational procedures
├── scripts/
│   └── smoke.js      k6 smoke test
└── .github/
    └── workflows/ci.yml
```

---

## Key environment variables

See `backend/.env.example` and `frontend/apps/web/.env.example` for the
full list with descriptions.  The non-negotiables for production:

| Variable | Where | Description |
|---|---|---|
| `SECRET_KEY` | backend | 32+ char random string for JWT signing |
| `DATABASE_URL` | backend | PostgreSQL connection string |
| `QDRANT_URL` | backend | Qdrant server URL |
| `STRIPE_SECRET_KEY` | backend | Stripe live secret key |
| `STRIPE_WEBHOOK_SECRET` | backend | Stripe webhook signing secret |
| `OPENROUTER_API_KEY` | backend | LLM inference |
| `SENTRY_DSN` | backend | Error monitoring |
| `NEXT_PUBLIC_API_URL` | frontend | Backend base URL |
| `NEXT_PUBLIC_SENTRY_DSN` | frontend | Browser error monitoring |
| `SENTRY_AUTH_TOKEN` | frontend CI | Source map upload |

---

## Running tests

```bash
# Backend
cd backend
pytest tests/ --cov=app --cov-fail-under=60

# Frontend (when tests exist)
cd frontend
pnpm --filter @paths/web test
```

---

## Further reading

- [Phase-by-phase plan](docs/plan/00_MASTER_PLAN.md)
- [Architecture overview](docs/architecture.md)
- [Backend README](backend/README.md)
- [Frontend README](frontend/README.md)
- [Runbooks](docs/runbooks/)
- [Contributing](CONTRIBUTING.md)
