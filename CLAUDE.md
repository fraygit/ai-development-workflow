# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Default Skill: Project Manager

**Use `/project-manager` as the entry point for any task, question, or prompt in this repository.**

The project manager reads `docs/requirements.md` (the authoritative requirements document), classifies your request, checks MVP scope, surfaces relevant open questions, and routes to the correct specialist skill:

| Task type | Specialist skill |
|---|---|
| User stories, backlog, UX design, sprint ceremonies | `/product-owner` |
| Cloud architecture decisions, service selection, cost analysis, multi-cloud strategy | `/senior-architect` |
| Terraform HCL, Cloud Build, GitLab CI, GitHub Actions, Dockerfiles, GCP ops, monitoring | `/senior-devops` |
| Fastify, Prisma, BullMQ, backend services, DB schema | `/backend-lead` |
| Next.js, React, shadcn/ui, TanStack Query, Clerk frontend | `/frontend-lead` |
| Scope decisions, decomposition, open questions, cross-cutting features | handled directly |

For any multi-discipline feature (e.g. "implement the Human Gate"), the project manager decomposes it into per-skill tasks in dependency order and gives you the exact commands to run for each.

**If you are unsure which skill to use, start with `/project-manager <your request>`.**

---

## What This Repository Is

This is the **AIDevFlow** project — currently in the planning and architecture phase. The repository contains the design documents and Claude Code skills that define what will be built.

- `docs/idea-brain-storm.md` — original pipeline concept: wrapping Claude Code CLI into an event-driven workflow engine (GCP error → Jira ticket → AI fix → PR → merge)
- `docs/idea-brain-storm-saas.md` — evolution to a multi-tenant SaaS model
- `docs/architecture-design.md` — the authoritative architecture document: GCP infra, service breakdown, DB schema, cost model, security design, MVP scope

### Sub-projects (separate Git repos, gitignored here)

| Folder | Repo | Description |
|---|---|---|
| `ai-development-workflow-web/` | [fraygit/ai-development-workflow-web](https://github.com/fraygit/ai-development-workflow-web) | Next.js 15 frontend — tenant dashboard, workflow designer, human gate UI |
| `ai-development-workflow-poc/` | [fraygit/ai-development-workflow-poc](https://github.com/fraygit/ai-development-workflow-poc) | POC — GitHub Actions + Claude Code CLI + Jira webhook, no platform infrastructure |
| *(remote only — not cloned locally)* | [fraygit/ai-development-workflow-sample-nodejs](https://github.com/fraygit/ai-development-workflow-sample-nodejs) | POC target app — Hello World Node.js app that the POC workflow clones and modifies |

The remaining monorepo services (`apps/api`, `apps/webhook-receiver`, `apps/compiler`, `packages/*`, `infra/`) will be added as additional sub-project repos as implementation begins, following the Turborepo structure defined in `docs/architecture-design.md`.

## What AIDevFlow Does

AIDevFlow is a **compiler + coordination layer** for AI-powered software development workflows. It:

1. Lets tenants design workflows in YAML
2. Compiles those workflows into GitHub Actions or GitLab CI pipeline YAML
3. Pushes compiled pipelines to tenant repos via GitHub App / GitLab tokens
4. Coordinates execution: persists step results, proxies service calls (Jira, SonarCloud, GitHub), manages human approval gates
5. Never executes code itself — execution always runs on the tenant's own CI/CD runners

The AI execution engine is **Claude Code CLI**, invoked as a controlled subprocess per skill, sandboxed in an isolated container.

## Planned Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Next.js 15 (TypeScript), shadcn/ui, Tailwind CSS, TanStack Query, Zustand |
| All backend services | Node.js + Fastify 5 (TypeScript) — single language across the entire monorepo |
| ORM | Prisma 6 (PostgreSQL) |
| Job queue | BullMQ (Redis-backed) |
| Auth | Clerk (OIDC/SSO, MFA, Next.js SDK) |
| Database | PostgreSQL 16 (Cloud SQL) with Row Level Security |
| Cache / queue backend | Redis 7 (Cloud Memorystore) |
| Secret store | GCP Secret Manager |
| Event bus | GCP Cloud Pub/Sub |
| Monorepo tooling | Turborepo |
| Infra | Terraform on GCP |
| CI/CD | Cloud Build + Artifact Registry + Cloud Run |

**Node.js is the only backend language.** Go was considered for the Webhook Receiver but rejected to preserve shared TypeScript types, shared Prisma client, and a single Dockerfile pattern across all services.

## Project Structure

The platform is split across multiple Git repositories. The web frontend has already been extracted into its own repo; remaining services will follow as implementation begins.

### Repos

| Folder (local) | GitHub Repo | Status |
|---|---|---|
| `ai-development-workflow-web/` | [fraygit/ai-development-workflow-web](https://github.com/fraygit/ai-development-workflow-web) | Active — Next.js 15 frontend |
| `ai-development-workflow-poc/` | [fraygit/ai-development-workflow-poc](https://github.com/fraygit/ai-development-workflow-poc) | Active — concept POC (GitHub Actions + Claude Code CLI) |
| *(remote only)* | [fraygit/ai-development-workflow-sample-nodejs](https://github.com/fraygit/ai-development-workflow-sample-nodejs) | Active — POC target app (cloned by the workflow runner, not locally) |
| *(planning repo root)* | fraygit/ai-development-workflow | Active — design docs + Claude Code skills |
| `aidevflow/apps/api/` | *(planned)* | Not started — Fastify Coordination API |
| `aidevflow/apps/webhook-receiver/` | *(planned)* | Not started — Fastify webhook receiver |
| `aidevflow/apps/compiler/` | *(planned)* | Not started — workflow YAML compiler |

### Web App Structure (`ai-development-workflow-web/`)

```
ai-development-workflow-web/          # standalone repo — gitignored in planning repo
├── app/
│   ├── (marketing)/                  # Public pages — SSR for SEO
│   ├── (auth)/                       # Clerk sign-in / sign-up / org selection
│   └── (dashboard)/                  # Authenticated tenant dashboard
│       ├── workflows/                # Workflow designer, list, detail, run trace
│       ├── plugins/                  # Plugin config (GitHub App, GitLab, Jira, AI provider)
│       ├── repos/                    # Repository registry
│       ├── runs/                     # Pipeline run history + live trace
│       └── settings/                 # Tenant settings, team, billing
├── components/
│   ├── ui/                           # shadcn/ui primitives
│   └── [feature]/                    # Feature-specific components
├── lib/
│   ├── api-client.ts
│   ├── query-client.ts
│   └── auth.ts
├── middleware.ts
└── next.config.ts
```

### Remaining Services (planned — single monorepo `aidevflow/`)

```
aidevflow/
├── apps/
│   ├── api/                    # Fastify — Coordination API (state, secrets, service proxying, human gates)
│   ├── webhook-receiver/       # Fastify — validates inbound webhooks, publishes to Pub/Sub
│   └── compiler/               # Node.js — compiles workflow YAML → GitHub Actions / GitLab CI YAML
├── packages/
│   ├── types/                  # Shared TypeScript types (workflow, task, skill schemas)
│   ├── db/                     # Prisma schema + migrations
│   ├── utils/                  # Shared Node.js utilities: hmac.ts, secret-manager.ts, pubsub.ts, redis.ts, logger.ts
│   ├── skill-library/          # Built-in skill YAML files (versioned)
│   └── workflow-templates/     # Global workflow templates (versioned)
└── infra/
    └── terraform/              # GCP resources — per-region modules (us-central1, europe-west1, asia-southeast1)
```

## Key Architectural Decisions (Already Locked)

- **Skill interface:** versioned YAML files co-located in the pipeline repo (`skills/<name>/<semver>.yaml`). Semver rules: patch = prompt tweak, minor = new optional input, major = breaking schema change.
- **Workflow definition format:** YAML files stored in the pipeline repo. Global templates + per-repo overrides.
- **Human gate primary channel:** Web UI dashboard (approve/reject buttons). Fallback: Jira comment trigger words (`!approve` / `!reject`) delivered via Jira webhook → Cloud Function → Pub/Sub → pipeline engine (no polling for trigger events — polling is for reminders/escalation only).
- **State store:** relational DB (Postgres-first). Key tables: `pipeline_runs`, `pipeline_run_steps`, `pipeline_run_events`, `task_executions`, `repo_skill_manifests`.
- **Secret management:** GCP Secret Manager only. Never in env vars, code, or logs.
- **Quality gate:** tool-agnostic `quality-gate-checker` skill; SonarQube/SonarCloud default, Semgrep alternative. Pipeline reads pass/fail only — all rule config stays in the tool.
- **Region routing:** tenant's region encoded in `AIDEVFLOW_TENANT_TOKEN`. Regional subdomains (`api-eu.aidevflow.io`, `api-us.aidevflow.io`) are the preferred routing approach.
- **AI execution:** Claude Code CLI in an isolated container per skill invocation. Network egress restricted to a named allowlist. Container destroyed after invocation.
- **Prompt injection:** every skill `prompt_template` wraps untrusted inputs in explicit data boundaries (`--- BEGIN ERROR DATA ---` / `--- END ERROR DATA ---`). Never string-concatenate untrusted input into instruction text.
- **Auto-merge disabled** on all AI-generated PRs regardless of CI/SonarQube status.

## MVP Scope (3–4 months)

Target: **GitLab + Jira** first (no well-funded competitor there vs. GitHub + Jira).

Included: Web UI auth + plugin config, Jira assignment trigger, `jira_analysis` workflow type, `api` execution tasks only (no `cli`), GitLab CI + GitHub Actions compilation, Run Trace UI, EU region, Trial tier.

Excluded from MVP: `cli` tasks (code-implementer, quality-gate-fixer), SonarCloud webhook, GCP Pub/Sub trigger, skill compilation to SKILL.md, dry run, US + AP regions.

## Security Constraints to Enforce

1. Authorised approver allowlist (Jira `accountId` + Web UI user UUID) on every human gate — stored in workflow YAML, managed via PR.
2. Webhook HMAC-SHA256 signature verification before any processing.
3. PostgreSQL Row Level Security enforced at DB level.
4. SHA-256 hash verification of skill files at load time.
5. Minimum 2 reviewers on PRs touching `skills/` or `workflows/` directories.
6. Claude subprocess output scanned for secret patterns before passing downstream.

## Epic & Task Tracking

Implementation work is tracked in `docs/epic/` as a two-level markdown hierarchy.

### Structure

```
docs/epic/
└── <epic-name>/
    ├── 0.0-task-breakdown.md       # Master index — all tasks with status table
    ├── <story>.0-<story-title>.md  # Story definition (acceptance criteria, notes)
    ├── <story>.<task>-<title>.md   # Individual task (skill, priority, AC, notes)
    └── ...
```

**Active epics:**

| Epic | Index | Stories |
|---|---|---|
| Authentication & Onboarding | [docs/epic/authentication-onboarding/0.0-task-breakdown.md](docs/epic/authentication-onboarding/0.0-task-breakdown.md) | 1.0–6.0 (27 tasks) |
| POC | [docs/epic/poc/0.0-task-breakdown.md](docs/epic/poc/0.0-task-breakdown.md) | 0.0–2.0 (6 tasks) — proves core concept with GitHub Actions + Claude Code CLI |

### Task file naming convention

- `0.0-task-breakdown.md` — master index for the epic (always present)
- `<N>.0-<title>.md` — story definition file (e.g. `1.0-tenant-signup-and-org-creation.md`)
- `<N>.<M>-<title>.md` — task file (e.g. `1.1-local-dev-environment-setup.md`)

### Status tracking

Task status is tracked in the `0.0-task-breakdown.md` index table in the **Status** column:

| Symbol | Meaning |
|---|---|
| `—` | Not started |
| `In Progress` | Currently being worked on |
| `Done` | Completed |
| `Blocked` | Blocked — reason noted in the task file |
| `Deferred` | Deferred to post-MVP or later sprint |

When a task is completed, the project-manager (or specialist skill) must update:
1. The **Status** column in `0.0-task-breakdown.md`.
2. The acceptance criteria checkboxes (`- [x]`) in the individual task file.

### Pivots and scope changes

If requirements change or a task needs to be revised:
1. Update the individual task file with the new AC and a `> **Pivoted:** reason` note.
2. Update `0.0-task-breakdown.md` if the title or skill assignment changes.
3. Note the change in the story `.0` file under a `## Change Log` section if it affects acceptance criteria.

---

## File Operation Permissions

**Edit and Write operations within `c:\Projects\Git\ai-development-workflow\` are pre-approved.** Claude Code will auto-approve all file edits and new file creation in this repository directory without prompting. This is configured in `.claude/settings.local.json`.

---

## Cost Reference

At launch (~50 tenants): ~$496/month GCP infra. Dominant cost is Cloud SQL (fixed per region). Cloud Run scales to zero for idle tenants. See `docs/architecture-design.md` for full breakdown and scale projections.