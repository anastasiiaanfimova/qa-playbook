---
name: tc-plan
description: >-
  Methodology for managing test plan composition — when a plan is best
  modeled as a derived view over the test corpus instead of a curated
  list, and the two-phase preview/apply pattern that keeps bulk
  membership changes safe. Tool-agnostic — applies with any TMS that
  supports a custom field on TCs and per-plan membership operations.
---

# tc-plan

Most teams treat a test plan as a hand-curated list of test cases. That
works at small scale and breaks invisibly at larger scale: TCs drift in
and out of relevance as features evolve, but the plan keeps pointing at
yesterday's snapshot. This methodology models a plan as a **derived
view** over the corpus, with explicit criteria and a safe update
protocol.

## A plan is a query, not a list

Each plan should have **formal criteria** that define what belongs in
it. Examples:

| Plan type | Criteria shape |
|---|---|
| `Smoke` | All ACTIVE/DRAFT TCs across products with `type=SMOKE`, priority HIGH |
| `Regression` | All ACTIVE/DRAFT TCs with `type=REGRESSION` |
| `Regression-<area>` | `Regression` filtered to the folders covering that area |
| `Release-<version>` | TCs touched since a date / linked to specific tickets |

Without explicit criteria, the plan accumulates whatever someone
remembered to add and never reflects deletions. With criteria, the plan
is reproducible: anyone running the same query gets the same set.

## Drift is the default — explicit reconciliation is the cure

If you accept the "plan as query" model, every plan will drift between
runs. New TCs land that match the criteria. Old TCs change status (an
ACTIVE goes to STALE, a DRAFT gets archived). The plan composition
needs to follow.

Reconciliation is the act of comparing **current plan composition**
against **what the criteria say it should be** and producing three
buckets:

- **ADD** — matches criteria, not currently in plan
- **REMOVE** — currently in plan, no longer matches criteria
- **No change** — in plan and still matches

Rebuilding the plan from scratch (drop all, add fresh) loses history,
breaks linked test runs, and triggers churn in any audit trail. The
diff approach preserves identity and minimizes change.

## The two-phase pattern — stage, then apply

Bulk membership changes are one of the easiest places to silently break
test runs. A plan can be referenced by automation, by a release
checklist, by other people's saved searches. A surprise change of 30
TCs in a Smoke plan cascades in unpredictable ways.

The methodology splits the work into two phases:

### Phase 1: Preview (stage the diff via a visible marker)

Compute the diff (ADD / REMOVE / no-change). Don't touch plan
membership. Instead, **mark each affected TC with a custom field** that
records the intent:

- `marker = "Add"` — TC is staged to enter the plan
- `marker = "Remove"` — TC is staged to leave the plan

The marker field is the staging mechanism. Anyone opening the TC in
the TMS UI sees it's pending a plan change. Anyone reviewing the plan
can run a saved search to see all marked TCs and cross-check before the
operation lands.

Then output a summary:

```
Plan: <name>
+ Add (N): list of titles or IDs
− Remove (M): list of titles or IDs
= Unchanged: K TCs

Review the markers in TMS, then run apply when ready.
```

Stop here. Wait for human review.

### Phase 2: Apply (commit + clear)

Re-fetch all TCs with non-empty marker for this plan. Show the
final list one more time:

```
Applying to plan <name>:
+ Adding: list
− Removing: list

Confirm? (yes/no)
```

Wait for explicit confirmation. Then for each item:

1. Make the membership change (add or remove).
2. **Clear the marker field** on that TC.

Report the result with counts and any per-item failures shown raw.

The marker-clearing step is non-negotiable: a leftover marker means the
plan is in a half-applied state, which is worse than either fully
applied or fully not.

## Hard rules

- **One plan at a time.** Never batch across multiple plans. One plan's
  criteria can mask another's; errors compound and become hard to
  attribute.
- **Apply requires a preview.** Preview is the staging gate; skipping
  it removes the human-review checkpoint that the whole pattern exists
  to provide.
- **Confirmation before apply.** Even after preview, show the resolved
  final list and wait for explicit "yes." The interval between preview
  and apply is exactly when someone notices a wrong-looking change.
- **Marker is plan-scoped.** Each plan needs its own marker (or each
  marker carries a plan reference). Two plans sharing one marker field
  collide on TCs that belong to both reconciliations.
- **Never modify TC content during plan ops.** A plan reconciliation
  changes membership and the marker field. Nothing else. Steps,
  priorities, statuses are out of scope.

## When the plan must exist before reconciliation

A first reconciliation against a plan that doesn't exist yet should
**create the plan empty** and then run the normal flow. The same diff
will be all-ADD, no-REMOVE — which is the correct first-fill behavior.
Don't conflate "create a plan" with "populate it"; the discipline is
the same diff-stage-apply path.

## When the criteria themselves change

If you decide a plan's criteria are wrong (e.g., Smoke should now
include MEDIUM priority too), that's a **separate decision**, made
explicitly:

1. Update the recorded criteria for the plan (in your team's source of
   truth — the playbook, a doc, a config).
2. Run a fresh preview against the new criteria.
3. The preview will surface a large diff. Treat it as you would any
   large diff — don't apply blind.

Don't quietly broaden criteria during a routine reconciliation. The
preview/apply protocol protects you from accidental composition
changes; it doesn't protect you from accidental criteria changes.

## Anti-patterns

- ❌ Hand-curated plan with no recorded criteria — drift becomes
  invisible.
- ❌ Apply without preview ("I know what I'm doing").
- ❌ Batching across plans in one operation.
- ❌ Forgetting to clear the marker after apply — plan stuck in
  half-applied state.
- ❌ Drop-and-rebuild instead of diff (loses history, breaks linked
  runs).
- ❌ Sneaking criteria changes into a routine reconciliation.
- ❌ Treating the preview output as the final action and skipping the
  apply confirmation.

## Why this is worth the ceremony

A test plan is a coordination artifact between the people who write
TCs, the people who run them, and the systems that automate them. The
two-phase pattern adds friction to a class of changes that benefits
from friction:

- The marker creates an in-TMS audit trail that survives the agent
  session.
- The preview/apply split puts a deliberate human checkpoint between
  intent and effect.
- Single-plan scoping keeps mistakes blast-radius small.

For a small plan you might never need this. For any plan that
release/automation/regression flows depend on, the ceremony pays back
the first time it catches a wrong reconciliation before it ships.
