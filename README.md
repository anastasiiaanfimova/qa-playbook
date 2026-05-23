# QA Playbook

> **Archived 2026-05-23.** Frozen methodology snapshot — read-only. The local operational config evolved past these patterns; the published version is preserved for visibility and reference. Forks welcome.

A reference collection of QA methodology — written as Claude Code skills,
but readable as standalone documentation.

Focused on the scenario where you are the **first QA on a team** at an
AI/SaaS product, building processes from scratch.

---

## What's inside

### Onboarding

- **`qa-onboarding-template.md`** — practical onboarding checklist for the
  first QA on a project without tests or processes. Covers access requests,
  conversations with product/dev, the exploratory-testing window, risk
  mapping, documentation, smoke vs regression splits, and a phased
  automation roadmap. Each section has an AI-first block for async
  generation flows (job submission → polling → assertion patterns,
  LLM provider failures, queue saturation, output acceptance criteria).

### Methodology skills

The `skills/` directory contains **methodology** — process knowledge,
decision frameworks, writing rules, anti-patterns. Not tool-coupled
working code.

Each `SKILL.md` describes how to do one piece of QA work in a way that
survives a stack change. Switch tracker, swap your TMS, change
monitoring vendor — the methodology still applies; only the tool calls
change.

| Skill | What it covers |
|---|---|
| `tc-create` | When to write a TC, where data comes from, status confidence levels, title patterns, priority criteria, duplicate-check algorithm |
| `tc-update` | Status-driven modification protocol (ACTIVE per-item, DRAFT/GUESS bulk-ok), STALE triage with gap-context-first reading, single vs bulk discipline, error surfacing |
| `tc-plan` | Plan-as-derived-view model with explicit criteria, two-phase preview/apply pattern via a staging marker field, drift reconciliation |
| `tc-gap` | Three-axis coverage analysis (analytics events / backend handlers / admin pages), skeleton-output pattern, STALE marking |
| `bug-dig` | Investigation methodology for "is this a real bug?" — root-cause-in-system principle, four-verdict model, multi-source quick-triage triangle, browser-side error path, code review checklist |
| `bug-nominate` | Single-writer pattern for persisting bug candidates: separation of investigation from durable write, fingerprint dedup, body-protection rule, interactive vs silent batch modes |
| `bug-review` | Multi-source signal convergence for weekly bug discovery: independent telemetry sources, two time-window discipline (delta vs trend), three-layer flow, idempotency under flexible cadence |
| `task-create` | Bug ticket structure: two-layer body (product + engineering), section requirements per bug type, priority logic with downstream-effect awareness, writing style |
| `task-comment` | Follow-up comment discipline: two genres (fresh numbers / research summary), strict writing rules, pre-publication self-check |
| `branch-analyze` | Feature-branch QA analysis: paired-branch detection, env-state decision tree, anomaly detection vs baseline, regression-zone smoke list, manual-flow generation |
| `billing-trace` | Payment trace methodology: DB lookup → webhook logs → external settlement → diagnostic tree, cross-environment routing trap |
| `daily` | Two-block daily log: product (narrative) + engineering (concrete), pruning rules, Friday weekly summary, hybrid content mapping |
| `qa-audit` | Tooling retrospective: time-windowed signal extraction, pain → solution mapping table, prioritized output |

### Agents

Sub-process agents in `agents/` — runnable tools dispatched for isolated
subtasks (writing test cases, designing test strategy, coverage analysis,
E2E / API / perf testing, QA news digest, bug reports).

| Agent | What it does | Model |
|---|---|---|
| `test-case-writer` | Writes structured test cases in checklist format suitable for any TMS or doc | haiku |
| `test-architect` | Designs test strategy from scratch — framework selection, folder structure, test pyramid, CI integration plan; audits the codebase first | sonnet |
| `coverage-analyst` | Gap report — finds untested critical paths, prioritizes what to cover next | haiku |
| `e2e-tester` | E2E web tests with Playwright | sonnet |
| `api-tester` | REST / gRPC tests — happy paths, edge cases, contract validation, auth, DB state | sonnet |
| `perf-tester` | Load & perf tests with k6, specialized for async AI/media pipelines | sonnet |
| `qa-researcher` | Digest of QA news and tool updates | sonnet |
| `bug-reporter` | Turns raw test notes into a structured bug report | haiku |

---

## How to use

These skills are written as **reference documents**. Read the
methodology, apply to your own stack. They aren't drop-in
configurations — your tracker / TMS / monitoring tool calls go in your
own local working version.

If you want to install one as an actual Claude Code skill in your
environment, copy the directory into `~/.claude/skills/` and adapt
the abstract tool references to your stack:

```bash
cp -r skills/tc-create ~/.claude/skills/
# Then edit ~/.claude/skills/tc-create/SKILL.md to reference your real
# TMS, MCP tool names, project IDs, etc.
```

Agent files install the same way:

```bash
cp -r agents ~/.claude/
```

---

## Who this is for

QA engineers working on AI products: LLM wrappers, generation pipelines,
SaaS with billing flows. The methodology assumes async pipelines,
external AI providers, and a small team where QA owns the entire
testing strategy.

Pairs well with
[claude-configs](https://github.com/anastasiiaanfimova/claude-configs) —
base Claude Code setup (hooks, agents, memory stack) that these skills
run on top of.

---

## Maintenance note

This is a published methodology snapshot, not a drop-in toolkit. Skills
update when their underlying methodology genuinely evolves — not on
every local working-version change.
