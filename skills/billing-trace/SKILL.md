---
name: billing-trace
description: >-
  Трейс платежа в <product>: по payment_id, tx hash, payment_address или email
  находит DB-запись, <logs> webhook-логи, on-chain статус. Главная цель — понять
  почему платёж не подтверждён. Crypto (ForumPay) first; Stripe/другие провайдеры
  по упрощённой схеме.
  Trigger: "billing-trace", "трейс платежа", "почему платёж не подтверждён",
  "проверь оплату", "payment stuck", "крипто не прошло".
---

# billing-trace

## Constants

- `WEBHOOK_URL_CRYPTO` = `https://api.zncr.pro/api/v1/payments/crypto/webhook`
- `WEBHOOK_HANDLER` = `backend/app/handlers/crypto/webhook.py:56`
- `FORUMPAY_CLIENT` = `backend/infra/external/clients/http/forumpay.py:62`
- `LOKI_PROD` = `{environment="prod", service_name="backend"}`
- `LOKI_OPENMOV_STG` = `{service_name="openmov-backend-staging"}`
- `GRAFANA_PROD_DS` = `efgtpv5l3vqpsc`

Принимать что есть — любой из: payment_id, payment_address, tx hash (0x...), user email, reference_no.

---

## Stage 1 — DB lookup

### Crypto payment

```sql
-- По адресу или ID платежа (staging MCP или <metrics> prod datasource)
SELECT id, user_id, status, payment_address,
       amount, currency, created_at, expired_at, confirmed_at, updated_at
FROM crypto_payment
WHERE payment_address = '<address>'
   OR id = '<payment_id>';
```

```sql
-- По email пользователя (последние 5)
SELECT cp.id, cp.status, cp.payment_address,
       cp.created_at, cp.expired_at, cp.confirmed_at, cp.updated_at
FROM crypto_payment cp
JOIN "user" u ON u.id = cp.user_id
WHERE u.email = '<email>'
ORDER BY cp.created_at DESC LIMIT 5;
```

**Читаем статус:**

| status | Что значит |
|---|---|
| `waiting` | Платёж создан, ждёт on-chain транзакции или webhook |
| `processing` | Webhook получен, обрабатывается |
| `completed` | Платёж подтверждён, кредиты начислены |
| `failed` / `expired` | Истёкший или упавший платёж |

⚠️ `confirmed_at = NULL` при `status = waiting` → webhook не пришёл или не нашёл запись.
⚠️ `updated_at = NULL` → запись не менялась с момента создания.

> **Важно:** `mcp__postgres__query` подключён к **staging** БД. Для prod → <metrics> datasource `GRAFANA_PROD_DS` через `mcp__grafana__query_prometheus` или `describe_<data-warehouse>_table` / `query_<data-warehouse>`.

### Другие провайдеры

```sql
-- Stripe
SELECT id, status, checkout_session_id, created_at
FROM stripe_transaction WHERE user_id = (SELECT id FROM "user" WHERE email = '<email>')
ORDER BY created_at DESC LIMIT 5;

-- Общий лог кредитов (любой провайдер)
SELECT ct.source, ct.amount, ct.resulting_amount, ct.created_at, ct.meta
FROM credit_transaction ct
JOIN "user" u ON u.id = ct.user_id
WHERE u.email = '<email>'
ORDER BY ct.created_at DESC LIMIT 10;
```

---

## Stage 2 — <logs> webhook logs

### ForumPay webhook hits (по payment_id)

```
{environment="prod", service_name="backend"} |= "<payment_id>"
```

**Что ищем:**
- `status=201` → webhook получен и обработан успешно
- `"Crypto payment not found for payment_id: ..."` → webhook пришёл, но payment_id не найден в той БД
- `status=400` с `"payment not found"` → мискаст (ForumPay шлёт на неправильный backend)
- Нет записей вообще → webhook не долетел

### Если опеnmov staging / payment со staging.openmov.ai

Проверять **оба** stream-а параллельно:
```
{environment="prod", service_name="backend"} |= "<payment_id>"
{service_name="openmov-backend-staging"} |= "<payment_id>"
```

⚠️ ForumPay шлёт webhook по callback URL, настроенному для `POS_ID` на ForumPay-дашборде. В коде callback URL **не передаётся** в StartPayment — он зафиксирован в настройках POS на стороне ForumPay.

### Scale query (масштаб за период)

```
sum(count_over_time({environment="prod", service_name="backend"} |= "crypto/webhook" [1h]))
```

---

## Stage 3 — On-chain (только ForumPay / crypto)

Если есть tx hash (0x...):
1. Проверить <logs>: `|= "<tx_hash>"` — видел ли backend
2. Etherscan (для ETH/USDC): паттерн `https://etherscan.io/tx/<hash>` — статус, timestamp, to-address
   - `to` должен совпадать с `crypto_payment.payment_address`
   - Timestamp транзакции vs `crypto_payment.expired_at` — была ли в окне оплаты?
3. Если on-chain confirmed, но DB status=waiting → webhook не прошёл → Stage 4

---

## Stage 4 — Diagnostic tree (crypto stuck)

```
Status = waiting + on-chain confirmed?
│
├── Webhook logs в <logs> для payment_id?
│   ├── ДА, status=400 "not found" →
│   │   Misrouted webhook: ForumPay шлёт на api.zncr.pro,
│   │   но платёж в другой БД (openmov staging ≠ <product> prod).
│   │   Причина: один POS_ID для staging + prod → один callback URL.
│   │   Fix: разные POS_ID для каждой среды, или динамический notify_url.
│   │
│   ├── ДА, status=201 → payment completed в <product> prod DB,
│   │   но смотришь в staging DB. Проверь prod datasource.
│   │
│   └── НЕТ webhook логов →
│       Webhook не приходил. Варианты:
│       a) ForumPay POS callback URL указывает на другой хост
│       b) ForumPay не получил on-chain confirmation (нужно проверить
│          ForumPay dashboard для POS_ID)
│       c) Сетевой блок (таймаут, неправильный POS_ID)
│
└── Webhook logs есть, status confirmed →
    Кредиты начислены? Проверить credit_transaction по user_id.
    Если нет → баг в webhook handler (Stage 3 код).
```

---

## Stage 5 — Code check (если webhook handler упал)

Читать `WEBHOOK_HANDLER`:
- Как ищет payment: по `payment_id` или `payment_address`?
- Что делает при `not found`: 400 или silent?
- Transaction scope: кредиты начисляются внутри `transaction_manager`?
- Idempotency: проверяет ли `confirmed_at IS NOT NULL` перед начислением?

---

## Output

1. **Статус платежа** — DB snapshot: status, created/expired/confirmed_at
2. **Webhook history** — что приходило, когда, с каким результатом
3. **On-chain** — confirmed / не confirmed, в окне или нет
4. **Diagnosis** — конкретная причина из diagnostic tree
5. **Рекомендация** — fix options или "всё OK, кредиты начислены"

Если нашли баг → направить в `/bug-nominate` или `/task-create`. Не создавать <task-tracker>-задачи автоматически.

---

## Anti-patterns

- ❌ Смотреть только в staging DB и делать вывод про прод
- ❌ "Webhook не пришёл" без проверки <logs> (может быть, пришёл на другой backend)
- ❌ Считать on-chain confirmation достаточным — нужно ещё что webhook его обработал
- ❌ Один POS_ID для staging + prod — это архитектурный риск, фиксировать в диагнозе
