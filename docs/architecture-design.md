# AIDevFlow SaaS — Architecture Design

## Overview

AIDevFlow is a **compiler + coordination layer** for AI-powered software development workflows. It compiles tenant-designed workflows into GitHub Actions or GitLab CI pipelines, coordinates human gates and service calls centrally, and tracks run state — but never executes code itself. Execution always happens on the tenant's own runners.

This document covers: system components, GCP infrastructure, tech stack decisions, cost estimates, and deployment topology.

---

## System Architecture

```mermaid
graph TB
    subgraph Tenants["Tenant Environments"]
        GHA[GitHub Actions Runner]
        GLC[GitLab CI Runner]
        JIRA[Jira]
        SC[SonarCloud]
        GCP_PS[GCP Pub/Sub]
    end

    subgraph Platform["AIDevFlow Platform — GCP"]
        subgraph PublicLayer["Public Layer (Cloud Load Balancing + Cloud Armor)"]
            WEB[Web UI\nNext.js / Cloud Run]
            WHR[Webhook Receiver\nGo / Cloud Run]
            APIGW[API Gateway\nCloud Load Balancing]
        end

        subgraph CoreLayer["Core Services (Cloud Run — per region)"]
            COORD[Coordination API\nNode.js + Fastify]
            COMPILER[Workflow Compiler\nNode.js]
            WORKER[Event Worker\nNode.js]
        end

        subgraph DataLayer["Data Layer"]
            PG[(Cloud SQL\nPostgreSQL)]
            REDIS[(Cloud Memorystore\nRedis)]
            GCS[Cloud Storage\nArtifacts]
            SM[Secret Manager]
        end

        subgraph AsyncLayer["Async / Scheduling"]
            PUBSUB[Cloud Pub/Sub\nInternal Bus]
            SCHEDULER[Cloud Scheduler]
            TASKS[Cloud Tasks]
        end

        subgraph ObsLayer["Observability"]
            LOGGING[Cloud Logging]
            MONITORING[Cloud Monitoring]
            TRACE[Cloud Trace]
        end
    end

    subgraph Auth["Identity"]
        CLERK[Clerk / Auth0]
    end

    %% Tenant → Platform
    GHA -->|step result POST| APIGW
    GLC -->|step result POST| APIGW
    JIRA -->|assignment webhook| WHR
    SC -->|analysis webhook| WHR
    GCP_PS -->|error alert| WHR

    %% Public layer internal
    WHR -->|validated event| PUBSUB
    WEB -->|API calls| APIGW
    APIGW --> COORD

    %% Core services
    PUBSUB --> WORKER
    WORKER --> COORD
    COORD --> COMPILER
    COMPILER -->|push YAML| GHA
    COMPILER -->|push YAML| GLC

    %% Data access
    COORD --> PG
    COORD --> REDIS
    COORD --> SM
    COORD --> GCS
    WORKER --> PG
    WORKER --> REDIS

    %% Scheduling
    SCHEDULER -->|dead job check| TASKS
    TASKS --> COORD

    %% Auth
    WEB --> CLERK
    COORD --> CLERK

    %% Observability
    COORD --> LOGGING
    WHR --> LOGGING
    WORKER --> LOGGING
    COORD --> TRACE
```

---

## Service Breakdown

### 1. Web UI — Next.js

The tenant-facing dashboard: workflow designer, run trace, task library, admin config.

**Why Next.js:**
- App Router with React Server Components for fast initial load
- API routes for BFF (backend-for-frontend) pattern — avoids exposing Coordination API directly to browser
- SSR for the marketing/pricing pages (SEO)
- Static export for the dashboard shell (CDN-cacheable)

**Hosting:** Cloud Run (containerised) behind Cloud CDN. Firebase Hosting is an alternative but Cloud Run gives more control over headers and auth middleware.

**Auth:** Clerk SDK for Next.js — handles login, SSO federation, MFA, session tokens. Zero auth code to write.

---

### 2. Webhook Receiver — Go

The only public-facing inbound endpoint. Receives webhooks from Jira, SonarCloud, GCP Pub/Sub push subscriptions, and generic HTTP callers.

**Why Go (not Node.js):**
- Stateless, high-throughput, latency-sensitive — Go is significantly faster and uses less memory than Node.js for this workload
- Small Docker images (~10MB vs ~150MB for Node)
- Cheap on Cloud Run: Go handles more concurrent requests per instance
- No shared code needed with the rest of the platform at runtime — just validates and publishes

**Responsibilities:**
- Validate webhook signatures (Jira HMAC-SHA256, SonarCloud secret, GCP Pub/Sub JWT)
- Check webhook URL token (unguessable UUID in path)
- Rate limit per tenant (Redis counter)
- Deduplicate events (Redis set with 5-minute TTL)
- Publish validated, structured event to Cloud Pub/Sub internal topic
- Return 200 immediately — never block on downstream processing

```mermaid
sequenceDiagram
    participant J as Jira
    participant WHR as Webhook Receiver (Go)
    participant R as Redis
    participant PS as Cloud Pub/Sub

    J->>WHR: POST /webhooks/{uuid}/jira (HMAC signed)
    WHR->>WHR: Verify URL UUID → resolve tenant
    WHR->>WHR: Verify HMAC signature
    WHR->>R: Check dedup key (event_id)
    R-->>WHR: Not seen → proceed
    WHR->>R: Write dedup key (TTL 5min)
    WHR->>R: Check rate limit (tenant, per minute)
    R-->>WHR: Under limit → proceed
    WHR->>PS: Publish structured event
    WHR-->>J: 200 OK
```

---

### 3. Coordination API — Node.js + Fastify

The brain of the platform. Handles all state, secrets, service proxying, and human gate management.

**Why Fastify over Express:**
- 2× faster throughput (benchmarked)
- First-class TypeScript support
- Plugin ecosystem (auth, rate limit, schema validation) is cleaner
- Built-in JSON schema validation with Ajv (fast, type-safe)

**Why Node.js over Go here:**
- Shared TypeScript types with the Web UI and Compiler (monorepo)
- Prisma ORM for PostgreSQL — excellent DX, type-safe queries, migration management
- BullMQ job queue (Redis-backed) — fits naturally in Node
- The Coordination API is IO-bound (DB reads, secret fetches, external API calls) — Node's async model is ideal

**Key responsibilities:**
- Authenticate runner requests (`AIDEVFLOW_TENANT_TOKEN` → `tenant_api_tokens` lookup)
- Persist step results, outputs, run state (`pipeline_run_steps`, `task_executions`)
- Proxy service calls: Jira, SonarCloud, GitHub (fetching secrets from Secret Manager at call time)
- Human gate open/close + notification dispatch
- Dry run enforcement (reject write-type callbacks during dry runs)
- Token rotation (re-provision secrets to GitHub/GitLab via their APIs)
- AIDEVFLOW_TENANT_TOKEN validation: constant-time HMAC comparison against stored hash

```mermaid
sequenceDiagram
    participant Runner as GitHub Actions Runner
    participant COORD as Coordination API
    participant SM as Secret Manager
    participant PG as PostgreSQL
    participant JIRA as Jira

    Runner->>COORD: POST /steps/create-jira-ticket\n(Bearer: TENANT_TOKEN)
    COORD->>PG: Validate token hash → resolve tenant
    COORD->>PG: Check dry_run flag for this run
    COORD->>SM: Fetch tenants/{id}/plugins/jira/api_token
    SM-->>COORD: Jira API token
    COORD->>JIRA: POST /rest/api/3/issue
    JIRA-->>COORD: {ticket_id, ticket_url}
    COORD->>PG: Persist step result + task_execution row
    COORD-->>Runner: {ticket_id, ticket_url}
```

---

### 4. Workflow Compiler — Node.js

Converts tenant workflow YAML into GitHub Actions or GitLab CI YAML, pushes it to the repo, provisions runner secrets, and maintains the `repo_skill_manifests` table.

**Runs as:** a separate Cloud Run service called by the Coordination API (or as a library within it — start as a library, extract when needed).

**Key operations:**
- Parse workflow YAML (js-yaml)
- Resolve task library entries, skill versions
- Generate target CI/CD YAML (GitHub Actions jobs or GitLab CI stages)
- Compile skill YAML → `.claude/skills/SKILL.md` or `.agents/skills/SKILL.md`
- Push files to repo via GitHub App installation token (scoped to specific repo) or GitLab token
- Provision `AIDEVFLOW_TENANT_TOKEN` and AI provider key as runner secrets via GitHub/GitLab API
- Run deploy-time validation (runner label warning, required plugins present, `return_assignee` valid)
- Diff `repo_skill_manifests` and delete orphaned skill files

```mermaid
flowchart TD
    A[Workflow YAML saved/updated] --> B[Validate: workflow_type + trigger + output consistent]
    B --> C{Validation passed?}
    C -->|No| D[Return validation errors to Web UI]
    C -->|Yes| E[Resolve all task@version from Task Library]
    E --> F[Compile skill YAMLs → native SKILL.md format]
    F --> G{ci_target?}
    G -->|github_actions| H[Generate .github/workflows/name.yml]
    G -->|gitlab_ci| I[Generate .gitlab-ci.yml]
    H --> J[Generate runner job per task type\nwith timeout-minutes, runs-on resolved from runner_label]
    I --> J
    J --> K[Diff repo_skill_manifests\nDelete orphaned skill files]
    K --> L[Push all files to repo\nvia GitHub App scoped token]
    L --> M[Provision runner secrets\nAIDEVFLOW_TENANT_TOKEN + AI key]
    M --> N[Update repo_skill_manifests in DB]
    N --> O{cli tasks + runner_label.cli = ubuntu-latest?}
    O -->|Yes| P[Return soft warning to Web UI]
    O -->|No| Q[Deploy complete]
```

---

### 5. Event Worker — Node.js

Subscribes to Cloud Pub/Sub internal topics. Translates validated webhook events into workflow run triggers.

**Responsibilities:**
- Pull from `aidevflow.internal.events` Pub/Sub subscription
- Resolve tenant from event (webhook UUID → `tenant_plugins`)
- Map event type → workflow (`tenant_system_users` for Jira assignments, `repo_registry` for SonarCloud)
- Check concurrency: is there an active run for `(tenant_id, workflow_id, ticket_id)`?
  - If yes: post Jira comment via Coordination API, ack message, done
  - If no: create `pipeline_runs` row, dispatch trigger to CI/CD via `repository_dispatch` (GitHub) or pipeline trigger API (GitLab)
- Ack Pub/Sub message only after successful dispatch

---

### 6. Scheduled Jobs — Cloud Scheduler + Cloud Run Jobs

| Job | Schedule | What it does |
|---|---|---|
| Dead job detector | Every 5 min | Marks steps stuck in `running` beyond timeout as `failed`; notifies tenant |
| Trial expiry check | Daily 00:00 UTC | Suspends tenants whose `trial_ends_at` has passed |
| Offboarding cleanup | Daily 01:00 UTC | Runs hard delete for tenants 30 days past suspension |
| Token rotation reminder | Weekly | Alerts tenants whose API tokens haven't been rotated in 90 days |

---

## Multi-Region Topology

```mermaid
graph LR
    subgraph Global["Global (single deployment)"]
        WEB[Web UI\nCloud Run + CDN]
        CLERK[Clerk Auth]
        GLB[Cloud Load Balancer\n+ Cloud Armor]
    end

    subgraph US["Region: us-central1"]
        COORD_US[Coordination API]
        PG_US[(Cloud SQL\nPostgreSQL)]
        REDIS_US[(Memorystore\nRedis)]
        SM_US[Secret Manager]
        PS_US[Pub/Sub]
        WORKER_US[Event Worker]
    end

    subgraph EU["Region: europe-west1"]
        COORD_EU[Coordination API]
        PG_EU[(Cloud SQL\nPostgreSQL)]
        REDIS_EU[(Memorystore\nRedis)]
        SM_EU[Secret Manager]
        PS_EU[Pub/Sub]
        WORKER_EU[Event Worker]
    end

    subgraph AP["Region: asia-southeast1"]
        COORD_AP[Coordination API]
        PG_AP[(Cloud SQL\nPostgreSQL)]
        REDIS_AP[(Memorystore\nRedis)]
        SM_AP[Secret Manager]
        PS_AP[Pub/Sub]
        WORKER_AP[Event Worker]
    end

    WEB --> GLB
    GLB -->|tenant.region=us| COORD_US
    GLB -->|tenant.region=eu| COORD_EU
    GLB -->|tenant.region=ap| COORD_AP

    COORD_US --> PG_US & REDIS_US & SM_US
    COORD_EU --> PG_EU & REDIS_EU & SM_EU
    COORD_AP --> PG_AP & REDIS_AP & SM_AP

    PS_US --> WORKER_US --> COORD_US
    PS_EU --> WORKER_EU --> COORD_EU
    PS_AP --> WORKER_AP --> COORD_AP
```

**Region selection:** tenant's region is encoded in `AIDEVFLOW_TENANT_TOKEN`. The Cloud Load Balancer uses URL-based routing: the token is passed in the `Authorization` header, but a lightweight auth sidecar (or the GLB itself via header inspection) routes to the correct regional backend. Simpler alternative: use regional subdomains (`api-eu.aidevflow.io`, `api-us.aidevflow.io`) — the Web UI calls the correct subdomain after login.

---

## Data Flow: Jira-Triggered Workflow

```mermaid
sequenceDiagram
    actor Human as Sarah (Team Lead)
    participant JIRA as Jira
    participant WHR as Webhook Receiver
    participant PS as Pub/Sub
    participant WORKER as Event Worker
    participant COORD as Coordination API
    participant GHA as GitHub Actions
    participant SM as Secret Manager

    Human->>JIRA: Assigns ticket PROJ-42 to aidevflow-analyst
    JIRA->>WHR: POST /webhooks/{uuid}/jira (assignment event)
    WHR->>WHR: Validate HMAC + UUID + dedup
    WHR->>PS: Publish {type: jira_assignment, ticket: PROJ-42, assignee: aidevflow-analyst}
    PS->>WORKER: Deliver message
    WORKER->>COORD: Check active run for (tenant, analyze-ticket, PROJ-42)
    COORD-->>WORKER: No active run
    WORKER->>COORD: Create pipeline_run row (status: running)
    WORKER->>GHA: repository_dispatch {run_id, ticket_id: PROJ-42}
    GHA->>GHA: Trigger compiled workflow pipeline
    GHA->>COORD: POST /steps/jira-read-thread (Bearer: TENANT_TOKEN)
    COORD->>SM: Fetch Jira API token
    COORD->>JIRA: GET /issue/PROJ-42 + comments
    JIRA-->>COORD: Ticket + 12 comments
    COORD-->>GHA: {ticket, comments}
    GHA->>GHA: Run ticket-analyst ai_skill job\n(api mode → Bedrock EU)
    GHA->>COORD: POST /steps/ticket-analyst/complete\n{plan: "Fix null check..."}
    COORD->>COORD: dry_run=false → proceed to output handler
    COORD->>SM: Fetch Jira token
    COORD->>JIRA: POST /issue/PROJ-42/comment (plan text)
    COORD->>JIRA: PUT /issue/PROJ-42 (assignee: sarah.chen)
    COORD->>COORD: Mark run complete, record task_executions
    JIRA-->>Human: Ticket reassigned to Sarah with AI comment
```

---

## Tech Stack

### Recommended Stack

| Layer | Technology | Why |
|---|---|---|
| **Frontend** | Next.js 15 (TypeScript) | App Router, RSC, Clerk integration, SSR for marketing |
| **UI Components** | shadcn/ui + Tailwind CSS | Accessible, composable, no runtime overhead |
| **Client state** | Zustand | Lightweight, TypeScript-first |
| **Data fetching** | TanStack Query | Server state, caching, optimistic updates |
| **Coordination API** | Fastify 5 (Node.js, TypeScript) | 2× faster than Express; JSON schema validation built-in |
| **Webhook Receiver** | Go 1.23 (Chi router) | High throughput, tiny memory footprint, small image |
| **ORM** | Prisma 6 | Type-safe queries, migration management, RLS support |
| **Job queue** | BullMQ (Redis-backed) | Reliable, retry logic, delayed jobs |
| **YAML parsing** | js-yaml | Parse workflow YAML; generate CI/CD YAML |
| **Auth** | Clerk | OIDC/SSO, MFA, Next.js SDK, zero auth code |
| **Database** | PostgreSQL 16 (Cloud SQL) | RLS, JSONB for config fields, mature, managed |
| **Cache / queue** | Redis 7 (Cloud Memorystore) | BullMQ backend, rate limiting, deduplication |
| **Secret store** | GCP Secret Manager | Per-tenant IAM scoping, audit logs, versioning |
| **Event bus** | GCP Cloud Pub/Sub | Push subscriptions, at-least-once delivery, regional |
| **Monorepo** | Turborepo | Shared packages (types, db schema, skill templates) |
| **Infra as code** | Terraform | GCP resources, per-region modules |
| **Containerisation** | Docker + Artifact Registry | Cloud Run deployments |
| **CI/CD (platform itself)** | Cloud Build | Build, test, push images |

### Alternative Languages Considered

```mermaid
quadrantChart
    title Language Trade-offs: Dev Speed vs Runtime Efficiency
    x-axis Low Dev Speed --> High Dev Speed
    y-axis Low Runtime Efficiency --> High Runtime Efficiency
    quadrant-1 Sweet spot for specific services
    quadrant-2 Production workhorse
    quadrant-3 Avoid for this project
    quadrant-4 Rapid prototyping
    Go: [0.55, 0.85]
    Rust: [0.15, 0.98]
    Python FastAPI: [0.80, 0.35]
    Node.js TypeScript: [0.75, 0.60]
    Bun TypeScript: [0.78, 0.70]
    Elixir Phoenix: [0.50, 0.75]
    Java Spring: [0.40, 0.72]
    Deno TypeScript: [0.72, 0.62]
```

**Node.js (TypeScript) — primary language for Coordination API, Compiler, Worker**
- Monorepo with Web UI — shared types, shared Prisma client, shared YAML schemas
- IO-bound workload (DB reads, secret fetches, external API calls) — Node is ideal
- Largest ecosystem of npm packages for the integrations needed
- Fastify provides enough performance overhead; horizontal scaling on Cloud Run handles load

**Go — Webhook Receiver only**
- Stateless, high-throughput, latency-critical
- Ideal for HMAC validation + Pub/Sub publish under load
- 10MB Docker image vs 150MB for Node
- No shared code needed with the TypeScript codebase at runtime

**Bun — future consideration**
- Drop-in Node.js replacement, 3-4× faster startup (important for Cloud Run cold starts)
- Built-in TypeScript transpilation, no `ts-node` needed
- Not yet production-battle-tested for all npm packages — adopt in 12 months when ecosystem is more stable

**Elixir/Phoenix — considered for real-time (WebSocket) features**
- Phoenix Channels are excellent for the Run Dashboard live updates
- But: adds a second language to the backend team; Phoenix LiveView covers most real-time needs without WebSockets
- Decision: use Server-Sent Events (SSE) from Fastify for live run updates instead — simpler, works with Cloud Run

**Python (FastAPI) — rejected**
- GIL is a liability for concurrent webhook handling
- Slower cold starts than Node on Cloud Run
- No shared types with TypeScript frontend

**Rust — rejected for v1**
- Development velocity too slow for a startup finding product-market fit
- Revisit for the Workflow Compiler once the YAML schema is stable and performance becomes a bottleneck

---

## Monorepo Structure

```
aidevflow/
├── apps/
│   ├── web/                    # Next.js 15 — Web UI
│   │   ├── app/                # App Router pages
│   │   ├── components/         # shadcn/ui components
│   │   └── middleware.ts       # Clerk auth middleware
│   │
│   ├── api/                    # Fastify — Coordination API
│   │   ├── src/
│   │   │   ├── routes/         # /runs, /steps, /human-gate, /tokens
│   │   │   ├── services/       # JiraService, GitHubService, SecretService
│   │   │   ├── workers/        # BullMQ job handlers
│   │   │   └── plugins/        # Fastify plugins (auth, rate-limit, cors)
│   │   └── Dockerfile
│   │
│   ├── webhook-receiver/       # Go — inbound webhook handler
│   │   ├── internal/
│   │   │   ├── handlers/       # jira.go, sonarcloud.go, pubsub.go
│   │   │   ├── validator/      # HMAC verification
│   │   │   └── publisher/      # Pub/Sub client
│   │   └── Dockerfile
│   │
│   └── compiler/               # Node.js — Workflow → CI/CD YAML
│       ├── src/
│       │   ├── parsers/        # Parse workflow YAML
│       │   ├── generators/     # github-actions.ts, gitlab-ci.ts
│       │   ├── skill-compiler/ # Platform skill YAML → SKILL.md
│       │   └── deployer/       # Push to repo, provision secrets
│       └── Dockerfile
│
├── packages/
│   ├── types/                  # Shared TypeScript types (workflow, task, skill schemas)
│   ├── db/                     # Prisma schema, migrations, seed data
│   │   ├── schema.prisma
│   │   └── migrations/
│   ├── skill-library/          # Built-in skill YAML files (versioned)
│   │   └── skills/
│   │       ├── code-implementer/1.0.0.yaml
│   │       ├── ticket-analyst/1.0.0.yaml
│   │       └── ...
│   └── workflow-templates/     # Global workflow templates (versioned)
│       └── templates/
│           ├── report-error/1.0.0.yaml
│           ├── analyze-ticket/1.0.0.yaml
│           └── ...
│
├── infra/
│   ├── terraform/
│   │   ├── modules/
│   │   │   ├── region/         # Cloud Run, Cloud SQL, Redis, Pub/Sub per region
│   │   │   └── global/         # Load Balancer, Cloud Armor, Artifact Registry
│   │   ├── us.tf
│   │   ├── eu.tf
│   │   └── ap.tf
│   └── cloudbuild/             # Cloud Build pipeline configs
│
└── turbo.json                  # Turborepo pipeline config
```

---

## GCP Infrastructure Detail

```mermaid
graph TB
    subgraph VPC["VPC Network (per region)"]
        subgraph PublicSubnet["Public Subnet"]
            CR_WHR[Cloud Run\nWebhook Receiver]
            CR_WEB[Cloud Run\nWeb UI]
        end

        subgraph PrivateSubnet["Private Subnet"]
            CR_API[Cloud Run\nCoordination API]
            CR_WORKER[Cloud Run\nEvent Worker]
            CR_COMPILER[Cloud Run\nCompiler]
            CR_JOBS[Cloud Run Jobs\nScheduled Tasks]
        end

        subgraph DataSubnet["Data Subnet (VPC-private only)"]
            CSQL[Cloud SQL\nPostgreSQL\nPrivate IP]
            MEM[Cloud Memorystore\nRedis\nPrivate IP]
        end
    end

    subgraph Managed["GCP Managed Services (no VPC)"]
        SM2[Secret Manager]
        PS2[Cloud Pub/Sub]
        GCS2[Cloud Storage]
        SCHED[Cloud Scheduler]
        TASKS2[Cloud Tasks]
    end

    INTERNET((Internet)) --> GLB[Cloud Load Balancer\n+ Cloud Armor WAF]
    GLB --> CR_WHR & CR_WEB
    CR_WHR --> PS2
    CR_WEB --> CR_API
    CR_API --> CSQL & MEM & SM2 & GCS2
    PS2 --> CR_WORKER
    CR_WORKER --> CR_API
    CR_API --> CR_COMPILER
    SCHED --> TASKS2 --> CR_JOBS
    CR_JOBS --> CR_API
```

**Key networking decisions:**
- Cloud SQL and Memorystore have **private IPs only** — not reachable from the internet
- Cloud Run services in the private subnet use **VPC connector** to reach private data layer
- Coordination API, Worker, Compiler are **not internet-facing** — all traffic via Load Balancer or internal VPC
- Webhook Receiver and Web UI are the only public-facing Cloud Run services

---

## Cost Estimation

### Assumptions (launch: 50 tenants)

- Mix: 30 Trial/Starter, 15 Pro, 5 Enterprise
- ~500 workflow runs/day, ~3,000 task executions/day
- Average job duration: `api` 30s, `service_call` 20s, `cli` skipped (runs on tenant runners)
- 3 regions active from day 1
- Coordination API: ~0.5 vCPU, 512MB per instance, scales to zero

### Monthly Cost Breakdown

```mermaid
pie title GCP Monthly Cost at Launch (~50 tenants)
    "Cloud SQL (3 regions)" : 90
    "Cloud Memorystore Redis (3 regions)" : 105
    "Cloud Run — all services" : 60
    "Cloud Load Balancing + Cloud Armor" : 35
    "Cloud Pub/Sub" : 10
    "Secret Manager" : 30
    "Cloud Storage" : 5
    "Cloud Scheduler + Tasks" : 5
    "Cloud Build + Artifact Registry" : 15
    "Cloud Logging + Monitoring" : 20
    "Bandwidth + misc" : 25
```

| Component | Config | Monthly Cost (USD) |
|---|---|---|
| **Cloud SQL PostgreSQL** | db-g1-small (1 vCPU, 1.7GB) × 3 regions, HA, 20GB SSD | ~$150 |
| **Cloud Memorystore Redis** | Basic 1GB × 3 regions | ~$105 |
| **Cloud Run — Coordination API** | 0.5 vCPU / 512MB, scales to 0, ~500k requests/month | ~$25 |
| **Cloud Run — Webhook Receiver** | 0.25 vCPU / 256MB (Go — tiny), ~50k webhooks/month | ~$5 |
| **Cloud Run — Compiler** | 1 vCPU / 1GB, on-demand only | ~$10 |
| **Cloud Run — Worker** | 0.5 vCPU / 512MB | ~$10 |
| **Cloud Run — Web UI** | 0.5 vCPU / 512MB + CDN | ~$15 |
| **Cloud Load Balancing** | 1 global rule + backend services | ~$20 |
| **Cloud Armor** | Standard policy | ~$10 |
| **Cloud Pub/Sub** | ~1GB messages/month | ~$5 |
| **Secret Manager** | ~500 secrets, 10k accesses/day | ~$30 |
| **Cloud Storage** | 10GB artifacts | ~$3 |
| **Cloud Scheduler** | 4 jobs | ~$1 |
| **Cloud Build** | ~60 build minutes/day | ~$15 |
| **Artifact Registry** | ~5GB images | ~$5 |
| **Cloud Logging + Monitoring** | ~50GB logs/month | ~$25 |
| **Bandwidth** | Egress ~100GB/month | ~$12 |
| **TOTAL** | | **~$496/month** |

### Cost at Scale

| Tenants | Task executions/day | Est. monthly infra cost | Revenue (avg $100/tenant) | Margin |
|---|---|---|---|---|
| 50 | 3,000 | ~$500 | ~$2,500 | 80% |
| 200 | 12,000 | ~$1,200 | ~$20,000 | 94% |
| 500 | 30,000 | ~$2,500 | ~$75,000 | 97% |
| 2,000 | 120,000 | ~$8,000 | ~$300,000 | 97% |

Cloud Run scales to zero — idle tenants cost nothing. The dominant cost is Cloud SQL (fixed per region regardless of load). At 500+ tenants consider moving to **AlloyDB** or **Cloud SQL Enterprise** for better performance; cost increases by ~2× but handles 10× the load.

**Clerk Auth pricing:** ~$0.02/active user/month. At 50 tenants × 10 users = 500 users = ~$10/month. Scales linearly. Switch to Clerk Enterprise ($1,000+/month) if tenants need custom SSO at scale.

---

## Database Schema (Key Relationships)

```mermaid
erDiagram
    TENANTS {
        varchar id PK
        varchar slug UK
        varchar region
        varchar plan
        varchar plan_status
        timestamp trial_ends_at
        varchar billing_email
    }

    TENANT_USERS {
        varchar id PK
        varchar tenant_id FK
        varchar idp_user_id
        varchar role
        varchar email
        boolean mfa_enabled
    }

    TENANT_PLUGINS {
        varchar id PK
        varchar tenant_id FK
        varchar plugin_type
        varchar plugin_name
        text config
        varchar secret_ref
        boolean enabled
    }

    TENANT_API_TOKENS {
        varchar id PK
        varchar tenant_id FK
        varchar token_hash UK
        varchar status
        timestamp revoked_at
    }

    TENANT_SYSTEM_USERS {
        varchar id PK
        varchar tenant_id FK
        varchar jira_account_id
        varchar workflow_id
        boolean managed
    }

    REPO_REGISTRY {
        varchar id PK
        varchar tenant_id FK
        varchar name
        varchar repo_url
        text tech_stack
        text quality_gate
        text test_pipelines
    }

    PIPELINE_RUNS {
        varchar id PK
        varchar tenant_id FK
        varchar workflow_id
        integer workflow_version
        varchar status
        boolean dry_run
        integer quality_gate_fix_iter
        varchar trigger_type
        varchar jira_ticket_id
    }

    PIPELINE_RUN_STEPS {
        varchar id PK
        varchar run_id FK
        varchar tenant_id FK
        varchar step_name
        varchar task_type
        varchar status
        text input_data
        text output_data
        integer duration_ms
    }

    TASK_EXECUTIONS {
        varchar id PK
        varchar tenant_id FK
        varchar run_id FK
        varchar step_id FK
        varchar task_name
        varchar task_type
        boolean billed
        integer duration_ms
    }

    REPO_SKILL_MANIFESTS {
        varchar id PK
        varchar tenant_id FK
        varchar repo_url
        varchar skill_file_path
        varchar workflow_id
    }

    TENANTS ||--o{ TENANT_USERS : has
    TENANTS ||--o{ TENANT_PLUGINS : configures
    TENANTS ||--o{ TENANT_API_TOKENS : issues
    TENANTS ||--o{ TENANT_SYSTEM_USERS : manages
    TENANTS ||--o{ REPO_REGISTRY : registers
    TENANTS ||--o{ PIPELINE_RUNS : owns
    PIPELINE_RUNS ||--o{ PIPELINE_RUN_STEPS : contains
    PIPELINE_RUN_STEPS ||--o{ TASK_EXECUTIONS : meters
    TENANTS ||--o{ REPO_SKILL_MANIFESTS : tracks
```

---

## Deployment Pipeline

```mermaid
flowchart LR
    DEV[Developer\npushes to main] --> CB[Cloud Build\ntriggered]
    CB --> TEST[Run tests\nTypeScript + Go]
    TEST --> BUILD[Build Docker images\nper service]
    BUILD --> AR[Push to\nArtifact Registry]
    AR --> DEPLOY_US[Deploy to\nus-central1]
    DEPLOY_US --> SMOKE_US[Smoke test\nUS]
    SMOKE_US --> DEPLOY_EU[Deploy to\neurope-west1]
    DEPLOY_EU --> SMOKE_EU[Smoke test\nEU]
    SMOKE_EU --> DEPLOY_AP[Deploy to\nasia-southeast1]
    DEPLOY_AP --> DONE[Deploy complete]
```

Sequential regional deployment (US → EU → AP) with smoke tests between regions. A failed smoke test rolls back that region via Cloud Run traffic splitting (keep 100% on previous revision until fixed).

---

## Security Architecture

```mermaid
graph TB
    subgraph PublicBoundary["Public Boundary"]
        CA[Cloud Armor\nWAF + DDoS]
        RL[Rate Limiting\nper IP + per tenant token]
    end

    subgraph AuthBoundary["Auth Boundary"]
        CLERK2[Clerk\nJWT validation]
        TKN[AIDEVFLOW_TENANT_TOKEN\nHMAC-SHA256 hash check]
        WHK[Webhook UUID\n+ HMAC signature]
    end

    subgraph DataBoundary["Data Boundary"]
        RLS2[PostgreSQL RLS\nRow Level Security]
        SM3[Secret Manager\nIAM-scoped per service account]
        ENC[Encryption at rest\nCloud SQL + GCS]
    end

    subgraph ExecutionBoundary["Execution Boundary"]
        PIN[AI CLI version pinned\n@1.2.3 in compiled YAML]
        HASH[bedrock-invoke.sh\nSHA-256 hash verified]
        NOEVAR[AI keys via temp file\nnot env var]
    end

    INTERNET2((Internet)) --> CA --> RL
    RL -->|Web UI requests| CLERK2
    RL -->|Runner API calls| TKN
    RL -->|Inbound webhooks| WHK
    CLERK2 & TKN & WHK --> RLS2
    RLS2 --> SM3 --> ENC
    TKN --> PIN & HASH & NOEVAR
```

---

## MVP Scope Recommendation

Given the competitive analysis (GitHub Copilot for Jira is advancing rapidly on the GitHub+Jira slice), the MVP should target the GitLab+Jira market first — that's where there's no well-funded competitor.

### MVP (3–4 months)

| Feature | Included |
|---|---|
| Web UI: auth, plugin config, system user setup | ✓ |
| Jira assignment trigger → workflow run | ✓ |
| `jira_analysis` workflow type (analyze-ticket pattern) | ✓ |
| `api` execution tasks only (no `cli` in MVP) | ✓ — simpler, no runner needed for analysis |
| GitLab CI compilation (lead with GitLab) | ✓ |
| GitHub Actions compilation | ✓ |
| Run Trace in Web UI | ✓ |
| EU region | ✓ — leads with GDPR compliance |
| Task Test Panel for `api` tasks | ✓ |
| Trial tier (50 executions, 30 days) | ✓ |

**Excluded from MVP:** `cli` tasks (code-implementer, quality-gate-fixer), SonarCloud webhook, GCP Pub/Sub trigger, skill compilation to SKILL.md format, dry run, multi-region (US+AP added after EU validates).

The MVP proves the Jira-assignment-chaining model works and gets real tenants using `analyze-ticket` + `analyze-implementation` workflows before building the more complex code execution layer.

---

## Brainstorm: What Could Change the Architecture

**If Bun matures (6–12 months):** Replace Node.js with Bun across all TypeScript services. Faster cold starts on Cloud Run (Bun boots in ~50ms vs Node's ~300ms), built-in TypeScript, smaller images. Same ecosystem, near-zero migration cost.

**If Cloud Run volume justifies it:** Move from Cloud SQL `db-g1-small` to **AlloyDB** (Postgres-compatible, 2–4× faster read throughput, columnar engine for analytics queries on run history). Cost: ~3× but handles 10× the concurrent connections.

**If real-time dashboard becomes a bottleneck:** Replace Server-Sent Events with **Cloud Pub/Sub push subscriptions to browser** via a WebSocket proxy (Fastify + `ws`). Or adopt **Elixir Phoenix** for just the real-time gateway — it handles millions of concurrent WebSocket connections cheaply.

**If the Workflow Compiler becomes complex:** Extract into a dedicated service written in **Rust** — the compiler is a pure transformation function (YAML in → YAML out), no IO, no shared state. Rust would make it ~10× faster and completely memory-safe. Worth doing once the YAML schema is stable.

**If multi-region data residency becomes a sales requirement for every enterprise deal:** Consider **Terraform Cloud + multi-region Atlantis** for tenant-specific infrastructure rather than shared regional stacks. Each Enterprise tenant gets their own Cloud Run namespace and Cloud SQL instance. Operational overhead is high — only for Enterprise tier at $1,000+/month.
