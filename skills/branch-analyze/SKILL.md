---
name: branch-analyze
description: >-
  Methodology for QA-analyzing a feature branch before testing. Walks from
  any input (ticket URL / branch name / ID) through MR/PR location, deploy
  state, paired-branch detection, diff reading, automated anomaly checks,
  and outputs a focused manual-test plan with real URLs and signal queries.
  Tool-agnostic.
---

# branch-analyze

Most "what should I test on this branch?" sessions either over-test
(every page, every flow, blind smoke) or under-test (skim the diff,
hit the happy path, hope). This skill captures the middle path: read
the actual changes, cross-reference with known regression zones,
verify the deploy state, run automated baseline-vs-feature comparisons,
and produce a list of *specific* manual flows with real URLs and
expected log queries.

The output is a test plan that earns its space — every flow has a
reason rooted in either the diff or known historical risks.

## Inputs you accept

Any of the below — the skill derives the rest:

- Ticket URL (Asana / Jira / Linear / etc.)
- Ticket ID (`DEV-1727`)
- Branch name (`fix/DEV-1727-...`)

If only a branch name without a ticket ID extractable → stop, ask.

## Stage 0 — Sync code, fetch all branches

Before anything else: refresh local clones (pull main on each repo,
update any code-graph caches if you have them). Then `git fetch
origin` on each repo so feature branches are visible for diffs.

Investigation against stale clones produces wrong diagnoses ("this
function doesn't exist") that waste an hour.

## Stage 1 — Parse the input

| Format | Extract |
|---|---|
| Tracker URL | Fetch task → extract ticket ID from name |
| `DEV-1727` | Use directly |
| `fix/DEV-1727-...` | Regex `DEV-\d+` |

Save: `TICKET_NUM` (uppercase canonical form), `task_id` (lowercase
for paths), `task_num` (digits only — used in some env URLs).

## Stage 2 — Find the MR/PR + paired-branch detection

For each repo (typical split: backend + frontend + admin), search the
VCS for MRs whose source branch contains `TICKET_NUM`. Pick the most
recent if multiple.

For each match, save: branch, URL, IID, state (`opened` /
`merged` / `closed`), and which repo.

If no MR found anywhere → stop, surface to user.

### Paired-branch detection

Branches sometimes carry a cross-repo reference: `feature/DEV-X-ref-DEV-Y`
means this MR depends on a sibling MR in the other repo. Detect via
regex on the source branch:

| Found in | What to do |
|---|---|
| Frontend only, with `-ref-Y` | Paired. Frontend = X, backend = Y. Fetch the backend MR for Y. |
| Frontend only, no `-ref-` | Standalone frontend; backend on review env runs `main` |
| Backend only, with `-ref-Y` | Paired. Backend = X, frontend = Y. Fetch the frontend MR for Y. |
| Backend only, no `-ref-` | Standalone backend; frontend on review env runs `main` |
| Both repos, same ticket | Both are part of one task — use both |
| Admin only | Admin typically has no review env; ask user about fallback |

Important consequence: when paired, env URLs depend on the *correct*
task ID per repo. The admin URL needs `back_task_num`; the frontend
URL needs `front_task_id`. Mixing produces an admin connected to the
wrong backend.

### Fallback when VCS API is down

Use local git: `git branch -r | grep DEV-<num>`, then take the most
recent commit timestamp per match. **Never guess the active branch
by name alone** — always by timestamp. Old branches with the same
ticket prefix often linger after refactor renames; picking the
abandoned one quietly skips the actual work.

## Stage 3 — Deploy state determines test environment

| MR state | Pipeline | Env to test |
|---|---|---|
| `merged` | — | Common stage |
| `closed` | — | Ask user: skip or test on stage |
| `opened` | `success` | Per-feature review env |
| `opened` | `running` / `pending` | Stop; tell user which pipeline is in flight |
| `opened` | `failed` / `canceled` | Stop; surface pipeline URL |
| `opened` | null | Stop; pipeline never ran |

For paired branches, deploy is ready only when **both** pipelines
succeed; deploy time = `max(front, back)`.

## Stage 4 — Read the ticket

Don't analyze code first — analyze the *intent*. Read:

1. Task description / acceptance criteria
2. Comments — especially developer comments explaining tradeoffs
3. **Parent task** if one exists — often holds the broader spec,
   design rationale, and AC missing from the child

Context from a parent often reframes what the diff is doing.

## Stage 5 — Diff each MR

```
git diff origin/main...origin/<branch> --name-only
```

Zero files → skip that repo. For paired, read both diffs — frontend
and backend changes typically complement each other; test flows
must cover both halves.

## Stage 6 — Categorize files by risk

Group changed files by area and assign risk priority. The list below
is illustrative — your own list comes from your codebase, but the
priority axes generalize:

| Area type | Priority | Why |
|---|---|---|
| Money flows (billing, credits, payment integrations) | P0 | Direct revenue / data integrity |
| External integrations (third-party clients) | P0 | Cross-system contract |
| DB schema (migrations, table definitions) | P0 | Data shape changes are hard to reverse |
| Auth | P1 | Security boundary |
| Core feature handlers | P1 | High user-visible impact |
| Templates / configurations | P2 | Behavior tweaks |
| UI changes | P2 | Visual / layout |
| Other | P3 | Background |

The categorization drives the order of test flows in the plan and
which automated checks must run before manual testing.

## Stage 7 — Read each changed file fully

Diff alone misses context. For each touched file: `git diff` then
`Read` the whole file. Things to look for:

- **Backend handlers:** transaction boundaries (where commit/rollback
  fall), credit operations, external-call additions, exceptions
  raised vs swallowed
- **Backend domain logic:** invariant changes, idempotency
  guarantees (lookup-before-insert patterns)
- **DB tables:** new columns (nullable? default?), constraints
  (UNIQUE / FK), migration reversibility
- **External clients:** error handling — does this client swallow
  with a warning, or raise to the caller?
- **Frontend:** affected user flows, validation changes, analytics
  event tracking (grep the diff for `track`/`logEvent` patterns —
  new events flagged for verification, changed args flagged as
  potential analytics regressions)
- **Admin:** new mutations, changed permissions

## Stage 8 — Cross-check existing bug candidates

Search the bug-candidates database for keywords from the changed
areas. Active candidates touching the same area → flag in the test
plan as ⚠️.

### Always-check regression zones

Some areas have a history of regressions and deserve a smoke pass
on every touch, regardless of the diff:

| Touched area | Always check |
|---|---|
| Money / credits | Race conditions, idempotency, balance after concurrent ops |
| Generation / batch processing | Load thresholds (specific to your system), timeout boundaries |
| Trust / role transitions | State machine: pending → active → suspended |
| Templates with parameters (aspect ratio, etc.) | Chronic regression — verify all parameter combinations |
| New AI tool / generation pipeline | Full smoke: submit → poll → result |
| UI changes | Mobile breakpoint (e.g. 375px) — text fitting, button reachability, panel overflow |
| Color / theme changes | Both light and dark modes |

The "regression zones" list grows from incident history. Every time
a regression slips through, add the zone with a note.

## Stage 9 — Automated baseline-vs-feature anomaly checks

The naive "show me errors on the feature env" produces noise — same
errors are also on stage and main. The signal is **anomaly relative
to baseline**, not raw count.

### Logs anomaly detection

1. **Query feature env** for ERROR-level events since deploy time
2. **Query stage env** over the same window length
3. **Group both sides** by normalized pattern (e.g. `<METHOD> <path>
   -> <status>`, with IDs in URLs replaced by `{id}`)
4. **Compare per pattern:**

```
ratio = count_feature / max(count_stage, 1)

if both counts < MIN_COUNT:
    silent (too small for statistics)
elif pattern only on feature, count_feature >= MIN_COUNT:
    ⚠️ "new on feature: {pattern} ({count} events)"
elif ratio >= HIGH_THRESHOLD (e.g. 3.0):
    ⚠️ "rose: {pattern} ({feature} vs {stage}, {ratio}×)"
elif ratio <= LOW_THRESHOLD (e.g. 0.3):
    ⚠️ "dropped: {pattern} ({feature} vs {stage})"
else:
    silent (background)
```

`MIN_COUNT` ≈ 10 keeps single-event flukes from triggering. Output
only the anomalies in the test plan; the full pattern list is noise.

### Worker / queue logs

If generation / async tasks are touched: same comparison for
worker-stream logs. A worker missing on the feature env is a
deployment problem worth flagging.

### Error monitoring

Filter to staging environment explicitly. Issues with `firstSeen >=
deploy_time` are candidates for "new since this branch" — but
verify they're tied to changed code, not coincidental.

## Stage 10 — Compose test flows

Translate the categorization (Stage 6) + deep-read findings (Stage 7)
+ ticket context (Stage 4) into specific user flows.

Rules for a flow:

- One flow = one scenario with concrete URL, action, expected result
- Order: P0 → P1 → P2
- Inline log query for each flow ("after step 3, run query X — expect
  no errors matching Y")
- For analytics-tracked actions (Stage 7 grep) — add an "analytics
  check" step
- Don't duplicate Stage 9 automated checks — only manual things

What's not a flow:

- "Open the page" (that's a precondition for many flows)
- Generic OAuth / login (only include if auth itself is being
  tested)

## Stage 11 — Output

A structured document with:

- Title with ticket ID + name
- Environment URLs (feature env or stage)
- MR link(s); for paired, both with cross-reference note
- Automated check results (one line per source if clean; table for
  anomalies)
- Flows P0 → P1 → P2
- Regression-zone smoke list
- "Test on stage" section only if auth/login flows require it
- Optional: link to UI snapshots / design baseline if the change is
  visual

The output isn't a generic checklist — it's calibrated to *this*
branch, *this* diff, *this* deploy state.

## Hard rules

- ❌ Don't create tracker tickets from this skill — output is a
  test plan, not new work
- ❌ Don't skip P0 areas (money flows are always fully covered)
- ❌ Don't proceed with `pipeline=failed` — surface and stop
- ❌ Don't auto-stop on `MR=closed` — ask the user (sometimes you
  test on stage anyway)
- ✅ Always: diff + full file read + automated checks before manual
  flows
- ✅ `merged` → stage env. `opened+success` → feature env. `closed`
  → ask
- ✅ Use anomaly comparison, not raw error filter, for log checks
- ✅ Bug-candidates cross-check is mandatory
- ✅ Reuse the ticket fetched in Stage 1 — don't re-fetch
- ✅ Paired branches → fetch the sibling MR; use the correct
  per-repo task ID in env URLs

## Anti-patterns

- ❌ "Generic smoke" plan with no diff-rooted flows — the test plan
  must explain why each flow exists
- ❌ Reading only the diff without opening files — diffs hide
  context (try/except shape, transaction boundaries)
- ❌ Listing every error from feature env as a regression — anomaly
  comparison filters baseline noise
- ❌ Picking branch by name when multiple match `DEV-X` — always
  use the most recent commit timestamp
- ❌ Mixing paired front/back IDs in one URL — admin connects to
  wrong backend
- ❌ Test flows that re-test what automated checks already cover
- ❌ Manual flow steps that don't include the log query to check
  afterwards
