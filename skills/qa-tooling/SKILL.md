---
name: qa-tooling
description: >-
  Tooling audit for <product> QA. Looks at recent work via diary, identifies
  pain points and manual steps, and suggests new skills, agents, automations,
  or workflow improvements. NOT a session summary — focused on "what should we
  build next to work more comfortably?"
  Trigger: "qa-tooling", "что улучшить в тулинге", "нужны новые скилы", "подведи итоги по тулингу".
version: 0.1.0
---

# QA Tooling — Tooling Audit

Look at recent work and answer: **what should we build or improve to make QA easier?**

Output: prioritized suggestions for new skills, agent improvements, MCP tools, automations, workflow changes.

## Trigger phrases

- `/qa-tooling`
- "что улучшить в тулинге", "нужны новые скилы", "подведи итоги по тулингу"

## Workflow

### Step 1 — Read recent diary

`mcp__mempalace__mempalace_diary_read` with `last_n=30`.

Filter entries: keep only those with `date` within the last **14 days** from today.
Note the actual date range covered (earliest → latest entry date) and include it in the report header — so the user always knows what time period is being analysed.

If fewer than 3 entries remain after filtering → say so explicitly and suggest the user runs the skill again after more sessions have accumulated.

Extract recurring signals from the filtered entries:
- What required manual steps that felt repetitive?
- What data was missing or hard to get?
- What produced unexpected/wrong output (false positives, wrong format)?
- What took multiple tries to get right?
- What was skipped because "too much work"?

### Step 2 — Inventory existing skills

```bash
ls ~/.claude/skills/
```

For each skill, note: what it does and any known limitations seen in diary.

### Step 3 — Synthesize suggestions

For each pain point from Step 1, map to a concrete suggestion:

| Pain point type | Possible solution |
|---|---|
| Repeated manual lookup | New skill or reference file |
| Tool produced wrong output | Improvement to existing skill |
| Missing data source | New MCP integration or API reference |
| Multi-step flow done manually | New agent or skill automation |
| Naming/format inconsistency | Update memory rule or skill guideline |

### Step 4 — Output retro

```
## QA Tooling — YYYY-MM-DD

### 🔧 Suggest to build / improve

**HIGH — solves a real pain:**
1. [Skill/tool name] — [what it does, why it's needed]
   Signal from diary: "[quote or pattern from diary]"

2. ...

**MEDIUM — would be convenient:**
...

**LOW / idea for later:**
...

### ✅ What works well (keep as is)
- [skill/tool] — [why it works]

### ⚠️ Known limitations (not critical, but keep in mind)
- [limitation] — [workaround if any]
```

### Step 5 — Write diary entry

`mcp__mempalace__mempalace_diary_write` with compact AAAK summary.
Topic: `qa-tooling`.

## Rules

- Base every suggestion on a concrete signal from diary — no speculative "it would be nice"
- One suggestion per pain point — don't bundle unrelated improvements
- If diary has no clear pain points, say so explicitly ("Diary shows no clear pain points — retro has nothing to fill")
- Don't reproduce what tc-gap does (coverage analysis) — that's a separate skill
- Keep total output under 15 items
