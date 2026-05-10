---
name: task-comment
description: >-
  Methodology for follow-up comments on existing bug tickets. Two genres
  (fresh numbers / research follow-up), strict writing rules to avoid
  noise, and a pre-publication self-check that catches common bad
  patterns. Tool-agnostic.
---

# task-comment

A follow-up comment is a low-budget artifact. The reader already has
context (the ticket); they're scanning for what's new or what's the
answer. Long comments dilute the signal; self-narrative dilutes the
trust; missing time-windows dilute the meaning.

This skill captures the writing discipline for follow-up comments so
they earn their place in the thread.

Hard rule: **always show a draft and wait for explicit confirmation
before posting.**

## Two genres

| Genre | When | Template |
|---|---|---|
| Fresh numbers | Investigation or weekly review found updated data on an open ticket | A |
| Research follow-up | The ticket asked a question; we have a structured answer | B |

Pick by what you have to share. Mixing them produces a confused
comment that's neither a metric update nor an answer.

## Template A — Fresh numbers

```
Fresh numbers (<date>)

<source>:
  <metric>: <prev> → <new> (<+delta>)
```

Sources: error monitor / logs / analytics / metrics dashboard. Only
what actually changed. If a metric didn't move, omit it — silence
beats noise.

Add one extra paragraph **only if** something materially shifted:

- A new user got affected
- A deploy landed near `first_seen`
- The root cause shifted
- A related fix landed

If none of those — the numbers block is the entire comment.

## Template B — Research follow-up

A structured answer to the question the ticket posed. Shape:

```
**Short answer:** [one-line direct answer to the ticket's question +
the main caveat]

[2–5 paragraphs with facts, time windows, concrete numbers]

[Closing link to the full technical write-up: candidate database,
wiki page, drawer]
```

Paragraphs separated by blank lines. Don't wrap your own paragraphs
in blockquotes — visually it reads as if everything is quoted from
elsewhere.

### What goes in the paragraphs

- Before/after numbers with time windows ("50/day → 0–3/day, since
  YYYY-MM-DD")
- What shipped / what's already resolved — facts, not announcements
- What remains open + a concrete promise of the next step
- Inline references to related tickets / latent bugs

### What stays out

- Restating what's already in the ticket body
- Detailed stack traces, class names, file paths — those go in the
  candidate-database entry or drawer, not the comment
- Self-narrative ("I investigated", "I found", "I want to observe")
  — rewrite impersonally

### Length

One screen is enough. If it grew longer, move details to the
candidate-database entry or drawer and link from the comment.

## Writing style — applies to both templates

These rules read as a checklist because every one of them turns up
in real drafts and degrades the comment when missed:

- **Impersonal fact > "I"-action.** "I found 10 cases" → "10 cases
  exist". "I checked that ..." → "logs show ...". Reader doesn't
  care who specifically found it; they care about the fact.
- **Don't reference the ticket's own name** in a comment on that
  ticket. "Today the additional fix DEV-1887 shipped" → "Today the
  additional fix shipped". Tautology — the comment is already
  attached.
- **Concrete short promise > verbose plan.** "I'd like to observe
  another day and confirm before closing" → "I'll re-check
  tomorrow." One verb + one window.
- **No filler adjectives.** "Known latent bug" → "latent bug".
  "Known" / "new" / "small" / "fairly large" carry no information.
- **Don't announce admin operations.** "I'll log this separately in
  the candidate database" → delete. If it's done, it'll be visible.
  Promised future writes that haven't happened are noise.
- **Every number carries a time window.** "14 days", "since
  YYYY-MM-DD", `first_seen — last_seen`, "last seen DATE". A bare
  number is incomplete.
- **"We"-language for code and team decisions, neutrally.** "Naive
  fix" → "fix that addresses one bug without considering the other".
- **Short direct sentences without padding.** "When testing, I ran
  into the situation that the filter returns 0 templates" → "the
  filter returns 0 templates".
- **Facts inline.** Numbers, dates, versions, paths in prose where
  they fit. `alembic_version=9f0e1d2c3b4a`, `192.168.14.3`,
  `template.gender` — inside the sentence, not in a separate aside.
- **Cause-effect with explicit connectors.** "Column missing. API
  returns null. Frontend filters empty." → "Column missing — and
  because of that, API returns null and frontend filters empty."
  One sentence binds the chain.

## Pre-publication self-check

Before showing the draft to the user, run this checklist on the
text. If a row matches, fix it silently in the draft — don't
describe what you found, just publish a cleaner version.

| # | Check | If found |
|---|---|---|
| 1 | "I"-form: "I found", "I checked", "I want to observe" | Rewrite impersonally: "exists", "logs show", "I'll re-check" |
| 2 | Mention of the ticket's own ID | Remove; the comment is already attached |
| 3 | Verbose promise ("I'd like to observe and confirm before closing") | Replace: one verb + concrete time window |
| 4 | Filler adjectives ("known", "new", "small", "important") | Remove entirely |
| 5 | Self-announcing admin operations ("I'll log separately", "I'll check later") | Remove; if done, will be visible |
| 6 | Bare ticket-link without rich-mention markup (where the tracker supports rich mentions) | Use the rich-mention form so reader sees status + name |

Six clean → show the draft. Otherwise fix first.

## Things never to write

- "Fix isn't ready yet" — the ticket is open; that's already
  understood
- "Verdict unchanged" — silence on an unchanged thing is the
  correct shape
- A new fix recommendation — the action plan is in the ticket body
- Restating what the ticket body already says
- Filler adjectives
- Announcements of own future operations
- The ticket's own ID
- "I"-form prose

## Anti-patterns

- ❌ Skip draft + confirmation — post immediately
- ❌ Template B without an opening "Short answer:" line — long
  comments must open with a direct one-liner
- ❌ Long comment with all technical details inline instead of a
  link to the candidate-database entry / drawer
- ❌ Numbers without a time window
- ❌ Blockquoting your own paragraphs — reads as if everything is
  someone else's quote
- ❌ Repeating what's already in the ticket body — wastes attention
- ❌ Mixing the two genres in one comment — pick one shape and use it
