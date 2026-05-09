# AIDevFlow — Refined Product Requirements

> **Document type:** Collaborative requirements  
> **Roles contributing:** Product Owner · Senior Architect · Backend Lead · Frontend Lead  
> **Source:** `docs/idea-brain-storm-saas.md`  
> **Date:** 2026-05-09  
> **Status:** Living document — update when locked decisions change

---

## 1. Product Vision & Goals

*Product Owner*

AIDevFlow is a **compiler and coordination layer** for AI-powered software development workflows. Tenants design workflows in YAML, the platform compiles them into GitHub Actions or GitLab CI pipelines, and the AI executes on the tenant's own CI/CD runners. The platform never runs code on shared infrastructure.

**Core value proposition:** bring your own tokens and runners, design workflows visually, let the platform handle AI orchestration plumbing.

**Strategic rationale for MVP target (GitLab + Jira):** no well-funded competitor in this combination vs. GitHub + Jira. AU region first (australia-southeast1, Sydney) to serve the ANZ market — the primary early adopter base. Note: GCP has no New Zealand region; `australia-southeast1` is the nearest available GCP region to NZ.

### Success metrics (MVP)

| Metric | Target |
|---|---|
| Time to first workflow run | < 30 minutes from signup |
| Workflow compilation success rate | > 99% |
| Platform-introduced run latency | < 1s for coordination steps |
| Human gate notification delivery | < 10s of workflow reaching gate |
| Trial → paid conversion rate | Baseline to measure from launch |

---

## 2. Tenant Model

*Product Owner · Senior Architect*

### 2.1 Tenant entity

Each **tenant** is an organisation. At signup the tenant receives:

```
Tenant
├── Identity & Users            — admin, engineer, viewer roles; SSO/OIDC via Clerk
├── Plugin Registry             — configured integrations (source control, issue tracker, AI provider, triggers)
├── AI Provider Config          — default inference endpoint; per-workflow overrides
├── Workflow Library            — global templates + tenant-custom workflows
├── Task Library                — built-in tasks + tenant-custom tasks
├── Skill Library               — AI prompt templates (built-in + tenant-custom)
├── Repository Registry         — linked repos + quality gate config
├── Pipeline Runs               — isolated run history and step state
└── Billing Seat                — plan tier and usage metering
```

### 2.2 Tenant isolation guarantees

| Layer | Isolation mechanism |
|---|---|
| Database | Row Level Security with `tenant_id` on every tenant-scoped table |
| Secrets | Scoped paths in GCP Secret Manager: `tenants/{tenant_id}/plugins/{plugin}/{key}` |
| Execution | Runs on tenant's own GitHub/GitLab runners — platform never executes code on shared infra |
| UI | Each tenant sees only their own data — no cross-tenant API exposure |
| Region | Tenant data stored exclusively in their chosen regional deployment |

### 2.3 Roles and permissions

| Action | Admin | Engineer | Viewer |
|---|---|---|---|
| Configure plugins, billing, SSO, runner labels | ✓ | — | — |
| Invite/remove users, assign roles | ✓ | — | — |
| Publish workflows / tasks / skills | ✓ | — | — |
| Author workflows / tasks / skills (draft) | ✓ | ✓ | — |
| Trigger manual runs | ✓ | ✓ | — |
| Retry failed steps | ✓ | ✓ | — |
| Approve / reject human gates | Allowlist-based | Allowlist-based | — |
| View run trace, dashboard, audit log | ✓ | ✓ | ✓ |
| View compiled YAML | ✓ | ✓ | — |
| Rotate `AIDEVFLOW_TENANT_TOKEN` | ✓ | — | — |

Human gate approval is **not role-based** — it is controlled by a per-workflow `authorized_approvers` allowlist stored in the workflow YAML and managed via PR.

### 2.4 Billing tiers

| Tier | Target | Key limits |
|---|---|---|
| Trial | Evaluation | Time-limited (14 days), 1 repo, no SSO, community support |
| Starter | Small teams | 500 task executions/month, community support, multi-repo |
| Growth | Scaling teams | 5,000 task executions/month, SSO, priority support |
| Enterprise | Large orgs | Unlimited executions, SLA, custom region, dedicated support |

**Counting rule:** only billable task executions (AI calls, service calls) count against quota. Dry runs and retries after platform errors do not count.

---

## 3. System Architecture

*Senior Architect*

### 3.1 Architecture principle

The platform is a **compiler + coordination layer**. It generates CI/CD pipeline files, manages secrets centrally, tracks run state, and coordinates human gates. It never executes code on shared infrastructure. Actual execution always runs on tenant's own CI/CD runners.

### 3.2 GCP infrastructure (MVP — AU region first)

| Service | Role |
|---|---|
| Cloud Run | All platform services (api, webhook-receiver, compiler) — scale to zero |
| Cloud SQL (PostgreSQL 16) | State store — runs, steps, events, tenant config |
| Cloud Memorystore (Redis 7) | BullMQ job queue + ephemeral cache |
| GCP Secret Manager | All secrets — tenant tokens, plugin credentials, AI keys |
| GCP Pub/Sub | Internal event bus — webhook events, trigger routing, step completion signals |
| Artifact Registry | Container images |
| Cloud Build | CI/CD — build, test, push, deploy |
| Cloud Armor | WAF + DDoS protection on external-facing endpoints |
| VPC + Cloud NAT | Private networking: Cloud Run → Cloud SQL / Memorystore |
| IAM + Workload Identity | Service-to-service auth — no static service account keys |

**Cost reference (~50 tenants at launch):** ~$496/month. Cloud SQL is the dominant fixed cost. Cloud Run scales to zero for idle tenants.

### 3.3 Multi-region deployment

Regional deployments are fully independent stacks. Tenant data never crosses regions.

| Region | GCP location | MVP |
|---|---|---|
| `au` | australia-southeast1 | ✓ MVP |
| `eu` | europe-west1 | Post-MVP |
| `us` | us-central1 | Post-MVP |
| `ap` | asia-southeast1 | Post-MVP |

Region is chosen at signup and is immutable without a migration. The `AIDEVFLOW_TENANT_TOKEN` encodes the tenant's region. DNS-based routing directs requests to the correct regional Coordination API.

**Regional subdomains:** `api-au.aidevflow.io`, `api-eu.aidevflow.io`, `api-us.aidevflow.io`, `api-ap.aidevflow.io`

### 3.4 Service topology

```
Web UI (Next.js 15, global)
    │
    ▼ HTTPS (BFF — Next.js API routes)
Coordination API (Fastify, Cloud Run, regional)
    ├── GCP Secret Manager   — secrets retrieval (server-side only)
    ├── GCP Pub/Sub          — event publishing and subscription
    ├── Cloud SQL            — state persistence
    ├── Redis                — BullMQ queues + cache
    └── External APIs        — Jira, GitHub, GitLab (proxied on tenant's behalf)

Webhook Receiver (Fastify, Cloud Run, regional)
    └── GCP Pub/Sub          — validated events published here

Compiler (Node.js, Cloud Run, regional)
    └── Tenant repos         — compiled CI/CD YAML pushed via GitHub App / GitLab token

Runners (GitHub Actions / GitLab CI — tenant-owned)
    └── Coordination API     — called back for service_call, human_gate, pipeline tasks
```

### 3.5 Locked architecture decisions

These are finalized. Revisit only with documented justification:

- **GCP-first platform infra.** Cloud Run, Cloud SQL, Memorystore, Pub/Sub, Secret Manager.
- **Node.js only for all backend services.** Fastify 5 + TypeScript. No Go, no Python.
- **Terraform on GCP** for all IaC. Per-region modules.
- **Cloud Build + Artifact Registry + Cloud Run** for CI/CD and deployment.
- **No static service account keys.** Workload Identity Federation only.
- **PostgreSQL Row Level Security** enforced at DB layer — not application-only.
- **Secrets in GCP Secret Manager only.** Never in env vars, code, Terraform state, or logs.
- **Claude Code CLI in isolated containers per skill invocation.** Network egress restricted to named allowlist. Container destroyed after invocation.
- **Webhook HMAC-SHA256 verification** before any processing.
- **Auto-merge disabled** on all AI-generated PRs regardless of CI/quality gate status.

---

## 4. Plugin System

*Product Owner · Backend Lead*

Plugins are integration connectors configured per-tenant. Zero or more plugins can be enabled per category.

### 4.1 Source control & CI/CD target

| Plugin | Auth | Compiled output |
|---|---|---|
| **GitHub** | GitHub App (org-level install; 1-hour installation access tokens) | `.github/workflows/<name>.yml` |
| **GitLab** | GitLab Group Access Token (org-scoped, not tied to a person) | `.gitlab-ci.yml` |

**Why GitHub App over PAT:** GitHub App is installed on the org, not tied to an individual. Survives personnel changes. 3× higher API rate limits (15,000/hr vs. 5,000/hr). Fine-grained per-repo permissions. Preferred for multi-tenant SaaS.

**Why GitLab Group Access Token:** Same rationale — scoped to the group/org, independently rotatable.

### 4.2 Issue tracker

| Plugin | Config | MVP |
|---|---|---|
| **Jira** | API token, base URL, project key, webhook secret | ✓ MVP |
| Linear | API key, team ID | Post-MVP |
| GitHub Issues | Inherits from GitHub source control plugin | Post-MVP |

### 4.3 Event source / trigger

| Plugin | Config | MVP |
|---|---|---|
| **Jira assignment** | Jira webhook + system user account ID | ✓ MVP |
| **Webhook** | Auto-generated HTTPS endpoint per tenant (UUID in path), optional HMAC secret | ✓ MVP |
| **Manual / Web UI** | Always available | ✓ MVP |
| **Cron** | Schedule expression | Post-MVP |
| GCP Pub/Sub | GCP project ID, subscription name, service account JSON | Post-MVP |
| SonarQube / SonarCloud webhook | Webhook secret, SonarQube project key | Post-MVP |

### 4.4 AI provider

Two distinct concerns:

1. **Inference endpoint** — where the AI prompt goes. Configured via the AI provider plugin.
2. **Code editing agent** — the CLI (Claude Code, Codex) that edits files on the runner.

These stack; they are not alternatives.

| Plugin type | CLI compatible | Data residency |
|---|---|---|
| `anthropic` | ✓ Claude Code CLI | Anthropic US infra |
| `openai` | ✓ Codex CLI | OpenAI US infra |
| `azure_ai_foundry` | ✓ Both CLIs (`ANTHROPIC_BASE_URL` swap) | Pinned to Azure region — EU available |
| `aws_bedrock` | ✗ API mode only (SigV4 auth incompatible with CLIs) | Pinned to AWS region |
| `gcp_vertex_ai` | ✗ API mode only (OAuth2 service account auth incompatible) | Pinned to GCP region |

**AI provider resolution cascade:** Tenant default → Workflow override → Skill override.

### 4.5 Quality gate

| Plugin | Config |
|---|---|
| **SonarCloud** | Token, organisation, webhook secret |
| **SonarQube** | Server URL, token, webhook secret |
| **Semgrep** | API token (SaaS) or CI-native |

Quality gate plugin is associated at **repo registry level**, not workflow level. A `quality_gate` block on a repo registry entry automatically enables gate-checking for any workflow that pushes a branch to that repo.

### 4.6 Runner config

```yaml
runner_label:
  cli:     self-hosted-au-east   # jobs that clone repo and run AI CLI
  default: ubuntu-latest          # all other jobs (api, service_call, human_gate)
```

Both default to `ubuntu-latest` at signup. Three-level precedence: tenant admin default → workflow-level override → task-level override.

**Data residency warning at deploy time:** if `cli` tasks are present and `runner_label.cli` is still `ubuntu-latest`, a soft warning is shown (not a hard block).

---

## 5. Execution Model: Workflow → Task → Skill

*Backend Lead · Senior Architect*

### 5.1 Hierarchy

```
Workflow
  └── Task (ordered sequence; data flows between tasks via {{steps.id.output.field}})
        └── Skill (AI prompt execution unit — only for ai_skill tasks)
```

### 5.2 Workflow fields

Every workflow declares three structural fields validated at save time. Invalid combinations are rejected:

| `workflow_type` | Valid trigger types | Valid output types |
|---|---|---|
| `jira_report` | `gcp_pubsub`, `webhook`, `cron` | `jira_ticket` |
| `jira_analysis` | `jira_assignment` | `jira_comment` |
| `code_implementation` | `jira_assignment` | `pull_request` |
| `quality_gate` | `sonarcloud_gate_failed` | `none` |
| `pipeline` | `sub_pipeline_call` | `sub_pipeline_result` |

### 5.3 Task execution types

| Type | What it does | Compiles to |
|---|---|---|
| `runner` | Runs on a GitHub/GitLab runner; calls Coordination API for sub-steps | GitHub Actions job / GitLab CI job |
| `container` | Runs a specific Docker image with a command | Inline `docker run` inside a runner job |
| `pipeline` | Invokes another workflow as a sub-pipeline | Platform API call to trigger child workflow run |
| `ai_skill` | Installs AI CLI, runs compiled native skill file, commits and pushes | Claude CLI `.claude/skills/` or Codex CLI `.agents/skills/` |
| `human_gate` | Pauses workflow; sends notifications; resumes on approval or rejection | GitHub Environments approval / GitLab `when: manual` + Web UI gate |
| `service_call` | Calls an external API plugin (Jira, SonarQube, etc.) | Platform Coordination API call |

### 5.4 AI skill execution modes

| `execution` | File access | Iteration | Compatible inference endpoints |
|---|---|---|---|
| `cli` | Full filesystem (clone → edit → test → commit) | Yes — CLI iterates until tests pass or max-turns | `anthropic`, `openai`, `azure_ai_foundry` |
| `api` | None — structured JSON in/out | Limited — caller must re-invoke explicitly | Any endpoint including `aws_bedrock`, `gcp_vertex_ai` |

**When to use `cli`:** code editing tasks (`code-implementer`, `quality-gate-fixer`).  
**When to use `api`:** analysis and planning tasks (`ticket-analyst`, `implementation-planner`, `repo-identifier`).

### 5.5 Skill compilation

When a workflow is deployed, the platform compiles each skill YAML into the native skill file format of the configured AI provider and pushes it alongside the CI/CD pipeline file:

| AI provider | Native skill format | File path |
|---|---|---|
| Claude Code | Markdown + YAML frontmatter | `.claude/skills/<name>/SKILL.md` |
| Codex CLI | Markdown + YAML frontmatter | `.agents/skills/<name>/SKILL.md` |
| Any other | No native format — prompt injected inline | n/a |

### 5.6 Execution isolation (how it is achieved)

| Task type | Isolation mechanism | Provider |
|---|---|---|
| `runner` | Each job gets its own VM/container | GitHub/GitLab runner infra |
| `container` | Each Docker container is isolated | Docker runtime inside runner job |
| `pipeline` | Own isolated run context (run_id scoping) | Platform state store |
| `ai_skill` | Runs in its own runner job (isolated VM/container) | GitHub/GitLab runner infra |
| `human_gate` | No execution — waits for event | n/a |
| `service_call` | Stateless HTTPS call | External service |

The Coordination API is never in the critical path for AI execution. It only handles `service_call`, `human_gate`, and `pipeline` steps. Horizontal scaling of the API is sufficient for coordination traffic.

### 5.7 Output types and handlers

The `output` block declares what the workflow produces. The platform's output handler runs automatically after the last step:

| `output.type` | Required fields | What happens automatically |
|---|---|---|
| `jira_ticket` | `project_key`, optionally `assign_to` | Creates Jira ticket; assigns if set; adds repo remote link if service name known |
| `jira_comment` | `return_assignee` | Posts designated step's output as comment; reassigns ticket to `return_assignee` |
| `pull_request` | `repo_plugin` | Creates PR; posts PR URL as Jira comment on the associated ticket |
| `none` | — | No post-run action; run is recorded and marked complete |
| `sub_pipeline_result` | — | Returns outputs to parent workflow run |

**Critical invariant:** `return_assignee` is validated at **save time** (live Jira API check), not derived at runtime. The `previous_assignee` antipattern is eliminated — no runtime guessing of who to hand back to.

---

## 6. Workflow Chaining via Jira Assignment

*Product Owner · Backend Lead*

### 6.1 System users (bot accounts)

Each tenant configures Jira bot accounts mapped to workflows. When a ticket is assigned to a system user, the Jira webhook fires, the platform routes it to the mapped workflow, and the workflow runs.

**Provisioning paths:**
1. Select an existing Jira user from a platform-fetched dropdown
2. Platform creates a new Jira user via Jira API (requires admin-level Jira token)

### 6.2 Back-and-forth pattern

```
Ticket assigned to system user
  → Workflow fires: reads full Jira thread
  → AI processes → posts analysis/plan as comment
  → Workflow reassigns to human responsible for review
  → Human reads comment:
      Not satisfied → adds feedback comment → reassigns to SAME system user → loop
      Satisfied     → reassigns to NEXT system user → next workflow triggers
```

Every iteration re-reads the full comment thread — complete context is always available.

### 6.3 Reference workflow chain

```
[Trigger event]
      │ GCP Pub/Sub / webhook
      ▼
[report-error]           → creates Jira ticket → assigns to aidevflow-analyst
      │ Jira assignment
      ▼
[analyze-ticket]         ◄───────────────── human not satisfied (reassigns back)
  → AI analysis comment → assigns to human team lead
      │ human satisfied
      ▼
[analyze-implementation] ◄───────────────── human not satisfied (reassigns back)
  → AI implementation plan comment → assigns to human developer
      │ human satisfied
      ▼
[dev-implementation]
  → reads thread → clones repo → AI implements → commits → pushes branch → creates PR
  → posts PR link as Jira comment → assigns to developer for review
      │ branch pushed
      ▼
[quality-gate-fix]       ← triggered by SonarCloud webhook (decoupled from dev-implementation)
  → reads SonarCloud issues → AI fixes → pushes updated branch → SonarCloud re-analyses
  → loops up to max_fix_iterations → escalates to human on limit
```

---

## 7. Task Library (Built-in)

*Backend Lead*

### 7.1 Service call tasks

| Task | Plugin required | Description |
|---|---|---|
| `jira-read-thread` | jira | Reads ticket description + full comment history; returns structured JSON |
| `jira-create-ticket` | jira | Creates a Jira issue; returns ticket ID + URL |
| `jira-post-comment` | jira | Posts a comment on an existing issue |
| `jira-assign` | jira | Reassigns a Jira ticket to a user |
| `jira-transition-status` | jira | Transitions a Jira issue to a new status |
| `jira-add-remote-link` | jira | Adds a remote link (e.g. repo URL) to a Jira ticket |
| `sonarcloud-read-issues` | sonarcloud | Reads open issues for a project + branch from SonarCloud API |
| `sonar-quality-gate` | sonarcloud / sonarqube | Reads pass/fail from quality gate API |
| `pr-creator` | github / gitlab | Opens a pull request in the linked repo |
| `branch-creator` | github / gitlab | Creates a branch off main/develop |

### 7.2 AI skill tasks

| Task | Execution | Description |
|---|---|---|
| `code-implementer` | `cli` | Clones repo, runs AI CLI, edits files, runs tests, commits and pushes branch |
| `quality-gate-fixer` | `cli` | Reads SonarCloud issues, applies fixes via AI CLI, pushes |
| `test-implementer` | `cli` | Writes test code files into a branch via AI CLI |
| `ai-compose-jira-ticket` | `api` | Reads raw error/log, composes structured Jira ticket |
| `ticket-analyst` | `api` | Reads full Jira thread, proposes or refines a solution plan |
| `implementation-planner` | `api` | Reads full Jira thread, writes concrete implementation plan |
| `repo-identifier` | `api` | Identifies which repo is responsible given a ticket or error |
| `test-planner` | `api` | Analyses service, proposes test coverage strategy |
| `ci-yaml-generator` | `api` | Generates GitHub Actions or GitLab CI YAML from a plan |
| `intent-parser` | `api` | Parses natural language into a structured action |
| `workflow-generator` | `api` | Generates a new workflow YAML from a description |

### 7.3 Global workflow templates

| Template | Description |
|---|---|
| `gcp-error-fix` | GCP alert → Jira ticket → identify repo → AI fix → PR → review |
| `create-test-pipeline` | Analyse service → generate tests → generate CI YAML → PR |
| `dependency-upgrade` | Detect stale deps → AI upgrade PR → SonarQube gate → review |
| `security-patch` | CVE alert → identify affected services → AI patch → PR |
| `code-review-assist` | On PR open → AI reviews, posts inline comments → human merges |

---

## 8. Backend Services

*Backend Lead*

### 8.1 Services

**`apps/api` — Coordination API (Fastify 5)**

Responsibilities:
- Tenant state: pipeline runs, steps, events, human gates
- Secret proxying: reads from GCP Secret Manager; never returns raw values to clients
- Service proxying: Jira, GitHub, GitLab API calls on tenant's behalf
- Human gate endpoints: open, approve, reject, notify
- SSE endpoint: streams run trace events to the web frontend
- BullMQ job dispatch for async operations

**`apps/webhook-receiver` — Webhook Receiver (Fastify 5)**

Responsibilities:
- HMAC-SHA256 signature verification before any processing
- Parses Jira, GitHub, GitLab webhook payloads
- Publishes validated events to GCP Pub/Sub
- Returns 200 immediately after publish — no synchronous processing

**`apps/compiler` — Workflow Compiler (Node.js)**

Responsibilities:
- Reads workflow YAML from tenant repo or DB
- Compiles to GitHub Actions YAML or GitLab CI YAML
- Compiles skill YAMLs to native AI tool skill files
- Validates compiled output schema before returning
- Pushes compiled pipeline file + skill bundle to tenant repo via GitHub App / GitLab token
- Provisions required runner secrets via GitHub/GitLab API on deploy

### 8.2 Performance targets

| Endpoint type | p99 target |
|---|---|
| Read endpoints (list, detail) | < 200ms |
| Write endpoints (create, update) | < 500ms |
| SSE connection establishment | < 1s |
| Webhook ingestion (receive → Pub/Sub publish) | < 300ms |

### 8.3 Database schema (core tables)

```sql
-- One row per organisation
tenants (id, name, slug, region, plan, plan_status, trial_ends_at, billing_email,
         created_at, updated_at)

-- Plugin configurations per tenant
tenant_plugins (id, tenant_id, plugin_type, plugin_name, config JSON,
                secret_ref, enabled, created_at)

-- Platform users within a tenant
tenant_users (id, tenant_id, idp_user_id, role, email, mfa_enabled, created_at)

-- Jira bot accounts mapped to workflows
tenant_system_users (id, tenant_id, jira_account_id, display_name,
                     workflow_name, managed_by_platform, created_at)

-- Linked repos + quality gate config
tenant_repos (id, tenant_id, repo_url, plugin_name, tech_stack JSON,
              quality_gate JSON, created_at)

-- Workflow YAML definitions
tenant_workflows (id, tenant_id, name, version, workflow_type, yaml_content,
                  status, compiled_at, created_at, updated_at)

-- Pipeline run instances
pipeline_runs (id, tenant_id, workflow_id, trigger_type, trigger_payload JSON,
               status, dry_run, started_at, completed_at, created_at)

-- Step state per run
pipeline_run_steps (id, run_id, tenant_id, step_id, task_name, task_version,
                    status, inputs JSON, outputs JSON, error_message,
                    rendered_prompt, ai_response, job_url,
                    started_at, completed_at)

-- Append-only event log
pipeline_run_events (id, run_id, tenant_id, event_type, payload JSON, created_at)
```

**Schema rules enforced:**
- Every tenant-scoped table has `tenant_id` and a matching RLS policy
- Primary keys are UUIDs
- All timestamps in UTC
- Soft deletes (`deleted_at`) on pipeline runs, steps, and events — no hard deletes
- All foreign keys have explicit `onDelete` / `onUpdate` behaviour

### 8.4 Security requirements (non-negotiable)

1. **HMAC-SHA256 on every inbound webhook.** Constant-time comparison via `crypto.timingSafeEqual`. Failures return 401; nothing is published.
2. **Clerk JWT verification on every authenticated route.** `tenantId` and `userId` extracted from verified token — never from the request body.
3. **Secrets never returned to clients.** Coordination API proxies service calls server-side; raw tokens never appear in responses, logs, or error messages.
4. **Prompt injection defence.** Every `prompt_template` wraps untrusted inputs in explicit data boundaries (`--- BEGIN DATA ---` / `--- END DATA ---`). No string concatenation of untrusted input into instruction text.
5. **Claude subprocess output scanning.** Before passing AI CLI output downstream, scan for secret patterns (AWS keys, GCP keys, JWTs, API key formats). Match → redact, log, and abort the step.
6. **Input validation at every API boundary.** Zod schemas from `packages/types`. Return 400 with structured error body; never a raw Zod error object.
7. **PostgreSQL RLS as last line of defence.** Application also scopes all queries to `tenant_id`. RLS policies defined in migration files as raw SQL — never optional, never dropped.
8. **Minimum 2 reviewers** on PRs touching `skills/` or `workflows/` directories.

---

## 9. Frontend Application

*Frontend Lead*

### 9.1 Architecture principles

- **RSC first.** Default to React Server Components for any component that fetches data or has no interactivity. Add `"use client"` only when `useState`, `useEffect`, refs, or browser APIs are genuinely needed.
- **BFF pattern.** Browser calls Next.js API routes only. API routes call the Coordination API using the Clerk session token. Internal service URLs, `AIDEVFLOW_TENANT_TOKEN`, and secret references must never reach the browser.
- **Multi-tenancy from day one.** Every data fetch scoped to `organizationId` from Clerk. Use `auth()` in Server Components, `useOrganization()` in Client Components.
- **TanStack Query for mutable server state.** Workflow CRUD, plugin config, run actions, gate operations. RSC for read-only page-level data.
- **Zustand for client-only state.** UI preferences, wizard step state, unsaved form drafts. Keep slices small; derive computed values via selectors.

### 9.2 Route structure

```
app/
├── (marketing)/          # Public pages — SSR for SEO (pricing, landing, docs)
├── (auth)/               # Clerk sign-in / sign-up / org selection
└── (dashboard)/          # Authenticated tenant dashboard
    ├── layout.tsx         # Clerk auth check + org context
    ├── workflows/         # Workflow designer, list, detail, run trace
    ├── plugins/           # Plugin config (GitHub App, GitLab, Jira, AI provider)
    ├── repos/             # Repository registry
    ├── runs/              # Pipeline run history + live trace
    └── settings/          # Tenant settings, team, billing
```

### 9.3 UI surfaces (MVP)

#### Workflow Designer
- Visual list of workflow definitions + YAML editor
- Validates `workflow_type` + `trigger.type` + `output.type` consistency before save
- Shows compiler validation errors inline next to the relevant field
- "Preview compiled YAML" button (static compilation — no runner needed)
- Template library browser (global templates + tenant forks)
- Empty state: explains what workflows are, CTA to create first or adopt a template

#### Plugin Config
- Per-tenant forms for GitHub App install, GitLab token, Jira connection, AI provider credentials
- Progressive disclosure: show only required fields first; optional fields behind "Advanced" toggle
- Confirm before revoking a token — show which workflows depend on it
- Per-plugin connection test before saving
- Empty state: onboarding wizard with ordered steps (source control → issue tracker → AI provider)

#### Repository Registry
- Link repos to the tenant account
- Set quality gate config per repo (SonarCloud project key, max fix iterations)
- Associate tech stack tags
- Link Jira tickets to repos manually (for pre-existing tickets not created by the platform)
- Empty state: CTA to link first repo

#### Run Trace (real-time)
- Real-time pipeline run view using Server-Sent Events from Coordination API streamed via an API route
- Step list with status (`running`, `complete`, `failed`, `skipped`), duration, and key outputs
- Expandable step rows showing inputs, outputs, rendered prompt, AI response, job URL link
- Re-run controls: retry output handler, retry from step, re-trigger full run
- Must never require manual refresh — SSE updates automatically
- Loading state while SSE connection is being established (< 1s target)

#### Human Gate UI
- Approve / Reject buttons visible only on paused runs with an open gate
- Verify logged-in user is in the workflow's `authorized_approvers` list before enabling buttons
- Non-authorised users see disabled button with plain-language explanation (not a hidden button)
- Show who the authorised approvers are and whether any have already acted
- Notification delivered < 10s of workflow reaching the gate

#### Trial Banner & Billing
- Usage meter: task executions consumed / plan limit, with breakdown by workflow
- Trial expiry warnings: 7 days out (info banner), 3 days out (warning banner), expired (blocking modal with upgrade CTA)
- Upgrade flow to Growth / Enterprise
- Billing managed via Stripe integration (not in MVP UI scope — upgrade triggers Stripe-hosted checkout)

### 9.4 UX requirements (non-negotiable)

**Async feedback:**
- Any operation > 300ms needs a loading indicator
- Any operation > 3s needs a progress state (not just a spinner)
- After save/delete/approve: inline confirmation — not a silent state change
- Run Trace must never require manual refresh

**Empty states:** every list or dashboard view must define:
- First use: explain what goes here + CTA to create the first item
- No results from filter: distinguish "nothing matches filter" from "nothing exists"
- Error loading: explain what failed, offer retry, never show a blank screen

**Error messaging:**
- Never show raw API errors or stack traces to tenants
- Every error: (1) what went wrong in plain language, (2) why if known, (3) what the user can do
- Validation errors appear inline next to the field, not only as a top banner

**Security UX:**
- Confirm before revoking a plugin token — show which workflows depend on it
- Human Gate: visually clear when logged-in user is NOT an authorised approver
- Trial expiry warnings are progressive, not a single pop-up

**Accessibility:** WCAG 2.1 AA minimum. All interactive elements keyboard-accessible. shadcn/ui primitives (Radix-based) used for ARIA compliance. Test keyboard navigation before marking UI work done.

---

## 10. Run Debugging

*Product Owner · Frontend Lead · Backend Lead*

### 10.1 What the platform captures per step

Stored in `pipeline_run_steps`:
- Step status: `running` / `complete` / `failed` / `skipped`
- Inputs passed to the step
- Outputs returned
- Error message and stack trace if failed
- Start time and duration
- For `api` steps: rendered prompt + raw AI response
- For `cli` steps: structured summary (files read/edited, tests run, iterations, branch pushed)
- For `service_call` steps: HTTP response status and body from external service
- `job_url`: GitHub Actions / GitLab CI job URL for raw shell log access

### 10.2 Re-run options

| Scenario | Action |
|---|---|
| Output handler failed (e.g. GitHub API error) | Retry output handler only — no steps re-run |
| `api` step failed (e.g. AI API timeout) | Retry from that step with same inputs |
| `cli` step failed (e.g. tests didn't pass) | Re-trigger runner job from that step — new branch created |
| Whole run failed at trigger (bad webhook payload) | Re-trigger full run with corrected inputs from Web UI |

**Quota counting for re-runs:** retries after platform errors do not count against quota. Tenant-initiated re-runs from the Web UI do count.

### 10.3 Task testing (pre-publish)

Three levels of testing before a task or workflow goes live. All test runs are isolated — no real Jira tickets, repos, or PRs affected.

**Level 1 — YAML Preview** (always available)  
Static compilation: shows the compiled GitHub Actions / GitLab CI YAML without running anything. Available from Workflow Library → select workflow → "Preview compiled YAML".

**Level 2 — Task Test Panel** (`api` tasks only)  
Task Library detail view → Test tab. Fill in inputs, click Run. Platform calls AI inference endpoint and shows rendered prompt + parsed outputs inline. No runner needed.

**Level 3 — Workflow Dry Run** (all task types)  
Executes full workflow but skips the output handler. `cli` tasks clone and run but do not push. `service_call` tasks run read-only APIs but skip writes. Full step trace recorded and shown in Run Trace view.

---

## 11. MVP Scope

*Product Owner*

### 11.1 In scope

| Area | What is included |
|---|---|
| Auth & onboarding | Clerk auth, org creation, trial activation, plugin setup |
| Source control | GitHub App + GitLab Group Access Token |
| Issue tracker | Jira (API token, webhook, system user mapping) |
| Triggers | Jira assignment, webhook, manual Web UI |
| Workflow types | `jira_analysis` only |
| Task execution | `api` tasks only (no `cli`) |
| CI/CD compilation | GitLab CI + GitHub Actions |
| AI provider | Anthropic direct (MVP); Azure AI Foundry optional |
| UI surfaces | Workflow Designer, Plugin Config, Repo Registry, Run Trace, Human Gate UI, Trial Banner |
| Region | AU only (australia-southeast1, Sydney) |
| Billing tier | Trial tier (14-day limit) |
| Testing | Level 1 (YAML Preview) + Level 2 (Task Test Panel) |

### 11.2 Out of scope for MVP (post-MVP backlog)

| Item | Reason for deferral |
|---|---|
| `cli` tasks (`code-implementer`, `quality-gate-fixer`) | Runner infra complexity; no competitor pressure yet |
| SonarCloud webhook trigger | Depends on `cli` tasks (quality-gate-fix) |
| GCP Pub/Sub trigger | Less common trigger; adds GCP dependency for tenants |
| Cron trigger | Not needed for Jira-driven workflows |
| Workflow Dry Run (Level 3) | Requires `cli` runner setup |
| EU + US + AP regions | AU first; re-evaluate after 10+ EU/US/APAC signups |
| Linear plugin | Lower market priority |
| GitHub Issues plugin | Lower market priority |
| Skill compilation to SKILL.md | Needed only when `cli` tasks are in scope |
| Marketplace (skill/task publishing) | Post-launch feature |
| Docker Cloud runner add-on | Future paid add-on |
| SSO federation (Okta, Azure AD) | Enterprise tier — post-MVP |
| MFA enforcement for non-admin roles | Configurable at Enterprise tier |
| Per-tenant DB isolation (separate schemas) | Defer until tenant count justifies cost |
| Region migration tooling | Out of v1 scope entirely |

### 11.3 MVP acceptance criteria (key flows)

**Tenant onboarding**
- Given a new email address, When a tenant signs up, Then an org is created, a 14-day trial begins, and the plugin setup wizard is shown as the first screen
- Given trial expires, When the tenant next logs in, Then a blocking modal with upgrade CTA is shown and no other actions are available

**Plugin config (Jira)**
- Given a Jira API token and base URL, When the tenant saves the Jira plugin, Then a live connection test confirms the token is valid before saving
- Given an invalid token, When save is clicked, Then an inline error explains the failure and no secret is stored

**Workflow designer**
- Given a workflow YAML with an invalid `workflow_type` + `output.type` combination, When save is clicked, Then the platform rejects it with an inline error identifying the specific conflict
- Given `return_assignee` set to an email, When save is clicked, Then the platform performs a live Jira API check and rejects saving if the user does not exist in the tenant's Jira instance

**Jira assignment trigger**
- Given a Jira ticket assigned to a configured system user, When the Jira webhook fires, Then the webhook receiver validates HMAC, publishes to Pub/Sub, and returns 200 within 300ms
- Given an invalid HMAC, When the webhook fires, Then the receiver returns 401 and publishes nothing

**Workflow compilation**
- Given a valid `jira_analysis` workflow YAML, When compilation is triggered, Then the platform produces a valid GitHub Actions YAML and a valid GitLab CI YAML
- Given the compiled YAML is deployed, Then the platform pushes it to the tenant repo and provisions `AIDEVFLOW_TENANT_TOKEN` as a runner secret via the GitHub/GitLab API

**Run trace**
- Given a running pipeline run, When the Run Trace page is open, Then step statuses update in real time via SSE without a page refresh
- Given a step failure, Then the Run Trace shows the error in plain language and offers a retry action

**Human gate**
- Given a workflow reaches a `human_gate` step, When an authorised approver is logged in, Then the approve and reject buttons are enabled and a notification is delivered within 10 seconds
- Given a non-authorised user views the same gate, Then the buttons are disabled with a plain-language explanation of who can approve

---

## 12. Engineering Epics

*Product Owner · Senior Architect*

| Epic | Scope | MVP |
|---|---|---|
| Authentication & Onboarding | Clerk auth, org creation, trial activation, plugin setup wizard | ✓ |
| Plugin Configuration | GitHub App install, GitLab token, Jira connection, AI provider config | ✓ |
| Workflow Designer | YAML editor, schema validation, template library, version history | ✓ |
| Repository Registry | Repo linking, tech stack tagging, quality gate config | ✓ |
| Workflow Compilation | Workflow YAML → GitHub Actions + GitLab CI YAML; secret provisioning | ✓ |
| Webhook Receiver | HMAC verification, Jira webhook parsing, Pub/Sub publish | ✓ |
| Pipeline Execution & Run Trace | Trigger routing, step state, SSE, run debugging | ✓ |
| Human Gate | Approve/reject UI, authorised approver enforcement, Jira fallback | ✓ |
| Billing & Trial | Trial banner, usage metering, plan upgrade flow, Stripe integration | ✓ (trial tier) |
| Platform Infra & DevOps | Cloud Run, Terraform (AU — australia-southeast1), Cloud Build CI/CD, Secret Manager bootstrap | ✓ |
| Skill Library & Compilation | Built-in skills, native skill file compilation, per-AI-provider targets | Post-MVP |
| CLI Task Execution | `code-implementer`, `quality-gate-fixer`, runner job design | Post-MVP |
| Quality Gate Integration | SonarCloud webhook trigger, quality-gate-fix workflow | Post-MVP |
| Multi-Region | EU + US + AP regional deployments, region migration | Post-MVP |
| Enterprise Auth | SSO federation, MFA enforcement, custom IdP | Post-MVP |

---

## 13. Open Questions

*All roles*

The following questions require resolution before implementation of the relevant areas can begin:

1. **Stripe integration depth at MVP:** does Trial → paid upgrade flow require a Stripe-hosted checkout page, or just capture a credit card token? (Affects billing epic scope.)
2. **Jira system user creation:** does the MVP include platform-managed Jira bot account creation via the Jira API, or does the tenant always pick an existing user from a dropdown?
3. **Compiled YAML conflict resolution:** if a workflow is already deployed and the tenant edits and redeploys, the platform overwrites the CI/CD file and skill bundle in the repo. Does the platform open a PR for the update (reviewable) or commit directly to main via the bot account?
4. **Audit log retention policy:** how long are audit log entries retained per tier? (Affects Cloud SQL storage cost and the data model's `deleted_at` / archival strategy.)
5. **SSE authentication:** the Run Trace SSE endpoint is long-lived. Define the token refresh strategy — should the client re-authenticate mid-stream on Clerk token expiry, or should SSE use a short-lived signed URL issued at connection time?
6. **Webhook endpoint URL security:** auto-generated tenant webhook URLs include an unguessable UUID. Should the platform also enforce HMAC verification on all inbound webhooks even when the tenant did not configure a secret (optional HMAC)? Or is UUID-in-path sufficient for tenants who skip HMAC?