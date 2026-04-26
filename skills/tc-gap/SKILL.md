---
name: tc-gap
description: >-
  Gap analysis: compares existing <tms> TCs against available signal sources
  per project — <analytics> events (Web), backend handlers (Back), admin
  operations (Admin) — to find uncovered areas in <product>. Produces a
  prioritized gap report. Trigger: "tc-gap", "gap analysis", "что не покрыто",
  "найди пробелы", "какие кейсы пропущены".
version: 0.2.0
---

# TC Gap — Coverage Gap Analysis

Find what's not covered in `<tms>`. Signal sources differ per project:

| Project | Primary signal | Additional signal |
|---|---|---|
| Web | `<analytics>` named events | `<error-monitoring>` JS errors (frontend crashes, unhandled rejections) |
| Back | Backend handlers (`<backend-repo>/app/handlers/`) | `<error-monitoring>` Python errors (unhandled exceptions in handlers) |
| Admin | Admin panel pages and their operations (`<admin-repo>/src`) | — |

**`<error-monitoring>` MCP:** `mcp__<error-monitoring>__list_issues`, `mcp__<error-monitoring>__list_events`, `mcp__<error-monitoring>__get_resource`

Run automatically when invoked — no clarifying questions.

## Configuration

Replace these placeholders before use:

| Placeholder | What to set |
|---|---|
| `<product>` | Your product name |
| `<tms>` | Your test management system name |
| `<analytics>` | Your analytics tool |
| `<error-monitoring>` | Your error monitoring tool |
| `<wiki>` | Your knowledge base / wiki tool |
| `<backend-repo>` | Absolute path to your backend repository |
| `<admin-repo>` | Absolute path to your admin repository |
| `<YOUR_WIKI_TC_GAP_PAGE_ID>` | Page ID of your TC Gap wiki page |
| `<YOUR_WIKI_TC_GAP_URL>` | URL of your TC Gap wiki page |
| `<YOUR_ANALYTICS_PROJECT_ID>` | Your analytics project/app ID |
| Project IDs (1, 2, 3) | Your TMS project IDs for Web, Back, Admin |

## Workflow

### Step 0 — Read existing wiki page

Before doing anything else, fetch the current TC Gap page:
**Page ID:** `<YOUR_WIKI_TC_GAP_PAGE_ID>`

Use `mcp__<wiki>__<wiki>-fetch` with that ID.

Extract and carry into the analysis:
1. **Date of last run** — compare with today, understand how stale the data is
2. **Anomalies / under question** — anything marked ⚠️ in the previous report (e.g. `Payment Successful ≠ Payment Succeeded?`) → add to "re-check" list in current analysis
3. **"📌 Notes" section** — if present, save verbatim: it is written manually and is **never overwritten** during updates

---

### Step 1 — Fetch all TCs from <tms> (parallel)

```
mcp__<tms>__<tms>_list_testcases(project_id=<web-project-id>)   # Web
mcp__<tms>__<tms>_list_testcases(project_id=<back-project-id>)  # Back
mcp__<tms>__<tms>_list_testcases(project_id=<admin-project-id>) # Admin
```

Build flat lists per project. Count totals.

---

### Step 2 — Collect signals per project

#### Web → <analytics> events

1. `mcp__<analytics>__get_context` → get projectId for Production (`<YOUR_ANALYTICS_PROJECT_ID>`)
2. `mcp__<analytics>__search` with queries for known event families — configure for your product:
   ```
   ["Job", "Payment", "Registration", "User", "Email", "Account", "Sign In"]
   ```
   entityTypes: `["EVENT"]`, appIds: `[<YOUR_ANALYTICS_PROJECT_ID>]`
3. Filter out SDK-internal and experiment prefixes (e.g. `[Guides-Surveys]`, `[Experiment]`).
4. Result: list of named product events.

#### Back → Backend handlers

Read handler files to discover what's implemented:

```bash
find <backend-repo>/app/handlers -name "*.py" | sort
```

For each handler file, extract: filename + first docstring or comment describing what it handles.
Group into functional areas: billing, generation, assets, auth, webhooks.

**Optional: enrich with `<error-monitoring>` errors**

After reading handlers, check `<error-monitoring>` for high-frequency unhandled exceptions that might reveal untested error paths:

```
mcp__<error-monitoring>__list_issues(query="is:unresolved", limit=20)
```

For each `<error-monitoring>` issue not covered by a Back TC — add to the gap list with priority based on event count (HIGH >1000/day, MEDIUM 100–1000, LOW <100).

If code is unavailable, use a known handler map as fallback. Example structure for a SaaS product with billing and async generation:

| Area | Handlers |
|---|---|
| Billing — success | payment provider webhooks (stripe, paypal, etc.) |
| Billing — failure | payment error, webhook retry, duplicate webhook |
| Billing — auto-topup | low balance trigger |
| Generation | job create, job fail/rescue, credits refund |
| Assets | upload, batch upload, format validation, preview |
| Auth | OAuth, email registration, session handling |

#### Admin → Admin panel operations

Read admin source to discover pages:

```bash
find <admin-repo>/src -name "*.tsx" -path "*/pages/*" | sort
```

If unavailable, use a known operations map as fallback. Example structure:

| Page | Operations |
|---|---|
| Users / UserDetail | search, view profile, promote, grant credits |
| Jobs / JobDetail | view job, view status, view details |
| Models / ModelDetail | create model, edit model |
| Templates | create, edit, transfer stage→prod |
| AIAssets | manage AI model assets |
| Banners | create, edit banners |
| Promocodes | create promocodes |
| SubscriptionPlans | manage subscription plans |
| GenerationServers | monitor servers, manage |
| ToolRestrictions | set tool restrictions |
| PaymentRefunds | process refunds |
| Transactions | view transaction history |
| JobStats | analytics (multiple tabs) |

---

### Step 3 — Cross-reference per project

**Web:** For each `<analytics>` event, check if any Web TC title contains all words from the event name (case-insensitive). Mark ✅ / ❌.

**Back:** For each handler/area, check if any Back TC title contains keywords from that area. Match rule: at least one keyword from the handler description appears in the TC title.

**Admin:** For each page+operation pair, check if any Admin TC title covers it. Match rule: page name or operation keyword appears in TC title.

---

### Step 4 — Output gap report

```
## TC Gap Report — <product>
Date: YYYY-MM-DD
<tms>: N total TC (Web: X | Back: Y | Admin: Z)

### Web — <analytics> events (M checked, P% covered)

❌ Gaps:
| Event | Priority | Suggested TC title |
|---|---|---|

✅ Covered (top 5):
| Event | TC |
|---|---|

### Back — Backend handlers (M areas checked, P% covered)

❌ Gaps:
| Handler area | Priority | Suggested TC title |
|---|---|---|

✅ Covered:
| Handler area | TC |
|---|---|

### Admin — Panel operations (M ops checked, P% covered)

❌ Gaps:
| Page / Operation | Priority | Suggested TC title |
|---|---|---|

✅ Covered:
| Page / Operation | TC |
|---|---|
```

**Priority for gaps:**
- HIGH: payment/auth/generation critical path, or high analytics volume (>50K)
- MEDIUM: important feature with no coverage, medium volume (5K–50K)
- LOW: edge case, low volume (<5K), view-only events

---

### Step 5 — Update wiki page

After producing the report, update the TC Gap page in `<wiki>`:
**Page ID:** `<YOUR_WIKI_TC_GAP_PAGE_ID>`
**URL:** `<YOUR_WIKI_TC_GAP_URL>`

Use `mcp__<wiki>__<wiki>-update-page` with `command: replace_content`.

**Page structure — in this order:**

```
[Callout: run date + coverage by project]

[Auto-report: three sections Web / Back / Admin]

---

## 📌 Notes
[Content from Step 0 — copy verbatim if it existed.
If section didn't exist — create it empty. User adds to this manually.]
```

**Update rules:**
- Callout and three report sections — always overwritten with fresh data
- `## 📌 Notes` section — **always preserved**: take content from Step 0 and insert unchanged
- If Step 0 had no Notes section — create `## 📌 Notes` heading with empty body
- Never delete anything from Notes, even if it looks stale — it's manual content

### Step 6 (optional) — Create TCs for top gaps

If user says "create them" / "добавь кейсы":
- Apply tc-create workflow for each (top 5 by priority)
- Status: `GUESS` by default (source confirmed, steps not verified hands-on)
- Folder per project: see `../tc-create/references/<tms>-api.md`

## Fallback: if <analytics> is unavailable

Skip Web analytics signals. Run Back and Admin analysis only.
For Web, fall back to critical path check (payment, auth, generation, content moderation).

## Notes

- Web match is fuzzy word-bag (known limitation: "Post Scheduled" can false-match "Post Published: publishType=scheduled")
- When in doubt about a match, mark as ❌ and note the ambiguity
- Back and Admin matching is looser by design — handler areas are broad
- Do not ask the user for scope — run all three projects and let them filter
