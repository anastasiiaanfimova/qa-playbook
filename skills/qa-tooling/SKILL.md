---
name: qa-tooling
description: >-
  Tooling audit for <product> QA. Looks at recent work via diary, identifies
  pain points and manual steps, and suggests new skills, agents, automations,
  or workflow improvements. NOT a session summary — focused on "what should we
  build next to work more comfortably?"
  Trigger: "qa-tooling", "что улучшить в тулинге", "нужны новые скилы", "подведи итоги по тулингу".
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

If a diary entry is too compressed to extract a clear signal → supplement with `mcp__episodic-memory__search` using the entry date + key term (e.g. `"2026-04-24 schemathesis"`). Episodic has the full conversation text for that session.

Extract recurring signals from the filtered entries:
- What required manual steps that felt repetitive?
- What data was missing or hard to get?
- What produced unexpected/wrong output (false positives, wrong format)?
- What took multiple tries to get right?
- What was skipped because "too much work"?

### Step 2 — Inventory existing skills

```bash
ls "$HOME/.claude/skills/"
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

### 🔧 Предлагаю построить / улучшить

**HIGH — решает реальную боль:**
1. [Skill/tool name] — [что делает, почему нужен]
   Сигнал из дневника: "[цитата или паттерн из diary]"

2. ...

**MEDIUM — было бы удобно:**
...

**LOW / идея на потом:**
...

### ✅ Что работает хорошо (оставить как есть)
- [skill/tool] — [почему работает]

### ⚠️ Известные ограничения (не критично, но держать в голове)
- [limitation] — [workaround если есть]
```

### Step 5 — Write diary entry

`mcp__mempalace__mempalace_diary_write` with compact AAAK summary.
Topic: `qa-tooling`.

## Rules

- Base every suggestion on a concrete signal from diary — no speculative "it would be nice"
- One suggestion per pain point — don't bundle unrelated improvements
- If diary has no clear pain points, say so explicitly ("Дневник не показывает явных болей — ретро нечем наполнить")
- Don't reproduce what tc-gap does (coverage analysis) — that's a separate skill
- Keep total output under 15 items
