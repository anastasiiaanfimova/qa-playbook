# QA Onboarding: Starting on a New Project

A template for the first QA on a project with no tests and no processes.
Focused on AI-first products — generation flows, async pipelines, LLM integrations.

---

## Principles

- **Risk first.** Test what matters for the business and users — not everything.
- **Fresh eyes are a one-time resource.** Exploratory testing starts in the first few days. After a month you stop noticing things.
- **Docs only survive if they're easy to update.** One current page beats ten outdated ones.
- **Process over coverage.** A regular regression with 50 cases beats 500 cases with no process.

---

## Onboarding

### Access — request on day one

- [ ] Staging / dev environment + test accounts (different plans and roles)
- [ ] Repository
- [ ] Jira
- [ ] TMS — agree on a choice (Qase, TestRail, etc.)
- [ ] Swagger / API docs
- [ ] Notion / Confluence / whatever they use for docs
- [ ] CI/CD — at least read-only
- [ ] Logs and monitoring (Sentry, Datadog, Kibana, etc.)
- [ ] Queue manager (Redis, RabbitMQ, etc.) — at least read-only
- [ ] Database — at least read-only on staging

### Talk to product — business risks

- [ ] What matters most to users? What would make them leave?
- [ ] Which features get used the most?
- [ ] What are users complaining about right now?
- [ ] What can never break, no matter what?
- [ ] What's shipping in the next 1–2 months?

### Talk to developers — technical risks

- [ ] What breaks most often?
- [ ] What parts of the code are people afraid to touch?
- [ ] What's changed in the last 2–3 months? (= unstable zones)
- [ ] Are there known bugs that aren't being fixed yet?
- [ ] Which external dependencies are critical and how stable are they?
- [ ] How does the release process work right now?

**AI-first:**
- Which LLM providers do you use?
- Have there been issues with provider availability or API changes?
- How are timeouts and provider errors handled?

### Support

Real bugs and user pain live here — understand how support works before you start.

- [ ] Who collects user feedback and where? (Intercom, Zendesk, Telegram, email, etc.)
- [ ] Is there dedicated support or does the dev / product team handle it?
- [ ] Where does support report issues? How are critical bugs escalated?
- [ ] Is there access to ticket history? — real bugs are usually already described there

### Shadow QA processes

Before QA existed, someone was still checking things — informally. Find it, don't break it.

- [ ] Who clicks through the product before a deploy? How does that work?
- [ ] Are there informal checklists or "don't forget to check X" notes somewhere?
- [ ] Who's the first to know when something breaks in prod?
- [ ] How is the "ok to deploy" decision made right now?

→ **Output:** a list of pain points from both sides (business + tech), understanding of shadow processes, support map.

---

## Exploration

Exploratory testing starts in the first few days — before your eyes adjust.

### Exploratory testing

- [ ] Walk through the product as a new user — no plan, no scenarios
- [ ] Note everything confusing, inconvenient, or broken without filtering
- [ ] Flag UX issues too: not just "doesn't work" but "works but unclear" or "awkward"
- [ ] Write down "why is it done this way?" questions — some of them will turn out to be bugs

First bug reports will come from here — don't wait for the right process to be in place.

**AI-first:**
- Run the full generation cycle: submit a job, wait for the result, download it
- Note how long it takes and whether the waiting state is clear to the user
- Check what happens on generation error or timeout

### Repository analysis

- [ ] Understand the structure: modules, services, core entities and their relationships
- [ ] Find the most frequently changed files (`git log --stat`) — unstable zones
- [ ] Find external integrations: payments, email, auth
- [ ] Look at the CI/CD config: what's being checked automatically right now

**AI-first:**
- Find job queues and generation workers
- Understand retry logic and provider error handling
- Find where media files are stored and how they're served

### Swagger / OpenAPI analysis

- [ ] List all endpoints by group (auth, billing, core features)
- [ ] Mark the critical ones — without which the product doesn't work
- [ ] Find non-obvious or poorly documented fields

**AI-first:**
- Map async endpoints separately: job submit, status check, result retrieval, webhooks
- Document all possible job states and transitions between them

### Existing documentation

- [ ] Find descriptions of key user flows
- [ ] Find known limitations and edge cases
- [ ] Check if docs are current — if they're old, flag it

### Monitoring

On smaller projects monitoring is often incomplete — one of the first growth areas for QA.

- [ ] Get access to existing dashboards and understand what's being monitored
- [ ] Identify what matters from a QA perspective: error rate, latency, failed jobs, payment errors
- [ ] Note what's not monitored but should be — that becomes a separate task

**AI-first:**
- Check if there are generation metrics: success rate, avg time, provider errors, queue depth
- If not — raise it with the team soon

→ **Output:** risk map + monitoring picture + first bug reports.

**Risk map format:**

| Area | Risk | Priority | Automation |
|------|------|----------|------------|
| Auth | No access to the product | Critical | Yes |
| Generation | Core feature doesn't work | Critical | Partial |
| Billing | Lost payment | Critical | No |
| Plan limits | Free access to paid features | High | Yes |

---

## Documentation

Write only what people will actually use. A doc that's hard to update will stop being updated.

### Request from the team

- [ ] User flow descriptions for key features (if missing — write them yourself after exploration and get them reviewed)
- [ ] Release process description: how they deploy, how often, who decides
- [ ] What's currently monitored: alerts, dashboards, on-call

**AI-first:**
- Agree on acceptance criteria for generation — before writing any tests
- Define what a "successful result" means: file formats, allowed sizes, max time, error codes
- Without this, manual testing is subjective and automation is impossible

### Create

- [ ] **Test strategy** (1–2 pages): what we test, what we don't and why, how we prioritize risks
- [ ] **Risk map** (from Exploration)
- [ ] **Environment guide**: how to get access, which accounts to use, staging quirks
- [ ] **Test case storage structure**: folder structure in TMS, checklist format, Jira linking

---

## Test Cases and Checklists

Format: checklists with comments — not step-by-step instructions. Step-by-step details are expensive to maintain and go stale fast.

### Approach

- Write against the risk map: Critical → High → Medium
- For each area: what we check + what to pay special attention to
- When a feature changes — update the related checklists right away
- Archive outdated cases, don't let them pile up

### Checklist example

```
## Image generation

- [ ] Successful cycle — check format, size, and file accessibility
- [ ] Provider error — user message is clear, job didn't get stuck
- [ ] Timeout — status updates, user is notified
- [ ] Invalid prompt — handled at validation level, doesn't 500
- [ ] Retry after error — works without a page reload
```

### Regression and smoke

Defined alongside writing checklists — not a separate phase.

- [ ] While writing a checklist, mark what goes into smoke (most critical, fast to run)
- [ ] Define a regression set for releases (Critical + High)
- [ ] Target: manual regression ≤ 4 hours, smoke ≤ 30 minutes
- [ ] Log each run: date, version, passed/failed, bugs found

**AI-first:**
- Include a full generation cycle on staging with a real provider in release regression

---

## Automation

Details are worked out separately — just the plan and key checkpoints here.

- [ ] Set up a framework for the project's stack
- [ ] Smoke suite: automate the most critical paths, runs in < 5 minutes
- [ ] Connect smoke to CI/CD on every staging deploy
- [ ] Expand coverage by risk map: API first, then E2E golden paths
- [ ] Load tests for critical async flows — a separate iteration

Only automate stable functionality. Actively changing code = constantly broken tests.

**AI-first:**
- Mock LLM providers at the HTTP transport level — real calls in CI are expensive and flaky
- Write helpers for the async pattern: `submit_job → poll_status → assert_result`
- Load tests first on the job queue: concurrent submissions, saturation point, generation time degradation under load
