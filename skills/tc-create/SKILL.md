---
name: tc-create
description: >-
  Create test cases for <product> in <tms>. Handles naming conventions,
  priority assignment, step format, and <tms> MCP creation. Supports bulk
  mode: creating multiple TCs at once with duplicate checking.
  Trigger: "create a test case", "write a TC", "add a test case",
  "добавь кейс", "создай тест-кейс", "напиши кейс".
---

# tc-create

## Constants

- `<tms>_WEB` = `1`
- `<tms>_BACK` = `2`
- `<tms>_ADMIN` = `3`

Folders + enum values → `references/<tms>-api.md`.

Все данные — из реального источника (<analytics>, code, <metrics>, <error-monitoring>). Шаги/значения не выдумывать.

**Два режима:**
- **BACKLOG mode** (основной): скилл читает BACKLOG-скелеты из <tms>, пользователь выбирает N для исследования → апгрейд существующего скелета
- **Direct mode** (fallback): пользователь описывает гэп текстом → создать TC с нуля (старый путь)

Modes:
- **Single** — interactive, default
- **Bulk** — list of TCs с duplicate check

---

## Step 0 — Load BACKLOG queue

Если пользователь не передал конкретный список гэпов текстом — загрузить BACKLOG из <tms>:

```
mcp__<tms>__<tms>_list_testcases(project_id=<tms>_WEB, status="BACKLOG")
mcp__<tms>__<tms>_list_testcases(project_id=<tms>_BACK, status="BACKLOG")
mcp__<tms>__<tms>_list_testcases(project_id=<tms>_ADMIN, status="BACKLOG")
```

Вывести список в чат, сгруппированный по priority:

```
🔴 HIGH (priority=1):
  [TC-id] Web / Auth: exchange_token: valid one-time token → session
  [TC-id] Back / Billing: Payblis webhook: missing signature → 401

🟠 MEDIUM (priority=2):
  ...

🟡 LOW (priority=3):
  ...

Итого в BACKLOG: N TC. Какие создаём? Укажи ID или порядковые номера.
```

Если пользователь передал конкретный список текстом — пропустить Step 0, использовать Direct mode.

---

## Hard Rules

- **ACTIVE TC** — explicit confirmation перед любым изменением. Stop, show, wait. Только после явного "да".
- **DRAFT TC** — обновляется после согласия в duplicate discussion.
- Status: ACTIVE / DRAFT / GUESS. Gap-report TC (source confirmed, steps не верифицированы) → **GUESS**.

**UI context для написания шагов:** перед TC для конкретной страницы — `find <product-dir>/ui-snapshots/output -name "*<slug>*.md"`. Карточка покажет точные названия кнопок, инпутов, заголовков → копируй в шаги вместо угадывания. Подробнее → `<product-dir>/CLAUDE.md` секция "UI Snapshots Catalog".

---

## Bulk Mode

### Step 1 — Fetch existing
```
mcp__<tms>__<tms>_list_testcases(project_id=<<tms>_*>)
```

### Step 2 — Duplicate check per TC

- Split titles на слова (lowercase, без пунктуации)
- Дубль если ≥2 значимых слов overlap (ignore: "не", "и", "в", "на", "→", ":", "=")
- Только same project

**Нет дубля:** silently в batch.

**Есть дубль:** pause + discuss. Если existing ACTIVE — confirmation gate:
```
⚠️ Возможный дубликат:
  Новый: "<title>"
  Существующий: "<title>" [ACTIVE ⚠️ требует подтверждения]

Создать новый / пропустить / обновить?
```

ACTIVE + "обновить" → ещё раз: "Что именно меняем?". Только после явного yes.

Wait per TC.

### Step 3 — Show plan, wait

```
Создаю N TC:
  ✅ [Project] "<title>" — папка, приоритет, статус
  ⏭ "<title>" — пропущен (дубликат)

Подтверждаешь?
```

### Step 4 — Create batch

`mcp__<tms>__<tms>_create_testcase` per TC. Папки — `mcp__<tms>__<tms>_create_folder` если нет, потом обновить `references/<tms>-api.md` с новым ID.

---

## Single Mode

**Step 1 — Clarify:**
- Project: <tms>_WEB / <tms>_BACK / <tms>_ADMIN
- Folder (см. references/<tms>-api.md)
- Source: <analytics> (`mcp__Amplitude__search`) / Code / <metrics> / <error-monitoring>

**Step 2 — Title pattern:**

| Type | Pattern | Example |
|---|---|---|
| <analytics> event + property | `EventName: prop=val` | `Task Created: type=videogen` |
| Negative/edge | `condition → consequence` | `credits=0 → Task Created не срабатывает` |
| UI/backend logic | `Object: short description` | `Promote Trusted: требует подтверждения` |

Rules:
- Не повторять folder в title
- Без флаффа: "Успешная", "Корректная", "happy path"
- <analytics> data — английский, `prop=value` notation
- Описательная часть — русский

**Step 3 — Priority:**

| Priority | Volume /30d | API |
|---|---|---|
| HIGH | >50K или critical path | 1 |
| MEDIUM | 5K-50K или важный edge | 2 |
| LOW | <5K или rare | 3 |

**Step 4 — Status:**
- `GUESS` — source confirmed, steps не верифицированы. Default для gap-report.
- `DRAFT` — partially verified, WIP.
- `ACTIVE` — каждый шаг верифицирован. Только по явному запросу.

В сомнении — GUESS.

**Step 5 — Steps.** Format: `action → expected` per line. Action — imperative. Expected — что должно случиться. UI button names не выдумывать.

**Step 6 — Create or upgrade:**

**BACKLOG mode — апгрейд скелета:**
Если TC пришёл из BACKLOG (есть ID):
1. `mcp__<tms>__<tms>_get_testcase(id=<id>)` — прочитать скелет
2. Из steps первой строки (GAP_META) извлечь: Source, SourceRef, GapReason — контекст для ресерча
3. Провести deep research (код, <error-monitoring>, <analytics> — как обычно)
4. `mcp__<tms>__<tms>_update_testcase(id=<id>, steps=<новые шаги>, status=<DRAFT или GUESS>, custom_fields={"LastReviewedAt": "<YYYY-MM-DD>"})`
   - DRAFT если шаги проверены и уверен
   - GUESS если есть сомнения (требует ручной верификации)
5. Переместить в правильный folder если TC сейчас без папки: `mcp__<tms>__<tms>_add_to_folder(testcase_id=<id>, folder_id=<id>)`

**Direct mode — создать с нуля** (только если нет ID из BACKLOG):
```
mcp__<tms>__<tms>_create_testcase(
  project_id=<<tms>_*>,
  title="...", folder_id=<from references>,
  priority=<1|2|3>, status="GUESS",
  steps="step1 → result1\nstep2 → result2",
  custom_fields={"LastReviewedAt": "<YYYY-MM-DD>"}
)
```

---

## Folders

```
mcp__<tms>__<tms>_list_folders(project_id=<<tms>_*>)
mcp__<tms>__<tms>_create_folder(project_id=<<tms>_*>, title="...", parent_id=0)
```

После создания — добавить ID в `references/<tms>-api.md`.

---

## Update

```
mcp__<tms>__<tms>_update_testcase(id=<tc_id>, title=..., priority=..., status=..., steps=...)
```

Etag — автоматически.

---

## Resources

- `references/<tms>-api.md` — folder IDs per project, enum values
- <error-monitoring> MCP — `list_issues`/`list_events` для error-path TC steps
