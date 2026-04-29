---
name: bug-create
description: >-
  Create a well-formatted <task-tracker> bug task for <product>. Use after bug-dig
  confirms a real bug. Covers required sections, field values, HTML formatting
  rules, and the writing style the team expects. Always show draft for approval
  first; NEVER create the task without explicit user confirmation.
  Trigger: "создай задачу", "заведи задачу", "оформи баг", "create bug task", "log a bug".
---

# Bug Create — <product> <task-tracker> task creation

**HARD RULE: always show the full draft in chat and wait for explicit "да / пиши / создавай" before writing anything.** Do not create or post silently.

## Routing

| Что сказал пользователь | Что делаем |
|---|---|
| "создай баг" / "заведи задачу" — без ссылки | Step 0 → дубль-чек → новая задача |
| "создай баг" + ссылка на существующую задачу | Полный комментарий → `asana_create_task_story` |
| Дубль найден, пользователь говорит "обновить" | Комментарий → `asana_create_task_story` |

Если дана ссылка на задачу — пишем полное описание в комментарий, задачу не трогаем.  
Если просят только обновить цифры — это `bug-comment`, не этот скил.

---
## Step 0 — Duplicate check

**Обязателен всегда** — кроме случая когда пользователь уже дал ссылку на задачу.

1. Из названия/описания бага выдели 2–3 ключевых слова (имя исключения, фича, провайдер, endpoint).
2. Вызови `asana_search_tasks` с этими словами, `project=1210659552423846`.
3. Покажи найденные задачи в чате (название + ссылка + статус). Если ничего нет — скажи «похожих задач не нашла» и переходи к Step 1.
4. Если нашлись похожие — спроси:

   > Нашла похожие задачи: [список]. Обновить одну из них или создать новую?

5. **"обновить"** → напиши полное описание как комментарий через `asana_create_task_story` (показать черновик перед отправкой).  
   **"новая"** → продолжай к Step 1.

**Не перебарщивай с запросами**: один-два точечных запроса достаточно. Если первый вернул >5 результатов — покажи топ-3 наиболее близких.

---
## Step 1 — Draft the task name

Pattern: `Что — Где — Когда`

Answer the three questions in this order. Include only what you know — skip what you don't.

- **Что** — what breaks or behaves wrong (required)
- **Где** — in which feature / endpoint / provider / screen (include if known)
- **Когда** — under what condition it happens (include if known and non-obvious)

Good examples:
- `Stripe / Mollie / Crypto — 500 при создании платежа` (что + где)
- `NoSuchKey в /api/v1/assets/{id}/preview: orphan assets возвращают 500 вместо 404` (что + где + когда)
- `<internal-service> CRM — 404 на api.<internal-service>.sh засоряет <error-monitoring> (5925 users / 12k events)` (что + где)
- `normalized_email is not in lower case` (что — где/когда очевидны из контекста)

No prefixes like `[BUG]`, no DEV-XXXX (auto-assigned), no severity emoji in the name.

---
## Что — Где — Когда: applies everywhere, not just the name

The same three questions apply to every number inside the description body.

- **Что** — what metric / what symptom
- **Где** — which endpoint, provider, screen
- **Когда** — under what condition *or* over what time window

**Когда** has two meanings and both are valid:
- Trigger condition: "при загрузке нескольких файлов", "только для iOS"
- Time window: "за 34 дня", "с 2026-03-24 по 2026-04-27"

**Rule: a number without a time window is incomplete.** 946 users over a year ≠ 946 users over a week. Always attach the period to every metric — in User Impact, in Evidence, in TL;DR if space allows. Use `first_seen — last_seen` from <error-monitoring> as the default range.

---
## Step 2 — Build html_notes from the template

Always wrap the entire body in `<body>...</body>`.

### Required sections (all bugs)

```
<strong>TL;DR</strong>

[1-2 sentences. What breaks. What the user sees as a result. No explanation of root cause here — just the symptom and impact at a glance.]

<strong>User Impact:</strong> да / нет / возможно

[If да: числа прямо здесь — N пользователей, N events за N дней. Что конкретно сломано для них — не могут заплатить, видят 500, зависают на 20 мин. Без ссылок — ссылки идут в Evidence.]
[If нет: почему всё равно проблема — <error-monitoring> quota, observability clutter, теоретический риск. Числа тоже если есть.]
[If возможно: при каком сценарии станет реальным и насколько вероятно. Числа если есть.]

<strong>Where It Breaks</strong>

<ul>
<li><code>backend/path/to/file.py</code></li>
</ul>

<strong>Possible Root Cause</strong>

[Claude analysis based on code + logs — not manually verified. Which line raises, which guard is missing, which external API misbehaves. Code snippets welcome if they fit in 3-5 lines.]

<strong>Fix Options (cheap → deep)</strong>

<ol>
<li>[cheapest fix, config or 1-line guard]</li>
<li>[proper code fix]</li>
<li>[deeper architectural fix if warranted]</li>
</ol>

<strong>Acceptance Criteria</strong>

<ul>
<li>[What "done" looks like — which error disappears, what user sees instead]</li>
</ul>

<strong>Evidence</strong>

<ul>
<li><error-monitoring>: <a href="https://<error-monitoring>.zncr.pro/..."><ERROR-ID>-XXXX</a> — [N users], [N events], [timeframe]</li>
<li><logs>: [if used]</li>
<li><analytics>: [if used]</li>
<li><wiki> Bug Candidates: [link if exists]</li>
</ul>
```

### Optional sections — add when relevant

| Section | When to add |
|---|---|
| `<strong>Traceback</strong>` | When <logs> log has the raw stacktrace and it explains the cause better than prose |
| `<strong>Deploy Correlation</strong>` | When <error-monitoring> `first_seen` matches a git commit date exactly — include commit hash + author |
| `<strong>Scope</strong>` | When impact is narrow (only one endpoint, only one user group, only stage) |
| `<strong><metrics></strong>` | When <metrics> 500 counts differ from <error-monitoring> — explain why |
| `<strong>Why This Still Matters</strong>` | Only when User Impact is нет — explain the downstream harm (quota, noise, reliability) |
| `<strong>Risks</strong>` | When a fix is proposed — list what could break. Format: `<область> — <оценка>: <объяснение>`. Если рисков нет и пользователь не просил явно — пропустить блок. |

---
## Step 3 — Set custom fields

Project ID: `1210659552423846`

Always set **Type** and **Priority**. Never set Difficulty (that's developers' field).

```
custom_fields = {
  "1210659552576086": "<type_gid>",     // Type
  "1210283371559268": "<priority_gid>"  // Priority
}
```

### Type GIDs

| Type | GID |
|---|---|
| Bug | `1210659552576089` |
| Technical debt | `1214086483702992` |
| Enhancement | `1210659552576088` |
| Feature | `1210659552576087` |
| Research | `1214086483702991` |
| QA | `1210875975774528` |

### Priority GIDs

| Priority | When | GID |
|---|---|---|
| Critical | Production broken for many / revenue loss | `1211912615894091` |
| High | Real user pain, reproducible, significant reach | `1210283371559271` |
| Medium | Real issue, limited reach or workaround exists | `1210283371559272` |
| Low | No user impact, noise, tech debt | `1210283371559273` |

---
## Step 4 — HTML formatting rules

These are hard constraints — <task-tracker> rejects or silently mangles anything else.

**Supported tags:** `<strong>`, `<em>`, `<u>`, `<s>`, `<code>`, `<ul>`, `<ol>`, `<li>`, `<a href="">`

**NOT supported (will cause 400 or break layout):** `<p>`, `<br>`, `<br/>`, `<h1>`–`<h3>`, `<hr>`, `<pre>`

**Spacing:** Add a blank line (`\n\n`) before each section heading. Pattern:
```
\n\n<strong>Заголовок</strong>\n\nконтент
```
Exception: no blank line before the very first block.

**Line breaks inside `<li>`:** Use `\n` for visual spacing inside a list item. Each key fact on its own line.

**No `&#10;`** — renders as literal text.

**No backslash escapes** (`\.`, `\-`) — <task-tracker> rejects with 400.

---
## Writing style

Write the description in the same tone you use when explaining something to a colleague — not a formal report. Follow Anastasiia's style:

- Short, direct sentences. No padding.
- Facts inline: numbers, dates, file paths directly in the prose where they fit.
- Cause-effect via "из-за того, что", "потому что", "чтобы".
- "Мы" language — "наш exception_map", "мы не логируем".
- Refer to affected users by role/count, not by email: "пользователь из задачи", "5 пользователей".
- Don't explain what code does if the reader can read it. Explain why it causes the problem.

---
## Anti-patterns

- ❌ Skipping the duplicate check (Step 0). Always search before drafting.
- ❌ Writing the task without user confirmation. Always draft first.
- ❌ Setting Difficulty field. Developers own it.
- ❌ Writing `Severity: Critical` in the body — use the Priority custom field only.
- ❌ Putting User Impact text buried in TL;DR. It must be its own `<strong>User Impact: да/нет</strong>` section.
- ❌ Offering only one fix option. Always 3, from cheapest to deepest.
- ❌ Forgetting Evidence. Developer needs to know where numbers came from.
- ❌ Using `<h2>`, `<p>`, `<br>` — <task-tracker> will 400 or mangle.
- ❌ Skipping TL;DR. It's the first thing a developer reads. Required.
