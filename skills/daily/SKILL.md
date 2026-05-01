---
name: daily
description: >-
  Write today's daily log for the <your-qa-repo> GitHub repo — concise, first-person,
  only what was actually done today (no QA tools/TMS unless mentioned).
  Trigger: "/daily", "напиши дейли", "дейли за сегодня", "daily log".
---

# daily

## Constants

- `ASANA_WORKSPACE` = `1208919739404549`
- `ASANA_PROJECT` = `<YOUR_TASK_TRACKER_PROJECT_ID>`
- `ASANA_USER_ID` = `1214134527687814`
- `QA_REPO_URL` = `https://github.com/anastasiiaanfimova/<your-qa-repo>.git`
- `QA_REPO_LOCAL` = `/tmp/<your-qa-repo>`

---

## Style

- Кратко, конкретно, от первого лица
- Claude — инструмент, не автор
- Не упоминать QA-репозиторий и связанное
- Не упоминать TMS/инструменты тестировщика если не всплывали сегодня
- Не брать пункты из qa playbook если не обсуждались
- В плане — sub-notes под каждым пунктом если есть уточнения

---

## Format задач в «Дела»

```
Завела задачи:
- [Название](https://app.<task-tracker>.com/1/ASANA_WORKSPACE/project/ASANA_PROJECT/task/{gid})

Работала с задачами:
- [Название](https://app.<task-tracker>.com/1/ASANA_WORKSPACE/project/ASANA_PROJECT/task/{gid}) — одна фраза что делала
```

- **Завела** — созданные мной (<task-tracker> с `created_by_any=ASANA_USER_ID`)
- **Работала с** — чужие задачи где комментировала/расследовала/обновляла. Из episodic memory + MemPalace (<task-tracker> API не фильтрует по комментариям)
- Блок «Работала с» только если были такие; иначе — пропустить
- Подзадачи не включать — только parent

---

## Структура недельного файла

`week-YYYY-MM-DD_MM-DD.md` (понедельник_пятница).

```
# Неделя YYYY-MM-DD / MM-DD

## Понедельник, MM-DD

### Дела
- ...

### Вопросы
- ...

### План
- [ ] ...
  → ...

---

## Вторник, MM-DD
...
```

---

## Workflow

### Step 1 — Даты недели

```bash
DOW=$(date +%u)
MONDAY=$(date -v-$((DOW-1))d +%Y-%m-%d)
FRIDAY=$(date -v+$((5-DOW))d +%Y-%m-%d)
FRIDAY_SHORT=$(date -v+$((5-DOW))d +%m-%d)
TODAY_SHORT=$(date +%m-%d)
WEEKFILE="daily/week-${MONDAY}_${FRIDAY_SHORT}.md"
```

### Step 2 — Что сделано сегодня

**Источники (все обязательны):**
1. Текущий разговор (уже в контексте)
2. Другие сессии сегодня — `mcp__episodic-memory__search` по дате + теме (`"2026-04-28 <product>"`). НЕ `read` — too large
3. MemPalace diary — `mempalace_diary_read last_n=3`
4. <task-tracker> задачи созданные сегодня:
```
mcp__asana__asana_search_tasks(
  workspace=ASANA_WORKSPACE,
  projects_any=ASANA_PROJECT,
  created_by_any=ASANA_USER_ID,
  created_at_after="<TODAY>T00:00:00Z",
  created_at_before="<TODAY>T23:59:59Z",
  opt_fields="name,created_at"
)
```
Из результата — только имена. Не дублировать с другими источниками.

**Filter для «Дела» — только результаты:**
- ✅ Создала: задачи в <task-tracker>, документы, тест-кейсы
- ✅ Расследовала / закрыла: баги, инциденты, задачи
- ✅ Улучшила в продакте/процессе
- ❌ Промежуточные шаги (поиск документов, форматирование, отладка)
- ❌ Действия Claude (нашёл, прочитал, обновил скилл)
- ❌ Мелочи обслуживания (формат <wiki>, заголовки)
- ❌ Внутренняя QA-кухня (скиллы, MCP/хуки, MemPalace, чистка)

### Step 3 — План на завтра

```bash
cat $QA_REPO_LOCAL/testing-plan.md
```
2-4 пункта с учётом сегодня + очереди. Конкретные задачи, не категории. Показать черновик, ждать confirmation.

### Step 4 — Уточнения

Если что-то непонятно — **спросить перед записью**.

### Step 5 — Repo + write

```bash
cd /tmp && git clone $QA_REPO_URL 2>/dev/null || true
cd $QA_REPO_LOCAL && git pull --quiet
```
- Нет файла `$WEEKFILE` → создать с заголовком `# Неделя ${MONDAY} / ${FRIDAY_SHORT}`
- Секция дня есть → обновить
- Нет → добавить в конец, разделитель `---`

### Step 6 — Push

```bash
cd $QA_REPO_LOCAL
git add $WEEKFILE
git commit -m "daily: $(date +%Y-%m-%d)"
git push
```

Дни недели: Понедельник, Вторник, Среда, Четверг, Пятница, Суббота, Воскресенье.
Сегодняшняя дата — из `currentDate` или спросить.
