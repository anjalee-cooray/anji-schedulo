---
title: Tech Stack
layer: 04-architecture
status: current
lastUpdated: 2026-06-28
---

# A1 · Tech Stack

## Overview

AnjiSchedulo is built as a Java 25 multi-module Gradle monorepo of Spring Boot 3.x microservices, deployed on AWS ECS Fargate and connected via an event-driven SNS/SQS message bus. Each bounded context maps to one Spring Boot application, built independently with Gradle 8, and deployed as a Docker container. The stack is chosen for operational simplicity, strong typing at scale, and compatibility with the event-driven architecture defined in ADR003.

---

## Backend Stack

| Component | Technology | Version | Purpose |
|---|---|---|---|
| Language | Java | 25 | Strong typing across all services and shared domain types |
| Framework | Spring Boot | 3.x | Dependency injection, modular architecture mapping to bounded contexts |
| Build Orchestration | Gradle 8 (multi-module) | 8.x | Monorepo task graph — lint, test, build across all services in parallel |
| Build Tool | Gradle bootJar | 8.x | Fast Spring Boot fat-jar packaging for production Docker images |
| Package Manager | Gradle wrapper | — | Reproducible builds via `./gradlew` — no separate install step |
| Database | PostgreSQL 15 | 15.x | Primary data store; RLS for multi-tenant isolation (ADR002) |
| Cache | Redis 7 | 7.x (AWS ElastiCache) | Slot availability cache, TenantConfig hot cache, idempotency key lookup |
| Message Bus | AWS SNS + SQS FIFO | Managed | Event fan-out (SNS) and per-consumer ordered delivery (SQS FIFO) — ADR003 |
| Containerisation | Docker | Latest | Multi-stage builds, eclipse-temurin base images, non-root user |
| Container Orchestration | AWS ECS Fargate | Managed | Serverless container execution — no EC2 node management |
| Service Discovery | AWS Cloud Map + ALB | Managed | Private DNS namespaces per environment; path-based routing via ALB |
| Connection Pooling | PgBouncer | 1.21.x | Transaction-mode pooling; max 200 connections per service; ECS sidecar |
| JDBC Pool | HikariCP (Spring Data JPA) | Bundled | Connection pool within each service JVM; backed by PgBouncer |
| DB Migrations | Flyway | 10.x | Versioned, forward-only migrations; run as ECS one-off task pre-deployment |
| Stream Processing | AWS Kinesis Data Streams | Managed | Real-time anomaly detection for analytics-service (in scope from v1.0) |
| Secrets | AWS Secrets Manager | Managed | Encrypted secret storage; automatic rotation; ECS task injection |

---

## Frontend Stack

| Component | Technology | Purpose |
|---|---|---|
| Framework | Next.js 14 | React Server Components for booking pages; client components for slot browser |
| State Management | Zustand | Lightweight client state for multi-step booking wizard |
| Hosting | AWS Amplify Hosting | CI/CD integration; SSR via Lambda@Edge; Git-based deployments |
| CDN | Amazon CloudFront | Static asset caching; booking page HTML at edge with tenant_id-aware cache keys |
| Tenant Routing | Subdomain routing | `{tenant_slug}.anji-schedulo.com` for Starter/Pro; custom CNAME for Enterprise |
| SSL | AWS Certificate Manager | Automatic TLS certificate provisioning for all subdomains and ALB listeners |

---

## Observability Stack

| Component | Technology | Purpose |
|---|---|---|
| Logs | Grafana Loki (S3 backend) | Structured log storage, retention 30 days |
| Metrics | Grafana Mimir | Prometheus-compatible metrics storage, retention 90 days |
| Traces | Grafana Tempo | Distributed tracing, retention 7 days |
| Dashboards & Alerts | Grafana 10+ | 7 dashboards; PromQL/LogQL alerting; PagerDuty + Slack channels |
| Log Collection | Fluent Bit | ECS sidecar; PII redaction filter; metadata enrichment from ECS task metadata |
| Instrumentation | OpenTelemetry SDK for Java (javaagent) | Auto-instrumentation for HTTP, JDBC, Lettuce Redis, AWS SDK |
| Metrics Exposure | Micrometer (spring-boot-actuator prometheus registry) | `/actuator/prometheus` endpoint on every Spring Boot service; scraped by Mimir |
| On-call Alerting | PagerDuty | P1/P2 severity alerts; escalation policy per on-call rotation |

---

## CI/CD Stack

| Component | Technology | Purpose |
|---|---|---|
| Pipeline | GitHub Actions | Checkstyle, SpotBugs, unit test, integration test, build, deploy |
| Container Registry | Amazon ECR | Docker image storage per service; immutable tags per commit SHA |
| Deployments | ECS Rolling Update | 50% minimum healthy / 200% maximum; automatic rollback on smoke test failure |
| DB Migrations | Flyway via ECS one-off task | Run against target environment before service rollout |
| Branch Strategy | `main` → production; `release/*` → staging; `feature/*` → dev on PR | |

---

## Security Stack

| Component | Technology | Purpose |
|---|---|---|
| Authentication | JWT RS256 (asymmetric) | Token signing (api-gateway private key); validation (public key distributed to all services) |
| Encryption at Rest | AWS KMS (AES-256) | RDS, S3, SQS, SNS, ElastiCache all encrypted with KMS managed keys |
| Encryption in Transit | TLS 1.3 | Enforced on all service-to-service and client-to-gateway connections |
| Tenant Isolation | PostgreSQL RLS | Row-Level Security policies on every table; safe-by-default null context returns zero rows |
| IAM | AWS IAM | Least-privilege policies per ECS task role; no cross-service resource access |
| Certificate Management | AWS Certificate Manager | Public TLS certificates for all CloudFront distributions and ALB listeners |

---

## Technology Choice Rationale

**Java 25** — The monorepo spans 10 services sharing domain types (event envelopes, entity schemas, business rule constants). Java's strong type system and records catch cross-service interface mismatches at compile time. This is especially important for event payload shapes where a wrong field type on `appointment.confirmed` would silently corrupt the dashboard view.

**Spring Boot 3.x** — Spring Boot's application context and component model maps directly to the bounded context model: each context is a Spring Boot application with its own beans, controllers, and event listeners. Dependency injection makes unit testing straightforward. The framework's opinionated structure reduces architectural drift across a 10-service monorepo.

**AWS ECS Fargate** — Serverless container execution eliminates EC2 node patching and AMI management. Auto-scaling on CPU utilisation and SQS queue depth is native to Fargate via Application Auto Scaling. Each service scales independently — a spike in availability queries does not require scaling the booking saga service.

**AWS SNS + SQS FIFO** — Fully managed pub/sub eliminates broker operations (no Kafka cluster to operate, no broker patching). SQS FIFO with `MessageGroupId = tenant_id` guarantees per-tenant event ordering without global throughput limits. Native DLQ support with configurable `maxReceiveCount` requires no additional infrastructure. New consumers can subscribe to an existing SNS topic without modifying the producer service (NFR015).

**PostgreSQL RLS** — Pool multi-tenancy with Row-Level Security provides per-tenant isolation at the database layer without schema-per-tenant provisioning complexity or the connection overhead of silo deployments. A missing or null `app.tenant_id` context returns zero rows — never all rows — making cross-tenant leaks impossible by default even if application code has a bug (BR005, ADR002).
