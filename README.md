# Taskforge

> A production-oriented background job processing system built with Go. It accepts work over a REST API, queues it in Redis, persists state in PostgreSQL, and executes it reliably with a concurrent worker pool, with retries, timeouts, idempotency, and dead-letter handling.

[![Go](https://img.shields.io/badge/Go-1.24-00ADD8?logo=go&logoColor=white)](https://go.dev/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-Database-4169E1?logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![Redis](https://img.shields.io/badge/Redis-Queue-DC382D?logo=redis&logoColor=white)](https://redis.io/)
[![Docker](https://img.shields.io/badge/Docker-Containerized-2496ED?logo=docker&logoColor=white)](https://www.docker.com/)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-Orchestration-326CE5?logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![Terraform](https://img.shields.io/badge/Terraform-IaC-844FBA?logo=terraform&logoColor=white)](https://www.terraform.io/)
[![Status](https://img.shields.io/badge/status-active%20development-orange)](#roadmap)

---

## Table of Contents

- [Overview](#overview)
- [Key Features](#key-features)
- [Architecture](#architecture)
- [Job Lifecycle](#job-lifecycle)
- [Reliability Design](#reliability-design)
- [Supported Job Types](#supported-job-types)
- [API](#api)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [Usage Example](#usage-example)
- [Testing](#testing)
- [Design Decisions](#design-decisions)
- [Roadmap](#roadmap)
- [Skills Demonstrated](#skills-demonstrated)
- [License](#license)

---

## Overview

Many backend operations should not run inside an HTTP request: generating large reports, processing files, delivering webhooks, sending bulk email, or exporting data. Doing so causes timeouts, resource exhaustion, and brittle failure handling.

This platform separates **request handling** from **background execution**. The API validates and persists a job, publishes it to a queue, and responds immediately. Dedicated workers process the job asynchronously and independently of the API.

```mermaid
flowchart LR
    C[Client] -->|POST /api/v1/jobs| A[Go API]
    A -->|persist| P[(PostgreSQL)]
    A -->|enqueue| R[(Redis Queue)]
    R --> W[Go Worker Pool]
    W -->|update state| P
    W --> X[Execute Job]
```

---

## Key Features

- **Asynchronous job processing**: the API returns immediately; workers do the heavy lifting
- **Concurrent worker pool**: goroutine and channel based, with configurable concurrency limits
- **Reliable execution**: automatic retries with exponential backoff, per-job timeouts, and a maximum attempt count
- **Dead-letter handling**: permanently failing jobs are isolated for inspection and manual retry
- **Idempotency**: protection against duplicate delivery and duplicate side effects
- **Graceful shutdown**: workers finish in-flight jobs before exiting
- **Pluggable executors**: new job types are added through an executor registry
- **Authentication and RBAC**: JWT-based auth with `USER`, `OPERATOR`, and `ADMIN` roles
- **Full execution history**: every attempt and state transition is recorded
- **Cloud-native deployment**: Docker, Kubernetes, Terraform, and CI/CD with GitHub Actions
- **Operational visibility**: structured logging, health and readiness probes, and a React dashboard

---

## Architecture

```mermaid
flowchart TB
    UI[React Dashboard] -->|REST / WebSocket| API

    subgraph Core[Application]
        API[Go API<br/>Auth · RBAC · Jobs · Projects]
        POOL[Go Worker Pool<br/>Worker 1..N]
    end

    API --> PG[(PostgreSQL<br/>users · projects · jobs<br/>attempts · events)]
    API --> RD[(Redis<br/>queue · locks · transient state)]
    RD --> POOL
    POOL --> PG

    POOL --> E1[Email Executor]
    POOL --> E2[Report Executor]
    POOL --> E3[File Executor]
    POOL --> E4[Webhook Executor]
    E1 & E2 & E3 & E4 --> EXT[External Services]
```

**Request flow**

1. Authenticate and authorize the request
2. Validate the job payload
3. Persist the job in PostgreSQL
4. Publish the job to the Redis queue
5. Return the job ID to the client
6. A worker consumes the job, executes it, and records the result

---

## Job Lifecycle

```mermaid
stateDiagram-v2
    [*] --> PENDING
    PENDING --> QUEUED
    QUEUED --> RUNNING
    RUNNING --> SUCCESS
    RUNNING --> FAILED
    FAILED --> RETRYING: attempts remaining
    RETRYING --> RUNNING
    FAILED --> DEAD_LETTER: max attempts reached
    QUEUED --> CANCELLED
    RUNNING --> CANCELLED
    SUCCESS --> [*]
    DEAD_LETTER --> [*]
    CANCELLED --> [*]
```

| State         | Description                              |
| ------------- | ---------------------------------------- |
| `PENDING`     | Created, not yet queued                  |
| `QUEUED`      | Waiting for a worker                     |
| `RUNNING`     | Being processed by a worker              |
| `SUCCESS`     | Completed successfully                   |
| `FAILED`      | Execution failed                         |
| `RETRYING`    | Waiting for another attempt              |
| `CANCELLED`   | Cancelled by a user or operator          |
| `DEAD_LETTER` | Exceeded its retry limit                 |

---

## Reliability Design

Distributed systems are defined by how they behave when things go wrong. This project treats failure handling as a first-class concern.

| Concern                | Approach                                                                          |
| ---------------------- | --------------------------------------------------------------------------------- |
| **Transient failures** | Automatic retries with exponential backoff (1s, 2s, 4s, 8s, ...)                  |
| **Poison jobs**        | Max attempts, then routed to a dead-letter state for investigation                |
| **Hung work**          | Per-job deadlines enforced with `context.Context`                                 |
| **Duplicate delivery** | Idempotency checks to prevent duplicate emails, webhooks, and exports             |
| **Worker crashes**     | Job recovery so another worker can pick up abandoned work                         |
| **Deploys / scaling**  | Graceful shutdown: stop accepting work, finish active jobs, release resources     |
| **Durability**         | PostgreSQL is the source of truth; Redis is treated as a fast, transient queue    |

**Failure scenarios explicitly tested:** Redis unavailable, PostgreSQL unavailable, worker crash, API crash, network timeout, external API failure, duplicate delivery, long-running jobs, and database transaction failure.

---

## Supported Job Types

| Type               | Purpose                                                      |
| ------------------ | ------------------------------------------------------------ |
| `send_email`       | Template-based application email                             |
| `generate_report`  | Asynchronous report generation (e.g., CSV)                   |
| `process_file`     | Validate, process, and store results for uploaded files      |
| `webhook_delivery` | HTTP delivery to external systems with retries and backoff   |
| `data_export`      | Export application data and notify the user                  |
| `sync_data`        | Background synchronization with another data source          |
| `cleanup`          | Scheduled maintenance of expired records and temporary files |

Jobs can be triggered by an **API request**, by **another job's completion event** (e.g., `generate_report` → `report.completed` → `send_email`), or on a **schedule**.

```mermaid
flowchart LR
    U[User] --> A[API] --> J1[generate_report]
    J1 --> OK{Success}
    OK --> EV[report.completed]
    EV --> J2[send_email]
    J2 --> N[User notified]
```

---

## API

Base path: `/api/v1`. Documented with OpenAPI.

```text
POST   /jobs                 Create a job
GET    /jobs                 List jobs
GET    /jobs/{id}            Get a job
POST   /jobs/{id}/retry      Retry a failed or dead-letter job
POST   /jobs/{id}/cancel     Cancel a job
GET    /jobs/{id}/history    View execution history

GET    /projects             List projects
POST   /projects             Create a project
GET    /users                List users (admin)

GET    /health               Liveness probe
GET    /ready                Readiness probe
```

**Roles**

| Role       | Capabilities                                                |
| ---------- | ----------------------------------------------------------- |
| `USER`     | Create, view, and cancel own jobs                           |
| `OPERATOR` | View all jobs, retry, cancel, and inspect failures          |
| `ADMIN`    | Manage users and projects; full system access               |

Internal service communication is defined with **gRPC** and Protocol Buffers (`SubmitJob`, `GetJob`, `CancelJob`).

---

## Tech Stack

| Area            | Technology                                          |
| --------------- | --------------------------------------------------- |
| Language        | Go                                                  |
| API             | REST (OpenAPI), gRPC, Protocol Buffers, WebSockets  |
| Data            | PostgreSQL, Redis                                   |
| Auth            | JWT, role-based access control                      |
| Frontend        | React                                               |
| Containers      | Docker, Docker Compose                              |
| Orchestration   | Kubernetes                                          |
| Infrastructure  | Terraform, AWS, GCP, Nginx                          |
| CI/CD           | GitHub Actions                                      |
| Events          | Domain events; Kafka where justified                |

---

## Project Structure

```text
go-job-platform/
├── cmd/
│   ├── api/                 # API entrypoint
│   └── worker/              # Worker entrypoint
├── internal/
│   ├── auth/                # Authentication, JWT, middleware
│   ├── config/              # Configuration
│   ├── database/            # PostgreSQL connection and migrations
│   ├── jobs/                # Job handlers, service, repository, executors
│   ├── projects/
│   ├── users/
│   ├── workers/             # Worker, pool, executor registry
│   ├── queue/               # Queue abstraction and Redis implementation
│   ├── grpc/
│   └── middleware/
├── api/
│   ├── openapi/
│   └── proto/
├── migrations/
├── tests/
│   ├── integration/
│   └── e2e/
├── deployments/
│   ├── docker/
│   └── kubernetes/
├── terraform/
│   ├── environments/
│   └── modules/
├── frontend/                # React dashboard
├── .github/workflows/       # CI/CD
├── docker-compose.yml
├── Dockerfile
└── go.mod
```

---

## Getting Started

### Prerequisites

- Go 1.24+
- Docker and Docker Compose
- Git
- *(Optional)* Node.js, Kubernetes, Terraform

### Setup

```bash
git clone https://github.com/<your-username>/go-job-platform.git
cd go-job-platform
```

Create a `.env` file:

```env
DATABASE_URL=postgres://postgres:postgres@localhost:5432/jobplatform
REDIS_URL=redis://localhost:6379
JWT_SECRET=change-me-in-real-environments
```

> Never commit real secrets.

Start dependencies, then run the API and a worker:

```bash
docker compose up -d postgres redis

go run ./cmd/api        # terminal 1
go run ./cmd/worker     # terminal 2
```

Or run everything in containers:

```bash
docker compose up
```

---

## Usage Example

Create a job:

```bash
curl -X POST http://localhost:8080/api/v1/jobs \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "generate_report",
    "payload": { "project_id": 123, "format": "csv" },
    "priority": 5,
    "max_attempts": 3
  }'
```

```json
{ "job_id": "job_01JXYZ123", "status": "QUEUED" }
```

Check its progress:

```bash
curl -H "Authorization: Bearer <token>" \
  http://localhost:8080/api/v1/jobs/job_01JXYZ123
```

```json
{ "id": "job_01JXYZ123", "type": "generate_report", "status": "SUCCESS", "attempts": 1 }
```

---

## Testing

```bash
go test ./...            # unit tests
go build ./...           # build check
```

| Layer       | Coverage                                                                           |
| ----------- | ---------------------------------------------------------------------------------- |
| Unit        | Job service, retry logic, validation, authorization                                |
| API         | HTTP request and response behaviour                                                |
| Integration | API + PostgreSQL + Redis                                                           |
| Worker      | Success, failure, retry, timeout, cancellation, duplicate delivery, max attempts   |
| End-to-end  | Create → queue → execute → persist result                                          |

---

## Design Decisions

**PostgreSQL for durability, Redis for speed.** PostgreSQL is the source of truth for jobs, history, and users. Redis handles fast queue operations, locks, and short-lived state. Redis is never the permanent record of job state.

**API and workers are separate services.** The API never executes long-running work. This lets each scale independently, for example by adding worker replicas when the queue grows.

```mermaid
flowchart LR
    Q[(Queue depth grows)] --> S[Scale worker replicas]
    S --> T[More concurrent throughput]
```

**Infrastructure must earn its place.** Each technology is introduced to solve a concrete problem: Redis for queue throughput, Kafka only if event streaming is justified, Kubernetes for scaling and orchestration, Terraform for reproducible infrastructure.

**Executor registry.** Job types map to executors through a registry, so adding a new workload does not require changing the worker core.

---

## Roadmap

- [ ] Go HTTP server and project foundation
- [ ] Job API
- [ ] PostgreSQL persistence and migrations
- [ ] Authentication and RBAC
- [ ] Concurrent worker pool
- [ ] Redis queue
- [ ] Retries, exponential backoff, and timeouts
- [ ] Dead-letter handling, cancellation, and idempotency
- [ ] Multiple job types and executor registry
- [ ] gRPC and Protocol Buffers
- [ ] Event-driven processing
- [ ] Docker and Docker Compose
- [ ] Automated tests and GitHub Actions CI/CD
- [ ] Kubernetes manifests
- [ ] Terraform infrastructure
- [ ] AWS and GCP deployment
- [ ] Observability and production hardening
- [ ] React operations dashboard

**Future ideas:** priority queues, scheduled jobs, job dependencies and batching, rate limiting, per-project quotas, worker autoscaling, workflow orchestration, multi-tenant isolation, audit logs, and chaos testing.

---

## Skills Demonstrated

| Area                    | Highlights                                                                   |
| ----------------------- | ---------------------------------------------------------------------------- |
| **Go**                  | Goroutines, channels, worker pools, context, interfaces, graceful shutdown   |
| **Backend**             | REST design, service boundaries, validation, error handling, auth, RBAC      |
| **Distributed systems** | Queues, retries, backoff, idempotency, dead letters, events, backpressure    |
| **Data**                | PostgreSQL schema design, transactions, indexing; Redis                      |
| **Infrastructure**      | Docker, Kubernetes, Terraform, Nginx, AWS, GCP                               |
| **Delivery**            | Testing strategy, GitHub Actions, observability, operational debugging       |

---

## License

Released for educational and portfolio purposes. See [`LICENSE`](LICENSE) for details.
