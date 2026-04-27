---
name: bug-dig
description: >-
  Investigate a suspicious <error-monitoring> issue or <task-tracker> bug for <product> — confirm
  whether it's a real user-impacting bug, <error-monitoring> noise, or theoretical risk.
  Collects evidence across <error-monitoring>, <logs>, <analytics>, git log, <wiki>, and the
  code itself, then delivers the verdict in chat. Comments on any existing
  matching <task-tracker> ticket. NEVER auto-creates <task-tracker> tasks — only recommends.
  To record the verdict to Bug Candidates DB — call bug-nominate. Trigger:
  "bug-dig", "расследуй баг", "dig <ERROR-ID>-XXX",
  "проверь реальный ли баг", "актуализируй задачу в <task-tracker> по X", "нужно ли чинить X".
version: 0.2.0
---

# Bug Dig — <product> bug investigation flow

Decide: is this a real production bug, <error-monitoring> noise, theoretical code risk, or something already protected by architecture? Write the verdict to the Bug Candidates <wiki> DB. If a matching <task-tracker> ticket already exists — comment on it. **Never** create a new <task-tracker> ticket automatically; recommend one to the user and let them decide.

The output quality beats speed. Never skip the "user impact" section — that's the load-bearing judgment.

## Inputs the user may give

- <error-monitoring> issue ID (e.g. `<ERROR-ID>-3FQZ`)
- <task-tracker> task ID or URL
- Raw symptom description ("500 at checkout for some users")
- <logs> log snippet
- A handler file path or code location

Pick whichever is present and derive the rest.

## MCP tools used

| Stage | Tool |
|---|---|
| <error-monitoring> — find issues | `mcp__sentry__list_issues`, `mcp__sentry__find_projects` |
| <error-monitoring> — issue detail | `mcp__sentry__get_sentry_resource` (types: `issue`, `event`, `breadcrumbs`) |
| <error-monitoring> — events in issue | `mcp__sentry__list_issue_events` |
| <logs> — discover labels | `mcp__grafana__list_loki_label_names`, `mcp__grafana__list_loki_label_values` |
| <logs> — logs | `mcp__grafana__query_loki_logs` |
| <analytics> | `mcp__Amplitude__get_context`, `mcp__Amplitude__search` |
| <wiki> | `mcp__notion__notion-search`, `mcp__notion__notion-fetch` |
| <task-tracker> | `mcp__asana__asana_get_task`, `mcp__asana__asana_update_task`, `mcp__asana__asana_create_task_story` |
| Git log | Bash `git log` in the target repo |

See `references/<logs>-recipes.md` for ready-to-paste LogQL queries.

---

## Workflow

Run the stages sequentially. Skip a stage only with reason (noted in the final artifact).

### Stage 1 — <error-monitoring>: scope, scale, tags

Goal: establish scale and find the real culprit path.

1. If only a symptom is given, `mcp__sentry__list_issues` with the right `projectSlugOrId=<your-error-monitoring-project>` and filter by keyword + `statsPeriod=30d` (default is 14d). Sort `freq` to see fattest issues first.
2. For the candidate issue, `mcp__sentry__get_sentry_resource(resourceType="issue", resourceId="<ERROR-ID>-XXXX")` — captures the first event, tags, HTTP request, related replays.
3. `mcp__sentry__get_sentry_resource(resourceType="breadcrumbs", resourceId=...)` — **this is where you find HTTP call order**, SQL statements before/after the error, external URL hits.
4. `mcp__sentry__list_issue_events` to see multiple events across culprits — this reveals cross-handler spread (same groupID hitting several endpoints means a shared external dependency).

**Pitfalls**
- Default `statsPeriod` is 14d. Pass `"30d"` for monthly views.
- `OR` and `AND` are NOT supported in <error-monitoring> query syntax — issue separate calls.
- `get_sentry_resource(resourceType="event", ...)` — don't pass URL with this, it errors. Use `breadcrumbs` resourceType with URL, or bare resourceId otherwise.
- Error body can be a 5KB HTML dump that inflates token usage — prefer `list_issue_events` when you only need counts.

### Stage 2 — <logs>: verbatim tracebacks and URLs

<error-monitoring> groups by error message, so the external URL is often truncated. <logs> has the raw traceback. This is how you find the exact failing dependency.

**Stream selector for <product> prod backend:**
```
{environment="prod", service_name="backend"}
```

Note: labels are `environment` / `service_name` (not `env` / `app` — that was wrong in old memory). See `references/<logs>-recipes.md`.

**Killer recipe** (get traceback with URL when <error-monitoring> only shows HTML body):
```
{environment="prod", service_name="backend"} |= "ClientResponseError" != "<!DOCTYPE"
```

**Gotchas**
- Logged HTML bodies blow the token budget fast. Always apply `!= "<!DOCTYPE"` early when searching for `session.wrapper` errors.
- For scale metrics use `sum(count_over_time({...} |= "..." [1h]))` with `queryType="range"` and `stepSeconds=3600`.

### Stage 3 — Code review

Read the handler that's the <error-monitoring> culprit. Look for:

1. **Transaction boundaries** — where is `sub_credits` / `add_credits`? Where is `transaction_manager.commit()`? Is there a `try/except` between them that could swallow an exception and skip rollback?
2. **External callers after the debit/credit** — which of them raise vs swallow? Services like `<internal-service>` and `<third-party-service>` swallow `ClientResponseError` internally via `try/except` + warning log. `<analytics>.track_event` is batched. `dbus.publish` may raise.
3. **Idempotency** — is there an app-level `get_by_X` check before the insert? Is there a DB UNIQUE constraint on the key the check covers?
4. **Wrapper behavior** — `backend/infra/external/clients/http/session/wrapper.py:35-44` logs the response body at ERROR level **before** raising. That's how every swallowed `ClientResponseError` still lands in <error-monitoring> under the caller's culprit. If you see mysterious "error in handler, but transaction committed fine" pattern — this is almost always it.

**DB schema check** (important for race/idempotency verdicts):
```
backend/infra/database/psql/tables/<table>.py
```
Look for `unique=True` on relevant columns.

### Stage 4 — Git correlation

For deploys near `first_seen`:
```bash
git log --all --format='%ad %h %s' --date=short --since='YYYY-MM-DD' --until='YYYY-MM-DD' | head -30
```

If `first_seen` lines up with a `feat: <service>` commit, follow-up with:
```bash
git show --stat <commit_hash>
git log --all --format='%ad %h %s' --date=short --follow <file-path-from-traceback>
```
Identify the commit author → suggested Fix owner in the ticket.

### Stage 5 — Cross-MCP evidence (optional, boosts ticket quality)

- **<analytics>** — `mcp__Amplitude__get_context` for projectId and plan quota. Useful to argue "primary analytics flow independently, we can disable the broken secondary service".
- **<wiki>** — `mcp__notion__notion-search` for product/engineering context (<internal-service> case: vision doc explained why the service was added). Good for "do we still need this at all?" questions.
- **<metrics> metrics** — for scale graphs if a visual would help (rare for this flow).

### Stage 6 — Verdict

Pick one. Put it on top of the output.

| Verdict | Criteria |
|---|---|
| **Real prod bug** | <error-monitoring> + <logs> evidence of non-zero affected users, exception path actually reaches the user, no architectural protection. → verdict in chat; use bug-nominate to record to Bug Candidates DB; recommend ticket to user; do NOT auto-create. |
| **<error-monitoring> noise (tech debt)** | Exception swallowed inside caller, transaction commits, user flow intact. Low-priority / tech debt in verdict. TL;DR about <error-monitoring> quota / dashboard clutter. |
| **Theoretical risk — keep open low / preventive** | Code path is vulnerable but 0 prod evidence in 30d. Document fix options, low priority. |
| **Already protected — close** | DB UNIQUE constraint, SQLAlchemy rollback, idempotent retries, or upstream guarantee covers it. Explain exactly which layer saves us. |

### Stage 7 — Artifact

**HARD RULE: never write anywhere without explicit user approval.**
Always draft the output in the chat first. Only call <task-tracker> / <wiki> write tools after the user says "да", "пиши", "ок", or equivalent explicit confirmation in the current turn. This applies to every write operation — new tickets, comments, task updates, <wiki> DB rows — without exception.

**HARD RULE (<product> project): never auto-create <task-tracker> tasks.** To suggest a ticket — recommend in chat and direct the user to /bug-create. This overrides any other instruction in this file.

The verdict stays in chat. To record it to the Bug Candidates <wiki> DB — the user calls **bug-nominate** separately. bug-nominate handles all <wiki> writes: finding existing rows, updating or creating, filling properties.

**Allowed <task-tracker> actions (read + comment on existing only):**
- `mcp__asana__asana_get_task`, `mcp__asana__asana_search_tasks` — read.
- `mcp__asana__asana_update_task` — only to fill notes / html_notes on an **existing** task, not to set `completed=true`.
- `mcp__asana__asana_create_task_story` — comment on an existing task with the verdict.

**Forbidden:**
- ❌ `mcp__asana__asana_create_task` — never call from bug-dig. Use /bug-create skill instead.
- ❌ `asana_update_task` with `completed=true` — don't auto-close tickets.
- ❌ `asana_delete_task` — never.

When verdict calls for a new ticket, end the chat reply with a one-liner like:
> Рекомендую завести <task-tracker>-тикет: {Type=Bug, Priority=Medium, Name="..."}. Вызови /bug-create когда будешь готова.

---

## Anti-patterns — check before submitting the artifact

Read each one. If you catch yourself doing any of these, fix before posting.

- ❌ **Creating <task-tracker> tasks from bug-dig.** Never call `mcp__asana__asana_create_task` — that's /bug-create. Suggest a ticket in chat, direct user to /bug-create. Logged incident 2026-04-24: "Не смей создавать тикеты в асана без моего разрешения!".
- ❌ **Severity/priority in description body.** Use the Priority custom field only. Don't write `Severity: 🔴 Critical` in notes — it duplicates and desyncs with the field.
- ❌ **Numbers from the old description without re-verification.** If the task came with "минимум N пользователей" from <metrics>, pull fresh <error-monitoring> numbers and explain the delta. <metrics> raw-500s and <error-monitoring> exception-events don't match 1:1 (different windows, different capture points).
- ❌ **Implicit scope.** Always say explicitly "only Stripe" / "all providers" / "only Web". The Stripe auto-topup ticket was confusing until "Scope:" line was added.
- ❌ **Missing "User impact: ДА/НЕТ" callout.** This is the single most important line. If no user impact, call it out loudly — it drives the priority.
- ❌ **One fix option.** Offer 3, from cheapest to deepest (config → code guard → systemic wrapper fix).
- ❌ **"Critical" priority because of noise.** Low / Technical-debt when user impact is zero, even if volume is huge.
- ❌ **Setting Difficulty.** Skip this custom field entirely — developers own it.
- ❌ **Missing deploy correlation when it matches.** If <error-monitoring> `first_seen` matches a git commit date one-to-one, include commit hash + author. That's the Fix owner.
- ✅ **Структура задачи — обязательные секции:**
  Каждая задача должна содержать: **TL;DR** (1-2 строки — обязательно, первым блоком, до того как читатель углубится в детали), **User Impact: ДА/НЕТ** (отдельная секция, не вшитая в текст), **Root Cause** или **Причина** (отдельный блок, не по ходу текста), **Как чинить**, **Источники**.
  Если есть traceback — выносить в отдельную секцию **Traceback**.
  Если есть критерии готовности — выносить в **Acceptance Criteria**.
- ✅ **Difficulty не трогать** — это поле разработчиков. В вердикте и рекомендации тоже.
- ✅ **Всегда указывать источники данных в задаче.** В конце описания добавлять секцию «Данные» — ссылки на <wiki>-документы, упоминание откуда брались цифры (<data-warehouse>, <analytics>, <error-monitoring>, <logs> и т.д.). Не нужно подробно — достаточно коротко обозначить откуда информация, чтобы разработчик мог перепроверить.
- ✅ **Читаемость текста в <task-tracker> важна.** Используй `<ol>/<li>` для нумерованных блоков — не смешивай нумерацию с жирными заголовками-через-точку. Внутри `<li>` разбивай длинное содержимое на строки через `\n` — каждый факт на свою строку. Длинный список в одну строку — плохо читается человеком.
- ✅ **Пустая строка до и после каждого заголовка секции.** Шаблон: `\n\n<strong>Заголовок</strong>\n\nконтент`. Исключение — самый первый блок на странице: перед ним пустая строка не нужна.
- ❌ **Wrong tags in <task-tracker> html_notes.** <task-tracker> rejects with `xml_parsing_error` (400) for many common HTML tags. **Supported:** `<strong>`, `<em>`, `<u>`, `<s>`, `<code>`, `<ul>`, `<ol>`, `<li>`, `<a href="">`. **NOT supported:** `<p>`, `<br/>`, `<br>`, `<h1>`, `<h2>`, `<h3>`, `<hr/>`, `<pre>`. Always wrap in `<body>...</body>`. Learned from 2026-04-27 debugging session.
- ❌ **`&#10;` for line breaks in html_notes.** It renders as literal text `&#10;`, not a newline. For visual spacing between sections use an actual `\n` newline character in the Python/curl string.
- ❌ **Escaping `\.` `\-` etc. in <task-tracker> html_notes.** <task-tracker> rejects with 400. Pass plain characters; no backslash escapes.

---

## Output

The final artifact is one of:
1. **Verdict in chat** — summary with evidence, User Impact ДА/НЕТ, decision, fix options. Always in chat first. To record to Bug Candidates DB — call **bug-nominate**.
2. **Comment posted to existing <task-tracker> ticket** (if one exists) via `asana_create_task_story` — verdict summary with evidence.
3. **Ticket recommendation** (if verdict warrants a new ticket): one-liner proposing Name + Type + Priority + suggested Fix owner. Direct user to /bug-create — never call `asana_create_task` from bug-dig.

Always summarize what was done in 2-3 lines so the user can review fast.
