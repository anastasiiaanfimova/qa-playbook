---
name: daily
description: >-
  Write today's daily log and push it to your QA notes repository.
  Trigger: "/daily", "write daily", "daily log", "напиши дейли", "дейли за сегодня".
version: 0.1.0
---

# /daily — write today's daily log

Write a concise daily log based on today's conversation and push it to your notes repository.

## Configuration

Set these before use:

| What | Where |
|---|---|
| Target repository | Clone to `/tmp/<your-qa-repo>/`, set remote to your repo URL |
| Output path | `daily/YYYY-MM-DD.md` inside the cloned repo |

## Style rules

- Brief and specific, written in first person
- Claude is a tool, not the author — don't attribute work to it
- Don't mention the QA repository or tooling internals unless they came up in the conversation today
- Don't pull items from a QA checklist or playbook if they weren't discussed today
- For tomorrow's plan — add sub-notes under each item if there are specifics

## File structure

```markdown
# Daily — YYYY-MM-DD

## Done

- ...

## Blockers

- ...

## Tomorrow

- [ ] ...
  → ...
```

## Process

1. Review today's conversation and extract what was actually done
2. If anything is unclear or uncertain — **ask first** before writing:
   - Should this action be mentioned?
   - How to phrase it more precisely?
   - Are there blockers or tomorrow's plans you don't know about?
3. After clarifications — write the file to `/tmp/<your-qa-repo>/daily/YYYY-MM-DD.md` and push to GitHub

Use `currentDate` from context for today's date, or ask.

## Push

```bash
if [ -d /tmp/<your-qa-repo>/.git ]; then
  git -C /tmp/<your-qa-repo> pull --rebase --quiet
else
  git clone https://github.com/<your-org>/<your-qa-repo> /tmp/<your-qa-repo>
fi

# write the file, then:
git -C /tmp/<your-qa-repo> add daily/YYYY-MM-DD.md
git -C /tmp/<your-qa-repo> commit -m "daily YYYY-MM-DD"
git -C /tmp/<your-qa-repo> push origin main
```
