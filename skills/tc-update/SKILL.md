---
name: tc-update
description: >-
  Update existing test cases in <tms> for <product>. Supports: bulk field
  updates (type, priority, status, automation), folder moves, content/steps
  edits, and renames. Handles etag flow automatically. ACTIVE TCs require
  explicit confirmation before any change.
  Trigger: "обнови кейс", "переименуй TC", "перемести в папку", "поменяй тип", "tc-update".
---

# tc-update

**Два режима:**
- **Manual mode** (текущий): точечные изменения по запросу — "переименуй TC-123", "поменяй тип", "перемести в папку"
- **STALE mode** (новый): запуск без аргументов или `--stale` — загружает STALE-очередь, deep research, обновляет или архивирует

## Constants

- `<tms>_WEB` = `1`
- `<tms>_BACK` = `2`
- `<tms>_ADMIN` = `3`

Folder IDs → `../tc-create/references/<tms>-api.md`. Etag — automatic.

---

## Hard Rules

- **ACTIVE TC** — explicit confirmation per TC перед изменением. Stop, show, wait, "да".
- **DRAFT/GUESS** — после plan confirmation (Step 4).
- Не менять fields, которые user не просил.
- **STALE TC** — перед архивированием всегда показать TC и ждать явного "да". Архив необратим.
- **GapReason** — читать как первый источник контекста при STALE mode. Не игнорировать.
- **LastReviewedAt** — всегда обновлять при переводе из STALE в DRAFT.

---

## STALE Mode Flow

Активируется когда: нет аргументов, или пользователь написал "обнови устаревшие" / "--stale" / "stale mode".

### Step S0 — Load STALE queue

```
mcp__<tms>__<tms>_list_testcases(project_id=1, status="STALE")
mcp__<tms>__<tms>_list_testcases(project_id=2, status="STALE")
mcp__<tms>__<tms>_list_testcases(project_id=3, status="STALE")
```

Вывести список в чат по priority. Спросить: "Какие обрабатываем? Укажи ID."

### Step S1 — Deep research per TC

Для каждого выбранного STALE TC:
1. `mcp__<tms>__<tms>_get_testcase(id=<id>)` — прочитать TC и GapReason из custom fields
2. GapReason = контекст: что именно изменилось (handler удалён, event переименован, feature удалена)
3. Проверить текущее состояние: найти соответствующий handler/event/feature в коде или <analytics>/<error-monitoring>
4. Принять решение:

**Если feature изменилась, TC ещё актуален:**
```
mcp__<tms>__<tms>_update_testcase(
  id=<id>,
  steps=<обновлённые шаги>,
  status="DRAFT",
  custom_fields={"LastReviewedAt": "<YYYY-MM-DD>"}
)
```

**Если feature удалена, TC больше не актуален:**
- Показать: "TC-<id> '<title>': feature удалена. Архивировать?"
- Ждать подтверждения "да"
- `mcp__<tms>__<tms>_update_testcase(id=<id>, status="ARCHIVED")`

**Если непонятно:**
- Задать конкретный вопрос пользователю с контекстом из GapReason

---

## Operations

| Op | Trigger |
|---|---|
| Bulk field update | "всем кейсам в папке Auth поставь type=SMOKE" |
| Folder move | "перемести TC-42, TC-55 в Billing" |
| Steps/content edit | "обнови шаги у TC-38" |
| Rename | "переименуй TC-71 в '...'" |
| Mixed | "поменяй приоритет и перемести" |

---

## Workflow

### Step 1 — Clarify

Если не указано:
- Какие TC? (IDs или filter: project + folder + status)
- Что меняем?

### Step 2 — Fetch

By filter:
```
mcp__<tms>__<tms>_list_testcases(project_id=<<tms>_*>)
```

By ID:
```
mcp__<tms>__<tms>_get_testcase(id=<tc_id>)
```

Carry per TC: id, status, title, current values.

### Step 3 — ACTIVE check

Per ACTIVE TC:
```
⚠️ TC-{id} "{title}" имеет статус ACTIVE.
Изменение: {field} → {new}

Применить? (да / пропустить)
```
Wait per TC.

### Step 4 — Show plan

```
Обновляю N TC:
  TC-{id} "{title}" [{status}]
    {field}: {old} → {new}
  ⏭ TC-{id} — пропущен (ACTIVE, отказ)

Подтверждаешь?
```

### Step 5 — Execute

**Field/rename/steps:**
```
mcp__<tms>__<tms>_update_testcase(
  id=<tc_id>,
  title=..., priority=<1|2|3>, status=...,
  steps="step1\nstep2"
)
```

**Folder move:**
```
mcp__<tms>__<tms>_add_to_folder(testcase_id=<id>, folder_id=<folder_id>)
```

### Step 6 — Report

```
Готово:
  ✅ TC-{id} — обновлён
  ❌ TC-{id} — ошибка: {http_error}
```

Errors — показать raw response, не silent skip.
