---
name: refresh-git
description: >-
  Pull latest changes from main branch for all configured <product> repos
  and incrementally update the code-review-graph for each.
  Run at the start of a session before code analysis or QA work.
  Trigger: "/refresh-git", "обнови код", "подтяни изменения", "свежий код".
version: 0.1.0
---

# Refresh — Sync Code & Update Graphs

Pull `main` on all repos and rebuild the knowledge graphs incrementally.

## Configuration

Set the actual paths to your repos:

| Alias | Path |
|---|---|
| backend | `<backend-repo>` |
| frontend | `<frontend-repo>` |
| admin | `<admin-repo>` |

Replace `<backend-repo>`, `<frontend-repo>`, `<admin-repo>` with absolute paths to your local repositories (e.g. `~/MyProduct/backend`).

## Workflow

### Step 1 — Pull each repo

For each repo, run in sequence:

```bash
cd <backend-repo> && git checkout main && git pull
cd <frontend-repo> && git checkout main && git pull
cd <admin-repo> && git checkout main && git pull
```

Capture output per repo: how many commits pulled (or "Already up to date.").

### Step 2 — Update graphs

For each repo that pulled **new commits** (not "Already up to date"), run:

```bash
cd <backend-repo> && code-review-graph update
cd <frontend-repo> && code-review-graph update
cd <admin-repo> && code-review-graph update
```

If a repo was already up to date — skip graph update for it (no new files to process).

### Step 3 — Report

Print a compact summary:

```
## Refresh — YYYY-MM-DD HH:MM

| Repo     | Branch | Commits pulled | Graph updated |
|----------|--------|----------------|---------------|
| backend  | main   | 3 new          | ✓             |
| frontend | main   | Already up to date | —         |
| admin    | main   | 1 new          | ✓             |

Ready to work.
```

## Rules

- Always checkout `main` before pulling — don't pull whatever branch happens to be active
- If `git pull` fails (e.g. merge conflict, auth error) — report the error for that repo and continue with the others
- If `code-review-graph update` fails — report it but don't block the summary
- Do not run `code-review-graph build` (full rebuild) — always use `update` (incremental)
- No diary entry needed for routine refresh
