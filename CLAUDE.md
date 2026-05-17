# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Kube-news is a Node.js news/blog application designed as a container learning example. It uses Express, EJS templates, Sequelize ORM, and PostgreSQL. The app exposes Prometheus metrics and Kubernetes liveness/readiness probes.

## Running Locally

**With Docker Compose (recommended):**
```bash
docker compose up
```
App runs at http://localhost:8080. Postgres credentials are hardcoded in `compose.yaml` (user: `kubenews`, password: `pg1234`, db: `kubenews`).

**Without Docker:**
```bash
cd src
npm install
DB_DATABASE=kubedevnews DB_USERNAME=kubedevnews DB_PASSWORD=Pg#123 DB_HOST=localhost npm start
```

**Build Docker image only:**
```bash
docker build -t kube-news ./src
```

## Environment Variables

| Variable | Default (dev) | Description |
|---|---|---|
| `DB_DATABASE` | `kubedevnews` | PostgreSQL database name |
| `DB_USERNAME` | `kubedevnews` | PostgreSQL user |
| `DB_PASSWORD` | `Pg#123` | PostgreSQL password |
| `DB_HOST` | `localhost` | PostgreSQL host |

## Architecture

All application code lives in `src/`:

- **`server.js`** — Express entry point; defines all HTTP routes, wires up middleware, starts on port 8080
- **`models/post.js`** — Sequelize `Post` model (PostgreSQL); calls `seque.sync({ alter: true })` on startup to auto-migrate the schema
- **`system-life.js`** — Health/readiness probe routes (`GET /health`, `GET /ready`, `PUT /unhealth`, `PUT /unreadyfor/:seconds`) used by Kubernetes probes
- **`middleware.js`** — Prometheus `http_requests_total` counter middleware
- **`views/`** — EJS templates: `index.ejs` (list), `view-news.ejs` (single post), `edit-news.ejs` (create form), `partial/` (shared partials)

**Request flow:** `countRequests` middleware → Prometheus bundle middleware → health middleware → routes

## API Endpoints

| Method | Path | Description |
|---|---|---|
| `GET` | `/` | List all posts |
| `GET` | `/post` | New post form |
| `POST` | `/post` | Create post (form submission) |
| `POST` | `/api/post` | Bulk create posts via JSON (`{ artigos: [...] }`) |
| `GET` | `/post/:id` | View single post |
| `GET` | `/health` | Liveness probe |
| `GET` | `/ready` | Readiness probe |
| `GET` | `/metrics` | Prometheus metrics |

## Kubernetes

`k8s/deployment.yaml` deploys both PostgreSQL and the app (10 replicas) with a `LoadBalancer` service on port 80 → 8080.

## CI/CD

- **`main_app-node-live.yml`** — Active workflow: builds from `./src`, deploys to Azure App Service (`app-node-live`) on push to `main`
- **`main.yml`** — Commented-out workflow: Docker build → Trivy scan → push to DockerHub → deploy to AKS via `k8s/deployment.yaml`

## Post Validation Rules

Title: < 30 chars, Summary: < 50 chars, Content: < 2000 chars (enforced in `server.js:38-44`).
