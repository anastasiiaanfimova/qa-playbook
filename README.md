# QA Playbook

Templates and resources for QA at AI-first products — async job pipelines, LLM wrappers, SaaS billing.

Focused on the scenario where you are the **first QA on the team**, starting from scratch with no existing tests or processes.

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

## Who this is for

QA engineers working on AI products: LLM wrappers, generation pipelines, SaaS with billing limits. The templates assume async flows, external AI providers, and a small team where QA owns the entire testing strategy.
