---
name: daily
description: >-
  Methodology for writing daily QA logs that serve two audiences in one
  document — product (narrative, no jargon) and engineering (concrete, with
  links). Covers structure, per-audience style rules, pruning rules,
  weekly summaries, and sources. Tool-agnostic.
---

# daily

A daily log is the simplest tool QA has for being **visible**. The
constraint: product readers and engineering readers want different
shapes of the same day. Solution — two blocks with shared concerns
(blockers, open questions) below them.

The document is structurally boring on purpose. The discipline is in
the writing rules and pruning rules — when to keep the block tight
vs when to expand it.

## Goal

Make QA work visible to both audiences from one document. Product
reads the upper block, engineering reads the lower block, both read
the questions. Each section can be copied independently into the
right meeting.

## Day section structure

```
## Day, MM-DD

### Product

**Done**
- [narrative phrase in product terms, no task IDs]

**Plan for tomorrow**
- [direction of work]

### Engineering

**Done**
- [status] [Task name](tracker-url) — what I did → outcome / verdict

Also looked at:
- [status] [Task name](tracker-url) — comment: verdict

**Plan for tomorrow**
- [status] [Task name](tracker-url) — action

### Questions / blockers

**Stuck today**
- what was hard → how it was unstuck

**Open questions**
- who I need / what kind of help

---
```

Layout is block-first. Status order within a block follows your
tracker columns (e.g. `to do → doing → testing → next release`).
Days append to a weekly file with `---` separators.

### Friday summary

If the day is Friday, append to the **product block**:

```
**For the week**
- Bug candidates: N (M already in tracker)
- Tasks closed: K
- Test runs executed: L
- TCs created: P
```

Skip if the metric is zero. "Closed tasks" usually means a real
status transition (e.g. `Testing → Next release` or `* → Done`),
not just a `completed=true` flag — many trackers update the flag
only on the final transition while the meaningful column move
happens earlier.

## Style — different per block

### Product block

- Narrative phrases ("we found that ...", "preparing for the fix",
  "tightened up X")
- No task IDs, HTTP codes, class names, file paths
- Internal tools and skills described by what they do for the
  process, not by their internal name. Bad: "updated the X skill".
  Good: "we now automatically check X in product metrics."
- MCP / infrastructure / agent internals not mentioned

### Engineering block

- Concrete with links and statuses
- Action → outcome via `—` or `→`
- Skills referenced by name ("updated `X`: added new step for Y")
- MCP / infrastructure OK

### Common

- First-person voice
- "We" for team actions, "I" for individual
- AI tools / agents are tools, not authors. "We found", not "Claude
  found"
- Short, no padding

## Hybrid content mapping

| Content type | Where |
|---|---|
| Skills / methodology updates | Both blocks, different phrasing |
| MCP / agent internals / infrastructure | Engineering only |
| Task work | Engineering with link; product narrative if relevant |
| Bug candidates / weekly review output | Product (count) + engineering (list with links) |

## Pruning rules — don't fill what isn't there

The trap is writing every section every day. If a section has no
content, skip it entirely.

| Section | Skip when |
|---|---|
| `Also looked at` | No comments on others' tasks |
| Bug candidates line | No new ones today |
| Test runs / TCs line | No work in TMS today |
| `Stuck today` | No real blockers |
| `Open questions` | No unresolved questions |
| Whole `Questions / blockers` | Both subsections empty |
| Yesterday reconciliation in product | No meaningful carry-overs |
| Friday summary | Not Friday, or zero activity |
| Plan bullet for a status | No tasks in that status assigned to me |

The reader trusts a sparse log more than a padded one. Empty content
in a "Stuck today" section produces "I had no problems today" — at
best meaningless, at worst dishonest.

## Sources

| Target | Source | What |
|---|---|---|
| Done — own (engineering) | Tracker | Tasks I'm assigned to, modified today |
| Done — "Also looked at" | Conversation memory + tracker | Task IDs from chat history, verified via task comments |
| Done (product) | Above + diary + bug candidates + TMS | Reformulation in product language |
| Bug candidates | Bug-candidates database | Pages I created/updated today |
| TMS work | TMS | TCs / runs by me today; skip line if zero |
| Plan | Tracker | My tasks where `completed=false`, filtered by section |
| Yesterday plan vs reality | Last day's section in weekfile | Carry-overs to surface in product narrative |
| Stuck / questions | Conversation context + memory | Hybrid detect; surface candidates in chat |
| Stale tasks (chat only) | Tracker | My tasks where section hasn't moved in N days |

### Status display

Status in `[brackets]` reflects the **current** column when the daily
is written, not the column on `TARGET_DATE`. When backfilling
(yesterday or earlier), prefix the chat output with: "Daily for
<weekday>, <date>. Statuses are current."

## Workflow

### Step 1 — Setup

Parse the date:
- no arg / "today" → `TARGET_DATE = today`
- "yesterday" → `TARGET_DATE = today - 1`
- explicit `YYYY-MM-DD` → as is, with disclaimer if >1 day off

Compute the week file: `daily/week-<MONDAY>_<FRIDAY-SHORT>.md`.

### Step 2 — Parallel data collection

Fire in parallel (single message, multiple tool calls):

1. Tracker — my tasks modified today
2. Tracker — my tasks `completed=false` (for the plan)
3. Bug-candidates DB — entries by me, today
4. TMS — TCs/runs by me, today
5. Conversation memory search — TARGET_DATE + topics ("comment",
   "stuck", "task ID")
6. Diary — last 3 entries
7. Yesterday's section in the weekfile (for plan reconciliation)

### Step 3 — Extract "Also looked at"

From memory + diary, find task IDs not present in Step 2 results.
For each, fetch task comments and verify a comment of mine is dated
on `TARGET_DATE`. Only verified entries go into the subsection.

### Step 4 — Stale tasks scan

From the plan results, filter tasks where the section column hasn't
changed in `>5` days. Surface in **chat only**, not in the file.
Pattern: "Stuck in one column: [list]". The file is a log, not an
audit; staleness alerts are a current-state signal.

### Step 5 — Hybrid detect for Questions

Search memory + diary + current chat for patterns:

- "stuck", "didn't work", "fixed", "figured out" → candidates for
  **Stuck today**
- "waiting for", "need from", "missing" → candidates for **Open
  questions**

Show candidates in chat for confirmation before adding to file.

### Step 6 — Reconcile with yesterday's plan

Parse "Plan for tomorrow" from the previous day in the weekfile.
Match against actual work today:

- Done → already in Done section, don't double-mention
- Carried over → write a one-liner in product ("carried X over from
  yesterday")
- Dropped → flag in chat for the user

### Step 7 — Drafts in chat

Show in this order:

1. Product block (Done + Plan)
2. Engineering block (Done + Also looked at + Plan)
3. Stuck/questions candidates
4. Stale tasks (informational, not in file)
5. Plan clarification flags

Wait for confirmation / edits.

### Step 8 — Friday summary

If `TARGET_DATE` is Friday and weekly data has any non-zero count,
aggregate:

- Bug candidates created this week
- Tasks closed this week — walk task histories for section
  transitions like `Testing → Next release` or `* → Done`. The
  `completed_at` filter alone misses tasks completed via meaningful
  transitions short of "Done".
- TMS runs and TCs created this week

Append the "For the week" block to the product block.

### Step 9 — Write to file + push

Pull the latest weekfile, append the day section (or update if
already exists), commit with `daily: <date>`, push.

The weekfile lives in a separate "QA logs" repo / location — not
mixed with code repos. Daily logs are documentation, kept versioned
for history.

## Anti-patterns

- ❌ HTTP codes / class names / file paths in the product block
- ❌ Same wording repeated in both blocks — they have different
  audiences
- ❌ "Updated skill X" in product block without explaining what it
  gives the process
- ❌ Filling the Questions section for completeness
- ❌ Plan bullets in product block with task IDs
- ❌ "Claude found / read / updated" — AI is a tool, not the
  author of the day
- ❌ Writing to file without confirmation when auto-detect surfaced
  candidates
- ❌ Stale-tasks list in the file (chat only)
- ❌ MCP / agent internals in the product block
- ❌ Intermediate steps as separate "Done" bullets ("searched docs",
  "debugged X") — the bullet should be the outcome, not the
  process
- ❌ Subsection "Own:" as a header — own tasks go directly under
  `**Done**`
- ❌ Backfill >1 day without the "statuses are current" disclaimer
- ❌ Reporting zero activity for a metric ("Bug candidates: 0") —
  skip the line
