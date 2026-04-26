---
name: tc-create
description: >-
  This skill should be used when the user asks to "create a test case",
  "write a TC", "add a test case", "добавь кейс", "создай тест-кейс",
  "напиши кейс" for <product> in <tms>. Handles naming conventions,
  priority assignment, step format, and TMS MCP creation.
  Also supports bulk mode: creating multiple TCs at once with duplicate checking.
version: 0.3.0
---

# TC Create — <product> <tms>

Create test cases in `<tms>` for `<product>` following established conventions.
All data must come from a real source (`<analytics>`, code, `<metrics>`). Never invent steps or property values.

Supports two modes:
- **Single** — one TC, interactive, step by step (default)
- **Bulk** — list of TCs, with duplicate check before creating

## Configuration

Replace these placeholders before use:

| Placeholder | What to set |
|---|---|
| `<product>` | Your product name |
| `<tms>` | Your test management system name (e.g. Testiny, TestRail, Qase) |
| `<analytics>` | Your analytics tool (e.g. Amplitude, Mixpanel, PostHog) |
| `<metrics>` | Your metrics/observability tool (e.g. Grafana, Datadog) |
| `<error-monitoring>` | Your error monitoring tool (e.g. Sentry, Rollbar) |
| `mcp__<tms>__<tms>_*` | Replace with your actual MCP tool names |
| `references/<tms>-api.md` | Create this file with your TMS folder IDs |

## MCP Tools Used

| Action | Tool |
|---|---|
| List existing TCs | `mcp__<tms>__<tms>_list_testcases` |
| List folders | `mcp__<tms>__<tms>_list_folders` |
| Create folder | `mcp__<tms>__<tms>_create_folder` |
| Create TC | `mcp__<tms>__<tms>_create_testcase` |
| Add TC to folder | `mcp__<tms>__<tms>_add_to_folder` |
| Update TC | `mcp__<tms>__<tms>_update_testcase` (auto-fetches etag) |
| Get TC details | `mcp__<tms>__<tms>_get_testcase` |

## Hard Rules

- **ACTIVE TCs require explicit confirmation before any change.** Never update or modify a TC with status=ACTIVE automatically. Always stop, show what you're about to change, and wait for the user to say yes. Only proceed after explicit approval in that message.
- **DRAFT TCs** can be updated when the user agrees during a duplicate discussion.
- **Status note:** MCP supports ACTIVE, DRAFT, and GUESS. For gap-report-generated TCs (source confirmed, steps not verified hands-on) → use **GUESS**.

---

## Bulk Mode

Triggered when the user provides a list of TCs to create (e.g. from a tc-gap report or manual list).

### Bulk Step 1 — Fetch existing TCs

```
mcp__<tms>__<tms>_list_testcases(project_id=<id>)
```

### Bulk Step 2 — Duplicate check per TC

For each TC in the list, check if an existing TC is a likely duplicate:
- Split both titles into words (lowercased, strip punctuation)
- A duplicate is suspected if **≥ 2 meaningful words overlap** (ignore: "не", "и", "в", "на", "→", ":", "=")
- Only flag a suspected duplicate if the existing TC is in the **same project**

**If no duplicate suspected:** add to the creation batch silently.

**If duplicate suspected:** pause and discuss this TC with the user before continuing.

Offer all three options regardless of status. But if the existing TC is **ACTIVE**, add a confirmation gate:
```
⚠️ Возможный дубликат:
  Новый:        "<new title>"
  Существующий: "<existing title>" [ACTIVE ⚠️ требует подтверждения для изменений]

Создать новый / пропустить / обновить существующий?
```
If user chooses "обновить" and the TC is ACTIVE → ask once more:
```
Ты уверена, что хочешь изменить ACTIVE кейс "<title>"? Что именно меняем?
```
Only proceed after explicit yes.

Wait for the user's answer before moving to the next TC.

### Bulk Step 3 — Show plan, wait for confirmation

After resolving all duplicates, show the full creation plan:

```
Создаю N TC:
  ✅ [Project] "<title>" — папка, приоритет, статус
  ✅ ...
  ⏭ "<title>" — пропущен (дубликат по решению пользователя)

Подтверждаешь?
```

Only proceed after explicit confirmation.

### Bulk Step 4 — Create batch via MCP

For each confirmed TC call `mcp__<tms>__<tms>_create_testcase`.

If a folder doesn't exist yet — create it first with `mcp__<tms>__<tms>_create_folder`, then use the returned ID.

After creation, update `references/<tms>-api.md` with any new folder IDs.

---

## Single Mode Workflow

**Step 1 — Clarify if not provided:**
- Which project: Web, Back, or Admin? (configure your project IDs in `references/<tms>-api.md`)
- Which folder? (see `references/<tms>-api.md` for folder IDs)
- What is the data source? Options:
  - **`<analytics>`** — event name, properties, volume (`mcp__<analytics>__search`)
  - **Code** — backend handler or frontend tracking file
  - **`<metrics>`** — logs, metrics, error traces (`mcp__<metrics>__*`)
  - **`<error-monitoring>`** — specific error/exception details (`mcp__<error-monitoring>__list_issues`, `mcp__<error-monitoring>__list_events`)

**Step 2 — Choose title pattern** (pick one based on TC type):

| Type | Pattern | Example |
|---|---|---|
| Analytics event + property | `EventName: property=value` | `Job Created: type=<job_type>` |
| Negative / edge case | `condition → consequence` | `balance=0 → Job Created doesn't fire` |
| UI behavior / backend logic | `Object: short description` | `Promote User: requires confirmation` |

Rules:
- Never repeat the folder name in the title
- No fluff words: no "Successful", "Correct", "Happy path" as prefix
- Analytics data (event names, properties, values) in English with `property=value` notation
- Descriptive part in your team's language

**Step 3 — Set priority** based on analytics event volume or criticality:

| Priority | Analytics volume /30d | API value |
|---|---|---|
| HIGH | >50 000 or critical path | 1 |
| MEDIUM | 5 000–50 000 or important edge case | 2 |
| LOW | <5 000 or rare scenario | 3 |

**Step 4 — Set status:**
- `GUESS` — source confirmed (code/analytics), but steps not verified hands-on. Default for gap-report TCs.
- `DRAFT` — steps partially verified, work in progress.
- `ACTIVE` — every step verified from a real source (code, analytics query, metrics), nothing assumed. Only set ACTIVE on explicit user request.

When in doubt — always GUESS.

**Step 5 — Write steps.** Format: one step per line, `action → expected result`.
- Action: imperative ("Open task", "Click Retry")
- Expected: what should happen ("credits returned", "HTTP 200")
- Never invent UI button names or page names not seen in code

**Step 6 — Create via MCP:**

```
mcp__<tms>__<tms>_create_testcase(
  project_id=<id>,
  title="<title>",
  folder_id=<folder_id>,      # from references/<tms>-api.md
  priority=<1|2|3>,
  status="GUESS",             # default for gap-report TCs
  steps="step1 → result1\nstep2 → result2\n..."
)
```

## Folder Management

List folders:
```
mcp__<tms>__<tms>_list_folders(project_id=<id>)
```

Create folder if missing:
```
mcp__<tms>__<tms>_create_folder(project_id=<id>, title="<name>", parent_id=0)
```

After creating a new folder, add its ID to `references/<tms>-api.md`.

## Updating TCs

```
mcp__<tms>__<tms>_update_testcase(id=<tc_id>, title="...", priority=..., status="...", steps="...")
```

Etag is fetched automatically — no manual etag management needed.

## Additional Resources

- **`references/<tms>-api.md`** — folder IDs per project, valid enum values
- **`<error-monitoring>` MCP** — use `mcp__<error-monitoring>__list_issues` / `mcp__<error-monitoring>__list_events` to verify real error messages and stack traces when writing steps for error-path TCs
