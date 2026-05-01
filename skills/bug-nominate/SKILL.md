---
name: bug-nominate
description: >-
  Single owner of writes to the Bug Candidates <wiki> DB. Two modes: interactive
  (draft+confirm in chat — default, used by bug-dig) and silent (write directly,
  used by bug-review batch). Auto-detects create vs update by fingerprint. Does
  not investigate — that's bug-dig. Does not create <task-tracker> tasks — that's task-create.
  Trigger: "bug-nominate", "запиши в кандидаты", "добавь в candidates",
  "зафиксируй вердикт", "номинируй баг". Also called automatically by bug-dig
  (interactive) and bug-review (silent).
---

# bug-nominate

**Единственный writer для Bug Candidates DB.** Все остальные скиллы (bug-dig, bug-review) делегируют запись сюда. Schema живёт здесь и в `bug-review/references/<wiki>-schema.md`.

## Constants

- `BUG_CANDIDATES_DS_ID` = `26b9b9ff63194e88af44b30a6978600d`
- `BUG_CANDIDATES_DS_URL` = `collection://26b9b9ff63194e88af44b30a6978600d`

Schema → `bug-review/references/<wiki>-schema.md`.

---

## Inputs

Принимает в любой комбинации:

| Field | Required | Notes |
|---|---|---|
| `title` | yes | Page title, 1 line |
| `fingerprint` | yes | Lowercase deterministic ID — `<error-monitoring>:<your-error-monitoring-project>-3fqz` / `<task-tracker>:1214140404778710` / `<data-warehouse>:tool=X:reason=Y` / `manual:short-slug` |
| `status` | yes | `Active` / `Tracked` / `Closed` / `Meta` / `Regression` |
| `verdict` | recommended | `Prod bug` / `Latent prod bug` / `<error-monitoring> noise` / `Analytics gap` / `Preventive` / `Meta / Risk signal` / `Protected` |
| `user_impact` | recommended | `Yes` / `No` / `Unknown` |
| `severity` | recommended | `Critical` / `High` / `Medium` / `Low` |
| `sources` | recommended | array из <error-monitoring> / <metrics> / <data-warehouse> / <analytics> / <vcs> / <task-tracker> / <wiki> |
| `trend` | optional | `New` для new; для updates не трогать (bug-review пересчитает) |
| `asana_link` | optional | URL если ticket существует |
| `body_markdown` | optional | Полное расследование. Если передан — пишется в content страницы. Если нет — см. правила ниже. |
| `silent` | optional, default `false` | `true` → пишет молча без draft+confirm. Используется bug-review для batch ops. |

Если в чате уже есть полный verdict от bug-dig — забрать данные оттуда.

---

## Body markdown — где живёт расследование

**Полное расследование = content страницы**, не properties. Schema cleanup от 2026-04-30.

Каноничный шаблон body (используется bug-dig + bug-nominate):

```markdown
## Symptom

<что видит пользователь / что сломано — 1-2 параграфа>

## Verdict

<одна строка с вердиктом + причиной>

## Signal refs

<bullet-list: <error-monitoring> IDs, <task-tracker> GIDs, <logs> queries, commit hashes, file paths>

## Root Cause

<параграф(ы) — почему это происходит, с кодовыми ссылками / номерами строк>

<!-- Mirrors task-create § Action items (имя секции синхронизировано) -->
## Action items

<директивный список действий команды на продуктовом языке. <wiki> = research doc, поэтому здесь можно держать несколько вариантов фикса (config / guard / systemic) для дальнейшего обсуждения. На <task-tracker> попадает уже выбранный план через task-create — там action items отбираются и сопровождаются метками уровней (Backend/Frontend, Hot-fix/Proper/Architectural, Required/UX/Optional) только когда правки разные по природе.>

## User Impact

<параграф — financial / UX / counts>

## Source data

<bullet-list: какие queries / files / commits легли в основу>
```

Секции можно опускать если нерелевантно (preventive — без User Impact; risk signal — без Action items; bug-review-минимум — только Symptom + Signal refs).

**Минимальный body** для bug-review при создании нового signal (когда расследования ещё нет):

```markdown
## Symptom

<title или 1 строка из source>

## Signal refs

<source refs: <error-monitoring> IDs, <task-tracker> GIDs, commits, queries>
```

---

## Workflow

### Step 1 — Resolve mode & gather inputs

- `silent=true`? Пропустить Step 2 (draft+confirm), идти сразу в Step 3.
- `silent=false` (default): идём в Step 2.

### Step 2 — Draft в чат (interactive mode only)

Покажи в чате:
- Title, Fingerprint, Status, Verdict, User Impact, Severity
- Первые 2-3 строки body (Symptom + Verdict)
- Mode: `CREATE` / `UPDATE properties only` / `UPDATE + replace body`

Жди подтверждения: «ок» / «пиши» / «да». Без подтверждения write-операции не делаем.

### Step 3 — Find existing by fingerprint

```python
mcp__notion__notion-search(
  query=<fingerprint>, filters={},
  data_source_url=BUG_CANDIDATES_DS_URL,
  page_size=10
)
```

`<wiki>-search` semantic, не exact. **Частичное совпадение fingerprint ок**, если остальные поля (title, sources) подходят по смыслу. Fetch топ-3 кандидата через `<wiki>-fetch`, выбрать тот, чей `Fingerprint` property содержит искомую часть, ИЛИ чей title совпадает по сути.

Found → Step 4. Not found → Step 5.

### Step 4 — UPDATE existing

```python
mcp__notion__notion-update-page(
  page_id=<found_id>,
  command="update_properties",
  properties={
    Status, Verdict, "User Impact", Severity,
    "date:Last seen:start", "Weeks seen", "<task-tracker> link",
    Sources,  # merge с existing — если новый source появился
    Fingerprint,  # перезапись допустима
  },
  content_updates=[]
)
```

**Bumps:**
- `Last seen` → today
- `Weeks seen` += 1 **только если ISO-неделя сменилась** относительно предыдущего last_seen (idempotent для daily/multi-run в одну неделю)
- `First seen` — НЕ трогать
- `Trend` — НЕ трогать (bug-review пересчитает)

**Body:**
- Если `body_markdown` передан → второй вызов update-page с `command="replace_content"`, `new_str=body_markdown`. Перезаписывает body полностью.
- Если `body_markdown` НЕ передан → body не трогаем. **Это важно** — защищает investigation, написанную bug-dig'ом, от стирания weekly bumps.

### Step 5 — CREATE new

```python
mcp__notion__notion-create-pages(
  parent={type:"data_source_id", data_source_id:BUG_CANDIDATES_DS_ID},
  pages=[{
    properties: {
      Title, Fingerprint, Status, Verdict, "User Impact", Severity, Sources,
      "date:First seen:start": today,
      "date:First seen:is_datetime": 0,
      "date:Last seen:start": today,
      "date:Last seen:is_datetime": 0,
      "Weeks seen": 1,
      Trend: "New",
      "<task-tracker> link": <if any>
    },
    content: <body_markdown OR минимальный шаблон Symptom+Signal refs>
  }]
)
```

### Step 6 — Report

В чат — всегда (даже при `silent=true`):

```
✓ Bug Candidates: <CREATE|UPDATE> «<title>»
   url: <page url>
   verdict: <verdict>, severity: <severity>, user impact: <yes/no>
```

Это даёт пользователю видимость даже при batch-операциях.

---

## Caller patterns

### Called by bug-dig (interactive)

bug-dig в Stage 7 после verdict собирает все args (включая `body_markdown` по каноничному шаблону) и вызывает `/bug-nominate` без `silent` flag → попадает в interactive mode → draft+confirm.

### Called by bug-review (silent batch)

bug-review в Layer 2 для каждого нового/изменённого signal вызывает `/bug-nominate silent=true`:
- Новый signal → CREATE с минимальным body (только Symptom + Signal refs)
- Existing signal → UPDATE properties only (без `body_markdown`) → defends investigation

### Called manually

Пользователь набирает `/bug-nominate` в чате — берёт verdict из чата, идёт в interactive mode.

---

## Hard rules

- ❌ Без verdict / inputs в текущем разговоре или args
- ❌ <task-tracker> tasks — это task-create
- ❌ В UPDATE без `body_markdown` НЕ переписывать body — защита investigation от bug-review weekly bumps
- ❌ В UPDATE НЕ трогать `First seen` и `Trend`
- ❌ Не использовать удалённые property `Signal refs` / `Root Cause` / `Fix Options` (мигрировано 2026-04-30)
- ✅ Draft + confirmation перед записью (interactive mode)
- ✅ Полное расследование — в body страницы (Markdown), не в properties
- ✅ Reporting в чат всегда — даже при `silent=true`
