---
name: tc-gap
description: >-
  Gap analysis: compares existing <tms> TCs against available signal sources
  per project — <analytics> events (Web), backend handlers (Back), admin
  operations (Admin) — to find uncovered areas in <product>. Produces a
  prioritized gap report. Trigger: "tc-gap", "gap analysis", "что не покрыто",
  "найди пробелы", "какие кейсы пропущены".
---

# tc-gap

## Constants

- `<tms>_WEB` = `1`
- `<tms>_BACK` = `2`
- `<tms>_ADMIN` = `3`
- `AMPLITUDE_PROD` = `<YOUR_ANALYTICS_PROJECT_ID>`
- `BACKEND_HANDLERS_PATH` = `<product-dir>/backend/backend/app/handlers`
- `ADMIN_PAGES_PATH` = `<product-dir>/admin/src`

Run automatically — без clarifying questions.

| Project | Primary signal | Additional |
|---|---|---|
| Web (<tms>_WEB) | <analytics> events | <error-monitoring> JS errors |
| Back (<tms>_BACK) | Backend handlers | <error-monitoring> Python errors |
| Admin (<tms>_ADMIN) | Admin pages + operations | — |

---

## Step 0a — Sync repos

`/git-refresh` (pull all 3 + `code-review-graph update`). Без свежего кода `find` по `BACKEND_HANDLERS_PATH` и `ADMIN_PAGES_PATH` пропустит новые handlers/pages.
Если `/git-refresh` уже выполнялся в этой сессии — пропустить.

---

## Step 1 — Fetch all TCs (parallel)

```
mcp__<tms>__<tms>_list_testcases(project_id=<tms>_WEB)
mcp__<tms>__<tms>_list_testcases(project_id=<tms>_BACK)
mcp__<tms>__<tms>_list_testcases(project_id=<tms>_ADMIN)
```

---

## Step 2 — Signals per project

### Web → <analytics>
```
mcp__Amplitude__get_context  # → AMPLITUDE_PROD
mcp__Amplitude__get_events(appId=AMPLITUDE_PROD)
```
Filter prefixes: `$`, `[<analytics>]`, `[Experiment]`, `[Guides-Surveys]`.

### Back → Handlers
```bash
find $BACKEND_HANDLERS_PATH -name "*.py" | sort
```
Per file: filename + first docstring/comment. Group: billing, generation, assets, auth, webhooks.

**Optional <error-monitoring> enrichment:**
```
mcp__sentry__list_issues(query="is:unresolved", limit=20)
```
<error-monitoring> issues не покрытые Back TC → в gap. Priority по event count: HIGH >1000/day, MEDIUM 100-1000, LOW <100.

**Fallback handler map (если код недоступен):**

| Area | Handlers |
|---|---|
| Billing — success | stripe, mollie, inwizo, tailored_pay, forumpay (crypto), paypal webhooks |
| Billing — failure | payment error, webhook retry, duplicate webhook |
| Billing — auto-topup | low balance trigger, get/update settings, admin trigger |
| Generation — tasks | task create, task fail/rescue, credits refund, get processing |
| Generation — calls | call create, execute, fail, rescue |
| Generation — results | get task result |
| Assets | upload, batch upload, finalize, presign, HEIC convert, preview, download, delete |
| Asset groups | create/add/get/download |
| Auth | Google OAuth, email signup, login, logout, verify email, reset/change password, TMA |
| Socials — OAuth | FB/IG, Threads, Twitter, YouTube: get_oauth_url, oauth_redirect |
| Socials — connections | delete, refresh_token, get_user_profile (×5) |
| Socials — publish | create_publication (×5) |
| Publications | create_batch, update_batch, publish, delete |
| Personas | CRUD, admin get/list, generations, groups |
| Notifications | get, dismiss, mark_read, mark_all_read |
| Templates | CRUD, use, react, admin |
| Photoshoot categories | CRUD, admin prompts CRUD |
| Posts | CRUD, global feed, admin |
| Analytics | overview, by_models/networks, calls, posts, publications, content_performance, refresh |
| LoRA / ServiceLoRA | CRUD, file upload (multipart), list |
| ComfyServers | CRUD, reset_health, check_health, sync_fleet |
| Credit plans | CRUD, admin list |
| Promocodes | CRUD, redeem, allowed_emails |
| Tool restrictions | CRUD, admin list |
| Users (admin) | block/unblock, change_role, delete, promote_trusted, add_credits, update, credit_transactions |
| Users (self) | get, update_nsfw, confirm_trust, delete, sessions, credit_transactions |
| Transactions | admin list |
| Stats (admin) | tasks: error_analysis/performance/provider_scoreboard/refund_analytics/user_segments; credits: stats/transactions/top_users |
| Domains | frontend_domains admin CRUD, banned_domains admin CRUD |
| Banners | CRUD, dismiss, get (user + admin) |
| Config | health |

### Admin → Pages
```bash
find $ADMIN_PAGES_PATH -name "*.tsx" -path "*/pages/*" | sort
```

**Fallback admin map:**

| Page | Operations |
|---|---|
| Users / UserDetail | search, view, promote trusted, grant credits, block/unblock, change role, delete |
| Tasks / TaskDetail | view list, status, details, filter by status/userId |
| Personas / PersonaDetail | CRUD |
| Templates | CRUD, transfer stage→prod |
| Loras / ServiceLoras | CRUD; Klein presets |
| Banners | CRUD |
| Promocodes | create, disable |
| CreditPlans | CRUD |
| ComfyServers | monitor, enable/disable, manage |
| ToolRestrictions | set restrictions |
| MollieRefunds | refunds, list, find payment |
| Transactions | history view |
| TaskStats | 6 tabs: Overview, Performance, Providers, Errors, Refunds, UserSegments |
| PhotoshootCategories | CRUD categories + prompts |
| Posts / PostDetail | view list/details |
| ErrorAnalysis | patterns, drilldown |
| Domains | manage frontend domains |
| Feed | view content |
| AssetUpload | upload assets |

---

## Step 3 — Cross-reference

**Web:** для каждого <analytics> event → есть ли Web TC где title содержит все слова event'а (case-insensitive). ✅/❌.

**Back:** для каждой area → есть ли Back TC где title содержит keyword из area description. Match: ≥1 keyword.

**Admin:** для каждой page+op → есть ли Admin TC где title содержит page/operation keyword.

---

## Step 4a — Write BACKLOG to <tms>

Для каждого нового гэпа (не найдено TC в Step 3):

**Дедупликация перед созданием:** проверить существующие TC проекта. Если есть TC с `status IN (BACKLOG, GUESS, DRAFT)`, чей title совпадает по ≥2 ключевым словам с suggested title — пропустить, не создавать дубликат.

Создать скелет:
```
mcp__<tms>__<tms>_create_testcase(
  project_id=<проект>,
  title=<suggested TC title из Step 3>,
  status="BACKLOG",
  priority=<1/2/3 по эвристике>,
  folder_id=<ближайший подходящий folder>,
  steps="GAP_META: source=<Source> | ref=<SourceRef> | reason=<GapReason> | detected=<YYYY-MM-DD> → (skeleton, awaiting tc-create)"
)
```

**Эвристика приоритета:**
- HIGH (1): payment / auth / generation critical path; <error-monitoring> >1000 events/day
- MEDIUM (2): важная фича без покрытия; <error-monitoring> 100–1000 events/day
- LOW (3): edge case; view-only; <100 events/day

---

## Step 4b — Mark STALE in <tms>

Для каждого существующего TC, у которого изменился или исчез сигнал:
- Handler переименован / удалён
- <analytics> event удалён из таксономии
- Admin-страница удалена
- <error-monitoring> issue: ошибка пропала после деплоя и TC стал неактуальным

Если TC уже STALE — обновить только GapReason (дописать новую причину), не создавать повторно.

```
mcp__<tms>__<tms>_update_testcase(
  id=<TC id>,
  status="STALE",
  custom_fields={
    "GapReason": "<старый reason если был> + [<дата>] <новая причина>",
    "DetectedAt": "<YYYY-MM-DD>"
  }
)
```

---

## Step 5 — Summary (chat only)

Вывести в чат итоговый отчёт. <wiki> больше не обновляется.

Формат:
```
## TC Gap — <product> [дата]
<tms>: N total (Web: X | Back: Y | Admin: Z)

### Новые BACKLOG-скелеты создано: N
[список: TC title | project | priority | source]

### Помечено STALE: N
[список: TC id + title | причина]

### Покрытие без изменений: N областей
```

---

## Fallback

<analytics> недоступен → пропустить Web <analytics>. Для Web — critical path check (payment/auth/generation/nsfw).
- Если <tms> недоступен → пропустить Step 4a/4b, вывести только чат-отчёт с пометкой "⚠️ <tms> write skipped"
- Если TC уже в статусе BACKLOG или STALE — не создавать дубликат, только обновить GapReason

## Notes

- Web match — fuzzy word-bag (известное ограничение: "Post Scheduled" может ложно матчить "Post Published: publishType=scheduled")
- При сомнении — ❌ + note ambiguity
- Back/Admin matching looser by design (areas широкие)
- Не спрашивать scope — все три проекта
