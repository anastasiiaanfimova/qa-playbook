# CLAUDE.md — qa-playbook

> **Archived 2026-05-23.** This repo is frozen — editing/syncing instructions below are historical. Open the GitHub repo for read-only reference.

This is the **public clone** of `github.com/anastasiiaanfimova/qa-playbook`.
Methodology snapshots of QA skills, intentionally tool-agnostic.

This is **not** where the live working skills live (those are in
`~/.claude/skills/`). This is the published artifact for visibility +
portability across job/stack changes.

## What's in here

```
skills/                    # 13 methodology variants
  tc-create/SKILL.md       # TC writing methodology
  tc-update/SKILL.md       # TC modification protocol + STALE triage
  tc-plan/SKILL.md         # Plan-as-query + two-phase preview/apply
  tc-gap/SKILL.md          # Coverage gap analysis
  bug-dig/SKILL.md         # Bug investigation methodology
  bug-nominate/SKILL.md    # Single-writer pattern for bug DB
  bug-review/SKILL.md      # Multi-source signal convergence
  task-create/SKILL.md     # Bug ticket structure (longest, densest)
  task-comment/SKILL.md    # Follow-up comment discipline
  branch-analyze/SKILL.md  # Feature branch QA analysis
  daily/SKILL.md           # Two-block daily log methodology
  qa-audit/SKILL.md        # Tooling retrospective methodology
  billing-trace/SKILL.md   # Payment trace methodology

agents/                    # 8 generic QA agents (already tool-agnostic)
README.md                  # Public entry point
qa-onboarding-template.md  # First-QA onboarding checklist
```

## The pattern (read first if editing)

These skills are **methodology-only**, not blueprints. They describe
processes, decision frameworks, writing rules — things that survive a
stack change. No `<placeholder>` syntax for tools, no MCP calls, no
project IDs.

**Job-change test:** every claim in a skill should still apply if you
switch tracker / TMS / monitoring vendor. If something fails the test,
it belongs in the local working skill (`~/.claude/skills/<name>/`),
not here.

History of how this pattern emerged (recorded in the project's
MemPalace decisions wing):

- Started with regex anonymization of full local skills → fragile
- Pivoted to blueprint with `<placeholders>` → still tool-coupled
- Pivoted again to methodology-only → final pattern (2026-05-09 evening)

## Editing workflow

1. Pick the skill you want to update
2. Edit `skills/<name>/SKILL.md` directly (this is the published version)
3. If editing in response to a meaningful methodology evolution, also
   consider whether the change applies to the local working skill in
   `~/.claude/skills/<name>/`
4. `git add . && git diff --staged` to review
5. `git commit -m "<msg>"` and `git push`

No automation, no scripts, no `qa-playbook-push` skill — that infra was
removed when this pattern was adopted. Manual is the right level for
how often this changes.

## Editing principles

- The methodology must read clean to a stranger from a different stack
- Every "rule" should be defensible without referring to a specific
  tool ("never use `mcp__sentry__*` for X" is not methodology — "never
  conclude 'no errors' from a single source" is)
- Keep examples generic — `type=<x>` over `type=videogen`, `<provider>`
  over `Stripe`
- When in doubt, the example illustrates the *shape* of input, not the
  specific values

## Don't do here

- Don't `cd` to your live project directory from this session — that's
  a different MemPalace context, different project memory
- Don't run any of the working skills (`/bug-dig`, `/task-create`,
  etc.) here — they expect the live tool stack (TMS, tracker,
  monitoring) and will fail or produce wrong output
- Don't try to install push-mirror infra back — it's gone for a reason

## Maintenance frequency

This is a **visibility artifact**, not a kept-current toolkit. Update
when methodology genuinely evolves (rare). Don't sync on every working
skill tweak.

If many sessions go by without updates, that's fine — methodology
doesn't churn weekly. The job-change test is the trigger: when you
realize a principle now is wrong / incomplete / better-stated, that's
the moment to update.

## When you'll come back here

- Major methodology change in a working skill (rewrote bug-dig's
  triage approach → consider updating the published variant)
- New methodology you want to share (cross-cutting principle that's
  worth its own SKILL.md)
- Bug or typo a reader pointed out

## Related public repo

`~/Claude/claude-configs/` — base Claude Code setup. Adopted the same
methodology-only pattern by analogy with this repo (2026-05-09 evening
follow-on). Same manual edit-and-commit workflow; no push automation
on either side.
