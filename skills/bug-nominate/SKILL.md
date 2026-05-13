---
name: bug-nominate
description: >-
  Methodology for the "single writer" pattern when persisting bug candidates
  to a durable knowledge base. Separates investigation from durable write,
  protects investigations from routine metadata bumps, handles fingerprint-
  based deduplication, and supports both interactive and silent batch modes.
  Tool-agnostic.
---

# bug-nominate

A bug knowledge base accumulates value over time only if writes are
disciplined. This skill captures the discipline as a "single writer"
pattern: one place where bug candidates are persisted, called by both
investigation flows (interactive) and weekly review batches (silent).

## Why a single writer

Investigation, batch review, and direct user input are three different
front-ends. They produce different shapes of evidence and run on
different rhythms. If each writes to the database independently:

- Schema drifts (different fields from different sources)
- Investigations get overwritten by routine "bump" updates
- Updates miss properties that another caller would have set
- No single place to enforce confirmation discipline

Solution: one skill owns writes. Other flows hand it inputs and
delegate. The schema, dedup logic, and protection rules live in one
place.

## Inputs (any combination)

| Field | Required | Purpose |
|---|---|---|
| `title` | yes | Page title, one line |
| `fingerprint` | yes | Deterministic ID for dedup (`<source>:<id>`, e.g. `error-monitor:PROJ-3FQZ`, `tracker:1234567`, `manual:short-slug`) |
| `status` | yes | Lifecycle state of the candidate |
| `verdict` | recommended | One of the four bug-dig verdicts (real / noise / theoretical / protected) |
| `user_impact` | recommended | yes / no / unknown |
| `severity` | recommended | Critical / High / Medium / Low |
| `sources` | recommended | Which signal sources support this candidate |
| `body_markdown` | optional | Full investigation. If present → written to page content. If absent → see protection rule below. |
| `silent` | optional, default `false` | When `true`, skip draft-and-confirm; used by batch flows |

If the chat already has a complete verdict from an investigation flow,
pull values from there rather than asking the user to repeat.

## Body lives in content, not properties

The full investigation goes into the page **content** (Markdown body),
not into properties. Properties carry indexable metadata (status,
severity, sources, dates) for filtering and aggregation. Body carries
prose (symptom, root cause, action items, evidence) for reading.

Why: properties have type constraints and length limits unsuitable for
prose; bodies allow rich Markdown structure. Mixing produces awkward
truncated descriptions in property cells and unfilterable text in
bodies.

### Canonical body template

```
## Symptom
<what user sees / what's broken — 1-2 paragraphs>

## Verdict
<one line: verdict + reason>

## Signal refs
<bulleted: error IDs, ticket IDs, log queries, commit hashes, file paths>

## Root Cause
<paragraph(s) — why it happens, with code references>

## Action items
<directive list, product language. The knowledge base allows multiple
fix options for discussion; the chosen one moves to the tracker via a
separate ticket-creation skill.>

## User Impact
<paragraph — counts, financial, UX>

## Source data
<bulleted: queries / files / commits the investigation drew on>
```

Sections can be omitted when not relevant. Preventive findings have
no User Impact; risk signals have no Action items; first-pass
candidates from batch review have only Symptom + Signal refs.

### Minimal body for batch creates

When the batch review surfaces a new signal *before* investigation
exists, write a minimal body — just enough that the entry isn't blank:

```
## Symptom
<title or one line from source>

## Signal refs
<source refs>
```

A later investigation flow will fill in the rest.

## Workflow

### Step 1 — Resolve mode

If `silent=true` → skip the draft-and-confirm step.
If `silent=false` (default) → continue.

### Step 2 — Draft to chat (interactive mode only)

Show:

- Title, fingerprint, status, verdict, user impact, severity
- First 2-3 lines of body (Symptom + Verdict)
- Detected mode: `CREATE` / `UPDATE properties only` / `UPDATE + replace body`

Wait for explicit confirmation. No write before.

### Step 3 — Find existing by fingerprint

Search the candidates database for the fingerprint. Note that most
search APIs are semantic, not exact-match — partial fingerprint
overlap is fine if title + sources also match.

Found → Step 4. Not found → Step 5.

### Step 4 — UPDATE existing

Update **properties** that change with each surfacing:

- `Last seen` → today
- `Weeks seen` += 1, but **only if the ISO week changed** since the
  previous last-seen. This makes the update idempotent for multiple
  runs in the same week.
- `Status`, `Verdict`, `User Impact`, `Severity` → overwrite with
  fresh values
- `Sources` → merge with existing (don't lose previous sources)
- `Tracker link` → set if newly available
- `Fingerprint` → may be overwritten if format changed

Do **not** touch:

- `First seen` — that's the historical anchor
- `Trend` — owned by the batch review flow, not individual writes

**Body protection rule.** If `body_markdown` was passed → replace the
content. If not → leave content alone. This protects investigations
written by deep-dive flows from being clobbered by routine weekly
bumps from batch review.

### Step 5 — CREATE new

Set the full property set:

- All metadata from inputs
- `First seen` = today
- `Last seen` = today
- `Weeks seen` = 1
- `Trend` = "New"
- Body content = `body_markdown` if provided, else minimal template

### Step 6 — Report (always, even silent)

```
✓ Candidates: <CREATE|UPDATE> "<title>"
   url: <page url>
   verdict: <verdict>, severity: <severity>, user impact: <yes/no>
```

Reporting on every write — including silent batch operations — gives
the user visibility into what changed without forcing them to open
the database. Don't suppress.

## Caller patterns

| Caller | Mode | What they pass |
|---|---|---|
| Investigation skill (deep-dive verdict) | Interactive | Full property set + canonical body |
| Batch review (weekly signal refresh) | Silent | Properties only; no body for existing entries; minimal body for new |
| Direct user invocation | Interactive | Whatever the user has in mind; ask for missing |

## Hard rules

- ❌ Don't write without inputs in current conversation or explicit
  args
- ❌ Don't create tracker tickets — that's a separate skill
- ❌ Don't overwrite body on UPDATE unless `body_markdown` was
  explicitly passed (protects investigations)
- ❌ Don't touch `First seen` or `Trend` on UPDATE
- ✅ Draft and confirmation before write in interactive mode
- ✅ Investigation prose lives in body; metadata in properties
- ✅ Always report to chat, even in silent mode

## Anti-patterns

- ❌ Multiple skills writing to the same database directly — schema
  drift inevitable
- ❌ Storing investigation text as a property — truncates, can't
  format, can't search
- ❌ Bumping "Weeks seen" daily instead of per-ISO-week — produces
  meaningless inflated counts
- ❌ Treating dedup search as exact-match when API is semantic —
  misses fingerprint variants
- ❌ Creating duplicate entries when partial fingerprint match exists
- ❌ Silent batch operations with no chat report — user has no
  visibility
- ❌ Mixing investigation flow with persistence flow in one skill —
  two responsibilities, conflate at your peril
