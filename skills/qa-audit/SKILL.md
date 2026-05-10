---
name: qa-audit
description: >-
  Methodology for tooling retrospectives. Reads recent work history,
  extracts pain signals, maps them to concrete tool/skill/automation
  suggestions, and outputs a prioritized list. NOT a session summary —
  the question is "what should we build next to work more comfortably?"
  Tool-agnostic.
---

# qa-audit

A QA tooling retrospective is the cheapest way to surface what's
actually slowing you down. The trick is anchoring it in real signals
— diary entries, conversation memory, recent friction — not in
opinions about what *might* be useful.

The output is a short prioritized list of suggestions, each rooted in
a specific recurring pain. Speculation ("it would be nice if...")
doesn't make the list.

## What this is, and isn't

| Is | Isn't |
|---|---|
| A retrospective on tooling friction | A summary of what got done |
| Suggestions for new skills / automations / references | A status report |
| Anchored in concrete signals from recent work | A wishlist |
| Short prioritized list (typically 5–15 items) | A long brain-dump |

## Time window

Default: 14 days. Wide enough to catch recurring patterns, narrow
enough to avoid stale frustrations from solved problems.

Always state the actual date range in the output header — "from
2026-04-25 to 2026-05-09, N entries" — so the reader knows what
period the audit covers.

If fewer than 3 entries fall in window, say so and suggest re-running
later. Don't try to make audits with insufficient signal.

## Workflow

### Step 1 — Read recent work history

Pull diary / conversation memory entries within window. If diary
entries are too compressed to extract a clear signal, supplement with
full conversation history search (using the entry date + a key term
from it) — diary captures the verdict, full conversation captures
the friction.

Extract recurring signals across entries:

- What required manual steps that felt repetitive?
- What data was missing or hard to get?
- What produced unexpected / wrong output (false positives, wrong
  format)?
- What took multiple tries to get right?
- What was skipped because "too much work"?

A signal becomes worth suggesting against when it appears **at least
twice** in the window. One-off frustrations belong in the diary, not
in the audit.

### Step 2 — Inventory current tooling

List your existing skills / agents / scripts. For each, note what it
does and any known limitations seen in the diary. This prevents
suggesting something that already exists in a half-broken form (the
right answer is often "fix the existing skill", not "build a new
one").

### Step 3 — Map pain → suggestion

| Pain pattern | Suggestion shape |
|---|---|
| Repeated manual lookup of the same data | New skill or reference file |
| Tool produced wrong output | Improvement to existing skill |
| Missing data source | New MCP integration or API reference |
| Multi-step manual flow | New skill or agent automation |
| Naming / format inconsistency surfacing repeatedly | Update memory rule or skill guideline |
| Same kind of question to teammates each week | Capture as a reference doc |
| Same kind of bug recurring after release | Add a regression-zone check to the test plan |

One suggestion per pain. Don't bundle unrelated improvements into a
"do all of these" item — that just delays the most-needed one.

### Step 4 — Output

```
## QA Tooling — YYYY-MM-DD (range: YYYY-MM-DD to YYYY-MM-DD, N entries)

### Suggested

**HIGH — solves a real recurring pain:**
1. [Skill / tool name] — [what it does, why needed]
   Signal from diary: "[quote or pattern]"

2. ...

**MEDIUM — would be convenient:**
...

**LOW / parking lot:**
...

### Working well (leave as is)
- [skill / tool] — [why it's working]

### Known limitations (not blockers)
- [limitation] — [workaround if any]
```

Keep the suggested list under 15 items total. A long list signals
either too wide a window or insufficient prioritization.

### Step 5 — Save the audit summary

Write a compact summary to your diary / memory (one entry, AAAK or
similar compression). The audit itself is durable artifact; the
diary entry is just so the next audit can see what was suggested
last time.

## Rules

- ✅ Every suggestion anchored in a concrete signal from history
- ✅ One suggestion per pain — no bundling
- ✅ State the time window explicitly in the header
- ✅ If diary shows no clear pains, say so — don't manufacture
  suggestions ("diary doesn't show clear pains — the retro has
  nothing to fill")
- ❌ Don't reproduce what other dedicated retrospective skills do —
  this skill is specifically about tooling friction, not coverage
  analysis or bug triage
- ❌ Don't suggest based on speculation — "it would be nice if X"
  isn't enough; "I had to do X manually 4 times this week" is

## Anti-patterns

- ❌ Bundling 5 unrelated improvements into one item — delays the
  important one
- ❌ Suggesting the same thing every audit — if it's been HIGH for
  three audits running and not built, the audit isn't the problem;
  raise it explicitly with the user
- ❌ Mixing audit with daily — the daily covers what got done; the
  audit covers what should get built
- ❌ Long output (>20 items) — signals lack of prioritization
- ❌ "Great progress this week!" / "we're improving!" — that's a
  vibes report, not a tooling audit
- ❌ Suggesting something based on a single entry from 2 weeks ago
  that's not been a problem since — windows mean recency, not
  archival
