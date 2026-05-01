---
name: qa-playbook-push
description: >-
  Sync local QA skills and agents to github.com/anastasiiaanfimova/qa-playbook,
  anonymizing private tool names and paths before push. Compares local vs repo,
  shows diffs, commits only changed files. Fully automatic.
  Trigger: "qa-playbook-push", "запуши qa playbook", "обнови qa-playbook", "sync qa playbook".
---

# Sync QA Playbook

Diffs local QA skills and agents against the GitHub repo, anonymizes private
tool/product names, commits changed files, and pushes.

The deterministic part (clone/pull, anonymize, diff, copy, stale-removal,
privacy scan, commit, push) lives in shared scripts under
`~/.claude/lib/push-mirror/`. This skill orchestrates them and handles the
parts that need human judgement (review, README update, diary).

## Constants

```
TARGET           = qa-playbook
PUSH_MIRROR_DIR  = ~/.claude/lib/push-mirror
PUSH_MIRROR_SH   = ~/.claude/lib/push-mirror/push-mirror.sh
COMMIT_PUSH_SH   = ~/.claude/lib/push-mirror/commit-and-push.sh
REPO_LOCAL       = /tmp/qa-playbook
REPO_URL         = https://github.com/anastasiiaanfimova/qa-playbook
DIARY_WING       = wing_claude-<product>
DIARY_TOPIC      = qa-playbook.sync
```

Target sources, registry sections, and anonymization toggles are declared in
`$PUSH_MIRROR_DIR/configs/qa-playbook.sh` — single source of truth, do not
duplicate here.

## Workflow

### Step 1 — Run the sync script

```bash
~/.claude/lib/push-mirror/push-mirror.sh --target qa-playbook
```

The script:
- clones or pulls `$REPO_LOCAL` (skips pull if local has uncommitted changes from a previous run)
- reads `~/.claude/skills/REGISTRY.yml` — sections `qa-playbook` (skills) and `qa-playbook-agents` (agents)
- anonymizes skills via `anon.py` using `replacements/qa-playbook.md`
- copies skills as `skills/<name>/SKILL.md`
- copies agents as `agents/<name>.md` **without anonymization** (`ANONYMIZE_AGENTS=false` for this target — agents are generic, no private references)
- removes stale skills/agents from the repo (anything in repo but not in registry)
- runs the privacy scan against `forbidden.txt` — exits 1 on any forbidden pattern

**If the script exits non-zero** — stop. Read the error, fix `replacements/qa-playbook.md` (add missing rule) or `forbidden.txt`, re-run.

**If "qa-playbook is up to date — nothing to commit"** — stop. No commit, no diary.

### Step 2 — Review changes with the user

```bash
git -C /tmp/qa-playbook status --short
```

Show the user what will be committed:
- changed files (`M` / `A` / `D`)
- compact diff per modified skill (the actual content delta)
- explicit callouts for deletions

**Then ask:** "Всё выглядит хорошо? Коммитить и пушить?" — wait for confirmation before Step 3.

### Step 3 — Update README.md (mandatory, before commit)

**Always run this step** — even if README.md itself wasn't in the diff.

Read `/tmp/qa-playbook/README.md` and verify these sections match current local state:

| Repo dir | README section | Row key | How to add a new row |
|---|---|---|---|
| `skills/` | `### Skills` table | dir name | read `SKILL.md` description field |
| `agents/` | agents table | filename without `.md` | read agent file for description + model |

Rules:
- File in repo but no row in README → add the row
- Row in README but no file in repo → remove the row
- File content changed → verify the row description still matches

**Link rule:** every tool mentioned must have a working URL if one exists. Never use placeholder links.

Edit `/tmp/qa-playbook/README.md` directly. README update is part of the same commit.

### Step 4 — Commit and push

```bash
~/.claude/lib/push-mirror/commit-and-push.sh --target qa-playbook
```

The script stages all changes, runs the global pre-commit hook explicitly (catches any leak that slipped past replacements), commits with `sync qa-playbook YYYY-MM-DD: <files>`, pushes to `main`, and prints the new commit hash.

If the pre-commit hook blocks → fix the leak in `replacements/qa-playbook.md` and re-run from Step 1.

### Step 5 — Diary entry

`mempalace_diary_write` with:
- `agent_name`: `"claude"`
- `wing`: `wing_claude-<product>`
- `topic`: `qa-playbook.sync`
- `entry`: AAAK — files changed (skills / agents), commit hash, anonymizations applied (e.g. "replaced <product>×5"), whether README was updated

---

## Notes

- **`replacements/qa-playbook.md`** — local-only (gitignored). Includes both privacy rules (mirroring `forbidden.txt`) and vendor-neutralization rules (<error-monitoring> → `<error-monitoring>`, <tms> → `<tms>`, etc. — playbook is vendor-agnostic).
- **`forbidden.txt`** is the master deny-list — used by the global pre-commit hook for any commit to a public anastasiiaanfimova/* repo. After anon, the privacy scan greps the repo dir against this file.
- **Agents are NOT anonymized** — they're generic. Before adding an agent to `qa-playbook-agents:` in REGISTRY.yml, verify it has no private references (grep for product names).
- **To add a skill:** add it to the `qa-playbook:` section in `~/.claude/skills/REGISTRY.yml` — that's the only edit needed.
- **To add an agent:** add it to the `qa-playbook-agents:` section.
