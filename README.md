# Project 5 — Multi-Container CI/CD Pipeline: Flask + PostgreSQL on EC2

Automated deployment of a multi-container Python API and PostgreSQL database to AWS EC2, with a full CI/CD pipeline that tests, builds, and deploys on every push.

---

## The Problem This Solves

Most containerization tutorials stop at a single container. Real production systems are never a single container — they have a web server, a database, a cache, and they all need to talk to each other, start in the right order, and be configured differently across environments. This project solves that: a multi-container system where two containers communicate across a Docker network, credentials are managed per-environment without hardcoding, and the entire system deploys automatically when tests pass.

---

## What Was Built

- **Python Flask API** containerized with Docker, served by Gunicorn
- **PostgreSQL 15** database running as a separate container
- **Docker Compose** declaratively managing both containers, the shared network, and the persistent data volume
- **Docker networking** with DNS resolution — Flask connects to Postgres by container name, not IP address
- **Environment-based configuration** — `.env` for local development, GitHub Secrets for CI and production, no credentials hardcoded anywhere
- **GitHub Actions CI pipeline** — runs pytest against a live Postgres service container on every push
- **GitHub Actions CD pipeline** — SSHes into EC2, pulls latest code, rebuilds and restarts the system, runs a health check
- **Startup retry logic** — Flask retries the database connection up to 5 times, tolerating Postgres initialization delay
- **Persistent storage** — Postgres data survives container restarts via a named Docker volume

---

## Architecture

```
┌─────────────────────────────────────────────┐
│                   EC2 Instance              │
│                                             │
│   ┌─────────────────────────────────────┐   │
│   │        Docker Network               │   │
│   │                                     │   │
│   │   ┌──────────┐    ┌─────────────┐   │   │
│   │   │  Flask   │───▶│  PostgreSQL  │  │   │
│   │   │ :5000    │    │   :5432      │  │   │
│   │   └──────────┘    └─────────────┘   │   │
│   │        │                  │         │   │
│   │        │          ┌───────────────┐ │   │
│   │        │          │ postgres_data │ │   │
│   │        │          │    (volume)   │ │   │
│   │        │          └───────────────┘ │   │
│   └────────┼────────────────────────────┘   │
│            │ :5001                          │
└────────────┼────────────────────────────────┘
             │
         Browser / Client
```

Flask finds Postgres by the DNS name `db` — Docker's internal DNS resolves container names to IPs automatically. No hardcoded addresses.

---

## Pipeline Structure

```
push to main
     │
     ▼
┌─────────┐
│  test   │  Spins up Postgres service container on the runner.
│         │  Sets DB credentials from GitHub Secrets.
│         │  Runs pytest with mocked database calls.
│         │  Fails fast — deploy never runs if tests fail.
└────┬────┘
     │ needs: test
     ▼
┌─────────┐
│ deploy  │  SSHes into EC2 via deploy key.
│         │  Pulls latest code with git pull.
│         │  Runs docker-compose down then up --build.
│         │  Waits, then curls the health endpoint.
│         │  Pipeline fails if health check fails.
└─────────┘
```

**Why two jobs, not one?** Testing happens on a fresh GitHub runner. Deployment happens on the persistent EC2 server. They need different environments, different credentials, different tools. Separating them makes each job's responsibility explicit and failures easier to diagnose.

---

## Key Technical Decisions

**1 — Container name as hostname, not IP address**
Flask connects to Postgres at host `db` — the container name — not a hardcoded IP. Docker's internal DNS resolves this automatically. IPs change when containers restart; names don't. This is how production systems handle container-to-container communication.

**2 — Retry logic instead of depending on startup order**
`depends_on` in Docker Compose only guarantees Postgres starts before Flask — not that it's ready to accept connections. On first boot, Postgres takes several seconds to initialize. Rather than adding arbitrary sleep time, Flask retries the connection up to 5 times with a 3-second delay. This is the correct pattern for any distributed system: design for temporary unavailability, not just correct ordering.

**3 — Environment variables as the single configuration mechanism across all environments**
The same codebase runs locally (reading from `.env`), in CI (reading from GitHub Secrets injected as env vars), and in production (reading from `.env` on EC2). No environment-specific code branches, no config files that differ per environment. The app reads from the environment; where those values come from is the infrastructure's problem, not the application's.

---

## Local Setup

```bash
git clone https://github.com/Vimukthi-Randunu/ci-cd-project-5.git
cd ci-cd-project-5

# Create .env with local credentials
echo "POSTGRES_USER=flask_user
POSTGRES_PASSWORD=localpassword
POSTGRES_DB=flask_db" > .env

# Start the full system
docker compose up
```

API available at `http://localhost:5001`

---

## Part of a Progressive CI/CD Learning Series

This is Project 5 in a series building toward production-grade CI/CD systems:

- **Project 1** — GitHub Actions CI with Jest tests
- **Project 2** — Full CD to EC2 with health checks and rollback
- **Project 3** — Language-agnostic CI/CD with Python Flask
- **Project 4** — Containerization with Docker, image tagging with Git SHA
- **Project 5** — Multi-container systems, Docker networking, Compose orchestration ← this project
