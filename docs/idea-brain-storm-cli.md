# AIDevFlow CLI — Open-Source Brainstorm

## The Pivot Case

The SaaS model requires significant upfront infrastructure — multi-tenant auth, Clerk, Cloud Run, Cloud SQL, Pub/Sub, Cloud Functions, a Web UI — before a single user gets value. The CLI pivot inverts this: ship the orchestration value first, earn adoption, and add a hosted layer later if warranted.

The GitHub CLI → GitHub model is the precedent: the CLI is OSS and beloved; the platform adds team and UI features on top of a proven core.

---

## Core Architecture: The CLI Lives Inside the Pipeline

This is the primary design insight. The CLI is not a tool that *generates* CI/CD pipelines — it is the orchestration brain that *runs inside* CI/CD pipelines.

### Before (original SaaS design)
```
Workflow YAML
     │
     ▼
Compiler service → generates GitHub Actions YAML → pushed to repo
     │
     ▼
GitHub Actions runs → calls back to SaaS coordination API for gates and state
```

### After (CLI-in-pipeline design)
```
GitHub Actions (thin trigger: checkout → install CLI → aidevflow run <workflow>)
     │
     ▼
CLI reads .aidevflow/workflows/<name>.yaml directly
     │
     ▼
CLI orchestrates: Jira API → Claude Code → git → PR → Jira comment
     │
     ▼
State written to .aidevflow/runs/<run-id>.json
```

The compiler disappears. The SaaS coordination API disappears. The CI platform provides compute; the CLI provides intelligence.

---

## Repo Structure

Every team repo that uses AIDevFlow gets a `.aidevflow/` directory committed alongside their code:

```
my-repo/
├── .aidevflow/
│   ├── config.yaml                  # root config: repo metadata, integration pointers, AI provider
│   └── workflows/
│       ├── jira-fix.yaml            # full loop: fetch ticket → analyze → gate → implement → PR
│       ├── code-review.yaml         # fetch PR diff → AI review → post comments
│       └── post-merge-test.yaml     # run tests after merge, reopen ticket on failure
├── .github/
│   └── workflows/
│       └── aidevflow.yml            # thin bootstrap — same file in every repo, never edited
├── skills/                          # optional: local skill overrides (fallback to community registry)
│   └── my-custom-skill/
│       └── 1.0.0.yaml
└── src/
    └── ...
```

`aidevflow init` scaffolds all of this in one command. Teams never write the `.github/workflows/aidevflow.yml` bootstrap manually.

---

## Root Config (`aidevflow/config.yaml`)

```yaml
repo:
  name: my-service
  main_branch: main
  tech_stack: [nodejs, postgres]

jira:
  project_key: PROJ
  base_url: https://myorg.atlassian.net
  token_env: JIRA_TOKEN              # env var name — value lives in GitHub Secrets

git:
  remote: https://github.com/myorg/my-service
  credentials_env: GITHUB_TOKEN

ai:
  provider: claude_code              # claude_code | codex
  api_key_env: ANTHROPIC_API_KEY

skill_registry: https://github.com/aidevflow/skills   # community registry (default)
```

Secret values are never in this file — only the env var name that holds them. In CI, those env vars come from GitHub Secrets or GitLab CI Variables. Locally, they come from `.env` (gitignored).

---

## Thin CI Trigger (`.github/workflows/aidevflow.yml`)

This file is identical across every repo. `aidevflow init` generates it. Teams never edit it — all logic lives in `.aidevflow/workflows/`.

```yaml
name: AIDevFlow
on:
  workflow_dispatch:
    inputs:
      workflow:
        description: Workflow name (matches .aidevflow/workflows/<name>.yaml)
        required: true
        default: jira-fix
      ticket:
        description: Jira ticket ID (e.g. PROJ-123)
        required: false
      run_id:
        description: Run ID to resume (leave blank for new run)
        required: false

jobs:
  run:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm install -g aidevflow
      - name: Run or resume workflow
        run: |
          if [ -n "${{ inputs.run_id }}" ]; then
            aidevflow resume ${{ inputs.run_id }}
          else
            aidevflow run ${{ inputs.workflow }} --ticket ${{ inputs.ticket }}
          fi
        env:
          JIRA_TOKEN: ${{ secrets.JIRA_TOKEN }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          AI_API_KEY: ${{ secrets.AI_API_KEY }}       # OpenAI or Anthropic key
          AIDEVFLOW_MODEL: ${{ vars.AIDEVFLOW_MODEL }} # e.g. openai or anthropic
```

GitLab CI equivalent uses the same `aidevflow run` command; only the wrapper syntax differs.

---

## Workflow YAML (`.aidevflow/workflows/jira-fix.yaml`)

The workflow YAML is interpreted directly by the CLI at runtime. No compilation step.

```yaml
name: jira-fix
version: 1.0.0
description: Full loop from Jira ticket to merged PR.

inputs:
  - name: ticket
    type: string
    required: true

steps:
  - name: fetch-ticket
    type: jira
    action: get_ticket
    params:
      ticket_id: "{{ticket}}"
    outputs:
      - ticket_data

  - name: analyze
    type: skill
    skill: jira-analysis@2.1.0
    inputs:
      jira_ticket_id: "{{ticket}}"
      error_details: "{{steps.fetch-ticket.ticket_data.description}}"
    outputs:
      - proposed_solution

  - name: human-gate-proposal
    type: human_gate
    message: "Proposed solution for {{ticket}} is ready for review."
    channels:
      - jira_comment                 # posts solution to ticket, polls for !approve / !reject
    on_approve: continue
    on_reject: goto:analyze          # loop back with feedback

  - name: implement
    type: skill
    skill: code-implementer@1.0.0
    inputs:
      solution: "{{steps.analyze.proposed_solution}}"
      branch: "fix/{{ticket}}"

  - name: create-pr
    type: skill
    skill: pr-creator@1.0.0
    inputs:
      ticket_id: "{{ticket}}"
      branch: "fix/{{ticket}}"

  - name: notify-complete
    type: jira
    action: add_comment
    params:
      ticket_id: "{{ticket}}"
      body: "PR created: {{steps.create-pr.pr_url}}"
```

---

## Human Gates — State + Resume Pattern

CI runners have execution time limits (GitHub Actions: 6 hours; GitLab CI: similar). A human gate waiting 24 hours for approval cannot block a live runner.

### Solution: state snapshot + resume

**Phase 1 — Run until the gate:**
```
aidevflow run jira-fix --ticket PROJ-123
  → executes steps: fetch-ticket, analyze
  → hits human-gate-proposal step
  → posts proposed solution as Jira comment
  → writes .aidevflow/runs/run-abc123.json (full state snapshot)
  → commits state file to a dedicated branch or uploads as a GitHub artifact
  → exits cleanly (exit 0)
Runner completes. No timeout.
```

**State file** (`.aidevflow/runs/run-abc123.json`):
```json
{
  "run_id": "run-abc123",
  "workflow": "jira-fix",
  "inputs": { "ticket": "PROJ-123" },
  "status": "waiting_on_human",
  "current_step": "human-gate-proposal",
  "iteration": 1,
  "completed_steps": {
    "fetch-ticket": { "ticket_data": { "..." } },
    "analyze": { "proposed_solution": "..." }
  },
  "created_at": "2026-05-11T10:00:00Z",
  "gate_posted_at": "2026-05-11T10:02:00Z"
}
```

**Phase 2 — Human reviews and approves:**

Human posts `!approve` on the Jira ticket.

Jira fires a webhook → thin forwarder → triggers `workflow_dispatch` on the repo with `run_id=run-abc123`.

(Or: team member manually triggers `workflow_dispatch` with the run ID — no webhook infrastructure needed for early adoption.)

**Phase 3 — Resume:**
```
aidevflow resume run-abc123
  → reads .aidevflow/runs/run-abc123.json
  → validates the !approve came from an authorised reviewer
  → continues from implement step
  → creates PR
  → posts completion comment on Jira
  → marks run complete in state file
Runner completes.
```

### Human gate channels

| Channel | How it works | Infrastructure needed |
|---|---|---|
| `jira_comment` | Posts to Jira, polls for `!approve` / `!reject` during a short watch window after resume | None beyond Jira token |
| `terminal` | Blocks with a `[y/n]` prompt — local runs only | None |
| `github_issue` | Posts gate as a GitHub issue comment, waits for `/approve` label | None beyond GitHub token |
| `slack` *(future)* | Posts to Slack channel, waits for emoji reaction | Slack token |

---

## Key CLI Commands

```
aidevflow init                           # scaffold .aidevflow/ config and the thin CI trigger YAML
aidevflow run <workflow>                 # run a workflow defined in .aidevflow/workflows/<name>.yaml
  --ticket PROJ-123                      # pass workflow inputs as flags
  --dry-run                              # preview all steps — no API calls, no commits
aidevflow resume <run-id>               # resume a run that paused at a human gate
aidevflow gate approve <run-id>         # approve a pending gate from the terminal
aidevflow gate reject <run-id>          # reject with optional --feedback "reason"
aidevflow status                         # list active and recent runs with current step and status
aidevflow run-trace <run-id>            # full step-by-step trace of a specific run
aidevflow skill add <name>@<version>    # install a skill from the community registry
aidevflow skill list                     # list installed skills and versions
aidevflow skill upgrade <name>          # upgrade a skill to the latest compatible version
aidevflow watch                          # poll Jira for new assignments and auto-trigger runs
  --jira-project PROJ                    # filter to a specific Jira project
  --interval 60                          # polling interval in seconds (default: 60)
```

`aidevflow compile` is intentionally absent — there is no compilation step. The CLI interprets workflow YAML directly at runtime.

---

## State Management

| Scope | Storage | Notes |
|---|---|---|
| Local / single developer | `.aidevflow/runs/*.json` files | Default. No server. |
| Team in CI | GitHub artifact or a dedicated `aidevflow-state` branch | State survives across runner invocations |
| Self-hosted team | SQLite or Postgres (`AIDEVFLOW_STATE_BACKEND=sqlite://` or `postgres://`) | Queryable run history |
| Managed platform (future) | `AIDEVFLOW_STATE_BACKEND=aidevflow://` | Web UI, cross-repo visibility, always-on triggers |

The env var `AIDEVFLOW_STATE_BACKEND` controls which backend the CLI uses. Same commands, same workflow YAMLs regardless of backend.

---

## Codex as the Standard Runner

Codex CLI is the single agent runner. It has built-in support for multiple model providers — OpenAI models and Anthropic Claude models are both configurable via `codex config.toml` or environment variables. The skill YAML has no adapter section. The component always invokes `codex`; which model runs is an org-level or repo-level variable.

### Provider configuration (done once per org, not per skill)

```toml
# codex config.toml — committed to the repo or set via CI env vars
[model_providers.anthropic]
base_url = "https://api.anthropic.com/v1/"
env_key  = "AI_API_KEY"
```

Teams using OpenAI point at `api.openai.com`. Teams using Anthropic point at `api.anthropic.com`. The skill YAML is identical either way.

### Skill YAML — no adapters section

```yaml
name: jira-analysis
version: 2.1.0
description: Analyzes a Jira ticket and proposes a solution approach.

inputs:
  - name: ticket_id
    type: string
    required: true
  - name: description
    type: string
    required: true
  - name: feedback
    type: string
    required: false
outputs:
  - name: proposed_solution
    type: string

ai:
  model_hint: coding-large      # abstract: coding-large | coding-fast | general
  max_tokens: 1500              # component resolves hint to actual model at runtime

prompt_template: |
  You are a senior software engineer. Analyze the following Jira ticket and propose a clear,
  minimal solution approach. Focus on root cause, not symptoms.

  Ticket: {{ticket_id}}

  --- BEGIN TICKET DATA ---
  {{description}}
  --- END TICKET DATA ---

  {% if feedback %}
  --- BEGIN HUMAN FEEDBACK ---
  {{feedback}}
  --- END HUMAN FEEDBACK ---
  Revise your proposal to address this feedback.
  {% endif %}

  Respond with: root cause, proposed approach (1-3 sentences), affected files, and risks.
```

No `adapters:` block. The component resolves `model_hint: coding-large` to `gpt-5.3-codex` for OpenAI teams or `claude-sonnet-4-6` for Anthropic teams, based on the org-level `AIDEVFLOW_MODEL` variable.

### Why not Claude via Azure AI Foundry?

Azure AI Foundry's Claude endpoints use Anthropic's Messages API format — they do not expose an OpenAI-compatible endpoint. Codex CLI cannot call them as an OpenAI-compatible provider. Anthropic's own OpenAI-compatibility shim is explicitly testing-only (missing extended thinking, prompt caching). The clean path is Codex CLI → Anthropic API directly, not Codex → Foundry → Claude.

### Model resolution

| `AIDEVFLOW_MODEL` var | `model_hint: coding-large` resolves to | `model_hint: coding-fast` resolves to |
|---|---|---|
| `openai` | `gpt-5.3-codex` | `gpt-5.4-mini` |
| `anthropic` | `claude-sonnet-4-6` | `claude-haiku-4-5` |
| Explicit model ID | used directly | used directly |

### Jira and GitHub integration

With Codex as the runner, Jira and GitHub integrations are REST API calls made by the component itself — not MCP. The component fetches the ticket, renders the skill prompt with the ticket data, invokes `codex`, and handles the Jira comment and PR creation after Codex finishes. This is slightly more component code but is fully provider-agnostic and requires no MCP server setup.

---

## What the CLI-in-Pipeline Design Eliminates (vs SaaS)

| SaaS component | Status |
|---|---|
| Workflow compiler service | **Gone** — CLI interprets YAML directly at runtime |
| Coordination API (Fastify) | **Gone** — state lives in files / artifacts |
| GCP Pub/Sub | **Gone** — GitHub Actions events are the message bus |
| GCP Cloud Functions (webhook receiver) | **Simplified** — optional thin webhook→`workflow_dispatch` forwarder only |
| GCP Secret Manager | **Replaced** — GitHub Secrets / GitLab CI Variables |
| GCP Cloud Run | **Replaced** — GitHub-hosted or self-hosted runners |
| Cloud SQL | **Replaced** — JSON state files (or self-hosted SQLite/Postgres for teams) |
| Clerk / multi-tenant auth | **Gone** — no platform to authenticate into |

What remains relevant for a future managed SaaS layer on top:
- Web UI for human gates and run traces (cross-repo visibility)
- Always-on Jira webhook → `workflow_dispatch` bridge (real-time triggers without polling)
- Managed state backend
- RBAC, audit logs, team collaboration features

---

## OSS CLI → Managed Platform Upgrade Path

| Layer | Form factor | Who runs it |
|---|---|---|
| Core engine | OSS CLI (NPM package) | Individual developer, locally or in CI |
| Team state | `aidevflow-state` branch or self-hosted Postgres | Team — no extra infra beyond what they have |
| Managed platform | Hosted AIDevFlow (SaaS) | Platform — adds Web UI, real-time triggers, cross-repo visibility |

The managed platform is the backend, not a replacement for the CLI. `AIDEVFLOW_STATE_BACKEND=aidevflow://app.aidevflow.io` in the config is the only change needed to graduate from self-hosted to managed.

---

## Open Questions

| Question | Options | Leaning |
|---|---|---|
| Packaging | NPM (`npx aidevflow`), Homebrew, single binary | NPM — lowest friction for Node.js dev teams |
| Skill registry | GitHub repo (community PRs), NPM packages | GitHub repo first — zero infra, community-native |
| State in CI | GitHub artifact, dedicated branch, or GitHub Gist | Artifact — cleanest, no branch pollution |
| Resume trigger | Manual `workflow_dispatch`, Jira webhook→forwarder, `aidevflow watch` | All three supported; webhook forwarder is optional |
| Human gate timeout in CI | Poll Jira within a long-running runner, or exit+resume | Exit+resume — avoids runner time limits entirely |
| `aidevflow watch` persistence | Foreground process, systemd service, Docker container | Document all three; user chooses |
| Codex MCP support | Wait for OpenAI, or REST fallback | REST fallback today; switch when Codex ships MCP |
| OSS license | MIT, Apache 2.0 | Apache 2.0 — patent clause protects contributors |
| First workflow to ship | `jira-fix` end-to-end | Yes — proves the whole concept in one command |

---

## What This Is Not

- **Not a CI/CD runner**: the CLI does not build or test code. It orchestrates AI agents and service calls inside an existing runner (GitHub Actions, GitLab CI, local terminal).
- **Not a replacement for Codex CLI**: the aidevflow components wrap Codex CLI as the AI execution engine. Codex handles file editing and code generation; aidevflow handles Jira integration, orchestration, and the approval loop.
- **Not a YAML compiler**: the CLI interprets workflow YAML at runtime. It does not generate GitHub Actions or GitLab CI YAML — the thin CI bootstrap is a one-time scaffold, not a compilation output.
- **Not another Zapier**: this is AI-powered code generation and PR automation for software development teams. It is not a general-purpose automation platform.
