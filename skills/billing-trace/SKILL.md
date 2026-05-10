---
name: billing-trace
description: >-
  Methodology for tracing a payment through DB record → webhook logs →
  external settlement → diagnostic tree, to answer "why isn't this payment
  confirmed?". Tool-agnostic; covers the cross-environment-routing trap and
  the difference between "settlement confirmed" and "credits granted".
---

# billing-trace

A "stuck" payment is rarely one thing. It's a chain: the user paid
somewhere, the provider recorded it, an external settlement layer
confirmed it, the provider sent a webhook to your backend, your
handler updated the DB, your business logic granted credits. Each
hop has a failure mode. The methodology walks the chain from
visible-to-user (DB record) outward to the parts only the provider
can see.

## What you accept as input

Any of:

- `payment_id` from the provider
- `payment_address` (for blockchain-backed payments)
- transaction hash (for on-chain payments)
- user email
- reference / order number from the provider

The skill derives the rest. Don't refuse to start because you "only
have an email" — the email finds the user, the user finds recent
payments, the payments give you everything else.

## Stage 1 — DB lookup

The local DB is your fastest signal. Before going to logs or
external systems, see what your own backend thinks happened.

Pull the payment record by whatever identifier you have:

- By `payment_address` or `id` (one row)
- By user email (last 5, joined through user table)

Read the status field with care:

| status | What it means |
|---|---|
| `waiting` | Created, awaiting external settlement or webhook |
| `processing` | Webhook received, mid-handling |
| `completed` | Confirmed, business effects (credits, etc.) granted |
| `failed` / `expired` | Terminal failure |

Watch for these tell-tale states:

- `confirmed_at = NULL` while `status = waiting` → webhook never
  arrived, or arrived but didn't match
- `updated_at = NULL` (or = `created_at`) → record hasn't moved at
  all since creation

⚠️ **Environment trap.** If your DB MCP is connected to staging,
queries return staging data. A payment confirmed in production but
queried in staging looks "stuck". Always know which environment
your queries hit; for prod, route through whatever read-only path
exists (metrics dashboard datasource, read replica, etc.).

For non-blockchain providers, also check the provider-specific
transaction table and the generic credit-transaction log — the
generic log is the place to confirm whether credits actually got
granted, regardless of which payment provider produced them.

## Stage 2 — Webhook logs

The DB record tells you what your backend *thinks*. Logs tell you
what your backend *received*.

```
{environment="prod", service_name="backend"} |= "<payment_id>"
```

What to look for:

- HTTP `200`/`201` from webhook handler → received and processed
- `"payment not found for payment_id: ..."` → webhook arrived, but
  no matching DB record (cross-environment routing — see below)
- `400` with "not found" → webhook misrouted (provider sent to the
  wrong host)
- No log entries at all → webhook never reached your backend

### Cross-environment routing trap

Most payment providers configure the webhook callback URL **on
their dashboard, per provider account / POS ID** — not per-request
from your code. If staging and production share the same provider
account / POS ID, both environments share the same callback URL.
Result: prod payments fire webhooks at staging (or vice versa);
the receiving environment doesn't know about the payment_id from
the other DB.

Symptoms of this trap:

- Logs show webhook arrived with `400 not found` errors
- Production DB has the payment record but it's stuck `waiting`
- Staging logs (a separate stream) show the matching webhook
  arrival

Mitigation patterns:

- Separate provider POS IDs per environment
- Dynamic `notify_url` passed to the provider on payment creation
  (if the provider supports it)
- Environment-aware proxy that re-routes misdirected webhooks

When this trap is the diagnosis, document it explicitly so the team
knows the architectural risk.

### Parallel-environment search

If the user-flow could span environments (legacy staging app
mounted under prod domain; cross-env preview deployments), search
**both** log streams in parallel:

```
{environment="prod", service_name="backend"} |= "<payment_id>"
{service_name="<other-backend-service>"} |= "<payment_id>"
```

The webhook can land in only one — the other shows nothing.

### Webhook callback URL — gotcha

For most providers, the callback URL is **not** passed in the
"start payment" call from your code. It's pinned to provider
configuration. Don't assume your code has control over where the
webhook goes; check the provider dashboard.

## Stage 3 — External settlement check (provider-side)

Some payment types (cryptocurrency, bank transfer, etc.) have an
external settlement step before the provider sends a webhook.

If you have a transaction reference (chain hash, bank reference):

1. **Check your logs** — does the backend already mention this
   reference?
2. **Check the external explorer / status page** — is the
   transaction confirmed? Confirmed when? With what destination?
3. **Match destination** against your DB record's expected address
   / account
4. **Match timestamp** against the payment's `expired_at` — was it
   in the valid window?
5. **If externally confirmed but DB still `waiting`** → webhook
   step failed → back to Stage 2

External settlement confirmation alone doesn't mean credits were
granted. Many "stuck" payments are settled externally but the
webhook→DB→credits chain broke at one of the later steps.

## Stage 4 — Diagnostic tree

```
DB status = waiting + external settlement confirmed?
│
├── Webhook logs found for payment_id?
│   ├── YES, status=400 "not found" →
│   │   Misrouted webhook (cross-environment routing trap).
│   │   Provider's callback URL points at the wrong backend
│   │   for this payment's environment.
│   │
│   ├── YES, status=2xx → Payment completed in another DB;
│   │   you're querying the wrong environment.
│   │
│   └── NO logs found →
│       Webhook never arrived. Possible:
│       a) Provider callback URL points at a host you don't see
│       b) Provider hasn't received the external confirmation yet
│          (check provider dashboard)
│       c) Network block on the callback (timeout, bad credential)
│
└── Webhook logs found, status confirmed →
    Were credits actually granted?
    Check the credit-transaction table by user_id.
    No → bug in the webhook handler / business-logic layer.
```

The tree narrows from "is the chain complete?" down to a specific
hop where evidence breaks.

## Stage 5 — Code check (only if handler logic is the suspect)

If the diagnostic tree points at the handler:

- How does the handler look up the payment — by ID or by address?
- What does it do on `not_found` — 400, or silent?
- Is the credit-grant inside the same transaction as the
  status-update? If not, partial-failure can leave status updated
  with credits ungranted.
- Idempotency: does the handler check `confirmed_at IS NOT NULL`
  before granting credits? If not, repeated webhooks can
  double-grant.

These are the four common handler bugs in payment-completion code.

## Output

A trace document with these sections:

1. **Payment status** — DB snapshot: status, timestamps
2. **Webhook history** — what arrived, when, with what result
3. **External settlement** — confirmed or not, in window or not
4. **Diagnosis** — the specific branch of the tree that fits
5. **Recommendation** — fix options or "actually OK, credits
   granted"

If a bug is found → recommend filing through the dedicated bug
recording skill / ticket creation skill. **Don't create tickets
automatically.**

## Hard rules

- ✅ Always know which environment your DB query hits
- ✅ Treat "external settlement confirmed" as one hop in the chain,
  not as the end
- ✅ Check both environments for webhook logs when cross-environment
  routing is plausible
- ✅ Surface the misrouted-webhook architecture risk explicitly when
  it's the diagnosis
- ❌ Don't conclude "webhook didn't arrive" before checking logs in
  every plausible environment
- ❌ Don't conclude "payment is fine" from external confirmation
  alone — credits granted is the real "fine"
- ❌ Don't auto-create tracker tickets

## Anti-patterns

- ❌ Querying staging DB and reporting on prod state
- ❌ "Webhook didn't arrive" without checking logs, or checking
  only one stream when multiple are plausible
- ❌ Treating settlement confirmation as the endpoint — credits
  granting is downstream and can fail independently
- ❌ Ignoring that the provider callback URL is on the *provider*,
  not in your code — assuming code controls something it doesn't
- ❌ Missing the cross-environment-routing diagnosis because both
  environments share one provider account — surface it explicitly
- ❌ Concluding "ok, webhook handled" without verifying credits
  actually landed in the user's balance / credit log
