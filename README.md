# QA Playbook

Templates, skills, and agents for QA at AI-first SaaS products — async job pipelines, LLM wrappers, billing.

Focused on the scenario where you are the **first QA on the team**, starting from scratch with no existing tests or processes.

---

## What's inside

### `qa-onboarding-template.md`

A practical onboarding checklist for the first QA on a project without tests or processes. Covers:

- **Access requests** — staging environments, Jira, TMS, Swagger, CI/CD, logs, queues, DB
- **Conversations with product and dev** — what breaks most, what can't break, what's shipping soon
- **Exploratory testing** — how to use the "fresh eyes" window before it closes
- **Risk mapping** — a simple table format: area → risk → priority → automation status
- **Documentation** — what to ask the team for, what to create first
- **Test cases and checklists** — checklist + comment format (not step-by-step), smoke vs regression split
- **Automation roadmap** — phased plan, CI integration, async pipeline helpers

Each section has an **AI-first block** with specifics for async generation flows:
- `submit_job → poll_status → assert_result` pattern
- LLM provider failures, timeouts, retry handling
- Queue saturation, job state transitions
- Acceptance criteria for generated output (format, size, time)

### `resources.md`

Curated list of QA tools — updated periodically:
- LLM / AI output testing (deepeval, promptfoo, llmtest)
- Async pipeline testing (tracetest)
- API testing (stepci, api-automation-agent)
- Load testing (k6)
- Curated indexes and references

---

## Claude Code skills

Slash command skills — invokable with `/skill-name` in any Claude Code session. Each skill defines a multi-step automated workflow that Claude executes inline (with full tool access and context).

**Install:** copy a skill directory to `~/.claude/skills/` — Claude Code auto-discovers them on startup:

```bash
mkdir -p ~/.claude/skills/tc-create
cp skills/tc-create/SKILL.md ~/.claude/skills/tc-create/
```

### Toolstack

Skills reference your specific toolstack via placeholders. Before use, replace them in each `SKILL.md`:

| Placeholder | Meaning | Examples |
|---|---|---|
| `<product>` | Your product name | MyApp, Acme |
| `<tms>` | Test management system | Testiny, TestRail, Qase |
| `<analytics>` | Product analytics | Amplitude, Mixpanel, PostHog |
| `<error-monitoring>` | Error monitoring | Sentry, Rollbar, Bugsnag |
| `<metrics>` | Metrics / dashboards | Grafana, Datadog |
| `<logs>` | Log aggregator | Loki, Datadog Logs, Papertrail |
| `<wiki>` | Knowledge base | Notion, Confluence |
| `<task-tracker>` | Task / bug tracker | Asana, Jira, Linear |
| `<vcs>` | Version control | GitLab, GitHub |
| `<data-warehouse>` | Data warehouse | ClickHouse, BigQuery, Redshift |
| `<backend-repo>` | Path to backend repo | `~/MyApp/backend` |
| `<frontend-repo>` | Path to frontend repo | `~/MyApp/frontend` |
| `<admin-repo>` | Path to admin repo | `~/MyApp/admin` |

MCP tool names follow the pattern `mcp__<toolname>__<action>` — replace `<toolname>` with your actual MCP server name.

### Skills

| Skill | What it does |
|---|---|
| `tc-create` | Creates test cases in `<tms>` following naming conventions, priority rules (based on analytics event volume), and step format. Verifies all data against a real source — analytics, backend code, or monitoring. Never invents steps. Supports single and bulk mode. |
| `tc-update` | Updates existing test cases in `<tms>`: bulk field changes (type, priority, status), folder moves, content/steps edits, renames. Handles etag flow automatically. ACTIVE TCs require explicit confirmation before any change. |
| `tc-gap` | Gap analysis: fetches all existing TCs, collects signal sources per project (analytics events for Web, backend handlers for Back, admin panel pages for Admin), cross-references, and outputs a prioritized gap report. Auto-updates a wiki page; preserves manually added notes. |
| `bug-candidates` | Weekly bug triage prep: refreshes signal sources (`<error-monitoring>`, `<vcs>`, `<metrics>`, `<analytics>`, `<data-warehouse>`, `<task-tracker>`, `<wiki>` docs) and rebuilds the Bug Candidates list with dedup against prior week. Output feeds into `bug-dig`. Never auto-creates `<task-tracker>` tasks. |
| `bug-dig` | Investigates a bug or error monitoring issue: confirms whether it's a real user-impacting bug, monitoring noise, or theoretical risk. Collects evidence across `<error-monitoring>`, `<logs>`, `<analytics>`, git log, `<wiki>`, and code. Writes verdict to Bug Candidates `<wiki>` DB; never auto-creates `<task-tracker>` tasks. |
| `refresh-git` | Pulls latest from main for all configured repos and rebuilds code-review-graph indexes. Run at the start of a session before code analysis or QA work. |
| `qa-tooling` | Tooling audit: reads MemPalace diary across recent sessions (last 14 days), identifies recurring pain points and manual steps, and suggests new skills, agents, or automations to build. Focused on "what should we build next?" — not a session summary. |
| `daily` | Writes today's daily log based on the current conversation and pushes it to your QA notes repository. Asks before writing if anything is unclear. Not QA-specific — useful for any work log. |
| `push-qa-playbook` | Syncs local QA skills and agents to this GitHub repo. Diffs local vs repo, anonymizes private tool and product names, commits only changed files. Handles all public skills and agents. Updates README if content changed. |

---

## Claude Code agents

Sub-process agents dispatched for isolated subtasks. Unlike skills, agents run in a separate context with their own tool set.

**Install:** copy any agent file to `~/.claude/agents/` — Claude Code auto-discovers them on startup:

```bash
cp agents/test-case-writer.md ~/.claude/agents/
```

| Agent | What it does | Model |
|---|---|---|
| `test-case-writer` | Writes structured test cases in checklist format — for a feature, endpoint, user flow, or bug fix. Output is suitable for Qase, TestRail, or documentation. | haiku |
| `test-architect` | Designs a test strategy from scratch — framework selection, folder structure, test pyramid, CI integration plan. Audits the codebase before recommending. Specialized for async AI/LLM pipelines. | sonnet |
| `coverage-analyst` | Analyzes existing test coverage — finds gaps, identifies untested critical paths, prioritizes what to cover next. Doesn't write tests, gives a gap report. | haiku |
| `e2e-tester` | Writes E2E automated tests for web UI using Playwright — critical user flows, form interactions, auth flows, cross-browser checks. | sonnet |
| `api-tester` | Writes automated tests for REST API or gRPC endpoints — happy paths, edge cases, error scenarios, contract validation, auth flows, PostgreSQL state verification. | sonnet |
| `perf-tester` | Writes load and performance tests — concurrent users, job queue saturation, SLA validation. Uses k6 by default; specializes in async AI/media generation pipelines. | sonnet |
| `qa-researcher` | Fetches QA news and tool updates — new frameworks, testing practices for AI/LLM products, async pipeline testing. Returns a focused digest. | sonnet |
| `bug-reporter` | Turns raw notes from manual testing into a structured bug report — clear reproduction steps, severity, impact, environment details. Ready to paste into Jira, Linear, or GitHub Issues. | haiku |

---

## Who this is for

QA engineers working on AI products: LLM wrappers, generation pipelines, SaaS with billing limits. The templates assume async flows, external AI providers, and a small team where QA owns the entire testing strategy.

Pairs well with [claude-configs](https://github.com/anastasiiaanfimova/claude-configs) — the base Claude Code setup (hooks, agents, memory stack) that these skills run on top of.
