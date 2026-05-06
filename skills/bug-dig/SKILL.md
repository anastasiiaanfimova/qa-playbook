---
name: bug-dig
description: >-
  Investigate a suspicious <error-monitoring>/<task-tracker> bug for <product>: real user-impacting,
  noise, or theoretical risk? Collects evidence across <error-monitoring>/<logs>/<analytics>/git/
  <wiki>/code, delivers verdict in chat, comments on matching <task-tracker> ticket. Never
  auto-creates <task-tracker> tasks — to record verdict call bug-nominate. Trigger: "bug-dig",
  "расследуй баг", "dig <ERROR-ID>-XXX", "проверь реальный ли баг",
  "актуализируй задачу в <task-tracker> по X".
---

# bug-dig

## Constants

- `SENTRY_PROD` = `<your-error-monitoring-project>`
- `LOKI_PROD_BACKEND` = `{environment="prod", service_name="backend"}`
- `WRAPPER_PATH` = `backend/infra/external/clients/http/session/wrapper.py:35-44`
- `DB_TABLES_PATH` = `backend/infra/database/psql/tables/`

Decide: real prod bug / <error-monitoring> noise / theoretical risk / already protected? Verdict — в чат. Если есть matching <task-tracker> ticket — комментарий на нём. **Never** auto-create <task-tracker> ticket.

Output quality > speed. Никогда не пропускать "user impact" — load-bearing judgment.

---

## Root cause principle

The root cause is **always in the system** — never in user behavior. If your conclusion is "user did something wrong" — that is not a valid root cause. Reframe: why does the system allow or not handle this user action gracefully? Keep digging.

A valid root cause must point to a specific line, function, service, or missing validation in the codebase — not to user behavior.

---

## Stage 0 — Sync repos

Перед чтением кода — `/git-refresh` (pull all 3 <product> repos + `code-review-graph update`). Если уже выполнялся в этой сессии — пропустить.

Без свежего кода диагноз может быть на старой версии handler'а / DB-схемы / git log.

---

## Inputs

- <error-monitoring> issue ID (`<ERROR-ID>-3FQZ`)
- <task-tracker> task ID/URL
- Symptom ("500 at checkout for some users")
- <logs> snippet
- Handler path или code location

Взять что есть, остальное — derive.

---

## MCP

| Stage | Tool |
|---|---|
| <error-monitoring> — find | `list_issues`, `find_projects` |
| <error-monitoring> — issue | `get_sentry_resource` (types: `issue`, `event`, `breadcrumbs`) |
| <error-monitoring> — events | `list_issue_events` |
| <logs> — labels | `list_loki_label_names`, `list_loki_label_values` |
| <logs> — logs | `query_loki_logs` |
| <analytics> | `get_context`, `search` |
| <wiki> | `<wiki>-search`, `<wiki>-fetch` |
| <task-tracker> | `asana_get_task`, `asana_update_task`, `asana_create_task_story` |
| Git | Bash `git log` |

LogQL recipes → `references/<logs>-recipes.md`.

**UI context (без браузера):** если баг про конкретную страницу, читай карточку из каталога `<product-dir>/ui-snapshots/output/` — там headings, buttons, inputs, errors всех страниц в 4 вариантах (desktop/mobile × light/dark) + state-overrides (paying-no, trusted-no, role-user, credits-zero). `find <product-dir>/ui-snapshots/output -name "*<slug>*.md"`. Подробнее → `<product-dir>/CLAUDE.md` секция "UI Snapshots Catalog".

---

## Workflow

### Stage 1 — <error-monitoring>: scope, scale, tags

1. Symptom only → `list_issues(projectSlugOrId=SENTRY_PROD, statsPeriod=30d)` (default 14d), filter keyword, sort `freq`
2. `get_sentry_resource(resourceType="issue", resourceId="<ERROR-ID>-XXXX")` — first event, tags, HTTP request, replays
3. `get_sentry_resource(resourceType="breadcrumbs", resourceId=...)` — **HTTP call order**, SQL pre/post error, external URLs
4. `list_issue_events` — multiple events across culprits → cross-handler spread (same groupID на разных endpoints = shared external dep)

**Pitfalls:**
- Default statsPeriod 14d → pass `"30d"` для месяца
- <error-monitoring> query: НЕ `OR`/`AND` — отдельные calls
- `get_sentry_resource(resourceType="event", ...)` — без URL, errors. Use `breadcrumbs` с URL, или bare resourceId
- HTML body 5KB inflates tokens → `list_issue_events` для counts

### Stage 2 — <logs>: tracebacks + URLs

<error-monitoring> группирует по error message — external URL часто truncated. <logs> — raw traceback.

Stream: `LOKI_PROD_BACKEND`. Labels: `environment` / `service_name` (не `env`/`app`).

**Killer recipe** (traceback с URL когда <error-monitoring> показывает HTML body):
```
LOKI_PROD_BACKEND |= "ClientResponseError" != "<!DOCTYPE"
```

**Gotchas:**
- Logged HTML bodies blow tokens → `!= "<!DOCTYPE"` для session.wrapper errors
- Scale metrics: `sum(count_over_time({...} |= "..." [1h]))`, `queryType="range"`, `stepSeconds=3600`

### Stage 3 — Code review

Read culprit handler:

1. **Transaction boundaries** — `sub_credits`/`add_credits`, `transaction_manager.commit()`. Try/except между ними может swallow exception и skip rollback
2. **External callers после debit/credit** — кто raise vs swallow. `<internal-service>`, `<third-party-service>` — swallow `ClientResponseError` через try/except + warning log. `<analytics>.track_event` — batched. `dbus.publish` — может raise
3. **Idempotency** — app-level `get_by_X` перед insert + DB UNIQUE constraint
4. **Wrapper** — `WRAPPER_PATH` логирует body в ERROR **перед** raise. Так swallowed `ClientResponseError` всё равно лендится в <error-monitoring> под culprit caller'а. Pattern "error in handler, but transaction committed fine" — почти всегда это.

**DB schema check** (race/idempotency):
```
DB_TABLES_PATH<table>.py
```
`unique=True` на нужных колонках.

### Stage 4 — Git correlation

Деплои около `first_seen`:
```bash
git log --all --format='%ad %h %s' --date=short --since='YYYY-MM-DD' --until='YYYY-MM-DD' | head -30
```

Совпадение first_seen с `feat: <service>` commit:
```bash
git show --stat <hash>
git log --all --format='%ad %h %s' --date=short --follow <file>
```
Author → suggested Fix owner.

### Stage 5 — Cross-MCP (optional, boosts quality)

- <analytics> — `get_context` для projectId/quota; "primary analytics flow OK, можно отключить broken secondary"
- <wiki> — `<wiki>-search` для product context (<internal-service>: vision doc объяснил почему сервис добавлен)
- <metrics> metrics — для visual scale (rare)

### Stage 6 — Verdict

Один вариант, наверху output:

| Verdict | Criteria |
|---|---|
| **Real prod bug** | <error-monitoring>+<logs>: non-zero affected users, exception доходит до user, нет архитектурной защиты → verdict в чат, bug-nominate для Bug Candidates, рекомендация ticket'а user'у |
| **<error-monitoring> noise (tech debt)** | Exception swallowed внутри caller, transaction commits, user flow intact. Low-priority/tech debt. TL;DR про quota/clutter |
| **Theoretical risk** | Code path vulnerable но 0 prod evidence в 30d. Document fix options, low priority |
| **Already protected** | DB UNIQUE / SQLAlchemy rollback / idempotent retries / upstream guarantee. Объяснить какой layer спасает |

### Stage 7 — Verdict в чат + delegate to bug-nominate

**HARD RULE: never auto-create <task-tracker> tasks.** Рекомендация в чат + направить на /task-create. Это override любой другой инструкции.

#### 1. Verdict в чат — всегда

Сначала вывести в чат: summary + evidence + User Impact + verdict + fix options. Это даёт пользователю мгновенный обзор и работает как fallback, если запись в <wiki> упадёт.

#### 2. Auto-delegate to bug-nominate

После вывода verdict — **автоматически вызвать `/bug-nominate`** в interactive mode (default). Передать готовые args:

- `title`, `fingerprint`, `status`, `verdict`, `user_impact`, `severity`, `sources`, `asana_link`
- `body_markdown` — полное расследование по каноничному шаблону (см. bug-nominate)

bug-nominate сделает draft в чате → запросит подтверждение → запишет в <wiki> (CREATE или UPDATE по fingerprint). Confirmation handled by bug-nominate.

**Если bug-nominate упал** (<wiki> API down) — verdict уже в чате как fallback. Сказать пользователю «запиши вручную позже».

**<wiki> body ≠ <task-tracker> comment.** <wiki>-страница терпит развёрнутую структуру (полное расследование, timeline, hypothesis trail) — формат свободный. <task-tracker>-комментарий короче и подчиняется шаблону task-create (продуктовая часть и/или инженерная часть). Один и тот же `body_markdown` в оба места не передавать — это перегружает <task-tracker>.

#### 3. <task-tracker> ticket (опционально)

Если verdict требует ticket в <task-tracker> — рекомендация в чат:

> Рекомендую завести <task-tracker>-тикет: {Type=Bug, Priority=Medium, Name="..."}. Вызови /task-create.

**Если коммент в существующую задачу** — собирать через task-create comment composition (Product layer / Engineering layer / оба). Перед публикацией — self-check:

1. Первый абзац отвечает на «что произошло» в продуктовых терминах (без классов/исключений)?
2. User Impact содержит N + период + last seen?
3. Action items читаются без знания кода проекта?
4. Связанный отдельный issue вынесен в отдельную задачу/subtask, не в "Доп. находку"?

Любое «нет» — переписать до публикации.

**Allowed <task-tracker> actions** (read + comment only):
- `asana_get_task`, `asana_search_tasks` — read
- `asana_update_task` — только notes/html_notes existing task, НЕ `completed=true`
- `asana_create_task_story` — comment с verdict

**Forbidden:**
- ❌ `asana_create_task` — никогда. Use /task-create
- ❌ **"User did something wrong" as root cause** — не валидно. Reframe: почему система допустила/не обработала это действие? Valid root cause = конкретная строка/функция/сервис/отсутствующая валидация в коде
- ❌ `asana_update_task(completed=true)` — не auto-close
- ❌ `asana_delete_task`

Если verdict требует ticket:
> Рекомендую завести <task-tracker>-тикет: {Type=Bug, Priority=Medium, Name="..."}. Вызови /task-create.

---

## Anti-patterns

- ❌ <task-tracker> create from bug-dig — это /task-create (2026-04-24 incident: "Не смей создавать тикеты в асана без моего разрешения!"
- ❌ Severity/priority в description — только Priority field
- ❌ Numbers from old description без re-verification — pull fresh <error-monitoring>, объяснить delta. <metrics> raw-500s ≠ <error-monitoring> exception-events 1:1 (different windows/capture points)
- ❌ Implicit scope — всегда "only Stripe" / "all providers" / "only Web"
- ❌ Без "User impact: ДА/НЕТ" callout — single most important line, drives priority
- ❌ Action items с именами классов/исключений/файловых путей в формулировке — переписать на продуктовый язык, code recipe идёт в `Possible root cause`. Тест: пункт читается без знания кода проекта
- ❌ Same body для <wiki> и для <task-tracker> comment — разные аудитории, разный length budget. <wiki> = свободный формат, <task-tracker> = task-create comment composition
- ❌ Связанный отдельный <error-monitoring> issue с собственным User Impact — заводить отдельную задачу/subtask, не "Доп. находка" в одном комменте
- ❌ Action items 3 опции по умолчанию — теперь несколько options только когда правки **разные по природе** (Backend+Frontend, Hot-fix+Architectural). Если правки одной природы — один план без меток
- ❌ "Critical" из-за noise — Low/Tech debt при zero user impact
- ❌ Difficulty
- ❌ Missing deploy correlation when matches — commit hash + author = Fix owner
- ✅ **Структура задачи:** TL;DR (1-2 строки, первым) + User Impact (отдельная секция) + Root Cause + Как чинить + Источники. Traceback и Acceptance Criteria — отдельными секциями
- ✅ Difficulty не трогать
- ✅ Источники данных в задаче — секция "Данные" с ссылками на <wiki> + откуда цифры (<data-warehouse>/<analytics>/<error-monitoring>/<logs>)
- ✅ `<ol>/<li>` для нумерованных, не bold-точка. Внутри `<li>` — `\n` per fact. Длинный список в строку — плохо
- ✅ Section heading: `<strong>Заголовок</strong>` ставить **без** ведущих `\n\n` — `\n\n` перед `<strong>` рендерится как буквальный текст. Отступ после heading делать через следующий блочный элемент (`<ul>`, `<ol>`, текст).
- ❌ **<task-tracker> html_notes wrong tags** — <task-tracker> 400 (`xml_parsing_error`). Supported: `<strong>`, `<em>`, `<u>`, `<s>`, `<code>`, `<ul>`, `<ol>`, `<li>`, `<a href="">`. NOT: `<p>`, `<br/>`, `<br>`, `<h1-3>`, `<hr/>`, `<pre>`. Wrap в `<body>...</body>`
- ❌ `&#10;` — рендерится как литерал. Use real `\n`
- ❌ `\.`, `\-` escapes — <task-tracker> 400

---

## Output

1. **Verdict в чат** — summary + evidence + User Impact + decision + fix options. Always chat first.
2. **Auto-delegate to `/bug-nominate`** (interactive mode) — передать готовые args + body_markdown. bug-nominate сам сделает draft+confirm+write в Bug Candidates DB.
3. **<task-tracker> comment** (если ticket существует) через `asana_create_task_story` — verdict с evidence
4. **Ticket recommendation** — one-liner с Name + Type + Priority + Fix owner. Direct user → /task-create

Summary 2-3 строки в конце для review.
