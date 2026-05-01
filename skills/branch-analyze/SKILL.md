---
name: branch-analyze
description: >-
  QA-анализ фиче-ветки по задаче. Принимает <task-tracker> URL, задачу DEV-XXXX или имя
  ветки. Находит MR в <vcs>, проверяет деплой на фиче-окружение, читает diff
  и задачу в <task-tracker>, прогоняет автоматические <logs>/<error-monitoring>-проверки, создаёт
  <wiki>-страницу с конкретными флоу для ручного тестирования.
  Triggers: "/branch-analyze", "проверь ветку", "что тестировать по DEV-XXXX",
  "чеклист для задачи", <task-tracker> URL.
---

# branch-analyze

## Constants

- `GITLAB_BACKEND` = `3`
- `GITLAB_FRONTEND` = `5`
- `GITLAB_ADMIN` = `6`
- `REPO_BACKEND` = `<product-dir>/backend`
- `REPO_FRONTEND` = `<product-dir>/frontend`
- `REPO_ADMIN` = `<product-dir>/admin`
- `SENTRY_BACKEND` = `<your-error-monitoring-project>`
- `SENTRY_FRONTEND` = `<product>-frontend`
- `SENTRY_STAGE_ENV` = `staging`
- `LOKI_UID` = `<logs>`
- `LOKI_BACKEND` = `backend-dev-{task_id}`
- `LOKI_SCHEDULER` = `call_scheduler-dev-{task_id}`
- `LOKI_WORKER_REGEX` = `worker_.*-dev-{task_id}`
- `LOKI_STAGE_BACKEND` = `backend`
- `LOKI_STAGE_SCHEDULER` = `call_scheduler`
- `LOKI_STAGE_WORKER_REGEX` = `worker_.*`
- `LOKI_STAGE_ENV` = `staging`
- `NOTION_BRANCH_PARENT` = `34f98d8a3c9b8124ad7ded1cac7522ac`
- `NOTION_BUG_CANDIDATES_DS` = `collection://26b9b9ff63194e88af44b30a6978600d`
- `ENV_FEATURE_FRONT` = `https://{task_id}.staging.zncr.pro`
- `ENV_FEATURE_API` = `https://{task_id}-api.staging.zncr.pro`
- `ENV_FEATURE_ADMIN` = `https://adm.zncr.pro?env=preview-{task_id}`
- `ENV_STAGE_FRONT` = `https://openmov.zncr.pro`
- `ENV_STAGE_API` = `https://api.zncr.pro`
- `ENV_STAGE_ADMIN` = `https://adm.zncr.pro`

`task_id` — lowercase из имени ветки: `fix/DEV-1727-fix-<internal-service>` → `dev-1727`.

---

## Шаг 0 — Sync репозиториев

Полный sync через `/git-refresh` (pull all 3 + `code-review-graph update`).
Дополнительно — fetch для feature-веток (нужен для diff в Шаге 5):

```bash
for r in $REPO_BACKEND $REPO_FRONTEND $REPO_ADMIN; do
  git -C $r fetch origin
done
```

Если `/git-refresh` уже выполнен в этой сессии — fetch достаточно.

---

## Шаг 1 — Парс инпута

| Формат | Действие |
|---|---|
| <task-tracker> URL | GID → `asana_get_task` → DEV-XXXX из названия |
| `DEV-1727` | использовать |
| `fix/DEV-1727-...` | regex `DEV-[0-9]+` |

Сохранить: `TASK_NUM` (uppercase), `TASK_ID` (lowercase), `asana_task` если уже фетчили.

Если DEV-XXXX не извлекается — стоп, спросить.

---

## Шаг 2 — MR в <vcs>

Для каждого репо:
```
mcp__gitlab__list_merge_requests(project_id=<GITLAB_*>, state="all", search=TASK_NUM)
```

Выбрать MR где `source_branch` содержит TASK_NUM. Если несколько — самый свежий по `created_at`.

Сохранить: `branch`, `mr_url`, `mr_iid`, `mr_state` (opened/merged/closed).

Если ни в одном репо нет MR — стоп, сообщить.

---

## Шаг 3 — Деплой

**`merged`** → тестируем на stage. Установить `ENV_TYPE = stage`:
- `FEATURE_URL = ENV_STAGE_FRONT`, `API_URL = ENV_STAGE_API`, `ADMIN_URL = ENV_STAGE_ADMIN`

**`closed`** → спросить пользователя: _"MR закрыт без влития. Тестируем на общем stage или пропускаем?"_
- Пропустить → стоп
- Тестировать на stage → установить `ENV_TYPE = stage`, `FEATURE_URL = ENV_STAGE_FRONT`, `API_URL = ENV_STAGE_API`, `ADMIN_URL = ENV_STAGE_ADMIN`

**`opened`** → `mcp__gitlab__get_merge_request` → `head_pipeline.status`:

| status | Действие |
|---|---|
| `success` | `deploy_time = head_pipeline.finished_at`, продолжать |
| `running`/`pending`/`created` | стоп, сообщить |
| `failed`/`canceled` | стоп, дать `head_pipeline.web_url` |
| `null` | стоп, "пайплайн не запускался" |

Если success: установить `ENV_TYPE = feature`, `FEATURE_URL/API_URL/ADMIN_URL = ENV_FEATURE_*`.

---

## Шаг 4 — <task-tracker> задача

Если `asana_task` уже есть — переиспользовать. Иначе:
```
asana_search_tasks(text=TASK_NUM) → asana_get_task(gid)
```

Обязательно прочитать:
1. Тело задачи (описание / Acceptance Criteria)
2. Комментарии: `asana_get_task_stories(task_gid)` — читать все, особенно от разработчиков
3. Если у задачи есть родитель (`parent` поле не null) → `asana_get_task(parent.gid)` + `asana_get_task_stories(parent.gid)` — тело и комментарии родительской задачи тоже

Контекст из родителя часто содержит общее ТЗ, дизайн-решения и AC, которых нет в дочерней задаче.

---

## Шаг 5 — Diff

```bash
git -C <repo> diff origin/main...origin/<branch> --name-only
```

Ноль файлов — пропустить репо.

---

## Шаг 6 — Категоризация изменений

| Path | Область | Риск |
|---|---|---|
| `backend/app/handlers/billing/`, `backend/domain/billing/` | Billing | P0 |
| `backend/app/handlers/credits/`, `backend/domain/credits/` | Credits | P0 |
| `backend/infra/external/clients/` | External integrations | P0 |
| `backend/infra/database/psql/tables/` | DB schema | P0 |
| `backend/app/handlers/tasks/`, `backend/domain/tasks/` | Generation | P1 |
| `backend/app/handlers/auth/`, `backend/domain/auth/` | Auth | P1 |
| `backend/app/handlers/trusted/`, `backend/domain/trusted/` | Trusted | P1 |
| `backend/infra/tools/base/tool.py` | Guardrails | P1 |
| `backend/app/handlers/templates/`, `backend/domain/templates/` | Templates | P2 |
| `backend/infra/external/clients/http/session/wrapper.py` | HTTP wrapper | P1 |
| `frontend/src/` | UI | P2 |
| `admin/` | Admin | P2 |
| остальное | Other | P3 |

---

## Шаг 7 — Глубокое чтение

Для каждого изменённого файла: diff + Read целиком.

**Backend handlers:** какие endpoints, границы транзакций (`commit/rollback`), операции с кредитами, изменения внешних вызовов (Stripe/Payblis/<analytics>/dbus), новые исключения raised/swallowed.

**Backend domain:** изменённый инвариант, idempotency-гарантии (`get_by_X` перед insert).

**DB tables:** новые колонки (nullable? default?), constraints (UNIQUE/FK), обратимость миграции.

**External clients:** провайдер, обработка ошибок swallow vs raise. `wrapper.py:35-44` логирует тело в ERROR перед raise — норма, не баг.

**Frontend:** затронутые флоу, billing/credit UI, валидация форм.

**Admin:** изменённые фичи, новые мутации.

---

## Шаг 8 — Bug Candidates cross-check

```
mcp__notion__notion-search(
  query=<keyword>, filters={},
  data_source_url=NOTION_BUG_CANDIDATES_DS
)
```

Активные пересечения с изменённой областью → ⚠️ в сценарий.

**Regression zones — всегда при касании:**

| Область | Что |
|---|---|
| Credits | Race conditions, idempotency, баланс |
| Generation | POST /assets/batch — порог 1640, 33s timeout |
| Trusted | pending → active → suspended |
| Templates | Aspect ratio (хроническая регрессия: DEV-1320, 1490, 1560) |
| Guardrails | Trusted обходят, обычные нет |
| Новый AI tool | Полный smoke: submit → poll → результат |

---

## Шаг 9 — Автопроверки

Окно: `deploy_time` из Шага 3 как `startRfc3339`. Для merged MR — дата последнего коммита:
```bash
git -C <repo> log origin/<branch> -1 --format=%ci
```

**<logs> backend:**

Feature-окружение (`ENV_TYPE = feature`):
```
{service_name="LOKI_BACKEND"} |= `ERROR`
```

Stage-окружение (`ENV_TYPE = stage`):
```
{environment="LOKI_STAGE_ENV", service_name="LOKI_STAGE_BACKEND"} |= `ERROR`
```

**<logs> workers** (если задета генерация — `backend/app/handlers/tasks/` или `backend/infra/tools/`):

Feature:
```
{service_name=~"LOKI_WORKER_REGEX"} |= `ERROR`
```

Stage:
```
{environment="LOKI_STAGE_ENV", service_name=~"LOKI_STAGE_WORKER_REGEX"} |= `ERROR`
```
Топ-3 уникальных. Извлечь задеплоенные `service_name`. Отсутствующий воркер → ⚠️ в флоу.

**<error-monitoring>:**

Всегда фильтровать по окружению: `environment=SENTRY_STAGE_ENV` (т.е. `environment=staging`).

Backend (если затронут backend):
```
mcp__sentry__list_issues(
  projectSlug=SENTRY_BACKEND,
  query="is:unresolved environment:staging firstSeen:><deploy_date>"
)
```

Frontend (если затронут frontend):
```
mcp__sentry__list_issues(
  projectSlug=SENTRY_FRONTEND,
  query="is:unresolved environment:staging firstSeen:><deploy_date>"
)
```

Формат фактов: `✅/⚠️/❌ <logs>: N ERROR ...`, `✅/⚠️ <error-monitoring> [SENTRY_BACKEND, env=staging]: N issues`, `✅/⚠️ <error-monitoring> [SENTRY_FRONTEND, env=staging]: N issues`.

---

## Шаг 10 — Тестовые флоу

На основе шагов 6-7 + контекста задачи (4) — конкретные пользовательские флоу.

- Один флоу = один сценарий с реальными URL и ожидаемыми результатами
- P0 → P1 → P2
- Указывать что смотреть в <logs>/<error-monitoring> после флоу
- Не дублировать автопроверки из Шага 9

**Структура:**
```
### Флоу N: <название>
<предусловие если нужно>

1. <действие> → <результат>
2. <действие> → <результат>
3. <logs>: `{service_name="LOKI_BACKEND"} |= "<keyword>"` — <что должно/не должно>
```

❌ "Открыть FEATURE_URL" не шаг — это общее предусловие.
❌ OAuth/whitelisted redirect — выносить в `### Тестирование на stage` с причиной.

---

## Шаг 11 — Промежуточный репорт

```
## Готово к созданию страницы — DEV-XXXX
**Фиче-окружение:** <FEATURE_URL>
**Admin:** <ADMIN_URL>
**Деплой:** ✅/❌

**Автопроверки:**
- <logs>: ...
- <error-monitoring>: ...

**Флоу:** N сценариев (P0: X, P1: Y, P2: Z)
**Bug Candidates:** ...

Создаю <wiki>-страницу...
```

---

## Шаг 12 — <wiki> страница

```
mcp__notion__notion-create-pages(
  parent={type: "page_id", page_id: NOTION_BRANCH_PARENT},
  pages=[{
    properties: {title: "<TASK_NUM> — <название>"},
    content: <структура ниже>
  }]
)
```

Перед вызовом: `<wiki>://docs/enhanced-markdown-spec`.

**Структура:**
```
## <TASK_NUM> — <название>

**Фиче-окружение:** <FEATURE_URL>
**Admin:** <ADMIN_URL>
**API:** <API_URL>
**MR:** <mr_url> (<mr_state>)

---

### Предусловие
Открыть: <FEATURE_URL>
<общие предусловия>

---

### Автоматические проверки
<результаты Шага 9>

---

### Флоу 1: ...
### Флоу 2: ...

---

### Дополнительные проверки
<regression zones и smoke не вошедшие в флоу>

---

### Тестирование на stage
<только если есть сценарии недоступные на review env>
Дождаться merge → ENV_STAGE_FRONT
<список с причинами>
```

После создания — вывести кликабельную ссылку в чат:
`[DEV-XXXX — <название>](https://www.<wiki>.so/<page_id_without_dashes>)`

---

## Hard rules

- ❌ Не создавать <task-tracker>-задачи
- ❌ Не пропускать P0 области (billing, credits всегда полностью)
- ❌ Деплой failed/идёт → дать `head_pipeline.web_url`, стоп
- ❌ MR closed без влития → не стопить автоматически, спросить пользователя
- ✅ Всегда: diff + полный файл (Read), автопроверки до <wiki>, флоу с реальными URL
- ✅ MR merged → stage (`ENV_TYPE = stage`), не feature
- ✅ MR closed + пользователь выбрал stage → `ENV_TYPE = stage`
- ✅ MR opened + pipeline success → feature env (`ENV_TYPE = feature`)
- ✅ <logs>: feature env — по service_name (LOKI_BACKEND); stage env — по environment+service_name (LOKI_STAGE_*)
- ✅ <error-monitoring>: всегда `environment=staging`, указывать projectSlug явно; frontend changes → проверять SENTRY_FRONTEND
- ✅ <logs> всегда с временным окном от деплоя
- ✅ Bug Candidates cross-check обязателен
- ✅ <task-tracker> задача из Шага 1 переиспользуется в Шаге 4
- ✅ После создания <wiki>-страницы — вывести кликабельную ссылку в чат
