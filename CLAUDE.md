# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

This is the **AIDevFlow** project — currently in the planning and architecture phase. There is no source code yet. The repository contains the design documents that define what will be built.

- `docs/idea-brain-storm.md` — original pipeline concept: wrapping Claude Code CLI into an event-driven workflow engine (GCP error → Jira ticket → AI fix → PR → merge)
- `docs/idea-brain-storm-saas.md` — evolution to a multi-tenant SaaS model
- `docs/architecture-design.md` — the authoritative architecture document: GCP infra, service breakdown, DB schema, cost model, security design, MVP scope

When code implementation begins, the monorepo will live at `aidevflow/` following the Turborepo structure defined in `docs/architecture-design.md`.

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

## Planned Monorepo Structure

```
aidevflow/
├── apps/
│   ├── web/                    # Next.js 15 — tenant dashboard, workflow designer, human gate UI
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

## Cost Reference

At launch (~50 tenants): ~$496/month GCP infra. Dominant cost is Cloud SQL (fixed per region). Cloud Run scales to zero for idle tenants. See `docs/architecture-design.md` for full breakdown and scale projections.