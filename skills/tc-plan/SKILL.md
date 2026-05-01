---
name: tc-plan
description: >-
  Управляет составом тест-планов в <tms>. Два режима: preview (помечает TC через cf__planmove=Add/Remove)
  и apply (вносит изменения и очищает поле). Всегда таргетирует один конкретный план.
  Trigger: "обнови план", "tc-plan Smoke", "tc-plan apply Regression-Billing".
---

# tc-plan

## Constants

- `<tms>_WEB` = `1`
- `<tms>_BACK` = `2`
- `<tms>_ADMIN` = `3`

### Plan Criteria

| Plan | project_ids | status | priority | testcase_type | folder_ids |
|---|---|---|---|---|---|
| `Smoke` | 1,2,3 | ACTIVE,DRAFT | 1 | SMOKE | — |
| `Regression` | 1,2,3 | ACTIVE,DRAFT | — | REGRESSION | — |
| `Regression-Billing` | 1,2,3 | ACTIVE,DRAFT | — | REGRESSION | Billing folders: 1,17,24 |
| `Regression-Auth` | 1,2,3 | ACTIVE,DRAFT | — | REGRESSION | Auth folders: 13,16 |

Folder IDs → `../tc-create/references/<tms>-api.md`.

---

## Hard Rules

- Всегда работать только с одним планом за раз
- Preview не меняет состав плана — только ставит cf__planmove
- Apply без preview не запускать (нужен staging перед применением)
- Перед apply показать итоговый список что будет добавлено/убрано и ждать "да"

---

## Step 0 — Identify plan and mode

Из аргументов извлечь:
- `plan_name` (обязательно): один из плановых имён из Constants
- `mode`: `preview` (default) или `apply`

Если plan_name не распознан → вывести список доступных планов и остановиться.

---

## Step 1 — Get current plan state

```
mcp__<tms>__<tms>_list_testplans(project_id=<первый из plan.project_ids>)
```

Найти план по имени → получить `testplan_id`.
Если план не существует → предложить создать: `<tms>_create_testplan(project_id=..., title=<plan_name>)`.

Получить текущий состав плана (TCs в нём).

---

## Step 2 — Preview mode: вычислить diff

Получить все TC по критериям плана из Constants (status IN {ACTIVE, DRAFT}, type соответствует плану).

Вычислить:
- **ADD**: TC соответствует критериям плана, но не в плане → `<tms>_update_testcase(id=<id>, custom_fields={"cf__planmove": "Add"})`
- **REMOVE**: TC в плане, но не соответствует критериям (status не ACTIVE/DRAFT, или другой folder/priority/type) → `<tms>_update_testcase(id=<id>, custom_fields={"cf__planmove": "Remove"})`
- **No change**: в плане и соответствует → ничего не делать

Вывести сводку в чат:
```
Plan: <plan_name>
+ Добавить (N): TC-312, TC-315, TC-318
− Убрать (M): TC-89, TC-102
= Без изменений: K TC

Проверь cf__planmove в <tms> UI. Когда готова — запусти /tc-plan apply <plan_name>
```

---

## Step 3 — Apply mode: применить staging

Загрузить все TC с непустым cf__planmove для данного плана.

Показать финальный список:
```
Применяю к плану <plan_name>:
+ Добавить: [список]
− Убрать: [список]
Продолжить? (да/нет)
```

После "да":
- Для каждого ADD: добавить TC в план. Проверить доступные MCP-инструменты: `<tms>_add_testcases_to_run` добавляет в run, не в plan напрямую. Если есть `<tms>_add_testcases_to_plan` — использовать его. Иначе — уточнить у пользователя как в их <tms>-инстансе добавляются TCs в план.
- Для каждого REMOVE: убрать TC из плана аналогично
- Очистить cf__planmove: `<tms>_update_testcase(id=<id>, custom_fields={"cf__planmove": null})`

Вывести итог: "Обновлено: +N / -M. cf__planmove очищен."
