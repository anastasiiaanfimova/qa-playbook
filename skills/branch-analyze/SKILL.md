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
- `LOKI_UID` = `<logs>` (datasourceUid для mcp__grafana__query_loki_logs)
- `LOKI_BACKEND` = `backend-{back_task_id}`
- `LOKI_WORKER_REGEX` = `worker_.*-{back_task_id}`
- `LOKI_STAGE_BACKEND` = `backend`
- `LOKI_STAGE_WORKER_REGEX` = `worker_.*`
- `LOKI_STAGE_ENV` = `staging`
- `NOTION_BRANCH_PARENT` = `34f98d8a3c9b8124ad7ded1cac7522ac`
- `NOTION_BUG_CANDIDATES_DS` = `collection://26b9b9ff63194e88af44b30a6978600d`
- `ENV_FEATURE_FRONT` = `https://{front_task_id}.staging.zncr.pro`
- `ENV_FEATURE_API` = `https://{back_task_id}-api.staging.zncr.pro`
- `ENV_FEATURE_ADMIN` = `https://adm.<product>.pro/login?env=preview-{back_task_num}`
- `ENV_STAGE_FRONT` = `https://openmov.zncr.pro`
- `ENV_STAGE_API` = `https://api.zncr.pro`
- `ENV_STAGE_ADMIN` = `https://adm.<product>.pro`
- `LOKI_ANOMALY_RATIO_HIGH` = `3.0` (feature/stage ratio выше → ⚠️ rose)
- `LOKI_ANOMALY_RATIO_LOW` = `0.3` (feature/stage ratio ниже → ⚠️ dropped)
- `LOKI_ANOMALY_MIN_COUNT` = `10` (минимум events на feature чтобы считать ratio — иначе single-event flukes)

### Variables (выводятся в Шаге 1-2)

| Variable | Пример | Где используется |
|---|---|---|
| `TASK_NUM` | `DEV-1727` | <task-tracker> search, заголовки <wiki> |
| `task_id` | `dev-1727` | парсинг ветки, до Шага 2.2 |
| `task_num` | `1727` | парсинг ветки, до Шага 2.2 |
| `front_task_id` / `front_task_num` | `dev-1653` / `1653` | URL фронта, diff фронт-репо |
| `back_task_id` / `back_task_num` | `dev-1745` / `1745` | URL API/admin, <logs>, diff бэк-репо |

**Standalone MR** (без `-ref-`): `front_* = back_* = task_*` — одна и та же задача.

**Paired MR** (`feature/DEV-X-ref-DEV-Y`): `front_*` и `back_*` разные.

**Правило:** в Шагах 3-12 всегда подставляй `front_*` / `back_*` в URL и <logs> templates. Для standalone они автоматически равны — отдельной ветви кода не надо. `task_id`/`task_num` после Шага 2 не используются (кроме `TASK_NUM` для <task-tracker> и заголовков).

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

Сохранить: `TASK_NUM` (uppercase, e.g. `DEV-1727`), `task_id` (lowercase, e.g. `dev-1727`), `task_num` (только цифры, e.g. `1727`), `asana_task` если уже фетчили.

Если DEV-XXXX не извлекается — стоп, спросить.

---

## Шаг 2 — MR в <vcs> + paired branch detection

### 2.1 — Найти исходный MR

Для каждого репо (backend, frontend, admin):
```
mcp__gitlab__list_merge_requests(project_id=<GITLAB_*>, state="all", search=TASK_NUM)
```

Выбрать MR где `source_branch` содержит TASK_NUM. Если несколько — самый свежий по `created_at`.

Сохранить для каждого найденного: `branch`, `mr_url`, `mr_iid`, `mr_state` (opened/merged/closed), `repo` (backend/frontend/admin).

Если ни в одном репо нет MR — стоп, сообщить.

**Fallback при недоступном <vcs>.** Если `list_merge_requests` тайм-аутит/возвращает ошибку — переключиться на локальные ветки:

```bash
git -C <repo> branch -r | grep "DEV-{task_num}"
```

Для каждого совпадения взять дату последнего коммита: `git -C <repo> log -1 --format=%ci origin/<branch>`. Активная ветка — самая свежая.

⚠️ Несколько веток с одним TASK_NUM — типичная ситуация (старая + переименованная с `-ref-Y`). Без даты коммита легко взять заброшенную ветку и упустить paired-зависимость. **Никогда не угадывать активную ветку по имени** — только по timestamp.

Без <vcs>: `mr_url`, `mr_iid`, `mr_state` неизвестны; `branch` берётся из локального match. Дальше идём в Шаг 2.2 как обычно.

### 2.2 — Paired branch detection

Из `source_branch` найденных MR извлечь паирность через regex `-ref-(?:DEV-)?(\d+)`:

| Где найден исходный MR | Что делать |
|---|---|
| **только в frontend** + есть `-ref-Y` | Это парная фронт-ветка. `front_task_*` = из исходного MR (X). `back_task_*` = из ref (Y). Зафетчить парный backend MR: `list_merge_requests(GITLAB_BACKEND, search="DEV-Y")` — добавить в анализ если найден. Если не найден — backend ветки нет, на review env поднят `main` бэка в namespace `dev-Y` (per CI). |
| **только в frontend** без `-ref-` | Standalone frontend MR. `front_* = back_* = task_*`. Backend на review env — `main` в namespace `dev-X`. |
| **только в backend** + есть `-ref-Y` | Парная бэк-ветка. `back_task_*` = из исходного MR. `front_task_*` = из ref. Зафетчить парный frontend MR. |
| **только в backend** без `-ref-` | Standalone backend MR. `back_* = front_* = task_*`. Frontend на review env — `main` в namespace `dev-X`. |
| **в обоих** (frontend и backend параллельно с одним TASK_NUM) | Редкий случай — один task с MR в обоих репо. Берём оба, `front_* = back_* = task_*`. |
| **только в admin** | Admin не имеет своего review env. См. Шаг 3 — `ENV_TYPE = stage` или спросить пользователя. |

После 2.2 сохранены:
- `front_mr` (или None)
- `back_mr` (или None)
- `front_task_id` / `front_task_num`
- `back_task_id` / `back_task_num`

Для **paired** случаев показать в чате: `"Paired: frontend dev-{front_task_num} ↔ backend dev-{back_task_num}"`. Это критично — ADMIN_URL и API_URL формируются из `back_*`, не из исходного task.

---

## Шаг 3 — Деплой

Состояние оценивается по основному MR (для paired — по тому, в чьём репо изначально искали, либо по обоим если состояния совпадают). Если состояния разные (например frontend opened, backend merged) — спросить пользователя.

**`merged`** → тестируем на stage. Установить `ENV_TYPE = stage`:
- `FEATURE_URL = ENV_STAGE_FRONT`, `API_URL = ENV_STAGE_API`, `ADMIN_URL = ENV_STAGE_ADMIN`

**`closed`** → спросить пользователя: _"MR закрыт без влития. Тестируем на общем stage или пропускаем?"_
- Пропустить → стоп
- Тестировать на stage → установить `ENV_TYPE = stage`, `FEATURE_URL = ENV_STAGE_FRONT`, `API_URL = ENV_STAGE_API`, `ADMIN_URL = ENV_STAGE_ADMIN`

**`opened`** → `mcp__gitlab__get_merge_request` → `head_pipeline.status`. Для paired — проверить пайплайны обоих MR (front_mr и back_mr если есть):

| status | Действие |
|---|---|
| `success` | `deploy_time = head_pipeline.finished_at`, продолжать. Для paired: `deploy_time = max(front, back)` — деплой готов когда оба прошли |
| `running`/`pending`/`created` | стоп, сообщить какой именно пайплайн ещё идёт |
| `failed`/`canceled` | стоп, дать `head_pipeline.web_url` упавшего |
| `null` | стоп, "пайплайн не запускался" |

Если success: установить `ENV_TYPE = feature`, собрать URLs из констант (раздел Constants) с подстановкой переменных:

- `FEATURE_URL` ← `ENV_FEATURE_FRONT` с `front_task_id`
- `API_URL` ← `ENV_FEATURE_API` с `back_task_id`
- `ADMIN_URL` ← `ENV_FEATURE_ADMIN` с `back_task_num`

⚠️ Для **paired** случаев `front_task_id ≠ back_task_id`. Подставлять буквально — иначе админка коннектится не на тот backend.

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

Для каждого MR из Шага 2 (front_mr, back_mr, и admin_mr если есть):

```bash
git -C <repo> diff origin/main...origin/<branch> --name-only
```

Ноль файлов — пропустить репо.

Для paired случаев diff читается **по обоим** MR — фронтовая правка и бэковая правка часто дополняют друг друга, и тестовые флоу должны учитывать обе.

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

**Frontend:** затронутые флоу, billing/credit UI, валидация форм. **UI context (без браузера):** для затронутых страниц читай карточки из `<product-dir>/ui-snapshots/output/` — `find <product-dir>/ui-snapshots/output -name "*<slug>*.md"`. Видишь текущий UI (headings, buttons, inputs) + 4 варианта (desktop/mobile × light/dark) + state-overrides. Подробнее → `<product-dir>/CLAUDE.md` секция "UI Snapshots Catalog". **<analytics> tracking** — grep по diff: `<analytics>\.track\|trackEvent\|logEvent\|<analytics>\.logEvent`. Найденные events выписать. Новые (не было в main) → отметить «требует verification после теста». Изменены аргументы существующего event → отметить «проверить что старая аналитика не сломалась».

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
| **UI changes** (`frontend/src/`) | **Responsive 375px** — обязательно. Открыть в DevTools → Device toolbar → 375×667 (iPhone SE). Проверить: текст не обрезан, кнопки кликабельны, превью-блоки скрываются если предусмотрено (`hidden sm:flex`), нижние панели не overflow. **Также** — проверить dark/light theme если затронуты цвета. |

---

## Шаг 9 — Автопроверки

Окно: `deploy_time` из Шага 3 как `startRfc3339`. Для merged MR — дата последнего коммита:
```bash
git -C <repo> log origin/<branch> -1 --format=%ci
```

**<logs> backend — anomaly detection** (не плоский фильтр, а сравнение с baseline):

Идея: ловим **все** ERROR-события на feature, сравниваем с stage за то же окно. Если pattern на feature ≈ stage → фон, silent. Если значимо выше/ниже — ⚠️.

Шаги:

1. **Query feature** (`ENV_TYPE = feature`):
   ```
   {service_name="LOKI_BACKEND"} |= `ERROR`
   ```
   Окно: с `deploy_time` до `now`.

2. **Query stage** за то же окно длительности (от `now - (now - deploy_time)` до `now`):
   ```
   {environment="LOKI_STAGE_ENV", service_name="LOKI_STAGE_BACKEND"} |= `ERROR`
   ```

3. **Группировка по pattern** на обеих сторонах. Pattern = `<METHOD> <path> -> <status>` (например `GET /api/overview -> 404`). Извлекать из лог-строки regex'ом, нормализовать ID-параметры в URL (`/users/123` → `/users/{id}`).

4. **Сравнение per pattern**:
   ```
   ratio = count_feature / max(count_stage, 1)
   
   if count_feature < LOKI_ANOMALY_MIN_COUNT and count_stage < LOKI_ANOMALY_MIN_COUNT:
       → silent (слишком мало для статистики)
   elif pattern только в feature (count_stage == 0) and count_feature >= LOKI_ANOMALY_MIN_COUNT:
       → ⚠️ "new on feature: {pattern} ({count_feature} events)"
   elif ratio >= LOKI_ANOMALY_RATIO_HIGH:
       → ⚠️ "rose: {pattern} ({count_feature} feature vs {count_stage} stage, {ratio}×)"
   elif ratio <= LOKI_ANOMALY_RATIO_LOW:
       → ⚠️ "dropped: {pattern} ({count_feature} feature vs {count_stage} stage)"
   else:
       → silent (фон)
   ```

5. **Output** в <wiki>-странице (Шаг 12):
   - Если аномалий нет: одна строка `✅ <logs> backend: N events feature ≈ M stage, no anomalies`
   - Если есть: таблица `pattern | feature | stage | verdict` с теми что попали в ⚠️
   - Полный список patterns не выводить — только anomalies, иначе шум

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
- Если флоу задевает <analytics> event (см. Шаг 7) — добавить шаг «<analytics> check: открыть https://app.<analytics>.com/analytics/<product> → Events → отфильтровать `event_type=<event_name>` last 1h → убедиться что событие пришло с ожидаемыми properties»
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
❌ OAuth/whitelisted redirect — выносить в `### Тестирование на stage` с причиной. **Только если** задача касается логина/auth-флоу или явно требует отдельной проверки через Google OAuth. Для остальных задач (UI-правки, не-auth фичи) этот раздел не нужен — на review env всё проверяется без OAuth.

---

## Шаг 10.1 — UI Snapshots lookup

**Триггер:** любая задача (фронт или бэк) где флоу упоминают конкретные страницы.

Из флоу (Шаг 10) извлечь все пути страниц. Для каждого пути взять slug последнего сегмента (например `/tools/photo-shoot` → `photo-shoot`, `/billing` → `billing`):

```bash
find <product-dir>/ui-snapshots/output -name "*<slug>*.md" | head -10
```

Из найденных путей:
- Категория = имя папки `NN-name` (второй сегмент после `output/`)
- Имя страницы = имя файла без `.md` (убрать state-prefix вида `state-*.../`, взять только имя файла)

Сгруппировать по категориям. Если несколько файлов одной страницы (разные state/viewport) — имя страницы указать один раз.

Результат (если что-то найдено):
```
**UI Snapshots:** https://ui-snapshots-<product>.pages.dev/viewer

**<категория-1>**
<page-name-1>, <page-name-2>

**<категория-2>**
<page-name-3>
```

Если ни одна страница не найдена в ui-snapshots — блок опустить полностью.

---

## Шаг 11 — Промежуточный репорт

Шаблон для **standalone**:

```
## Готово к созданию страницы — DEV-XXXX
**Env:** <FEATURE_URL>
**Admin:** <ADMIN_URL>
**MR:** <mr_url> (<state>, <repo>)
**Деплой:** ✅/❌

**Автопроверки:**
- <logs>: ...
- <error-monitoring>: ...

**Флоу:** N сценариев (P0: X, P1: Y, P2: Z)
**Bug Candidates:** ...

Создаю <wiki>-страницу...
```

Для **paired** строку `**MR:**` заменить блоком (далее — **PAIRED_MR_BLOCK**):

```
**MR (front):** <front_mr_url> (<state>) — dev-{front_task_num}
**MR (back):** <back_mr_url> (<state>) — dev-{back_task_num}
**Paired:** frontend dev-{front_task_num} ↔ backend dev-{back_task_num}
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

**Структура (standalone):**
```
## <TASK_NUM> — <название>

**Env:** <FEATURE_URL>
**Admin:** <ADMIN_URL>
**API:** <API_URL>
**MR:** <mr_url> (<mr_state>)

---
```

Для **paired** строку `**MR:**` заменить блоком **PAIRED_MR_BLOCK** (см. Шаг 11).

Остальная структура — общая для обоих случаев:
```
---

### Предусловие
Открыть: <FEATURE_URL>
<общие предусловия>

<если найдены UI Snapshots (Шаг 10.1)>
**UI Snapshots:** https://ui-snapshots-<product>.pages.dev/viewer

**<категория>**
<page-name-1>, <page-name-2>
</если>

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
<ВКЛЮЧАТЬ ТОЛЬКО если задача касается логина/auth-флоу или явно требует Google OAuth. Иначе раздел опустить полностью — review env покрывает всё.>
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
- ✅ UI Snapshots (Шаг 10.1): если флоу упоминают страницы — найти в ui-snapshots, добавить viewer-ссылку и имена в <wiki> под «Предусловие». Триггер — любая задача (фронт или бэк), не только frontend diff
- ✅ Paired detection: source_branch с `-ref-DEV-Y` → fetch парного MR в противоположном репо, считать `front_task_*` ≠ `back_task_*`
- ✅ ADMIN_URL: `adm.<product>.pro/login?env=preview-{back_task_num}` — `back_task_num` это **только цифры** (1745, не dev-1745)
- ❌ Подставлять `task_id` (с префиксом `dev-`) в env параметр — даст `preview-dev-1745`, админка не подключится
