# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the Application

**With Docker Compose (recommended for local dev):**
```bash
docker compose up --build
```
App runs at http://localhost:8080. PostgreSQL data is persisted in `.docker/postgres/`.

**Locally (requires a running PostgreSQL):**
```bash
cd src
npm install
npm start
```

**Build Docker image:**
```bash
docker build -t davicarneiro/imersao-kube-news:v1 .
```

## Environment Variables

| Variable | Default | Description |
|---|---|---|
| `DB_DATABASE` | `kubedevnews` | PostgreSQL database name |
| `DB_USERNAME` | `kubedevnews` | PostgreSQL user |
| `DB_PASSWORD` | `Pg#123` | PostgreSQL password |
| `DB_HOST` | `localhost` | PostgreSQL host |
| `DB_PORT` | `5432` | PostgreSQL port |
| `DB_SSL_REQUIRE` | `false` | Enable SSL (string `"true"`/`"false"`) |

## Architecture

Single-file Express app (`src/server.js`) with no router abstraction — all routes are defined directly on the app object. The app starts on port 8080 and calls `models.initDatabase()` at startup, which runs `sequelize.sync({ alter: true })` — schema migrations happen automatically on boot.

**Key files:**
- `src/server.js` — all HTTP routes (GET/POST `/`, `/post`, `/post/:id`, `/api/post`)
- `src/models/post.js` — Sequelize model + DB connection; reads env vars here
- `src/system-life.js` — health/readiness endpoints and the `healthMid` middleware that returns 500 for all requests when the app is set "unhealthy"
- `src/middleware.js` — request counter middleware
- `src/views/` — EJS templates; `partial/` contains shared header and footer

**Data flow:** request → `healthMid` → Prometheus metrics middleware → routes → Sequelize → PostgreSQL.

**No test suite.** `npm test` exits with error by design.

## Kubernetes Manifests

`k8s-bo/` contains manifests for a basic (bare-bones) deployment:
- `postgres-secret.yml` — base64-encoded DB credentials referenced by the app deployment
- `postgres-pvc.yml` + `postgres-deployment.yml` + `postgres-service.yml` — stateful PostgreSQL
- `app-deployment.yml` + `app-service.yml` — app deployment with liveness (`/health`) and readiness (`/ready`) probes configured

Apply order: secret → PVC → postgres deployment → postgres service → app deployment → app service.

## Chaos / Health Simulation Endpoints

Used to test Kubernetes probe behavior:
- `PUT /unhealth` — makes all subsequent requests return 500 (in-memory flag, resets on restart)
- `PUT /unreadyfor/:seconds` — makes `/ready` return 500 for N seconds

## Seed Data

Use `popula-dados.http` with the VS Code REST Client extension or curl to POST sample articles to `POST /api/post`.
