---
name: git-diff
description: >-
  QA analysis of git changes. Branch mode: deep read of feature branch,
  risk-categorized, cross-referenced with Bug Candidates DB and regression
  zones, output is <wiki> test checklist. Release mode: overview since last
  tag, output is chat integration checklist. Trigger: "/git-diff",
  "проверь ветку", "что тестировать на ветке X", "чеклист для релиза",
  "что изменилось с последнего релиза".
---

# git-diff — QA analysis of git changes

Two modes:
- `branch` — deep code analysis of a feature branch → <wiki> page with test checklist + TC IDs for a manual <tms> run
- `release` — surface-level overview of changes since last release → integration checklist in chat

## Sub-commands

| Command | What it does |
|---|---|
| `/git-diff branch [branch] [--repo back\|front\|admin\|all]` | Deep analysis of feature branch vs main |
| `/git-diff release [from-ref] [to-ref]` | Release integration checklist |

Default repo: all three. Default branch: current git branch. Default release range: last tag to HEAD.

## Repos

| Short | Path |
|---|---|
| `back` | `<product-dir>/backend` |
| `front` | `<product-dir>/frontend` |
| `admin` | `<product-dir>/admin` |

## <wiki> folder

Branch analysis pages go here (private):
- Page ID: `34f98d8a-3c9b-8124-ad7d-ed1cac7522ac`
- URL: https://www.<wiki>.so/34f98d8a3c9b8124ad7ded1cac7522ac

Page naming convention:
- Branch: `<repo> / <branch-name>` (e.g. `backend / feat-billing-webhook-fix`)
- Release: `<repo> / YYYY-MM-DD` (e.g. `backend / 2026-04-28`)

## <tms> project IDs

| Repo | Project ID |
|---|---|
| front (Web) | 1 |
| back (Back) | 2 |
| admin (Admin) | 3 |

---

## Branch mode

### Step 1 — Determine target branch and repos

```bash
# If branch not given → use current:
git -C <product-dir>/backend rev-parse --abbrev-ref HEAD
```

If `--repo` not given → run for all three repos.
If a repo has no commits on that branch → skip it with a note.

### Step 2 — Collect changed files

Per repo:
```bash
git -C <product-dir>/<repo> diff main...<branch> --name-only
```

If zero changed files in a repo → skip that repo.

### Step 3 — Categorize changes

Map file paths to functional areas:

| Path pattern | Area | Risk |
|---|---|---|
| `backend/api/handlers/billing*`, `backend/domain/billing*` | Billing / payments | P0 |
| `*credits*`, `backend/domain/credits*` | Credits system | P0 |
| `backend/infra/external/clients/*` | External service integrations | P0 |
| `backend/infra/database/psql/tables/*` | DB schema | P0 |
| `backend/api/handlers/generation*`, `backend/domain/generation*` | Generation pipeline | P1 |
| `backend/api/handlers/auth*`, `backend/domain/auth*` | Auth / sessions | P1 |
| `backend/api/handlers/trusted*`, `backend/domain/trusted*` | Trusted mode | P1 |
| `backend/api/handlers/templates*`, `backend/domain/templates*` | Templates | P2 |
| `backend/infra/external/clients/http/session/wrapper.py` | HTTP session wrapper | P1 |
| `frontend/src/` | UI / UX | P2 |
| `admin/` | Admin panel | P2 |
| Everything else | Other | P3 |

Note: a single file can map to multiple areas (e.g. a credits handler is both Credits + an API endpoint).

### Step 4 — Deep read of each changed file

For each changed file — two reads:

1. **Diff**: `git -C <product-dir>/<repo> diff main...<branch> -- <file>`
2. **File** (Read tool): read current state to understand surrounding context

Analyze and note:

**For backend API handlers:**
- Which HTTP endpoints changed (method + route)
- Whether transaction boundaries were touched (`commit()`, `rollback()`)
- Whether credit debit/add operations are in the changed path
- Whether external calls were added/removed/reordered (Stripe, Payblis, <analytics>, dbus...)
- Whether new exceptions are raised vs swallowed

**For backend domain (business logic):**
- What invariant or rule changed
- Whether idempotency guarantees still hold (does `get_by_X` check still exist before insert?)

**For DB tables:**
- New columns: nullable? has default?
- New constraints: UNIQUE, FK
- Whether the migration is reversible without data loss

**For external client code (`infra/external/clients/`):**
- Which service this is (identify the provider)
- Whether error handling changed: does it now swallow vs raise `ClientResponseError`?
- Note: `wrapper.py:35-44` logs response body at ERROR before raising — this is expected behavior, not a bug

**For frontend components/pages:**
- Which user-facing flows are affected
- Whether billing/credit UI was touched
- New form fields, validation changes

**For admin panel:**
- Which admin features changed
- Whether any data-mutation actions were added

### Step 5 — Cross-reference with known risk zones

**Bug Candidates DB** — check for active issues in affected areas:

```
mcp__notion__notion-search(
  query="<affected area keyword>",
  filters={},
  data_source_url="collection://26b9b9ff-6319-4e88-af44-b30a6978600d"
)
```

For each result: fetch and verify it's Active status. If an active bug candidate overlaps with a changed area → flag it: "⚠️ Known active issue in this area".

**Regression zones to always check when the area is touched:**

| Area | What to verify |
|---|---|
| Credits | Debit/credit race conditions, idempotency (duplicate prevention), balance consistency |
| Generation | POST /assets/batch — 1640-error threshold, 33s timeout boundary |
| Trusted mode | Status transitions: pending → active → suspended |
| Templates | Aspect ratio rendering (common regression: DEV-1320, 1490, 1560) |
| Any new AI model/tool | Full smoke: submit → poll → assert result |

### Step 6 — Search existing TCs in <tms>

For each affected area + each repo:

```
mcp__<tms>__<tms>_list_testcases(project_id=<id>)
```

Filter by folder name or title keywords that match the area (e.g. "billing", "credits", "generation"). Collect TC IDs.

Do not include TCs from unrelated areas just to pad the list.

### Step 7 — Build prioritized checklist

Order areas P0 → P1 → P2 → P3. For each area:

```
## [Area] — [N] files changed
Risk: P0/P1/P2/P3
⚠️ Known active issue: [link] (if from Step 5)

**What changed:**
- handler X: added external call to Payblis after credit debit
- table Y: new UNIQUE constraint on (user_id, provider)

**Verify:**
- [ ] [specific check based on what changed]
- [ ] [regression zone check if applicable]

**Existing TCs:** TC-123, TC-456
```

Final section: **Manual checks (no TC yet)**
All checklist items for which no TC was found in Step 6. This section makes coverage gaps explicit.

### Step 8 — Create <wiki> page

One page per repo where changes were found:

```
mcp__notion__notion-create-pages(
  parent={type: "page_id", page_id: "34f98d8a-3c9b-8124-ad7d-ed1cac7522ac"},
  pages=[{
    properties: {title: "<repo> / <branch>"},
    content: <checklist content>
  }]
)
```

Before calling — fetch `<wiki>://docs/enhanced-markdown-spec` to use correct <wiki> markdown.

Print the <wiki> page URL when done.

### Step 9 — Report TC IDs for test run

If TCs were found in Step 6:
```
Нашла N релевантных ТК: TC-123, TC-456, TC-789
Создай тест-ран в <tms> вручную и добавь их туда.
(<tms>.io → проект → Test Runs → New run → выбери ТК)
```

If no TCs found:
```
ТК для этих областей пока нет — всё в чеклисте как ручные проверки.
```

---

## Release mode

### Step 1 — Determine range

If refs not given:
```bash
# Get last tag per repo:
git -C <product-dir>/backend describe --tags --abbrev=0
```

Range: `<last-tag>..HEAD` across all three repos.
If no tags found → use `--since="7 days ago"` as fallback.

### Step 2 — Collect changes (surface level only)

Per repo — do NOT read individual file diffs:
```bash
git -C <product-dir>/<repo> log <from>..<to> --oneline
git -C <product-dir>/<repo> diff <from>..<to> --name-only
```

Aggregate: count changed files per area. Read commit messages to identify intent.

### Step 3 — Categorize at area level

Same path-to-area mapping as branch mode, but aggregate counts only.
Note which repos contributed to each area.

### Step 4 — Pull "Must-check per release" context

Read TC Gap <wiki> page (`34b98d8a-3c9b-81e7-8a16-c6d685dd0742`) — look for "Must-check per release" section. Use only the items relevant to areas that changed this release, plus any marked as "always check".

### Step 5 — Generate integration checklist in chat

```
## Release QA checklist — YYYY-MM-DD
Range: <from>..<to>

### Changed areas
- Billing: N files (back)
- Generation: N files (back, front)
- Templates: N files (back, front, admin)
...

### Integration checks by area (P0 first)
- [ ] [specific integration scenario for what changed]
...

### Regression zones to verify
(from known chronic areas, only those relevant to this release)
- [ ] Credits: debit/credit idempotency
...

### Must-check always
- [ ] Smoke: registration → topup → generate → download
- [ ] [items from TC Gap "must-check" section]
```

Output in chat only — no <wiki> page for release mode.

---

## Hard rules

- ❌ Never create <task-tracker> tasks — checklists only.
- ❌ Never skip P0 areas — billing and credits always get full analysis.
- ❌ Never list TCs from unrelated areas to inflate the run.
- ✅ Always show file count per area — gives a sense of change scope.
- ✅ Always separate "existing TCs" from "manual checks (no TC yet)" — makes coverage gaps visible.
- ✅ For release: always include "must-check always" items regardless of whether that area changed.
- ✅ Branch mode: read the actual diff AND the current file — diff alone misses context.
- ✅ Cross-reference Bug Candidates DB every time — an active bug in a touched area is critical info.
