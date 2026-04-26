---
name: tc-gap
description: >-
  Gap analysis: compares existing <tms> TCs against available signal sources
  per project — <analytics> events (Web), backend handlers (Back), admin
  operations (Admin) — to find uncovered areas in <product>. Produces a
  prioritized gap report. Trigger: "tc-gap", "gap analysis", "что не покрыто",
  "найди пробелы", "какие кейсы пропущены".
version: 0.2.0
---

# TC Gap — Coverage Gap Analysis

Find what's not covered in <tms>. Signal sources differ per project:

| Project | Primary signal | Additional signal |
|---|---|---|
| Web (id=1) | <analytics> named events | <error-monitoring> JS errors (frontend crashes, unhandled rejections) |
| Back (id=2) | Backend handlers (`/backend/backend/app/handlers/`) | <error-monitoring> Python errors (unhandled exceptions in handlers) |
| Admin (id=3) | Admin panel pages and their operations (`/admin/src`) | — |

**<error-monitoring> MCP:** `mcp__sentry__list_issues`, `mcp__sentry__list_events`, `mcp__sentry__get_sentry_resource`

Run automatically when invoked — no clarifying questions.

## Workflow

### Step 0 — Read existing <wiki> page

Before doing anything else, fetch the current TC Gap page:
**Page ID:** `34b98d8a-3c9b-81e7-8a16-c6d685dd0742`

Use `mcp__notion__notion-fetch` with that ID.

Extract and carry into the analysis:
1. **Дата последнего запуска** — сравнить с сегодня, понять насколько данные устарели
2. **Аномалии / под вопросом** — всё что было помечено ⚠️ в предыдущем репорте (например, `Payment Successful ≠ Payment Succeeded?`) → добавить в список "перепроверить" в текущем анализе
3. **Раздел "📌 Заметки"** — если он есть на странице, сохранить его содержимое дословно: он написан вручную и НЕ перезаписывается при обновлении

---

### Step 1 — Fetch all TCs from <tms> (parallel)

```
mcp__<tms>__<tms>_list_testcases(project_id=1)  # Web
mcp__<tms>__<tms>_list_testcases(project_id=2)  # Back
mcp__<tms>__<tms>_list_testcases(project_id=3)  # Admin
```

Build flat lists per project. Count totals.

---

### Step 2 — Collect signals per project

#### Web → <analytics> events

1. `mcp__Amplitude__get_context` → get projectId for Production (<YOUR_ANALYTICS_PROJECT_ID>)
2. `mcp__Amplitude__search` with queries for known event families:
   `["Task", "Payment", "Registration", "Post", "Logged", "Email", "Person", "Account", "Sign In"]`
   entityTypes: `["EVENT"]`, appIds: `[<YOUR_ANALYTICS_PROJECT_ID>]`
3. Filter out `[Guides-Surveys]`, `[<analytics>]`, `[Experiment]` prefixes.
4. Result: list of named product events.

#### Back → Backend handlers

Read handler files to discover what's implemented:

```bash
find <product-dir>/backend/backend/app/handlers -name "*.py" | sort
```

For each handler file, extract: filename + first docstring or comment describing what it handles.
Group into functional areas: billing, generation, assets, auth, webhooks.

**Optional: enrich with <error-monitoring> errors**

After reading handlers, check <error-monitoring> for high-frequency unhandled exceptions that might reveal untested error paths:

```
mcp__sentry__list_issues(query="is:unresolved", limit=20)
```

For each <error-monitoring> issue not covered by a Back TC — add to the gap list with priority based on event count (HIGH >1000/day, MEDIUM 100–1000, LOW <100).

If code is unavailable, use this known handler map as fallback:

| Area | Handlers |
|---|---|
| Billing — success | stripe, mollie, inwizo, tailored_pay, forumpay, paypal webhooks |
| Billing — failure | payment error, webhook retry, duplicate webhook |
| Billing — auto-topup | low balance trigger |
| Generation | task create, task fail/rescue, credits refund |
| Assets | upload, batch upload, HEIC validation, s3 key, preview |
| Auth | Google OAuth, email registration, long transaction |

#### Admin → Admin panel operations

Read admin source to discover pages:

```bash
find <product-dir>/admin/src -name "*.tsx" -path "*/pages/*" | sort
```

If unavailable, use this known operations map as fallback:

| Page | Operations |
|---|---|
| Users / UserDetail | search, view profile, promote trusted, grant credits |
| Tasks / TaskDetail | view task, view status, view details |
| Personas / PersonaDetail | create persona, edit persona |
| Templates | create, edit, transfer stage→prod |
| Loras / ServiceLoras | manage LoRA models |
| Banners | create, edit banners |
| Promocodes | create promocodes |
| CreditPlans | manage credit plans |
| ComfyServers | monitor servers, manage |
| ToolRestrictions | set tool restrictions |
| MollieRefunds | process refunds |
| Transactions | view transaction history |
| TaskStats | analytics (6 tabs) |

---

### Step 3 — Cross-reference per project

**Web:** For each <analytics> event, check if any Web TC title contains all words from the event name (case-insensitive). Mark ✅ / ❌.

**Back:** For each handler/area, check if any Back TC title contains keywords from that area. Match rule: at least one keyword from the handler description appears in the TC title.

**Admin:** For each page+operation pair, check if any Admin TC title covers it. Match rule: page name or operation keyword appears in TC title.

---

### Step 4 — Output gap report

```
## TC Gap Report — <product>
Date: YYYY-MM-DD
<tms>: N total TC (Web: X | Back: Y | Admin: Z)

### Web — <analytics> events (M checked, P% covered)

❌ Gaps:
| Event | Priority | Suggested TC title |
|---|---|---|

✅ Covered (top 5):
| Event | TC |
|---|---|

### Back — Backend handlers (M areas checked, P% covered)

❌ Gaps:
| Handler area | Priority | Suggested TC title |
|---|---|---|

✅ Covered:
| Handler area | TC |
|---|---|

### Admin — Panel operations (M ops checked, P% covered)

❌ Gaps:
| Page / Operation | Priority | Suggested TC title |
|---|---|---|

✅ Covered:
| Page / Operation | TC |
|---|---|
```

**Priority for gaps:**
- HIGH: payment/auth/generation critical path, or high <analytics> volume (>50K)
- MEDIUM: important feature with no coverage, medium volume (5K–50K)
- LOW: edge case, low volume (<5K), view-only events

---

### Step 5 — Update <wiki> page

After producing the report, update the TC Gap page in <wiki>:
**Page ID:** `34b98d8a-3c9b-81e7-8a16-c6d685dd0742`
**URL:** https://www.<wiki>.so/TC-Gap-34b98d8a3c9b81e78a16c6d685dd0742

Use `mcp__notion__notion-update-page` with `command: replace_content`.

**Структура страницы — строго в таком порядке:**

```
[Callout: дата запуска + coverage по проектам]

[Авто-репорт: три секции Web / Back / Admin]

---

## 📌 Заметки
[Содержимое из Step 0 — скопировать дословно если было.
Если раздела не было — создать пустым. Пользователь добавляет сюда вручную.]
```

**Правила обновления:**
- Callout и три секции репорта — всегда перезаписываются свежими данными
- Раздел `## 📌 Заметки` — **всегда сохраняется**: взять содержимое из Step 0 и вставить без изменений
- Если в Step 0 раздела не было — создать заголовок `## 📌 Заметки` с пустым телом
- Ничего не удалять из раздела заметок, даже если кажется устаревшим — это ручной контент

### Step 6 (optional) — Create TCs for top gaps

If user says "create them" / "добавь кейсы":
- Apply tc-create workflow for each (top 5 by priority)
- Status: `GUESS` by default (source confirmed, steps not verified hands-on)
- Folder per project: see `../tc-create/references/<tms>-api.md`

## Fallback: if <analytics> is unavailable

Skip Web <analytics> signals. Run Back and Admin analysis only.
For Web, fall back to critical path check (payment, auth, generation, nsfw).

## Notes

- Web match is fuzzy word-bag (known limitation: "Post Scheduled" can false-match "Post Published: publishType=scheduled")
- When in doubt about a match, mark as ❌ and note the ambiguity
- Back and Admin matching is looser by design — handler areas are broad
- Do not ask the user for scope — run all three projects and let them filter
