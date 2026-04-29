---
name: bug-comment
description: >-
  Post a short follow-up comment on an existing <product> <task-tracker> bug task —
  fresh numbers and what changed, nothing else. Use when re-checking an open
  ticket after bug-dig or during weekly review. Trigger: "обнови задачу",
  "добавь свежие цифры", "откомментируй задачу X", "обнови цифры".
  Always show draft first. NEVER post without explicit user confirmation.
---

# Bug Comment — fresh numbers on existing <task-tracker> task

Use `asana_create_task_story`, not `asana_create_task` or `asana_update_task`.

**HARD RULE: show the draft in chat and wait for "да / пиши / ок" before posting.**

---

## Template

```
Fresh numbers (<date>)

<источник>:
  <метрика>: <prev> → <new> (<+delta>)
  <метрика>: <prev> → <new> (<+delta>)
```

Источник — что угодно: <error-monitoring>, <logs>, <analytics>, <metrics>. Писать только то, что реально менялось. Если показатель не изменился — не писать.

Добавить строку только если что-то материально изменилось:
- новый пользователь затронут
- появился деплой рядом с датой first_seen
- сместился root cause
- влетел связанный фикс

Если ничего из этого нет — блок с цифрами и есть весь комментарий.

---

## Never write

- "Фикса ещё нет" — задача открыта, и так понятно
- "Verdict unchanged" — не нужно говорить о том, что не изменилось
- Recommendation — Fix Options уже в теле задачи
- Пересказ того, что уже написано в задаче
