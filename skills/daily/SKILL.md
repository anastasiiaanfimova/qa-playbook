---
name: daily
description: >-
  Write daily log for <product> QA — two-block structure (продуктовое +
  техническое) with separate Дела/План plus shared Вопросы/сложности section.
  Pulls from <task-tracker>, <wiki> bug candidates, <tms>, episodic memory and diary.
  Supports backfill for yesterday. Pushes to weekly file in <your-qa-repo>
  GitHub repo.
  Trigger: "/daily", "напиши дейли", "дейли за сегодня", "дейли за вчера", "daily log".
---

# daily

## Constants

- `ASANA_WORKSPACE` = `1208919739404549`
- `ASANA_PROJECT` = `<YOUR_TASK_TRACKER_PROJECT_ID>`
- `ASANA_USER_ID` = `1214134527687814`
- `QA_REPO_URL` = `https://github.com/anastasiiaanfimova/<your-qa-repo>.git`
- `QA_REPO_LOCAL` = `/tmp/<your-qa-repo>`
- `STALE_DAYS_THRESHOLD` = `5`
- `<tms>_PROJECT_IDS` = `[1, 2, 3]` (Web, Back, Admin)
- `STATUS_ORDER` = `[to do, doing, testing, next release]`

---

## Цель

Сделать QA-работу видимой двум аудиториям сразу: продуктовым (которые сейчас не видят что делает QA) и техническим. Один документ — два независимых блока + общая секция вопросов.

---

## Структура секции дня

```
## День, MM-DD

### Продуктовое

**Дела**
- [нарративная фраза в продуктовых терминах, без task IDs]

**План на завтра**
- [направление работы]

### Техническое

**Дела**
- [статус] [Task name](<task-tracker>-url) — что делала → суть/вердикт

Также смотрела:
- [статус] [Task name](<task-tracker>-url) — комментарий: вердикт

**План на завтра**
- [статус] [Task name](<task-tracker>-url) — действие

### Вопросы / сложности

**Затыки сегодня**
- что было трудно → как разрулили

**Открытые вопросы**
- к кому вопрос / какая нужна помощь

---
```

- **Layout:** block-first. Удобно копировать целый блок на нужный митинг.
- **Status order внутри блока:** `to do → doing → testing → next release`. Один статус подряд.
- **Дни** добавляются в конец недельного файла, разделитель `---`.

### Пятничная сводка

Если TARGET_DATE = Пятница, в конец **продуктового** блока:

```
**За неделю**
- Bug candidates: N (M уже в <task-tracker>)
- Закрыто задач: K
- Прогнали testruns: L
- Создали TCs: P
```

Пропускается если активность за неделю нулевая.

**Что считается «Закрыто задач»** (ZC-specific): section transition в течение недели, не `completed=true` flag:
- задача была на мне в `Testing`, перешла в `Next release` (или дальше)
- задача на мне (`assignee=me`) перешла в `Done`

`completed_at`-фильтр ловит только второй случай. Основной поток (Testing → Next release) виден только через `asana_get_task_stories`.

---

## Style

### Продуктовый блок
- Нарративные фразы («обнаружили что», «готовим к фиксу», «докрутили»)
- Без task IDs, HTTP-кодов, имён классов, file paths
- Скиллы — через что они дают процессу («теперь автоматически проверяем X в продуктовых метриках»), не «обновили скилл X»
- MCP / Claude / infra не упоминать

### Технический блок
- Конкретно, с task links и статусами
- Action → суть/вердикт через `—` или `→`
- Скиллы — своим именем («обновила bug-dig: добавила <error-monitoring>-шаги для product layer»)
- MCP/инфра OK

### Общее
- От первого лица
- «Мы»-язык для командных действий, «я» для индивидуального
- Claude — инструмент, не автор («нашли», не «Claude нашёл»)
- Кратко, без padding

### Content mapping (hybrid)

Скиллы — в **оба** блока, разными формулировками. MCP / Claude / infra — **только** в технический.

---

## Pruning rules — когда не пишем

| Что | Когда не пишем |
|---|---|
| `Также смотрела:` | Чужих задач с моими комментариями нет |
| Bug candidates строка | Нет за день |
| <tms> строка | Нет TCs/runs за день |
| `Затыки сегодня` | Подтверждённых затыков нет |
| `Открытые вопросы` | Подтверждённых открытых нет |
| Секция `Вопросы / сложности` | Оба подпункта пусты |
| Yesterday reconciliation в продуктовом | Нет значимых переносов |
| Пятничная сводка | Не пятница, или нулевая активность |
| План — пункт со статусом | В этом статусе нет задач на мне |

---

## Источники

| Целевой блок | Источник | Запрос |
|---|---|---|
| Дела свои (тех) | <task-tracker> | `assignee=me`, `modified` в TARGET_DATE |
| Дела «Также смотрела» (тех) | episodic + <task-tracker> | task IDs из episodic, верификация через `get_task_stories` |
| Дела (прод) | tech + diary + <wiki> + <tms> | переформулировка |
| Bug candidates | <wiki> | Bug Candidates DB страницы создан/обновлён мной TARGET_DATE |
| <tms> | <tms> | TCs/runs мной TARGET_DATE. Пропуск если 0 |
| План | <task-tracker> + testing-plan.md | `assignee=me`, `completed=false`, фильтр section.name |
| Yesterday plan vs реальность | weekfile | секция «План на завтра» из TARGET_DATE - 1 |
| Вопросы / сложности | контекст + episodic + diary | гибрид-детект, кандидаты в чат |
| Stale tasks (только в чате) | <task-tracker> | `assignee=me`, секция не менялась >5 дней |
| Пятничная сводка | агрегация недели | bug candidates, closed, testruns, TCs |

### <task-tracker> queries

**Свои задачи где было движение в TARGET_DATE:**
```
asana_search_tasks(
  workspace=ASANA_WORKSPACE,
  projects_any=ASANA_PROJECT,
  assignee_any=ASANA_USER_ID,
  modified_at_after="<TARGET_DATE>T00:00:00Z",
  modified_at_before="<TARGET_DATE>T23:59:59Z",
  opt_fields="name,memberships.section.name,modified_at,completed"
)
```

**Задачи в работе (для Плана) — текущее состояние:**
```
asana_search_tasks(
  workspace=ASANA_WORKSPACE,
  projects_any=ASANA_PROJECT,
  assignee_any=ASANA_USER_ID,
  completed=false,
  opt_fields="name,memberships.section.name,modified_at"
)
```

Фильтрация в коде по `memberships.section.name in {to do, doing, testing, next release}`.

### <wiki> query (Bug Candidates)

`<wiki>-search` по Bug Candidates DB с фильтром `created_time` или `last_edited_time` = TARGET_DATE, фильтр по автору = me. Все совпавшие страницы — счётчик + список названий + ссылки. Если в схеме есть поле «<task-tracker> link» — отмечать продвинулось ли в <task-tracker>; если нет — отметка опускается.

### <tms> query

`<tms>_list_testcases` с фильтром по `updated_at` = TARGET_DATE, owner = me. Аналогично `<tms>_list_testruns`. Если оба пусты — секция и упоминание в продуктовом не пишутся.

### Status display

Статус в скобках = **текущий** column на момент записи дейли (не на TARGET_DATE).

При `TARGET_DATE != today` в чате префикс: «Дейли за <day>, <YYYY-MM-DD>. Статусы — актуальные на сейчас.»

Backfill scope: `today` (default) или `вчера`. За >1 день — с дисклеймером, без consistency warranties.

---

## Workflow

### Step 1 — Setup

Парсинг даты:
- no arg / `сегодня` → `TARGET_DATE = today`
- `вчера` → `TARGET_DATE = today - 1`
- `YYYY-MM-DD` → as-is (с дисклеймером если разница >1 день)

```bash
DOW=$(date +%u)
MONDAY_OFFSET=$((DOW-1))
MONDAY=$(date -v-${MONDAY_OFFSET}d +%Y-%m-%d)
FRIDAY_OFFSET=$((5-DOW))
FRIDAY_SHORT=$(date -v+${FRIDAY_OFFSET}d +%m-%d)
TARGET_SHORT=$(date -j -f %Y-%m-%d "$TARGET_DATE" +%m-%d)
WEEKFILE="daily/week-${MONDAY}_${FRIDAY_SHORT}.md"
```

Если TARGET_DATE из другой недели — пересчитать MONDAY/FRIDAY относительно TARGET_DATE.

### Step 2 — Параллельный сбор данных

Запустить параллельно (один message, несколько tool calls):

1. <task-tracker>: задачи `assignee=me`, `modified` в TARGET_DATE (для Дел свои)
2. <task-tracker>: задачи `assignee=me`, `completed=false` (для Плана)
3. <wiki>: bug candidates by me TARGET_DATE
4. <tms>: TCs/runs by me TARGET_DATE
5. `mcp__episodic-memory__search` по TARGET_DATE + темам ("комментарий", "затык", "task #")
6. `mempalace_diary_read last_n=3`
7. Чтение TARGET_DATE - 1 секции из weekfile (вчерашний План)

### Step 3 — Извлечение «Также смотрела»

Из episodic + diary взять упоминания task IDs которых нет в <task-tracker>-результатах Step 2.

Для каждого: `asana_get_task_stories(gid)` → проверить есть ли мой комментарий с `created_at` внутри TARGET_DATE. Только подтверждённые попадают в подсекцию.

### Step 4 — Stale tasks scan

Из <task-tracker>-плана отфильтровать задачи где section не менялся `> STALE_DAYS_THRESHOLD дней` (через `modified_at` или истории если нужно).

Показать в чате (НЕ в файле): «Висят в одном статусе: [список с ссылками]».

### Step 5 — Гибрид-детект для Вопросов

По episodic + diary + текущему разговору искать паттерны:
- «застряли», «не получалось», «починили», «разобрались» → кандидаты в **Затыки**
- «ждём от», «нужно от», «не хватает» → кандидаты в **Открытые вопросы**

Показать в чате как кандидатов, ждать confirm.

### Step 6 — testing-plan.md

```bash
cat $QA_REPO_LOCAL/testing-plan.md
```

Сопоставить с задачами Плана:
- есть упоминание + конкретика → ок
- есть упоминание без конкретики → флагать в чате, предлагать вариант, спросить
- нет упоминания → берём из <task-tracker>-статуса

### Step 7 — Yesterday reconciliation

Парсинг секции «План на завтра» из `TARGET_DATE - 1` в weekfile (если файл и секция существуют).

Сопоставить с фактическими делами TARGET_DATE:
- сделано → не упоминаем отдельно (уже в Делах)
- перенесено → готовим строку для продуктового («перенесли X с прошлого дня»)
- выпало → флагать в чате

### Step 8 — Черновики в чате

Префикс если `TARGET_DATE != today`:
> «Дейли за <day>, <YYYY-MM-DD>. Статусы — актуальные на сейчас.»

Показать последовательно:

1. Продуктовый блок (Дела + План)
2. Технический блок (Дела + Также смотрела + План)
3. Кандидаты Затыков и Открытых вопросов
4. Stale tasks (информативно, не в файл)
5. Уточнения по плану (флаги missing testing-plan info)

Ждать confirmation/правок.

### Step 9 — Пятничная сводка

Если `TARGET_DATE` = Пятница и есть данные за неделю:

Агрегация:
- <wiki>: bug candidates за неделю (created в `[MONDAY..TARGET_DATE]`)
- <task-tracker>: closed задачи на мне за неделю — пройти по `asana_get_task_stories` для всех моих задач, modified в неделю; искать `resource_subtype=section_changed` с переходами `Testing → Next release` (или дальше) и `* → Done`. `completed_at`-фильтр недостаточен.
- <tms>: testruns за неделю + TCs created за неделю

Добавить блок «За неделю» в конец продуктового блока.

### Step 10 — Запись + push

```bash
cd /tmp && git clone $QA_REPO_URL 2>/dev/null || true
cd $QA_REPO_LOCAL && git pull --quiet
```

- Нет файла `$WEEKFILE` → создать с заголовком `# Неделя ${MONDAY} / ${FRIDAY_SHORT}`
- Секция дня (по TARGET_DATE) есть → обновить
- Нет → добавить в конец, разделитель `---`

```bash
cd $QA_REPO_LOCAL
git add $WEEKFILE
git commit -m "daily: $TARGET_DATE"
git push
```

Дни недели (для заголовков): Понедельник, Вторник, Среда, Четверг, Пятница, Суббота, Воскресенье.

---

## Anti-patterns

- ❌ HTTP-коды / имена классов / file paths в продуктовом блоке
- ❌ Дублирование одной формулировки в обоих блоках
- ❌ «Обновила скилл X» в продуктовом без объяснения что это даёт процессу
- ❌ Заполнение секции Вопросы ради заполненности
- ❌ План продуктовый с task IDs
- ❌ «Claude нашёл / прочитал / обновил» — Claude инструмент
- ❌ Запись в файл без подтверждения (auto-detect кандидаты Затыков → требуют confirm)
- ❌ Stale tasks list в файле (только в чате)
- ❌ MCP / Claude infra в продуктовом блоке
- ❌ Промежуточные шаги в Делах («поиск документов», «отладка»)
- ❌ Подсекция «Свои:» как заголовок (свои задачи идут сразу под `**Дела**`)
- ❌ Backfill за >1 день без дисклеймера про несинхронные статусы
- ❌ Inline статус [doing → testing] на старте — не в MVP
