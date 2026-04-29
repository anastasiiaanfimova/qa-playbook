---
name: qa-playbook-push
description: >-
  Sync local QA skills and agents to github.com/anastasiiaanfimova/qa-playbook,
  anonymizing private tool names and paths before push. Compares local vs repo,
  shows diffs, commits only changed files. Fully automatic.
  Trigger: "qa-playbook-push", "запуши qa playbook", "обнови qa-playbook", "sync qa playbook".
---

# Sync QA Playbook

Diffs local QA skills and agents against the GitHub repo, anonymizes private tool/product names,
commits changed files, and pushes.

## Config

### Repo
```
Owner:  anastasiiaanfimova
Repo:   qa-playbook
Branch: main
Local:  /tmp/qa-playbook
```

### Files to sync

**Skills** — only those listed under `qa-playbook:` in `~/.claude/skills/REGISTRY.yml` are synced. `REGISTRY.yml` is the single source of truth — do not maintain a list here.

Each skill is synced as `skills/<name>/SKILL.md`.

**Agents** — only those listed under `qa-playbook-agents:` in `~/.claude/skills/REGISTRY.yml` are synced (no anonymization needed — they're generic).

Each agent is synced as `agents/<name>.md`.

### Anonymization

Replacement rules and the privacy-scan pattern live in a **local-only** file (never synced):

```
~/.claude/skills/qa-playbook-push/replacements.md
```

To add a new private name, append a line:
```
(?i)newname → <placeholder>
```
Then update the `SCAN:` line to include it.

**Never anonymize:** `anastasiiaanfimova` (GitHub owner), generic QA terms, public tool names.

---

## Workflow

### Step 0 — Clone or pull the repo

```bash
if [ -d /tmp/qa-playbook/.git ]; then
  git -C /tmp/qa-playbook pull --rebase --quiet
else
  git clone https://github.com/anastasiiaanfimova/qa-playbook /tmp/qa-playbook
fi
```

### Step 1 — Build anonymization function

Save to `/tmp/qa-anon.py` once at the start of the skill run (use a different path from claude-config-push to avoid conflicts):

```python
# qa-anon.py — reads patterns from replacements.md, applies to stdin
import sys, re, os

skill_dir = os.path.expanduser('~/.claude/skills/qa-playbook-push')
repl_file = os.path.join(skill_dir, 'replacements.md')
replacements = []
with open(repl_file) as f:
    for line in f:
        line = line.strip()
        if not line or line.startswith('#') or line.startswith('SCAN:'):
            continue
        if ' → ' in line:
            pat, rep = line.split(' → ', 1)
            replacements.append((pat.strip(), rep.strip()))

text = sys.stdin.read()
for pat, rep in replacements:
    text = re.sub(pat, rep, text)
print(text, end='')
```

### Step 2 — Diff and copy skills

Read `~/.claude/skills/REGISTRY.yml` to get PUBLIC_QA_SKILLS, and PUBLIC_QA_AGENTS from Config above. Build both arrays here — reuse in Steps 2b and 2c:

```bash
PUBLIC_QA_SKILLS=($(python3 -c "
import yaml, sys
with open(sys.argv[1]) as f: reg = yaml.safe_load(f)
print(' '.join(reg.get('qa-playbook', [])))
" ~/.claude/skills/REGISTRY.yml))
PUBLIC_QA_AGENTS=($(python3 -c "
import yaml, sys
with open(sys.argv[1]) as f: reg = yaml.safe_load(f)
print(' '.join(reg.get('qa-playbook-agents', [])))
" ~/.claude/skills/REGISTRY.yml))
```

Iterate over skills:

```bash
for name in "${PUBLIC_QA_SKILLS[@]}"; do
  src="$HOME/.claude/skills/$name/SKILL.md"
  dest="/tmp/qa-playbook/skills/$name/SKILL.md"
  [ -f "$src" ] || { echo "WARNING: $src not found, skipping"; continue; }
  mkdir -p "$(dirname "$dest")"
  python3 /tmp/qa-anon.py < "$src" > /tmp/skill_anon.md
  if [ ! -f "$dest" ]; then
    cp /tmp/skill_anon.md "$dest"
    echo "NEW: $name"
  elif ! diff -q /tmp/skill_anon.md "$dest" > /dev/null 2>&1; then
    cp /tmp/skill_anon.md "$dest"
    echo "CHANGED: $name"
  fi
done
```

### Step 2b — Diff and copy agents

Agents don't need anonymization (reuse `$PUBLIC_QA_AGENTS` from Step 2):

```bash
for name in "${PUBLIC_QA_AGENTS[@]}"; do
  src="$HOME/.claude/agents/$name.md"
  dest="/tmp/qa-playbook/agents/$name.md"
  [ -f "$src" ] || { echo "WARNING: $src not found, skipping"; continue; }
  mkdir -p "$(dirname "$dest")"
  if [ ! -f "$dest" ]; then
    cp "$src" "$dest"
    echo "NEW: $name"
  elif ! diff -q "$src" "$dest" > /dev/null 2>&1; then
    cp "$src" "$dest"
    echo "CHANGED: $name"
  fi
done
```

### Step 2c — Remove stale files from repo

Delete skill dirs and agent files not in the arrays (reuse arrays from Step 2):

```bash
for dir in /tmp/qa-playbook/skills/*/; do
  skill_name=$(basename "$dir")
  found=0
  for s in "${PUBLIC_QA_SKILLS[@]}"; do [ "$s" = "$skill_name" ] && found=1 && break; done
  if [ $found -eq 0 ]; then
    git -C /tmp/qa-playbook rm -r "skills/$skill_name"
    echo "DELETED stale skill: $skill_name"
  fi
done

for f in /tmp/qa-playbook/agents/*.md; do
  agent_name=$(basename "$f" .md)
  found=0
  for a in "${PUBLIC_QA_AGENTS[@]}"; do [ "$a" = "$agent_name" ] && found=1 && break; done
  if [ $found -eq 0 ]; then
    git -C /tmp/qa-playbook rm "agents/$agent_name.md"
    echo "DELETED stale agent: $agent_name"
  fi
done
```

### Step 2d — Privacy scan

Before staging anything, scan all repo files for private names that should have been anonymized:

```bash
SKILL_DIR="$HOME/.claude/skills/qa-playbook-push"
REPL_FILE="$SKILL_DIR/replacements.md"
SCAN_PATTERN=$(grep "^SCAN:" "$REPL_FILE" | sed 's/^SCAN: *//')

LEAKS=$(grep -rn "$SCAN_PATTERN" /tmp/qa-playbook/ \
  --include="*.md" --include="*.json" \
  2>/dev/null | grep -v ".git" | grep -v "anastasiiaanfimova" | grep -v "/README.md:")

if [ -n "$LEAKS" ]; then
  echo "PRIVACY LEAK DETECTED — aborting push:"
  echo "$LEAKS"
  echo "Fix replacements.md patterns and re-run."
  exit 1
fi
echo "Privacy scan: clean"
```

If any leaks are found → **stop immediately**, do not commit. Report which file and line.

### Step 3 — Review and confirm

```bash
git -C /tmp/qa-playbook status --short
```

If output is empty → print "qa-playbook is up to date, nothing to push." and stop.

If changes exist → show the user what will be committed:
- List all changed files (`M` = modified, `A` = new, `D` = deleted)
- For each modified skill show a compact diff
- Call out any deleted stale files explicitly

**Then ask:** "Всё выглядит хорошо? Коммитить и пушить?" — and wait for confirmation.

### Step 4 — Update README.md (mandatory, before commit)

**Always run this step** — even if README.md itself wasn't in the diff.

Read `/tmp/qa-playbook/README.md` and verify these sections match current local state:

**Completeness check:**

| Repo dir | README section | Row key | How to add a new row |
|---|---|---|---|
| `skills/` | `### Skills` table | dir name | read `SKILL.md` description field |
| `agents/` | agents table | filename without `.md` | read agent file for description + model |

Rules:
- **File in repo but no row in README** → add the row
- **Row in README but no file in repo** → remove the row
- **File content changed** → verify the row description still matches

**Link rule:** every tool mentioned must have a working URL if one exists. Never use placeholder links.

Edit `/tmp/qa-playbook/README.md` directly for any outdated sections. README update is part of the same commit.

### Step 5 — Commit and push

```bash
cd /tmp/qa-playbook
git add -A

CHANGED_FILES=$(git diff --cached --name-only | tr '\n' ' ')
git commit -m "sync qa-playbook $(date +%Y-%m-%d): $CHANGED_FILES"

git push origin main
```

Print the commit hash:
```bash
git -C /tmp/qa-playbook log --oneline -1
```

### Step 6 — Write diary entry

`mcp__mempalace__mempalace_diary_write` with compact AAAK entry:
- `agent_name`: `"claude"`
- `wing`: `"wing_claude-<product>"`
- `topic`: `"qa-playbook.sync"`
- `entry`: include — which files changed (skills / agents), commit hash, anonymizations applied (e.g. "replaced <product>×5"), whether README was updated

---

## Notes

- **Always anonymize skills before diffing** — run through qa-anon.py even if the file looks clean.
- **Agents are NOT anonymized** — they're generic and have no private tool references.
- Uses `/tmp/qa-anon.py` (not `/tmp/anon.py`) to avoid conflicts with claude-config-push runs.
- `replacements.md` is **local-only** — it's never synced to qa-playbook.
- To add a skill: add it to the `qa-playbook:` section in `~/.claude/skills/REGISTRY.yml` — that's the only place to edit. Step 2 reads the list from there at runtime.
- To add an agent: add it to the `qa-playbook-agents:` section in `~/.claude/skills/REGISTRY.yml` — same rule.
- Before adding an agent to PUBLIC_QA_AGENTS: verify it has no private references first (grep for product names).
