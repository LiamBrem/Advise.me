# Contributing to Advise.me
---

## Prerequisites

Make sure you have these installed before anything else:

- [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- [Node.js](https://nodejs.org/) (v20+)
- [Python](https://www.python.org/) (3.12+)
- [Git](https://git-scm.com/)

---

## First Time Setup

### 1. Clone the repo

```bash
git clone https://github.com/LiamBrem/Advise.me
cd advise.me
```

### 2. Set up environment variables

```bash
cp .env.example .env
```

Open `.env` and fill in the values. Ask someone for any secrets you need.

### 3. Start the database and backend

```bash
docker-compose up db backend --build
```

This starts Postgres (with pgvector) and the FastAPI backend. First run will take a minute to pull images and build.

### 4. Run the frontend natively

In a separate terminal:

```bash
cd frontend
npm install
npm run dev
```

Frontend runs at `http://localhost:3000`
Backend runs at `http://localhost:8000`
Backend API docs at `http://localhost:8000/docs`

---

## Development

You don't need to rebuild every time. Normal workflow:

```bash
# Terminal 1 — start db and backend
docker-compose up db backend

# Terminal 2 — start frontend
cd frontend && npm run dev
```

---

## Docker Commands

| Command | What it does |
|---|---|
| `docker-compose up` | Start all services |
| `docker-compose up -d` | Start all services in the background |
| `docker-compose down` | Stop all services |
| `docker-compose down -v` | Stop all services and wipe the database (fresh start) |
| `docker-compose up --build` | Rebuild images and start (run after changing Dockerfile or dependencies) |
| `docker-compose logs backend` | See logs for a specific service |
| `docker-compose ps` | See what's running |

---

## Project Structure

```
pittpal/
├── frontend/        # Next.js app
├── backend/         # FastAPI app
├── docker-compose.yml
└── .env.example     # Copy this to .env and fill in values
```

---

## Backend

The backend is FastAPI with Python. If you need to install a new dependency:

```bash
cd backend
source venv/bin/activate       # Mac/Linux
# or: venv\Scripts\activate    # Windows
pip install <package>
pip freeze > requirements.txt  # commit this
```

Database migrations use Alembic:

```bash
# Create a new migration after changing a model
alembic revision --autogenerate -m "describe your change"

# Apply migrations
alembic upgrade head
```

---

## Frontend

The frontend is Next.js. If you need to install a new dependency:

```bash
cd frontend
npm install <package>   # commit the updated package.json and package-lock.json
```

---

## Environment Variables

Never commit `.env` or any file containing real secrets. The `.env.example` file contains all required variable names with empty values - keep it up to date when you add new variables.

If you accidentally commit a secret, tell the team immediately so we can rotate it.

---

## Branches and PRs

- `main` — stable, should always be working
- `dev` — active development, merge your branches here
- Feature branches: `your-name/short-description`

Keep PRs small and focused. One feature or fix per PR. Write a short description of what you changed and why.