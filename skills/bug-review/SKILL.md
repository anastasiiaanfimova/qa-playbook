---
name: bug-review
description: >-
  Weekly refresh of <product> QA data sources + rebuild of Bug Candidates list.
  Two-layer flow: refresh individual source pages in <wiki> (<vcs>, <error-monitoring>,
  <metrics>, <analytics>, <data-warehouse>, <task-tracker>, <wiki> docs), then rebuild the "Bug
  Candidates" <wiki> DB with smart dedup against prior week. Main output is
  Bug Candidates — the input for bug-dig triage. Never auto-creates <task-tracker>
  tasks. Trigger: "bug-review", "weekly-review", "обнови ревью", "собери кандидатов", "свежий bug list".
---

# bug-review

## Constants

- `NOTION_REVIEW_PARENT` = `34998d8a3c9b806da443da66e4a7542c`
- `NOTION_QA_REVIEW` = `34898d8a3c9b81ffb128c375efe0f113`
- `NOTION_SRC_GITLAB` = `34898d8a3c9b8170a985e4b16841e510`
- `NOTION_SRC_ASANA` = `34898d8a3c9b81d0a060e2aa8665e537`
- `NOTION_SRC_GRAFANA` = `34898d8a3c9b81c0adeffc4de6a2f0ec`
- `NOTION_SRC_<data-warehouse>` = `34898d8a3c9b818c9e54f814d43bc10f`
- `NOTION_SRC_AMPLITUDE` = `34b98d8a3c9b813da281dbd908b6195b`
- `NOTION_SRC_NOTION_DOCS` = `34898d8a3c9b81b097efcd6b96bb1d4a`

`NOTION_SRC_SENTRY` создаётся при первом refresh под `NOTION_REVIEW_PARENT` (изначально не существует).

Bug Candidates DB schema → `references/<wiki>-schema.md`. Главный output — **Bug Candidates DB**, всё остальное вспомогательное.

---

## Sub-commands

| Cmd | Что |
|---|---|
| `/bug-review refresh [source?]` | Layer 1. Update source pages. Без arg — все. С arg (`<vcs>`/`<error-monitoring>`/`<task-tracker>`/`<metrics>`/`<analytics>`/`<data-warehouse>`/`<wiki>-docs`) — одна. |
| `/bug-review bugs` | Layer 2. Rebuild Bug Candidates DB. |
| `/bug-review qa` | Regenerate QA Review синтез. |
| `/bug-review all` | refresh → bugs → qa. |

**Cadence: гибкий.** Минимум — weekly. Daily ОК и даже полезно для активного мониторинга — при условии что Layer 2 использует ISO-week-based bumping счётчика повторов (см. `merge-algorithm.md`). Twice a day тоже безопасно: внутри одной ISO-недели runs идемпотентны для счётчика.

---

## Два разных временных окна

Скилл работает с **двумя независимыми окнами** — не путать:

### Delta window — "что нового с прошлого прогона"

Окно: `[last_run, today]`. Используется для discovery новых сигналов: новые <error-monitoring> issues, новые <task-tracker> BUG-задачи, новые коммиты. Гибкое — зависит от каденса прогонов.

### Trend window — "стало хуже / лучше WoW"

Окно: всегда `this_week = [today-7d, today]` vs `prior_week = [today-14d, today-7d]`. Фиксированное, **не зависит от `last_run`**. Используется для:
- <data-warehouse> failure rate per (tool, reason)
- <logs> error-class rate / ratio
- <analytics> error event volumes / funnel conversions
- <error-monitoring> issue frequency (events / users) сравнения

Если `last_run = today - 2 дня`, мы всё равно сравниваем последние 7 дней с предыдущими 7 — иначе deltas сжимаются и выглядят как "всё улучшилось" просто потому что окно стало короче.

**Ratio формула не зависит от длины окна — только от равенства окон между собой.**

---

## Sources

Automated: <error-monitoring>, <vcs> (3 repos), <task-tracker>, <metrics>/<logs>, <analytics>, <data-warehouse>, <wiki> docs.
Manual (НЕ в скилле): Trustpilot, Telegram dev chat.

References:
- `references/<wiki>-schema.md` — DB schema + IDs
- `references/merge-algorithm.md` — dedup/update логика
- `references/fingerprints.md` — fingerprints per source
- `references/sources/*.md` — per-source collectors

---

## Layer 1: Refresh

Для каждого source:

1. **Find `last_run`.** Read source page. Search `last_run:` в коде/параграфе (не HTML-комментарий — <wiki> escape'ит). Если нет — 7 days ago.
2. **Collect delta** через `references/sources/<source>.md`:
   - **Discovery** (новые сигналы) — окно `[last_run, today]`.
   - **WoW trend** (rate / ratio / volume сравнения) — фиксированное окно `last 7d` vs `prior 7d`, **независимо от `last_run`**.
3. **Update source page** — prepend `## Updates YYYY-MM-DD`, bump `last_run` (плейн code block: `` `last_run: YYYY-MM-DD` ``). Никогда не переписывать старое.
4. **Emit signals** — list `{fingerprint, title, source, severity_hint, signal_refs}` для Layer 2.

Source pages: `NOTION_SRC_GITLAB`, `NOTION_SRC_ASANA`, `NOTION_SRC_GRAFANA`, `NOTION_SRC_<data-warehouse>`, `NOTION_SRC_AMPLITUDE`, `NOTION_SRC_NOTION_DOCS`.
<error-monitoring>: создать под `NOTION_REVIEW_PARENT` если нет.

**First-run.** Source pages без `last_run` → 7 days back + добавить marker. <error-monitoring> не существует → создать под Review.

---

## Layer 2: Bugs

Применить `references/merge-algorithm.md` к Bug Candidates DB. **Все writes делегируются `/bug-nominate silent=true`** — Layer 2 сам в <wiki> не пишет.

1. Query все existing rows
2. Build `fingerprint → row_id` map
3. Per incoming signal — собрать args (`title`, `fingerprint`, `status`, `severity`, `sources`, `asana_link`, минимальный `body_markdown` = Symptom + Signal refs) и вызвать `/bug-nominate silent=true`. bug-nominate сам решит CREATE vs UPDATE по fingerprint:
   - new fingerprint → CREATE с trend=New, status=Active
   - existing & not Closed → UPDATE properties (без `body_markdown` → не трогать body bug-dig'а)
   - existing & Closed → передать `status=Regression` → bug-nominate перезапишет
4. Existing Active/Tracked НЕ matched в этом прогоне:
   - <error-monitoring> resolved / <task-tracker> closed → `/bug-nominate silent=true status=Closed trend=Gone`
   - else → `/bug-nominate silent=true trend=Declining` (без обновления даты последнего события)
   - 3 weeks Declining подряд → auto Closed
5. Compute Severity + rank score (см. merge-algorithm.md), передать в args
6. Log в чат после прогона: `N new | M regressed | K closed | total active = X`

**Никогда не удалять rows.** Closed — для regression detection.

### Почему silent mode

- bug-review batch-обрабатывает 50+ сигналов; интерактивный confirmation на каждом сделает скилл неюзабельным
- bug-nominate в silent mode пишет молча, но всё равно репортит в чат строку `✓ Bug Candidates: <CREATE|UPDATE> «...»` для каждой записи — сводный лог сохраняется
- Для всей schema-логики (mapping property names, dedup по fingerprint, защита body от перезаписи) — bug-nominate single source of truth. Layer 2 не дублирует.

### Что Layer 2 не делает

- ❌ Не пишет в <wiki> напрямую (всё через bug-nominate)
- ❌ Не дописывает root-cause анализ / план фиксов / вердикт — это работа bug-dig'а, который потом тоже идёт через bug-nominate
- ❌ Не передаёт `body_markdown` в UPDATE existing — иначе сотрёт investigation

---

## Layer 3: QA synthesis

Regenerate `NOTION_QA_REVIEW` from scratch. Pull from:
- Bug Candidates (top Active by score)
- Source pages `## Updates` last run
- Deltas: "N new this week", "M regressions", "K tracked → closed"

Если page `deleted=true` — recreate под `NOTION_REVIEW_PARENT` с title "QA Review", обновить `NOTION_QA_REVIEW` в Constants.

Структура страницы — стабильная, чтобы week-over-week diff читался.

---

## Output

В чат после `all` или `bugs`:
```
Weekly review YYYY-MM-DD — done
  Sources refreshed: 7/7
  Candidates: N new | M regressed | K auto-closed | X active total
  Top 5 by score:
    1. [High] ... (<error-monitoring> + <task-tracker>, 3 weeks)
    ...
  Run bug-dig on top candidates.
```

Не дампить все строки. Полная таблица в <wiki>.

---

## Source collectors

| Source | Produces candidates? | Recipe |
|---|---|---|
| <error-monitoring> | Yes | `sources/<error-monitoring>.md` |
| <vcs> | Yes (revert/hot-file/critical-path) | `sources/<vcs>.md` |
| <task-tracker> | Yes (open BUG tasks) | `sources/<task-tracker>.md` |
| <metrics>/<logs> | Yes (error-rate spikes) | `sources/<metrics>.md` |
| <analytics> | Yes (errors + funnel drops); new events = Low/FYI | `sources/<analytics>.md` |
| <data-warehouse> | Yes (tool failure rate) | `sources/<data-warehouse>.md` |
| <wiki> docs | **No** — context-only | `sources/<wiki>-docs.md` |

**First-run caveats:**
- <error-monitoring> source page нет — создать
- <data-warehouse> schema — derive from dashboard panel (Option A в `<data-warehouse>.md`), cache
- <wiki>-docs watched-doc IDs — resolve и cache на первом run

**Cross-source merge:**
- <task-tracker> → <error-monitoring>: regex `<ERROR-ID>-XXX` в <task-tracker> notes мерджит в <error-monitoring> candidate (<task-tracker>: fingerprint, Status=Tracked, <task-tracker> link)
- <vcs> → <error-monitoring>: hot-file classifier читает Active candidates, fingerprint commits
- <metrics> → <error-monitoring>: same handler + error class merged во второй pass

---

## Anti-patterns

- ❌ Append без `## Updates YYYY-MM-DD`
- ❌ Skip `last_run` update → дублирующиеся signals
- ❌ Удалить Closed rows
- ❌ Auto-create <task-tracker> tasks
- ❌ Non-deterministic fingerprints (timestamps, free-text)
- ❌ Использовать `[last_run, today]` для WoW rate/ratio/volume — для трендов окно ВСЕГДА фиксированное `last 7d vs prior 7d`
- ❌ Bump счётчика повторов если ISO-неделя последнего события совпадает с текущей. Daily-runs идемпотентны
- ❌ Manual Trustpilot/Telegram сигналы автоматически в Bug Candidates
