---
name: tc-update
description: >-
  Methodology for changing existing test cases — single edits, bulk
  operations, and resolving stale entries. Centers on the rule that
  modification rights depend on the TC's confidence status, and that
  bulk operations need the same discipline as single edits, just with
  scaled output. Tool-agnostic — applies with any TMS.
---

# tc-update

A test case is a piece of trust between the person who wrote it and
everyone who reads it later. Updating one is a small renegotiation of
that trust. The methodology here keeps that explicit.

## What "update" covers

Five operation shapes, all governed by the same rules:

| Operation | Examples |
|---|---|
| Field update | priority, type, automation flag, owner |
| Folder move | reorganization across areas |
| Content edit | rewriting steps, expected results |
| Rename | clarifying a vague title |
| Mixed | "change priority and move to Billing" |

The shape doesn't change the rules; the **status of the TC being
modified** does.

## Hard rule: status determines the modification protocol

A TC's status is a confidence claim (`GUESS` < `DRAFT` < `ACTIVE`, plus
`STALE` and `ARCHIVED` as lifecycle states). Modification rights flow
from it:

| Status | Modification protocol |
|---|---|
| `GUESS` / `DRAFT` | Bulk-confirm acceptable: show the plan, one "yes" covers all |
| `ACTIVE` | Per-item confirmation required, per change. No batch shortcut. |
| `STALE` | Triage flow (see below) — never modified silently |
| `ARCHIVED` | Read-only. Edits require a deliberate un-archive first. |

The reason `ACTIVE` requires per-item confirmation: ACTIVE means a human
walked the steps and trusted the result. Bulk-flipping fields on ACTIVE
TCs without re-confirming each one re-uses that trust without earning it.

## Single vs bulk — same rules, scaled output

Bulk is single repeated, not single relaxed. The discipline scales:

```
clarify → fetch → ACTIVE check (per item) → show plan → confirm → execute → report
```

What changes between single and bulk is **density of output**, not
strictness of rules. A plan listing twenty TCs has the same level of
detail as a plan for one — just twenty rows. A confirmation skipped
quietly during bulk is the same bug as one skipped on a single edit;
the volume only hides it.

## Don't change what wasn't asked

This sounds obvious until you see how often it goes wrong:

- "Move TC-42 to Billing" — don't also bump priority because Billing TCs
  tend to be HIGH. The user asked for one move.
- "Rename TC-71" — don't reformat steps even if they're messy.
- Fixing a typo in a title is fine. Restructuring "while you're in
  there" is a separate change.

If you notice a second issue worth fixing, surface it as a question, not
as a silent extra change. The user can say yes to a follow-up; they
can't easily un-see a drift you slipped in.

## Errors: show raw, never silently skip

In bulk operations, when one item fails:

- Don't drop it from the report.
- Don't summarize it as "1 error" — show the actual response.
- Don't retry blindly.

A silent skip in a hundred-item batch is the worst class of bug: it
looks successful, the failed cases stay broken, and nobody sees the gap
until something downstream depends on the missing change.

## STALE lifecycle

`STALE` is the marker for "verified once, but the world changed." A
field event was renamed, a handler was removed, an admin page was
restructured. Something the TC was anchored to no longer matches.

### Why STALE is its own state, not just DRAFT

DRAFT means "in progress, not yet trusted." STALE means "was trusted,
but the anchor moved." They demand different review effort. A DRAFT
needs completing; a STALE needs investigating *what* changed.

### Reading the gap context first — before any decision

A STALE TC should carry a note about *why* it became stale (which event
was renamed, which handler was deleted, which feature was retired).
Read this **before** any code/system check. The note is the cheapest,
most direct signal you have. Skipping it and rediscovering the cause
from scratch is wasted work.

If your TMS doesn't have a structured "why stale" field, capture it in
the TC body when marking STALE. Future-you will need it.

### Three exits from STALE

After reading the context and verifying current product state, exactly
one of these applies:

1. **Feature changed, TC still valid** — update the steps, drop the
   status to `DRAFT` (revalidation needed), record a "last reviewed"
   timestamp.
2. **Feature gone** — archive the TC. **Per-item confirmation required**.
   Archive is operationally irreversible at scale; treat it as such.
3. **Cannot determine** — ask a specific question with the gap context
   already quoted. Don't waste the asker's time having them re-derive
   what the gap note already explained.

### Always timestamp the transition out of STALE

A "last reviewed" date is the single field that lets a future review
loop skip TCs that were just resolved. Without it, every cleanup pass
re-touches the same things.

## Anti-patterns

- ❌ Modifying `ACTIVE` silently (the cardinal sin — see `tc-create`).
- ❌ Archiving `STALE` without reading the gap context.
- ❌ Bulk-archiving without per-item confirmation.
- ❌ "While I was in there" — touching fields outside the request.
- ❌ Silent error skip in bulk operations.
- ❌ Forgetting the timestamp on STALE→DRAFT transitions (causes
  re-touching in next review).
- ❌ Treating bulk as a license to relax confirmation.

## When you should refuse to act

Push back, don't proceed, when:

- The request is "update everything in folder X to type Y" with no
  filter on status — ACTIVE TCs would be silently flipped.
- The request is "archive all STALE older than N months" — implies
  bulk archival without per-item review.
- The status of the target TCs is ambiguous from the description.

Refusing is not blocking; it's asking the next-tighter question. "Of
the 84 TCs in folder X, 12 are ACTIVE — do those need a different
treatment?" is the right next move.
