---
name: task-comment
description: >-
  Post a short follow-up comment on an existing <product> <task-tracker> bug task —
  fresh numbers and what changed, nothing else. Use when re-checking an open
  ticket after bug-dig or during weekly review. Trigger: "обнови задачу",
  "добавь свежие цифры", "откомментируй задачу X", "обнови цифры".
  Always show draft first. NEVER post without explicit user confirmation.
---

# task-comment

Use `asana_create_task_story` (не create_task / update_task).

**HARD RULE: показать draft в чате, ждать "да/пиши/ок" перед постом.**

---

## Template

```
Fresh numbers (<date>)

<источник>:
  <метрика>: <prev> → <new> (<+delta>)
```

Источник — <error-monitoring> / <logs> / <analytics> / <metrics>. Только то, что реально менялось.

Доп. строка только если что-то материально изменилось:
- Новый user затронут
- Деплой рядом с first_seen
- Сместился root cause
- Влетел связанный фикс

Ничего из этого нет — блок с цифрами и есть весь комментарий.

---

## Never write

- "Фикса ещё нет" — задача открыта, и так понятно
- "Verdict unchanged" — не нужно говорить о неизменном
- Recommendation — план фиксов уже в задаче
- Пересказ того что в задаче
