# DX1 · Local Setup Guide

**Project:** AnjiSchedulo  
**Version:** 1.0.0  
**Last Updated:** 2026-06-28

---

## 1. Prerequisites

Install the following before cloning the repo:

| Tool | Version | Install |
|---|---|---|
| Java 25 JDK | 25 | `brew install --cask temurin@25` (Adoptium) or [https://adoptium.net](https://adoptium.net) |
| Docker Desktop | 4.x+ | https://www.docker.com/products/docker-desktop |
| AWS CLI | 2.x | `brew install awscli` |
| Flyway CLI | 10.x | `brew install flyway` (optional — also runs via Gradle) |
| Git | 2.x | `brew install git` |

Note: the Gradle wrapper (`./gradlew`) is checked in to the repo — no separate Gradle installation is needed.

Optional but recommended:

| Tool | Purpose |
|---|---|
| `jenv` | Java version manager |
| `direnv` | Auto-load `.envrc` per project |
| TablePlus or DBeaver | PostgreSQL GUI |
| RedisInsight | Redis key browser |

---

## 2. Clone and Install

```bash
git clone https://github.com/anji-schedulo/anji-schedulo.git
cd anji-schedulo
./gradlew build      # compiles all modules and runs unit tests
```

The repo is a Gradle 8 multi-module monorepo. All `./gradlew` commands run from the root.

---

## 3. Start Local Infrastructure

Docker Compose starts all backing services — PostgreSQL, Redis, LocalStack (SNS/SQS), and the Grafana LGTM stack.

```bash
docker compose up -d
```

This starts:

| Container | Port | Purpose |
|---|---|---|
| `postgres` | 5432 | PostgreSQL 15 |
| `redis` | 6379 | Redis 7 |
| `localstack` | 4566 | SNS + SQS (AWS-compatible) |
| `loki` | 3100 | Log store |
| `grafana` | 3000 | Dashboards (http://localhost:3000) |
| `tempo` | 4317 (gRPC) | Trace store |
| `mimir` | 9009 | Metrics store |

Wait for all containers to be healthy:

```bash
docker compose ps
```

---

## 4. Environment Variables

Copy the example env file:

```bash
cp .env.example .env.local
```

`.env.local` is git-ignored. Never commit it. Default values for local development:

```bash
# Database
DB_HOST=localhost
DB_PORT=5432
DB_NAME=anjischedulo
DB_USER=booking_app_user
DB_PASSWORD=local_dev_password

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_AUTH_TOKEN=local_dev_token

# JWT (local dev — use the pre-generated dev keys from .env.example)
JWT_PRIVATE_KEY="-----BEGIN RSA PRIVATE KEY-----\n..."
JWT_PUBLIC_KEY="-----BEGIN PUBLIC KEY-----\n..."

# AWS / LocalStack
AWS_ENDPOINT_URL=http://localhost:4566
AWS_REGION=eu-west-1
AWS_ACCESS_KEY_ID=test
AWS_SECRET_ACCESS_KEY=test

# Stripe (use test mode keys)
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_test_...

# SendGrid (sandbox mode — no real emails sent)
SENDGRID_API_KEY=SG.test...

# Observability
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
LOKI_URL=http://localhost:3100

# Service
APP_ENV=development
SERVICE_NAME=api-gateway
```

**Never use real Stripe live keys, real SendGrid keys, or real Twilio credentials locally.** Use test-mode credentials only.

---

## 5. Run Database Migrations

```bash
# Run Flyway migrations against local PostgreSQL
./gradlew flywayMigrate

# Or using Flyway CLI directly
flyway -url=jdbc:postgresql://localhost:5432/anjischedulo \
       -user=flyway_user \
       -password=flyway_local_password \
       migrate
```

Verify migrations ran:

```bash
./gradlew flywayInfo
# All migrations should show 'Success'
```

---

## 6. Seed Local Data

```bash
./gradlew dbSeed
```

This creates:
- 2 demo tenants (Starter plan: `demo-salon`, Pro plan: `demo-clinic`)
- 3 staff members per tenant
- 10 sample appointments per tenant (mix of statuses)
- 1 platform operator account

Login credentials are printed to the console after seeding.

---

## 7. Start Services

### Option A — Start all services

```bash
./gradlew bootRunAll
```

Starts all 10 Spring Boot services in parallel using Gradle parallel execution. Ports:

| Service | Port |
|---|---|
| `api-gateway` | 4000 |
| `booking-command-service` | 4001 |
| `availability-service` | 4002 |
| `payment-service` | 4003 |
| `tenant-service` | 4004 |
| `notification-service` | 4005 |
| `dashboard-service` | 4006 |
| `analytics-service` | 4007 |
| `ops-service` | 4008 |
| `outbox-relay` | 4009 (internal, no HTTP) |

### Option B — Start a specific service

```bash
./gradlew :services:booking-command-service:bootRun
```

### Option C — Start the frontend only

```bash
cd web && npm run dev
# Next.js app at http://localhost:3001
```

---

## 8. Verify Everything Works

```bash
# Health check all services
curl http://localhost:4000/health

# Create a test tenant (use seed credentials)
curl -X POST http://localhost:4000/auth/token \
  -H 'Content-Type: application/json' \
  -d '{"email":"admin@demo-salon.test","password":"DemoPassword1!"}'

# Check slot availability
curl http://localhost:4000/tenants/demo-salon/availability?date=2026-07-01&staffId=...
```

Open Grafana at http://localhost:3000 (admin / admin) — you should see the Platform Overview dashboard.

---

## 9. Running Tests

```bash
# Unit tests (all services)
./gradlew test

# Unit tests (specific service)
./gradlew :services:booking-command-service:test

# Integration tests (Testcontainers starts containers automatically)
./gradlew integrationTest

# All tests + coverage report (JaCoCo)
./gradlew test integrationTest jacocoTestReport
```

Integration tests spin up a clean PostgreSQL database inside Docker Compose for each test run. They do not use the main dev database.

---

## 10. Common Issues

| Problem | Fix |
|---|---|
| `docker compose up` fails on port 5432 | Stop local PostgreSQL: `brew services stop postgresql` |
| `./gradlew bootRun` fails with "No main class found" | Verify `spring.main.class` is set in the service's `build.gradle.kts` |
| `./gradlew flywayMigrate` fails | Check PostgreSQL is running: `docker compose ps postgres` |
| LocalStack SNS/SQS not reachable | Check LocalStack container logs: `docker compose logs localstack` |
| JWT validation fails locally | Verify `JWT_PUBLIC_KEY` matches `JWT_PRIVATE_KEY` in `.env.local` |
| Redis connection refused | Verify `REDIS_AUTH_TOKEN` matches what LocalStack or Redis container expects |

---

## 11. Stopping the Environment

```bash
# Stop all services (Ctrl+C in the ./gradlew bootRunAll terminal)

# Stop Docker Compose (keeps data)
docker compose stop

# Stop and remove all containers and volumes (clean slate)
docker compose down -v
```
